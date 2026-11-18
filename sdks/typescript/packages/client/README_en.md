# @ag-ui/client

用于连接 **Agent-User Interaction (AG-UI) Protocol** 服务器的客户端 SDK。

`@ag-ui/client` 提供处理 AG-UI 通信完整生命周期的 Agent 实现：连接服务器、处理流式事件、管理状态变更，并提供响应式订阅者 Hook。

## 安装

```bash
npm install @ag-ui/client zod
pnpm add @ag-ui/client zod
yarn add @ag-ui/client zod
```

zod（3.25.18+ 或 4.x）是 peer 依赖。客户端的强制校验阶段会用 `@ag-ui/core/schemas` 上的 Schema 校验每个事件，使用的是你应用安装的 zod，而非内置副本。

## 特性

- 🔗 **HTTP 连接** – `HttpAgent` 支持 SSE/protobuf，可直接连接服务器
- 🏗️ **自定义 Agent** – `AbstractAgent` 基类，用于构建自己的传输层
- 📡 **事件流** – 完整的 AG-UI 事件处理，含校验与转换
- 🔄 **状态管理** – 自动跟踪消息/状态，支持响应式更新
- 🪝 **订阅者系统** – 中间件风格的 Hook，用于日志、持久化和自定义逻辑
- 🎯 **中间件支持** – 用函数或类中间件转换和过滤事件

## 快速示例

```ts
import { HttpAgent } from "@ag-ui/client";

const agent = new HttpAgent({
  url: "https://api.example.com/agent",
  headers: { Authorization: "Bearer token" },
});

const result = await agent.runAgent({
  messages: [{ role: "user", content: "Hello!" }],
});

console.log(result.newMessages);
```

## 强制校验限制

Agent 收到的每个事件，以及 `HttpAgent` 发送的每个输入，都会被剥离并对照协议 Schema 校验。为控制开销，客户端会拒绝单个事件或输入中容器条目超过 1,000,000 个的情况（数组元素、对象键和记录键；`state` 等不透明字段不计入）。超出时运行失败，抛出 `AGUIEnforcementLimitError`，并指明超限的路径。

```ts
// 提高限制，或传入 Infinity 禁用限制。
const agent = new HttpAgent({ url, maxEnforcedEntries: 5_000_000 });
```

该限制只覆盖剥离和校验。在此之前执行的单次线性遍历（如解析传输流）不计入，不透明字段也完全不受限：巨大的 `state` 能通过强制校验，并在下游以完整开销被克隆和应用。

## 使用中间件

```ts
import { HttpAgent, FilterToolCallsMiddleware } from "@ag-ui/client";

const agent = new HttpAgent({
  url: "https://api.example.com/agent",
});

// 添加中间件以转换或过滤事件
agent.use(
  // 函数中间件用于日志
  (input, next) => {
    console.log("Starting run:", input.runId);
    return next.run(input);
  },

  // 类中间件用于过滤工具调用
  new FilterToolCallsMiddleware({
    allowedToolCalls: ["search", "calculate"]
  })
);

await agent.runAgent();
```

## 文档

- 概念与架构：[`docs/concepts`](https://docs.ag-ui.com/concepts/architecture)
- 完整 API 参考：[`docs/sdk/js/client`](https://docs.ag-ui.com/sdk/js/client/overview)

## 贡献

欢迎提交 Bug 报告和 Pull Request！请先阅读我们的[贡献指南](https://docs.ag-ui.com/development/contributing)。

## 许可证

MIT © 2025 AG-UI Protocol Contributors

## 活动快照

`MESSAGES_SNAPSHOT.metadata["@ag-ui/client"].authoritativeActivityTypes` 控制快照替换哪些活动类型。数组表示拥有这些类型；`[]` 表示不拥有任何类型；`null` 表示拥有所有类型，即使快照为空。未声明时适用现有规则：包含任何活动的快照替换所有活动，而仅含对话记录的快照保留它们。命名空间或字段缺失时使用此旧规则。命名空间或字段无效时不拥有任何类型，包括包含非字符串条目的数组。

授权控制被省略活动消息的删除。匹配的消息 ID 始终原地更新，即使其类型不在声明范围内。现有消息位置保持不变，新 ID 按快照顺序追加。此约定不协调变更的 ID，也不重新定位恢复的活动。

历史投影器必须保留完整授权，并用其重建的类型扩展显式范围。`withAuthoritativeActivityTypes` 实现了此规则；在替换快照消息之前，先对传入的快照调用它。