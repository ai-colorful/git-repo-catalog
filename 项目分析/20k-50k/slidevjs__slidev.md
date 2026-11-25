# slidevjs/slidev 项目分析

> 星数 49179（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：面向开发者的 Markdown 幻灯片工具——用 Markdown + Vue 组件写幻灯片，借助 Vite 实现秒级热更新，可导出 PDF/PNG/PPTX 或直接部署为 Web 应用。

| 项目 | 值 |
| --- | --- |
| 名称 | slidevjs/slidev（npm 包 `@slidev/cli`） |
| 星数 | 49179（快照值） |
| 语言 | TypeScript + Vue 3 |
| 许可证 | MIT（`LICENSE`，Copyright 2020-PRESENT Anthony Fu） |
| 创建时间 | 2021-04-24（gh api） |
| 最近 push | 2026-10-23（gh api），分析时点高度活跃 |
| 最新 release | v53.0.0，2026-09-16（gh api `releases/latest`） |
| 贡献者 | 约 384 人（gh api contributors 分页页码推算，含匿名） |
| 官网 | https://sli.dev（`package.json` homepage；文档在仓库 `docs/` 内 VitePress 构建） |

## 2. 项目是什么，解决什么问题

Slidev（"Slides for Developers"）解决的问题是：开发者做技术分享时，PowerPoint/Keynote 难以承载代码、公式、图表和交互演示，而纯 Markdown 工具（如 Marp）又缺乏交互性和可编程性。Slidev 的答案是"Markdown 为源码 + Vue 为渲染层 + Vite 为开发服务器"（`README.md` Features 一节与 `packages/client/` 全貌）。

它的核心工作方式是：用户在一个 `slides.md` 中用 `---` 分隔符切分幻灯片，每页可写 Markdown、内嵌 Vue 组件（`<Mermaid/>`、`<Tweet/>` 等内置组件见 `packages/client/builtin/`）、代码块自动经 Shiki 高亮并支持 TwoSlash 类型标注与 Magic Move 动画（依赖 `@shikijs/twoslash`、`@shikijs/magic-move`，见 `packages/slidev/package.json`）。`v-click` 指令与 `<v-clicks/>` 组件提供点击步进动画，点击上下文的实现位于 `packages/client/composables/useClicks.ts`（支持 `+1`/`-1` 相对偏移与区间值）。

与竞品（Marp、reveal.js）相比的差异点：一是"幻灯片即 Vue 应用"，可以直接 import 任意 npm 包、写自定义组件、用 UnoCSS 原子类排版（`packages/slidev/node/vite/unocss.ts`）；二是主题即 npm 包，`slidev-theme-*` 可发布共享，CLI 找不到主题时会提示自动安装（`packages/slidev/node/resolver.ts` 的 `createResolver()`）；三是完整的开发者工作流：VSCode 扩展（`packages/vscode/`）、演讲者模式、录屏、画板标注（drauu）、LaTeX（KaTeX）与图表（Mermaid/PlantUML）内置。

近期（v52/v53 时代）的明显方向是拥抱 AI：CLI 提供 `slidev mcp` 子命令启动 MCP 服务器，供 AI Agent 检查/编辑/翻页幻灯片（`packages/slidev/node/cli.ts` 第 545 行附近的 `mcp` 命令与 `packages/slidev/node/mcp/server.ts`），仓库还内置了官方 Agent Skill（`skills/slidev/SKILL.md`），并在 dev server 上暴露 `/__mcp` HTTP 端点（`packages/slidev/node/mcp/server.ts` 头部注释）。

## 3. 技术栈与依赖

