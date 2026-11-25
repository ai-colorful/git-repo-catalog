# mvanhorn/last30days-skill 项目分析

> 星数 65187（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：一个安装到 AI Agent（Claude Code / Codex / Cursor / Gemini CLI 等 50+ 宿主）里的"研究技能"，输入任意话题，并行检索 Reddit、X、YouTube、TikTok、HN、Polymarket、GitHub 等 25+ 社交/社区/预测市场信源，按真实互动量（点赞、评论、真金下注）排序并合成为一份"最近 30 天"中文/多语研究简报。

| 事实 | 值 | 出处 |
|---|---|---|
| 名称 | last30days-skill（插件名 last30days） | `.claude-plugin/plugin.json` |
| 星数 | 66,024（gh api 实时值；快照 65,187），fork 5,752 | gh api repos/mvanhorn/last30days-skill |
| 版本 | v3.31.0（2026-10-27 发布） | `pyproject.toml`、CHANGELOG.md（gh api 补齐） |
| 语言 | Python 3.12+（引擎 ~72k 行）+ Go（MCP server 3.7k 行） | `wc -l` 统计、`mcp/go.mod` |
| 许可证 | MIT | `LICENSE` |
| 创建/最近推送 | 2026-01-23 / 2026-10-28 | gh api |
| 贡献者 | 约 152 人（分页 100+52） | gh api contributors |
| Release | 累计 56 个；近 30 天提交 ≥100 | gh api releases、commits |

## 2. 项目是什么，解决什么问题

这是 2026 年"Agent Skills"浪潮中最成功的技能包之一（README 徽章自称 GitHub Trending 每日 #1，出处 `README.md`）。它解决的核心问题是：**各大社交平台互为围墙花园，任何单一 AI 都无法同时检索它们**——Google 不索引 Reddit 评论和 X 帖子，ChatGPT 有 Reddit 协议但搜不了 X/TikTok，Gemini 有 YouTube 但没有 Reddit（`README.md` "Why this exists"一节）。last30days 的解法是"bring your own keys"：用户自带各平台 API key、浏览器 cookie 会话或免费无键路径，由引擎并行搜索全部可达信源。

与普通搜索引擎"编辑聚合"相反，它按**人用注意力投的票**排序：Reddit 的 upvote、X 的 like、YouTube 播放与转录、Polymarket 有真钱背书的赔率（`README.md` "Sources, scored by the people"表格）。产出的不是链接列表，而是一份引用到具体帖子/评论/转录的简报，带互动数、时效权重与"KEY PATTERNS"总结。

产品形态非常规：**"技能"即产品**（`CONCEPTS.md`：The Skill is the product）。`skills/last30days/SKILL.md` 是写给宿主大模型看的"散文契约"（含 11 条 LAW 输出法规），Python 引擎 `scripts/last30days.py` 是实现；两者之间是严格的标志位/输出形状合同。作者自述最初为自己追踪 AI 圈动态而写，后扩展到销售调研、出行攻略、竞品对比等场景（`README.md`）。

除单话题研究外还有三种衍生模式：无话题的 Discovery（"什么正在爆"三段式模型裁判协议）、X vs Y 对比（每实体一个并行子运行）、以及 watchlist/briefing/library 的持续监测与本地研究库（`README.md` Configuration 节、`skills/last30days/scripts/watchlist.py`、`briefing.py`）。

## 3. 技术栈与依赖

