# metabase/metabase 项目分析

> 星数 49819（快照 2026-10-24）｜主语言 Clojure｜档位 20k-50k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：Metabase 是最流行的开源 BI（商业智能）平台，让不懂 SQL 的业务人员也能通过图形界面自助查询数据、搭仪表盘，同时为开发者提供 SQL 编辑器、REST API 与嵌入式分析能力。

| 项 | 值 |
| --- | --- |
| 仓库 | metabase/metabase（默认分支 master，`gh api` 确认） |
| 星数 | 49819（快照），分析时 gh api 实测已到 50143 |
| 主语言 | 后端 Clojure，前端 TypeScript/React（CLJS 为辅） |
| 许可证 | 混合：`enterprise/` 目录外为 AGPL，目录内为 Metabase Commercial License（`LICENSE.txt`、`LICENSE-AGPL.txt`、`LICENSE-MCL.txt`） |
| 创建时间 | 2015-02（gh api） |
| 最近活跃 | master 最近 push 2026-11-03（分析前一天），极度活跃 |
| 规模 | src 约 359k 行 Clojure（1777 个 clj/cljc 文件），frontend/src 约 975k 行 TS/JS（9494 个文件），test 约 480k 行（实测统计） |

## 2. 项目是什么，解决什么问题

Metabase 解决的核心问题是"让全公司的人都能安全地问数据问题"。用户连接数据库（官方/社区驱动覆盖 20+ 种数据源，见 `modules/drivers/` 与 `docs/databases/connecting.md`）后，Metabase 自动同步库表元数据（`src/metabase/sync/`），并以图形化查询构建器让用户拖选维度/指标/过滤条件生成图表，无需写 SQL。有经验的用户可用原生 SQL 编辑器（README "Key Features" 与 `frontend/src/metabase/query_builder` 一带）。

数据以"问题（Question/Card）—仪表盘（Dashboard）—集合（Collection）"三层内容体系组织，附带权限体系（`src/metabase/permissions/`）、提醒与订阅（`src/metabase/notification/`、pulse）、语义层与 Data Studio（README 提到 Library/Data Studio/Transforms，对应 `src/metabase/data_studio/`、`src/metabase/transforms/` 模块）。查询以 MBQL（Metabase Query Language）这一中间表示描述，再由查询处理器编译成各数据源的 SQL/Mongo 查询等。

商业化采用 open-core：AGPL 开源核心 + `enterprise/` 下的付费功能（沙箱行级权限、SSO、审计、缓存粒度控制等，896 个 clj 文件），通过 `premium_features` 钩子接入，embedding SDK 允许把整套 Metabase 嵌入第三方应用（`frontend/src/embedding-sdk/`、rspack.embedding-sdk-*.config.js）。README 还宣传了 AI 能力 Metabot（`src/metabase/metabot/`、`src/metabase/agent_api/`）与 MCP 支持（`src/metabase/mcp/`）。

## 3. 技术栈与依赖

- 后端：Clojure 1.12 + Java（Dockerfile 安装 Temurin JDK 25），构建用 tools.deps（`deps.edn`），无 Leiningen。
- ORM/DB：Toucan 2（`toucan2.core`）+ Honey SQL 2（自封装为 `h2x`，见 `src/metabase/dashboards/models/dashboard.clj` 的 require）；Liquibase 管理 schema 迁移（`resources/migrations/`，编号至 065/2026 目录，仅 `*_update_migrations.yaml` 计数就超 2 万行）。
- 关键库（`deps.edn`，每条依赖均附用途注释）：Malli（数据校验）、Methodical（多层 multimethod）、Ring + buddy（JWT/加密）、cheshire（JSON）、Quartzite（定时任务）、iapetos（Prometheus 指标）等。
- 前端：React 19、TypeScript 6.0.3、Redux Toolkit、Emotion、d3、echarts 6（`package.json`，150 个 dependencies）；打包器为 rspack（Rust 版 webpack，多份 `rspack.*.config.js`），包管理用 Bun（`bun.lock`），Node >= 22。
- ClojureScript 桥：shadow-cljs（`shadow-cljs.edn`）——MBQL 5 查询库以 `.cljc` 同时编译给 JVM 后端与浏览器前端。
- 任务与 CI 工具：Babashka 任务器 `./bin/mage`（`bb.edn`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `src/metabase/` | 后端主体，149 个领域模块（api/、query_processor/、driver/、sync/、permissions/、collections/、dashboards/ 等） |
| `src/metabase/lib/` | MBQL 5 查询语言核心，131 个 `.cljc` 文件，前后端共享 |
| `src/metabase/legacy_mbql/` | 旧版 MBQL 的 schema 与归一化（兼容层） |
| `enterprise/` | 付费版后端（896 个 clj 文件）与前端，MCL 许可 |
| `frontend/src/` | React 前端（9494 个 js/ts 文件），含 `metabase/`（应用）、`metabase-lib/`（MBQL 5 的 TS 侧封装）、`embedding-sdk/` |
| `modules/drivers/` | 可选官方驱动 16 个：bigquery-cloud-sdk、snowflake、redshift、mongo、oracle、clickhouse、databricks、athena、sqlserver 等 |
| `e2e/` | Cypress 端到端测试（384 个 spec：206 ts + 178 js，`e2e/test/scenarios/`） |
| `docs/` | 用户文档 + `developers-guide/` 开发者手册（driver 开发、代码评审等） |
| `resources/migrations/` | Liquibase 迁移；`release/`、`hooks/`、`bin/` 发布与脚本 |

