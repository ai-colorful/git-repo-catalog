# preactjs/preact 项目分析

> 星数 41500（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：一个约 4kB 的 React 替代品，用自研的同步递归 Virtual DOM diff 提供与 React 相同的现代组件 API（hooks/class 组件），是前端"小而精"路线的代表作。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | preactjs/preact | package.json `repository` |
| 当前版本（main） | 11.2.0 | package.json（README 自述 "Fast 4kB alternative to React"，v10 时代为 3kB） |
| 星数 | 41500（任务快照）；gh api 2026-11-14 实测 39,005 | gh api repos/preactjs/preact |
| 语言 | JavaScript（含 TypeScript 类型声明与类型测试） | src/*.js、src/*.d.ts |
| 许可证 | MIT | LICENSE、package.json |
| 创建时间 | 2015-09-11 | gh api `created_at` |
| 最近 push | 2026-11-13（分析前一日） | gh api `pushed_at` |
| 最新 release | 11.2.0（2026-11-06） | gh api releases |
| 贡献者 | 358 人 | gh api contributors 分页头（per_page=1，last=358） |

## 2. 项目是什么，解决什么问题

Preact 解决的问题是 React 太重：React 18/19 全家桶（react + react-dom）下载体积大、运行时复杂。Preact 用一个约 4kB（gzip）的包提供 React 的核心编程模型——JSX/组件/hooks/Context/Ref/Portal——并借 `preact/compat` 兼容层直接复用 React 生态（将 bundler 的 `react`/`react-dom` 别名到 `preact/compat` 即可，见 vitest.config.mjs 中的 alias 配置，项目自身测试就是这么跑 compat 的）。

它的实现哲学与 React 明显不同：React 18+ 采用 Fiber 可中断的并发渲染架构（代码量数十万行级），Preact 则坚持"同步递归 diff"——`diff()` 一旦开始就把整棵子树的 DOM 更新做完，只把"组件级重渲染"排进微任务队列按深度排序批量执行（src/component.js 的 `enqueueRender`/`process`）。牺牲时间切片，换来 2000 行级别的核心实现与更低常驻开销；Suspense/流式 SSR 通过"渲染时抛出 Promise → 捕获后挂起、保留 DOM、Promise resolve 后重放"实现（src/diff/index.js 的 catch 分支与 `$s` 注释标记区间）。

main 分支正处于 v11 世代（2026-09-30 发布 11.0.0）：v11 相比 v10 的显著变化包括 createPortal 收进核心包（src/create-portal.js）、流式水合（hydrate 挂载 SSR 流，`$s` 范围标记续接）、LIS 最小移动集的子节点对账算法（src/diff/children.js）、MathML 支持与 scoped custom element registry 支持（src/diff/index.js `diffElementNodes`）。

单仓多包架构：核心 `preact` 与 `preact/compat`、`preact/hooks`、`preact/debug`、`preact/devtools`、`preact/jsx-runtime`、`preact/test-utils` 全部在同一仓库内（compat 虽保留 `preact-compat` 私有包名，但已并入本仓，见 compat/package.json 与根 package.json 的 `exports` 表）。SSR 渲染器 `preact-render-to-string` 为独立仓库，在根 package.json 中作为可选 peerDependency。

## 3. 技术栈与依赖

- 语言/运行时：纯 JavaScript（ESM 源码）+ JSDoc/TypeScript 声明（src/internal.d.ts、src/index.d.ts，dom.d.ts 达 113KB，是手工维护的完整 DOM 类型表）。
- 构建：Rollup 4 → Babel（含自研 `babel-plugin-fast-rest`，scripts/build.mjs）→ Terser；`mangle.json` 统一做内部属性名混淆（regex `^_[^_]`），保证压缩产物属性名稳定一致。
- 测试：Vitest 5（browser 模式，Playwright Chromium 驱动真实浏览器）+ @vitest/coverage-v8（Coveralls 徽章）；另有 `tsc` 类型测试与 node 环境测试。
- 代码质量：oxlint + oxfmt（Rust 实现，.oxlintrc.json/.oxfmtrc.json）、husky + nano-staged 预提交格式化。
- 工程化：pnpm 11（packageManager 锁定 pnpm@11.3.0、node 22.22.2），pnpm-workspace.yaml 含 `minimumReleaseAge` 供应链防护（依赖发布 1440 分钟后才可安装）、`strictDepBuilds`、`trustPolicy: no-downgrade`。
- 运行时依赖：核心包零运行时依赖；`@preact/signals` 仅出现在测试/开发依赖（test/browser/signals.test.jsx 验证 signals 集成）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| src/ | 核心包 `preact`：vnode 创建、diff 引擎、组件基类、render/hydrate、Context、Portal |
| src/diff/ | diff 引擎四件套：index.js（组件/元素 diff 主流程）、children.js（子节点对账）、props.js（DOM 属性）、catch-error.js（错误边界捕获） |
| compat/ | `preact/compat`：React 全 API 兼容层（Children、PureComponent、forwardRef、memo、Suspense、lazy、use/useTransition 等，compat/src 共 1101 行） |
| hooks/ | `preact/hooks`：全部 hooks 实现（hooks/src/index.js 单文件 566 行） |
| debug/ | `preact/debug`：开发期警告（prop 校验 check-props.js、组件栈 component-stack.js） |
| devtools/ | `preact/devtools`：对接 React DevTools 协议 |
| jsx-runtime/ | `preact/jsx-runtime` 与 jsx-dev-runtime：编译器输出的自动 JSX runtime |
| test-utils/ | `preact/test-utils`（含 act） |
| test/ | 核心包测试（browser/、node/、shared/、ts/ 类型测试、fixtures/） |
| demo/ | pnpm workspace 成员，本地演示页 |
| config/、scripts/ | 构建辅助（compat-entries.js、build.mjs、babel-plugin-fast-rest.mjs） |
| benchmarks/ | git submodule 指向 preactjs/benchmarks（tarball 中为空壳，内容未取） |
| .github/workflows/ | 8 个 workflow：ci.yml、build-test.yml、size.yml、benchmarks.yml、release.yml 等 |

组织方式评述：单仓多包（非独立 npm 包发布形态）——所有子包以目录形式存在、由根 package.json 统一 `exports` 映射并在构建后发布；好处是跨包原子修改（如 v11 给核心加 `_mask` 标记供 hooks 的 useId 使用）一次提交完成，compat 的 Suspense 可直接 import 核心内部常量（compat/src/suspense.js 引 `../../src/constants`）。

## 5. 核心架构与关键实现

### 5.1 扩展机制的基石：options 钩子（src/options.js，16 行）

整个生态建立在核心 diff 流程埋的回调点上：`options._diff/_render/diffed/_commit/_root/unmount/_catchError/_hook`。hooks、debug、devtools、compat 全部通过"包一层旧回调"的方式挂载（如 hooks/src/index.js 顶部保存 `oldBeforeDiff` 后重写 `options._diff`）。这让核心保持极小，附加功能按需加载。VNode 本体只用 `VNODE_MARKER = Symbol.for('pv')` 构造器标记（src/constants.js），diff 入口即校验，拒绝外来对象进入 diff（防注入/防混淆，对应 test/browser/hardened-js.test.jsx）。

### 5.2 diff 主流程（src/diff/index.js，821 行）

`diff(parentDom, newVNode, oldVNode, ...)` 是同步递归函数：函数组件先取/建组件实例（类组件直接 new，函数组件包一个 `BaseComponent` 并令 `c.render = doRender`，见文件末尾 `doRender`——函数组件与类组件在 diff 层面统一），执行生命周期与 `shouldComponentUpdate` bail-out（只跳过子树遍历，`break outer`），然后 `do { render } while (dirty && count<25)` 循环渲染。核心代码骨架：

```js
// 函数组件的兜底实例与渲染循环（src/diff/index.js:130-141, 256-260）
newVNode._component = c = new BaseComponent(newProps, componentContext);
c.constructor = newType;
c.render = doRender;
...
do {
    c._bits &= ~COMPONENT_DIRTY;
    if (renderHook) renderHook(newVNode);
    tmp = c.render(c.props, (c.state = c._nextState), c.context);
} while (c._bits & COMPONENT_DIRTY && ++count < 25);
```

错误/Suspense 处理在同一文件的 catch 分支：若抛出物是 Promise（`e && e.then`），不销毁 DOM，把该子树标记 `MODE_SUSPENDED` 并保留起止注释标记（`$s` 范围，`rangeEnd()` 解析嵌套深度），Promise resolve 后从标记处续接 diff——这就是流式 SSR 水合与 Suspense 共用的一套"停车站"机制。Portal 也在此文件内联实现：vnode props 带 `_parentDom` 即切换渲染容器并接管命名空间（src/diff/index.js:295-316）。

### 5.3 子节点对账：skew 匹配 + LIS 最小移动集（src/diff/children.js，523 行）

这是 preact diff 的精华。`constructNewChildrenArray` 先对每个新 child 做 `findMatchingIndex` 匹配旧 child（key+type 相等；优先命中"斜移后同位"——`skewedIndex = i + skew`，skew 随插入/删除/移动动态调整，避免大多数场景的线性搜索；未命中才向两侧线性扫描，`MATCHED` 位防重复匹配）。一旦检测到真实乱序（`moved=1`），执行 patience sorting 求"旧索引最长递增子序列"，LIS 上的节点原地不动，其余标 `INSERT_VNODE`，把 DOM 移动次数压到最小：

```js
// LIS：二分求 tails，回溯标记必须移动的节点（src/diff/children.js:315-351）
let tails = [], lisLengths = [];
for (i = 0; i < newChildrenLength; i++) {
    childVNode = newChildren[i];
    if (childVNode && childVNode._flags & LIS_CANDIDATE) {
        skew = 0; let hi = tails.length;
        while (skew < hi) { /* 二分插入点 */ }
        tails[skew] = childVNode._index;
        lisLengths[i] = skew + 1;
    }
}
// 回溯：不在 LIS 上的 matched 节点标记 INSERT_VNODE
```

DOM 移动本身用新标准 `parentDom.moveBefore`（无状态损失的真移动）并回退到 `insertBefore`（src/diff/children.js:417-425）。代码中多处注释强调保持 V8 fast path（避免闭包捕获导致 context allocation、保持 Smi 比较等），可见性能打磨极深。

### 5.4 hooks：完全寄生在 options 钩子上的实现（hooks/src/index.js，566 行）

hooks 状态存于组件实例 `__hooks._list[index]`，靠 `options._render` 在每次渲染前把 `currentIndex` 归零实现"按调用顺序索引"。要点：`useState` 即 `useReducer(invokeOrReturn)`，dispatch 里 `ObjectIs` 比较后写 `_nextValue` 并调用核心 `setState({})`；首次使用时给组件安装一个 `shouldComponentUpdate`，扫描所有 hook 的 `_nextValue` 与 `_value` 是否相等实现"批量 setState 后值未变则整体 bail-out"（hooks/src/index.js useReducer 分支）。`useEffect`（passive）被推入 `_pendingEffects`，经 `requestAnimationFrame`+setTimeout 兜底在绘制后执行（RAF_TIMEOUT=35ms）；`useLayoutEffect` 进 `_renderCallbacks` 在 commit 阶段同步执行。`useRef` 就是 `useMemo(() => ({current}), [])`。`useId` 依赖核心 vnode 的 `_mask` 计数（'P0-1' 形式），SSR 一致。卸载时 passive 清理延迟到绘制后统一执行、错误路由回存活的边界组件（hooks/src/index.js 顶部 `unmountCleanups` 注释与 #4299）。

### 5.5 compat 兼容层：把 React API 映射到 Preact 语义（compat/src，1101 行）

- 伪装 React：`version = '19.0.0'`、元素带 `REACT_ELEMENT_TYPE = Symbol.for('react.element')`（compat/src/index.js、render.js:24），第三方库的 `isValidElement` 检测直接通过。
- 归一化：`options.event` 给事件对象补 `persist/nativeEvent/isPropagationStopped`；className↔class、style 数值自动加 px、onChange→oninput、驼峰属性转连字符（compat/src/render.js `handleDomVNode`）。
- React 19 API：`use`（Promise/Suspense 联动）、`useSyncExternalStore`、`useTransition`/`startTransition`、`useDeferredValue`、`useEffectEvent`、`useInsertionEffect`（compat/src/hooks.js、render.js）。
- Suspense 独立实现（compat/src/suspense.js，275 行）：重写 `options._catchError`，Promise 异常向上找 `_childDidSuspend` 的 Suspense 组件，fallback 展示期间克隆/挂起子树（`detachedClone` 清理 passive effects），resolve 后还原。
- 附带 `compat/server`（SSR）、`compat/client`（React 18 createRoot 形态）、`compat/scheduler`、`compat/jsx-runtime` 等入口（根 package.json exports 表）。

与 React 的真实成本差异：React Fiber 支持并发优先级与时间切片，代价是调度器+双缓冲 fiber 树的复杂度与体积；Preact 选择同步提交（一旦开始 diff 不可中断）、组件级微任务批处理 + 深度排序保证父先于子，绘制后跑 passive effects——在绝大多数真实 UI 场景行为一致，但长任务无法让出主线程（这是架构上的明确取舍，README 以"transparent asynchronous rendering with a pluggable scheduler"提供 `options.debounceRendering`/`requestAnimationFrame` 两个调度注入口）。

## 6. 构建与运行

来自 package.json scripts 与 CONTRIBUTING.md：

```bash
# 环境：node 22（volta/engineStrict 锁 22.22.2），pnpm@11.3.0
pnpm install            # 含 prepare: husky + playwright install chromium
pnpm build              # node ./scripts/build.mjs，Rollup+Babel+Terser 出 dist/*.mjs
pnpm dev                # watch 构建；dev:hooks / dev:compat 单包 watch
pnpm lint               # oxlint + tsc（jsconfig-lint.json）
pnpm format             # oxfmt --write
pnpm test               # build + lint + test:unit
pnpm test:vitest        # COVERAGE=true，vitest browser 模式（Playwright Chromium）
pnpm test:vitest:min    # MINIFY=true：对压缩产物跑全量测试（验证 mangle 后行为）
pnpm test:ts            # tsc 类型测试（test/ts/ 与 compat/test/ts/）
```

亮点：`test:vitest:min` 把"压缩后属性名混淆是否破坏运行时"纳入常规门禁；CI（.github/workflows/ci.yml）还含 compressed size 检查（size.yml）与基准测试（benchmarks.yml，改动 src 才触发，paths-filter 过滤）。

## 7. 测试与工程质量

- 规模：109 个测试文件、约 42,458 行、约 1406 个 it/test 用例（本次统计：core test/ 42 文件 22,373 行；compat/test 35 文件 11,377 行；hooks/test 15 文件 4,968 行；debug/test 10 文件 2,372 行）。
- 风格：真实浏览器行为测试（Vitest browser mode + Playwright，test/browser/*.test.jsx 按 API 面/生命周期/特性分文件，如 keys.test.jsx、hydrate.test.jsx、lifecycles/*）；少量 node 测试（test/node）；类型层面测试（*.test-d.tsx）。
- 覆盖：coverage-v8 + Coveralls 徽章（README）；CI 对 js 变更做 size 回归与基准回归。
- 质量观察：源码注释密度高且指向真实 issue 编号（如 #3883、#5065、#4299）；对 V8 优化路径有明确意识（children.js 中两处"keep this a plain loop"注释）；供应链防护（minimumReleaseAge/strictDepBuilds）在同类项目中相当先进。

## 8. 社区与维护状态

- 活跃度：gh api `pushed_at` 为 2026-11-13（分析前一日），37 个 open issues，仓库 2015 年创建、持续维护 11 年。
- Release 节奏（gh api releases 实测）：v11 线密集迭代——11.0.0（2026-09-30）→ 11.0.1（10-08）→ 11.0.2（10-23）→ 11.1.0/11.1.1（10-31）→ 11.2.0（11-06），六周 5 个稳定版；同时并行维护 v10 线（10.29.7/10.29.8，2026-07/08）。发布由 .github/workflows/release.yml 自动化（npm provenance 开启，package.json publishConfig）。
- 贡献者：358 人（gh api 分页统计）；核心团队 markers 明显（CODEOWNERS 存在 .github/CODEOWNERS）。
- 资金：OpenCollective（package.json funding），README 列 Deno、Songsterr、SentDM 等赞助方。
- CI 分支监听含 `v11`/`restructure` 历史分支名，可佐证 v11 大重构路径（.github/workflows/ci.yml）。

## 9. 亮点与风险

亮点：
1. diff 引擎是教科书级实现：skew 匹配 + LIS 最小移动集 + moveBefore，核心仅 821+523 行（src/diff/index.js、src/diff/children.js），性能注释直指 V8 fast path。
2. options 钩子扩展架构：hooks/debug/devtools/compat 全部零侵入寄生，核心包保持极小且可选功能按需引入。
3. 工程门禁硬核：压缩产物跑全量测试、真实浏览器测试矩阵 42k 行、类型测试、size 回归、基准回归、供应链防护俱全。
4. compat 兼容层覆盖到 React 19 API（use/useTransition/useSyncExternalStore/useEffectEvent），版本号伪装规避第三方库检测。
5. v11 快速演进中（流式水合、Portal 入核、MathML、scoped custom elements），近三个月 5 个稳定版。

风险/局限：
1. 无时间切片：同步递归 diff 在超大树上单次更新不可中断，长任务阻塞主线程（与 React Fiber 的根本取舍，见 src/diff/index.js 与 src/component.js）。
2. compat 是"行为模拟"而非 100% 等价：并发特性（Concurrent Rendering 的优先级调度）、部分边界行为与 React 有差异，迁移复杂 React 应用需回归测试（compat/src 大量 #issue 注释即为行为差异修正记录）。
3. benchmarks 为 submodule，本仓 tarball 不含其内容，基准结论未能直接核验（未确认）。
4. 版本快照注意：main 为 v11.2.0（README 口径 4kB），npm 上大量教程/资料仍基于 v10（3kB 口径），两者存在 API 细节差异（如 createPortal 移入核心）。

## 10. 总结评价

Preact 用约 5000 行运行时代码（全部子包合计，本次实测 5,171 行 JS）复刻了 React 的编程模型，核心 diff 引擎在算法与工程细节上达到极高水准，是"读源码学 VDOM 对账"的首选项目；options 钩子架构展示了如何用 16 行代码支撑一个插件生态。适合：追求体积与性能的轻量前端项目（尤其嵌入式/插件/Widget 场景）、需要复用 React 生态但不想背 React 运行时成本的团队；适合精读人群：想理解 vdom diff、hooks 原语、错误边界/Suspense 底层机制与浏览器级测试工程的工程师。维护活跃、v11 迭代迅速，长期风险主要在其对 React 行为"追赶式兼容"的持续成本。
