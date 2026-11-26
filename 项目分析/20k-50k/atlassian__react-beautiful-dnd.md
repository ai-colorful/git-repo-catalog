# atlassian/react-beautiful-dnd 项目分析

> 星数 33,500（快照 2026-10-24；2026-11-21 gh api 复核 33,878）｜主语言 JavaScript（gh api 口径；源码用 Flow 类型标注，非 TypeScript）｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：Atlassian 出品的 React 列表拖拽库，以"漂亮 + 无障碍"为核心卖点，是 2017-2022 年间 React 生态事实上的列表排序方案；**已于 2024-10 官方宣布弃用、2025-04-30 归档**，官方推荐迁移至自家后继项目 Pragmatic Drag and Drop。

| 事实 | 值 |
|---|---|
| 名称 | react-beautiful-dnd（rbd） |
| 星数 / fork | 33,878 / 2,683（gh api 2026-11-21） |
| 语言 | JavaScript + Flow 类型标注（`src/` 全部 `// @flow`，无 .ts 文件） |
| 许可证 | Apache-2.0（`package.json`；`LICENSE` 为精简版权声明故 gh api 显示 NOASSERTION） |
| 归档状态 | **archived: true**（只读）；README 首段即 "🔒 Archived" 弃用声明（`src/README.md`） |
| 最终版本 | v13.1.1（2022-08-30，GitHub Releases） |
| 最后 push | 2025-08-18（gh api） |
| 维护者 | Alex Reardon（作者，已个人停止维护）→ Atlassian 团队接棒（`src/README.md` Maintainers 段） |

## 2. 项目是什么，解决什么问题

`react-beautiful-dnd` 解决的是**列表场景下的拖拽重排**：垂直列表、水平列表、跨列表移动、嵌套列表，而不是像 `react-dnd` 那样的通用拖拽原语（README "Not for everyone" 一节明确做了这一定位对比，`src/README.md`）。它刻意绕开了浏览器原生 HTML5 DnD API（该 API 交互不一致且不可控样式），全部基于鼠标/触摸/键盘事件自行实现拖拽状态机，因此能做出符合物理直觉的动画（抬起、落点动画、惯性位移），这是它"beautiful"的来源（`docs/about/animations.md`）。

第二个差异化支柱是**无障碍**：完整键盘操作（空格抬起、方向键移动、跨列表移动、combine）、屏幕阅读器实时播报、焦点管理，并内置英文播报文案且支持完全定制与国际化（`docs/about/accessibility.md`）。README 的 Motivation 一节原话是"create beautiful drag and drop for lists that anyone can use - even people who cannot see"。

第三个卖点是**性能与零侵入**：不创建额外包裹 DOM 节点（对 flexbox 和焦点管理友好）、拖拽期间不触发 React 重渲染（样式走 style marshal 直接改内联样式，见第 5 节）、支持 10,000 条目 @ 60fps 的虚拟列表模式（`docs/patterns/virtual-lists.md`）。

对外 API 极小：`<DragDropContext />`、`<Droppable />`、`<Draggable />` 三个组件加 `resetServerContext()`（`src/index.js`），配合 `onDragStart/onDragUpdate/onDragEnd` responders，这是它被大规模采用的重要原因。

## 3. 技术栈与依赖

- **语言/类型**：JavaScript，内部用 Flow 精确对象类型 `{| ... |}` 做强类型约束（`flow-bin` 0.110.1，`.flowconfig`）；对外同时导出 public Flow types（`src/index.js`），TypeScript 用户依赖社区 `@types/react-beautiful-dnd`（`docs/guides/types.md`）。
- **React 生态**：React/ReactDOM peerDependencies `^16.8.5 || ^17.0.0 || ^18.0.0`（`package.json`）；视图层全部函数组件 + Hooks（16.8+ 才支持正是为此）。
- **运行时依赖仅 6 个**（`package.json` dependencies）：`redux` + `react-redux`（拖拽状态机）、`css-box-model`（盒模型测量）、`memoize-one` / `use-memo-one`（计算缓存，后者是作者自写的 hook 级 memo）、`raf-schd`（requestAnimationFrame 节流）。
- **构建**：Rollup 产出 cjs/esm/umd 三格式（`rollup.config.js`），esm 产物 gzip 约 32.6KB（`.size-snapshot.json`）。
- **测试与开发**：Jest、Cypress、Storybook 5、Lighthouse（a11y 自动审计）、enzyme + @testing-library/react。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `src/state/` | 拖拽状态机：reducer、action-creators、11 个 Redux middleware、auto-scroller、dimension-marshal、droppable 碰撞计算等 |
| `src/view/` | React 视图层：三大组件、sensor 框架（use-sensor-marshal）、announcer、style marshal、placeholder 等 |
| `src/types.js` | 全局 Flow 类型定义（485 行，含 5 相位状态联合类型） |
| `docs/` | 极完善的文档（about / api / guides / patterns / sensors / support 六类，约 30 篇 md） |
| `stories/` | Storybook 示例，含 `stories/src/virtual/react-window/` 等虚拟列表示例 |
| `test/` | Jest 单元 + 集成测试（301 个文件） |
| `cypress/` | 浏览器端到端测试（6 个 spec） |
| `.circleci/` | CircleCI 配置（node 10.24.0 镜像） |
| `src/debug/` | 开发调试 middleware（log / user-timing / action-timing） |
| `src/view/drag-drop-context/` | Context 组件本体 + 启动校验（React 版本/doctype/错误边界 error-boundary.jsx） |
| `src/view/use-style-marshal/` | 拖拽期间直接写内联样式的样式编组（绕开 React 渲染） |
| `src/view/placeholder/` | 拖拽占位节点渲染 |

