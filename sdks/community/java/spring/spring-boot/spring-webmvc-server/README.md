# ag-ui-spring · spring-webmvc-server

[AG-UI 协议](https://docs.ag-ui.com)的 Spring **WebMVC**（Servlet）服务端集成。

它通过 `SseEmitter` 将 `Agent` 暴露为 `text/event-stream` 端点，并提供协议所需的基于 Jackson 的 `Serializer`。它是 [`ag-ui-spring-webflux-server`](../spring-webflux-server) 的 Servlet 对应版本——如果你的应用基于 Servlet（`spring-boot-starter-web`）而非响应式，请选择此模块。两者都与 `HttpAgent` 客户端在传输层兼容。

## 内容概览

| 类型 | 用途 |
|------|---------|
| [`AgUiController`](src/main/java/com/agui/community/spring/webmvc/AgUiController.java) | WebMVC `@RestController`，接受 `RunAgentInput` POST 请求，并通过 `SseEmitter` 以 Server-Sent Events 流式发送 Agent 事件。将 `/agent/{id}` 路由到 `AgentRegistry` 中的 Agent（基础路径上的单 Agent 别名；未知 id → `404`）。基础路径默认为 `/agent`（可通过 `ag-ui.server.path` 覆盖）。 |
| [`AgUiServerAutoConfiguration`](src/main/java/com/agui/community/spring/webmvc/AgUiServerAutoConfiguration.java) | Spring Boot 自动配置（仅在 **Servlet** Web 应用中生效）：提供 `Serializer`、默认的 `AgentRegistry`（以所有 `Agent` bean 的 **bean 名称**为键）以及控制器。 |

基于 Jackson 的 `Serializer`（`JacksonSerializer`）和 `AgentNotFoundException` 来自 [`spring-server-core`](../spring-server-core)，与 WebFlux 服务端共享。

## 用法

定义一个 `Agent` bean，端点即自动配置：

```java
@Configuration
class AgUiConfig {
    @Bean
    Agent agent() {
        return input -> subscriber -> { /* emit events */ };
    }
}
```

向 `/agent` 发送带 JSON `RunAgentInput` 的 `POST` 请求，即可通过 SSE 流式接收 Agent 事件。格式错误的输入返回 `400 Bad Request`；运行失败会以终止性 `RUN_ERROR` 事件的形式在流中呈现。

### 多个 Agent

定义多个 `Agent` bean，每个可通过 `/agent/{beanName}` 访问，未知 id 返回 `404`。只有一个 Agent 时，裸 `/agent` 路径仍然可用（别名）。如需使用 bean 名称以外的 id，请定义自己的 `AgentRegistry` bean——它会覆盖默认值：

```java
@Bean
AgentRegistry agents(Agent weather, Agent support) {
    return AgentRegistry.of(Map.of("weather", weather, "support", support));
}
```

### 手动装配（不使用自动配置）

```java
Serializer serializer = new JacksonSerializer(objectMapper);
AgUiController controller = new AgUiController(AgentRegistry.of(Map.of("chat", agent)), serializer);
```

## 依赖

```xml
<dependency>
    <groupId>com.ag-ui.community</groupId>
    <artifactId>ag-ui-spring-webmvc-server</artifactId>
    <version>4.1.0</version>
</dependency>
```

> 此模块独立版本化，跟随其目标 **Spring Boot 3.4.x** 系列。项目概览见[根 README](../../README.md)。