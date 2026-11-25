# slab/quill 项目分析

> 星数 38500（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：Quill 是基于 contenteditable 的现代所见即所得富文本编辑器，以 JSON Delta 文档模型为核心抽象，为兼容性与可扩展性而生。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | slab/quill（npm 包 `quill`，当前版本 2.0.3） | `package.json`、`packages/quill/package.json` |
| 星数 | 快照 38500；gh api 实查 47,372 stars / 3,670 forks（2026-11-05 查询） | gh api repos/slab/quill |
| 语言 | TypeScript（v2.0 起由 JS 迁移，附带官方类型声明） | `packages/quill/package.json`、`CHANGELOG.md` v2.0.0 |
| 许可证 | BSD-3-Clause | `LICENSE`、gh api |
| 最近活跃 | 最新 release v2.0.3（2024-11-30）；最近 push 2025-07-25（gh api 查询，维护节奏已放缓） | gh api releases / repos |
| 官网 | https://quilljs.com（源码在 `packages/website`） | `README.md` |
| 代码规模 | 编辑器包约 7700 行 TS 源码（core/modules/themes/blots/ui，不含样式与测试）+ 38 个测试 spec | `src/`、`test/` 行数统计 |

## 2. 项目是什么，解决什么问题

浏览器原生 `contenteditable` 直接编辑 HTML 的体验很差：各浏览器行为不一致、无法可靠表达"文档是什么"（HTML 只是渲染结果）、撤销/协同/粘贴清洗都无从谈起。Quill 的解法是把编辑器拆成"数据模型 + DOM 映射 + UI 壳"三层（`README.md`，`packages/quill/src/` 布局）：

- **数据模型**：文档不是 HTML 而是一串 Delta 操作（`{ insert, delete, retain }` + attributes 的 JSON 数组），由 `quill-delta` 库表达（`packages/quill/package.json` 依赖 `"quill-delta": "^5.1.0"`）。一切 API（`getContents`/`updateContents`/`editor.applyDelta`）都围绕 Delta 进行。
- **DOM 映射**：Parchment 库（`"parchment": "^3.0.0"`）维护"blot 树"与 DOM 节点的双向对应，Quill 在其上定制 Scroll/Block/Inline/Embed 等 blot（`packages/quill/src/blots/`），用 MutationObserver 监听浏览器编辑产生的 DOM 变化并同步回模型。
- **UI 壳**：工具栏、tooltip、选择器等界面全部由"模块（Module）+主题（Theme）"提供，core 构建不含任何 UI（`packages/quill/src/core.ts` 只注册 blots 与核心 modules；`webpack.common.cjs` 中 `quill.core` 与 `quill` 两个入口分离）。

这种设计带来的直接收益：粘贴时把任意 HTML"翻译"为 Delta 再渲染（`src/modules/clipboard.ts` 的 matcher 管线，并为 Google Docs/MS Word 写了专门 normalizer，`src/modules/normalizeExternalHTML/normalizers/`）；撤销重做直接用 `delta.invert()`（`src/modules/history.ts:90`）；文档模型天然适合 OT 协同（package.json keywords 即含 "operational transformation"）。

项目由 Jason Chen 与 Byron Milligan 创建，现由 Slab 公司维护（`README.md:24`）。v2.0（2024-04，`CHANGELOG.md`）是里程碑：正式 ESM 化、迁移 TypeScript、改进 IME/拼写检查支持、嵌套 Quill 支持、大幅优化大文档插入性能。

## 3. 技术栈与依赖

- 语言：TypeScript 5.4（`packages/quill/package.json` devDependencies），`"type": "module"` 纯 ESM 包，浏览器目标 `browserslist: ["defaults"]`。
- 运行时依赖仅 4 个，克制且各司其职：`quill-delta`（文档差分模型）、`parchment`（DOM↔blot 映射）、`eventemitter3`（事件）、`lodash-es`（工具函数）。
- 构建：webpack 5（UMD 产物 + ts-loader/babel + stylus 产 CSS + svg 内联 loader），见 `packages/quill/webpack.common.cjs` 与 `babel.config.cjs`；发布脚本 `packages/quill/scripts/build`（tsc 生成 d.ts → babel 转译 → webpack 打包）。
- 测试：Vitest（browser mode，provider 为 Playwright，`test/unit/vitest.config.ts`）、Playwright E2E（`test/e2e/`）、自写模糊测试（`test/fuzz/`）。
- Monorepo：npm workspaces（根 `package.json` `"workspaces": ["packages/*"]`），官网为 Next.js 14 + MDX + Sandpack（`packages/website/package.json`）。
- 质量工具：ESLint + Prettier + tsc `--noEmit`（`npm run lint`）。

