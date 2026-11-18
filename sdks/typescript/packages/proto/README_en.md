# @ag-ui/proto

用于 **Agent-User Interaction (AG-UI) Protocol** 事件的 Protocol Buffer 编码/解码。

`@ag-ui/proto` 使用 Protocol Buffers 为 AG-UI 事件提供高性能二进制序列化。它包含生成的 TypeScript 定义，以及用于在 AG-UI 的 JSON 事件格式与紧凑二进制表示之间转换的工具。

## 安装

```bash
npm install @ag-ui/proto zod
pnpm add @ag-ui/proto zod
yarn add @ag-ui/proto zod
```

zod（3.25.18+ 或 4.x）是 peer 依赖。`encode` 使用 `@ag-ui/core/schemas` 上的 Schema 校验每个事件，所用 zod 来自你的应用安装的版本，而非内置副本。

## 特性

- ⚡ **高性能** – 二进制 protobuf 编码，带宽占用最小
- 🔄 **往返安全** – JSON 与二进制格式之间可兼容转换
- 📋 **生成的类型** – 从 `.proto` Schema 自动生成 TypeScript 定义
- 🔧 **长度前缀** – 流式协议使用标准 4 字节长度头
- 🖼️ **多模态快照** – 支持消息快照中的多模态用户内容部分

## 快速示例

```ts
import { encode, decode, AGUI_MEDIA_TYPE } from "@ag-ui/proto";
import { EventType } from "@ag-ui/core";

const event = {
  type: EventType.TEXT_MESSAGE_START,
  messageId: "msg_123",
  role: "assistant",
};

// Encode to binary protobuf format
const encoded = encode(event);

// Decode back to AG-UI event
const decoded = decode(encoded);
console.log(decoded); // Original event object
```

## 文档

- 概念与架构：[`docs/concepts`](https://docs.ag-ui.com/concepts/architecture)
- 完整 API 参考：[`docs/sdk/js/proto`](https://docs.ag-ui.com/sdk/js/proto)

## 贡献

欢迎提交 Bug 报告和 Pull Request！请先阅读我们的[贡献指南](https://docs.ag-ui.com/development/contributing)。

## 许可证

MIT © 2025 AG-UI Protocol Contributors