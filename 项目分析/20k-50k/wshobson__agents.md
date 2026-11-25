# wshobson/agents 项目分析

> 星数 42481（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：目前规模最大、跨客户端支持最广的 AI 编码工具插件集合——94 个插件（含 202 个 agent、185 个 skill、105 个 command）以 Markdown 单源编写，由 Python 适配器生成器适配到 Claude Code、Codex CLI、Cursor、OpenCode、Antigravity CLI、GitHub Copilot、Pi 七个"harness"，并配有一套静态打分 + LLM 评审 + 蒙特卡洛的质量评测框架。

| 项 | 值 |
|---|---|
| 名称 | wshobson/agents |
| 星数 | 42481（快照）；fork 4487、watch 314（gh api，2026-11-14） |
| 语言 | Python（工具链）；主体内容为 Markdown |
| 许可证 | MIT（`LICENSE`，Copyright 2024 Seth Hobson） |
| 创建时间 | 2025-07-24（gh api `created_at`） |
| 最近推送 | 2026-11-09（gh api `pushed_at`） |
| 贡献者 | 91 人（gh api contributors 分页计数） |
| Release | 无正式 release（gh api releases 为 0），版本管理按插件粒度进行 |

## 2. 项目是什么，解决什么问题

2025 年以来各 AI 编码工具（Claude Code、Codex CLI、Cursor、OpenCode 等）各自形成了一套"插件/子代理/技能/斜杠命令"生态，但格式互不兼容：Claude Code 读 YAML frontmatter 的 Markdown agent，Codex CLI 的 agent 是 TOML 且 skill 正文限 8KB，OpenCode 用 `permission:` 块替代 `tools:` 白名单，Pi 靠扩展发现子代理。使用者若想在这些工具间复用同一套领域知识，只能手工维护多份副本。本仓库的解法是"写一次，多端生成"：贡献者只在 Claude Code 插件格式下维护一份 Markdown 源（`plugins/`），`tools/adapters/` 下的六个适配器按各 harness 的能力矩阵生成对应产物，CI 负责再生成、校验，甚至把 OpenCode/Antigravity/Pi 产物装进真实 CLI 做冒烟验证（`README.md`、`ARCHITECTURE.md`）。

内容本身是"领域专家知识库"：每个插件聚合一组子代理（如 `python-development` 的 `python-pro.md`、`fastapi-pro.md`、`django-pro.md`）、技能（如 `python-testing-patterns/SKILL.md`，含"何时使用"、核心概念、最佳实践）与脚手架命令（如 `/python-development:python-scaffold`，见 `plugins/python-development/`）。agent 前置元数据声明模型档位（`opus`/`sonnet`/`haiku`/`fable`/`inherit`），适配器在生成时映射为各 harness 的具体模型 ID（`tools/adapters/capabilities.py`）。

仓库刻意遵循 OpenAI 公开的 harness-engineering 实践（`ARCHITECTURE.md` 明确引用）：单一事实源、上下文文件限 ~150 行、skill 正文限 8KB 的渐进披露、每个 lint 发现都附带修复提示。这使它不只是"一堆 prompt"，而是一个带工程纪律的内容发行系统：生成物一律 gitignore（仅指向源清单的注册表入库），文档漂移由 `doc_gardener.py` 定期扫描，README 中的条目计数若与实际不符会在 CI 报错。

值得注意的边界：LLM 评审与蒙特卡洛两层评测被作者明确标注为实验性、未与人工标注对齐（`README.md`、`evals/README.md`）——作者对"评测分数不等于任务真的成功"有清醒的自我声明，这在同类项目中少见。

## 3. 技术栈与依赖

