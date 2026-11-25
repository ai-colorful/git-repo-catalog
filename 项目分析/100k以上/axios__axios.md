# axios/axios 项目分析

> 星数 109307（快照 2026-10-24）｜主语言 JavaScript｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：基于 Promise 的跨端 HTTP 客户端库，用同一套 API 覆盖浏览器（XHR/Fetch）与 Node.js（http/https/http2），是 JavaScript 生态事实上的 HTTP 请求标准。

| 项 | 值 |
| --- | --- |
| 名称 | axios/axios |
| 星数 | 109307（gh api 实测 2026-10-30） |
| Forks | 11965；Open issues 111 |
| 主语言 | JavaScript（ESM 源码 + TypeScript 类型声明） |
| 许可证 | MIT（`LICENSE`） |
| 版本 | v1.21.0（`package.json`；CHANGELOG 记录发布于 2026-10-21） |
| 默认分支 | v1.x（gh api） |
| 最近 push | 2026-10-28（gh api 实测） |
| 创建时间 | 2014-08-18（gh api） |
| 贡献者 | 约 687 人（contributors API 分页推算，含匿名） |

## 2. 项目是什么，解决什么问题

axios 解决的核心问题是"在浏览器和 Node.js 之间用同一套 API 发 HTTP 请求"（`package.json` 描述："Promise based HTTP client for the browser and node.js"）。2014 年它诞生时浏览器原生只有回调风格的 XHR，Node 端有 request/superagent 但 API 各异；axios 把 XHR 包装成 Promise、提供请求/响应拦截器、自动 JSON 序列化、XSRF 防护和上传下载进度事件，恰好填上了当时的空档，加上早期 Vue/React 教程统一采用，逐步成为事实标准（`README.md`、`docs/`）。

从源码看，今天的 axios 是一个"适配器化的请求管线"：`lib/core/Axios.js` 负责配置合并与拦截器链编排，`lib/core/dispatchRequest.js` 做数据变换与适配器分发，`lib/adapters/` 下三套适配器分别对接 XHR、Fetch、Node http/https/http2。三套适配器共享 `lib/core/settle.js`（按 `validateStatus` 决定 resolve/reject）、`lib/core/AxiosHeaders.js`（大小写无关的头容器）和 `lib/cancel/`（取消机制），以此保证跨端行为一致。

功能面覆盖很宽：拦截器（含同步拦截器与 `runWhen` 条件执行，`lib/core/Axios.js`）、请求取消（AbortSignal 与遗留 CancelToken 双轨，`lib/cancel/`）、自动转换 JSON/FormData/URLSearchParams（`lib/defaults/index.js` 的 `transformRequest/transformResponse`）、超时与进度事件、代理与环境变量代理（`lib/adapters/http.js`）、HTTP/2 会话复用（`lib/helpers/Http2Sessions.js`）、响应流式处理（`lib/helpers/AxiosTransformStream.js`）。Node 适配器还支持 gzip/deflate/brotli/zstd 解压（`lib/adapters/http.js` 中 `ACCEPT_ENCODING` 相关定义）。

API 设计上有几个辨识度很高的细节：实例是"函数 + 属性"的混合体——`lib/axios.js` 的 `createInstance` 把 `Axios.prototype.request` bind 到实例后，再把原型和上下文属性 `utils.extend` 到函数上，于是 `axios(config)`、`axios.get(url)`、`axios.interceptors` 三种用法并存；方法别名自动生成 `get/delete/head/options` 与 `post/put/patch/query` 两组（`lib/core/Axios.js` 底部），并额外派生 `postForm/putForm/patchForm` 快捷方式；还支持 `axios('url')` 的 fetch 式调用。

命名空间安全是该库近期版本的明显主线：v1.21.0 的 CHANGELOG 与代码中出现大量防原型污染处理，如 `utils.getSafeProp`、null 原型描述符（`lib/core/Axios.js` 的 `isHeadersBucket`）、`__proto__` 头名保护、拦截器 ID 注册表（`lib/core/InterceptorManager.js`），说明作为被海量下游依赖的库，它在把"恶意配置对象"当作威胁模型认真对待（仓库还带有 `THREATMODEL.md`）。

## 3. 技术栈与依赖

- 语言与模块体系：纯 JavaScript ESM（`"type": "module"`，`package.json`），类型由手写声明文件提供（`index.d.ts` 804 行 ESM 版、`index.d.cts` CJS 版）。
- 运行时依赖极小（4 个，`package.json`）：
  - `follow-redirects`：Node 端跟随重定向；
  - `form-data`：Node 端 multipart 表单；
  - `https-proxy-agent`：HTTPS 代理 CONNECT 隧道；
  - `proxy-from-env`：读取 `HTTP(S)_PROXY`/`NO_PROXY` 环境变量。
  - HTTP/2、gzip/brotli/zstd 解压、AbortController 等均直接用 Node 内建能力，无额外依赖（`lib/adapters/http.js`）。
