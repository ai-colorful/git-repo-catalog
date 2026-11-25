# louislam/uptime-kuma 项目分析

> 星数 92792（快照 2026-10-24）｜主语言 JavaScript｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：Uptime Kuma 是一个易用的自托管监控（Uptime Monitoring）工具，提供花哨的仪表盘、20+ 种探测方式和 100+ 个通知渠道，常被视为 Uptime Robot 的开源自托管替代品。

| 项目 | 内容 |
| --- | --- |
| 名称 | uptime-kuma |
| 星数 | 92,792（快照 2026-10-24） |
| 主语言 | JavaScript（Vue 3 前端 + Node.js 后端） |
| 许可证 | MIT（`package.json`） |
| 当前版本 | 3.0.0-beta.0（master HEAD，`package.json`）；最新稳定版 2.5.8（GitHub Releases，2026-10-26 发布） |
| 最近活跃 | 最近 push 2026-10-30（gh api `repos/louislam/uptime-kuma`） |
| 代码规模 | server/ + src/ 约 454 个 js/ts/vue 文件、约 6.9 万行（wc 统计） |

## 2. 项目是什么，解决什么问题

Uptime Kuma 解决的是"我的服务到底活没活着"这一持续性观测问题。用户部署一个实例后，可以添加任意数量的"监控器"（Monitor），由服务端按设定的间隔（最短 20 秒）周期性发起探测——HTTP(s)、TCP 端口、Ping、DNS 记录、关键字匹配、JSON Query、WebSocket 握手、推送（Push）、Steam/游戏服务器、Docker 容器、各种数据库连接等（`README.md` Features 一节，`server/monitor-types/` 下 27 种类型实现）。每次探测结果记录为心跳（Heartbeat），用于计算可用率、绘制 ping 曲线、驱动状态页。

与传统 SaaS 监控（Uptime Robot 等）相比，它的差异化在于：数据完全自托管（SQLite/内嵌 MariaDB 单文件起步）、探测频率不受免费套餐限制、UI 精美且支持多语言（`src/lang/` 下 81 种语言文件）、可自定义多个状态页并绑定独立域名。它同时支持 Prometheus 指标暴露（`server/prometheus.js`）与 API Key（`server/routers/api-router.js`），能嵌入现有监控体系。

通知体系是其生态护城河：Telegram、Discord、Slack、飞书、钉钉、Server 酱、WxPusher 等 111 个通知 provider（`server/notification-providers/` 文件数），覆盖即时通讯、短信、邮件、Webhook、事件管理（PagerDuty/Opsgenie）等类别，且支持通过 Liquid 模板自定义消息内容（`server/notification-providers/notification-provider.js` 的 `renderTemplate`）。

项目采用单实例架构（非分布式），强调"部署即用"：一条 `docker compose up -d` 即可运行（`compose.yaml`）。v3（当前 master）正在引入多数据库支持（SQLite / MariaDB / PostgreSQL 等，`server/database.js`）、better-auth 认证（`server/better-auth.ts`）、MCP server（`server/routers/mcp-router.ts`）等能力。

## 3. 技术栈与依赖

