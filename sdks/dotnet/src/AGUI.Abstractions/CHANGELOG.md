# 更新日志

## 1.0.1 — 2026-10-12

- 按照 AG-UI 1.0 的要求，即使宿主将序列化器上下文插入自己的 `JsonSerializerOptions`，无值的可选字段现在也不会写入 JSON。在 .NET 10 上的 Microsoft Agent Framework 下，事件曾带有 `null` 可选字段（如 `parentRunId`、`input`、`subagentRunId`）。记录了这些 `null` 键的快照测试或基线测试需要更新。

### 破坏性变更

无。

## 1.0.0 — 2026-09-17

- 模型现在由冻结的 AG-UI 1.0 JSON Schema 生成，涵盖内容部分、工具结果部分、能力、运行结果、含缓存写入的 token 用量以及文件来源。
- `PROTOCOL_VERSION` 现在为 "1.0"；生成的请求头使用 1.0 schema 地址。
- 生成器会生成 `AGUIProtocol` 类型。

### 破坏性变更

- 模型已重命名为 1.0 名称；请对照生成的类型检查现有用法。
- 序列化时会省略可选 null，但载荷中的 null 会保留。
- 协议版本常量值已改为 "1.0"。