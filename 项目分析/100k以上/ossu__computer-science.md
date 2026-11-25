# ossu/computer-science 项目分析

> 星数 210261（快照 2026-10-24）｜主语言 HTML｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：OSSU（Open Source Society University）推出的"免费自学计算机科学完整本科课程路线图"——用全球最好的 MOOC 课程拼出一张对标 ACM/IEEE CS2013 本科培养方案的课表，是 GitHub 上最具影响力的系统性自学 CS 课程清单。

| 项目 | 事实 |
| :-- | :-- |
| 名称 | ossu/computer-science |
| 星数 / Fork / Watch | 210,261 / 26,035 / 5,984（gh api 快照 2026-10-30） |
| 主语言 | HTML（GitHub 标签；实际内容以 Markdown 为主，HTML 来自 Jekyll 站点模板） |
| 许可证 | MIT（LICENSE，Copyright 2015-2023 Open Source Society University） |
| 创建时间 | 2014-05-04（gh api） |
| 最近 push | 2026-07-14（gh api），持续活跃近 12 年 |
| 官网 | cs.ossu.dev（CNAME 指向，Jekyll + minima 主题发布） |
| 贡献者 | 约 155 人（gh api contributors 分页推算） |
| 维护团队 | Eric Douglas（创始人）、Josh Hanson（技术主维护）、Waciuma Wanjohi（学术主维护），README "Team" 一节 |

## 2. 项目是什么，解决什么问题

互联网上免费课程极多（MIT OCW、Coursera、edX、Harvard CS50……），但零散、质量参差、缺乏先后顺序。学习者最常见的困境是"不知道学什么、按什么顺序学、学到什么程度算够"。OSSU 的答案是：**按本科 CS 培养方案组织一套完整课程序列**。README 开篇（README.md Summary 一节）明确声明其定位不是职业培训，而是"完整、全面的计算机科学教育"，对标本科 CS 专业要求，只去掉通识课（默认学习者已在其他领域受过教育）。

课程遴选有四条硬标准（README.md）：必须开放选课、定期开课（最好自学节奏）、教学质量高、内容符合 ACM/IEEE **CS2013** 本科课程大纲。找不到合格课程时用教材补位；进不了正课的好资源放入 `extras/courses.md` 与 `extras/readings.md`。这套"以权威大纲为锚、宁缺毋滥"的筛选逻辑是它与普通 awesome 清单的本质区别（CURRICULAR_GUIDELINES.md 明确以 CS2013 报告为准绳）。

课程体系分四段（README.md Organization 一节）：**Intro CS**（试水，判断自己适不适合 CS）→ **Core CS**（约等于本科前三年的必修课：编程、数学、CS Tools、系统、理论、安全、应用、伦理 8 个板块）→ **Advanced CS**（大四选修：编程/系统/理论/信息安全/数学 5 个方向）→ **Final Project**（自选项目或专项课程集，验证并展示所学）。README 中主课表含 60 余门课程条目（README.md 表格约 61 行带"N weeks"时长、65 行课程链接），每门都标注时长、每周投入、先修要求和 Discord 讨论入口；按每周 20 小时投入估算约 2 年完成。

各板块选课的代表性证据（均见 README.md 课表）：

| 板块 | 代表课程（来源） |
| :-- | :-- |
| Intro CS | MIT 6.100L《Introduction to CS and Programming using Python》（14 周） |
| Core programming | Systematic Program Design / Class-based Program Design（HTDP 体系）、华盛顿大学 Programming Languages、NEU 面向对象设计、Coursera 软件架构 |
| Core math | MITx 微积分三部曲 18.01.1x-3x、MIT 6.042J 离散数学（附两个社区解题仓库链接） |
| Core systems | Nand2Tetris 一部曲与二部曲、OSTEP 操作系统（自制讲义页）、UMass 计算机网络 |
| Core theory | Stanford《Algorithms: Design and Analysis》Part 1/2（附 algorithmsilluminated.org 备选） |
| Core security | RIT Cybersecurity Fundamentals、Coursera 安全编码/漏洞识别等 3+1 门 |
| Core applications | Stanford 数据库三部曲、DeepLearning.AI 机器学习、UCSD 图形学、UBC 软件工程 |
| Final project | Fullstack Open、Coursera 各 Specialization（机器人、大数据、云、数据科学等 9 个方向示例） |

