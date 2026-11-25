# streamich/react-use 项目分析

> 星数 44600（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-06

## 1. 项目速览

一句话定位：React 自定义 Hook 的"瑞士军刀"合集，一个 npm 包提供 110+ 个覆盖传感器、UI、状态、副作用、生命周期等场景的现成 Hook（`package.json` description 即 "Collection of React Hooks"）。

| 项目 | 值 | 出处 |
| --- | --- | --- |
| 名称/版本 | streamich/react-use v17.6.1 | `package.json`、CHANGELOG.md |
| 星数 | 44600（快照）；gh api 实时 43,982 | gh api repos/streamich/react-use |
| 语言 | TypeScript（strict 模式） | `tsconfig.json` |
| 许可证 | Unlicense（公有领域，无署名限制） | `package.json`、`LICENSE` |
| 导出 API | index.ts 共 113 条 export：107 个 use* 与 6 个 create*（工厂），其中 8 条为无 default 的具名导出，useEnsuredForwardedRef 额外附带 ensuredForwardRef | `src/index.ts` grep 统计 |
| 创建时间 | 2018-10-27 | gh api |
| 最近 push | 2026-06-10（v17.6.1 发布同日） | gh api、CHANGELOG.md |
| 代码规模 | src/ 共 127 个文件、约 5,511 行 TS | find/wc 统计 |

## 2. 项目是什么，解决什么问题

react-use 是 React Hooks 时代的代表性工具库（README 自述为作者前作 `libreact` 的 Hooks 移植版，见 `README.md`）。它把浏览器/日常开发中反复出现的横切逻辑——电池状态、地理定位、鼠标/键盘事件、媒体查询、滚动位置、localStorage、防抖节流、挂载卸载——逐个封装成单一职责的自定义 Hook，开发者 `npm i react-use` 后按需 `import { useMedia } from 'react-use'` 即可，不必在每个项目里重写这些模板代码。

Hook 按 README 分为 7 大类（`README.md` 与 `docs/` 目录一一对应）：Sensors（传感器，27 条）、State（状态，27 条）、Side-effects（副作用，14 条）、Lifecycles（生命周期，11 条）、UI（9 条）、Animations（7 条）、Miscellaneous（1 条）。每个 Hook 在 `docs/` 下有独立文档页（docs/ 共 117 个 md），README 中每条目附 CodeSandbox/Storybook 在线 demo 链接，文档-示例配套完整。

Sensors 类监控浏览器/硬件环境（useBattery、useGeolocation、useMouse、useWindowSize、useMedia 等），State 类提供增强版 useState（useList、useMap、useSet、useStateWithHistory、useGetSet、useToggle 等），Side-effects 类包装异步与持久化（useAsync、useAsyncFn、useCookie、useLocalStorage、useSessionStorage、useTimeoutFn 等），Lifecycles 类是最薄的一层（useMount、useUnmount、useUpdateEffect 等各仅数行）。部分 README 条目成对收录近似 Hook（如 useHover/useHoverDirty 的受控 ref 与内部 ref 两种形态），因此 96 个条目对应 111 个实现文件（统计自 `README.md` 与 `src/` 文件清单）。

与竞品（ahooks、react-use 之外的 swiper 式大而全）相比，它的特点是"一 Hook 一文件"的极简风格：绝大多数实现只有几十行，直接读源码就能理解，适合作为 Hook 编写范本。项目同时通过 `factory/` 目录提供 `createReducer`（带 Redux 风格中间件）、`createGlobalState`（跨组件共享状态）、`createBreakpoint` 等工厂函数，把"生成定制 Hook"的模式也纳入 API（`src/factory/createReducer.ts`）。

需要注意它是 React 17 时代成型的库：devDependencies 锚定 React 17 与已弃用的 `@testing-library/react-hooks`（`package.json`），近年几乎无功能性迭代（见第 8 章）。

## 3. 技术栈与依赖