## 4. 目录结构与代码组织

| 目录 | 职责 | 出处 |
| --- | --- | --- |
| `packages/quill/src/core/` | 编辑器内核：quill.ts（门面/配置）、editor.ts（Delta 应用引擎）、selection.ts、emitter.ts、composition.ts（IME）、module.ts、theme.ts | 文件实存 |
| `packages/quill/src/blots/` | 定制 Parchment blot：scroll/block/inline/embed/text/break/cursor/container | 同上 |
| `packages/quill/src/formats/` | 20+ 种格式定义（bold、list、header、image、table…），多为 Attributor 或 blot 子类 | 同上 |
| `packages/quill/src/modules/` | 功能模块：toolbar、clipboard、keyboard、history、syntax、table、tableEmbed、uploader、input、uiNode、normalizeExternalHTML | 同上 |
| `packages/quill/src/themes/` | base/snow/bubble 主题（工具栏与 tooltip UI） | 同上 |
| `packages/quill/src/ui/` | 主题用 UI 组件：picker、color-picker、icon-picker、tooltip、icons | 同上 |
| `packages/quill/src/assets/` | stylus 样式与 SVG 图标（snow/bubble/core 三套 CSS） | 同上 |
| `packages/quill/test/` | unit（vitest 浏览器态）/ e2e（playwright）/ fuzz / types 四类测试 | 同上 |
| `packages/website/` | quilljs.com 官网（Next.js） | `packages/website/package.json` |
| `.github/` | CI workflows、CONTRIBUTING、DEVELOPMENT、release.yml | 文件实存 |

组织评述：典型双包 monorepo——一个发布包（quill）+ 一个文档站包（website），npm workspaces 原生工作区而非 lerna/turbo；编辑器内部则按"core 无 UI / full 注册全部格式与主题"两层组织，入口 `src/core.ts` 与 `src/quill.ts` 分别对应 webpack 的 `quill.core` 与 `quill` 产物。

## 5. 核心架构与关键实现

### 5.1 Delta 文档模型与 Editor 引擎（src/core/editor.ts）

文档状态保存在 `editor.delta`，所有变更以 Delta 描述。`applyDelta`（editor.ts:28-123）是全编辑器的心脏：先把 Delta 规范化并按行拆分（`splitOpLines`），逐 op 调 `scroll.insertAt/formatAt/deleteAt` 落到 blot 树，处理"块级文档必须以 \n 结尾"的隐式换行补齐，最后 `scroll.optimize()` 触发 DOM 重建并 `update()` 派发 TEXT_CHANGE。关键设计是"模型先行"：DOM 是 Delta 的投影，而不是反过来：

```ts
applyDelta(delta: Delta): Delta {
  this.scroll.update();
  let scrollLength = this.scroll.length();
  this.scroll.batchStart();
  const normalizedDelta = normalizeDelta(delta);
  ...
  this.scroll.batchEnd();
  this.scroll.optimize();
  return this.update(normalizedDelta);
}
```

（`src/core/editor.ts:28-123`；隐式换行判定见 `isImplicitNewlineAppended` 等局部逻辑。）

### 5.2 Parchment 映射层与 Scroll blot（src/blots/scroll.ts）

`Scroll extends Parchment.ScrollBlot`（scroll.ts:31），是 blot 树的根，挂载在 `.ql-editor` 容器上，通过 MutationObserver 监听原生编辑行为。它做了三件关键事：批量合并（`batchStart/batchEnd`，把一批 DOM mutation 攒起来一次性 update，editor 的每次 applyDelta 都包在其中）；行级语义（`deleteAt` 删跨行时把相邻行合并、维护 `allowedChildren = [Block, BlockEmbed, Container]`）；向 core 发事件（`emitMount/emitEmbedUpdate` → `Emitter.events.SCROLL_UPDATE`，quill.ts:241-267 据此回写模型）。嵌入式 blot 还支持 `updateContent(change)` 增量更新协议（scroll.ts:23-29）。

### 5.3 Module/Theme 扩展机制（src/core/module.ts、src/core/theme.ts、src/core/quill.ts:793-877）

