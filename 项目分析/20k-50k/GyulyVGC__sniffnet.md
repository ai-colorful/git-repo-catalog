# GyulyVGC/sniffnet 项目分析

> 星数 32500（快照 2026-10-24）｜主语言 Rust｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

**一句话定位**：Sniffnet 是一个用纯 Rust 编写的图形化网络流量监控工具——选一块网卡就能实时看到"谁在和你的电脑通信、 exchanging 什么协议/服务/程序、花了多少流量"，跨 Windows/macOS/Linux 三平台开箱即用。

| 项目 | 事实 |
|---|---|
| 名称 | sniffnet（仓库 GyulyVGC/sniffnet） |
| 星数 | 42,695（gh api watchers，2026-11-14 核实；任务快照 32,500） |
| 语言 | Rust（Rust 2024 edition，`Cargo.toml` workspace.package） |
| 版本 | v1.5.1（2026-07-22 发布，`CHANGELOG.md`；main 分支 Cargo.toml 同为 1.5.1，含未发布的 IPFIX 收集器功能） |
| 许可证 | MIT OR Apache-2.0 双许可（`Cargo.toml` workspace.license、仓库含 LICENSE-MIT/LICENSE-APACHE；GitHub 元数据识别为 Apache-2.0） |
| 创建时间 | 2022-07-31（gh api created_at） |
| 最近 push | 2026-11-13（gh api pushed_at，分析前一天仍在提交，高度活跃） |
| 贡献者 | 88 人（gh api contributors），实际核心维护者 1 人（详见第 8 章） |
| Forks / Open issues | 2,010 / 57（gh api） |

## 2. 项目是什么，解决什么问题

传统抓包工具 Wireshark 功能强大但对普通用户门槛高，而 `tcpdump`/`nethogs` 等命令行工具又缺乏可视化。Sniffnet 的切入点是"中间地带"：把 libpcap 抓包能力包装成一个漂亮、易用、纯 Rust 实现的桌面应用，用户点几下鼠标就能回答日常问题——"哪个程序在偷跑流量""我正在和哪些国家/公司的服务器通信""有没有连到可疑 IP"。功能清单见 `README.md` "Features" 一节：网卡选择与 BPF 过滤、实时图表、地理定位、rDNS/ASN 解析、6000+ 上层服务识别、程序级流量归属、PCAP 导入导出、自定义 IP 黑名单、事件通知、多主题、50+ 界面语言。

它最重要的差异化是**技术栈纯度**：GUI 用纯 Rust 的 Iced 框架而非 Electron/Tauri（无浏览器内核），抓包走 libpcap（pcap crate），整个 workspace `unsafe_code = "forbid"`（`Cargo.toml` workspace.lints.rust），单二进制分发。README 还提供 Windows 免费代码签名（SignPath.io），这些工程细节显著降低了"下载一个流量嗅探器"的心理门槛。

main 分支正在开发的下一版（`CHANGELOG.md` [UNRELEASED]）引入了 IPFIX 收集器能力（#1270）：不再是只能看本机网卡，还能监听 UDP socket 接收远端路由器/交换机导出的 NetFlow/IPFIX 流量记录，这把应用从"本机嗅探器"扩展为"小型流量采集分析站"，也是本次分析快照里最大的一块新代码（`src/networking/ipfix/` 9 个文件约 2,000 行）。

## 3. 技术栈与依赖

全部出自 `Cargo.toml`（workspace + root crate）：

- **GUI**：`iced 0.14.0`（features: tokio/svg/advanced/lazy/image）——纯 Rust Elm 架构 GUI 框架，2026-04 的 v1.5.0 刚完成向 Iced 0.14 的大迁移（#1032）
- **抓包**：`pcap 2.5.0`（libpcap 绑定）；`etherparse 0.21.0`（协议头解析，用于独立子 crate）
- **异步运行时**：`tokio`（macros/fs）+ `async-channel`（后端→GUI 消息通道）+ `tokio::sync::broadcast`（冻结/恢复抓包的广播）
- **DNS/网络**：`dns-lookup`（rDNS）、`reqwest`（更新检查、远程通知）、`surge-ping`（连接延迟测量，v1.5.1 新增）、`listeners`（端口→进程归属）
- **地理数据**：`maxminddb`，内嵌 GeoLite2-Country.mmdb 与 GeoLite2-ASN.mmdb（`resources/DB/`），允许用户在设置里替换自定义 MMDB（`src/gui/sniffer.rs` MmdbReaders）
- **图表**：`plotters` + `plotters-iced2`（在 Iced canvas 里画实时流量图）
- **其他**：`clap`（CLI 参数）、`confy`（TOML 配置持久化）、`nom 8`（IPFIX wire 解码）、`rodio`（通知音效 mp3）、`rfd`（文件对话框）、`phf`（编译期服务端口哈希表）、`rustrict`（构建期域名审查，见 build.rs）、`picon`（进程图标）
- **平台特定**：`gag`（Windows release 下重定向 stdout/stderr，`src/main.rs`）；打包配置覆盖 deb（cargo-deb metadata）、rpm（generate-rpm + setcap）、wix（Windows MSI）、Dockerfile、Cross.toml

