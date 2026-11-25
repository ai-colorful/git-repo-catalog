# supabase/supabase 项目分析

> 星数 113711（快照 2026-10-24）｜主语言 TypeScript｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：Supabase 是"基于 Postgres 的开发平台"，用企业级开源工具（PostgREST、GoTrue、Realtime 等）拼出 Firebase 式的开发体验；本仓库是其**门面仓库**——承载管理面板 Studio、官网、文档站、TS 端元数据库 `pg-meta`、自托管 Docker 编排与官方示例。

| 项目 | 事实 |
|---|---|
| 名称 | supabase/supabase（master 分支，`gh api` 确认） |
| 星数 | 113,711（任务快照） |
| 语言 | TypeScript（pnpm monorepo，Node >=22.13） |
| 许可证 | Apache-2.0（根 `package.json`；`packages/pg-meta` 为 MIT） |
| 创建/最近 push | 2019-10-12 / 2026-10-29（`gh api repos/supabase/supabase`） |
| 贡献者 | 约 2,122（GitHub contributors API，anon=true） |
| Release | GitHub Release 为月度"Developer Update"（最新 v1.26.08，2026-08-07）；自托管 compose 发布独立版本号（最新 self-hosted/v0.8.3，2026-10-16，`docker/CHANGELOG.md`） |

## 2. 项目是什么，解决什么问题

Supabase 把"从数据库到 API"这条链路产品化：给一个 Postgres 实例自动生成 REST（PostgREST）与 GraphQL（pg_graphql）API，配上 JWT 认证（GoTrue）、数据库变更实时推送（Realtime，Elixir）、S3 对象存储（storage-api）、Deno 边缘函数（edge-runtime）与连接池（Supavisor），开发者不用写后端即可上线应用（`README.md`"How it works"一节）。

需要特别说明**本仓库与核心组件的边界**：真正干活的内核大多不在本仓库——PostgREST、supabase/auth（GoTrue）、supabase/realtime、supabase/storage、supabase/postgres-meta、supabase/edge-runtime、supabase/supavisor、supabase/pg_graphql 及其定制的 supabase/postgres 都是独立仓库。本仓库的组合形态是"**面板 + 内容站 + 自托管编排 + 示例**"（`docker/README.md` 的服务清单逐一链接到外部仓库）。

本仓库内最大的两块实体代码是：`apps/studio`（管理面板，部署于 supabase.com/dashboard，也随自托管镜像发布，见 `docker/versions.md` 中的 `supabase/studio:2026.10.12`）和 `packages/pg-meta`——一个纯 TypeScript 实现的 Postgres 目录（catalog）内省 SQL 构造库，是 Studio 读写数据库元数据的底层依赖（`packages/pg-meta/README.md`、`packages/pg-meta/src/pg-meta-tables.ts`）。

商业与开源的关系：托管平台是商业收入来源，自托管 Docker Compose（`docker/`）与本地 CLI 开发（`supabase/config.toml`、示例）保证开源可用性。仓库还内置了大量面向 AI 代理的基建——根目录 `AGENTS.md`/`CLAUDE.md`/`.agents/skills/`（约 20 个 SKILL.md，覆盖 API 类型、文档编辑、studio 测试规范等），以及 Studio 内嵌的 AI Assistant（见第 5 节）。

## 3. 技术栈与依赖

- **工程化**：pnpm 11.13.1 workspace + Turborepo 2.9.14（`package.json`、`turbo.jsonc`、`pnpm-workspace.yaml`），Node >=22.13（`.nvmrc`），共享依赖版本走 pnpm `catalog:` 协议。
- **Studio**：Next.js（pages router）与 TanStack Start（Vite + `@tanstack/react-router`/`react-query`/`react-table`/`react-virtual`）**双运行时并行**，正在做框架迁移（`apps/studio/TANSTACK_MIGRATION.md`、`apps/studio/package.json`）；UI 用自家 `@supabase/ui`、`ui-patterns`，编辑器 Monaco + Lexical，监控 Sentry，特性开关 ConfigCat，支付 Stripe。
- **AI 栈**：Vercel AI SDK（`@ai-sdk/openai`、`@ai-sdk/amazon-bedrock`、`@ai-sdk/mcp`）、MCP 官方 SDK 与 `@supabase/mcp-server-supabase`、Braintrust 评测（`apps/studio/package.json` dependencies 与 evals 脚本）。
- **pg-meta**：仅依赖 zod，自研 `pg-format`（带 `safeSql` 防 SQL 注入的标识符/字面量拼接，`packages/pg-meta/src/pg-format/index.ts`）。
- **文档站/官网**：Next.js（docs 用 `next build --webpack`），docs 有 markdown 生成、向量嵌入（`embeddings` 脚本）等构建链（`apps/docs/package.json`）。
- **测试/E2E**：Vitest（单元，coverage v8）+ Playwright 1.59（`e2e/studio/package.json`）。
- **自托管**：Docker Compose 13 个服务 + 多个网关/存储/PG 版本 override（`docker/docker-compose*.yml`）。

