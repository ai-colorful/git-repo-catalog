# HKUDS/LightRAG 项目分析

> 星数 24500（快照 2026-10-24，gh api 2026-11-14 实测 40.7k）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：港大数据智能实验室（HKUDS）出品的轻量级图增强 RAG 框架——用"知识图谱 + 向量"双层检索替代微软 GraphRAG 的社区报告方案，主打更低成本、更快响应、可增量更新。

| 项目 | 内容 |
|---|---|
| 名称 | HKUDS/LightRAG（PyPI: lightrag-hku） |
| 星数 | 快照 24.5k；gh api 实测 40,687（2026-11-14），仍在高速增长 |
| 语言 | Python 3.11+（后端）+ TypeScript/React（WebUI） |
| 许可证 | MIT（LICENSE） |
| 当前版本 | v1.5.8（lightrag/_version.py） |
| 最近活跃 | push 2026-11-13（gh api，前一天仍在提交，极活跃） |
| 创建时间 | 2024-10-02（gh api） |
| 贡献者 | 335 人（gh api contributors 分页推算） |
| 论文 | arXiv:2410.05779（README Citation 节） |

## 2. 项目是什么，解决什么问题

LightRAG 要解决传统向量 RAG 的"碎片化上下文"问题与微软 GraphRAG 的"成本高、更新难"问题（README "About LightRAG"）。传统 RAG 只按文本块做向量相似检索，无法回答跨文档、跨实体的关联问题；GraphRAG 用社区报告（community report）做全局理解，但索引期需要海量 LLM 调用，且增量更新困难。

LightRAG 的方案是双层架构（dual-layer）：同一份数据同时维护知识图谱（实体、关系及其摘要）和向量嵌入（文本块、实体、关系三套向量库），检索时在"具体实体层"和"全局主题层"并行召回，不依赖社区报告和多跳推理，大幅降低索引与查询期的 LLM 调用量（README "Features & Advantages"）。论文在农业、计算机、法律、混合四个领域对比 NaiveRAG/RQ-RAG/HyDE/GraphRAG，LightRAG 在全面性与多样性维度全面领先，与 GraphRAG 对比 Overall 约 52-85% 胜率（README "Replicating Findings in the Paper" 表格 + docs/Reproduce.md）。

工程上它已经远超"论文配套代码"：v1.5 起合并了姊妹项目 RAG-Anything 的多模态能力（MinerU/Docling 文档解析、VLM 图像分析），支持四种分块策略、四类 LLM 角色配置、25 个存储后端实现，并提供完整 API 服务 + WebUI + Ollama 兼容层（README News 与 lightrag/ 目录结构）。删除文档时可利用索引期 LLM 缓存快速重建受影响的实体/关系（README "Incremental Updates" + operate.py `rebuild_knowledge_from_chunks`）。

同门生态：RAG-Anything（多模态 RAG）、VideoRAG（长视频 RAG）、MiniRAG（小模型 RAG）均出自 HKUDS（README "Related Projects"）。

## 3. 技术栈与依赖

- 核心语言：Python 3.11+（pyproject.toml `requires-python = ">=3.11"`），asyncio 异步架构贯穿全部存储与 LLM 调用
- 关键依赖（pyproject.toml）：
  - `networkx`：默认图存储引擎；`nano-vectordb`：默认向量存储（均为同作者轻量实现）
  - `tiktoken`：token 计数（要求 >=0.7.0 保证多线程安全）；`json_repair`：修复 LLM 输出 JSON
  - `pydantic`/`fastapi`/`uvicorn`/`gunicorn`：API 服务层（api extra）
  - 文档处理：`pypdf`/`python-docx`/`python-pptx`/`openpyxl`/`cairosvg`；分块：`langchain-text-splitters`/`langchain-experimental`；`spacy 3.8` 用于 docx smart_heading 标题识别
  - 存储客户端（offline-storage extra）：`asyncpg`/`pgvector`/`neo4j`/`pymilvus`/`pymongo`/`qdrant-client`/`opensearch-py`/`redis`/`faiss-cpu`
  - LLM 客户端（offline-llm extra）：`openai`/`anthropic`/`ollama`/`google-genai`/`aioboto3`(Bedrock)/`voyageai`/`zai-sdk` 等
