# reworkd/AgentGPT 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-17

## 1. 项目速览

一句话定位：浏览器里"起个名字、给个目标就能跑"的自主 AI Agent 平台，是 2023 年 AutoGPT 浪潮的网页版代表作品（AutoGPT 官方 Web UI 即由它演化而来）。

| 项 | 值 |
|---|---|
| 名称 | reworkd/AgentGPT |
| 星数 | 39500（快照）；gh api 实测 36268（2026-11-17，归档后只减不增） |
| 主语言 | TypeScript（前端）+ Python（FastAPI 后端） |
| 许可证 | GPL-3.0（`LICENSE`，gh api 确认） |
| 创建时间 | 2023-04-07（gh api） |
| 最近活跃 | 仓库已 archived（gh api 实测）；最后一次 push 2025-04-29（仅 docs 更新），实质性开发止于 2024 年 |
| 官网 | https://agentgpt.reworkd.ai |

## 2. 项目是什么，解决什么问题

AgentGPT 让用户在浏览器中"装配、配置并部署自主 AI Agent"：输入一个目标（如"调研某市场并写报告"），Agent 会自动分解任务、逐个执行（联网搜索/写代码/生成图片）、根据结果反思并追加新任务，循环往复直到目标完成或达到上限。它把 AutoGPT 式的"目标→任务分解→工具执行→反思追加"循环从命令行搬到了网页上，是 2023 年 4 月爆火的现象级项目（README `README.md`）。

核心卖点有三：一是零安装、纯浏览器体验，打开网页输入目标即可；二是可视化执行过程，任务列表、思考过程、执行结果实时流式展示；三是开源自部署，提供一键 setup CLI + docker-compose（README "Getting Started"，`docker-compose.yml`）。

与 AutoGPT 相比，AgentGPT 刻意做减法：不追求本地文件系统/Shell 等高危能力，工具面收窄为搜索、图片生成、代码、推理等（`platform/reworkd_platform/web/api/agent/tools/tools.py`），更像一个"目标驱动的联网研究助手"，这也是它能做成公开托管服务的原因。

解决了三类用户的真实需求：

- **普通用户**：不需要装 Python/配 API 环境也不需要懂 prompt，输入一句中文/英文目标就能围观 agent 自主干活，是 2023 年大众认识 agent 概念的第一入口。
- **开发者/自部署者**：docker-compose + setup CLI 十分钟内可跑起完整私有实例（前端/后端/MySQL 三容器），GPL-3.0 下可自由魔改。
- **Reworkd 公司**：托管版按订阅计费（Stripe 内建，`platform/reworkd_platform` 依赖 stripe），开源仓库本身即获客漏斗与信任背书。

公司背景：项目由 Reworkd（YC W23 创始人 asim-shrestha、awtkns 等，见 `platform/pyproject.toml` authors）开发。Reworkd 后来转向企业级 Agent 数据抽取产品（AgentGPT 托管服务成为其获客入口），开源仓库随之进入维护停滞，最终于 2025 年被 GitHub 归档（gh api `archived: true`）。

## 3. 技术栈与依赖

前后端分离的双栈 monorepo：

