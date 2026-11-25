# prettier/prettier 项目分析

> 星数 52379（快照 2026-10-24）｜主语言 JavaScript｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

Prettier 是"有主见的代码格式化器"（opinionated code formatter）：把代码解析成 AST 后按自身规则重新打印，只保留极少量配置项，用"统一风格"终结团队间的格式之争。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | prettier/prettier | package.json `repository` |
| 版本 | 3.10.0-dev（最新 release 3.9.10，2026-10-10） | package.json；GitHub releases API |
| 星数 | 52379（快照）/ 52403（2026-11-04 实测） | 任务参数；`gh api repos/prettier/prettier` |
| 主语言 | JavaScript（纯 ESM，`"type": "module"`） | package.json |
| 许可证 | MIT | package.json、LICENSE（MIT 文本，版权 James Long and contributors） |
| 最近 push | 2026-11-03 | `gh api repos/prettier/prettier` |
| 默认分支 | main；Node >= 22；包管理 yarn 4.18.1 | `gh api`；package.json `engines`/`packageManager` |
| 规模 | src/ 下 551 个 JS/TS 文件、约 5.3 万行 | 源码统计 |
| 官网 | https://prettier.io | package.json `homepage` |

## 2. 项目是什么，解决什么问题

Prettier 解决的是"代码风格不一致"与"风格讨论无休止"两个问题。与 ESLint 类工具"检查 + 部分自动修复"的思路不同，Prettier 走"解析 → 全量重打印"路线：用各语言现成的解析器把源码解析成 AST，再由自己的 printer 从零生成输出（README "Intro"；docs/technical-details.md）。由于输出完全由 AST 决定，同一份代码在任何机器上格式化结果都一致，diff 噪音随之消失。

"Opinionated"体现在选项策略上：Prettier 刻意只保留极少数选项（如 printWidth、tabWidth、semi），CONTRIBUTING.md 有专门的 "No New Options" 一节，明确拒绝为个别偏好新增选项，这是它能维持"一次配置、全队统一"定位的关键设计决策。

语言覆盖面是其护城河：开箱支持 JavaScript/TypeScript/Flow/JSX/JSON、CSS/SCSS/Less、HTML/Vue/Angular、GraphQL、Markdown、YAML、Handlebars（README 支持列表；src/language-* 共 8 个语言目录），并对外开放插件 API 让社区补齐其他语言（docs/plugins.md）。Markdown/YAML 这类"非代码"文件的格式化也让它从编辑器插件扩展为全仓库工具。

内嵌语言格式化（multiparser）是另一个特色：HTML 里的 `<script>`/`<style>`、JS 模板字符串里的 CSS/GraphQL/HTML/Markdown、Markdown 代码块等都能用对应语言的 parser 格式化后再拼回外层文档（src/main/multiparser.js、src/language-js/embed/index.js）。

它已成为 JS 生态事实标准：eslint-config-prettier 专门负责关掉 ESLint 冲突规则（docs/integrating-with-linters.md）， editors/CI/预提交钩子集成文档齐全（docs/ 目录 20+ 篇）。

## 3. 技术栈与依赖

- **语言与运行时**：纯 JavaScript ESM（部分 `.evaluate.js` 构建期求值、JSDoc 类型标注替代 TS 源码），Node >= 22（package.json）。
- **零"格式化逻辑"外购，解析器全外购**——按语言分派（package.json dependencies）：
  - JS/TS：@babel/parser、espree、acorn(+jsx)、meriyah、oxc-parser、@typescript-eslint/typescript-estree、flow-parser、hermes-parser、yuku（src/language-js/parse/ 下 10 个 parser 包装）
  - CSS：postcss + postcss-scss + postcss-less + postcss-values-parser
  - HTML/Vue/Angular：angular-html-parser、@angular/compiler；Handlebars：@glimmer/syntax
  - Markdown：remark-parse/unified + mdast-util-gfm/math + micromark 系列
  - YAML：yaml + yaml-unist-parser；GraphQL：graphql 17；TOML（front-matter）：smol-toml
