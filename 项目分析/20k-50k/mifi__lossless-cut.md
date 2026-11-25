# mifi/lossless-cut 项目分析

> 星数 40500（快照 2026-10-24）｜主语言 JavaScript (TypeScript)｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：基于 FFmpeg `-c copy`（流复制、不重编码）的跨平台无损视频/音频剪辑 GUI，"无损剪辑界的瑞士军刀"。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | LosslessCut（lossless-cut） | `package.json` |
| 版本 | 3.70.2 | `package.json` |
| 星数 | 40500（快照）；2026-11-05 实测 gh api 为 45,713，仍在快速上涨 | gh api `repos/mifi/lossless-cut` |
| 语言 | TypeScript（全部 170 个 ts/tsx 文件约 2.87 万行） | `find src` 统计 |
| 许可证 | GPL-2.0-only | `package.json`、`LICENSE` |
| 创建/最近 push | 2016-10-30 / 2026-11-03（分析前 2 天仍在提交） | gh api |
| 最新 release | v3.70.2，2026-10-28 发布，16 个资产 | gh api `releases/latest` |
| 主作者 | Mikael Finstad（mifi），单人维护 + 捐助/商店付费版变现 | `package.json` author、`README.md` |
| 仓库形态 | Electron 桌面应用单体仓库，内嵌 Astro 官网子目录 | `website/` |

## 2. 项目是什么，解决什么问题

LosslessCut 解决的核心痛点是"只想剪掉几 GB 视频里的废镜头，却要为此付出整段重编码的时间和画质损失"。它把 FFmpeg 的 stream copy 能力包装成可视化 GUI：切割/合并不重新编码，只重新封装媒体包，因此速度快（接近磁盘拷贝速度）且零画质损失（`README.md`："Everything is extremely fast because it does an almost direct data copy, fueled by the awesome FFmpeg"）。

典型场景是粗剪相机、GoPro、无人机拍下的大文件：标记保留片段 → 无损导出 → 丢掉剩余部分。除切割外还支持：无损合并（要求编解码一致）、添加/删除/提取音轨字幕轨、remux 换容器（如 MKV↔MP4）、无损裁剪（通过 H264/HEVC 码流滤镜改 SPS 元数据而非重编码）、场景/黑帧/静音检测自动分段、关键帧跳转、帧导出、CSV/EDL 等十余种剪辑决策清单（EDL）格式的导入导出（`docs/features.md`、`src/renderer/src/edlFormats.ts`）。

一个聪明的折中是"FFmpeg 辅助播放"：Chromium 只能原生解码 MP4/WebM 等少数格式，遇到不支持的文件时，主进程用 ffmpeg 实时转码成 h264/aac 分段 MP4 通过 MediaSource 喂给播放器，仅用于预览——导出仍从原始文件流复制，保证无损（`src/main/ffmpeg.ts` 的 `createMediaSourceProcess`、`src/main/compatPlayer.ts`）。

作者坚持"GitHub 发布免费 + 商店付费"的双轨模式：Mac App Store / Microsoft Store / Snap 上的版本收费，GitHub Releases 手动下载永久免费（`README.md` Download 一节）。项目由作者一人维护，`AGENTS.md` 显示其日常开发已深度使用 AI agent（提交规范、发布流程均为 agent 编写操作指引）。

## 3. 技术栈与依赖

