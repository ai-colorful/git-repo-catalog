# pathwaycom/serviette 项目分析

> 星数 58776（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：**serviette 是 Pathway 团队推出的"无代码、实时同步"RAG 服务器**——指向一个文档目录，用一份 YAML 选定向量数据库与 embedder，两条命令即得到一个带网页聊天界面和 REST API 的问答服务，文档的增删改在数秒内自动反映到答案中（出处：`README.md`、`pyproject.toml` description）。

| 基本事实 | 值 | 出处 |
|---|---|---|
| 名称 | pathwaycom/serviette（前身 `pathwaycom/llm-app`，GitHub API 对旧名做重定向确认） | gh api、release "serviette 0.1.8 — llm-app is now serviette" |
| 星数 | 58,732（2026-11-03 gh api 实测；任务快照 58,776） | gh api repos/pathwaycom/serviette |
| 语言 | Python（≥3.10，支持到 3.14） | `pyproject.toml` |
| 许可证 | MIT（Copyright (c) 2026 Sergey Kulik） | `LICENSE` |
| 创建/最近推送 | 2023-07-19（原 llm-app 时期）/ 2026-11-02 | gh api |
| 版本 | 0.1.8（唯一 release，2026-11-02 发布） | `pyproject.toml`、gh api releases |
| forks / contributors | 1,547 / 24 | gh api |

**重要背景**：本仓库 2023-07 以 Pathway 官方 LLM 应用模板集（llm-app）的身份创建并积累了绝大部分星数；2026 年团队将模板迁回 `pathwaycom/pathway` 的 `examples/llm-app`（`README.md` 顶部即有迁移说明），并把本仓库整体转型为新产品 serviette（v0.1.8 release 名"llm-app is now serviette"是直接证据）。因此 58.7k 星反映的是历史积累，当前代码库本体是 2026 年重写的新项目。

## 2. 项目是什么，解决什么问题

传统 RAG 落地有两个长期痛点：一是搭建链路长——文档解析、切块、嵌入、写入向量库、检索、生成，每一步都要写胶水代码；二是"过时"——文档更新后必须手动重新跑一遍索引流水线，答案与文档永远差一拍。serviette 的回答是把这两件事同时消掉（出处：`README.md`、`docs/README.md`）：

- **无代码**：所有配置收敛到一个 YAML 文件（文档源、向量库、embedder、检索策略），另有 `serviette wizard` 交互式生成配置。默认路径（本地嵌入 + 内嵌 DuckDB）零外部依赖、零凭据，`serviette demo` 一条命令即可在本地起一个可聊天的完整栈。
- **实时同步**：索引器构建在 Pathway 流式引擎上，以 `only_metadata` 模式监视文件系统/GDrive/S3/SharePoint 等源，文件变化触发增量管道，写入端使用 Pathway 原生连接器的 snapshot/upsert 模式，使"删除文档"变成向量库里的真实 delete 而非残留脏数据（出处：`serviette/indexer/graph.py` 模块 docstring）。

架构上是刻意的两半解耦（出处：`README.md` Architecture 一节）：**indexer（写路径）与 server（读路径）是两个独立进程，唯一契约是向量库本身**。server 无状态可横向扩展；indexer 通过 `pathway spawn` 按配置分片到多 worker 进程。任何一侧宕机不影响另一侧，重启的 indexer 从持久化快照恢复且无需重新嵌入。这带来一个有意思的产品立场——"向量库保持是你的"：数据以公开 schema 存在用户自己选的库里，BI、其他应用可以直接读同一集合，不被平台锁定。

**多模态开箱即用**是另一卖点（出处：`README.md` Multimodality 表格、`serviette/indexer/graph.py` 的 `ParserRegistry`）：文本/Markdown 原样、PDF 内置 pypdf（`serviette[docling]` 可升级为版面感知解析）、Office 用 unstructured、扫描件用 PaddleOCR（`serviette[ocr]`）、音频走 Whisper、视频走 TwelveLabs。路由策略是"keyless 优先"——每种格式默认路由到不需要 API key 的最佳解析器，需要 key 的模态在 key 缺席时跳过并告警而非崩溃。