- 构建：Rollup（`rollup.config.js`）产出 4 种包——`dist/esm/axios.js`（ESM）、`dist/axios.js`（UMD）、`dist/browser/axios.cjs`、`dist/node/axios.cjs`；gulp 负责清理与版本号注入（`gulpfile.js`、`lib/env/data.js`）。
- 测试：Vitest 4（`vitest.config.js`），浏览器测试用 `@vitest/browser-playwright` 驱动 chromium/firefox/webkit。
- 质量/流程工具：ESLint 10（`eslint.config.js`）、husky + commitlint（conventional commits）+ lint-staged/Prettier、lockfile-lint（锁文件完整性校验）、zizmor（GitHub Actions 安全扫描）、构建可复现性校验（`.github/workflows/verify-build-reproducibility.yml`）。
- 文档站：VitePress（`docs/package.json` 含 vitepress 1.6.4），多语言 docs/es、docs/fr、docs/zh。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `lib/` | 全部运行时源码（ESM），发布到 npm 的核心 |
| `lib/core/` | 管线核心：Axios 类、拦截器管理、请求分发、头容器、配置合并、settle |
| `lib/adapters/` | 三套环境适配器：http.js（Node，1609 行）、xhr.js（384 行）、fetch.js（749 行） |
| `lib/cancel/` | CanceledError、遗留 CancelToken、isCancel |
| `lib/defaults/` | 默认配置、transformRequest/Response、transitional 过渡开关 |
| `lib/helpers/` | 40+ 独立小工具（URL 拼装、进度节流、流式变换、信号组合等） |
| `lib/platform/` | 浏览器/Node 平台抽象（FormData、URLSearchParams、Blob、Buffer 差异） |
| `index.js` / `index.d.ts` / `index.d.cts` | 公共入口与类型声明 |
| `dist/` | Rollup 产物（不入库） |
| `tests/` | unit / browser / module / smoke(cjs/esm/deno/bun) 五层测试 |
| `docs/` | VitePress 文档站（多语言） |
| `examples/`、`sandbox/` | 可运行示例与本地调试沙盒 |
| `.github/workflows/` | 8 个 workflow：CI、发布、bundle 体积、锁文件、可复现构建等 |

组织方式评述：单包（非 monorepo），但通过 package.json 的 `exports` 条件导出 + `browser`/`react-native` 字段做"源码级多端切换"——Node 专属文件在浏览器构建中被映射为 `lib/helpers/null.js`（`package.json` 的 browser 字段），platform 层进一步隔离环境差异。模块划分清晰、文件粒度小（多数 helper 一两百行），`./unsafe/*` 导出把内部模块作为逃生舱口暴露给生态（`package.json`）。

## 5. 核心架构与关键实现

### 5.1 请求管线与拦截器链（lib/core/Axios.js）

`Axios.request()` 是整个库的调度中心：合并配置 → 校验 `transitional` 过渡选项 → 展平头默认值 → 把请求/响应拦截器展开成 `[fulfilled, rejected, ...]` 交替数组，与 `dispatchRequest` 串成一条 Promise 链。亮点是支持 `synchronous` 拦截器：全部请求拦截器都同步时，走 try/catch 同步循环而非 Promise 链，减少微任务开销；`runWhen(config)` 允许拦截器按请求条件跳过（如仅对特定 URL 刷新 token）。代码片段（`lib/core/Axios.js` `_request` 内）：

```js
const chain = [dispatchRequest.bind(this), undefined];
chain.unshift(...requestInterceptorChain);
chain.push(...responseInterceptorChain);
let promise = Promise.resolve(config);
while (i < len) {
  promise = promise.then(chain[i++], chain[i++]);
}
```

另外 `request()` 外层专门重建错误堆栈：捕获后用 `Error.captureStackTrace(dummy)` 取调用方帧拼接回 `err.stack`，并处理 `Caused by:` 分段去重（v1.21.0 改进，`lib/core/Axios.js` 72-123 行），对用户排错非常友好。

### 5.2 适配器体系（lib/adapters/）

`lib/adapters/adapters.js` 维护 `knownAdapters = { http, xhr, fetch }` 注册表并导出 `getAdapter`：按配置顺序逐个尝试，环境不支持（如浏览器里没有 Node http 模块导出）就记入拒绝原因，最终抛出带逐条原因列表的 `ERR_NOT_SUPPORT`。默认顺序 `['xhr', 'http', 'fetch']`（`lib/defaults/index.js`）——浏览器命中 xhr，Node 命中 http，其余环境（Bun/Deno/Workers）落到 fetch。fetch 适配器设计为惰性解析（`fetch: { get: config => ... }`），在真正请求时才探测全局 `fetch` 可用性。

