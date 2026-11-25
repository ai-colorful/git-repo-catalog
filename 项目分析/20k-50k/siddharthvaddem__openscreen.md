# siddharthvaddem/openscreen 项目分析

> 星数 40033（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：开源免费版 Screen Studio——Electron 桌面应用，录屏后自动加光标跟随缩放、运动模糊、字幕、摄像头画中画等"产品演示美化"效果，导出无水印 MP4/GIF，可商用。

| 项目 | 内容 |
|---|---|
| 名称 | siddharthvaddem/openscreen |
| 星数 / Fork | 40033 / 3246（gh api 核实） |
| 语言 | TypeScript（含 Swift、C++ 原生辅助层） |
| 许可证 | MIT（LICENSE: "Copyright (c) 2025 Siddharth Vaddem"） |
| 创建时间 | 2025-10-10（gh api） |
| 最后推送 | 2026-10-17（gh api），**仓库已归档** |
| 最新版本 | v1.5.0（2026-06-06），共 13 个 release |
| 贡献者 | 99 人（gh api contributors 分页统计） |
| 平台 | macOS（ScreenCaptureKit 原生）、Windows（WGC 原生）、Linux（Electron 采集路径） |

## 2. 项目是什么，解决什么问题

Screen Studio 是 macOS 上售价 $29/月（买断制）的屏幕录制美化工具，核心卖点是"录完即出片"：自动跟随光标的平滑缩放、美化的光标、背景与阴影包装。OpenScreen 用开源方式复刻这一核心工作流（README.md 明确自称 "your free, open-source alternative to Screen Studio"，并声明"不是 1:1 克隆"）。

产品形态是典型的"录制 → 编辑 → 导出"三段式：启动后选择窗口/整屏 + 麦克风/系统声/摄像头组合开始录制；录制产物进入时间线编辑器，用户可裁剪、调速、加标注、换壁纸背景；导出时渲染引擎逐帧合成缩放、光标、模糊、画中画并编码为 MP4 或 GIF。README 列出的核心功能包括：窗口/整屏录制、摄像头 PiP（可拖拽定位/镜像/形状遮罩）、自动或手动缩放（自动缩放跟随光标）、光标主题与点击特效、离线端侧自动字幕、壁纸/渐变/背景图、运动模糊、时间线裁剪与分段变速、标注（文字/箭头/图片）、13 种界面语言。

定位口号是"100% free, no subscriptions, no watermarks, free for commercial use"（README + repo description），作者在 README 中强调"软件应当可及，没有付费档、没有高级功能锁定"。同时作者坦诚警告："This started as a side project that blew up; not production grade and you'll hit bugs"——即从个人副业项目爆发式走红（一年 4 万星），工程完成度按"能用的好工具"而非企业级标准。

**生命周期（已归档）**：2025-10-10 创建，2026 年 3-6 月密集发版（v1.3.0 2026-04-02、v1.4.0 2026-05-06、v1.5.0 2026-06-06），最后一次代码活动为 2026-06-17 的 README 更新，2026-10-17 以提交 "update spinoff github repo" 归档——该提交在 README 顶部加入归档声明："OpenScreen is now archived and no longer maintained"，并指向社区接续维护的 fork `getopenscreen/openscreen`。归档的具体原因（精力/商业化/维护负担）作者未在仓库内说明，**未确认**；可考的客观事实只有 README 中"副业项目爆发"的自述与维护停止的时间线。从创建到归档整一年，4 万星几乎全部在活跃期内获得。

## 3. 技术栈与依赖

来源：`package.json`。

