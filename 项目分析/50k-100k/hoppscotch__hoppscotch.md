# hoppscotch/hoppscotch 项目分析

> 星数 80678（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：Hoppscotch 是一个开源 API 开发生态（前身为 Postwoman），提供 Web、桌面端、CLI、浏览器扩展多形态客户端，覆盖 REST/WebSocket/SSE/Socket.IO/MQTT/GraphQL 全协议的 API 调试、测试与团队协作。

| 项目 | 内容 |
| --- | --- |
| 名称 | hoppscotch/hoppscotch（Hoppscotch，原 Postwoman） |
| 星数 | 80,711（2026-10-30 gh api 实测；快照 80,678） |
| 语言 | TypeScript（Vue 3 前端 + NestJS 后端 + Rust 引擎） |
| 许可证 | MIT（根 `LICENSE`） |
| 创建时间 | 2019-08-21（gh api） |
| 最近 push | 2026-10-30（gh api，分析前一天） |
| 最新 release | 2026.10.0，发布于 2026-10-30（日期化版本号） |
| 贡献者 | 357+（contributors API 分页页码推断） |
| 仓库规模 | 2,500 文件：.ts 1116 / .vue 407 / .graphql 229 / .rs 108（解压实测） |

## 2. 项目是什么，解决什么问题

Hoppscotch 定位为 Postman 的开源替代品，是"API 开发生态"而非单一调试工具（`README.md`）。核心问题域：开发者需要一个轻量、快速、跨平台的工具来构造和发送 HTTP 请求、调试实时协议（WebSocket/SSE/Socket.IO/MQTT）、编写 GraphQL 查询，并对请求做前置/后置脚本处理与环境变量管理。

产品形态覆盖四个客户端（`README.md`、`packages/` 目录）：

- **Web**（hoppscotch.io）：PWA，可安装、离线可用，云端账号同步（GitHub/Google/Microsoft/Email 登录，SSO 为企业版特性）。
- **Desktop**：Tauri v2 桌面应用（Windows/Linux/macOS），可绕过浏览器限制，支持 Cookie、客户端证书等能力。
- **CLI**（`@hoppscotch/cli`）：在 CI 环境运行 Hoppscotch 测试脚本（`packages/hoppscotch-cli/package.json`）。
- **Agent**：本地系统服务（端口 9119），为 Web 端提供本地网络能力——自定义证书、代理、本地地址访问，通过 OTP 注册建立加密通道（`packages/hoppscotch-agent/README.md`）。

团队协作与自托管是它区别于纯工具类竞品的关键：后端提供团队空间、共享集合、基于角色的访问控制、管理后台（`packages/hoppscotch-backend/src/`），并官方支持 Docker Compose 自托管（根 `docker-compose.yml`，含 all-in-one、无数据库、仅后端、仅管理台等多种 profile）。企业版功能（SSO 等）以 `[^EE]` 脚注标注，体现开源核心 + 商业增值的模式。

## 3. 技术栈与依赖

- **前端应用**：Vue 3（3.5.x）+ Vite 7（`packages/hoppscotch-common/package.json`：`"vite": "7.3.2"`、`"vue": "3.5.41"`）。注意：虽常被称"Nuxt 项目"，当前代码已无 Nuxt 依赖（grep 各包 package.json 无结果），是纯 Vite SPA；早期 Nuxt 历史仅见于社区印象。
- **函数式风格**：fp-ts + rxjs 贯穿请求引擎（`packages/hoppscotch-common/src/helpers/network.ts` 大量 `TaskEither`/`Observable`）。
- **依赖注入**：自研轻量 DI 库 `dioc`（`packages/hoppscotch-common/src/modules/dioc.ts`、服务目录 `src/services/`）。
- **UI**：自研组件库 `@hoppscotch/ui`、Tailwind CSS、vue-i18n（国际化，`TRANSLATIONS.md`）、CodeMirror 6（含自研 `codemirror-lang-graphql` 包）。
- **后端**：NestJS 11 + Apollo GraphQL + Prisma 7 + PostgreSQL（`packages/hoppscotch-backend/package.json`、`src/app.module.ts`），JWT/Passport 认证、Throttler 限流、argon2/bcrypt。
- **网络引擎**：Rust —— `hoppscotch-relay` 基于 curl-rust 定制分支 + 静态 OpenSSL（`packages/hoppscotch-relay/README.md`、`src/relay.rs`），桌面端与 Agent 共用。
- **桌面**：Tauri 2 + tauri-plugin-shell/store/dialog/fs（`packages/hoppscotch-desktop/src-tauri/Cargo.toml`）。
- **脚本沙箱**：`@hoppscotch/js-sandbox`，提供 web/node 双实现，兼容 `pm`（Postman）命名空间与自有 `hopp` 命名空间（`packages/hoppscotch-js-sandbox/src/__tests__/`）。
- **工程化**：pnpm 10 workspace + husky + commitlint（conventional commits）+ Vitest + GitHub Actions。

