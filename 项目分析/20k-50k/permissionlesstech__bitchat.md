# permissionlesstech/bitchat 项目分析

> 星数 37158（快照 2026-10-24）｜主语言 Swift｜档位 20k-50k｜分析日期 2026-11-17

## 1. 项目速览

一句话定位：bitchat 是一款"无账号、无手机号、无中心服务器"的去中心化聊天应用——近距离用 BLE（蓝牙低功耗）mesh 组网离线通信，远距离经 Nostr 中继（默认走 Tor）触达全球，交互保持 IRC 风格（`/msg`、`/who`、`/slap`）。

| 项目 | 内容 |
|---|---|
| 名称 | permissionlesstech/bitchat |
| 星数 | 37158（gh api 2026-11-17 复核一致） |
| 语言 | Swift（SwiftUI + CoreBluetooth） |
| 许可证 | Unlicense（公有领域，`LICENSE`；注意非 MIT） |
| 创建时间 | 2025-07-04（gh api） |
| 最近活跃 | main 最新提交 2026-10-25（PR #1807），仓库 push 2026-11-11；最新 release v1.8.0（共 21 个） |
| 平台 | iOS 16+ / macOS 13+（本仓）；Android 为姊妹仓 bitchat-android（Kotlin，7999 星） |
| 生态 | 姊妹仓 georelays（Nostr 中继目录，141 星）；App Store / Play Store 双上架 |

## 2. 项目是什么，解决什么问题

bitchat 解决的核心问题是"在基础设施不可用时仍然能通信"。它面向抗议现场、灾区、偏远地区等互联网被切断或不可信的场景：附近设备通过 BLE 广播自发组成 ad-hoc mesh，消息多跳中继（README 与 `WHITEPAPER.md`），完全不依赖任何网络基础设施；一旦任一端有互联网，则通过 Nostr 中继把隔离的 mesh 孤岛桥接起来（`README.md`）。

与 Signal/WhatsApp 不同，它没有任何账号体系：身份即密钥（Curve25519 静态密钥 + Ed25519 签名密钥，存于 Keychain，`WHITEPAPER.md` §3），昵称可随意改，密钥指纹即身份。与 Briar、Bridgefy 同类相比，bitchat 的差异点在"双传输 + 四层存储转发"（§5 详述）：发消息时路由器优先走直连 mesh 链路，不可达则退到 Nostr，再不行则交给"信使（courier）"系统——把密封密文委托给可能物理遇见收件人的路人设备（spray-and-wait 复制预算扩散），实现"收件人当时不在场"的最终送达。

另一个显著特征是刻意保持的 IRC 文化：22 个斜杠命令（`bitchat/Services/CommandProcessor.swift`）包括 `/slap`、`/hug`、`/ping`（测 RTT 与跳数）、`/trace`、`/pay`（Cashu ecash 支付解码，`bitchat/Services/CashuTokenDecoder.swift`）。基于 geohash 的位置频道（`block`/`neighborhood`/`city`/`province`/`region` 五级精度）则让"同城陌生人聊天"成为独立玩法。

需要客观指出：README 明确写道"本仓库曾遭遇下架要求（takedown demands）"（`README.md`），这也是项目强调"从可信渠道获取构建 + 逐 release 哈希清单验证"（`docs/VERIFYING-A-BUILD.md`）的背景。白皮书对自身弱点的披露异常坦诚（§8：元数据是该设计最弱的一环），这种"不过度承诺"的姿态是其口碑的一部分。

## 3. 技术栈与依赖

- **语言/框架**：Swift 5.9（`Package.swift`），SwiftUI 为主（`bitchat/Views/` 下 37 个文件 import SwiftUI，少量 UIKit 25 处，用于图像选择器等）；CoreBluetooth 实现 GATT 双角色。
- **密码学**：
  - 自实现 Noise 协议框架（XX/X 模式，Curve25519 + ChaCha20-Poly1305 + SHA-256），核心 `bitchat/Noise/NoiseProtocol.swift`（1040 行），会话管理 `NoiseSessionManager.swift`（1289 行）；
  - `swift-secp256k1` 0.24.0（P256K，Nostr 的 secp256k1/Schnorr 密钥）；
  - 自实现 XChaCha20-Poly1305 兼容层（CryptoKit 无 XChaCha，`bitchat/Nostr/XChaCha20Poly1305Compat.swift`）。
