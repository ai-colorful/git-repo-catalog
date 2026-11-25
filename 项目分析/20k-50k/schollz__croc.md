# schollz/croc 项目分析

> 星数 41500（快照 2026-10-24）｜主语言 Go｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：任何两台电脑之间"一条命令 + 一个短语"即可完成端到端加密文件传输的跨平台 CLI 工具。

| 事实 | 值 | 出处 |
|---|---|---|
| 名称/模块 | `github.com/schollz/croc/v11`（当前版本 11.6.5） | `go.mod`、`src/version/version.go` |
| 星数 | 40930（gh api 实测 2026-11-14，任务快照 41500） | gh api repos/schollz/croc |
| 主语言 | Go（233 个 .go 文件）+ TypeScript 浏览器客户端 | 源码统计 |
| 许可证 | MIT | `LICENSE` |
| 创建/最近 push | 2017-10-17 / 2026-11-13（前一天仍在提交） | gh api |
| 贡献者 | 154 人（contributors 分页 last=154） | gh api |
| 官网 | https://getcroc.com | `README.md`、gh api homepage |

## 2. 项目是什么，解决什么问题

croc 解决的是"把一个文件从这台电脑传到另一台"这一高频但一直缺乏好工具的场景：双方可能在不同网络、不同操作系统、没有公网 IP、也不想注册任何账号。传统方案要么要搭服务器/配端口转发（scp、rsync），要么依赖第三方网盘（丢失端到端加密），magic-wormhole 虽给出了 PAKE（口令认证密钥协商）思路但依赖 Python 环境不易分发。croc 在 README 致谢中明确说明其灵感来自 @warner 的 magic-wormhole（`README.md` Acknowledgements），但用 Go 实现并编译成**单二进制**，通过 `curl https://getcroc.com | bash` 或 brew/scoop/pacman 等几乎所有主流包管理器安装（`README.md` Install 节）。

工作流程极简：发送方 `croc send file` 得到一个三个英文单词的短语（如 `code-phrase`），接收方在任意另一台机器上执行 `croc code-phrase` 即开始传输。短语用于 PAKE 协商出会话密钥，之后所有数据端到端加密，中继服务器只看到密文（`README.md` Usage 节，实现见 `src/pakekey/pakekey.go`）。

croc 自述是唯一同时满足以下全部条件的 CLI 传输工具（`README.md` About 节）：任意两机互联（P2P 优先、中继兜底）、PAKE 端到端加密、跨平台（含浏览器端 getcroc.com）、多文件、断点续传、无需本地服务器/端口转发、IPv6 优先 IPv4 兜底、可走代理（如 Tor）。

发展至今（v11）项目已远超"传文件"本身：`croc ssh` 可共享共享终端（基于 WireGuard + SSH，`src/sshshare/`）、`croc store` 提供客户端加密的中转存储（`src/storecrypto/`）、`croc relay` 自托管中继、`croc-web` 提供浏览器客户端（`web/`、`cmd/croc-web/`），传输层还集成了 Tailscale 用户态 WireGuard（Tailcat）以获得更强的 NAT 穿透。

## 3. 技术栈与依赖

- **语言**：Go 1.27.3（`go.mod`）为主，浏览器客户端为 TypeScript/React 19 + Vite（`web/package.json`），另有 Go WASM 运行时（`web/sshwasm/`）。
- **核心第三方库**（`go.mod`）：
  - `github.com/schollz/pake/v3`——作者自研的 PAKE 库（SPAKE2 类），端到端密钥协商的核心；
  - `tailscale.com v1.104.6` + `gvisor.dev/gvisor`——内嵌 Tailscale 用户态 WireGuard（Magicsock/DERP）与 netstack，用于 Tailcat 数据传输；
  - `golang.org/x/crypto`——AES-GCM、ChaCha20-Poly1305、argon2、hkdf、pbkdf2（`src/crypt/crypt.go`）；
  - `github.com/tailscale/gliderssh` + `creack/pty`——croc ssh 的 SSH 服务端与伪终端；
  - `github.com/schollz/peerdiscovery`——局域网组播发现，实现同网直连；
  - `github.com/kalafut/imohash`、`xxhash`、`highwayhash`、`murmur3`——可选择的快速哈希（`src/cli/cli.go` --hash 标志）；
  - `github.com/coder/websocket`——croc-web 的 WebSocket 中继桥。
