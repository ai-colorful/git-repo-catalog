# qdrant/qdrant 项目分析

> 星数 35651（gh api 实测 2026-11-21，任务快照 39500）｜主语言 Rust｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：Qdrant 是用 Rust 编写的高性能向量搜索引擎与向量数据库，以"过滤友好的向量检索"（filterable HNSW）为核心卖点，服务于语义搜索、RAG、推荐等 AI 应用。

| 项目 | 内容 |
|---|---|
| 名称/仓库 | qdrant/qdrant |
| 星数 | 35651（fork 2833，open issues 826，gh api 2026-11-21） |
| 语言 | Rust（edition 2024，rust-version 1.98，Cargo.toml） |
| 许可证 | Apache-2.0（LICENSE） |
| 版本 | v1.20.2（2026-11-16 发布，gh api） |
| 最近活跃 | master 最近 push 2026-11-20，持续高频活跃 |
| 维护方 | Qdrant 公司（商业本体，另有 Qdrant Cloud 托管服务） |

## 2. 项目是什么，解决什么问题

Qdrant 提供生产级服务：存储、搜索、管理"点"（point = 向量 + 任意 JSON payload），通过 REST（actix-web，默认 6333 端口）和 gRPC（tonic，6334 端口）双协议暴露 API（README Getting Started、src/actix/api/ 与 src/tonic/api/）。README 明确其差异化定位："tailored for extended filtering support"——即在带过滤条件的向量搜索上做到低延迟高召回，这是相对朴素"先过滤后 ANN"方案的核心优势。

它解决的问题：神经网络/嵌入模型输出的向量需要高效相似度检索，而真实业务查询往往还带元数据过滤（如"只搜某用户、某颜色、某价格段的相似商品"）。传统做法要么过滤破坏 ANN 图结构（召回崩塌），要么全量暴力扫描（延迟爆炸）。Qdrant 通过 payload 索引 + 图搜索过程中动态判断可达性（filterable HNSW）来同时兼顾两者。

仓库自称是原 Go 原型的 Rust 重写（任务参数 DESC），目前版本已迭代到 1.20，功能面覆盖：稠密/稀疏向量混合检索（lib/sparse、lib/bm25）、多向量查询（Recommend/Discover/Context 等复杂查询，见 lib/segment/src/vector_storage/query/）、量化压缩、GPU 建图（lib/gpu）、分布式集群（Raft + 分片复制）。

新趋势是 Qdrant Edge（README "Qdrant Edge" 一节、lib/edge/）：进程内嵌入版，无服务器无共识，面向移动端/边缘设备，含 Python 绑定（lib/edge/python）与 FFI（lib/edge/ffi），支持与服务端同步。这使 Qdrant 从纯服务端数据库扩展为"服务端 + 嵌入式"双形态。

## 3. 技术栈与依赖

- 语言与运行时：Rust（2024 edition），异步基于 tokio；主 HTTP 框架 actix-web，gRPC 框架 tonic（根 Cargo.toml 依赖段）
- 共识：tikv/raft-rs（Cargo.toml 中 raft-proto 以 git rev pin 引用），slog 日志桥接
- 关键库（Cargo.toml）：parking_lot（锁）、serde/schemars（类型与 OpenAPI schema 生成）、prometheus（指标）、jsonwebtoken + rustls（RBAC/JWT 与 TLS）、validator（请求校验）
- 序列化/接口：prost（gRPC proto 在 lib/api/src/grpc/proto/），OpenAPI 3.0 规范在 openapi/ 与 docs/redoc
- 构建/工程化：clippy.toml（自定义 lint 阈值）、rustfmt（nightly）、rusty-hook（git hook）、ast-grep 规则（tools/ast-grep/sgconfig.yml）、release profile 开 fat LTO + codegen-units=1（Cargo.toml [profile.release]）
- 绑定：lib/edge/python（PyO3 体系，发布为 qdrant-edge crate，经 publish/amalgamate.py 组装，CLAUDE.md）

## 4. 目录结构与代码组织

Cargo workspace：二进制主 crate 在 src/，库 crate 全部在 lib/（Cargo.toml [workspace] members，根 CLAUDE.md 给出官方 code map）。

