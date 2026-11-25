# usebruno/bruno 项目分析

> 星数 36500（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：**开源、离线优先、纯文本文件驱动的 API 调试与测试 IDE**，Postman/Insomnia 的轻量替代品，核心卖点是集合以 `.bru` 纯文本文件存放在文件系统里、天然 Git 友好。

| 项 | 值 |
|---|---|
| 名称 | usebruno/bruno（npm 包名 `bruno`，见 `packages/bruno-electron/package.json`） |
| 星数 | 36500（快照）；gh api 实时查询 2026-11-05 已达 48120 |
| 主语言 | JavaScript（前端 React、主进程 Electron/Node） |
| 许可证 | MIT（`license.md`，Copyright 2022 Anoop M D, Anusree P S and Contributors） |
| 最近活跃 | main 分支 push 时间 2026-11-04（gh api）；最新 release v4.3.0 发布于 2026-10-22 |
| 贡献者 | 420 人（gh api contributors，分页计）；open issues 1960，forks 3078 |
| 规模 | packages 下 18 个包（根 `package.json` workspaces 17 个 + `bruno-docs`），非测试源码约 1869 个 js/ts 文件、约 30 万行 |

## 2. 项目是什么，解决什么问题

Bruno 是一个桌面端 API 客户端（HTTP/gRPC/WebSocket/GraphQL 全支持），定位是"探索与测试 API 的 IDE"（`readme.md`）。它要解决的核心痛点是 Postman 这类工具的两个结构性问题：其一，集合数据锁在云端/私有格式里，无法用版本管理工具协作；其二，客户端强制在线、数据主权不在用户手里。

Bruno 的答案是**把集合直接存为文件系统上的目录树**。每个请求是一个 `.bru` 纯文本文件（`packages/bruno-lang/example/request.bru`、`packages/bruno-tests/collection/` 下大量真实样例），环境是 `.env` 风格文件，集合级配置是 `collection.bru`。用户用 Git 或任何 VCS 协作，diff/merge/代码评审走常规工程流程。README 明确宣言："Bruno is offline-only. There are no plans to add cloud-sync to Bruno, ever."（`readme.md` 第 41 行），把离线与数据主权写成了产品宪法。

这个设计决定了整个代码库的形态：一个专门为 `.bru` 格式服务的 PEG 语法与解析器包（`bruno-lang`）、一个文件系统监听层（Electron 主进程里 `chokidar` 驱动的 `collection-watcher.js`、`workspace-watcher.js`）、以及一层文件读写抽象（`bruno-filestore`、`bruno-io`）。文件即数据库，UI 只是文件树的一个投影。

生态位上，Bruno 同时向下覆盖 CLI 与 CI/CD（`@usebruno/cli`，官方 Docker 镜像 `usebruno/cli`，`readme.md` Docker 一节），向上覆盖企业场景（Golden Edition 付费版，见第 9 节）。它把"API 集合即代码"这个理念做到了比 Postman 更彻底的程度——Postman 的 Git 集成是同步机制，Bruno 的文件系统就是唯一事实源。

## 3. 技术栈与依赖

- **前端**（`packages/bruno-app`）：React + Redux Toolkit + formik/yup，代码编辑器 CodeMirror（`codemirror-graphql`），富文本 TipTap，UI 为 Tailwind + Tabler Icons，终端 xterm.js。构建工具是 **rsbuild**（`packages/bruno-app/rsbuild.config.mjs`），不是常见的 webpack 直接配置。
- **桌面壳**（`packages/bruno-electron`）：Electron + electron-builder；网络层基于 axios（叠加 `axios-ntlm`、`aws4-axios`、代理 agent 等，见 `packages/bruno-electron/package.json` dependencies）；文件监听 chokidar；Git 操作 simple-git；内置 mock server 用 express；进程池 workerpool。
- **协议扩展**：gRPC 用 `@grpc/grpc-js` + `@grpc/proto-loader`（`packages/bruno-electron/src/ipc/network/prepare-grpc-request.js`），AWS 凭证用 `@aws-sdk/credential-providers`。
- **AI 能力**：Vercel AI SDK（`ai`、`@ai-sdk/anthropic`、`@ai-sdk/openai`），主进程 `packages/bruno-electron/src/ipc/ai/` 下有 chat/autocomplete/script-prompts 等提示词编排。
- **脚本沙箱**：双实现——Node `vm` 模块（`packages/bruno-js/src/sandbox/node-vm/index.js`）与 QuickJS WASM（`quickjs-emscripten`，`packages/bruno-js/src/sandbox/quickjs/index.js`）。
- **`.bru` 解析**：ohm-js PEG 语法（`packages/bruno-lang/v2/src/bruToJson.js` 内嵌 grammar 字符串）。
- **本地索引**：`bruno-sqlite`（私有 native 包，`packages/bruno-sqlite/package.json`，tsx 脚本生成产物）用于大集合的本地索引/缓存。
- **根工程**：npm workspaces monorepo（根 `package.json`），Jest 单测 + Playwright e2e，ESLint 9 flat config（`eslint.config.js`），husky + nano-staged 钩子。

