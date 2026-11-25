# swisskyrepo/PayloadsAllTheThings 项目分析

> 星数 81992（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：Web 应用安全领域的渗透测试 Payload 与绕过技术速查手册（cheatsheet 集合），按漏洞类型分章组织"方法 + 载荷 + 工具 + 实验室 + 参考文献"，是安全研究/渗透测试社区最常被引用的参考资源之一。

| 项目 | 内容 |
| --- | --- |
| 名称 | swisskyrepo/PayloadsAllTheThings |
| 星数 / Fork | 81,992 / 17,542（gh api，2026-10-31） |
| 创建时间 | 2016-10-18（gh api） |
| 最近推送 | 2026-10-30（gh api，分析前一天仍活跃） |
| 默认分支 | master |
| 语言 | 文档为主（Markdown 142 个文件、约 2.04 万行），附带 11 个 Python 辅助脚本 |
| 许可证 | MIT（LICENSE） |
| 主页 | https://swisskyrepo.github.io/PayloadsAllTheThings/（mkdocs-material 构建的在线版） |
| 贡献者 | 约 325 人（gh api contributors 分页推算） |
| 类型 | 内容型仓库：方法论与 Payload 索引，无业务代码架构 |
| 体量 | 452 个文件、142 个 Markdown、11 个 Python 脚本（tarball 实测）；GitHub 侧仓库约 23MB（gh api size=23349KB） |
| 主题标签 | payloads、penetration-testing、pentest、bugbounty、cheatsheet、redteam、privilege-escalation、bypass、vulnerability 等 17 个（gh api topics） |

## 2. 项目是什么，解决什么问题

PayloadsAllTheThings（简称 PATP）解决的是一个很具体的问题：渗透测试与漏洞研究者在实战中需要在几十秒内找到"某类漏洞有哪些已验证的利用手法和载荷写法"，而这些知识散落在无数博客、Paper 和工具文档里。该项目把 Web 应用安全中常见漏洞类型（SQL 注入、XSS、XXE、SSRF、反序列化、SSTI、JWT、文件上传等）逐一分章，每章沉淀成一份结构化速查文档（README.md 开篇）。

每个技术章节遵循统一的内容契约（见 CONTRIBUTING.md 与 `_template_vuln/README.md`）：**Tools**（相关工具清单）→ **Methodology**（利用方法论与可复制的载荷片段）→ **Labs**（Root Me / PortSwigger / HackTheBox 等配套练习）→ **References**（带作者、日期的外部文献）。以 `SQL Injection/README.md` 为例，其小节覆盖入口点探测、DBMS 识别、认证绕过、UNION/Error/Blind/Stacked/Polyglot/Routed/Second-Order 注入，直到通用 WAF 绕过，形成了从发现到绕过防御的完整路线图。

项目的独特定位是"武器化的知识索引"：它不重复造工具，而是把散落的载荷与手法按攻击面归档，并要求所有载荷脱敏（见第 7 章），使其同时可用于学习与红队作业前的快速备课。README.md 明确接受社区贡献，"Feel free to improve with your payloads and techniques!"，因此内容随社区实战经验持续演进。

除 Web 主体外，`Methodology and Resources/` 目录（33 个文件）横向扩展到内网与平台化主题：Active Directory 攻击、AWS/Azure 云渗透、Docker/Kubernetes 容器逃逸、Windows/Linux 提权、Cobalt Strike/Metasploit 速查等，使项目从"Web 漏洞清单"长成了"渗透测试全流程参考库"。仓库还维护 `_LEARNING_AND_SOCIALS/`（BOOKS/TWITTER/YOUTUBE 三份学习资源精选），强化其作为学习入口的生态位。

使用路径有三条（均来自仓库文件）：直接在 GitHub 上按目录跳读（README.md 的章节即目录名）；访问 mkdocs 构建的在线站点，享受全文搜索与暗色主题（mkdocs.yml）；或把各章 `Intruder/` 下的载荷文件直接导入 Burp Intruder 用作模糊测试字典（README.md、CONTRIBUTING.md 中对 Intruder 文件的定义）。

## 3. 技术栈与依赖

作为内容型仓库，技术栈服务于"文档的编写、校验与发布"：

