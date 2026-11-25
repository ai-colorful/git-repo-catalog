# duckdb/duckdb 项目分析

> 星数 31000（快照 2026-10-24）｜主语言 C++｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：进程内嵌入式分析型（OLAP）SQL 数据库，"数据界的 SQLite"——把列式存储、向量化执行和高性能 OLAP 引擎塞进一个无服务端、零外部依赖的单机库。

| 项 | 值 |
|---|---|
| 名称 | duckdb/duckdb |
| 星数 | 31000（快照）；实际查询时已 43k（`gh api repos/duckdb/duckdb`，2026-11-13） |
| 主语言 | C++（C++17，`CMakeLists.txt`） |
| 许可证 | MIT（`LICENSE`，gh api 确认） |
| 最近活跃 | push 时间 2026-11-13（当天仍在提交）；最新 release v2.0.0（2026-10-21） |
| 贡献者 | 约 1000 人（contributors API 分页尾页 997，2026-11-13） |
| Fork / Open Issues | 4200 / 1205（gh api） |

## 2. 项目是什么，解决什么问题

DuckDB 是一个可直接嵌入应用进程的分析型数据库（`README.md`：runs in-process inside your application, so there is no separate server to manage and no external dependencies）。传统上数据分析要么用 SQLite 这类行式 OLTP 嵌入库（聚合扫描慢），要么部署 ClickHouse/Postgres 这类独立服务（运维重）。DuckDB 的答案是：把列式存储 + 向量化执行引擎做进进程内库，同时像 SQLite 一样 `DuckDB db("file.db")` 就能用。

它读写任意数据源：CSV、Parquet、JSON、Excel 可直接 `SELECT * FROM 'myfile.parquet'`（README 示例），本地或 S3/HTTP/Azure/GCS 远端均可；还能通过扩展直连 PostgreSQL、MySQL、SQLite 的原生协议，以及 Iceberg/Delta/DuckLake 等 lakehouse 格式（README "Connects to other databases" 一节，代码见 `extension/` 与 `src/include/duckdb/main/extension_entries.hpp` 的 autoload 列表）。

工程形态上它是"一个内核 + 多语言客户端"：核心是纯 C++ 库（`src/`），对外提供 C API（`src/include/duckdb.h`，10394 行，`duckdb_open/duckdb_connect/duckdb_query` 系列入口）以及基于 C API 的 Python/R/Java/Rust/Go/Wasm 客户端；CLI（`tools/shell/`）与 sqllogictest 测试器（`tools/sqllogic/`）也构建于同一内核。

数据组织上，DuckDB 默认把数据库存为单个文件（单文件块管理器 `src/storage/single_file_block_manager.cpp`），列式 RowGroup 存储 + 压缩段 + WAL + checkpoint，配合 MVCC 事务实现"分析负载 + 一定事务能力"的组合。

## 3. 技术栈与依赖

