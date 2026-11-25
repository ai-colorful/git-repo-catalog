# lib-pku/libpku 项目分析

> 星数 30000（快照 2026-10-24）｜主语言 TeX（gh api 核实）｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：北京大学课程资料（试卷/答案/笔记/讲义）的学生社区共建仓库，GitHub 上"高校课程资料民间整理"现象的代表项目之一。

| 基本事实 | 值 | 来源 |
|---|---|---|
| 名称 | lib-pku/libpku | gh api |
| 星数 | 快照 30000（2026-10-24）；gh api 实测 34190（2026-11-21） | gh api repos/lib-pku/libpku |
| Fork 数 | 8165 | gh api |
| 主语言 | TeX（GitHub 语言统计口径；内容实际以 PDF/图片/Word 为主） | gh api |
| 许可证 | 无 LICENSE 文件（license=null）；README "License" 章节声明资料权利归原作者所有 | gh api、README.md |
| 仓库体积 | ~698 MB（714371 KB），tarball 561 MB | gh api、实测 |
| 最近活跃 | master 最后提交 2020-01-17（merge PR #57）；pushed_at 2022-01-05（master 上无对应提交，仅如实记录）；未 archived | gh api commits/branches |
| 提交/贡献者 | 132 commits、15 名贡献者（commits 口径） | gh api |
| Issue/PR | Issue 43（open 11）；PR 34（open 3，closed 31） | gh api search/issues |

## 2. 项目是什么，解决什么问题

libpku 自述为"贵校课程资料民间整理"（README.md 标题）——刻意使用泛化的"贵校"指代北京大学，是一个纯内容型仓库：不包含软件工程意义上的代码主体，而是按课程分类存放历年试卷、考试答案、课程笔记、讲义等学习资料的集合。README 的前言整段引用自浙江大学同类项目 QSCTech/zju-icicles，阐述动机：课程资料原本依靠口口相传、学长代代相授，获取"纯属偶然"，项目希望将其"变为公开的、易于获取的和大家能够共同完善、积累的共享资料"（README.md Preface）。

内容组织上，仓库按课程类型分为六个顶层目录（git 文件树实测）：专业课（81 个课程目录、717 个文件）、通选课（按 A–F 六个领域字母分类，54 个课程目录、215 个文件）、暑期课（1 个课程目录、17 个文件）、英语课（4 个课程目录、5 个文件）、公选课与政治课（各仅 1 个占位文件，政治课/README.md 指向 issue #21 说明资料缺失）。全部约 960 个文件、约 140 个课程目录，仓库总目录树仅 1148 个条目但体积近 700 MB，说明单文件普遍很大。

文件格式构成（git 树统计）：PDF 292 个、JPG 284 个、DOCX 113 个、DOC 84 个、PNG 60 个、TXT 43 个、PPT/PPTX 40 个、RAR/ZIP 压缩包 11 个、Markdown 11 个、TeX 3 个等。大量早期资料是试卷拍照（jpg/png），后期贡献逐渐转向 PDF 与电子文档。单个最大文件如 `专业课/计算概论A/计算概论实验班_2016_2017.rar`（47.3MB）、`专业课/高等代数/高等代数_2016_2017_大一上.zip`（28MB）、`通选课/D/中国美术史/中国美术史笔记2016夏.pdf`（21.8MB）。

文件命名习惯携带检索关键信息，普遍包含"年份 + 院系/教师 + 考试类型"，例如 `专业课/高等数学B/` 下有 `09级高数下杨磊期末.jpg`、`13信科期末答案.pdf`、`16期中 史逸.jpeg`、`关启安数分I期中1.jpg` 等（102 个文件），教师姓名直接入文件名是本仓库显著的组织特征。

需要说明：本分析只针对目录组织、协作模式与项目演化，不涉及资料内容本身的使用。

## 3. 技术栈与依赖

这是内容仓，技术栈极薄，只有一套站点生成与部署脚本：

- **Node.js（仅构建期）**：`index.js`（2197 字节）依赖 `directory-tree`、`remark`、`remark-toc`、`remark-html` 四个 npm 包（.travis.yml before_script 中安装），无 package.json、无锁文件——依赖完全由 CI 脚本现场安装。
- **Travis CI**：`.travis.yml`（578 字节）配置 node_js "10"、仅 master 分支触发，负责把目录树渲染成 HTML 并发布到 GitHub Pages。
- **GitHub Pages**：产出物推送到独立仓库 `lib-pku/lib-pku.github.io`，作为浏览入口 https://lib-pku.github.io/（README.md Usage；index.js 中硬编码的 HTML 模板与 og:url 元数据）。
- **内容文件本身**：PDF/图片/Office 文档，无任何运行时依赖；仓库根目录只有 4 个文件（.gitignore、.travis.yml、README.md、index.js，git 树实测）。

## 4. 目录结构与代码组织

顶层结构（git 树 1148 条目实测）：

