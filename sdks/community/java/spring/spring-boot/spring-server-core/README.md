# ag-ui-spring · spring-server-core

AG-UI Spring 服务端模块（[`spring-webflux-server`](../spring-webflux-server) 和 [`spring-webmvc-server`](../spring-webmvc-server)）共享的框架无关代码。它**不依赖 Spring**——只依赖 Jackson 和 `ag-ui` 的 `server` 模块。

## 内容概览

| 类型 | 用途 |
|------|------|
| [`JacksonSerializer`](src/main/java/com/agui/community/spring/server/core/JacksonSerializer.java) | 基于 Jackson 的 `Serializer`，配置为多态处理密封的 `Event`（按 `type`）和 `Message`（按 `role`）层级，并将 `Role`/`EventType` 绑定到其传输值。 |
| [`AgentNotFoundException`](src/main/java/com/agui/community/spring/server/core/AgentNotFoundException.java) | 请求指向未知 agent id 时抛出；各服务端控制器将其映射为 `404`。 |
| [`jackson`](src/main/java/com/agui/community/spring/server/core/jackson) mix-in | `Event`/`Message` 的 Jackson mix-in，使 `ag-ui` 的 `core` 模块不含 Jackson 注解。 |

各服务端模块在此基础上添加自己的传输层控制器和自动配置。通常应依赖某个服务端模块（或其 starter），而非直接依赖 `spring-server-core`。

## 依赖

```xml
<dependency>
    <groupId>com.ag-ui.community</groupId>
    <artifactId>ag-ui-spring-server-core</artifactId>
    <version>4.1.0</version>
</dependency>
```

> 版本跟随 **Spring Boot 3.4.x** 线（共用该线的 Jackson 版本）。项目概览见[根 README](../../README.md)。