# ChatApp Wear OS 示例

一个独立的 Wear OS 示例，复用 `chatapp-shared` Kotlin 多平台核心，在手表上提供 AG-UI 聊天体验。该应用演示如何消费共享的网络、状态和工具确认层，同时使用 `androidx.wear.compose:compose-material3` 渲染 Wear 优化界面。

## 亮点
- 通过 `chatapp-shared` 暴露的 `ChatController` 连接 AG-UI 代理
- 在紧凑的可穿戴布局中展示流式响应、工具确认和错误恢复
- 通过内置代理管理器直接在手表上添加、编辑和激活代理
- 作为独立 Wear OS 应用发布（无需手机伴侣）

## 项目结构
```
chatapp-wearos/
  ├─ wearApp/              # Wear OS 应用模块
  │   ├─ src/main/java/com/agui/example/chatwear/ui
  │   └─ src/main/res
  ├─ build.gradle.kts      # 共享插件声明
  └─ settings.gradle.kts   # 包含 chatapp-shared 以复用
```

## 配置默认代理
应用可通过 Gradle 属性预置初始代理。将以下条目添加到 `~/.gradle/gradle.properties`（或 `local.properties`）文件：

```
chatapp.wear.defaultAgentUrl=https://your-agent-host/v1
chatapp.wear.defaultAgentName=Wear Demo Agent
chatapp.wear.defaultAgentDescription=Sample configuration for the Wear OS demo
chatapp.wear.defaultAgentApiKey=sk-your-api-key
chatapp.wear.defaultAgentApiKeyHeader=X-API-Key
chatapp.wear.quickPrompts=Hello|What can you do?|Summarize today’s updates
```

如果代理不需要身份验证，请将 API 密钥字段留空。未提供默认值时，请在手表上打开 **管理代理** 手动配置。

## 构建与运行

从仓库根目录执行：

```bash
./gradlew :sdks:community:kotlin:examples:chatapp-wearos:wearApp:assembleDebug
```

使用 Android Studio Hedgehog（或更新版本）以及运行 API 30+ 的 Wear OS 模拟器或设备安装生成的 APK。

## 后续步骤
- 将示例指向你自己的 AG-UI 代理，并尝试快捷提示
- 使用 Tiles、Complications 或语音输入扩展 UI，实现免提撰写消息