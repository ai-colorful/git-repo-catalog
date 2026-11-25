# files-community/Files 项目分析

> 星数 38500（快照 2026-10-24）｜主语言 C#｜档位 20k-50k｜分析日期 2026-11-05
>
> 注：分析当日 `gh api repos/files-community/Files` 实测星数已达 46499，主分支 `main` 最近 push 为 2026-11-04，项目仍在高强度活跃开发中。

## 1. 项目速览

一句话定位：Windows 平台开源现代文件管理器（File Explorer 替代品），C#/WinUI 3 + Windows App SDK 构建，已上架 Microsoft Store。

| 基本事实 | 内容 |
| --- | --- |
| 名称 | files-community/Files |
| 星数 | 38500（快照 2026-10-24，实测 2026-11-05 为 46499） |
| 主语言 | C#（含少量 C++/ATL 原生工程） |
| 许可证 | MIT（`.github/NOTICE.md`、LICENSE-MIT；另有 LICENSE-MPL 针对部分组件，顶层两份 LICENSE 并存） |
| 当前版本 | v4.3（GitHub Release 2026-10-11），快照包清单版本 4.3.2.0（`src/Files.App/Package.appxmanifest`） |
| 最近活跃 | main 分支 push 于 2026-11-04（gh api），v4.2.x 在 2026-08 连发 4 个补丁版本 |
| 平台 | 仅 Windows 10 19041+（Windows.Desktop），x86/x64/arm64 三架构 |
| 代码规模 | 1197 个 .cs + 146 个 .xaml（src+tests 统计） |

## 2. 项目是什么，解决什么问题

Files 是一个对标 Windows 资源管理器的现代文件管理器，由 files-community 开源社区维护。README（`.github/README.md`）明确其使命是"为 Windows 构建最好的文件管理器"，并以完全开源的方式开发。它提供的核心价值包括：多标签页与双窗格多任务、文件标签（Tags）系统、与 Windows Shell 的深度集成（右键菜单、跳转列表、库、回收站）、直观的 Fluent 设计。

与资源管理器相比，Files 补齐了许多效率特性：Details/Grid/Column/Columns 四种视图布局（`src/Files.App/Views/Layouts/`）、内置预览窗格、批量重命名、归档压缩（集成 7-Zip）、Git 状态显示（集成 LibGit2Sharp，`src/Files.App/Services/Git/`）、FTP 网络位置（FluentFTP）、UDF 光盘镜像读取（DiscUtils.Udf）等。仓库还包含将系统"打开/保存"对话框替换为 Files 的原生 COM 组件（`src/Files.App.OpenDialog/`、`src/Files.App.SaveDialog/`），体现其"深入接管 Windows 文件体验"的野心。

项目当前运行在 WinUI 3 / Windows App SDK 2.6.5 上（`Directory.Packages.props` 中 `Microsoft.WindowsAppSDK 2.6.5`，`Files.App.csproj` TFM 为 `net10.0-windows10.0.26100`）。本仓库早期基于 UWP，后迁移至 Windows App SDK（本快照中无迁移历史记录，此为公开背景信息，仓库内证据是当前纯 WinAppSDK 技术栈与 `Files.Storage/Legacy/` 这类遗留适配层的存在）。

商业模式也值得一提：应用免费开源，靠 Microsoft Store 付费版与 GitHub Sponsors 支持开发（README 安装章节、`.github/FUNDING.yml`）。

## 3. 技术栈与依赖

