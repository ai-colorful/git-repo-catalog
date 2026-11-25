# solidjs/solid 项目分析

> 星数 39,500（任务快照 2026-10-24；gh api 2026-11-17 实测 36,273）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-17

## 1. 项目速览

一句话定位：SolidJS 是一个无 Virtual DOM 的声明式 UI 框架——JSX 在编译期转为真实 DOM 操作，运行时靠细粒度响应式系统（signal/computation）精准更新，组件函数只运行一次。

| 项 | 值 |
| --- | --- |
| 名称 | solidjs/solid（核心包 `solid-js`，当前 main 分支版本 1.9.19，`packages/solid/package.json`） |
| 星数 | 39,500（快照）；gh api 2026-11-17 核实 36,273 |
| 语言 | TypeScript（gh api 核实：language=TypeScript） |
| 许可证 | MIT（`LICENSE`，Copyright (c) 2016-2025 Ryan Carniato） |
| 作者 | Ryan Carniato（`package.json` author 字段） |
| 创建时间 | 2018-04-24（gh api `created_at`） |
| 最近 push | 2026-11-16（gh api `pushed_at`，活跃） |
| 最近发版 | 1.x 线为 solid-js 1.9.19（`packages/solid/CHANGELOG.md`）；2.0 线已发布 rc.23 系列（gh api releases，2026-11-16 发布 `solid-js@2.0.0-rc.23`） |
| 贡献者 | 约 208 人（gh api contributors 分页 Link header 末页 208） |

## 2. 项目是什么，解决什么问题

Solid 是"用响应式原语写 UI"的框架：状态用 `createSignal` 创建，派生状态就是一个普通函数，任何一块状态变化时只有依赖它的代码重跑（`README.md`："Instead of using a Virtual DOM, it compiles its templates to real DOM nodes and updates them with fine-grained reactions"）。README 的 Counter 示例明确标注组件函数体只执行一次，`{doubleCount()}` 文本节点是唯一会重渲染的部分。

它解决的是传统 VDOM 框架的两类开销：
1. **VDOM diff 开销**。React 每次状态变化重跑组件函数、生成新虚拟树再 diff；Solid 在编译期把 JSX 模板提取成静态 HTML 模板字符串，运行时只需 `cloneNode` 再对动态点位做精确绑定（`README.md` 编译产物示例：`_$template`、`_$insert`、`_$delegateEvents`），更新路径没有 diff。
2. **组件级更新粒度**。Vue3 虽有 proxy 响应式，但组件层仍是 VDOM patch；Solid 的更新粒度直达单个 DOM 操作，与组件边界无关。

JSX 的选择是有意的：不引入模板 DSL，保留 TypeScript/工具链兼容性，但把 JSX 的语义从"每次执行的描述"编译成"一次性结构 + 响应式点位"（`README.md` 与 `CONTRIBUTING.md` 第 187 行附近对生态的描述）。核心响应式思想承自 Adam Haile 的 S.js——`src/reactive/signal.ts` 与 `src/reactive/array.ts` 文件头均带 S.js 的 MIT 版权声明。

生态位：核心库之外有 SolidStart 元框架（`CONTRIBUTING.md` 第 66 行，类比 SvelteKit/Next）、solid-primitives、Kobalte 组件库等（README 生态段落）。

## 3. 技术栈与依赖

- 语言：TypeScript ~5.7（根 `package.json` devDependencies），Node 24（`.nvmrc`=24）。
- 构建：Rollup 4 + @rollup/plugin-babel（`packages/solid/rollup.config.js`），pnpm workspace + Turborepo（`pnpm-workspace.yaml`、`turbo.json`）。
- 发布管理：Changesets（`.changeset/`、根 `package.json` 的 `bump`/`release:only` 脚本）。
- 测试：Vitest 4 + jsdom（根 devDependencies；`packages/solid/vitest.server.config.mjs`）。
- 关键运行时依赖（编译期配套，均为作者生态内项目）：
  - `babel-plugin-jsx-dom-expressions` ^0.40.14 / `dom-expressions` 0.40.14：JSX 编译插件与 DOM 运行时（template/insert 等），独立仓库，`packages/solid/package.json` devDependencies。
  - `seroval` ~1.6.11：SSR 流式序列化器（Promise/循环引用可序列化），根 devDependencies。
- 无运行时第三方依赖：`packages/solid/package.json` 的 dependencies 为空（导出全部自研），仅 devDependencies 有构建配套。

## 4. 目录结构与代码组织

pnpm monorepo（`pnpm-workspace.yaml`：`packages/*`），Turborepo 编排 build/types/test/coverage 任务（`turbo.json`）。

