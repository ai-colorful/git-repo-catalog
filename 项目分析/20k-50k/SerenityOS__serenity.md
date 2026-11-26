# SerenityOS/serenity 项目分析

> 星数 31000（快照 2026-10-24）｜主语言 C++｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：从零实现的现代 Unix 类图形操作系统——"90 年代 UI 美学 + 21 世纪工程实践"的自研全栈 hobby OS，内核、窗口系统、浏览器引擎、全部用户空间程序均在本仓库内从零写成。

| 项目 | 事实 |
| --- | --- |
| 名称 | SerenityOS/serenity |
| 星数 | 31000（快照）；gh api 实测 33983（2026-11-21） |
| 语言 | C++（CONTRIBUTING.md 明确采用 "idiomatic SerenityOS C++26"） |
| 许可证 | BSD-2-Clause（LICENSE，gh api 确认） |
| 支持架构 | x86_64 / aarch64 / riscv64（README.md、Kernel/Arch/ 三套实现） |
| 代码规模 | 约 18,800 个文件；.cpp 3,768 + .h 3,782，合计约 129.6 万行 C++（全仓统计） |
| 最近活跃 | push 于 2026-11-20（分析前一天），单日十余个 commit（gh api commits） |
| Release | 0 个 GitHub release：纯 master 滚动开发，无版本发布（gh api releases） |
| 贡献者 | 1,412 人（gh api contributors 分页统计） |
| 创建时间 | 2018-12-02，发起人 Andreas Kling（awesomekling，累计 16,364 个 commit） |

## 2. 项目是什么，解决什么问题

SerenityOS 是一个从第一行代码全部自研的图形化 Unix 类操作系统，目标是"一台为自己而造的梦之机"：界面致敬 90 年代生产力软件（Win95/OS/9 风格），底层则吸收现代 *nix 的能力（README.md "About" 一节自述 "a marriage between the aesthetic of late-1990s productivity software and the power-user accessibility of late-2000s *nix"）。它不解决任何商业问题，而是最大化"可黑进性（hackability）、可问责性（accountability）与乐趣"——这句原话出自 Documentation/FAQ.md："The SerenityOS project tries to maximize hackability, accountability, and fun(!) by implementing everything ourselves."

与 Linux 发行版的根本区别在于"零外部依赖"：README.md 明确 "...all of the above are right in this repository, no extra dependencies, built from-scratch by us"。内核是自写的 64 位抢占式多线程内核；C 库、窗口服务器、浏览器引擎（LibWeb/LibJS）、加解密/TLS、图形库、音视频编解码全部自研，不链接 OpenSSL/skia/ffmpeg/ICU 等（Ladybird/README.md 逐一列出了这些"替代了谁"）。第三方软件通过 367 个 Ports 源码编译引入（Ports/AvailablePorts.md 表格实测 367 行条目）。

它也是操作系统教学与浏览器引擎研究的活教材：所有子系统在同一仓库、同一风格、可互相追溯，配合 YouTube 开发视频（README 作者列表上方链接 Andreas Kling / Linus Groh 等人的频道），形成了"全栈可读"的独特价值。项目定位极其克制——Documentation/FAQ.md 直接写明 "There are no ISO images. This project does not cater to non-technical users."，CONTRIBUTING.md 的 issue 政策称 "Its target audience is its own developers"，拒绝功能请求式参与。

浏览器引擎 LibWeb/LibJS 是仓库中最活跃的部分，2024 年从本仓库分叉出独立的 Ladybird 浏览器项目（LadybirdBrowser/ladybird），仓内 Ladybird/README.md 仍保留一份旧版浏览器壳用于测试 LibWeb/LibJS（详见第 8 章）。

## 3. 技术栈与依赖

