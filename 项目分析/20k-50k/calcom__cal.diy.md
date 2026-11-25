# calcom/cal.diy 项目分析

> 星数 49226（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：Cal.diy 是 Cal.com 剥离全部商业/企业代码后的 100% MIT 开源社区版——一个自托管的日程预约（scheduling）平台，可理解为"开源版 Calendly"。

| 事实 | 内容 | 出处 |
| --- | --- | --- |
| 名称 | calcom/cal.diy（原 calcom/cal.com 更名而来，gh api 确认真实存在） | gh api repos/calcom/cal.diy，2026-11-04 |
| 星数 | 49226（快照）/ 49680（分析日实测） | gh api |
| 主语言 | TypeScript（strict），monorepo 共约 5015 个 .ts/.tsx 文件 | `apps/`、`packages/` 实测统计 |
| 许可证 | MIT（版权行 "Copyright (c) 2020-present Cal.com, Inc."） | `LICENSE` |
| 默认分支 / 最近推送 | main / 2026-11-03（分析前一天） | gh api `pushed_at` |
| 最新 release | v6.2.0（2026-03-01 发布），Changesets 管理版本 | gh api releases、`.github/workflows/changesets.yml` |
| 形态 | Yarn 4 workspaces + Turborepo monorepo，3 apps + 约 20 packages | `package.json`、`turbo.json` |

## 2. 项目是什么，解决什么问题

Cal.diy 解决"多人跨日历的预约排期"问题：用户在平台上定义事件类型（EventType，如 30 分钟例会），平台聚合各人绑定的 Google/Microsoft/CalDAV 等日历忙闲、个人可用时段（schedules）、请假（OOO）、假期（holidays）与时差行程（travel schedules），向预约者（booker）开放可约时段；预约完成后通过 EventManager 写入日历、发邮件/短信、触发 webhook，并可挂载视频会议（Daily.co 等）与支付类 App（`packages/features/`、`packages/app-store/`）。

它与主产品 Cal.com 的关系（`README.md` 第 1-52 行）写得非常直白：Cal.diy 是 Cal.com 的社区 fork，**移除了全部企业版功能**——Teams、Organizations、Insights、Workflows、SSO/SAML 等，不需要 license key，无 Open Core 拆分；定位是"个人/自托管、非生产用途"，商业场景请用 Cal.com 官方托管版。README 顶部专门放了 WARNING 提示自托管需要数据库与运维能力。`packages/features/` 下实测确认已无 `ee/` 目录，与声明一致。

产品形态上是三端一体的系统（`apps/` 目录）：`apps/web` 是面向用户的 Next.js 主应用（管理端 + 面向预约者的 booker 页面），`apps/api` 下的 v2 是给平台/嵌入式客户用的 NestJS REST API（`apps/api/v2/src/modules/` 约 30 个模块），`apps/docs` 是文档站。`packages/platform/atoms` 把核心能力（如 availability）抽成可跨框架复用的"原子"，供第三方在自己的产品里嵌入 Cal 能力。

社区协作面貌也值得一提：仓库根目录 `AGENTS.md`（9KB）是一份写给 AI 编码代理的完整工程规范——强制 Prisma `select` 而非 `include`、禁用 `as any`、PR 限 500 行/10 文件、错误类选型（非 tRPC 用 `ErrorWithCode`）、可选的 spec 驱动开发流程（`SPEC-WORKFLOW.md`），并有 `agents/rules/`、`agents/commands.md`、`agents/knowledge-base.md` 配套，属于"AI-agent 优先"的仓库工程范式样本。

## 3. 技术栈与依赖

| 层 | 技术 | 证据 |
| --- | --- | --- |
| 框架 | Next.js（Pages Router 与 App Router 混用：`apps/web/pages/` + `apps/web/app/`），React 19.2.x | `apps/web/next.config.ts`、根 `package.json` resolutions |
| API | tRPC（type-safe RPC，routers 按域拆分）+ NestJS（apps/api/v2） | `packages/trpc/server/routers/`、`apps/api/v2/src/modules/` |
| ORM/数据库 | Prisma 6.16（PostgreSQL），100 个 model，schema 2851 行；另用 Kysely 做部分查询 | `packages/prisma/schema.prisma`、`packages/kysely/` |
| 校验/认证 | Zod（所有 tRPC schema 与 booking 表单 schema）、next-auth | `packages/trpc/**/*.schema.ts` |
| 样式/组件 | Tailwind CSS，自维护 `packages/ui` 与 `packages/coss-ui` | `packages/ui/` |
| 任务/定时 | trigger.dev 后台任务（提醒、短信、webhook），另有 10+ 条 GitHub cron workflow | `packages/features/trigger.config.ts`、`.github/workflows/cron-*.yml` |
| 可观测 | Sentry（`apps/web/sentry.*.config.ts`）、instrumentation、Checkly 生产拨测 | `apps/web/instrumentation.ts`、`checkly.config.ts`、`__checks__/` |
| 构建/质量 | Turborepo 2.7、Yarn 4.12、Biome 2.3、Changesets、husky+lint-staged | 根 `package.json` devDependencies |
| 测试 | Vitest 4（16 个 workspace 项目）+ Playwright 1.57（8 个 project）+ Prismock | `vitest.workspace.ts`、`playwright.config.ts` |
| i18n | next-i18next（打补丁），44 种语言 | `packages/i18n/locales/` 实测 44 个目录 |