## 4. 目录结构与代码组织

pnpm monorepo（根 `pnpm-workspace.yaml`、`package.json` workspaces），12 个子包按"平台外壳 + 共享内核"划分：

| 目录 | 职责 |
| --- | --- |
| `packages/hoppscotch-common` | 应用共享核心：全部页面/组件/服务/请求引擎，平台差异通过 `src/platform/` 抽象注入 |
| `packages/hoppscotch-kernel` | 跨平台运行时内核（IO/存储/Relay 调用的类型化 SDK，API 带 v1 版本号，`src/io/index.ts`） |
| `packages/hoppscotch-relay` | Rust 高性能 HTTP 请求引擎（curl + OpenSSL），供桌面/Agent 使用 |
| `packages/hoppscotch-desktop` | Tauri v2 桌面客户端（Vue 前端 + `src-tauri/` Rust 壳） |
| `packages/hoppscotch-agent` | Tauri v2 本地 Agent（端口 9119，Web 端的本地网络代理） |
| `packages/hoppscotch-backend` | NestJS 后端：用户/团队/集合/环境/管理台/发布文档等 GraphQL API |
| `packages/hoppscotch-selfhost-web` | 自托管 Web 前端构建产物 |
| `packages/hoppscotch-sh-admin` | 自托管管理后台 |
| `packages/hoppscotch-cli` | CI 用 CLI（`hopp test` 等，`src/commands/`） |
| `packages/hoppscotch-js-sandbox` | pre-request/test 脚本沙箱（web/node 双实现） |
| `packages/hoppscotch-data` | 公共数据结构与校验/迁移（`src/`：collection、environment、rest、graphql…） |
| `packages/codemirror-lang-graphql` | CodeMirror GraphQL 语言支持 |

组织评述：典型的"核心共享 + 多平台壳"monorepo。`hoppscotch-common` 是绝对主体（组件、服务、helpers、newstore），各平台壳通过 `platform` 抽象与 `kernel` SDK 接入；Rust 仅出现在需要原生能力的两处（relay、tauri 壳），边界干净。根目录的 `docker-compose.yml`、`prod.Dockerfile`、`aio_run.mjs`、Caddyfile 等自托管编排文件与代码并列，说明自托管是一等公民。

## 5. 核心架构与关键实现

### 5.1 平台抽象层（platform definition）

`packages/hoppscotch-common/src/platform/index.ts` 定义 `PlatformDef`，把认证、分析、内核 IO、拦截器、实例信息、同步等十余个能力面声明为接口，各平台（web/desktop/selfhost/cli）注入各自实现，并可用 `platformFeatureFlags` 声明能力开关（如 `cookiesEnabled`、自托管用 Cookie 认证而云/桌面用 Bearer token——见该文件注释）。这是整个多端复用的枢纽：一套 UI 与请求逻辑，通过此层适配不同宿主。

### 5.2 请求引擎（RequestRunner + network + kernel interceptor）

请求主流程分三层（`packages/hoppscotch-common/src/helpers/`）：

- `RequestRunner.ts`（1256 行）：编排层——合并环境变量/集合变量、跑 pre-request 沙箱脚本、发网络请求、跑测试脚本、迭代数据驱动（`runner/`、`utils/EffectiveURL.ts`）。用 fp-ts Either 管道处理可失败步骤。
- `network.ts`：把规范化请求交给 `KernelInterceptorService`，返回 `Observable<HoppRESTResponse>` 流（loading → success/failure），并支持 `noCookieJar` 选项隔离嵌入场景的 Cookie。

```ts
// packages/hoppscotch-common/src/helpers/network.ts
const execResult = RESTRequest.toRequest(req).then((kernelRequest) => {
  if (kernelRequest && streamOptions?.noCookieJar) {
    kernelRequest.meta = {
      ...kernelRequest.meta,
      options: { ...kernelRequest.meta?.options, cookies: false },
    }
  }
  ...
  return service.execute(kernelRequest)
})
const service = getService(KernelInterceptorService)
```

