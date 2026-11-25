# localsend/localsend 项目分析

> 星数 95326（快照 2026-10-24）｜主语言 Dart｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：开源免费的"隔空投送"跨平台替代品——在同一局域网内通过 REST API + HTTPS 在附近设备间安全互传文件与消息，无需互联网和第三方服务器。

| 项目 | 内容 |
|---|---|
| 名称 | localsend/localsend |
| 星数 | 95326（快照 2026-10-24；分析日 gh api 实测 96661） |
| 语言 | Dart（Flutter UI）+ Rust（协议核心，双语言 monorepo） |
| 许可证 | Apache-2.0（gh api `license.spdx_id`） |
| 创建时间 | 2022-12-16（gh api） |
| 最近 push | 2026-10-29（分析前一天，非常活跃） |
| 最新 release | v1.18.2（2026-08-21，gh api） |
| 默认分支 | main |

## 2. 项目是什么，解决什么问题

LocalSend 解决的是"跨生态设备间传文件难"的问题：iPhone 传 Android、Windows 传 macOS 等场景下，各厂商的近场传输方案（AirDrop、Quick Share、华为分享）互不相通，而微信/网盘中转则慢、有大小限制且隐私存疑。LocalSend 让同一 Wi-Fi 局域网内的设备直接互相发现并传输，数据不离开局域网（README "About" 节）。

其工作方式是：每台设备运行一个内嵌 HTTPS 服务器（默认端口 53317，TCP+UDP，README "Setup" 表格），设备间用标准 REST API 通信；TLS 证书由各设备运行时动态自签生成，"确保最大安全性"（README "How It Works"）。协议规范独立开源在 localsend/protocol 仓库，因此第三方可以实现兼容客户端（仓库内 `cli/` 即是基于 Protocol v2 的官方终端客户端）。

分发覆盖极广：Windows（winget/Scoop/Chocolatey/EXE/Portable ZIP）、macOS（App Store/Homebrew/DMG）、Linux（Flathub/Nixpkgs/Snap/AUR/TAR/DEB/AppImage）、Android（Play/F-Droid/APK/Amazon Fire OS）、iOS（App Store），共六大平台约 20 种渠道（README "Download" 表格）。应用无自动更新，依赖包管理器分发（README 明确说明）。

兼容面从 Android 7.0 / iOS 13 / macOS 11 / Windows 10 起步（README "Compatibility" 表）。特色功能包括：接收端可 PIN 码校验、通过浏览器链接下载文件（web download，v1.18.2 更新日志）、收藏设备直连、便携模式（exe 旁放 `settings.json`）、`--hidden` 托盘启动（README "Portable Mode"/"Start hidden"）。

## 3. 技术栈与依赖

- **UI 层**：Flutter/Dart，SDK `^3.11.0`（根 `pubspec.yaml`），CI 锁定 Flutter 3.41.9（`.github/workflows/ci.yml`），用 fvm 管理版本（README "Getting Started"）
- **状态管理**：refena（`app/pubspec.yaml` 中 `refena_flutter: 3.5.0`，`app/lib/main.dart` 用 `RefenaContainer`/`RefenaScope` 初始化）——注意不是 riverpod
- **路由**：routerino；**i18n**：slang（`app/assets/i18n/` 共 118 个翻译文件，Weblate 平台托管）
- **模型代码生成**：dart_mappable + freezed_annotation + build_runner（`app/pubspec.yaml` dev_dependencies）
- **Rust 核心与生态**（根 `Cargo.toml` workspace，4 个 crate）：
  - `packages/core`：协议实现，tokio 异步运行时 + hyper 手写 HTTP 路由、rcgen/rsa（自签证书）、x509-parser（证书校验）、reqwest（客户端）
  - `cli`：终端客户端（clap 交互式设备选择）
  - `server`：axum WebSocket 信令服务器（`server/README.md`）
  - `packages/localsend_isolates/rust`：Flutter 插件 crate，经 flutter_rust_bridge（FRB）暴露给 Dart（`packages/localsend_isolates/flutter_rust_bridge.yaml`）
