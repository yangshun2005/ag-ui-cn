# 跨语言 E2E 测试：.NET ↔ TypeScript 互操作

## 目标

验证 C# 服务端和 C# 客户端在端到端场景中与 TypeScript 生态**完全互操作**——不仅是线格式一致性，而是完整链路：用户发送消息，服务端与 LLM 通信，将响应转换为 AG-UI 事件，客户端产出正确的最终结果。

具体而言：
1. **C# 服务端 + TS 客户端**：现有 dojo e2e 测试（Playwright）目前针对 TS 服务端后端运行，也应针对 C# ASP.NET Core 服务端运行——证明 C# 服务端产出的事件能被 TS 客户端（CopilotKit React）正确渲染。
2. **TS 服务端 + C# 客户端**：针对 TS 服务端运行的相同场景，应能被 C# `AGUIChatClient` 消费——证明 C# 客户端能正确处理真实 TS 服务端输出，并产出相同的最终结果。

C# 服务端的响应可能与 TS 服务端略有差异（分块方式、字段顺序、可选字段不同），但从客户端视角必须产出**相同的最终结果**。

## 现有 E2E 测试的工作方式

### 技术栈

```
┌──────────────────────────────────────────────────────────────────┐
│  Playwright（浏览器）                                             │
│  └── CopilotKit React UI（TS 客户端，消费 AG-UI 事件）            │
│      └── HttpAgent → POST /agui → SSE 事件                       │
├──────────────────────────────────────────────────────────────────┤
│  AG-UI 服务端（如 server-starter、langgraph-typescript 等）       │
│  └── 调用上游 LLM API（OpenAI chat/completions）                  │
├──────────────────────────────────────────────────────────────────┤
│  @copilotkit/aimock（LLMock，端口 5555）                          │
│  └── 用基于 fixture 的响应模拟 OpenAI API                         │
│      （如 "I am duaa" → "Hello duaa!"）                          │
└──────────────────────────────────────────────────────────────────┘
```

### 关键组件

- **LLMock**（`@copilotkit/aimock`）：运行在端口 5555，拦截对 `OPENAI_BASE_URL` 的调用。根据消息内容匹配返回预设的补全/工具调用。
- **Fixtures**（`apps/dojo/e2e/fixtures/openai/*.json`）：按用户消息定义 LLM 响应（如 "I am duaa" → 文本响应，"background color to blue" → 工具调用）。
- **Dojo 应用**（`apps/dojo/`）：带 CopilotKit 的 React 应用，针对多个服务端后端渲染聊天 UI。
- **Playwright 测试**（`apps/dojo/e2e/tests/`）：断言 UI 行为——消息出现、工具执行、状态更新渲染。

### 现有 .NET 覆盖

已有 `microsoftAgentFrameworkDotnetTests/` 文件夹，含一个基础测试：
```typescript
test("[MS Agent Framework .NET] Agentic Chat sends and receives a message", async ({ page }) => {
  await page.goto("/microsoft-agent-framework-dotnet/feature/agentic_chat");
  const chat = new AgenticChatPage(page);
  await chat.openChat();
  await chat.sendMessage("Hi, I am duaa");
  await chat.assertAgentReplyVisible(/Hello/i);
});
```

这证明该模式已可行。我们需要**扩展它**，覆盖其他服务端测试的所有场景。

## 针对真实 LLM 录制 Fixtures

跨语言 Vitest fixtures（`tests/CrossLanguage.Vitest/fixtures/*.json`）是确定性的，但可以**从真实 LLM 录制**，以确保 C# 服务端符合实际模型输出——与 .NET 集成测试同一原则。

### 拓扑

AIMock 是 LLM 的替身；它位于 C# 服务端与真实模型之间，而非客户端与服务端之间：

```
录制：  TS 客户端 ──AG-UI──► C# 服务端 ──OpenAI──► AIMock（代理 + 保存）──► 真实 LLM
回放：  TS 客户端 ──AG-UI──► C# 服务端 ──OpenAI──► AIMock（匹配 fixture，无网络）
```

Fixture 捕获的是 **LLM 的 OpenAI chat-completion 响应**，而非 C# 服务端的 AG-UI 输出。客户端↔服务端的 AG-UI 交换在每次回放时实时、确定性地重新推导，因此回放仍会每次执行真实的 C# 映射——只有 LLM 被冻结。