| 目录 | 职责（依据 CLAUDE.md 与源码核对） |
|---|---|
| src/ | 服务器入口 main.rs、actix REST handlers（src/actix/api/，24 个文件）、tonic gRPC handlers（src/tonic/api/）、consensus.rs（Raft 循环，1766 行）、settings.rs、快照/调试工具 |
| lib/storage | 集群级状态：content_manager/toc（TableOfContent 持有全部 collection）、dispatcher.rs（元数据操作走共识）、rbac/（JWT 访问控制）、snapshots |
| lib/collection | 单个 collection：hash_ring.rs（点路由到 shard）、shards/replica_set（复制）、shards/transfer + wal_delta.rs（分片迁移）、update_handler/update_workers（WAL 与优化器调度）、grouping/recommendations/discovery（复合查询） |
| lib/shard | 单个本地分片：segment_holder/、optimizers/（合并/建索引/vacuum）、proxy_segment/（优化期间承接写入）、query/（通用查询引擎）、wal.rs |
| lib/segment | 单个段（存储与索引的基本单元）：types.rs（核心配置与过滤类型）、payload_storage/、id_tracker/、vector_storage/、index/（HNSW、payload 索引、稀疏索引）、spaces/（SIMD 距离度量）、segment_constructor/ |
| lib/quantization | 向量量化：PQ（encoded_vectors_pq.rs）、二值量化（encoded_vectors_binary.rs）、标量量化（encoded_vectors_u8.rs/tq）、turboquant/、kmeans.rs |
| lib/wal | 预写日志：segment.rs、mmap_view_sync.rs |
| lib/sparse、lib/bm25、lib/posting_list | 稀疏向量、BM25 全文、倒排链路 |
| lib/api | REST/gRPC 类型与 proto 定义 |
| lib/gpu | GPU 辅助 HNSW 建图 |
| lib/edge | Qdrant Edge 嵌入式分片（python/ffi 子 crate，无服务器/共识） |
| lib/common/* | 通用工具、cancel、issues 等子 crate |
| openapi/、docs/、config/、tests/ | OpenAPI 规范、文档、默认配置（config/config.yaml）、Python 集成测试 |

组织方式评述：标准的 Rust workspace 分层 crate 架构，依赖方向严格单向（api → storage → collection → shard → segment），每层一个领域概念（集群状态/集合/分片/段），边界清晰；segment 与 quantization 拆成独立 crate 也利于单独基准测试。2118 个 .rs 文件、根 CLAUDE.md 自带 code map，工程治理成熟。

## 5. 核心架构与关键实现

### 5.1 Filterable HNSW（核心卖点）

HNSW 实现在 lib/segment/src/index/hnsw_index/。图持久化为 graph.bin + links.bin，另有压缩格式 links_compressed.bin / links_comp_vec.bin（hnsw.rs 常量定义）；graph_layers_healer.rs 提供图修复，pipnn/ 子目录为另一种链接策略。

lib/segment/src/index/hnsw_index/graph_layers.rs 模块文档列出 SEARCH-LAYER 的多个变体：常规 beam search（层 0）、`search_on_level_acorn`（ACORN-1 算法，用于硬过滤场景）、beam=1 的 `search_entry_on_level`（高层定向）、以及向量内嵌图（inline storage）变体。ACORN-1 实现（graph_layers.rs L157 起）在过滤条件收紧时不依赖预先取回的过滤 id 集，而是沿 1-hop/2-hop 邻居按预算（hop1_limit/hop2_limit）探索并现场打分过滤，避免高选择性过滤下候选枯竭：

```rust
// Limits are per every explored 1-hop or 2-hop neighbors, not total.
// This is necessary to avoid over-scoring when there are many
// additional graph links.
let m = self.get_m(level);
let (hop1_limit, hop2_limit, capacity);
if m != 0 {
    hop1_limit = m;
    hop2_limit = m;
    capacity = hop1_limit * hop2_limit.min(16);
} else {
    // `m = 0` graphs only have payload links: take all
}
```

打分链路在 lib/segment/src/index/hnsw_index/point_scorer.rs，文件头用 ASCII 图定义了三层组合：FilteredScorer（含 ConditionChecker、deleted 点/向量位图）→ RawScorer（访问模式）→ QueryScorer（度量 + 复杂查询 RecoQuery/DiscoverQuery/ContextQuery）。过滤条件先经 lib/segment/src/index/query_optimization/（optimized_filter，过滤规划器）估算选择性，再决定走"预取过滤 id 位图 + 图内跳过"或 ACORN 式现场过滤。

### 5.2 Segment 与存储后端（mmap 优先）

段是存储与索引基本单元。lib/segment/src/vector_storage/dense/ 提供 AppendableMmapDenseVectorStorage（可变段，mmap chunked vectors）、immutable_dense_vectors.rs（不可变段）、volatile_dense_vector_storage.rs（内存易失段，用于 WAL 回放临时）、graph_inline_dense_vector_storage.rs（向量直接内嵌 HNSW 图节点，配合 inline storage 降低随机 IO）。appendable_dense_vector_storage.rs L31-64 确认其基于 ChunkedVectors<T, MmapFile> + BitvecFlags<MmapFile>（删除位图也是 mmap 管理），并支持 populate 预热整段页。payload 侧对应 payload_storage/（mmap/in-memory 双实现）、id_tracker/（mutable/immutable/disk/compressed 多种 id 映射）。

这种"可变段 appendable mmap + 不可变段只读 mmap"的设计是 Qdrant 低内存占用、快启动的关键：无小文件随机读，全部顺序 chunk 化 mmap。lib/shard/optimizers/ 后台把小可变段合并成大不可变段并建 HNSW 索引，期间 proxy_segment/ 代理写入（WAL 缓存重放），保证优化不阻塞写。

### 5.3 分布式：Raft 元数据 + 分片/复制

元数据（collection/shard 拓扑）走 Raft 强一致：src/consensus.rs（1766 行）封装 tikv/raft-rs 的 Node/Config、tick 循环与 entry 应用；lib/storage/content_manager/consensus/ 处理集群操作，dispatcher.rs 将元数据操作提交共识（CLAUDE.md）。数据面则按 collection → shard（hash_ring.rs 路由，lib/collection/src/hash_ring.rs）→ replica_set（lib/collection/src/shards/replica_set/，多副本读写一致性）拆分；shard 迁移通过 shards/transfer/ + wal_delta.rs 增量同步。相比 Milvus 的 etcd/Pulsar/MinIO 多组件微服务架构，Qdrant 分布式是单二进制内建（Raft 库内嵌，无外部协调服务），部署复杂度显著更低；代价是集群规模上限与元数据吞吐依赖单 Raft 组。

## 6. 构建与运行

来自根 CLAUDE.md 与 README（均已核对）：

```bash
# 运行（docker）
docker run -p 6333:6333 qdrant/qdrant

# 构建（本地不建议 --release：fat LTO 极慢，用 perf profile）
cargo build --profile perf
# 格式与 lint
cargo +nightly fmt --all
cargo clippy --workspace --all-targets -- -D warnings
# ast-grep 自定义规则（CI 强制）
sg scan --config tools/ast-grep/sgconfig.yml
# Rust 测试（推荐 nextest 按 crate 跑）
cargo nextest run -p <crate>
# Python 集成测试（需 uv，多数需要运行中的 server；consensus_tests 自建集群）
uv --project tests run pytest tests/openapi
```

功能开关示例（Cargo.toml [features]）：`gpu`（GPU 建图）、`tracing`、`chaos-testing`、`data-consistency-check`、`staging`。

## 7. 测试与工程质量

- 测试规模：2118 个 .rs 文件中 571 个含 `#[cfg(test)]` 内嵌单测；tests/ 目录（Python）含 openapi 一致性测试、consensus_tests（自建多节点集群）、long-e2e-tests（.github/workflows/long-e2e-tests.yml）
- CI：17 个 GitHub Actions workflow（.github/workflows/），包括 rust.yml（单测）、rust-lint.yml、integration-tests.yml、coverage.yml、nightly-model-testing.yml（夜间模型/基准回归）、rust-gpu.yml、edge-test.yml 与 edge 发布流水线——覆盖面在同类项目中属于第一梯队
- 质量约束：clippy 全量 `-D warnings`、ast-grep 自定义规则、rustfmt nightly、rusty-hook 本地 hook（CLAUDE.md）；单元测试风格以 crate 内嵌 + fixtures/ 构造段为主（lib/segment/src/fixtures/、lib/segment/src/index/hnsw_index/tests/）

## 8. 社区与维护状态

- 活跃度：master 最近 push 2026-11-20（gh api），开源 issues 826 个——高活跃、高吞吐维护
- 发版节奏（gh api releases，近 6 个月）：v1.18.2(2026-06-04) → v1.18.3(07-17) → v1.19.0(08-05) → v1.19.1(09-04) → v1.19.2(10-05) → v1.20.0(10-22) → v1.20.1(11-13) → v1.20.2(11-16)，即每月 1-2 个版本，minor 周期稳定（~3 个月一个 minor）
- 商业化：Qdrant Cloud（README 徽章 cloud.qdrant.io，含 free tier）+ 开源 Apache-2.0 核心；Qdrant Edge 打开移动端/嵌入式增量市场（README、lib/edge/）
- 贡献者结构：主要作者 Andrey Vasnetsov 与 Qdrant Team（Cargo.toml authors）；公司主导开发（全员/多数为公司工程师）为典型模式，具体贡献者比例未逐人核实（未确认）
- 生态：官方客户端 Go/Rust/JS/Python/.NET/Java（README Clients 一节）；另提供面向 AI 编程助手的 agent skills 仓库 qdrant/skills

## 9. 亮点与风险

亮点：
1. Filterable HNSW 是过滤向量检索的标杆实现：ACORN-1 变体 + 过滤规划器（query_optimization/）+ 位图/图内双路径，工程深度明显领先同类（lib/segment/src/index/hnsw_index/）
2. 全 mmap、chunk 化的段设计带来极低内存占用与秒级启动/恢复，appendable 与 immutable 段配合 proxy 写代理实现无阻塞优化（lib/segment/src/vector_storage/dense/、lib/shard/src/proxy_segment/）
3. 纯 Rust 单二进制、无外部协调依赖的分布式方案，部署/运维成本远低于 Milvus 类微服务架构
4. 功能完整性：稠密+稀疏+BM25 混合检索、量化（PQ/二值/标量/turboquant）、GPU 建图、多向量复杂查询（Reco/Discover/Context）
5. 工程质量高：官方 CLAUDE.md code map、17 条 CI 流水线、clippy/ast-grep 强制门禁、月度发版

风险/局限：
1. 元数据单 Raft 组的横向扩展上限（无多 Raft 组分片元数据机制，src/consensus.rs + lib/storage/content_manager/consensus/），超大规模多租户场景弱于 Milvus 的多级 MQ/etcd 架构（未确认具体规模阈值）
2. release profile fat LTO 使本地构建缓慢，贡献者编译体验有成本（CLAUDE.md 明确提醒避免 --release）
3. 商业重心在 Cloud 与 Edge，开源核心的某些高级能力（如部分 RBAC/配额特性）与云版本存在边界，用户需自行确认功能边界（README 仅给 Cloud 链接，未列功能对比，未确认）
4. Edge 为新方向（v1.20 期前后引入），lib/edge 双形态架构仍在快速演进期，API 稳定性待观察（CLAUDE.md 注明公开 API 仅 lib.rs re-export）

## 10. 总结评价

Qdrant 是当前 Rust 生态中工程质量与商业化成熟度最高的向量数据库：以 filterable HNSW（ACORN-1 + 过滤规划）为技术招牌，以全 mmap 段存储为性能底座，以单二进制 Raft 集群为部署卖点，并以 Qdrant Cloud + Edge（嵌入式 Python/FFI）完成"云-端"商业化闭环。与 Milvus（Go+C++，云原生微服务、存算分离）相比，Qdrant 走"单二进制、低运维、过滤友好"路线，更适合中小规模语义检索/RAG 场景与嵌入式集成；Milvus 则适合超大规模、多组件云原生部署。适合后端工程师作为生产向量库选型，也极适合检索系统研究者精读——建议从 lib/segment/src/index/hnsw_index/graph_layers.rs（ACORN-1）与 point_scorer.rs（打分层组合）两个文件入手，这是全仓库技术密度最高的部分。