- 前端：Vue 3（`~3.5.39`）+ Vue Router + vue-i18n + Bootstrap 5.1.3 + Chart.js（ping 图）+ Vite 8 构建（`package.json` devDependencies）。
- 后端：Node.js（engines 要求 `>= 26.2.0`）+ Express 4 + Socket.IO 4（WebSocket 实时推送心跳）、`tsx` 运行 TS 文件（`server/uptime-kuma-server.ts` 等混用 TS/JS）。
- ORM/数据库：`redbean-node`（活跃记录风格 ORM）+ `better-sqlite3`（默认库，`server/database.js:335`）；v3 通过 knex 增加多数据库支持，依赖中出现 mysql2/pg/mssql/oracledb/mongodb，并内置 EmbeddedMariaDB（`server/embedded-mariadb.js`）。
- 关键库：`croner`（后台 cron 任务）、`axios`（HTTP 探测）、`websocket`/`ws`、`@grpc/grpc-js`、`net-snmp`、`mqtt`、`kafkajs`、`gamedig`（游戏服）、`globalping`（全球分布式探测）、`playwright-core`（Real Browser 监控）、`liquidjs`（通知模板）、`prom-client`（Prometheus 指标）、`better-auth` + `@better-auth/api-key`（v3 认证）、`zod` + `zod-openapi`（v3 API 校验与 OpenAPI）。
- 运行环境：官方提供 Docker 镜像（`docker/dockerfile`，基于 debian-base 三阶段构建）与非 Docker 安装脚本。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `server/` | Node 后端：Express + Socket.IO 服务、监控模型与调度、通知、数据库、认证、API 路由 |
| `server/monitor-types/` | 27 种监控类型的探测实现（每种一个文件，继承 `MonitorType`） |
| `server/notification-providers/` | 111 个通知渠道实现（每种一个文件，继承 `NotificationProvider`） |
| `server/model/` | 数据模型（`monitor.js` 2204 行是全项目核心） |
| `server/socket-handlers/` | Socket.IO 事件处理（状态页、Docker、维护窗口等） |
| `server/routers/` | HTTP API 路由（api / better-auth / mcp / status-page） |
| `db/` | 数据库模板与迁移脚本（`knex_migrations/`、`old_migrations/`） |
| `src/` | Vue 3 前端源码（165 个 .vue 组件，pages/components/layouts） |
| `src/components/notifications/` | 每个通知渠道对应的前端表单组件 |
| `extra/` | 发布、迁移、演示部署等工程脚本（release、healthcheck、reset-password 等） |
| `test/` | 后端测试（83 个文件）、Playwright e2e（19 个文件）、手工测试脚手架 |
| `.github/workflows/` | 22 个 CI 工作流 |
| `docker/` | Dockerfile 与 compose 开发环境 |

组织方式评述：前后端同仓单包（`CONTRIBUTING.md` 明确"前后端共用同一个 package.json"），后端按"模型 + 类型插件 + 通知插件 + socket/HTTP 处理器"分文件组织，扩展点（监控类型、通知渠道）都走"基类 + 每渠道一文件 + 注册表"的模式，第三方贡献新渠道的成本极低，这是通知生态能膨胀到 111 个的直接原因。缺点是 `server/model/monitor.js` 单文件 2200 行，HTTP/keyword/json-query 探测逻辑内联在 beat 循环中而非独立类型文件（见第 5 章）。

## 5. 核心架构与关键实现

### 5.1 监控调度：每 Monitor 独立的 setTimeout 自循环

调度核心在 `server/model/monitor.js` 的 `start()`（478 行起）。每个监控器由独立的异步 `beat()` 闭包驱动：执行探测 → 写心跳 → 推送前端 → 计算下次间隔 → `setTimeout(safeBeat)` 自我调度，而非集中式定时器池。关键片段（`server/model/monitor.js` 约 1094-1105 行）：

```js
if (!this.isStop) {
    let intervalRemainingMs = Math.max(1,
        beatInterval * 1000 - dayjs().diff(dayjs.utc(bean.time)));
    this.heartbeatInterval = setTimeout(safeBeat, intervalRemainingMs);
}
```

要点：
- 下一次探测时间扣除本次探测耗时，保证节奏不漂移；`safeBeat` 捕获异常后按 interval 重试，防止单次崩溃杀死调度循环。
- 间隔随状态自适应：PENDING（重试中）用 `retryInterval`，DOWN 期间用 `retryInterval` 计数直到 `maxretries` 次后转 DOWN 间隔（`monitor.js` 960-990 行）。
- DOWN 状态下按 `resendInterval` 周期性重发通知（`monitor.js` 1013-1028 行）。
- 服务启动时通过 `server/start-monitors.ts` 的 `firstBeatDelayMs()` 把每个监控器的首次探测随机打散到整个 interval 内，避免重启后同一秒内并发探测所有目标（`start-monitors.ts:18-24`）。
- Push 类型是反向的：由外部客户端向 `/api/push` 推送，服务端超时未收到即判 DOWN（`monitor.js` 529-537 行的 push 特判与 `server/routers/api-router.js`）。

### 5.2 监控类型扩展：基类 + 注册表

所有非 HTTP 监控类型继承 `server/monitor-types/monitor-type.js` 的 `MonitorType` 基类，只需覆写一个方法：

```js
async check(monitor, heartbeat, server) {
    throw new Error("You need to override check()");
}
```

