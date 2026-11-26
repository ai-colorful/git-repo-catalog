# halfrost/LeetCode-Go 项目分析

> 星数 33000（快照 2026-10-24）｜主语言 Go｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：中文社区最系统的 **LeetCode Go 语言题解合集**，以"单题目录 + 中文解析 + 表驱动测试"的模式覆盖近 800 道题目，并借自建自动化管线把题解渲染成在线电子书《LeetCode Cookbook》（books.halfrost.com）与可下载 PDF。

| 项目 | 内容 |
| --- | --- |
| 名称 | halfrost/LeetCode-Go |
| 星数 | 快照 33000；gh api 实测 33,805、forks 5,677（2026-11-21） |
| 语言 | Go（go.mod 声明 go 1.19，`src/go.mod`） |
| 许可证 | 代码 MIT（`src/LICENSE`，2017 年起）；电子书内容 CC（README 徽章，未确认具体子类） |
| 创建 / 最近推送 | 2017-07-18 创建；最近 push 2026-10-12（gh api，快照活跃度核实通过） |
| 最新 release | v1.7.97（2026-06-28，电子书 PDF 版本，gh api） |
| 规模 | 6,381 个文件；leetcode/ 题解目录 798 个；Go 代码约 8.3 万行（其中题解约 7.9 万行）；测试文件 810 个 |
| 作者 | halfrost（一亩三分地社区知名作者，博客 halfrost.com，LeetCode 账号 id 8751） |

## 2. 项目是什么，解决什么问题

这个仓库解决的是"刷题者面对一道 LeetCode 题目，想看一份**可运行、可验证、带中文解析**的 Go 参考实现"的问题。作者 halfrost 自 2017 年起把每道题的解法沉淀为独立目录：每个目录固定三件套——`README.md`（题目原文 + 中文"题目大意"+ 中文"解题思路"）、题解 `.go`、同题 `_test.go`（如 `src/leetcode/0001.Two-Sum/` 下三个文件）。题解代码声称严格遵循 Google Golang Style Guide（`src/README.md`）。

与多数"题解仓库只贴代码"不同，本仓库的核心差异是**工程化**：所有题解都是同一个 Go module 里可编译、可测试的包（`package leetcode`），`bash ./gotest.sh` 一条命令即可跑完所有题的测试并产出覆盖率文件 `coverage.txt`（仓库根目录实际提交了 10,587 行的覆盖率 profile）。仓库简介宣称 "100% test coverage"，CI（`src/.github/workflows/test.yml`）会在日志中打印总覆盖率与所有未达 100% 的函数名，用透明度支撑这一声明。

第二个差异是**内容出版化**。题解不止停留在 Git 仓库里：`website/` 目录是一个 Hugo 站点，把全部题解编排成四章在线书籍《LeetCode Cookbook》，支持 PWA 安装与暗色模式（`src/README.md`；`src/website/config.toml`）；GitHub Actions（`src/.github/workflows/deploy.yml`）在 master 更新后自动构建并 rsync 到作者服务器，同时用 PAT 自动刷新 README 上的"总字数"徽章（当前 1,049,299 字）。README 明确这是 word-count 自动化零人工的产物（deploy.yml 注释）。

关于《剑指 Offer》：仓库**没有**独立的剑指专题章节，剑指题目以"扩展题"形式内嵌进相关题解代码中——例如 `src/leetcode/0215.Kth-Largest-Element-in-an-Array/215. Kth Largest Element in an Array.go` 第 53 行有 `// 扩展题 剑指 Offer 40. 最小的 k 个数` 及其实现 `getLeastNumbers`。全库 grep "剑指" 仅命中 PDF 源文件、215 题解与对应书页等少数位置，说明剑指覆盖是点缀性而非系统性。

此外仓库自带一套算法模板库：`src/template/`（树状数组 BIT、线段树 SegmentTree、并查集 UnionFind、LRU/LFU/CLRU Cache、桶）与 `src/structures/`（Heap、PriorityQueue、Stack、Queue、ListNode、TreeNode、Interval、Point、NestedInteger 等 9 种数据结构），每个均配 `_test.go`，可独立复用（各自带 go.mod 可作本地 module，主 `go.mod` 用 replace 指向本地目录）。

