# AG-UI Kotlin SDK

AG-UI（Agent User Interaction）协议的 Kotlin Multiplatform 实现，支持 JVM、Android、iOS、macOS 和 Wasm（浏览器）平台。

## 特性

- 🎯 **Kotlin Multiplatform** - 一次编写，运行于 JVM、Android、iOS、macOS 和 Wasm
- 🔄 **完整协议支持** - 完整实现 AG-UI 协议
- 📦 **模块化架构** - 三个专注模块：core、client 和 tools
- 🌐 **多种传输方式** - HTTP、SSE（Server-Sent Events）和可扩展的传输层
- 📱 **原生 iOS 支持** - 以 .klib 构建产物发布至 Maven Central
- 🔧 **工具执行框架** - 内置熔断器和重试逻辑

## 模块

- **kotlin-core** - 协议类型、事件和消息定义
- **kotlin-client** - HTTP 传输、SSE 解析、状态管理和高层 Agent API
- **kotlin-tools** - 带注册表和编排的工具执行框架

## 安装

### Maven Central 坐标

SDK 以 `com.ag-ui.community` 组发布至 Maven Central。

最新版本：**查看 [Maven Central](https://central.sonatype.com/artifact/com.ag-ui.community/kotlin-core) 获取当前版本**

### JVM / Android 项目

将 Maven Central 添加到仓库并引入依赖：

```kotlin
// build.gradle.kts
repositories {
    mavenCentral()
}

dependencies {
    val agUiVersion = "0.2.3" // Check Maven Central for latest version

    implementation("com.ag-ui.community:kotlin-core:$agUiVersion")
    implementation("com.ag-ui.community:kotlin-client:$agUiVersion")
    implementation("com.ag-ui.community:kotlin-tools:$agUiVersion")
}
```

## 快速开始

### 基本 Agent 用法

```kotlin
import com.agui.client.HttpAgent
import com.agui.core.RunAgentInput
import kotlinx.coroutines.flow.collect

// Create an HTTP agent
val agent = HttpAgent(baseUrl = "https://your-agent-api.com")

// Run the agent and collect events
agent.run(RunAgentInput(prompt = "Hello, agent!")).collect { event ->
    when (event) {
        is TextMessageDeltaEvent -> println(event.delta.text)
        is RunFinishedEvent -> println("Run completed")
        // Handle other event types...
    }
}
```

### 有状态 Agent（维护对话历史）

```kotlin
import com.agui.client.StatefulAgUiAgent

val statefulAgent = StatefulAgUiAgent(baseUrl = "https://your-agent-api.com")

// First request
statefulAgent.run("Tell me about Kotlin")

// Follow-up request (maintains context)
statefulAgent.run("What about multiplatform support?")

// Access conversation history
val messages = statefulAgent.getMessages()
```

## iOS 项目（Kotlin Multiplatform）

iOS 构建产物以 `.klib` 文件发布至 Maven Central，可直接在 Kotlin Multiplatform 项目中使用。

#### 在 Kotlin Multiplatform 项目中配置

```kotlin
// In your shared module's build.gradle.kts
kotlin {
    // Configure iOS targets
    iosX64()
    iosArm64()
    iosSimulatorArm64()

    sourceSets {
        val commonMain by getting {
            dependencies {
                val agUiVersion = "0.2.3" // Check Maven Central for latest version

                implementation("com.ag-ui.community:kotlin-core:$agUiVersion")
                implementation("com.ag-ui.community:kotlin-client:$agUiVersion")
                implementation("com.ag-ui.community:kotlin-tools:$agUiVersion")
            }
        }
    }
}
```

Kotlin Multiplatform 插件会自动解析正确的 iOS 变体：
- `kotlin-core-iosx64` - 用于 Intel Mac 上的 macOS/iOS 模拟器
- `kotlin-core-iosarm64` - 用于物理 iOS 设备
- `kotlin-core-iossimulatorarm64` - 用于 Apple Silicon Mac 上的 iOS 模拟器

#### 与 Xcode 配合使用

1. **构建共享框架**（在你的 Kotlin Multiplatform 项目中）：
   ```bash
   ./gradlew :shared:linkDebugFrameworkIosArm64
   # or for simulator:
   ./gradlew :shared:linkDebugFrameworkIosSimulatorArm64
   ```

2. **链接框架**到你的 Xcode 项目，与任何 KMP 框架一样

3. **在 Swift 中导入并使用**：
   ```swift
   import Shared

   // Use AG-UI types
   let agent = HttpAgent(/* ... */)
   ```

#### iOS 开发者注意事项

- ✅ **iOS 构建产物已发布至 Maven Central**（自 0.2.3 版本起）
- 📦 iOS 构建产物使用 Kotlin 原生 `.klib` 格式
- 🔧 必须通过 Kotlin Multiplatform 共享模块使用
- ⚠️ 没有 KMP 框架层，无法直接在纯 Swift/Objective-C 项目中使用
- 🎯 KMP 插件会根据构建目标自动处理变体解析

#### 替代方案：本地 Maven 安装

如需本地测试或使用未发布版本：

```bash
cd library
./gradlew publishToMavenLocal
```

然后将 `mavenLocal()` 添加到仓库：

```kotlin
repositories {
    mavenLocal()
    mavenCentral()
}
```

## 平台支持

| 平台 | 状态 | 说明 |
|----------|--------|-------|
| **JVM** | ✅ 完全支持 | Java 21+ |
| **Android** | ✅ 完全支持 | API 26+（Android 8.0） |
| **iOS** | ✅ 完全支持 | arm64、x64、模拟器 arm64 |
| **macOS** | ✅ 完全支持 | arm64 |
| **Wasm** | ✅ 完全支持 | `wasmJs` 浏览器目标 |

## 依赖

- **Kotlin** 2.2.20，使用 K2 编译器
- **Ktor** 3.1.3，用于 HTTP 客户端
- **Kotlinx Serialization** 1.8.1
- **Kotlinx Coroutines** 1.10.2
- **Kotlinx Datetime** 0.6.2
- **Kermit** 2.0.6，用于多平台日志

## 开发

### 从源码构建

```bash
# Build all modules
./gradlew build

# Run tests
./gradlew allTests

# Run tests for specific module
./gradlew :kotlin-core:jvmTest
./gradlew :kotlin-client:jvmTest
./gradlew :kotlin-tools:jvmTest

# Run tests for specific platform
./gradlew jvmTest                    # JVM platform tests
./gradlew iosSimulatorArm64Test      # iOS simulator tests
./gradlew connectedDebugAndroidTest  # Android device tests

# Publish to local Maven
./gradlew publishToMavenLocal

# Generate documentation
./gradlew dokkaHtmlMultiModule
# View at: build/dokka/htmlMultiModule/index.html

# Generate coverage reports
./gradlew koverHtmlReportAll
```

### 项目结构

```
library/
├── core/               # Core protocol types and events
├── client/             # HTTP client and state management
├── tools/              # Tool execution framework
├── build.gradle.kts    # Root build configuration
└── settings.gradle.kts # Module configuration
```

## 文档

- 📚 [API 文档](../../docs/) - 生成的 KDoc 文档
- 💡 [示例](../examples/) - 所有平台的示例应用
- 🌐 [AG-UI 协议规范](https://github.com/ag-ui-protocol/ag-ui)
- 🔧 [开发指南](../CLAUDE.md) - 构建命令和架构

## 许可证

MIT 许可证 - 详见 [LICENSE](../../../../LICENSE)

## 贡献

欢迎贡献！请参阅 [CONTRIBUTING.md](../../../../CONTRIBUTING.md) 了解指南。

## 支持

- 🐛 [报告 Issue](https://github.com/ag-ui-protocol/ag-ui/issues)
- 💬 [讨论](https://github.com/ag-ui-protocol/ag-ui/discussions)
- 📧 社区：[AG-UI Protocol](https://github.com/ag-ui-protocol/ag-ui)

## 致谢

由 AG-UI 社区用 ❤️ 构建。属于 [AG-UI Protocol](https://github.com/ag-ui-protocol/ag-ui) 项目的一部分。