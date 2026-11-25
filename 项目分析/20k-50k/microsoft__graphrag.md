# microsoft/graphrag 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-17

## 1. 项目速览

一句话定位：微软研究院出品的图结构 RAG（GraphRAG）参考实现——用 LLM 从非结构化文本中抽取实体/关系构建知识图谱，经层次化 Leiden 社区检测与社区摘要生成后，在查询端提供 local/global/drift/basic 四种检索模式，尤以 global search 的 map-reduce 社区报告汇总机制成为"全局性问题的语料级问答"范式。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | microsoft/graphrag | gh api |
| 星数 | 39500（快照）；gh api 2026-11-17 复核 36.7k / forks 3.9k | 任务参数 / gh api |
| 语言 | Python（>=3.11,<3.14） | packages/graphrag/pyproject.toml |
| 许可证 | MIT | LICENSE；pyproject `license = {text = "MIT"}` |
| 当前版本 | 3.3.0 | packages/graphrag/pyproject.toml |
| 创建时间 | 2024-03-27 | gh api repos/microsoft/graphrag |
| 最近 push | 2026-11-16（分析前一日） | gh api |
| 维护状态 | 官方宣布"基本进入维护模式"，只修 bug/CVE，不再收新功能 PR | src/README.md 顶部 WARNING |
| 代码规模 | packages+tests 共 551 个 .py、约 4.96 万行 | 本地统计 |

## 2. 项目是什么，解决什么问题

传统向量 RAG 把文档切块、嵌入后做近邻检索，回答"局部事实"类问题很有效，但面对"整个语料讲了什么""数据集里的主要主题是什么"这类全局性、总结性问题时会失效——因为答案散落在所有 chunk 里，任何一次 top-k 检索都只能覆盖局部。GraphRAG（论文 arXiv:2404.16130，README.md 外链）的思路是：先离线把语料加工成一张"带语义的知识图谱 + 层次化社区摘要"，查询时不再面向 chunk，而是面向图的社区结构做检索与汇总（src/README.md "data pipeline and transformation suite"）。

整个系统分两半。**索引端**（`packages/graphrag/graphrag/index/`）是一条工作流管线：文档加载 → 切片 → LLM 实体/关系抽取（或 NLP 快速通道）→ 构图 → 层次化 Leiden 聚类成社区 → 用 LLM 为每个社区生成递归式报告 → 生成文本嵌入。**查询端**（`packages/graphrag/graphrag/query/`）提供四种结构化搜索：local search（实体邻域展开，答具体问题）、global search（map-reduce 汇总社区报告，答全局问题）、drift search（2025 年新增的 agentic 混合检索）、basic search（纯向量 RAG 对照基线）。

README 明确两个定位信息：其一，这是研究项目、"not an officially supported Microsoft offering"（README.md Repository Guidance）；其二，2026 年 11 月的当前快照中项目已进入维护模式——"Since our first release in July 2024 … largely in maintenance mode, won't be accepting new PRs or implementing new features"（README.md WARNING 块），即它已从 2024 年的爆火明星项目转入保育阶段，但代码仍在活跃修复（最近 push 为分析前一日）。

配套还有 prompt-tune 子命令（`graphrag/cli/prompt_tune.py`）针对领域数据自动调优抽取 prompt，以及 RAI_TRANSPARENCY.md 负责任 AI 问答文档，这在同类开源 RAG 项目中少见。

## 3. 技术栈与依赖

- 语言与运行时：Python 3.11–3.13（packages/graphrag/pyproject.toml `requires-python`）。
- 包管理：uv workspace monorepo（根 pyproject.toml `[tool.uv.workspace] members = ["packages/*"]`），构建后端 hatchling。
- LLM 接入：LiteLLM 统一多家模型（packages/graphrag-llm/graphrag_llm/completion/lite_llm_completion.py）；v3 起弃用自研 fnllm（breaking-changes.md v3 节）。
- 图算法：graspologic-native 提供层次化 Leiden 聚类（`graphs/hierarchical_leiden.py` 直接调用 `gn.hierarchical_leiden`，且刻意 pin `<1.3` 以保回归测试金标数据一致，packages/graphrag/pyproject.toml 注释）；networkx 做图结构。
- NLP：nltk（pin 3.9，同理保确定性 noun-phrase 抽取）、spacy、textblob——支撑 "fast" 索引方法的 NLP 建图通道（packages/graphrag/pyproject.toml 及其内嵌注释）。
- 数据处理：pandas、numpy、pyarrow（Parquet 表存储）。
- Azure 生态：azure-identity、azure-storage-blob、azure-search-documents；缓存/存储可换 CosmosDB（CHANGELOG 3.1.0）。
- CLI 框架：typer；配置模型：pydantic v2（config/models/ 下 16 个配置类）。
- 开发工具链：pytest、pytest-asyncio、pytest-xdist、ruff、pyright、poethepoet、semversioner、mkdocs-material。