- **Python 3.12+，运行时零第三方依赖**：`pyproject.toml` 的 `dependencies = []`——引擎全部用标准库（sqlite3、concurrent.futures、threading 等），dev 组才装 pytest/towncrier。这是刻意的：技能要被丢进任意宿主环境运行，不能假设用户装了包管理器。
- **外部工具链**（按需探测，非硬依赖）：`yt-dlp`（YouTube 搜索与转录，`scripts/lib/youtube_yt.py`）、Node.js（vendored 的 X 搜索客户端 `scripts/lib/vendor/bird-search/bird-search.mjs`，MIT）、Chrome/Safari cookie（X 免费后端，`scripts/lib/chrome_cookies.py`、`safari_cookies.py`）、macOS Keychain/pass（`scripts/setup-keychain.sh`、`setup-pass.sh`）。
- **Go + mark3labs/mcp-go**：MCP server（`mcp/cmd/last30days-pp-mcp/main.go`、`mcp/internal/tools/research.go`），把 Python 引擎以子进程方式包装成 `research` 工具，供 Claude Desktop 的 .mcpb 包使用。
- **商业 API（全部可选）**：ScrapeCreators（TikTok/Instagram/LinkedIn 等，1 万次免费）、X 官方 API、xAI、Xquik、Perplexity、Brave Search、Bright Data（Amazon 评论）等，完整矩阵见 `CONFIGURATION.md`（110KB 的配置文档）。
- **零键保底**：Reddit（公开列表+shreddit/arctic-shift 归档兜底）、HN（Algolia）、Polymarket（Gamma）、GitHub 可无任何凭据工作（`README.md` 安装节、`scripts/lib/reddit_public.py`、`reddit_arctic.py`）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 | 出处 |
|---|---|---|
| `skills/last30days/SKILL.md` | 模型可读的技能契约：分发路由、首启门、11 条输出 LAW、自检清单 | 262 行 |
| `skills/last30days/references/` | 15 个分阶段参考文档（runtime、research-runbook、synthesis、discovery、setup-wizard、comparison 等） | `ls` |
| `skills/last30days/scripts/last30days.py` | CLI 主引擎入口（4,666 行）：参数解析、Python 版本门、SIGTERM 清理、模式分发 | `wc -l` |
| `scripts/lib/` | 107 个模块：每个信源一个适配器（reddit/bird_x/youtube_yt/…）+ 流水线（pipeline/fusion/rerank/planner/freshness/grounding/cluster）+ 基建（http/subproc/env/schema） | `ls` |
| `scripts/lib/vendor/bird-search/` | vendored 的 X 网页搜索 JS 客户端（自带 query-ids/features.json） | 目录 |
| `mcp/` | Go MCP server：internal/engine（embed 引擎源码、按用户版本化缓存、子进程管理）、internal/tools（research/preflight 工具） | `mcp/internal/` |
| `.claude-plugin/` `.codex-plugin/` `.grok-plugin/` `.agents/plugins/` `gemini-extension.json` | 各宿主插件清单（lockstep 同版本 3.31.0） | 各 json |
| `tests/`（git 内，tarball 被 export-ignore 排除） | 315 个文件、287 个 test_*.py + eval 套件 | gh api git trees |
| `.github/`（同上排除） | 9 个 workflow、release 脚本、issue 模板、dependabot | gh api git trees |
| `docs/`（同上排除） | 33 个文档：how-search-works、eval 参考、solutions 案例库、release 计划 | gh api git trees |
| `changelog.d/` | towncrier 新闻片段（每 PR 一个文件） | `CONTRIBUTING.md` |

组织方式评述：单包 monolith，但"契约层（SKILL.md+references）与实现层（lib/）物理分离"是鲜明的架构决策；每个信源一个模块、共用 schema/fusion/rerank 干线，新增信源即新增一个 lib 模块。`.gitattributes` 用 export-ignore 把 tests/docs/CI 从 git archive 剔除，使 Claude Code 插件安装可以直接拉 tarball 当运行载荷（`.gitattributes` 内注释说明）。

## 5. 核心架构与关键实现

### 5.1 流水线编排（`scripts/lib/pipeline.py`，6,813 行）

v3 编排核心导入全部 60+ 信源模块（第 23-84 行 import 块）。三档深度预置预算（第 95-99 行）：

```python
DEPTH_SETTINGS = {
    "quick": {"per_stream_limit": 6, "pool_limit": 15, "rerank_limit": 12},
    "default": {"per_stream_limit": 12, "pool_limit": 40, "rerank_limit": 40},
    "deep": {"per_stream_limit": 20, "pool_limit": 60, "rerank_limit": 60},
}
```

