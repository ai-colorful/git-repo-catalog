# MHSanaei/3x-ui 项目分析

> 星数 47000（快照 2026-10-24）｜主语言 Go｜档位 20k-50k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：**3X-UI 是一个基于 Go + React 的 Xray-core Web 管理面板，把手工编辑 JSON 配置的代理服务器运维工作图形化、多用户化，并提供订阅分发、流量计费与多节点管理**（出处：`README.md`、`main.go` 包注释）。

| 事实 | 值 | 出处 |
|---|---|---|
| 名称 | MHSanaei/3x-ui（module `github.com/mhsanaei/3x-ui/v3`） | `go.mod` |
| 星数 | 47000（快照）；gh api 实测 49290（2026-11-04） | gh api |
| 主语言 | Go（后端）+ TypeScript/React（前端） | `go.mod`、`frontend/package.json` |
| 许可证 | GPL-3.0 | `LICENSE`、gh api |
| 创建 / 最近 push | 2023-02-09 / 2026-11-03 | gh api |
| 最新 release | v3.11.0（2026-10-25），另有 dev-latest 滚动预发布 | gh api releases |
| 规模 | Go 约 23.3 万行（含测试，tarball 统计）；tarball 共 2539 个条目 | tar -tzf / tar -xzOf 统计 |
| 文档站 | docs.sanaei.dev（独立 Next.js/Fumadocs 站点，在 `docs/`） | `README.md`、`CLAUDE.md` |

## 2. 项目是什么，解决什么问题

3X-UI 自述为 "advanced, open-source web control panel for managing Xray-core servers"，是原 X-UI 项目的增强分支（`README.md`）。Xray-core 本身是纯命令行的代理内核，配置为一份大 JSON；在多用户、多协议、流量配额、到期管理等场景下手工运维成本很高。3X-UI 用一个 Web 面板把这件事产品化：单二进制启动（前端产物经 `go:embed` 打进 Go 二进制，`internal/web/web.go`），浏览器里完成入站配置、用户增删、流量/到期/IP 限制管理、订阅链接分发。

协议与传输覆盖（`README.md` Features、`internal/database/model/model.go` 的 Protocol 枚举）：
- 入站协议：VMess、VLESS、Trojan、Shadowsocks、WireGuard、AmneziaWG、Hysteria(2)、TUIC v5、MTProto、HTTP、SOCKS/Mixed、Dokodemo-door/Tunnel/TUN、MASQUE（`model.go` validate `oneof` 含 `masque`）。
- 传输层：TCP(Raw)、mKCP、WebSocket、gRPC、HTTPUpgrade、XHTTP；安全层 TLS、XTLS、REALITY；支持单端口 Fallback 多协议。
- 出站与路由：WARP、NordVPN、PIA（WireGuard 客户端，`internal/pia/`）、自定义路由规则、负载均衡、出站链式代理。

除"配置生成器"外，面板还内置了一批运营设施：每客户端流量配额、到期时间、IP 数限制（fail2ban 联动）、HWID 设备限制、周期性流量重置、内置订阅服务器（raw/JSON/Clash，按 User-Agent 自动选择，`internal/sub/`）、Telegram 与 Discord 机器人（`internal/web/service/tgbot/`、`discord/`）、以及 master/node 多机管理（克隆入站到其他节点）。README 明确声明"intended for personal use only"，定位个人自用工具而非商业产品。

## 3. 技术栈与依赖

**后端（Go 1.27，`go.mod`）**
- Web 框架：`gin-gonic/gin` + `gin-contrib/sessions`（cookie 会话）+ `gin-contrib/gzip`。
- ORM/存储：`gorm.io/gorm` + `mattn/go-sqlite3`（默认 SQLite，需 CGO）+ `jackc/pgx/v5`（可选 PostgreSQL）；`XUI_DB_TYPE`/`XUI_DB_DSN` 切换（`README.md` Database Options）。
- Xray 集成：`github.com/xtls/xray-core` 作为 Go 库引入（取其配置类型与 gRPC stats/handler/router API），同时以子进程方式托管运行真正的 xray 二进制（`CLAUDE.md`、`internal/xray/process.go`）。
- 代理协议相关：`quic-go`（两套：`quic-go/quic-go` 与 `apernet/quic-go`）、`refraction-networking/utls`、`amnezia-vpn/amneziawg-go/v3`、`gvisor`（AmneziaWG 用户态网络栈）。
- 运营设施：`robfig/cron/v3`（定时任务）、`shirou/gopsutil/v4`（系统监控）、`mymmrac/telego`（Telegram）、`go-ldap/ldap`（LDAP 同步）、`skip2/go-qrcode`、`xlzd/gotp`（TOTP 两步验证）。

