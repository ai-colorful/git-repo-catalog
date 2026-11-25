# practical-tutorials/project-based-learning 项目分析

> 星数 287710（快照 2026-10-24）｜主语言 Python（GitHub 标记，实际主体为 Markdown 内容）｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：一份按编程语言分类的"做项目学编程"教程精选清单，只收录免费、无墙、跟随做完能得到一个完整成品的教程。

| 项目 | 值 | 出处 |
|---|---|---|
| 名称 | practical-tutorials/project-based-learning | gh api 元数据 |
| 星数 | 287,710（快照 2026-10-24；2026-10-30 实查 288,384，仍在增长） | gh api |
| Forks / Watchers | 36,751 / 3,461 | gh api |
| 主语言 | Python（唯一代码文件 `scripts/check_readme.py`；内容主体为 Markdown） | 仓库文件 |
| 许可证 | MIT（`LICENSE.md`，Copyright (c) 2021 practical-tutorials） | LICENSE.md |
| 创建时间 | 2017-04-12 | gh api |
| 最近 push | 2026-10-26（分析前 4 天，持续活跃） | gh api |
| 默认分支 | master；仓库共 14 个文件，约 460KB | tarball 解包统计 |

## 2. 项目是什么，解决什么问题

这是一个典型的"curated list"内容型仓库，但定位非常聚焦：README.md（`README.md`，676 行、约 52KB）本身就是全部交付物——一份手工整理的教程索引，每个条目 `- [标题](链接)` 都指向一篇"带你从零做一个应用"的教程，例如用 C 写解释器（craftinginterpreters.com）、写一个 Shell、写一个哈希表、用 Flutter 复刻 TikTok 等。

它解决的痛点是：网上编程教程海量，但多数是概念讲解或收费课程；初学者想找"照着做完就能有一个拿得出手的成品"的免费教程，筛选成本很高。CONTRIBUTING.md 明确了两条硬性收录标准：一是"免费开放——无付费墙、无登录墙、不强制订阅"，二是"project-based——跟随教程能做出一个完整、可运行的成品（而不是概念解释）"。

内容按主语言分为 22 个一级分类（C/C++、C#、Clojure、Dart、Elixir、Erlang、F#、Go、Haskell、HTML/CSS、Java、JavaScript、Kotlin、Lua、OCaml、PHP、Python、R、Ruby、Rust、Scala、Swift）外加一个 Additional Resources（`README.md` 目录与正文标题）。多部曲教程用嵌套缩进列表组织（如 10 部的 "Writing a Linux Debugger" 系列）。

该项目 2017 年创建，是 GitHub 历史上最早、星数最高的教程清单之一。它不提供任何学习内容本身，价值完全在于"编辑把关 + 长期防死链维护"——这也是下文自动化体系的意义所在。

## 3. 技术栈与依赖

- 内容层：纯 Markdown（README.md），零依赖、零构建，任何阅读器可直接消费。
- 工具层：Python 3 标准库。`scripts/check_readme.py`（1179 行）文件头注释明确声明"Python 3 stdlib only -- no third-party dependencies to audit"（出处：`scripts/check_readme.py` 第 6 行）。唯一的第三方接触面是 GitHub Actions 官方 action（checkout、actions/stale）。
- CI 层：3 个 GitHub Actions workflow（`.github/workflows/`），用 `gh` CLI 完成发 Issue/发 PR 的自动化。
- 协作层：Gitter 聊天室（README 徽章与 `.github/ISSUE_TEMPLATE/config.yml`）。

没有 package.json / requirements.txt 等工程清单——这个仓库刻意保持零依赖。

## 4. 目录结构与代码组织