组织方式评述：典型的大型 Clojure 单体 + 前端 monorepo 混合体。后端正从"集中式 models/api/query_processor 大目录"向"149 个按领域划分的模块"迁移——`src/metabase/models/DO_NOT_ADD_NEW_FILES_HERE.txt` 明确写着新模型必须放进对应领域模块（如 `dashboards/models/dashboard.clj`），`src/metabase/api/` 同样留有 DO_NOT_ADD 文件；这是理解当前代码布局的关键。

## 5. 核心架构与关键实现

### 5.1 查询处理器（QP）与 MBQL 中间表示

QP 入口 `src/metabase/query_processor.clj` 把一次查询描述为 middleware 管线：around-middleware（审计、敏感字段拒绝、userland 包装、异常捕获）+ 预处理链 + 编译 + 执行 + 后处理。`src/metabase/query_processor/preprocess.clj:55` 定义了约 45 个预处理中间件组成的向量，覆盖：MBQL 归一化/校验 → 展开 Metric/Measure/宏 → 参数替换 → 隐式 join/字段补全 → 分桶（binning）与时间粒度 → 沙箱与企业级改写 → 特性检查。摘录（有删节）：

```clojure
(def ^:private middleware
  "All of these middlewares assume MBQL 5."
  [#'normalize/normalize-preprocessing-middleware
   #'fetch-source-query/resolve-source-cards
   #'metrics/adjust  #'measures/adjust  #'expand-macros/expand-macros
   #'parameters/substitute-parameters
   #'qp.auto-bucket-datetimes/auto-bucket-datetimes
   #'qp.middleware.enterprise/apply-sandboxing
   #'qp.add-remaps/add-remapped-columns
   #'optimize-temporal-clauses/optimize-temporal-clauses
   #'check-features/check-features])
```

MBQL 5（早期称 MLv2）实现在 `src/metabase/lib/`，是纯数据驱动的查询表示，用 Malli schema（`lib.schema` 等）严格校验，通过 `.cljc` 在 JVM 与 ClojureScript 间共享——前端查询构建器和后端编译器操作同一套数据结构，这是 Metabase 近年最重要的架构重构（`src/metabase/lib/convert.cljc` 负责 legacy MBQL ↔ MBQL 5 双向转换）。查询取消通过 core.async channel 传递（`src/metabase/query_processor/pipeline.clj` 的 `*canceled-chan*`）。

### 5.2 驱动抽象

`src/metabase/driver.clj`（2208 行）用 Methodical multimethod 定义驱动契约：`initialize!`、`can-connect?`、`execute-reducible-query`、`describe-database` 等数十个 defmulti，按 `dispatch-on-initialized-driver` 分派，支持层级继承——抽象驱动 `:sql`、`:sql-jdbc` 提供通用实现，具体驱动只覆写差异点（`src/metabase/driver/sql/`、`src/metabase/driver/sql_jdbc/`）。内置驱动（h2/postgres/mysql/sqlite）在 `src/metabase/driver/` 下，其余 16 个重量级驱动隔离在 `modules/drivers/`（如 mongo 共 8570 行 clj），经 `src/metabase/plugins/` 按需加载为插件。新数据源的接入成本被这套抽象压得很低，这也是 Metabase 数据源覆盖面广的直接原因。

### 5.3 数据模型：Toucan 2 + 领域模块

模型层基于 Toucan 2（Honey SQL 2 之上）与 Methodical 的多层派发。以 `src/metabase/dashboards/models/dashboard.clj` 为例：`:model/Dashboard` 映射到表 `report_dashboard`，通过 `derive` 挂上 `:hook/timestamped?`、`:hook/entity-id`、`:perms/use-parent-collection-perms` 等行为标签，权限判断（`mi/can-write?`）、序列化（serdes）、搜索、事件都按 model 多态派发。核心实体（Database/Table/Field/Card/Dashboard/Collection/User 等）连同其 REST 端点（`*_rest/` 模块）分散在各领域模块中，`src/metabase/models/` 只剩派发/接口基座。

### 5.4 同步引擎与调度

`src/metabase/sync/` 负责把外部数据库"搬进"Metabase 元数据库：fetch_metadata（拉表/列/约束）→ analyze（指纹采样、字段语义类型推断）→ field_values（枚举值缓存）→ schedules（Quartzite 定时）。描述性统计驱动了"自动分桶""智能推荐图表"等体验（配合 `src/metabase/analyze/`、`src/metabase/xrays/`）。

## 6. 构建与运行

来自 `docs/developers-guide/build.md`、`deps.edn` 注释与 `package.json`：

