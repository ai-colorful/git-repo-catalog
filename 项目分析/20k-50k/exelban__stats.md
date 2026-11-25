# exelban/stats 项目分析

> 星数 33500（快照 2026-10-24）｜主语言 Swift｜档位 20k-50k｜分析日期 2026-11-13
> 注：元数据核查时（2026-11-13）实际星数已达 43374，仓库持续快速增长。

## 1. 项目速览

一句话定位：macOS 菜单栏系统监控工具——把 CPU/GPU/内存/磁盘/网络/电池/传感器/风扇等指标以可定制小组件的形式常驻菜单栏，点击展开详细图表，是 macOS 平台 iStat Menus 最著名的开源替代品。

| 项目 | 内容 |
|---|---|
| 名称 | exelban/stats（作者 Serhiy Mytrovtsiy） |
| 星数 | 33500（快照）；2026-11-13 实测 43374（gh api） |
| 语言 | Swift（123 个 .swift 文件，约 4.7 万行，含少量 ObjC） |
| 许可证 | MIT（LICENSE） |
| 最近活跃 | pushed 2026-11-12；最新 release v3.0.26（2026-11-08） |
| 创建时间 | 2019-05-29，默认分支 master |
| 发行渠道 | GitHub Releases DMG + Homebrew（`brew install stats`）+ 官网 mac-stats.com |

## 2. 项目是什么，解决什么问题

Stats 解决的问题是 macOS 原生系统不提供常驻、低开销、可定制的性能监控入口。macOS 自带的"活动监视器"是一个独立窗口应用，无法在菜单栏一眼看到实时负载；商业方案 iStat Menus 收费且闭源。Stats 以纯菜单栏形态存在（默认无 Dock 图标，`Stats/AppDelegate.swift` 中 `NSApplication.ActivationPolicy` 动态切换），每个监控维度是一个独立"模块"，模块可以在菜单栏渲染多种形态的小组件（迷你数字、折线图、柱状图、饼图、转速表、标签等），点击小组件弹出该维度的详细面板（进程排行、历史图表、传感器列表等）。

支持的监控维度（`Stats/AppDelegate.swift` 顶部的模块数组）：CPU、GPU、RAM、Disk、Net、Battery、Sensors（温度/电压/功率/风扇转速）、Bluetooth、Clock（多时区时钟）、Remote（远程监控）。README 中的功能列表与之一致。传感器与风扇数据依赖对 Apple SMC（System Management Controller）的底层读取，风扇调速通过一个独立的特权 Helper 完成（`SMC/Helper/`），这是该项目工程复杂度的主要来源之一。

项目自 2025 年起扩展出云同步的远程监控能力：`Kit/plugins/SystemStats.swift` 定义了 `api.system-stats.com` / `oauth.system-stats.com` / MQTT broker（`wss://broker.system-stats.com:8084`）与 free/pro/organization 三档账户计划，`Modules/Remote/` 负责在菜单栏展示其他机器的指标（文件头注释显示该模块创建于 2026-05），README 提供 SSH 无头安装脚本（`curl -fsSL https://cdn.mac-stats.com/install.sh | bash`）。这意味着项目正在从单机工具演变为"本机 + 云端"的监控体系。

值得注意的社区策略：README 明确写着"Open source, but not open contribution"——源码 MIT 完全开放，但不接受未经讨论的 PR，仅翻译和已有重大贡献者的代码例外。这是单人维护大型项目的务实选择，也解释了其贡献者结构（见第 8 章）。

## 3. 技术栈与依赖

