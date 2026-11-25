# outline/outline 项目分析

> 星数 36500（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：Outline 是一个面向成长型团队的知识库/Wiki——快速、实时多人协作、Markdown 兼容，React + Node.js 全栈 TypeScript 编写，既提供 getoutline.com 托管服务，也可自托管。

| 项目 | 值 | 出处 |
|---|---|---|
| 名称 | outline/outline | package.json `name: outline` |
| 版本 | 1.12.0（最新 release v1.12.0，2026-10-23 发布） | package.json / GitHub releases API |
| 星数 | 快照 36,500；查询时 GitHub 实时为 41,262 | 派发参数 / gh api repos/outline/outline |
| 主语言 | TypeScript（React 19 + Node.js） | package.json |
| 许可证 | BSL 1.1（Change Date 2030-10-23 后转 Apache 2.0；禁止用作竞品"Document Service"） | LICENSE |
| 最近活跃 | 最近 push 2026-11-13（查询当日仍在提交，非常活跃） | gh api repos/outline/outline |
| 代码规模 | 约 2,400 个 TS/TSX 文件：app 119k 行 / server 174k 行 / shared 61k 行 / plugins 29k 行（合计约 38 万行） | 源码统计 |
| 贡献者 | 约 253 人（contributors API 分页推算） | gh api contributors |

## 2. 项目是什么，解决什么问题

Outline 解决的核心问题是"团队知识的组织与协作写作"。它把文档组织成 Collection（集合）→ Document（文档树）两级结构，支持子文档无限嵌套（`server/models/Document.ts` 中 `parentDocumentId` 自引用）、拖拽排序（Collection 层用 `fractional-index` 库生成排序键，`server/models/Collection.ts`）、以及基于 Postgres tsvector 的全文检索（`plugins/search-postgres/`）。

与纯 Markdown 文件方案不同，Outline 走"富文本编辑 + Markdown 兼容"路线：编辑器基于 ProseMirror 深度定制（`shared/editor/`），支持表格、数学公式（@benrbray/prosemirror-math）、Mermaid 图、Diagrams.net 嵌入、代码块等（`shared/editor/nodes/`、`shared/editor/extensions/`），同时通过 `prosemirror-markdown` 与 `markdown-it` 实现 Markdown 导入导出（package.json 依赖）。文档内容以 ProseMirror JSON（`content` JSONB）+ Yjs 二进制状态（`state` BLOB）双形态落库（`server/models/Document.ts`）。

实时协作是它的招牌能力：所有打开同一文档的用户通过 WebSocket（路径 `/collaboration`，`server/services/collaboration.ts`）连接到基于 Yjs CRDT 的协作服务，光标/在线状态（awareness）同步，多人同时编辑无锁合并。这区别于早期 OT 方案，是 Outline 从自研协作层迁移到 Yjs + Hocuspocus 的结果（`server/collaboration/` 各 Extension）。

产品层面 Outline 定位商业开源：托管版 getoutline.com 为主要收入来源，仓库同时可 Docker 自托管（docker-compose + Dockerfile）。README 明确"如果只想用 Outline，不必运行这些代码，直接用托管版"。围绕文档还构建了完整生态：SSO（OIDC/Google/Azure AD/Slack/Discord/Passkeys/邮箱魔法链接）、集成（GitHub/GitLab/Gitea/Figma/Linear/Notion/Slack/Zapier/Import）、Webhook、API/OAuth、以及内建 MCP 服务器（`server/mcp/`）供 Claude 等 AI 客户端操作知识库。

## 3. 技术栈与依赖

