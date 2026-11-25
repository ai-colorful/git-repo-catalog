# ruvnet/RuView 项目分析

> 星数 99267（快照 2026-10-24）｜主语言 Rust｜档位 50k-100k｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：RuView（内部名 wifi-densepose）是一个把普通 WiFi 信号（CSI，信道状态信息）变成空间智能的感知平台——无摄像头、无穿戴设备，穿墙检测人体存在、呼吸/心率、17 关键点姿态，并直连智能家居生态。

| 项目 | 事实 |
|---|---|
| 名称 | ruvnet/RuView（"π RuView"），homepage: Cognitum.One/RuView（gh api） |
| 星数 | 快照 99267；分析当日 gh api 实测 101693 stars、13210 forks、516 open issues |
| 语言 | Rust（`v2/` 工作区 60+ crates，约 35 万行 .rs，排除 vendor）；Python 为冻结参考实现与 PyPI 绑定 |
| 许可证 | MIT（`LICENSE`，Copyright 2024 rUv） |
| 创建/最近推送 | 2025-06-07 创建；pushed_at 2026-10-30（分析当天，gh api） |
| 硬件载体 | ESP32-S3/C6/C5（约 $7 节点）+ 联发科 MT7981 路由器（`firmware/esp32-csi-node/`、`crates/wifi-densepose-mtk-bridge/`） |

## 2. 项目是什么，解决什么问题

WiFi 路由器发出的无线电波充满整个空间，人体移动、呼吸甚至静坐都会以可测量的方式扰动这些波。RuView 用低成本 ESP32 上的 CSI 采集（`firmware/esp32-csi-node/README.md`）捕获这些扰动，在 Rust 后端实时解算为：存在/人数检测、呼吸率（0.1–0.5 Hz 带通）、心率（0.8–2.0 Hz 带通 + 自相关峰）、姿态（17 关键点）、活动识别、房间级 RF 指纹。README 开篇即概括为 "See through walls with WiFi"（`README.md`）。

与学术 demo 的区别在于它做成了"产品化全家桶"：一键接入 Home Assistant（MQTT HA-DISCO 发布器）、Apple Home（HAP-1.1 bridge）、Matter bridge、Google/Alexa（`docs/integrations/home-assistant.md`、ADR-115/ADR-122），每节点对外发布 21 个实体（11 个原始信号 + 10 个语义状态如 someone-sleeping、possible-distress），另有 3 个 HA Blueprint（README）。本地自动化由 HOMECORE 承担——这是 Home Assistant 核心状态机的 Rust 移植（`v2/crates/homecore/src/lib.rs`，ADR-127）。

隐私是这个项目的核心叙事而非装饰：BFLD（ADR-118）专门度量和门控 WiFi 波束赋形反馈捕获中的"身份泄露"风险（`v2/crates/wifi-densepose-bfld/`），PrivShield 提供"合规波形反感知"（ADR-291），`--privacy-mode` 会从 REST/WS/录音中剥离生物特征字段（`CHANGELOG.md` PR #2125）。

值得一提的是项目的"证据文化"：`PROOF.md` 开头直言该项目曾被公开质疑为 "AI slop / fake"，回应方式是 `scripts/prove.sh` 一键复现脚本，把每条主张标注为 MEASURED（本机复现且有失败的回归测试钉住）/ CLAIMED（引用来源未复现）/ DATA-GATED、HARDWARE-GATED；固件 README 用"当前契约"表格诚实标注呼吸/心率输出为 "Experimental… not a medical measurement"（`firmware/esp32-csi-node/README.md`）。这种自我标注贯穿 CHANGELOG 与文档。

## 3. 技术栈与依赖

