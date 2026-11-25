# markedjs/marked 项目分析

> 星数 39,500（快照 2026-10-24，gh api 2026-11-17 实测 37,306）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-17

## 1. 项目速览

一句话定位：marked 是一个"为速度而生"的轻量级 Markdown 解析/编译器，零运行时依赖，浏览器、Node、CLI 三端通用，是 npm 生态中被依赖最多的 Markdown 引擎之一。

| 项目 | 内容 |
|---|---|
| 名称 | markedjs/marked（npm 包名 `marked`） |
| 星数 | 39,500（快照）；37,306（gh api 实测 2026-11-17） |
| 主语言 | TypeScript（src/ 下 13 个 .ts 文件，约 4,400 行） |
| 许可证 | MIT（`package.json` 声明 MIT；根 `LICENSE` 文件为"CLA + MIT 正文"混合格式，故 GitHub API 显示 NOASSERTION） |
| 当前版本 | v18.2.0（`package.json`，与最新 release 一致） |
| 最近活跃 | 最近 push 2026-11-16（gh api，前一天仍在提交），开放 issue 仅 31 个 |
| 默认分支 | master |
| 生态位 | CommonMark + GFM 兼容的"底层编译器"，不缓存、不阻塞、不做 sanitize |

## 2. 项目是什么，解决什么问题

marked 最初由 Christopher Jeffrey 于 2011 年编写（`README.md` 版权行：2011-2018 Christopher Jeffrey，2018+ MarkedJS 社区组织接管），目标是把 Markdown 文本编译成 HTML 字符串，核心卖点自始至终是"built for speed"（`package.json` description 与 `README.md` 首条特性）。它定位为 low-level compiler：不做缓存、不阻塞长时间运行，适合单次 parse 需要毫秒级完成的场景，如编辑器实时预览、静态站点生成、CLI 管道（`README.md`）。

与"大而全"的解析器不同，marked 刻意保持内核极小：全库零运行时依赖（`package.json` 无 `dependencies` 字段），核心只有词法分析（Lexer）、块级渲染（Parser）、行内 token 化（Tokenizer）、HTML 渲染（Renderer）四个类加一张正则规则表。功能扩展（脚注、数学公式、GFM 之外的一切语法）一律交给 `marked.use()` 扩展机制，官方文档明确写着"非规范特性不进核心"（仓库根 `AGENTS.md` 第 2 节 Extension-First Architecture）。

兼容性上，marked 以 CommonMark 0.31.2 与 GFM 为目标规范，仓库自带完整的规范符合度测试集（`test/specs/commonmark/`、`test/specs/gfm/`），同时用 250 个自建回归用例（`test/specs/new/`）守住历史行为。但对 marked 而言"速度优先于边角规范符合度"是写进贡献者守则的决策原则（`AGENTS.md`："Speed is #1 Priority…… Minor edge-case spec divergences can be tolerated"）。

安全模型上，marked 从 v8.0.0 起彻底移除了内置 `sanitize`/`sanitizer` 选项（`docs/USING_ADVANCED.md` 选项表标注"Removed in v8.0.0"），README 以醒目警告要求用户自行在输出 HTML 上调用 DOMPurify 等净化库。这是 marked 最著名的社区争议点：官方立场是"单一职责"，解析器只负责 Markdown→HTML，XSS 防护责任移交下游。

## 3. 技术栈与依赖

