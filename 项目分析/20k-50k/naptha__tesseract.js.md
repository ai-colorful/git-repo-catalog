# naptha/tesseract.js 项目分析

> 星数 41500（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：把 Tesseract OCR 引擎（C++）的 WebAssembly 移植包装成纯 JavaScript 库，让浏览器与 Node.js 都能离线做 100+ 语言的图片文字识别。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | naptha/tesseract.js | gh api repos/naptha/tesseract.js |
| 星数 | 41500（任务快照 2026-10-24）；gh api 实测 38829（2026-11-14） | gh api |
| 主语言 | JavaScript（CommonJS，含 TypeScript 类型声明） | `package.json`、`src/index.d.ts` |
| 许可证 | Apache-2.0 | `package.json`、`LICENSE.md` |
| 默认分支 | master，最近 push 2026-05-17 | gh api |
| 最新版本 | v7.0.0（2025-12-15 发布），master 上 `package.json` 版本 7.0.0 | gh api releases、`package.json` |
| 贡献者 | 79 人 | gh api contributors（Link header last=79） |
| 贡献者署名 | Kevin Kwok（antimatter15）、Guillermo Webster、Jerome Wu（jeromewus） | `src/worker-script/index.js` 文件头注释 |

## 2. 项目是什么，解决什么问题

Tesseract.js 是 2015 年创建的老牌项目（gh api：created 2015-06-24），目标是把 Google 的开源 OCR 引擎 Tesseract（C++）搬到 Web 与 Node.js 环境中。它本身不实现任何 OCR 算法，而是作为"胶水层"包装另一个仓库 naptha/tesseract.js-core——那是用 Emscripten 把 Tesseract C++ 编译成 WebAssembly 的产物。README 的 Project Scope 一节明确写到本项目"不修改 Tesseract 核心特性，不支持 PDF 输入，不修改识别模型"，功能外需求引导用户使用其作者参与的 Scribe.js（`README.md` Project Scope 节）。

它解决的核心问题有三个。第一是跨环境运行：OCR 计算被隔离在独立线程（浏览器 Web Worker / Node worker_threads）中执行，不阻塞 UI，且同一套 API 同时支持浏览器（script 标签、webpack、ESM、CDN）和 Node.js（`README.md`；`package.json` 的 `browser` 字段映射）。第二是零安装依赖：主程序、WASM 核心、语言训练数据全部可从 jsDelivr CDN 按需加载，用户无需本地安装 Tesseract 二进制（`src/worker-script/browser/getCore.js`、`src/worker-script/index.js` 的 loadLanguage）。第三是易用性：v5 之后 `createWorker('eng')` 一行即可完成核心加载、语言下载、初始化全流程，内部自动串行调度（`src/createWorker.js`）。

语言支持方面，`src/constants/languages.js` 定义了 102 个语言代码常量（如 `CHI_SIM: 'chi_sim'`），对应 Tesseract tessdata 4.0.0 的全部训练数据；GitHub 仓库描述为 "Pure Javascript OCR for more than 100 Languages"（任务描述中的"62 语言"为更早时期口径，当前代码为 102 项）。语言数据按 `@tesseract.js-data/<lang>` 的 npm 包发布在 jsDelivr，按需下载（`src/worker-script/index.js` loadLanguage 内默认 URL）。

版本演进反映了对体积与内存的持续优化：v5 将默认文件显著缩小（README 称英文缩小 54%、中文缩小 73%，首次运行提速约 50%）；v6 修复内存泄漏、默认只输出 `text`；v7 移除 regenerator-runtime、要求 Node ≥16（`README.md` Major changes 各节；gh api 最近提交 "Dropped Node.js v14 support (#1041)"、"Removed regenerator-runtime per #1044 (#1066)"）。

## 3. 技术栈与依赖

