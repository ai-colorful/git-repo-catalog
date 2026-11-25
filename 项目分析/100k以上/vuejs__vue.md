# vuejs/vue 项目分析

> 星数 212629（快照 2026-10-24）｜主语言 TypeScript｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

Vue 2 官方源码仓库——渐进式 JavaScript 前端框架的第二代主仓库，已于 2023-12-31 正式 EOL，最新版本为 2.7.16 "Swan Song"。本仓库是 Vue 2 唯一权威源码（Vue 3 已迁至 vuejs/core）。

| 项目 | 值 | 出处 |
| --- | --- | --- |
| 名称 | vuejs/vue | GitHub |
| 当前版本 | 2.7.16（2023-12-24 发布，代号 Swan Song） | `package.json`、`CHANGELOG.md` |
| 星数 | 212,629（快照）/ 212,569（API 实时） | GitHub |
| 语言 | TypeScript（2.7 起由 Flow 全面迁移为 TS） | `package.json`、`src/**/*.ts` |
| 许可证 | MIT | `LICENSE`、`package.json` |
| 默认分支 | main | GitHub API |
| 最近提交 | 2024-10-10（chore: fix spelling mistakes） | GitHub API commits |
| 最近 release | v2.7.16（2023-12-24） | GitHub API releases |
| 维护状态 | EOL（End of Life），仅剩杂项提交，未归档 | `README.md` |

## 2. 项目是什么，解决什么问题

Vue 是一个"渐进式"前端框架（`README.md`：progressive framework，"incrementally adoptable"），核心思路是声明式、组件化的视图层：开发者用模板描述 UI 与数据的绑定关系，框架负责在数据变化时高效地更新 DOM。它的定位介于"库"（引一个 script 标签即可增强页面）与"框架"（配合 vue-router/vuex/vue-cli 构建完整 SPA）之间，这是"渐进式"一词的含义（`README.md` Ecosystem 一节）。

Vue 2 解决的核心问题按源码可归纳为三层：一是**响应式数据绑定**——把普通 JS 对象转成可观察对象，读写自动收集/触发依赖（`src/core/observer/`）；二是**模板编译**——把 HTML 风格的模板字符串编译成渲染函数，并在编译期做静态子树提升等优化（`src/compiler/`）；三是**虚拟 DOM 与补丁算法**——渲染函数产出 vnode 树，diff 后最小化操作真实 DOM（`src/core/vdom/`）。三层共同构成"数据变 → 依赖触发 → 重渲染 → diff → DOM 更新"的闭环。

2.7 版本的特殊意义：它是 Vue 2 的最终形态，从 Vue 3 反向移植了 Composition API（`setup()`、`ref`、`reactive`、`computed`、`watch`、`provide/inject`、`effectScope` 等，见 `src/v3/`），让 Vue 2 用户在不升级大版本的情况下使用 Vue 3 的编程范式，为迁移铺路。仓库 `README.md` 顶部明确声明 EOL：不再有新特性、更新或修复，发行渠道（CDN、npm）仍可用，并为有合规需求的用户指向 Herodevs 的商业支持"Vue 2 NES"。

作为 GitHub 上星数最高的前端框架仓库之一，它的历史价值在于：定义了"模板 + 响应式 + 渐进式"这一被后续大量框架借鉴的产品形态，其响应式与 diff 实现至今仍是学习框架原理的经典教材。

## 3. 技术栈与依赖

