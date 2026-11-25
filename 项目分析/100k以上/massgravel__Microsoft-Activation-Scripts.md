# massgravel/Microsoft-Activation-Scripts 项目分析

> 星数 195691（快照 2026-10-24）｜主语言 Batchfile｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

MAS（Microsoft Activation Scripts）是一个开源的 Windows/Office 激活工具集，以纯批处理脚本实现四种激活技术（HWID、Ohook、TSforge、Online KMS）及配套的激活状态检查、版本切换与故障修复功能。本仓库只承载源码（.cmd 脚本），无二进制产物。

| 项目 | 事实 |
|---|---|
| 名称 | Microsoft Activation Scripts (MAS) |
| 星数 / Fork | 195691 / 18663（gh api，2026-10-30 实测 196826） |
| 主语言 | Batchfile（内嵌 PowerShell 与运行时编译的 C#） |
| 许可证 | GPL-3.0（`src/LICENSE`） |
| 创建时间 | 2020-01-12（gh api） |
| 最近 push | 2026-10-17（gh api） |
| 最新版本 | v3.12 "8 Years of MAS"，2026-07-04（gh api releases；README.md 亦标 3.12） |
| 官网/文档 | massgrave.dev（README.md 链接） |

## 2. 项目是什么，解决什么问题

MAS 是一个面向 Windows 与 Office 批量/个人授权场景的激活脚本集合。它把过去散落在各论坛的多种激活技术整合为一套带交互菜单、可无人值守运行（命令行开关）的脚本，并通过 GitHub / Azure DevOps 镜像 / 自建 Gitea 三处分发（README.md 底部徽章区、`MAS/Separate-Files-Version/_ReadMe.html` 重定向）。每个激活方法覆盖的产品与时效不同（`MAS/Separate-Files-Version/Activators/_ReadMe.txt`）：

- HWID：Windows 10/11 数字许可证，永久
- Ohook：Office，永久
- TSforge：Windows / ESU / Office，永久
- Online KMS：Windows / Office，180 天，可由续期任务维持

技术上，四个方法对应四条不同路径：HWID 走微软官方的"数字许可证"通道（伪造硬件指纹票据）；Ohook 以 DLL 劫持方式拦截 Office 的授权校验；TSforge 直接改写软件保护平台（SPP）的本地二进制许可存储；Online KMS 则是传统 KMS 协议模拟，连接公共 KMS 服务器。除脚本外，仓库还包含大量修复工具：重建许可令牌、修复 WMI、修复服务权限等（`MAS/Separate-Files-Version/Troubleshoot.cmd`），这些功能使其不只是一个激活器，也是一套 Windows 授权子系统的诊断修复工具箱。

需要强调：本项目属于**授权绕过工具**，其全部功能价值与全部风险均来自这一点（详见第 9 章）。本文仅做客观技术分析。

## 3. 技术栈与依赖

- 语言：Batch（cmd.exe 脚本）为主体；内嵌 PowerShell 代码块（`%psc%` 调用，脚本内自切分提取）；TSforge 另嵌入约 5000 行 C#（`@'...'@` here-string，运行时经 `[scriptblock]::Create` 执行并编译）。
- 运行时依赖（均为 Windows 自带）：cmd.exe、Windows PowerShell 5.x、`slmgr.vbs`/`clipup.exe`/`DISM`/`schtasks`/`reg.exe`/`cscript`（ospp.vbs）、WMI（SoftwareLicensingProduct / OfficeSoftwareProtectionProduct）。
- 第三方代码：TSforge 内嵌的 `ActivationWs` 命名空间改编自微软员工 Daniel Dorner 的开源项目 activationws（MIT，脚本内保留版权声明，`TSforge_Activation.cmd` 约 3756 行起）。
- 无包管理器、无构建系统、无 CI：仓库即最终发布物，发布为 git tag + 多站点镜像。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `README.md` | 安装入口（irm/iex 一行命令、直链下载）、版本号、社区链接 |
| `LICENSE` | GPL-3.0 全文 |
| `MAS/All-In-One-Version-KL/MAS_AIO.cmd` | 全家桶单文件版，19,228 行 / 748KB，主菜单整合全部功能 |
| `MAS/Separate-Files-Version/Activators/HWID_Activation.cmd` | HWID 数字许可证激活，2,151 行 |
| `MAS/Separate-Files-Version/Activators/Ohook_Activation_AIO.cmd` | Office DLL 劫持激活，3,362 行 |
| `MAS/Separate-Files-Version/Activators/TSforge_Activation.cmd` | SPP 存储改写激活，9,806 行 |
| `MAS/Separate-Files-Version/Activators/Online_KMS_Activation.cmd` | KMS 协议模拟，4,235 行 |
| `MAS/Separate-Files-Version/Change_Windows_Edition.cmd` / `Change_Office_Edition.cmd` | 版本切换（DISM / 许可证重装），1,459 / 1,693 行 |
| `MAS/Separate-Files-Version/Check_Activation_Status.cmd` | 激活状态查询（署名 @abbodi1406），1,599 行 |
| `MAS/Separate-Files-Version/Troubleshoot.cmd` | 授权子系统修复合集，1,747 行 |
| `MAS/Separate-Files-Version/Extract_OEM_Folder.cmd` | 提取 $OEM$ 部署文件夹，843 行 |

