# ag-ui-spring

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](https://www.apache.org/licenses/LICENSE-2.0)
[![Java](https://img.shields.io/badge/Java-17%2B-orange.svg)](https://adoptium.net/)

[**AG-UI 协议**](https://docs.ag-ui.com) 的 Spring 集成，构建于与框架无关的 `ag-ui` Java 库之上。

本仓库独立维护，使核心 `ag-ui` 模块保持轻依赖；Spring（及 Reactor）依赖树仅存在于此处。

## 目录结构

集成拆分为两个独立发布的**子 reactor**，各对应一条框架线。顶层有一个轻量聚合器（`pom.xml`），一次性构建并测试两者（`mvn verify`），但自身不发布。

- [`spring-boot/`](spring-boot) — **Spring Boot 服务端**线（`ag-ui-spring-boot-parent`，跟踪 Spring Boot `4.1.x`）。
- [`spring-ai/`](spring-ai) — **Spring AI** 线（`ag-ui-spring-ai-parent`，跟踪 Spring AI `2.x`）；其 Boot starter 依赖服务端线。

## 模块

| 模块 | 构建产物 | 版本线 | 说明 |
|--------|----------|--------------|-------------|
| [`spring-boot/spring-server-core`](spring-boot/spring-server-core) | `ag-ui-spring-server-core` | 跟踪 **Spring Boot**（`4.1.x`） | 两个服务端共享的框架无关代码：基于 Jackson 的 `Serializer`（针对 AG-UI 密封层级配置）和 `AgentNotFoundException`。无 Spring 依赖。 |
| [`spring-boot/spring-webflux-server`](spring-boot/spring-webflux-server) | `ag-ui-spring-webflux-server` | 跟踪 **Spring Boot**（`4.1.x`） | 响应式 **Spring WebFlux** 端点，将 `Agent` 的事件以 Server-Sent Events 流式输出，并提供 Spring Boot 自动配置。使用 `spring-server-core` 的共享序列化器。 |
| [`spring-boot/spring-webmvc-server`](spring-boot/spring-webmvc-server) | `ag-ui-spring-webmvc-server` | 跟踪 **Spring Boot**（`4.1.x`） | Servlet（**Spring WebMVC**）等价实现，通过 `SseEmitter` 流式输出 `Agent` 的事件。路由和共享序列化器相同——若应用基于 Servlet 而非响应式，选此模块。 |
| [`spring-boot/spring-webflux-boot-starter`](spring-boot/spring-webflux-boot-starter) | `ag-ui-spring-webflux-boot-starter` | 跟踪 **Spring Boot**（`4.1.x`） | 基于 `spring-webflux-server` 的开箱即用 starter：引入后定义一个 `Agent` bean 即可获得可用的响应式 `/agent` 端点。 |
| [`spring-boot/spring-webmvc-boot-starter`](spring-boot/spring-webmvc-boot-starter) | `ag-ui-spring-webmvc-boot-starter` | 跟踪 **Spring Boot**（`4.1.x`） | 基于 `spring-webmvc-server` 的开箱即用 starter：WebFlux starter 的 Servlet 等价实现。 |
| [`spring-ai/spring-ai`](spring-ai/spring-ai) | `ag-ui-spring-ai` | 跟踪 **Spring AI**（`2.x`） | 将 Spring AI `ChatClient` 适配为 AG-UI `Agent`，把其流式响应转换为 AG-UI 事件生命周期。 |
| [`spring-ai/spring-ai-spring-boot-starter`](spring-ai/spring-ai-spring-boot-starter) | `ag-ui-spring-ai-spring-boot-starter` | 跟踪 **Spring AI**（`2.x`） | 零代码 starter（**响应式 / WebFlux**）：从自动配置的 `ChatClient.Builder` 自动注册 `SpringAiAgent`，并在 `/agent` 暴露。 |
| [`spring-ai/spring-ai-webmvc-boot-starter`](spring-ai/spring-ai-webmvc-boot-starter) | `ag-ui-spring-ai-webmvc-boot-starter` | 跟踪 **Spring AI**（`2.x`） | **Servlet / WebMVC** 等价零代码 Spring AI starter：同样的自动注册，通过基于 `SseEmitter` 的 `/agent` 提供服务。 |

## 版本管理

两条线**独立版本化并独立发布**，因为各自跟踪不同框架的兼容性：

- **spring-boot** 线（`ag-ui-spring-boot-parent`）按其目标 **Spring Boot** 线版本化（如 `4.1.0`）；其中所有模块共享该版本；
- **spring-ai** 线（`ag-ui-spring-ai-parent`）按其目标 **Spring AI** 线版本化（如 `2.0.0`）；其中所有模块共享该版本。

每个子 reactor 自包含（拥有自己的框架 BOM 和构建/发布配置），因此两条线可按不同节奏升级和发布。按需混用版本。

### 发布

两条线均通过 monorepo 的 [`publish-release.yml`](../../../../.github/workflows/publish-release.yml) 工作流发布到 Maven Central。该工作流检测各子 reactor `pom.xml` 中的版本变更（在 `scripts/release/release.config.json` 中注册为 `integration-spring-boot-java` 和 `integration-spring-ai-java` 作用域），并使用 `release` Maven profile 部署该 reactor。该 profile 会扁平化各模块的 POM（内联 reactor 父 POM），因此两条线以自包含产物发布，聚合器无需被消费者解析。

**spring-ai** 线的 Boot starter 依赖 **spring-boot** 服务端模块，因此服务端线先构建并部署（工作流保持此顺序）。`ag-ui` 产物（`java-core`、`:java-server`）必须已在 Central 上——Central Portal 拒绝 SNAPSHOT 依赖。

## 要求

- **Java 17+**
- **Spring Boot 4.1.x** / **Spring AI 2.x**
- `ag-ui` 产物（`com.ag-ui.community:java-core`、`:java-server`）——从 Maven Central 解析（当前为 `0.1.1`；见各子 reactor POM 中的 `ag-ui.version`）。

## 快速开始

选择匹配你现有环境的 starter：

**暴露自己的 agent**——添加 `ag-ui-spring-webflux-boot-starter`（响应式）或 `ag-ui-spring-webmvc-boot-starter`（Servlet），并定义一个 bean：

```java
@Bean
Agent agent() {
    return input -> subscriber -> { /* emit events */ };
}
```

**零代码暴露 Spring AI 模型**——添加 `ag-ui-spring-ai-spring-boot-starter`（响应式 / WebFlux）或 `ag-ui-spring-ai-webmvc-boot-starter`（Servlet / WebMVC）以及一个 Spring AI 模型（如 `spring-ai-starter-model-openai`）。`SpringAiAgent` 会从自动配置的 `ChatClient.Builder` 自动注册并自动提供服务。

两种方式下端点均为 `/agent`（可用 `ag-ui.server.path` 覆盖）；将 `HttpAgent` 客户端指向它即可。

## 构建

```bash
mvn clean install
```

> 需先将 `ag-ui` 产物放入本地仓库（或已配置的仓库）。在 `ag-ui` 发布到 Maven Central 之前，请在 `ag-ui` 项目中运行 `mvn install`。

## 贡献

参见组织的贡献指南和行为准则。

## 许可证

基于 [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0) 授权。