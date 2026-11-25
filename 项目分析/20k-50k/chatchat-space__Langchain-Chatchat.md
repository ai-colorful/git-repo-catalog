# chatchat-space/Langchain-Chatchat 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：中文场景最知名的本地知识库 RAG + Agent 一体化开源应用（原 langchain-ChatGLM），可全离线部署，通过 OpenAI 兼容 API 与 Streamlit WebUI 对外服务。

| 项目 | 内容 |
|---|---|
| 名称 | chatchat-space/Langchain-Chatchat（原 langchain-ChatGLM，2023-08 改名并迁至独立组织） |
| 星数 | 38,715（gh api 实测 2026-11-14；快照 39,500） |
| 语言 | Python（gh api 确认） |
| 许可证 | Apache-2.0（LICENSE 文件；注意 libs/chatchat-server/pyproject.toml 内却写 `license = "MIT"`，两处不一致） |
| 当前版本 | master 上 `__version__ = "0.3.1.3"`（libs/chatchat-server/chatchat/__init__.py）；最新 release 为 v0.3.1（2024-07-12，gh api） |
| 最近活跃 | 最近 push 2025-11-10（gh api），master 上仍在开发 MCP 等新功能但未发版 |
| 贡献者 | 约 155 人（contributors 分页 last=155，gh api） |
| 服务规模 | 服务端 Python 代码约 2.33 万行（wc -l 统计 chatchat/ 与 langchain_chatchat/） |

## 2. 项目是什么，解决什么问题

Langchain-Chatchat 是"利用 langchain 思想实现的基于本地知识库的问答应用"（README.md 概述），目标是建立一套**对中文场景与开源模型支持友好、可离线运行**的知识库问答方案。典型 RAG 流程为：加载文件 → 文本分割 → 向量化 → 相似度匹配 top-k → 拼入 prompt → LLM 生成回答（README.md 实现原理一节）。项目不涉及微调训练，专注于把开源 LLM + Embedding + 向量库串成开箱即用的私有化问答服务。

它解决的核心痛点是：企业/个人想在不出内网的前提下，用 ChatGLM、Qwen、Llama 等开源模型对自己的文档做问答。项目通过对接 Xinference、Ollama、LocalAI、FastChat 等推理框架（README 模型部署框架表）实现全离线链路，同时也能通过 One API 接入 OpenAI/Azure/Claude/智谱/百川等在线 API。

0.3.x 版本将功能大幅扩展（README 0.3.x 功能一览表）：统一 File RAG（BM25+KNN 多路检索）、数据库对话（text2sql）、多模态图片对话、ARXIV/Wolfram/搜索引擎对话、文生图，核心能力由 Agent 驱动。0.3.x 的所有模型接入均兼容 OpenAI SDK（README 对比表），这一标准化是 0.2.x→0.3.x 重构的关键转变——0.2.x 用 fastchat+自研 ModelWorker，0.3.x 改为统一的 model_provider/oneapi 平台配置。

关于"剥离 langchain"：实际核实，0.3.1.3 仍依赖 langchain 0.1.17 / langchain-community 0.0.36（libs/chatchat-server/pyproject.toml），并非完全剥离；但项目新建了自研适配层 `langchain_chatchat` 包（ChatPlatformAI、PlatformToolsRunnable、GLM3/Qwen 输出解析器、MCP kit），把对各家模型平台与 Agent 协议的适配收拢为自己可控的代码（libs/chatchat-server/langchain_chatchat/）。

命名与组织迁移历史（README 项目里程碑）：2023-04 以 langchain-ChatGLM 0.1.0 起家 → 2023-08 改名 Langchain-Chatchat 并发布 0.2.0 → 2023-10 推出 Agent、获 Founder Park & 智谱 & Zilliz 黑客马拉松三等奖 → 2023-12 破 20K stars → 2024-06 发布重构后的 0.3.0。它是 THUDM/ChatGLM 生态孵化出的第一个大规模知识库问答标杆，也是国内 RAG 应用生态的事实起点之一。

## 3. 技术栈与依赖