- **前端（next/）**：Next.js 13 + TypeScript + React 18（create-t3-app 脚手架，README），tRPC v10 + TanStack Query、Next-Auth（Google/GitHub/Discord OAuth）、Prisma ORM + MySQL（Planetscale）、TailwindCSS + HeadlessUI + Radix、Zustand 状态管理、Zod 校验、i18next 多语言（`next/package.json`，47 个 dependencies）。
- **后端（platform/）**：Python 3.11 + FastAPI 0.98 + SQLAlchemy 2.0（async）+ aiomysql、SQLModel、LangChain 0.0.295 + openai Python SDK 0.28（2023 年代的旧版 API）、pydantic v1、lanarky（LangChain 流式响应）、tiktoken 计 token、Sentry + Loguru、Stripe 计费、aiokafka、boto3/AWS Secrets Manager（`platform/pyproject.toml`）。
- **外部服务**：OpenAI/Azure OpenAI（唯一 LLM 通道）、Serper.dev（Google 搜索）、Replicate（图片生成）、Helicone（LLM 网关/观测，可选）、Pinecone（向量记忆，`services/pinecone/`）、SID（浏览器代理工具，`@sid-hq/sid`）。
- **基础设施**：MySQL 8.0（`db/setup.sql`）、Docker Compose 三容器（frontend/platform/agentgpt_db）、setup CLI（`cli/`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `next/` | Next.js 13 前端 + tRPC 层：页面、组件、agent 客户端循环、Prisma |
| `platform/` | FastAPI 后端 `reworkd_platform`：agent 执行引擎、工具、模型工厂、OAuth、计费 |
| `cli/` | 一键安装 CLI（生成 .env、拉起 docker-compose） |
| `db/` | MySQL 初始化 SQL 与 Dockerfile |
| `docs/` | Mintlify 文档站（中/英/匈牙利语 README 亦在此） |
| `scripts/` | post-sync/prepare-sync 脚本 |
| `.github/workflows/` | Node/Python 双 CI（node.js.yml、python.yml）、sponsors、webhooks |

组织方式评述：典型"前端/后端各自独立工程"的双包结构（非 npm workspace/turbo 单仓多包），两个工程各有独立锁文件、CI 按路径触发。前端 `next/src` 内部又是 create-t3-app 标准分层（pages/server/components/services/stores），后端 `platform/reworkd_platform` 采用 FastAPI-template 分层（web/api、db、services、schemas）。

## 5. 核心架构与关键实现

### 5.1 浏览器端驱动的 Agent 主循环（最独特的架构决策）

主循环不在服务端，而在浏览器里。`next/src/services/agent/autonomous-agent.ts` 的 `AutonomousAgent` 类实现了一个工作队列状态机：workLog 队列依次执行 StartGoalWork → AnalyzeTaskWork → ExecuteTaskWork → CreateTaskWork，每个 Work 的 `next()` 决定下一棒，`addTasksIfWorklogEmpty()` 在队列空时把剩余任务继续灌入，形成"执行→反思→追加任务"的闭环；lifecycle 支持 running/pausing/paused/stopped，暂停后恢复会补执行 `lastConclusion`。每次失败带 2 秒退避重试（`withRetries`）。

每个 Work 通过 `next/src/services/agent/agent-api.ts` 调用 FastAPI 端点（`NEXT_PUBLIC_BACKEND_URL` + `/api/agent/start|analyze|execute|create|summarize`，见 `next/src/services/api-utils.ts`）。即：**浏览器是编排器，服务端只提供单步 LLM 调用**。这是 2023 年 agent 项目的典型形态——没有服务端长任务队列，刷新页面即暂停。

一棒接一棒的工作流转大致如下：

1. `StartGoalWork`：POST `/api/agent/start`，把目标变成最多 5 个初始搜索式任务（Plan-and-Solve prompt）。
2. `AnalyzeTaskWork`：POST `/api/agent/analyze`，模型以 function calling 从工具箱里选一件工具并给出 reasoning/arg。
3. `ExecuteTaskWork`：POST `/api/agent/execute`，实际调用所选工具（搜索/画图/写码），流式返回结果。
4. `CreateTaskWork`：POST `/api/agent/create`，把上一任务结果喂回模型反思，产出追加任务或空（终止信号）。
5. 队列空但有剩余任务时回到第 2 步循环；`MAX_LOOPS=100` 上限在 `.env.example` 中默认配置。

### 5.2 服务端单步执行引擎（FastAPI）

`platform/reworkd_platform/web/api/agent/views.py` 定义 7 个端点：`/start`（目标→初始任务）、`/analyze`（选工具）、`/execute`（执行并流式返回）、`/create`（反思追加任务）、`/summarize`、`/chat`、GET `/tools`。实现类为 `OpenAIAgentService`（`agent_service/open_ai_agent_service.py`），遵循 `agent_service.py` 的 Protocol 接口（另有 `mock_agent_service.py` 用于 FF_MOCK_MODE）。

