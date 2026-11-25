# videojs/video.js 项目分析

> 星数 40180（快照 2026-11-14，任务派发时为 39500）｜主语言 JavaScript（TypeScript）｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

全球最知名的开源网页视频播放器框架。**注意：main 分支已不再是经典的 Video.js 7/8，而是 2026-10 发布的 Video.js 10 —— 一次彻底的 monorepo 重构**，定位从"HTML5 & Flash video player"变为"Modern, modular, and composable media player framework for Web and React"（`README.md`、GitHub repo description）。

| 事实 | 值 | 出处 |
|---|---|---|
| 名称 | videojs/video.js（monorepo 名 `@videojs/monorepo`） | `package.json` |
| 星数 | 40180（2026-11-14 gh api） | GitHub API |
| 创建时间 | 2010-05-14（15 年历史） | GitHub API |
| 许可证 | Apache-2.0（`LICENSE` 含 Brightcove 商标声明，由 TSC 治理） | `LICENSE` |
| 最近 push | 2026-11-13（前一天） | GitHub API |
| 最新版本 | v10.4.0（2026-11-13 发布） | `CHANGELOG.md` / releases |
| 贡献者 | 367 位（contributors API 分页推算） | GitHub API |
| 语言 | TypeScript（全仓），发布为 JS | `packages/**/*.ts` |

## 2. 项目是什么，解决什么问题

Video.js 自 2010 年起解决的核心问题是：浏览器原生 `<video>` 在不同浏览器下外观、行为、格式支持不一致，且缺乏统一的播放器 UI 与扩展模型。经典版本（7.x/8.x）通过"自定义 DOM 皮肤 + 统一 Player API + Tech 抽象层"成为事实标准，npm 周下载长期在千万级。Flash 时代由 videojs-flash 承担，随 Flash 淘汰已彻底移除（当前 main 分支无任何 Flash 代码，adapters 目录可证）。

2026-10-06 发布的 v10.0.0 是一次推倒重来（release API、`rfc/player-api.md`）。设计动因记录在 `rfc/player-api.md`：旧版"用例预设作为主抽象"导致 API 不透明、能力不可树摇、React 类型推断差。v10 改为 **Feature（特征）作为组合单元**：每个能力（播放、音量、字幕、全屏……）是一个携带状态与 action 的 slice，可单独导入、可 tree-shake，React 与 Custom Element 两条平台路径共享同一状态图。

适配引擎的模型也换了：旧版 Tech 抽象（HTML5/Flash/VHS 都挂在 Player 里）被 **adapter 包**替代 —— 每个播放引擎/嵌入一个独立 npm 包（hls.js、DASH、Shaka、YouTube、Twitch、Vimeo 等 18 个），核心包零 DOM 依赖、零引擎依赖（`AGENTS.md` 仓库地图）。HLS/DASH 能力由 `@videojs/spf`（自研流处理引擎，约 2.7 万行）加 hlsjs/dash/shaka 适配器承担。

迁移策略务实：`video.js` 包本身（10.4.0）变成"指针包"，指向 `@videojs/html` 与 `@videojs/react`；老用户按 v8 写法调 `videojs('my-video')`、`registerPlugin` 等会触发带错误码（`VJS8_LEGACY_*`）的抛错，引导到专门的错误文档页（`packages/video.js/src/videojs.ts`）。经典 8.x 代码仍在 `8.x` 分支维护（branches API），5.x~7.x 分支也保留。

## 3. 技术栈与依赖

- **语言**：全仓 TypeScript（`tsconfig.base.json` + 各包 tsconfig），ESM（`"type": "module"`，根 `package.json`）
- **构建**：Vite + vite-plus（`@voidzero-dev/vite-plus-core` 作为统一任务运行器，`vite.config.ts`、catalog `pnpm-workspace.yaml`）；CSS 用 Tailwind v4 + lightningcss（catalog 注释：`@videojs/vjsc` 用同一引擎编译皮肤 CSS）
- **包管理**：pnpm workspace（`pnpm-workspace.yaml`，含 `minimumReleaseAge` 供应链防护与 trustPolicy）
- **测试**：Vitest 5（含 browser-playwright 模式）+ Playwright e2e（含 axe 无障碍测试、BrowserStack、skin-parity 视觉对比，`apps/e2e/package.json`）
- **Lint**：oxlint（`tools/oxlint/`、`lint:anti-slop` 脚本）
- **文档站**：Astro + Netlify（`site/`、pnpm-workspace 注释提及 resvg/musl 细节）
- **发布**：release-please + npm trusted publishing（`.github/workflows/cd.yml`）
- **运行时依赖极少**：适配器各自引入引擎（如 hls.js），核心链路自研；`@svta/cml-*`（Common Media Library，SVTA 标准）出现在 peer 规则中（`pnpm-workspace.yaml`）