**前端（`frontend/package.json`）**
- React 19 + Ant Design 6 + Vite 8 + TypeScript，Node >= 26；路由 react-router 8，数据层 TanStack Query 5，表单 react-hook-form + zod 4。
- 组件库外围：CodeMirror 6（JSON 编辑）、uplot（流量图表）、swagger-ui-react（面板内 API 文档）、otpauth、i18next、persian-calendar-suite（波斯历，作者背景相关）。
- 质量工具：oxlint/oxfmt（Rust 系 lint/格式化）、Storybook 10、Vitest 5（含 browser-playwright 模式）、MSW（接口 mock）。

**工具链**：golangci-lint（`.golangci.yml`）、govulncheck、CodeQL、mutation testing（`.github/workflows/mutation.yml`）、husky + lint-staged（`frontend/.husky`）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 | 出处 |
|---|---|---|
| `main.go` | 入口 + `x-ui` CLI 子命令（run/migrate/migrate-db/encrypt-tokens/setting/cert/install-awg-kernel） | `main.go` |
| `internal/web/` | Gin 服务器总装（624 文件）：`controller/` REST API、`service/` 业务逻辑、`job/` 19+ 个 cron 任务、`middleware/`、`runtime/`（master/node mTLS 通道）、`session/`（CSRF）、`websocket/`、`translation/`（14 个语言包） | `internal/web/web.go`、`CLAUDE.md` |
| `internal/xray/` | xray-core 子进程生命周期、配置生成、gRPC API 调用、热更新 diff、geodata 流式解析 | `internal/xray/` |
| `internal/database/` | GORM 初始化、AutoMigrate + 手写迁移、SQLite dump/restore、SQLite→Postgres 数据迁移 | `internal/database/db.go`、`migrate_data.go` |
| `internal/tuic/`、`internal/amneziawg{,net,kern}/`、`internal/mtproto/`、`internal/pia/` | 内置协议栈：TUIC v5 原生 QUIC 服务、AmneziaWG（gVisor 用户态 netstack）、MTProto（mtg-multi sidecar 进程）、PIA WG 客户端 | `CLAUDE.md` repo map |
| `internal/sub/` | 订阅服务器：links 生成、Clash/JSON 输出、模板、占位符变量 | `internal/sub/` |
| `internal/config/`、`util/`、`eventbus/`、`logger/`、`crypto/`、`tunnelmonitor/` | 环境配置、工具、进程内 pub/sub（xray.crash、cpu.high 等事件）、日志、令牌加密、隧道健康探测 | `CLAUDE.md` |
| `frontend/` | React SPA（src 下 pages 296、test 286、components 101、schemas 86 个文件），构建产物输出到 `internal/web/dist` 被 embed | `frontend/package.json` |
| `docs/` | Next.js/Fumadocs 文档站（独立 CI `docs-ci.yml`，维护第三份 share-link 生成实现） | `CLAUDE.md` |
| `deploy/`、`tools/`、`media/` | cloud-init/云市场部署、openapigen 代码生成器、截图 | 目录清单 |
| `x-ui.sh`、`install.sh`、`update.sh`、`Dockerfile`、`docker-compose.yml` | Linux 管理菜单脚本、一键安装/升级、容器化 | 顶层文件 |

组织方式：**单仓库单二进制**的后端（前端构建产物 embed 进 Go），加一个同仓的前端 workspace 与独立文档站。不是 monorepo 多服务，而是一个自包含面板程序 + 周边配套。后端严格分层（controller → runtime → service → xray/db），并有契约测试固化（见第 7 节）。

## 5. 核心架构与关键实现

### 5.1 Xray 子进程托管与热更新（`internal/xray/`）

面板不重新实现代理协议，而是托管 xray-core 子进程：`process.go` 负责启动/停止/版本管理，`config.go` 生成完整 Xray JSON，`api.go` 通过 gRPC 调用 Xray 的 handler/stats API。关键的工程价值在 `hot_diff.go`：对"旧配置 vs 新配置"做结构化 diff，只对运行中实例下发必要的 gRPC 变更，避免重启进程断开现有连接。

```go
// internal/xray/hot_diff.go
type HotDiff struct {
    TuicRelayChanges   []InboundReplacement
    RemovedInboundTags []string
    AddedInbounds      [][]byte
    RemovedUsers       []UserOp
    AddedUsers         []UserOp
    DroppedClients     []UserOp
    RemovedOutboundTags []string
    AddedOutbounds     [][]byte
    RoutingConfig      []byte // full new routing section; nil when unchanged
}
```

变更粒度细到单用户（`UserOp` 即 Xray 的 `AlterInbound` 操作）。信号面：SIGHUP 重启面板 HTTP 服务、SIGUSR1 只重启 xray-core（`main.go` runWebServer）。

### 5.2 多节点主从架构（`internal/web/runtime/`）

