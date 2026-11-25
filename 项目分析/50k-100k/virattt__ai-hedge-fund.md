# virattt/ai-hedge-fund 项目分析

> 星数 64177（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：**一个"AI 对冲基金"教学系统**——用 LLM 投资人 agent（巴菲特、芒格等风格化人设）加量化模型充当分析师，以真实对冲基金的组织架构（策略 pod → 组合构建 → 风险硬约束 → 执行 → 账本）在终端里回测和模拟盘交易，**不接真实资金、不构成投资建议**。

| 事实 | 值 | 出处 |
|------|-----|------|
| 名称 | ai-hedge-fund（PyPI 包名 `aihf`，v2.6.3） | `pyproject.toml` |
| 星数 / fork | 64,376 / 11,303（2026-10-31 gh api 实测） | gh api repos/virattt/ai-hedge-fund |
| 语言 / 许可证 | Python ≥3.11 / MIT | `pyproject.toml`、`LICENSE` |
| 创建时间 | 2024-11-29 | gh api |
| 最近 push | 2026-10-30（前一天） | gh api |
| 最新 release | v2.6.3（2026-10-30），10 月内连发 4 个版本 | gh api releases |
| 代码规模 | 120 个 Python 文件共 20,029 行，其中测试约 7,870 行（39 个 test_*.py） | tarball 统计 |

## 2. 项目是什么，解决什么问题

这是 virattt（Virat Singh）2024 年底创建的 AI 交易教学项目，当前 HEAD 已是**完全重写后的 v2 引擎**（`hedge_fund/README.md` 自述"ground-up rebuild"）。它要解决的问题在 `VISION.md` 里说得很清楚：市面上多数"AI 交易"项目是一次性脚本——跑一次、出一个信号、退出；本项目反其道而行，把**基金做成一个持续存在的对象**：你给它一份 mandate（投资规章），给它配备分析师，它按日程持续运行，并且把每一个决策连同分析师写下的论据完整记账。

架构上刻意映射真实对冲基金的组织结构（`VISION.md`）：研究员换成 alpha 模型（LLM 投资人 agent + 量化模型两种口味）、基金经理换成组合构建、CIO 换成资金分配器、风控总监换成硬性限制、交易台换成 broker 执行、后台账本换成 append-only 哈希链账本。三层全部可插拔：FUND = allocator over STRATEGIES，STRATEGY = blend policy over ANALYSTS，ANALYST = 一个输出 `Signal`（[-1,+1] 信念度 + 书面论据）的模型。

项目的核心卖点（也是设计铁律，`hedge_fund/README.md` "Principles" 节）：**同一动词 `advance` 跑三种模式**——回测（历史时钟 + 模拟 broker + 内存状态）、模拟盘 paper（实时时钟 + paper broker + 账本状态）、未来实盘（只差一个真实 broker）。"回测即实盘系统"，同一代码路径，不存在悄悄分叉的"研究版实现"；决策在 T 日收盘做出、T+1 日收盘执行，任何数据都不允许 lookahead。

必须如实记录：项目在 `README.md`、`VISION.md`、`ROADMAP.md` 反复声明**仅供教育与科研**，不用于真实交易，不提供投资建议，系统"does not actually make any trades"；LLM 人设是"对公开投资哲学的风格化近似，非本人、非背书"。实盘 broker 在路线图中明确标记为"opt-in、默认关闭、计划中"（`ROADMAP.md` Risk & execution 节）。

## 3. 技术栈与依赖

来源：`pyproject.toml`（Poetry 工程，包名 `aihf`）。