- **语言**：TypeScript 4.8（`package.json` devDependencies），2.7 起从 Flow 迁移；类型声明独立维护于 `types/` 并用 api-extractor 生成打包类型。
- **包管理**：pnpm 8.9.2，`pnpm-workspace.yaml` 声明 `packages/*` 为 workspace 子包（compiler-sfc、server-renderer、template-compiler）。
- **构建**：Rollup 2.x（`scripts/config.js`、`scripts/build.js`），按 TARGET 环境变量产出 20+ 种 dist 变体；类型打包用 `@microsoft/api-extractor`。
- **运行时依赖极简**：`dependencies` 仅 `@vue/compiler-sfc`（workspace 内）与 `csstype`；模板 HTML 实体解码用 `he`，模板表达式解析用 `@babel/parser`（均为编译期/devDependencies 侧）。
- **测试**：Vitest 1.x（单元/SSR/SFC）、Karma + Jasmine + Puppeteer（transition 与 e2e，`package.json` scripts `test:transition`/`test:e2e`）。
- **工程化**：prettier（统一格式）、yorkie + lint-staged（pre-commit）、verify-commit-msg（Angular 风格提交校验，`scripts/verify-commit-msg.js`）、conventional-changelog（CHANGELOG 自动生成）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `src/core/` | 平台无关核心：实例生命周期、响应式、虚拟 DOM、全局 API |
| `src/compiler/` | 模板编译器：parser → optimizer → codegen 三段式 |
| `src/platforms/web/` | Web 平台专属代码与各构建入口（entry-runtime / runtime-with-compiler 等） |
| `src/shared/` | 编译器与运行时共享常量、工具 |
| `src/v3/` | 从 Vue 3 反向移植的 Composition API 实现（setup/ref/reactive/computed/watch/effectScope 等） |
| `packages/compiler-sfc/` | 单文件组件（.vue）解析与编译子包（parse/compileScript/compileStyle/compileTemplate） |
| `packages/server-renderer/` | 服务端渲染子包（含 string-render 与 optimizing-compiler 两套） |
| `packages/template-compiler/` | 供 vue-template-compiler 兼容用途的子包 |
| `types/` | 对外 TypeScript 类型声明（index.d.ts、options、plugin、jsx 等） |
| `dist/` | 预构建产物（vue.common.js、vue.runtime.common.js、vue.runtime.mjs） |
| `test/` | unit（vitest）/ e2e（puppeteer）/ transition（karma）三层测试 |
| `examples/`、`benchmarks/` | classic 与 composition 两组示例；big-table/dbmon/ssr 等性能基准 |
| `scripts/` | Rollup 构建配置、发布脚本、commit 校验 |

组织方式评述：采用"单主包 + pnpm workspace 小子包"的混合结构——核心框架在 `src/` 内按 core/compiler/platforms 分层，编译期与运行期能力（SFC 编译、SSR）拆为独立 workspace 包以控制主包体积；平台差异通过 `platforms/web` 入口文件组合 core 能力实现（多入口构建而非运行时分支）。整个 `src/` 共 148 个 .ts 文件、约 1.64 万行，体量精悍。

## 5. 核心架构与关键实现

### 5.1 响应式系统：Object.defineProperty + Dep/Watcher（src/core/observer/）

Vue 2 响应式的基石是 `Object.defineProperty` 拦截 getter/setter（`src/core/observer/index.ts` `defineReactive`）。getter 中当存在当前求值的 `Dep.target`（即 Watcher）时执行 `dep.depend()` 收集依赖，并对嵌套对象/数组补挂 `childOb.dep`；setter 中先 `hasChanged` 短路，再 `observe(newVal)` 递归观测新值并 `dep.notify()`（`src/core/observer/index.ts` L155-213）。这是"基于 getter/setter 而非 Proxy"的经典实现，也因此带来 Vue 2 无法侦测属性新增/删除的固有局限（需要 `Vue.set`/`Vue.delete`，同文件 L221 起）。

数组无法用 defineProperty 拦截索引，故另取巧路：以 `Object.create(Array.prototype)` 派生 `arrayMethods`，重写 push/pop/shift/unshift/splice/sort/reverse 七个变更方法，在调用原方法后手动 `ob.dep.notify()`（`src/core/observer/array.ts` L14-54）。

依赖的调度由 Watcher + Scheduler 完成：`Watcher` 类同时服务于渲染 watcher、`$watch` 和 computed（`src/core/observer/watcher.ts`，"used for both the $watch() api and directives"）；`queueWatcher` 把 watcher 放入队列在 nextTick 中批量异步刷新，去重（`has` 表）、按 id 排序保证父先于子更新，并以 `MAX_UPDATE_COUNT = 100` 检测循环更新（`src/core/observer/scheduler.ts` L1-199）。2.7 中 Watcher 还实现了 Vue 3 的 `DepTarget` 接口并接入 `activeEffectScope`（`watcher.ts` import `v3/reactivity/effectScope`），使 Composition API 的 effect 与旧响应式系统共用一套依赖机制。

### 5.2 模板编译器：parser → optimizer → codegen 三段管线（src/compiler/）

编译管线为：HTML 模板字符串 → AST → 优化后的 AST → render 函数字符串，入口在 `src/compiler/index.ts`，由 `create-compiler.ts` 组装。parser 基于自研 HTML 解析器（`src/compiler/parser/html-parser.ts`，341 行状态机式标签解析）+ 表达式解析（`text-parser.ts`、`filter-parser.ts`），主流程 `src/compiler/parser/index.ts` 近千行，处理元素/属性/指令/插槽作用域等全部语法。