- **前端密码学**：`@noble/curves`、`@noble/hashes` 在浏览器中复刻了与 Go 端 `pake/v3` 完全兼容的线协议，包括作者自定义的 SIEC 曲线（`web/src/protocol/pake.ts`，该文件开头注释"Wire and scalar compatibility with the pinned github.com/schollz/pake/v3"）。
- **CLI 框架**：`internal/cli/` 是项目内嵌（vendored）的极简 CLI 框架（带独立 LICENSE 与 README），非 urfave/cli 等外部依赖。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `main.go`、`internal/cli/` | 入口 + 内嵌 CLI 参数解析框架 |
| `src/cli/` | 子命令编排（send/store/ssh/update/relay）与全局 flags |
| `src/croc/` | **核心**：传输状态机、分块收发、断点续传、传输通道选择（47 个文件，约 1.7 万行） |
| `src/tcp/` | 中继服务器与客户端协议：房间、握手、准入限流、多端口管道 |
| `src/pakekey/` | PAKE 身份绑定与 HKDF 通道密钥推导 |
| `src/crypt/`、`src/compress/`、`src/message/` | AEAD 加密、压缩、JSON 消息编解码 |
| `src/codephrase/` | 短语生成/解析 + EFF 词表 |
| `src/comm/`、`src/utils/` | 连接封装、分块/续传工具函数 |
| `src/tailcattransport/`、`src/croc/tailcat_*.go` | Tailscale WireGuard 数据通道 |
| `src/sshshare/`、`internal/tailcat/`、`internal/sshclient/` | croc ssh 终端共享 |
| `src/store/`、`src/storecrypto/`、`src/storeclient/`、`src/publicrelay/` | 加密中转存储（服务端/格式/客户端/公共实例） |
| `src/receivefs/`、`src/bundle/`、`src/diskusage/` | 接收端文件系统视图、小文件打包、空间检查 |
| `web/`、`cmd/croc-web/` | React 浏览器客户端 + 独立 web 服务器 |
| `packaging/` | 各发行版打包配置（deb/rpm/arch/nix/alpine 等） |
| `src/docs/` | 设计文档：SSH_SHARING.md、STORED_TRANSFERS.md、TESTING.md、各类基准报告 |

组织方式为标准 Go 单仓单模块（v11 module path），按协议层次分层清晰：`message`（线协议）→ `crypt/pakekey`（安全）→ `tcp`（中继）→ `croc`（会话状态机）→ `cli`（用户界面）。`internal/cli` 与 `src/docs` 内嵌设计文档是亮点——协议演进（Tailcat、store、bundle）都有对应的设计/测试文档。

## 5. 核心架构与关键实现

### 5.1 PAKE 密钥协商与端到端加密（src/pakekey/pakekey.go）

croc v11 把 PAKE 交换与参与者身份、会话、用途、线字节 transcript 全部绑定（协议版本 2，fail-closed）。接收方恒为 party A、发送方恒为 party B，身份由 `purpose+room+角色` 域分隔帧（length-prefixed frame）构造（`Identities()`），杜绝未知密钥共享与反射攻击：

```go
// src/pakekey/pakekey.go:51
func Identities(purpose, room string) (idA, idB []byte, err error) {
    ...
    idA = frame([]byte(identityDomain), version, []byte(purpose), []byte(room), []byte("receiver"))
    idB = frame([]byte(identityDomain), version, []byte(purpose), []byte(room), []byte("sender"))
    return idA, idB, nil
}
```

PAKE 会话密钥再经 HKDF-SHA256（32 字节 salt + 完整线 transcript 作为 info）扩展出 32 字节流量密钥与 A/B 双方确认标签，`Confirm()` 用 `hmac.Equal` 恒时比较（`Derive()`/`Confirm()`，`pakekey.go:77-126`）。数据面加密在 `src/crypt/crypt.go`：AES-128-GCM（随机 nonce 前置）、 ChaCha20-Poly1305（argon2 派生密钥），接收循环用 `DecryptAEADInPlace` 原地解密避免分配（`crypt.go:102`，调用点 `src/croc/croc.go:3730` 附近）。

短语本身分两用（`src/codephrase/codephrase.go:194-222` `Parse()`）：三词码的第一词 SHA-256("word"+"croc") 生成中继房间名，其余词作 PAKE 口令——同一个短语既定位房间又认证身份，中继只见到房间哈希与密文。

### 5.2 中继服务器协议（src/tcp/tcp.go，1072 行）

中继是极简的"房间登记 + 字节管道"服务：客户端以 `password+room` 握手（默认口令 `pass123`，`src/models/constants.go:27-32`），同房间的两个连接由 `pipe()` 互转字节，服务器不解析业务消息。关键工程化设计：

- **多端口多路复用**：默认 9009-9013 五个端口，控制消息走第一端口、文件数据分散到其余端口以绕过单连接吞吐限制（`src/cli/cli.go` relay --ports，`src/croc/staged_transport.go` 的 `stagedRelayIndices`/`relayRampIndices` 渐进开通道）。
- **准入限流**：每端口 64 个未完成握手（每源 IP 32）、首消息 60 秒、握手 5 分钟超时，另有每源/每房间 join 速率限制（`src/tcp/admission.go`、`pending_handshakes.go`，README Self-host Relay 节同参数）。
- **公共中继路由**：默认公共池 `croc.schollz.com` 等（`models/constants.go`），v11.6 起按 `SHA-256(code) mod N` 把短语确定性地映射到有序三中继池之一，自动生成的发送方探测三台中继选首个健康者并缓存 `best-relay`（`codephrase.go:145` `RelayIndex()`、`src/publicrelay/publicrelay.go`、README Custom Code Phrase 节）。

