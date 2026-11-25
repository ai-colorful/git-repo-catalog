# webadderallorg/Recordly 项目分析

> 星数 37153（快照 2026-11-21，gh api 实测）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：开源跨平台的"Screen Studio 平替"——无需剪辑技巧即可录制并自动加工出精致演示视频（自动缩放、光标美化、背景装帧、字幕、导出 MP4/GIF）的桌面应用。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | Recordly（webadderallorg/Recordly） | gh api |
| 星数 / Fork | 37,153 / 3,062 | gh api（2026-11-21） |
| 语言 | TypeScript（Electron 桌面应用 + Swift/C++ 原生捕获助手） | package.json、electron/native/ |
| 许可证 | AGPL-3.0（源码头带署名提醒，如 `src/components/video-editor/videoPlayback/motionSmoothing.ts` 首行） | LICENSE.md、README.md |
| 创建时间 | 2026-03-12 | gh api |
| 最近推送 | 2026-11-21（分析当日仍在推送） | gh api |
| 版本 | v1.4.0（最新 release，2026-09-08）；main 分支 package.json 为 2.0.0-beta.1 | gh api releases、package.json |
| 贡献者 | 83 人，但提交高度集中于 webadderall 与 young 两人 | gh api contributors、commits |
| 出身 | OpenScreen 的 fork，自称 80%+ 代码已分叉 | README.md Credits |

## 2. 项目是什么，解决什么问题

做产品演示视频的传统流程是：录屏 → 交给动效设计师加缩放、光标动画、背景装帧 → 剪辑导出。商业工具 Screen Studio 把这条流水线产品化，但闭源、收费且仅限 macOS。Recordly 用一个 Electron 应用把"录制 + 自动加工 + 剪辑 + 导出"合为一体，免费、开源、跨三平台（macOS 14+、Windows 10 19041+、现代 Linux 发行版）（README.md "System Requirements"）。

核心卖点集中在三类"无技巧出片"能力：其一，基于光标遥测的自动缩放建议——录制时采集全局点击/悬停事件，事后自动在时间轴上生成缩放区域（`src/components/video-editor/timeline/zoomSuggestionUtils.ts`）；其二，光标美化——用平滑弹簧动画、点击弹跳、运动模糊的合成光标覆盖真实光标（`src/components/video-editor/videoPlayback/` 下 cursorFollowCamera.ts、motionSmoothing.ts 等）；其三，成片装帧——壁纸、纯色、渐变背景 + 圆角、内边距、背景模糊、投影、宽高比预设，外加"随缩放反向补偿"的摄像头气泡画中画（README.md "Core Features"）。

编辑器是时间轴驱动的：拖拽剪辑片段、裁剪、变速区域、文字/图片标注、额外音轨，全部持久化为 `.recordly` 工程文件以便重开续编（README.md，工程状态结构见 `src/components/video-editor/projectPersistenceTypes.ts`）。导出支持 MP4 与 GIF，含质量、帧率、尺寸与循环选项（README.md "Export"）。

生态上有一个社区扩展系统：扩展运行在编辑器 renderer 中，通过权限门控的宿主 API 注册渲染钩子、光标特效、设置面板与素材包（光标音效、设备外框、浏览器 mockup、壁纸），从官方 Marketplace 分发（`EXTENSIONS.md`）。

从 README Credits 可知项目 2026 年 3 月以 OpenScreen fork 起步，此后 8 个月冲到 37.2k 星。客观背景是：需求端存在 Screen Studio 建立的"一键精致演示视频"认知，供给端缺少免费开源跨平台实现；项目上线后获 CodeRabbit 社区赞助展示位（README.md "Backed by the community"）、Ko-Fi 赞助者名单，并接入 winget、Microsoft Store、Homebrew tap、AUR 等全渠道分发（`.github/workflows/` 下 winget-releaser.yml、microsoft-store.yml、homebrew-tap.yml 等）。

## 3. 技术栈与依赖

- 运行时形态：Electron 43 + React 19 + Vite 6 + TypeScript 5（`package.json`）
- 场景合成/渲染：PixiJS 8 + pixi-filters（README.md "How It Works"、package.json），WebGPU 优先、WebGL 兜底（`src/lib/exporter/modernExportPipeline.ts`）
- 媒体处理：ffmpeg-static/ffprobe-static（导出编码）、mediabunny、mp4box、web-demuxer、gif.js、@fix-webm-duration（package.json）
- 原生层（`electron/native/`，29 个 Swift/C/C++ 文件 + 预编译二进制 bin/）：
  - macOS：ScreenCaptureKit 录制器（`ScreenCaptureKitRecorder.swift`，1759 行，含音频队列、麦克风、生命周期恢复等分块实现）、窗口列表、原生光标监视器（`NativeCursorMonitor.swift`，507 行）
  - Windows：WGC 捕获 + WASAPI 环回系统音频 + Media Foundation 编码（`electron/native/wgc-capture/src/`）、GPU 导出探针、NVIDIA CUDA 合成器、光标监视器（`electron/native/gpu-export-probe/`、`nvidia-cuda-compositor/`）
