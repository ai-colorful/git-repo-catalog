# THU-MAIC/OpenMAIC 项目分析

> 星数 43462（快照 2026-10-24，gh api 复核 2026-11-14）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：清华 MAIC 团队开源的"多 Agent 互动课堂"平台——输入一个主题或文档，多 Agent 编排自动生成含幻灯片、测验、交互模拟和 PBL 的完整课程，由会说话、会画白板的 AI 讲师和 AI 同学实时授课。

| 项目 | 值 |
| --- | --- |
| 名称 | THU-MAIC/OpenMAIC（Open Multi-Agent Interactive Classroom） |
| 星数 | 43,462（8 个月，gh api `repos/THU-MAIC/OpenMAIC`） |
| 主语言 | TypeScript（`package.json` 全栈 TS，engine 要求 Node >= 22.19） |
| 许可证 | MIT（`LICENSE`；注意 `packages/mathml2omml` 为 LGPL-3.0，`packages/pptxgenjs` 为第三方 MIT，见 README License 节） |
| 版本 | v1.5.0-rc.2（2026-11-12，`package.json` version 1.5.0-rc.2） |
| 最近活跃 | 最近 push 2026-11-13，最新提交 `fix(pbl): batch review evidence reads...`（gh api commits） |
| 贡献者 | 126 人（gh api contributors --paginate），核心 3 人贡献绝大多数提交（wyuc 404 / cosarah 72 / YizukiAme 70 次） |
| 学术背书 | JCST'26 论文《From MOOC to MAIC: Reimagine Online Teaching and Learning through LLM-driven Agents》（README Citation 节） |

## 2. 项目是什么，解决什么问题

OpenMAIC 把"听课"变成"上课"：用户给一个主题或上传课件/PDF，系统自动完成大纲规划、逐场景内容生成（幻灯片/测验/交互 HTML/PBL 项目）、配图与 TTS 语音合成，最终进入一个可实时互动的课堂页面——AI 讲师讲解、AI 同学（助教、好奇宝宝、显眼包等角色）插话讨论、白板上逐步画公式和流程图，学习者随时举手提问（README Overview/Features 节；`packages/@openmaic/generation/src/outline-generator.ts`、`scene-generator.ts`）。

与静态课程生成器（如 Gamma 类 PPT 工具）的本质区别在于"多 Agent 课堂交互"：课程播放期间由一个服务端 Director 模型驱动多个不同人设的 Child Agent 轮流发言，发言内容转成语音、白板动作和幻灯片聚光灯等课堂动作实时执行（`lib/chat/pi/director-loop.ts`、`lib/action/engine.ts`）。这正是其论文提出的从 MOOC（人讲给万人听）到 MAIC（Agent 课堂）的转变。

项目同时是可自部署的产品：课程、生成任务、Agent 会话全部存 PostgreSQL，生成过程"server-first"——关掉浏览器课程继续生成（v1.2.0，README News）；支持 PPTX/离线 HTML/课堂 ZIP 导出（`lib/export/`、`packages/@openmaic/importer/` 反向导入 PPTX）；通过 `skills/openmaic/SKILL.md` 接入 OpenClaw、Codex、DeepSeek 等 Agent 工作台，从飞书/Slack/Telegram 发消息即可生成课堂。

Deep Interactive Mode 还提供 3D（Three.js）、仿真、编程沙箱（E2B + xterm + Monaco）等五类交互场景（README Deep Interactive Mode 节；`package.json` 依赖 three、@xterm/xterm、monaco-editor、e2b）。

## 3. 技术栈与依赖

