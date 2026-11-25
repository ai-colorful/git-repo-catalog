# Graphify-Labs/graphify 项目分析

> 星数 133639（快照 2026-10-24）｜主语言 Python｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：**把任意代码库（连同文档、SQL、配置、PDF、图片、音视频）在本地解析成一张可查询的知识图谱，以 `/graphify` 技能的形式嵌入 Claude Code、Cursor、Codex 等 20+ AI 编程助手，用"查图谱"替代"grep 全仓"。**

| 项目 | 事实 |
|---|---|
| 名称 | Graphify-Labs/graphify（默认分支 `v8`，gh api 确认） |
| 星数 | 133,639（快照）；2026-10-30 实测 138,600（gh api） |
| 主语言 | Python（包内约 90,797 行，`wc -l` 实测） |
| 许可证 | Apache-2.0（另有 LICENSE-MIT 与 NOTICE，`LICENSE`/`LICENSE-MIT`） |
| 创建时间 | 2026-04-03（gh api），约 7 个月冲到 13 万+ 星 |
| 最近活跃 | 2026-10-27 发布 v0.9.97 并 push（CHANGELOG.md 首行 + gh api releases） |
| 背景 | YC S26 公司 Graphify Labs（README.md 徽章），有商业云产品 Graphify Cloud |

## 2. 项目是什么，解决什么问题

AI 编程助手回答代码库问题时，最常见的做法是反复 grep、逐个读文件，token 消耗大且容易漏掉跨文件关系。graphify 的方案是：把整个项目预先解析成一张**持久化的知识图谱**——节点是概念（类、函数、文档章节、论文概念），边是关系（`calls`/`imports`/`inherits`/`uses` 等），之后助手通过 `graphify query`（自然语言提问返回子图）、`graphify path`（两概念间最短路径）、`graphify explain`（解释单个节点）来检索，而不是每次重读源码（README.md "Common commands"、`graphify/skill.md`）。

它刻意与主流 RAG 划清界限：**不用 embedding、不建向量库**。代码部分完全由 tree-sitter AST 确定性解析，"构建图谱零 LLM 花费、代码不出本机"是核心卖点（README.md 开头三条要点、BENCHMARKS.md）。每条边都带置信标签：`EXTRACTED`（源码中显式存在）、`INFERRED`（由解析器推断）、`AMBIGUOUS`（标记待人工复核），让用户能区分"读到的事实"和"推断的猜测"（ARCHITECTURE.md "Confidence labels"）。

一次运行产出三个文件：`graph.html`（可点击的力导向图）、`GRAPH_REPORT.md`（要点报告）、`graph.json`（完整图谱，schema 带版本号便于集成方检测格式变化）（README.md "Get started"、`graphify/export.py`）。

覆盖面也超出代码：文档、PDF、图片、音视频都会进入同一张图。音视频用 faster-whisper 本地转写（`docs/how-it-works.md` Pass 2），文档/图片走可选的 LLM 语义抽取（Pass 3，需配置 API key）。SQL 模式、`# NOTE:`/`# WHY:` 注释、ADR/RFC 引用都被提升为一等节点（README.md "What it does" 表格）。

商业闭环清晰：本地 CLI 免费开源，Graphify Cloud（app.graphify.com）提供常驻更新、跨仓库图谱、形式化验证、Sentry/Jira 连接器等付费能力——README 中有大量 Cloud 推广入口。

"常驻"是它区别于一次性建图工具的关键设计：`graphify install` 会按平台写入 Hook（Claude Code/Gemini CLI 的 PreToolUse 钩子，在助手发起搜索类工具调用前自动触发）或指令文件（Codex 的 AGENTS.md、Cursor 的 alwaysApply 规则等），把"先查图谱再 grep"变成助手的默认行为，并用 `graphify watch` 在文件变动时自动重建（README.md "Make your assistant always use the graph"、`graphify/hooks.py`、`graphify/watch.py`）。

## 3. 技术栈与依赖

（来源：`pyproject.toml`）