依赖治理痕迹很重：根 `package.json` 有 120+ 条 `resolutions` 锁定传递依赖的安全版本（axios、tar、yaml、hono 等大量安全补丁式升级），并用 `patch:` 维护 dayjs、libphonenumber-js 等本地补丁（`.yarn/patches/`）。

## 4. 目录结构与代码组织

顶层（`ls` 实测）：

| 目录 | 职责 |
| --- | --- |
| `apps/web` | 主 Next.js 应用：管理端 + 预约页（booker），Pages/App Router 并存，`modules/` 约 30 个业务视图模块，`pages/api/book/` 是公开预约 API 入口 |
| `apps/api` | API v1 入口（`index.js`）+ v2 NestJS 应用（`v2/src/modules/` 约 30 模块，按日期版本化如 `slots-2024-04-15`） |
| `apps/docs` | 文档站（Next.js + MDX） |
| `packages/features` | 业务特性层，约 60 个特性模块（bookings、slots、schedules、availability、busyTimes、embed、webhooks、flags、holidays、ooo、bot-detection…），内部按 repository/service/DI 分层 |
| `packages/trpc` | tRPC 路由层（`server/routers/` 下 viewer/publicViewer/loggedInViewer 三组） |
| `packages/prisma` | schema、迁移、seed、zod-utils（100 个 model） |
| `packages/app-store` | 153 个条目，约 120 个第三方集成（日历/视频/CRM/支付/分析），`*_generated.ts` 由 app-store-cli 生成 |
| `packages/embeds` | embed 体系：embed-core（vanilla JS）、embed-snippet、embed-react |
| `packages/lib`、`ui`、`emails`、`i18n` | 共享工具、组件、邮件模板、多语言 |
| `packages/platform` | atoms/libraries：给 API v2 与外部平台复用的能力层 |
| `packages/features/di` | 手写 DI 容器（tokens/modules/containers） |
| `.changeset`、`.github`（50 个 workflow）、`specs/`、`agents/` | 发布、CI、规格文档、AI 协作规则 |

组织方式评述：典型的大型 Turborepo。分层哲学在 `AGENTS.md`「Project Structure」与代码中可验证——**router（tRPC/API）→ Service（业务）→ Repository（数据访问）**，且业务逻辑禁入 repository（AGENTS.md 明文规定）。`packages/features` 是实际重心：从旧版 Cal.com 的单体 core 演进为按特性拆分 + 手写 DI 注入（如 `packages/features/di/modules/AvailableSlots.ts` 给 AvailableSlotsService 注入 12 个依赖），这是读懂本仓库的关键。

## 5. 核心架构与关键实现

### 5.1 可约时段计算（调度核心）

查询链：tRPC `viewer.slots.getSchedule` → `packages/trpc/server/routers/viewer/slots/getSchedule.handler.ts`（仅 8 行，委托 DI 容器）→ `AvailableSlotsService._getAvailableSlots`（`packages/trpc/server/routers/viewer/slots/util.ts`，1435 行）。该服务编排：动态组预约事件类型（`_getDynamicEventType`）、slot 预占与过期清理（`_getReservedSlotsAndCleanupExpired`，配合 `reserveSlot.handler.ts`）、预订限制忙时（`_getBusyTimesFromLimitsForUsers`）、OOO 折算（`_getOOODates`）、主持人可用性聚合（`calculateHostsAndAvailabilities`）。真正的忙闲聚合在 `packages/features/availability/lib/getUserAvailability.ts`：同时吃进日历 credential（`getCalendar`）、预订限制（`getBusyTimesFromLimits`）、OOO、假期服务、时差行程（TravelSchedule）、工作时段 date-ranges（`packages/features/schedules/lib/date-ranges` 的 `buildDateRanges`/`subtract`），是全仓库语义最密集的文件之一（其 import 列表即可佐证）。

```ts
// packages/trpc/server/routers/viewer/slots/getSchedule.handler.ts
export const getScheduleHandler = async ({ ctx, input }: GetScheduleOptions) => {
  const availableSlotsService = getAvailableSlotsService();
  return await availableSlotsService.getAvailableSlots({ ctx, input });
};
```

