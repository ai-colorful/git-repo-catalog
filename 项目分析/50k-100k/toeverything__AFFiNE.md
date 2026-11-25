# toeverything/AFFiNE 项目分析

> 星数 73774（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：开源的 Notion + Miro 一体化替代品——文档、白板、数据库三合一的本地优先（local-first）知识工作台。

| 项 | 值 |
| --- | --- |
| 名称 | toeverything/AFFiNE |
| 星数 | 73774（快照 2026-10-24） |
| 主语言 | TypeScript（另有大量 Rust 原生层） |
| 许可证 | 客户端 MIT；`packages/backend/server` 与 `packages/common/native` 为 AFFiNE 企业版（EE）源可用许可（根 `LICENSE`、`packages/backend/server/LICENSE`） |
| 当前版本 | 0.27.5（根 `package.json`），另按日期发 canary（如 `v2026.10.27-canary.909`） |
| 默认分支 | canary |
| 最近活跃 | 最近 push 2026-10-27（gh api），每日 canary release |
| 贡献者 | 约 272 人（gh api contributors 分页统计） |

## 2. 项目是什么，解决什么问题

AFFiNE 自我定位是 "Write, Draw and Plan All at Once"——把结构化文档（类 Notion）、无限画布（类 Miro）、多视图数据库合并到同一个块（block）体系中（`README.md`）。用户既能在页面模式下写文档，也能一键切换到 edgeless 模式，把任意块（富文本、便签、嵌入网页、多视图数据库、双链页面、图形甚至幻灯片）拖上画布自由排版，这是它区别于多数"伪画布"编辑器的核心卖点。

数据上坚持 local-first：数据永远先落在本地（浏览器 IndexedDB 或桌面端 SQLite），再按需同步到云端或自建服务端；同时支持基于 CRDT 的实时协同（`README.md` Features 一节，以及 `packages/common/nbstore/README.md` 的存储抽象）。商业形态上提供官方云（含 AFFiNE AI 付费能力），但整条自托管链路——客户端、同步服务端、数据库 schema——全部开源可自查。

技术形态是一个超大型 monorepo：编辑器内核 BlockSuite 直接内嵌在 `blocksuite/` 目录（相当于把上游编辑器仓库整体并入），应用层在 `packages/frontend/core`，多端壳（Web/Electron/iOS/Android）在 `packages/frontend/apps`，云服务端在 `packages/backend/server`，性能关键路径（SQLite 存储、blob/文档同步、索引）用 Rust 通过 napi 实现并编译到各端（`Cargo.toml` workspace 含 7 个 crate）。

## 3. 技术栈与依赖

- 语言与规模：TypeScript/TSX 约 85.5 万行（6991 个文件）、Rust 约 10.2 万行（353 个文件），静态统计自 tarball。
- 前端：React + radix-ui（`packages/frontend/core/src/blocksuite/block-suite-editor/blocksuite-editor.tsx` 引入 `useService`/`useLiveData` 与 radix Slot）；编辑器视图层为 Lit Web Components，React 侧通过 `lit-adaper.tsx` 桥接。
- 编辑器内核：BlockSuite（本仓库内 `blocksuite/`），数据层构建在 Yjs CRDT 之上（`blocksuite/framework/store`）。
- 服务端：NestJS 11 + Apollo GraphQL + Prisma + socket.io（`packages/backend/server/package.json`）。
- 原生层：Rust workspace（`Cargo.toml`）：`packages/frontend/native/nbstore`（SQLite 文档/blob/索引存储与同步）、`affine_doc_loader`、`affine_importer`、`affine_preview` 等，经 napi 暴露给 JS。
- 多端：Electron（forge.config.mjs，`packages/frontend/apps/electron/`）、iOS/Android 壳（`packages/frontend/apps/ios|android`）。
- 工程链：Yarn workspaces monorepo、Node >= 22.12、vitest 单测、Playwright e2e、oxlint+oxfmt lint、tsc -b typecheck（根 `package.json` scripts）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `blocksuite/` | 内嵌的 BlockSuite 编辑器内核仓库：`framework/`（store/sync/std/global 四个基础包）+ `affine/`（约 20 种块、gfx 图形引擎、组件、数据视图等）+ playground/integration-test |
| `packages/frontend/core` | 主应用核心（2188 个 TS 文件）：`src/modules` 70 个业务模块、`src/blocksuite` 编辑器集成层、`src/components`、desktop/mobile 两套入口 |
| `packages/frontend/apps` | 各端壳：web、electron、electron-renderer、ios、android、mobile |
| `packages/frontend/component` | 共享 UI 组件库（348 个文件） |
| `packages/frontend/native` | Rust↔JS 桥：nbstore SQLite 绑定、磁盘同步、预览 |
| `packages/common` | 跨端基础库：infra（DI/响应式）、nbstore（存储抽象）、realtime（实时通道协议）、graphql、env、auth、theme 等 |
| `packages/backend/server` | NestJS 云服务端（701 个文件，EE 许可）：GraphQL API、协同、账户/会员、copilot |
| `tests/` | Playwright e2e 套件：affine-local、affine-cloud、affine-desktop(-cloud)、affine-mobile 等 8 个子项目 |
| `tools/`、`docs/`、`scripts/` | CLI、commitlint、构建文档（BUILDING.md、CONTRIBUTING.md） |

