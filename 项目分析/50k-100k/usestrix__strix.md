# usestrix/strix 项目分析

> 星数 71341（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：开源自主 AI 渗透测试 agent——以"AI 黑客团队"的形式对授权目标做侦察、漏洞发现、PoC 实证与修复建议，输出可直接进 CI 的安全报告。

| 项目 | 内容 |
|---|---|
| 名称 | usestrix/strix（PyPI 包名 `strix-agent`，见 `pyproject.toml`） |
| 星数 | 71341（快照 2026-10-24；gh api 实测 72946，仍在快速增长） |
| 语言 | Python 3.12+（约 4.5 万行），辅以 Go（TUI）、TypeScript（本地查看器前端） |
| 许可证 | Apache-2.0（`LICENSE`、`pyproject.toml`） |
| 创建时间 | 2025-08-05（gh api） |
| 最近推送 | 2026-10-30（gh api，分析前一天，极活跃） |
| 最新版本 | v1.7.0（`pyproject.toml` version；release 2026-10-05） |
| 贡献者 | 约 74 人（gh api contributors 分页推算） |
| 商业形态 | 开源 CLI + Strix Cloud（SaaS）+ Enterprise（README.md） |

## 2. 项目是什么，解决什么问题

Strix 定位为"autonomous AI pentesting agents"（`README.md`）：与传统漏扫器靠静态规则产生大量误报不同，它让 LLM agent 像真实渗透测试者一样动态运行目标应用、构造验证请求、用可复现的 PoC 确认漏洞，并给出修复补丁。目标用户是开发者与安全团队，解决"人工渗透测试贵且慢、静态扫描误报多"的痛点（`README.md` Strix Overview 一节）。

核心能力来自三件事的组合。其一是完整的进攻性安全工具箱：沙箱镜像内置 Kali Linux 常用工具（nmap、sqlmap、nuclei、subfinder、naabu、ffuf 等，见 `containers/Dockerfile`），agent 通过统一的 shell 工具调用它们（`strix/tools/shell/README.md`）。其二是多 agent 编排（Graph of Agents）：根 agent 只负责拆解任务与调度，按"漏洞类型 × 组件"派生专职子 agent，形成"发现 → 验证 → 报告"的分工链（`strix/agents/prompts/system_prompt.jinja` multi_agent_system 一节）。其三是工程化的报告管线：发现以结构化字段（CVSS、CWE、OWASP 分类、fix_before/fix_after 补丁片段）落盘，同时产出 Markdown、JSON 与 SARIF 2.1.0，可直接被 GitHub code-scanning 消费（`strix/report/sarif.py` 模块文档字符串）。

支持三种测试模式：black-box（只有域名/URL）、white-box（提供源码，要求静态扫描工具 semgrep/gitleaks/trivy 与动态验证结合）、combined（代码 + 线上环境互相印证）（`system_prompt.jinja` TESTING MODES 一节）。目标可以是本地目录、GitHub 仓库、URL/域名，也支持 OpenAPI/Postman 契约驱动的 API 测试（`README.md` Usage Examples）。

商业模式是典型 open-core：本地开源版自带 Docker 沙箱与用户自己的 LLM key；Strix Cloud 提供免安装扫描、自动修复 PR；Enterprise 增加 SSO、合规报告与私有化部署（`README.md`、`AGENTS.md`）。同时项目对"其他 coding agent"友好：`skills/` 目录提供 9 个 SKILL.md，`AGENTS.md` 是给 Claude Code/Cursor 等 agent 的使用速查。

## 3. 技术栈与依赖

