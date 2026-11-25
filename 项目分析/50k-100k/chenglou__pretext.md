# chenglou/pretext 项目分析

> 星数 50886（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：纯 JavaScript/TypeScript 实现的多行文本测量与排版库——不触发浏览器 layout reflow 就能算出任意文字在给定宽度下折成几行、多高、每行长什么样。

| 事实 | 值 | 出处 |
|---|---|---|
| npm 包名 | `@chenglou/pretext`（v1.0.0，2026-10-14 发布） | `package.json`、`CHANGELOG.md` |
| 星数 | 50886（快照）；gh api 实测 50986 | gh api repos/chenglou/pretext |
| 语言/工具链 | TypeScript（tsconfig target ESNext），Bun 运行时构建测试 | `package.json`、`tsconfig.json` |
| 许可证 | MIT（"Copyright (c) 2026 Pretext contributors"） | `LICENSE`、`package.json` |
| 仓库创建 | 2026-03-07，7 个月冲到 5 万星 | gh api |
| 最近推送 | 2026-10-15（快照前约一周，高度活跃） | gh api、文件 mtime |
| 贡献者 | 9 人（含作者 Cheng Lou，前 React 团队成员） | gh api contributors |
| Open issues | 66 | gh api |

## 2. 项目是什么，解决什么问题

Web 平台里"这段文字在 320px 宽的容器里折成几行、总高多少"这个最基本的问题，长期以来只有一条路：把文本塞进 DOM 再读 `offsetHeight`/`getBoundingClientRect`，而这会触发整页 layout reflow——浏览器中最昂贵的操作之一。虚拟列表、masonry、聊天气泡收缩包裹（shrink-wrap）、文字绕排图片等大量 UI 模式因此要么靠估算+缓存，要么忍受性能损耗。Pretext 的答案（`README.md`）：自己实现文本测量与断行逻辑，用浏览器自己的字体引擎（Canvas `measureText`）作为宽度真值源，断行规则则直接移植 Chrome(Blink)、Firefox(Gecko)、Safari(WebKit) 三家引擎各自的断行扫描器，从而"预测"浏览器的换行结果，误差以像素计。

API 分两层（`README.md`）：

1. **纯测量**：`prepare()` 一次性完成空白规整、断行机会切分、Canvas 测宽，返回不透明句柄；之后 `layout(prepared, maxWidth, lineHeight)` 是纯算术热路径，零 DOM、零字符串分配。还提供 `measureLineStats()`、`walkLineRanges()`、`measureNaturalWidth()` 等免序列化变体。
2. **手动逐行排版**：`prepareWithSegments()` + `layoutWithLines()` / `layoutNextLine()` 返回每行的文本、宽度与起止游标，可逐行 `fillText` 到 Canvas/SVG，或实现文字绕浮动图片、`-webkit-line-clamp` 式省略号等 CSS 给不了的自定义排版（`README.md`、`pages/demos/dynamic-layout.ts`）。

另有 `@chenglou/pretext/rich-inline` 子入口（`src/rich-inline.ts`）：把多字体、@提及、行内图片/表情/徽章等富内联片段排成一个段落，行断点与浏览器排同一段 HTML 一致，每行输出带 `gapBefore`/`sourceStart` 的 fragment 列表，供逐行绘制。

README 明确不做的：不是完整字体渲染引擎，不给 bidi 视觉顺序（建议混合双向文本整段交给 DOM 排），只覆盖 `white-space: normal/pre-wrap`、`word-break: normal/keep-all`、`overflow-wrap: break-word` 这一常见组合（`README.md` Caveats 节）。

项目诞生语境（`thoughts.md`）：作者认为"用户代码若能直接控制文本，CSS 规范 80% 可以绕开"，目标是让开发者做出更有表现力的布局而不必等 Web 标准放行；README 还强调这套"以浏览器字体引擎为真值"的迭代方式"对 AI 友好"。

## 3. 技术栈与依赖

