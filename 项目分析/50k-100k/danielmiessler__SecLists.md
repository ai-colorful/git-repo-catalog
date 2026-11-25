# danielmiessler/SecLists 项目分析

> 星数 74306（快照 2026-10-24）｜主语言 PHP｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：SecLists 是渗透测试领域事实上的标准词表合集——把安全评估各环节（目录发现、密码爆破、用户名枚举、模糊测试、敏感数据模式匹配等）所需的各类清单集中到一个仓库，让测试人员在拿到新测试机时一次拉取、全程够用（`README.md` 自述）。

| 项目 | 内容 |
|---|---|
| 名称 | danielmiessler/SecLists |
| 星数 | 74306（快照 2026-10-24） |
| 主语言 | GitHub 标注 PHP（仓库主体实为 .txt 词表数据，PHP 仅来自 Web-Shells/Pattern-Matching 中少量 PHP 相关文件） |
| 许可证 | MIT（`LICENSE`，Copyright (c) 2018 Daniel Miessler） |
| 创建时间 | 2012-02-19（gh api repos/danielmiessler/SecLists） |
| 最近推送 | 2026-10-30（gh api，与本次分析仅隔一天） |
| 最新版本 | 2026.2（2026-10-27 发布，日历版本号） |
| 体量 | 6277 个文件 + 156 个目录，解压约 1.97GB（tarball HEAD 条目统计） |
| 贡献者 | 约 317 人（GitHub contributors API 分页计数） |
| 默认分支 | master（gh api） |

## 2. 项目是什么，解决什么问题

SecLists 解决的是渗透测试中一个朴素但高频的痛点：测试过程中随时需要各类"清单"——猜目录要字典、爆破登录要密码表、枚举子域名要子域列表、测 XSS 要 payload 集——这些清单散落在各处、质量参差、格式不一。SecLists 把它们收集到一个地方，按统一格式与分类组织，测试者 `git clone` 一次即可长期使用（`README.md` About 段）。

项目定位是"安全测试伴侣"（the security tester's companion），覆盖用户名、密码、URL、敏感数据模式、模糊测试 payload、web shell 等类型。它本身不是工具、没有可执行主程序，而是一个被工具消费的数据库：`funding.json` 自述它是 Burp Suite、OWASP ZAP、FFuF 等工具词表的"de facto standard"（事实标准，工具集成情况未逐一验证）。

从规模看（tarball HEAD 条目统计），全仓 6277 个文件约 1.97GB，其中密码类词表独占约 661MB、发现/子域名类约 509MB、模糊测试 payload 约 382MB。单个文件如 `Passwords/Common-Credentials/Pwdb_top-10000000.txt`（94MB）、`Usernames/xato-net-10-million-usernames.txt`（85MB）都是千万级条目的生产级字典。

项目由 Daniel Miessler 于 2012 年创建，是目前安全社区资历最老、引用最广的词表仓库之一，已进入 Kali Linux 与 BlackArch 官方软件源（`README.md` Install 段：`apt install seclists` / `pacman -S seclists`），这也是它能累积 74306 星、25178 fork 的重要渠道原因。

## 3. 技术栈与依赖

内容型仓库，"技术栈"极薄，主要由三部分构成：

- **数据本体**：纯文本 .txt 词表（占绝对多数），少量 .csv（如 `Ai/LLM_Testing/Ethical_and_Safety_Boundaries/` 下的 jailbreak prompts）、.tar.gz（rockyou 原始包，`Passwords/Leaked-Databases/`）、.md（目录级文档）。
- **自动化脚本**：Python 3 与 shell/awk。唯一读到的第三方依赖是 `requests`（`.bin/wordlist-updaters/updater.py` import 头部）；`.bin/` 下另有十余个独立小脚本（去脏话词、OS 名变形、XML 解析、Trickest 补丁器等，见 `.bin/README.md`）。
- **CI**：GitHub Actions，`.github/workflows/` 下 9 个工作流（1 个 readme 徽章更新器、1 个远程词表更新器、6 个衍生词表拼接器、1 个校验器（已停用））。

无构建系统、无运行时依赖、无包管理清单——词表即数据，clone 即安装。

## 4. 目录结构与代码组织

顶层目录（文件数与体积来自 tarball HEAD 条目统计）：

| 目录 | 文件数 | 体积 | 职责 |
|---|---|---|---|
| Fuzzing | 4657 | 382MB | 模糊测试 payload：XSS、SQLi、LFI、数据库、日期、User-Agent 等（User-Agents 子目录独占 4506 个文件） |
| Passwords | 723 | 661MB | 密码字典：通用凭证、泄露数据库（rockyou）、默认凭证、哈希 |
| Discovery | 435 | 509MB | 内容/基础设施发现：Web-Content 目录爆破字典、DNS 子域列表 |
| Miscellaneous | 272 | 259MB | 杂项：各国词汇表、安全问题答案、脏话列表等 |
| Web-Shells | 61 | 237KB | web shell 样本集（laudanum-1.0、Vtiger、FuzzDB） |
| Payloads | 37 | 57MB | 文件型 payload：Zip-Bombs、Zip-Traversal、Flash、图片 |
| .bin | 29 | 4.2MB | 自动化与清洗脚本（唯一的"代码"目录） |
| Usernames | 18 | 92MB | 用户名枚举字典（xato 千万级用户名） |
| Ai | 13 | 5.2MB | LLM 安全测试语料：越狱 prompt、偏见测试、数据泄露探测 |
| .github | 14 | 43KB | CI 工作流、issue/PR 模板 |
| Pattern-Matching | 11 | 54KB | 敏感数据/代码审计用 grep 模式 |
| （根目录） | 7 | 64KB | README、CONTRIBUTING、CONTRIBUTORS、LICENSE、funding.json、.gitattributes、.gitignore |