- 语言：Python >=3.10,<3.12（libs/chatchat-server/pyproject.toml；根 pyproject.toml 为 workspace 外壳 package-mode=false）
- Web 框架：FastAPI ~0.109.2 + uvicorn + sse_starlette（流式 SSE 输出）
- WebUI：Streamlit 1.34.0 及其组件生态（streamlit-chatbox、streamlit-antd-components、streamlit-aggrid 等，pyproject.toml webui 段）
- Langchain 链：langchain 0.1.17、langchain-community 0.0.36、langchain-openai 0.0.6、langchain-experimental 0.0.58（pyproject.toml）
- RAG 基础件：faiss-cpu ~1.7.4（默认向量库）、jieba 0.42.1 + rank_bm25 0.2.2（BM25 中文检索）、sentence_transformers（CrossEncoder 重排，reranker/reranker.py）、nltk、unstructured ~0.11.0、PyMuPDF、rapidocr_onnxruntime（OCR）、python-docx
- 向量库后端：elasticsearch、pymysql/pg（relyt）、milvus/zilliz、chromadb（经 kb_service 适配器接入）
- 数据库：SQLAlchemy ~2.0.25（知识库元数据，默认 sqlite）
- MCP：mcp >=1.4.1,<1.5（agent_toolkits/mcp_kit，改编自 langchain-mcp-adapters）
- 可观测性：langfuse（kb_chat.py 中按环境变量接入）
- 工程：Poetry 1.7.1 + click CLI + ruff + loguru + pydantic v2/pydantic-settings（YAML 配置）

## 4. 目录结构与代码组织

monorepo 组织（docs/contributing/repo_structure.md 明确说"chatchat 沿用了 monorepo 的组织方式"，直接借用了 langchain 仓库的模板）：

| 目录 | 职责 |
|---|---|
| libs/chatchat-server/ | 服务端主包（发布名 langchain-chatchat），含 chatchat/ 与 langchain_chatchat/ 两个包 |
| libs/chatchat-server/chatchat/server/ | 服务端核心：api_server、chat、knowledge_base、file_rag、reranker、db、agent、agents_registry |
| libs/chatchat-server/chatchat/webui_pages/ | Streamlit 页面：dialogue、kb_chat、knowledge_base、mcp、model_config |
| libs/chatchat-server/langchain_chatchat/ | 自研 Agent/Chat 适配层：agents（GLM3/Qwen/structured-chat 解析器）、chat_models、agent_toolkits（含 mcp_kit）、callbacks |
| libs/chatchat-server/chatchat/data/ | 随包分发的样本知识库（大模型技术系列 md）与 nltk_data |
| libs/python-sdk/ | Python 客户端 SDK（包目录名 open_chatcaht，拼写沿用上游未改） |
| docker/ | Dockerfile + docker-compose.yaml（xinference + chatchat 双容器） |
| docs/ | 部署与开发文档（contributing、install） |
| .github/workflows/ | 测试、集成测试、release、docker 构建 CI |

组织方式评述：双包结构清晰分离"平台适配层"与"应用层"，配置全部 YAML 化（settings.py 定义 BasicSettings/KBSettings/ApiModelSettings/ToolSettings/PromptSettings 五组，由 `chatchat init` 生成到 CHATCHAT_ROOT）。repo_structure.md 还提到规划中的 frontend 目录，本快照中不存在（未落地）。

## 5. 核心架构与关键实现

### 5.1 API 服务层：统一入口 + OpenAI 兼容面

FastAPI 应用在 server/api_server/server_app.py 的 `create_app()` 中组装，挂载 6 个 router：chat、kb、tool、openai、server、mcp，另有 `/other/completion` 补全接口。其中 openai_routes.py 提供完整 OpenAI 兼容面（`/models`、`/chat/completions`、`/completions`、`/embeddings`、`/images/generations|variations|edit`），使任何 OpenAI SDK 客户端可直接把 Chatchat 当模型端点用；chat_routes.py 的 `/chat/completions` 是兼容 openai 的统一 chat 接口（含 RAG/Agent 能力）。多进程启动由 startup.py 负责：api 进程跑 uvicorn，webui 进程跑 streamlit bootstrap（startup.py L158-161、L259-296）。

### 5.2 知识库管线：多后端适配器 + BM25/向量混合检索

知识库后端以 KBService 抽象（server/knowledge_base/kb_service/base.py，518 行）+ 工厂模式实现 8 种后端：faiss（默认）、milvus、zilliz、pg、relyt、es、chromadb、default（kb_service/ 目录）。检索默认走 ensemble 混合检索（faiss_kb_service.py `do_search`），实现于 file_rag/retrievers/ensemble.py：