一次运行 = 多子查询 × 多信源并行流 → 融合 → 聚类 → 重排 → 渲染。同源重复探测类流会被封顶（trustpilot/amazon/meta_ads 封顶 1，第 113-119 行注释解释了每多一流就是纯冗余+一次计费）。对比模式按实体拆成并行子运行，但共享 YouTube 取数闸与付费预算（`CONCEPTS.md` "Comparison sub-run"）。

### 5.2 加权 RRF 融合（`scripts/lib/fusion.py`）

标准 Cormack 2009 RRF（K=60）做跨流融合，关键设计是**窗口外证据永远排在窗口内之下**（第 15-27 行 `_candidate_sort_key` 第一元组位），注释直言："排了一支九个月前的视频在第一位的 last-30-days 简报违背了自己的合同"。去重用规范化 URL（剥 utm_*、www/m/old 前缀），同一 Reddit 帖的多个副本会合并并保留"更富"的那份（top_comments/transcript 等 enrichment 键，第 66-79 行）。

### 5.3 LLM 重排 + 本地兜底（`scripts/lib/rerank.py`，1,285 行）

双模式：配置了推理 provider（OPENAI/XAI/OPENROUTER/GEMINI key）时用 LLM 打分；无 key 走本地评分（Keyless path，`CONCEPTS.md`）。防御性常量很有代表性（第 13-40 行）：实体未命中罚 25 分（"典型分差 30-70，25 分足以把跑题项压下去"）；一方便主贴（first-party post）只加 5 分——"目标是停止埋没一方便帖，而不是让署名自动夺冠"。Discovery 还有带年龄权重的互动速度分：`engagement * 1/sqrt(age+1)`，7 天内 ×1.5，跨源佐证每多一源 ×1.15（第 68-98 行），并设绝对置信度下限，宁可输出"Nothing solid this window"也不把 1 个赞的推文包装成趋势（第 101-110 行注释记录了 2026-07-12 的失败案例）。

### 5.4 模型即规划者/裁判（`scripts/lib/planner.py` + SKILL.md LAW 7/11）

命名实体话题强制 `--plan <file>`：由宿主推理模型生成 JSON 查询计划（handle/subreddit/hashtag 解析），无外部 planner key；MCP 场景才允许引擎内部规划（`mcp/internal/tools/research.go` 第 28-32 行 `LAST30DAYS_ALLOW_ENGINE_PLAN=1` 注释解释了 LAW 7 门）。Discovery 则是三段协议：引擎 nominate → 宿主模型写 judgments → 引擎 finalize（`SKILL.md` LAW 11、`CONCEPTS.md` "Handoff checkpoint"——checkpoint 带 bundle id 与 TTL，身份不符报错而非重跑昂贵段）。

### 5.5 X 信源的后端链与宿主让渡（`scripts/lib/bird_x.py` 等六个后端模块）

X 是最"军备竞赛"的信源，引擎为其维护一条有序后端链：免费浏览器 cookie 会话优先（bird-search JS 客户端 / Chrome、Safari、Dia 浏览器），其后是 xAI API、Xquik、xurl CLI、官方 X API bearer（`CONCEPTS.md` "X backend chain"、"Opt-in backend"——Grok CLI 与官方 API 默认不进链，防止环境里残留凭据静默消耗计费额度）。另支持"宿主自取让渡"：宿主模型用自己的 X 连接器取帖，以 `--x-posts` 信封文件喂给引擎；信封顶层严格（格式/过期/跑题即失败）、逐行宽松（坏行丢弃计数），且引用 URL 一律由引擎按帖子 id 重建、绝不信信封自带的链接（`CONCEPTS.md` "Host-fetched lane"、`SKILL.md` Security 节）。Grok Bot 宿主强制官方许可后端链、禁读浏览器会话（`SKILL.md` GROK BOT HOST RULE）。

### 5.6 可验证新鲜度（`scripts/lib/freshness.py`）

对简报中的事实性断言（排名、星数、赔率）抽取 `Claim` 并用 refetcher 复核（`freshness.py` 第 15-38 行 dataclass），属于对"AI 报告数据过期"问题的工程化回答。

