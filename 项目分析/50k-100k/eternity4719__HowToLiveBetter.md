# eternity4719/HowToLiveBetter 项目分析

> 星数 68268（快照 2026-10-24）｜主语言 HTML｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：《高性价比人生指南》——一本面向中国大陆的循证人生决策开源书，767 条建议每条写明成本、收益、证据等级和原始文献出处，"用最少的钱、时间和精力，换回最多的寿命、金钱和人身自由"（README.md）。

| 项目 | 事实 |
| --- | --- |
| 名称 | HowToLiveBetter（高性价比人生指南） |
| 星数 | 68268（任务快照）；查询时 API 已达 71614，仍在快速增长 |
| 主语言 | HTML（实为"Markdown 内容 + 单文件静态检索页"，HTML 只占 `index.html` 一个文件） |
| 许可证 | 正文 CC BY 4.0（`LICENSE`）；代码 MIT（`LICENSE-CODE`） |
| 创建时间 | 2026-09-07（gh api repos/eternity4719/HowToLiveBetter） |
| 最近 push | 2026-10-21（同上） |
| 默认分支 | main；主页 https://eternity4719.github.io/HowToLiveBetter/ |
| 体量 | 正文 34 节 767 条，`book/` 2.4MB；`docs/` 3.3MB；仓库约 19MB |

## 2. 项目是什么，解决什么问题

这是典型的"内容型仓库"：没有产品代码，正文本身就是全部产出。它想回答的是一类非常具体的问题——"这件事划不划算、该不该做"。覆盖面刻意做得很宽：防早死与慢病、急救、省钱理财、法律红线、失业与工伤、医保社保、恋爱婚育、怀孕育儿、创业合规、出国留学，共 34 节（README.md 目录章节，`book/01`–`book/34`）。

与同类书最大的差别在方法论。每条建议是固定六字段结构：成本、"说人话"（把统计量翻成日常话）、收益（保留 HR/RR/置信区间原值）、证据等级（A/B/C）、来源（只引期刊论文 DOI/PubMed 与 WHO/CDC/官方文件）、备注（争议、适用人群）。据实测 grep 统计：A 级 495 条、B 级 220 条、C 级 52 条、备注以"争议"开头 88 条，与 README 徽章完全一致。`CLAUDE.md` 明确禁止引用知乎、公众号等二手转述（CLAUDE.md 引用规则节）。

项目把"收益"分成四种不互相折算的口径：寿命、时间与精力、金钱、人身自由（README"四种资源"节）。还把"好处落在谁身上"分为自己/直系亲属/朋友/陌生人四档，急救等涉他条目要求把被讹、被报复的风险面一起写（CLAUDE.md 受益人口径节）。这种"证据等级"与"性价比档"正交、口径之间不比较的设计，是全书算账体系的底座。

书明确声明在 Claude Code 协助下编写：AI 负责查文献、核对数字、维护交叉引用和构建脚本，选题与审定由作者负责（README 尾部声明）。风格上参考了 geekan/HowToLiveLonger 的量化路线但范围更大（CLAUDE.md 项目定位节）。

商业化路径也已经跑通：电子书平台的官方版本由果麦文化独家出品，2026 年 10 月上架；网页版、EPUB、PDF 照旧免费（README.md）。

## 3. 技术栈与依赖