自托管即 `croc relay`，Docker 镜像 + `disco.json` 部署公共站点（`Dockerfile`、`disco.json`、README Deployment 节）。

### 5.3 分块传输与断点续传（src/croc/chunk_queue.go、src/utils/utils.go）

接收端按 4 字节边界分块落盘；续传的巧思在 `MissingChunks()`（`src/utils/utils.go:443-512`）：重传时逐块读取本地已有文件，**全零块视为缺失**——因为发送端发送前会把待传块清零占位，所以接收方能以 O(n) 扫描精确算出缺失区间，返回 `[chunkSize, start, count, ...]` 游程编码而非逐块列表：

```go
// src/utils/utils.go:443（节选）
func MissingChunks(fname string, fsize int64, chunkSize int) (chunkRanges []int64) {
    ...
    if allZero(buffer[:bytesread]) {   // 零块 = 未收到
        if runCount == 0 { runStart = currentLocation }
        runCount++
    } else { flushRun() }              // 游程压缩成 (start,count)
```

发送端 `requestedChunkQueue`（`src/croc/chunk_queue.go:28`）按区间惰性发放偏移量，"Workers may join after transfer start without changing ownership of chunks already claimed"——多端口连接可中途加入认领未完成块，配合 `transferWithReconnect()`（`croc.go:810`）的重连退避实现传输中断自动恢复。文件完整性可选 xxhash/imohash/md5/highway 快速哈希 + 精确哈希复核消息（`TypeExactHashRequest`，`src/message/message.go`）。

### 5.4 NAT 穿透与传输通道选择（src/croc/tailcat_negotiation.go、local_relay.go）

通道选择是 v11 的最大架构演进，`--transport auto|derp|relay`（`croc.go:151-171` `ParseTransportMode`）：

1. **局域网直连**：发送方组播（IPv4+IPv6 并行，`discoverReceivePeers()` `croc.go:1973`）发现同网接收方后起**本地中继**（`local_relay.go`，动态预留端口段）实现零公网流量；
2. **Tailcat（Tailscale WireGuard）**：PAKE 握手完成后，双方基于 PAKE 结果派生 Tailcat 节点身份（feature 串 `experimental-tailcat-v1`，`tailcat_negotiation.go:17-23`），进程内起用户态 WireGuard，Magicsock 先经 DERP 中转再升级为直连 UDP——相当于内置了一套 mini-Tailscale，无需账号；
3. **经典中继兜底**：浏览器端/老版本/Tailcat 失败时回落 croc 中继多端口密文通道；auto 模式下仅当传输量超过 `autoTailcatThresholdBytes = 128MB` 才切换 Tailcat（`tailcat_negotiation.go:22`）。

另有 `--socks5` 全程代理支持（中继域名由代理解析，README Use a Proxy 节，`src/croc/socks5_test.go`）与 CVE-2023-43621 的应对——Unix 下推荐 `CROC_SECRET=... croc` 环境变量传短语避免进程列表泄露（README）。

## 6. 构建与运行

```bash
# 构建（Makefile，CGO_ENABLED=0 纯静态）
make build        # croc + croc-update 两个二进制
make build-web    # 先 npm install + 生成 web 产物，再构建 croc-web
go install github.com/schollz/croc/v11@latest   # 源码安装（README，Go >= 1.27.3）

# 测试
go test ./...     # CI 用法见 .github/workflows/ci.yml

# 运行
croc send file            # 发送，得到三词短语
croc <code-phrase>        # 接收
croc relay                # 自托管中继（默认端口 9009-9013）
croc ssh                  # 共享终端
croc store f1 f2          # 加密中转存储
croc-web getcroc.com      # 独立 web 服务器（127.0.0.1:9014）
```

发布流程由 GitHub Actions "Prepare Release" 工作流自动 stamp 版本、打 tag、构建签名全部产物并生成 draft release（`AGENTS.md` "Releasing croc" 节、`.github/workflows/release.yml`）。

## 7. 测试与工程质量