- **语言**：Swift 5.0/6.0 混用（各文件头注释标注），传感器读取用少量 Objective-C（`Modules/Sensors/reader.m`、`Kit/lldb/lldb.m`）。
- **UI 框架**：主体 AppKit（Cocoa）；仅桌面小组件扩展用 SwiftUI + WidgetKit（`Widgets/widgets.swift`）。
- **系统层**：IOKit（SMC 读取、电池、HID 传感器）、Mach 层 API（`host_processor_info` 等）、sysctl、私有 API `IOHIDEventSystemClient`（`Modules/Sensors/reader.m`）。
- **存储**：vendored LevelDB 静态库（`Kit/lldb/libleveldb.a`，ObjC 封装 `Kit/lldb/lldb.m`），Makefile 提供 `leveldb` 目标从源码编译（google/leveldb，CMake 双架构）。
- **第三方 Swift 依赖**：无。`Stats.xcodeproj/project.pbxproj` 中无 `XCRemoteSwiftPackageReference`，4 处 `packageProductDependencies` 均为本地 framework（Kit、各模块）。定时器、日志、图表全部自研（`Kit/plugins/Repeater.swift`、`Logger.swift`、`Charts.swift`）。
- **本地依赖模块**：`Kit/`（共享框架）、`LaunchAtLogin/`（开机自启）、`SMC/`（SMC CLI 与特权 Helper）。
- **构建工具链**：Xcode（`Stats.xcodeproj`，多 target）+ Makefile + xcodebuild/notarytool；CI 脚本用 Python（`Kit/scripts/`）。
- **最低系统**：macOS 12 Monterey（`Stats.xcodeproj` 中 `MACOSX_DEPLOYMENT_TARGET = 12.0`，桌面小组件扩展 target 为 14.0；README 同步声明）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `Stats/` | 主应用：AppDelegate、设置窗口（`Views/Settings.swift` 等 7 个视图文件）、启动装配 |
| `Kit/` | 共享框架：模块基类（`module/`）、菜单栏小组件库（`Widgets/` 16 种）、自研插件（`plugins/`：图表/DB/日志/更新器/SystemKit/SystemStats）、LevelDB 封装、构建脚本（`scripts/`） |
| `Modules/` | 10 个监控模块，每模块一个目录（CPU/GPU/RAM/Disk/Net/Battery/Sensors/Bluetooth/Clock/Remote），各自编译为独立 framework |
| `SMC/` | SMC 读取实现（`smc.swift`）、命令行工具（`main.swift`）、特权 Helper（`Helper/`，NSXPC + launchd） |
| `Widgets/` | WidgetKit 桌面小组件扩展（`WidgetsExtension.appex`） |
| `Tests/` | XCTest 与独立脚本测试（6 个文件，约 1400 行） |
| `LaunchAtLogin/` | 登录自启动支持 |
| `Stats.xcodeproj/` | Xcode 工程定义（多 target、build phase 脚本同步版本号） |

组织方式评述：这是典型的"框架 + 插件式模块"单仓库结构，不是 monorepo/多包。所有模块实现同一套基类协议并各自打包成 framework，主应用在 `AppDelegate` 中静态实例化全部 10 个模块（`Stats/AppDelegate.swift`）。模块间零耦合，新增监控维度只需新建一个目录并实现固定文件布局——每个模块都遵循完全一致的文件模板：`main.swift`（模块类）+ `readers.swift`（采集）+ `widget.swift`（菜单栏回调）+ `popup.swift`（下拉面板）+ `settings.swift` + `notifications.swift` + `portal.swift` + `preview.swift` + `config.plist`（声明式元配置）。约 4.7 万行 Swift 全部自研，零第三方 Swift 依赖，是"控制依赖面"的极致样本。

## 5. 核心架构与关键实现

### 5.1 模块基类与声明式配置（Kit/module/module.swift）

`Module` 基类（414 行）承担模块生命周期：从模块 bundle 内的 `config.plist` 读取声明式配置（`module_c` 结构体，`Kit/module/module.swift:20-118`），据此自动实例化小组件、挂载菜单栏、装配窗口与设置页。构造函数通过 NotificationCenter 订阅全局事件（模块开关、弹窗开关、widget 开关），并创建 `Window`（设置/通知容器）与 `PopupWindow`（下拉详情）。模块可用性检查 `isAvailable()` 由子类覆写（如 Sensor 模块在无 SMC 支持的机器上返回 false）。enable/disable 统一驱动 readers 启停与菜单栏显隐，并持久化到 `Store`：

```swift
public func enable() {
    guard self.available else { return }
    self.enabled = true
    Store.shared.set(key: "\(self.config.name)_state", value: true)
    self.startReaders()
    self.menuBar.enable()
    self.window?.setState(self.enabled)
}
```
（出处 `Kit/module/module.swift:215-225`）

`Modules/CPU/config.plist` 展示了配置驱动的 widget 声明方式：6 种 widget 各带 `Default`（是否默认启用）、`Order`（菜单栏排序）、`Preview`（设置页预览值）、`Unsupported colors`，设置 UI 完全由该 plist 生成，模块代码无需写死界面选项。

