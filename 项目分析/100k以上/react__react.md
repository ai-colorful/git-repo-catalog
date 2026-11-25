# react/react 项目分析

> 星数 251717（快照 2026-10-24）｜主语言 JavaScript｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

React 是用于构建 Web 与原生用户界面的声明式 JavaScript 库，也是当前前端生态事实上的 UI 标准之一。本次快照仓库名为 `react/react`（原 `facebook/react`，org 已迁移，`gh api repos/react/react` 确认 full_name 即 react/react）。

| 项 | 值 |
| --- | --- |
| 仓库 | react/react（main 分支，HEAD 快照） |
| 星数 | 251,717（快照），查询时 251,819 |
| 主语言 | JavaScript（运行时用 Flow 类型标注，编译器为 TypeScript/Rust） |
| 许可证 | MIT（LICENSE，版权归 Meta Platforms, Inc. and affiliates） |
| 当前版本 | 19.3.0（`ReactVersions.js`，2026-09-09 发布，见 `CHANGELOG.md`） |
| 最近活跃 | 最近 push 2026-10-29（gh api），持续高频维护 |
| 仓库规模 | 运行时源码约 23.7 万行 JS，测试约 30.5 万行 JS（本仓库实测统计） |

## 2. 项目是什么，解决什么问题

React 解决的核心问题是：当应用状态频繁变化时，如何让开发者以"声明式"的方式描述任意时刻 UI 应该长什么样，而把"如何高效地把 UI 更新到目标状态"交给库本身（`README.md`：Declarative / Component-Based / Learn Once, Write Anywhere 三大卖点）。开发者写组件函数，React 负责协调（reconcile）、调度（schedule）并提交（commit）最小化的 DOM 变更。

从源码看，今天的 React 远不止是一个 DOM 库，而是一个"UI 运行时家族"：

- **客户端渲染**：`react-dom`（Web）与 `react-native-renderer`（原生）共享同一个与宿主无关的 reconciler（`packages/react-reconciler`），通过注入 HostConfig 适配不同平台（`packages/react-reconciler/src/ReactFiberConfig.js` 及十余个 `ReactFiberConfigWithNo*.js` 可选能力声明）。
- **服务端渲染**：Fizz（流式 SSR，`packages/react-server/src/ReactFizz*.js`）与 Flight（React Server Components 序列化协议，`packages/react-server/src/ReactFlightServer.js`，约 7000 行），`react-server-dom-webpack/parcel/turbopack/esm` 等包为不同打包器提供 RSC 通道。
- **并发能力**：时间切片、优先级调度、Suspense、`useTransition`/`useDeferredValue`，以及 19.3 新增的 `<ViewTransition />`、Fragment Refs、`browser()` API（`CHANGELOG.md` 19.3.0 节）。
- **React Compiler**：随 monorepo 同仓开发（`compiler/` 目录），自动记忆化组件，含 Babel 插件（TS）与 Rust 重写版 crate。

它采用三通道发布：`@latest`（稳定版）、`@canary`（下个版本的预览，带 commit sha）、`@experimental`（无版本号、带日期与 sha），由根目录 `ReactVersions.js` 作为唯一版本事实源（该文件头部注释详述三通道方案）。

## 3. 技术栈与依赖

- **语言与类型**：运行时为 JavaScript + Flow（`flow-bin ^0.317.0`，`package.json`）；React Compiler 为 TypeScript（`typescript ^5.4.3`），另有 Rust crate（`compiler/Cargo.toml`，`compiler/crates/` 下 13 个 `react_compiler_*` crate，覆盖 HIR、SSA、类型推断、优化等阶段）。
- **运行时依赖**：零。核心包 `packages/react` 无 `dependencies`（`packages/react/package.json` 实测），全部能力自包含。
- **构建**：Rollup（`rollup ^3.29.5`）+ Babel 多 preset（flow/react/typescript），构建入口 `scripts/rollup/build.js` 与 `build-all-release-channels.js`；支持 `__DEV__`/`__EXPERIMENTAL__`/`__VARIANT__`（FB www）等编译期常量多通道产物。
- **测试**：Jest 30（`jest ^30.4.1`、`jest-environment-jsdom`），配套 `jest-react`、`react-noop-renderer`（reconciler 测试用无宿主渲染器）、`dom-event-testing-library` 等自研测试设施（`packages/` 下可见）。
- **配套工具**：DevTools（`packages/react-devtools-shared`，约 4.3MB，backend/bridge/frontend 三层）与 `eslint-plugin-react-hooks` 均在同仓开发、随通道发版。
- **包数量**：`packages/` 下共 44 个包，含 9 个 react-devtools 变体（extensions/inline/core/fusebox/cdt-mcp 等）与 6 个 `react-server-dom-*` 通道包（webpack/parcel/turbopack/esm/unbundled/fb），版本统一由根 `ReactVersions.js` 管理。
- **其他工程设施**：ESLint 7 + 自研规则（`scripts/eslint-rules`）、Prettier（hermes-parser 系）、`google-closure-compiler`（产 min 包）、yarn workspaces。

