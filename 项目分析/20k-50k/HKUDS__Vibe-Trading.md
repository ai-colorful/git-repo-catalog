# HKUDS/Vibe-Trading 项目分析

> 星数 38380（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-17

## 1. 项目速览

一句话定位：Vibe-Trading 是香港大学数据智能实验室（HKUDS）出品的"个人交易 Agent"——用自然语言驱动金融研究、回测、模拟交易与（授权范围内的）实盘下单的 LLM Agent 平台。

| 项目 | 值（gh api 核实，2026-11-17） |
|---|---|
| 名称 | HKUDS/Vibe-Trading |
| 星数 / Fork | 38380 / 6245 |
| 主语言 | Python（另有 React/TypeScript 前端） |
| 许可证 | MIT（LICENSE） |
| 创建时间 | 2026-04-01，7.5 个月达成 38.4k 星 |
| 最近 push | 2026-11-16（昨日仍在推送） |
| 版本 | v0.1.19（PyPI 包名 `vibe-trading-ai`，共 16 个 release：v0.1.4→v0.1.19） |
| 主页 | https://vibetrading.wiki/ |
| 规模 | agent/ 主包 2407 个 .py、约 56.8 万行 Python；测试 1122 个文件 |

## 2. 项目是什么，解决什么问题

Vibe-Trading 把"金融问题"变成三样东西：研究报告、可运行的回测、模拟盘组合。README（`README.md`）的原话是 "turns finance questions into research reports, runnable backtests and paper portfolios"；实盘交易是可选项——用户显式授权某个券商通道、设定限额（mandate），随时可一键 halt，资金始终留在券商处。README 末尾 Disclaimer 明确声明：这是研究与交易软件，不构成投资建议，不托管资金，不运行撮合场所，broker 实盘能力是实验性的、未经作者在真实账户上验证（`README.md` Disclaimer 节）。

它解决的问题可以概括为：把过去散落在 Wind/同花顺/TradingView/QuantConnect 等付费工具里的工作流（行情获取、因子回测、估值建模、组合分析、复盘）统一塞进一个对话式 Agent。用户在 CLI/Web 里说"回测 BTC-USDT 的 20/50 双均线策略"，Agent 会自动取数、写策略、跑回测、给出与 buy-and-hold 的对比和最大回撤（`README.md` Quick Example）。内置的"Alpha Zoo"可一键基准测试 5 个研究库（GTJA191、Alpha101、Qlib158、academic、fundamental）共 478 个因子（`agent/src/factors/zoo/`，gtja191 目录确有 alpha_001..alpha_191 共 191 个文件）。

面向的交易场景覆盖很宽：A 股、港股、美股、加密货币、期货、外汇、期权，以及印度/韩国/越南/巴西/阿根廷等新兴市场（`agent/backtest/engines/` 17 个市场引擎）。数据源接入 28 个行情源并有自动 fallback 链（`agent/backtest/loaders/` 约 47 个 loader 文件）；账户侧接 19 家券商（`agent/src/trading/connectors/`：alpaca、binance、etoro、futu、ibkr、okx、robinhood、tiger、longbridge、zerodha、upbit、kis 等）。

项目还有几个特色玩法：Swarm 多 Agent 投委会（30 个预设团队 yaml，`agent/src/swarm/presets/`）、Pixel Office 人工任务看板（`pixel-office/README.md`：分配任务不启动 agent 也不下单，"组织职级不授予交易权限"）、Shadow Account 交易日志复盘（导入券商成交记录做规则化重放对比）、定时研报推送到 Slack/Telegram/邮件等 12 种 IM 通道（`pyproject.toml` channels extras）。

## 3. 技术栈与依赖

