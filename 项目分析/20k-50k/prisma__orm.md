# prisma/orm 项目分析

> 星数 43000（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：Prisma ORM 的下一代版本——**Prisma 8，一次彻底的纯 TypeScript 重写**，以 contract-first（契约优先）架构取代旧的"代码生成 + Rust 查询引擎"模式。

| 项 | 值 |
|---|---|
| 名称 | prisma/orm（原 prisma/prisma，GitHub 302 跳转；v8 开发期间曾用名 prisma-next，见 `CHANGELOG.md` 头部说明） |
| 星数 | 47,660（gh api 实时值，快照 43,000） |
| 语言 | TypeScript 100%（monorepo 内 0 个 Cargo.toml，已无 Rust） |
| 许可证 | Apache-2.0（`LICENSE`、gh api license 字段） |
| 当前版本 | 8.0.0-rc.22（`package.json` version；最新 GitHub Release v8.0.0-rc.22，2026-10-30 发布） |
| 最近活跃 | main 分支最后 push 2026-11-03（gh api），最近提交 2026-11-02 |
| 仓库年龄 | 创建于 2019-06-20（gh api created_at） |
| 支持数据库 | PostgreSQL（一等）、MongoDB（一等）、SQLite（概念验证）、MySQL（开发中，rc.22 已有读支持） |

## 2. 项目是什么，解决什么问题

Prisma 是 Node.js/TypeScript 生态最知名的数据层工具之一，提供声明式 schema（PSL，Prisma Schema Language）、类型安全客户端与迁移工具。本仓库的现状值得特别注意：**它不再是大多数人记忆里的 Prisma 6/7**。仓库 `README.md` 明确写道 "Prisma 8 is a TypeScript rewrite of Prisma ORM"，旧版（Prisma 7）源码移至 `v7` 分支维护（8.0.0 final 后再维护 18 个月），而 main 分支上是全新的 Prisma 8。

旧架构的三个痛点在 `ARCHITECTURE.md` 开篇被点名：PSL、生成的客户端、运行时执行三层紧耦合，导致僵化、每次改 schema 都要重新生成代码、以及生成物对工具/AI agent 不透明。Prisma 8 的解法是 contract-first：

1. **契约取代生成客户端**。`prisma contract emit` 把 schema 编译为一份确定性的 JSON 契约（含 `contractHash` 哈希）+ 一组 TypeScript 类型。契约是稳定的、可 diff 的数据工件，描述数据库结构本身，而不是"燃料"——运行时不再执行任何重型代码生成（`ARCHITECTURE.md` Contract-First Design 节）。
2. **查询在运行时组合**。应用代码用可组合 DSL（`sql().from(...).select(...)`）或轻量 ORM 客户端（`orm.Post.where(...).select(...).include(...)`）写查询；每次查询产生结构化 Plan（AST + 引用列 + 契约哈希），可被校验、观测与拦截。
3. **为 AI agent 而设计**。契约 JSON 机器可读、DSL 是类型化可组合 AST、每条查询带结构化 Plan——agent 可以直接读 schema、生成合法查询并验证，无需逆向黑盒客户端（`ARCHITECTURE.md` Agent-Accessible Design 节）。配套地，npm 包内置 agent skills（`skills/` 目录，`prisma-8` skill 随包分发），仓库里还有 `drive/` 目录——团队自用的 AI agent 工作流"项目记忆"库（triage/spec/plan/qa/retro 分类），说明这套 ORM 本身就是用 AI agent 工作流开发的。
4. **极小内核 + 公共 SPI**。README 声称"包括 Postgres 支持本身在内，一切都建立在面向任何作者的公共 SPI 之上"。官方扩展已有 pgvector、PostGIS、ParadeDB 全文检索、Supabase、arktype JSON 校验、citext 等包（`packages/9-public/@prisma/`、`packages/3-extensions/`）。

