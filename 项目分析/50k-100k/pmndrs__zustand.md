# pmndrs/zustand 项目分析

> 星数 58864（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：React 生态最流行的极简状态管理库——"你的 store 就是一个 hook"，用约 100 行核心代码实现 flux 式状态容器，靠中间件与 TypeScript 类型系统扩展能力。

| 项 | 值 |
|---|---|
| 名称 | zustand（pmndrs/zustand） |
| 版本 | v5.0.16（package.json `version`） |
| 星数 | 58894（gh api，2026-11-03 实测） |
| 语言 | TypeScript（全量源码约 1500 行） |
| 许可证 | MIT（package.json、LICENSE） |
| 作者 | Paul Henschel（pmndrs 组织），核心贡献者 Daishi Kato、Jeremy Holcomb |
| 最近活跃 | 最近 push 2026-10-22，同日发布 v5.0.16（gh api `pushed_at`/releases） |
| 创建时间 | 2019-04-09（gh api `created_at`） |

## 2. 项目是什么，解决什么问题

Zustand 解决的是 React 状态管理的"重"问题。README（仓库根目录）开篇自述："A small, fast and scalable bearbones state-management solution using simplified flux principles"，即一个基于简化 flux 原则、以 hooks 为核心 API、无样板代码、无强 opinions 的状态管理方案。它刻意不做 Redux 那样的 action/reducer/dispatch 全套仪式，也不像 Context 那样一变全树重渲染——组件通过 selector 订阅 store 的一个切片，只有切片变化才重渲染。

README 强调它处理了 React 状态管理中几个著名的坑：zombie child problem（子组件读到已过期的 props/state）、react concurrent rendering 兼容、以及混合渲染器之间的 context loss，并称"可能是 React 生态里把这些问题都处理对的状态管理器"。这些处理直接体现在实现选型上：React 绑定层不是手写订阅，而是直接使用 React 官方的 `useSyncExternalStore`（src/react.ts 第 30 行），并发安全与 tearing 问题交由 React 官方原语保证。

典型用法极简（README 示例）：`create((set) => ({ bears: 0, increase: () => set((s) => ({ bears: s.bears + 1 })) }))` 创建一个 hook，组件里 `useBearStore((s) => s.bears)` 订阅切片，无需 Provider 包裹。状态更新要求不可变、`set` 默认做浅合并。

适用范围不止 React：核心 `vanilla.ts` 与 React 解耦，可在任何 JS 环境使用；package.json 的 exports 条件里有 `react-native` 入口；peerDependencies 里 react 只是可选依赖（package.json `peerDependenciesMeta`），middleware 还提供 `ssrSafe`（src/middleware/ssrSafe.ts）应对 SSR 场景。

## 3. 技术栈与依赖

- 语言：TypeScript（严格类型驱动，大量类型体操支撑中间件类型推导，见 src/vanilla.ts 的 `Mutate`/`StateCreator`）
- 运行时依赖：零（package.json 无 `dependencies` 字段，仅 peerDependencies）
- peerDependencies：react >=18、use-sync-external-store >=1.2、immer >=9（三者均可选，package.json）
- 构建：Rollup 4 + rollup-plugin-esbuild + @rollup/plugin-typescript，多配置分别构建 base/vanilla/react/middleware/shallow/traditional 等入口（package.json `build:*` 脚本、rollup.config.mjs）
- 测试：Vitest 5 + jsdom + React Testing Library + @vitest/coverage-v8（package.json devDependencies、vitest.config.mts）
- 代码质量：ESLint 9（flat config，eslint.config.mjs）+ Prettier（无分号、单引号）
- 包管理：pnpm 11（packageManager 字段），工作区含 examples/demo、examples/starter（pnpm-workspace.yaml）
- 发布产物：双格式 CJS + ESM，ESM 的 .d.ts 重命名为 .d.mts 并补 `.mjs` 后缀引用（package.json `patch-esm-ts` 脚本），typesVersions 兼容 TS <4.5（给占位报错文件，package.json）

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| src/vanilla.ts | 核心 createStore：与框架无关的状态容器（100 行） |
| src/vanilla/shallow.ts | `shallow` 浅比较工具（vanilla 侧） |
| src/react.ts | `create`/`useStore`：React 绑定层（64 行） |
| src/react/shallow.ts | `useShallow` hook |
| src/traditional.ts | `createWithEqualityFn`：基于 use-sync-external-store with-selector 的自定义相等函数版本 |
| src/middleware.ts | 中间件聚合入口（`zustand/middleware`） |
| src/middleware/ | devtools(451 行)/persist(403)/immer(87)/subscribeWithSelector(73)/redux(50)/ssrSafe(25)/combine(15) |
| src/index.ts | 主入口，re-export vanilla + react + 各中间件 |
| tests/ | 13 个测试文件共约 7800 行（vitest） |
| docs/ | 官网文档源码：learn/getting-started、learn/guides（18+ 篇）、reference/apis/hooks/middlewares/migrations |
| examples/ | demo 与 starter 两个工作区示例 |
| .github/workflows/ | 8 个 CI workflow |
| rollup.config.mjs | 多入口构建配置 |

