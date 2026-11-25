# khoj-ai/khoj 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-14
>
> gh api 核实（2026-11-14）：stars 38124、forks 2549、license AGPL-3.0、语言 Python、默认分支 master、最近 push 2026-08-02。

## 1. 项目速览

一句话定位：Khoj 是一个开源可自托管的"AI 第二大脑"——把你的笔记（Markdown/org-mode/PDF/Word/Notion/GitHub）建成个人知识库，通过 RAG 检索 + LLM 进行问答、研究与自动化，支持网页、Obsidian、Emacs、桌面、Android、WhatsApp 等多端接入。

| 项目 | 内容 |
|---|---|
| 名称 | khoj-ai/khoj（原 Debanjum/khoj，现归 Khoj Inc. / khoj-ai 组织） |
| 星数 | 38124（gh api 2026-11-14；任务快照 39500 @2026-10-24） |
| 主语言 | Python（服务端）+ TypeScript/React（Web 前端与 Obsidian 插件） |
| 许可证 | AGPL-3.0-or-later（`pyproject.toml` 第 9 行 + `LICENSE`，gh api 核实一致） |
| 最近活跃 | 最近 push 2026-08-02；最新 release 2.0.0-beta.28（2026-03-26，gh api） |
| 创建时间 | 2021-08-16（gh api） |

## 2. 项目是什么，解决什么问题

Khoj 要解决的核心问题是：个人知识散落在 Obsidian 库、org-mode 文件、PDF、Word、Notion、GitHub 仓库等异构载体中，传统搜索只能做关键词匹配，无法回答"综合我的笔记，这个问题怎么看"这类需要理解与整合的问题。Khoj 把这些文档统一摄取为向量索引，再用 LLM 基于检索结果回答问题，形成"个人知识 + 大模型"的第二大脑（`README.md`，`src/khoj/processor/content/` 各处理器）。

项目从最初的 Emacs/org-mode 搜索工具演化为完整的个人 AI 平台：支持接入本地与在线 LLM（llama3、qwen、gemma、mistral、gpt、claude、gemini、deepseek，见 `README.md` 与 `src/khoj/processor/conversation/` 下 openai/anthropic/google 三套适配器）；支持创建带自定义知识、人设、模型与工具的 agent（`src/khoj/routers/api_agents.py`、`database/models` 中 `Agent` 模型）；支持自动化研究任务——按 cron 计划执行查询并把结果以邮件/通知送达（`src/khoj/routers/api_automation.py`、`routers/helpers.py` 的 `create_automation`）。

部署形态上强调"从本地单机平滑扩展到云端"：自托管用 `docker-compose.yml` 一键拉起 Postgres(pgvector)、SearxNG 搜索、terrarium 代码沙箱、khoj-computer 操作环境与 Khoj 服务；官方同时运营 app.khoj.dev 云端服务与企业版（`README.md` Enterprise 小节）。云版相关代码也开源在本仓库（Stripe 订阅 `routers/api_subscription.py`、Twilio/WhatsApp `routers/twilio.py`、邮件 `routers/email.py`，`pyproject.toml` 的 `[project.optional-dependencies].prod` 含 stripe/twilio/boto3/gunicorn）。除问答外还内置多模态能力：图像生成（`routers/helpers.py` 的 image prompt 链路）、语音对话（whisper 转写 + 语音回复）、Excalidraw/Mermaid 图表生成（`generate_excalidraw_diagram`/`generate_mermaidjs_diagram`），以及用户长期记忆（`routers/api_memories.py`、`UserMemory` 模型）。

README 显示团队已发布新一代独立产品 Pipali（khoj-ai/pipali，"跑在你电脑上的 AI coworker"），本仓库进入 2.0.0-beta 系列的稳定维护期（见第 8 章）。

## 3. 技术栈与依赖

