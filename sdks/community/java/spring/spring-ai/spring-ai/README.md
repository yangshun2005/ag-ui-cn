# ag-ui-spring · spring-ai


[AG-UI 协议](https://docs.ag-ui.com)的 Spring AI 集成。

它将 Spring AI 的
[`ChatClient`](https://docs.spring.io/spring-ai/reference/api/chatclient.html) 适配为
AG-UI 的
`Agent`，
因此任何 Spring AI 模型（OpenAI、Ollama 等）都能驱动 AG-UI 前端。
`ChatClient` 是高层入口（advisors、记忆、默认提示词、已注册工具），构建于任意 `ChatModel` 之上。

## 内容概览

| 类型 | 用途 |
|------|---------|
| [`SpringAiAgent`](src/main/java/com/agui/community/spring/ai/SpringAiAgent.java) | 包装 `ChatClient`。将 `RunAgentInput` 对话映射为 Spring AI 提示词，流式返回响应（`Flux<ChatResponse>`），并发出 AG-UI 事件生命周期。 |

## 事件映射

一次运行产生以下 AG-UI 事件：

```
RUN_STARTED
  REASONING_START                     (for <think>...</think> content)
    REASONING_MESSAGE_START
      REASONING_MESSAGE_CONTENT*
    REASONING_MESSAGE_END
  REASONING_END
  TEXT_MESSAGE_START                  (when the model produces text)
    TEXT_MESSAGE_CONTENT*
  TEXT_MESSAGE_END
  TOOL_CALL_START                     (per tool call the model requests)
    TOOL_CALL_ARGS*
  TOOL_CALL_END
RUN_FINISHED
```

**推理**通过流式文本中的内联 `<think>...</think>` 标签识别（这是推理模型通用的、与提供商无关的约定，也是 Spring AI 提供 `ThinkingTagCleaner` 的原因）。标签内的内容作为推理子流发出；标签外的内容作为文本发出。跨流式分块拆分的标签会被重新拼接。不使用 think 标签的模型不受影响——所有内容都作为文本发出。

运行对话历史中返回的推理——AG-UI 的 `ReasoningMessage`（角色为 `reasoning`，在 ag-ui `0.1.1` 中引入）——会以包裹相同 `<think>...</think>` 标签的 assistant 消息形式回放给模型，使推理轮次对称往返。空的推理消息（例如仅携带提供商特定 `encryptedValue` 的消息）会被丢弃。

Agent 拥有**工具执行循环**，区分两类工具：

- **客户端工具**——即 `RunAgentInput` 上的工具。向模型声明，并以 `TOOL_CALL_START/ARGS/END` 事件呈现给前端执行；Agent 不执行它们。
- **后端工具**——通过 `SpringAiAgent.builder(client).tools(...)` 注册，**或直接注册在 `ChatClient` 上**（通过 `ChatClient.builder(model).defaultTools(...)`）。当模型调用其中之一时，Agent 发出 `TOOL_CALL_START/ARGS/END`，**执行它**（通过 Spring AI 的 `ToolCallingManager`），发出 **`TOOL_CALL_RESULT`**，然后用结果重新提示模型——循环直到模型停止调用后端工具。如果一轮中混合了后端和客户端调用，则发出后端结果并停止运行，由前端处理客户端调用。当 Agent 驱动此循环时（只要声明了任何客户端工具、后端工具、状态工具或中断工具），它会抑制 `ChatClient` 自身的自动工具执行，因此它自己执行 `ChatClient` 的 `defaultTools(...)` 回调，而不是将它们泄漏到前端。

**工具调用映射**：流式参数分块按工具调用 id 关联；第一个携带 id（或名称）的分块开启调用，后续分块追加参数增量。省略 id 的提供商（如 Ollama）会获得一个合成的 id。任何打开的文本消息在工具调用开始前关闭。

**对话历史**：下一轮时前端会回传先前的 assistant 消息（含其工具调用）和 `tool` 结果消息。Agent 将它们重建为 Spring AI 的 `AssistantMessage`（携带 `ToolCall`）和 `ToolResponseMessage`——这样模型能看到它请求的客户端工具已经运行并作出响应，而不是每轮重复发出相同的调用。

**生成式 UI**：每个 `TOOL_CALL_RESULT` 作为**独立的**对话消息发出——使用新的 `messageId`（区别于发起调用的 assistant 轮次），角色为 `role: tool`。CopilotKit 等前端按 id 索引消息，因此复用 assistant 消息 id 会用结果覆盖 assistant 工具调用——以及由它渲染的任何生成式 UI。将结果保留为独立消息，可让渲染的组件与后续文本并存。

**状态**（可选启用）同步 AG-UI [共享状态](https://docs.ag-ui.com/concepts/state)。启用后 Agent 会：

- 在运行开始时发出初始 `STATE_SNAPSHOT`，回显 `RunAgentInput.state`；
- 将当前状态注入提示词，并声明一个 `update_state` 工具（接收完整的新状态）；
- 拦截模型对该工具的调用并发出新状态——该调用**不会**作为 `TOOL_CALL_*` 事件呈现，也不会在服务端执行。

状态变更默认作为完整的 `STATE_SNAPSHOT` 发出，或配置 `StateUpdates.DELTA` 时作为 `STATE_DELTA`（RFC 6902 JSON Patch，与运行的输入状态做差异比较）发出。模型始终提供完整的新状态；Agent 计算增量，因此无论模型生成 JSON Patch 的能力如何都能保持可靠。

如果模型出错，会发出终止性的 `RUN_ERROR` 事件，而不是传播失败——与协议的带内错误处理一致。

## 用法

在任意 `ChatModel` 上构建 `ChatClient`——按需添加 advisors、聊天记忆、默认提示词和已注册工具：

```java
ChatClient chatClient = ChatClient.builder(chatModel)
        .defaultSystem("You are a helpful assistant.")
        .defaultTools(myTools)
        .build();
Agent agent = new SpringAiAgent(chatClient);
```

快速上手时，`ChatClient.create(chatModel)` 会构建一个带默认配置的客户端。

要启用 AG-UI 共享状态，使用 builder：

```java
Agent agent = SpringAiAgent.builder(chatClient)
        .shareState(true)
        .stateUpdates(SpringAiAgent.StateUpdates.DELTA)   // optional; default is SNAPSHOT
        .build();
```

结合 `ag-ui-spring-webflux-server`（响应式）或
`ag-ui-spring-webmvc-server`（Servlet）通过 HTTP 暴露。 
[`ag-ui-spring-ai-spring-boot-starter`](../spring-ai-spring-boot-starter)（WebFlux）
和 [`ag-ui-spring-ai-webmvc-boot-starter`](../spring-ai-webmvc-boot-starter)
（Servlet）会从自动配置的 `ChatClient.Builder` 自动完成此操作；要自定义客户端，自行定义 `Agent` bean（会覆盖自动配置的 bean）：

```java
@Bean
Agent agent(ChatClient.Builder builder) {
    return new SpringAiAgent(builder.defaultSystem("…").build());
}
```

## 范围

向模型声明**输入工具**，并将流式**文本**、
**推理**（`<think>` 标签）、**工具调用**以及——启用状态共享时——
**共享状态**（`STATE_SNAPSHOT` 或 `STATE_DELTA`）映射为 AG-UI 事件。

`MESSAGES_SNAPSHOT` 通过 `SpringAiAgent.builder(client).emitMessagesSnapshot(true)` 可选启用：
启用后 Agent 会在 `RUN_FINISHED` 之前发出本次运行的持久对话快照——即运行输入的消息加上本次运行产生的 assistant 消息（文本和工具调用）以及后端 `tool` 结果消息。它是附加性的（相同消息已通过流式事件传达），因此从流重建历史的前端不受影响；流式推理不包含在快照中。`TOOL_CALL_RESULT` 针对**后端**工具（由 Agent 执行）发出；**客户端**工具结果由运行该工具的前端产生。

## 依赖

```xml
<dependency>
    <groupId>com.ag-ui.community</groupId>
    <artifactId>ag-ui-spring-ai</artifactId>
    <version>2.0.0</version>
</dependency>
```

> 此模块独立版本化，并跟随其目标的 **Spring AI 2.x** 系列。项目概览见[根 README](../../README.md)。