- **文档 IR**：自研 doc 抽象（fork 自 recast 的 printer，算法换成 Wadler "A prettier printer"，见 docs/technical-details.md、commands.md）。
- **测试/工具链**：Jest 30 + 快照、c8 覆盖率、esbuild 打包、ESLint 10、cspell、knip、Puppeteer（浏览器测试）、tinybench（benchmarks/）。devDependencies 里有 `"prettier": "3.9.10"`——用自己格式化自己（`lint:prettier` 脚本）。

## 4. 目录结构与代码组织

顶层（tarball 根，即仓库根）：

| 目录/文件 | 职责 |
|---|---|
| `src/main/` | 核心流水线：parse → AST→doc → doc→string，选项归一化、注释附着、插件加载 |
| `src/document/` | doc 中间表示：builders（group/line/indent…）与 Wadler 打印器 |
| `src/language-css/` `language-graphql/` `language-handlebars/` `language-html/` `language-js/` `language-json/` `language-markdown/` `language-yaml/` | 每语言一个目录：parser 包装 + printer + options + languages 声明 |
| `src/plugins/` | 13 个"解析器族"聚合文件，供 standalone 产物按需打包（如 babel.js=babel+json parsers） |
| `src/common/` `src/utilities/` `src/universal/` | AstPath、错误类型、行尾处理；通用工具；浏览器/Node 双端实现 |
| `src/cli/` `src/experimental-cli/` | 现行 CLI 与实验性 CLI（worker 进程模型，experiments 目录含 worker.js） |
| `bin/` `scripts/` | `prettier.cjs` 入口；esbuild 构建脚本、generate-builtin-plugins、changelog 生成等 |
| `tests/` | format（1489 个 format.test.js）、integration、unit、dts、config（测试基建） |
| `docs/` | 用户文档（options/cli/plugins/technical-details 等 23 篇） |
| `website/` | Docusaurus 官网 + Playground |
| `changelog_unreleased/` | 按语言分目录的未发布 changelog 碎片（PR 必须带，有 lint 脚本校验） |
| `types/` `vendors/` `benchmarks/` | 类型定义、内置第三方产物、基准测试 |

组织方式评述：单仓单包（非 monorepo，`packages/` 仅放网站附属包），但内部高度插件化——每个语言目录都长成"潜在插件"的形状，再由 `scripts/generate-builtin-plugins.js` 生成惰性加载注册表（src/main/plugins/builtin-plugins/production-plugins.js 头部注明 "Generated file, do NOT edit"）。核心（main+document）与语言实现严格分离，新增语言即新增一个 language-* 目录 + plugins 聚合文件。

## 5. 核心架构与关键实现

### 5.1 三段式流水线（src/main/core.js）

`formatWithCursor` 是总入口：选项归一化（normalize-format-options.js）→ BOM/行尾归一（core.js `normalizeInputAndOptions`）→ `coreFormat`。核心三步在 `coreFormat` 中一目了然：

```js
// src/main/core.js
const { ast, text } = await parseText(originalText, opts);   // 1. 文本→AST
let doc = await printAstToDoc(ast, opts, addAlignmentSize);  // 2. AST→doc IR
const result = printDocToStringWithoutNormalizeOptions(doc, opts); // 3. doc→字符串
```

光标保持（cursorOffset）用 diff 实现：把光标当字符插入旧文本区域，与格式化后区域做 diff 定位新位置（core.js 第 148-175 行注释详述）。`formatRange` 支持只格式化选区，通过把选区当全文格式化再拼回（core.js formatRange）。

### 5.2 AST→doc 递归打印与打印器契约（src/main/ast-to-doc.js）

`printAstToDoc` 沿 AstPath 递归下降，调用 printer 的 `print()`，子节点通过回调 `mainPrint` 再次进入循环并按节点缓存结果（`cache = new Map()`，ast-to-doc.js 第 36、90-100 行）。打印前先 `attachComments`（注释附着）与 `preprocess`（语言级预处理）。对外暴露的打印器契约（见 src/language-js/printers.js 的 estree 对象）包括：`print`、`embed`、`preprocess`、`handleComments`、`canAttachComment`、`willPrintOwnComments`、`printComment`、`getVisitorKeys`、`hasPrettierIgnore`、`massageAstNode`、`insertPragma`——插件照此实现即可接入。代码库正迁移到"不改 AST"模式：`experimental_avoidAstMutation: true`（printers.js）。

