# azl397985856/leetcode 项目分析

> 星数 55721（快照 2026-10-24）｜主语言 JavaScript｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：中文互联网影响力最大的个人 LeetCode 题解集之一（"力扣加加"/作者 lucifer/西法），以"思路+关键点+多语言代码"的统一模板沉淀了约 400 篇题解和 30 余篇数据结构与算法专题长文，并配套 GitBook 电子书、Anki 卡片、91 天学算法活动等衍生生态。

| 事实 | 内容 |
| --- | --- |
| 仓库 | azl397985856/leetcode（master 分支） |
| 星数 | 55721（快照 2026-10-24）；fork 9336、watch 1307（gh api 2026-11-03） |
| 创建时间 | 2018-03-30（gh api repos/azl397985856/leetcode） |
| 最近推送 | 2025-07-16（gh api pushed_at，与仓库内最新题解编号 3599 的时间线吻合） |
| 主语言 | JavaScript（题解代码以 Python/JS 为主，多语言并存） |
| 许可证 | CC BY-NC-ND 4.0（LICENSE.txt，gh api 显示 NOASSERTION） |
| 贡献者 | 78 人（gh api contributors 翻页统计） |
| 内容规模 | 约 100947 行 Markdown 正文（problems/thinkings/selected/daily 四目录合计） |

## 2. 项目是什么，解决什么问题

这是算法面试学习者的中文题解仓库。它解决的核心问题是：网上题解质量参差、缺少统一的中文叙述结构。仓库把每道题都按固定模板组织——题目地址、题目描述、前置知识、高频公司、思路、关键点、多语言代码，读者可以按同一套认知路径复刷任意题目（见 `problems/1.two-sum.md`）。README 明确仓库分五个部分：经典题解、数据结构与算法专题总结、Anki 记忆卡片、每日一题活动记录、后续计划（README.md 第 62-72 行）。

与代码型项目不同，这是典型的内容型仓库：主体是 544 个 Markdown 文件，代码以代码块形式内嵌在题解中，独立成文件的源码只有 `daily/answers/` 下 16 个 JS 解答文件。题解覆盖按简单/中等/困难三级在 README 中建立索引（README.md 第 192-580 行），并标注"91"（出自 91 天学算法活动）与👍（作者推荐）标签。

仓库的第二主体是 `thinkings/` 专题：链表、树、堆（上下）、二分（上下）、动态规划、滑动窗口、位运算、并查集、单调栈、前缀树、布隆过滤器等 31 个主题（`ls thinkings`，中文 31 篇＋同名 .en.md），写法是教学式的——先讲基础结构（如 `thinkings/monotone-stack.md` 从"栈是什么"讲起），再推导模板，最后给推荐题目链接，相当于一本免费的中文算法讲义。

作者把仓库做成了一个学习品牌矩阵：GitBook 电子书《西法的刷题秘籍》（`book.json`、SUMMARY.md）、Anki 卡包（`assets/anki/leetcode.apkg`）、面试 cheatsheet PDF（`assets/cheatsheet.pdf`）、91 天学算法付费活动、Chrome 插件 leetcode-cheatsheet、出版书《算法通关之路》（README.md 第 15-25 行）。README 的大事件时间线记录了成长轨迹：2019-07 破 1 万星（`thanksGiving.md`）、2019-10 破 2 万、2020-04 破 3 万、2021-02 破 4 万（README.md 第 601-612 行）。

内容时效性良好：题解编号已覆盖到力扣 2025 年新题（如 `problems/3599.partition-array-to-minimize-xor.md`、`problems/3428.maximum-and-minimum-sums-of-at-most-size-k-subsequences.md`），说明作者在 2025 年仍在按周赛节奏更新，与 pushed_at 2025-07-16 相互印证。

作者对"怎么学"也有明确主张：`thinkings/README.md` 开篇给出学习心法——先掌握基础数据结构与暴力法、理解暴力法的瓶颈，再按"搜索（BFS/DFS/二分本质也是搜索）→ 剪枝 → 空间换时间（哈希表、前缀树）"的路径逐步逼近最优解。`91/README.md` 则阐述了 91 天学算法的动机：反对"7 天搞定算法"的速成宣传，用 91 天长周期 + 强制打卡来保证量变到质变，往期讲义（二分、双指针）直接开源在本仓库 `91/` 目录下。

## 3. 技术栈与依赖

