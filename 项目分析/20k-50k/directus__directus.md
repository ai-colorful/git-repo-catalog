# directus/directus 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：Directus 是"数据库镜像"式 Headless CMS / 数据平台——连上任意 SQL 数据库，即时获得 REST + GraphQL API、Vue 可视化管理 Studio，以及面向 AI Agent 的原生 MCP 服务器。

| 项目 | 内容 |
|---|---|
| 名称 | directus/directus |
| 星数 | ~38.7k-39.5k（gh api 实测 38,711，快照 39,500） |
| 语言 | TypeScript（Vue + Node.js） |
| 许可证 | MSCL-1.0-GPL（源码可用许可，发布满 4 年自动转 GPL-3.0；`license` 文件） |
| 最新版本 | v12.6.1（2026-11-04，gh api releases/latest） |
| 最近活跃 | push 2026-11-13（持续高活跃） |
| 贡献者 | ~441 人（gh api contributors 分页推算） |
| 维护方 | Monospace Inc（作者 Rijk van Zanten、Ben Haynes，见 `api/package.json`） |

## 2. 项目是什么，解决什么问题

传统 CMS（如 WordPress）自带数据模型和数据库，而 Directus 走另一条路：**不拥有数据模型，而是镜像已有数据库**。它通过 schema 内省（introspection）读取任意 SQL 数据库的表、字段、关系，动态生成 REST 与 GraphQL API，并配套一个可视化管理面板（官方称 "Data Studio"），让非技术成员直接操作线上数据（`directus/readme.md`）。

这解决了两类痛点：一是"已有数据库想快速获得 API 和后台"的场景，无需迁移数据或改表结构，Directus 直接包裹现有库；二是"团队协作"场景——工程师控制 schema 和权限，内容/运营人员用可视化界面干活，AI Agent 也通过同一套权限访问数据。

2025-2026 年的项目重心明显转向 AI（`directus/readme.md`："The Collaborative Backend for Builders & AI"）：内置 AI Assistant（Studio 内创建内容、翻译、触发工作流）、原生 MCP 服务器（Claude/Cursor/ChatGPT 等工具直连数据），且强调"AI 默认受治理"——AI Agent 与人类用户走同一套策略权限，无特殊通道。

产品形态：可自托管（Docker/本地/自有基础设施）或使用官方 Directus Cloud；商业上对年收入低于 $5M、员工少于 50 人的组织免费（Open Innovation Grant 许可，`directus/readme.md`）。

## 3. 技术栈与依赖

- **后端（api/）**：Node.js（HEAD 要求 >=26，`api/package.json` engines）+ Express 4 + Knex 3（多方言 SQL 构建器）；数据库驱动作为 optionalDependencies：pg / mysql2 / sqlite3 / tedious（MSSQL）/ oracledb，另有 cockroachdb 支持（`api/src/database/index.ts`、`pnpm-workspace.yaml` catalog）
- **前端（app/）**：Vue 3.5 + Vite 8 + Pinia + vue-i18n + TipTap 富文本 + maplibre-gl 等（`pnpm-workspace.yaml` catalog、`app/package.json`）
- **SDK（sdk/）**：零依赖 TypeScript 客户端，基于全局 fetch/WebSocket，组合式 `with()` 扩展设计（`sdk/src/client.ts`）
- **AI 栈**：Vercel AI SDK（ai + @ai-sdk/openai/anthropic/google/openrouter）+ @modelcontextprotocol/sdk（`api/package.json`）
- **关键三方库**：graphql-compose（GraphQL 生成）、liquidjs（模板）、sharp（图像处理）、argon2（密码哈希）、otplib（TOTP 2FA）、isolated-vm（扩展沙箱）、@tus/server（断点续传上传）、ioredis + keyv（缓存/总线）、pino（日志）、prom-client（指标）、liquidjs、rollup/tsdown（构建）、vitest（测试）
- **工程化**：pnpm 12 monorepo（catalog 严格模式锁定全部依赖版本，`pnpm-workspace.yaml`）、changesets（版本发布）、ESLint + Prettier + Stylelint、CodeQL

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `api/` | Node 后端：REST/GraphQL/WebSocket API、认证、权限、Flows、AI/MCP、数据库层（`@directus/api` v41，1266 个 TS 文件） |
| `app/` | Vue 3 管理面板（Data Studio）：接口/显示/布局/面板/模块体系（1556 个 TS/Vue 文件） |
| `sdk/` | TypeScript SDK（223 个 TS 文件，REST/GraphQL/Realtime/Auth 客户端） |
| `directus/` | npm 发布入口包：bin CLI + 版本号，依赖 `@directus/api`（v12.6.1） |
| `packages/` | 35+ 共享包：`schema`（schema 内省）、`schema-builder`、`types`、`env`、`errors`、`extensions*`、`storage*`（6 种存储驱动）、`memory`、`pressure`、`ai`、`system-data`、`validation`、`create-directus-*` 等 |
| `tests/` | `e2e/`（跨方言集成测试）+ `sandbox/`（扩展开发沙箱） |

