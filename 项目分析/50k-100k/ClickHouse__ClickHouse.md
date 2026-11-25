# ClickHouse/ClickHouse 项目分析

> 星数 50560（快照 2026-10-24）｜主语言 C++｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

ClickHouse 是开源的**列式 OLAP 数据库管理系统**，主打"实时生成分析报表"，以极致的聚合查询吞吐著称。

| 项目 | 事实 |
|---|---|
| 名称 | ClickHouse/ClickHouse（官方仓库，ClickHouse Inc. 维护） |
| 星数 | 50735（gh api 实测 2026-11-04，任务参数快照 50560） |
| 语言 | C++（src/ 10,061 个文件；另有 rust/ 工作区） |
| 许可证 | Apache-2.0（`LICENSE`） |
| 最近活跃 | 最近 push 2026-11-03（gh api）；最新 release v26.10.2.103-stable（2026-10-24） |
| 规模 | HEAD tarball 381MB 压缩 / 约 1.03GB 解压，95,989 个条目（实测）；GitHub 仓库约 15GB（含历史与 submodule） |
| 发布节奏 | 月度版本，2026 年已发 26.1–26.10 共 10 版，其中 26.3、26.8 为 LTS（`CHANGELOG.md`） |

## 2. 项目是什么，解决什么问题

ClickHouse 解决的是**海量事件/日志/埋点数据的实时交互式分析**问题。传统行式 OLTP 库在十亿行级别做 GROUP BY、聚合、过滤时吞吐不足，数据仓库批量 ETL 又无法做到秒级可见；ClickHouse 用列式存储 + 稀疏主键索引 + 向量化执行，把单机聚合打到每秒数亿行（README 定位："column-oriented database management system that allows generating analytical data reports in real-time"，`README.md`）。其团队 2024 年发表的 VLDB 论文《ClickHouse - Lightning Fast Analytics for Everyone》被官方列为架构入门材料（`CONTRIBUTING.md`）。

其核心设计是 **MergeTree 家族存储引擎**：写入时数据按分区键切成一个个不可变的数据 part（相当于 LSM-Tree 的 memtable flush 产物），后台线程持续把小 part 合并成大 part；查询时通过分区剪枝 + 主键稀疏索引 + 跳数索引把扫描范围缩到最小（`src/Storages/MergeTree/MergeTreeData.h`）。这套"不可变 part + 后台 merge + 变体引擎（Replacing/Summing/Aggregating/Collapsing 等）"的模型是它高吞吐写入与高压缩比的根源。

**分布式是内建的而非外挂的**：`ReplicatedMergeTree` 通过 ZooKeeper/ClickHouse Keeper 的日志复制实现多副本强一致（`src/Storages/MergeTree/ReplicatedMergeTreeQueue.cpp`），`Distributed` 表引擎做分片路由（`src/Storages/StorageDistributed.cpp`），协调组件 ClickHouse Keeper 基于 NuRaft 自研（`src/Coordination/`），不依赖第三方 Raft 库的进程。

项目由 ClickHouse Inc. 商业化运营（ClickHouse Cloud），开源版本与云版本共享核心代码（`CMakeLists.txt` 中有 `CLICKHOUSE_CLOUD` 开关）。2026 年 26.6 版是其开源十周年特别版（`CHANGELOG.md`）。2025-2026 年间还在向两个方向扩展：可观测性（clickstack/OTel 生态，docs 有大量 observability 指南）与 AI/向量检索（src/Functions 中含向量与 GPU 相关代码，`src/GPU/`、`AI_POLICY.md`）。

## 3. 技术栈与依赖

