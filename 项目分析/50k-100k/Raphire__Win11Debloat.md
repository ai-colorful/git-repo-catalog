# Raphire/Win11Debloat 项目分析

> 星数 60547（快照 2026-10-24）｜主语言 PowerShell｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：一个免安装的 PowerShell 精简脚本，用于批量移除 Windows 10/11 预装应用、关闭遥测与广告、定制界面行为，提供 GUI 与 CLI 双前端，并内置注册表备份/恢复与配置导入导出。

| 事实 | 内容 |
| --- | --- |
| 仓库 | Raphire/Win11Debloat（默认分支 master） |
| 星数 | 60,547（快照 2026-10-24）；查询时 61,543（gh api，2026-11-03） |
| 主语言 | PowerShell（另有少量 Batch、XAML、JSON 配置） |
| 许可证 | MIT（LICENSE） |
| 创建时间 | 2020-10-27（gh api repos/Raphire/Win11Debloat） |
| 最近活跃 | 最近提交 2026-10-30（`chore: release 2026.10.30`），最新 push 2026-11-01（gh api） |
| 最新版本 | Release 2026.10.30，发布于 2026-10-31，累计 30 个 release（gh api） |
| 规模 | 主脚本 647 行 + Scripts/ 83 个 .ps1 共约 1.6 万行；测试 50 个文件、632 个 Pester 用例 |
| 运行环境 | Windows PowerShell 5.1（明确拒绝 pwsh 7）、需管理员权限、依赖 WinGet ≥1.4 |

## 2. 项目是什么，解决什么问题

Windows 10/11 出厂预装了大量用户不需要的应用（Bing 系列、Clipchamp、Xbox 组件、厂商工具等），同时系统内置遥测、广告推荐、Copilot/AI 功能和大量不可直接关闭的界面行为。手动逐个卸载、逐项改设置既繁琐又容易遗漏。Win11Debloat 把这些操作收敛为一个脚本：用户在交互界面勾选，或用命令行参数/配置文件声明意图，脚本一次性完成应用卸载与注册表调整（README.md「Features」一节）。

功能覆盖面很宽，按仓库自身分类（Config/Features.json 的 Categories，共 12 类）包括：应用移除、隐私与建议内容、AI 功能（Copilot/Recall/Click to Do/各应用内 AI）、系统行为（右键菜单、指针加速、快速启动、BitLocker 自动加密、Modern Standby 联网）、Windows 更新策略、外观、开始菜单与搜索、任务栏、文件资源管理器、多任务、可选 Windows 功能（Sandbox/WSL）及其他（Game Bar、Brave 精简）。

项目的差异化在于把"系统修改"这类高风险操作做得工程化：每次应用变更前先做注册表备份（可跳过 `-SkipRegistryBackup`），可选创建系统还原点（`-CreateRestorePoint`），所有注册表调整都提供对应的 Undo .reg 文件，脚本运行全程 Start-Transcript 记录日志（`Win11Debloat.ps1`；`Scripts/Features/Invoke-Changes.ps1` 的 `Invoke-AllChanges` 四阶段编排）。README 顶部仍明确警告"use at your own risk"。

面向的人群不止个人用户：README 强调面向系统管理员与高级用户的能力——强大的 CLI 参数（约 90 个 switch 参数，`Win11Debloat.ps1` param 块）、Windows 审计模式（Sysprep 模式，把修改写入 Default 用户配置文件使新用户自动继承）、可指定目标用户（`-User`/`-AppRemovalTarget`）、配置文件导入导出便于批量部署（`Scripts/Helpers/Import-ConfigToParams.ps1`）。

## 3. 技术栈与依赖

- **语言**：Windows PowerShell 5.1 为主（`Win11Debloat.ps1` 开头显式检查 PSEdition -eq 'Core' 时退出，因 Appx 模块在 pwsh 7 不可用，引用 issue #675）；`Run.bat` 为启动器；XAML 用于 GUI。
- **GUI**：纯 WPF——`Add-Type -AssemblyName PresentationFramework,...` 后用 `XamlReader.Load` 解析 `Schemas/MainWindow.xaml`（82KB）等 10 个 XAML 文件，深色模式资源切换由 `Scripts/GUI/Set-WindowThemeResources.ps1` 处理，文本本地化由 `ConvertTo-LocalizedXaml` 在加载前注入（`Scripts/GUI/Show-MainWindow.ps1`）。
- **外部工具依赖**：
  - `winget` ≥1.4：应用卸载通道之一，脚本检测缺失时警告（`Win11Debloat.ps1`）；
  - `reg import`：注册表导入首选通道，目标用户配置单元已加载时回退到 PowerShell 注册表写入器（`Scripts/Features/Import-RegistryFile.ps1`）；
  - `Get-AppxPackage`/`Remove-AppxPackage`（Appx 模块）、`Get-CimInstance`、计划任务 cmdlet 等。
