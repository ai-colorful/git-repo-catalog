# DayuanJiang/next-ai-draw-io 项目分析

> 星数 36997（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-17

## 1. 项目速览

一句话定位：用自然语言对话生成和编辑 draw.io 图表的 AI 画图工具，提供 Web 应用、桌面应用（Electron）与 MCP server 三种形态，MCP server 可被 Claude Code、Cursor、Codex、VS Code 等 AI agent 直接调用。

| 项目 | 内容 |
| --- | --- |
| 名称 | DayuanJiang/next-ai-draw-io |
| 星数 / Fork | 36997 / 3839（gh api，2026-11-17 核实） |
| 主语言 | TypeScript（447 个 TS/TSX 文件，约 11.5 万行，含测试） |
| 许可证 | Apache-2.0（`LICENSE`、`package.json`、gh api 三处一致；注意是 Apache-2.0 而非 MIT） |
| 创建时间 | 2025-03-23（gh api） |
| 最近推送 | 2026-11-16（gh api，持续高频活跃） |
| 最新 Release | v0.5.1（2026-10-31，gh api） |
| 贡献者 | 70 人（gh api contributors 分页计数） |

## 2. 项目是什么，解决什么问题

用户用一句话描述需求，AI 在真实的 draw.io 画布上画出架构图、流程图、时序图、ER 图、BPMN 等，并且画完之后它仍然是一张普通的 draw.io 图：可以手动拖拽、改名、多页编辑，随时导出 `.drawio`、`.png`、`.svg`（README.md「Highlights」节）。它解决的痛点是"AI 生图"类工具普遍只给一张静态图片——本项目的输出是可继续手工编辑的矢量源文件，且 AI 的每次修改都落成可对比、可回滚的版本卡片（`stores/versions-store.ts`，`MAX_VERSIONS = 20`）。

与 draw.io 自带的 Generate 工具（Arrange > Insert > Generate）相比，README 明确列出了差异：本项目每条消息都附带当前画布（包括用户的手工修改），每次 AI 修改都可对比/恢复，并支持 25 家模型供应商含本地 Ollama。这也解释了产品定位：不是替代 draw.io，而是给 draw.io 加一层"懂上下文的 AI 操作层"。

项目支持上传截图/草图让 AI 重绘，也支持上传 PDF、Markdown、代码等文本文件生成内容图（README「Draw」节；对应 `lib/pdf-utils.ts`、`lib/use-file-processor.tsx`）。内置 AWS/Azure/GCP/Kubernetes 等图标库（`docs/shape-libraries/` 下有 aws4、azure2、gcp2、kubernetes、alibaba_cloud、android、atlassian 等十余份语法文档），连接线可带流动动画（`flowAnimation=1`，见 `lib/chat-tools.ts` 工具描述）。

第三种形态是 MCP server（`packages/mcp-server`，npm 包 `@next-ai-drawio/mcp-server`）：AI agent 调用工具后，本机起一个内嵌 HTTP server 并打开浏览器画布，图"边画边出现"在用户眼前，且自带版本历史、多页、导出与自动保存（`~/.next-ai-drawio/`），draw.io 编辑器随包内置、离线可用（packages/mcp-server/README.md 与 `packages/mcp-server/src/index.ts`）。该项目已进入 Claude Code 官方社区插件市场（README 安装命令）。

增长背景：项目创建于 2025 年 3 月（MCP 协议生态爆发、AI 生成内容工具热潮叠加期），靠 MCP 集成与 TrendShift 热榜（README 徽章）出圈，当前日活提交数十次（gh api commits，2026-11-12 至 11-14），是"AI 图表生成"赛道星数最高的开源项目之一。

## 3. 技术栈与依赖

