# poteto/hiring-without-whiteboards 项目分析

> 星数 52861（快照 2026-10-24）｜主语言 JavaScript｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：一份不搞"白板式"算法题面试（CS 八股/LeetCode live coding）的公司清单，帮助求职者避开坏面试流程、帮助公司宣传好的面试方式。

| 项目 | 值 |
| --- | --- |
| 名称 | poteto/hiring-without-whiteboards |
| 星数 | 52,861（快照）；gh api 实时 53,118（`gh api repos/poteto/hiring-without-whiteboards`） |
| 主语言 | JavaScript（仅工程脚手架，实际主体是 Markdown 内容） |
| 许可证 | MIT（`LICENSE.md`，Copyright 2017 Lauren Elizabeth Tan） |
| 创建时间 | 2017-03-15（gh api `created_at`） |
| 最近推送 | 2026-09-24（gh api `pushed_at`，与本仓 tarball 内文件 mtime 2026-09-24 一致） |
| Fork / Watch | 3,987 / 936（gh api） |
| 内容规模 | README.md 828 行、约 145KB，公司条目 761 条（实测统计） |

## 2. 项目是什么，解决什么问题

这是前端圈知名开发者 Lauren Tan（poteto，Ember/React 生态，`package.json` 的 author 字段：`Lauren Tan <arr@sugarpirate.com>`）在 2017 年发起的社区清单。它不是代码项目，而是一份由社区众包维护的"面试体验白名单"：列出那些面试流程贴近日常工作的公司——例如结对编程解决真实问题、有时间盒的 take-home 项目、围绕候选人过往经验的技术讨论等（`README.md` 开头段落）。

"Whiteboard" 在这里是一种隐喻，代指与岗位无关的 CS 八股、脑筋急转弯、算法谜题等坏面试实践，而非白板本身。README 的 tl;dr 写得很清楚：讨论真实世界问题（有无白板均可）是好的；做与工作无关的 CS 问答、解谜（有无白板均可）是坏的（`README.md` 头部 "tl;dr" 小节）。

收录有一条硬性门槛（`CRITERIA.md`）：公司面试流程的任何环节都不得 (1) 提问与应聘岗位无关的 CS 琐题/谜题/脑筋急转弯；(2) 使用 HackerRank、LeetCode 之类 live-coding 平台。唯一例外是岗位本身要求 CS 知识（例如招人写包管理器就需要懂 DAG）。这条二分法让清单有了可执行的判定标准，而不是靠主观体验。

反面是 README 中的 "Duds" 机制：如果候选人的真实面试体验与条目描述不符，可以发 PR 将该公司移除（`README.md` "Duds" 小节、`CONTRIBUTING.md` "Removals" 一节）。这种"由候选人验证、可被撤销"的设计，是清单长期保持可信度的关键。

项目还附带了给公司的反向建议（`RECOMMENDATIONS.md`）：把筛选题改造成贴近真实工作的题目；take-home 应付费、限时（如 4 小时内）、只用标准库、与实际工作相关；给候选人提供替代选项（无法做 take-home 的可以带电脑现场结对）。这让项目从"求职者工具"延伸成了"招聘改进指南"。

## 3. 技术栈与依赖

这是一个内容型仓库，代码只有 1 行占位（`index.js`：`module.exports = {};`），技术栈全部服务于"用 CI 保证清单质量"：

- **运行时**：Node.js 20（CI 配置 `.github/workflows/node.js.yml`），`.nvmrc` 为 `lts/*`。
- **包管理**：Yarn（`yarn.lock` 84KB、CI 使用 `yarn install --frozen-lockfile`）。
- **Lint 工具链**（`package.json` dependencies / remarkConfig）：
  - `remark-cli` 12 + `remark-lint` 10：Markdown 格式检查，`yarn test` 实际就是 `remark-cli/cli.js README.md -f -q`；
  - `remark-lint-alphabetize-lists`：强制清单按字母序排列——这是 PR 高频违反的规则，直接交给 CI 把关；
  - `remark-lint-hiring-without-whiteboards-links` 0.3.2：项目专属插件，用于校验条目链接格式；
  - 其余：`remark-lint-no-tabs`、`remark-lint-no-trailing-spaces`、`remark-lint-no-url-trailing-slash`、`remark-lint-unordered-list-marker-style`、`remark-preset-lint-recommended`。
- **链接检查**：lychee-action v2 在 CI 中对 README 全量链接做存活检查，要求 HTTPS、失败即挂（`.github/workflows/node.js.yml` 的 `links` job）。

依赖极少且没有运行时依赖——8 个直接依赖全部是 lint 插件，`yarn.lock` 中绝大多数为传递依赖。这是一个典型的"内容仓库当软件项目养"的配置。

## 4. 目录结构与代码组织