构建质量约束（`Cargo.toml` `[workspace.lints]`）：`unsafe_code = "forbid"`，clippy `pedantic` 全开，`unwrap_used`/`expect_used`/`panic`/`print_stdout` 等均 warn。

## 4. 目录结构与代码组织

Cargo workspace 双成员：根 crate `sniffnet` + `lib/sniffnet-packet-parser`（版本 0.2.1，独立发布，可按 feature 裁剪）。全仓 167 个 .rs 文件、约 35,500 行。

| 目录 | 职责（证据：目录内模块名） |
|---|---|
| `src/networking/` | 核心：抓包与统计后端。`capture.rs`（后端线程入口）、`parse_packets.rs`（pcap 管线）、`manage_packets.rs`（1856 行，流量聚合）、`ipfix/`（9 文件，IPFIX 收集器）、`types/`（27 个类型文件：InfoTraffic、AddressPortPair、Host 等） |
| `src/gui/` | Iced 前端：`sniffer.rs`（2616 行，应用状态与 update/subscription）、`pages/`（welcome/initial/overview/inspect/connection_details/thumbnail/settings 各页）、`components/`、`styles/`（7 套自定义主题）、`types/`（conf/filters/message 等） |
| `src/chart/` | 实时流量图与环形图（plotters 封装） |
| `src/mmdb/` | GeoLite2 国家/ASN 库读取封装 |
| `src/report/` | Inspect 页的连接报表搜索/排序（内存数据） |
| `src/notifications/` | 事件通知（声音/日志/远程 webhook） |
| `src/translations/` | 50+ 界面语言的翻译表（enum Language 约 56 变体，`translations/types/language.rs`） |
| `src/cli/`、`src/utils/`、`src/countries/` | 命令行、工具函数、国旗 SVG 渲染 |
| `lib/sniffnet-packet-parser/` | 独立发布的报文解析 crate：etherparse 之上产出 `ParsedPacket{link_info, net_info, transport_info}`（`packet.rs`） |
| `resources/` | 内嵌资源：GeoLite2 mmdb、字体（Sarasa Mono SC 子集）、图标、音效、服务/端口表（services.txt）、打包素材、themes、`thesis.pdf`（3.9MB，作者学位论文） |
| `build.rs` | 编译期生成服务端口 phf map（services.txt → `SERVICES`）、Windows 图标资源 |
| 顶层文档 | README、ROADMAP、SECURITY、THREAT_MODEL、INCIDENT_RESPONSE、CONTRIBUTING、CONTRIBUTORS（tarball 因 .gitattributes export-ignore 缺失的 CHANGELOG/CONTRIBUTING/CONTRIBUTORS 已用 gh api 补齐核对） |

组织方式评述：单体应用 + 一个抽出的解析子 crate（为了 crates.io 独立发布和 feature 裁剪），分层清晰——`networking`（后端线程）经 `async_channel` 向 `gui`（Iced 主循环）单向推送消息，两世界只在消息类型上耦合，是 Elm 架构做实时数据应用的教科书式布局。

## 5. 核心架构与关键实现

### 5.1 抓包管线：双线程 + 有界队列 + 1 秒批量上报（`src/networking/parse_packets.rs`、`src/networking/capture.rs`）

