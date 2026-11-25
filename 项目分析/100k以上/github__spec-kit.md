# github/spec-kit 项目分析

> 星数 142546（快照 2026-10-24）｜主语言 Python｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

GitHub 官方出品的"规格驱动开发"（Spec-Driven Development, SDD）工具包：把 AI 编码 agent 的"先写提示词再祈祷"变成"先定规格，再生成计划与代码"的结构化流程。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | github/spec-kit | GitHub API |
| 星数 | 143,894（查询时实时值） | `gh api repos/github/spec-kit` |
| 语言 | Python（CLI）+ Markdown/模板资产 | `pyproject.toml` |
| 许可证 | MIT | `LICENSE`、GitHub API |
| 创建时间 | 2025-08-21 | GitHub API |
| 最近 push | 2026-10-29 | GitHub API |
| 最新版本 | v1.2.5（2026-10-28），代码版本 1.2.6.dev0 | `CHANGELOG.md`、`pyproject.toml` |
| CLI 入口 | `specify`（PyPI 包名 specify-cli） | `pyproject.toml` |

## 2. 项目是什么，解决什么问题

Spec Kit 解决的核心问题是：AI 编码 agent 能力很强，但缺乏"意图—规格—计划—实现—验证"的过程留痕，导致结果不可预测、团队经验无法沉淀（`README.md`、`spec-driven.md`）。它提供三套开箱即用的流程，外加一套可组合的扩展机制：

- **SDD（核心内置）**：`/speckit-constitution` 定项目宪法 → `specify` 写规格 → `plan` 定技术方案 → `tasks` 拆任务 → `implement` 实现 → `converge` 收敛，循环直到报告 Converged（`README.md` SDD 流程节、`workflows/speckit/workflow.yml`）。
- **Bug 修复**（`extensions/bug/`）：assess → fix → test 三段式，产出落在 `.specify/bugs/<slug>/`，最终结论只有 verified 才算修复成功（`extensions/bug/extension.yml`）。
- **Idea 评估**（`extensions/assess/`）：intake → research → define → shape → decide，产出 go / needs-clarification / kill 决策，允许"有据可查地放弃"。

与纯提示词集合不同，Spec Kit 把流程做成了**可安装、可审计的软件资产**：规格模板强制用户故事按 P1/P2/P3 排序、每条故事必须独立可测试并给出 Given/When/Then 验收场景（`templates/spec-template.md`）。这使得规格不是形式文档，而是后续 plan/tasks/implement 各阶段的机器可核对输入。

项目的另一条主线是"与 agent 无关"：同一套命令（speckit.specify、speckit.plan…）通过集成层适配到 40+ 个编码 agent（Claude Code、Copilot、Gemini CLI、Cursor 等），命令文件按各 agent 要求写入不同目录与格式。工具本身还内置 MCP server，可被 agent 作为工具调用（`src/specify_cli/mcp_server/server.py`）。

## 3. 技术栈与依赖

全部出自 `pyproject.toml`：

- **语言**：Python ≥3.11，构建后端 hatchling。
- **CLI 框架**：typer + click（命令行）、rich（终端美化）、readchar（交互选择）。
- **数据与配置**：pyyaml（extension/workflow YAML）、pydantic（模型校验）、json5、packaging（版本约束）、pathspec（ignore 规则）。
- **MCP**：`mcp>=2.2.0`，用于内置 MCP stdio server。
- **质量工具**：ruff（含 S602/S604/S605 强制禁止 shell=True 的安全规则锁）、pytest + pytest-cov + pytest-xdist；CI 用 uv 管理（`.github/workflows/test.yml`：Python 3.13/3.14 矩阵，`uv run pytest -n auto`）。
- **资产打包**：通过 `force-include` 把 templates/scripts/extensions/presets/workflows 打进 wheel 的 `core_pack/`，使 `specify init` 离线可用（气隙/企业环境，`pyproject.toml` 30-59 行）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `src/specify_cli/` | specify CLI 全部 Python 源码（293 个文件，约 7.8 万行） |
| `templates/` | SDD 核心模板：spec / plan / tasks / constitution / checklist + 命令模板 |
| `scripts/` | 工作流支撑脚本，bash / powershell / python 三实现并列且功能对齐 |
| `extensions/` | 一方扩展（bug、assess、git、agent-context、github、template）与社区目录 |
| `presets/` | 预设（lean、constitution-sync 等），用于调整模板/命令/术语 |
| `workflows/` | 可自动化工作流定义（speckit、bugfix、assess 的 workflow.yml） |
| `bundles/` | 打包的角色化安装集合与目录快照（catalog.json、catalog.community.json） |
| `docs/` | GitHub Pages 文档站源（github.github.io/spec-kit） |
| `tests/` | 363 个测试文件，约 17.9 万行 |
| `design/`、`examples/`、`integrations/`、`media/`、`newsletters/` | 设计文档、示例、集成示例、品牌与通讯素材 |