- **服务端**：Python >=3.10,<3.13（`pyproject.toml`）。混合框架架构：**FastAPI** 承载 API 路由与 WebSocket，**Django 5.1** 承载 ORM、认证、admin（django-unfold）、迁移（`src/khoj/main.py`：Django 以 ASGI 子应用挂载到 `/server`）。
- **存储**：PostgreSQL + **pgvector**（`docker-compose.yml` 用 `pgvector/pgvector:pg15`；`database/models/__init__.py` 中 `Entry.embeddings = VectorField(dimensions=None)`）。
- **检索模型**：sentence-transformers + torch 2.6；默认 bi-encoder `thenlper/gte-small`、cross-encoder `mixedbread-ai/mxbai-rerank-xsmall-v1`（`database/models/__init__.py:558-566`、`processor/embeddings.py:32`）。
- **文档解析**：pymupdf（PDF）、docx2txt（Word）、rapidocr-onnxruntime（图片 OCR）、markdownify、langchain-text-splitters（分块）。
- **LLM 接入**：openai、anthropic、google-genai SDK；openai-whisper（语音转文字，`processor/conversation/openai/whisper.py`）。
- **agent 工具**：e2b-code-interpreter（代码沙箱，`processor/tools/run_code.py`）、MCP 客户端（`processor/tools/mcp.py`，支持 stdio/SSE 两类服务器）、SearxNG 网络搜索（`processor/tools/online_search.py`）。
- **任务调度**：APScheduler + django_apscheduler（`main.py`）。
- **Web 前端**：Next.js + React + TypeScript，bun 管理依赖，Tailwind + Radix UI + framer-motion（`src/interface/web/package.json`、`app/` 目录含 chat/search/agents/automations/settings 等路由页）。构建产物静态导出进 Django static（package.json `export` 脚本）。
- **客户端**：Obsidian 插件（TypeScript + esbuild，`src/interface/obsidian/`，manifest.json 显示版本 2.0.0-beta.28 已上 Obsidian 市场）、Emacs 包（`src/interface/emacs/khoj.el`）、Electron 桌面（`src/interface/desktop/`）、Android（Java，`src/interface/android/`）。
- **构建**：hatchling + hatch-vcs（版本取自 git tag），uv 锁定（`uv.lock`），ruff + mypy 质量工具（`pyproject.toml`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `src/khoj/main.py`、`configure.py` | 服务器组装：FastAPI+Django 启动、路由挂载、中间件、调度器 |
| `src/khoj/routers/` | API 层：api_chat（含 `/ws` WebSocket）、api_agents、api_automation、api_content（文件摄取入口）、api_model、api_memories、notion、auth、email、twilio、api_subscription（Stripe）、storage、research |
| `src/khoj/processor/content/` | 文档摄取管线：org_mode/markdown/pdf/docx/plaintext/images/notion/github 8 类处理器 + 公共基类 `text_to_entries.py` |
| `src/khoj/processor/conversation/` | 多模型适配：openai/gpt.py、anthropic/anthropic_chat.py、google/gemini_chat.py、whisper.py、prompts.py |
| `src/khoj/processor/tools/` | agent 工具：online_search.py（SearxNG）、run_code.py（e2b 沙箱）、mcp.py |
| `src/khoj/processor/operator/` | 计算机操作 agent：浏览器/桌面环境抽象 + anthropic/openai/uitars 多种操作模型实现 |
| `src/khoj/search_type/text_search.py` | 语义检索编排（bi-encoder 召回 + cross-encoder 重排） |
| `src/khoj/search_filter/` | 查询过滤语言：word_filter（+词/-词）、file_filter（glob）、date_filter（自然语言日期） |
| `src/khoj/database/` | Django models + adapters（2380 行的 `adapters/__init__.py` 是业务数据访问层）、99 个迁移文件 |
| `src/khoj/app/` | Django settings 等应用配置 |
| `src/interface/web/` | Next.js Web 前端（app router，chat/search/agents/automations/settings/share 页面） |
| `src/interface/obsidian/` 等 | Obsidian 插件、Emacs 包（`khoj.el` + ert 测试）、Electron 桌面、Android 客户端 |
| `tests/` | 24 个 pytest 测试文件（223 个测试函数）+ evals 质量评估 |
| `documentation/` | Docusaurus 文档站（部署到 docs.khoj.dev，github_pages_deploy.yml） |
| `scripts/` | 辅助脚本 |
| 顶层工程文件 | `pyproject.toml`/`uv.lock`（Python 打包与锁定）、`Dockerfile`/`prod.Dockerfile`/`computer.Dockerfile`（分层镜像）、`.github/workflows/` 11 条流水线 |

组织方式评述：单仓 monorepo，但边界清晰——`src/khoj` 是可发布的 Python 包（hatch 打包只含 `src/khoj`），四个客户端各成子项目独立构建发布；`pyproject.toml` 用 `[project.scripts] khoj = "khoj.main:run"` 提供单命令启动。models/adapters/routers/processor 分层是典型的 Django 变体：adapters 充当 service 层，routers 是 FastAPI 侧薄控制器。