## 4. 目录结构与代码组织

pnpm monorepo，`packages/*` + `apps/*` + `site`（`pnpm-workspace.yaml`）。

| 目录 | 职责（依据 `AGENTS.md` 仓库地图 + 实际 ls） |
|---|---|
| `packages/core` | 运行时无关播放器逻辑：feature 定义、i18n（55 种语言 locale 文件）、text-track 契约（约 121 行的 `player.ts` + 类型系统） |
| `packages/media` | 媒体契约与状态；浏览器 host 与第三方引擎适配在 `/dom` |
| `packages/store` | 框架无关状态库：slice/store/selector（核心约 10 个文件），带 `/html` `/react` 绑定 |
| `packages/spf` | 自研流处理引擎：signals 响应式原语、网络层（带宽估计/EWMA/重试）、ABR、buffer、DRM、HLS/MP4/MPEG-TS 解析（约 2.7 万行） |
| `packages/html` | DOM 平台播放器：createPlayer 工厂、custom element 基建、42 个 UI 组件目录（slider/menu/popover/hotkey/gesture 等，约 1.4 万行） |
| `packages/react` | React 平台播放器：hooks、provider、text-track hooks（CHANGELOG 佐证） |
| `packages/element` | custom-element 基类 |
| `packages/adapters/*` | 18 个引擎适配器包：hlsjs、dash、shaka、native-hls、mux、youtube、twitch、vimeo、wistia、tiktok、spotify、soundcloud、dailymotion、mixcloud、cloudflare、playerjs、mux-audio 等 |
| `packages/extensions/*` | 播放器扩展：google-cast、google-analytics、mux-data |
| `packages/skins` | 皮肤系统，含 shadcn 兼容构建（`build:shadcn`、`test:shadcn` 脚本） |
| `packages/video.js` | 经典 `video.js` npm 名的迁移指针包 + v8 legacy 错误码 stub |
| `packages/vjsc` / `icons` / `installation` / `utils` / `cdn` / `cli` | CSS 编译器、图标、安装 schema/代码生成、工具、CDN 构建产物、CLI |
| `apps/e2e` | Playwright 套件（player/skin-parity/sandbox/browserstack/drm） |
| `apps/react-native` | React Native 播放器 POC（RFC `react-native-player-api.md`、PR #9551） |
| `apps/sandbox` | Vite 手play 场 |
| `rfc/` + `internal/` | 设计 RFC 与架构决策记录（player-api.md 标注 status: implemented） |
| `.agents/` + `AGENTS.md` + `CLAUDE.md` | AI 辅助开发规范与 agent skills（20+ 个） |

组织方式评述：严格的分层 monorepo —— "框架无关（core/store/spf/media）不依赖 DOM 与框架，平台行为放 adapter/binding 包"是写进 AGENTS.md 的硬规则。包粒度极细（30+ 包），统一版本号发布。

## 5. 核心架构与关键实现

### 5.1 Feature-Slice 状态模型（替代旧版 Component 体系）

`packages/core/src/core/player.ts` 定义了核心抽象：`PlayerFeature` = store 的 `Slice`（挂到 target 的状态切片）+ 可选 `config`（HTML 属性到 action 的映射）。配套类型 `PlayerFeatureConfig` 用条件类型保证"HTML 属性只能传文本，因此映射的 action 必须接受字符串/null"（player.ts 中 `ConfigActionKey` 的条件类型推断）。RFC 明确决策："Features are the unit of composition"、"One store owns one composite player target"、"State access is selector-based"（`rfc/player-api.md`）。

```ts
// packages/core/src/core/player.ts
export type PlayerFeature<
  State, Derived = object,
  Config extends PlayerFeatureConfig = Record<never, never>,
  Target = any,
> = Slice<Target, State, Derived> & {
  config?: Config;
};
```

### 5.2 store：细粒度状态库

