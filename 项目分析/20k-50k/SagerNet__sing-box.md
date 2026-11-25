# SagerNet/sing-box 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Go（iOS/macOS 客户端 Swift）｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：通用代理平台（The universal proxy platform），以统一内核同时承接"多协议出站/入站、透明代理 TUN、DNS 路由、规则引擎、多平台 GUI 内嵌 SDK"五类能力的 Go 实现。

| 事实 | 值 | 出处 |
| --- | --- | --- |
| 名称 | SagerNet/sing-box | gh api |
| 星数 | 39,937（2026-11-14 核实；任务快照 39,500） | gh api repos/SagerNet/sing-box |
| 主语言 | Go（go.mod 声明 go 1.25.5）；Apple 图形端为独立仓（Swift） | go.mod；.gitmodules |
| 许可证 | GPL-3.0 + 商标附加条款（"衍生作品不得使用本应用名称或暗示关联"），GitHub 标记 NOASSERTION | LICENSE；README.md |
| 默认分支 | testing（另有 oldstable/stable/testing/unstable 四线分支） | gh api；.github/workflows/test.yml |
| 最近活跃 | 最近 push 2026-11-10，最近 release v1.14.4 / v1.15.0-alpha.13（均 2026-10-24） | gh api |
| 规模 | tarball 内 1846 个文件、1231 个 .go 文件（HEAD/testing 分支） | tar 统计 |
| 创建时间 | 2022-06-30 | gh api |

## 2. 项目是什么，解决什么问题

sing-box 是一个由 SagerNet 组织维护的 Go 代理内核，自述为"通用代理平台"。它把一个完整代理栈所需的各个子系统——协议实现、入站承接（socks/http/mixed/tun/redirect）、DNS 服务器与解析路由、流量规则匹配、出站拨号——收敛到同一个二进制/库内，并用统一的 JSON 配置模型驱动（`docs/configuration/` 有 266 个配置文档文件，含中英双语，`docs/schema.json` 可由 `cmd_schema.go` 自动生成）。

与"单一协议工具"不同，sing-box 的设计重心在"平台"二字：核心代码按注册表（Registry）模式组织，入站/出站/端点/DNS 传输/服务/证书提供者六类组件都通过 `include/registry.go` 汇总注册，每类组件均可独立增删（构建标签裁剪，见 `release/DEFAULT_BUILD_TAGS`）。同时它通过 `experimental/libbox` 打包成 Android/iOS/桌面可用的库（gomobile 与 sing-ffi 两条路径，`Makefile` 的 `lib_android`/`lib_apple`/`lib_windows` 目标），官方图形客户端 SFI/SFM/SFT（iOS/macOS/tvOS）、SFA（Android）与桌面端均为独立仓库，以 git submodule 挂在本仓 `clients/` 下（`.gitmodules`：sing-box-for-apple、sing-box-for-android、sing-box-for-desktop）。

协议覆盖面非常宽：`protocol/` 下有 29 个协议目录，横跨传统代理（socks/http）、Shadowsocks 系（shadowsocks/shadowtls/snell/anytls/naive）、V2Ray/Xray 系（vmess/vless/trojan）、QUIC 系（hysteria/hysteria2/tuic/masque/cloudflared）、VPN 系（wireguard/openvpn/openconnect/tailscale/tor/ssh）与平台组件（direct/block/mixed/redirect/tun/group/bridge/dns）。也就是说它同时是这些协议的客户端与服务端实现，而非某一协议的专属工具。

路由层（`route/`）与 DNS 层（`dns/`）被拆成独立子系统，规则同时支持路由规则与 DNS 规则两套模型，规则可编译为自有二进制格式 srs（`cmd_rule_set_compile.go`，`common/srs/`，iOS 上用 mmap 直接映射加载，`route/rule/rule_set_mmap_ios.go`），这是区别于 Clash/Xray 生态的自有规则资产格式。

测试分支（HEAD 即 testing）中还能看到明显超出"代理"范畴的新方向：`service/ccm` 与 `service/ocm` 是基于 anthropic-sdk-go / openai-go 的 AI API 网关服务（`service/ccm/service.go` 内置 200000/1000000 上下文窗口常量与 Anthropic 格式错误响应），另有 daemon（gRPC 管理守护进程）、usbip over TCP 等试验能力。

## 3. 技术栈与依赖