- 语言/运行时：TypeScript（严格 ESM，`"type": "module"`），要求 Node >= 22.12（根 `package.json` engines）
- 构建/编排：pnpm 12.15 + pnpm catalog（依赖统一版本管理，`pnpm-workspace.yaml` 的 `catalog:` 协议遍布各 package.json）+ Turborepo（`turbo.json`）；产物打包用 tsdown（`packages/slidev/tsdown.config.ts`）
- 前端框架：Vue 3 + vue-router + @vueuse 全家桶（`packages/client/package.json`）
- 服务端：Vite 7 时代插件体系（`packages/slidev/node/vite/` 下 23 个插件文件），dev server 用 connect 中间件
- Markdown 渲染：unplugin-vue-markdown 把 `.md` 编译为 Vue 组件（`packages/slidev/node/vite/markdown.ts`）；markdown-it 插件栈（markdown-exit、markdown-it-footnote、markdown-it-github-alerts）
- 代码高亮：Shiki（含 twoslash、magic-move）、Monaco Editor（浏览器内编辑器，类型自动加载用 `@typescript/ata`）
- 公式/图表：KaTeX、Mermaid、plantuml-encoder
- 导出：Playwright（无头 Chromium 渲染）、pdf-lib + @lillallol/outline-pdf（PDF 目录书签）、pptxgenjs（PPTX）
- CLI/工程化：yargs（CLI 参数）、fast-glob、chokidar（HMR 监听）、@antfu/ni（主题自动安装）；MCP 官方 SDK（`@modelcontextprotocol/server`）+ zod（工具参数校验）
- 样式：UnoCSS 按需原子类 + @unocss/extractor-mdc（Markdown 中提取类名）

## 4. 目录结构与代码组织

pnpm + Turbo monorepo，共 7 个发布包（根 `pnpm-workspace.yaml` + `packages/`）：

| 目录 | 职责 |
| --- | --- |
| `packages/parser` | `@slidev/parser`：Markdown→幻灯片解析核心（约 1.2k 行），纯逻辑、无 Vue 依赖 |
| `packages/slidev` | `@slidev/cli`：CLI 入口（bin `slidev`）、Node 侧 Vite 插件、导出实现、MCP 服务器（约 12.8k 行） |
| `packages/client` | `@slidev/client`：浏览器端 Vue 应用——布局、内置组件、点击动画、演讲者模式、画板/录屏（约 15k 行） |
| `packages/types` | `@slidev/types`：跨包共享类型定义（约 1.5k 行） |
| `packages/vscode` | VSCode 扩展：语法高亮、侧边预览编辑器、语言客户端（约 2.6k 行） |
| `packages/create-app` / `packages/create-theme` | `npm init slidev` 与主题脚手架 |
| 其他 | `docs/`（VitePress 文档站）、`demo/`（starter/composable-vue/vue-runner 三个演示）、`cypress/`（e2e）、`test/`（集成测试）、`skills/`（官方 Agent Skill）、`scripts/`（zx 发布/演示脚本） |

组织方式评述：典型的 antfu 风格分层——`parser`（纯函数、零 UI 依赖，可被 CLI/VSCode/测试复用）与 `client`/`slidev`（Node/浏览器两端）严格解耦；`fs.ts` 中还有注释明确说明 parser "must stay independent of @slidev/slidev"（`packages/parser/src/fs.ts` 的 `isPathInsideRoots` 注释）。CLI 与浏览器端通过"虚拟模块"契约（`/@slidev/slides`、`/@slidev/layouts` 等，`packages/slidev/node/virtual/`）通信，这是 Vite 生态很干净的 Node↔客户端桥接方式。

## 5. 核心架构与关键实现

### 5.1 Markdown→Slides 解析管线（packages/parser/src/core.ts）

`parse()` 做逐行状态机扫描：遇到 `---` 行切片一页，但有两个关键防御——跳过围栏代码块（代码内的 `---` 不会误切分，按起始反引号数量配对）和跟踪 HTML 注释状态（`<!-- -->` 内的 `---` 不切分）：