| 目录 | 职责 |
| --- | --- |
| `packages/solid/` | 核心包 `solid-js`（版本 1.9.19） |
| `packages/solid/src/reactive/` | 响应式内核：signal.ts（1903 行）、scheduler.ts、array.ts、observable.ts |
| `packages/solid/src/render/` | 客户端渲染层：component.ts、flow.ts（控制流）、hydration.ts、Suspense.ts |
| `packages/solid/src/server/` | SSR 独立实现：reactive.ts（394 行）+ rendering.ts（745 行），无副作用的服务端响应式模型 |
| `packages/solid/web/` | `solid-js/web` 出口；web/src/core.ts 是注入 dom-expressions 的 rxcore 胶水；web/server/server.ts 仅一行 `export * from "dom-expressions/src/server"`（流式 SSR 运行时在外部仓库） |
| `packages/solid/store/` | `solid-js/store`：Proxy 代理 store（store.ts 555 行）、reconcile/produce 修饰器（modifiers.ts）、createMutable（mutable.ts） |
| `packages/solid/h/`、`html/`、`universal/` | 替代 JSX 的 hyperscript（h）、tagged-template HTML（html）、跨平台渲染出口（universal） |
| `packages/solid/test/`、`web/test/`、`store/test/` | 单元/类型测试 |
| `packages/babel-preset-solid/` | Babel 预设，仅 34 行，包一层 `babel-plugin-jsx-dom-expressions`（`index.js`） |
| `packages/solid-element/` | Web Components 适配（custom element 包装） |
| `packages/solid-ssr/` | SSG 工具 `renderStatic`（`README.md`：静态生成器，entry/output/url 三元组） |
| `packages/test-integration/` | 集成测试：下载真实下游仓库（gitly）装包跑测试（`tests/downloaded.spec.ts`） |
| `packages/solid/bench/` | 性能基准（bench.cjs、libraries/、results.md） |

组织方式评述：核心包内部按"响应式内核 / 渲染层 / 平台出口"三层切分，同一份响应式内核配 server/ 与 web/ 两套出口（`package.json` exports 按 browser/node/worker/deno/development 条件导出多个 dist 产物）。编译器与 DOM 运行时拆到外部仓库（dom-expressions），本仓库保持精简（全仓 TS 源码约 1.76 万行）。

## 5. 核心架构与关键实现

### 5.1 细粒度响应式内核（`packages/solid/src/reactive/signal.ts`，1903 行）

两个核心数据结构：

```ts
export interface SignalState<T> extends SourceMapValue {
  value: T;
  observers: Computation<any>[] | null;   // 订阅了此 signal 的 computation
  observerSlots: number[] | null;         // 每个 observer 的反向 slot 索引
  comparator?: (prev: T, next: T) => boolean;
}
export interface Computation<Init, Next extends Init = Init> extends Owner {
  fn: EffectFunction<Init, Next>;
  state: ComputationState;      // 0 干净 / 1 STALE / 2 PENDING
  sources: SignalState<Next>[] | null;   // 它依赖的 signal
  sourceSlots: number[] | null;
  pure: boolean;
}
```

依赖追踪是运行时双向登记：`readSignal` 在有 `Listener`（当前正在执行的 computation）时把 Listener 压入 `observers`/`observerSlots`，同时把自己记进 Listener 的 `sources`/`sourceSlots`（signal.ts `readSignal`，1328-1378 行）。写入端 `writeSignal`（1379 行起）做推送标记：值变更后遍历 observers，把未标记的 observer 置为 `STALE`，`pure` 的（memo/computed）进 `Updates` 队列，非 pure 的（user effect）进 `Effects` 队列，并向下级联 `markDownstream`；超过 10e5 次更新直接抛"Potential Infinite Loop"。

执行调度是"标记推送 + 求值拉取"混合：`runTop`（1561 行）从触发节点沿 `owner` 链回溯祖先，从最顶层按序 `updateComputation`，保证 memo 依赖顺序正确；`completeUpdates` 串行清空 Updates 再执行 Effects。这套 pull/push 混合是 S.js 谱系的经典设计，与 Vue3 的"proxy 拦截触发副作用队列"不同：Vue 的最小调度单位仍是组件渲染函数（组件内再做 patch），Solid 的最小调度单位是单个 computation（对应一段具体 DOM 更新或一个 memo）。

其他内核能力：`Transition`（signal.ts 117-133 行，tValue/tState 双份状态实现并发过渡，Suspense 联动）；`createResource`（587 行起，异步数据）、`batch`/`untrack`/`on`/`createSelector`/`createDeferred`。`reactive/scheduler.ts`（177 行）是从 React Scheduler 移植的 MessageChannel 时间切片器（注释注明"Basic port modification of Reacts Scheduler"，yieldInterval=5ms、isInputPending 支持），供 transition 的 `requestCallback` 使用。

### 5.2 编译期渲染：为何无 VDOM 仍高效