- **语言与运行时**：Python >= 3.12，uv 管理依赖（`pyproject.toml`、`uv.lock`）；Go（Bubble Tea TUI sidecar，`strix/interface/tui/go.mod`）；TypeScript/Vite（本地查看器，`strix/interface/viewer/frontend/`）。
- **Agent 框架**：`openai-agents[litellm]>=0.19`（`pyproject.toml`）——直接采用 OpenAI Agents SDK 的 `SandboxAgent`、sandbox capabilities（Shell/Filesystem）与 Session 机制（`strix/agents/factory.py`、`strix/runtime/session_manager.py` 的 import），Strix 在其上做编排层。
- **多 LLM 接入**：LiteLLM 统一路由（OpenAI/Anthropic/Google/OpenRouter/Vercel AI Gateway/本地 Ollama 等），`google-auth`、`boto3` 覆盖 Vertex AI/Bedrock（`pyproject.toml` dependencies；`STRIX_LLM` 环境变量见 `README.md`）。
- **沙箱**：`docker>=7.1.0`，Kali rolling 基础镜像，两阶段构建（Go 工具静态编译后拷入运行层，`containers/Dockerfile`）。
- **HTTP 代理**：`caido-sdk-client`——集成 Caido 作为全流量拦截代理（`strix/tools/proxy/`、`strix/runtime/caido_bootstrap.py`）。
- **报告与数据**：`cvss`（评分）、`reportlab`/`pypdf`（PDF 报告）、`markdown-it-py`、`pydantic>=2.11`（`pyproject.toml`）。
- **工程质量链**：mypy strict + pyright strict 双类型检查、ruff（40+ 规则集）、bandit、pre-commit、Dependabot（`pyproject.toml`、`.pre-commit-config.yaml`、`.github/dependabot.yml`）。
- **供应链防护**：`[tool.uv] exclude-newer = "5 days"`——依赖解析只取发布满 5 天的包，注释明确针对 2026-10-21 litellm 1.107.0 投毒事件的防御（`pyproject.toml`）。
- **打包**：PyInstaller 独立二进制（`strix.spec`）+ 带 Go TUI sidecar 的平台 wheel（`scripts/tui_sidecar_hook.py`、`.github/workflows/build-release.yml`），覆盖 macos arm64/x86_64、linux x86_64/arm64、windows x86_64 五个目标。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `strix/agents/` | agent 构造与提示词：`factory.py`（870 行，装配根/子 agent 的全部工具）、`prompts/system_prompt.jinja`（543 行主提示词）、`prompts/scope.jinja` |
| `strix/core/` | 扫描执行中枢：`runner.py`（顶层扫描 runner）、`execution.py`（agent 循环与子 agent 孵化）、`agents.py`（AgentCoordinator，agent 图状态机）、`hooks.py`（预算/用量钩子）、`inputs.py`、`sessions.py` |
| `strix/tools/` | agent 可用工具集：`agents_graph`（create_agent/send_message/wait_for_agents 等图操作）、`reporting`（漏洞报告 CRUD）、`notes`/`todo`/`thinking`/`threat_model`/`coverage`/`finish`/`wait_for_user`/`web_search`、`proxy`（Caido 交互）、`mcp`（外部 MCP 服务器接入）、`shell`/`agent_browser`/`apply_patch`（SDK 工具的文档说明） |
| `strix/runtime/` | 沙箱生命周期：`backends.py`（后端注册，默认 docker）、`session_manager.py`（每扫描会话与 /workspace 装载）、`docker_client.py`（注入 NET_ADMIN/NET_RAW 与 host-gateway）、`caido_bootstrap.py` |
| `strix/llm/` | 上下文工程：`compaction.py`（超窗时会话压缩，保留 tool-call 配对）、`context_budget.py`（按 LiteLLM 元数据解析 token 预算） |
| `strix/report/` | 报告管线：`state.py`（全局发现状态与 artifact 落盘）、`sarif.py`（SARIF 2.1.0）、`dedupe.py`（LLM 判重）、`pricing.py`、`writer.py`、`coverage.py` |
| `strix/interface/` | 用户界面：`cli.py`/`main.py`（CLI 入口）、`tui/`（Go Bubble Tea 交互界面，编译为 sidecar）、`viewer/`（本地 web 仪表盘，Vite 前端 + Python server）、`cloud/`（app.strix.ai 的 REST CLI） |
| `strix/skills/` | 内部知识包（agent 运行时经 load_skill 加载）：analysis、cloud、coordination、frameworks、protocols、reconnaissance、scan_modes、technologies、tooling、vulnerabilities 十类 |
| `skills/` | 面向外部 coding agent 的 9 个消费者技能（SKILL.md） |
| `containers/` | 沙箱镜像 Dockerfile 与入口脚本 |
| `tests/` | 115 个 pytest 文件、约 3.7 万行 |
| `docs/`、`benchmarks/`、`scripts/` | Mintlify 文档站（`docs.json`）、基准说明、构建/安装脚本 |

组织方式评述：单包（single package）单仓，但内部模块边界清晰——"编排（core/agents）/工具（tools）/执行环境（runtime）/呈现（interface）/产出（report）"五层职责分明；代码内注释密度高且解释"为什么"（大量 lazy import 循环依赖的注释，`pyproject.toml` per-file-ignores 段几乎是架构注释集）。Go TUI 与 TS 前端以 sidecar/预编译产物形式内嵌进 Python 包发布。

