# ventoy/Ventoy 项目分析

> 星数 80297（快照 2026-10-24）｜主语言 C｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：开源多系统 U 盘启动工具——装一次 Ventoy，之后把任意 ISO/WIM/IMG/VHD(x)/EFI 镜像文件直接拷进 U 盘即可开机引导，无需反复格式化或逐个写盘。

| 项 | 值 |
|---|---|
| 仓库 | ventoy/Ventoy（master 分支） |
| 星数 | 80297（快照 2026-10-24） |
| 主语言 | C（另含大量 shell 脚本） |
| 许可证 | GPL-3.0（`COPYING`） |
| 创建时间 | 2020-03-10（gh api） |
| 最近 push | 2026-10-22（gh api） |
| 最新 release | v1.1.18（2026-10-09，gh api） |
| forks / open issues | 5027 / 1011（gh api） |

## 2. 项目是什么，解决什么问题

传统启动盘制作工具（Rufus、dd、UltraISO 等）一次只能写入一个镜像，换系统就要重新格式化写盘。Ventoy 的做法是把 U 盘一次性划成两个分区（一个大容量数据分区 + 一个约 32MB 的 VTOYEFI 引导分区，见 `DOC/BuildVentoyFromSource.txt` 与 `INSTALL/Ventoy2Disk.sh`），之后用户只需把镜像文件当普通文件拷贝进数据分区，启动时 GRUB 菜单自动枚举并列出所有镜像供选择。镜像文件无需连续存放、支持 >4GB、升级 Ventoy 版本不破坏数据（`README.md` Features 列表）。

实现"免解压直接引导"的关键在于 device-mapper 拼接：选中镜像后，运行时环境把 U 盘上镜像文件所在的扇区区间通过 dm 设备虚拟成一块完整的"光盘/磁盘"，再把 ISO 里的内核与 initrd 拼接加载。这样对被引导的 Linux 内核来说，看到的就是一块正常的安装介质（`IMG/cpio/ventoy/ventoy_chain.sh`、`GRUB2/MOD_SRC/grub-2.04/grub-core/ventoy/ventoy_cmd.c`）。

兼容性是它最大的工程投入：不同发行版的 initrd 组织、init 程序、udev/systemd 行为差异巨大，Ventoy 在 `IMG/cpio/ventoy/hook/` 下维护了 40+ 个发行版目录（debian、arch、rhel7、suse、nixos、alpine、gentoo、android 等），每个目录里放置针对该发行版的挂载/注入 hook 脚本；`README.md` 声称已测试 1300+ ISO，覆盖 distrowatch 90%+ 发行版。

平台覆盖上，同一套方案同时支持 x86 Legacy BIOS、IA32/x86_64/ARM64/MIPS64EL UEFI 与 MBR/GPT 分区表；Windows 侧提供 Ventoy2Disk.exe 图形安装器，Linux 侧提供 shell 脚本安装器与 GTK/Qt 图形界面（`LinuxGUI/`）。

## 3. 技术栈与依赖

