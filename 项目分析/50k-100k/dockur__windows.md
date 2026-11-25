# dockur/windows 项目分析

> 星数 53674（快照 2026-10-24）｜主语言 Shell｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：在 Docker 容器里通过 QEMU/KVM 运行完整 Windows（从 Windows 95 到 Windows 11 / Server 2025），一条命令完成镜像下载与全自动无人值守安装。

| 项目 | 内容 |
|---|---|
| 名称 | dockur/windows（镜像名 dockurr/windows） |
| 星数 | 53674（快照）；分析当日 gh api 显示 54067 |
| 语言 | Shell（Bash，本仓库约 1.7 万行脚本） |
| 许可证 | MIT（license.md） |
| 创建时间 | 2024-01-14（gh api） |
| 最近推送 | 2026-11-02（gh api），极活跃 |
| 最新 release | v6.06（2026-10-05，gh api releases） |
| 默认分支 | master |

## 2. 项目是什么，解决什么问题

给 Linux 容器装一个 Windows 传统上是件麻烦事：手工找 ISO、写 autounattend.xml、配 QEMU 参数、处理驱动和激活前的安装流程。dockur/windows 把这条链路完全自动化——用户只需 `docker compose up`，然后浏览器打开 8006 端口等桌面出现（readme.md "How do I use it?" 一节）。它本质上是"容器编排的完整虚拟机"而非"容器化的 Windows"：容器内跑 QEMU，Windows 以 VM 形式运行，KVM 提供接近原生的性能（readme.md Features）。

版本覆盖面是它最突出的卖点之一：通过 `VERSION` 环境变量可选 Windows 11/10/8.1/7/Vista/XP/2000/ME/98/95、Server 2003–2025，甚至 Tiny11 与 ReactOS，共 20 余种（readme.md 版本表）。ISO 一律从微软官方服务器下载（含校验和验证），项目本身不分发任何 Windows 镜像（readme.md "Is this project legal?"，src/define.sh getMido 函数中的 URL 均指向 software-static.download.prss.microsoft.com）。

