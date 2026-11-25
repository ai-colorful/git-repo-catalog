# Tencent/WeKnora 项目分析

> 星数 37923（快照 2026-10-24）｜主语言 Go｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：腾讯（微信对话开放平台团队）开源的企业级 LLM 知识平台——把散落文档变成可检索的 RAG、能自主执行多步任务的推理 Agent、以及随资料更新自动维护的 Wiki，三能力共享同一知识库。

| 项 | 值 |
|---|---|
| 名称 | Tencent/WeKnora（维娜拉） |
| 星数 / Fork | 37923 / 5288（gh api 核实，2026-11-21） |
| 语言 | Go 后端（go.mod 声明 go 1.26）+ Vue 3 前端 + Python 解析服务 |
| 许可证 | MIT（LICENSE 文件，腾讯版权；第三方组件例外见 THIRD_PARTY_NOTICES.md） |
| 版本 / 最近 release | v0.9.0（2026-10-29，gh api + CHANGELOG.md） |
| 最近活跃 | pushed_at 2026-11-21（分析当日仍在推送，gh api 核实） |
| 创建时间 | 2025-07-22，16 个月到 37.9k 星（约 2.4k 星/月，超高速增长） |
| 官网 | https://weknora.weixin.qq.com |

## 2. 项目是什么，解决什么问题

WeKnora 面向企业文档理解、语义检索与智能推理场景，核心主张是"查询资料用 RAG、处理多步任务用 Agent、整理知识用 Wiki，三种能力共享同一知识库"（README_CN.md）。它解决的是团队知识的"存、查、用、养"四件事：文档汇入后自动解析切片向量化进知识库（存）；混合检索 + 重排 + 引用溯源回答问题（查）；Agent 携带知识和工具（shell、沙箱、浏览器、MCP）执行多步任务（用）；摄取管线同步把文档整理成结构化 Wiki 页并随文档更新而更新（养）。最后这一项是与普通 RAG 产品的核心差异。

数据源覆盖面是它的企业级卖点：飞书/Lark、Confluence、GitLab、Notion、语雀、钉钉、Seafile、RSS、腾讯 IMA 等十余种自动同步（internal/datasource/ 189 个 Go 文件），格式覆盖 PDF、Word、图片、Excel、XMind 等十余种（docreader/ 与 internal/infrastructure/docparser/anydoc/）。渠道侧支持企业微信、飞书、Slack、Telegram IM 内直接问答（internal/im/），并有网站嵌入 Widget、内置 MCP Server 供 Cursor/Claude 接入（internal/mcpserver/）、Chrome 插件与微信小程序（miniprogram/）。

Agent 部分的工程深度超出常见 RAG 项目：ReAct 循环带上下文压缩（compaction）、token 估算在线校准、运行中消息注入（steer）、人机协同审批（HITL/approval/hostreview）、挂起/恢复、后台任务、多 Agent 协作（internal/agent/collaboration/），技能可从 ClawHub/SkillHub/Git/ZIP 安装并在 Docker/E2B/Cube 沙箱执行（internal/agent/skills/、internal/sandbox/）。权限侧有四级 RBAC、空间审计日志、API Key 与 Principal 模型（README_CN.md；internal/application/access/）。

Wiki 机制：文档摄取时由 LLM 生成摘要页、抽取实体/概念页、规划目录树（taxonomy），页面类型分 Summary/Entity/Concept/Synthesis/Comparison 五类（internal/application/service/wiki_ingest.go、internal/agent/prompts_wiki.go）；Agent 自身持有 wiki_* 工具集可读取、搜索、改名、替换文本、挂 issue、修 issue（internal/agent/tools/wiki_*.go），形成"知识自动整理 + Agent 自维护"的闭环。

## 3. 技术栈与依赖