- `http.js`（1609 行）最重：代理体系（`proxy-from-env` 解析环境变量、`https-proxy-agent` 建 CONNECT 隧道、按代理配置缓存 agent 复用 socket 池）、`follow-redirects` 重定向 + `beforeRedirects` 钩子（proxy/auth/sensitiveHeaders/config 四类分流，300 行附近）、HTTP/2 会话池 `Http2Sessions`（按 authority 复用、GOAWAY 后停止复用）、压缩响应解压。
- `xhr.js`（384 行）：浏览器 XSRF cookie 注入（支持异步 Cookie Store）、上传/下载进度经 `progressEventReducer` 节流、请求完成后清理全部监听器与取消订阅。
- `fetch.js`（749 行）：用 `composeSignals`（`lib/helpers/composeSignals.js`）把用户 signal、CancelToken 的 `toAbortSignal()`、超时定时器合成一个 AbortController；`trackStream` 处理流式进度；还处理 data URL 与 Cloudflare Workers 等运行时的 `cache: 'default'` 兼容性。

### 5.3 取消机制（lib/cancel/ + composeSignals）

双轨制：老 API `CancelToken`（`lib/cancel/CancelToken.js`）是"promise + 监听器数组"模式——cancel 时向所有订阅者广播 `CanceledError`，并重写了 `promise.then` 让 `token.promise.then()` 返回带 `.cancel()` 的新 promise（可退订）；新 API 直接用标准 `AbortSignal`（`config.signal`，`lib/core/dispatchRequest.js` 的 `throwIfCancellationRequested` 在发出前与收到响应前各检查一次）。`CanceledError` 继承 `AxiosError` 并带 `__CANCEL__ = true` 标记，`isCancel()` 据此判定，让用户能区分"取消"与"失败"。

### 5.4 数据变换与 settle（lib/core/dispatchRequest.js、transformData.js、settle.js）

请求侧 `transformRequest` 数组按序变换（FormData/流/Blob 直通、URLSearchParams 设头、对象自动 JSON、文件列表自动 multipart），响应侧 `transformResponse` 默认尝试 JSON.parse 但失败回退原文（`silentJSONParsing` 可配，`lib/defaults/index.js`）。`settle.js` 仅 30 行却统一了三个适配器的状态码语义：`validateStatus` 通过则 resolve，否则按 4xx/5xx 归类 `ERR_BAD_REQUEST`/`ERR_BAD_RESPONSE` 并附上完整 response——这是 axios"非 2xx 默认抛错"行为的唯一裁决点。

## 6. 构建与运行

来自 `package.json` scripts 与 `AGENTS.md`（仓库对人类与 AI agent 的统一贡献指南）：

```bash
npm ci                      # 安装（.npmrc 强制 ignore-scripts=true）
npm run build               # gulp clear && rollup -c，产出 dist/ 四种包
npm run lint                # eslint lib/**/*.js
npm run test:vitest:unit    # Node 单测（tests/unit/）
npm run test:vitest:browser:headless   # 浏览器测试（需先 npx playwright install）
npm run test:smoke:deno / test:smoke:bun   # 跨运行时冒烟
npm run docs:dev            # VitePress 文档站
node ./sandbox/server.js    # 本地沙盒服务
```

CI 顺序（`AGENTS.md` 明确记录）：install → lint → build → Playwright 安装 → unit → browser headless → pack → CJS/ESM module 与 smoke → Bun/Deno smoke。

## 7. 测试与工程质量

- 规模：`tests/` 下 158 个测试相关文件；unit + browser 目录中 `it(/test(` 出现约 1306 次，即千余条用例。分层清晰：`tests/unit/`（77 个文件，按 core/adapters/cancel/helpers/utils 划分）、`tests/browser/`（真实浏览器）、`tests/module/`（对 npm pack 产物做 CJS/ESM 引入验证）、`tests/smoke/`（cjs/esm/deno/bun 四运行时冒烟）。
- 测试风格：Vitest，大量用本地 express 服务器 + formidable/multer 模拟真实 HTTP 往返；浏览器测试同一套用例跑 chromium/firefox/webkit 三内核（`vitest.config.js`）。
- 供应链安全姿态突出（`AGENTS.md` "Setup And Safety" 一节）：`ignore-scripts=true` + lockfile-lint 校验 HTTPS 与完整性哈希、Dependabot 更新强制 7 天延迟且包管理类 PR 仅限维护者、Actions 用 zizmor 扫描、构建可复现性校验 workflow、AI 参与 PR 必须打 `ai-assisted` 标签。
- 代码质量观察：源码注释密度高，多为解释"为什么"的防御性注释（如 `lib/core/InterceptorManager.js` 对迭代中 eject 的处理、`lib/core/dispatchRequest.js` 对时钟回拨的 clamp）；几乎每个模块末尾都有对应单测文件。