- **Rust 1.89（stable）**：核心运行时。`v2/Cargo.toml` 为 workspace（resolver 2），`v2/rust-toolchain.toml` 钉 1.89。
- **信号处理**：`ndarray`、`num-complex`（`v2/crates/wifi-densepose-signal/src/csi_processor.rs`）；自研 DSP（Hampel 滤波、Fresnel 区、相位净化、子载波选择）。
- **NN 推理三后端**：ONNX Runtime（`ort`）、tch-rs、Candle，特性开关切换（`v2/crates/wifi-densepose-nn/Cargo.toml`、`src/inference.rs`）；RF 编码器为纯 Rust f32 ABI（`src/rf_encoder.rs`，ADR-146）。
- **服务端**：Axum（REST + WebSocket + 静态 UI），tokio（`v2/crates/wifi-densepose-sensing-server/src/main.rs`）。
- **状态/并发**：DashMap + tokio broadcast（`v2/crates/homecore/src/lib.rs`）。
- **WASM**：wifi-densepose-wasm/-wasm-edge、homecore-plugins 的签名 Wasm 插件运行时（`v2/Cargo.toml` 注释，ADR-128）。
- **Python**：`python/` 为独立 maturin/pyo3 绑定 crate（`wifi-densepose-py` 2.0.0-alpha.1，`python/Cargo.toml`），PyPI 包名 `wifi-densepose`；`archive/v1/` 是重写前的 Python 参考管线（冻结，CI 仅作卫生检查且 continue-on-error，`​.github/workflows/ci.yml`）。
- **固件**：ESP-IDF v5.4/v6.0.2，C 语言 + WASM 边缘模块（`firmware/esp32-csi-node/CMakeLists.txt`）。
- **Node**：`harness/ruview`（@ruvnet/ruview 0.9.3，MCP server + Claude Code mod）、`dashboard/`（Vite + TS + Playwright）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `v2/crates/` | 核心：60+ Rust crate 的 workspace（signal/nn/vitals/engine/sensing-server/homecore 家族/cog-\*/隐私层） |
| `firmware/esp32-csi-node/` | ESP32 固件：CSI 采集、UDP 流、WASM 模块热插拔、OTA、多目标 sdkconfig |
| `python/` | PyPI 绑定（pyo3/maturin），独立于 v2 workspace（ADR-117 §5.2） |
| `archive/v1/` | Python 旧版管线（冻结参考） |
| `harness/` | 面向人/AI agent 的操作层：@ruvnet/ruview MCP、homecore、ruview-kernel（WASM） |
| `ui/` | sensing-server 8080 端口托管的静态仪表盘（observatory、pose-fusion） |
| `dashboard/` | Vite + TS + Playwright 的新版仪表盘 |
| `docs/` | 1034 个 .md，含 **322 篇 ADR**（`docs/adr/`）、用户指南、基准、安全审计 |
| `scripts/`、`Makefile`、`install.sh` | 复现脚本 `prove.sh`、分 profile 安装器 |
| `.github/workflows/` | 约 26 条 workflow（见第 7 章） |

组织方式评述：这是典型的"monorepo + 多 crate 工作区"，但规模罕见地大——顶层 45 个目录、Rust 963 个文件约 35 万行（排除 vendor），配合 322 篇 ADR 做设计溯源。模块按领域垂直切分（signal → nn → engine → server → integrations），crate 间通过 workspace path 依赖组合；部分组件（worldgraph）拆到外部 submodule。README 与 crate 文档存在少量漂移：`v2/crates/README.md` 仍列出已删除的 wifi-densepose-config/-db crate（`v2/Cargo.toml` 注释明确 #578 移除）。

## 5. 核心架构与关键实现

### 5.1 CSI 信号管线（`v2/crates/wifi-densepose-signal/`，约 2.9 万行）

核心类型 `CsiData`（时间戳、天线×子载波幅度/相位矩阵、中心频率、带宽、SNR，`src/csi_processor.rs`）。模块覆盖 Hampel 离群滤波（`hampel.rs`）、Fresnel 区建模（`fresnel.rs`）、相位净化（`phase_sanitizer.rs`）、子载波选择、频谱图与运动检测；`ruvsense/` 子模块实现多基站融合（multistatic fuser）、RF-SLAM 与质量评分，被 engine 组合调用（`v2/crates/wifi-densepose-engine/src/lib.rs` 的 use 列表）。

### 5.2 生命体征四级管线（`v2/crates/wifi-densepose-vitals/src/lib.rs`，ADR-021）