- 前端：Next.js 16（App Router）+ React 19 + TypeScript 5 + Tailwind CSS 4，UI 组件基于 shadcn/Radix（`package.json`），富文本编辑用 ProseMirror，状态管理 Zustand + immer
- LLM 层：Vercel AI SDK（`ai` + `@ai-sdk/openai|anthropic|google|azure|amazon-bedrock`）做多提供商接入；Agent 编排框架为 `@earendil-works/pi-agent-core` / `pi-ai`（`lib/chat/pi/`）；另集成 CopilotKit、MCP SDK（`@modelcontextprotocol/sdk`）
- 模型提供商约 20 家，国产覆盖尤其全：DeepSeek、Qwen、Kimi、MiniMax、GLM 智谱、Doubao、腾讯混元、小米 MiMo、SiliconFlow、百度、TokenDance，另有 Ollama/Lemonade 本地推理（`.env.example` 的 `*_API_KEY` 清单、README Providers 节）
- 媒体能力：TTS（Azure/Doubao/ElevenLabs/VoxCPM2 自托管）、ASR（FunASR 本地/Azure/OpenAI/Qwen）、图片视频生成、MinerU 文档解析（`.env.example`）
- 持久化：PostgreSQL 16（`pg`），`@openmaic/storage` 抽象出 document/runtime/KV/asset/agent-session 可插拔 store，资产可存 PG 或 S3（`packages/@openmaic/storage/`）
- 导出链路：定制版 pptxgenjs + mathml2omml（MathML 转 Office 公式）；MP4 视频导出走独立的 render-service 容器（Chromium + FFmpeg，`render-service/`）
- 工程化：pnpm workspace（9 个包）+ Vitest + Playwright e2e + ESLint/Prettier + Docker Compose

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `app/` | Next.js App Router：`api/agent|stages|generate-classroom|chat|pbl|persistence|export-video` 等 API 路由，`classroom/[id]/` 播放页，`workbench/` v1.5 新首页 |
| `lib/` | 核心业务逻辑（约 50 个子目录）：`chat/pi/` 多 Agent 编排、`server/agent-runtime/` 持久 Agent 运行时、`playback/` 播放状态机、`action/` 动作执行、`whiteboard/` 白板运行时、`export/`、`persistence/`、`pbl/`、`rag/`、`sandbox/` 等 |
| `components/` | React UI：`slide-renderer/` 画布式幻灯片编辑器、`scene-renderers/` 测验/交互/PBL 场景、`workbench/`、`chat/`、`whiteboard/`、`agent/` |
| `packages/@openmaic/` | workspace 包：`dsl`（课程数据契约与校验）、`renderer`、`editor`、`generation`（两段式生成管线）、`storage`、`importer`（PPTX 导入）、`geometry` |
| `packages/` 其余 | 定制 `pptxgenjs`、`mathml2omml` |
| `render-service/` | 独立 MP4 渲染服务（Chromium + FFmpeg，自带 package.json/vitest） |
| `skills/` | OpenClaw/ClawHub 技能包：`openmaic/SKILL.md` 引导式 SOP + `agent-runtime/`、`classroom/` 课堂技能 |
| `tests/` | 单元测试主体（1,256 个测试文件）；`e2e/` Playwright；`eval/` LLM 效果评测（PBL planner、白板布局、编排答问） |
| `.github/` | 8 个 workflow（ci、publish-packages、storage-pg-contract、issue-triage 等）+ 自研 issue-triage agent 脚本 |

组织方式评述：单体 Next.js 应用 + pnpm monorepo 混合。把可复用的数据契约（dsl）、渲染（renderer）、生成（generation）拆成 workspace 包并发 npm（`publish-packages.yml`），而 UI 和 API 留在应用内；lib 下按能力域（50 个子目录）切分但共享一个 tsconfig。规模可观：全仓 3,709 个 TS/TSX 文件、约 7.8 万行（不含 node_modules，wc 统计）。

模型配置面也值得单独一提：服务端用 YAML 声明"槽位"——每个 AI 用途（默认对话、大纲、逐场景内容、TTS、图片、视频、联网搜索）各绑一个提供商:模型组合，key 经 `${VAR}` 引用环境变量，可按槽位 `lock` 或禁止用户自带 key（README Configure 节、`openmaic.example.yml`）：