- **框架**：Electron 45 + React 19，构建用 electron-vite 5（Vite 7），主/preload 进程 target node24.22，渲染进程 target chrome156（`package.json`、`electron.vite.config.ts`）
- **语言与质量工具**：TypeScript ~6.0（`@tsconfig/strictest`）、oxlint（type-aware，`--deny-warnings`）、vitest 4、Yarn 4.18（Berry，含 patch 协议）（`package.json` scripts）
- **ffmpeg 交互**：execa 9 驱动随应用分发的 ffmpeg/ffprobe 二进制（按平台从 `script/downloadFfmpeg.ts` 下载打包，Linux 下注入 `LD_LIBRARY_PATH`）
- **UI 库**：@radix-ui/themes、react-icons、sweetalert2、motion、@dnd-kit、@tanstack/react-virtual、leaflet/react-leaflet（GPS 轨迹显示）
- **主进程侧**：express 5（本地 HTTP API）、winston（日志）、electron-store 5（设置）、zod 4（`--settings-json` 校验）、mitt（事件总线，yarn patch 版）
- **i18n**：i18next 26 + react-i18next，36 种语言，翻译文件经 i18next-cli 扫描提取，打包为 extraResources（`locales/`、`package.json` build.extraResources）
- **打包**：electron-builder 26，mac dmg+mas(universal)、win 7z+appx、linux tar.bz2/AppImage/snap，启用 electron fuses（禁 runAsNode、asar 完整性校验）与 macOS 公证（`package.json` build 段）

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `src/main/` | Electron 主进程：ffmpeg 进程封装、HTTP API、菜单、i18n、更新检查、兼容播放器（25 个文件） |
| `src/preload/` | 预加载脚本：Proxy 形式的 RPC 桥（仅 2 个文件） |
| `src/common/` | 主/渲染共享：类型、常量、ffprobe 类型定义、公共工具 |
| `src/renderer/src/` | React 应用：App.tsx（2882 行总控）+ 23 个 hooks + components/dialogs/util/worker |
| `locales/` | 36 种语言的 translation.json |
| `script/` | 构建/发布辅助脚本（下载 ffmpeg、生成图标/文档/版本说明、上传产物） |
| `versions/` | 每个版本的 What's New 说明（供应用内"新功能"弹窗） |
| `website/` | Astro 官网，部署到 Cloudflare（wrangler.jsonc） |
| `.github/workflows/` | build/test/website/store-release 共 5 个工作流 |
| `docs/`、`docs.md` | 面向用户的文档源（发布到 losslesscut.app） |

组织方式评述：标准 Electron 三进程（main/preload/renderer）划分 + common 共享层的单体结构，无 monorepo 工具但内嵌独立打包的 website 子项目。代码量高度集中：`App.tsx`（2882 行）、`useFfmpegOperations.ts`（1166 行）、`useSegments.tsx`（1027 行）三个文件承载了主要业务逻辑，属于"重 hook + 单一大组件"风格。

## 5. 核心架构与关键实现

### 5.1 ffmpeg 进程层（src/main/ffmpeg.ts，738 行）

主进程用 execa 统一驱动 ffmpeg/ffprobe：所有运行中的进程记录在 `runningFfmpegs` Set 中，每个绑一个 AbortController，用户取消导出时 `abortFfmpegs()` 全部中止；进度通过 readline 逐行解析 stderr 的 `time=` 行（`parseFfmpegProgressLine`）。关键设计：

```ts
const runningFfmpegs = new Set<{ process, abortController }>();
function runFfmpegProcess(args, customExecaOptions, additionalOptions) {
  const process = execa(ffmpegPath, args, getExecaBufferOptions({ ...customExecaOptions, cancelSignal: abortController.signal }));
  ...
}
```

波形渲染是"双 ffmpeg 管道"：进程 1 解码音频（可选重采样至 10kHz）输出 matroska 到 stdout，管道接入进程 2 的 `showwavespic` 滤镜生成 2000x300 PNG（`renderWaveformPng`）。黑帧/静音/场景检测则用 `-f null -` + filter 输出行解析出时间段（`detectIntervals`、`detectSceneChanges`）。

### 5.2 无损切割命令构造（src/renderer/src/hooks/useFfmpegOperations.ts，1166 行）

`losslessCutSingle` 是核心：不重编码，只组合 `-ss`/`-t`/`-map`/`-c copy`（c copy 由 `-map` + 不指定编码器隐式实现）。两种切割模式：关键帧切割（`-ss` 放 `-i` 前，瞬间 seek 但对齐关键帧）与精确切割（`-ss` 放 `-i` 后，丢起始非关键帧包，起点精确但起始几帧可能灰屏）。周边细节极多且都有 issue 引用佐证：Matroska B 帧的 `-itsoffset` seek 补偿（#1216）、音频-only 导出加 `-copyinkf` 防起点后移、`-avoid_negative_ts` 按封面流降级（#3009）、mov/mkv muxer 私有 flag 只对对应容器注入。无损裁剪/改宽高比不重编码的秘诀是码流滤镜：

```ts
if (codecName === 'h264') bitstreamFilters.push(`h264_metadata=${cropParams}`); // crop_left/right/top/bottom
else if (codecName === 'hevc') bitstreamFilters.push(`hevc_metadata=...`);
...
ret.push(`-bsf:${outputIndex}`, bitstreamFilters.join(','));
```