## 3. 技术栈与依赖

- **语言与版本**：Go 1.19（`src/go.mod`），测试工作流用 `go-version-file: go.mod` 锁定版本以"与题解代码年代匹配"（`src/.github/workflows/test.yml` 注释）。
- **直接依赖**（`src/go.mod`）：`spf13/cobra`（CLI 框架）、`BurntSushi/toml`（ctl 配置解析）、`mozillazg/request` + `bitly/go-simplejson` + `golang.org/x/net`（访问 LeetCode API 抓取题目数据）；题解本身零第三方依赖，全部标准库。
- **本地子模块**：`structures/`、`template/`、`ctl/util`、`ctl/models` 通过 go.mod `replace` 指向本地目录（`src/go.mod`），是"多模块单仓库"的小型实践。
- **网站**：Hugo 0.74.3 extended 版编译电子书（`src/.github/workflows/deploy.yml`，注释说明了必须用 extended 版编译 book.scss）；PWA 用 workbox（`src/website/workbox-config.cjs`）。
- **CI**：GitHub Actions 两个工作流——`test.yml`（Go 测试 + 覆盖率 + Codecov）与 `deploy.yml`（Hugo 构建 + rsync 部署 + 字数徽章刷新）。

## 4. 目录结构与代码组织

| 顶层目录 | 职责 |
| --- | --- |
| `leetcode/` | 题解主体：798 个题目目录（790 个 `NNNN.Problem-Name/` 编号目录 + 8 个 `999xxxx` 付费题内部 ID 目录，如 `9990085.Maximal-Rectangle/`） |
| `structures/` | 题解共用数据结构定义（9 种，各带测试，独立 go.mod） |
| `template/` | 算法模板（BIT/线段树/并查集/LRU/LFU 等，各带测试，独立 go.mod） |
| `ctl/` | 自研 CLI `leetcode-go`（cobra）：抓取 LeetCode 数据、渲染 Markdown、统计账户进度、生成 PDF |
| `website/` | Hugo 电子书站点（3,869 个文件，ChapterFour 下 827 个题目章节页），含主题、PWA、部署脚本 |
| `note/` | 学习笔记：`time_complexity.md`（复杂度速查）、`grokking_algorithms.md`（《算法图解》笔记） |
| `topic/` | 10 张算法专题图（Two Pointers/Sliding Window/Union Find 等 PNG） |
| 根目录 | `README.md`/`README_zh.md`（各 3,100 行中英双语目录索引）、`gotest.sh`、`coverage.txt`、`PDF v1.7.97.md`（79,122 行整本书 Markdown 源） |

组织方式评述：这是典型的**内容型单体仓库**——题解按"一题一目录"平铺（目录名 `编号.题目名` 保证了有序性与 URL 友好），而非按算法专题分组；专题维度由 README 的 3,100 行索引表和 Hugo 站点的 ChapterTwo（19 个算法专题页）提供。README 的题目总表列出 2,444 道题（与作者 LeetCode 账户数据同步，由 ctl 工具维护），其中 789 行带有指向题解目录的链接，其余约 1,655 行为规划中未完成的题（抽样可见多为 Hard，如 0010 Regular Expression Matching 无链接且无对应目录）——即"账面覆盖 2444，实际完成 798 目录/789 链接"，引用时需区分。

## 5. 核心架构与关键实现

### 5.1 题解的统一模式（抽样 5 题）

