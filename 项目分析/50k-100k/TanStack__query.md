# TanStack/query 项目分析

> 星数 50467（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：框架无关的异步状态管理 / 服务端状态缓存库，以 `@tanstack/query-core` 为核心、为 React/Vue/Solid/Svelte/Angular/Lit/Preact 提供统一 Hooks 适配的 monorepo。

| 项目 | 内容 |
| --- | --- |
| 名称 | TanStack/query（原 React Query） |
| 星数 | 50467（快照 2026-10-24） |
| 语言 | TypeScript（全仓 ~49k 行 TS/TSX，`packages/*/src` 统计） |
| 许可证 | MIT（LICENSE，Copyright 2021-present Tanner Linsley） |
| 版本 | v5.105.3（packages/query-core/package.json、packages/react-query/package.json） |
| 最近活跃 | 2026-11-03 push（gh api 元数据），创建于 2019-09-10 |
| 包管理 | pnpm 12 + Nx monorepo + changesets（package.json、.changeset/） |

## 2. 项目是什么，解决什么问题

TanStack Query 解决的是前端应用中"服务端状态"（server-state）的管理问题。组件库自带的状态管理（useState/useReducer）适合客户端状态，但服务端数据的缓存、去重、失效、重试、后台刷新、分页等一系列横切逻辑，每个项目都要重复实现。README.md 明确定位为"async state management library built to simplify fetching, caching, synchronizing, and updating server state"，核心能力包括：协议无关的请求（REST/GraphQL/任意 Promise）、缓存与后台重取、分页与无限滚动、mutation、依赖查询、预取、取消与 Suspense 支持（README.md）。

项目的前身是 React Query（2019 年创建），v4 起把框架无关的逻辑抽到 `@tanstack/query-core`，各框架只保留一层薄适配（packages/query-core/package.json 描述："The framework agnostic core that powers TanStack Query"）。当前 monorepo 内同时维护 React、Vue、Solid、Svelte、Angular（experimental）、Lit、Preact 七个适配器及各自的 devtools 与 persist-client 插件（packages/ 目录）。

设计上的核心抽象是"查询缓存 + 观察者"：数据以 `queryHash` 为键存入全局 QueryCache，组件通过 QueryObserver 订阅自己关心的查询；缓存条目无人观察后按 gcTime 自动垃圾回收，组件重新挂载时若数据已过期（stale）则触发后台重取。这一模型让"声明式写查询、命令式做失效"成为可能——组件只声明 `queryKey + queryFn`，失效与刷新由 `invalidateQueries` 统一驱动。

## 3. 技术栈与依赖

