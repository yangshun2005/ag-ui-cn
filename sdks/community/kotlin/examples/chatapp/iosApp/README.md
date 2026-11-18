# AG-UI Kotlin SDK 聊天客户端 iOS App

这是 AG-UI Kotlin SDK 聊天客户端示例的 iOS 实现。

## 环境要求

- **Xcode 15.0 或更高版本**（推荐）
- **iOS 14.1+ 部署目标**
- **搭载 Apple Silicon 或 Intel 处理器的 macOS**
- **JDK 21**（用于构建 Kotlin 框架）

## 快速开始

### 1. 在 Xcode 中打开

```bash
# 从 chatapp 目录执行
open iosApp/iosApp.xcodeproj
```

### 2. 选择模拟器并运行

1. 在 Xcode 中，从设备下拉菜单选择模拟器（例如 **iPhone 16 Pro**）
2. 按 **Cmd+R** 或点击 **Play** 按钮（▶️）
3. Xcode 会自动构建 Kotlin 框架并启动 App

## 构建与运行

### 方法 1：使用 Xcode（推荐）

**分步说明：**

1. **打开 Xcode 项目：**
   ```bash
   open iosApp/iosApp.xcodeproj
   ```

2. **等待项目索引完成**（首次可能需要几分钟）

3. **选择目标设备：**
   - 点击 scheme 旁的设备/模拟器下拉菜单
   - 选择 iOS 模拟器（例如 iPhone 16 Pro、iPad Pro）
   - 或连接实体 iOS 设备

4. **构建并运行：**
   - 按 **Cmd+R** 或点击 **Play** 按钮
   - 首次构建耗时较长，因为需要编译 Kotlin 框架
   - App 会自动启动

### 方法 2：命令行构建

```bash
# 仅构建（不运行）
xcodebuild -project iosApp/iosApp.xcodeproj -scheme iosApp -sdk iphonesimulator build

# 在指定模拟器上构建并运行
xcodebuild -project iosApp/iosApp.xcodeproj \
  -scheme iosApp \
  -sdk iphonesimulator \
  -destination 'platform=iOS Simulator,name=iPhone 16 Pro' \
  build
```

### 方法 3：iOS 设备（需要 Apple 开发者账号）

1. **通过 USB 或无线连接** iPhone/iPad
2. 设备提示时**信任此电脑**
3. 在 Xcode 的设备下拉菜单中**选择你的设备**
4. **配置签名：**
   - 进入项目设置 → Signing & Capabilities
   - 选择你的开发团队
   - 确保 bundle identifier 唯一
5. **构建并运行**（Cmd+R）

## 可用模拟器

查看可用模拟器：
```bash
xcrun simctl list devices available | grep iPhone
```

常用测试模拟器：
- **iPhone 16 Pro** - 最新 iPhone，支持全部功能
- **iPhone SE（第 3 代）** - 小屏幕测试
- **iPad Pro 11 英寸** - 平板界面测试

## 构建流程说明

### 自动构建框架

Xcode 项目包含一个 **"Run Script" 构建阶段**，会自动：

1. 在编译 Swift 代码前**构建 Kotlin Multiplatform 框架**
2. **执行：** `./gradlew :shared:embedAndSignAppleFrameworkForXcode`
3. **生成：** 框架文件到 `shared/build/xcode-frameworks/`
4. **链接：** 将框架与 iOS App 链接

### 手动构建框架（如需要）

如果自动构建失败，可手动构建：

```bash
# 从 chatapp 目录执行
./gradlew :shared:embedAndSignAppleFrameworkForXcode

# 或清理后重新构建
./gradlew clean :shared:embedAndSignAppleFrameworkForXcode
```

## 项目结构

```
iosApp/
├── iosApp.xcodeproj/          # Xcode 项目文件
│   └── project.pbxproj        # 项目配置
├── iosApp/                    # iOS App 源码
│   ├── iOSApp.swift          # App 主入口（@main）
│   ├── ContentView.swift     # Compose 的 SwiftUI 封装
│   ├── Info.plist            # iOS App 配置
│   └── Assets.xcassets/      # App 图标和资源
└── README.md                  # 本文件
```

### 关键文件说明

- **`iOSApp.swift`** - Swift App 入口，设置主窗口
- **`ContentView.swift`** - 在 SwiftUI 中封装 Kotlin Compose UI
- **`Info.plist`** - iOS App 元数据、权限、部署目标
- **`project.pbxproj`** - Xcode 项目配置、构建设置

## 功能

### ✅ 可用功能

- **完整 AG-UI 协议支持** - 连接 AI 智能体
- **原生 iOS 界面** - SwiftUI + Compose Multiplatform
- **实时聊天** - 消息流式传输与响应
- **多智能体** - 在不同 AI 服务间切换
- **认证支持** - API key、Bearer token、Basic auth
- **位置工具** - iOS CoreLocation 集成，支持基于位置的 AI 工具
- **跨平台数据** - 共享设置和聊天历史

