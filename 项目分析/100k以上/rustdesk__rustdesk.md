# rustdesk/rustdesk 项目分析

> 星数 126706（快照 2026-10-24）｜主语言 Rust｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：开源自托管的远程桌面软件，Rust 内核 + Flutter 界面，可完全自建中继服务器、数据不经第三方。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | rustdesk/rustdesk（当前版本 1.5.1，master） | `Cargo.toml` |
| 星数 | 126706（快照），分析时 gh api 实测 127497 | 任务参数 / gh api |
| 语言 | Rust 约 18.7 万行 + Dart 约 7.9 万行 | 仓库内 wc -l 统计 |
| 许可证 | AGPL-3.0（文件名 LICENCE） | `LICENCE`、gh api |
| 最近活跃 | push 于 2026-10-29（前一天），最新 release 1.5.0（2026-09-30） | gh api |
| 平台 | Windows / macOS / Linux(X11+Wayland) / Android / iOS | `libs/scrap/src/`、`flutter/` |

## 2. 项目是什么，解决什么问题

RustDesk 是一个"开箱即用、无需配置"的远程桌面方案（`README.md`），对标 TeamViewer/AnyDesk：受控端与控制端通过 ID + 密码建立会话，提供屏幕控制、文件传输、TCP 隧道、远程打印、终端等能力。核心卖点是**数据自主权**：官方 rendezvous（hbbs）/relay（hbbr）服务器只是可选的公共入口，用户可以自建服务器（`README.md` 指向 rustdesk/rustdesk-server 与 rustdesk-server-demo），甚至自己实现 rendezvous 协议服务端。

连接建立采用多级降级策略（`src/rendezvous_mediator.rs`）：受控端先向 hbbs 注册公钥与 NAT 类型，控制端请求连接时优先 UDP 打洞直连，失败则 TCP 打洞，再失败走 hbbr 中继，另支持 WebRTC 数据通道与 KCP（`Cargo.toml` 中 kcp-sys、webrtc fork 补丁，`src/rendezvous_mediator.rs` 的 PunchPool/ICE 逻辑）。全程密钥由端侧协商，服务器不经手明文。

仓库本体是客户端（含被控服务），服务端代码独立在 rustdesk/rustdesk-server；两仓共享协议库 hbb_common（以 git submodule 引入，`.gitmodules`、`AGENTS.md`）。README 顶部有明确的"禁止滥用"免责声明，说明团队意识到远控软件的灰色用途风险。

盈利模式为开源核心 + Pro 服务器（README 中 RustDesk Server Pro 徽章），社区版功能完整，这也是其能在 127k 星规模的远程桌面赛道立足的关键。

## 3. 技术栈与依赖

- **核心语言**：Rust（edition 2021，rust-version 1.75），tokio 异步运行时贯穿全部网络层（`Cargo.toml`）。
- **UI**：Flutter（`flutter/`，pubspec 名 flutter_hbb 1.5.1+68）经 flutter_rust_bridge 1.80 与 Rust 交互；旧 Sciter UI 保留但已弃用（`src/ui/`，README File Structure）。
- **编解码**：libvpx（VP8/VP9）、aom（AV1）、H264/H265 走 hwcodec/vram/mediacodec 硬编解，libyuv 做色彩转换、opus 音频（README vcpkg 安装清单、`libs/scrap/src/common/codec.rs`）。
- **采集**：`libs/scrap` 按 DirectX DXGI / X11 / Wayland / macOS Quartz / Android 分平台实现。
- **输入/剪贴板**：`libs/enigo`（fork 自 enigo）、`libs/clipboard`、arboard fork。
- **加密**：sodiumoxide（libsodium：Ed25519 签名 + X25519 + secretbox）、totp-rs（2FA）、ring/openssl/tls 由 reqwest 引入（`Cargo.toml`、`src/common.rs`）。
- **显著特征**：大量依赖固定为 rustdesk-org 下的 fork（rdev、arboard、webrtc、tungstenite、cpal 等 15+ 个，`Cargo.toml` [patch.crates-io] 与 git 依赖），webrtc fork 注释详述了修复的 IPv6/SCTP 拥塞控制问题。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `src/` | 应用主体：客户端会话、被控服务、平台层 |
| `src/server/` | 被控端：connection（连接/鉴权）、video/audio/input/clipboard/terminal 各服务 |
| `src/client/`、`src/client.rs` | 控制端会话与消息处理（5911 行） |
| `src/rendezvous_mediator.rs` | 与 hbbs 通信：注册、打洞、中继（1811 行） |
| `src/platform/`、`src/ipc/` | 平台相关代码与进程间通信 |
| `libs/hbb_common/` | 与服务端共享的 crate（rendezvous proto、socket 封装、Config）——git submodule，tarball 中为空 |
| `libs/base/` | 客户端专用 crate：message.proto（1041 行）、文件传输 fs、配置键 |
| `libs/scrap/` | 屏幕采集 + 编解码器封装（codec.rs 1157 行） |
| `libs/enigo/`、`libs/clipboard/` | 键鼠注入、剪贴板 |
| `libs/virtual_display/`、`libs/remote_printer/`、`libs/portable/` | 虚拟显示器、远程打印、便携化 |
| `flutter/` | Flutter 前端（lib/ 下 7.9 万行 Dart，含 test/ 23 个测试） |
| `res/`、`flatpak/`、`appimage/`、`fastlane/` | 打包分发资源 |
| `build.py` | 60KB 的全平台构建编排脚本 |