- **语言**：C++17（`CMakeLists.txt`、README Development 节），构建工具 CMake + Python3；支持 WASM 构建（`Makefile` 的 wasm_mvp/wasm_eh/wasm_threads 目标）
- **构建系统**：根 `CMakeLists.txt`（约 1300+ 行，大量 option：`ENABLE_SANITIZER`、`WASM_LOADABLE_EXTENSIONS`、`BUILD_COMPLETE_EXTENSION_SET` 等）+ 顶层 `Makefile`（make/make debug/make unit/make allunit）
- **内存分配器**：第三方 jemalloc 内置于 `third_party/jemalloc/`，默认在支持平台启用（`CMakeLists.txt:722` "Enable jemalloc as the memory allocator"），可覆盖 new/delete
- **第三方库**（全部 vendored 于 `third_party/`，共 28 个目录）：fmt（格式化）、re2（正则）、yyjson（JSON）、thrift（Parquet 元数据）、parquet（自研编解码）、zstd/lz4/snappy/brotli/miniz（压缩）、fsst（字符串压缩）、fast_float、fastpforlib/pdqsort/ska_sort/vergesort（算法）、hyperloglog/tdigest（近似统计）、concurrentqueue（无锁队列）、httplib（HTTP）、mbedtls（TLS/加密）、utf8proc（Unicode）、catch（单元测试框架）、pcg（随机数）
- **无运行时外部依赖**：第三方库全部静态编入，这是"单文件嵌入"哲学的工程保障

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `src/` | 内核本体：38MB、3363 个 cpp/hpp 文件、约 70.7 万行（含头文件） |
| `src/parser/` | SQL 解析（209 文件），含自研 PEG 解析器 `peg/` |
| `src/planner/` | 逻辑计划/绑定（238 文件） |
| `src/optimizer/` | 查询优化器（161 文件，约 50 个优化 pass） |
| `src/execution/` | 物理执行引擎（229 文件）：算子、表达式执行、join/hash 表 |
| `src/parallel/` | task 调度与 pipeline 并行框架（25 文件） |
| `src/storage/` | 列式存储、缓冲管理、压缩、checkpoint、WAL（138 文件） |
| `src/transaction/` | MVCC 事务实现（12 文件） |
| `src/catalog/`、`src/function/`、`src/common/`、`src/main/` | 目录/函数库/基础设施/对外 API（41/234/216/175 文件） |
| `src/include/` | 头文件集中放置（1672 个 .hpp，`duckdb/` 下按模块镜像源码布局） |
| `extension/` | in-tree 扩展（41MB、533 文件）：parquet、icu、json、tpch、tpcds、delta、autocomplete、core_functions |
| `tools/` | CLI shell、sqllogictest 运行器、C++/Swift 客户端示例 |
| `third_party/` | 28 个 vendored 第三方库（17MB） |
| `test/` | SQLLogicTest 用例 6503 个 .test 文件 + 402 个 C++ 测试文件（58MB） |
| `benchmark/`、`data/`、`scripts/`、`api_spec/` | 性能基准、测试夹具数据（大）、构建/发布脚本、API 规范 |

组织方式评述：典型的"集中式单内核 + 扩展插件"单体。内核按查询生命周期分阶段划目录（parser→planner→optimizer→execution→parallel→storage），头文件与实现分离但同仓管理；扩展有明确的 in-tree/OOTE（out-of-tree）分层治理（`extension/README.md` 详述三类：in-tree 静态链接、DuckDB Managed OOTE 官方签名分发、社区 External OOTE）。

## 5. 核心架构与关键实现

### 5.1 向量化执行引擎（精读）

执行模型为经典的"拉取式向量化算子流"：每个算子以 `DataChunk`（2048 行 × N 列）为输入输出单元。

- 向量尺寸常量：`STANDARD_VECTOR_SIZE = 2048`，编译期固定且必须为 2 的幂（`src/include/duckdb/common/vector_size.hpp`）
- Vector 三元组设计：数据指针 + validity mask（空值位图）+ selection vector（选择向量），过滤/切片零拷贝（`src/include/duckdb/common/types/vector.hpp:122`："It contains (1) a data pointer, (2) a validity mask, and (3) a selection vector"）
- `DataChunk` 注释明确其定位："intermediate representation used by the execution engine… represents a subset of a relation"（`src/include/duckdb/common/types/data_chunk.hpp`）
- 物理算子约 60 种（`src/include/duckdb/common/enums/physical_operator_type.hpp`：HASH_JOIN、PIECEWISE_MERGE_JOIN、IE_JOIN、ASOF_JOIN、PERFECT_HASH_GROUP_BY、PARTITIONED_AGGREGATE 等），实现位于 `src/execution/operator/`（scan/join/aggregate/order/projection/persistent 等 123 文件）
- 并行采用 pipeline + task 调度：`src/parallel/pipeline.cpp` 定义 Pipeline（源-算子链-汇），`pipeline_executor.cpp` 的 `PipelineExecutor::Execute(max_chunks)` 按 chunk 循环推数据；全局 `TaskScheduler` 线程池（`src/parallel/task_scheduler.cpp`，支持动态调整线程数与异步线程）
- 工程细节丰富：自适应过滤器（`src/execution/adaptive_filter.cpp`）、radix 分区聚合哈希表（`src/execution/radix_partitioned_hashtable.cpp`，支持聚合状态溢出磁盘 `aggregate_state_spilling.cpp`）、runtime filter 调度（`src/parallel/runtime_filter_schedule.cpp`）

