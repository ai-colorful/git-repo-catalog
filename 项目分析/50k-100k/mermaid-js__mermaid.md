# mermaid-js/mermaid 项目分析

> 星数 90827（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：用 Markdown 风格文本描述并渲染各类图表的 JavaScript DSL 库，是"文本即图"（diagrams-as-code）赛道的事实标准，GitHub 原生支持渲染。

| 项目 | 内容 |
| --- | --- |
| 名称 | mermaid-js/mermaid |
| 星数 | 90933（gh api 实测 2026-10-31，快照 90827） |
| 语言 | TypeScript（pnpm monorepo） |
| 许可证 | MIT（`LICENSE`、package.json、gh api 一致） |
| 创建时间 | 2014-11-01（gh api） |
| 最近 push | 2026-10-30（分析前一天，极活跃） |
| 最新版本 | mermaid@12.2.0（2026-10-13 发布） |
| 默认分支 | develop |
| 贡献者 | 约 366 人（contributors 分页推断） |

## 2. 项目是什么，解决什么问题

Mermaid 是一个基于 JavaScript 的图表绘制工具：用户用受 Markdown 启发的文本语法定义流程图、时序图、类图、甘特图、Git 图等，库负责解析文本并渲染成 SVG（`README.md` About 章节）。作者把它定位为解决 "Doc-Rot"（文档腐化）问题——图表若以图片形式存在会很快过时，以文本形式存在则可进版本库、可 diff、可 review（`README.md`）。

核心价值在生态位：GitHub 自 2022 年起在 Markdown 代码块中直接渲染 mermaid（`README.md` 引 github.blog 链接），加上 mermaid.live 在线编辑器、mermaid-cli、各类 IDE 插件，使它成为文档内嵌图表的默认选择。对非程序员，Live Editor 提供了零门槛入口；对工程师，它可以嵌入生产脚本自动生成图。

当前版本（12.x）支持 30 余种图表类型：flowchart、sequence、class、state、ER、gantt、pie、gitGraph、mindmap、timeline、sankey、xychart、packet、radar、venn、wardley、cynefin、ishikawa、treemap、railroad 等（`packages/mermaid/src/diagrams/` 下 36+ 个子目录为证）。

安全是其文档中反复强调的一环：图表常渲染不可信用户输入，标准 HTML 消毒会破坏图表语法，因此项目自研消毒流程并提供 sandboxed iframe 渲染模式（`README.md` Security 章节、`src/packages/mermaid/src/mermaidAPI.ts` 中 DOMPurify 调用）。

## 3. 技术栈与依赖

- 语言/运行时：TypeScript（严格 ESM），Node >= 22.12，pnpm@10.30.3 workspace（根 `package.json`）。
- 运行时关键依赖（`packages/mermaid/package.json`）：
  - 布局/绘制：`dagre-d3-es`（DAG 层次布局）、`elkjs`（ELK 布局，约 1.6MB，懒加载）、`cytoscape` + cose-bilkent/fcose（力导向）、`d3` v7、`roughjs`（手绘风 look）、`d3-sankey`、`@upsetjs/venn.js`。
  - 解析：`jison` 0.4.18（构建期，19 个文法）、`langium` ~4.4 + `chevrotain` ~13.2（新 DSL 独立包 `@mermaid-js/parser`）。
  - 安全：`dompurify`（SVG 消毒）、`@braintree/sanitize-url`、stylis（CSS 处理）。
  - 其他：`katex`（数学公式）、`marked`（Markdown 标签）、`dayjs`（甘特日期）、`khroma`（颜色）、`ts-dedent`、`es-toolkit`。
- 构建：esbuild + 自研插件（`.esbuild/build.ts`、`jisonPlugin.ts`、`jsonSchemaPlugin.ts`）；类型由 JSON Schema 生成（`scripts/create-types-from-json-schema.mts`）。
- 质量工具：Vitest 3（单测）、Playwright + Applitools/Argos（视觉回归）、ESLint 9 + Prettier + cspell、changesets（发版）、Husky。
- 文档：VitePress + TypeDoc（`packages/mermaid/src/docs/`）。

## 4. 目录结构与代码组织

pnpm monorepo，`packages/` 下 9 个包（各 `package.json`）：