- **内容主体**：Markdown（544 个 .md 文件）。内嵌代码块按语言统计（problems+thinkings+daily+selected 全量 grep）：Python 约 1041 个（py 605 + python 423 + Python 13）、JavaScript/JSX 约 386 个、Java 172、C++ 74、C 106、Go 22、PHP 18、TypeScript 2——官方标榜 JS，实际 Python 占比最高，是名副其实的多语言题解集。
- **电子书工具链**：GitBook 旧版，`book.json` 定义书名《西法的刷题秘籍》、语言 zh-hans、katex（公式）与 ace（代码高亮）插件；`package.json` 仅一个脚本 `book`（gitbook epub/pdf/mobi 导出），无 dependencies，`yarn.lock` 存在但无锁定的运行时依赖。
- **站点托管**：GitHub Pages（`_config.yml` + `.nojekyll`），在线阅读走 GitBook 站点（README.md 第 78 行）。
- **CI/机器人**：GitHub Actions 两个 workflow——`ci.yml`（PR 触发 markdown lint）与 `calibreapp-image-actions.yml`（PR 自动压缩图片）；另配置 stale 机器人（`.github/stale.yml`）。
- **图片源文件**：59 个 .drawio + 3 个 .excalidraw（README.md 第 619 行说明可用 draw.io 编辑）。
- 实质上没有"运行时"：这是一个纯内容仓库，技术栈只服务于排版、发布与协作。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `problems/` | 400 篇单题题解（文件名 `题号.题名.md`），46 篇有英文版 `*.en.md` |
| `thinkings/` | 31 个算法/数据结构专题长文（中文 31 + 英文 31） |
| `selected/` | 13 篇精选深度文章（字节面试题、LIS/LCS/LSS 系列等） |
| `daily/` | 46 篇每日一题活动记录 + `answers/` 16 个 JS 独立解答文件 |
| `91/` | 91 天学算法活动讲义（二分、双指针、season2） |
| `collections/` | 按难度分的题目合集页（easy/medium/hard） |
| `templates/` | 题解模板（`templates/problems/1014.best-sightseeing-pair.md`）与每日一题模板 |
| `backlog/` | 待写计划（动态规划等） |
| `assets/` | 218 张图片（png 120/jpg 77/gif 14/svg 4 等）、59 个 drawio 源、Anki 卡包、cheatsheet PDF |
| `.github/` | CI、issue 模板（daily-problem、translation）、stale 配置、FUNDING、格式检查脚本 |
| 根目录 | README.md（中文索引主入口）、README.en.md、SUMMARY.md（GitBook 目录）、book.json、CONTRIBUTING.md |

组织方式评述：扁平的单体内容仓库，没有构建系统与包管理。命名约定是唯一的结构化手段——题号前缀排序、`.en.md` 后缀区分语言、`thinkings/` 与 `problems/` 分离"方法论"和"单题"。索引三层冗余（README 索引、collections 合集、SUMMARY.md 电子书目录）需要人工同步，这是该组织方式的主要维护成本。

## 5. 核心架构与关键实现

内容型仓库没有传统架构，核心资产是它的"内容生产协议"：

**（1）题解统一模板**（出处：`templates/problems/1014.best-sightseeing-pair.md` 与 `problems/1.two-sum.md`）
固定八段式：题目地址 → 题目描述 → 前置知识 → 公司 → 思路 → 关键点 → 代码（多语言）→ 参考文献。以 `1.two-sum.md` 为例，"思路"从暴力 O(N^2) 写起再推导哈希表优化，"关键点"提炼为三行口诀（"求和转换为求差"、"以空间换时间"）。这种"先暴力再优化"的叙事是全仓库 400 篇的一致风格。

**（2）多语言代码规范**（出处：`problems/1.two-sum.md` 第 68-120 行、`CONTRIBUTING.md`）
每篇题解声明"语言支持：JS, Go, CPP, Java, Python"，代码块按 `js/go/cpp/java/py` 标注。CONTRIBUTING.md 要求"思路一致、命名可按语言最佳实践（python 蛇形、JS 驼峰）"，并给出参考 PR 链接。抽样看 `1.two-sum.md` 中 JS 用 `Map`、Go 用 `map[int]int`、C++ 用 `unordered_map`，确实做到了算法等价、习惯各自地道。

**（3）专题教学文**（出处：`thinkings/monotone-stack.md`）
先讲底层结构（栈的定义、复杂度、应用场景）→ 引出受限变体（单调栈的定义与合法示例）→ 给出模板 → 推荐配套题目（394、946、1381）。专题文与单题解通过"题目推荐"链接互通，构成网状知识图谱；`SUMMARY.md` 把它们组织成 GitBook 三章结构（算法专题 / 91 天学算法 / 精选题解）。