- 后端：Go 1.26，Web 框架 gin + swaggo（Swagger 注解，cmd/server/main.go）；DI 用自建 container（internal/container/，35 文件含 engine_factory.go）；ORM gorm（wiki_ingest.go import）。
- 任务队列：hibiken/asynq（Redis 后端），Lite 模式可退化为进程内执行（wiki_ingest.go 中 ErrWikiIngestConcurrent 与 liteLocks 逻辑）。
- 向量/检索后端 10 种（internal/application/repository/retriever/ 子目录）：postgres(pgvector)、elasticsearch、opensearch、milvus、qdrant、weaviate、腾讯向量数据库 tencentvectordb、sqlite(sqlite-vec)、neo4j（知识图谱）、doris。
- 模型接入：internal/models/ 分 chat/embedding/rerank/vlm/asr/providers/catalog，README 称内置 29 家厂商（OpenAI、DeepSeek、Qwen、智谱、混元、Gemini、MiniMax、NVIDIA、LiteLLM、Ollama 等）；包装层有 langfuse_wrapper.go、concurrency_wrapper.go（internal/models/chat/）。
- 文档解析：docreader/ 为独立 Python gRPC 服务（parser/ 下 30+ 解析器注册表：pdf/docx/excel/epub/html/xmind/image 等，OCR 走 MinerU、PaddleOCR-VL）；Go 进程内 anydoc 引擎（internal/infrastructure/docparser/anydoc/，含 MinerU 云/本地客户端）解析 Office 文档。
- 前端：Vue 3 + Vite + TDesign（腾讯设计体系）+ Pinia + vue-i18n + xterm（终端）+ noVNC（沙箱图形桌面）+ pdfjs + mermaid + katex + pagefind（frontend/package.json）。
- 桌面端：wails（go.mod 中 github.com/wailsapp/wails/v2，cmd/desktop/、frontend/src/wailsjs/）。
- 可观测：OpenTelemetry + Langfuse（internal/tracing/langfuse/）。
- 沙箱：Docker / E2B / 腾讯 Cube（go.mod 中 go-e2b、tencentcloud CubeSandbox SDK；internal/sandbox/ 151 文件、internal/localsandbox/）。

## 4. 目录结构与代码组织

| 目录 | 职责（依据源码） |
|---|---|
| cmd/server | gin HTTP 服务入口（main.go，JWT + API Key 双认证注解） |
| cmd/desktop | Wails 桌面客户端入口 |
| internal/agent | ReAct Agent 引擎与工具（417 文件，tools/ 246 个） |
| internal/application | service（786 文件，业务编排）+ repository（239 文件，含 retriever 10 后端） |
| internal/handler | HTTP handler（339 文件） |
| internal/datasource | 外部数据源连接与同步（189 文件，feishu/notion/gitlab 等） |
| internal/infrastructure | chunker（28 文件切片器）、docparser/anydoc、web_search 等 |
| internal/models | 模型接入层（254 文件：chat/embedding/rerank/vlm/asr） |
| internal/sandbox, localsandbox | 沙箱执行（Docker/E2B/Cube；本地沙箱） |
| internal/im | IM 渠道适配（dingtalk/feishu/slack 等，124 文件） |
| internal/mcpserver, mcp | 内置 MCP Server 与 MCP 客户端 |
| docreader | Python 文档解析 gRPC 服务（133 文件） |
| frontend | Vue 3 Web UI（1205 文件）+ e2e |
| cli | 独立 Go module 的 CLI（326 文件，含 MCP server 模式与验收测试） |
| mcp-server | 独立发布的 MCP Server 构建物（35 文件，有独立 CI） |
| migrations, helm, docker, deploy | 数据库迁移（PostgreSQL 000119-000134 / SQLite 000037-000051，CHANGELOG）、K8s、容器编排 |
| miniprogram, packages, website-docs | 微信小程序、共享包、官网文档 |

组织方式评述：单体 Go 后端 + 多语言配套服务（Python docreader、Vue 前端、独立 CLI module、独立 mcp-server）的 monorepo。分层清晰：handler → service → repository → infrastructure，模型与沙箱独立成层；规模约 3810 个 Go 文件 / 90 万行 Go 代码（含测试），前端另有 1205 文件。service 层按业务前缀拆成 613 个文件/目录（agent_*、wiki_*、knowledge_* 等），模块边界靠包名约定而非子目录划分，是重业务单体的典型形态。

## 5. 核心架构与关键实现

### 5.1 RAG 混合检索与融合（internal/application/service/）

检索采用向量 + 关键词 + 实体/图谱 + Web 多路并行（chat_pipeline/search_parallel.go），再经 RRF（Reciprocal Rank Fusion）加权融合归一到 [0,1] 分数段（knowledgebase_search_fusion.go）：

```go
// fuseOrDeduplicate — knowledgebase_search_fusion.go
//   - vector only: the engines' cosine similarity, unchanged (FAQ thresholds
//     depend on it);
//   - keyword only: BM25 divided by the best score of its own list;
//   - hybrid: weighted RRF divided by its maximum (a chunk ranked first by
//     both retrievers scores 1).
```

chat_pipeline/ 是一条显式的阶段管道（每阶段一个文件）：query_understand → query_expansion → merge_history → memory_recall → search_parallel → filter_top_k → rerank → into_chat_message → chat_completion_stream → references/citation_sources，另有 wiki_boost、web_fetch、data_analysis、extract_entity 等旁路阶段。向量后端通过 repository/retriever/factory.go 按 KB 配置选择，10 种后端实现同一接口（types/retriever.go）。

