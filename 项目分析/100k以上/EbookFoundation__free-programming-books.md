# EbookFoundation/free-programming-books 项目分析

> 星数 399,903（快照 2026-10-24）｜主语言 Python｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：GitHub 历史上最著名的免费编程学习资源清单，用 Markdown 收录全球 40+ 种语言的免费书籍、课程、交互教程、播客/ screencast 与练习题集，是 "awesome 清单" 文化的鼻祖级项目。

| 项目 | 事实 |
| --- | --- |
| 名称 | EbookFoundation/free-programming-books |
| 星数 | 399,903；fork 67,028；watch 9,768（gh api，2026-10-30） |
| 创建时间 | 2013-10-11（gh api） |
| 主语言 | Python（仓库内唯一可执行代码为 Python 检查脚本；内容主体是 Markdown） |
| 许可证 | CC-BY-4.0（`LICENSE`：Creative Commons Attribution 4.0 International） |
| 最近活跃 | 最近 push 2026-10-27（gh api），与快照日仅差 3 天，持续高频更新 |
| 贡献者 | 约 430+（gh api contributors 分页推算，未精确去重） |
| 形态 | 内容型仓库（多语言资源清单 + 质量保障脚本/CI），渲染为 GitHub Pages 静态站 |

## 2. 项目是什么，解决什么问题

互联网上的编程学习资料海量但良莠不齐，"免费" 的界定尤其混乱——很多搜索引擎结果指向需要注册邮箱、甚至变相付费的网盘页面。本项目用一个社区共同维护、机器校验的链接清单解决这个问题：只收录确认免费、可公开下载的书籍与课程，并按编程语言、主题和自然语言三个维度分类，让任何人能快速找到某种语言/某个主题下真正免费的学习资源（`README.md`、`docs/CONTRIBUTING.md`）。

项目起源于 StackOverflow 著名问答 "List of Freely Available Programming Books"，2013 年由 Victor Felder 移入 GitHub 协作维护（`README.md` Intro 段），后交由非营利组织 Free Ebook Foundation 管理。它是 GitHub Octoverse 统计中最受欢迎的仓库之一，也是 awesome 清单运动的标志性项目。

收录内容按资源形态分为六类，由 `docs/CONTRIBUTING.md` 明确定义，落点与规模（本次 HEAD 快照统计）：

| 资源类型 | 目录 | 文件数 | 条目数（`* [` 行） |
| --- | --- | --- | --- |
| Books（书籍） | `books/` | 49 | 5,568 |
| Courses（课程） | `courses/` | 41 | 5,274 |
| Podcasts & Screencasts | `casts/` | 21 | 602 |
| Interactive Tutorials / Playgrounds / Cheatsheets / Problem Sets | `more/` | 11 | 1,070 |
| 合计 | — | 122 | 约 12,514 |

规模（本次对 HEAD 快照实际统计）：`books/` 49 个文件 5,568 条、`courses/` 41 个文件 5,274 条、`casts/` 21 个文件 602 条、`more/` 11 个文件 1,070 条，合计 122 个清单文件、约 12,514 个条目，全清单共约 10,932 个外链（`grep -oE 'https?://...' | wc -l`）。除英语外，中文、西语、俄语、阿拉伯语等 40+ 语言各有独立清单；`books/free-programming-books-en.md` 仅是一个指向 langs/subjects 两个文件的历史跳转页。

条目还带有统一的辅助标记：失效资源经 Wayback Machine 恢复后标注 `archived`（如 langs.md 中 `:card_file_box: archived` 条目），未完书标注 `in process`，需要邮箱注册的下载页标注 `(email address *requested*, not required)`——这些约定全部写入 `docs/CONTRIBUTING.md`，机器与人共用同一套语义。

