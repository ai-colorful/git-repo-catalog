# novuhq/novu 项目分析

> 星数 38500（快照 2026-10-24，gh api 实测 40336 @ 2026-11-14）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：开源通知/通信基础设施——用一个 API、一套统一会话模型，把产品与 AI Agent 接入 Inbox（应用内）、Email、SMS、Push、Chat（Slack/Teams/Telegram/WhatsApp 等）所有用户所在渠道。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | novuhq/novu | gh api repos/novuhq/novu |
| 星数 | 40336（2026-11-14 实测，任务快照 38500） | gh api |
| 语言 | TypeScript（Nx + pnpm monorepo） | `package.json` |
| 许可证 | 双许可：社区部分 MIT（`LICENSE-MIT`），enterprise 部分专有（`LICENSE-ENTERPRISE`、`EE-PACKAGES-LICENSE`）；gh api 显示 NOASSERTION | 仓库根目录 |
| 默认分支 | `next` | gh api |
| 最近活跃 | 最近 push 2026-11-13（分析前一天，日均多个 commit） | gh api + `gh api commits` |
| 规模 | .ts 7939 个 + .tsx 1831 个，合计约 125 万行（含测试）；测试文件 858 个 spec + 264 个 e2e | 本地统计 |

## 2. 项目是什么，解决什么问题

Novu 要解决的问题是：任何产品（以及现在的 AI Agent）都需要把消息送达用户，而送达意味着对接几十家渠道服务商（SendGrid、Twilio、FCM……）、维护消息模板与用户订阅偏好、处理去重/限流/重试。Novu 把这些抽象为一层基础设施：开发者只需定义"工作流（workflow）+ 触发事件"，剩余的渠道投递、状态回执、偏好管理、Inbox UI 全部由 Novu 承担（`README.md` "Why Novu?" 节）。

产品形态有两种用法（`README.md`）：一是面向产品的通知基础设施——通过一个 API 发送 In-App/Email/SMS/Push/Chat 消息，支持工作流、摘要聚合（digest）、可嵌入的 `<Inbox />` React 组件；二是 Agent 通信基础设施（ACI）——把已构建的任意 Agent 经统一会话模型接入 Slack、Teams、Telegram、WhatsApp、Email 等渠道。README 明确了战略转向：软件正在变得"会对话"，从单向广播转向可继续交互的线程。

工作流分两种编写方式：UI 内拖拽编辑（dashboard 应用，内容模板用 Handlebars 占位）或代码优先——`@novu/framework` 允许用 TypeScript 函数定义多渠道工作流，并可作为 bridge（桥接端点）在用户自己的服务器上执行步骤（`packages/framework/README.md`、`apps/api/src/app/events/usecases/parse-event-request/parse-event-request.usecase.ts` 中 bridge URL 的持久化注释）。

运行形态为 Novu 云（SaaS）或自托管：社区版 compose 提供 api/worker/ws/dashboard 四个服务容器加 Redis/Mongo（`docker/community/docker-compose.yml`）；企业能力（计费、多租户翻译、agent 运行时等）放在 `enterprise/` 目录，源码以符号链接指向私有 `.source/` 子模块，tarball 中不存在，属于云/付费边界。

## 3. 技术栈与依赖