- 语言/编译：TypeScript 4.1（strict、target es5、输出 declaration），Babel 用于 Jest/Storybook（`tsconfig.json`、`babel.config.js`、`package.json`）。
- 运行时依赖仅 14 个，均为小而专的工具（`package.json` dependencies）：`js-cookie`（useCookie）、`nano-css`（useCss）、`screenfull`（useFullscreen）、`throttle-debounce`（useDebounce/useThrottle）、`fast-deep-equal`/`fast-shallow-equal`（useDeepCompareEffect 等）、`@xobotyi/scrollbar-width`（useScrollbarWidth）、`copy-to-clipboard`、`resize-observer-polyfill`、`react-universal-interface`、`tslib` 等。
- peerDependencies：`react: "*"`、`react-dom: "*"`（不锁 React 大版本，`package.json`）。
- 测试链路：Jest 26 + ts-jest + `@testing-library/react-hooks` 7 + `jest-localstorage-mock` + `@shopify/jest-dom-mocks`。
- 工程化：ESLint + Prettier + husky/lint-staged（pre-commit/pre-push 钩子）、Renovate 自动依赖更新（`renovate.json`）、semantic-release 自动发版、Storybook 6.4 做 demo 站（`stories/` 118 个文件）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `src/` | 全部 Hook 源码：根目录平铺约 101 个 `useXxx.ts`（一 hook 一文件） |
| `src/factory/` | 9 个 `createXxx.ts` 工厂（生成 Hook 的 Hook），如 createReducer、createGlobalState |
| `src/component/` | 仅 `UseKey.tsx` 1 个组件形态封装（render-prop 风格残留） |
| `src/misc/` | 5 个工具文件：util.ts（on/off/isBrowser）、hookState.ts、isDeepEqual.ts、types.ts |
| `src/index.ts` | 117 行统一出口，113 条 export |
| `docs/` | 117 个 md：7 个分类页 + 每 Hook 一页文档 |
| `stories/` | 118 个 Storybook story，全部 hook 的可交互 demo |
| `tests/` | 77 个文件，与 hook 一一对应的单测 |
| `.github/workflows/` | check-codebase.yml（CI）、release.yml（semantic-release） |

组织方式评述：**单包单体**，非 monorepo、无按 hook 拆分的子包。物理布局（根目录平铺 + factory/component/misc）与文档分类（Sensors/State/UI…）是两套体系：文档面向使用者按场景分类，源码面向贡献者按实现形态归类。`package.json` 中 `"sideEffects": false` + 同时产出 CJS（`main: lib/index.js`，`tsc` 直译）与 ESM（`module: esm/index.js`，`tsc -m esNext`）双构建，靠打包器 tree-shaking 实现"按需引入"，而非发多个子包。

## 5. 核心架构与关键实现

### 5.1 通用事件绑定与 SSR 防线（src/misc/util.ts + src/useEvent.ts）

`misc/util.ts` 提供 `on/off`（带空值保护的 addEventListener 包装）、`isBrowser/isNavigator` 常量，是全部传感器类 Hook 的地基。`useEvent.ts` 用 TypeScript 判别联合同时支持 `addEventListener` 风格与 `on/off` 风格两类事件目标：

```ts
export type UseEventTarget = ListenerType1 | ListenerType2;
const useEvent = <T extends UseEventTarget>(name, handler, target = defaultTarget, options?) => {
  useEffect(() => {
    if (isListenerType1(target)) { on(target, name, handler, options); }
    else if (isListenerType2(target)) { target.on(name, handler, options); }
    return () => { /* 对称解绑 */ };
  }, [name, handler, target, JSON.stringify(options)]);
};
```

以 `JSON.stringify(options)` 作依赖数组元素是对 options 对象的廉价深比较 hack（`src/useEvent.ts:60`），避免引用变化导致反复解绑——简单但非零成本。