- **运行时无第三方模块依赖**；开发/CI 依赖 Pester 5（固定 5.9.0，`Scripts/Run-Tests.ps1`）与 PSScriptAnalyzer（`.github/workflows/tests.yml`）。
- **数据文件**：`Config/Apps.json`（162 个应用条目，含 `SelectedByDefault`/`Recommendation`/`RemovalMethod` 字段）、`Config/Features.json`（104 个功能定义 + 12 分类 + GUI 分组 UiGroups）、13 种语言的界面翻译（Config/Languages/，含 zh-CN、zh-TW）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `Win11Debloat.ps1` | 唯一入口：约 90 个参数声明、环境守卫、点源加载全部函数、按参数选择 GUI/CLI 模式并调用 `Invoke-AllChanges` |
| `Scripts/AppRemoval/` | 应用卸载：`Remove-SelectedApps.ps1`（WinGet/Appx 双通道）、`Invoke-ForceRemoveEdge.ps1`、WinGet 已装清单查询等 4 个文件 |
| `Scripts/Features/` | 功能执行核心：`Invoke-Changes.ps1`（编排）、注册表备份/恢复 4 个文件、`Import-RegistryFile.ps1`、`Replace-StartMenu.ps1`、遥测计划任务、可选功能等 15 个文件 |
| `Scripts/GUI/` | WPF 前端 24 个文件：主窗口、应用选择、备份恢复对话框、导入导出配置窗口、本地化、窗口 chrome 等 |
| `Scripts/CLI/` | 字符交互前端 8 个文件（菜单、默认模式、应用移除、待变更摘要） |
| `Scripts/FileIO/` | JSON/语言/设置读写 9 个文件（配置导入、LastUsedSettings、语言加载） |
| `Scripts/Helpers/` | 21 个通用工具：参数提升、用户配置单元定位（`Resolve-UserProfilePath.ps1` 30KB）、.reg 文件解析（`Get-RegFileOperations.ps1`）、配置一致性校验等 |
| `Scripts/Threading/` | GUI 非阻塞调用与 DoEvents 2 个文件 |
| `Scripts/Get.ps1` | 远程一键引导脚本（README 的 `irm https://debloat.raphi.re/` 即其 CDN）：下载 zip 到临时目录、UAC 提升运行、结束后清理（保留 Config/Logs/Backups） |
| `Scripts/Build-Release.ps1`、`Run-Tests.ps1` | 发布打包与本地测试运行器 |
| `Config/` | 数据驱动核心：Apps.json、Features.json、DefaultSettings.json（默认模式勾选项）、Languages/ 13 语言包 |
| `Regfiles/` | 90 个应用侧 .reg + `Undo/` 64 个撤销 .reg + `Sysprep/` 92 个审计模式 .reg（UTF-16 编码） |
| `Schemas/` | 10 个 XAML 窗口定义与共享样式（SharedStyles.xaml 40KB） |
| `Tests/` | 50 个 Pester 测试文件 + TestData 夹具 |
| `.github/workflows/` | tests.yml（PR/push CI）、release.yml（发布流水线） |

组织方式评述：这是把"单体脚本"演进为"函数点源 + 数据驱动"的典型结构——入口仍是一个大 .ps1（社区脚本的使用习惯使然），但 83 个函数文件按前端（GUI/CLI）、领域（Features/AppRemoval/FileIO/Helpers/Threading）分层，行为数据全部外置到 JSON 与 .reg，翻译独立成语言包。发布时 `Build-Release.ps1` 剔除 Tests/.github/构建脚本后打包。

## 5. 核心架构与关键实现

### 5.1 数据驱动的功能清单（Config/Features.json）

每个功能声明了 id、分类、GUI 文案、应用与撤销各自的 .reg 文件、以及 Windows 构建版本范围。主脚本加载后以 `$script:Features[FeatureId]` 哈希表供全局查询（`Win11Debloat.ps1` 加载段）：

