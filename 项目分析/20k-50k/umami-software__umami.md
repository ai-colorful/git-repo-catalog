# umami-software/umami 项目分析

> 星数 35500（快照 2026-10-24）｜主语言 JavaScript（实际为 TypeScript/React）｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：Umami 是一个开源、自托管、隐私优先的网站与产品分析平台，作为 Google Analytics 的轻量替代品，无 Cookie、可绕过广告拦截器、单二进制式部署（Next.js 一体化应用）。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | umami-software/umami | package.json `name: "umami"` |
| 版本 | v3.5.0（2026-11-12 发布） | package.json / gh api releases |
| 星数 | ~35.5k（watchers 40122，forks 8542） | gh api repos/umami-software/umami |
| 主语言 | TypeScript（src/ 共 1376 个 ts/tsx 文件、约 13.4 万行） | find 统计 |
| 许可证 | MIT（Copyright (c) 2022 Umami Software, Inc.） | LICENSE |
| 默认分支 | master，最近 push 2026-11-12 | gh api |
| 最近活跃 | 极活跃：每月发版（v3.3.0 8/12 → v3.5.0 11/12） | gh api releases |

## 2. 项目是什么，解决什么问题

Google Analytics 存在两大痛点：一是隐私合规负担（GDPR 下需要 Cookie 同意横幅），二是被 uBlock 等广告拦截器默认拦截导致数据缺失。Umami 用两个核心设计回答了这两个问题：

**无 Cookie 会话识别。** 服务端不为访客种任何 Cookie 或 localStorage 标识，而是用确定性哈希现场计算会话 ID：`uuid(sourceId, ip, userAgent, sessionSalt, distinctId)`（src/lib/collection-session.ts:111），其中 salt 由服务端 `APP_SECRET` 与按周期（默认月、可配天/周）轮换的时间盐混合（src/lib/crypto.ts 的 `uuid()`/`getSalt()`）。IP 与 User-Agent 只参与哈希、从不明文落库，且哈希中混入服务端密钥，外部无法反推原始 IP。因此 Umami 不触发 Cookie 同意横幅义务，README 与官网也以此为核心卖点。

**规避广告拦截。** 追踪脚本名义路径为 `/script.js`（rollup 产物 public/script.js，rollup.tracker.config.js），服务端通过 `next.config.ts` 的 rewrites 支持任意改名（`TRACKER_SCRIPT_NAME` 环境变量，逗号分隔多个别名）、任意收集端点改名（`COLLECT_API_ENDPOINT` rewrite 到 `/api/send`）、外链脚本（`TRACKER_SCRIPT_URL`），甚至整站 `BASE_PATH` 迁移。 adversaries 看到的是普通静态 JS 与普通 POST。

**从"页面统计"到"产品分析"的演进。** v3 已远超 PV/UV 统计：事件数据支持任意 JSON（数字 4 位精度、字符串 500 字符上限、对象 50 属性上限，src/tracker/index.ts 注释）；提供漏斗/路径/留存/归因/收入/热图报表（src/app/(compat)/compat/api/reports/ 下 9 类）、rrweb 会话回放（依赖 rrweb，package.json；src/recorder/ 产出 recorder.js）、错误采集（/api/send 处理 `type: 'error'`，src/lib/errors/collect）、Web Vitals（payload 中 lcp/inp/cls/fcp/ttfb，src/app/api/send/route.ts:44-49）。

**产品矩阵扩张。** v3 还内置了带统计的短链服务（`/q/[slug]` 重定向，src/app/(collect)/q/[slug]/route.ts）、转化追踪像素（`/p/[slug]` 返回 1x1 GIF，src/app/(collect)/p/[slug]/route.ts）、看板（Board）、团队协作（Team/TeamUser）、分享链接（Share）、API Key、双因素认证（TwoFactorAuth 系列 4 个 model）、OpenAPI 契约 + 自动生成的 TS 客户端（packages/api-client）、MCP 服务器（packages/mcp，让 AI 助手直接查询分析数据）。

**商业模式。** 同一代码库支撑官方云服务：`CLOUD_MODE` 环境变量开启云路径（next.config.ts 中 /script.js rewrite 到 cloud.umami.is）；src/lib/subscription.ts 定义 Free（1 网站/0 团队）与 Pro（20 网站/10 团队）限额及 isPro/isBusiness 标志；send 路由注释"Account collection blocks are enforced by the cloud gateway"（src/app/api/send/route.ts:63）。开源版 = 云版的代码基座，是典型的 open-core。仓库根目录还保留了 soc2.md（SOC 2 准备笔记），印证云服务面向企业客户的合规路线。

## 3. 技术栈与依赖

