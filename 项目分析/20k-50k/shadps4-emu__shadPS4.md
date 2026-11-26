# shadps4-emu/shadPS4 项目分析

> 星数 20000（快照 2026-10-24）｜主语言 C++｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：shadPS4 是用 C++ 编写的早期 PlayStation 4 模拟器，支持 Windows / Linux / macOS（Apple Silicon），已能运行《血源诅咒》《黑暗之魂 重制版》《荒野大镖客》等 3A 大作。

| 项目 | 事实 |
|---|---|
| 名称 | shadps4-emu/shadPS4（`shadps4.net`） |
| 星数 | 快照 20000；gh api 核实 2026-11-21 为 34,322，增长很快 |
| 语言 | C++23（CMake ≥3.24，src/ 下自有代码 952 个 .cpp/.h、约 9.9 万行） |
| 许可证 | GPL-2.0-or-later（REUSE.toml 做 SPDX 逐文件标注） |
| 创建时间 | 2022-10-24（gh api） |
| 最近活跃 | 极活跃：最近 push 2026-11-20（分析前 1 天），open issues 337，forks 2,915 |
| 贡献者 | 约 246 人（gh api contributors 分页统计），核心团队 8 人（README "Main team"） |
| 发布节奏 | 版本化 release 约每月一个（v0.12.5 → v0.21.0，共 26 个），另有按日期命名的每日 pre-release |
| 形态 | 本仓库为"模拟器核心 + CLI"，Qt 图形界面已拆分到独立仓库 shadps4-qtlauncher |

## 2. 项目是什么，解决什么问题

shadPS4 在 PC 上模拟 PS4 主机，让用户以本机性能运行 PS4 游戏，同时是 PS4 系统软件（kernel / GNM 图形驱动 / SDK 库）的逆向研究成果的工程化实现。README 明确"项目为乐趣而开始"，走小步快跑的持续更新路线（`README.md` "Why" 节）。

与技术上最接近的对照是 RPCS3（PS3 模拟器）：PS3 的 Cell PPU 是 PowerPC 架构，RPCS3 需要 LLVM 重编译器做指令翻译；而 **PS4 游戏本身就是 x86-64 代码**，因此 shadPS4 选择了"原生执行 + 运行时补丁"的路线——不写 CPU JIT，把 guest 代码加载进模拟的地址空间后直接在本机 CPU 上跑，仅对少数本机不支持/语义不同的指令做即时打补丁（见第 5 节）。这是该项目能在两年多内达到可玩 3A 大作的关键架构决策（`src/core/cpu_patches.cpp`、`src/core/linker.cpp`）。

README 还体现了项目的生态位管理：兼容性追踪放在独立组织仓库 `shadps4-compatibility/shadps4-game-compatibility`，最终用户 GUI 放在独立仓库 `shadps4-qtlauncher`，本仓库专注核心。项目明确致谢 yuzu（shader 编译器以其 Hades 编译器为蓝本）、Panda3DS、fpPS4（PS4 系统库逆向协助）等（`README.md` "Special Thanks"）。

## 3. 技术栈与依赖