| 层 | 技术 | 证据 |
|---|---|---|
| 前端 | React 19.3 + MobX 6（状态）+ styled-components 6（样式）+ Vite（构建） | package.json、docs/ARCHITECTURE.md |
| 编辑器 | ProseMirror 全家桶（view/state/model/transform/tables/history/markdown 等）+ y-prosemirror | package.json dependencies |
| 后端 | Koa 3 + @koa/router 15（API）、Sequelize 6 + sequelize-typescript（ORM） | package.json、docs/ARCHITECTURE.md |
| 数据库 | PostgreSQL（pg 8）主存储 + Redis（ioredis）缓存/队列/协作总线 | docker-compose.yml、package.json |
| 队列 | Bull 4 + @bull-board/koa（任务与事件处理） | package.json、server/queues/ |
| 实时协作 | Yjs 13（CRDT）+ @hocuspocus/server 1.1.3 + @hocuspocus/extension-redis（多实例同步）+ ws | package.json、server/services/collaboration.ts |
| 认证 | @outlinewiki/koa-passport + passport-oauth2/google-oauth2/slack-oauth2/azure-ad | package.json、plugins/* |
| 可观测 | Sentry（@sentry/node）、hot-shots（StatsD）、结构化 JSON 日志 | package.json、server/logging/ |
| 工程链 | TypeScript、oxlint（type-aware）+ oxfmt、husky + lint-staged、Vitest、patch-package、i18next | package.json scripts |
| 运行环境 | Node >=22.23.5 <23 或 >=24.22.3 <25 或 >=26.13.1 <27（锁得很窄） | package.json engines |

依赖规模：dependencies 214 个 + devDependencies 69 个；`patches/` 目录存放 patch-package 补丁。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `app/` | React 前端：scenes（页面）、components、models/stores（MobX）、actions、routes、editor 包装层 |
| `server/` | Node 后端：routes/api（41 个 API 模块）、routes/auth、models（67 个 Sequelize 模型）、commands（跨模型业务）、policies（cancan 授权）、presenters、middlewares、queues、collaboration（协作扩展）、mcp（MCP 服务器）、services（六个可拆分服务）、migrations（50 个迁移 + baseline） |
| `shared/` | 前后端共享：editor（ProseMirror 编辑器本体，最大子目录 61k 行）、i18n/locales（28 种语言）、types、utils、validations |
| `plugins/` | 24 个插件：认证（oidc/google/azure/discord/email/passkeys/slack）、集成（github/gitlab/gitea/figma/linear/notion/slab/zapier/webhooks）、分析（googleanalytics/matomo/umami）、search-postgres、storage、enterprise、diagrams |
| `docs/` | ARCHITECTURE.md（官方架构文档）、TRANSLATION.md 等 |
| `public/` | 静态资源与 Logo |
| `patches/` | patch-package 依赖补丁 |

组织方式评述：这是"单仓库多区域"结构而非 npm/yarn workspace monorepo——package.json 无 workspaces 字段，前后端通过 `@server/*`、`@shared/*`、`@app/*` 路径别名共享代码（tsconfig.json、docs/ARCHITECTURE.md）。共享代码集中在 `shared/`（尤其是编辑器），前端后端各自独立目录，业务插件按"客户端 + 服务端"双目录成对组织（如 `plugins/oidc/client/` + `plugins/oidc/server/`，各含 plugin.json 清单），是一套自研插件装配机制。前端 Vite 构建、后端由自定义 `build.js` 打包，整体是典型的"全栈 TypeScript 单体、按服务拆分部署"设计。

## 5. 核心架构与关键实现

### 5.1 六服务拆分部署模型

后端不是一个单体进程，而是六个可按命令行参数选择性启动的服务（`server/services/index.ts`，懒加载 import 以避免无关依赖进入内存）：

```ts
const services = {
  websockets: () => import("./websockets"),
  collaboration: () => import("./collaboration"),
  admin: () => import("./admin"),
  web: () => import("./web"),
  worker: () => import("./worker"),
  cron: () => import("./cron"),
} as const;
```

`package.json` 的 dev 脚本以 `node build/server/index.js --services=cron,collaboration,websockets,admin,web,worker` 一次拉起全部；生产可拆分扩容（如协作服务独立横向扩展，配 `@hocuspocus/extension-redis` 跨实例同步）。

### 5.2 实时协作：Yjs CRDT + Hocuspocus

协作服务（`server/services/collaboration.ts`）用 Hocuspocus Server（Yjs 生态的协作服务器）挂载 10 个自研 Extension（`server/collaboration/`）：Authentication（鉴权）、Persistence（持久化）、APIUpdate（API 写入同步进 CRDT）、Views（阅读统计）、ConnectionLimit、AwarenessIdentity、EditorVersion（编辑器版本协商，`shared/editor/version.ts` 中 `EDITOR_VERSION = "17.0.0"`）、Logger、Metrics、APIUpdate。关键配置：

```ts
const hocuspocus = Server.configure({
  debounce: 3000, timeout: 30000, maxDebounce: 10000,
  extensions: [
    ...(env.REDIS_COLLABORATION_URL ? [new Redis({...})] : []),
    new Throttle({ throttle: env.RATE_LIMITER_COLLABORATION_REQUESTS, ... }),
    new AuthenticationExtension(), new PersistenceExtension(), ...
  ],
});
```

持久化采用"CRDT 状态 + 派生内容"双写：`PersistenceExtension`（server/collaboration/PersistenceExtension.ts）加载时若文档已有 `state`（BLOB）则 `Y.applyUpdate` 还原；首次无 state 时加锁（`transaction.LOCK.NO_KEY_UPDATE` 防并发创建）从 `content`/`text` 初始化。保存时由 `server/commands/documentCollaborativeUpdater.ts` 执行 `Y.encodeStateAsUpdate(ydoc)` 写回 state，并把 ProseMirror JSON 派生为 `content` 字段。带 5 次重试与失败计数，防止单文档持久化失败拖垮协作。这是"CRDT 而非 OT"的直接证据（yjs、y-protocols、y-prosemirror 均在 package.json dependencies）。

### 5.3 文档模型：树 + 分数索引 + 结构缓存

`server/models/Document.ts`（约 1500 行核心模型）：`parentDocumentId` 自引用构成文档树，`collectionId` 归属集合，`state`（BLOB，Yjs 二进制）、`content`（JSONB，ProseMirror 文档）、`text`（纯文本）、`searchVector`（tsvector 检索）、协作编辑者列表等。移动/嵌套有环检测（`model.parentDocumentId === model.id` 及祖先链检查，Document.ts 第 626-642 行附近）。

`server/models/Collection.ts`：用 `fractional-index` 库（第 1 行 import）生成有序键实现 O(1) 拖拽排序；`documentStructure`（JSONB，NavigationNode 数组）是整棵文档树的冗余缓存，文档增删改时同步维护（Collection.ts 第 276 行及 153 行附近的 exclude 优化）；每个 Collection 有 10 字符 `urlId`（第 212 行校验）用于短链接。整体是"关系库 + JSONB 缓存树"的混合设计，避免每层递归查询。

### 5.4 ProseMirror 定制编辑器（shared/editor）

编辑器本体在 `shared/editor/`（前后端共用，服务端用它做导入导出/快照渲染），含 38 个 node（`shared/editor/nodes/`：Table、Math/MathBlock、Mermaid、Diagrams、ToggleBlock、Notice、Mention、Emoji、Video、CodeFence 等）、15 个 extension（`shared/editor/extensions/`：Diff（协作变更高亮，prosemirror-changeset）、MaxLength、History、DateTime、TrailingNode 等）、marks/nodes/rules/plugins/queries/selection 各子目录完整覆盖。`app/editor/` 是 React 包装层（menus、components）。编辑器通过 `EDITOR_VERSION`（shared/editor/version.ts）与协作端做版本协商（EditorVersionExtension），保证新旧客户端混连时的 schema 兼容。

### 5.5 API 与授权层

API 按 REST 资源切分为 41 个模块（`server/routes/api/`：documents、collections、comments、shares、groups、integrations、oauthClients、installation 等），每个模块配 `schema.ts`（参数校验）与 colocated 测试；响应经 `server/presenters/`（模型→JSON 的前后端接口层）；授权集中在 `server/policies/`（cancan 策略）；跨模型复杂操作走 `server/commands/`（documentMover、documentDuplicator、accountProvisioner 等 30+ 命令，测试覆盖率高）。事件驱动部分：API/协作产生的领域事件进 Redis/Bull 队列，`server/queues/processors/` 消费（通知、webhook、集成同步等）。

### 5.6 插件系统与 SSO

24 个插件（`plugins/*/plugin.json` 含 name/description 清单）统一装配：认证类（oidc、google、azure、discord、email 魔法链接、passkeys）各自实现 passport 策略并通过 `server/routes/auth/index.ts` 动态挂载 provider 路由；集成类（github/gitlab/figma/linear 等）负责链接 unfurl 与内容导入；`plugins/search-postgres/` 是可替换的搜索引擎实现（Postgres tsvector，为将来换 ElasticSearch 等留的抽象位）。注意：本仓库内未找到 SAML 实现（全仓 grep "saml" 仅命中无关文件），SAML/SCIM 属于托管版/企业版能力，代码不在开源仓库中——按规范标注"未确认，推断为企业版闭源部分"（`plugins/enterprise/` 仅含客户端翻译文件，`plugins/enterprise/client/translations.tsx`）。

另有一个亮点：内建 MCP 服务器（`server/mcp/`，入口 `server/routes/mcp`），暴露 tools 与 skills（`skills/list`、`skills/get`），README.md 示例展示用 Claude 直连 Outline 操作知识库，是较新趋势的落地。

## 6. 构建与运行

来自 Makefile、package.json scripts 与 docker-compose.yml（均为仓库真实文件）：

```bash
# 本地开发（Postgres + Redis 用 docker 起，前端 Vite + 后端 watch 并行）
make up          # = docker compose up -d redis postgres && yarn install-local-ssl && yarn install --immutable && yarn dev:watch

