# milvus-io/milvus 项目分析

> 星数 36500（快照 2026-10-24）｜主语言 Go (C++)｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：云原生分布式向量数据库，面向十亿级向量的高性能相似度检索与混合查询，是 AI 应用（RAG、图像/多模态检索）的核心基础设施。

| 项目 | 内容 |
|---|---|
| 名称 | milvus-io/milvus |
| 星数 | 36500（快照 2026-10-24；gh api 实测 2026-11-05 已达 46595，仍在快速增长） |
| 语言 | Go（控制面/数据面服务）+ C++（segcore 执行与向量索引） |
| 许可证 | Apache-2.0（`LICENSE`） |
| 默认分支 | master（gh api 确认），最近 push 2026-11-04 |
| 代码规模 | Go 非测试代码约 96.6 万行、测试代码约 101.9 万行（2403 个 `*_test.go`）；C++ 核心约 47.4 万行（`internal/core/src`） |
| 最新版本 | v3.0.5（2026-10-30），同时维护 v2.6.26 稳定线 |

## 2. 项目是什么，解决什么问题

Milvus 诞生于 2019 年（repo created 2019-09-16），由 Zilliz 主导开发、捐献给 LF AI & Data Foundation（`README.md`）。它解决的问题是非结构化数据（文本、图像、多模态）经 embedding 向量化后的高效组织与检索：在十亿级向量规模下，传统数据库无法支撑毫秒级 topK 近邻搜索，Milvus 通过专门的向量索引（ANN 算法）、分布式并行检索和标量/向量混合过滤来实现。

其核心主张是"fully-distributed and K8s-native"（`README.md`）：存储与计算彻底分离，所有组件无状态化，状态下沉到三类外部系统——元数据存 etcd/TiKV、消息走 Pulsar/Kafka/RocksMQ、数据落对象存储（S3/MinIO/Azure/GCS）。任一节点都可以独立水平扩缩容，故障后快速恢复。同时它也提供 Standalone（单机）和 Milvus Lite（`pip install` 内嵌）模式覆盖开发与轻量场景。

数据写入采用日志先行的流式架构：insert/delete 先进入 WAL（Write-Ahead Log），按 pchannel（物理通道）→ vchannel（虚拟通道）→ collection/partition 分发；后台异步把 WAL 数据物化成 Segment 落到对象存储，再由 compaction 合并、建索引。查询时走"增量段（growing，内存）+ 历史段（sealed，磁盘/索引）"的双路合并，兼顾实时性与吞吐。

查询能力上，除向量 ANN 搜索外，还支持标量过滤（内部有独立的 Bool 表达式语言与 ANTLR 语法 `Plan.g4`）、HybridSearch（多路召回）、聚合、全文/BM25（`internal/storage/bm25_stats_*.go`）、迭代器等，v3 已演进为一个"向量优先的通用查询引擎"。

## 3. 技术栈与依赖

- **语言**：Go 1.26（`go.mod`，module github.com/milvus-io/milvus）为主，C++（CMake + Conan 2.x）承载执行与索引核心，少量 Rust 工具链参与构建（`DEVELOPMENT.md`）。
- **关键 Go 依赖**（`go.mod`，326 行）：
  - `go.etcd.io/etcd/{api,client,server}/v3` + `tikv/client-go/v2`：元数据存储双后端
  - `apache/pulsar-client-go`、Kafka SDK、内置 Rocksmq（`pkg/mq/mqimpl/rocksmq/`）：消息队列
  - `minio/minio-go`、`aws-sdk-go-v2`、`azure-sdk-for-go`、`cloud.google.com/go/storage`：对象存储四家
  - `grpc` + protobuf：全部组件间通信；`prometheus/client_golang`：可观测性；`casbin`：RBAC 鉴权
  - `apache/arrow/go/v17`：列式内存格式，新 binlog（Parquet）读写的基础
  - `ants`（协程池）、`zap`（日志）、`roaring`（bitmap）、`antlr4-go`（表达式解析）
