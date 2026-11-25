# HeyPuter/puter 项目分析

> 星数 38500（快照 2026-10-24，分析时 gh api 实测 44092）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-06

## 1. 项目速览

一句话定位：**跑在浏览器里的"互联网操作系统"——一个开源、可自托管的云桌面环境 + 其背后的云存储/AI 后端 + 一套让第三方网页应用直接获得云端能力的 SDK**。

| 项目 | 值 | 出处 |
| --- | --- | --- |
| 名称 | HeyPuter/puter（gh api 确认未迁移组织） | `gh api repos/HeyPuter/puter` |
| 定位 | "Desktop environment in the browser!" | `package.json` description |
| 星数 / Fork | 44092 / 4144（2026-11-06 实测） | gh api |
| 语言 | JavaScript（前端纯 JS）+ TypeScript（后端为主） | 目录统计 |
| 许可证 | AGPL-3.0（`package.json` 标 `AGPL-3.0-only`） | `LICENSE.txt`、gh api |
| 创建时间 | 2024-03-03 | gh api |
| 最近 push | 2026-11-05（分析前一天，极活跃） | gh api |
| 最新 release | 26.11（2026-11-02，日历版本号月更） | gh api releases |
| 贡献者 | 约 434 人（contributors 分页 last=434 推算） | gh api |
| 规模 | 后端 TS 约 49.5 万行、前端 GUI 约 13.4 万行、puter.js SDK 约 7.3 万行 | find+cat 统计 |

## 2. 项目是什么，解决什么问题

Puter 自称 "The Internet OS / Open-Source Internet Computer"（`README.md`），要做的是把"个人云电脑"整体搬进浏览器：桌面、窗口、任务栏、文件管理器、应用商店，以及文件存储、键值数据库、AI 调用、静态网站托管等操作系统级服务。对普通用户，它是开箱即用的网页桌面；对开发者，它是"写一个 HTML 页面就能获得后端"的云平台（`README.md` For Developers 一节）。

整个系统由三部分组成（均在同一 monorepo）：浏览器端桌面环境（`src/gui`）、Node.js 后端（`src/backend`，提供 FS/KV/AI/应用等 API）、以及 `puter.js` 前端 SDK（`src/puter-js`）。第三方应用以 iframe 网页形式被桌面"启动"，加载 SDK 后即可调用用户授权的云能力（`src/gui/src/helpers/launchApp.js` 的 iframe_url 构造）。

商业模式的根基是 "User-Pays Model"（`src/docs/src/user-pays-model.md`）：开发者零基础设施成本——应用没有自己的 API key 和服务器，每个最终用户用自己的 Puter 账号支付其所消耗的存储/AI 用量，每人每月有免费额度，超出后引导升级订阅。这解释了"免费托管"策略：免费额度获客，订阅（puter.com/dashboard#billing）变现，开源仓库则保证自托管与生态信任。

自托管是一等公民：`install.sh` 一行脚本 + `docker-compose.yml`（`doc/self-hosting.md`）把 Puter、MariaDB、Valkey、DynamoDB-local、RustFS（S3 兼容）、Caddy、可选 Ollama（AI profile）全部编排起来。仓库同时维护 `AGENTS.md`（面向 AI 编码代理的编码规范）与 `BUG-BOUNTY.md`，工程与安全治理成熟度明显高于同类社区项目。

## 3. 技术栈与依赖

- **前端 GUI**：纯 JavaScript（无 Vue/React 框架），UI 库为 jQuery 3.6.1 + jQuery UI（`src/gui/puter-gui.json` lib_paths），构建用 webpack + esbuild + clean-css（根 `package.json` devDependencies）。
- **后端**：Node.js ≥24（engines 字段），TypeScript + Express + helmet + compression + cookie-parser（`src/backend/server.ts` import 区），实时通信用 socket.io（`puter-gui.json`）。
- **存储**：SQL 三选一——sqlite（默认）/ postgres / mysql（`src/backend/clients/database/index.ts` 第 42 行）；文件 blob 走 S3 协议（`stores/fs/S3ObjectStore.ts`，AWS SDK）；KV 用 DynamoDB 客户端（`src/backend/clients/dynamodb/`）；缓存/限流用 Redis/Valkey（`src/backend/clients/redis/`）；图片处理用 sharp。
- **Serverless Workers**：`miniflare`（Cloudflare workerd 本地仿真）实现自托管版 Workers（`src/backend/services/localworker/LocalWorkerService.ts` 第 1 行）。
- **测试**：vitest（含 coverage-v8 与 bench 配置）+ Playwright（e2e）+ husky/eslint/prettier/ts-eslint（根 `package.json`）。
- **接入层**：Caddy 反代（`caddy/`），npm workspaces 管理 `src/*` 七个子包（根 `package.json` workspaces）。
- 备注：根目录有 `rust-toolchain.toml`（nightly + wasm32 target），但仓内无 `.rs` 文件（find 统计 0 个），推测为 wasm 依赖构建预留，未确认具体用途。

