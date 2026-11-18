# AG-UI Kotlin SDK SwiftUI 示例客户端

本示例演示如何将核心 **AG-UI Kotlin 库**与遵循原生 iOS 架构规范的 **SwiftUI** 界面结合。多平台业务逻辑现已移至独立的 `../chatapp-shared` 模块，本项目则新增一个轻量级 Kotlin 桥接层，将共享 Flow 暴露给 Swift。

## 功能特性

- 📱 面向 iPhone 和 iPad 的原生 SwiftUI 体验
- 🤖 由 Kotlin AG-UI 客户端支持的实时流式聊天
- 🧑‍🤝‍🧑 多智能体管理，支持持久化存储
- 🔐 灵活认证（无、API Key、Bearer、Basic、OAuth2、自定义）
- 🧰 工具执行，支持内联确认提示
- 🧵 线程化对话，带临时状态指示

## 项目结构

```
chatapp-swiftui/
├── iosApp/                 # SwiftUI 源码和 XcodeGen 项目定义
├── shared/                 # Kotlin 桥接层，封装 chatapp-shared 供 Swift 使用
├── build.gradle.kts
├── settings.gradle.kts
├── gradlew / gradlew.bat
└── README.md
```
Gradle 构建通过包含项目引用复用 `../chatapp-shared`。Kotlin UI 代码以原生方式在 Swift 中实现。

## 前置条件

- macOS，Xcode 15+
- Android Studio 或 IntelliJ IDEA（用于 Kotlin 开发）
- Kotlin 2.0+ 工具链（由 Gradle wrapper 安装）
- [XcodeGen](https://github.com/yonaskolb/XcodeGen)（用于生成 Xcode 项目）

## 快速开始

1. **构建 Kotlin 框架**

   ```bash
   ./gradlew :shared:assembleXCFramework
   ```

   该任务将 `shared.xcframework` 输出到 `shared/build/XCFrameworks/release/`，供 SwiftUI 项目使用。

2. **生成 Xcode 项目**

   ```bash
   cd iosApp
   xcodegen generate
   ```

3. **打开项目**

   在 Xcode 中打开 `ChatAppSwiftUI.xcodeproj`，选择模拟器或设备，运行应用。

### Swift Package 配置

生成的项目包含一个本地 Swift 包，用于封装 Kotlin 框架。Kotlin 源码变更后需重新运行 Gradle 构建以刷新二进制。

## SwiftUI 架构

Swift 层采用单向数据流：

- `ChatAppStore` 通过 `shared` 桥接模块暴露的 `ChatViewModelBridge` 和 `AgentRepositoryBridge` 辅助类，将 Kotlin Flow 桥接为兼容 Combine 的 `@Published` 属性。
- SwiftUI 视图（`ChatView`、`AgentListView`、`AgentFormView`）订阅 store，并将用户意图派发回 Kotlin 处理。
- Kotlin 仍负责持久化、AG-UI 协议流式传输、认证和工具协调（通过共享的 `ChatController`），展示层交由 SwiftUI。

## 测试与验证

- Kotlin 单元测试仍可通过共享模块运行：`./gradlew :shared:check`
- 框架生成后，可在 Xcode 中构建 SwiftUI 预览快照。

## 故障排查

- 若 Swift 编译器找不到 `shared.xcframework`，请确认 Gradle 构建已成功完成，且 Xcode 指向 release 输出目录。
- 认证密钥通过 `NSUserDefaults` 存储，与 Kotlin 示例使用相同的安全存储后端。
- 工具确认对话框以 SwiftUI 警告形式呈现，与 Compose 的 UX 保持一致。

## 许可证

本示例继承 AG-UI 仓库的许可证。