组织方式评述：单体仓库内的 Cargo workspace（9 个成员 crate，`Cargo.toml` [workspace]），按"协议共享层(hbb_common) / 客户端基础层(base) / 能力库(scrap/enigo/clipboard) / 应用层(src,flutter)"分层；hbb_common 因跨仓共享被拆为 submodule，`AGENTS.md` 明确说明"改它要多一轮 PR，客户端专属代码应放 libs/base"，这是仓库自身认可的架构约束。

## 5. 核心架构与关键实现

### 5.1 连接建立与中继协议（src/rendezvous_mediator.rs）

受控端常驻 RendezvousMediator：向 hbbs 周期性 register_pk（携带 Ed25519 公钥，`register_pk` 于 1244 行），响应 PunchHole / RequestRelay 两种入站请求（539、546 行的 message union 分派）。打洞资源有精心设计的并发预算——UDP/TCP 各 32 个槽位的 PunchPool（116-119 行），WebRTC answerer 限 16 个并发（74 行 MAX_WEBRTC_ANSWERERS），注释明确说明这是防未鉴权连接耗尽资源的防线。中继回退时通过 RelayResponse 把 relay 服务地址与一次性 UUID 发给双方（`handle_request_relay` 661 行）。

```rust
// src/rendezvous_mediator.rs
/// Two pools, so that neither transport pays for the other's crowd: a connection
/// costs a place in each, the controller's preferred request punching UDP and its
/// TCP fallback request punching TCP, and the transports that lose the race hold
/// theirs for the whole wait.
struct PunchPool { places: AtomicUsize, max: usize }
static UDP_PUNCHES: PunchPool = PunchPool::new(32);
static TCP_PUNCHES: PunchPool = PunchPool::new(32);
```

### 5.2 被控端连接与鉴权（src/server/connection.rs，7876 行）

Connection 是被控端一切会话的中枢：先发 Hash 挑战（随机 challenge + 自动生成的 6 位临时密码，545-548 行），LoginRequest 到达后按"密码 / 2FA / ID 白名单"三个独立桶做失败限速（191-194 行 LOGIN_FAILURES，注释说明白名单桶不计入密码/2FA 预算），支持 TOTP 2FA（`require_2fa` 字段，377 行；`src/auth_2fa.rs`）、持久密码哈希校验（hbb_common password_security）。登录前有强制 deadline（`login_deadline` 181 行），登录成功后才延迟枚举显示器（576 行注释）。同一文件还管理权限协商、switch-sides（反向控制）、隐私模式等。

### 5.3 端到端密钥协商（src/common.rs）

服务器身份用 Ed25519 验签：`decode_id_pk_dtls` 用 hbbs 公钥验证签名的 IdPk（含 id、对端 X25519 公钥、DTLS 指纹、kx 版本），WebRTC 场景还要求签名指纹与实际协商的 DTLS 指纹一致（`dtls_fingerprint_bound`，空指纹 fail-closed）。会话密钥用 X25519 静态 Diffie-Hellman 封装：

```rust
// src/common.rs:2433
pub fn create_symmetric_key_msg(their_pk_b: [u8; 32]) -> (Bytes, Bytes, secretbox::Key) {
    let their_pk_b = box_::PublicKey(their_pk_b);
    let (our_pk_b, out_sk_b) = box_::gen_keypair();
    let key = secretbox::gen_key();
    let nonce = box_::Nonce([0u8; box_::NONCEBYTES]);
    let sealed_key = box_::seal(&key.0, &nonce, &their_pk_b, &out_sk_b);
    (Vec::from(our_pk_b.0).into(), sealed_key.into(), key)
}
```

即：随机对称密钥用对端 X25519 公钥加密后经服务器转发，只有持有对应私钥的受控端能解开，hbbs/hbbr 全程只见密文。

### 5.4 编解码链路（libs/scrap/src/common/codec.rs + src/client.rs）

编码端按双方 CodecAbility 协商格式（`message.proto` 的 CodecAbility/SupportedEncoding，109-117 行）：默认 VP9，AV1 可用时优先级可配置，检测到双方硬解支持则选 H264/H265（`create_encoder`，202-298 行；vram/GPU 纹理、hwcodec、Android mediacodec 三种硬件路径由 feature 门控）。解码运行在独立视频线程（`src/client.rs` 的 `start_video_thread`，3979 行起）：mpsc 收 MediaData::VideoFrame，VideoHandler 逐帧解码为 ImageRgb/纹理后回调 UI，并处理格式切换、录制（record_screen）与丢帧队列。

## 6. 构建与运行

来自 README 与工程文件（未实际运行验证）：