组织方式评述：这是一个"每脚本自包含"的单仓库多入口结构。各 .cmd 共享约 400 行相同的启动序言（环境变量修正、x86/x64/ARM64 重启动、管理员提权、PowerShell 可用性检测、QuickEdit 处理、DNS 式更新检查，见 `HWID_Activation.cmd` 行 27-407），而不是抽公共库——因为分发形态要求单文件可独立运行。AIO 版则是把四个激活器 + 工具脚本逐字拼接并以 `:MainMenu` 分发（`MAS_AIO.cmd` 行 426-510）。全部约 4.6 万行代码集中在 10 个脚本内，模块边界即文件边界。

## 5. 核心架构与关键实现

### 5.1 HWID：伪造数字许可证票据（`Activators/HWID_Activation.cmd`）

流程：按 SKU 从内表选通用零售密钥并安装（`:hwiddata key`，行 1979 起的密钥表，密钥字符用 `%f%` 切分防 AV 字符串匹配）→ 必要时临时把 Windows 区域改为美国（注释：部分地区商店许可不可用）→ 用内嵌 RSA 私钥签名生成 `GenuineTicket.xml` → 先后用"重启 ClipSVC 服务"与 `clipup -v -o` 两种方式应用票据以提高成功率 → 触发 SPP 重评估激活。

票据签名的核心（`:sign:` 段，行 2042 起）：

```powershell
[byte[]] $key = 0x07,0x02,0x00,0x00,0x00,0xA4,0x00,0x00,0x52,0x53,0x41,0x32,...  # 1024+ 字节 CSPBlob
$rsa = New-Object Security.Cryptography.RSACryptoServiceProvider
$rsa.ImportCspBlob($key)
$PropertiesStr = "OA3xOriginalProductId=;OA3xOriginalProductKey=;SessionId=$SessionId;..."
$SignatureStr = SignProperties $PropertiesStr $rsa   # rsa-sha256 PKCS1
$xml = "<?xml ...><genuineAuthorization ...><signature name=""clientLockboxKey"" method=""rsa-sha256"">$SignatureStr</signature>..."
```

即脚本内自带一份能通过微软票据签名校验的私钥材料，将设备属性签名后写入 ClipSVC 票据目录，让官方流程签发数字许可证。

### 5.2 Ohook：Office sppc.dll 劫持（`Activators/Ohook_Activation_AIO.cmd`）

- 在 Office 的 C2R vfs 目录（`%_oRoot%\vfs\System[X86]`）用 `mklink` 把系统真实 `sppc.dll` 链接为 `sppcs.dll`，再释放一个自带的同尺寸劫持 DLL 作为 `sppc.dll`（`oh_hookinstall` 段，行 1131-1172）：Office 加载劫持 DLL，DLL 再转发真实调用，从而在授权判定处返回"已授权"。
- 劫持 DLL 以 URL-safe 变体 base64 内嵌于脚本（`-`→`A`、`_`→`a`），按 `:sppc32.dll:` / `:sppc64.dll:` 分隔提取；释放时由 `:hexedit:` 段在内存中动态组装 PE 并**随机改写时间戳字段改变校验和**（`:oh_extractdll`，行 3127；注释原文："to lower the potential false positive detection by antivirus's. On each install, it will install a unique sppc dll file"）。
- 同时安装 MAK 密钥与 `.xrm-ms` 许可文件（经 WMI `InstallLicenseArr`，行 1125），并写注册表抑制"非正版"横幅（行 750-765）。

### 5.3 TSforge：SPP Physical Store 直接改写（`Activators/TSforge_Activation.cmd`）

最大的脚本（9,806 行），核心是 `:tsforge:` 段（行 3744 起）内嵌的两个 C# 命名空间：