组织方式评述：这是典型的"单包多入口"小型核心库，不是 monorepo 多包结构。src 下每个公开入口（zustand、zustand/vanilla、zustand/middleware、zustand/shallow、zustand/traditional）对应一个独立文件，rollup 按 entry 分别打包，package.json 的 wildcard exports `./*` 自动映射。核心与中间件严格分层：vanilla 不 import react，react.ts 不 import 中间件，中间件通过包装 `StateCreator` 与 patch `api` 方法叠加。

## 5. 核心架构与关键实现

### 5.1 createStore：闭包 + Set 订阅的最小实现（src/vanilla.ts）

整个库的地基只有 100 行：一个闭包持有 `state` 与 `listeners: Set<Listener>`，暴露 `setState/getState/getInitialState/subscribe` 四个方法。

```ts
const setState: StoreApi<TState>['setState'] = (partial, replace) => {
  const nextState =
    typeof partial === 'function'
      ? (partial as (state: TState) => TState)(state)
      : partial
  if (!Object.is(nextState, state)) {
    const previousState = state
    state =
      (replace ?? (typeof nextState !== 'object' || nextState === null))
        ? (nextState as TState)
        : Object.assign({}, state, nextState)
    listeners.forEach((listener) => listener(state, previousState))
  }
}
```

要点：`Object.is` 判等跳过无效更新；默认浅合并（`Object.assign({}, state, nextState)`），传 `replace: true` 则整体替换；通知是同步遍历 listener，携带 `(state, prevState)` 双参。`subscribe` 返回解绑函数。类型侧定义了 `StoreApi`、`StateCreator`、以及中间件类型体系的根基 `StoreMutators`/`Mutate`——中间件通过 TypeScript module augmentation 向 `StoreMutators` 接口注册键名（如 `'zustand/subscribeWithSelector'`），`Mutate` 递归展开元组实现类型叠加（src/vanilla.ts 第 20-41 行）。

### 5.2 React 绑定：直接站在 useSyncExternalStore 上（src/react.ts）

`useStore` 一行核心调用（src/react.ts 第 30-34 行）：

```ts
const slice = React.useSyncExternalStore(
  api.subscribe,
  React.useCallback(() => selector(api.getState()), [api, selector]),
  () => selector(api.getInitialState()),
)
```

三个参数分别对应：订阅、getSnapshot（用 useCallback 缓存避免无限重渲染）、getServerSnapshot（SSR 时返回初始状态，这是 ssr.test.tsx 能测 SSR 的原因）。`create` 的实现是把四个 store API 直接 `Object.assign` 到 hook 函数上（src/react.ts 第 53-61 行），所以 `useBearStore.setState(...)` 这样的"hook 即 store"用法才成立。`create()` 不传参时返回柯里化工厂（配合中间件 `devtools(...)(...)` 的写法）。src/traditional.ts 提供需要自定义相等函数的版本，借助 use-sync-external-store 的 with-selector shim 实现。

### 5.3 中间件：统一 "包装 creator + patch api" 模式