README 与 Makefile 中的 `node-e2e` 目标（"master + node sync end to end with two real panel processes"）表明这是头等功能。状态变更必须经由 `runtime.Runtime` 分发，`CLAUDE.md` 把它列为硬规则："A direct call passes every local test and silently breaks every multi-node deployment"。安全设计配套：node 间凭据是可加密的 bearer token（`main.go` `initNodeTokenCrypto`，key 来自文件或环境变量，token 只存哈希），配套 `node_heartbeat_job.go`、`node_traffic_sync_job.go` 两个 cron 任务与 `internal/crypto/nodetoken/` 编解码器；入站模型里有 `NodeID`/`OriginNodeGuid` 字段支撑跨节点溯源（`internal/database/model/model.go`）。

### 5.3 数据模型与双后端存储（`internal/database/`）

核心模型约 24 个（`CLAUDE.md`）：`User`（面板管理员，首启随机生成密码且只打印一次，`main.go` `EnsureFirstUser`）、`Inbound`（监听/协议/配置 JSON 字符串 + 流量/到期/重置计划字段）、`ClientTraffic`（每客户端流量统计）、`Setting`（KV 设置）、`OutboundTraffics`、`InboundClientIps`（IP 限制）等。SQLite 为默认，PostgreSQL 为可选；`migrate-db --dsn` 子命令做在线迁移，另支持 SQL 文本 dump/restore（`main.go` migrateDbCmd、`internal/database/dump_sqlite.go`、`migrate_data.go`）。

### 5.4 协议内建栈与 Go→前端代码生成

两个值得注意的实现方向：
- **非 Xray 协议的面板内实现**：TUIC v5 不是塞进 Xray 配置，而是面板进程内的原生 QUIC 服务器（`internal/tuic/server.go`，含 UDP associations、socks 桥接、relay 恢复/关闭等 15 个文件）；AmneziaWG 跑在 gVisor 用户态 netstack 上再转入回环 SOCKS5 给 Xray（`internal/amneziawgnet/`，45 文件），免内核模块；MTProto 则托管独立的 mtg-multi 二进制 sidecar，通过其管理 API 热应用 secret/配额变更（`CLAUDE.md`、`internal/mtproto/`）。
- **Go→TS 单向代码生成**：`tools/openapigen` 从 Go 结构体生成 `frontend/src/generated/` 的类型与 Zod schema 及 OpenAPI 文档，前端请求校验直接复用后端定义；新增路由必须同步 `endpoints.ts`，由 `TestRouteRegistryContract`（`internal/web/routes_contract_test.go`）双向钉死，漏掉则 `make test-go` 失败（`CLAUDE.md` Hard rules）。

## 6. 构建与运行

来自 `Makefile` 与 `README.md` 的实际命令（不运行，仅摘录）：

```bash
# 前端构建到 internal/web/dist，再编 Go 二进制（CGO 必须开启，SQLite 驱动需要 C 编译器）
make build                 # = cd frontend && npm run build && go build ./...
make verify                # 本地 PR 门槛：gen-check/lint/format/typecheck/test/build/storybook/docs 检查
make test-go               # go test -shuffle=on -count=1 ./...
make race                  # -race 全量（internal/web/service 放宽超时到 25m）
make node-e2e              # 双真实面板进程的 master/node 端到端
cd frontend && npm test    # vitest

# 运行与运维
x-ui run                   # 启动面板（默认子命令）
x-ui migrate-db --dsn "postgres://..."   # SQLite → PostgreSQL
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)  # 一键安装
docker compose --profile postgres up -d  # 带内建 PostgreSQL 的容器化
```

发布资产每个都附 `.sha256` 校验和，install.sh 与升级逻辑会先验签再落地（`README.md`）。支持 Linux 多发行版与 Windows，架构 amd64/386/arm64/armv7/armv6/armv5/s390x（`README.md` Supported Platforms）。

## 7. 测试与工程质量

- **测试规模**：Go 测试文件 658 个（tarball 统计，含 40+ 个数据库迁移回归测试，如 `internal/database/*_migration_test.go`）；前端 `src/test/` 下 286 个文件（`frontend/src/test/`）。使用 `pgregory.net/rapid` 做属性测试（`go.mod`）。
- **CI（`.github/workflows/ci.yml` 等 10 个 workflow）**：Go 测试矩阵含真实 PostgreSQL 16 service（`postgres-durable-first` job）；另有 CodeQL（`codeql.yml`）、mutation 测试（`mutation.yml`）、govulncheck（Makefile `vulncheck`）、依赖审计（`npm audit --omit=dev`，独立于主门槛建模为"会漂移的检查"）、文档站独立 CI、缓存清理、以及两个 AI 辅助流程（`claude-pr-review.yml`、`claude-issue-analyst.yml`）。
- **契约化防回归**：路由↔API 文档契约（`routes_contract_test.go`）、i18n 死键测试（新增英文 key 必须同步 14 个语言包且被引用，`frontend/src/test/i18n-dead-keys.test.ts`）、OpenAPI 新鲜度三处对齐（frontend/public、docs/public、MDX）。
- **代码质量观察**：`CLAUDE.md` 硬规则要求"正确修复优先于小修复"、注释每块 ≤2 行、Conventional Commits；`.golangci.yml` + oxlint + oxfmt + tsc --noEmit 全量启用。整体工程纪律在同类面板项目（多为单人快速迭代风格）中显著偏高。

