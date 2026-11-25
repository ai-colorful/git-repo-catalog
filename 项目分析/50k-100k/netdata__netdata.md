# netdata/netdata 项目分析

> 星数 81103（快照 2026-10-24）｜主语言 Go｜档位 50k-100k｜分析日期 2026-10-31
>
> 注：清单标注主语言为 Go，但仓库实际为 C 为主体的多语言单仓：按文件数 Go 4646（.go，主要是 go.d 采集器）、C 870 + 头文件 629、Rust 794（.rs）、Python 85；核心 daemon 是 C（本仓库 `src/` 下 find 统计）。GitHub 语言条以 Go 居前源于 go.d 插件代码量。

## 1. 项目速览

一句话定位：开源实时基础设施监控平台——每秒粒度采集一切指标、内置无监督异常检测与多层存储，"X-Ray Vision for your infrastructure"（README.md）。

| 项目 | 值 |
|---|---|
| 名称/仓库 | netdata/netdata（创建于 2013-06-17，gh api） |
| 星数 | 81226（gh api 2026-10-30 实测，快照 81103） |
| 主语言 | C（daemon 核心）+ Go（采集插件）+ Rust（日志/OTel 引擎），另有 Python/Shell |
| 许可证 | GPL-3.0（LICENSE；README 声明 Agent 为 GPL v3+） |
| 最近活跃 | 最后 push 2026-10-30（gh api），CHANGELOG.md 顶部为 unreleased 且 PR 编号已达 #24132 |
| Forks / Open issues | 6704 / 455（gh api） |

## 2. 项目是什么，解决什么问题

Netdata 是一个跑在每台节点上的监控 Agent：自动发现并采集系统（CPU/内存/磁盘/网络）、容器、服务与应用的上千类指标，以每秒精度存入自带时序数据库 dbengine，在浏览器中提供无需查询语言的交互式仪表盘，并用无监督机器学习逐秒为每条指标打"异常位"（README.md "Key Features" 表、src/ml/README.md）。

它解决的核心痛点是传统监控（Prometheus 类拉取式 + 集中式存储）的三难：分辨率低（通常 15s-1m）、部署配置复杂、大规模集中存储昂贵。Netdata 的路线是"边缘自治 + 可选分层汇聚"：Agent 在本地完成采集、存储、告警评估，不需要中心服务即可工作；多节点时通过 Parent-Child 流式复制把数据汇聚到父节点（src/streaming/README.md），Netdata Cloud 只做控制面与告警去重，不收走指标数据（README.md "Netdata Ecosystem" 表）。

项目强调资源效率：README 引用阿姆斯特丹大学 ICSOC 2023 论文称其为监测 Docker 系统能耗最低的工具；存储侧 dbengine 官方数据约 0.5 字节/样本（src/database/README.md）。作者 Costa Tsaousis 在 README 中讲述了 2013 年因"监控工具指标少、分辨率低、扩展差、成本高"而自研的起源。

生态由三部分组成（README.md）：Netdata Agent（本仓库，GPL v3+）、Netdata Cloud（企业/协作层，闭源，免费社区档）、Netdata UI（仪表盘前端，NCUL1 许可，经 CDN 分发，不在本仓库构建产物中）。

## 3. 技术栈与依赖