- **C++ 侧**：Knowhere 向量索引引擎（`internal/core/thirdparty/knowhere/`，独立仓库、版本由 `.github/workflows/update-knowhere-commit.yaml` 自动同步）、Google Test、simdjson、tantivy（全文索引）、rocksdb、jemalloc（`internal/core/thirdparty/` 目录）。
- **构建体系**：顶层 Makefile（Go + CGO + CMake 联动，`MILVUS_GO_BUILD_TAGS := dynamic,sonic,with_jemalloc`，支持 PGO）+ `internal/core/CMakeLists.txt` + Conan。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `cmd/` | 入口；`cmd/roles/roles.go` 定义全部可启角色（Proxy/QueryNode/DataNode/StreamingNode/RootCoord/QueryCoord/DataCoord/MixCoord） |
| `internal/` | 核心业务（36 个包，见下） |
| `pkg/` | 可复用库：`pkg/proto`（20+ 个 .proto 定义全部 RPC 与消息）、`pkg/mq`、`pkg/streaming`、`pkg/config`、`pkg/kv`、`pkg/metrics` |
| `client/` | 独立 Go SDK（`client/v3`，有自己的 go.mod，milvusclient/ 目录） |
| `internal/core/` | C++ 核心：segcore（段执行）、exec（向量化执行引擎）、index（Knowhere 封装）、storage、expr/plan |
| `configs/` | `milvus.yaml` 全量默认配置（约 2000+ 行，含 PyUDF、TiKV、SSL 等） |
| `tests/` | python/go/java/restful 四语言客户端测试 + `integration/`（Go 集成测试）+ chaos（混沌测试） |
| `build/`、`deployments/`、`ci/` | Docker 构建、helm 部署、CI 脚本 |
| `docs/`、`DEVELOPMENT.md` | 开发文档、贡献指南 |

`internal/` 内的关键包：`proxy`（网关）、`rootcoord`（DDL/元数据）、`datacoord`（段分配/压缩/索引/GC）、`querycoordv2`（查询调度）、`mixcoord`（三 coord 合体的单机形态）、`streamingcoord`+`streamingnode`（新流式架构的协调与 WAL 服务）、`querynodev2`（查询执行）、`datanode`+`flushcommon`（落盘）、`storage`/`storagev2`（对象存储与 binlog 格式）、`distributed`（各组件 gRPC 服务壳）、`compaction`、`cdc`（变更数据捕获）、`parser`（表达式 plan 解析）。

组织方式评述：单仓库多模块的"微服务单仓"——所有服务组件、公共库、SDK、测试、CI 都在一个仓库；`pkg/`（可导出库）与 `internal/`（私有）边界清晰，client/ 甚至独立成 Go module 以便外部引用。代码按"角色（coord/node）× 层次（接口/实现/gRPC 壳）× 版本代际（querycoordv2、querynodev2、storagev2）"三维组织，代际标记暗示了架构大版本的演进史。

## 5. 核心架构与关键实现

### 5.1 角色体系与存算分离拓扑

所有可执行角色在 `cmd/roles/roles.go` 中声明（`EnableProxy/EnableQueryNode/EnableDataNode/EnableStreamingNode/EnableRootCoord/EnableQueryCoord/EnableDataCoord` 及合并态 `MixCoord`），同一二进制通过环境变量裁剪成任意服务形态，Standalone 即全部组件共进程。控制面三协调器职责：rootcoord 管 DDL 与时间戳/ID 分配，datacoord 管段的分配、compaction、索引与 GC（其依赖关系明确写在 `internal/datacoord/README.md`：etcd 存元数据、msgstream 与 DataNode 通信、RootCoord 提供时间戳），querycoordv2 管段的加载布局与查询负载均衡。数据面执行节点 querynodev2/datanode/streamingnode 无状态可横向扩容；`internal/distributed/` 是各组件的 gRPC 网络壳，与业务实现解耦。

### 5.2 消息流：从 msgstream 到新 WAL 架构

旧一代消息抽象在 `pkg/mq/msgstream/`：`MsgStream` 接口（`msgstream.go`，AsProducer/Produce/Broadcast/Consume）统一了 Pulsar/Kafka/Rocksmq 三种后端（`mqwrapper/`），并定义了 Insert/Delete/TimeTick 等 tsMsg 的序列化与 repack 规则。新一代流式架构把消息层升级为真正的 WAL：`pkg/streaming/walimpls/wal.go` 定义了极简的 `WALImpls` 接口，实现插件化注册（`walimpls/impls/` 下 kafka/pulsar/rmq/wp）：

```go
// WALImpls is the underlying implementation for a wal that supports read and write operations.
type WALImpls interface {
    ROWALImpls // WALName / Channel / Close / Read(ScannerImpls)

    // Append writes a record to the log.
    // If ErrFenced is returned, the wal write operation is fenced forever
    // and cannot recover. The caller should close the wal and recreate
    // a new wal instance to recover the write operation.
    Append(ctx context.Context, msg message.MutableMessage) (message.MessageID, error)

    // Truncate truncates the wal to the given id (inclusive).
    Truncate(ctx context.Context, id message.MessageID) error
}
```