所有中间件签名都是 `(config) => (set, get, api) => state`，通过改写 `api` 上的方法注入行为。三个代表：

- subscribeWithSelector（src/middleware/subscribeWithSelector.ts）：重写 `api.subscribe`，若带 selector 则包装 listener——每次状态变化重算 `selector(state)`，用 `equalityFn`（默认 `Object.is`）比较切片，不等才触发，并支持 `fireImmediately`。这把订阅粒度从"整个 state"降到"选中切片"。
- persist（src/middleware/persist.ts，403 行）：最重的中间件。patch `api.setState` 与传入的 `set` 实现"写穿透"（每次更新后 `setItem` 持久化 `partialize` 后的状态，第 233-247 行附近）；启动时 `hydrate()` 从 storage 回放状态，内部用 `hydrationVersion` 计数器防止并发 `rehydrate()` 的竞态（第 190 行附近注释）；`createJSONStorage` 包装 localStorage 等同步/异步 storage（第 22-59 行）；自研 `toThenable` 兼容同步与异步 storage 的链式调用（第 150-180 行附近）。
- immer（src/middleware/immer.ts）：patch `api.setState`，函数式 updater 一律走 immer 的 `produce`，让用户可以直接写"可变"更新。
- 其余：devtools（451 行，对接 Redux DevTools 扩展，把每次 setState 转成 dispatch 动作并带 action 名与堆栈信息，src/middleware/devtools.ts）；redux（50 行，注入 `dispatch` 提供 Redux 风格 reducer API）；combine（15 行，`Object.assign` 合并初始状态与 actions）；ssrSafe（25 行，SSR 安全实验性 API，index.ts 中以 `unstable_ssrSafe` 导出）。

### 5.4 shallow 与 useShallow（src/vanilla/shallow.ts、src/react/shallow.ts）

`shallow` 是逐 key 的浅比较（vanilla/shallow.ts 74 行，含对 Map/Set 等的处理）；`useShallow` 用 `useRef` 缓存上次结果，selector 输出浅相等时返回旧引用，从而避免"每次 render 生成新对象"导致的重渲染（src/react/shallow.ts 全文 12 行）。

## 6. 构建与运行

来自 package.json scripts（pnpm）：

```bash
pnpm install          # 安装依赖（node >= 12.20）
pnpm run test         # = test:format + test:types + test:lint + test:spec
pnpm run test:spec    # vitest run（jsdom 环境，coverage 用 v8 provider）
pnpm run build        # rollup 多入口构建：build:base/vanilla/react/middleware/... 
                      #   + postbuild（patch d.ts、拷贝 README/LICENSE 到 dist、生成 esm）
pnpm run fix          # eslint --fix + prettier --write
```

文档站与 demo 在 docs/、examples/ 下（pnpm 工作区成员，pnpm-workspace.yaml）。本次分析仅静态阅读，未实际运行构建（符合任务约束）。

## 7. 测试与工程质量

- 测试规模：tests/ 下 13 个文件共约 7815 行（wc -l 实测）。基本盘 basic.test.tsx 744 行、persistAsync 1244 行 + persistSync 760 行、devtools 2926 行、类型测试 middlewareTypes.test.tsx 859 行 + types.test.tsx 252 行；vanilla/ 下另有独立于 React 的纯容器测试。
- 风格：单元 + 集成混合，普遍用 @testing-library/react 渲染真实组件断言渲染次数与输出；专门的 SSR 测试（ssr.test.tsx 113 行）验证 getServerSnapshot 路径；类型层测试独立成文件，用 tsc 断言类型错误/正确性。
- CI：.github/workflows/ 下 8 个 workflow——test.yml（push/PR 跑 format、types、lint、spec、build 五道关卡，实测读取）；test-old-typescript.yml（老版本 TS 兼容）；test-multiple-versions.yml / test-multiple-builds.yml（多 React 版本/多构建矩阵）；compressed-size.yml（产物体积监控，与 README 的 bundle size 徽章呼应）；publish.yml、preview-release.yml（发布）、docs.yml（文档部署）。所有 action 均按 commit SHA 锁定版本。
- 工程质量观察：零运行时依赖、无分号统一风格、d.ts 发布链路有专门的 patch 脚本保证 CJS/ESM 双类型正确。测试对中间件的边界情况（persist 竞态、异步 storage、SSR 水合）覆盖相当细致，属于小型库中罕见的质量投入。