- **语言**：TypeScript（strict 全家桶：`noUncheckedIndexedAccess`、`exactOptionalPropertyTypes`、`verbatimModuleSyntax` 等，见 `tsconfig.json`）；TypeScript 编译器版本钉在 `7.0.2`（`package.json`）。
- **运行时/工具**：Bun 承担安装、测试（`bun test --parallel`）、demo 服务器（`bun start`，经 `pages/*.html` 直接跑 HTML 服务）与脚本；构建产物仅 `tsc -p tsconfig.build.json` 编译到 `dist/`，无打包器。
- **devDependencies 极简**（`package.json`）：`typescript`、`oxlint` + `oxlint-tsgolint`（type-aware lint）、`knip`（死代码检查）、`marked`（demo 站 Markdown 渲染）、`@types/bun`。运行时零依赖。
- **引擎数据**：`scripts/engine-data/` 内直接检入三家引擎的原始断行/字形数据（`chrome-153/line_normal.brk`、`firefox-156/segmenter_break_line_v1.rs.data`、`safari-27.0/BreakablePositions.cpp` 等），由 `scripts/generate-engine-break-data.ts`（898 行）编译成 `src/generated/engine-break-data.ts`（35KB 打包 varint/字符串表）。
- **测试浏览器**：Chrome/Firefox 用钉版本的本地副本，Safari 走系统 WebKit 经自建 `webkit-host` 后台应用驱动（`harness/README.md`）。

## 4. 目录结构与代码组织

单包仓库，无 monorepo；库本体极小、外围"证据链"极重。

| 目录 | 职责 | 出处 |
|---|---|---|
| `src/` | 库本体约 6700 行 TS（不含测试）：分析、断行、测量、排版、富内联 | `src/*.ts` |
| `src/generated/` | 生成物：引擎断行/字形数据表、WebKit generic family 表 | `src/generated/` |
| `harness/` | 浏览器行为测试装置：录制-预测-评分、基准测速、不变量检查、webkit-host | `harness/README.md` |
| `harness/cases/` | 约 66MB NDJSON 录制语料：catalog.ndjson 30.7MB/38145 例为主 | `ls harness/cases/` |
| `corpora/` | 跨语言测试文本（英/中/日/韩/泰/高棉/缅甸/阿拉伯/希伯来/印地/乌尔都等 17 种），`sources.json` 记录来源 | `corpora/` |
| `scripts/` | 引擎数据生成器、demo 站构建、包冒烟测试、grapheme 检查 | `scripts/` |
| `pages/` | demo 服务器与 10+ 个示例（聊天、气泡、masonry、省略号、绕排、对齐对比等） | `pages/demos/` |
| 根文档 | `RESEARCH.md`（研究日志+决策日志）、`PLATFORM_BUGS.md`（向各引擎上报的 bug 台账）、`ENGINE_FOLLOWUPS.md`、`TODO.md`、`AGENTS.md`、`DEVELOPMENT.md`、`thoughts.md`、`SECURITY.md` | 顶层 |

组织方式评述：这不是"库+示例"的常规结构，而是一座工程证据库——每个架构决策都有 `RESEARCH.md` 决策日志编号，每个浏览器怪癖都在 `PLATFORM_BUGS.md` 立案（含向 Chromium/WebKit/Mozilla 上报的 issue 号），录制语料直接进 git 以保证离线可复现（`DEVELOPMENT.md` "Engine Data" 节明言 30MB catalog "删了也不省体积，git 历史里还在"）。

## 5. 核心架构与关键实现

### 5.1 三引擎断行移植（`src/line-breaks.ts`、`src/gecko-line-breaks.ts`）

这是项目的灵魂。与其自创断行规则，Pretext 把 Blink、WebKit、Gecko 的断行扫描逐文件行号级移植（`src/gecko-line-breaks.ts` 头注释：源引 Gecko at Firefox 156.0、icu_segmenter 2.1.2 src/line.rs；`src/line-breaks.ts` 头注释：源引 ICU 78.2 in Chromium 152、Apple libicucore 78.1）。文本先按"段类型"打标（`src/analysis.ts` 的 SegmentBreakKind：`text/space/preserved-space/tab/zero-width-break/soft-hyphen/zero-width-glue/hard-break/control`），断行扫描再在段序列上找机会点。

### 5.2 测量与浏览器怪癖修正（`src/measurement.ts`，1458 行，最大模块）

维护三份 `EngineProfile`（chrome/gecko/webkit），处理大量已知平台 bug：如 Canvas 在小字号下把 emoji 测宽的 Chromium/Mozilla bug，Pretext 的对策是每个字体读一次隐藏 DOM span 求"emoji 加宽量"，再从 `countEmojiGlyphs` 逐字形扣除（`src/prepare.ts` 头注释 + `PLATFORM_BUGS.md` "Canvas measures emoji too wide" 节）。还有 Han 标点挤压（`src/han-kerning.ts`，逐条引用 Blink `han_kerning.cc` 行号的 CJK 全角标点 trim 规则）、CJK 脚本 run 判断（`src/prepare.ts` 头部 40 行注释详解 Blink `ScriptRunIterator` 的位表示）、`<html lang>`/`setLocale()` 对断行规则与字体选择的影响等。

