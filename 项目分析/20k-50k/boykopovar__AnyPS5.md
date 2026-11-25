# boykopovar/AnyPS5 项目分析

> 星数 51397（快照 2026-10-24 为 51107，分析日实测 51397）｜主语言 C++｜档位 20k-50k｜分析日期 2026-11-06

## 1. 项目速览

一句话定位：把 PS5 游戏可执行文件"重链接"（relink）为 Linux/Windows 原生程序的转换工具——不做 CPU 模拟，转换后的程序作为普通进程直接运行在重实现的 PS5 系统库之上。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | boykopovar/AnyPS5 | GitHub API |
| 星数 / Fork / Watch | 51397 / 4891 / 403（2026-11-06 实测） | `gh api repos/boykopovar/AnyPS5` |
| 主语言 | C++（C++20），辅助 Python 工具链 | `CMakeLists.txt`、`tools/` |
| 许可证 | GPL-2.0-only | `LICENSE` |
| 创建时间 | 2026-08-03 | GitHub API `created_at` |
| 最近推送 | 2026-11-05（分析前一日） | GitHub API `pushed_at`、`gh api commits` |
| 默认分支 / Release | main / v0.1.4（2026-10-30），v0.1.3、v0.1.2 均在 10 月内 | GitHub API releases |
| 代码规模 | `core/` 约 118 万行 C/C++（不含 `3rdparty/`） | 本地统计 |

## 2. 项目是什么，解决什么问题

AnyPS5 官方自述为"Tool for automatic executables porting to Linux and Windows"（`README.md`）。其核心机制与主流主机模拟器不同：它不是模拟器，而是一个重链接器。PS5 游戏本体是 x86-64 ELF（PS5 CPU 为 Zen 2 架构），AnyPS5 把这些可执行文件转换为目标系统的原生格式——Linux ELF、Windows PE，或（relinker-only 构建）macOS x86-64 Mach-O——并依赖操作系统的动态链接器按 NID（导出函数哈希名）把导入绑定到项目重实现的系统库上（`README.md`、`docs/dev/ARCHITECTURE.md`）。转换后的程序没有任何 CPU 指令翻译运行时开销，也没有独立的模拟器进程；除少数 AMD 专有指令需要 `--to-intel` 改写外，游戏代码原生执行。

转换管线（`docs/dev/ARCHITECTURE.md` 总览图、`core/relinker/relinker/src/pipeline/RelinkerPipeline.cpp`）依次为：`--to-intel` 指令降级（可选）→ RelinkerPipeline（按 NID 读取导入、扫描 syscall 引用、过滤未使用 NID、构建 SysV 动态节）→ GuestModuleBuilder（转换游戏自带的 `sce_module/` 模块）→ LinuxElfPatcher / WindowsPePatcher 写出目标格式。NID 即 PS5 系统库导出符号的哈希名：`core/libs/nid/src/NidCompute.cpp` 用自实现 SHA-1（`core/libs/nid/src/Sha1.cpp`）从"符号名+库名"计算 NID，构建后由 `nid_patcher` 把每个导出重命名为对应 NID，使 OS 加载器能直接绑定。

系统库侧是项目的另一大主体：`core/libs/prx/` 下 406 个 PS5 系统库的重实现（libkernel、libc、libSceAgcDriver、libSceAudioOut 等，均编译为共享库），每个库有明确的未实现约定——导出函数要么按通用语义实现，要么抛异常（`NotImplemented_nid_no_patch`），绝不返回伪造的成功值（`CONTRIBUTING.md`、`AGENTS.md`）。

图形侧没有走"驱动级翻译"捷径：PS5 GPU 基于 AMD RDNA2，游戏通过 GNM 风格的 AGC API 提交命令缓冲。项目在 `core/libs/prx/libSceAgcDriver/`（约 37 万行，本地统计）中用 Vulkan 完整实现了 AGC 驱动层：PM4 包解析、绘制管线管理、管线日志重放（PipelineJournal）、着色器磁盘缓存等（`docs/dev/ARCHITECTURE.md` Graphics 一节）；同时 `core/shader/recompiler/` 把 RDNA2 着色器 ISA 重编译为 SPIR-V（RDNA 译码 → 控制流图与结构化 → IR 翻译 → 优化 → SPIR-V 后端，`core/shader/recompiler/Recompiler.cpp`）。