### 5.2 列式存储与 MVCC 事务（精读）

- **RowGroup 组织**：默认 RowGroup 122880 行（`src/include/duckdb/storage/storage_info.hpp:23` `DEFAULT_ROW_GROUP_SIZE 122880ULL`），RowGroup 内按列切 segment（`src/storage/table/row_group.cpp`、`row_group_collection.cpp`、`column_data.cpp`）
- **压缩**：`src/storage/compression/` 提供十余种列压缩：bitpacking、dictionary（含 dict_fsst）、RLE、FSST 字符串压缩、ALP/ALPRD（浮点）、Chimp/Patas（浮点快速解压）、roaring（validity）、zstd、constant 等专有算法目录
- **缓冲管理**：`BufferManager` 支持 `SetMemoryLimit`（`src/include/duckdb/storage/buffer_manager.hpp:108`），`standard_buffer_manager.cpp` + `block_allocator.cpp` 管理块复用；临时文件/溢出由 `temporary_file_manager.cpp`、`temporary_memory_manager.cpp` 协调
- **持久化**：WAL（`src/storage/write_ahead_log.cpp`、`wal_replay.cpp`）+ checkpoint（`checkpoint_manager.cpp`、`src/storage/checkpoint/`），新写入先走本地存储 `local_storage.cpp`（optimistic_data_writer.cpp 乐观写）
- **MVCC**：事务实现集中在 `src/transaction/`（DuckTransaction、undo_buffer、commit/rollback state、WAL 写出）；行版本管理在存储层 `src/storage/table/row_version_manager.cpp`，update 通过 `update_segment.cpp` 原位更新 + 旧版本入 undo buffer（`src/include/duckdb/transaction/duck_transaction.hpp:121`："The undo buffer is used to store old versions of rows that are updated"）。定位为单写者多读者的分析型 MVCC，而非通用 OLTP 并发

### 5.3 优化器（规模统计为主）

`src/optimizer/optimizer.cpp`（643 行）以固定顺序串联约 50 个 pass，每个 pass 用 `RunOptimizer(OptimizerType::X, [&]{...})` 包裹且可被用户禁用。代表性 pass（`src/optimizer/` 目录逐一对应实现文件）：filter pushdown/pullup、join_order/（多 pass 连接顺序）、statistics_propagator + statistics/（基于统计的基数估计）、row_group_pruner、late_materialization、topn_optimizer（ORDER BY+LIMIT 转 TopN）、compressed_materialization（按需物化压缩列）、common_subplan_optimizer（CSE）、runtime filter 相关（cte_runtime_filter_pushdown 等）。

### 5.4 SQL 解析：libpg_query 已被自研 PEG 解析器取代（精读 + 重要发现）

旧版 DuckDB 基于派生自 PostgreSQL 的 libpg_query 解析器；本次 HEAD 快照中 `third_party/` 已**不存在 libpg_query**，解析器换成自研 PEG（Parsing Expression Grammar）体系，证据：