## 5. 核心架构与关键实现

### 5.1 Agent 图编排：根 agent 只做指挥官

`strix/agents/prompts/system_prompt.jinja` 用 Jinja2 条件区分根/子 agent。根 agent 被明确禁止亲手测试："YOU ARE THE ROOT AGENT. Your job is ORCHESTRATION, not hands-on testing... you do NOT run scanners, crawlers, fuzzers"（`system_prompt.jinja` root_agent_directive）。子 agent 按"每个漏洞类型 × 每个组件一个专职 agent"孵化，黑盒流程固定为 Discovery → Validation → Reporting 三段链，验证通过后才派报告 agent（同文件 SIMPLE WORKFLOW RULES 段）。生命周期由专用工具闭环：根 agent 以 `finish_scan` 结束、子 agent 以 `agent_finish` 结束、`wait_for_agents`/`wait_for_user` 是唯一的等待/让位方式——纯文本回合既不结束也不暂停执行（同文件 AUTONOMOUS/INTERACTIVE BEHAVIOR 段）。

图状态由 `strix/core/agents.py` 的 `AgentCoordinator` 单点管理：每个 agent 有 `running/waiting/completed/stopped/crashed/failed/budget_paused` 状态，运行时对象持有 `asyncio.Task`、`wake` 事件与 mailbox（消息先入箱再唤醒）：

```python
@dataclass(slots=True)
class AgentRuntime:
    session: Session | None = None
    task: asyncio.Task[Any] | None = None
    stream: Any | None = None
    interrupt_on_message: bool = False
    resumable: bool = True
    wake: asyncio.Event = field(default_factory=asyncio.Event)
    mailbox: list[dict[str, Any]] = field(default_factory=list)
    user_wake_required: bool = False
```
（`strix/core/agents.py` 第 25-34 行附近）

预算治理同样在这层：`BudgetPolicy = Literal["stop", "pause"]`，超预算时 `trigger_budget_stop()` 唤醒所有 parked agent 使其退出（`strix/core/agents.py`）。

### 5.2 沙箱执行环境：SDK 沙箱 + Kali 镜像 + Caido 代理

`strix/runtime/backends.py` 以注册表模式选择执行后端（当前仅 `docker`），把 openai-agents SDK 的 `DockerSandboxClient` 包上 `StrixDockerSandboxClient`，注入 NET_ADMIN/NET_RAW capabilities 与 `host.docker.internal` host-gateway（nmap 需要原始套接字）。`strix/runtime/session_manager.py` 负责把目标代码/文件以单个 tar 流上传进容器 `/workspace`（"Extra files travel as one tar archive: a single upload plus one extraction"，避免逐文件往返）。

工具侧，agent 的 shell 走 SDK `ShellTool` 的 `exec_command`/`write_stdin`（PTY 支持交互式进程），且 `http_proxy/https_proxy` 环境变量把所有 shell 子进程流量导经 Caido 代理，实现全量请求记录与重放（`strix/tools/shell/README.md`）。HTTP 检查类工具（list_requests/repeat_request/view_sitemap 等）在 `strix/tools/proxy/tools.py`，浏览器测试由 agent-browser（共享 Chromium，按 `--session` 隔离，空闲 3 分钟回收，`system_prompt.jinja` multi_agent_system 段）承担。

### 5.3 上下文工程与 LLM 层

`strix/llm/compaction.py` 实现与 provider 无关的会话压缩：会话超窗后把旧回合摘要成单个检查点、最近回合原样保留，并保持 tool-call/tool-result 配对以产出合法的 provider 输入；token 预算由 `strix/llm/context_budget.py` 从 LiteLLM 模型元数据解析（带可配置大回退）。模型路由通过 `STRIX_LLM` 环境变量（如 `vercel_ai_gateway/zai/glm-5.3`、`openrouter/...`、`chatgpt/gpt-5.4` 订阅模式），支持 ChatGPT 订阅登录（`strix/interface/auth_cli.py`、`README.md`）。

### 5.4 报告管线：结构化发现到 CI 可消费的产物

