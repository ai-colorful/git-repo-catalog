# ShareX/ShareX 项目分析

> 星数 35500（快照 2026-10-24）｜主语言 C#｜档位 20k-50k｜分析日期 2026-11-14
> 注：撰写时 gh api 实测星数已升至 41016，默认分支 develop，最近 push 2026-11-13。

## 1. 项目速览

一句话定位：Windows 平台功能最全的开源截图/录屏/文件分享/效率工具箱，围绕"热键触发 → 捕获 → 链式后期任务 → 上传分享"的工作流引擎构建。

| 项 | 值 |
|---|---|
| 名称 | ShareX/ShareX |
| 星数 | 41016（gh api 2026-11-14 实测；任务快照 35500） |
| 语言 | C#（.NET 10，`Directory.build.props`/各 csproj） |
| 许可证 | GPL-3.0（`LICENSE.txt`） |
| 版本 | 22.4.0（`Directory.build.props`；最新 release v22.4.0 发布于 2026-11-08） |
| 默认分支 | develop |
| 平台 | Windows x64 / ARM64（Steam、Microsoft Store、便携版多渠道分发） |
| 代码规模 | 约 3500 个 .cs 文件、约 31.7 万行（各项目统计合计，见第 4 章） |

## 2. 项目是什么，解决什么问题

ShareX 是 Windows 上的"截图 + 录屏 + 文件分享 + 效率工具"一体化桌面应用（`README.md`）。它要解决的核心问题是：截图这件事在大多数工具里止步于"截下来"，而真实工作流需要"截下来 → 标注脱敏 → 保存 → 上传 → 拿到可分享链接"的完整链条自动化。ShareX 把这条链条做成可配置引擎：每个热键绑定一种捕获方式与一组链式动作，捕获完成后自动执行复制/保存/上传/短链等步骤（`README.md` "Custom Workflows and Hotkeys"、`ShareX/Enums.cs` 的 `AfterCaptureTasks`/`AfterUploadTasks`）。

捕获形态覆盖全屏、活动窗口、活动显示器、区域、滚动截图、自定义区域、上次区域、自动连续捕获九类（`ShareX/CaptureHelpers/` 下 11 个捕获类），区域选择内嵌标注工具（矩形/箭头/文字/步号/模糊/像素化/聚光灯等，`README.md` "Region Capture and Annotation"）。录屏支持视频与 GIF，且已从 FFmpeg 外部进程迁移到 GPU 原生管线（见第 5 章）。

第二个核心是"上传到任何地方"：内置约 60 个具名上传目的地（图床/网盘/短链/分享服务，统计见第 5 章），并提供 JSON 语法的自定义上传器（Custom Uploader），让任意 HTTP API 都能接入（`ShareX.UploadersLib/CustomUploader/`）。上传后可自动复制 URL、生成二维码、用短链服务再加工。

第三个核心是"不止截图"的工具箱：31 个子工具（颜色拾取、标尺、贴图 Pin to Screen、OCR、二维码、哈希校验、视频转换/裁剪/缩略图、去背景等，`ShareX.Tools/Tools/` 目录、`README.md` "Productivity Tools"），使 ShareX 成为开发/设计/客服/写作人群的桌面常驻工具。

项目始于 2007 年（`Directory.build.props` Copyright 2007-2026；仓库创建于 2013-10-08，gh api），是 Windows 开源截屏领域事实上的功能标杆，星数长期位居 C# 桌面应用前列。

## 3. 技术栈与依赖