## 8. 社区与维护状态

- 活跃度：gh api 实测 `pushed_at` 为 2026-10-22，与最新 release v5.0.16 同日发布，截至分析日仅 5 个 open issue——维护非常活跃、issue 清理及时。
- Release 节奏：v5.0.14（2026-05-28）→ v5.0.15（2026-08-13）→ v5.0.16（2026-10-22），稳定在 2-3 个月一个补丁版本的节奏（gh api releases）。
- 贡献者结构：contributors 分页头显示共 343 页（per_page=1，即约 343 位贡献者，gh api 实测）；package.json 中正式列名贡献者为 Jeremy Holcomb 与 Daishi Kato（dai-shi，长期核心维护者，也是 docs/ 中多个实践指南的推动者）。
- 社区渠道：README 提供 Discord（pmndres）与赞助横幅；项目隶属 pmndrs（poimandres）组织，同门有 react-three-fiber 等。
- 老化风险低：仓库创建于 2019-04，主版本已到 v5，docs/reference/migrations 保留了 v4→v5 迁移文档。

## 9. 亮点与风险

亮点：

1. 极致的核心克制：vanilla 核心 100 行（src/vanilla.ts）实现完整订阅-更新语义，零运行时依赖，产物体积小（README 徽章实时展示 bundle size，CI 有 compressed-size.yml 盯守）。
2. 正确性靠官方原语兜底：React 绑定直接用 `useSyncExternalStore`（src/react.ts），并发渲染、tearing、SSR 水合问题交由 React 保证，而不是自己造轮子。
3. 中间件类型体系设计精巧：借 module augmentation + `Mutate` 递归类型（src/vanilla.ts 第 20-41 行），中间件叠加后 `setState` 等方法签名能精确变化，7.8k 行测试中有 1100+ 行专门测类型。
4. 中间件覆盖真实痛点：persist 处理了异步 storage、水合竞态、partialize/merge 定制（src/middleware/persist.ts）；subscribeWithSelector、immer、devtools、redux 各司其职且可组合。
5. 工程与文档双强：8 个 CI workflow、多 TS/React 版本矩阵、343 位贡献者的社区规模，docs/ 下含 getting-started、18+ 篇实践指南与完整 API 参考。

风险/局限：

1. 心智模型陷阱：`set` 默认浅合并、selector 返回新对象会破坏引用相等导致多余重渲染，需要配 `useShallow`——对新手有学习曲线（docs/learn/guides/prevent-rerenders-with-use-shallow.md 专门讲此事）。
2. 无结构约束：没有 action 命名、devtool 时间旅行等强制规范，大型团队用起来依赖自觉（redux 中间件只是可选补丁）；原生 Redux DevTools 集成是"桥接"而非一等公民（src/middleware/devtools.ts）。
3. 实验性 API 不稳定：`unstable_ssrSafe` 等以 unstable 前缀导出（src/index.ts），SSR/多渲染器场景部分能力仍在演进。
4. 生态绑定：深度绑定 React ≥18 的 useSyncExternalStore 语义，非 React 场景只能用 vanilla 层自行接线。

## 10. 总结评价

Zustand 是"少即是多"的典范：用不到 1500 行源码（wc -l 实测 1468 行）支撑起近 5.9 万星、每周数百万下载量级的 React 状态管理事实标准之一。它的核心竞争力是三层叠加——极简的 vanilla 容器、站在 React 官方 useSyncExternalStore 之上的正确绑定、以及类型安全可组合的中间件体系；再加上 pmndrs 社区的持续维护与高质量文档/测试，使其同时服务新手（三行代码上手）与重度用户（persist/devtools/自定义相等）。适合所有规模的 React 项目作为默认状态方案，也非常值得任何库作者阅读源码——尤其是它"用 100 行闭包 + module augmentation 类型体操构建可扩展体系"的做法，是小型核心库工程化的教科书样本。
