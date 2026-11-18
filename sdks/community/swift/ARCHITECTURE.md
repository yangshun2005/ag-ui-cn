# AGUISwift 架构

AGUISwift 采用清晰的模块化架构，遵循严格的领域驱动设计（DDD）分层原则：

```
┌─────────────────────────────────────────────────────────────┐
│                    你的应用                                   │
│              (iOS, macOS, tvOS, watchOS)                      │
├─────────────────────────────────────────────────────────────┤
│                  AGUIAgentSDK                                │
│  ┌─────────────┐  ┌───────────────────┐  ┌──────────────┐  │
│  │ AgUiAgent  │  │StatefulAgUiAgent  │  │   Builders   │  │
│  │            │  │                   │  │              │  │
│  └─────────────┘  └───────────────────┘  └──────────────┘  │
├─────────────────────────────────────────────────────────────┤
│        AGUIClient              │      AGUITools              │
│  ┌────────────┐  ┌─────────────┐ │ ┌──────────────────────┐ │
│  │ HttpAgent  │  │AbstractAgent│ │ │    ToolRegistry      │ │
│  ├────────────┤  ├─────────────┤ │ ├──────────────────────┤ │
│  │EventStream │  │EventDecoder │ │ │   ToolExecutor      │ │
│  │ SseParser  │  │   Manager   │ │ │ToolExecutionManager │ │
│  └────────────┘  └─────────────┘ │ └──────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│                    AGUICore                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              基础设施层                                │  │
│  │  ┌──────────────────┐  ┌─────────────────────────┐  │  │
│  │  │  序列化           │  │      适配器              │  │  │
│  │  │ ┌──────────────┐ │  │ ┌─────────────────────┐ │  │  │
│  │  │ │Serializable  │ │  │ │ValueObjectAdapter   │ │  │  │
│  │  │ │ThreadId      │ │  │ │                     │ │  │  │
│  │  │ │Serializable  │ │  │ │                     │ │  │  │
│  │  │ │RunId         │ │  │ │                     │ │  │  │
│  │  │ │Serializable  │ │  │ │                     │ │  │  │
│  │  │ │EventTimestamp│ │  │ │                     │ │  │  │
│  │  │ └──────────────┘ │  │ └─────────────────────┘ │  │  │
│  │  └──────────────────┘  └─────────────────────────┘  │  │
│  ├──────────────────────────────────────────────────────┤  │
│  │              协议层                                   │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │  │
│  │  │ AGUIEvent    │  │ EventType    │  │EventDecoder│  │  │
│  │  │             │  │              │  │           │  │  │
│  │  │RunStarted   │  │              │  │           │  │  │
│  │  │RunFinished  │  │              │  │           │  │  │
│  │  └──────────────┘  └──────────────┘  └───────────┘  │  │
│  ├──────────────────────────────────────────────────────┤  │
│  │              领域层                                   │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │  │
│  │  │ 值对象        │  │    事件       │  │  错误      │  │  │
│  │  │             │  │              │  │           │  │  │
│  │  │ ThreadId    │  │ RunStarted   │  │DomainError │  │  │
│  │  │ RunId       │  │ RunFinished  │  │           │  │  │
│  │  │EventTimestamp│ │              │  │           │  │  │
│  │  └──────────────┘  └──────────────┘  └───────────┘  │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## 模块概览

### AGUICore
协议类型、事件和消息定义，采用严格的 DDD 分层。

**领域层**（纯领域逻辑，无基础设施依赖）：
- **值对象**：`ThreadId`、`RunId`、`EventTimestamp` — 类型安全的领域值对象
- **事件**：领域事件定义（待实现）
- **错误**：`DomainError` — 领域校验错误

**协议层**（协议特定类型）：
- **AGUIEvent**：所有 AG-UI 协议事件的基础协议
- **EventType**：所有 AG-UI 协议事件类型的枚举
- **EventDecoder**：多态事件解码辅助工具
- **具体事件**：`RunStartedEvent`、`RunFinishedEvent`（后续会添加更多）

**基础设施层**（序列化与适配器）：
- **序列化**：`SerializableThreadId`、`SerializableRunId`、`SerializableEventTimestamp` — 用于 JSON 序列化的基础设施包装器
- **适配器**：`ValueObjectAdapter` — 在领域层与基础设施层之间转换

### AGUIAgentSDK
面向常见 Agent 交互模式的高层 API。
- **AgUiAgent**：无状态客户端，适用于无需持续上下文的场景
- **StatefulAgUiAgent**：有状态客户端，维护对话历史
- **Builders**：便捷的 Agent 配置构建器模式

### AGUIClient
底层客户端基础设施与传输实现。
- **HttpAgent**：底层 HTTP 传输实现
- **AbstractAgent**：自定义 Agent 实现的基类
- **SseParser**：用于流式响应的 Server-Sent Events 解析器
- **EventStreamManager**：事件流管理与处理

### AGUITools
用于扩展 Agent 能力的工具执行框架。
- **ToolExecutor**：实现自定义工具的协议
- **ToolRegistry**：工具注册与管理
- **ToolExecutionManager**：带熔断器模式的工具执行

## 设计原则

### 领域驱动设计（DDD）
- **领域层**：纯业务逻辑，无基础设施依赖
- **基础设施层**：处理序列化、网络和外部关注点
- **清晰边界**：领域层与基础设施层严格分离
- **值对象**：类型安全的领域概念（ThreadId、RunId、EventTimestamp）

### 分层架构
- **应用层**：高层 API（AGUIAgentSDK）
- **客户端层**：底层传输与基础设施（AGUIClient）
- **领域层**：核心业务逻辑与领域模型（AGUICore）
- **基础设施层**：技术实现细节（AGUICore）

### 模块化
- **AGUICore**：包含协议定义的基础层
- **AGUIClient**：底层客户端基础设施与传输
- **AGUIAgentSDK**：高层 Agent API 与构建器
- **AGUITools**：工具执行框架
- 各模块可独立使用，也可组合使用

## 当前实现状态

### ✅ 已完成
- 领域值对象（ThreadId、RunId、EventTimestamp）
- 基础设施序列化包装器
- 领域/基础设施适配器
- AGUIEvent 协议与 EventType 枚举
- RunStartedEvent 与 RunFinishedEvent 实现
- 事件的完整单元测试
- DDD 分层结构

### 🚧 进行中
- 更多事件类型（TextMessage、ToolCall、StateManagement 事件）
- 客户端 Agent 实现
- 工具执行框架

### 📋 计划中
- HTTP 传输层
- SSE（Server-Sent Events）解析器
- 状态管理
- 认证支持
- 错误处理与恢复