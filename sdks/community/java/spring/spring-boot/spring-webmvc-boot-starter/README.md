# ag-ui-spring · spring-webmvc-boot-starter

Spring Boot starter，通过 Servlet（**WebMVC**）HTTP Server-Sent Events 端点暴露 AG-UI `Agent`。

只需添加这一个依赖并定义一个 `Agent` bean —— 自动配置（来自 [`ag-ui-spring-webmvc-server`](../spring-webmvc-server)）会围绕它装配一个 `Serializer` 和基于 `SseEmitter` 的 `/agent` 端点。响应式（WebFlux）应用请改用 [`ag-ui-spring-webflux-boot-starter`](../spring-webflux-boot-starter)。

## 用法

```xml
<dependency>
    <groupId>com.ag-ui.community</groupId>
    <artifactId>ag-ui-spring-webmvc-boot-starter</artifactId>
    <version>4.1.0</version>
</dependency>
```

```java
@Bean
Agent agent() {
    return input -> subscriber -> { /* emit events */ };
}
```

现在向 `POST /agent` 发送 JSON `RunAgentInput`，即以 SSE 流式返回该 Agent 的事件。多个 `Agent` bean 分别可通过 `/agent/{beanName}` 访问；可用 `ag-ui.server.path` 覆盖基础路径。

本 starter 的版本跟随 **Spring Boot 4.1.x** 系列。项目概览见[根 README](../../README.md)。