**（4）格式自动化防线**（出处：`.github/workflows/ci.yml`、`.github/check-format.js`）
PR 到 master 触发 mdl（markdown lint，风格定义在 `.mdl_style.rb`、经 `.mdlrc` 启用）对变更文件做 lint，再跑 `node .github/check-format.js`。值得注意的是 check-format.js 的主体逻辑全部被注释掉，实际只剩 `console.log('TO BE IMPLEMENTED')`——格式校验实际只靠 mdl，作者预留的深度检查从未实现。

**（5）多入口索引体系**（出处：`collections/easy.md`、`README.md`、`SUMMARY.md`）
`collections/easy.md` 是 README 简单难度合集的镜像（内容几乎逐句相同，仅链接改为 `../problems/` 相对路径），medium/hard 合集同构。这套"README 主索引 + collections 难度合集 + SUMMARY 电子书目录"的三入口设计服务于不同阅读场景（浏览仓库 / 按难度刷题 / 电子书阅读器），代价是每次新增题解要在三处手工同步（CONTRIBUTING.md 也只要求同步 README.md 与 README.en.md 两处）。

**代表性内容抽样**（供直接阅读验证）：
- `problems/1.two-sum.md` —— 模板的完整范本，暴力→哈希的多解法递进
- `problems/42.trapping-rain-water.md` —— 困难题代表（接雨水，README 标注👍）
- `thinkings/monotone-stack.md` —— 专题文风格：从"栈是什么"讲到单调栈模板
- `selected/atMostK.md` —— 精选深文"一次搞定前缀和"，作者招牌写法
- `templates/problems/1014.best-sightseeing-pair.md` —— 官方贡献模板，新题解以此为骨架

## 6. 构建与运行

本仓库无需构建即可"运行"（阅读即使用）。实际可用的命令：

```bash
# 电子书导出（来自 package.json scripts.book，需全局安装旧版 gitbook-cli）
gitbook epub . && gitbook pdf . && gitbook mobi .

# Markdown lint（来自 .github/workflows/ci.yml）
gem install mdl
mdl <变更文件列表>
node .github/check-format.js   # 目前为空实现，仅打印 TO BE IMPLEMENTED
```

在线阅读不依赖本地构建：GitHub Pages（`.nojekyll` 存在）与 GitBook 站点（README.md 第 78 行、gh api homepage 字段）。图片可从 `assets/drawio/` 用 draw.io 打开再编辑（README.md 第 619 行）。

## 7. 内容质量与工程化实践

内容类型与规模一览（均来自本仓库实际文件统计）：

| 内容类型 | 位置 | 规模 | 说明 |
| --- | --- | --- | --- |
| 单题题解 | `problems/` | 400 篇（46 篇含英文版） | 文件名 `题号.题名.md`，八段式模板 |
| 算法专题 | `thinkings/` | 31 个主题（中英各 31 篇） | 教学式长文，附推荐题目 |
| 精选深文 | `selected/` | 13 篇 | 字节面试、LIS/LCS/LSS 系列等 |
| 每日一题 | `daily/` | 46 篇记录 + `answers/` 16 个 JS 文件 | 2019-06 至 2019-10 的群活动存档 |
| 活动讲义 | `91/` | 3 份 | 91 天学算法第一/三期（二分、双指针） |
| 难度合集 | `collections/` | 4 页 | easy（中英）/medium/hard 索引页 |
| 配套资料 | `assets/` | 218 张图、59 个 drawio、Anki 卡包、cheatsheet PDF | 仓库体积的主要来源（tarball 71.6MB） |

- **质量一致性**：统一模板 + mdl lint（`.mdlrc`、`.mdl_style.rb`）保证基础排版；PR 粒度的图片自动压缩（calibreapp/image-actions）控制仓库体积。深度格式校验（`.github/check-format.js`）是空壳，内容质量实际依赖作者本人审校。
- **内容深度抽样**：`problems/1.two-sum.md`、`thinkings/monotone-stack.md`、`problems/3599.partition-array-to-minimize-xor.md` 均做到"多解法递进 + 复杂度分析 + 多语言代码"，与同类题解仓相比叙事结构明显更规范。
- **双语覆盖不完整**：problems 400 篇仅 46 篇有英文版，thinkings 31 篇全有英文版；翻译通过 issue 认领制推进（`.github/ISSUE_TEMPLATE/translation.md`、CONTRIBUTING.md 第 3-8 行）。
- **图片治理成本可见**：assets 约 218 张图 + 59 个 drawio 源文件随仓库走，tarball 71.6MB 中绝大部分是图片；README 内另有大量 `p.ipic.vip` 图床外链（README.md 第 3-5 行等），存在图床失效风险（作者甚至在 README 专节提示"图片加载不出来如何解决"并链接自己的科学上网仓库，README.md 第 27-29 行）。