除主仓库外还有配套的动态搜索站（`README.md` 中链接到 ebookfoundation.github.io/free-programming-books-search），主仓库本身通过 Jekyll 渲染为静态站点（`_config.yml`），README 顶部内嵌 GitHub 表单式搜索框。语种覆盖示例：中文版 `books/free-programming-books-zh.md` 按 "语言无关/语言相关" 两大类组织，含 382 条；印地语、孟加拉语、缅甸语、高棉语等小众语种均有独立清单（`books/`、`courses/` 文件名后缀）。

## 3. 技术栈与依赖

这是一个内容型仓库，技术栈围绕 "Markdown 清单 + 自动化质检" 构建：

- **内容格式**：纯 Markdown（kramdown 方言，清单条目统一为 `* [标题](URL) - 作者 (格式)`，见 `books/free-programming-books-langs.md`）。
- **站点渲染**：Jekyll + GitHub Pages，`_config.yml` 指定 `remote_theme: pages-themes/minimal`，插件 jemoji、jekyll-relative-links；`_includes/head-custom.html` 注入自定义 head。
- **自研工具**：`scripts/rtl_ltr_linter.py`（Python 3.11，依赖 PyYAML、python-bidi），约 605 行，是仓库唯一的自研代码。
- **CI 工具链**：Ruby 的 `awesome_bot`（链接可用性检查）、npm 包 `free-programming-books-lint`（清单格式/字母序检查），均由 GitHub Actions 驱动（`.github/workflows/`）。
- **仓库维护自动化**：Dependabot（`dependabot.yml`，每周六更新 Actions 版本）、stale bot、冲突 PR 检测等。

没有 package.json/pyproject 等工程清单文件，依赖声明直接内联在各 workflow 中。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `books/` | 免费书籍清单：`free-programming-books-langs.md`（按编程语言，2,784 行 1,788 条）、`free-programming-books-subjects.md`（按主题，890 条）+ 47 个分语种文件（中文 791 行 382 条等） |
| `courses/` | 免费课程清单，41 个分语种文件（英语版 1,993 行 1,467 条） |
| `casts/` | 播客与 screencast 清单，21 个分语种文件 |
| `more/` | 交互教程、playgrounds、速查表、竞赛题集清单（11 个文件） |
| `docs/` | 94 个 md：CONTRIBUTING（24 个语言版）、CODE_OF_CONDUCT（33 个语言版）、HOWTO 新手指南（36 个语言版） |
| `scripts/` | 唯一代码：`rtl_ltr_linter.py` + 配置 `rtl_ltr_linter_config.yml` |
| `.github/` | 7 个 workflow、自定义 action `awesomebot-gh-summary-action`、PR 模板、Dependabot 配置 |
| `_config.yml`、`_includes/`、`favicon.ico` | GitHub Pages 站点元配置 |
| `README.md` | 项目门户：总索引、贡献方式、翻译目录 |

组织方式评述：按 "资源类型 × 语种" 的二维矩阵拆成 122 个扁平文件，每文件一个语种/主题，避免单文件膨胀；配套文档（贡献指南、行为准则、新手教程）全部多语化，与其 "多语言资源清单" 的定位一致。这是典型的单体内容仓库结构，无构建系统、无包管理，复杂度全部集中在 CI 质检上。

## 5. 核心架构与关键实现

### 5.1 清单条目格式规范

所有条目遵循统一文法（见 `docs/CONTRIBUTING.md` Formatting 节与 `books/free-programming-books-langs.md` 实例）：

```text
* [Bash Guide for Beginners (2008)](https://www.tldp.org/...) - M. Garrels (HTML)
```

即 `[标题](URL) - 作者 (格式)`，并约定 `in process`（未完书）、`archived`（Wayback Machine 保存）等标记。该正则文法同时被人（贡献者）与机器（linter、脚本）共用——`scripts/rtl_ltr_linter.py:94-99` 的 `BOOK_ITEM_RE` 就是它的程序化表达。严格字母序 + 固定空行规则让 12,000+ 条目可以用 diff 精确审查。

