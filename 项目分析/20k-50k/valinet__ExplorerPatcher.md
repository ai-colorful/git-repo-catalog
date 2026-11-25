# valinet/ExplorerPatcher 项目分析

> 星数 34,378（gh api 核实 2026-11-21；派发快照 26,000 @2026-10-24）｜主语言 C｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

ExplorerPatcher 是一个 Windows Shell 定制工具：通过向 explorer.exe 等系统进程注入 DLL 并 hook 系统接口，在 Windows 11 上恢复 Windows 10 风格的任务栏、开始菜单、Alt+Tab 切换器与文件资源管理器 UI。它是同类工具中少数完全开源的项目。

| 项 | 值 | 出处 |
|---|---|---|
| 仓库 | valinet/ExplorerPatcher，默认分支 master | gh api |
| 星数 / Fork / Watch | 34,378 / 1,387 / 288 | gh api 2026-11-21 |
| 创建时间 | 2021-08-09 | gh api |
| 最近 push | 2026-11-20（前一天仍在发版） | gh api |
| 许可证 | GPL-2.0 | LICENSE |
| 主语言 | C（C++ 用于 COM/WinRT 部分），纯 Windows 原生代码 | 源码统计 |
| 构建 | MSVC/MSBuild，12 个 vcxproj 项目 | ExplorerPatcher.sln |
| 版本号模式 | 跟随 Windows 24H2 build 号（如 26100.9711.71.3） | version.h |
| 贡献者 | 19 人（per_page=100 全量），以作者 valinet 为绝对主体 | gh api contributors |

## 2. 项目是什么，解决什么问题

Windows 11 用全新的 WinUI/XAML 任务栏（Taskbar.dll，经 TrayUIComponent 加载）替换了 Win10 的任务栏实现，同时移除了任务栏图标拖动排列、紧凑布局、经典开始菜单磁贴布局等能力。ExplorerPatcher 的做法不是重画一个仿制任务栏，而是激活系统二进制内仍保留的旧代码路径：Windows 11 的 explorer.exe 内部仍然带着 Win10 任务栏的实现函数 `TrayUI_CreateInstance()`（ExplorerPatcher/Taskbar10.cpp 顶部注释详细记录了这一机制），开始菜单同样保留了旧的 StartDocked/StartUI 组件。EP 通过 hook 把系统引导到这些旧实现上，从而得到"原汁原味"的 Win10 UI，而不是像素级模仿。

对 26002 之后的 Windows build（Win10 原生任务栏代码路径被微软从 explorer.exe 中移除），项目提供了第二条路径：独立的 `ep_taskbar.<版本代号>.dll`——一个以 Win10 任务栏为蓝本重建的完整任务栏实现（ExplorerPatcher/utility.h:1711 `PickTaskbarDll()`）。这些 DLL 不在本仓库源码内，由作者在独立的 `ExplorerPatcher/ep_taskbar_releases` 仓库以预编译产物发布，CI 构建时下载打包进安装器（.github/workflows/build.yml:97-122）。

功能面覆盖：任务栏样式与居中/对齐（TaskbarCenter.cpp）、Win10 开始菜单（ep_startmenu/）、Win10 Alt+Tab 切换器（sws 子模块 Simple Window Switcher）、文件资源管理器搜索栏/地址栏/右键菜单（HideExplorerSearchBar.c、TwinUIPatches.cpp）、系统托盘/飞行窗口（ImmersiveFlyouts.c）、输入切换器（InputSwitch.cpp）、任务栏天气小组件（ep_weather_host/）、Win10 启动音效（StartupSound.cpp）等。配置集中在任务栏右键"Properties"打开的面板（ep_gui）。

## 3. 技术栈与依赖