- 语言与运行时：C# / .NET 10。主程序 `net10.0-windows10.0.22621.0`，跨平台编辑器与 Avalonia 项目为 `net10.0`（`ShareX/ShareX.csproj`、`ShareX.ImageEditor/ShareX.ImageEditor.csproj`、`ShareX.Avalonia/ShareX.Avalonia.csproj`）。
- UI：正在从 WinForms 迁移到 **Avalonia 12.1.6**——主程序仅剩 `ShareX/Forms/MainForm.cs` 一个 WinForms 窗体，27 个 `.axaml` 窗口已在 `ShareX/Presentation/` 下按功能域组织（MainWindow/TaskSettings/HotkeySettings/ScreenRecording 等）。
- 图像管线：**SkiaSharp 3.119.4** 全面替代 GDI+（全仓 489 个 .cs 文件引用 SkiaSharp；`ShareX/TaskManager.cs` 直接 `using Bitmap = SkiaSharp.SKBitmap`）。
- GPU/媒体：**Vortice.Direct3D11 / Direct2D1 / MediaFoundation 3.8.3**（原生录屏、HDR 色调映射、相机叠加；`ShareX.ScreenRecordingLib/Video/`、`Encoding/`）。
- MVVM：CommunityToolkit.Mvvm 8.4.2（ImageEditor 内 17 处 ObservableObject 派生）。
- 上传与网络：Newtonsoft.Json 13.0.4、FluentFTP 55.1.0、SSH.NET 2026.0.0（`Directory.Packages.props`）。
- OCR/视觉：Microsoft.ML.OnnxRuntime.DirectML 1.24.4（Background Remover）、PaddleOCR（`Licenses/PaddleOCR_license.txt`、`ShareX.OCR/`）；ZXing.Net 0.16.11（二维码）。
- 历史存储：Microsoft.Data.Sqlite 11.0.0（`ShareX.HistoryLib/Managers/HistoryManagerSQLite.cs`）。
- FFmpeg 9.0：外部 exe，CI 打包（`.github/workflows/build.yml` 第 44、115-127 行），现仅用于视频工具与旧录屏路径。
- 构建配置：x64/ARM64 两平台 × Release/Debug/Steam/MicrosoftStore/MicrosoftStoreDebug 五配置（`ShareX.sln` GlobalSection）。

## 4. 目录结构与代码组织

单仓库多项目解决方案（`ShareX.sln` 共 18 个工程 + 独立 `ShareX.ImageEditor.sln`），按功能域分层：

| 目录 | 职责 | 规模（.cs 文件/行数） |
|---|---|---|
| `ShareX/` | 主程序：任务引擎、热键、设置、捕获编排 | 138 / 42579 |
| `ShareX.HelpersLib/` | 通用库：剪贴板、输入、原生 API、GIF、本地化 | 180 / 37755 |
| `ShareX.UploadersLib/` | 全部上传目的地与自定义上传器 | 185 / 27134 |
| `ShareX.ScreenCaptureLib/` | 区域捕获窗口、滚动截图、旧 FFmpeg 录屏 | 36 / 10936 |
| `ShareX.ScreenRecordingLib/` | 新原生录屏（MF/D3D11/WGC） | 31 / 4012 |
| `ShareX.ImageEffectsLib/` | 图像效果滤镜库 | 69 / 7096 |
| `ShareX.HistoryLib/` | 任务历史（SQLite/JSON/XML 三后端 + Mock） | 23 / 7144 |
| `ShareX.ImageEditor/` | 新 Avalonia 图像编辑器（最大模块） | 437 / 78997 |
| `ShareX.ImageEditor.App/` | 编辑器独立应用壳 | — / — |
| `ShareX.PhotoEditor/` | 图层式照片编辑器 | 124 / 38618 |
| `ShareX.PixelEditor/` | 像素级编辑器 | 50 / 19295 |
| `ShareX.Tools/` | 31 个效率子工具 | 154 / 37741 |
| `ShareX.Avalonia/` | Avalonia 通用控件/主题/输入集成 | 19 / 4381 |
| `ShareX.OCR/` | OCR 封装 | 7 / 1304 |
| `ShareX.Icons/` | 图标资源包 | — / — |
| `ShareX.Setup/` | 安装器 | — / — |
| `ShareX.Steam/` | Steam 渠道适配 | — / — |
| `ShareX.NativeMessagingHost/` | 浏览器扩展通信宿主 | 1 / 71 |

组织评述：清晰的"主程序 + 功能库 + 工具集"三层结构，库之间单向依赖；`Scripts/` 存放翻译/许可/对比度校验脚本，`.agents/skills/` 内置维护者使用的 AI agent 工作流（commit、merge-external-pr、merge-pr、sharex-changelog、sharex-resx，各含 SKILL.md）——是少数把 AI 辅助维护流程纳入版本库的大型项目。

## 5. 核心架构与关键实现

### 5.1 链式工作流引擎（TaskManager / WorkerTask）