无人值守安装按 Windows 世代采用三代不同的应答技术：Vista 以后用 autounattend.xml（src/answer.sh + assets/*.xml），NT5 世代（XP/2000/2003）用 winnt.sif（src/sif.sh），Win9x 世代用 MSBATCH.INF（src/batch.sh），另有 src/legacy.sh 处理遗留安装路径。这让"全自动装 XP"和"全自动装 Win11"成为同一条命令。

容器状态在 /storage 卷中持久化（版本、安装源、启动模式、磁盘镜像等状态文件，见 src/install.sh finishInstall 写 stateFile），因此容器重启后直接引导已装好的系统，不会重装。安装过程中断也能恢复：中止的 Win11 安装会在下次启动时自动补开 Secure Boot/TPM 以满足硬件检查（src/install.sh L435-L448）。

该项目已成为一个生态支点：WinBoat、WinPodX、WinApps 等桌面集成项目都以它为后端（readme.md "Desktop applications"）；同一作者维护着 dockur/macos、dockur/windows-arm 姊妹项目与 qemux/qemu 通用基础镜像。

## 3. 技术栈与依赖

- **Bash**：全部编排逻辑（src/*.sh 12 个脚本共约 1.7 万行，`set -Eeuo pipefail` 严格模式）。
- **QEMU**：qemu-system-x86_64 来自预编译镜像 qemux/qemu-windows:11.2.23（Dockerfile L54）；通用 VM 脚本模块（start.sh/utils.sh/disk.sh/network.sh/boot.sh/finish.sh 等）来自基础镜像 qemux/qemu:7.50（Dockerfile L6、src/entry.sh L11-L35 的 source 列表）。
- **ISO/镜像处理**：wimtools、mtools（mformat/mcopy 构建 FAT32 setup 镜像）、cabextract、gcab、libarchive-tools、xmlstarlet（应答文件 XML 注入）、自建 udfread:1.2.0（Dockerfile L19-L53）。
- **网络发现与共享**：Samba + wsdd-native 1.27（Windows 网上邻居发现，Dockerfile L38-L40）、dxvk-native 3.1.1（宿侧 Vulkan 渲染路径）。
- **驱动**：virtio-win WHQL 驱动包 1.9.61，构建时 ADD 到 /var/drivers.txz 供安装时注入（Dockerfile L56）。
- **打包**：多阶段 Dockerfile，amd64 从 scratch 组装，arm64 直接基于 dockurr/windows-arm；入口 tini -s /run/entry.sh（Dockerfile L58-L72）。
- **CI**：GitHub Actions（shellcheck、hadolint、XML/JSON/YAML 校验、真实安装测试）。

## 4. 目录结构与代码组织

| 路径 | 职责 |
|---|---|
| `Dockerfile` | 多阶段构建：基础镜像 + Windows 专属工具链 + 驱动包 |
| `compose.yml` / `kubernetes.yml` | Docker Compose 与 K8s（PVC + Deployment + Service）部署样例 |
| `src/entry.sh` | 容器入口编排器（80 行） |
| `src/define.sh`（2242 行） | 版本/语言/edition 解析，静态下载目录与镜像清单 |
| `src/mido.sh`（1467 行） | Windows ISO 下载：微软在线路由 → 静态目录回退 |
| `src/answer.sh`（2234 行） | autounattend.xml 生成与定制（账户/域/区域/产品键） |
| `src/image.sh`（2035 行） | ISO/ESD 检测、解包、版本识别、setup 镜像构建 |
| `src/install.sh`（1748 行） | 安装主流程状态机（startInstall/bootWindows/finishInstall） |
| `src/batch.sh` / `src/sif.sh` / `src/legacy.sh` | Win9x（MSBATCH.INF）与 NT5（winnt.sif）无人值守 |
| `src/display.sh`（1106 行） | 显示后端：VNC/web/VGA 设备、GPU/Vulkan 运行时探测 |
| `src/samba.sh` / `src/power.sh` | 宿客文件共享；优雅关机与引导状态探测 |
| `assets/*.xml` + `*.cmd` | 各版本 autounattend 模板与安装后脚本（33 个文件） |
| `.devcontainer/` | 21 个按版本预设的 Codespaces/devcontainer 配置 |
| `.github/workflows/` | build/check/test/install/validate 等 9 个工作流 |

组织方式：无任何应用代码依赖管理，是"shell 单体 + 数据模板"结构。Windows 专属逻辑集中在 src/，与基础镜像的通用 VM 模块通过 entry.sh 的 source 顺序拼装，职责切分清晰。

## 5. 核心架构与关键实现

### 5.1 启动编排（src/entry.sh）

入口只有 80 行：以固定顺序 source 约 23 个模块（基础镜像的通用模块 + 本仓库的 Windows 模块），随后创建命名管道 qemu.pipe 收集 QEMU 串口输出，用 sed 流式过滤 ANSI 转义序列与噪音后转发到 PTY——power.sh 据此判断引导状态。默认 `exec qemu-system-x86_64` 前台运行：

```bash
cmd=(qemu-system-x86_64)
pipe="$QEMU_DIR/qemu.pipe"
rm -f "$pipe" && mkfifo "$pipe"
tee "$QEMU_PTY" <"$pipe" |
sed -u -e 's/\x1B\[[=0-9;]*[a-z]//gi' ...
output=$!
if ! enabled "$SHUTDOWN"; then
  exec "${cmd[@]}" ${ARGS:+ $ARGS} >"$pipe" 2>&1
fi
```

QEMU 运行参数（CPU/内存/磁盘/网络/显示）由各模块依次向 `ARGS` 追加，环境变量（RAM_SIZE、CPU_CORES、DISK_SIZE 等，默认值见 Dockerfile L67-L70 与 docs/environment.md）驱动每一步。

### 5.2 ISO 下载与校验（src/mido.sh、src/define.sh）

下载采取三层策略（src/mido.sh getWindows L556-L657）：优先走微软在线路由（downloadWindows/downloadWindowsEval/getCatalog 解析 catalog.cab 与 ESD 结构）；失败后回退到 define.sh getMido 内置的静态目录——每条记录带官方 URL、精确字节数与 SHA256；verifyFile（mido.sh L1077）负责校验。静态回退同样指向微软官方 CDN：

```bash
"win11x64" )
  size=7736125440
  sum="d141f6030fed50f75e2b03e1eb2e53646c4b21e5386047cb860af5223f102a32"
  url="https://software-static.download.prss.microsoft.com/dbazure/.../...CLIENT_CONSUMER_x64FRE_en-us.iso"
  ;;
```

语言与平台的约束判断也在这一层（非英语不可得的版本直接报错、ARM64 仅限显式列出的路由，mido.sh L576-L604）。

### 5.3 无人值守安装（src/answer.sh、src/image.sh、src/install.sh）

按检测到的镜像 edition 从 assets/$id.xml 取模板，用 xmlstarlet 注入账户、区域、域加入、产品键等配置（answer.sh generateAnswerFile L229-L356），并刻意保留空的 ProductKey 结构但移除可能误选 edition 的具体键（L265-L271）。安装介质方面，image.sh createSetupImage（L619 起）用 mtools 构建一个 FAT32 overlay 镜像，装入应答文件、$OEM$/OEM 目录与用户自定义 install.bat，且先在临时文件上构建校验后再发布，避免半写状态成为引导介质。install.sh 是安装状态机：startInstall 准备临时区、bootWindows 从状态恢复、finishInstall（L385-L521）把版本/安装源/启动模式/系统镜像持久化到 /storage，Win11 中止安装自动切换 windows_secure 模式。整个流程对用户表现为"开机 → 全自动 → 出桌面"。

### 5.4 访问暴露：Web 查看器与 RDP（src/display.sh、compose.yml）

显示后端由 `DISPLAY` 变量决定，默认 `web`：QEMU 以 VNC 监听一个 Unix socket 上的 websocket（`-display vnc=:N,websocket=unix:$QEMU_DIR/vnc-wss.sock`，display.sh L57-L68），8006 端口的 web 查看器由基础镜像 server.sh 提供并代理该 socket；`vnc` 模式则开放原生 5900。RDP 是 Windows 自身的远程桌面服务：容器将 3389/tcp+udp 映射到宿主（compose.yml），经 QEMU 网络转发进入 guest，默认账户 Docker/admin（readme.md）。display.sh 其余约千行是 GPU 加速路径的探测（virtio-gpu/venus、NVIDIA/Mesa Vulkan、VMware SVGA），失败则回退普通 VGA。

## 6. 构建与运行

运行（来自 readme.md）：

```bash
docker run -it --rm --name windows -e "VERSION=11" -p 8006:8006 \
  --device=/dev/kvm --device=/dev/net/tun --cap-add NET_ADMIN \
  -v "${PWD:-.}/windows:/storage" --stop-timeout 120 docker.io/dockurr/windows
```

或 `docker compose -f compose.yml up`；K8s 用 `kubectl apply -f kubernetes.yml`。要求 Linux 宿主支持 KVM（Docker Desktop on Linux/macOS 不支持，readme.md Requirements）。本仓库无本地构建/测试命令；镜像由 CI 构建（build.yml：shellcheck 通过后 buildx 多平台推送 Docker Hub + GHCR 并打 release 标签）。

## 7. 测试与工程质量

- **静态检查**：每个 PR 跑 shellcheck（`-x --source-path=src`）+ hadolint（Dockerfile）+ assets 下 XML 校验 + JSON/YAML 校验（.github/workflows/check.yml）。
- **端到端安装测试**：install.yml 是可复用工作流，参数化 VERSION、QEMU 机型、CPU 模型、磁盘控制器、网络后端、网卡型号等，在 GitHub runner 上做真实安装；validate.yml 在 release 发布后按矩阵（Win11 Pro/LTSC 等各版本的 expected_caption/expected_edition/minimum_build）验证安装产物。以纯 Bash 项目而言这是相当重的真实验证投入。
- **代码质量观察**：脚本普遍防御式书写——显式错误码分级（50/66/67/70/72/74/76）、mktemp+原子 mv 发布、btrfs 关 COW 特判（image.sh L651-L658）、进程替换的 find 显式 wait 收状态（image.sh L666-L677）。无单元测试框架，质量保障完全依赖 lint + 真实安装 CI。

## 8. 社区与维护状态

- 2024-01 创建，两年多积累 5.4 万星（gh api 54067），5092 forks，33 位贡献者（gh api contributors）。
- 推送活跃：最近一次 2026-11-02；release 节奏约每月一版（v6.02 至 v6.06，2026-07 至 2026-10，gh api releases）。
- issue 模板分四类（issue/feature/bug/question，.github/ISSUE_TEMPLATE/），开启数 19（gh api），维护响应良好（未确认具体响应时长）。
- Dependabot + Renovate 配置齐备（.github/dependabot.yml、renovate.json），Docker 镜像在 Docker Hub 与 GHCR 双发布。

## 9. 亮点与风险

**亮点**

1. 端到端全自动：空容器 → 自动下载校验 ISO → 自动应答安装 → 8006 网页看桌面，用户体验门槛极低。
2. 下载链路干净：全部来源为微软官方服务器并带 SHA256 校验，静态回退目录同样官方化，项目不分发 Windows 镜像（src/define.sh、readme.md 合规声明）。
3. 版本矩阵罕见地宽：三代无人值守技术（autounattend/winnt.sif/MSBATCH.INF）覆盖 95→11 与 Server 2003–2025。
4. CI 真装验证：install/validate 工作流按机型/驱动/网络矩阵做真实安装回归，对 shell 项目少见。
5. 状态机完备：安装状态持久化于 /storage、重启续用、中止安装自愈（补 TPM/Secure Boot），细节工程化程度高。

**风险与局限**

1. **Windows 许可合规边界（如实记录）**：项目代码为 MIT 且不含 Windows 本体；README 明示代码中的产品键是微软公开发布的通用安装键，仅用于安装流程选择版本，不是有效激活许可，并要求用户自备合法许可、遵守微软许可条款（readme.md "Is this project legal?"）。仓库内未发现激活/破解工具代码。能否在容器中运行 Windows、以何种方式合规授权，取决于用户所处场景与微软许可协议，属使用者责任。
2. 依赖微软官方下载端点与目录结构的稳定性，微软调整页面/API/CDN 路径时下载路由可能失效（mido.sh 中大量 fallback 与错误分支反映了这一脆弱性）。
3. 性能与平台受限：无 KVM 的环境（macOS、Docker Desktop on Linux 等）不可用或严重降速；模拟开销下默认 2 核 4G 只够轻量使用。
4. 暴露面与默认凭据：8006/3389 端口对外、默认账户 Docker/admin、Samba 共享默认开启，公网部署必须改凭据并做网络隔离。

## 10. 总结评价

dockur/windows 是"把麻烦事做对"的典范工程：它没有发明新虚拟化技术，而是把 QEMU 编排、微软官方下载源、三代 Windows 无人值守应答、驱动注入和状态机管理组合成一条开箱即用的流水线，并以真实安装 CI 维护着一个横跨 30 年的版本矩阵——这正是它能两年冲到 5 万+ 星的原因。适合需要临时、一次性或自动化 Windows 环境（CI 测试、软件兼容性验证、隔离沙箱）的运维与开发者直接使用；适合想学习大型 Bash 项目组织、QEMU 参数编排与 Windows 无人值守安装机制的工程师阅读源码。生产环境长期运行前，务必自行解决 Windows 授权、凭据与网络暴露问题。