- **桥接**：flutter_rust_bridge + cargokit（`packages/localsend_isolates/rust_builder`），Dart 侧配套 `packages/typed_isolates`（类型化 Isolate 通道）

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `app/` | Flutter 应用（org.localsend.localsend_app）：UI、provider、持久化、平台通道，273 个 dart 文件 |
| `packages/core/` | Rust crate `localsend`：协议、HTTP 服务端/客户端、加密、多播、WebRTC；不依赖 Flutter |
| `packages/localsend_isolates/` | Dart isolate 层 + FRB 绑定 + `rust/`（插件 crate）+ `rust_builder/`（cargokit） |
| `packages/typed_isolates/` | 独立小包：给 Dart Isolate 包一层类型化收发通道 |
| `cli/` | Rust CLI crate（localsend-cli），基于 packages/core 的 v2 HTTP + 多播 |
| `server/` | Rust WebRTC 信令服务器（WebSocket `/v1/ws`），带 Dockerfile 独立部署 |
| `support/` | 文档与脚本：`support/scripts/` 放各平台打包脚本（MSIX、Inno Setup、FOSS 剥离） |
| `fastlane/` | F-Droid 打包元数据 |
| `.github/workflows/` | 18 个工作流：1 个 ci.yml + 各平台构建/发布（含 ARM64 变体）+ winget 同步 |

组织方式评述：这是一个"UI（Dart）与协议（Rust）彻底分离"的多语言 monorepo（根 `AGENTS.md` 的仓库布局表格是权威描述）。四个 Rust crate 共享一个 Cargo workspace（单一 `Cargo.lock`、根级 `[profile.*]`）。历史脉络清晰：协议栈正从 Dart 整体迁往 Rust，Dart 侧只剩 UI 与 isolate 调度；`app/lib/provider/network/server/server_provider.dart` 注释直言 "The HTTP server itself runs in Rust (inside the server isolate)"。

## 5. 核心架构与关键实现

### 5.1 发现协议：UDP 多播（packages/core/src/multicast/mod.rs）

设备向多播组周期性广播 JSON 自述（别名、协议版本、设备型号/类型、指纹、HTTP 端口、是否 TLS），**UDP 只发不应**——收到广播的设备改为向广播者发 HTTP register 请求来"回应"：

```rust
pub const DEFAULT_MULTICAST_GROUP: Ipv4Addr = Ipv4Addr::new(224, 0, 0, 167);
pub const DEFAULT_MULTICAST_GROUP_V6: Ipv6Addr =
    Ipv6Addr::new(0xff12, 0, 0, 0, 0, 0, 0xfd3a, 0xe420);
pub const DEFAULT_PORT: u16 = 53317;
const ANNOUNCE_DELAYS: [Duration; 3] = [
    Duration::from_millis(100),
    Duration::from_millis(500),
    Duration::from_millis(2000),
];
```

（packages/core/src/multicast/mod.rs 第 28-49 行）

工程细节扎实：选 `224.0.0.0/24` 段是因部分 Android 设备只收得到该段的 UDP 多播；每次上线连发 3 个包对抗丢包（100ms/500ms/2000ms 间隔）；每个网卡各绑一个 socket，连续 10 次接收错误才放弃该网卡（容忍 Windows 上 ICMP 错误浮到下次 receive 的怪癖）；用 fingerprint 字段过滤自己开环回的广播；iOS 挂起时 socket 被系统回收，会发 `SocketsFailed` 事件让应用重启发现。IPv6 发现是 v2.2 之上的 LocalSend 扩展，与 IPv4 并行宣告。

### 5.2 传输协议：HTTPS REST API（packages/core/src/http/server/mod.rs, v2.rs）

服务端是手写 hyper 路由表（`mod.rs` 第 631-682 行的 match 分发）：

| 端点 | 作用 |
|---|---|
| `POST /api/localsend/v2/register` | 设备注册（发现的 HTTP 回应） |
| `GET /api/localsend/v1\|v2/info` | 查询设备信息 |
| `POST /api/localsend/v2/prepare-upload` | 发送方报价文件清单，接收方决定收哪些 |
| `POST /api/localsend/v2/upload` | 按会话+文件 ID 上传内容 |
| `POST /api/localsend/v2/cancel` | 取消会话 |
| `POST /api/localsend/v2/prepare-download` + `GET .../download` | 反向：浏览器/接收方拉取 |
| `POST /api/localsend/v2/show` | 远程唤起接收界面（.internal 用途） |
| `POST /api/localsend/v3/nonce`、`v3/register` | v3 协议草案端点（与 v2 并存） |