## 4. 目录结构与代码组织

| 目录/包 | 职责 |
|---|---|
| `packages/bruno-app` | React 前端（v2.0.0），全部 UI、Redux 状态、主题、i18n（20+ 语言 readme 亦多语言） |
| `packages/bruno-electron` | Electron 主进程（v2.0.0）：IPC、网络请求、文件监听、mock server、AI、打包 |
| `packages/bruno-lang` | `.bru` 文本格式解析器（v1/v2 两套语法），ohm-js 实现 |
| `bruno-js` | 脚本/断言/测试运行时与沙箱（`bru`、`req`、`res` API 对象，见 `src/bruno-request.js` 等） |
| `bruno-schema` / `bruno-schema-types` | 集合 JSON Schema 与 TS 类型（ ajv 校验） |
| `bruno-filestore` / `bruno-io` / `bruno-sqlite` | 文件持久化抽象、读写 IO、本地 SQLite 索引 |
| `bruno-requests` | 请求构造：auth（多种）、cookies、grpc、network、ws（`src/auth/`、`src/network/`） |
| `bruno-cli` | CLI（v1.16.0，`@usebruno/cli`），`bru run` 命令 |
| `bruno-converters` | Postman/Insomnia/OpenAPI 等格式互转 |
| `bruno-common` / `bruno-query` / `bruno-toml` / `bruno-graphql-docs` | 公共工具、查询、TOML、GraphQL 文档查看 |
| `bruno-tests` | 自托管 echo 测试服务器与示例集合（含 auth/assert/docker 场景） |
| `tests/` | Playwright e2e（533 个 spec，约 60 个功能域目录） |
| `scripts/` | 构建/开发编排（`build-electron.js`、`dev.js`、`dev-hot-reload.js` 等） |
| `.github/workflows/` | 7 个 CI 工作流（三平台测试、lint、benchmark、flaky 检测、CLI 发布） |

组织方式评述：典型的**业务分层 monorepo**——前端、壳、格式、运行时、持久化、CLI 各自成包，依赖方向清晰（app/electron 依赖 filestore/lang/js/requests，底层包不反向依赖）。大量包以 `@usebruno/*` 私有 scope 管理（0.x 版本号），仅 `bruno`（Electron 应用）和 `@usebruno/cli` 面向 npm 发布（各包 `package.json` 中 version 字段）。

## 5. 核心架构与关键实现

### 5.1 `.bru` 纯文本格式与 ohm-js 语法（bruno-lang）

`.bru` 文件是块结构文本。注释写在 `packages/bruno-lang/v2/src/bruToJson.js`：文件由字典块（headers）、文本块（body:json）、列表块（tags）组成，语法用 ohm-js PEG 描述（grammar 内嵌于源码）：

```
BruFile = (meta | http | grpc | ws | query | params | headers | metadata |
           auths | bodies | varsandassert | script | tests | app | settings |
           docs | example)*
auths = authawsv4 | authbasic | authbearer | authdigest | authNTLM |
        authOAuth1 | authOAuth2 | authwsse | authapikey | authedgegrid | ...
```

一个真实样例（`packages/bruno-tests/collection/auth/bearer/via auth/Bearer Auth 200.bru`）：