- 后端：NestJS 11（`apps/api/package.json`：`@nestjs/core` 11.1.27），MongoDB 持久化 + Redis 队列
- 任务队列：BullMQ（封装为 `BullMqService`，`apps/worker/src/app/workflow/services/standard.worker.ts`），云上可切换 SQS（`SqsService`），本地/测试可退化为内存 Provider（`WorkflowInMemoryProviderService`）
- 实时推送：Socket.IO + Redis adapter + admin-ui（`apps/ws/package.json`）
- 模板引擎：Handlebars（`apps/api/package.json`、`libs/application-generic/src/services/content.service.ts`）
- 前端：React + Vite（`apps/dashboard`），Inbox 组件体系 `packages/react`、`packages/js`、`packages/nextjs`、`packages/react-native`
- 邮件富文本编辑器：`libs/maily-core`（基于 Tiptap，源自 maily.to 开源项目）
- 工程化：Nx monorepo + pnpm workspace（`pnpm-workspace.yaml`），Biome 做 lint/format（根 `package.json`、`biome.json`），husky + conventional commit
- AI 相关：`@novu/framework` 支持 Vercel AI SDK 与 LangChain 桥接（`packages/framework/src/ai-sdk/`、`src/langchain/`）

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `apps/api` | @novu/api-service：REST API（触发事件、工作流/订阅者/集成/偏好管理、Inbox API），NestJS 模块按域划分（`src/app/` 下 40+ 模块：events、inbox、integrations、preferences、subscribers-v2、channel-connections、agents 等） |
| `apps/worker` | @novu/worker：BullMQ 消费端，工作流执行引擎（`src/app/workflow/`） |
| `apps/ws` | Socket.IO 服务：Inbox 实时消息推送 |
| `apps/webhook` | 入站 webhook 接收服务（chat 渠道交互回调等） |
| `apps/inbound-mail` | 入站邮件解析（含 `python/` 子目录，回复邮件处理策略在 worker 侧 `usecases/inbound-email-parse`） |
| `apps/dashboard` | 管理后台前端（工作流编辑器、订阅者、Topics、Contexts、Agent 配置等页面，`src/pages/`） |
| `apps/human-dashboard` | 面向"人工"侧的新仪表盘（React Canary，`src/app/`） |
| `apps/mcp-server` / `apps/agent-inbox-mcp` | MCP 服务器：让 AI Agent 通过 MCP 协议操作 Novu/Inbox（`src/tools/`） |
| `libs/dal` | MongoDB 仓储层（`src/repositories/`） |
| `libs/application-generic` | 通用服务：BullMqService/SqsService、ContentService 模板编译、健康检查、加密、限流等 |
| `libs/internal-sdk` | 后端内部 SDK（react-query、models） |
| `libs/maily-core` / `maily-render` | 邮件编辑器核心与渲染 |
| `packages/providers` | 全部渠道 Provider 实现（email/sms/push/chat/tool） |
| `packages/framework` | @novu/framework：代码优先工作流 SDK + bridge agent + `npx novu connect` |
| `packages/js|react|nextjs|react-native|add-inbox` | 面向用户的 Inbox 组件与客户端 SDK |
| `packages/agent-toolkit`、`agent-event-protocol`、`agentinbox`、`chat-adapter-*` | Agent 通信生态：工具集、事件协议、聊天适配器（email/photon-imessage/sendblue/web-chat） |
| `packages/novu`、`stateless`、`shared` | 旧版 Node SDK、无状态渲染、全仓共享类型/常量 |
| `enterprise/` | 企业版包（agents/ai/api/auth/billing/shared-services/translation），src 为指向私有 `.source/` 的符号链接，tarball 中内容缺失（专有代码） |
| `docker/community`、`docker/local` | 自托管 compose 与本地开发环境 |

组织方式评述：典型的 Nx 多应用多包 monorepo，应用间通过 `libs/` 共享层与 `packages/shared` 类型解耦；近一年明显向"Agent 通信"方向扩张（agent-* 目录群与 `.agents/skills/` 仓库技能、`skills-lock.json`），同时保留了原通知基础设施主体。

## 5. 核心架构与关键实现

### 5.1 事件触发 → 队列 → 执行的流水线

API 侧 `ParseEventRequest`（`apps/api/src/app/events/usecases/parse-event-request/parse-event-request.usecase.ts`）校验工作流存在后调用 `dispatchEventToWorkflowQueue`，经 `WorkflowQueueService.add` 写入 BullMQ（组 ID 为组织 ID），worker 侧 `StandardWorker` 消费。worker 继承自 `application-generic` 的 `StandardWorkerService`，可选 SQS 承载（`apps/worker/src/app/workflow/services/standard.worker.ts`）：

```ts
export class StandardWorker extends StandardWorkerService {
  constructor(...) {
    super(new BullMqService(workflowInMemoryProviderService), sqsService, logger);
    this.initWorker(this.getWorkerProcessor(), this.getWorkerOptions(), true);
    this.bullMqWorker?.on('failed', async (job, error) => { await this.jobHasFailed(job, error); });
```

执行引擎入口 `RunJob`（`apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts`，千行级）按步骤类型分派；`send-message/` 下按渠道各有 usecase（in-app/email/sms/chat/push/tool），并配有 digest（`digest.usecase.ts`、regular/backoff 两种聚合窗口）、throttle、delay、channel-endpoint-resolution 子模块。`queue-next-job` usecase 负责步骤间的链式推进。