- `src/leetcode/0001.Two-Sum/1. Two Sum.go`：哈希表 O(n) 一遍扫描，函数式极简实现，`twoSum` 全部标准库。
- `src/leetcode/0005.Longest-Palindromic-Substring/5. Longest Palindromic Substring.go`：单文件给出 **4 种解法**（`longestPalindrome` 用 Manacher 算法 + `longestPalindrome1/2/3` 备选），注释风格示例——`// 解法一 Manacher's algorithm，时间复杂度 O(n)，空间复杂度 O(n)`，关键行有中文行注（"这一行代码是 Manacher 算法的关键所在"）。
- `src/leetcode/0215.Kth-Largest-Element-in-an-Array/215. Kth Largest Element in an Array.go`：quickselect 实现，并附剑指 Offer 40 扩展题。
- `src/leetcode/0052.N-Queens-II/52. N-Queens II.go`：回溯 + 列/对角线布尔数组标记。
- 每题 README 固定四节：`题目`（原题英文）→ `题目大意`（中文一句话）→ `解题思路`（中文分析）→ `代码`。

### 5.2 表驱动测试模式

所有测试共用一套惯例（`src/leetcode/0001.Two-Sum/1. Two Sum_test.go`）：定义 `question<N>` 结构体嵌入 `para<N>`（参数）与 `ans<N>`（答案）两个子结构体，测试函数 `Test_Problem<N>` 用 `qs := []question1{...}` 表驱动批量断言。该惯例使 810 个测试文件风格完全一致，也支撑了覆盖率目标。

### 5.3 自动化管线 ctl/

`src/ctl/command.go` 定义 cobra 根命令 `leetcode-go`，子命令 `build / pdf / refresh`：`request.go` 拉取 LeetCode 账户与题目数据，`statistic.go`/`rangking.go` 汇总进度，`template_render.go` 渲染 Markdown，`pdf.go` 生成 PDF。这条管线解释了 README 2,444 行题目表（含通过率、难度列）如何与作者 LeetCode 账户保持同步，以及 `PDF v1.7.97.md`（79,122 行）的由来——release 版本号即电子书版本。

### 5.4 电子书部署链

`src/.github/workflows/deploy.yml`：master push → Hugo 构建（`cd ./website && hugo -D --minify`）→ rsync 到服务器 `books.halfrost.com`；另有 word-count job 用仓库管理员 PAT 直接 push 回受保护的 master 刷新徽章（yml 注释详述了分支保护与 PAT 的取舍）。站点内容 `website/content/ChapterFour/` 有 827 个文件，即全书题目章节。

## 6. 构建与运行

```bash
# 运行全部题解测试并生成覆盖率文件（src/gotest.sh，唯一官方测试入口）
bash ./gotest.sh
# 等价于: go test -covermode=atomic -coverprofile=coverage.txt ./leetcode/...

# 查看总覆盖率与未达标函数（CI 中使用的检查方式，src/.github/workflows/test.yml）
go tool cover -func=coverage.txt | tail -n 1
go tool cover -func=coverage.txt | grep -v '100.0%'

# 构建电子书网站（需要 Hugo extended 0.74.3，src/.github/workflows/deploy.yml）
cd website && hugo -D --minify

# 本地 CLI（go.mod 替换已配好本地子模块）
go run ./ctl --help   # leetcode-go build / pdf / refresh 子命令（src/ctl/command.go）
```

注意：题解包内多数文件带 `package leetcode` 同名声明但文件名含空格与中文注释，属正常设计（Go 支持），无构建障碍。

## 7. 测试与工程质量（内容仓特写：覆盖声明的工程化方式）

- **规模**：810 个 `*_test.go` 对应 798 个题解目录（部分题含多解法测试），tests 与 solutions 同目录同包。
- **覆盖率机制**：`gotest.sh` 的注释专门解释了历史坑——旧的"每包分别 -coverprofile 再 cat"会产生多个 `mode: atomic` 头导致新 Codecov 解析为 0%，Go 1.10+ 对多包一次性产出单一 profile 解决（`src/gotest.sh`）。根目录提交的 `coverage.txt` 有 10,587 行真实 profile。
- **CI 执法**：`test.yml` 在 CI 日志打印"总覆盖率 + 所有低于 100% 的函数"，并上传 coverage-profile 工件与 Codecov（`fail_ci_if_error: false`，Codecov 故障不阻塞测试）。"100% coverage"由此变成可审计的声明而非口号。
- **质量观察**：测试全部是纯函数级表驱动，无 mock、无集成测试（题解本身也无需）；README 双语 3,100 行 ×2 全部手工维护的索引表 + 自动刷新的字数徽章，工程化程度在内容仓中罕见。