架构是事件驱动的核心-应用解耦：服务器不发业务决策，而是把事件抛给上层应用，用 oneshot channel 等应用回填决策。例如上传请求事件：

```rust
PrepareUpload {
    session_id: String,
    ip: PeerIp,
    info: RegisterDtoV2,
    // mTLS 握手中验证过的发送方证书指纹，不可伪造
    cert_fingerprint: Option<String>,
    files: HashMap<String, FileDto>,
    decision_tx: oneshot::Sender<PrepareUploadDecisionV2>,
},
```

（packages/core/src/http/server/v2.rs 第 46-69 行；`Accept(HashSet)` 响应 204/206，`Decline` 响应 403）

安全设计：mTLS 强制客户端证书，`Register`/`PrepareUpload` 事件携带的 `cert_fingerprint` 来自 TLS 握手验证，注释明确 "Unlike `info.fingerprint`, this value cannot be spoofed"（v2.rs 第 58-62 行）；v1.18.2 还加入"不跟随对端 HTTP 重定向"的加固（`app/assets/CHANGELOG.md`）。

### 5.3 设备身份：动态自签证书（packages/core/src/crypto/cert.rs）

`generate_self_signed()` 运行时生成 RSA-2048 密钥对与自签证书，`CN=LocalSend User` 且不带 SAN——设备身份完全不依赖证书内容，而以证书 SHA-256 指纹为准（第 7-31 行文档注释：HTTPS 模式下 fingerprint 即证书哈希）。这个指纹同时承担多播自环过滤与 mTLS 对端识别双重职责，设计自洽。

### 5.4 Dart/Rust 桥与 provider 架构（app/lib/provider/）

- `packages/localsend_isolates/lib/isolate.dart` 导出 parent/child 两套 isolate API：HTTP 服务器整体跑在 child isolate 的 Rust 侧（`server_isolate.dart`），事件（`HttpServerPrepareUploadEvent` 等）回传 parent isolate，Dart 侧 `serverProvider` 启动它、监听事件、持有 `ServerState`（`app/lib/provider/network/server/server_provider.dart` 头部注释）
- 发送流程集中在 `send_provider.dart`（851 行，仓库内最大的 provider）：`SendNotifier` 管理 `Map<String, SendSessionState>` 发送会话，校验和计算与 prepare-upload 请求都持 Rust 侧取消令牌（`rust_cancel.RsCancellationToken`），确认 HTTP 客户端也已下沉 Rust
- `nearby_devices_provider.dart`（259 行）聚合多播事件与手动 IP 探测；`scan_facade.dart` 统一扫描入口
- 全局 22 个 provider 覆盖设置、收藏、接收历史、安全（PIN）、内购（`[FOSS_REMOVE]` 标记供 F-Droid 剥离）、TV 模式等

## 6. 构建与运行

```bash
# 开发运行（README "Getting Started"；注意须用 .fvmrc 锁定的旧版 Flutter）
cd app && fvm flutter pub get && fvm flutter run

# 各平台打包（README "Building"，maintainers only）
flutter build apk          # Android（appbundle/ipa/windows/macos/linux 同理）
flutter pub run msix:create --store   # Windows Store 包
appimage-builder --recipe AppImageBuilder.yml   # Linux AppImage

# Rust 侧（根 workspace）
cargo build -p localsend-cli          # CLI
cargo test -p localsend               # 协议核心测试

# CLI 安装（macOS/Linux）
brew install localsend-cli            # README "Command Line Interface"

# 防火墙（README "Setup"）
sudo ufw allow 53317                  # TCP+UDP
```

## 7. 测试与工程质量

