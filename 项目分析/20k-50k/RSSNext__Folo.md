# RSSNext/Folo 项目分析

> 星数 39287（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：Folo（原名 Follow）是开源的 AI RSS 阅读器——把博客、新闻、Newsletter、播客、视频、社交动态统一进一条时间线，并叠加 AI 问答、摘要、翻译与定时简报，一套 TypeScript 代码跑遍 Web/iOS/Android/macOS/Windows/Linux。

| 项 | 值 |
| --- | --- |
| 名称 | RSSNext/Folo（默认分支 dev） |
| 星数 | 39,287（gh api 核实） |
| 语言 | TypeScript（2783 个 .ts/.tsx 文件，实测统计） |
| 许可证 | AGPL-3.0（`icons/mgc` 目录图标除外，见 README License 节） |
| 最近活跃 | 最近 push 2026-11-10（gh api）；最新 release Desktop v1.17.0 / Mobile v0.6.2（2026-11-02） |
| 官网 | https://app.folo.is ；README 定位语 "Folo is an open-source AI RSS reader"（README.md） |

## 2. 项目是什么，解决什么问题

传统 RSS 阅读器的痛点是"信息多而不精、格式单一、跨端割裂"。Folo 的答案分三层（README.md ✨ Features 与源码互证）：

1. **聚合一切内容形态**。支持 RSS/Atom/JSON feed，加上 RSSHub 数千条为无 feed 站点提供的路由（README "And More"；`apps/desktop/layer/renderer/src/modules/rsshub/`、`apps/mobile/src/modules/rsshub/`）。文章、社交帖、图片、视频、播客各有专门排版（`modules/entry-content`、`modules/player`），并内置 TTS 朗读。还有独立收件箱（每个账号一个 `@follow.re` 邮箱地址收 Newsletter，默认值见 `packages/internal/shared/src/env.common.ts` 的 `INBOXES_EMAIL`）。
2. **AI 直接长在阅读流里**。对单篇文章、整个 feed、文件夹或全部订阅提问（`modules/ai-chat`）；时间线一键生成当日摘要（Timeline Summary，会话 ID 前缀 `TIMELINE_SUMMARY`，见 `modules/ai-chat/services/index.ts`）；逐篇翻译与摘要（`packages/internal/store/src/modules/summary`、`modules/translation`）；AI Tasks 定时生成日报/周报投递到聊天或邮箱（`modules/ai-task`，`followApi.aiTask.*`）。AI 可挂 MCP 服务、支持自带 API Key（BYOK，见第 5 节）。
3. **六端同步**。订阅、已读、收藏在 Web/iOS/Android/桌面端实时同步，靠一套"类 Linear 同步引擎"的本地优先数据层（`packages/internal/store/src/sync/sync-engine.ts` 注释原文 "modelled on Linear's sync engine"）。

项目 2024-04-08 创建（gh api），原名 Follow，2024 年下半年曾登 GitHub 月度 Trending 第一，现已随产品线更名 Folo（改名证据见第 8 节）。README 自述为 App Store "Top 1 News App"，无广告、代码开源。

## 3. 技术栈与依赖

- **语言/运行时**：TypeScript 6.0.3（pnpm catalog 统一版本），Node ≥18（CLI）。React 19.2.8（workspace overrides 锁定，pnpm-workspace.yaml）。
- **桌面端**：Electron 44.4.3 + electron-vite 5 + electron-forge 打包（apps/desktop/package.json）；渲染层同一套代码可出 Web 版（`build:web`，部署到 Cloudflare，.github/workflows/deploy-cloudflare-desktop.yml）。
- **移动端**：Expo/React Native + NativewindCSS（apps/mobile，EAS 构建，OTA 热更走自建 `apps/ota`）。
- **状态管理**：Jotai（atoms）+ Zustand（复杂 store）+ TanStack Query（服务端状态）——根 AGENTS.md "Architecture quick reference" 明文约定。
- **本地数据库**：Drizzle ORM + SQLite 双实现：桌面/Web 用 wa-sqlite（WASM，IDBMirrorVFS 挂 IndexedDB，`packages/internal/database/src/db.desktop.ts`），移动端用 expo-sqlite（`db.rn.ts`），共享同一套 schema（`src/schemas/index.ts`，16 张表）。
- **AI**：Vercel AI SDK `ai` 7.0.109——客户端用 `HttpChatTransport` 流式对接服务端 `/ai/chat`（`modules/ai-chat/store/transport.ts`）；模型列表由服务端下发（`followApi.ai.config()`）。
- **网络/API**：自研 `@follow-app/client-sdk`（npm 包）封装 REST；认证用 better-auth 会话 Cookie（`lib/api-client.ts`）。
- **其他关键库**：fuse.js（本地模糊搜索）、@mozilla/readability+jsdom/linkedom+DOMPurify（全文提取与消毒，`packages/readability`）、TailwindCSS 4 + Apple UIKit 色板、motion/react 动效、Sentry/PostHog（遥测）、i18next 多语言。
- **构建编排**：pnpm workspaces + Turborepo（turbo.json），vitest 单测 + Playwright e2e。