- **框架**：Next.js 16（App Router、`[lang]` i18n 动态路由、Turbopack dev，`package.json`）+ React 19 + Tailwind CSS 4 + Radix UI + zustand 5（四个 store：canvas/settings/ui/versions）
- **AI 层**：Vercel AI SDK v6（`ai` 包 + `@ai-sdk/react`），这是整个产品的骨架；`zod` 4 定义工具 schema，`jsonrepair` 修复模型输出的非法 JSON
- **Provider 适配**：`@ai-sdk/anthropic`、`@ai-sdk/openai`、`@ai-sdk/google`、`@ai-sdk/amazon-bedrock`、`@ai-sdk/azure`、`@ai-sdk/deepseek`、`@ai-sdk/google-vertex`、`@ai-sdk/openai-compatible`、`ollama-ai-provider-v2`、`@openrouter/ai-sdk-provider`、`@aihubmix/ai-sdk-provider` 等（package.json dependencies）
- **XML 处理**：`saxes`（流式 XML 解析校验）、自研 `xml-validation.ts`/`xml-syntax.ts` 校验与自动修复；`pako`（压缩）
- **MCP**：`@modelcontextprotocol/sdk` ^1.31.0、`linkedom`（服务端 DOM polyfill）
- **可观测/运营**：Langfuse（tracing）+ OpenTelemetry、AWS DynamoDB（配额管理 `lib/dynamo-quota-manager.ts`）、访问码机制（`lib/access-code.ts`）、管理后台（`app/[lang]/admin/`）
- **桌面**：Electron 45 + electron-builder（win/mac/linux 三平台），main/preload 用 esbuild 打包（`package.json` scripts）
- **质量工具**：Biome（lint+format，替代 ESLint/Prettier）、Vitest、Playwright、husky + lint-staged、renovate（`.github/renovate.json`）

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `app/[lang]/` | i18n 路由的主应用（画布+聊天面板页）、admin 后台、docs 站、gallery、mcp 介绍页 |
| `app/api/` | 12 个 API route：chat（核心流式对话）、validate-diagram、admin、配额、模型配置等 |
| `components/` | 聊天（chat/）、画布（canvas/）、AI 元素（ai-elements/）、设置（settings/）、工作区（workspace/）等 UI |
| `lib/` | 业务核心：ai-providers/providers（25 家供应商）、chat-tools/chat-helpers/system-prompts、diagram-validator/diagram-diff/diagram-pages、drawio/editor-bridge（画布桥接）、i18n、langfuse、配额 |
| `contexts/` | diagram-context.tsx（740 行，画布状态与 AI 工具应用的中枢） |
| `hooks/` | use-diagram-tool-handlers（浏览器侧工具执行）、use-validate-diagram（VLM 视觉校验）等 |
| `stores/` | zustand：canvas-store、versions-store（版本卡片）、settings-store、ui-store |
| `packages/mcp-server/` | 独立 npm 包（约 1.5 万行）：16 个 MCP 工具、内嵌 HTTP server、浏览器画布 shell、XML 校验/布局后处理 |
| `packages/claude-plugin/`、`packages/dsh-plugin/` | Claude Code 社区市场插件、DSH 插件（cordis 生态） |
| `electron/` | 桌面壳：main（内嵌 Next server 的 UtilityProcess、端口/代理/窗口/PDF 导出管理）+ preload |
| `edge-functions/`、`wrangler.jsonc`、`Dockerfile`、`edgeone.json` | 四套部署目标的适配：腾讯 EdgeOne Pages、Cloudflare Workers、Vercel、Docker |
| `docs/` | en/cn/ja 三语文档站内容 + shape-libraries 图标库语法手册 |
| `tests/` | unit（118 个测试文件）+ e2e（33 个 Playwright spec） |

组织方式评述：不是严格 monorepo（无 workspace 配置），而是"主应用 + 内嵌子包"结构——最巧妙的一点是 **web 应用直接 import MCP server 包内的源码**（如 `lib/chat-tools.ts` 第 4-15 行 import `@/packages/mcp-server/src/diagram-types`、`shape-library`、`xml-examples`），使"绘图规则、XML 校验、布局后处理"在 Web 与 MCP 两种形态间保持单一事实来源，CI 还专门守卫这一耦合（`.github/workflows/test.yml` 中"bundled code changed 未升版本则告警"）。

