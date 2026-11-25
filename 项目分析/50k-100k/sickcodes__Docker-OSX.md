# sickcodes/Docker-OSX 项目分析

> 星数 52997（快照 2026-10-24）｜主语言 Shell｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：把 macOS 完整塞进 Docker 容器里跑的 QEMU/KVM 封装套件——`docker run` 一条命令即可在 Linux（及部分 Windows 主机）上启动 macOS 桌面。

| 项 | 值 |
|---|---|
| 名称 | sickcodes/Docker-OSX |
| 星数 | 53,002（gh api 实测 2026-11-04） |
| 主语言 | Shell（实为 Dockerfile + Bash 脚本 + 1 个 Python 脚本） |
| 许可证 | GPL v3+（`LICENSE`） |
| 创建时间 | 2020-06-04（gh api） |
| 最近 push | 2025-11-11（gh api；提交"Add tahoe / Revert default to sequoia"） |
| Fork / 贡献者 | 2,882 forks / 51 contributors（gh api contributors 分页） |
| 发行方式 | 无 GitHub Releases，经 Docker Hub 镜像 tag 分发（`sickcodes/docker-osx:latest` 等） |

## 2. 项目是什么，解决什么问题

Docker-OSX 解决的问题是"在非 Apple 硬件上以最低门槛获得一个可用的 macOS 虚拟机"。裸用 QEMU/KVM 跑 macOS（上游是 kholia/OSX-KVM，见 `README.md` 致谢部分）需要手工准备 OpenCore 引导盘、OVMF 固件、序列号、磁盘镜像和一长串 QEMU 参数；Docker-OSX 把这整套流程烘焙进 Docker 镜像：基础镜像 `archlinux:base-devel` 里装好 qemu-desktop、edk2-ovmf、libguestfs 等依赖（`Dockerfile` L55-178），构建期预生成两个 OpenCore 引导盘（带 picker / 不带 picker，`Dockerfile` L254-276），运行期只需 `--device /dev/kvm` 加环境变量即可开机（README Quick Start 部分）。

项目的第二个卖点是无盘化的合法镜像获取路径：容器内没有预置任何 macOS 版权内容，首次启动时若 `BaseSystem.img` 不存在，会调用 OSX-KVM 的 `make` 目标（内部走 `fetch-macOS.py`）从 Apple 官方软件更新目录 `swscan.apple.com` 下载恢复镜像并 `qemu-img convert` 成 qcow2（`Dockerfile` L364-368，注释称 "DMCA compliant download process"；`fetch-macOS.py` L68-74 定义了各版本 catalog URL）。

第三个卖点是 iMessage/Apple 生态的安全研究支撑：通过 osx-serial-generator 子模块生成"接近真机"的 SMBIOS 序列号组合（SERIAL/MLB/SmUUID/MAC），配合 `-e ADDITIONAL_PORTS` 端口转发与 usbfluxd（iPhone USB 转发），可以做需要 Apple ID 登录环境的安全研究（README Capabilities 部分，`Dockerfile` L283-285）。

项目定位偏研究/开发用途：README 明确面向"Conduct Security Research on macOS using both Linux & Windows"，并以 Apple Bug Bounty 研究者为目标用户之一（`README.md` L580 附近）。

## 3. 技术栈与依赖