- **核心**：Python 3.10+，setuptools 构建，uv 管理依赖（uv.lock 达 1.4MB）；CLI 入口 `graphify = graphify.__main__:main`、`graphify-mcp = graphify.serve:_main`
- **图计算**：`networkx>=3.4`（图数据结构）、`numpy`、`rapidfuzz`（模糊匹配）、Leiden 社区发现可选 `graspologic`/`graspologic-native`（`graphify/cluster.py` 直接调 Rust 原生 leiden 绕过 graspologic 的 7-19s 导入开销，注释写明实测数据）
- **AST 解析**：核心依赖内置 27 个 tree-sitter 语法包（python/js/ts/go/rust/java/c/cpp/ruby/csharp/kotlin/scala/php/swift/lua/zig/elixir 等，pyproject.toml L28-54），运行时锁定 `tree-sitter>=0.25` 以匹配 ABI 15 的语法包（注释详述踩坑原因）
- **可选 extras**（约 25 组）：`mcp`（mcp + starlette + anyio，锁定在 CVE 修复版本之上）、`neo4j`/`falkordb`（图库推送）、`pdf`/`office`/`google`、`video`（faster-whisper + yt-dlp）、`openai`/`anthropic`/`gemini`/`ollama`/`bedrock`（语义层后端）、`leiden`、`watch`、`sql`/`postgres`、`chinese`（jieba 分词）等
- **工程质量工具**：pytest、hypothesis、ruff（保守基线 E9/F63/F7/F82）、pyright（basic）、bandit、pip-audit、pre-commit（pyproject.toml dev 组）

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `graphify/` | 主包：流水线各阶段 + CLI + 20+ 平台的 skill 文档（`skill*.md`）作为包数据分发 |
| `graphify/extractors/` | 38 个文件，按语言拆分的 AST 抽取器（`MIGRATION.md` 记录从 extract.py 迁出的进度） |
| `graphify/exporters/` | 导出器子包 |
| `graphify/always_on/`、`graphify/skills/` | 各助手平台的"常驻注入"指令块与 references 边车文档 |
| `graphify/pi/` | Pi 助手的原生 guard 扩展（package.json + index.ts，TypeScript） |
| `tools/skillgen/` | skill 文档生成器：skill 文件由片段生成，禁止手改（CI 有 --check 校验） |
| `tests/` | 472 个测试文件、约 14.3 万行，另有 `fixtures/` 语言样例与 `pi.test.ts` |
| `worked/` | 演示/基准语料（httpx、karpathy-repos、mixed-corpus 等） |
| `docs/` | how-it-works、33 种语言的 README 翻译、RFC 草稿 |
| `ARCHITECTURE.md`、`BENCHMARKS.md`、`CHANGELOG.md`（569KB，240 个版本） | 架构文档、基准测试、变更日志 |

组织方式：**单包单体**。核心流水线模块化清晰（每阶段一个模块），但 `graphify/extract.py`（11,513 行）与 `graphify/extractors/engine.py`（10,869 行）是两个巨型模块，项目正通过 `extractors/MIGRATION.md` 记录的方式把语言抽取器逐步搬出 monolith。skill 文档采用"片段源 → 生成器 → 产物"的代码生成模式，用 CI 防止手改产物漂移（`.github/workflows/ci.yml` skillgen-check 任务）。

## 5. 核心架构与关键实现

**流水线主线**（`ARCHITECTURE.md`，签名由 `tests/test_architecture_doc.py` 导入校验，文档不会漂移）：

```
detect() → extract() → build() → cluster() → analyze helpers → report.generate() → export.to_*()
```

各阶段以普通 dict 和 NetworkX 图通信，基本无共享状态（仅 extractor 依赖环境式的扫描根状态）。

**三遍抽取**（`docs/how-it-works.md`）：Pass 1 代码结构（tree-sitter 本地解析，25+ 语言，纯确定性）；Pass 2 音视频（faster-whisper 本地转写，转写 prompt 用当前 god nodes 做领域种子）；Pass 3 文档/图片（LLM 子代理并行输出 JSON 片段再合并，纯代码语料时整段跳过）。社区发现直接用图结构当相似度信号，"无 embedding 步骤"。

**build.py 的三层去重**（`graphify/build.py` 文件头注释）：

```python
# 1. Within a file (AST): each extractor tracks a `seen_ids` set.
# 2. Between files (build): NetworkX G.add_node() is idempotent —
#    Nodes are added in extraction order (AST first, then semantic),
#    so if the same entity is extracted by both passes the semantic node
#    silently overwrites the AST node. This is intentional.
# 3. Semantic merge (skill): before calling build(), the skill merges cached
#    and new semantic results using an explicit `seen` set keyed on node["id"].
```