费用与时长也写进了机制：README "Duration" 一节给出"每周 20 小时、约 2 年完成"的基线，并配套官方 Google Sheets 估算表（复制后填入开始日期与每周时数，自动推算完成日）；"Cost" 一节明确课程材料免费但部分平台的作业批改/证书可能收费，并附 Coursera 与 edX 助学金申请链接。付费内容一律标注为可选（FAQ.md 与各 coursepage 反复强调"All paid additions should be considered optional"）。

边界同样清晰（FAQ.md）：OSSU **不发文凭**（FAQ 首问"Does OSSU offer a degree?"答 No），**不做语言/框架培训**（FAQ 指向 Hackr.io 与 Reddit 语言社区），也不维护遗留的第三方应用——这些边界声明把用户预期管理做在了文档里，减少了大量 issue 纠纷。

它同时是一个**社区**而非纯清单：每门课挂 Discord 频道（README 每张课表行都有 "Discussion" 列），官方建议学习者 fork 本仓库、在课表上打 ✅ 当作个人看板（README "How to show your progress"）。仓库、课程、社区三者互为支撑，这是它能持续涨星并留住学习者的关键设计。

## 3. 技术栈与依赖

作为内容型仓库，技术栈极薄，全部用于把 Markdown 课表发布为静态网站：

- **Markdown**：22 个 .md 文件构成全部内容（README、FAQ、coursepages/、extras/ 等）。
- **Jekyll + GitHub Pages**：`_config.yml` 指定 `remote_theme: jekyll/minima`（锁定到具体 commit `7d91bb5`，保证渲染稳定），`nav_pages` 配 FAQ/HELP 导航；`_layouts/page.html` 只有一个 10 行的极简页面布局；`_includes/` 仅 custom-head/footer/nav-items 三个片段。
- **GitHub Actions**：仅一条 workflow `.github/workflows/delete-empty-issues.yml`，用 actions-cool/issues-helper 自动回复并关闭空 issue。
- **无任何代码构建、无包管理器、无第三方运行时依赖**——仓库的"产品"是信息本身。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| :-- | :-- |
| `README.md`（453 行） | 核心课表本体：从先修要求到 Final Project 的全部课程表格与说明 |
| `coursepages/intro-cs/` | MIT 6.100L 入门课详情页：逐 PSET 节奏、自评方式、环境兼容注意事项 |
| `coursepages/intro-programming/` | 零基础备选入口（Harvard CS50P 等三选一），给 Intro CS 跟不上的学生兜底 |
| `coursepages/spd/`、`class-based/` | 两门程序设计方法课（HTDP 体系）的说明与配套材料 |
| `coursepages/ostep/` | 操作系统课：Base/Extended 双路径 + 5 个 xv6 内核项目页（Project-1B、Project-2A、Scheduling、vm-xv6 等） |
| `extras/courses.md`（102 行） | 落选正课的优质课程（CS50、SICP、Java MOOC 等），按主题分表 |
| `extras/readings.md`（102 行） | 经典书单（SICP、设计模式、重构、代码大全等），按主题分表并注作者 |
| `extras/other_curricula.md` | 5 个同类自学路线对照与 OSSU 差异化自述 |
| `extras/puzzles-practice-plods.md` | 脱离课程的刷题/实践站点 |
| `CURRICULAR_GUIDELINES.md` | 声明课程遴选锚点：ACM/IEEE CS2013 大纲 |
| `FAQ.md`（114 行） | 13 个高频问题：学位、顺序、免费性、alt 链接含义等 |
| `HELP.md` / `CONTRIBUTING.md` | 求助渠道（FAQ → Discord 分级）、贡献指南 |

FAQ 覆盖的问题清单（FAQ.md 目录，反映运营中沉淀的真实痛点）：是否发文凭、如何补数学先修、学特定语言去哪、Firebase 旧应用为何失效、课程顺序、免费边界（Coursera/edX 付费墙）、edX 证书是否必须购买、alt 备选链接的含义、为何不覆盖某主题、为何缺先修、为何软工课要求先做过大项目、如何找特定主题课程。
| `CHANGELOG.md`（331 行） | 课表版本史：semver 管理，当前 8.x，正向 v9 评审过渡 |
| `_config.yml`、`_layouts/`、`_includes/`、`CNAME` | Jekyll 静态站配置与模板，发布到 cs.ossu.dev |
| `.github/` | RFC issue 模板 + 关闭空 issue 的 workflow |

