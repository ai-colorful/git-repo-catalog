# PKUFlyingPig/cs-self-learning 项目分析

> 星数 76386（快照 2026-10-24）｜主语言 HTML｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：中文互联网影响力最大的系统性 CS 自学指南——一本由北大学生以亲身自学经历写成的开源"书"（在线站点 csdiy.wiki），将全球名校公开课按计算机本科知识体系组织成可执行的学习路线。

| 项目 | 内容 |
| --- | --- |
| 名称 | PKUFlyingPig/cs-self-learning（CS 自学指南） |
| 在线站点 | https://csdiy.wiki （域名见 `docs/CNAME`） |
| 星数 | 76386（快照 2026-10-24） |
| 主语言 | HTML（实际内容为 Markdown + mkdocs 构建，HTML 为构建产物占比） |
| 许可证 | MIT（作者编写部分；引用的课程/书籍资源遵循原作者许可，见 `README.md` 许可节与 `LICENSE`，版权人 Yinmin Zhong，2021） |
| 创建时间 | 2021-10-20（GitHub API） |
| 最近推送 | 2026-09-17（GitHub API pushed_at，与 tarball 内文件 mtime 2026-09-17 一致） |
| 贡献者 | 167 人（GitHub API contributors 分页推算） |
| 规模 | 全仓 286 个文件，docs 下 270 个 Markdown：136 个中文 + 134 个英文翻译，中文正文约 355KB |

## 2. 项目是什么，解决什么问题

本书解决的自学痛点在 `README.md` 与前言（`docs/index.md`）中写得很直白：欧美名校把极高质量的 CS 课程全部开源后，"好课虽多，却只能在收藏夹里吃灰"——课程繁多不知如何选择、资料零散残缺、作业难度不知深浅。作者把大一到大四一路自学走过的路线、踩过的坑、每门课的真实体验整理成册，让一个小白能凭开源社区资源在 2-3 年内完成接近完整 CS 本科的训练。

内容覆盖面按"一本完整的书"来设计（`docs/index.md`）：使用指南、一份参考性 CS 学习规划、必学工具、好书推荐，然后是主体——按 数学基础/编程入门/数据结构与算法/计算机系统基础/体系结构/操作系统/并行与分布式/系统安全/计算机网络/数据库/编译原理/图形学/Web 开发/数据科学/AI/机器学习/深度学习/大模型 等方向分章的国内外高质量课程汇总。

每个课程页面不是干巴巴的链接列表，而是作者（或贡献者）亲学后的评价：`docs/计算机系统基础/CSAPP.md` 中对 CMU 15-213 给出五星难度、150 学时的预估、先修要求（CS61A/CS61B）、B 站中文讲解资源、以及北大 ICS 课程各 Lab 的第三方详尽笔记链接——这种"带个人体验的导学"正是它区别于 awesome 清单的核心价值。

本书有明确的社区化机制：README 专门写了贡献流程（新课程参考 `template.md` 模板、在 `mkdocs.yml` 添加 navigation、遵守中英混排规范、需提供英文翻译），页面下方通过 Giscus 挂载 GitHub Discussions 评论，读者可在课程页下自建学习群互助（`README.md` 交流社区节）。

定位与生态位：它是中文 CS 自学圈公认的"总路线图"级项目，与 trendshift 徽章（README 第 9 行）和 76k 星的量级互相印证；同类仓库多为纯清单，本书以"个人经历 + 课程地图 + 双语站点 + 评论社区"的组合占据独特位置。

## 3. 技术栈与依赖

技术选型极克制，是一个标准的 mkdocs 文档站工程（`requirements.txt`）：

| 依赖 | 版本 | 用途 |
| --- | --- | --- |
| mkdocs-material | 9.5.2 | 站点主题（`mkdocs.yml` theme.name: material） |
| mkdocs-static-i18n | 1.2.0 | 中英双语构建，zh 为默认语言、en 通过 `nav_translations` 映射导航名（`mkdocs.yml` plugins.i18n） |
| mkdocs-minify-plugin | 0.7.1 | HTML 压缩 |
| mkdocs-git-revision-date-plugin | - | 页脚显示最后修订时间 |
| mkdocs-open-in-new-tab | 1.0.8 | 外链新窗口打开 |
| jinja2 | 3.1.2 | 模板引擎（mkdocs 自身依赖） |