- 语言：全仓 C++（CONTRIBUTING.md 要求 C++26 方言），明确禁止写 C（"Write in C." 列入 Don'ts）；Shell 脚本仅用于构建流程（Meta/*.sh 约 40 个）。
- 构建体系：官方构建为 CMake（≥3.25）+ Ninja，入口包装脚本 Meta/serenity.sh；另保留实验性 GN 构建（Meta/gn/README.md 自述 "The GN build is experimental and best-effort... Serenity's official build system is CMake"）——GN/CMake 双轨中 CMake 为唯一权威轨道。
- 自研替代清单（Ladybird/README.md 列证）：LibCrypto/LibTLS 替代 OpenSSL、LibGfx 替代 skia、LibArchive 替代 zlib/libarchive、LibUnicode/LibLocale 替代 ICU、LibAudio/LibVideo(现并入 LibMedia) 替代 ffmpeg、LibHTTP 为自写 HTTP/1.1 客户端。
- 唯一系统性外部依赖是"构建宿主"侧工具链：Toolchain/ 提供交叉编译 GCC/Clang 工具链构建脚本；Nix flake（flake.nix）与 .devcontainer/ 为开发者环境复现服务。
- 测试/模糊测试：自研 LibTest 框架 + Tests/ 目录（4,192 个文件）；Meta/Lagom/Fuzzers/ 73 个 fuzz target 接入 OSS-Fuzz（README 徽章）。
- 运行时宿主：QEMU（Meta/serenity.sh run），支持硬件/软件虚拟化；也有裸机安装文档（Documentation/BareMetalInstallation.md）。

## 4. 目录结构与代码组织

单体 monorepo：操作系统全栈 + 全部应用 + 第三方 Ports 在一个仓库内按目录分层。

| 目录 | 职责（实测文件数） |
| --- | --- |
| AK/ | 自研基础库（STL 替代），187 个头文件；智能指针/字符串/容器/格式化/SIMD（247 文件） |
| Kernel/ | 自研内核 1,226 文件：Arch 三架构、Syscalls、Memory、FileSystem、Devices、Net、Security 等 |
| Userland/ | 用户空间全量 7,507 文件：Libraries(81 库)/Services(29 守护进程)/Applications(51)/Utilities(224 个工具)/Shell/Games(13)/Demos(11)/DevTools(4) |
| Base/ | 根文件系统骨架（etc/home/res/root/usr/var），3,829 文件；man 手册以 Markdown 存于 Base/usr/share/man |
| Tests/ | 47 个测试目录、4,192 文件，与被测库同名镜像组织 |
| Ladybird/ | 浏览器壳（127 文件）：跨平台 Qt6/AppKit UI + 多进程 helper，用于在仓内测试 LibWeb/LibJS |
| Meta/ | 构建脚本、CMake 模块、Lagom 宿主构建/模糊测试、GN 二轨、CI 脚本（490 文件） |
| Ports/ | 367 个第三方软件移植（bash 脚本 + 补丁），含 AvailablePorts.md 索引 |
| Toolchain/ | 交叉工具链（GNU/Clang）构建脚本与补丁 |
| Documentation/ | 60 个开发者文档：BuildInstructions、CodingStyle、SmartPointers、Patterns、HIG 等 |

组织方式评述：这是"组件化单体"而非微内核多仓——所有代码同一 coding style、同一 AK 类型系统、同一 IPC DSL，目录即模块边界；Library（被复用）与 Service（守护进程）与 Application（用户直接运行的 GUI）与 Utility（命令行）在 Userland/ 下四层正交分类，职责划分非常清晰。Documentation/Patterns.md 与 Documentation/AsynchronousDesign.md 把横切设计模式文档化，属于少见的自我约束。

## 5. 核心架构与关键实现

### 5.1 AK 基础库：一套完整替换 STL 的类型系统

AK/（Andreas Kling 的基础库）共 187 个头文件，是全仓唯一允许的容器/字符串来源（CONTRIBUTING.md："using the `AK` containers in all code"）。要点：

