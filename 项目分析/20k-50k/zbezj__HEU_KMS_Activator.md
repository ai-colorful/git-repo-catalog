# zbezj/HEU_KMS_Activator 项目分析

> 星数 44269（快照 2026-10-24）｜主语言 —（无，gh api 返回 language=null）｜档位 20k-50k｜分析日期 2026-11-06

## 1. 项目速览

一句话定位：**Windows / Office 激活工具 "HEU KMS Activator" 的纯发布仓库**——仓库本体不含任何源码，只放了两份 PDF 使用手册和一个空 README，真正的可执行程序全部通过 GitHub Releases 以 .rar/.iso 二进制资产分发。

| 项目 | 事实 | 出处 |
|---|---|---|
| 名称 | zbezj/HEU_KMS_Activator | gh api repos/... |
| 星数 / Fork / Open Issue | 44,269 / 3,958 / 182 | gh api（2026-11-06） |
| 主语言 | 无（language=null，树内无代码文件） | gh api |
| 许可证 | 无（license=null，无 LICENSE 文件） | gh api + tarball 文件列表 |
| 仓库创建 | 2019-06-23 | gh api |
| 默认分支 | master，最后 push 2026-07-06 | gh api |
| HEAD 树内容 | 仅 3 个文件：README.md（19 字节）+ 2 份 PDF 手册（合计约 3.7MB） | src.tar.gz（tar -tzf） |
| Releases | 54 个（tag 与 release 数一致），v24.1.0（2021-07）→ v64.0.0（2026-07） | gh api releases/tags |
| 贡献者 | 1 人（zbezj） | gh api contributors |

## 2. 项目是什么，解决什么问题

HEU KMS Activator 是一款 Windows 桌面图形程序，自称的功能范围覆盖 Windows（Vista~11、Server 2008~2025）与 Office（2010~2024、Microsoft 365）的多种激活方式，包括：数字许可证（HWID）激活、TSForge 激活、KMS38 激活、KMS 180 天激活、OEM 模拟激活（SLIC）、Office 的 OHook 激活与 Retail 转 Volume（R2V），以及版本转换、密钥管理、激活信息备份还原等辅助功能（出处：仓库内 `HEU_KMS_Activator使用说明手册.pdf`，21 页全文提取）。

需要首先澄清仓库性质：这是典型"**发布仓库**"（distribution-only repo）。HEAD 树只有 3 个文件——`README.md`（内容仅一行 `# HEU_KMS_Activator`，19 字节）和两份不同版本的使用手册 PDF。没有任何源代码、构建脚本或工程清单；GitHub 上标记的仓库体积约 107MB（gh api size=109713KB）与 HEAD 树约 3.8MB 的差额，主要沉淀在 git 历史中。程序本体以 `HEU_KMS_Activator_v64.0.0.rar` / `.iso` 形式挂在 Releases（单包约 5~6MB）。

手册对该工具的机制有客观描述：其"智能激活"按 数字许可证(HWID) → TSForge → KMS38 → KMS 的优先级自动选择方式；其中 HWID 需联网向微软激活服务器请求写入数字许可证；TSForge 向软件保护平台（SPP）的信任存储写入伪造的电话激活数据（release notes 注明"基于开源程序 MAS3.0"，见 v50.0.0 tag）；KMS38 将 180 天批量激活延长至 2038 年；KMS 方式则在本机模拟一个 KMS 主机（见第 5 章）。这些均为该软件文档的自述内容，此处仅作客观记录。

仓库的社区角色更像一个"下载页 + 更新日志归档"：54 个 release 的 body 都是逐版本的更新日志（中文，含每版 exe 的 MD5/SHA1 校验值），issue 区（182 个 open）承担用户反馈渠道。44k 星所度量的并非代码资产，而是这个二进制工具的用户规模与传播广度。

其官方手册（`HEU_KMS_Activator使用说明手册.pdf`，21 页）的章节结构即功能范围的客观清单：

