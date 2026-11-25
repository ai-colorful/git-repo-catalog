# microsoft/WSL 项目分析

> 星数 28500（快照 2026-10-24）｜主语言 C++｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

Windows Subsystem for Linux（WSL）——微软官方的"在 Windows 上不加改动直接运行 Linux 命令行工具与应用"的子系统。本仓库是"现代 WSL"（Store 发行版，即 `wsl --install` 安装的那一份）的源码仓库，2024 年 5 月由纯 issue 跟踪器转为开源源码仓库。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | microsoft/WSL | gh api |
| 星数 | 28500（快照）；34,437（2026-11-21 gh api 实测） | gh api |
| 主语言 | C++（C++20，另含 C#/WinUI3 设置界面） | CMakeLists.txt |
| 许可证 | MIT（LICENSE） | LICENSE |
| 默认分支 | master | gh api |
| 仓库创建 | 2016-04-06 | gh api |
| 最近 push | 2026-11-21（分析当天仍在推送） | gh api |
| 开源 issue | 958 | gh api |
| 最新版本 | 3.0.5（2026-11-03） | gh api releases |

## 2. 项目是什么，解决什么问题

WSL 让开发者在 Windows 上直接运行 `bash`、`apt`、`docker`、`gcc` 等未经修改的 Linux 工具，无需传统虚拟机或双系统。它提供两种运行模式：WSL1 通过 Windows 内核中的系统调用翻译层（随 Windows 发行，本仓库不含其源码）直接执行 Linux 二进制；WSL2 则通过 Host Compute System（HCS）启动一个轻量级实用工具虚拟机，运行真正的 Linux 内核（README.md:11、doc/docs/technical-documentation/boot-process.md:46-61）。

本仓库开源的是"现代 WSL"整体产品：Windows 服务 `wslservice.exe`（中枢）、命令行 `wsl.exe`、安装/升级器、设置 GUI（WinUI3）、Linux 侧的系统发行版 init 与网络引擎（GNS）、Plan 9 文件服务器等。相关联但不属于本仓库的部分：Linux 内核（microsoft/WSL2-Linux-Kernel）、Linux GUI 支持（microsoft/WSLg）、官方文档（microsoftdocs/wsl），三者均在 README.md:25-27 中显式列为姊妹仓库。

仓库历史上的特殊性值得记录：2016-04-06 创建后近八年，本仓库主要作为 issue 跟踪器与 `DistributionInfo.json` 等配置的存放地（gh api 提交历史：2023-11 仅有 "ai-similar-issues" 等配置类提交）；2024 年 5 月（Build 2024 期间）微软将 Store 版 WSL 源码导入本仓库并开源（gh api：最早代码提交可追溯到 2024-05-16 前后，2024-05-28 起提交密集）。这一"跟踪器 → 源码仓库"的转变使其 issue 区沉淀了从 2016 年至今的完整用户反馈历史。

发行版管理上，`distributions/DistributionInfo.json` 维护可安装发行版清单（Ubuntu 20.04-26.04、Debian、Kali、openSUSE、SUSE LE、AlmaLinux、Rocky Linux Plus、Fedora、Arch、Azure Linux、eLxr 等），每日由 GitHub Actions 校验下载 URL 与 SHA256（distributions/validate.py、.github/workflows/distributions.yml）。

## 3. 技术栈与依赖

- **语言与标准**：C++20（MSVC 编译 Windows 侧；CMakeLists.txt:296）；Linux 侧用 VS 自带 clang 交叉编译到 `x86_64/aarch64-unknown-linux-musl` 并完全静态链接（CMakeLists.txt:465-531）。设置 GUI 为 C#/WinUI3（约 78 个 .cs 文件，WSL_BUILD_WSL_SETTINGS 开关控制，CMakeLists.txt:184、604-607）。
- **第三方库（FetchContent，全部带 SHA256 锁定）**：GSL 4.0.0、nlohmann/json 3.12.0、yaml-cpp 0.9.0（.wslconfig 解析）、Boost headers 1.90.0（CMakeLists.txt:68-105）。
- **NuGet 依赖（关键）**：Microsoft.WSL.Kernel（Linux 内核二进制包，CMakeLists.txt:129）、Microsoft.WSLg（GUI 插件，:134）、Microsoft.Direct3D.Linux（GPU 半虚拟化，:124）、WIL、TAEF（测试）、MSAL（身份代理）、MSRDC（session host）、bsdtar、Microsoft.WSL.LinuxSdk（musl sysroot）、Wix（MSI 打包）、Windows App SDK/CommunityToolkit（WinUI3）。完整清单见 CMakeLists.txt:124-161 与 nuget.config。
- **构建系统**：CMake ≥3.25 + Visual Studio 生成器（VS2022/VS2026，需 Clang 组件），Windows SDK 10.0.26100.0 强制校验（CMakeLists.txt:2、26-28）。
- **平台**：x64 与 ARM64 双架构（CMakeLists.txt:17-23）。

