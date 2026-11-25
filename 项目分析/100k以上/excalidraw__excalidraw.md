# excalidraw/excalidraw 项目分析

> 星数 134429（快照 2026-10-24）｜主语言 TypeScript｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：开源的手绘风格虚拟白板，可协作、端到端加密，同时以 npm 库 `@excalidraw/excalidraw` 的形式供第三方嵌入。

| 项 | 值 |
| --- | --- |
| 名称 | excalidraw/excalidraw |
| 星数 | 134,429（快照） |
| 主语言 | TypeScript（React + Vite） |
| 许可证 | MIT（LICENSE；gh api 确认） |
| 创建时间 | 2020-01-02（gh api） |
| 最近 push | 2026-10-29（gh api，分析前一天仍在提交） |
| 默认分支 | master |
| Fork 数 | 15,911；open issues 3,436（gh api） |
| 形态 | yarn workspaces monorepo（`package.json`） |

## 2. 项目是什么，解决什么问题

Excalidraw 是一个"无限画布 + 手绘风"的图表/线框图白板。用户在浏览器中画矩形、圆、菱形、箭头、自由绘制、文本等元素，所有图形以 roughjs 生成带随机抖动的手绘笔触，降低制图的"正式感"门槛（README.md：Infinite, canvas-based whiteboard / Hand-drawn like style）。保存格式是开放的 `.excalidraw` JSON 文件，可导出 PNG/SVG/剪贴板。

仓库同时是两样东西（CLAUDE.md 明确了这一分层）：其一是 `packages/excalidraw` 发布的 React 组件库，第三方可把整个编辑器嵌入自己的应用；其二是 `excalidraw-app`，即 excalidraw.com 官方站，展示完整能力——PWA 离线、实时协作、端到端加密、本地优先自动保存、只读分享链接（README.md "Excalidraw.com" 一节）。

协作与隐私是它的核心卖点：协作者通过 socket.io 房间同步元素增量（`excalidraw-app/collab/Portal.tsx`），媒体文件存 Firebase，但场景数据在上传前用 AES-GCM 加密，密钥放在 URL 片段里，服务器只见密文（`packages/excalidraw/data/encryption.ts`、`excalidraw-app/data/firebase.ts`）。

元素模型天然为多人编辑设计：每个元素带 `version`/`versionNonce`/fractional index 排序串（`packages/element/src/types.ts`），冲突解决是确定性的"高版本胜、同版本低 nonce 胜"（`packages/excalidraw/data/reconcile.ts`），撤销重做也是基于增量 delta 而非全量快照（`packages/element/src/store.ts` 的 `StoreDelta`）。

近年还叠加了 AI 能力：mermaid 图转 Excalidraw（`packages/excalidraw/mermaid.ts`、TTDDialog 文本转图对话框）、Diagram-to-Code 插件（`packages/excalidraw/components/DiagramToCodePlugin/`），以及 iframe/embeddable 元素可内嵌网页。

## 3. 技术栈与依赖