### 5.2 Reader 采集层（Kit/module/reader.swift + 各模块 readers）

`Reader<T>` 泛型基类（289 行）是全部数据采集的骨架：持有 `Repeater` 定时器（自研，`Kit/plugins/Repeater.swift`），支持按模块设置更新间隔、秒边界对齐（`alignToSecondBoundary`，将采集对齐到整秒以获得更平滑的图表）、弹窗/预览可见性感知的"睡眠模式"（无人观看时暂停采集省电）。采集值经 `callback()` 统一分发：写内存、记录历史（`recordHistory` → LevelDB，带时间有效性校验防时钟漂移污染）、推送给 `SystemStats`（远程监控）、按 `interval*10` 节流落盘：

```swift
public func callback(_ value: T?) {
    let moduleKey = "\(self.module.stringValue)@\(self.name)"
    self.value = value
    if let value {
        self.recordHistory(value)
        self.callbackHandler(value)
        SystemStats.shared.send(key: moduleKey, value: value)
        ...
    }
}
```
（出处 `Kit/module/reader.swift:115-130`）

CPU 模块演示了典型接线（`Modules/CPU/main.swift:143-194`）：6 个 reader（负载/进程/温度/频率/限制/平均负载）按平台条件编译——x86_64 挂 LimitReader，arm64 挂 FrequencyReader。负载读取直接走 Mach 层 `host_processor_info(PROCESSOR_CPU_LOAD_INFO)` 差分计算每核利用率（`Modules/CPU/readers.swift:38-90`），核数用 sysctl `hw.physicalcpu`/`hw.logicalcpu` 获取，E/P/S 核分组依赖 `SystemKit` 的设备信息。数据回流到 `loadCallback` 后按 widget 类型分发（`Modules/CPU/main.swift:197-269`）：Mini 拿总值、BarChart 按 E/P 核簇着色、PieChart 拆 system/user 两色，最后可选推送给 WidgetKit 桌面小组件。

### 5.3 SMC 底层与特权 Helper（SMC/）

`SMC/smc.swift`（722 行）是完整的 Apple SMC 用户态实现：定义 17 种 SMC 数据类型编码（`ui8`/`sp78`/`fpe2`/`flt ` 等）与 IOConnect 调用结构体，通过 IOKit 的 `IOServiceGetMatchingServices`/`IOConnectCallStructMethod` 读取任意 key。同目录 `main.swift` 是独立 CLI（list/fan/set/reset 子命令）。写操作（风扇调速）需要特权，因此单独做了 launchd 守护进程 `SMC/Helper/`：以 `NSXPCListener(machServiceName: "eu.exelban.Stats.SMC.Helper")` 监听（`SMC/Helper/main.swift:53`），主应用通过 `Kit/helpers.swift` 中的 XPC 客户端连接，空闲自动退出（`HelperConnectionState.beginShutdownIfIdle`）。安装脚本 `Kit/scripts/uninstall.sh` 提示"需要管理员权限移除 SMC helper"，对应 README 中 Homebrew 安装的_privileged helper_ 处理。Sensors 模块则用了另一条路：`Modules/Sensors/reader.m` 通过私有 API `IOHIDEventSystemClientRef` 读取 HID 传感器事件页（温度等），并在文件头声明了 service 发现与缓存逻辑，这是 Apple Silicon 上获取温度的主要途径之一。

### 5.4 菜单栏 UI 与双端 widget 体系（Kit/module/widget.swift + Kit/Widgets/）

菜单栏渲染为纯 AppKit：`MenuBar` 类（`Kit/module/widget.swift:384`）持有 `NSStatusItem`，每个模块可同时激活多个小组件（`SWidget` 协议），支持"合并为一格"模式（`CombinedModules`，`Stats/Views/CombinedView.swift`）。16 种小组件实现在 `Kit/Widgets/`（BarChart/LineChart/PieChart/Tachometer/Memory/Battery/NetworkChart/Label/Dot/Icon 等），全部是 `WidgetWrapper`（NSView 子类）直接绘制，例如 `Kit/Widgets/BarChart.swift`。下拉详情面板 `PopupWrapper` 同样是 NSStackView + 手写布局（`Kit/module/popup.swift:44-60`），图表绘制复用 `Kit/plugins/Charts.swift`（1723 行自研图表库）。SwiftUI 仅出现在 `Widgets/` 目录的 WidgetKit 扩展中（`Widgets/widgets.swift:23-33`：CPUWidget/GPUWidget/RAMWidget/DiskWidget/NetworkWidget/UnitedWidget 六个桌面小组件），且默认关闭（README FAQ 说明因 chronod 高负载问题默认禁用，需在设置中手动开启）。

