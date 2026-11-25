# motiondivision/motion 项目分析

> 星数 35,413（gh api 实测 2026-11-21；任务快照 28,000）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：Motion（前身 Framer Motion）是 React / 原生 JS / Vue 通用的动画库，以"混合引擎"（JS 驱动 + 浏览器原生 WAAPI）著称，Framer 出品后迁至独立组织 motiondivision 并更名。

| 事实 | 值 | 出处 |
|---|---|---|
| 名称/仓库 | motiondivision/motion（framer/motion 302 重定向至此） | gh api repos/motiondivision/motion |
| 星数 | 35,413（快照后仍在增长） | gh api，2026-11-21 |
| 语言 | TypeScript（纯 TS，无 JS 源码） | packages/*/src |
| 许可证 | MIT | LICENSE.md、package.json |
| 创建时间 | 2018-11-16（framer org 时代） | gh api created_at |
| 最近 push | 2026-11-20（分析前一天） | gh api pushed_at |
| 当前版本 | 14.14.0（2026-11-13） | packages/motion/package.json、CHANGELOG.md |
| 官网/文档 | motion.dev（独立仓库，不在本仓内） | README.md、scripts/push-to-site.js |
| 发布渠道 | npm（lerna publish），无 GitHub Release 对象 | package.json scripts、gh api releases 为空 |

## 2. 项目是什么，解决什么问题

Motion 解决"让 Web 界面动起来"这件事的三层痛点。第一是 API 层：React 用户用声明式的 `<motion.div animate={{ x: 100 }} />`（`packages/framer-motion/src/index.ts` 导出的组件族），原生 JS 用户用命令式的 `animate("#box", { x: 100 })`（`packages/framer-motion/src/dom.ts`），Vue 走独立的 motion-v 包（README.md）。第二是物理层：内置 spring 弹簧物理、惯性（inertia）、手势（拖拽/捏合/悬停/按压）、布局动画（layout projection，元素位置尺寸变化时自动补间）、滚动联动与时间线，这些都是"开箱即用"而非插件拼装。第三是性能层：宣称的混合引擎按值类型选择动画执行路径——可硬件加速的属性（opacity/transform/filter/clipPath）交给 WAAPI 在合成器线程跑，其余用 JS 在 rAF 帧环里逐帧计算，见 `packages/motion-dom/src/animation/waapi/utils/accelerated-values.ts`。

项目的演进史本身就是动画库生态的一个切片：2018 年以 framer/motion 起家（gh api created_at），2019 年 2 月 CHANGELOG 记录到 0.3.0（`CHANGELOG.md` 最早条目）；2021-05-19 motiondivision 组织创建；此后仓库从 framer org 转移到 motiondivision（`gh api repos/framer/motion` 302 重定向证实），README 明确写有"Framer Motion is now Motion. Import from `motion/react` instead of `framer-motion`"（README.md）。2025-01 的 v12.0.0（CHANGELOG.md 第 1713 行）吸收了原独立的 Motion One 项目（"Replaced Motion One"），把 `motion/react`、`motion`（JS DOM API）、`motion/mini` 等子路径统一到一个 npm 包下。

商业化与开源的关系是"开源核心 + 付费增值"：MIT 开源库本体在本仓，motion.dev 上的 Motion+（450+ 示例、110+ 教程、Cursor/Ticker 等 Premium API、MotionScore 审计）单独售卖，README 有完整的赞助体系（Partners：Framer、Cursor、Figma；Platinum：Linear/Sanity/Clerk）。Framer 仍是深度合作方——"Motion powers the animations for all websites built with Framer"（README.md），且 Makefile 头部注释写着"This is coming from our common.mk in the FramerStudio repo"，说明构建体系仍与 Framer 内部共享。

## 3. 技术栈与依赖

- **语言/运行时**：TypeScript 5.4.3，React 18/19 双支持（根 package.json 有 `react-19-env` overrides，dev/ 下有 react-19 独立开发环境）。
- **monorepo 工具链**：Yarn 3.6.4 workspaces + Turborepo（turbo.json）+ Lerna 4（仅用于 publish），根 package.json name 为 "motion-one"（Motion One 遗留名）。
- **构建**：Rollup 4（含 rollup-plugin-dts 生成类型、rollup-plugin-preserve-directives 保留 "use client" 指令、terser），每个包 `tsc` 出类型 + rollup 出产物。
- **测试**：Jest 29（jsdom 单测）、Cypress 4（React 浏览器 e2e）、Playwright（vanilla JS 浏览器 e2e）、react-testing-library。
- **性能基准对手**：devDependencies 直接装了 gsap、@gsap/react、animejs 3（`packages/*/rollup.size.config.mjs` + dev/inc/bundlesize.mjs 用于包体积对比与预算管控）。
- **运行时依赖极轻**：framer-motion 仅依赖 motion-dom、motion-utils、tslib；peerDependencies 只有 react/react-dom/remix（packages/framer-motion/package.json）。
- **AI 工具链**：.github/workflows/claude.yml（@claude 驱动 PR/issue 自动化）、AGENTS.md/CLAUDE.md 双 agent 说明、skills-lock.json 锁定 shadcn/improve skill。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| packages/motion-utils | 纯函数层：easing、clamp、subscription-manager、warn-once 等，零依赖，972 行 |
| packages/motion-dom | 引擎核心：动画、帧环、手势、布局投影、渲染效果、MotionValue，53k 行 |
| packages/framer-motion | React 绑定层 + 历史入口（./dom、./mini、./m、./projection），50.6k 行 |
| packages/motion | 新主门面包：./react、./mini、./three、./vgpu、./remix 等子路径，全部薄再导出，仅 2.4k 行 |
| packages/config | 私有共享 tsconfig |
| dev/ | 六个开发沙盒：html(vite)、react(vite)、react-19、next、remix、inc(体积测量工具) |
| tests/ | Playwright e2e 顶层目录（animate/gestures/layout/scroll/view 等 9 类） |
| plans/ | 33 个编号开发计划 + PERFORMANCE_AUDIT.md + issues/，AI 协作的任务台账 |
| scripts/ | changelog CSV 生成、推送 motion.dev 站点、Slack 发布通知 |
| AGENTS.md / CLAUDE.md | 面向编码 agent 的仓库说明（包结构、三类测试约定、命令） |
| playwright.config.ts | 浏览器 e2e 配置（dev/html 作为静态站点伺服） |
| lerna.json / turbo.json | 发布编排与构建任务编排 |
| FUNDING.json | 以 drips/ethereum 方式接收资助 |
| Makefile | 继承自 FramerStudio 内部 common.mk 的构建入口（bootstrap/build/clean） |

组织方式是"四层依赖塔"：`motion → framer-motion → motion-dom → motion-utils`（各 package.json dependencies 实测）。最有信息量的设计在于 npm 包与内部包的不对称：发布给用户的 `motion` 包其实是个门面——`packages/motion/src/index.ts` 只有一行 `export * from "framer-motion/dom"`（默认入口是 JS 动画 API 而非 React），`react.ts` 再导出 framer-motion 全量。这保证了 `framer-motion`（存量用户包名）与 `motion`（新包名）版本永远同步（同为 14.14.0），同时让"无 React 依赖的 DOM 引擎"（motion-dom）成为真正的可复用底座。AGENTS.md 明确写下重构方向："framer-motion: For React-specific code. More of this should be refactored to motion-dom where possible."——即把框架无关逻辑持续下沉到 motion-dom。

## 5. 核心架构与关键实现

### 5.1 帧环（frameloop）：8 步渲染管线

`packages/motion-dom/src/frameloop/order.ts` 定义了每帧的固定阶段：`setup → read → resolveKeyframes → preUpdate → update → preRender → render → postRender`，读写在阶段间隔离以避免布局抖动。每个阶段是一个"双 Set 队列"步骤（`frameloop/render-step.ts`）：thisFrame/nextFrame 两个 Set 逐帧交换以避免 GC，支持 keepAlive 进程与 `immediate` 插队语义，且递归 flush（flushSync 场景）标记到下一帧防死循环。需要同步 DOM 时 `frameloop/flush-frame.ts` 可只跑 update/preRender/render 三步。

### 5.2 MotionValue：响应式值的订阅模型

`packages/motion-dom/src/value/index.ts` 中的 MotionValue 是一切动画的载体：持有 latest 值 + 时间戳，维护 Subscriber 订阅（SubscriptionManager 来自 motion-utils），做基于帧间隔的被动速度估计（MAX_VELOCITY_DELTA=30ms），派生值（useTransform/transform-value.ts、spring-value.ts、follow-value.ts、map-value.ts）构成值依赖图。plans/012 号计划（motion-value-derivation-graph-spike.md）显示团队正在评估把派生统一为 mark-dirty/pull-compute 图——演进方向有据可查。

### 5.3 双路径动画执行：JSAnimation vs NativeAnimation

动画生成器（`animation/generators/`：spring.ts、inertia.ts、keyframes.ts）产出逐帧采样函数；执行时二选一：

- **JS 路径**：`animation/JSAnimation.ts`（142 行）+ `MultiValueAnimation.ts`，rAF 驱动逐帧采样 generator，支持任意值类型与 spring 分析导数（getGeneratorVelocity）。
- **WAAPI 路径**：`animation/NativeAnimation.ts`（285 行），把 generator 预生成 keyframes 后交给 `element.animate()`，合成器线程运行；`effects/style/output.ts` 提供 `animateStyle` 与 WebKit 兼容的 `createStyleAnimation`（先建 KeyframeEffect 再 play，规避 WebKit ready-time 跳变，注释引 #4034）。

路径选择的核心约束在 `waapi/utils/accelerated-values.ts`：仅 `opacity, clipPath, filter, transform` 四类可走 WAAPI，注释解释颜色被排除是因为"WAAPI 在 sRGB 混色而 JS 动画（和 WAAPI 中断采样器）在线性色彩空间混色，中断时会可见跳变"。spring 是"预生成再交给 WAAPI"的：`generators/spring.ts` 用牛顿迭代解过阻尼频率缩放（overdampedFreqScale），支持 stiffness/damping/mass 物理参数与 duration/bounce 视觉参数（对齐 SwiftUI）双语法。

### 5.4 React 绑定层：Proxy 组件 + 特性包注入

`packages/framer-motion/src/render/components/create-proxy.ts` 用 ES Proxy 实现 `motion.div` 的惰性缓存工厂（未命中才 createMotionComponent，缓存进 Map），`m` 组件（`render/components/m/proxy.ts`）是同一工厂但不带预载特性包——配套 `LazyMotion` 组件可在运行时注入 `domAnimation`/`domMax` 特性包（`render/dom/features-max.ts`：domMax = domAnimation + drag + layout），实现"按需加载拖拽/布局能力"。`motion/index.tsx` 的组件核心串联 React context 体系（MotionContext/PresenceContext/LayoutGroupContext 等 11 个 context），AnimatePresence（components/AnimatePresence/）处理退出动画与 popChild 测量。React 18/19 兼容通过根 package.json overrides 与 dev/react-19 沙盒双轨测试。

### 5.5 手势系统

手势在 motion-dom 有完整实现（`packages/motion-dom/src/gestures/`：press/、hover.ts、pan/、pinch/、drag/），React 层的 `packages/framer-motion/src/gestures/` 做生命周期绑定。近期演进密集：v14.11.0 新增多指捏合旋转（CHANGELOG.md），v14.11.1 给捏合吸附点加了基于速度的预测旋转定位，v14.10+ 修复捏合中拖拽打断动画的问题——手势子系统处于活跃迭代期。plans/019-024 一串编号计划（drag-pan 引擎移植、vanilla drag 共享投影树、press 键盘监听生命周期、hover 双 start 过滤）显示手势正在从 React 专属向 motion-dom 全面下沉。

### 5.6 布局投影与边界能力

布局动画的投影树下沉到了 motion-dom：`projection/node/`（DocumentProjectionNode/HTMLProjectionNode/create-projection-node）、`layout/LayoutAnimationBuilder.ts`（命令式 animateLayout，全树投影节点持久化于 WeakMap，同一 tick 的 builder 合并成单次"commit"快照）。门面包还实验性地暴露了 `motion/three`（MotionValue 绑定 Three.js uniforms/材质，`packages/motion/src/three.ts`）与 `motion/vgpu`（WebGPU 效果，rgba/hsla 直写 GPU 属性）。

## 6. 构建与运行

来自 CLAUDE.md / 根 package.json / Makefile 的实际命令：

```bash
make bootstrap            # 首次安装（Makefile 管理 node_modules 增量更新）
yarn build                # turbo run build，全包构建
yarn watch                # 全包 watch 模式
yarn test                 # Jest 单测 + typecheck（仅 packages/*）
yarn test-playwright      # 构建 + playwright e2e（tests/）
make test-e2e             # 全部 e2e（Next/HTML/React/React19/Playwright）
yarn measure              # 构建并跑 bundlesize 预算（dev/inc/bundlesize.mjs）
yarn new                  # 发布：changelog CSV + playwright + lerna publish from-package
yarn new-alpha            # lerna canary alpha 发布
# 单包测试（CLAUDE.md）：cd packages/framer-motion && yarn test-client / yarn test-server
make test-single          # 跑单个 Cypress 用例（Makefile 内改 spec 路径）
yarn size                 # dev/inc/size-scenarios.mjs，体积场景对比
```

不提供任务规范之外的运行结论；本分析未执行构建。

## 7. 测试与工程质量

- **三层测试**（AGENTS.md 明文约定）：Jest 单测与源码同目录 `__tests__/` 共 289 个 .test.ts 文件；Cypress 127 个 spec 测 React 浏览器行为（packages/framer-motion/cypress/integration）；Playwright 101 个 .ts spec 测 vanilla JS（tests/，按 animate/gestures/layout/scroll/view 分类）。
- **体积预算即测试**：每个发布包的 package.json 内嵌 bundlesize 数组，如 framer-motion 的 `size-rollup-motion.js ≤ 46.9 kB`、`size-rollup-scroll.js ≤ 3.75 kB`、`size-rollup-waapi-animate.js ≤ 3.2 kB`，motion-dom 的 style-effect ≤ 4.35 kB——超预算会挂，且 plans/038（motion-dom-filesize-audit-pass）显示预算被主动审计下调。
- **CI 的特殊形态**：.github/workflows 只有 claude.yml（GitHub 上没有传统 CI yml），正式 CI 在 Framer 内部（Makefile 承接 FramerStudio common.mk 也佐证）；质量门禁由本地 `yarn test` + bundlesize + plans/007（ci-lint-gate）计划描述的 lint gate 承担。
- **文档内嵌**：核心文件注释密度高、且习惯引用 issue 号（如 NativeAnimation 的 WebKit #4034、react.ts 的 Turbopack OOM #3741），历史决策可追溯。

## 8. 社区与维护状态

- **极活跃**：gh api 统计近一年 main 分支约 2,277 个 commit（2025-11-21 至今 per_page=1 翻页 page=2277），最近 commit 在分析前一天；当前仅 3 个 open PR、3 个 open issue——问题消化速度极快。
- **体量与流行度**：1,581 fork（gh api）；README 挂 framer-motion 每月 npm 下载徽章与 jsDelivr 月命中徽章（具体数值以 npm 官方数据为准，本分析未请求 npm API）。
- **贡献者**：contributors API 首页满 100 人，per_page=1 翻页显示约 133 位贡献者。根 package.json author 为 Matt Perry（核心作者，2018 年至今主导）。
- **发版节奏**：无 GitHub Release（gh api releases 为空），发布走 lerna + npm + Slack 通知（scripts/notify-slack.js）；CHANGELOG.md 记录 890 个版本条目，从 0.3.0（2019-02-05）到 14.14.0（2026-11-13），近期每周 1-4 个版本。
- **AI 原生开发是显著特征**：近期大量 merge commit 的作者署名为 "Claude"（如 #4096/#4107/#4098 等 PR 分支名 claude/project-thread-*），仓库以 AGENTS.md + CLAUDE.md 约束 agent 行为，plans/ 目录用编号计划文件（含 Priority/Effort/Risk/依赖与 STOP conditions）驱动任务分解，33 个计划 + PERFORMANCE_AUDIT.md 构成公开的工程台账。这是"单人主导 + AI 放大产能"模式的少见公开样本。
- **迁移历史（客观记录）**：仓库 2018-11-16 创建于 framer org；motiondivision org 2021-05-19 创建；仓库现 302 重定向自 framer/motion；组织成员列表 API 返回空（成员公开性关闭），迁移的具体公告时间未在本仓内找到文字记录（标注：未确认）。README 的赞助结构表明 Framer 仍以 Partner 身份持续投入。
- **文档站独立**：motion.dev 不在本仓，scripts/push-to-site.js 将 changelog.csv 镜像到本地 `~/Sites/motion-api`（站点源码仓目录名），docs 与代码库分离。

## 9. 亮点与风险

**亮点**

1. 架构分层干净且持续演进：utils→dom→React→门面的单向依赖塔（各包 package.json 可证），AGENTS.md 公开写下"React 无关逻辑下沉 motion-dom"的重构方向，motion 包作为门面使包名迁移零成本。
2. 混合引擎的取舍有明确工程依据：WAAPI 白名单仅四个合成器友好属性，连"颜色不进 WAAPI"都给出了色彩空间中断跳变的具体原因（accelerated-values.ts 注释）。
3. 把包体积当测试管：bundlesize 预算逐导出路径设定（最小 1.9 kB），并配备对 GSAP/animejs 的基准依赖。
4. 测试金字塔完整（289 单测 + 127 Cypress + 101 Playwright）且 React 18/19 双轨。
5. 框架边界不断外扩且有独立入口承载：`motion/remix`（framer-motion/src/remix.ts 重导出投影/布局工具）、`motion/three`、`motion/vgpu` 均为独立子路径，不污染主入口体积。
5. AI 原生工程实践公开可审计：AGENTS.md/CLAUDE.md/plans/ 台账 + 署名透明的 AI 提交。

**风险/局限**

1. 单人主导风险：架构决策与发布高度集中于 Matt Perry（author 字段、commit 模式），133 位贡献者的实际合并权集中在极少数维护者手中（未确认具体比例）。
2. 传统开源 CI 缺位：GitHub 上无 lint/test workflow，质量门禁依赖 Framer 内部设施，外部贡献者只能在本地验证。
3. 版本演进激进：12.0 吸收 Motion One、14.x 仍在大改内部结构（大量内部导出供 Motion UI 使用，CHANGELOG 14.14.0），下游若依赖内部 API 会频繁破坏。
4. 271MB 的 tarball 体量偏大（含 dev 沙盒与测试夹具），浅层使用者克隆成本高。

## 10. 总结评价

Motion 是 TypeScript 动画库里工程成熟度最高的一档：API 面覆盖 React/JS/Vue 三端，引擎层在"物理仿真精度"与"合成器加速"之间给出了有注释背书的工程取舍，React 绑定层用 Proxy + 特性包把按需加载做到了导出路径级别。它同时是观察"商业公司开源 + 单人专家主导 + AI agent 放大产能"的罕见样本——开源核心 MIT、增值内容（Motion+/Studio）商业售卖、Framer 以 Partner 而非所有者身份存续、AI 提交署名公开。适合需要 production 级动画方案的 React 团队直接采用；适合引擎研究者精读 motion-dom 的 frameloop/generators/effects 三件套（这才是库的真正心脏，React 层只是壳）；对维护者与 agent 工程实践感兴趣的人，plans/ 目录本身就是一份可复用的任务分解模板。读源码建议路径：frameloop/order.ts → value/index.ts → animation/generators/spring.ts → animation/NativeAnimation.ts → render/components/create-proxy.ts。