`concatFiles` 用 concat demuxer，片段清单文本通过 stdin 管道喂入（`-protocol_whitelist file,pipe,fd`），避免命令行长度限制。多段导出 `cutMultiple` 用 pMap 串行执行并聚合进度。

### 5.3 Smart Cut（src/renderer/src/smartcut.ts + useFfmpegOperations.ts）

"智能切割"解决精确起点与关键帧对齐的矛盾：先用 ffprobe `-show_packets` 读目标时间附近的包，筛出关键帧（`readKeyframesAroundTime`，10 秒窗口找不到扩大到 60 秒）；若起点恰在关键帧上则纯流复制；否则把"起点→下一个关键帧"的一小段用与源一致的编码参数重编码，"关键帧→终点"段流复制，最后 concat 拼接（`needsSmartCut` → `cutEncodeSmartPart` → `concatFiles`）。若整个片段落在两个关键帧之间则整段重编码。编码时强制 `-bf` 与源一致以保证拼接处 B 帧连续。

### 5.4 进程间通信与扩展接口

preload 用 Proxy 把所有方法调用转发为 `ipcRenderer.invoke('__electron_rpc__', method, args)`，主进程按方法名分发，避免为每个 API 手写 handler（`src/preload/index.ts`）。另内置 express 本地 HTTP API：`POST /api/action/:action` 触发任意键盘快捷键动作，`POST /api/await-event/:eventName` 阻塞等待应用事件，带 host/origin 白名单防止浏览器页面远程调用（`src/main/httpServer.ts`）。这让 LosslessCut 可被脚本/快捷工具自动化编排。

### 5.5 数据可视化数据流

时间线（`Timeline.tsx`）的波形/缩略图/关键帧条均按需懒加载：`useWaveform.ts` 调 mainApi → 主进程 `renderWaveformPng`（整条音频 resample 10000 后一次出图）；`useThumbnails.ts` 批量抽帧；`useKeyframes.ts` 缓存 ffprobe 包扫描结果（带 `createdAt` 时间戳支持失效重读）。剪辑片段/标记模型在 `segments.ts`（nanoid 标识 + 标签 + 颜色索引），持久化为 `.llc` 项目文件（JSON5），并支持 14+ 种 EDL 格式互转（`edlFormats.ts`：CMX3600、FCP XML、Xmeml、cue、PBF、YouTube、SRT、DJI GPS、CSV/TSV、DV Analyzer、cutlist、mplayer EDL）。

## 6. 构建与运行

来自 `package.json` scripts 与 `CONTRIBUTING.md`（Yarn 4）：

```bash
yarn install                # postinstall 会 electron-builder install-app-deps
yarn dev                    # electron-vite dev -w 启动开发模式
yarn test                   # vitest
yarn tsc                    # tsc --build 全量类型检查
yarn lint                   # oxlint --type-aware --deny-warnings
yarn build                  # generate-icon + electron-vite build
yarn pack-mac|pack-win|pack-linux   # electron-builder 打包
yarn download-ffmpeg-darwin-arm64   # 按平台下载 ffmpeg 到 ffmpeg/<platform>-<arch>/
yarn check                  # tsc+lint+test+build+license+i18n 扫描+dedupe+docs 一键全套
```

注意：ffmpeg 二进制不入库，需先跑对应平台的 `download-ffmpeg-*` 脚本；`generate-docs` 的产物受 CI 检查（`.github/workflows/test.yml` 的 "Check docs" 步骤 `git diff --quiet`）。

## 7. 测试与工程质量

- **测试**：14 个 `*.test.ts`，全部为 vitest 单元测试，集中在纯逻辑模块：进度行解析、EDL 格式解析、时间段运算、流映射、码率计算、时间码等（`src/main/*.test.ts`、`src/renderer/src/**/*.test.ts`），另有 `__snapshots__` 快照与 `test-manual/` 手动测试说明。UI/ffmpeg 集成路径无自动化测试（未确认有 e2e）。
- **CI**（`.github/workflows/`）：`test.yml` 在 ubuntu+windows 双平台跑 test/tsc/lint/build/docs-diff/license 全链；`build.yml` 由 `v*` tag 触发，macos/ubuntu/windows-2022/windows-11-arm 四矩阵构建，产出 draft release，并有 `check-release-assets` job 校验资产齐全且体积不比上一版小 10%（防止打出缺 ffmpeg 的坏包）——这个防呆设计少见且实用。
- **代码质量观察**：strictest tsconfig + type-aware oxlint 全开并拒绝警告；源码内大量引用 GitHub issue 的注释解释"为什么"，可维护性好；`check-licenses` 保证依赖许可证合规（GPL 兼容）；i18n 键值扫描纳入 CI 防漏译。弱点：巨型文件（App.tsx 2882 行）状态耦合较重。