### 5.3 prepare → layout 两段式与并行数组句柄（`src/layout.ts`、`src/line-break.ts`）

`prepareInternal()`（`src/layout.ts:108`）一次语言读取、一次分析、一次测量，产出带品牌类型的句柄：

```ts
export type PreparedText = {
  readonly [preparedTextBrand]: true
}
```

注释明言"故意让句柄不透明，避免公共 API 围绕当前的并行数组表示固化"（`src/layout.ts:42-45`）。内部数据是逐段的 `widths: number[]` + `segmentFlags: Uint8Array`（`src/line-break.ts` 的 `PreparedLineBreakData`），并有 `simpleLineWalkFastPath` 标志让普通文本走单数值循环的 `layout()` 快路径。断行器产出 `LayoutCursor{segmentIndex, graphemeIndex}` 游标而非字符串偏移，`materializeLineRange()` 才按需构造行文本——这正是 `walkLineRanges()` 能零分配做"二分搜索最优宽度"的基础。

### 5.4 富内联合并段落模型（`src/rich-inline.ts`，1268 行）

1.0.0 的最大架构变更（`CHANGELOG.md` 1.0.0 Changed 节）：0.0.9 逐 item 独立排版导致跨 item 断行错误；现在把所有 item 的文本拼接成一个段落做一次断行分析，每个 item 只负责在自己的字体下测宽，使行断点与浏览器排同一段富文本完全一致，并补齐软连字符、bidi 控制符空白塌缩等边界语义。

## 6. 构建与运行

均出自 `package.json` scripts 与 `DEVELOPMENT.md`：

```bash
bun install                      # 安装
bun test --parallel              # 单元测试 + harness 离线测试 + 文档引用检查（无浏览器）
bun run check                    # tsc×2 + oxlint(type-aware) + knip + 生成数据一致性检查
bun start                        # demo 服务器，http://localhost:3000/demos（Windows: bun run start:windows）
bun run build:package            # tsc -p tsconfig.build.json 产出 dist/（prepack 自动执行）
bun run package-smoke-test       # 打包并导入产物做冒烟测试
bun run generate:engine-break-data   # 从 scripts/engine-data/ 再生成数据表
bun harness bench <tag>          # 三浏览器基准测速（见 harness/README.md）
```

## 7. 测试与工程质量

- **单元测试**：`src/layout.test.ts` 8198 行、191 个顶层 test；harness 侧 `harness.test.ts`（1366 行/79 个）、`invariants.test.ts`、`icu.test.ts`（仅 macOS，对照系统 ICU 即 Safari 真值）。断言风格覆盖 API 输出、行数一致性、跨 API 不变量（"所有行 API 必须对行/宽/文本达成一致，且 prepare 后不得再调 measureText"，`harness/README.md`）。
- **行为 harness（核心创新）**：`harness/cases/catalog.ndjson` 38145 个"段落×宽度"用例，浏览器实测结果录制进 git；每次改动在真实浏览器中重新预测并评分（行数错= `count`，断点错= `breaks`），使精度主张可量化、可离线回归（`harness/README.md`、`DEVELOPMENT.md`）。另有 bench（性能回归）、grapheme check（对三引擎 `Intl.Segmenter` 全量比对）。
- **质量机制**：`bun run check` 串联双 tsc、type-aware oxlint（deny-warnings）、knip、以及"生成表必须与检入的引擎源数据一致"的检查（`scripts/generate-engine-break-data.ts --check`）。release 流程（`DEVELOPMENT.md` Releasing 节）要求对上一版本实测性能数字后才写进 CHANGELOG。
- **CI**：仅 `.github/workflows/pages.yml`（每次 push main 发布 demo 站到 GitHub Pages）。测试不进 CI——浏览器 harness 依赖钉版本浏览器与 macOS 环境，只能本地/作者机器跑（未确认是否有其他 CI 源）。
- **文档质量**：极高。CHANGELOG 的每条目带 PR 号、性能倍数与机器环境；`PLATFORM_BUGS.md` 是一份可以直接当浏览器排版 bug 教科书读的台账。