- 语言与构建：C++23、CMake（FetchContent 拉取依赖，`CMakeLists.txt`）；支持 Windows/Linux/macOS 三平台，Apple 平台仅 Apple Silicon（`CMakeLists.txt` 中 `CMAKE_OSX_DEPLOYMENT_TARGET 26.0`），另提供 Docker 与 Nix（`flake.nix`）构建。
- 图形：Vulkan（唯一后端）；macOS 上通过 `externals/mesa-kosmickrisp`（Mesa 的 Vulkan-on-Metal 驱动）获得 Vulkan 能力——这是要求 macOS 26.0 的原因。
- 窗口与输入/音频：SDL3（`externals/sdl3`；窗口封装 `src/sdl_window.cpp`，音频输出/输入 HLE 用 SDL 音频流 `src/core/libraries/audio/sdl_audio_out.cpp`、`sdl_audio_in.cpp`，audio3d 走 OpenAL）。
- shader 编译链：glslang + sirit（SPIR-V 构建器，继承自 yuzu 生态，`externals/sirit`）；代码补丁用 Zydis（x86 反汇编）+ Xbyak（x86 汇编 JIT，`externals` 内）。
- 基础库：fmt、spdlog、magic_enum、ext-boost、zstd、cmrc（资源内嵌）、imgui（调试/大屏界面）、RenderDoc（帧捕获，`src/video_core/renderdoc.cpp`）、cpp-httplib、protobuf（`src/shadnet/shadnet.proto`）、libressl、ffmpeg-core 等（`externals/` 全景约 45 个三方组件）。
- 许可合规：全仓库 REUSE.toml + SPDX 文件头声明版权与许可证（`REUSE.toml`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `src/common/` | 跨平台基础层：arch 定义、日志、断言、内存映射原语、路径工具、RDTSC 时钟、Discord RPC |
| `src/core/` | 模拟器核心：内存/地址空间、linker、CPU 补丁、信号分发、调试器、IPC、devtools、`libraries/`（55 个 HLE 库模块） |
| `src/core/libraries/kernel/` | PS4 内核 HLE：线程/pthread、互斥锁、事件队列 equeue、文件系统、内存管理、AIO、时间 |
| `src/core/libraries/videoout/` | 视频输出 HLE（显示端口、buffer、flip 事件） |
| `src/core/libraries/gnmdriver/` | GNM 图形驱动 HLE：收集 guest 提交的 PM4 命令缓冲并投递给 Liverpool |
| `src/core/libraries/np/` 等 | 网络/PSN 相关 HLE：trophy、matching2、web api、np_auth 等 |
| `src/video_core/` | GPU 模拟：`amdgpu/`（Liverpool 命令处理器 + GCN 寄存器）、`renderer_vulkan/`、`buffer_cache/`、`texture_cache/`、`host_shaders/` |
| `src/shader_recompiler/` | GCN 着色器 → IR → SPIR-V 编译器（frontend/ir/backend） |
| `src/shader_recompiler` 同级的 `src/input/`、`src/imgui/` | SDL 输入抽象（含鼠标转摇杆）、ImGui 调试层与大屏 UI、shadnet 好友/通知层 |
| `src/shadnet/` | 自研网络层客户端（protobuf 协议，配合 np/ HLE 提供联机相关能力雏形） |
| `scripts/` | 符号数据库：`aerolib.inl`（17 万行 NID→符号名桩表，由 `ps4_names2stubs.py` 生成） |
| `externals/` | vendored 三方库（SDL3、glslang、sirit、xbyak、mesa-kosmickrisp 等） |
| `tests/` | GTest 单元测试（gcn 指令翻译、网络栈、设置、ngs2） |
| `documents/` | 各平台构建文档、changelog、调试指南、截图 |

组织方式：单仓单体（CMake 单树构建），模块按"common / core(HLE) / video_core / shader_recompiler / 前端外壳"分层，边界清晰；GUI 拆出为独立仓库。核心代码风格与 yuzu/citron 一脉（命名、IR、缓存分层高度相似）。

## 5. 核心架构与关键实现

### 5.1 CPU：无 JIT 的"原生执行 + 运行时补丁"（最核心的设计）

PS4 游戏是 x86-64 ELF，shadPS4 将其直接映射到预留的 guest 地址空间原生执行（`src/core/address_space.cpp`、`src/core/memory.cpp`），CPU 补丁层（`src/core/cpu_patches.cpp`，约 3000 行）用 Zydis 扫描代码段，对本机语义不一致的指令用 Xbyak 现场生成替代代码并改写：

- TLS 访问（guest 用 `fs`/`gs` 段访问 TCB，`GenerateTcbAccess`）；
- `RDTSC` 缩放（`PatchGuestRdtsc`，把 guest 计数按主机频率归一化，配合 `src/common/rdtsc.h`）；
- 近似精度指令 `RSQRTPS/RCPPS/EXTRQ/INSERTQ`、`MOVNT*` 非临时写等 SSE 特例（`GenerateRSQRTPS`、`ReplaceMOVNT` 等）。

无法翻译的访存违规/非法指令交给信号处理框架（`src/core/signals.h` 的 `SignalDispatch`，Windows 下走 SEH，POSIX 下走 sigaction），配合页面管理器实现惰性内存提交（`src/video_core/page_manager.cpp` 监控 GPU 脏页）。Linux 上可选 userfaultfd（`src/main.cpp` CLI 选项）。macOS 路线：不提供 arm64 guest 翻译，而是让 x86-64 版 shadPS4 整体跑在 Rosetta 2 下——`src/main.cpp` 用 `sysctlbyname("sysctl.proc_translated")` 拒绝真 Intel Mac，提示"仅支持 Apple Silicon"。这种"借宿主编译器/宿主 OS 的翻译能力"是罕见的务实取舍。

### 5.2 GPU：Liverpool 命令处理器 + Vulkan 光栅化