- **运行时**：Python ≥3.11；Poetry 构建，`poetry-core` 后端。
- **LLM 层**：langchain-anthropic（锁 0.3.5）、langchain-openai、langchain-deepseek、langchain-google-genai、langchain-xai；`hedge_fund/llm/registry.py` + `api_models.json` 维护模型清单，支持 Anthropic/OpenAI/DeepSeek/Google/xAI/Kimi/TypeSafe(Jev) 七家 provider（`.env.example`）；Ollama 本地模型在路线图中（`ROADMAP.md`：v1 保留 Ollama 是 v2 未迁移的最后一个阻塞项）。
- **数据**：`requests` 直连 Financial Datasets API（financialdatasets.ai，行情/基本面/财报），自建磁盘缓存层（`hedge_fund/data/`，缓存落 `~/.hedge-fund/cache/`）。
- **数值计算**：pandas、numpy、scipy、matplotlib（回测曲线绘图）。
- **交互界面**：textual（终端 TUI，v2 主界面）+ rich；CLI 亦是完整客户端。
- **数据模型**：pydantic v2 贯穿全部记录（Signal/SessionRecord/CycleRecord 等）。
- **配置**：pyyaml（mandate 即 YAML 数据文件）。
- **开发工具**：pytest、black（line-length=420，见 `pyproject.toml`——刻意放宽）、isort、flake8、textual-dev、watchfiles。

## 4. 目录结构与代码组织

顶层即单包 monorepo：`hedge_fund/` 一个 Python 包 + 3 个说明文档 + 4 个策略 YAML。组织方式评述：**按领域垂直分包、每包自带测试**（test_*.py 与实现同目录），无独立 src/tests 分离；规模适中（2 万行）却保持了清晰的分层——数据层在底、模型接口居中、引擎（pipeline）在上、客户端（tui/CLI）在最外，`llm/registry.py` 的注释明确记录了为避免 import 环而把模型注册表放在 `llm` 而非 `tui` 的决策。

| 目录 | 职责（行数，不含测试） | 出处 |
|------|------|------|
| `hedge_fund/pipeline/` | 引擎核心：`advance` 单会话推进、两阶段评估/执行、时钟抽象 | 2,323 行 |
| `hedge_fund/tui/` | Textual 交互应用：paper 控制台、回测看板、mandate 构建器 | 5,796 行（最大模块） |
| `hedge_fund/paper/` | 模拟盘：部署、哈希链账本、幂等 tick、分钟级 runner | 1,682 行 |
| `hedge_fund/llm/` | LLM 客户端工厂、prompt 缓存、模型注册表 | 1,756 行 |
| `hedge_fund/data/` | `DataClient` 协议、Financial Datasets 客户端、磁盘缓存、交易日历 | 1,732 行 |
| `hedge_fund/signals/` | `AlphaModel` 接口 + 5 个 LLM 人设 + PEAD 量化模型 + 注册表 | 1,310 行 |
| `hedge_fund/features/` | 点时（point-in-time）基本面快照、盘中上下文、断点表 | 1,204 行 |
| `hedge_fund/backtesting/` | 回测驱动：`advance` 循环、绩效指标、研究结果落盘 | 781 行 |
| `hedge_fund/fund/` | `FundSpec`/`StrategySpec`——mandate 即数据；example.yaml | 847 行 |
| `hedge_fund/brokers/` | `Broker` 协议、SimBroker、PaperBroker、交易成本模型 | 450 行 |
| `hedge_fund/scorecard/` | 分析师评分卡（命中率、盈亏比、IC） | 669 行 |
| `hedge_fund/portfolio/`、`risk/` | 信号混合 → 目标权重；硬性风险约束 | 355 + 325 行 |
| `strategies/*.yaml` | 策略库：fundamental-ls、deep-value、inflections、earnings-drift | 数据文件 |
| `VISION.md`、`ROADMAP.md` | 设计哲学（含"什么绝不妥协"清单）与能力地图（✅/🚧/⬜） | 文档 |
| `hedge_fund/paths.py`、`run.py` | `~/.hedge-fund/` 目录布局与 CLI 入口（`aihf` 脚本指向 `run:main`） | 入口 |

## 5. 核心架构与关键实现

### 5.1 `advance`：一个动词跑所有模式（`hedge_fund/pipeline/session.py`）

