# pingcap/tidb 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Go｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

TiDB 是 PingCAP 开源的分布式 NewSQL 数据库：计算层（本仓库）与存储层（TiKV，独立仓库）分离，100% 兼容 MySQL 协议与语法子集，通过 Raft 多副本保证强一致，通过 TiFlash 列存副本实现 HTAP（行存事务 + 列存分析混合负载）。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | pingcap/tidb | README.md |
| 星数 | 39500（快照）；2026-11-14 API 实测 40734，fork 6293 | gh api repos/pingcap/tidb |
| 主语言 | Go（go.mod 声明 go 1.27.3） | go.mod |
| 许可证 | Apache-2.0（企业级功能同样开源，见 README "Open Source Commitment"） | LICENSE、README.md |
| 规模 | pkg/ 下非测试 Go 代码约 92.6 万行 + 测试约 68.7 万行（3870 个 .go 文件） | 实测统计 |
| 最近活跃 | 最近 push 2026-11-13（前一天），open issues 7613 | gh api |
| 最新 release | v8.5.8 LTS（2026-08-27）、v7.5.8 维护版（2026-09-17） | gh api releases |
| 贡献者 | ≥392 人（contributors 分页统计） | gh api contributors |

## 2. 项目是什么，解决什么问题

关系型单机数据库（MySQL）在数据量超过单机容量、写入需要水平扩展时遇到瓶颈；而传统的分库分表方案丢失了跨分片事务与全局索引能力，NoSQL 又牺牲了 SQL 与强一致。TiDB 的解法是把 MySQL 的"计算 + 存储"拆开：**TiDB Server（本仓库）是无状态的计算层**，负责解析 SQL、优化、执行分布式计划；**TiKV（独立仓库）是分布式行存 KV**，用 Raft 复制三副本提供强一致；两者之间用 Range 切分与 PD 调度实现弹性伸缩（README.md "Key Features" 节）。

本仓库即计算层的完整实现：MySQL 线协议接入（`pkg/server/conn.go`）→ 自研 parser（`pkg/parser`）→ 优化器（`pkg/planner`）→ 分布式执行器（`pkg/executor` + `pkg/distsql`）→ 通过 `tikv/client-go` 访问 TiKV（`pkg/store/driver/tikv_driver.go`）。事务采用 Percolator 派生的两阶段提交（prewrite/commit），由 client-go 实现，本仓库在 `pkg/store/driver/txn/txn_driver.go` 封装提交时间戳上界检查等接口。

HTAP 是 TiDB 的第二张牌：TiFlash 作为 TiKV 的列存 Learner 副本（Multi-Raft Learner 实时同步），优化器可把分析查询以 MPP（Massively Parallel Processing）方式推给 TiFlash 执行，`pkg/store/copr/mpp.go` 中 `buildBatchCopTasks*` 与 `kv.TiFlash` 存储类型即是该链路的入口（pkg/store/copr/mpp.go:74-153）。

兼容性策略上，TiDB 目标兼容 MySQL 8.0：语法由自研 yacc 文法（parser.y）覆盖， wire protocol 由 `pkg/server` 直接实现 MySQL 协议包解析，使现有 MySQL 驱动/ORM 无需改造即可接入（README.md "MySQL Compatibility"）。

商业路径：PingCAP 以 TiDB Cloud（全托管）与 TiDB Operator（K8s 部署）构成云原生产品线，README 明确"All source code is available ... including enterprise-grade features"，即企业特性也回灌 Apache 2.0 主仓（README.md）。

## 3. 技术栈与依赖

- **语言**：Go 1.27.3（go.mod），构建系统 Bazel（MODULE.bazel/WORKSPACE）为主、Makefile 包装（Makefile）。
- **存储访问**：`github.com/tikv/client-go/v2`（TiKV RPC 客户端，go.mod；`pkg/store/driver/tikv_driver.go` 引用），`github.com/pingcap/kvproto`（TiKV/PD 的 protobuf 协议）。
- **元数据/协调**：etcd 客户端（`pkg/store/etcd.go`）、PD 类型（`pkg/store/pdtypes/`）。
- **云对象存储**（备份/导入用）：AWS S3、Azure Blob、GCS、阿里云 OSS 等 SDK（go.mod require 列表，367 个直接依赖条目）。
- **表达式/矢量执行辅助**：`golang.org/x/exp`、Apache Arrow（列式数据交换，br/ 与 TiFlash 客户端场景）。
- **测试**：go-sqlmock、testkit 自研测试框架（`pkg/testkit/`）、failpoint 注入（`github.com/pingcap/failpoint`，pkg/planner/optimize.go import 可见）。
- **依赖治理**：`DEPS.bzl` + `ThirdPartyNotices.txt` + `errors.toml`（错误码集中管理，Makefile `errdoc` 目标）。