组织方式评述：典型 pnpm monorepo，`pnpm-workspace.yaml` 中 `ignoreWorkspaceCycles: true` 显式容忍 api↔directus 循环依赖（注释标注"temp fix"）；依赖用 catalog 严格模式集中管理（catalogMode: strict），这是 pnpm 12 的新实践。README 顶层 `readme.md` 是指向 `directus/readme.md` 的符号链接（tarball 已确认）。前端不单独发包，构建后由 API 服务静态托管（`SERVE_APP` 环境变量，`api/package.json` scripts）。

## 5. 核心架构与关键实现

### 5.1 数据库抽象层（Knex 多方言）

`api/src/database/index.ts` 单例管理 Knex 实例：从 `DB_*` 环境变量读取配置，按 client 注入方言特化逻辑——SQLite 开启 `PRAGMA foreign_keys`，CockroachDB 设置 `serial_normalization=sql_sequence`，OracleDB 强制 ISO 时间格式会话变量，MySQL 将 client 改写为 mysql2 并清理冲突连接参数。单例同时持有 `@directus/schema` 的 SchemaInspector（内省器），并接入 metrics/日志。这种"一个 Knex 实例 + 方言钩子 + optionalDependencies 驱动"的设计让安装体积按需伸缩（不装 oracledb 也能跑 Postgres）。

### 5.2 Schema 内省与快照迁移（数据库镜像的核心）

这是 Directus 独特卖点的实现载体：

- `packages/schema/src/dialects/` 含 6 个方言文件（postgres.ts、mysql.ts、sqlite.ts、mssql.ts、oracledb.ts、cockroachdb.ts），实现 `createInspector()` 内省已有库的表/列/外键。
- `api/src/utils/schema/get-snapshot.ts` 把"用户库 + Directus 系统表"合并为带版本号的 Schema Snapshot（collections/fields/relations 三元组，过滤系统集合与非托管字段）。
- `api/src/services/schema.ts` 的 SchemaService 提供 `snapshot → diff → apply` 流水线：diff 生成 JSON 迁移补丁（可人工审阅），`applyDiff` 应用到目标库。这使"外部改表 → 拉快照 → 生成 diff → 应用"成为官方工作流，Directus 自身表结构变更也可版本化。

```ts
// api/src/services/schema.ts（节选）
async apply(payload: SnapshotDiffWithHash, options?: { force?: boolean }): Promise<void> {
    const currentSnapshot = await this.snapshot();
    const snapshotWithHash = this.getHashedSnapshot(currentSnapshot);
    if (!validateApplyDiff(payload, snapshotWithHash, options?.force)) return;
    await applyDiff(currentSnapshot, payload.diff, { database: this.knex });
}
```

### 5.3 查询引擎：AST 统一执行

`api/src/database/run-ast/README.md`（项目自带架构文档）说明：REST、GraphQL、WebSocket 请求统一转为 AST（FieldNode / M2ONode / O2MNode / A2MNode / FunctionFieldNode），根节点经 `getDBQuery()` 翻译为 SQL，关系字段递归调用 `runAst()` 后把结果注入上层——深度嵌套关联查询由此实现。权限校验（process-ast）在 AST 阶段注入过滤条件，而非事后过滤。