### 5.7 Go MCP 外壳（`mcp/`）

MCP server 把引擎源码 embed 进二进制（`mcp/internal/engine/embed.go`），运行时解压到按版本隔离的用户缓存（`cache_staging.go`、`EnsureUserCache`），再调本机 Python 3.12+ 执行（`python.go`、`run.go`），带缓存锁（`cache_lock_windows.go` 等平台分文件）与进程组清理（`proc_group_unix.go`）。工具面刻意只有两个：`research` 与 `preflight`（`mcp/internal/tools/research.go`、`preflight.go`）。

## 6. 构建与运行

```bash
# 安装（Claude Code 推荐，marketplace 自动更新）
/plugin marketplace add mvanhorn/last30days-skill && /plugin install last30days
# Codex/Cursor/Copilot/Gemini CLI 等 50+ 宿主
npx skills add mvanhorn/last30days-skill -g

# 直跑引擎（脚本/cron/开发用）
python3 skills/last30days/scripts/last30days.py "AI coding agents" --emit=compact
python3 skills/last30days/scripts/last30days.py --preflight        # 权限检查器
python3 skills/last30days/scripts/last30days.py --discover "AI agents"  # 趋势发现

# 开发
uv sync --group dev && uv run pytest          # 全量测试
uv run pytest --cov                           # 覆盖率（门槛 fail_under=81）
bash skills/last30days/scripts/build-skill.sh # 构建 claude.ai 用的 .skill 包
uv run python .github/scripts/prepare_release.py --bump patch  # 维护者发版
```
出处：`README.md`、`CONTRIBUTING.md`、`AGENTS.md` Commands 节、`pyproject.toml`。

## 7. 测试与工程质量

- **规模罕见地大**：git HEAD 上 `tests/` 315 个文件、287 个 test_*.py（gh api git trees）；覆盖每个信源适配器（test_reddit_*、test_youtube_*、test_x_url 系列等逐文件对应）。
- **测试金字塔**：单测 + `tests/eval/` 质量评估套件（8 类话题 fixture：person/tech-product/comparison/breaking-event/emerging-event/niche/non-english-cjk，带 baseline.json 与 harness，`tests/eval/`）+ `tests/e2e_comparison.py` + `tests/hermes/test_no_critical_scan_findings.py`（安全扫描门）+ `tests/network_guard.py`（防测试外呼）+ `tests/skill_contract.py`（校验 SKILL.md 契约与引擎一致）。
- **CI 9 个 workflow**（gh api trees）：validate（pytest `-n auto --dist worksteal` + coverage + Windows 子进程专项 job + eval job，`validate.yml`，所有 action 按commit SHA 锁定）、changelog-guard（强制 changelog 片段）、osv-scanner、scorecard、zizmor（workflow 安全审计）、security、release/tag-release/prepare-release（towncrier lockstep 发版）。
- **覆盖率有据可查**：`pyproject.toml` 第 76-84 行注释记录 2026-09-23 实测 82.28%，门槛 81 且"只升不降、下调需在 PR 说明"。仓库无 linter 配置文件在 tarball 中（ruff noqa 仅为局部豁免，未确认全局 lint 设置）。
- 代码质量观察：关键常量与阈值几乎都带事故出处注释（日期 + 失败案例），信源模块彼此独立、共享 schema 层，schema-drift 已是 `source_status` 的显式状态（`SKILL.md` Per-run source outcomes 节）。

## 8. 社区与维护状态

