# denysdovhan/wtfjs 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

**一句话定位**：一份"有趣又刁钻的 JavaScript 示例"合集（What the f\*ck JavaScript?），以每条示例 + 结果 + 规范级解释的格式收录 JS 的离奇行为，既是 GitHub 上可读的手册，也是可 `npm i -g wtfjs` 在终端翻阅的 Node 手册。

| 基本事实 | 值 | 出处 |
| --- | --- | --- |
| 名称 | denysdovhan/wtfjs | `package.json`（name: wtfjs） |
| 星数 | 39500（快照 2026-10-24）；gh api 2026-11-14 实测 37693 | 任务参数 / gh api |
| 主语言 | JavaScript（实际主体为 Markdown 内容） | gh api / `package.json` |
| 许可证 | WTFPL 2.0（"你随便想干嘛就干嘛"） | `LICENSE`、`package.json` |
| npm 包 | wtfjs v1.22.8（CLI 手册） | `package.json`（version/bin） |
| 创建时间 | 2017-07-14 | gh api |
| 最近推送 | 2026-10-05（仍在维护） | gh api pushed_at / tarball 内文件 mtime |
| Forks / Open issues | 2673 / 51 | gh api（2026-11-14） |

## 2. 项目是什么，解决什么问题

JavaScript 以"充满意外行为"著称：`[] == ![]` 为 `true`、`NaN === NaN` 为 `false`、`0.1 + 0.2 !== 0.3`、`Math.max() < Math.min()`。wtfjs 把这类"让人喊 WTF"的行为收集成一条条示例，每条给出**可直接运行验证的代码**、**注释形式的实际输出**，以及最关键的——**解释它为什么这样**（多数引用 ECMA-262 规范原文条款，如 `NaN === NaN` 条目直接引用 "7.2.14 Strict Equality Comparison" 的判定步骤，出处 `README.md` 的 NaN 小节）。

项目的自我定位写在 `README.md` 的 Motivation 区（"Just for fun"，引 Linus Torvalds 书名）：对初学者，它是深入 JavaScript、激发阅读规范兴趣的入门材料；对专业开发者，它是语言边界与坑位的参考清单。

渊源方面，`README.md` 开头明确说明：原始创意来自 Brian Leroux 在 **dotJS 2012 的演讲 "WTFJS"**（附 YouTube 链接与封面图）；同属 2012 年同类题材、Gary Bernhardt 的著名闪电演讲 **"Wat"（CodeMash 2012）** 被列在文末 "Other resources" 区（`README.md` 第 2217 行），同区还有 wtfjs.com 网站和 Kyle Simpson 的 "What the... JavaScript?" 演讲——即本项目是 2012 年那波"JS 怪异行为娱乐化讲解"热潮的文字沉淀版。

分发形态是"双通道"：GitHub README 直接阅读 + npm 发布的 CLI 手册（`npm i -g wtfjs` 后运行 `wtfjs` 命令在终端分页阅读，README 称 "Node Packaged Manuscript"），支持 `--lang` 切换翻译版本。

## 3. 技术栈与依赖

- **内容主体**：纯 Markdown。共 9 个 README 文件（主 README + 8 份翻译），合计 18549 行（`wc -l README*.md` 实测）。
- **CLI 实现**（`wtfjs.js`，唯一代码文件，约 100 行，Node ESM）：
  - `meow` — CLI 参数解析（`--lang/-l` 选择翻译）
  - `msee` — Markdown 转终端 ANSI 渲染
  - `default-pager` — 交给 `$PAGER`（默认 less）分页
  - `through2` — 文件流处理；`chalk` + `boxen` — 版本更新提示框；`update-notifier` — 升级检测
- **工程工具链**（`package.json` devDependencies）：`prettier`（格式化，`npm test` 就是 `prettier --check .`）、`doctoc`（自动生成 README 目录）、`husky` + `lint-staged`（pre-commit 时对 README 跑 doctoc + prettier）、`semantic-release`（自动发版到 npm）、dependabot（月度检查 npm 与 GitHub Actions 依赖，`.github/dependabot.yml`）。
- **无运行时框架、无构建产物**：内容即源码，源码即发布物（semantic-release 把 README*.md 一并提交回仓库，`package.json` release 配置）。

## 4. 目录结构与代码组织

| 顶层条目 | 职责 |
| --- | --- |
| `README.md` | 英文主手册：2251 行、66 条示例，全部内容所在 |
| `README-zh-cn.md` 等 8 个文件 | 中文、印地语、法语、巴西葡语、波兰语、意大利语、韩语、僧伽罗语翻译 |
| `wtfjs.js` | CLI 入口（bin: wtfjs），终端阅读器实现 |
| `package.json` / `package-lock.json` | npm 包元数据、脚本、依赖 |
| `.github/workflows/` | `validate.yml`（push/PR 跑 prettier 检查）、`release.yml`（master 推送生成 TOC → 格式化 → semantic-release 发 npm） |
| `.github/ISSUE_TEMPLATE/new-example.md`、`PULL_REQUEST_TEMPLATE/*` | 新示例 issue 与翻译 PR 模板 |
| `.husky/pre-commit` | lint-staged 钩子 |
| `CONTRIBUTING.md` | 贡献规则（新示例必须带解释；翻译走 PR 且需母语者校对） |