### 5.4 权限系统：Policies + 字段/条目级 RBAC

`api/src/permissions/` 是独立模块化子系统：`lib/` 负责按 accountability 拉取 policies 与 permissions（含缓存 `cache.ts`）；`modules/` 有 13 个原子能力——process-ast/process-payload（把权限编译进查询与写入）、validate-access、fetch-allowed-fields（字段级白名单）、fetch-global-access、fetch-policies-ip-access（IP 条件权限）等。权限模型为 roles → policies → permissions（每个 permission 可细化到 collection/field/action/权限规则），同一套策略同时约束人与 AI Agent。管理员进入 App 还需叠加 `with-app-minimal-permissions.ts` 的最小权限检查。

### 5.5 Flows 自动化引擎

`api/src/flows.ts`（FlowManager 单例）注册五类触发器（事件 filter/action、cron、手动、websocket 操作），执行树由 `construct-flow-tree` 构建；每个操作节点在 `api/src/operations/` 有独立实现：item-create/read/update/delete、condition、exec（沙箱执行）、mail、notification、request、sleep、transform、throw-error、log、json-web-token，以及新的 ai-text / ai-image（AI 内容生成操作）。执行时通过 `fetchPermissions` 以指定用户身份运行（Flow 可降权），支持 p-queue 并发与 p-retry 重试，日志脱敏（redact-object）。

### 5.6 AI 与 MCP 服务器

`api/src/ai/` 含 chat/tools/providers/telemetry 子模块（基于 Vercel AI SDK 接入 OpenAI/Anthropic/Google/OpenRouter）；`api/src/controllers/mcp/` 实现原生 MCP 服务器（含 OAuth clients、同意页 templates）——外部 Agent 用 OAuth 授权后以 MCP 协议访问数据，权限走同一 policies 体系（`directus/readme.md` "Governed by default"）。

### 5.7 认证与 SSO/MFA

`api/src/auth/drivers/` 提供 local、ldap、oauth2、openid、saml 五种驱动（SSO 开箱即用）；2FA 基于 otplib 的 TOTP（`api/src/services/tfa.ts`）；密码哈希 argon2。`api/src/controllers/auth.ts` 暴露 login/refresh/logout/me 端点。

### 5.8 前端 Studio 与扩展体系

`app/src/modules/` 八大模块（content/files/users/insights/flows/settings/visual/deployment）；可组合的三类 UI 扩展原语：interfaces（47 个）、displays（19 个）、layouts（calendar/cards/kanban/map/tabular，`app/src/interfaces`、`app/src/layouts`、`app/src/panels`）。`api/src/extensions/manager.ts` 是运行时扩展管理器（配合 isolated-vm 沙箱与 `packages/extensions-registry` 的 npm 式注册表）。

## 6. 构建与运行

来自根 `package.json`、`api/package.json` 与 `AGENTS.md`（要求 Node 26 + pnpm 12）：

```bash
pnpm install                      # 安装依赖
pnpm build                        # 全 workspace 构建（tsdown/rollup/vite）
pnpm dev                          # api: tsx watch src/start.ts（开发服务器，SERVE_APP=true 托管前端）
pnpm test                         # 全仓 vitest
pnpm --filter @directus/api test  # api 单测
npx create-directus-project       # 脚手架创建新项目（packages/create-directus-project）
docker compose up                 # 仓库自带 docker-compose.yml（app+db+缓存）
```

Dockerfile 多阶段构建（builder 编译 → 生产镜像），另有 `Dockerfile.dhi` 变体；`ecosystem.config.cjs` 提供 pm2 生产配置。

## 7. 测试与工程质量