组织方式评述：这是典型的"分类即架构"仓库——没有模块、没有接口，目录树就是唯一的索引体系。约 49 个目录配有自己的 `README.md`，逐一说明所含词表的用途与来源（此约定写进了 `CONTRIBUTING.md`，并指定 `Discovery/Web-Content` 的 README 作为范本）。`.bin/` 是唯一承载逻辑的目录，与数据严格分离；根目录极干净，7 个文件全部是治理类文档。

## 5. 核心架构与关键实现（资源组织与收录机制）

按内容型仓库视角，本项目的"核心实现"不是代码而是三套机制：

**（1）声明式上游同步管线。** 词表更新不靠人工搬运，而是 `.bin/wordlist-updaters/sources.json` 声明任务清单，每条任务定义来源 URL/仓库、输出路径与刷新频率，由 `.bin/wordlist-updaters/updater.py`（Python 3 + requests）执行，GitHub Action 每小时检查是否到达刷新条件（`sources.json` 示例条目）：

```json
[
    {
        "name": "Jwt secrets update",
        "type": "file",
        "source": "https://raw.githubusercontent.com/wallarm/jwt-secrets/master/jwt.secrets.list",
        "output": "Passwords/scraped-JWT-secrets.txt",
        "post_run_script": "",
        "frequency": "3d"
    }
]
```

支持 `file`（单文件抓取）与 `git_dir`（整个上游仓库目录）两种源类型，并可挂 `post_run_script` 后处理（如 `.bin/trickest-patcher.py` 对 Trickest 词表打补丁）。任务状态记录在 `status.json`，禁止人工在 PR 中修改（`.bin/wordlist-updaters/README.md`）。

**（2）衍生词表自动拼接。** 组合类词表由 Action 在上游文件变更时自动重算，例如 `Discovery/Web-Content/combined_words.txt` 由 8 个源词表拼接去重生成（`.github/workflows/wordlist-updater_combined_words.yml`）：

```bash
cat common.txt raft-small-words* raft-medium-words* big.txt raft-large-words* \
  | awk '! seen[$0]++' > combined_words.txt
```

以 `github-actions[bot]` 身份自动提交推送。README 中的仓库体积、克隆耗时徽章同样由 bot 维护（`.github/workflows/readme-updater.yml`，受仓库变量 `RUN_UPDATERS == 1` 门控，配套脚本 `.bin/get-and-patch-readme-repository-details.py`）。

**（3）收录质量护栏。** `CONTRIBUTING.md` 定义了硬性规范：词表内容不得带前导斜杠（保证工具兼容）、必须去重、文件内不得写注释（说明放入目录 README）、URL 参数用占位符；目录名用 Train-Case。`wordlist-validator.yml.disabled` 原本在每次 push 时校验"新增行是否以 / 开头"，当前处于停用状态（文件名后缀 `.disabled`）。另有两条约束值得注意：禁止上传完整数据泄露（只允许不带 PII 的密码部分）；禁止以任何方式使用 AI 生成贡献（理由是 LLM 幻觉与审阅成本）。

## 6. 构建与运行

无构建环节，以下获取方式均来自 `README.md`：

```bash
# 推荐：浅 clone（跳过 14 年历史，最快）
git clone --depth 1 https://github.com/danielmiessler/SecLists.git

# 完整 clone：约 9m49s @50Mb/s（README 徽章由 CI 自动标注）
git clone https://github.com/danielmiessler/SecLists.git

# 发行版包管理器（已进入官方源）
apt -y install seclists          # Kali Linux（kali.org/tools/seclists）
sudo pacman -S seclists          # BlackArch

# zip 直下
wget -c https://github.com/danielmiessler/SecLists/archive/master.zip -O SecList.zip
```

消费方式是"路径引用"：词表被 Burp Intruder、ZAP fuzzer、ffuf（`-w` 参数指向本地 SecLists 路径）等工具直接加载（`funding.json` 自述定位；具体工具集成未逐一验证）。`.bin/` 下的脚本供维护者与用户自行清洗/变形词表使用，不属于运行时依赖。

## 7. 收录机制与内容质量保障（内容型特例）