| 目录 | 课程目录数 | 文件数 | 职责 |
|---|---|---|---|
| 专业课/ | 81 | 717 | 院系专业课程资料，按课程名建目录（如 高等数学B、数据结构与算法、计算概论A、量子力学） |
| 通选课/ | 6（A–F 领域）→ 54 课程 | 215 | 北大通选课，按领域字母 A–F 一级分组、课程名二级分组；E 类（文史哲艺术类）最多（98 条目） |
| 暑期课/ | 1 | 17 | 暑期小学期课程（acm-icpc暑期课） |
| 英语课/ | 4 | 5 | 英语类课程 |
| 公选课/ | 0 | 1 | 占位（无资料） |
| 政治课/ | 0 | 1 | README.md 指向 issue #21 说明情况 |
| 根文件 | — | 4 | README.md、index.js、.travis.yml、.gitignore |

组织方式评述：典型的"分类目录树即数据库"模式——不建索引清单文件，目录名本身就是课程分类体系，靠 GitHub 的目录浏览和自动生成的 Pages 站点导航。文件最多的课程目录：高等数学B（102 文件）、高等代数（50）、数据结构与算法（47）、高等数学D（44）、数学分析（43）、线性代数（40），理工科基础课（数理信）覆盖最厚，人文类课程多依赖单份大文档（如 `专业课/文化产业导论/notes.md` 68KB 课程笔记）。通选课 A–F 字母分组与北大通选课领域划分对应，但仓库内未附领域名称说明文件。

## 5. 核心架构与关键实现

**核心实现 1：目录树 → 静态站点生成（index.js）**

`index.js` 是仓库唯一"代码"：先用 `directory-tree` 递归扫描排除 node_modules/.git 后的目录树，`solve()` 递归函数把树渲染为 Markdown TOC（目录→`#` 标题+链接，文件→链接行，前缀硬编码 `https://github.com/lib-pku/libpku/raw/master/`），再经 remark + remark-toc + remark-html 转成 HTML，拼入硬编码的页面模板（`<aside id="sidebar">` 侧栏切分逻辑：`String(file).replace("<h1 id=", "</aside><h1 id=")`）。关键片段：