**检索质量**提供五个可选增强策略（出处：`README.md` Retrieval quality、`serviette/server/`）：混合检索（进程内 BM25 + 向量 RRF 融合）、重排（本地 cross-encoder 或 LLM 打分）、自适应 RAG（LLM 报告上下文不足时自动扩 k 重试）、查询分解（多跳问题拆子查询后 round-robin 合并）、MMR 多样性，以及"文档级问题"模式（"库里有几份文档？""两个版本的合同差在哪？"这类无法靠 chunk 检索回答的问题）。

## 3. 技术栈与依赖

- **核心引擎**：Pathway ≥ 0.32.1（`pathway[xpack-llm]`）——Rust 实现的流式数据框架，serviette 自己不实现任何解析/切分/嵌入，全部复用 `pathway.xpacks.llm` 的 parsers/splitters/embedders（出处：`serviette/indexer/graph.py` docstring、`pyproject.toml`）。
- **服务层**：FastAPI + uvicorn + httpx + pydantic v2 + PyYAML（`pyproject.toml` dependencies）。
- **默认向量库**：DuckDB ≥ 1.0（内嵌、纯 wheel、库内 `list_cosine_similarity` 向量检索，`serviette/server/accessors/duckdb.py`）；另 7 个后端按 extra 安装：pgvector/asyncpg、qdrant-client、pymilvus、chromadb-client、weaviate-client、pinecone、pymongo。
- **文档解析**：pypdf、unstructured[docx,pptx,xlsx]（内置）；docling、paddleocr（可选）。
- **本地嵌入**：sentence-transformers（`local` extra，PyTorch 栈，README 提示用 CPU 源省 ~4.2GB）。
- **工程细节**：`pyproject.toml` 里有一组罕见的"解析器护栏"注释（`numba>=0.59` 防 py3.12 回溯到无 wheel 的旧版、`protobuf<6` 与 `google-api-core` 双上限化解 pathway 传递依赖冲突），显示作者在 pip 依赖地狱上花了真实功夫。

## 4. 目录结构与代码组织

顶层结构（tarball 实测，167 个文件）：

| 目录/文件 | 职责 |
|---|---|
| `serviette/` | Python 包本体，约 9.9k 行 |
| `serviette/cli.py` | 唯一入口，分发 indexer/server/up/demo/frontend/wizard 子命令 |
| `serviette/config/schema.py` | 1003 行 Pydantic v2 配置 schema，支持 `${ENV_VAR}` 插值 |
| `serviette/indexer/` | 写路径：`graph.py`（1028 行，Pathway 管道）、`main.py`（多 worker spawn）、`prepare.py`、`sources.py`、`fingerprint.py` |
| `serviette/server/` | 读路径：`main.py`（765 行 FastAPI）、`documents.py`（972 行文档级问答）、`hybrid.py`/`bm25.py`/`ranking.py`/`reranker.py`/`embedder.py`/`llm.py`/`decompose.py` |
| `serviette/server/accessors/` | 8 个向量库后端，统一 `AsyncVectorAccessor` 抽象（`abstract.py`）+ DuckDB 后端共用的 `KeywordHybridMixin` |
| `serviette/up.py` | 662 行的"刻意笨拙"监督器：拉起 indexer+server、健康等待、信号转发 |
| `serviette/wizard.py` | 829 行交互式配置向导 |
| `serviette/frontend/` | 可选独立前端代理层（UI 与 API 分机部署用） |
| `serviette/demo/corpus/` | 内置演示语料（虚构咖啡公司 Lumina Coffee，5 个 md 文件） |
| `tests/` | 33 个测试文件、288 个测试函数、约 7.5k 行 |
| `benchmarks/` | 两套自包含基准：`realtime-data-indexing/`（索引资源）与 `frames/`（FRAMES 检索评测，含 `REPORT.md` 技术报告） |
| `docs/README.md` | 635 行完整手册（安装、配置参考、持久化、扩缩容、安全） |
| `.github/workflows/ci.yml` | 三段式 CI |