## 8. 社区与维护状态

- 活跃度：默认分支 v1.x 最近 push 2026-10-28（gh api 实测），v1.21.0 发布于 2026-10-21（`CHANGELOG.md`）；2026 年 1-10 月已发 13 个版本，节奏约每月 1-2 个（CHANGELOG 版本列表），v1.13.4（1 月）→ v1.21.0（10 月）。
- 团队：`package.json` contributors 列出 Matt Zabriskie（作者）、Jay（jasonsaayman）、Dmitriy Mozgovoy（DigitalBrainJS）等 13 位核心维护者；总贡献者约 687（API 分页推算）。
- 治理：有 `COLLABORATOR_GUIDE.md`、`CODEOWNERS`、`CONTRIBUTORS.md`、issue/PR 模板、`SECURITY.md` 与 `THREATMODEL.md`；v1.21.0 CHANGELOG 显示贡献流程已明确纳入 AI 协作规范（AGENTS.md 为唯一权威来源）。
- 商业支持：README 顶部为 OpenCollective 铂金/金牌赞助位（thanks.dev、SAP、Principal 等），并有 HeroDevs 为已 EOL 的 v0.x 提供付费安全续命服务（`README.md`）。
- 未确认：issue 平均处理时长、PR 合并速率等细粒度健康指标（本次仅静态分析，未拉取 issue 数据）。

## 9. 亮点与风险

**亮点**

1. 跨端适配器架构成熟：三套适配器共享 settle/headers/cancel 等核心件，行为对齐是显式工程目标（v1.21.0 专门修复"跨适配器对齐"，`CHANGELOG.md`），`getAdapter` 的失败原因逐条报错设计对排错友好（`lib/adapters/adapters.js`）。
2. 拦截器机制 expressiveness 领先：同步拦截器路径、`runWhen` 条件执行、新旧排序兼容开关（`transitional.legacyInterceptorReqResOrdering`），在保持兼容的同时持续演进（`lib/core/Axios.js`）。
3. 运行时依赖只有 4 个，且 supply-chain 防御在同体量项目里罕见地严格（ignore-scripts、lockfile-lint、7 天依赖延迟、可复现构建）。
4. 测试金字塔完整：千余条用例覆盖四个运行时与三浏览器内核，module/smoke 层直接验证发布产物。
5. 错误体验打磨：AxiosError 携带 config/request/response 全上下文、堆栈重建与 `Caused by:` 分段、错误码体系（`lib/core/AxiosError.js`）。

**风险/局限**

1. 三适配器长期存在行为漂移，CHANGELOG 中反复出现"aligned ... across adapters"类修复，fetch 适配器（v1.7+ 引入）仍在追赶期，Workers/Deno 等环境兼容性修补频繁。
2. minor 版本内含破坏性变更：v1.21.0 改变了 header 默认值的识别规则（CHANGELOG 明确标注 Breaking，需 `transitional` 开关兜底），对锁死在自动更新链路的下游有升级成本；`transitional` 选项本身就是兼容债的累积证据。
3. 遗留 API 双轨维护：CancelToken 与 AbortSignal 并存，CancelToken 仍在主代码路径中被适配器各自订阅/清理，复杂度与测试面翻倍。
4. 单文件体量偏大：`lib/adapters/http.js` 1609 行承载代理/重定向/HTTP2/解压多重职责，边界耦合，贡献者改动的回归风险集中于此（该文件对应的历史安全修复也最多，如 NO_PROXY 相关 DoS 修复）。

## 10. 总结评价

axios 是那种"定义了一个时代的库"：它把 2014 年碎片化的前端 HTTP 实践收敛成一套至今未变的 API，10 万+ star 是生态统治力的直接体现。今天的 axios 不再靠新功能取胜，而是靠三件事维持地位——跨端行为一致性工程、极小的依赖面与供应链防御、以及每周可见的细致维护（v1.21.0 的 CHANGELOG 粒度到"错误堆栈拼接去重"级别）。对使用者，它仍是浏览器与 Node 通用请求的首选默认项，但新项目若只跑现代运行时，原生 fetch + 轻量封装也是合理选择；对源码读者，`lib/core/Axios.js` 的拦截器链编排、`lib/adapters/adapters.js` 的适配器解析、`lib/cancel/` 的双轨取消机制是三处最值得精读的样本，堪称"如何给一个老库做低风险持续演进"的教科书。