## 4. 目录结构与代码组织

| 目录 | 职责 | 规模（文件数） |
|---|---|---|
| `apps/studio` | 管理面板（云平台 + 自托管共用），最大代码块 | ~6,760 |
| `apps/www` | 官网/营销站（Next.js + 博客 + changelog） | ~6,223 |
| `apps/docs` | 文档站（MDX 内容 + 生成式 API 参考） | ~2,981 |
| `apps/ui-library`、`apps/design-system` | 组件展示站与设计系统 | ~1,364 |
| `apps/kb`、`apps/learn`、`apps/lite-studio` | 知识库、教程站、精简版面板 | ~411 |
| `packages/pg-meta` | Postgres 目录内省 SQL 构造库（TS 重写版 postgres-meta 核心） | 167 |
| `packages/ui`、`packages/ui-patterns` | 基础 UI 与业务组件库（含 grid、icons、图表封装） | ~614 |
| `packages/api-types` | 平台管理 API 的 OpenAPI 类型生成 | 18 |
| `packages/ai-commands`、`packages/docs-search` 等 | AI 命令、文档检索、icons、config、eslint-config、shared-data 等 16 个包 | — |
| `docker/` | 自托管 Docker Compose 编排 + CHANGELOG/versions + 测试 | 83 |
| `supabase/` | 本仓库自己的本地开发配置（config.toml、migrations、functions、seed.sql） | 72 |
| `examples/` | 官方示例应用（ai、auth、realtime、edge-functions、slack-clone 等 20+ 主题） | ~2,144 |
| `e2e/` | Playwright E2E（studio/docs/www/kb 四套） | 143 |
| `.github/workflows/` | CI（60 个 workflow，见第 7 节） | 66 |

组织方式评述：典型的 **turborepo 多包单仓**，按"应用(apps)/共享包(packages)/内容(examples)/编排(docker)"分层；`apps/studio` 内部再按 `pages|routes`（双运行时路由）、`components/grid`（表格编辑器）、`data`（112 个目录的按领域组织的数据层，TanStack Query 风格）、`lib`、`state` 分层。遗留痕迹较少（`components/to-be-cleaned` 仅 3 个文件）。

## 5. 核心架构与关键实现

### 5.1 pg-meta：TS 版 Postgres 内省层（`packages/pg-meta/src/`）

每个 `pg-meta-*.ts`（tables/policies/roles/triggers/…约 22 个域）产出"查 catalog 的 SQL + zod schema"：SQL 查询结果经 zod 校验成类型安全对象，写操作则由 `query/Query.ts` 链式 API（`QueryFilter`/`QueryModifier`）生成对应 DDL。亮点是自研 `pg-format` 的 `safeSql` 模板标签，把标识符/字面量的转义收口在一处（`pg-meta-tables.ts` 顶部 import）。

```ts
// packages/pg-meta/src/pg-meta-tables.ts（节选）
const pgTableZod = z.object({
  id: z.number(),
  schema: z.string(),
  name: z.string(),
  rls_enabled: z.boolean(),
  rls_forced: z.boolean(),
  replica_identity: z.enum(['DEFAULT', 'INDEX', 'FULL', 'NOTHING']),
  bytes: z.number(),
  ...
})
```

`sql/tables.ts` 的文档注释展示了相当深的 Postgres 查询优化功底：单表查询时传 `targetOid` 标量片段，让 planner 按 initplan 走 INDEX scan，避免全 catalog 计算 PK/FK/体积（对比 CTE 物化造成的优化屏障），这是对真实性能问题的工程化回应。

