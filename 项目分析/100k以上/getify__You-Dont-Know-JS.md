# getify/You-Dont-Know-JS 项目分析

> 星数 184833（快照 2026-10-24）｜主语言 —（纯 Markdown 内容仓库）｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：Kyle Simpson（getify）的《You Don't Know JS》JavaScript 深度技术书系列官方全文仓库——GitHub 上被引用最广的免费 JS 进阶读物，系列已完结出版，仓库转为静态存档与免费阅读入口。

| 项目 | 内容 |
| --- | --- |
| 名称 | getify/You-Dont-Know-JS |
| 星数 | 184,836（gh api 实测 2026-10-30；任务快照 184,833） |
| 语言 | 无代码语言（GitHub `language: null`，全部为 Markdown 书稿） |
| 许可证 | CC BY-NC-ND 4.0（`LICENSE.txt`；GitHub 标记 NOASSERTION） |
| 创建时间 | 2013-11-16（gh api） |
| 最近 push | 2026-02-15（gh api） |
| 默认分支 | `2nd-ed`（第二版书稿；第一版在 `1st-ed` 分支，README.md 链接注明） |
| Forks / Open issues | 33,355 / 2（gh api） |
| 主题标签 | book, book-series, education, javascript, closures, prototypes, async, es6, learn-to-code, training-materials 等 13 个（gh api） |
| 仓库体量 | 55 个 .md，约 141,253 词，1,424 个代码围栏块（tarball 内实测统计） |

## 2. 项目是什么，解决什么问题

这是《You Don't Know JS》（YDKJS / 第二版名《You Don't Know JS Yet》，YDKJSY）书系列的**全文书稿仓库**，不是软件项目。作者 Kyle Simpson 在顶层 `preface.md` 中阐明了系列立场：与《JavaScript: The Good Parts》的"只学好部分"相反，主张"学习并使用 JS 的所有部分（all the parts）"，目标是让开发者真正理解每一行 JS 的运行机制，而不是"能跑就行"（preface.md "The Parts"、"The Mission" 两节）。书籍定位明确不是入门教程，默认读者已有 6-9 个月 JS 经验（preface.md）。

仓库当前主体是**第二版**（默认分支 `2nd-ed`）。按 README.md 的阅读顺序为：《Get Started》（完成，已出版）→《Scope & Closures》（完成，已出版）→ "The Unbooks" 电子书合集（含《Objects & Classes》draft stable 与《Types & Grammar》rough draft）。原计划的《Sync & Async》与《ES.next & Beyond》两册**已取消**——这两个目录下只剩目录页和标注 "TODO / Work in progress" 的 ch1 占位（`sync-async/ch1.md`、`es-next-beyond/ch1.md`）。README.md 明确宣布："This book series is now complete, and is **not open to further contributions**"。

它解决的核心问题：JS 社区长期缺乏一套免费、深入、系统讲解语言底层机制（作用域、闭包、原型、类型与强制转换）的教材。第一版（2013-2015，存于 `1st-ed` 分支）经 Kickstarter 众筹出版后成为事实上的社区标准读物；第二版（2019 年启动）沿用了"线上免费全文 + 付费出版"的模式——`preface.md` 与各分册 README 均声明线上永久免费，同时通过 GetiPub 自出版在 Leanpub/Amazon 销售。

已出版两册的章节构成（各册 README.md）：

- 《Get Started》：What Is JavaScript? → Surveying JS → Digging to the Roots of JS → The Bigger Picture，附录 A "Exploring Further" 与附录 B "Practice, Practice, Practice!"。序言作者 Brian Holt。
- 《Scope & Closures》：What's the Scope? → Illustrating Lexical Scope → The Scope Chain → Around the Global Scope → The (Not So) Secret Lifecycle of Variables → Limiting Scope Exposure → Using Closures → The Module Pattern，同带两个附录。序言作者 Sarah Drasner。

阅读节奏上作者在 `preface.md` "The Path" 一节明确建议**不要连读**：一章读两遍、逐节配代码练习，"一年读完整个系列都不算慢"——这与其说是一本教程，不如说是一套需要配合实战消化的参考体系。

## 3. 技术栈与依赖

内容型仓库，没有工程清单（仓库内无 package.json、无构建脚本、无 CI 配置，`.github/` 下只有 4 个 issue 模板）。实际"技术栈"为：

