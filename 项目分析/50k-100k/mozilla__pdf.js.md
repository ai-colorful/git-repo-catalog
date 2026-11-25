# mozilla/pdf.js 项目分析

> 星数 54037（快照 2026-10-24）｜主语言 JavaScript｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：**Firefox 内置的 PDF 渲染器，也是 Web 上最通用的"纯前端解析 + 渲染 PDF"平台库**——把 PDF 字节流在浏览器/Node 中解析为内部对象与绘图操作符，再画到 HTML5 canvas 上。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | mozilla/pdf.js | GitHub |
| 星数 | 54037（快照）；gh api 实测 54090 | 任务参数 / gh api |
| 主语言 | JavaScript（纯 JS + WASM 组件） | 仓库 |
| 许可证 | Apache-2.0 | `LICENSE` |
| 创建时间 | 2011-04-26 | gh api |
| 最近 push | 2026-11-03（分析前一天，极活跃） | gh api |
| 当前版本 | 6.5.438（stableVersion） | `pdfjs.config` |
| 规模 | src/ 222 个 JS 文件约 13.85 万行；web/ 约 3.73 万行 | 实测统计 |
| 默认分支 | master | gh api |

## 2. 项目是什么，解决什么问题

2011 年前后浏览器无法原生渲染 PDF，Mozilla 发起 PDF.js，目标是用 Web 标准技术（无插件）实现通用 PDF 渲染平台。它随 Firefox 19 起成为浏览器内置查看器（`README.md`："PDF.js is built into version 19+ of Firefox"），同时提供 Chrome 扩展（`npx gulp chromium` 构建，`README.md`、`extensions/chromium/`）和可嵌入的通用库。

项目同时是"库"和"应用"两种形态：库形态通过 `src/display/api.js` 的 `getDocument()` 暴露 API，把解析渲染放进 Web Worker（`src/pdf.worker.js`），主线程只拿 Promise 结果；应用形态是 `web/` 下的完整 viewer（工具栏、缩略图、查找、注释编辑等），Firefox 内置版、Chrome 扩展版与 generic 版共用同一 viewer 代码，通过构建目标区分（`web/firefoxcom.js`、`web/chromecom.js`、`web/genericcom.js`）。

其核心技术路线是自实现整条 PDF 管线：不依赖系统 PDF 引擎，自己解析 PDF 语法（对象、交叉引用表、流过滤器 Flate/LZW/JBIG2/JPX/CCITT 等）、自己解释内容流操作符、自己把字体转译成 Web 字体绘制、最后用 Canvas 2D（可选用 OffscreenCanvas/WebGPU）绘制。这使它的渲染结果跨平台一致，也带来了海量的兼容性工程（`test/pdfs/` 下 1540 个真实 PDF 回归语料即为此服务）。

近年方向（从源码可见）：XFA 表单（`src/core/xfa/` 23 个文件）、注释编辑器（`src/display/editor/`、`web/` 编辑相关文件）、数字签名、WASM 化的性能组件（openjpeg、qcms 色彩管理、quickjs 脚本引擎、PostScript 函数编译为 WASM），以及探索 WebGPU 渲染（`src/display/webgpu.js`）。

## 3. 技术栈与依赖