主机游戏 CPU 侧依赖 Zen 2 扩展（AVX2、FMA、F16C、BMI1/2、LZCNT、MOVBE 等），转换后的程序启动时会检测宿主 CPU 扩展并在缺失时中止（`docs/user/USAGE.md`）。此外项目提供纯前端 Web 转换器（`demo/web/`，WASM 后端 `core/relinker/wasm/`），文件不离开本机（`README.md`）。

## 3. 技术栈与依赖

- **语言**：C++20（relinker 仅用标准库，`CONTRIBUTING.md`）；Python ≥3.9 用于回归测试与约 60 个开发工具脚本（`tools/`）；Web 前端为原生 ESM JavaScript（`demo/web/app.mjs`、`worker.mjs`）。
- **构建**：CMake ≥3.22.1 + Ninja；Windows 用 MinGW-w64 GCC（DWARF/SEH 异常均有校验，`docs/dev/BUILD.md`）；relinker-only 模式不需要任何第三方依赖。
- **关键第三方库**（27 个 submodule，`.gitmodules`）：SDL2（窗口/输入/音频）、Vulkan-Headers、SPIRV-Tools/glslang（SPIR-V 校验与优化）、VulkanMemoryAllocator、ffmpeg-core、freetype、mimalloc（可选分配器）、tracy（性能剖析）、quickjs-ng、mbedtls、zstd、ZArchive、vgmstream、opus/ogg/vorbis、LibAtrac9（取自 shadps4-emu 组织）、libusb、nghttp2 等。其中 `3rdparty/LibAtrac9`（shadps4-emu）与 `3rdparty/ffmpeg-core`（KytyPS5）来自 PS4/PS5 模拟器生态上游。
- **测试框架**：CTest 注册的各语言测试（含 Python 测试）；libFuzzer 用于 relinker 转换 fuzz（`docs/dev/BUILD.md` Fuzzing 一节）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `core/relinker/` | 重链接器本体（约 4.3 万行）：`pipeline/` 转换管线、`analysis/`（SyscallScanner、UnusedNidFilter、CallSiteResolver、RelativeJumpTables）、`output/`（SysV 动态节构建器）、`patching/`（GamePatch，TOML 子集补丁）、`elfpatcher/`（linux/macos/windows 出格式写入器）、`codegen/`（AMD 专有指令→x86-64 降级）、`wasm/`（Emscripten 绑定）、`fuzz/` |
| `core/libs/` | `prx/` 406 个 PS5 系统库重实现（libkernel 约 3.9 万行、libSceAgcDriver 约 37 万行）；`nid/` NID 补丁器（ELF/PE/Mach-O 三格式）；`allocator/` mimalloc 接入；`tests/` 1113 个测试文件 |
| `core/shader/recompiler/` | RDNA2→SPIR-V 着色器重编译器（约 8.5 万行）：RdnaDecoder、ControlFlow、Translation、Optimization、SpirvBackend、磁盘缓存 |
| `core/Decoder/` | Jpeg/Png 小型解码器（8 个文件） |
| `tools/` | 约 60 个 Python 脚本：约定检查、fuzz 种子、性能对比、SELF 解包（`unwrap_self.py`）、release 打包、进度统计等 |
| `docs/` | `user/`（USAGE、INPUT_MAPPING、7 种语言 README）、`dev/`（ARCHITECTURE、BUILD、CONVENTIONS、REVERSE_ENGINEERING、HW_ORACLE、TRACING、TechnicalDebt）与 `dev/debt/<library>.md` 逐库技术债文件 |
| `demo/web/` | Web 转换器前端（21 个文件） |
| `3rdparty/` | 27 个 submodule，全部源码构建（默认不 `find_package`，`cmake/SystemDeps.cmake` 提供系统依赖可选开关） |