- **载体**：纯 Markdown 文件，UTF-8（`.editorconfig` 指定 charset=utf-8、md 用 4 空格缩进）；插图用 `<figure>/<figcaption>` HTML 嵌入 PNG（如 `scope-closures/ch1.md` 中的 Fig.1）。
- **示例代码**：内嵌于 Markdown 的代码块。全仓 1,424 个围栏代码块，其中 680 个标注为 `js`（对全部 55 个 .md 用 grep 统计）。
- **出版链路（仓库外）**：Leanpub 电子书、Amazon 纸书，由 GetiPub 自出版（README.md "Publishing" 节）；仓库本身就是出版排版前的母稿。
- **周边生态**：作者在 Frontend Masters 的视频课程作导流（README.md 赞助节）；`external-logos/` 存放 Frontend Masters 与 Doppler 的赞助方 SVG logo。

## 4. 目录结构与代码组织

顶层目录（tarball 实测）：

| 目录/文件 | 职责 |
| --- | --- |
| `get-started/` | 第 1 册《Get Started》：4 章 + 2 附录 + foreword + toc，约 24,256 词，已完成出版 |
| `scope-closures/` | 第 2 册《Scope & Closures》：8 章 + 2 附录，约 46,029 词，已完成出版 |
| `objects-classes/` | 《Objects & Classes》：5 章，约 32,779 词，draft stable，随 "The Unbooks" 合集出版 |
| `types-grammar/` | 《Types & Grammar》：4 章（第 5 章永远停在 TODO），约 35,340 词，rough draft |
| `sync-async/`、`es-next-beyond/` | 已取消的两册，各只剩 README/toc/foreword + ch1 的 "TODO" 占位 |
| `preface.md` | 全系列共享序言（"学习 JS 的所有部分"宣言） |
| `external-logos/` | 赞助商 logo（Frontend Masters、Doppler） |
| `unbooks-cover.png`、`fixed-it-for-you.png` | 合集封面等顶层图片资源 |
| `.github/ISSUE_TEMPLATE/` | 4 个 issue 模板：内容提问、翻译申请、技术错误报告、文字勘误 |

组织方式评述：**"一书一目录、册内扁平"** 的极简结构——每册固定为 `README.md`（标题 + 封面图 + 购买链接 + 目录）、`foreword.md`（请业界名人作序：Brian Holt、Sarah Drasner、Rick Waldron，见各册 foreword.md/README.md）、`toc.md`、`chN.md`、`images/`（全仓 19 张配图）。没有 monorepo 工具、没有多包管理，内容即文件树。各册章节数与规模（wc -w 实测）：

| 分册 | 章节 | 词数 | 状态 |
| --- | --- | --- | --- |
| get-started | 4 章 + 2 附录 | 24,256 | 完成并出版 |
| scope-closures | 8 章 + 2 附录 | 46,029 | 完成并出版 |
| objects-classes | 5 章 | 32,779 | draft stable（Unbooks 合集） |
| types-grammar | 4 章（ch5 = TODO） | 35,340 | rough draft（Unbooks 合集） |
| sync-async / es-next-beyond | 各 1 个占位章 | 各 125 | 已取消 |

值得注意的是每个 md 文件顶部都重复完整书名 + 章节标题，CONTRIBUTING.md 解释这是为出版管线服务：**"this repo is not optimized for your reading experience"**——仓库的定位是母稿管理与免费公开，不是在线阅读站。`objects-classes/thanks.md` 还保留着 371 名 Kickstarter 支持者的完整鸣谢名单，是众筹出版史的直接存档。

## 5. 核心架构与关键实现

以内容深读代替代码架构分析。

**（1）三大支柱的知识体系设计。** 第二版把 JS 语言组织为"三大支柱"：作用域与闭包（Pillar 1）、原型（Pillar 2）、类型与强制转换（Pillar 3），在 `get-started/toc.md` 的 Chapter 4 "The Bigger Picture" 中明确提出，随后每册书各占一柱。这个分册框架本身就是系列的"架构"：第 1 册建立心智模型（含 "JS 是编译型语言"的论证），后续册逐支柱深挖。