- **语言**：TypeScript（`tsconfig.json` 严格模式），发布产物为 ESM（`lib/marked.esm.js`）+ UMD（`lib/marked.umd.js`）双格式，`package.json` 中 `"type": "module"`，Node >= 20。
- **运行时依赖**：零（`package.json` 无 dependencies 字段，`AGENTS.md` 明文规定核心不得加运行时依赖）。
- **构建**：esbuild 打包（`esbuild.config.js`），dts-bundle-generator 生成单文件类型声明（`package.json` build:types 脚本）。
- **测试相关 devDeps**：`node:test` 内置测试器（无第三方测试框架）、`@markedjs/testutils`（规范用例加载与 HTML 对比）、`recheck`（正则回溯静态分析）、`@arethetypeswrong/cli`（类型导出验证）、`commonmark`/`markdown-it`（基准对比对象，`test/bench.js`）、`highlight.js`、`cheerio`。
- **发布**：semantic-release + Conventional Commits，GitHub Actions OIDC 提供 npm provenance（`.releaserc.json`、`.github/workflows/tests.yml` Release job、`package.json` publishConfig.provenance）。
- **工程特色**：仓库根有 10KB 的 `AGENTS.md`，专门写给 AI 编码代理，罕见地把项目哲学、优先级标签体系（L0 安全 > L1 崩溃 > L2 烦人 > RR 重构 > NFS/NFU 新特性）、提交规范系统化成文。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `src/`（内层） | 全部核心源码，13 个 TypeScript 文件，约 4,400 行 |
| `src/Lexer.ts`（701 行） | 块级词法分析器：逐条尝试块级 tokenizer 规则表，驱动行内解析队列 |
| `src/Tokenizer.ts`（1251 行） | 全部块级/行内 token 的具体切分实现（最大文件） |
| `src/rules.ts`（647 行） | 所有正则规则：normal/gfm/pedantic 三套块级与行内规则表 |
| `src/Parser.ts`（206 行） | 块级 token 流 → 调 Renderer 产出字符串 |
| `src/Renderer.ts` / `TextRenderer.ts` | HTML 渲染器 / 纯文本渲染器（可整体替换） |
| `src/Instance.ts`（404 行） | `Marked` 类：`use()` 扩展注册、`walkTokens`、异步 parse 流水线 |
| `src/Hooks.ts`（69 行） | preprocess/postprocess/processAllTokens/emStrongMask 等生命周期钩子 |
| `src/MarkedOptions.ts` / `Tokens.ts` | 全部公共类型：选项接口与 22 种 token 类型定义 |
| `src/marked.ts`（166 行） | 默认单例外观层：`marked.parse`/`use`/`walkTokens` 及全部导出 |
| `src/helpers.ts`（328 行） | escape、cleanUrl、splitCells、emStrong closer 扫描缓存等工具 |
| `bin/` | CLI 入口（`bin/marked.js` → `bin/main.js`，297 行，支持 stdin/配置文件） |
| `test/` | specs（commonmark/gfm/new/original/redos 五套）+ unit（6 个 .test.js）+ bench |
| `docs/` | 官网源码（marked.js.org，自举：文档站本身用 marked 渲染） |
| `man/` | man 手册源文件（marked-man 生成） |
| `.github/workflows/tests.yml` | CI + semantic-release 发布流水线 |

组织方式评述：典型的"单包极简内核"结构，无 monorepo、无子包；`src/` 内外两层同名目录（仓库根 `src/` 与 `src/src/`）是 tarball 解包的视觉产物，实际源码全在 `src/src/`。代码组织高度面向扩展点设计：每个类的方法都可被 `marked.use()` 逐个覆写。

## 5. 核心架构与关键实现

### 5.1 两阶段流水线：块级 lex → 行内 lex → 块级 parse

编译主流程在 `src/Instance.ts` `parseMarkdown()`（310-382 行）：`hooks.preprocess → Lexer.lex → processAllTokens → walkTokens → Parser.parse → hooks.postprocess`，`async: true` 时同一流水线全程 await。`src/Lexer.ts` `lex()`（101-117 行）先跑 `blockTokens` 把源文切成块级 token，期间所有行内文本只进 `inlineQueue` 队列，块级扫描结束后统一跑 `inlineTokens`——这是典型的两遍式设计，保证引用定义（link definitions）可以先收集完再解析行内 reflink。

### 5.2 块级循环 = 规则表 + fail-fast tokenizer（Lexer.ts）

`blockTokens()`（122-318 行）是一个显式优先级循环：扩展 block tokenizer 最先尝试，然后按 space → code → fences → heading → hr → blockquote → list → html → def → table → lheading → paragraph → text 的固定顺序逐个调用 `Tokenizer` 方法；每个 tokenizer 用单条锚定正则匹配源文头部，命中即消费 `token.raw.length` 并 continue。两道防线保证不死循环：`srcLength` 单调递减校验，违反即抛 "Infinite loop on byte"（`infiniteLoopError`，693-700 行，`silent` 选项下降级为 console.error）。规则按选项换挡：`pedantic` 用 block.pedantic，`gfm` 用 block.gfm（Lexer 构造函数 52-69 行）。

