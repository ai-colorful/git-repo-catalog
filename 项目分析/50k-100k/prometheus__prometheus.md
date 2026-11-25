# prometheus/prometheus 项目分析

> 星数 66716（快照 2026-10-24）｜主语言 Go｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：CNCF 毕业的开源监控与告警系统，以多维数据模型 + PromQL 查询语言 + 内置时序数据库（TSDB）为核心，通过 HTTP 拉取（pull）方式采集指标。

| 项目 | 事实 |
| --- | --- |
| 名称 | prometheus/prometheus |
| 星数 | 66716（快照），2026-10-30 gh api 实测 66865 |
| 语言 | Go（go.mod 声明 go 1.26.10；CI 用 Go 1.27 构建） |
| 许可证 | Apache-2.0（LICENSE） |
| 版本 | v3.15.1（VERSION 文件；release 2026-10-21） |
| 最近活跃 | 最近 push 2026-10-30（gh api），持续高频维护 |
| 规模 | 752 个 Go 文件约 39.7 万行（不含 web/ui；其中测试文件 296 个） |

## 2. 项目是什么，解决什么问题

Prometheus 解决的是云原生环境下"服务是否健康、性能如何变化"的持续观测问题。它周期性地从配置的目标（服务、主机、中间件 exporter）抓取 `/metrics` HTTP 端点，把指标样本存入本地 TSDB，再通过 PromQL 对这些时间序列做聚合、率计算、分位数等运算，并基于规则触发告警发送到 Alertmanager（README.md"features"一节，cmd/prometheus/main.go 组装全链路）。

它与其他监控系统的本质差异在于数据模型：每条时间序列由"指标名 + 一组键值 label"唯一确定，同一指标可按 job/instance/handler 等维度裂成任意多条序列。PromQL 正是围绕这个维度模型设计的——可以按 label 选择、聚合、连接（README.md；promql/parser/ast.go 定义 Vector/Matrix 等类型）。

架构上 Prometheus 强调"单节点自治"：不依赖分布式存储，一台机器上的 TSDB 即可独立完成采集、存储、查询与告警；水平扩展靠分片（每个实例抓不同目标）或联邦（federation），长期存储与高可用则交给 Thanos/Mimir/VictoriaMetrics 等外围项目通过 Remote Write API 对接（storage/remote/，prompb/remote.proto）。这"单体自治 + 开放接口"的取舍是它成为监控事实标准的关键。

同时它也是一个完整的生态枢纽：内置 31 种服务发现提供器（discovery/ 子目录），自动感知 Kubernetes、Consul、各大云厂商的目标变化；支持 OTLP 协议直接接收 OpenTelemetry 数据（storage/remote/otlptranslator/）；web/ui 提供自带的 React 图形界面与 promtool 命令行工具（cmd/promtool/）。

## 3. 技术栈与依赖

- **主语言**：Go 1.26+（go.mod），少量前端 TypeScript/React（web/ui，pnpm workspace）。
- **关键依赖**（go.mod）：
  - `gogo/protobuf`、`golang/snappy`：Remote Write 的 protobuf 压缩传输（prompb/）。
  - `edsrzf/mmap-go`：TSDB 对 Head chunk 的内存映射（tsdb/chunks/）。
  - `oklog/run`、`fsnotify`：进程编排与配置文件热加载。
  - `alecthomas/kingpin/v2`：命令行解析（cmd/）。
  - 各云厂商 SDK（aws-sdk-go-v2、azure-sdk、godo、gce 等）：服务发现（discovery/）。
  - `KimMachineGun/automemlimit`：按容器内存限制自动设 GOMEMLIMIT（CHANGELOG 3.15.0）。
- **前端**：web/ui/mantine-ui（React + Mantine，当前主力 UI），web/ui/react-app 为遗留目录；通过 web/ui/ui.go 打包为 Go embed 嵌入二进制。
- **构建工具链**：Makefile + promu（Makefile.common）、buf 管理 protobuf（prompb/buf.yaml）、goyacc 生成 PromQL parser（promql/parser/generated_parser.y）。
- go.work 聚合 5 个模块：主仓、compliance 兼容性测试、remote_storage 示例、internal/tools、UI 侧 PromQL 工具（go.work）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| cmd/prometheus | 主程序入口，配置加载/组件编排（main.go 2394 行） |
| cmd/promtool | CLI 工具：规则单测、配置检查、TSDB 调试、数据迁移 |
| tsdb/ | 自研时序数据库：Head/WAL/块/压缩/索引/乱序（约 5.9 万行） |
| promql/ | PromQL 引擎 + parser（goyacc 生成），含 promqltest 测试框架 |
| scrape/ | 抓取管理：scrapePool/scrapeLoop/Target/重标签/ST 合成 |
| discovery/ | 31 个服务发现提供器 + 统一 Manager |
| storage/ | 存储抽象接口、fanout/merge，remote/ 远端读写与 OTLP 接入 |
| rules/ | 告警与记录规则组、调度执行 |
| notifier/ | 告警发往 Alertmanager（含 HA 集合、sendloop） |
| web/ | HTTP 服务：web.go Handler + api/v1（REST API + OpenAPI）+ ui（前端） |
| config/ | YAML 配置模型与校验 |
| model/ | 公共数据类型：labels、textparse、histogram、exemplar 等 |
| prompb/ | gogo protobuf 类型（remote.proto、types.proto） |
| docs/ | 用户文档源码（prometheus.io 的内容） |

