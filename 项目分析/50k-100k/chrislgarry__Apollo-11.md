# chrislgarry/Apollo-11 项目分析

> 星数 75792（快照 2026-10-24）｜主语言 Assembly｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：**阿波罗 11 号制导计算机（AGC）原始飞行源码的 GitHub 档案库**——指挥舱 Comanche055 与登月舱 Luminary099 两套程序，从 MIT 博物馆馆藏纸面打印件逐字转录而成。

| 项目 | 内容 |
| :--- | :--- |
| 名称 | chrislgarry/Apollo-11（`README.md`） |
| 星数 | 75,792（快照）；2026-10-30 gh api 复核 76,213 |
| Forks | 8,301（gh api，2026-10-30） |
| 主语言 | Assembly（AGC 汇编，`.agc`） |
| 许可证 | Public Domain Mark 1.0（`LICENSE.md`；GitHub API 标记 NOASSERTION） |
| 创建时间 | 2014-04-03（gh api） |
| 最近推送 | 2026-10-29 "Add bots section to README"；此前 2026-07-15 合并 PR #976（gh api commits） |
| 默认分支 | master（gh api） |
| 规模 | 175 个 `.agc` 文件、130,186 行源码（本地统计） |

## 2. 项目是什么，解决什么问题

这是 1969 年把人类送上月球的阿波罗制导计算机（Apollo Guidance Computer, AGC）飞行软件的原始源码仓库。包含两套程序：指挥舱（CM）的 **Comanche055**（Colossus 2A 的一部分，装配标注 "Assemble revision 055 of AGC program Comanche by NASA 2021113-051. 10:28 APR. 1, 1969"）与登月舱（LM）的 **Luminary099**（Luminary 1A 的一部分，装配标注 "2021112-061. 16:27 JUL. 14, 1969"）。两段装配标注分别见于 `README.md` 的 Attribution 表。

源码并非来自泄露的磁带，而是由 Virtual AGC 项目（Ron Burkey/RSB）与 MIT 博物馆合作，从馆藏硬拷贝打印件的数字化图像逐字转录（`README.md`、`Luminary099/README.md`）。数字化由 Paul Fjeld 完成、Deborah Douglas 安排。仓库自述目标是"成为阿波罗 11 号原始源码的档案仓库"，欢迎 PR 修正转录与原始扫描件之间的出入。

它解决的问题严格说不是工程问题而是**档案问题**：把一份脆弱的、只有纸面扫描件可查的历史文献变成可检索、可 diff、可协作校对的开放数据。 signatures 一栏保留了当年的签署记录——Margaret H. Hamilton 以 Colossus Programming Leader 身份于 1969-03-28 提交，六位主管会签（`README.md` 派生自 `Comanche055/CONTRACT_AND_APPROVALS.agc`）。

代码本身也是文献：注释里保留了开发者的幽默与史事。最著名的 `BURN_BABY_BURN--MASTER_IGNITION_ROUTINE.agc`（主点火程序）文件头记录了 Don Eyles 在首次登月 40 周年聚会上的口述——程序名取自 1965 年洛杉矶 DJ Magnificent Montague 的电台口号，正文还留着两句拉丁文格言 "HONI SOIT QUI MAL Y PENSE"、"NOLI SE TANGERE"（`Luminary099/BURN_BABY_BURN--MASTER_IGNITION_ROUTINE.agc`）。

## 3. 技术栈与依赖