- 智能指针四件套：`OwnPtr/NonnullOwnPtr`（独占）、`RefPtr/NonnullRefPtr`（共享引用计数）、`WeakPtr`，配套 `RefCounted`/`AtomicRefCounted`（AK/RefCounted.h、AK/NonnullRefPtr.h、Documentation/SmartPointers.md）。设计亮点是 Nonnull 变体用类型系统消灭空指针：`NonnullOwnPtr` 不能赋回 `OwnPtr` 之外的空值路径，函数签名直接声明所有权。引用计数带溢出检查（AK/RefCounted.h 中 `Checked<RefCountType>::addition_would_overflow`）。
- 字符串族三代同堂：`ByteString`（C 风格兼容）、新式 `String/StringBase/StringData`（不可变 + FlyString 字符串驻留）、`StringView/Utf8View/Utf16View`（AK/String.h、AK/FlyString.h、AK/StringData.h）。
- 基础设施：`Span`（AK/Span.h）、`Vector/Optional/Variant/Tuple`、fmt 风格的编译期格式串检查 `CheckedFormatString`（AK/Format.h）、`DistinctNumeric/Badge/Traits` 等。
- 内核同用：AK 内含 kmalloc.cpp/kstdio.h，基础库在内核与用户空间共享编译。

### 5.2 Kernel：一次 syscall 一个文件的清晰边界

Kernel/ 共 1,226 文件，子系统划分：

- Arch/：x86_64、aarch64、riscv64 三架构完整实现（Processor/PageFault/TrapFrame/中断控制器等，265 文件）。
- Syscalls/：73 个文件，每个系统调用一个文件（fork.cpp、execve.cpp、mmap.cpp……），可直接对应 POSIX 语义；含 OpenBSD 移植的安全机制 pledge.cpp 与 unveil.cpp（Kernel/Syscalls/）。
- Memory/：经典 VMM 分层——MemoryManager（物理页/PhysicalRegion/PhysicalZone）、AddressSpace（每进程地址空间）、Region、VMObject 抽象（Anonymous/Inode/MMIO 三类，Kernel/Memory/）。
- Security/：内核态 ASan/UBSan 支持（Security/AddressSanitizer.cpp）、Credentials 凭据模型。
- 启动链：Boot/ + Prekernel/ + EFIPrekernel/（EFI 预内核）+ Firmware/（ACPI 等）。
- 用户态/内核边界：Kernel/API/ 提供 syscall 编号与用户态可见头，Userland/Libraries/LibC/（79 cpp，17.5k 行）是另一侧的 libc 实现。

### 5.3 图形栈：WindowServer 组合器 + LibGUI 部件库

图形栈完全自研且按"服务 + 客户端库"分层：

- WindowServer（Userland/Services/WindowServer/，12,367 行）：系统级组合器。Compositor.cpp（1,776 行）实现双缓冲（CompositorScreenData 中 m_front_bitmap/m_back_bitmap，见 Compositor.h）、遮挡重算（recompute_occlusions）与脏矩形增量重绘（`dirty_rect.intersected(frame_rect_on_screen)` 后逐窗口 invalidate，Compositor.cpp:185-208）；WindowManager.cpp（2,504 行）管窗口栈/焦点/菜单；Screen/HardwareScreenBackend/VirtualScreenBackend 分离屏幕逻辑与后端。
- 客户端协议：每个服务用自研 .ipc DSL 声明端点，如 Userland/Services/WindowServer/WindowServer.ipc 中 `endpoint WindowServer { create_menu(i32 menu_id, [UTF8] String name, ...) =| ... }`，由构建期代码生成生成两端 stub——IPC 面向类型且带 UTF8/共享位图等自定义参数类型。
- LibGUI（Userland/Libraries/LibGUI/，125 cpp，33,568 行）：完整部件库（Button/TableView/ComboBox……），UI 用 .gml 声明式标记文件描述（如 Userland/Libraries/LibGUI/AboutDialog.gml），运行时解析加载。
- LibGfx（108 cpp，54,401 行）：2D 绘制（Painter/AntiAliasingPainter）、位图、Font/、ICC/ 色彩管理、ImageFormats/（PNG/JPEG/GIF/PDF 等自研解码器，Tests/LibGfx 有 493 个测试文件）；LibGL+LibSoftGPU 提供软件渲染的 OpenGL，LibAccelGfx/LibVirtGPU/LibV3DGPU 是硬件 GPU 驱动实验线（Userland/Libraries/）。