需要强调的是 8.0 仍是 release candidate（README：final 预计 4-8 周内），且功能面尚在追赶：`scorecard.md` 用一张按集成测试证据逐格打勾的功能矩阵，公开列出各数据库对 Prisma 8 的支持缺口——SQLite 只是概念验证、MySQL 仍在补读写能力，这是本次重写最务实的自我陈述。

## 3. 技术栈与依赖

| 层 | 选型 | 出处 |
|---|---|---|
| 语言/运行时 | TypeScript 全仓，Node.js >= 24（engines 字段） | `package.json` |
| Monorepo | pnpm 10.27.0 工作区 + Turborepo（turbo run build/test） | `pnpm-workspace.yaml`、`package.json` scripts、`turbo.json` |
| 构建 | tsdown（各包 `tsdown.config.ts`），catalog 统一版本管理 | `pnpm-workspace.yaml` catalog 节 |
| 测试 | Vitest + @vitest/coverage-v8 | 根 `vitest.config.ts`、`coverage.config.json` |
| Lint/格式化 | Biome 2.5.8 + dependency-cruiser（架构边界校验） | `biome.jsonc`、`dependency-cruiser.config.mjs` |
| Postgres 驱动 | `pg`（node-postgres）+ `pg-cursor`（命名游标流式读取） | `packages/3-targets/postgres/3-driver/package.json` dependencies |
| Mongo 驱动 | `mongodb` ^7.4 + `bson` ^7（catalog 注明两者 major 必须对齐） | `pnpm-workspace.yaml` |
| 测试用嵌入式库 | `@electric-sql/pglite`（WASM Postgres）、mongodb-memory-server | postgres driver devDependencies、`pnpm-workspace.yaml` allowBuilds |
| Schema 校验 | arktype ^2.2.2（契约 JSON 校验） | `pnpm-workspace.yaml`、各 runtime 包依赖 |
| 关键第三方 | pg-connection-string、pg-cursor、gray-matter、husky、lint-staged | `package.json`、各包 manifest |

没有任何原生二进制/引擎下载步骤——allowBuilds 里出现的 `@prisma/engines@6.18.0`/`prisma@6.18.0` 仅用于 v7 兼容测试 fixture（`pnpm-workspace.yaml`），v8 自身纯 TS。

## 4. 目录结构与代码组织

顶层（`ls` 实测）：

| 目录 | 职责 |
|---|---|
| `packages/` | 全部 96 个内部包（`find packages -name package.json` 计数），按数字前缀分层 |
| `packages/1-framework/` | 目标无关框架域：`0-foundation/contract`（契约类型与规范化）、`1-core`（operations/errors/ts-render）、`2-authoring`（psl-parser/psl-printer）、`3-tooling`（cli、emitter、migration、language-server、vite-plugin） |
| `packages/2-sql-family/` | SQL 家族域：`1-core`（SqlContract/schema-ir/operations）、`2-authoring`（contract-psl/contract-ts/contract-prisma7）、`3-tooling/emitter`、`4-lanes`（relational-core AST + sql-builder DSL）、`5-runtime`（SQL 运行时）、`9-family`（@internal/family-sql 组装） |
| `packages/2-mongo-family/` | Mongo 家族：foundation→authoring→tooling→query→query-builders→transport→runtime→family 分层 |
| `packages/3-targets/` | 具体数据库目标：`postgres`、`mongo`、`mysql`、`sqlite`，每个统一为 `1-target`（类型/编解码/DDL/PSL infer）→`2-adapter`（AST→SQL 渲染）→`3-driver`（连接/事务/游标）→`9-facade`（一行式入口） |
| `packages/3-extensions/` | pgvector、postgis、paradedb、supabase、citext、arktype-json、middleware-cache 等扩展包 |
| `packages/9-public/@prisma/` | 面向 npm 发布的 `@prisma/orm-*` 系列（orm-framework、orm-postgres、orm-mongo、orm-sqlite、orm-mysql、orm-target-*、orm-family-*、orm-extension-*、orm-toolchain，共 19 个） |
| `apps/`、`examples/` | 内部应用（如 lsp-playground）与 14 个示例工程（prisma-8-demo、supabase、cloudflare-worker 等） |
| `test/` | `integration/`（集成测试）、`e2e-tests`、`prisma-bin`、共享 utils |
| `docs/` | 架构文档：237 份 ADR（`docs/architecture docs/adrs/`）、子系统规格、版本策略 |
| `skills/`、`skills-contrib/`、`drive/`、`.agents/` | agent skills 与 AI 工作流协议资产 |
| `scorecard/` | 18 个分类的功能矩阵（123 行迁移功能、81 行 Mongo 功能等） |