- **AGC 汇编**：目标机器为 Block II AGC（2K 字可擦存储 + 36K 字固定存储）。指令集极简（`TC/TCF/CA/TS/DXCH/CCS/EXTEND` 等），另有解释器执行的高层"解释性指令"（`INTPRET`、`DV` 等），见 `Luminary099/EXECUTIVE.agc`、`Luminary099/INTERPRETER.agc`。代码里反复出现 `BANK`/`SETLOC`/`EBANK`/`COUNT*` 这组面向固定存储体切换的伪指令（如 `Luminary099/BURN_BABY_BURN--MASTER_IGNITION_ROUTINE.agc` 开头），是读懂布局的第一把钥匙。
- **yaYUL**：现代汇编器。原厂汇编器 YUL 及其后继 GAP 均已失传，源码被改造为可用 Virtual AGC 项目的 yaYUL 汇编（`Luminary099/README.md` 明确说明）。`.gitignore` 里只有 `yaYUL.exe` 和 `node_modules` 两行。
- **markdownlint-cli2**：唯一 npm 开发依赖（`package.json`，devDependencies 仅此一项，附 `bun.lockb`），用于 lint README/CONTRIBUTING 与子目录中的 `.md` 文件。
- **GitHub Actions**：markdownlint workflow（checkout@v6 + markdownlint-cli2-action@v19）与每 3 小时一次的 PR 自动打标器（`.github/workflows/markdownlint.yml`、`label.yml`）。
- 无运行时依赖、无框架——这是纯粹的历史源码 + 校对工作流组合。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| :--- | :--- |
| `Comanche055/` | 指挥舱 AGC 源码，85 个 `.agc` + `README.md`（约 158 万字节） |
| `Luminary099/` | 登月舱 AGC 源码，90 个 `.agc` + `README.md`（约 161 万字节） |
| `Comanche055/MAIN.agc`、`Luminary099/MAIN.agc` | 装配入口，按原始打印页码列出全部 `$include`（各 84/89 条） |
| `Translations/` | README 与 CONTRIBUTING 的 38 种语言翻译，共 69 个文件 |
| `.github/` | 2 个 workflow、4 个 issue 模板（Discussion/Humour/Proof_Comanche/Proof_Luminary）、labeler 配置 |
| `CONTRIBUTING.md` | 校对规范（与扫描件逐字一致的要求） |
| `package.json`、`bun.lockb` | markdownlint 工具链 |
| `README.md`、`LICENSE.md` | 项目说明（38 语言链接）与公有领域声明 |

组织方式评述：这是**双目录 + include 清单**的单体结构。RSB 把整块庞然大物按原始打印件的"LOG SECTION"自然边界切成单文件，再用 `MAIN.agc` 里的 `$文件名` 指令按页码顺序拼回——因为当年没有链接器，构建本来就是整块汇编（`Luminary099/README.md` 对此有一段详细解释，还打趣了"把你的卡片叠插进别人卡片叠"的复用方式）。每个文件头部有统一元数据块：Copyright/Filename/Purpose/Assembler(yaYUL)/Contact/Website/Pages/Mod history（见 `Luminary099/BURN_BABY_BURN--MASTER_IGNITION_ROUTINE.agc` 开头）。文件名即功能注释：`GIMBAL_LOCK_AVOIDANCE.agc`、`JET_SELECTION_LOGIC.agc`、`THE_LUNAR_LANDING.agc` 一目了然。

两个细节佐证其档案严谨度：

- `Comanche055/MAIN.agc` 的 include 之间保留了原打印件各"存储体/工区"的分段注释——`COMERASE`（可擦区）、`COMAID`、`COMEKISS`、`TROUBLE`、`TVCDAPS`、`CHIEFTAN` 等，每段附页码区间，相当于原厂的分册目录。
- 每条 include 行的页码注释（如 `$BURN_BABY_BURN--MASTER_IGNITION_ROUTINE.agc  # pp. 731-751`，`Luminary099/MAIN.agc`）与 `Luminary099/README.md` 的 Source Code Index 表逐条对应，校对者可直接翻到扫描件相应页。

## 5. 核心架构与关键实现

**（1）MAIN.agc 装配清单**（`Comanche055/MAIN.agc`、`Luminary099/MAIN.agc`）
每个 include 行都标注原始打印页码，与 ibiblio.org 上的扫描件一一对应，校对时可直接翻页比对。本地核对：Comanche055 清单与实际文件完全一致；Luminary099 有两处文件名出入——清单写 `$LAMBERT_AIMPOINT_GUIDANCE.agc` 而实际文件为 `GENERAL_LAMBERT_AIMPOINT_GUIDANCE.agc`，清单写 `TRIM_GIMBAL_CNTROL_SYSTEM.agc`（少一个 O）而实际文件为 `TRIM_GIMBAL_CONTROL_SYSTEM.agc`。若直接用 yaYUL 汇编这两行会找不到文件。