| 目录 | 职责 |
| --- | --- |
| `packages/mermaid/` | 核心库 mermaid（810 个 ts 文件，含 jison/langium 双解析器与全部渲染） |
| `packages/parser/` | `@mermaid-js/parser`：Langium 编写的新一代 DSL 文法（pie、packet、radar、gitGraph、treemap、wardley 等 16 个语言目录） |
| `packages/tiny/` | 精简构建 mermaid.tiny.js（剔除 ELK/思维图等大特性） |
| `packages/mermaid-layout-elk/` | 独立分发的 ELK 布局引擎（给 tiny 构建外挂） |
| `packages/mermaid-layout-tidy-tree/` | tidy-tree 布局引擎 |
| `packages/mermaid-zenuml/` | ZenUML 时序图集成插件 |
| `packages/mermaid-example-diagram/` | 外部自定义图表的官方示例 |
| `packages/mermaid-local-editor/` | 本地编辑器静态页 |
| `packages/examples/` | 示例图集 |

核心包 `packages/mermaid/src/` 内部：

| 目录 | 职责 |
| --- | --- |
| `diagram-api/` | 图表类型注册表、类型探测（detectType）、frontmatter/指令解析 |
| `diagrams/<type>/` | 每种图一个目录，内含 `*Db.ts`（数据）、`*Detector.ts`（探测）、`parser/`（jison 文法或 langium 适配）、`renderer` 实现 |
| `rendering-util/` | 通用渲染层：布局算法注册（dagre/elk/swimlanes/cose-bilkent 等）与 113 个形状模块（`rendering-elements/shapes/`） |
| `themes/`、`config.ts`、`schemas/` | 主题、全局配置、配置 JSON Schema |
| `mermaid.ts`、`mermaidAPI.ts`、`Diagram.ts` | 对外 API 与渲染主管线 |

顶层还有 `e2e/`（Playwright，按 pr-check/nightly-check/visual-check 分层目录）、`docs/`（由 `packages/mermaid/src/docs` 生成的镜像）、`scripts/`、`tests/`（webpack 集成、release 验证）、25 个 `.github/workflows/`。组织方式是"核心大单体 + 周边小包"：主体逻辑都在 mermaid 包里，新 DSL 与可插拔布局被逐步拆出独立包。

## 5. 核心架构与关键实现

### 5.1 图表类型注册与探测机制

注册表是两个纯对象映射。`registerDiagram` 把 `{db, parser, renderer, styles, init}` 五件套登记进 `diagrams`，同时登记文本探测器并注入公共工具（`packages/mermaid/src/diagram-api/diagramAPI.ts`）：

```ts
const diagrams: Record<string, DiagramDefinition> = {};
export const registerDiagram = (id, diagram, detector?) => {
  diagrams[id] = diagram;
  if (detector) addDetector(id, detector);
  addStylesForDiagram(id, diagram.styles);
  diagram.injectUtils?.(log, setLogLevel, getConfig, sanitizeText, ...);
};
```

文本类型探测：先剥掉 YAML frontmatter 和 `%%{...}%%` 指令，再按注册顺序逐个跑 detector 正则，首个命中者胜出（`packages/mermaid/src/diagram-api/detectType.ts`）。全部内置图在 `addDiagrams()` 中以懒加载形式注册（`packages/mermaid/src/diagram-api/diagram-orchestration.ts`）：

```ts
registerLazyLoadedDiagrams(agentflow, c4, kanban, classDiagramV2, er, gantt,
  info, pie, requirement, sequence, swimlanes, flowchartV2, timeline, git,
  stateV2, journey, quadrantChart, sankey, packet, xychart, block, ...);
```

每个 detector 文件自带 loader，例如 pie：`/^\s*pie/` 匹配后 `await import('./pieDiagram.js')` 才真正加载实现（`packages/mermaid/src/diagrams/pie/pieDetector.ts`）——36 种图不会全部进首屏 bundle。该机制同时开放给第三方：`diagramAPI.ts` 顶部 re-export `log/getConfig/sanitizeText/getCommonDb` 等，配合 `ExternalDiagramDefinition` 即可外挂图表（`packages/mermaid-example-diagram/` 是官方示范）。

### 5.2 DSL 解析：Jison 与 Langium 双轨

两代解析器并存（本仓库快照实测）：

