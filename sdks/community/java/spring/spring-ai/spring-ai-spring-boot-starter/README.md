# ag-ui-spring · spring-ai-spring-boot-starter

这个 Spring Boot starter 通过 SSE 端点将 Spring AI 模型暴露为 AG-UI agent，**无需编写任何应用代码**。

它打包了 [`ag-ui-spring-ai`](../spring-ai) 和 [`ag-ui-spring-webflux-server`](../../spring-boot/spring-webflux-server)，并添加自动配置：从自动配置的 [`ChatClient.Builder`](https://docs.spring.io/spring-ai/reference/api/chatclient.html)（只要 classpath 上有聊天模型，Spring AI 就会提供）注册一个 `SpringAiAgent`，再由服务端自动配置将其暴露。

## 用法

```xml
<dependency>
    <groupId>com.ag-ui.community</groupId>
    <artifactId>ag-ui-spring-ai-spring-boot-starter</artifactId>
    <version>2.0.0</version>
</dependency>
```

classpath 上有 Spring AI 模型（如 `spring-ai-starter-model-openai`）时，Spring AI 会自动配置一个 `ChatClient.Builder` bean；本 starter 用它构建 `ChatClient`，转为 AG-UI `Agent`，并在 `/agent` 提供服务。无需定义任何 `@Bean`。

## 装配方式

| 条件 | 结果 |
|-----------|--------|
| 存在 `ChatClient.Builder` bean 且未定义 `Agent` bean | 注册一个 `SpringAiAgent` `Agent` bean |
| 存在 `Agent` bean（本 starter 的或你自己的） | 配置 `/agent` SSE 端点 |

通过属性启用 AG-UI **共享状态**（`update_state` 工具和状态事件）：

```properties
ag-ui.spring-ai.share-state=true
# 可选：发出 STATE_DELTA（RFC 6902 JSON Patch）而非完整的 STATE_SNAPSHOT
ag-ui.spring-ai.state-updates=DELTA
```

定义自己的 `Agent` bean 可退出自动配置的适配器，同时保留端点。这也是自定义 `ChatClient`（advisors、memory、默认提示词、注册工具）或状态行为的方式：

```java
@Bean
Agent agent(ChatClient.Builder builder) {
    return SpringAiAgent.builder(builder.defaultSystem("…").build())
            .shareState(true)
            .build();
}
```

本 starter 基于 **Spring AI 2.x** 版本线。Servlet（WebMVC）版本为 [`ag-ui-spring-ai-webmvc-boot-starter`](../spring-ai-webmvc-boot-starter)。项目概览见[根 README](../../README.md)。