- `services/kernel-interceptor.service.ts`：可插拔拦截器注册表。每个拦截器（浏览器 fetch/官方代理/Agent/桌面 Relay）声明 id、能力（`RelayCapabilities`）与 `execute()`，UI 可据此展示为何某拦截器不可用（`SelectableStatus` 的 unselectable reason）。Web 端发不了本地请求时引导装 Agent，桌面端走 Rust Relay——同一份请求代码按平台自动选择通道。

### 5.3 Rust Relay 引擎与 Kernel SDK

`packages/hoppscotch-relay/src/relay.rs` 用 curl `Easy` 句柄逐项配置请求（method、URL、证书、代理），带 `CancellationToken` 取消与进度回调；README 说明使用 curl-rust 定制分支 + 静态 OpenSSL 保证各平台 TLS 行为一致。能力覆盖客户端证书（PEM/PKCS#12）、自定义 CA、HTTP 代理（含 NTLM）。TS 侧通过 `@hoppscotch/kernel`（版本化 SDK，`src/io/index.ts` 导出 `v1` 并提供 `latest`）和 `interop.rs` 定义的 `RequestWithMetadata/ResponseWithMetadata` 边界序列化通信。

### 5.4 脚本沙箱（Postman 兼容）

`packages/hoppscotch-js-sandbox/src/` 分 `web/` 与 `node/` 两个运行时实现 + `bootstrap-code/` 启动代码，测试目录显示同时支持 `pm`（Postman 脚本兼容层，16 个 `pm-namespace` 测试）、`pw`、`hopp` 三套命名空间。这使 Hoppscotch 可以直接导入并运行 Postman 集合的测试脚本，是生态迁移的关键兼容策略（测试文件如 `__tests__/pm-namespace/`）。

### 5.5 后端（NestJS + GraphQL）

`packages/hoppscotch-backend/src/app.module.ts` 装配约 20 个业务模块：user/team/team-collection/team-environments/team-request/team-invitation/admin/shortcode/infra-config/mailer/posthog/health/access-token/pubsub/mock-server/published-docs 等；GraphQL 用 Apollo driver + 自动 schema + 订阅支持（websocket 鉴权从 cookie/header 提取 token），并自实现 `GQLComplexityPlugin` 做查询复杂度限制。

## 6. 构建与运行

来自根 `package.json`（pnpm 10 必需，`preinstall: npx only-allow pnpm`）：

```bash
pnpm install            # 安装全部 workspace 依赖
pnpm dev                # 递归执行各包 do-dev（并行起各端开发服务）
pnpm generate           # 递归 do-build-prod 全量构建
pnpm test               # 递归 do-test（Vitest）
pnpm lint / pnpm typecheck
pnpm gen-gql            # 生成后端 GraphQL schema 到 gql-gen/backend-schema.gql
pnpm start              # http-server 起自托管 web 构建产物（端口 3000）
```

自托管（根 `docker-compose.yml` 注释）：

```bash
docker compose --profile default up         # 全家桶 + Postgres + 自动迁移
docker compose --profile default-no-db up   # 外接数据库
docker compose --profile just-backend up    # 仅后端
```

CI 测试前需 `.env`（`mv .env.example .env`），并注入 `DATABASE_URL` 与 `DATA_ENCRYPTION_KEY`（`.github/workflows/tests.yml`）。

## 7. 测试与工程质量

- 测试规模：221 个 `*.spec.ts`/`*.test.ts` 文件（find 实测），集中在 `hoppscotch-common/src/services/__tests__`（20）、`js-sandbox/__tests__`（pm 16 + combined 14 等）、`cli/__tests__/unit`（10）、helpers 与 auth 类型（各 10+）。
- 框架：Vitest（`vitest.config.mts`、`setupTests.ts`）；偏单元测试 + 沙箱行为测试，未见端到端测试目录（未确认是否在别处）。
- CI（`.github/workflows/tests.yml`）：push/PR 到 main/next/patch 触发，Node 22 矩阵（注释明确因 Node 24 有测试失败而钉住版本），pnpm install + `pnpm test`。另有桌面构建（`build-hoppscotch-desktop.yml`）、Agent 构建、Docker 发布三条 workflow（`.github/workflows/`）。
- 质量观察：commitlint 强约束 conventional commits；根 `package.json` 维护了 40+ 条 pnpm overrides 钉住传递依赖的安全版本，表明有依赖安全巡检习惯；前后端 typecheck/lint 全接入 pre-commit（husky + lint-staged）。