- 旧轨 Jison：19 个 `.jison` 文法（flow、sequence、class、state、er、gantt、c4、block、sankey、timeline、mindmap、kanban、quadrant、requirement、venn、xychart、journey、ishikawa、agentflow），位于各 `diagrams/*/parser/*.jison`。构建期经 esbuild 插件即时转换、不落盘生成物（`.esbuild/jisonPlugin.ts` 调用 `.build/jisonTransformer.js` 的 `transformJison`）。Jison 解析器实例通过 `parser.parser.yy = db` 挂接数据层（`packages/mermaid/src/Diagram.ts` 45-48 行）。
- 新轨 Langium：独立 npm 包 `@mermaid-js/parser`（`packages/parser/`），用 langium ~4.4 声明文法，运行时基于 chevrotain。已覆盖 pie、info、packet、radar、gitGraph、treemap、wardley、cynefin、eventmodeling、railroad（含 abnf/ebnf/peg 方言）、architecture 等 16 个语言目录（`packages/parser/src/language/`）。使用方拿到 AST 后填充 db，例如 pie：

```ts
export const parser: ParserDefinition = {
  parse: async (input: string): Promise<void> => {
    const ast: Pie = await parse('pie', input);
    populateDb(ast, db);
  },
```

（`packages/mermaid/src/diagrams/pie/pieParser.ts`）

入口 `Diagram.fromText()` 把两轨串起来：detectType → 命中则查注册表，未注册则调 loader 懒加载并 registerDiagram → 清空 db、执行 init → `encodeEntities` 后交给对应 parser（`packages/mermaid/src/Diagram.ts` 19-85 行）。此处还实现了 `maxTextSize` 上限与"保留注释解析"（供报告源码位置的图使用）。

### 5.3 渲染管线

`mermaid.render()` → `mermaidAPI.render()` → `renderDiagram()`（`packages/mermaid/src/mermaidAPI.ts` 477-682 行）：创建临时 div → 解析（解析失败且未禁止时改渲染 error 图并把真实错误信息塞给 db，558-589 行）→ 生成主题/用户 CSS 插入 `<style>` → 调 `diag.renderer.draw(text, id, version, diag)` 画 SVG → 序列化并 `cleanUpSvgCode` → 非 loose 安全级别下用 DOMPurify 消毒、沙箱模式下套 iframe（636-661 行）→ 返回 `{ diagramType, svg, bindFunctions }`。

布局算法本身也是注册表 + 懒加载，且支持 frontmatter 里 `layout:` 指定（`packages/mermaid/src/rendering-util/render.ts`）：

```ts
registerLayoutLoaders([
  { name: 'dagre', loader: async () => await import('./layout-algorithms/dagre/index.js') },
  { name: 'swimlane', loader: async () => await import('./layout-algorithms/swimlanes/index.js') },
  ...(injected.includeLargeFeatures ? [{ name: 'cose-bilkent', ...}, ...elkLayoutLoaders()] : []),
]);
```

细节见功力：layout 注册表用 `Object.create(null)` 防原型链注入（`layout` 可从 frontmatter 设置，`__proto__` 之类的键不能借道通过）；elkjs 约 1.6MB，仅在完整构建中保留，tiny 构建回退 dagre。通用形状库在 `rendering-util/rendering-elements/shapes/`（113 个模块），供各图表复用。

## 6. 构建与运行

来自根 `package.json` scripts 与 CONTRIBUTING（未运行验证，仅静态摘录）：

```bash
pnpm install            # pnpm@10.30.3, node>=22.12
pnpm build              # esbuild 产物 + JSON Schema 生成 TS 类型（build:esbuild + build:types）
pnpm dev                # esbuild 开发服务器
pnpm test               # lint(eslint+prettier+cspell+jison lint) + vitest run
pnpm ci                 # 仅 vitest run
pnpm e2e                # playwright test（需 .env）
pnpm lint               # eslint --quiet --cache + lint:jison + prettier --check
pnpm coverage           # vitest 覆盖率 + 覆盖率构建 + e2e 覆盖率合并
pnpm build:mermaid:full # 本地起 dist 编辑器（含 dompurify vendor 拷贝）
```

发版走 changesets：`pnpm changeset:version` / `changeset:publish`，9 个包同版本联动（根 `package.json` 50-51 行；release.yml workflow）。

## 7. 测试与工程质量