| 手册章节 | 自述功能范围 |
|---|---|
| 1.1 概述 | 主界面、两套 UI（新版/经典）、三语（简体/繁体/英文）、激活方案 1/2、调试日志（%temp%） |
| 1.2 智能激活页面 | Windows+Office 一键自动选择激活方式（HWID/TSForge/KMS38/KMS 优先级）、Office R2V、OHook、ESU 激活、KMS 自动续期、激活状态查看/授权重置/许可证重建 |
| 1.3 KMS 页面 | 内置 KMS 服务器（两种注入模式 + 在线 KMS）、KMS4K、自动续期（自动续订模式/任务计划模式）、清除 KMS 客户端信息、手动搭建 KMS 服务器（5 步按钮流）、KMS HOST 激活 |
| 1.4 数字激活页面 | HWID 数字许可证激活（联网、永久）、KMS38 激活与添加/解除 KMS38 保护 |
| 1.5 OEM 激活页面 | Legacy 模式（HEUGR 引导 + GRUB4DOS）与 UEFI 模式（WindSLIC.efi）、卸载 OEM 激活、生成 $OEM$ 部署目录 |
| 1.6 其他页面 | Win10/11 版本转换、Office 管理工具（VL 证书安装/自动 R2V/清除证书/修复"盗版受害者"横幅）、密钥管理（安装/卸载/查看/确认 ID）、激活信息备份还原 |
| 1.7 静默参数 | 命令行参数体系（可组合），供系统封装场景部署阶段调用 |
| 第 2 节 | 免责声明（见第 9 章风险） |

## 3. 技术栈与依赖

仓库内无源码与工程清单，无法从代码层面确认技术栈（未确认）。从手册与 release notes 可客观记录的运行期依赖与技术构成：

- **目标运行环境**：Windows XP 至 Windows 11、Windows Server 2008R2~2025（手册概述与 v42.0.1、v24.2.0 等 release notes）。
- **运行时落盘组件**（手册 KMS 章节）：KMS 模式 1 复制 `SECOPatcher.dll`、`kms-server.exe` 到 `C:\Windows\System32`；模式 2 复制 `SppExtComObjHook.dll`；OEM 激活在 Legacy 模式写引导文件 `HEUGR` 并修改 bootcode（基于 GRUB4DOS），UEFI 模式写入 `WindSLIC.efi` 到 `\EFI\WindSLIC\`；自动续期安装计划任务 `HEU KMS Renewal`（`C:\Windows\HeuKmsRenewal\HEU_KMS_Renewal.exe`）。
- **上游方法依赖**：release notes 明确注明 TSForge"基于开源程序 MAS3.0"（v50.0.0）、OHook 的 OSPP 支持"基于开源程序 MAS3.2"（v62.0.0）、数字激活 IID 更新"来自 massgravel/Microsoft-Activation-Scripts"（v64.0.0）——即本工具的激活方法与开源项目 MAS 同源。
- **分发格式**：Releases 资产为 .zip（早期，带密码，文件名含 Password_HEU168）→ .rar → v62.0.0 起双格式 .iso + .rar 并附 MD5/SHA1。
- **界面**：自述支持简体/繁体/英文三语、新/经典两套 UI（手册 1.1 概述）。实现语言未确认（二进制不可审计）。

## 4. 目录结构与代码组织

HEAD 树（tarball `src.tar.gz`，3,317,163 字节 gzip）完整成员清单，共 3 个文件：

| 文件 | 大小 | 职责 |
|---|---|---|
| `README.md` | 19 B | 仅一行标题 `# HEU_KMS_Activator`，无任何说明 |
| `HEU_KMS_Activator使用说明手册.pdf` | 1,745,413 B | 21 页主手册：使用方法 7 小节 + 免责声明 |
| `HEU_KMS_Activator_v60使用说明.pdf` | 2,063,426 B | 26 页 v60 版本手册（章节结构略有不同的旧版手册） |

（出处：`tar -tzf src.tar.gz`；文本经流式 `tar -xzOf` + pypdf 提取，未落盘解压。）

组织方式评述：仓库不存在代码组织，"结构"实际上由 **Releases 通道** 承担——54 个 tag（版本号跳跃式增长：24.x→25→26→27→28→30.x→40~42→50→60→64，跳号无版本说明，未确认原因），每个 release body 即该版 changelog，资产组织随时间演化（zip+密码 → rar → iso+rar 双格式，近五个版本的 .iso 字节数完全相同，均为 6,045,696 B，疑似固定尺寸容器）。文档、分发、反馈三种职责分别落在：树内 PDF（文档）、Releases（分发）、Issues（反馈）。

