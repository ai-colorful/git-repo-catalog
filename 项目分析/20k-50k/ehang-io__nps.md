# ehang-io/nps 项目分析

> 星数 30000（快照 2026-10-24，核实日实时 34256）｜主语言 Go｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：一款用 Go 编写的轻量级内网穿透代理服务器，服务端（nps）+ 客户端（npc）双端架构，支持 TCP/UDP/HTTP/SOCKS5/P2P 等多种隧道模式，并内置 beego Web 管理面板。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | ehang-io/nps（原名 cnlh/nps，GitHub 重定向证实） | gh api repos/cnlh/nps → full_name=ehang-io/nps |
| 星数 | 34256（2026-11-21 实时）；forks 6060 | gh api |
| 语言 | Go（62 个 .go 文件） | 源码统计 |
| 许可证 | GPL-3.0 | gh api + LICENSE |
| 创建时间 | 2018-11-04 | gh api |
| master 最后 commit | 2021-10-09（ab648d6） | gh api commits |
| 仓库最后 push | 2024-05-30（核心开发停滞于 2021） | gh api |
| 最新 release | v0.26.10（2021-04-08） | gh api releases + build.sh |
| archived | 否 | gh api |

## 2. 项目是什么，解决什么问题

nps 解决"无公网 IP 的内网机器如何被外部访问"的问题。README_zh.md 列举的典型场景包括：微信开发调试需要外网回调域名（域名代理模式）、在外网 SSH/远程桌面连接内网机器（TCP 代理）、外网访问内网 UDP 服务如 DNS（UDP 代理）、外网用 HTTP 代理访问内网站点（httpProxy 模式）、以及把内网当作 VPN 使用（socks5/P2P 模式）。与同类项目 frp 相比，nps 的差异化定位是"自带 Web 管理面板的多用户代理服务器"——穿透规则在 Web 界面上按客户端逐条配置，而不是手写配置文件（README_zh.md"操作简单，只需简单的配置即可在 web ui 上完成其余操作"）。

整体上它是一个 C/S 双端系统：服务端 nps 部署在有公网 IP 的机器上，负责暴露各隧道模式的监听端口；客户端 npc 部署在内网机器上，主动向服务端建立桥接连接。外部访问服务端端口时，服务端通过桥接连接把目标地址指令下发给客户端，由客户端在本地连接真正的内网目标并做双向转发（server/proxy/base.go 的 `DealClient` → bridge/bridge.go 的 `SendLinkInfo` → client/client.go 的 `handleChan`）。

功能面覆盖很广：隧道模式有 tcp、udp、httpProxy（内网 HTTP 代理）、tcpTrans（透明转发）、socks5、secret（加密私密代理）、p2p（点对点打洞）、file（文件访问）、httpHostServer（域名解析模式，服务端按 Host/SNI 分流到不同客户端），由 server/server.go 的 `NewMode` 工厂统一注册（server/server.go:126-157）。扩展能力包括流量/带宽/连接数限制、snappy 压缩、AES/TLS 加密、健康检查摘除后端、端口复用、多用户注册等（server/proxy/base.go `CheckFlowAndConnNum`、lib/rate、lib/cache、bridge/bridge.go `GetHealthFromClient`）。

客户端支持三种接入形态：命令行（cmd/npc）、配置文件模式（client/control.go `StartFromFile`，客户端在桥接通道上向服务端注册 host/task）、以及 C SDK 与 Android GUI（cmd/npc/sdk.go 产出 c-shared so/dll、gui/npc 用 fyne）。本文仅作架构与实现层面的技术分析，不涉及使用指导。

## 3. 技术栈与依赖