## 4. 目录结构与代码组织

| 目录 | 职责（依据目录内文件实测） |
|---|---|
| `pkg/parser/` | 自研 SQL parser：parser.y（18620 行 yacc 文法）+ lexer.go + ast/ + 生成 parser.go（28616 行）；自带独立 go.mod 可单测（pkg/parser/go.mod） |
| `pkg/planner/` | 优化器：逻辑/物理计划、CBO（core/cost/）、find_best_task.go、规则/算子子包（core/rule、core/operator） |
| `pkg/executor/` | 执行器（496 个文件），内部核心算子在 `internal/exec/executor.go` |
| `pkg/distsql/` | 分布式请求构建（request_builder.go）与结果归并（select_result.go） |
| `pkg/store/` | 存储适配层：driver/（TiKV 驱动）、copr/（coprocessor + MPP 客户端）、gcworker/、mockstore/unistore/（内嵌单机 TiKV 模拟器） |
| `pkg/kv/` | 存储抽象接口：Transaction/Client/Snapshot/Storage（kv.go:264-867） |
| `pkg/session/` | 会话与 SQL 生命周期（session.go 6195 行） |
| `pkg/server/` | MySQL 协议服务端（conn.go、server.go 1403 行）+ HTTP 状态 API（http_status.go） |
| `pkg/ddl/` | 分布式 DDL（286 个文件，owner 选举 + job 队列） |
| `pkg/domain/` | 进程内域管理：加载信息 schema、统计信息、DDL worker 等 |
| `pkg/expression/` | 表达式体系与内建函数（195 个文件） |
| `pkg/infoschema/`、`pkg/statistics/`、`pkg/bindinfo/` | 元数据缓存、统计信息（CBO 依据）、执行计划绑定 |
| `br/`、`lightning/`、`dumpling/` | 同仓三件套：分布式备份恢复、快速导入、全量导出工具（br/README.md） |
| `tests/` | 集成测试：integrationtest（271 个 .test 用例，MySQL-test 风格 r/ 结果文件）、realtikvtest、globalkilltest、graceshutdown 等 |
| `cmd/tidb-server/` | 服务器 main 入口 |
| `AGENTS.md`、`llms.txt`、`CLAUDE.md` | 面向 AI 编码代理的仓库指南（AGENTS.md 186 行，含 MUST/SHOULD 规范与 ExecPlan 流程） |

组织方式评述：典型 Go 单体大仓（mono-repo），按"协议层→parser→planner→executor→kv 抽象→存储 driver"的分层依赖单向排列；周边工具（br/lightning/dumpling）与内核同仓维护但各自有独立构建目标；parser 保持独立 go.mod 以便单独生成与测试。PL 中还有 `.agents/skills/` 目录与 AGENTS.md 呼应，反映其工程对 AI 辅助开发的前沿投入。

## 5. 核心架构与关键实现

### 5.1 自研 parser（yacc 文法驱动）

TiDB 没有用现成 SQL 引擎，而是自研 parser：`pkg/parser/parser.y` 以 LALR(1) 文法定义 MySQL 方言（18620 行），由 goyacc（pkg/parser/goyacc/）生成 28616 行的 parser.go；入口在 `yy_parser.go` 的 `Parse` 系列。文法即兼容性的"单一事实来源"——支持哪些语法直接看 parser.y。hint 有独立文法（hintparser.y），AST 定义在 `pkg/parser/ast/`。`Makefile` 中 `parser_yacc` 校验目标确保生成物一致。

### 5.2 优化器：CBO + 规则改写 + 计划缓存