组织方式评述：**单包单体、按读写路径划分模块**，没有 monorepo/多包开销。代码密度高且注释质量突出——几乎每个模块的 docstring 都在解释"为什么这样设计、不这样会出什么事故"（如 `indexer/main.py` 解释信号不转发会留下孤儿 worker 占着 DuckDB 锁；`fingerprint.py` 解释配置漂移会导致删除时 chunk id 失配、向量库残留孤儿行）。

## 5. 核心架构与关键实现

### 5.1 实时索引管道（`serviette/indexer/graph.py`）

管道拓扑在模块 docstring 里一句话讲清：`pw.io.fs.read(only_metadata)` → 解析 UDF（按路径取字节→xpack 解析，带持久化缓存）→ xpack 切分器 → 展平为每 chunk 一行 → 嵌入 → 向量库 sink。**删除语义**是最精妙的一处：

```python
# graph.py docstring（节选）
# The parse UDF is registered `deterministic=False` (Pathway's default), so on
# a file removal Pathway re-emits the memoised parsed text *negated* and never
# re-reads the now-deleted bytes; the retraction flows through split/embed and
# the sink removes exactly the matching vectors.
```

即删除时**不重读已消失的文件**，靠 Pathway 对 UDF 输出的记忆化回放生成"负记录"，让流经 split/embed 的撤销流在 sink 端精确删除对应向量。而切块/嵌入 UDF 则注册为 `deterministic=True`（删除时重算而非回放），这引出下面的一致性护栏。

### 5.2 配置指纹护栏（`serviette/indexer/fingerprint.py`）

`deterministic=True` 的切块 UDF 在回撤时**重新执行**，只有输出字节级一致才安全。若两次重启之间用户改了切块参数或升级了 tiktoken，旧文档在删除时会按新规则重新切块，chunk id 对不上，向量库就留下孤儿行。fingerprint 模块把"影响输出的全部要素"（切分器配置、embedder 身份与所有形参、pathway/tiktoken/langchain 版本、chunk-id 方案）以**明文对象**（非仅哈希）存入 `<workdir>/serviette-fingerprint.json`，启动时对比，任何差异以人类可读 diff 呈现并拒绝启动，除非用户交互确认或设 `SERVIETTE_ACCEPT_FINGERPRINT_CHANGES=1`。

### 5.3 进程内混合检索（`serviette/server/hybrid.py` + `ranking.py`）

7/8 后端通过混入 `KeywordHybridMixin` 获得同构的 BM25 混合检索：后端只需实现 `_hybrid_count`/`_hybrid_version`/`_hybrid_fetch_all` 三个钩子，mixin 负责缓存失效（廉价变更信号 = 行数 + 最新 `seen_at`，兜底每 30s 后台刷新）、规模上限（超 500 万 chunk 放弃关键词腿并告警降级为纯向量）、以及 RRF 融合：

```python
# serviette/server/ranking.py
RRF_K = 60
def rrf_merge(result_lists, limit):
    for hits in result_lists:
        for rank, hit in enumerate(hits):
            key = hit["text"]
            scores[key] = scores.get(key, 0.0) + 1.0 / (RRF_K + rank + 1)
```

查询分解的多路结果则用 `interleave_merge`（轮询合并）而非 RRF——保证每个子查询的第一名都有席位，适配"每跳各需一块拼图"的多跳问题。Pinecone 无法枚举向量，`hybrid: true` 在启动时快速失败并说明原因（`README.md` 后端能力表）。

### 5.4 后端抽象与多 worker（`server/accessors/abstract.py`、`indexer/main.py`）