- 语言与模块体系：纯 JavaScript（CommonJS），`type: "commonjs"`，TypeScript 类型由 `src/index.d.ts` 手写提供（`package.json`）。
- 运行时依赖（`package.json` dependencies，全部小而专）：
  - `tesseract.js-core ^7.0.0`：Tesseract C++ 的 Emscripten/WASM 编译产物（独立仓库），提供 `TessBaseAPI` 等 C++ 导出类；
  - `wasm-feature-detect`：运行时探测 SIMD / relaxed-SIMD 支持，选最优 WASM 核心；
  - `idb-keyval`：浏览器端用 IndexedDB 缓存 `.traineddata` 语言包；
  - `zlibjs`：浏览器端 gunzip 解压语言包（Node 端用内置 zlib，见 `src/worker-script/node/gunzip.js`）；
  - `bmp-js`：BMP 图像格式支持；`is-url`：区分 langPath 是 URL 还是本地路径；`node-fetch ^2.6.9`：老版本 Node 的 fetch 兜底；`opencollective-postinstall`：赞助提示。
- 构建工具链：webpack 5（主线程 UMD 包 + 浏览器 worker 包）、rollup（ESM 包）、babel（`scripts/webpack.config.prod.js`、`scripts/rollup.esm.mjs`、`babel.config.json`）。
- 测试链：mocha + expect.js + nyc（覆盖率）+ karma（ChromeHeadless/FirefoxHeadless）+ npm-run-all/wait-on（`package.json` scripts、`karma.conf.js`）。
- 代码规范：ESLint 7 + airbnb-base 配置（`package.json` devDependencies、`.eslintrc`）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `src/index.js` | 入口，导出 createWorker/createScheduler/recognize/detect/languages/OEM/PSM |
| `src/Tesseract.js` | 一次性便捷 API：`recognize(image, langs)` 内部临时建 worker→识别→terminate |
| `src/createWorker.js` | 主线程侧 worker 生命周期与 RPC 客户端（核心，246 行） |
| `src/createScheduler.js` | 多 worker 调度池（82 行） |
| `src/createJob.js` | 生成带唯一 id 的任务对象 |
| `src/worker/{browser,node}/` | 主线程侧按运行时区分的 spawnWorker/onMessage/send/loadImage 适配 |
| `src/worker-script/index.js` | worker 线程侧统一业务逻辑：load/FS/loadLanguage/initialize/recognize/detect（543 行） |
| `src/worker-script/{browser,node}/` | worker 线程侧运行时适配器：getCore（WASM 加载）、cache（缓存）、gunzip、index（接线） |
| `src/constants/` | OEM、PSM、默认输出、默认选项、102 个语言代码 |
| `src/utils/` | getEnvironment（运行环境判断）、log、resolvePaths、getId |
| `src/index.d.ts` | 全量 TypeScript 声明 |
| `docs/` | api.md、faq.md、performance.md、workers_vs_schedulers.md、image-format.md、local-installation.md 等文档 |
| `tests/` | 4 个 mocha 集成测试（421 行）+ 图像资产 |
| `examples/` | browser 4 个 / node 4 个官方示例 |
| `benchmarks/` | node 与 browser 的速度、内存、自动旋转基准（`benchmarks/readme.md`） |
| `scripts/` | webpack/rollup 配置、开发服务器 server.js、test-helper.mjs |
| `.github/workflows/` | Node.js CI（lint+test 矩阵）与 CodeQL 扫描 |