```ts
// packages/parser/src/core.ts (parseSync 主循环节选)
if (line.startsWith('---')) {
  slice(i)
  const next = lines[i + 1]
  if (line[3] !== '-' && next?.trim()) {   // frontmatter 头，跳到闭合 ---
    start = i
    for (i += 1; i < lines.length; i++)
      if (lines[i].trimEnd() === '---') break
    contentStart = i + 1
  }
}
else if (line.trimStart().startsWith('```')) { // 代码块整体跳过
  const codeBlockLevel = line.match(RE_LEADING_BACKTICKS)![0]
  let j = i + 1
  for (; j < lines.length; j++) if (lines[j].startsWith(codeBlockLevel)) break
  if (j !== lines.length) i = j
}
```

每页再经 `parseSlide()` 细化：frontmatter 支持 `---` 头部和 ```` ```yaml ```` 代码块两种风格（`matter()`），页尾最后一个 HTML 注释提升为演讲者备注，标题从首个 `#` 标题提取（先剥掉代码块防误判），并做特性检测（katex/monaco/mermaid/tweet/bluesky，`detectFeatures()`）——检测结果直接决定构建期是否引入 KaTeX CSS 等（`packages/slidev/node/virtual/conditional-styles.ts`）。`parse()` 还支持 `SlidevPreparserExtension` 三级钩子（transformRawLines/transformSlide/transformNote），供用户在 `setup/preparser.ts` 中注入自定义语法；多文件 `src:` 引用由 `packages/parser/src/fs.ts` 的 `load()` 递归装载并用 `isPathInsideRoots()` 防路径逃逸。解析结果可用 `stringify/prettify` 无损写回 Markdown（`slidev format` 命令与编辑器回写都依赖它）。

### 5.2 幻灯片→Vue 组件的编译链（packages/slidev/node/vite/）

每页幻灯片被 unplugin-vue-markdown 编译成异步 Vue 组件。`packages/slidev/node/virtual/slides.ts` 动态生成聚合模块：

```ts
// packages/slidev/node/virtual/slides.ts (节选)
statements.push(
  `import { meta as f${no} } from '${VIRTUAL_SLIDE_PREFIX}${no}/frontmatter'`,
  `const load${no} = async () => {`,
  `  try { return componentsCache[${idx}] ??= await import('${VIRTUAL_SLIDE_PREFIX}${no}/md') }`,
  `  catch (e) { console.error('slide failed to load', e); return SlideError }`,
  `}`,
)
return `{ no: ${no}, meta: f${no}, load: load${no}, component: getAsyncComponent(${idx}, load${no}) }`
```

即：每页有独立的 `/@slidev/slides/<no>/md` 与 `/frontmatter` 模块，按页懒加载、单页出错不拖垮整个 deck，HMR 精确到页。Markdown 文本变换分四段执行（pre → preCodeblock → postCodeblock → post，`packages/slidev/node/vite/markdown.ts` + `packages/slidev/node/setups/transformers.ts`），transformer 用 MagicStringStack 做可叠加的源码改写（KaTeX、代码片段引入 `snippet.ts`、拖拽语法、链接处理等都在 `packages/slidev/node/syntax/` 下，每个文件带独立单测）。`packages/slidev/node/vite/loaders.ts` 中的 `slidev:loader` 插件还注册了 dev server 中间件：`GET/POST /@slidev/slides/<no>` 可读写单页内容与备注并回写到磁盘上的 Markdown（配合 MCP 与网页内编辑器实现"改即存源"）。

### 5.3 主题与扩展体系（packages/slidev/node/resolver.ts + options.ts）

主题/插件即 npm 包。`createResolver('theme', officials)`（`packages/slidev/node/resolver.ts`）按优先级解析：本地绝对路径 → `@/`（用户项目根）→ `@slidev/theme-*`/`slidev-theme-*` 本地 node_modules → 提示自动安装（用 @antfu/ni 兼容 npm/yarn/pnpm），并对名称做安全包名校验（`RE_SAFE_PKG_NAME`）防止注入。主题包提供 `layouts/`、`styles/index.{ts,js,css}`、`setup/` 等目录；聚合逻辑在 `packages/slidev/node/options.ts` 的 `getLayouts()`（client 内置布局 → 主题 → 用户项目逐层 glob 覆盖）与 `packages/slidev/node/virtual/conditional-styles.ts`（各 root 的样式以 import glob 叠加）。用户可通过项目根的 `setup/` 目录扩展 transformers、Shiki 选项、KaTeX、UnoCSS、代码运行器、快捷键、Mermaid 渲染器、preparser（`packages/slidev/node/setups/` 的 loadSetups 按名字发现加载）。编译期特性开关通过 `getDefine()` 生成的 `__SLIDEV_FEATURE_*` 常量做死代码消除（`packages/slidev/node/options.ts`）。