状态类 Hook 采用组合复用而非继承：`useList`、`useGetSet` 等容器型 Hook 共用两块基石——`useUpdate`（用 useReducer 计数器强制重渲染，`src/useUpdate.ts`）与 `misc/hookState.ts` 的 `resolveHookState`（统一处理"传值 or 传函数"两种 setState 入参）；`useGetSet` 以 ref 持最新值、操作函数经 useMemo 收敛为稳定引用避免闭包陈旧读（`src/useGetSet.ts`）。`useFirstMountState`/`usePrevious`/`useLatest` 这类 5 行级 Hook 则作为其他 Hook 的内部构件被反复引用（如 useRafState 内嵌 useUnmount，`src/useRafState.ts`），形成自底向上的依赖塔；而 useMethods 未被任何其他 Hook 引用，属独立导出（tarball 全量 grep 验证）。

### 5.2 代表性 Hook 的边界处理

- **useLocalStorage**（`src/useLocalStorage.ts`）：三重防御——① 文件头 `if (!isBrowser) return [initialValue, noop, noop]` 做 SSR 短路（代价是 6 处 `eslint-disable react-hooks/rules-of-hooks`，条件调用 Hook 的反模式被显式豁免）；② 初始化与写入全部包 try/catch，注释明确"隐私模式/存储受限会抛错"；③ 支持自定义 serializer/deserializer 或 `raw` 直存，且 key 变化时用 `useLayoutEffect` 重新读取（第 56 行）。
- **useMedia**（`src/useMedia.ts`）：专门处理 SSR 水合——传了 `defaultState` 就不读 `window.matchMedia`，避免 hydration mismatch；未传时在开发环境 console.warn 提示；监听回调带 `mounted` 标志防止卸载后 setState（第 28-34 行）。
- **useScroll**（`src/useScroll.ts`）：基于 `useRafState`（`src/useRafState.ts`）把高频 scroll 事件合并到 requestAnimationFrame 帧内再 setState，且以 `{ passive: true }` 绑定监听器不阻塞滚动，卸载时 cancelAnimationFrame——高频事件 Hook 的标准性能范式。

### 5.3 工厂模式（src/factory/createReducer.ts）

`createReducer` 是最重的模块（约 100 行）：composeMiddleware 按 Redux 惯例右折叠组合中间件，返回定制的 reducer Hook，内部用 ref 持最新 state、dispatch 同步更新后再 setState。同目录 `createGlobalState`/`createStateContext` 解决跨组件状态共享，`createReducerContext` 把 reducer 接入 Context。这层"造 Hook 的 Hook"使库能覆盖超出单 Hook 表达力的场景。

## 6. 构建与运行

来自 `package.json` scripts：

```bash
npm i react-use          # 使用（React 17/18 由 peerDeps "*" 放行）
yarn install             # 开发
yarn test                # Jest jsdom 单测（--maxWorkers 2）
yarn test:ssr            # node 环境 SSR 测试（jest.config.node.ts）
yarn test:coverage       # 覆盖率
yarn lint / lint:types   # ESLint / tsc --noEmit
yarn build               # build:cjs(tsc→lib) + build:es(tsc -m esNext→esm)
yarn storybook           # 本地 demo 站（端口 6008）
yarn release             # semantic-release 自动发版
```

## 7. 测试与工程质量

- 规模：`tests/` 77 个文件、约 472 个 `it()` 用例，覆盖 111 个 Hook 中的 76 个左右（约 2/3，部分纯声明型 hook 如 useLogger 无独立测试文件——统计自 tests/ 文件数与 src 文件数对比）。
- 风格：`@testing-library/react-hooks` 的 `renderHook` 为主；亮点是**双环境测试**——`jest.config.ts`（jsdom）之外还有 `jest.config.node.ts`（node 环境），并有 `renderHookSSR`（来自 `@testing-library/react-hooks/server`）专门测水合前后行为（`tests/useMedia.test.ts` 用 mock matchMedia 验证 SSR/hydrate 两态）。
- CI（`.github/workflows/check-codebase.yml`）：build + lint + tsc + 单测矩阵（macos/ubuntu × node 20/22）；`release.yml` 在 master/next push 时跑全量检查后由 semantic-release 发版并生成 CHANGELOG。
- 质量观察：tsconfig 开 strict、noUnusedLocals/Parameters；但 `noImplicitAny: false` 放宽了类型（`tsconfig.json`）。部分 hook 依赖 eslint-disable 豁免规则（useLocalStorage 的 6 处 rules-of-hooks），是 SSR 设计带来的已知妥协。