## 5. 核心架构与关键实现

### 5.1 FastAPI + Django 双框架混合服务器

`src/khoj/main.py` 是理解整个项目的入口：启动时先跑 Django `migrate` 与 `collectstatic`（第 50-58 行），创建 FastAPI 主应用，把 Django ASGI 应用挂载到 `/server`（第 188 行），再由 `configure.py` 挂载 12 个路由组（`/api/chat`、`/api/agents`、`/api/automation`、`/api/model`、`/api/content`、`/api/notion` 等，`configure.py:331-356`）。分布式部署时用数据库 `ProcessLock` 做"调度领导者选举"——多实例中只有一个执行定时任务，其余 paused（`main.py:165-181`）。聊天走 WebSocket 流式输出（`routers/api_chat.py:1489` `@api_chat.websocket("/ws")`），CORS 预置了 Obsidian 桌面/iOS/Android 与 Khoj 桌面的来源（`main.py:73-80`），体现多端优先的设计。

### 5.2 文档摄取管线（8 源同步器 + 统一分块）

所有格式处理器继承 `TextToEntries` 抽象基类（`src/khoj/processor/content/text_to_entries.py`）。核心流程：格式解析 → 编译（标题/正文/行号）→ 分块 → 嵌入 → 写库。分块用 LangChain `RecursiveCharacterTextSplitter`，256 token 上限、段落>句子>词的切分优先级，非首块自动回填标题前缀保持上下文；并对 chunk 做行号回溯（把 chunk 在 raw 文本中的偏移换算为 `file://...#line=N` 的 URI，保证引用可跳转到源文件原行，`text_to_entries.py:87-118`）。超长词直接丢弃而非截断，避免污染嵌入质量（`remove_long_words`）。各源处理器：org_mode（自研 orgnode 解析）、markdown、pdf（pymupdf）、docx（docx2txt）、images（rapidocr OCR）、plaintext、notion（API 拉取，`routers/notion.py` 做 OAuth 与增量同步）、github（仓库文件同步）。内容源配置持久化在 Django 模型（`NotionConfig`、`GithubConfig` 等）。

### 5.3 RAG 检索层：bi-encoder 召回 + 过滤语言 + cross-encoder 重排

检索入口 `src/khoj/search_type/text_search.py` 的 `query()`：用 bi-encoder 嵌入查询 → `EntryAdapters.search_with_embeddings` 在 pgvector 上算余弦距离、按置信阈值过滤（默认 0.18，`models/__init__.py:582`）、取 top_k=10 → `rerank_and_sort_results()` 用 cross-encoder 重排。向量检索的核心就是一段 Django ORM 注解（`database/adapters/__init__.py:2150-2178`）：

```python
relevant_entries = EntryAdapters.apply_filters(user, raw_query, file_type_filter, agent)
relevant_entries = relevant_entries.filter(owner_filter).annotate(
    distance=CosineDistance("embeddings", embeddings)
)
relevant_entries = relevant_entries.filter(distance__lte=max_distance)
if file_type_filter:
    relevant_entries = relevant_entries.filter(file_type=file_type_filter)
relevant_entries = relevant_entries.order_by("distance")
return relevant_entries[:max_results]
```查询支持内联过滤语法，在 `apply_filters`（`adapters/__init__.py:2090-2147`）中解析为 SQL 条件：`+词`/`-词` 做包含/排除、glob 模式过滤文件路径、自然语言日期（"last week"）经 `date_filter` 转为 `EntryDates` 日期范围。需要说明：召回是"向量 + 结构化过滤"的混合，并非倒排全文索引（全库未见 tsvector/BM25），关键词匹配层通过 `Q(raw__icontains=...)` 实现；另有本地 grep 工具（`tests/test_grep_files.py` 对应的 RegexSearchFiles）补正则检索。嵌入支持增量更新——按条目 ID 合并旧嵌入、只编码新条目并持久化 torch 张量（`text_search.py:compute_embeddings`），也可把编码/重排外包给推理服务器（`SearchModelConfig` 的 `*_inference_endpoint` 字段）。

### 5.4 Agent 编排：迭代式工具循环 + MCP 扩展 + 自动化