- **语言**：Bash（生成/编排脚本）、Dockerfile（4 个变体 + helm/vnc 各自的 Dockerfile）、Python 3 仅 1 个文件（`fetch-macOS.py`、`scripts/apply_appleid_kernelpatch.py`）。
- **虚拟化栈**：QEMU（qemu-desktop）+ KVM 加速（无 KVM 时回退 TCG，`Dockerfile` L315 `ENV KVM='accel=kvm:tcg'`）、OVMF/edk2 固件（OSX-KVM 提供 `OVMF_CODE.fd`/`OVMF_VARS-1024x768.fd`）、OpenCore 引导（acidanthera OpenCorePkg，经 thenickdude/KVM-Opencore 适配 KVM，`README.md` L36-38）。
- **镜像构建辅助**：libguestfs/guestfs-tools（在容器内制作 OpenCore qcow2 引导盘，`Dockerfile` L171-178）、`opencore-image-ng.sh` 源自 kraxel/imagefish（`custom/opencore-image-ng.sh` 头部注释）。
- **外部上游仓库**（构建时 clone，均为浅 clone）：kholia/OSX-KVM（固件、EFI、macserial 工具、Makefile）、sickcodes/osx-serial-generator（git submodule，见 `.gitmodules`）。
- **macOS 镜像获取**：`fetch-macOS.py`（Apache 2.0，源自 munki/macadmin-scripts 的 installinstallmacos.py，`fetch-macOS.py` L1-22）访问 Apple swscan.apple.com 目录。
- **部署周边**：docker-compose.yml、Helm Chart（`helm/Chart.yaml`、`helm/templates/deployment.yaml`）、GitHub Actions 推送 Docker Hub（`.github/workflows/docker-build.yml`）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `Dockerfile` | 主镜像（404 行）：Arch 基础 + QEMU 全家桶 + 构建期生成 Launch.sh + 运行期 CMD 编排 |
| `Dockerfile.auto` | 预装好的 Catalina 桌面镜像（user/alpine），基于 `:latest` 再构建 |
| `Dockerfile.naked` / `.naked-auto` | 自带磁盘镜像（bring-your-own-image）版本 |
| `vnc-version/Dockerfile(.nakedvnc)` | Xvfb + TigerVNC 无头版本（VNC 端口 8888） |
| `fetch-macOS.py` | 从 Apple 官方目录下载 BaseSystem/InstallAssistant |
| `osx-serial-generator/` | **git submodule**（sickcodes/osx-serial-generator）：序列号生成 + OpenCore 引导盘制作 |
| `custom/` | 子模块迁移前的旧副本（plist 模板 + 3 个脚本，README 注明"moved to its own repository"） |
| `helm/` | Kubernetes Helm Chart（deployment/PVC/service/ingress/configmap） |
| `scripts/apply_appleid_kernelpatch.py` | Sonoma 蓝牙启用的 OpenCore 内核补丁生成器 |
| `tests/` | 截图式启动验证脚本 + 各版本参考截图（*_master.png） |
| `.github/workflows/docker-build.yml` | CI：master 推送即构建并发布 Docker Hub latest |
| `rankmirrors` | Arch 镜像测速排序工具（构建加速用） |

组织方式评述：典型的"Dockerfile 即代码"单体仓库——没有传统意义的源码目录，QEMU 启动逻辑以 `tee -a Launch.sh <<< '...'` 的形式逐行写在 Dockerfile 里；四个 Dockerfile 变体之间存在大量复制粘贴（同一 CMD/ENV 块重复四份），可维护性一般但独立性强。功能外溢部分（序列号生成）已拆成独立仓库以 submodule 引用（`custom/README.md`）。

## 5. 核心架构与关键实现

### 5.1 启动编排：构建期生成 Launch.sh，运行期 CMD 状态机

`Dockerfile` 的核心技巧是在构建期用 `tee` 把 QEMU 完整命令行写进 `Launch.sh`（L185-217），把所有可调项暴露成环境变量：

```bash
exec qemu-system-x86_64 -m ${RAM:-4}000 \
-cpu ${CPU:-Penryn},${CPUID_FLAGS:-vendor=GenuineIntel,+invtsc,...,check,}${BOOT_ARGS} \
-machine q35,${KVM-"accel=kvm:tcg"} \
-smp ${CPU_STRING:-${SMP:-4},cores=${CORES:-4}} \
-device qemu-xhci,id=xhci \
-device isa-applesmc,osk=ourhardworkbythesewordsguardedpleasedontsteal\(c\)AppleComputerInc \
-drive if=pflash,format=raw,readonly=on,file=/home/arch/OSX-KVM/OVMF_CODE.fd \
-drive id=OpenCoreBoot,if=none,snapshot=on,format=qcow2,file=${BOOTDISK:-...OpenCore.qcow2} \
-device ide-hd,bus=sata.2,drive=OpenCoreBoot \
-netdev user,id=net0,hostfwd=tcp::${INTERNAL_SSH_PORT:-10022}-:22,... \
-device ${NETWORKING:-vmxnet3},netdev=net0,id=net0,mac=${MAC_ADDRESS:-52:54:00:09:49:17}
```

