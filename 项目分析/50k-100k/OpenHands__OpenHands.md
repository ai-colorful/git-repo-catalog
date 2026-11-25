# OpenHands/OpenHands 项目分析

> 星数 91647（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：OpenHands（原 OpenDevin，"AI-Driven Development"）的主仓库现已转型为 **Agent Canvas** —— 一个自托管的 AI 编码 agent 控制中心（开发者控制台），用于启动对话、编排自动化，并连接本地/远程/云端多种 agent 后端。

| 项目 | 内容 |
|---|---|
| 名称 | OpenHands/OpenHands（npm 包 `@openhands/agent-canvas` v1.34.0） |
| 星数 | 92,270（gh api 实测 2026-10-31；快照 91,647） |
| 语言 | TypeScript（本仓库 2279 个 TS/TSX 文件，Python 仅 17 个） |
| 许可证 | MIT |
| 创建时间 | 2024-03-13；最近 push 2026-10-30（gh api） |
| 贡献者 | 约 590 人（contributors API 分页推算） |
| 形态 | 前端 + 本地栈编排 + Electron 桌面端；agent 执行本体已拆分到 `software-agent-sdk`（Python）仓库 |

## 2. 项目是什么，解决什么问题

OpenHands 最初（OpenDevin 时代）是"AI 软件工程师"单体仓库：Python 后端实现 agent 循环、事件流与 Docker runtime 沙箱，外加 React 前端。基于本次 HEAD 快照的真实文件形态（`README.md`、`AGENTS.md`），该仓库已完成架构重塑：**agent 循环、工具、事件与沙箱执行全部迁往 `OpenHands/software-agent-sdk`（Python SDK + Agent Server）**，本仓库专职承载 "Agent Canvas" —— 面向开发者的控制中心 UI 与本地栈启动器（`README.md` "Repository boundaries" 一节有明确的四仓职责表）。

它解决的问题是把散落的编码 agent（OpenHands 自家 agent，以及 Claude Code、Codex、Gemini CLI、OpenCode 等任何兼容 ACP——Agent-Client Protocol 的第三方 agent）收敛到统一界面：在同一个前端里启动对话、切换后端、配置 LLM，并把 agent 接入自动化流水线（定时任务、webhook 触发，对接 Slack/GitHub/Linear 等，见 `README.md` 特性表）。底层 "Agent Server" 是一个 REST API 服务，单机可运行多个 agent；Canvas 可同时连接多个 Agent Server 并随时切换（`docs/architecture.md`）。

产品形态覆盖四种：npm 全局命令 `agent-canvas` 一键拉起完整本地栈（`bin/agent-canvas.mjs`）；Docker all-in-one 镜像（`docker/Dockerfile`）；Electron 桌面应用（`electron/`）；以及可作为组件库嵌入宿主应用的库构建（`package.json` 的 subpath exports）。

安全模型上做了分层：本地默认只绑定 `127.0.0.1` 并自动生成会话密钥注入页面（`bin/agent-canvas.mjs` 帮助文本）；绑定非回环地址或 `--public` 模式时密钥不注入、需手动输入；同时 README 反复警告"无沙箱模式下 agent 拥有完整文件系统访问权"，推荐 Docker 沙箱模式（`README.md` Option 1 警告块、`docs/SELF_HOSTING.md`）。

## 3. 技术栈与依赖

来源：`package.json`（依赖全部精确锁版本）。

- **核心框架**：React 19.3 + react-router 7（framework mode，含 `@react-router/node`）+ Vite 8 + TypeScript 6.0；Node >= 24，npm 锁定 11.12.1（`packageManager` 与 Volta 双重钉死）。
- **状态与数据**：zustand 5（26 个 store，`src/stores/`）、@tanstack/react-query 5（hooks/query）、axios + ws。
- **UI**：TailwindCSS 4、HeroUI、Monaco Editor（代码编辑）、xterm（终端）、framer-motion、react-markdown 全家桶（remark-gfm/rehype-sanitize 等）。
- **国际化**：i18next，单一 `translation.json` 源文件 + 脚本生成声明与语言包（`scripts/make-i18n-translations.cjs`）。
- **桌面与分发**：Electron 43 + electron-builder；`bin/` CLI 入口；npm trusted publishing 发布（`docs/architecture.md`）。
- **Python 侧（运行时拉取，不在本仓库）**：agent-server 与 automation 后端通过 `uvx` 直接从 PyPI/git ref 运行，无需 clone（`scripts/dev-with-automation.mjs` 头注释）。
- **供应链安全特色**：强制"依赖发布满 7 天才可引入"策略，`npm install --min-release-age=7`，Dependabot cooldown 7 天，安全更新例外需维护者批准（`AGENTS.md` "Dependency release-age policy"）。