## 4. 目录结构与代码组织

顶层是 uv workspace 单仓多包（monorepo），核心库 `graphrag` 与 7 个基础设施子包严格分层：

| 目录 | 职责 |
| --- | --- |
| packages/graphrag | 主包：index/ 索引管线、query/ 查询引擎、cli/、api/、config/、prompts/、prompt_tune/、graphs/（Leiden 等）、data_model/、tokenizer/ |
| packages/graphrag-llm | LLM Provider 抽象：completion/embedding 工厂、LiteLLM 适配、重试/限流/缓存/指标/分词 |
| packages/graphrag-cache | 缓存抽象（本地文件、SQLite、Blob；3.2.0 新增并发安全 SQLite provider） |
| packages/graphrag-storage | 存储抽象（表 Table 抽象，Parquet/Blob/Cosmos 等） |
| packages/graphrag-chunking | 文本切块策略 |
| packages/graphrag-input | 输入文档加载（csv/text/json 等 loader） |
| packages/graphrag-vectors | 向量库抽象（默认 local、Azure AI Search） |
| packages/graphrag-common | 各包共享的 Factory 等基元 |
| tests/ | unit / verbs / integration / smoke / notebook 五层测试 |
| docs/ + mkdocs.yaml | mkdocs-material 文档站，发布至 microsoft.github.io/graphrag（.github/workflows/gh-pages.yml） |
| unified-search-app/ | 四种搜索模式对比的演示应用（Dockerfile + streamlit 类 app，README 自述仅 demo 用途） |
| .semversioner/、.vsts-ci.yml、.github/workflows/ | 语义化版本自动管理与 CI |

组织方式评述：v1 时代是单包 + DataShaper 管线，v2 重命名输出表摆脱 DataShaper，v3 把 LLM/缓存/存储/向量等横切能力抽成独立可复用的子库并整体瘦身（breaking-changes.md v2/v3 节）。分层清晰：主包只做业务编排，所有 IO/模型调用都经子包抽象，这正是它能支撑 Azure 生产场景的结构基础。

## 5. 核心架构与关键实现

### 5.1 索引管线：workflow 编排 + Leiden 社区检测

索引不是硬编码 DAG，而是 23 个可组合 workflow（`packages/graphrag/graphrag/index/workflows/`：load_input_documents → create_base_text_units → extract_graph/extract_graph_nlp → create_communities → create_community_reports → generate_text_embeddings 等），由 `workflows/factory.py` 按 IndexingMethod（standard/fast/standard-update/fast-update，`config/enums.py`）装配，update_* 系列支持增量索引。社区检测核心非常薄——直接委托 graspologic-native 的层次化 Leiden（`packages/graphrag/graphrag/graphs/hierarchical_leiden.py`）：

```python
def hierarchical_leiden(
    edges: list[tuple[str, str, float]],
    max_cluster_size: int = 10,
    random_seed: int | None = 0xDEADBEEF,
) -> list[gn.HierarchicalCluster]:
    return gn.hierarchical_leiden(
        edges=edges, max_cluster_size=max_cluster_size, seed=random_seed,
        starting_communities=None, resolution=1.0, randomness=0.001,
        use_modularity=True, iterations=1,
    )
```

`create_communities.py` 把聚类结果按层级流式写入 communities 表；随后 `summarize_communities/` 操作对每个社区递归生成 LLM 报告（prompt 模板见 `prompts/index/community_report.py`）。实体抽取 prompt 采用 `("entity"<|><entity_name><|>…)` 的分隔符协议并用 json-repair 容错解析（`prompts/index/extract_graph.py`、主包依赖 `json-repair`）。