组织方式评述：单包仓库（非 monorepo），但生态上是"三仓协作"——本仓库（JS 胶水）、tesseract.js-core（WASM 编译）、@tesseract.js-data/*（100+ 语言数据包）。双运行时不是靠 if/else，而是靠 npm `browser` 字段映射（`package.json`：`"./src/worker/node/index.js": "./src/worker/browser/index.js"`）+ 运行环境探测（`src/utils/getEnvironment.js`），主线程与 worker 线程各有一层对称的 browser/node 适配目录，业务逻辑集中在无环境依赖的 `src/worker-script/index.js`。代码规模紧凑：全部源码 118 个文件（tarball 与 GitHub tree 逐文件核对一致，无 export-ignore 缺失）。

## 5. 核心架构与关键实现

### 5.1 主线程-Worker 通信协议（RPC over postMessage）

任务以 `{workerId, jobId, action, payload}` 数据包发送（`src/createWorker.js` startJob），worker 侧 `src/worker-script/index.js` 的 `dispatchHandlers` 按表分发到 8 个 action：`load / FS / loadLanguage / initialize / setParameters / recognize / detect / terminate`。响应统一为 `{status: 'resolve'|'reject'|'progress', data}`，主线程以 `${action}-${jobId}` 为键存取 Promise（`src/createWorker.js` 第 51-64、202-222 行）：

```js
onMessage(worker, ({ workerId, jobId, status, action, data }) => {
  const promiseId = `${action}-${jobId}`;
  if (status === 'resolve') { promises[promiseId].resolve({ jobId, data }); ... }
  else if (status === 'reject') { ... }
  else if (status === 'progress') { logger({ ...data, userJobId: jobId }); }
});
```

进度事件直接映射到用户可注入的 `logger` 回调；WASM 内部的识别进度由 Emscripten 回调 `TesseractProgress(percent)` 映射为 `(percent-30)/70` 归一化到 0-1（`src/worker-script/index.js` 第 46-53 行）。createWorker 内部把 `load → loadLanguage → initialize` 串成 Promise 链后才 resolve worker 对象，用户拿到即是"预热完毕"的实例（`src/createWorker.js` 第 239-245 行）。

### 5.2 双运行时适配层（browser/node 对称目录）

- 主线程侧：`src/worker/browser/spawnWorker.js` 用 Blob URL + `importScripts` 包一层创建 `new Worker`（规避跨域限制，可 `workerBlobURL: false` 关闭）；`src/worker/node/spawnWorker.js` 用 `new Worker(workerPath)`（worker_threads）。
- worker 侧：业务模块只通过 `setAdapter()` 注入的抽象接口调用环境能力（`src/worker-script/index.js` 第 540-542 行）：`getCore`（加载 WASM）、`gunzip`、`fetch`、`readCache/writeCache/deleteCache/checkCache`。浏览器实现（`src/worker-script/browser/`）：gunzip 用 zlibjs，缓存用 idb-keyval（IndexedDB）；Node 实现（`src/worker-script/node/`）：gunzip 用内置 zlib，缓存直接是 `fs.readFile/writeFile/unlink`——即 Node 下 `.traineddata` 默认缓存在本地磁盘文件。

### 5.3 WASM 核心加载（tesseract.js-core 集成）

`src/worker-script/browser/getCore.js` 用 `wasm-feature-detect` 探测 relaxed-SIMD/SIMD，按"能力 × 是否 LSTM-only"二维组合选择 4 个产物之一（`tesseract-core-relaxedsimd-lstm.wasm.js` 等），默认从 `https://cdn.jsdelivr.net/npm/tesseract.js-core@v7...` 加载，`importScripts` 后挂到 `global.TesseractCore`，并兼容老版本全局名 `TesseractCoreWASM`；Node 版（`src/worker-script/node/getCore.js`）直接 `require('tesseract.js-core/tesseract-core-*')`。加载产物拿到的是 Emscripten 模块工厂，实例化时注入进度回调；此后 C++ 侧通过 `TessModule.FS`（Emscripten 虚拟文件系统）读写训练数据与调试文件，通过 `TessModule.TessBaseAPI` 类完成 Init/Recognize/SetVariable/FindLines/GetGradient/DetectOS（`src/worker-script/index.js` initialize/recognize/detect）。LSTM-only 精简核心在 OEM 为 DEFAULT/LSTM_ONLY 且未指定 `legacyCore` 时启用（`src/createWorker.js` 第 36 行），这是 v5 体积优化的一半来源（另一半是精简语言包）。

### 5.4 语言数据按需加载与缓存自愈

`loadLanguage`（`src/worker-script/index.js` 第 69-200 行）对每个语言：先查缓存 → 未命中则按 `lstmOnly` 决定 URL（`.../@tesseract.js-data/${lang}/4.0.0_best_int` 或 `/4.0.0`，即 LSTM-only 精简包或全量包）→ 下载后靠 gzip 魔数 `1F 8B` 判断是否需解压 → 写入 Emscripten FS → 回写缓存。亮点是初始化失败的自愈逻辑：`api.Init` 返回 -1 时删除可疑缓存，读 Emscripten FS 里的 `/debugDev.txt`，若匹配 "components are not present"（缓存数据缺 Legacy 模型）则重新下载支持该 OEM 的数据并二次 Init（第 262-301 行）——用错误消息文本匹配解决了"缓存无法预知是否含所需模型"的问题。

### 5.5 识别流程与调度器

`recognize`（`src/worker-script/index.js` 第 342-443 行）支持 `rotateAuto` 自动纠偏（先 FindLines 由 C++ 算出倾斜角 GetGradient，角度 ≥0.005rad 才二次 setImage）、rectangle 局部识别、`SaveParameters/RestoreParameters` 保护全局参数，`processOutput` 统计所需输出格式、全部为免识别格式（如仅 layoutBlocks）时跳过 `api.Recognize`。结果由 `src/worker-script/utils/dump.js` 按 text/blocks/hocr/tsv/pdf 等开关组装。`src/createScheduler.js` 实现多 worker 池：`addWorker` 加入空闲队列，`addJob` 入队后逐个空闲 worker 派发，一个 worker 同一时刻只跑一个 job，适合并行 OCR 多图（文档 `docs/workers_vs_schedulers.md` 专门解释何时用哪个）。

## 6. 构建与运行

来自 `package.json` scripts 与 `README.md` Contributing 节：

```bash
npm install          # 安装依赖（postinstall 含 opencollective 提示）
npm start            # 开发服务器 http://localhost:3000/examples/browser/basic-efficient.html，src 变更自动重建
npm run build        # rimraf dist && webpack prod（tesseract.min.js + worker.min.js）&& rollup（ESM）
npm run lint         # eslint src
npm test             # 并行起本地 server + karma 浏览器测试 + mocha Node 测试
npm run test:node    # nyc mocha ./tests/*.test.mjs
```

产物（`scripts/webpack.config.prod.js`）：`dist/tesseract.min.js`（主线程 UMD，全局变量 Tesseract）与 `dist/worker.min.js`（浏览器 worker 脚本）；ESM 版 `dist/tesseract.esm.min.js` 由 rollup 生成（`scripts/rollup.esm.mjs`）。Node.js v7.x 要求 Node ≥16（`README.md`）。用户侧最简用法：`const worker = await createWorker('eng'); const ret = await worker.recognize(image); ret.data.text`（`README.md` 示例）。

## 7. 测试与工程质量

- 测试规模与风格：`tests/` 共 4 个文件 421 行——`recognize.test.mjs`（348 行，主体）、`FS.test.mjs`、`detect.test.mjs`、`scheduler.test.mjs`。全部为真实 OCR 集成测试：加载真实图像资产（`tests/assets/`）跑完整识别流程，断言输出文本与黄金答案逐字相等（如 `TESTOCR_TEXT`、`CHINESE_TEXT`，`tests/constants.mjs`）。同一组测试文件通过 `IS_BROWSER` 标志同时跑在 karma（ChromeHeadless + FirefoxHeadless，`karma.conf.js`）与 mocha/nyc（Node）下，双运行时覆盖是刻意设计。
- CI：`.github/workflows/node.js.yml` 对 push/PR 跑 lint + test，Node 版本矩阵 16/18/20/22/24；另有 CodeQL 安全扫描（`.github/workflows/codeql-analysis.yml`）。
- 质量观察：airbnb ESLint 规范；代码注释里保留了对 C++ 侧参数来源的考据（如 `src/worker-script/index.js` 第 203-211 行列出只能初始化时设置的参数清单，注明来自对 Tesseract 代码库 `[type]_INIT_MEMBER` 的搜索）；`benchmarks/` 内置速度/内存基准脚本。整体工程化程度高，但单元测试几乎没有——全部是端到端断言，靠黄金文本兜底。

## 8. 社区与维护状态

- 活跃度：默认分支最近一次提交 2026-05-17（"Removed regenerator-runtime per #1044"），此前 2025-12-15 集中发布 v7.0.0（含 relaxed SIMD 启用、放弃 Node 14）。2026 年下半年无新提交，属于"发布密集期与间歇期交替"的成熟维护节奏。
- Release 节奏（gh api releases）：v5.0.3（2023-10）→ v5.0.4/5（2023-12/2024-02）→ v5.1.0/1（2024-05/08）→ v6.0.0（2025-01）→ v6.0.1（2025-04）→ v7.0.0（2025-12）。每年一个大版本、期间若干补丁，节奏稳定可预期。
- 贡献者结构：79 位贡献者（gh api Link header），核心开发高度集中于 jeromewu（Jerome Wu，`package.json` contributors 唯一署名，同时也是 tesseract.js-core 与语言数据包的维护者），naptha 为原作者。open issues 61 个（gh api）。
- 资金支持：Open Collective（`package.json` collective 字段、README Financial Contributors 徽章）。
- npm 周下载与 jsDelivr 徽章见 `README.md`（具体数值未在本次核实，标注"未确认"）。

## 9. 亮点与风险

亮点：
1. 架构分层干净：业务逻辑（`src/worker-script/index.js`）与运行时能力（adapter + browser/node 对称目录 + npm browser 字段映射）彻底解耦，一套代码同时服务浏览器与 Node，且 worker 线程隔离不阻塞 UI。
2. 零安装体验做到了极致：WASM 核心、语言包（102 语言）全部按需从 CDN 拉，配合 IndexedDB/文件系统缓存，二次使用零网络开销；LSTM-only 精简核心与精简语言包的"按 OEM 分包"是少见的体积工程实践。
3. 自愈式缓存校验：初始化失败时通过 Emscripten FS 中的调试文件识别"缓存数据缺模型"并自动重新下载（`src/worker-script/index.js` 第 262-301 行），解决了语言包版本/OEM 不匹配的真实痛点。
4. 双运行时同一组集成测试（karma Chrome/Firefox + Node mocha），CI 五版本矩阵 + CodeQL，质量保障与运行时覆盖严格。
5. 文档体系完整：api.md、faq.md、performance.md、workers_vs_schedulers.md、本地安装指南，外加官方示例与基准脚本。

风险/局限：
1. 能力受限于上游 WASM 产物：本项目不改识别模型、不支持 PDF 输入、准确率提升依赖 tesseract.js-core 的编译更新（README Project Scope 明示），功能诉求外溢到 Scribe.js 系项目，长期存在生态分裂压力。
2. 大版本破坏性变更频繁（v4 createWorker 变 async、v5 砍掉 load/loadLanguage 显式调用、v6 默认输出只剩 text），升级成本对存量用户较高，README 需维护 v2→v5 迁移指南侧面印证。
3. 核心开发集中于单一维护者（jeromewu），总线因子低；2026 年以来提交稀疏，响应速度存在不确定性。
4. 默认依赖公共 CDN（jsDelivr）加载核心与语言数据，强离线/内网场景必须自行托管 corePath/langPath；语言包按 URL 命名约定约定 `@tesseract.js-data/<lang>`，CDN 变更即破坏性影响（`src/worker-script/index.js` loadLanguage）。

## 10. 总结评价

tesseract.js 是"把重引擎搬进浏览器"这一技术方向的教科书实现：C++ 的复杂度被封在 WASM 层，JS 层用极小的代码量（业务核心两文件约 800 行）搭起 RPC 协议、双运行时适配、按需数据加载与缓存自愈，让"网页里一行代码 OCR 100+ 语言"从噱头变成日常可用能力。八年维护、每年一个大版本的稳定节奏和完整的文档/示例/基准使其成为浏览器 OCR 事实标准。适合前端与 Node 开发者直接使用；对做 WebAssembly 工程化、worker 线程 RPC 协议设计或按需资源加载实践的人，`src/createWorker.js` + `src/worker-script/index.js` 这对主/辅文件是非常值得精读的样本；需要更高识别精度或 PDF 支持的用户则应转向其 README 推荐的 Scribe.js。