- **语言**：C 为主（dllmain.c、symbols.c、ep_setup.c 等），C++/WinRT 用于 COM 组件与 XAML 相关 patch（TwinUIPatches.cpp、Taskbar10.cpp、ShellExperienceHostPatches.cpp）。
- **Hook 引擎**：KNSoft.SlimDetours 1.1.4-beta（NuGet，inline hook），经 ExplorerPatcher/hooking.h 包装为 funchook 兼容接口；IAT（导入表）级 patch 由自研子模块 libvalinet 的 `VnPatchIAT` 提供（dllmain.c:34 include valinet/hooking/iatpatch.h）。
- **WIL**（Microsoft.Windows.ImplementationLibrary，NuGet）：COM/WinRT 资源管理（TwinUIPatches.cpp:7-9）。
- **子模块 6 个**（.gitmodules）：libvalinet（IAT patch、PDB 解析、符号下载等自研基础库）、sws（Win10 Alt+Tab）、zlib（安装器解包）、ep_dwm、ep_starttiledata（Win10 磁贴数据）、ExplorerPatcher-L10N（多语言翻译）。
- **符号处理**：自研 PDB 解析（libvalinet 的 `VnGetSymbols`），从微软符号服务器获取 PDB 提取函数偏移（ExplorerPatcher/symbols.c:48-49）。
- 无脚本语言、无托管运行时，产物全部为原生 DLL/EXE；无 CMake，纯 MSVC 工程体系。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| ExplorerPatcher/ | 核心 DLL 源码（注入 explorer.exe 的主模块），12 项目中体量最大 |
| ep_setup/ | 安装器/卸载器（单 C 文件 1757 行，自带 minizip 与 AES 加密代码 rijndael-alg-fst.c） |
| ep_setup_patch/ | 把编译好的 DLL 嵌入 ep_setup.exe 的后处理工具 |
| ep_gui/ | 任务栏右键"Properties"配置面板（GUI.c 4301 行） |
| ep_startmenu/ | 注入 StartMenuExperienceHost 的 Win10 开始菜单引导 DLL |
| ep_weather_host/ + ep_weather_host_stub/ | 任务栏天气小组件的 out-of-proc COM server 及其 stub |
| ep_dwm/（子模块） | DWM 相关辅助（Win10 圆角规避等） |
| ep_extra/ + ep_extra_valinet.win7alttab/ | 第三方扩展 DLL 加载器与示例扩展（Win7 风格 Alt+Tab） |
| ep_starttiledata/（子模块） | Win10 开始菜单磁贴布局数据层 |
| ep_generate_release_name/description/ | 发布名与发布说明生成小工具（CI 用） |
| libs/（子模块） | libvalinet、sws、zlib |
| .github/workflows/build.yml | 完整 CI/CD（构建→打包→发版一条龙） |

组织评述：典型的"多 DLL 协作的单体解决方案"。一个 .sln 含 12 个 VC 项目，每个注入目标（explorer.exe、StartMenuExperienceHost.exe、ShellExperienceHost.exe）对应独立 DLL，另配安装器与工具链小项目；跨项目共享代码靠子模块（libvalinet）与顶层公共头（debug.h、version.h）。没有 CMake/包管理器现代化，是纯 MSVC 世界。

## 5. 核心架构与关键实现

### 5.1 加载入口：dxgi.dll 代理 + COM 注册双通道