- 语言与最低版本：Go 1.15（go.mod `go 1.15`）。
- Web 框架：beego v1.12.0，但通过 `replace github.com/astaxie/beego => github.com/exfly/beego v1.12.0-export-init` 换成了 fork 版（go.mod），目的是导出 `InitBeforeHTTPRun` 以便把 beego 挂到自建 listener 上（server/proxy/tcp.go `WebServer.Start`）。
- 多路复用：`ehang.io/nps-mux`——作者自研的独立库，桥接通道的单 TCP/KCP 连接上复用出任意条逻辑流（bridge/bridge.go、client/client.go 均以 `nps_mux.NewMux` 使用）。
- KCP：`xtaci/kcp-go v5.4.20`——桥接可跑在 KCP（`bridge_type=kcp`，bridge/bridge.go `StartTunnel`）上，P2P 数据通道也用 KCP over UDP（client/client.go `newUdpConn`）。
- STUN 打洞：`ccding/go-stun`（client 端 handleP2PUdp 配合 server/proxy/p2p.go 的地址交换）。
- 协程池：`panjf2000/ants/v2`——双向拷贝任务进池（lib/conn/conn.go `CopyWaitGroup` → lib/goroutine `CopyConnsPool`）。
- 系统信息：`shirou/gopsutil/v3`（server/server.go `GetDashboardData` 的 CPU/内存/网络面板数据）。
- 系统服务：`kardianos/service`——install/start/stop 注册为 systemd/sysv/Windows 服务（cmd/nps/nps.go）。
- 压缩：`golang/snappy`（lib/conn/snappy.go）；解包 `c4milo/unpackit`（自更新场景）。
- 前端：Go template（web/views/）+ 原生 js/css（web/static/），无现代前端框架与 node 构建链。
- GUI：`fyne.io/fyne/v2`（gui/npc，Android APK）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| cmd/nps、cmd/npc | 服务端与客户端的 main 入口，含 daemon/service 化逻辑 |
| server/ | 服务端核心：server.go（任务生命周期）、proxy/（8 种隧道模式实现）、connection/（监听器与端口复用）、tool/（端口探测等） |
| bridge/ | 桥接层，仅 bridge.go 一个文件（536 行）：客户端连接握手、认证、工作模式分发 |
| client/ | npc 客户端：client.go（TRPClient 主循环）、control.go（配置模式接入）、local.go（本地 socks5/文件服务）、health.go（健康检查）、register.go |
| lib/ | 通用库：conn（协议封装）、crypt（TLS/AES/ClientHello 解析）、file（JSON 数据库）、pmux（端口复用）、rate（限速）、cache、config、daemon、install、goroutine（ants 封装）、sheap、version |
| web/ | 管理面板：controllers/（index/login/client/auth）、routers/、views/（Go template）、static/ |
| conf/ | 默认配置样例：nps.conf、npc.conf、multi_account.conf、clients.json/tasks.json/hosts.json、server.key/pem |
| gui/npc | fyne Android 客户端 |
| docs/ | docsify 文档站源文件 |
| build.sh / build.android.sh | 30+ 平台交叉编译发布脚本 |

组织方式是经典的单仓多二进制 Go 项目（一个 module `ehang.io/nps`），分层清晰但耦合通过全局单例发生：`file.GetDb()`（lib/file/db.go sync.Once 单例 JSON 数据库）、`server.Bridge`（server/server.go 全局变量）被 bridge、proxy、web 各层直接引用，属"紧凑的单体"风格。多路复用这一最核心的传输组件被拆到独立仓库 nps-mux，本仓库不包含其实现。

## 5. 核心架构与关键实现

### 5.1 桥接层：一条连接上的认证与模式分发（bridge/bridge.go）

npc 向 nps 的桥接端口（默认 8024）发起的每条连接都经过 `cliProcess` 握手：先读 3 字节测试标志，再互校版本号（客户端版本不匹配即断开），服务端回写 `md5(version)`，随后客户端提交 32 字节 vKey，服务端 `file.GetDb().GetIdByVerifyKey` 验证后进入 `ReadFlag` 分发（bridge/bridge.go:166-210）。七种工作模式用 4 字节标志区分：

- `WORK_MAIN`：信号连接，承载服务端→客户端的指令（如 P2P 打洞通知 `NEW_UDP_CONN`），设置 TCP keepalive 5 秒（bridge/bridge.go:231-251）；
- `WORK_CHAN`：数据隧道连接，包装成 `nps_mux.NewMux`，之后服务端每要一条到内网的连接就 `tunnel.NewConn()` 复用出一条逻辑流（bridge/bridge.go:252-256、bridge/bridge.go:314-359）；
- `WORK_CONFIG`：无 Web 界面的配置接入通道，客户端可以在此通道上注册新客户端、下发 host/task（bridge/bridge.go:391-536）；
- `WORK_SECRET`：私密代理的密钥配对（SecretChan 交给 server/server.go `DealBridgeTask` 处理）；`WORK_FILE`：文件模式的独立 mux；`WORK_REGISTER`：IP 注册（配合 ip_limit）；`WORK_P2P`：打洞握手。

一个客户端身份同时维持 signal 与 tunnel（以及可选 file）三条物理连接，由 `bridge.Client` 结构持有；`ping` 协程每 5 秒巡检，tunnel/signal 缺失连续 3 次即剔除客户端（bridge/bridge.go:361-388）。

### 5.2 服务端隧道模式矩阵与统一抽象（server/）

