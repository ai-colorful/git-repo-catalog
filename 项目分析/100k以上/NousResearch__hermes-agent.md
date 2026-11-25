# NousResearch/hermes-agent 项目分析

> 星数 256443（快照 2026-10-24）｜主语言 Python｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：Nous Research 出品的"自我改进型"个人 AI Agent —— 内置学习闭环（自主创建/改进技能、自管理记忆、跨会话搜索），同一套 agent 内核可跑在 CLI、TUI、消息网关（Telegram/Discord 等 28 个平台）与 Electron 桌面端上。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | hermes-agent | `README.md` |
| 星数 | 256,443（快照） | 任务参数 |
| 主语言 | Python（核心）+ TypeScript（TUI/桌面/Web） | `pyproject.toml`、`package.json` |
| 许可证 | MIT（Copyright (c) 2025 Nous Research） | `LICENSE` |
| 创建时间 | 2025-07-22 | gh api repos/NousResearch/hermes-agent |
| 最近 push | 2026-10-29（分析前一天） | gh api |
| 贡献者 | ~397 人（contributors 分页 397 页） | gh api contributors |
| 规模 | Python 源码约 105 万行（另测试约 156 万行）；TS/TSX 5,247 个文件 | 仓库统计（find/wc） |

## 2. 项目是什么，解决什么问题

Hermes Agent 自我定位是 "The self-improving AI agent"（`README.md`）。它解决的问题是：通用 agent 工具（如 Claude Code 类产品）在会话结束后"什么都记不住"，每次都从零开始。Hermes 把三件事做成了一等公民（`README.md` 特性表、`AGENTS.md` "What Hermes Is"）：

1. **学习闭环**：agent 在解决一个复杂问题后自主把方法保存为 skill（过程性记忆）；每次会话后台审查可能把经验写入 MEMORY.md/USER.md；还能 FTS5 全文搜索自己的历史会话做跨会话回忆。
2. **随处运行**：同一内核经网关接入 Telegram、Discord、Slack、WhatsApp、Signal、微信、飞书、钉钉、QQ 等 28 个平台插件（`plugins/platforms/` 目录）；终端后端六种 —— local、Docker、SSH、Modal、Daytona、Vercel Sandbox（`tools/environments/`），可跑在 $5 VPS 或 GPU 集群上，Serverless 后端闲置时休眠近乎零成本。
3. **模型无关**：内置 39 个 model-provider 插件（`plugins/model-providers/`：OpenAI、Anthropic、Gemini、OpenRouter、DeepSeek、Ollama 等），`hermes model` 一条命令切换，无锁定。

它同时面向研究场景：批量轨迹生成（`batch_runner.py`）与轨迹压缩（`trajectory_compressor.py`）用于训练下一代工具调用模型（`README.md` "Research-ready"）。

值得注意的社区语境：README 有"从 OpenClaw 迁移"专章（`README.md` §Migrating），支持导入 OpenClaw/Claude Code/Codex/Cursor 的记忆、技能与配置 —— 明显定位于承接 OpenClaw 类"个人 agent"生态的用户。作者 Nous Research 是开源模型社区（Hermes 系列模型）的发起方，星数与该品牌社区驱动强相关。

## 3. 技术栈与依赖

- **Python 3.11–3.14**（新装仅支持 3.14，`pyproject.toml` requires-python 注释），核心依赖全部**精确锁版**（`==X.Y.Z`），注释明确写明是 2026-05 应对 PyPI "Mini Shai-Hulud" 供应链蠕虫而收紧（`pyproject.toml` 头部注释）。
- 关键依赖（`pyproject.toml` dependencies）：`openai`（SDK 与消息格式基准）、`httpx`、`rich`、`prompt_toolkit`（TUI 输入）、`croniter`（定时任务）、`pydantic`、`fire`、`ruamel.yaml`、`cryptography`/`PyJWT`（Skills Hub 认证）、`firecrawl-anydoc`（PDF/Office 转 Markdown 供 read_file 用）。
- **Node/TypeScript 工作区**（`package.json` workspaces）：`apps/desktop`（Electron 桌面端）、`ui-tui`（Node 终端 UI）、`web`、`tests-js`；engines 要求 Node >= 26。
- **存储**：SQLite —— 会话持久化 + WAL + FTS5 全文检索（`hermes_state_schema.py` import sqlite3、`hermes_state_fts.py`、`hermes_state_wal.py`）。
- **工具链**：uv/PM（自研包管理器，`hermes pm`）、ruff（`ruff.toml`）、lefthook（`lefthook.yml`）、just（`justfile`）、Nix flake（`flake.nix`）、Docker（`Dockerfile` 34KB）。