`advance(fund, state, session, broker, data_client, universe)` 是基金每日/每 bar 的最小工作单元，序列固定为 **reconcile → execute 待决决策 → mark 持仓 → assess 下一决策**（`session.py` 模块 docstring 与实现）。三个关键设计：

- **T 决策、T+1 执行**：T 日收盘评估出的 `DecisionRecord` 挂在 T 的 `SessionRecord.decision` 上，T+1 的记录携带执行结果 `executed`；`is_rebalance_session()` 只用当前与上一会话判断"是否调仓日"，规则在实时 tick 上可计算，回测与模拟盘因此天然一致（`session.py:240`）。
- **先对账，后交易**：`reconcile()` 逐票比对 broker 持仓与账本推导状态，现金按分比较；不一致抛 `BookMismatch`，"永不修复"（`session.py:224`）。
- **账本哈希链**：`SessionRecord.compute_hash()` 对整条记录做 SHA-256，链入 `prev_hash`；可选字段（如 `snapshot_times`）缺失时序列化保持与旧版本逐字节兼容，旧账本仍可验证（`session.py:109`）。

### 5.2 两阶段评估/执行与信号聚合（`hedge_fund/pipeline/stages.py`、`hedge_fund/portfolio/construction.py`、`hedge_fund/risk/limits.py`）

`assess_fund()` 遍历每个策略 pod：pod 内每位分析师对每只票调用 `model.predict(ticker, as_of, view)`，产出 `Signal` 后由 `blend_signals()` 做信念度加权投票——**弃权（abstained）剔除、中性仍投票；long-only 分析师的看空只能减少做多证据、绝不制造做空证据**（`construction.py:74`）。三种混合模式：long_only / long_short / dollar_neutral（多空各分一半总敞口，缺任一侧则整体归零）。各 pod 的权重再按资金份额 `slice_` 净额合并成单一账本（`stages.py:79`）。

风险层（`risk/limits.py`）是纯确定性算术，模块注释一语中的："Conviction requests, risk disposes"——**LLM 的影响力止步于 Signal**。三段式只缩不涨：单票上限 → 净敞口带 → 总敞口缩放；被削减的敞口留在现金，绝不重分配给其他票。每次触发记录 `ClampEvent` 审计轨迹；drawdown 止损在 `advance` 中触发后清仓并停在现金（`stages.py:stop_out_decision`），成交量上限（`max_adv_pct`，20 日均量）则在执行阶段截断订单量。

### 5.3 LLM 投资人 agent 与量化模型（`hedge_fund/signals/`）

两种分析师实现同一 `AlphaModel` 接口（`signals/base.py`：`predict(ticker, date, data_client) -> Signal`，类属性声明 `investment_approach` 与 `horizon_days` 评级期限）。`signals/__init__.py` 的 `ALPHA_MODEL_REGISTRY` 注册了 6 个模型：量化 PEAD（盈余公告漂移，BEAT 后做多 / MISS 后做空，horizon 60 交易日，`pead.py`）+ 5 个 LLM 人设（buffett/munger/graham/lynch/druckenmiller）。

LLM agent 的工程亮点（`signals/llm_agent.py`）：**人设只是一个 system prompt**（`buffett.py` 全部 57 行就是一段"你是巴菲特，按清单审查这家公司"的提示词 + JSON 输出 schema）；机器全部在基类——点时基本面快照（`features/snapshot.py`，数据层按 filing_date 而非报告期过滤，快照哈希即 LLM 缓存键，两次财报之间不重复付费）、prompt 缓存命中即返回、LLM 调用/解析失败**弃权**（value=0 + metadata.abstained）而数据层错误**传播**（fail-loud）、每次调用的完整 prompt/response 落盘留痕。另有 `blind` 模式：回测时隐去股票代码、行业、日期与绝对金额（市值只以分位数呈现），防止 LLM 靠训练记忆"作弊"——这是对 LLM 回测有效性的罕见诚实处理。

### 5.4 回测、模拟盘与账本（`hedge_fund/backtesting/fund.py`、`hedge_fund/paper/ledger.py`）

