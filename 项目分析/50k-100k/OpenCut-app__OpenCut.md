# OpenCut-app/OpenCut 项目分析

> 星数 95867（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：号称"开源 CapCut 替代品"的免费视频编辑器，浏览器/桌面/移动多端布局；96k+ 星由已归档的经典版（Next.js Web 编辑器）赢得，仓库当前处于**从零重写期**（Rust 核心 + GPUI 桌面 + TanStack Web）。

| 项目 | 内容 |
| --- | --- |
| 名称 | OpenCut-app/OpenCut（GitHub 简介 "The open-source CapCut alternative"） |
| 星数 / Fork / Open issues | 96835 / 9612 / 387（gh api，2026-10-31 实测） |
| 主语言 | TypeScript（经典版）；重写版为 TS + Rust 混合 |
| 许可证 | MIT（`LICENSE`） |
| 创建 / 最近 push | 2025-06-22 / 2026-09-24（gh api） |
| 官网 | https://opencut.app（经典版）；new.opencut.app（重写版，`apps/web/wrangler.jsonc`） |
| 仓库现状 | 默认分支 main 仅 130 个文件（~950KB）：重写版骨架；经典版全量代码移至 opencut-app/opencut-classic（291 星，2026-05-17 冻结） |

## 2. 项目是什么，解决什么问题

OpenCut 的出发点写在经典版 README（`opencut-classic: README.md`）：**隐私**（视频不出本机）、**免费**（CapCut 大量基础功能已收费）、**简单**（CapCut 证明了大众要的是低门槛时间线编辑器）。它是一个纯浏览器端运行的视频编辑器：导入素材、多轨时间线剪辑、关键帧动画、遮罩、变速、字幕/自动转写，最后在本地导出 MP4/WebM——服务端只承担账号、项目元数据与限流。

星数增长集中在 2025 年中至 2026 年上半年：经典版以"CapCut 开源替代"叙事 + 完全浏览器内处理（无需上传）击中痛点，星数一路冲到 96k+。经典版迭代记录见 `changelog/0.1.0.md`（2026-02-23，属性面板重做、1000+ 字体）、`0.2.0.md`（2026-03-01，关键帧、逐 clip 特效、波纹剪辑）、`0.3.0.md`（2026-04-15，遮罩、关键帧曲线图编辑器、变速、性能大修）。

2026-04 的 0.3.0 起项目转向激进重构（`changelog/0.3.0.md` technical 小节）：WebGL 渲染器被 Rust/wgpu 合成器（WASM）替换，时间改为整数 tick 的 `MediaTime`，桌面端以 GPUI 起步。2026-05 经典代码整体迁出到 opencut-classic 并冻结，主仓库清空为重写版。当前主仓库 README（`README.md` Status 一节）明确：**正在从头重写**，目标是 Editor API、插件优先架构、一套代码跑桌面/移动/浏览器（Rust 核心）、MCP server（供 AI agent 调用）、无头渲染/批处理、编辑器内置脚本 Tab；并提示"今天要用请选 opencut-classic"。

需要特别说明：本仓库 HEAD 里**没有可用的编辑器功能**——`apps/web/src/routes/editor.tsx` 只有"Coming soon"占位页，`apps/desktop` 四个面板也只是渲染文字标签的骨架。因此本文第 5 章对"时间线编辑器架构、渲染管线、状态管理"的深入分析基于经典版源码（opencut-classic，同为 MIT、同团队），这也是 96k 星对应的真实代码。

## 3. 技术栈与依赖

**经典版（opencut-classic/apps/web，实际服役中）**
- Next.js 16.1.3 + React 19 + Tailwind，Radix UI 组件；OpenNext 部署 Cloudflare Workers（`apps/web/package.json`、`open-next.config.ts`）
- 状态：zustand 5（仅 UI 态）+ 自研 EditorCore 管理器体系（领域态）
- 媒体：mediabunny 1.29.1（浏览器内 demux/decode/encode）、wavesurfer.js（波形）、soundtouchjs（变速保音高）
- AI：@huggingface/transformers（本地模型自动转写字幕）
- 服务端：better-auth + drizzle-orm + postgres（账号/项目）、Upstash Redis 限流；Docker Compose 起本地 db/redis（经典版 README）
- 渲染核心：Rust + wgpu + WGSL，编译为 WASM，以 npm 包 `opencut-wasm` 0.2.10 发布（`rust/wasm/Cargo.toml`）