- **主体语言**：C++（src/ 10,061 文件，实测 tarball）；查询函数库 `src/Functions` 1,223 文件、聚合函数 `src/AggregateFunctions` 253 文件。
- **构建**：CMake ≥3.25 + Ninja（`CMakeLists.txt`：`cmake_minimum_required(VERSION 3.25)`，`project(ClickHouse LANGUAGES C CXX ASM)`），配 `.clang-format`/`.clang-tidy` 统一代码风格。
- **自研 base 库**：`base/`（1,208 文件）内 vendor 了 Poco（网络/文件系统框架）、pcg-random、glibc-compatibility（glibc 版本无关性）、widechar_width 等，减少系统依赖（`base/`、`base/CMakeLists.txt`）。
- **第三方依赖以 git submodule 管理**：`.gitmodules` 中 157 个 submodule，contrib/ 在 tarball 中仅有 1,111 个占位条目（内容不含在 tarball 内），典型如 libuv、aws-sdk、arrow 等大数据生态库。
- **Rust**：`rust/` 为 Cargo workspace（含 chcache 编译缓存工具等，`rust/README.md`、`rust/VENDOR.md`），用于渐进引入 Rust 组件。
- **协调层**：NuRaft（`src/Coordination/RaftServerConfig.h`、`ReadBufferFromNuraftBuffer.h`）。
- **CI 基建**：自研 "praktika" Python CI 框架（`ci/praktika/` 114 文件 + `ci/jobs/` 267 文件）。

## 4. 目录结构与代码组织

顶层（tarball 实测条目数）：

| 目录 | 条目数 | 职责 |
|---|---|---|
| `tests/` | 48,763 | 功能测试（0_stateless 40,092 个）、集成、性能、fuzz、jepsen |
| `docs/` | 33,207 | 官方文档源码，含 zh/ru/pt-BR/ko/ja 各 3,018 个多语言镜像 |
| `src/` | 10,061 | 数据库全部核心 C++ 代码 |
| `base/` | 1,208 | 自维护基础库（Poco fork、随机数、glibc 兼容层） |
| `contrib/` | 1,111 | 第三方库 submodule 占位（157 个，`.gitmodules`） |
| `ci/` | 647 | 自研 praktika CI 框架与构建脚本 |
| `utils/` | 315 | 外围工具 |
| `programs/` | 266 | 单一二进制 `clickhouse` 的子命令实现：server/client/keeper/benchmark/compressor/local/disks/git-import 等（实测目录清单） |
| `docker/`、`packages/`、`benchmark/` | — | 打包与基准 |

`src/` 内部按层次组织（实测条目数）：`Storages` 1,750（存储引擎，其中 MergeTree 380 文件）、`Functions` 1,223（SQL 函数）、`Processors` 1,185（算子与查询计划）、`Common` 1,103（HashTable/线程池等基础设施）、`Interpreters` 874（查询解释与上下文）、`Parsers` 670（SQL 词法/语法）、`IO` 470、`DataTypes` 285、`Analyzer` 209（新查询计划表示）、`Core` 195（Block/Field 核心类型）。

组织方式评述：**超大单体仓库（monolithic monorepo）**——核心引擎、函数库、客户端、Keeper、文档、测试、CI 全在一个仓库；第三方库用 submodule 而非包管理器拉入 contrib/；对编译产物复用（ccache）与 Rust 渐进混编有专门基建。这种结构适合对性能与构建链路有极致控制需求的数据库项目，但代价是克隆/构建门槛极高。

## 5. 核心架构与关键实现

### 5.1 MergeTree 存储引擎（写多读快的根基）

`MergeTreeData` 是 MergeTree 家族的基类，同时继承 `IStorage`（表接口）与 `IBackgroundOperation`（后台合并调度），声明体 2,561 行，实现 `MergeTreeData.cpp` 单文件 **16,035 行**（实测），管理活跃 part 集合（boost::multi_index 多索引容器）、part 状态机（Precommitted/Committed/Outdated 等）与元数据：

```cpp
// src/Storages/MergeTree/MergeTreeData.h:233（有删节）
class MergeTreeData : public WithMutableContext,
                      public IStorage,
                      public IBackgroundOperation
{
public:
    using DataPartState = MergeTreeDataPartState;  // part 状态机
    using DataPartStates = std::initializer_list<DataPartState>;
    using DataPartStateVector = std::vector<DataPartState>;
    struct DataPartStateAndInfo { DataPartState state; ... };
    ...
};
```