- 输入遥测：uiohook-napi 全局键鼠钩子（package.json dependencies）
- 本地语音字幕：sherpa-onnx-node 跑 Whisper（可下载 small 模型）与 Parakeet（词级时间戳）双引擎（`electron/ipc/captions/whisper.ts`、`parakeet.ts`）
- 云服务：Supabase JS SDK（登录、反馈、错误上报）、Stripe 计费邮件（`services/email/stripe/`）、Cloudflare Worker + R2/D1 的 Recordly Share 分享服务（`services/recordly-share/README.md`）
- UI/工程化：Tailwind CSS 4 + HeroUI、dnd-timeline（时间轴）、Biome（lint/format）、Vitest + Playwright（package.json）
- 国际化：12 种语言（`src/i18n/locales/`），自动翻译中文 CI（`.github/workflows/auto-translate-zh.yml`）

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `electron/` | Electron 主进程与预加载：窗口/更新/权限/工程文件 IO、IPC 处理器（ipc/ 下 recording、export、cursor、captions、ffmpeg 等十余子域）、原生助手源码与二进制（native/） |
| `src/` | 渲染进程 React 应用：video-editor/（编辑器全部状态、时间轴、导出控制器、字幕、设置）、videoPlayback/（Pixi 播放与光标/相机渲染）、lib/exporter/（导出管线）、i18n/ |
| `services/` | 配套服务源码：supabase/（迁移、Edge Function、测试）、recordly-share/（Cloudflare Worker 分享服务）、email/（Stripe 计费邮件渲染）、browser-pages/ |
| `scripts/` | 构建/发布脚本：原生助手编译、pin 版 FFmpeg 构建、发布校验和、Windows Store 打包等 40+ 脚本 |
| `tests/` | Playwright UI 测试（ui/、studio/） |
| `build/`、`branding/`、`icons/`、`public/` | 打包配置（appx、mac entitlements）、品牌素材、静态资源 |
| `docs/` | FFmpeg 分发合规、云分享、认证、Microsoft Store 上架等内部文档 |

组织方式评述：单体仓库（非 monorepo），但内部按"主进程 IPC 子域 + 渲染进程功能域 + 独立服务目录"切分得相当细。渲染端 video-editor/ 一个目录就有 200+ 文件，状态拆成 state/、hooks/、timeline/、export/ 等子域；导出器 lib/exporter/ 同时存在 modern*（新管线）与 legacy*（旧管线）两代实现并存。规模上非测试 TS/TSX 约 1164 个文件、18.9 万行（实测 find/wc），是典型的"爆发期高速演进"形态。

## 5. 核心架构与关键实现

### 5.1 平台捕获层（stdout 协议驱动的原生助手）

Electron 主进程按平台 spawn 原生助手进程，通过 stdout 行协议通信：Windows 上 `wgc-capture.exe` 输出稳定标记（如 WARNING、"Recording started"），主进程据此决定浏览器捕获回退，避免管道竞态（`electron/native/wgc-capture/src/main.cpp` 头部注释）；macOS 光标监视器输出 `STATE:<光标类型>` 与 `INTERACTION:mouseup:1` 行，`electron/ipc/cursor/monitor.ts` 的 `handleCursorMonitorStdout()` 逐行正则解析并广播给全部窗口。macOS 用 ScreenCaptureKit 干净地排除真实光标，Windows 依赖 19041+ 的 WGC 光标隐藏，Linux 走 Electron 捕获且不支持光标隐藏（README.md "Limitations"，`electron/native/ScreenCaptureKitRecorder.swift` CaptureConfig 支持 display/window/region 三种源 + 系统音频/麦克风）。

### 5.2 自动缩放建议算法（与 Screen Studio/openscreen 的核心差异点）

录制期通过 uiohook 与原生监视器积累光标遥测 `{timeMs, cx, cy, interactionType, cursorType}` 采样；编辑期由两级检测器生成缩放建议（`src/components/video-editor/timeline/cursorInteractionCandidates.ts`、`zoomSuggestionUtils.ts`）：