- **Tor**：进程内 Arti（Tor 官方 Rust 实现）以静态库 xcframework 形式 vendored 于 `localPackages/Arti`，经 SOCKS5 127.0.0.1:39050 暴露，无 tor 二进制/控制端口（`docs/TOR-INTEGRATION.md`）。
- **本地 SPM 包**：`BitFoundation`（共享基础）、`BitLogger`、`Arti`。
- **其他**：LZ4 压缩、GCS 布隆过滤器（gossip 同步）、Cashu ecash 解码、推送即讲（PTT）语音编解码（`bitchat/Features/voice/`）。
- **工程工具**：Justfile、SwiftLint（`.swiftlint.yml`）、Periphery 死代码检测（`.periphery.yml` + baseline）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `bitchat/` | 主应用 281 个 Swift 文件、约 8.1 万行（find/wc 统计） |
| `bitchat/Services/BLE/` | BLE mesh 核心，50+ 个职责单一的文件（链路层、分片、中继策略、peer 注册表等） |
| `bitchat/Noise/`、`Services/NoiseEncryptionService.swift` | Noise 握手/会话/速率限制 |
| `bitchat/Nostr/` | Nostr 客户端（协议、中继管理 1946 行、身份桥、地理中继目录） |
| `bitchat/Protocols/` | 二进制包格式（TLV 编码、分片、文件包、语音突发包） |
| `bitchat/Services/`（其余） | MessageRouter、Courier/（信使）、Gateway/（mesh↔互联网桥）、Prekeys/、Groups/、Board/ |
| `bitchat/ViewModels/`、`bitchat/App/` | SwiftUI 状态层（ChatViewModel 2450 行 + 多个 coordinator） |
| `bitchat/Views/`、`Features/voice/` | SwiftUI 视图、PTT 语音 |
| `bitchatTests/` | 217 个测试文件：EndToEnd/Integration/Simulation/Performance/Noise/Nostr 等 |
| `bitchatShareExtension/` | iOS 分享扩展 |
| `localPackages/` | Arti（Tor）、BitFoundation、BitLogger 三个本地 SPM 包 |
| `docs/` | 15 篇架构文档（BLE-ARCHITECTURE-V3、TOR-INTEGRATION、SOURCE_ROUTING、PEER-ID-ROTATION 等） |
| `relays/` | 442 个 Nostr 中继及 GPS 坐标 CSV |
| `scripts/`、`Configs/`、`.github/workflows/` | 构建/校验脚本、xcconfig、5 个 CI 工作流 |

组织方式评述：单体 Xcode 工程 + 本地 SPM 包混合。架构正处于文档化的"V3 重构"中段——`docs/BLE-ARCHITECTURE-V3.md` 直言历史上 `BLEService` 曾是 8.3k 行"上帝对象"（快照时仍 7187 行，为全仓最大文件），目标是拆成"链路层 → mesh 引擎 → 功能模块 → Transport 边界"的四层包无线电栈。App 层已完成 coordinator 化拆分（ChatViewModel 之下十余个 `Chat*Coordinator`）。

## 5. 核心架构与关键实现

### 5.1 双传输路由与存储转发（`bitchat/Services/MessageRouter.swift`、`Transport.swift`）

