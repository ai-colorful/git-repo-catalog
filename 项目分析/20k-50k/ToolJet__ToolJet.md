# ToolJet/ToolJet 项目分析

> 星数 38,500（快照 2026-10-24，分析日 gh api 实测 41,228）｜主语言 JavaScript（实为 TypeScript/JavaScript）｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：开源低代码内部工具平台（Retool 的开源替代），通过可视化画布 + 90+ 数据源连接器构建管理面板、仪表盘和业务应用，现已叠加 AI 建应用与 MCP agent 集成。

| 项目 | 内容 |
|---|---|
| 名称 | ToolJet/ToolJet |
| 星数 | 41,228（2026-11-14 gh api 实测；派发快照 38,500） |
| 主语言 | TypeScript + JavaScript（清单标注 "JavaScript (Ruby)" 有误，源码树 0 个 .rb 文件、无 Gemfile，`find` 全仓验证） |
| 许可证 | AGPL-3.0（`LICENSE` 文件全文 + gh api license 字段一致） |
| 默认分支 | main（gh api） |
| 最近 push | 2026-11-13（分析前一天，高度活跃） |
| 贡献者 | 约 407（gh api contributors 分页计数） |
| Forks / Open issues | 5,507 / 957（gh api） |
| 最新版本 | v3.21.96-lts（2026-11-13）+ v3.20.257-lts + v3.21.95-beta 双轨发布 |

## 2. 项目是什么，解决什么问题

ToolJet 解决企业内部工具开发的"重复造轮子"问题：团队需要大量 admin panel、审批面板、运营仪表盘，传统方式是每个都写一个前后端；ToolJet 把这一过程压缩为"连接数据源 → 拖拽组件 → 绑定查询"。README.md 开篇即定位为 "open-source platform for building internal tools"，自托管或使用官方云。

与普通表单构建器不同，ToolJet 的核心是**查询中心**架构：每个应用由页面、组件和一组可复用的 data query 组成，query 绑定到 PostgreSQL、MongoDB、REST API、S3 等外部系统，组件事件（点击/变化）通过 event handler 触发 query 并把结果注入组件状态（README.md Features 节、`server/src/entities/data_query.entity.ts`）。数据流默认走服务端代理（仅凭据加密存储于服务端），不把数据库凭据暴露给浏览器（README "Secure by Design: proxy-only data flow"，机制见 `server/src/modules/data-queries/util.service.ts`）。

2025-2026 年的项目重心明显转向 **AI-native**：README 主标题区第一屏就是 "Build with your coding agent"——ToolJet 内置 ToolJet AI（自然语言生成页面/查询，Enterprise 功能）并自带 MCP server，使 Claude Code/Codex/Cursor 等编码 agent 通过 ToolJet 的组件与数据契约直接构建应用，而不是生成自由代码。仓库本身也高度 AI 化：根目录有 `AGENTS.md`、`.agents/context/architecture-map.md`（159 行自维护架构图）、`.claude/`，且 20+ 个 agent skill（`.agents/skills/`，含 create-plugin、cut-release、review-pr 等）。

商业化模式：社区版（CE，AGPL-3.0，本仓库主体）+ 企业版（EE）。EE 代码不在开源仓库中，但仓库内保留了完整的 EE 扩展点（见第 5 章）。企业版叠加 AI 建应用、RBAC/SCIM、审计日志、GitSync、多环境、白标、嵌入式应用等（README Community Edition vs ToolJet AI (Enterprise) 对照表）。

## 3. 技术栈与依赖