```json
"FeatureId": "DisableTelemetry",
"Label": "Disable telemetry, tracking & targeted ads",
"Category": "Privacy & Suggested Content",
"RegistryKey": "Disable_Telemetry.reg",
"ApplyText": "Disabling telemetry and diagnostic data collection",
"RegistryUndoKey": "Enable_Telemetry.reg",
"MinVersion": null, "MaxVersion": null
```

新增一个注册表类功能只需加 JSON 条目 + 两个 .reg 文件，GUI/CLI/备份/撤销链路自动获得该功能。`Config/Apps.json` 对 162 个预装应用标注 `Recommendation`（safe 等）与 `RemovalMethod`（Appx/WinGet），驱动"默认模式"勾选与卸载通道选择。

### 5.2 执行编排：Invoke-AllChanges 四阶段（Scripts/Features/Invoke-Changes.ps1）

主流程为：收集 apply/undo 两列 featureId → 按 `Test-FeatureSupportedOnBuild` 剔除当前 Windows 构建不支持的功能（仅当需要时才做注册表备份）→ 依序执行：

1. **注册表备份**：`New-RegistrySettingsBackup` 只备份将被触碰的键；
2. **系统还原点**（可选，`Invoke-SystemRestorePoint.ps1`）；
3. **应用阶段** `Invoke-ApplyFeatures`：逐个调 `Invoke-FeatureApply`——注册表类直接 `Import-RegistryFile` 后执行副作用（如 DisableTelemetry 追加禁用计划任务、DisableCopilot 追加卸载 Copilot 包），自定义类（RemoveApps/ForceRemoveEdge/ClearStart/WSL 等）走专用逻辑；
4. **撤销阶段** `Invoke-UndoFeatures`：导入 `Undo/` 下的对应 .reg。

每个 feature 之间检查 `$script:CancelRequested`（GUI 取消按钮），GUI 模式通过 `$script:ApplyProgressCallback` 回调进度，失败计数区分"功能失败"与"应用移除失败"避免双重上报。

### 5.3 注册表备份与恢复体系（Scripts/Features/Backup-RegistrySnapshotCapture.ps1 等）

备份不是简单整库导出：`Get-RegistryBackupCapturePlans` 先用 `Get-RegFileOperations`（`Scripts/Helpers/Get-RegFileOperations.ps1`）解析本次将导入的 .reg 文件得到精确的键路径操作集合，再把待应用功能与撤销功能的目标键合并成捕获计划（Path/IncludeSubKeys/CaptureAllValues/AllowDeleteKey/ValueNames），据此从真实注册表导出"变更前状态"存到 `%LOCALAPPDATA%\Win11Debloat\Backups`；恢复时由 `Restore-RegistryBackup.ps1`（11KB）与校验逻辑 `Registry-BackupValidation.ps1`（33KB）配合 GUI 对话框（`Scripts/GUI/Show-RestoreBackupDialog.ps1`）选择性回滚。删除空键的清理逻辑也有独立模块（`Registry-EmptyKeyCleanup` 相关测试）。

### 5.4 执行安全设计（多处守卫，均在 Win11Debloat.ps1 与 Invoke-Changes.ps1）

- **权限提升**：非管理员时用 `Format-ElevatedArg` 对参数做 Win32 级引号转义后 `Start-Process -Verb RunAs` 重启自身，未确认则退出；
- **运行时守卫**：拒绝 PowerShell 7（Appx 模块不可用）、拒绝 `LanguageMode -ne FullLanguage`（受限环境）、SYSTEM 账户运行且未指定 `-User`/`-Sysprep` 时直接抛错（`Invoke-AllChanges` 开头守卫）；
- **预检**：检查必需文件齐全、检测域加入环境并警告"组策略可能覆盖修改"、检测 Modern Standby 支持与否决定对应开关可用性；
- **WhatIf 全链路**：`SupportsShouldProcess` + `Remove-SelectedApps`/`Import-RegistryFile` 内的 WhatIf 分支可在不落盘的情况下演练（`Import-RegistryFile.ps1` 的 WhatIf 走 `Invoke-RegistryOperationsFromRegFile` 解析而不导入）；
- **无操作保护**：当传入参数全为控制类参数（无任何功能开关）时直接退出不做修改（`Win11Debloat.ps1` 尾部 guard）；
- **目标用户解析**：`Resolve-UserProfilePath.ps1`/`User-HiveHelpers.ps1` 处理离线配置单元（Sysprep 写 HKEY_USERS\Default），配置单元已加载时自动从 reg.exe 切换到 PowerShell 注册表写入器并做 SID 重映射。

