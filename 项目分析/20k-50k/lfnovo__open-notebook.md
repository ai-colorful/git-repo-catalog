# lfnovo/open-notebook 项目分析

> 星数 41936（快照 2026-10-24）｜主语言 TypeScript（GitHub 统计；实际为 Python 后端 + Next.js/TS 前端）｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：**开源自托管的 NotebookLM 替代品**——把 PDF、网页、YouTube、音频等多源内容收进"笔记本"，提供向量/全文检索、引用式问答、内容变换（Transformation）和多说话人播客生成，支持 20+ 家模型供应商并可完全本地运行。

| 项 | 值 | 出处 |
|---|---|---|
| 名称/版本 | open-notebook，v1.15.1 | `pyproject.toml`、gh releases（v1.15.1，2026-11-04） |
| 星数 | 41936 | gh api（2026-11-14 核实） |
| 语言 | Python（约 2.2 万行）+ TypeScript（前端约 4.1 万行，故 GitHub 标 TypeScript） | 仓库内 wc 统计 |
| 许可证 | MIT | `LICENSE`、gh api license=mit |
| 后端框架 | FastAPI + LangGraph + SurrealDB | `pyproject.toml` |
| 前端 | Next.js 16 + React 19 + Tailwind 4 | `frontend/package.json` |
| 最近活跃 | 2026-11-13 仍在推送（gh api pushed_at） | gh api |
| 部署形态 | Docker Compose（SurrealDB + 应用单镜像），UI 8502 / API 5055 | `docker-compose.yml` |

## 2. 项目是什么，解决什么问题

Open Notebook 是 Google NotebookLM 的开源自托管复刻，主打三件 NotebookLM 做不到的事（`README.md` 对比表）：数据主权（自托管、可全本地）、模型自由（20+ 供应商，含 Ollama/LM Studio/vLLM 等 OpenAI 兼容端点）、更强的播客控制（1-4 个说话人自定义档案 vs 固定 2 人）。

核心工作流围绕"笔记本（Notebook）→ 源（Source）→ 笔记（Note）"三层（`open_notebook/domain/notebook.py`）：Source 是摄入的原始材料（文件/URL/文本），Note 是人工或 AI 产出的沉淀，SourceInsight 是对 Source 跑"变换"（Transformation，摘要/要点提取等，`open_notebook/domain/transformation.py`）得到的结构化洞察，可一键另存为 Note。所有内容统一进入检索层，供"Ask"深度问答、笔记本/单源聊天和播客生成复用。

与 NotebookLM 的差异化能力还包括：完整 REST API（`api/routers/` 约 20 个路由，可全自动化）、细粒度上下文开关（选择给模型看哪些源/笔记，`open_notebook/utils/context_builder.py`）、回答带引用（前端 `frontend/src/lib/utils/source-references.tsx`）、MCP 集成与可选密码保护。项目愿景明确写在 `VISION.md`，多用户、新输出格式、MCP agent 化列为"方向而非承诺"。

值得注意的工程文化：仓库自带 `AGENTS.md`/`CLAUDE.md`/`.claude/agents/` 等 AI 协作规范，`cubic.yaml` 接入 AI 代码评审，CHANGELOG（12 万字节）逐条写明问题号与修复理由——这是一套重度 AI 辅助开发 + 高质量评审流程的样板。

## 3. 技术栈与依赖

**后端（Python 3.12，uv 管理）**——`pyproject.toml`：
- FastAPI + uvicorn：REST API（`api/main.py`，启动时跑 SurrealDB 异步迁移）
- LangGraph + LangChain 1.x：所有 AI 工作流的状态图编排；`langgraph-checkpoint-sqlite` 做聊天持久化
- **esperanto**（同作者库）：统一 20+ 模型供应商的 LLM/嵌入/STT/TTS 抽象，经 `to_langchain()` 转 LangChain 模型（`open_notebook/ai/provision.py`）
- **content-core**（同作者库）：多源内容抽取——PDF、网页、YouTube 字幕、音频转写、Office 文档；可选 Docling（重解析/OCR）与 Crawl4AI 引擎（`open_notebook/graphs/source.py`）
- **podcast-creator**（同作者库）：大纲→逐字稿→TTS 合成的播客管线（`commands/podcast_commands.py`）
- **surreal-commands**：基于 SurrealDB 的异步任务队列（进程源/嵌入/播客三种 worker 命令，`commands/`）
- SurrealDB（官方 v2 镜像）：文档存储 + 图关系（reference/artifact 边）+ 全文/向量检索一体（`docker-compose.yml`）
- 其他：ai-prompter（Jinja2 提示词）、tiktoken（token 计数）、numpy（嵌入均值池化）、loguru