- `src/parser/peg/README.md`：完整描述 PEG 语法系统、`.gram` 语法文件（`src/parser/peg/grammar/statements/` 下每语句类型一个文件）、packrat 解析与 transformer 代码生成
- 实现：`parser_packrat.cpp`（packrat 记忆化解析）、`matcher*.cpp`（match 状态机）、`tokenizer/`、`transformer/`（AST 变换）
- 入口 `src/parser/parser.cpp`：`ParserOptions::Builtin()` 返回 `CompiledGrammar::DefaultGrammar()`，即语法预编译
- 扩展性设计：`dialect_extension.cpp`/`passthrough_dialect.cpp` 支持扩展自定义方言语法（旧架构难做到的能力）

### 5.5 扩展机制：静态链接 + 动态加载 + autoload

- **静态链接（in-tree）**：`extension/` 下 8 个扩展随主二进制编译（`extension/README.md`："statically linked into DuckDB's executables… automatically make them available"）；Parquet 扩展规模最大（111 个 cpp/hpp，`extension/parquet/`：reader/writer/decoder 三层）
- **动态加载**：`src/main/extension/extension_load.cpp`（982 行）负责外部扩展的安装与 dlopen 加载，含签名校验（Makefile 的 `verify-extension-signing` 目标）；加载后注册到 `linked_extension_registry.cpp` / `external_extension_provider.cpp`
- **autoload**：28 个官方扩展列入 autoload 白名单（`src/include/duckdb/main/extension_entries.hpp:1668-1677`：parquet、json、httpfs、icu、aws、azure、ducklake、iceberg、postgres_scanner、mysql_scanner、sqlite_scanner、motherduck、quack、ui 等），首次使用时自动下载加载；`extension_helper.cpp` 的 `CanAutoloadExtension`/`AllowAutoInstall` 控制白名单与自动安装
- **CSV/Parquet 直读**：CSV 读取器内置内核 `src/execution/operator/csv_scanner/`（scanner/sniffer/state_machine/encode），sniffer 含方言探测、表头探测、类型探测与类型精炼（`sniffer/dialect_detection.cpp`、`type_detection.cpp`、`type_refinement.cpp`）——实现 `SELECT * FROM 'file.csv'` 的零配置体验

## 6. 构建与运行

来自 `README.md` Development 节与 `Makefile`：

```bash
# 依赖：CMake、Python3、C++17 编译器
make            # release 构建（CLI 产物 build/release/duckdb）
make debug      # 非优化调试构建
make unit       # sqllogic 测试（依赖 debug 构建）
make allunit    # release 全量测试

# 基准测试
BUILD_BENCHMARK=1 BUILD_TPCH=1 make
./build/release/benchmark/benchmark_runner
```

CMake 关键开关（根 `CMakeLists.txt`）：`DISABLE_UNITY`、`BUILD_COMPLETE_EXTENSION_SET`、`WASM_LOADABLE_EXTENSIONS`（Wasm 扩展）、`ENABLE_SANITIZER/ENABLE_THREAD_SANITIZER/ENABLE_UBSAN`、`COVERAGE`；扩展配置走 vcpkg（`Makefile` 的 extension_configuration 目标生成 `build/extension_configuration/vcpkg.json`）。

## 7. 测试与工程质量

- **SQLLogicTest 体系**：6503 个 `.test` 文件（`test/sql/` 下 5491 个常规 + 816 个 `.test_slow`），另有 parquet/persistence/fuzzer/api/storage 等专项目录；运行器为 `tools/sqllogic/`，驱动脚本 `test/unittest.cpp` + `test/load.sql`
- **C++ 单元测试**：402 个测试源文件，框架为 vendored catch（`third_party/catch/`）
- **CI 覆盖面**（`.github/workflows/`，26 个 workflow）：Main.yml（主 CI）、NightlyTests.yml、ExtendedTests.yml、Windows.yml/OSX.yml/Android.yml/Swift.yml（全平台）、DockerTests.yml、Regression.yml（性能回归）、coverity.yml（静态扫描）、test_configs_*（多种 sanitizer/配置组合，见 Makefile）
- **质量基建**：.clang-format/.clang-tidy 强制风格；`.sanitizer-leak-suppressions.txt`/`.sanitizer-thread-suppressions.txt` 配套 ASan/TSan；`test/fuzzer/` + `test/ossfuzz/` 做模糊测试；codecov 配置（`.codecov.yml`）
- 观察小结：测试金字塔明显偏向端到端 SQL 一致性测试（sqllogic 占绝对大头），这是数据库类项目的主流且有效风格；性能回归（Regression.yml）单列说明团队重视执行器性能稳定性