### 5.3 doc IR 与 Wadler 打印器（src/document/）

builders 提供 16 种原语：group、indent、align、line（软换行）、fill、if-break、indent-if-break、join、label、line-suffix、trim、cursor 等（src/document/builders/）。打印器 src/document/printer/printer.js 是经典 Wadler 算法：命令栈 + 双模式（MODE_BREAK/MODE_FLAT），核心是 `fits()`——flat 模式下"测量"后续输出是否超宽，不满足则在 group 边界换行（printer.js 第 53 行起）。`propagateBreaks` 预先把 hardline 的换行意图沿树传播（document/utilities/）。docs/technical-details.md 明确说明该 printer 是 recast fork + Wadler 算法。

### 5.4 注释附着（src/main/comments/attach.js）

Prettier 最难的工程之一。`decorateComment` 对按位置排序的子节点做二分查找，为每条注释标定 `precedingNode / enclosingNode / followingNode`，再递归下钻到完全包含注释的子节点（attach.js 第 20-70 行）；随后分三类挂接：leading / trailing / dangling（comments/utilities.js），语言 printer 可用 `handleComments` 钩子微调（如 JSX 需要括号包裹的场景）。打印阶段 `printComments`（comments/print.js，290 行）拼回，最后 `ensureAllCommentsPrinted` 断言无注释丢失（ast-to-doc.js）。

### 5.5 内嵌语言 multiparser（src/main/multiparser.js）

`printEmbeddedLanguages` 先序遍历外层 AST（按 printer.embed 的 getVisitorKeys），对每个节点调用 `embed()`；返回异步打印函数的先收集，之后统一执行。嵌入文本经 `textToDoc` 换用目标 parser 走完整"parse→print"流程，并带上 `parentParser` 记录，结果 `stripTrailingHardline` 后存入 `embeds` Map，主打印时替换原节点输出（multiparser.js 第 40-54、108-132 行）。JS 侧实现见 src/language-js/embed/index.js：模板字符串按 CSS/GraphQL/HTML/Angular/Markdown 的 `isEmbed*` 谓词分派；HTML 侧见 src/language-html/embed.js（script/style 内容递归格式化）。

### 5.6 插件机制（src/main/plugins/）

外部插件：`load-plugin.js` 支持三种导入形态——URL、绝对路径、从 cwd 向上查 node_modules（importFromDirectory），带 Map 缓存（load-plugin.js 第 34-55 行）。内置插件：`generate-builtin-plugins.js` 生成 production/development 两份注册表，`toLazyLoadPlugins` 做惰性 `load: () => import(...)`（builtin-plugins/production-plugins.js）。插件对象契约：`{ languages, parsers, printers, options, defaultOptions }`（docs/plugins.md；main/plugins/ 各 load 文件）。

## 6. 构建与运行

来自 package.json scripts 与 CONTRIBUTING.md（未实际运行，按规范只静态阅读）：

```bash
yarn install            # yarn 4，需 Node >= 22
yarn build              # node ./scripts/build/build.js，esbuild 产 dist/
yarn test               # jest（开发态）
yarn test:production    # 对 dist 产物测试（含 INSTALL_PACKAGE 安装态）
yarn test:production:browser:chrome  # Puppeteer 驱动的浏览器真跑
yarn lint               # eslint + tsc + prettier --check + cspell + knip 等并行
yarn debug <file>       # 用源码态 prettier 格式化单个文件调试
node bin/prettier.cjs   # CLI 入口
```

发布约束：`prepublishOnly` 强制"只能从 dist/ 发布"（package.json scripts）。

## 7. 测试与工程质量

