# hyprwm/Hyprland 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 C++｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：Hyprland 是一个 100% 独立实现（不基于 wlroots/libweston/kwin/mutter）的动态平铺 Wayland 合成器，以动画特效（blur、渐变边框、贝塞尔/spring 动画曲线）和高度可定制著称。

| 项目 | 内容 |
|---|---|
| 名称 | hyprwm/Hyprland |
| 星数 | 39,500（快照 2026-10-24）；2026-11-14 实查 40,647（gh api） |
| 语言 | C++（C++26 标准，src/CMakeLists.txt:101） |
| 许可证 | BSD 3-Clause，Copyright (c) 2022-2026 vaxerski（LICENSE） |
| 版本 | 0.57.0（VERSION 文件） |
| 最近活跃 | 最近 push 2026-11-13（gh api），提交极频繁 |
| 创建时间 | 2022-03-16（gh api） |
| 贡献者 | 约 784 人（gh api contributors 分页计数） |
| forks / open issues | 2,180 / 207（gh api，2026-11-14） |

## 2. 项目是什么，解决什么问题

Wayland 生态中的平铺合成器长期面临一个矛盾：Sway 等注重功能但视觉朴素，而要做出 blur、动画、阴影等"eyecandy"通常要牺牲性能或依赖复杂的补丁。Hyprland 的定位是把动态平铺、强定制与视觉效果放在一个二进制里，同时保持响应速度（README.md#Features）。

技术上它最重要的决策是**摆脱 wlroots**：README 明确宣称 "100% independent, no wlroots, no libweston"，并在 Special Thanks 中把 wlroots 列为 "For powering Hyprland in the past"（README.md:40,95）。底层 DRM/KMS/输入后端已迁移到自研的 aquamarine 库（CMakeLists.txt:157 `aquamarine>=0.15.0` 必装依赖），Wayland 协议的服务端实现也全部在仓库内自写（src/protocols/ 约 78 个协议实现），由自研的 hyprwayland-scanner 从 XML 生成胶水代码（CMakeLists.txt:296,495-618）。这意味着 Hyprland 是少数自己实现整套合成器核心的项目之一。

另一个显著特点是**功能面宽**：完整窗口规则系统、特殊工作区（scratchpad）、窗口分组（tabbed）、swallow（吞没终端）、tearing（游戏低延迟）、原生 IME 支持、基于 socket 的 IPC、C/C++ 插件 + 内置插件管理器 hyprpm（README.md:36-65；hyprpm/）。布局支持 Dwindle、Master、Monocle、Scrolling 四种内置算法，并允许插件或 Lua 注册自定义算法（src/layout/algorithm/，PluginAPI.hpp:180-181）。

值得注意：**主分支上配置系统正在从 hyprlang(.conf) 迁移到 Lua 5.5**。配置管理枚举只剩 `CONFIG_LUA`（src/config/ConfigManager.hpp:41-43），默认配置以 C++23 `#embed` 直接内嵌 `example/hyprland.lua`（src/config/lua/DefaultConfig.hpp:15），全仓库源码中 hyprlang 仅剩字符串痕迹（src/helpers/Splashes.hpp）。这是 0.57 开发周期的大改，老用户需迁移配置（未确认旧版兼容层是否在其他发布分支维护）。

## 3. 技术栈与依赖

- 语言：C++26（大量使用 `std::expected`、`std::ranges`、`std::print`、`#embed`，见 src/main.cpp:14、src/config/lua/DefaultConfig.hpp:15）；Lua 5.5 用于配置与脚本（CMakeLists.txt:294）
- 构建：CMake ≥ 3.30 + 自封装 Makefile（Makefile），Nix flake 一等支持（flake.nix），Python3 必需（生成 LuaLS stubs，CMakeLists.txt:472-491）
- 自研 hyprwm 生态库（均经 pkg-config 必装）：
  - aquamarine ≥0.15.0 —— DRM/KMS/seat 后端抽象（CMakeLists.txt:152,157）
  - hyprutils ≥0.14.3、hyprcursor ≥0.1.7、hyprgraphics ≥0.5.1（CMakeLists.txt:153-160）
  - hyprland-protocols ≥0.8.0 —— 自有协议定义（CMakeLists.txt:541）
  - hyprwayland-scanner ≥0.3.10 —— 协议代码生成器（CMakeLists.txt:296）
