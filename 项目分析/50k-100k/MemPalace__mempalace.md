# MemPalace/mempalace 项目分析

> 星数 59717（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：本地优先（local-first）的 AI 记忆系统——把 AI 会话与项目文件按"记忆宫殿"结构逐字存储，再用可插拔的语义检索引擎取回，默认零 API 调用。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | MemPalace/mempalace | gh api repos/MemPalace/mempalace |
| 星数 | 60245（分析时 gh api 快照；任务快照 59717） | 同上 |
| 语言 | Python（约 86k 行）+ Rust（约 1k 行） | 源码统计 |
| 许可证 | MIT | LICENSE、gh api |
| 版本 | 3.12.0 | pyproject.toml、CHANGELOG.md |
| Python 要求 | >=3.10 | pyproject.toml |
| 仓库创建 | 2026-04-05 | gh api created_at |
| 最近 push | 2026-10-31（默认分支 develop） | gh api pushed_at；tarball 文件 mtime 2026-10-31 |
| 主页 | mempalaceofficial.com | gh api homepage |

## 2. 项目是什么，解决什么问题

MemPalace 解决的是 AI 编码助手（Claude Code、Codex、Cursor 等）"会话一压缩/一结束就失忆"的问题：把历史会话逐字保留并可检索，而不是靠摘要压缩。README.md 开篇即定位："Local-first AI memory. Verbatim storage, pluggable backend, 96.6% R@5 raw on LongMemEval — zero API calls."

与"摘要式记忆"路线的根本区别在存储原则：**逐字存储（verbatim）**。CLAUDE.md 把它列为不可协商的设计原则——"Never summarize, paraphrase, or lossy-compress user data. The system searches the index and returns the original words."。检索层做语义搜索，但返回的永远是原文抽屉，而不是 LLM 生成的摘要。

组织模型借用了"记忆宫殿"（method of loci）与 Zettelkasten 卡片盒隐喻（CLAUDE.md "The Mission" 节）：人/项目成为 **wing**（翼），时间/主题成为 **room**（房间），原文块存入 **drawer**（抽屉），索引层叫 **closet**（壁柜，`topic|entities|→drawer_ids` 指针行）。关键规则是"closet 命中只提升排序、永不拦截搜索"（CLAUDE.md Architecture 节），保证索引错误不会造成内容不可达。wing 之间还有 hallway/tunnel 交叉链接（mempalace/hallways.py、mempalace/palace_graph.py）。

摄取面覆盖三类输入（mempalace/cli/__init__.py 模块 docstring）：`mempalace mine <dir>` 挖掘项目文件；`--mode convos` 挖掘会话导出（Claude Code、Claude.ai、ChatGPT、Slack）；`--mode extract` 借 MarkItDown 抽取 PDF/DOCX/PPTX/XLSX/RTF/EPUB（pyproject.toml `[extract]` extra）。另有自动保存 hooks（hooks/ 下 Claude Code、Codex、Cursor、Antigravity 四套脚本）在后台把会话逐字落盘。

隐私是其卖点也是架构约束："Nothing leaves your machine unless you opt in"（README.md）。嵌入用本地 ONNX 模型（minilm 或 embeddinggemma-300m），知识图谱用本地 SQLite，LLM 精修只对接用户显式配置的本地或 BYOK 服务（CLAUDE.md Design Principles）。

项目背景有一段独特的"纠错史"：仓库 2026-04-05 创建，上线数小时内社区即抓出 README 基准表问题，作者随后撤回"30x 无损压缩""+34% palace boost"等声明，并在 docs/HISTORY.md 建立公开勘误账本；2026-04-14 社区审计（issue #875）修正了把检索召回率与竞品端到端 QA 准确率混排的错误。这种把"公开认错"制度化的做法在同类项目中少见，也部分解释了社区信任与星数增长。

## 3. 技术栈与依赖