| 路径 | 职责 |
|---|---|
| `README.md` | 唯一内容交付物：676 行教程清单（22 个语言分类 + Additional Resources） |
| `CONTRIBUTING.md` | 贡献标准（免费/项目式两条硬标准、格式规范、本地校验命令） |
| `LICENSE.md` | MIT 许可证 |
| `scripts/check_readme.py` | 1179 行 README 校验/链接巡检工具（lint/check-diff/check-links/report/prune 五个子命令） |
| `scripts/linkcheck-domains.txt` | 会封锁爬虫的域名策略表（medium、reddit、x、linkedin、udemy 等 9 条） |
| `scripts/lint-allow-duplicates.txt` | 允许重复出现的 URL 白名单（带原因注释，共 2 条） |
| `.github/link-rot-state.json` | 链接巡检状态库：456 个 URL 的二振（2-strike）记录，119KB |
| `.github/workflows/validate-pr.yml` | PR 校验工作流 |
| `.github/workflows/link-rot.yml` | 每周链接巡检工作流 |
| `.github/workflows/stale-prs.yml` | 每日 PR 保活工作流 |
| `.github/ISSUE_TEMPLATE/*.yml` | 死链报告、教程建议两个表单化模板；禁用空白 Issue |

组织方式评述：单文件内容 + 单脚本工具的极简结构。内容（README.md）与治理（脚本、CI、模板）完全分离，内容可独立消费，治理可独立演进。仓库历史上有过 `.github/stale.yml`（probot 版），现已被注释说明为何废弃（`stale-prs.yml` 第 1-4 行：旧配置"把正当的死链报告埋掉了好几年"）。

## 5. 核心架构与关键实现

作为内容型仓库，"核心实现"有两块：内容组织方式，以及守护内容的自动化工具。

### 5.1 内容组织：README 即数据库

README 采用严格的可解析文法：`## 语言:` 一级分类、`### 子方向` 二级分类、`- [Title](url)` 条目、嵌套缩进表示多部曲系列（`README.md` 全文；格式规范见 `CONTRIBUTING.md`）。这套文法是 `scripts/check_readme.py` 的 `classify_line()`（第 101-133 行）定义的——注释里的设计思想是"README-as-data"：把 README 当结构化数据解析，lint、PR diff 检查、每周巡检三个功能共用同一套解析器，"so lint checks, PR-diff checks, and the weekly link-rot sweep can never drift apart"（`scripts/check_readme.py` 第 2-6 行）。条目规模：480 个链接条目行、约 462 个外链、23 个 H2 标题（实仓统计：grep 统计 README.md）。

### 5.2 五合一校验工具 check_readme.py

单文件、五个子命令（`scripts/check_readme.py` 的 argparse 定义在第 1129-1168 行）：

- `lint`：全文校验——文法（E001）、系列无子条目（E002）、目录锚点与 H2 标题双向对账（E003/E004）、重复 URL（E005）、短链域名黑名单 18 个（E102，第 43-47 行）。
- `check-diff`：只校验 PR 新增行（解析 unified diff），新增 `http://` 直接报错 E101。
- `check-links`：并发（默认 8 线程）存活检查，HEAD 失败按 400/403/405/501 降级 GET，重试 2 次并遵守 Retry-After，维护 2-strike 状态库。
- `report`：渲染成 Markdown 表格发到 Issue。
- `prune`：安全自动修复——只删"连续 2 周确认死亡且无重定向"的条目、只改写有新地址的重定向 URL；系列条目子链接全部删除时级联删除父标题（`cmd_prune`，第 1026-1100 行）。

几处值得注意的工程细节：

```python
# scripts/check_readme.py 第 8-11 行
# SECURITY INVARIANT: README.md content and PR diffs are untrusted input (this
# runs against fork PRs). Never eval/exec it, never pass it to a shell, never
# use it to build a command line. Only urllib/http.client network calls and
# plain string/regex parsing are allowed on that data.
```

- 慢速滴漏防护：`_read_bounded()`（第 539-560 行）用 `read1()` 代替 `read()` 并强制总墙钟 deadline，注释解释了纯 socket 超时挡不住"每次只滴几字节"的服务器。
- 域名限速：`DomainLimiter`（第 717-743 行）对同一注册域最多 2 并发、最短间隔 1 秒，避免巡检自己变成攻击者。
- 报告防注入：`escape_md_cell()`（第 887-901 行）转义 `|` 并给 `@` 插零宽字符，防止"来自当前占坑服务器的页面标题"在公开 Issue 里破坏表格或触发 @mention 通知——攻击面分析写在注释里。
- 软 404 识别：解析响应页 `<title>` 匹配 "page not found" 等短语判 SUSPECT（第 49-53、685-689 行）。

