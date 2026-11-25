# zhongyang219/TrafficMonitor 项目分析

> 星数 33500（快照 2026-10-24）｜主语言 C++｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：Windows 平台的网速/CPU/内存桌面悬浮窗监控工具，可嵌入任务栏显示、支持皮肤与插件系统的经典 MFC 桌面软件。

| 项目 | 内容 |
| --- | --- |
| 名称 | TrafficMonitor |
| 星数 | 33500（快照；GitHub API 当前 46813，见注） |
| 主语言 | C++（C++20，MFC/Win32） |
| 许可证 | Anti-996 License 1.0（`LICENSE`，GitHub 标记 NOASSERTION） |
| 最新版本 | V1.86（2026-03-29，`UpdateLog/update_log.md`） |
| 最近提交 | 2026-09-27（gh api commits） |
| 创建时间 | 2017-07-30，fork 3837，贡献者 57（gh api） |
| 默认分支 | master |

注：任务快照星数 33500 与 gh api 实时返回的 46813 有差距，以快照为准记录；仓库实际增长更快。

## 2. 项目是什么，解决什么问题

TrafficMonitor 是一款 Windows 桌面悬浮窗软件，实时显示当前网速、CPU 与内存利用率，并支持将显示内容直接嵌入到系统任务栏中，是国内 C++ 桌面工具领域星数最高的项目之一（README.md「TrafficMonitor 简介」）。它解决的核心痛点是：Windows 自身不提供常驻、轻量、可视化的网速与系统负载监视，而第三方任务栏显示方案大多依赖 WPF/Qt 等较重框架，本项目用原生 Win32/MFC 实现了几乎零依赖的方案。

功能上它覆盖了三块：其一是数据监控——网速（可自动/手动选择网卡、支持汇总所有连接）、CPU/内存利用率、CPU 频率、显卡与硬盘利用率（V1.86 起 Lite 版也支持），标准版还有 CPU/显卡/主板/硬盘温度监控（README 版本对比表）；其二是展示形态——主悬浮窗（可换肤、透明、鼠标穿透）+ 任务栏窗口（Win7~Win11 均适配，含副显示器任务栏、Wine 环境，见 `WineTaskbarDlg.cpp`）；其三是外围功能——历史流量统计（按日/月，`HistoryTrafficFile.cpp`）、网络详细信息、多语言（18 种语言的外部 ini，`TrafficMonitor/language/`）、开机自启、更新检查（GitHub/Gitee 双源，`version_utf8.info`）。

从 V1.86 开始项目做了一个重要的架构收缩：温度监控功能从主程序剥离到独立的"硬件监控插件"（README 明确说明标准版将停止提供，仅保留 Lite 版，温度交给 `TrafficMonitorPlugins` 仓库的插件）。这体现了作者从"单体内置一切"向"核心+插件生态"演进的方向。

代码组织上提供两个解决方案：`TrafficMonitor.sln`（含 OpenHardwareMonitorApi 温度监控）与 `TrafficMonitor_Lite.sln`（`WITHOUT_TEMPERATURE` 条件编译，不需要管理员权限运行）。README 中专门给出了两版功能对比表并建议普通用户使用 Lite 版。

## 3. 技术栈与依赖

- **语言/标准**：C++20，`PlatformToolset v143`（VS2022），`WindowsTargetPlatformVersion 10.0`（`TrafficMonitor/TrafficMonitor.vcxproj`）。
- **UI 框架**：MFC（CDialogEx/CDialog 体系，约 90 个对话框/控件类）；绘制层抽象为 `IDrawCommon`，实现有 GDI 与 Direct2D/DirectWrite（`D2D1Support.cpp`、`Dxgi1Support2.cpp`、`DCompositionSupport.cpp`、`DrawCommonFactory.cpp`），仅透明且支持 D2D 时任务栏窗口才启用 D2D 渲染（`TaskBarDlg.cpp:1004` 附近 `DisableRenderFeatureIfNecessary`）。
- **系统 API**：
  - 网速：IP Helper（`GetIfTable` 读 `MIB_IFTABLE` 的 `dwInOctets/dwOutOctets`、`GetAdaptersInfo` 读网卡信息），见 `TrafficMonitorDlg.cpp:1201-1250`、`AdapterCommon.cpp:14`。
  - CPU/磁盘/GPU：PDH 性能计数器（`PdhOpenQuery/PdhAddCounter`，`TrafficMonitor/PdhHardwareQuery/`），CPU 失败时回退 `GetSystemTimes`（`CPUUsage.cpp:GetCpuUsageByGetSystemTimes`）。
  - 内存：`GlobalMemoryStatusEx`（`TrafficMonitorDlg.cpp:1460`）。
  - 温度/硬件：C++/CLI 包装 .NET 库 LibreHardwareMonitorLib.dll（`OpenHardwareMonitorApi/OpenHardwareMonitorImp.cpp`，使用 `IHardware^` 托管句柄）。