### 5.3 行内解析与性能工程（Lexer.ts / rules.ts / helpers.ts）

行内扫描是 marked 近两年迭代最重的部分，多处针对 ReDoS 和二次复杂度做硬优化，且带注释说明动机：

- **遮罩层（masking）**：`#maskSrc()`（639-691 行）把链接、代码 span、原始 HTML、转义字符替换成等长的 `'[aaa...]'` 占位串，让 emStrong/del 的正则"看不见"其中的 `*_~` 分隔符；等长替换是因为 emStrong 按位置对齐 src 与 maskedSrc（注释："Every mask must keep the length it replaces"）。无 `*_~` 的文本段直接跳过构建 mask（`#inlineTokensInner` 381-407 行）。
- **closer 扫描缓存**：一次扫描 maskedSrc 记下"最后一个可能的 em/strong/del 闭合符位置"，供所有 opener 共享（`beginEmStrongDelCloserScan`，401-406 行）；`*`/`_`/`~` 连续 run 若前方不存在任何 closer，整段 run 一步消费为文本，消除逐字符循环的 O(n²)（474-517 行，注释引用 issue #4099）。
- **link 括号预判**：一次 `src.includes(')')` 预扫（`inlineTokens` 362-379 行），无 `)` 的输入让 `Tokenizer.link` 直接跳过，规避 `'[a](b'.repeat(n)` 类输入的二次回溯。
- **ReDoS 回归测试**：`test/specs/redos/` 有 41 个用例（quadratic_*.cjs 等），且 `npm run test:recheck` 用 recheck 库静态扫描 `rules.ts` 全部正则的指数/多项式回溯风险（`AGENTS.md` 第 86 行）。

### 5.4 marked.use() 扩展系统（Instance.ts）

`use()`（111-295 行）合并四类扩展：(1) `extensions`：带 `name`/`level(block|inline)` 的新 tokenizer，block 级插到规则表最前（unshift），可附 `start` 函数告知 Lexer 截断段落让位给扩展；(2) `renderer`/`tokenizer` 覆写：逐属性包装原方法，返回 `false` 时回落到上一个实现（链式覆写，190-221 行）；(3) `hooks`：preprocess/postprocess/processAllTokens/emStrongMask/passThroughHooks（`src/Hooks.ts` 15-26 行定义两类钩子集合），异步模式下自动 promise 化；(4) `walkTokens`：递归遍历含 table/list/childTokens 声明的深层 token 树（64-100 行），返回 Promise 时由 `Promise.all` 等待——异步语法高亮（如 marked-highlight）正是靠这条通道集成。同名扩展 renderer 也形成"新的先试、false 回落旧的"责任链（128-141 行）。

## 6. 构建与运行

来自 `package.json` scripts 与 `AGENTS.md`：

```bash
npm install                 # 仅装 devDependencies（零运行时依赖）
npm run build               # esbuild 打包 ESM+UMD + tsc/dts-bundle-generator 类型 + man
npm run build:esbuild       # 快速构建（改代码后跑测试前必做，测试针对 lib/ 产物）
npm test                    # 全套：build + docs + specs + unit + umd + cjs + types + lint
npm run test:specs          # 规范符合度测试（commonmark/gfm/new/original/redos）
npm run test:unit           # node --test 跑 test/unit/*.test.js
npm run bench               # 与 commonmark、markdown-it 对比基准
npm run test:recheck        # recheck 静态分析正则回溯风险
```

使用（README）：`npm install marked` 后 `import { marked } from 'marked'; marked.parse('# hi')`；CLI：`marked -o hello.html`（stdin 输入）；浏览器可用 UMD/ESM CDN 产物。

## 7. 测试与工程质量