- **主体内容**：Markdown + YAML frontmatter（94 个插件、202 agents、185 skills、105 commands 均为纯文本）
- **工具链语言**：Python 3.12+，标准库为主——适配器框架零第三方依赖（自研容错 YAML 解析器，`tools/adapters/base.py` 的 `parse_frontmatter`）；仅 `doc_gardener.py` 引入 `yaml`、`validate_generated.py` 用 `tomllib`
- **评测子项目**：`plugins/plugin-eval/` 是独立 Python 包，含 `pyproject.toml` 与 `uv.lock`，可选 `llm` 依赖组用于 LLM 评审层
- **运行入口**：GNU Make（`Makefile`，约 30 个目标）；安装器需要 `uv`
- **CI**：GitHub Actions 6 个工作流（`.github/workflows/`：validate、code-quality、eval-report、claude-code-review、claude、mlops）
- **外部工具依赖（仅 CI/冒烟）**：OpenCode、Antigravity CLI（`agy`）、Pi CLI、Node（npx skills）、GitHub CLI 2.90+（`gh skill`）

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `plugins/` | 事实源：92 个本地插件，每个含 `.claude-plugin/plugin.json`、`agents/`、`skills/<n>/SKILL.md`、`commands/` |
| `plugins/plugin-eval/` | 评测工具本身也是一个插件，内含完整 Python 包（`src/plugin_eval/`）与自己的测试 |
| `tools/adapters/` | 适配器框架：`base.py`（ABC + 解析）、`capabilities.py`（能力矩阵）、`codex/cursor/opencode/copilot/antigravity/pi.py` 六个适配器、`cursor_rules/` 手工规则 |
| `tools/` | `generate.py`（统一生成 CLI）、`validate_generated.py`（结构校验）、`doc_gardener.py`（漂移检测）、`install_*.py` 四个全局安装器、`check_version_bumps.py` 等 |
| `tools/tests/` | pytest 套件：适配器、校验器、gardener、安装器、round-trip、真实源文件、CLI 冒烟 |
| `evals/` | `static-score-snapshot.json` 全量 skill 静态分快照（按内容 sha256 键控）+ 说明 |
| `docs/` | 10 篇参考文档：`harnesses.md`（跨端能力矩阵）、`authoring.md`（写作规范）、`plugin-eval.md`（评测指南）、`plugins.md`（目录）等 |
| `.claude-plugin/` `.agents/` `.cursor-plugin/` | 三份入库的原生注册表（指向 `plugins/` 源） |
| 根 | `AGENTS.md`（唯一上下文文件）、`ARCHITECTURE.md`、`CONTRIBUTING.md`、`Makefile` |

组织方式评述：这是"内容单源 + 代码工具链"的双层 monorepo——内容层按插件目录扁平组织（92 个目录、命名小写连字符），代码层约 5800 行 Python（工具链）+ 2400 行（plugin-eval），全部服务于内容的生成与质检。生成产物不进版本库是关键设计（`ARCHITECTURE.md` 不变量 1）。

## 5. 核心架构与关键实现

### 5.1 能力矩阵驱动的适配器框架

`tools/adapters/capabilities.py` 用一个冻结 dataclass 描述每个 harness：是否原生支持 skill/agent/command/marketplace、工具名大小写约定（Claude 用 CamelCase，OpenCode/Copilot 用 lowercase）、skill 正文字节上限（Codex 硬性 8KB，其余不限）、上下文文件行数上限（150）等。所有适配器共享这一矩阵，实现"优雅降级"：不支持命令的 harness（Codex、Copilot）把 command 转成 skill 或提示模板，不支持 per-agent 工具白名单的（Cursor）丢弃该字段。框架核心是 ABC：

```python
class HarnessAdapter(ABC):
    harness_id: str = ""          # 对应 capabilities.CAPABILITIES 键
    @abstractmethod
    def emit_plugin(self, plugin: PluginSource) -> EmitResult: ...
    def emit_global(self, plugins: list[PluginSource]) -> EmitResult:
        return EmitResult()       # 注册表、上下文文件等跨插件产物
```
（`tools/adapters/base.py:507-538`）

### 5.2 Codex 适配器：最重的一处格式翻译

`tools/adapters/codex.py`（601 行，六个适配器中最长）要处理三件事：Markdown YAML → TOML 的手写转换（stdlib 无 TOML writer，`_toml_kv`/`_escape_toml_multiline`）；8KB 正文硬上限的感知 fence 切分——超限时把溢出内容写进 `references/` 文件并在正文中留引用（`_split_body_if_oversized`，第 195 行起）；agent 的 `sandbox_mode` 启发式推断。生成的 `.codex/` 产物 gitignore，但 marketplace 注册表入库（point-at-source 设计）。

### 5.3 OpenCode 适配器：权限模型翻译

`tools/adapters/opencode.py` 把 Claude 的 `tools:` 白名单反向翻译成 OpenCode 的 `permission:` 拒绝式块（第 88-124 行的 `_build_permission_block` 有完整的映射表与四种边界情形说明），并处理 skill 名的 OpenCode 合法性。这是"工具权限跨端语义转换"的代表性实现。

### 5.4 质量评测：三层打分 + 快照防漂移

`plugins/plugin-eval/src/plugin_eval/layers/static.py` 定义静态层 7 个子维度及权重：