PS4 的 APU 代号 Liverpool，`src/video_core/amdgpu/liverpool.h/.cpp` 完整复刻其命令流：GNM 驱动 HLE（`src/core/libraries/gnmdriver/gnmdriver.cpp`）把 guest 提交的 dcb/ccb/acb 缓冲投给 Liverpool，后者在专用 GPU 线程里以协程（`Task`）逐条解析 PM4 命令（`pm4_cmds.h`/`pm4_opcodes.h` 定义指令集），驱动 `renderer_vulkan/vk_rasterizer.cpp` 画图。寄存器组按 GCN 真实布局建模（`regs*.h` 拆成 color/depth/primitive/shader/texture/vertex 六个头文件），tile 格式转换在 `amdgpu/tiling.cpp`。

着色器路径：前端解码 GCN 微码生成 CFG 并做结构化（`shader_recompiler/frontend/control_flow_graph.cpp`、`structured_control_flow.cpp`），中端为 SSA IR（`ir/`），后端发 SPIR-V（`backend/spirv/`）。工程还包括管线序列化（`renderer_vulkan/vk_pipeline_serialization.cpp`，持久化 pipeline cache）与 `host_shaders/` 内嵌的全屏 pass（FMV/缩放等）。

### 5.3 HLE 系统调用/SDK 库层：NID 符号解析

PS4 动态库符号用 NID（库 + 版本 + 模块 + hash）标识。linker（`src/core/linker.cpp` `Relocate/Resolve`）装载 eboot.bin 与 `.sprx`/伪模块，把 guest PLT 导入解析到 HLE 桩；桩注册用统一宏（`src/core/libraries/libs.h`）：

```cpp
#define LIB_FUNCTION(nid, lib, libversion, mod, function)                                          \
    LinkSymbolImpl(sym, nid, lib, libversion, mod, reinterpret_cast<u64>(HOST_CALL(function)),     \
                   Core::Loader::SymbolType::Function)
```

`src/core/libraries/` 下 55 个模块按 PS4 SDK 库名分目录实现（videoout、audio、pad、save_data、np、network、kernel 等），每个导出函数都标 `PS4_SYSV_ABI` 并走同一套错误码头（`orbis_error.h`/`posix_error.h`）。以 `sceVideoOutAddFlipEvent`（`libraries/videoout/video_out.cpp`）为例：HLE 直接操作 `Kernel::Equeue` 把 flip 事件挂到 guest 事件队列，与 kernel HLE 深度联动。未实现符号用 `scripts/aerolib.inl` 的 17 万行 STUB 表兜底成空桩，保证加载不崩。固件边界（中性记录）：README 声明约 34 个固件 `.sprx` 模块（字体/音频解码等）需要用户从自购 PS4 dump 放入 `sys_modules`，模拟器本体不分发固件。

### 5.4 kernel / audio / input 外围

kernel HLE（`src/core/libraries/kernel/`）覆盖线程创建（含 guest 栈切换汇编 `threads/stack_asm.cpp`）、同步原语、equeue、文件系统挂载（`/app0`、`/hostapp`，`src/core/file_sys/`）。save_data 有独立子目录（`save_memory.cpp`、`save_instance.cpp`、`save_backup.cpp`，含对话框），对应存档管理。输入层 `src/input/controller.cpp` 支持手柄即插即用与按键自定义（README 控制映射表）；ImGui 层（`src/imgui/`）提供大屏 UI、通知、好友层与 `src/core/devtools/` 调试工具（含 GCN/PM4 检查器）。

## 6. 构建与运行

```sh
# 构建（三平台各有文档：documents/building-windows.md / building-linux.md / building-macos.md / building-docker.md）
cmake --preset windows-debug          # 或 linux-release / macos-arm 等（cmake/CMake*.json 预设）
cmake --build --preset windows-debug

# 运行（README 示例）
shadPS4 CUSA00001                      # 在游戏目录列表中查找 CUSA00001 并启动
shadPS4 --fullscreen true --config-clean CUSA00001
shadPS4 /path/to/game.elf              # 直接启动 PS4 ELF
shadPS4 CUSA00001 -- -flag1            # 透传参数给游戏 argv

# 测试（可选开关 -DENABLE_TESTS=ON，GTest）
ctest                                  # CI 中在 Windows/Linux/macOS 矩阵上运行
```

## 7. 测试与工程质量