**前端（TypeScript）**——`frontend/package.json`：Next.js 16 App Router、React 19、Tailwind CSS 4、Radix UI 组件族、TanStack Query、zustand、react-hook-form + zod、i18next（15 种语言）、KaTeX + react-markdown；测试 vitest + Testing Library。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `open_notebook/` | 核心 Python 包：`domain/`（Pydantic 领域模型）、`ai/`（模型注册与供给）、`graphs/`（LangGraph 工作流）、`database/`（仓储层 + 27 个 SurrealQL 迁移）、`utils/`（分块/嵌入/加密/错误分类等）、`podcasts/`（播客档案模型） |
| `api/` | FastAPI 应用：`main.py` 装配、`routers/` 约 20 个路由、auth/CORS/体积限制中间件、podcast_service |
| `commands/` | surreal-commands worker：`source_commands.py`（process_source）、`embedding_commands.py`（embed_source/embed_note）、`podcast_commands.py`（generate_podcast） |
| `frontend/` | Next.js 前端（App Router），`src/app` 页面、`src/components` 组件、`src/lib`（API 客户端/hooks/i18n/状态） |
| `prompts/` | 9 个 Jinja2 提示词模板（ask/chat/source_chat/transformation/podcast） |
| `docs/` | 83 篇 Markdown：安装、用户指南、开发者文档（含 ADR） |
| `tests/` | 83 个后端 pytest 文件 + fixtures |
| `.github/workflows/` | test.yml（pytest+coverage+ruff+mypy、vitest）、build-and-release.yml、build-dev.yml |

组织方式评述：单仓三层（领域包 + API + 前端子目录），不是 monorepo 工具链管理而是约定式分层；作者把自己的五个基础库（esperanto/content-core/podcast-creator/ai-prompter/surreal-commands）外置为独立 PyPI 包，主仓只做产品装配——边界清晰但强耦合于作者个人生态。无 symlink，`.gitattributes` 仅有换行符规则，tarball 完整。

## 5. 核心架构与关键实现

### 5.1 领域模型与存储：Pydantic + SurrealDB 图文档一体（`open_notebook/domain/notebook.py`，963 行）

七类实体共用 `ObjectModel` 基类（`open_notebook/domain/base.py`）直接映射 SurrealDB 表：`notebook`、`source`、`note`、`chat_session`、`source_insight`、`source_embedding`、`episode/speaker_profile`。关系用图边表达：Source—`reference`→Notebook、Note—`artifact`→Notebook、ChatSession—`refers_to`→Notebook/Source。Source 可同时挂多个笔记本（共享），删除笔记本时区分"独占源（真删）"与"共享源（仅解链）"（`Notebook.delete()`，notebook.py:231）。嵌入单独存 `source_embedding` 表避免拉取全文时搬运向量。

### 5.2 摄入管线：content-core 抽取 → 变换 → 异步嵌入（`open_notebook/graphs/source.py` + `commands/source_commands.py`）

`source_graph` 是 LangGraph 图：content-core `extract_content()`（URL/文件/文本自动路由，YouTube 转写语言偏好列表在 source.py:34，无字幕时引导用户配 STT 模型）→ 可选执行若干 Transformation（独立变换图）→ 提交 embed_source 后台任务。整条链由 `process_source` worker 执行，重试策略精细（`commands/source_commands.py:45-66`）：max_attempts=15 + 指数抖动，但 ValueError/ConfigurationError 等永久错误不重试——重试只为 SurrealDB 事务冲突。引擎选择带运行时自愈：库内记录的 Docling/Crawl4AI 引擎若环境变量未启用，自动降级 "auto"（source.py:67-85，ADR-007）。

### 5.3 RAG 检索层：SurrealDB 原生向量 + 全文双路（`open_notebook/utils/embedding.py`、`chunking.py`、notebook.py 底部）