统一抽象是 `proxy.Service` 接口（`Start/Close`）加函数指针注入：TCP 类模式共用 `TunnelModeServer`，差异只在 `process` 回调——tcp 用 `ProcessTunnel`、httpProxy 用 `ProcessHttp`、tcpTrans 用 `HandleTrans`（server/proxy/base.go:16-42、server/proxy/tcp.go:14-33、server/server.go:126-157）。隧道模式目录清单：

| 文件 | 模式 | 实现要点 |
|---|---|---|
| proxy/tcp.go | tcp / file / webServer | 监听端口，`CheckFlowAndConnNum` 后走 process；WebServer 直接把 beego BeeApp 挂到 listener |
| proxy/udp.go | udp | `ReadFromUDP` 收包，按来源地址 `addrMap` 维护客户端侧会话 |
| proxy/socks5.go | socks5 | 完整 SOCKS5：CONNECT/BIND/UDP ASSOCIATE 与用户名密码认证（`Auth`） |
| proxy/http.go | httpProxy + 域名模式 | host 解析、可选响应缓存（lib/cache）、404/error 页 |
| proxy/https.go | https 域名分流 | 依据 lib/crypt/clientHello.go 手写的 TLS ClientHello 解析取 SNI，按证书路由（多证书站点保护） |
| proxy/p2p.go | p2p | 纯 UDP 打洞协调器：按密码记录 visitor/provider 两侧 UDP 地址并互换（proxy/p2p.go:47-70），后续流量不再经过服务端 |
| proxy/transport.go | tcpTrans | 透明转发辅助 |

所有模式共享 `BaseServer`：流量统计（FlowAdd）、HTTP Basic 认证、客户端流量/连接数配额检查、以及统一的 `DealClient`——构造 `conn.Link`（含目标地址、是否加密/压缩）发给客户端，然后 `CopyWaitGroup` 双向泵数据（server/proxy/base.go:87-100）。secret 与 p2p 模式不占监听端口，仅在 RunList 里占位（server/server.go:186-190）。

### 5.3 传输管线：协议封装、加密压缩与多路复用（lib/conn、lib/crypt）

应用层自定义二进制协议在 lib/conn/conn.go：控制消息统一为"4 字节小端长度 + 内容"（`GetLenBytes`/`GetShortContent`），对象用 JSON 序列化后走 `SendInfo`（conn.go:231-254），flag 为 4 字节常量（common 包 WORK_* 系列）。数据面由 `CopyWaitGroup` 处理：先用 `GetConn` 按 link 参数层层包装连接——crypt=true 包 TLS，snappy=true 包 SnappyConn，再统一包限速 `rate.NewRateConn`——然后交给 ants 协程池做双向拷贝（conn.go:376-417）。

加密的组织方式（仅描述机制）：桥接数据面的 TLS 使用服务端**启动时即时自签**的 10 年期证书，客户端配置 `InsecureSkipVerify: true`（lib/crypt/tls.go:23-49），即密钥交换由标准 TLS 完成但不校验身份；另有 AES-CBC/PKCS5 实现用于 Web API 的 auth_key 加密（lib/crypt/crypt.go:15-40，IV 取密钥前 blockSize 字节）。桥接控制面的认证是明文 vKey 长度比较（bridge/bridge.go:191-203），无挑战-响应或密钥协商。lib/crypt/clientHello.go 是不依赖 crypto/tls 的 ClientHello 手工解析器，专用于 HTTPS 域名分流取 SNI（约 130 行，被 proxy/https.go 引用）。

多路复用依赖外部库 nps-mux：桥接的 WORK_CHAN 连接被 `nps_mux.NewMux` 包装，服务端 `SendLinkInfo` 里 `tunnel.NewConn()` 按需开逻辑流并下发 Link 指令，客户端 mux `Accept` 后拨号内网目标（bridge/bridge.go:320-353、client/client.go:155-179）。KCP 参数调优集中在 `conn.SetUdpSession`（窗口 1024、MTU 1600、nodelay，conn.go:364-373）。

端口复用是另一个亮点：lib/pmux 在 bridge_port 与 http/https/web 端口冲突时启用 `PortMux`，按流量特征把单端口拆成 client/http/https/manager 四个 listener（server/connection/connection.go:19-73）。

### 5.4 客户端主循环与配置接入（client/）