### 5.4 导出实现（packages/slidev/node/commands/export.ts，约 690 行）

导出统一走"起本地服务 + 无头 Chromium 渲染打印视图"路线：Playwright 以 1920x1080 视口打开带 `?print=true` 的页面（视口高度按页数放大以触发 iframe 内懒渲染），随后是一长串确定性等待（loading 元素分离、`data-waitfor` 元素可见、所有 frame 加载、Mermaid 渲染容器清空、Monaco aria 容器隐藏），然后按格式分流：PDF 用 `page.pdf()` + pdf-lib 注入 TOC 书签（`makeOutline()`）；PNG 逐页截图；PPTX 把 PNG 塞进 pptxgenjs；`pptx-editable` 将文本以可编辑文本框导出；`withClicks` 会把每个点击态展开为独立页面（点击页数从 DOM `.print-slide-container` 的 id 统计回推，见 `getSlidesIndex()`）。演讲者备注单独导出为 `exportNotes()`（打开 `/presenter/print` 页打印）。

### 5.5 MCP 服务器（packages/slidev/node/mcp/）

`server.ts` 用官方 MCP SDK 注册约 8 个工具（`server.registerTool(...)` 多处调用），覆盖读取 deck 概要、单页读写、插入/删除/移动页、frontmatter 修改，以及（dev server 在跑时）实时翻页导航 `nav.go(page, clicks)`。同一套工具通过 stdio（`slidev mcp` 命令）和 dev server 的 `/__mcp` HTTP 端点双通道暴露，zod 做 schema 校验，并有对应的 `test/mcp.test.ts` 集成测试。

## 6. 构建与运行

来自根 `package.json` scripts 与文档（`docs/`）：

```bash
pnpm install                # pnpm 12，Node >= 22.12
pnpm dev                    # turbo 构建 watch 各包
pnpm demo:dev               # 跑 demo/starter 开发预览（等同最终用户 `npm init slidev` 后 `npm run dev`）
pnpm build                  # turbo run build --filter="./packages/*"（CLI 用 tsdown 产出 dist）
pnpm test                   # vitest（多 project：packages/* + test）
pnpm typecheck              # vue-tsc --noEmit
pnpm lint                   # eslint（@antfu/eslint-config）+ prettier-plugin-slidev
pnpm cy / pnpm cy:fixture   # Cypress e2e（fixture 是一个真实 deck）
pnpm verify                 # build + typecheck + lint + test 全量校验
pnpm docs / docs:build      # VitePress 文档站（netlify.toml 托管）
```

最终用户侧：`npm init slidev` 建项目；`npx slidev dev/build/export/export-notes/format/theme/mcp`。导出 PDF 需 `playwright-chromium`（CLI 依赖里已带）。

## 7. 测试与工程质量

- 单元/集成（Vitest，`vitest.config.ts` 声明 `packages/*` + `test` 双 project）：parser 全套（`packages/parser/src/*.test.ts`、`test/parser.test.ts`）、语法 transformer 逐个覆盖（`packages/slidev/node/syntax/*.test.ts`，约 10 个测试文件含快照）、MCP 集成（`test/mcp.test.ts`）、Mermaid 渲染（`test/mermaid-renderer.test.ts`）、API 快照（`test/api-snapshots.test.ts`）、import 守卫（`test/import-guard.test.ts`，防止包间非法依赖）
- E2E（Cypress，`cypress/e2e/examples/`）：basic、base-path、iframe-layout、presenter-resizer、smoke 五个 spec，跑在真实 dev server 上
- CI（`.github/workflows/`）：`test.yml` 在 ubuntu/windows/macos 三平台跑 build+test（LF 强制、CYPRESS_INSTALL_BINARY=0 提速），另有 typecheck、`smoke.yml`（发布冒烟）、`release.yml`（npm 发包走 `zx scripts/publish.mjs`）、`autofix.yml`（自动 lint fix PR）、`cr.yml`（代码评审辅助）。提交钩子用 simple-git-hooks + lint-staged（根 `package.json`）
- 工程质量观察：严格 ESM 全仓、TypeScript 严格模式 + vue-tsc、pnpm catalog 锁版本、依赖 patch 通过 `patches/` 管理；代码里可见大量防御性细节（路径逃逸检查、包名校验、HMR 缓存失效处理），成熟度高