后端由专用 OS 线程驱动而非 async：`packet_stream` 线程阻塞式调用 `cap.next_packet()`，把报文塞进容量 10,000 的 `std::sync::mpsc::sync_channel`；主解析线程以 150ms 超时消费（`parse_packets.rs:57-74`）。这是关键设计——pcap 阻塞读与统计计算解耦，突发流量先缓冲，统计线程每个 interval 批量聚合，GUI 每秒收到一次 `TickRun` 消息而不是每个包刷新一次：

```rust
// src/networking/parse_packets.rs
let (pcap_tx, pcap_rx) = std::sync::mpsc::sync_channel(10_000);
let _ = thread::Builder::new()
    .name("thread_packet_stream".to_string())
    .spawn(move || packet_stream(cap, &pcap_tx, &mut freeze_rx_2, &filters))?;

loop {
    ...
    let (packet_res, cap_stats) = pcap_rx
        .recv_timeout(Duration::from_millis(150))
        .unwrap_or((Err(pcap::Error::TimeoutExpired), None));
    ...
    if let Some(parsed) = ParsedPacket::from_bytes(&packet.data, link_type) {
        ...
        let (traffic_direction, service) = modify_or_insert_in_map(
            &mut info_traffic_msg, &key, ...);
```

暂停/恢复用 `tokio::broadcast` 双通道（freeze_rx），两个线程都监听，恢复时抓包线程先 `pause()`/`resume()` 设备（`parse_packets.rs:219-245`）。丢包数从 pcap `stats()` 读取透传给 UI（`dropped_packets`）。PCAP 导入（离线文件）按报文时间戳推进 tick，遇到时间空洞发 `OfflineGap` 让图表跳变而不是卡死（`parse_packets.rs:190-217`，CHANGELOG 显示专门修过大时间差导致内存耗尽的 bug）。

rDNS 解析单独做一个 **5 线程池**（`capture.rs:35, 87-103`），注释明说"一个慢的阻塞查询不能拖累其他解析"——把 DNS 慢查询从统计路径完全剥离；未解析的地址先记在 `addresses_waiting_resolution`，解析完成后在下一次 tick 里把累计流量合并进 `HostMessage` 补发（`capture.rs:213-231`）。

### 5.2 数据存储：无 SQLite，纯内存 HashMap + PCAP 落盘（`src/networking/types/info_traffic.rs`）

需要澄清一个常见误解：**Sniffnet 不使用 SQLite**（全仓 grep "sqlite" 零命中，依赖表也无相关 crate）。所有统计都在内存里，每秒批量更新：

```rust
// src/networking/types/info_traffic.rs
pub struct InfoTraffic {
    pub tot_data_info: DataInfo,
    pub dropped_packets: Option<DroppedPackets>,
    pub map: HashMap<AddressPortPair, InfoAddressPortPair>,   // 每条连接
    pub services: HashMap<Service, DataInfo>,                  // 每种服务聚合
    pub hosts: HashMap<Host, DataInfoHost>,                    // 每个主机聚合
    ...
}
```

持久化只有两类：完整报文可导出为 PCAP 文件（`capture_context.rs` 的 `savefile()`，边抓边写）；用户设置/收藏持久化为 TOML，走 `confy`，且有一套防御性反序列化（每个字段 `deserialize_or_default`，加载后 `sanitize()` 校验缩放/音量等取值范围，`src/gui/types/conf.rs:28-162`）——这是配置文件向后兼容的稳妥写法。

### 5.3 服务识别：编译期 phf 表 + 启发式打分（`build.rs`、`src/networking/manage_packets.rs`）

6000+ 端口/服务映射（含木马、蠕虫标注）在 `resources/services/services.txt`，由 `build.rs::build_services_phf` 在编译期生成 phf 零散射哈希表，运行时查表零成本。两端口的候选服务用打分函数抉择：周知端口（<1024）加分、远端端口方向加分（`manage_packets.rs:34-77`）。build.rs 还用 `rustrict` 对服务名做审查过滤，避免脏词进入 UI。程序归属（哪个进程产生流量）走 `listeners` crate 查 socket 表 + `picon` 拉进程图标，独立线程 `thread_lookup_program`/`thread_get_picon` 执行（`src/gui/sniffer.rs:1156-1170`）。

### 5.4 IPFIX 收集器：第二抓包后端（`src/networking/ipfix/collect.rs`，未发布功能）