（`Dockerfile` L194-217，有删节。）要点：macOS 必需的 SMC 模拟（`isa-applesmc` + OSK key）、OVMF 固件双 pflash、OpenCore/安装介质/系统盘分别挂 sata.2/3/4、user 网络的 SSH 与 VNC 端口转发。

运行期 `CMD`（L364-400）是一个纯 Bash 状态机：缺 BaseSystem 就 `make` 下载并转换 → 按需 chown `/dev/kvm`、`/dev/snd` → `NOPICKER=true` 时用 `sed -i '/^.*InstallMedia.*/d' Launch.sh` 原地删除安装盘 drive 并换 nopicker 引导盘（L371-374）→ `GENERATE_UNIQUE=true` 时调 `generate-unique-machine-values.sh` 现场生成序列号引导盘并落盘 `/env`（L376-386，`/env` 文件可 `-v` 挂出供下个容器复用序列号）→ 最后 `enable-ssh.sh && ./Launch.sh`。

### 5.2 OpenCore 引导集成

引导盘制作在子模块 `osx-serial-generator`（tarball 中缺失，经 gh api 补齐分析）。`generate-specific-bootdisk.sh` 的流程：

1. 从 OSX-KVM 取 `OpenCore/EFI` 与 `OcBinaryData/Resources`（Apple 驱动/图标资源），组装出完整 EFI 目录，并写入 `startup.nsh`（内容 `fs0:\EFI\BOOT\BOOTx64.efi`）实现免菜单直启；
2. 对 `config-custom.plist` 模板做占位符替换：`{{DEVICE_MODEL}}、{{SERIAL}}、{{BOARD_SERIAL}}、{{UUID}}、{{ROM}}、{{WIDTH}}、{{HEIGHT}}`（该脚本 help_text 中 "Placeholders" 一节，sed 逐项替换成真实 SMBIOS 值）；
3. `opencore-image-ng.sh`（libguestfs/guestfish）把 EFI 目录打成 qcow2 引导盘。

`generate-unique-machine-values.sh` 的随机序列号来源（L278-286 一带）：`macserial`（OpenCorePkg 工具，OSX-KVM 内置）按机型批量产出 SERIAL/BOARD_SERIAL 对，`uuidgen` 生成 SmUUID，MAC 地址从 `vendor_macs.tsv` 里挑真实 Apple vendor 前缀再拼随机三字节。分辨率/机型/序列号通过 plist 模板固化进引导盘——这就是"换分辨率/换机型必须重建 15MB 引导盘"的设计原因（Dockerfile 注释称 /bootdisk 是 "a useful persistent place to store the 15Mb serial number bootdisk"）。

### 5.3 设备直通与外设方案

直通方案全部通过 Docker/QEMU 两层透传实现（无内核模块改动）：主机层 `--device /dev/kvm`（计算）与 `/dev/snd`（音频），`docker-compose.yml` 则用了更粗暴的 `privileged: true` + 挂载整个 `/dev`；USB 直通靠 `-e EXTRA="-usb -device usb-host,hostbus=1,hostaddr=8"` 直接透传 QEMU 参数（`Dockerfile` L50-53），iPhone/USB 热插拔依赖 corellium 的 usbfluxd（README Capabilities）；显示层三种模式——X11 socket 挂载（`VOLUME /tmp/.X11-unix`）、VNC（vnc-version 镜像，Xvfb + TigerVNC）、GPU 直通（`helm/INSTALL-QEMU-AND-GPU-IOMMU.md` 提供 IOMMU + vfio 全套步骤）。另外 `scripts/apply_appleid_kernelpatch.py` 通过 OpenCore kernel patch（把 `kern.hv_vmm_present` 字符串替换为 `hv_vmm_present=0` 语义的二进制补丁，脚本内 patch1/patch2 的 Find/Replace base64）解决 Sonoma 虚拟机蓝牙不可用问题。