- **框架**：Electron 41 + React 18 + Vite 7 + TypeScript 5.9；`main` 入口为 `dist-electron/main.js`，通过 vite-plugin-electron 组织双进程。
- **渲染引擎**：`pixi.js` 8 + `pixi-filters`（MotionBlurFilter 运动模糊）+ `@pixi/filter-drop-shadow`——编辑器预览与导出帧渲染共用一套 PixiJS 场景。
- **媒体处理**：`mediabunny`（MP4 封装/复用）、`web-demuxer` + `mp4box`（解封装）、WebCodecs（VideoEncoder/AudioEncoder，浏览器 API）、`gif.js`（GIF 导出）。
- **端侧 AI**：`@xenova/transformers`——Whisper 语音转文字完全在本地 Web Worker 运行，不联网上传。
- **动效**：`motion`（spring 生成器，用于缩放/光标弹簧平滑）、`gsap`（标注文字动画）。
- **UI**：Radix UI 全家桶 + Tailwind CSS + lucide-react；`dnd-timeline` 时间线、`react-rnd` 可拖拽元素。
- **原生层**：macOS 为 Swift Package（ScreenCaptureKit + AVFoundation，`Package.swift` 声明两个可执行 helper）；Windows 为 C++/CMake（Windows.Graphics.Capture + WASAPI + Media Foundation + DirectShow）。
- **质量工具**：Biome（lint+format）、Vitest（node + browser 双模式，browser 模式跑 Playwright chromium-headless-shell）、Playwright E2E、fast-check（属性测试）、husky + lint-staged。
- **开发环境**：`flake.nix` 提供 Nix 开发环境；`.nvmrc` 锁 Node 22.22.1。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `electron/` | 主进程：main.ts、窗口管理（windows.ts）、IPC handlers、native-bridge 服务层、原生 helper 源码与桥接适配 |
| `electron/native/screencapturekit/` | macOS Swift helper 源码（ScreenCaptureKit 录制 + 光标遥测两个可执行） |
| `electron/native/wgc-capture/` | Windows C++ helper 源码（wgc_session、cursor-sampler、wasapi_loopback、dshow_webcam、mf_encoder） |
| `electron/native-bridge/` | 主进程侧统一原生桥：adapter（光标遥测）、services（cursor/project/system）、store |
| `src/`（渲染端） | React 应用：launch 录制入口、video-editor 编辑器、lib 核心算法库、hooks（useScreenRecorder 等）、i18n |
| `src/lib/exporter/` | 导出引擎：帧渲染、音视频编解码、封装、GIF（约 5000 行） |
| `src/lib/captioning/` | 端侧字幕：音频抽取、Whisper 转写 worker、字幕段生成 |
| `src/lib/cursor/` | 光标路径平滑（240Hz 预计算弹簧阻尼）、光标主题 |
| `src/components/video-editor/` | 编辑器 UI 与时间线（VideoEditor.tsx 2961 行、VideoPlayback.tsx 2167 行） |
| `docs/` | 架构文档（native-bridge.md）、macOS/Windows 原生录制路线图、测试文档 |
| `scripts/` | 原生 helper 构建（macos-screencapturekit / windows-wgc）、字幕模型下载、i18n 校验、各类平台测试脚本 |
| `tests/e2e/` | Playwright E2E（GIF 导出、Windows 原生清单） |

组织方式为单包 Electron 应用（非 monorepo），但内部按"主进程服务层 / 渲染端算法库 / 平台原生 helper"三层清晰切分；`docs/architecture/native-bridge.md` 明确了分层原则：原生能力统一走版本化契约（`src/native/contracts.ts`）+ 单一 IPC 通道 `native-bridge:invoke`，渲染端只消费 `src/native/client.ts`。

## 5. 核心架构与关键实现

### 5.1 屏幕捕获管线（平台原生 helper + Electron 兜底）

三条路径（`package.json` build 脚本、`docs/engineering/*roadmap*.md`、`src/src/hooks/useScreenRecorder.ts`）：