所有后端实现 `AsyncVectorAccessor`（异步检索避免阻塞 FastAPI 事件循环），`supports_embeddings`/`supports_catalog` 两个类属性声明能力（MMR 需要取回向量、文档级问答需要枚举文档）。DuckDB 后端（`accessors/duckdb.py`）值得单看：检索完全在 DuckDB 内用 `list_cosine_similarity` 列式原生扫描；写入端配合 `detach_between_batches` 在 minibatch 间释放单写者文件锁，读取端短连接只读、在锁窗口内重试，超时抛 `IndexBusyError`→HTTP 503 友好提示，而非栈跟踪。多 worker 则由 `indexer/main.py` 通过官方 `pathway spawn --processes N` 机制实现，父进程先做一次后端准备再重执行自身，并实现了完整的进程组信号转发（15s 宽限后升级 SIGKILL，退出码按 shell 惯例 128+N）。

## 6. 构建与运行

来自 `README.md`/`docs/README.md` 的实际命令：

```bash
pip install serviette                       # OpenAI 嵌入路径
pip install "serviette[local]" --extra-index-url https://download.pytorch.org/whl/cpu
                                            # 本地免费嵌入（CPU 版 PyTorch ~0.8GB）
export PATHWAY_LICENSE_KEY=...              # 免费授权：pathway.com/framework/get-license
export OPENAI_API_KEY=sk-...                # 可选：无 key 时答案引用原文片段

serviette demo --embedder openai            # 四命令零到聊天 → http://localhost:8989
serviette wizard                            # 交互式生成 config.yaml（7 问出默认路径）
serviette up --config config.yaml           # indexer+server 同起（dev/demo）
serviette indexer --config config.yaml      # 生产：两进程分开跑
serviette server --config config.yaml

curl -X POST http://localhost:8989/api/v1/retrieve \
  -H 'Content-Type: application/json' -d '{"query": "...", "k": 5}'
```

REST API（`serviette/server/main.py`）：`/api/v1/health|retrieve|rag|stats|documents|config`，顶层旧路径保留为 deprecated 别名；`/` 直接托管聊天 UI（同端口同源无 CORS）。

## 7. 测试与工程质量

- **三层测试**（`pyproject.toml` markers、`.github/workflows/ci.yml`）：`not slow` 快速单测（不起引擎，py3.10/3.12/3.14 三版本矩阵）；`slow` 端到端（真实 Pathway 运行时，受 `PATHWAY_LICENSE_KEY` secret 存在与否优雅跳过）；`integration` 矩阵——**8 个后端每个都有对真实实例的集成测试**（`tests/test_integration_*.py` + 共享驱动 `tests/integration_common.py`）：真索引两文档→检索（余弦 ~1.0）→删文件→验证向量消失的快照语义。容器用后即弃（`tests/dockerutil.py`，纯 subprocess 调 Docker CLI，端口动态分配，缺 Docker 自动跳过）。
- **规模**：33 文件、288 个测试函数、约 7.5k 行测试代码，与 ~9.9k 行产品代码接近 1:0.76。
- **质量观察**：ruff（line-length 100）+ mypy 进 dev 依赖与 CI；配置 schema `extra="forbid"`（拼错的配置键直接报错）；错误路径打磨极细（旧 pathway 版本给升级提示而非 AttributeError、拼错的源目录在启动期拦截而非变成空索引）。基准工程（`benchmarks/`）把原始结果 JSON/CSV、运行环境、方法论与**测量局限**全部提交入库（`benchmarks/frames/REPORT.md` 明确声明不做什么），学术级的诚实度罕见。

## 8. 社区与维护状态