## 5. 核心架构与关键实现

### 5.1 AI 生成管线：五个工具 + 浏览器执行 + 多层修复

服务端 `app/api/chat/route.ts`（727 行）用 AI SDK 的 `streamText` 驱动，工具定义在 `lib/chat-tools.ts`，分为两类：`display_diagram`（新建全图）、`edit_diagram`（按 cell id 增/改/删）、`append_diagram`（续写被截断的输出）**没有 `execute` 函数**——它们由浏览器端 `hooks/use-diagram-tool-handlers.ts`（663 行）执行，结果通过 `addToolOutput` 回传继续生成；`get_shape_library`、`get_diagram_guide` 在服务端执行（读 `docs/shape-libraries/` 文档返回给模型）。

工具 schema 本身就是"防呆提示词"：`display_diagram` 的描述里写死了 6 条校验规则（只要 mxCell、不要包裹标签、不要 root 单元格、id 从 2 起、转义规则等，`lib/chat-tools.ts:27-46`），并用 `SWIMLANE_EXAMPLE`、`STYLE_CLASS_EXAMPLE`（来自 `packages/mcp-server/src/xml-examples.ts`）做 few-shot。容错链路完整：`jsonrepair` 修工具入参 JSON（route.ts import）、`fixToolInputJson`/`dropInvalidToolCalls`（`lib/chat-helpers.ts`）、客户端自动重试被拒的工具调用 3 次 + 续写 2 次（route.ts 第 83-86 行注释 `FOLLOW_UPS_PER_TURN = 5`）。

上下文机制值得注意：画布不随每条消息重复发送，而是以 `[Canvas when this message was sent ...]` 注记形式嵌入用户消息（`lib/system-prompts.ts`），用户手工改动后注记升级为"画布快照即事实来源"，避免长对话中上下文爆炸。

### 5.2 XML 校验与 VLM 视觉闭环

语法层：`packages/mcp-server/src/xml-validation.ts`（共享模块，头注释明说"shared by the MCP server and the web app"）+ `xml-syntax.ts`（saxes 解析）做结构校验与属性去重自动修复。视觉层：每次出图后截屏送 `/api/validate-diagram`，由 VLM 检查重叠、连线穿越等问题，失败结果经 `formatValidationFeedback`（`lib/diagram-validator.ts`）格式化为工具错误回传给模型重新生成，最多重试 3 次（`hooks/use-diagram-tool-handlers.ts` 中 `MAX_VALIDATION_RETRIES = 3`）。这是少见的"生成→渲染→看图→自我修正"完整闭环。

### 5.3 draw.io 集成：同源 iframe + 内部对象直连

`scripts/fetch-drawio.mjs` 在 dev/build 前下载 pin 版 draw.io（draw.war，SHA-256 校验，版本与 MCP server 共享 `packages/mcp-server/src/drawio-version.json`）到 `public/drawio`。关键决策在脚本头注释里写得非常直白：**同源嵌入让页面可以直接调用 draw.io 内部对象**（跨源 iframe 只能走 postMessage 协议，做不到高亮 AI 改动、读取选区、接管撤销）。`lib/drawio/editor-bridge.ts`（1251 行）封装了对 `EditorUi`/`mxGraph` 私有 API 的全部访问，且每个调用都做存在性检查——draw.io 升级改了内部命名时功能降级为 console 警告而非崩溃（文件头注释与 `warnOnce` 实现）。AI 改动后画布上橙色描边高亮约 3 秒、Ctrl+Z 一步撤销 AI 修改（`system-prompts.ts` "App Features" 节与 `editor-bridge.ts`）。`lib/diagram-diff.ts` 的 `diffDiagrams` 负责生成改动摘要，驱动版本卡片描述。

### 5.4 MCP server：内嵌 HTTP server + 浏览器画布 + 编辑门禁