- **macOS 原生**：Swift helper（`electron/native/screencapturekit/Sources/OpenScreenScreenCaptureKitHelper/main.swift`）用 ScreenCaptureKit 采集显示器/窗口，通过 stdin/stdout JSON 契约接收录制请求（`schemaVersion: 1`，见 `src/src/lib/nativeMacRecording.ts` 的请求/事件类型定义），直接输出 MP4 并隐藏系统光标（光标改由渲染层绘制可编辑版本）；另有 `OpenScreenMacOSCursorHelper` 负责高精度光标遥测。构建脚本 `scripts/build-macos-screencapturekit-helper.mjs` 在 `build:mac` 时编译打包。
- **Windows 原生**：C++ helper（`electron/native/wgc-capture/src/main.cpp` 的 CaptureConfig）用 Windows.Graphics.Capture 抓屏、WASAPI loopback 抓系统声、DirectShow 抓摄像头、Media Foundation 编码（`mf_encoder.cpp`）；`cursor-sampler.cpp` 采集光标位置与点击。构建走 CMake（`scripts/build-windows-wgc-helper.mjs`），package.json 里为它配了 8 个细分测试脚本（test:wgc-full:win 等）。
- **Electron 兜底/Linux**：`useScreenRecorder.ts`（1686 行）走 `navigator.mediaDevices.getDisplayMedia`（主进程 `main.ts` 用 setDisplayMediaRequestHandler 供源，Wayland 下启用 PipeWire/WebRTC 采集），MediaRecorder 编码 WebM，**分块经 IPC 流式写盘**——`electron/ipc/recordingStream.ts` 的 RecordingStreamRegistry 逐 chunk 追加写文件，注释明确这是 #616 的修复："a long recording never buffers the whole video in the renderer"。

### 5.2 光标遥测与自动缩放（Screen Studio 核心卖点的实现）

- **采集**：录制期间原生 helper 按 `CursorTelemetryPoint { timeMs, cx, cy }`（归一化坐标，`src/src/lib/cursorTelemetryBuffer.ts`）产出遥测流，主进程用有界环形缓冲 + FIFO 批次（按 recordingId 丢弃/重试）持久化，点击时间戳单独记录。读取经 native-bridge 的 CursorService（`electron/native-bridge/services/cursorService.ts`）。
- **自动生成缩放区域**：`src/src/components/video-editor/timeline/zoomSuggestionUtils.ts` 的 `buildAutoZoomSuggestions` 是纯函数——从遥测中检测"驻留"（光标移动距离 ≤0.02 的连续段，时长 450–2600ms 算候选），按驻留时长排序、强制相邻建议间隔 ≥1800ms、避开已有缩放区域，产出一组 `{span, focus}` 建议。该函数同时服务"魔法棒"一键加缩放和载入时的自动建议。
- **平滑运动**：目标是"预览与导出完全一致"。缩放相机对缓动目标做弹簧追踪（`videoPlayback/zoomSpring.ts`）：每轴一个 spring，含"移动目标过冲钳制"——步进越过目标即贴合并清零速度，避免果冻感；弹簧参数固定 stiffness 320 / damping 40 / mass 0.92（`motionSmoothing.ts` getZoomSpringConfig）。光标跟随用距离自适应指数平滑并按内容时间校正（`cursorFollowUtils.ts` advanceFollowFocus），保证暂停时相机静止、预览与导出收敛一致。光标路径另有离线预平滑：`src/src/lib/cursor/cursorPathSmoothing.ts` 先重采样到 240 步/秒再跑一次弹簧阻尼并预计算整条路径（"deterministic, so preview and export match exactly"）。

### 5.3 导出编码链

`src/src/lib/exporter/videoExporter.ts` 编排全流程：

1. **解码**：`streamingDecoder.ts`（779 行，基于 web-demuxer/mediabunny）流式解码源视频与摄像头视频，配 `TimestampedVideoFrameQueue` 做帧对齐与背压。
2. **渲染**：`frameRenderer.ts`（1140 行）跑离屏 PixiJS Application，逐帧合成：缩放弹簧变换、壁纸背景 + 圆角阴影、原生光标贴图（含点击弹跳、运动模糊）、摄像头 PiP 布局与反应式缩放、文字/箭头/图片标注；运动模糊用 pixi-filters 的 MotionBlurFilter；3D 旋转走独立的 WebGL2 着色器 pass（`threeDPass.ts`，自定义 vertex/fragment shader，CSS 约定的旋转矩阵与预览对齐）。
3. **编码**：WebCodecs VideoEncoder 输出 AVC；音频由 `audioEncoder.ts`（825 行）处理——优先 AAC（mp4a.40.2）128kbps、不支持时回退 Opus，含变速区域的音频重采样与混音；设 15s 编码器卡死超时与 20s flush 超时兜底。
4. **封装**：`muxer.ts` 用 mediabunny 的 Mp4OutputFormat（fastStart in-memory）合成 MP4。无任何编辑（默认裁剪/无缩放/无变速等）时走 source-copy 快路径直接 remux（`isSourceCopyFastPathEligible`）。GIF 导出独立走 `gifExporter.ts`（gif.js）。