其余技术点：搜索用 mkdocs-material 内置 search（zh+en 双语索引，`mkdocs.yml`）；评论系统为 Giscus（GitHub Discussions 驱动，`overrides/partials/comments.html`，并有一段同步明暗主题的自定义 JS）；站点统计用 Google Analytics（`mkdocs.yml` extra.analytics，ID G-KJB4RXSKZG）。内容本身为纯 Markdown + 少量 HTML/attr_list 语法，无任何自研代码。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `docs/` | 全部内容：27 个分类子目录（必学工具、数学基础、编程入门、操作系统……）+ 顶层页（index、使用指南、CS学习规划、好书推荐、后记） |
| `docs/*.en.md` | 对应中文页的英文翻译（134 个，覆盖几乎所有中文页） |
| `docs/images/` | 站点图片资源（title.png、favicon 等） |
| `docs/CNAME` | 自定义域名 csdiy.wiki |
| `mkdocs.yml` | 站点核心配置：主题、插件、i18n 语言映射、完整导航树（约 170 行 nav） |
| `overrides/partials/comments.html` | material 主题定制：Giscus 评论组件与主题同步脚本 |
| `template.md` / `template.en.md` | 新增课程页的标准模板（中英两份） |
| `requirements.txt` | Python 依赖清单 |
| `.github/workflows/ci.yml` | 唯一 CI：push 到 master/main 时构建并 gh-deploy |
| `README.md` / `LICENSE` / `.editorconfig` | 项目说明、MIT 许可、编辑器格式约定 |

组织方式评述：单体单源内容仓——一份数据（Markdown）+ 一份导航声明（mkdocs.yml）+ 极少量定制模板。没有多包/monorepo 概念，工程复杂度几乎为零，维护重心完全在内容质量上。

## 5. 核心架构与关键实现

内容型仓库无传统代码架构，其"核心实现"是三套机制：

**（1）导航即课程地图**（`mkdocs.yml` nav 节）。全书结构硬编码在这约 170 行 nav 树中，共 29 个一级分类、约 116 门课程页 + 14 个工具教程页（由 nav 条目逐项统计）。层级做了二级分组，例如"编程入门"下再分 Python/C/C++/Java/Rust/函数式语言，"深度生成模型"下含"大语言模型"子节（CMU 11-868/11-667/11-711），使新手能按语言/方向渐进检索。

**（2）统一的课程页模板**（`template.md`）。每个课程页强制同一骨架：所属大学/先修要求/编程语言/课程难度（星级）/预计学时 + 课程简介 + 课程资源（网站/视频/教材/作业）+ 资源汇总（指向贡献者个人维护的作业实现仓库）。模板还内嵌了 markdownlint 与中英混排规范链接，保证上百个作者各异的页面风格一致。实际页面严格遵循该模板（见 `docs/计算机系统基础/CSAPP.md`）。

**（3）双语构建管线**（`mkdocs.yml` plugins.i18n + `docs/*.en.md`）。中文页为默认（`index.md`），英文页以 `index.en.md` 命名约定并存，导航名通过 `nav_translations` 字典翻译；一次 `mkdocs gh-deploy` 同时产出中英两个站点（英文站 csdiy.wiki/en/）。这是全仓唯一的"工程亮点"，让一个纯内容项目拥有了国际化能力。

**（4）社区评论层**（`overrides/partials/comments.html`）。Giscus 脚本按页面标题映射到 Discussions（data-mapping="title"），自定义 JS 监听 material 调色板切换事件同步评论区的明暗主题——这是全仓唯一一段手写 JS（约 40 行）。

## 6. 构建与运行

来自 `requirements.txt`、`.github/workflows/ci.yml` 与 README：

```bash
# 本地构建/预览
pip3 install -U -r requirements.txt
mkdocs serve        # 本地预览（material 标准命令）

# 部署（CI 同款，发布到 gh-pages，即 csdiy.wiki）
mkdocs gh-deploy --force
```

CI（`.github/workflows/ci.yml`）仅 3 步：checkout（fetch-depth: 0，供 git-revision-date 插件取提交时间）→ setup-python → `pip3 install -U -r requirements.txt && mkdocs gh-deploy --force`。无测试、无 lint 门禁。

## 7. 内容规模与更新机制