- WebUI（lightrag_webui/package.json）：React + TypeScript + Vite + Radix UI + Tailwind + `@react-sigma/*`（知识图可视化）+ i18n，Bun 构建
- 评估与可观测：`ragas`（evaluation extra）、`langfuse`（observability extra）（pyproject.toml）
- 包管理：uv（uv.lock 锁定，README Installation）

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `lightrag/` | 核心 Python 包，179 个 .py 约 17 万行 |
| `lightrag/lightrag.py` | 主类 `LightRAG`（8506 行），存储装配与 insert/query 门面 |
| `lightrag/operate.py` | 算法核心（7236 行）：实体抽取、图合并、双层检索、上下文构建 |
| `lightrag/pipeline.py` | 文档处理管线（`_PipelineMixin`，入队/并发/取消/进度） |
| `lightrag/kg/` | 25 个存储后端实现 + 工厂（factory.py）与注册表（__init__.py） |
| `lightrag/llm/` | 14+ 个 LLM/VLM 提供商绑定（openai/anthropic/gemini/ollama/bedrock 等） |
| `lightrag/api/` | FastAPI 服务：lightrag_server.py（3596 行）、routers、auth、限流、中间件 |
| `lightrag/parser/` + `chunker/` | v1.5 文档解析引擎（MinerU/Docling/native/legacy）与四种分块策略 |
| `lightrag/tools/` | 运维 CLI：rebuild-vdb、clean-llmqc、hash-password、download-cache 等 |
| `lightrag/evaluation/` | RAGAS 评估集成 |
| `lightrag_webui/` | React WebUI 源码（Bun/Vite），构建产物进 PyPI 包（MANIFEST.in、pyproject package-data） |
| `tests/` | 573 个 test_*.py，11MB，按模块分子目录 |
| `examples/` | 20 个 SDK 示例（各 LLM/存储组合） |
| `docs/` | 37 篇参考文档，多数中英双语 |
| `prompts/`、`reproduce/`、`scripts/`、`k8s-deploy/` | 提示词、论文复现、安装向导脚本、K8s 清单 |

组织方式评述：单体 Python 包 + 前端子目录的双栈仓库。核心类正在从巨型 lightrag.py 向 Mixin 拆分（pipeline.py、llm_roles.py、storage_migrations.py），kg/ 与 llm/ 各自内聚为"实现类 + 工厂"模式，是典型的可插拔后端设计。

## 5. 核心架构与关键实现

### 5.1 双层检索（查询核心）

入口 `kg_query`（lightrag/operate.py:4721）先把用户查询交给 KEYWORD 角色 LLM 抽取两组关键词：低层（ll_keywords，具体实体）与高层（hl_keywords，主题概念），随后 `_resolve_kg_search_queries`（operate.py:5365）按模式分配检索目标：

```python
# operate.py:5397 核心分配逻辑
entity_query = ll_keywords if mode in ("local", "hybrid", "mix") else ""
relation_query = hl_keywords if mode in ("global", "hybrid", "mix") else ""
```

即：**local 查实体向量库 + 图邻域**（ll_keywords 驱动），**global 查关系向量库**（hl_keywords 驱动），hybrid/mix 两者兼取；naive 完全绕过图、只查 chunks 向量库；mix 再叠加 naive 的 chunks 召回（QueryParam.mode 定义于 lightrag/base.py:93，共 6 种含 `bypass` 直通 LLM）。弱模型关键词分组失误时有兜底：local 组为空则用 hl_keywords 代查实体库（operate.py:5379-5395 注释详述）。三组 embedding（query/ll/hl）合并为单批请求减少 API 往返（operate.py:_perform_kg_search 内注释），最终经 `_apply_token_truncation` 按 max_entity_tokens/max_relation_tokens/max_total_tokens 统一预算裁剪（operate.py:5614）。

### 5.2 知识图构建管线（索引核心）

`extract_entities`（operate.py:3958）对每个 chunk 并发调用 EXTRACT 角色 LLM 抽取实体/关系（支持纯文本分隔符与 JSON 两种输出格式，`entity_extraction_json_system_prompt` 见 prompt.py:175）；`merge_nodes_and_edges`（operate.py:3520）将同名的实体/关系描述合并摘要（`_merge_nodes_then_upsert`/`_merge_edges_then_upsert`，operate.py:2435/2788）后写入图存储，同时把实体、关系分别嵌入 entities_vdb/relationships_vdb 两个向量库（kg_query 的函数签名即为这三套存储）。文档删除走 `rebuild_knowledge_from_chunks`（operate.py:1108），利用索引期缓存的 LLM 抽取结果重建受影响节点，避免全量重跑（README "Incremental Updates & Selective Deletion"）。抽取结果可按角色/模式落入 LLM 缓存 KV，工具 `lightrag-clean-llmqc` 可单独清理查询缓存而保留昂贵抽取缓存（pyproject.toml scripts + lightrag/tools/）。