Module 基类只有 12 行（构造时注入 `quill` 实例与 options），扩展点全部围绕它：`Quill.register('modules/xxx', Class)` 挂到静态 imports 注册表；`expandConfig` 把 `Quill.DEFAULTS.modules`、主题 DEFAULTS、用户传入 `modules` 三层 merge，`true` 简写展开为 `{}`，再按 `modules/<name>` 找回类并 merge 其 `DEFAULTS`；主题在 `theme.addModule(name)` 中实例化（theme.ts:43-51）。内核构造函数固定初始化 keyboard/clipboard/history/uploader/input/uiNode 六个模块（quill.ts:229-234），toolbar/syntax/table 等则由 full 构建（`src/quill.ts` 注册）或用户配置引入。snow/bubble 主题差异只是 UI 呈现：snow 固定顶栏 + 链接 tooltip（`themes/snow.ts:13-18` 的 TOOLBAR_CONFIG 与 SnowTooltip），bubble 选区浮动工具栏（`themes/bubble.ts`）。

### 5.4 History：以 Delta 逆变换实现撤销（src/modules/history.ts）

撤销栈不存快照而存"逆 Delta"：监听 TEXT_CHANGE 时 `record(changeDelta, oldDelta)` 用 `changeDelta.invert(oldDelta)` 算出撤销操作（history.ts:111-120），1 秒内的连续编辑合并为一条（`delay` 配置，history.ts:25-29）；执行撤销时对当前内容再取一次逆，推入重做栈（history.ts:85-101）：

```ts
change(source: 'undo' | 'redo', dest: 'undo' | 'redo') {
  const item = this.stack[source].pop();
  const base = this.quill.getContents();
  const inverseDelta = item.delta.invert(base);
  this.stack[dest].push({ delta: inverseDelta, ... });
  this.quill.updateContents(item.delta, Quill.sources.USER);
}
```

协同场景下非用户来源的变更通过 `transform(value)` 重放对齐（history.ts:47-52），`userOnly` 选项控制是否只记录用户输入。选区也随 Delta 一并记录与变换（v2.0 新增，CHANGELOG.md #3823）。

### 5.5 Keyboard 快捷键绑定（src/modules/keyboard.ts）

833 行的 keyboard 模块维护一张"绑定表"：`addBinding({ key, shortKey, shiftKey, ... }, handler)` 按修饰键从宽到窄匹配，`shortKey` 自动映射 Mac Cmd / 其他平台 Ctrl；内置 binding 覆盖 Enter（拆分/继承行格式）、Backspace/Delete（跨格式删除的 `deleteRange`）、Tab、格式化组合键等（keyboard.ts:87-105 起的默认注册段）。与 IME 相关的保护逻辑放在 composition.ts 与 input 模块（`src/core/composition.ts`、`src/modules/input.ts`），v2.0 专门改进了 Safari 输入合成期 Enter/Backspace 的副作用（CHANGELOG.md #4201）。

### 5.6 Clipboard 粘贴清洗管线（src/modules/clipboard.ts）

粘贴被 `onCapturePaste`（clipboard.ts:196 起）拦截：优先取 `text/html`，文件走 uploader，URI-list 归一化；HTML 经 DOMParser 解析后先过 `normalizeExternalHTML`（内置 googleDocs.ts、msWord.ts 两个专门的脏 HTML 清洗器），再由 `CLIPBOARD_CONFIG` 的选择器→matcher 顺序管线（clipboard.ts:32-48，含 `matchBlot`/`matchAttributor`/`matchStyles`/表格与列表匹配）递归遍历 DOM 产出 Delta，最后走常规 `updateContents`。复制同理反向：把选中 Delta 序列化为 html+text 写回剪贴板（`onCaptureCopy`）。

## 6. 构建与运行

来自 `.github/DEVELOPMENT.md` 与各 package.json scripts（npm workspaces，不支持 yarn/pnpm）：

```bash
npm install                 # 安装 workspace 依赖
npm start                   # 并行起 webpack-dev-server(:9080) 与官网(:9000)
npm run build               # = build:quill + build:website
npm run lint                # eslint + tsc --noEmit
npm run test:unit -w quill  # vitest（浏览器模式，chromium）
npm run test:fuzz -w quill  # 模糊测试
npm run test:e2e  -w quill  # playwright
```

发布产物（`packages/quill/scripts/build` + `webpack.common.cjs` 入口）：`dist/quill.js`（full，UMD 全局 `Quill`）、`dist/quill.core.js`、`quill.snow.css`/`quill.bubble.css`/`quill.core.css`，同时拷贝 package.json/README/LICENSE 与 babel 生成的 ESM+d.ts。

## 7. 测试与工程质量