`Transport` 协议统一抽象 BLE mesh 与 Nostr 两路传输，`MessageRouter`（@MainActor）按"mesh 优先 → Nostr 兜底 → 信使/outbox"投递。私聊投递四层兜底（`WHITEPAPER.md` §6）：① 发件 outbox（每 peer 100 条、24h TTL、重发上限 8 次，落盘用仅存 Keychain 的 ChaChaPoly 密钥密封）；② 信使系统——密封信封交给最多 3 个已连接 peer，收件地址是"收件人静态密钥 HMAC 当日 UTC 日期"的 16 字节轮换 tag（信使无法得知收发双方），spray-and-wait 复制预算 4→8，在人群中扩散；③ 公共频道 gossip 同步（每 ~15s 用 GCS 过滤器对账，6h 窗口缓存 1000 包，可跨分区"人肉搬运"历史）；④ Nostr 邮箱（重连回看 24h）。路由器对投递失败/被信使接手均有显式回调（`onMessageDropped`/`onMessageCarried`），UI 不会永远停在"发送中"。

### 5.2 BLE mesh 层（`bitchat/Services/BLE/`、`Protocols/BitchatProtocol.swift`）

- **GATT 设计**：设备同时做 GATT central 与 peripheral（`BLERadioController.swift`）；自定义服务 UUID `F47B5E2D-4A9E-4C5A-9B3F-8E1D2C3A4B5C`（mainnet）/`…4B5A`（testnet），特征 UUID `A1B2C3D4-E5F6-4A5B-8C9D-0E1F2A3B4C5D`（`bitchat/Services/BLE/BLEService.swift:187-191`），扫描按服务 UUID 过滤。
- **受控洪泛**：TTL 起点 7；链路度 ≥6 时钳到 5、≤2 时全深度（`WHITEPAPER.md` §4.2）。去重用 LRU seen-set（1000 条、5 分钟）。中继抖动 10–220ms 让重复抑制生效。广播按消息 ID 哈希选取约 log₂(度数) 的链路子集发送（split horizon 排除入站链路）；定向流量（握手/私聊/信封）不裁剪。
- **源路由**：announce 携带最多 10 个直连邻居 ID，形成浅拓扑图（60s 新鲜度）；有双向确认路径则源路由，失败回落洪泛（§4.3，实现在 `MeshTopologyTracker.swift` 等）。
- **分片与填充**：超 MTU 按 ~469B 分片（128 并发重组、30s 超时、1MiB 上限）；仅 Noise 帧填充到 256/512/1024/2048 字节桶，其余类型自然长度——白皮书如实说明"多数流量的负载长度可被观察"。
- **元数据现实**：包头 8 字节 sender ID 是 Noise 静态密钥指纹前 8 字节，**不轮换**（仅 panic wipe 才变），announce 明文携带静态公钥与昵称——白皮书 §8 明确承认"被动监听者可跨时间地点关联设备"，并列为未来工作（`docs/PEER-ID-ROTATION.md` 为轮换设计文档）。

### 5.3 加密体系（`bitchat/Noise/`、`WHITEPAPER.md` §5）

在线会话用 Noise **XX** 模式（互认证 + 前向保密，`NoiseProtocol.swift` 头部文档给出完整握手流程）；离线密封（信使信封）用单向 **X** 模式封到收件人静态密钥——白皮书明确写"此路径无前向保密，静态密钥泄露将暴露未投递的密封邮件，prekey 是未来工作"（`Services/Prekeys/` 目录已存在）。Nostr 私信用自研私有信封：内层 kind 14 未签名消息 → kind 13 Schnorr 签名封印 → kind 1059 一次性密钥签名的公共信封，内容为 `v2:` 前缀 + base64url 的 24B nonce + XChaCha20-Poly1305 密文；密钥来自 secp256k1 ECDH + HKDF-SHA256（复用 "nip44-v2" info 标签但非 NIP-44 密钥排布），外层时间戳随机化 ±15 分钟（§5.3）。README 与 SECURITY.md 反复强调该格式**不兼容 NIP-17/44/59**，仅 BitChat 客户端互通。测试用已发布 iOS（733098bb）与 Android（b7f0b33d）版本生成的冻结信封做跨端接收回归（`Package.swift:70-74` 注释、`bitchatTests/Nostr/Fixtures/`）。

