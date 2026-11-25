# twentyhq/twenty 项目分析

> 星数 59581（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：Twenty 是一个可完全自定义的开源 CRM——通过"对象/字段元数据引擎"让用户像搭 Notion 表格一样自定义业务数据模型，并通过 SDK 把 CRM 配置当代码管理。

| 项目 | 事实 |
| --- | --- |
| 名称 | twentyhq/twenty（Twenty CRM） |
| 定位 | "The #1 Open-Source CRM"（README.md） |
| 星数 | 59581（任务快照）；forks 9914（gh api，2026-11-02） |
| 语言 | TypeScript（前后端全 TS） |
| 许可证 | AGPL-3.0 为主 + Enterprise 商业双许可（`LICENSE`，标注 `/* @license Enterprise */` 的文件走商业条款） |
| 默认分支 | main（gh api）；最近 push 2026-11-02（gh api，次日仍活跃） |
| 最新版本 | twenty/v2.54.0（2026-10-30 发布，gh api releases） |
| 贡献者 | 约 464 人（gh api contributors 分页推算，未逐页核实） |
| 代码规模 | 服务端 13,586 文件 / 约 123.7 万行 TS；前端 12,111 文件 / 约 88.3 万行 TS（find+wc 实测） |

## 2. 项目是什么，解决什么问题

Twenty 自我定位是"给技术团队的积木式 CRM"：传统 SaaS CRM（如 Salesforce）封闭且昂贵，Twenty 提供开源可自托管、数据模型可完全自定义的替代品（README.md "Why Twenty"）。产品形态上它不是一个"固定字段的 CRM"，而是一个**元数据驱动的 CRM 引擎**：公司、人员、机会只是预置的"标准对象"，用户可以在界面里继续创建任意自定义对象、字段、视图、权限，底层由服务端动态建表、动态生成 API（详见第 5 章）。

仓库近期演进方向（README.md + package.json 可见）：v2 阶段主打 **"CRM as Code"**——用 `twenty-sdk` 的 `defineObject()` 把对象/字段/视图声明成 TypeScript 代码，通过 `twenty app publish` 发布为可版本化、可复用的 App；配套 `twenty-cli`、`create-twenty-app`、App 市场（packages/twenty-apps），甚至提供 `twenty-agent-skills` 让 Claude Code/Cursor 等编码代理直接参与 CRM 配置开发（README.md 安装节、SKILLS.md）。

内置能力覆盖现代 CRM 全家桶：联系人/公司/机会管道、任务与笔记、邮件与日历同步（messaging/calendar 模块）、工作流引擎（workflow）、AI 代理（server 端集成 Vercel AI SDK 的 anthropic/openai/google/bedrock/xai 等十余家 provider，packages/twenty-server/package.json）、GraphQL + REST + MCP 三套 API（packages/twenty-server/src/engine/api/ 下 graphql/rest/mcp 目录）。商业模式为云托管 + 自托管双轨，企业特性（SSO、行级权限等）走商业许可。

## 3. 技术栈与依赖

**后端**（packages/twenty-server/package.json）：
- NestJS 12（@nestjs/common 12.3.0、@nestjs/graphql 14 patched、@nestjs/jwt、@nestjs/schedule）
- TypeORM 0.3.31（**打了 patch**：packages/twenty-server/patches/typeorm+0.3.31.patch，修改 PickKeysByType 等）；PostgreSQL 驱动 pg 8.12
- Redis：ioredis 5.10.1；队列：bullmq 5.78.0
- GraphQL 16.14.2 + @graphql-tools/schema（动态 schema 组装）
- Vercel AI SDK 全家桶（@ai-sdk/anthropic、openai、google、bedrock、azure、mistral、xai 等）；AWS SDK（S3/Lambda）
- ClickHouse 迁移脚本（packages/twenty-server/src/database/clickhouse/，用于事件/分析数据）

**前端**（packages/twenty-front/package.json）：
- React 19 + Apollo Client 4 + Vite 构建
- TipTap 3 富文本编辑器（含协作扩展）、nivo 图表、dnd-kit 拖拽、Monaco 编辑器
- lingui（i18n 多语言）、linaria（CSS-in-JS）、Stripe（计费前端）、Sentry