optimizer 是 Vue 2 编译期的招牌优化：两遍扫描标记纯静态节点（`markStatic`）与静态子树根（`markStaticRoots`），被标记的子树在重渲染时直接提升为常量 vnode 缓存、并在 patch 时整体跳过（`src/compiler/optimizer.ts` 头部注释明确写出这两个目的）。codegen 则递归遍历 AST 生成 `_c/_v/_s` 调用形式的 render 函数字符串（`src/compiler/codegen/index.ts`，668 行），事件/过滤器分别在 `events.ts`、`filter-parser.ts` 处理。

### 5.3 虚拟 DOM 与 diff（src/core/vdom/）

`vnode.ts` 定义 vnode 数据结构；`create-element.ts` 把 render 结果转成 vnode 树；`patch.ts`（907 行）是核心 diff。节点可复用判定 `sameVnode`：key 相同、asyncFactory 相同，且 tag/isComment/data 存在性/sameInputType 全一致（`src/core/vdom/patch.ts` L36-44）。diff 采用双端对比：同时维护新旧子列表的首尾四个指针，依次尝试 oldStart/newStart、oldEnd/newEnd、oldStart/newEnd、oldEnd/newStart 四组 `sameVnode` 命中即复用，否则建 key→index 映射查找，最后统一处理插入与移除（`patch.ts` L444-513）。对比 React 的单向遍历，双端 diff 在头部插入/反转等场景下移动次数更少，是 Vue 2 渲染性能的关键设计之一。

组件在 vnode 层面被抽象为 `componentVNodeHooks` 的 init/prepatch 等 hook（`create-component.ts`），使组件实例化与数据更新统一走同一条 patch 路径。

### 5.4 实例构造：mixin 组合模式 + v3 能力反哺（src/core/instance/、src/v3/）

`Vue` 构造函数极简，只有 `_init` 一行；全部实例能力通过五个 mixin 在类外按功能注入——`initMixin/stateMixin/eventsMixin/lifecycleMixin/renderMixin`（`src/core/instance/index.ts` L11-27，用 `@ts-expect-error` 压制 TS 对动态挂载的抱怨）。`init.ts` 依序执行合并配置、初始化生命周期/事件/渲染/inject/provide/state，`state.ts`（393 行）负责 props/data/methods/computed/watch 的响应式初始化。`src/v3/` 则承载 Composition API：`apiSetup.ts`（246 行，setup 环境与返回值处理）、`reactivity/ref.ts|reactive.ts|computed.ts|effect.ts|effectScope.ts` 提供 Vue 3 风格原语并与 5.1 的 Dep/Watcher 打通——这正是 2.7 "反向移植"的实现方式。

## 6. 构建与运行

以下命令均来自 `package.json` scripts（未实际运行，静态确认）：

```bash
pnpm install                 # pnpm@8.9.2（packageManager 字段）
pnpm run dev                 # rollup -w 监听构建 full-dev 产物
pnpm run build               # node scripts/build.js，全量产出 dist 各变体
pnpm run build:ssr           # runtime-cjs + server-renderer 两个 target
pnpm run build:types         # tsc 生成声明 + api-extractor 打包类型

pnpm run test:unit           # vitest run test/unit
pnpm run test:ssr            # build:ssr 后 vitest run server-renderer
pnpm run test:sfc            # vitest run compiler-sfc
pnpm run test:e2e            # 构建 full-prod 后 vitest run test/e2e
pnpm run test:transition     # karma + Chrome
pnpm run ts-check            # tsc --noEmit 源码类型检查
pnpm run release             # node scripts/release.js 发布流程
```

## 7. 测试与工程质量