新后端把 UDP socket 当作"网卡"的等价物：`collect_ipfix` 接收 65535 字节 datagram，`nom` 写的 `wire.rs` 解析消息头/模板集/数据集，`TemplateCache`/`BaselineCache` 跨 datagram 维护模板与计数基线，产出的 `FlowRecord` 复用与 pcap 后端完全相同的 `modify_or_insert_in_map`/`update_connection_stats` 聚合路径（`collect.rs:1-45`）。架构上这是干净的策略模式：`CaptureContext::{Live, Offline, Ipfix}` 三种来源，一种聚合出口。

### 5.5 GUI 层：Iced 0.14 的标准用法（`src/gui/sniffer.rs`、`src/main.rs`）

`Sniffer` 结构体承载全部状态（2616 行），Elm 三件套 `update/view/subscription` 齐备；后端消息经 `async_channel` 订阅进 GUI 主循环。值得借鉴的细节：用 capture id 计数丢弃上一轮抓包残留消息（`current_capture_rx: (usize, Option<Receiver<_>>)`，`sniffer.rs:98-101`）；Wayland 下必须设置 application_id（`main.rs:83-84` 注释引用 issue #292）；Windows release 构建用 `gag` 重定向 stdout/stderr 防止控制台弹出（`main.rs:47-55`）。图表用 `plotters-iced2` 把 plotters 绘制结果画进 Iced 元素（`src/chart/types/traffic_chart.rs`）。

### 5.6 权限与跨平台处理

- Linux：RPM 安装时 `setcap cap_net_raw,cap_net_admin=eip` 赋予抓包能力（`Cargo.toml` `[package.metadata.generate-rpm]`）；deb 依赖 libpcap0.8；v1.6 开发中专门排除了 Linux D-Bus 虚拟设备防止系统操作挂起（CHANGELOG #1320）
- 抓包设备打开失败统一收敛为 `CaptureContext::Error(String)` → GUI 弹 `CaptureError::Fatal` 错误页（`src/networking/types/capture_context.rs:27-37`），把"没权限/缺 libpcap"以人话呈现，README 另有 Troubleshooting 章节
- 分发矩阵极宽：MSI x64/arm64/x86、DMG Intel/Apple Silicon、DEB/RPM/AppImage 多架构（README Download 表 + release 15 个资产），Docker 镜像同时支持 X11 与 Wayland（`Dockerfile`）

## 6. 构建与运行

来自 `Cargo.toml`、`.github/workflows/rust.yml`、`Dockerfile`、README：

```bash
# 依赖：libpcap（Unix）或 Npcap（Windows）；Linux 另需 libasound2、libfontconfig1
cargo install sniffnet            # crates.io 安装（crates.yml workflow 手动触发 cargo publish --workspace）
cargo run                         # 源码运行（需 libpcap-dev 等开发头文件）
cargo build --release             # release: lto + strip + codegen-units=1
cargo fmt --all -- --check        # CI 第一步（rust.yml）
cargo clippy -- -D warnings       # CI 第二步
cargo test --verbose -- --nocapture  # CI 第三步
cargo deb / cargo generate-rpm    # 本地打包（Cargo.toml 内置 metadata）
```

无独立运行参数时的 CLI：`clap` derive，支持 `--config_path` 等（`src/cli/`）。

## 7. 测试与工程质量

- **测试规模**：243 个 `#[test]`，分布在 42 个文件（多为模块内 `mod tests`，45 处）；dev-deps 用 `rstest`（参数化）、`serial_test`（共享状态测试串行化）、`serde_test`。覆盖偏单元与模块级：`conf.rs`、`manage_packets.rs`、`ipfix/*`、`filters.rs` 等核心逻辑文件均带测试；GUI 层测试较少（符合 Iced 应用常态）。未发现独立 e2e/UI 自动化目录（未确认）
- **CI**：6 个 workflow（`.github/workflows/`，gh api 核实）——`rust.yml`（fmt+clippy+test，push/PR）、`codeql.yml`（安全扫描）、`package.yml`（三平台产物构建与签名）、`docker.yml`、`ppa.yml`（Ubuntu PPA）、`crates.yml`（手动发布 crates.io）
- **代码质量观察**：`unsafe_code = "forbid"` 全仓禁用 unsafe；clippy pedantic + 禁 unwrap/expect/panic/stdout 打印，错误统一走 `ErrorLogger` + `location!()` 宏（`src/utils/error_logger.rs`）；线程均用 `thread::Builder` 命名，便于排查；配置反序列化全字段防御。安全文档（SECURITY/THREAT_MODEL/INCIDENT_RESPONSE）齐全到一个独立维护者项目少见的程度，Windows 安装器有 SignPath 免费签名（README）

