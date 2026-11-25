# medusajs/medusa 项目分析

> 星数 39500（快照 2026-10-24，2026-11-17 经 gh api 复核为 37.3k）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-17

## 1. 项目速览

一句话定位：开源电商后端框架——Shopify 的开源替代，以"可组合模块 + 工作流"架构提供商品/订单/购物车/支付等商务原语，供开发者自建 B2B、DTC、市场等定制化电商系统。

| 项目 | 内容 |
| --- | --- |
| 名称 | medusajs/medusa |
| 星数 | 39500（快照）；gh api 复核 37273（2026-11-17） |
| 主语言 | TypeScript（gh api 核实） |
| 许可证 | Open-core：主体 MIT + 企业版条款（`LICENSE`、`ENTERPRISE-LICENSE.md`） |
| 当前版本 | v2.22.7（`packages/medusa/package.json`） |
| 最近活跃 | develop 分支最近 push 2026-11-16，最新 release 2026-11-13（gh api） |
| 创建时间 | 2020-01-18，MedusaJS 团队（YC 背景）， forks 5534 |

## 2. 项目是什么，解决什么问题

Medusa 是一个开源电商平台，自述"Building blocks for digital commerce"（`README.md`），即把商品、购物车、订单、支付、库存、促销等电商核心逻辑做成积木块，让开发者不需要"重造核心商务逻辑"就能拼装出自己的电商应用：B2B/DTC 商城、分销平台、市场（marketplace）、PoS 等（`README.md`）。

与 Shopify 这类托管 SaaS 相比，Medusa 走的是"你拿代码"路线：应用部署在你自己的 Node.js + PostgreSQL 环境里，所有业务逻辑可通过自定义模块、API 路由、工作流进行扩展；官方口号已更新为"为 agent 和开发者打造的最灵活商务平台"（gh api repo description），重心明确放在开发者与 AI 时代的可编程性上。其商业模式是 open-core：核心开源（MIT），Medusa Cloud 提供托管部署（README 快速上手首选引导 Cloud），企业特性走 `ENTERPRISE-LICENSE.md` 商业授权。

仓库当前处于 v2 架构（2.x 系列版本号，`packages/medusa/package.json` 中 version 2.22.7）。v1 时代是"单体 Express 应用 + plugin"形态（根 `CHANGELOG.md` 中 v1.0.x 记录可证其 2020 年起点）；v2 是一次彻底重写：按领域拆成 30+ 独立 commerce module，模块间用 link-modules 关联，写操作统一走可回滚的 Workflow 编排（此为基于仓库现状与官方文档 `AGENTS.md` 描述的结构性结论）。

v2 的核心扩展模型（`AGENTS.md`，根目录）："Medusa apps are customized, not forked"——应用在自己工程的 `src/` 下放自定义模块（`src/modules`）、模块关联（`src/links`）、工作流（`src/workflows`）、API 路由（`src/api`）、后台 UI 扩展（`src/admin`），而不是改框架源码。变更（mutations）必须走工作流而非直接调 service，以获得回滚（补偿）能力与钩子（`AGENTS.md`）。

## 3. 技术栈与依赖

- 语言：TypeScript 全仓（约 11,600 个 .ts/.tsx 文件，`find` 统计，排除 node_modules）。
- 运行时与依赖：Node.js v22.22.0+，PostgreSQL 必需（`README.md`）。
- HTTP 层：**Express**——`packages/core/framework/src/http/express-loader.ts` 引入 express、express-session、cookie-parser、compression、morgan，并用 ioredis 做会话存储；框架目录中未发现 Fastify。
- ORM：**MikroORM + PostgreSQL**。`packages/core/utils/src/dal/mikro-orm/` 下是整套 DAL 实现（repository、base-entity、big-number-field、软删除过滤器、全文搜索过滤器、跨模块查询等）；`packages/core/framework/package.json` 以 `@medusajs/framework/mikro-orm/*` 路径重导出 mikro-orm core/knex/migrations，并自带 `medusa-mikro-orm` CLI。
- 数据模型 DSL：模块的实体不直接写 MikroORM 装饰器，而是用自研 `model.define()` 链式 DSL 声明（`packages/modules/product/src/models/product.ts`，如 `title: model.text().searchable().translatable()`），再编译为 MikroORM 实体。
- 依赖注入：awilix 容器（`packages/core/modules-sdk/src/medusa-app.ts` 使用 `@medusajs/deps/awilix`、`createMedusaContainer`）。
- 编排：自研 workflow/orchestration 引擎（`packages/core/orchestration`），支持 inmemory 与 redis 两种执行状态存储（`packages/modules/workflow-engine-inmemory`、`workflow-engine-redis`）。
- 构建与工程化：Yarn workspaces（根 `yarn.lock`，注意是 Yarn 而非 pnpm）+ Turborepo（`turbo.json`），SWC 转译 + esbuild/rollup 打包，Jest（SWC jest + define_jest_config.js）为主、admin 侧 Vitest。
- Admin UI：React 18 + Vite + TanStack Query/Table/Virtual + react-hook-form + react-i18next（`packages/admin/dashboard/package.json`），组件库 `packages/design-system/ui`。