- **测试规模**：`test/unit` 下 108 个 `.spec.ts`，按 features（component/directives/options/v3 等）与 modules（compiler/observer/vdom/patch 等按源码模块镜像）组织（`test/unit/` 目录结构）。另有 `test/e2e`（puppeteer 驱动 6 个示例应用）、`test/transition`（karma）。
- **CI**：GitHub Actions（`.github/workflows/ci.yml`）四个 job：unit-test、ssr-sfc-test、e2e-test、type-test（`ts-check` + `test:types`），Node 18 + pnpm。README 徽章仍指向旧 CircleCI/Codecov（README 与实际 CI 已不同步，属 EOL 停维护痕迹）。
- **类型质量**：源码全 TS + `tsc --noEmit` 把关；对外类型用 `types/` 手写声明 + `test:types`（编译 `types/test/` 下的类型用例）验证，另用 api-extractor 保证声明不意外破坏（`package.json` `build:types`）。
- **代码质量观察**：注释质量高，大量 issue 编号内联引用（如 `observer/index.ts` `#7981`）；`__DEV__` 分支携带开发期告警与调试钩子（`TrackOpTypes`/`TriggerOpTypes` 调试事件）；提交信息强制 Angular 规范（`scripts/verify-commit-msg.js`）。遗留小瑕疵：`observer/array.ts` 顶部注释仍写"flow doesn't play well"（Flow 时代残留）。

## 8. 社区与维护状态

- **EOL 状态（重点）**：`README.md` 顶部声明 Vue 2 于 2023-12-31 到达 End of Life，不再接收新特性、更新与修复；发行渠道继续可用，并向存量用户推荐迁移指南（v3-migration.vuejs.org）或 Herodevs 的"Vue 2 NES"商业延长支持。
- **仓库形态**：未归档（GitHub API `archived: false`），但停止实质开发。最后实质 release 为 v2.7.16 "Swan Song"（2023-12-24，GitHub API）；EOL 后仅剩 4 次杂项提交——2023-12-31 更新 issue 模板与 EOL README，2024 年三次 chore（修链接、修拼写），最后推送 2024-10-10（GitHub API commits）。
- **issue 处理**：open issues 637（GitHub API），EOL 后官方不再修复，多为历史存量。
- **贡献者结构**：contributors API 单页即达 100 上限，实际贡献者数百（未精确确认）；2.7 主体由 Evan You 与核心团队完成（CHANGELOG 与发布记录可证），未确认。
- **生态位置**：npm `vue` 包 2.x 仍是最广泛部署的前端运行时之一；本仓库的 `dist/` 预构建产物（vue.common.js 等）与 `packages/template-compiler` 是存量 Vue 2 项目的兼容层。

## 9. 亮点与风险

**亮点**

1. 响应式实现教科书级：defineProperty 拦截 + 数组方法劫持 + 依赖批量调度 + 循环检测，代码量小而机制完整（`src/core/observer/` 全部 6 个文件合计不足 1000 行）。
2. 编译期优化先于"编译时框架"流行多年：static subtree 提升与 patch 跳过（`src/compiler/optimizer.ts`）是 Vue 3 静态提升、block tree 思想的直接前身。
3. 2.7 的架构缝合出色：`src/v3/` 的 Composition API 与旧 Dep/Watcher 系统共享 effectScope，同一响应式内核跑两代 API。
4. 工程完整度高：单仓库内含编译器、运行时、SSR、SFC 工具链、类型测试、e2e 与发布脚本，且测试按源码模块镜像组织，可读可查。
5. 代码注释带 issue 编号与设计动机，可考古性强。

**风险/局限**

1. 已 EOL：无安全补丁、无新特性，生产使用必须接受无官方维护或购买商业支持（README 明示）。
2. 响应式固有局限：无法侦测属性增删与数组索引赋值，需 `Vue.set`；这是设计选择（IE 兼容）而非缺陷，但迁移时是常见踩坑点（`src/core/observer/index.ts`）。
3. 技术栈已老：TS 4.8、Vitest 1.x、karma/puppeteer 老版本，新环境（新 Node/依赖）下构建测试可能需要自行修补，CI 基线停留在 Node 18（`.github/workflows/ci.yml`）。
4. 文档与仓库存在漂移：README 徽章指向已停用的 CircleCI/Codecov，`observer/array.ts` 残留 Flow 注释，仓库不再清理此类不一致。

## 10. 总结评价

vuejs/vue 是全球星数最高的仓库之一，也是最成功的第二代前端框架的完整源码：以约 1.6 万行 TypeScript 实现了响应式、模板编译、虚拟 DOM、SSR 与 Composition API 的完整闭环，工程与注释质量至今仍是范本。它已于 2023 年底 EOL，仓库进入"冻结但有最终形态"的状态——2.7.16 就是句号（Swan Song），此后只有零星杂务提交。今天读它的最佳理由是学习框架内核设计（尤其 observer 与 compiler 两个目录），以及维护必须停留在 Vue 2 的存量系统；新项目应直接使用 vuejs/core（Vue 3）。