- 语言/框架：TypeScript 5.9.3 + React 19（`@types/react` 19.0.10）+ Vite 5（`package.json` devDependencies）。
- 状态管理：jotai 2.11（`packages/excalidraw/package.json`），编辑器内部状态与 app 状态用 jotai atom 组织（`packages/excalidraw/editor-jotai.ts`、`excalidraw-app/app-jotai.ts`）。
- 绘图内核：roughjs（`packages/element/src/renderElement.ts` 顶部 `import rough from "roughjs/bin/rough"`），手绘风格由此而来。
- 数学库：自维护 `@excalidraw/math`（点/线/曲线/多边形/PCA，`packages/math/src/`），为碰撞检测与绑定服务。
- 协作栈：socket.io-client 4.7.2 + firebase 11.3.1（`excalidraw-app/package.json`）；加密用浏览器原生 WebCrypto AES-GCM（`packages/excalidraw/data/encryption.ts`）。
- 测试：Vitest 3（jsdom 环境，`vitest.config.mts`、`setupTests.ts`），E2E 用 Playwright 1.58.2。
- 工程化：yarn 1 workspaces、ESLint + Prettier（`@excalidraw/eslint-config`）、husky + lint-staged、vite-plugin-pwa（PWA）、Vercel（`vercel.json`）与 Docker 双部署。
- 字体：自带 Excalifont、Virgil（经典手绘体）、Xiaolai（中文手绘体）等 11 套字体文件与子集化加载（`packages/excalidraw/fonts/Fonts.ts`）。
- i18n：Crowdin 托管，59 个语言包（`crowdin.yml`、`packages/excalidraw/locales/`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `packages/excalidraw/` | 核心编辑器，npm 包 `@excalidraw/excalidraw`；组件、渲染器、动作系统、字体、数据序列化都在这 |
| `packages/element/` | 元素模型：创建/变异/几何/碰撞/绑定/箭头/框架，纯 TS 不含 UI |
| `packages/math/` | 几何数学原语（点、线、贝塞尔、PCA） |
| `packages/common/` | 跨包常量、工具、类型 |
| `packages/fractional-indexing/` | 协作排序用的分数索引字符串生成 |
| `packages/laser-pointer/` | 激光笔轨迹轮廓生成 |
| `packages/utils/` | 导出工具（供脚本嵌入场景用） |
| `excalidraw-app/` | excalidraw.com 官方应用：协作（collab/）、Firebase 持久化（data/）、Sentry、分享 |
| `examples/` | NextJS / 浏览器 script 标签集成示例 |
| `dev-docs/` | Docusaurus 开发者文档站（代码库结构、JSON schema、API 文档） |
| `scripts/` | release、 locales 覆盖率、bundle 统计等构建脚本 |
| `.github/workflows/` | lint、test、size-limit、语义化 PR 标题、Docker 构建/发布、自动 release |

组织方式评述：清晰的"库 + 应用"双层 monorepo。`packages/element` 与 `packages/math` 保持零 UI 依赖，使核心几何逻辑可独立测试与复用；`excalidraw-app` 只做胶水（协作、存储、部署配置），保证 npm 库不沾应用专属逻辑。代码量约 14.4 万行 TS/TSX（不含测试与声明文件，find+wc 统计）。

## 5. 核心架构与关键实现

### 5.1 元素模型：不可变对象 + 协作元数据（packages/element/src/types.ts）

所有图形是带品牌类型的 `Readonly` TS 对象，基座 `_ExcalidrawElementBase` 关键字段（types.ts L45-108）：

```ts
type _ExcalidrawElementBase = Readonly<{
  id: string; x: number; y: number;
  strokeColor: string; backgroundColor: string; fillStyle: FillStyle;
  width: number; height: number; angle: Radians;
  /** Random integer used to seed shape generation so that the roughjs shape
      doesn't differ across renders. */
  seed: number;
  /** Integer that is sequentially incremented on each change. Used to reconcile
      elements during collaboration or when saving to server. */
  version: number;
  versionNonce: number;   // 同版本冲突时确定性 tiebreak
  index: FractionalIndex | null; // 多人排序，rocicorp/fractional-indexing
  isDeleted: boolean;     // 软删除，协作日志友好
  groupIds: readonly GroupId[]; frameId: string | null;
  boundElements: readonly BoundElement[] | null;
  ...
}>
```

设计要点：`seed` 固定 roughjs 随机种子保证同一元素每次渲染笔迹一致；`isDeleted` 软删除而非物理删除；`index` 分数索引让协作方无需重排整个数组。变体类型（rectangle/ellipse/arrow/elbow-arrow/freedraw/text/image/frame/iframe/embeddable/stickynote 等）通过 `_ExcalidrawElementBase & {...}` 交叉类型组合（types.ts L93-210）。元素更新一律走 `newElementWith`（`packages/element/src/mutateElement.ts` L149）：纯复制并自动 `version+1`、重掷 `versionNonce`、刷新 `updated`，杜绝原地修改。

### 5.2 渲染：三层画布 + 手绘笔触（packages/excalidraw/renderer/ + components/canvases/）