- **活跃度**：最近推送 2026-11-02（分析前一天）；main 分支约 260 个提交，最新 100 个提交的时间跨度始于 2024-06（gh api commits），说明 2026 年的 serviette 重写期提交非常密集。
- **发布节奏**：v0.1.8 是当前唯一 GitHub release（2026-11-02），PyPI 上已有 `serviette` 包；项目显然处于发布早期（0.1.x）。
- **贡献者**：24 人（gh api contributors）；近期提交集中于 Sergey Kulik/Sergey（Pathway 核心团队，`LICENSE` 版权人），近期贡献高度单人化——转型期由团队内部主导。
- **社区机制**：CONTRIBUTING.md（改编自 Pathway 的贡献指南）明确表示"开 issue 是最有价值的贡献"——因为团队用前沿模型写代码，最稀缺的是问题定义与评审带宽；社区主阵地在 Discord。
- **CI 健全**：三 job（lint+fast 矩阵 / slow e2e / integration docker 矩阵），引擎套件依赖免费 license secret，未配置时警告并跳过而非红屏。

## 9. 亮点与风险

**亮点**
1. **删除语义的工程完整性**：`deterministic=False` 回放负记录 + `deterministic=True` 配合配置指纹护栏，把"向量库与文档集合严格一致"这件事做到了少见的严谨程度（`serviette/indexer/graph.py`、`fingerprint.py`）。
2. **解耦架构与零成本默认路径**：indexer/server 只共享向量库，server 无状态可横扩；默认 DuckDB 内嵌 + 本地嵌入让试跑成本为零，数据不出本机（`README.md`、`accessors/duckdb.py`）。
3. **八后端同等公民 + 真实实例集成测试**：每个宣称支持的后端都有对真实 Docker 实例的端到端测试，能力差异（hybrid/catalog 支持）在文档中逐后端标明。
4. **诚实到罕见的基准工程**：50GB/1700 万文件索引基准与 FRAMES 评测（0.50 gold recall vs 论文 BM25 基线 0.15）全部脚本、原始数据、局限说明入库（`benchmarks/`）。
5. **注释即设计文档**：模块 docstring 讲故障模式与设计取舍，可读性极高。

**风险/局限**
1. **对 Pathway 的深度绑定**：引擎、解析器、嵌入器全部来自 `pathway[xpack-llm]`，且运行需申请免费 license key（需 GitHub/LinkedIn 登录换取）——这是商业开源常见的供应商耦合，自托管门槛略高于纯 MIT 直觉。
2. **星数与成熟度错位**：58.7k 星绝大部分继承自 llm-app 模板仓库时期，serviette 本体仅 v0.1.x（发布 2026-11），生产采用需按新项目评估。
3. **功能边界明确**：单轮问答、无对话记忆、无内置用户账号/鉴权（默认只监听 localhost，暴露需自配反向代理，`README.md` Security 节）；Pinecone 无 hybrid/文档目录能力。
4. **进程内 BM25 有规模上限**（默认 500 万 chunk 上限后降级纯向量），超大语料的原生服务端 BM25 尚在计划中（`README.md`、`hybrid.py`）。

## 10. 总结评价

serviette 是一次教科书级的"老仓库转型"：Pathway 团队把积累近 60k 星的模板仓库换成了一件打磨精良的新产品——用约 1 万行 Python 把 Pathway 流式引擎包装成"指向文件夹即得实时 RAG"的服务器，核心价值在实时同步语义的工程严谨性（回撤、指纹护栏、失败隔离）和"数据库始终是你的"这一不锁用户的立场。代码量克制、注释质量上乘、基准方法学诚实，非常适合三类人：想自托管一个不糊弄的实时 RAG 的工程师（DuckDB+本地嵌入路径十分钟可用）、研究流式数据系统如何处理删除/一致性等脏问题的开发者（读 `indexer/graph.py` 与 `fingerprint.py`），以及想学习如何给多后端抽象写集成测试的团队（读 `tests/integration_common.py` 与 `accessors/`）。使用前需接受两点：Pathway license key 的存在，以及按 v0.1.x 新项目的标准而非 6 万星老项目的标准来评估它的成熟度。