### 5.4 系统服务架构：29 个守护进程 + 688 行 IPC 内核

Userland/Services/ 实测 29 个守护进程：WindowServer、AudioServer、NetworkServer、RequestServer、WebContent、WebServer、SQLServer、SystemServer（init）、LoginServer、CrashDaemon、FileSystemAccessServer、Clipboard、LaunchServer、LookupServer/DHCPClient（DNS/DHCP）、WebSocket、WebDriver、WebWorker、TelnetServer/SSHServer、SpiceAgent 等。设计特征：

- 一切皆服务：GUI、音频、网络、剪贴板、文件权限全部走进程隔离 + IPC；核心 IPC 框架 LibIPC 仅 4 个 cpp 共 688 行（Userland/Libraries/LibIPC/），复杂度被 DSL 代码生成吸收。
- 浏览器内容隔离：Userland/Services/WebContent/ 以 PageHost/PageClient + ConnectionFromClient 组织每个标签页的渲染进程，配套 ImageDecoder、RequestServer、SQLServer（cookies）三个辅助进程，与 Ladybird 多进程架构（Ladybird/README.md）同构——恶意网页内容与系统其余部分隔离开（README.md "web-content isolation"）。
- WebContent 同时内嵌 WebDriverConnection.cpp 实现 WebDriver 自动化协议，说明服务即浏览器测试基础设施。

### 5.5 LibWeb/LibJS：按 Web 规范组织的引擎（浏览器分叉的遗产）

LibWeb（798 cpp，202,501 行，全仓最大库）按 W3C/WHATWG 规范域切分约 60 个子目录：DOM/CSS/HTML/Layout/Painting/Fetch/Streams/WebIDL/SVG/WebGL/WebAssembly/IndexedDB/ServiceWorker/WebSockets/WebVTT……（Userland/Libraries/LibWeb/ 目录列表），WebIDL 接口用 idl_files.cmake 收集并在 Bindings/ 生成绑定代码。LibJS（271 cpp，95,200 行）为自研 JavaScript 引擎（含 test262 合规追踪，README 徽章链接）。这套引擎即 Ladybird 的心脏：2024 年分叉出独立仓库 LadybirdBrowser/ladybird，本仓保留 Ladybird/ 壳与全部引擎代码（Ladybird/README.md 原文："this version is kept as a developer convenience for the testing of LibWeb and LibJS libraries included with SerenityOS"）。

## 6. 构建与运行

来自 Meta/serenity.sh 使用说明与 Documentation/BuildInstructions.md 的实际命令：

```bash
# 依赖（Ubuntu 示例）
sudo apt install build-essential cmake curl libmpfr-dev libmpc-dev libgmp-dev \
  e2fsprogs ninja-build qemu-system-gui qemu-system-x86 qemu-utils ccache rsync unzip texinfo libssl-dev zlib1g-dev

# 构建跨平台工具链（首次）
Meta/serenity.sh rebuild x86_64        # 或 aarch64 / riscv64 / lagom（宿主构建）

# 构建 + 打镜像 + 在 QEMU 中运行
Meta/serenity.sh build x86_64
Meta/serenity.sh image x86_64
Meta/serenity.sh run x86_64            # gdb 子命令可起远程调试会话

# Lagom：把 Serenity 的库编译到宿主 Linux/macOS 上跑（测试/fuzz 用）
Meta/serenity.sh run lagom js          # 在宿主机直接运行自研 JS 引擎
Meta/serenity.sh test lagom            # 宿主机跑单元测试

# 系统自测试模式（内核测试）
Meta/serenity.sh test x86_64           # 传 system_mode=self-test 启动（serenity.sh 内说明）
```