编辑器由三张叠放 canvas 组成（`packages/excalidraw/components/canvases/`：`StaticCanvas.tsx`、`InteractiveCanvas.tsx`、`NewElementCanvas.tsx`）：静态层画已确认元素（节流重绘 `renderStaticSceneThrottled`，staticScene.ts L645），交互层画选择框/变换手柄/绑定预览（interactiveScene.ts，约 2200 行），新元素层画正在拖拽的草稿。实际形状绘制在 `packages/element/src/renderElement.ts`（1416 行），每元素经 roughjs 的 `rough.canvas(canvas)` 生成手绘路径（L325）。容错设计很有意思——渲染失败按"单元素"粒度处理（staticScene.ts L53 注释）：

```ts
/**
 * A render failure is per-element on purpose, on export too: one bad element
 * must not blank the canvas or fail the export, the rest of the scene
 * survives it.
 */
const logElementRenderError = (error, element) => { ... }
```

### 5.3 协作层：socket.io 房间 + 版本调和 + E2E 加密（excalidraw-app/collab/ + packages/excalidraw/data/）

`Portal.tsx`（257 行）管理 socket.io 生命周期：`init-room` → `join-room` → 监听 `room-user-change`（Portal.tsx L43-58）；文件经 `excalidraw-app/data/firebase.ts` 走 Firebase Storage，场景 JSON 先 AES-GCM 加密再上传。调和逻辑在 `packages/excalidraw/data/reconcile.ts`：

```ts
export const shouldDiscardRemoteElement = (localAppState, local, remote) => {
  if (local &&
    (local.id === localAppState.editingTextElement?.id ||   // 本地正在编辑
     local.id === localAppState.resizingElement?.id ||
     local.id === localAppState.newElement?.id ||
     local.version > remote.version ||                       // 本地更新
     (local.version === remote.version &&
      local.versionNonce <= remoteNonce)))                   // 同版本确定性裁决
    return true;
  return false;
};
```

"本地正在编辑的元素拒绝被远端覆盖"是实际使用中很关键的细节，避免协作时输入被打断。数组顺序由 `@excalidraw/element` 的 `syncMovedIndices`/`syncInvalidIndices` 维护（`packages/element/src/Scene.ts`），`Scene` 类作为场景状态中枢，向 React 组件广播变更。

### 5.4 撤销/重做与动作系统：Delta 驱动（packages/element/src/store.ts + packages/excalidraw/actions/）

撤销重做不是全量快照而是增量 delta：`store.ts` 定义 `StoreSnapshot` → `StoreDelta.calculate(prev, next)`（拆成 `ElementsDelta` + `AppStateDelta` 两部分），并区分 `DurableIncrement`（进历史）与 `EphemeralIncrement`（不进历史）两档变更（store.ts L480-530）。`history.ts` 的 `History`/`HistoryDelta` 类消费这些 delta。用户操作则统一收敛在 `actions/` 目录约 40 个 action 文件（actionAlign、actionFlip、actionExport……），每个 action 声明可执行的上下文（`App.tsx` 作为事件分发中枢，11,493 行，用 `App.bucketFill.ts`、`App.clipboard.ts` 等 partial 文件拆分事件处理逻辑）。

## 6. 构建与运行

来自根 `package.json` scripts 与 README/CONTRIBUTING：

```bash
yarn                        # 安装（yarn 1 workspaces，node>=18）
yarn start                  # 本地启动 excalidraw-app（Vite dev server）
yarn build                  # 构建 excalidraw-app 生产包
yarn build:packages         # 按依赖序构建各包 ESM 产物
yarn test                   # Vitest（watch）
yarn test:all               # typecheck + eslint + prettier + vitest 全量检查
yarn test:coverage          # Vitest 覆盖率（@vitest/coverage-v8）
yarn test:update            # 更新快照（CLAUDE.md 要求提交前执行）
docker compose up           # Dockerfile + docker-compose.yml 容器化部署
```

## 7. 测试与工程质量

