# pola-rs/polars 项目分析

> 星数 37500（快照 2026-10-24）｜主语言 Rust｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：Rust 编写的分析型查询引擎与 DataFrame 库——以 Apache Arrow 列式内存为基础，DataFrame API 与 SQL 双前端共享同一优化器和执行引擎，主打多线程向量化性能。

| 项目 | 内容 |
|---|---|
| 名称 | pola-rs/polars |
| 星数 | 37500（快照）；gh api 实查 40335（2026-11-14） |
| 主语言 | Rust（另有大规模 Python 绑定层） |
| 许可证 | MIT（`LICENSE`；`Cargo.toml` workspace.license = "MIT"） |
| 版本 | Rust workspace 0.55.1（`Cargo.toml`）；Python 包 2.1.0（`py-polars/pyproject.toml`） |
| 创建时间 | 2020-05-13（gh api） |
| 最近 push | 2026-11-13（gh api），高频活跃 |
| 最新发布 | py-2.1.0（2026-10-25，gh api releases） |
| 代码规模 | crates/ 下约 59.3 万行 Rust、2253 个 .rs 文件（find+wc 统计） |

## 2. 项目是什么，解决什么问题

Polars 是一个用 Rust 从零构建的分析查询引擎。README 明确定位为 "Extremely fast Query Engine for DataFrames and SQL"（`README.md`）：用户可以用 DataFrame 表达式 API、SQL 或两者混合来写查询，所有查询经过同一个优化器与同一个执行引擎运行。这解决了传统 DataFrame 库（如 pandas）的两个根本痛点：一是单线程/解释执行导致的性能瓶颈，二是 API 与 SQL 生态割裂导致的语义不一致。

内存模型上，Polars 采用 Apache Arrow Columnar Format 作为数据表示（README "Interoperable" 条目，`crates/polars-arrow/`），实现与 pyarrow、numpy 等的零拷贝数据共享，避免传统 DataFrame 库频繁的序列化/反序列化开销。与 pandas 的"每列一个 ndarray"不同，Polars 的列由多个 Arrow chunk 拼接（ChunkedArray），天然支持并行与增量写入。

执行模型上，Polars 是惰性求值：用户构建查询计划，优化器做谓词下推、投影裁剪、公共子表达式消除等改写，再交给执行引擎。执行引擎有两种可选：in-memory 引擎（`crates/polars-mem-engine/`）和新的流式引擎（`crates/polars-stream/`），后者以 morsel（数据分片）为单位异步流水线执行，并可通过 `crates/polars-ooc/` 的内存管理器把超内存数据 spill 到磁盘（out-of-core），因此能处理超过 RAM 的数据集，这是它区别于 pandas 的核心能力之一（README "Spill to disk" 条目）。

生态位上，Polars 与同类系统的差异：相比 DuckDB（嵌入式 OLAP SQL 数据库），Polars 首要接口是表达式 DataFrame API、SQL 为辅（`crates/polars-sql/` 基于 sqlparser 实现前端，最终编译为与 DataFrame API 相同的 IR）；相比 Spark（JVM 分布式），Polars 是单机优先、纯 Rust、无 JVM 依赖、启动零开销，分布式能力通过可选的 polars-cloud/分布式引擎暴露（Python 侧 `engine-support:: in-memory, streaming, distributed` 标注，`py-polars/src/polars/lazyframe/frame.py`）。此外支持 NVIDIA GPU 加速（`Engine::Gpu` 变体，`crates/polars-config/src/engine.rs`）。

## 3. 技术栈与依赖

- **语言**：Rust（edition 2024，`Cargo.toml`），Python 侧为纯 Python 封装层 + PyO3 编译运行时（`py-polars/pyproject.toml`：`polars` 主包用 setuptools 构建，依赖 `polars-runtime-32`；runtime 包用 maturin 构建，requires-python >= 3.11）。
- **核心自研底座**：`polars-arrow`（Arrow 列式实现的 fork/自研版）、`polars-parquet`（自研 Parquet 读写）、`polars-json`（自研 JSON）、`polars-buffer`、`polars-row`、`polars-schema`——I/O 与内存格式基本不依赖官方 arrow-rs。
- **关键第三方**：`pyo3 0.29`（Python 绑定）、`sqlparser`（SQL 前端，`crates/polars-sql/Cargo.toml` L27）、`object_store`（云存储：aws/azure/gcp/http feature）、`crossbeam`/`async-channel`/`futures`（并发与流式管线）、`chrono`/`chrono-tz`（时区）、`numpy`（可选互操作）。
- **构建与质量工具**：maturin（Python 轮子）、dprint + rustfmt + clippy（`dprint.json`、`rustfmt.toml`、`crates/clippy.toml`）、cargo-deny（`deny.toml`）、Nix flake（`flake.nix`）。