### 5.2 Studio 双运行时迁移（`apps/studio/`）

`TANSTACK_MIGRATION.md` 记录了非常规做法：Vite/TanStack 构建是**当前生产默认**，Next.js 构建保留为可回退旁路；`pages/`（197 个 tsx）与 `routes/`（222 个 tsx）并存，Path A 页面在 routes 里 re-export pages 的默认导出，因此 Next 文件对两个运行时都是"承重墙"，禁止在迁移期删除；还配了 CodeRabbit 规则（`.coderabbit.yaml`）对任何 `pages/**` 改动发提醒，要求镜像同步到 `routes/**`。这是一份值得框架迁移团队细读的活样本。

### 5.3 Studio 数据层与 AI Assistant（`apps/studio/data/`、`apps/studio/lib/ai/`）

`data/` 下 112 个按产品域划分的目录（branches、api-keys、compute、custom-domains…），统一走 `@tanstack/react-query`，配合 `packages/api-types` 的管理 API 类型（`package.json` 中 `api:codegen` 脚本由 OpenAPI 生成并在 CI 里 `verify-production-api-types` 校验）。AI 侧，Studio 内嵌 Assistant：依赖 Vercel AI SDK 接 OpenAI/Bedrock，接 MCP server（`@supabase/mcp-server-supabase`），并用 Braintrust 跑离线评测与线上 scorer（`apps/studio/package.json` 的 `evals:*`、`scorers:deploy` 脚本，`apps/studio/evals/`）。

### 5.4 自托管编排（`docker/`）

`docker-compose.yml` 定义 13 个服务：studio、api-gw（Envoy，默认）、auth（GoTrue）、rest（PostgREST）、realtime、storage(+imgproxy)、meta（postgres-meta）、functions（edge-runtime + deno-cache）、db（PG17 + db-config）、supavisor。可选 override 覆盖网关（envoy/kong/nginx/caddy）、PG 版本（pg15/pg17）、存储后端（rustfs/s3-MinIO）与连接池（pgbouncer）；`CHANGELOG.md` 明示 kong、MinIO、pg15 即将废弃。`update.sh`/`run.sh`/`setup.sh` 提供运维入口，`volumes/` 存放各服务的配置（如 kong/logflare 配置）。

## 6. 构建与运行

来自根 `package.json` scripts（Turbo 编排）与 `docker/` 文档：

```bash
# 本地面板开发（需先起本地栈）
pnpm install                      # 强制 pnpm（preinstall only-allow）
pnpm setup:cli                    # supabase start -x studio 并生成 env
pnpm dev:studio                   # turbo dev --filter=studio
pnpm dev:studio-local             # NODE_ENV=test 本地模式
# 构建与检查
pnpm build:studio                 # turbo build --filter=studio
pnpm typecheck && pnpm lint
pnpm test:studio                  # vitest --run --coverage
pnpm e2e:setup:cli && pnpm e2e    # Playwright（e2e/studio）
# Studio 自托管镜像
pnpm build:studio:docker          # docker build -f apps/studio/Dockerfile
# 自托管整套栈
cd docker && cp .env.example .env && ./setup.sh && docker compose up -d
# pg-meta 包测试
pnpm --filter @supabase/pg-meta test   # test/run-tests.sh（需本机 PG）
```

`pg-meta` 测试通过 `test/run-tests.sh` 包装（起临时 Postgres 跑集成断言），说明它不是纯 mock 单测。

## 7. 测试与工程质量