文档注释明确定义四级流水：EMA 静态分量抑制 → 呼吸 0.1–0.5 Hz 带通 + 过零 BPM → 心率 0.8–2.0 Hz 带通 + 自相关峰检测 + 子载波相位一致性加权 → Welford 运行统计的 z-score 异常告警（呼吸暂停/心动过速/过缓）。另有 `groundtruth.rs`（ADR-293）对照参考设备 CSV 做 Bland-Altman 一致性评级——把"准不准"也工程化了。

### 5.3 信任可追溯的组合层（`v2/crates/wifi-densepose-engine/src/lib.rs`）

`#![forbid(unsafe_code)]` 的组合根。`StreamingEngine::process_cycle` 四步：融合打分 → 盖校准版本戳 → 隐私降级（若融合记录了可容忍矛盾，隐私等级单调降一级，信息只减不增）→ 语义状态写入 WorldGraph 并强制挂 provenance 与 DerivedFrom 边。每个 `TrustedOutput` 必须自述证据引用（ADR-137）、模型版本（ADR-136）、校准版本与隐私决策。

### 5.4 传感服务器与生态接入（`v2/crates/wifi-densepose-sensing-server/`，约 5.9 万行，最大 crate）

Axum 单体：UDP 5005 收 ESP32 CSI 帧 → 信号处理 → WS 8765 广播 → 8080 托管 UI；内置 MediaTek MT7981 MTC1 桥、Qualcomm/Realtek CSI 解析器、Matter 端点、bearer 认证与 edge registry（`src/main.rs` 模块表）。CHANGELOG 社区批次加固了"关认证时拒绝可路由绑定、隐私模式剥离生物特征、陈旧 `/sensing/latest` 报 stale"等真实安全问题（PR #2134/#2125/#2119）。

## 6. 构建与运行

```bash
# 安装（交互式 / 分 profile：verify|python|rust|browser|iot|docker|field|full）
./install.sh                       # 或 ./install.sh --profile rust --yes   （install.sh）
make check                         # 仅硬件/环境检测（Makefile）
make build-rust && make build-wasm # Rust 构建 + WASM 构建（Makefile）
make test-rust && make bench       # 测试与基准（Makefile）
make run-api                       # 运行 API（Makefile）

# 复现一切主张（项目招牌）
bash scripts/prove.sh [--full]     # scripts/prove.sh：MEASURED/CLAIMED 证据分级

# 固件（ESP-IDF 6.0.2 / >=5.5.2）
cd firmware/esp32-csi-node
idf.py -B build-c5 -D IDF_TARGET=esp32c5 -D SDKCONFIG=build-c5/sdkconfig build

# AI 操作层
npx @ruvnet/ruview@0.9.1 doctor|devices|guidance   # README
# Python 绑定
pip install wifi-densepose          # python/README.md
```

## 7. 测试与工程质量

- **Rust 测试规模可观**：`v2/crates` 中 5459 处 `#[test]`，分布在 612 个 .rs 文件（本次解压树内 grep 统计）；含集成测试如 sensing-server 的多节点 UDP 测试（`scripts/prove.sh` 注释提到 `tests/multi_node_test.rs`）。
- **Python**：`python/tests` 13 个文件；`archive/v1` 冻结管线仍有 lint/类型检查但全部 continue-on-error（`​.github/workflows/ci.yml`）。
- **前端**：`dashboard/` 带 Playwright 配置与 `ui/sw.test.mjs`。
- **CI 矩阵专业**：约 26 条 workflow——ci.yml、python-ci、firmware-ci、firmware-qemu（QEMU 跑固件）、mqtt-integration、bfld-mqtt-integration、bench-regression、model-release-gate、nightly-sota-agent（AI agent 夜间跑 SOTA）、readme-visuals（README SVG 资产自动化）等（`.github/workflows/`）。Actions 均按 SHA 钉版本。
- **质量观察**：crate 文档注释密度高、几乎处处引用 ADR 编号；`wifi-densepose-engine` 显式 forbid unsafe；CHANGELOG 直接记录"已知未修复问题"（如混合 192/306 帧形丢帧未关 #1894），这种坦率在同类项目里少见。基准表（如 "Full Pipeline ~810x 提速"，`v2/crates/README.md`）为项目自测数据，未注明第三方复现。