| 层 | 技术 | 出处 |
| --- | --- | --- |
| 框架 | Next.js 16.4.2（App Router + Turbopack）+ React 19 | package.json |
| 语言 | TypeScript 6，biome 做 lint/format | package.json, biome.json |
| 状态 | zustand（客户端 store）+ @tanstack/react-query | src/store/, package.json |
| UI | 自研组件库 @umami/react-zen + chart.js + lucide-react | package.json |
| ORM | Prisma 7（@prisma/adapter-pg + read-replicas 扩展） | package.json, src/lib/prisma.ts |
| 分析引擎 | @clickhouse/client（可选） | package.json, src/lib/clickhouse.ts |
| 流式接入 | kafkajs（可选，云规模事件管道） | package.json, src/lib/kafka.ts |
| 缓存 | redis（slug/网站缓存 + fetch 缓存助手） | src/lib/redis.ts |
| 地理 | maxmind GeoLite2-City（构建期下载）+ x-vercel-ip-* 头 | scripts/build-geo.js, src/lib/detect.ts |
| 回放 | rrweb / rrweb-player 2.1.7 | package.json |
| i18n | next-intl，54 种 UI 语言 | public/intl/messages/ 共 54 个 json |
| 会话/安全 | jsonwebtoken、bcryptjs、otplib（2FA） | package.json |
| 测试 | vitest（单元）+ Playwright（e2e/API） | vitest.config.ts, playwright.config.ts |

注意：v2 曾支持 MySQL，当前 v3.5.0 已收敛为 PostgreSQL-only（prisma/schema.prisma `provider = "postgresql"`、README 要求 PostgreSQL v12.14+、src/lib/db.ts 的 `getDatabaseType` 仅识别 postgres）+ 可选 ClickHouse。这是与旧资料的重要差异。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| src/app | Next.js App Router：`(main)` 仪表盘 UI、`(collect)` 短链/像素入口、`(compat)` 旧报表兼容层、api/ 全部 REST 端点（send、batch、record、websites、teams、auth 等 17 个子路由）、share/ 公开分享、mcp/ |
| src/tracker | 浏览器追踪脚本源码（构建为 /script.js） |
| src/recorder | 会话录制脚本（构建为 /recorder.js） |
| src/queries | 数据查询层：prisma/（关系型元数据查询 20 个模块）、sql/（分析查询，Prisma 与 ClickHouse 双实现） |
| src/lib | 基础设施：db/clickhouse/prisma/redis/kafka、crypto、detect、auth、collection-session、i18n、csp 等 60+ 模块 |
| src/components, src/store, src/i18n | React 组件、zustand 状态、i18n 配置 |
| prisma | schema.prisma（31 个 model）+ 28 个迁移 |
| db | clickhouse/（schema.sql + 16 个迁移）、postgresql/（迁移辅助） |
| packages | pnpm workspace：api-client（OpenAPI 生成的 TS 客户端）、mcp（MCP 服务器） |
| scripts | 构建/检查脚本：build-geo（GeoIP 下载）、check-db/check-env、seed、telemetry 等 |
| tests | api/（Playwright API 测试）、e2e/ |
| docker, Dockerfile, docker-compose.yml | 自托管部署（多阶段构建，standalone 输出） |

组织方式评述：单仓单应用（非 monorepo 但含两个 workspace 子包），前后端同仓——Next.js App Router 把 UI（(main)）与 API（api/）收敛在一个 router 里，路由组 `(collect)`/`(compat)` 用括号命名隔离不进 URL 的路径，是 App Router 组织大型应用的教科书式用法。追踪脚本作为独立子工程用 rollup 单独打包，与主应用构建流水线解耦（package.json build 链中 build:tracker 独立步骤）。

## 5. 核心架构与关键实现

### 5.1 无 Cookie 会话模型（隐私核心）

会话不存状态、只做哈希。收集请求到达时：

```ts
// src/lib/collection-session.ts (L107-119, 有删节)
const sessionSalt = process.env.ENABLE_CLIENT_SALT && salt ? hash(baseSalt, salt) : baseSalt;
const visitSalt = hash(startOfHour(createdAt).toUTCString());
const sessionId = uuid(sourceId, ip, userAgent, sessionSalt, distinctId ?? '');
const visitId = cache?.visitId || uuid(sessionId, visitSalt);
```

- `uuid()` 内部是 UUIDv5(hash(参数 + 服务端 APP_SECRET))（src/lib/crypto.ts:61-66），服务端密钥保证外部无法从 session ID 反推 IP/UA。
- sessionId 的盐按周期轮换（默认月，`SALT_ROTATION` 可配），轮换后同一访客自然产生新会话，避免长期追踪；visitId 每小时轮换实现"回访"统计（migration 05_add_visit_id 专门为此加了列）。
- 需要"识别"登录用户时走 opt-in 的 `umami.identify(distinctId)`（src/tracker/index.ts L740），身份值由站点自己提供，Umami 不自动采集。
- 浏览器端 localStorage 仅用于 `umami.disabled` 退出开关与显式 identify（src/tracker/index.ts:272-274, 459），全程无 Cookie。