- 语言与构建：Go 1.25.5（`go.mod`），cobra CLI（`cmd/sing-box`），goreleaser 发布（`Makefile` release 目标），golangci-lint 多 GOOS 检查（`Makefile` lint 目标、`.golangci.yml`）。
- 自研/自维护 fork 生态（go.mod 直接依赖）：`sagernet/sing`（基础网络库）、`sagernet/quic-go`、`sagernet/wireguard-go`、`sagernet/gvisor`（用户态 TCP/IP 栈）、`sagernet/tailscale`（内嵌 Tailscale）、`sagernet/sing-tun`、`sing-mux`、`sing-vmess`、`sing-shadowsocks(2)`、`sing-shadowtls`、`sing-quic`、`sing-snell`、`sing-anytls`、`sing-cloudflared`、`sing-openconnect`、`sing-openvpn`、`sing-usbip`、`smux`、`gomobile`。协议核心大多下沉到这些兄弟仓库，本仓持有适配与组装层。
- 关键第三方库：`miekg/dns`（DNS 报文）、`metacubex/utls`（uTLS 指纹）、`caddyserver/certmagic` + 一组 `libdns/*`（ACME 自动证书，`service/acme/`）、`google.golang.org/grpc` + `protobuf`（v2ray-api 与 daemon）、`go-chi/chi`（clashapi/ai 网关 HTTP 路由）、`oschwald/maxminddb-golang`（GeoIP）、`anthropics/anthropic-sdk-go` 与 `openai/openai-go/v3`（AI 网关服务）、`sagernet/cronet-go`（Chromium Cronet，go.mod 中含约 30 个平台三元组的预编译库，用于移动端）。
- 构建标签体系：`release/DEFAULT_BUILD_TAGS_OTHERS` = `with_quic,with_dhcp,with_wireguard,with_utls,with_acme,with_clash_api,with_tailscale,with_ccm,with_ocm,with_cloudflared,with_usbip,with_openvpn,with_openconnect` 等，由 `include/*_stub.go` 成对文件实现条件编译裁剪。

## 4. 目录结构与代码组织

| 目录 | 职责（依据 tar 包内文件） |
| --- | --- |
| `box.go` | 组合根：Box 结构体持有全部 Manager，分阶段启动 |
| `adapter/` | 全部核心抽象接口（Inbound/Outbound/Endpoint/Service/DNS/Router/规则动作等，57 文件） |
| `protocol/` | 29 个协议实现目录（含 tun、group、direct 等平台型组件） |
| `route/` | 路由引擎：Router、连接路由、规则（`route/rule/` 48 个规则项文件）、网络环境（per-OS 文件）、FakeIP |
| `dns/` | DNS 服务器框架：transport（udp/tcp/tls/https/quic/dhcp/hosts/fakeip/local/mdns）、DNS 路由、传输管理 |
| `common/` | 41 个横切组件：sniff（协议嗅探）、tls（含 tlsfragment/tlsspoof/ja3/ktls/schannel）、dialer、listener、mux、geoip/geosite、srs、uot、windivert 等 |
| `transport/` | 传输层扩展：v2ray 全家桶（ws/http/grpc/httpupgrade/quic）、simple-obfs、sip003、wireguard/openvpn/openconnect/masque 设备层 |
| `service/` | 长驻服务组件：api、acme、ccm/ocm（AI 网关）、derp、origin_ca、resolved、ssmapi、oomkiller、usbip、powerreport |
| `experimental/` | libbox（多平台 SDK，60+ 文件）、clashapi、v2rayapi、cachefile、clashmode、boxdd、locale、deprecated |
| `include/` | 构建标签接线层：把各协议/服务注册进六大 Registry |
| `option/` | 全部 JSON 配置结构定义（70 文件），与 docs/configuration 一一对应 |
| `cmd/sing-box` | CLI：run/check/format/merge/generate/rule-set(compile|convert|decompile)/geoip/geosite/tools/api（100+ 命令文件） |
| `daemon/` | 桌面守护进程：gRPC 管理/实例服务（proto + pb） |
| `test/` | 独立 Go module 的端到端集成测试（47 个测试文件） |
| `docs/` | mkdocs 文档站，i18n 双语（suffix 结构，`mkdocs.yml`），configuration 266 文件 |
| `clients/` | 三个图形端 submodule 挂载点（tar 包内为空壳，本体在独立仓库） |
| `.github/` | CI 工作流 5 个 + 大量平台适配脚本（iOS/Windows7/WindowsARM64 的 Go 补丁等） |
| `release/` | 构建标签、LDFLAGS、打包脚本 |