- **语言/运行时**：C#（`LangVersion=preview`，nullable 警告视为错误），.NET 10（`global.json` 指定 SDK 10.0.102；`Directory.Build.props` 设 `net10.0`）。
- **UI 框架**：WinUI 3，Windows App SDK 2.6.5 + Win2D；大量 CommunityToolkit.WinUI 控件库（SettingsControls/Sizers/ColorPicker 等，`Directory.Packages.props`）。
- **MVVM/DTO**：CommunityToolkit.Mvvm 8.4.2（源生成器实现 `[ObservableProperty]`/命令），Microsoft.Extensions.DependencyInjection/Logging 10.0.12。
- **互操作**：Microsoft.Windows.CsWinRT 2.3.1 + CsWin32 0.3.363 源生成 P/Invoke（禁用 `DllImport`/`ComImport`，见 `AGENTS.md` 评审规则），`src/Files.Storage/NativeMethods.txt` 声明 426 个 Win32 API。
- **AOT/裁剪**：主工程 `IsAotCompatible=true`、`IsTrimmable=true`，`CsWinRTAotWarningLevel=3`，Release 发布启用 ReadyToRun+Composite（`src/Files.App/Files.App.csproj`）。
- **功能库**：LibGit2Sharp（Git）、FluentFTP（FTP）、DiscUtils.Udf（光盘）、SevenZipSharp/SharpZipLib（压缩）、TagLibSharp（媒体元数据）、Microsoft.Data.Sqlite（本地数据库）、Sentry（崩溃上报）、OwlCore.Storage。
- **AI**：引入 `Microsoft.WindowsAppSDK.AI`/`Microsoft.Windows.AI.MachineLearning`（`Directory.Packages.props`，面向 Windows 机器学习能力的集成）。
- **原生 C++**：3 个 vcxproj（Launcher/OpenDialog/SaveDialog），ATL COM DLL 形式替换经典文件对话框。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `src/Files.App` | 主 WinUI 应用（892 个 .cs）：Views/ViewModels/Services/Actions/Helpers/Dialogs/UserControls |
| `src/Files.App.Controls` | 可复用自定义控件（102 个 .cs，如 TabBar、Toolbar 配套控件） |
| `src/Files.Storage` | 存储实现与 Win32/Shell 互操作层（CsWin32 生成、STA 线程辅助、Shell 项封装） |
| `src/Files.App.Server` | 进程外 WinRT 服务端（68 个 .cs）：驱动器/标签/回收站/设置/更新等 10 个服务 |
| `src/Files.App.BackgroundTasks` | 后台任务工程 |
| `src/Files.Core.SourceGenerator` | Roslyn 源生成器/分析器（16 个 .cs） |
| `src/Files.Shared` | 共享特性、扩展与公共代码 |
| `src/Files.App.Launcher|OpenDialog|SaveDialog` | C++/ATL 原生工程：启动器与 Open/Save 对话框 COM 替换组件 |
| `tests/Files.App.UITests` | 应用启动冒烟 UI 测试 |
| `tests/Files.InteractionTests` | Appium + Axe.Windows 交互/无障碍测试 |
| `.github/` | CI/CD（6 个 workflow）、README、CONTRIBUTING、issue/PR 模板、截图素材 |

组织方式：单解决方案（`Files.slnx`，新一代 XML 解决方案格式）多工程分层，UI 层（Files.App）与存储/互操作层（Files.Storage）、进程外服务（Files.App.Server）解耦清晰；依赖版本集中管理（`Directory.Packages.props` CPM）。仓库根有 `AGENTS.md`（2.3 万字节的 AI 协作开发规范），是近年少见的把工程约束写成"AI 代理评审规则"的项目。

## 5. 核心架构与关键实现

### 5.1 标签页与多窗格模型（Shell 体系）

每个标签页对应一个 Shell 视图实例：`MainPageViewModel` 维护 `ObservableCollection<TabBarItem> AppInstances` 与多任务控件接口 `ITabBar`（`src/Files.App/ViewModels/MainPageViewModel.cs:35-46`）；页面侧由 `ShellPanesPage` 承载窗格（双 Pane：`Views/Shells/BaseShellPage.cs`、`ModernShellPage`、`ColumnShellPage`）。`BaseShellPage`（986 行）实现 `IShellPage`，持有 `TabBarItemParameter` 标识各标签的导航参数并在 `RaiseContentChanged` 中广播内容切换（`src/Files.App/Views/Shells/BaseShellPage.cs:130-701`）。视图布局方面 `BaseLayoutPage`（1803 行）派生出 Details/Grid/Column/Columns 四种布局页并支持分组（`BaseGroupableLayoutPage`）。

### 5.2 文件系统操作抽象（双实现策略）

文件复制/移动/删除等操作统一走 `IFilesystemOperations` 接口（`src/Files.App/Utils/Storage/Operations/IFilesystemOperations.cs`），有两个实现：

- `FilesystemOperations`：基于 WinRT Storage API 的纯托管实现，可产生撤销历史（`IStorageHistory`）；
- `ShellFilesystemOperations`（957 行）：包装 Shell COM 操作，支持回收站语义、权限提示与系统进度条，内部持有前者做降级回退。构造函数从 `Ioc.Default` 取回收站服务并绑定所属标签页实例（`src/Files.App/Utils/Storage/Operations/ShellFilesystemOperations.cs:15-31`），进度通过 `IProgress<StatusCenterItemProgressModel>` 上报到状态中心 UI。

这种"托管 API 优先、Shell COM 兜底"的分层让普通操作走沙盒友好路径，而系统集成场景（回收站/提权）走 Shell 路径。

### 5.3 进程外 WinRT 服务（Files.App.Server）