类上通过 `supportsConditions`、`conditionVariables`、`allowCustomStatus` 三个声明式属性控制前端表单行为（监控条件、变量下拉）。类型实例在 `server/uptime-kuma-server.ts:107-133` 一次性注册到静态 `monitorTypeList`，前端 `EditMonitor.vue` 据此渲染类型选择。每种类型一个文件（如 `server/monitor-types/snmp.js`、`mqtt.js`、`grpc.js`），新增类型零侵入。值得注意的是：http/keyword/json-query 三种最常用类型的探测逻辑直接写在 `Monitor.beat()` 内（`monitor.js` 546-560 行 `this.type === "http" || "keyword" || "json-query"` 分支），并未拆成独立类型文件——历史原因造成的一致性破口。

### 5.3 通知体系：111 个 provider 的插件化派发

`server/notification.js` 维护 `providerList` 注册表：`Notification.init()` 实例化 111 个 provider 并做重名/无名校验（`notification.js:121-243`）。所有 provider 继承 `server/notification-providers/notification-provider.js` 的 `NotificationProvider`，公共能力已上提：

- `extractAddress()` 按监控类型从 monitorJSON 提取目标地址（hostname/url）；
- `renderTemplate()` 用 liquidjs 渲染用户自定义消息模板，提供 `NAME`、`STATUS`、`HOSTNAME_OR_URL` 等上下文变量；
- 代理支持（http/https/socks agent）。

派发链路：`Monitor.sendNotification()`（`monitor.js` 静态方法）按 monitor 关联的通知配置逐个调用 `Notification.send()` → 按 `notification.type` 路由到 `providerList[type].send()`。通知与监控是多对多关系，还支持"应用到所有监控器"（`notification.js:301` 的 `applyNotificationEveryMonitor`）。前端对应 165 个组件中约 111 个是渠道配置表单（`src/components/notifications/`），前后端一一对应。

### 5.4 可用率计算：内存 LimitQueue 分级聚合

`server/uptime-calculator.js` 为每个监控器维护一个 `UptimeCalculator` 单例：分钟粒度环形队列存 24 小时、小时粒度存 30 天（`minutelyUptimeDataList = new LimitQueue(24*60)`），每次心跳到达时增量更新分区统计，状态页与徽章的 24h/30d 可用率直接查内存队列而非扫历史表。配套 `server/monitor-stats.ts`（分组统计缓存，Prometheus 更新"从分组统计缓存取，从不取全量历史"，`monitor.js:1101` 注释）与每天凌晨清理旧数据、每 5 分钟增量 VACUUM 的后台任务（`server/jobs.js`，croner 调度）。

## 6. 构建与运行

来自 `package.json` scripts 与 `README.md`：

```bash
# Docker 一键运行（官方推荐）
docker run -d --restart=always -p 3001:3001 -v uptime-kuma:/app/data --name uptime-kuma louislam/uptime-kuma:2

# 开发（前端 vite :3000 + 后端 tsx，concurrently 并行）
npm run dev

# 生产构建
npm run build          # vite build 输出到 dist/
npm run start          # node --run start-server（node --import=tsx server/server.js）

# 代码检查与格式化
npm run lint           # eslint + stylelint
npm run fmt            # prettier

# 测试
npm run test-backend   # node:test 后端单测（含 testcontainers 集成测试）
npm run test-e2e       # Playwright e2e
npm test               # 两者串联

# 备份/恢复（README）
npm run reset-password # extra/reset-password.mts 重置管理员密码
```

## 7. 测试与工程质量

- 测试规模：后端测试 83 个文件（`test/backend-test/`，覆盖监控 beat、通知 provider、迁移 v1→v3、认证、API、维护窗口等），e2e 用 Playwright（`test/e2e/` + `playwright.config.js`），另有 frontend-test 与 mock 数据库脚手架。
- 集成测试形态突出：devDependencies 引入 8 个 `@testcontainers/*` 包（mysql/mariadb/mssql/postgres/oracle/rabbitmq/hivemq），后端测试可拉起真实数据库容器验证多数据库支持（`test/backend-test/README.md`）。
- CI：`.github/workflows/` 下 22 个工作流——`auto-test.yml` 按 OS×Node 版本矩阵跑测试与 e2e，`codeql-analysis.yml` 安全扫描，另有大量社区治理自动化（issue 去重、PR 标题校验、冲突标记、AI slop 检测 `ai-slop.yml`、stale-bot 等）。
- 代码质量观察：后端 JS/TS 混用且正迁移到 TS（v3 新文件多为 .ts）；`monitor.js` 2200 行巨文件与内联 HTTP 探测是明显的技术债；作者本人在 `CONTRIBUTING.md` 开头坦承"部分代码结构和注释不佳"。工程脚本完备（`extra/` 下 release 自动化、演示环境部署、迁移工具）。