**工程设施**：yarn 4 workspaces + Nx 23.3.1 任务编排 + jest（单元/集成）+ cypress 类 e2e（packages/twenty-e2e-testing）+ Storybook（twenty-ui）+ knip（死代码检查，knip.json）。要求 Node ^26.5.1、强制 yarn（engines 里 npm: "please-use-yarn"，根 package.json）。

## 4. 目录结构与代码组织

Yarn workspaces + Nx monorepo，根 package.json 声明 19 个 workspace，磁盘上有 21 个包目录。按文件数排序的核心包：

| 目录 | 职责 | 文件数 |
| --- | --- | --- |
| packages/twenty-server | NestJS 后端（元数据引擎 + 业务模块） | 13,586 |
| packages/twenty-front | React 前端 SPA | 12,112 |
| packages/twenty-docs | 文档站（4051 文件，含 i18n 翻译） | 4,051 |
| packages/twenty-apps | 官方 App 示例/市场应用 | 3,506 |
| packages/twenty-ui | 独立 UI 组件库（含 Storybook） | 1,951 |
| packages/twenty-front-component-renderer | 前端组件远程渲染（@remote-dom） | 1,719 |
| packages/twenty-website | 官网（Next.js/OpenNext/Cloudflare） | 1,506 |
| packages/twenty-shared | 前后端共享类型/常量/工具 | 1,468 |
| packages/twenty-sdk / twenty-cli / twenty-client-sdk | App SDK、CLI、REST 客户端 | 271/748/117 |
| packages/twenty-docker | Docker Compose/Helm/K8s 部署物 | 116 |
| packages/twenty-zapier / twenty-emails / twenty-companion 等 | Zapier 集成、React Email 模板、AI 伴生工具 | <150 |

**服务端内部分层**（packages/twenty-server/src/）：
- `engine/metadata-modules/`：元数据引擎，80+ 子模块（object-metadata、field-metadata、view、page-layout、role、webhook、logic-function…），另有 40+ 个 `flat-*` 平坦化镜像模块
- `engine/core-modules/`：100+ 平台模块（auth/billing/jwt/sso/file-storage/message-queue/tool/upgrade…）
- `engine/twenty-orm/`：自研 workspace 作用域 ORM（查询/变更构建器、动态仓储）
- `engine/api/`：graphql（workspace schema builder/resolver builder）、rest（通用 CRUD 解析器）、mcp、open-api
- `engine/workspace-manager/`：workspace 建库、迁移构建/执行、标准对象预置
- `modules/`：24 个业务 CRM 模块（company/person/opportunity/task/note/messaging/calendar/workflow/timeline…）
- `queue-worker/`、`command/`、`database/`：独立进程入口与迁移命令

**前端内部结构**（packages/twenty-front/src/）：`modules/` 下 64 个领域模块（object-record、settings、views、workflow、command-menu…），`generated/`、`generated-metadata/` 存放从服务端 schema 生成的 GraphQL 类型，`pages/`、`hooks/`、`locales/`。组织方式是"按业务域垂直切分 + generated 类型桥接后端"，元数据驱动渲染贯穿始终。

## 5. 核心架构与关键实现

### 5.1 对象/字段元数据模型（自定义对象引擎的心脏）

每个 workspace 的"数据模型"本身存在一组核心库元数据表中。字段类型枚举共 25 种（TEXT、NUMBER、CURRENCY、RELATION、MORPH_RELATION、SELECT、ADDRESS、FULL_NAME、LINKS、PHONES、RICH_TEXT、TS_VECTOR 等，packages/twenty-shared/src/types/FieldMetadataType.ts）。字段定义实体是强类型的 TypeORM 实体，workspace 隔离与关系约束用数据库级索引/Check 表达（packages/twenty-server/src/engine/metadata-modules/field-metadata/field-metadata.entity.ts:50-71）：

```ts
@Entity('fieldMetadata')
@Check('CHK_FIELD_METADATA_MORPH_RELATION_REQUIRES_MORPH_ID',
  `("type" != 'MORPH_RELATION') OR ("type" = 'MORPH_RELATION' AND "morphId" IS NOT NULL)`)
@Unique('IDX_FIELD_METADATA_NAME_OBJECT_METADATA_ID_WORKSPACE_ID_UNIQUE',
  ['name', 'objectMetadataId', 'workspaceId'])
export class FieldMetadataEntity<TFieldMetadataType extends FieldMetadataType ...>
```