组织方式评述：**极端扁平的单文件内容仓库**。没有 src/lib/docs 目录之分，内容（9 个 README）、代码（1 个 CLI 文件）、配置全在顶层。条目之间零交叉引用、无模块化——这是内容型仓库的合理选择，代价是单文件达 72KB（`README.md` 72533 字节），阅读靠 doctoc 生成的目录导航。

## 5. 核心架构与关键实现

**（1）内容架构：统一的条目格式与记号约定。** `README.md` 的 Notation 区约定三种注释记号：`// ->` 表示表达式求值结果，`// >` 表示 `console.log` 输出，`//` 为普通说明注释。每条示例的结构固定为：`## 标题`（标题即现象陈述）→ js 代码块 → `### 💡 Explanation:` → 解释文字 + 规范/论文/StackOverflow 引用链接。以 `NaN is not a NaN` 条目为例（`README.md` 约 253 行起），解释部分逐条引用 ECMA-262 7.2.14 的判定流程并附 IEEE 754 关于 NaN 无序性的引文——"规范级说理"是本项目区别于同类搞笑清单的核心设计。

**（2）CLI 手册实现**（`wtfjs.js`）。核心链路约 25 行：把 `--lang` 参数规范化为 `README-<lang>.md` 文件名 → `fs.createReadStream` 读文件 → `through2` 流中先用 `msee.parse` 把 Markdown 渲染成终端文本（顺带插入 update-notifier 的 boxen 升级提示框）→ `pipe(pager())` 交给系统分页器：

```js
fs.createReadStream(translation)
  .pipe(
    obj(function (chunk, enc, cb) {
      // ...update-notifier 提示框...
      this.push(msee.parse(chunk.toString(), mseeOpts));
      cb();
    })
  )
  .pipe(pager());
```

实现思路是"把仓库里的 README 当作数据文件直接流式渲染"，因此 npm 包的内容更新与仓库 README 天然同步，翻译切换只是换文件路径（`wtfjs.js` 第 64-73 行）。

**（3）自动化发布流水线**（`.github/workflows/release.yml`）：master 分支每次推送 → `npm run toc`（doctoc 重新生成目录）→ `npm run format`（prettier）→ `semantic-release` 按 commit 语义自动发 npm 并回写版本文件。配套 `validate.yml` 在所有 push/PR 上跑 `prettier --check .` 作为门禁。

## 6. 构建与运行

来自 `package.json` scripts 与 README，均为实际可用命令：

```bash
# 终端阅读（项目推荐用法）
npm install -g wtfjs
wtfjs                    # 英文手册
wtfjs --lang zh-cn       # 中文翻译

# 本地开发
npm install
npm run toc              # doctoc 重新生成全部 README 目录
npm run format           # prettier 格式化
npm test                 # prettier --check .（唯一"测试"）
npm run release          # semantic-release 自动发版
```

无需构建步骤：README 即最终产物，CLI 直接流式读取 README。

## 7. 条目组织与规模统计（内容型仓库特例）

**条目规模（基于 tarball 内真实文件统计）**：

- 主 `README.md`：2251 行 / 72533 字节。Examples 区（第 168-2214 行）共 **66 条示例**（该区间 `## ` 标题数）；`### 💡 Explanation:` 块 70 个（个别条目含多块解释）；**`🎬` Demo 标记为 0 处**——当前版本示例仅用"代码 + 💡 解释"两级结构，无独立演示区（对全部 9 个 README 统计均为 0）。
- 条目**无分类体系**：66 条平铺排列（从 `[] is equal ![]` 到 `Non-strict comparison of a number to true`），仅靠 doctoc 自动生成的两级目录导航，未按主题（类型转换/原型/作用域等）分组。
- 文末 "Other resources" 区给出 wtfjs.com、Gary Bernhardt "Wat" 演讲、Kyle Simpson 演讲等 4 个延伸链接。

**翻译规模（各语言 README 抽样统计）**：

| 文件 | 行数 | 示例条目（## 数） | 💡 块 |
| --- | --- | --- | --- |
| README.md（主） | 2251 | 66 | 70 |
| README-zh-cn.md | 2267 | 65 | 71 |
| README-kr.md | 1782 | 55 | 58 |
| README-it-it.md | 1786 | 55 | 58 |
| README-fr-fr.md | 1718 | 54 | 57 |
| README-hi.md | 1714 | 54 | 53 |
| README-pl-pl.md | 1743 | 54 | 57 |
| README-pt-br.md | 1731 | 54 | 57 |
| README-si.md | 3557 | 112（仅 56 唯一） | 114 |