- 主语言 Python 3.10+（约 157 个 .py、86k 行，`find mempalace -name "*.py" | xargs wc -l`）；可选 Rust 加速（crates/ 工作区，mempalace-core / mempalace-py(PyO3) / mempalace-cli，约 1k 行 .rs）。
- 核心依赖（pyproject.toml `[project].dependencies`）：chromadb>=1.5.4（默认向量库）、numpy、huggingface_hub + tokenizers（拉取 ONNX 嵌入模型，惰性下载）、pyyaml、python-dateutil。
- 构建：hatchling（pyproject.toml `[build-system]`）；仓库带 uv.lock，README 推荐 `uv tool install mempalace` 隔离安装。
- 嵌入层（mempalace/embedding.py）：minilm（默认，all-MiniLM-L6-v2，384 维，英文）/ embeddinggemma（300m ONNX，多语言，约 300MB 惰性下载）/ openai-compat（任意 OpenAI 兼容 /v1/embeddings 端点，LM Studio、vLLM、Ollama 等）；设备支持 auto/cpu/cuda/coreml/dml，CoreML 会静默产出非法向量，代码里明确将其从 embeddinggemma 的 auto 选择中剔除。
- 可选 extras（pyproject.toml）：milvus（pymilvus + milvus-lite）、pgvector（psycopg[binary]）、gpu/dml/coreml（onnxruntime 变体）、extract（markitdown + striprtf）、embeddinggemma2 / multimodal（torch + transformers + sentence-transformers）、spellcheck。
- 质量工具链：ruff==0.16.6（check + format，三处 pin 有测试保证一致，CI lint job 与 pyproject 注释）、pytest + pytest-cov + hypothesis（属性测试）+ pre-commit + mypy（未进 CI 门禁，见 pyproject dev extra 注释）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| mempalace/ | 主包。cli/、searcher/、palace/、mcp_server/、backends/ 五个子包 + 矿工/嵌入/知识图谱等约 100 个模块 |
| crates/ | Rust 原生向量引擎：mempalace-core（解码+Rayon 并行余弦扫描）、mempalace-py（PyO3）、mempalace-cli（独立可执行文件） |
| tests/ | 298 个测试文件、约 144k 行（含 tests/benchmarks/ 性能基准与 tests/mcp/ 协议测试） |
| benchmarks/ | LongMemEval/LoCoMo/ConvoMem/MemBench 复现脚本 + 提交的结果 JSONL（results_*.jsonl） |
| hooks/、skills/、commands/、rules/ | Claude Code/Codex/Cursor/Antigravity 钩子脚本与三个 Agent Skill（mempalace、mempalace-recall、mempalace-task） |
| .claude-plugin/ 等五个插件目录 | 按 agent 分发的插件清单（claude/codex/cursor/antigravity/dsh） |
| website/、landing/、deploy/ | 官网源（32 个 md 页）、落地页、服务端 docker-compose |
| docs/ | RFC（001 存储后端插件、002 源适配器、003 logstream 协同、004 副本宫殿、005 agent 身份路由）、HISTORY.md 勘误账本、schema.sql |
| .github/ | ci.yml（三平台 + Milvus + Rust 五类 job）、issue/PR 模板、CODEOWNERS、dependabot |

组织方式评述：单包单体 + 少量原生扩展，不是 monorepo。最非常规的是**碎片 exec 架构**：cli/、searcher/、palace/、mcp_server/ 四个包的 `__init__.py` 按依赖顺序把 `_*.py`/`cmd_*.py`/`tools_*.py` 碎片 exec 进包命名空间（mempalace/cli/__init__.py 注释自述：避免"一个 4000 行模块"，同时保留 `from mempalace.cli import main` 与测试 monkeypatch 兼容；pyproject.toml `[tool.ruff.lint.per-file-ignores]` 为此豁免 F821）。可读性上是双刃剑——跳转与静态分析体验受损。

## 5. 核心架构与关键实现

### 5.1 存储后端契约（mempalace/backends/base.py，982 行）

RFC 001 的产物：`BaseCollection`/`BaseBackend` 两个 ABC + 类型化 `QueryResult`/`GetResult` 数据类，取代 Chroma 的 dict 返回形态；错误体系统一（`PalaceNotFoundError` 故意继承 `FileNotFoundError` 以兼容旧调用方）。六个后端经 Python entry-point 组 `mempalace.backends` 注册（pyproject.toml）：chroma（默认）、sqlite_exact（NumPy 精确检索）、rust_exact（Rust 引擎，复用同一 sqlite 文件零迁移）、milvus、qdrant、pgvector。后端必须显式声明能力（`UnsupportedFilterError`：静默丢弃未知过滤算子被规范禁止）与维护类型，并有统一符合性套件（tests/test_backend_conformance.py、tests/_backend_conformance.py）跨后端跑。

### 5.2 分层唤醒栈（mempalace/layers.py）

```python
# layers.py docstring
Layer 0: Identity        (~100 tokens) — Always loaded. "Who am I?"
Layer 1: Essential Story (~500-800)    — Always loaded. Top moments.
Layer 2: On-Demand       (~200-500 each) — Loaded when a topic/wing comes up.
Layer 3: Deep Search     (unlimited)   — Full semantic search.
```

`mempalace wake-up` 只注入 L0+L1（约 600-900 token），L2 按话题/wing 现取，L3 才是全量语义搜索——这是控制上下文预算的核心设计，所有读操作经 `read_only=True` 打开（layers.py `_open_for_read` 注释）。