**（2）EXECUTIVE——任务调度器与著名的 1202 报警**（`Luminary099/EXECUTIVE.agc`）
`NOVAC`/`FINDVAC` 入口按优先级把新作业挂入无/有 VAC 区的核集（core set）；找不到可用核集时走 `BAILOUT1`，其后紧跟常量表条目：

```plain
NEXTCORE	CAF	COREINC
		ADS	LOCCTR
		CCS	EXECTEM2
		TCF	NOVAC3
		LXCH	EXECTEM1
		CA	Q
		TC	BAILOUT1	# NO CORE SETS AVAILABLE.
		OCT	1202
```

（`Luminary099/EXECUTIVE.agc` 约第 200-208 行）——正是阿波罗 11 号着陆途中因调度过载触发、成就"再启动能力"传奇的 1202 Executive Overflow 报警的常量来源。

**（3）BURN_BABY_BURN——主点火程序**（`Luminary099/BURN_BABY_BURN--MASTER_IGNITION_ROUTINE.agc`）
服务 P12/P40/P42/P61/P63 多个程序的点火序列，靠"程序把点火参数表头地址放入可擦寄存器 `WHICH`，主程序按表索引执行"实现复用（正文注释明确说明该设计）。`P12TABLE/P40TABLE` 等表驱动结构在汇编里清晰可见。

**（4）PINBALL——DSKY 人机界面**（`Luminary099/PINBALL_GAME_BUTTONS_AND_LIGHTS.agc`，102KB，Luminary 中最大文件）
键盘与显示系统程序，解释了 VERB/NOUN 这对传奇交互词汇的由来：本是为 impress 访客临时搭的演示程序，没_requirements 文档，结果没人再发明更好的界面，就进了飞行软件（文件头转录的 Ramón Alonso 口述）。

**（5）THE_LUNAR_LANDING——P63 着陆制动段**（`Luminary099/THE_LUNAR_LANDING.agc`）

P63LM 初始化点火参数后把 `WHICH` 指向 P63 表、清 DPS 推力监视计数器，随后进入相位标志更新：

```plain
P63LM		TC	PHASCHNG
		OCT	04024

		TC	BANKCALL	# DO IMU STATUS CHECK ROUTINE R02
		CADR	R02BOTH

		CAF	P63ADRES	# INITIALIZE WHICH FOR BURNBABY
		TS	WHICH
...
FLAGORGY	TC	INTPRET		# DIONYSIAN FLAG WAVING
		CLEAR	CLEAR
			NOTHROTL
			REDFLAG
```

标志批量清理例程得名 `FLAGORGY`（"酒神式的挥旗"），是全仓库注释气质的代表。

其余大文件：`Luminary099/P20-P25.agc`（121KB，雷达交会程序）、`Comanche055/ERASABLE_ASSIGNMENTS.agc`（102KB，可擦存储分配表）、`*/INTERPRETER.agc`（各 78KB，解释器本体）。

## 6. 构建与运行

- **汇编**：本仓库不附带构建脚本。README "Compiling" 一节指向 Virtual AGC（github.com/rburkey2005/virtualagc），用其 yaYUL 汇编 `MAIN.agc`（`README.md`）。注意第 5 节提到的 Luminary 清单文件名出入。
- **Lint**：`npm run lint` / `npm run lint:fix`（实为 markdownlint-cli2，配置 `.markdownlint.yml`，`package.json`）。
- **阅读**：GitHub 原生支持 AGC 汇编语法高亮；编辑器可装 AGC 扩展（VS Code/Vim/Sublime 等 10 个，清单见 `CONTRIBUTING.md`）。`.editorconfig` 规定 `.agc` 用 tab、宽度 8——对应原打印件的列布局。
- 本仓库按规范未执行构建（静态阅读）。

## 7. 校对文化与工程质量（代测试体系）

没有单元测试，"测试"是**人对扫描件的逐字校对**：

