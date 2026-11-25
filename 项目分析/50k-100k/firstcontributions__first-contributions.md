# firstcontributions/first-contributions 项目分析

> 星数 56484（快照 2026-10-24）｜主语言 —（纯 Markdown/文档仓）｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：全球最著名的"开源新手第一课"——教初学者用一次真实的 fork-clone-branch-PR 流程完成人生第一个 GitHub 贡献，并用自动化机器人把它们全部合并。

| 项目 | 值 | 出处 |
| --- | --- | --- |
| 名称 | firstcontributions/first-contributions | gh api |
| 星数 | 56,484（快照）/ 56,673（2026-11-03 实查） | gh api repos/firstcontributions/first-contributions |
| Fork 数 | 112,151（星数的近 2 倍，因为每个学员都要 fork 一次） | 同上 |
| 语言 | 无代码（Markdown 教程 + GitHub Actions YAML） | tarball 全量 |
| 许可证 | MIT（Copyright 2016-present Roshan Jossey） | LICENSE |
| 创建时间 | 2016-09-20 | gh api |
| 最近 push | 2026-11-02（分析前一天仍在高频合并 PR） | gh api |
| 默认分支 | main | gh api |
| Release | 无（内容型仓库，不发布版本） | gh api releases/latest → 404 |

## 2. 项目是什么，解决什么问题

"参与开源"最大的门槛不是技术，而是对完整协作流程的陌生：新人不知道 fork、分支、commit、push、PR 这些动作如何串成一条真实链路，也不敢单独面对一个真实项目。first-contributions 的解法是把整个仓库变成一个**安全的练习场**：学员只需按 README（`README.md`，246 行）的图文步骤，在 `Contributors.md` 名单里加上自己的一行名字并提交 PR——这就是一个真实但零风险的 PR。

关键设计在于"提交即被合并"：仓库根目录唯一的自动化工作流 `.github/workflows/auto-pr-merge.yml` 会自动判断该 PR 是否只改了 `Contributors.md` 且改动行数极小，符合即秒级 squash 合并，并留下一条带随机庆祝 GIF 的评论。学员在几分钟内就能收到"你的 PR 已被合并"的邮件通知，完整体验一次开源协作的闭环与正反馈。

项目的覆盖面刻意做宽：命令行怕动手的人可以用 GitHub Desktop、VS Code、GitKraken、Sourcetree、IntelliJ、Visual Studio 等图形工具教程（`docs/gui-tool-tutorials/`）；非程序员也有专门的"能做什么"文档（`docs/additional-material/Things a Non programmer can do.md`）；主 README 顶部挂着约 85 个语言旗帜链接（`README.md` 第 6-89 行），指向 `docs/translations/` 下的本地化版本。出错也不怕——`docs/additional-material/git_workflow_scenarios/` 24 篇场景文档覆盖改错 commit、解决冲突、撤销变更等所有新手高频事故，merge bot 遇到冲突时会把对应教程链接直接贴回 PR。

它由 Roshan Jossey（Roshanjossey）于 2016 年创建，如今已发展为一个组织（firstcontributions org），配套 Web 应用（firstcontributions.github.io）、进阶练习仓 code-contributions、开源徽章项目 open-source-badges 等姊妹项目（`README.md` 第 235-237 行、`.github/workflows/auto-pr-merge.yml` 第 91-92 行）。`.github/funding.json` 自述"自 2016 年以来已有超过 90,000 名用户"。

## 3. 技术栈与依赖

这是一个"无代码"仓库，技术面几乎全部由 GitHub 平台自身承载：

- **内容层**：纯 Markdown（约 380 个 .md 文件），图片全部外链至 `firstcontributions.github.io`、jsdelivr CDN 与 Wikimedia（`README.md`）。
- **自动化层**：GitHub Actions，唯一 workflow `.github/workflows/auto-pr-merge.yml` 基于 `actions/github-script@v8`（第 19、53 行），在 workflow 内直接写 JavaScript 调用 GitHub REST API（`pulls.listFiles`、`pulls.merge`、`issues.createComment` 等），无任何 npm 依赖、无需构建。
- **治理配置**：`.github/` 下的 issue/PR 模板、SECURITY.md、CODE_OF_CONDUCT.md、FUNDING.yml、funding.json（GitHub Sponsors / OpenCollective）。
- **防污染配置**：6.5KB 的 `.gitignore`，基本是 Visual Studio 官方模板的全量拷贝——这是有意为之：新手学员用各种 IDE 意外提交的构建产物，在这里被提前挡掉。