`packages/babel-preset-solid/index.js` 全部 34 行只是转发到外部插件 `babel-plugin-jsx-dom-expressions`，配置 `generate: "dom"` 并列出 10 个内置控制流组件（For/Show/Switch/Match/Suspense/Portal/Index/Dynamic/ErrorBoundary）。编译器做的事（从 `README.md` 的编译产物示例可证）：静态 HTML 抽成 `_$template` 模板字符串，运行时每个实例 `cloneNode` 得到真实 DOM；动态表达式替换为 `_$insert`/`setProp` 等细粒度绑定函数；事件统一 `delegateEvents`。因此运行时既没有 createElement 树，也没有 reconcile diff——"高效"来自把 diff 在编译期消掉了。

运行时 DOM 原语（template/insert/delegateEvents）在外部 `dom-expressions` 仓库；`packages/solid/rollup.config.js` 用 `babel-plugin-transform-rename-import` 把其中的 `rxcore` 导入替换为本仓库 `web/src/core.ts`，后者用 solid 的 `createMemo` 等原语接住响应式——这就是编译产物与响应式内核的接缝。

### 5.3 控制流组件与"组件只跑一次"（`packages/solid/src/render/`）

`createComponent`（`render/component.ts` 97 行）的实现只有一层 `untrack(() => Comp(props))`：组件函数不在任何响应式上下文中执行，所以天然只运行一次，props 通过 `mergeProps` 的 getter 惰性求值传递。`render/flow.ts`（297 行）中 `<For>` 就是 `createMemo(mapArray(() => props.each, props.children))`（34-45 行）；`mapArray`（`reactive/array.ts`，改良自 S-array）对列表做键控复用——只对变化的 item 重算映射，每个 item 有独立 root 与 disposers，`Index` 则提供按位置更新的索引 accessor。Show/Switch/ErrorBoundary 同样以 memo + 纯函数实现，全部在 flow.ts 一个文件内。

### 5.4 SSR 流式水合

服务端有一套独立的响应式模型：`src/server/reactive.ts`（394 行）实现无 effect 调度的 server 版 signal/memo，`src/server/rendering.ts`（745 行）实现 `ssr` 模板原语、sharedConfig、错误序列化（355 行 `ctx.serialize(id, error)`）与资源去重（542-546 行，`ctx.serialize(id, p, options.deferStream)`，基于 seroval 把 Promise 序列化进 HTML 流——1.8.0 CHANGELOG 的"De-duping Streaming Serialization"正是该机制，跨 Suspense 边界去重）。水合协议走 `render/hydration.ts` 的 `HydrationContext {id, count}`：服务端按树形路径生成 id（`getContextId` 把计数编码为 `id+字母+数字`，hydration.ts 32-34 行），客户端 `registry: Map<string, Element>` 按相同 id 路径认领已有 DOM 节点，`createComponent` 在水合时切换 context（component.ts 97-112 行）。流式传输（renderToStream/pipeToNodeWritable）本体在外部 dom-expressions/src/server（`web/server/server.ts` 一行 re-export），`packages/solid-ssr` 提供静态生成（SSG）CLI。

## 6. 构建与运行

来自根 `package.json` scripts 与 `CONTRIBUTING.md`（第 91-99 行）：

```bash
pnpm install            # preinstall 强制 pnpm（only-allow）
pnpm build              # turbo run build（rollup 产物 dist/ + tsconfig-replace-paths 生成 types/）
pnpm test               # turbo run test test-types（Vitest 单测 + TS 类型测试）
pnpm coverage           # turbo run coverage（@vitest/coverage-v8 → Coveralls）
pnpm format             # prettier
```

发布：`pnpm bump`（changeset add）→ `pnpm publish`（build + types + changeset publish）。单包测试可用 vitest 直接跑 `packages/solid`（`packages/solid/package.json` 与 `vitest.server.config.mjs`）。

## 7. 测试与工程质量

- 规模：全仓 35 个 spec 文件（Vitest），核心三块——`packages/solid/test/`（signals 1005 行、signals.memo 515 行、resource 442 行、component 516 行等）、`packages/solid/web/test/`（DOM/控制流：for 444 行、show 375 行、switch 419 行、suspense 440 行、transition 专项等 14 个）、`packages/solid/store/test/`（store 1269 行、modifiers 530 行、mutable 406 行）。
- 类型测试是一等公民：`test/signals.type-tests.ts`（1129 行）、`component.type-tests.ts` 等 `.type-tests.ts` 文件由 `turbo run test-types` 执行 expect-type 式校验（根 `package.json` test 脚本）。
- 服务端行为单独覆盖：`test/server/` 下 8 个 spec（suspense-nested、reconcile-array-length、null-prototype-store 等）。
- 集成测试：`packages/test-integration/tests/downloaded.spec.ts` 用 gitly 下载真实下游仓库、安装打包产物跑构建，验证发布物可用性。
- CI：`.github/workflows/main-ci.yml`——PR/主分支推送时 pnpm install→build→test→coverage→Coveralls；另有 `cve-lite.yml`。代码质量观察：内核带防御性注释（如 readSignal 中"Build both arrays before assigning either"处理分配失败一致性，1328 行附近）、保留 transition OOM 回归测试（`web/test/transition-oom-4728.spec.tsx`）。