## 8. 社区与维护状态

- 活跃度极高：最近 push 2026-10-30（gh api），最新稳定版 2.5.8 于 2026-10-26 发布（gh api releases），tag 序列 2.5.8→2.5.4 连续密集发布，说明稳定版以补丁节奏滚动，master 同时处于 3.0.0-beta.0 大版本开发期。
- 贡献者：contributors API 第一页即满 100 人（分页查询第二页起为 0 是 API 上限假象，精确总数未确认；Open Collective 徽章显示另有大量赞助者）。
- 社区基础设施齐全：81 种语言的 Weblate 翻译平台（README 徽章）、Open Collective 赞助、专门的多语言文档仓库（README 链接）。open issues 852、fork 8.6k（gh api）。
- 治理特点：`CODE_OF_CONDUCT.md`、`SECURITY.md`、22 个 CI 治理工作流显示重度社区运营；v2→v3 期间仍保持 2.x 补丁发布（`package.json` scripts 中 `setup` 固定 `git checkout 2.5.8`）。

## 9. 亮点与风险

亮点：
1. 插件化扩展点设计极致：新监控类型只需继承 `MonitorType.check()`（`server/monitor-types/monitor-type.js`），新通知渠道只需继承 `NotificationProvider.send()`，前后端各加一文件——111 个通知渠道基本都由社区贡献，是项目星数增长的发动机。
2. 调度实现务实：setTimeout 自循环 + 耗时扣除 + 启动随机打散 + 异常自愈，代码简单但在数万监控器场景下被验证可靠（`server/model/monitor.js`、`server/start-monitors.ts`）。
3. 本地化与开箱即用做到极致：81 种语言、compose 一键部署、默认 SQLite 零配置（`db/kuma.db` 模板 + `server/database.js`）。
4. 测试体系完整且现代化：83 个后端测试文件 + testcontainers 多数据库集成测试 + Playwright e2e + CodeQL（`.github/workflows/`）。
5. v3 演进方向清晰：多数据库、better-auth、OpenAPI/zod 化的 API、MCP server（`server/routers/mcp-router.ts`），在保持单实例简单性的同时补企业化能力。

风险/局限：
1. 单实例单点架构：无内置多副本/分布式探测（globalping 只是借用第三方节点），监控器数量极大时依赖单 Node 进程。
2. 技术债集中：`server/model/monitor.js` 2204 行、HTTP 类探测内联在 beat 循环、JS/TS 混用，重构期（v3）贡献者上手成本高。
3. 主版本断层风险：master 是 3.0.0-beta.0，稳定用户在 2.x，两条线并存期间文档/插件兼容性维护负担重（`package.json` version 与 Releases tag 对照）。
4. engines 要求 Node >= 26.2.0（`package.json`），对非 Docker 用户的前沿版本要求较激进（未确认发行版打包可用性）。

## 10. 总结评价

Uptime Kuma 是"单实例自托管监控"品类的事实标准：以 92.8k 星位居监控类开源项目头部，靠的是极致的开箱体验、花哨的 UI 和一个几乎全自动生长的通知渠道生态，而非分布式架构上的技术深度。它的源码是学习"插件化扩展点设计"的优秀范本——监控类型与通知渠道两条扩展轴都遵循基类 + 注册表 + 一文件一实现，社区贡献可以被架构直接消化；同时它也展示了这类项目随规模增长的真实代价（巨文件、混用范式、大版本迁移）。适合需要自建可用性监控的运维/个人用户直接部署使用，也适合想学习 Node + Vue 全栈、WebSocket 实时系统与社区驱动插件体系的开发者阅读源码；阅读建议从 `server/model/monitor.js` 的 `beat()` 与 `server/monitor-types/`、`server/notification-providers/` 各抽两个实现入手。
