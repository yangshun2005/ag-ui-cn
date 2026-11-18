# ChatApp 共享核心

一个 Kotlin Multiplatform 模块，提供聊天应用示例复用的非 UI 逻辑。

## 职责

- 通过多平台 `AgentRepository` 实现 Agent 持久化
- 认证提供方（API Key、Bearer、Basic、OAuth2/自定义 Hook 点）
- 通过与 UI 无关的 `ChatController` 进行聊天编排，现已将 Kotlin 客户端 `AgentSubscriber` Hook 接入，跨平台填充对话历史和临时数据
- 平台工具（设置存储、用户 ID 管理、字符串辅助函数）
- 跨平台共享的工具确认集成

Compose Multiplatform、SwiftUI 和 Android Views（chatapp-java）示例均依赖此模块处理网络、持久化和业务规则。

## 目标平台

该模块提供 Android、JVM 桌面和 iOS 目标。使用方直接编译（Compose）或将其打包进 XCFramework（SwiftUI）。

## 构建

在 `chatapp-swiftui` 或 `chatapp` 项目根目录下，可通过 `:chatapp-shared` 引用该 Gradle 项目。

```bash
./gradlew :chatapp-shared:assemble
```

SwiftUI 示例的 `:shared` 模块将此核心组件打包为 `shared.xcframework` 供 Xcode 使用。