## 8. 社区与维护状态

- 活跃度：gh api `pushed_at` 2026-11-16（下载当日仍在推送）；open issues 仅 15 个，维护响应快（gh api）。
- 发版节奏：1.x 线持续发补丁（`packages/solid/CHANGELOG.md` 顶部 1.9.19，含 2025 年的 store 内存泄漏修复、DOM Expressions 0.40.14 升级条目）；同时 2.0 线处于 RC 冲刺——gh api releases 显示 2026-11-16 一天发布了 `@solidjs/signals`、`@solidjs/web`、`@solidjs/html` 等 8 个包的 `2.0.0-rc.23`，tag 线从 v2.0.0-experimental.0 一路走到 rc.23（gh api tags）。2.0 将响应式内核拆为独立包 `@solidjs/signals`（`CHANGELOG.md` 1.8.0 条目亦提及与 solidjs/signals 仓库的过渡策略）。
- 贡献者结构：约 208 名贡献者（gh api contributors 分页）；README 列有 Open Collective 赞助与 Discord 社区；作者 Ryan Carniato 深度主导，核心协作者如 lxsmnsyc（seroval/编译器）、intrnl 等在 CHANGELOG 中被点名。
- 修复质量信号：1.9.19 条目包含对 Astro（issue #8365 关联）等下游生态问题的回溯修复（`packages/solid/CHANGELOG.md`、`CHANGELOG.md` 1.8.0 "Hydration Improvements"）。

## 9. 亮点与风险

亮点：
1. 细粒度响应式内核完整、自洽且极小：signal.ts 一个文件容纳 signal/memo/effect/resource/transition 全部语义（1903 行），依赖追踪双向登记 + push/pull 混合调度，教科书级实现（`src/reactive/signal.ts`）。
2. 编译期/运行时分工清晰：本仓库只管内核与渲染语义，DOM 代码生成下沉到 dom-expressions 仓库，编译产物直接是模板克隆 + 点位绑定，无 diff（`babel-preset-solid/index.js`、`rollup.config.js` rxcore 注入、`README.md` 编译示例）。
3. 工程质量高：类型测试 1129 行级别、集成测试用真实下游仓库验证发布物、防御性修复带注释与回归用例（`test/signals.type-tests.ts`、`test-integration/`、`web/test/transition-oom-4728.spec.tsx`）。
4. 多端出口完备：同一内核配 browser/node/worker/deno/development 条件导出，SSR 有独立无副作用实现与流式序列化（`packages/solid/package.json` exports、`src/server/`）。

风险/局限：
1. 双版本线过渡期的心智成本：main 是 1.9.x，2.0 拆包（`@solidjs/signals` 等）在 RC，API/包结构将变，选型需评估迁移（gh api releases、`CHANGELOG.md` 1.8.0 说明）。
2. 高度依赖作者个人主导与外部配套仓库（dom-expressions、babel-plugin、seroval），版本必须与编译插件严格配套（`CHANGELOG.md` 1.8.0 明确"your version of Solid and the Babel Plugin should be the same"）。
3. 生态规模小于 React/Vue：SSR 流式水合、transition 等高级路径的边界 bug 相对多（1.8/1.9 CHANGELOG 多条水合修复可证）。
4. 本仓库 tarball 未能统计 git 提交节奏（HEAD 快照无 git 历史，mtimes 同为解压时间），主分支提交频率未确认；星数快照（39.5k）与 gh api 实测（36.3k）存在 3k 级差异，以 GitHub API 为准。

## 10. 总结评价

Solid 是"细粒度响应式 + 编译期优化"路线的代表作：用 S.js 谱系的 signal/computation 内核把更新粒度压到 DOM 操作级，再用 Babel 插件把 JSX 编译成模板克隆，从而在保持 JSX/TS 开发体验的同时绕开了 VDOM diff。代码库是学习响应式系统设计（依赖追踪、拓扑调度、transition 并发、SSR 水合协议）的最佳范本之一，规模适中（内核单文件千余行）且注释与测试质量高。适合：想深入理解 signal 谱系响应式的工程师读源码；追求接近原生 DOM 性能、偏好"组件只跑一次"心智模型的前端团队选型；但需接受 2.0 大版本迁移在即与相对较小的生态。分析基于 HEAD 快照（solid-js 1.9.19）与 gh api 元数据（2026-11-17）。
