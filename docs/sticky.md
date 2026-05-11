# 滑动窗口粘性 Key 轮询策略设计

## 背景

当前普通分组的 API Key 选择使用 Round-Robin 策略。每次请求都会从分组的 `active_keys` 列表中轮询选择一个 Key。该策略负载分散，但连续请求容易使用不同 Key，无法利用上游在同一 Key 下可能存在的短期缓存。

目标是在保持现有 Round-Robin 行为不变的前提下，新增一种简单的滑动窗口粘性策略。该策略在窗口内复用同一个 Key，窗口过期或该 Key 触发现有失败处理后，再回到 Round-Robin 选择下一个 Key。

## 目标

- 新增 Key 选择策略配置，支持 `round_robin` 和 `sticky`。
- 新增粘性空闲窗口配置，单位为分钟，默认 30 分钟。
- 支持全局配置和普通分组级覆盖，沿用现有“分组配置 > 系统设置 > 默认值”的生效逻辑。
- 不改变现有 Round-Robin 的实现和默认行为。
- 不改变聚合分组的子分组选择策略。
- 聚合分组选中子分组后，子分组按自己的有效 Key 选择策略执行。
- 尽量减少代码改动，不新增数据库表或字段。

## 非目标

- 不实现按用户、IP、请求头或代理 Key 的多维度粘性。
- 不实现 Key 权重、额度、冷却时间或优先级策略。
- 不改变现有 Key 失败判断、失败计数和黑名单逻辑。
- 不改变现有请求日志和 `last_used_at` 更新逻辑。
- 不要求配置变更后立即清理已有粘性缓存。
- 不要求并发过期瞬间严格只选择一个 Key。

## 配置设计

新增两个动态配置项：

```text
key_selection_strategy
sticky_key_idle_timeout_minutes
```

`key_selection_strategy` 支持：

```text
round_robin
sticky
```

`sticky_key_idle_timeout_minutes` 为整数，默认值为 `30`，建议最小值为 `1`。

全局配置保存到现有 `system_settings` 表，不需要修改表结构。示例：

```text
setting_key = key_selection_strategy
setting_value = sticky

setting_key = sticky_key_idle_timeout_minutes
setting_value = 30
```

分组配置保存到现有 `groups.config` JSON 字段，不需要修改表结构。示例：

```json
{
  "key_selection_strategy": "sticky",
  "sticky_key_idle_timeout_minutes": 30
}
```

配置生效逻辑沿用现有模式：

```text
分组配置 > 系统设置 > 默认值
```

聚合分组自身的 Key 选择配置不用于覆盖子分组。聚合分组只负责选择子分组；实际使用哪个 Key，由选中的子分组自己的有效配置决定。

## 运行态数据设计

粘性绑定只属于实际承载请求的标准分组。运行态数据保存在现有 Store 中，可以是 Redis 或 MemoryStore。

缓存 Key：

```text
group:{groupID}:sticky_key
```

缓存 Value：

```text
key_id
```

缓存 TTL：

```text
sticky_key_idle_timeout_minutes
```

每次命中粘性 Key 时刷新 TTL，以实现滑动窗口语义。

采用 Store TTL 的原因是实现简单，符合现有缓存抽象。配置从较大值改为较小值时，已有粘性缓存可能保留旧 TTL，直到自然过期。该行为为可接受的简化边界。

## 请求流程

### Round-Robin 策略

当有效配置为 `round_robin` 时，完全保持现有行为：

```text
调用现有 SelectKey(group.ID)
从 active_keys 列表执行 Rotate
```

### Sticky 策略

当实际承载请求的标准分组有效配置为 `sticky` 时：

```text
1. 读取 group:{groupID}:sticky_key。
2. 如果 sticky_key 存在，读取 key:{keyID} 详情。
3. 如果 Key 存在且 status == active，则使用该 Key，并刷新 sticky_key 的 TTL。
4. 如果 sticky_key 不存在、已过期、Key 不存在或 Key 非 active，则删除 sticky_key，并调用现有 SelectKey(group.ID)。
5. SelectKey 成功后，将选中的 Key 写入 sticky_key，并设置 TTL。
6. 使用选中的 Key 发起上游请求。
```

命中 sticky 时不推进 Round-Robin 队列。只有 sticky 缺失、过期、失效或失败清除后，才调用现有 Round-Robin 逻辑。

示例：

```text
active_keys 初始轮询顺序：1 -> 2 -> 3
首次请求调用 Rotate 选中 1，并设置 sticky = 1
30 分钟滑动窗口内继续使用 1，不推进轮询队列
超过窗口后 sticky 过期，下次调用 Rotate 选中 2，并设置 sticky = 2
```

## 聚合分组行为

聚合分组仍按现有平滑加权轮询选择子分组，不改变该策略。