- **内容本体**：Markdown（142 个 `.md`，共约 20,373 行），含代码块、表格、目录锚点。
- **站点构建**：MkDocs + mkdocs-material 主题（`mkdocs.yml`），启用暗/亮切换、搜索建议/分享、代码复制等 material 特性；部署时额外安装 `mkdocs-git-revision-date-localized-plugin`、`mkdocs-git-committers-plugin`、`mdx_truly_sane_lists`（`.github/workflows/mkdocs-build.yml`）。
- **质量校验**：markdownlint-cli2（配置 `.github/.markdownlint.json`），通过 GitHub Action 与 pre-commit 双通道执行；pre-commit 钩子用 Docker 容器 `davidanson/markdownlint-cli2` 运行（`.pre-commit-config.yaml`）。
- **辅助脚本**：11 个 Python 文件，均为章节配套的利用辅助/PoC 工具，如 `File Inclusion/Files/LFI2RCE.py`、`Server Side Request Forgery/Files/ip.py`、`Upload Insecure Files/CVE FFmpeg HLS/gen_xbin_avi.py`、`Web Sockets/Files/ws-harness.py`（find 统计）。
- **无运行时依赖**：没有 package.json / requirements.txt，仓库本身不是可安装的软件包，Python 版本仅在 CI 中取 `3.x`。

## 4. 目录结构与代码组织

顶层共 66 个目录（`ls -d */`），其中 63 个为漏洞技术章节（每个漏洞类型一个顶层目录），3 个为机制性目录：

| 目录 | 职责 |
| --- | --- |
| `<漏洞类型>/`（63 个，如 SQL Injection、XSS Injection、XXE Injection、File Inclusion、Insecure Deserialization、Upload Insecure Files 等） | 每个目录一章：README.md 为主文档，Images/ 存配图，Files/ 存引用的脚本与样本，Intruder/ 存 Burp Intruder 载荷文件（README.md、CONTRIBUTING.md 约定） |
| `Methodology and Resources/` | 33 个横向方法论文档：AD 攻击、云（AWS/Azure）、容器（Docker/K8s）、Windows/Linux 提权与持久化、Reverse Shell/Cobalt Strike/Metasploit 速查等 |
| `_LEARNING_AND_SOCIALS/` | BOOKS.md、TWITTER.md、YOUTUBE.md 学习资源精选 |
| `_template_vuln/` | 新章节模板（仅 README.md），规定 Tools/Methodology/Labs/References 四段式骨架 |
| `.github/` | CI workflow、markdownlint 配置、mkdocs overrides、赞助商素材、FUNDING.yml |
| 根文件 | README.md、CONTRIBUTING.md、DISCLAIMER.md、LICENSE、mkdocs.yml、custom.css |

63 个技术章节按攻击面可粗分为几族（顶层目录清单）：

| 族 | 章节示例 |
| --- | --- |
| 注入类 | SQL/NoSQL/LDAP/XPATH/XSLT/LaTeX Injection、Command Injection、GraphQL Injection、SSTI、Server Side Include Injection、CSV Injection、CSS Injection、CRLF Injection、Prompt Injection |
| 客户端与浏览器 | XSS Injection、DOM Clobbering、Clickjacking、Tabnabbing、XS-Leak、Headless Browser |
| 服务端请求与反序列化 | SSRF、XXE Injection、Insecure Deserialization、Java RMI、ORM Leak |
| 认证与会话 | JSON Web Token、OAuth Misconfiguration、SAML Injection、Account Takeover、Type Juggling、Mass Assignment、Insecure Direct Object References |
| 文件与上传 | File Inclusion、Directory Traversal、Upload Insecure Files、Zip Slip |
| 基础设施与配置 | CORS Misconfiguration、Request Smuggling、Web Cache Deception、Reverse Proxy Misconfigurations、DNS Rebinding、Dependency Confusion、Prototype Pollution、Race Condition |
| 其他 | CVE Exploits、Denial of Service、Regular Expression、Web Sockets、Hidden Parameters、Virtual Hosts、Business Logic Errors 等 |

组织方式评述：典型的"目录即章节"扁平结构，没有 monorepo/多包概念，也没有构建产出的代码模块。规模靠横向加目录（新增漏洞类型）与纵向加深（每章内多个细分 md，如 SQL Injection 下还有 MySQL Injection.md、SQLmap.md）增长。文件最多的章节是 Upload Insecure Files（116 个文件）、XSS Injection（61）、SQL Injection（34），体量最大的单文件为 `SQL Injection/MySQL Injection.md`（36KB）、`SQL Injection/README.md`（33KB）、`XSS Injection/README.md`（29KB）。

## 5. 核心架构与关键实现

对内容型仓库，"核心实现"是它的**内容生产管线与发布链路**，关键机制如下：