## 4. 目录结构与代码组织

顶层（约 2831 文件）：

| 目录 | 职责 |
|---|---|
| `src/` | 主应用，约 6.36 万行 TS/TSX（wc 统计） |
| `__tests__/` | 与 src 平行的单元测试树（约 8.5MB） |
| `tests/e2e/` | Playwright E2E（live、mock-llm、docker、bind-policy 四套配置） |
| `scripts/` | 本地栈启动器与运维脚本（核心 `dev-with-automation.mjs` 2097 行） |
| `bin/` | `agent-canvas` CLI 入口 |
| `docker/`、`helm/` | all-in-one 镜像与 K8s 部署 |
| `electron/` | 桌面壳（main/preload） |
| `config/` | `defaults.json`（栈版本/端口/兼容性基线） |
| `.agents/skills/` | 面向 AI 贡献者的技能文档（telemetry、e2e、API 契约、local-stack、electron 等 9 个） |
| `.github/workflows/` | 20 个 workflow（CI、三平台桌面、docker、npm 发布、release-please 等） |

`src/` 内部（`docs/architecture.md` "Frontend modules" 与实际 ls 对照）：

- `api/` 约 60 个服务适配器模块：conversation-service、event-service、backend-registry（多后端注册）、cloud（云代理）、automation-service、mars（托管 agent/隧道）、settings、secrets、mcp、skills、tool-catalog 等，并带 `no-direct-agent-server-calls.test.ts` 这类架构守护测试。
- `components/` 按 feature 分组：conversation、conversation-events、browser、files、terminal、sidebar、settings、features（automations/backends 等）、providers、ui（shadcn 风格基础件）。
- `hooks/` 263 个文件（含大量 co-located 测试）、`stores/` 26 个 zustand store、`routes/` react-router 路由文件、`contexts/` 全局 provider、`i18n/`、`mocks/`（MSW）。

组织方式评述：单仓多形态（SPA + 库 + 桌面 + CLI），前端代码量占绝对主体；与 Python agent 能力解耦成多仓体系，依赖方向被强制约束为 "Agent Server 契约 → TypeScript client → Agent Canvas"（`AGENTS.md` "Repository Ownership"）。

## 5. 核心架构与关键实现

### 5.1 本地栈编排（launcher）

`scripts/dev-with-automation.mjs` 是整个产品的"发动机"：单脚本编排 ingress 代理（`scripts/ingress.mjs`，基于 httpxy）、agent-server（uvx 启动 Python 服务，默认 :18000）、automation 后端（uvx，:18001）与前端（Vite 或静态 sirv）。CLI 与生产栈共用同一脚本（`bin/agent-canvas.mjs` 尾部直接 `import("../scripts/dev-with-automation.mjs")`）。

agent-server 启动逻辑（`scripts/dev-with-automation.mjs:1132-1189`）：通过 `buildAgentServerCommand` 解析命令源（PyPI 版本 / git ref / 本地 SDK checkout 三级优先），注入会话密钥 `OH_SESSION_API_KEYS_0`、`LOG_JSON=true` 等环境变量后 spawn；还把自动化服务的 API key 主动 seed 进 agent-server 的 secrets store（`seedAutomationSecret`，带 5 次重试），让对话中的 agent 能用密钥回调 automation。

会话密钥策略集中在 `applySessionKeyPolicy`：回环绑定自动注入 HTML，非回环/`--public` 时改走 API-key 输入页（`scripts/dev-with-automation.mjs:1511-1527` 注释明确"Local loopback mode keeps transparent auth"）。

### 5.2 事件流：双 WebSocket + 断线恢复

对话实时通道由 `src/contexts/conversation-websocket-context.tsx`（1616 行）承载，是前端最核心的状态机：同时维护主对话 socket 与 planning socket 两条连接（文件内 `handleMainMessage` / `handlePlanningMessage`，:647/:931）。要点：