`packages/mcp-server/src/index.ts`（2291 行）注册 16 个工具：`start_session`、`get_drawing_guide`、`get_shape_library`、`create_new_diagram`、`edit_diagram`、`get_diagram`、`get_selection`、`screenshot_diagram`、`export_diagram`、`list/add/rename/delete_page`、`restore_version`、`load_diagram`、`list_saved_diagrams`。`start_session`（index.ts:566 起）启动内嵌 HTTP server（`http-server.ts`）并打开浏览器画布，画布通过轮询拉取更新（session state 的 `lastPolled`）；重复调用同一会话不会开第二个标签页，还会检测标签页是否已被用户关闭（`TAB_CLOSED_MS` 静默期后重开）。`edit-gate.ts` 实现编辑门禁：模型没看过当前画布状态就拒绝编辑，防止基于过时认知的破坏性修改。`diagram-types/` 目录含 sequence 布局引擎、grid-route 网格布线、line-repair 连线修复等后处理，弥补小模型空间推理的短板。npm 发布前 CI 会把打包 tarball 装进空目录、跑 headless Chromium 全链路 E2E（test.yml "MCP shell E2E" 步骤）。

### 5.5 多 Provider 与双形态部署

`lib/providers.ts` 定义 25 家供应商（含 OpenAI-compatible 万能预设），`lib/ai-providers.ts`（1523 行）处理服务端凭据、思考模式开关、提示缓存（`CACHE_POINT`）、输出 token 上限降级。密钥可留在浏览器（`settings-store.ts`）或服务端统一配置（`AI_MODELS_CONFIG`/`ai-models.json` + `/admin` 管理台）。部署形态覆盖 Vercel、Cloudflare Workers（@opennextjs/cloudflare）、腾讯 EdgeOne Pages（还附赠 DeepSeek 免费额度，README）、Docker 与离线内网（docs/en/offline-deployment.md）。桌面版用 Electron 45：`electron/main/next-server.ts` 在 UtilityProcess 里跑内嵌 Next server（端口自动分配、健康检查轮询），另有代理管理、PDF 导出、菜单 i18n 等模块。

## 6. 构建与运行

```bash
# Web 应用（README「Self-host」节 + package.json scripts）
git clone https://github.com/DayuanJiang/next-ai-draw-io && cd next-ai-draw-io
npm install
cp env.example .env.local    # 配置 provider 与 API key
npm run dev                  # Next dev (Turbopack)，端口 6002；predev 钩子自动下载 pin 版 draw.io

# 质量检查
npm run check                # biome ci（lint）
npx tsc --noEmit             # 类型检查（CI 同款）
npm run test -- --run        # vitest 单测
npm run test:e2e             # Playwright E2E

# 桌面版
npm run dist:win             # electron:build + prepare + electron-builder（另有 dist:mac/dist:linux/dist:all）

# MCP server（独立包）
npm --prefix packages/mcp-server install
npm --prefix packages/mcp-server run build   # tsc + copy-assets + fetch-drawio + build-shell
npm --prefix packages/mcp-server test        # vitest

# MCP 客户端接入（README）
claude mcp add --scope user drawio -- npx @next-ai-drawio/mcp-server@latest
```

## 7. 测试与工程质量

测试规模在同类项目中突出：主应用 118 个单测文件（Vitest + jsdom + Testing Library，含快照测试，`tests/unit/`）+ 33 个 Playwright E2E spec（`tests/e2e/`，覆盖聊天、画布状态、多轮对话、文件上传、i18n、无障碍 screen-reader 等）；MCP server 另有 20+ 个单测与独立 E2E（`packages/mcp-server/tests/`）。CI（`.github/workflows/test.yml`）三条流水线：lint+tsc+单测；MCP 构建产物校验（check-package、drawio 文件清单守卫）+ 打包 tarball 装 npm 后 headless Chromium E2E；Web E2E 按 6 分片并行。质量基建：Biome 统一 lint/format、husky + lint-staged 提交卡口、renovate 自动升级依赖、auto-format 工作流。代码注释质量极高（大量"为什么"而非"是什么"的英文注释，如 route.ts 里对 undici abort 丢失 bug 的解释），且关键容错路径（截断续写、跨源降级、标签页重开）都有注释指明成因。整体是观察到的 20k+ 星档位里工程质量第一梯队的水准。