## 4. 目录结构与代码组织

顶层目录（均为仓库根实测）：

| 目录 | 职责 |
|---|---|
| `crates/` | 35 个 Rust crate 组成的 workspace 主体 |
| `py-polars/` | Python 包：纯 Python 层 + 3 个编译 runtime wheel + 测试与文档 |
| `pyo3-polars/` | 官方插件框架（用户用 Rust 写表达式/IO 插件给 Python 用） |
| `docs/` | 用户指南与 API 文档源码（mkdocs 系） |
| `examples/` | 示例数据集 |
| `tools/` | 维护脚本 |
| 根文件 | `Cargo.toml`（workspace）、`Makefile`、`deny.toml`、`dprint.json` |

`crates/` 关键成员（`ls crates/` 实测，35 个）：

| crate | 职责 |
|---|---|
| `polars-core` | ChunkedArray/Series/DataFrame 数据模型（3.2MB） |
| `polars-arrow` | Arrow 列式内存与计算原语（2.1MB） |
| `polars-plan` | DSL→IR 转换、逻辑计划、优化器（3.9MB，最大 crate） |
| `polars-expr` | 表达式求值运行时（1.4MB） |
| `polars-stream` | 新流式引擎（2.8MB） |
| `polars-mem-engine` | 旧 in-memory 引擎（239KB） |
| `polars-io` | CSV/IPC/JSON 等扫描与写出（1.3MB） |
| `polars-ops` | 列级算子实现库（1.3MB） |
| `polars-sql` | SQL 前端（1.4MB） |
| `polars-python` | PyO3 绑定层（1.2MB） |
| `polars` | 聚合门面 crate，靠 Cargo feature 组合开关 |
| `polars-compute`/`polars-parquet`/`polars-time`/`polars-ooc`/`polars-utils` 等 | 计算内核/Parquet/时间/超内存/工具 |

组织方式评述：典型的"多 crate workspace + 薄门面"monorepo。模块间依赖严格分层（utils→arrow→core→ops→expr/plan→engine→io→python），顶层 `polars` crate 几乎不含实现，只通过约 60 个 Cargo feature 把 30 多个 crate 的能力按需组装（`crates/polars/Cargo.toml` `[features]`），用户可以裁剪出只含核心/CSV/Parquet 的最小构建。语言绑定（Python）放在 `polars-python` + `py-polars`，Node.js 与 R 绑定已迁出到独立仓库（README 徽章指向 nodejs-polars/r-polars，本仓库内无对应目录，未在本仓库内确认源码）。

## 5. 核心架构与关键实现

### 5.1 数据模型：ChunkedArray + 类型擦除的 Series（polars-core）

`crates/polars-core/src/chunked_array/mod.rs` L140：

```rust
pub struct ChunkedArray<T: PolarsDataType> {
    pub(crate) field: Arc<Field>,
    pub(crate) chunks: Vec<ArrayRef>,
    pub(crate) flags: StatisticsFlagsIM,
    length: usize,
    null_count: usize,
    _pd: std::marker::PhantomData<T>,
}
```

设计要点：`ChunkedArray<T>` 是"类型化的心"（该文件模块注释原话），泛型 `T` 提供编译期类型分派；底层数据是多个 Arrow `ArrayRef` chunk 的向量，追加/过滤只产生新 chunk，避免整列拷贝，`should_rechunk` 按需合并。其上是两层类型擦除：`Series` 是 `Arc<dyn SeriesTrait>`（`crates/polars-core/src/series/mod.rs` L151），而 `DataFrame` 只持有 `Vec<Column>` 和缓存 schema（`crates/polars-core/src/frame/dataframe.rs` L85-91）。`Column` 是个巧妙的枚举（`crates/polars-core/src/frame/column/mod.rs` L45-48）：要么是普通 `SeriesColumn`，要么是 `ScalarColumn`（单标量重复），让 `pl.lit(1)` 广播类操作零拷贝。

### 5.2 双前端同一 IR：Expr DSL → AExpr arena → 优化器（polars-plan）

用户侧表达式是 `Expr` 枚举树（`crates/polars-plan/src/dsl/expr/mod.rs` L67 起）：`Column/Literal/BinaryExpr/Sort/Gather/Agg/Function/Over(窗口)/Ternary` 等 30 余个变体，构成可组合的表达式 DSL。collect 时 DSL 被转换为 arena 化的 IR：表达式变成 `AExpr`（`crates/polars-plan/src/plans/aexpr/mod.rs` L175），子表达式以 `Node`（arena 索引）引用；逻辑计划是 `IR` 枚举（`crates/polars-plan/src/plans/ir/mod.rs` L56 起）：`Scan/DataFrameScan/Filter/Select/Join/GroupBy/Slice/Union...`。SQL 前端（polars-sql）和 DataFrame API 都降落到同一套 IR，因此优化一次写好、两端受益。