- **第三方/内置源码**：tinyxml2（`TrafficMonitor/tinyxml2/`，解析皮肤 XML 与配置）；GDI+（`SkinFile.h` 引 `<gdiplus.h>`）。
- **无包管理器**：全部依赖为 Windows SDK + MFC + 一个随仓库分发的 .NET DLL，构建零 NuGet 依赖（`TrafficMonitor.sln` 仅 3 个 vcxproj）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `TrafficMonitor/` | 主程序全部源码（约 193 个文件），含对话框、绘制、皮肤、插件管理、PDH 采集等 |
| `TrafficMonitor/PdhHardwareQuery/` | PDH 性能计数器封装：CPU/GPU/磁盘利用率与 CPU 频率 |
| `TrafficMonitor/skins/` | 15 套内置皮肤（每套 `skin.ini` + `background.png`/`background_l.png`），另有 `xml_test/` 为 XML 格式样例 |
| `TrafficMonitor/language/` | 18 个语言 ini（V1.86 起多语言字符串外置） |
| `TrafficMonitor/tinyxml2/` | 内置的 tinyxml2 源码 |
| `OpenHardwareMonitorApi/` | C++/CLI 工程，包装 LibreHardwareMonitorLib.dll 提供温度/硬件信息接口 |
| `PluginDemo/` | 官方示例插件（系统时间/日期/自绘项目），是插件开发模板 |
| `include/` | 对外头文件：`PluginInterface.h`（插件 ABI）、`OpenHardwareMonitor/` |
| `UpdateLog/` | 三语更新日志 |
| `Screenshots/`、`images/` | README 截图与素材 |
| 根目录 | `TrafficMonitor.sln`、`TrafficMonitor_Lite.sln`、`version.info`/`version_utf8.info`（更新源元数据）、`皮肤制作教程.md` |

组织方式评述：单解决方案多工程的典型 Windows 桌面单体架构——主工程 + 一个 C++/CLI 桥接工程 + 一个示例插件工程。代码全部平铺在 `TrafficMonitor/` 下（无子目录分层，靠文件名前缀区分职责），221 个 C++/H 文件约 4.2 万行。这种平铺在规模变大后可读性一般，但每个功能域（Skin、Taskbar、HistoryTraffic、Plugin）都有清晰的前缀类族。

## 5. 核心架构与关键实现

### 5.1 网速采集（IP Helper + 差分计算）

数据源是 `GetIfTable`：每个监控周期读取各网卡的累计字节数 `dwInOctets/dwOutOctets`，与上次值做差再除以时间间隔得到速率（`TrafficMonitor/TrafficMonitorDlg.cpp` 的 `DoMonitorAcquisition`，约 1201-1250 行）。支持"当前选中连接"与"所有连接求和"两种模式；网卡元信息（IP/掩码/网关）用 `GetAdaptersInfo` 获取（`AdapterCommon.cpp:14-60`）。两个健壮性细节值得注意：`GetIfTable` 全程用 SEH `__try/__except` 包裹，异常时重建缓冲区重试；睡眠恢复后重新初始化连接（V1.86 更新日志）。网卡与字节数的映射维护在 `AdapterCommon.cpp:77-110` 的 `GetIfTableInfo`。

### 5.2 CPU/内存/硬件监控（PDH 为主，多重回退）

CPU 利用率优先走 PDH 计数器，Win10+ 用 `\Processor Information(_Total)\% Processor Utility`，旧系统用 `% Processor Time`；PDH 失败则回退 `GetSystemTimes` 用两次采样差计算（`TrafficMonitor/PdhHardwareQuery/CPUUsage.cpp:11-79`）。PDH 封装 `PdhQuery.cpp:20-30` 先 `PdhAddCounter`，失败再试 `PdhAddEnglishCounter`（解决本地化计数器名问题）。内存直接 `GlobalMemoryStatusEx` 取 `dwMemoryLoad`（`TrafficMonitorDlg.cpp:1460-1465`）。GPU/硬盘利用率同样走 PDH（`GpuUsage.cpp`、`DiskUsage.cpp`）。温度监控单独成桥接工程：`OpenHardwareMonitorApi/OpenHardwareMonitorImp.cpp` 用 C++/CLI 调用 LibreHardwareMonitorLib，主程序通过 `theApp.m_pMonitor->GetHardwareInfo()` 访问，且用 `CSingleLock` 加锁、SEH 包裹（`TrafficMonitorDlg.cpp:1471-1485`），并用 `WITHOUT_TEMPERATURE` 宏整体隔离。

### 5.3 任务栏嵌入（项目最有技术含量的部分）