### 5.2 Global Search：map-reduce 社区报告汇总（核心创新）

`query/structured_search/global_search/search.py` 是论文思想的直接落地。`search()` 文档字符串明确两步：先把社区报告按 token 预算分批，对每批并发调用 LLM 产生中间答案（map），再把所有中间答案合并成最终答案（reduce）：

```python
map_responses = await asyncio.gather(*[
    self._map_response_single_batch(
        context_data=data, query=query,
        max_length=self.map_max_length, **self.map_llm_params)
    for data in context_result.context_chunks
])
# Step 2: Combine the intermediate answers ... generate the final answer
reduce_response = await self._reduce_response(
    map_responses=map_responses, query=query, **self.reduce_llm_params)
```

并发由 `asyncio.Semaphore(concurrent_coroutines=32)`（search.py:104）节流，map 阶段强制 JSON 输出便于打分（`json_mode` 把 `response_format_json_object` 注入 map 参数，search.py:96-100），支持流式（stream_search 先流式跑完 map 再流式 reduce）。MAP/REDUCE 系统 prompt 与"无可归答"短路（NO_DATA_ANSWER）在 `prompts/query/global_search_map_system_prompt.py` / `global_search_reduce_system_prompt.py`。

### 5.3 四种搜索模式与上下文构建器

查询端统一基类 `BaseSearch[ContextBuilder]`（`query/structured_search/base/`），四种实现共享 context_builder 体系（`query/context_builder/builders.py`）：local search 以向量匹配实体为种子展开邻域关系/协变量（`local_search/search.py`）；drift search（`drift_search/`：primer.py 生成初版答案 → action.py 迭代细化 → state.py 维护查询状态，内部复用 LocalSearch）是 2025 年论文新增的 agentic 模式；basic search 是纯向量 RAG 基线。API 层 `api/query.py` 为每种模式提供同步与 streaming 双入口（global_search/local_search/drift_search/basic_search 共 8 个函数），CLI 层 `cli/main.py` 的 typer 应用暴露 init/index/update/prompt-tune/query 五个子命令。

### 5.4 LLM/嵌入 Provider 抽象

`packages/graphrag-llm/` 用注册式工厂解耦模型接入：`completion/completion_factory.py` 继承 `graphrag_common.factory.Factory`，通过 `register_completion` 允许用户注册自定义 provider，内置 LiteLLM 实现（多厂商统一）与 mock 实现；同一抽象覆盖 embedding、tokenizer、cache（key 生成）、rate_limit、retry、metrics。这让"模型类型"配置从 v3 起简化为 `chat`/`embedding` + LiteLLM 的 model_provider（breaking-changes.md v3 节）。

## 6. 构建与运行

来自根 pyproject.toml `[tool.poe]` 脚本与 README/docs（未实际运行，静态核对）：

```bash
# 安装（uv 管理 workspace）
uv sync
# CLI 使用（PyPI 包入口 graphrag = "graphrag.cli.main:app"）
graphrag init --root ./ragtest        # 生成 settings.yaml 与 prompt 模板
graphrag index --root ./ragtest       # 建索引（standard 或 --method fast）
graphrag query --root ./ragtest --method global --query "主要主题是什么"
# 测试
uv run poe test_unit          # pytest ./tests/unit
uv run poe test_verbs         # pytest ./tests/verbs
uv run poe test_integration   # pytest ./tests/integration
# 文档站
mkdocs serve / mkdocs build   # mkdocs.yaml
```

版本发布由 semversioner 驱动（`.semversioner/`、`.github/workflows/semver.yml`、RELEASE.md），小版本升级需重跑 `graphrag init --force`（README.md Versioning 节）。

## 7. 测试与工程质量