## 4. 目录结构与代码组织

顶层（tarball 实测 1566 文件、解压约 39MB，C/C++ 约 636 文件 17 万行；测试另计约 8.9 万行）：

| 目录 | 职责 |
|---|---|
| src/windows/service | `wslservice.exe` Windows 服务中枢（约 23.4k 行）：VM/实例生命周期、发行版注册、网络、插件 |
| src/windows/common | Windows 侧共享库（约 42.7k 行）：HCS 虚拟机后端、IO relay、wslc CLI 公共层、Plan 9 重定向器控制 |
| src/windows/wsl | `wsl.exe` 用户入口（薄壳，实际逻辑在 common/wslc 层） |
| src/windows/wslc、wslcsession、WslcSDK | WSL 容器（wslc）CLI、会话与 SDK，独立 CODEOWNERS（wslc-team） |
| src/windows/wslg、wslhost、wslrelay | Linux GUI 启动器、后台宿主（接管 interop）、IO 中继 |
| src/windows/wslinstall、wslinstaller | 安装与升级逻辑及 MSIX 安装器 |
| src/windows/libwsl、wslsettings | C#/WinUI3 设置应用（可选构建） |
| src/linux/init | 系统发行版 `/init`（约 26.9k 行）：GNS 网络引擎、DNS 隧道、drvfs 挂载、binfmt、localhost 转发、seccomp 分发 |
| src/linux/plan9 | Plan 9 文件服务器（Linux 侧文件共享协议实现，约 8.9k 行） |
| src/linux/mountutil、netlinkutil | 挂载工具、netlink 封装 |
| src/shared | Windows/Linux 两侧共用：configfile（.wslconfig）、公共头文件 |
| test/、cloudtest/ | TAEF 测试（windows+linux）与云测试（仅流水线） |
| doc/docs/ | 开发者文档 + 20 篇技术文档（boot-process、gns、plan9、systemd 等） |
| distributions/ | 发行版清单与每日校验脚本 |
| msipackage/、msixinstaller/、msixgluepackage/、nuget/ | 四种发布打包形态 |
| localization/ | 22 种语言的 CLI 字符串 |
| triage/ | issue 排查辅助（trace 收集配置、最新版安装脚本） |
| tools/、cmake/、intune/、storebroker/ | 开发脚本、CMake 模块、企业部署、商店对接 |

组织方式评述：单仓库单 CMake 工程，但产物是多进程/多包形态（service、若干 exe、MSIX/MSI/Nuget）。Windows 侧与 Linux 侧在同一 CMake 中分别配置工具链，代码按"宿主进程"而非按库分层；共享逻辑集中在 common 与 shared 两目录。

## 5. 核心架构与关键实现

**（1）WSL2 启动全链路**（doc/docs/technical-documentation/boot-process.md，以下引用同）：
`wsl.exe` 经 COM 调用 `wslservice.exe` 的 `CreateInstance()`（在注册表查找 `DistributionRegistration`，见 src/windows/service/exe/DistributionRegistration.cpp），再由 `WslCoreVm.cpp` 通过 HCS JSON（schema 见 src/windows/common/hcs_schema.h）创建 VM。VM 内启动顺序为 `mini_init`（initramfs，仅含此一个二进制）→ fork 出 `gns`（网络配置）→ 挂载发行版 VHD 后启动系统发行版内的 `/init` → session leader → `relay` → 用户 bash。Linux 侧三个组件对应 src/linux/init/（mini_init 与 init 同目录编译）与 relay。

**（2）VM 创建（WslCoreVm.cpp，2585 行）**：内核路径解析逻辑展示"内置内核 vs 自定义内核"的优先级——`m_vmConfig.KernelPath` 为空则回退到随包安装的 `WSL_KERNEL_PATH`（CMake 注入的常量，来自 Microsoft.WSL.Kernel NuGet 包），用户可在 .wslconfig 中覆盖：

```cpp
// src/windows/service/exe/WslCoreVm.cpp:191-200（节选）
m_defaultKernel = m_vmConfig.KernelPath.empty();
if (m_defaultKernel)
{
    m_vmConfig.KernelPath = TEXT(WSL_KERNEL_PATH);
}
else
{
    m_vmConfig.KernelPath = m_rootFsPath / LXSS_VM_MODE_KERNEL_NAME;
}
```