1. **统一章节模板**（`_template_vuln/README.md`）：所有新章节必须从模板出发，固定为 Summary → Tools → Methodology → Labs → References 的骨架，并强制目录锚点链接。模板骨架原文：

    ```markdown
    ## Summary
    * [Tools](#tools)
    * [Methodology](#methodology)
        * [Subentry 1](#subentry-1)
    * [Labs](#labs)
    * [References](#references)

    ## Tools
    * [username/tool1](https://github.com/username/tool1) - Description of the tool

    ## Labs
    * [Root Me - Lab 1](https://root-me.org)
    * [PortSwigger - Lab 2](https://portswigger.net)
    ```

    这保证了 63 个章节体例一致，读者在任何章节都能按同一心智模型检索——这是清单类项目可维护性的核心设计。

2. **载荷脱敏规范**（CONTRIBUTING.md）：每个 PR 必须遵守 sanitize 规则——RCE 概念验证只用 `id`、`whoami`；回调地址写 `[ATTACKER.DOMAIN.TLD]`；IP 用 `10.10.10.10/11`；特权账号统一 `Administrator`、普通账号 `User`；默认密码统一 `P@ssw0rd` 等。这把全库载荷约束在"无害化示例"层面，是该资源能长期公开托管的关键。

3. **双通道 Markdown 质量门禁**（`.github/workflows/check-markdown.yml`）：push/PR 时用 `tj-actions/changed-files` 检出变更的 `.md`，只对增量文件跑 `markdownlint-cli2-action`；规则在 `.github/.markdownlint.json`（关闭行宽 MD013、允许内联 HTML MD033，约束缩进 4 与标题重复策略）。配合 `.pre-commit-config.yaml` 的本地 Docker 版钩子，写作规范在本地与 CI 保持同一套配置。

4. **自动化发布链路**（`.github/workflows/mkdocs-build.yml`）：master 分支每次推送触发 `mkdocs gh-deploy --force`，把仓库自身作为 docs 目录（workflow 内以 submodule 形式重新挂载自仓）构建 material 站点并发布到 GitHub Pages，形成"GitHub 阅读 + 在线站点阅读"双形态；git 插件还把每页的修订时间与提交者信息渲染进站点。

5. **章节内容纵深示例**（`SQL Injection/README.md`）：章节内部的 Methodology 不是罗列载荷，而是按攻击者决策树展开——从 Entry Point Detection、DBMS Identification（关键词/报错两条路径）到 UNION/Error/Blind（布尔、时间、OAST 带外）/Stacked/Polyglot 注入，最后是 No Space/No Comma/No Equal 三类 WAF 绕过小节。这种"流程化组织"是其区别于普通 payload 列表的核心价值。

## 6. 构建与运行

仓库无需构建、无测试套件（内容型仓库）。实际可用的操作命令（来自 CONTRIBUTING.md 与 workflow 文件）：

```bash
# 本地校验 Markdown（CONTRIBUTING.md 提供，Windows PowerShell 形式）
docker run -v $PWD:/workdir davidanson/markdownlint-cli2:v0.15.0 "**/*.md" --config .github/.markdownlint.json --fix

# 本地预览/构建站点（依赖由 mkdocs-build.yml 推断）
pip install mkdocs-material mkdocs-material[imaging] mdx_truly_sane_lists
mkdocs serve    # 本地预览
mkdocs build    # 构建静态站点（配置见 mkdocs.yml）
```

在线版由 CI 自动发布至 https://swisskyrepo.github.io/PayloadsAllTheThings/ （mkdocs-build.yml）。

## 7. 收录机制与质量控制（内容型特例）

- **收录范围**：63 个 Web 漏洞类型章节 + 33 个横向方法论文档，覆盖 OWASP 常见类别（注入类、认证类、上传类、反序列化类、配置类）并延伸至红队/内网主题。全库 Markdown 约 2.04 万行、3,555 个外部链接（grep 统计），单章链接密度高（XSS README 143 个链接、SQLi README 57 个）。
- **章节内标准件**：每章除 README.md 外按约定可含 Intruder/（Burp Intruder 载荷文件）、Images/（配图）、Files/（引用的脚本样本），约定见 README.md 与 CONTRIBUTING.md；实操脚本如 `Web Sockets/Files/ws-harness.py`（WebSocket fuzz 框架）、`Upload Insecure Files/CVE FFmpeg HLS/gen_xbin_avi.py`（FFmpeg HLS 解析漏洞 PoC 生成器）均归档在对应 Files 目录下。
- **更新机制**：纯 PR 驱动 + 维护者审核。规范规定了载荷脱敏与引用格式（引用必须有作者、标题、链接、日期，失效链接走 Wayback Machine，见 CONTRIBUTING.md），markdownlint 保证格式一致性。无内容自动化流水线，质量依赖人工与模板约束。
- **条目可追溯性**：每章 References 强制带日期与作者，引用 GitHub 工具须用 `[author/tool](URL) - Description` 格式（CONTRIBUTING.md），使内容可回溯原始出处。
- **姊妹项目生态**：README 列出同族项目 InternalAllTheThings（AD/内网）、HardwareAllTheThings（硬件/IoT），形成 "AllTheThings" 系列品牌； sponsors（SerpApi、ProjectDiscovery、VAADATA）提供项目资金支持（README.md、.github/FUNDING.yml）。