| 路径 | 职责 |
| --- | --- |
| `README.md` | 唯一的核心内容：项目说明 + 761 条公司清单（828 行，145KB） |
| `CRITERIA.md` | 收录标准：两条硬性禁令 + 一个例外（510 字节） |
| `CONTRIBUTING.md` | 贡献规范：条目格式、字母序、新增/移除流程（1.6KB） |
| `RECOMMENDATIONS.md` | 给面试官的建议：如何设计更好的面试（1KB） |
| `CODE_OF_CONDUCT.md` | 行为准则（3.2KB） |
| `LICENSE.md` | MIT 许可证 |
| `index.js` | 1 行占位 `module.exports = {};`，仅为满足 npm 包语义（package.json 的 main 字段） |
| `package.json` / `yarn.lock` | remark lint 工具链与锁文件 |
| `.github/workflows/node.js.yml` | CI：remark lint + lychee 链接检查 |
| `.github/stale.yml` | probot-stale 配置：PR 60 天无活动标记 stale，7 天后关闭，仅作用于 pulls |
| `.github/PULL_REQUEST_TEMPLATE.md` | PR 模板：4 个勾选框（读过规范/同意 CoC/遵守格式/补充上下文） |
| `.editorconfig` / `.nvmrc` / `.gitignore` | 编辑器与开发环境约定 |

组织方式评述：单文件内容仓库的极简形态——所有"业务数据"集中在一个 README.md，其余文件全部围绕"如何让这个 README 可信、有序、不过期"服务。没有构建系统、没有多包结构，内容与治理规则（CRITERIA/CONTRIBUTING）分离清晰，是 awesome 类清单中治理结构最完整的一个。

## 5. 核心架构与关键实现

内容型仓库没有传统意义的架构，本项目的"核心实现"是三套机制：

**(1) 条目数据格式**（`CONTRIBUTING.md` "Format" 一节）。每条目一行，三段式竖线分隔，这是全项目唯一的数据 schema：

```markdown
- [Company or team name within company](https://jobspage) | locations | description of interview process
```

地点必须是城市名，用 `/`、`;`、`&` 分隔（如 `San Francisco, CA; Tokyo, Japan; Moscow, Russia`）。这个格式让 761 条记录保持机器可解析的一致性，贡献者违反格式会被 CI 拒绝。

**(2) 字母序分区索引**（`README.md`）。正文分 10 个二级标题区段：`0 - 9`、`A - C`、`D - F`、`G - I`、`J - L`、`M - O`、`P - R`、`S - U`、`V - X`、`Y - #`（含数字开头公司兜底），外加头部"Discussion and other reads"外部文章区与尾部"Also see"姊妹站链接。761 个条目严格按公司名字母序插入（`CONTRIBUTING.md` 要求"add the company in alphabetical order"，PR 模板还特别强调"Do not sort other listings"，避免贡献者顺手重排整个文件造成 diff 污染）。

**(3) 自动化质量门禁**（`.github/workflows/node.js.yml`）。CI 包含两个 job：

- `remark`：跑 `yarn test`（即 remark-cli 对 README.md 的 lint，见 `package.json` scripts），其中 `remark-lint-alphabetize-lists` 自动校验字母序、专属插件校验链接格式；
- `links`：用 lychee 全量检查 README 中约 1600 个外链的存活与 HTTPS 合规，`fail: true` 保证死链即红。

配套的 `.github/stale.yml`（probot-stale，仅对 pulls 生效）自动关闭 60 天无活动的 PR，防止贡献积压腐化。这套"格式 lint + 链接检查 + stale 机器人 + PR 模板勾选框"的组合拳，是它能在千余贡献者规模下长期维持单文件秩序的根本原因。

## 6. 构建与运行

没有构建产物，本地验证只有一条命令（`package.json`）：

```bash
yarn install --frozen-lockfile   # 安装 remark lint 工具链
yarn test                        # 对 README.md 执行 remark lint（-f 自动修复 -q 静默）
```

CI 额外执行（`.github/workflows/node.js.yml`，本地通常不跑）：

```bash
# lychee 链接存活检查（--require-https，失败退出非零）
lychee --verbose --no-progress --require-https --cache 'README.md'
```

Node 版本要求：`.nvmrc` 为 `lts/*`；CI 固定 Node.js 20.x。

## 7. 内容规模与更新机制

**条目规模**（实测 `README.md`）：

- "## 0 - 9" 至 "## Y - #" 区间内顶级条目 761 条（`awk '/^## 0 - 9/,/^## Also see/' | grep -c '^- \['`）；其中 9 行含附加外链。
- 按关键词统计：提到 take-home 的 418 条、提到 Remote 的 289 条、提到 pair programming 的 198 条——可见"付费/限时 take-home + 结对编程"是清单公司的主流面试形态，而非另一种应试。