AST 节点保精确行号、语义节点保富标签，冲突时语义层覆盖——这是有意设计并在注释中言明。

**Leiden 的性能取舍**（`graphify/cluster.py` `_native_leiden`）：不走 graspologic 的 Python 包装（其导入链会 numba-JIT 编译用不到的 umap，实测 7-19 秒），而是直接调用 `graspologic_native.leiden()`，失败时逐级回退到 graspologic.partition.leiden 再到 networkx Louvain。还用 `_suppress_output()` 拦截原生库的 ANSI 输出，避免弄花 Windows PowerShell 5.1 的滚动缓冲。

**边方向的持久化约定**（`ARCHITECTURE.md` "Edge direction"）：`graph.json` 声明 `"directed": false` 但边实际有序——方向藏在 arc order 与 `_src`/`_tgt` 标记中，必须经 `paths.restore_arc_direction()` 读取，否则 networkx 的无向加载会按节点位置重排端点。文档把这一坑点写得非常直白。

**安全层**（`graphify/security.py` + `SECURITY.md`）：所有外部输入过统一入口——`validate_url`（仅 http/https，禁 file:// 重定向）、`safe_fetch`（限尺寸/超时）、`validate_graph_path`（限定在 graphify-out/ 内）、`sanitize_label`（去控制字符、限 256 字符）；最新版本还在缓存写入前统一做凭据脱敏（CHANGELOG 0.9.97 "rationale redaction"）。

## 6. 构建与运行

（来源：README.md、pyproject.toml）

```bash
# 用户安装（PyPI 包名是双 y 的 graphifyy，README 特别警告其他 graphify* 为仿冒）
uv tool install graphifyy          # 或 pipx install graphifyy
graphify install                   # 探测已装助手并注册 /graphify 技能
# 可选能力按需加 extras，如：uv tool install "graphifyy[all]"

# 使用（在 AI 助手中输入）
/graphify .                        # 全量建图
/graphify ./docs --update          # 增量重抽取变更文件
/graphify . --cluster-only         # 只重跑聚类
graphify query "<question>" / graphify path A B / graphify explain X

# 本地开发
uv sync                            # 依赖锁定在 uv.lock
pytest tests/ -q                   # ARCHITECTURE.md 声明的标准测试命令
pre-commit run --all-files
```

发布：GitHub Release 触发 `publish.yml`，以 OIDC trusted publishing 无 token 发布到 PyPI（`.github/workflows/publish.yml` 头注释）。

## 7. 测试与工程质量

- **规模**：472 个测试文件、约 142,968 行（`wc -l` 实测），另有 fixtures 目录与 TypeScript 写的 `tests/pi.test.ts`。每模块一个测试文件是明文约定（ARCHITECTURE.md "Testing"）。
- **风格**：单元测试为主，"避免网络访问与不受控的文件系统副作用，大多用临时目录与环境夹具隔离"（同上）；dev 依赖里有 hypothesis（属性测试）。覆盖从语言抽取器逐语言到 CLI 行为，颗粒度非常细——CHANGELOG 里几乎每个 fix 都对应测试。
- **自校验文档**：`tests/test_architecture_doc.py` 导入 ARCHITECTURE.md 中列出的每个符号，架构文档与代码不可能漂移（ARCHITECTURE.md 第 15 行明说）——这是比较少见的高质量实践。
- **CI**（`.github/workflows/ci.yml`）：skillgen 产物一致性检查（含 coverage 审计、monolith 往返校验）+ 测试矩阵（Python 3.10/3.12/3.13/3.14），分支列表 v1..v8 表明主分支随大版本滚动更名。
- **质量观察**：ruff 仅启用 E9/F63/F7/F82 极保守规则集（自述"在上游采用更广 lint 政策前保持保守基线"）；pyright basic 模式；bandit + pip-audit 进 dev 依赖；SECURITY.md 有完整威胁模型。安全响应很快（CHANGELOG 连续多个版本做依赖 CVE 地板抬升与凭据脱敏加固）。

## 8. 社区与维护状态

