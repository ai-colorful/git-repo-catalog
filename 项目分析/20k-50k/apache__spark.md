# apache/spark 项目分析

> 星数 42500（快照 2026-10-24）｜主语言 Scala (Java/Python)｜档位 20k-50k｜分析日期 2026-11-06

## 1. 项目速览

一句话定位：Apache Spark 是大规模数据处理领域的统一分析引擎，以 RDD/DataFrame 抽象 + 内存计算为核心，覆盖批处理、流处理、机器学习与图计算，是大数据计算的工业标准之一。

| 项目 | 内容 |
|---|---|
| 名称 | apache/spark |
| 星数 | 44427（API 实时值，快照 42500）｜forks 29584｜open issues 751 |
| 主语言 | Scala（辅以 Java/Python/R） |
| 许可证 | Apache-2.0（`LICENSE`） |
| HEAD 版本 | 5.0.0-SNAPSHOT（`pom.xml`），默认分支 master |
| 语言基线 | Scala 2.13.18、Java 21（`pom.xml`） |
| 最近活跃 | 2026-11-05（GitHub pushed_at，分析前一天仍在 push） |
| 创建时间 | 2014-02-25（GitHub created_at） |

## 2. 项目是什么，解决什么问题

Spark 解决的是"大规模数据上做统一分析"的问题：在 Hadoop MapReduce 时代，迭代式与交互式负载被迫反复读写 HDFS、任务启动开销巨大。Spark 用**基于内存的弹性分布式数据集（RDD）**替代磁盘中间结果，用 **DAG 执行图**替代单轮 map/reduce，使多步数据流在同一引擎内流水线执行（`core/src/main/scala/org/apache/spark/Dependency.scala`、`core/src/main/scala/org/apache/spark/scheduler/DAGScheduler.scala`）。

在此引擎之上，Spark 提供了分层的高层工具栈：Spark SQL（SQL 与 DataFrame/Dataset API）、pandas API on Spark、MLlib（机器学习）、GraphX（图处理）、Structured Streaming（流处理），全部共享同一调度与执行内核（`README.md` 第 3-8 行）。"统一"是它的关键卖点：一套集群资源模型、一套优化器、一套 API，覆盖 SQL 分析、ETL、流计算与 ML 训练。

从源码体量看，项目重心早已是 Spark SQL：sql/ 目录约 169 万行代码文件（含测试与资源），超过 core/（31 万行）的五倍，Catalyst 优化器 + AQE 自适应执行 + Spark Connect 远程客户端是近年主线。HEAD 上另一个信号是 R API 已标注 "Deprecated"（`README.md` 第 4 行），同时出现了新的 sql/pipelines（声明式数据管道，`docs/declarative-pipelines-programming-guide.md`）与 common/variant（Variant 半结构化类型）模块。

Spark Connect（`sql/connect/`）代表 5.0 时代的架构方向：客户端与服务端通过 gRPC/protobuf 解耦（`sql/connect/common/src/main/protobuf/spark/connect/base.proto` 第 1334 行定义 `service SparkConnectService`），Python/Scala/JVM 客户端不再嵌入 JVM 进程，为交互式笔记本、多租户与云原生场景服务。对普通用户而言，这意味着从"把驱动跑在自己笔记本上"转向"驱动在集群侧、客户端只发计划"，资源需求与稳定性都由服务端承担。

## 3. 技术栈与依赖

- **核心语言**：Scala 2.13（主力，约 214 万行，含测试）+ Java（约 21 万行，网络/底层）+ Python（约 50 万行，PySpark 客户端与 worker）
- **构建**：Maven 多模块（约 50 个 `<module>`，`pom.xml`），自带 `build/mvn` 包装脚本；另有 SBT 辅助（`project/`）
- **关键依赖**：Hadoop（HDFS/YARN 生态，`hadoop-cloud/`）、Kubernetes（`resource-managers/kubernetes/`）、Netty（RPC 与 shuffle 传输，`common/network-common/`）、Hive Metastore（`sql/hive/`）、Py4J（Python↔JVM 桥，`python/pyspark/java_gateway.py`）、Kafka/Avro/Protobuf 连接器（`connector/`）
- **Python 侧**：`pyproject.toml` 管理，pytest 驱动测试，`cloudpickle` 序列化闭包
- **CI**：GitHub Actions 49 个 workflow（`.github/workflows/`），覆盖 Java 21/25、Python 3.11-3.15（含 nogil）