## 4. 目录结构与代码组织

典型巨型 pnpm monorepo（workspace 定义见 pnpm-workspace.yaml）：

| 目录 | 职责 |
| --- | --- |
| apps/desktop | Electron 桌面应用，`layer/main` 主进程（轻量：IPC/updater/语言检测）、`layer/renderer` 渲染层（1.1k+ 文件，主战场），同一渲染层可构建纯 Web 版 |
| apps/mobile | Expo/React Native 移动端（840 个 TS 文件），另有 `web-app/html-renderer` 渲染文章 HTML |
| apps/ssr | 部署在 Cloudflare Workers 的 SSR/分享页服务（Fastify 兼容运行时，处理 OG meta 与外链预览） |
| apps/cli | `folocli` 命令行客户端（commander），可做自动化，随包附 `skill.md` 供 AI agent 使用 |
| apps/landing | 官网落地页（部署 Cloudflare） |
| apps/ota | 自建 OTA 更新服务（Hono on Workers，fzstd 解压增量包） |
| packages/internal | 端间共享核心：atoms/components/hooks/store（同步引擎+12 个领域模块）/database（Drizzle schema 与迁移）/logger/tracker/utils |
| packages/readability | 全文提取与 HTML 消毒的独立小包 |
| packages/configs | 共享 lint/ts 配置 |
| locales/ | i18n 文案（按 ai/app/common/errors/mobile/settings 等命名空间分目录） |
| store-assets/ | 应用商店素材与演示账号 |
| docs/ | 2026-09-22 批量依赖升级记录 + superpowers 子目录（agent 工作流材料） |

组织方式评述：按"端"拆 app、按"层"拆包；renderer 是最大的单体（36 个 modules 子目录：ai-chat、entry-column、entry-content、player、settings、spotlight、integration、wallet 等）。每个 app/package 自带 AGENTS.md 供 AI 协作（根 AGENTS.md 明确"就近优先"规则），仓库本身重度面向 AI 辅助开发。

## 5. 核心架构与关键实现

### 5.1 类 Linear 的本地优先同步引擎（最有含金量的部分）

`packages/internal/store/src/sync/sync-engine.ts` 开头注释直接写明设计来源：

> "Incremental synchronization of user-owned models, modelled on Linear's sync engine. The client keeps `lastSyncId`, the highest change-log id it applied… Pending local transactions keep precedence through the transaction queue's overlays."

客户端本地保存游标 `lastSyncId`：首次 bootstrap 从服务端取全量快照，之后只拉取变更日志增量；本地未完成的操作通过事务队列（`sync/transaction-queue.ts`）以 overlay 覆盖在本地状态上，保证离线可写、上线对账。领域模型拆在 `packages/internal/store/src/modules/`（entry/feed/subscription/list/inbox/collection/unread/summary/translation/user 等 12 个模块），与 SQLite 表一一对应。已读状态这类"易老化"数据定期用快照校准（同文件注释）。

### 5.2 订阅数据模型（Drizzle + SQLite）

`packages/internal/database/src/schemas/index.ts` 定义 16 张表，核心四张：

```ts
export const feedsTable = sqliteTable("feeds", {
  id: text("id").primaryKey(),
  url: text("url").notNull(),
  ...
  subscriptionCount: integer("subscription_count"),
  updatesPerWeek: integer("updates_per_week"),
})
export const subscriptionsTable = sqliteTable("subscriptions", {
  feedId: text("feed_id"),
  listId: text("list_id"),
  inboxId: text("inbox_id"),
  view: integer("view").notNull().$type<FeedViewType>(),
  isPrivate: integer("is_private", { mode: "boolean" }).notNull(),
  type: text("type").notNull().$type<"feed" | "list" | "inbox">(),
})
```

订阅统一为 `feed | list | inbox` 三态：普通源、人工策展的 List（可含 `fee`/`purchaseAmount`，即付费订阅）、Newsletter 收件箱；`view` 字段对应产品里的文章/社媒/图片/视频/播客五种视图（FeedViewType 来自 `@follow/constants`）。另有 entries/collections/unread/sync_meta/ai_chat/ai_chat_messages 等表，AI 聊天记录同样落本地库再与远端同步（`modules/ai-chat/services/index.ts` 中 `AIPersistService`）。