## 4. 目录结构与代码组织

顶层目录（源自码根目录实际列表）：

| 目录 | 职责 |
| --- | --- |
| `packages/medusa` | 主包 `@medusajs/medusa`：Store/Admin API 路由、CLI、subscribers、jobs，聚合全部模块 |
| `packages/core/*` | 核心库：framework（运行时/HTTP/DI）、types、utils（含 DAL）、modules-sdk、workflows-sdk、orchestration、core-flows（预置工作流）、query（查询引擎）、js-sdk |
| `packages/modules/*` | 30+ 商务模块（product、cart、order、inventory、pricing、payment、promotion、region、sales-channel、tax、fulfillment、customer、auth、user、api-key、index、link-modules…）与基础设施模块（event-bus-local/redis、cache-redis、locking、file、notification、workflow-engine-*） |
| `packages/modules/providers/*` | 模块 provider：支付（Stripe/Adyen）、文件（S3/local）、通知（SendGrid）、认证（emailpass/GitHub/Google/OIDC）、缓存/锁（Redis/PG）等 |
| `packages/admin/*` | admin dashboard（React）及其打包器 admin-bundler、admin-vite-plugin、admin-sdk |
| `packages/cli/*` | create-medusa-app、medusa-cli、oas（OpenAPI 规范）、http-types-generator |
| `packages/design-system/*` | ui 组件库、icons、toolbox、ui-preset |
| `packages/plugins/*` | cloud（Medusa Cloud 集成）、draft-order、loyalty |
| `integration-tests/` | http（API 级集成测试）与 modules 两套 |
| `www/` | 文档 monorepo（book/resources/ui 三个 MDX 文档站，随仓库维护） |
| `.github/workflows/` | 43 个 CI/自动化 workflow |

组织方式评述：典型的大型 Yarn workspaces monorepo，`packages/` 下按"主包 / core 基础设施 / 领域模块 / provider / admin / cli"分层；每个领域模块自带 models/services/migrations/joiner-config（如 `packages/modules/product/src/`），可独立发版与替换。根目录还有 `AGENTS.md`（根级 + 每包一个，面向 AI 编程代理的开发说明）、`CLAUDE.md`、`thoughts/`、`memory/` 等 AI 辅助开发痕迹，属于较激进的"AI 原生工程化"仓库。

## 5. 核心架构与关键实现

### 5.1 模块系统：一切商务能力皆为可替换模块

每个模块用 `Module()` 工厂导出，以 product 为例（`packages/modules/product/src/index.ts`）：

```ts
import { Module, Modules } from "@medusajs/framework/utils"
import { ProductModuleService } from "@services"

export default Module(Modules.PRODUCT, {
  service: ProductModuleService,
})
```

启动时由 `packages/core/modules-sdk/src/loaders/register-modules.ts` 依据 `definitions.ts` 的 ModulesDefinition 注册模块解析（ModuleResolution）：默认模块声明可被用户配置覆盖，也支持用户自定义模块（`getCustomModuleResolution`）；外部（EXTERNAL scope）模块声明目前直接抛 "External Modules are not supported yet"（该文件内可见）。模块服务最终挂到 awilix 容器，业务代码用 `Modules.<NAME>` 从容器解析（`AGENTS.md`、`packages/core/modules-sdk/src/medusa-app.ts`）。模块间的数据关联不靠外键，而由 link-modules 定义（`packages/modules/link-modules/`），查询侧由 Query 引擎跨模块 join（`packages/core/query/src`）。