### 5.2 预约创建流水线

公开入口 `POST /api/book/event`（`apps/web/pages/api/book/event.ts`）：Cloudflare Turnstile 校验 → 机器人检测（`packages/features/bot-detection`）→ 按 IP 哈希限流（`checkRateLimitAndThrowError` + `piiHasher`）→ `RegularBookingService.createBooking`。编排器 `packages/features/bookings/lib/service/RegularBookingService.ts`（2693 行，DI 组装）串起 `lib/handleNewBooking/` 下拆分后的步骤：`getEventType` → `loadAndValidateUsers` → `ensureAvailableUsers`（重查可用性防并发）→ `createBooking`（写库）→ `getRequiresConfirmationFlags`（需审批则转 pending）→ EventManager 落日历/发邮件 → webhook。衍生能力同层：周期预约 `pages/api/book/recurring-event.ts`、座位制 `lib/handleSeats/`（座位合并/迁移）、改期关联回原预约（`originalRescheduledBookingUtils.ts`）。旧单体 `handleNewBooking.ts` 已拆成 20 余个 ≤300 行的单元，测试按场景组织（`handleNewBooking/test/` 下 fresh-booking、reschedule、spam-booking、date-overrides 等 18 个场景文件）。

### 5.3 嵌入式 embed 体系

`packages/embeds/README.md` 与多张 mermaid 图（`embed-message-protocol.mermaid`、`modal-embed-lifecycle.mermaid` 等）完整描述了这套对外 SDK：`embed-core/src/embed.ts`（1713 行，vanilla JS）注册自定义元素 `cal-modal-box`/`cal-floating-button`/`cal-inline`，提供 `Cal("inline"|"modal"|"floatingButton"|"prerender")` API；父子页通过 iframe 消息协议通信（`embed-iframe.ts` 788 行），事件带 namespace 隔离多实例（`sdk-action-manager.ts`）；iframe 未就绪时指令进入队列（InstructionQueue）。性能设计是亮点：区分 preload（仅预热静态资源）与 prerender（复用已渲染 iframe 直接预约），README 给出 4 条明确的 iframe 复用/重载判定规则与骨架屏支持矩阵。发布走 `yarn publish-embed` 到 npm `@calcom/embed-*`（根 `package.json`），`embed-react/src/Cal.tsx` 提供 React 包装。

### 5.4 集成生态与生成代码

`packages/app-store` 每个集成一个目录（calendar/video/crm/payment 等），通过 app-store-cli 生成 `apps.server.generated.ts`、`apps.metadata.generated.ts` 等注册表文件（AGENTS.md 禁止手改）。凭据加密依赖 `CALENDSO_ENCRYPTION_KEY`（`.env.example`）；日历同步走 destination calendar 与 `packages/features/calendar-subscription`。API v2 与 web 共享能力经由 `packages/platform/libraries/index.ts` 中转导出（AGENTS.md 专节说明了 tsconfig path 限制的原因）。

## 6. 构建与运行

来自 README 与根 `package.json` scripts（实测）：

```bash
yarn                      # 安装（Yarn 4.12，Node >=18，Postgres >=13）
cp .env.example .env      # 必填 NEXTAUTH_SECRET、CALENDSO_ENCRYPTION_KEY（openssl rand 生成）
yarn dx                   # 一键开发环境（turbo dx，含 db 迁移 + seed + dev）
yarn dev                  # 只起 web（turbo run dev --filter=@calcom/web）
yarn db-deploy / db-seed  # 数据库迁移 / 种子数据（packages/prisma/seed.ts）
yarn test                 # TZ=UTC vitest run（单元/集成）
yarn e2e                  # NEXT_PUBLIC_IS_E2E=1 playwright test --project=@calcom/web
yarn type-check:ci        # tsc 全仓类型检查（CI 必过项）
yarn build                # turbo run build --filter=@calcom/web...
```

生产部署：`Dockerfile`（node:22 多阶段构建，EXPOSE 3000，healthcheck wget 自检）+ `docker-compose.yml`（postgres + redis + calcom + calcom-api + prisma studio 五个服务，镜像走 `calcom.docker.scarf.sh/calcom/cal.diy`）；另有 Heroku `Procfile` 与 `deploy/` 目录。Windows 开发有已知坑：`packages/prisma/.env` 是指向根 `.env` 的符号链接，需手动复制（README 第 110-115 行），本次 tarball 解压也正因该符号链接在 Windows 上中断（本体文件均在包内，无内容损失）。

## 7. 测试与工程质量