### 5.2 RTL/LTR 混排 Linter（仓库唯一实质代码）

`scripts/rtl_ltr_linter.py`（605 行）解决一个鲜有人会想到的问题：阿拉伯语、希伯来语等从右向左（RTL）书写的清单里混入英文书名、`C#` 等符号时，Unicode 双向算法会导致显示顺序错乱。核心设计：

- 逐行解析列表项，支持 `<span dir="rtl">` 嵌套切换方向上下文（`split_by_span`，`:125`）；
- 用 python-bidi 计算"视觉顺序"，与逻辑顺序比对发现 BIDI 错配（`:196` 的 `lint_file`）；
- 按五类问题分级（bidi_mismatch=error、keyword/symbol=warning、pure_ltr/author_meta=notice），级别可在 `rtl_ltr_linter_config.yml` 覆盖，内置 HTML/CSS/Python 等 60+ 需要补 `&rlm;` 的 LTR 关键词表；
- `main()`（`:445`）支持 `--changed-files`：只对 PR 改动行输出 GitHub Actions 行内注解，改动行有 error/warning 时退出码为 1。

多语言清单项目的这个痛点由本仓库用约 600 行代码优雅解决，这在 awesome 系清单中相当罕见。

### 5.3 CI 质检流水线

`.github/workflows/` 共 7 个 workflow，两条主线：

- `check-urls.yml`：对 push/PR 中**变更的** .md/.yml 逐文件起 matrix job（max-parallel 10），Ruby `awesome_bot --allow-redirect --allow-dupe --allow-ssl` 检查外链存活，结果 JSON 上传 artifact，再用自研 action `awesomebot-gh-summary-action` 汇总写进 GitHub Step Summary。增量检查是对万级外链仓库唯一可行的策略。
- `fpb-lint.yml`：npm 全局安装 `free-programming-books-lint`，对 books/casts/courses/more 执行格式与字母序检查，日志清洗后作为 artifact 反馈给贡献者。

社区自动化三件套：`stale.yml`（过期 issue 处理）、`comment-pr.yml`（PR 自动评论）、`detect-conflicting-prs.yml`（冲突 PR 检测），外加 `issues-pinner.yml` 置顶管理。

## 6. 构建与运行

仓库无需构建、无测试套件，"运行" 即 GitHub Pages 自动渲染：

- 本地预览（由 `_config.yml` 推断）：`bundle exec jekyll serve`（需 Jekyll 环境，README 未提供本地运行文档，此条为推断）；
- 在线使用：https://ebookfoundation.github.io/free-programming-books/ （静态站）与 free-programming-books-search（动态搜索站，独立仓库，本仓库内未见其源码，见 `README.md` 链接）；
- 本地跑质检（命令取自 workflow 文件）：
  - `fpb-lint books casts courses more`（Node，npm 包 free-programming-books-lint，见 `.github/workflows/fpb-lint.yml`）
  - `python scripts/rtl_ltr_linter.py books courses casts more`（Python 3.11 + PyYAML + python-bidi，见 `.github/workflows/rtl-ltr-linter.yml`）
  - `awesome_bot <file.md> --allow-redirect --allow-dupe --allow-ssl`（Ruby gem，见 `.github/workflows/check-urls.yml`）

## 7. 内容质量保障机制（测试与工程质量的特例改写）