关键流程——工具选择用 OpenAI function calling（`analyze_task_agent`）：把候选工具转成 functions 列表传给模型，解析返回的 `function_call`（action+reasoning+arg）为 `Analysis` 对象（`analysis.py`），pydantic 校验失败则回退默认 Search 工具。任务分解 prompt 采用 Plan-and-Solve 风格（`prompts.py` 头部注释引用 AGI-Edgerunners/Plan-and-Solve-Prompting），强制要求返回 JSON 数组。执行阶段全部走 lanarky StreamingResponse（text/event-stream）流式吐字。

### 5.3 模型接入层：OpenAI 单一依赖

`web/api/agent/model_factory.py` 的 `create_model()` 只支持两条路：OpenAI 官方 API 或 Azure OpenAI（`WrappedChatOpenAI`/`WrappedAzureChatOpenAI`），外加 Helicone 网关与用户自带 key（custom_api_key）分支。无本地模型、无多厂商抽象——2023 年中 Rewrite 后的取舍。多处硬编码旧模型名（`gpt-3.5-turbo`、`gpt-3.5-turbo-16k`，见 `open_ai_agent_service.py` summarize/chat 与 `next/src/server/api/routers/agentRouter.ts` 起名逻辑），也印证了停更时点。token 预算由 `services/tokenizer/token_service.py` 的 `calculate_max_tokens` 按模型上下文裁剪。

### 5.4 工具系统与外部集成

工具基类 `tools/tool.py`，注册表 `tools/tools.py`，全部工具一览：

| 工具 | 实现文件 | 依赖的外部服务 | 说明 |
|---|---|---|---|
| Search（默认） | `tools/search.py` | Serper.dev | Google 搜索，`CitedSnippet` 保留引用来源（`tools/utils.py`） |
| Image | `tools/image.py` | Replicate | 文生图 |
| Code | `tools/code.py` | 无（纯 LLM） | 写代码 |
| SID | `tools/sidsearch.py` | SID（@sid-hq/sid） | 浏览器代理/登录态搜索，需 OAuth 授权 |
| Reason / Conclude | `tools/reason.py`、`tools/conclude.py` | 无 | 纯推理与终止，无参数 |
| Wikipedia | `tools/wikipedia_search.py` | — | 已注释停用（"TODO: Remove if async doesn't work"） |

`get_user_tools` 结合 OAuth（`db/crud/oauth.py`）做动态可用性过滤（如 SID 需用户授权）。搜索结果经 `tools/utils.py` 的 summarize_with_sources 压缩后回灌模型。

## 6. 构建与运行

来自 README 与 docker-compose.yml：

```bash
# 一键安装（推荐，cli/ 提供）
git clone https://github.com/reworkd/AgentGPT.git && cd AgentGPT
./setup.sh        # Windows: ./setup.bat

# 手动：三容器
docker compose up   # frontend:3000  platform:8000  agentgpt_db:3307

# 仅前端开发
cd next && npm install && npm run dev
# 仅后端开发（poetry）
cd platform && poetry install && poetry run python reworkd_platform/__main__.py
```

前置条件：Node >= 18、Docker、OpenAI API key（必需）、Serper/Replicate key（可选）。环境变量模板 `.env.example`（前缀 `REWORKD_PLATFORM_`）。setup CLI 本身是独立 TypeScript 工程（`cli/package.json`），交互式引导填写 API key 并依次拉起数据库、后端、前端（README "Getting Started"）。

## 7. 测试与工程质量