- **内容**：纯 Markdown（`book/*.md` 34 节 + `docs/*.md` 10 篇长文 + 239 篇核实记录），条目内嵌 HTML 注释成本标签供机器解析（`book/01-不要早死.md`）。
- **检索页**：`index.html`（约 110KB 单文件）+ `selfcheck.html`（生活自查）。零依赖、无后端、无数据库，直接 `fetch()` 本仓库 `book/*.md` 作为数据源（index.html 1380 行附近）。
- **构建工具**：`tools/` 下约 1846 行零依赖 Node.js（ESM）脚本；唯一第三方依赖是 EPUB 构建用的 `marked@18`（tools/epub/package.json）。Anki 牌组生成用 Node ≥ 22.13 自带的 `node:sqlite`（tools/anki/build.mjs）。
- **排版链**：PDF 用 pandoc ≥ 3.1 + typst ≥ 0.13（README"自己跑一份"节）。
- **CI/CD**：GitHub Actions 两条流水线（`.github/workflows/book.yml`、`links.yml`）；GitHub Pages（Deploy from a branch，`.nojekyll`）托管检索页。
- **AI 集成**：`skills/life-decision-guide/` 提供 Claude Code / Codex 双兼容的 SKILL.md，`.claude/skills/` 是指向正本的薄指针，`AGENTS.md`/`CLAUDE.md` 是 agent 工作规范。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `book/` | 正文 34 节，每节一个 md 文件（单文件超 GitHub 512KB 渲染上限才拆分，README"正文"节） |
| `docs/` | 10 篇场景长文 + `核实记录/` 239 篇逐条核实档案 + `引用对照.md` |
| `index.html` | 在线检索页：按关键词/章节/证据等级/成本/人群筛选，性价比档实时计算 |
| `selfcheck.html` | 生活自查：几十道是否题，定位读者没做到的条目 |
| `tools/` | 14 个构建/校验脚本 + `lib/` 共用解析器（`lib/book.mjs`、`lib/split-items.mjs`、`lib/zip.mjs`） |
| `tools/epub|pdf|anki|offline/` | 四种发布格式的生成器，产物进 `dist/` |
| `skills/`、`.claude/` | AI 助手 skill（正本 + 指针，避免两处不一致） |
| `.github/` | 两条 workflow + 纠错/新内容两个 issue 模板 + FUNDING |
| `README.md` | 不只是门面：同时是检索页和 EPUB/PDF 构建的结构数据源（tools/lib/book.mjs 直接解析 README 标题分段） |
| `updates.json` | 近期条目变更索引，供检索页显示"最近更新"（tools/updates.mjs 生成） |
| `CLAUDE.md` | 116KB 编辑规范正本：格式、引用、语言风格、统计同步等全流程规则 |

组织方式评述：这是"数据即文件"的单体结构——Markdown 是唯一事实源，四种发布格式、检索页、AI skill 全部从它派生。没有 package.json 在根目录（仅 `tools/epub/` 有），根目录对普通读者就是一本书，对机器是一个可解析的数据集。

## 5. 核心架构与关键实现

**（1）条目数据结构：人读和机读共用一份 Markdown。** 每条以三级标题 + 一行 HTML 注释成本标签构成，如 `book/01-不要早死.md` 第 1 条：

```markdown
### 1. 系安全带，前排后排都系
<!-- 成本标签: 钱=0 时间=少 毅力=否 收益=大 口径=死亡率 -->
- 成本：不花钱。每次上车多花 2 秒。
- 说人话：坐轿车前排系上安全带，出车祸时被撞死的概率降四成多。……
- 收益：美国交通安全主管部门（NHTSA）估计，……下降 45%。……
- 证据等级：A
- 来源：NHTSA (2024). ……
- 备注：后排也要系。……
```

`人群=` 可选标签（怀孕/婴幼儿/老人/35岁/60岁/女性等 7 值，逗号或、加号且）支持"我的情况"侧栏筛选（CLAUDE.md 条目格式节；值表在 index.html 的 `WHO_LABEL` 与 tools/sync-stats.mjs 的 `WHO`）。

**（2）性价比档算法：三行代码的透明公式。** 检索页按收益量级与三项成本合成档位，权重与判定规则全部暴露在前端（index.html 1034、614-615 行）：

```javascript
const COST_W = { money:{'0':0,'少':1,'多':2}, time:{'少':0,'中':1,'多':2}, will:{'否':0,'些':1,'是':2} };
e.cs = (COST_W.money[e.money] ?? 0) + (COST_W.time[e.time] ?? 0) + (COST_W.will[e.will] ?? 0);
e.ratio = e.level === '大' ? (e.cs === 0 ? '极高' : (e.cs <= 2 ? '高' : '一般')) : ...
```

全书 767 条中"极高"123 条（16%）、"高"344 条（45%）、"一般"300 条（39%）（README 性价比档节）。AI skill 甚至要求 LLM 回答前现场 `grep` 这两行照着算，不许凭记忆（skills/life-decision-guide/SKILL.md 第 4 步）。