# 生产构建与启动
yarn build       # vite:build + i18n 提取 + build.js 打包 server
yarn start       # node ./build/server/index.js

# 数据库
yarn db:migrate  # sequelize db:migrate
yarn db:reset    # drop + create + migrate

# 测试（Vitest，多 project：app / shared-node / shared-jsdom / server*）
yarn test        # TZ=UTC vitest run
yarn test:server # 仅 server 项目

# 自托管生产栈
docker compose build --pull outline   # make build
```

依赖服务：PostgreSQL、Redis；文件存储经 `plugins/storage/`（S3/本地，见 `server/storage/files/`）。Node 版本要求见 engines（22/24/26 窄区间）。HTTPS 本地开发有 `yarn install-local-ssl`（mkcert，`server/scripts/install-local-ssl.js`）。

## 7. 测试与工程质量

- 测试规模：410 个 `*.test.ts(x)` 文件——server 253、shared 82、plugins 35、app 40（源码统计）。模型、命令、协作扩展（如 `server/collaboration/PersistenceExtension.test.ts`、`AuthenticationExtension.test.ts`）均有覆盖；API 路由含快照测试（`server/routes/api/documents/__snapshots__/`）。
- 框架与组织：Vitest 多 project 配置（vitest.config.mts：app 用 jsdom、shared 分 node/jsdom 两个 project、server 带 `server*` 通配）；测试与源码 colocated，另设 `server/test/`、`app/test/` 放工具与 fixtures。
- CI（.github/workflows/ci.yml）：按变更路径分发的矩阵——lint（oxlint type-aware）、actionlint、types（tsc）、audit、test、test-server 等独立 job；另有 codeql-analysis.yml、docker.yml/docker-build.yml/docker-nightly.yml（镜像与 nightly）、release.yml（发布流程）、dependabot。
- 代码质量观察：oxlint + oxfmt（Oxc 工具链）替代 ESLint/Prettier，构建快；Node engines 锁窄说明对运行时行为敏感；README 有明确的贡献前置要求（必须先开 issue 讨论）。

## 8. 社区与维护状态

- 活跃度：查询时最近 push 为 2026-11-13（当天），最新 release v1.12.0 于 2026-10-23 发布（gh api），迁移目录最新条目为 2026-11-12（`server/migrations/20261112120000-*.js`）——持续高频迭代。
- 贡献者：约 253 人（contributors API 尾页推算）；核心由 General Outline, Inc. 团队主导（LICENSE Licensor、package.json author），社区贡献以集成/翻译为主，README 特别声明"不接受 AI 生成的 PR"。
- 国际化：Crowdin 托管翻译（crowdin.yml、README 徽章），`shared/i18n/locales/` 下 28 种语言，CI 有 merge-translations.yml / close-translation-prs.yml 自动化。
- 历史脉络：仓库创建于 2016-05-22（gh api created_at），十年老项目；许可演变经标签核实——v0.43.0（2020-05-22）及以前为 BSD-3-Clause，v0.45.0（2020-07-18）起切换为 BSL 1.1（对两个标签的 LICENSE 原文用 gh api contents 核实），早于 2024 年业界 BSL 热潮；当前 LICENSE 的 Change Date 2030-10-23 恰为 v1.12.0 发布日 + 4 年，符合"每次发布四年后转 Apache 2.0"的标准 BSL 条款。

## 9. 亮点与风险

亮点：

1. 协作架构成熟：Yjs CRDT + Hocuspocus + Redis 多实例同步 + WebSocket 节流/连接数限制/编辑器版本协商，工程细节完整（server/services/collaboration.ts、server/collaboration/*）。
2. 全栈 TypeScript 且共享彻底：编辑器、类型、校验在 shared/ 前后端复用，38 万行单语言代码库保持一致的 lint/format/type 检查。
3. 服务可拆分：六个服务按参数启动，协作/Worker/API 可独立扩容（server/services/index.ts）。
4. 插件化 SSO 与集成：24 个插件、OIDC/Passkeys 等现代认证齐备，搜索引擎抽象成可替换插件（plugins/）。
5. 测试与 CI 认真：410 个测试文件、多 project Vitest、变更感知的 CI 矩阵与快照测试。

风险/局限：

1. 许可证非 OSI 开源：BSL 1.1 限制"Document Service"竞争用途（LICENSE Additional Use Grant），把它当 Confluence 竞品做 SaaS 化二次分发是违约的；2030-10-23 后本版本才转 Apache 2.0。
2. 企业功能不完全在仓库内：SAML/SCIM 等能力未见源码（grep 无实现，plugins/enterprise 仅剩客户端文案），自托管用户拿到的功能集与托管版有差距。
3. 自托管组件较重：需要 Postgres + Redis，生产还涉及 S3 存储、多服务编排，对中小团队运维有门槛（docker-compose.yml、Makefile）。
4. 运行时版本窗口窄：Node engines 只允许 22.23+ / 24.22+ / 26.13+ 三个窄区间，升级节奏需跟紧（package.json engines）。

## 10. 总结评价

Outline 是"商业开源"路线的经典样本：以十年积累的 38 万行全栈 TypeScript 代码库，做出了托管级质量的知识库产品，其 Yjs CRDT 协作实现、ProseMirror 深度定制编辑器与服务拆分架构都是同类自托管项目中的第一梯队水平，且文档结构模型（fractional-index 排序 + JSONB 树缓存）与插件化 SSO 设计都有很高的参考价值；代价是 BSL 1.1 许可（禁商用竞品、四年后才转 Apache 2.0）和企业功能闭源带来的"开源但非自由"争议。适合团队自托管替代 Confluence/Notion 使用，也极适合想学习实时协作编辑器、CRDT 落地与大型 React/Node 工程组织的开发者精读源码；若要在 BSL 约束内做二次开发，需先对照 LICENSE 的 Additional Use Grant 划清用途边界。