### 工作原理

`helpers/llmock.ts` 暴露 `record` 选项，调用 AIMock 的 `enableRecording`。当请求没有匹配的 fixture 时，AIMock 将其代理到配置的上游，将折叠后的响应保存到 `fixtures/recorded/`（gitignored），并转发回来。由于 C# 服务端对 AIMock 使用纯 OpenAI 协议（`POST /v1/chat/completions`），AIMock 将上游 base 与路径拼接——因此将上游指向 Azure 的 OpenAI **v1** 接口（`https://<resource>.cognitiveservices.azure.com/openai`）会得到 `.../openai/v1/chat/completions`。认证原样转发：C# 服务端将 `OPENAI_API_KEY` 作为 bearer token 呈现，因此 Entra ID（AAD）token 可用于 Azure。

`helpers/record-config.ts` 从环境解析录制配置（见下文），并在未提供 `OPENAI_API_KEY` 时通过 `az account get-access-token` 生成 AAD token。

### 环境变量

| 变量 | 用途 |
| --- | --- |
| `AIMOCK_RECORD=true` | 启用录制（代理未匹配的调用）。 |
| `AZURE_OPENAI_ENDPOINT` | Azure 资源端点；上游变为 `<endpoint>/openai`。 |
| `AIMOCK_RECORD_UPSTREAM` | 显式上游 base（覆盖 Azure 推导）。 |
| `OPENAI_CHAT_MODEL_ID` | 模型 / Azure 部署（录制模式默认 `gpt-5-mini`）。 |
| `OPENAI_API_KEY` | 显式 key/token；否则通过 `az` 生成 AAD token。 |

### 工作流（PowerShell）

```powershell
cd sdks/dotnet/tests/CrossLanguage.Vitest
$env:AIMOCK_RECORD = "true"
$env:AZURE_OPENAI_ENDPOINT = "https://<resource>.cognitiveservices.azure.com"
$env:OPENAI_CHAT_MODEL_ID = "gpt-5-mini"
# 先 az login；AAD token 会自动生成。
npx vitest run tests/<scenario>.test.ts
```

录制**填补空缺**：已提交的 fixtures 仍会优先加载，因此删除某个已提交 fixture（或省略它）可强制其场景重新录制。捕获的文件落在 `fixtures/recorded/`，供你整理为命名的 `fixtures/*.json`。

### 多轮场景：匹配工具结果轮，而非用户消息

工具调用后，客户端会重放**同一对话**并附加工具结果——最后一条*用户*消息不变。因此基于 `userMessage` 匹配的第 2 轮 fixture 会重新匹配第 1 轮 fixture 并循环。修复方法是匹配**工具结果轮**：

- 编程方式：`predicate: (req) => req.messages.at(-1)?.role === "tool"`。
- 在 JSON fixtures 中（我们的情况，因为 `predicate` 是函数），任选：
  - **`toolCallId`** — 精确匹配请求**最后一条消息**的 `tool_call_id`，该消息本身必须是 `role: "tool"`。埋在对话更早位置的工具结果*不*符合：aimock 1.16.4 将此从"历史中任意位置最近的 `role: "tool"` 消息"收紧，因为宽松形式会让 `toolCallId` fixture 遮蔽新用户轮次的 `userMessage` 匹配器。
  - **`hasToolResult: true | false`** — 最简单的两段判别器，自 1.37.3 起限定于*当前*轮（最后一条 `user` 消息之后的消息）。当你只需判断"是否已有工具结果返回？"时优先用它而非 `toolCallId`，因为它不硬编码录制 id（重新录制时会变）。
  - **`toolResultContains: "<substring>"`** — 用于共享同一 `tool_call_id`、仅工具结果载荷不同的段。

将特定工具结果 fixture 列在**最前**，使其在第 2 轮胜出；第 1 轮（无工具消息）落到 `userMessage` fixture。这与轮数无关——无需 `sequenceIndex`。已提交的 `mixed-tool-invocation.json` 正是此形状：