组织方式评述：三层 monorepo——编辑器内核（BlockSuite，可独立演化并对外发布）／应用框架与领域模块（frontend/core + common）／端与服务（apps + backend/server）。`package.json` workspaces 用 `blocksuite/**/*`、`packages/*/*`、`tests/*` 等通配覆盖约百余个子包；领域代码高度模块化（见第 5 章），块和扩展以细粒度子包发布（如 `@blocksuite/affine-block-paragraph`）。

## 5. 核心架构与关键实现

### 5.1 BlockSuite 数据模型：Yjs 之上的块文档（`blocksuite/framework/store`）

`framework/store/src` 分为 `model/`（块模型：block、draft、flat-sync-controller 等）、`yjs/`（CRDT 适配与 awareness）、`schema/`、`transformer/`（slice/json 快照，用于导入导出与复制粘贴）、`adapter/`。块树以 Yjs 文档为底座，BlockSuite 在其上提供 schema 校验与响应式查询；上层应用不直接操作 Yjs，而是通过 `Store`（`@blocksuite/affine/store`）API 读写块（见 `blocksuite-editor.tsx` 中 `page: Store` 的类型声明）。

### 5.2 应用与编辑器的集成边界（`packages/frontend/core/src/blocksuite`）

React 宿主通过两层管理器装配编辑器，这是理解 AFFiNE 插件化的关键：

- `manager/store.ts`：数据侧扩展。用 `StoreExtensionManager`（来自 `@blocksuite/affine/ext-loader`）组装内部扩展 + `AIStoreExtension` + `FeatureFlagStoreExtension`，单例缓存。
- `manager/view.ts`：视图侧扩展。`ViewExtensionManager` 按能力逐项配置（foundation、theme、database、comment、pdf、mobile、electron、turbo-renderer 等 17 个配置项），最终产出 `BlockStdScope`。
- `block-suite-editor/lit-adaper.tsx`：把 Lit 实现的 `PageEditor`/`EdgelessEditor` 包装成 React 组件，注入各业务 Service（FeatureFlagService、EditorSettingService、JournalService 等），并提供 `AffineEditorContainer` 的代理兼容层。

这种"应用侧扩展注入编辑器内核"的分层让 AI、评论、PDF 预览等功能都不侵入编辑器本体（`view-extensions/` 下 17 个目录、`store-extensions/` 下 ai/feature-flag）。

### 5.3 本地优先存储与同步层（`packages/common/nbstore` + `workspace-engine`）

`nbstore` 定义了统一的 `SpaceStorage` 抽象，可插拔后端：`impls/` 下有 idb（浏览器 IndexedDB）、sqlite（桌面，经 Rust napi）、cloud、broadcast-channel（多标签页广播），并内置 v1 旧版兼容存储。同步按关注点拆为 `sync/doc`、`sync/blob`、`sync/awareness`、`sync/indexer` 四条线。应用侧 `packages/frontend/core/src/modules/workspace-engine/impls/cloud.ts` 展示了典型组装方式：按端选择 IndexedDB 或 SQLite 存储 + 云端 `CloudBlobStorage`/`StaticCloudDocStorage`，配合 LiveData/effect 驱动同步生命周期。实时协作通道协议在 `packages/common/realtime/src/index.ts` 中以强类型请求/主题映射定义（workspace.access、doc.share-state、copilot.delegated 等十余类消息），底层走服务端 socket.io。

### 5.4 依赖注入驱动的模块化应用（`packages/common/infra` + `modules/`）

自研 DI 框架 `@toeverything/infra`（`packages/common/infra/src/framework/core`：Framework/FrameworkProvider/Scope/identifier），配套 LiveData 响应式原语与 React hooks（`useService`/`useLiveData`）。`packages/frontend/core/src/modules/index.ts` 逐个调用 70 个模块的 `configureXxxModule` 完成装配；每个模块统一采用 entities/services/stores/providers/scopes 五件套目录（如 `modules/doc/`、`modules/workspace/`），形成清晰的可替换边界——这也是云端/本地双形态（workspace-engine 有 local.ts 与 cloud.ts 两套实现）能共存的原因。

## 6. 构建与运行

来自根 `package.json` 与 `docs/BUILDING.md`：