**（3）GNS 网络引擎**（src/linux/init/GnsEngine.cpp，746 行 + src/windows/service/exe/GnsRpcServer.cpp）：WSL2 的 NAT/镜像/桥接三种网络模式均由 Windows 侧下发的 netlink 配置驱动——`wslservice.exe` 通过 hvsocket 向 VM 内 `gns` 发送 `LxGnsMessageInterfaceConfiguration`，gns 调用 `src/linux/netlinkutil` 操作内核网络栈；镜像模式（MirroredNetworking.cpp）将 Windows 主机网络接口镜像进 VM，DNS 隧道由 src/linux/init/DnsTunneling*.cpp 实现。

**（4）系统发行版与文件共享**：`/init`（src/linux/init/main.cpp）承担发行版引导（drvfs 挂载 Windows 盘符、binfmt 注册 Windows interop、timezone 同步），文件共享走 Plan 9 协议——Linux 侧服务器在 src/linux/plan9/，Windows 侧通过随系统发行的 `p9rdr.sys` 重定向器挂载为 `\\wsl$` 路径（本仓库仅有控制代码 src/windows/common/Redirector.cpp，重定向器驱动本身随 Windows 发行、不开源）。

**WSL1 边界**：`LxssInstance`（src/windows/service/exe/LxssInstance.cpp）负责 WSL1 模式实例，但真正的系统调用翻译层（lxcore 驱动）随 Windows 发行且闭源——本仓库构建仅引用其头文件（`WSLDEPS_SOURCE_DIR/include/lxcore`，CMakeLists.txt:476、554），这是"开源边界"最有代表性的证据。

## 6. 构建与运行

来自 doc/docs/dev-loop.md:60-93 与 CMakeLists.txt：

```bash
# 前置：VS2022/2026（含 C++ Clang for Windows 组件）、Windows SDK 10.0.26100、管理员终端
cmake .                        # 生成 wsl.sln
cmake --build .                # 全量构建（自动交叉编译 Linux 侧）
cmake . -A arm64               # ARM64 包（构建 bundle 前需先出 ARM64）
cmake . -DBUILD_BUNDLE=TRUE    # MSIX bundle（x64+ARM64）
cmake --build . -- -m && bin\<platform>\<target>\test.bat /name:*UnitTest*   # 测试
```

- 依赖通过 `nuget.config`（内含 NuGet 包）与 FetchContent 自动获取；本地开发可复制 `UserConfig.cmake.sample` 为 `UserConfig.cmake` 加速增量部署（如 WSL_DEV_BINARY_PATH 指向预生成 system.vhd/kernel）。
- 官方构建要求版本号形如 `X.Y.Z.0`（Store 强制 revision=0，CMakeLists.txt:228-230），由 `cmake/findVersion.cmake` 从 git tag 推导。
- 注意：普通开发者可完整构建，但部分二进制（内核、WSLg 插件、D3D Linux 库、wsldevicehost.dll）只能以 NuGet 二进制形式引入，无法从本仓库源码重建（CMakeLists.txt:124-147）。

## 7. 测试与工程质量

- **规模**：test/ 下 101 个 .cpp、约 8.87 万行，测试代码量约为产品源码的一半；分 test/windows（服务/VM 后端/CLI）与 test/linux（init 单测），另有 test/windows/testplugin（服务插件）与 cloudtest/（仅流水线启用，CMakeLists.txt:617-619）。
- **框架**：TAEF（Microsoft.Taef NuGet），运行方式 `TE.exe <test>.dll` 或生成的 `test.bat /name:*UnitTest*`，需管理员权限（test/README.md）。
- **CI**：.github/workflows/ 含 distributions.yml（每日 + PR 校验发行版清单）、winget.yml（发布联动 winget）、new_issue/new_issue_comment/issue_edited.yml（issue 自动化分流）、modern-distributions.yml 等；构建流水线本身在微软内部（仓库内以 PIPELINE_BUILD_ID / OFFICIAL_BUILD 变量接入）。
- **代码质量**：编译期即强约束——`/W3 /WX`（警告即错误）、`/sdl`、Release 加 `/guard:cf /Qspectre`、x64 链接 `/CETCOMPAT`、所有 FetchContent 依赖 SHA256 校验（CMakeLists.txt:314-365）；clang-format 通过生成的 pre-commit 钩子强制（WSL_PRE_COMMIT_MODE 可选 warn/error/fix，CMakeLists.txt:458-462）。 Unity build（WSL_UNITY_BATCH_SIZE）支持可选加速。

