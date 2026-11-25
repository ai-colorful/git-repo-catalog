# vuejs/core 项目分析

> 星数 54790（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：Vue.js 3 的核心源码仓库——包含响应式系统、模板编译器与运行时渲染器的渐进式 JavaScript 框架内核。

| 项目 | 内容 |
| --- | --- |
| 名称 | vuejs/core（vuejs/vue 的 Vue 3 后继仓库，Vue 2 已归档） |
| 星数 | 54790（快照 2026-10-24） |
| 主语言 | TypeScript（严格模式，`tsconfig.json`） |
| 许可证 | MIT（`LICENSE`，Copyright 2018-present Yuxi (Evan) You and Vue contributors） |
| 包管理 | pnpm 12 monorepo（`pnpm-workspace.yaml`，`package.json` 中 `pnpm@12.11.2`） |
| 当前版本 | main 分支 3.5.44（`package.json`）；3.6.0-rc.14 于 2026-10-30 从 `minor` 分支发布 |
| 最近活跃 | 最近 push 2026-11-02（GitHub API），分析前一天仍有提交，极活跃 |
| 规模 | HEAD 共 703 个文件（GitHub API 树核对）；非测试 TS 源码约 5.78 万行，测试约 9.5 万行（`wc -l` 统计） |

## 2. 项目是什么，解决什么问题

Vue 3 是一个用于构建用户界面的渐进式框架，本仓库是它的"内核"：不包含路由（vue-router）、状态管理（Pinia）等周边，只负责三件事——把响应式数据变化追踪出来、把模板编译成高效渲染代码、把虚拟 DOM 渲染/更新到真实 DOM（`README.md` 将用户导向 vuejs.org 文档，仓库本身只有 Getting Started 一句话）。

第一件事由 reactivity 包解决：基于 ES Proxy 的依赖追踪系统。组件渲染函数或 `watch`/`computed` 在运行中读取响应式属性时，依赖被记录为"属性(Dep)—订阅者(Effect)"的关联；属性被写入时通知订阅者重新执行。这一层与 DOM 完全解耦，可以单独作为 `@vue/reactivity` 使用（`packages/reactivity/package.json` 独立发布了全部构建格式）。

第二件事由 compiler-* 三段编译器解决：模板字符串 → parser 生成 AST（`packages/compiler-core/src/parser.ts`）→ transform 对 AST 做标记与优化（`packages/compiler-core/src/transforms/`）→ codegen 生成渲染函数代码（`packages/compiler-core/src/codegen.ts`）。编译期知道模板结构，因此能生成带 PatchFlags 的静态优化提示，让运行时跳过大部分无关节点的 diff——这是 Vue 3 相对 Vue 2 的核心性能来源。`compiler-sfc` 在此之上处理单文件组件（SFC）的 `<script setup>`、`<style scoped>`、`<template>` 拼装。

第三件事由 runtime-core/runtime-dom 解决：`renderer.ts` 实现平台无关的虚拟 DOM patch 算法，`runtime-dom` 提供浏览器 DOM 的 nodeOps/patchProp 适配层，二者通过 `createAppAPI(render, hydrate)`（`packages/runtime-core/src/apiCreateApp.ts:253`）组装成 `createApp`。这种"自定义渲染器"接口也是 runtime-test（测试用内存 DOM）与各类三方可视化渲染器的基础。

整体上，这个仓库解决的是"如何用声明式模板获得接近手写优化代码的性能"：编译器负责把声明翻译成带提示的指令序列，响应式系统负责精确知道"什么变了"，渲染器负责只更新需要更新的部分。三者互为因果，构成本仓库的骨架。

## 3. 技术栈与依赖