**重写版（本仓库 HEAD）**
- Monorepo 工具链：moonrepo（moon 2.3.3）+ proto 版本 pin（bun 1.3.11、rust 1.97.0，`.prototools`）
- `apps/web`：TanStack Start（Vite 8 + React 19.2 + TanStack Router），Tailwind 4，74 个 shadcn 风格 ui 组件，wrangler 部署 Workers（`apps/web/package.json`、`wrangler.jsonc`）
- `apps/api`：Elysia + CloudflareAdapter 的 Worker，仅 `/`、`/health`、`/echo`（`apps/api/src/index.ts`）
- `apps/desktop`：GPUI 0.2.2（Zed 的 Rust UI 框架），目标 macOS Metal / Windows DirectWrite / Linux Vulkan-Blade（`apps/desktop/README.md`）
- `crates/media`：FFmpeg 8.1.3（LGPL-2.1）解码底座，6 平台预构建 pin 体系（`crates/media/setup/ffmpeg.json`）

## 4. 目录结构与代码组织

**本仓库 HEAD（重写版，130 文件）**

| 目录 | 职责 |
| --- | --- |
| `apps/web/` | 新 Web 前端骨架（TanStack Start），/editor 路由占位 |
| `apps/api/` | Cloudflare Worker API 骨架（Elysia） |
| `apps/desktop/` | GPUI 桌面壳：Shell + Browser/Preview/Inspector/Timeline 四面板 + 自建组件库（button/context_menu/resizable 等，`apps/desktop/src/components/`） |
| `crates/media/` | 媒体 crate 的 FFmpeg 拉取/源码构建脚本（crate 本体尚未出现在仓库） |
| `changelog/` | 经典版 0.1.0–0.3.0 发布记录（含高价值 technical 注记） |
| `.moon/`、`.prototools`、`moon.yml` | moonrepo 工作区与工具链 pin |
| `.github/workflows/` | `bun-ci.yml`（三 OS 跑 `moon ci`）、`media-deps.yml`（源码构建 FFmpeg 六平台并发 prerelease） |

**经典版（opencut-classic，1128 文件，web 端 ~685 个 TS/TSX、约 9.1 万行；rust/ 25 个 .rs 约 4300 行，实测统计）**

| 目录 | 职责 |
| --- | --- |
| `apps/web/src/core/` | EditorCore 单例 + 12 个 manager（时间线/播放/音频/场景/工程/媒体/渲染/命令/保存/选择/剪贴板/诊断） |
| `apps/web/src/timeline/` | 时间线域：store、controllers（拖拽/缩放/裁剪/播放头）、snapping、group-move/group-resize、placement、bookmarks、audio-separation |
| `apps/web/src/services/renderer/` | 渲染管线：scene-builder、CanvasRenderer、nodes/（10 种节点）、compositor/wasm-compositor 桥 |
| `apps/web/src/preview/`、`src/animation/`、`src/effects/`、`src/masks/`、`src/text/`、`src/graphics/`、`src/stickers/`、`src/subtitles/`、`src/transcription/` | 预览画布交互与各素材域 |
| `apps/web/src/media/`、`src/export/`、`src/services/{video-cache,waveform-cache,storage}/` | 媒体解码、导出、缓存与 IndexedDB/OPFS 存储（迁移 v0→v25） |
| `apps/web/src/commands/` | 命令模式实现（timeline/scene/project/media 四组） |
| `rust/crates/{compositor,effects,masks,gpu,time,bridge}` | wgpu 合成器、特效管线、JFA 遮罩羽化、GPU 上下文、时间系统、WASM 导出宏 |
| `rust/wasm/` | `opencut-wasm` npm 包入口 |
| `docs/`、`notes/` | 架构文档（effects-renderer、keyframes、actions）与架构随笔（primitives-vs-domains） |