`TRPClient.Start` 建立 signal（WORK_MAIN）后异步建 tunnel（WORK_CHAN），随后 `handleMain` 阻塞读服务端指令（目前只处理 P2P 的 NEW_UDP_CONN）（client/client.go:54-119）。`handleChan` 收到 Link 后 `net.DialTimeout(lk.ConnType, lk.Host)` 连内网目标，http 类型连接会先读出请求打日志再转发（client/client.go:173-219）。断线重连逻辑是简单的外层 `goto retry` + 5 秒退避（client/client.go:56-66、control.go:99-113）。

配置文件模式下（npc.conf），客户端通过 WORK_CONFIG 通道把 hosts/tasks 列表以 JSON 注册到服务端（client/control.go `StartFromFile` → `SendInfo(v, common.NEW_HOST/NEW_TASK)`），服务端对应实现即 bridge/bridge.go `getConfig`。P2P 侧 `handleP2PUdp` 用 go-stun 探测 NAT 后以 KCP 建立点对点通道（client/client.go:121-152）。健康检查（client/health.go + bridge/bridge.go `GetHealthFromClient`）由客户端定时探测 target，把失活后端从服务端 targetArr 摘除。

### 5.5 管理面板与数据层（web/、lib/file）

Web 后端是 beego v1.12：router.go 注册 Index/Login/Client/Auth 四个 AutoRouter controller（web/routers/router.go），支持 web_base_url 命名空间前缀。认证两层：Web 会话 session，API 用 `md5(auth_key + timestamp)` 且时间戳窗口 ±20 秒（web/controllers/base.go `Prepare`），另有 AES-CBC 生成对外 API key 的接口（web/controllers/auth.go）。视图层 web/views/（index/client/login/public 四组模板）+ web/static/ 原生 js/css，仪表盘数据来自 server/server.go `GetDashboardData`（gopsutil 采样）。

持久化是自研 JSON 文件数据库：lib/file/db.go 用 sync.Map 内存态保存 Client/Tunnel/Host，`flowSession` 定时器按 flow_store_interval 周期落盘（server/server.go:449-460），无事务无 WAL，属嵌入式单文件方案。配置本体是 beego ini 格式的 conf/nps.conf（cmd/nps/nps.go 启动时 `beego.LoadAppConfig("ini", ...)`），运行期改动主要走 Web 面板而非文件热更新；npc 端 conf/npc.conf 由 lib/config 解析（含 config_test.go 单测）。

## 6. 构建与运行

来自 Makefile 与发布脚本（静态摘录，未执行）：

- 本地构建：`make build`（等价 `go build cmd/nps/nps.go && go build cmd/npc/npc.go`，Makefile:12-15）。
- 测试：`make test`（`go test -failfast -race -coverpkg=./... ./...`，Makefile:22-25）。
- 发布产物：build.sh 交叉编译 linux/freebsd/darwin/windows × 386/amd64/arm v5-v7/mips 系列 30+ 组合，另有 `-buildmode=c-shared` 产出 npc_sdk.so/.dll、UPX 压缩；build.android.sh 经 fyne-cross 出 Android APK（build.sh:1-50）。
- CI：Travis CI（.travis.yml，tag 触发 build.sh 后 deploy 32 个文件到 releases）；GitHub Actions release.yml 在 release published 时跑 4 个 job——build_assets（同上产物）、build_android、build_spk（群晖套件，克隆 cnlh/spksrc）、build_docker（多架构镜像推送 DockerHub）（.github/workflows/release.yml）。
- Docker：Dockerfile.nps / Dockerfile.npc。

## 7. 测试与工程质量

- 单元测试仅 2 个文件：lib/config/config_test.go、lib/pmux/pmux_test.go；server/test/test.go 是端口探测辅助代码而非测试。核心的桥接/转发/加密路径无测试覆盖。Travis 的 CI 命令也只跑 `go test -v ./cmd/nps/`（.travis.yml:7）。
- 无 lint 强制门禁（Makefile `lint` 目标存在且自注 "TODO: fix tests and lll issues"，未接入 CI）。
- 代码风格：单文件偏大（bridge.go 536 行、socks5.go 393 行）、全局单例（GetDb、server.Bridge）、大量被注释掉的旧代码留存（lib/conn/conn.go:378-396、server/server.go:161 等），整体是"快速演进的功能型"代码质量。
- 依赖健康度：beego v1.12.0、gopsutil v3.21.3 等均为 2021 前版本，依赖未再升级；2022 年起只有 3 条 dependabot 分支（golang.org/x/crypto、x/net、x/sys，因 CVE 而开）未被合入（gh api branches）。

## 8. 社区与维护状态