## 8. 社区生态与维护状态

- **更新节奏**：gh api 显示 pushed_at 2025-07-16；README 索引含 2024-2025 年力扣周赛题（2817、2865、2939、3377、3404、3428、3599 等），2023 年后基本是作者单人低频维护，但未停止。当前 open issues 仅 13（gh api）。
- **星数里程碑**（出处 README.md 第 601-612 行"大事件"与三篇致谢文）：2019-07-10 破 1 万（`thanksGiving.md`，文中回顾 2017-05-30 个人刷题占位仓起源）；2019-10-08 破 2 万，GitHub 搜索"LeetCode"排名第一（`thanksGiving2.md`）；2020-04-12 破 3 万（`thanksGiving3.md`）；2020-04-14 官网 leetcode-solution.cn 上线；2021-02-23 破 4 万。从 4 万到当前 55721 用时约五年，增速放缓但仍高于绝大多数同类仓。
- **贡献者结构**：78 名贡献者（gh api contributors 翻页），另有独立致谢页 `thanksGiving.md/thanksGiving2.md/thanksGiving3.md` 记录里程碑。贡献入口有三条：新增题解、多语言代码、翻译认领（CONTRIBUTING.md），issue 模板覆盖每日一题与翻译两种场景。
- **商业化与生态**：FUNDING.yml 挂 Patreon（fe_lucifer）；README 导流至微信公众号《力扣加加》、91 天学算法付费活动、1V1 辅导、Chrome 插件、出版书与官网 leetcode-solution.cn（README.md 第 39-55、608 行）。这是典型的"开源内容 + 个人品牌"模式，仓库即流量入口。
- **自动化社区运营**：stale 机器人 60 天无活动自动标记、7 天后关闭（`.github/stale.yml`），解释了 open issue 常年极低的现象。

## 9. 亮点与风险

**亮点**
1. 模板化的题解体系是同类仓库中完成度最高的：八段式结构 + 多语言代码 + "先暴力再优化"叙事，400 篇保持了惊人的一致性（problems/ 目录与模板文件可证）。
2. 内容分层清晰：单题解（problems）→ 方法论专题（thinkings）→ 精选深文（selected）→ 活动讲义（91），网状链接构成可自学的知识图谱。
3. 持续维护近八年，题解覆盖到 2025 年新题，未像多数题解仓那样停更。
4. 工程化意识超出内容仓平均水平：mdl lint、图片自动压缩、issue 模板、stale 机器人、翻译认领制一应俱全。
5. 衍生生态完整（电子书/Anki/插件/书籍/付费营），把"刷题"做成了闭环学习产品。

**风险/局限**
1. 许可证为 CC BY-NC-ND 4.0（禁止商用、禁止演绎再分发），比常见 MIT 内容仓限制严格，二次利用需注意（LICENSE.txt）。
2. 索引三处冗余（README/collections/SUMMARY.md）纯人工同步，README 中已出现个别失效路径（如 README.md 第 501 行 `problems/57.insert-interval.md` 写成了相对无 `./` 的 `problems/...`，第 545 行 1345 题文件名是 `1435.jump-game-iv.md`）。
3. 图片双重来源（仓库 assets + p.ipic.vip 图床），图床外链失效率高，长期可读性受损。
4. CI 中的深度格式检查是未完成的占位实现（`.github/check-format.js`），内容正确性（题解代码是否有 bug）无自动化保障。

## 10. 总结评价与适合人群

azl397985856/leetcode 是"个人内容仓库做成 55k 星"的代表样本：它赢的不是代码，而是一套可复制的内容生产协议（统一模板 + 分层组织 + 社区认领 + 自动化防线），以及作者近八年不间断的个人投入与品牌化运营。对准备国内算法面试的学习者，它是中文题解的第一梯队选择；对想学习"如何运营技术内容仓库"的人，它的模板设计、贡献流程与生态导流方式本身就是很好的研究对象；对想读"源码"的工程师，这里没有架构可读，值得读的是 `CONTRIBUTING.md` + `templates/` + `.github/` 这套轻量但完整的内容工程化配置。