组织方式评述：经典版是"按领域分目录的单体 Next.js 应用 + Rust 核心 crate 树"，域内聚明显、几乎无循环依赖约束文档化（见 `notes/primitives-vs-domains.md` 提出的"原语类型 vs 领域类型"判据）；重写版则收敛为 moonrepo 多包工作区，TS 侧重 Web 骨架，业务逻辑重心移向 Rust（Cargo 工作区暂只挂 `apps/desktop`，`crates/*` 被注释——`Cargo.toml`）。

## 5. 核心架构与关键实现

### 5.1 状态管理：EditorCore 管理器 + 薄 zustand（经典版）

领域状态不走全局 store，而集中在 `EditorCore` 单例（`opencut-classic: apps/web/src/core/index.ts`）派生的 12 个 manager（timeline 935 行、project 707 行、audio 700 行为三大件），manager 之间通过持有 `EditorCore` 引用互相调用，用 `listeners: Set<() => void>` 手写订阅通知。zustand 只承载 UI 态，且刻意做薄——`apps/web/src/timeline/timeline-store.ts` 头部注释：

```ts
/**
 * UI state for the timeline
 * For core logic, use EditorCore instead.
 */
```

播放进度同样如此：`core/managers/playback-manager.ts` 用 `playbackStartWallTime` + `playbackStartTime`（均为 `MediaTime`）推算当前播放头，向 timeline/scenes 订阅"作用域调和"（`bindTimelineScope`）。这套设计的收益是撤销/重做与选择快照完全内聚在命令层，代价是订阅样板较多、初读门槛高。

### 5.2 时间线架构：类型化轨道 + 命令模式 + 波纹剪辑

轨道模型在类型层面锁死排列规则（`apps/web/src/timeline/types.ts:76`）：

```ts
export interface SceneTracks {
  overlay: OverlayTrack[];   // 视频/文本/图形/特效轨
  main: VideoTrack;          // 主视频轨（单例）
  audio: AudioTrack[];
}
```

所有变更都是 Command 对象：`Command` 抽象类提供 `execute/undo/redo`（`commands/base-command.ts`），`core/managers/commands.ts` 的 `CommandManager` 维护 `history/redoStack` 并随每条命令保存选择快照，undo 时只对"显式改过选择"的命令还原选择，避免吞掉用户的框选。时间线相关的 25+ 个命令（Insert/Delete/Split/Move/UpsertKeyframe/…）注册在 `core/managers/timeline-manager.ts`。波纹编辑（ripple）以算法层独立实现：命令执行后统一走 `computeRippleAdjustments → applyRippleAdjustments`（`core/managers/commands.ts` 的 `applyRippleIfEnabled`），与触发命令解耦——这是 `changelog/0.3.0.md` 中"diff + 区间集"重构的落点。指针交互由 `timeline/controllers/`（drag-drop、resize、playhead、zoom、keyframe-drag 等 7 个控制器）分担，拖拽预览通过 `TimelineManager.previewOverlay/previewTracks` 做免提交的临时态。

### 5.3 渲染管线：TS 场景图 → WASM/wgpu 合成器

每帧流程在 `services/renderer/canvas-renderer.ts` 的 `CanvasRenderer.render` 中一目了然：

```ts
await resolveRenderTree({ node, renderer: this, time });       // 解析素材/动画
const { frame, textures } = await buildFrameDescriptor({...}); // 生成帧描述
wasmCompositor.syncTextures(textures);                          // 纹理上传（带缓存）
wasmCompositor.render(frame);                                   // Rust/wgpu 合成出帧
```

`scene-builder.ts` 把 `SceneTracks` 展开为节点树（Root/Video/Image/Text/Sticker/Graphic/Color/BlurBackground/EffectLayer 九类，`services/renderer/nodes/`），按 startTime 排序控制层序；变换/不透明度/混合模式从参数系统解析（`rendering/animation-values.ts`）。合成端 `rust/crates/compositor/src/compositor.rs` 用 wgpu 搭三条管线（layer/blend/mask，WGSL shader 内嵌于 `shaders/`），配合 `TexturePool`/`TextureStore` 复用显存；特效走 `effects` crate 的多 pass 管线（如 blur 拆 H/V 两趟），遮罩羽化用 Jump Flooding Algorithm（`masks` crate）。桥接层 `services/renderer/compositor/wasm-compositor.ts` 以 contentHash 缓存 `OffscreenCanvas` 纹理，静态内容不必每帧重新上传——对应 0.3.0 的"播放性能大幅改善"。特效扩展是声明式的：TS 侧只声明 shader 标识与 uniforms，Rust 负责设备与 pass 执行（`docs/effects-renderer.md`）。