### 5.5 双前端与 GUI 失败回退

默认启动 GUI（`Show-MainWindow.ps1`），XAML 加载或运行异常时 catch 后提示"继续进入 CLI 模式"（`Win11Debloat.ps1` 的 GUI try/catch）。部署类参数（`-User`/`-Sysprep`/`-AppRemovalTarget`）出现时强制走 CLI。GUI 的应用选择、Tweaks 构建器（`MainWindow-TweaksBuilder.ps1` 30KB）、导入导出、备份恢复窗口拆分为独立文件，窗口 chrome（无边框、深浅主题）单独处理。

## 6. 构建与运行

运行（来自 README.md「Usage」）：

```powershell
# 快捷方式：远程下载并启动（支持参数）
& ([scriptblock]::Create((irm "https://debloat.raphi.re/")))

# 本地：管理员 PowerShell
Set-ExecutionPolicy Bypass -Scope Process -Force
.\Win11Debloat.ps1                      # GUI 模式
.\Win11Debloat.ps1 -RunDefaultsLite     # 默认精简模式
.\Win11Debloat.ps1 -Config my.json -Silent
```

测试与构建（来自 `Scripts/Run-Tests.ps1`、`Scripts/Build-Release.ps1`、`.github/workflows/*.yml`）：

```powershell
./Scripts/Run-Tests.ps1 -Bootstrap      # Pester 5.9.0 全量测试（CI 在 windows-latest 上运行）
./Scripts/Build-Release.ps1 -Version 2026.10.30 -OutputDirectory release-assets
# 产出 Win11Debloat.zip + SHA256SUMS + Get.ps1（CI 中上传至 draft release）
```

发布流程（`.github/workflows/release.yml`）：手工触发并传入 `YYYY.MM.DD[-rN]` 标签 → 正则替换主脚本内版本号 → 全量测试 → 提交并打 tag → 下载 zipball 调用 Build-Release → 创建 draft release 上传三个资产，人工确认后发布。无传统的"构建编译"步骤，发布即打包源码。

## 7. 测试与工程质量

- **测试规模**：Tests/ 下 50 个 `*.Tests.ps1`、632 个 `It` 用例，Pester 5（锁定 5.9.0，`Run-Tests.ps1` 自带 `-Bootstrap` 安装）；覆盖面包括核心执行器（Invoke-Changes.Tests.ps1 30KB）、应用移除（Remove-SelectedApps.Tests.ps1 37KB）、注册表备份创建/删除/校验/快照应用、配置导入、语言加载、GUI 部署逻辑与 CLI 菜单等，附 TestData JSON 夹具。
- **测试安全规范**：`Tests/TestSuiteSafety.Tests.ps1` 是一项元测试——列出 47 个高危命令（Remove-AppxPackage、dism、reg、schtasks、Set-MpPreference 等），断言任何测试文件不得直接执行它们，防止测试本身破坏开发机。这是对"测试系统修改类脚本"这一难题的成熟应对。
- **契约测试**：`XamlContracts.Tests.ps1` 校验 XAML 中 FindName 引用的控件与定义一致；`FeatureParameterContracts.Tests.ps1` 校验参数、Features.json 与 UI 分组的对应关系；`LanguageCatalogs.Tests.ps1` 校验语言包完整性——用测试锁住"数据文件与代码同步"这一最容易腐烂的环节。
- **CI**（`.github/workflows/tests.yml`）：PR 与 master push 触发两个 job——Pester（Windows PowerShell 5.1）与 PSScriptAnalyzer（仅 Error 级别，EnableExit）。checkout 步骤均用提交哈希固定 Action 版本且 `persist-credentials: false`。
- **代码质量观察**：函数普遍有基于注释的 help 块与明确的返回值约定；对边界情况处理细致（如 `ConvertTo-ConfigBoolean` 对 1/1.0/1e0/"false" 的类型区分，`Scripts/Helpers/Import-ConfigToParams.ps1`；UAC 参数转义；SCCM 环境无 LOCALAPPDATA 的回退）。代码注释中直接引用 issue 编号（#675、#720）说明修 bug 有据可查。