- 单测：318 个 `*.spec.ts`（mermaid 包内含大量 per-diagram parser spec，如 `diagrams/flowchart/parser/flow-*.spec.js`）+ packages/parser 16 个 `*.test.ts`，框架 Vitest 3，workspace 含 docs 站点的 vite 配置（`vitest.workspace.js`）。
- E2E/视觉回归：Playwright 26 个 spec（`e2e/`），fixture 按 pr-check / nightly-check / visual-check 三层分目录，PR 只跑快层、夜间全量；接 Applitools（发版基线）与 Argos（PR 视觉对比），README 徽章可证。
- 覆盖率：v8 coverage + Playwright/nyc + monocart 合并上报 Codecov（`scripts/coverage.ts`）。
- CI/安全：25 个 workflow——build、test、lint、e2e、nightly-e2e、codeql、OpenSSF scorecard、dependency-review、validate-lockfile 等；pnpm overrides 钉住 dompurify/js-yaml 版本并 patch fastdom/roughjs（根 `package.json` 184-206 行）。
- 观察：配置类型从 JSON Schema 单源生成并带 `types:verify-config` 校验；文档与 README 同步有专门的 `check-readme-in-sync.yml`；工程化程度在本体量项目中属第一梯队。

## 8. 社区与维护状态

- 活跃度：gh api 显示最近 push 为 2026-10-30（分析前一天），2022 个 open issue，638 watchers；最新 release 12.2.0 于 2026-10-13，此前 12.1.0 于 2026-10-02，约每月 1-2 个 minor/patch 节奏（releases API）。
- 贡献者：contributors 分页共 366 页（per_page=1 推断约 366 人）；CHANGELOG（`packages/mermaid/CHANGELOG.md`）12.2.0 一条就含 10+ 位不同作者，社区贡献占主导，核心维护者（filipsajdak、ashishjain0512 等）主导评审。
- 生态：姊妹项目 mermaid-cli、mermaid-live-editor（`README.md` Related projects）；官方书、教程、集成清单见 docs/ecosystem。
- 资助：FUNDING.json + Discord 社区；有 issue-triage、unlock-reopened-issues 等自动化运营 workflow。

## 9. 亮点与风险

亮点：

1. 插件式图表注册架构清晰：detector 正则 + 懒加载 loader + 五件套 Definition，第三方扩展有官方示例包（`diagram-api/`、`packages/mermaid-example-diagram/`）。
2. 渐进式解析器现代化：新 DSL 走 Langium 并独立发 `@mermaid-js/parser` 包，旧 Jison 文法由 esbuild 插件构建期消化，无生成物入库。
3. 安全工程扎实：DOMPurify 消毒、sandbox iframe、sanitize-url、防原型链注入的 layout 注册表、CodeQL + Scorecard 全套（`mermaidAPI.ts`、`render.ts`）。
4. 性能意识：36 种图全部懒加载、elkjs 1.6MB 按需 chunk、tiny 精简构建、`maxTextSize` 防御超大输入。
5. 测试金字塔完整：318 个单测 + 分层视觉回归（pr/nightly/visual 三档），配合覆盖率合并上报。

风险/局限：

1. 核心包体量大：`packages/mermaid/src` 约 16.6 万行（含文法），36+ 图表目录各自维护 db/parser/renderer，风格与抽象层次不完全一致，新贡献者上手曲线陡。
2. Jison 技术债仍在：jison 0.4.18 是多年未大版本更新的老解析器生成器，19 个文法尚未迁完 Langium，双轨将长期共存。
3. 全局可变配置模式复杂：渲染期间"持有 config 作用域"的注释自述了竞态隐患（`mermaidAPI.ts` 684-687 行），跨图状态（db 单例）依赖严格的 clear 纪律。
4. 依赖面广（d3/cytoscape/elkjs/katex/marked/dayjs…），完整构建体积与供应链审核成本相应增加（已有 validate-lockfile、dependency-review 对冲）。

## 10. 总结评价

Mermaid 是"文本生成图表"赛道的定义者与事实标准：90k+ 星背后是 GitHub 原生渲染带来的不可替代生态位、十年磨出的 30 余种图表、以及一套高度工程化的 monorepo（懒加载注册表 + 双解析器 + 三档视觉回归 + 完整安全链）。它最好的读法是从 `Diagram.fromText()` 入口顺藤摸瓜——注册（diagramAPI）、探测（detectType）、解析（jison/langium 双轨）、渲染（renderer.draw + 布局注册表）四段式架构在 `packages/mermaid/src/` 中一目了然。适合：想给自己的产品加图表能力的工程师（直接用）、研究 DSL 设计与插件注册模式的学习者（读 5.1/5.2 两条线）、想贡献新图表类型的开发者（照 `mermaid-example-diagram` 起步）。不适合追求像素级排版控制或服务端静图批量渲染的场景——后者建议搭配 mermaid-cli。