组织方式评述：单 monorepo、单一 CMake 树（顶层 `CMakeLists.txt` 约 4.7 万字节）。406 个 prx 库统一以 `EXCLUDE_FROM_ALL` 构建、经自定义 `libs` 目标显式产出（`docs/dev/BUILD.md` Commands 一节），构建产物按 NID 相互咬合。工程以极强的书面约定治理（见第 7 节），`AGENTS.md` 直接面向 AI 编码代理写作。

## 5. 核心架构与关键实现

### 5.1 重链接管线（core/relinker/relinker/src/pipeline/RelinkerPipeline.cpp）

入口 `RelinkerPipeline::Relink()` 先校验输入 ELF 类型：仅接受 `ET_DYN` 及 PS5 专有的 `0xfe10`/`0xfe18` 类型，`ET_EXEC`、core dump、可重定位文件均直接抛异常；随后收集可执行 `PT_LOAD` 段与 `PT_SCE_DYNLIBDATA`（PS5 动态库数据段）。关键代码（65-110 行节选）：

```cpp
RelinkResult RelinkerPipeline::Relink() {
    const auto inputType = elfReader->ReadHeader().Type;
    if (inputType == kEtCore)
        throw RelinkerException("ET_CORE core dumps are not executable images");
    if (inputType != ET_DYN && inputType != kEtSceDynExec && inputType != kEtSceDynamic) {
        std::ostringstream message;
        message << "Unsupported ELF object type 0x" << std::hex << inputType
                << ": expected ET_DYN, 0xfe10 or 0xfe18";
        throw RelinkerException(message.str(), 0x10);
    }
    auto programHeaders = elfReader->ReadProgramHeaders();
    ...
}
```

输出侧（`core/relinker/relinker/src/output/SysVDynamicSectionBuilder.cpp`）构造标准 SysV 动态节，使 Linux 的 ld.so / Windows PE 加载器无需任何定制即可按 NID 解析导入。游戏自带模块优先于宿主库参与符号查找（`ARCHITECTURE.md` 概览与 `GuestModuleBuilder.cpp`）。

### 5.2 AMD 专有指令降级（core/relinker/codegen/）

PS5 游戏可能使用非 x86-64 基线的 AMD 专有指令（如 ANDN、位域指令等 BMI/TBM 类）。`codegen/src/Amd64OnlyConverter.cpp` 配合 `x86/Amd64OnlySubstitutionTable.hpp`、`AndnLowering.hpp`、`BitfieldLowering.hpp` 等把这类指令改写为等价 x86-64 序列，报告区分"in place"（原位改写）与"stub"（外跳板）两种 lowering（`pipeline/Conversion.cpp` 73-85 行）。该步骤是可选的（`--to-intel`），且无法处理游戏运行时自生成的 AMD 专有指令（`docs/user/USAGE.md` 明示此局限）。

### 5.3 着色器重编译器（core/shader/recompiler/Recompiler.cpp，2285 行）

`ShaderRecompiler::Recompile` 按固定阶段执行：`RdnaDecoder`（10 个译码器文件：标量/向量/内存/图像/导出操作分类译码，`RdnaDecoder/src/`）→ `ControlFlow`（GraphBuilder + Structurizer）→ `Translation`（RDNA→IR）→ `Optimization`（BindingAllocator、ConstantFolder、DeadCodeEliminator、ResourceTracker 等 10+ 个 pass，见 Recompiler.cpp 33-45 行 include 表）→ `SpirvBackend` 输出 SPIR-V；`ANYPS5_ENABLE_SPIRV_TOOLS` 时经 SPIRV-Tools 校验（`README.md`）。编译变体按内容哈希入磁盘缓存（`ShaderDiskCache.cpp`），格式版本严格拒绝旧格式条目（`ARCHITECTURE.md`）。

### 5.4 系统库重实现与 NID 绑定（core/libs/prx/、core/libs/nid/）

每个 prx 库编译为共享库，构建后 `nid_patcher`（`core/libs/nid/src/ElfNidPatcher.cpp`、`PeNidPatcher`、`MachONidPatcher`）把导出名改写为 NID。`APS5_EXPORT("<nid>", func)` 允许直接声明未知名字的 NID（`ARCHITECTURE.md`）。线程语义以 FreeBSD libthr 固定 commit 为参考契约（`ARCHITECTURE.md` Thread cancellation 一节，引用 freebsd-src@5ed7eb0d）；未实现导出统一调 `NotImplemented_nid_no_patch(__func__)`（如 `core/libs/prx/libSceAudioIn/Export.cpp`），缺失导入在启动时明确报错而非静默回退（`CONTRIBUTING.md`）。

