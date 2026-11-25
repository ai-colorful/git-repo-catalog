# drizzle-team/drizzle-orm 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-17

## 1. 项目速览

一句话定位：Drizzle 是一个"长得像 SQL"的 TypeScript ORM——以原生 SQL 模板引擎为核心，无 Rust 引擎、无数据代理层，零运行时依赖、可 tree-shaking，配合独立 CLI（drizzle-kit）覆盖 schema 定义、迁移与 25+ 数据库驱动适配。

| 项目 | 内容 |
|---|---|
| 名称 | drizzle-team/drizzle-orm |
| 星数 | 39500（快照 2026-10-24；gh api 实测 36283，2026-11-17） |
| 语言 | TypeScript（纯 TS 实现，无本地二进制） |
| 许可证 | Apache-2.0 |
| 当前版本 | drizzle-orm 0.45.4 / drizzle-kit 0.31.11（main 分支快照）；v1.0.0-rc.5 已于 2026-10-21 发布 |
| 创建时间 | 2021-06-24 |
| 最近活跃 | git push 2026-11-16（gh api），持续高频发版 |
| 组织 | pnpm + turbo monorepo，9 个工作区包 |

## 2. 项目是什么，解决什么问题

Drizzle 自我定位是 "Headless ORM for NodeJS, TypeScript and JavaScript"（`README.md`），卖点是轻量（约 7.4kb min+gzip）、零依赖、可 tree-shaking，且"不需要 Rust 二进制、不需要 serverless 适配层，开箱即用"（`README.md`）。传统 ORM（如 Prisma）依赖查询引擎二进制或数据代理，在 Edge/Serverless 环境部署困难、冷启动慢；Drizzle 直接把查询构建与 SQL 生成放在 JS 层完成，每个驱动适配器只是薄薄一层会话抽象（如 `src/node-postgres/` 仅 4 个 ts 文件），因此能在 Node、Bun、Deno、Cloudflare Workers、Vercel Edge、浏览器等所有运行时直接运行（`README.md`、`drizzle-orm/package.json` 的 25+ peerDependencies 清单）。

Drizzle 的核心设计哲学是"不做 SQL 的抽象，而是 SQL 的类型化外壳"。它提供三层查询接口：一是 SQL-like 查询构建器（`db.select().from().where()`），写法贴近 SQL 本身而非发明私有查询语言；二是关系查询构建器 RQB（`db.query.users.findMany({ with: { posts: true } })`），用 JSON 配置做嵌套加载（`src/pg-core/query-builders/query.ts`）；三是任何地方都可以插入原生 `sql` 模板片段做逃生舱（`src/sql/sql.ts`）。这与 Prisma 的"自研 DSL + 引擎翻译"路线形成鲜明对比，也与 TypeORM 的装饰器 class 模型不同；相对 Kysely（纯类型化查询构建器、无 schema/迁移生态），Drizzle 补齐了 schema DSL、迁移工具与 Studio 图形界面，生态位介于 Kysely 与 Prisma 之间。仓库里直接内置 `src/kysely/`、`src/knex/` 类型兼容导出与 `src/prisma/` 驱动适配器，侧面印证它把"从这些工具迁移过来"当作明确目标。

生态由 monorepo 内多个包组成：ORM 核心（drizzle-orm）、迁移/内省 CLI（drizzle-kit）、四个 schema 校验适配器（drizzle-zod / drizzle-typebox / drizzle-valibot / drizzle-arktype，把表定义映射为对应校验库 schema）、数据填充工具（drizzle-seed）与自定义 lint 规则（eslint-plugin-drizzle）。README 还提及独立的 Drizzle Studio 数据浏览器与官网、Discord 社区。

## 3. 技术栈与依赖