组织方式：**单包单体**（非 monorepo），`src/` 下 231 个 js/jsx 文件，按 state（纯逻辑、无 DOM）/ view（DOM 与 React）严格分层，state 层不 import React——这个分层使状态机可以脱离 React 完整测试。组件间通过 props/context 传递 store，`connected-draggable.js` / `connected-droppable.js` 用 react-redux `connect` 订阅状态（`src/view/draggable/connected-draggable.js`）。

## 5. 核心架构与关键实现

### 5.1 类型化 action 流 + Redux 状态机

拖拽全程是一个显式相位机，`src/types.js:331-393` 定义五相位联合类型：

```
IDLE → COLLECTING（收集尺寸）→ DRAGGING → DROP_PENDING → DROP_ANIMATING → IDLE
```

所有事件统一转化为 Redux action（`src/state/action-creators.js`）：`LIFT`、`INITIAL_PUBLISH`、`PUBLISH_WHILE_DRAGGING`、`UPDATE_DROPPABLE_SCROLL`、`DROP_ANIMATION_FINISHED`、`FLUSH` 等，每个 action payload 用 Flow 精确类型描述，例如：

```js
export type LiftArgs = {|
  // lifting with DraggableId rather than descriptor
  // as the descriptor might change after a drop is flushed
  id: DraggableId,
  clientSelection: Position,
  movementMode: MovementMode,
|};
```

（`src/state/action-creators.js`；注释说明用 id 而非 descriptor 是因为 drop flush 后 descriptor 可能已变——这类设计动机注释贯穿全库。）`src/state/create-store.js` 用 `applyMiddleware` 组装 11 个 middleware（style → dimensionMarshalStopper → lift → drop → dropAnimationFinish → dropAnimationFlushOnScroll → pendingDrop → autoScroll → scrollListener → focus → responders），并保留了接入 Redux DevTools 的口子。reducer 本体仅 457 行（`src/state/reducer.js`），复杂度分散在各 middleware 与纯函数模块（`get-drag-impact/`、`move-in-direction/`、`calculate-drag-impact/`）中。

关键设计是**拖拽期间不走 React 渲染路径**：位移通过 style middleware 直接写内联 transform/style（`src/view/use-style-marshal/`），middleware 注释明确"Style updates do not cause more actions"，且强调样式更新必须发生在 responders 之前以规避 Firefox transition 残留（`src/state/create-store.js` 注释）。

### 5.2 Sensor 统一抽象（鼠标/键盘/触摸的感知层）

`src/view/use-sensor-marshal/use-sensor-marshal.js`（495 行）是三类输入的统一管理层：核心是一个**锁机制**（`lock.js`），相位 `PRE_DRAG → DRAGGING → COMPLETED`，sensor 通过 `tryGetLock()` 竞争唯一拖拽权，拿到 `PreDragActions/FluidDragActions/SnapDragActions` 三种受控 API 才能驱动拖拽。锁失效或相位错配时给出明确的开发者警告（`isActive()` 的 warning 文案）。

三个内置 sensor（`src/view/use-sensor-marshal/sensors/`）实现各自的监听逻辑：
- **mouse**（`use-mouse-sensor.js`，386 行）：mousedown 先进入 pending 相位，位移超过 `sloppyClickThreshold` 才正式 lift，期间键盘按下或 window 滚动会取消 pending（对注释与 `pending` 相关逻辑）；
- **keyboard**（`use-keyboard-sensor.js`）：空格抬起/落下、方向键单步移动、Escape 取消、pageUp/pageDown/home/end 触发 `scrollJumpKeys` 快速跳转，走 SNAP（吸附）模式而非 fluid 模式；
- **touch**：基于 touchstart 的延迟启动与强制滚动检测。