### 5.4 时间系统与媒体/导出

时间不是浮点秒，而是 Rust 定义的整数 tick：`rust/crates/time/src/media_time.rs` 中 `TICKS_PER_SECOND = 120_000`（可被 23.976/29.97/30 等 framerate 分母整除），`FrameRate` 为有理数 `{numerator, denominator}`，经 tsify 生成 TS 类型、以 `opencut-wasm` 导出，TS 侧封装在 `apps/web/src/wasm/media-time.ts`。时间线元素的 `duration/startTime/trimStart/trimEnd` 全部是 `MediaTime`（`timeline/types.ts`），从根上消除了浮点累计误差与帧对齐问题。解码/编码交给 mediabunny：`media/mediabunny.ts` 负责 demux/decode，导出在 `services/renderer/scene-exporter.ts` 用 `Output + CanvasSource + Mp4OutputFormat/WebMOutputFormat` 把合成画布逐帧写入容器；音频调度在 `core/managers/audio-manager.ts`（700 行，含 timing slip 恢复），变速保音高用 soundtouchjs，自动字幕用 @huggingface/transformers 本地推理。项目数据存 IndexedDB/OPFS（`services/storage/`），迁移链 v0→v25 每步带回归测试。

### 5.5 重写版现状（本仓库 HEAD）

桌面壳已成型但无功能：`apps/desktop/src/shell.rs` 将四面板各建为 GPUI `Entity`（跨帧保状态），`main.rs` 含 WSLg Wayland 兼容规避；组件库（button 217 行、context_menu 276 行、resizable 170 行）与语义化主题（`theme.rs`，随系统深浅色切换）是仅有的"真实现"。基建先行是重写版的特点：`media-deps.yml` 用 llvm-mingw 交叉编译等手段从源码构建 FFmpeg 8.1.3 六平台共享库并发 prerelease，`crates/media/setup/setup.ts` 负责 pin/校验 sha256/落入 `.cache/media` 供 `build.rs` 链接——为 Rust 媒体核心铺路。

## 6. 构建与运行

经典版（opencut-classic，`README.md` + `apps/web/package.json`）：

```bash
docker compose up -d db redis serverless-redis-http   # 可选，本地 db/redis
cp apps/web/.env.example apps/web/.env.local
cd apps/web && bun install
bun run dev        # next dev --turbopack
bun run build      # next build
bun run lint       # eslint src --ext .ts,.tsx
bun run db:push:local  # drizzle-kit 建表
```

重写版（本仓库，`README.md`；需先 `proto use` 安装 pin 工具链）：

```bash
moon run web:dev       # localhost:5173（TanStack Start/Vite）
moon run api:dev       # localhost:8787（Elysia Worker）
moon run desktop:dev   # cargo run（GPUI 桌面壳）
moon run desktop:build # cargo build --release
bun run crates/media/setup/setup.ts   # 拉取/构建 FFmpeg 到 .cache/media
```

测试：重写版 `apps/web` 配置了 vitest + testing-library（`apps/web/package.json` scripts.test），暂无测试文件；经典版用 bun test 跑 `__tests__/`。规范要求不实际运行构建，以上来自项目文档与工程文件。

## 7. 测试与工程质量