- **前端**（`frontend/package.json`，244 个依赖）：React 18 + react-router 6，状态管理以 Zustand 为主（`frontend/src/_stores/` 20 个 store），实时协作用 Yjs（yjs 13.5）；构建从 webpack 迁移到 Rspack（`frontend/rspack.config.js`，webpack 保留为后备）；样式 Tailwind 3 + SCSS；内嵌 Python 运行时 Pyodide 0.23.2（`frontend/assets/libs/pyodide-0.23.2/`，whl 合计约 900MB，是 tarball 体积主因）
- **后端**（`server/package.json`，181 个依赖）：NestJS 11 + TypeORM 0.3 + PostgreSQL（pg 8）+ ioredis/BullMQ 队列 + bcrypt 加密 + socket.io WebSocket；ORM 配置双连接：主库 + ToolJet DB（`server/ormconfig.ts`）
- **连接器**（`plugins/packages/common/`）：knex 作 SQL 方言层、ssh2 隧道、OAuth 工具；每个连接器是独立 lerna/npm 包
- **工程化**：npm 10 + Node 22（根 `package.json` engines），TypeScript 5.9，ESLint 9 flat config，husky + lint-staged；monorepo 非 npm workspaces，而是 prefix-script 编排（根 `package.json` scripts 全部 `npm --prefix xxx run`），plugins/marketplace 内部用 lerna

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `frontend/` | React 应用：编辑器（AppBuilder）、发布/嵌入应用 Viewer、ToolJet DB UI、WorkflowEditor、市场页（4951 文件；源码约 2600 ts/jsx，含 pyodide 资产约 930MB） |
| `server/` | NestJS API 与全部业务逻辑：`src/modules/` 下约 80 个 feature 模块 + `src/entities/` 60+ TypeORM 实体 + `migrations/`（3597 文件） |
| `plugins/` | 48 个内置连接器包 + `common/` 共享库，lerna 管理（693 文件） |
| `marketplace/` | 49 个可安装市场插件（独立于内置包，同构 manifest/operations/index） |
| `packages/widget-definitions/` | 组件定义契约（供 AI/agent 使用的组件 schema） |
| `cli/` | @tooljet/cli，脚手架生成新连接器（`bin/`、`src/`、自带测试） |
| `cypress-tests/` | 513 个 e2e 测试文件，按 platform/appbuilder/marketplace/workflows/tooljet-database 五条流水线拆分 |
| `docs/` | 文档站点 mdx 源（710MB，主要为图片，本次未解压细看） |
| `docker/`、`deploy/` | 生产镜像、Kubernetes/Helm/OpenShift 部署定义 |
| `.agents/`、`.claude/`、`AGENTS.md` | AI 协作配置：自维护架构图、产品图、20+ agent skills |

组织方式评述：典型多包 monorepo，前后端边界清晰（React SPA + API server），连接器以"插件包"为第一公民。值得注意的特殊安排：`frontend/ee/` 与 `server/ee/` 目录在本 tarball 中为空（`ls` 验证），EE 实现代码私有，开源仓库只保留 CE 实现和动态装配机制；根目录另有游离的 `queryPanel/`（3 个条目，疑似误提交的草稿目录）。

## 5. 核心架构与关键实现

### 5.1 CE/EE 同仓分层的运行时装配（server）

整个 server 以动态模块装配实现单代码库多版本：所有 feature 模块继承 `SubModule`，运行时按 `TOOLJET_EDITION` 环境变量决定从 CE 还是 EE 路径 import provider，并按"子类+配置签名"缓存 DynamicModule 避免重复装配：

```ts
// server/src/modules/app/constants/index.ts
export const getImportPath = async (isGetContext?: boolean, edition?: TOOLJET_EDITIONS) => {
  // ... CE: `${join(process.cwd(), baseDir, 'src')}`
  //     EE: `${join(process.cwd(), baseDir, 'ee')}`   （EE 目录私有，开源包中为空）
};

// server/src/helpers/utils.helper.ts:579
export function getTooljetEdition(): string {
  if (process.env.TOOLJET_EDITION) { ... }
```

`server/src/modules/app/sub-module.ts` 的 `buildCacheKey` 显式把 edition 纳入缓存键，注释说明是为了支持测试在同一进程对 'ee'/'ce' 双注册。这解释了为何 `workflows` 等 CE 模块部分方法是 stub（`.agents/context/architecture-map.md` 自述），完整逻辑在私有 EE 中。

### 5.2 连接器协议：双 JSON 声明式契约（plugins）

每个连接器是同构的 5 文件包：`lib/manifest.json`（数据源连接表单，JSON Schema 2020-12 + `tj:source`/`tj:encrypted` 扩展）、`lib/operations.json`（查询操作 UI）、`lib/index.ts`（实现）、`types.ts`、`icon.svg`。以 postgresql 为例（`plugins/packages/postgresql/lib/`）：

```json
{
  "$schema": "https://json-schema.org/draft-07", 
  "tj:source": { "name": "PostgreSQL", "kind": "postgresql", "type": "database" },
  "properties": {
    "connection_type": { "enum": ["manual", "string"], "default": "manual" },
    "password": { "type": "string" }
  },
  "tj:encrypted": ["password"]
}
```

`operations.json` 用 `dropdown-component-flip` 控件实现"SQL mode / GUI mode"切换动态表单（`plugins/packages/postgresql/lib/operations.json`）。前端据此零代码渲染连接器 UI，server 只需调用统一接口（`plugins/packages/common/lib/query_service.interface.ts`）：