## 8. 社区与维护状态

- **活跃度**：2016 年创建至今持续开发，最近 push 2026-11-03；版本节奏为每月 1-3 个 minor/patch（`versions/` 目录已有 3.68.0~3.70.2 等近期版本说明）。open issues 160、forks 2305（gh api）。
- **维护结构**：单人主导（mifi），README 明言"maintained by me alone"；发布流程（`RELEASING.md`）高度流程化：GitHub 先发 → 观察 regression → 延后推 Mac App Store/Microsoft Store，`stores/apple`、`stores/microsoft` 分支存放商店当前在线版本以便商店级热修；nightly 构建发布在 mifi.no。release notes 由 commit 的 `Changelog:` trailer 自动收集（`AGENTS.md`、`script/releaseNotes.ts`）。
- **贡献者情况**：具体 contributor 数量未确认（tarball 无 git 历史）；从 AGENTS.md 可见开发工作流已包含 AI agent 的职责边界（可准备发布、不可擅自发布/发商店）。
- **商业模式**：商店版付费 + 捐助（PayPal/crypto），GitHub 版永久免费，资金反哺全职开发（`README.md`）。

## 9. 亮点与风险

**亮点**
1. 无损理念贯彻到底：切割/合并/remux/裁剪/改宽高比全部避开重编码，裁剪用 `h264_metadata` 码流滤镜改 SPS 而非像素操作（`useFfmpegOperations.ts`），是学习"容器与码流层能做什么"的最佳范例代码。
2. Smart Cut 设计优雅：精确起点与关键帧约束的两难被拆成"编码头部小段 + 复制主体 + concat"三步，编码参数自动匹配源文件（`src/renderer/src/smartcut.ts`）。
3. ffmpeg 工程化细节密度极高：进度解析、进程取消、双进程波形管道、LD_LIBRARY_PATH 注入、Windows 参数转义（#2151）、ffprobe 超时保护——几乎每行都有 issue 链接佐证，堪称 Electron 调用 ffmpeg 的百科全书。
4. 发布工程成熟：四平台 CI 矩阵、资产完整性校验 job、商店分轨分支、GPG 签名 tag、AI agent 友好的 RELEASING.md。
5. EDL 互操作广：14+ 种格式（含 DJI GPS 轨迹、YouTube 时间戳）导入导出，切入专业剪辑工作流。

**风险/局限**
1. 无损切割天然受关键帧约束，非关键帧起点只能 Smart Cut（仅支持单视频流文件）或接受灰帧，这是模式本身的用户预期管理问题（`smartcut.ts` 中 `Can only smart cut video with exactly one video stream`）。
2. 单人维护风险：核心逻辑集中在 3 个巨型文件，bus factor 为 1。
3. Chromium 解码能力墙：预览依赖 HTML5 video 或 ffmpeg 转码，H265/ProRes 等格式预览体验依赖硬件解码与 colorspace 补丁（`createMediaSourceProcess` 里大段 bt709 兼容注释），修复跟随 Electron bug（#47947）。
4. GPL-2.0 + 商店分发的组合对再分发者有传染性约束；仅捆绑特定版本 ffmpeg，用户自带旧版 ffmpeg 不受支持（`AGENTS.md` FFmpeg 节）。

## 10. 总结评价

LosslessCut 是"把一个命令行原语（ffmpeg -c copy）做成极致产品"的典范：范围极窄（无损操作）却靠工程细节和场景外延（EDL、检测、GPS、HTTP API）做到 4 万+ 星。代码是 TypeScript + Electron 中大型应用的优秀学习样本——尤其是 ffmpeg 进程管理、命令构造的防御性细节和发布流水线；对普通用户则是相机/无人机素材粗剪的效率神器。适合想学 Electron 三进程架构、FFmpeg 封装工程化、或"个人开源项目如何商业化运营"（GitHub 免费 + 商店付费 + 捐助）的读者精读；不适合想学习组件化/状态管理最佳实践的人（大文件单体风格）。
