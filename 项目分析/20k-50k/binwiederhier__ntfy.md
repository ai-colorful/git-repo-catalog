# binwiederhier/ntfy 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Go｜档位 20k-50k｜分析日期 2026-11-21（gh api 实时 35.7k stars，pushed 2026-11-20）

## 1. 项目速览

一句话定位：用一条 `curl -d "消息" ntfy.sh/mytopic` 就能把推送发到手机/桌面的 HTTP pub-sub 通知服务，自托管友好。

| 事实 | 值 | 出处 |
|---|---|---|
| 名称 | ntfy (heckel.io/ntfy/v2) | `go.mod` |
| 星数 | 39500（快照）；35668（2026-11-21 gh api） | 任务参数 / gh api |
| 主语言 | Go（go 1.26），300 个 .go 文件 | `go.mod`、目录统计 |
| 许可证 | Apache-2.0 / GPLv2 双许可 | `LICENSE`、`LICENSE.GPLv2`、README.md:344 |
| 最新 release | v2.38.1（2026-11-16），一个月内连发 6 个版本 | gh api releases、`docs/releases.md` |
| 最近活跃 | pushed 2026-11-20，open issues 388，贡献者 447 | gh api |
| 客户端 | Android/iOS 独立仓库 + React Web app + Go CLI | README、`web/`、`cmd/` |

## 2. 项目是什么，解决什么问题

ntfy 解决的是一个极常见又一直没被很好满足的需求：从脚本、CI、cron、IoT 设备往手机发一条通知。传统方案要么要注册账号写 SDK（FCM/APNs），要么依赖 IM 机器人（Slack/Telegram bot）。ntfy 把"发布一条通知"简化成一次任意语言都能发起的 HTTP PUT/POST——URL 路径就是 topic，请求体就是消息（`docs/publish.md`，全文 5002 行 API 文档）。订阅端则是安卓/iOS/Web/CLI 多端 app，长连接收流。

服务端是纯 Go 单二进制：内嵌 Web 管理界面、docs 静态页（`Makefile` 的 `cli-deps-static-sites` 把 web/docs 打进二进制）、消息缓存、附件存储、用户与计费系统，`docker-compose.yml` 一行即可自托管。免费公共实例 ntfy.sh 与自托管共用同一份代码。

商业模型是 ntfy Pro：不提供代码外的主要差别是 ntfy.sh 上的账号配额（消息量、消息保留时长、附件带宽、topic 预留数、电话/邮件转发额度），全部抽象为 `user.Tier` 结构（`user/types.go:106`），计费走 Stripe（`server/server_payments.go`），免费匿名用户就是 `visitors.ipLimits` 的 IP 限流（`server/server_payments.go:66`）。整个 Pro 层可用 `nopayments` build tag 编译剥离（`payments/payments.go` 头部注释）。

一个有趣的架构约束是 iOS：APNs 不允许任意服务器推送，ntfy 的做法是 iOS app 只连 ntfy.sh 做轮询，自托管实例配置 `upstream-base-url: "https://ntfy.sh"` 把消息哈希转发给官方服务，由官方代为唤醒（`docs/config.md:1653`、`server/server.go` 的 `forwardPollRequest`）。Android 则只在 Google Play 版对 ntfy.sh 主机启用 FCM，F-Droid 版纯自托管连接（`docs/subscribe/phone.md:101`）。

## 3. 技术栈与依赖

- **服务端**：Go 1.26，标准库 net/http 单 mux；无 web 框架（`server/server.go`）
- **存储**：SQLite（`mattn/go-sqlite3`）默认，PostgreSQL（`jackc/pgx/v5`）可选，消息/用户/WebPush 订阅/附件各有双实现（`message/`、`user/`、`webpush/`、`attachment/`）
- **订阅协议**：`gorilla/websocket`（WS）；SSE/JSON stream/raw 原生 HTTP 流
- **推送通道**：`firebase.google.com/go/v4`（FCM，安卓唤醒）、`SherClockHolmes/webpush-go`（浏览器 Web Push）、`emersion/go-smtp`（内嵌 SMTP 服务器，邮件发通知）、自研 Twilio 电话客户端（`twilio/`）
- **计费**：`stripe/stripe-go/v74`
- **CLI**：`urfave/cli/v2`；HTML 净化 `bluemonday`；Prometheus 指标 `prometheus/client_golang`
- **Web 前端**：React 19 + MUI + Vite + Dexie(IndexedDB) + i18next，Vitest 单测 + Playwright e2e（`web/package.json`）
- **文档**：mkdocs（`mkdocs.yml`、`docs/`）

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `server/` | HTTP 服务核心：路由、发布/订阅/鉴权/限流/计费/集群/Firebird 转发，61 文件 27 测试 |
| `message/` | 消息缓存抽象 + SQLite/Postgres 双实现与 schema 迁移 |
| `topic/` | UnifiedPush topic 集群态（新近重构），server 内 `server/topic.go` 则是订阅者注册表 |
| `cluster/` | v2.38 新增多节点集群：PG 注册表控制面 + HTTP NDJSON 数据面（`cluster/cluster.go`） |
| `client/` | Go 客户端库（发布/订阅/`client.yml` 配置文件模式） |
| `cmd/` | CLI 命令：serve/publish/subscribe/user/tier/token/access/webpush 等 31 文件 |
| `user/` | 用户、Token、ACL、Tier（SQLite/Postgres 双实现） |
| `visitor/` | 基于 IP/user 的限流与带宽计量（`visitor/limits.go`） |
| `attachment/`、`s3/` | 附件存储抽象：本地文件系统 / 自研 S3 客户端 |
| `webpush/`、`web/` | 浏览器 Web Push 订阅存储；React PWA 前端 |
| `payments/`、`mail/`、`twilio/`、`template/`、`ban/`、`metrics/`、`db/` | Stripe 封装、邮件格式化、电话呼叫、消息模板、IP ban、指标、DB 连接层 |
| `docs/` | mkdocs 全套文档（publish.md 5002 行） |
| `test/` | Go 端测试辅助工具包 |