- 分块：内容类型感知（HTML/Markdown/纯文本分别用对应 LangChain splitter），默认 400 token、15% 重叠，环境变量可调（`chunking.py`）。
- 嵌入：批量（默认 50，可配）+ 3 次重试；超长文本先分块嵌入再做归一化均值池化合成单向量（`embedding.py` 的 `mean_pool_embeddings`）。
- 检索：查询编译成 SurrealQL 函数 `fn::text_search` / `fn::vector_search`（实现写在 27 个迁移文件中，如 `open_notebook/database/migrations/26.surrealql`），支持按笔记本范围过滤（migration 24 加的 notebook_ids 参数）。健壮性细节：全文检索遇 SurrealDB highlight "position overflow" 自动回退向量检索（notebook.py:892-912，issue #648）；Python 侧再兜底按相似度重排序（notebook.py:953-959，修复 #1431 排序失效）。
- Ask 图（`open_notebook/graphs/ask.py`）是三段式 Agentic RAG：策略模型产出≤5 个结构化搜索（PydanticOutputParser）→ `Send` 并行扇出到向量检索 + 逐路作答 → 最终综合，输出预算 8192 token，空搜索词/空回答都有显式失败路径（防推理模型把预算耗在思考上）。图结构（ask.py:194-203）：

```python
agent_state = StateGraph(ThreadState)
agent_state.add_node("agent", call_model_with_messages)      # 产出搜索策略
agent_state.add_node("provide_answer", provide_answer)       # 每路检索+作答
agent_state.add_node("write_final_answer", write_final_answer)
agent_state.add_edge(START, "agent")
agent_state.add_conditional_edges("agent", trigger_queries, ["provide_answer"])
agent_state.add_edge("provide_answer", "write_final_answer")
agent_state.add_edge("write_final_answer", END)
```

- Chat 图（`graphs/chat.py`）则是极简单节点图 + `SqliteSaver` 检查点持久化会话；`source_chat`/`transformation`/`prompt` 各有独立图，提示词全部外置在 `prompts/*.jinja` 便于定制。

### 5.4 多模型供给与播客管线（`open_notebook/ai/provision.py`、`commands/podcast_commands.py`、`open_notebook/podcasts/models.py`）

`provision_langchain_model()` 是所有图的模型入口：内容超 10.5 万 token 自动切"大上下文模型"，其次尊重显式 model_id，否则用该类型默认模型——模型选择策略集中在这一处。模型注册在 `open_notebook/ai/models.py`（Model/ModelManager，模型记录存库）+ `provider_registry.py`（供应商配置与连通性测试），用户在 UI 里配 key、选模型、指定默认分工（chat/embedding/tools/stt/tts/large_context 等类型）。

播客侧三张档案表（`open_notebook/podcasts/models.py`）：EpisodeProfile（outline_llm/transcript_llm/max_tokens）、SpeakerProfile（voice_model + 每说话人覆盖）、PodcastEpisode；`generate_podcast` worker 调 podcast-creator 库完成大纲→逐字稿→TTS，提示词在 `prompts/podcast/outline.jinja` 与 `transcript.jinja`。输出目录用 UUID 防路径注入（podcast_commands.py:45-59），失败信息映射成可操作提示（换模型/查 voice_id 等，podcast_commands.py:64-110）。生成的音频挂到 `./notebook_data` 卷下（`docker-compose.yml`）。

## 6. 构建与运行

来自 `docker-compose.yml`、`Makefile`、`README.md` 与 `frontend/package.json`：

```bash
# 用户部署（官方路径）：仅 docker-compose.yml + .env
curl -o docker-compose.yml https://raw.githubusercontent.com/lfnovo/open-notebook/main/docker-compose.yml
docker compose up -d          # UI http://localhost:8502，API :5055

# 开发（uv + 本地 SurrealDB）
uv sync
docker run -d -p 8000:8000 surrealdb/surrealdb:v2 start --user root --pass root
python run_api.py             # API；make backend/frontend 亦可（Makefile）

# 测试与质量
uv run pytest tests/ -v --cov=open_notebook --cov=api   # CI 同款（test.yml）
uv run ruff check . && uv run mypy .
cd frontend && npm run test   # vitest run
```

镜像为多阶段多架构构建（node:22 建前端 → python:3.12 + uv 装后端，预缓存 tiktoken 支持离线），另有 `--target single` 单容器变体（内置 SurrealDB，`Dockerfile`、`supervisord.conf`）。

## 7. 测试与工程质量