工具链可选 GNU 或 Clang（`TOOLCHAIN` 参数，serenity.sh 头注释）；GN 二轨用法见 Meta/gn/README.md（`gn gen out && ninja -C out`），但官方明确 CMake 为准。CI 矩阵（.github/workflows/ci.yml）：GNU/Clang × x86_64/aarch64/riscv64 + ALL_DEBUG 组合，另有 nightly.yml 跑 CodeCoverage、lintcommits.yml 校验提交信息格式。

## 7. 测试与工程质量

- 规模：Tests/ 共 4,192 个文件、47 个目录，与库同名镜像（Tests/LibWeb、Tests/AK、Tests/Kernel……）。重心：LibWeb 3,155 个（Layout/Ref/Screenshot/Text/ 与 WPT 目录——已导入 Web Platform Tests，见 Tests/LibWeb/WPT/）、LibGfx 493（图像解码回归样张）、AK 102、Kernel 63（经 Lagom 在宿主机跑内核组件测试，Documentation/RunningTests.md）。
- 框架：自研 LibTest（Userland/Libraries/LibTest/）+ 少量 CTest 集成；LibWeb 单测以 Test*.cpp 命名，像素级回归用 Ref/ 基准图。
- 模糊测试：Meta/Lagom/Fuzzers/ 73 个 fuzz target，OSS-Fuzz 持续集成（README.md 徽章）；内核还内置 Sanitizer 支持（Kernel/Security/AddressSanitizer.cpp、SanCov.cpp）。
- 静态工程化：约 20 个 lint 脚本（Meta/check-*.py/sh：style/markdown/newlines-at-EOF/png-sizes/emoji/idl 等）+ pre-commit 配置（.pre-commit-config.yaml）+ clang-format-20 全仓统一（CONTRIBUTING.md）。
- 质量观察：提交信息纪律极强（`Category: 描述` 祈使句格式，lintcommits.yml 强制；实测近期提交 100% 符合）；每文件带 SPDX 版权头；Documentation/CodingStyle.md 与 Patterns.md 把设计约定成文。无代码覆盖率门禁，但 nightly 有 CodeCoverage 任务（.github/workflows/nightly.yml）。

## 8. 社区与维护状态

- 活跃度（gh api 实测，2026-11-21）：最近 push 2026-11-20；近 30 个 commit 分布在 11-18 至 11-20 三天，单日可达 16 个，主力为 Undefine（cqundefine，RTL8168 网卡驱动系列）、Sönke Holz（spholz，aarch64/ACPI）等社区维护者——项目早已不是单人项目，而是志愿社区驱动（SECURITY.md 自述 "hobby operating system developed by volunteers"）。
- 维护模式与 Andreas Kling：发起人 Kling 累计 16,364 个 commit（gh api search/commits），2024 年随浏览器分叉转向 LadybirdBrowser/ladybird；但实测其最近一次提交为 2026-09-20（LibWeb 相关 3 个提交），说明仍有回归性贡献，日常开发已由社区承接。README 作者列表仍以 Kling 居首（100+ commit 者共 69 人列名）。
- Ladybird 分叉的客观历史：LibWeb/LibJS 诞生于本仓库并支撑了仓内 Browser 应用；2024 年浏览器项目独立为 Ladybird 浏览器组织仓库；本仓保留 Ladybird/ 壳目录（127 文件，Qt6/AppKit UI + ImageDecoder/RequestServer/SQLServer helper）专用于测试引擎（Ladybird/README.md）。仓内 LibWeb 至今仍在高频演进（近期提交多为 LibWeb/WPT）。
- 发布与用户策略：0 个 GitHub release、无 ISO 镜像、无二进制包管理器、ABI 无稳定性保证（Documentation/FAQ.md 明文）；第三方软件走 367 个源码 Ports。历史上 Kling 曾设立 bug 赏金制度（自费奖励漏洞报告者）——此制度为社区渠道（Discord/视频）宣布，仓内文件无记载，未确认。
- 基础设施：GitHub Actions 全矩阵 CI + nightly 覆盖率 + OSS-Fuzz；man.serenityos.org 在线手册由 Base/usr/share/man 的 Markdown 自动生成（.github/workflows/manpages.yml）；Discord 为主要社区（README）。

