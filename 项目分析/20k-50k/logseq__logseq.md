# logseq/logseq 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 ClojureScript｜档位 20k-50k｜分析日期 2026-11-05
> 注：经 gh api 核实，2026-11-05 实际星数已升至 45,478， forks 2,866，open issues 1,028（`gh api repos/logseq/logseq`）。

## 1. 项目速览

一句话定位：隐私优先、本地优先的双链大纲笔记/知识管理平台，2026 年完成从「文件图（md/org）」到「数据库图（SQLite）」的整体重构（DB version），正在押注实时协作（RTC）与 AI（MCP/语义搜索）。

| 项目 | 事实 |
| --- | --- |
| 名称 | logseq/logseq |
| 星数 | 39,500（快照）；45,478（2026-11-05 实测） |
| 主语言 | ClojureScript（GitHub 语言字段标 Clojure）+ OCaml（CLI） |
| 许可证 | AGPL-3.0（LICENSE.md；gh api license.spdx_id） |
| 最近活跃 | pushed_at 2026-11-04（前一天仍在提交） |
| 最新 release | 2.0.2（2026-10-07）；nightly 持续构建（2026-10-25） |
| 创建时间 | 2020-05-23 |

## 2. 项目是什么，解决什么问题

Logseq 是一个开源的大纲式双链笔记与知识管理平台，灵感来自 Roam Research、Org Mode、TiddlyWiki、Workflowy 与 Cuekeeper（README.md "Inspiration" 节）。核心主张是隐私、长久与用户控制：笔记默认存在本地，支持 Markdown/Org-mode 文件格式、PDF 标注、任务管理与庞大的插件/主题生态（README.md "Why Logseq?"）。

本项目最重要的当代事实是 **数据库版本（DB version）重构**：README.md 顶部第一节就是 "Database Version"，明确 master 分支已是 DB 版主线（稳定 DB 版走 `test/db` 分支），DB 图以 SQLite 为存储、带新的移动端（Capacitor/iOS）和 RTC（Real Time Collaboration）同步；README 同时警告 DB 版仍处 beta、RTC/移动端处于 alpha、可能丢数据（README.md L69-81）。发布历史印证了这条迁移路径：0.10.x（文件版）最后一个是 0.10.15（2025-12-01），之后直接进入 2.0.1（2026-07-13）、2.0.2（2026-10-07）（gh api /releases）。

架构层面，仓库自带的 `CODEBASE_OVERVIEW.md` 与 `docs/agent-guide/implemented/architecture/2026-08-24-logseq-runtime-and-engineering-guide.md` 给出了权威的系统模型：浏览器渲染进程只负责 UI，图数据的事务与查询全部收进独立的 **db-worker** 线程；桌面端由 Electron main 管理 **db-worker-node** 本地守护进程（文件系统 SQLite 为唯一事实源）；CLI 与桌面共享同一守护进程与锁协议；远端同步由 Cloudflare Worker（D1/DO SQLite + R2）承担。这是一次把「文件解析 + 内存 DataScript」范式彻底换成「单一 SQLite 事实源 + 多运行时共享」的改造。

解决的核心问题可归纳为三点：一是把个人知识库从「一堆 md 文件 + 内存索引」升级为强 schema 的本地数据库，使属性（property）、类（class）、查询在大库上可靠且快；二是让同一份图跨桌面/移动/CLI/浏览器使用（同一 db-worker 核心配不同平台适配器）；三是为协作与 AI（语义搜索、MCP server）预留基础设施。

## 3. 技术栈与依赖