- 语言：TypeScript（严格模式，多版本 TS 矩阵验证，见下）；构建产物 ESM + CJS 双格式。
- 构建：tsdown（Rolldown 系）打包，tsc 生成 d.ts（packages/query-core/package.json scripts）。
- 测试：Vitest 5 + @testing-library（jsdom 环境），覆盖率用 istanbul provider（根 package.json devDependencies）。
- 构建/任务编排：Nx（`nx affected` 按 changed 范围跑测试与构建，.github/workflows/pr.yml）。
- 包依赖极少：`@tanstack/react-query` 运行时唯一依赖是 `@tanstack/query-core`（packages/react-query/package.json dependencies），query-core 本身零运行时依赖（packages/query-core/package.json 无 dependencies 字段），全部能力基于原生 Promise/AbortController。
- 工程质量工具链：oxlint + oxfmt（格式化/静态检查）、eslint、sherif（workspace 依赖一致性）、knip（死代码检测）、publint + attw（构建产物规范校验）、size-limit（体积预算，.size-limit.json）。
- 类型兼容矩阵：同时对 TypeScript 5.7/5.8/5.9/7.0/7.1-dev 跑类型测试（packages/query-core/package.json test:types:ts57~ts71，typescript57~71 别名依赖，根 package.json）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| packages/query-core | 框架无关核心：QueryClient/Query/QueryCache/MutationCache/Observer/Retryer（~10.8k 行） |
| packages/react-query | React 适配：useQuery/useMutation/useSuspenseQuery 等（~28.7k 行含测试） |
| packages/{vue,solid,svelte,angular-query-experimental,lit,preact}-query | 其余六个框架适配器 |
| packages/query-devtools 及各框架 *-devtools | 开发者工具面板（React 实现为基准，其余封装） |
| packages/query-persist-client-core、query-{async,sync}-storage-persister | 缓存持久化插件体系 |
| packages/eslint-plugin-query、query-codemods | 生态插件：lint 规则与 v3→v5 迁移 codemod |
| packages/query-test-utils | 共享测试工具包 |
| docs/ | 文档源码（docs/framework/{react,vue,solid,svelte,angular,lit,preact}） |
| examples/ | 七个框架的示例工程（examples/react/* 等） |
| integrations/、scripts/、.github/ | 集成测试、文档生成/链接校验脚本、CI 工作流 |

组织方式评述：典型的多包 monorepo，"一个 core + N 个适配器"的扇形结构，devtools 与 persist 也按框架分包。所有包由根 package.json 的 Nx 目标统一编排（test:lib/test:types/test:build/test:knip 等），包间用 `workspace:*` 协议互引；各包配置文件通过符号链接复用根配置（本次 tarball 中 42 个符号链接均为此类指针文件，指向根 oxlint.config.ts 与 scripts/getTsdownConfig.js）。

## 5. 核心架构与关键实现

### 5.1 QueryCache 与 Query：哈希索引 + 状态机 + GC

缓存本体是 `Map<string, Query>`，以 `queryHash`（queryKey 序列化）为键（packages/query-core/src/queryCache.ts:145 `this.#queries = new Map<string, Query>()`）。每个 Query 是一台小型事件状态机，状态迁移统一走 `#dispatch`，事件类型只有 8 种（query.ts:211-246：failed/fetch/success/error/invalidate/pause/continue/setState），所有副作用（重试、暂停、GC 计时重置）都收敛在这条事件管道里，这让订阅与调试非常干净。

GC 直接做在 `Removable` 基类上，Query 与 Mutation 共用：无人观察后按 gcTime 定时移除，客户端默认 5 分钟、服务端 Infinity（packages/query-core/src/removable.ts:42-48 `isServerEnvironment() ? Infinity : 5 * 60 * 1000`）；`optionalRemove` 只有在无观察者且 fetchStatus 为 idle 时才真正出缓存（query.ts:376-380），避免删掉正在使用的条目。

fetch 流程：`#fetch` 先按查询类型选择 behavior（无限查询包 `infiniteQueryBehavior`），保存 `#revertState` 以支持取消回滚，再交给 `createRetryer` 执行，retryer 的 onFail/onPause/onContinue 回调逐一映射为 dispatch 事件（query.ts:844-874）。

### 5.2 Retryer：重试、网络感知与暂停-恢复

packages/query-core/src/retryer.ts（314 行）是请求生命周期的引擎：指数退避重试 `Math.min(1000 * 2 ** failureCount, 30000)`（retryer.ts:87-88），内置 CancelledError 与取消传播；结合 onlineManager 实现 networkMode——默认 `online` 模式下离线则暂停、上线后继续（retryer.ts:96-98 `canFetch`，retryer.ts:275 "Pause if ... the device is offline"）。Suspense、mutation、observer 全部复用这一个 retryer。

### 5.3 QueryObserver：观察者与"乐观结果"

框架适配层不直接操作 Query，而是持有 QueryObserver（packages/query-core/src/queryObserver.ts:56）。Observer 负责：订阅/退订生命周期（`onSubscribe` 首个订阅者时把自身加入 query 并判断是否 shouldFetchOnMount，queryObserver.ts:113-125）、staleTime 判定、refetchInterval 定时、以及 `getOptimisticResult`——在渲染期不触发订阅的情况下先给出乐观结果，避免闪烁。还实现了"按需追踪"（`#trackedProps`，queryObserver.ts:84），只对组件实际读取的 result 字段发通知，减少无效重渲染。

### 5.4 React 适配层：薄封装 + useSyncExternalStore

React 侧所有查询 Hook 共享 `useBaseQuery`：创建/复用 Observer，用 `React.useSyncExternalStore` 订阅 `observer.subscribe(notifyManager.batchCalls(onStoreChange))`（packages/react-query/src/useBaseQuery.ts:112-117），Suspense 与错误边界通过 suspense.ts + QueryErrorResetBoundary 注入。通知合并由 notifyManager 统一批处理（packages/query-core/src/notifyManager.ts 的 schedule/queue/transactions 机制）。Vue 适配器结构对称（packages/vue-query/src/ 下同名 useQuery/useMutation + vueQueryPlugin 插件安装），验证了 core 抽象的干净程度。

## 6. 构建与运行

来自根 package.json 与 CONTRIBUTING.md（要求 pnpm、Node 版本按 .nvmrc）：

```bash
pnpm install            # 安装依赖（CONTRIBUTING.md:42）
pnpm build:all          # 全量构建所有包（CONTRIBUTING.md:60）
pnpm dev / pnpm watch   # Nx watch 增量构建开发模式
pnpm test:lib           # 单元测试（Nx affected，vitest）
pnpm test:types         # 多 TS 版本类型测试
pnpm test:pr            # CI 同款：lint+knip+docs+lib+types+build
pnpm run generate-docs  # 从 devtools 类型生成文档（scripts/generate-docs.ts）
```

单包操作：`cd packages/query-core && pnpm test:lib`（vitest watch 加 `:dev`）。不运行项目即可阅读：docs/ 内含全部框架的文档源，examples/ 可作为运行示例。

## 7. 测试与工程质量

- 测试规模：packages 内 `__tests__` 目录共 272 个 .ts/.tsx 文件，其中测试文件 172 个（find 统计）；query-core 与 react-query 的测试行数占包体积一半以上（react-query src 含测试 28.7k 行 vs core 10.8k 行）。
- 测试风格：单元为主 + 大量使用 @testing-library/react-render-stream 做渲染序列断言（packages/react-query/package.json devDependencies），共享工具沉淀在 packages/query-test-utils。
- CI：PR 工作流用 Nx Cloud 分布式执行 `test:pr`（.github/workflows/pr.yml），并发布每提交预览包（pkg-pr-new）；release 工作流走 changesets/action 自动 version + publish（.github/workflows/release.yml:50-56）；另有 autofix.yml（自动修复）、zizmor.yml（workflow 安全审计）。
- 质量门禁密集：oxlint/eslint 双 lint、knip 死代码、sherif 依赖一致性、publint/attw 产物校验、size-limit 体积预算、6 个 TS 版本矩阵，这在同类 OSS 项目里属于最高配。

## 8. 社区与维护状态

- 活跃度：最近 push 2026-11-03（gh api），当前开放 issue 116、fork 4224；本次快照内 .changeset/ 有 2 个待发布 changeset 文件，说明处于持续小步发布节奏。
- 发布：changesets 自动化，query-core 与 react-query 版本号同步为 5.105.3（各包 package.json），v5 大版本下已迭代百余个 minor/patch。
- 治理：CONTRIBUTING.md 规范完整（pnpm/Nx 流程、文档生成、PR 约定）；仓库有 AGENTS.md 与 AI_POLICY.md（AI 协作政策），并接入 CodeRabbit 评审（.coderabbit.yaml、README Partners 区块）；赞助与商业支持（FUNDING.json、Tanner Linsley sponsors）。贡献者具体人数未在源码内可直接统计（tarball 无 git 历史），未确认。

## 9. 亮点与风险

亮点：

1. 零运行时依赖的核心：query-core 无任何 dependencies，AbortController/Promise 全原生（packages/query-core/package.json），适配器只依赖 core，架构扇形干净。
2. 事件驱动状态机 + 统一 retryer：8 种 dispatch 事件收敛全部状态迁移（query.ts），重试/离线暂停/取消回滚逻辑全仓唯一实现（retryer.ts），可测试性极强。
3. 精细的重渲染控制：trackedProps 按需追踪 + notifyManager 批处理 + optimistic result，性能是同类库标杆（queryObserver.ts、notifyManager.ts）。
4. 工程化天花板：Nx affected + Nx Cloud 分布式 CI、6 版本 TS 矩阵、knip/publint/attw/size-limit 全套门禁（根 package.json、.github/workflows/pr.yml）。
5. 生态完整：7 框架适配、devtools、persist 插件、eslint-plugin、codemods、7 框架示例与文档全部同仓维护。

风险/局限：

1. 表面积大带来的维护成本：27 个包 × 多框架，Angular 适配仍叫 experimental，lit-query 等小适配器的成熟度低于 React/Vue（packages/ 目录命名）。
2. 类型复杂度高：types.ts 达 2439 行，重类型体操对使用者报错信息与编译时间有影响（packages/query-core/src/types.ts）。
3. 运行时行为约定多（staleTime/gcTime/networkMode/refetchOn* 的交互），新手心智负担集中在这些隐式默认值上（removable.ts、retryer.ts）。
4. 快速演进中的 API 变动：源码内可见 `ensureQueryData` 已标 deprecated 改为 `queryClient.query({staleTime:'static'})`（queryClient.ts:206），v5→v6 迁移成本已在酝酿。

## 10. 总结评价

TanStack/query 是"服务端状态管理"这一品类的定义者：core 抽象（QueryCache + Observer + Retryer + Removable GC）小而锋利，多框架适配证明了其与 UI 层解耦的成功，工程化与测试强度在 50k 星级项目中属于第一梯队。它适合作为理解"缓存即状态"设计范式的必读源码——建议从 packages/query-core/src/query.ts 与 queryObserver.ts 入手，配合 removable.ts/retryer.ts 三个千行内的小文件即可看懂主链路。对使用者而言，React/Vue 适配成熟度最高可直接生产使用；对框架作者而言，query-core 的 adapter 接口是给自家框架做数据层绑定的现成蓝本。