```python
faiss_retriever = vectorstore.as_retriever(
    search_type="similarity_score_threshold", ...)
docs = list(vectorstore.docstore._dict.values())
bm25_retriever = BM25Retriever.from_documents(
    docs, preprocess_func=jieba.lcut_for_search)
ensemble_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, faiss_retriever], weights=[0.5, 0.5])
```

即 jieba 分词 BM25 与 FAISS 向量各占 0.5 权重的混合召回；重排用 sentence_transformers CrossEncoder 封装的 LangchainReranker（reranker/reranker.py）。FAISS 实例通过 kb_cache/faiss_cache.py 的 ThreadSafeFaiss + kb_faiss_pool 做线程安全的加载池化，避免并发重复加载。中文优化的切分器在 file_rag/text_splitter/（chinese_text_splitter.py、chinese_recursive_text_splitter.py、zh_title_enhance.py 标题增强），文档加载器含 OCR（rapidocr）与 PDF/PPT/IMG 适配（document_loaders/）。

### 5.3 多模型接入与 Agent 链

模型平台配置统一为 PlatformConfig（settings.py L258 起）：platform_type 支持 xinference/ollama/oneapi/fastchat/openai/custom openai，api_concurrencies 控制单模型并发，`auto_detect_model` 可自动发现平台上可用的 llm/embed/rerank/text2image/image2text/speech 模型列表。默认模型 glm4-chat + bge-m3（ApiModelSettings，settings.py）。自研适配层 langchain_chatchat/ 中：chat_models/base.py 的 ChatOpenAI 封装、agents/output_parsers/ 下针对 GLM3 与 Qwen 的工具调用输出解析器、agents_registry.py 统一注册；Agent 工具集在 agent_toolkits/all_tools/（代码解释器、文生图、浏览器）与 mcp_kit/（stdio/SSE 双 transport 的 MCP 客户端，client.py 改编自 langchain-mcp-adapters 并注明出处）。kb_chat.py（server/chat/kb_chat.py）是 RAG 对话编排核心：local_kb/temp_kb/search_engine 三种知识来源 → 检索 → format_reference 引用渲染 → 可选 Langfuse 回调 → 流式输出 OpenAIChatOutput。

### 5.4 WebUI 与 MCP 管理

Streamlit WebUI 提供 4 个页面：多功能对话、RAG 对话、知识库管理、MCP 管理（webui.py）。MCP 管理是 master 上新增（未发版）的功能：mcp_routes.py 提供完整的 MCP profile/连接 CRUD API（/profile、/ 连接列表等 10+ 端点），配合 webui_pages/mcp/ 可视化管理 MCP 服务器连接，说明项目正把 Agent 工具生态转向 MCP 协议。

## 6. 构建与运行

```bash
# pip 安装部署（README 安装部署节）
pip install langchain-chatchat -U
# 或源码安装
poetry install   # libs/chatchat-server 下，poetry 1.7.1

# 初始化（生成配置 yaml、复制 samples 知识库、建表）
chatchat init [-x xinference地址] [-l glm4-chat] [-e bge-large-zh-v1.5] [-r]
# 重建知识库
chatchat kb -r
# 启动 API(7861) + WebUI(8501)
chatchat start -a

# Docker 部署（docker/docker-compose.yaml）
docker compose up -d   # xinference: v0.12.3 + chatchat:0.3.1.3 双容器

# 测试（.github/workflows/_test.yml）
pytest tests/unit_tests    # 单测
pytest tests/              # 全量（api/kb_vector_db 需真实服务）
```

## 7. 测试与工程质量