`ErrFenced` 语义（写入被永久隔离即必须重建实例）为日志主从切换提供了明确边界。服务层在 `internal/streamingnode/server/wal/`（README 明确 OpenerBuilder→Opener→WAL→Scanner 的生命周期分层，配套 interceptors、recovery、vchannel 管理），pchannel 的分配均衡由 `internal/streamingcoord/server/balancer/` 负责。消息协议本体在 `pkg/proto/messages.proto`（1012 行，定义 Insert/Delete/Flush/TimeTick/RecoveryBarrier/Txn 等全部流式消息体）。

### 5.3 存储引擎：Segment 模型与 binlog

数据以 Segment 为基本单元：`internal/querynodev2/segments/segment.go` 中 `SegmentType` 直接复用 `commonpb.SegmentState`——`Growing`（内存中的增量段，接收实时写入）、`Sealed`（已封口待建索引的历史段）、`L0`（只存 delete 的删除段，`segment_l0.go`）：

```go
const (
    SegmentTypeGrowing = commonpb.SegmentState_Growing
    SegmentTypeSealed  = commonpb.SegmentState_Sealed
)
```

落盘格式在 `internal/storage/`：自研 binlog（`event_header.go`/`binlog_reader.go` 等 event 流格式）+ 删除数据 `delta_data.go` + 统计文件；新一代写入器 `binlog_record_writer.go` 的 `PackedBinlogRecordWriter` 已切换为 Arrow RecordBatch/Parquet 打包格式，`internal/storagev2/`（loon_root.go、packed/）是新的存储格式试验层。对象存储抽象覆盖 MinIO/S3/Azure/GCP（`azure_object_storage.go`、`gcp_native_object_storage.go` 等）。C++ 侧对应 `internal/core/src/segcore/`（`ChunkedSegmentSealedImpl.h`、`SegmentGrowingImpl.h`、`InsertRecord.h`、`DeletedRecord.h`），Go 通过 CGO 驱动（`segments/segcore.go`、`cgo_util.go`）。

### 5.4 向量索引与执行引擎（C++ Knowhere 集成）

Knowhere 是独立维护的 C++ 向量索引引擎，Milvus 侧的封装层在 `internal/core/src/index/`：`vector/KnowhereEngine.cpp` 持有原生引擎句柄与 mmap 文件所有权，按"Builder/Artifact/Loader/Reader"四角色拆分（`vector/README.md` 有完整职责表）——`VectorMemBuilder` 内存建索引、`VectorDiskBuilder` + DiskANN 磁盘索引、`VectorMemLoader` 支持 materialize 与 mmap 两种加载、`VectorIndexReader` 统一 search/refine/迭代器查询入口；索引按家族注册（`VectorFamilies.cpp`），支持按物理类型分发。`index/growing/` 处理增量段实时索引，`index/scalar/` 与 tantivy 支撑标量/全文索引。Knowhere 版本独立演进，由 CI 工作流 `update-knowhere-commit.yaml` 自动同步提交。查询执行则演进为向量化引擎：`internal/core/src/exec/` 有 `Driver.cpp`/`Task.cpp`/`VectorHasher.cpp`（哈希聚合）与 operator 体系（`FilterBitsNode`、`ProjectNode`、`AggregationNode`、`MvccNode` 等），查询计划由 `internal/parser/planparserv2/`（ANTLR `Plan.g4`）从 Bool 表达式编译而来。

## 6. 构建与运行

```shell
# 完整构建（Go + C++，需要 cmake>=3.26、conan 2.x、go 1.26、rust 工具链）
make                                  # 默认 Release（DEVELOPMENT.md / Makefile）
CONAN_CMD=conan-1 make                # 在维护 release-2.5/2.6 分支时用 Conan 1.x
make milvus                           # 仅构建 milvus 二进制（Makefile target，未逐项确认）
make unittest                         # 单元测试（Makefile）
./bin/milvus run standalone           # 单机模式启动（cmd/milvus）
```

部署形态：Docker（`build/docker/`）、helm（`deployments/`）、docker-compose（`docker-compose.yml`）。Python 侧 `pip install -U pymilvus` + `MilvusClient("local.db")` 即可用 Milvus Lite（`README.md` Quickstart）。

## 7. 测试与工程质量

