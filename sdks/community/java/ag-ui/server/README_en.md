# ag-ui · server

为 [AG-UI 协议](https://docs.ag-ui.com) 提供服务器端支持。

本模块通过该协议暴露一个 [`Agent`](../core/src/main/java/com/agui/community/core/agent/Agent.java)：读取 `RunAgentInput`，运行 Agent，并将产生的 `Event` 以 Server-Sent Events 形式流式返回给客户端。它是 [`client`](../client) 模块 `HttpAgent` 的服务器端对应实现。

核心是**传输无关**的，不依赖任何 HTTP 框架。线路帧格式被隔离在 `EventEncoder` 之后，因此同一核心今天可以服务 Server-Sent Events，日后只需替换编码器即可支持其他协议（如 WebSocket）。附带一个基于 JDK 的 SSE 适配器，无需第三方依赖即可运行真实端点。

## 内容概览

### 传输无关核心（`com.agui.community.server`）

| 类型 | 用途 |
|------|---------|
| [`AgentRunHandler`](src/main/java/com/agui/community/server/AgentRunHandler.java) | 将请求体解析为 `RunAgentInput`，运行 `Agent`，并通过 `EventEncoder` 将事件帧化后转发到 `EventSink`。不感知 HTTP 或线路协议。 |
| [`AgentRegistry`](src/main/java/com/agui/community/server/AgentRegistry.java) | 从 agent id 到 `Agent` 的查找表，使传输层能将 `/agent/{id}` 路由到多个 agent 之一。`single()` 在恰好注册一个 agent 时暴露该唯一 agent。 |
| [`EventEncoder`](src/main/java/com/agui/community/server/EventEncoder.java) | 为特定协议帧化 `Event`——SSE 与未来传输（如 WebSocket）之间的唯一差异点。 |
| [`EventSink`](src/main/java/com/agui/community/server/EventSink.java) | 已编码帧的传输无关目标；适配器在其响应上实现它。 |
| [`SseEventEncoder`](src/main/java/com/agui/community/server/SseEventEncoder.java) | SSE 的 `EventEncoder`：使用注入的 `Serializer` 将 `Event` 编码为 `data:` 帧。 |
| [`EventRelaySubscriber`](src/main/java/com/agui/community/server/EventRelaySubscriber.java) | 一个 `Flow.Subscriber<Event>`，将编码帧写入 sink，每次请求一个事件以实现背压，并将故障以带内终止 `RUN_ERROR` 帧的形式呈现。 |
| [`OutputStreamEventSink`](src/main/java/com/agui/community/server/OutputStreamEventSink.java) | 基于阻塞 `OutputStream` 的 `EventSink`（每帧刷新）。 |

### JDK 传输（`com.agui.community.server.jdk`）

| 类型 | 用途 |
|------|---------|
| [`JdkAgentHttpHandler`](src/main/java/com/agui/community/server/jdk/JdkAgentHttpHandler.java) | 用于 JDK 内置 `com.sun.net.httpserver` 的 `HttpHandler`。将 `/agent/{id}` 路由到 `AgentRegistry` 中的 agent（基础路径上的单 agent 别名），以 `text/event-stream` 流式返回事件，未知 id 返回 `404`，格式错误输入返回 `400`，非 `POST` 返回 `405`。无第三方依赖。 |

## 用法

```java
import com.sun.net.httpserver.HttpServer;
import com.agui.community.core.agent.Agent;
import com.agui.community.server.jdk.JdkAgentHttpHandler;

import java.net.InetSocketAddress;

Agent agent = /* your Agent */;
Serializer serializer = /* your Serializer implementation */;

HttpServer server = HttpServer.create(new InetSocketAddress(8080), 0);
server.createContext("/agent", new JdkAgentHttpHandler(agent, serializer));
server.start();
```

将 [`HttpAgent`](../client) 客户端指向 `http://localhost:8080/agent` 即可驱动它。

### 多个 agent

传入 [`AgentRegistry`](src/main/java/com/agui/community/server/AgentRegistry.java)，通过路径中的 id（`/agent/{id}`）寻址多个 agent。未知 id 返回 `404`：

```java
AgentRegistry registry = AgentRegistry.of(Map.of(
        "weather", weatherAgent,
        "support", supportAgent));

server.createContext("/agent", new JdkAgentHttpHandler(registry, serializer));
// POST /agent/weather -> weatherAgent, POST /agent/support -> supportAgent
```

上面的单 agent 构造函数是单条目注册表的简写：它在基础路径（别名）和 `/agent/default` 上提供服务。

### 自带传输层

要与其他技术栈（servlet、WebFlux 等）集成，直接使用传输无关核心：构建 `AgentRunHandler`，在你的响应上实现 `EventSink`，然后 `parse` 请求体（将失败映射为 `400`）并 `run`：

```java
AgentRunHandler handler = new AgentRunHandler(agent, serializer);

RunAgentInput input = handler.parse(requestBody);   // throws on bad input -> 400
handler.run(input, myEventSink).join();             // completes when the stream ends
```

要服务不同的线路协议（例如 WebSocket 消息帧而非 SSE `data:` 帧），提供你自己的 `EventEncoder`——核心其余部分不变：

```java
EventEncoder encoder = event -> serializer.serialize(event); // bare JSON per message
AgentRunHandler handler = new AgentRunHandler(agent, serializer, encoder);
```

> Spring（WebFlux / Spring Boot starter）适配器和 Spring AI 集成位于单独的 `ag-ui-spring` 仓库，以保持本模块无依赖。

## 依赖

```xml
<dependency>
    <groupId>com.ag-ui.community</groupId>
    <artifactId>java-server</artifactId>
    <version>0.2.0</version>
</dependency>
```

传递引入 [`core`](../core)。项目概览见[根 README](../README.md)。