`Files.App.Server` 是独立可执行文件，以 `OutOfProcessServer` 形式注册于包清单（`src/Files.App/Package.appxmanifest` 中 `uap5:IdentityType="activateAsActivator"`），通过 `Program.cs`/`ServerHost.cs`/`AppService.cs` 暴露 10 个服务：Drive、FileTags、Icon、JumpList、Location、PinnedFolders、RecycleBin、Settings、Shell、Update（`src/Files.App.Server/Services/`），DTO 位于 `Contracts/`。跨进程异步统一用自研 `ServerAsync.Run` 包装 `IAsyncOperation`，处理客户端断连时回调投递的 RPC 异常（`src/Files.App.Server/ServerAsync.cs`，规则详见 `AGENTS.md` "WinRT server and client interoperability" 一节，包括 `IIterable<T>`/`IIterator<T>` 代理注册、Native AOT 类型根注册 `WinRTExposedExternalTypes.cs` 等硬约束）。典型服务如 `FileTagsService` 仅几行：

```csharp
public sealed partial class FileTagsService
{
    public static IAsyncOperation<IReadOnlyList<TaggedFileInfo>> GetTaggedFilesAsync()
        => ServerAsync.Run(token => TagCatalog.Items.GetAsync().WaitAsync(token));
}
```

（`src/Files.App.Server/Services/FileTagsService.cs:7-11`）

把设置、图标缓存、回收站监听等移出主进程，既隔离了 Shell COM 崩溃风险，也为提权操作、多进程状态共享（多窗口设置实时同步）铺路。

### 5.4 设置持久化与 DI

设置按域拆分为 11 个服务接口（General/Appearance/Folders/Layout/Actions 等，`src/Files.App/Services/Settings/`），全部由 `UserSettingsService` 派生共享上下文（`GetSharingContext()`），在 `AppLifecycleHelper.ConfigureHost` 中集中注册 DI（`src/Files.App/Helpers/Application/AppLifecycleHelper.cs:287-317`）。持久化底层是 `ServerSettingsDatabase`（实现 `IJsonSettingsDatabase`，`src/Files.App/Utils/Serialization/Implementation/ServerSettingsDatabase.cs`），即设置读写实际发生在进程外服务器中，序列化采用源生成 `JsonSerializerContext`（`UserSettingsService.cs:45` 的 `UserSettingsJsonSerializationContext`）以满足 Native AOT 禁反射的要求。

### 5.5 Win32 互操作规范（CsWin32 中心化）

所有原生调用集中在 `src/Files.Storage`，通过 `NativeMethods.txt`（426 行 API 声明）+ `NativeMethods.json` 由 CsWin32 源生成；`AGENTS.md` 明令评审拒绝新增 `DllImport`/`ComImport`。Shell 对象操作放在专用 STA 工作线程（`src/Files.Storage/Windows/Managers/STATask.cs`、`Windows/Shell/ThreadWithMessageQueue.cs`），避免污染 UI 线程套间——这是 WinUI 3 桌面应用处理 Shell 互操作的教科书式做法。

## 6. 构建与运行

来自 `AGENTS.md` 与 `.github/workflows/`：

```powershell
# 还原+构建整个解决方案（需 VS 2026 / .NET 10 SDK + Windows App SDK，x64|x86|arm64）
msbuild -restore Files.slnx -p:Configuration=Debug -p:Platform=x64 -v:quiet -clp:ErrorsOnly

# 只构建主应用
msbuild -restore src/Files.App/Files.App.csproj -p:Configuration=Debug -p:Platform=x64
```

- 前置要求见官方文档（README 指向 files.community/docs/contributing/building-from-source）；`.github/CONTRIBUTING.md` 要求先开 issue 讨论、PR 标题带 `Fix:`/`Feature:`/`Code Quality:` 前缀（发布说明据此自动组织）。
- 交互测试（CI 内）：MSTest + Appium.WebDriver + Axe.Windows，工程 `tests/Files.InteractionTests/Files.InteractionTests.csproj`，使用 Microsoft.Testing.Platform 运行器（`global.json`）。
- 本规范约定本任务不执行构建，以上为文档/工程文件记载的命令。

## 7. 测试与工程质量

- **测试规模**：两套自动化测试工程（`tests/Files.App.UITests` 12 个 .cs，应用启动冒烟；`tests/Files.InteractionTests` 10 个 .cs，Appium UI 交互 + Axe.Windows 无障碍扫描）。无传统单元测试目录——`AGENTS.md` 直言"当前没有适合 AI 代理的测试集，保证构建通过即可"，测试重心在 UI 层。快照中 `AGENTS.md` 提到的 `Files.App.UnitTests` 不存在于 slnx 与 tarball（疑似近期移除/合并，未确认）。
- **CI**（`.github/workflows/ci.yml`）：main 推送与 PR 触发，windows-2025-vs2026 运行器；流程含 XamlStyler 格式检查（独立 `format-xaml.yml` 也会跑）→ msbuild 还原构建 → 交互测试 → 上传构件。路径过滤忽略 docs/assets。
- **CD**（4 个工作流）：`cd-store-preview/stable.yml`（构建 msixupload 并发布商店分支，workflow_dispatch 手动触发）、`cd-sideload-preview/stable.yml`（自签名旁加载包）。发布物经 `Configure-AppxManifest.ps1` 注入 Store 发布者、Bing Maps Key、Sentry DSN 等 secret。
- **质量观察**：`WarningsAsErrors=nullable`、Roslyn strict 模式、AOT 警告级别 3 全开、XAML 样式统一由 Settings.XamlStyler 约束；`AGENTS.md` 的 Code Review Rules（反射/AOT/COM 套间/WinRT ABI 数据契约）是相当深入的工程治理文档，说明团队在 Native AOT + 进程外 WinRT 这条少有人走的路上踩过并沉淀了大量坑。