- 规模：38 个 spec 文件；unit 32 个约 8312 行（core/blots/formats/modules/theme/ui 全覆盖，`test/unit/`），e2e 4 个 spec 约 610 行（full/history/list/replaceSelection，`test/e2e/`），fuzz 2 个约 448 行（对 editor 随机 Delta 序列做不变量校验，`test/fuzz/editor.spec.ts`）。另有 `*.test-d.ts` 类型测试（vitest typecheck，`test/unit/vitest.config.ts`）。
- 风格：unit 跑在真实浏览器（vitest browser mode + playwright provider），贴近编辑器强依赖 DOM/选区的现实；fuzz 测试在富文本编辑器项目中少见，是亮点。
- CI：`.github/workflows/main.yml`（push main）与 `pull-request.yml` 复用 `_test.yml`；另有 changelog.yml、label.yml、release.yml（release notes 由 `.github/release.yml` 配置自动生成，CHANGELOG.md 头部注有此说明）。
- 质量观察：类型化彻底，但代码内遗留约 150 处 `@ts-expect-error`（多数带 "Fix me later" 注释，集中在 quill.ts:25、keyboard.ts:25、clipboard.ts:14、syntax.ts:17 等），属 v1→v2 迁移的技术债；核心逻辑（editor/scroll）类型干净。

## 8. 社区与维护状态

- gh api 实查（2026-11-05）：stars 47,372、forks 3,670、open issues 664、`pushed_at` 2025-07-25、未归档、默认分支 main。
- Release 节奏：v2.0.0（2024-04-17）→ v2.0.1/2.0.2（2024-05）→ v2.0.3（2024-11-30），此后无新 release；最近一次 push 为 2025-07，距本次分析已逾一年，维护明显放缓（gh api 实查；CHANGELOG.md 头部止于 v2.0.2，2.0.3 仅见于 GitHub releases 与 package.json）。
- 贡献者：contributors API 分页推算约 160+ 人（per_page=1 最后一页为 161，含匿名）。核心维护者集中于 Slab 团队（如 luin 大量出现在 release notes，`CHANGELOG.md` v2.0.2）。
- 社区通道：GitHub Issues + Discussions（`README.md:99-100`）；官网文档与 playground 由 `packages/website` 承载。
- 快照星数（38500）与 API 实查值存在出入，以上传 API 实查为准。

## 9. 亮点与风险

**亮点**
1. 架构分层干净：Delta 模型（quill-delta）/ DOM 映射（parchment）/ UI（module+theme）三层解耦，core 构建可零 UI 使用（`src/core.ts` 与 `webpack.common.cjs` 双入口）。
2. 运行时依赖仅 4 个，均为核心抽象本身，供应链面小（`packages/quill/package.json`）。
3. 文档模型即数据接口：API、剪贴板、撤销（`delta.invert`）、协同扩展共享同一 Delta 表示，`test/fuzz/` 对模型做随机化不变量验证。
4. 工程现代化到位：TypeScript + ESM + vitest 浏览器态测试 + Playwright e2e + fuzz + 类型测试（v2.0.0 CHANGELOG 与 test/ 目录佐证）。
5. 粘贴体验打磨深：针对 Google Docs/MS Word 的专门 normalizer（`src/modules/normalizeExternalHTML/normalizers/`），这是同类编辑器的经典痛点。

**风险/局限**
1. 维护放缓：最后 release 2024-11、最后 push 2025-07（gh api 实查），664 个 open issue 长期积压。
2. `@ts-expect-error` 约 150 处散布在模块与主题层，API 周边的类型体验有折扣（各模块文件统计）。
3. contenteditable + MutationObserver 路线的固有复杂度仍集中在 editor.ts/scroll.ts 的隐式换行与行合并等边界逻辑，改动风险高（`src/core/editor.ts:40-112`）。
4. 内置 table/syntax 模块能力相对基础（`src/modules/table.ts` 仅 160 行），复杂表格需求实际依赖第三方扩展（官网 playground 生态，未在本仓内核实）。

## 10. 总结评价

Quill 是 contenteditable 富文本编辑器领域"模型驱动"路线的代表作品：以 Delta 为唯一事实源、以 Parchment 做 DOM 双向映射、以 Module/Theme 做无侵入扩展，四依赖的精简运行时和覆盖 unit/fuzz/e2e 的测试体系使其成为学习编辑器架构设计的一流范本。v2.0 完成现代化改造后，工程面貌焕然一新，但 2024 年末以来社区维护节奏明显放缓，选型时需把"后续重大功能主要靠自身扩展"纳入预期。适合：需要可控富文本能力且愿意基于 Module/Format 机制二次开发的应用团队；以及想深入理解文档模型、DOM 同步与粘贴清洗设计的编辑器方向开发者读源码（建议入口：`src/core/editor.ts` → `src/blots/scroll.ts` → `src/modules/clipboard.ts`）。