核心思路是把自己的窗口 `SetParent` 成任务栏的子窗口。关键代码在 `TrafficMonitor/TaskBarDlg.cpp:1017`：

```cpp
m_hTaskbar = FindTaskbarHandle(m_is_secondary_display); //查找任务栏的句柄
::GetWindowRect(m_hTaskbar, m_rcTaskbar);   //获得任务栏的矩形区域
m_connot_insert_to_task_bar = !(::SetParent(this->m_hWnd, GetParentHwnd()));
```

句柄查找按系统版本分三路：主屏任务栏是 `FindWindow("Shell_TrayWnd")`（`TaskBarDlg.cpp:628`）；副屏任务栏用 `EnumWindows` 枚举 `Shell_SecondaryTrayWnd` 类并按显示器位置排序（`TaskbarHelper.cpp:35-115`）；Win7-10 经典任务栏再深入一层 `Shell_TrayWnd → ReBarWindow32/WorkerW → MSTaskSwWClass/MSTaskListWClass`，并把窗口挂到 ReBar 容器上（`ClassicalTaskbarDlg.cpp:72-102`，`GetParentHwnd` 返回 `m_hBar`）；Win11 则直接挂到 `Shell_TrayWnd`，通过 `TrayNotifyWnd` 与 `Start` 两个子窗口的实时矩形计算自己的位置，避开通知区和开始按钮（`Win11TaskbarDlg.cpp:11-102`），还要处理"任务栏居中/靠左"、"小组件按钮占位"、"Web Experience Pack"等 Win11 变量。嵌入失败时降级为置顶窗口跟随任务栏矩形（`TaskBarDlg.cpp:488-493`）。配套工程细节包括 `WS_EX_TOOLWINDOW` 隐藏自身任务栏图标（`TaskBarDlg.cpp:1006`）、按所在显示器 DPI 缩放（`SetDPI/DPIFromRect`）、以及 100ms 的 `TASKBAR_TIMER` 高频重绘（`TrafficMonitorDlg.cpp:1180`）。

### 5.4 皮肤与插件体系

皮肤是"ini 布局 + PNG 背景图"的组合：每个皮肤目录含 `skin.ini`（`[skin]` 文本颜色/作者、`[layout]` 中每个显示项的 x/y/width/align，且大小两套坐标 `_l/_s`）与背景图，`SkinFile.h` 定义 `SkinInfo/Layout/LayoutItem` 结构解析加载，`SkinManager.cpp` 管理皮肤列表；同时兼容 XML 格式（`skins/xml_test/skin.xml`）。主窗口换肤、每皮肤独立字体/颜色/文本保存是 V1.86 的新特性。

插件接口定义在 `include/PluginInterface.h`：纯虚类 `IPluginItem`（显示项：名称/ID/标签文本/数值文本/自绘 `DrawItem`/鼠标键盘事件）与 `ITMPlugin`（插件入口），注释明确要求插件在 `DataRequired` 回调里取数而非在 `GetItemValueText` 里实时取。主程序侧 `PluginManager.cpp:35-110` 扫描 Plugins 目录下所有 dll，`LoadLibrary` 后取插件实例，还通过修改插件导入表 hook 了 User32 的 DrawText 系列函数（`PluginManager.cpp:428-469`），用于插件项目的暗色模式适配——这是相当底层的处理。`PluginDemo/` 提供了完整示例，温度监控插件也基于同一接口。

### 5.5 线程与刷新模型

UI 线程三个定时器：`MAIN_TIMER` 1000ms 刷新界面、`TASKBAR_TIMER` 100ms 刷新任务栏窗口、`MONITOR_TIMER`（用户可配 `monitor_time_span`）只置位 `m_monitor_data_required` 标志；实际采集在 `AfxBeginThread` 启动的独立线程 `MonitorThreadCallback` 中执行（`TrafficMonitorDlg.cpp:1154、1583-1613`），采集完成后通知 UI，避免慢速 API 卡界面。崩溃捕获（`crashtool.cpp`，V1.86 支持调用堆栈显示与写日志）、开机自启（`auto_start_helper.cpp`）、双源更新检查（`UpdateHelper.cpp` + `version_utf8.info`）构成外围支撑。

## 6. 构建与运行

- 环境：Visual Studio 2022（v143 工具集，Windows 10 SDK），无第三方包管理依赖。
- 构建：打开 `TrafficMonitor.sln`（标准版，含温度）或 `TrafficMonitor_Lite.sln`（Lite 版），选择 `Release (lite)`/Release 配置，x86/x64/ARM64EC 三平台，直接 msbuild。CI 中的实际命令（`.github/workflows/main.yml`）：

```bash
msbuild TrafficMonitor_Lite.sln -p:configuration="Release (lite)" -p:platform=x64 -p:platformToolset=v143
```