- `ActivationWs`：对接微软官方批量激活 Web 服务（BatchActivate，含 HMAC-SHA256 签名的请求构造），支撑 StaticCID 方法——向微软服务器提交确认 ID。代码注明改编自官方 activationws 示例。
- `LibTSforge`：按 Windows 版本（Vista/Win7/Win8/WinBlue/WinModern）实现 SPP `data.dat` 物理存储的解密/解析/重加密（`PhysStoreCrypto.Decrypt/EncryptPhysicalStore`，行 5775 起；`SPPUtils.GetStore`，行 5416），随后写入伪造的产品密钥数据、TrustedStore 数据，或直接放入确认 ID（ZeroCID 离线法 / StaticCID 在线法），并内置 KMS v4/v5 响应二进制块（`Constants.KMSv4Response/KMSv5Response`，行 3985 起）支撑 KMS4k 方法。batch 侧调用形式：`$f=[IO.File]::ReadAllText('!_batp!') -split ':tsforge\:.*'; . ([scriptblock]::Create($f[1])) %tsids%`（`:ts_act`，行 1951）。

TSforge 还支持 ESU 包、KMS 主机（CSVLK）、APPX 旁加载激活，以及"重置重新计数/评估期/篡改状态"（`/Z-Reset` 参数，行 44）。

### 5.4 Online KMS：公共 KMS 服务器轮换（`Activators/Online_KMS_Activation.cmd`）