- 语言：TypeScript 全量编写，编译期用 `__DEV__`/`__BROWSER__`/`__FEATURE_OPTIONS_API__` 等全局常量做死代码消除（见 `vitest.config.ts` 的 define 段与 `rollup.config.js` 的 replace 插件），一份源码派生出开发/生产、浏览器/SSR 等十余种构建变体。
- 构建：开发用 esbuild（`scripts/dev.js` 注释"Using esbuild for faster dev builds"），正式产物用 Rollup（`rollup.config.js`），类型声明单独走 `tsc + rollup.dts.config.js`（`package.json` 的 `build-dts` 脚本）。
- 测试：Vitest（根 `vitest.config.ts`），含 jsdom 单测、@vitest/browser-playwright 浏览器 e2e、dts 类型测试与 vitest bench 基准。
- 运行时零第三方依赖：`packages/*` 的 dependencies 全部是 workspace 内包（如 runtime-core 依赖 `@vue/shared`、`@vue/reactivity`）。
- 编译器侧第三方库（`pnpm-workspace.yaml` catalog 统一管理版本）：`@babel/parser`/`@babel/types`（解析 `<script>` 与表达式）、`estree-walker`（AST 遍历）、`magic-string`（源码改写与 sourcemap）、`entities`（HTML 实体解码）、`source-map-js`。
- 工程链：ESLint 9（`eslint.config.js`）、Prettier、simple-git-hooks + lint-staged（`package.json`）、Renovate（`.github/renovate.json5`），并启用 `minimumReleaseAge: 1440`（依赖发布 24 小时内不升级）与 `trustPolicy: no-downgrade` 等供应链防护（`pnpm-workspace.yaml`）。

## 4. 目录结构与代码组织

顶层是典型 pnpm monorepo，`packages/` 放对外发布的包，`packages-private/` 放不发布的辅助工程：

| 目录 | 职责 |
| --- | --- |
| `packages/shared` | 跨包共享的纯工具函数（`isArray`/`extend` 等），无依赖 |
| `packages/reactivity` | 响应式核心：Proxy 代理、依赖追踪、computed/watch/effectScope |
| `packages/compiler-core` | 平台无关模板编译器：tokenizer/parser/transform/codegen |
| `packages/compiler-dom` | 浏览器 HTML 方言：指令、原生元素、实体、`v-html` 安全处理 |
| `packages/compiler-ssr` | SSR 专管：不走 vdom，直接把模板编译成字符串拼接函数 |
| `packages/compiler-sfc` | 单文件组件：`parse`/`compileScript`/`compileTemplate`/`compileStyle` |
| `packages/runtime-core` | 平台无关运行时：vnode、组件实例与生命周期、渲染器、调度器 |
| `packages/runtime-dom` | 浏览器宿主：`nodeOps`、`patchProp`、事件/Transition/KeepAlive 模块、`defineCustomElement` |
| `packages/runtime-test` | 面向测试的自定义宿主实现 |
| `packages/server-renderer` | SSR 渲染入口，消费 compiler-ssr 产物 |
| `packages/vue` | 面向用户的完整包：`index.ts`（全量编译器版）、`runtime.ts`（仅运行时版）、`dev.ts` |
| `packages/vue-compat` | Vue 2 兼容层，帮助迁移 |
| `packages-private/dts-test` | 类型定义的"快照测试"（*.test-d.ts） |
| `packages-private/sfc-playground` | 在线 SFC 演练场 |
| `packages-private/template-explorer` | 模板编译输出调试工具 |
| `scripts/` | dev/build/release 等脚本（esbuild/rollup 封装） |
| `changelogs/` | 按 minor 版本拆分的历史 CHANGELOG（3.0~3.4），根目录 `CHANGELOG.md` 为 3.5+ |

组织方式评述：严格的分层依赖图（shared → reactivity/compiler-core → runtime-core/compiler-dom → runtime-dom/server-renderer → vue），每个包自带 `package.json`/`LICENSE`/`README` 独立发布；编译器与运行时通过"编译产物约定"（渲染辅助函数 `createVNode`/`createElementBlock` 等）而非运行时接口耦合，保持两层可独立替换。这是学习 monorepo 分包与"编译器-运行时协同"设计的范本。