- **Rust 集成测试 10 个**（`packages/core/tests/`）：`v2_server.rs`、`v2_tls_pinning.rs`（TLS 指纹绑定）、`multicast.rs`、`discovery.rs`、`connections.rs`、`hash.rs`、`internal_server.rs`、`v2_web_download.rs`、`accept_resilience.rs`、`event_backpressure.rs`——覆盖面直指协议关键路径，`multicast/mod.rs` 内还有单测验证 v2 兼容的 `announce` 旗标序列化
- **Dart 测试 21 个文件**（`app/test/`，分 `unit/`（model/provider/util/i18n）与 `widget/`，用 mockito 生成 mock）
- **CI**（`.github/workflows/ci.yml`）：format（`dart format --set-exit-if-changed`）→ test → analyze；另有 17 个平台构建工作流（含 linux arm64 deb/tar、windows arm64 zip 等变体）与 winget 版本同步
- **质量观察**：`packages/core` 全库密集的 rustdoc 注释（多到能直接当协议文档读）；根目录有 `AGENTS.md`/`CLAUDE.md` 给 AI 编码工具立规矩，并明确"不接受 AI 生成的贡献，除非是 bug 修复/极小改动/能证明专业能力"；`[profile.dev.package.rsa] opt-level = 2` 这种注释解释了为什么——RSA 生成在 debug 下慢 10 倍

## 8. 社区与维护状态

- 活跃度：push 时间 2026-10-29（分析前一天）；open issues 1013（体量大但符合近 10 万星项目水位）（gh api）
- 贡献者：contributors API 首页 100 人满页（≥100 人，精确数未确认）；CHANGELOG 每条都署名贡献者（如 @ShlomoCode、@leopalmieri-spec），单人批量贡献活跃
- Release 节奏：v1.18.2（2026-08-21）→ v1.18.1（2026-08-11，移动端热修）→ v1.18.0；Unreleased 已积累 5 条修复，约月度级节奏（`app/assets/CHANGELOG.md`）
- 国际化：Weblate 托管、118 个 locale 文件（`app/assets/i18n/`），README 有翻译徽章
- 双镜像：GitHub + Codeberg 并行（README 头部链接）；另有 Discord 社区
- 商业化：`purchase_provider.dart` + `in_app_purchase: 3.0.0 # [FOSS_REMOVE]` 表明应用商店版带内购支持开发者，F-Droid 构建自动剥离

## 9. 亮点与风险

**亮点**

1. 无服务器、无互联网的局域网互传，协议规范独立开源（localsend/protocol），形成可互操作的开放标准，CLI 即第三方实现样板
2. "Dart UI + Rust 协议核"的架构迁移彻底且干净：服务器跑在 Rust isolate、事件驱动解耦、oneshot channel 回填决策，UI 框架可整体替换而不动协议
3. 安全设计自洽：动态自签证书 + SHA-256 指纹即身份 + mTLS 客户端证书防伪造 + 不跟随对端重定向
4. 分发矩阵近乎满分：6 平台约 20 渠道（含 ARM64），Windows 签名策略公开（CODE_SIGNING.md）
5. 多播实现细节考虑了大量真实世界坑（Android 多播段限制、iOS 挂起回收 socket、Windows ICMP 怪癖、双栈去重）

**风险/局限**

1. 双语言栈维护成本高：Dart + Rust + FRB 代码生成 + cargokit + fvm 旧版 Flutter 锁定（README 专门提醒构建问题多来自版本不匹配），新人上手门槛陡
2. 协议迁移过渡期并存 v1/v2/v3 端点与新旧 Dart 实现残留，理解成本与回归面变大
3. 依赖局域网环境质量：路由器 AP 隔离、VPN、Windows 公共网络策略都会导致设备不可见（README Troubleshooting 表），Android SAF 流速慢为已知上游问题
4. 无自动更新（README 明说），完全依赖包管理器/商店的滞后分发

## 10. 总结评价

LocalSend 是"用标准 Web 技术解决封闭生态互联"的典范项目：不做魔法配对，就用 HTTPS + 自签证书 + UDP 多播把 AirDrop 的体验搬到了所有平台，并以开源协议规范建立了跨实现的互操作标准。代码层面正处于 Dart 到 Rust 的架构迁移收官阶段，`packages/core` 的 Rust 协议实现文档密度和测试针对性都属上乘，是学习"跨语言 Flutter 应用工程化"与"局域网 P2P 协议设计"的一流教材。适合：想要跨生态传文件的普通用户（开箱即用）、研究 mTLS/多播/证书指纹设计的网络工程师、以及想借鉴 Flutter+Rust 混合架构的应用开发者读源码。