组织方式评述：单体多模块仓库（单 go module），但通过"接口层 adapter + 注册表 + 条件编译"实现了插件式解耦；协议实现大量外包给 sagernet 生态的兄弟仓库，本仓更像"组装与编排层"。抽象层次清晰：`protocol/*` 只依赖 `adapter` 接口与 `option` 配置，新增协议即新增一个目录 + 一行注册。

## 5. 核心架构与关键实现

### 5.1 组合根与生命周期（box.go）

`Box` 结构体是唯一组合根，持有 network/endpoint/inbound/outbound/service/certificateProvider/dnsTransport/dnsRouter/connection/router/referenceManager 十余个 Manager（`box.go` 第 332-352 行）。依赖注入采用"服务注册表进 context"的 service-locator 模式：`box.Context()` 把六类 Registry 注入 context（`box.go`），组件内部用 `service.FromContext[adapter.Router](ctx)` 取协作对象（见 `route/rule/rule_default.go`）。启动严格分阶段：`StartStateInitialize → Start → PostStart → Started` 四阶段遍历组件（`box.go` 的 `start()`），保证"出站先于入站可用"这类顺序约束。关键代码（box.go）：

```go
type Box struct {
    createdAt           time.Time
    logFactory          log.Factory
    network             *route.NetworkManager
    endpoint            *endpoint.Manager
    inbound             *inbound.Manager
    outbound            *outbound.Manager
    dnsTransport        *dns.TransportManager
    dnsRouter           *dns.Router
    connection          *route.ConnectionManager
    router              *route.Router
    ...
}
```

### 5.2 入站/出站抽象层（adapter/）

抽象刻意极简：`Outbound` 只要求 `Type/Tag/Network/Dependencies + N.Dialer`（`adapter/outbound.go`），即任何出站本质是一个可拨号的 Dialer，代理协议与"直连"同构；`Endpoint` = `Outbound + Lifecycle`（`adapter/endpoint.go`），承载 wireguard/tailscale/openvpn 这类"有状态双向节点"而非单向出站。入站侧的 `InboundContext`（`adapter/inbound.go`）是全链路元数据中心：来源/目的地、嗅探结果（Protocol/Domain/Client）、进程信息、GeoIP、FakeIP、TLS 分片/伪装标记，以及一组规则匹配缓存位段（`ResetRuleMatchCache()`），规则引擎靠它避免重复匹配。

### 5.3 路由规则引擎（route/ + route/rule/）

连接路由入口在 `route/route.go` 的 `routeConnection()`：先处理 detour 防环与 DNS 劫持（tun 入站的 DNS 流直接 hijack），再 `matchRule()` 得到选中规则，按 `RuleActionRoute / RuleActionBypass / RuleActionReject / RuleActionHijackDNS` 分派——路由结果不是"出站名"而是"动作对象"。规则模型分 Default（AND 组合）与 Logical 两类（`route/rule/rule_default.go`），每条规则由若干 `RuleItem` 组成，`RuleItem` 接口只有 `Match(metadata) bool` + `String()` 两个方法：

```go
type RuleItem interface {
    Match(metadata *adapter.InboundContext) bool
    String() string
}
```

`route/rule/` 下有 48 个规则项文件，覆盖 domain/keyword/regex、ip_cidr、port、protocol（嗅探结果）、process_name/path、package_name（Android）、wifi_ssid/bssid、network_type/expensive/constrained（蜂窝网络感知）、rule_set（引用编译规则集）等；`rule_abstract.go` 把 items 分为 source/destination/deferred-IPCIDR 分组以支持"域名先匹配、IP 后确认"的两段式匹配（DeferredIPCIDRMatchGroups 位段）。规则集支持本地 srs（mmap，iOS 专用路径 `rule_set_mmap_ios.go`）与远程 HTTP 下载（`rule_set_remote.go` + `rule_set_updater.go`），并有更新校验测试（`rule_set_update_validation_test.go`）。DNS 侧另有平行的 DNS 规则（`route/rule/rule_dns.go`、`dns/router.go`）。

### 5.4 多平台 SDK（experimental/libbox）

