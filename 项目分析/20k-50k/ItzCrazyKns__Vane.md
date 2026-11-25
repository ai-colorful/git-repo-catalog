# ItzCrazyKns/Vane 项目分析

> 星数 38201（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

Vane 是一个**自托管的 AI 应答引擎（answering engine）**——即 Perplexity 式"搜索 + 综合 + 带引用回答"的开源替代品。它由个人开发者 ItzCrazyKns 创建，前身即知名项目 Perplexica（改名证据：仓库 topics 仍保留 `perplexica`、`searxng-copilot` 标签，README 中 Sealos/ClawCloud 一键部署模板仍写 `templateName=perplexica`，见 README.md:220-223）。

| 项 | 值 |
| --- | --- |
| 名称 | ItzCrazyKns/Vane（v1.12.2，package.json） |
| 星数 | 38,201（gh api，2026-11-14 核实） |
| 主语言 | TypeScript（gh api 核实） |
| 许可证 | MIT（LICENSE，Copyright (c) 2026 ItzCrazyKns） |
| 创建时间 | 2024-04-09（gh api） |
| 最近提交 | 2026-09-01 "Update README.md"（gh api commits） |
| 最新 Release | v1.12.2（2026-04-10，gh api releases） |
| 代码规模 | src/ 下 160 个文件、约 17,400 行 TS/TSX（实测统计） |

## 2. 项目是什么，解决什么问题

用户向搜索引擎提问时，传统引擎返回链接列表，需要自行点开、筛选、拼凑答案；ChatGPT/Perplexity 将这一过程自动化为"直接给出带引用的综合答案"，但都是闭源云服务，查询内容会离开用户设备。Vane 解决的是：**在本机/私有服务器上运行一个完整的 AI 应答引擎**——搜索通过自带的 SearxNG 元搜索完成（匿名聚合多个搜索引擎），LLM 可接本地 Ollama，全部数据（聊天记录、上传文件）落在本地 SQLite，从而做到"搜索隐私 + 答案质量"兼得（README.md:11 声明 "privacy-focused ... runs entirely on your own hardware"）。

产品形态是 Next.js 全栈 Web 应用：首页即搜索框，支持浏览器地址栏接入（`/?q=%s`，README.md:199-206）。回答时展示研究过程（搜索了哪些词、读了哪些网页）、来源卡片、内联 `[数字]` 引用，以及按需弹出的天气/股票/计算等小组件。三类检索源可选：web（网页）、academic（学术论文）、discussions（Reddit 讨论）；另支持图片/视频垂直搜索、文档上传问答（PDF/DOCX/TXT，基于本地 embedding 的语义检索）与 Discover 新闻浏览（src/app/api/discover/route.ts 硬编码了 tech/finance/art/sports/entertainment 五个主题的站点+关键词组合，用 SearxNG `site:` + `bing news` 引擎拉取）。

与竞品定位对比：相对 **Perplexity/ChatGPT search**，Vane 是可自托管、模型与搜索后端完全可控的开源对应物，但检索质量依赖 SearxNG 免费元搜索而非自建索引，无个性化记忆/账号体系。相对 **LibreChat** 这类开源聊天前端，Vane 的核心差异在"应答引擎"专用管线（分类→多轮检索→综合写作），而不是通用聊天 UI 加插件；相对 SearxNG 本体，Vane 在其上加了 LLM 综合与引用层。同类中 Morphic（morphic.sh）思路相近但更偏实验性，Vane 是该赛道星数最高的项目之一。

## 3. 技术栈与依赖