```python
_SKILL_WEIGHTS = {
    "frontmatter_quality": 0.32,
    "orchestration_wiring": 0.23,
    "progressive_disclosure": 0.14,
    "structural_completeness": 0.10,
    "token_efficiency": 0.09,
    "ecosystem_coherence": 0.06,
    "harness_portability": 0.06,
}
```
（`plugins/plugin-eval/src/plugin_eval/layers/static.py:12-21`）

纯 Python、确定性、<2 秒；上层是实验性的 `judge.py`（LLM 评审）与 `monte_carlo.py`（多次模拟）。`evals/static-score-snapshot.json` 给每个 skill 存静态分 + 内容 sha256，`make test` 里的两个测试保证"评分代码没变则分数不能变"与"快照不过期"（`evals/README.md`）。`tools/doc_gardener.py` 另做 9 类漂移检测（过期产物、超长上下文、死链、超限 skill、marketplace 孤儿、README 计数失真、同名 agent 正文分叉等），每个发现附 `Fix:` 修复行。

## 6. 构建与运行

来自 `Makefile` 与 `README.md` 的实际可用命令：

```bash
# 工具链（需 Python 3.12+ / uv）
make generate-all        # 为全部 harness 再生成产物
make validate STRICT=1   # 结构校验（错误阻断，STRICT 下警告也阻断）
make garden              # 文档/产物漂移检测
make test                # pytest 套件（含静态分快照一致性）
make lint / make format  # ruff 风格检查与格式化
make smoke-test          # 真实 CLI 冒烟（需安装 OpenCode/Antigravity/Pi）

# 单 skill 静态评分
uv run --project plugins/plugin-eval plugin-eval score \
  plugins/python-development/skills/python-testing-patterns --depth quick

# 按客户端安装（clone 后）
make install-opencode / install-antigravity / install-copilot / install-pi

# Claude Code 内直接用
/plugin marketplace add wshobson/agents
/plugin install python-development@claude-code-workflows

# 仅装 skill（不 clone）
gh skill install wshobson/agents python-testing-patterns
npx skills add wshobson/agents --skill python-testing-patterns
```

## 7. 收录范围、分类体系与条目规模

**条目规模**（全部经本地 find 实测，与 `README.md` "What's inside" 表完全一致）：94 个 marketplace 插件 = 92 个本地插件（`ls plugins | wc -l`）+ 2 个外部条目（pensyve、deja-vu，见 `.claude-plugin/marketplace.json`，实测 94 条）；本地源含 202 个 agent（`find plugins -path '*/agents/*.md'`）、185 个 skill（`find plugins -name SKILL.md`）、105 个 command（`find plugins -path '*/commands/*.md'`）。

**分类体系**：`docs/plugins.md` 按领域分 26 类，横跨工程全生命周期——Development(6)、Documentation(5)、Workflows(8)、Testing(1)、Quality(4)、Utilities(5)、AI & ML(6)、Data(2)、Database(2)、Operations(4)、Performance(2)、Infrastructure(5)、Security(6)、Governance(3)、Modernization(2)、API(2)、Marketing(5)、Business(5)、Languages(10)、Blockchain(1)、Finance(1)、Payments(1)、Gaming(1)、Accessibility(1)、Creative(3) 及外部 Memory(2)。覆盖面从纯编码（python-development、javascript-typescript）延伸到营销、支付、无障碍等非编码领域，是同类集合中最"泛"的。

**组织方式**：插件内三件套（agents/commands/skills）按固定目录约定自动发现（`ARCHITECTURE.md` "Plugin component model"）；skill 支持渐进披露（`references/` 放详情、`assets/` 放模板）；agent 前置元数据带四档模型层级（Opus=架构/安全/评审，inherit=复杂任务，Sonnet=文档/测试，Haiku=快速运维，`ARCHITECTURE.md` "Model tiers"）。

## 8. 更新机制与质量评测体系

**贡献流程**（`CONTRIBUTING.md`）：新建插件 → 按约定写三件套 → 更新 marketplace.json → `make generate-all` 刷新入库注册表 → `make validate` + `make garden` 自检。任何 `plugins/` 文件改动必须对插件版本号做 patch bump（`.claude-plugin/plugin.json` 与 marketplace.json 双处），由 `tools/check_version_bumps.py` 强制。命名禁用 `__`（适配器命名空间分隔符）。

**CI 门禁**（`.github/workflows/validate.yml` + `ARCHITECTURE.md`）：每个 PR 跑 JSON 校验、marketplace 条目解析校验、`make validate`、`make garden`、`make test`；另有 cli-smoke-test job 真实安装 OpenCode、Antigravity CLI、Pi 并对生成产物运行其 CLI，同时用 `gh skill` 与 `npx skills` 抽测源 skill，`gh skill publish --dry-run` 充当 agentskills.io 规范门。另有 eval-report.yml 输出评测报告。