## 4. 目录结构与代码组织

规模说明：全仓共 31159 个归档条目、解压后 251MB；Scala 约 214 万行、Python 约 50 万行、Java 约 21 万行（统计自 tarball 实际文件，含测试）。`sql/` 一个目录就有 21090 个文件，是绝对重心。

| 目录 | 职责 | 规模（代码行，含测试） |
|---|---|---|
| `core/` | Spark 内核：RDD、调度器、shuffle、内存管理、网络 | 31.3 万 |
| `sql/` | Spark SQL：Catalyst、执行引擎、Hive、Connect、Pipelines | 168.9 万 |
| `python/` | PySpark：classic 桥接 + Spark Connect 客户端 + pandas API | 47.9 万 |
| `mllib/` + `mllib-local/` | DataFrame 风格 ML（spark.ml） | 15.0 万 |
| `common/` | 共享基础设施：network、unsafe 内存、variant、kvstore、config | 8.3 万 |
| `connector/` | Kafka/Avro/Protobuf/Kinesis/Ganglia/credential 连接器 | 5.9 万 |
| `resource-managers/` | YARN 与 Kubernetes 集群管理器 | 4.2 万 |
| `streaming/` | 传统 DStream 流式 API（legacy） | 3.5 万 |
| `R/` | SparkR（HEAD 已标记 deprecated） | 3.5 万 |
| `graphx/` | 图计算 | 0.9 万 |
| `examples/` | 各语言/组件官方示例（542 个文件） | 4.2 万 |
| `dev/` | 测试运行器（run-tests）、代码风格检查、依赖审计脚本 | 1.2 万 |
| `repl/`、`launcher/`、`assembly/`、`docs/`、`bin/`、`sbin/` | 交互式 Shell、启动器、打包、文档、启停脚本 | 0.9 万+ |

组织方式为**巨型 Maven 多模块单体**：按引擎分层（common → core → sql/mllib/graphx/streaming）+ 外围（connector、resource-managers），模块间依赖单向清晰（`pom.xml` 的 `<modules>` 列表）。sql/ 内部再细分 api/catalyst/core/hive/connect/pipelines 六个子工程（`sql/README.md` 明确说明五分法 + Connect/Pipelines 演进）。

## 5. 核心架构与关键实现

### 5.1 DAG 调度与 Stage/Task 模型（core/scheduler）

`DAGScheduler.scala`（5600 行）是执行引擎的顶层大脑。类头注释权威地描述了三级模型（第 86-98 行）：

```scala
 *  - Jobs (represented by [[ActiveJob]]) are the top-level work items submitted to the scheduler.
 *  - Stages ([[Stage]]) are sets of tasks that compute intermediate results in jobs, where each
 *    task computes the same function on partitions of the same RDD. Stages are separated at shuffle
 *    boundaries ... There are two types of stages: [[ResultStage]], for the final stage that
 *    executes an action, and [[ShuffleMapStage]], which writes map output files for a shuffle.
 *  - Tasks are individual units of work, each sent to one machine.
```

关键机制：以 ShuffleDependency 为边界把 RDD 血缘切成 ResultStage/ShuffleMapStage（`createShuffleMapStage`，第 737 行）；缓存追踪避免重算；基于数据本地性计算 preferred locations；FetchFailed 触发丢失 Stage 的重提交（第 115-119 行）。其下由 `TaskScheduler` + 可插拔 `SchedulerBackend`（YARN/K8s/Standalone/local）把 TaskSet 派发给 Executor，`scheduler/` 目录还含 FIFO/FAIR 调度（`SchedulingAlgorithm.scala`、`Pool.scala`）与事件总线（`LiveListenerBus.scala`）。入口 `SparkContext.scala`（3984 行）负责装配 SparkEnv、心跳、BlockManager 等运行时。

