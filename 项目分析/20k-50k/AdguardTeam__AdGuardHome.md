# AdguardTeam/AdGuardHome 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Go（后端）｜档位 20k-50k｜分析日期 2026-11-14
>
> gh api 核实（2026-11-14）：stargazers 38,441，created 2016-07-06，pushed 2026-11-13，forks 2,611，open issues 1,278，license GPL-3.0，最新 release v0.107.80（2026-11-03）。

## 1. 项目速览

一句话定位：**全网级广告与追踪拦截 DNS 服务器**——部署在路由器或家里任意一台机器上，覆盖所有设备、无需装任何客户端的"Pi-hole 替代品"，由商业广告拦截公司 AdGuard 出品并持续维护。

| 项 | 值 |
| --- | --- |
| 名称 | AdGuardTeam/AdGuardHome |
| 星数 | 39,500（快照）；gh api 2026-11-14 实测 38,441 |
| 主语言 | Go（后端）+ TypeScript/SolidJS（新前端） |
| 许可证 | GPL-3.0（`LICENSE.txt`；注意是 GPL-3.0 而非 AGPL） |
| 最近活跃 | pushed 2026-11-13，release v0.107.80 发布于 2026-11-03 |
| 代码规模 | internal/ 非测试 Go 约 62,272 行 + 测试约 36,021 行；前端 client_v2/ 696 个 TS/TSX 文件 |
| 安装形态 | 单个静态二进制（go:embed 内嵌前端），支持 Linux/macOS/Windows/BSD、Docker、Snap |

## 2. 项目是什么，解决什么问题

AdGuard Home 是一个 DNS 服务器：把全网的 DNS 查询劫持到本地，将广告/追踪域名解析到"黑洞"地址，从而让电视、手机、IoT 等所有无需装软件的设备也获得拦截能力（`README.md`）。它基于 AdGuard 自家公共 AdGuard DNS 服务所用的软件，"两者共享大量代码"（`README.md` 开头说明），因此其核心 DNS 处理能力经过了公共服务的规模化检验。

相比"传统浏览器广告拦截器"，它的拦截发生在 DNS 层，覆盖全部协议与应用，但也因此无法处理与页面同域的广告；相比 Pi-hole，README 内置了一张逐项对比表（`README.md#comparison-pi-hole`）：开箱即用的 DoH/DoT/DNSCrypt 加密上游、可自身作为 DoH/DoT 服务端、内置 DHCP、跨平台单二进制、家长控制与强制安全搜索、无需 root 运行等，均为 Pi-hole 需要额外软件或手工配置才能实现的能力。

功能面上，除了规则列表拦截（AdGuard 语法/hosts 文件），还包括：DNS 重写（自定义域名解析）、拦截服务列表（封禁具体服务的域名组）、安全浏览/家长控制（基于哈希前缀查询云端库，`internal/filtering/hashprefix/`）、每客户端（按 IP/ CIDR/MAC）独立配置、访问控制、IPset 集成、DNS64/NAT64、查询日志与统计（`internal/dnsforward/process.go` 的处理链可见全貌）。

架构上正处于一次重要演进的半途：默认构建仍是旧单体入口（`main.go` → `internal/home`），带 `next` 构建标签的新架构（`main_next.go` → `internal/next/`）已完成 v0.108.0 草稿 changelog（`internal/next/changelog.md`），提供全新的分模块 HTTP API 与 SIGHUP 全量重载等能力。

## 3. 技术栈与依赖

后端（`go.mod`，go 1.27.4）：

- `AdguardTeam/dnsproxy` v0.86.0：DNS 转发/代理核心库，实现 DoH/DoT/DoQ/DNSCrypt 上游与多种解析策略（同为 AdGuard 官方库）。
- `AdguardTeam/urlfilter` v0.23.4：AdGuard 拦截规则引擎（Adblock 语法 + hosts 规则），提供 `urlfilter.DNSEngine`（`internal/filtering/rulelist/engine.go`）。
- `AdguardTeam/golibs`、`AdguardTeam/dnscrypt`：自研工具库与 DNSCrypt 协议实现。
- `miekg/dns`：DNS 消息构造与解析的事实标准库。
- `quic-go`：DNS-over-QUIC 支持。
- `insomniacslk/dhcp` + `gopacket`/`mdlayher/*`：DHCP 服务器与底层网络报文处理。
- `go.etcd.io/bbolt`：查询日志与统计的嵌入式存储（`internal/stats/stats.go:391` 处 `bbolt.Open`）。
- `kardianos/service`：以系统服务方式运行（Windows 服务/systemd 等）。