- **seq 简历游标**：每帧推进 resume cursor，事件带 `seq` 戳（:163-169 注释），重连后从游标回放积压（:661 "A reconnect replays the backlog from a stale anchor"）。
- **重连工程**：`src/hooks/use-websocket.ts` 实现指数退避 1s→30s 封顶，随机 30% jitter 防止主/规划两条 socket 同步重试压垮服务器（:133-140 注释），另有握手看门狗 `startHandshakeWatchdog` 防止 CONNECTING 状态卡死占用 Chrome 每主机握手锁（:69-72）。
- **页面可见性**：`use-page-visibility-reconnect` 后台断开、回前台立即重连（:1376-1380）。
- **历史回填走 REST**：`src/api/event-service/event-service.api.ts` 的 `searchEvents` 用 `next_page_id` 游标分页；云端模式拆分两个上游——事件历史走云 App API（`/api/v1/conversation/{id}/events/search`），实时 runtime 端点直连每会话沙箱主机（文件头注释详细解释了双主机认证差异，:17-38）。

用户消息发送链路：`use-send-message.ts` 把聊天输入（文本/图片/文件）转成 SDK 的 `MessageContent` 数组（type: "text"/"image"），经 conversation WebSocket context 以 `{role:"user", content}` 发出。

### 5.3 Runtime 与沙箱

本仓库不含沙箱实现，但负责"选择与接线"：`OH_CONVERSATION_RUNTIME=docker` 时每个新对话在独立 Docker 容器中运行（各带自己的 Agent Server），workspace 与状态以卷挂载实现容器替换后数据不丢（`README.md` Option 3）；launcher 层相应把主机别名切为 `host.docker.internal` 并改写 automation URL（`scripts/dev-with-automation.mjs:1450-1475`）。会话元数据中的 `runtime_status`（available/starting/missing/ownership_lost/error）由前端统一归一化（`src/api/conversation-service/agent-server-conversation-service.api.ts:193-207`）。真正的容器生命周期、工具执行、agent 循环都在 `software-agent-sdk` 仓库（本仓库 Dockerfile 直接引用上游 `ghcr.io/openhands/agent-server` 镜像，`docker/Dockerfile` 头注释）。

### 5.4 运行时服务信息回注 agent

一个有意思的设计：launcher 把各服务端口/URL 组装成 runtime services info，经 agent-server 的 `/server_info.runtime_services` 暴露，前端再把它作为上下文后缀注入新对话的系统提示，让 agent 自己知道 automation/编辑器等服务的正确地址而非猜端口（`docs/architecture.md` "Runtime services"、`scripts/dev-with-automation.mjs:1444-1469`、`scripts/runtime-services-info.mjs`）。

### 5.5 前端组织

- **双构建目标**：`build:app` 出独立 SPA，`build:lib` 出可嵌入库，按 browser/conversation/files/settings/sidebar/terminal/i18n 拆 subpath exports，附 CSS 隔离方案（`package.json` exports、`docs/DEVELOPMENT.md`）。
- **多后端注册表**：`src/api/backend-registry/` 统一管理 local/remote/cloud 后端身份与切换，事件与设置层都按 active backend 分流（`event-service.api.ts:81` `getActiveBackend()`）。
- **MSW mock 全覆盖**：`VITE_MOCK_API=true` 用 mocks/ 里的 handler 跑纯前端开发与测试（`package.json` dev:mock）。

## 6. 构建与运行

来自 `package.json` scripts 与 README（Node >= 24，贡献者需 npm 11.12.1）：

```sh
npm ci                 # 重放 lockfile（贡献者）
npm run dev            # 完整本地栈：agent-server + automation(uvx) + Vite + ingress
npm run dev:minimal    # 仅 agent-server + Vite
npm run dev:mock       # MSW mock 模式前端
npm test               # vitest 单测（先 make-i18n）
npm run lint           # typecheck + eslint + prettier
npm run build          # SPA 构建；build:lib 库构建
npm run test:e2e:mock-llm        # Playwright mock-LLM E2E
npm run test:mutation  # Stryker 变异测试
npm install -g @openhands/agent-canvas && agent-canvas   # 终端用户一键起栈
OH_CONVERSATION_RUNTIME=docker agent-canvas             # 每对话独立容器
```

## 7. 测试与工程质量