- **活跃度**：极高。CHANGELOG 共 240 个版本，最近 8 天连发 5 个 release（v0.9.93 10-20 至 v0.9.97 10-27，gh api releases），最后 push 2026-10-27（gh api）。
- **贡献者**：结构呈"核心 + 大量外部贡献者"——榜首 safishamsi 1,224 次提交，第二名 ayushcodes10 178 次，随后 rajatnagda45、abhay-codes07 等（gh api contributors）。CHANGELOG 每个 fix 都具名感谢贡献者，仅最近两版就出现 15+ 个不同 GitHub 用户名，外部贡献相当活跃。总贡献者数未确认（未翻全分页）。
- **社区热度**：1,652 个 open issues、13,215 forks、410 watchers（gh api）；有 Discord、YouTube、LinkedIn、讨论区 Q&A 模板（`.github/DISCUSSION_TEMPLATE/`）、`welcome.yml` 自动欢迎工作流。
- **增长背景**：仓库 2026-04-03 创建，约 7 个月达 13.8 万星，README 挂 trendshift 趋势榜徽章并维护 33 种语言翻译——增长与"AI 助手技能生态"红利和 YC S26 的市场投入强相关。
- **版本状态**：仍处 0.x（v0.9.97），API/格式尚有变动空间（graph.json 靠 `graph.schema_version` 做兼容检测，README 有说明）。

## 9. 亮点与风险

**亮点**

1. **差异化定位精准**：不堆 embedding 的确定性图谱路线，"0 LLM credits 建图 + 每条边可解释"切中了 RAG 黑箱痛点（README.md、BENCHMARKS.md：LOCOMO recall@10 0.497 约为 mem0 的 10 倍，构建成本为零）。
2. **工程质量超出同类**：14.3 万行测试、自校验架构文档、skill 产物代码生成 + CI 防漂移、统一安全入口与凭据脱敏（`tests/`、`tests/test_architecture_doc.py`、`tools/skillgen/`、`graphify/security.py`）。
3. **迭代速度罕见**：7 个月 240 个版本、平均 1-3 天一个 release，每个 fix 都标注 issue 号与贡献者，维护纪律性极强（CHANGELOG.md）。
4. **平台覆盖面广**：20+ AI 助手的技能注册、Hook/指令文件双机制常驻化（`graphify/skill*.md`、`always_on/`），吃满整个 agent 生态入口。
5. **文档坦诚**：ARCHITECTURE.md 主动暴露边方向坑点、extract() root 推断陷阱等设计取舍，BENCHMARKS.md 公开复现命令与评委一致性指标。

**风险/局限**

1. **巨型模块债务**：`graphify/extract.py` 11.5k 行、`extractors/engine.py` 10.9k 行，迁移工作正在进行但远未完成（`extractors/MIGRATION.md`），改动热点文件的心理成本高。
2. **0.x 阶段 + 爆发式迭代**：格式与行为变化快（AST cache 已经历 schema 7→9 多次淘汰，ARCHITECTURE.md），重度集成方需要频繁跟进。
3. **商业化侵蚀观感**：README 多处硬性 Cloud 推广 CTA，开源 CLI 明显是云产品的获客漏斗；长期需关注开源版与云版的功能边界。
4. **热度与成熟度的错位风险**：7 个月 13.8 万星、1.3 万 fork 的增长曲线偏营销驱动（trendshift、33 语翻译、基准对比），代码年轻，长期留存有待验证——此为推断，非文件证据。

## 10. 总结评价

graphify 是 AI 编程助手浪潮里完成度很高的一个开源工具：用 tree-sitter 确定性 AST 把代码库变成"每条边都有出处"的知识图谱，以 0 LLM 成本、全本地、无向量库的姿态差异化于 GraphRAG/memory 赛道的 embedding 系方案，再以 20+ 平台技能注册和 1-3 天一个 release 的迭代速度快速铺开。工程侧（测试规模、文档自校验、安全分层）明显高于同类新项目，代价是 extract.py 级别的单体债务和 0.x 阶段的格式动荡。适合三类人：想给 AI 助手配"代码库地图"以省 token 的日常用户；做 GraphRAG/memory 系统想找无 embedding 对照方案的工程师；以及想学习"技能即代码 + 生成器防漂移 + 自校验文档"这套工程实践的贡献者。读源码建议从 `ARCHITECTURE.md` 入手，按 detect → extract → build → cluster 的流水线顺序对照 `graphify/` 下同名模块阅读。