```
meta {
  name: Bearer Auth 200
  type: http
  seq: 1
}
get {
  url: {{host}}/api/auth/bearer/protected
  body: none
  auth: bearer
}
auth:bearer { token: {{bearer_auth_token}} }
assert { res.status: 200 }
script:post-response { bru.setEnvVar("foo", "bar"); }
```

v1/v2 双语法并存（`packages/bruno-lang/v1/src/`、`v2/src/`），v2 增加了 grpc/ws/example 等块与注解解析（`v2/src/common/attributes.js`）。JSON↔bru 双向转换（`bruToJson.js`/`jsonToBru.js`）保证了 UI 编辑与文件格式互逆。

### 5.2 脚本沙箱双实现（bruno-js）

用户脚本（pre-request/post-response/tests）在受控环境执行。`packages/bruno-js/src/sandbox/node-vm/index.js` 用 Node `vm` + 自定义 CJS loader（`cjs-loader.js`，限制可 require 的路径根到集合目录）+ 注入的 `bru`/`req`/`res` API；另一实现 `sandbox/quickjs/index.js` 用 quickjs-emscripten WASM 引擎，连 npm 库都以预打包代码注入（`getBundledCode`）。QuickJS 路径提供更强隔离与可回收 runtime（源码中有 runtime 重建/回收计数逻辑）。两套实现共享 `utils/sandbox.js` 的 `wrapScriptInClosure`。

### 5.3 Electron 主进程：文件即数据库 + IPC 网络

主进程入口 `packages/bruno-electron/src/index.js`。三类关键模块：

- **监听**：`src/app/collection-watcher.js`（chokidar 监听集合目录，文件变更回灌 UI，含同名 spec 测试）、`workspace-watcher.js`、`dotenv-watcher.js`——UI 状态完全跟随磁盘。
- **网络 IPC**：`src/ipc/network/` 下 `prepare-request.js`（插值变量、组装 headers/body/auth）、`axios-instance.js`（代理/证书）、grpc/ws 事件编排；请求在主进程发出（`src/ipc/network/index.js` 注册 channel）。
- **进程隔离**：`src/services/runner-exchange/` 与 `workerpool` 把集合运行（Runner）放到独立进程，避免长跑阻塞 UI。

### 5.4 CLI 与 CI 复用同一内核

`packages/bruno-cli` 复用 bruno-lang/js/requests，`bru run [file|folder] --env Local`（`readme.md` CLI 一节）在 CI 中重放与桌面版语义一致的集合——这是"集合即代码"从 IDE 延伸到流水线的闭环。

## 6. 构建与运行

来自根 `package.json` scripts 与 contributing.md：

```sh
npm run setup            # 安装依赖+构建内部包（scripts/setup.js）
npm run dev              # 开发模式（electron + app）
npm run dev:web          # 仅前端
npm run build:electron   # 打包桌面应用（scripts/build-electron.js）
npm run build:electron:mac|win|linux|deb|rpm|snap
npm run test:e2e         # Playwright（default/system-pac/mock-server 项目）
npm run lint             # eslint（4GB 堆）

# CLI
npm install -g @usebruno/cli
bru run folder --env Local

# Docker（CI）
docker run -v $(pwd):/bruno usebruno/cli run
```

## 7. 测试与工程质量

- **单测**：约 602 个 `.spec.ts` + 491 个 `.spec.js`（全仓 find 统计，不含 node_modules），Jest；解析器包语法测试覆盖 v1/v2（`packages/bruno-lang/v1|v2/tests/`），沙箱有独立 spec（`sandbox/node-vm/index.spec.js`）。
- **e2e**：`tests/` 下 533 个 Playwright spec，按 60 个功能域组织（auth、graphql、grpc、mock-server、proxy、ssl、runner、onboarding……），直接驱动 Electron 窗口（`playwright/electron.ts`）；另有 ssl、auth、mock-server 专属 project 与基准测试（`playwright.benchmark.config.ts`、`.github/workflows/benchmarks.yml`）。
- **CI**：7 个工作流（`.github/workflows/`）——三平台分别跑 tests（linux/macos/windows）、lint-checks、benchmarks、flaky-test-detector（专门抓不稳定用例）、npm-bru-cli（CLI 发布）。
- **质量观察**：依赖治理严格——根 `package.json` 的 overrides 大量钉版本（axios 1.21.0、tar 7.5.22 等，明显在主动封堵供应链告警）；eslint-plugin-diff 使 lint 只对变更行生效，降低存量噪音。