## 8. 社区与维护状态

- **活跃度**：2026-11-13 仍有 push（gh api）；v1.5.1 发布于 2026-07-22。近三年 release 节奏稳定：1.3.0（2024-04）→ 1.3.1（2024-07）→ 1.3.2（2025-01）→ 1.4.0（2025-06）→ 1.4.1（2025-09）→ 1.4.2（2025-11）→ 1.5.0（2026-04）→ 1.5.1（2026-07），约每季度一次小版本，每次 10-15 个安装包资产（gh api releases）
- **贡献者结构**：88 名贡献者中，GyulyVGC（Giuliano Bellini，Cargo.toml 唯一作者）2565 次提交，之后是机器人（dependabot 201、allcontributors 146、ImgBotApp 25），人类第二名 joshuamegnauth54 仅 45 次。**典型的单维护者项目**：代码架构、发布、路线图均一人主导，社区贡献集中在翻译（50+ 语言靠 all-contributors 管理，CONTRIBUTORS.md）、issue 反馈和小修补
- **风险提示**：bus factor = 1。但 ROADMAP.md、THREAT_MODEL.md 等文档质量高，CHANGELOG 每条改动都关联 PR/issue 编号，工程延续性风险被文档部分对冲
- 赞助渠道存在（`.github/FUNDING.yml`，README "Support Sniffnet's development" 章节）

## 9. 亮点与风险

**亮点**

1. **"阻塞抓包线程 + 有界队列 + 批量 tick 上报"的实时管线**（`parse_packets.rs`）：把每秒数万包的压测压力隔离在后端，GUI 只收每秒一条消息，Iced 单线程渲染不被拖垮——Rust 桌面实时应用的优秀参考实现
2. **彻底的类型驱动**：27 个 networking 类型文件、无 sqlite/无全局可变状态，统计全靠 HashMap 结构体 + 消息传递，`unsafe` 全禁，读起来像教程
3. **零成本服务识别**：build.rs 编译期生成 phf 表 + 打分启发式（`manage_packets.rs`），6000+ 服务查询 O(1)
4. **分发工程一流**：三平台 15 种安装包、RPM 自动 setcap、Windows 免费代码签名、crates.io + GitHub Releases 双渠道、Docker/PPA 兜底
5. **双许可 + 嵌入式 GeoLite2**：开箱即用无需用户自行申请 MaxMind 下载

**风险/局限**

1. **单维护者**：核心架构与发布全压在一人身上（第 8 章），突发停更无兜底
2. **纯内存统计**：无历史持久化（无 SQLite），重启丢数据；超长时间抓包时 HashMap 随连接数增长，内存上限未设防（仅 PCAP 导出是逃生通道）
3. **每包全量解析的性能天花板**：2.5Gb/s+ 场景下 etherparse 级别的逐头解析与 10k 队列可能跟不上，项目定位个人主机监控而非链路级采集（IPFIX 后端正是对此的架构回应，尚未发布）
4. **依赖面较宽**（Cargo.lock 701 个包），GUI 应用编译时间长、release 二进制对库版本敏感（CHANGELOG 记录过 glibc 2.39 兼容性 bug）

## 10. 总结评价

Sniffnet 证明了 Rust + Iced 可以做出不输 Electron 产品的桌面工具：功能聚焦（看流量、认主机、认程序）、分发完善、代码纪律严苛（forbid unsafe + pedantic clippy + 243 个测试 + CodeQL）。它的源码尤其适合两类读者：想学 Iced/Elm 架构做实时 GUI 的 Rust 开发者，以及想看"libpcap 之上如何组织一个现代抓包应用"的网络编程学习者——从双线程抓包管线到编译期服务表，每个模块都短小且自洽。作为工具，它是个人用户回答"我的流量去哪了"的最佳免费选择之一；作为项目，它是单维护者开源项目健康运转的范例，只是长期押注在一位作者的持续投入上。正在开发的 IPFIX 收集器若顺利发布，将从"主机嗅探器"升级为可接收远程流量的轻量采集分析站，值得持续关注。