## 4. 目录结构与代码组织

顶层为"扁平模块 + 包"混合的单体仓库（无 monorepo 工具，但 npm workspaces 管理多个前端应用）：

| 目录/文件 | 职责 | 规模（py 文件/行） |
|---|---|---|
| `run_agent.py` | AIAgent 公共门面（13 个 mixin 组装） | 88KB 单文件 |
| `agent/` | agent 内核：turn 循环、provider 适配、压缩、记忆、子代理 | 410 文件 / 14.9 万行 |
| `gateway/` | 消息网关：平台接入、会话调度、流式分发、cron | 340 文件 / 12.7 万行 |
| `tools/` | 40+ 模型工具：终端、浏览器、文件、MCP、委派、TTS 等 | 366 文件 / 12.9 万行 |
| `plugins/` | 插件：28 平台、39 模型 provider、memory、browser、kanban 等 | 302 文件 / 9.2 万行 |
| `hermes_cli/` | CLI 实现 + PM 包管理器 | 809 文件 / 30.8 万行 |
| `hermes_state_*.py`（约 50 个顶层文件） | SQLite 会话状态层：schema、WAL、FTS、压缩、修复、变更 | 约 130 万行中占 ~90 万行（含 `hermes_state_messages.py` 131KB 等） |
| `cli.py` / `hermes_bootstrap.py` / `hermes_constants.py` | REPL 入口 / 引导 / 常量 | 82KB / 29KB / 67KB 单文件 |
| `skills/` | 内置技能库（14 类目录） | 50 文件 / 1.2 万行 |
| `evals/`、`batch_runner.py`、`trajectory_compressor.py` | 评测与训练数据管线 | 156 文件 / 2.2 万行 |
| `apps/`、`ui-tui/`、`web/`、`tui_gateway/` | Electron 桌面、Node TUI、Web UI | TS 为主 |
| `tests/`、`tests-js/` | 测试 | 7,477 文件 / 156 万行 |

组织方式评述：核心特征是"**门面 + 海量小 sibling 模块**"——`agent/AGENTS.md` 明言"Each phase of an iteration is its own sibling, so a change to (say) overflow handling touches one ~600-line file"。同时存在大量超大单文件（`hermes_state_messages.py` 131KB、`cli.py` 82KB），与 AGENTS.md 提出的"god-file → module refactors 是 Wanted 贡献"相互印证：项目自己也认为还在持续拆分中。

## 5. 核心架构与关键实现

### 5.1 Agent 循环：同步 while 循环 + 每阶段一个 sibling 文件

`run_agent.py` 中 `AIAgent` 由 13 个 mixin 组装（ClientLifecycle、StreamDelivery、SessionPersistence、CompressionFacade、TurnFacade 等，见 `run_agent.py` class 定义）。一回合 = `agent/conversation_loop.py::run_conversation`，其骨架在 `agent/AGENTS.md` 中被如实描述为：

```python
while (api_call_count < self.max_iterations and self.iteration_budget.remaining > 0) \
        or self._budget_grace_call:
    if self._interrupt_requested: break
    response = client.chat.completions.create(model=model, messages=messages, tools=tool_schemas)
    if response.tool_calls:
        for tc in response.tool_calls:
            messages.append(tool_result_message(handle_function_call(tc.name, tc.args, task_id)))
    else:
        return response.content
```

真正的工程量在 `agent/turn_*.py` 约 40 个阶段文件里：preflight、request_assembly、api_call、api_error、response_intake、tool_round、overflow、truncation、recovery、stop_gates、usage、finalizer……每个 600 行级。消息采用 OpenAI chat 格式，并强制"严格角色交替"（`agent/AGENTS.md` "Message-flow invariants"）。

两条架构铁律（`AGENTS.md` 根文件，原文）：
- **"Per-conversation prompt caching is sacred"**——会话中途禁止任何会打破前缀缓存的改动（改系统提示、换工具集、重载记忆都禁止），唯一例外是上下文压缩。
- **"The core is a narrow waist; capability lives at the edges"**——核心工具准入门槛极高，新能力一律走 CLI 命令+skill、service-gated tool 或插件；AGENTS.md 给出五级"足迹阶梯"（footprint ladder）。

### 5.2 记忆与成长机制（本项目的招牌）

四层机制协同（出处均为对应源码/文档）：