- 测试规模：小而精。`tests/` 含 GTest 工程：`tests/gcn/`（GCN 着色器指令翻译器逐指令对照测试、谓词重建测试）、`tests/network/`（17 个文件：HTTP 栈、net kernel、P2P 编解码与 TCP 互操作）、`test_emulator_settings.cpp`、`test_ngs2.cpp`。图形/HLE 主路径仍靠游戏实测而非单测覆盖。
- CI（`.github/workflows/build.yml`，名"Build and Release"）：三平台构建矩阵 + 独立 ctest 测试 job（Windows/Linux/macOS 矩阵）+ 发布 job（自动以日期 + git hash 命名 pre-release，如 2026-11-20-bf08ce9）。
- 工程质量观察：REUSE/SPDX 逐文件许可标注规范；日志分级细致（几乎每个 HLE 入口都有参数 dump 日志）；错误码体系完整；externals 统一由根 CMakeLists 管理。存在 `AGENTS.md`（面向 AI 助手的贡献规范）。

## 8. 社区与维护状态

- 活跃度（gh api，2026-11-21）：34,322 星 / 2,915 fork / 337 open issues；最近 push 为分析前 1 天，pre-release 每日滚动发布，极活跃。
- 贡献者结构：约 246 名贡献者；核心 8 人团队（georgemoralis、psucien、viniciuslrangel、roamic、squidbus、baggins183、StevenMiller123、kalaposfos13，见 README），社区多维护者形态，另有 Discord 社区与 Ko-fi 赞助（`.github/FUNDING.yml`）。
- 发布节奏：版本号月度迭代（2025-11 v0.12.5 → 2026-10 v0.21.0，一年 9 个 minor），版本代号诙谐（"UltraPersona""RE6_PRIG"等，gh api releases）；`documents/changelog.md` 自 v0.4.0（2024-10-31）起持续记录。
- 兼容性追踪：独立组织仓库 `shadps4-compatibility/shadps4-game-compatibility` 汇总游戏可玩性评级（README 链接），与主仓解耦。
- 生态：与 Panda3DS/fpPS4/yuzu/felix86 等模拟器社区有明确的技术互助关系（README Special Thanks）。

## 9. 亮点与风险

亮点：
1. **CPU 无 JIT 原生执行路线**：利用"PS4 = x86-64"的架构巧合，把 CPU 模拟成本压缩为运行时补丁（Zydis+Xbyak），是 PS4 模拟器里性能与工程量的最优解（`src/core/cpu_patches.cpp`）。
2. **高保真 GPU 建模**：Liverpool/PM4/GCN 寄存器按真实硬件布局建模，shader recompiler 以 yuzu Hades 为蓝本，管线缓存可序列化（`src/video_core/amdgpu/`、`vk_pipeline_serialization.cpp`）。
3. **HLE 体系工程化程度高**：55 个 SDK 库模块、统一 NID 注册宏、17 万行符号桩数据库兜底、完整错误码体系（`src/core/libraries/`、`scripts/aerolib.inl`）。
4. **发布与社区运转成熟**：月度版本 + 每日 pre-release + 独立兼容性仓库 + 三平台 CI，2 年内从零到跑通 3A 大作。
5. **调试设施完善**：内置 ImGui devtools、RenderDoc 集成、GDB 式调试器与 IPC（`src/core/devtools/`、`src/core/debugger.cpp`）。

风险/局限：
1. **早期阶段**：官方自述"don't expect a flawless experience"，兼容性靠外部仓库逐游戏记录，大量 HLE 仍为空桩。
2. **macOS 依赖 Rosetta 与 macOS 26**：仅 Apple Silicon、需系统自带翻译层，稳定性受制于苹果政策（`src/main.cpp`）。
3. **固件版权边界**（中性事实）：部分游戏需要自购主机 dump 的固件 `.sprx` 模块，模拟器本体不提供，用户责任自负（README "Firmware files"）。
4. **GPL-2.0 与快速迭代**：核心代码与 yuzu 系生态同源较多，贡献合规与代码Review压力随每日 pre-release 增大（未确认具体流程细节）。

## 10. 总结评价

shadPS4 是当前 PS4 模拟化路线中工程完成度与社区活跃度最高的项目：它没有走"通用虚拟机/JIT"的笨重路线，而是围绕"PS4 是 x86-64 + AMD GCN"这一事实，用原生执行 + 补丁、Liverpool 命令流重放、GCN→SPIR-V 重编译三件套，把模拟器做成了"一个非常认真的系统级兼容层"。适合对逆向工程、GPU 架构、HLE 设计感兴趣的工程师读源码（`cpu_patches.cpp`、`liverpool.cpp`、`shader_recompiler/frontend/` 是三份最佳教材）；对普通玩家，则建议直接使用官方 QtLauncher 发行版并对照兼容性仓库选择游戏。