```bash
# 前置：Node 22（.nvmrc），Rust 工具链（rust-toolchain.toml）
yarn install                 # postinstall 自动执行 yarn affine init
yarn dev                     # = yarn affine dev，启动开发服务
yarn build                   # = yarn affine build
yarn test                    # vitest 单测
yarn test:coverage           # 覆盖率（codecov.yml）
yarn typecheck               # tsc -b tsconfig.json
yarn lint                    # oxlint --deny-warnings + oxfmt --check
yarn e2e                     # tests/affine-local 等 Playwright 套件（各子包内）
```

桌面端与云端另有专门文档：`docs/building-desktop-client-app.md`、`docs/developing-server.md`。服务端初始化：`yarn prisma migrate dev && yarn data-migration run`（`packages/backend/server/package.json`）。

## 7. 测试与工程质量

- 单元测试：vitest；blocksuite 下 101 个 `*.spec.ts`，packages 下 251 个，另有 blocksuite/integration-test 集成测试目录。
- e2e：Playwright（`tests/affine-local/package.json` 固定 `@playwright/test@1.58.2`），共 369 个 e2e 相关 TS 文件，按形态分 8 个子项目（本地、云、桌面、桌面+云、移动）。
- CI（`.github/workflows/`）：build-test、build-images、release-desktop(-platform)、release-mobile、pr-title-lint（强制 Conventional Commits）、auto-labeler，另有 copilot-test 自动化实验。
- 质量观察：lint 采用 oxlint `--deny-warnings` + oxfmt，lint-staged 覆盖 ts/toml/rs；要求 CLA 签署、PR 标题规范（`docs/CONTRIBUTING.md`）；工程约束严格，但 canary 高频发布意味着主干变化快。

## 8. 社区与维护状态

- 活跃度：最近 push 2026-10-27，最近 5 个 release 全在最近一周内（每日 canary：`v2026.10.27-canary.909` 等，gh api releases），维护极度活跃。
- 贡献者：约 272 人（contributors API 分页合计）；主仓库背后为 toeverything 商业团队，核心提交由团队驱动，社区以 issue/翻译/外围功能贡献为主（依据 CONTRIBUTING 流程与 all-contributors 徽章，贡献者内部结构未逐人确认）。
- issue 管理：开放 issue 802 个，配备 auto-labeler 与 issue-triaging 文档（`docs/issue-triaging.md`）。
- 渠道：Discord、官方博客 release note（CHANGELOG.md 指向 affine.pro/blog）、docs.affine.pro 文档站。

## 9. 亮点与风险

亮点：

1. 编辑器内核与应用彻底分层：BlockSuite 独立成体系（store/sync/std/blocks），通过 ext-loader 的 Store/View 双扩展管理器注入应用能力，插件边界清晰（`blocksuite/framework`、`packages/frontend/core/src/blocksuite/manager/`）。
2. local-first 架构完整落地：nbstore 多后端（IDB/SQLite/云）+ doc/blob/awareness/indexer 四路同步 + Rust 原生性能层，是同类开源项目中工程化程度最高的（`packages/common/nbstore`、`packages/frontend/native`）。
3. 自研 DI + LiveData + 70 个领域模块的统一模块范式，双形态（本地/云）实现可替换（`packages/common/infra`、`modules/index.ts`）。
4. 测试矩阵覆盖 web/桌面/移动 × 本地/云，Playwright 多子项目 + vitest + 集成测试三层。
5. 发布自动化成熟：每日 canary、多平台 release workflow、强 Conventional Commits。

风险/局限：

1. 许可证分裂：客户端 MIT，但服务端与部分 native 目录为 EE 订阅许可，生产环境自托管服务端需商业授权（根 `LICENSE`、`packages/backend/server/LICENSE`），二次分发展 derby 需仔细审查。
2. 体量巨大：85 万行 TS + 10 万行 Rust，完整构建需 Node+Rust 双工具链，源码阅读与贡献门槛高。
3. canary 迭代极快，API（尤其 BlockSuite 面向扩展的 API）稳定性对第三方插件作者不友好。
4. 编辑器内核 vendored 进主仓，跨 `blocksuite/` 与 `packages/` 的耦合面大，升级 BlockSuite 版本依赖同一团队节奏。

## 10. 总结评价

AFFiNE 是目前开源知识工具里少数同时啃下"块编辑器内核、CRDT 本地优先存储、多端原生壳、云服务端"四座大山的项目，工程复杂度接近小型办公套件操作系统；其扩展管理器 + DI 模块化 + 多后端存储的设计对同类产品极具参考价值。适合：想研究现代协同编辑器与 local-first 同步架构的工程师（重点读 `blocksuite/framework` 与 `packages/common/nbstore`）；想自托管 Notion/Miro 替代品的个人与小团队（客户端免费、服务端注意 EE 许可）；以及寻求一体化文档+白板体验的最终用户。不适合只想轻量定制一个编辑器的初学者——建议从 BlockSuite 独立文档与 playground 入手而非直接改主仓。