## 8. 社区与维护状态

- **活跃度**：push 时间 2026-11-13（与下载当天同日），持续高频开发（gh api）
- **Release 节奏**：最新 v2.0.0（2026-10-21）；历史上 DuckDB 保持约每季度一个 minor 大版本的节奏（README 徽章指向 releases 页；本仓内无 CHANGELOG 文件，具体历史节奏未确认）
- **贡献者**：约 1000 人（API 分页推算）；商业主体 DuckDB Labs（Ducklabs.com）雇佣核心开发者全职开发，README "Support" 页指向其商业支持
- **生态位**：README 展示 Python/R/Java/Rust/Go/Wasm 全客户端矩阵、Discord 社区、MotherDuck（云上 DuckDB）等强生态；`extension_entries.hpp` 中 motherduck/ducklake/quack 等条目佐证官方生态扩张（云服务 + lakehouse 格式 + DuckDB 间互联协议）

## 9. 亮点与风险

**亮点**
1. 教科书级的向量化执行实现：2048 向量 + selection vector + validity mask 的零拷贝模型贯穿全栈，配 radix 分区哈希聚合、自适应过滤等精细优化（`src/execution/`、`src/include/duckdb/common/types/vector.hpp`）
2. 自研 PEG 解析器替代 libpg_query：语法文件化（`.gram`）、可编译、可做方言扩展，摆脱了 PG 解析器的历史包袱（`src/parser/peg/`）
3. 扩展治理成熟：in-tree/托管 OOTE/社区 OOTE 三层分类、签名分发、autoload 白名单，核心保持小而稳（`extension/README.md`、`src/main/extension/`）
4. 列压缩算法军械库：bitpacking/FSST/ALP/Chimp/Patas/roaring 等前沿算法均有生产化实现（`src/storage/compression/`）
5. 测试体系厚实：6503 个 sqllogic 用例 + 全平台 CI + fuzzing + sanitizer 矩阵（`test/`、`.github/workflows/`）

**风险/局限**
1. 规模庞大：内核约 70.7 万行 C++（含头），介入执行器/存储层的心智负担高；unity build 与模板重度使用会显著拉长编译时间
2. 事务模型为单写者多读者，不适合高并发 OLTP 场景（`src/transaction/` 设计定位使然）
3. 大版本波动：v2.0.0 于 2026-10 刚发布，C API/C++ API 存在迁移期风险（duckdb.h 与 capi_v2 并存可佐证 API 演进中）
4. 仓库体量对分析/构建不友好：`data/` 测试夹具占上百 MB，tarball 121MB，完整构建需编译 28 个 vendored 库 + 全部 in-tree 扩展

## 10. 总结评价

DuckDB 是过去十年数据库工程领域少有的"学术研究直接转化为工业级顶流"的范本：一个进程内、零依赖、MIT 许可的分析数据库，用 SQLite 的易用形态 delivering 列存 + 向量化 + MVCC 的 OLAP 性能。源码分层极清晰（parser/planner/optimizer/execution/parallel/storage/transaction 与查询生命周期一一对应），且保持了数据库内核项目里罕见的高测试密度与文档化注释。它适合三类人：数据工程师（开箱即用的 Python/CLI 数据分析利器）、嵌入式应用开发者（需要一个能跑复杂 SQL 的进程内存储）、数据库内核学习者（向量化执行与列存压缩的 best-practice 实现，其中 `src/execution/` 与 `src/storage/compression/` 尤其值得精读）。