### 5.4 版本矩阵与镜像变体

同一套 Dockerfile 通过 `--build-arg SHORTNAME=<version>` 覆盖 High Sierra 到 Sequoia（当前默认 `ENV SHORTNAME=sequoia`，`Dockerfile` L360；2025-11-11 曾切换默认到 Tahoe 因白屏回退，见 gh api 提交记录）。SHORTNAME 决定 `fetch-macOS.py` 下载的 BaseSystem 版本；Sonoma/Sequoia 使用子模块中单独的 `config-custom-sonoma.plist`（58KB，gh api 文件列表可见）。

## 6. 构建与运行

来自 README/源码的实际命令（未实际运行，按规范仅静态阅读）：

```bash
# 官方镜像直接跑（Catalina）
docker run -it --device /dev/kvm -p 50922:10022 \
    -v /tmp/.X11-unix:/tmp/.X11-unix -e "DISPLAY=${DISPLAY:-:0.0}" \
    -e SHORTNAME=catalina sickcodes/docker-osx:latest

# 本地构建（可指定版本与磁盘大小）
docker build -t docker-osx --build-arg SHORTNAME=sonoma --build-arg SIZE=200G .

# 指定序列号/分辨率生成引导盘启动（Monterey+ 推荐）
docker run ... -e GENERATE_UNIQUE=true -e WIDTH=1920 -e HEIGHT=1080 \
    -e MASTER_PLIST_URL='.../config-custom.plist' -e SHORTNAME=monterey ...

# SSH 进入容器内 macOS：宿主机 50922 -> 容器 10022 -> guest 22
ssh user@localhost -p 50922
```

运行条件（README "Requirements"）：x86_64 主机且 BIOS 开启虚拟化、KVM 可用、磁盘 20GB 起步（Xcode 建议 50GB+）。Windows 主机需 WSL2/Hyper-V 场景，README 有专节说明。Kubernetes 场景用 `helm/` chart（镜像 `sickcodes/docker-osx-vnc`）。

## 7. 测试与工程质量

- **测试**：`tests/test.sh` + `tests/boot-images.sh` 是作者自用的可视化冒烟测试——在 DigitalOcean 主机上用 Xvfb（`DISPLAY=:99`）无头启动各版本镜像，`scrot` 截图后与 `tests/*_master.png` 参考图比对（high-sierra 到 sonoma 共 7 张参考图，`tests/boot-images.sh` L19-27）。没有单元测试框架；测试不进 CI。
- **CI**：仅一条流水线 `.github/workflows/docker-build.yml`——master push 触发 docker build 并打 master/latest 两个 tag 推送 Docker Hub；无 lint、无 shellcheck、无构建矩阵。
- **工程质量观察**：ENV 变量文档化做得好（Dockerfile 内逐项注释），脚本普遍 `set -eux`；但四个 Dockerfile 变体大量重复、Launch.sh 靠 tee 逐行拼接可读性差；CHANGELOG.md 最后条目停在 2021-09（Monterey），此后两年的 Sonoma/Sequoia/Tahoe 支持未再记录，版本管理松散；`docker-compose.yml` 请求 `privileged: true` 与全部 capabilities，属于安全上最宽松的示例配置。

## 8. 社区与维护状态