## 4. 目录结构与代码组织

npm workspaces monorepo，`src/` 下按子系统分包：

| 目录 | 职责 |
| --- | --- |
| `src/backend` | Node 后端全部：controllers/drivers/services/stores/clients 分层（约 49.5 万行 TS，含 492 个测试文件） |
| `src/gui` | 浏览器桌面环境：窗口管理器、桌面、任务栏、i18n（44 种语言翻译文件 `src/gui/src/i18n/translations/`） |
| `src/puter-js` | 面向第三方开发者的 SDK `@heyputer/puter.js` v2.6.5（`src/puter-js/package.json`） |
| `src/worker` | Serverless Worker 的 preamble 构建器（`@heyputer/worker`，配合 miniflare） |
| `src/dev-center` | 开发者中心内置应用（`config.default.json` builtin_apps 指向） |
| `src/cli` | Puter CLI |
| `src/docs` | docs.puter.com 文档站源码（含 `user-pays-model.md` 等关键策略文档） |
| `src/mcp-connector` | MCP 连接器（AI 工具协议接入） |
| `extensions/` | 后端可选扩展：metering、thumbnails、whoami、serverInfo、workerSandbox 等 9 个 |
| `doc/` | 仓库级设计文档：`architecture.md`、`self-hosting.md`、`pagination.md` 等 |
| `caddy/`、`tools/`、`docker-compose.yml` | 反代配置、构建/启动脚本、自托管编排 |

组织方式评述：典型的"单仓多包"monorepo，前后端 + SDK + 文档同仓同版本演进；后端内部严格分层，前端刻意保持无框架的独立技术栈。`AGENTS.md` 是全仓编码规范的单一事实源，甚至写明"后端新代码用 TS，GUI/puter.js 保持纯 JS 不引入 TS"。

## 5. 核心架构与关键实现

### 5.1 后端分层架构（Controllers → Drivers → Services → Stores → Clients）

`doc/architecture.md` 完整给出了分层图与规则：每层只依赖下层，由 `PuterServer`（`src/backend/server.ts`，1853 行）按序实例化并注入；请求级状态用 `AsyncLocalStorage` 封装的 `Context`（`src/backend/core/context.ts`）传递；鉴权主体抽象为 `Actor`（`src/backend/core/actor.ts`），区分 `app`（直接持 token 的应用）与 `effectiveApp`（实际生效应用），是全部权限作用域的基础。Drivers 层把 `puter-kvstore`、`puter-chat-completion` 等能力以 RPC 形式暴露在 `/drivers/*`（`src/backend/drivers/` 下含 ai-chat/ai-image/ai-tts/ai-video 等 7 类 AI 驱动与 kv/subdomain 等）。扩展系统允许第三方在任意层注册（`extension.registerService/...`，`src/backend/extensions.ts`）。

```ts
// doc/architecture.md 引用 extensions API（src/backend/extensions.ts）
const services = extension.import('service');
extension.get('/healthcheck/deep', { subdomain: 'api', adminOnly: true },
    async (_req, res) => { res.json(await services.health.getStatus()); });
extension.on('puter.signup.success', (_key, data) => { /* ... */ });
```

### 5.2 前端桌面环境与"应用=iframe"模型

窗口管理器核心是手写的 `UIWindow.js`（5667 行）+ `UIDesktop.js`（2633 行）+ `UITaskbar.js`（769 行），全部位于 `src/gui/src/UI/`。应用生命周期由 `ProcessService`/`ExecService`（`src/gui/src/services/`）管理，桌面进程树以 NULL UUID 的 InitProcess 为根（`ProcessService.js` 第 16-27 行）。应用即网页：`launchApp.js` 第 458-478 行按应用元数据拼 iframe URL——默认 `https://{name}.{app_domain}/index.html`（子域名路由），内置应用走 `https://builtins.namespaces.puter.com/` 前缀重写为 `${gui_origin}/builtin/<name>`（`src/backend/controllers/static/StaticAssetsController.ts` 第 150-156 行）。`godmode` 应用可获得全权限 token 并自动续期（`launchApp.js` 第 513-521 行）。SDK 在 iframe 内注入后，第三方页面就能调用 `puter.fs/kv/ai/...`。