### 5.3 存储后端矩阵（25 个实现）

四类存储各注册于 lightrag/kg/__init__.py 的 `STORAGE_IMPLEMENTATIONS`：

| 类型 | 实现 |
|---|---|
| KV（5） | Json、Redis、PG、Mongo、OpenSearch |
| Graph（7） | NetworkX、Neo4j、PGGraph(Apache AGE)、PGTable、Mongo、Memgraph、OpenSearch |
| Vector（8） | Noop、NanoVectorDB、Milvus、PGVector、Faiss、Qdrant、Mongo、OpenSearch（Chroma 已注释） |
| DocStatus（5） | Json、Redis、PG、Mongo、OpenSearch |

默认四件套（JsonKV/NanoVectorDB/NetworkX/JsonDocStatus）全部为内存态 + WORKING_DIR 本地文件持久化，README 明确警告仅限测试、生产需换 PostgreSQL（官方推荐，单后端可同时服务四类）或 Mongo/OpenSearch。解析由 lightrag/kg/factory.py：四个默认实现直连导入，其余经 `STORAGES` 注册表惰性 import。另有 shared_storage.py 处理多进程共享（gunicorn 多 worker 场景）。

### 5.4 API 服务与 WebUI

`lightrag_server.py`（3596 行）组装 FastAPI 应用：query_routes（查询 + 上下文返回）、document_routes（上传/删除/扫描/状态）、graph_routes（图可视化数据）、ui_customization_routes；`routers/ollama_api.py` 实现 Ollama 协议（/api/chat、/api/generate、/api/tags、/api/version），使任意 Ollama 客户端可直接把 LightRAG 当知识库模型调用。安全层完整：JWT + bcrypt 账号认证（auth.py）、登录限速（login_rate_limit.py）、请求体大小限制（body_limit_middleware.py）、准入控制（admission.py），PyPI 依赖还显式钉住多个 CVE 安全版本下限（pyproject.toml:108-112 注释）。WebUI 双入口：/webui 管理台（文档管理 + Sigma.js 知识图可视化 + 查询调试）与 /workspace 纯查询页（面向移动端最终用户，README "WebUI Entries"）。

## 6. 构建与运行

```bash
# 安装（README Installation）
uv tool install "lightrag-hku[api]"     # PyPI 包自带预构建 WebUI
uv sync --extra api                      # 源码安装（仅服务）
uv sync --extra test --extra offline    # 贡献者全量安装

# 构建 WebUI（源码方式需要）
cd lightrag_webui && bun install --frozen-lockfile && bun run build

# 配置与启动
cp env.example .env                      # 或 make env-base 交互式向导
lightrag-server                          # 默认 0.0.0.0:9621，/webui 与 /docs
lightrag-gunicorn                        # 多 worker 生产模式（pyproject scripts）

# 测试
pytest                                   # offline 门禁测试（tests/，不含 tests/setup）
pytest tests/setup                       # 安装向导 shell 流程测试（CI 单独工作流）
pytest -m integration --run-integration  # 需真实后端的集成测试（pyproject markers）

# 运维工具（pyproject.toml [project.scripts]）
lightrag-rebuild-vdb                     # 重建全部向量库（换 embedding 模型后）
lightrag-clean-llmqc                     # 清理查询态 LLM 缓存
lightrag-hash-password                   # 生成 AUTH_ACCOUNTS bcrypt

# Docker
docker compose up                        # 官方镜像含 Postgres/Neo4j 栈（docker-compose.yml）
```

## 7. 测试与工程质量