- 测试规模小：后端 pytest 16 个文件（`platform/reworkd_platform/tests/`，覆盖 agent 分析/工具/模型工厂/输出解析等核心路径）；前端 jest 仅 3 个测试文件（`next/__tests__/`：message-service、whitespace、with-retries，配置见 `next/jest.config.cjs`），另有 `next/__mocks__/` 提供 OpenAI mock。
- CI 双流水线按路径触发：`node.js.yml`（npm test + prisma sqlite 切换）、`python.yml`（black/mypy/flake8 分 job）。
- 工程质量明显高于同期 agent 项目：后端 mypy strict、pre-commit、wemake-python-styleguide（`platform/pyproject.toml`）；前端 tRPC + Zod 全链路类型安全。但历史包袱明显：pydantic v1、openai SDK 0.28、langchain 0.0.295 均为已废弃的大版本，续命成本高。

## 8. 社区与维护状态

- **已归档**：gh api 实测 `archived: true`（GitHub 只读），README 本身无维护状态声明。
- 停更时间线：2023-04 创建即爆火；release 节奏密集（v0.8.0-beta 2023-06 → **v1.0.0 2023-11-02**，此后再无 release，gh api releases）；2023 年底至 2024 年零星维护。
- 最后的提交（gh api commits）：2025-03 创始人 asim-shrestha 改按钮/加教程/换 banner，2025-04-29 由 Himank Pathak 提交最后一次纯文档更新后归档。
- 贡献者约 72 人（gh api contributors 分页推算），核心代码由 Reworkd 团队（asim-shrestha、awtkns、Himank Pathak 等）把持，社区贡献以文档/i18n 为主。
- 多语言生态：README 提供简体中文、匈牙利语版本（`docs/README.zh-HANS.md` 等），文案高度依赖社区翻译。
- 热度存续：星数至今仍在增长（快照 39500 vs 归档时 36268），说明其"历史地标"地位带来的持续关注；归档时仍有 219 个 open issues 未处理（gh api），无人再回应。

## 9. 亮点与风险

**亮点**
1. 浏览器端 AutonomousAgent 状态机（`autonomous-agent.ts`）：把 agent 循环做成可暂停/恢复/带重试的前端工作队列，交互设计在 2023 年独树一帜，至今仍有参考价值。
2. 工程化程度高：双栈 monorepo、tRPC+Zod 全类型链、mypy strict、按路径触发的 CI、一键 setup CLI，是同期 agent 项目里少有的"产品级"工程（对比 AutoGPT 早期的脚本式代码）。
3. 工具选择用 OpenAI function calling + pydantic 校验 + 默认回退（`analysis.py`），比纯 prompt 解析健壮。
4. 可观测与成本意识：Helicone 网关接入、TokenService 按 token 预算裁剪上下文、Stripe 计费内建。

**风险/局限**
1. 已归档停更：依赖全部停留在 2023 年版本（pydantic v1、openai 0.28、langchain 0.0.295），跑在新环境需大量升级，无安全补丁。
2. LLM 通道单一：仅 OpenAI/Azure，无 Anthropic/本地模型支持；多处硬编码 gpt-3.5 系模型。
3. 架构局限：循环在浏览器侧，刷新/关页即中断，无服务端任务队列与断点续跑；Pinecone 长期记忆仅是壳（`services/pinecone/`），主流程未深度使用。
4. GPL-3.0 传染性许可对商用二次开发不友好。

## 10. 总结评价

AgentGPT 是 2023 年"浏览器自主 Agent"赛道的标志性项目：它用最直白的方式（网页输入目标→流式看 agent 干活）把 agent 概念推向了千万级大众，其浏览器端状态机 + 服务端单步执行的架构是那个时代"demo 优先"思潮的典型标本；也是"爆款如何停更"的教科书案例——母公司转向企业产品后开源仓库三年无实质更新直至归档。今天读它的价值在于历史与交互设计层面：想理解 MCP/函数调用标准化之前 agent 工具编排的原生形态、或复刻"可视化 agent 执行流"的产品设计，这两个代码库（前端状态机 `next/src/services/agent/`，后端单步引擎 `platform/reworkd_platform/web/api/agent/`）依然是最好的切片之一；想找可长期依赖的 agent 框架则应转向后继者（AutoGPT 本体、LangGraph、CrewAI 等）。