- 语言与运行时：TypeScript 5.6.3（根 `package.json`），Node 20（CI `.github/workflows/release-latest.yaml`），同时适配 Bun/Deno/Workers。
- 工程：pnpm 10.6.3 + Turborepo（`pnpm-workspace.yaml`、`turbo.json`）、tsup 打包 + 自研 `drizzle-orm/scripts/build.ts` 与 `fix-imports.ts`、dprint 格式化（`dprint.json`）、eslint（含 `no-instanceof`、自研 `eslint-plugin-drizzle-internal` 规则）。
- 测试：vitest（单元）、tsc 编译期类型测试（`drizzle-orm/type-tests/` 80 个文件）、真实数据库集成测试（`integration-tests/`）。
- 运行时依赖：**0 个**（`drizzle-orm/package.json` 无 dependencies 字段，`sideEffects: false`）；数据库驱动全部为可选 peerDependencies（pg、postgres、mysql2、better-sqlite3、@libsql/client、@neondatabase/serverless、@aws-sdk/client-rds-data、@planetscale/database 等 25+，`drizzle-orm/package.json`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `drizzle-orm/` | ORM 核心，545 个 ts 文件（不含 dist） |
| `drizzle-orm/src/sql/` | SQL 模板引擎与表达式库（eq/and/or/like 等通用表达式） |
| `drizzle-orm/src/{pg,mysql,sqlite,gel,singlestore}-core/` | 5 个方言核心：列定义 DSL、dialect（SQL 生成器）、db、query-builders |
| `drizzle-orm/src/{node-postgres,postgres-js,mysql2,better-sqlite3,...}` | 约 25 个驱动/平台适配目录（每个仅 4-11 个文件：drizzle() 入口 + session） |
| `drizzle-orm/src/{kysely,knex,prisma}` | 第三方兼容/适配层 |
| `drizzle-orm/src/cache/` | 查询结果缓存抽象 + Upstash 实现（含按表自动失效） |
| `drizzle-orm/src/relations.ts` | 关系 API（Relations/One/Many + RQB 类型推导，725 行） |
| `drizzle-kit/` | 迁移/内省/推送 CLI（151 个 ts 文件），bin 命令独立分发 |
| `drizzle-zod/` 等 4 个 | 表定义 → zod/typebox/valibot/arktype 校验 schema 工厂 |
| `drizzle-seed/` | 测试数据填充 |
| `integration-tests/` | 真实多数据库集成测试（96 个测试文件） |
| `changelogs/` | 每个包独立手写 changelog |
| `eslint/`、`patches/`、`docs/` | 内部 lint 规则、第三方补丁、少量文档 |

组织方式评述：标准 pnpm monorepo，用 turbo 声明包间构建依赖（如 drizzle-kit 构建依赖 drizzle-orm 构建，`turbo.json`）。横向结构为"方言核心 + 驱动适配"两层分离：方言核心（pg-core 等）只管 SQL 生成与查询构建，驱动适配目录只管连接与执行（session），新增一个平台只需写一个薄 session 适配——这是它能快速覆盖 Neon/Turso/D1/PlanetScale 等新 serverless 平台的架构原因。

## 5. 核心架构与关键实现

### 5.1 SQL 模板引擎：一切皆 SQL 片段（`drizzle-orm/src/sql/sql.ts`）

`sql` 是全库的原子操作：它是一个 tagged template，把模板字符串与插值参数包装成 `SQL`/`Param`/`StringChunk` 树，任何列、条件、子查询都实现 `SQLWrapper` 接口，可任意嵌套组合。这是"SQL-like"定位的根基——库没有私有的查询 AST 概念，只有"带占位参数的 SQL 片段树"：

```ts
export function sql(strings: TemplateStringsArray, ...params: SQLChunk[]): SQL {
	const queryChunks: SQLChunk[] = [];
	if (params.length > 0 || (strings.length > 0 && strings[0] !== '')) {
		queryChunks.push(new StringChunk(strings[0]!));
	}
	for (const [paramIndex, param] of params.entries()) {
		queryChunks.push(param, new StringChunk(strings[paramIndex + 1]!));
	}
	return new SQL(queryChunks);
}
```

（出处：`drizzle-orm/src/sql/sql.ts:485`。同文件提供 `sql.identifier`、`sql.join`、`sql.raw`、`sql.empty` 等组合子。）

### 5.2 表定义 DSL 与类型系统（`drizzle-orm/src/table.ts`、`column.ts`、`column-builder.ts`）

表对象把元数据（表名、schema、列、别名、原始名）存在 `Symbol.for('drizzle:...')` 键下（`src/table.ts:20-40`），列类型通过带 brand 与 phantom 泛型的 `ColumnBuilder` 推导：`$inferSelect`/`$inferInsert` 声明属性直接从表泛型反推插入/查询类型。列 DSL 用"可传名/可不传名"双重载实现（`src/pg-core/columns/text.ts`），方言列目录覆盖 PostgreSQL 全类型谱系（含 point/line/ inet/macaddr 等 30+ 文件，`src/pg-core/columns/`）。运行时对象识别不走 `instanceof`，而是自研 `is()`：沿原型链比对 `Symbol.for('drizzle:entityKind')`（`src/entity.ts`），保证多副本/双 ESM-CJS 构建下类型判断依然正确——配合根 package.json 的 `eslint-plugin-no-instanceof` 强制执行。

