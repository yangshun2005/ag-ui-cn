# AG-UI Flutter Dojo

这是一个 Flutter 示例，使用父级 Dart SDK 和配套的 [Go 示例服务器](../../../go/example/server)。本应用取代了之前的 CLI 示例。本次首次导入保留了源功能；后续集成切片将修复生命周期、协议对齐和确定性验证。

## 工具链与安装

使用 Flutter **3.47.1**（Dart **3.13.1**）和 Git LFS。应用要求 Dart >=3.9、Flutter >=3.35；父级 SDK 保留其独立的 SDK 下限。

```bash
# 从仓库根目录执行：
git lfs pull
cd sdks/community/dart/example
flutter pub get
```

`ag_ui` 依赖解析到 `../`。无需同级检出、Python 后端、pnpm 或私有模块。

## 运行配对

在终端 1 中（该终端已设置有效的 `OPENAI_API_KEY`）：

```bash
# 从仓库根目录执行：
cd sdks/community/go/example/server
go run ./cmd
```

在终端 2 中：

```bash
# 从仓库根目录执行：
cd sdks/community/dart/example
flutter run -d chrome
# 原生 macOS 运行器：
flutter run -d macos
```

桌面/Web 默认地址为 `http://127.0.0.1:8080`。如需使用不同的 Go `PORT` 或主机，编译时指定 `flutter run -d chrome --dart-define=AG_UI_BASE_URL=http://127.0.0.1:9090`。仅在 shell 中导出 `AG_UI_BASE_URL` 不会配置 Flutter。提供商凭据只应放在 Go 服务器环境中。

Web 和 macOS 是本次迁移要求的验证目标；构建和交互结果将在集成后记录。Android、iOS、Linux 和 Windows 运行器保留，目前未验证。物理设备需要可达的主机和显式的 Go 绑定地址；Android 模拟器通常需要其主机桥接地址。不声明支持移动端 HTTP。

## 源码来源

| 来源 | 修订版本 | 处置 |
| --- | --- | --- |
| `github.com/mattsp1290/ag_ui_demo` | `1d1fa327d70d416cac909288497a6a07f94b313d` | 导入的应用、三个测试、六个平台运行器和有意保留的资源。在两个 iOS xcconfig 中加入了本地可选 Pods include，加入了 iOS/macOS Podfile，并在目标位置解析了 macOS Pod 锁文件。 |
| 先前 monorepo Flutter 迁移 | `9ef67a555c0f13ca2e88c4aecaf2c677fe6412fd` | 复用了相对 SDK 接线、未发布包配置和 CLI 移除。当前源码取代了旧页面行为；包含原生运行器。 |
| `github.com/mattsp1290/ag-ui-go-server-example` | `ad756b7afd1c9abc4ebf422aaa28d338d154d2c3` | 保留现有 monorepo 导入用于基于行为的对齐；排除源模块替换和提供商迁移。详细对齐随后与 Go 契约切片一起说明。 |

两个源检出均保持完整。源规划/IDE 文件、日志、生成的 SDK 路径、缓存、Pods 和构建产物已排除。源同级启动器和先前合并的 Go/Flutter 启动器已有意停用。原生 bundle/项目标识保持不变。

## 验证状态

本次导入使用 Flutter 3.47.1 解析依赖，未使用源 `meta` 覆盖。初始分析报告 41 条继承诊断；Flutter 行为切片负责修复这些诊断，不做全局抑制。22 个导入的 Flutter 测试和 macOS 调试构建通过；Flutter 将 macOS 目标迁移到 12.0 并生成了可移植的 Swift Package Manager 接线。父级 Dart SDK 使用 `dart pub get --no-example` 通过 724 个测试，另有 8 个现有跳过。完整契约、交互平台和实时提供商结果待定；仅本次导入不能确立端到端成功。