- **单测**：api 415 个 `*.test.ts`、app 400 个、sdk 也有独立 tests/（vitest + @vitest/coverage-v8；api 用 knex-mock-client mock 数据库层）——文件级统计自 `find api/src -name "*.test.ts"` 等。
- **E2E**：`tests/e2e/` 123 个测试文件，按 16 个领域组织（app/auth/collections/field-types/flows/graphql/permissions/schema/websockets/deployments 等），vitest 多 project 跑多数据库方言（默认 sqlite project，`ALL=true` 跑全部方言，`tests/e2e/package.json`）。
- **CI**：`.github/workflows/` 19 个 workflow——check（lint+typecheck）、changeset-check、e2e.yml/e2e-pr.yml（跨方言）、codeql-analysis、release/prepare-release（changesets 自动发版）、DockerHub README 同步、stale/lock-threads 社区自动化。
- **质量观察**：核心复杂模块自带架构文档（`api/src/database/run-ast/README.md`、tests/e2e/readme.md）；仓库有 `AGENTS.md` 指导 AI 编码助手，并支持 🤖🤖🤖 标题后缀的 Agent PR 快速通道；安全响应痕迹明显（items.ts 注释引用 GHSA-2xcm-7h22-3m66 权限漏洞修复）。

## 8. 社区与维护状态

- **活跃度**：极高。gh api 显示 push 时间 2026-11-13（分析前一天）；最新 release v12.6.1 发布于 2026-11-04，节奏约每月多个 minor/patch。
- **发版机制**：changesets 驱动（`.changeset/` 目录 HEAD 下有 22 个待发布变更集），`packages/release-notes-generator` 自动生成 release notes。
- **贡献者结构**：~441 名 GitHub 贡献者，核心维护者为 Monospace 团队（api/package.json maintainers）；贡献需签 CLA（cla.yml workflow + cla.md）；国际化靠 crowdin.yml 社区翻译。
- **商业化**：Monospace Inc 提供 Directus Cloud 与企业许可；README 明示"低于 $5M 营收/50 员工免费"的 Open Innovation Grant 策略。

## 9. 亮点与风险

**亮点**

1. 数据库镜像架构：内省已有库而非锁定自有模型，快照-diff-apply 迁移流水线（`api/src/services/schema.ts`）让"先有库后有 CMS"成为一等公民。
2. 统一查询引擎：REST/GraphQL/WebSocket 归一为 AST 再编译成 SQL，权限在 AST 层注入，保证三种入口行为一致（`api/src/database/run-ast/README.md`）。
3. 人机同权：MCP 服务器与 AI Assistant 复用同一套 policy RBAC，AI 治理设计领先同类（`api/src/controllers/mcp/`）。
4. 工程化成熟：35+ 内部包 catalog 严格锁版、全方言 e2e 矩阵、CodeQL、AI 协作规范（AGENTS.md），单体仓库质量高。
5. 可扩展性强：interfaces/displays/layouts/modules 四类前端扩展 + endpoint/hook/operation 后端扩展 + 6 种存储驱动。

**风险/局限**

1. 许可证变更风险：已从 GPL-3.0 转为 MSCL-1.0-GPL 源码可用许可（禁止"Competing Use"商业竞争用途，license 文件第 31-45 行），GPL-3.0 权利要等发布满 4 年才生效——下游 SaaS/二次分发需重新评估合规。
2. 版本组合激进：HEAD 要求 Node >=26、pnpm 12、api 包版本 v41（内部包高频 breaking），跟进成本较高。
3. 已知结构债：`pnpm-workspace.yaml` 显式 `ignoreWorkspaceCycles: true` 容忍 api↔directus 循环依赖（自注 "temp fix"）。
4. 复杂度门槛：权限（policies/permissions/accountability）与 AST 引擎概念密集，深度定制学习曲线陡峭。

## 10. 总结评价

Directus 是 Headless CMS/数据平台赛道中工程完成度最高的开源项目之一：以"内省镜像任意 SQL 库"的架构差异点立足，近年叠加 AI（Assistant + 原生 MCP）抢占了"AI 时代协作后端"的生态位，monorepo 内 3000+ 个 TS/Vue 源文件、800+ 测试文件和多方言 e2e 矩阵体现出成熟的研发体系。它的主要不确定性在商业许可转向（MSCL）带来的合规边界。适合需要给存量数据库快速加 API+后台的团队、想自建内容平台的开发者选用；想研究"多方言 SQL 抽象""AST 查询引擎""细粒度 RBAC"或"AI 权限治理"实现的工程师，其 `api/src/database/`、`api/src/permissions/`、`api/src/controllers/mcp/` 是极佳的源码教材。