## 5. 核心架构与关键实现

### 5.1 reactivity：Dep/Link 双向链表 + 版本号

3.5 重写后的依赖追踪不再是简单的 Set 容器：每个响应式属性对应一个 `Dep`，每个副作用/计算属性是一个 `Subscriber`，二者的多对多关系由 `Link` 节点同时挂成两条双向链表——一条是订阅者的依赖链，一条是属性的订阅者链（`packages/reactivity/src/dep.ts:32-62` 的类注释明确说明这一设计）。双向链表让依赖清理与重挂链接近 O(1)，显著降低大数组/深层对象场景的 GC 压力。

```ts
// packages/reactivity/src/dep.ts:32
export class Link {
  version: number
  nextDep?: Link; prevDep?: Link   // 订阅者的 deps 链表
  nextSub?: Link; prevSub?: Link   // 属性的 subs 链表
  prevActiveLink?: Link
  constructor(public sub: Subscriber, public dep: Dep) {
    this.version = dep.version
    ...
  }
}
```

另有一个全局递增 `globalVersion`："Incremented every time a reactive change happens"（`dep.ts:14-19`），配合每个 `Dep` 自身的 `version`，让 computed 在批量触发时能先比对版本号、完全无变化则跳过重算（快路径在 `computed.ts` 的 `ComputedRefImpl`，它本身实现 `Subscriber` 接口，见 `packages/reactivity/src/computed.ts:44` 起的类定义）。`effect.ts` 中 `activeSub` 指针 + `startBatch/endBatch` 实现"拉取式"的订阅去重与批量刷新。`reactive.ts`/`baseHandlers.ts`/`collectionHandlers.ts` 负责 Proxy get/set/has/delete 拦截，`arrayInstrumentations.ts` 对 `includes/indexOf` 等数组方法做了代理感知的特殊处理。

### 5.2 compiler-*：tokenizer/parser → transform → codegen 三段流水线

模板先由手写的 `tokenizer.ts`（1197 行，状态机扫描，把标签/属性/文本解析为 token 流）喂给 `parser.ts`（1079 行，构建带 `SourceLocation` 的 AST），随后 `transform.ts` 按深度优先执行一组插件式 transform（`packages/compiler-core/src/transforms/`：`transformElement`、`vOn`、`vFor`、`vIf`、`vSlot`、`cacheStatic` 静态提升与块级缓存等），最后 `codegen.ts`（1118 行）生成渲染函数。SSR 路径（compiler-ssr）复用 parser 但换成生成字符串拼接代码。`compiler-sfc/src/compileScript.ts`（1417 行）处理 `<script setup>`：用 @babel/parser 解析、estree-walker 遍历标识符，把顶层声明编译成 `setup()` 返回值并生成宏（`defineProps/defineEmits`）的运行时代码。

关键洞察在编译产物里：codegen 给动态节点打上 `PatchFlags`（如 `TEXT`/`PROPS`/`CLASS`）并收集 `dynamicChildren`，运行时据此只 diff 标记过的部分。

### 5.3 runtime-core：带优化提示的 patch 与组件调度

渲染器的入口 `patch` 函数（`packages/runtime-core/src/renderer.ts:384`）先做同类型判断，不同类型直接卸载重建；`optimized` 开关由 vnode 是否带 `dynamicChildren` 决定——带提示时走"只遍历动态子节点"的快速路径，不带则回退完整 `patchChildren`（含最长递增子序列算法的 `patchKeyedChildren`，`renderer.ts:1834`）。

```ts
// packages/runtime-core/src/renderer.ts:384
const patch: PatchFn = (n1, n2, container, anchor = null,
    parentComponent = null, parentSuspense = null, ...) => {
  if (n1 === n2) return
  // patching & not same type, unmount old tree
  if (n1 && !isSameVNodeType(n1, n2)) {
    anchor = getNextHostNode(n1)
    unmount(n1, parentComponent, parentSuspense, true)
    n1 = null
  }
  if (n2.patchFlag === PatchFlags.BAIL) { optimized = false; ... }
```