数据物理上分为 **Wide**（每列一个 .bin/.mrk 文件对）与 **Compact**（全部列共存一个文件）两种 part 格式，小 part 自动落 Compact 省 inode，超过 `min_bytes_for_wide_part`（默认 10MB，DEBUG 构建为 1KB；`MergeTreeSettings.cpp`）转 Wide：对应 `MergeTreeDataPartWide.{h,cpp}` 与 `MergeTreeDataPartCompact.{h,cpp}`。稀疏索引粒度 `index_granularity` 默认 **8192 行**（`MergeTreeSettings.cpp`：`DECLARE(UInt64, index_granularity, 8192, ...)`），并支持自适应粒度（`MergeTreeIndexGranularityAdaptive.cpp`）。

写路径由 `MergeTreeDataWriter.cpp`（1,512 行）落盘；后台 `MergeTreeDataMergerMutator.cpp`（879 行）负责 merge 与 mutation（数据变更也是通过重写 part 实现的异步任务）。读路径是"索引分析 → 生成 mark range → 并行读取"：`MergeTreeSelectProcessor.cpp`（852 行，含 PREWHERE 列裁剪优化，其头文件注释了 `ChunkAndProgress` 语义）包装为 `MergeTreeSource`（`MergeTreeSource.h:12`，`class MergeTreeSource final : public ISource`）接入管线。跳数索引（skip index）有 BloomFilter、text、JSON 子列等实现（`MergeTreeIndexBloomFilter*.cpp`）。26.x 新增了 Patch Parts（轻量补丁 part，`MergeTree/PatchParts/`）与 UniqueKey/DeleteBitmap（`MergeTree/UniqueKey/`）等新机制。

### 5.2 基于日志的副本复制（ReplicatedMergeTree）

复制不靠主从协议，而是**所有副本共享一条 ZooKeeper/Keeper 上的操作日志**，每个副本各自消费日志保持一致。日志条目共 12 种类型（`src/Storages/MergeTree/ReplicatedMergeTreeLogEntry.h`）：

```cpp
enum Type {
    EMPTY,
    GET_PART,       /// Get the part from another replica.
    ATTACH_PART,    /// Attach the part, possibly from our own replica...
    MERGE_PARTS,    /// Merge the parts.
    DROP_RANGE,     /// Delete the parts in the specified partition...
    REPLACE_RANGE,  /// Drop certain range of parts and replace them by new ones
    MUTATE_PART,    /// Apply one or several mutations to the part.
    ALTER_METADATA, /// Apply alter modification according to global /metadata...
    ...
};
```

`ReplicatedMergeTreeQueue.cpp`（3,045 行）负责日志消费、去重（`deduplication_block_ids` 块级去重，写入幂等）与任务调度；`ReplicatedMergeTreeAttachThread`、`ReplicatedMergeTreeCleanupThread` 等后台线程（各独立文件）处理恢复与清理。写入块哈希去重 + part 哈希校验使复制天然容错。

### 5.3 查询执行：QueryPlan → Pipeline → Processor 双层模型

执行层采用"查询计划（逻辑）→ 算子管线（物理）"两层。逻辑层 `Processors/QueryPlan/`（445 文件）表达 Join/Aggregation/Exchange 等步骤；物理层把计划编译成 `QueryPipeline`（`src/QueryPipeline/QueryPipeline.h`，50 文件）——由 `IProcessor`（输入/输出 Port 的有向图节点）组成的数据流图，由 `Processors/Executors/` 下的 `PullingPipelineExecutor`、`PushingPipelineExecutor`、`CompletedPipelineExecutor`（及 Async 变体）调度执行。数据单元是列式的 `Block`（`src/Core/Block.h`，231 行，由 `ColumnWithTypeAndName` 组成，`src/Core/ColumnWithTypeAndName.h`），`Field`（`src/Core/Field.h`，1,003 行）是动态单值类型。

**查询分析已统一到新 Analyzer**：2026-10 的 26.10 版本彻底删除了旧 analyzer——"The query analysis that preceded the analyzer is removed... `enable_analyzer` became obsolete"（`CHANGELOG.md` 26.10 Backward Incompatible Change，PR #120311/#121465），AST 级谓词下推也改由计划优化器承担。