```js
let prefix = `https://github.com/lib-pku/libpku/raw/master/`;
function solve(e, depth) {
  ...
  if (e.type == "directory") {
    for (let i = 0; i < depth; ++i) s += "#";
    res.push(`${s} [${e.name}](${prefix + encodeURIComponent(e.path)})`);
  } else if (e.type == "file") {
    res.push(`[${e.name}](${prefix + encodeURIComponent(e.path)})`);
  }
```

设计含义：站点内容永远等于仓库目录结构，贡献者上传文件即自动更新导航页，维护者无需手工维护索引——这是"目录即界面"的零维护成本设计。

**核心实现 2：CI 自动部署（.travis.yml）**

master 每次 push 触发 Travis：`npm install directory-tree remark remark-toc remark-html` → `node index.js > lib-pku.github.io/index.html` → 以 Travis 机器人身份 force push 到 lib-pku.github.io 仓库（`${DEPLOY_TOKEN}` 凭据注入）。整条流水线 578 字节完成"内容提交 → 站点更新"闭环，是小型内容仓 CI 的极简样本。

**核心实现 3：README 即贡献指南与检索入口**

README.md（4531 字节）承担了 CONTRIBUTING/网站导航/免责声明三重职责：Usage 指向 Pages 站点并给出 DownGit 工具用于整目录下载；Contribution 章节给出 Web 端上传的具体操作路径；Related Works 章节收录 12 个同类高校仓库链接。

## 6. 构建与运行

来自 .travis.yml 的完整可复现流程（本分析未运行，仅静态摘录）：

```bash
npm install directory-tree remark remark-toc remark-html   # 无 package.json，手动装 4 个包
node index.js > index.html                                  # 生成目录导航页
# .travis.yml 中再由 CI 克隆 lib-pku.github.io 并 force push 部署
```

日常"运行"即浏览 https://lib-pku.github.io/；单目录批量下载依赖第三方 DownGit（README.md Usage 章节）。仓库本身无测试、无 lint、无构建校验——CI script 仅 `echo 'XD'`，唯一实际产物是 after_success 的站点部署。

## 7. 自动化与工程质量（内容仓特例观察）

- **工程质量口径换算**：对内容仓而言"质量"体现为分类一致性与命名规范。观察结论：课程目录命名基本统一（课程名为主），文件命名携带年份/教师/考试类型信息，但无强制规范文件（无 CONTRIBUTING.md、无模板、无校验脚本，根目录仅 4 文件）。
- **CI 极简但有实效**：Travis 从 2018 年起承担 Pages 部署（.travis.yml），master 独占分支，无 PR 保护、无 review 门槛（PR #57 直接 merge 为最后一次提交）。
- **健壮性弱点**：依赖不经锁文件固定、Node 版本写死 "10"、HTML 模板硬编码在 JS 字符串中，长期维护成本接近零但也几乎不可扩展。

## 8. 社区与维护状态

**项目时间线（gh api commits 实测，132 个提交全量按月分布）：**

| 时间 | 提交数 | 事件 |
|---|---|---|
| 2018-11 | 19 | 2018-11-22 建仓 + Initial commit（librarian-pku），首轮资料导入 |
| 2018-12 ~ 2019-03 | 10 | 缓慢增长 |
| 2019-04 | 54 | **峰值月**，占全仓库 41% 提交，社区集中贡献期 |
| 2019-05 ~ 2019-07 | 22 | 常规补充 |
| 2019-12 ~ 2020-01 | 27 | 最后一波活跃（期末季） |
| 2020-01-17 | — | 最后一次 master 提交：merge PR #57（brenner8023） |
| 2022-01-05 | — | pushed_at 时间戳（master 上无对应提交，如实记录） |
| 2026-11 | — | 仓库未 archived；事件流仅剩 WatchEvent（仍在涨星），11 个 open issue、3 个 open PR 无人处理，处于事实停更状态 |

**协作模式**：README.md Contribution 章节明示三条贡献通道——(1) Fork 后 GitHub Web 端 Upload File 发 PR（官方推荐，"由于本项目体积很大"）；(2) 附件投递到 Issue 由维护者添加；(3) 邮件 libpku@protonmail.com。这解释了 15 名直接提交者撬动 3.4 万星的放大结构：大量资料经 issue/邮件由维护者代提交，或 fork 后在 Web 端上传。8165 个 fork 也主要作为 Web 上传的工作副本存在。

**维护者结构**：15 名贡献者（gh api contributors），创始账号 librarian-pku 完成大部分 merge；无 .github/ 目录、无 issue 模板、无 CODEOWNERS。

**社区现象（客观描述）**：README Related Works 章节列出 12 个同类项目——浙大 zju-icicles、清华 REKCARC-TSC-UHT、中科大 USTC-Course、上交 SJTU-Courses、东南 seucourseshare、南大 NJU-Review-Materials、中大 SYSU-Exam、郑大 ZZU-Courses-Resource、广工 gdut-course 等，以及北大校内的 project-hover（气垫船计划，自称"免费、去中心化的往年题资料库"）、PKUCourse、信科学生会资料库。2018-2019 年间全国高校课程资料仓批量涌现，libpku 是其中星数最高的项目之一；这类仓库普遍采用相同的 README 互链方式形成生态网络，libpku 前言直接引用 zju-icicles 文本，体现了同类项目间的模板传播。

## 9. 亮点与风险

**亮点：**
1. "目录即数据库、CI 即发布"的零成本架构：index.js + Travis 全程仅 ~2.8KB 配置，把维护成本压到最低（index.js、.travis.yml）。
2. 命名规范内嵌检索信息：文件名普遍携带年份/教师/考试类型（如 `专业课/高等数学B/13信科期末答案.pdf`），无需索引文件即可 grep/GitHub 搜索定位。
3. 贡献通道分层设计：Web 上传 PR / Issue 附件 / 邮件三通道并存（README.md Contribution），对不熟悉 git 的学生群体友好，是 15 名提交者撬动 3.4 万星的关键。
4. 免责与权利声明清晰前置：Disclaimer 与 License 章节明确"资料权利由原作者所有、不收费、侵权联系邮箱下架"（README.md）。

**风险/局限：**
1. 仓库近 700MB 且持续增长，clone/tarball 代价高，官方文档已引导改用 Pages + DownGit（README.md）。
2. 内容未经授权收录的课程资料版权边界，是这类仓库客观存在的法律事实（README.md Disclaimer 亦承认并留有下架通道）；本报告仅客观记录此事实，不作展开。
3. 实质停更：master 最后提交 2020-01-17，2020 年后的课程资料不再进入本仓，信息时效性随时间衰减；11 open issue / 3 open PR 冻结无人处理（gh api 实测 2026-11-21）。
4. 工程链脆弱：Node 10 写死、依赖无锁、Travis 免费额度收紧后 CI 部署链路已不可复现性存疑（未确认，仅配置层面的静态判断）。

## 10. 总结评价

libpku 是中国高校"课程资料 GitHub 化"浪潮中星数最高的标本之一：它用一个 2KB 的 Node 脚本和一套课程目录约定，把原本封闭在 QQ 群和学长硬盘里的资料沉淀为公开、可检索、可持续共建的仓库，其"目录即数据库 + CI 自动发布 + Web 端低门槛贡献"的组合是内容型开源仓的教科书式设计。项目已于 2020 年初事实停更，但截至 2026 年仍以每月数百的增速涨星，说明这类"人找资料"的存量需求长期存在；其后续生态（如 README 中列出的去中心化尝试 project-hover）也预示了版权与可持续性问题的另一种演进方向。适合研究者把它当作高校数字互助社区与轻量内容仓协作模式的案例样本阅读——读它的 README、目录树与 2.8KB CI 配置，比读任何一份资料本身更有价值。