## 5. 核心架构与关键实现

无源码可读，本章基于官方手册（`HEU_KMS_Activator使用说明手册.pdf`）与 release notes 对软件文档化的机制做客观技术性简述。

**（1）分发架构（仓库本身的核心"实现"）**
Releases 是唯一分发面：v64.0.0（2026-07-06，最新）资产为 `HEU_KMS_Activator_v64.0.0.iso`（6,045,696 B）+ `.rar`（5,112,665 B）；release body 以固定格式书写更新日志并附 exe 的 MD5/SHA1（如 v63.3.4 给出 MD5），供下载者自行校验。早期版本（2022 年，v25.0.0~v26.2.1）的 zip 资产文件名直接标注密码 `HEU168`。仓库 HEAD 上的两份 PDF 是随树更新的官方文档（最新树 mtime 2025-03-24，对应 v60 时期）。

**（2）多激活方式的调度（手册 1.2 智能激活）**
"智能激活"按固定优先级自动选择：联网时首选数字许可证（HWID），离线首选 TSForge，失败或不支持则回退 KMS38、KMS；检测到已永久激活则跳过。这构成工具的主入口逻辑（自述，未审计）。

**（3）KMS 模拟（手册 1.3 KMS 页面）**
手册对 KMS 协议背景有客观说明：KMS（Key Management Service）是微软面向企业批量授权的激活方式，局域网内一台 KMS Host 先行激活，客户端向其请求激活，有效期 180 天、自动续期（未激活客户端每 2 小时尝试、已激活每 7 天续订，Windows 需 25 台/Office 需 5 台客户端阈值）。本工具的"内置 KMS 服务器"即在本机模拟 KMS Host（默认地址 10.0.0.1/127.0.0.2、端口 1688），提供两种注入模式（System32 落地不同 DLL，见第 3 章），外加"在线 KMS"共三种模式。KMS4K 为 TSForge 变体，自述激活至约公元 4085 年。

**（4）非 KMS 路线（手册 1.2/1.4/1.5 + release notes）**
TSForge 向 SPP 信任存储写入伪造电话激活数据实现"永久"激活（仅限 SPP 托管且支持电话激活的版本，覆盖 Win7~11 含 ESU、Server 2008R2+、Office 2013~2024）；KMS38 通过锁定 KMS 服务器地址（置 127.0.0.2 并加注册表权限）把批量激活延至 2038；HWID 数字激活与微软服务器交互；OEM 路线通过 GRUB4DOS/WindSLIC 在引导阶段模拟 SLIC 表；OHook 针对 Office 2013+ 以 DLL 钩子方式激活（含 Office 365）。另有 $OEM$ 目录生成以支持系统部署阶段（Sysprep/OOBE）自动激活（v63.4.0 修复的就是该场景 bug）。

各激活方式按手册自述口径的客观对照（不含操作步骤）：

| 方式 | 手册自述有效期 | 是否需联网 | 自述适用范围（节选） |
|---|---|---|---|
| HWID 数字许可证 | 永久（重装联网自动恢复） | 是 | Win10/11（LTSC 2021/2024 经转 IoT LTSC 支持） |
| TSForge | 永久（伪造电话激活数据） | 否 | Win7~11 含 ESU、Server 2008R2+、Office 2013~2024（限 SPP 托管） |
| KMS38 | 至 2038 年 | 否 | 所有 Windows 批量版本（含 Server 2016~2025、LTSC/IoT LTSC） |
| KMS | 180 天（可自动续期） | 可离线（内置服务器） | Vista~11、Server 2008~2025、Office 2010~2024 批量授权版 |
| OEM（SLIC 模拟） | 随主板 SLIC | 否 | 主板含 SLIC 的机器（Legacy/UEFI 两条路径） |
| OHook | 永久（Office） | 否 | Win8+ 上的 Office 2013+，含零售版与 Office 365 |

（来源：`HEU_KMS_Activator使用说明手册.pdf` 1.2~1.5 节；表中为该软件文档自述，未经独立验证。）

## 6. 构建与运行

