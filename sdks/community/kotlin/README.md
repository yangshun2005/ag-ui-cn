# AG-UI Kotlin SDK

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Kotlin](https://img.shields.io/badge/kotlin-2.1.21-blue.svg?logo=kotlin)](http://kotlinlang.org)
[![Platform](https://img.shields.io/badge/platform-Android%20%7C%20iOS%20%7C%20macOS%20%7C%20JVM%20%7C%20Wasm-lightgrey)](https://kotlinlang.org/docs/multiplatform.html)
[![API](https://img.shields.io/badge/API-26%2B-brightgreen.svg?style=flat)](https://android-arsenal.com/api?level=26)

一个可用于生产环境的 Kotlin Multiplatform 客户端库，用于将应用连接到实现 [Agent User Interaction Protocol (AG-UI)](https://docs.ag-ui.com/) 的 AI 智能体。

## 📚 文档

**[📖 完整 SDK 文档](../../../docs/sdk/kotlin/)**

完整文档涵盖：

- [快速开始](../../../docs/sdk/kotlin/overview.mdx) - 安装与快速上手
- [客户端 API](../../../docs/sdk/kotlin/client/) - AgUiAgent、StatefulAgUiAgent、构建器
- [核心类型](../../../docs/sdk/kotlin/core/) - 协议消息、事件和类型
- [工具框架](../../../docs/sdk/kotlin/tools/) - 可扩展的工具执行系统

## 🚀 快速开始

```kotlin
dependencies {
    implementation("com.agui:kotlin-client:0.2.3")
}
```

```kotlin
import com.agui.client.*

val agent = AgUiAgent("https://your-agent-api.com/agent") {
    bearerToken = "your-api-token"
}

agent.sendMessage("Hello!").collect { event ->
    // Handle streaming responses
}
```

## 💻 开发环境搭建

```bash
git clone https://github.com/ag-ui-protocol/ag-ui.git
cd ag-ui/sdks/community/kotlin/library
./gradlew build
./gradlew test
```

## 发布

Kotlin 使用共享的 [prepare-release](../../../.github/workflows/prepare-release.yml) 和
[publish-release](../../../.github/workflows/publish-release.yml) 工作流。选择
`sdk-kotlin`，使用 `create-pr` 操作，并选择稳定的补丁、次版本或主版本升级。
合并 `release/next` 前，请审阅生成的三条模块变更日志条目。
首次自动发布在没有先前包标签时，历史记录是近似值；请仔细审阅这些说明。Maven Central 发布仅支持稳定版本。

合并后，共享 CI 会将三个模块版本与 Maven Central 比对，并对新版本运行 Gradle 测试和无签名暂存。暂存仓库在同一工作流运行的作业之间，于所选源提交处传递。
Kotlin 最后发布，等待生成的 POM 可见，然后记录包标签和已审阅的说明。全部 24 个 Gradle 发布产物都会被暂存，
包括面向 Intel Mac 用户的 iosX64。

工作流试运行会构建待发布的包，但不执行部署、发布记录、分支清理或通知。已发布的版本无需暂存。

本地命令（在本目录运行）需要 JDK21、Python3.12、jq、Android SDK36
和 build-tools36.0.0，macOS 上还需 Xcode 以进行多平台暂存：

```bash
./publish.sh --dry-run       # Real clean/tests/staging; no credentials
./publish.sh --stage-only    # Same staging path; retain unsigned repository
./publish.sh                 # Stage and upload the full repository
./publish.sh --deploy-only --repository /path/to/staged/repository
```

仅部署模式使用暂存仓库；
不会重新构建或测试。部署需要环境变量
`JRELEASER_MAVENCENTRAL_SONATYPE_USERNAME`、
`JRELEASER_MAVENCENTRAL_SONATYPE_PASSWORD`、`JRELEASER_GPG_PASSPHRASE`、
`JRELEASER_GPG_PUBLIC_KEY` 和 `JRELEASER_GPG_SECRET_KEY`。请将凭据保存在
你的密钥管理器中。本地命令不会创建共享工作流的发布记录。

## 📄 许可证

MIT 许可证 - 详见 [LICENSE](LICENSE)。