- **测试规模**：101 个 `_test.go` 文件、约 2.5 万行测试代码（源码统计），与约 1.7 万行的核心包 `src/croc` 相比测试密度极高；风格以标准库 `testing` 为主，`stretchr/testify` 辅助（`go.mod`）。
- **CI 分层**（`.github/workflows/ci.yml`）：`unit-tests`、`security-regressions`（安全回归专列）、`security-race-tests`（race detector）、`security-fuzz-smoke`（fuzz 冒烟）、`small-file-bundles`（冻结历史客户端的跨版本兼容测试，用 Python 脚本对真实二进制做双向互通验证），另有 windows-path-check、stale、lock-closed-threads 等运维工作流。
- **协议兼容测试**：`web/testdata/oracle/`、`src/storecrypto/testdata/` 存放跨语言（Go/TS）golden 数据，浏览器与 CLI 的 PAKE 线协议一致性有回归保障（`web/src/protocol/native-interop.test.ts`）。
- **文档/基准文化**：`src/docs/` 内含 CLI/SSH/文件夹传输的 benchmark JSON+MD 与 TESTING.md，工程化程度在同类 CLI 项目中罕见。
- **构建质量**：`-trimpath -tags netgo,osusergo -ldflags="-s -w"` 静态单二进制（`Makefile`），CGO 关闭保证跨平台交叉编译。

## 8. 社区与维护状态

- **活跃度极高**：最近 push 2026-11-13（分析前一天），open issues 仅 3（gh api）。
- **发版节奏**：近 12 个 release 全部在 2026-09-06 至 2026-11-07 两个月内（v11.5.0 → v11.6.5），11 月第一周连发 5 个 patch（gh api releases），节奏远超常规——单维护者高强度迭代 + 自动化发布流水线。
- **贡献者结构**：154 名贡献者（gh api 分页推算），但提交高度集中于作者 schollz 本人（未逐条核实占比，标注"未确认"）；项目通过 GitHub Sponsors 请求赞助（`README.md` 顶部横幅、`.github/FUNDING.yml`）。
- **生态分发**：进入 Homebrew/Chocolatey/Scoop/pacman/conda-forge/Termux/Nix 等几乎所有主流包管理器（`README.md` Install 节、`packaging/distributions/`），并有第三方桌面应用 CrocApp 上架 App Store（README Desktop 节）。

## 9. 亮点与风险

**亮点**
1. 单二进制 + 三词短语做到"零配置、端到端加密"的文件传输，UX 与安全模型都是同类标杆（`README.md`、`src/codephrase/`）。
2. PAKE 密钥与身份/用途/完整线 transcript 域分离绑定（protocol v2，fail-closed），并有安全回归、race、fuzz 专列 CI（`src/pakekey/pakekey.go`、`.github/workflows/ci.yml`）。
3. 传输通道三级择优：局域网组播直连 → 内嵌 Tailscale WireGuard（DERP 兜底+UDP 升级）→ 多端口中继，覆盖各种网络环境（`src/croc/tailcat_negotiation.go`、`local_relay.go`）。
4. 基于全零块扫描的轻量断点续传与多连接块认领队列，实现优雅（`src/utils/utils.go:443`、`src/croc/chunk_queue.go`）。
5. Go/TypeScript 双端协议复刻 + golden 数据互操作测试，浏览器与 CLI 真正互通（`web/src/protocol/pake.ts`）。

**风险/局限**
1. 迭代极快（两月 12 个 release），协议特性（Tailcat 实验 feature、three-word 码路由）持续变动，第三方实现与旧客户端兼容性承压（`tailcat_negotiation.go` 的 `experimental-tailcat-v1`）。
2. 依赖树被 Tailscale/gVisor 显著放大（`go.mod` 直接+间接依赖近百项），二进制体积与审计面扩大，与"极简单二进制"的初始定位有张力。
3. 公共中继与公共存储依赖作者自维护基础设施（croc.schollz.com、getcroc.com），赞助模式能否长期支撑未确认（`README.md` 赞助横幅）。
4. 短语熵有限：三词 EFF 码约 30.7 bit，PAKE 抗在线猜测但房间名可被枚举碰撞，安全边界依赖中继限流设计（`codephrase.go`、`src/tcp/admission.go`；SSH 码已增至六词以补熵，`codephrase.go:59-65` 注释）。

## 10. 总结评价

croc 是"把复杂留给工程、把简单留给用户"的典范：用户侧只需一个三词短语，工程侧却完整实现了 PAKE 身份绑定、三级传输通道择优、断点续传、准入限流与跨语言协议互通。它从 magic-wormhole 的思想出发，用 Go 单二进制超越了原型，并在 v11 阶段通过内嵌 Tailscale 用户态 WireGuard 和浏览器客户端把"任意两台设备"做到字面意义成立。高度活跃的维护、密集的自动化测试与设计文档使其源码质量位居 Go CLI 项目前列。适合所有需要跨网络传文件的用户直接使用；对学习 PAKE 工程化、中继协议设计、NAT 穿透策略与 Go 单仓分层的开发者，`src/pakekey`、`src/tcp`、`src/croc/tailcat_*.go` 与 `src/docs/` 都是极好的读本。