- 内置 15 个公共 KMS 主机名（`_tasksetserv`，行 2229 起，主机名同样以 `-%-%` 切分混淆），随机选取后用 `ping -4`/`pathping` 解析出 IP（注释：直接用主机名激活易被微软/AV 标记），写入 SPP 与 Office 两条注册表路径的 `KeyManagementServiceName/Port`（`_taskregserv`，行 2191 起）。
- 创建每周续期计划任务 `Activation-Renewal`（XML 任务定义内嵌于脚本，行 2351 起；落盘于 `C:\Program Files\Activation-Renewal\`，含日志），保证 180 天周期续满。
- 顺带写 `NoGenTicket` 注册表项，阻止 Windows 向微软回传 KMS 客户端激活数据（引用微软隐私文档，行 979 起）。

### 5.5 横切能力：诊断、自检与修复

- `:dk_chkmal`（HWID 行 1205 起）：检测系统中是否残留 KMSPico 等 PUP 激活器、hosts 文件是否屏蔽四大杀软域名、是否存在文件感染型病毒迹象，并删除 R@1n-KMS 每分钟跑一次的残留任务——即主动帮用户识别"从 Google 搜到的有毒激活器"。
- 每个脚本启动时自检：Null 服务存活、行尾必须为 CRLF（`findstr /v "$"`，行 133）、PowerShell 语言模式、按 `updatecheck<版本号>.activated.win` 的 DNS 解析做更新检查（行 371-405）。
- `Troubleshoot.cmd` 提供重建 ClipSVC 许可、重建 SPP/OSPP 令牌、WMI 仓库修复、令牌目录权限修复等十余项修复动作，是激活失败时的兜底层。

## 6. 构建与运行

项目无构建环节，源码即成品。基于 README 与脚本内参数注释（如 `HWID_Activation.cmd` 行 14-23）可归纳的运行形态：

- 交互菜单：运行任一 `*.cmd`（如 `MAS_AIO.cmd`）出现数字菜单选择。
- 无人值守：每个脚本支持命令行开关，如 `/HWID`、`/Ohook`、`/Ohook-Uninstall`、`/K-Windows`、`/K-NoRenewal-Task`、`/Z-WindowsESUOffice`、`/Z-Reset` 等；亦可在脚本头部把 0 改 1 后运行。
- 无测试命令、无 lint/CI 配置（仓库无 `.github/` 工作流文件，tarball 目录清单可证）。

## 7. 测试与工程质量

- **测试**：无任何自动化测试、无 CI。正确性依赖对目标 OS 构建的逐版本手工适配（脚本内大量按 `winbuild`、SKU 分支的兼容代码），质量问题以 issue/社区渠道反馈修复。未确认是否有私有验证流程。
- **兼容性工程**：覆盖率罕见的宽——从 Windows Vista 到 26100+（Win11 新构建）各分支均有处理（TSforge 的 PSVersion 枚举、Vista/Server2008 专用密钥表，`TSforge_Activation.cmd` 行 885-903）；x86/x64/ARM64 重启动、Sysnative 处理、Terminal/QuickEdit 兼容（含对 microsoft/terminal#15212 的 workaround，各脚本行 48）。
- **代码质量**：注释密度高且多为解释"为什么"（如为什么改区域、为什么两次应用票据、为什么混淆字符串）；错误处理为逐检查点打印彩色状态行并在失败处给出指向官方排障页的链接。单文件 2 万行的形态牺牲了可维护性，换来了零依赖分发。
- **AV 对抗是显式设计目标**：字符串切分混淆、每次安装改写 PE 时间戳、IP 化 KMS 服务器等均写有注释说明动机（见 5.2/5.4）。仓库内多份脚本首行即是对杀软误报的申诉（`MAS_AIO.cmd` 行 1-5）。

## 8. 社区与维护状态

- 活跃度：master 分支最近 push 2026-10-17（gh api）；近 5 个 release 间隔约 2-8 个月（v3.8 2025-11-11 "R.I.P. KMS38" 移除了 KMS38 方法，v3.12 2026-07-04），节奏稳定且持续增删激活方法。
- 贡献者：API 列出 9 名 contributors；`Check_Activation_Status.cmd` 署名知名开源开发者 @abbodi1406（文件头注释）。核心代码风格单一，推断实际维护者极少（未确认具体人数）。
- 分发生态：GitHub 之外还有 Azure DevOps 镜像与自建 Gitea（README.md 行 52-79），主站 massgrave.dev 提供文档；社区渠道为 Discord/Reddit/Bluesky/X。
- 事件痕迹：脚本头部与 `:oh_extractdll` 注释记录了杀软误报拉锯与脚本反复应对的历史；v3.8 移除 KMS38 说明作者随微软策略变化主动重构功能集。

## 9. 亮点与风险

亮点（技术层面）：

1. 单文件脚本做到了极致的自包含与自诊断：环境自愈、架构自适配、行尾自检、DNS 更新检查，几乎不依赖任何外部组件即可在 Vista~Win11 全系运行。
2. 四种激活方法各自代表一条清晰的技术路线（票据伪造 / DLL 劫持 / 本地存储改写 / 协议模拟），TSforge 的 LibTSforge 对 SPP 物理存储格式（加解密 + 版本分叉）的逆向实现具有研究价值。
3. 附带的 Troubleshoot 与 dk_chkmal 使其同时是授权子系统修复与"恶意激活器"检测工具。
4. 批处理工程实践参考价值高：延迟展开切换、字符转义、动态拼 PowerShell/C#、单文件内多语言代码的切分执行模式。

风险与局限（客观记录）：

1. **法律与合规**：本项目功能本质是绕过微软产品激活与授权校验，违反微软许可条款（EULA），在多数司法辖区可能违反反规避类法律；GPL-3.0 只覆盖代码本身，不改变行为的违法性。本分析不构成使用建议。
2. **杀毒软件检测（本地实证）**：本次分析在解压源码后数秒内，本机杀软实时防护即将 4 个激活器脚本及 Change_Office_Edition、Extract_OEM_Folder、Troubleshoot 共 7 个文件自动隔离删除（仅剩非激活类脚本），本文对激活器源码的读取改由 tarball 流式完成——脚本被主流杀软广泛标记是既成事实，部署环境会因此持续冲突。
3. **系统完整性风险**：脚本重写 SPP 二进制存储（`data.dat`）、替换系统 DLL 副本、批量改注册表与计划任务，失败或半途可能损坏授权子系统；作者为此内置了修复模块，反向印证风险真实存在。
4. **供应链面**：README 主推的 `irm https://get.activated.win | iex` 属"下载即执行"模式，域名可被 ISP/DNS 屏蔽或仿冒；README 自己也提醒用户警惕第三方伪造 URL。

## 10. 总结评价

MAS 是 GitHub 上规模与完成度都罕见的批处理项目：约 4.6 万行纯 cmd/PowerShell/C# 混合代码，以零依赖、零构建的方式把四种激活技术、全套自诊断与修复工具压进 10 个自包含脚本，工程细节（兼容性、错误呈现、无人值守参数、AV 规避）体现出长期的实战打磨，这解释了其近 20 万星的关注度——多数用户为使用而来，但代码本身也确实是一部"Windows 授权子系统攻防"的活教材。它的合法性缺陷是根本性的，任何使用决策涉及法律风险，应由使用者自行判断。适合谁读源码：Windows 安全研究员（理解 SPP/ClipSVC/票据机制与对应攻防）、批处理高级技巧学习者（单文件多语言架构、环境韧性处理）；不适合作为可复用的工程模板（单文件形态与强攻击意图使其不可移植）。