- 第三方：wayland-server ≥1.22.91、xkbcommon ≥1.11.0、libinput ≥1.29、libdrm、gbm、cairo/pango/pixman（文本渲染）、glslang + OpenGL ES3（着色器编译）、glaze（JSON）、re2、muparser、lcms2（色彩管理）、libcanberra（CMakeLists.txt:271-292）；可选 Tracy profiler 与 udis86（subprojects/，用于函数 hook 反汇编）
- XWayland 可选（NO_XWAYLAND 开关，依赖 xcb 系列，CMakeLists.txt:385-409）

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| src/ | 全部核心源码，约 14.2 万行 C++（实测统计），按子系统分约 28 个子目录 |
| src/protocols/ | Wayland 服务端协议实现，157 文件/2.57 万行，最大模块 |
| src/render/ | 渲染器：GLRenderer、pass/元素化渲染管线、scene、blur、装饰、shaders（1.92 万行） |
| src/config/ | Lua 配置引擎 + 值系统 + 绑定（1.81 万行，135 文件） |
| src/desktop/ | 桌面对象层：window/layer/popup/group/session-lock/规则（1.68 万行） |
| src/managers/ | 事件循环、Seat、协议、权限、输入、截屏共享等管理器（1.49 万行） |
| src/layout/ | 布局框架与 dwindle/master/monocle/scrolling 算法（9.8 千行） |
| src/ipc/ | s1=hyprctl 命令 socket，s2=事件广播 socket |
| src/xwayland/ | XWayland 集成（XConnection、DnD、数据源） |
| src/helpers/ | 数学/信号/内存/时间等基础工具 |
| protocols/ | 内置的 wlr-* 与自有协议 XML 定义（14 个） |
| hyprctl/ | 独立的 hyprctl CLI 工具（CMake 子项目） |
| hyprpm/ | 插件管理器（独立二进制，拉源码+头文件编译插件） |
| hyprtester/ | 集成测试框架（clients/fixtures/plugin/protocols） |
| tests/ | GTest 单元测试（13 个子系统目录） |
| assets/、docs/、example/、systemd/、start/ | 资源、man 页、默认 Lua 配置、session 文件、start-hyprland 看门狗启动器 |

组织方式评述：单仓库单体应用 + 少量 vendored 子项目（tracy、udis86），依赖的 hyprwm 小库全部走系统包管理器安装，不用 submodule/subproject 拉取（subprojects/ 仅 2 个）。代码风格高度统一（C class `C*`、struct `S*`、接口 `I*`、成员 `m_*`，AGENTS.md 有成文规范），近期经历大规模重构（desktop/view、layout/algorithm、render/pass 的分层都是新式抽象）。

## 5. 核心架构与关键实现

### 5.1 事件循环：以 wl_display_run 为干心的多路整合

`CEventLoopManager::enterLoop()`（src/managers/eventLoop/EventLoopManager.cpp:124-140）把四类事件源挂到 wayland 事件循环上：timerfd 定时器（自管理优先队列 `nudgeTimers` 计算最近触发时刻，:191-213）、aquamarine 后端 FD（DRM/libinput 动态增删，`pollFDsChanged` 信号驱动 `syncPollFDs`）、配置 inotify 热重载 FD、以及 `doLater` 空闲任务队列（wl_event_loop_add_idle，:215-233）。主线程最终阻塞在 `wl_display_run`（:137），由 `CCompositor::startCompositor()` 收尾进入（src/Compositor.cpp:841-843）。这个设计把 libinput/DRM 事件、wayland client 事件、定时器与配置监听统一在单线程循环内，是理解全项目并发模型（基本单线程 + 显式 async）的入口。

### 5.2 渲染：接口化 Renderer + Pass 元素管线

渲染层已抽象出 `Render::IHyprRenderer` 接口与 GL 后端 `Render::GL::CHyprGLRenderer`（src/render/GLRenderer.hpp:14），纹理/帧缓冲/同步 FD 全是接口（createTexture/createFB/createSyncFDManager），为未来 Vulkan 等后端留了路。实际绘制走"场景-通道"模型：src/render/scene/MonitorScene.cpp 收集每个显示器的绘制元素，src/render/pass/ 下 26 种 PassElement（SurfacePassElement、BorderPassElement、ShadowPassElement、PreBlurElement 等）构成有序渲染队列，blur 作为独立 provider（IGLBlurProvider）插在 pass 之间，带 damage 跟踪（SurfaceDamage.cpp、expandBlurDamage）。着色器以 `.frag/.vert` 存放并在构建期由 scripts/generateShaderIncludes.sh 转成可 `#include` 头文件（CMakeLists.txt:26-34）。

### 5.3 桌面对象与窗口：多继承的 CWindow

窗口类 `CWindow : public virtual IView, CGeometricMovableAnimated, IAlphaModifiable, CPopupOwner, CSubsurfaceOwner`（src/desktop/view/window/Window.hpp:132），但具体能力被拆成十余个控制器组合：WindowFullscreenPolicy、WindowGroupMembership（tab 组）、WindowSwallowController、WindowEffectsController、WindowStateTracker、WindowPresentation 等（同目录）。桌面层规则系统在 src/desktop/rule/，窗口/图层/监视器规则统一匹配执行。这种"门面类 + 专项控制器"的结构是近期重构产物，替代了旧的单体 CWindow。