### 5.3 矿工与增量摄取（mempalace/miner.py，2541 行）

项目矿工读取 mempalace.yaml 确定 wing/room，逐字切块成 drawer，写入前经 `sanitize_name()`/`sanitize_content()` 校验（mempalace/config.py）。关键安全设计是**增量与崩溃安全**：append-only 摄取、mine 锁（palace/mine_lock.py、palace_lock.py）、WAL（mempalace/wal.py）、失败不得破坏已有宫殿（CLAUDE.md "Incremental only"）；v3.12.0 CHANGELOG 特别说明"write 失败一半时不能留下没有 drawer 的文件"。会话侧 convo_miner.py（1171 行）+ normalize.py 做多格式 transcript 检测归一化，sweeper.py 支持逐消息幂等回填。

### 5.4 混合检索与时间知识图谱（mempalace/searcher/ + mempalace/knowledge_graph.py）

searcher/ 拆为 candidates、ranking（BM25 停用词缓存、语言规范化）、filters、sqlite_bm25、query、render 等碎片。README 基准：LongMemEval 原始语义检索 R@5 96.6%（无 LLM）；Hybrid v4 在未见过的 450 题上 98.4%（关键词加权 + 时间邻近加权 + 偏好模式抽取）；LLM rerank 后 >=99%，但 README 明确拒绝宣传"100%"——因为最后 0.6% 是对着错题调的，BENCHMARKS.md 自我标注"teaching to the test"。knowledge_graph.py（1269 行）实现带 `valid_from`/`valid_to` 有效期窗口的实体-谓词-实体三元组，纯本地 SQLite，MCP 暴露 add/query/invalidate/timeline 工具。

### 5.5 MCP 服务与 agent 协同（mempalace/mcp_server/）

`schemas.py` 实测恰好 47 个 `mempalace_*` 工具（正则统计去重），覆盖读写、知识图谱、跨 wing 导航、抽屉管理、agent 日记与 logstream 协同，与 README 声明一致。入口三个（pyproject `[project.scripts]`）：`mempalace-mcp`（代理模式）、`mempalace-light-mcp`（轻量服务）、CLI 本体。RFC 003/004/005 定义了多 agent 共享"大脑中枢"（hub）的 logstream 事件与制品交接协议，v3.9.0 发布名即"one hub for every agent"（CHANGELOG.md）。

## 6. 构建与运行

来自 CLAUDE.md 与 README.md 的实际命令：

```bash
uv sync --extra dev                 # 安装开发环境（或 pip install -e ".[dev]"）
uv run pytest tests/ -v --ignore=tests/benchmarks          # 测试
uv run pytest ... --cov=mempalace --cov-report=term-missing
uv run ruff check . && uv run ruff format .                # lint/format

uv tool install mempalace           # 安装 CLI（隔离环境）
mempalace init ~/projects/myapp     # 初始化宫殿
mempalace mine ~/projects/myapp     # 挖掘项目文件
mempalace mine ~/.claude/projects/ --mode convos           # 挖掘 Claude 会话
mempalace search "why did we switch to GraphQL"            # 搜索
mempalace wake-up                   # 注入 L0+L1 上下文
mempalace audit                     # 宫殿组织度评分（只读）

docker run -i --rm -v mempalace-data:/data ghcr.io/mempalace/mempalace   # MCP stdio
```

Rust 侧（crates/README.md）：`pip install ./crates/mempalace-py` 构建原生轮子；`cargo test -p mempalace-core -p mempalace-cli --locked`；`mempalace-native bench --db ...` 自测性能（官方未公布本版本数字，早前测量因正确性加固已撤回——又一处自我纠错的痕迹）。

## 7. 测试与工程质量

- 规模：tests/ 298 个 .py、约 144k 行，是源码体量（86k 行）的约 1.7 倍；pytest 配置默认排除 benchmark/slow/stress 三类标记（pyproject.toml `[tool.pytest.ini_options]`）。
- 后端符合性测试是结构化的一等公民（tests/test_backend_conformance.py + _backend_conformance.py），六后端同跑；MCP 有独立协议/工具/守卫测试目录（tests/mcp/，11 个文件）。
- 覆盖率门禁：包级 fail_under=85（Windows 降为 80，因 ChromaDB 文件锁清理遗留未执行行；pyproject `[tool.mempalace.coverage]` 注释），milvus.py 单列一个 CI job 测量（.github/workflows/ci.yml test-milvus）。
- CI（ci.yml）：Linux 3.10/3.11/3.13 矩阵 + Windows + macOS + Milvus + lint + Rust（3 平台，clippy `-D warnings`，`MEMPALACE_REQUIRE_NATIVE=1` 强制原生路径过符合性测试）。Windows job 只对 ChromaDB HNSW 两种已知瞬时错误做 scoped 重试（`--only-rerun`），真失败首轮即红。
- 细节质量高：lint 工具三处 pin 一致性有专门测试（test_ruff_pins_match，CI lint job 注释记载了一次漂移事故）；hypothesis 属性测试、pytest-rerunfailures 的使用动机都在 pyproject 注释里写明。
- 观察到的代价：碎片 exec 架构使四个子包依赖 F821 豁免（静态分析器视为未定义名）；ruff `max-complexity = 25` 相对宽松。