## 8. 社区与维护状态（含博客联动与维护节奏）

- **贡献者**：gh api 返回 10 位贡献者，halfrost 1,441 次贡献绝对主导；第二位 tphyhFighting 112 次、novahe 36、brenobaptista 20、tyd2000 12。本质是**单作者仓库**，外部贡献多为题解修订。
- **活跃度**：最近 push 2026-10-12（快照"最近推送 2026-10"核实一致）；最新 release v1.7.97 于 2026-06-28 发布（电子书 PDF 按 book 版本号发 release）；open issues 仅 4 个。仓库已连续维护 8 年以上（2017 至今），deploy.yml/test.yml 中大量 2025-2026 年代风格的中文注释说明近期仍在积极重构工具链。
- **博客联动**：作者博客 halfrost.com（README 徽章 "Blog-Halfrost-Field"）；题解电子书独立部署在 books.halfrost.com/leetcode/（PWA 可装到 iOS/Android 主屏），README 中的书截图来自 img.halfrost.com。博客与书是同一作者内容生态的两端，仓库是书的源码。
- **传播**：README 提供 PDF 下载徽章（GitHub Releases）、微信公众号二维码（`website/static/wechat-qr-code.png`），中文社区（知乎/微博/一亩三分地）均有作者账号入口。

## 9. 亮点与风险

**亮点**
1. 题解工程化标杆：一题一目录三件套（README/题解/测试）高度一致，810 个测试文件同一表驱动惯例，"100% coverage"有 CI 与提交的 coverage.txt 双重佐证（`gotest.sh`、`.github/workflows/test.yml`）。
2. 内容出版化闭环：自研 ctl CLI 抓 LeetCode 数据 → Hugo 电子书 → PDF release → 自动部署，全链路零手工（`ctl/`、`website/`、`deploy.yml`）。
3. 解析质量高：抽样 5 题均见中文"题目大意 + 解题思路"、多解法对比（0005 题 4 种解法）、复杂度逐解法标注、关键行注释。
4. 附赠可复用资产：`structures/` 与 `template/` 是带测试的独立 Go module（堆/并查集/线段树/LRU 等），可直接拷用。
5. 中英双语 3,100 行索引 + 专题分类表，检索性好。

**风险/局限**
1. 单作者结构：1,441/1,600+ 次贡献来自一人，长期可持续性依赖个人精力；题目表 2,444 vs 完成 789，大量"规划中"条目可能长期空置。
2. 技术栈偏旧：Go 1.19 锁定、Hugo 0.74.3（deploy.yml 注释自认 extended 版是"关键修复"），升级需动全量题解。
3. 剑指 Offer 仅以"扩展题"点缀（全库仅零星命中），并非系统性专题，引用仓库简介时需注意。
4. 8 个付费题用 `999xxxx` 内部 ID 命名（如 `9991044.Longest-Duplicate-Substring/`），与官方题号脱节，检索易混淆。

## 10. 总结评价

halfrost/LeetCode-Go 是"题解仓"这一品类里工程化与出版化做到极致的代表：它不只是代码合集，而是一个由 CLI 数据抓取、Hugo 站点、PDF 发布、CI 覆盖率执法组成的个人内容生产线，八年持续投入使 798 道题解在中文社区形成了事实上的 Go 刷题参考书。它适合三类人：准备面试刷 LeetCode 的 Go 学习者（直接按题号查目录）、想借鉴"内容仓如何工程化"的作者（看 ctl/ 与两条 workflow）、需要现成数据结构/算法模板的 Go 工程师（structures/ 与 template/）。想读懂"如何运营一个高星教育仓"的人，这个仓库本身的工具链比题解更值得读。