### 🚀 iOS 专属增强

- **原生 iOS 键盘**处理
- **iOS 导航模式**
- **支持 iOS 深色/浅色模式**
- **原生 iOS 分享**（如已实现）
- **iOS 通知支持**（如需要）

## 测试

### 内置测试

运行 iOS 实现的测试：

```bash
# 测试 iOS 位置提供器
./gradlew :tools:iosSimulatorArm64Test

# 测试 iOS 平台函数
./gradlew :shared:iosSimulatorArm64Test

# 测试所有平台
./gradlew test
```

### 手动测试清单

**基础功能：**
- [ ] App 启动无崩溃
- [ ] 聊天界面正常显示
- [ ] 可在聊天输入框输入消息
- [ ] 设置界面可访问

**智能体连接：**
- [ ] 可添加新的智能体配置
- [ ] 认证方式可用（API key、Bearer token）
- [ ] 可连接智能体并发送消息
- [ ] 响应在聊天中正确显示

**iOS 专属：**
- [ ] 键盘正常显示/隐藏
- [ ] App 在竖屏和横屏下均可用
- [ ] App 间切换正常
- [ ] 内存占用合理

**位置工具（如可用）：**
- [ ] 位置权限对话框出现
- [ ] 授予权限后位置工具可用
- [ ] 权限被拒时有正确的错误处理

## 故障排查

### 常见构建问题

**1. Kotlin 框架构建失败**
```bash
# 清理并重新构建框架
./gradlew clean
./gradlew :shared:embedAndSignAppleFrameworkForXcode
```

**2. Xcode 构建错误**
- **清理构建文件夹：** Xcode 中按 Shift+Cmd+K
- **清除派生数据：** Xcode → Preferences → Locations → Derived Data → Delete
- **重启 Xcode** 后重试

**3. 代码签名问题**
- 进入 **项目设置 → Signing & Capabilities**
- 选择你的**开发团队**
- 开发时使用**自动签名**
- 确保 **bundle identifier 唯一**

**4. 模拟器问题**
```bash
# 重置模拟器
xcrun simctl erase all

# 列出可用模拟器
xcrun simctl list devices available

# 启动指定模拟器
xcrun simctl boot "iPhone 16 Pro"
```

**5. 缺少命令行工具**
```bash
# 安装/更新 Xcode 命令行工具
xcode-select --install

# 验证安装
xcode-select -p
```

### 性能建议

**首次构建优化：**
- 首次构建耗时 2-5 分钟（编译 Kotlin 框架）
- 后续构建快得多（增量编译）
- 保持 Xcode 打开以维持构建缓存

**内存管理：**
- 关闭未使用的模拟器以释放内存
- 开发时使用 "Debug" 构建配置
- "Release" 构建针对分发优化

## 位置功能

### 位置权限设置

App 包含 iOS CoreLocation 集成。使用位置功能：

1. **使用位置工具时会自动请求位置权限**
2. **添加位置用途说明**（已包含在 Info.plist 中）：
   ```xml
   <key>NSLocationWhenInUseUsageDescription</key>
   <string>This app needs location access to provide location-based features to AI agents</string>
   ```

### 测试位置功能

**在模拟器中：**
- Simulator → Features → Location → Custom Location
- 输入坐标测试位置功能
- 尝试不同的精度设置

**在设备上：**
- 提示时授予位置权限
- 在不同环境中测试（室内/室外）
- 验证精度级别正常工作

## 高级配置

### 自定义 Bundle Identifier

如需修改，在项目设置中更新：
```
com.agui.example.chatapp
```

### iOS 部署目标

当前：**iOS 14.1+**
- 支持大多数现代 iOS 设备
- 兼容 SwiftUI 和 Compose Multiplatform
- 如需可降低（请检查兼容性）

### 构建配置

- **Debug：** 启用调试的开发构建
- **Release：** 针对分发优化的构建

## 支持与后续步骤

### 开发工作流

1. 在 Kotlin 共享代码中**进行修改**
2. **构建框架：** `./gradlew :shared:embedAndSignAppleFrameworkForXcode`
3. **在 Xcode 中运行**以测试修改
4. 按需**重复**

### 分发

**TestFlight** 或 **App Store** 分发：
1. 归档 App（Product → Archive）
2. 上传到 App Store Connect
3. 配置 App 元数据和截图
4. 提交审核

### 获取帮助

- **Xcode 问题：** 查看 Xcode Console 获取详细错误信息
- **Kotlin/Multiplatform 问题：** 查看 Gradle 构建输出
- **iOS 相关问题：** 参考 Apple 开发者文档
- **AG-UI 协议问题：** 查看主项目文档