组织方式评述：这是教科书级的**分层 monorepo**——包名数字前缀即依赖方向，`ARCHITECTURE.md` 给出依赖流 "Core → Authoring → Targets → Lanes → Runtime Core → Family Runtime → Adapters"，并用 `architecture.config.json` + `pnpm lint:deps`（dependency-cruiser + 多个自研 lint 脚本）机器强制域/层/面（Domain/Layer/Plane）三维边界。命名上 `@internal/*` 是内部包，`@prisma/orm-*` 是发布面，二者经 `9-public` 映射。

## 5. 核心架构与关键实现

### 5.1 契约（Contract）：schema 的确定性 IR

`packages/1-framework/0-foundation/contract/`（43 个 TS 文件）定义 `Contract`/`Source` 类型、canonicalization（`canonicalization.ts`、`canonicalization-storage-sort.ts` 保证 JSON 确定性排序）、校验（`contract-validation-error.ts`）。SQL 侧类型在 `packages/2-sql-family/1-core/contract/`（`SqlContract`/`SqlStorage`）。schema 有两种书写面：PSL（`packages/1-framework/2-authoring/psl-parser`，169 个 TS 文件的自研 parser + binder + attribute-spec）与直接用 TypeScript 常量（`2-sql-family/2-authoring/contract-ts/`）；还有 `contract-prisma7/` 兼容读取旧版 `schema.prisma`（CHANGELOG 提到 `prisma7Schema(...)` 读取器与 `prisma db sign`）。示例 PSL（`packages/1-framework/2-authoring/psl-parser/test/fixtures/schema.psl`）首行 `// use prisma-8`，语法与旧版相近。

### 5.2 查询执行链：Plan → Adapter → Driver

查询以 AST 表达（`packages/2-sql-family/4-lanes/relational-core/`，43 文件：`ast/`、`expression.ts`、`functions.ts`），两条用户面：`sql-builder`（28 文件，`sql().from(t.user).select(...)` 的 table-proxy/field-proxy 实现）与更高层的 ORM 客户端（`9-family` + collections）。执行时的关键降级函数在 `packages/2-sql-family/5-runtime/src/lower-sql-plan.ts`：

```ts
export function lowerSqlPlan<Row>(
  adapter: Adapter<AnyQueryAst, Contract<SqlStorage>, LoweredStatement>,
  contract: Contract<SqlStorage>,
  queryPlan: SqlQueryPlan<Row>,
): SqlExecutionPlan<Row> {
  const lowered = adapter.lower(queryPlan.ast, {
    contract,
    params: queryPlan.params,
  });
  // ... bind 槽只允许出现在 prepare() 路径，ad-hoc 遇到即抛 RUNTIME.PREPARE_BIND_ON_ADHOC
  return Object.freeze({ sql: lowered.sql, params, ast: queryPlan.ast, meta: queryPlan.meta });
}
```

即：**SQL 文本由各目标的 adapter 渲染**（Postgres 在 `packages/3-targets/postgres/2-adapter/src/core/sql-renderer.ts`，1141 行），返回冻结的 `{sql, params, ast, meta}`。Plan 是一等公民：带 AST、绑定槽、内容哈希（`5-runtime/src/content-hash.ts`、`fingerprint.ts`），可供中间件与预算/守护（`guardrails/`、`middleware/`）审查。