## 4. 目录结构与代码组织

顶层（`ls` 实测）：

| 目录/文件 | 职责 |
| --- | --- |
| `packages/` | 全部 44 个 npm 包源码（react、react-dom、react-reconciler、scheduler、react-server、react-devtools 系列等） |
| `compiler/` | React Compiler 独立 workspace（Babel 插件、ESLint 插件、runtime、MCP server、Rust crates、自己的 CHANGELOG/CI） |
| `scripts/` | 构建（rollup）、测试（jest 封装）、错误码提取、发布自动化、flags 管理等全部工程脚本 |
| `fixtures/` | 集成夹具：dom、concurrent、fizz、flight（含 parcel/esm/ssr-bench）、packaging、view-transition 等 26 个 |
| `.github/workflows/` | 27 个 CI workflow（runtime 构建测试、fuzz、sizebot、release-from-ci、compiler rust/ts、nightly prerelease 等） |
| `ReactVersions.js` | 版本唯一事实源（latest/canary/experimental 三通道） |
| `MAINTAINERS` | 24 位核心维护者清单（acdlite、gaearon、sebmarkbage、sophiebits、josephsavona 等） |
| `flow-typed/`、`babel.config*.js`、`yarn.lock` | Flow 类型声明与构建配置 |

组织方式评述：典型的**按包拆分的 yarn workspaces monorepo**（根 `package.json` `workspaces: ["packages/*"]`；注意是 yarn 而非 pnpm，根目录为 `yarn.lock`）。其最大特点是"一套源码、多通道多平台编译"：`packages/react/src/ReactServer.js` 同名文件存在 `.fb.js`、`.experimental.js`、`.development.js` 等变体，由 rollup 构建按 release channel 与环境（node/browser/bun/edge）选择性拼装（`scripts/rollup/bundles.js` 体系）。compiler/ 是仓库内的"子 monorepo"，有独立 `package.json`、`yarn.lock`、CHANGELOG 与 CI workflow。

## 5. 核心架构与关键实现

### 5.1 Fiber 协调器：与宿主无关的工作循环

`packages/react-reconciler/src/ReactFiberWorkLoop.js`（5691 行）是渲染核心。可中断的并发渲染按时间片让出主线程，`workLoopConcurrent`（第 3065 行起）的让出策略是：Transition/Retry 类工作每 25ms 让出一次，Idle 工作每 5ms 让出，注释明确说明目的是"把动画帧率下限控制在 30fps，防止渲染饿死动画"：

```js
function workLoopConcurrent(nonIdle: boolean) {
  if (workInProgress !== null) {
    const yieldAfter = now() + (nonIdle ? 25 : 5);
    do {
      // $FlowFixMe[incompatible-type] flow doesn't know that now() is side-effect free
      if (now() >= yieldAfter) { /* yield */ }
      ...
    } while (workInProgress !== null && !shouldYield());
  }
}
```

配合 `ReactFiberRootScheduler.js`（737 行）：多 root 的同步工作统一在 microtask 中去重并 flush（`flushSyncWorkOnAllRoots`，第 171 行），这是 19.x 把调度从"每个 root 各自循环"重构为"跨 root 微任务调度"的实现。各组件类型的进入逻辑在 `ReactFiberBeginWork.js`（4508 行），diff 对比子节点在 `ReactChildFiber.js`（2240 行）。

### 5.2 Lane 优先级模型