报告由 `create_vulnerability_report` 工具写入 `ReportState`（`strix/tools/reporting/tool.py`、`strix/report/state.py`），每次保存同步产出 CSV/Markdown/JSON/SARIF 四类 artifact。`strix/report/sarif.py` 的设计注释很能代表工程质量：规则以 CWE 归一为键、5 级严重度折叠进 SARIF 3 级但保留原始 CVSS、`security-severity` 供 GitHub 排序、含 `fix_before/fix_after` 的发现转成 SARIF `fixes` 一键建议、路径校验拒绝 URI/绝对路径/穿越模式、DAST 类无文件位置的发现用 `logicalLocations` 锚定。判重（`strix/report/dedupe.py`）干脆用 LLM 来做——把新发现交给模型与既有报告比对。头less 模式退出码语义化：0 干净、1 致命错误、2 发现漏洞（`AGENTS.md`），方便 CI 直接门控。

## 6. 构建与运行

来自 `README.md`、`AGENTS.md`、`Makefile`：

```bash
# 安装（独立二进制，或 pipx/uv/pip 安装 strix-agent）
curl -sSL https://strix.ai/install | bash

# 配置 LLM 并扫描本地目录
export STRIX_LLM="vercel_ai_gateway/zai/glm-5.3"
export LLM_API_KEY="your-api-key"
strix --target ./app-directory          # 交互模式
strix -n -t ./ --scan-mode quick        # headless，CI 用
strix view                              # 打开本地结果仪表盘

# 开发
make dev-install                        # uv sync + pre-commit
make check-all                          # ruff + mypy + pyright + bandit
uv run pytest                           # 测试
uv run strix --target <target>          # 源码运行
make viewer / make tui-build / make wheel   # 前端/TUI/打包
```

首次运行自动拉取沙箱 Docker 镜像；结果落盘 `strix_runs/<run-name>/`（`penetration_test_report.md`、`vulnerabilities/*.md`、`vulnerabilities.json`、`findings.sarif`、`run.json`，出处 `AGENTS.md`）。前置条件：Docker 运行中 + 任一支持的 LLM API key。

## 7. 测试与工程质量

- **测试规模**：115 个测试文件、约 37,270 行 pytest（实测统计），asyncio_mode=auto（`pyproject.toml`）。覆盖面按文件名可见一斑：agent 图协调（`test_agent_graph_coordination.py`）、预算生命周期（`test_e2e_budget_lifecycle.py`）、压缩（`test_compaction.py`）、SARIF（`test_sarif.py`、`test_sarif_stride.py`）、MCP 韧性（`test_mcp_resilience.py`）、TUI 协议一致性（`test_tui_protocol_conformance.py`）、打包正确性（`test_packaging.py` 直接断言 wheel 内含 Go sidecar）等，以单元/契约测试为主，未见真实网络型 e2e（符合"静态阅读、CI 不打真实目标"的合理边界）。
- **CI**：`.github/workflows/ci.yml` 拆成 5 个可复用 workflow（python/tui/viewer/package/workflows）+ 一个 `ci-passed` 汇总门禁（任何子任务非 success 即失败）；`build-release.yml` 五平台矩阵构建 PyInstaller 二进制与平台 wheel。
- **类型与静态检查**：mypy strict 与 pyright strict 同时启用（`pyproject.toml` 两个完整配置段），ruff 启用 40+ 规则族（含 flake8-bandit 的 S 规则），bandit 另跑，pre-commit 钩子齐备。
- **代码质量观察**：~4.5 万行主代码配 ~3.7 万行测试，比例健康；`pyproject.toml` 的 per-file-ignores 大量条目解释了每处豁免的架构原因（lazy import 防循环依赖、启动路径避开秒级 import 的 litellm/docker 等），属于罕见的高质量注释实践；`uv.lock` + 5 天供应链冷却期显示对依赖投毒的主动防御。

## 8. 社区与维护状态

- **活跃度**：仓库创建于 2025-08-05，最近推送 2026-10-30（gh api），即分析时前一天仍在提交——14 个月冲到 7 万+ 星，README 带 trendshift 周榜徽章，属于当期现象级项目。
- **Release 节奏**：约每月一版——v1.7.0（2026-10-05）、v1.6.2（2026-09-05）、v1.6.1（2026-09-02）、v1.6.0（2026-09-01）、v1.5.3（2026-08-10）（gh api releases）。
- **贡献者结构**：约 74 名贡献者（gh api 分页推算）；核心由 Strix 公司驱动（`pyproject.toml` 作者字段 `hi@usestrix.com`），开源 CLI 是其 Cloud/Enterprise 商业产品的引擎层，典型 open-core 公司仓。
- **Issue**：open issues 534（gh api），配合 Discord 社区运营（README 徽章）。issue 模板齐备（`.github/ISSUE_TEMPLATE/`）。