## 8. 社区与维护状态

- 活跃度：仓库 2026-10-30 仍有 push，最新 release `2026.10.0` 发布于 2026-10-30（gh api），采用 `YYYY.MM.0` 日期化版本（各子包 version 同步为 2026.10.0），发布节奏约为月度。
- 贡献者：contributors API 第 1 页返回 `page=357` 为末页（per_page=1），即 350+ 贡献者；README 挂 OpenCollective 赞助与 contributor 墙。
- 社区渠道：Discord/Telegram/GitHub Discussions（`README.md` 徽章）；翻译走 `TRANSLATIONS.md` 流程（Crowdin 类众包，未确认具体平台）。
- 公司化：Hoppscotch 团队（support@hoppscotch.io）主导开发，企业版 SSO/高级能力商业化为支撑（README `[^EE]` 脚注）；周边项目 `proxyscotch`、`hoppscotch-extension` 在同组织下独立仓库。
- Open issues 885（gh api），对 80k 星项目属于正常水位。

## 9. 亮点与风险

亮点：

1. **平台抽象设计成熟**：`platform/` + `@hoppscotch/kernel` 版本化 SDK + 可插拔 kernel interceptor，一套代码适配 Web/桌面/CLI/Agent 四端，多端架构在同类项目中少见地干净（`packages/hoppscotch-common/src/platform/index.ts`）。
2. **原生性能与浏览器限制的务实解法**：Rust Relay（curl + 静态 OpenSSL）+ 本地 Agent（OTP 注册加密通道）组合，让 Web 应用获得证书、代理、本地访问等原生能力（`packages/hoppscotch-relay/`、`packages/hoppscotch-agent/README.md`）。
3. **Postman 兼容沙箱**：`pm` 命名空间兼容层大幅降低迁移成本，配合 CLI 打通 CI 场景（`packages/hoppscotch-js-sandbox/`、`packages/hoppscotch-cli/`）。
4. **自托管一等公民**：docker-compose 多 profile、AIO 镜像、Caddy 子路径/多端口样例、独立管理后台，部署路径完整（根 `docker-compose.yml`、`aio_run.mjs`、`prod.Dockerfile`）。
5. **工程纪律**：fp-ts 类型化错误处理、DI 服务化、commitlint、依赖安全 overrides、月度日期化 release。

风险/局限：

1. **复杂度门槛高**：12 包 monorepo + fp-ts + 自研 DI/kernel 平台抽象，新人上手曲线陡峭；`RequestRunner.ts` 单文件 1256 行，编排逻辑集中。
2. **TS 侧测试偏薄**：221 个测试文件相对 1500+ TS/Vue 源文件覆盖率有限，主要保障 services/sandbox/CLI，UI 层与请求编排层大量逻辑无直接测试（基于目录对比推断）。
3. **开源核心与商业边界**：SSO、部分高级能力标注为企业版，自托管完整体验依赖官方闭源组件与否需自行评估（`README.md` `[^EE]`）。
4. **Node 版本兼容遗留问题**：CI 注释承认 Node 24 存在测试失败而钉在 Node 22（`.github/workflows/tests.yml`）。

## 10. 总结评价

Hoppscotch 是当前开源 API 客户端赛道完成度最高的项目之一：以 Vue 3 单页应用为核心，通过平台抽象层与版本化内核 SDK 复用到 Web、桌面、CLI、Agent 四种形态，用 Rust Relay 补齐浏览器能力短板，用 NestJS 后端与 Docker Compose 支撑团队协作与自托管，再以 Postman 兼容沙箱降低迁移门槛。工程风格函数式、模块边界清晰、发布节奏稳定，是"多端复用 + 开源核心商业化"的教科书样本。适合寻找 Postman 替代品的开发者与团队直接使用；适合想学习 Vue 大型应用架构、Tauri 混合开发、跨平台内核抽象设计的工程师精读 `hoppscotch-common` 的 platform/kernel/RequestRunner 三条主线，以及 `hoppscotch-relay` 的 Rust 网络引擎。