- **规模**：仓库内 `*.test.ts(x)` 共 890 个（find 统计），单测树 `__tests__/` + hooks co-located 测试 + 5 套 Playwright 配置（默认/live/mock-llm/mock-llm-docker/bind-policy）。
- **质量门**（`.github/workflows/ci.yml`）：lint（typecheck+eslint+prettier）→ vitest → build app → build lib → `npm pack --dry-run` 验证包内容，多 OS 矩阵；另有 live E2E（凭据门控、fork PR 跳过）与 docker E2E 独立 workflow。
- **变异测试**：Stryker 集成 + incremental/diff 模式（`stryker.config.mjs`、`scripts/stryker-diff.mjs`），这在同类项目里少见。
- **架构守护**：`src/api/no-direct-agent-server-calls.test.ts` 强制所有 Agent Server 调用走适配层；translation completeness CI 检查语言包完整度。
- **AI 协作规范成熟**：`AGENTS.md` 定义 TDD 基线（AAA 结构、mock 服务而非 hook）、PR 必须含人类撰写的 `HUMAN:` 段、"验证技能与实现者隔离"规则（verify-* skill 不得随产品代码修改）——是高度 agent-native 的仓库治理样本。

## 8. 社区与维护状态

- **极度活跃**：最近 push 2026-10-30（gh api）；release-please 驱动，v1.30.0→v1.34.0 共 6 个 release 落在 2026-10-22 至 10-27 五天内（releases API），npm 包迭代极快。
- **社区规模**：约 590 贡献者、12.3k forks、1153 open issues（gh api 2026-10-31）；从 All-Hands 公司主导演进为多仓生态（sdk/automation/extensions/typescript-client），README 提供 Slack 社区与 `#proj-agent-canvas` 频道。
- CI 含 issue 查重/就绪检查、PR 描述校验（`.github/workflows/pr-description-check.yml`）、自动截图工件（pr-artifacts）等大量流程自动化。星数增长主要来自 OpenDevin/OpenHands 早期 agent 时代的品牌积累，当前仓库角色已变（未确认增长归因）。

## 9. 亮点与风险

**亮点**

1. 事件流的工程深度：seq 游标断线回放、双 socket 抖动错峰、握手看门狗、页面可见性策略，是生产级实时前端的教科书实现（`src/contexts/conversation-websocket-context.tsx`、`src/hooks/use-websocket.ts`）。
2. 单 CLI 一键起完整多服务栈，且版本/端口/兼容性基线集中管理在 `config/defaults.json`，显著降低自托管门槛。
3. 供应链安全意识领先：7 天依赖冷却期 + npm `--min-release-age` + 精确锁版本 + trusted publishing。
4. 多后端抽象（local/docker/VM/cloud + 任意 ACP agent）让前端与执行环境彻底解耦，agent 可插拔。
5. 面向 AI 协作者的仓库治理（AGENTS.md + .agents/skills + 验证隔离）本身即是"agent 开发 agent 平台"的实践范本。

**风险/局限**

1. 仓库身份漂移：星数承载的是旧 "AI 软件工程师" 单体的声誉，代码本体已变为前端控制台；看 agent 循环/沙箱实现需另去 software-agent-sdk，两仓间通过 `compatibility.minimumAgentServer` 做兼容基线管理，升级耦合脆弱（`AGENTS.md` 共享前端变更清单）。
2. 无沙箱模式下 agent 拥有完整主机访问权，安全边界高度依赖用户自觉与密钥纪律（README 两处显式 WARNING）。
3. 依赖全部精确钉死 + 7 天冷却，非安全修复的依赖更新偏慢；Node 24 硬门槛对旧环境不友好。
4. 前端单仓体量大（api 层约 60 个服务模块），新人理解成本高，行为路径矩阵（local/cloud × openhands/ACP × 标准/规划/委派会话 × 独立/嵌入）在 AGENTS.md 中也承认难以穷举测试。

## 10. 总结评价

OpenHands 仓库完成了一次教科书式的平台化转身：把当年拿下 9 万星的 "AI 软件工程师" 单体拆成 SDK（Python 执行层）+ Canvas（本仓库，控制层）+ automation（调度层）的多仓生态，自己专注做"任何 ACP agent 的统一驾驶舱"。代码质量与工程规范（事件流韧性、测试矩阵、变异测试、供应链安全、AI 协作治理）均属第一梯队，release 节奏近乎日更。它适合三类人：想自托管多 agent 工作流的开发者（直接用 `agent-canvas`）；研究实时 agent UI 状态管理的前端工程师（读 conversation-websocket-context 与 api 适配层）；以及研究"如何组织 agent 驱动开发流程"的团队（读 AGENTS.md 与 .agents/skills）。想读 agent 循环与 Docker 沙箱实现源码的，请移步 `OpenHands/software-agent-sdk` 仓库。
