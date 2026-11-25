# TriliumNext/Trilium 项目分析

> 星数 38923（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：Trilium Notes 是一个面向大型个人知识库的开源跨平台层级笔记应用，由原作者 zadam 移交给社区后由 TriliumNext 组织接棒维护，同时支持桌面端、自托管服务器与纯浏览器运行。

| 项目 | 内容 |
| --- | --- |
| 名称 | TriliumNext/Trilium（原 zadam/trilium 社区接棒版） |
| 星数 | 38923（gh api 快照 2026-11-14） |
| 主语言 | TypeScript（gh api 核实） |
| 许可证 | AGPL-3.0-or-later（LICENSE、package.json） |
| 当前版本 | 0.107.0（package.json，v0.107.0 发布于 2026-11-08） |
| 最近推送 | 2026-11-14（分析当日，gh api pushed_at） |
| 默认分支 | main，forks 2629，open issues 588 |
| 代码规模 | 非测试 TS/TSX 约 39.9 万行、3632 个 TS 文件；测试 spec 文件 1596 个 |

## 2. 项目是什么，解决什么问题

Trilium Notes 解决的核心问题是"如何组织和管理规模达数万条以上的个人知识"。它把笔记组织成任意深度的树，并允许一条笔记通过克隆（cloning）同时挂载在多个位置——树关系与笔记本体是分离存储的（`notes` 表无父指针，`branches` 表记录 `noteId + parentNoteId`，见 `packages/trilium-core/src/assets/schema.sql`）。在此之上提供了标签/关系属性系统、全文搜索、笔记版本历史、笔记级加密、每日笔记、关系图谱、画布（Excalidraw）、思维导图（Mind Elixir）、地理地图（Leaflet）等能力（README.md 特性列表）。

它是一个"多形态单体"：同一套核心代码以四种形态运行——Node 服务器（`apps/server`）、Electron 桌面应用（`apps/desktop`，主进程内嵌完整服务器）、纯浏览器 standalone 版（`apps/standalone`，核心跑在 Web Worker + SQLite WASM/OPFS 上）、Capacitor 移动壳（`apps/mobile`）。自托管服务器与桌面客户端之间通过私有同步协议保持多端数据一致，还支持将笔记公开发布到互联网（share）和浏览器网页剪藏（web-clipper）。

社区接棒背景明确：README「Why TriliumNext?」一节写明原 Trilium 开发者 zadam 已将仓库交给社区项目；从 zadam/trilium v0.63.7 迁移无需任何特殊步骤（沿用现有数据库），但 v0.90.4 之后同步版本号已提升，不再与原版互通。`contributors.json` 中 zadam 仍被列在贡献者名单中，lead-dev 为 Elian Doran（eliandoran）。

近年演进方向清晰：monorepo 化重构（trilium-core 抽出共享核心）、i18n 全量化（Weblate 40+ 语言）、AI 能力引入（内置 LLM 聊天、MCP server、多种 provider），以及 pure-browser 形态（standalone/mobile）。这些都能从目录结构与依赖清单中直接看出。

## 3. 技术栈与依赖