## 8. 社区与维护状态

- **活跃度**：极高。gh api 显示最近推送 2026-11-16，2026-11-12 至 11-14 三天内 30+ 次提交；35 个 open issue，3839 fork。
- **发布节奏**：npm MCP 包 0.4.x→0.5.1 稳步推进；GitHub Release v0.5.1（2026-10-31）、v0.5.0（2026-10-10），桌面安装包随 Release 发布（electron-release.yml 工作流）。
- **贡献者**：70 人出现在 contributors 列表，但从提交密度与代码风格一致性看，核心开发高度集中在作者 Dayuan Jiang（GitHub 账号 2017 年注册，350 followers）个人（此判断基于提交流观察，精确占比未确认）。
- **国际化社区**：README 与 docs 均有英/简中/日文三语版本（`docs/cn/`、`docs/ja/`），日语文档的存在暗示有非中文语境的贡献或用户群（具体未确认）。
- **商业化迹象**：GitHub Sponsors + demo 站配额制（访问码、DynamoDB 配额、EdgeOne 赞助额度），README 明确求赞助以维持 demo 站运行。

## 9. 亮点与风险

**亮点**

1. "生成即版本"的产品设计：每次 AI 修改都是带缩略图的版本卡片，可对比/恢复/一步撤销，把 AI 当合作者而非黑盒（`stores/versions-store.ts`、`lib/diagram-diff.ts`）。
2. 同源 iframe + draw.io 内部对象直连的工程决策，换来跨源 postMessage 做不到的深度集成（高亮、选区、撤销），并有优雅降级保护（`lib/drawio/editor-bridge.ts`）。
3. Web 与 MCP server 共享同一套绘图规则/XML 校验/布局后处理代码，两形态能力零漂移（`lib/chat-tools.ts` 直接 import `packages/mcp-server/src/*`）。
4. 完整的生成可靠性闭环：schema 内嵌校验规则 → jsonrepair → 工具重试 → 截断续写 → VLM 看图自检，层层兜住小模型的输出瑕疵。
5. 测试与 CI 之扎实（672 行的 test.yml、打包产物 E2E）在个人项目中罕见。

**风险/局限**

1. 与 draw.io 私有 API（EditorUi/mxGraph）强耦合，上游大版本重构会成片触发功能降级；项目用 pin 版 + SHA-256 锁住 draw.io 缓解，但本质是"追着内部实现跑"（`scripts/fetch-drawio.mjs`、`drawio-version.json`）。
2. 长图生成受模型输出 token 上限约束，复杂大图依赖 append_diagram 续写机制，小模型仍易出碎片化结果（README 自己也提示"small models tend to produce broken diagrams"）。
3. 核心开发单点集中在一人，70 贡献者多为外围；作者个人节奏决定项目存续（bus factor 风险，未确认但结构性存在）。
4. MCP 画布依赖"本机浏览器 + 轮询"而非 WebSocket 长连接，远程/无头环境下体验受限（`packages/mcp-server/src/http-server.ts` 有 remote-preview 相关测试佐证该场景已考虑）。

## 10. 总结评价

这是"AI + 传统开源工具"结合的标杆样本：它没有重造画布，而是把 draw.io 这个 20 年历史的老牌工具变成 AI 的手和眼，再靠 MCP 协议把自己变成所有 AI agent 的绘图器官——三种形态（Web/桌面/MCP）共享一套核心代码，工程纪律与产品完成度都远超星数同级平均水平。适合想研究"如何给 LLM 输出做多层容错""如何逆向集成无 API 的前端应用（同源 iframe 直连内部对象）""如何写一个体验良好的 MCP server"的开发者精读源码；适合所有需要把想法快速变成可手工精修的正式图表的工程师、架构师与文档作者作为日常工具。