## 8. 社区与维护状态

- **节奏**：2026-03-07 建仓，约半年内从 0 做到 1.0.0（2026-10-14 发首版正式 release，gh api releases 仅此一条），之后有一段极密集的 0.x 迭代（CHANGELOG 中 #228→#525 的 PR 编号跨度）。快照前一周（2026-10-15）仍有推送。
- **贡献者结构**：9 人（gh api contributors），本质上是作者主导、少量协作者的项目；issue 区 66 个开放 issue，CHANGELOG 显示 PR 由作者逐一评审合并。
- **上游影响力**：`PLATFORM_BUGS.md` 记录了向 Chromium、Mozilla、WebKit 提交的多份 bug 报告（如 Chromium #489494015、Mozilla #2020894、WebKit #283408），库的行为随上游修复自动"解锁"。
- **风险点**：`TODO.md` 出现"End of project"分区，部分能力（如 bidi、worker emoji 修正）标记为待上游或待决策，未确认项目 1.0 后的长期投入计划。

## 9. 亮点与风险

**亮点**

1. 问题选得准且曾是空白：免 reflow 的多行文本测量是虚拟化/masonry/聊天气泡的公共痛点，README 列举的四类收益均有对应 demo 实证（`pages/demos/`）。
2. 工程方法论罕见地严谨：断行逻辑按引擎逐行移植并注明 `文件:行号` 出处（`src/line-breaks.ts`、`src/gecko-line-breaks.ts`），38145 例录制行为语料作为可回归的真值库，精度主张全部可复现。
3. API 设计分层清晰：`prepare()` 预计算 + `layout()` 纯算术的热路径分离，范围游标（LayoutCursor）与惰性物化（materialize）避免字符串分配，`walkLineRanges()` 支撑宽度二分等高阶用法。
4. 文档与证据链一流：RESEARCH.md 决策日志、PLATFORM_BUGS.md 引擎 bug 台账、AGENTS.md（面向 AI 协作者的仓库地图）共同构成"为什么是这样"的完整论证，对 AI 辅助开发时代极具示范意义。
5. 兼容矩阵诚实：README 的 Caveats 与 PLATFORM_BUGS.md 把每个已知不准确场景、每个上游 bug 都写明，甚至给出绕行办法。

**风险/局限**

1. 体积与启动成本上升：1.0.0 主入口约 40KB gzipped（0.0.9 为 16KB），其中约 23KB 是三引擎断行/字形数据（`CHANGELOG.md` 1.0.0 Changed 首条）。
2. 环境依赖刚性：必须有 Canvas 2D（Node/Bun/jsdom 直接抛错）与 Unicode property escapes，东南亚无空格文字还需 `Intl.Segmenter`；worker 场景缺 emoji 修正（`README.md` Notes、PLATFORM_BUGS.md）。
3. 平台验证偏科：全部测量基于 macOS + Chrome 153/Firefox 156/Safari 27 钉版本，`PLATFORM_BUGS.md` 自述"Windows is untested"；Linux/移动端与未钉版本的保真度未确认。
4. 维护集中度高：核心逻辑（尤其是引擎移植与数据再生）高度依赖作者个人的持续投入，9 名贡献者中实际承担者更少（gh api contributors 结合 CHANGELOG 判断）。

## 10. 总结评价

Pretext 是 2026 年前端基础库中少见的"以一己之力把一个平台级空白填掉"的项目：它没有发明新排版规则，而是把三家浏览器的断行器逐行移植、把浏览器字体引擎当作测量真值，再用一套录制-评分 harness 把"和浏览器一致"变成可量化、可回归的工程承诺。7 个月 5 万星的爆发，本质上是对"前端长期忍受 DOM reflow 做文本测量"这一痛点的精准一击，叠加作者在 React 社区的声望与教科书级的文档证据链。它适合三类人：正在做虚拟列表/masonry/聊天气泡/富文本编辑器、被文本高度不确定性折磨的应用开发者直接引入；做 Canvas/SVG 自绘排版、需要逐行控制的高级用户读 `layoutWithLines`/`rich-inline` 一层；而任何想学习"如何用工程方法驯服浏览器黑盒"的人，都值得把 `RESEARCH.md`、`PLATFORM_BUGS.md` 和 `harness/` 当作范本通读。