## 8. 社区与维护状态

- 活跃度：创建于 2026-04-05，最近 push 2026-10-31（gh api）；约 6 个月内从零到 60k+ 星，增长极快。CHANGELOG 记录 20+ 个版本，GitHub releases 约每 2-4 周一个（v3.11.0 于 2026-10-14，v3.9/3.8/3.7.1 依次 2026-08~09，gh api releases），节奏稳定。
- 贡献者：gh api contributors 返回 30 个条目；open_issues 819。作者用笔名（pyproject.toml `authors = {name = "milla-jovovich"}`；MISSION.md 署名同），联合创始人 @bensig 承担后端工程（MISSION.md）。贡献者结构集中在极少数核心成员，属典型创始人驱动项目。
- README 顶部 CAUTION 警告存在仿冒网站（mempalace.tech 等）分发恶意软件，docs/HISTORY.md 是官方公开勘误账本（2026-04-07 致歉信、04-11 仿冒域名、04-14 基准表重写三个条目）。
- Discord 社区（README 徽章）、CONTRIBUTING.md + PR 模板 + CODEOWNERS 齐全；不接受总结化存储、云同步、遥测类贡献（CLAUDE.md Contributing）。

## 9. 亮点与风险

亮点：
1. 逐字存储 + "索引只加权不拦截"的架构承诺，把检索召回做成可验证的第一性指标（benchmarks/ 结果文件随仓库提交，可复现）。
2. 工程纪律罕见地强：85% 覆盖率门禁、跨后端符合性套件、三平台 + Rust CI、公开勘误账本（docs/HISTORY.md）、RFC 流程（docs/rfcs/ 5 篇）。
3. 可插拔后端契约设计成熟（base.py 类型化 ABC + entry-point 注册 + 能力/维护声明 + 统一错误语义），六后端零迁移切换（rust_exact 与 sqlite_exact 共盘）。
4. 本地优先隐私做得彻底：默认无 API key、无遥测、嵌入/图谱全本地，GPU/DML/CoreML 加速路径齐全。
5. 发行面完整：PyPI、多架构 Docker、GPU 镜像、原生 Rust 轮子、三个 Agent Skill、五家 agent 插件清单。

风险/局限：
1. 项目仅约 6 个月历史，已到 3.12.0 且有破坏性变更（Python 3.10 门槛、时间戳语义改为 UTC 比较等，v3.12.0 CHANGELOG），API/磁盘格式稳定性尚待时间检验。
2. 星数增速与项目年龄不成比例，且作者匿名化运营，README 自曝仿冒站问题——采用前建议自行验证分发包来源。
3. 基准数字均为自报，虽有 held-out 集与可复现脚本背书，且项目自身有过度声明前科（已修正）；与竞品无可比口径的直接对比（README 明确拒绝横向表）。
4. 碎片 exec 架构对 IDE 跳转、mypy、外部贡献者不友好（四个子包依赖 F821 豁免）；819 个 open issue 显示支持负担在快速累积。

## 10. 总结评价

MemPalace 是 2026 年"AI 记忆"赛道中工程完成度与自我纠错透明度都突出的新星：以"逐字存储、永不摘要"为宪法，围绕 wing/room/drawer/closet 的宫殿模型与六后端可插拔检索，把 Claude Code 等 agent 的长期记忆做成了 CLI + MCP + hooks + skills 的全套发行面，并用 144k 行测试、85% 覆盖率门禁和公开勘误账本支撑其可信度声明。它不是被验证过多年的基础设施，版本与磁盘格式仍在快速演化，星数增长亦需冷静看待——但其代码库本身是研究"如何把隐私优先的记忆系统做成严肃工程"的优质样本。适合两类人：想为 agent 装持久记忆、且在意数据不出本机的个人开发者（直接用）；想学习可插拔后端契约、覆盖率门禁与符合性测试如何落地的 Python 工程师（读源码，从 mempalace/backends/base.py、mempalace/layers.py 与 tests/_backend_conformance.py 入手）。