### 5.2 Agent 引擎（internal/agent/engine.go）

ReAct 循环 + 强工程化护栏。引擎跨轮次无状态，历史每轮由 service.LoadAgentHistory 从 DB 重建（engine.go 顶部注释）；上下文窗口管理是亮点：token 估算器带在线校准（calibration.go，按 provider 的实际 token/估算 token 比例学习），超限触发 Compactor 压缩旧历史并可持久化 checkpoint（checkpointSink），每轮只允许一次"压缩重试"（overflowRecovered 标志防止无限烧预算）。用户可在运行中注入消息（steerSink，每轮边界 drain，最多溢出一轮 maxSteerOverruns=1）。工具执行带失败连击熔断（loopGuards.recordFailedToolCall）、并行工具调用的 VLM 描述信号量（imageDescribeSlots）、以及精确一次的完成事件保证（emitCompletion + context.WithoutCancel）。

工具面（internal/agent/tools/，~85 个非测试文件）分五类：知识类（search_knowledge、query_knowledge_graph、list_documents、faq_snippet）、执行类（shell_exec/async、sandbox_write/edit/ls/diff、data_analysis、python_syntax 校验）、网络类（read_web_page、browserskill 经 chromedp 操作用户 Chrome）、协作类（ask_user、secure_form、host_approval、collaboration、sequentialthinking、todo_write）、集成类（mcp_tool/mcp_oauth、skill_file/skill_resources、memory）。技能管理器走"渐进式披露"（Progressive Disclosure，engine.go 字段注释）。

### 5.3 Wiki 自维护管线（internal/application/service/wiki_ingest*.go + agent/prompts_wiki.go）

文档摄取完成后以 asynq 任务驱动 Wiki 生成：WikiSummaryPrompt 生成摘要页（刻意不把文件名传给 LLM，防止扫描件文件名诱导幻觉——prompts_wiki.go 注释举例 "MX5280.pdf"）；WikiKnowledgeExtractPrompt 一次调用同时抽实体与概念（带 previous_slugs 防重复建页）；WikiTaxonomyPlanPrompt 单次调用为整批页面分配目录路径，要求优先字符级复用已有文件夹，且 reduce 阶段只对无分类的页面生效以免搅动用户编辑。并发控制：标准模式取消独占锁改用行级认领（claimPendingList）+ 每 slug reduce 锁（withSlugLock），Lite 模式进程内串行。生成的页面供 Agent 通过 wiki_read_page/wiki_search 消费，Agent 还能用 wiki_flag_issue/wiki_update_issue/wiki_replace_text/wiki_rename_page 对 Wiki 提出并执行修正——"自维护"即指这条摄取-生成-修正闭环。

### 5.4 文档摄取管线（docreader/ + internal/infrastructure/）

解析双轨：Python docreader gRPC 服务负责重格式与 OCR（parser/registry.py 注册 30+ 解析器，MinerU/PaddleOCR-VL 云端转换），Go 进程内 anydoc 引擎负责 Office 系（internal/infrastructure/docparser/anydoc/，含 SSRF 防护测试 mineru_converter_ssrf_test.go、资源限制 resource_limit_test.go）。切片在 Go 侧 internal/infrastructure/chunker/：标题层级切分（heading_hierarchy.go）+ 启发式兜底（heuristic_splitter.go）+ token 预算（budget.go/tokens.go）+ 围栏代码保护（fenced_code.go）+ 校验器（validator.go），并支持父子块偏移（parent_child_offsets）。任务经 asynq 入队，嵌入生成有独立通道（service/embed_channel.go、embed_session.go）。

## 6. 构建与运行

```bash
# Docker Compose 部署（README_CN.md）
git clone https://github.com/Tencent/WeKnora.git && cd WeKnora
cp .env.example .env            # 或 .env.lite.example（Lite 版）
docker compose pull && docker compose up -d
# 可选 profile：full / neo4j / minio / langfuse
docker compose --profile neo4j --profile minio up -d
```

服务地址：Web UI http://localhost、后端 API :8080、Langfuse :3000。本地开发用 Makefile（顶层，含 Go 构建/检查目标）；前端 `pnpm dev` / `pnpm build` / `vue-tsc`（frontend/package.json）；CLI 有独立 Makefile 与 `weknora doctor` 自检（cli/README.md）；macOS Lite 版有 Homebrew Formula（Formula/weknora-lite.rb）。数据库迁移用 golang-migrate（go.mod）。

## 7. 测试与工程质量

