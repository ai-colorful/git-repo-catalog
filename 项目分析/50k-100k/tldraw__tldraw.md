# tldraw/tldraw 项目分析

> 星数 51055（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：tldraw 是一个"无限画布白板 SDK"——用 React 把完整的多画布引擎（绘图、形状、协同、渲染、撤销）以 `<Tldraw />` 组件一行代码嵌入应用，同时允许深度定制形状/工具/绑定，是白板类产品的"底座引擎"。

| 项 | 值 |
| --- | --- |
| 仓库 | tldraw/tldraw（monorepo） |
| 星数 / forks / open issues | 51055 / 3601 / 568（gh api，2026-11-04） |
| 主语言 | TypeScript（全部） |
| 许可证 | 自定义 "tldraw license"（GitHub 标记 NOASSERTION，见 `LICENSE.md`） |
| 最新 release | v5.6.0（2026-10-28 发布，gh api） |
| 最近 push | 2026-11-03（gh api）——活跃中 |
| 维护方 | tldraw Inc.（伦敦团队，Series A，`COMPANY.md`） |

## 2. 项目是什么，解决什么问题

tldraw 解决的是"画布应用的基础设施重复造轮子"问题。任何想做白板、思维导图、流程图、无限画布 AI 应用的团队，都需要处理平移缩放、选择框、几何吸附、撤销重做、协同冲突解决、渲染剔除等大量细节。tldraw 把这些打包成一个可嵌入的 React SDK（`README.md`：Feature highlights 列出 multiplayer、绘图/图表、Editor 运行时 API、可扩展 shape/tool/binding 等）。

面向开发者的第一印象刻意做得极轻：`npm i tldraw` 后渲染 `<Tldraw />` 一个组件即得到完整白板（README Quick start）；进阶路径是"Starter kits"（Multiplayer/Agent/Workflow/Chat 等 7 种模板，`npx create-tldraw@latest`），每个 kit 都是 MIT 许可的独立模板（README Starter kits 一节）——即模板代码真开源，核心 SDK 受水印许可约束。

它的商业模式直接写在代码里：默认 SDK 带 "Made with tldraw" 水印，不允许生产环境使用；购买商业授权后通过 License Key 解锁水印与协同等高级特性。`LICENSE.md` 明确 Production Environment 须经授权；`packages/editor/src/lib/license/LicenseManager.ts` 用 RSA 公钥验签 license、按 bit flag 区分年付/永久/水印/协同功能，并有 30 天宽限期逻辑（`GRACE_PERIOD_DAYS = 30`）。这正是"开源式分发 + 水印 + 商业许可"的典型组合（sync-core 的 `README.md` 也写明"保留水印即可商用，去除需购买 business license"）。

仓库同时托管了 tldraw.com 官网（`apps/dotcom`，基于 Cloudflare Workers/Durable Objects + Zero）、文档站（`apps/docs`）、示例站（`apps/examples`）与 VS Code 插件（`apps/vscode`），是"SDK + SaaS"一体化的 monorepo。

值得注意的是 tldraw 近年的 AI 叙事：README 提供 AI 集成入口，仓库根目录有 `AGENTS.md`、`CLAUDE.md`、`skills/`（commit/release 等 agent 技能），`apps/mcp-app` 提供 MCP 应用，`@tldraw/driver`（`packages/driver/README.md`）提供命令式 API 供脚本/自动化/AI 驱动画布（`driver.click(100,200).pointerDown(...)`）。

## 3. 技术栈与依赖