回测 = `advance` 沿基准（如 SPY）实际交易日网格循环（"假期与半周自然落出，无需交易所日历数学"，`fund.py` docstring），产出含每会话记录的 `FundBacktestResult`（NAV 曲线、年化 Sharpe、最大回撤、相对基准超额、交易成本）。模拟盘侧，`paper/ledger.py` 每会话一个 JSON 文件构成 append-only 哈希链，状态靠 `next_state` 折叠重放推导（"状态是派生的，从不手改"），分钟级基金按日建目录加 checkpoint 缓存；`aihf paper tick` 幂等、可安全挂 cron，漏掉的会话绝不跳过而是补跑（`run.py` docstring、`ROADMAP.md`）。`control.json` 是 kill switch，由 `replay` 读入 `FundState.halted`，任何客户端都无法绕过。

## 6. 构建与运行

来自 `README.md` 与 `hedge_fund/README.md`（本规范不运行，仅转录）：

```bash
# 安装（终端应用）
pipx install aihf          # 或 uv tool install aihf / pip install aihf
aihf                       # 无参数启动 Textual TUI（paper 交易 + 回测两个模式）

# 开发
git clone https://github.com/virattt/ai-hedge-fund.git && cd ai-hedge-fund
poetry install
poetry run aihf            # 等价 python -m hedge_fund.tui
poetry run pytest hedge_fund/   # 测试（Makefile: make test）
make dev                   # textual 热重载开发模式（Makefile）

# API key：首次运行时应用提示输入，存 ~/.hedge-fund/.env
# 需要 FINANCIAL_DATASETS_API_KEY（数据）+ 任一 LLM provider key

# CLI
aihf backtest ~/.hedge-fund/mandates/example.yaml --universe AAPL,MSFT,NVDA --start 2024-01-02 --end 2024-06-28
aihf paper create alpha --mandate example.yaml --universe AAPL,MSFT
aihf paper tick alpha      # 幂等，cron 安全
aihf paper halt alpha --reason "..."   # kill switch
aihf paper scorecard alpha # 分析师评分卡 JSON
```

## 7. 测试与工程质量

- **规模**：39 个 `test_*.py` 共约 7,870 行，对约 12,158 行非测试代码，测试/代码比约 0.65:1——对一个"教学项目"而言异常扎实；引擎核心测试密度最高（`pipeline/test_stages.py` 498 行、`backtesting/test_fund.py` 395 行）。
- **风格**：pytest，测试与实现同目录；`hedge_fund/conftest.py` 提供共享 fixture；测试覆盖契约类断言（如 `data/test_client_contract.py`、`test_timing.py`、逐模块 test_*.py）。测试通过 `__init__` 注入假 LLM 与假数据客户端（`llm_agent.py` 构造函数的 `llm`/`cache` 参数即为测试缝）。
- **CI**：**无**。`.github/` 下只有 ISSUE_TEMPLATE 两个模板，没有 workflows——代码质量靠测试与作者自律，没有自动化门禁（未确认是否在其他系统跑 CI）。
- **代码质量观察**：文档字符串密度极高且解释"为什么"而非"是什么"；pydantic 模型自带向后兼容序列化器（哈希链账本可跨版本验证，`models.py:33`、`risk/limits.py:50`）；fail-loud 原则贯彻一致（基础设施错误抛异常，只有真正"无数据"返回空）。black line-length=420 属反常规配置，实为作者刻意的行内长注释风格。

## 8. 社区与维护状态