| 层 | 技术 | 出处 |
| --- | --- | --- |
| 全栈框架 | Next.js 16（App Router，`next build --webpack`）+ React 18 | package.json |
| 语言 | TypeScript 5.9，路径别名 `@/` | tsconfig.json |
| UI | Tailwind CSS 3 + headlessui/radix、lucide/phosphor 图标、motion 动效、react-syntax-highlighter、react-text-to-speech | package.json |
| 搜索后端 | SearxNG（Docker 镜像内从源码构建捆绑；slim 版外接），REST JSON 接口 | Dockerfile、src/lib/searxng.ts |
| 网页抓取 | Playwright（chromium-headless-shell）+ jsdom + @mozilla/readability 正文抽取 | src/lib/scraper.ts |
| 数据库 | better-sqlite3 + drizzle-orm（2 张表：chats/messages），启动时自动迁移 | src/lib/db/schema.ts、CONTRIBUTING.md |
| LLM 接入 | ollama、openai、@google/genai（Gemini）、Anthropic、Groq、LM Studio、Lemonade（AMD 本地 NPU 推理）SDK；自写薄抽象层 | src/lib/models/providers/ |
| Embedding | 各 provider 的 embedding + @huggingface/transformers 本地模型 | src/lib/models/providers/transformers/ |
| 结构化输出 | zod（schema 校验）、partial-json、@toolsycc/json-repair | package.json |
| 其他关键库 | rfc6902（JSON Patch 增量推送）、async-mutex、js-tiktoken、mathjs/yahoo-finance2（widget）、pdf-parse/mammoth/officeparser（文档解析） | package.json |

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `src/app/` | Next.js App Router：页面（`/`、`/c/[chatId]`、`/discover`、`/library`）+ 全部 API route（chat/search/images/videos/discover/suggestions/providers/config/uploads 等 16 个） |
| `src/components/` | React 组件：ChatWindow、MessageRenderer（含 Citation）、Settings、Setup 向导、Widgets（Weather/Stock/Calculation）等 |
| `src/lib/agents/` | 核心：search（classifier/researcher/widgets）、suggestions、media（图片/视频） |
| `src/lib/models/` | LLM/Embedding 抽象（base/）与 8 家 provider 实现、registry 加载器 |
| `src/lib/prompts/` | 分类器、researcher、writer、suggestions 等 prompt 模板 |
| `src/lib/db/` | drizzle schema 与迁移 |
| `src/lib/config/` | 配置管理（data/config.json 持久化 + 环境变量引导） |
| `src/lib/uploads/`、`src/lib/scraper.ts`、`src/lib/searxng.ts`、`src/lib/session.ts` | 文件上传语义检索、网页抓取单例、SearxNG 客户端、流式会话管理 |
| `drizzle/`、`searxng/`、`data/` | SQL 迁移产物、SearxNG 配置（settings.yml 开启 json 格式）、运行时数据目录 |
| `.github/workflows/` | 仅 docker-build.yaml（多架构镜像构建发布） |

主要 API 路由（src/app/api/）：`POST /chat`（主应答流）、`POST /search`（程序化调用）、`GET /providers` 与 `/providers/[id]/models`（模型清单）、`GET|POST /chats`（历史管理）、`GET /discover`、`POST /images`、`POST /videos`、`GET /suggestions`、`POST /uploads`、`GET /weather`、`POST|GET /reconnect/[id]`（断线重连）、`GET /config`（含 setup-complete 状态）。

组织方式评述：单仓库单体应用，无 monorepo/子包；前后端同仓同构（Next.js 全栈），分层清晰（app=接口层，lib=业务层），后端逻辑与 UI 框架解耦良好，agents/models/providers 均有注册表式扩展点（CONTRIBUTING.md:29-48 与实际结构一致）。

## 5. 核心架构与关键实现

### 5.1 三段式应答管线（分类 → 并行研究/组件 → 写作）