- 测试规模：573 个 test_*.py（tests/，约 11MB），覆盖 api/chunker/kg/llm/parser/pipeline/extraction/sidecar/evaluation 等全部核心模块，另有多份针对边界的单点测试（如 test_cosine_similarity_nonfinite、test_get_env_value_padded_bool）
- 分层：pytest markers 分为 offline（CI 默认门禁）/ integration / requires_db / requires_api / pg_smoke（pyproject.toml markers）；安装向导 320 个 shell 测试独立工作流（.github/workflows/setup-tests.yml 注释）
- CI：11 个工作流（.github/workflows/）——主测试、setup 向导、RAGAS 评估栈、PG 冒烟、linting（ruff + pre-commit）、pypi-publish、三个 docker 构建、stale 管理、copilot 环境
- 工程细节质量高：依赖注释解释每处版本下限的原因（如 pyproject.toml:27-30 解释 google-api-core floor 是为防 uv 静默回退）；tests/test_requirements_sync.py 保证 spaCy 模型钉在三处副本一致；AGENTS.md/CLAUDE.md 为 AI 协作编码提供仓库级约定

## 8. 社区与维护状态

- 活跃度：gh api 显示最近 push 2026-11-13（前一天），405 个 open issues，5775 forks，335 贡献者；2024-10 创建，两年内冲至 40k+ 星
- 发版节奏：PyPI 包 lightrag-hku，近期 release：v1.5.5（2026-07-31）、v1.5.6（2026-08-06）、v1.5.7rc1/rc2（2026-08-17/19）、v1.5.7（2026-09-02），HEAD 为 v1.5.8（lightrag/_version.py）——平均月度一版，先 rc 后正式
- 功能节奏（README News）：2026 年内持续合并大特性——Smart Heading（2026.07）、RAG-Anything 并入（2026.05）、四种分块策略与四角色 LLM 配置（2026.05）、OpenSearch 统一后端（2026.03）
- 文档：37 篇 docs/ 且多数附中文版，含安全警告（0.0.0.0 绑定与配额风险，README:104-105）、Cosign 镜像签名验证（DockerDeployment.md）
- 商业背景：HKUDS（港大数据智能实验室）系列项目矩阵化运营（LightRAG/RAG-Anything/VideoRAG/MiniRAG/DeepTutor），有 Discord/微信社群

## 9. 亮点与风险

**亮点**
1. 双层检索机制设计聪明：以"实体库/关系库分属 low/high 关键词"实现局部与全局召回，完全绕开 GraphRAG 的社区报告与多跳推理，索引和查询成本数量级下降（operate.py:5365-5405、README 对比表）
2. 存储后端矩阵完整（25 个实现），且提供 PG/Mongo/OpenSearch 三个"单后端全包"选项，生产路径清晰（lightrag/kg/__init__.py）
3. 工程成熟度罕见地高：573 个测试文件、五级测试分层、CVE 驱动的依赖安全下限、JWT/bcrypt/限流完整安全层、离线部署工具链
4. 删除重建机制：利用索引期 LLM 缓存做选择性重建（rebuild_knowledge_from_chunks），解决图 RAG 增量维护痛点
5. 文档与生态：37 篇双语参考文档、20 个示例、Ollama 兼容层 + WebUI 双入口、配套论文可复现（docs/Reproduce.md）

**风险**
1. 核心文件巨型化：lightrag.py 8506 行、operate.py 7236 行，Mixin 拆分尚在进行中，阅读与维护成本高
2. 默认存储为内存态，新手若不换后端极易在数据量增长后 OOM（README 明确警告）
3. 索引期成本仍显著：每 chunk 至少一次 LLM 调用，大文档库首次建图的开销与弱模型下的抽取质量仍是门槛（README "Selecting LLM Models" 详细讨论）
4. 版本迭代快、配置面大（env.example 项极多、6 种查询模式、4 种分块策略、25 个存储），升级与调参的心智负担不低

## 10. 总结评价

LightRAG 是"论文驱动的学术项目"成功进化为"生产级图 RAG 基础设施"的少数范例：算法上用双层检索替代社区报告，把图 RAG 的成本打到可用区间；工程上以 25 个存储后端、完整 API/WebUI/Ollama 兼容层、573 个测试文件与活跃的月度发版，覆盖了从本地玩具到 PG 生产集群的完整光谱。它是当前 GitHub 上事实上的 GraphRAG 开源替代标准。

适合谁用：需要在垂直领域（法律/金融/学术）做跨文档关联问答、且能接受索引期 LLM 成本的团队；有 PostgreSQL/Milvus/Neo4j 基础设施、想要"拿来即用"图 RAG 服务的工程团队。适合谁读源码：研究图 RAG 检索策略的算法工程师（operate.py 的双层召回与 token 预算裁剪是精华）、以及想学习可插拔存储架构与大型 asyncio 项目测试组织的 Python 工程师。