```ts
export interface QueryService {
  run(sourceOptions, queryOptions, dataSourceId?, dataSourceUpdatedAt?, context?): Promise<QueryResult>;
  getConnection?(...): Promise<object>;
  testConnection?(sourceOptions): Promise<ConnectionTestResult>;
  invokeMethod?(...): Promise<unknown>;
}
```

GUI 模式的 SQL 拼装抽到共享查询构建器 `plugins/packages/common/lib/queryBuilder.ts`（operator 白名单 eq/neq/gt/in/between、聚合函数集合、多 dialect 生成）。新增连接器的成本极低——CLI 脚手架 `cli/` 一条命令生成五件套，这直接支撑了 90+ 数据源生态（48 内置 + 49 市场）。

### 5.3 查询执行链路（server）

浏览器 QueryManager 构造的 query 经 REST 到达 `server/src/modules/data-queries/util.service.ts:70` 的 `runQuery`：解析 appVersion/branch（分支感知，workspace-branches 功能）、按环境（`app-environments`）取加密凭据 `dataSourceOptions`、`fetchServiceAndParsedParams` 动态 import 对应连接器包得到 service、执行 `run`、收集 `collectedSecretValues` 在 finally 中从结果元数据中剥离明文密钥（该文件 840 行，含节流 `throttler/` 与状态追踪 `DataQueryStatus`）。凭据隔离的另一层：数据源选项按 environment 存储在 `data_source_options.entity.ts`，AES-256-GCM 加密由 `modules/encryption/` 提供。

### 5.4 前端编辑器组织（frontend）

`frontend/src/AppBuilder/` 是编辑器核心：`AppCanvas/`（373 行画布 + Grid 栅格 + Selecto 多选 + WidgetWrapper/RenderWidget 渲染层）、`Widgets/` 115 个组件目录（README 宣称 80+，实际目录更多，含新旧并存：Table/NewTable、Dropdown/DropdownV2）、`QueryManager/`+`QueryPanel/`（查询编辑）、`_stores/` 20 个 Zustand store（appDataStore/editorStore/currentStateStore/dataQueriesStore 等）、`RealTimeEditor/`（Yjs 多人协作）。Viewer 侧 `frontend/src/ViewerApp.jsx` 按 `/applications/*` 与 `/embed-apps/*` 路由加载轻量发布应用 bundle（根架构图 `frontend/src/RootRouter.jsx`）。组件契约沉淀在 `packages/widget-definitions/`，供 MCP agent 使用。

### 5.5 ToolJet Database（内置数据库）

CE 自带 no-code 数据库：独立 PostgreSQL 连接（`server/ormconfig.ts` 的 `buildToolJetDbConnectionOptions`），每 workspace 独立 schema，行级 CRUD 通过 server 签发 JWT 后代理到 PostgREST（`server/src/modules/tooljet-db/`，PostgrestProxyService），元数据与数据面分离。

## 6. 构建与运行

```bash
# 生产构建（根 package.json）：连接器 → 前端 → 后端 顺序
npm run build          # = build:plugins:prod + build:frontend + build:server
npm run start:prod     # node server/dist/src/main
npm run db:migrate     # TypeORM 迁移（npm --prefix server run db:migrate）

# 本地开发（docker-compose.yaml，三容器 plugins/client/server + postgres）
docker compose up      # client 暴露 8082（rspack serve --hot）

# 快速体验（README.md）
docker run --name tooljet -p 80:80 tooljet/try:ee-lts-latest

# 测试
npm --prefix server run test        # Jest 单元（227 个 spec 文件）
npm --prefix server run test:e2e    # scripts/run-e2e.sh
npm --prefix frontend run test      # Jest（258 个 test/spec 文件）
# cypress-tests/ 五条独立流水线（ci.yml 引用 5 个 cypress-*.yml workflow）
```

## 7. 测试与工程质量

- 测试金字塔完整：server Jest 单元/集成 227 个 spec + e2e（`server/test/jest-e2e.config.ts`）；frontend 258 个测试文件；cypress e2e 513 个文件按五条业务线拆分；`plugins/`、`marketplace/` 各有少量测试（3/4 个）与 `__tests__/`
- CI（`.github/workflows/ci.yml`，1561 行）：dorny/paths-filter 按 server/frontend/plugins/marketplace/migrations 变更门控下游任务，支持 `run-ci-changed` 标签的"快车道"（只测变更模块），main PR 全量模式；配套 36 个 workflow：CodeQL、license-compliance、dependency-vulnerability-gate、release-vulnerability-gate、grype 通知、ai-tests（AI 相关回归）等
- 代码质量观察：lint-staged + husky 强制双端 ESLint；前端仍以 .jsx 为主（1861 个 vs tsx 29 个），类型化程度偏低但近期代码（server 模块）注释质量高（如 sub-module.ts 的缓存键注释、util.service.ts 的密钥剥离注释）；根 `package.json` 通过 overrides 钉住 js-yaml/tmp 版本修复漏洞
- 仓库自带 `UBIQUITOUS_LANGUAGE.md`（领域术语表）与 `THIRD-PARTY-NOTICES`，工程文化规范

