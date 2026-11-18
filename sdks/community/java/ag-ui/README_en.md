# ag-ui

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](https://www.apache.org/licenses/LICENSE-2.0)
[![Java](https://img.shields.io/badge/Java-17%2B-orange.svg)](https://adoptium.net/)

一个模块化的 Java 库，用于 [**AG-UI 协议**](https://docs.ag-ui.com) —— 将 AI 智能体连接到面向用户应用的开放、基于事件的协议。

该库将协议建模为一系列类型化的**事件**（文本、工具调用、推理、状态增量、生命周期），由 [`Agent`](core/src/main/java/com/agui/community/core/agent/Agent.java) 响应 `RunAgentInput` 时发出。流式类型使用 JDK 内置的 `java.util.concurrent.Flow.Publisher`，因此核心**没有第三方运行时依赖**。

## 模块

| 模块 | 描述 |
|--------|-------------|
| [`core`](core) | 核心类型和协议原语 —— 消息、事件、智能体抽象以及可插拔的 `Serializer`。无外部运行时依赖。 |
| [`client`](client) | 客户端实现。`HttpAgent` 通过 HTTP 调用远程 AG-UI 端点，将 Server-Sent Events 响应解码为事件流。基于 JDK `HttpClient` 构建。 |
| [`server`](server) | 服务端支持，用于通过 AG-UI 协议暴露 `Agent`。 |

## 环境要求

- **Java 17+**
- **Maven 3.8+**

## 构建

```bash
mvn clean install
```

仅运行测试：

```bash
mvn test
```

## 使用

### 连接远程智能体（客户端）

`HttpAgent` 将 `RunAgentInput` POST 到 AG-UI 端点，并将 Server-Sent Events 响应暴露为 `Flow.Publisher<Event>`。你需要提供 `Serializer`（该库不绑定具体的 JSON 实现）。

```java
import com.agui.community.client.HttpAgent;
import com.agui.community.core.agent.Agent;
import com.agui.community.core.agent.RunAgentInput;
import com.agui.community.core.event.Event;

import java.net.URI;
import java.util.concurrent.Flow;

Serializer serializer = /* your Serializer implementation */;

Agent agent = new HttpAgent(
        URI.create("https://example.com/agent"),
        serializer);

RunAgentInput input = /* build the run input: messages, tools, context, state */;

agent.run(input).subscribe(new Flow.Subscriber<>() {
    @Override public void onSubscribe(Flow.Subscription s) { s.request(Long.MAX_VALUE); }
    @Override public void onNext(Event event)              { System.out.println(event); }
    @Override public void onError(Throwable t)             { t.printStackTrace(); }
    @Override public void onComplete()                     { System.out.println("done"); }
});
```

每次订阅都会触发一次新的运行：请求在订阅时发送，事件按顺序发出，流结束时 publisher 完成（失败时发出 `onError`）。

### 实现智能体

`Agent` 是函数式接口 —— 实现 `run` 来发出你自己的事件流：

```java
Agent agent = input -> {
    // produce a Flow.Publisher<Event> describing the run
};
```

## 作为依赖使用

构建产物发布在 `com.ag-ui.community` group id 下。添加你需要的模块：

```xml
<dependency>
    <groupId>com.ag-ui.community</groupId>
    <artifactId>java-client</artifactId>
    <version>0.2.0</version>
</dependency>
```

> `core` 会由 `client` 和 `server` 传递引入；如果只需要协议类型，可直接依赖它。

## 贡献

欢迎贡献！请先阅读[贡献指南](https://docs.ag-ui.com/development/contributing)。

## 许可证

基于 [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0) 授权。