组织方式评述：单仓多组件（monorepo 式资产仓）。CLI 是主体；模板/脚本/扩展/预设/工作流都是随 wheel 一起发布的"数据资产"。最大的复杂度集中在 CLI 内部：`integrations/`（70 文件）、`workflows/`（82 文件）、`bundles/`（36 文件）、`presets/`（24 文件）四个子系统并列，共享 `artifacts/`（产物清单）与 `events/`（生命周期事件）基础设施。

## 5. 核心架构与关键实现

### 5.1 多 agent 集成层（最大亮点）

`src/specify_cli/integrations/base.py`（约 2000 行）定义 `IntegrationBase` 抽象基类：每个 agent 集成声明自己的注册配置（命令写入目录、格式 markdown/TOML、参数占位符、文件扩展名）、执行参数构造（`build_exec_args`）、命令分发（`dispatch_command`）、模板加工（`process_template`）与生命周期事件支持（`setup/teardown/emit_events`）。`integrations/` 下有 44 个集成目录（含 generic/none），覆盖 claude、copilot、codex、gemini、cursor_agent、qwen、kimi、trae、if 绝大多数主流工具。所有集成注册进单一事实源 `INTEGRATION_REGISTRY`（`integrations/__init__.py:30`），`agents.py` 的 `CommandRegistrar` 从中派生 AGENT_CONFIGS，避免多处硬编码。

以 Claude Code 为例（`integrations/claude/__init__.py`）：命令作为 skills 写入 `.claude/skills/`、格式 markdown、占位符 `$ARGUMENTS`；事件模型映射到 Claude 原生钩子（SessionStart、PreToolUse…）。值得注意的是 `FORK_CONTEXT_COMMANDS` 为空字典并附注释：曾经让 `/speckit-analyze` 在 fork 子代理里跑，实测发现 300-500 行报告注入回主会话导致长会话卡死（issue #3185），于是回退——体现对真实 agent 上下文成本的持续调优。

### 5.2 工作流引擎

`src/specify_cli/workflows/engine.py`（约 1500 行）加载并执行 YAML 定义的工作流：解析 → 校验 → 顺序执行步骤（dispatch 到步骤类型）→ 状态持久化支持断点续跑 → 控制流（分支、循环、fan-out/fan-in）。`workflows/speckit/workflow.yml` 是自举样例：input 带 prompt 交互、`{{ inputs.integration }}` 插值、`type: gate` 步骤强制人工 approve/reject 才能继续，并通过 `requires.speckit_version` 做版本兼容声明。

### 5.3 扩展/预设/包三层分发体系

- **Extension**（`extensions/__init__.py`，单文件 6238 行）：声明式 `extension.yml`（id、version、provides.commands、requires.speckit_version），安装时按集成把命令文件写到各 agent 目录；支持 hooks（在核心命令前后插入）、运行时事件、`.extensionignore`。
- **Preset**（`presets/`）：不改扩展即可换模板/命令/术语，支持项目本地覆盖与解析顺序（README Capabilities 表）。
- **Bundle**（`bundles/`，36 文件）：把扩展+预设+工作流+自定义步骤打包成角色化安装包，支持 bundle 引用 bundle（CHANGELOG #5067）与事务化回滚（#4827）。
- **Catalog**：内置 `catalog.json` + `catalog.community.json` 两个快照供离线发现，`specify extension search` 等命令检索；README 明确声明目录条目不是审计背书。

### 5.4 跨平台脚本与安全工程

`scripts/` 下同一套功能（check-prerequisites、create-new-feature、setup-plan、setup-tasks、resolve-template）提供 bash/powershell/python 三份实现，并用 `test_*_python_parity.py` 系列测试保证行为一致（tests/ 下 6 个 parity 测试）。安全上有两个显眼投入：`_download_security.py`（1275 行）处理下载资产安全——归档格式嗅探、SSRF 防护（环回/本地地址与重定向目标检测）；ruff 配置锁定 S602/S604/S605，任何 shell=True 回归必须显式 noqa 才能过审（`pyproject.toml` 88-97 行）。

## 6. 构建与运行

```bash
# 用户安装（README.md）
uv tool install specify-cli
specify init my-project --integration copilot

# 开发（.github/workflows/test.yml）
uv sync --extra test          # 安装依赖与测试扩展
uvx ruff@0.15.0 check src tests
uv run pytest -n auto          # 并行跑全量测试
specify --help                 # Typer CLI，入口 specify_cli:main
```