入口 `POST /api/chat`（src/app/api/chat/route.ts）校验请求后创建 SessionManager，立即返回流式响应，后端异步跑 SearchAgent（src/lib/agents/search/index.ts）。第一步用 LLM 结构化输出做**分类与查询改写**（src/lib/agents/search/classifier.ts）：一次调用同时产出 `skipSearch`（寒暄类直接跳过搜索）、academic/discussion 源选择、widget 触发标志，以及把追问改写成独立问题 `standaloneFollowUp`。随后 `Promise.all` 并行执行 widget 计算与 Researcher 检索（index.ts:62-96），最后把检索结果拼成带 `<search_results>`/`<widgets_result>` 注记的上下文交给 writer prompt（src/lib/prompts/search/writer.ts）流式生成答案；writer 强制"每句话至少一个 `[number]` 引用"，quality 模式额外注入"不少于 2000 词的研究报告"指令（writer.ts:36）。

### 5.2 Researcher 工具调用 Agent 循环

src/lib/agents/search/researcher/index.ts 实现了一个基于 LLM function-calling 的多轮研究循环：迭代上限由模式决定——**speed=2、balanced=6、quality=25**（index.ts:15-20）。注册的工具（actions/index.ts）有 `web_search`、`academic_search`（固定 arXiv/Google Scholar/PubMed 三引擎，academicSearch.ts:51）、`social_search`（Reddit，socialSearch.ts:51）、`scrapeURL`、`uploads_search`、`plan`、`done`，外加一个 `__reasoning_preamble` 伪工具把模型的"计划"文本实时推给前端展示。每轮把工具结果以 `role: 'tool'` 消息回灌，直到模型调用 done 或达上限；结束后按 URL 去重合并来源并 emit source 块（index.ts:185-214）。工具描述按模式区分：speed 提示"只能调一次、一次性出齐 3 个查询"，quality 提示"至少 5-6 轮迭代"（webSearch.ts）。

### 5.3 检索结果处理：embedding 重排 vs LLM 摘取

src/lib/agents/search/researcher/actions/search/baseSearch.ts 按模式分两条路径：
- **speed/balanced**：SearxNG 返回的每条结果用 embedding 算与查询的相似度，过滤 `<0.5`，再以 embedding 相似度 `>0.75` 去重，按相似度排序取前 20（baseSearch.ts:37-171）。embedding 失败时降级为全保留（baseSearch.ts:73-86）。
- **quality**：不靠 embedding，而是先用 LLM 当"结果挑选器"从全部结果中选出最多 3 个 URL（pickerPrompt，baseSearch.ts:240-286），再逐个用 Playwright 抓全文、按 4000 字符/500 重叠切块、每块再调 LLM 做"事实抽取"（要求保留原始数值、电报式要点，baseSearch.ts:321-418）。

网页抓取器 src/lib/scraper.ts 是全局单例：懒启动 chromium-headless-shell，伪造 UA 并抹掉 `navigator.webdriver`，domcontentloaded + 5 秒容错等待，Readability 抽正文，空闲 30 秒自动关浏览器省资源（scraper.ts:13-47）。

### 5.4 块式流式协议与断线重连

流式层不是逐 token 文本，而是**结构化 Block 流**：SessionManager（src/lib/session.ts）维护 text/research/source/widget/reasoning 等类型的有 ID 块，更新时用 rfc6902 JSON Patch 生成增量补丁；订阅者除实时接收外，还能回放事件历史（subscribe 先补发历史再挂监听，session.ts:80-102），配合 30 分钟 TTL 与 `/api/reconnect/[id]` 路由实现**刷新页面/断线后恢复现场**。chat route 用 TransformStream 把事件序列化为 NDJSON（每行一个 JSON 对象，Content-Type 仍是 `text/event-stream`，route.ts:155-246）；前端 useChat.tsx 用 `res.body.getReader()` 逐行解析并 applyPatch。回答完成后所有块持久化进 SQLite messages 表的 `responseBlocks` JSON 列（src/lib/db/schema.ts:6-19）。

### 5.5 Widgets、媒体搜索与文件问答等周边能力