配套设计：每个元数据模块都有 `flat-*` 对应物（如 flat-object-metadata、flat-field-metadata），把嵌套实体拍平成 Map 供高频读取与缓存（engine/metadata-modules/ 下可见 flat-entity 公共层 + workspace-cache/workspace-cache-storage 缓存层）。标准对象（Company/Person/Opportunity 等）只是预置的 workspace-entity 类声明，如 CompanyWorkspaceEntity 定义 name/domainName/annualRevenue/address 及到 Person/Opportunity/Task 的关系（packages/twenty-server/src/modules/company/standard-objects/company.workspace-entity.ts:20-51），新 workspace 建立时由 workspace-manager 预置为元数据。

### 5.2 每 workspace 一个 Postgres schema + 元数据变更转 DDL

数据隔离采用"共享数据库、每 workspace 一个 PG schema"：`createWorkspaceDBSchema(workspaceId)` 直接 `CREATE SCHEMA`（packages/twenty-server/src/engine/workspace-datasource/workspace-schema.service.ts:34-47，并有 `WORKSPACE_SCHEMA_DDL_LOCKED` 热升级保护）。用户在界面创建自定义对象/字段时，变更先写元数据表，再经 workspace-migration-builder 生成迁移动作、由 workspace-migration-runner（含 action-handlers/jobs/registry，engine/workspace-manager/workspace-migration/）在实际 workspace schema 上执行 DDL 建表加列——即"用户操作即迁移"。

### 5.3 动态 GraphQL/REST/MCP API 生成

API schema 不是静态写死的，而是按 workspace 元数据实时组装（packages/twenty-server/src/engine/api/graphql/workspace-schema.factory.ts:22-70）：

```ts
const schemaSDLResult = await this.workspaceGraphqlSchemaSDLService
  .getOrComputeSchemaSDL(workspace, applicationId);   // SDL 缓存
const autoGeneratedResolvers = await this.workspaceResolverFactory.create(
  flatObjectMetadataMaps, flatFieldMetadataMaps, flatIndexMaps, idByNameSingular, ...);
const executableSchema = makeExecutableSchema({ typeDefs: parse(sdl), resolvers: {...} });
```

查询落地由自研 ORM 执行：workspace-select-query-builder.ts（1284 行）与 workspace-mutation-query-builder.ts 在 TypeORM 之上按元数据拼 SQL（packages/twenty-server/src/engine/twenty-orm/query-builder/）。同一套元数据还对外暴露 REST（engine/api/rest/ 下 filter/order/group-by/upsert 等十余个通用请求解析器）与 MCP server（engine/api/mcp/，AI 代理可直接操作 CRM 数据）。

### 5.4 前端：元数据驱动渲染

前端没有"公司页面/人员页面"这种硬编码视图，而是 object-record 模块族（45+ 子模块：record-table、record-board、record-calendar、record-inline-cell、record-filter、record-form…，packages/twenty-front/src/modules/object-record/）按元数据在运行时渲染表格/看板/日历；字段类型渲染器与 `src/generated/` 下的 GraphQL 类型配合 Apollo Client 4 缓存（object-record/cache/）。设置中心（modules/settings/）本身就是元数据的可视化编辑器。

## 6. 构建与运行

依赖 Node ^26.5.1 + yarn 4（root package.json engines；禁 npm）：

```bash
# 安装
yarn
# 一键起前后端 + worker（根 package.json scripts.start）
yarn start          # nx run-many -t start -p twenty-server twenty-front + wait-on + worker
# 单包构建
npx nx run twenty-front:build        # Vite 生产构建（NODE_OPTIONS=--max-old-space-size=8192）
npx nx run twenty-server:build       # NestJS 构建，产物 dist/
# 数据库初始化/迁移（server package.json）
yarn database:init:prod              # setup-db.js + run-instance-commands migrate --include-slow
# 自托管
cd packages/twenty-docker && docker compose up   # docker-compose.yml / helm / k8s
# App 开发（README.md）
npm install -g twenty && twenty app init my-app && twenty app publish --private
```

## 7. 测试与工程质量