核心是一个 Flags 枚举驱动的任务管线。捕获后动作 23 项、上传后动作 6 项，用户在热键设置里自由勾选组合：

```csharp
[Flags]
public enum AfterCaptureTasks // ShareX/Enums.cs
{
    None = 0,
    ShowQuickTaskMenu = 1,
    ShowAfterCaptureWindow = 1 << 1,
    ...
    CopyImageToClipboard = 1 << 6,
    SaveImageToFile = 1 << 9,
    DoOCR = 1 << 19,
    UploadImageToHost = 1 << 21,
    DeleteFile = 1 << 22
}
```

`ShareX/TaskManager.cs` 是静态任务队列：注册 6 个事件（StatusChanged/ImageReady/UploadProgressChanged 等）驱动 UI，`StartTasks()` 按 `ApplicationState.Settings.UploadLimit` 控制并发数，队列任务状态机为 InQueue→Preparing→Working→Stopped/Failed/Completed（`ShareX/Enums.cs` `TaskStatus`）。`ShareX/WorkerTask.cs`（1247 行）承载单任务生命周期：`DoUploadJob()` 先做大文件告警与 `BeforeUploadWindow` 确认，再按 `Info.TaskSettings.AfterCaptureJob` 标志位逐步执行；上传失败按 `MaxUploadFailRetry` 自动重试，`DoUpload()` 按 `EDataType`（Image/Text/File）分发到不同上传通道。

### 5.2 热键系统与解耦抽象

`ShareX/HotkeyManager.cs`（235 行）持有 `List<HotkeySettings>`，通过 **`IHotkeyHost` 接口**注册/注销系统热键——这是 WinForms→Avalonia 迁移的关键解耦点：热键逻辑不感知 UI 框架。触发时校验 `DisableHotkeysOnFullscreen`（检测活动窗口是否全屏则暂时忽略热键，`CaptureHelpers.IsActiveWindowFullscreen()`），每个热键绑定一个 `HotkeyType`（`ShareX/Enums.cs` 约 50 种，分 Upload/Screen capture/Tools 三类目录标注）与独立 `TaskSettings` 副本。

### 5.3 上传器体系：反射注册表 + 自定义上传器 DSL

`ShareX.UploadersLib/UploaderFactory.cs` 用反射一次性发现所有 `UploaderService<T>` 子类，按枚举键缓存为五张字典：

```csharp
public static Dictionary<ImageDestination, ImageUploaderService> ImageUploaderServices { get; } = CacheServices<ImageDestination, ImageUploaderService>();
public static Dictionary<TextDestination, TextUploaderService> TextUploaderServices { get; } = CacheServices<TextDestination, TextUploaderService>();
public static Dictionary<FileDestination, FileUploaderService> FileUploaderServices { get; } = CacheServices<FileDestination, FileUploaderService>();
```

服务规模实测（`ShareX.UploadersLib/Enums.cs`）：FileDestination 32 项（Dropbox/FTP/OneDrive/Google Drive/Box/MEGA/Amazon S3/GCS/Azure/B2/Nextcloud/Immich/Pomf/Uguu/Seafile/Plik/YouTube 等 29 个具名 + 共享文件夹/邮件/自定义），ImageDestination 8 项（Imgur/Flickr/Chevereto 等），TextDestination 12 项（Pastebin/Gist/PrivateBin 等），短链 + 分享服务合计 25 项。全库约 **73 个上传器实现类 + 75 个 Service 描述类**（基类派生 grep 统计），统一继承 `FileUploader` 抽象基类（含 OAuth1/2 体系 `ShareX.UploadersLib/OAuth/`）。`UploadersConfig.cs`（504 行、172 个属性）集中序列化全部凭据与配置。

无内置服务的目标走 **Custom Uploader**：`ShareX.UploadersLib/CustomUploader/ShareXCustomUploaderSyntaxParser.cs` 实现 JSON DSL，支持 `{variable}` 占位与函数表达式（`Functions/`），社区以此维护大量第三方上传器配置。

### 5.4 双路径录屏：原生 GPU 管线为未来，FFmpeg 为兼容

