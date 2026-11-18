# 更新日志

## 1.0.1 — 2026-10-12

- 运行启动辅助方法现在会在其构建的 `RUN_STARTED` 事件上将 `ProtocolVersion` 设为 `AGUIProtocol.Version`，这是 AG-UI 1.0 的要求。调用方传入的 `RunStartedEvent` 原样透传。事件流的快照测试或基线测试会新增 `protocolVersion` 字段。

### 破坏性变更

无。

## 1.0.0 — 2026-09-17

- 迁移到生成的 1.0 模型。
- 新增对 AG-UI 客户端状态的类型化访问。

### 破坏性变更

- 模型跟随 1.0 Schema 名称；升级后请重新验证类型和序列化行为。