```js
// src/gui/src/helpers/launchApp.js:465-471（应用=iframe 的三种来源）
if ( ! app_info.index_url ) {
    iframe_url = new URL(`https://${options.name}.${window.app_domain}/index.html`);
} else if ( app_info.index_url.startsWith(BUILTIN_PREFIX) ) {
    const name = app_info.index_url.slice(BUILTIN_PREFIX.length);
    iframe_url = new URL(`${window.gui_origin}/builtin/${name}`);
} else { iframe_url = new URL(app_info.index_url); }
```

### 5.3 puter.js SDK 与 User-Pays 免费策略

SDK 单包 245 个源文件、约 7.3 万行，模块化暴露 `puter.ai / puter.fs / puter.kv / puter.hosting / puter.apps / puter.auth / puter.os / puter.workers / puter.net / puter.peer`（`src/puter-js/src/modules/` 目录列表，`src/puter-js/src/index.js` import 区）。它支持浏览器全局注入（`<script src="https://js.puter.com/v2/">`）、ESM/CJS、Node token 初始化、以及 workerd 环境（测试 runners 覆盖 node/browser/workerd 三种，根 `package.json` test:puterjs:*）。免费策略的产品化落地在 `src/docs/src/user-pays-model.md`：用户侧每人每月免费额度，开发者侧零账单；`puter.ai` 的 AI 能力由平台统一计费代理（后端 `drivers/ai-*`），配合 `extensions/metering.ts` 计量。这使它成为"AI 生成应用"场景的热门后端（`src/puter-js/README.md` 明确点名 Claude Code、Lovable、Replit 等）。

### 5.4 虚拟文件系统（VFS）与存储设计

文件系统核心 `FSService`（`src/backend/services/fs/FSService.ts`，5700 行）实现元数据与内容分离：目录树/条目元数据存 SQL（`stores/fs/FSEntryStore.ts`），文件内容写 S3 兼容对象存储（`stores/fs/S3ObjectStore.ts`）。上传链路相当工程化：上传配额预检与预留（`#reserveUploadStorage`/`#settleUploadReservations`，FSService.ts 第 1162-1337 行）、分片 multipart 签名直传（`signMultipartParts`，第 2765 行）、URL 写入（`startUrlWrite`）、sha256 校验流（`#createCountingStream`，第 1425 行）、批写入两阶段提交（`prepareBatchWrites`/`finalizePreparedBatchWrites`）。权限通过 permissions 系统的 rewriter/implicator/exploder 钩子挂在 FS 路径上（FSService.ts 第 333-554 行）。`services/fs/homeRegion.ts` 表明支持按用户 home 分区（多区域数据本地性）。trash 隔离、应用卸载数据清理均有专门测试（`FSService.trashIsolation.test.ts` 等）。

## 6. 构建与运行

来自根 `package.json` scripts 与 `README.md`/`doc/self-hosting.md`：

```bash
# 本地开发（Node >= 24）
npm install
npm start                     # 启动于 http://puter.localhost:4100（tools/start.mjs）

# 构建
npm run build                 # setupExtensions + tsc + gui build.js + puter-js build
npm run typecheck             # tools/typecheck.mjs

# 测试
npm run test:backend          # vitest（src/backend/vitest.config.ts）
npm run test:backend:postgres # 换 postgres 引擎跑
npm run test:puterjs          # SDK 测试（node/browser/workerd 三 runners）
npm run test:puterjs:coverage # SDK 覆盖率报告（tools/puterjsCoverageReport.mjs）

# 自托管
curl -fsSL https://puter.com/selfhost | sh    # 一行安装
docker compose up -d                          # 或手动编排（MariaDB+Valkey+DynamoDB+RustFS+Caddy）
```

自托管要求泛域名 DNS（`*.your-domain.com`），因 Puter 按子域名路由 api/site/app（`doc/self-hosting.md`）。

## 7. 测试与工程质量