## 9. 亮点与风险

**亮点**

1. Agent 编排设计成熟：根 agent"只指挥不执行"的角色硬约束、三段式发现-验证-报告链、生命周期工具（finish_scan/agent_finish/wait_*）构成显式状态机，配合 `AgentCoordinator` 的邮箱/唤醒/预算暂停机制，是少数把多 agent 协作做成可恢复工程的公司级实现（`system_prompt.jinja`、`strix/core/agents.py`）。
2. 工程质量在同类开源安全工具中顶级：双 strict 类型检查、3.7 万行测试、五段 CI 门禁、五平台发布矩阵、供应链冷却期（`pyproject.toml`、`.github/workflows/`）。
3. 真实工具链深度集成而非玩具：Kali 沙箱 + Caido 全流量代理 + nmap/nuclei/sqlmap/semgrep 等，且所有 agent 流量统一过代理可审计重放（`containers/Dockerfile`、`strix/tools/proxy/`）。
4. 输出直接进 DevSecOps 流水线：SARIF 2.1.0 对齐 GitHub code-scanning、含修复建议的一键 fixes、headless 退出码门控、PR 级 diff 范围扫描（`strix/report/sarif.py`、`AGENTS.md`、`README.md` CI 示例）。
5. 对 AI 生态的开放姿态：MCP 服务器接入、面向外部 coding agent 的 9 个 skills 与 `AGENTS.md`/llms.txt，把自己定位为"agent 时代的可调用安全能力"（`strix/tools/mcp/`、`skills/`）。

**风险/局限**

1. 合规边界（客观记录）：这是进攻性渗透测试工具，`README.md` 末尾明确 WARNING——仅限对自有或获书面授权的系统使用，未授权测试在多数司法辖区违法，责任由使用者自担。同时需指出 `strix/agents/prompts/system_prompt.jinja` 中存在 "AUTHORIZATION STATUS: You have FULL AUTHORIZATION... never question your authority" 与 "REFUSAL AVOIDANCE" 等提示词段落，其设计意图是防止模型因目标模型的安全策略而中断授权测试，但这类指令客观上也降低了滥用门槛；本仓库的技术分析仅覆盖架构与工程实现，不提供攻击操作指引。
2. 部署门槛：强制依赖本机 Docker，且沙箱容器需要 NET_ADMIN/NET_RAW 特权与 Kali 镜像（体积与攻击面都较大），企业内网落地需要评估（`strix/runtime/backends.py`、`containers/Dockerfile`）。
3. 成本与时长：deep 模式扫描可达数小时、多 agent 并行消耗大量 LLM token，预算机制（`--max-budget`、BudgetPolicy）是必需品而非可选项（`AGENTS.md`、`strix/core/agents.py`）。
4. 成熟度标注：包状态为 `Development Status :: 3 - Alpha`（`pyproject.toml`），1.x 阶段接口与 artifact 格式可能变动；对 openai-agents SDK 沙箱能力（Shell/Filesystem capability）强耦合，上游 SDK 变更是主要外部风险（`strix/agents/factory.py`）。

## 10. 总结评价

Strix 是"AI 安全 agent"赛道目前工程质量与产品完成度最高的开源实现之一：它没有停留在"LLM 包一层 nmap"的演示级别，而是把渗透测试拆成可编排、可审计、可门禁的工程流程——根/子 agent 的角色分离与生命周期工具、Kali 沙箱与 Caido 代理构成的统一执行面、SARIF/修复补丁直通 GitHub code-scanning 的产出链，以及双 strict 类型检查 + 3.7 万行测试的开发纪律，都达到了商业产品的标准；配合 open-core 商业模式（CLI 开源引流、Cloud/Enterprise 变现），14 个月 7 万+ 星的增长有真实产品力支撑。适合安全团队与开发者在其授权资产上做动态验证型安全测试，也适合研究多 agent 编排、agent 沙箱隔离与上下文压缩的工程师读其源码（推荐入口：`strix/core/agents.py`、`strix/agents/factory.py`、`strix/agents/prompts/system_prompt.jinja`、`strix/report/sarif.py`）。使用前务必确认目标授权——这是该类工具不可妥协的前提。