**评测体系**：静态层（确定性、可快照防回归）+ LLM 评审层 + 蒙特卡洛层（两者实验性、明示未对齐人工标注）；trace-based 评测在进行中，方法论遵循 Hamel Husain / Shreya Shankar 的"先做错误分析再信任 judge"路径（`evals/README.md`）。测试代码约 6900 行（`tools/tests/` 13 个文件）+ 2400 行（plugin-eval 自有测试），含 round-trip（生成→回读→比对）与 real-world 测试。

**维护状态**：2025-07 创建至 2026-11-09 仍在活跃推送（gh api）；91 位贡献者；文档中的计数、矩阵与实际文件经本次抽查完全一致，说明 gardener 机制在起作用。

## 9. 亮点与风险

**亮点**
1. "单源多端生成"工程化到少见的高度：能力矩阵是唯一事实源，六个适配器消费同一矩阵优雅降级，生成物全部 gitignore、注册表只指向源，杜绝双源漂移（`tools/adapters/capabilities.py`、`ARCHITECTURE.md`）。
2. 评测自省诚实：明确声明 LLM judge 与蒙特卡洛未对齐人工标注、分数不代表真实成功率（`README.md`、`evals/README.md`），并用内容哈希快照机制把"评分代码变更"与"内容变更"解耦。
3. CI 真实闭环：不只静态校验，还把三个 harness 的真实 CLI 装进 CI 跑冒烟，`gh skill publish --dry-run` 把 agentskills.io 规范也纳入门禁。
4. 每个 lint 发现都带 `Fix:` 修复串（harness-engineering 实践），贡献者不必逆向理解各端格式。
5. 条目规模与文档计数经实测完全一致（202/185/105/94），多年维护未见腐烂。

**风险/局限**
1. 内容质量本质上是"prompt 工程"，静态分权重 32% 给 frontmatter 质量，无法证明某个 agent 真能提升编码效果；评测层的自我声明恰恰说明该问题尚无解。
2. 强耦合快速演进的私有格式：七个 harness 任一改动格式（如 Codex 的 8KB 限制、Pi 的扩展要求）都需要同步改适配器，维护成本随 harness 数线性增长。
3. 无 release、无版本化分发（按插件 patch bump 粒度管理），对下游工具链的可用性依赖 marketplace 拉取时刻的仓库状态。
4. 部分 skill 内容具有时效性（如 skill 文内"2024's fastest package manager"这类表述，`plugins/python-development/skills/uv-package-manager/`），长尾内容更新依赖贡献者。

## 10. 总结评价：生态位与增长逻辑

wshobson/agents 是 2025-2026 年 AI 编码插件生态繁荣期最成功的社区集合：它起于 Claude Code 子代理集合（创建于 2025-07，恰逢 Claude Code 插件体系成型），在 42.5k 星的驱动下逐步演化为"Markdown 单源 + 七端适配 + CI 实测 + 分层评测"的内容发行系统。与 anthropics/claude-plugins-official 这类单厂商官方市场相比（后者只服务单一 harness、由官方策展），本仓库的价值主张是跨端中立与规模：94 个插件、492 个组件、26 个领域，任何主流 AI 编码工具都能消费同一份源；与一般 awesome 清单相比，它多了真正的生成/校验/评测工具链，内容不是链接列表而是可直接安装的组件。增长逻辑清晰：踩中"每个 AI 编码工具都要建插件生态但互不兼容"的真实痛点，用工程纪律（而非仅靠 prompt 数量）建立信任，再借 2026 年 agent/skill 标准化浪潮（agentskills.io、AGENTS.md 规范、OpenAI harness-engineering 实践）完成从"prompt 合集"到"基础设施"的叙事升级。

**适合谁用**：同时使用多个 AI 编码工具、想要一套可维护的领域专家配置的个人与团队；想给自家 harness 做插件兼容的厂商（能力矩阵是现成的需求清单）。**适合谁读源码**：想学习"内容即代码"流水线（生成→校验→评测→快照防漂移）的工程师——`tools/adapters/base.py`、`doc_gardener.py` 与 `plugins/plugin-eval/src/plugin_eval/` 是三个教科书级模块；设计文档 `ARCHITECTURE.md` 与 `docs/architecture.md` 本身就值得作为 harness-engineering 的实践样本阅读。