## 8. 社区与维护状态

- 活跃度：gh api 显示最近 push 为 2026-10-23（分析前一日），open issues 234，forks 2269——对 49k 星项目来说 issue 积压很小，说明响应快
- 发布节奏：tag 列表显示 v52.x 密集补丁（v52.17→v52.20.1）后于 2026-09-16 发 v53.0.0，月均多个版本，语义化版本执行严格（根 `package.json` version 53.0.0 与最新 tag 一致）
- 贡献者结构：约 384 名贡献者（含匿名，gh api 分页推算）；作者 Anthony Fu（antfu）为主理人，README 首屏即其赞助计划入口，核心维护者来自 Vue/Vite 生态圈（unplugin/unocss 系工具均出自同一生态）
- 文档/国际化：`docs/` 为 VitePress 站，`CONTRIBUTING.md`、`CODE_OF_CONDUCT.md` 齐备；中文/其他语言翻译采用姊妹仓库模式（README 有徽章，未在 tarball 中确认具体仓库，标注"未确认"）

## 9. 亮点与风险

亮点：

1. 解析与渲染分层极干净：`@slidev/parser` 纯逻辑可独立复用（VSCode 扩展、测试、CLI 共享），Node 侧与浏览器侧靠虚拟模块契约桥接，是 Vite monorepo 的教科书式架构
2. 全链路可编程：Markdown 既是源码也是可无损回写的目标（stringify/prettify + 单页 PATCH 中间件），叠加 preparser/transformer/setup 文件三级扩展点，主题即 npm 包
3. 导出实现务实且稳健：基于 Playwright 打印视图，等待策略覆盖 iframe/Mermaid/Monaco 等慢资源，clicks 展开、PPTX 可编辑文本、PDF 书签等长尾需求都做了
4. 面向 AI 的工作流前瞻性强：MCP 服务器（stdio + HTTP 双通道）、官方 Agent Skill、`packages/vscode/src/lmTools.ts` 的语言模型工具，v53 已产品化
5. 工程纪律好：三平台 CI、import-guard 防包依赖越界、pnpm catalog 统一版本、每个语法特性独立单测

风险/局限：

1. 依赖面很广（CLI 直接依赖 80+，含 Monaco、Shiki、MCP SDK、pptxgenjs 等重型库），供应链与体积压力大，`patches/` 目录的存在也说明对上游有侵入性补丁
2. Node >= 22.12 的硬门槛（根 `package.json` engines）+ pnpm catalog 等新特性，对老环境/公司内网用户不友好
3. 导出强依赖无头 Chromium 与 dev server 端口（默认 18724/3030），沙箱或 CI 环境下排错成本高；`perSlide` 等导出选项已知会破坏跨页链接（`export.ts` 注释自述）
4. 核心贡献者高度集中（antfu 个人及其赞助体系），bus factor 有单一节点风险（无证据表明后继无人，标注为观察）

## 10. 总结评价

Slidev 是"开发者工具吃掉办公软件"路线里最成功的一例：它没有停留在 Markdown 渲染器层面，而是把幻灯片做成了完整的 Vite/Vue 应用平台——解析、编译、主题、扩展、导出、协作编辑、再到 2026 年的 MCP/Agent 支持，每一层都有清晰的代码边界和测试覆盖。约 3.3 万行 TypeScript（packages 内统计）撑起 49k 星，含金量高于多数同量级仓库。适合：需要做技术分享/代码演示的开发者直接使用；想学 Vite 插件体系、虚拟模块设计、monorepo 分层的人精读 `packages/parser/src/core.ts`、`packages/slidev/node/vite/`（尤其 loaders.ts 与 virtual/）和 `packages/slidev/node/commands/export.ts` 这三处；想做演示类产品的人可以研究它的主题 npm 包机制与导出管线。
