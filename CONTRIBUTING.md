# 为 AG-UI 做贡献

感谢你关注 AG-UI！无论你是来修复 bug、交付功能、改进文档，还是只想弄清楚它是怎么运作的——我们都很高兴你在这里。

以下是参与方式：

---

## 有疑问或遇到了问题？

选对地方，我们能更快帮到你：

- **我想做贡献 [修复 / 功能请求]** → [GitHub Issues](https://github.com/ag-ui-protocol/ag-ui/issues)
- **"我该怎么做……？"** → [Discord](https://discord.gg/Jd3FzfdJa8) → `#-💎-contributing`
- **自我介绍** → [Discord](https://discord.gg/Jd3FzfdJa8) → `🤝-intro`

---

## 想贡献代码？

首先，一个重要的请求：
**在开始任何新功能或现有功能的重要工作之前，请务必先联系我们。**

我们热爱社区贡献！但话说回来，我们希望在你开始之前大家能达成共识。
投入大量时间和精力后才发现它与上游项目不一致，这种感觉很糟糕，我们不希望发生这种事。
这也有助于确认你计划的工作尚未有人在做。

如果你已确认 **[x]** 工作尚未开始，请先提交一个 Issue：https://github.com/ag-ui-protocol/ag-ui/issues

1. **找到要做的事**
   浏览 [GitHub](https://github.com/ag-ui-protocol/ag-ui/issues) 上的开放 Issue。
   有自己的想法？先开一个 Issue，方便我们开始讨论。

2. **申请被指派**
   在 Issue 下评论并 @ 一位代码负责人：
   → [代码负责人](https://github.com/ag-ui-protocol/ag-ui/blob/main/.github/CODEOWNERS)

3. **进入路线图**
   获批后，你将被指派该 Issue，它会被加入我们的[路线图](https://github.com/orgs/ag-ui-protocol/projects/1)。

4. **与他人协调**
   - 如果你在协作或需要反馈，在 Discord 的 `#-💎-contributing` 中发起讨论
   - 或直接私信被指派人

5. **提交 Pull Request**
   - 准备好后，提交你的 PR
   - 在描述中包含：`Fixes #<issue-number>`
     （这会将你的 PR 关联到该 Issue 并自动关闭它）

6. **审查与合并**
   - 维护者会审查你的代码，必要时留下评论
   - 获批后，我们会合并它并将 Issue 移至"完成"。

**注意：** 所有社区集成（即 .NET、Golang SDK 等）都需要由做出贡献的社区成员维护。

---

## 工具链

本仓库在根目录 `.node-version` 指定的 Node 版本上开发和测试。`fnm` 直接读取该文件并解析裸主版本号，因此在仓库根目录执行 `fnm use` 即可获得正确的 Node。`nvm` 只读取 `.nvmrc`，所以需要传入版本号（`nvm use "$(cat .node-version)"`）。`nodenv` 和 `asdf` 也读取 `.node-version`——`asdf` 需在 `~/.asdfrc` 中设置 `legacy_version_file = yes`——但两者都会将其解析为精确安装的版本名，因此在纯 `nodenv` 下裸主版本号根本无法解析：请安装精确版本（`nodenv install 22.11.0`），并添加 [`nodenv-aliases`](https://github.com/jasonkarns/nodenv-aliases) 插件或设置 `NODENV_VERSION`。

在 CI 中，全部 12 个 `actions/setup-node` 步骤都通过 `node-version-file` 读取该文件，而不是内联指定版本。目前还没有任何机制自动强制这一点，因此新增一个内联写版本的步骤不会被发现——如果你要添加，请复制现有步骤，而不是从头编写输入。pnpm 也是如此，目前有两种不同的固定方式：11 个 `pnpm/action-setup` 步骤中有 9 个内联固定 `version:`，2 个依赖 `package.json#packageManager`。

Python 构建工具链以同样的方式固定，记录在 [`.github/python-toolchain.env`](.github/python-toolchain.env) 中，但这一项*是*被强制执行的——当工作流中的字面值与记录不一致时，`scripts/release/verify-python-toolchain-pins.sh` 会让 CI 失败。为 Node 和 pnpm 填补同样的缺口记录在 [PNI-280](https://linear.app/copilotkit/issue/PNI-280) 中；该模式被强制执行后的版本见第 7 步的 Python 说明。

要升级 Node 主版本，编辑 `.node-version`——每个 `actions/setup-node` 步骤都会自动跟随。然后检查 `publish-release.yml` 中的 npm-OIDC 变通方案，它存在只是因为 Node 22 自带的 npm 无法使用 OIDC 可信发布，因此后续主版本大概应该删除它。不会有任何东西提醒你这件事，所以写在这里。

以上都不是 `package.json#engines.node`。根清单是私有的且从不发布，所以它的 `>=18` 是*本仓库内*安装的下限——也是在此集中化之前 `node-version-file: "package.json"` 所解析的内容。已发布的 `@ag-ui/*` 包大多根本不声明 engines 范围，因此修改根值不是消费者兼容性决策。

---

## 添加集成 PR 的分步指南

本指南带你完成向 AG-UI 提交集成 PR 所需的一切。涵盖添加集成代码、示例、dojo 配置、端到端测试和 CI 设置。

请全程参考 `integrations/` 中的现有集成（例如 `integrations/adk-middleware/` 或 `integrations/langgraph/`）作为实现范例。

### 第 1 步：添加集成文件夹

你的集成代码放在 `integrations/` 文件夹内，以你的集成命名的子文件夹下（例如 `integrations/my-framework/`）。

- **语言子文件夹**——按语言组织。例如，如果你的集成是 Python，放在 `integrations/my-framework/python/` 下。如果支持多种语言（例如 Python 和 Rust），使用 `python/` 和 `rust/` 等独立子文件夹。
- **示例子文件夹**——在语言文件夹内包含一个 `examples/` 目录（例如 `integrations/my-framework/python/examples/`）。dojo 示例必须放在这里，但你也可以包含额外示例。
- **TypeScript 客户端文件夹（必需）**——无论集成使用什么语言，都必须包含一个 `typescript/` 文件夹。至少包含重新导出 HTTP agent 的 TypeScript 客户端代码。你可以从 `integrations/adk-middleware/typescript/` 等现有集成复制作为参考。它包含 `package.json`、TypeScript 配置和客户端代码本身。如果你的框架原生支持 TypeScript，完整的 TypeScript 实现也应放在此包中。
- **`js/` 文件夹（可选）**——当 `typescript/` 已存放 HTTP 客户端，且你还想要原生 Node.js 实现时使用。示例：`integrations/adk-middleware/js/`。dojo 可以进程内运行它，因此不需要服务器端口。

**示例结构：**
```
integrations/my-framework/
├── python/
│   ├── examples/          # Dojo 示例放在这里
│   │   ├── pyproject.toml
│   │   └── ...
│   ├── pyproject.toml     # 集成包
│   └── ...
└── typescript/
    ├── package.json
    ├── tsconfig.json
    └── src/
        └── index.ts       # 重新导出 HTTP agent
```

### 第 2 步：在 Dojo 中注册你的集成

你需要更新 `apps/dojo/src/` 中的三个文件，让 dojo 识别你的集成：

- **`agents.ts`**——为你的集成添加条目。你选择的**对象键**很重要，因为它必须在其他配置文件中完全匹配。如果你的框架支持多种变体——不同语言、运行时或传输模式——每个变体都有自己的独立条目。例如，LangGraph 有 LangGraph Platform（Python）、LangGraph FastAPI（Python）和 LangGraph TypeScript 的条目。
- **`menu.ts`**——将你的集成添加到侧边栏菜单。**`id`** 必须与你在 `agents.ts` 中使用的对象键匹配。**`name`** 是左侧边栏显示的人类可读标签，不需要与 ID 匹配。每个条目还定义它支持的功能（例如 `agentic_chat`、`human_in_the_loop`、`agentic_generative_ui`）。此文件是集成配置的唯一事实来源。
- **`env.ts`**——定义你的 agent 托管 URL 的环境变量（每个 agent 一个）。dojo 借此知道运行时如何访问你的 agent。默认值应与示例代码使用的 host/port 匹配。

### 第 3 步：配置 Agent 映射

`agents.ts` 中的每个条目包含功能键的映射。这通常是一对一映射，每个键对应一个 agent。对大多数集成来说这很简单——一个功能映射到一个 agent 名称。如果你的框架处理多个 agent 协同工作，可能会列出多个 agent，但每个仍有自己的条目。

### 第 4 步：设置环境变量

你的示例代码必须：

- **绑定到 host `0.0.0.0`**（或可通过 `HOST` 环境变量覆盖）
- **遵循 `PORT` 环境变量**——当 dojo 设置特定端口时，你的 agent 必须绑定到该确切端口

`env.ts` 中定义的端口值必须与 `agents.ts` 中配置的 URL 匹配。如果不对齐，dojo 将无法找到你的 agent。

### 第 5 步：添加 Dojo 脚本

在 `apps/dojo/scripts/` 的 dojo 脚本配置中为你的集成添加条目。有两个脚本需要更新：

- **`prep-dojo-everything.js`**——这是"准备"命令。它安装依赖并构建你的模块（例如 `pnpm install`、`uv sync`、`poetry install`、`go build`）。它**不**启动任何服务器。
- **`run-dojo-everything.js`**——这是"运行"命令。它启动你的集成的 agent 服务器。

在两个脚本中，你都要向 `ALL_TARGETS` 对象添加条目。**对象键必须匹配**你在 `agents.ts` 中使用的键。每个条目包括：
- 用于日志的**名称**
- 要执行的**命令**（例如 prep 用 `uv sync`，run 用 `uv run ...`）。使用纯 `uv sync`——示例应用故意以非冻结方式同步（见第 7 步的 Python 说明）。
- **工作目录**（指向你的 `integrations/` 示例文件夹）
- **环境变量**（可选）——例如 `PORT`

**`run-dojo-everything.js` 的重要规则：**
- **端口不得冲突**任何其他集成。选择下一个可用的最高端口号。
- 同一文件中的 `dojo` 和 `dojo-dev` 条目需要指向你服务端口的环境变量，以便 dojo 知道如何访问你的 agent。
- 如果你的集成运行**多个 agent**，run 中可以有多个条目。参见 `a2a-middleware` 了解此模式的示例。

此时，你应该能在本地启动 dojo 并看到你的集成正常工作。

### 第 6 步：添加端到端测试

你在侧边栏条目（`menu.ts` 中）列出的每个功能都需要对应的端到端测试。**没有测试，你的 PR 不会被视为就绪。**

- **创建测试文件夹**，在 `apps/dojo/e2e/tests/` 内为你的集成创建（例如 `apps/dojo/e2e/tests/myFrameworkTests/`）。你支持的每个功能在此文件夹内都有自己的 spec 文件。
- **遵循现有测试模式**——查看其他集成如何实现测试。如果其他框架使用 `apps/dojo/e2e/featurePages/` 中的共享辅助函数，你也应使用 `featurePages`。不过，有些测试使用 `apps/dojo/e2e/pages/<framework-name>/` 中框架特定的页面对象。如果其他框架的相同测试位于 `pages/some-framework`，你需要将其复制到 `pages/my-framework` 并为你的集成适配。
- **提交 PR 前在本地运行测试**。在 `apps/dojo/` 下，一个终端中：
  ```bash
  ./scripts/prep-dojo-everything.js --only dojo,my-framework
  ./scripts/run-dojo-everything.js --only dojo,my-framework
  ```
  然后在另一个终端中，从 `apps/dojo/e2e/`：
  ```bash
  pnpm install
  pnpm test tests/myFrameworkTests/
  ```

### 第 7 步：添加 CI 配置

端到端测试也需要在 CI 中运行。更新 `.github/workflows/dojo-e2e.yml` 的 GitHub Actions 工作流文件：

- **将你的集成添加到工作流顶部的测试矩阵**。条目名称必须与你在 `agents.ts` 中使用的键匹配。这告诉 CI 使用哪个测试路径（例如 `tests/myFrameworkTests`）。
- **添加 services 部分**，定义要构建和运行的服务。服务名称映射回 `prep-dojo` 和 `run-dojo` 脚本。CI 工作流使用 `wait-on` 命令在运行测试前检查服务是否响应（通过 TCP/HTTP）。

**注意：** 外部 PR 默认不会运行测试。团队会从仓库内另开一个 PR 触发 CI，测试通过后再合并原始贡献者 PR。

**Python 集成：** CI 为每个 Python 作业固定一个 uv 版本和一个 CPython 版本，记录在 [`.github/python-toolchain.env`](.github/python-toolchain.env) 中。由此有两点要求，CI 都会强制执行。

**1. 新的 Python CI 步骤必须使用固定值。** 如果任何工作流的字面值与该文件不一致，或 `setup-uv` 调用以其他方式获取版本，`python-toolchain-pins` 作业就会失败。在工作流顶层 `env:` 中声明这两个值（工作流无法从文件中读取它们），然后引用：

```yaml
env:
  UV_VERSION: "0.12.1"      # 必须等于 .github/python-toolchain.env
  PYTHON_VERSION: "3.12"

# ...
      - uses: astral-sh/setup-uv@<sha>
        with:
          version: ${{ env.UV_VERSION }}
          python-version: ${{ env.PYTHON_VERSION }}
```

venv 缓存键必须携带两个版本，这样由一个工具链构建的环境绝不会被恢复到期望另一个工具链的作业中。匹配现有写法：

```
py${{ env.PYTHON_VERSION }}-uv${{ env.UV_VERSION }}
```

`actions/setup-python` 步骤从同一固定值获取 `python-version`。注意作用域：`env:` 是按工作流/作业/步骤的，因此步骤看不到声明的 `${{ env.UV_VERSION }}` 会展开为空，作业会静默安装最新的 uv。

要在推送前检查你的工作，或在移动固定值时找出所有仍需更新的文件：

```bash
bash scripts/release/verify-python-toolchain-pins.sh
```

它会列出每个不一致的文件和行。注意它*不*验证什么：固定版本是否正确。这来自一次绿色运行，因此 `python-toolchain.env` 记录了其值来源的那次运行——移动固定值时也要更新它。

**2. 锁文件是检查的，不是修复的。** Python 作业使用 `uv sync --locked` 安装，`lockfiles` 作业对每个第一方 `uv.lock` 运行 `uv lock --check`，每个安装依赖的 Python 作业以 `.github/actions/assert-lockfiles-unchanged` 结束，因此重写已提交锁文件的作业会失败，而不是隐藏漂移。如果 CI 报告锁文件漂移，在包目录中运行 `uv lock` 并提交结果。

唯一的例外是 `examples/` 应用：`prep-dojo-everything.js` 故意以非冻结方式同步它们（它与本地开发共享，本地开发需要重新锁定），因此它们的锁文件不在两项检查范围内。这就是为什么纯 `uv sync` 是示例应用的正确 prep 命令。

### 第 8 步（可选）：更新 CODEOWNERS

仅当你希望被添加为共同所有者、无需核心团队审查即可合并你的集成更改时才需要此步骤。如果适用，更新 `.github/CODEOWNERS` 文件，将自己与团队并列添加：

```
integrations/my-framework @ag-ui-protocol/copilotkit @your-github-username
```

对大多数贡献者来说这不是必需的——核心团队默认已拥有所有路径。

### 快速参考清单

提交前使用此清单验证你的 PR 是否完整：

- [ ] 在 `integrations/` 下添加了集成文件夹，含语言子文件夹 + 示例
- [ ] 包含 TypeScript 客户端文件夹（即使是非 TS 集成）
- [ ] `agents.ts` 已更新集成条目和功能映射（对象键是唯一事实来源）
- [ ] `menu.ts` 已更新侧边栏条目（`id` 匹配 `agents.ts` 键，`name` 人类可读）
- [ ] `env.ts` 已更新 agent URL 环境变量
- [ ] 示例代码绑定到 `0.0.0.0` 并遵循 `HOST`/`PORT` 环境变量
- [ ] 已添加 `prep-dojo-everything.js` 和 `run-dojo-everything.js` 条目（对象键匹配 `agents.ts`）
- [ ] `run-dojo-everything.js` 中的端口不与现有集成冲突
- [ ] `dojo`/`dojo-dev` 条目已更新，环境变量指向你服务的端口
- [ ] 为每个支持的功能添加了端到端测试 spec 文件
- [ ] 测试在本地通过
- [ ] `.github/workflows/dojo-e2e.yml` 中 CI 工作流矩阵已更新（条目名称匹配 `agents.ts`）
- [ ] **仅 Python：** 新的 CI 步骤使用与 `.github/python-toolchain.env` 匹配的 `${{ env.