`ShareX/ScreenRecordManager.cs` 同时持有新旧两个录制器，由 `taskSettings.CaptureSettings.ScreenRecordUseNative` 切换。新管线在 `ShareX.ScreenRecordingLib/`：Windows Graphics Capture 抓帧（`Video/GraphicsCapture.cs`）→ D3D11 GPU 处理（`Video/GpuVideoProcessor.cs`、`GraphicsDevice.cs`）→ HLSL 着色器（`Video/HdrToneMapShader.cs`、`CameraCircleShader.cs`、`ShaderCompiler.cs` + `Native/D3DCompiler.cs`）→ Media Foundation 编码写出（`Encoding/MediaFoundationWriter.cs`），支持相机圆角叠加（`CameraOverlay.cs`）与 GIF（`Encoding/GifWriter.cs`）。旧路径 `ShareX.ScreenCaptureLib/ScreenRecording/ScreenRecorder.cs` 走 FFmpeg 外部进程 + 两遍编码（`ImageCache`/`HardDiskCache`）。FFmpeg exe 目前主要服务于 `ShareX.Tools/Tools/VideoConverter|VideoTrimmer|VideoThumbnailer`。

### 5.5 捕获层与多显示器

`ShareX/CaptureHelpers/` 定义 11 个多态捕获类，统一继承 `CaptureBase`：模板方法 `Capture()` → 延迟截图 → 隐藏桌面图标/主窗 → `Execute()` 抽象捕获 → 产出 `TaskMetadata`。例如 `CaptureMonitor` 直接按显示器矩形取图，多显示器支持与 DPI 适配下沉到 HelpersLib 的屏幕工具与 Avalonia 窗口层；`ShareX/Presentation/DisplayDiagnostics/DisplayDiagnosticsCollector.cs` 含 GetDpiForMonitor 相关原生调用（`app.manifest` 不再声明 dpiAware，交由 Avalonia PerMonitor 处理）。区域选择 UI 为 Avalonia 实现（`ShareX.ScreenCaptureLib/Presentation/RegionCapture/RegionCaptureWindow.axaml`）。

## 6. 构建与运行

- 构建：`dotnet build ShareX.sln`（需 .NET 10 SDK，Windows）；平台/配置矩阵为 x64|ARM64 × Release|Debug|Steam|MicrosoftStore（`ShareX.sln`、`Directory.build.props`）。
- CI 参考流程（`.github/workflows/build.yml`）：windows-latest + dotnet 10.0.x，下载 BtbN FFmpeg 9.0 构建放入输出目录，随后打包安装器。
- 运行：直接运行 `ShareX` 项目输出（依赖 ffmpeg.exe 与 Windows 10+，net10.0-windows10.0.22621.0）；普通用户建议走 GitHub Releases / Microsoft Store / Steam（`README.md` "Download ShareX"）。
- 维护脚本：`Scripts/Resx.ps1`、`Scripts/ValidateTranslations.ps1`、`Scripts/UpdateLicenseNotices.ps1`（PowerShell 7.2+）。

## 7. 测试与工程质量

- **无单元测试项目**：全仓 grep `[TestClass]`/`[Fact]`/xunit/NUnit 零命中；唯一的"测试"是 `Scripts/Test-ResxTools.ps1`——翻译资源工具链的 PowerShell 自校验脚本。质量保障依赖 CI 全配置矩阵编译通过 + 维护者手动验证。
- CI：`.github/workflows/build.yml`（5 配置 × 2 平台矩阵 + release 环境 id-token）、`pr.yml`（PR 触发同样矩阵）、`stale.yml`（过期 issue 清理）。
- 代码质量观察：GPL 头注释与 #region 规范统一（`.editorconfig`）；中央包管理 + 传递依赖锁定（`Directory.Packages.props` `CentralPackageTransitivePinningEnabled`）；近期大规模重构（WinForms→Avalonia、GDI+→SkiaSharp、FFmpeg→Media Foundation）呈现"新旧双轨并存"的过渡形态——旧录屏器、旧编辑器（PixelEditor/PhotoEditor）与新管线共存，代码总量偏大但边界清晰。

## 8. 社区与维护状态

