# ag-ui · client

[AG-UI 协议](https://docs.ag-ui.com)的客户端实现。

本模块让你以 [`Agent`](../core/src/main/java/com/agui/community/core/agent/Agent.java) 的身份与远程 AG-UI 端点通信。它仅基于 JDK 的 `HttpClient` 和注入的 `Serializer` 构建，因此不依赖具体的 JSON 库。

## 模块内容

| 类型 | 用途 |
|------|------|
| [`HttpAgent`](src/main/java/com/agui/community/client/HttpAgent.java) | 一个 `Agent`，将 `RunAgentInput` 序列化为 JSON，POST 到远程端点，并将 Server-Sent Events 响应解码为 `Flow.Publisher<Event>`。 |
| [`SseEventParser`](src/main/java/com/agui/community/client/SseEventParser.java) | 将 Server-Sent Events 字节/行流解析为 AG-UI 事件。 |
| [`HttpAgentException`](src/main/java/com/agui/community/client/HttpAgentException.java) | 请求失败或返回错误状态时抛出（通过 `onError`）。 |

## 用法

```java
import com.agui.community.client.HttpAgent;
import com.agui.community.core.agent.Agent;
import com.agui.community.core.agent.RunAgentInput;
import com.agui.community.core.event.Event;

import java.net.URI;
import java.util.concurrent.Flow;

Serializer serializer = /* your Serializer implementation */;

Agent agent = new HttpAgent(URI.create("https://example.com/agent"), serializer);

RunAgentInput input = /* messages, tools, context, state */;

agent.run(input).subscribe(new Flow.Subscriber<>() {
    @Override public void onSubscribe(Flow.Subscription s) { s.request(Long.MAX_VALUE); }
    @Override public void onNext(Event event)              { System.out.println(event); }
    @Override public void onError(Throwable t)             { t.printStackTrace(); }
    @Override public void onComplete()                     { System.out.println("done"); }
});
```

每次订阅都会触发一次新的运行：订阅时发送请求，事件按顺序发出，流结束时 publisher 完成。

### 自定义传输

默认构造函数使用 `HttpClient.newHttpClient()`、缓存线程池（`Executors.newCachedThreadPool()`）和 5 分钟请求超时。消费 SSE 响应是长时间阻塞操作，因此使用缓存线程池，避免耗尽共享的有界线程池。使用完整构造函数可控制 `HttpClient`、消费响应流的 `Executor` 以及每次请求的超时：

```java
Agent agent = new HttpAgent(
        URI.create("https://example.com/agent"),
        serializer,
        myHttpClient,
        myExecutor,
        Duration.ofSeconds(30));
```

## 依赖

```xml
<dependency>
    <groupId>com.ag-ui.community</groupId>
    <artifactId>java-client</artifactId>
    <version>0.2.0</version>
</dependency>
```

会传递引入 [`core`](../core)。项目概览见[根 README](../README.md)。