- **C**：自有代码约 21.6 万行（按 IMG/VtoyTool/vtoycli/Plugson/Ventoy2Disk/vtoyjump/LinuxGUI/GRUB2 模块/EDK2 模块/FUSEISO/VBLADE 的 `.c` 文件粗估）。
- **GRUB 2.04**：引导核心，仓库内 `GRUB2/MOD_SRC/grub-2.04/` 保存对上游的增量修改（`buildgrub.sh` 用官方 tarball + 补丁源码树重新编译）。
- **EDK2 stable201911**：编译 UEFI 下的 Ventoy EFI 应用与 VDiskChain 驱动（`EDK2/edk2_mod/`）。
- **第三方集成**：exFAT 1.3.0 + libfuse 2.9.9（`ExFAT/`）、squashfs-tools 4.4（`SQUASHFS/`）、zstd/lz4/lzip/xz 解压组件（`ZSTD/`、`LZIP/`）、vblade（AoE，`VBLADE/`）、wimboot、ipxe（`IPXE/`）。
- **预编译二进制**：busybox、imdisk、memdisk、7za、dmsetup 等，来源与 SHA-256 清单见 `DOC/BuildVentoyFromSource.txt` 第 5 节与 `BLOB_List.md`。
- **构建环境**：CentOS 7.8 + musl/dietlibc/交叉工具链（gcc-linaro aarch64、mips64el musl 等，`DOC/BuildVentoyFromSource.txt`）。
- **Windows 侧**：Visual Studio 2013+ 解决方案（`Ventoy2Disk/Ventoy2Disk.sln`、`vtoyjump/vtoyjump.sln`）。
- **Plugson**：自有 C HTTP 服务器 + 纯 HTML/JS 前端（`Plugson/src/Web/ventoy_http.c`、`Plugson/www/`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `GRUB2/` | grub-2.04 的 Ventoy 修改源码（grub-core/ventoy/ 23.8k 行）与构建脚本 |
| `EDK2/` | UEFI 端 Ventoy/VDiskChain EFI 应用源码（基于 tianocore EDK2） |
| `IMG/` | 运行时 initrd 环境：cpio 打包脚本 + ventoy/hook/ 各发行版 hook + loop 模式脚本 |
| `INSTALL/` | 发布包骨架：grub 配置（grub.cfg 2838 行）、安装/更新脚本、工具二进制、插件样例 |
| `Ventoy2Disk/` | Windows 安装器（VS 工程，xz-embedded 内嵌） |
| `vtoyjump/` | Windows PE 下挂载 ISO 的辅助程序（VS 工程） |
| `vtoycli/` | Linux 命令行工具：FAT 分区扩容、GPT 操作、CRC32 |
| `VtoyTool/` | 磁盘镜像/DM 相关命令行工具（vtoydm、vtoydump 等，约 3.9k 行） |
| `Plugson/` | VentoyPlugson：Web GUI 插件配置器（C HTTP 服务器 + www 前端） |
| `LinuxGUI/` | Linux 图形安装器（GTK/Qt 双后端）+ WebUI |
| `Unix/` | FreeBSD/DragonFly 等的 kmod 源码与打包 |
| `FUSEISO/` | vtoy_fuse_iso：用户态挂载 ISO 供浏览 |
| `ExFAT/ SQUASHFS/ ZSTD/ LZIP/ VBLADE/ DMSETUP/ cryptsetup/ vtoyfat/ vtoygpt/` | 各第三方组件源码/构建脚本 |
| `LiveCD/ LiveCDGUI/` | Ventoy 自身可启动 LiveCD 镜像构建 |
| `DOC/` | 从源码构建的完整文档（BuildVentoyFromSource.txt 等） |
| `.github/workflows/` | CI（ci.yml、sync2gitee.yml） |

组织方式评述：这是"宿主仓库 + 大量 vendored 第三方源码 + 预编译二进制"的单体仓库，不是标准 autotools/cmake 工程——顶层没有统一的构建入口文件，构建入口是 `INSTALL/all_in_one.sh` 与 `DOC/BuildVentoyFromSource.txt` 手册。每个子目录自带 build.sh，各自负责一个组件；发布时由 `INSTALL/ventoy_pack.sh` 组装成最终安装包。这种结构对上游构建者门槛较高，但对普通用户透明。

## 5. 核心架构与关键实现

### 5.1 GRUB 定制模块：启动链的大脑

`GRUB2/MOD_SRC/grub-2.04/grub-core/ventoy/`（13 个文件共 23838 行）以 GRUB 模块形式注入 70+ 个 `vt_*` 命令（命令注册表见 `ventoy_cmd.c` 第 7000 行附近的 `ventoy_cmds[]` 数组），例如：

```c
{ "vt_img_sector", ventoy_cmd_img_sector, ... },
{ "vt_load_cpio", ventoy_cmd_load_cpio, ... },
{ "vt_trailer_cpio", ventoy_cmd_trailer_cpio, ... },
{ "vt_select_auto_install", ventoy_cmd_sel_auto_install, ... },
{ "vt_linux_parse_initrd_isolinux", ventoy_cmd_isolinux_initrd_collect, ... },
```

关键机制：`vt_img_sector` 记录所选镜像在磁盘上的扇区分布；`vt_load_cpio` 把 Ventoy 运行时 cpio（busybox + hook 脚本）与后续镜像内 initrd 拼接；`vt_linux_parse_initrd_isolinux/grub` 主动解析目标发行版 initrd 的组装规则，保证 Ventoy 的 cpio 能正确混入。`ventoy_plugin.c`（3657 行）在 GRUB 内部解析 `/ventoy/ventoy.json`，实现菜单别名/类/密码/自动安装模板/持久化/注入文件/配置替换等十余类插件配置（数据结构见该文件头部 `g_menu_alias_head`、`g_persistence_head` 等链表）。

### 5.2 grub.cfg 菜单引擎