### 5.4 Tor 与审查规避（`docs/TOR-INTEGRATION.md`、`Services/NetworkActivationService.swift`）

全部互联网流量（Nostr 中继 websocket + 地理中继目录拉取）默认经进程内 Arti 走 Tor，且**fail-closed**：Tor 未就绪时请求排队而非回落明网；`torEnforced` 为编译期常量，release 构建强制开启。设置中的"Tor 路由"开关默认开，关闭时 UI 明示"每个中继运营者都能看到你的 IP"。已知缺口同样写入文档：Arti 构建未启用 pluggable transports，无 obfs4/snowflake/meek——在封锁公共 Tor 节点的国家网络下引导无法完成，应用会明确报告卡住而非假装在连。另有无桥接依赖风险：内置中继集仅 4 个域名。mesh 孤岛间桥接另有 `Services/Gateway/BridgeService.swift`：可选开启，用按 geohash 单元派生的不可链接 Schnorr 身份签署"汇合事件"发布到确定性地理中继，带显式同意模型与三条防回环规则。

## 6. 构建与运行

来自 `README.md` 与 `Justfile`：

```bash
# Xcode 打开（真机签名需先配置团队 ID）
open bitchat.xcodeproj
cp Configs/Local.xcconfig.example Configs/Local.xcconfig

# macOS Debug 构建（免签名）
xcodebuild -project bitchat.xcodeproj -scheme "bitchat (macOS)" \
  -configuration Debug CODE_SIGNING_ALLOWED=NO build

# SwiftPM 全量测试 / iOS 模拟器测试
swift test
xcodebuild -project bitchat.xcodeproj -scheme "bitchat (iOS)" \
  -sdk iphonesimulator -destination 'platform=iOS Simulator,name=iPhone 17' test

# just 工作流
brew install just && just check && just run
```

## 7. 测试与工程质量

- **规模**：217 个测试文件、约 220+ 个 `func test`（grep 统计），分层明确：EndToEnd / Integration / Fragmentation / Simulation / Performance / Noise（含官方测试向量 JSON）/ Nostr（冻结信封 fixtures）/ Localization。
- **仿真测试架构**：`bitchatTests/README.md` 描述的内存总线 `MockBLEService`——全局注册表 + 邻接表模拟任意 mesh 拓扑，支持广播洪泛开关、TTL 行为断言、Noise 解密失败后的再握手验证（`IntegrationTests.testRehandshakeAfterDecryptionFailure`）。
- **CI**（`.github/workflows/`）：`swift-tests.yml` 在 macOS 上按 app/BitFoundation/BitLogger 三包矩阵跑，25 分钟硬超时 + 5 分钟测试看门狗 + 性能地板门禁（`scripts/check-perf-floors.sh`、`bitchatTests/Performance/perf-floors.json`）；`periphery.yml` 死代码检测；`arti-provenance.yml` 强制 Tor 二进制来源可追溯（"仅更新二进制不可接受"，`docs/ARTI-BINARY-PROVENANCE.md`）；另有源码清单与地理中继数据校验工作流。
- **观察**：工程卫生水平显著高于同规模社区项目——`Justfile` 有防"clean 恢复/删除源码"的静态守卫脚本；`BLE-ARCHITECTURE-V3.md` 对既往重构失败的复盘（"卫星策略为何停滞"、ABBA 死锁）写得像一篇工程博客；最新提交（PR #1807）把 Android 的 3D 地球仪选点器移植到 iOS 并顺手修了 Android 端同样的反经线渲染 bug，显示双端团队在互相搬运改进。

## 8. 社区与维护状态

