# MetaCubeX/mihomo 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Go（代理核心所在分支；仓库默认分支现状见 §8）｜档位 20k-50k｜分析日期 2026-11-21

**分析口径说明**：本仓库当前默认分支 `main` 已被替换为一个第三方 Python 封装库，Go 代理核心实际位于 `Meta` / `Alpha` 分支。本文所有源码结论均基于 `Meta` 分支（稳定版分支，tarball 1252 个条目，下称"核心源码"），引用路径均为该分支内路径。仓库异常的客观记录见 §8。

## 1. 项目速览

一句话定位：clash 原版删库后由 MetaCubeX 延续的规则分流代理核心（原 Clash.Meta，2023 年更名 mihomo），Go 语言单二进制实现，靠 RESTful API 与 GUI 生态协同。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | MetaCubeX/mihomo（旧名 MetaCubeX/Clash.Meta，旧名 API 请求 301 重定向回本仓库，实测 2026-11-21） | gh api |
| 代理核心语言 | Go（module `github.com/metacubex/mihomo`，go 1.20 起，986 个 .go 文件、约 17.1 万行，实测统计） | `go.mod`、tarball 统计 |
| 许可证 | GPL-3.0（核心源码 `LICENSE`） | `LICENSE` |
| 最近活跃 | v1.19.34 发布于 2026-11-11；Alpha 分支 2026-11-19 有 push | gh api releases/events |
| 官网/文档 | https://wiki.metacubex.one | `README.md`、gh api homepage |
| 分支模型 | Meta=稳定发布 / Alpha=开发主线（`Makefile` 按 `git branch --show-current` 区分版本号） | `Makefile`、`constant/version.go` |

## 2. 项目是什么，解决什么问题

2023 年 11 月 clash 原版（Dreamacro/clash）删库后，clash 配置生态需要一个持续维护的内核。MetaCubeX 此前维护的 Clash.Meta 更名 mihomo 并接续开发（`constant/version.go` 中 `Meta = true`、`MihomoName = "mihomo"` 留有更名痕迹）。它继承 clash 的核心形态：本地起 HTTP/SOCKS/mixed/透明代理/TUN 入站，对每个连接按规则表决定走哪个出站节点，从而把"流量分流"从应用层下沉到系统代理/网关层（`README.md` Features 节）。

在此基础上，mihomo 相对原版 clash 的主要扩展点（均可在源码定位）：出站协议大幅扩充（`constant/adapters.go` 枚举 24 种协议类型 + 新增 anytls/mieru/masque/sudoku/trusttunnel/shadowquic/gost-relay 等）；规则引擎增加逻辑规则（AND/OR/NOT/SUB-RULE）与二进制 rule-set（`rules/parser.go`、`rules/provider/mrs_*.go`）；DNS 子系统覆盖 DoH/DoT/DoQ/DHCP，并保留 clash 独有的 fakeip 模式（`dns/`、`component/fakeip/`）；同时它把 clash 的 RESTful API 完整保留并扩展（`hub/route/server.go`），成为整个 GUI 生态的事实接口标准。

另一个重要特征是"双端"能力：`listener/` 下有 31 个入站子目录，除代理入口外还包含 trojan/vmess/vless/shadowsocks/tuic/hysteria2 等协议的服务端监听实现，即 mihomo 不仅能作为客户端核心，也能直接充当代理服务端（`listener/` 目录列表）。

它不解决 GUI 问题：核心进程无界面，全部状态读写走 external-controller API（官方仪表盘 metacubexd、Clash Verge Rev、ClashMetaForAndroid 等均以此衔接，见 §8）。

## 3. 技术栈与依赖