## 6. 构建与运行

构建依赖 macOS + Xcode（本仓库无法在 Windows 上构建，以下命令来自 Makefile 与 CI，供参考）：

```bash
# 完整发布流水线（Makefile:12）：版本号自增 → 归档 → 公证 → 签名 → 校验 → DMG
make build            # 等价于 test-smc clean next-version archive notarize sign verify prepare-dmg prepare-dSYM open

# 单步归档（Makefile:16-35）
xcodebuild -scheme Stats -destination 'platform=OS X,arch=x86_64' -configuration Release archive \
  -archivePath build/Stats.xcarchive
xcodebuild -exportArchive -exportOptionsPlist exportOptions.plist ...

# SMC 测试（独立于 Xcode test target）
make test-smc         # python3 Kit/scripts/run-smc-tests.py

# 重建 vendored LevelDB 静态库
make leveldb          # clone google/leveldb → cmake 双架构 → 拷入 Kit/lldb/libleveldb.a

# 用户安装
brew install stats    # 或 GitHub Releases 下载 Stats.dmg
```

公证与签名通过 `xcrun notarytool submit --keychain-profile "AC_PASSWORD"` + `xcrun stapler staple`（Makefile:37-52），DMG 用第三方 create-dmg 工具生成（Makefile:85-103）。版本号管理：`CFBundleVersion`（build 号，当前 856）由 Makefile 每次构建自增，营销版本号由 release 时写入 pbxproj，build phase 脚本再把版本同步进 Widgets 扩展的 Info.plist（`Stats.xcodeproj/project.pbxproj:2157`）。

## 7. 测试与工程质量

- **测试规模**：`Tests/` 6 个文件共约 1400 行。`Tests/SMC.swift`（1027 行）是最大的一块——配合 `Kit/scripts/run-smc-tests.py` 实现了一套巧妙的"边界注入"测试：脚本用正则从源码中抽取真实的 `HelperConnectionState` 等逻辑类，把 `IOServiceGetMatchingServices`、`IOConnectCallStructMethod`、`usleep`、`NSXPCConnection` 等系统边界替换为 Fake（`run-smc-tests.py:1-50` 注释明确说明"Production source is unchanged"），从而在无特权、无真实 SMC 硬件的 CI 环境里测试特权 Helper 的生命周期。其余为 XCTest 单元测试（如 `Tests/Kit.swift` 测版本比较逻辑）。
- **CI**（`.github/workflows/`）：`build.yaml`（push/PR 跑 xcodebuild Release 归档，tag 时跳过）、`release.yaml`（v* tag 触发：changelog.py 生成变更日志 → 导入证书 → 归档 → 导出 → 公证 → 发布）、`smc-tests.yaml`（按路径触发 SMC 测试）、`i18n.yaml`（i18n.py 校验翻译完整性）、`linter.yaml` + 根目录 `.swiftlint.yml`。
- **质量观察**：代码风格高度统一（每文件统一头注释格式、SwiftLint 约束）；自研了日志（NextLog）、定时器（Repeater）、存储缓存（Store 双层缓存 + 串行队列）、DB 等基础设施以保持零依赖；reader 的睡眠模式、秒对齐、历史校验等细节显示对功耗与数据质量有工程级考量（`Kit/module/reader.swift`）。翻译覆盖 52 个 .lproj 目录（全仓库统计），本地化是社区参与的主要形式。

## 8. 社区与维护状态