主 DLL `ExplorerPatcher.amd64.dll` 在 CI 中被直接复制为 `dxgi.dll`（build.yml:129-133），安装到 Windows 目录。它导出系统 dxgi.dll 的全部 20 个函数并逐一转发到 `System32\dxgi.dll` 真实现（ExplorerPatcher/dxgi_imp.cpp 全文，即经典的"代理 DLL/导入表劫持"机制——explorer.exe 按搜索顺序加载到该副本，进程内即获得执行权）。安装器写入该文件时在头部嵌入对官方 ep_setup.exe 的 32 字节哈希（ep_setup/ep_setup.c:572-588 `GetHardcodedHash`），用于区分官方部署与第三方放置。入口导出 `DXGIDeclareAdapterRemovalSupport`、`CreateDXGIFactory1` 与 COM `DllGetClassObject`，三者都汇入同一 `EntryPoint()`（dllmain.c:13921），按宿主进程分派：explorer.exe / StartMenuExperienceHost.exe / ShellExperienceHost.exe 各走不同注入通道（DLL_INJECTION_METHOD_* 0-3，dllmain.c:13917-13920），并对 mpc-hc、vlc、powertoys 等易冲突应用维护黑名单直接退出（dllmain.c:13966-13975）。DllMain 本身极薄，仅在 PROCESS_ATTACH 记录模块句柄（dllmain.c:14103-14126），实际工作全部延后到入口导出被调用，避免 loader 锁问题。

### 5.2 符号解析：PDB 下载 + 注册表偏移缓存

系统函数没有导出符号，EP 为 4 个系统二进制维护符号清单：explorer.exe（6 个符号，如 `CTaskBand_CreateInstance`）、twinui.pcshell.dll（7 个）、StartDocked/StartUI（symbols.h:16-49）。加载时对目标文件计算哈希，调用 libvalinet 的 `VnDownloadSymbols` 获取微软符号服务器对应 PDB，再由自研 PDB 解析器 `VnGetSymbols` 提取函数 RVA，按 `HKCU\Software\ExplorerPatcher\<模块>` 注册表缓存并带版本号校验（symbols.c:51-115 `CheckVersion/SaveVersion/ProcessExplorerSymbols`）。符号下载可被用户关闭（bEnableSymbolDownload，dllmain.c:108）。CHANGELOG 中高频出现的"Pattern scan keep-ups"（如 26100.9711.71 的十余条）即微软每次更新 OS 后按字节模式重新定位这些符号的跟进修复。

### 5.3 任务栏/开始菜单/资源管理器三条 UI 恢复路径

- **Win10 任务栏（bOldTaskbar 三态）**：配置值经 `AdjustTaskbarStyleValue`（utility.h:1768）自适应——0=Win11 原生；1=激活 explorer.exe 内遗留的 Win10 任务栏（仅 build<26002 可用，`IsStockWindows10TaskbarAvailable` utility.h:1702）；2+=加载重建版 `ep_taskbar.<代号>.dll`（dllmain.c:10807-10850 按版本选 DLL 并校验其 GetVersion 导出）。实现 1 的关键在 Taskbar10.cpp 顶部注释所述：hook `CoCreateInstance` 让 `CTray::InitializeTrayUIComponent` 拿到自实现的 `ITrayUIComponent`，进而调用 explorer.exe 中的 `TrayUI_CreateInstance()`（Taskbar10.cpp:10-27 注释与 EPTrayUIComponent 类）。
- **Win10 开始菜单**：ep_startmenu/ 通过 COM 类激活劫持把 StartMenuExperienceHost 的 XAML 启动引到旧 StartDocked/StartUI 组件（ep_sm_main.c 中直接引用其 App/XamlMetaDataProvider 接口 IID，ep_sm_main.c:15-19），系统缺失旧组件时回落到 `StartUI_.dll` 自带副本（ep_sm_main.c:36-54）。
- **文件资源管理器**：TwinUIPatches.cpp（4264 行）与 HideExplorerSearchBar.c 处理 Win11 右键菜单禁用、地址栏收缩、经典驱动器分组等；Win11 上下文菜单关闭利用 Windows 内部 feature staging API（dllmain.c:10244 获取 ntdll 的 RtlQueryFeatureConfiguration，inc/rtlfeatureconfig.h）。
- **版本适配层**：`PickTaskbarDll()`（utility.h:1711-1752）按 build 号把 Windows 10 1703 → Windows 11 24H2+ 及 Server 分支映射到 6 个 ep_taskbar 变体（rs2/fe/co/ni/zn/ge，代号对应 Windows 开发分支）。所有版本判定走 `RtlGetVersion`+UBR 查询（InitializeGlobalVersionAndUBR，dllmain.c:13928）。

