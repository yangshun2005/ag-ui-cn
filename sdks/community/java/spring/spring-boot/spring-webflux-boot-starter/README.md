# ag-ui-spring · spring-webflux-boot-starter

Spring Boot starter，通过响应式（**WebFlux**）HTTP Server-Sent Events 端点暴露 AG-UI `Agent`。

只需添加这一个依赖并定义一个 `Agent` bean —— 自动配置（来自 [`ag-ui-spring-webflux-server`](../spring-webflux-server)）会围绕它装配 `Serializer` 和 `/agent` 端点。若使用 Servlet（WebMVC）应用，请改用 [`ag-ui-spring-webmvc-boot-starter`](../spring-webmvc-boot-starter)。

## 用法

```xml
<dependency>
    <groupId>com.ag-ui.community</groupId>
    <artifactId>ag-ui-spring-webflux-boot-starter</artifactId>
    <version>4.1.0</version>
</dependency>
```

```java
@Bean
Agent agent() {
    return input -> subscriber -> { /* emit events */ };
}
```

现在向 `POST /agent` 发送 JSON `RunAgentInput` 即可将 Agent 的事件以 SSE 流式返回。可通过 `ag-ui.server.path` 覆盖路径。

> 当你自带 `Agent` 时使用本 starter。若想零代码暴露 **Spring AI** 模型，请改用 [`ag-ui-spring-ai-spring-boot-starter`](../../spring-ai/spring-ai-spring-boot-starter)。

本 starter 的版本跟随 **Spring Boot 4.1.x** 系列。项目概览见[根 README](../../README.md)。