- **语言/工具链**：TypeScript 全栈；TypeScript 6.0.3（编译 API 用）+ `@typescript/native` 7.0.2（typecheck 用，双版本共存是刻意设计，见 CLAUDE.md）；Vite 8 打包、Vitest 5 测试、ESLint、Playwright E2E；pnpm 12 workspace + Nx（`pnpm-workspace.yaml`、`.nxignore`）。
- **服务端**：Express 5.3.0、better-sqlite3 13.0.3、helmet、express-session、openid-client（OpenID 登录）、time2fa（TOTP 两步验证）（`apps/server/package.json`）。
- **桌面端**：Electron 45.2.3 + electron-forge 打包（`apps/desktop/package.json`）；渲染进程经 `trilium-app://` 自定义协议加载，contextIsolation 开启、nodeIntegration 关闭（CLAUDE.md）。
- **前端**：Preact 11（新 UI）+ 遗留 jQuery widget 体系（FancyTree 树组件）；CKEditor 5 v49.1.0 从源码在 `packages/ckeditor5` 内构建（含 mathlive 公式、Vim 模式适配插件）；CodeMirror 6（代码笔记，workspace 内 `overrides` 强制全树单例锁定，`pnpm-workspace.yaml`）；Excalidraw、Mind Elixir、Leaflet、Tabulator、Mermaid、PDF.js（README「Shoutouts」+ packages/ 目录）。
- **AI**：Vercel AI SDK 7（openai/anthropic/google/deepseek/openrouter provider）、@modelcontextprotocol/sdk（内置 MCP server）、@anthropic-ai/claude-agent-sdk、codex-acp（`apps/server/package.json` dependencies）。
- **浏览器 standalone**：`@sqlite.org/sqlite-wasm` 持久化到 OPFS，Web Locks 选举持有数据库的标签页，Service Worker 转发其余标签页请求（CLAUDE.md「Standalone」节）。
- **i18n**：i18next + Weblate 托管 40+ 语言，三套英文 catalog 按加载方拆分（CLAUDE.md「Internationalization」节）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `apps/server` | Node 后端：Express 应用、ETAPI、share 发布、MCP、Node 专属 provider |
| `apps/client` | Preact/jQuery 前端，被 server/desktop/standalone 共用；零依赖 core 包，仅经 REST/WebSocket 交互 |
| `apps/desktop` | Electron：主进程内嵌完整 server + IPC 消息桥 |
| `apps/standalone` | 纯浏览器全栈：core 在 Web Worker + SQLite WASM/OPFS |
| `apps/mobile` | Capacitor 壳（Android/iOS 原生请求桥） |
| `apps/web-clipper` | 浏览器扩展（网页剪藏） |
| `apps/edit-docs`、`build-docs`、`website`、`dump-db`、`db-compare`、`icon-pack-builder`、`script-deployer` | 文档编辑/站点/工具类辅助应用 |
| `packages/trilium-core` | 核心共享包（590 个 TS 文件）：becca 实体缓存、服务、SQL、迁移、同步、内部路由 |
| `packages/commons` | 前后端共享类型与工具 |
| `packages/ckeditor5`、`codemirror`、`highlightjs`、`pdfjs-viewer`、`share-theme`、`splitjs`、`turndown-plugin-gfm` | 编辑器与渲染组件定制构建 |
| `packages/trilium-e2e` | 共享 Playwright E2E 测试 |
| `.changelog/` | 按日期命名的变更条目（frontmatter 标 type/area），release 流程的输入 |
| `scripts/` | release.mts 发布自动化、版本同步等脚本 |

组织方式评述：典型的 pnpm monorepo，但拆分原则很讲究——`trilium-core` 是"后端核心"（server、desktop、standalone 三方共享，因此禁止 Node 内建依赖，通过 `PlatformProvider` 抽象平台差异，`packages/trilium-core/src/services/platform.ts`），而 client 与 core 之间只共享类型（`@triliumnext/commons`），前端永远通过 API 调用后端。这个"后端-前端二分"而非"Node-浏览器二分"的边界是整个仓库最重要的架构决策（CLAUDE.md「Monorepo Structure」节）。

## 5. 核心架构与关键实现

### 5.1 三层缓存与实体系统（笔记树模型）

数据访问全部经内存缓存：服务端 Becca（`packages/trilium-core/src/becca/`）、前端镜像 Froca（`apps/client/src/services/froca.ts`，注释明言"FROntend CAche，懒加载 noteId 及其父子 branch"）、公开分享专用 Shaca（`apps/server/src/share/`）。实体类在 `packages/trilium-core/src/becca/entities/`：`BNote`（1643 行，内容/类型/搜索记分）、`BBranch`（多父级关系，注释强调"branch 身份易变，应始终以 noteId 为准"）、`BAttribute`（label/relation，支持可继承属性、`child:` 前缀与 template 关系三种继承机制）、`BRevision`、`BBlob`、`BAttachment`。所有实体继承 `AbstractBeccaEntity`，内建变更跟踪与 hash 生成。

SQLite schema 仅 12 张表（`packages/trilium-core/src/assets/schema.sql`）：`notes`、`branches`、`attributes`、`blobs`、`revisions`、`attachments`、`entity_changes`、`options`、`recent_notes`、`etapi_tokens`、`sessions`、`user_data`。内容与元数据分离（`blobId` 外键到 `blobs`），这是 0216 号迁移 `move_content_into_blobs` 的成果。

### 5.2 同步协议

同步由 `entity_changes` 表驱动：任何实体修改都会产生 EntityChange 记录（CLAUDE.md 强制"必须走缓存方法写库，直接 SQL 不产生同步记录"）。客户端同步实现在 `packages/trilium-core/src/services/sync.ts`（900 行）：登录用 HMAC（document secret + 时间戳）认证，随后按"push → pull → push"轮次推进，配合基于 sector 的内容哈希校验。代码中有两个防御性常量：`MAX_SYNC_ROUNDS = 50`（防单次同步无限循环）与 `MAX_SECTOR_RESYNC_ATTEMPTS = 3`（同一 sector 哈希三次对不齐即判定不可调和分歧并报错，注释详细解释了为何"无限重排队是同步空转的根源"）。`sync_mutex.ts`（16 行）保证进程内互斥。