- 规模实测：417 个 `*.test.ts(x)` 单元/集成文件 + 68 个 e2e spec；Vitest workspace 16 个项目、Playwright 8 个 project（`vitest.workspace.ts`、`playwright.config.ts`）。
- CI（`.github/workflows/all-checks.yml`）：type-check、lint、unit-test、integration-test、build、build-atoms 六道门；另有 e2e 系列五条 workflow 与 `check-prisma-migrations.yml` 校验迁移一致性。
- 生产拨测：Checkly（`checkly.config.ts` + `__checks__/calcom-dashboard.check.js`、`location-link.check.js`）。
- 质量观察：错误类分层（`ErrorWithCode` vs `TRPCError`）、`withReporting` 包装可观测、`piiHasher` 对 IP 做 PII 哈希（`apps/web/pages/api/book/event.ts`）、统一 `defaultResponder`。仓库对 AI 协作的规范密度（AGENTS.md + agents/rules/ + spec 流程）在同类项目中罕见，本身就是质量约束的一部分。

## 8. 社区与维护状态

- 活跃度：`pushed_at` 2026-11-03，与本次分析仅隔一天，高频维护中（gh api）。50 个 GitHub workflow 中 10+ 条是 cron（提醒邮件/短信/WhatsApp、webhook 触发、时区变更、月度摘要、同步 App 元数据等），线上任务体系仍在持续运营。
- 发布节奏：Changesets + `changesets.yml`/`draft-release.yml`；npm 侧 `@calcom/embed-*` 独立发布（`publish-embed` 脚本）。最近 release 为 v6.2.0（2026-03-01），monorepo 内大量包版本由 `.changeset/` 管理（gh api releases）。
- 贡献者：具名贡献者约 436 人（gh api contributors 分页统计；含匿名约 988）。`CONTRIBUTING.md`、`CODE_OF_CONDUCT.md`、`PERMISSIONS.md`、`SECURITY.md` 齐备，README 挂 Help Wanted 徽章引导社区认领。
- 社区版定位对维护的影响：README 明示仅推荐个人/非生产自托管；企业功能不再向此仓库回流，商业化集中在 Cal.com 主仓（本次快照范围外，未确认）。

## 9. 亮点与风险

亮点：

1. 企业级排期引擎完整开源：从时段计算（忙闲聚合 + slot 预占 + 限流 + OOO/假期/时差）到预约事务（防并发、审批、座位、周期、改期）全链路可读，路径见第 5 节。
2. embed 体系是"把预约页嵌入任意网站"的成熟方案：消息协议、namespace 隔离、指令队列、prerender 复用 iframe，文档与 mermaid 图齐备（`packages/embeds/`）。
3. 分层重构示范：router → service → repository + 手写 DI，把 2693 行编排器与 20 余个可测步骤拆开，配套 18 个场景级测试文件，是大型 TS 项目渐进式重构的活教材。
4. AI-agent 优先的工程规范（AGENTS.md + agents/）：把 PR 粒度、错误处理、Prisma 用法写成机器可执行规则。
5. 集成生态规模大：约 120 个 App 全部开源 + 代码生成注册表，44 种语言。

风险/局限：

1. 自我声明不适合生产（README WARNING）：团队/组织/工作流/SSO 等企业功能被裁剪，自托管需自行承担安全与运维。
2. 裁剪残留：根 `package.json` scripts 仍引用不存在的 `@calcom/website`、`@calcom/console`、`@calcom/swagger`、`@calcom/ai`（apps/ 实测只有 api/docs/web）；`AGENTS.md` 仍引用已删除的 `packages/features/ee/workflows` 路径；`apps/api/v2/src/modules/` 仍保留 organizations/teams 模块——fork 边界不彻底，新人易困惑。
3. 体量与门槛：yarn.lock 1.5MB、100 个 Prisma model、50 条 workflow，跑通 dev 环境需要 Postgres+Redis+大量第三方 key；Windows 开发有符号链接坑。
4. 社区版 release 节奏依赖 Changesets 手动触发，最近 tag 为 v6.2.0（2026-03-01），与主仓的版本对应关系未确认。

## 10. 总结评价

Cal.diy 是当前开源世界最完整的"排期基础设施"参考实现：它把 Cal.com 多年打磨的可用性聚合、预约事务、嵌入 SDK 与集成生态以纯 MIT 形式交给社区，同时以清晰的 router/service/repository 分层和高度规范化的工程流程维持近乎每日推送的活跃度。想自建预约系统、研究时段计算与并发预约正确性的工程师，直接读 `packages/features/availability`、`packages/trpc/server/routers/viewer/slots` 与 `packages/features/bookings` 三处即可值回票价；想做嵌入式日历产品的人则应精读 `packages/embeds`。但它明确不是开箱即用的 SaaS 替代品——追求生产级团队协作与权限体系，仍需面对它刻意留下的企业功能空白与社区版支持风险。