研究模式的核心是 `src/khoj/routers/research.py`（706 行）的迭代循环：`apick_next_tool()`（第 277 行起）把可用工具渲染成 LLM 的 function-calling 菜单（`ToolDefinition` 统一各家 schema 格式，`utils/helpers.py:343-366`），LLM 选择工具后 `execute_tool()` 执行，结果累积进 `ResearchIteration`，直到 LLM 选择 Text 工具（即给出最终答案）或达到 `max_iterations=5`。工具箱：SemanticSearchFiles/RegexSearchFiles/ViewFile/ListFiles（文档四件套）、SearchWeb（SearxNG，子查询去重）、ReadWebpage、PythonCoder（e2b/terrarium 沙箱）、OperateComputer（`processor/operator/` 计算机操作 agent）、以及外部 MCP 服务器工具（`processor/tools/mcp.py` 的 `MCPClient` 支持 stdio/SSE，工具名带 `server/tool` 命名空间，`research.py:376-391`）。工具暴露受 agent 配置与服务器开关约束（无笔记隐藏文档工具、未开沙箱隐藏代码工具等，`research.py:344-366`）；用户侧 agent 的五类输入工具（Notes/Webpage/Online/Code/Operator）映射到上述研究工具子集（`research.py:331-337`）。自动化任务（`routers/helpers.py:2624 create_automation`）把"定时执行查询"注册为 APScheduler cron job，执行后经 `should_notify` 判断是否值得打扰用户再发邮件/通知。多模型接入由 `processor/conversation/` 三家适配器 + `ChatModel` 数据库配置实现，含思考深度分级（`ServerChatSettings` 的 think/free/deep 字段）。

## 6. 构建与运行

```bash
# 自托管（官方推荐，docker-compose.yml）
docker compose up -d          # 拉起 pgvector/pg15 + searxng + terrarium 沙箱 + khoj 服务（默认端口 42110）

# 从源码运行服务端（pyproject.toml）
uv pip install -e ".[dev]"    # 或 pip install -e .；本地 PG 可用 [local] extra（pgserver）
khoj --verbose                # 入口 khoj = "khoj.main:run"，单命令完成 migrate+collectstatic+启动

# Web 前端开发（src/interface/web/package.json）
bun install && bun run dev    # next dev；bun run export 构建静态产物并拷回 Django static

# 测试（tests/，pytest.ini）
pytest tests/ -x              # 集成测试需 Postgres；tests/evals/eval.py 跑对话质量评估

# Obsidian 插件（src/interface/obsidian/）
npm install && npm run dev    # esbuild 打包

# Emacs 客户端（src/interface/emacs/）
# khoj.el 发布于 MELPA；本地 ert 测试：emacs -batch -l ert -l tests/khoj-tests.el -f ert-run-tests-batch
```

## 7. 测试与工程质量