- **活跃度**：创建于 2024-11-29，最近 push 2026-10-30（分析前一天），10 月连发 v2.5.0 → v2.6.3 四个 release——维护节奏处于高强度期（gh api releases）。
- **贡献者结构**：41 位贡献者，但**高度单核**——virattt 859 次提交，第二名仅 6 次（gh api contributors）。这本质上是一个作者的"开放构建"项目：路线图明确邀请社区实现分析师/策略，v2 的 `AlphaModel` 接口就是贡献面（`ROADMAP.md` "Analysts — the main contribution surface"）。
- **历史版本**：早期 tag 用日历版本号（v2026.7.10 等，gh api tags），现转 v2.x 语义化版本；v1（约 17 个 agent 人设的 CLI 版）已被 v2 取代，路线图注明"retiring the v1 CLI，剩余人设待移植"（`ROADMAP.md` Current focus；注：`hedge_fund/README.md` 提到的 v1 `src/`、`app/` 目录在当前 HEAD 已不存在，该引用已过时）。
- **Issue**：184 个 open issues（gh api），fork 数 11.3k——关注/试验热度远高于实质贡献率。

## 9. 亮点与风险

**亮点**

1. **"回测即实盘"的单一代码路径**：`advance` 一套逻辑跑回测/paper/未来实盘，T 决策 T+1 执行、哈希链 append-only 账本、先对账后交易——把金融回测最容易作假的三个点（lookahead、执行时点、状态漂移）从机制上堵死（`pipeline/session.py`、`hedge_fund/README.md` Principles）。
2. **LLM 与交易的严格隔离**：LLM 只产出信念与论据，仓位与订单由确定性代码完成，风险限制是 agent 不可逾越的硬门（`risk/limits.py`）；LLM 输入还做了 blind 化处理防止训练记忆泄漏进回测（`llm_agent.py`）。
3. **可解释性**：每个决策保留分析师书面论据、每次 clamp 有 ClampEvent 审计、每次 LLM 调用 prompt/response 落盘——"基金能解释自己，且你可以 fork 这个解释"（`VISION.md`）。
4. **贡献面设计清晰**：新增分析师 = 实现一个 `predict` + 注册 + 附测试，引擎零改动；新增策略 = 丢一个 YAML 进 `strategies/`（`hedge_fund/README.md` Contributing）。
5. **测试与文档质量远超同体量项目**：7,870 行测试、路线图逐项标注完成度、VISION/ROADMAP 双文档把"已建/在建/愿景"分得清清楚楚。

**风险/局限**

1. **零 CI、单核维护**：无自动化门禁，859/865 的提交集中在一人，bus factor 为 1。
2. **数据源单一且依赖付费服务**：行情/基本面全押 Financial Datasets API（`data/client.py`、`.env.example`），无免费兜底（Ollama 解决的是 LLM 费用，不是数据）。
3. ** alpha 模型目前很薄**：注册表仅 6 个模型，量化侧只有 PEAD 一个（`signals/__init__.py`）；路线图上动量、均值回归、regime 检测等全为 ⬜；validation 模块（CPCV/PBO 防过拟合）是空壳（`hedge_fund/validation/__init__.py` 为空），自我宣称的"验证门"尚未建成。
4. **教育定位与星数的落差**：README 明确"不做真实交易、无投资建议"，实盘 broker 未实现；星数反映的是"AI 交易"话题热度，使用者不应误读为可用交易系统。

## 10. 总结评价

这是 AI 交易领域少见的"工程认真"项目：它没有把 LLM 神化成印钞机，而是把它降格为"会写论据的分析师"，用真实基金行业的组织结构、点时数据纪律、确定性风控与 append-only 账本，搭出一个**能被审计、能被证伪、能被 fork 的模拟交易研究系统**。文档（VISION/ROADMAP）与代码的双向诚实——明确标注哪些已建、哪些是愿景——同样罕见。它不是投资工具，也不该当投资工具用；但对想学习"如何工程化地做 LLM 金融 agent + 回测"的开发者、想理解对冲基金分层架构的学习者、以及想给 LLM 回测找反作弊范式的实践者，这份 2 万行的代码库是极佳的教材。适合阅读源码的人群：Python 后端工程师（学 pydantic 建模与 fail-loud 设计）、LLM 应用开发者（学 prompt 缓存与 agent 失败契约）、量化入门者（学点时回测的正确姿势）。