- **语言**：现代 JavaScript（ESM，`package.json` 中 `"type": "module"`），用 JSDoc + TypeScript 做类型检查（`tsconfig.json`、`jsdoc.json`、CI 的 `types_tests.yml`），源码中大量 `@import`/`@typedef` 注释。
- **运行时依赖：零**。`package.json` 只含 `devDependencies`，无 `dependencies`；所有第三方能力被静态打包进仓库：openjpeg（`external/openjpeg/`，JPX/JPEG2000 解码 WASM）、qcms（`external/qcms/`，ICC 色彩）、quickjs（`external/quickjs/quickjs-eval.wasm`，执行 PDF JavaScript）、brotli 解码（`src/core/brotli_stream.js`）、Fluent 本地化（`l10n/`）。
- **构建工具链**：gulp 任务系统（`gulpfile.mjs`，3100+ 行）+ webpack + Babel（`devDependencies` 可见），`PDFJSDev` 预处理宏区分 MOZCENTRAL/GENERIC/TESTING 等构建目标（`src/display/api.js` 多处）。
- **测试工具**：Jasmine（单元/浏览器测试）、vitest + @codspeed（性能基准）、Puppeteer（集成测试）、istanbul/codecov（覆盖率）。
- **运行环境**：浏览器 + Node（Node 侧用 `@napi-rs/canvas`，见 `src/display/node_utils.js`、`src/display/dom_canvas_factory.js` 的 NodeCanvasFactory 分支）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `src/pdf.js` / `src/pdf.worker.js` | 两个 bundle 入口：主线程 API 与 Worker 线程（各约 20-230 行，聚合导出） |
| `src/display/` | 主线程侧：`api.js`（getDocument/PDFDocumentProxy/PDFWorker/WorkerTransport）、`canvas.js`（CanvasGraphics 渲染执行）、text_layer、annotation_layer、网络加载（fetch/http/stream） |
| `src/core/` | Worker 侧引擎：语法解析、xref、求值器、字体、图像/流解码、加密、XFA、编辑器数据模型（93 个文件） |
| `src/shared/` | 主/Worker 共享工具：`message_handler.js`（RPC 消息协议）、`util.js` 等 |
| `src/scripting_api/` | PDF JavaScript 的 AcroForm API 模拟层（app/doc/field/event…） |
| `web/` | 完整查看器应用：`app.js`（PDFViewerApplication，3604 行）、`pdf_viewer.js`（页面容器）、查找/缩略图/大纲/签名/打印/多端适配层，及 `web/internal/`（PDF 调试器） |
| `external/` | 预编译第三方资产：openjpeg、quickjs、qcms、bcmaps（CJK 字符映射 100+ 个）、standard_fonts |
| `test/` | `unit/`（80 个 Jasmine spec）、`integration/`（Puppeteer e2e）、`pdfs/`（1540 个参考 PDF + `test_manifest.json`）、`font/`、`chromium/` |
| 其他 | `examples/`（最小用法）、`extensions/chromium/`、`docs/`（内容文档）、`l10n/`、`gulpfile.mjs` |

组织方式评述：单一 monorepo 式单包结构（非 npm workspace 多包），按"主线程 display / Worker 线程 core / 共享 shared"的线程边界切分模块，是本项目最鲜明的架构约定；viewer 与库同仓但分层清晰（`web/` 只 import `src/display` 的公共 API）。

## 5. 核心架构与关键实现

### 5.1 双线程消息协议（worker 架构底座）

主线程 `getDocument()`（`src/display/api.js:228`）创建 `PDFDocumentLoadingTask` 与 `PDFWorker`，经 `WorkerTransport`（`api.js:2365`）用 `MessageHandler`（`src/shared/message_handler.js`，抽象了 "GetDocRequest/GetPage/Render..." 等 RPC 与流式传输 StreamKind）与 Worker 侧通信。Worker 侧 `WorkerMessageHandler.createDocumentHandler`（`src/core/worker.js:93`）做版本一致性强校验后建 `pdfManager`：

```js
// src/core/worker.js
const { docId, apiVersion } = docParams;
...
if (apiVersion !== workerVersion) {
  throw new Error(
    `The API version "${apiVersion}" does not match the Worker version "${workerVersion}".`
  );
}
```

主/Worker 版本不匹配直接报错，避免半兼容的协议解析。无 Worker 环境自动退化为 fake worker（`PDFWorker`，`api.js:2088`），Node 与浏览器同构。

### 5.2 解析层（src/core）：按需字节流 + 对象系统

PDF 文件不一定完整下载，`ChunkedStream`（`src/core/chunked_stream.js`，539 行）把原始字节切成 64KB 块按需向主线程请求（range request），`MissingDataException` 驱动的"缺数据即暂停、补齐后重放"贯穿解析器。`xref.js`（1072 行）解析交叉引用表/交叉引用流，`object_loader.js` 缓存间接对象，`parser.js`（1450 行）实现 `Parser`（对象/流/内联图，`filter()` 方法分发到 Flate/LZW/JBIG2/JPX/CCITT/Brotli 等 12 种过滤器实现）与 `Lexer`（数字/字符串/名字/命令词法）。`document.js`（2415 行）的 `PDFDocument`/`Page` 组装目录树、加密（`crypto.js`，AES/RC4）与页面资源。

### 5.3 求值器：内容流 → 操作符列表（解析与渲染的解耦点）

`PartialEvaluator.getOperatorList`（`src/core/evaluator.js:1691`，全文 5546 行）把页面内容流翻译成编码后的绘图指令对：`fnArray`（OPS 操作码）+ `argsArray`（参数），存于 `operator_list.js`。这是双层架构的接口：Worker 侧不画图，主线程不理解 PDF 语法。实现上用 `TimeSlotManager` 分时让出 Worker 事件循环、四个本地缓存（图像/色彩空间/GState/TilingPattern）+ `seenRefs` 防循环引用。

### 5.4 渲染层（src/display/canvas.js）：时间片执行操作符