组织方式：经典 Go 单体多包，按"数据流"分层（发现 → 抓取 → 存储 → 查询/规则 → 展示），包间通过 storage、model 中的接口解耦；前端作为独立 pnpm workspace 嵌入同一仓库，属"主仓 + 内嵌子工程"模式而非 monorepo。

## 5. 核心架构与关键实现

### 5.1 TSDB：Head + 分层块（tsdb/）

数据流为：样本先写入内存 Head（tsdb/head.go，3482 行），随 WAL 落盘保证持久性；Head 内超过 2h（chunkRange）的样本被压缩为不可变块（Block）写入磁盘，后台按指数级时间跨度层层合并（tsdb/compact.go）。查询时 DB 将 Head 与所有 Block 的 querier 合并（tsdb/db.go:310 `type DB struct` 持有 `blocks []*Block` 与 `head *Head`，tsdb/querier.go 实现合并查询）。

Head 的写入路径支持乱序（OOO）：`headAppender.Append` 在 OOO 窗口开启时不再直接拒绝旧时间戳样本，而是转入独立的 OOO Head 与 WBL（Write-Behind Log，tsdb/head.go:94 `wal, wbl *wlog.WL`，tsdb/ooo_head.go）；普通路径则快速失败返回 `storage.ErrOutOfBounds`（tsdb/head_append.go:444-450）。内存对象大量使用 zeropool 复用以压低 GC 压力（tsdb/head.go:99-108 的 refSeriesPool/floatsPool 等）。

磁盘格式有精确文档：chunk 段文件上限 512MiB，magic 0x85BD40DD，每个 chunk 为 `len(uvarint) + encoding + data + CRC-32C`，编码含 XOR/XOR2/原生直方图（tsdb/docs/format/chunks.md）；倒排索引文件 magic 0xBAAAD700，由符号表、series、label index、postings、TOC 组成（tsdb/docs/format/index.md），压缩算法源自 Gorilla 论文（tsdb/README.md）。agent/ 子目录则是裁剪版 TSDB（只有 WAL + 内存，无块落盘），供 Agent 模式使用。

### 5.2 PromQL 引擎（promql/）

engine.go（5683 行）是查询执行核心：`NewInstantQuery`/`NewRangeQuery` 解析并构建查询（promql/engine.go:788、835），`Engine.exec` 负责并发闸门（maxConcurrentQueries Gauge）、查询日志（slog 结构化输出，含 traceID）与统计（promql/engine.go:1091-1130）。求值器 `evaluator.Eval` 按表达式类型递归执行（promql/engine.go:1779），范围查询通过 `rangeEval` 对每个 step 复用 `EvalNodeHelper` 的缓存做向量化求值，二目运算的向量匹配用 join signature 序号加速（promql/engine.go:1794-1817 的 EvalSeriesHelper/EvalNodeHelper 注释）。

```go
// promql/engine.go:1779
func (ev *evaluator) Eval(ctx context.Context, expr parser.Expr) (v parser.Value, ws annotations.Annotations, err error) {
	defer ev.recover(expr, &ws, &err)
	v, ws = ev.eval(ctx, expr)
	if ev.enableDelayedNameRemoval {
		v = ev.cleanupMetricLabels(v)
	}
	return v, ws, nil
}
```

parser 由 goyacc 从 generated_parser.y 生成，支持 UTF-8 指标名（parser/unquoted_utf8_test.go）；函数库 functions.go 3260 行实现 rate/histogram_quantile 等全部内置函数。较新的实验方向包括查询成本护栏（estimate.go、matrix_budget.go、query_cost.go，配合 API 侧 `/query_cost` 端点，web/api/v1/api.go:489-492）。

### 5.3 抓取与服务发现链路（scrape/ + discovery/）

服务发现 Manager 维护 `map[poolKey]map[string]*targetgroup.Group` 并把增量更新分发给各 scrapePool（discovery/manager.go:173-186）；31 个 provider（kubernetes、consul、aws、azure 等，discovery/ 目录）都实现同一 Discoverer 接口，只吐"目标组变更"事件。

抓取侧由 manager.go（scrapePool 池化）与 scrape.go（scrapeLoop）组成：每个 target 一个 loop，按 `interval` 加种子抖动错峰（scrape/target.go:179 `offset(interval, offsetSeed)`），loop 持有 scrapeCache 做增量 label/序列映射与陈旧序列（staleness）标记，抓到的文本经 model/textparse 解析后经 Appendable 写入存储（scrape/scrape.go:929-954 `type scrapeLoop struct`）。近期增强包括 Unix Domain Socket 抓取、OM2.0 格式、zstd 压缩响应与 ST（start timestamp）合成（scrape/st_synthesis.go，CHANGELOG 3.15.0）。