- **Widgets**：weather/stock/calculation 三个结构化小组件（src/lib/agents/search/widgets/），由分类器触发标志决定是否执行，计算走 mathjs、股价走 yahoo-finance2；结果以 `widget` 块先于正文渲染给用户，并在 writer 上下文中明确标注"已展示给用户、不得作为引用来源"（search/index.ts:114-120）。
- **图片/视频搜索**：独立端点 `POST /api/images`、`POST /api/videos`，先用 chat 模型生成聚焦查询再经 SearxNG 拉取（WORKING.md:70-72），代理实现在 src/lib/agents/media/image.ts、video.ts；前端支持"自动媒体搜索"开关（config/index.ts 的 autoMediaSearch）。
- **文件上传问答**：支持 PDF（pdf-parse）、DOCX（officeparser/mammoth）、TXT（src/lib/uploads/manager.ts 的 supportedMimeTypes），切块后存本地，查询时用所选 embedding 模型做相似度检索（uploadsSearch 动作接入 Researcher 循环）。
- **搜索建议**：`/api/suggestions` 基于历史与当前输入用 LLM 生成后续问题（src/lib/agents/suggestions/）。
- **Discover**：硬编码 5 大主题 × 固定媒体站点列表（techcrunch/wired/bloomberg/espn 等），用 `site:` 限定 + `bing news` 引擎聚合当日新闻（src/app/api/discover/route.ts:1-24）。

### 5.6 模型抽象与配置系统

src/lib/models/base/llm.ts 定义 `generateText/streamText/generateObject/streamObject` 四方法的抽象基类，8 家 provider 各自实现（如 anthropicLLM.ts），registry.ts 按 config 中的 providerId+key 动态加载。配置全部落在 `data/config.json`（路径可由 DATA_DIR 环境变量改，src/lib/config/index.ts:8-11），首次启动自动把环境变量（如 SEARXNG_API_URL）注册成 provider/搜索配置并去重（initializeFromEnv），UI 里有 Setup 向导与 Settings 界面直接改写该文件——这使项目**零 .env 门槛**，装完 Docker 即可在页面里配 key。

## 6. 构建与运行

```bash
# Docker（推荐，镜像内置 SearxNG）
docker run -d -p 3000:3000 -v vane-data:/home/vane/data --name vane itzcrazykns1337/vane:latest

# 自带 SearxNG 时用 slim 版外接
docker run -d -p 3000:3000 -e SEARXNG_API_URL=http://your-searxng:8080 ... itzcrazykns1337/vane:slim-latest

# 源码运行
npm i && npm run build && npm run start   # 或 npm run dev
# 辅助命令：npm run lint；npm run format:write（Prettier）
```

要求 SearxNG 开启 JSON 输出格式与 Wolfram Alpha 引擎（README.md:86-91、119）；首次访问 http://localhost:3000 走 Setup 向导配模型。对外集成用 `GET /api/providers` + `POST /api/search`（支持 `stream: true`），文档在 docs/API/SEARCH.md。构建细节：yarn.lock 锁定依赖，Dockerfile 两阶段构建（node:24.5.0-slim，运行阶段从源码 git clone searxng/searxng 编译安装 + Playwright chromium），entrypoint.sh 先起 SearxNG（flask，端口 8080）等健康检查再 `node server.js`。

## 7. 测试与工程质量