```ts
// packages/trilium-core/src/services/sync.ts
export const MAX_SECTOR_RESYNC_ATTEMPTS = 3;
export const MAX_SYNC_ROUNDS = 50;
```

### 5.3 脚本系统（前后端 note 脚本）

后端脚本：`packages/trilium-core/src/services/script.ts` 用 sucrase 转译脚本笔记内容，经 `ScriptContext` 注入 API 后执行；执行前检查 `note.isContentAvailable()`（受保护笔记需先解锁），环境不匹配（前端脚本送后端跑）会通过 WebSocket 推 toast 提示。`scripting_guard.ts` 提供启用开关。前端脚本 API：`apps/client/src/services/frontend_script_api.ts` 暴露 froca、widget 基类、搜索、对话框等能力给用户脚本，并有 Preact 版扩展 `frontend_script_api_preact.ts`。外部自动化另有 ETAPI（`apps/server/src/etapi/`，token 认证、OpenAPI 描述文件 `internal.openapi.yaml`，要求保持向后兼容）。

### 5.4 多形态运行时（desktop / standalone / mobile）

桌面端 `apps/desktop/src/main.ts` 的 import 列表直接展示了架构：Electron 主进程内嵌 `@triliumnext/server` 的完整 Express 应用与 better-sqlite3 provider，渲染进程走 `trilium-app://` 协议，与主进程仅通过 preload 桥（`window.electronApi`）通信；主进程构建为 ESM + 代码分割，重依赖靠动态 import 落入懒加载 chunk。Standalone 把同一套 core 跑进浏览器 Worker（sqlite-wasm + OPFS，Web Locks 选主），因此 core 内禁止 `process.env` 与 Node 内建模块（含 `path`），路径操作走 `packages/trilium-core/src/services/utils/path.ts`。移动端则是 standalone 的 Capacitor 壳，原生侧只代理 GET/HEAD 请求。

### 5.5 AI 与 MCP（新版能力）

`packages/trilium-core/src/services/llm/`（chat、tools、web_search、skills、note_hint、tool_approval 等模块）实现内置 AI 聊天与工具调用，服务端路由在 `apps/server/src/routes/api/llm_chat.ts`；同时内置 MCP server（`apps/server/src/services/mcp/mcp_server.ts`），可让外部 MCP 客户端操作笔记。

## 6. 构建与运行

来自 README 与 CLAUDE.md 的实际命令：

```bash
npm install -g pnpm && pnpm install
pnpm server:start          # 开发服务器 http://localhost:8080
pnpm desktop:start         # Electron 开发应用
pnpm standalone:start      # 纯浏览器版
pnpm --filter <pkg> test <pattern>   # 最小范围测试（推荐）
pnpm typecheck             # 全项目类型检查（native tsc）
pnpm run --filter desktop electron-forge:make --arch=x64 --platform=win32  # 打桌面安装包
pnpm docker-build-debian   # 服务器 Docker 镜像
pnpm chore:release         # 交互式发布流程（scripts/release.mts）
```

测试约束：server 测试必须串行（共享数据库，forks pool、max 6 workers）；core 的 spec 由 server 与 standalone 两套运行时各跑一遍（better-sqlite3 vs sqlite-wasm），`pnpm --filter server test` 绿了不算完，还要 `pnpm --filter standalone test`（CLAUDE.md「Testing」节）。CLAUDE.md 明确警告本地永远别跑 `test:all`/全量 coverage 和 ESLint（ESLint 会 OOM，CI 也不跑 lint，门禁是 typecheck + per-package coverage）。

## 7. 测试与工程质量