- **准入规则**：`docs/CONTRIBUTING.md` 定义了相当完整的收录标准——必须免费、拒绝网盘直链（Google Drive/Dropbox/Mega 等）、优先作者官网、https 优先、URL 去尾斜杠取最短形式、老书标注出版年、失效链接用 Wayback 标注 `archived`。PR 模板要求贡献者回答 "How do we know it's really free?"（`.github/PULL_REQUEST_TEMPLATE.md`）。
- **机器校验**：字母序与格式由 fpb-lint 强制；外链存活由 awesome_bot 增量检查；RTL 语种清单由自研 linter 检查显示顺序。三道闸全部卡在 PR 阶段。
- **人工复审**：PR 模板 + 维护者复审 + Hacktoberfest 参与记录（README 顶部的 Hacktoberfest 2025 徽章）。
- **工程质量观察**：作为内容仓库，其 "质量体系" 完成度高于多数代码仓库；唯一弱点是 `check-urls.yml` 中 `awesome_bot ... || true`——链接检查结果只写 artifact/Summary 不阻断合并，坏链清理依赖人工跟进（`.github/workflows/check-urls.yml`）。

## 8. 社区与维护状态

- 活跃度：最近 push 2026-10-27（gh api），fork 67,028、开放 issue 仅 89 个，说明维护者消化能力强、积压少。
- 贡献者：约 430+ 名（gh api contributors 分页 last=429 推算），是典型的大众众包模式；HOWTO 文档专门面向第一次发 PR 的新手（`docs/HOWTO.md`），Hacktoberfest 长期参与。
- 治理：由 Free Ebook Foundation（美国非营利组织，官网 ebookfoundation.org）管理，接受美国可抵税捐赠（`README.md`）；贡献即同意 CC-BY-4.0 与 Contributor Covenant 行为准则；`docs/CODE_OF_CONDUCT.md` 基于 Contributor Covenant 1.3 修改。
- 多语言社区：贡献指南 23 个语言版、行为准则 32 版、新手指南 35 版（`docs/` 统计），每个语种清单背后都有对应语言的维护社区。
- 无 release 概念（内容仓库，不适用版本发布）；Dependabot 每周自动更新 Actions 依赖。

## 9. 亮点与风险

亮点：

1. 12,500+ 条目、40+ 语言、十年持续维护的免费编程资源聚合，覆盖广度和时效性无可替代。
2. 六类资源定义清晰（books/courses/tutorials/playgrounds/casts/problem sets），三维度分类（语言、主题、自然语言）检索友好。
3. 增量式 CI 质检体系（格式 + 字母序 + 链接存活 + BIDI 显示）把万级链接清单的维护成本降到可控。
4. 自研 RTL/LTR linter 是多语言清单项目的独特贡献，约 600 行代码解决真实的 Unicode 双向排版痛点。
5. 贡献文档极致多语化，吸纳全球新手 PR，形成自增强的社区飞轮。

风险/局限：

1. 链接腐烂是内容清单的宿命：URL 检查不阻断合并（`|| true`），部分坏链只能靠 issue 驱动清理。
2. 收录质量依赖人工判断 "是否真免费"，个别条目可能随源站变动失效或变相收费。
3. 单文件仍偏大（langs.md 近 2,800 行、courses-en 近 2,000 行），GitHub 网页端加载与 diff 审查有压力。
4. 对初学者而言清单只是链接目录，缺少评价/难度标注，无法替代体系化学习路径。
5. CC-BY-4.0 要求署名，商用二次分发需保留归属（`LICENSE`），比代码仓库的 MIT/Markdown 镜像站复用门槛略高。

## 10. 总结评价

这不是一个 "代码项目"，而是 GitHub 上运转最成功的众包内容工程：用清晰的文法定义、严格的字母序纪律、增量式 CI 质检和 40 多个语言社区，把一个 2013 年的 StackOverflow 答案养成了近 40 万星、12,500+ 条目的全球免费编程资源索引。它适合所有编程学习者作为找书/找课的第一站，也适合任何想运营大型 awesome 清单的人当教科书来研究——重点读 `docs/CONTRIBUTING.md` 的规则设计、`.github/workflows/` 的质检流水线，以及 `scripts/rtl_ltr_linter.py` 这个多语言清单特有的工程巧思；想读 "源码" 的人，读到的将是一套经过十年打磨的社区协作与自动化治理机制。