- 活跃度：master 最后 commit 2021-10-09（ab648d6，"Merge pull request #866"），dev 分支止于 2021-04-07，v2.0-dev 分支止于 2022-01-23（仅两个 commit 的重构尝试）。仓库 pushed_at 为 2024-05-30，与 master 最后 commit 相差近三年，判断为分支级/元数据操作，核心开发实际停滞于 2021 年（gh api commits/branches）。
- Release 节奏：共 10 个 tag，v0.26.6→v0.26.10 集中在 2020-2021 年，最后一版 v0.26.10（2021-04-08）与 build.sh 中 `VERSION=0.26.10` 一致。
- 规模与贡献者：master 约 700 个 commit；contributor 名单约 35 人（含匿名），前两名 ffdfgdfg（226 次）与原作者 cnlh（156 次，即 commit author "he liu"），其余均为个位数贡献（gh api contributors）。
- 待处理积压：open issue 503、open PR 21（gh api search/issues），停更后无人消化。
- 仓库更名史：README 徽章与 docs 内仍引用 cnlh/nps（docs/README.md:2、travis-ci.org/cnlh/nps 徽章），`gh api repos/cnlh/nps` 现重定向到 ehang-io/nps，证明仓库由 cnlh 组织迁至 ehang-io 并更名。社区流传的作者删仓后恢复事件无法通过 GitHub API 直接核实（删除重建会改变仓库 ID，而当前 ID 156084530 创建于 2018-11-04 且旧名重定向有效），本文仅记录可考证部分，不作推断。
- 社区分叉：停更后出现多个接力项目，其中 yisier/nps 自述"基于 NPS 0.26.10 版本二开而来，NPS 接力项目"，3413 星且 2026-11 仍持续推送（gh api repos/yisier/nps）。

## 9. 亮点与风险

亮点：

1. 隧道类型矩阵在同体量项目中几乎最全（tcp/udp/httpProxy/socks5/secret/p2p/file/域名模式），且通过 Service 接口 + process 函数指针保持每类实现独立文件、结构可读（server/proxy/）。
2. 单端口多路复用体系完整：自研 nps-mux 承载逻辑流，KCP 可选桥接，lib/pmux 还能让 bridge/http/https/web 共用一个端口（server/connection/connection.go）。
3. Web 管理面板开箱即用，仪表盘指标（流量/CPU/连接数/各模式计数）数据面完整（server/server.go GetDashboardData）。
4. 工程发布成熟：30+ 平台交叉编译、Android APK、群晖 spk、Docker 多架构镜像、C SDK 全部脚本化（build.sh、release.yml）。
5. 运维细节考虑多：健康检查自动摘除后端、流量/带宽/连接数配额、断线自动重连、系统服务注册。

风险与局限：

1. 项目实质停更：master 停在 2021-10，release 停在 2021-04，503 个 open issue 无人处理；依赖（beego v1.12 等）存在未修复的历史 CVE 面（dependabot 分支未合并为佐证）。
2. 认证与加密强度有限（客观技术描述）：桥接认证为明文 vKey 比对无挑战-响应（bridge/bridge.go:191-203）；数据面 TLS 客户端固定 `InsecureSkipVerify: true` 且证书为启动时自签（lib/crypt/tls.go:44-48）；AES-CBC 的 IV 取密钥前缀（lib/crypt/crypt.go:22）；Web API 时间戳窗口 ±20 秒的 md5 方案（web/controllers/base.go:25-27）。
3. 默认配置弱口令：conf/nps.conf 默认 admin/123、public_vkey=123，历史版本曾被大量以默认配置暴露公网（未在本仓库内考证，为社区普遍报告的现象；README_zh.md 亦有"正式使用一定要更改"提示）。
4. 用途合规性：内网穿透工具本身中性，但同类工具在现实中常被用于绕过网络边界管控、将不可达服务暴露公网等场景；使用者需自行确认符合所在网络环境与法律法规，本项目 GPL-3.0 无任何合规担保（LICENSE）。

## 10. 总结评价

nps 是 2018-2021 年间 Go 生态中功能最全的内网穿透方案之一，其价值在于把"多隧道模式 + Web 多用户管理 + 单端口多路复用"打包成了开箱即用的服务端；代码组织清晰、发布工程完善，适合阅读 server/proxy 的模式矩阵、bridge 的握手分发与 nps-mux 的复用设计来学习穿透类工具的典型分层。但项目自 2021 年起核心停更（2024-05 的 push 未带来代码演进），认证强度与依赖新旧程度按今日标准均需谨慎评估，生产场景社区普遍转向 yisier/nps 等接力分叉或其它替代品。作为"架构教科书 + 历史项目"它的信息量仍然很高，作为活跃依赖则不合适。