- 语言：Go（最低 1.20，CI 实测到 1.26，`.github/workflows/test.yml`）。构建标签 `with_gvisor` 启用 gVisor TUN 网络栈（`Makefile`、`README.md`）。
- 依赖结构最显著的特点是"自维护 fork 群"：`go.mod` 中 40+ 个直接依赖指向 `github.com/metacubex/*`，包括 sing 全家桶（sing、sing-mux、sing-quic、sing-shadowsocks、sing-shadowsocks2、sing-tun、sing-vmess、sing-wireguard）、wireguard-go、quic-go、utls、gvisor、tailscale、zerotier-go、amneziawg-go、wazero、bbolt、bart（路由表）等——即 mihomo 与 sing-box 共享同一批底层库的血缘（MetaCubeX 维护自己的 fork 版本）（`go.mod`）。
- 其他关键依赖：`metacubex/chi`（RESTful API 路由，fork 自 go-chi）、`logrus`（日志）、`go.yaml.in/yaml/v3`（配置）、`oschwald/maxminddb-golang`（GeoIP mmdb）、`miekg/dns`（DNS 报文）、`dlclark/regexp2`（规则正则）、`enfein/mieru/v3`、easytier-go（`go.mod`）。
- 无 CGO 默认构建（`Makefile` 中 `CGO_ENABLED=0`），跨平台目标覆盖 darwin/linux/windows/freebsd/android/loong64 等 30+ 组合，amd64 按 GOAMD64 v1-v3 分档（`Makefile`）。

## 4. 目录结构与代码组织

| 目录 | 职责（均经源码核实） |
|---|---|
| `constant/` | 全项目接口与常量层：Proxy/Rule/Inbound/DNS 抽象接口、Metadata、AdapterType 枚举、feature 编译开关 |
| `adapter/` | 出站/入站协议适配：`outbound/`（40 文件协议实现）、`outboundgroup/`（selector/fallback/urltest/loadbalance）、`provider/`（订阅源与健康检查）、`inbound/`（原生入站） |
| `listener/` | 入站监听层，31 个子目录（http/socks/mixed/redir/tproxy/sing_tun + 各协议服务端） |
| `tunnel/` | 分发引擎核心：规则匹配、UDP worker 池、统计（`tunnel/tunnel.go` 760 行） |
| `rules/` | 规则引擎：`common/` 20 种基础规则、`logic/` 逻辑组合、`provider/` rule-set 与 .mrs 二进制格式 |
| `dns/` | DNS 子系统：resolver、doh/dot/doq/dhcp/system 上游、fakeip/mapping enhancer、policy 分流、本地 DNS server |
| `transport/` | 协议传输细节实现，28 个子目录（vmess/vless/trojan/ss/tuic/hysteria/kcptun/simple-obfs/v2ray-plugin/xhttp 等） |
| `hub/` | 对外枢纽：`route/` RESTful API、`executor/` 配置热加载应用 |
| `component/` | 基础组件：fakeip、trie（域名匹配）、geodata、sniffer（TLS/HTTP/QUIC 嗅探）、dialer、resolver、process、mmdb、updater 等 36 个 |
| `config/` | YAML 配置解析（`config/config.go` 1982 行，单体集中） |
| `common/`、`context/`、`log/`、`ntp/` | 工具与横切层 |
| `test/` | 独立 go.mod 的集成测试模块（docker、各协议 e2e） |
| `.github/` | build/test CI、CMFA 联动工作流、systemd/fpm 打包物料 |

组织方式为典型的 Go 单体多包：一个 module、两层依赖方向（constant 为接口核心，adapter/listener/dns 依赖它，tunnel 编排它们，hub 对外）。没有 monorepo/子模块划分，但通过 `constant/features/` 的编译标签开关（cmfa/low_memory/no_tailscale/with_gvisor 等）支撑 Android 端（CMFA）与精简构建复用同一套源码（`constant/features/` 目录）。

## 5. 核心架构与关键实现

### 5.1 分发引擎（`tunnel/tunnel.go`）

TCP 由各 listener 直接推入 `HandleTCPConn`；UDP 采用固定 worker 池：按 GOMAXPROCS 启动多个 `processUDP`，对包 Key 做 hash 分片选队列以保持同一会话的串行性，队列满则直接丢弃（容量 64，`queueCapacity`）。规则匹配主循环 `match()` 为线性顺序匹配，匹配到代理后沿 `Unwrap` 链穿透多层包装（嵌套组），遇到 Pass/Rematch 类型有特殊语义——Rematch 会触发一次"假拨号"更新元数据后重走规则表，并有环检测：