### 5.4 配置系统与更新通道

配置面板是纯 Win32 自绘 UI（ep_gui/GUI.c，4301 行，经主 DLL 导出 `ZZGUI` 触发，dllmain.c:14126-14131），多语言文案来自 ExplorerPatcher-L10N 子模块。配置持久化为注册表 `HKCU\Software\ExplorerPatcher`（ExplorerPatcher/def.h:4 REGPATH），dllmain.c 顶部 100+ 个 DWORD 全局变量与注册表值一一对应（dllmain.c:75-150）；组策略项通过调用系统 reg.exe 提权写入（GUI.c:13-41 SetPolicy）。内置更新器（ExplorerPatcher/updates.cpp，1437 行）用 WinRT Toast 通知提示新版本，支持检查/下载/安装自更新。天气小组件 ep_weather_host 是独立 out-of-proc COM server，抓取 Google 天气页脚本解析渲染（ep_weather_provider_google_script.h）。

## 6. 构建与运行

来自 .github/workflows/build.yml 与 build.proj（官方唯一构建路径，未提供开发者本机 README 指引，文档在 wiki）：

```
nuget restore ExplorerPatcher.sln            # 还原 SlimDetours/WIL
BuildDependenciesRelease.bat                 # 构建 zlib 等依赖
# CI 从 ExplorerPatcher/ep_taskbar_releases 下载预编译 ep_taskbar.*.dll 放入 build/Release
msbuild /m /p:Configuration=Release build.proj   # 编排 IA-32/x64/ARM64(+ARM64X) 四套平台
cp ExplorerPatcher.amd64.dll -> dxgi.dll     # 生成代理 DLL
ep_setup_patch.exe ExplorerPatcher.amd64.dll ep_setup.exe   # DLL 嵌入安装器
```

发布：tag push 自动触发，`ep_generate_release_name/description` 生成发布名与说明，commit message 以 `rel_` 前缀标记正式版否则为 prerelease（build.yml:170-214）。仓库不运行任何测试，`BuildDependencies*.bat` 与 `.sln` 是仅有的工程入口；无 CMake。

## 7. 测试与工程质量

- **无自动化测试**：仓库不含任何测试工程/目录，质量保障依赖 CHANGELOG 中"Tested on OS builds ..."列出的大量实机 build 验证（CHANGELOG.md 头部列出 22621/22631/26100/26300/28000/29695 等十余个 x64/ARM64 build）。
- **CI**：单一 build.yml，windows-2025-vs2026 runner，30 分钟超时，MSBuild 并行 + ARM64X 双架构产物，发布全自动化。依赖固定到 commit SHA（actions 均带 pin）。
- **代码质量观察**：核心 dllmain.c 达 14,132 行单文件，混合 C/C++/WinRT，风格紧凑偏"作者个人工程"；防御性强（进程黑名单、crash 计数器 CrashCounterHandleEntryPoint dllmain.c:10712、符号缺失优雅降级 ep_taskbar 检测 ITrayUI 接口兼容性后回落原生任务栏，CHANGELOG 71.1 条目）。近期提交显示它会在不兼容系统上主动退出而非崩溃。

## 8. 社区与维护状态

