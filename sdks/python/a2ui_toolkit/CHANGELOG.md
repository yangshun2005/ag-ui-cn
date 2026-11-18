# 更新日志

## 0.0.6 — 2026-11-02

- 新增 `A2UIGenerationError`。适配器可在子代理调用中抛出该异常并附带结构化错误；`run_a2ui_generation_with_recovery` 会记录失败的尝试，并在下一次尝试时将错误反馈给模型，且不超过现有尝试次数上限。其他异常（传输失败、取消）仍原样向上传播。
- 新增可选工具参数 `inference_format`（`"direct_json"` 或 `"express"`，默认 `"direct_json"`），供支持 A2UI Express 的适配器使用。现有调用方不受影响。

### 破坏性变更

无。

## 0.0.5 — 2026-10-15

- 在已发布包的元数据中新增稳定的更新日志 URL，现已显示在 PyPI 上。

### 破坏性变更

无。