`packages/store/src/core/slice.ts` 中 Slice 的 attach 上下文提供 `AbortSignal` 注册表：`signals.base`（detach 时中止）、`signals.supersede(key)`（新 seek 中止旧 seek）、`signals.clear()`（换源清空）—— 用标准 Web API 解决异步生命周期，是整个播放器异步安全的地基（`packages/store/src/core/slice.ts`、`abort-controller-registry.ts`）。

### 5.3 spf：自研流处理引擎（VHS 的继任者）

`packages/spf/src/` 约 2.67 万行，含：响应式 signals（`core/signals/primitives.ts`，`signal/untrack/update/effect` 原语，Solid.js 风格）；状态机基建（`core/machine.ts`，Actor 消息驱动 + Reactor 信号驱动两种原语共享同一 snapshot-signal 核心）；网络层（`network/ewma.ts`、`bandwidth-estimator.ts`、`retry.ts`）；媒体层（`media/abr/`、`buffer/`、`mp4/`、`mpeg-ts/`、`hls/` 播放列表解析、Apple HLS JSON 章节解析）。DRM 做成可组合的 key-system 模块（Widevine/PlayReady/FairPlay 各自独立导出以保 tree-shaking，`spf/src/hls.ts` 顶部注释详述）。错误词汇采用 SVTA 2070 标准（`media/errors.ts`）。

```ts
// packages/spf/src/core/machine.ts
export function createMachineCore<FullState extends string, Snapshot extends MachineSnapshot<FullState>>(
  initialSnapshot: Snapshot
) {
  const snapshotSignal = signal(initialSnapshot);
  const getState = (): FullState => untrack(() => snapshotSignal.get().value);
  const transition = (to: FullState): void => update(snapshotSignal, (current) => ({ ...current, value: to }));
  return { snapshotSignal, getState, transition };
}
```

### 5.4 双平台播放器工厂与 UI 组件

`packages/html/src/player/create-player.ts` 用函数重载提供 `createPlayer({ features: videoFeatures })`，返回类型化的 `PlayerElement`（custom element）+ `PlayerController`（UI 控件绑定用，`new PlayerController(this, selectPlayback)`），feature 元组驱动 TypeScript 推断。UI 是 42 个 custom element 组件目录（`packages/html/src/ui/`：slider、menu、popover、hotkey、gesture、buffering-indicator、captions-radio-group 等），基类 `media-ui-element.ts` + `PlayerController` 选择器订阅模式。React 侧（`packages/react/src/`）有平行的 player/presets/ui/text-track 结构。

### 5.5 适配器模式与迁移兼容层

每个引擎一个包，如 `packages/adapters/hlsjs-video/src/` 拆为 adapter.ts、cap-level、chapters、drm、live、media-tracks、text-tracks 等职责文件。旧 v8 API 兼容：`packages/video.js/src/videojs.ts` 将 `videojs()`、`registerPlugin`、`registerComponent` 等全部 stub 化，调用即抛 `VJS8_LEGACY_PLUGIN` 等错误码，注释明确说"与其让用户看到 undefined is not a function，不如给一个只能导向错误页的代码"。

## 6. 构建与运行

来自根 `package.json` scripts 与 `AGENTS.md`/`CONTRIBUTING.md`：

```bash
pnpm install          # pnpm >= 12.3.4，Node 由 packageManager/mise 管理
pnpm dev              # 全 workspace watch 开发
pnpm build            # 递归构建全部包
pnpm test             # 各包 test:ci（Vitest）
pnpm -F <pkg> test    # 单包测试
pnpm test:e2e         # Playwright（test:all = player + skin-parity + sandbox）
pnpm test:e2e:drm     # DRM e2e
pnpm typecheck        # workspace 类型检查
pnpm lint             # vite-plus lint（另有 lint:anti-slop）
pnpm size             # bundle 体积检查（.github/scripts/bundle-size.js）
npx @videojs/cli agents init   # 给 AI agent 输出版本匹配的安装指引（README）
```

## 7. 测试与工程质量