- **高度活跃**：最近 push 2026-11-20，最新正式版 26100.9711.71.3 于同日发布；2026 年内 release 轨迹：5 月 70.1 → 6 月 70.2/70.3 → 7 月 70.4(pre) → 10 月 71.1 → 11 月 71.2(pre)/71.3，平均每月 1-2 个版本，微软每个 OS 累积更新后都有对应"pattern scan keep-up"适配版（gh api releases、CHANGELOG）。
- **贡献者结构**：19 人（per_page=100 全量列表），架构与核心修复几乎全部出自作者 valinet（46 岁上下、罗马尼亚籍独立开发者，FUNDING.yml 挂 GitHub Sponsors），外部贡献以符号/模式修复小提交与 L10N 翻译为主；ep_taskbar 等衍生仓库（ExplorerPatcher 组织）亦由其主导。单人高强度维护是该项目的现实结构。
- **社区渠道**：395 个 open issue，Discord 服务器（README 徽章），wiki 承载文档（符号机制、更新配置等）。
- **生态定位**：同类闭源商业产品有 StartAllBack、StartIsBack；开源同类有 Windhawk（模块化，同作者生态）、Open-Shell（仅开始菜单）。ExplorerPatcher 是其中唯一同时覆盖任务栏+开始菜单+Alt+Tab 且 GPL 开源的项目，其"激活系统内遗留代码"路线与 StartAllBack 的"自绘重实现"路线在技术上对立，也决定了两者受 Windows 更新冲击的模式不同。

## 9. 亮点与风险

**亮点**
1. 技术路线聪明：不重绘 UI，而是复活 Windows 系统二进制内残留的 Win10 代码路径，视觉与行为保真度远超模仿实现（Taskbar10.cpp 注释是少见的公开机制说明）。
2. 完整的自研工具链闭环：PDB 解析器、IAT patch 库、SlimDetours 包装、符号服务器缓存、崩溃计数自保护、不兼容自动回落（symbols.c、hooking.h、dllmain.c:10712）。
3. 版本适配工程化：6 个按 Windows 分支代号的 ep_taskbar 变体 + 按构建号自动选择 + CI 自动发版，把"每个补丁日都要修"的苦活做成了流水线（utility.h:1711、build.yml）。
4. 维护强度罕见：Windows 11 每个累积更新后数日内即有适配版本发布，5 年持续（2021-08 至今，gh api/CHANGELOG 可证）。
5. 全栈开源（GPL-2.0），是研究 Windows shell 内部结构与 COM/XAML 加载机制的高价值样本。

**风险/局限**
1. ** Windows 更新强耦合**：工作原理决定其补丁面随每个 OS 更新漂移，更新后出现 explorer 崩溃循环是历史上有过的问题类别（CHANGELOG 中大量 crashloop 修复条目）；安装即改系统进程行为，属于非官方系统修改，微软不承诺兼容，且内部符号/接口随时可能被移除。
2. **安全面**：向 explorer.exe/StartMenuExperienceHost 注入并 hook 系统 API 的机制本身要求高信任；项目以安装器形式提权部署，安全审计应将其视同内核级 shell 扩展对待，第三方渠道分发的安装器存在替换风险（源码内已内置官方安装器哈希自校验以缓解，ep_setup.c:572-588）。
3. **单点维护**：核心符号偏移、模式扫描与 ep_taskbar 重建任务集中在一人手上，bus factor 低。
4. **代码可读性**：1.4 万行 dllmain.c 与全局变量式配置使二次开发门槛偏高；无测试设施，回归只能靠多 build 实机验证。

## 10. 总结评价

ExplorerPatcher 是 Windows 个性化工具链中工程强度最高、维护最活跃的开源项目：它把"Windows 更新日即发版日"做成了 CI 流水线，用 PDB 符号解析、字节模式扫描、IAT/inline hook 双引擎和按 Windows 分支代号组织的多任务栏变体，把对系统内部的依赖转化为一套可持续的适配体系。对想理解 Windows shell 内部机制（CTray 初始化路径、StartDocked 组件激活、feature staging、PDB 符号消费）的开发者，它的源码几乎是公开教材；对需要在 Windows 11 上保留 Win10 工作流的用户，它是最完整的开源选项，但使用前应清楚其系统级修改属性与每次 OS 更新后的兼容不确定性。星数 34k+、5 年不间断的 release 记录印证了其在同类生态中的位置。