- **主体语言**：ClojureScript（src/main + deps/ 下 937 个 .cljs，约 26.7 万行）+ 31 个 .cljc；CLI 为 OCaml 71 个 .ml 文件（约 4 万行，经 Melange 编译到 JS，见 cli/logseq-cli.opam：依赖 dune>=3.23、ocaml>=5.1.1、melange、melange-edn-melange）。
- **编译/构建**：shadow-cljs 3.4.6（cljs→js，shadow-cljs.edn 定义 app / db-worker / db-worker-node / electron / mobile / publishing / test 共 7 个构建目标）+ webpack（多 config：app/mobile）+ gulp 5（静态资源与 Electron 打包）+ Babashka（bb.edn 任务）。
- **UI**：React 19.2.6（package.json）+ `io.factorhouse/hsx`（Rum 的替代品，组件用 hsx 写）+ `io.github.logseq/rfx`（自维护的 re-frame 式状态管理，deps.edn）；样式用 Tailwind + stylelint，并自研组件库 **Shui**（deps/shui/README.md："Shui is the component library for logseq"，面向统一视觉与插件作者复用）。
- **数据库**：DataScript（**logseq/datascript fork**，deps.edn）做内存 Datalog 查询；持久层 `@sqlite.org/sqlite-wasm` 3.51.2（浏览器 OPFS）与 `better-sqlite3` 12.9.0（Node/桌面，package.json）。`deps/db` 是 DB 图的 schema 与 API 库（`logseq.db` 命名空间，deps/db/README.md）。
- **解析器**：mldoc npm 包 ^1.5.9（OCaml 写的 md/org 解析器，wasm 供 cljs 调用；README.md 致谢节）。
- **桌面/移动**：Electron + Capacitor 8.3.1（android/、ios/ 目录，capacitor.config.ts）。
- **其他关键库**：promesa（Promise）、comlink（worker RPC，src/main/frontend/worker/db_worker.cljs）、SCI 0.12.51（Clojure 解释器，用户查询/脚本，src/main/frontend/extensions/sci.cljs）、isomorphic-git（版本历史）、reitit-frontend（路由）、tongue（i18n）、playwright 1.58.2（E2E，devDependencies）。
- **服务端**：Cloudflare Worker + D1 + Durable Objects + R2（deps/db-sync/worker/wrangler.toml 及 migrations/），并提供 Node 适配器自托管（deps/db-sync/README.md）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| src/main/frontend/ | 前端主体：components/（100 个 UI 组件文件）、handler/（约 40 个系统级 handler）、db/、worker/（db-worker 核心）、extensions/（sci、pdf、graph、srs、latex、zotero 等）、modules/、util/ |
| src/main/logseq/ | 插件 API（api/、api.cljs）、sdk/、cli/ 守护进程共享代码、db_worker/（daemon、graph_backup、server_list） |
| src/main/mobile/ | 新移动端（DB 版）代码 |
| src/electron/ | Electron 主进程：db_worker.cljs（守护进程生命周期）、plugin.cljs、mcp_server.cljs、mcp_transport.cljs、embedding_server.cljs、keychain 等 |
| deps/ | 自有库集合：db（DB 图 schema/API）、db-sync（同步服务端）、outliner（大纲操作）、graph-parser（文件图解析，兼容旧版）、graph-lifecycle、common、publish/publishing（发布）、shui（组件库）；每个库自带 deps.edn/bb.edn/test |
| cli/ | OCaml CLI（dune 工程，lib/ 含 agent.ml、auth_command.ml、block.ml、command_registry.ml 等），编译产物经 Vite 打成 Node 可用的 static/logseq-cli.js（docs/cli/logseq-cli.md） |
| clj-e2e/、cli-e2e/ | JVM 驱动的前端 E2E 与 CLI E2E（bb.edn 驱动） |
| android/、ios/ | Capacitor 移动壳 |
| docs/ | 开发文档 + docs/agent-guide/implemented/（带日期的"已实现"架构决策文档） |
| sidecar/ | embedding_server.py（本地语义向量服务，Python） |
| src/test/ | 255 个 *_test.cljs 单元测试 |
| .github/workflows/ | 24 个 CI 工作流 |