- 输出：`Bin/x64/Release (lite)/TrafficMonitor.exe` 及配套 dll（main.yml artifact 路径）。
- 运行：标准版需管理员权限（读温度），Lite 版普通权限即可；插件放入程序目录 `Plugins/` 子目录自动加载（`PluginManager.cpp:35-41`）。规范要求不运行构建，以上均来自工程文件与 CI 配置。

## 7. 测试与工程质量

- 测试：无单元测试框架与测试目录；`Test.cpp/Test.h` 是主程序内的内部调试对话框而非自动化测试。质量保障主要靠 CI 编译 + 用户反馈 issue。
- CI：`.github/workflows/main.yml` 为 Release CI，仅由 `v*` tag 或手动触发，做 x86/x64（含 ARM64EC）三平台 msbuild 并上传 artifact/PDB，不跑测试。
- 工程质量观察：优点是健壮性处理密集——SEH 包裹系统 API、PDH 多级回退、DPI/多显示器/睡眠唤醒等边界场景都有对应代码，且更新日志的 bugfix 粒度极细（V1.86 修了十几个精确到控件行为的 bug）；缺点是 193 个文件全部平铺在单目录、大量 3000+ 行的巨型类（`TrafficMonitorDlg.cpp` 3132 行、`TaskBarDlg.cpp` 1501 行），且源码为 GBK 编码混合 UTF-8 BOM 文件，跨工具协作有一定摩擦。

## 8. 社区与维护状态

- 活跃度：最近提交 2026-09-27（"readme更新"），此前 2026-08-31 有两个功能修复提交（gh api commits），仓库持续维护中。
- Release 节奏：V1.80（2021-05）→ V1.86（2026-03），其中 2022-11 至 2025-01 有约两年空档，之后恢复活跃（`UpdateLog/update_log.md` 版本列表）。每个 release 提供 x86/x64/arm64ec × 标准/Lite 共 6 个 zip，GitHub 与 Gitee 双渠道分发（`version_utf8.info`）。
- 贡献者：57 人（gh api contributors），核心由作者 zhongyang219 主导，issue 数量很大（open issues 1370，多数为用户求助，gh api）。
- 文档：README 中英双语 + 详细 Wiki + `Help.md` 常见问题 + `皮肤制作教程.md` + 三语更新日志，文档体系在同类个人项目中非常完善。

## 9. 亮点与风险

亮点：
1. 任务栏嵌入实现完整覆盖 Win7→Win11（含副屏、Wine、DPI、Web Experience Pack 变量），是 Windows 桌面逆向式适配的教科书级参考（`TaskBarDlg.cpp`/`ClassicalTaskbarDlg.cpp`/`Win11TaskbarDlg.cpp`）。
2. 数据采集层健壮且无依赖：PDH 多级回退 + SEH 保护 + GetSystemTimes 兜底，普通权限即可运行 Lite 版。
3. 插件 ABI 设计清晰（纯虚接口 + C 字符串边界），并演进出了真实的插件生态（硬件监控插件已独立成 TrafficMonitorPlugins 仓库）。
4. 渲染层抽象（GDI/D2D/DirectWrite 可切换）与皮肤系统（ini 布局+PNG）解耦，社区产出了十余套第三方皮肤。
5. 本地化做到 18 种语言且字符串外置 ini，中文开源桌面工具中罕见。

风险/局限：
1. Anti-996 许可证限制了商业使用，企业环境采用需注意（`LICENSE`）。
2. 代码全平铺 + 巨型对话框类 + GBK 编码，二次开发与代码导航成本偏高。
3. 强耦合 Windows 未公开行为（任务栏子窗口类名、ReBar 结构），Windows 大版本更新可能随时破坏嵌入逻辑，历史上已多次出现适配性修复。
4. 无自动化测试，回归依赖人工与用户反馈；温度监控从主程序迁往插件后，标准版功能处于过渡状态。

## 10. 总结评价

TrafficMonitor 是"把一个看似简单的小工具做到极致"的典范：用约 4.2 万行原生 C++/MFC 实现了网速/CPU/内存监控、任务栏注入、皮肤引擎、插件系统和 18 语国际化，全部零第三方依赖。它非常适合三类人：想要轻量网速/负载监视的 Windows 用户（直接用 Lite 版）；研究 Win32 任务栏机制、DPI 适配、PDH 性能计数器、C++/CLI 混编的开发者（读 `TaskBarDlg`、`PdhHardwareQuery`、`OpenHardwareMonitorApi` 三个模块收益最大）；以及想开发 TrafficMonitor 插件的贡献者（`include/PluginInterface.h` + `PluginDemo` 即可上手）。对想学习现代 C++ 工程组织的人它不是好样本，但对想深入 Windows 桌面底层机制的人来说，它是中文开源世界里少有的高质量实战教材。