**（2）论证式写法与反直觉立场。** 以 `scope-closures/ch1.md` 为例：开篇即论证"JS 事实上在执行前有独立的解析/编译阶段"，用编译器三阶段（Tokenizing/Lexing → Parsing → AST）讲解作用域为何在编译期确定，直接反驳"JS 是解释型脚本语言"的普遍认知。这种"先破除错误心智模型、再给机制细节"的写法贯穿全书，也是系列最具辨识度的价值。

**（3）第二版新增的练习附录（apB.md）。** 两本已完成册各带 "Practice" 附录：如 `scope-closures/apB.md` 的 "Buckets of Marbles" 练习——要求读者写出含嵌套函数与块级作用域、且涂色后至少需要六种颜色的程序，用"着色作用域气泡"（呼应 ch2 的 Fig.2 配图）把抽象作用域具象化。附录开头明确要求"在真正的代码编辑器里做，别偷看答案"。第一版没有这种体系化练习，是第二版教学设计的主要升级点。`get-started/apB.md` 同样定位为练习册（章名 "Practice, Practice, Practice!"，见 get-started/README.md 目录）。

**（4）作者个人品牌贯穿细节。** 顶层 `fixed-it-for-you.png`（1.2MB）是社区梗图式的自嘲插图；README.md 赞助节直接把读者导流到作者在 Frontend Masters 的全部工作坊页面（带推广码 `?code=simpson`）；序言、致谢、赞助、众筹名单在册间交叉引用顶层 `preface.md`——整仓是一套围绕个人 IP 精心编织的出版与品牌资产，而非松散文档堆。

**（4）版权与免费发布的组合拳。** `LICENSE.txt` 为完整的 CC BY-NC-ND 4.0 文本（署名-非商业-禁止演绎），README.md 版权节标注 "© 2019-2025 Kyle Simpson"。ND（禁止演绎）条款正是模式的关键：全文免费可读（无付费墙），但改编、商用、二次分发被法律禁止，变现只能走官方出版与赞助渠道。

## 6. 构建与运行

无构建、无测试、无运行时——仓库只被"阅读"。实际可用的使用方式：

- 在线阅读：直接在 GitHub 逐个打开 `get-started/ch1.md` 等，或按各册 `toc.md` 顺序读；阅读入口为仓库根 `README.md`，推荐顺序为 Get Started → Scope & Closures → The Unbooks。
- 本地阅读：`git clone https://github.com/getify/You-Dont-Know-JS.git`（或下载 tarball），用任意 Markdown 阅读器打开。
- 购买正式版（作者推荐的最佳阅读体验，见 CONTRIBUTING.md）：Leanpub 电子书 `leanpub.com/ydkjsy-get-started` / `ydkjsy-scope-closures` / `ydkjsy-unbooks`，或 Amazon 纸书（各册 README.md 内链接）。
- 翻译（唯一开放通道，CONTRIBUTING.md）：Fork 仓库 → 在自己 Fork 内完成整册翻译（至少一整本，最好六本）→ 提 issue 申请以语言 ISO 码建分支 → PR 合入 → 被提升为该语言分支 maintainer 自行维护；翻译必须免费开放、不得变现，且第二版仅 3 册可译，第一版不接受新译本。
- 提问/纠错：通过 `.github/ISSUE_TEMPLATE/` 的 4 个模板提 issue，但模板已声明内容更正不再受理。

## 7. 内容质量管控（代替"测试与工程质量"）

- **流水线前移**：质量保障主要发生在仓库外的专业编辑/出版流程（"typo will almost certainly be caught during the editing process"，见 `PULL_REQUEST_TEMPLATE.md` 引言）。仓库内没有自动化校验，无 CI、无 lint。
- **刻意关闭众包修正**：`.github/ISSUE_TEMPLATE/content-question.md` 模板顶部即声明"不再接受内容更正"；CONTRIBUTING.md 对 typo 的态度是"我们不在乎了，书已出版"；PR 模板要求贡献者承诺"已读贡献指南"并汇总小修。这是内容型仓库少见的"冻结式维护"策略——用降低社区通道优先级换取作者精力解放。
- **issue 模板即路由**：4 个模板把剩余社区流量分流为内容提问 / 技术错误 / 勘误 / 翻译四类，其中仅翻译类仍有实际流程。
- 整体观察：作为母稿仓库，文件一致性（统一命名 chN.md/apA.md、统一文件头格式）靠作者自律而非工具，实测 55 个 md 命名完全规整。