- **活跃度**：2025-07-04 创建，约 16 个月达 37158 星；PR 编号已到 #1807，21 个 release（最新 v1.8.0），170 个 open issue，5895 forks（gh api）。main 最近提交 2026-10-25，仓库 push 2026-11-11——截至分析日仍高频活跃。
- **贡献者**：30 人（gh api contributors）。首位 `jackjackbits` 688 次提交，占绝对主导；其后 nothankyou1（128）、qalandarov（97）、callebtc（25，Cashu 协议作者，与 `/pay` 功能呼应）、nadimkobeissi（9）。典型单人主导 + 小圈子协作结构。
- **作者背景**：发布主体为 GitHub 组织 `permissionlesstech`，仓库内不披露真实身份。外界媒体广泛报道该项目与 Jack Dorsey 相关（本人亦曾公开提及），但本次分析未做网络核实，**未确认**；从仓库内证据只能确认"匿名组织 + 高速单人主导开发"。
- **平台生态**：Android 为独立公开仓 `permissionlesstech/bitchat-android`（Kotlin，7999 星，push 2026-11-15），并非未开源；另有 `georelays` 仓维护 442 个中继坐标（`relays/online_relays_gps.csv` 与 CI 定时刷新）。
- **合规压力**：README 自述曾遭下架要求；SECURITY.md 强调用户身处敌意网络环境、漏洞报告走 GitHub 私密通道且"历史上确认的协议/密钥问题数日内修复"，无 bug bounty。

## 9. 亮点与风险

**亮点**
1. 协议白皮书与安全文档异常诚实：主动披露元数据可链接性（静态 peer ID、明文 announce、邻居表、跳数定位）、密封邮件与 Nostr 信封无前向保密、填充仅覆盖 Noise 帧——不是营销文档而是工程契约（`WHITEPAPER.md` §4.1/§8）。
2. 存储转发四层栈是罕见的完整落地：outbox/信使（不可链接轮换 tag + spray-and-wait）/gossip 同步（GCS 过滤器）/Nostr 邮箱，"收件人不在场"有真实答案而非纸面设计（§5.1）。
3. Tor 进程内集成（Arti xcframework）+ fail-closed 默认路由，且二进制有 provenance CI 强制溯源——移动应用里少见的供应链纪律。
4. 测试体系专业：内存 mesh 总线仿真、Noise 官方向量、跨端冻结信封回归、性能地板门禁。
5. 文档驱动重构：BLE V3 拆解计划、并发契约（engine→bleQueue 单向同步序）、失败复盘全部落在 `docs/`。

**风险/局限**
1. 隐私短板客观存在：非 Noise 包长度可观察、mesh 元数据可被动关联（白皮书自认）；对"防交通分析"没有覆盖流量。
2. Nostr 私有信封为专有格式（不兼容 NIP-17/44/59），形成生态孤岛；密封路径无前向保密，prekey 尚未完成（`Services/Prekeys/` 在建）。
3. 审查规避有硬缺口：Arti 未启用 pluggable transports，无桥接/混淆；内置 Nostr 中继仅 4 个域名，易被定向封锁（`docs/TOR-INTEGRATION.md`）。
4. `BLEService.swift` 仍 7187 行，V3 重构未完成；iOS 仓与 Android 仓代码独立演进，协议一致性依赖冻结 fixture 回归而非共享实现。

## 10. 总结评价

bitchat 是 2025 年"去中心化消息"浪潮中工程质量最扎实的作品之一：它把 BLE mesh、Noise 加密、Nostr/Tor、存储转发四层栈完整地塞进一个无服务器的移动应用，并用一份诚实到自我批判的白皮书和一套仿真测试体系托住可信度；16 个月 37.2k 星、双端上架、持续高频迭代证明了其社区吸引力。它不是匿名性承诺者（白皮书明确说"不可链接的在场"尚未实现），而是"无账号、无服务器、审查抗性"的务实实现。适合关注 mesh 组网、BLE 协议设计、Noise 工程化与隐私应用取舍的开发者精读源码（建议从 `WHITEPAPER.md` → `docs/BLE-ARCHITECTURE-V3.md` → `bitchat/Services/MessageRouter.swift` → `bitchat/Services/BLE/` 的顺序进入）；适合需要在断网/受控网络环境通信的用户使用，但不应将其当作对抗强 adversary 的匿名工具。
