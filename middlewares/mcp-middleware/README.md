# MCP 中间件

AG-UI 中间件，将一次 agent 运行连接到一个或多个 [MCP](https://modelcontextprotocol.io) 服务器。它列出每个服务器的工具，将其注入运行，在服务端执行产生的工具调用，并循环运行 agent 直到没有剩余的 MCP 工具调用——对消费者而言，整个过程呈现为一次连续的运行。

## 用法

```ts
import { MCPMiddleware } from "@ag-ui/mcp-middleware";

agent.use(
  new MCPMiddleware([
    {
      type: "http",
      url: "https://example.com/mcp",
      serverId: "example",
      headers: { Authorization: "Bearer <token>" },
    },
  ]),
);
```

## 行为

- **工具注入。** 服务器上报的每个工具都以 `mcp__{serverId}__{tool}` 命名空间暴露给 agent（清洗为 `[a-zA-Z0-9_-]`，截断至 64 个字符，冲突时以 `_N` 后缀去重）。省略 `serverId` 时默认为 `server{index}`。列表仅在每个中间件实例上获取一次并缓存。
- **执行循环。** 当一次已结束的运行留下未处理的 MCP 工具调用时，中间件会执行它们（并行），为每个调用发出 `TOOL_CALL_RESULT`，并在没有其他未处理调用时，附带结果启动另一次运行。如果仍有非 MCP 工具调用未处理（例如前端工具），则停止并交由前端处理。不指向已注入 MCP 工具的工具调用永远不会被触碰。
- **单次运行呈现。** 整个多轮循环对消费者而言看起来像一次运行：首个 `RUN_STARTED` 被转发，后续 `RUN_STARTED` 事件被抑制，仅在循环停止时刷新一个终止 `RUN_FINISHED`。
- **失控保护。** `maxIterations`（默认 `32`）限制工具执行轮数。取值会被钳制为正整数。

## 配置

```ts
interface MCPClientConfig {
  type: "http" | "sse";
  url: string;
  serverId?: string;
  headers?: Record<string, string>;
}

interface MCPMiddlewareOptions {
  maxIterations?: number; // default 32
}
```

按请求认证可通过为每个请求构造中间件并设置 `headers` 来实现——它们会通过传输层的 `requestInit` 附加到出站 MCP 请求上。

> **SSE 注意事项：** 对于 `sse` 传输，`headers` 仅作用于 POST 通道；SSE 事件流使用 `eventSourceInit`。当 headers 必须覆盖所有流量时，优先使用 `http`（可流式）传输。