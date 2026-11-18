# 为 AGUISwift 做贡献

感谢你为 AGUISwift 做贡献！本指南将帮助你了解我们的开发流程。

## 分支策略

我们使用功能分支工作流，以保证代码质量并支持代码审查：

- **`main`** - 生产就绪代码。受保护分支，禁止直接提交。
- **`feature/*`** - 新功能（例如 `feature/agent-memory-system`）
- **`fix/*`** - Bug 修复（例如 `fix/parsing-edge-case`）
- **`refactor/*`** - 代码重构（例如 `refactor/transport-layer`）
- **`docs/*`** - 文档更新（例如 `docs/api-examples`）

## 开发流程

### 1. 创建功能分支

始终从最新的 `main` 开始：

```bash
# Update main branch
git checkout main
git pull origin main

# Create and switch to feature branch
git checkout -b feature/your-feature-name

# Or for bug fixes
git checkout -b fix/bug-description
```

### 2. 进行修改

遵循 `CLAUDE.md` 中的开发规范：

```bash
# Make changes to code
# Run tests frequently
swift test

# Format code
swift package plugin --allow-writing-to-package-directory swiftformat

# Check for linting issues
swiftlint lint
```

### 3. 提交修改

使用约定式提交信息：

```bash
git add .
git commit -m "feat: add user authentication to HttpAgent

- Implement bearer token authentication
- Add configuration options for auth headers
- Include tests for auth flows

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

**提交信息格式**：
- `feat:` - 新功能
- `fix:` - Bug 修复
- `refactor:` - 代码重构
- `docs:` - 文档变更
- `test:` - 测试新增/更新
- `chore:` - 构建/工具变更

### 4. 推送分支

```bash
# First time pushing the branch
git push -u origin feature/your-feature-name

# Subsequent pushes
git push
```

### 5. 创建 Pull Request

**使用 GitHub CLI**（推荐）：

```bash
# Create PR with title and description
gh pr create --title "Add user authentication" --body "$(cat <<'EOF'
## Summary
Implements user authentication for HttpAgent with bearer token support.

## Changes
- Add `AuthenticationProvider` protocol
- Implement `BearerTokenAuth` provider
- Update `HttpAgentConfiguration` with auth options
- Add comprehensive test coverage

## Testing
- [x] Unit tests pass
- [x] Integration tests pass
- [x] SwiftLint passes
- [x] SwiftFormat applied

## Related Issues
Closes #42

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"

# Or create PR interactively
gh pr create
```

**使用 GitHub Web UI**：

1. 访问 https://github.com/paduh/ag-ui
2. 点击 "Compare & pull request"
3. 填写 PR 模板
4. 点击 "Create pull request"

### 6. 代码审查与合并

- 等待 CI 检查通过
- 等待**已批准的代码审查者**批准（合并必需）
- 通过修改并推送更新来处理审查反馈
- 获得指定审查者批准后，PR 即可合并
- 使用 "Squash and merge" 保持提交历史整洁

## 提交前检查清单

创建 PR 前，请确保：

- [ ] 所有测试通过（`swift test`）
- [ ] 代码已格式化（`swift package plugin swiftformat`）
- [ ] 生产代码无 SwiftLint 违规（`swiftlint lint`）
- [ ] 文档已更新
- [ ] 提交信息遵循约定
- [ ] 分支已与 `main` 同步

## 与 Main 同步

保持功能分支最新：

```bash
# While on your feature branch
git fetch origin
git rebase origin/main

# Or merge if you prefer
git merge origin/main
```

## PR 最佳实践

### PR 标题
使用与提交信息相同的格式：
- `feat: add authentication to HttpAgent`
- `fix: resolve UTF-8 boundary handling in SSE parser`
- `refactor: simplify state management actor`

### PR 描述模板

```markdown
## Summary
Brief description of what this PR does.

## Changes
- List of main changes
- Each change on a new line
- Be specific and clear

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests pass
- [ ] Manual testing performed

## Screenshots (if applicable)
[Add screenshots for UI changes]

## Related Issues
Closes #123
Relates to #456

## Checklist
- [ ] Tests pass
- [ ] Code formatted
- [ ] SwiftLint passes
- [ ] Documentation updated
```

## 测试要求

所有 PR 必须包含遵循 TDD 原则的测试：

1. **先写测试**（红阶段）
2. **实现功能**（绿阶段）
3. **重构**（清理阶段）

详细测试指南见 `CLAUDE.md`。

## 代码审查指南

### 致作者
- 保持 PR 聚焦且小巧（建议 < 400 行）
- 及时回应反馈
- 乐于接受建议
- 范围变更时更新 PR 描述

### 致审查者

**注意**：只有具有批准权限的指定代码审查者才能批准 PR 合并。

#### 审查指南
- 在 24-48 小时内完成审查
- 建设性且尊重他人
- 关注代码质量，而非个人偏好
- 检查所有清单项是否完成
- 验证测试通过并覆盖新功能
- 仅在符合项目标准时批准

## 分支保护规则

`main` 分支受以下保护：
- **合并前需要 Pull Request** - 禁止直接提交到 main
- **需要状态检查通过** - 所有 CI 检查必须为绿色
- **需要指定代码审查者批准** - 只有已批准的审查者才能批准 PR
- **新提交时撤销过期审查** - 新变更会使之前的批准失效
- **禁止强制推送** - 保护提交历史
- **禁止删除** - 分支不可删除

### 设置已批准的审查者

仓库管理员可在 GitHub 中配置已批准的审查者：

1. 进入 **Settings** → **Branches** → **Branch protection rules**
2. 编辑 `main` 分支规则
3. 启用 **"Require a pull request before merging"**
4. 启用 **"Require approvals"**（设为 1 或更多）
5. 启用 **"Restrict who can approve pull requests"**
6. 将指定审查者加入批准列表

这确保只有具备专业知识的可信审查者才能批准代码进入生产。

## 紧急热修复

针对关键生产问题：

```bash
# Create hotfix branch from main
git checkout main
git pull origin main
git checkout -b fix/critical-security-patch

# Make minimal changes
# Test thoroughly
# Create PR with "HOTFIX:" prefix

gh pr create --title "HOTFIX: patch XSS vulnerability"
```

热修复 PR 可快速通道处理，但仍需审查。

## 获取帮助

- 查看 `CLAUDE.md` 了解项目规范
- 参考现有 PR 作为示例
- 如有疑问，在 PR 评论中提问
- 联系维护者获取指导

## 许可证

参与贡献即表示你同意你的贡献将按与项目相同的许可证授权。