- 语言/运行时：全 TypeScript（根 `package.json` 要求 node >= 22.12、TypeScript ^7.0.2、pnpm@12.4.2）。
- 构建/任务编排：turbo（`turbo.json`）+ lerna（`lerna.json`，管 packages/* 发版）+ esbuild/swc；api-extractor 维护公共 API 报告（`api-report.api.md` 散布各包）。
- 前端核心：React 19；富文本用 Tiptap 3 / ProseMirror（`packages/tldraw/package.json` dependencies）。
- 协同：自研 `@tldraw/sync-core`（WebSocket 协议 + diff/patch），官方部署栈为 Cloudflare Durable Objects + SQLite（`packages/sync-core/src/lib/DurableObjectSqliteSyncWrapper.ts`）；dotcom 侧使用 Rocicorp Zero（`pnpm-workspace.yaml` 依赖门禁可见）。
- 测试：vitest 5（根 `vitest.config.ts` 用 glob 把每个包的 vitest.config.ts 组成 projects）+ Playwright（e2e/perf/快照）。
- 代码质量：oxlint + oxfmt（根 `package.json` lint-staged），husky pre-commit。
- 供应链防护：`pnpm-workspace.yaml` 设 `minimumReleaseAge: 10080`（新发布依赖冻结 7 天才可用）、`allowBuilds` 白名单默认禁用第三方 postinstall 脚本——在同类仓库中属于少见的高安全水位。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `packages/editor` | 核心：无 UI 的画布引擎（Editor 类、工具、形状基类、渲染组件、16+ 管理器），248 个 ts 文件 |
| `packages/tldraw` | 产品层 SDK：默认形状（geo/draw/arrow/note/text/image 等 12 类）、默认工具、完整 UI 组件、外部内容处理，582 个 ts 文件 |
| `packages/state` | 自研信号库 @tldraw/state（Atom/Computed/EffectScheduler/事务），框架无关 |
| `packages/state-react` | state 的 React 绑定（useValue/useStateTracking 等） |
| `packages/store` | 信号驱动的记录型存储 @tldraw/store（Store/Schema/迁移/副作用/查询） |
| `packages/tlschema` | 画布文档模型：shape/asset/binding/camera/page 记录定义与迁移 |
| `packages/sync` / `sync-core` / `sync-collaboration` | 多人实时协同：客户端 hook、房间协议引擎、协同特性层 |
| `packages/utils` / `validate` / `assets` | 工具函数、运行时校验器、资源生成 |
| `packages/driver` | 命令式驱动 API（自动化/测试/AI 用） |
| `apps/dotcom` | tldraw.com 官网产品（客户端 + sync worker + Zero cache） |
| `apps/examples` | 示例站（SDK 示例 + Playwright e2e/perf 测试） |
| `apps/docs` | 文档站 tldraw.dev 内容 |
| `apps/vscode`、`apps/mcp-app`、`apps/analytics`、`apps/bemo-worker` | VS Code 插件、MCP 应用、统计、bemo 服务 |
| `internal/` | 发版脚本（huppy）、配置、Grafana、健康检查等 |
| `assets/`、`templates/`、`skills/` | 图标/字体/翻译/水印资源、Starter 模板、内部 agent 技能 |

组织方式评述：严格的分层 monorepo——`state → store → tlschema → editor → tldraw → sync` 自下而上单向依赖（根 `package.json` 的 `check-circular-deps` 脚本显式守护循环依赖）。核心引擎与产品皮肤分离：`editor` 不含默认形状，`tldraw` 才组装默认体验，这是它作为 SDK 的关键设计。

## 5. 核心架构与关键实现

### 5.1 信号系统 @tldraw/state（packages/state/src/lib）

tldraw 没有用第三方响应式库，自研了 signals：`Atom.ts` 是可写的源信号，支持 `historyLength` 滚动 diff 缓冲与自定义 `isEqual`；`Computed.ts` 实现惰性求值 + 依赖追踪，通过全局 epoch（`transactions.ts`）判断过期，`transact` 提供事务与回滚。编辑器的几乎所有状态（选区、相机、历史栈）都是信号，React 侧经 `@tldraw/state-react` 按需订阅，避免整树重渲。

```ts
// packages/state/src/lib/Computed.ts（节选）
const count = atom('count', 0)
const double = computed('double', (prevValue) => {
	if (isUninitialized(prevValue)) { /* 首次计算 */ }
	return count.get() * 2
})
```

### 5.2 存储层 @tldraw/store（packages/store/src/lib/Store.ts）

Store 是"带信号索引的文档数据库"：记录（shape/asset/binding…）存于 `AtomMap`，`StoreQueries` 维护按记录类型的派生查询缓存（`createComputedCache`），`StoreSchema` 管版本迁移（`store-migrations.ts` 见 `packages/tlschema/src`），`StoreSideEffects` 提供创建/更新/删除副作用钩子供 SDK 用户拦截。`Store.ts` 头部定义了 `RecordFromId` 类型推导——记录 ID 即类型。

### 5.3 Editor 上帝对象与渲染管线（packages/editor/src/lib）

`editor/Editor.ts` 单文件 11792 行、约 226 个方法，是所有画布操作的运行时 API：创建/变换形状、管理选区/相机/快照，内部聚合 16 个管理器（`editor/managers/`：HistoryManager、SnapManager、EdgeScrollManager、SpatialIndexManager、InputsManager、FontManager 等）。`HistoryManager.ts` 用一个 atom 保存 undo/redo 双栈，通过 `store.addHistoryInterceptor` 把每次 store 变更（仅 `source === 'user'`）折叠进 pendingDiff。

渲染是 DOM-based 而非 WebGL：`components/Shape.tsx` 注释明确两阶段策略——外层容器用 CSS transform 定位/透明度，内层 shape JSX 经 memo 只在 props 变化时重渲；视口剔除由 `hooks/useShapeCulling.ts` 的 `ShapeCullingProvider` 集中管理，单个 reactor 统一切换所有 shape 容器的 display，而非每个 shape 各自订阅。画布叠层分两类容器：`HTMLContainer` 承载形状内容，`SVGContainer` 承载选择框/手柄等覆盖物（`packages/editor/src/lib/components/`）。形状扩展契约是抽象类 `ShapeUtil`（`editor/shapes/ShapeUtil.ts`）：必须实现 `getDefaultProps/getGeometry/component/getIndicatorPath`，并支持 `static props` 校验与 `static migrations` 属性级迁移；默认产品层的 12 种形状（geo/draw/arrow/line/note/text/frame/image/video/highlight/bookmark/embed）都在 `packages/tldraw/src/lib/shapes/` 下各自成目录，是最现成的 ShapeUtil 实现范例。

### 5.4 多人协同 @tldraw/sync-core（packages/sync-core/src/lib）

`TLSyncRoom.ts` 是房间核心：服务端持有文档状态，客户端通过 socket 协议（`protocol.ts`、`diff.ts`、`chunk.ts`）收发记录级 diff，支持重连续传（`RoomSession.ts`）。存储抽象 `TLSyncStorage` 有 InMemory/SQLite 两种实现，`DurableObjectSqliteSyncWrapper.ts` 适配 Cloudflare Durable Objects——README 的 Multiplayer starter kit 即此栈。React 侧 `packages/sync/src/useSync.ts` 把连接状态、presence（光标/选区）通过 `createPresenceStateDerivation` 接进信号系统。

### 5.5 许可证与水印（商业模式的技术实现）

`packages/editor/src/lib/license/LicenseManager.ts`：license 是签名 payload，`FLAGS` 位域编码年付/永久/内部/带水印/评估/原生与协同子特性（FEAT_COLLABORATION/FEAT_COMMENTING/TRACKED），`HOSTS` 限制部署域，内置 `CANONICAL_TLDRAW_HOSTS` 白名单免去自有域验签；`license/Watermark.tsx` + 自动生成的 `watermarks.ts`（内嵌桌面/移动两套 "MADE WITH TLDRAW" SVG）负责水印渲染。`assets/watermarks/` 存有源 SVG。整个许可体系位于 `editor` 包内，随 SDK 一起分发。

## 6. 构建与运行

来自根 `package.json` scripts 与 README：

```bash
pnpm install                 # postinstall 会跑 husky + refresh-assets
pnpm dev                     # turbo 起 examples + bemo + image-resize worker
pnpm build                   # turbo run build（全量）
pnpm build-package           # 仅构建 packages/*
pnpm test                    # vitest（全仓 projects 模式）
pnpm test-coverage           # vitest --coverage
pnpm e2e                     # examples 的 Playwright e2e
pnpm typecheck               # turbo run build-types
pnpm api-check               # 构建 + 校验公共 API 面（api-extractor）
```

终端用户只需 `npm i tldraw` 后渲染 `<Tldraw />`（README Quick start）。要求 Node >= 22.12。运行官网全栈开发环境另有 `pnpm dev-app`（process-compose 编排 dotcom + Zero + sync worker）与 `pnpm dev-app:clean` 清理本地 SQLite，见根 `package.json` scripts；发布流程为手动触发 `publish.yml` 从 production 分支出包（`RELEASES.md`）。

## 7. 测试与工程质量

- 测试规模：全仓 584 个 `*.test.ts(x)`，packages 与 apps 各自带 `vitest.config.ts`（根 `vitest.config.ts` glob 聚合）。核心包覆盖密：`store` 20+ 测试文件、`Editor.test.ts`、`LicenseManager.test.ts`、sync-core 几乎每文件配对测试。
- e2e：`apps/examples/e2e/` 含 tests/perf/mermaid-readability 与 baselines（视觉基线），配套 5 个 Playwright 工作流（`playwright-examples/dotcom/perf/mermaid/update-snapshots.yml`）。
- CI：`.github/workflows/` 下 35 个工作流，覆盖 checks、发版（`publish.yml` 手动触发、从 production 分支发布）、i18n 字符串上传下载、license-report、staging 冒烟、issue 自动分诊（`issue-triage.yml`）。
- 工程质量观察：公共 API 用 api-extractor 冻结并做 `api-check`；`check-circular-deps` 防循环依赖；oxlint/oxfmt；依赖版本策略写在 RELEASES.md——minor 每月发、可能含 breaking change（靠提前告警 + 迁移工具缓解），对下游是有代价的激进节奏。

## 8. 社区与维护状态

- 活跃度极高：最近 push 2026-11-03，最新 release v5.6.0 于 2026-10-28（gh api），月度发版节奏稳定（RELEASES.md 自述"monthly"）。
- 贡献者结构：特殊——`CONTRIBUTING.md` 明确"**不再接受外部 PR**"（指向 issue #7695），仅接受 issue 报告；代码全部由 tldraw 公司团队（伦敦）维护，fork 数 3601 说明外部派生需求仍在。星数增长更多来自产品口碑与 README 传播而非社区共建。
- 客户背书：README 列举 Google、Shopify、BlackRock、Autodesk、ClickUp、Replit 等使用方；提供 showcase 案例。
- 商业实体：`COMPANY.md` 自述为 Series A 公司；周边运维（Grafana、analytics、health-worker、post-release-crew 等工作流）显示成熟的公司级工程体系。贡献者人数无法从 tarball 确认（未确认）。
- 国际化与内容运营：`assets/translations/` 收录多语言翻译，CI 中有 i18n 字符串上传/下载/校验工作流（`i18n-upload-strings.yml` 等），社区可翻译但不可改代码；`apps/docs/content/releases/` 的发布说明与 GitHub draft release 联动（`RELEASES.md` 描述的发布门禁）。

## 9. 亮点与风险

亮点：
1. 自研三层内核（state 信号 → store 记录库 → editor 引擎）边界清晰，信号粒度响应式让 DOM 渲染方案也够快；单向依赖 + 循环依赖检查（`packages/state`→`editor` 链条，根 scripts）。
2. 可扩展性设计完整：ShapeUtil/工具/绑定/副作用钩子/组件覆盖一整套 SDK 面，且 `@tldraw/driver` 提供命令式驱动，正对上 AI 操作画布的需求（`packages/driver/README.md`）。
3. 协同自研到协议层（sync-core 的 diff/chunk/续传 + Durable Objects 适配），可自托管。
4. 工程防御水位高：`pnpm-workspace.yaml` 的 minimumReleaseAge + allowBuilds 白名单、api-extractor 冻结 API、584 个测试文件与 5 套 Playwright 流水线。
5. 商业闭环内嵌于代码：水印/LicenseManager/域名白名单全部可在源码核对（`packages/editor/src/lib/license/`）。

风险/局限：
1. 许可证非 OSI 开源：生产环境禁止、去水印需付费（`LICENSE.md`），对商用集成是硬门槛；GitHub license 显示 NOASSERTION。
2. 拒绝外部 PR（`CONTRIBUTING.md`），生态演化完全绑定单一公司，社区可参与度低；issue 积压 568。
3. 激进发版：minor 可能含 breaking change（`RELEASES.md`），v5.x 阶段对升级方维护成本不低。
4. Editor.ts 11792 行的"上帝对象"与 monorepo 规模（约 2400+ TS 文件）抬高了阅读与二开成本；dotcom 依赖栈（Zero/Durable Objects/Clerk 等）较重。

## 10. 总结评价

tldraw 是"画布引擎产品化"的标杆：它把一个白板应用拆成 state/store/editor/tldraw 四层 SDK，用自研信号系统串起存储、渲染与协同，再以水印 + 商业许可完成变现，代码质量、测试密度与供应链防御都是同类项目顶级水平。它适合三类人：想快速搭白板/画布应用的产品团队（接受水印或付费）、想深入理解图形编辑器架构与信号式响应式的工程师（重点读 `packages/state` 与 `packages/editor/src/lib/editor/Editor.ts`）、以及研究"开源分发 + 商业 SDK"模式的创业者（license 实现本身就是教材）。需要社区共建或宽松许可的团队则应另寻 Excalidraw 等替代。
