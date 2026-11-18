# 更新日志

## 1.0.1 — 2026-10-12

- 与 `AGUI.Abstractions` 和 `AGUI.Server` 1.0.1 同步发布；本包无变更。

### 破坏性变更

无。

## 1.0.0 — 2026-09-17

- 客户端现在声明其使用的协议版本：每个 `RunAgentInput` 都携带 "1.0"，该值读取自生成的 `AGUIProtocol.Version`。
- 客户端会校验生产者的协议版本，与 TypeScript 客户端的握手行为一致。
- 已迁移至生成的 1.0 模型。

### 破坏性变更

- 协议版本为生成的 `AGUI.Abstractions.AGUIProtocol.Version`；其旁不再有手写常量。
- 线上协议版本现以 "1.0" 发送；模型名称与 null 处理遵循 1.0 Schema。