没有 package.json / requirements 等任何工程清单，没有测试框架。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `README.md`（246 行） | 主教程：fork→clone→branch→改名单→commit→push→PR，内嵌 6 个 GUI 工具教程入口表 |
| `Contributors.md`（409KB，8063 行） | 学员名单，约 6,741 条自登记条目，全仓库唯一"被贡献"的文件 |
| `docs/translations/`（109 个文件） | README 的 85+ 语种翻译（含 `README.en-pirate.md` 海盗英语这类彩蛋），`Translations.md` 为索引 |
| `docs/gui-tool-tutorials/`（11 个文件） | GitHub Desktop/VS Code/GitKraken/Sourcetree/IntelliJ/VS2017/Sublime Merge 图形工具教程 |
| `docs/cli-tool-tutorials/`（18 个文件） | git CLI 与 GitHub CLI 教程，含印地语、孟加拉语、泰米尔语等多语种版本 |
| `docs/additional-material/` | Git 工作流场景手册（24 篇）、非程序员指南（3 语种）、附加材料翻译（15+ 语种子目录） |
| `docs/how-to-contribute-to-open-source-projects.md` | 开源贡献综合指南（动机、找项目、礼仪等长文） |
| `.github/workflows/auto-pr-merge.yml`（261 行） | 全仓库唯一的"代码"：自动合并机器人 |
| `.github/` 其余 | 治理模板（ISSUE_TEMPLATE 指派给 Roshanjossey、PR 模板含"玩得开心"清单、SECURITY.md） |
| `LICENSE` | MIT |

组织方式评述：单层扁平结构，根目录只有教程、名单和许可证三件套，全部内容下沉到 docs/。没有代码架构可言，真正的"工程"集中在两个文件：名单（数据）和 auto-pr-merge.yml（逻辑）。

## 5. 核心架构与关键实现

### 5.1 自动合并机器人（贡献工厂的心脏）

`.github/workflows/auto-pr-merge.yml` 是整个机制的核心，261 行 YAML 内嵌三段 JavaScript：

**触发与准入判定**（第 1-46 行）：只监听 `pull_request_target` 的 opened/synchronize 事件且 `paths` 限定 `Contributors.md`；第一步脚本拉取 PR 文件列表，要求**恰好只改这一个文件**，同时要求 `additions < 5 && deletions < 4`（第 31-32 行），把三个布尔信号输出给后续步骤。

**自动合并与庆祝评论**（第 50-175 行）：满足条件即 `merge_method: "squash"` 合并（第 63-68 行），然后从 12 张 GIF 数组随机抽一张（第 73-88 行），拼出包含三段内容的评论——祝贺、**请求点星**（"please star this repo"，第 112 行）、以及指向 webapp/code-contributions 和 Facebook/Reddit/LinkedIn/dev.to/HackerNews/Bluesky 的**预填文案分享链接**（第 91-99、114 行）。合并失败时按错误码分流：405（冲突）回帖指引到仓库自带的冲突解决教程 `docs/additional-material/git_workflow_scenarios/resolving-merge-conflicts.md`（第 154 行），把"教学事故"变成"教学素材"。

**人工复核分流**（第 177-260 行）：改了其他文件 → 打 `waiting for review` 标签转人工；只改名单但行数超标 → 以 `REQUEST_CHANGES` 事件创建 review 并附 diff 预览（第 252 行），提示"教程要求只加一行"；bot 还会复用已有评论而非刷屏（第 244-257 行）。

### 5.2 为什么这个设计能撑住每天上百个 PR

三点组合拳：其一，`pull_request_target` + 服务端判定而非信任 PR 内容，改了任何其他文件都不可能自动合并，攻击面极小；其二，行数阈值（+5/-4）让"顺手重排名单"这类破坏性改动天然被拦截；其三，所有被拒绝的情形都会收到指向具体教程的礼貌回复，守门动作本身也是教学内容。

### 5.3 病毒式增长回路

auto-pr-merge.yml 第 110-119 行构造的合并评论是增长引擎：点星请求直接转化星级，六个社交平台的预填分享链接把"我完成了第一次开源贡献"变成可一键发布的社交内容——这解释了星数（56k）与 fork 数（112k）的双高，以及该仓库长年占据 GitHub trending 新人话题的现象。

## 6. 构建与运行

本仓库无构建、无测试、无运行时。使用方式即"参与方式"（`README.md`）：

```bash
git clone git@github.com:<你的用户名>/first-contributions.git
git switch -c add-<你的名字>
# 编辑 Contributors.md，在中间任意位置加一行 "- [你的名字](你的主页)"
git add Contributors.md
git commit -m "Add <你的名字> to Contributors list"
git push -u origin add-<你的名字>
# 然后在 GitHub 上发起 PR，等待机器人自动合并
```

相关生态站点：https://firstcontributions.github.io （Web 应用、项目推荐列表、社交分享页，见 README.md 第 233-237 行）。

## 7. 内容规模与翻译治理（特例章节）