## 9. 亮点与风险

亮点：

1. 全栈自研且全栈可读：从 EFI 预内核到浏览器引擎约 129 万行 C++ 全在同一仓库、同一风格（README.md、AK/、Kernel/、Userland/），是极少数"任何一行代码都能顺藤摸瓜"的操作系统工程。
2. 类型系统级安全设计：Nonnull 智能指针变体（Documentation/SmartPointers.md）、W^X、pledge/unveil 能力机制、每标签页进程隔离的 WebContent（README.md Features、Kernel/Syscalls/pledge.cpp），把安全偏好写进了语言用法而非补丁。
3. IPC DSL + 服务化架构：688 行 LibIPC 核心 + .ipc 声明文件（Userland/Services/WindowServer/WindowServer.ipc）支撑 29 个守护进程，接口即文档，代码生成消除两端漂移。
4. 测试与 fuzz 文化扎实：4,192 个测试文件、WPT 导入、73 个 OSS-Fuzz target、内核级 ASan（Tests/、Meta/Lagom/Fuzzers/、Kernel/Security/）。
5. 工程纪律罕见地强：提交信息 lint、约 20 个仓库 lint 脚本、三架构 × 双工具链 CI 矩阵（CONTRIBUTING.md、Meta/check-*、.github/workflows/ci.yml）。

风险/局限：

1. 无任何发布与 ABI 稳定性承诺：0 release、无 ISO、ABI 随时可变（Documentation/FAQ.md），不可作为生产系统或依赖库使用。
2. 社区定位收缩：分叉 Ladybird 带走浏览器主线资源与部分核心开发者后，本仓重心退回"自用系统"，issue 政策明确排斥功能请求（CONTRIBUTING.md），外部贡献门槛高（首个 PR 不得新增大型组件）。
3. 体量与构建成本：全仓构建需自建交叉工具链 + QEMU，首次构建时间以小时计；文档自认构建问题不予受理（FAQ "If it builds on CI, it should build for you too"）。
4. 文档滞后于现实：Ladybird/README.md 仍引用已并入 LibMedia 的 LibVideo（Userland/Libraries/ 下无 LibVideo 目录），README 作者名单可能未随人员流动更新——阅读时以源码为准。

## 10. 总结评价

SerenityOS 是开源世界里罕见的"为乐趣与可读性而生的全栈操作系统"：它用约 129 万行 C++、1,412 位贡献者和近乎苛刻的工程纪律，把内核、图形栈、IPC、浏览器引擎与几十个日常应用装进同一个可通读的 monorepo。它的星数与其说来自实用性（明确无发布、无 ABI 承诺），不如说来自其作为"现代操作系统该怎么写"的参考实现价值——AK 类型系统、IPC DSL、服务隔离与测试文化都是可以直接借用的设计范本。适合操作系统/编译器/图形学学习者通读源码，适合想理解"浏览器引擎内部"的工程师从 LibWeb/Layout 与 LibJS 入手，也适合任何团队借鉴其 CONTRIBUTING/CodingStyle/CI 的工程治理实践；不适合寻找可部署 OS 或稳定 C 库的使用者。
