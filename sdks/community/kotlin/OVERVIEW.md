# AG-UI Kotlin SDK 概述

AG-UI Kotlin SDK 是一个 Kotlin Multiplatform 客户端库，用于连接实现了 [Agent User Interaction Protocol (AG-UI)](https://docs.ag-ui.com/) 的 AI 智能体。该库提供传输机制、状态管理和工具集成，支持 Kotlin 应用与 AI 智能体在 Android、iOS 和 JVM 平台上的通信。

## 📚 完整文档

**[📖 完整 SDK 文档](../docs/sdk/kotlin/)**

完整文档详细涵盖以下内容：

- **[快速开始](../docs/sdk/kotlin/overview.mdx)** - 安装、架构和快速入门指南
- **[客户端 API](../docs/sdk/kotlin/client/)** - AgUiAgent、StatefulAgUiAgent、HttpAgent 和便捷构建器
- **[核心类型](../docs/sdk/kotlin/core/)** - 协议消息、事件、状态管理和序列化
- **[工具框架](../docs/sdk/kotlin/tools/)** - 可扩展的工具执行系统，含注册表和执行器

## 架构概要

AG-UI Kotlin SDK 遵循 TypeScript SDK 的设计模式，同时利用 Kotlin 的多平台能力和基于协程的并发模型：

- **kotlin-core**：协议类型、事件和消息定义
- **kotlin-client**：HTTP 传输、状态管理和高层智能体 API
- **kotlin-tools**：工具执行框架，含注册表和熔断器

SDK 在概念上与 TypeScript 实现保持一致，同时提供原生 Kotlin 惯用法，如密封类、挂起函数和用于流式响应的 Kotlin Flow。

## 生命周期订阅者与角色保真

- **AgentSubscriber Hook** – 智能体现提供订阅 API，应用可在内置处理器执行前观察运行初始化、逐事件投递和状态变更。这样无需 Fork 流水线即可实现分析、追踪或自定义持久化等横切关注点。
- **角色感知的文本流式传输** – 文本消息事件在分块转换和状态应用过程中保留其声明的角色（developer、system、assistant、user），确保下游 UI 状态与协议载荷完全一致。