### 5.2 Shuffle 机制与统一内存管理（core/shuffle、core/memory）

Shuffle 默认实现是排序式：`shuffle/sort/SortShuffleManager.scala` 在 `registerShuffle`（第 91-102 行）里按条件三选一——分区数小于 `spark.shuffle.sort.bypassMergeThreshold` 走 bypass 合并、序列化器支持重定位走 serialized shuffle、否则普通 SortShuffleWriter；map 端输出经 `IndexShuffleBlockResolver.scala` 建立分区索引供 reduce 端按块拉取。`ShuffleBlockPusher.scala` 实现了 push-based shuffle（map 端主动推块给 reduce），配合 `common/network-shuffle/` 的外部 shuffle 服务。shuffle/ 目录整体按 local/sort/streaming 三包组织，`ShuffleManager.scala`/`ShuffleWriter.scala`/`ShuffleReader.scala` 定义 SPI 式扩展点。

内存管理采用统一池模型：`memory/UnifiedMemoryManager.scala` 把 (堆 - 300MB) × `spark.memory.fraction`（默认 0.6，`core/src/main/scala/org/apache/spark/internal/config/package.scala` 第 496-505 行）划为 execution 与 storage 共享池，storage 预留区默认占比 0.5。二者可互相借用，execution 不足时执行 `maybeGrowExecutionPool`（第 156-171 行）驱逐缓存块回收 storage 借走的内存，这是 Spark 1.6 之后取代静态分区的核心设计。

### 5.3 Catalyst 优化器与执行引擎（sql/catalyst、sql/core）

Catalyst 是基于不可变表达式树的规则优化框架。优化管线在 `sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/Optimizer.scala`（第 167-304 行）以数十个 Batch 串联：Finish Analysis → Inline CTE → Operator Optimization（fixedPoint 迭代）→ Subquery → Join Reorder（CBO）→ Push Down/Push Up 系列 → Check Cartesian Products 等；`optimizer/` 目录下有 50 个独立规则文件（如 `CostBasedJoinReorder.scala`、`InjectRuntimeFilter.scala`、`DecorrelateInnerQuery.scala`）。

物理执行在 `sql/core/src/main/scala/org/apache/spark/sql/execution/`：`WholeStageCodegenExec.scala` 实现 Tungsten 风格全阶段代码生成（把流水线内的算子折叠为单个 Java 函数以消除虚函数与中间行对象），UnsafeRow 二进制行格式与堆外内存由 `catalyst/expressions/` 与 `common/unsafe/` 支撑。3.x 引入的 AQE 自适应执行在 `execution/adaptive/`（约 6700 行）：`AdaptiveSparkPlanExec.scala` 在 shuffle 边界用真实统计替换逻辑计划，`CoalesceShufflePartitions.scala`/`OptimizeSkewedJoin.scala` 等规则在运行时合并小分区、拆分倾斜 join。

### 5.4 PySpark 双通道桥接（python/pyspark）

PySpark 同时维护两条通路。**classic 通道**：`java_gateway.py`（第 28-29 行 import py4j 的 ClientServer/JavaGateway）在客户端拉起 JVM；`daemon.py` 用 `os.fork()`（第 222 行）管理 Python worker 进程池；`worker.py` 是 Executor 侧的 Python 计算进程，通过 `serializers.py`（pickle/UTF8 流）与 JVM 交换数据，闭包经 `cloudpickle/` 序列化——这条进程间 socket 通路是 Spark 1.x 以来 Python 集成的基础。**Connect 通道**：`pyspark/sql/connect/` 提供与 `sql/connect/server` 对接的 gRPC 客户端（proto 定义在 `sql/connect/common/src/main/protobuf/spark/connect/`，含 relations/expressions/types/commands/ml/pipelines），DataFrame 逻辑计划在客户端组装、序列化后发给远端执行；`sql/connect/client/jdbc` 与 `client/jvm` 则提供纯 JVM 客户端。目录里 `sql/classic/` 与 `sql/connect/` 并存，两套 DataFrame API 由 `pyspark/sql/dataframe.py` 统一门面。`inprocess/` 是新增的进程内运行时（无 JVM fork 场景）。