## 8. 社区与维护状态

- **活跃度**：极高。gh api（2026-11-05）：main 最近 push 2026-11-04；最新 release v4.3.0（2026-10-22），按 releases 时间线推断月度级发版节奏（v4.x 已到 4.3）；open issues 1960、forks 3078。
- **贡献者**：420 名贡献者（gh api contributors 分页统计），主创 Anoop M D（商标持有者，`readme.md` Trademark 节）与其伙伴（license 版权行为 Anoop M D, Anusree P S）。
- **本地化**：README 翻译 20 种语言（`docs/readme/`）、contributing 翻译 18 种（`docs/contributing/`），社区翻译机制成熟。
- **治理**：`governance.md`、`publishing.md`（新包管理器发布流程）、`security.md`、`CODING_STANDARDS.md` 齐备，属于商业化团队主导 + 开放社区贡献的混合模式。
- 未确认：企业客户的付费规模；Discord 社区规模（README 有邀请链接，仓库内无数据）。

## 9. 亮点与风险

**亮点**

1. **格式即战略**：`.bru` 纯文本 + 文件系统存储（`bruno-lang`、`bruno-filestore`）把"Git 友好"从营销词做成了数据模型本身，配合 CLI/Docker 天然打通 CI/CD——这是与 Postman 差异化的根基。
2. **沙箱双引擎**：node:vm（性能/生态）与 QuickJS WASM（隔离/可回收）双实现并存（`packages/bruno-js/src/sandbox/`），安全边界上有明确的多层设计（路径根校验、closure 包装、预打包库注入）。
3. **测试工程标杆**：1093 个单测 spec + 533 个 Playwright e2e + 三平台 CI + flaky 检测 + 基准测试，桌面应用做到这个测试密度相当少见。
4. **协议覆盖广**：HTTP/gRPC/WS/GraphQL/SSE 全支持（`ipc/network/prepare-grpc-request.js`、`ws-event-handlers.js`），还内置 mock server（`src/app/mock-server/`）与 OpenAPI 同步（`ipc/openapi-sync.js`）。
5. **立场清晰**：承诺永不云同步（README 第 41 行 + long-term vision discussion #269），匿名遥测 ID 显式存 localStorage（GoldenEdition 组件内可见），隐私姿态可审计。

**风险/局限**

1. **规模与复杂度**：约 30 万行、18 包 monorepo，v1/v2 双语法并存增加维护面（`bruno-lang/v1` 与 `v2`）。
2. **商业化与开源边界需持续博弈**：Golden Edition（`packages/bruno-app/src/components/Sidebar/GoldenEdition/index.js` 引导至 usebruno.com/pricing）功能在闭源侧，开源核心与付费能力的界线由公司单方控制；MIT 许可证下也存在 fork 竞争面。
3. **Electron 资源占用**：桌面壳 + chokidar 全目录监听 + SQLite 索引，超大集合下的内存/句柄开销是固有挑战（基准工作流的存在侧面印证）。
4. **issue 积压**：1960 个 open issue（gh api），社区需求消化压力大。

## 10. 总结评价

Bruno 是"本地优先 + 纯文本 + Git 工作流"理念在 API 客户端品类里执行得最彻底的产品：用 ohm-js 自造 `.bru` 格式、用文件系统当数据库、用同一内核贯通 IDE/CLI/Docker 三形态，再用业内少见的 e2e 测试密度托住一个 18 包 Electron monorepo 的工程质量。它证明了"反云"可以同时是产品哲学和商业护城河——开源核心获客，Golden Edition 与团队版变现，430+ 贡献者与月度发版维持高速度。适合把 API 集合纳入代码评审的团队、对数据主权敏感的组织，以及想研究"文件格式驱动桌面应用"架构的开发者读源码（入口推荐 `packages/bruno-lang/v2/src/bruToJson.js` → `packages/bruno-js/src/sandbox/` → `packages/bruno-electron/src/ipc/network/`）。