入口 `pkg/planner/optimize.go:148 Optimize()`：先查计划缓存（optimizeCache, :217），未命中走 optimizeNoCache（:230）——AST 转逻辑计划（`planner/core/`）→ 规则改写（`core/rule/`）→ 物理算子枚举（core/exhaust_physical_plans.go）→ 基于统计信息的代价选择（core/find_best_task.go、core/cost/）。统计信息来自 `pkg/statistics/`，SQL 绑定来自 `pkg/bindinfo/`。HTAP 的关键决策点也在这里：优化器按代价选择走 TiKV 行存还是 TiFlash 列存/MPP（pkg/store/copr/mpp.go 中按 `kv.TiFlash` 存储类型构建 batch cop task）。

### 5.3 存储抽象与分布式事务（Percolator 派生 2PC）

`pkg/kv/kv.go` 定义了四层接口：`Client`（:353，按 key range 取 Snapshot）、`Snapshot`（:815）、`Transaction`（:264）、`Storage`（:867）。TiKV 实现由 `pkg/store/driver/tikv_driver.go` 桥接到 `tikv/client-go/v2`；事务提交模型（prewrite → commit 两阶段、AsyncCommit、悲观锁）在 client-go 中，本仓库侧的证据：`pkg/store/driver/txn/txn_driver.go:286` 暴露 `SetCommitTSUpperBoundCheck`（commitTS 上界校验钩子），`pkg/kv/keyflags.go`/`option.go` 定义 prewrite 标志与事务选项，`pkg/sessiontxn/isolation/base.go` 实现隔离级别语义。测试环境用 `pkg/store/mockstore/unistore/`（内嵌 MVCC + lockstore 的单机 TiKV 模拟器，含 mvcc.go 的 prewrite/commit 实现）无需真实集群即可跑事务语义测试。

### 5.4 MySQL 协议层与执行器

`pkg/server/conn.go:1157 Run()` 是每连接主循环：读包→分发（:1555 `handleQuery` 处理 COM_QUERY，conn_stmt.go 处理 prepared statement）→ `pkg/session/session.go`（6195 行）驱动 parse/planner/executor 流水线 → 结果按 MySQL 线协议写回。执行器为火山模型 + 批量/向量化的混合（`pkg/executor/internal/exec/executor.go`），算子通过 `pkg/distsql/request_builder.go` 把 TableScan/IndexScan 下推为 coprocessor 请求发往 TiKV/TiFlash（`pkg/store/copr/coprocessor.go` 含结果缓存 coprocessor_cache.go）。

## 6. 构建与运行

```bash
# 构建 TiDB Server（Makefile default 目标）
make server            # 产物 bin/tidb-server（cmd/tidb-server/）
make server-admin-check  # 带管理检查构建

# 开发校验（Makefile "dev" 目标拆解）
make check             # lint/revive/golangci-lint + errdoc + license + bazel ABI 检查
make parser_yacc       # 校验 parser.y 与生成物一致
bazel test //pkg/...    # Bazel 单测（CI 用 prow.tidb.net，README 徽章）

# 运行
bin/tidb-server        # 单机模式（需配合 TiKV/PD 组成完整集群；可用 TiUP playground 本地起全集群，README Quick Start）
```

集成测试：`tests/integrationtest/run-tests.sh`（MySQL-test 风格，271 个 .test 用例 + r/ 期望结果）；真实 TiKV 集群测试在 `tests/realtikvtest/`。

## 7. 测试与工程质量

- **测试规模**：pkg/ 内 1765 个 _test.go 文件、约 68.7 万行测试代码，测试/源码比约 0.74（实测统计）；另有 tests/ 顶层集成套件（clusterintegrationtest、globalkilltest、graceshutdown、readonlytest、cncheckcert 等，见 tests/ 目录）。
- **风格**：单元测试就近放置 + 自研 testkit（pkg/testkit/）做 DB 级断言；集成测试采用 golden file（.test + r/ 对照，tests/integrationtest/）；failpoint 注入故障（pkg/planner/optimize.go import pingcap/failpoint）。
- **CI**：三层——GitHub Actions（.github/workflows/ 下 bazel 文件生成与编译检查 6 个 workflow）、Jenkinsfile、Prow（prow.tidb.net，README 徽章，merged-tidb-build 等 job）；OWNERS 文件（K8s 风格 approvers/reviewers 机制）驱动 Prow 自动指派评审。
- **质量观察**：错误码集中登记（errors.toml + Makefile errdoc）、依赖白名单（DEPS.bzl）、静态检查含自定义 forbidigo/makezero 等 linter（go.mod）；AGENTS.md 明确"Correctness first""不手改生成物"等硬约束。