- **单元测试**：Studio 内 854 个 `*.test.ts(x)`（全仓 `find` 统计），Vitest + coverage v8；packages 层另有 107 个（pg-meta、ui、ui-patterns、common 等）。
- **E2E**：四套 Playwright 工程（`e2e/studio`、`e2e/docs`、`e2e/www`、`e2e/kb`），studio E2E 依赖本地 `supabase start` 栈并使用 `otpauth`/`faker` 做账号与 MFA 场景。
- **CI**：`.github/workflows/` 下约 60 个 workflow，覆盖各包单测、studio E2E、docker 镜像发布（`publish_image.yml`、`studio-docker-build.yml`）、类型校验（`typecheck.yml`、`verify-production-api-types.yml`）、knip 死代码检查、prettier、typo 修复，以及少见的**Lint 棘轮**（`lint-ratchet.yml`、`studio-lint-ratchet.yml` + `apps/studio/scripts/ratchet-rules.json`，只许降不许升）和 **AI 评测**（`braintrust-evals.yml`、`ai-tests.yml`）。
- **工程安全**：`zizmor.yml`（GitHub Actions 静态扫描）、CSP/security headers 独立模块（`apps/studio/csp.ts`、`security-headers.ts`）、`.coderabbit.yaml` AI 代码评审。
- 观察：工程质量装置（棘轮、生产 API 类型防漂移、文档锚点检查 `docs-removed-anchors.yml`）密度明显高于一般开源项目，体现平台公司对"面板即生产环境"的防御姿态。

## 8. 社区与维护状态

- 活跃度：`pushed_at` 2026-10-29（分析前一天），1,179 个 open issue；contributors API 显示累计约 2,122 名贡献者（含匿名计数）。
- 发布节奏：本仓库 GitHub Release 不承载二进制，而是月度"Developer Update"（v1.26.08，2026-08-07）；工程上真正的发布物是 Studio Docker 镜像与自托管 compose，`docker/CHANGELOG.md` 显示 self-hosted 版本 2026 年已发布 0.6.x→0.8.3（约每月 1-2 次），`versions.md` 逐月记录镜像 bump（2026-09-09、2026-10-16…），下游组件（gotrue v2.197、postgrest v14.18、realtime v2.143 等）随版本表滚动升级。
- 贡献结构：主体为 Supabase 公司全职工程师（面板/文档/编排高频变更），外部贡献集中在 examples 与文档（未做 git 级别统计，属合理推断）。

## 9. 亮点与风险

**亮点**
1. 自研 pg-meta 把 Postgres 内省做成了带 zod 类型与安全 SQL 拼接的纯 TS 库，浏览器端即可安全生成/执行 catalog 查询（`packages/pg-meta/src/`）。
2. Studio 的 Next→TanStack 双运行时迁移工程学：双构建并存、可回退、re-export 策略 + AI 评审护栏（`apps/studio/TANSTACK_MIGRATION.md`）。
3. 自托管编排质量高：13 服务一键起，网关/PG/存储可插拔 override，版本表与 CHANGELOG 支持精确回滚（`docker/`）。
4. CI 工程化深：60 个 workflow、Lint 棘轮、生产 API 类型校验、Actions 安全扫描（`.github/workflows/`）。
5. AI 原生程度高：仓库级 agent skills、MCP server、LLM 助手离线评测进 CI（`.agents/`、`apps/studio/evals/`）。

**风险/局限**
1. 本仓库不含数据面内核（auth/realtime/storage 等在外部仓库），只看这个仓库无法理解全栈行为；`docker/versions.md` 的镜像版本才是真正的全局版本清单。
2. Studio 双运行时并存是过渡态：`pages/` 改动需人工镜像到 `routes/`，迁移完成前存在回归窗口（TANSTACK_MIGRATION.md 自述）。
3. 体量与门槛：pnpm catalog + turbo + 8 个 app + 16 个包，新贡献者冷启动成本高（`DEVELOPERS.md` 试图缓解）。
4. 自托管依赖十余个镜像版本组合，升级需跟随 compose 变更（CHANGELOG 中多处"requires update"提示）。

## 10. 总结评价

supabase/supabase 是"开源数据库平台"这一品类里星数最高、工程化最重的门面仓库：它不是数据库本身，而是把面板（Studio）、目录内省库（pg-meta）、文档/官网、自托管编排和示例组织成一个高纪律 monorepo，真正的服务内核分散在 Supabase 组织的十余个独立仓库中。维护极度活跃、测试与 CI 装置齐全，当前最大的技术看点是 Studio 向 TanStack Start 的双运行时迁移与 AI 助手的评测化工程。适合想自托管或二次开发面板的团队、想学习大型 turborepo 与框架迁移工程的前端工程师，以及想研究"浏览器端安全生成 Postgres SQL"的实现者阅读源码；想看认证/实时/存储内核源码的读者应转去 supabase/auth、supabase/realtime、supabase/storage 等仓库。
