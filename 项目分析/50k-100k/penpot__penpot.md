# penpot/penpot 项目分析

> 星数 61537（快照 2026-10-24）｜主语言 Clojure｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

Penpot 是开源的**设计与原型协作平台**（对标 Figma），以"设计即代码"为核心理念：文件原生基于 SVG/CSS/HTML 开放标准，支持自托管、实时协作、Design Tokens、插件体系与 MCP（AI）工作流。

| 项 | 值 |
|---|---|
| 名称/仓库 | penpot/penpot（GitHub repo id 48769241） |
| 星数 | 61537（快照），fork 4359，open issues 937（`gh api repos/penpot/penpot`，2026-11-02） |
| 主语言 | Clojure + ClojureScript（另有 Rust 渲染引擎、TypeScript 插件/MCP 层） |
| 许可证 | MPL-2.0（`LICENSE`） |
| 默认分支 | develop（`gh api`） |
| 版本 | package.json 2.17.0；CHANGES.md 顶部为 2.19.0 (Unreleased) |
| 最近活跃 | 最近 push 2026-11-02（`gh api pushed_at`），tarball 内文件 mtime 2026-11-02，维护极活跃 |
| 贡献者 | 约 357 人（contributors API 分页 per_page=1 末页 358，含匿名） |

## 2. 项目是什么，解决什么问题

Penpot（由西班牙公司 Kaleidos 开发）为设计团队提供可**完全自持**的设计基础设施：浏览器内使用或部署到自己的服务器，避免厂商锁定，满足合规与治理要求（README.md "full ownership of your design infrastructure"）。

它解决的核心痛点是设计-开发割裂：设计文件以 SVG/CSS/HTML 等开放标准表达，开发者在 Inspect 模式直接拿到可用的 SVG/CSS/HTML 代码（README.md Inspect mode 节）；CSS Grid 与 Flex Layout 让设计稿"行为像真实代码"；原生 Design Tokens 在设计与开发之间建立单一事实源（README.md，`common/src/app/common/types/tokens_lib.cljc`）。

第二层是**平台可编程性**：插件系统允许在沙箱中扩展平台能力（`plugins/`），MCP 服务器让 LLM 通过 Plugin API 对设计文件做查询、修改与创建，打通 design-to-design、code-to-design、design-to-code 双向工作流（`mcp/README.md`）。

第三层是**渲染架构升级**：从早期 SVG DOM 渲染迁移到 Rust+Skia 编译的 WebAssembly 引擎（`render-wasm/`），同一套 Rust 源同时服务浏览器工作台与无头导出服务，保证两者渲染结果一致（`render-wasm/README.md` Render targets 表）。

## 3. 技术栈与依赖

- **后端**：Clojure JVM。Integrant 做依赖注入（`backend/src/app/main.clj` 大量 `ig/ref`/`ig/init-key`）；PostgreSQL 经 `next.jdbc`（`backend/deps.edn`）；Redis（`backend/src/app/redis.clj`）；S3 对象存储用 AWS SDK v2（`software.amazon.awssdk/s3`，`backend/deps.edn`）；HTTP 路由 metosin/reitit（`backend/deps.edn`）。
- **前端**：ClojureScript，shadow-cljs 构建（ESM + browser 多 target，`frontend/shadow-cljs.edn`）；React 封装用 funcool/rumext（`frontend/deps.edn`）；SCSS 332 个；渲染跑在 Web Worker 中（`frontend/src/app/main/worker.cljs`）。
- **共享层**：.cljc 双端文件 264 个（`common/`），geom/svg/schema/types 序列化协议（fressian/transit，`common/src/app/common/`）。
- **渲染引擎**：Rust + Emscripten（`wasm32-unknown-emscripten`），2D 绘制用 rust-skia 预编译二进制，几何用 bezier-rs（`render-wasm/Cargo.toml`）。
- **插件/MCP/工具链**：TypeScript；插件运行时基于 Angular 构建 + SES（Secure EcmaScript）沙箱（`plugins/libs/plugins-runtime/src/lib/ses.ts`）；MCP 服务器为 TS 包（`mcp/package.json`）；monorepo 用 pnpm workspace（`pnpm-workspace.yaml`）+ 各模块 `deps.edn`。
- **导出服务**：无头 ClojureScript 跑在 Node（`exporter/src/app/wasm.cljs`）。