## 8. 社区与维护状态

- **生命周期**：2013-11 建仓（gh api），第一版 6 册 2013-2015 年间以 Kickstarter 众筹 + 开放章节方式陆续发布，星数主要在这一时期随"免费高质量 JS 书"口碑累积；第二版 2019 年启动（YDKJSY），`objects-classes/thanks.md` 列出 371 名 Kickstarter 支持者。当前系列宣告完结，仅剩两册取消（README.md Titles 节）。
- **活跃度**：最近 push 2026-02-15（gh api），open issues 仅 2——结合 CONTRIBUTING 的冻结声明，实质是**低频维护的存档状态**（改动大概率为赞助 logo、链接修补类）。tarball 内文件 mtime 一致（2026-01-18），无逐文件演化线索，未确认具体改动内容。
- **贡献者结构**：contributors API 分页约 211 人（per_page=1 测得 page=212），但 CONTRIBUTING.md 已关闭贡献，历史贡献者以翻译分支维护者为主。作者 Kyle Simpson 是唯一核心内容作者。
- **商业生态**：前两册由 Frontend Masters 独家赞助（README.md）；销售渠道 Leanpub/Amazon；作者另有 GitHub Sponsors / Patreon / PayPal 赞助入口（README.md Publishing 节）。

## 9. 亮点与风险

**亮点**

1. 免费合法的深度 JS 教材标杆：20 万词级别、约 1,400 个代码示例（实测统计）完整开放，深度覆盖作用域/闭包/原型/类型 coercions 等面试与实战核心话题。
2. "all the parts" 的独立立场：反对 JS 社区流行的"只用 good parts / 拥抱新特性"风向，论证角度独特且经十几年读者检验（`preface.md`；get-started/foreword.md 中 Brian Holt 自述"读了才发现自己不懂 JS"）。
3. 可持续的免费+商业模式：CC BY-NC-ND 全文免费引流，出版销售、赞助、视频课程、众筹多渠道变现，11 年跑通。
4. 名人序 + 知识体系化：每册请业界知名开发者作序，第二版补齐练习附录，形成"阅读-思考-练习"闭环。
5. 文档即出版母稿的极简组织：一书一目录、命名规整，任何自动化工具都能直接消费，也便于其他 awesome 列表与课程链接引用。
6. 免费与尊重读者并重：尽管"仓库不为阅读体验优化"，作者仍坚持全文无付费墙、无注册门槛，连练习答案都一并公开。

**风险/局限**

1. **许可证限制再利用**：CC BY-NC-ND 禁止商用与演绎——不能改编成课件、不能修改重分发，给二次创作者留的空间极小（LICENSE.txt）。
2. **内容冻结**：系列完结且不再接受更正，JS 新特性（ES2021+）不在覆盖范围；`types-grammar/ch5` 等占位永远不会被填上。
3. **系列不完整**：第二版实际只出了 4 册（2 完成出版 + 2 草稿），异步与 ES.next 两册取消；需要覆盖这些主题的读者仍要回到 `1st-ed` 分支的第一版（该分支内容本分析未取样，组织细节未确认）。
4. **阅读体验欠优化**：单册单文件长 Markdown、无锚点交叉链接（CONTRIBUTING.md 明言拒绝改进），在 GitHub 网页上通读体验一般。

## 10. 总结评价

这不是一个软件项目，而是 GitHub 上最成功的"书即仓库"案例：18 万+ 星源自它把一套出版级 JS 深度教材完整、免费、合法地放在了开发者每天聚集的地方。系列已完成历史使命并转入冻结存档——第二版 4 册全文 + 第一版 6 册分支构成了 JS 进阶教育的公共基础设施。它适合作为中高级 JS 工程师的系统进阶读物（建议按 README 顺序从 Get Started 读起，配合 apB 练习），适合翻译者按其翻译流程贡献语言分支；但其源码"仓库"本身已无可挖掘的活跃开发价值，后续观察意义在于链接与赞助资源的维护，而非内容演进。对研究开源内容项目商业模式的人，这个仓库的 CC 授权 + 免费全文 + 自出版组合本身就是一份好案例。