1. 显式事件级：解析按压区间，按点击后的轨迹分类为 double-click-like/dropdown-open/text-selection/text-field-click 等，赋 900–1500 强度权重；
2. 启发式补漏：驻留检测（450–2600ms 内移动 < 0.02 归一化坐标）判 dwell/click-like/text-focus-like；
3. 聚类布位：显式点击按 2500ms 间隙聚类、首尾加 500ms padding，缩放块以最强点击为锚点居中，并逐一避开已保留区间（手动缩放区、相邻建议块）；裁剪/变速片段还会把录制时间正确映射回编辑时间轴。

关键在于 `buildInteractionZoomSuggestions()` 的冲突消解：候选窗先过滤保留区间，再尝试居中 → 边界外扩 → 最小时长内移的三级落位（`zoomSuggestionUtils.ts:237-420`）。相比此前分析的 openscreen（被 fork 的前身）与 Screen Studio，Recordly 的差异是"事后可编辑的建议 + 显式点击事件驱动"而非录制期实时变焦，建议可直接在时间轴上二次拖改。

### 5.3 光标跟随相机与弹簧平滑

缩放期间相机不是死跟光标，而是"安全区跟随"：保持持久相机中心，仅当光标移出当前缩放视野的内圈安全区（snapToEdgesRatio=0.25）时才平移刚好把光标拉回区内，缩出时相机冻结焦点（`src/components/video-editor/videoPlayback/cursorFollowCamera.ts`）。光标位置用弹簧动力学平滑（stiffness/damping/mass + 可调乘子，0–2 平滑档位），配合点击弹跳、运动模糊与 macOS 风格光标资产栅格化（`videoPlayback/motionSmoothing.ts`、`cursorAssetRasterization.ts`）。摄像头气泡可选随缩放反向缩放以维持视觉平衡（`video-editor/webcamOverlay.ts`）。

### 5.4 导出编码链（双管线 + 多级回退）

预览与导出共用同一 Pixi 场景逻辑。导出走 `src/lib/exporter/modernExportPipeline.ts` 的重试状态机：原生阶段（H.264 流送原生编码器）→ 失败则 raw FFmpeg rawvideo 重试；渲染阶段 WebGPU 失败则整条管线重建切 WebGL（帧与 GPU 资源不可复用，见该文件注释）。Windows 上另有 GPU 导出进程与 NVIDIA CUDA 合成器 opt-in 路线（`electron/ipc/export/nativeVideo/`、`scripts/build-windows-gpu-export.mjs`、`useNvidiaCudaExportOptIn.ts`）；GIF 走 gif.js/流式 GIF 编码器并带体积优化估计（`lib/exporter/gifExporter.ts`、`streamingGifEncoder.ts`）。FFmpeg 分发做了合规工程：macOS 安装器现场构建 pin 版 FFmpeg 8.1.2（静态 x264/libvpx，保留系统框架），含 SHA-256 校验与链接检查（`docs/ffmpeg-distribution.md`、`scripts/build-redistributable-ffmpeg.mjs`）。

### 5.5 本地字幕引擎

字幕生成完全本地化：主进程按"麦克风侧车优先、系统音频兜底"取伴生音轨（`electron/ipc/captions/audioCandidates.ts`），Whisper 模型按需下载并汇报进度（`whisper.ts`），Parakeet 路线经 sherpa-onnx 输出 SentencePiece token 级时间戳再重组词级 cue（`parakeet.ts`）。两条引擎结果可合并（`mergeSources.ts`）。

## 6. 构建与运行

来自 README.md 与 package.json scripts（未实际运行验证）：

```bash
# 开发
npm install && npm run dev          # Vite dev + Electron
# 打包（先编译全部原生助手：whisper runtime、native helpers、wgc、gpu-export、cuda compositor、cursor-monitor）
npm run build                       # 全平台 electron-builder
npm run build:mac | build:win | build:linux
# 质量与测试
npm run typecheck                   # tsc --noEmit
npm run lint                        # biome lint
npm test                            # vitest --run（430 个 *.test.ts(x/mjs) 文件）
npm run test:ui                     # playwright test
```

平台前置：macOS 需 Xcode 26+（打包用 actool）；Linux 需 build-essential/cmake 与 X11 开发库；Windows 需 VS2022 C++ 工作负载与 CMake（README.md "Prerequisites"）。

## 7. 测试与工程质量