libbox 是 GUI 客户端共用的 SDK 层：gomobile 绑定（Android/Apple，`Makefile` lib_android/lib_apple 目标，`cmd/internal/build_libbox`）与 sing-ffi 生成 C# 绑定（Windows，`experimental/libbox/ffi.json`）。其 API 面（60+ 文件）包括：本地命令服务器/客户端（`command_server.go`/`command_client.go`，移动端 UI 通过 Unix socket 控制）、平台接口（`platform.go` 实现 `adapter.PlatformInterface`）、TUN 管理、日志桥接、OOM/电量上报（`oom_report.go`/`power_report.go`）、邻居表/网络质量/STUN 查询等。cronet-go 预编译库（go.mod indirect 约 30 个平台三元组）让移动端可复用系统 HTTP 栈。桌面另有 `daemon/`：gRPC 管理服务（`managed_service.proto`、`started_service.proto`）托管多实例并暴露 taildrop/usbip 等能力，`cmd/sing-box` 的 `cmd_api_*` 系列命令则通过 API 服务操控运行中的实例。

### 5.5 与 Xray/Clash 生态的兼容与自建（客观描述）

- 协议兼容：vmess/vless/trojan/naive 均有 client+server 实现（`protocol/vless`、`protocol/vmess` 等）；v2ray 传输层（ws/http/grpc/httpupgrade/quic）在 `transport/v2ray*` 独立成包；v2ray-protobuf 统计 API 兼容在 `experimental/v2rayapi`。
- Clash 兼容：`experimental/clashapi`（17 文件）实现 Clash External Controller REST 语义（configs/connections/proxies/rules/providers，含 api_meta 扩展），使 Clash 面板生态可直接对接；`experimental/clashmode` 提供 mode 切换。
- 自建资产：规则集 srs 二进制格式与编译/反编译 CLI（`cmd_rule_set_*.go`、`common/srs/`）、自有新协议（hysteria2/tuic 的完整实现、anytls、snell 客户端、masque）不依赖上述两系生态。

## 6. 构建与运行

来自 Makefile 与 CI 的实际命令（未运行，静态摘录）：

- 本地构建：`make build`（等效 `go build -tags "$(cat release/DEFAULT_BUILD_TAGS_OTHERS)" -ldflags "-X 'github.com/sagernet/sing-box/constant.Version=...' ..." ./cmd/sing-box`）
- 测试：`make test`（`go test -v ./...` + 进入 `test/` 独立 module 再跑一轮，CI 在 ubuntu/windows/macos × Go 1.25/1.26 矩阵执行，`-exec sudo` 用于需要 root 的 TUN 用例）
- Lint：`make lint`（golangci-lint 按 GOOS=linux/android/windows/darwin 各跑一遍）
- 文档：`make docs`（mkdocs-material + mkdocs-static-i18n）
- 移动库：`make lib_android` / `make lib_apple`（gomobile）、`make lib_windows`（sing-ffi + ffi.json）
- 发布：`make release`（goreleaser + ghr，草稿 pre-release）；图形端发布走 `../sing-box-for-android`、`../sing-box-for-apple` 的 xcodebuild/gradlew 目标（Makefile build_android/build_ios/build_macos 等）
- 运行：`sing-box run -c config.json`；配套 `sing-box check`、`sing-box format`、`sing-box merge`、`sing-box rule-set compile`（`cmd/` 下对应 cmd_run.go/cmd_check.go 等）

## 7. 测试与工程质量

- 单元测试：主 module 内 107 个 `_test.go`，集中在 `common/sniff`（每种嗅探协议一个测试）、`common/srs`、`common/geosite`、`route/rule`（match_state/nested_action/semantics 等）与 `adapter`；`route/dns/router_race_test.go` 含 race 检测。
- 集成测试：`test/` 为独立 Go module，47 个测试文件按"协议×场景"组织（box_test、clash_test、hysteria2_test、http_h3_test、ktls_test、ech_test、docker_test 等），属黑盒端到端风格：起两个 Box 实例互连并断言流量路径。
- CI：`.github/workflows/` 有 test/build/docker/lint/linux 五条流水线；test.yml 覆盖 3 OS × 2 Go 版本并对四个分支（oldstable/stable/testing/unstable）同时生效；lint.yml 配合 `.golangci.yml`。
- 工程质量观察：抽象接口层薄且稳定（adapter 目录多为纯接口文件）；条件编译成对文件（*_stub.go）纪律良好；`include/registry.go` 单点登记全部组件便于审计；版本迁移有专门文档与 `cmd_merge.go`/`deprecated` 包承接旧配置。代码量 1231 个 .go 文件对单一模块而言体量大，但目录边界清晰。