- 测试规模：tests/ 下 128 个 .py 文件，分五层——unit（config/indexing/query/storage 等 15 个子目录）、verbs（管线算子级）、integration（真实 LLM/Azure 集成）、smoke、notebook（.github/workflows/python-notebook-tests.yml 单独跑 notebook）。
- 单测矩阵：python-unit-tests.yml 在多 OS × Python 3.11-3.13 矩阵上跑 unit 与 verbs，用 uv + poe。
- CI 双轨制：Azure Pipelines（.vsts-ci.yml：CredScan、Component Governance 等微软内部合规阶段）+ GitHub Actions（unit/integration/smoke/notebook/publish/semver/spellcheck/dependabot/gh-pages 共 11 个 workflow）。
- 工程质量观察：ruff + pyright 静态检查；全库 py.typed；依赖 pin 带有清晰的理由注释（graspologic-native、nltk 的 pin 注释明确指向回归测试金标数据，packages/graphrag/pyproject.toml）；CHANGELOG 由 semversioner 自动聚合；breaking-changes.md 对 CLI/API/配置/数据模型四个面的兼容性承诺写得很专业。整体是微软水准的工程化范式。

## 8. 社区与维护状态

- 活跃度：gh api 显示最近 push 2026-11-16（分析前一日）、open issues 仅 70，仓库未 archived——与 README"维护模式"声明一致：代码仍在动，但只做修复与依赖维护。
- Release 节奏：近 10 个 release 为 v3.3.0(2026-10-08)、v3.2.0(2026-09-24)、v3.1.2(2026-08-21)、v3.1.1(2026-07-18)、v3.1.0(2026-05-28)、3.0.9(2026-04-13)、3.0.8(2026-03-27)、3.0.7(2026-03-24)、3.0.6(2026-03-06)、3.0.5(2026-02-27)（gh api releases），约每月 1 版，进入 v3 后节奏稳定。
- 贡献者结构：contributors 分页 last=51，即至少 51 名贡献者（gh api）；根 pyproject authors 列出 16 名微软员工邮箱成员（Alonso Guevara、Jonathan Larson 等），核心团队以微软研究院/云 AI 人员为主。
- 生态位：2024-07 首发即爆火（README 自述 first release July 2024），论文 arXiv:2404.16130 被广泛引用，"GraphRAG"一词已成为图 RAG 类系统的代名词，社区衍生实现众多（未逐一核实）。

## 9. 亮点与风险

亮点：
1. Global search 的 map-reduce 社区汇总机制是概念与工程的双创新，直接解决了向量 RAG 答不了全局问题的痛点（query/structured_search/global_search/search.py）。
2. 横切能力全部库化：graphrag-llm/cache/storage/chunking/vectors 子包各自可独立复用，工厂+注册机制扩展性好（packages/graphrag-llm）。
3. 工程化教科书级：五层测试、双 CI、semversioner 自动版本、依赖 pin 带理由注释、breaking-changes.md 明确兼容面承诺。
4. standard/fast 双索引方法与 update 增量索引兼顾成本与时效（config/enums.py IndexingMethod）。
5. prompt_tune 子命令 + RAI 透明度文档，对落地用户与合规场景友好。

风险/局限：
1. 官方已宣布维护模式、不接受新功能 PR——长期演进预期应放低（README.md WARNING）。
2. 索引成本高：对全语料做 LLM 实体抽取 + 社区报告，token 开销远超普通向量 RAG，README 也专门警告 "indexing can be an expensive operation"。
3. 明确自述"研究参考实现、非微软官方支持产品"，生产使用需自担（README.md）。
4. 依赖面广且多处人为 pin（graspologic-native、nltk），升级需连带刷新回归金标数据；Azure 组件默认耦合较深。

## 10. 总结评价

GraphRAG 是 2024 年 RAG 领域最有影响力的开源项目之一：它把"知识图谱 + 层次社区摘要"变成可复现的工程管线，用 global search 的 map-reduce 机制开创了语料级问答范式，其 v1→v2→v3 的重构轨迹（去 DataShaper、去 fnllm、横切能力子包化）也示范了研究代码如何走向工程化。如今它已功成身退进入维护模式，v3.3.0 是稳定的收官形态。适合两类人：要在私有数据上做全局性问答/主题分析的工程团队（可直接用 CLI/API，注意索引成本），以及想学习"图结构 RAG"与 LLM 管线工程化设计的开发者（index/workflows 编排、global_search 实现、graphrag-llm 工厂抽象都值得精读）。若寻求仍在快速演进的前沿，应关注其社区衍生项目而非本仓库本身。