- **收录范围与来源**：聚合上游安全社区的公开词表并强制署名。可考的来源包括 RAFT 词表、SkullSecurity（Ron Bowes）、FuzzDB（Adam Muntner）、Clarkson 大学密码研究、Trickest、xato、n0kovo 等（`CONTRIBUTORS.md`、`.bin/wordlist-updaters/sources.json`、各目录 README 的 Source 标注）。
- **收录门槛**：见第 5 章第（3）节——统一格式（无前导斜杠、无行内注释、去重、占位符）、逐词表写目录 README（说明用途 `Use for:` 与来源 `Source:`）、新内容走 PR 或 issue 双通道（`CONTRIBUTING.md`）。
- **更新机制**：`sources.json` 驱动的每小时 Action 巡检 + 衍生词表自动重算，使第三方上游词表在本仓库内保持同步，这是它区别于静态"awesome 清单"的关键：内容是活的。
- **治理与署名**：46KB 的 `CONTRIBUTORS.md` 逐条致谢来源作者/项目；`funding.json`（fundingjson.org v1.1 规范）声明募集资金用于 CI/CD 基础设施与自动化验证管线的运行成本。
- **类目演化**：仓库随威胁面扩张而扩类，最新例子是 `Ai/LLM_Testing/`——包含越狱 prompt 语料（注明来源于 ACM CCS 2024 论文 "Do Anything Now" 的官方仓库）、偏见测试、数据泄露与记忆召回测试集（`Ai/LLM_Testing/README.md`），表明其定位已从 Web 渗透字典扩展到 AI 安全评测语料。

## 8. 社区与维护状态

- **活跃度**：2012-02-19 创建，最近推送 2026-10-30（gh api），13 年持续维护、基本无断档；open issues 仅 9 个，响应及时。
- **Release 节奏**：日历版本号（年.序号），近年为 2024.4 → 2025.1/2025.2/2025.3 → 2026.1（3 月）→ 2026.2（10 月），每年 2-4 个版本（gh api releases 列表）。
- **维护者结构**：创始人 Daniel Miessler，现任主维护者 Ignacio Portal（ItsIgnacioPortal，2021 年起，README 与 funding.json 均以其为资助主体），联合维护者 Jason Haddix 与 g0tmi1k（`README.md`）。
- **贡献者**：约 317 人（contributors API 分页末页号推算），以词表捐赠型贡献为主。
- **未确认**：PR 平均合并时长、issue 首响时间、具体赞助金额（tarball 无 git 历史，未做逐项统计）。

## 9. 亮点与风险

**亮点**

1. **事实标准 + 发行版渠道**：进入 Kali/BlackArch 官方软件源，被主流安全工具引用（`funding.json` 自述，README 提供发行版安装页链接），分发位极难复制。
2. **内容型仓库少见的自动化密度**：声明式上游同步、衍生词表自动拼接、README 徽章自更新（`sources.json` + 9 个工作流），把"清单维护"做成了数据工程。
3. **13 年沉淀的分类体系与格式规范**：顶层 10 类、156 个目录、约 49 个目录级 README，格式约定统一，检索成本低。
4. **强署名文化**：`CONTRIBUTORS.md` 逐条溯源，合规使用上游内容的标杆做法。
5. **随威胁面演化**：新增 Ai/LLM 测试类目，保持与行业同步。

**风险 / 局限**

1. **体量负担**：工作树约 1.97GB、GitHub 仓库计 3.6GB（gh api size 字段），完整 clone 十分钟级；官方自己也只推荐 `--depth 1`。低带宽环境建议按目录用 raw 链接按需取用。
2. **杀软误报是常态而非意外**：`README.md` 结尾自述"下载本仓库大概率触发杀毒软件报警"；本次分析流水线也按预案采用 tar 流式读取、未整体落盘（规避 Windows Defender 实时隔离）。主要触发源是 `Web-Shells/`（laudanum、Vtiger 等真实 web shell 样本集合）与部分 Payloads。
3. **合规边界必须自知**：仓库定位为授权安全测试用途；`CONTRIBUTING.md` 明确禁止上传完整数据泄露与任何可回溯到个人的 PII，但 rockyou 等泄露密码表仍属敏感内容，企业环境引入前需过合规审查，不得存放于对外服务器（README 亦提示 LFI 风险）。
4. **元数据噪音**：GitHub 语言标签显示 PHP，与实际内容（纯文本数据 + 少量运维脚本）严重不符，做统计或选型时应忽略该字段；仓库 homepage 指向 OWASP IoT 项目页，亦属历史遗留（gh api 字段）。

## 10. 总结评价

SecLists 是"把一件事做满 13 年"的典型：技术上没有任何难度，但凭借先发的聚合定位、严格到近乎苛刻的格式与署名规范、以及内容型项目里罕见的自动化维护管线，成长为渗透测试词表的事实标准，并借 Kali/BlackArch 官方源完成了对目标人群的全量分发。74306 星对应的不是代码价值，而是"数据基础设施"的网络效应——工具默认引用它、教程默认使用它，后来者几乎无法绕开。它适合三类人：安全从业者（直接当字典库用）、安全工具作者（把 SecLists 路径作为默认词表集成）、以及研究开源数据型项目如何做治理与自动化维护的人（`.bin/` + `.github/workflows/` 这套"给数据仓库上工程"的实践非常值得抄）。