`CanvasGraphics.executeOperatorList`（`src/display/canvas.js:730`，类起于 555 行，全文 4564 行）在主线程/渲染 Worker 上回放操作符：

```js
// src/display/canvas.js
const chunkOperations =
  argsArrayLen - i > EXECUTION_STEPS &&
  typeof continueCallback === "function";
const endTime = chunkOperations ? Date.now() + EXECUTION_TIME : 0;
...
while (true) {
  ...
  this[fnId](i, ...fnArgs);   // OPS → CanvasGraphics 方法直查表
}
```

每个 OPS 操作码绑定为 `CanvasGraphics.prototype[op]`（`canvas.js:4560`），长任务按 `EXECUTION_TIME` 时间片切片让出主线程保证 UI 流畅。滤镜用 CSS/Canvas/Worker 三种工厂（`dom_filter_factory.js`/`worker_filter_factory.js`），可选 WebGPU 做 mesh shading（`src/display/webgpu.js:143`）。文本层、注释层、XFA 层（`text_layer.js`/`annotation_layer.js`/`xfa_layer.js`）与 canvas 叠加合成。

### 5.5 字体、图像与 WASM 化的性能内核

- 字体：`src/core/fonts.js`（3856 行）把 PDF 内嵌字体（CFF/Type1/TrueType，`cff_parser.js`/`type1_parser.js`/`glyf.js`）转译为浏览器 FontFace，字形路径提取在 `font_renderer.js`（947 行）。
- 图像：`jpg.js`（纯 JS JPEG）、`jpx.js` + `external/openjpeg/openjpeg.wasm`（JPEG2000，`WasmLoader` 按需取 `wasmUrl` 并跨文档共享编译缓存，`src/core/wasm_loader.js`）、JBIG2/CCITT 传真（`jbig2_stream.js`/`ccitt_stream.js`）。
- 一个亮点：`src/core/postscript/wasm_compiler.js` 把 PDF Type 4 函数（PostScript 计算式，用于着色函数/裁剪框）直接编译成手写 WASM 字节码执行，替代逐 token 解释，配合 `postscript/lexer.js`/`ast.js`。
- PDF JavaScript：`external/quickjs/quickjs-eval.wasm` 在沙箱里执行文档脚本，`src/scripting_api/` 提供 AcroForm 兼容对象模型。

### 5.6 viewer 应用（web/）

`web/app.js` 的 `PDFViewerApplication`（3604 行）是组装根：接管事件、偏好、下载、打印；`web/pdf_viewer.js`（2823 行）管理页面滚动与渲染队列 `pdf_rendering_queue.js`；查找、缩略图、大纲、注释编辑、签名（`signature_manager.js`）、撤销栏（`editor_undo_bar.js`）各为独立组件；Firefox/Chrome/generic 三端差异收敛在 `firefoxcom.js`/`chromecom.js`/`genericcom.js` 适配层；`web/viewer-geckoview.js` 是 Firefox Android（GeckoView）专用变体。

## 6. 构建与运行

来自 `README.md`（60-183 行）与 `gulpfile.mjs`：

```bash
npm ci                     # 安装依赖（Node >= 22.22.3，见 package.json engines）
npx gulp server            # 本地开发服务器
npx gulp generic           # 通用版构建 → build/generic
npx gulp generic-legacy    # 老浏览器兼容版
npx gulp chromium          # Chrome 扩展 → build/chromium

npx gulp unittestcli --coverage        # Node 单元测试
npx gulp unittest --coverage           # 浏览器单元测试
npx gulp integrationtest --coverage    # Puppeteer 集成测试
npx gulp botbrowsertest --coverage     # 参考回归测试
npx gulp makeref                       # 生成参考图
npx gulp benchmark                     # CodSpeed 性能基准
```

注意 `package.json` 无 `scripts` 字段，所有命令统一走 gulp task（`gulpfile.mjs`）。

## 7. 测试与工程质量