**（3）交叉引用防错位系统。** 正文约 257 处"第 X 条"引用只记条号不记内容，插入条目会让后续条号集体顺延且通常仍在范围内，越界检查抓不到。`tools/check-refs.mjs`（321 行）把每处引用实际指向的条目标题写进入库的 `docs/引用对照.md`，git diff 直接暴露"条号没变而指向变了"的错位；CI 单独 job 跑 `--check`（check-refs.mjs 头部注释记录了 2026-09-19 第 7 节 6 处指错的起因）。

**（4）统计数字单一来源同步。** 条目数、A/B/C 数、争议数、链接数等散布在 README 九处、index.html 四类位置、og.png 三处，`tools/sync-stats.mjs` 一键重算回写，CI 跑 `--check` 防止网页上直接合并的 PR 绕过（CLAUDE.md 条目格式节）。配套 `tools/updates.mjs` 从 git 历史生成 `updates.json`：实测 since 2026-10-01 约 3 周内 393 条变更（136 新增、257 修改），可见维护强度。

**（5）AI skill 作为"第六种发行格式"。** `skills/life-decision-guide/SKILL.md`（118 行）定义了完整的 RAG 式回答流程：急症/自杀念头/法律程序三类问题先拦截 → 浅克隆或 curl 取正文 → grep 定位条目 → 按 index.html 公式排序 → 固定答复结构，并强制"没查到就别答""数字照抄条目一个不改"。这使整本书可以被任何 AI 客户端当作可信知识库调用。

## 6. 构建与运行

来自 README"自己跑一份"节（原文给出的命令）：

```bash
# 本地预览检索页（纯静态，必须经 http 打开，双击 file:// 会空白）
python -m http.server 8000        # 浏览器开 http://localhost:8000/

# 生成四种发布格式（在仓库根目录，产物在 dist/）
npm --prefix tools/epub ci && npm --prefix tools/epub run build   # EPUB（依赖 marked）
node tools/offline/build.mjs      # 离线单文件 HTML（含检索与筛选）
node tools/pdf/build.mjs          # PDF，另需 pandoc ≥ 3.1 和 typst ≥ 0.13
node tools/anki/build.mjs         # Anki 牌组，要 Node ≥ 22.13

# 校验（CI 同款）
node tools/check-refs.mjs --check; node tools/check-plain.mjs
node tools/check-split.mjs; node tools/sync-stats.mjs --check
node tools/updates.mjs --check; node tools/check-selfcheck.mjs
node tools/check-links.mjs        # 全量外链检查约一分钟
```

## 7. 内容质量保障机制（特例：替代"测试与工程质量"）

- **CI 六道检查 + 构建**（`.github/workflows/book.yml`）：交叉引用检查、说人话检查（`check-plain.mjs` 查长度/研究行话/抽象说法，2026-09-28 一次查出 173 条不合格并全部改写，CLAUDE.md）、拆分规则一致性、统计数字检查、自查条件检查，之后构建四种格式并跑 epubcheck 5.4 校验，发布到固定 tag `epub-latest`（下载链接不变）。PR 只校验不发布。
- **每周外链体检**（`links.yml`）：每周一跑全书外链，只报告不报错；"确定失效"（404/410/DOI 查无）才修，并区分了 gov.cn 挡境外 IP、老政府站加密握手不兼容等误报场景（links.yml 头部注释）。
- **239 篇核实记录**（`docs/核实记录/`）：每处纠错、补来源、口径变更都有独立档案，含日期、改动条号、关键原文短句和推理过程，抽查 `docs/核实记录/纠错-2026-10-21-个人养老金.md`：纠正了"个人养老金只能退休取"的遗漏（补人社部发〔2025〕39 号新增领取条件）并重算跨档节税例子。这是全书可审计性的核心证据。
- **语言质量硬标准**：CLAUDE.md 记录了 2026-09-21 全书通俗化改写——句子平均长度从 66 字降到 29 字，超 70 字句子从 1385 句降到 64 句；并系统总结了"AI 腔" blacklist（元叙述、段尾升华、连用"也就是"等）及保真改写规则（取自读者推荐的 shuorenhua skill）。
- **零依赖工具链本身即质量信号**：`tools/` 14 个脚本 1846 行，注释里写满设计决策与历史事故复盘，工程水准高于多数同规模内容仓库。

## 8. 社区与维护状态（特例：含生态位）