```go
GetRules:
    for _, rule := range getRules(metadata) {
        if matched, ada := rule.Match(metadata, helper); matched {
            adapter, ok := proxies[ada]
            ...
            for adapter := adapter; adapter != nil; adapter = adapter.Unwrap(metadata, false) {
                if adapter.Type() == C.Pass { continue GetRules }
                if adapter.Type() == C.Rematch { rematchProxy = adapter; break GetRules }
            }
```

（`tunnel/tunnel.go`，760 行；mode 支持 rule/global/direct 三态，`tunnel/mode.go`）

### 5.2 协议适配矩阵（`constant/adapters.go`、`adapter/`）

`AdapterType` 枚举实测包含：基础设施类（Direct/Reject/RejectDrop/Compatible/Pass/PassRule/Rematch/Dns）、分组类（Relay/Selector/Fallback/URLTest/LoadBalance，实现在 `adapter/outboundgroup/`）、以及 24 种协议类型：Shadowsocks、ShadowsocksR、Snell、Socks5、Http、Vmess、Vless、Trojan、Hysteria、Hysteria2、WireGuard、Tuic、Ssh、Mieru、AnyTLS、Sudoku、Masque、TrustTunnel、ShadowQuic、OpenVPN、Tailscale、ZeroTier、EasyTier、GostRelay。`adapter/outbound/` 实际还有 jls/restls/tlsmirror 等实现文件（40 个文件）。对比口径：这与 sing-box 的协议覆盖高度重叠（同源 sing 库），而相对 xray-core（聚焦 VLESS/VMess/Trojan/REALITY 传输安全系）明显更"全家桶"，其中 tailscale/zerotier/easytier 组网型出站是 mihomo 独有方向。

分组实现（`adapter/outboundgroup/`：selector/fallback/urltest/loadbalance + groupbase）配合 `adapter/provider/` 的 proxy-provider（订阅拉取、`healthcheck.go` 健康检查、`override.go` 节点覆写）构成动态节点管理。入站侧 `adapter/inbound/` 只保留原生 http/socks 基座，其余 31 类监听全部在 `listener/`（含各协议服务端与 sing_tun TUN 设备）。

### 5.3 规则引擎（`rules/`）

`rules/parser.go` 的 `ParseRule` switch 实测支持 40+ 规则类型：DOMAIN 系列（精确/后缀/关键字/正则/通配）、GEOSITE/GEOIP/SRC-GEOIP、IP-ASN/SRC-IP-ASN、IP-CIDR(SRC)/IP-SUFFIX(SRC)、DST/SRC/IN-PORT、DSCP、PROCESS 六种（name/path × 精确/正则/通配）、NETWORK、UID、IN-TYPE/IN-USER/IN-NAME/REMATCH-NAME、SUB-RULE、AND/OR/NOT、RULE-SET、MATCH。匹配模型是"顺序线性匹配 + 数据结构加速"：域名类规则经 `component/trie/`（domain set/map 的 succinct 实现）合并加速，rule-set 支持文本与 `.mrs` 二进制格式（`rules/provider/mrs_reader.go`、`mrs_converter.go`），IP 类用 bart 路由表（`go.mod` 中 metacubex/bart）。逻辑规则（AND/OR/NOT/SUB-RULE）在 `rules/logic/logic.go` 中递归组合子规则。

### 5.4 DNS 子系统（`dns/`、`component/fakeip/`）