自定义 sensor 是公开 API（`SensorAPI` 类型导出自 `src/index.js`，文档 `docs/sensors/sensor-api.md`），官方用它实现了脚本化拖拽测试。所有 DOM 事件绑定经由统一的 `bind-events`（`src/view/event-bindings/`）收口，保证共享 `event.preventDefault`/`stopPropagation` 语义。

### 5.3 可访问性设计

- **屏幕阅读器公告**：`src/view/use-announcer/use-announcer.js` 在 body 挂载一个视觉隐藏 div（`aria-live="assertive"` + `aria-atomic="true"`，id 为 `rbd-announcement-{contextId}`），拖拽状态变化由 responders middleware 调用 `announce()` 写入 textContent 触发播报（`src/state/middleware/responders/responders-middleware.js`）。卸载时延迟移除，避免挂载期公告丢失；若 `onDragEnd` 中先卸载了 Context 会给出专门警告提示先 announce。
- **元素描述**：`src/view/use-hidden-text-element/use-hidden-text-element.js` 为每个 Draggable 生成 `aria-describedby` 隐藏文本（教屏幕 reader 用户如何操作，如"按空格拾起"），id 规则 `rbd-hidden-text-{contextId}-{uniqueId}`。
- **焦点管理**：`src/view/use-focus-marshal/` 记录拖柄焦点，拖拽结束重排后焦点跟回原/新位置的 Draggable。
- **CI 保障**：`test:accessibility` 用 Lighthouse 对 Storybook 页面自动打分并解析报告（`a11y-audit-parse.js`、`package.json` scripts）。

### 5.4 虚拟列表集成（react-window 落点）

rbd **不内置虚拟化**，而是提供适配协议（`docs/patterns/virtual-lists.md`）：`<Droppable mode="virtual">` + `renderClone`（拖拽项滚出窗口被卸载时渲染克隆，复用 reparenting 机制）+ 要求开启 overscanning。官方示例在 `stories/src/virtual/react-window/{list,board}.jsx`（同时支持 react-virtualized、react-virtuoso）。state 层配套实现了拖拽中动态增删条目的能力：`src/state/publish-while-dragging-in-virtual/`（index.js + adjust-additions-for-scroll-changes.js + offset-draggable.js）处理 `PUBLISH_WHILE_DRAGGING` action，把新增条目按滚动偏移校正后合并进 impact 计算——这同样支撑了"拖拽过程中添加/移除条目"特性。

### 5.5 尺寸收集与碰撞模型

`src/state/dimension-marshal/` 是惰性尺寸收集器：`src/state/registry/` 中每个 Draggable 注册 `getDimension` 回调（`registry-types.js`），lift 时才批量测量（COLLECTING 相位），避免拖拽期间反复布局读取。碰撞判定基于中心点/四角与 droppable 边框盒的几何关系（`src/state/get-droppable-over.js` 用 `isPositionInFrame` 判定中心点归属，`get-center-from-impact/` 做反向求心），所有测量基于 `css-box-model` 的 border box。

## 6. 构建与运行

来自 `package.json` scripts（未实际运行，按规范仅静态阅读）：

```bash
yarn test              # jest --config ./jest.config.js
yarn test:browser      # cypress open（e2e）
yarn test:accessibility # lighthouse a11y 审计
yarn typecheck         # flow check --max-warnings=0
yarn validate          # prettier + eslint + stylelint + flow
yarn build             # rimraf dist && rollup -c（cjs/esm/umd）+ 生成 .flow 指针文件
yarn storybook         # start-storybook -p 9002
```

## 7. 测试与工程质量

- **规模**：`test/` 下 301 个测试文件，分 `test/unit/state/`（纯状态机）、`test/unit/integration/`（含 accessibility、三种 sensor、SSR 快照等子目录）与 `test/unit/docs/`（文档代码示例测试）；另有 `cypress/integration/` 6 个端到端 spec（reorder、move-between-lists、focus、reorder-virtual、content-security-policy）。
- **CI**：CircleCI（`.circleci/config.yml`，node 10.24.0），流水线含 install（greenkeeper 锁文件校验）→ validate（flow/eslint/prettier）→ 单测 → cypress → bundle size 检查（`SNAPSHOT=match` 对比 `.size-snapshot.json`）。
- **质量观察**：state/view 分层使核心逻辑可无 DOM 测试；dev 环境有大量启动校验（React 版本匹配 `check-react-version.js`、doctype 检查 `check-doctype.js`、ref 校验），错误信息带排查 Tips，工程化程度在同代库中属最高档。风险点：技术栈已明显老化（node 10 镜像、enzyme、Storybook 5、React 16.13 dev 依赖），归档后无人升级。