- 活跃度：仓库创建于 2020-06，2020-2021 年为开发高峰（CHANGELOG 密集记录），此后进入低频维护；最近一次 push 为 2025-11-11（新增 macOS Tahoe 支持后因白屏回退默认 Sequoia，gh api commits），本次分析日（2026-11-04）距上次提交约一年，期间 2025 年只有零星几次提交（6 月、11 月）——**处于"能用但慢速维护"状态，未确认是否有活跃的日常维护计划**。
- Issue：open issues 419 个（gh api），Issue 模板两份（`​.github/ISSUE_TEMPLATE/`），社区规模大但积压明显。
- 贡献者：51 人（gh api contributors 分页 rel=last 推算），核心提交集中在作者 sickcodes 本人。
- Release：无 GitHub Releases；实际发布走 Docker Hub 多 tag（:latest/:auto/:naked/:naked-auto/:high-sierra/:mojave/:big-sur/:monterey/:ventura/:sonoma 及 vnc 变体，README "Technical details" 部分），CI 每次 master push 都会覆盖 latest——发布节奏即提交节奏。
- 社区渠道：Discord（sickchat）与 Telegram 群（README 顶部），Helm Chart 还发布了 Artifact Hub（`helm/artifacthub-repo.yml`）。

## 9. 亮点与风险

**亮点**
1. 把"跑一个 macOS VM"的十几个手工步骤压缩成一条 docker run，OpenCore/OVMF/序列号/磁盘全部自动化，且引导盘只在运行期按需生成（`Dockerfile` CMD 状态机）。
2. 版权内容零分发设计：镜像不含 macOS，运行时才从 Apple 官方服务器拉恢复镜像（`fetch-macOS.py` + `Dockerfile` L356 注释），这是它长期存活的关键设计。
3. 环境变量即配置面：RAM/SMP/CPU/CPUID_FLAGS/EXTRA/ADDITIONAL_PORTS 等覆盖 QEMU 几乎所有常用调参场景，`EXTRA` 透传保证了无限扩展性（`Dockerfile` L38-53）。
4. 序列号体系与主仓库解耦成独立 submodule，可单独复用（osx-serial-generator 同样支持独立使用），引导盘 15MB 可持久化携带序列号身份。
5. 部署形态完整：Docker run / compose / VNC 无头 / Kubernetes Helm 四种开箱形态齐备。

**风险与局限**
1. **Apple EULA/合规边界**：macOS 最终用户许可协议限定 macOS 仅能运行于 Apple 硬件，在非 Apple 硬件上虚拟化运行 macOS 与该许可条款相冲突；本项目不分发 macOS 二进制、引导用序列号由用户自担责任，README 将用法导向 Apple 安全研究场景并另文讨论合法性（`README.md` L580 指向 sick.codes 的合法性分析），但用户在非 Apple 硬件上使用仍需自行评估许可与当地法律风险。此外 Launch.sh 内嵌 Apple SMC OSK 字符串（`Dockerfile` L200），其授权状态存在争议。本分析仅客观记录，不构成规避指导。
2. 需要宿主 KVM 与 x86_64：在纯云 Windows/无嵌套虚拟化环境不可用；TCG 回退模式性能极差（`Dockerfile` L315）。
3. 维护趋缓：默认版本跟随新 macOS 的兼容性靠少量提交维持（Tahoe 白屏问题即例证），419 个 open issue 积压；CHANGELOG 与实际进度脱节两年以上。
4. 镜像与工程质量成本：基础镜像内置整套 QEMU/libvirt/libguestfs 与 200G 稀疏磁盘，体积大；四个 Dockerfile 变体重复度高，修改需多处同步；compose 示例默认 privileged，直接照抄有安全暴露面。

## 10. 总结评价

Docker-OSX 是"把困难的事变成一条命令"的典范项目：它本身几乎没有可执行的核心代码，价值全部在于对 OSX-KVM + OpenCore 这套脆弱链路的工程化封装——构建期生成 QEMU 脚本、运行期状态机编排、序列号体系子模块化、以及刻意不碰版权红线的镜像获取设计，共同成就了 53k 星。对 Apple 安全研究者、需要在 CI/容器环境做 macOS 兼容验证的工程师、以及想读懂 QEMU/OpenCore/macOS 引导链如何拼装的读者，它的 Dockerfile 就是最好的教学材料；但生产使用需清醒认识 EULA 边界与性能天花板，且 2025 年后维护节奏放缓，新 macOS 版本跟进存在滞后与不确定性。