### 5.3 目标与驱动：纯 TS 的 Postgres/Mongo 适配层

Postgres 驱动 `packages/3-targets/postgres/3-driver/src/postgres-driver.ts` 直接基于 `pg` + `pg-cursor`，抽象三种绑定：`{kind:'url'}` 自建连接池、外部 `pgPool`、外部单 `pgClient`；文件头注释精确约定了 per-client FIFO 串行、游标流持锁范围与嵌套查询死锁边界。1-target 层（201 个 TS 文件）承载 Postgres 的全部领域知识：编解码注册表（`codec-registry.ts`、`date-codecs.ts`、`temporal-codecs.ts`）、DDL 与迁移渲染（`ddl/`、`migrations/render-ops.ts`）、RLS 策略（`postgres-rls-policy.ts`）、全文检索（`full-text-*.ts`）、从现有库反向推导契约的 `psl-infer/`，以及 `prisma7-binding.ts`/`prisma7-type-map.ts` 的旧版兼容。Mongo 家族同构（`2-mongo-family/7-runtime` 的 `mongo-runtime.ts`、`mongo-execution-stack.ts`）。facade 一行建库：`examples/mongo-blog-leaderboard/src/db.ts` 中 `export const db = mongo({ contract })`；SQL 侧 `examples/prisma-8-demo/src/prisma/db.ts` 演示组合契约 + collections + 扩展 + 中间件（cache、`lints()`、`budgets()`）。

### 5.4 迁移与中间件

迁移不再是无序脚本：`packages/1-framework/3-tooling/migration/src/` 以契约图（`graph.ts`、`graph-ops.ts`）+ 快照库（`contract-snapshot-store.ts`）+ 账本（`ledger-origin.ts`）做确定性迁移计划，CLI 命令 `migration plan`/`db migrate`（scorecard/15-migrations.md 记了 123 行功能）。中间件观察"活动树"（statements/transactions/ORM calls），钩子为 `beforeCompile`（SQL 文本已生成前）、`beforeEncode`、`intercept`、`onRow`，事件为 `activityStarted/Ended/Finalized`（`ARCHITECTURE.md` 及 ADR 271）。

## 6. 构建与运行

来自根 `package.json` scripts（未实际执行构建，按规范只静态阅读）：

```bash
npm install -g pnpm@10        # packageManager 固定 pnpm@10.27.0
pnpm install                  # Node >= 24
pnpm build                    # turbo run build
pnpm dev                      # turbo watch build（仅 packages/**）
pnpm test:packages            # 先构建再 vitest run
pnpm test:integration         # pnpm --filter integration-tests test（需 DATABASE_URL）
pnpm test:e2e / test:examples / test:all
pnpm typecheck                # turbo run typecheck
pnpm lint / lint:deps         # Biome + dependency-cruiser 架构边界校验
pnpm fixtures:check           # 契约 fixture 发射后 git diff 校验
```

用户侧：`npm create prisma`（脚手架，选 Next.js/Hono/Nuxt 等模板）或 `npx prisma orm init`（写 `prisma.config.ts`、发契约）、`npx prisma skills sync`（安装 agent skills）。README 前置条件明确要求 Node.js 24+。

## 7. 测试与工程质量

- 规模：全仓 2863 个 `*.test.ts`（`find` 计数，不含 node_modules）；`test/integration/` 走真实数据库（Postgres 经 PGlite、SQLite 经真实驱动、MongoDB 经 mongodb-memory-server），`test/e2e/` 覆盖 CLI 全流程。
- 质量门禁的严谨度是亮点：`scorecard.md` 规定功能矩阵里 `✅` 必须有**执行数据库的集成测试**背书，"单元级测试（仅断言 SQL/AST/plan）不算"，且每个数据库独立举证——这是把"测试即规格"落成了文档制度。
- CI：`.github/workflows/` 共 19 个工作流——ci.yml、integration.yml、bundle-size.yml（示例工程 bundle 体积门禁）、codeql-analysis、pr-code-security、publish.yml，另有 5 个 `v7-*` 工作流（v7 分支每日构建/测试/发 dist-tag）表明旧版仍在流水线内维护。
- 工程治理：dependency-cruiser + 自研 lint 脚本约 30 个（`pnpm lint:deps` 串联 5 个脚本；`scripts/` 下还有 40+ 个脚本自身的 `*.test.mjs`）；husky + lint-staged；coverage 用 vitest blob 合并分片报告。