- 后端 83 个 pytest 文件（`tests/`），覆盖领域逻辑、全部路由（含 404/鉴权/CORS）、搜索排序、上传竞态（TOCTOU）、加密、迁移重试等；CI 跑覆盖率上传（`.github/workflows/test.yml`）。前端 43 个 vitest 测试文件，组件与 hooks 均有对应 `.test.tsx`。
- 质量工具链完整：ruff + mypy（`mypy.ini`）+ pre-commit（`.pre-commit-config.yaml`）+ pytest-cov。
- 代码可读性突出：几乎每个非显然决策都有注释并引用 issue 号（如 #648、#1431、#1466、#1510），CHANGELOG 按 Keep a Changelog 维护且条目极详。风格偏"防御式"——重试预算、事务冲突退避、路径逃逸校验（`utils/url_validation.py`、podcast 输出目录）、API 密钥加密存储（`utils/encryption.py`）。
- CI 有 SHA 固定的 action 版本与安全 floor 依赖约束（`pyproject.toml` constraint-dependencies），v1.15.1 即纯安全发布。

## 8. 社区与维护状态

- 活跃度：gh api 显示 2026-11-13（昨日）仍有 push；issue 134 个开放。tarball 内 CHANGELOG Unreleased 区已有大量条目，说明主干持续滚动。
- Release：30 个 release，近五次为 v1.15.1（2026-11-04，安全）、v1.15.0（2026-10-04）、v1.14.0（2026-07-21）、v1.13.0（2026-07-14）、v1.12.0（2026-07-12）——月度级发布节奏，版本名带主题。
- 商业化：仓库内未见付费代码或 license 检查，纯 MIT 开源；配套官网 open-notebook.ai 与 Discord 社区运营（README.md），未确认是否有托管服务等商业动作。
- 贡献者：contributors API 满页 100+（未确认精确总数）；README 招募前端/测试/文档贡献者，Discord 社区运营，多语言文档由 zdoc.app 自动同步、阿拉伯/土耳其语手动维护（README.md:44 注释）。
- 维护者结构：作者 Luis Novo（lfnovo）主导，配套自研库生态（esperanto/content-core/podcast-creator 均为其仓库）；重度依赖 AI agent 协作开发（仓库内 AGENTS.md 体系 + cubic.yaml 评审）。

## 9. 亮点与风险

**亮点**
1. 端到端产品完整度：摄入→检索→问答→变换→播客→REST API 全链闭环，且每环可替换模型（`open_notebook/ai/`）。
2. 自研基础库分层干净：模型抽象（esperanto）、内容抽取（content-core）、播客合成（podcast-creator）独立成包，主仓只做装配与产品逻辑。
3. 单一数据库承载文档+图+全文+向量（SurrealDB），部署面极小（两个容器），对自托管用户友好。
4. 工程质量显著高于同体量开源项目：注释引用 issue 号、防御式重试/降级遍布关键路径、83+43 个测试文件、安全响应快（v1.15.1 安全发布 + 依赖 floor 锁）。
5. Ask 三段式 Agentic RAG 设计简洁有效（`graphs/ask.py`），并行扇出 + notebook 范围过滤是实用增强。

**风险/局限**
1. 强绑定作者个人生态：esperanto/content-core/podcast-creator 任一断维护都直接影响核心链路（`pyproject.toml` 依赖表）。
2. 无内置多用户/团队协作（VISION.md 明示暂不做），鉴权仅单密码（`api/auth.py`），公网部署需自行加层。
3. 嵌入/检索逻辑分散在 SurrealQL 迁移函数 + Python 兜底排序中（`database/migrations/`、notebook.py 尾部），跨层调试与替换向量库成本高；SurrealDB 向量索引在大库上的性能上限未确认。
4. 单容器同时跑 UI/API/worker（supervisord），重摄入或播客合成时资源竞争；worker 并发默认仅 5（docker-compose.yml 注释）。

## 10. 总结评价

Open Notebook 是目前开源 NotebookLM 替代品中完成度最高的项目之一：功能面覆盖了 Google 原版的全部核心场景并额外提供 REST API、20+ 模型供应商与可控播客生成，部署门槛压到"一条 docker compose 命令"，同时以罕见的注释密度、测试覆盖和安全响应速度维护着 4 万星项目。它的架构选择（SurrealDB 一库多用、自研库分层、LangGraph 全编排）很适合想搭建个人知识库/研究助手产品的开发者参考；想学习"如何把 Agentic RAG 工程化"（重试、降级、引用、上下文控制）的人，`graphs/` + `domain/notebook.py` + `commands/` 这条主线是很好的教材；而想要多租户协作或企业级权限的团队，目前应等待其 VISION 中的方向落地或自行加层。