整条链路完全在客户端（渲染进程 WebCodecs + Worker），无服务端参与，与"无水印、可商用、无订阅"的定位互为支撑。

### 5.4 端侧自动字幕

`src/src/lib/captioning/`：先抽单声道 16kHz（`extractMono16k.ts` / web-demuxer 变体），在 Web Worker 里调 Transformers.js 的 Whisper pipeline（`transcribeCore.ts`，纯函数化以便独立单测）；长音频按 12 分钟切片限制 WASM 内存、尾部切片补零并钳制时间戳；转写出的带时间戳文本段经 `annotationsFromCaptions.ts`（604 行）对齐裁剪区域后变成字幕标注。全程离线，README 强调"generated on-device with no upload (works offline)"。模型文件由 `scripts/fetch-caption-model.mjs` 预下载（CI 里缓存以避免 HuggingFace 限流，见 build.yml）。

## 6. 构建与运行

来源：`package.json` scripts 与 README。

```bash
npm ci
npm run dev              # Vite 开发服务器（vite-plugin-electron 联动主进程）
npm run build            # tsc && vite build && electron-builder（跨平台）
npm run build:mac        # 先编 Swift helper，再打包 macOS
npm run build:win        # 先编 WGC helper，再打包 Windows（--config.npmRebuild=false）
npm run build:linux      # AppImage / deb / pacman
npm run test             # Vitest node 模式
npm run test:browser     # Vitest browser 模式（需先 test:browser:install 装 chromium-headless-shell）
npm run test:e2e         # Playwright E2E
npm run lint             # Biome check
npm run i18n:check       # 13 语言包完整性校验
```

安装分发：macOS Homebrew cask（公证签名）/dmg，Windows winget（`SiddharthVaddem.OpenScreen`）/exe，Linux deb/pacman/AppImage（README Installation 节）。

## 7. 测试与工程质量

- **测试规模**：35 个测试文件（`*.test.ts(x)` + 2 个 e2e spec）。单元测试覆盖算法纯函数层：缩放弹簧、路径平滑、音频编码、流式解码、字幕对齐、遥测缓冲、合成布局等；browser 模式（vitest.browser.config.ts）在真实浏览器里跑帧渲染/GIF/导出器测试；E2E 仅 2 条（GIF 导出、Windows 原生功能清单）。
- **平台专项**：Windows 侧有大量手动/半自动测试脚本（scripts/test-windows-wgc-helper.mjs 支持 --window/--system-audio/--microphone/--webcam 组合，docs/testing/windows-native-cursor.md、macos-native-cursor.md 为验收清单文档）。
- **CI**（`.github/workflows/ci.yml`）：PR/push 到 main 触发四个 job——lint（Biome）、typecheck（tsc --noEmit）、test（node + browser 双跑）、build（electron-builder 打包验证）。另有 build.yml 手动触发的全平台打包、homebrew/winget/nix 版本升级自动化、Discord 公告 workflow。
- **代码质量观察**：注释密度高且多解释"为什么"（如 recordingStream.ts 对 #616 修复、zoomSpring.ts 对过冲钳制的动机说明）；算法核心普遍纯函数化并配套单测。但巨型文件突出：VideoEditor.tsx 2961 行、VideoPlayback.tsx 2167 行、useScreenRecorder.ts 1686 行、audioEncoder.ts 825 行，作者自己也承认"not production grade"。仓库无 CHANGELOG 文件，版本说明依赖 GitHub Releases。