## 8. 社区与维护状态

- **活跃度**：最近 push 2026-11-03（gh api），tarball 快照与最新代码几乎同步，属高强度持续开发。
- **release 节奏**：约每月一版：v3.7.0（08-24）→ v3.8.0（09-14）→ v3.8.5（09-16）→ v3.9.0（10-03）→ v3.10.0/10.2（10-12/14）→ v3.11.0（10-25），另有按 commit 产出的 dev-latest 预发布（gh api releases）。
- **贡献者**：contributors API 首页即满 100 人；12,638 forks、247 open issues（gh api，2026-11-04）。
- **文档与本地化**：7 种语言 README、面板 14 种 UI 语言、独立文档站 + 面板内 swagger（`README.md` 顶部语言条、`internal/web/translation/`）。
- 维护者结构未做逐一统计（未确认），但从 `.github/claude/` 配置与 CLAUDE.md 的存在看，仓库采用 AI 辅助的 issue 分析与 PR review 流程。

## 9. 亮点与风险

**亮点**
1. 工程成熟度远超同类面板：658+286 个测试文件、契约测试钉死跨层一致性、mutation/CodeQL/vulncheck 多层质量网（第 7 节所引各处）。
2. 热更新设计：配置变更通过 Xray gRPC API 增量下发而非重启进程，`hot_diff.go` 是理解面板/Xray 边界的核心文件。
3. 协议矩阵完整且实现路径清晰：Xray 内协议走配置生成，非 Xray 协议（TUIC/AmneziaWG/MTProto/PIA）给出自建栈或 sidecar 两种集成范式。
4. 双数据库 + 迁移工具链（SQLite↔Postgres↔dump），且迁移代码配了成体系的回归测试（`internal/database/`）。
5. Go→前端 codegen 消除双端类型漂移，OpenAPI 三处同步由测试强制（`tools/openapigen/`、`CLAUDE.md`）。

**风险 / 局限**
1. **合规与法律风险（客观陈述）**：本项目是网络代理管理面板，在部分司法辖区部署或运营此类工具可能违反当地法律；README 亦自我声明仅供个人使用、勿用于非法或生产环境（`README.md` IMPORTANT 段）。此类仓库的安装脚本在本机落盘时可能被安全软件（如 Windows Defender）拦截，属该品类常见现象。
2. **攻击面**：面板默认暴露 HTTP 管理端口于公网，天然是扫描与爆破目标；代码可见的缓解包括登录限速（`internal/web/controller/login_limiter.go`）、API token 仅存哈希（`main.go` GetApiToken 注释）、CSRF 会话（`internal/web/session/`）、mTLS 节点通道；历史 CVE 情况未逐一核实（未确认）。
3. **供应链与安装模式**：`curl | bash` 一键安装 + 安装期拉取 mtg-multi 预编译二进制（`CLAUDE.md`、`DockerInit.sh`），虽然 release 带 sha256 校验，但属于信任链较弱的分发模式。
4. **构建约束**：CGO 强依赖（mattn/go-sqlite3），`CGO_ENABLED=0` 不可用（`CLAUDE.md`）；Go 1.27/Node 26 等版本门槛高，二次开发需较新工具链。

## 10. 总结评价

3X-UI 是目前 Xray 生态中完成度最高的自托管管理面板之一：它把"生成一份 Xray JSON"扩展成了带计费、限速、订阅分发、多节点同步和机器人运维的完整小型运营系统，同时用一套明显高于社区平均水准的工程纪律（契约测试、迁移回归、多语言 CI、代码生成）维持着每月一版的快节奏演进。对想研究"面板如何托管代理内核"的人，`internal/xray/`（进程管理 + 热更新 diff）与 `internal/web/runtime/`（主从通道）是最有阅读价值的两块；对想自建代理管理面板的运维者，它是功能覆盖与文档完备度的标杆；对构建 Go+React 全栈单二进制应用的开发者，其 go:embed 前端、Go→TS codegen 与测试组织方式也值得参考。需要注意的只是它的品类属性：公网管理面 + 代理工具带来的合规与安全暴露面，使用者需自行评估所处环境的法律约束。