组织方式评述：典型的"**单页课表 + 详情子页 + 附加清单**"三层结构。README 承担一屏总览（用表格压缩全部信息），需要展开的课程才给独立 `coursepages/` 页面；extras 收纳落选资源而不污染主线。整个仓库无代码模块划分可言，但信息架构非常清晰，是同类学习路线仓库中被模仿最多的范本。

## 5. 核心架构与关键实现

内容型仓库，此处"架构"指课表的信息组织与治理机制，选取三个关键点：

**1）以 CS2013 大纲为唯一遴选标准（CURRICULAR_GUIDELINES.md + CONTRIBUTING.md）**
仓库把 ACM/IEEE CS2013 报告作为学术宪法，CONTRIBUTING.md 明确列出可接受的活动 criticism 类型："把选修课列为必修"、"缺必修主题的课"、"同一主题冗余多门"、"存在更优课程"。这把主观争议转化为可裁决的规则问题——大型社区清单能不烂尾的核心治理设计。

**2）RFC 治理流程（.github/ISSUE_TEMPLATE/request-for-comment-template.md）**
实质性课程变动必须开 RFC issue，模板要求一句话问题描述 + 引用大纲条款证明 OSSU 违反了哪条，并规定"通常公示 1 个月"收集社区意见后再合入。配合 CHANGELOG.md 的 semver 语义（MAJOR=主题变化、MINOR=换课不换主题、PATCH=顺序与美化），课程演进有据可查。CHANGELOG 头部注明当前正对照大纲做 v9 全量评审（"curriculum is currently undergoing review for v9"）。

**3）课程详情页的"讲义化"（coursepages/intro-cs/README.md、coursepages/ostep/README.md）**
对外部 MOOC 只给链接不够的部分，社区自写操作指南。以 intro-cs（MIT 6.100L）为例：逐条列出作业截止节奏（PSET 0-5 对应课次）、自评方式、Python 3.8 版本兼容注意事项、"要不要买证书"的利弊分析；ostep 页则设计了"Base（约 80 小时）/ Extended（200+ 小时内核开发）"两条学习路径并附 xv6 项目页。这类页面是 OSSU 相对"纯链接清单"的实质增值。

同时课表内置了分层兜底：intro-cs 页顶部标注 "⚠️ Under Review" 并链接 RFC issue #1266，给跟不上的学习者指回 `coursepages/intro-programming/`（Harvard CS50P 等三门自学节奏课三选一）；先修数学则整个剥离到姊妹仓库 ossu/precollege-math，主线只保留链接。这种"主线瘦身 + 旁路兜底"的写法让 453 行的 README 保持了可扫读性。

**4）轻量反垃圾自动化（.github/workflows/delete-empty-issues.yml）**
星数 20 万+ 带来大量空 issue，一条 30 行的 Action 在 issue 打开时检测空 body 或模板占位文本，自动留言教育并关闭（`.github/workflows/delete-empty-issues.yml`，pin 到 actions-cool/issues-helper v3 的具体 commit）——用最小成本守住维护者精力。

**5）课表主线的板块切分（README.md）**
Core CS 八个板块的顺序本身是设计产物：先"Core programming（含函数式，用 Standard ML/Racket 双语言训练抽象）+ Core math（微积分、离散数学）"打底，再 CS Tools 补工程工具（MIT Missing Semester），然后系统（Nand2Tetris 两部曲 + OSTEP + 计算机网络）、理论（Stanford 算法两部曲）、安全（3 门必修 + 2 选 1）、应用（数据库三部曲、机器学习、图形学、软件工程）、伦理收尾。每个板块先用 "Topics covered" 标签行声明覆盖的大纲知识点，再给课程表——板块与 CS2013 Knowledge Area 一一呼应，可对照审计。

## 6. 构建与运行