### 5.4 分布式查询与 Keeper

分布式查询由 `DistributedPlanExecutor.cpp`（2,375 行）驱动：把查询计划切分为带 `LogicalExchangeStep` 的子计划下发到各分片（shard），通过 `RemoteQueryExecutor`（`src/QueryPipeline/RemoteQueryExecutor.cpp`）与 `NativeCompressedSource/Sink` 在节点间传输块数据；`src/Interpreters/Cluster.h`（450 行）定义 Cluster/ Shard/ Replica 拓扑，`ClusterProxy/` 处理分布式表的路由转发；26.x 还引入了 Stateless Workers 与 Streaming Exchange（`src/Server/StatelessWorker/`、`src/Server/DistributedQuery/`，见 `DistributedPlanExecutor.cpp` 的 include 列表）。集群元数据协调依赖 ClickHouse Keeper（基于 NuRaft 的 ZooKeeper 兼容实现，`src/Coordination/`），Keeper 既是复制的协调者，也可作为独立服务部署（`programs/keeper/`）。

## 6. 构建与运行

官方构建命令在 docs 站点（`CONTRIBUTING.md` 指向 https://clickhouse.com/docs/development/build ），仓库内可确认的工程事实：

- **配置**：CMake ≥3.25 + Ninja（`CMakeLists.txt`），需要 submodule（`git clone --recursive` 或先 `git submodule update --init`， contrib/ 为 157 个 submodule）。
- **构建**（官方文档惯用命令，与 `CMakeLists.txt` 约束一致）：`cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=RelWithDebInfo && ninja -C build`，产物为单一多合一二进制 `clickhouse`（`programs/main.cpp` 分发 server/client/keeper/benchmark 等子命令，目录清单实测）。
- **运行**：`clickhouse server --config-file=... --daemon`（配置样例 `programs/server/config.xml`，其中多个 config.d 文件是指向 `tests/config/` 的符号链接，tarball 实测）；单机试用 `clickhouse local`。
- **安装**：`curl https://clickhouse.com/ | sh`（`README.md`）。
- **测试**：功能测试由 `clickhouse-test` 驱动 `tests/queries/0_stateless/`（40,092 个用例，实测），集成测试在 `tests/integration/`（7,079 项）。

## 7. 测试与工程质量

- **测试规模罕见地大**：tarball 中 tests/ 共 48,763 条目——0_stateless 功能测试 40,092 个（SQL 驱动、期望结果文件模式）、integration 7,079、performance 879、benchmarks 256、fuzz 47、jepsen（分布式一致性验证，`tests/jepsen.clickhouse`），另有 `tests/stress/`（实测统计）。
- **CI 体系重工业级**：`.github/workflows/` 下 30 个 workflow（master.yml、pull_request.yml、nightly.yml、nightly_fuzzers.yml、nightly_jepsen.yml、nightly_extended_performance.yml、weekly_cfi.yml、auto_releases.yml、backport_branches.yml、revert_broken_prs.yml 等，实测清单），配合自研 praktika 框架（`ci/praktika/`）与 647 个 ci 文件。
- **代码质量基建**：`.clang-format`/`.clang-tidy` 强制风格；`.githooks/`；`AI_POLICY.md` + `AGENTS.md`/`.claude/`/`.cursor`/`.agents` 表明已正式拥抱 AI 辅助开发并出台 AI 使用政策。
- 工程债务痕迹：`MergeTreeData.cpp` 16k 行的超大文件、`Context.h` 2,443 行的全局上下文，说明核心路径经过多年快速迭代，重构仍在进行（Analyzer 替换旧解释器即为近例）。

## 8. 社区与维护状态