- **活跃度**：创建于 2026-09-07，至快照日不到两个月冲到 68k+ 星；最近 push 2026-10-21（合并 PR #149 正文审校）；`updates.json` 显示 10 月单月改了 393 个条目。创建两个月即有如此迭代密度，属于当期 GitHub 最活跃的内容仓库之一（gh api commits/releases）。
- **贡献者结构**：21 名贡献者（gh api contributors）。主体模式是"作者 + AI 协作写正文、读者提 issue 补纠错"：累计 159 个 issue，两个 issue 模板（纠错/新内容）都强制要求给出期刊论文、法条原文等"依据"，并明确拒收知乎/公众号转述（`.github/ISSUE_TEMPLATE/1-纠错.yml`）。核实记录中大量"追加-issue30/issue64…"文件证明 issue 驱动的条目增补已流程化。
- **Release 节奏**：只有一个滚动 release `epub-latest`，正文每次更新自动重建四格式并覆盖上传，下载链接永久不变（book.yml）。
- **生态位与衍生网络**：`衍生项目.md` 收录十余个读者做的阅读站、小程序、iOS App、闯关游戏、MCP 服务；三个翻译项目覆盖英/俄/西/葡/阿/印尼/越南语（README"其他语言"节）。原书 CC BY 4.0 允许衍生，衍生项目统一"以本仓库中文原文为准"，形成以中文正本为中心的辐射结构。
- **商业化**：果麦文化独家电子书上架（README），页内有微信赞赏码和一个 Minecraft 服务器广告位（ads/ 目录，商务邮箱 admin@mcyyy.com）。

## 9. 亮点与风险

**亮点**

1. **方法论文档化到罕见的程度**：证据分级、受益人四档、口径不折算、性价比公式全部成文且机器可执行（CLAUDE.md + index.html + SKILL.md 三处互锁），"说人话"栏让统计结论对非专业读者可用。
2. **可审计性极强**：767 条的每处改动有 239 篇核实记录 + `docs/引用对照.md` diff + CI 六道检查兜底，错误能被定位、回放和追责。
3. **一份 Markdown 派生六种出口**：网页检索、离线单文件、EPUB、PDF、Anki、AI skill，全部自动化且下载链接稳定，发行工程是内容仓库的范本。
4. **网络效应已形成**：多语言翻译、十余个衍生站点/App、MCP 接入，均按 CC BY 4.0 规范归流回中文正本。
5. **对 AI 时代内容生产的工程化示范**：从"AI 协助编写"到"防 AI 腔规范"到"防 AI 幻觉的核实档案"（`docs/核实记录/追加-AI幻觉.md`），形成完整闭环。

**风险/局限**

1. ** jurisdiction 局限**：法律、医保、社保内容按中国大陆现行规定写，政策地区差异大且变动频繁，书内已自认"可能过时、可能有错"（README）；时效性条目（如第 32 节留学签证新规）依赖持续人工维护。
2. **医学内容的责任边界**：虽有证据分级和"不替代医生"声明，68k 星传播后条目易被断章摘引，"说人话"一行脱离备注使用可能误导（项目已在 SKILL.md 里防御，但无法约束转发截屏场景）。
3. **单人审定瓶颈**：选题与最终审定集中于作者一人（README 声明），21 名贡献者主要做纠错；作者停更则时效性条目整体衰减。
4. **星数与内容消耗错位**：这类"查得到但不常读"的工具书，star 更接近收藏行为；长期价值取决于检索页与 AI skill 的实际调用量，此数据仓库内不可见（未确认）。

## 10. 总结评价

这是近两年中文内容型开源仓库里工程化程度最高的一个：把"人生建议"做成了带证据等级、成本标签、交叉引用校验和六格式自动发行的数据集，并用 239 篇核实记录为每处改动留痕；两个月 68k 星说明"循证 + 性价比 + 说人话"的内容配方与 GitHub 的分发机制契合，而围绕它的翻译、衍生站和 MCP 生态已经自转。它不适合从头读到尾（作者自己也这么劝），适合当一本可检索、可让 AI 代查的决策参考书；对想研究"如何用软件工程方法维护一个可信内容仓库"的人，`CLAUDE.md` + `tools/` + `docs/核实记录/` 这条链路本身就是最好的源码。