**更新机制**：纯 PR 众包。新增/更新须符合格式、按字母序插入、带官网链接；移除须说明该公司为何违反 `CRITERIA.md`，且会给原添加者一次申诉机会（`CONTRIBUTING.md` "Removals"：若岗位确实要求 CS 知识可重新加回）。PR 模板强制勾选确认读过规范。CI 自动挡住格式错误与死链，维护者主要裁决的是"该公司是否真的符合收录标准"。

**生态位与姊妹项目**：尾部 "Also see" 链接到姊妹站 They Whiteboarded Me（they.whiteboarded.me，README.md 第 826 行）——一个基于本清单数据的反面投票站（"被白板过"的公司）；头部 Discussion 区链接两次 HN 大讨论（2017、2020，2020 的链接走 web.archive.org 存档，README.md 第 22-23 行）。它被广泛视为求职面试准备领域的"反向 LeetCode 清单"，是 awesome 类清单中治理范本式的存在。

**为什么能涨到 5 万+ 星**：(1) 命中痛点——算法面试军备竞赛让大量工程师反感，这个清单把"去哪找工作不刷题"变成一个可查的答案；(2) 规则极简——两条硬性标准可一句话复述，容易传播；(3) 作者是知名开发者，2017 年 HN 首发（README 内链接）带来第一波流量；(4) 长期维护——从 2017 年到 2026 年仍有稳定条目进出的活跃 PR 流，星数持续累积。

## 8. 社区与维护状态

- **活跃度**：gh api 显示 `pushed_at` 2026-09-24（约本次分析前 6 周），tarball 内全部文件 mtime 同为 2026-09-24，说明仓库仍在活跃更新中。快照 52,861 星，实时 53,118 星，仍在自然增长。
- **贡献者结构**：contributors API（含匿名）分页末页为 1,180，即约 1,180 名贡献者——条目主要由各公司员工/前员工自行提交，长尾结构典型。发起人为 Lauren Tan（`package.json` author、`LICENSE.md` 版权人），2017-03 创建。
- **维护负担控制**：open issues 仅 13 个（gh api），probot-stale 自动清理不活跃 PR（`.github/stale.yml`），维护成本被工具链压得很低。
- **Release 节奏**：无 release/CHANGELOG（tarball 内不存在相关文件），对内容型仓库无意义，未发现版本发布机制。

## 9. 亮点与风险

**亮点**：

1. 判定标准极简且可执行：两条禁令（无关 CS 八股、HackerRank/LeetCode live-coding）+ 一个例外（岗位必需 CS 知识），见 `CRITERIA.md`，任何人都能据此判断一家公司该不该在列。
2. 工程化治理教科书：格式 lint（含项目专属 remark 插件与字母序检查）+ lychee 死链检查 + stale 机器人 + PR 模板，用 CI 解决了千余贡献者污染单文件的经典难题（`.github/workflows/node.js.yml`、`.github/stale.yml`、`package.json` remarkConfig）。
3. 双向价值：既是求职者的避坑指南，也通过 `RECOMMENDATIONS.md` 给公司提供可操作的面试改进建议（付费 take-home、限时、贴近真实工作）。
4. 有纠错闭环："Duds" 移除机制允许候选人以真实经历发起除名，且保留被指控公司的申辩权（`README.md` Duds 小节、`CONTRIBUTING.md` Removals）。

**风险/局限**：

1. 信息衰减不可避免：公司招聘流程随时在变，761 条目无法逐一复核，README 本身也承认需要候选人反馈来除名——清单只能保证"曾经符合"，死链由 lychee 兜底但描述真实性无人担保。
2. 无结构化数据：全部信息挤在一个 145KB 的 README 里，无法按地点/远程/技术栈检索（仅靠 GitHub 页内搜索），未提供 JSON/API 等机器可读形态。
3. 判定标准有灰色地带：take-home 是否付费、时间盒多长、现场白板讨论真实问题算不算"白板"，不同 PR 裁决尺度可能不一致（`CRITERIA.md` 未细化，属人治范畴）。
4. 地域覆盖偏英文世界：条目以欧美、印度为主，中国公司极少（抽样 `README.md`，如 Wiredcraft 上海是少数中国地点条目），对中文求职者参考价值有限。

## 10. 总结评价

这是一个把"社区清单"当成软件项目来运营的典范仓库：核心资产是 761 条由近 1,200 名贡献者九年时间积累的公司面试体验条目，而真正让它在 5 万星规模下不腐烂的，是 CRITERIA 两条硬规则、三段式条目 schema、以及 remark 字母序 lint + lychee 死链检查 + stale 机器人组成的最小自动化治理体系（均可在 `README.md`、`CRITERIA.md`、`CONTRIBUTING.md`、`.github/`、`package.json` 中逐一验证）。它适合两类人"读源码"：一是想找工作但不想刷题的工程师——直接当检索工具用，注意核实时效；二是 awesome 清单/众包文档的维护者——它的治理设计（格式 lint、字母序强制、除名申诉流程）几乎可以直接抄走。对纯代码学习者而言，仓库里只有一行 JS 可读，价值不在代码而在流程。