组织方式评述：典型的**Clojure monorepo**（deps.edn + git 依赖 + shadow-cljs 多目标），但已不是当年单一前端应用：现在是「一个 worker 核心 + 多运行时（浏览器/Electron/Node/CLI/移动）+ 云端 Worker」的家族式布局，外加一条 OCaml 技术线（CLI）和一条 Python 线（embedding sidecar）。库以 deps/* 独立成包（各有测试与 CI），边界清楚。

## 5. 核心架构与关键实现

### 5.1 db-worker：图数据的唯一权威（线程模型）

DB 版最大的架构决定是把所有图事务/查询移出 UI 线程。浏览器端入口 `src/main/frontend/worker/db_worker.cljs` 用 Comlink 暴露 db-core 代理，并显式禁止 React 进入 worker：

```clojure
(defn- check-worker-scope!
  []
  (when (or (gobj/get js/self "React")
            (gobj/get js/self "module$react"))
    (throw (js/Error. "[db-worker] React is forbidden in worker scope!"))))
```

worker 通过 `frontend.worker.platform` 抽象宿主能力（env/storage/kv/broadcast/websocket/crypto/timers/sqlite 七个必选适配段）：浏览器适配器走 OPFS + WebSocket，Node 适配器走文件系统 + `node:sqlite DatabaseSync`（docs/agent-guide/.../2026-08-24-logseq-runtime-and-engineering-guide.md）。桌面端同一 worker 核心跑成 `db-worker-node` 守护进程，绑 127.0.0.1 随机端口，暴露 `/healthz`、`/v1/events`（SSE）、`/v1/invoke`（Transit 编码 RPC）、`/v1/shutdown` 四个端点，并用锁文件保证"一张图最多一个活跃写者"（同上指南 + docs/cli/logseq-cli.md）。

### 5.2 事务与大纲：串行队列 + outliner 库

UI 侧发起的写操作经 `src/main/frontend/db/transact.cljs` 按 repo 串行入队（每个 graph 一条 promise 链，`*repo->outliner-mutation-tail`），交给 `deps/outliner`（logseq.outliner.core/pipeline/op）生成 DataScript 事务并回报变更（datascript_report）。DB 图的 schema 与实体 API 集中在 `deps/db`（`logseq.db.frontend.schema.cljs`、`logseq.db.frontend.malli_schema.cljs` 用 malli 定义校验；AGENTS.md 规定新增属性必须登记到 `logseq.db.frontend.property/built-in-properties`）。

### 5.3 双链/块引用：datom 级引用计算

引用体系建立在 DataScript 的 ref 属性（`:block/refs`、`:block/path-refs`）之上，核心算法在 `deps/db/src/logseq/db/common/reference.cljs`，为了避免实体重建开销直接扫 datom：

```clojure
(defn- datom-vs
  "Return all values for attr a on entity e, normalized to entids, as a set."
  [db e a]
  (into #{}
        (comp (map :v) (map entid))
        (d/datoms db :eavt e a)))

(defn- get-path-refs
  [db entity]
  (let [refs (mapcat :block/refs (ldb/get-block-parents db (:block/uuid entity))) ...]))
```

删除块时级联清理引用在 `deps/db/src/logseq/db/common/delete_blocks.cljs`。查询 DSL（`{{query ...}}`）的执行也被推到 worker 内：UI 侧 `src/main/frontend/db/query_dsl.cljs` 自述"Query execution belongs to the db worker. This namespace only keeps pure query-string transforms"，高级用户仍可写 DataScript Datalog 查询（CODEBASE_OVERVIEW.md "DataScript" 节）。

### 5.4 RTC 同步与云端：Cloudflare Worker + D1/DO

`deps/db-sync/` 是同步服务端：Cloudflare Worker 实现远程图访问与有序同步（D1/DO SQLite 存元数据、R2 存大对象），schema 变更走 wrangler D1 migrations（deps/db-sync/README.md）；提供 Node 适配器供自托管（`pnpm --dir deps/db-sync run test:node-adapter`，package.json scripts）。worker 目录里还能看到 AI 面向的 API：`mcp_request.mjs`、`semantic_api_tools.mjs`、`chatgpt_app.mjs`（deps/db-sync/worker/）。客户端侧的 RTC 流程在 `src/main/frontend/handler/db_based/rtc_flows.cljs`、`sync.cljs` 与 `src/main/frontend/db/rtc/`。

### 5.5 插件系统与桌面 AI 能力

插件 API 实现于 `src/main/logseq/api.cljs` + `api/`（app/block/editor/db 等域）与 `api/db_based.cljs`（DB 版适配层），SDK 在 `src/main/logseq/sdk/`，文档站 plugins-doc.logseq.com（README.md "Plugin API"）；宿主逻辑在 `src/main/frontend/handler/plugin.cljs`，UI 复用 Shui 组件库降低插件视觉不一致（deps/shui/README.md）。桌面端自带 **MCP server**（src/electron/electron/mcp_server.cljs、mcp_transport.cljs）与语义搜索 embedding 服务（embedding_server.cljs + sidecar/embedding_server.py），说明 AI 入口已进入桌面产品形态。

## 6. 构建与运行

环境（docs/develop-logseq.md + .github/workflows/build.yml）：Node 24、pnpm、Java 21 + Clojure CLI 1.12.4、Babashka 1.12.215、OCaml 5.1.1（仅 CLI 构建）。

```bash
pnpm install
pnpm watch          # 浏览器开发，http://localhost:3001（shadow-cljs nREPL 8701）
pnpm dev-electron-app          # 开发 Electron 桌面端
pnpm release-electron          # 打包桌面应用（gulp electronMaker）
pnpm cljs:release              # 发布 web（app + db-worker + db-worker-node + publishing）
pnpm test                      # cljs:test + cljs:run-test（单元测试）
bb dev:lint-and-test           # lint + 单测（AGENTS.md 推荐）
bb -f clj-e2e/bb.edn test      # 前端 E2E；CLI E2E: bb -f cli-e2e/bb.edn test
pnpm cli:release && pnpm db-worker-node:release:bundle   # 构建 logseq CLI
```

shadow-cljs 构建目标（shadow-cljs.edn）：`:app`（init-fn `frontend.core/init`）、`:db-worker`、`:db-worker-node`、`:electron`、`:mobile`、`:publishing`、`:test`。运行时可通过 shadow env 开关插件（ENABLE_PLUGINS）与本地 DB 同步（ENABLE_DB_SYNC_LOCAL）。

## 7. 测试与工程质量

- **单元测试**：src/test/ 255 个 *_test.cljs；deps/* 各库独立测试（db 19、outliner 12、graph-parser 6、db-sync 24、common 7 等，共约 73 个），以 nbb-logseq + nbb-test-runner 运行（deps/db/README.md）。
- **E2E**：clj-e2e/（JVM 驱动浏览器的前端 E2E，自带 bb.edn 与 CI clj-e2e.yml、clj-rtc-e2e.yml）；cli-e2e/（CLI E2E，cli-sync-stress.yml 专测同步压力）；devDependencies 里有 playwright。
- **CI**：24 个工作流——总 CI（build.yml：typos 拼写检查、cljs 测试、deps 库矩阵）、每个 deps 库独立 workflow（deps-db.yml、deps-outliner.yml 等）、桌面/移动/iOS 构建发布、web 部署（deploy-web-app.yml、deploy-web-nightly.yml）、i18n lint、stale issues、pr-labeler。
- **工程规范**：根目录 AGENTS.md 与各目录 AGENTS.md 定义了构建/测试命令、错误处理哲学（fail-fast、不加兼容层）、i18n 强制 skill、提交规范；`.agents/skills/`（如 logseq-i18n）、`.spec-dev-tool/`、`prompts/review.md`、`docs/agent-guide/implemented/` 显示团队已把 AI 辅助开发流程制度化（2026-08~09 的架构决策文档均为"已实现"状态并可溯源）。lint 工具链：clj-kondo、cljfmt、stylelint、typos、i18n-lint。

## 8. 社区与维护状态

- **活跃度**：pushed_at 2026-11-04，nightly release 2026-10-25，稳定 release 2.0.2（2026-10-07）——截至分析日仍是高频开发状态（gh api）。2026 年节奏：2.0.1（7 月）→ 2.0.2（10 月），0.10.x 文件版止于 2025-12。
- **贡献者结构**：顶部贡献者 tiensonqin 11,922 commits（创始人）、logseq-cldwalker 2,927、xyhp915 2,604、RCmerci 2,141、andelf 916、sprocketc 888、pengx17 802（gh api /contributors）。前四名均属公司核心团队，社区贡献者呈"长尾多、单量小"形态；Open Collective 赞助 + JetBrains 赞助（README.md）。
- **issue 管理**：open issues 1,028；DB 版 bug 走 logseq/db-test 仓库，功能请求走 Discord #db-feedback 频道（README.md L81-86），官方明显在把讨论导向 Discord/论坛。
- **文档**：仓库内文档密度极高（CONTRIBUTING.md、docs/develop-logseq.md、docs/agent-guide/、各 deps 库 README），AGENTS.md 表明文档同时服务人类与 AI 贡献者。

## 9. 亮点与风险

**亮点**

1. db-worker 单一权威 + 平台适配器架构干净：浏览器 worker（OPFS SQLite）、Node 守护进程（文件 SQLite）、CLI（OCaml/Melange）共享同一核心，一张图一个写者的锁协议明确（docs/agent-guide 运行时指南）。
2. DataScript/Datalog 保留为查询语言，属性/类/引用做成 datom 级实现（deps/db），查询执行下沉 worker，UI 线程保持纯渲染。
3. 测试与 CI 覆盖面在同类开源笔记应用中罕见：255 个单测文件 + 双 E2E 体系 + 24 个工作流 + 同步压力测试（cli-sync-stress.yml）。
4. 插件生态与开发者体验：SDK + MCP server + Shui 组件库 + 完整插件文档站，桌面端直接内置 AI 代理入口。
5. 工程流程高度文档化且可溯源（docs/agent-guide/implemented/ 带日期、状态"implemented"、对照源码验证）。

**风险/局限**

1. 迁移阵痛期：master 已是 DB 版 beta，README 明示"数据丢失可能"（README.md L73），0.10.x 文件版用户（md/org 本地文件是很多人的核心诉求）面临去留抉择；文件图仅靠 deps/graph-parser 维持兼容导入。
2. 技术栈门槛极高：ClojureScript + OCaml + Cloudflare Worker 三线并行，社区外部贡献成本大，核心提交集中于 4 名团队成员。
3. AGPL-3.0 许可对商业集成不友好（LICENSE.md）。
4. RTC/移动端仍在 alpha，同步一致性与数据安全尚需时间验证；自托管 db-sync 虽有 Node 适配器但文档偏开发者向。

## 10. 总结评价

Logseq 是 2020 年代双链笔记浪潮中体量最大、工程最重的开源作品：39k+ 星的背后是一个从 Rum/re-frame 时代一路重构到「React 19 + hsx + 自研 rfx + SQLite 单一事实源 + 多运行时 worker」的野心项目，2026 年的 DB version + RTC + MCP 组合拳显示它已从个人笔记工具转向"本地优先的知识协作平台"。代码库组织清晰、文档与测试密度高，非常适合想学习 ClojureScript 大型应用架构、DataScript/Datalog 数据建模、worker 进程隔离与本地优先同步设计的工程师精读；对普通用户则建议：追求稳定仍用 0.10.x，尝鲜 DB 版务必开自动备份。