`INSTALL/grub/grub.cfg`（2838 行）+ `INSTALL/grub/` 下 checksum/debug/localboot/power/keyboard 等子配置。选镜像后进入：

```grub
function vt_vcfg_pre_proc {
    vt_img_sector "${vtoy_iso_part}${vt_chosen_path}"
    vt_load_cpio  $vtoy_path  "${vt_chosen_path}" ${vtoy_iso_part} "busybox=$ventoy_busybox_ver"
    vt_trailer_cpio $vtoy_iso_part "$vt_chosen_path"
}
```

（`INSTALL/grub/grub.cfg`）即"计算镜像扇区 → 前置加载运行时 cpio → 追加尾部 cpio"，随后引导 ISO 内核。

### 5.3 运行时链式引导与发行版 hook

`IMG/cpio/ventoy/ventoy_chain.sh`（517 行）是被注入内核 initrd 的第一个用户态程序，流程分五步（源码注释即 Step1-5）：解析内核参数 → `ventoy_get_os_type()` 用 `/proc/version` 与特征文件识别 40+ 发行版 → 执行 `hook/$VTOS/ventoy-hook.sh`（借助 `hook/default/` 下的 udev 规则把 Ventoy 分区上的镜像挂为 dm 设备）→ 执行用户 LiveInjection 钩子 → `exec` 交还真实 init：

```sh
VTOS=$(ventoy_get_os_type)
if [ -e "$VTOY_PATH/hook/$VTOS/ventoy-hook.sh" ]; then
    $BUSYBOX_PATH/sh "$VTOY_PATH/hook/$VTOS/ventoy-hook.sh"
fi
```

（`IMG/cpio/ventoy/ventoy_chain.sh` 416-423 行）`hook/debian/` 下还有 antix/tails/knoppix/pve/truenas 等衍生发行版的细分 hook，是项目兼容性工作量的直接体现。

### 5.4 插件体系与 Plugson 配置器

插件配置是数据分区 `/ventoy/ventoy.json`（样例 `INSTALL/plugin/ventoy/ventoy.json` 含 control/theme/menu_class/menu_alias/auto_install 等段）。GRUB 端（ventoy_plugin.c）负责菜单类配置，运行时端 hook 脚本负责持久化/注入类配置。Plugson 是本地 Web GUI 配置器：`Plugson/src/Web/ventoy_http.c`（5503 行）用 C 实现内嵌 HTTP 服务器，跨平台入口 `Plugson/src/main_linux.c`/`main_windows.c`，前端为 `Plugson/www/` 下 30+ 个 html 页面，由 `INSTALL/VentoyPlugson.sh` 拉起，把可视化表单写回 ventoy.json。

### 5.5 安装器组织

Windows 安装器 `Ventoy2Disk/Ventoy2Disk/`（VDS/diskpart/WMI 多磁盘服务后端：`DiskService_vds.c`、`DiskService_diskpart.c`、`DiskService_wmsa.c`）；Linux 侧 `INSTALL/Ventoy2Disk.sh` 按 `uname -m` 选择 x86_64/aarch64/mips64el/i386 工具目录，配合 `vtoycli/`（FAT 分区无损扩容 `partresize.c` 等）；UEFI 原生环境由 `EDK2/edk2_mod/.../Application/Ventoy/Ventoy.c`（1356 行）与 VDiskChain 提供。

## 6. 构建与运行

- 一键构建：`cd INSTALL && sh all_in_one.sh`（仅重编 grub2/EDK2/IPXE，其余组件使用预编译二进制，`DOC/BuildVentoyFromSource.txt`）。
- Docker 构建（CI 同款）：仓库根 `docker compose up`（`docker-compose.yml`，privileged 容器内跑 `INSTALL/docker_ci_build.sh` 下载工具链与上游二进制后编译打包）。
- 单组件构建：如 `cd GRUB2 && sh buildgrub.sh`、`cd EDK2 && sh buildedk.sh`、`cd Plugson && sh build.sh`。
- Windows 安装器：VS2013+ 打开 `Ventoy2Disk/Ventoy2Disk.sln` 编译。
- 用户运行：下载 release 包，Windows 执行 Ventoy2Disk.exe，Linux 执行 `sh Ventoy2Disk.sh -i /dev/sdX`（`INSTALL/Ventoy2Disk.sh`）；Web 控制台 `VentoyWeb.sh`、插件配置器 `VentoyPlugson.sh`。
- CI：`.github/workflows/ci.yml` push/PR 到 master 触发 docker compose 构建，上传 windows/linux/livecd 产物与 SHA256SUM。
- 注意：仓库无单元测试，`all_in_one.sh` 不构建 32 位组件（见构建文档说明）。