## 8. 社区与维护状态

- **极度活跃**：pushed_at 2026-10-30（分析当天）；release 高频滚动——最新 5 个 release 中有 CI 快照 v3446（2026-10-29）、固件 v0.8.17（gh api releases）。
- **贡献者**：contributors 分页计数约 45（per_page=1 翻页，含 anon，未确认去重口径）；CHANGELOG 记录了"aepod 提交 PR #2100–#2157 社区批次、集中评审合入"的维护模式，说明有真实外部贡献流。
- **文档驱动**：322 篇 ADR + WITNESS-LOG（`docs/WITNESS-LOG-110.md`）+ 1065 行 CHANGELOG；根目录 `AGENTS.md`/`CLAUDE.md` 是给 AI 编码 agent 的仓库契约，佐证大量代码由 agent 辅助产出。
- **增长异常快**：2025-06 创建，17 个月到 10 万星、1.3 万 fork——Star 增速与内容体量在 GitHub 罕见，README 的大量自制 SVG 营销资产与"千星流水线"式传播明显起了作用；真实性无法从仓库内证实，标注为"未确认"。

## 9. 亮点与风险

**亮点**
1. 端到端完整：从 $7 ESP32 固件（ADR-018 二进制格式、QEMU CI）到 Rust DSP/NN 到智能家居四生态接入，全链路在一个仓库闭环（`firmware/` → `v2/crates/` → `docs/integrations/`）。
2. 证据分级文化独树一帜：`PROOF.md` + `scripts/prove.sh` 把每条能力主张变成可复现命令，MEASURED/CLAIMED 分级并在 README/固件文档中如实标注"不是医疗测量"。
3. 322 篇 ADR + 5459 个 Rust 测试的设计溯源密度，工程纪律远超同体量开源项目平均水平。
4. 隐私被当作一等架构约束（BFLD 身份泄露门控、PrivShield、隐私降级单调性设计，`wifi-densepose-engine`），而非事后补丁。
5. NN 推理三后端抽象（ONNX/tch/Candle）+ 纯 Rust f32 张量 ABI，兼顾可部署性与确定性（`wifi-densepose-nn/`）。

**风险/局限**
1. 精度主张部分仍是 CLAIMED：README 自己承认统一 RF 世界模型 "accuracy is still synthetic until real-data validation"；生命体征输出为实验级（`firmware/esp32-csi-node/README.md` 能力表）。
2. 极度激进的营销外衣（自制 SVG 动画、千倍提速自测表、三个月千星节奏）与"曾被公开质疑 AI slop"的历史，需要使用者自己跑 `prove.sh` 验证。
3. 仓库过大导致漂移：crate README 与 workspace 清单不一致（config/-db 已删仍列）、CHANGELOG 自认遗留 #1894 帧形丢帧；60+ crate 的维护负担集中在单一作者（rUv）+ agent 流水线。
4. 生命体征/姿态这类能力天然敏感，误报与滥用（如未告知的监测）风险由部署者自担，MIT 许可无任何使用限制。

## 10. 总结评价

RuView 是近年少见的一个"学术热点工程化"样本：把 WiFi CSI 感知从论文搬进了带固件、带 CI、带智能家居生态和隐私门控的产品级 monorepo，并用 ADR + 证据分级 + 一键复现脚本正面回应了"AI 生成代码不可信"的质疑——它对自身局限的标注（CLAIMED、Experimental、not medical）比多数同类项目诚实得多，但营销包装的激进程度同样罕见，星数增速也远超常规。适合三类人：想做 RF 感知/DSP 落地的工程师（读 `wifi-densepose-signal`、`wifi-densepose-vitals` 与固件源码，这是仓库最扎实的部分）；Home Assistant/Matter 深度玩家（直接用 HA 接入与 HOMECORE）；研究 AI-agent 大规模协作开发流程的人（AGENTS.md/CLAUDE.md/ADR/prove.sh 这套"agent 时代工程治理"本身就是研究对象）。医疗、安防等严肃场景请先跑 `scripts/prove.sh` 并对照 MEASURED/CLAIMED 清单自行评估。