## 8. 社区与维护状态

- **贡献者**：121 人（gh api contributors 分页 Link 头，2026-11-21）；核心为 Atlassian 团队，作者 Alex Reardon 已声明不再亲自维护（`src/README.md` Creator 段）。
- **release 节奏**：v12（2019 末）→ v13.0.0（2020-02）→ v13.1.0（2021-03）→ **v13.1.1（2022-08-30，最终版）**，之后近三年无版本发布；open issues 642 个且仓库只读。
- **弃用时间线**（gh api 核实）：
  - 2024-10-14：官方公告 issue #2672 "react-beautiful-dnd is now deprecated"——npm 包标记 deprecated（安装时出 console 警告），并预告 6 个月后（2025-04-30）归档 GitHub 仓库；迁移建议指向自家 **Pragmatic drag and drop**（`atlassian/pragmatic-drag-and-drop`，官方定位为性能、灵活性与无障碍更好的新方案）。
  - 2025-04-30 起：仓库归档只读（gh api `archived: true`）；README 置顶 "🔒 Archived" 声明并链接迁移资源（issue #2672）。
- **社区 fork**：`hello-pangea/dnd`（原名 @hello-pangea/dnd）是最活跃的延续性 fork，gh api 显示 4,053 星、未归档、2026-11-20 仍有 push，主要修复 React 18 StrictMode 等兼容问题——为仍需 rbd API 的项目提供了事实上的维护出口。

## 9. 亮点与风险

**亮点**
1. 完整的"类型化 action 流 + 相位状态机 + middleware 管线"架构，纯逻辑层（state/）与 DOM 层（view/）彻底分离，`src/state/reducer.js` 仅 457 行承载全部相位迁移，是 Redux 架构教学级样本。
2. Sensor 锁抽象（`use-sensor-marshal`）把鼠标/键盘/触摸/自定义输入统一到同一套 `tryGetLock` + 受控 API 协议上，扩展性设计出色。
3. 无障碍做到行业标杆：aria-live 播报、aria-describedby 操作指引、焦点重排、键盘自动滚动、Lighthouse 进 CI（`src/view/use-announcer/`、`docs/about/accessibility.md`）。
4. 拖拽期间绕开 React 渲染直接操作样式（style marshal + memoize-one），配合惰性尺寸收集，实现了 10k 条目级别的性能（`docs/patterns/virtual-lists.md`）。
5. 文档质量罕见地好：`docs/` 六大主题约 30 篇，每篇均可从 README 索引直达。

**风险/局限**
1. **项目已死**：npm deprecated + GitHub 归档只读，无任何后续修复（含安全问题）；新项目不应采用。
2. React 18/StrictMode 兼容缺陷官方未修：issue #2350（"React 18 StrictMode rendering breaks ability to find node by it's draggable id"，39 条讨论）与 #2306 至归档仍处 open 状态——根因是 StrictMode 双渲染下 sensor marshal 按 id 找 DOM 节点的假设失效，v13.1.1 的 peerDeps 虽声明支持 ^18 但实际体验有坑。
3. 类型口径为 Flow 而非 TypeScript：TS 用户依赖社区 `@types/react-beautiful-dnd`，归档后类型包也不会再跟进新 API（实际无新 API）。
4. 只覆盖"列表重排"场景（不支持任意自由拖拽、不支持父子列表间移动），且属高抽象层封装，定制空间受限于其状态机。

## 10. 总结评价

react-beautiful-dnd 是"用户体验驱动的工程范式"的典范之作：用 Redux 相位机 + sensor 锁抽象 + 样式旁路渲染这套组合，在 2022 年之前几乎定义了 React 列表拖拽该有的样子，其无障碍实现至今仍有很高的学习价值。它的落幕同样有代表性——Atlassian 转向更贴近浏览器原语、性能天花板更高的 Pragmatic Drag and Drop，官方走完了"npm 弃用公告（2024-10）→ 归档只读（2025-04-30）→ 指明迁移路径"的标准退场流程；生态由社区 fork hello-pangea/dnd（4k+ 星、持续维护）事实接盘。适合谁：新项目直接选 Pragmatic DnD（需要 Atlassian 系）或 hello-pangea/dnd（想继续用 rbd API）；读源码则 rbd 依然值得——尤其推荐 `src/state/create-store.js`（middleware 编排）、`src/view/use-sensor-marshal/`（输入抽象）与 `src/view/use-announcer/`（a11y 实现）三条线，它们是"如何把复杂交互做成可测试的确定性状态机"的优质教材。