- **C（主体）**：daemon、RRD 数据模型、dbengine 存储、流复制、健康引擎、Web API。自研基础库 src/libnetdata/（499 个文件，含环形缓冲、引用计数、-aral 分配器等）。
- **Go（采集主力）**：go.d.plugin，独立模块 src/go/go.mod，4271 个 .go 文件、142 个采集模块（src/go/plugin/go.d/collector/ 下 ls 统计），另有 cgroup-name、ebpfgo、alarm-notify 等 6 个独立 go.mod（仓库内 find）。
- **Rust（新引擎）**：src/crates/ 下 33 个 crate（Cargo.toml 工作区），含 journal-*（日志数据库）、otel-*（OpenTelemetry 摄取/日志）、sfsq/sfst（文件队列）、wal、ferryboat 等，经 corrosion 接入 CMake（根 CMakeLists.txt 注释）。
- **C++（ML）**：src/ml/ 用 dlib 实现 k-means，逐指标训练轻量模型（src/ml/README.md）。
- **Python/Shell/Bash**：python.d.plugin（85 个 .py）、charts.d.plugin、kickstart 安装器（netdata-installer.sh 53KB）。
- **构建**：CMake ≥3.16（根 CMakeLists.txt，模块化在 packaging/cmake/Modules/），支持 Linux/macOS/FreeBSD/Windows；CI 中有 windows-latest 任务（.github/workflows/build.yml）。
- 关键第三方库（根 CMakeLists.txt 注释提及）：libuv、SQLite、lz4/zstd/brotli（压缩）、libmongoc（MongoDB 导出）、dlib（ML）等。

## 4. 目录结构与代码组织

顶层是文档 + 打包 + 单一大源码目录 src/ 的"单体多语言"组织：

