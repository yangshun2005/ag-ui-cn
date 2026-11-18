# AG-UI Android (Views) 示例

基于 Android View 的聊天客户端，使用共享的 Kotlin Multiplatform 核心（`chatapp-shared`），同时 UI 保持传统的 XML/ViewBinding 技术栈。界面仍兼容 Java，但业务逻辑、Agent 存储和流式行为直接来自 Compose 和 SwiftUI 示例所用的共享 Kotlin 模块。

## 亮点

- ♻️ **共享核心** – 复用 `chatapp-shared` 模块处理仓库、认证、聊天编排、工具确认和存储。
- 🧱 **Views + ViewModel** – UI 保持 XML/ViewBinding，Java 的 `ChatActivity` 调用 Kotlin 的 `ChatViewModel`/`ChatController` 桥接层。
- 🧑‍🤝‍🧑 **多 Agent 设置** – 与其他示例相同的 Agent CRUD 体验，由 `AgentRepository` 支撑，通过新的 Kotlin `MultiAgentRepository` 包装器暴露。
- ⚙️ **零 RxJava** – 纯协程/LiveData 互操作；没有围绕 AG‑UI 流程定制的 Java 适配器。
- 🧩 **Kotlin + Java 互操作** – Kotlin 文件提供粘合层（ViewModel、仓库、列表适配器），聊天界面保持 Java 以演示互操作体验。

## 项目结构

```
chatapp-java/
├── app/
│   ├── src/main/java/com/agui/chatapp/java/
│   │   ├── ChatJavaApplication.kt     // 初始化共享平台设置
│   │   ├── repository/                // 围绕 chatapp-shared AgentRepository 的 Kotlin 包装器
│   │   ├── viewmodel/                 // Kotlin ChatViewModel，向 Java UI 暴露 LiveData
│   │   ├── ui/                        // ChatActivity (Java) + SettingsActivity (Kotlin)
│   │   │   └── adapter/               // MessageAdapter (Java) + AgentListAdapter (Kotlin)
│   │   └── model/ChatMessage.kt       // 基于 DisplayMessage 构建的 UI 友好视图状态
│   └── src/main/res/...               // 未改动的 Material 3 XML 布局
├── settings.gradle                    // 通过复合构建包含 :chatapp-shared
└── build.gradle                       // 添加 Kotlin 插件 + AGP 8.12
```

## 各部分如何协作

```
      Java UI (ChatActivity) ─────────┐
         ↑                            │ LiveData
         │ uses ViewModelProvider     │
         │                            ▼
Kotlin ChatViewModel (AndroidViewModel) ── ChatController (chatapp-shared)
         │                            │
         │ coroutines/flows           │
         ▼                            ▼
Kotlin MultiAgentRepository ─── AgentRepository (chatapp-shared)
```

- `ChatController` 处理流式传输、工具确认、认证和消息状态。
- `AgentRepository`（共享）管理持久化 Agent；`MultiAgentRepository` 用 LiveData/CompletableFuture 包装它，供 Java UI 使用。
- `ChatMessage.kt` 将 `DisplayMessage` 对象转换为 RecyclerView 友好的模型。

## 前置条件

- 已安装 Android SDK / 命令行工具（`local.properties` 中的 `sdk.dir` 或 `ANDROID_HOME` 环境变量）。
- JDK 21。
- Kotlin Gradle 插件 2.2.20（通过插件管理自动拉取）。

## 构建

```bash
# 在 chatapp-java/ 目录下
./gradlew :chatapp-shared:assemble     # 可选：预构建共享核心
./gradlew :app:assembleDebug          # 构建 Android 示例
```

> ℹ️ 共享核心需要 Android context。`ChatJavaApplication` 在启动时调用 `initializeAndroid(this)`。

## 从 UI 更新 Agent

- 打开设置 ➜ 添加/编辑/删除 Agent（认证类型：None、API Key、Bearer、Basic）。
- 激活 Agent 会调用 `AgentRepository.setActiveAgent`，立即重连 `ChatController`。
- 系统提示词和认证头通过多平台 `Settings` 存储（Android 上为共享 Preferences）。

## 与原始 Java 示例的差异

| 领域                | 之前                               | 现在 |
|--------------------|--------------------------------------|-----|
| 业务逻辑     | 手写的 Java 仓库和适配器| 共享 `chatapp-shared` 模块 |
| 流式桥接   | 基于 Flow 的 RxJava 包装器             | 直接使用 `ChatController` + LiveData |
| 认证模型        | 自定义 Java `AuthMethod`             | 共享 KMP `AuthMethod` | 
| Agent 存储      | SharedPreferences 手动 Schema      | 共享多平台 `AgentRepository` |
| UI 技术栈           | Java Activity                      | 聊天界面仍为 Java；设置和适配器为方便起见移至 Kotlin |

该示例现在与 Compose/SwiftUI 架构保持一致，同时为尚未准备好使用 Compose 的团队保留经典 Android View 层。