# AG-UI Kotlin SDK Compose Multiplatform 客户端

一个使用 AG-UI 协议连接 AI 代理的 Compose Multiplatform 聊天客户端。

## 功能特性

- 🎨 **现代 UI**：简洁的极简聊天界面，采用 Material 3 设计
- 🔐 **灵活认证**：支持 API Key、Bearer Token、Basic Auth 和自定义方式
- 🌍 **跨平台**：运行于 Android、iOS 和桌面端（JVM）
- 🤖 **多代理支持**：添加和管理多个 AI 代理
- 💬 **实时流式传输**：逐字符显示 AI 响应
- ⚙️ **设置管理**：持久化存储代理配置
- 🖼️ **A2UI 支持**：渲染代理驱动的动态 UI 界面

## A2UI（Agent-to-UI）支持

聊天客户端支持 [A2UI](https://github.com/google/A2UI)，这是 Google 的代理驱动用户界面规范。连接到支持 A2UI 的代理后，应用可直接在聊天中渲染丰富的交互式 UI 界面。

### 支持的组件

支持全部 18 个标准 A2UI 组件：

| 类别 | 组件 |
|----------|------------|
| **布局** | Column、Row、List、Card、Tabs、Modal、Divider |
| **显示** | Text、Image、Icon |
| **输入** | TextField、CheckBox、Slider、MultipleChoice、DateTimeInput |
| **操作** | Button |
| **媒体** | Video、AudioPlayer（占位 UI） |

### 功能特性

- **动态界面**：代理可实时创建、更新和替换 UI 界面
- **双向数据绑定**：输入组件自动与代理同步状态
- **用户操作**：按钮点击和表单提交会回传给代理
- **流式更新**：UI 界面通过 JSON Patch（RFC 6902）增量更新

### 工作原理

1. 代理通过 AG-UI 的 `ActivitySnapshot`/`ActivityDelta` 事件发送 A2UI 消息
2. `SurfaceStateManager` 处理消息并维护界面状态
3. 来自 [a2ui-4k](https://github.com/AIsOfTheWater/a2ui-4k) 的 `A2UISurface` 使用 Compose Multiplatform 渲染组件树
4. 用户交互生成 `UserActionEvent`，通过 `forwardedProps` 回传

### 连接到 A2UI 代理

chatapp 通过 CopilotKit 桥接连接 A2UI 代理。添加代理时使用桥接端点 URL：

```
http://localhost:3000/api/copilotkit
```

该演示已使用 [CopilotKit/with-a2a-a2ui](https://github.com/CopilotKit/with-a2a-a2ui) 餐厅代理示例进行测试。

## 架构

客户端遵循整洁架构模式，并消费位于 `../chatapp-shared` 的共享核心模块：

- **UI 层**：基于 Material 3 的 Compose Multiplatform UI
- **ViewModel 层**：围绕可复用 `ChatController` 的屏幕特定适配器
- **共享核心**：可复用的仓库、认证和聊天编排逻辑
- **仓库层**：数据管理与持久化
- **认证层**：可扩展的认证提供者系统

## 快速开始

### 前置条件

- JDK 21 或更高版本（构建所需）
- Android Studio 或 IntelliJ IDEA，并安装 Compose Multiplatform 插件
- Xcode 14+（用于 iOS 开发）
- Kotlin 2.2.0 或更高版本

### 运行客户端

#### Android
```bash
./gradlew :androidApp:installDebug
```

#### 桌面端（JVM）
```bash
./gradlew :desktopApp:run
```

#### iOS
1. 在 Xcode 中打开 `chatapp/iosApp/iosApp.xcodeproj`
2. 选择目标设备或模拟器
3. 构建并运行（⌘+R）

**注意**：iOS 应用需要先构建 Kotlin 框架。通过 Xcode 构建时会自动完成。

## 使用说明

### 添加代理

1. 启动应用
2. 点击右上角的设置图标
3. 点击 + 按钮添加新代理
4. 输入代理详情：
   - **名称**：代理的友好名称
   - **URL**：AG-UI 端点（例如 `https://api.example.com/agent`）
   - **描述**：可选描述
   - **认证**：选择并配置认证方式

### 认证方式

#### 无认证
适用于无需认证的公开代理。

#### API Key
- 输入你的 API key
- 可选自定义请求头名称（默认：`X-API-Key`）

#### Bearer Token
- 输入你的 bearer token
- 自动添加 `Authorization: Bearer <token>` 请求头

#### Basic Auth
- 输入用户名和密码
- 自动编码并添加 `Authorization: Basic <encoded>` 请求头

### 与代理聊天

1. 从设置界面选择一个代理
2. 返回聊天界面
3. 输入消息并点击发送
4. 实时观看 AI 响应流

## 扩展认证

添加自定义认证方式：

1. 创建新的 `AuthMethod` 子类：
```kotlin
@Serializable
data class CustomAuth(
    val customField: String
) : AuthMethod()
```

2. 实现 `AuthProvider`：
```kotlin
class CustomAuthProvider : AuthProvider {
    override fun canHandle(authMethod: AuthMethod): Boolean {
        return authMethod is CustomAuth
    }
    
    override suspend fun applyAuth(
        authMethod: AuthMethod, 
        headers: MutableMap<String, String>
    ) {
        // Add your custom headers
    }
}
```

3. 在 `AuthManager` 中注册提供者：
```kotlin
authManager.registerProvider(CustomAuthProvider())
```

## 自定义

### 主题
应用使用 Material 3 主题。在以下文件中自定义颜色：
- `shared/src/commonMain/kotlin/com/agui/example/chatapp/ui/theme/Color.kt`
- `shared/src/commonMain/kotlin/com/agui/example/chatapp/ui/theme/Theme.kt`

### 存储
代理配置使用平台特定的偏好设置存储：
- **Android**：SharedPreferences
- **iOS**：NSUserDefaults
- **桌面端**：Java Preferences

## 生产构建

### Android
```bash
./gradlew :androidApp:assembleRelease
```

### 桌面端
```bash
./gradlew :desktopApp:packageDistributionForCurrentOS
```

### iOS
1. 在 Xcode 项目设置中配置开发团队
2. 配置代码签名和描述文件
3. 通过 Xcode 归档并分发（Product → Archive）

## 故障排查

### 连接问题
- 确认代理 URL 正确且可访问
- 检查认证凭据
- 确保代理实现了 AG-UI 协议

### 性能
- 应用使用 Kotlin 协程实现高效的异步操作
- 消息流式传输经过优化，可平滑更新 UI
- 大型对话历史通过懒加载高效处理

## 依赖

- **agui-kotlin-sdk**：核心 AG-UI 协议实现
- **a2ui-4k**：Compose Multiplatform 的 A2UI 渲染引擎
- **Compose Multiplatform**：UI 框架
- **Voyager**：导航和 ViewModel
- **Ktor**：HTTP 客户端（继承自 agui-kotlin-sdk）
- **kotlinx.serialization**：JSON 处理
- **Multiplatform Settings**：跨平台偏好设置存储

## 许可证

MIT License - 参见父项目的 LICENSE 文件