## 8. 社区与维护状态

- 高度活跃：最近 push 2026-11-13（分析前一天）；gh api 显示 2026-11-06 至 11-13 间 5 个 release
- 发布节奏：LTS（v3.20.x-lts）+ beta（v3.21.x-beta）双轨并行，近乎每日出包（`gh api releases`：11-13 v3.21.96-lts、11-12 v3.20.257-lts 与 v3.21.95-beta）；docker-release.yml、updating-dockertag.yml 自动化发布
- 贡献者约 407 人（gh api contributors 分页），有 CODEOWNERS 与 `.agents/skills/commit`、`create-pr` 等 AI 辅助贡献流程
- 开源/商业双引擎：CE 功能在 README 中明确列举，EE 功能（RBAC、SCIM、审计、GitSync、白标）为付费卖点；.agents/context/product-map.md 记录产品边界（本次未逐行核读）
- 许可历史：当前 LICENSE 为 AGPL-3.0，gh api license 字段一致；README 徽章亦标 AGPL；本次未发现许可变更证据（未做 git 历史核查，标注"未确认"——任务提示的"AGPL→许可变更"未在本仓库快照中得到支持）

## 9. 亮点与风险

**亮点**
1. 连接器即插件包：manifest.json + operations.json 双 JSON 声明式契约让前端零代码渲染连接器 UI，server 侧仅依赖 3 方法的 `QueryService` 接口，社区扩展成本极低（`plugins/packages/`、`plugins/packages/common/lib/query_service.interface.ts`）
2. 单仓多版分层：SubModule + getImportPath 动态装配使开源仓库能携带完整 EE 扩展点而不泄漏私有代码，CE 用户可用全部基础设施（`server/src/modules/app/sub-module.ts`）
3. 测试与安全工程扎实：单元/集成/e2e 三层 + 五条 cypress 流水线 + CodeQL + 依赖与许可双重门禁 + 查询执行密钥剥离（`server/src/modules/data-queries/util.service.ts`）
4. AI-native 布局超前：MCP server、组件契约（packages/widget-definitions）、20+ agent skills、自维护 architecture-map，是少数把编码 agent 当一等公民的开源低代码平台（README、`.agents/`）
5. LTS + beta 双轨高频发布，生产升级路径清晰

**风险/局限**
1. 仓库物理体积失控：pyodide whl（约 900MB）与 docs 图片（约 710MB）直接提交入库，tarball 863MB，clone/CI 负担重（`frontend/assets/libs/pyodide-0.23.2/`）
2. AGPL-3.0 对想二次分发/集成的商业用户有强传染性约束，SaaS 化改造需注意第 13 条网络服务开源义务
3. 前端类型化债务：1861 个 .jsx 且新旧组件并存（Table/NewTable、Dropdown/DropdownV2 双版本维护），核心编辑器长期演进成本高（`frontend/src/AppBuilder/Widgets/`）
4. 部分能力 CE 是空壳：workflows 等 CE 模块存在 stub，完整功能需 EE 许可（`.agents/context/architecture-map.md` 自述；`server/src/modules/workflows/`）

## 10. 总结评价

ToolJet 是目前功能完整度与工程成熟度最高的开源 Retool 替代品之一：以"声明式连接器契约 + 查询中心执行链 + CE/EE 运行时分层"三件套支撑了 97 个数据源生态和日均发版节奏，测试、CI、安全门禁的完备程度在同类开源项目中属第一梯队；当前明显押注 AI-native（MCP + agent skills + AI 建应用）作为第二增长曲线。适合需要自托管内部工具平台的企业直接采用（注意 AGPL 与 EE 功能边界），也适合学习低代码平台架构的工程师精读——推荐入口为 `server/src/modules/app/sub-module.ts`（版本分层）、`plugins/packages/common/lib/query_service.interface.ts`（连接器抽象）、`server/src/modules/data-queries/util.service.ts`（查询执行）与 `frontend/src/AppBuilder/`（编辑器组织）。任务清单标注的主语言 Ruby 不属实，实测语言构成为 TypeScript/JavaScript（0 个 Ruby 文件），星数快照滞后约 2,700（当前 41,228）。