1. **双文件记忆 + 冻结快照**：`~/.hermes/memories/MEMORY.md`（2,200 字符上限）与 `USER.md`（1,375 字符上限）在会话开始时一次性注入系统提示，会话中不再变动以保住提示缓存；agent 用 `memory` 工具（`tools/memory_tool.py`）增删改，超限时**报错而不自动压缩**，迫使 agent 自己腾空间（`website/docs/user-guide/features/memory.md`）。
2. **自主技能（过程性记忆）**：`skill_manage` 工具（`tools/skill_manager_tool.py`）让 agent 在"解决了值得复用的多步工作流/踩坑后找到正确路径/被用户纠正"时创建-修补-删除技能；写入时跑咨询型 linter（`tools/skill_linter.py`，规则如 incident-log-shape、references-sprawl）+ AST 安全审计（`tools/skills_ast_audit.py`）。`/learn` 命令（`agent/learn_prompt.py`）可把任意来源（代码目录、URL、聊天记录）炼成技能，内置 60 字符描述等 HARDLINE 写作规范。
3. **后台审查（nudge）**：每回合结束后 `run_conversation` 可能派生守护线程，在 fork 出的 AIAgent 里回放对话快照，自问"该保存或更新什么技能/记忆？"；fork 继承父进程 runtime 与提示缓存，写直落存储、不碰主对话（`agent/background_review.py` 模块 docstring）。
4. **Curator（技能园丁）**：`agent/curator.py` 按不活跃计时（默认 7 天间隔、空闲 2 小时触发）做确定性状态迁移 active→stale(14 天)→archived(30 天)，可选 LLM 整合 pass（合并重叠技能为 umbrella，opt-in），永不真删、pin 与 cron 引用的技能免疫（`website/docs/user-guide/features/curator.md`）。另有 `agent/learning_graph.py` 把技能+记忆节点组成"learning made visible"图谱供桌面端展示。

跨会话回忆由 FTS5 会话搜索（`tools/session_search_tool.py`、`hermes_state_fts.py`）+ LLM 摘要完成；外部记忆提供方走插件（`plugins/memory/`），内置支持 Honcho 辩证式用户建模（`README.md` 特性表）。

### 5.3 压缩：唯一被批准的缓存破坏点

上下文压缩分两层（`agent/AGENTS.md` §Compression）：网关会话卫生（85% 阈值）与 agent 内 `ContextCompressor`（50% 阈值，可按模型覆写）。算法先无 LLM 地修剪旧工具结果，再选边界、再用 auxiliary 模型生成结构化摘要；`agent/` 下有 15+ 个 compression/compaction sibling 文件（`context_compressor_*.py`、`conversation_compression_*.py`、`micro_compaction.py` 等）处理水位、检查点、降级与遥测。这是全项目唯一允许改写历史上下文的路径。

### 5.4 网关与委派

- 网关（`gateway/`）是独立进程：入站消息 → 会话准入/租约（`session_lease`、`turn_lease.py`）→ 复用或拉起 agent runtime → 流式回投平台。`gateway/platforms/` 是基类与 API server，具体平台在 `plugins/platforms/`（telegram、discord、slack、whatsapp、whatsapp_cloud、signal、matrix、teams、email、feishu、dingtalk、wecom、weixin、qqbot、irc、line 等 28 个）。
- 子代理委派：`delegate_task` 工具（`tools/delegate_tool.py` + 9 个 sibling）spawn 隔离上下文的子 AIAgent，仅最终摘要回传父上下文；后台运行返回 handle，完成后经可靠投递队列回发（`website/docs/user-guide/features/delegation.md`）。
- 提示缓存治理延伸到辅助任务：视觉、压缩、标题、curator 等 side-LLM 调用统一走 `agent/auxiliary_client.py` 的槽位路由，每槽可独立指定 provider/model。

## 6. 构建与运行

来自 `README.md` 与 `pyproject.toml`（未实际运行验证）：

```bash
# 安装（Linux/macOS/WSL2）
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
# Windows 原生
iex (irm https://hermes-agent.nousresearch.com/install.ps1)

hermes              # 交互式 CLI
hermes model        # 选择 provider/model
hermes gateway      # 启动消息网关
hermes setup        # 全量配置向导（含 --portal 一键接 Nous Portal）
hermes doctor       # 诊断

# 开发
source ./activate            # 配置 PM 环境（activate.ps1/fish 也有）
python scripts/check         # 跑全部 CI 阻断性 lint（AGENTS.md）
hermes curator run --dry-run # 预览技能维护
```

入口点：`hermes = hermes_cli.main:main`、`hermes-acp = acp_adapter.entry:main`（`pyproject.toml` [project.scripts]）。

## 7. 测试与工程质量