```jsonc
{
  "fixtures": [
    // 第 2 轮：续接的最后一条工具结果是 get_weather 的（服务端经 FICC 解析），
    // 因此匹配其 call id 并返回最终文本。
    { "match": { "toolCallId": "call_XHrYFpdLMi6841ix1cwXZyy6" },
      "response": { "content": "Your current city is Tokyo, Japan. ... Berlin ..." } },
    // 第 1 轮：首次请求，尚无工具结果——暴露两个工具调用。
    { "match": { "userMessage": "What is my current city and the forecast for Berlin?" },
      "response": { "toolCalls": [ /* get_user_location, get_weather */ ] } }
  ]
}
```

#### 从 LLM 捕获两轮

**一次录制即可捕获所有轮次。** `buildFixtureMatch` 在每次捕获上标记 `userMessage`、`model`、`turnIndex`（`assistant` 消息数）和 `hasToolResult`，并注册到实时 fixture 池。因此第 2 轮不会匹配第 1 轮的捕获——它们在 `hasToolResult` 上不同，这是硬性拒绝门——所以请求未命中，代理到上游并被录制。两轮都落在 `fixtures/recorded/`。

本节过去描述的两遍 `sequenceIndex` 舞蹈仅在 aimock 1.19.3 之前必要，当时捕获仅以 `userMessage` 为键，第 1 轮会遮蔽第 2 轮。

**将捕获整理为已提交 fixture 时，决定额外匹配键的去留，而非原样粘贴：**

- **去掉 `model`**，除非你确实需要。录制默认 `gpt-5-mini`（`helpers/record-config.ts`），而回放默认 `gpt-4o`（`helpers/dotnet-server.ts`），因此捕获的 `"model": "gpt-5-mini"` 在回放时永不匹配——fixture 静默未命中，你会得到 404 且无明显原因。
- **去掉 `turnIndex`**，除非你想要基于深度的选择。本仓库所有 fixture 目前都省略它，且 `selectByTurnIndex` 仅当至少一个候选携带它时才重排——因此多个 fixture 文件依赖的"先注册先匹配"顺序成立*正是因为*无人设置它。引入一个会改变这一点。
- **保留 `hasToolResult`**，如果场景有工具结果段；它比固定录制的 `toolCallId` 更持久。

整理后，清除 `OPENAI_API_KEY`/`AIMOCK_RECORD` 并重新运行，确认场景离线回放通过。

> **针对推理模型录制：** `RecordConfig` 默认 `upstreamTimeoutMs` 和 `bodyTimeoutMs` 均为 30s。推理模型思考阶段的 token 发射间隙可能超过此值。触发时捕获会被**完全丢弃**——不写 fixture——aimock 在 error 级别记录 `Proxy request failed: Upstream response timed out after 30s`，客户端看到 502 或流被销毁。遇到时调高 `bodyTimeoutMs`。

## 方案

### 场景 A：为 C# 服务端扩展 Dojo E2E 测试

C# 服务端（`AGUIDojoServer` 或新变体）支撑与 `server-starter-all-features` 测试相同的 dojo 路由。我们用 C# 后端复用**相同的 Playwright 测试**。

**我们需要：**
1. 一个 C# 服务端，处理与 `server-starter-all-features` 相同的功能：
   - Agentic 聊天（文本流式）
   - 后端工具调用
   - 前端/客户端工具调用（human-in-the-loop）
   - 共享状态（状态快照 + 增量）
   - 自定义事件
   - 推理/思考事件

2. C# 服务端使用 `Microsoft.Extensions.AI` 的 `IChatClient` 调用 LLM——指向 LLMock（`http://localhost:5555/v1`）而非真实 OpenAI。

3. C# 服务端已注册为 dojo 路由（`/microsoft-agent-framework-dotnet/feature/{feature}`）——Playwright 直接导航到它。

4. 扩展 `.NET` 后端的 Playwright 测试，覆盖与 `serverStarterAllFeaturesTests` 相同的所有场景。

**实现路径：**
- 现有 `samples/AGUIClientServer/AGUIDojoServer/` 已有工具调用、状态管理等。
- 它已接入 dojo 应用，由 `run-dojo-everything.js` 在端口 8016 启动
- 确保设置 `OPENAI_BASE_URL=http://localhost:5555/v1`，让 LLMock 处理 AI 响应
- 扩展 `microsoftAgentFrameworkDotnetTests` 测试，匹配 `serverStarterAllFeaturesTests` 覆盖

