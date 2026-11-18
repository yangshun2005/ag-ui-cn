# ag-ui-spring · spring-webflux-server

[AG-UI 协议](https://docs.ag-ui.com)的 Spring WebFlux 服务端集成。

它将 `Agent` 暴露为响应式 `text/event-stream` 端点，并提供协议所需的基于 Jackson 的 `Serializer`。它与 `HttpAgent` 客户端线上兼容，是 `ag-ui` `server` 模块中 JDK handler 的 Spring 对应实现。

> 基于 Servlet 的应用？请改用 [`ag-ui-spring-webmvc-server`](../spring-webmvc-server)——API 相同，底层为 `SseEmitter`。

## 内容概览

| 类型 | 用途 |
|------|------|
| [`AgUiController`](src/main/java/com/agui/community/spring/server/AgUiController.java) | WebFlux `@RestController`，接受 `RunAgentInput` POST，并以 Server-Sent Events 流式推送 Agent 事件。将 `/agent/{id}` 路由到 `AgentRegistry` 中的 Agent（基础路径上的单 Agent 别名；未知 id → `404`）。基础路径默认为 `/agent`（可通过 `ag-ui.server.path` 覆盖）。 |
| [`AgUiServerAutoConfiguration`](src/main/java/com/agui/community/spring/server/AgUiServerAutoConfiguration.java) | Spring Boot 自动配置：提供 `Serializer`（复用应用的 `ObjectMapper`）、以所有 `Agent` bean 的 **bean 名称**为键的默认 `AgentRegistry`，以及控制器。 |

基于 Jackson 的 `Serializer`（`JacksonSerializer`）和 `AgentNotFoundException` 来自 [`spring-server-core`](../spring-server-core)，与 WebMVC 服务端共用。

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

向 `/agent` POST 一个 JSON `RunAgentInput`，即可将 Agent 事件以 SSE 流式推送。格式错误的输入返回 `400 Bad Request`；运行失败以终止性 `RUN_ERROR` 事件在带内呈现。

### 多 Agent

定义多个 `Agent` bean，每个可通过 `/agent/{beanName}` 访问，未知 id 返回 `404`：

```java
@Bean Agent weather() { … }   // POST /agent/weather
@Bean Agent support() { … }   // POST /agent/support
```

只有一个 Agent 时，裸 `/agent` 路径仍可用（别名）。如需使用 bean 名称以外的 id，定义自己的 `AgentRegistry` bean——它会覆盖默认值：

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

## 说明

- `JacksonSerializer` 在本库的客户端与服务端之间可正确往返。与参考 TypeScript/Python 实现完全逐字节一致（可选字段的确切命名等）可能还需进一步映射配置。

## 依赖

```xml
<dependency>
    <groupId>com.ag-ui.community</groupId>
    <artifactId>ag-ui-spring-webflux-server</artifactId>
    <version>4.1.0</version>
</dependency>
```

> 本模块独立版本化，跟随其目标 **Spring Boot 3.4.x** 系列。项目概览见[根 README](../../README.md)。