- **构建**：不可构建。仓库无任何源码、工程文件或 CI 配置（tarball 全量成员仅 3 个文件，见第 4 章），二进制构建过程完全黑盒（未确认）。
- **获取**：官方渠道即 GitHub Releases 页（tar/clone 拉不到程序本体）。本分析遵守安全约束：全程未解压、未运行任何下载资产，仅流式读取树内文本/PDF。
- **运行**（仅客观记录手册描述的存在性，不构成操作指导）：程序为单 exe 图形工具，手册记录了静默参数体系（如 `/smart` 组合参数，日志写入 %temp%），并自述建议杀软白名单其续期组件。具体激活操作请参见其官方手册原文，本文不予复述。

## 7. 测试与工程质量

- **测试**：无。仓库无测试目录、无 CI（`.github/` 不存在于 tarball 成员列表）。
- **工程质量信号（仅就发布流程而言）**：
  - changelog 纪律强：54 个 release 全部附中文逐条更新日志，格式统一，多数版本附二进制哈希（MD5/SHA1），可核对（出处：各 release body）。
  - 缺陷修复响应密集：日志中大量"修复激活失败/卡死/闪退"条目（如 v63.3.4 修复输出管道阻塞卡死、v63.3.3 修复 ClipSVC 导致的无响应、v24.4.1Beta 适配 Win11 22483 移除 WMIC），显示对 Windows 版本演化的持续适配。
  - 但 182 个 open issue 长期挂起（gh api），无公开的 issue 处理记录；单维护者，代码不可审计，工程质量无从验证。

## 8. 社区与维护状态

- **维护者结构**：contributors API 返回 1 人（zbezj），单人项目（gh api contributors）。
- **活跃度**：仓库创建于 2019-06-23；Releases API 可见的最早发布为 v24.1.0（2021-07-24）——2019-2021 年间的分发方式不可见（未确认，推测早于 GitHub Release 或未保留）。2021-07 至 2026-07 五年间发布 54 个版本，年均约 10 个，节奏稳定；最近一次 push/发布为 2026-07-06（v64.0.0），截至分析日约 4 个月未更新，但历史上有 2025-03~2025-11 的更长间隔（v50.0.0→v63.3.0 间跳过 8 个月），属正常波动。
- **社区规模**：44,269 星、3,958 fork、182 open issues。无 wiki 内容确认（has_wiki=true 但内容未检查，未确认）。README 无内容、无 LICENSE、无贡献指南——社区参与仅限 issue 反馈与星标，贡献面为零。
- **版本演进的客观时间线**（release body）：2021（v24.x）→ 2022（v25~27，静默参数、Win11 适配）→ 2023（v28~42，全新界面 v30.0.0、OHook Office 永久激活 v40.0.0）→ 2024（v42.x，Office 2024/Server 2025 支持）→ 2025（v50~63.2，TSForge 上线 v50.0.0、KMS4K）→ 2026（v63.3~64，ESU/WNC 适配、IID 更新）。

按年代分段的 release 明细（全部出自 gh api releases body，逐条核对）：

| 年代 | 版本段 | 关键节点（release notes 原文摘要） |
|---|---|---|
| 2021-07~2021-11 | v24.1.0~v24.6.5 | Win11 22483 砍 WMIC 适配（24.4.1Beta）、Office C2R 转 VL 修复、Win10 IoT LTSC 2021 支持（24.5.0）、Set.ini 自定义配置（24.6.4） |
| 2022-07~2022-12 | v25.0.0~v27.0.2 | Win11 版本转换（25.0.0）、ARM64 激活修复（24.6.1/27.0.2）、安全软件误报修复（27.0.0）、WinXP 支持（27.0.2） |
| 2023-02~2023-11 | v28.0.0~v42.0.0 | 全新界面（30.0.0）、hosts 自动修改复原（28.0.0）、Office"盗版受害者"弹窗处理（24.6.3/30.x 延续）、数字门票安装 ID 方法致谢 @asdcorp（40.0.0）、OHook Office 永久激活上线（40.0.0）、Office 2024 Preview 支持（42.0.0） |
| 2024-01~2024-11 | v42.0.1~v42.3.2 | Win Server 2025 KMS/KMS38 支持（42.0.1）、Office 2024 GVLK 更新（42.2.0）、Office 2024 激活信息备份（42.3.1）、KMS 服务器卡死修复（42.3.2） |
| 2025-03~2025-08 | v50.0.0~v63.2.0 | TSForge 上线、基于开源 MAS 3.0（50.0.0）、KMS4K（60.0.0 手册）、OHPP OSPP 支持基于 MAS 3.2（62.0.0）、评估版重置（63.2.0） |
| 2025-11~2026-07 | v63.3.0~v64.0.0 | TSForge 优化与 ESU 默认化（63.3.0/63.4.0）、Sysprep/OOBE 部署修复（63.4.0）、Win11 WNC 支持（63.3.2）、HWID 新 IID 接入 MAS（64.0.0） |