### 5.2 追踪脚本 /script.js

src/tracker/index.ts（779 行 TS）经 rollup 打包为 public/script.js（rollup.tracker.config.js:8-9）。特点：payload 极小（website id + url/referrer/title/screen/language/hostname）、支持自定义事件 `umami.track(name, data)`、Web Vitals 指标随页面上报、错误上报（errors.ts 的 captureException）、`salt-rotation` 配置支持客户端口径的盐轮换（hourly/daily/weekly/monthly，tracker/index.ts:224-225）。服务端 side：/api/send 用 `x-umami-cache` JWT 令牌缓存网站与会话解析结果，减少高流量下的 DB 查询（src/app/api/send/route.ts:56-60）。isbot 库过滤爬虫（route.ts 首行 import）。

### 5.3 多数据库查询层（Prisma / ClickHouse / Kafka）

统一的运行时分发函数是整个分析查询层的枢纽：

```ts
// src/lib/db.ts
export async function runQuery(queries: any) {
  if (process.env.CLICKHOUSE_URL) {
    if (queries[KAFKA]) return queries[KAFKA]();   // 云规模写入走 Kafka
    return queries[CLICKHOUSE]();
  }
  const db = getDatabaseType();
  if (db === POSTGRESQL) return queries[PRISMA]();
}
```

src/queries/sql/ 下每个分析查询（getWebsiteStats、funnels、attribution 等 40+ 文件）同时提供 clickhouse 与 prisma 两个实现，按环境变量选择：小规模自托管用 PG 单库即可，大规模（云版）切 ClickHouse + Kafka。ClickHouse 侧是列式 + 物化视图方案：db/clickhouse/schema.sql 定义 website_event（MergeTree）、website_event_stats_hourly（AggregatingMergeTree 物化视图）、event_data_pivot/session_data_pivot（属性透视预聚合）、session_data（ReplacingMergeTree 去重）。Postgres 侧由 src/lib/prisma.ts（954 行）用原生 SQL 拼装同构查询，并支持 `@prisma/extension-read-replicas` 读写分离。Redis 用于 slug→Link/Pixel 解析缓存与通用 fetch 缓存（src/lib/redis.ts:84）。

### 5.4 端点伪装与多租户路由

next.config.ts 是"反拦截 + 云租户"的配置中枢：`TRACKER_SCRIPT_NAME` 支持同时挂多个脚本别名（防封锁轮换）、`COLLECT_API_ENDPOINT` 把任意路径 rewrite 到 /api/send、`API_URL` 把整个 API 挪到任意前缀、`CLOUD_MODE` 下脚本直连云端。`(compat)` 路由组保留 v2 报表 API 兼容（next.config.ts rewrites 中 `/api/reports/:path*` → compat），`(collect)` 路由组的 `/p/[slug]`、`/q/[slug]` 复用同一个 send 处理器（route.ts 内 `import { POST } from '@/app/api/send/route'`），短链点击与像素加载本质上是伪装成导航/图片请求的事件收集。

## 6. 构建与运行

```bash
# 源码运行（README.md）
git clone https://github.com/umami-software/umami.git && cd umami
pnpm install
# .env: DATABASE_URL=postgresql://... APP_SECRET=$(openssl rand -hex 32)
pnpm run build && pnpm run start   # next start

# Docker 自托管（docker-compose.yml：umami + postgres:15-alpine）
docker compose up -d   # 健康检查 /api/heartbeat

# 构建/检查命令（package.json scripts）
pnpm build            # check:env → build:db(prisma client) → tracker → recorder → geo → openapi → packages → next build
pnpm test             # vitest（pretest 自动 prisma generate）
pnpm test:e2e         # playwright；test:api 用 docker-compose.test.yml 起真实 PG（可加 --clickhouse）
pnpm lint / format    # biome
```

要求：Node.js 20.19+（22 LTS 推荐）、PostgreSQL 12.14+；可选 CLICKHOUSE_URL、KAFKA_URL+KAFKA_BROKER、REDIS_URL、MAXMIND_LICENSE_KEY（GeoLite2）。

## 7. 测试与工程质量