- **维护强度极高**：创建于 2026-01-23，十个月内 66k 星、56 个 release、v3.29→v3.31 发布间隔仅数天（CHANGELOG.md），近 30 天提交 ≥100（gh api，per_page 截断）。最近推送 2026-10-28。
- **贡献者约 152 人**，但主体开发呈"作者 + 编码 agent"形态：`CONTRIBUTING.md` 开篇即写"Most PRs here are opened by coding agents following AGENTS.md"；`AGENTS.md`（16KB）是为 agent 贡献者写的完整工程规范；PR 模板含 "Agent disclosure" 项（`.github/PULL_REQUEST_TEMPLATE.md`，gh api trees）。changelog 中 issue 编号已到 #1335+，可见吞吐量。
- **分发生态经营得很广**：Claude Code marketplace、Grok 官方 marketplace（`xai-org/plugin-marketplace` 收录，README）、OpenClaw/clawhub、claude.ai 网页版 .skill、Claude Desktop .mcpb、Telegram/WhatsApp 试运行链接（README "Quick Try-Link"）；README 有 7 种语言版本。
- **数据处理立场**：本地优先——简报存 `~/Documents/Last30Days/`，watchlist 数据存本地 SQLite，rerank 外呼推理 API 属 opt-in、无 key 则纯本地评分，库发布（ht-ml.app）需显式开通（README "Data handling"）。
- 未确认： backers/赞助情况；核心 maintainer 之外是否有人有发版权（release workflow 受众未查）。

## 9. 亮点与风险

**亮点**
1. "SKILL.md 散文契约 + Python 引擎"的双层设计是 Agent Skills 产品的教科书实现：模型侧 11 条 LAW 把输出质量约束写成可执行法规，引擎侧输出形状（badge/证据块/footer）被 `tests/skill_contract.py` 固定。
2. 工程纪律与产品热度罕见地匹配：72k 行 Python 配 287 个测试文件、81% 覆盖率门、9 条 CI 流水线、towncrier lockstep 发版、SHA 锁定的 actions、多平台进程管理（SIGTERM 进程组清理见 `last30days.py` 第 59-73 行注释）。
3. 零依赖 + 零键保底 + 按需升级的准入曲线设计出色：Reddit/HN/Polymarket/GitHub 开箱即用，setup wizard 30 秒解锁其余信源（README）。
4. 排序哲学一以贯之：窗口外证据降级、实体未命中判罚、绝对置信度下限、"Nothing solid"是一等公民结果（`fusion.py`、`rerank.py`、`CONCEPTS.md` Confidence floor）。
5. 大量"从真实失败学来"的注释文化：常量旁标注事故日期与案例（rerank.py 第 17-19 行、第 101-110 行），可读性与可维护性俱佳。

**风险/局限**
1. **平台对抗性脆弱**：X 免费后端依赖浏览器 cookie 逆向客户端（vendored bird-search、`chrome_cookies.py`），Reddit 依赖公开页 + shreddit/arctic-shift 兜底（CHANGELOG 3.31.0），随时可能被平台封堵；CHANGELOG 里 YouTube 超时、Instagram 404、Reddit 数据中心 403 等修复密度很高。
2. **合规灰色地带**：读取浏览器会话抓取 X/Reddit 涉及各平台 ToS；`SKILL.md` Security 节虽列明数据流向与"不写密钥"，但无法消除上游平台风险。
3. **功能深度依赖付费/凭据**：TikTok/Instagram/LinkedIn 等走 ScrapeCreators 计费，rerank 走推理 API；免费档与付费档差距明显（README key 矩阵）。
4. 巨型单文件：`pipeline.py` 6,813 行、`last30days.py` 4,666 行，新贡献者上手成本高；Claude Desktop MCP 版 Windows 支持被明确推迟（README）。

## 10. 总结评价

last30days-skill 是"Agent Skills 元年"里把rompt/工程/分发三件事同时做到位的那一类罕见项目：产品概念（多围墙花园并行检索 + 按真实互动排序）足够锋利，工程实现（零依赖、合同测试、9 条 CI、覆盖率门）足够严肃，分发（7 宿主原生清单 + 7 语 README + .skill/.mcpb 制品）足够勤奋，三者叠加解释了十个月 66k 星。它同时是很好的两种教材：想理解"技能即产品"如何设计模型契约的，读 `SKILL.md` 与 `CONCEPTS.md`；想看大规模多信源聚合系统如何处理融合、去重、重排与预算的，读 `pipeline.py`、`fusion.py`、`rerank.py`。不适合期待纯库/框架的人——它本质是一个运行在 Agent 宿主里的研究工作流，且其长期生命力取决于与各平台对抗性接口的持续军备竞赛。