## 9. 亮点与风险

**亮点（客观描述）**
1. 发布流程成熟：双格式资产 + 逐版哈希 + 统一 changelog，54 版连续可追溯（各 release body）。
2. 文档投入罕见地高：两份共 47 页的图文手册随仓库维护，远超同类仓库的文档水平（树内 PDF）。
3. 对 Windows/Office 版本演化响应快：从 Win11 砍 WMIC（v24.4.1Beta）到 Office 2024 密钥（v42.2.0，微软正式发布次日更新）、Win10 ESU（v63.4.0）均有跟进。
4. 长期单人稳定维护五年以上，覆盖 XP~Win11/Server 2025 的极宽版本面（手册 + release notes）。

**风险与局限**
1. **许可合规风险（核心风险）**：本工具的功能是绕过微软激活机制（伪造许可证数据、模拟 KMS Host、SLIC 模拟等），使用它激活未获授权的 Windows/Office 违反微软许可条款（EULA），在多数司法辖区存在法律与合规争议；其手册自带免责声明（声明仅供研究、反对盗版、要求 24 小时内删除、援引《计算机软件保护条例》第十七条），但此类自述免责不改变激活绕过行为本身的授权冲突性质（手册第 2 节免责声明原文）。
2. **二进制不可审计**：无源码、无 LICENSE，安全研究人员与用户都无法验证 exe 行为；手册与 release notes 自述其机制涉及向 System32 复制 DLL/EXE、修改 bootcode、修改 hosts 文件（v28.0.0）、改注册表等与恶意软件行为特征重叠的操作，且手册明示建议加入杀软白名单——无论开发者意图如何，这构成实质的信任负担（多处 release notes 亦自述"修复安全软件误报"）。
3. **单点维护与单点信任**：唯一贡献者，无外部审计，历史版本曾用密码压缩包（Password_HEU168）进一步降低可扫描性（v25~v26 资产命名）。
4. **仓库元数据残缺**：README 空置、无简介、无 LICENSE、版本号跳号（无 29/31~39/43~49），新用户难以从仓库本身获得最低限度的背景信息（tarball + gh api）。
5. **方法论上受制于微软对抗**：其激活方式依赖绕过 SPP/激活服务，随微软封堵随时可能失效——v64.0.0 更新日志自述"数字许可证激活旧 IID 已失效，使用新的 IID"，即该风险的直接证据（release body）；此类工具需要持续追赶，任何长期不更新期都意味着功能衰退。

## 10. 总结评价

zbezj/HEU_KMS_Activator 是一个"仓库不是项目、Releases 才是项目"的极端样本：树内只有一个空 README 和两份用心编写的 PDF 手册，44k 星全部由 Releases 通道里 5MB 级的二进制及其五年 54 版的持续迭代挣来。从纯技术观察角度，它在发布工程（双格式资产、哈希、changelog 纪律、对 Windows 版本演化的快速跟进）和用户文档上做得相当扎实，是研究"二进制分发型仓库如何运营社区"的典型素材；但它的全部功能围绕绕过微软产品激活展开，无源码、无许可证、单点维护，叠加杀软冲突与授权合规问题，决定了它只能作为合规风险案例与技术机制研究对象来看待，不适合任何生产环境引用。对源码研究者而言此仓库无源码可读；对合规与安全分析者而言，它是一个值得记录的、文档完备但完全黑盒的争议性分发仓库。

---
*分析依据：codeload HEAD tarball（流式读取，未解压落盘）+ gh api 元数据/releases/tags/contributors（2026-11-06）。所有 release 引文出自 GitHub Releases body，手册引文出自树内两份 PDF 的文本提取。全程未运行任何下载的可执行文件。*