- **名单规模**：`Contributors.md` 409,127 字节 / 8,063 行，其中 `- [` 开头的条目约 6,741 条；抽样可见条目含个人签名、宣传语甚至误贴的截图（`Contributors.md` 第 1-20 行），明显不做内容审核，只靠行数阈值拦截格式性破坏。GitHub 官方 API 统计的 contributors 为 459 人（gh api，per_page=1 翻页最后页码 459）——两者差距说明 squash 合并把大量提交归属归并了，自登记名单是更真实的"参与人数"。
- **翻译治理**：主 README 的 85 个旗帜链接 → `docs/translations/` 109 个文件；`Translations.md` 作为印度各语种（Gujarati/Hindi/Marathi/Malayalam/Kannada/Telugu/Odia/Chhattisgarhi 等）的二级索引。翻译完全靠社区 PR 驱动，无校对流程标记、无翻译进度看板（未发现 translation CI 或状态清单）；教程类文档还有第三层翻译（gui/cli 教程内的 `.zh-cn.md`、`.hi.md` 等后缀文件），三层体系之间靠手工约定维护。
- **更新机制**：内容更新走 issue → PR → 人工 review；`.github/CONTRIBUTING.md` 明确两条铁律——"不要改 Readme.md"（保持主教程稳定）与"先读 issue #35892 的设计决策"；issue 模板默认指派给 Roshanjossey。2025 年后新增了对 LLM/AI agent 的使用立场（CONTRIBUTING.md "On LLMs, coding agents" 一节：要求自己写评论、自己验证）。

## 8. 社区与维护状态

- **活跃度**：最近 push 2026-11-02（分析前一天），创建于 2016-09-20，十年持续运转无停摆期（gh api）；open issues 616，其中大量是学员提的"我做完了"式打卡和教程反馈，属健康噪声。
- **维护节奏**：无 release（合理——仓库没有可发布物）；核心维护由创始人 Roshanjossey 加 firstcontributions org 承担，issue 模板硬编码 assignee Roshanjossey（`.github/ISSUE_TEMPLATE/issue-template.md`）。
- **资金**：`.github/funding.json` 披露目标为每年 3 万美元维护者补偿 + 每月 30 美元托管费，但实际收入 2023 年 5 美元、2024 年 3 美元——项目规模与变现能力严重不对称，纯靠热情驱动。
- **安全治理**：SECURITY.md 提供私有漏洞报告通道 + firstcontributions@gmail.com。

## 9. 亮点与风险

**亮点**

1. 把"自动合并"做成了产品：秒级正反馈 + 庆祝 GIF + 点星引导 + 社交分享预填链接，一条 261 行的 workflow 完成教学、留存、增长三件事（`.github/workflows/auto-pr-merge.yml`）。
2. 守门即教学：行数/文件数双重准入 + 分场景礼貌拒绝文案，被拒的 PR 也能学到东西，冲突回复直接链到自家教程（同文件第 154 行）。
3. 多路径入门设计：CLI/GUI/非程序员/85+ 语种全覆盖，把"不会命令行"和"不写代码"的人群都纳入。
4. 降维护成本的手段务实：paths 过滤 + 行数阈值 + `.gitignore` 防污染 + "不许改 README" 的红线，让 11 万 fork 的流量洪峰只需极少人工。
5. 治理文档现代化：LLM 使用政策、设计决策 issue、私有安全通道一应俱全。

**风险/局限**

1. 名单文件已膨胀到 409KB/6,741 条，无清理策略，含大量非规范内容（签名、截图外链），git 历史也因十年日更膨胀到约 55MB（gh api size=55878KB），拖慢 clone。
2. 单点依赖创始人，资金几乎为零（funding.json 收入记录 2023 年 5 美元），长期可持续性存疑。
3. 翻译体系三层并行、无同步机制，主 README 每次改动都会让 85 个译本腐化，实际未翻译比例未知（未确认）。
4. `pull_request_target` 工作流虽有 paths 与内容双重限制，但依赖行数阈值这类启发式规则，防不住低频蓄意的内容注入（如把名字行写成宣传语）。

## 10. 总结评价

first-contributions 是"用软件工程方法运营社区教育"的教科书案例：仓库本身没有一行真正的应用代码，但一条精心设计的 GitHub Actions 工作流把 fork→PR→merge 的完整协作流程变成分钟级的正反馈体验，配以 85+ 语种翻译、GUI 工具教程和场景化故障手册，构成了开源世界最大的新手漏斗——十年、11 万 fork、6.7 千人上榜、9 万用户。它适合三类人：想完成第一次开源贡献的新人（直接照 README 做）、想研究社区运营与增长机制的维护者（精读 auto-pr-merge.yml 和 CONTRIBUTING.md）、以及想借鉴"低维护成本高吞吐 PR 工厂"模式的项目方。读源码的价值不在代码量，而在那个 261 行 workflow 里浓缩的自动化治理思想。