上游协议覆盖 DoH（`dns/doh.go`）、DoT（`dot.go`）、DoQ（`doq.go`）、UDP/TCP 传统、DHCP（`dhcp.go`）、系统 DNS（`system*.go`）、甚至 tailscale/easytier 内网 DNS（`tailscale.go`、`easytier.go`）；`dns/policy.go` 支持按域名/GeoIP 给不同域名分配不同上游；`middleware.go` 做中间件式包装。缓存增强器 `dns/enhancer.go` 有三种模式：fakeip、mapping、normal，其中 fakeip 模式由 `component/fakeip/pool.go` 管理 v4/v6 双池（`fakeIPPool`/`fakeIPPool6`），`skipper.go` 处理 fakeip 过滤网段——这是 clash/mihomo 系与 sing-box 默认行为的标志性差异之一。

### 5.5 Clash API 与外部枢纽（`hub/`）

`hub/route/server.go` 用 chi 注册完整 RESTful API：`/logs /traffic /memory /version /configs /proxies /group /rules /connections /providers/proxies /providers/rules /cache /dns /storage /restart /upgrade` + `addExternalRouters`（兼容 clash 原版路由），支持 HTTP、HTTPS（含 mTLS client-auth）、Unix socket、Windows named pipe 四种监听方式，可内嵌静态仪表盘（`/ui`）与内嵌 DoH（`hub/route/doh.go`）。`hub/executor/executor.go`（538 行）负责配置热重载。这套 API 与 clash 原版保持兼容，是 metacubexd、Yacd-meta、Clash Verge Rev、OpenClash 等面板无需改动即可对接 mihomo 的原因；`/upgrade`、`/restart`、`/memory` 等为 mihomo 增量扩展。

## 6. 构建与运行

```shell
# 构建（README.md / Makefile）
git clone https://github.com/MetaCubeX/mihomo.git && cd mihomo && go mod download
go build                            # 默认构建
go build -tags with_gvisor          # gVisor TUN 栈
make linux-amd64-v3                 # 单平台；make releases 全平台 30+ 目标
# 测试
go test ./...                       # 单测（Makefile vet 目标同）
cd test && go test                  # 集成测试（独立 go.mod，需 docker/真实节点配置）
# 运行
mihomo -d <配置目录> -f config.yaml  # main.go flag 定义
mihomo generate ech-keypair <域名>   # 内置生成器（main.go / component/generator）
```

CI（`.github/workflows/build.yml`）按 30+ 目标矩阵构建并发布 deb/rpm/pacman 包、Docker 镜像（docker.io）与 GitHub Release；`trigger-cmfa-update.yml` 在发版后自动触发 Android 端 ClashMetaForAndroid 更新，说明核心与官方客户端是发版流水线耦合的。

## 7. 测试与工程质量

- 单元测试：129 个 `_test.go` 与源码同目录分布（实测统计），覆盖 fakeip 池、trie、规则策略、DNS 等纯逻辑单元；风格为 testify 断言（`go.mod` 依赖）。
- 集成测试：`test/` 为独立 go.mod 模块，10 个测试文件，含 docker 环境 e2e（`docker_test.go`）与 vmess/vless/trojan/ss/hysteria/snell 协议连通性测试，`test/config/` 内置各协议服务端配置样例。
- CI 强度可观：`test.yml` 为 6 操作系统（含 windows-arm64、macos-intel）× 7 个 Go 版本（1.20-1.26）的 42 组合矩阵；`build.yml` 为 30+ 平台目标并带并发取消与多包管理器发布。
- 代码质量：根与 test 目录各有一份 `.golangci.yaml`；核心热点文件体量偏大（`config/config.go` 1982 行、`adapter/outbound/vless.go` 918 行），配置解析集中在单文件，属于"能用但密集"的风格。

## 8. 社区与维护状态