- **规模**：700 个测试文件，测试代码约 4.07 万行 vs 生产代码约 5.17 万行（不含 i18n locale 与测试，`find|wc -l` 统计），比例约 0.8:1，属高覆盖投入
- **分层**：单测与源码同目录（`*/tests/` colocated，AGENTS.md 规定"current behavior: implementation and colocated tests"）；Vitest browser-playwright 模式跑真实浏览器；e2e 五类套件（player、skin-parity、sandbox、registry、drm）+ BrowserStack + axe 无障碍（`apps/e2e/package.json`）
- **CI**：17 个 workflow（`.github/workflows/`）：ci.yml 按 PR 变更文件智能选择受影响包的测试矩阵（`package-test-matrix.js`）；bundle-size.yml、browserstack.yml、pkg-pr-new.yml（每个 PR 产出可安装预览包）、issue-triage/issue-to-pr 等
- **质量机制**：commitlint + git-cliff 生成 CHANGELOG（`cliff.toml`）、cspell 拼写检查、workspace 一致性校验（`build/scripts/check-workspace.mjs`）、npm 供应链防护（minimumReleaseAge=1440 分钟）

## 8. 社区与维护状态

- **极度活跃**：最近 push 2026-11-13（分析前一天）；v10.0.0（2026-10-06）后一个月内连发 10.1.0 → 10.4.0，全仓 30+ 包统一版本号同步发布（releases API 显示 2026-11-13 一天发布全部 10.4.0 包）
- **治理**：Brightcove 持有商标、开源社区 + Technical Steering Committee 治理（`LICENSE` 商标声明）；核心贡献者含 heff（创始人）、luwes、mihar-22、sampotts（Plyr 作者）、cjpillsbury 等（CHANGELOG 署名）
- **贡献者**：367 位（contributors API）
- **AI 辅助开发是显性工程实践**：CONTRIBUTING 有 "Using AI" 章节，推荐 Claude Code；`.agents/skills/` 有 20+ 个仓库专用 agent skill（commit-pr、create-html-component、create-rolldown-plugin 等）；分支列表可见大量 `claude/*` 工作分支（branches API）
- **渠道**：Discord、GitHub Discussions（`README.md`）

## 9. 亮点与风险

**亮点**
1. 架构重设计教科书级：Feature/Slice/selector 模型同时满足 React 类型推断、custom element 声明式、tree-shaking 三个目标，且决策过程全部留档（`rfc/` + `internal/decisions/`）
2. 迁移工程体贴：老包名继续发布但 stub 化报错并给出明确错误码与文档，而非静默失败（`packages/video.js/src/videojs.ts`）
3. 适配器广度惊人：18 个引擎/平台适配器（YouTube/Twitch/TikTok/Spotify 等嵌入型），媒体播放器做成了"统一状态层 + 可换引擎"的框架
4. 工程化天花板：affected-test 矩阵 CI、PR 预览发包、bundle size 门禁、供应链防护、无障碍 e2e 一应俱全
5. 自研 spf 引擎掌控核心链路（ABR/DRM/HLS 解析），不绑死任何单一第三方

**风险/局限**
1. v10 与 v7/8 是断裂式不兼容：整个 Plugin/Component/Tech 生态（历史数千篇教程、大量插件）只适用于 8.x 分支，main 分支文档与存量生态存在割裂期阵痛
2. 30+ 包细粒度 monorepo + 统一版本发布，每次 release 全家桶同跳版本号，包间实际变更不均衡，消费者升级噪音较大
3. 构建链激进（vite-plus 为 voidzero 新产品、TypeScript native-preview beta、oxlint 等），工具成熟度依赖上游快速迭代
4. 分析基于 main 快照，spf 引擎虽大但生产验证时间尚短（v10 2026-10 才 GA），大规模生产回传数据未知

## 10. 总结评价

Video.js 是 15 年 web 视频基础设施的老牌王者，当前 main 分支正处于 2026-10 启动的 v10 全面重构稳定期：以 Feature 为组合单元、单一状态图、双平台（Custom Elements + React）、18 个可换引擎适配器的现代化框架，辅以自研 spf 流处理引擎和业界一流的工程化/测试体系。它不适合想找"经典 videojs() 调法"的用户——那请用 8.x 分支或旧版 npm 包；但对新项目，尤其是需要在 React 与原生 Web 间统一播放器状态、要接 HLS/DASH/DRM 或嵌入多平台视频源的场景，v10 的源码是当前开源播放器领域最值得精读的架构样本：读 `rfc/player-api.md` 看决策，读 `packages/core/src/core/player.ts` + `packages/store/src/core/slice.ts` 看状态模型，读 `packages/spf/src/` 看流媒体引擎，读 `packages/video.js/src/videojs.ts` 看如何优雅地杀死一个旧 API。