### 场景 B：C# 客户端针对 TS 服务端

与上述相同但方向相反。我们将 TS `server-starter` 作为真实进程运行，由 LLMock 支撑（与 dojo e2e 测试完全一致），并用 C# `AGUIChatClient` 连接。

C# 测试项目：
1. 启动 LLMock（端口 5555）——与 dojo 的 `aimock-setup.ts` 相同
2. 启动 TS server-starter（端口 5100）——通过 `OPENAI_BASE_URL` 指向 LLMock
3. 用 `AGUIChatClient` 连接到 `http://localhost:5100/agui`
4. 发送与 Playwright 测试相同的消息
5. 断言 `AGUIChatClient` 收到正确的 `ChatResponseUpdate` 输出（相同消息、相同工具调用、相同内容）

```
┌──────────────────────────────────────────────────────────────────┐
│  C# 集成测试（xUnit）                                             │
│  └── AGUIChatClient → POST http://localhost:5100/agui → SSE      │
├──────────────────────────────────────────────────────────────────┤
│  TS AG-UI 服务端（server-starter，真实进程，端口 5100）           │
│  └── 调用 http://localhost:5555/v1/chat/completions（LLMock）    │
├──────────────────────────────────────────────────────────────────┤
│  @copilotkit/aimock（LLMock，真实进程，端口 5555）                │
│  └── 从 fixtures/openai/*.json 返回预设响应                       │
└──────────────────────────────────────────────────────────────────┘
```

## 详细设计

### 目录结构

```
sdks/dotnet/tests/
├── AGUI.CrossLanguage.IntegrationTests/    # C# 客户端 → TS 服务端测试
│   ├── AGUI.CrossLanguage.IntegrationTests.csproj
│   ├── TsServerFixture.cs                  # 管理 LLMock + TS 服务端进程
│   ├── AgenticChatTests.cs                 # 聊天场景
│   ├── ToolCallScenarioTests.cs            # 后端/前端工具场景
│   ├── BackendToolRenderingTests.cs        # 后端工具渲染
│   ├── HumanInTheLoopTests.cs              # 审批 / 恢复场景
│   ├── StateEventsTests.cs                 # 共享状态场景
│   ├── ActivitySnapshotTests.cs            # Activity 事件
│   ├── PassthroughEventTests.cs            # 自定义/原始事件透传
│   └── ReasoningTests.cs                   # 推理/思考场景
└── CrossLanguage.Vitest/                   # Vitest：TS 客户端 → C# 服务端测试
    ├── package.json
    ├── vitest.config.ts
    ├── helpers/
    │   ├── dotnet-server.ts                # 启动/停止 C# 服务端进程
    │   ├── step-server.ts                  # 启动/停止 GettingStarted 步骤服务端
    │   ├── global-setup.ts                 # Vitest 全局设置
    │   ├── record-config.ts                # 从环境解析录制模式
    │   ├── transport.ts                    # SSE/protobuf 传输参数化
    │   └── llmock.ts                       # 带 fixtures 启动/停止 LLMock
    ├── server/
    │   ├── main.ts                         # 测试服务端入口
    │   └── fakeAgents.ts                   # 确定性假 agent
    └── tests/
        ├── agentic-chat.test.ts            # TS HttpAgent → C# 服务端（按 SSE + protobuf）
        ├── backend-tool.test.ts
        ├── frontend-tools.test.ts
        ├── human-in-the-loop.test.ts
        ├── mixed-tool-invocation.test.ts
        ├── parallel-tool-calls.test.ts
        ├── protobuf-parity.test.ts
        └── state-events.test.ts            # 按 SSE + protobuf
```

### 第 1 部分：TS 客户端 → C# 服务端（Vitest）

这些测试使用 TS `HttpAgent`（来自 `@ag-ui/client`）针对真实 C# 服务端，由 LLMock 提供上游 LLM 响应。

```typescript
// tests/agentic-chat.test.ts
import { describe, it, expect, beforeAll, afterAll } from "vitest";
import {
```