- **单元测试**：`test/unit/` 80 个 `*_spec.js`（Jasmine 风格，共 354 个 `describe`），覆盖 evaluator、fonts、parser、crypto、image、message_handler 等核心模块（实测统计）。
- **参考回归**：`test/pdfs/` 1540 个真实世界 PDF + `test/test_manifest.json` 驱动像素级/文本级对比（`driver.js`、CI `ref_tests_makeref.yml`），另配专门的字体渲染比对（`test/font/`、CI `font_tests.yml`）。这是 PDF 渲染器最关键的质量资产。
- **端到端**：`test/integration/` 24 个 Puppeteer spec（查找、编辑器各类型、无障碍、脚本、geckoview…）。
- **CI**：`.github/workflows/` 下 19 个工作流：`ci.yml`（外部测试 + CLI 单测 + codecov）、`unit_tests.yml`、`integration_tests.yml`、`coverage_browser_tests.yml`、`types_tests.yml`（类型检查）、`codeql.yml`（安全扫描）、`codspeed.yml`（性能基准）等，全部以 Node 22-26 矩阵运行。
- **质量观察**：eslint 配置极严格（`eslint.config.mjs` 含 no-unsanitized、regexp、perfectionist 等插件），stylelint、svglint、license lint（`gulpfile.mjs` 的 `lint-licenses`/`lint-bom`）；源码注释详尽，JSDoc 类型贯穿全库。

## 8. 社区与维护状态

- **活跃度**：gh api 显示最近 push 为 2026-11-03（分析前一天），479 个 open issue，10734 fork；主干几乎每日有提交，属 GitHub 上维护最活跃的头部项目之一。
- **维护主体**：Mozilla 官方支持（`README.md`："community-driven and supported by Mozilla"），有专门的安全/发布流程（`publish_release.yml`、`notify-pdf-sync.yml` 用于向 Firefox 同步）；社区贡献走 `.github/CONTRIBUTING.md` 与 Matrix 房间。
- **发布节奏**：`pdfjs.config` 显示版本 6.5.438（versionPrefix 6.5，baseVersion 绑定具体 commit），patch 号高频递增，属滚动式小版本发布；每周左右一次 stable（未从 tarball 直接确认，属合理推断）。
- **生态位**：Firefox 内置、Chrome 官方扩展（由 @Rob--W 维护）、React/Vue 等大量包装库的底层依赖。

## 9. 亮点与风险

**亮点**

1. 完整自研 PDF 管线：词法/语法解析 → 12 种流过滤器 → 求值器 → canvas 回放，全部无运行时第三方依赖（`src/core/`、`src/display/`），这是对 Web 平台能力的极限利用。
2. 主/Worker 双线程架构干净利落：线程边界即模块边界（display/core/shared），`message_handler.js` 统一 RPC + 流式传输，且自带版本握手（`src/core/worker.js`）。
3. 工程质量天花板：1540 个参考 PDF 的像素回归、19 条 CI 流水线、严格 lint、JSDoc 类型化纯 JS，是"大型纯 JS 项目如何做质量"的教科书样本。
4. 持续演进的技术内核：PostScript 函数编译为 WASM（`src/core/postscript/wasm_compiler.js`）、quickjs 沙箱脚本、qcms 色彩管理、WebGPU 渲染试验（`src/display/webgpu.js`）。
5. 可复用性分层清晰：只取 `src/pdf.js`+worker 即可嵌入任意应用；viewer 三端适配层把 Firefox/Chrome/generic 差异压到最小的几个文件。

**风险/局限**

1. 体量与复杂度：核心 13.8 万行 + 4.5 千行/文件级巨石模块（`evaluator.js` 5546 行、`canvas.js` 4564 行），修改门槛高，测试依赖大批参考语料。
2. 自研字体/图像栈长尾兼容性问题多（issue 数 479 长期高位），复杂 PDF 的渲染保真度与 Acrobat 仍有差异（渲染器对损坏/畸形文件采取容错跳过策略，`parser.js` recoveryMode）。
3. 构建体系重：gulpfile 3100+ 行、多目标预处理宏 `PDFJSDev`，新贡献者上手成本高；`package.json` 无 scripts，入口不直观。
4. 主线程渲染仍是瓶颈场景（`canvas.js` 时间片只是缓解；WebGPU 路径尚在演进，`src/display/webgpu.js` 中 WebGPU 不可用时回退）。

## 10. 总结评价

PDF.js 是"用 Web 技术重写桌面软件能力"的典范项目：十四年持续演进，从让 Firefox 摆脱插件依赖，长成覆盖解析、渲染、表单、编辑、脚本沙箱的完整 PDF 平台，并被整个 Web 生态当作默认 PDF 方案。它的架构——线程边界切分模块、操作符列表作为解析/渲染解耦点、大规模参考语料驱动回归——非常值得浏览器内核与前端基础设施开发者精读；普通应用开发者则适合直接把 `getDocument()` 与 viewer 构建产物作为嵌入 PDF 能力的标准选择。想读源码者建议按 `src/display/api.js` → `src/core/worker.js` → `src/core/parser.js` → `src/core/evaluator.js` → `src/display/canvas.js` 的顺序进入，正好对应"一次 getDocument 调用"的完整数据流。