组件更新不直接递归，而是把渲染函数包装成带 id 的调度任务放进 `scheduler.ts` 的最小堆队列：job 按自增 id 排序保证父先于子执行，`SchedulerJobFlags.PRE/ALLOW_RECURSE` 等位标志（`packages/runtime-core/src/scheduler.ts:6-24`，注释里还引用了 #1740/#1727 两个历史 issue 说明设计动机）控制 pre-flush watcher 与递归更新保护，重复入队自动去重。组件实例系统在 `component.ts`（1313 行）：`setup()` 结果规范化、props/emits 校验、生命周期钩子注册都汇聚于此；`apiCreateApp.ts:253` 的 `createAppAPI(render, hydrate)` 闭包注入具体平台实现，形成"核心 + 可插拔宿主"的最终组装点（浏览器宿主见 `packages/runtime-dom/src/nodeOps.ts` 与 `patchProp.ts`）。

## 6. 构建与运行

以下命令均来自根 `package.json` 的 scripts 与 `.github/contributing.md`：

```bash
pnpm i                                  # 安装（preinstall 强制 pnpm，postinstall 装 git hooks）
pnpm dev                                # esbuild 增量构建单个包（scripts/dev.js）
pnpm dev-compiler                       # 编译器调试：template-explorer + 本地服务
pnpm build                              # Rollup 全量构建（scripts/build.js）
pnpm build-dts                          # tsc + rollup 生成 .d.ts
pnpm check                              # tsc --noEmit 全仓类型检查
pnpm lint / pnpm format                 # ESLint / Prettier
pnpm test                               # vitest watch 模式
pnpm test-unit                          # 单元测试（unit projects）
pnpm test-e2e                           # 构建 global 包后跑 e2e + 浏览器测试
pnpm test-dts                           # 构建类型后跑 packages-private/dts-test
pnpm test-coverage                      # v8 覆盖率
pnpm bench / pnpm bench-compare         # 性能基准与对比
pnpm release                            # 发布脚本（scripts/release.js，配合 conventional-changelog）
```

不提供"运行应用"入口——本仓库是库源码；体验编译输出可用 `pnpm open`（template-explorer，`package.json`）或 `packages-private/sfc-playground`。

## 7. 测试与工程质量

- 测试规模：197 个 `*.spec.ts`，共约 95,454 行（`find packages -name '*.spec.ts' | wc -l` + `wc -l` 统计），测试代码约为源码 1.65 倍；另有 `packages-private/dts-test/` 下数十个 `*.test-d.ts` 类型测试和 `packages/reactivity/__benchmarks__/` 6 个基准文件。
- 风格：测试与源码同包同目录（`src/__tests__/` 镜像源文件名，如 `reactivity/__tests__/computed.spec.ts` 对应 `src/computed.ts`），单测（jsdom）、浏览器 e2e（Playwright）、SSR/水合、类型测试、GC 压力测试（`reactivity/__tests__/gc.spec.ts`）分层齐全。
- CI（`.github/workflows/`）：`ci.yml`（pnpm build + pkg-pr-new 按 PR 发布临时包）、`test.yml`、`size-report.yml`（每个 PR 报告 bundle 体积变化）、`ecosystem-ci-trigger.yml`（对下游生态仓库做回归，Vue 生态健康度的关键保障）、`autofix.yml`（自动 lint 修复）、`release.yml`、`lock-closed-issues.yml`。
- 质量观察：类型系统用法极深（`renderer.ts` 的泛型 HostNode 抽象、`vue/macros` 全局类型）；错误处理有专门的 `errorHandling.ts` 错误码体系；`compiler-core/src/errors.ts` 集中管理全部编译错误码与中英链接文档。这是公开可读性最好的一线框架内核之一。

## 8. 社区与维护状态

