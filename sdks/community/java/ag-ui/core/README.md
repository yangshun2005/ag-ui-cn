# ag-ui · core

[AG-UI 协议](https://docs.ag-ui.com)的核心类型与协议原语。

本模块是库的基础。它定义协议的数据模型和 `Agent` 抽象，且**无第三方运行时依赖**——流式传输使用 JDK 的 `java.util.concurrent.Flow.Publisher` 表达，序列化交由可插拔的 `Serializer` 处理。

## 内容概览

| 包 | 内容 |
|---------|----------|
| [`agent`](src/main/java/com/agui/community/core/agent) | `Agent` 函数式接口，以及描述单次运行的 `RunAgentInput` 和 `Context`。 |
| [`event`](src/main/java/com/agui/community/core/event) | `Event` 模型——一个 `sealed interface`，每个事件变体对应一个 record，由 `EventType` 区分。 |
| [`message`](src/main/java/com/agui/community/core/message) | 对话消息：`UserMessage`、`AssistantMessage`、`SystemMessage`、`DeveloperMessage`、`ToolMessage`，以及 `Role`、`ToolCall` 和 `FunctionCall`。 |
| [`tool`](src/main/java/com/agui/community/core/tool) | `Tool` 和 `ToolParameters`，描述智能体可用的工具。 |
| [`serialization`](src/main/java/com/agui/community/core/serialization) | `Serializer` 接口（及 `SerializationException`）——JSON 绑定的接缝。核心不提供具体实现。 |

## 事件模型

`Event` 是一个 [sealed interface](src/main/java/com/agui/community/core/event/Event.java)；每个事件都是固定集合中的一个 record，因此可以用 `switch` 穷尽处理：

```java
String describe(Event event) {
    return switch (event) {
        case RunStartedEvent e        -> "run started";
        case TextMessageContentEvent e -> "text: " + e.delta();
        case ToolCallStartEvent e      -> "tool call: " + e.toolCallName();
        case RunFinishedEvent e        -> "run finished";
        default                        -> event.type().value();
    };
}
```

每个事件都带有通用的 `type()`、`timestamp()` 和 `rawEvent()` 字段。完整变体列表（生命周期、文本消息、工具调用、推理、状态、活动，以及 `RAW` / `CUSTOM` / `META_EVENT` 特殊类型）见 [`EventType`](src/main/java/com/agui/community/core/event/EventType.java)。

## `Agent` 抽象

```java
@FunctionalInterface
public interface Agent {
    Flow.Publisher<Event> run(RunAgentInput input);
}
```

每次订阅代表一次运行：发布者按顺序发出该运行的事件后完成，失败时发出 `onError`。

## 序列化

`core` 不绑定任何 JSON 库。实现 `Serializer` 即可接入 Jackson、Gson 或其他库；客户端和服务端模块接受 `Serializer`，使选择保留在应用边缘。

## 依赖

```xml
<dependency>
    <groupId>com.ag-ui.community</groupId>
    <artifactId>java-core</artifactId>
    <version>0.2.0</version>
</dependency>
```

项目概览见[根 README](../README.md)。