- 规模：172 个 `*.test.ts(x)` 文件；分布以 `packages/excalidraw/tests`（64）和 `packages/element/tests`（28）为主，math/element/actions/collab/firebase 均有覆盖（find 统计）。
- 风格：单元 + 组件测试用 Vitest + jsdom，含 13 个快照文件（`packages/excalidraw/tests/__snapshots__`）；E2E 用 Playwright（`excalidraw-app/package.json` devDeps；`excalidraw-app/tests/` 下有 collab.test.tsx、firebase.test.ts 等集成测试）。
- CI：`.github/workflows/` 含 lint.yml、test.yml、test-coverage-pr.yml（PR 覆盖率门禁）、size-limit.yml（bundle 体积门禁）、semantic-pr-title.yml（PR 标题规范）、locales-coverage.yml（翻译覆盖率）、stats.yml（性能统计）。
- 观感：工程纪律严整——typecheck/lint/prettier 三道静态关 + 快照回归 + bundle 尺寸门禁；元素模型、调和、分数索引等协作关键路径有专门测试与开发期校验（`Scene.ts` 的 `validateIndicesThrottled` 在 dev/test 环境抛错）。

## 8. 社区与维护状态

- 活跃度：gh api 显示最近 push 2026-10-29（分析前一天）；`packages/excalidraw/CHANGELOG.md` "Unreleased" 有 2026-10-18 的 API 变更条目（PR #12347 `exportScene`），说明正处于高频迭代期。
- Release 节奏：CHANGELOG 每条带 PR 链接，遵循 Features/Fixes/Chore/Refactor 分组；`scripts/release.js` + `autorelease-excalidraw.yml` 自动化发布 npm 包（latest/next/test 三通道）。
- 贡献结构：15,911 forks、3,436 open issues（gh api）；i18n 由 Crowdin 社区翻译（59 语言包）；CONTRIBUTING.md 指向 dev-docs 文档站，且有专门的 `semantic-pr-title` 机器人把关。
- 资金/商业：README 徽章显示 Discord 社区与 Excalidraw+ 商业版（plus.excalidraw.com）并存，开源核心 + 托管服务模式；贡献者数量未逐一确认。

## 9. 亮点与风险

亮点：

1. 元素模型为协作而生：不可变对象 + version/versionNonce/fractional index 的组合让冲突解决、撤销重做、多人排序都有确定性答案（`packages/element/src/types.ts`、`data/reconcile.ts`、`element/src/store.ts`）。
2. E2E 加密协作的完整工程范本：加密在客户端完成、密钥不入库，且"本地编辑中元素不被远端覆盖"等细节考虑周全（`data/encryption.ts`、`excalidraw-app/collab/`）。
3. 库/应用分层干净：`packages/element`、`packages/math` 零 UI 依赖，核心可独立测试复用；npm 库与官方站互不污染（CLAUDE.md、目录结构佐证）。
4. Delta 化的 undo/redo 与容错渲染：历史记录只存增量省内存；单元素渲染失败不拖垮整个画布或导出（`store.ts`、`renderer/staticScene.ts`）。
5. 工程纪律与基础设施成熟：三道静态检查 + 覆盖率/bundle 门禁 + 自动发布 + 59 语言 i18n 管道。

风险/局限：

1. 巨型文件集中：`App.tsx` 11,493 行、`interactiveScene.ts` 约 2,200 行，虽用 partial 文件与 action 系统缓解，新贡献者的阅读成本仍高。
2. 协作服务端不在本仓库：socket.io 服务器与 Firebase 配置是部署方自己的事，开箱自建协作需额外工作（`excalidraw-app/collab/Portal.tsx` 只含客户端）。
3. 冲突模型是"元素级 last-write-wins"而非 OT/CRDT 文本合并：同元素并发细粒度编辑（如同一段文字的两个字符）仍会整段丢一边（由 reconcile.ts 语义决定）。
4. 版本 0.18 阶段仍以 breaking change 迭代（CHANGELOG Unreleased 中 `exportScene` 替换导出 API），嵌入式使用方升级有迁移成本。

## 10. 总结评价

Excalidraw 是"白板类产品里工程质量天花板"级别的开源项目：手绘风格让它出圈，而真正支撑 13 万星的是一套为实时协作精心设计的元素模型（不可变 + 版本向量 + 分数索引）、克制而完整的加密协作实现，以及严整的 monorepo 工程纪律。它适合三类人：想把白板/图表能力嵌入自己产品的开发者（npm 库 + iframe 嵌入）；研究实时协作、冲突调和、增量 undo/redo 的工程师（element/store/data 三处源码是最好的教科书）；以及单纯想要一个隐私安全、本地优先画图工具的用户（excalidraw.com 即开即用，PWA 离线可用）。