## 7. 测试与工程质量

- 无自动化测试框架：未发现任何 test 目录或测试 runner（全仓搜索仅命中 vendored xz-embedded 自带的 `xz_dec_test.c` 等，属上游代码）。质量保障依赖真实硬件上的镜像兼容性测试（1300+ ISO 测试列表，`README.md`），并有 issue 模板 `success_image_report.yml` 收集用户"测试成功报告"作为众包回归。
- CI 只做"能编译、能打包"验证（`.github/workflows/ci.yml`），无 lint/静态分析配置。
- 代码质量观察：shell 脚本大量使用全局变量与字符串拼接，防御性较强（日志 `$VTLOG`、调试断点 `VTOY_BREAK_LEVEL` 机制，`ventoy_chain.sh`）；C 侧日志与防御式编程（`ventoy_log.c`、参数校验）风格统一；多架构抽象良好（DiskService 三后端、TOOLDIR 按架构切换）。

## 8. 社区与维护状态

- 活跃度高：创建于 2020-03，最近 push 2026-10-22（gh api），近 4 个月连发 v1.1.14-1.1.18（2026-06-24/25、07-24、10-09，gh api releases），节奏稳定。
- 强个人主导：作者 longpanda（admin@ventoy.net，多数源文件版权头）为核心，社区以 issue 反馈 + 测试报告参与；贡献者结构细节未在仓库内确认（tarball 无 git 历史）。
- open issues 1011（gh api），体量大但与用户基数相符；README 提供 issue 模板（含硬件成功报告专用模板 `.github/ISSUE_TEMPLATE/`）。
- 生态：CI 同步镜像到 Gitee（`.github/workflows/sync2gitee.yml`），官网 ventoy.net，文档与插件手册独立维护在官网。

## 9. 亮点与风险

亮点：
1. 巧妙的 dm 拼接 + initrd cpio 注入设计，实现"拷贝即启动"，绕开了逐发行版适配内核参数的大部分工作（`GRUB2/.../ventoy_cmd.c` + `IMG/cpio/ventoy/ventoy_chain.sh`）。
2. 兼容性工程极其扎实：40+ 发行版 hook 目录 + 双 BIOS/UEFI 五架构同构支持（`IMG/cpio/ventoy/hook/`）。
3. 插件闭环完整：json schema → GRUB 内解析 → 运行时消费 → Plugson Web GUI 生成，覆盖自动安装/持久化/注入/密码等高级场景（`Plugson/`、`INSTALL/plugin/`）。
4. 全架构交付纪律好：每个二进制（含第三方）在 `DOC/BuildVentoyFromSource.txt` 第 5 节登记 URL 与 SHA-256，可审计。
5. 文档质量高，从源码构建每一步可复现。

风险/局限：
1. 构建体系重：CentOS 7.8 基线、多套交叉工具链、依赖已停更的 yum 源，新贡献者复现成本高（`DOC/BuildVentoyFromSource.txt`）。
2. 仓库携带大量预编译二进制（.exe/.efi/.ko 等），供应链信任面大，虽有名单位 SHA-256 清单缓解。
3. 兼容性本质上是"追着发行版打补丁"的模式：每次上游 initrd/init 变化都可能破坏 hook，长期维护负担线性增长（hook 目录规模可证）。
4. 无自动化测试与静态检查，回归主要靠人工 + 社区报告，主分支质量依赖作者个人验证。

## 10. 总结评价

Ventoy 是系统运维领域少有的"改写了启动盘使用方式"的工具，星数与其实用性完全匹配。它的核心价值不在单点代码技巧，而在三层设计：GRUB 模块层的扇区级镜像虚拟化、initrd 层的发行版自动识别与 hook 注入、应用层的插件 + Web 配置器闭环。适合所有需要频繁引导多种系统镜像的运维/装机人员使用；对想深入 UEFI/BIOS 启动链、GRUB 扩展开发或 device-mapper 应用的人来说，`GRUB2/MOD_SRC/grub-2.04/grub-core/ventoy/` 与 `IMG/cpio/ventoy/` 是两份极具学习价值的真实工程范本，但参与上游构建需要做好重建 CentOS 7 环境的心理准备。