- 测试规模：22 个测试文件（tests/ 目录）。分四类：api/ 集成测试（test_kb_api、test_stream_chat_api 等 8 个）、kb_vector_db/ 各向量库实测（faiss/milvus/pg/relyt）、integration_tests/（MCP platform tools、SDK 导入）、unit_tests/（mcp_prompts 等）。python-sdk 另有 5 个测试脚本。
- CI：_test.yml 三平台矩阵（ubuntu/windows/macos × Python 3.8-3.11，注意与 pyproject 要求的 >=3.10 不一致，矩阵沿用 langchain 模板未收紧）；_integration_test.yml、_release.yml（PyPI 发布，仅 master 触发）、docker-build.yaml、issue 管理自动化（close-issue.yml、label_ad_issue.yml）。
- 质量观察：代码注释以中文为主、覆盖率高，关键路径（并发 FAISS 池、流式回调 AsyncIteratorCallbackHandler）处理细致；但代码中残留较多 TODO（settings.py 多处"似乎重复"）、注释掉的调试块（kb_chat.py、reranker.py），且 humanlayer=0.7.6 这类依赖声明格式可疑，整体工程质量属"活跃但粗糙"。

## 8. 社区与维护状态

- 活跃度：gh api 显示最近 push 2025-11-10，仓库未归档；但发版明显放缓——最新 release v0.3.1 停在 2024-07-12，master 上的 0.3.1.3 + MCP 新功能均未正式发版（README 徽章引用 trendshift，社区热度仍在）。
- 社区结构：约 155 名贡献者；核心作者 Liu Qian(qdaxb)、glide-the、liunux4odoo 等（README 引用条目）；6.2k forks、23 open issues（gh api）。
- 生态位：中文 LLM 知识库问答的事实标准起点，2023 年国内 RAG 项目的共同上游/参照；提供微信交流群、公众号、Telegram（README 联系我们），并给出 BibTeX 引用条目，学术与工业引用广泛。
- Docker 镜像双源（Docker Hub chatimage + 腾讯云国内镜像），AutoDL 一键镜像（README），对国内用户部署友好。

## 9. 亮点与风险

**亮点**
1. 全链路私有化：开源模型 + 开源向量库 + 可离线部署，且全部模型接入兼容 OpenAI SDK，一个配置文件切换 Xinference/Ollama/One API 平台（settings.py PlatformConfig）。
2. 中文 RAG 工程化完整：jieba BM25 + FAISS 混合检索、CrossEncoder 重排、中文文档切分器、OCR、标题增强，这些中文特化是同期英文项目不具备的（file_rag/ 目录）。
3. 8 种向量库后端 + 工厂适配器，实际生产可选型灵活（kb_service/）。
4. 面向 Agent 的演进路线清晰：GLM3/Qwen 专属输出解析器、结构化 Agent、MCP 双 transport 客户端与可视化管理（langchain_chatchat/、mcp_routes.py）。
5. 文档与部署体验好：CLI 三步走（init → kb -r → start -a）、docker-compose、样本知识库随包分发。

**风险/局限**
1. 维护节奏放缓：超过两年未发正式版（v0.3.1，2024-07），MCP 等新能力长期停留在 master；使用者需自行承担未发版代码的风险。
2. 依赖链陈旧且锁定：langchain 0.1.17 时代早已被上游淘汰，无法平滑跟进新版 langchain；unstructured/nltk 等重依赖安装问题多（README 专门写了 Windows python-magic-bin FAQ）。
3. 元数据不一致：pyproject.toml 声明 MIT 而 LICENSE 为 Apache-2.0；CI 测试矩阵 Python 3.8 与包要求 >=3.10 矛盾，工程收尾不够严谨。
4. 性能设计偏 demo→生产过渡态：BM25 每次从 docstore 全量构建（ensemble.py TODO 注释承认），大知识库下召回开销与缓存策略存在瓶颈；默认 top_k=3、SCORE_THRESHOLD=2.0 等默认值偏保守（settings.py）。

## 10. 总结评价

Langchain-Chatchat 是中文 RAG 应用工程化的里程碑项目：它把"本地知识库问答"从论文和 notebook 变成了普通人 pip install 就能跑起来的完整产品，0.3.x 重构确立的"OpenAI 兼容 API + 平台化模型接入 + Agent 化功能"设计影响了后续大量国内同类项目。虽然当前依赖链与发版节奏显示项目已进入低维护期（新方向疑似转向 MCP 与下一代架构但未落地），其源码仍是学习"如何把 RAG 管线、多模型适配、混合检索、流式服务做工程化封装"的最佳中文范本之一。适合：想私有化部署知识库问答的团队（注意锁定版本与依赖）、想借鉴中文 RAG 工程细节的开发者；不太适合追求新栈（langchain 0.3+/LlamaIndex 生态）或需要长期官方维护保障的生产系统。