优化器位于 `crates/polars-plan/src/plans/optimizer/`，约 40 个独立 pass：`predicate_pushdown/`、`projection_pushdown/`、`collapse_and_project.rs`、`cse/`（公共子表达式消除）、`join_pushthrough.rs`、`join_order/`、`simplify_expr/`、`slice_pushdown_lp.rs`、`parquet_metadata_prune.rs`、`aggregate_from_statistics.rs` 等，按序组合应用——这是 Polars "lazy 查询被开箱优化"（README）的实体。

### 5.3 执行引擎：morsel 驱动的流式管线 + 超内存 spill（polars-stream / polars-ooc）

引擎选择是显式枚举（`crates/polars-config/src/engine.rs`）：

```rust
pub enum Engine {
    Auto = 0,
    Streaming = 1,
    InMemory = 2,
    Gpu = 3,
}
```

新流式引擎 `crates/polars-stream/` 是异步 morsel 流水线：`src/morsel.rs` 定义带全序 `MorselSeq` 的数据分片（保证顺序语义下并行重排），`src/nodes/` 下有 30+ 物理算子节点（joins/group_by/io_sources/io_sinks 等），物理计划由 `physical_plan/` 从 IR 编译，入口 `build_streaming_query_executor`（`src/dispatch.rs`）。内存压力交给 `crates/polars-ooc/`：自定义全局分配器（`global_alloc.rs`）估算用量、`memory_grant.rs`/`memory_manager.rs` 管理内存预算，`spill_frame.rs`/`spill_file.rs` 把 DataFrame 序列化落盘再换入。旧 in-memory 引擎保留在 `crates/polars-mem-engine/` 作为回退，用户侧 `LazyFrame.collect(engine=...)` 可显式选择，`Engine::from_str` 明确报错 "the 'old-streaming' engine has been removed"（`crates/polars-config/src/engine.rs`）——显示了引擎代际迁移已完成。

### 5.4 多语言绑定：runtime wheel 分离 + 句柄直通（py-polars / polars-python）

Python 绑定采用"纯 Python 包 + 可切换编译运行时"的独特结构：`polars` 主包是纯 Python（setuptools，`py-polars/pyproject.toml`），实际二进制在 `polars-runtime-32/64/compat` 三个 maturin wheel 里（`py-polars/runtime/`）；`py-polars/src/polars/_plr.py` 在 import 时按环境变量/可用性把 `sys.modules[__name__]` 替换为对应 runtime 的 Rust 模块，从而支持 32 位特性差异与兼容模式，且允许运行时升级二进制不动 Python 层。表达式桥接是"句柄直通"式：Python `Expr` 对象只持有一个 `expr._pyexpr` Rust 句柄（`py-polars/src/polars/expr/expr.py` L333），每个方法调用转成对 PyO3 `PyExpr` 的调用（Rust 侧在 `crates/polars-python/src/expr/` 按域拆分为 18 个模块）；执行时 `PyLazyFrame::collect` 直接把 DSL 计划递给 Rust：`ldf.collect_with_engine(engine.0)`（`crates/polars-python/src/lazyframe/general.rs` L708-730），Python 侧不重写任何执行逻辑，因此 Python/Rust API 天然同构。另有 `pyo3-polars/` 让第三方用 Rust 写自定义表达式/IO 插件编译成 wheel 注入 Python Polars（README "Extensible"）。

## 6. 构建与运行

来自 `Makefile` 与 `py-polars/Makefile`（实测）：

```bash
# Rust 侧
cargo build --locked -p polars                 # 构建 Rust 门面 crate
cargo test --locked --features=...             # CI 同款（.github/workflows/test-rust.yml）

# Python 侧开发安装（maturin 编译 runtime 包）
make build            # = build-mindebug
make build-release    # 优化构建
pytest py-polars      # 运行 Python 测试（Makefile test 目标）
```

CLI：`crates/polars-cli/` 提供 `polars` 命令行查询工具（`polars --help` 可执行 SQL 查询 parquet/csv，`crates/polars-cli/src/main.rs`，未运行验证仅静态确认存在）。

## 7. 测试与工程质量

