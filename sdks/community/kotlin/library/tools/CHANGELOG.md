# 更新日志

## 0.5.0 — 2026-11-05

- 新增 wasmJs（浏览器）和 macosArm64 目标。
- CircuitBreaker 现在接受注入的 kotlinx.datetime Clock；原有的单参数构造函数保留，默认使用 Clock.System。

### 破坏性变更

- Group ID 改为 com.ag-ui.community；移除了 com.contextable 引用。
- minSdk 降至 24；Kotlin 固定为 2.1.20。