## 8. 社区与维护状态

- 贡献者约 197 人（gh api contributors 分页统计），issue 677 开、PR 311 开（gh api，2026-11-06 查询）——PR 积压严重。
- 最近提交（gh api commits）：2026-06-10 合并 js-cookie 升级并发布 v17.6.1；2025-08~09 一串提交全部是 Renovate 机器人依赖升级；**最后一个功能性 feature 是 2024-12-09 的 v17.6.0**（给 useWindowSize 加 onChange 回调，见 CHANGELOG.md）。项目处于事实上的维护模式：只合并依赖升级，不做功能开发，社区 PR 基本不处理。
- 发版节奏：semantic-release 自动化，17.5.1→17.6.0 间隔约 5 个月、17.6.0→17.6.1 间隔约 1.5 年，且 17.6.1 仅是依赖修复。
- 未确认：官方是否声明放弃维护（无公告文件）；React 18/19 下的实际兼容性问题量（issue 区 677 个开放项中占比未逐一核实）。

## 9. 亮点与风险

**亮点**
1. API 覆盖面极广且粒度均匀：113 个导出、一 hook 一文件，查找成本极低（`src/index.ts`）。
2. 实现质量有代表性范式可抄：useRafState 的高频事件节流、useMedia 的 hydration 处理、useLocalStorage 的隐私模式 try/catch，都是社区常引用的写法（各源文件）。
3. 测试工程规范：双环境（jsdom/node）单测 + 472 用例 + CI 多 OS/Node 矩阵（`jest.config*.ts`、`.github/workflows/`）。
4. 文档-示例-源码三对齐：docs/ 117 页 + stories/ 118 个 story + CodeSandbox 链接（`README.md`、`docs/`）。
5. 发布工程成熟：semantic-release 自动 CHANGELOG/发版，Renovate 保依赖不过期（`release.yml`、`renovate.json`）。

**风险/局限**
1. 功能停滞：最后一个 feature 距今近两年，311 个 PR 开放未处理，选型时不能期待新 Hook 或 API 演进（gh api commits、CHANGELOG.md）。
2. React 17 时代包袱：测试栈锚定已弃用的 `@testing-library/react-hooks` v7，React 18 StrictMode 双调用等新语义未系统性验证（`package.json` devDependencies；兼容性结论未确认）。
3. useLocalStorage 等 Hook 的条件调用 Hook（SSR 短路）与 `JSON.stringify(options)` 依赖数组等写法属于对规则的有意违反，照抄需理解代价（`src/useLocalStorage.ts`、`src/useEvent.ts`）。
4. 单包发布：即使 `sideEffects:false` 可 tree-shaking，14 个运行时依赖仍会整体进入 lockfile，依赖审计面大于"每 hook 一包"的方案（`package.json`）。

## 10. 总结评价

react-use 是 2018-2021 年 React Hooks 生态爆发期最具影响力的 Hook 合集之一：广度、文档、测试、发布自动化都做到了当时的标杆水准，至今仍是阅读"怎么把一个浏览器 API 封装成优雅 Hook"的优质教材。但项目已进入只修依赖、不进功能的维护尾声，新项目若要 React 18+ 并发特性下的深度适配，ahooks 等仍在活跃迭代的库是更稳妥的选择；而在"按需抄写单个 Hook 实现"和"快速原型"场景下，react-use 依然完全可用。适合作为工具直接取用的普通业务开发者、想学习 Hook 封装范式的库作者读源码（重点：useRafState/useMedia/useLocalStorage/createReducer）。