## 8. 社区与维护状态

- 活跃度：极高。创建于 2016-10，最近推送 2026-10-30（gh api），十年间保持持续更新；38 个 open issues，17,542 forks（gh api）。
- 贡献者：contributors API 分页显示约 325 人，README 用 contrib.rocks 徽章展示并致谢，是典型的社区共建型清单。
- 发布节奏：无版本号/release 概念（内容型仓库，master 即最新版）；"发布"即 GitHub Pages 站点的自动重建。
- 维护者：主维护者为 swisskyrepo（渗透测试工具作者，同族项目作者身份见 README），CI 提交署名 "Swk"（mkdocs-build.yml）。贡献者结构的长尾分布为行业惯例，未做逐人确认。
- 商业支持：README 列出 SerpApi、ProjectDiscovery、VAADATA 三家赞助商（均附 logo 与一句话介绍），另有 GitHub Sponsors 按钮（README 徽章与 `.github/FUNDING.yml`、`.github/sponsors/` 素材），项目具备自我造血能力。

## 9. 亮点与风险

**亮点**

1. 体例统一到罕见的程度：63 章共用一套模板与命名约定，Labs 小节直接给出可练手的 Root Me/PortSwigger 实验链接，学习路径闭环（`_template_vuln/README.md`、CONTRIBUTING.md）。
2. 载荷脱敏规范明确（CONTRIBUTING.md），兼顾实用性与公开托管合规性。
3. 双阅读形态：GitHub 原生渲染 + mkdocs-material 在线站点，检索体验好（mkdocs.yml、mkdocs-build.yml）。
4. 内容纵深真实：单章按攻击者决策树组织而非平面罗列，附带可运行的辅助脚本（如 `File Inclusion/Files/LFI2RCE.py`）与 Burp Intruder 载荷文件。
5. 十年持续维护、325 贡献者、生态品牌（姊妹项目 + 赞助），社区信任度即其护城河。

**风险 / 局限**

1. **合规边界**：内容为公开的安全研究教育用途，但存在被滥用于未授权测试的可能；仓库以 DISCLAIMER.md 明确"仅供教育与研究、须获得明确授权、作者不承担滥用责任"，使用者须自行承担法律责任。
2. **时效性风险**：手法与工具链接（3,555 个外链）依赖社区修复，个别条目可能过时或失效；引用规范强制 Wayback 兜底，但无法完全避免。
3. **准确性依赖人工审核**：无自动化的技术正确性校验（markdownlint 只管格式），个别载荷写法可能随目标环境（版本/WAF）变化而失效，读者需自行验证。
4. **非软件属性**：无代码架构、无测试、无版本发布，不适合以软件工程视角评估；Python 脚本仅为章节附件，无统一维护标准。

## 10. 总结评价

PayloadsAllTheThings 是"社区共建 + 严格模板 + 脱敏规范"三件事做到极致的内容型仓库：它把渗透测试中最常被查阅的手法知识固化为一套 63 章的标准化速查手册，用十年不间断的社区维护换来了 Web 安全领域事实上的参考标准地位（8 万+ 星、17.5k fork）。它涨星的逻辑很朴素——对每个从业者都是高频刚需的案头工具，且打开即用、体例始终一致。适合渗透测试工程师、红队成员、Bug Bounty 猎手和安全学习者作为速查与学习地图使用；对想研究"如何组织大型知识型开源项目"的人，其 `_template_vuln` 模板 + 贡献规范 + markdownlint 双通道质量门禁 + 自动站点发布的组合是很好的范本。使用时须遵守仓库免责声明：仅在获得明确授权的系统上进行测试，将其内容严格用于合法的安全研究与教育目的。