## 8. 社区与维护状态

- **活跃度**：创建于 2020-10-27，至今持续高频维护——最新提交 2026-10-30、最新 push 2026-11-01（gh api，2026-11-03 查询）；open issues 39。
- **发布节奏**：采用日期版本号（YYYY.MM.DD[-rN]），累计 30 个 release，最新 2026.10.30 于 2026-10-31 发布，基本保持月度/半月度节奏（gh api releases）。
- **贡献者结构**：41 位贡献者（gh api contributors 分页计数）；CONTRIBUTING.md（24KB）提供完整的本地运行、测试、PR 指南与问题模板（bug_report/feature_request 为 YAML 表单）；另有 SECURITY.md、dependabot.yml、FUNDING.yml（ko-fi 赞助）。主作者 Raphire 主导，发布提交由 github-actions[bot] 完成。
- **下游分发**：README 提供三种启动方式，其中 `debloat.raphi.re` 域名服务 `Scripts/Get.ps1` 一键引导；被众多 Windows 精简教程与系统管理员部署方案引用（第三方引用情况未在源码内确认）。

## 9. 亮点与风险

**亮点**

1. **数据驱动的可扩展架构**：功能、应用、翻译、默认值全部外置于 JSON/.reg，加功能基本零代码改动，且 XAML/参数/语言包一致性有契约测试兜底（Config/、Tests/FeatureParameterContracts.Tests.ps1、Tests/XamlContracts.Tests.ps1）。
2. **完整的可回滚链路**：按变更精确捕获的注册表快照备份 + Undo .reg + 可选系统还原点 + WhatIf 演练，在"改系统"类工具中安全工程化程度突出（Scripts/Features/Backup-*.ps1、Restore-*.ps1）。
3. **测试文化罕见地强**：632 用例、测试套件自身的安全元测试、PSScriptAnalyzer 进 CI，对一个 PowerShell 脚本项目而言是第一梯队水平（Tests/）。
4. **面向部署的进阶能力**：Sysprep 审计模式、指定目标用户/离线配置单元、配置文件导入导出与静默模式，使其能进入真实的企业镜像制作流程（Win11Debloat.ps1、Scripts/Helpers/）。
5. **发布工程严谨**：日期版本号正则注入、发布前强制全量测试、SHA256SUMS 校验和、draft release 人工确认（.github/workflows/release.yml）。

**风险与局限**

1. **系统修改的固有风险**：脚本批量改注册表并卸载应用（含 `Invoke-ForceRemoveEdge.ps1` 强制移除 Edge 这类激进操作），Windows 更新可能恢复或破坏部分调整；README 明示"use at your own risk"，域环境还会被组策略覆盖。
2. **仅支持 Windows PowerShell 5.1**：pwsh 7 下直接退出（依赖 Appx 模块），对习惯 pwsh 的用户是硬约束（Win11Debloat.ps1 版本守卫，源自 issue #675）。
3. **一键远程执行模式的安全面**：`irm https://debloat.raphi.re/` 直接执行远端代码，依赖该域名与仓库供应链的完整性；仓库缓解措施是 SHA256SUMS 与发布测试，但用户侧无法本地审计每次变更。
4. **tarball/压缩包在杀软下可能误报**：本仓库含大量系统修改类 .ps1/.reg，此类内容在 Windows Defender 实时防护下存在被隔离的可能（本次分析采用 tar 流式读取规避，未见仓库内相关说明，此点为环境经验而非仓库声明）。

## 10. 总结评价

Win11Debloat 是"系统修改类工具做成正经软件工程"的代表样本：对外是一个双击即用的精简脚本，对内是配置驱动、四阶段编排、精确备份回滚、632 个测试用例守护的数据管道。它的星数（6 万+）来自切中了 Windows 用户去广告去预装的普遍痛点，而能长期留住用户则靠月度发布节奏与罕见于脚本项目的工程质量。适合想一键净化个人 Windows 的普通用户（走 GUI/默认模式）、需要批量部署镜像的系统管理员（CLI/Sysprep/配置文件），也非常值得任何维护"大规模系统操作脚本"的开发者读其源码——尤其是 Config/Features.json 的数据驱动设计、Backup-RegistrySnapshotCapture 的按需备份思路与 Tests/TestSuiteSafety.Tests.ps1 的测试自保护约定。