- **学习者视角**：无需构建。直接阅读 README.md 或访问 cs.ossu.dev（GitHub Pages 自动渲染 Jekyll）；fork 仓库打 ✅ 跟踪进度（README "How to show your progress"）；配合官方 Google Sheets 时间估算表排期。
- **本地/自托管预览**：标准 Jekyll 流程（`bundle exec jekyll serve`），仓库未带 Gemfile，主题经 `_config.yml` 的 remote_theme 远程加载（`_config.yml`，锁 commit 的 minima 主题，skin: auto 适配深浅色）。
- **无测试、无 CI 构建、无发布脚本**：`_config.yml` 之外无任何工程化文件；唯一自动化是上述 issue 管理 workflow；站点发布由 GitHub Pages 按 master 分支自动完成（默认分支 master，gh api）。

## 7. 社区生态与更新机制（内容型特例）

- **收录范围与规模**：主课表 60+ 门课（README.md），extras 追加数十门备选课（extras/courses.md，102 行）与约 40 本经典书（extras/readings.md，102 行），另有刷题站与 5 个同类路线对照（extras/puzzles-practice-plods.md、other_curricula.md）。README 中共 154 处课内外链接，全部指向 MIT、Harvard、Stanford、Princeton、UBC 等公开课或官方教材站。
- **分类体系**：四段式（Intro → Core → Advanced → Final Project）+ Core 内 8 板块 + Advanced 内 5 方向；extras 按主题分表。课程统一用"课程｜时长｜每周投入｜先修｜讨论"五列表格呈现，条目粒度、标注字段完全一致，可机读性良好。
- **更新机制**：双轨制——非实质性修正（链接失效、错字）直接 PR（CONTRIBUTING.md）；课程取舍走 RFC issue 公示约 1 个月（.github/ISSUE_TEMPLATE/）。extras/courses.md 中还能看到社区留下的链接巡检注释（如"currently 404's as of 2025-06-23"），说明死链治理靠社区人工巡查。
- **姊妹项目**：ossu/precollege-math（先修数学，README 先修节以 ossu.dev 域名引用）、ossu/code-of-conduct（README 尾部）；官网 cs.ossu.dev、Discord 服务器（badges 与各课表行内嵌邀请链接）。
- **生态位**：`extras/other_curricula.md` 主动列出 Teach Yourself CS、Google 技术发展指南、P1xt guides 等竞品并自我定位："OSSU 专注本科 CS 全景，以 CS2013 为纲"——即同类路线中**最体系化、最接近学历教育**的一支。
- **为什么能涨到 20 万星**（综合上述证据的判断）：一是时间红利——2014 年即上线，是 MOOC 浪潮期最早给出"完整本科等效"答案的项目（gh api created_at 2014-05-04）；二是权威背书——Awesome 清单收录（README 徽章）+ ACM/IEEE 大纲锚点，被大量博客、院校与 Reddit 帖子转引；三是低门槛高完成度——零代码、打开 README 即用，而 fork 打勾机制把 star 转化为长期留存的"用户资产"；四是社区网络效应——Discord 同伴压力与逐课讨论频道让清单具备了在线大学的黏性，星数随届届学习者复利增长。
- **版本节奏**：CHANGELOG.md 记录到 8.0.0（2017-11-01），此后改动直接进主课表，当前处于 v9 评审期（CHANGELOG.md 头注）；仓库 2026-07 仍有 push（gh api）。

## 8. 社区与维护状态