### 5.3 三个工作流构成的治理闭环

- `validate-pr.yml`：对每个改动 README 的 PR 做三步——全文 lint、新增行 check-diff、新增链接存活试查，全部结果写入 Step Summary（显式告知 BLOCKED 表示"域名拒绝自动化检查，请人工验证"）。
- `link-rot.yml`：每周一 06:17 UTC（cron `17 6 * * 1`）全量巡检 456 个 URL：提交状态库 diff、把安全修复推成 `automated/link-rot-fixes` 分支 PR、把完整报告 upsert 到单个"🔗 Link rot report" Issue（第 27-114 行）。状态库当前记录：OK 367、BLOCKED 77、HARD_DEAD 10、SUSPECT 2（`.github/link-rot-state.json`，2026-10 快照）。
- `stale-prs.yml`：每日标记 90 天无活动的 PR、再 30 天后关闭；Issue 明确豁免（`days-before-issue-stale: -1`），理由写在文件头注释里。

## 6. 构建与运行

无构建、无安装。可用的实际命令（出自 `CONTRIBUTING.md` 第 24-31 行与 workflow 文件）：

```bash
# 本地校验（提 PR 前必须退出码为 0）
python3 scripts/check_readme.py lint

# 校验一段 diff（CI 中对 PR 使用）
git diff --unified=0 origin/master...HEAD -- README.md | python3 scripts/check_readme.py check-diff

# 全量链接巡检（写状态库）
python3 scripts/check_readme.py check-links --all --state .github/link-rot-state.json
```

消费侧"运行"方式即打开 README.md 阅读或 fork 后自行维护。

## 7. 内容体系：收录范围、分类与条目规模

收录范围：22 种语言/技术的项目式教程 + 1 个综合资源区。条目按语言分布高度不均（grep 实测 `README.md`）：