### 5.4 布局引擎：算法插件化

布局被重构为 `Layout::ITiledAlgorithm` / `IFloatingAlgorithm` 接口（src/layout/algorithm/TiledAlgorithm.hpp），内置 tiled/{dwindle,master,monocle,scrolling} 与 floating/default。Dwindle 是经典二叉树：`SDwindleNodeData` 持有 parent/children 弱引用、splitRatio 与 preserve_split 标记，`recalcSizePosRecursive` 递归重算（src/layout/algorithm/tiled/dwindle/DwindleAlgorithm.hpp:10-31）。插件可通过 `addTiledAlgo/addFloatingAlgo` 注册新算法（src/plugins/PluginAPI.hpp:180-182），Lua 也可（config/lua/bindings/layout/）。

### 5.5 插件与 IPC：dlopen + 函数 hook，双 socket

插件是运行时 dlopen 的共享库（CPluginSystem，src/plugins/PluginSystem.hpp:38-48），API 全为 `extern "C"`（src/plugins/PluginAPI.hpp:60），最激进的是 `createFunctionHook`——用 udis86 反汇编对任意成员函数做二进制内联 hook（PluginAPI.hpp:205）。为此构建必须 `-fno-lto` 并导出符号（CMakeLists.txt:123,125）。IPC 双 socket：s1 是 hyprctl 的请求/响应 socket，命令以 `socket.registerCommand(...)` 注册，含 80+ 内置命令（workspaces/clients/dispatch/keyword/…，src/ipc/s1/Commands.cpp:2078-2110+，文件 2157 行）；s2 是事件广播 socket（src/ipc/s2/）。插件还能注册自己的 hyprctl 命令（PluginSystem.hpp 中 `m_registeredHyprctlCommands`）。

### 5.6 协议层：78 个自写服务端协议

src/protocols/ 每协议一对 .cpp/.hpp（如 FocusGrab、ColorManagement、DRMSyncobj、ExtWorkspace、BackgroundEffect），XML 在顶层 protocols/ 与系统 wayland-protocols/hyprland-protocols 中，经 hyprwayland-scanner 生成类型安全的 C++ 包装（CMakeLists.txt:495-618 注册约 60 个，含 ext-/staging 最新协议如 ext-background-effect-v1、commit-timing、fifo）。wlroots 的私有协议（screencopy、gamma-control、foreign-toplevel 等）也被兼容实现，保证截图/热键等第三方工具生态可用。

## 6. 构建与运行

```bash
# 标准 CMake 构建（Release，需先装好 aquamarine/hyprutils/hyprcursor/hyprgraphics 等 pkg-config 包）
cmake --generate / 或直接：
make cmake            # Makefile 封装：自动 cmake -S . -B ./build（Makefile:1-25）
make release          # Release 构建
make install          # 安装，含 hyprland/hyprctl/hyprpm、assets、man 页、session 文件
make installheaders   # 安装插件开发头文件（CMakeLists.txt:470）

# 测试（Debug 构建自动开启 BUILD_TESTING，CMakeLists.txt:87-95）
make gtest            # GTest 单元测试（tests/，gcov 覆盖率，CMakeLists.txt:707-738）
make hyprtest         # hyprtester 集成测试（hyprtester/）
make format-check     # clang-format 检查
cmake --build ./build --target run-hyprtester   # 构建目标（CMakeLists.txt:740-750）

# 运行
Hyprland -c /path/to/hyprland.lua   # 配置入口已是 Lua；--verify-config 只校验不运行（src/main.cpp:27-41）
```

发行版打包普遍走 Nix flake（flake.nix）或各发行版源码包；构建期会执行 shader 生成脚本与 LuaLS stub 生成（meta/generateLuaStubs.py，CMakeLists.txt:473-478），需要 Python3。

## 7. 测试与工程质量

- 单元测试：tests/ 下 13 个子系统目录（config/desktop/protocols/render/managers/…），GoogleTest 框架，`gtest_discover_tests` + gcov 覆盖率插桩（CMakeLists.txt:714-737）；Debug 构建默认开启。
- 集成测试：hyprtester/ 是自研框架，含 headless clients、fixtures、插件加载测试（hyprtester/plugin/）、协议一致性测试（hyprtester/protocols/）、pointer-scroll.txt 之类的场景脚本与 test.lua（Lua 驱动）。
- CI：.github/workflows/ci.yaml（Arch 容器构建产物上传、clang-format 强制检查），nix-ci.yml/nix-test.yml（Nix 构建/测试），codeownersplus.yml（自动请求 review），man-update.yaml、translation-ai-check.yml 等。
- 代码质量观察：AGENTS.md 成文规定了审查重点（性能敏感路径回归、配置项不得静默破坏行为、新配置必须同步 wiki PR）与命名/格式规范；-Wall -Wextra -Wpedantic 全开（CMakeLists.txt:108-120）；ASan/gprof/Tracy 均有构建开关（CMakeLists.txt:318-351）。