## 8. 社区与维护状态

- 活跃度：最近 push 2026-11-10（gh api），tar 包内 1231 个文件持续高频变动；open issues 414。
- 维护节奏：四分支并行（oldstable/stable/testing/unstable，test.yml 触发分支列表）；release 双轨——稳定线 v1.14.2（09-24）→ v1.14.3（10-09）→ v1.14.4（10-24），前置线 v1.15.0-alpha.7 至 alpha.13（09-22 至 10-24，约每周一发）（gh api releases）。tarball HEAD 默认分支为 testing，代表 v1.15 开发态。
- 贡献者：GitHub contributors 共 103 人（gh api 分页 100+3），核心提交高度集中于单一维护者 nekohasekai（contributors 排序首位，`Makefile`/大量协议仓 fork 均为其个人命名空间）；中文 issue 模板（`bug_report_zh.yml`）与双语文档显示中英双社区并重。
- 生态外围：文档站 sing-box.sagernet.org（README）、repology 打包状态徽章（README.md）、repology 显示多发行版打包。GUI 三端与 libbox 绑定均由本仓 Makefile 统一驱动，版本号同轨（`update_android_version`/`update_apple_version` 目标）。

## 9. 亮点与风险

亮点：
1. 抽象层极简且自洽：Outbound=Dialer、Endpoint=有状态 Outbound、规则=动作对象（`adapter/outbound.go`、`adapter/endpoint.go`、`route/route.go`），新协议/新动作接入成本低。
2. 规则引擎工程化程度高：48 个规则项 + 两段式 IP 延迟匹配 + srs 编译格式 + mmap 加载（`route/rule/`、`common/srs/`、`rule_set_mmap_ios.go`），面向移动端做了真实优化。
3. 平台纵深完整：从 Linux nftables/netlink 到 Windows windivert/schannel/wintun、macOS/Apple 全端、Android cronet，均有对应实现文件（`common/`、`route/network_environment_*.go`），甚至维护 Windows7/老 macOS 的 Go 补丁脚本（`.github/patch_go_for_ios.sh`、`setup_go_for_windows7.sh`）。
4. 兼容与自建并举：既实现 Clash/V2Ray 生态的 API 与协议兼容面（`experimental/clashapi`、`experimental/v2rayapi`），又推进自有协议与规则资产，降低单点依赖。
5. 测试分层清晰：单元（107 文件）+ 独立 module 黑盒集成（47 文件）+ 3 OS/2 Go 版本矩阵 CI。

风险/局限：
1. 供应链自持率高：核心 quic-go/gvisor/tailscale/wireguard-go 等均为自维护 fork（go.mod），上游安全补丁的吸收依赖单人维护者的持续投入。
2. GPL-3.0 附加名称条款超出 GPL 语义（LICENSE 附言），商用集成与二开需额外注意商标边界；GitHub 许可证识别为 NOASSERTION。
3. 默认分支为 testing，HEAD 含大量未达稳定的试验特性（AI 网关 ccm/ocm、usbip、daemon），引用 HEAD 代码时结论可能不适用于 stable 分支；本报告即基于 testing 快照。
4. 功能面极宽（29 协议 + AI 网关 + 隧道 + VPN），单二进制的攻击面与审计成本随之上升；构建标签裁剪是必要而非可选项。

## 10. 总结评价

sing-box 是当前少见的"把代理栈做成平台化工程"的项目：它没有停留在协议集合层面，而是用极简接口 + 注册表 + 条件编译构建了一个可裁剪、可嵌入、可跨端复用的内核，并围绕它建起了自有规则格式、多平台 SDK、统一发布流水线与图形端矩阵；对 Xray 与 Clash 两系生态保持了协议与 API 双向兼容，同时持续孵化自有协议。工程上接口边界干净、测试分层完整、平台覆盖扎实，代价是自维护 fork 群与单人核心维护带来的可持续性风险。适合作为多协议代理内核的架构参考——尤其值得读 `adapter/`（接口设计）、`route/rule/`（规则引擎）、`experimental/libbox/`（Go 库多端分发）三处；对于想直接使用或二次开发的用户，则应锁定 stable 分支与已发布版本，而非 HEAD。