`packages/react-reconciler/src/ReactFiberLane.js`（1304 行）用 31 位二进制位掩码表达优先级：`TotalLanes = 31`（第 40 行），`SyncLane`（同步）、`DefaultLane`、16 条 `TransitionLane`、`RetryLane` 等分段占位（第 46-94 行）。更新按 lane 合并、按最高优先级抢占、按 lane 分批完成，是并发特性（可中断渲染、transitions、hydration 重试）的底层账本。

### 5.3 Hooks 的实现位置

Hooks 逻辑集中在 `packages/react-reconciler/src/ReactFiberHooks.js`（5265 行），公开包里只有薄转发：`packages/react/src/ReactHooks.js` 仅 241 行，将 `useState`/`useEffect` 等解析到当前 dispatcher（服务端 `ReactFizzHooks` 与客户端各有一套 dispatcher 实现，这是 React 能在 SSR 与 CSR 间复用同一组件代码的机制）。diff 算法在 `packages/react-reconciler/src/ReactChildFiber.js`（2240 行）。

### 5.4 服务端双引擎：Fizz 与 Flight

`packages/react-server/src/` 同时容纳两套服务端运行时：`ReactFizz*.js`（流式 SSR，Suspense 边界可以先吐 fallback 后补内容）与 `ReactFlightServer.js`（7026 行，把组件树序列化为 RSC payload）。`packages/react-dom/src/server/` 则按运行环境出入口：Node/Bun/Edge/Browser 各有独立构建文件（`ReactDOMFizzServerNode.js` 等 8 个变体），一处协议、多环境适配。

### 5.5 React Compiler（Rust 化进行中）

`compiler/` 目录：TS 实现的 `babel-plugin-react-compiler` 自动记忆化组件；`compiler/crates/` 已有 13 个 Rust crate（`react_compiler_hir`、`react_compiler_ssa`、`react_compiler_typeinference`、`react_compiler_optimization`、`react_compiler_validation` 等），对应完整编译器流水线的 Rust 移植（`compiler/README.md`、`compiler/Cargo.toml`），并有独立发布流水线（`.github/workflows/compiler_rust_publish.yml`）。`compiler/packages/` 下另有 `react-compiler-runtime`、`react-mcp-server`、`react-compiler-healthcheck`、`snap` 等配套包（目录实测）。

## 6. 构建与运行

来自根 `package.json` scripts 与 `CONTRIBUTING.md`（后者极简，仅 5 行，指向线上贡献指南）：

```bash
yarn                        # 安装依赖（postinstall 会生成 Flow 配置）
yarn build                  # 全量构建所有 release channel 产物
yarn build react/index,react-dom/index --type=NODE_DEV   # 按包/环境选择性构建
yarn test                   # Jest 测试（scripts/jest/jest-cli.js 封装）
yarn test --watch           # 观察模式；yarn test-stable / test-www / test-classic 按通道
yarn flow                   # Flow 类型检查；yarn lint / prettier-check
yarn fixtures/dom 后: yarn && yarn test   # DOM 夹具回归（package.json test-dom-fixture）
yarn build-for-devtools      # DevTools 联调产物（另有 --dev/--prod 变体）
yarn generate-changelog      # 由 PR 记录生成 CHANGELOG（scripts/tasks/generate-changelog）
yarn extract-errors          # 提取错误码表（scripts/error-codes/）
```

注意：`yarn build` 前有 `prebuild`（`scripts/react-compiler/link-compiler.sh`）把 compiler 包链接进来。产物支持 `--type=NODE_DEV/NODE_PROD`、`RELEASE_CHANNEL=experimental` 等组合。

## 7. 测试与工程质量

- **规模**：`__tests__` 目录 545 个测试文件、约 30.5 万行测试代码（实测统计），超过 23.7 万行运行时源码；仅 react-reconciler 就有 77 个测试文件。
- **风格**：以行为级单元测试为主，大量使用 `react-noop-renderer`（无宿主渲染器直接断言 fiber 行为）与 `react-test-renderer`；`fixtures/dom` 做真实浏览器行为回归，`fixtures/flight-*`、`fixtures/packaging` 验证打包器兼容性。
- **口径说明**：以上统计为 `packages/` 下排除 `__tests__` 与 npm 构建产物目录后的 `*.js`；`compiler/` 的 TypeScript 与 Rust 代码（364 个 TS 源文件）未计入。
- **CI**：27 个 workflow（`.github/workflows/`），除常规构建测试外还有 `runtime_fuzz_tests.yml`（模糊测试）、`runtime_sizebot_comment.yml`（PR 上自动报包体积变化）、`runtime_release_from_ci.yml`（CI 内自动发版）、`shared_lint.yml`。
- **依赖防退化**：`fixtures/dom` 维护一份 attribute 行为基准（`fixtures/attribute-behavior` 记录各浏览器属性行为），防止 DOM 语义悄悄改变。
- **质量观察**：错误码集中提取（`scripts/error-codes/`，生产包只保留错误码、由 URL 反查文案）；Flow 全量类型标注；ESLint 自研内部规则强制禁止 `for-of`、限制属性访问等历史兼容约束。