- **测试规模罕见**：`tests/` 7,477 个 Python 文件、约 156 万行（占全仓 Python 60%），另 `tests-js/` 21 个 TS 文件；覆盖单元、`tests/e2e/`、`tests/integration/`、`tests/compat/`（升级兼容契约）、`tests/conformance/`、`tests/perf_guards/` 等约 40 个子目录（`ls tests/`）。
- **CI 矩阵庞大**：`.github/workflows/` 共 56 个 workflow，包括 ci.yaml（588 行）、tests.yml、56 项中的 install-e2e（macOS/Windows 实装）、desktop-bundle-smoke、osv-scanner、supply-chain-audit、uv-lockfile-check、skills-index-freshness、prompt-surface-diff 等 —— 后几项体现"提示词/技能索引也进 CI"的工程化思路。
- **质量文化**：根 `AGENTS.md` 是给 AI 编码助手与人类贡献者的 hub，用字符预算（根 12k、根到叶子链 30k）约束加载体积；有明确的"贡献评估表"（Wanted/Rejected 清单）与安全上报分级（`SECURITY.md` 为范围权威）。依赖精确锁版 + 供应链审计 workflow（`pyproject.toml` 注释、`.github/workflows/supply-chain-audit.yml`）。

## 8. 社区与维护状态

- 高度活跃：创建于 2025-07-22，最近 push 2026-10-29（分析前一天，gh api）；约 397 名贡献者；56,748 forks、51,829 open issues —— issue 洪量巨大，项目为此写了专门的 triage 规则（`AGENTS.md` "triage sweeper" 只允许三类自动关闭，其余留给维护者）。
- 发布节奏：workflow 含 stable-release、stable-release-publication、canary-release（`ls .github/workflows/`），有 stable 与 canary 双通道（README Termux 段提及）。
- 文档站独立部署（`website/`，docs-search-monitor、docs-site-checks workflows），README 有中/西/乌尔都语版本；社区生态含 Skills Hub（agentskills.io）与第三方插件（HermesClaw 微信桥、computer-use-linux，`README.md` Community 节）。
- 贡献者结构细节（公司雇员 vs 社区比例）未确认（tarball 无 git 历史）。

## 9. 亮点与风险

**亮点**

1. 把"自我改进"做成可审计的工程闭环：linter+AST 审计守门的自写技能、nudge 后台审查、curator 园丁、永不真删、写审批门（`agent/background_review.py`、`website/docs/user-guide/features/skills.md`）。
2. 提示缓存即预算的架构纪律罕见地严格："prompt caching is sacred" 写进根 AGENTS.md 并作为每次变更的评审透镜（`AGENTS.md`）。
3. 测试投入巨大（156 万行测试 vs 105 万行源码）+ 56 个 CI workflow，覆盖安装 E2E、桌面打包、供应链审计。
4. 模型/平台/终端后端三维解耦：39 个 provider 插件、28 个平台插件、6 种终端后端，均插件化可换（`plugins/`、`tools/environments/`）。
5. 依赖供应链防御有真实事故叙事支撑（PyPI 蠕虫 → 全量精确锁版，`pyproject.toml`）。

**风险/局限**

1. 体量与复杂度极高（Python 200+ 万行、7,000+ 测试文件、51k open issues），个人贡献者理解成本大，issue 积压本身是维护压力信号。
2. 大量巨型单文件（`hermes_state_messages.py` 131KB、`cli.py` 82KB、`run_agent.py` 88KB）与 `_scratch` 文件（`hermes_constants_scratch*.py`）提示仍在高速演化中，接口稳定性存疑。
3. 强绑定自家生态： Nous Portal 订阅推销贯穿 README；技能/记忆格式虽宣称兼容 agentskills.io 开放标准（`README.md`），但工作流深度绑定 `~/.hermes` 目录约定。
4. Python 仅完整支持 3.14 的激进选择（`pyproject.toml` requires-python 注释），对系统集成环境不友好。

## 10. 总结评价

hermes-agent 是目前"个人 AI agent"赛道工程化程度最高的开源实现之一：它不追求 agent 框架的抽象优雅，而是把提示缓存、上下文压缩、记忆/技能生命周期、多平台投递、供应链安全这些真实运维问题逐一做成系统，并以根 AGENTS.md 为"宪法"让几百名人类与 AI 贡献者在其约束下协作。20 个月冲到 25 万+星，品牌社区（Nous Research）、"自我改进"叙事与开箱即用的多平台体验三者缺一不可。适合想研究生产级 agent 架构（尤其记忆/上下文管理）的工程师通读 `agent/AGENTS.md` + `agent/turn_*.py` + `agent/background_review.py`；适合想要"一个跑在 VPS 上、Telegram 里随时可找、越用越懂你"的私人助理的用户直接安装使用；不适合期待轻量库或稳定 API 的集成方——它是产品，不是 SDK。