- 测试规模：`tests/` 下 24 个 `test_*.py`、223 个测试函数，pytest 生态（pytest-django、pytest-asyncio、pytest-xdist 并行、factory-boy 造数、freezegun 冻结时间）。覆盖面偏集成：每个格式处理器都有对应 `test_*_to_entries.py`，检索（`test_text_search.py`）、多用户隔离（`test_multiple_users.py`）、自动化、agent、CLI 均有专项；pyproject 中定义 `chatquality` 标记区分对话质量评估与常规测试。
- 客户端测试同样成体系：Emacs 端 ert 测试（`src/interface/emacs/tests/khoj-tests.el`）有独立 CI 工作流 `test_khoj_el.yml`，Obsidian 插件随 release.yml 构建校验。
- 质量评估：`tests/evals/eval.py` + CI `run_evals.yml` 专门跑检索与问答质量基准（README 也宣称有 benchmark 成绩），这是同类自托管 RAG 项目少见的投入。
- CI：`.github/workflows/test.yml` 对 src/khoj/** 变更触发，Python 3.10/3.11/3.12 三版本矩阵 + 容器化测试；另有 `pre-commit.yml`（ruff/mypy/pre-commit）、`test_khoj_el.yml`（Emacs Lisp 测试，有 ert 测试文件 `src/interface/emacs/tests/khoj-tests.el`）、`desktop.yml`、`dockerize.yml`、`pypi.yml`、`release.yml`（tag 触发，发布 Obsidian 插件与多端产物）。
- 代码质量观察：ruff（E/F/I，行宽 120）+ mypy 全量类型检查 `src/khoj`；`adapters/__init__.py` 单文件 2380 行、`routers/helpers.py` 3458 行、`api_chat.py` 1761 行，几个巨型文件是主要的技术债；99 个 Django 迁移反映五年快速迭代的历史，近期迁移已合并清理（如删除 `SearchModelConfig` per-user 配置）。

## 8. 社区与维护状态

- 活跃度：gh api 显示最近 push 2026-08-02（最后提交 "Make chat export robust and fix export truncation #1314"），2026-06-24 前提交密集、之后明显放缓；贡献者 76 人（含 anon），fork 2549，open issues 170。
- 发版节奏：tag 触发 release，Obsidian 插件 versions.json 记录了 1.0 到 2.0.0-beta.28 共 170+ 个版本；gh api 显示 2025-12 至 2026-03 保持约每月 1-2 个 beta（beta.23 于 2025-12-29、beta.28 于 2026-03-26），最新为 2.0.0-beta.28。2026 年 4 月后无新 release，开发重心似已转向姊妹项目 Pipali（README "New" 小节）。
- 团队：pyproject 作者署名 Debanjum Singh Solanky 与 Saba Imran（原始开发者），近期提交含 Debanjum 及团队成员（SyncWithRaj、Kobi Hikri 等）；README 顶部为 Khoj Inc. 品牌（app.khoj.dev 云服务、Discord 社区、blog.khoj.dev）。
- 商业化：开源核心 + 云托管/企业版双轨。云版代码（Stripe 订阅、Twilio WhatsApp、Resend 邮件、boto3）同仓库维护，通过 `prod` extra 与 `api_subscription` 路由区分；docker 镜像分 `ghcr.io/khoj-ai/khoj`（社区）与 `khoj-cloud`（生产）两条线（`docker-compose.yml` 注释、`.github/workflows/dockerize_telemetry_server.yml`）。遥测独立成服务（`src/telemetry/` + 独立部署流水线）。

## 9. 亮点与风险

**亮点**
1. 多端体验完整度罕见：Obsidian/Emacs/浏览器/桌面/Android/WhatsApp 六种客户端同源一个后端 API，CORS 与认证都为此专门设计（`main.py:73-80`、`routers/twilio.py`）。
2. RAG 工程细节扎实：分块保留行号引用、超长词丢弃保护嵌入质量、嵌入增量合并持久化、置信阈值过滤、可选 cross-encoder 重排与远程推理端点（`text_to_entries.py:87-118`、`text_search.py:40-84`）。
3. Agent 层可扩展性好：统一 `ToolDefinition` 抽象 + MCP 协议接入外部工具 + 计算机操作 agent，工具暴露随环境自动裁剪（`research.py`）。
4. 有质量评估文化：专门的 evals 工作流与 benchmark 投入（`tests/evals/eval.py`、`.github/workflows/run_evals.yml`）。
5. 单命令自托管友好：`khoj` CLI 自动完成迁移与静态资源，compose 一键全家桶含搜索/沙箱/操作环境。

**风险/局限**
1. 重度依赖重 ML 栈（torch 2.6 + transformers），本地部署内存门槛高；向量检索只在 Postgres 单库内做，无独立向量引擎，超大规模语料（百万级 chunk）性能未验证。
2. 维护节奏放缓：2026 年 4 月后无 release、最近 push 距今 3 个多月，团队重心转移 Pipali，2.0 仍处 beta。
3. 代码集中度偏高：adapters/helpers/api_chat 三个文件合计 7500+ 行，二开定位与修改成本较高。
4. AGPL-3.0 对商业闭源集成不友好（服务化改造需开源），自托管云服务场景需注意合规。

## 10. 总结评价

Khoj 是"个人 AI 第二大脑"赛道中完成度最高的开源实现之一：文档摄取、混合检索、多模型对话、agent 工具循环、自动化任务、六端客户端、云版商业化在同一个仓库里形成了完整闭环，且 AGPL 全开。它的代码是学习"如何把 RAG + LLM agent 做成可持续维护的产品"的优质样本——尤其推荐精读 `processor/content/text_to_entries.py`（摄取管线）、`search_type/text_search.py` + `database/adapters/__init__.py`（pgvector 检索与过滤语言）、`routers/research.py`（迭代式 agent 编排）。适合想自托管个人知识助手的知识工作者直接使用；对想构建同类产品的开发者，它是比 LangChain 模板更贴近真实产品形态的参考实现；但需要注意其维护节奏已放缓、重型依赖与 AGPL 许可带来的约束。