前端（`client_v2/package.json`）：新 UI 用 **SolidJS + @ark-ui/solid + @solidjs/router** + TypeScript 5.4，Webpack 构建，vitest 单测 + Playwright e2e，**orval 从 OpenAPI 规范自动生成 API 客户端**（`api:generate` 脚本）。旧前端 `client/` 为 React 16（已被 linguist 标记 vendored，Makefile 的 `CLIENT_DIR = client_v2` 表明新前端才是默认构建目标）。

API 规范：`openapi/openapi.yaml`（现行 API）与 `openapi/next.yaml`（next 新 API），并有独立的 `openapi/CHANGELOG.md` 管理 API 版本。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `main.go` / `main_next.go` | 两个入口，前者 `//go:build !next` 走旧单体 `internal/home.Main`，后者带 `next` 标签走新架构（均用 `go:embed` 内嵌前端产物 `build/`） |
| `internal/agh*`（aghalg/aghhttp/aghnet/aghos/aghtls/aghslog…） | 横切工具包：HTTP 辅助、网络探测、TLS、slog 封装等 |
| `internal/dnsforward` | DNS 转发服务器核心（约 12,300 行含测试）：请求处理管线、上游选择、访问控制、DNS64、ipset、客户端 ID |
| `internal/filtering` | 拦截引擎：规则列表（AdGuard 语法）、hosts、DNS 重写、安全搜索、拦截服务列表 |
| `internal/dhcpsvc` / `internal/dhcpd` | DHCP 服务器；`dhcpsvc` 为新版（约 9,700 行），`dhcpd` 为旧版（go.mod 注释标注将随依赖移除） |
| `internal/home` | 旧架构 Web 管理后端：认证、/control/* HTTP API、安装向导、服务控制 |
| `internal/next` | 新架构（build tag `next`）：cmd/dnssvc/websvc/configmgr 分层，目标 v0.108.0 |
| `internal/querylog` / `internal/stats` | 查询日志与统计，bbolt 持久化 |
| `internal/client` / `internal/rdns` / `internal/whois` / `internal/arpdb` | 客户端画像：持久客户端、反向 DNS、Whois、ARP 表 |
| `internal/configmgr` / `internal/configmigrate` / `internal/permcheck` / `internal/ossvc` / `internal/updater` / `internal/version` | 配置管理、配置迁移、权限检查、OS 服务集成、自更新、版本信息 |
| `client/` 与 `client_v2/` | 旧 React 前端 / 新 SolidJS 前端（两套并存） |
| `openapi/`、`docker/`、`snap/`、`build/`、`scripts/` | API 规范、多套 Dockerfile（ci/build/frontend）、Snap 打包、构建脚本 |

组织方式评述：典型的"Go 后端单体 + 内嵌前端"应用，但模块边界划分非常清晰——`internal/` 下 30 个包几乎每个都有独立的配置结构体与 `_internal_test.go`；横切工具用 `agh*` 前缀命名空间隔离；同时通过"新旧双入口 + build tag"实现大型重构平滑过渡，这在开源项目里是比较少见且值得借鉴的迁移手法。依赖上大量复用 AdGuard 自家开源库（dnsproxy/urlfilter/golibs），核心能力（加密上游、规则引擎）实际上沉淀在可独立复用的库里。

## 5. 核心架构与关键实现

### 5.1 DNS 请求处理管线（internal/dnsforward）

整个服务器实现为 `dnsproxy.Proxy` 的一个 handler，请求按固定顺序流过 9 个处理阶段（`internal/dnsforward/requesthandler.go:18`）：

```go
mods := []modProcessFunc{
    s.processInitial,
    s.processDDRQuery,
    s.processDHCPHosts,
    s.processDHCPAddrs,
    s.processFilteringBeforeRequest,
    s.processUpstream,
    s.processFilteringAfterResponse,
    s.ipset.process,
    s.processQueryLogsAndStats,
}
for _, process := range mods {
    r := process(ctx, l, dctx)
    switch r {
    case resultCodeSuccess: // 继续下一阶段
    case resultCodeFinish:  return nil   // 已生成响应，直接返回
    case resultCodeError:   return dctx.err
    }
}
```

每个阶段返回 success/finish/error 三态结果码，形成一条可中断的中间件链：初始校验 → DDR（Discovery of Designated Resolvers）应答 → DHCP 主机名解析 → 前置过滤（命中拦截即终止）→ 上游解析 → 后置过滤 → ipset 写入 → 日志与统计。上游选择逻辑在 `internal/dnsforward/upstreams.go`：`proxy.ParseUpstreamsConfig` 支持按域名指定上游（`[/domain/]upstream` 语法），bootstrap 解析器用 `upstream.ParallelResolver` 并发查询并包一层 `NewCachingResolver` 缓存，`/etc/hosts` 解析以 `upstream.ConsequentResolver` 串联在并行解析器之前（`upstreams.go:newBootstrap`）。默认上游已从 Quad9 换为 Cloudflare（`dnsforward.go` 的 `defaultDNSUpstream = https://cloudflare-dns.com/dns-query`，此变更即 v0.107.80 更新项，`CHANGELOG.md`）。

### 5.2 拦截引擎（internal/filtering + urlfilter）

`DNSFilter`（`internal/filtering/filtering.go:255`）是规则匹配的门面：`CheckHost` 先处理重写规则（`processRewrites`），再依次走规则列表、hosts 文件、安全浏览/家长控制。AdGuard 语法的实际匹配委托给 urlfilter 的 `DNSEngine`，`internal/filtering/rulelist/engine.go` 的 `Engine` 用 `sync.RWMutex` 保护引擎与存储、支持多规则列表原子刷新：

```go
type Engine struct {
    mu      *sync.RWMutex
    engine  *urlfilter.DNSEngine      // 组合多个规则列表的匹配引擎
    storage *filterlist.RuleStorage   // 规则存储，负责释放
    filters []*Filter                 // 各规则列表元信息
}
```

匹配结果统一收敛为 `Result` 与 `Reason` 枚举（`internal/filtering/reason.go`：FilteredBlockList/NotFilteredAllowList/Rewritten/Customized 等），白名单规则（`Whitelist` network rule）与 IPv4/IPv6 hosts 规则的优先级处理在 `matchHostProcessDNSResult`（`filtering.go`）。hosts 文件解析在 `hosts.go`，安全搜索/家长控制的云端查询走 `hashprefix/`（对域名做哈希前缀本地缓存后查 AdGuard 云端库）。旧版 `dhcpd` 之外，新 DHCP 服务 `internal/dhcpsvc` 已自带 IPv4/IPv6 handler、JSON 租约存储与接口检测，代码规模约 9,700 行。

### 5.3 Web 管理后端与新旧架构迁移（internal/home、internal/next）

旧架构 `internal/home.Main`（`internal/home/home.go`，入口见 `main.go`）在一个进程里组装全部组件：认证（basic auth + 速率限制 `authratelimiter.go`，GL.iNet 路由器专用认证 `authglinet.go`）、安装向导（`controlinstall.go`）、自更新（`controlupdate.go`）、TLS 配置（`control.go` 中 `/control/tls/configure|validate`）以及大量 `/control/*` JSON API（`control.go`、`dns.go`、`clientshttp.go`、`filtering/http.go` 等分别注册，control.go 内 13 处 `httpReg.Register`，全仓更多）。DoH 服务端能力在 `internal/home/doh.go`——AdGuard Home 自己也能作为加密 DNS 入口对外提供服务。

新架构 `internal/next/` 按服务分层（`cmd/`、`dnssvc/`、`websvc/`、`configmgr/`、`jsonpatch/`），配套 `openapi/next.yaml` 全新 API 设计与 `internal/next/changelog.md` 的 v0.108.0 草稿（SIGHUP 全量重载、`--web-addr` 等新能力）。通过 `go build -tags next` 即可构建实验版本，两套代码互不干扰。

## 6. 构建与运行

来自 `README.md`（How to build from source）与 `Makefile`（POSIX 风格，明确标注不支持 `-j` 并行）：

```bash
# 依赖：Go 1.27+、Node.js 18+、npm 10.8+
make init        # 准备开发环境（拉取前端依赖等）
make             # 本地构建（前端 client_v2 + Go 后端，前端产物 embed 进二进制）

make js-build    # 仅构建前端
make test        # Go 测试
make release     # 发布构建（版本号/渠道/GPG 签名通过变量传入，见 Makefile VERSION/CHANNEL/SIGN）

# 构建实验性新架构
go build -tags next

# 运行
./AdGuardHome                            # 默认监听 3000 端口初始化向导
./AdGuardHome -s install                 # 注册为系统服务（`-s` 见 README service 管理）
```

安装快捷方式：`curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v`（`README.md`）。Docker 镜像 `adguard/adguardhome`，Snap 包 `snapcraft.io/adguard-home`（README 徽章 + `docker/`、`snap/` 目录）。

## 7. 测试与工程质量

- 测试规模：`internal/` 内测试代码 36,021 行 / 非测试代码 62,272 行，比例约 58%（tarball 内 wc 统计）。风格以同包内部测试为主（大量 `*_internal_test.go`），断言用 `testify` + `google/go-cmp`，核心包（dnsforward、filtering、dhcpsvc、rulelist）均有独立测试文件与 `testdata/` 固定数据。
- 前端测试：vitest 单测 + Playwright e2e（`client_v2/package.json` scripts）。
- CI：`.github/workflows/` 含 build.yaml（多平台构建）、release.yaml（发布流水线）、snapcraft.yaml（Snap 商店）、mirror.yaml、workflow.yaml、private.yaml；另有 `docker/ci.Dockerfile` 专用 CI 构建镜像。
- 静态质量：`staticcheck.conf`（staticcheck 检查配置在仓库根部）、HACKING.md 与 CONTRIBUTING.md 约定贡献流程；`openapi/CHANGELOG.md` 对 API 变更单独做版本管理，工程质量意识突出。
- 代码中存在大量结构化 TODO 注释（如 go.mod 中标注废弃依赖的移除计划），显示明确的债务管理。

## 8. 社区与维护状态

- 活跃度：gh api 显示最近 push 为 2026-11-13（分析前一天），开放 issue 1,278，属高强度维护。项目创建于 2016-07-06，已持续迭代近十年。
- Release 节奏：CHANGELOG.md 记录 v0.107.74（2026-04-16）→ v0.107.75（05-19）→ v0.107.76（05-21）→ v0.107.77（06-01）→ v0.107.78（07-13）→ v0.107.79（08-18）→ v0.107.80（2026-11-03），平均 1-2 个月一个 minor/patch 发布，节奏稳定。
- 贡献者结构：AdGuard 公司团队主导（代码注释中的 a.garipov、e.burkov、ameshkov、f.setrakov 等为长期核心开发者），社区通过 issue/PR/翻译参与（README 有专门翻译贡献章节）；精确贡献者人数未确认（tarball 无 git 历史，未做 gh api 全量统计）。
- 商业背书：AdGuard 将其作为公共 AdGuard DNS 的同源软件维护（README 自述），并有多语言 i18n（`internal/home/i18n.go`、前端 locales 脚本）。
- 版本规划：v0.108.0（next 新架构）已有草稿 changelog，v1.0.0 在 CHANGELOG 中预留占位（`CHANGELOG.md` 注释块）。

## 9. 亮点与风险

亮点：

1. **架构迁移范式**：用 build tag 双入口（`main.go`/`main_next.go`）+ 分层新包（`internal/next/`）+ 新 API 规范（`openapi/next.yaml`）完成平滑重构，新旧互不阻塞，值得任何在做大改造的项目参考。
2. **核心能力库化**：DNS 加密上游（dnsproxy）、规则引擎（urlfilter）沉淀为独立开源库，既服务自家公共 DNS 商业服务，也让社区可复用——开源与商业形成良性循环。
3. **工程质量高**：约 58% 的测试代码占比、结构化 TODO 债务管理、独立 API changelog、POSIX 规范 Makefile、staticcheck 约束，在 38k 星级项目中属第一梯队。
4. **部署面完整**：单静态二进制内嵌前端、跨 6 类 OS、Docker/Snap/自动安装脚本/自更新（`internal/updater/`）全链路覆盖，Pi-hole 对比表中"开箱即用"的定位在代码层面得到支撑。

风险/局限：

1. **GPL-3.0 而非 AGPL**：作为网络服务型软件，GPL-3.0 不强制公开修改后以服务形式提供的源码（注意：仓库实际许可证为 GPL-3.0，`LICENSE.txt`，坊间常误传为 AGPL）。
2. **双前端/双 DHCP 的过渡期负担**：`client/`（React 旧版）与 `client_v2/`（SolidJS 新版）、`dhcpd` 与 `dhcpsvc` 并存，新用户阅读代码或自行构建时容易踩错目录；旧 dhcpd 连同 gopacket 旧依赖已在 go.mod 标注待移除。
3. **单机定位**：面向家庭/小型网络单节点部署，无内建集群/多节点同步；统计与日志依赖本地 bbolt，大规模场景需另行取舍（`internal/stats/stats.go`）。
4. **DNS sinkholing 的原理性局限**：README 自己承认无法处理与内容同域名的广告（`README.md#comparison-adblock`），且 DNS 层拦截对 CDN 共享域名的误伤需要靠白名单维护。

## 10. 总结评价

AdGuard Home 是"公司级工程能力反哺开源"的标杆案例：AdGuard 把公共 DNS 服务的同源代码以 GPL-3.0 开源，用约 10 万行 Go + 全新 SolidJS 前端做出了 Pi-hole 的强力替代品，功能上（加密上游、DHCP、家长控制、每客户端配置）全面超集，工程上（测试占比、模块边界、API 版本管理、双入口平滑重构）堪称 Go 后端项目的优秀教材。它正处在 v0.107 → v0.108 架构迁移的中途，是少数能让人"看着"一个成熟项目完成重生过程的样本。适合家庭/小团队用户直接部署使用，适合 Go 开发者精读 `dnsforward` 管线与 `next` 迁移策略；想研究规则引擎本体的人则应顺藤摸瓜去看 `AdguardTeam/urlfilter` 与 `dnsproxy` 两个子项目。