### 5.3 AI 能力层：服务端推理 + BYOK/MCP + 本地缓存

- 推理在服务端：聊天流走 `${API_URL}/ai/chat`（`modules/ai-chat/store/transport.ts`，Vercel AI SDK `HttpChatTransport` + SSE 解析，含流式超时保护 `utils/chat-stream-timeout`）；模型清单、默认模型由 `followApi.ai.config()` 下发（`hooks/useAIConfiguration.ts`），摘要标题走 `followClient.api.ai.summaryTitle`，AI 任务走 `followApi.aiTask.*`（`modules/ai-task/query.ts`）。
- **BYOK（自带 Key）**：`packages/internal/shared/src/settings/interface.ts` 定义 `ByokProviderName = "openai" | "google" | "vercel-ai-gateway" | "openrouter"`，用户 Key 存本地（`settings/defaults.ts` 默认关闭），"actual connections managed via server API"（interface.ts 注释）——即 Key 由服务端代持转发。
- **MCP**：`mcpEnabled/mcpServices` 设置项（interface.ts），可把外部工具接进 Folo AI。
- 摘要/翻译按 entryId × 语言缓存于本地 summary/translation store（`packages/internal/store/src/modules/summary/store.ts` 的 `Record<entryId, 语言 → SummaryData>`），生成状态机 `SummaryGeneratingStatus` 管理轮询。

### 5.4 多端形态与开放边界

- 同一 `packages/internal` 被桌面/Web/移动三端复用；桌面 renderer 既是 Electron UI 又是 Web 版（`apps/desktop/package.json` 同时有 `dev:electron` 与 `dev:web`，CI 分别构建部署）。
- RSS 抓取、解析、AI 推理都在闭源服务端 `api.folo.is`（`packages/internal/shared/src/env.common.ts`：`API_URL: "https://api.folo.is"`，开发环境 api.dev.folo.is）；客户端只通过 `@follow-app/client-sdk` 消费 REST。全文阅读不依赖服务端的部分由 `packages/readability`（@mozilla/readability + linkedom/jsdom + DOMPurify 消毒，`src/sanitize.ts` 带单测）在本地补齐。
- 这是典型 open-core：客户端（含 CLI、SSR 分享页、OTA 服务）开源，核心爬取/AI 服务不开源。

## 6. 构建与运行

来自根 package.json、AGENTS.md 与各 app package.json：

```bash
pnpm install                      # postinstall 自动构建 packages + link-cf-build-tools
pnpm typecheck                    # turbo 全仓类型检查（提交前第 1 道门）
pnpm lint                         # eslint + tsslint（自定义 TS lint，tsslint.config.ts）
pnpm test                         # 递归跑各包 vitest
pnpm build:web                    # 桌面渲染层出 Web 版
cd apps/desktop && pnpm dev:web   # 推荐：浏览器渲染层开发
cd apps/desktop && pnpm dev:electron
cd apps/mobile && pnpm dev        # Expo（ios/android 可指定）
cd apps/ssr && pnpm dev
cd apps/desktop && pnpm e2e       # Playwright e2e（web / electron 两个 project）
```

发布链路：`changelogithub.config.ts` 生成 changelog，`apps/desktop/scripts/apply-release-config.ts` + release.json/release-plan.json 控制版本；Windows 用 SignPath 签名、macOS/iOS 走 Apple 公证，产物带 GitHub artifact attestations（README Code signing policy 节）。

## 7. 测试与工程质量

- 单测：259 个 `*.test.ts(x)`（实测统计），Vitest，与源码同目录放置（AGENTS.md 约定）；覆盖同步引擎（sync-engine.test.ts、transaction-queue.test.ts、account.test.ts）、store 模块、CLI 全部命令、readability 消毒等核心逻辑。
- e2e：Playwright，桌面端分 web/electron 两个 project，支持 `FOLO_E2E_PROFILE=prod` 对生产档跑（apps/desktop/e2e/）。
- CI：16 条 workflow（.github/workflows/）——按端拆分的 build-desktop/build-android/build-ios/build-web，三条 Cloudflare 部署（desktop-web/landing/ssr），lint、pr-title-check、issue-labeler、similar-issues、translator（翻译协作文案）、publish-ota（Expo OTA 发布）。
- 质量基础设施成熟度少见：tsslint 自研 TS lint 规则、pnpm `minimumReleaseAge` 式依赖冷静期清单（pnpm-workspace.yaml 尾部注释 "Exact versions in the reviewed dependency graph…"）、6 个补丁（patches/，如对 @mozilla/readability 的定制）、docs/ 里 16 份按依赖域拆分的升级记录（2026-09-22）。
- 观察：质量门槛（typecheck→lint→test）写入 AGENTS.md 且 CI 强制；lint-staged 甚至对移动端源码变更自动递增 build id（根 package.json）。