## 4. 目录结构与代码组织

顶层是"Clojure 模块 + pnpm 子 workspace"混合 monorepo（`deps.edn` 仅做 clojure-lsp 汇聚：frontend/backend/common/exporter 四个 local/root）：

| 目录 | 职责 |
|---|---|
| `frontend/` | ClojureScript 工作台/查看器 SPA（887 个 clj*/cljs 文件，163MB 含资源） |
| `backend/` | Clojure 后端：RPC API、实时协作、存储、任务、MCP（305 个 clj 文件） |
| `common/` | 前后端共享 .cljc 数据模型与几何/序列化（285 个文件） |
| `render-wasm/` | Rust+Skia 的 WASM 渲染引擎（107 个 rs 文件） |
| `exporter/` | 无头导出服务（SVG/PNG/PDF，42 个 Clojure(Script) 文件） |
| `plugins/` | 插件运行时 + 官方示例（独立 pnpm workspace，TS） |
| `mcp/` | 官方 MCP 服务器（TS，经 WebSocket 连接 Penpot MCP 插件） |
| `library/` | @penpot/library：内存构建 .penpot 文件并导出 ZIP（`library/README.md`） |
| `media-processor/` | TS 媒体处理微服务（esbuild+vitest） |
| `docker/`、`manage.sh` | devenv 镜像与本地编排（compose 分 infra/main/gpu 三层，`docker/devenv/`） |
| `docs/` | 内嵌文档（98MB）；`.github/workflows/` 28 条 CI 流水线 |

组织方式评述：按运行时边界拆分（浏览器/JVM/Node/WASM），跨端逻辑全部下沉到 `common`（cljc）与 WASM 二进制协议，是 Clojure 全栈 + Rust 混合体的典型样板。

## 5. 核心架构与关键实现

### 5.1 后端：Integrant 系统与 RPC API

整个服务端是一个 Integrant 配置图（`backend/src/app/main.clj`，1060 行），组件含 db pool、redis、msgbus、http、jobs/worker、metrics、migrations 等，通过 `(ig/ref ...)` 声明依赖。对外 API 统一走 RPC 命令表：`backend/src/app/rpc/commands/` 下 29 个文件（files、teams、projects、comments、fonts、search、webhooks……），每个命令经 `app.common.schema` 校验、`permissions.clj` 鉴权、`rlimit`/`climit` 限流、`loggers/audit` 审计（`backend/src/app/rpc.clj` require 列表可见全链路）。数据库迁移为 174 个纯 SQL 文件（`backend/src/app/migrations/sql/0001-...sql` 起编号）。实时协作（多光标/评论/presence）由 `backend/src/app/msgbus.clj`（Redis 支撑）+ `backend/src/app/http/websocket.clj` 提供。

### 5.2 渲染引擎：Rust+Skia→WASM 的二进制协议

`render-wasm/` 是 Rust crate，目标 `wasm32-unknown-emscripten`，绘制走 rust-skia 自定义二进制（`render-wasm/README.md`）。同一源码出两个产物：`frontend`（release -O3，供浏览器）与 `export`（size -Oz，供 exporter）。渲染子系统按功能分文件：`render-wasm/src/render/` 下 fills/filters/fonts/shadows/strokes/text/grid_layout/images/pdf/svg 等，`tiles.rs` 负责分块渲染，`shapes.rs`/`math/` 承接几何。