### 5.5 连接器与资源管理（边界）

`connector/` 下 Kafka（0-10 + SQL + assembly）、Avro、Protobuf、Kinesis、Ganglia、credential-aws、profiler 独立成模块；`resource-managers/` 含 yarn 与 kubernetes/core；`common/network-yarn` 补 YARN 网络层。数据源生态的开放接口（DataSource V2）在 `sql/core/.../datasources/`，第三方存储（Iceberg/Delta 等）以外部包形式接入，不在本仓。

## 6. 构建与运行

来自 `docs/building-spark.md` 与 `dev/`：

```bash
# 构建（build/mvn 自带 Maven，自动拉取依赖）
./build/mvn -DskipTests clean package                       # building-spark.md:58
./build/mvn -Pyarn -Dhadoop.version=3.5.0 -DskipTests clean package   # :87
./build/mvn -Pkubernetes -DskipTests clean package          # :109
./build/mvn scala:cc                                        # Scala 增量编译 :132
# 测试
./dev/run-tests                                             # dev/run-tests 脚本
./build/mvn test
# Python 侧（pyproject.toml + pytest）
```

运行：`bin/spark-submit`、`bin/spark-shell`（Scala REPL，`repl/`）、`bin/pyspark`、`bin/spark-sql`（`sql/hive-thriftserver` 提供 JDBC/ODBC 服务）。`conf/` 提供模板配置。Spark Connect 场景可另起服务端，或用 `python/pyspark/sql/connect/local_server.py` 在本地进程内拉起 Connect 服务做开发调试。

## 7. 测试与工程质量

- **规模**：全仓约 2383 个 `*Suite.scala` 文件、841 个 Python 测试文件；core/sql/pyspark 内 test 相关文件超 1082 个。Scala 测试统一用 ScalaTest `AnyFunSuite` 风格（`AGENTS.md` "Scala Test Base Classes" 节）。
- **SQL 金样测试**：`SQLQueryTestSuite` 及变体管理 golden file（`.sql.out` 由工具再生、禁止手改，`AGENTS.md` Development Notes 节），是 SQL 行为回归的骨干。
- **CI**：`.github/workflows/` 共 49 个 workflow，含 master/branch-4.x/branch-4.3/4.2/4.1/4.0 六个分支的调度器（branch4x_scheduler.yml 等）、Java 21/25、Python 3.11-3.15（含 3.14 nogil）、Maven 构建、覆盖率、codegen JDK、UDS 传输等矩阵（`README.md` 第 29-80 行状态表可见全貌）。
- **工程细节**：`build/` 目录自带 Maven/Scala 安装器保证构建可复现；`dev/` 下含依赖漏洞审计与 git-hooks 脚本；error-class 错误码体系（`python/pyspark/errors/`、SQL 侧 error-classes.json）统一用户可见错误消息。
- **代码质量**：Databricks Scala 编码规范 + scalastyle（`scalastyle-config.xml`）+ pre-commit（`.pre-commit-config.yaml`）；`AGENTS.md` 明确 100 字符行宽 lint、非 ASCII 检查等纪律；Spark Connect proto 有独立 README 约束演进。

## 8. 社区与维护状态