- 持续活跃近 12 年：创建于 2014-05，最近 push 2026-07-14（gh api），29 个 open issues，约 155 名贡献者（contributors API 分页 Link 头 last=155 推算）。
- 维护结构明确：README "Team" 列出创始人 Eric Douglas、技术主维护 Josh Hanson、学术主维护 Waciuma Wanjohi 三人分工（技术/学术双主维护是内容型项目少见的设置）。
- 社区规模证据：README 头部挂 Awesome 精选徽章与 OSSU 品牌徽章，Community 节的 Discord 徽章实时显示服务器在线人数（服务器 744385009028431943，2019 年起启用）；为组织化学习还提供了 LinkedIn 学校主页（README Community 节）。
- **用户社区在 GitHub 之外**：Discord 为第一求助与讨论场所（HELP.md、README Community 节，服务器 ID 744385009028431943，徽章显示在线人数）；HELP.md 规定了分级求助路径——课程内容问题先找平台（Coursera/edX 论坛），再回 Discord 并"说明已尝试的步骤"；LinkedIn 学校主页、学习进度 spreadsheet 等周边齐备。
- **贡献者结构**：约 155 名贡献者（contributors API 分页推算），改动以课表微调 PR 为主；三位核心维护者长期稳定（README Team 节自 2014 年项目化运营以来延续）。
- **风险信号**：README 明确警告存在大量第三方/过时的 OSSU 衍生物（废弃的 Firebase app、Trello 板、Notion 模板），官方要求学习者只认官网与本仓库（README Warning 节）——侧面反映项目影响力已外溢到失控边缘。
- **issue 侧证据**：当前 29 个 open issues（gh api 2026-10-30），配合 RFC 模板与自动关闭空 issue 的 Action，主仓库 issue 区保持了可治理状态；RFC 评审周期约 1 个月（模板规定），大型调整（如 intro-cs 是否加兜底路径）会以单个 RFC 长期跟踪（#1266）。
- release 节奏：无 GitHub release 惯例（未确认具体 release 列表），版本演进以 CHANGELOG + 课表直接更新为主。

## 9. 亮点与风险

**亮点**
1. 以 ACM/IEEE CS2013 官方大纲为遴选锚点，"完整本科等效"的定位在自学路线类仓库中独树一帜，权威性远超一般 awesome 清单（CURRICULAR_GUIDELINES.md）。
2. 治理成熟：RFC 公示期 + semver CHANGELOG + 明确的可裁决变更标准，12 年课表演进全程可追溯（.github/ISSUE_TEMPLATE/、CHANGELOG.md）。
3. 对重点课程提供社区自写讲义级详情页（coursepages/，如 intro-cs 的逐 PSET 节奏表、ostep 的 Base/Extended 双路径），信息增值而非转贴链接。
4. 社区闭环：每门课挂 Discord 频道、fork 打勾进度追踪、时间估算表，把"读清单"变成"跟课打卡"。
5. 星数 210k、fork 26k，是 GitHub 上被引用最多的自学 CS 路线之一，已进入 awesome 精选（README 徽章）。
6. 工程细节讲究：minima 主题锁 commit 防渲染漂移、Action pin 具体版本、alt 链接机制为每门课备好平台失效后的退路（_config.yml、README 课表与 FAQ）。

**风险/局限**
1. 强依赖外部平台：课表中大量 Coursera/edX 链接会随平台改版失效或转为收费（README "Cost" 一节承认部分作业/证书要付费；extras/courses.md 已有 404 巡检注释），链接腐烂是长期主要维护负担。
2. 无证书、无学分：FAQ 首条明示 OSSU 不提供学位，对需要学历背书的学习者价值有限（FAQ.md "Does OSSU offer a degree?"）。
3. Advanced 阶段引导较弱：选修方向只列课程不排顺序，完全依赖学习者自驱；个别课程条目的讨论链接指向 Discord 频道 ID 直链，外部可发现性差（README Core programming 表）。
4. v9 评审进行中（CHANGELOG 头注），课表处于新旧交替期，部分页面（如 coursepages/intro-cs 标注 "⚠️ Under Review"、RFC issue #1266 悬而未决）短期存在内容不一致。

## 10. 总结评价

OSSU computer-science 是"内容型仓库"的标杆：它没有一行代码，却用大纲锚定 + RFC 治理 + 社区运营把一份免费课表做成了持续运营 12 年、21 万星的"在线大学"。它解决的不是技术问题而是信息与秩序问题——把全球最好的 MOOC 按本科培养方案串成两年可执行的计划，并配套社区答疑与进度追踪。适合零基础或转行者系统自学 CS、在校生补强体系、以及任何想做"高质量课程策展"的人参考其治理模式；对研究者而言，它也是观察大规模开源社区如何维护纯内容项目（防链接腐烂、防低质 issue、双维护者分工）的绝佳样本。想读"源码"的人应该读的是 README.md 的课表结构、CONTRIBUTING.md 的变更标准与 coursepages/ 的讲义写法。