- **测试规模**：Python 侧 497 个测试文件（`py-polars/tests/`，按 `constructors/expr/io/streaming/ooc/sql/interop` 等目录分域，另有 `benchmark/`）；Rust 侧 181 个测试文件（crates 内 `tests/` 或模块内 `#[cfg(test)]`）。流式引擎与 OOC 各有专测目录（`py-polars/tests/unit/streaming/`、`py-polars/tests/unit/ooc/`）。
- **CI**（`.github/workflows/`，22 个 workflow）：`test-rust.yml`（ubuntu+windows 矩阵、多 feature 组合）、`test-python.yml`（多 OS/多 Python 版本）、`test-coverage.yml`、`test-bytecode-parser.yml`、`benchmark.yml`（性能基准）、`lint-rust.yml`/`lint-python.yml`/`lint-global.yml`、`release-python.yml`/`release-rust.yml`、`release-drafter.yml`。
- **质量观察**：强约束工具链——rustfmt + clippy 自定义 lint 配置（`crates/clippy.toml`）、cargo-deny 供应链审计（`deny.toml`）、dprint 统一非 Rust 文件格式。Rust 侧大量使用 feature 门控保证"裁剪可用"，且 `#![deny(missing_docs)]` 出现在 polars-sql（`crates/polars-sql/src/lib.rs`）。unsafe 集中在 core/arrow/compute 并有注释说明（如 `chunked_array/mod.rs` 顶部 `allow(unsafe_op_in_unsafe_fn)`）。

## 8. 社区与维护状态

- **活跃度**：gh api 实查 pushed_at 2026-11-13（次日仍在 push），open issues+PR 2945，是持续高吞吐维护的仓库；创建于 2020-05-13，5 年半从 0 到 40k+ stars。
- **发布节奏**：gh api releases 显示 Python 包约每 1-3 周一版（py-1.43.0 2026-07-21 → py-1.44.x 2026-08-24 → py-2.0.0 2026-10-06 → py-2.1.0 2026-10-25，12 个 release）；Rust crate 独立发版（rs-0.55.x 2026-08）。Python 已进入 2.x 大版本，Rust 仍为 0.55.x，双版本线并行（Python 版号 ≠ Rust 版号，未在本仓库内找到映射说明，标注"未确认"细节动机）。
- **贡献者结构**：contributors API 单页返回 100（per_page 上限），即至少 100 人；作者 Ritchie Vink 仍是 Cargo.toml 里的唯一 workspace author，由 Polars Inc.（polars-cloud 等商业组件可见于依赖 `polars_cloud >= 0.13.0`，`py-polars/pyproject.toml`）主导，社区 PR 流量高（CI 中有 pr-labeler/issue-labeler 自动分流）。

## 9. 亮点与风险

**亮点**

1. **单优化器双前端**：DataFrame API 与 SQL 编译到同一 IR（`crates/polars-plan/src/plans/`），表达式树 DSL 设计干净，是学习查询引擎优化器组织方式的优质范本。
2. **流式引擎 + 超内存能力**：morsel 异步流水线（`crates/polars-stream/src/morsel.rs`）配合全局分配器与 spill（`crates/polars-ooc/src/`），单机即可处理超出内存的数据，且新引擎代码结构清晰可读。
3. **极致模块化**：35 crate + 约 60 个 feature 门控（`crates/polars/Cargo.toml`），可裁剪构建，依赖分层严格。
4. **绑定工程先进**：纯 Python 包 + 可切换 runtime wheel 的分层（`py-polars/runtime/`）解决了二进制/纯 Python 解耦、CPU 特性检测、兼容模式等实际问题。
5. **自研底层栈**：polars-arrow/polars-parquet/polars-json 自研，摆脱官方 arrow-rs 的 API 约束，为性能优化留足空间。

**风险/局限**

1. **编译成本高**：59 万行 Rust + maturin + LTO，从源码构建 Python 包非常耗时（Makefile 提供了 fast-release/no-debug 等多档 profile 来缓解）。
2. **双版本线心智负担**：Rust 0.x 与 Python 2.x 版本不对应、旧 streaming 引擎已移除，跨语言升级需关注 breaking 变化（`Engine::from_str` 对 old-streaming 直接报错）。
3. **API 语义与 pandas 不同**：表达式风格、null 处理、索引缺失对 pandas 用户有迁移成本（README 定位即非 pandas 克隆）。
4. **GPU/分布式依赖外部组件**：`Engine::Gpu` 与分布式引擎需要闭源/独立组件（README、`py-polars/pyproject.toml` 中 polars-cloud 依赖），开源仓库内不可见实现。

## 10. 总结评价

Polars 是过去五年里 DataFrame/查询引擎领域最成功的 Rust 项目：以 Arrow 列式内存为地基，用一套 IR 统一 DataFrame API 与 SQL 两个前端，靠 morsel 流水线 + 超内存 spill 把单机分析性能与数据容量同时拉满，再用高度模块化的 workspace 和先进的绑定工程把这套引擎送到 Python/Rust/Node/R 多语言用户手中。它不是"更快的 pandas"，而是一个接口更现代的单机分析查询引擎。适合数据规模在单机量级、追求性能且愿意接受新 API 的数据工程师/科学家作为 pandas 替代；对查询引擎内部实现、Rust 大型 workspace 组织、PyO3 绑定工程感兴趣的开发者，`polars-plan` 的优化器与 `polars-stream` 的流式管线是非常值得精读的源码。
