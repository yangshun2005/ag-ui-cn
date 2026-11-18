# 更新日志

## 0.0.2 — 2026-09-11

- 从 MCP 服务器列出工具，以 `mcp__{server}__{tool}` 命名空间注入（截断至 64 字符并去重），并在服务端执行工具调用。
- 多轮工具循环现在表现为单次运行：续轮时抑制 RUN_STARTED，仅在循环停止时发出 RUN_FINISHED。
- 通过 `requestInit` 将 `serverConfig.headers` 转发给两种传输方式，用于按请求鉴权；按实例缓存 `listTools` 结果。
- 将工具结果同步到下游 `agent.messages`，使后续轮次消费结果而非重复发出调用。
- 缓冲 RUN_FINISHED 直至工具结果发出，避免 RUN_FINISHED 之后的校验失败。
- 将 `onRunComplete` 的拒绝路由到 `subscriber.error`，并保护 `client.close()`，防止失败挂起流。
- SSE 传输改为延迟导入，避免 Bun 下 `eventsource` 加载时失败。
- 在服务端记录工具执行和按服务器列出的失败。
- 添加 MIT LICENSE 文件及发布包的 license 字段。

### 破坏性变更

- `@ag-ui/client` 从依赖项移至 peer 依赖（>=0.0.40）；使用者需自行安装。