```bash
# 前端开发（含 CLJS 热更新）
bun install && bun run build-hot
# 后端开发 REPL
clojure -A:dev:drivers:drivers-dev     # 含测试命名空间
clojure -A:dev:ee:ee-dev               # 含企业版代码
(dev/start!)                           # REPL 内启动服务
# 后端直跑
clojure -M:run
# 后端测试（CI 同款，.github/workflows/backend.yml）
clojure -X:dev:ci:test
# 前端构建/发布
bun run build        # = build:cljs + build:js（shadow-cljs + rspack）
# Babashka 任务
./bin/mage           # 项目自带 CLI 任务器
```

生产部署以 Docker 为主（`Dockerfile` 多阶段构建：node:22 构建前端 + JDK 25 运行，`MB_EDITION=oss/ee`），另有 jar 发布。

## 7. 测试与工程质量

- 后端测试约 1350 个 clj/cljc 文件、48 万行（`test/`），规模与 src 相当（1.3:1），CI 用 `clojure -X:dev:ci:test` 分 edition/驱动矩阵跑（`backend.yml`、`drivers.yml`），覆盖率用 cloverage（`backend-cloverage.yml`）上报 codecov。
- 前端 jest 多套配置（`jest.config.js`、`jest.tz.unit.conf.js` 时区专项等）；e2e 为 Cypress，384 个 spec 按 `scenarios/` 分域，CI 有 e2e-matrix-builder、e2e-coverage 等 workflow，另有 loki 视觉回归（`loki.config.js`）与 chromatic。
- CI 共 40+ workflow（`.github/workflows/`），含跨版本测试（`cross-version.yml`）、跨分支迁移检查（`cross-branch-migrations.yml`）、bundle 体积监控、auto-backport 等工程化配置，成熟度很高。
- 代码质量观察：`deps.edn` 对依赖逐条写用途注释并钉住 Jackson 版本（注明 CVE 原因）；namespace 分模块 + clj-kondo 自定义 lint（`eslint.config.module-boundaries.mjs` 约束前端模块边界）；`DO_NOT_ADD` 文件显式圈住迁移期架构。

## 8. 社区与维护状态

- 极度活跃：master 最近 push 为 2026-11-03（分析前一天）；issue 池约 4.6k 开放。
- 发布节奏：10 月底同时发布 v0.65.2、v0.64.4，还有 v0.63.22、v0.58.38 等多条维护线在并行 backport（gh api releases），月度多版本 + 长期分支维护。
- 贡献者：contributors API 分页推断约 388 人；仓库 2015 年创建，十年持续演进。
- 商业支撑：Metabase 公司全职维护，open-core 商业模式（enterprise/ 目录 + Cloud），这也是项目能长期高投入的原因。
- 未确认：单月 PR 数、issue 关闭时长等细粒度指标未统计。

## 9. 亮点与风险

亮点：
1. MBQL 5 查询中间表示 + `.cljc` 前后端共享（`src/metabase/lib/`，131 个文件），前端构建器与后端编译器共用同一套 schema 化数据结构，是开源 BI 里少见的工程设计。
2. 约巴级中间件 QP 管线（`preprocess.clj` 45 个中间件）+ 层级驱动抽象（`driver.clj`），扩展点清晰，社区驱动生态（16 个官方可选驱动）即其证明。
3. 测试体量与 CI 成熟度极高三者并存：测试/源码行数比约 1.3:1，40+ 个 CI workflow，多版本并行维护。
4. 大型 Clojure 代码库正在完成"领域模块化"重构（149 个模块 + DO_NOT_ADD 守门），架构演进路径可作参考样本。

风险/局限：
1. 双许可证理解成本高：AGPL 对商用集成有传染性要求，`enterprise/` 与嵌入 SDK 受 MCL 约束，二次开发前必须读 `LICENSE-EMBEDDING.txt`。
2. 体量巨大：全仓 Clojure + TS 近 150 万行，构建依赖 Bun/rspack/shadow-cljs 三件套，贡献门槛与冷启动成本高。
3. MBQL 5 与 legacy MBQL 双轨并存（`legacy_mbql/` + `lib/convert.cljc`），新旧代码并存期阅读成本增加。
4. AGPL 核心中混有大量企业版钩子（`premium_features`、`qp.middleware.enterprise`），纯社区版功能边界需实际跑起来才能确认。

## 10. 总结评价

Metabase 是"开源 BI 事实标准"级别的项目：产品定位（人人可用的数据问答）与工程实现（Clojure 后端 + schema 化查询中间表示 + 层级驱动抽象）互相成就，近 5 万星来自十年如一日的活跃维护与 open-core 商业闭环。它最适合作为企业自助分析的落地工具直接部署，也极适合两类读者深读源码：一是想学习大型 Clojure 工程如何组织模块、测试与 CI 的开发者（QP 管线与 Toucan 2 模型层是教科书级素材），二是想接入新数据源的驱动开发者（`docs/developers-guide/drivers/` + `modules/drivers/` 样板齐全）。阅读源码建议从 `src/metabase/query_processor.clj` 与 `src/metabase/lib/` 入手，这是全项目的骨架。