| 目录 | 职责 |
|---|---|
| src/daemon/ | 主进程生命周期、静态线程表、单元测试入口（135 文件；main.c） |
| src/libnetdata/ | 自研基础库（499 文件） |
| src/database/ | RRD 数据模型（rrdhost/rrdset/rrddim）+ engine/ 即 dbengine（125 文件） |
| src/collectors/ | C/Python/Shell 采集插件 38 个目录（apps、cgroups、proc、ebpf、statsd、systemd-journal、windows 等） |
| src/go/ | go.d.plugin（8589 文件，全仓最大子目录） |
| src/crates/ | Rust 工作区：日志数据库 journal-*、OTel 引擎 otel-* 等（936 文件） |
| src/health/ | 告警引擎 + 142 个内置告警配置 health.d/*.conf（821 文件） |
| src/streaming/ | Parent-Child 流式复制与副本回填（65 文件） |
| src/web/ | HTTP API v1/v2、websocket、MCP 桥（282 文件） |
| src/ml/ | C++ 异常检测（dlib k-means，29 文件） |
| src/exporting/ | 对 Graphite/OpenTSDB/Prometheus remote write/MongoDB/Kinesis/PubSub 的导出连接器（80 文件） |
| src/aclk/、src/claim/ | Netdata Cloud 链路（ACLK）与节点认领 |
| src/plugins.d/ | 外部插件协议（24 文件） |
| src/daemon/pulse/ | Agent 自身资源占用的自监控指标 |
| src/nrpc/、src/registry/ | 内部 RPC 与浏览器会话注册表（registry） |
| packaging/、integrations/、docs/、tests/ | CMake 模块与打包、自动生成的集成文档（2150 个 .md 大头）、用户文档、测试工具 |

组织评述：多语言单仓但边界清晰——每个可执行插件都有独立目录与构建脚本；integrations/ 下海量 README 由 CI 自动再生（.github/workflows/generate-integrations.yml、CHANGELOG 中反复出现 "Regenerate integrations docs"），文档即代码。仓库根有 69KB 的 AGENTS.md（CLAUDE.md/GEMINI.md 为其符号链接，从 tar 解包日志确认），是面向 AI 编码助手的工程规约，本身也是维护方式的一个信号。按文件数分布：go.d 占 8589、C 采集插件 1124、Rust 936、health 821、libnetdata 499、web 282（src/ 下 find 统计），足见"采集面"（Go 插件）与"存储/核心面"（C）并重的双引擎形态。

## 5. 核心架构与关键实现

### 5.1 采集架构：进程外插件 + 统一 plugin.d 协议

采集按"数据源类型"拆成独立进程，通过 src/plugins.d/ 的协议接入主 daemon，崩溃互不影响：C 原生插件（apps.plugin 进程树、cgroups.plugin 容器、proc.plugin /proc 解析、ebpf.plugin eBPF、statsd.plugin、systemd-journal.plugin）、Go 的 go.d.plugin（142 个网络服务/云服务模块，src/go/plugin/go.d/collector/，含 docker、k8s、redis、clickhouse、aws 等）、python.d.plugin 与 charts.d.plugin（shell）。服务发现内置（src/collectors/SERVICE-DISCOVERY.md；go.d 内有 src/go/plugin/go.d/discovery/）。Windows 由 windows.plugin/windows-events.plugin 原生覆盖（src/collectors/ 目录列表）。

daemon 主进程采用静态线程表模型：main.c 的 netdata_main() 取得 static_threads_get() 生成的线程表并逐个启动（src/daemon/main.c:344、373、1272），线程清单按平台从 static_threads_linux.c/freebsd.c/macos.c/windows.c 装配（src/daemon/ 目录文件），意味着采集、存储、健康、流收发、导出、ACLK 等子系统各占独立线程或线程组，由 libuv 事件循环与 libnetdata 提供的线程原语承载（src/libnetdata/）。

### 5.2 dbengine：多层级时序存储引擎（src/database/engine/，约 2.4 万行 C）

核心设计（src/database/engine/README.md）：数据点按"固定步长页（page）"组织——同一指标、同一 update every 的连续值构成一个纯数组页，时间戳由槽位隐含，因此元数据开销极小，可支撑每秒数百万点写入。采集中的页是内存中的 hot page，写满成为 dirty page 调度落盘，已落盘为 clean page；磁盘侧由 datafile（数据段）+ journalfile（日志段）组成。高层级（tier）不是简单均值，而是保存 sum/count/min/max/异常率，保证按 tier 查询时能精确还原 avg/min/max（engine/README.md "Data Points"）。

默认三层保留（src/database/README.md 表格）：tier0 每秒精度 14 天/1GiB、tier1 每分钟 3 个月、tier2 每小时 2 年，各层独立限空间与时间，超限按整文件删除最旧 datafile（软上限，README 明确"无硬性磁盘封顶"）。查询时跨层透明合并，长区间查询命中低层。

关键组件：mrg.c（624 行）是指标注册表（Metric Registry），以 UUID+section 索引并管理引用计数与保留时间，统计字段全部用 relaxed 原子做无锁采样（src/database/engine/mrg.h）；cache.c（3572 行）实现页缓存与压缩 extent 缓存两级；pdc.c（1595 行）为查询侧页描述符缓存；page.c（1175 行）负责页内打包/压缩/解包。三个全局实例 `multidb_ctx_storage_tier0/1/2` 体现多 tier 实例化（src/database/engine/rrdengineapi.c:8-10）。另有 dbengine-unittest.c 与 dbengine-stresstest.c 自带单元/压力测试。

### 5.3 告警体系：边缘评估 + 声明式配置（src/health/）

告警是"声明式配置 + 边缘自治评估"：142 个 stock 配置（src/health/health.d/，ls 统计）用类 ini 语法声明告警，例如 cpu.conf 的 `10min_cpu_usage` 模板——`on: system.cpu` 绑定图表、`lookup: average -10m` 声明查询、`warn/crit` 用表达式（可引用前序告警变量与状态实现迟滞）、`delay: down 15m multiplier 1.5` 做通知退避（src/health/health.d/cpu.conf）。引擎侧 health.c（218 行）驱动事件循环（health_event_loop.c 维护各状态计数），rrdcalc.c 解析/评估规则，rrdvar.c 提供 Chart/Host 变量解析，health_variable.c 支持跨图表变量。评估在边缘执行，Parent 对流上来的指标独立再评估，Cloud 层再去重合并通知（src/health/README.md 架构图）。通知分发由独立进程 alarm-notify.sh / alarm-notify Go 组件承担（src/health/notifications/，含独立 go.mod）。

### 5.4 异常检测与数据分布（src/ml/、src/streaming/）

ML 侧每条指标维度维护轻量模型：基于最近 4 小时特征向量训练 k=2 的 k-means（dlib 实现），按"多而轻"原则逐秒打分，异常标志直接存进数据点的 anomaly bit，随数据点进入存储与查询（src/ml/README.md mermaid 流程、src/database/engine/README.md 数据点定义；实现 ml_kmeans.cc、ml_dimension.h）。刻意不用深度学习以避免重依赖（ml/README.md 注）。

流式侧 stream-sender.c（935 行）/stream-receiver.c 建立 TLS 压缩流（stream-compression/），断线后由 stream-replication-sender/receiver.c 做精确副本回填（从 dbengine 重放缺口），支持多 Parent 与 green-red 集群高可用（src/streaming/PARENT-CLUSTERS.md）。运行时占用自监控：src/daemon/pulse/ 暴露 Agent 自身 CPU/内存/IO 指标。

对外数据出口分两类：一是同生态的 Parent 流复制（见上），二是第三方导出连接器 src/exporting/——Graphite、OpenTSDB、Prometheus remote write（含 filter）、JSON、MongoDB、AWS Kinesis、GCP PubSub 等，由 exporting_engine.c 统一调度（src/exporting/ 目录列表）。查询/API 侧在 src/web/：api/ 提供 v1 与 v2 REST 接口，websocket/ 支持推送，mcp/ 内置 Model Context Protocol 桥（含独立 Go 实现的 stdio bridge，src/web/mcp/bridges/stdio-golang/go.mod），前端静态资源由服务端从 CDN 获取版本（src/web/server/）。与 Cloud 的上行链路由 src/aclk/（ACLK 协议）与 src/claim/（节点认领）承担。

## 6. 构建与运行

- 安装（面向用户，README 推荐 kickstart）：
  ```bash
  wget -O /tmp/kickstart.sh https://get.netdata.app/kickstart.sh && sh /tmp/kickstart.sh
  # 或 docker run -d -p 19999:19999 --cap-add SYS_PTRACE -v netdataconfig:/etc/netdata netdata/netdata
  ```
  （docker 镜像由 .github/workflows/docker.yml 构建；顶层 Dockerfile 为指向 packaging/docker/Dockerfile 的符号链接）
- 源码构建（由根 CMakeLists.txt 与 packaging/cmake/ 模块定义；CI build.yml 流程）：
  ```bash
  cmake -S . -B build -DCMAKE_BUILD_TYPE=RelWithDebInfo
  cmake --build build -j$(nproc)
  sudo cmake --install build   # 或使用顶层 netdata-installer.sh
  ```
  Rust 依赖需工具链（corrosion 自动接入），Go 插件可单独构建：`cd src/go && go build ./...`（src/go/go.mod）。
- 自检/测试命令：`./netdata -W unittest`（src/daemon/unit_test.c 注册，含 dbengine/mrg/health-config 等单元测试，见 src/database/engine/dbengine-unittest.c、src/health/health-config-unittest.c）。
- 文档化源码构建路径：装依赖（packaging/installer/methods/manual.md:61-70 列出 zlib/libuuid/libuv/lz4/openssl/libelf/libmnl/protobuf 等）后运行 `./netdata-installer.sh`（manual.md:131，内部驱动上述 CMake 构建；支持 `--install-prefix /opt`、`--use-system-protobuf` 等选项）。

## 7. 测试与工程质量

- Go 测试规模可观：src/go 下 1647 个 *_test.go（find 统计），每个采集模块带 config_schema_test 与集成诊断（如 diagnostics_integration_test.go）。
- C/C++ 单元测试内嵌于组件：dbengine-unittest.c、mrg-unittest.c、health-config-unittest.c、ml-unittest.cc、web_client-unittest.c、daemon/unit_test.c 等约 29 个文件（find 统计），通过 `netdata -W unittest` 运行。
- 系统级测试在 tests/：k6 脚本、query-corpus（独立 go.mod 的查询语料回归）、stress.sh、spawn-server-tests 等。
- CI 共 25+ workflow（.github/workflows/）：build.yml 多平台矩阵（含 windows）、go-tests.yml、codeql.yml、coverity.yml、docker.yml、release.yml、installer-tests 等，另有 cloud_regression、dem-runtime-tests 专项。文档与告警配置也纳入门禁（check-markdown.yml、yamllint 配置在根目录）。
- 质量观察：代码注释与架构文档密度高（src/database/engine/README.md 自带设计文档与 dbengine-diagram.xml）；根 CMakeLists.txt 用长注释精确说明配置阶段顺序约束及其失败模式，体现资深 C 工程实践；.codacy.yml、.clang-format、.shellcheckrc、.yamllint 等门禁齐备。

## 8. 社区与维护状态

- 极活跃：最后 push 2026-10-30（gh api，与统计日仅隔 1 天）；unreleased 段 PR 编号至 #24132，即历史合并 PR 超 2.4 万（CHANGELOG.md）。
- 发布节奏稳定：v2.12.0（2026-09-30）、v2.11.0（2026-08-12）、v2.10.0（2026-04-09）、v2.9.0（2026-02-16）、v2.8.0（2025-11-19）（CHANGELOG.md 版本标题），约 2-3 个月一个 minor。
- 组织化维护：CHANGELOG 显示大量功能 PR 来自 go.d/otel/windows 等专项线；eBPF 内核采集器以独立仓 ebpf-co-re/kernel-collector 维护并定期同步回主仓（CHANGELOG unreleased 段 "Sync repos" PR）。近期功能面示例：go.d/redis 兼容 Valkey/Dragonfly/KeyDB/Kvrocks/Garnet（#24130）、Windows .NET Framework 指标（windows.plugin #22219）、scripts.d 原生指标快照（#24096）。贡献者结构未逐项确认，但 netdata 公司全职团队 + 社区 PR 的混合模式可从 CI 的 review/labeler/monitor-releases 工作流佐证。open_issues 455（gh api），相对仓库规模处于健康区间。

## 9. 亮点与风险

亮点：
1. 每秒粒度 + 零配置：自动发现上千种数据源，开箱即用（README Key Features；142 个 go.d 模块与 38 个 C 插件目录）。
2. dbengine 存储工程出色：固定步长页、三级缓存、多 tier 聚合保真（sum/count/min/max/异常率），约 0.5B/样本（src/database/engine/）。
3. 边缘自治架构：采集/存储/ML/告警全在节点本地完成，Cloud 只做控制面，数据主权与扩展性兼得（src/health/README.md、README Ecosystem）。
4. 逐指标轻量 ML 异常检测，异常位随数据点持久化，可与图表联动（src/ml/、src/database/engine/README.md）。
5. 工程规范严格：AGENTS.md 69KB 工程规约、自带单元+压力测试、25+ CI 工作流、多平台（含 Windows）支持。

风险/局限：
1. 多语言栈（C/Go/Rust/C++/Python）显著抬高贡献与维护门槛，Rust 引擎（crates/）尚在快速演进期，稳定性待观察。
2. GPL-3.0 对嵌入式/商业集成不友好；UI 与 Cloud 功能闭源，开源仓库不含完整前端产物（README Ecosystem 表）。
3. 存档依赖自身栈：长期冷存主要靠 tier 分层，官方文档自述无硬性磁盘封顶（src/database/README.md "Retention Size Enforcement"），大节点需谨慎配额。
4. Agent 单节点进程众多（主 daemon + 各插件进程），极小内存设备上资源占用仍需调优（文档提供 sizing 指南：docs/netdata-agent/sizing-netdata-agents/）。

## 10. 总结评价

Netdata 是监控领域少有的"把数据库、流式系统、ML 和可观测性全部塞进一个边缘 Agent"的重型工程，81k 星对应的是真实、持续的使用与维护——近 24 小时仍有 push，minor 版本保持 2-3 月一发的节奏。它以 C 写的 daemon + dbengine 为心脏，以 Go 插件生态为采集外延，再用 Rust 重构日志/OTel 等新引擎，架构上坚持"数据留在边缘"。适合运维/SRE 部署为节点级实时监控与告警，也适合系统程序员精读 dbengine（src/database/engine/）与流复制（src/streaming/）——这是开源时序存储与边缘流式架构的一手教材；对想理解"插件化采集 + 声明式告警"设计的人，src/health/ 与 src/go/plugin/go.d/ 同样值得通读。