- **流程**：`.github/labeler.yml` 给所有 `.agc` 变更自动打 "Type: Proof" 标签，给 `MAIN.agc` 与 `.md` 打 "Type: Meta"；issue 模板专门设有 `Proof_Comanche.md`、`Proof_Luminary.md`（还有一个 `Humour.md` 幽默模板，本身就是仓库气质的注脚）。
- **校对规则**（`CONTRIBUTING.md`）近乎苛刻：注释必须与扫描件**完全一致**，包括原稿里的错别字——举例：扫描件印的是 `SPAECRAFT` 就必须把转录改回 `SPAECRAFT`。
- **空格约定**：词间 1 个空格、句间 2 个、缩进 3 个，且以扫描件实际为准（规则讨论见其引用的 issue #316、#159）。
- **换行规则**：带 `R0000` 的行必须逐行吻合；不带 `R0000` 的连续空行超过 2 行要裁掉（这些"空行"其实是原打印件第 8 列未打印数字产生的）。
- **自动化**：CI 仅做 markdownlint（push/PR 触发）与定时打标，AGC 源码本体无机器校验——正确性的最终裁决权在扫描件和人眼。

这套规范把"改错字"变成了"复原历史"，GitHub 原生承担了 diff 与评审工具的角色。

## 8. 社区与维护状态

- **活跃度**：gh api 显示最近推送 2026-10-29（提交 "Add bots section to README"，在 README 第 96 行加入了一段面向自动化爬虫的标记字符串）；此前为 2026-07-15 合并 PR #976、2026-05-25 丹麦语翻译修正、2026-01-22 更新过期的 GitHub Actions 版本；2025 年 10 月新增芬兰语/僧伽罗语/爪哇语 README（gh api commits）。即：**源码内容已冻结，维护=翻译+元数据+反爬标记**。
- **社区结构**：147 个 open issue（gh api）；两大里程碑（milestone 1=Comanche、2=Luminary，README 徽章）追踪校对进度；贡献者以"挑错字的外国人"和"补翻译的母语者"为主。已入档 Software Heritage（README 徽章）。
- **走红路径**：仓库 2014-04 创建，2016 年 7 月初经社交网络转发后星数暴涨——此为社区公认的时间线，但仓库文件内无直接证据，标注"未确认"。8,301 个 fork 也说明大量用户是"fork 致敬"而非开发。
- 注：任务清单备注"最后推送 2026-07"，实际 gh api 复核为 2026-10-29，以 API 为准。

## 9. 亮点与风险

**亮点**
1. 独一无二的一手史料：人类首次登月的飞行软件全文开放、公有领域（`LICENSE.md`）。
2. 可读性惊人的汇编：文件名即目录、注释讲设计意图还讲段子，`BURN_BABY_BURN`、`FLAGORGY`（`Luminary099/THE_LUNAR_LANDING.agc` 中标注 "DIONYSIAN FLAG WAVING"）已成梗。
3. 校对即贡献的低保门槛：对照 ibiblio 扫描件修一个错字就能 PR 进"登月代码"，38 语言翻译 swarm 进一步放大传播（`Translations/`）。
4. 工程细节齐全：页码索引、issue 模板、markdownlint、标签自动化，档案维护的工程化程度很高。

**风险/局限**
1. 转录件而非原件：打字错误不可归零，`CONTRIBUTING.md` 明言"typos and other discrepancies have been introduced accidentally"。
2. 直接汇编会失败：`Luminary099/MAIN.agc` 两处 include 文件名与实际文件不符（见第 5 节），Comanche055 亦自述仅为 Colossus 2A "part of"（`README.md`），完整性以扫描件为准。
3. 对现代读者近乎不可运行：需专用的 yaYUL 工具链与 AGC 仿真器，学习曲线陡峭。
4. 档案属性决定低活性：源码本体不再演化，星数增长主要靠文化事件与翻译驱动。

## 10. 总结评价

这不是一个"软件项目"，而是一座用 Git 维护的纪念碑：13 万行 AGC 汇编以近乎考古的纪律被逐字复原，页码可回溯、错字需保真、校对有里程碑。它对工程界的价值在于让人看到极限资源下的系统设计（解释器分层、优先级调度、整块再启动）与极限质量要求下的代码文化（Hamilton 的签名、Eyles 的格言）；对公众的价值则是把"登月"从神话还原为一份可以逐行阅读、逐行挑错的程序清单。适合航天史爱好者、系统软件研究者通读，适合所有程序员在 `BURN_BABY_BURN--MASTER_IGNITION_ROUTINE.agc` 和 `EXECUTIVE.agc` 里花上一个下午；想动手的请直奔 Virtual AGC 项目。