- Go 测试 2113 个 _test.go 文件、约 41.5 万行（占 Go 代码 46%），框架 testify + sqlmock + miniredis（go.mod）；测试粒度极细，agent 引擎的行为边界（orphan_think、stream_retry_partial、tool_goroutine_panic、failed_tool_loop 等）都有对名测试（internal/agent/）。
- 前端 285 个 *.test.ts + Playwright e2e（frontend/e2e/novnc.spec.ts，验证 noVNC 沙箱桌面）。
- CI 13 条工作流（.github/workflows/）：app、frontend、cli、cli-e2e、docreader、docker-image、helm、mcp-server、go-lint（含缓存优化 go-lint-cache）、release-lite、anydoc、dsh-plugin——每个子构建物有独立流水线；dependabot.yml 管依赖。
- 质量观察：注释密度高且多为"为什么"型注释（engine.go、wiki_ingest.go 大段设计说明）；安全意识强（SSRF 测试、输出预算 output_budget.go、MCP call guard、channel_security.go）；CHANGELOG 极为详尽，含精确迁移号与升级注意事项。golangci 配置见 .golangci.yml。

## 8. 社区与维护状态

- 活跃度：16 个月 37.9k 星、5288 fork、1042 open issues（gh api，2026-11-21）；pushed_at 即分析当日，持续高频开发。
- Release 节奏：CHANGELOG.md 显示 2026-05 至 2026-10 共 12 个版本（0.5.2→0.9.0），约每 2-3 周一版；最新 v0.9.0 于 2026-10-29 发布。
- 贡献者：contributors 分页前 100 已满（gh api），总数 100+，精确数未确认；主力应为腾讯微信团队（Docker 镜像命名 wechatopenai/weknora-docreader，README 关联微信对话开放平台）。
- 生态：官方 Chrome 插件、ClawHub Skill、npm 包 @wxg-prc-cpg/dsh-weknora、微信小程序、四语 README（中/英/日/韩，Trendshift 徽章）。

## 9. 亮点与风险

亮点：
1. RAG/Agent/Wiki 三合一共享知识库，Wiki 自动摄取管线（taxonomy 规划 + 实体/概念页 + Agent 修正闭环）是同类项目罕见的完整实现（internal/application/service/wiki_ingest*.go、internal/agent/tools/wiki_*.go）。
2. Agent 工程深度高：上下文压缩 + token 校准 + steer/HITL/挂起恢复 + 多 Agent 协作 + 三种沙箱，护栏细节（失败熔断、单次压缩重试）体现大量真实迭代（internal/agent/engine.go）。
3. 检索后端 10 种可插拔 + RRF 融合 + 重排，部署面从 sqlite-vec 单机到 Milvus/腾讯向量库集群全覆盖（internal/application/repository/retriever/）。
4. 测试与 CI 覆盖极厚（2113 个 Go 测试文件、13 条流水线），企业级权限/审计/迁移文档齐全。
5. 数据源与渠道生态广（10+ 数据源、4+ IM、MCP 双向：内置 server + 可挂载外部服务）。

风险/局限：
1. 体量大（Go 90 万行 + Python + 前端），自部署运维复杂度高：依赖 PostgreSQL/MySQL、Redis、可选 10 余项服务，升级需按迁移号逐档操作（CHANGELOG 0.9.0 Upgrade Notes）。
2. 深度依赖 LLM 质量与外部 OCR/解析云服务（MinerU、PaddleOCR-VL、腾讯云），弱模型下 Wiki 与 Agent 效果会显著退化（prompts_wiki.go 对空内容有防幻觉规则，但成本仍在）。
3. 开源版与腾讯云/Lite 版能力边界需自行甄别（release-lite、.env.lite.example 表明存在功能分档；License 为 MIT 但附第三方例外清单）。
4. service 层 613 文件的单包组织随增长可能加剧耦合，新人定位代码依赖命名约定（internal/application/service/）。

## 10. 总结评价

WeKnora 是目前开源 LLM 知识平台里工程完成度最高的项目之一：它不是"RAG demo 加壳"，而是把文档摄取、混合检索、ReAct Agent、沙箱执行、Wiki 自维护、IM 分发、权限审计做成了一条完整的生产链路，90 万行 Go 代码配 46% 的测试代码、13 条 CI 流水线和双周一版的节奏，体现了腾讯微信团队的工业化投入；16 个月 37.9k 星的增长在同类中属于头部水平。适合需要私有化部署企业知识库、又想要 Agent 执行能力而非仅问答的团队直接采用；对做 RAG/Agent 平台的工程师，agent 引擎的上下文管理（calibration/compaction/steer）与 wiki_ingest 的并发设计是最值得精读的两块源码。