8 份翻译中 7 份（zh-cn/hi/fr/pt-br/pl/it-it/kr）覆盖主手册约 82%（54-65/66 条）。两处异常值得注意：**README-si.md（僧伽罗语）未列入主 README 的 Translations 链接清单**（清单只列 7 种语言），且其内容整体重复了一遍（112 个 `## ` 标题仅 56 个唯一，`sort | uniq -d` 可复现）；zh-cn 翻译质量抽样良好（`baNaNa` 条目译文准确，README-zh-cn.md 第 275 行起）。

**示例真实性抽检（Node v24.13.0 实测，与 README 代码逐字对照）**：抽 7 条全部复现，与文档记载一致——

- `[] == ![]` → `true`（README 条目 "`[]` is equal `![]`"）
- `'b' + 'a' + +'a' + 'a'` → `'baNaNa'`（条目 "baNaNa"）
- `NaN === NaN` → `false`；`NaN !== NaN` → `true`；`typeof NaN` → `'number'`
- `0.1 + 0.2 === 0.3` → `false`（条目 "Precision of 0.1 + 0.2"）
- `Math.max() < Math.min()` → `true`（条目 "Math.max() less than Math.min()"）
- `true + true + true === 3` → `true`（条目 "Math with true and false"）

客观评价：抽检样本零翻车，代码块可运行、结果标注准确；解释引用的规范条款（如 7.2.14、7.2.15 Abstract Equality Comparison、22.1.3.21 String.prototype.split）链接均指向 tc39 现行规范页面。

## 8. 更新机制与维护状态

- **更新机制**：两类贡献走不同通道（`CONTRIBUTING.md`）。新示例必须**附带解释**（最好引规范/博客/论坛链接），"无解释的 issue 会被关闭"——该规则同时写进 `.github/ISSUE_TEMPLATE/new-example.md` 模板；翻译一律走 PR（`.github/PULL_REQUEST_TEMPLATE/translation.md`），且要求"找到一位母语者校对"。TOC 与格式由 CI 和 pre-commit 自动维护，贡献者只需改内容。
- **维护状态**：仓库最近推送 2026-10-05（分析日 2026-11-14 前约 5 周，gh api pushed_at），内容仍活跃；但 **npm 发版停留在 v1.22.8（2022-09-30 发布，gh api releases/latest）**——即 CLI 包 4 年未发新版，期间的新示例只进了仓库 README。贡献者约 90 人（gh api `contributors?per_page=1&anon=true` 末页页码 90，含匿名贡献）。
- **CI/质量门禁**：`validate.yml`（prettier 格式检查）+ dependabot 月度升级依赖；**没有示例代码的自动化验证**——示例正确性完全靠贡献者人工与评审保证（这正是"必须附解释"规则的制度性补偿）。

## 9. 亮点与风险

**亮点**：

1. **"示例 + 规范引文"的说理式内容设计**：不只罗列怪象，每条给出 ECMA-262 条款级解释（如 NaN 条目引用 7.2.14，`README.md`），把娱乐性清单做成了规范导读。
2. **真实可复现**：抽检 7 条经典示例在 Node v24.13.0 下全部复现，代码与结果标注可信。
3. **双形态分发**：GitHub 手册 + `wtfjs` npm CLI（msee 终端渲染 + pager 分页 + `--lang` 切换翻译），在"README 仓库"类项目中分发形态罕见地完整。
4. **多语言生态**：8 份翻译、总计 18549 行 Markdown，中/韩/印地/僧伽罗等覆盖面远超同类项目。
5. **全自动工程链**：doctoc + prettier + semantic-release + dependabot + husky，维护成本压到极低。

**风险/局限**：

1. **翻译滞后与不一致**：各语言 54-65 条对主手册 66 条；si 翻译未入清单且内容整体重复（README-si.md 统计见第 7 章）。
2. **无示例自动化验证**：`npm test` 仅是 prettier 格式检查（package.json），示例结果正确性无 CI 兜底，规范版本演进时可能失真。
3. **CLI 发版停滞**：npm 上最新 v1.22.8 为 2022-09-30，新读者装到的包不含近 4 年新增示例。
4. **内容偏娱乐向**：约半数条目是 `==` 强转类历史行为，在 ESLint/TypeScript 主流化后实战价值有限，更像语言文化标本（README 自述定位"just for fun"）。

## 10. 总结评价

wtfjs 是"内容型仓库"的标杆样本：单一创意（JS 离奇行为 + 规范级解释）持续运营 9 年（2017-2026，gh api created_at/pushed_at），靠"无解释不收录"的内容纪律、极低的工具链维护成本和多语言社区翻译，把一份 dotJS 2012 演讲（Brian Leroux "WTFJS"，与 Gary Bernhardt "Wat" 同潮）沉淀成 66 条、9 语言、18549 行的规范式手册。它不能让你写出更好的业务代码，但能系统性地告诉你 JavaScript 的边界在哪里。适合三类人：JS 初学者当规范导读的引子；准备面试/分享会的人当"怪行为"素材库；想运营内容型开源仓库的人当组织与自动化范本来抄。读"源码"的意义不在 `wtfjs.js` 那 100 行 CLI，而在 README 条目的结构与贡献流程的设计。