- **规模**：tests/format 下 1489 个 format.test.js、共 8010 个文件；全仓 1529 个 .snap 快照。每个测试目录一个 format.test.js 调 `runFormatTest`（CONTRIBUTING.md；tests/config/format-test/）。
- **风格**：目录即用例的快照测试 + 金样本（input/expected 成对）；自定义 harness 额外断言幂等性（test-second-format.js：格式化两次结果必须一致）、AST 等价（test-ast-compare.js）、BOM/行尾/占位符替换等。
- **外部标准测试套件**：commonmark/gfm/graphql/json/jsx/less/yaml-test-suite 均为 devDependency（package.json），符合标准文档用例直接回归。
- **测试矩阵**：开发态 / production(dist) / standalone(浏览器 bundle) / 真浏览器(chrome/firefox × esm/umd)，由环境变量切换（jest.config.js、package.json scripts）。
- **工程质量观察**：覆盖率 c8（package.json `c8` 段）；lint 组合了 eslint(10)+tsc+cspell+knip(死代码)+自研 format-test lint；changelog 有专门 lint 脚本；CI 工作流约 20 个（.github/workflows/：lint、dev-test、prod-test、bundler-friendly、check-sizes、release-script-test 等）。devDependencies 依赖 prettier 自身做自格式化，是强自洽信号。

## 8. 社区与维护状态

- **活跃度**：最近 push 2026-11-03（分析前一天），open issues 1483、forks 5025（gh api）。tarball 无 git 历史，据 CHANGELOG.md 推断发版节奏稳定且密集（3.9.8→3.9.10 连续补丁，含性能回退快速修复如 #20266）。
- **发版机制**：changelog_unreleased/ 按语言/主题分目录收集碎片，`yarn generate-changelog` 汇总成 CHANGELOG.md（scripts/generate-changelog.js）；latest release 3.9.10（2026-10-10）。
- **贡献者结构**：CHANGELOG 近期条目高频出现 fisker、sosukesuzuki 等核心维护者；团队治理与资助入口在 FUNDING.json / README（细节未逐一确认）。
- **Issue 治理**：.github/ISSUE_TEMPLATE/ 区分 formatting 与 integration 双模板，另 lock/no-response/support 等机器人工作流，属成熟开源治理形态。

## 9. 亮点与风险

**亮点**
1. 打印器架构教科书级：解析、打印策略（语言插件）、排版算法（doc IR + Wadler）三层彻底解耦，新增语言不动核心（src/main、src/document、src/language-* 边界清晰）。
2. 注释附着这一 formatter 最难问题有系统化方案：二分定位 + 三分类 + 语言钩子 + "全部打印否则报错"兜底（src/main/comments/）。
3. 测试体系覆盖真实格式化器最难的三件事：幂等性、AST 等价、跨 parser/standalone/浏览器矩阵（tests/config/format-test/、jest.config.js）。
4. 内置插件全部生成式惰性加载，standalone 产物可按解析器族裁剪体积（scripts/generate-builtin-plugins.js、check-sizes 工作流）。
5. 用"拒绝新选项"的产品纪律换取长期可维护性（CONTRIBUTING "No New Options"）。

**风险/局限**
1. Opinionated 定位即取舍：不满足其风格偏好的团队只能离开或依赖受限选项，无中间态。
2. 升级即重新格式化：小版本间的打印输出变化仍会给大仓库带来 diff 噪音（CHANGELOG 中的行为修复条目即此类来源）。
3. 解析器依赖面很宽（10 个 JS parser + postcss 系 + unified 系等），依赖升级与产物体积是持续成本（renovate.json5、check-sizes 工作流佐证）。
4. Node >= 22 与纯 ESM 源码对下游嵌入式场景和旧运行时不友好（package.json engines；standalone bundle 缓解但未消除）。

## 10. 总结评价

Prettier 是"用一种强观点换全行业一致性"的少数成功案例：技术上把 formatter 拆成 parser/打印策略/Wadler 排版三层，产品上用极简选项 + 无可辩驳的确定性输出终结风格争论。代码库本身是 JS 生态工程质量的天花板样本——分层克制、契约清晰、测试体系针对格式化器的真实难点设计。适合三类人使用或研读：想把统一格式化落地到任意规模仓库的团队（直接用）；想给新语言写格式化器/插件的作者（读 plugins 契约 + document/builders）；想学"中间表示 + 测量式排版"这一经典编译器技巧的工程师（精读 src/document/printer/printer.js 与 src/main/ast-to-doc.js）。
