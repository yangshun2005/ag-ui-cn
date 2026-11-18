# 更新日志

本文件记录 AG-UI-4K Chat App 示例的所有重要变更。

格式基于 [Keep a Changelog](https://keepachangelog.com/en/1.0.0/)，
本项目遵循 [语义化版本](https://semver.org/spec/v2.0.0.html)。

## [未发布]

### 新增
- **clawg-ui 配对支持** - 为 clawg-ui 网关端点自动配对设备
  - 检测 `/v1/clawg-ui` URL，未配置 Bearer token 时启动配对流程
  - 显示配对对话框，包含：
    - 大号可复制的配对码
    - 用户操作说明
    - 网关所有者的批准命令（如 `openclaw pairing approve clawg-ui <code>`）
  - 配对完成后自动将 Bearer token 保存到 Agent 配置
  - 带重试逻辑的 token 验证，用于处理待批准状态
  - 完整状态机：Idle → Initiating → PendingApproval → RetryingConnection → AwaitingApproval/Failed

### 新增文件
- `ClawgUiPairingService.kt` - 配对协议核心实现
- `ClawgUiPairingResponse.kt` - 配对响应与状态的数据模型
- `ClawgUiPairingDialog.kt` - 覆盖所有配对状态的 Compose UI 对话框

### 技术细节
- 使用 AG-UI SDK 类型（`RunAgentInput`、`UserMessage`）进行 token 验证
- 使用 `kotlinx.datetime.Clock` 获取时间戳，兼容多平台
- 使用正确的 SSE 请求头（`Accept: text/event-stream`）以兼容 AG-UI 端点
- 解析服务器配对信息结构中的嵌套 JSON 响应

## [0.2.5] - 2025-12-29

### 新增
- **A2UI（Agent-to-UI）支持** - 在聊天中渲染 Agent 驱动的动态 UI 界面
  - 支持全部 18 个标准 A2UI 组件（Layout、Display、Input、Action、Media）
  - 通过 JSON Patch（RFC 6902）实时更新界面
  - 输入组件支持双向数据绑定
  - 用户操作事件通过 `forwardedProps` 回传给 Agent
- **完整的 iOS 实现**，功能与 Android 和 Desktop 版本对齐
- iOS 应用结构，使用 SwiftUI 包装 Compose Multiplatform UI
- iOS 平台特定实现：
  - `IosPlatform.kt`，基于 NSUserDefaults 的配置存储
  - `MainViewController.kt`，使用 ComposeUIViewController 作为 iOS 应用入口
- iOS source set 配置，具备正确的依赖层级
- iOS 特定测试：
  - `IosSettingsTest.kt`，测试 NSUserDefaults 持久化
  - `IosUserIdManagerTest.kt`，测试 iOS 特定 UserIdManager 功能
- iOS 应用项目（`iosApp/`），包含：
  - Xcode 项目配置
  - SwiftUI ContentView 包装 Kotlin Multiplatform UI
  - iOS 15.0+ 部署目标
  - 与共享 Kotlin 代码的框架集成

### 变更
- **升级 Compose Multiplatform** 至 1.9.3
- **用 Kotlin Multiplatform 方案替换 JVM 特定线程构造**：
  - 用 `kotlinx.atomicfu.atomic` 替换 `@Volatile` 和 `synchronized`，实现线程安全单例
  - 更新 `UserIdManager` 和 `AgentRepository` 使用原子操作
- **修复多平台兼容性问题**：
  - 文件大小工具中用多平台兼容的字符串格式化替换 `String.format()`
  - 移除 iOS 平台不可用的 `@TestOnly` 注解
- **增强 ID 生成**：`AgentConfig.generateId()` 加入随机成分，防止 ID 重复
- **更新 iOS 部署目标**：从 14.1 升至 15.0，以匹配框架要求
- **改进字符串格式化**：`Extensions.kt` 中改为跨平台兼容写法
- **升级依赖**：
  - Gradle wrapper 升级至 8.14
  - Kotlin 插件升级至 2.2.0
- **增强构建配置**：
  - 添加 `org.gradle.console=plain`，减少控制台格式化错误
  - 修复 Gradle wrapper 缺失文件
  - 配置 iOS source set 层级及正确的目标依赖

### 修复
- **A2UI DataChangeEvent 处理** - 按 A2UI 协议过滤 DataChangeEvents（仅本地状态更新，不发送到服务器）
- **A2UI 操作名兼容性** - 同时发送 `name`（A2UI 规范）和 `actionName`（CopilotKit 变通方案），兼容演示应用
- **隐藏 A2UI 操作消息** - 聊天中不再显示 "[A2UI Action]" 占位消息
- **抑制取消错误** - CancellationExceptions 不再显示为错误消息
- **iOS 模拟器支持** - 在 Xcode 项目中添加 `SUPPORTED_PLATFORMS`，支持模拟器构建
- **Android 明文 HTTP** - 启用 `usesCleartextTraffic`，便于本地开发
- **Xcode 构建脚本路径问题** - 修正 iOS 构建阶段中 gradlew 路径解析
- **Java 运行时检测** - 解决 Xcode 构建环境中的 JDK 路径问题
- **线程兼容性** - 移除 JVM 特定并发构造
- **Source set 冲突** - 解决重复的平台实现
- **框架链接** - 修复 Swift 代码与 Kotlin 框架的集成
- **构建工具集成** - 确保 Xcode 环境中 Java/Gradle 正确集成

### 依赖
- **a2ui-4k**（`com.contextable:a2ui-4k:0.8.1`）- Compose Multiplatform 的 A2UI 渲染引擎

### 技术细节
- **Kotlin Multiplatform**：三个平台（Android、Desktop、iOS）现在共享公共业务逻辑
- **Compose Multiplatform**：所有平台统一 UI 框架
- **平台特定存储**：
  - Android：SharedPreferences
  - Desktop：Java Preferences
  - iOS：NSUserDefaults
- **认证**：跨平台认证提供者系统，支持 API Key、Bearer Token 和 Basic Auth
- **测试**：覆盖所有平台的完整测试套件，含平台特定测试实现

### 平台支持
- ✅ Android（API 26+）
- ✅ Desktop/JVM（Java 21+）
- ✅ iOS（15.0+）- **新增**

### 开发者体验
- 完整的 iOS 开发流程文档
- 可直接用于 iOS 开发的 Xcode 项目
- 跨平台测试套件
- 支持所有平台的统一构建系统