- 后端：Python 3.11+，FastAPI + uvicorn + sse-starlette（API Server），pydantic（`pyproject.toml`）
- Agent 编排：LangChain 1.x + LangGraph（含 langgraph-checkpoint）、自研 ReAct 循环 `agent/src/agent/loop.py`；MCP 通过 fastmcp/mcp 实现（`agent/mcp_server.py` 3498 行）
- 数据与计算：pandas/numpy/scipy/scikit-learn/duckdb/exchange-calendars/bottleneck；行情依赖 ccxt、yfinance、akshare、tushare、baostock（extra）等
- 报告与文档：matplotlib、weasyprint + reportlab（PDF 回退）、jinja2、openpyxl、python-docx/pptx、pypdf
- 前端：React 19 + TypeScript + Vite + Tailwind + zustand + echarts + i18next（`frontend/package.json`，11 种界面语言）
- 桌面：Electron 壳（`desktop/electron/`），带签名更新机制（CHANGELOG 2026-11-14）
- 部署：Docker Compose（`docker-compose.yml`，含 VT-007 运行时加固注释）、ZimaOS 模板（`zimaos/`）
- 质量工具：pytest + pytest-cov + pytest-socket（禁网单测）、ruff、black（`pyproject.toml` dev extras）

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `agent/` | Python 主包：CLI、API Server、MCP Server、Agent 循环、回测、技能、工具、全部测试 |
| `agent/src/agent/` | ReAct 核心循环与上下文管理（loop.py 1661 行、context_compact.py、tool_lifecycle.py） |
| `agent/src/swarm/` | 多 Agent DAG 编排（runtime.py 拓扑分层调度 + 30 个团队预设） |
| `agent/src/live/` | 实盘安全层：mandate（限额契约）、halt、order_guard、enforcement、audit |
| `agent/src/trading/` | 券商连接框架与 19 个 connector 子目录 |
| `agent/src/providers/` | 26 个 LLM 供应商目录与统一网关（llm_providers.json） |
| `agent/src/skills/` | 89 个技能包（SKILL.md + 示例），如 backtest-diagnose、pair-trading、chanlun、perp-funding-basis |
| `agent/src/tools/` | 约 60 个 Agent 工具模块（backtest_tool、alpha_bench_tool、options_payoff_tool、ocr 等） |
| `agent/backtest/` | 17 个市场引擎 + loaders + metrics/optimizers/validation/run_card |
| `agent/src/factors/` | Alpha Zoo：5 库 478 因子 + bench/compare 运行器 |
| `agent/src/shadow_account/` | 交易日志复盘（HTML/PDF 报告模板） |
| `agent/src/{memory,goal,governance,session,research_artifacts,research_lab,scheduled_research,scheduling,data_gateway,quantlib,portfolio,strategy_discovery,strategy_store}` | 记忆/目标/治理/会话/研究工件/研究实验室/定时研究/调度/数据网关/量化库/组合/策略发现与仓库 |
| `frontend/` | React 19 Web UI，16 个页面（Home/Agent/AlphaZoo/OptionsLab/PaperTrading/PixelOffice/Portfolio/RunDetail 等） |
| `desktop/`、`zimaos/`、`pixel-office/` | Electron 桌面壳、ZimaOS 部署模板、Pixel Office 说明（真身在 frontend 页面 + agent 工具） |
| `guides/`、`wiki/`、`schema/`、`tools/`、`scripts/` | 用户文档、文档 wiki、工具契约 schema、CI/网关探针脚本、构建脚本 |

组织方式评述：单体大仓（monorepo 式），Python 主包内再按领域切了 40+ 子模块；打包上 `[tool.setuptools] package-dir = {"" = "agent"}`（`pyproject.toml`），PyPI 一个包同时提供 CLI、API、MCP 三个入口。前后端同仓、桌面壳独立顶层目录。整体是"平台型单体"：模块边界清晰（每个能力一个子目录 + SKILL.md 声明式技能），但体量巨大，56.8 万行 Python 已超出常规个人项目的数量级。

## 5. 核心架构与关键实现

### 5.1 Agent 循环与五层上下文管理（`agent/src/agent/loop.py`）