- 活跃度：最近 push 2026-11-02（GitHub API），近 6 次提交跨 2026-10-21 至 11-02，保持每周多提交节奏；964 个 open issue。
- 发布节奏：稳定版走 main（v3.5.44，2026-10-13）；下一个 minor 从 `minor` 分支发预发布，近四次 release 为 v3.6.0-rc.14（10-30）、rc.13（10-23）、rc.12（10-16）、v3.5.44（10-13）——约每周一个 RC，即将落地的 3.6 是大版本。
- 分支模型：`.github/maintenance.md:99-119` 官方文档明确"main=补丁线、minor=特性线，稳定 minor 发布后 main fast-forward 对齐"，因此默认分支 HEAD（3.5.44）中不含任何 vapor 字符串（全文 grep 为 0），3.6 的 Vapor 无虚拟 DOM 渲染器在 `minor` 分支开发——读源码时需注意分支选择。
- 贡献者结构：BACKERS.md 赞助名单 + README 中的 contributors 图（前 500 名）； Evan You 仍是最终维护者，另有核心团队（具体名单在仓库中未单独列出，"未确认"细节）。治理文档 `.github/maintenance.md` 公开了 issue 分诊、PR 评审与合并规则，成熟度很高。

## 9. 亮点与风险

亮点：

1. 教科书级的分层：响应式/编译器/运行时/宿主四层通过清晰接口解耦，任何一层都可独立阅读、替换或单独发布（见第 4 章依赖图与 `packages/reactivity/package.json` 独立发布配置）。
2. 3.5+ 响应式实现的现代化：Dep/Link 双向链表 + 版本号快路径（`dep.ts`），在 computed 链、大数组、GC 压力等场景有可衡量的改进，配了专门基准与 GC 测试。
3. 编译期优化彻底：PatchFlags/dynamicChildren/静态提升（`transforms/cacheStatic.ts`）让运行时 diff 只碰必要节点，是"编译器为运行时兜底"架构的代表作。
4. 工程质量标杆：9.5 万行测试、类型测试、体积报告、生态回归 CI、供应链防护（minimumReleaseAge）、公开的维护手册一应俱全。
5. 平滑迁移意识：vue-compat 包 + 编译器 compat 警告体系（`compiler-core/src/compat/`）为 Vue 2 用户提供了可机检的迁移路径。

风险/局限：

1. 分支模型易误读：默认分支 main 不含 3.6/Vapor 内容，基于 HEAD 的第三方分析容易过时（本次实测 main 全仓无 vapor 代码）。
2. 核心复杂度集中在少数文件：`renderer.ts` 2681 行、`compileScript.ts` 1417 行，阅读与贡献的局部认知负担大。
3. Vue 2→3 迁移遗留负担：vue-compat 与编译器 compat 层（含 deprecation 警告）会长期占据维护带宽。
4. issue 体量大（964 open），社区问题主要靠文档与论坛分流，issue 区不答疑（`README.md` 明确说明）。

## 10. 总结评价

vuejs/core 是 TypeScript monorepo 与前端框架内核设计的双重标杆：响应式系统用 Proxy + 双向链表依赖追踪做到了"精确知道什么变了"，编译器用三段流水线把声明式模板翻译成带优化提示的指令，运行时则用带 PatchFlags 的 patch 算法与最小堆调度器把每次更新压到最小。仓库自带的 9.5 万行测试、体积报告与生态回归 CI，体现了头部开源项目的工程质量。它正处于 3.6（Vapor 模式）发布前夜，main/minor 双分支并行，读源码时应先确定目标版本分支。适合三类人：想理解现代前端框架内部机制的工程师（从 `reactivity/src/dep.ts` 与 `runtime-core/src/renderer.ts` 入手最佳）、准备给 Vue 提 PR 的贡献者（先读 `.github/contributing.md` 与 `maintenance.md`）、以及想借鉴 monorepo 分包/CI/发布流水线实践的维护者。