## 8. 社区与维护状态

- **活跃度**：main 分支最近 push 2026-11-04（gh api）；快照内 1197 个 .cs 的 tar mtime 均为 HEAD 提交时间，无法据此推断节奏，但 v4.3（2026-10-11）后 v4.2.x（2026-08 月内 4 个版本）显示月度级发布节奏；Release 由 PR 类型自动组织说明（`AGENTS.md`）。
- **社区渠道**：Discord（README 徽章）、Crowdin 本地化（`crowdin.yml`；主工程资源限定符列出 50+ 语言，`Files.App.csproj` AppxDefaultResourceQualifiers）、GitHub 任务看板（README 指向 org projects）。
- **贡献流程**：`.github/CONTRIBUTING.md` 要求 issue 先行、PR 小步提交；dependabot 已启用（`.github/dependabot.yml`）。贡献者规模未确认（未调 contributors API 细分）。
- **商业化**：Microsoft Store 付费 + 经典安装器免费（README），配套 FUNDING.yml；Sentry 生产崩溃上报（csproj Sentry 配置段）。

## 9. 亮点与风险

**亮点**

1. **进程外 WinRT 服务架构**（`src/Files.App.Server` + `Package.appxmanifest` OutOfProcessServer）：把 Shell/设置/回收站等高风险组件与 UI 进程隔离，`ServerAsync.cs` 对跨进程异步完成与客户端断连的处理细节罕见地完备。
2. **Native AOT 全链路认真推进**：`IsAotCompatible` + CsWinRT 警告级别 3 + 源生成 JSON 序列化 + CsWin32 禁手写 P/Invoke，`AGENTS.md` 把 AOT/ABI 约束写成强制评审规则，工程治理水平高。
3. **清晰的分层与双实现文件操作**：`IFilesystemOperations` 托管/Shell 两实现、`Files.Storage` 集中互操作、布局视图统一基类，主工程 892 个 .cs 仍有清晰边界可循。
4. **完整 CI/CD 与发布流水线**：从 XAML 格式检查到商店 msixupload 全自动化，发布说明由 PR 前缀驱动，社区协作路径（issue 先行 + 任务看板 + Crowdin）成熟。
5. **AI 协作开发规范样本**：根级 `AGENTS.md`（含 WinRT ABI 数据契约表、示例 manifest 片段）是开源项目规范 AI 辅助开发的优秀参考。

**风险/局限**

1. **仅限 Windows + 极新工具链**：.NET 10 preview 语言版本 + VS 2026 + Windows App SDK 2.6，构建门槛高，无 CI 之外的轻量验证手段。
2. **测试金字塔倒置**：几乎全部自动化测试在 Appium UI 层，核心文件操作/设置逻辑缺少单元级回归保护，重构风险靠构建与评审兜底。
3. **Native AOT + 进程外 WinRT 属非主流路径**：CsWinRT 代理注册、IID 推导等约束强依赖工具链版本（`AGENTS.md` 自己也提醒"升级 C#/WinRT 后需复查"），升级维护成本不低。
4. **多进程设置的复杂度**：设置经 `ServerSettingsDatabase` 落在服务进程，调试链路跨两个进程，社区贡献者排查问题的心智负担较高。

## 10. 总结评价

Files 是"用开源社区再造一个 Windows 文件管理器"的标杆项目：38k+ 星、MIT 许可、上架商店、月度发布、50+ 语言本地化，产品与工程双成熟。技术上它代表了 WinUI 3 桌面应用的前沿实践——进程外 WinRT 服务、Native AOT、CsWin32 源生成互操作、Roslyn 源生成 MVVM——并用一份罕见的 `AGENTS.md` 把这些约束制度化为评审规则。想学习现代 Windows 桌面架构（尤其 Shell 互操作与 WinRT ABI 细节）的开发者，这个仓库几乎是必读教材；想在 Windows 上替代资源管理器的普通用户则可直接从商店安装。短板是构建工具链新、UI 层测试为主、核心逻辑单测薄弱，贡献者需要一定 C#/WinRT 功底才能安全上手。