- **活跃度**：单维护者驱动（README"developed and maintained by a single person"）。仓库 2026-11-12 仍有 push，最新 release v3.0.26 发布于 2026-11-08（gh api），构建号已到 856（`Stats/Supporting Files/Info.plist`），发布节奏约为每周多个 patch 版本。
- **贡献者**：GitHub contributors API 分页显示约 194 名贡献者（含匿名），结合"不接受未讨论 PR"的 README 政策，绝大多数为翻译贡献者（README 中列出了 35+ 种语言及其贡献者致谢）。核心 Swift 代码基本出自作者一人。
- **issue 管理**：README 有详细的 FAQ 解释 issue 关闭策略（重复即关）、崩溃报告模板（`.github/ISSUE_TEMPLATE/bug_report.yml`）、且明确不支持 macOS beta。
- **商业化路径**：2026 年新增的 Remote/SystemStats 云服务（free/pro/organization 三档账户，`Kit/plugins/SystemStats.swift:19-22`）+ 官网分发 + 无头安装脚本，显示项目正走向"开源核心 + 托管服务"模式。风扇控制明确标记为 legacy 不再维护（README FAQ），作者坦承无精力支持。
- **增长势头**：星数从任务快照 33500（2026-10-24）到 43374（2026-11-13 实测），约三周增长近万，处于快速上升期。

## 9. 亮点与风险

**亮点**
1. 极致的模块化架构：config.plist 声明式配置 + 统一模块文件模板 + 零耦合 framework，新增监控维度成本极低，是 Swift 桌面应用插件式架构的优秀教材（`Kit/module/module.swift`、`Modules/CPU/config.plist`）。
2. 系统层深度：自研 SMC 协议实现、特权 XPC Helper、Mach 层差分采集、私有 HID 传感器 API、vendored LevelDB，全套底层能力不依赖任何第三方 Swift 包，可读性和可控性极强（`SMC/`、`Kit/lldb/`）。
3. 功耗工程：reader 睡眠模式（无可视消费者即暂停）、按模块估价的可关闭性、秒边界对齐采集，把菜单栏常驻应用的能耗做到认真程度（`Kit/module/reader.swift:156-181`、README"energy impact"FAQ）。
4. 发布流水线完整：CI 归档 → 公证 → 签名 → DMG → 自更新（DMG 挂载安装，`Kit/plugins/Updater.swift`）全链路自动化，普通用户 `brew install stats` 即用。
5. 测试巧思：用源码注入 Fake 边界的方式在 CI 上测试特权 Helper，绕开了"特权代码不可在 CI 运行"的死结（`Kit/scripts/run-smc-tests.py`）。

**风险/局限**
1. 单人维护 + 关闭式贡献策略：PR 基本不接受，bus factor 为 1；新硬件（新 Apple Silicon 芯片的传感器 key）适配速度依赖作者个人，README 中作者公开征集 Apple Silicon 传感器映射知识。
2. 依赖私有/不稳定系统接口：`IOHIDEventSystemClient` 私有 API 与 SMC key 布局随 macOS 版本漂移，macOS 大版本升级后容易出现数据缺失（README 关于 macOS 26 菜单栏隐私开关、chronod 的 FAQ 即为实例）。
3. 云化带来的信任面扩大：Remote 模块接入 OAuth + MQTT + 自有云服务，与"无遥测"承诺并存，商业化走向待观察（`Kit/plugins/SystemStats.swift`）。
4. 测试覆盖偏科：仅 Kit/SMC/RAM/Disk/Bluetooth 有少量测试，模块 UI、采集精度等核心路径无自动化保障；仓库无 CHANGELOG 文件（发布时由脚本从 git tag 生成，`.github/workflows/release.yaml`）。

## 10. 总结评价

Stats 是 macOS 菜单栏监控领域事实上的开源标杆：以零第三方 Swift 依赖的约 4.7 万行自研代码，覆盖了商业产品 iStat Menus 的绝大部分能力，并在 2026 年借 Remote 模块迈出云端化一步。它的最大价值在于架构示范——声明式模块配置、统一 Reader 抽象、特权 Helper 隔离、功耗感知采集，每一层都值得 Swift 系统级应用开发者精读；同时它也是"单人维护开源产品如何做减法"的活案例（拒绝随意 PR、放弃风扇控制维护、聚焦核心体验）。适合三类人：想要免费可靠系统监控的 macOS 用户（直接 brew 安装）、想学习 AppKit 菜单栏应用与 macOS 底层（SMC/XPC/IOKit/Mach）的 Swift 开发者（读 Kit/ 与 SMC/）、以及研究开源可持续性的人（观察其 open-source-but-not-open-contribution 模式与云服务化路径）。