## 8. 社区与维护状态

- **活跃度**（gh api，2026-11-21）：当天仍有 push；open issues 958；fork 1898；watcher 595。提交总量约 1698（全部来自 2024-05 开源之后，per_page=1 分页统计）。
- **发版节奏**：GitHub Releases 高频——3.0.5（2026-11-03）、3.0.4（2026-10-20）、3.0.3（2026-10-09）、3.0.2（2026-10-05）、3.0.1（2026-09-29）、2.9.13（2026-09-25），约 2-4 周一版，3.0 已是当前主线（gh api releases）。
- **贡献者**：153 人（含匿名，gh api contributors 分页头实测），前 100 位中微软 WSL 团队成员占主导；CODEOWNERS 将全部路径指派给 `@microsoft/wsl-reviewers`（wslc 相关另加 `@microsoft/wslc-team`），外部贡献需签 CLA（CONTRIBUTING.md:14）。
- **issue 治理**：结构化 Bug_Report.yaml（必填 Windows 版本、WSL 版本、WSL1/2/WSLC 类型）+ feature_request.md；triage/ 提供现场排查工具（收集 WPA trace 的 no-filter.wpaProfile、"装最新版再试"脚本）；微软员工以 label/模板驱动分类，2023 年即有 AI 相似 issue 去重的自动化实践（gh api 提交历史 2023-11-13）。
- **发布渠道**：MSIX（Store/msixinstaller）为主，另有 MSI（企业）、NuGet、GitHub Releases 与 winget；官方构建版本号强制 `X.Y.Z.0` 以满足 Store 要求（CMakeLists.txt:228-230）。inbox 版（随 Windows 内置）与 Store 版并行，微软官方文档（learn.microsoft.com/windows/wsl）推荐 Store 版并通过 `wsl --update` 升级；本仓库开源的是 Store 版代码（README.md:13-17，具体双渠道支持策略未在本仓库内详述，未确认细节以官方文档为准）。

## 9. 亮点与风险

**亮点**
1. 微软核心基础设施真实开源：17 万行 C++ 产品源码 + 20 篇架构文档（含 mermaid 时序图），文档与代码同仓库同步演进（doc/docs/technical-documentation/）。
2. 工程化水平极高：一份 CMake 同时管理 MSVC 与 clang-musl 交叉编译双工具链，依赖全部 SHA256 锁定，编译期安全加固齐全（CFG、Qspectre、CETCOMPAT），/WX 零警告容忍。
3. 测试投入大：TAEF 测试约 8.9 万行，接近产品代码量，覆盖 VM 后端、服务、CLI 到 Linux init 单测。
4. 架构分层清晰：服务（wslservice）—宿主进程（wslhost/wslg/relay）—VM 内组件（mini_init/gns/init）三层均可在源码中一一对应，hvsocket + COM 边界明确。
5. 维护节奏健康：当天仍有提交、月均 1-2 个 release、issue 区持续治理近十年。

**风险/局限**
1. 开源非全量：Linux 内核、WSLg、GPU 驱动（dxgkrnl/D3D）、WSL1 翻译层（lxcore）、p9rdr 重定向器均在仓库之外（外部仓库或随 Windows 闭源发行），无法仅凭本仓库复现完整产品。
2. 构建门槛高：强依赖 VS2022/2026 特定组件、固定版本 Windows SDK 与微软 NuGet 源（含内部二进制包），非 Windows 环境无法构建，社区 fork 的二次开发空间受限。
3. 贡献高度集中：CODEOWNERS 全指向微软团队，外部 PR 需 CLA，社区贡献以 issue 反馈为主。
4. 版本演进快：2.x→3.0 在一年内完成，文档与源码版本可能存在短暂错位。

## 10. 总结评价

microsoft/WSL 是"闭源基础设施逐步开源"的标杆案例：从 2016 年的纯 issue 跟踪器到 2024 年一次性开源 Store 版产品源码，仓库如今包含约 17 万行高质量 C++20 与近乎等量的测试代码，配套文档之详细在微软开源项目中名列前茅。它不是一个"能 fork 就跑"的项目——完整产品依赖内核包、GPU 驱动、系统驱动等仓库外组件——但对于理解现代 WSL 的进程模型、HCS 虚拟机管理、跨 Windows/Linux 的网络与文件共享实现，这是唯一且权威的源码入口。适合：系统级 Windows 开发者、虚拟化/网络方向的工程师研读源码；普通用户则更适合将其视为 issue 反馈与版本发布渠道。