### 5.2 步骤类型与渠道 Provider 层

步骤类型全集（`packages/shared/src/types/channel.ts`）：`in_app / email / sms / chat / push / tool / digest / trigger / delay / throttle / custom / http_request`——除投递类步骤外还有流程控制类步骤，这就是"通知编排"的核心表达力。

Provider 层按通道分目录（`packages/providers/src/lib/`，目录数统计，排除 webhook 占位）：

- Email 19 个：sendgrid、ses、mailgun、postmark、resend、brevo、braze、infobip、mailjet、mandrill、mailersend、mailtrap、netcore、nodemailer、outlook365、plunk、sparkpost、emailjs、anypost
- SMS 38 个：twilio、nexmo、sns、telnyx、plivo、messagebird、gupshup、bandwidth、sinch、kannel 等
- Push 9 个：fcm、apns、expo、one-signal、pusher-beams、pushpad、appio 等
- Chat 20 个：slack、msTeams、discord、telegram、whatsapp-business、line、google-chat、mattermost、rocket-chat、webex、zulip、ryver、getstream、sendblue、photon-imessage 等
- 另有 `tool/`（grafana、opsgenie、pagerduty、tool-webhook）

合计约 86 个 Provider，全部实现统一基类接口（`packages/providers/src/base.provider.ts`）。

### 5.3 内容模板系统（Handlebars + 变量抽取）

`ContentService`（`libs/application-generic/src/services/content.service.ts`）是模板中枢：用 `Handlebars.parseWithoutProcessing` 解析 AST 抽取 `{{变量}}`，区分保留变量（`subscriber`/`steps` 等）与自定义变量，并从步骤 filters 中提取子变量，供 dashboard 做变量 schema 提示：

```ts
extractVariables(content: string): IMustacheVariable[] {
  const ast: hbs.AST.Program = Handlebars.parseWithoutProcessing(content);
  return getTemplateVariables(ast.body);
}
```

变量类型定义（String/Array/Boolean 等及默认值）在 `packages/shared/src/types/channel.ts`（`TemplateVariableTypeEnum`）。代码优先路径则由 `packages/framework` 直接在用户代码里渲染 JSX/函数结果，经 bridge 回传给 Novu 执行器（`apps/worker/src/app/workflow/usecases/execute-bridge-job/`）。

### 5.4 订阅偏好与 Inbox

偏好管理按层级组织：API 提供 `inbox/usecases/update-preferences`、`bulk-update-preferences`、`get-inbox-preferences`（`apps/api/src/app/inbox/`），订阅者级偏好 `subscribers/usecases/get-preferences-by-level`、`subscriptions/usecases/create-subscription-preferences`。Inbox 实时性由 `apps/ws`（Socket.IO + Redis adapter）推送，前端可嵌入组件体系在 `packages/react`/`packages/js`。Agent 侧通过 `apps/agent-inbox-mcp`（MCP 工具：inbox.ts、card.ts、interactions.ts 等）以工具调用方式读写 Inbox。

## 6. 构建与运行

```bash
# 安装与构建（根 package.json）
pnpm install
pnpm build              # nx run-many --target=build --all
pnpm build:api          # nx build @novu/api-service（nest build）
pnpm build:v2           # api/worker/ws/dashboard + packages

# 本地开发（scripts/dev-environment-setup.sh、mprocs 多进程）
pnpm dev:portless       # node scripts/mprocs-dev.mjs

# 测试（nx + jest/jest.config.js，包内 vitest 如 providers）
pnpm nx test @novu/worker

# 自托管（docker/community/docker-compose.yml）
cd docker/community && docker compose up -d
# 服务：redis、mongo 8.0、ghcr.io/novuhq/novu/{api,worker,ws,dashboard}:3.19.0

# SDK/CLI
npx novu@latest connect   # 连接 agent 到渠道（packages/framework README）
```

## 7. 测试与工程质量