```yaml
providers:
  openai:
    preset: openai
    apiKey: ${OPENAI_API_KEY}
  anthropic:
    preset: anthropic
    apiKey: ${ANTHROPIC_API_KEY}
slots:
  llm: openai:gpt-5.5            # 默认对话模型
  course.outline: anthropic:claude-sonnet-4-6
  video: null                    # 关闭某个能力
```

## 5. 核心架构与关键实现

### 5.1 两段式课程生成管线（`packages/@openmaic/generation/src/`）

Stage 1 由 `outline-generator.ts` 根据用户主题/PDF 文本与图片生成场景大纲（SceneOutline[]），带 PDF 图片视觉排序、图片占位符、大纲兜底逻辑；Stage 2 由 `scene-generator.ts` 逐场景生成完整内容——幻灯片元素（经 `@openmaic/dsl` 的 normalizeElement 校验、KaTeX 公式安全选项）、测验题、交互 HTML（`interactive-post-processor.ts` 后处理 + `interactive-script-validator.ts` 语法校验）以及讲解动作序列（`action-parser.ts` 从结构化输出解析动作）。`json-repair.ts` 负责修复 LLM 输出的破损 JSON。异步作业入口在 `app/api/generate-classroom/`，提交后轮询，生成在长驻服务进程中继续。

### 5.2 多 Agent 课堂编排：Director + call_agent（`lib/chat/pi/`）

课堂实时讨论的核心是一个 Director 循环：`director-loop.ts` 的 `runPiDirectorLoop` 把当前场景证据、白板状态注入上下文，Director 模型通过工具决定"谁来发言、做什么"——`tools/call-agent.ts`（参数仅 `agentId` + `instruction`，按人设生成对应 Agent 的发言与动作）、`cue-user.ts`（点名用户）、`native-whiteboard.ts`/`native-spotlight.ts`（白板绘制与聚光灯）、`read-scene.ts`、`web-search.ts`。每个 Child Agent（讲师/助教/学生）由 `lib/agent/runtime/build-agent.ts` 构造，经 `runNativeChild` 独立跑一轮 LLM，其可见文本流式返回课堂并通过 TTS 播报。

Agent 角色体系是静态人设注册表：`lib/orchestration/registry/built-in.ts` 内置 default-1 "AI teacher"（讲师，优先级 10，可全部幻灯片与白板动作）、default-2 "AI助教"（补位答疑）、default-3 "显眼包"（插科打诨的活跃学生）、default-4 "好奇宝宝"（追问型学生）、default-5 "笔记员"（总结要点），每人有独立 persona、avatar、allowedActions 白名单；课程可携带自定义 roster，`agent-selection.ts` 负责预设选择与课程自带角色的恢复策略。

Director 的核心工具契约刻意保持极简（`lib/chat/pi/tools/call-agent.ts`）：

```ts
const CallAgentParams = Type.Object({
  agentId: Type.String({
    description: 'ID of the classroom agent that should speak next.',
  }),
  instruction: Type.String({
    description: 'Specific instruction and context for the selected agent response.',
  }),
});
```

即"导演只决定谁来说、说什么方向"，台词和课堂动作由对应人设的 Child Agent 自主生成——人设提示词与 `buildLengthGuidelines`（讲师 70 字/助教 60 字/学生 40 字的中文长度约束，`lib/chat/pi/prompts.ts`）共同控制课堂节奏。

工程细节相当讲究（`lib/chat/pi/README.md`）：每个模型回合前 Director/Child 各自读取白板快照作为临时 state 消息（不入历史）；配置 `RENDER_SERVICE_URL` 且模型支持视觉时，一次请求最多附一张白板 PNG（5 秒渲染时限，失败即降级纯文本）；上下文超限走 `director-compaction.ts` 压缩。讨论链路通过 SSE 流式推送（`app/api/chat/`，README 项目结构注明 "Multi-agent discussion (SSE streaming)"）。