Clojure(Script) 侧通过**字节缓冲协议**驱动：`common/src/app/common/render_wasm/`（README 明确"host-agnostic"：把 shape 序列化成 WASM 消费的 buffer，禁止依赖 DOM/WebGL，必须能在 Node 下跑）；浏览器工作台的驱动层在 `frontend/src/app/render_wasm/`（api/、gesture、shape、text_editor 等），并运行于 Web Worker（`frontend/src/worker/wasm.cljs`）。旧 SVG 渲染管线仍在（`frontend/src/app/main/render.cljs`："Rendering utilities and components for penpot SVG"），体现渐进迁移。导出端复用同一协议：`exporter/src/app/renderer/`（bitmap/pdf/svg/wasm）。

### 5.3 插件体系：SES 沙箱运行时

`plugins/libs/plugins-runtime/src/` 是 TypeScript 实现的插件宿主：`plugin-manager.ts` 管理生命周期，`load-plugin.ts`/`parse-manifest.ts` 加载与校验清单，`create-sandbox.ts` + `ses.ts` 用 SES `Compartment`/`harden` 构建安全沙箱（三方插件代码与平台状态隔离），`create-modal.ts` 提供插件 UI，`drag-handler.ts` 支持拖拽交互。官方示例在 `plugins/apps/`（contrast-plugin、example-styles，见 `plugins/README.md`）。插件能力通过 Penpot Plugin API 访问当前 page/file/selection；MCP 服务器（`mcp/`）本身也是经 WebSocket 连接"MCP 插件"的客户端，使 LLM 能在同一 API 上操作设计文件（`mcp/README.md` Architecture 节）。CHANGES.md 2.19 修复项显示插件 API 权限检查与 postMessage 来源校验在持续加固。

### 5.4 共享数据模型与 Design Tokens

设计文件的核心抽象集中在 `common/src/app/common/types/`：`shape.cljc`（含 `shape/` 子目录）、`file.cljc`、`page.cljc`、`component.cljc`、`grid.cljc`（CSS Grid 布局）、`text.cljc`、`tokens_lib.cljc`/`token.cljc`（Design Tokens）。几何算法在 `common/src/app/common/geom/`（point/rect/shapes/path），SVG 生成在 `common/src/app/common/svg/`。这层 cljc 同时被 JVM（导出、校验）与浏览器（工作台、渲染序列化）加载，是"前后端不漂移"的关键。

## 6. 构建与运行

- 官方开发环境：`./manage.sh` 构建/运行 devenv Docker 镜像（`manage.sh`，compose 拆分 `docker/devenv/docker-compose.{infra,main,gpu}.yml`）；文档入口 https://help.penpot.app/technical-guide/developer/devenv/ 。
- 后端/前端均为标准 Clojure 工具链：各模块 `deps.edn` + shadow-cljs（`frontend/shadow-cljs.edn` 有 :esm/:browser/:node-script 多 build）；`deps.edn` 顶层注释说明四模块 local/root 结构。
- 渲染引擎：`cd render-wasm && ./build`（或 `./watch` 增量），产物 .js/.wasm 拷贝到各消费端（`render-wasm/README.md`）。
- 前端包管理 pnpm（根 `package.json` 指定 `pnpm@12.16.1`，`pnpm-workspace.yaml`）；插件示例 `pnpm run start:styles-example`（`plugins/README.md`）。
- 库安装：`pnpm add @penpot/library`（`library/README.md`）。
- 生产部署：官方 docker compose（`docker/images/`，CI `build-docker.yml` 自动构建镜像）。

## 7. 测试与工程质量