- 规模：twenty-server 内 1791 个 `*.spec.ts/test.ts`；twenty-front 内 1660 个 spec/test/stories；另有 packages/twenty-e2e-testing 独立 e2e 包（find 统计）
- 风格：服务端单元测试与模块同目录共置，集成测试单列（twenty-server/test/integration/，并有 jest-integration.config.ts、jest-integration-secure.config.ts 和自定义 sequencer/reporter）
- CI 极重：.github/workflows/ 下 50+ 工作流——ci-server、ci-server-integration-test、ci-front、ci-ui（Storybook）、ci-e2e-main、ci-cross-version-upgrade（跨版本升级演练）、visual-regression-dispatch（视觉回归）、ci-breaking-changes、knip 死代码、多语言 i18n 自动化（docs/website i18n pull/push）、merge-queue 等
- 质量观察：根 package.json 用注释块逐条解释每个 resolutions 依赖 pin 的 CVE 与移除条件，供应链治理细致；TypeORM/@nestjs/graphql 靠 yarn patch 修上游，需随升级维护

## 8. 社区与维护状态

- 活跃度：仓库最近 push 2026-11-02（gh api），本次 tarball 文件时间戳 2026-11-03 01:30，处于每日提交状态；open issues 162（偏高审需求管控）
- 发布节奏：twenty/v2.54.0 于 2026-10-30 发布；版本号到 v2.54 表明高频小步发布（周更量级，未逐条核实历史间隔）
- 贡献者：contributors 分页至 465 页（每页 1 条），约 464 名贡献者；有 external-contributor-pr-auto-draft.yaml 等_contributor 流程自动化
- 商业实体：Twenty（twenty.com）全职团队维护，Discord/Figma 公开，roadmap 公示（README 徽章链接）

## 9. 亮点与风险

**亮点**
1. 元数据引擎设计完整且自洽：元数据表 → flat-entity 缓存 → 动态 DDL → 动态 GraphQL/REST/MCP schema 一条龙（5.1-5.3 章路径全部有源码对应），是"运行时自定义数据模型"的教科书实现
2. "CRM as Code" 闭环：defineObject SDK + CLI + App 市场 + agent-skills（README.md），把 CRM 配置纳入版本管理，契合技术团队
3. 工程化标杆：Nx monorepo、50+ CI 工作流、跨版本升级演练、视觉回归、依赖 CVE 逐条注释治理（.github/workflows/、根 package.json resolutions 注释）
4. 前后端同构 TS + 共享类型包（twenty-shared），元数据类型 25 种字段枚举前后端共用
5. AI 原生：服务端集成十余家 LLM provider，MCP server 让任意 AI 代理操作 CRM

**风险/局限**
1. 体量巨大（服务端 124 万行 + 前端 88 万行），二次开发与自审成本高；动态 schema + 自研 ORM + yarn patch 意味着排障链条长
2. AGPL-3.0 对商用集成不友好，企业功能（`/* @license Enterprise */` 标注文件）走商业许可，自托管可用功能边界需仔细核对 LICENSE
3. 每 workspace 独立 PG schema + 每次元数据变更即 DDL，大规模租户/高频改模型场景下的迁移风暴与锁表风险需运维验证（代码有 DDL 锁保护但未见基准数据）
4. 强依赖新版本 Node（^26.5.1）与 yarn 4，贡献者环境门槛偏高

## 10. 总结评价

Twenty 是当前开源 CRM 领域完成度与工程化水平最高的项目之一：它把"可自定义数据模型"做成了真正的运行时引擎（元数据 → DDL → API 全链路自动生成），再叠加 workflow、邮件/日历同步、AI 代理与 as-Code SDK，产品野心直指替代 Salesforce 的长尾需求。对使用者，它是值得认真评估的 Salesforce/HubSpot 开源替代，适合想完全掌控数据的 B2B 团队；对工程师，它的元数据引擎、workspace 级多租户隔离与动态 GraphQL 生成是学习"元数据驱动架构"的一手材料——但 200 万行量级决定了通读不现实，建议沿第 5 章给出的五个入口（field-metadata.entity.ts、workspace-schema.factory.ts、workspace-migration/、workspace-select-query-builder.ts、object-record 模块族）定点阅读。