请求入口为聚合分组时：

```text
1. 聚合分组按现有逻辑选择一个子分组。
2. 实际请求落到该子分组。
3. 子分组按自己的有效 Key 选择策略选择 Key。
```

如果子分组配置为 `sticky`，则使用子分组自己的粘性缓存：

```text
group:{subGroupID}:sticky_key
```

如果子分组配置为 `round_robin`，则保持现有轮询行为。

一次请求的失败重试仍在已选中的子分组内进行，不重新选择聚合分组的子分组。

## 失败处理

Key 失败判断完全沿用现有逻辑。只有当前代码会调用失败处理的位置，才清除 sticky。

也就是说，当请求错误或上游状态码命中现有故障转移规则，并触发：

```text
UpdateStatus(apiKey, group, false, errorMessage)
```

时，同时删除该标准分组的 sticky 绑定：

```text
group:{groupID}:sticky_key
```

之后如果还有重试次数，下一次重试会调用现有 Round-Robin 选择下一个 Key，并重新写入 sticky。

不应因为以下错误清除 sticky，除非它们进入现有 Key 失败处理路径：

- 客户端中断等可忽略错误。
- 请求体读取失败。
- 参数覆盖失败。
- 模型重定向配置错误。
- 没有可用 Key。

如果最终所有重试都失败，sticky 应为空或被最后一次失败清除，避免下一次请求继续粘到失败 Key。

## 边界行为

### 单 Key 分组

单 Key 分组下，`sticky` 和 `round_robin` 行为接近一致。失败后会清除 sticky，但如果该 Key 未被拉黑，下一次 Round-Robin 仍可能选回同一个 Key。

### Key 被删除或失效

命中 sticky 后必须校验对应 Key 是否存在且状态为 `active`。如果不满足，删除 sticky 并回退到现有 Round-Robin。

### 并发过期

当 sticky 过期后，多个并发请求可能同时发现 sticky 不存在，并分别调用 Round-Robin 选出不同 Key。第一版接受该短暂竞争，不引入分布式锁或 SetNX 抢占逻辑。

### 并发失败与成功

同一个 sticky Key 被并发请求使用时，可能出现一个请求失败清除 sticky，另一个请求成功刷新 sticky 的竞争。第一版接受最终一致行为。若该 Key 持续失败，现有失败计数和黑名单逻辑会继续生效。

### 流式请求

流式请求按请求开始时刷新 sticky TTL，不等待流式响应结束。该选择实现简单，并与 Key 选择发生在请求转发前的现有流程一致。

### 配置变更

策略从 `sticky` 改为 `round_robin` 后，代码不再读取 sticky 缓存，旧缓存自然过期。

策略从 `round_robin` 改为 `sticky` 后，新请求开始写入 sticky 缓存。

粘性窗口从较大值改为较小值时，已有 sticky 缓存可能保留旧 TTL，直到自然过期。第一版不做主动清理。

## 预期改动范围

后端预计涉及：

- 在 `types.SystemSettings` 增加全局配置字段。
- 在 `models.GroupConfig` 增加分组覆盖字段。
- 在系统设置和分组配置的有效配置合并逻辑中接入新增字段。
- 在 i18n 文案中增加配置名称和说明。
- 在 `keypool.KeyProvider` 增加 sticky 选择和清除方法，复用现有 `SelectKey`。
- 在代理请求失败路径清除 sticky。

前端预计涉及：

- 设置页依赖后端元数据动态渲染，新增全局配置后会自动展示。
- 分组配置页依赖后端分组配置选项，新增配置选项后可按现有方式展示。
- 如需更好的选择体验，可将 `key_selection_strategy` 从普通输入优化为下拉框；第一版可先按现有动态配置机制处理。

数据库预计影响：

- 不新增表。
- 不新增列。
- 不需要数据库迁移。

## 测试建议

需要覆盖以下行为：

- 默认配置下仍使用 Round-Robin。
- 全局配置为 sticky 时，普通分组窗口内复用同一个 Key。
- sticky 窗口过期后，调用 Round-Robin 选中下一个 Key。
- sticky Key 请求失败后清除绑定，重试选择下一个 Key。
- 分组配置能覆盖全局配置。
- 聚合分组选中子分组后，子分组按自己的策略选择 Key。
- sticky 命中的 Key 不存在或非 active 时，回退到 Round-Robin。

## 推荐结论

第一版采用分组级滑动窗口粘性策略。它不引入用户维度粘性、不改变现有 Round-Robin 数据结构、不修改数据库表结构，也不改变现有失败判断逻辑。该方案能满足连续请求复用同一 Key 的缓存诉求，同时将改动范围控制在配置合并、Key 选择和失败清除三个局部区域。