- **活跃度极高**：pushed_at 2026-11-05（分析前一天），GitHub Actions 每日构建 master 与 4.x 系列分支。
- **贡献者**：GitHub contributors API 末页序号 331（按默认分支提交计的 API 快照，实际历史贡献者更多，未精确确认）。
- **发布机制**：不走 GitHub Releases（releases API 返回空），按 Apache 惯例经官方发行站与 Maven Central 发布（`README.md` Maven Central 徽章），版本节奏每年一个 minor（4.0→4.3 分支并存，5.0 已在 master 开发）。
- **治理**：Apache 顶级项目，`.asf.yaml` 配置仓库行为，PR 模板 + CONTRIBUTING.md 流程完备。
- **多分支并行维护**：master（5.0）与 branch-4.3/4.2/4.1/4.0 四条发布线同时由 CI 调度器覆盖（`.github/workflows/branch40_scheduler.yml` 至 `branch43_scheduler.yml`），说明旧版本仍在持续接补丁。

## 9. 亮点与风险

**亮点**
1. 引擎分层教科书级：DAGScheduler/TaskScheduler/SchedulerBackend 三层解耦，shuffle 与内存均为 SPI 式扩展点（`core/src/main/scala/org/apache/spark/scheduler/`、`core/src/main/scala/org/apache/spark/shuffle/ShuffleManager.scala`），十年演进仍保持内核概念稳定。
2. 优化器+执行持续自我革新：Catalyst 规则管线（50 个独立规则文件）→ AQE 运行时优化 → 全阶段代码生成，代表关系引擎在 JVM 上的工程极限（`sql/catalyst/.../optimizer/`、`sql/core/.../execution/adaptive/`）。
3. Spark Connect 架构前瞻：gRPC/protobuf 解耦客户端，为云原生与多语言生态铺路，Python 侧 classic/connect 双通道平滑过渡（`python/pyspark/sql/connect/`）。
4. 工程纪律与测试矩阵罕见地严格：49 个 CI workflow、金样 SQL 测试、多 JDK/多 Python 版本矩阵。
5. 仓库自带 `AGENTS.md`/`CLAUDE.md` AI 协作开发指南（含 PR 流程、测试基类选择、lint 自检脚本），是大型开源项目拥抱 agent 开发的先行范例。

**风险/局限**
1. 体量巨大（源码约 290 万行、50 个 Maven 模块），贡献者上手成本极高，构建+全量测试需重型机器。
2. 遗留包袱并存：`streaming/`（DStream）与 R API 在 HEAD 已事实边缘化（README 标 R 为 Deprecated），MLlib RDD API 与 spark.ml 双轨长期并存，新用户选型易混淆。
3. Scala 停留在 2.13，未跟进 Scala 3；JVM 语言生态吸引力相对下降，重心已明显向 Python/Connect 倾斜。
4. 资源管理深度绑定 Hadoop 版本矩阵，升级 Hadoop/K8s 客户端兼容成本高（`hadoop-cloud/`、`resource-managers/`）。
5. SQL 行为面极宽（解析器、 ANSI 模式、金样测试依赖大量生成文件），跨版本升级存在计划不稳定与行为变更风险（`docs/migration-guide.md` 单独成册）。

## 10. 总结评价

Apache Spark 是大数据计算领域当之无愧的基础设施级项目：42k+ 星并非潮流驱动，而是十余年作为"统一分析引擎"事实上标准的沉淀。HEAD（5.0.0-SNAPSHOT）显示它仍在快速演进——Spark Connect 的客户端-服务端解耦、声明式 Pipelines、Variant 类型与 Python 生态（含 nogil 支持）是当下主线，而经典内核（DAG 调度、SortShuffle、统一内存、Catalyst/AQE）则展现了惊人的架构稳定性。适合数据平台工程师作为生产引擎、分布式系统学习者精读 core/ 与 sql/ 的调度-优化-执行三层设计；对想理解"如何在 JVM 上做关系引擎与向量化/代码生成"的开发者，Catalyst + WholeStageCodegenExec + AQE 这条线是业内最完整的公开范本。门槛方面，阅读源码建议从 `DAGScheduler` 类注释与 `sql/README.md` 入手，避免直接陷入 290 万行细节。