组织方式是"单 Go module 多领域包"：每个领域一个顶层包 + 按后端分文件（`cache_sqlite.go`/`cache_postgres.go`），可选依赖（Firebase/Stripe）用 build tag dummy 文件隔离（`server_firebase_dummy.go`、`server_payments_dummy.go`），使社区自托管构建可以裁剪云依赖。前端与 Go 代码同仓但独立构建，最终静态产物嵌入二进制——标准的服务端 monorepo。

## 5. 核心架构与关键实现

### 5.1 极简 API：一个 URL 就是全部

整个服务只有一个 `mux.HandleFunc("/", s.handle)`，进入 900 行的 if/else 路由链（`server/server.go:769`）。发布端点的设计哲学是把 REST 缩到极致：

```go
} else if (r.Method == http.MethodPut || r.Method == http.MethodPost) && r.URL.Path == "/" {
    return s.transformBodyJSON(s.limitRequestsWithTopic(s.authorizeTopicWrite(s.handlePublish)))(w, r, v)
} else if (r.Method == http.MethodPut || r.Method == http.MethodPost) && (topicPathRegex.MatchString(r.URL.Path) || updatePathRegex.MatchString(r.URL.Path)) {
    return s.limitRequestsWithTopic(s.authorizeTopicWrite(s.handlePublish))(w, r, v)
```
（`server/server.go:897-903`）

`PUT/POST /mytopic` 即发布，`GET /mytopic` 返回 Web 界面；发布参数全部走 Header（X-Title、X-Priority、X-Tags…）或 JSON body（`transformBodyJSON` 包装 `/` 根路径），HTTP HEAD 也被当消息体支持。鉴权、限流、Topic 授权都以函数装饰器叠加，中间件顺序即安全边界（`server/server_middleware.go`）。

### 5.2 订阅：同一 topic 的四种流编码

订阅统一落到 `handleSubscribeHTTP` 骨架，只换编码器：JSON stream（`/json`）、SSE（`/sse`）、raw（`/raw`）、WebSocket（`/ws`）（`server/server.go:1653-1813`）。内存订阅者注册表在 `server/topic.go`：每个 topic 一个 `topic` 结构，`Subscribe` 返回 subscriberID，`Publish` 向所有订阅者 channel 投递，`deliver` 支持从缓存回放 `since`/`scheduled` 消息，`ExpireDelivered` 定期清理已确认投递记录（`server/topic.go:67-209`）。

### 5.3 消息存储：SQLite/Postgres 双后端 + 定时消息

消息缓存抽象 `message/cache.go`（806 行）之下是双实现。SQLite 版用单表 24 列覆盖消息与附件元数据，`published` 标志把"定时消息"（`when` 延迟发布）与已发布消息同一张表管理：

```go
sqliteInsertMessagesQuery = `INSERT INTO messages (mid, sequence_id, time, event, expires,
    topic, message, title, priority, tags, click, icon, actions, attachment_name, ...,
    published) VALUES ...`
sqliteSelectMessagesSinceTimeIncludeScheduledQuery = `... WHERE topic = ? AND time >= ? ...`
```
（`message/cache_sqlite.go:21-31`、`message/cache_sqlite_schema.go:13-49`）

索引设计对应三大查询路径：`(topic, published, time)` 回放、`expires` 过期清理、`sender/user` 配额审计。v2.38 时代的批插入（一次最多 1000 行）和 `json_each` 补读优化都是为了集群下的高吞吐（`message/cache_sqlite.go:14-18` 注释）。

### 5.4 扇出 dispatch 与集群