```bash
# 依赖（各发行版包列表见 README "Raw Steps to build"）
vcpkg install libvpx libyuv opus aom    # 设 VCPKG_ROOT
# 桌面最小运行（Sciter 版，需自取 sciter 动态库）
cargo run
# Flutter 版：先编译 flutter_rust_bridge，再走 .github/workflows/flutter-build.yml 流程
python build.py --flutter --debug      # build.py 为全平台构建编排入口
# Docker 构建
docker build -t rustdesk-builder . && docker run --rm -it -v $PWD:/home/user/rustdesk rustdesk-builder
```

注意：`libs/hbb_common` 为 submodule，clone 需 `--recurse-submodules`（README 中 Docker 小节），tarball 构建会因 hbb_common 缺失而失败。

## 7. 测试与工程质量

- Rust 侧 393 个 `#[test]`/`#[tokio::test]` 分布在 96 个文件（grep 统计），集中在文件传输（`libs/base/src/fs/download_tests.rs`）、音频管线（`src/client/tests/`）、更新元数据（`src/update_metadata_tests.rs`）、编解码硬件探测（`libs/scrap/src/common/hwcodec/tests.rs`）等；无独立 tests/ 目录，多为模块内联测试。
- Flutter 侧 `flutter/test/` 约 23 个 widget/单元测试（终端、快捷键、文件管理、无障碍等）。
- CI（`.github/workflows/`）：ci.yml（Rust 检查）、flutter-build.yml（多平台矩阵发布构建，含自定义 Flutter 引擎替换、Syft 生成 SBOM、update-metadata 签名校验）、flutter-nightly.yml（每夜构建）、fdroid.yml 等 11 个工作流。
- 代码质量观察：`AGENTS.md`（17KB）给出硬性规约——生产代码禁用 unwrap()/expect()；本快照大量模块带有极详尽的英文设计注释（如 rendezvous_mediator.rs 对并发预算的推导），注释密度远超同规模项目平均水平；同时存在 7876 行的 connection.rs 这类超大文件，属于复杂度集中点。

## 8. 社区与维护状态

- gh api（2026-10-30）：star 127497、fork 20154、open issues 206、pushed_at 2026-10-29——以星数体量而言 issue 积压极低，每日仍有提交。
- release 节奏：最新正式版 1.5.0（2026-09-30），master 已推进到 1.5.1（`Cargo.toml`、`flutter/pubspec.yaml`），另有 nightly 持续通道（README）。
- 贡献者结构：未在 tarball 中获得 contributors 数据（未确认）；但 README 提供 26 种语言翻译、指向 CONTRIBUTING.md 与 Discord/Reddit 社区，docs/ 下有多语言 README，说明有成熟的多语言社区协作机制。
- 商业主体：winres 版权声明为 Purslane Tech Pte. Ltd.（`Cargo.toml` 259 行），即官方公司化维护。

## 9. 亮点与风险

**亮点**
1. 全链路自托管与端到端密钥协商（Ed25519 身份 + X25519 密钥封装，`src/common.rs`），中继服务器无法窥探会话。
2. 连接策略工程化程度高：UDP/TCP 打洞、中继、WebRTC、KCP 多通道降级，且对未鉴权阶段的资源有量化预算（PunchPool、answerer 上限），安全设计写在注释里可追溯。
3. 平台覆盖完整：五端统一 Rust 内核，采集/输入/剪贴板全部分平台实现并纳入 workspace 管理。
4. 硬编解 + 软编解自适应协商（VP9/AV1/H264/H265），弱硬件自动回退，协议层携带 CodecAbility。
5. 维护极活跃且工程纪律好：每日提交、SBOM、签名校验、AGENTS.md 规约、超高质量设计注释。

**风险/局限**
1. AGPL-3.0 对商业集成和二次分发有强传染约束，企业内嵌需法务评估。
2. 15+ 个依赖固定为 rustdesk-org 私有 fork 且部分按 rev 锁定，构建可复现性依赖作者维护，供应链面较大。
3. 巨型文件集中复杂度：connection.rs 7876 行、client.rs 5911 行，修改鉴权/会话逻辑的回归风险高。
4. hbb_common submodule 跨仓耦合，客户端改动可能触发服务端仓库联动 PR（`AGENTS.md` 自述），外部贡献者上手成本高。

## 10. 总结评价

RustDesk 是"开源自托管远控"事实上的标杆：协议设计（打洞/中继/加密）扎实且有据可查，五端覆盖与硬编解能力达到商业产品水准，127k 星与每日提交证明了长期健康的社区与商业双轮维护。它最适合的读者是两类人：需要数据不出内网、想在自有服务器上跑远控的团队与个人用户；以及想学习 Rust 网络编程、NAT 穿透、实时音视频编解码与跨平台系统集成的开发者——`src/rendezvous_mediator.rs`、`libs/scrap/src/common/codec.rs` 与 `src/common.rs` 的密钥协商是三处最值得精读的实现。阅读源码前建议先补齐 hbb_common submodule，并对照 `libs/base/protos/message.proto` 建立协议全貌。