核心是自研 ReAct 循环 AgentLoop，文件头注释给出五层上下文策略：L1 microcompact（内存压力下剪旧工具结果）→ L2 context_collapse（不调 LLM 折叠长文本块，零成本）→ L3 auto_compact（LLM 结构化摘要 + token 预算尾部保护）→ L4 compact 工具（模型显式调用触发 L3）→ L5 iterative update（第 N 次压缩更新此前摘要而非重开）。工具执行支持只读批处理：连续只读工具用线程池并行（`loop.py` 头部注释与 import 区）。配套文件分工明确：`tool_lifecycle.py`（1087 行，工具调用生命周期）、`instrument_identity.py`（1071 行，标的身份台账，防止把 7203.T 之类不认识的代码错误套到 A 股规则）、`trace.py`（JSONL 全量 trace）、`tuning.py`（超时可调）。

### 5.2 Swarm 多 Agent 编排（`agent/src/swarm/runtime.py`）

"Swarm DAG orchestration runtime"：按拓扑分层调度 worker，层内并行、层间串行，后台守护线程执行，支持取消与事件回调（`runtime.py` 模块注释）。任务依赖经 `task_store.py` 的 `resolve_dependencies/topological_layers/validate_dag` 校验；worker 产出在 `worker.py` 汇集，并有 `grounding.py`（证据锚定）与 `evidence_handoff.py`（证据交接）。30 个预设团队 yaml（`agent/src/swarm/presets/`）定义了 investment_committee（多空辩论）、risk_committee、ml_quant_lab、statistical_arbitrage_desk 等，README 示例：`vibe-trading --swarm-run investment_committee '{"target":"TSLA","market":"US"}'`。

### 5.3 实盘安全层：Mandate 不可变契约（`agent/src/live/mandate/model.py`）

实盘权限用 frozen dataclass 表达"有界自治契约"，文件头注释：mandate 启动时读入后不可变，"agent 无法利用的验证面为零"。HardCaps 层包括：account_funding_usd（券商侧隔离资金，绝对上限，本地仅镜像做防御性预检）、max_order_notional_usd、max_total_exposure_usd、max_leverage（1.0=纯现金）、allowed_instruments 白名单（空=全部拒绝，fail-closed）、max_trades_per_day。外层还有 `live/halt.py`（halt 扫)、`live/order_guard.py`、`live/enforcement.py`、`live/audit.py` 与 kill switch。README 券商矩阵中 Live 能力仅 alpaca/binance/etoro/futu/mt5/okx/robinhood/tiger 8 家声明支持，且 Disclaimer 明示该能力实验性、未在真实账户验证过。

### 5.4 回测引擎与数据网关（`agent/backtest/engines/` + `agent/backtest/loaders/` + `agent/src/data_gateway/`）

17 个市场引擎继承 `engines/base.py`。以 `engines/china_a.py` 为例，文件头硬编码 A 股规则：T+1、零售无卖空、主板 ±10%/创业板科创板 ±20%/北交所 ±30% 涨跌停、100 股整手、佣金万 2.5 最低 5 元、印花税卖出 0.05%、无杠杆，`can_execute()` 在撮合前逐条执行这些约束。数据侧约 47 个 loader（tencent/eastmoney/akshare/tushare/baostock/yahoo/yfinance/stooq/sina/ccxt/okx/binance/alphavantage/fmp/tiingo/finnhub/gildata/sec_edgar/mt5 等），`source: "auto"` 时按市场推断 fallback 链（README 数据源表：A 股/港股先腾讯再东财；美股先 Yahoo 再 Stooq/Sina；加密先 OKX/Binance/CCXT）。2026-11-15 的 news 引入统一数据网关 `agent/src/data_gateway/`：多源 fallback、共享缓存、有界并发，响应带 source/currency/复权元数据。回测结果由 `backtest/run_card.py` 生成"运行卡"，记录数据源、模型、执行记录与指标引用，README 强调"只有引擎自己的输出才能佐证回测报告"（v0.1.15 贡献者记录 #1685）。

### 5.5 LLM 集成（`agent/src/providers/`）