- **规模统计**（基于本地 tarball 实测）：docs 下 270 个 Markdown，中文 136 + 英文 134，双语覆盖率约 98%；中文正文合计约 355KB（`find docs -name "*.md" ! -name "*.en.md" | xargs wc -c`），折合十万余字。
- **分类与条目**：29 个一级分类中，课程类约 116 门（如 数据结构与算法 5 门、操作系统 4 门、系统安全 6 门、深度学习 11 门、机器学习系统 6 门），另设"必学工具"14 篇（Vim/Git/CMake/LaTeX/Docker/信息检索/毕业论文等）与"好书推荐"汇总页（`docs/好书推荐.md`，按"先中文后英文、先开源后闭源"的注释规范维护，条目附豆瓣链接）。
- **更新机制**：纯 PR 流程 + 人工 review。README 明确新课程需按 template.md 编写、在 mkdocs.yml 登记 nav、补英文翻译（引用 issue #222）；排版校对依据 issue #114 约定的中英混排规范。无任何自动化工具链（无链接检查 bot、无 CI lint），全靠维护者把关。
- **演进方向**：从导航树可见内容仍在扩张——2024-2025 年新增了机器学习系统、深度生成模型、大语言模型等前沿分类（`mkdocs.yml` nav 尾部）；最新 release v1.2.0 即"更新了深度生成模型学习路线"（`docs/index.md` 顶部公告）。
- **社区层**：Giscus 评论 + 读者自建学习群，使站点本身成为准社区产品（`README.md`、`overrides/partials/comments.html`）。

## 8. 社区与维护状态

- **活跃度**：创建于 2021-10-20，最近推送 2026-09-17（GitHub API），五年持续维护未断档；tarball 内全部文件 mtime 为 2026-09-17，与最近推送吻合。
- **贡献者结构**：167 位贡献者（GitHub API contributors 分页推算），典型"创始人主导 + 长尾贡献"模式——作者 PKUFlyingPig（Yinmin Zhong，北大信科，`LICENSE` 版权人）撰写主干，社区贡献新增课程页与勘误；contrib.rocks 徽章列于 README。
- **Issue/Release**：open issues 167 个，累计 5 个 release（GitHub API）；issue 除勘误外还被用作学习群入口（README 所述）。
- **发布节奏**：无固定版本周期，release 仅作为里程碑公告（v1.2.0 对应深度生成模型路线上线）；日常更新即 push 即部署（CI 直连 gh-deploy）。

## 9. 亮点与风险

**亮点**
1. 内容真实可信：全部课程评价基于作者/贡献者亲身学习经历（每页含难度、学时、踩坑提示），远超链接聚合的 awesome 类清单（`docs/计算机系统基础/CSAPP.md` 等全文可证）。
2. 体系完整成"书"：从工具、数学、编程入门到系统、AI、大模型形成完整知识图谱，且有"使用指南"按不同人群定制路径（`docs/使用指南.md`、`docs/CS学习规划.md`）。
3. 工程虽简但到位：双语 i18n 构建、Giscus 评论、明暗主题同步、自动修订日期，用最小依赖（6 个插件）做出专业文档站。
4. 贡献规范清晰：模板 + 混排规范 + 翻译要求 + nav 登记流程，使 167 位贡献者的内容风格保持统一（`template.md`、`README.md`）。
5. 品牌与社区效应：csdiy.wiki 简短好记，评论区和学习群形成自传播循环。

**风险/局限**
1. 链接腐化风险：大量依赖 B 站、Coursera、课程官网等外部链接，且无自动化链接检查，失效只能靠读者 issue 反馈。
2. 单点维护：核心导航与规划由作者一人定夺，review 瓶颈与个人视角偏差同时存在（"一个仅供参考的学习规划"作者自己也在标题中作了限定）。
3. 内容规模与深度不均：116 门课程页详略差异大，部分页面仅数 KB，个别方向（如数据科学仅 1 门）覆盖薄弱（由各文件体积与 nav 统计可见）。
4. 工作量门槛：全书路线预计学时极高（单 CSAPP 一门即 150 小时），对时间有限的读者需要自行取舍——使用指南部分缓解了该问题。

## 10. 总结评价

cs-self-learning 是"个人经验产品化"的典范：作者把自己四年的自学路径沉淀为一份结构化、双语化、社区化的开源课程地图，用最简单的 mkdocs 工程承载了极高的内容密度，五年来以 167 位贡献者之力持续扩张至约 130 个条目，成为中文 CS 自学圈事实上的入口级项目。它没有代码可"读源码"，其值得借鉴的是三件事——用统一模板保证众包内容质量、用 i18n 命名约定实现零成本双语、用 Giscus 把静态文档变成学习社区。适合所有计划自学计算机的学生作为路线参考，也适合任何想维护大型协作文档项目的人当作组织范本。