### 5.3 方言层：Dialect 把配置对象拼成 SQL（`drizzle-orm/src/pg-core/dialect.ts`）

每个方言核心有一个 `Dialect` 类，查询构建器只负责收集 `PgSelectConfig`（fields/where/joins/orderBy/limit...），SQL 生成分离在 Dialect 里。`buildSelectQuery`（1445 行文件中的核心，约 341 行起）逐段拼装 where/having/order by/group by/limit/for update 片段，并带严格校验——引用了未 join 表的列会直接抛出指出具体列与路径的错误：

```ts
throw new Error(
	`Your "${f.path.join('->')}" field references a column "${tableName}"."${f.field.name}", but the table "${tableName}" is not part of the query! Did you forget to join it?`,
);
```

（出处：`drizzle-orm/src/pg-core/dialect.ts:355-370`。同类 dialect 文件：`mysql-core/dialect.ts` 1337 行、`sqlite-core/dialect.ts` 1047 行。）链式 API 有类型级"每个子句只能调用一次"约束：`where()` 返回 `PgSelectWithout<this, TDynamic, 'where'>`，从类型上抹掉已调用过的子句，`$dynamic()` 显式切换为可重复调用模式（`src/pg-core/query-builders/select.ts:765-771` 及类型定义）。

### 5.4 关系 API 与 RQB（`drizzle-orm/src/relations.ts` + `src/pg-core/query-builders/query.ts`）

关系在表定义之外单独声明：`relations()` 接收回调注入的 `one`/`many` 助手，生成 `Relations` 对象（`src/relations.ts:502-641`）。RQB 入口 `db.query.users.findMany(config)` 的 `RelationalQueryBuilder.findMany` 把嵌套 JSON 配置编译成 SQL（内部 JOIN/子查询策略），再由 `mapRelationalRow` 把扁平行折叠回嵌套对象（`src/relations.ts` 导出、`src/pg-core/query-builders/query.ts:31-44`）。类型层面通过 `DBQueryConfig` 与 `KnownKeysOnly` 保证 where/with 配置键必须在 schema 中存在，拼错即编译失败。

### 5.5 drizzle-kit：快照驱动的声明式迁移（`drizzle-kit/src/`）

kit CLI 基于 `@drizzle-team/brocli` 提供 generate/push/pull/migrate/check/drop/exportRaw/studio 命令（`drizzle-kit/src/cli/index.ts`），旧版按方言拆分的命令（generate:pg 等）已标记 deprecated。迁移采用"快照 diff"架构：用户 schema → `serializer/` 序列化为 JSON 快照 → `snapshotsDiffer.ts`（4331 行）用 zod 描述快照结构并对前后快照做 jsonDiffer → `sqlgenerator.ts`（4170 行）把语句差异生成 SQL 迁移文件；反向 `introspect-*.ts` 把已有库拉取成快照用于 push/pull。运行时迁移由方言内置 `dialect.migrate()` 执行：自动建 `__drizzle_migrations` 表、按时间戳逐条回放（`src/pg-core/dialect.ts:74-110`）。

## 6. 构建与运行

仓库内（来自根 `package.json`、`drizzle-orm/package.json`、`turbo.json`）：

```bash
pnpm install
pnpm build          # turbo: 全部包 build + test:types + lint
pnpm build:orm      # 仅 drizzle-orm
pnpm test           # turbo 各包测试（drizzle-orm 内为 vitest run）
pnpm lint           # dprint check；pnpm lint:fix 格式化
cd drizzle-orm && pnpm test:types   # type-tests 目录 tsc 编译期断言
```

drizzle-orm 构建链：先 `prisma generate`（为 prisma 驱动适配器生成 client），再 `scripts/build.ts`（tsup 出 CJS/ESM + dts，`resolve-tspaths` 修路径，`@arethetypeswrong/cli` 校验导出）。CI 中 `pnpm build` 后跑测试再发包（`.github/workflows/release-latest.yaml`）。

用户侧典型流程（README/docs）：`drizzle(driver, schema)` 初始化 → `drizzle-kit generate/push` 迁移 → `db.select()...` 查询。

## 7. 测试与工程质量