- **规模**：后端 492 个 `.test.ts` 对 458 个源 `.ts`（测试文件数超过源文件数）；GUI 120 个 `.test.js`；SDK 另有 Playwright e2e（`src/puter-js/playwright.config.js`）与覆盖率工具链。均为本仓实数统计。
- **风格**：vitest 单元/集成为主，含基准测试配置（`src/backend/vitest.bench.config.ts`）；支持按数据库引擎（sqlite/postgres）切换的集成测试（`PostgresDatabaseClient.integration.test.ts`）。
- **CI**：11 个 workflow（`.github/workflows/`）——backend-tests、puterjs-tests、ai-provider-integration-tests、docker-image、release-please、notify-prod 等，覆盖测试、镜像发布与月度 release 流水线。
- **质量治理**：`AGENTS.md` 用整章规范注释、命名、安全扫描（禁止日志泄密、禁止 ticket 引用、no abstraction without a second caller 等）；husky pre-commit；eslint9 + prettier + TS 严格类型；`BUG-BOUNTY.md`/`SECURITY.md` 安全响应流程。工程化程度在 4 万星项目中属第一梯队。

## 8. 社区与维护状态

- **活跃度**：最近 push 2026-11-05（分析前一天），最新 release 26.11 发布于 2026-11-02；历史 release 26.09/26.10/26.10.1/26.11（gh api）呈月更节奏。
- **贡献者**：分页推算约 434 人（含匿名），头部贡献者 2132 次提交（gh api contributors）；2024-03 创建至今持续扩张。
- **商业化与治理**：公司化运作（Puter Technologies Inc.，`package.json` author），附带 AI 供应商集成测试、issue 模板、dependabot automerge、release 自动 PR（`.github/workflows/main.yml`）。
- 未确认项：hosted 版与开源版的私有差异（计费/风控等显然不在仓内，`SuggestedAppsService.ts` 中 editor/code 等一等应用也是外链 URL 而非仓内代码）。

## 9. 亮点与风险

**亮点**
1. 罕见的完整"浏览器操作系统"实现：窗口管理器 + 进程模型 + VFS + 应用商店 + SDK 全栈自洽，且前后端边界清晰（`doc/architecture.md`、`src/gui/src/helpers/launchApp.js`）。
2. User-Pays 模式把"免费云能力"变成可持续商业模式，开发者接入成本趋近于零，完美契合 AI 生成应用浪潮（`src/docs/src/user-pays-model.md`）。
3. 测试文化突出：后端测试文件多于源文件、SDK 三运行时覆盖、AI 供应商集成测试与基准测试齐备。
4. 自托管一等公民：单机 docker compose 拉起全套依赖，含本地 AI（Ollama profile）与 miniflare 版 Serverless Workers。
5. 面向 AI 协作时代写规范：`AGENTS.md` 明确到命名/注释/抽象规则，本身就是一份高质量工程文档。

**风险/局限**
1. AGPL-3.0 对商用集成与二次分发有较强传染性约束，企业内自托管需法务评估（`LICENSE.txt`）。
2. 代码量巨大（约 75 万行）且前端为自研无框架技术栈（jQuery + 手写 UI），外部贡献者与二次开发的上手门槛高。
3. 一等应用（editor、terminal、code 等）不在仓内，完整体验依赖官方托管的外部站点，纯离线自托管时桌面生态受限（`SuggestedAppsService.ts` 仅返回名字，`launchApp.js` 按 URL 加载）。
4. 规模化部署需要多组件协同（MariaDB/Valkey/DynamoDB/S3），单机之外的水平扩展与运维文档相对薄弱（`doc/self-hosting.md` 聚焦单主机）。

## 10. 总结评价

Puter 是把"云操作系统"从口号做成产品的少数项目：浏览器端手写桌面环境、Node 端分层清晰的云后端、以及以 User-Pays 商业模式串联三者的 puter.js SDK，构成一个产品逻辑、技术架构与商业模式互相咬合的闭环。它的源码适合三类读者：想研究"Web 桌面/应用沙箱（iframe + 子域名 + token 注入）"的前端工程师（读 `src/gui`）、想学习大规模 Node 后端分层与权限设计的后端工程师（读 `src/backend` 与 `doc/architecture.md`）、以及想复用"零基建免费云能力"叙事的独立开发者（直接用 `src/puter-js`）。对企业用户，AGPL 与自托管复杂度是主要门槛；对社区而言，它 2024 年创建即冲上 4 万+星且保持月更，是近年增长最快、工程化最完整的明星项目之一。