## 6. 构建与运行

以下命令均出自 `docs/dev/BUILD.md` 与 `docs/user/USAGE.md`：

```sh
# Relinker-only（无需 submodule/Vulkan，Linux/Windows/macOS 通用）
cmake -S . -B build-relinker -G Ninja -DCMAKE_BUILD_TYPE=Release -DANYPS5_RELINKER_ONLY=ON -DBUILD_TESTING=ON
cmake --build build-relinker --parallel
ctest --test-dir build-relinker --output-on-failure

# 完整构建（需先 git submodule update --init --recursive）
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build --parallel
cmake --build build --target libs --parallel   # 406 个 prx 库需显式构建

# Fuzz（libFuzzer，Clang + ASan/UBSan）
cmake -S . -B build-fuzz -G Ninja -DANYPS5_RELINKER_ONLY=ON -DANYPS5_BUILD_FUZZERS=ON ...
build-fuzz/tests/relinker_conversion_fuzzer build-fuzz/fuzz-corpus -fork=2 -max_total_time=1200

# 转换示例
relinker source/input.elf app.elf            # Linux
relinker --windows source/input.elf app.exe  # Windows
relinker --macos --to-rosetta source/input.elf eboot  # macOS Rosetta
```

运行前需将转换出的可执行文件与游戏 `sce_module/` 模块、`libs/*.prx` 按布局摆放；`--dry-run` 可全内存试跑并列出全部计划输出（`docs/user/USAGE.md`）。

## 7. 测试与工程质量

- **测试规模**：CMake 中 `add_test`/`gtest_discover_tests` 注册点约 2798 个（本地 grep 统计），`core/libs/tests/` 1113 个文件；relinker 有独立 Python 回归测试与 20000 输入的 fuzz ctest（`docs/dev/BUILD.md`）。CI 检查"每个测试可执行文件都必须注册进 ctest"（`.github/workflows/sanitizers.yml` 82 行）。
- **CI**（`.github/workflows/`，31 个 workflow 文件）：build.yml 带资格门控，支持把最多 40 个 PR 合批构建、并只运行"构建输入被 PR 改动"的测试（`select` 输入）；sanitizers.yml 全量测试跑 ASan+UBSan；另有 windows-msys2、emscripten-libraries、input-mapper、progress/pages 等专项流水线。
- **代码风格治理**（`docs/dev/CONVENTIONS.md`、`CONTRIBUTING.md`）：代码内禁止注释（仅 `#endif` 与 namespace 结尾例外）；每个 `APS5_VABI` 导出必须在公共头中恰好声明一次（Python 测试 `test_public_headers.py` 强制）；着色器语义改动必须注明来源——硬件实测（`docs/dev/HW_ORACLE.md`）或 ISA/LLVM/ACO 精确出处，评审者在硬件上复核；单测试上限 30 秒。
- **观察**：这是一套罕见地"面向机器可验证"的工程纪律，配套 `AGENTS.md`（明确面向 AI 编码代理的行为规则）与 PR 模板中的 `AI-assisted` 字段，构成大规模代理辅助贡献的治理框架。

## 8. 社区与维护状态

- **活跃度**：创建于 2026-08-03，分析时最近提交为 2026-11-05（`gh api commits`，当日多条 merge）。3 个月内 PR 编号已达 #16060， contributors API 分页显示约 410 位贡献者（`gh api contributors?per_page=1` 末页 410），open issue+PR 共 2127。最近提交记录显示 main 分支持续合并大量来自不同作者的小型分支（feat/fix 前缀，Conventional Commits）。
- **Release 节奏**：v0.1.2（10-14）、v0.1.3（10-20）、v0.1.4（10-30），约每周至两周一个 patch；Release 同时产出 Web 转换器与 Linux/Windows 库包（`docs/dev/BUILD.md` Web converter releases 一节）。
- **进度可视化**：GitHub Pages 徽章与进度地图（`README.md` Status 区）、每游戏一个 bring-up 讨论串（`.github/DISCUSSION_TEMPLATE/game-bring-up.yml`）。
- **增长客观事实**：0→51k 星仅用约 3 个月，日均数百星；同期 PR 流量与贡献者数量同量级放大。增长节奏与 3 个月内 1.6 万个 PR 的吞吐在数据上自洽；具体流量构成（自然传播/媒体曝光等）无公开数据，未确认。
- **维护人结构**：commit 记录与 PR 合并均可见单人仓库 + 大量外部贡献者的形态；核心维护者 boykopovar 的合并量占比未确认。