- 单元测试规模可观：全仓 430 个测试文件，与源码同目录放置（colocated），Vitest 框架；关键纯逻辑（缩放建议、裁剪几何、弹簧平滑、IPC 校验）均有对应测试，如 `zoomSuggestionUtils.test.ts`、`useAutoApplyZooms.test.ts`、`modernExportPipeline.test.ts`
- 原生层测试到 Swift 侧：`electron/native/` 下 ScreenCaptureKitRecorder 的音频、生命周期、恢复、质量均有 TS 侧测试文件（.test.ts 与 Swift 实现并置），另有 Windows WGC 的音频样本对齐/帧填充测试
- E2E：Playwright 覆盖 UI 与 announcement studio（playwright.config.ts、tests/ui、tests/studio）
- CI：quality.yml 在 PR/push 上跑 typecheck + biome lint + 格式检查（advisory），Node 22；发布侧有 build.yml、macos-release-candidate.yml、microsoft-store.yml、winget-releaser.yml、homebrew-tap.yml 全渠道矩阵
- 工程化观察：主进程 main.cjs 有规范化与冒烟脚本（normalize-electron-main-cjs、smoke-electron-main-cjs），发布产物有校验和与分发验证脚本；二进制助手带 helpers-manifest.json 与 PE/Mach-O 内容哈希校验（scripts/pe-content-hash.mjs、macho-content-hash.mjs）。服务端（Supabase/Share Worker）也有 SQL 回滚式权限/配额测试（services/supabase/README.md）。

## 8. 社区与维护状态

- 活跃度极高：gh api 显示创建于 2026-03-12，最近 30 次提交全部发生在 2026-11-19 至 11-21（分析当日前两天内），提交人几乎全部为 webadderall 与 young 两人——核心团队极小，bus factor 约 2
- 贡献者 83 人（gh api contributors），README 明示 "Accepting PRs"，CONTRIBUTING.md 列出最需要帮助的方向：导出优化、Linux 原生录制、壁纸、扩展生态
- Release 节奏：30 个 release；最新正式版 v1.4.0（2026-09-08），此前 v1.3.x 系列约每月一版；main 分支已进入 2.0.0-beta.1，即 v2 正在开发中尚未发正式版
- Issue 积压 520 个 open issues（gh api），相对星数属于高位，符合"单人主导 + 爆发增长"项目的典型负载
- 商业化基础设施已就位但处于早期：桌面对接 Supabase 云账号、Recordly Share 云分享、Stripe 计费邮件与 Pro/EarlyBird 定价 UI（`src/components/billing/ClassicPricingModal.tsx`、`BillingProvider.tsx`），Ko-Fi 赞助通道（README.md）

## 9. 亮点与风险

亮点：
1. 自动缩放建议算法工程完整：显式点击 + 启发式驻留双通道、聚类冲突消解、变速/裁剪时间映射，纯函数实现且测试充分（`zoomSuggestionUtils.ts`）
2. 平台捕获层动真格：三个平台各有原生助手（ScreenCaptureKit/WGC+MF/WASAPI/光标监视器/CUDA 合成器），而非停留在 Electron 通用捕获（`electron/native/`）
3. 导出链的多级回退设计成熟：原生编码 → raw FFmpeg → WebGPU→WebGL 整管线重建，失败路径有指标与诊断上报（`modernExportPipeline.ts`）
4. 本地化字幕（Whisper/Parakeet 双引擎、词级时间戳）与扩展系统（权限门控宿主 API）显著超出"录屏工具"基本盘
5. 工程纪律好：430 个测试文件、合规化的 FFmpeg 分发文档与校验、12 语言 i18n 自动化

风险/局限：
1. 团队集中度极高：提交近乎全部来自两人，520 个 open issue 积压，长期维护性存疑
2. AGPL-3.0 + 源码内署名提醒（如 `motionSmoothing.ts` 首行）对商业集成不友好，fork 复用需谨慎
3. Linux 是短板：不支持光标隐藏、系统音频依赖 PipeWire，真实光标可能与美化光标同框（README.md "Limitations"）
4. 18.9 万行单仓、modern/legacy 导出双管线并存、原生二进制随仓库分发（electron/native/bin/），新贡献者上手成本高

## 10. 总结评价

Recordly 是 2026 年开源工具里增长最陡的案例之一：8 个月 37.2k 星，靠"免费开源跨平台的 Screen Studio 平替"这一清晰定位，加上远超同类新项目的工程完成度（三平台原生捕获、双导出管线、本地 AI 字幕、扩展市场）立住口碑；它 fork 自 openscreen 但已在捕获、导出、字幕、商业化四个方向全面分叉。代码质量整体高于爆发期平均水平，但资源几乎全部系于两位核心开发者。适合三类人：想给产品做精致演示视频又买不起/不想装闭源工具的创作者直接用；想做 Screen Studio 类产品、研究"自动变焦+光标遥测"实现的开发者读 `zoomSuggestionUtils.ts` 与 `electron/native/`；想研究 AGPL 开源 + Supabase/Stripe/Cloudflare 商业化组合的维护者看 `services/` 与 `src/components/billing/`。