- 发版节奏：稳定版 v1.19.21→v1.19.34，2026-03-09 至 2026-11-11 间 14 个 release，约 1.7 个/月（gh api releases 实测）；Alpha 分支为滚动开发，2026-11-19 仍有维护者（wwqgtxx）push（gh api events）。
- 贡献者结构：GitHub contributors API 当前仅返回 2 人（KT-Yeh 25、seriaati 1），明显受默认分支内容影响，Go 侧真实贡献者规模未确认；已知活跃维护者 wwqgtxx（events 记录）。
- **仓库异常客观记录（2026-11-21 实测）**：默认分支 `main` 当前内容为 KT-Yeh 的 Python 库——"崩坏：星穹铁道 Mihomo API 的 Pydantic 模型封装"（`main` 分支 README 自述 `pip install -U git+https://github.com/KT-Yeh/mihomo.git`，MIT，Copyright 2023 KT），main 提交历史最早为 2023-06（作者 KT/seria_ati）。仓库 language/topics/description 随默认分支显示为 Python/星穹铁道；但同一仓库对象上 `Meta`/`Alpha` 分支与 v1.19.x 标签完好，release 持续发布至 2026-11-11，旧名 `MetaCubeX/Clash.Meta` 的 API 重定向也仍指向本仓库。发生时间与原因未确认（本清单 2026-10-24 快照仍记录为 Go、39500 星；2026-11-21 实测星数 35952，一个月内净减约 3500）。**结论：获取源码/构建必须使用 `Meta` 分支，`HEAD` tarball 拿到的是 Python 内容。**

## 9. 亮点与风险

亮点：
1. 协议覆盖为 Go 系代理核心中最广之一：24 种出站协议类型 + 31 类入站监听（`constant/adapters.go`、`listener/`），含组网型出站（wireguard/tailscale/zerotier/easytier）。
2. 规则引擎表达力强：40+ 规则类型、逻辑规则与 sub-rule、.mrs 二进制 rule-set、succinct 域名集与 bart IP 表加速（`rules/`、`component/trie/`）。
3. Clash API 生态位无可替代：API 兼容 clash 原版并扩展，是庞大 GUI 生态（metacubexd、Clash Verge Rev、CMFA、OpenClash 等）的统一内核接口（`hub/route/`）。
4. 工程化程度高：42 组合测试矩阵、30+ 发布目标、自维护依赖 fork 群使 TLS/QUIC/WireGuard 层可快速跟进补丁（`go.mod`、CI）。
5. 单二进制多形态：客户端核心与服务端监听同体，CGO=0 交叉编译覆盖路由器/手机/桌面（`Makefile`）。

风险/局限：
1. 默认分支内容与实际核心不符（§8），对自动化工具、文档引用和新用户构成事实性干扰，仓库元数据暂时失真。
2. 依赖高度自闭环：40+ 直接依赖为 metacubex 自有 fork，上游安全补丁依赖单一组织的维护带宽；go.mod 存在多个 replace（`go.mod`）。
3. 配置复杂度持续膨胀：`config/config.go` 近 2000 行、示例配置 `docs/config.yaml` 项极多，YAML 手写门槛高。
4. 项目受众与使用场景敏感，各发行渠道（第三方 GUI 分发、镜像站）的合规与供应链一致性无法在仓库内核实（未确认）。

## 10. 总结评价

mihomo 是 clash 系规则内核事实上的正统延续：它把原版 clash 的"规则分流 + fakeip + RESTful API"骨架保留为兼容层，在此之上叠加了 Go 代理核心中最宽的协议矩阵、最细的规则粒度和最强的跨平台工程化流水线；与 sing-box 的关系是共享 sing 底层库但产品形态分叉（mihomo 坚持 clash 配置模型与 API 生态，sing-box 走自有配置模型），与 xray-core 则是协议侧重互补（xray 深耕 VLESS/REALITY 传输安全，mihomo 广度优先）。当前唯一显著的非技术问题是仓库默认分支被无关内容占据的异常状态——这不影响 Meta 分支上的核心开发与发布，但让仓库元数据暂时不可信。适合使用者：需要 clash 生态配置模型、要求协议广度与规则细粒度并依赖 GUI 面板的用户；适合读源码者：想理解"规则引擎怎么组织、协议适配层怎么抽象、TUN/DNS/fakeip 怎么协作"的 Go 网络编程学习者，可从 `tunnel/tunnel.go`、`constant/adapters.go`、`hub/route/server.go` 三个入口读起。