- 测试规模：全仓 1596 个 spec 文件（`*.spec.ts/tsx/mts`），单测（Vitest：server 串行 + client happy-dom 并行）、集成（ETAPI 契约测试在 `apps/server/spec/etapi/`）、E2E（Playwright：`apps/server/e2e`、`apps/desktop/e2e`、`packages/trilium-e2e`）、浏览器模式测试（ckeditor5 经 @vitest/browser-playwright 驱动真实 Chromium）四层齐全。
- CI：`.github/workflows/` 共 26 个工作流——checks.yml、playwright.yml、codeql.yml（安全扫描）、release.yml（多 OS 矩阵构建）、nightly.yml（每夜构建）、release-flathub.yml / release-winget.yml（商店分发）、docker-publish.yml、i18n.yml（Weblate 同步）等，覆盖发布链路的每个环节。
- 质量工程细节突出：`codecov.yml`、`renovate.json`（依赖自动化，含大量安全下限 override）、`scripts/release.mts` 是一个可断点续跑的交互式发布向导（自动从 changelog 条目推断版本号、生成 release note、打 tag、合并 stable 分支）；commit 规范（`type(area): summary` + issue 关闭关键词必须在 subject）与 AI 协作规范（Assisted-by trailer）都写进了 CLAUDE.md。
- 观察：ESLint 在本地与 CI 均不运行（内存问题，packages/* 干脆不配置 lint），代码风格主要靠 format-check 与评审；搜索计分刻意全内存进行（放弃 SQL 层 LIMIT/OFFSET 换取打分正确性，CLAUDE.md「Important Patterns」）。

## 8. 社区与维护状态

- 活跃度极高：gh api 显示最后推送为分析当日（2026-11-14T00:51Z）；`.changelog/` 内最近条目日期到 2026-11-11，每晚 nightly 发布。
- Release 节奏（gh api releases）：v0.107.0（2026-11-08）、v0.106.1（2026-10-23）、v0.106.0（2026-09-25）、v0.105.0（2026-08-19）、v0.104.1（2026-07-25）——约每月一个功能版本，中间夹补丁版。
- 贡献者：gh api contributors 分页 Link 头显示 386 人；`contributors.json` 确认 lead-dev 为 eliandoran（Elian Doran），zadam 位列其中。开发高度依赖单一主导者，通过 GitHub Sponsors / Liberapay / PayPal 赞助维持（README）。
- 社区渠道：Matrix（桥接 XMPP）、GitHub Discussions/Issues；翻译由 Weblate 社区托管（40+ 语言）；有配套生态（TriliumDroid 安卓客户端、Trinote iOS 客户端、awesome-trilium、TriliumRocks 教程站，README）。
- 治理：原仓库 2017-05-23 创建（gh api created_at），zadam 弃坑后整体移交社区；CLAUDE.md 还透露工作树以 AI 协作为常态（.claude/ 技能目录、AI 提交规范），是"AI 辅助维护成熟开源项目"的典型案例。

## 9. 亮点与风险

**亮点**
1. 一套核心多形态复用做得非常干净：trilium-core 通过 PlatformProvider 抽象同时支撑 Node/Electron/浏览器 Worker 三种运行时，且用测试矩阵强制保证（CLAUDE.md「Platform Abstraction」）。
2. 数据模型设计成熟：notes/branches 分表支持多父克隆，blob 分表支撑内容去重，entity_changes 表驱动增量同步——十余年演进的稳定 schema。
3. 同步协议工程化程度高：HMAC 认证、sector 哈希校验、双重防死循环上限，注释直接记录了踩坑教训（sync.ts）。
4. 发布与 CI 自动化完备：26 个工作流覆盖从 nightly 到 Flathub/Winget 商店分发，release.mts 支持断点续跑。
5. 46KB 的 CLAUDE.md 架构文档 + 24 个内部技能文档，使 AI 辅助开发有章可循，反过来也是极高documentation 质量的人类贡献者入门材料。

**风险/局限**
1. AGPL-3.0 对商业化托管/二次分发有传染性约束。
2. 与原版 zadam/trilium 的同步协议已在 v0.90.4 后分叉，两边客户端不能混用（README）。
3. 前端处于 jQuery 遗留体系与 Preact 11 双轨过渡期（CLAUDE.md 明言"不要在同一个组件里混用"），新贡献者需理解两套范式。
4. 核心开发集中于 eliandoran 一人，赞助驱动的可持续性依赖个人；CLAUDE.md 自述 ESLint 因 OOM 实际不跑，静态检查存在盲区（未确认具体债务规模）。

## 10. 总结评价

TriliumNext/Trilium 是"个人知识库"赛道中工程深度最高的开源实现之一：它用一张分离的 branch 表和三层内存缓存撑起十万级笔记的树形组织，用 entity_changes 流水账支撑跨端同步，又把同一套后端核心塞进了 Electron、浏览器 Worker 和手机壳里。社区接棒近三年（自 zadam 2024 年停更后）不仅维持了月度发版，还完成了 monorepo 化、全量国际化、纯浏览器形态与 AI 能力四件大事，主导者 eliandoran 的投入清晰可见。适合重度笔记用户（尤其需要本地优先 + 自托管同步 + 脚本自动化的人）作为 Obsidian/Notion 之外的自托管替代品认真评估；对想研究"层级数据模型 + 增量同步协议 + 多运行时 monorepo"的工程师，`packages/trilium-core` 的 becca 实体体系与 `services/sync.ts` 是最值得精读的两块源码。