| 语言 | 条目数 | 语言 | 条目数 |
|---|---|---|---|
| Python | 129 | PHP | 14 |
| C/C++ | 98 | Dart(Flutter) | 13 |
| HTML/CSS | 73 | Ruby | 11 |
| Go | 33 | Haskell | 6 |
| Rust | 17 | 其余 9 类(Clojure/C#/Swift/JS/Java/R/Scala/OCaml/Erlang/Elixir/Kotlin/F#等) | 1-5 |
| Lua | 16 | Additional Resources | 12 |

热门语言覆盖深（Python/C++/前端合计 300 条），小语种覆盖浅（Kotlin、F# 各 1 条）——反映的是"免费且项目式"这一过滤条件的真实供给，而非人为凑数。分类体系为两级：语言为纲、子方向为目（如 C/C++ 下的 Network programming、OpenGL；Dart 下的 Flutter），18 个 H3 子分类。

内容质量标准成文于两处：`CONTRIBUTING.md`（免费、项目式、无短链、直链、一 PR 一教程、利益披露）和 `.github/PULL_REQUEST_TEMPLATE.md`（8 项 checklist，含"我本地跑过 lint 且退出码 0"）。`scripts/lint-allow-duplicates.txt` 表明例外管理也有纪律：仅 2 个白名单条目且各带原因注释（craftinginterpreters.com 跨 C/Java 收录、LLVM Kaleidoscope 跨 C++/OCaml 收录）。

## 8. 更新机制与质量管控自动化

更新机制 = 人工 PR（增量）+ 机器巡检（存量）双轨：

- 增量：贡献者按模板提交一教程一 PR；CI（`validate-pr.yml`）自动跑 lint、diff 校验、新链接存活检查，结果落在 PR 的 Step Summary 里供维护者决策。
- 存量：每周自动全量巡检所有链接（456 个在册 URL），二振出局——连续 2 次确认 HARD_DEAD/SUSPECT 才进报告，且 SUSPECT 只列报告不自动动（`link-rot.yml` 第 56-62 行注释与 `cmd_prune` docstring）。自动修复只做两类无歧义操作，且以 PR 形式落地供人工复核，绝不直推 master。
- 拒检域名单（`scripts/linkcheck-domains.txt`）：medium/reddit/x/linkedin/udemy 等 9 个会封 CI 出口的域名标记为 BLOCKED 而非死亡，避免把反爬误报成链接腐烂。
- Issue 侧治理：死链报告模板引导用户先查自动生成的 Link rot report Issue，避免重复；空白 Issue 被禁用（`config.yml`）。

这套体系明显经过一次认真的工程化重构（脚本内成熟的安全注释、状态机设计不像临时脚本），把"清单类仓库最难防的链接腐烂"变成了可观测、可自动修复的常规流程。

## 9. 社区与维护状态

- 活跃度：最近 push 2026-10-26（gh api），持续 9 年以上维护（创建于 2017-04-12）。
- 社区规模：36,751 forks、3,461 watchers、236 个 open issues（gh api，2026-10-30）。fork 数巨大说明大量用户把它当个人学习清单 fork 自用——这是其传播飞轮的一部分。
- 贡献者结构：仓库极简，日常运营主体是维护者本人（CONTRIBUTING.md 留有联系邮箱 tuvtran97@gmail.com）+ github-actions[bot]；具体 contributor 人数未确认（tarball 无 git 历史，未额外拉取）。
- 维护哲学有明确痕迹：`stale-prs.yml` 头注释承认旧 probot stale 配置曾"埋掉正当死链报告多年"，新配置永不 stale Issue——说明维护者对过往失误有过复盘。
- 节奏：无 release 机制（内容型仓库无版本概念，未在文件中发现 release 流程配置）。

## 10. 总结评价（亮点、风险与生态位）

亮点：

1. 收录标准罕见地严格执行——"免费 + 项目式"两条硬标准让这份清单在同质化 awesome 系中辨识度极高（CONTRIBUTING.md）。
2. 内容型仓库做出了工程级治理：1179 行零依赖校验工具 + 每周巡检 + 二振状态机 + 安全自动修复 PR，链接腐烂问题被系统性压制（scripts/check_readme.py、.github/workflows/link-rot.yml）。
3. 安全意识超出多数应用项目：把 README/PR diff/远端页面标题都当不可信输入处理，注释里写明 eval/exec/shell 注入与 @mention 通知滥用等具体攻击面（check_readme.py 第 8-11、887-901 行）。
4. 消费门槛为零：单 Markdown 文件、无注册、无广告，即开即用。
5. 例外管理有白名单制度且强制写明理由（scripts/lint-allow-duplicates.txt）。

风险/局限：

1. 内容深度极不均衡：Kotlin、F#、Scala 等只有 1-2 条，部分分类（如 C# 最近条目仍是 Xamarin，已停止维护的技术栈）更新缓慢（README.md 第 157-163 行）。
2. 456 个在册链接中 77 个 BLOCKED（无法自动验证）+ 10 个 HARD_DEAD，死链清理依赖每周巡检与人工复核，小语种区无人力增量（.github/link-rot-state.json）。
3. 单维护者模式：9 年高星仓库的日常治理实质依赖一人 + bot，bus factor 低（CONTRIBUTING.md 联系方式为个人邮箱）。
4. Dart 区条目以 YouTube 视频为主（13 条中 12 条是 youtu.be/YouTube 链接，README.md 第 173-189 行），视频链接的长期存活率天然偏低——尽管 check-diff 已对新增 youtu.be 短链给出 W101 警告。

生态位与为什么能涨到 28 万星：它不是 awesome-* 那类"工具/资源大杂烩"，而是把"新手学完语法之后做什么"这个普适问题做成了单品答案；Zero-friction 消费 + 教程自带成就感（做完有成品）+ 2017 年起步的先发积累 + fork 即书签的传播机制，共同造就了 GitHub 全站顶级星数。同样重要的是后期的防劣化投入：当同类清单普遍被死链拖垮时，它是少数用工程手段守住内容质量的——这也是 9 年后仍被持续推荐的原因。

适合谁：编程初学者按语言找第一个/下一个"做得完的项目"；教师与培训者挑选课程项目；以及任何想学习"如何用最小工程量长期维护一个大型 curated list"的维护者——`scripts/check_readme.py` 本身就是一份优秀的 README-as-data 工程范本。