- **单元测试**：`drizzle-orm/tests/`（vitest）：casing（五种方言命名转换）、exports 导出面守门、is()、pg array 解析等，规模不大但打在关键点。
- **类型测试**：`drizzle-orm/type-tests/` 80 个 ts 文件，纯编译期断言（tsc 跑通即通过），是 TypeScript ORM 必须的重型保障。
- **集成测试**：`integration-tests/tests/` 96 个文件，按 pg/mysql/sqlite/singlestore/gel/xata/awsdatapi/relational/seeder/extensions/imports 分目录，`imports/` 子目录专门验证 ESM/CJS/bun 等多形态导入；`relational/` 覆盖 RQB。
- **CI**：`.github/workflows/` 含 codeql 安全扫描、release-latest（build+test+publish）、release-feature-branch、drizzle-pulse-publish、postgres-publish、router。
- **质量观察**：零依赖纪律（无 dependencies）、`sideEffects:false`、entityKind 统一对象识别、自研 eslint 内部规则插件、每个包独立 changelog 且逐条关联 issue/PR——工程纪律明显高于同体量社区项目。

## 8. 社区与维护状态

- **活跃度**：gh api 显示最近 push 2026-11-16，开放 issue 2056，持续高频维护。
- **发版节奏**：gh api releases 近一年时间线——0.45.2（2026-03-27）→ v1.0.0-beta.18~22（3-4 月，密集 beta）→ rc.1/rc.2（4-5 月）→ 0.45.3 + drizzle-kit@0.31.11（9-21）→ 0.45.4（10-08）→ v1.0.0-rc.5（10-21）。双轨并行：main 上 0.45.x 稳定线高频 patch，1.0 RC 从分支推进做破坏性清理（v1.0.0-rc.5 发布说明含 20+ 条 BREAKING 修复，均为收尾性质）。
- **贡献者结构**：gh api contributors——AndriiSherman 1067 次提交、dankochetov 841、L-Mario564 140、AlexBlokh 128、Sukairo-02 107。两个核心维护者贡献了绝大多数提交，公司化小团队模式（Drizzle Team 公司运营，README 有 sponsors 与 Discord）。
- **星数波动**：gh api 实测 2026-11-17 为 36283，低于任务快照 39500（2026-10-24），原因未确认（可能为快照口径差异）。

## 9. 亮点与风险

**亮点**

1. 架构轻盈且边界清晰：SQL 片段树（`sql/sql.ts`）+ 方言核心/驱动适配两层分离，新增平台成本极低，25+ 适配目录每个仅 4-11 个文件（`drizzle-orm/src/` 实测）。
2. 类型工程深：单次子句约束（`PgSelectWithout`）、`$inferSelect` 推导、type-tests 80 文件编译期保障，"SQL-like + 类型安全"兑现度高（`select.ts`、`type-tests/`）。
3. 迁移体系成熟：快照 diff + SQL 生成器双件共 8500 行（`drizzle-kit/src/`），支持 generate/push/pull/check 全流程与 5 方言内省。
4. 生态完整度高：zod/typebox/valibot/arktype 校验适配、seed、缓存自动失效（`cache/`）、OpenTelemetry tracing（`tracing.ts`）、Studio/官方文档站。
5. 交付节奏快且 changelog 透明（逐条 issue 关联，`changelogs/`），1.0 稳定版在收尾途中。

**风险/局限**

1. 贡献者高度集中于 2 人（gh api 数据），bus factor 偏低。
2. 0.x 期破坏性调整频繁：v1.0.0-rc.5 发布说明含 20+ BREAKING，升级成本集中在 0.4x→1.0。
3. 关系 API 处于演进期：main 快照（0.45.4）仍是经典 `relations()` + RQB，1.0 RC 重新打磨了过滤器语义（EmptyFilter、空条件跳过等），两代文档并存增加学习成本（v1.0.0-rc.5 发布说明）。
4. 开放 issue 2056 偏多（gh api），多方言 × 多驱动组合的 bug 面天然宽。

## 10. 总结评价

Drizzle 是近四年 TS ORM 赛道最有代表性的"反抽象"路线：把 SQL 留给用户、把类型安全和工程化（迁移/校验/Studio）补齐给生态，用零依赖薄层实现换来了 serverless 场景的统治级覆盖。对比 Prisma（引擎/代理、schema-first DSL），Drizzle 胜在轻量、SQL 直感与边缘运行时兼容；对比 TypeORM（装饰器、活跃度下滑），胜在维护节奏与现代 TS 支持；对比 Kysely（纯查询构建器），胜在有完整 schema/迁移/生态闭环。适合看重性能与部署灵活性、希望"写 SQL 但要类型"的后端/全栈团队，以及想在服务器less/Edge 上用关系数据库的场景；想读源码的人首选 `src/sql/sql.ts` → `src/pg-core/dialect.ts` → `src/table.ts` 三条主线，可完整看懂"类型化 SQL 片段树如何被拼装与执行"，`drizzle-kit/src/snapshotsDiffer.ts` 则是声明式迁移工程实现的好教材。