- 经典版 37 个测试文件，风格为纯函数/迁移单测：storage 迁移 v0→v25 每步一测（`services/storage/migrations/__tests__/`）、ripple、retime 的 resolve/split、masks snap、animation、params、fps、keybindings 持久化（实测统计 `find -path "*__tests__*"`）。无 e2e。
- 自定义 ESLint 规则带自己的测试（`eslint/rules/__tests__/`）；`.github/copilot-instructions.md` 全量引入 Ultracite/Biome 规则（强类型、a11y、禁 any/枚举/namespace 等），工程纪律意识强。
- CI：`bun-ci.yml` 在 ubuntu/windows/macos 三平台跑 `moon ci`；重写版另有 FFmpeg 构建流水线含 sha256 校验与 checksums 发布。
- 代码质量观察：注释解释"为什么"（如 `main.rs` 的 WSLg 规避注释、undo 选择快照注释），`docs/`/`notes/` 有架构自我批判文档；架构分层清晰，但经典版 manager 体系单文件偏大（timeline-manager 935 行）。

## 8. 社区与维护状态

- 活跃度：主仓库最近 push 2026-09-24（gh api），387 个 open issues，9.6k fork；但 main 上代码量极小且 README 声明"暂不接受外部贡献，架构设计期"（`README.md` Contributing）。
- 发布节奏：经典版 2026-02 至 2026-04 间发布 0.1.0/0.2.0/0.3.0（`changelog/`）；FFmpeg 依赖以 prerelease tag 独立发布（`media-deps.yml`）。重写版尚无版本发布（未确认，gh api release 数据未拉取）。
- 生态：Discord 社区（README 徽章）、X @opencutapp；赞助商 fal.ai（`README.md` Sponsors），经典版曾获 Vercel OSS Program 支持（classic README）。
- 团队动向：2026-05 将经典版整体迁出冻结（opencut-classic 仅存活 1 天的 push 窗口，gh api），主仓库转向重写——96k 星仓库处于"招牌已立、正片重拍"阶段。

## 9. 亮点与风险

**亮点**
1. 真正的浏览器内全链路剪辑/导出（mediabunny 解码 + wgpu/WASM 合成 + 本地编码），隐私叙事与实现一致。
2. 时间系统教科书级：120k ticks/秒整数 `MediaTime` + 有理数 `FrameRate`（`rust/crates/time`），从类型上消灭浮点帧对齐误差。
3. 命令模式 + 选择快照 + 算法化波纹剪辑的编辑器内核，undo/redo 语义细致（`core/managers/commands.ts`）。
4. 高标准工程实践：迁移测试全覆盖、自定义 lint 规则、架构文档与自我批判（`notes/primitives-vs-domains.md`）、三平台 CI。
5. 重写方向清晰且有基建深度（FFmpeg 多平台源码构建流水线、GPUI 桌面壳、插件优先/MCP/无头模式愿景）。

**风险/局限**
1. 当前 HEAD 无可用编辑器功能，主仓库实质处于"空仓"状态；96k 星对应的可用代码在 opencut-classic（已归档不维护）。
2. 重写版公开仓库仅 130 文件，`crates/*` 未入 Cargo 工作区（`Cargo.toml`），核心 Rust 代码疑似未开源或未就绪（未确认），存在长期"占坑"风险。
3. 无 e2e 测试、重写版尚无任何测试落地；编辑器内核大量手写订阅，接管成本高。
4. 浏览器内处理 4K/长视频的内存与性能天花板仍在（OPFS/配额管理只能缓解，`services/storage/quota.ts`）。

## 10. 总结评价

OpenCut 是 2025 年最成功的"叙事驱动"开源项目之一：用"开源 CapCut 替代 + 视频不出本机"精准命中的不只是功能缺口，更是用户情绪，经典版以约一年时间做到 96k 星并交付了一个架构相当讲究的浏览器视频编辑器——整数 tick 时间系统、wgpu/WASM 合成器、命令模式内核都超出同类 demo 水准。如今项目正把这场胜利押注在彻底重写上：Rust 核心、GPUI 桌面、插件优先与 AI 接口（MCP），代价是主仓库暂无可用品、社区只能围观。适合三类人关注：想研究"浏览器内视频编辑器如何落地"的工程师（直接读 opencut-classic 的 `core/`、`services/renderer/`、`rust/crates/`）；押注本地优先创作工具方向的产品人；以及愿意等重写兑现、提前熟悉 moonrepo + GPUI + FFmpeg 基建的潜在贡献者。读源码请以 opencut-classic 为正文、本仓库 HEAD 为附录。