## 8. 社区与维护状态

- 活跃度极高：main 分支约 8,553 个提交（gh api Link header per_page=1 last page 推算），最近 100 个提交集中在 16 天内（约 6 commit/天）；commit message 形如 "TML-3780: …"，对应 Linear 工单体系。
- 发布节奏：CHANGELOG 已收 56 个版本条目（`grep -c '^## '`），rc.22 于 2026-10-30 发布，rc 周期逐日发版；每个 breaking change 附带编号的升级配方（app/extension recipe）。
- 贡献者：contributors API 仅 29 人；且 main 上不存在 2021-06 之前的提交（`commits?until=2021-06-01` 返回空）——v8 重写时 main 历史被重根，旧仓库十余年历史不在此分支上，故该数字不代表整个 Prisma 团队规模（是否另有大名单"未确认"）。仓库 open issues 2834（gh api）。
- 社区面：Discord、X(@prisma)、博客公告；README 有"extension authors 召集令"；issue 模板与 GitHub Discussions 自动关闭工作流齐备。

## 9. 亮点与风险

亮点：
1. **架构代际跨越**：contract-first + 确定性 IR + 结构化 Plan，把 ORM 从"黑盒生成代码"变成"开放数据契约"，天然服务 AI agent 场景（`ARCHITECTURE.md`）。
2. **彻底去 Rust 化**：查询引擎/迁移/驱动全 TS（0 Cargo.toml），驱动直接骑在 `pg`/`mongodb` 等社区标准驱动上，去掉引擎二进制下载这一历史性痛点。
3. **工程纪律罕见地强**：机器强制的分层边界（lint:deps）、237 份 ADR、以集成测试为证据的 scorecard 制度、fixture 契约 diff 门禁。
4. **扩展生态先行**：pgvector/PostGIS/ParadeDB/Supabase 等官方扩展与内核走同一条公共 SPI，`examples/` 里有完整组合示例。
5. **平滑迁移路径**：`contract-prisma7` 读取旧 schema、`db sign` 收编现有库、v7 分支 + 每日 CI 双轨维护 18 个月。

风险/局限：
1. **尚未 GA**：8.0.0-rc.22，RC 期仍允许 breaking changes（README 明示），生产采用需谨慎。
2. **功能仍在补课**：官方 scorecard 显示 MySQL 仅部分读写、SQLite 概念验证，相对 Prisma 7 的全数据库覆盖存在缺口。
3. **学习与迁移成本**：DSL/ORM 客户端 API 与 Prisma 7 差异大，rc 间频繁 breaking（如 DateTime 便利类型改造），升级依赖配方脚本。
4. **文档滞后**：大部分权威说明在仓库内 ADR/scorecard，prisma.io 文档站与快速演进的核心可能存在时间差（未确认）。

## 10. 总结评价

prisma/orm 是一次罕见的"顶星级项目自我重写"：以 contract-first 和纯 TS 运行时取代服役多年的"代码生成 + Rust 引擎"架构，用机器强制的分层与以集成测试为证据的功能矩阵维持重写期的工程秩序。对使用者，它当下适合 PostgreSQL/MongoDB 新项目与愿意跟随 RC 迭代的团队；对 ORM/数据层作者与 AI 工具链开发者，这套"契约 IR + 可组合 DSL + 中间件活动树"的分层实现（配合 237 份 ADR）是目前公开源码里最完整的一份参照实现，值得精读。