- **活跃度极高**：最近 push 2026-11-03（分析前一天，gh api）；8,767 open issues、9,214 forks（gh api 2026-11-04）。
- **发版节奏**：严格月度——2026 年 26.1(01-29)、26.2、26.3(LTS)、…、26.10(10-22) 共 10 版，每版配套公开发布电话会议与幻灯片（`CHANGELOG.md` 目录 + `README.md` Release Call 章节）。
- **贡献者**：GitHub API 分页统计约 312 名非匿名贡献者（gh api，2026-11-04，API 只返回主要贡献者，实际规模更大，精确数未确认）；`AUTHORS` 文件将作者列表指向服务器 `system.contributors` 表（`AUTHORS`）。
- **治理**：公司化运营（ClickHouse Inc.），外部 PR 需签 CLA（`CONTRIBUTING.md` Legal Info 章节）；核心开发由公司工程师主导，`ci/` 与发布流程高度公司化。
- **社区渠道**：官方 Slack/Telegram/YouTube/Meetup 网络活跃（`README.md` 有 2026 年 10-11 月全球 meetup 列表）。

## 9. 亮点与风险

**亮点**

1. **存储引擎设计经典且持续进化**：不可变 part + 后台 merge 的 MergeTree 模型（`src/Storages/MergeTree/`）兼顾了写吞吐、压缩率与扫描性能，Wide/Compact 双格式与自适应粒度（默认 8192）等细节都是生产打磨的结果。
2. **复制与分布式原生内建**：日志式副本协议（12 种日志条目，`ReplicatedMergeTreeLogEntry.h`）+ 块级去重，配合自研 Keeper（NuRaft，`src/Coordination/`），不依赖外部组件即可搭集群。
3. **测试与 CI 规模顶级**：4 万+ 功能测试、jepsen 一致性验证、fuzz/CFI 周常化（`tests/`、`.github/workflows/`），对数据一致性底线的投入肉眼可见。
4. **执行引擎现代化完成**：旧解释器在 26.10 全面退役，统一为 QueryPlan/Pipeline/Processor 架构（`CHANGELOG.md`、`src/Processors/`），架构账已还清大半。
5. **文档与国际化**：五语言文档镜像各 3,018 文件（实测），月度 release call 视频化，社区透明度高。

**风险/局限**

1. **构建与贡献门槛极高**：157 个 submodule（`.gitmodules`）、完整构建需大量内存与磁盘（官方用专门构建镜像），个人贡献者很难在本地走通全流程。
2. **核心代码体量与复杂度**：单文件 16k 行（`MergeTreeData.cpp`）、2.4k 行的 `Context.h`，阅读与修改成本高，改动容易牵一发动全身。
3. **迭代快带来兼容成本**：26.10 直接移除旧 analyzer 并删设置项（`CHANGELOG.md`），混合版本集群升级需严格按官方顺序执行，运维方需保持跟进。
4. **公司主导的单极治理**：路线图、CI、发布均由 ClickHouse Inc. 控制，社区对方向的影响力有限（CLA 要求见 `CONTRIBUTING.md`）。

## 10. 总结评价

ClickHouse 是开源分析型数据库中工程强度最高的项目之一：以 MergeTree 列式引擎 + 向量化管线为核心，用日志式复制和自研 Keeper 把分布式做成了产品默认能力，再用 4 万级功能测试与月度发版维持一个高速演进又相对可靠的迭代节奏。对**需要秒级交互式查询海量事件数据的团队**（可观测性、用户行为分析、监控、广告报表），它几乎是该量级的默认选项；对**数据库内核学习者**，`src/Storages/MergeTree/`（不可变 part 与后台 merge 的教科书实现）、`src/Processors/`（现代算子管线）与 `src/Storages/MergeTree/ReplicatedMergeTreeLogEntry.h`（日志复制协议）是最值得精读的三条主线，建议配合官方 VLDB 2024 论文与架构文档食用。

---
*分析基于 2026-11-04 抓取的 HEAD tarball（381MB，95,989 条目）选择性解压的 src/Core、src/Storages/MergeTree、src/Interpreters、src/QueryPipeline、base/ 等目录与 GitHub API 元数据；contrib/（submodule 占位）与 src/Functions 等仅做规模统计，未逐文件阅读。*