- **规范符合度**：CommonMark 0.31.2 官方测试集 652 例（`test/specs/commonmark/commonmark.0.31.2.json`，gfm:false 跑）+ GFM 28 例（`test/specs/gfm/gfm.0.29.json`）；runner 会输出完成度表（`test/run-spec-tests.js` 的 outputCompletionTable），即维护方持续跟踪符合率而非宣称 100%。
- **自建回归**：`test/specs/new/` 250 对 .md/.html 用例（支持 YAML front-matter 设定 gfm/pedantic/renderExact 等选项），覆盖大量列表/tab/懒续行边角；`test/specs/original/` 22 例是 markdown.pl 时代遗产（pedantic 模式跑）。
- **单元测试**：6 个文件约 229 个 `it()` 用例（test/unit/：Lexer 74、marked 57、Hooks 33、bin 33、Parser 23、instance 9），基于内置 node:test，无第三方断言库。
- **工程化程度**：CI 三 Node 版本矩阵（20/lts/latest，`tests.yml`）；类型测试 tsc + attw；UMD/CJS 兼容冒烟测试；ESLint；正则回溯静态检查；基准回归。对一个 ~4,400 行的解析器，这套测试纵深在同类项目中属于顶级水平。

## 8. 社区与维护状态

- **发版节奏**：semantic-release 自动化，Conventional Commits 驱动。近 3.5 个月发 12 个版本（v18.0.8 2026-08-04 → v18.2.0 2026-11-09，gh api releases），平均 1-3 周一版，patch 为主。
- **贡献者**：约 297 名（gh api contributors 分页 last=298，含匿名），社区组织 markedjs 治理（README 版权 2018+ MarkedJS）。
- **活跃度**：最近 push 2026-11-16（前一天）；开放 issue 31 个、fork 3,834。仓库维护响应快、积压小。
- **版本号**：已到 v18.x——历史上 marked 因严格语义化版本曾多次 major（1.0 于 2020 年发布，此后小破坏即升 major），发版频繁但单次破坏面小（此结论基于 release tag 序列推断，具体 major 事件未逐条核实）。
- **依赖面**：npm 周下载量长期千万级（README 徽章位，具体数字未核实）。

## 9. 亮点与风险

**亮点**

1. 速度与体积工程做到极致：零运行时依赖、esbuild 单包产物、热路径系统性去二次复杂度（masking、closer 缓存、括号预判，`src/Lexer.ts`），并配基准回归与 ReDoS 静态检查。
2. 扩展系统设计成熟：tokenizer/renderer/hook/walkTokens 四层扩展点全部支持责任链式覆写与 async，marked-highlight、marked-gfm-heading-id 等官方扩展生态即构建于此（`src/Instance.ts`）。
3. 测试纵深罕见：CommonMark 官方 652 例 + 250 自建回归 + 41 ReDoS 用例 + 229 单测 + 类型/UMD/ lint 多重验证（`test/`）。
4. 工程治理文档化：`AGENTS.md` 把"速度优先、零依赖、不 sanitize、扩展优先"写成可执行的优先级制度（L0-L2/RR/NFS/NFU），对人类与 AI 贡献者同样适用。
5. 文档站自举（docs/ 用 marked 自身构建渲染），dogfooding 到位。

**风险/局限**

1. 不做输出 sanitize 是使用方最大踩坑点：v8.0.0 起内置选项已移除（`docs/USING_ADVANCED.md`），直接 innerHTML 输出 marked 结果而不接 DOMPurify 存在真实 XSS 面，责任完全在用户。
2. 规范符合度是"接近但非 100%"：项目哲学明确允许为速度牺牲边角用例（`AGENTS.md`），对比 markdown-it/CommonMark 参考实现，个别 CommonMark 边角行为存在已知偏差（完成度表随版本变化，未逐一核实具体偏差项）。
3. 输出为字符串而非 AST：核心 API 面向 HTML 字符串拼接（`src/Renderer.ts`），做编辑器双向绑定、语法高亮定位等需要 AST 的场景不如 unified/remark 体系顺手。
4. 版本号膨胀（v18.x）：严格 semver 下 major 频繁，跟随升级有一定心智成本。

## 10. 总结评价

marked 是 Markdown 解析器里"性能极简主义"的代表：四个类、一张正则规则表、零依赖，就把 CommonMark/GFM 主流语法做到了工业可用，并用 900+ 规范/回归用例、ReDoS 静态检查和自动化发版把一个 4,400 行的小内核维护出了大项目的工程水准。它的架构演进（尤其行内解析的 masking 与 closer 缓存优化）对任何做文本解析/正则引擎的人都是极佳的精读材料；对使用者而言，它适合"Markdown → HTML 字符串、要快、要稳、要轻"的场景，但必须自行配套输出净化；如果需要 AST 操作或 100% CommonMark 边角符合度，则应考虑 markdown-it 或 unified/remark。