## 8. 社区与维护状态

- **活跃度**：gh api 显示最近 push 2026-11-10（任务派发时点仍在推送）；release 节奏约每 2 周双端各发一版（Desktop v1.16.0 10-16 → v1.17.0 11-02，Mobile v0.6.0/v0.6.1/v0.6.2 同期）。
- **贡献者**：161 人（gh api contributors 分页计算）；头部为 Innei（2842 次提交）、DIYgod（2809）、hyoban（990）、lawvs（375）、kovsu（125）。DIYgod 即 RSSHub 作者——Folo 官方把 RSSHub 路由作为内容源生态（README 明确提及），两者同属中文开源社区；Innei、hyoban 等亦为活跃中文开发者，属于中文团队主导、国际化协作的项目（未在本仓库内找到公司主体文件，此判断基于贡献者构成与 RSSHub 关联，主体信息未确认）。
- **改名历史**：仓库原名 Follow，现已更名 Folo。证据：根 package.json `repository.url` 仍指向 `github.com/RSSNext/follow`；应用商店包名 `is.follow`（README Google Play 链接）；RSSNext 组织下残留 follow-contracts、follow.is、follow-email-worker、follow-gh-bot、follow-image-proxy 等旧仓（gh api orgs/RSSNext/repos）；收件箱域名仍是 `@follow.re`、env 注入变量名 `__followEnv`。
- **社区渠道**：Discord、X（@folo_is）；第三方打包生态完善——AUR、Flatpak、Nix（已进 nixpkgs）、Homebrew cask、Scoop 均有社区维护的安装方式（README）。
- **商业化线索**：renderer 有 `modules/plan`（升级订阅）、`modules/power` + `modules/wallet`（Power 积分与交易记录）、List 的 `fee/purchaseAmount` 字段——存在付费订阅与内容付费机制（源码证据，具体商业模式未在仓库内展开）。

## 9. 亮点与风险

**亮点**

1. 本地优先同步引擎对标 Linear（`packages/internal/store/src/sync/sync-engine.ts`）：游标增量 + 事务 overlay + 本地 SQLite，六端体验一致，是仓库里工程质量最高的模块（附完整单测）。
2. 一套 TypeScript 代码复用到 6 个平台：桌面（Electron）、移动（RN）、Web、CLI、SSR、落地页共享 packages/internal，边界切分清晰。
3. AI 功能设计务实：服务端统一推理（模型清单下发），但开放 BYOK（OpenAI/Google/Vercel Gateway/OpenRouter）与 MCP，摘要/翻译按语言粒度缓存，AI Task 可定时投递。
4. 工程与发布体系工业级：tsslint、Playwright 双 project e2e、按端 CI、OTA 热更、签名公证与 artifact attestations，另配套面向 AI agent 的 AGENTS.md 体系与 CLI skill.md。
5. 内容源生态借力 RSSHub，覆盖大量无 feed 站点，形成 RSSHub→Folo 的上下游闭环。

**风险/局限**

1. open-core 边界偏向闭源：RSS 抓取、解析、AI 推理、账号体系全在 api.folo.is（`packages/internal/shared/src/env.common.ts`），自托管无法复刻完整产品，重度依赖官方服务可用性。
2. 必须登录账号使用（401 拦截强制弹登录框，`apps/desktop/layer/renderer/src/lib/api-client.ts`），无纯本地模式。
3. monorepo 体量大（2783 个 TS 文件、pnpm workspace 强 overrides/catalog 锁版本），外部贡献者上手成本高；pnpm-workspace.yaml 中的依赖冷静期与补丁清单也说明依赖维护压力大。
4. AGPL-3.0 对商用集成不友好；`icons/mgc` 图标单独限制再分发（LICENSE 例外条款）。

## 10. 总结评价

Folo 是近两年 RSS 阅读器赛道少见的"产品、工程、AI 三线并进"的项目：产品上用 AI 摘要/翻译/问答/定时简报重做了阅读器的信息消费流程并拿下 App Store 新闻类第一；工程上用一套 TS 代码打通六端、以类 Linear 的本地优先同步引擎保证跨端一致性；生态上背靠 RSSHub 的中文开源社区。它的主要代价是核心服务闭源带来的自托管缺失与账号绑定。适合普通用户当作跨端 AI 阅读器直接使用；适合前端/全栈工程师研读其同步引擎（packages/internal/store/src/sync）、多端共享层设计（packages/internal）与 monorepo 工程化实践——这是仓库最有学习价值的部分；想二次开发自建服务的人则应先认清客户端开源、服务端托管的边界。