- 测试规模：858 个 `*.spec.ts` 单元测试 + 264 个 `*.e2e.ts`（根目录统计），核心模块测试密度高——如 `send-message.*.spec.ts` 每渠道一份、`run-job.usecase.spec.ts`
- 分层：单元（jest，根 `jest.config.js`）+ 每服务 e2e（CI 中 `reusable-api-e2e.yml`、`reusable-worker-e2e.yml`、`reusable-ws-e2e.yml`、`reusable-webhook-e2e.yml`、`reusable-dashboard-e2e.yml`、`reusable-inbound-mail-e2e.yml` 等独立可复用工作流）
- CI 共 45+ 个 workflow 文件（`.github/workflows/`）：on-pr 主流水、CodeQL 安全扫描、CodeQL/workflow-security-lint、conventional-commit 校验、submodule 同步检查、preview 包发布、灰度/回滚（`rollback.yml`）
- 代码质量：Biome 统一 lint/format，hygen 代码生成器（`g:module`/`g:usecase`）保证 NestJS 模块结构一致性；commit 全部 conventional commits 并挂 Linear 工单号（NV-xxxx，见 `gh api commits`）

## 8. 社区与维护状态

- 活跃度：极活跃。最近 push 2026-11-13；最近 5 个 commit 全在一天内（安全升级 better-auth 1.7.19、worker 令牌校验、SDK DTO 修复），全部走 PR + Linear 工单（gh api commits）
- 贡献者：contributors 分页 377 页（每页 25，约 9400 人，含 issue/小修补贡献者），社区规模大（gh api contributors Link header）
- 发版节奏：releases 第一页即 100 个（2025-08-27 ~ 2026-09-28，约每周 2 个）；自托管镜像版本 v3.19.0（2026-08-07 发布），tag 序列 v3.11.0→v3.19.0 平滑推进；npm 包（@novu/framework v2.14.0）独立发版（gh api releases/tags）
- 商业化：Novu 云 SaaS + 企业版双许可；`linear-release-community.yml` / `linear-release-enterprise.yml` 区分社区与企业发版流水线（`.github/workflows/`）

## 9. 亮点与风险

亮点：

1. Provider 矩阵业界罕见：Email/SMS/Push/Chat 合计约 86 个渠道适配器统一接口化（`packages/providers/src/base.provider.ts`），换服务商只改配置
2. 编排表达力完整：12 种步骤类型覆盖投递 + digest/delay/throttle/条件过滤/custom/http_request，双模式（UI 拖拽与代码优先 bridge）共享同一执行引擎
3. 工程化成熟：Nx 影响面构建、per-service e2e 可复用 CI、Biome、代码生成器、conventional commit + 工单闭环
4. 测试覆盖扎实：1100+ 测试文件，核心执行引擎每渠道/每环节均有 spec
5. 顺应 Agent 时代快速演进：MCP 服务器、agent-toolkit、agent-event-protocol、bridge agent（AI SDK/LangChain）一整套 ACI 生态已落地

风险/局限：

1. 双许可边界复杂：`enterprise/` 包 src 在公开 tarball 中为空壳（私有 submodule `.source/`），自托管者拿不到完整企业能力（billing/translation/agents 运行时），需要书面授权（`EE-PACKAGES-LICENSE` 明确"Approval Required"）
2. 仓库处于大迁移期：`apps/web` 已被 `dashboard`/`human-dashboard` 取代、`packages/novu`/`stateless` 为遗留 SDK，新旧并存增加理解与升级成本
3. 执行引擎单文件体量大（`run-job.usecase.ts` 逾千行、分支密集），二次开发需要较强上下文
4. 自托管依赖面广（Mongo 8 + Redis + 4 个服务容器），小团队运维成本不低；云版功能与自托管存在差距（keyless、domain-connect 等模块面向云）

## 10. 总结评价

Novu 是"通知即服务"赛道的开源标杆，且已完成从多渠道通知工具向 Agent 通信基础设施（ACI）的战略升级：一个 API + 统一会话模型 + 86 个渠道 Provider + 12 类编排步骤，把产品与 AI Agent 同时接入所有用户触点。代码组织是教科书级的 TypeScript monorepo（NestJS + BullMQ + Socket.IO + React），测试与 CI 体系完整，维护节奏约为每周两次发版、次日安全响应。适合需要多渠道消息触达的中大型产品团队自托管或直接使用 Novu 云；想研究通知编排、任务队列分层、Provider 抽象的工程师，重点读 `apps/worker/src/app/workflow/`（执行引擎）与 `packages/providers/src/lib/`（渠道矩阵），会有很高收获。