### 5.2 Workflow 编排：写操作的事务与回滚内核

`packages/core/orchestration/src/transaction/transaction-orchestrator.ts` 定义 `TransactionOrchestrator extends EventEmitter`，负责"管理和执行分布式事务"；配合 `distributed-transaction.ts`、`transaction-step.ts` 实现按步骤的补偿回滚，并有 `datastore/` 持久化执行状态（可存 inmemory 或 Redis，对应 workflow-engine-* 模块）。`packages/core/workflows-sdk/src/medusa-workflow.ts` 用全局注册表暴露所有已编译 workflow：

```ts
class MedusaWorkflow {
  static workflows: Record<string, (...) => ...> = {}
  static registerWorkflow(workflowId, exportedWorkflow) { ... }
  static getWorkflow(workflowId): ExportedWorkflow { ... }
}
global.MedusaWorkflow ??= MedusaWorkflow
```

官方预置业务编排集中在 `packages/core/core-flows/src/`，覆盖约 40 个域（api-key、cart、order、payment、promotion…），仅 order 一个域就有 cancel/claim/archive/create-fulfillment 等数十个 workflow 文件（`packages/core/core-flows/src/order/workflows/`）。`packages/medusa/src/api` 的路由本身保持薄，写操作委托 workflow（架构约定见 `AGENTS.md`）。

### 5.3 HTTP 层与 API 组织

`packages/core/framework/src/http/express-loader.ts` 组装 Express 应用：session（生产环境强制 `Secure` cookie 并以 `sameSite=lax` 防 CSRF，注释中引用了具体 CVE 编号 GHSA-jhvc-qx3m-6r3q）、安全响应头、压缩、`/health` 健康检查。路由加载由同目录 `routes-loader.ts`、`middlewares.ts`、`routes-sorter.ts` 等文件式约定完成。主包 API 按 `packages/medusa/src/api/{admin,store,auth,cloud,hooks,rbac}` 划分，store 侧有 carts、orders、products、payment-collections、product-categories、returns、shipping-options 等约 20 个资源目录（目录列表实测）。

### 5.4 Admin Dashboard 与应用装配

admin dashboard 是完整 React SPA（`packages/admin/dashboard/src/`，40 个路由域：products、orders、promotions、price-lists、inventory、policies、refund-reasons 等），由 `packages/admin/admin-bundler`/`admin-vite-plugin` 在应用构建时打进主服务，`packages/medusa/src/loaders/admin.ts` 负责挂载。主应用装配链在 `packages/medusa/src/loaders/`（index → api/admin/search）与 `packages/core/framework/src/medusa-app-loader.ts`。扩展点还包括 subscribers（`packages/medusa/src/subscribers/`：payment-webhook、search-ingestion、configurable-notifications）与定时 jobs（`packages/medusa/src/jobs/`）。

## 6. 构建与运行

```bash
# 创建应用（官方推荐入口，README.md）
npx create-medusa-app@latest my-medusa-store   # Node >= 22.22.0 + PostgreSQL

# monorepo 开发（根 package.json / turbo.json）
yarn                # 安装 workspace 依赖
turbo run build     # turbo 构建（依赖拓扑 ^build）
turbo run test      # 单元测试
turbo run test:integration:chunk   # 集成测试分片

# 数据库
medusa migrations:run        # 迁移（medusa-mikro-orm 体系）
medusa db:create / medusa seed  # medusa-cli 命令
```

模块包均可经 npm 单独安装（`@medusajs/product` 等），应用通过 `medusa-config.js` 声明模块与 provider（`integration-tests/http/medusa-config.js` 为实例）。

## 7. 测试与工程质量

- 测试规模：约 733 个 `.spec.ts` + 242 个 `.test.ts(x)` ≈ 975 个测试文件（find 统计，排除 node_modules）；单元测试为主（各包 `__tests__/`，Jest + SWC），admin/dashboard 用 Vitest + Testing Library。
- 集成测试独立成 workspace：`integration-tests/http/`（带独立 jest.config、medusa-config.js，跑真实 HTTP API）与 `integration-tests/modules/`（模块级），另在 `packages/core/utils/src/dal/mikro-orm/integration-tests/` 有一组 DAL 集成测试。
- CI：`.github/workflows/` 共 43 个 workflow。除常规测试外，自动化程度很高：`pr-guardrails.yml`（PR 必须关联 issue，含定时清理）、`oas-test.yml` + `validate-http-types.yml`（OpenAPI 规范与 HTTP 类型一致性）、`test-cli-with-database.yml`、docs 系列自动化、以及 `agent-docs-scan.yml`、`kapa-*.yml`、`review-prs.yml` 等 AI 参与的文档/评审流程。
- 代码质量观察：类型化程度高（http-types 生成器把路由类型喂给 JS SDK）；模块边界清晰、迁移与模型同包；根 `AGENTS.md` 明确编码约定与包级说明，工程纪律良好。