## 8. 社区与维护状态

- **活跃度**：2025-10 创建后一年内 99 名贡献者、4 万星；发版节奏 2026 年 3–6 月约每月一个 minor（v1.3.0 → v1.5.0，gh api releases）。**代码活跃期止于 2026-06-17**（最后一次实质提交为 README 更新），此后仅 2026-10-17 一条 "update spinoff github repo" 完成归档动作。
- **归档状态**：gh api `archived: true`。README 顶部归档声明指向社区 fork `getopenscreen/openscreen` 继续"以他们自己的方式"维护。归档具体原因仓库内无官方说明，**未确认**；仓库近期 events 仅剩 Watch/Fork（2026-11 仍在涨星，说明需求仍在）。
- **发布与渠道**：13 个 release，自动化分发覆盖 Homebrew cask、winget、Nix（.github/workflows/ 下对应 update-homebrew-cask.yml、publish-winget.yml、bump-nix-package.yml），macOS 构建做了公证（README 提及 notarized signature）。
- **协作规范**：CONTRIBUTING.md 为标准 fork-branch-PR 流程；有 CODEOWNERS、PR 模板与结构化 issue 模板（bug_report.yml / feature_request.yml）。

## 9. 亮点与风险

**亮点**

1. 把 Screen Studio 的核心体验（光标跟随自动缩放 + 平滑运动 + 光标美化 + 运动模糊）完整开源，且预览与导出共用同一套确定性弹簧算法（`zoomSpring.ts`、`cursorPathSmoothing.ts`），"所见即所得"不是口号而是架构约束。
2. 采集层敢于下沉到平台原生：Swift ScreenCaptureKit 与 C++ WGC/WASAPI/MF 双 helper，系统光标排除、系统声采集、多源时钟对齐这些 Electron 采集做不好的点都有工程解，并保留 Electron 路径兜底（含 Linux Wayland/PipeWire）。
3. 全链路客户端化：录制分块流式写盘、导出流式解码 + WebCodecs、字幕端侧 Whisper，无服务器、无上传、无水印，MIT 许可对商业使用零限制。
4. 工程自动化成熟：13 语言 i18n 校验、三平台原生 helper 构建/测试脚本化、CI 四道闸门、三渠道包管理器自动升级。
5. 文档驱动：`docs/` 下有原生桥架构文档与两份平台录制路线图（目标/非目标/架构边界），对后来接手者（如 getopenscreen fork）非常友好。

**风险/局限**

1. 已归档、不再维护：macOS 系统升级可能随时打破 ScreenCaptureKit helper，bug 无人修；官方续命在第三方 fork，长期生命力存疑。
2. 作者自评"not production grade"；最大三个文件近 7000 行（VideoEditor/VideoPlayback/useScreenRecorder），二次开发门槛高。
3. 源码构建依赖平台工具链（Swift 5.9 + CMake/MSVC），非原生层贡献者难以在单机完成双平台构建。
4. E2E 覆盖薄（仅 2 条 spec），Windows 原生路径大量依赖手动验收清单，自动化回归不足以保护如此复杂的采集矩阵。

## 10. 总结评价

OpenScreen 是"一年生命周期的爆发型开源明星"样本：单人发起、切中 Screen Studio 高定价的真实痛点、用原生采集 + 确定性动画引擎把"录屏美化"做到接近商业产品，一年吸 4 万星；又在爆发后一年内主动归档并指路社区 fork。对用户，它是至今仍可安装使用的免费 Screen Studio 替代（v1.5.0）；对开发者，它是学习"Electron + 平台原生捕获 + WebCodecs 导出 + 端侧 Whisper"完整桌面媒体管线的极佳教材——native-bridge 分层、遥测缓冲、弹簧平滑预览/导出同源这些设计都值得精读。适合想构建同类录制工具或研究桌面媒体工程的工程师读源码；生产环境长期使用则应转向社区 fork 并自担维护风险。