## 8. 社区与维护状态

- 活跃度：极活跃。gh api 显示最近 push 2026-11-13（前一天）；release 节奏 2026 年内 v0.56.0(7-20)→v0.56.1(7-27)→v0.56.2(8-5)→v0.57.0(10-28)→v0.57.1(11-3)→v0.57.2(11-9)，月度小版本+补丁级滚动发布。
- 贡献者结构：约 784 名贡献者（gh api 分页），但项目由 Vaxerski（Vaxry）创立并长期主导架构决策（LICENSE 版权行、Compositor 单例全局对象 `g_pCompositor` 的中心化设计可见一斑）。客观记录：Vaxry 在社区有已知争议（如曾将 Freedesktop SDK 排除、社区管理风格引发的讨论），这些属社区舆论，未在本仓库代码内体现，"未确认"细节真伪；仓库内的治理信号是：issue/PR 采用 vouch（担保人白名单）机制过滤（.github/workflows/vouch-issues.yml、vouch-prs.yml，配合 ISSUE_TEMPLATE 要求模板填写），代码所有者用 codeowners.toml 自动请求 review 但不强制 approval。
- 文档：配置与用户文档在 wiki.hypr.land（README 链接），仓库内仅 man 页 docs/Hyprland.1、hyprctl.1 与 AGENTS.md 开发规范。
- 生态：hyprwm 组织下 hyprland-protocols/hyprutils/aquamarine/hyprpaper/hyprlock/hypridle 等配套项目形成桌面环境生态位（依赖关系见 CMakeLists.txt）。

## 9. 亮点与风险

亮点：
1. 真正独立实现合成器核心（aquamarine 后端 + 自写 78 个协议 + 自研 scanner），摆脱 wlroots 依赖，掌握全部底层（README.md:40、CMakeLists.txt:152-160、src/protocols/）。
2. 视觉与功能并存：pass 元素渲染管线把 blur/边框/阴影/发光做成可组合管线且带 damage 跟踪（src/render/pass/、src/render/blur/）。
3. 可扩展性三层递进：Lua 脚本/配置 → dlopen 插件（含二进制函数 hook）→ 完整 IPC（hyprctl 命令 + 事件 socket）（src/plugins/、src/ipc/）。
4. 工程纪律好：C++26 前沿特性有节制地使用、AGENTS.md 成文规范、单元+集成双层测试、clang-format 强制 CI。
5. 发布与迭代速度在小体量核心团队下罕见：月度版本、补丁数天内跟进（gh api releases）。

风险/局限：
1. 配置系统正在经历 .conf→Lua 的破坏性迁移（ConfigManager.hpp 仅剩 CONFIG_LUA），老配置需重写，周边教程/示例短期混乱（未确认旧版长期维护承诺）。
2. C++26 + 最新协议跟进激进，对发行版打包者门槛高：要求 cmake≥3.30、GCC 足够新、10+ pkg-config 依赖（CMakeLists.txt:1,271-292），很多发行版跟不上导致用户流向第三方构建。
3. 单体全局单例风格（g_pCompositor 等大量全局状态）+ 插件可 hook 任意函数，稳定性依赖作者纪律，大重构（desktop/view、layout）期间第三方插件频繁失效（hyprpm 每版本需重编插件）。
4. 文档重心在 wiki 而仓库内开发者文档有限；部分治理决策（vouch 白名单）对外部贡献者不够透明（.github/workflows/vouch-*.yml）。

## 10. 总结评价

Hyprland 是过去四年 Wayland 桌面最成功的"后起者"：从 2022 年的玩具项目成长为 40k 星、784 贡献者、月度发版的主流合成器，靠的是"把 Sway 的平铺严肃性和 eyecandy 的观赏性一次性给全"的产品直觉，以及敢于整体替换 wlroots、整体重写配置系统的工程魄力。它最适合：想要高颜值动态平铺环境的日常 Linux 桌面用户；想研究现代合成器如何在没有 wlroots 帮助下实现 DRM/KMS、输入、协议栈的 C++ 开发者（src/managers/eventLoop/、src/protocols/ 是绝佳教材）；以及愿意追新版本、接受破坏性变更的折腾型用户。不建议：需要 LTS 式稳定 API 的嵌入式/车载场景，或希望只读代码不追 master 重构节奏的贡献者。