### 5.3 持久 Agent 运行时（`lib/server/agent-runtime/runner.ts`）

Workbench 的课程搭建 Agent 是数据库化长会话：runner 以"租约协调的后台执行"为设计——每个应用进程都可运行该循环，PostgreSQL 是租约认领、租约代际、事件排序、取消和会话恢复的唯一权威，"客户端连接从不属于执行生命周期"。具备心跳、崩溃恢复、运行中追加指令（steer）、取消等语义；工具面覆盖课程 CRUD、场景原子补丁、素材抽取检索、图片/视频/语音生成、PPTX 导入、技能创建等 50 余个文件（`lib/server/agent-runtime/` 目录）。

### 5.4 播放引擎与动作系统（`lib/playback/`、`lib/action/`）

`lib/playback/engine.ts` 是 idle → playing → live 的播放状态机（README Key Architecture），`action-navigation.ts` 把动作分为白板类（wb_open/wb_draw_text/wb_draw_shape/wb_draw_chart/wb_draw_latex/wb_draw_code 等 12 种）和不安全跳转类（play_video/discussion/widget_*），支持任意动作跳转后的前缀重构与断点续播（`action-resume.ts`）；`lib/action/engine.ts` 执行语音、白板、聚光灯、激光笔等课堂动作，使 Agent 输出逐帧变成可视课堂。

## 6. 构建与运行

来自 `package.json` scripts 与 README Quick Start：

```bash
pnpm install            # postinstall 会依次构建全部 workspace 包
cp .env.example .env.local            # 至少一个 LLM key（如 OPENAI_API_KEY）
cp openmaic.example.yml openmaic.yml  # providers/slots 模型槽位配置
pnpm db:up              # docker compose 起 PostgreSQL 16（需配 DATABASE_URL）
pnpm dev                # http://localhost:3000
pnpm build && DATABASE_URL=... pnpm start   # 生产：长驻进程，生成不随浏览器中断
docker compose up --build                     # 双容器部署（app + PostgreSQL，默认单用户模式）
```

测试与质量命令：`pnpm test`（vitest）、`pnpm test:e2e`（Playwright）、`pnpm lint`、`pnpm check`（prettier）、`pnpm eval:pbl-v2-planner` 等 LLM 评测脚本。模型配置用 YAML 声明槽位（llm/course.outline/image/video/tts/webSearch），key 经 `${VAR}` 引用 `.env.local`，启动时按字段名报错校验（README Configure 节、`openmaic.example.yml`）。

## 7. 测试与工程质量

- 测试规模：1,541 个 `*.test.ts(x)` 文件，其中 `tests/` 1,256、`packages/` 161、`render-service/` 32（find 统计）；另有 `e2e/` Playwright 端到端与 `eval/` 三个 LLM 输出质量评测集（编排答问、白板布局、PBL 规划）
- CI：`.github/workflows/ci.yml` 拆分 lint+typecheck、4 分片单测、逐包（importer/geometry/dsl/renderer/editor/generation/storage）测试与 TS 检查、包版本 bump 检查、Node engine 契约检查、render-service 独立安装测试；另有 storage-pg-contract、docs-build、flaky-report、issue-triage（自研 agent 自动分诊，`.github/scripts/issue-triage.mjs`）
- 安全工程化突出：v1.0.2/v1.0.3/v1.1.1/v1.1.2/v1.1.3 五个版本连续修复 SSRF、DNS rebinding、XSS 净化等问题并发布 GHSA 公告（README News），代码中有 `sanitize/`、SSRF 用的 `ipaddr.js`/`tldts` 依赖
- 观察到的短板：单仓库 7.8 万行 + 50 个 lib 子目录对新人门槛高；构建链长（postinstall 串行构建 9 个包 + 3 个同步脚本）

## 8. 社区与维护状态