- 活跃度：gh api 显示最近 push 2026-11-13（分析前一日），最新 release v22.4.0 发布于 2026-11-08，612 个开放 issue——处于高强度持续开发期，且正处 UI 框架换代窗口。
- 贡献者：共 **252 人**（contributors API 分页实测）；高度集中于创始人 Jaex（8401 次贡献，占绝对主导），其后 McoreD 308、L1Q 149、CubityFirst 120、sylveon 116——典型"单核心维护者 + 长尾贡献者"结构。
- 协作门槛：`.github/CONTRIBUTING.md` 明示维护者评审时间有限，"PR 不保证被评审""大功能大概率不合并，优先小而聚焦的 bug 修复"；配合 `.agents/skills/merge-external-pr/` 等内置 agent 工作流，维持单人高强度产出。
- 发布节奏：GitHub Releases + Microsoft Store + Steam 三渠道并行（`ShareX.sln` 含 Steam/MicrosoftStore 专用配置佐证）。
- 本地化：29 个 resx 翻译文件（`ShareX/Localization/`），含简体中文 `Strings.zh-CN.resx` 与繁体中文 `Strings.zh-TW.resx`；`ShareX/Enums.cs` `SupportedLanguage` 枚举列 28 种界面语言。
- **中文用户占比（客观陈述）**：抽样最近 60 天全部 264 个 issue 标题（search API，2026-09-14 之后），仅 1 个标题含中文字符（约 0.4%），issue 交流以英文为主；中文用户主要通过 zh-CN/zh-TW 本地化界面使用，在官方 issue 区的中文发声占比很低（该数据不覆盖 GitHub Discussions、Reddit r/sharex 等渠道，未确认）。

## 9. 亮点与风险

**亮点**
1. 工作流引擎设计优秀：23+6 个 Flags 动作位自由组合，把"捕获→编辑→上传→分享"做成可编程链条（`ShareX/Enums.cs`、`ShareX/WorkerTask.cs`），是同类工具的功能天花板。
2. 上传器反射注册表 + JSON 自定义上传器 DSL（`UploaderFactory.cs`、`CustomUploader/ShareXCustomUploaderSyntaxParser.cs`）：内置约 60 个具名目的地且可无限扩展，生态护城河。
3. 录屏管线现代化：Windows Graphics Capture + D3D11 + Media Foundation + HLSL 的原生 GPU 路径（`ShareX.ScreenRecordingLib/`），摆脱外部 FFmpeg 进程依赖。
4. 前瞻性架构迁移：WinForms→Avalonia、GDI+→SkiaSharp、热键/消息框全部走接口抽象（`IHotkeyHost`），编辑器模块（ImageEditor 79K 行）已可独立跨平台编译。
5. 29 语言本地化、4 种历史存储后端、多渠道分发（含 Steam/商店签名配置）体现成熟工程的运维面。

**风险/局限**
1. 无自动化测试，大规模 UI 框架迁移期回归风险高（第 7 章证据；近期 issue 中"Regression in Region Capture"等回归报告亦见于标题抽样）。
2. 单核心维护者瓶颈：PR 评审不保证（`CONTRIBUTING.md` 原文），外部大功能贡献难落地。
3. GPL-3.0 对闭源集成/商业再分发有限制；Windows 专属（Avalonia 迁移尚未发布非 Windows 版本，未确认时间表）。
4. 双轨代码并存（新旧录屏器、三个编辑器项目）抬高维护面与包体积，过渡期较长。

## 10. 总结评价

ShareX 是 Windows 截图分享工具链的"功能全集"式开源标杆：以热键驱动的链式工作流为骨架、以反射注册的上传器生态为血肉，历经 19 年持续演进，正处在 WinForms→Avalonia、GDI+→SkiaSharp、FFmpeg→GPU 原生录屏的三线现代化迁移中——本次分析快照恰好捕捉到它从"成熟的 WinForms 老将"走向"跨平台现代化新架构"的中间态，对研究大型 C# 桌面应用如何渐进式换骨架极具样本价值。它适合所有需要高频截图/录屏/文件分享的 Windows 用户（开发者、技术写作、客服、内容创作者）直接使用；适合想学习 Flags 枚举工作流引擎、反射服务注册表、Windows 媒体原生 API（WGC/Media Foundation/D3D11 HLSL）或 Avalonia 大型应用组织的开发者阅读源码，推荐入口为 `ShareX/WorkerTask.cs`、`ShareX.UploadersLib/UploaderFactory.cs` 与 `ShareX.ScreenRecordingLib/ScreenRecorder.cs`。
