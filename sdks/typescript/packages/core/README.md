# @ag-ui/core

**Agent-User Interaction (AG-UI) 协议**的 TypeScript 定义与运行时 Schema。

`@ag-ui/core` 提供所有其他 AG-UI 包所依赖的强类型基础构件：消息与状态模型、运行输入，以及完整的流式事件类型。

## 安装

```bash
# 仅类型和常量 — 无运行时依赖
npm install @ag-ui/core
pnpm add @ag-ui/core
yarn add @ag-ui/core

# 如需使用 @ag-ui/core/schemas 上的校验器，请安装 zod（3.25.18+ 或 4.x）
npm install @ag-ui/core zod
```

zod 是可选的 peer 依赖：主入口从不加载它，因此只需要类型的应用无需安装其他依赖。

## 特性

- 🧩 **类型化数据模型** – `Message`、`Tool`、`Context`、`RunAgentInput`、`State` …
- 🔄 **流式事件** – 30 多种事件类型，覆盖助手消息、工具调用、状态更新、推理、活动，以及运行和子代理生命周期。
- ✅ **运行时校验** – Schema 可及早捕获格式错误的载荷。
- 🚀 **框架无关** – 可在 Node.js、浏览器及任何能输出 JSON 的代理框架中运行。

## 快速示例

```ts
import { EventType } from "@ag-ui/core";
// 校验器位于 /schemas 子路径，需要安装 zod；主入口仅包含类型和常量，
// 因此仅使用类型的消费者无需 zod。
import { EventSchemas } from "@ag-ui/core/schemas";

// 校验传入的事件
EventSchemas.parse({
  type: EventType.TEXT_MESSAGE_CONTENT,
  messageId: "msg_123",
  delta: "Hello, world!",
});
```

## 文档

- 概念与架构：[`docs/concepts`](https://docs.ag-ui.com/concepts/architecture)
- 完整 API 参考：[`docs/sdk/js/core`](https://docs.ag-ui.com/sdk/js/core/overview)

## 贡献

欢迎提交 Bug 报告和 Pull Request！请先阅读我们的[贡献指南](https://docs.ag-ui.com/development/contributing)。

## 许可证

MIT © 2025 AG-UI Protocol Contributors