- 单元测试：186 个 *.test.ts(x)（src/ 与 packages/ 内，vitest），与被测模块同目录就近放置，覆盖面广——从 lib（crypto/filters/jwt 等几乎每个模块）到 queries（getWebsiteStats.test.ts 等）、组件与页面（page.test.tsx、MobileNav.test.tsx）。
- 集成/API 测试：tests/api + tests/e2e 共 57 个 Playwright 文件；scripts/test-api.ts 驱动 docker-compose.test.yml 起真实 Postgres（可选 ClickHouse）跑契约级 API 测试，支持 --coverage-report。
- CI（.github/workflows/ci.yml）：push/PR 触发，pnpm → lint:i18n → openapi:generate/check → api client check → build packages → vitest → 全量 build。契约先行（OpenAPI schema 与生成客户端有专门的 check 命令，scripts/check-openapi.ts、generate-api-client.ts）。
- CD：cd.yml 按 v*.*.* tag 构建多架构镜像推 GHCR + Docker Hub；cd-cloud.yml 发布云版；gitleaks.yml 做密钥泄漏扫描；stale-issues.yml 自动管理 issue。
- 代码质量观察：biome 统一 lint/format；TS `ignoreBuildErrors: true`（next.config.ts）说明类型检查并未完全前置到构建（依赖独立 tsc 步骤）；`/api/send` 等热路径大量使用 zod schema 校验（src/app/api/send/request-schema.ts）。注：仓库根的 soc2.md 含内部 AWS 账号/SSO 细节与整改清单（如"公开数据库无入站限制"），这类内部合规笔记出现在公开仓库中值得商榷。

## 8. 社区与维护状态

- 活跃度：gh api 显示最近 push 2026-11-12（分析当日 2 天前），125 个 open issues，8542 forks；仓库创建于 2020-07-17。
- 发版节奏：近 5 个 release 间隔约 3-5 周（v3.3.0 2026-08-12 → v3.5.0 2026-11-12），每月级稳定发版。
- 贡献者：contributors 首页 100 人均有关联提交，榜首 4124 次提交（作者 Mike Cao，未逐一核实全量人数）；核心团队另有 brian、francis 等出现在 soc2.md 的 Identity Center 用户清单中。
- 生态：官网 umami.is 文档站、官方 Docker/GHCR 镜像、Vercel/Netlify 部署配置（vercel.json、netlify.toml）、54 语言社区翻译（public/intl/messages，另有 scripts/check-missing-messages.js 保证键完整）。

## 9. 亮点与风险

亮点：
1. 隐私设计是工程级而非口号：盐轮换哈希会话 + 服务端密钥混淆 + 无任何 Cookie/存储标识，代码路径清晰可审计（src/lib/collection-session.ts、src/lib/crypto.ts）。
2. 双存储引擎架构优雅：runQuery 分发 + queries/sql 下 PG/ClickHouse 双实现，同一代码覆盖"个人博客"到"云服务海量事件"两端（src/lib/db.ts、db/clickhouse/schema.sql）。
3. 产品完整度高：统计之外的漏斗/回放/热图/错误采集/短链/像素/团队/API/MCP 全部内置且共享同一收集管道。
4. 工程纪律好：OpenAPI 契约先行并自动生成客户端、gitleaks 扫描、54 语言完整性检查、186 个单测 + 真实 DB 的 API 集成测试。
5. 兼容性用心：`(compat)` 路由组保留旧报表 API，旧脚本路径经 rewrites 无损迁移。

风险/局限：
1. 大版本迁移有破坏性：v3 移除 MySQL 支持并强制 PostgreSQL（README/prisma schema），老用户升级成本高。
2. 会话哈希模型的固有限制：换 IP（如移动网络）会拆分会话；盐轮换周期内无法区分"同一访客的多次会话"，统计口径与 Cookie 类工具不可直接对比。
3. 功能膨胀带来维护面扩大：31 个 Prisma model、17 个 API 路由域，且 `typescript.ignoreBuildErrors=true`，类型安全依赖约定而非强制。
4. soc2.md 等内部运维笔记出现在公开仓库，存在信息泄露面扩大风险（内容本身已声明"非合规结论"）。

## 10. 总结评价

Umami 是隐私优先分析工具这个赛道事实上的开源标杆：35k+ 星、月度发版、单仓库同时供养自托管社区版与商业云版。它的源码最值得看三处——`collection-session` 的无 Cookie 会话哈希（隐私工程范式）、`runQuery` + 双实现查询层（一套代码通吃 PG 与 ClickHouse 的教科书）、`next.config.ts` 的端点伪装与云租户路由。适合需要"装完即用、不惹 Cookie 麻烦"的站长/团队自托管，也适合研究 SaaS 化开源产品如何平衡单租户简洁与多租户规模的架构师；对想学习 Next.js App Router 大型应用组织方式的开发者，src/app 的路由组划分同样是很好的参考。