- **测试**：仓库内无任何测试文件（无 jest/vitest 配置、无 *.test.*/*.spec.*），CONTRIBUTING.md 也只要求"自行充分测试"——质量完全靠手工验证与用户反馈，这是与同等星数项目的明显差距。
- **CI**：仅 docker-build.yaml 一个工作流：master/canary 推送与 release 触发，full/slim 两种变体 × amd64/arm64 双架构构建推送 DockerHub 并生成 manifest；**没有 lint/test/类型检查 CI**。
- **代码质量观察**：TypeScript 全量覆盖、zod 做边界校验（API 入参与 LLM 输出双重校验）、ESLint+Prettier 配置齐全；代码风格统一、模块边界清晰。短板：部分文件过长（baseSearch.ts 420+ 行且 speed 与 quality 两分支大量重复代码）、错误处理多为 console.log 吞掉、config 的 migrateConfig 留空 TODO（config/index.ts:170-173）。

## 8. 社区与维护状态

- **贡献者结构**：48 名贡献者，但高度单核——创始人 ItzCrazyKns 895 次提交，第二名 WanQuanXie 仅 33 次（gh api contributors，2026-11-14），是典型的单维护者项目，社区贡献以小修为主。有 Discord 社区，获 Exa 赞助（README.md:43-60），接受以太坊捐赠。
- **活跃度**：创建于 2024-04（Perplexica 时代），2026-09-01 最后一次推送（仅 README 更新）；最后一次实质性代码提交约在 2026-04-11（"feat(docker): update slim image"，gh api commits）。**2026-04 后开发明显放缓，2026-09 后完全停滞**（截至分析日约 2.5 个月无推送）。
- **Release 节奏**：v1.11.x（2025-10）→ v1.12.0/v1.12.1（2025-12）→ v1.12.2（2026-04-10），约数月一个 minor 版本；最新代码（HEAD）比最新 release 新但差异不大。
- **Issue 积压**：379 个 open issue（gh api），单维护者消化能力有限。

## 9. 亮点与风险

**亮点**
1. 应答引擎管线完整且透明：分类→查询改写→多轮工具调用检索→引用写作，且把研究过程以结构化块实时可视化（含 reasoning/searching/reading 子步骤），源码是学习 Perplexity 式产品架构的最佳教材之一（src/lib/agents/search/）。
2. 隐私自托管闭环：SearxNG 捆绑 + Ollama/本地 embedding + SQLite 本地存储，Docker 一条命令起全套，无需任何 .env 配置即可走完 Setup（Dockerfile、config/index.ts）。
3. 流式协议设计精细：JSON Patch 增量更新 + 事件回放 + 断线重连 + 块持久化，比裸 SSE 逐 token 推送更适合"过程可见"的应答引擎（session.ts、api/chat/route.ts）。
4. 模型接入广且抽象薄：8 家 provider（含 Ollama/LM Studio/Lemonade 本地路线）共享一个小接口，新增 provider 成本低（src/lib/models/providers/）。
5. 三档 speed/balanced/quality 贯穿 agent 迭代上限、检索策略、writer 指令三层，同一管线给出清晰的时延/质量旋钮。

**风险**
1. **维护停滞**：2026-09-01 后无推送，最后实质开发在 2026-04；379 个 open issue 无人处理，若停滞持续将成为"活跃度高但弃养"的项目。
2. **零测试、零质量门禁 CI**：全靠作者手工验证，fork 或深度定制后回归风险高。
3. 检索质量天花板受制于 SearxNG 免费元搜索（无自有索引、易被目标站反爬），quality 模式"LLM 挑 3 个 URL + 逐块抽取"成本高且单次覆盖面窄；README 承诺的 Tavily/Exa 集成尚未落地。
4. 无认证/多用户体系（README Upcoming Features 已承认），会话与配置存于单文件 SQLite/JSON，不适合直接公网多人共享部署。

## 10. 总结评价

Vane（原 Perplexica）是"开源 Perplexity"赛道事实上的头部项目：以约 1.7 万行 TypeScript 的单体 Next.js 应用，实现了从查询分类、多轮 agentic 检索、并行 widget 到引用写作的完整应答引擎管线，流式协议与隐私自托管设计都相当用心，代码可读性好、扩展点清晰。它的问题同样典型——单维护者、零测试、2026 年中后开发放缓，长期可用性存在不确定性。适合三类人：想自建隐私搜索问答服务的 homelab 用户（Docker 5 分钟上手）；研究"AI 应答引擎该怎么做"的工程师（agents/search 目录值得精读）；以及想在其上改造检索源/换掉 SearxNG、接入自有知识库的开发者。若作为生产服务长期依赖，需自行评估社区停滞风险并做好 fork 自维护的准备。