- 测试分布：后端 114 个 `*_test.clj`（`backend/test/app/`）；前端 179 个测试 cljs（`frontend/test/`）；E2E 用 Playwright，`frontend/playwright/` 下 pages/specs/visual-specs/render-wasm-specs 共 85 个测试文件（含视觉回归）；渲染引擎用 insta 快照测试（`render-wasm/Cargo.toml` dev-dependencies `insta`）；media-processor 用 vitest（`media-processor/vitest.config.ts`）。
- CI 按模块隔离：`.github/workflows/` 有 tests-backend/common/e2e/exporter/frontend/library/mcp/plugins/wasm 九条测试流水线 + build-develop/staging/tag/docker/release 等，wasm 测试仅对 `render-wasm/**` 路径触发（`tests-wasm.yml`），并带 concurrency 取消旧跑。
- 质量观察：CHANGES.md 中 bug 修复条目普遍带 issue/PR 编号且大量外部贡献者署名（@filipsajdak、@0xTHAC0 等），说明 review 与回归跟踪规范；schema 校验、权限、限流、审计内建于 RPC 层（`backend/src/app/rpc.clj`）。

## 8. 社区与维护状态

- 极活跃：最近 push 2026-11-02（tarball 前一天仍有代码，`gh api pushed_at`）；open issues 937，fork 4359。
- 发布节奏：CHANGES.md 顶部 2.19.0 (Unreleased) 列出 4 个 epic（可配置快捷键、路径编辑器、四边独立描边、Design Tokens 内部库同步），版本从 2.17→2.19 演进（`CHANGES.md`、`package.json`）。
- 贡献者结构：约 357 名贡献者（API 分页推断）；公司 Kaleidos 主导 + 社区贡献并存；THANKYOU.md、CODE_OF_CONDUCT.md、SECURITY.md、社区论坛（community.penpot.app）齐备（仓库顶层文件）。
- 生态：Verified Digital Public Goods 徽章（README.md）；管理公开透明（Taiga 看板徽章）。

## 9. 亮点与风险

**亮点**
1. 全链路开放标准（SVG/CSS/HTML/JSON）+ 可自托管，是 Figma 之外唯一成熟的开源替代（README.md，`docs/`）。
2. Rust+Skia WASM 渲染引擎与 ClojureScript 序列化层解耦，浏览器与无头导出共用同一二进制协议，架构干净（`render-wasm/README.md`、`common/src/app/common/render_wasm/README.md`）。
3. 插件用 SES 沙箱 + MCP 服务器原生打通 AI 工作流，可编程性领先多数设计工具（`plugins/libs/plugins-runtime/src/lib/ses.ts`、`mcp/README.md`）。
4. cljc 共享模型 + 174 个 SQL 迁移 + Integrant 组件图，工程纪律好，测试覆盖九条 CI 流水线。
5. 代码内自带 AI 协作规范（`.agents/skills/` 下 20+ 工程技能文档），对 AI 辅助开发友好。

**风险/局限**
1. 技术栈门槛高：Clojure/ClojureScript/Rust/TS 四栈并存，社区贡献者上手成本大（目录结构所然）。
2. 新旧渲染管线并存（SVG DOM 与 WASM），迁移完成前存在双路径维护负担（`frontend/src/app/main/render.cljs` vs `render-wasm/`）。
3. 部署组件多（PostgreSQL/Redis/S3/导出服务/media-processor），自托管运维复杂度高（`docker/devenv/` compose 拆三层可佐证）。
4. 仓库体量大（源码 tarball 207MB、含大量媒体资源），轻量参与不便。

## 10. 总结评价

Penpot 是"开源设计平台"这一赛道里工程完成度最高的项目：以 Clojure 全栈为骨架、Rust/WASM 为渲染内核、TypeScript 为扩展层，把自托管、开放标准、Design Tokens、插件与 MCP/AI 整合成一个连贯平台，且提交与发布节奏极为活跃。它适合需要设计资产自主可控的团队自托管使用；对 ClojureScript 大型应用工程、WASM 渲染引擎设计、SES 沙箱插件体系感兴趣的开发者，这个仓库提供了教科书级的真实样本——建议从 `backend/src/app/main.clj`（系统组装）、`common/src/app/common/render_wasm/`（渲染协议）与 `plugins/libs/plugins-runtime/`（插件宿主）三处切入读源码。