### 5.4 HTTP API 与 Web（web/）

api/v1/api.go（3015 行）注册了完整的 v1 REST API：query/query_range/query_exemplars、labels/series、targets/scrape_pools、status/* 等（web/api/v1/api.go:478-525），`wrapAgent` 包装器让同一套路由在 Agent 模式下自动关闭不适用端点。v3 开始引入 OpenAPI 规范与覆盖测试（openapi*.go、openapi_coverage_test.go）和一个新搜索 API（search.go 909 行）。前端 UI 用 embed 打进二进制（web/ui/ui.go），实现"单二进制即完整监控系统"。

## 6. 构建与运行

```bash
# 构建（需先产出前端资源；Go-only 跳过 UI）
make build                # Makefile:189，含 assets assets-compress common-build
make GO_ONLY=1 common-build   # CI 内使用的纯 Go 构建（.github/workflows/ci.yml:28）

# 测试
make test                 # Makefile:177，含 UI 测试与 lint
go test ./tsdb/ -test.tsdb-isolation=false    # CI 对 TSDB 的隔离关闭跑法（ci.yml:29）

# 运行
./prometheus --config.file=prometheus.yml
docker run -p 9090:9090 prom/prometheus       # README.md

# protobuf 生成
buf generate（prompb/buf.yaml）
```

## 7. 测试与工程质量

- 测试规模：296 个 `_test.go`，与 456 个非测试文件比接近 1:1.5；TSDB 测试极重（db_append_v2_test.go 7884 行、head_test.go 6710 行、db_test.go 5523 行）。
- 自研 PromQL 表驱动测试框架 promqltest/（promql/promqltest），PromQL 规范化的测试语言，也被外部实现复用。
- CI（.github/workflows/ci.yml）分 test_go / test_go_more / test_windows / test_ui 等矩阵，另有 codeql、govulncheck、oss-fuzz 模糊测试（fuzzing.yml）、OpenSSF Scorecard、zizmor（工作流安全审计）共 16 个 workflow。
- 代码质量观察：注释密度高且带设计链接（tsdb/README.md 附系列博客）；大量基准测试（*_bench_test.go）；go.work 隔离工具依赖；配置有严格 schema 校验（config/）。

## 8. 社区与维护状态

- CNCF 毕业项目（README.md 徽章），CII Best Practices、OpenSSF Scorecard 认证齐全。
- 活跃度极高：gh api 显示最近 push 2026-10-30，open issues 约 1020，contributor 数未确认（API 分页头获取失败）。
- 发布节奏：约 6-8 周一个 minor（3.15.0 于 2026-09-24、3.15.1 于 2026-10-21，CHANGELOG.md），patch 常含安全响应（3.15.1 即为 CA 信任与 Go CVE 修复）。
- 治理文件完善：MAINTAINERS.md、RELEASE.md、CODEOWNERS、SECURITY-INSIGHTS.yml、SECURITY.md 齐备（顶层目录）。

## 9. 亮点与风险

**亮点**
1. TSDB 是少数达到工业级成熟度的自研单机时序存储：mmap Head、WAL/WBL、倒排索引、Gorilla 压缩、OOO 支持全部有文档（tsdb/docs/format/）且有重测试兜底。
2. PromQL 引擎经过十年打磨，向量化 step 求值 + signature join + 精细的内存池管理（promql/engine.go、tsdb/head.go）。
3. 生态接口开放：Remote Write/Read protobuf 协议、OTLP 接入、31 种服务发现，使其成为可替换内核的生态底座（storage/remote/、discovery/）。
4. 工程纪律突出：约 40% 文件为测试、16 条 CI 流水线含供应链安全扫描（scorecards、zizmor）。
5. 单二进制交付体验：UI embed、promtool、Agent 模式全在一个仓库内。

**风险/局限**
1. 核心复杂度陡峭：engine.go/head.go 等单文件 3000-5700 行，新人上手成本高。
2. 单机 TSDB 的横向扩展仍需外部项目（Thanos/Mimir），官方架构不含分布式存储（README.md）。
3. 大量实验特性并行推进（query_cost、ST 编码、search API），配置面与行为在版本间持续变动（CHANGELOG 频繁的 CHANGE 条目）。
4. 仓库体量大（Go+TS 前端+文档站源码），全量构建依赖 Go/Node/buf/goyacc 多工具链。

## 10. 总结评价

Prometheus 是监控领域的基础设施级开源项目：以简洁的 pull 模型、多维标签和 PromQL 建立了行业标准，又用自研 TSDB 与单二进制交付把"开箱即用"做到极致，同时通过开放协议让整个云原生监控生态（Thanos、Grafana、OTel）长在自己身上。代码库测试覆盖与文档密度都属于 Go 开源第一梯队，适合 SRE/平台工程师作为生产监控系统直接采用，也适合想深入理解时序存储（tsdb/）或查询引擎设计（promql/engine.go）的工程师精读——建议从 tsdb/README.md 与 docs/format/ 入门，再对照 head.go 与 querier.go 读实现。