26 个供应商收在 `llm_providers.json` 一个目录里（openai/gpt-5.5、anthropic/claude-sonnet-4-6、deepseek、gemini、dashscope/qwen、zhipu/glm-5.1、moonshot/kimi、minimax、ollama、github-copilot、claude-cli 本地 `claude -p`、openai-codex OAuth 等）。`registry.py`/`chat.py`/`llm.py` 统一抽象，2026-11-14 news 称"Agent/CLI/API/Swarm 共享一个目录与配置层"；配套 `stream_guard.py`、`endpoint_guard.py`（跨源重定向在转发凭据前拦截）、`content_filter.py`、`tool_policy.py`。原生 Anthropic 适配器默认 16384 输出 token、可 `ANTHROPIC_MAX_TOKENS` 覆盖（CHANGELOG）。

## 6. 构建与运行

来自 `README.md` Quick Start / `pyproject.toml` / `docker-compose.yml`：

```bash
# PyPI 安装（CLI + API + MCP 同包）
pip install vibe-trading-ai
vibe-trading init      # 向导选择模型供应商
vibe-trading           # 进入交互式终端（/help、/model、/history、/show、/pine）
vibe-trading serve --port 8899   # API Server，/docs 可交互
vibe-trading-mcp       # MCP stdio 服务（Claude Desktop 配置 mcpServers 即用）

# Web（前后端一起，http://localhost:8899）
cp agent/.env.example agent/.env   # 配置模型供应商与 key
docker compose up --build

# 示例
vibe-trading run -p "Backtest a BTC-USDT 20/50 moving-average crossover for 2024..."
vibe-trading alpha bench --zoo gtja191 --universe csi300 --period 2018-2025 --top 20

# 测试与 lint（pyproject.toml：testpaths=["agent/tests"]，markers unit/integration/slow）
pytest agent/tests
ruff check agent
```

可选 extras 按需安装：`ibkr/longbridge/mt5/zerodha/upbit/deepseek/copilot/anthropic/openbb/krx/stats/ashare/channels` 等（`pyproject.toml`，lazy-import + 可行动的报错提示，保持基础安装精简）。

## 7. 测试与工程质量

- 规模：`agent/tests/` 1122 个测试文件，按子系统分目录（api/、factors/、quantlib/、memory/、scheduling/、harness_eval/ 等），覆盖风格是"合同测试 + 回归测试"：如 `loader_contract.py`（统一 OHLCV 不变量）、`tests/fixtures/`、evals 离线用例（`agent/evals/grounding/`、`agent/evals/harness/`，README 称 20 个离线任务重放用例）。
- CI：`.github/workflows/test.yml` 在 Linux x64 / macOS ARM64 / Windows x64 三平台跑锁文件哈希与前后端测试（注释详尽解释为何 `npm ci --no-audit`）；另有 `docker-build.yml`、`loader-health.yml`（每周行情源健康金丝雀）、`desktop-windows.yml`（签名更新）、`wiki.yml`。
- 质量观察：ruff + black + pytest-socket（单测禁网）进 dev extras；`tools/` 里有大量网关内存/容量/恢复探针脚本；`schema/tool_contracts/` 存工具契约。工程纪律明显强于同类：README 的 news 区几乎每日一条 fix 记录，CHANGELOG 细到"SPY/GLD 回测与独立台账 206/140 笔成交全部对齐"（README news 2026-11-12）。局限：无系统覆盖率数据（coverage fail_under=0，`pyproject.toml`），56.8 万行代码对纯社区维护是高负担。

## 8. 社区与维护状态