## 8. 社区与维护状态

- **活跃度**：极高。gh api 显示最近 push 为 2026-10-29（前一天），工作日基本每日多 commit。
- **发版节奏**：stable 大版本数月一次（19.3.0 于 2026-09-09，`CHANGELOG.md`），canary 通道近乎每日随 main 产 nightly（`compiler_prereleases_nightly.yml`、`runtime_prereleases_*.yml`）。canary 版本号形如 `<version>-canary-<sha>`，供 Next.js 等框架跟踪 main 进展（`ReactVersions.js` 头部注释）。
- **贡献者结构**：根目录 `MAINTAINERS` 列出 24 位核心维护者（多为 Meta 员工）；CHANGELOG 中 PR 作者高度集中在核心团队，外部贡献以修复类 PR 为主——Meta 主导开发的格局未变。
- **当前演进方向**（CHANGELOG 19.3.0 与源码新文件）：View Transitions、Fragment Refs、`<Activity>` 隐藏树、手势调度（`ReactFiberGestureScheduler.js`）、Trusted Types 集成、`react-markup` 静态渲染等。
- **仓库迁移**：`facebook/react` 已更名/迁移为 `react/react`（gh api 确认 full_name，旧名自动重定向），README 徽章与 CHANGELOG 链接均指向 `github.com/react/react`。

## 9. 亮点与风险

**亮点**

1. 单一 reconciler 内核服务 DOM、Native、ART、Flight 多平台（`ReactFiberConfig.js` 注入机制），架构解耦教科书级示范。
2. Lane 位掩码优先级 + 时间切片（25ms/5ms 让出策略）+ 跨 root 微任务调度，构成业界最精细的前端调度系统。
3. 核心零运行时依赖，从一套源码编译出 stable/canary/experimental × node/browser/bun/edge × dev/prod 的全部产物，工程化能力极强。
4. 测试投入罕见：30 万行测试 + fuzz 测试 + sizebot 体积门禁 + 26 个打包/环境夹具。
5. 仓库内自带 React Compiler（TS + Rust 双实现），"库 + 编译器"一体推进。

**风险/局限**

1. 核心文件巨大（WorkLoop 5691 行、Hooks 5265 行、FlightServer 7026 行），加上 `.fb.js`/`.experimental.js` 文件变体与编译期常量分叉，直接阅读源码门槛很高。
2. 运行时类型用 Flow 而非 TypeScript，社区主流已是 TS，外部贡献者需要适应 Flow（`flow-typed/`）。
3. canary/experimental API 演进快（如 19.3 新增的 `browser()`、Fragment Refs），依赖新特性的项目面临 API 不稳定风险。
4. 实际开发决策集中在 Meta 内部（www 分支变体、`.fb.js`），开源仓库并非全部开发场景的完整镜像。

## 10. 总结评价

React 是前端领域工程化、架构化程度最高的开源项目之一：一个与宿主无关的 Fiber 内核，向上支撑 Web、Native 与服务端渲染三大场景，向下通过 Lane 优先级与时间切片实现细粒度并发调度，仓库内还孵化着 Rust 版 React Compiler。它 25 万+ 星的地位来自"声明式编程模型 + 长期稳定的 API + Meta 级别的持续投入"。想深入理解并发渲染与调度机制的工程师，`packages/react-reconciler`（建议从 `ReactFiberWorkLoop.js`、`ReactFiberLane.js` 入手）与 `packages/scheduler` 是必读源码；想研究编译器/自动记忆化的，则看 `compiler/` 目录。对普通使用者而言，它仍是构建 Web UI 的默认安全选择。