- 活跃度极高：2026-03-11 建仓，8 个月 43.5k 星（日均约 180 星，超常规增长节奏）；昨天（2026-11-13）仍在推送；11 月两周内连发 v1.4.0/1.4.1/1.4.2/1.5.0-rc.1/rc.2 五个 release（gh api releases，README News）
- 贡献者结构：126 名贡献者，但呈典型核心团队模式——前 3 人（wyuc/cosarah/YizukiAme）合计 546 次提交占绝对主导，论文署名 21 人来自清华（README Citation）；外部贡献多为单点修复（如 @mahmud035、@LING-6150，CHANGELOG 1.5.0-rc.2）
- 生态运营：Discord + 飞书社区双通道（README 徽章），托管 Demo open.maic.chat，ClawHub 一键安装技能（`clawhub install openmaic`），12 语言 i18n（`lib/i18n/locales/`）
- 增长背景可客观陈述的要素：清华品牌与顶刊论文背书、AI 教育 2026 年高热赛道、"开源可自部署的多 Agent 课堂"品类稀缺、国产模型全家桶适配 + 中文优先（中英双语 README、飞书社区）触达国内开发者、与 OpenClaw 等当红 Agent 工作台联动借力

## 9. 亮点与风险

亮点：
1. 品类完整度罕见——从生成（大纲/场景/配图/TTS）到课堂交互（多 Agent 讨论/白板/聚光灯）到导出（PPTX/离线 HTML/MP4）全链路闭环，且全部开源（`packages/@openmaic/*`）
2. 多 Agent 编排设计务实：Director 决策与人设执行分离、白板状态作为共享黑板每回合注入、视觉模型可选看图（`lib/chat/pi/`，设计决策记录在同目录 README.md）
3. 服务端工程成熟：租约式持久会话、崩溃恢复、逐场景修订计数、server-first 生成不丢（`lib/server/agent-runtime/runner.ts`）
4. 测试与 CI 在同类 AI 应用项目中属于第一梯队（1,500+ 测试文件、逐包流水线、LLM eval 分离）
5. 模型与部署中立：20+ 提供商、槽位化 YAML 配置、本地推理（Ollama/Lemonade/FunASR/VoxCPM2）支持内网部署

风险/局限：
1. 资源门槛不低：必须 PostgreSQL、生成需长驻进程，1.2.0 起不再支持 Vercel serverless；视频导出/白板视觉还需额外 render-service 容器
2. 代码体量与迭代速度带来维护压力：单仓 7.8 万行、两周 5 个 release（含 rc），API 与配置在 1.1.x → 1.2.x 出现过破坏性迁移（README Vercel/Upgrading 节）
3. 核心编排依赖外部框架 `@earendil-works/pi-agent-core`（锁 0.80.3 精确版本），该包生态与维护风险独立于本项目
4. 贡献高度集中于 3 人核心，社区 BUS 因子偏低；商业化路径未声明，仅留合作邮箱与 MIT 授权（README Partnerships）

## 10. 总结评价

OpenMAIC 是 2026 年 AI 教育赛道完成度最高的开源作品之一：它不是"AI 生成 PPT"，而是把课程变成一场由讲师/助教/学生多个 Agent 实时演出的课堂，生成管线、持久运行时、播放引擎、导出链路四块拼图都有扎实的工程实现和测试覆盖。8 个月 43.5k 星是清华品牌、JCST 论文、产品级完成度与中文生态运营共同作用的结果，但 126 名贡献者中绝大多数提交来自 3 人核心团队，说明它仍是一个"产品化的实验室项目"。适合三类人：想自部署一套 AI 课堂的教育者/机构（Docker 一键起）；做多 Agent 编排的工程师（Director-Child 分工、白板黑板机制、租约会话都值得抄作业）；想基于课程 DSL 做二开的开发者（`@openmaic/dsl`+renderer+generation 有 npm 包与 SKILL.md 指南）。读源码建议按 lib/chat/pi → packages/@openmaic/generation → lib/server/agent-runtime → lib/playback 的顺序切入。