## 9. 亮点与风险

**亮点**
1. 技术路线独特：以"重链接 + OS 原生加载器"替代 CPU 模拟，转换产物是无运行时的原生程序，CPU 侧零模拟开销（`docs/dev/ARCHITECTURE.md`、`README.md`）。
2. 三个月写出 118 万行 C++ 且带 2700+ 注册测试、ASan/UBSan 全覆盖、libFuzzer 持续 fuzz——规模与工程纪律的组合极为罕见（`docs/dev/BUILD.md`、CI 配置）。
3. 语义严谨：未实现即抛异常、禁止按单一游戏特判、逐库技术债文件（`docs/dev/debt/`）、着色器语义强制溯源硬件或 ISA 文档（`CONTRIBUTING.md`）。
4. 着色器重编译器与 Vulkan AGC 驱动层（37 万行）是独立可读的大型系统软件资产：RDNA2 译码、结构化控制流、SPIR-V 后端、管线日志重放俱全。
5. 文档深度超出同规模项目常态：231 行架构文档信息密度极高，7 语言 README、明确的反向工程方法论（`docs/dev/REVERSE_ENGINEERING.md`）。

**风险 / 局限**
1. 平台与 CPU 依赖：产物固定基址加载（Linux `0x400000`，无 ASLR）、无控制台沙箱，转换出的游戏以用户全权限运行；宿主 CPU 必须带 Zen 2 全套扩展（`docs/user/USAGE.md` 安全提示节）。
2. 兼容性非目标级承诺：系统库覆盖度以"项目已声明函数"为分母（`README.md` 脚注），未实现导出会抛异常中止；逐游戏兼容状态依赖社区 bring-up 报告。
3. 法律边界（客观陈述）：项目不附带、不要求任何版权软件、固件或密钥，自述用途为互操作性/研究/保存（`README.md` Disclaimer）；但其工作对象是受版权与反规避法保护的商业主机软件，在欧盟 2001/29/EC、美国 DMCA 1201 条等不同法域下合法性结论不同，使用者需自行确认当地法律；"PlayStation/PS5"为 SIE 商标，项目与索尼无关联（同上）。
4. 治理模式可复制性未知：`AGENTS.md` 驱动的代理辅助贡献在 3 个月内撑起了当前吞吐，但单维护者合并 1.6 万 PR 的可持续性、以及 AI 辅助代码的长期维护成本，均无历史数据可验证。

## 10. 总结评价

AnyPS5 用一条与主流模拟器截然不同的路线切入 PS5 兼容性问题：不模拟 CPU，而是把游戏可执行文件重链接成原生程序，让重实现的系统库以普通共享库身份被 OS 加载器绑定；GPU 侧则以 37 万行 Vulkan 代码完整接住 AGC 命令流，配合 RDNA2→SPIR-V 重编译器跑通着色器。它在 2026 年 8 月至 11 月的三个月里从零冲到 51k 星、1.6 万 PR、约 410 位贡献者，背后是一套为大规模（含 AI 代理）协作设计的极端严谨的工程规范——禁注释、禁伪造成功、逐库技术债、硬件溯源测试、2798 个注册测试与全量 sanitizer CI。对系统软件工程师，它是 ELF/PE 双格式链接器内幕、NID 哈希绑定、RDNA2 ISA 与 Vulkan 管线管理的活教材；对工程管理者，它是"如何用书面规范 + 机器可验证门禁治理海量外部贡献"的样本；对普通用户，它属于需要自备游戏文件、自担法律与安全责任的开发者工具，而非开箱即用的模拟器。