一条消息发布后的扇出集中在一个函数：先 fire-and-forget 转发给集群节点，再本地 topic.Publish，随后按配置异步投递 FCM/Web Push/邮件/电话/上游转发（`server/server.go:1069-1104`）。集群是 v2.38 的重头：PostgreSQL 仅作节点注册表（控制面），消息在节点间以 NDJSON 直接 HTTP 对推（数据面），单节点默认 `cluster_nop.go` 零开销（`cluster/cluster.go:1-25`）。

## 6. 构建与运行

```bash
# 服务端（需先构建内嵌前端与文档）
make web docs cli            # Makefile:94
make cli-linux-server        # 静态编译各平台，Makefile:215

# 直接跑
go run main.go serve         # 或编译后 ./ntfy serve
ntfy publish mytopic "hi"    # CLI 发布（cmd/publish.go）

# 测试
make test                    # .github/workflows/test.yaml 分 checks/server/rest 三片并行
cd web && npm run test:e2e   # Playwright e2e
```
Docker：`docker-compose.yml` 提供 binwiederhier/ntfy 镜像；系统包见 `server/ntfy.service`（systemd 单元）。

## 7. 测试与工程质量

- Go 端 300 文件中 120 个 `*_test.go`（40%），核心 server 包 61 文件配 27 测试文件，账号/计费/集群各有 20-65KB 的大体量测试（`server/server_account_test.go`、`server_admin_test.go`）
- CI（`.github/workflows/test.yaml`）：测试矩阵按耗时把 server 包单独分片（注释自述"server 包耗时约等于其他全部之和"），起 Postgres 17 service、注入 S3 测试密钥，且要求先 `make docs && make web`——因为集成测试会请求内嵌静态页
- pin 版本严格：`go.mod` 顶部注释 pin `go-smtp v0.17.0` 因上游破坏性变更；workflow 中所有 action 用完整 commit SHA pin
- `docs/releases.md` 的 v2.38.1 是安全响应版本：修复模板 DoS、S3 慢上传拖垮 goroutine、Web Push 订阅按 IP 前缀限流——维护者对多租户滥用面的关注一以贯之（对应 `ban/`、`visitor/` 的存在）
- codecov + Go Report Card 徽章（README 头部），Web 端 eslint/prettier/vitest/playwright 全套

## 8. 社区与维护状态

- 维护者结构：Philipp Heckel 单人主导的"独裁式"项目，447 位贡献者中绝大多数是文档/小修；核心代码集中度高（`gh api contributors`、server 包风格统一的自注释长文件）
- 发版节奏极快：2026-10-31 至 11-16 连发 v2.36.0 → v2.37.0/1/2 → v2.38.0 → v2.38.1 六版（gh api releases），大功能（集群、Postgres）与安全修复穿插
- 活跃度：pushed 2026-11-20（分析前一天），open issues 388——单维护者消化 issue 的压力可见
- 生态：ntfy-android / ntfy-ios 独立仓库；Discord/Matrix 社区；ntfy.sh 公共实例 + Pro 订阅（$5/月起，README）支撑开发
- 商业化证据：`server_payments.go` 25KB、Stripe webhook、Tier 管理 API 全在主仓，仅密钥在运行时注入

## 9. 亮点与风险

**亮点**
1. API 设计教科书级：URL 即 topic、body 即消息、Header 即参数，零 SDK、零注册即可用（`docs/publish.md`）
2. 部署光谱完整：单二进制 SQLite 单机 → Postgres 多节点集群（`cluster/`），同一份代码从树莓派到生产
3. 可选云依赖全部 build tag 隔离，自托管版本零 FCM/Stripe 依赖也能编译（`server_firebase_dummy.go` 等）
4. 订阅协议四编码一套骨架，客户端矩阵（安卓/iOS/Web PWA/CLI/SMTP/电话）覆盖极端全面
5. 文档即产品：`docs/` 30+ 篇深度文档内嵌进二进制，随版本一起发布

**风险/局限**
1. 单维护者 + 极快发版：388 open issues，总线因素明显
2. `server/server.go` 2479 行 99KB 单文件 if/else 路由链，新人可读性差（对比 `topic/`、`message/` 的新包已明显改善）
3. iOS 架构依赖 ntfy.sh 上游转发（`upstream-base-url`），自托管 iOS 体验与官方服务强耦合
4. v2.38.1 安全公告表明消息模板、附件上传、Web Push 均曾是 DoS 面——公网多租户运营需持续紧跟版本

## 10. 总结评价

ntfy 是"一个人 + 一条 curl"重塑推送通知的典范：把 FCM/APNs 的复杂度收进服务端，把用户侧 API 压缩到 HTTP 路径级别，再用完善的多端客户端和自托管能力把开发者、homelab 用户和公共服务三方统一起来。它是 Go 后端工程的优秀读物——单文件路由的取舍、双数据库后端的抽象、build tag 裁剪云依赖、内存订阅者注册表 + 持久化缓存的双轨设计，都有真实生产规模的答案。适合想给项目加通知的运维/后端工程师直接用，也适合想学"小 API 大工程"Go 服务设计的开发者精读 `server/server.go` 与 `message/`。
