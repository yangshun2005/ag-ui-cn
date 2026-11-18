# @ag-ui/encoder

**Agent-User Interaction (AG-UI) 协议**的事件编码工具。

`@ag-ui/encoder` 负责 AG-UI 事件的内容协商和格式编码。它根据客户端的 `Accept` 请求头，自动在 Server-Sent Events (JSON) 和 Protocol Buffers 之间选择，确保最优的传输效率。

## 安装

```bash
npm install @ag-ui/encoder zod
pnpm add @ag-ui/encoder zod
yarn add @ag-ui/encoder zod
```

zod（3.25.18+ 或 4.x）是 peer 依赖。Protobuf 编码使用 `@ag-ui/core/schemas` 上的 Schema 校验每个事件，使用的是你的应用所安装的 zod，而非内置副本。

## 特性

- 🎯 **内容协商** – 根据 `Accept` 请求头自动选择格式
- 📦 **双编码** – 支持 SSE (JSON) 和 Protocol Buffer
- ⚡ **高效二进制** – 长度前缀的 protobuf 编码，适用于高吞吐场景
- 🔄 **无缝回退** – 不支持 protobuf 时优雅降级为 SSE

## 快速示例

```ts
import { EventEncoder } from "@ag-ui/encoder";
import { EventType } from "@ag-ui/core";

const encoder = new EventEncoder({
  accept: "application/vnd.ag-ui.event+proto, text/event-stream",
});

const event = {
  type: EventType.TEXT_MESSAGE_CONTENT,
  messageId: "msg_123",
  delta: "Hello, world!",
};

// Returns protobuf-encoded binary data
const encoded = encoder.encodeBinary(event);
```

## 文档

- 概念与架构：[`docs/concepts`](https://docs.ag-ui.com/concepts/architecture)
- 完整 API 参考：[`docs/sdk/js/encoder`](https://docs.ag-ui.com/sdk/js/encoder)

## 贡献

欢迎提交 Bug 报告和 Pull Request！请先阅读我们的[贡献指南](https://docs.ag-ui.com/development/contributing)。

## 许可证

MIT © 2025 AG-UI Protocol Contributors