- 活跃度：gh api 显示 pushed_at=2026-11-16（分析前一天）；README news 区 2026-10-31 至 2026-11-16 每日都有条目，`CHANGELOG.md` 按日记录。
- 发布节奏：16 个 release（v0.1.4→v0.1.19，最新 2026-11-05），v0.1.19 单版本即合并 116 个社区 PR + 21 个维护者提交、来自 26 位贡献者（README news 2026-11-05）。贡献者名单覆盖 v0.1.10 以来的每个周期，数百人规模（README Contributors 节），README 称"维护者移植（maintainer ports）"机制——未过审的修复由维护者落地并保留署名。
- 组织背景：HKUDS（HKU 数据智能实验室）出品，README Ecosystem 节列出同组织 NanoBot、AI-Trader、CLI-Anything、OpenSpace、ClawTeam；派发方核实同组织 LightRAG/DeepTutor 均已在此前分析。
- 增长背景（客观陈述）：创建于 2026-04-01，7.5 个月 38.4k 星，与该组织前作同属高增长案例。可核实的客观因素包括：README 展示 Trendshift 每日榜徽章与 topic 星数排名（自称 2026-11-04 backtesting #1 / algorithmic-trading #3 / quantitative-finance #4）、6 语种 README + 11 语种 UI 本地化、Discord/微信/飞书社区、每日 news 滚动维护、大量结构化 good-first-issue 引流（README Contributing 节）。具体星数构成（自然流量 vs 社区运营占比）未确认。
- 贡献者结构：核心维护者（如 warren618/Haozhe Wu 负责发布集成）+ 高产外部贡献者（rudycelekli 92+58 PR 修复扫描、Shizoqua、he-yufeng 等）+ 长尾一次性贡献者。

## 9. 亮点与风险

**亮点**
1. 安全设计认真：实盘走"不可变 mandate 契约 + fail-closed 白名单 + 券商侧资金隔离 + halt/kill switch"多层防线（`agent/src/live/mandate/model.py`、`live/halt.py`），模拟盘 `paper_trade_reconcile` 工具可独立对账（README Accounts 节）。
2. 回测务实：17 个市场引擎把真实交易规则（A 股 T+1/涨跌停/印花税、印度印花税、加密资金费率、期货保证金）写进撮合约束（`agent/backtest/engines/china_a.py` 等），并有 run_card 溯源与防"训练数据内作弊"的标记（v0.1.14 贡献者记录 #1618）。
3. 工程化程度高：56.8 万行 Python、1122 个测试文件、三平台 CI、统一数据网关与 LLM 网关、全量 JSONL trace、MCP 工具契约 schema。
4. 生态完整：CLI/Web/Electron/IM 频道/MCP 五种交互形态，26 个 LLM 供应商 + 28 个行情源 + 19 家券商，几乎全部 lazy-import 可选安装。
5. 研究资产可复用：478 因子 Alpha Zoo + 版本化研究工件（research_artifacts，可对比任意两版修订并导出）。

**风险/局限**
1. 体量与复杂度：56.8 万行 Python 对审计和贡献都是高门槛，核心 loop.py 1661 行且模块间耦合面大。
2. 实盘能力未经作者验证：README Disclaimer 明示 broker 实盘为实验性、"not verified by us against a real broker account — use it at your own risk"。
3. 数据源依赖免费/公开接口（腾讯、东财、Yahoo、BaoStock 等），稳定性和合规性随上游波动；loader-health 金丝雀只能发现不能预防。
4. 版本仍处 0.1.x Beta（pyproject classifier "Development Status :: 4 - Beta"），CHANGELOG 显示每日大量语义级修复，接口与行为可能频繁变化。

## 10. 总结评价

Vibe-Trading 是目前开源世界里把"LLM Agent + 全流程量化/交易工作流"塞得最满的一个项目：从行情网关、因子库、多市场回测、模拟盘、影子复盘，到带限额契约的授权实盘和 30 个 Swarm 研究团队，工程纪律和防幻觉设计（grounding 证据门、标的身份台账、回测溯源 run card）明显高于同类玩具级"AI 炒股"项目，免责声明也写得清楚克制——它是研究与模拟工具，不是收益承诺，实盘部分尚属实验性。7.5 个月 38.4k 星与其说是功能结果，不如说是 HKUDS 实验室成熟的流量运营 + 金融 Agent 风口 + 高频发布节奏共同作用的客观结果。适合三类人：想搭个人研究/模拟盘工作流的量化爱好者（重点读 backtest/ 与 factors/）、做多 Agent 编排的工程师（重点读 swarm/ 与 live/mandate 的安全设计）、以及想把 LLM Agent 工程化经验（上下文五层压缩、工具生命周期、证据门控）搬回自己项目的人。任何使用决策与资金操作请以项目自身 Disclaimer 为准，本分析不构成任何投资建议。