未提供 Makefile；发布走 `.github/workflows/publish-pypi.yml` + `release-trigger.yml`（`.github/workflows/RELEASE-PROCESS.md` 有完整流程文档）。

## 7. 测试与工程质量

- **规模**：363 个测试文件、约 17.9 万行、7144 个测试函数（对 tests/ 目录 grep 统计），测试代码规模超过被测 CLI 本体。
- **分层**：`tests/specify_cli/`（209 文件，单元为主）、`tests/contract/`（9，契约）、`tests/workflows/`（13）、`tests/extensions/`（16）、`tests/integration/`、`tests/hooks/`；另有大量专项安全测试（`test_download_security.py`、`test_registrar_path_traversal.py`）。
- **特色测试**：bash/powershell/python 脚本 parity 测试（如 `test_setup_plan_python_parity.py`），防止三平台实现漂移——对一个靠脚本驱动的工具这是关键保障。
- **CI**：`test.yml`（ruff + 3.13/3.14 矩阵 pytest）、`codeql.yml`、`security.yml`、`lint.yml`、`docs.yml`；`.github/workflows/` 里还有 add-community-extension/preset/bundle 等自动化工作流，社区组件入库走受控流程。
- **工程文化**：CONTRIBUTING.md 明确回答"Does Spec Kit use Spec Kit? Yes"——官方团队自举/dogfooding，feature-assess 工作流是自动化路径。

## 8. 社区与维护状态

- **极度活跃**：2026-10-16 至 10-28 连发 v1.2.1→v1.2.5 五个版本，几乎日更（`CHANGELOG.md`）；最近 push 2026-10-29。
- **官方背书**：github 组织旗舰项目；README 有 Open Source Friday 视频两期（2026-05 与 2026-10，讲解 Spec Kit 1.0）。
- **生态成型**：CHANGELOG 每个版本都收录多个第三方扩展/预设进社区目录（Salesforce SFSpeckit、CSDD、Linear Weave、Domain Spec Maintenance 等），社区已从"使用者"变成"供给方"。
- **文档四语种**：README 英/中/日/韩（README.zh-CN.md 等），星数增长有国际化助推。
- 开放 issue 219（GitHub API）；贡献者结构未确认（tarball 无 git 历史，未做 contributors API 拉取）。

## 9. 亮点与风险

**亮点**

1. 多 agent 适配做成了一等公民：44 个集成、单一注册表、统一的命令注册/事件/hook 模型（`src/specify_cli/integrations/`），是同类项目里最完整的 agent 兼容层。
2. 方法论产品化：constitution→spec→plan→tasks→implement→converge 不只是提示词，而是带 gate、状态持久化、断点续跑的工作流引擎（`workflows/engine.py`）。
3. 资产可离线：wheel 强制打包 core_pack，`specify init` 气隙环境可用（`pyproject.toml` force-include 注释）。
4. 工程质量高：17.9 万行测试、跨平台 parity 测试、SSRF 防护、ruff 安全规则锁、CodeQL。
5. 自我闭环：官方 dogfooding + MCP server + 自举工作流（`.github/workflows/` 里的 speckit 系工作流）。

**风险/局限**

1. 复杂度膨胀明显：`extensions/__init__.py` 单文件 6238 行、CLI 近 8 万行，extension/preset/bundle/workflow 四套机制的学习曲线陡峭（`src/specify_cli/` 结构即证）。
2. 强绑定"每个 agent 一个集成"的维护面：agent 生态迭代快，44 个集成长期跟进成本高，个别长尾集成（dsh、mcode、hermes 等）质量未确认。
3. 流程开销：SDD 全流程对小型改动偏重；README 虽提供 lean preset 缓解，但"先写规格"的采纳门槛仍在。
4. 仓库创建仅一年多、版本仍处于 1.x 快速演进期，模板与命令接口可能持续破坏性变更（CHANGELOG 修复密度高）。

## 10. 总结评价

Spec Kit 是 GitHub 用官方资源把"AI 编码该有工程纪律"这件事做成的开源基础设施：以 8 万行 Python CLI 为骨架，以模板/脚本/扩展/预设/工作流五类资产为血肉，通过 44 个 agent 集成把规格驱动开发铺到几乎所有主流编码工具上。它大概率定义了"spec-driven development"这一品类的默认范式，社区目录生态也已起飞。适合：想让 AI 编码从"玩具 demo"走向团队工程化的开发者与团队负责人（用）；想学习如何设计 agent 无关的提示词资产分发体系、或研究 agent 生命周期事件/工作流引擎实现的人（读源码，重点读 `integrations/base.py`、`workflows/engine.py` 与 `templates/spec-template.md`）。不适合：只想要轻量提示词模板的用户——对这类人，直接抄 `templates/` 下的 Markdown 即可，无需安装整套 CLI。