## 8. 社区与维护状态

- 活跃度：develop 最近 push 2026-11-16（gh api），持续高强度维护。
- Release 节奏（gh api releases）：2026-11-13 v2.22.7、11-09 v2.22.6/5、11-06 v2.22.4、11-04 v2.22.3、10-30 v2.22.2、10-23 v2.22.1、10-21 v2.22.0、9-28 v2.21.2、9-11 v2.21.0、8-13 v2.19.0——补丁级每周 1-3 个，minor 约每月一个，节奏极快。
- 贡献者：约 594 人（contributors API 分页统计，含匿名）；CODEOWNERS 存在（根目录），核心由 MedusaJS 公司团队主导 + 社区 PR。
- 商业实体：MedusaJS, Inc.（YC 背景，任务参数与 `LICENSE` 版权行 "Copyright (c) 2021 Medusajs" 佐证）；商业化路径为 Medusa Cloud 托管 + 企业版授权（`README.md`、`ENTERPRISE-LICENSE.md`）。
- 星数说明：任务快照 39500（2026-10-24），gh api 于 2026-11-17 读得 37273，差额较大，未确认原因（可能含快照口径差异）。

## 9. 亮点与风险

亮点：
1. 模块化架构彻底：30+ 领域模块 + provider 体系 + link-modules 跨模块关联，几乎每个商务能力都可替换或独立部署（`packages/modules/`）。
2. 自研 workflow 编排内核：带分布式事务、步骤补偿与状态持久化（inmemory/Redis），把"可回滚的业务流程"做成一等公民（`packages/core/orchestration/`）。
3. 开发者体验完善：create-medusa-app 一键起站、JS SDK 与 OpenAPI 类型自动生成、docs 与代码同仓同步（`packages/cli/*`、`www/`）。
4. AI 原生工程化：根级 + 包级 `AGENTS.md`、`memory/`、大量自动化与 AI 辅助 CI workflow，2025-2026 时代少见的工程配置。
5. 测试金字塔完整：近千个测试文件 + 独立集成测试工作区 + OAS 校验（`integration-tests/`、`.github/workflows/oas-test.yml`）。

风险/局限：
1. License 已从纯 MIT 变为 open-core（`LICENSE` 顶部的 EE 例外声明 + `ENTERPRISE-LICENSE.md`），商用集成前需甄别企业版边界。
2. 发版与重构节奏极快（月度 minor、周度 patch），跟进升级成本高；v1→v2 已是一次不兼容重写，历史上发生过，未来仍可能再有。
3. 仅支持 PostgreSQL（README 明确要求），且 EXTERNAL 模块机制尚未开放（`packages/core/modules-sdk/src/loaders/register-modules.ts` 直接抛错），多形态部署能力受制于官方路线。
4. 体量大（1.1 万+ TS 文件、30+ 包），初学者理解框架全貌（modules/workflows/links/query/DI）的学习曲线明显陡于"单体电商模板"。

## 10. 总结评价

Medusa 是目前开源电商后端里工程化最完整、架构最"平台化"的一个：它不满足于做 Shopify 的免费替身，而是把电商能力拆成可组合模块、把业务变更收敛到可回滚工作流、把扩展点做成约定式文件系统，再用 Cloud 与企业授权构成商业闭环。README、AGENTS.md 与 packages/ 结构一致地展示了这条主线。适合需要深度定制商务逻辑（B2B、市场、多仓多渠道）的工程团队作为基座，也适合研究"模块化后端 + 工作流编排"架构设计的人精读 `packages/core`（modules-sdk、orchestration、framework）——这是全仓最有含金量的部分；只想快速搭标准商店的团队则可能被其体量与升级节奏劝退，反而是 Medusa Cloud 或 Next.js starter 更合适。