## 8. 社区与维护状态

- **活跃度**：最近 push 2026-11-13（分析前一天），open issues 7613，fork 6293——持续高强度迭代（gh api）。
- **release 节奏**：双 LTS 线并行——v8.5.x 当前 LTS（v8.5.8，2026-08-27）+ v7.5.x 维护线（v8.5.4 于 2025-11-27、v7.5.8 于 2026-09-17），约每季度补丁（gh api releases）。
- **贡献者结构**：contributors ≥392 人（API 分页 last=392）；治理采用 SIG（特别兴趣小组）制：OWNERS_ALIASES 将 go.mod 等敏感路径指派给 `sig-critical-approvers-dep`，成员名单同步自 pingcap/community 仓库（OWNERS 头部注释）。
- **治理与商业**：PingCAP 主导开发 + TiDB Cloud 商业化，同时 README 强调企业功能全部开源；配套 dev-guide（pingcap.github.io/tidb-dev-guide）与 community 仓库定义贡献流程。
- **前沿实践**：仓库内建 AI 协作设施（AGENTS.md、llms.txt 文档索引、.agents/skills/、PLANS.md 的 ExecPlan 流程），在头部开源项目中属于先行者。

## 9. 亮点与风险

**亮点**
1. 自研 parser + 优化器 + 执行器全栈自主实现，MySQL 方言兼容的文法级控制力强（pkg/parser/parser.y、pkg/planner/）。
2. 存储/计算分离 + KV 接口抽象干净：`pkg/kv/kv.go` 四层接口使 TiKV、unistore 模拟器、mockstore 可互换（pkg/store/mockstore/）。
3. HTAP 落地在优化器与 copr 层真实打通：MPP 任务构建、TiFlash 副本读策略、聚合下推均有代码可循（pkg/store/copr/mpp.go）。
4. 工程化程度业界标杆：Bazel+Prow+OWNERS+golden-file 集成测试+错误码治理，68 万行测试代码护航。
5. AI 时代工程实践先行：AGENTS.md/ExecPlan/skills 体系让大型代码库对人类与 AI 贡献者都更可导航。

**风险/局限**
1. 体量巨大（pkg 约 92 万行 + 367 个直接依赖），普通贡献者上手曲线陡峭；构建强依赖 Bazel 与工具链自举（Makefile check-setup）。
2. MySQL 兼容是"子集"而非全兼容：语法覆盖由 parser.y 决定，个别存储引擎特性/权限细节有差异（README 表述为"compatible with MySQL 8.0"而非 100%）。
3. 事务与调度的核心逻辑在仓外（tikv/client-go、PD），仅读本仓库无法完整理解 2PC 与 Region 调度实现。
4. 本地裸跑需 PD+TiKV+TiDB 三组件（或 TiUP），单二进制仅适合测试/嵌入场景；unistore 模拟器缓解但语义与真实集群不完全等价（未确认细节差异程度）。

## 10. 总结评价

TiDB 是中国开源基础软件走向世界的代表作，也是"计算-存储分离 + Raft 强一致 + HTAP"架构路线最完整的开源实现；本仓库（计算层）以 92 万行 Go 代码呈现了一个工业级分布式 SQL 引擎的完整剖面——自研 parser、CBO 优化器、分布式执行器、事务适配层、MySQL 协议层与配套工具链一应俱全。适合分布式数据库工程师、数据库内核研究者与 Go 大型工程维护者精读：入门路径建议按 `pkg/server/conn.go`（协议）→ `pkg/parser/yy_parser.go`（文法）→ `pkg/planner/optimize.go`（优化）→ `pkg/distsql/` + `pkg/store/copr/`（分布式执行）→ `pkg/store/driver/txn/`（事务接入）的调用链逐层下钻；想要完整事务与存储语义，还需配合 tikv/client-go 与 TiKV 仓库一起读。