- 测试体量惊人：2403 个 `*_test.go`、约 102 万行测试代码，超过业务代码本身；使用 testify，风格以表驱动 + mock（`internal/mocks/` 生成 mock）为主。
- 分层测试：Go 单测内嵌于各包；`tests/integration/` 是 Go 写的集群级集成测试（compaction、coordrecovery、flushall、failpoint 等 40+ 场景目录）；`tests/python_client/`、`go_client/`、`java_client/`、`restful_client_v2/` 覆盖 SDK 端到端；`tests/chaos/` 做故障注入。
- CI：`.github/workflows/` 共 21 个工作流，包括主 CI（main.yaml）、daily/weekly-release、mac 构建、GPU 镜像、license 检查、python lint、`update-knowhere-commit.yaml`（自动同步 Knowhere 子模块版本），另有 mergify 与 codecov（`codecov.yml`）。
- 质量观察：OWNERS 文件遍布各目录（CODEOWNERS 式按模块负责制）；有 CODE_REVIEW.md、rules.go（ruleguard 自定义静态规则）；但巨型文件仍然存在（`internal/proxy/impl.go` 7651 行），CGO 边界和 36 个 internal 包的学习曲线陡峭。

## 8. 社区与维护状态

- 活跃度极高：gh api 显示最近 push 2026-11-04（几乎每日提交）；open issues 1549，forks 4328。
- Release 节奏：v3.0.5（2026-10-30）、v3.0.4（2026-10-26）、v2.6.26（2026-10-26）、v3.0.3（2026-10-14）——双版本线并行，v3 约两周一个 patch，v2.6 稳定线持续维护。
- 贡献者：contributors 分页接口单页 30（总数未确认，未做全量分页）；README 与 governance 显示 Zilliz 是主要贡献者，项目归 LF AI & Data Foundation 治理，有 MAINTAINERS/COMMITTERS/OWNERS 三级列表。
- 治理特征：issue 模板、claude-code-review.yml（AI 代码评审）等新式工程实践已落地。

## 9. 亮点与风险

**亮点**
1. 云原生架构教科书级实现：三类外部系统（etcd/MQ/对象存储）承载全部状态，任一角色无状态化、按需伸缩（`configs/milvus.yaml`、`internal/distributed/`）。
2. WAL 抽象设计干净：`walimpls.WALImpls` 四接口 + registry 插件注册 + fencing 语义，新增 MQ 后端成本极低（`pkg/streaming/walimpls/wal.go` 及 README）。
3. C++/Go 边界划分合理：Go 管分布式与调度，C++ 专注单机执行（segcore/exec）与索引（Knowhere），通过 CGO + 明确的 Builder/Loader/Reader 契约通信。
4. 测试文化强：测试代码量超过业务代码，覆盖单测/集成/SDK E2E/混沌四个层面。
5. 演进有序：querycoordv2/storagev2/wal 新架构与旧路径共存、可回退，大版本重构不停服迭代。

**风险/局限**
1. 体量与复杂度：百万行级代码、36 个 internal 包、三种 MQ + 四种对象存储 + 两种元数据后端的矩阵，部署与排障门槛高。
2. 部署依赖链长：完整集群至少需要 etcd、对象存储、消息队列三类外部组件（docker-compose 可一键但生产调优复杂）。
3. 巨型文件与 CGO：`internal/proxy/impl.go` 7651 行等超长文件增加维护成本；CGO 边界导致 Go 工具链（pprof/race）在 C++ 路径上受限。
4. 架构代际叠加：msgstream 与新 WAL、storage 与 storagev2 双轨并存，读代码需要分辨目标版本。

## 10. 总结评价

Milvus 是当前开源向量数据库中工程规模最大、架构最完整的代表：Go 编排的存算分离分布式骨架 + C++ 向量化执行与 Knowhere 索引引擎，配合极简的 WAL 抽象和 Segment 生命周期模型，把"十亿级向量、毫秒检索、K8s 原生"这个命题做成了可落地的系统；百万行代码配百万行测试的开发纪律在开源项目中罕见。它适合作为 AI 应用（RAG/语义搜索/多模态检索）的生产级向量存储，尤其在大规模、多租户、云上部署场景；对系统工程师而言，`pkg/streaming/walimpls/`、`internal/querynodev2/segments/`、`internal/core/src/exec/` 三条线是学习分布式日志、段式存储与向量化查询引擎的一流教材，但建议先读各包 README 与 `DEVELOPMENT.md` 再入代码，否则极易在包迷宫中迷路。
