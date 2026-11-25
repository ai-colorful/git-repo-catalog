# Wei-Shaw/sub2api 项目分析

> 星数 45521（快照 2026-10-24）｜主语言 Go｜档位 20k-50k｜分析日期 2026-11-06

## 1. 项目速览

一句话定位：Sub2API 是一个自部署的 AI API 网关平台，把 Claude、OpenAI（Codex）、Gemini、Grok 等订阅制账号（OAuth/订阅会话）与 API Key 统一纳管为可调度的上游账号池，对外暴露 OpenAI/Anthropic/Gemini 兼容接口，由平台完成鉴权、调度、计费与多用户分摊。

| 项目 | 事实 |
|------|------|
| 仓库 | Wei-Shaw/sub2api（`gh api` 核实） |
| 星数 / Fork / Open Issues | 45,521 / 9,829 / 3,842（2026-11-06） |
| 主语言 | Go 1.27.3（`backend/go.mod`），前端 Vue 3.5（`frontend/package.json`） |
| 许可证 | LGPL-3.0（`LICENSE`） |
| 创建时间 | 2025-12-18（`gh api repos/Wei-Shaw/sub2api` 的 `created_at`） |
| 最近 push | 2026-11-05（分析前一天，极活跃） |
| 最新 release | v0.2.19（2026-10-28），约每周一版 |
| 贡献者 | 约 409 人（contributors API 分页推算） |
| 数据依赖 | PostgreSQL 15+、Redis 7+（`deploy/docker-compose.yml`） |

## 2. 项目是什么，解决什么问题

Claude Code、Codex、Gemini CLI 等工具的订阅套餐按月计费、额度大，但只能绑定单一账号与原生客户端使用；而官方 API 按 token 计费成本高。Sub2API 解决的问题是：把多个订阅账号汇成一个账号池，对外转换成标准 API（Anthropic Messages / OpenAI Responses / Chat Completions / Gemini v1beta），让任何 SDK 或编码工具改个 Base URL 就能调用，多人共享分摊订阅成本（`README_CN.md` 项目概述、`backend/internal/server/routes/gateway.go`）。

平台侧补齐了订阅转 API 后缺失的商业化能力：用户体系（注册/登录/余额）、API Key 发放与额度、Token 级计费与用量统计、内置支付（易支付/支付宝/微信/Stripe，`docs/PAYMENT_CN.md`、`backend/internal/payment/`）、Web 管理后台（`frontend/src/views/admin/`，含账号、分组、用户、渠道监控、风控、公告、兑换码、订阅套餐等页面）。管理员在上游侧维护账号池与代理，在下游侧运营一个"API 中转站"。

上游账号支持多种接入类型：OAuth（Claude OAuth PKCE 流程，`backend/internal/pkg/oauth/oauth.go` 中硬编码 `claude.com/cai/oauth/authorize` 授权端点）、API Key、Cookie 等（`backend/ent/schema/account.go` 的 `platform`/`type` 字段）。调度层按分组（Group）把账号分配给不同用户/Key，支持粘性会话、优先级、负载感知与跨平台混合调度（`backend/internal/service/gateway_scheduling.go`）。

项目作者在 README 顶部显著警示：使用本项目可能违反 Anthropic 等上游服务商的服务条款，账号封禁等风险由使用者自行承担，且未授权任何商业化运营（`README_CN.md` "重要提醒"节）。客观来说，其增长与 2026 年订阅制 AI 编程工具的普及直接相关——仓库 2025-12-18 创建，一年内达到 45k+ 星，README 中带有 trendshift 徽章并接入了大量赞助商，社区生态还包括第三方移动端 sub2api-mobile（Expo/React Native，`README_CN.md` 生态项目表）。

## 3. 技术栈与依赖

后端（`backend/go.mod`）：

- Web 框架 Gin 1.9；ORM Ent（entgo.io/ent，schema 在 `backend/ent/schema/`）；依赖注入 google/wire（`backend/internal/handler/wire.go`）
- PostgreSQL 驱动 lib/pq；Redis go-redis v9；配置 viper；日志 zap
- HTTP 客户端 imroc/req v3；WebSocket coder/websocket + gorilla/websocket
- `refraction-networking/utls`：TLS ClientHello 指纹定制（见第 5 节）
- 协议处理：tidwall/gjson、sjson（定点 JSON 改写）、tiktoken-go/tokenizer（token 计量）、modelcontextprotocol/go-sdk（MCP 支持）
- 支付：stripe-go v85、alipay v3、wechatpay-go；验证码：阿里云/腾讯云 SDK
- 工程效率：alitto/pond v2（worker pool）、ristretto/patrickmn go-cache（本地缓存）、robfig/cron（定时任务）
- 测试：testify、testcontainers-go（postgres/redis 模块）、miniredis、go-sqlmock

前端（`frontend/package.json`）：Vue 3.5 + Vite + TypeScript + Pinia + vue-router + vue-i18n（en/zh 双语，`frontend/src/i18n/`）、TailwindCSS、chart.js/vue-chartjs（图表）、Stripe/Airwallex SDK（支付组件）。

规模：后端 3119 个 `.go` 文件（其中 1662 个 `_test.go`，占比约 53%）；SQL 迁移 324 个（`backend/migrations/`）；前端 323 个 `.vue`、ts/vue 合计 940 个文件。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|------|------|
| `backend/cmd/` | 入口：server 主程序、jwtgen、profit-preview 等辅助命令 |
| `backend/internal/handler/` | HTTP 层：网关（gateway/openai/gemini/grok）、认证、支付、用量等 handler，约 300+ 文件 |
| `backend/internal/service/` | 业务层核心，602 个非测试文件，含调度、计费、账号池、协议桥接 |
| `backend/internal/pkg/` | 协议与基础设施包：apicompat（协议转换）、claude/openai/xai/gemini/geminicli/antigravity（各上游模型类型）、tlsfingerprint、oauth 等 |
| `backend/internal/server/` | 路由注册（`routes/`）与中间件（`middleware/`，JWT/API Key 鉴权、限流、审计） |
| `backend/internal/payment/` | 支付Provider注册表、金额/费率/负载均衡 |
| `backend/internal/repository/` `ent/` | 数据访问层与 Ent 生成的 ORM 代码 |
| `backend/internal/securityaudit/` | 安全审计协调器（内容审计） |
| `backend/internal/managementmcp/` | 管理面 MCP 服务 |
| `backend/migrations/` | 324 个 SQL 迁移 |
| `frontend/` | Vue 3 管理面板 + 用户门户（views 分 admin/user/auth/public） |
| `deploy/` | install.sh 一键脚本、多套 docker-compose、systemd unit、Caddyfile、config.example.yaml |
| `docs/` | 支付、插件开发、复合分组、MCP、异步图片任务等文档及 legal 合规模板 |
| `openspec/` | 规格驱动开发流程目录（changes/ 下有待实施规格） |
| `skills/`、`tools/`、`.github/` | 管理 CLI skill、审计脚本、CI workflow |

组织方式为前后端同仓的单体（后端 Go + 前端嵌入二进制发布，见 `Dockerfile` 多阶段构建）；`backend/internal` 按 handler/service/repository 分层，service 层因领域庞大做了按文件域拆分（如 `gateway_*.go`、`openai_gateway_*.go`、`account_*.go` 系列），部分文件头注释标明"由 gateway_service.go 纯移动拆分而来"（`internal/service/gateway_scheduling.go`），可见重构活跃。

## 5. 核心架构与关键实现

### 5.1 协议转换层（apicompat + 各平台 GatewayService）

对外路由在 `backend/internal/server/routes/gateway.go`：`/v1/messages`（Anthropic）、`/v1/responses`、`/v1/chat/completions`、`/v1/embeddings`、`/v1beta`（Gemini）、图片/视频批处理、`/sub2api/billing` 等。转换核心在 `backend/internal/pkg/apicompat/`，实现 Anthropic Messages ↔ OpenAI Responses ↔ Chat Completions 三方互转（`anthropic_to_responses.go`、`responses_to_anthropic_request.go`、`chatcompletions_anthropic_bridge.go`、`chatcompletions_to_responses.go`），包注释写明目标是"多协议支持，让不同 API 格式的客户端经由统一网关被服务"（`apicompat/types.go`）。转换保留 `metadata` 等透传字段，注释中说明 OAuth 路径依赖 `metadata.user_id` 参与上游"是否为官方 Claude Code 请求"的判定（`types.go` AnthropicRequest.Metadata 注释）。

平台侧各有 GatewayService：Anthropic（`gateway_service.go` 系列）、OpenAI/Codex（`openai_gateway_*.go` 系列，含 Responses/Chat/WS 多通道）、Gemini/Antigravity（`antigravity_gateway_*.go`）、Grok（`openai_gateway_grok_*.go`，含 SSE 过滤与工具协议桥）、Bedrock（`gateway_bedrock.go`）。Composite 分组按请求模型自动路由到目标平台（`internal/handler/composite_platform.go` 的 `DetectModelPlatform`）。

### 5.2 订阅会话桥接与客户端指纹模拟（客观技术描述）

对 OAuth 订阅账号，网关在转发前对请求体做规范化：注入/重写 `metadata.user_id`（`internal/service/gateway_claude_oauth_body.go` 的 `ensureClaudeOAuthMetadataUserID`/`RewriteUserIDWithMasking`）；`sanitizeSystemText` 会把客户端系统提示词中的 OpenCode 固定身份句替换为标准 Claude Code 横幅，注释说明原因是"Anthropic 可能将其视为非 Claude Code 指纹"。请求是否被视为真 Claude Code 客户端由 `isClaudeCodeClient` 判定：User-Agent 匹配 `claude-cli/X.Y.Z` 且 `metadata.user_id` 通过格式校验（`internal/service/gateway_upstream_response.go` 开头）。

TLS 层使用 utls 定制 ClientHello：`internal/pkg/tlsfingerprint/dialer.go` 的 Profile 结构可配置加密套件、曲线、扩展顺序等，包注释写明"模拟 Node.js/Claude Code 客户端的 TLS 连接"；指纹模板存数据库 `tls_fingerprint_profiles` 表并通过 `Account.Extra.tls_fingerprint_profile_id` 绑定到具体账号（`backend/ent/schema/tls_fingerprint_profile.go`）。另有 `internal/pkg/anthropicfp/dateline.go` 处理请求头特征。这些机制共同构成上游伪装链路，属于本仓库代码中明确实现的功能（其效果与合规后果见第 9 节风险）。

### 5.3 账号池调度与故障转移

调度入口 `SelectAccountForModelWithExclusions`（`internal/service/gateway_scheduling.go`）：按分组平台解析 → Claude Code 限制回退分组 → 通道定价限制预检 → anthropic/gemini 平台走混合调度（可混入开启 mixed_scheduling 的账号），其余走单平台选择；再上层是带客户端亲和（二维：客户端 ID + 用户 ID）的负载感知选择 `SelectAccountWithLoadAwareness`。粘性会话以 `session_hash` 保持同一会话落在同一账号，Nginx 反代需开 `underscores_in_headers on` 保住 `session_id` 头（`README_CN.md` 部署注意事项）。

调度状态机落在 Account 实体上：priority、load_factor、concurrency、schedulable、rate_limited_at/rate_limit_reset_at、overload_until、temp_unschedulable_until、session_window 窗口、quota_dimension（global/spark）、parent_account_id（子账号，代码中称"spark 影子账号"，凭据透传母账号，写入点有防泄漏不变量，`internal/service/account_credentials_persistence.go`）。调度缓存用 Redis 分桶（SchedulerBucket = GroupID:Platform:Mode）+ epoch 写令牌 + 生命周期租约做并发围栏（`internal/service/scheduler_cache.go`）。

失败转移双轨：上游错误重试（最多 5 次、300ms 指数退避上限 3s、总耗时 10s，`internal/service/gateway_forward.go` 常量区）与账号切换（401/403/429/529/5xx 触发 failover，`maxAccountSwitches` 默认 10 次、Gemini 3 次，`internal/handler/gateway_handler.go` 构造函数）。流式响应有 SSE 泵与保活：对真 Claude Code 客户端按版本注入 noop delta keepalive（`gateway_upstream_response.go`），另有 `failover_loop.go` 处理流中断后的续传/切换。

### 5.4 计费、配额与多用户模型

三层额度：平台账号池（上游订阅额度）、用户余额与用户×平台日/周/月美元限额（`UserPlatformQuotaCacheEntry`，`internal/service/billing_service.go`）、API Key 维度配额与 5h/1d/7d 滚动窗口限速（`backend/ent/schema/api_key.go` 的 quota/quota_used/rate_limit_5h/1d/7d 字段）。余额走 Redis 缓存扣减 + 异步落库（BillingCache 接口、UsageRecordWorkerPool）；Token 成本计算按模型定价（`gateway_request_pricing.go`、`account_stats_pricing.go`），并有利润控制门（`gateway_profit_control.go`）与批量图片任务的计费冻结/回收（`batch_image_billing_hold.go`/`recovery.go`）。

多用户/拼车模型：用户（User：余额、冻结余额、并发、RPM 限制、TOTP 二次验证、`backend/ent/schema/user.go`）— 分组（Group，绑定平台与账号集合）— API Key（绑定分组、IP 黑白名单、额度）三层关联；账号凭据在管理接口中做脱敏返回（`account_credentials_redact.go`）。支付模块抽象 Provider 注册表与负载均衡（`internal/payment/registry.go`、`load_balancer.go`），支持易支付/支付宝官方/微信官方/Stripe 四通道（`docs/PAYMENT_CN.md`）。

## 6. 构建与运行

```bash
# 一键安装（Linux，从 GitHub Releases 拉预编译二进制）
curl -sSL https://raw.githubusercontent.com/Wei-Shaw/sub2api/main/deploy/install.sh | sudo bash

# Docker Compose（app + postgres:18 + redis:8）
docker compose -f deploy/docker-compose.yml up -d

# 本地构建（顶层 Makefile）
make build          # = build-backend (backend/Makefile) + build-frontend (pnpm --dir frontend run build)
make test           # = test-backend + test-frontend（lint + typecheck + 关键 vitest 用例）

# 后端测试分层（backend/Makefile，CI 中使用）
make -C backend test-unit
make -C backend test-integration
```

配置模板 `deploy/config.example.yaml`（server/cors/security/gateway/simple_mode 等段）；生产可用 systemd unit（`deploy/sub2api.service`）或 Caddy 反代（`deploy/Caddyfile`）。前端构建产物嵌入 Go 二进制（多阶段 `Dockerfile`：node:24-alpine 构建前端 → golang:1.27.3-alpine 编译 → alpine:3.21 运行）。

## 7. 测试与工程质量

- 测试规模：后端 1662 个 `_test.go` 对 3119 个源文件，大量测试与被测逻辑同包同前缀成对出现（如 `gateway_handler_*_test.go`、`openai_gateway_ws_*_test.go`），覆盖单元与集成两层；集成测试用 testcontainers-go 起 PostgreSQL/Redis，单测用 miniredis/go-sqlmock（`backend/go.mod`）。
- CI（`.github/workflows/backend-ci.yml`）：push/PR 触发，三 job——部署 shell 脚本测试、Go 单测+集成（Go 版本硬断言 1.27.3）、前端 lint+typecheck+关键路径 vitest（顶层 `Makefile` 列出约 30 个 critical spec）。
- 安全扫描（`security-scan.yml`）：govulncheck + gosec + pnpm audit；`.github/audit-exceptions.yml` 维护豁免清单，配套 `tools/check_pnpm_audit_exceptions.py`。
- 发布（`release.yml` + `.goreleaser.yaml`/`.goreleaser.simple.yaml`）：tag 触发，二进制与镜像发布，`.github/release-tools/` 含发布矩阵脚本与测试。
- 质量观察：注释密度高且多为中文，关键路径附"安全不变量/外审第N轮"等批注（如 `account_credentials_persistence.go`），可见经过多轮安全评审；`openspec/` 与 `DEV_GUIDE.md` 显示团队使用规格驱动与 AI 辅助开发的流程管理。

## 8. 社区与维护状态

- 活跃度：`gh api` 显示最近 push 为 2026-11-05（分析前一天），open issues 3842，fork 9829；release 节奏约每周一版（v0.2.15 2026-10-09 → v0.2.19 2026-10-28）。
- 贡献者：约 409 人（contributors API 末页 410 推算），有 CLA 协议（`CLA.md`、`cla.yml` workflow）；`DEV_GUIDE.md` 提到 fork 仓库协作模式（bayma888/sub2api-bmai）。
- 生态：README 接入 20 余家赞助商横幅；社区衍生移动端 sub2api-mobile；周边代理/CDN/中转服务商形成明显商业生态（`README_CN.md`）。
- 增长背景（客观陈述）：仓库创建于 2025-12-18，2026 年内从零增长至 45k+ 星，README 带 trendshift 徽章；与订阅制 AI 编程工具（Claude Code、Codex、Gemini CLI）在 2026 年的大规模普及时间线吻合，具体增长动因未确认。

## 9. 亮点与风险

亮点：

1. 协议转换完备：三大协议（Anthropic Messages/OpenAI Responses+Chat/Gemini）互转 + 各上游方言（Codex、Antigravity、Grok、Bedrock）适配，且转换层有独立包与 testdata 驱动的测试（`backend/internal/pkg/apicompat/`）。
2. 工程化程度高：测试占比约 53%、CI 三层校验、安全扫描、发布矩阵、迁移 324 个，接近生产级开源服务标准。
3. 调度与容错精细：粘性会话、负载感知、混合调度、Redis 分桶写围栏、重试/切号双轨、流式保活，是账号池网关的完整工程参考实现（`internal/service/gateway_scheduling.go`、`scheduler_cache.go`）。
4. 商业闭环内置：用户/Key/分组/配额/支付/审计/风控/渠道监控一应俱全，管理面板 Vue3 双语（`frontend/src/views/admin/`）。

风险与局限：

1. 服务条款风险（项目自身亦明示）：将订阅账号转 API 并多人共享、配合 TLS/UA/metadata 伪装，可能违反 Anthropic/OpenAI/Google/xAI 的用户协议，存在账号封禁与法律合规风险（`README_CN.md` 重要提醒；伪装实现见 `tlsfingerprint/dialer.go`、`gateway_claude_oauth_body.go`）。
2. 敏感面大：凭据存储（OAuth token、Cookie）、支付通道、用户余额均在一体内，自部署方需自行承担安全运营责任；代码虽有脱敏与审计机制，错误配置仍可能泄露凭据（`account_credentials_redact.go`）。
3. 上游耦合脆弱：大量依赖上游非公开接口行为（metadata 判定、模型目录、配额头），上游一变即可能失效，从 `openai_gpt55_retirement` 等文档与高频发版可见维护成本高（`docs/OPENAI_GPT55_RETIREMENT.md`）。
4. 复杂度门槛：3119 个 Go 文件、PostgreSQL+Redis 双依赖、多通道配置，小型使用者部署与运维成本不低。

## 10. 总结评价

Sub2API 是当前"订阅转 API"赛道中工程完成度最高的开源实现之一：以 Go+Vue 单体交付了协议互转、账号池调度、计费支付、管理面板的完整网关平台，代码注释与测试密度都显著高于同类项目，一年内 45k 星反映了该需求在 2026 年的规模。它的技术价值与它的争议性一体两面——它是研究 API 网关、负载调度、协议转换的优质源码材料，同时也是上游服务商风控的直接对抗对象。适合后端工程师阅读其调度与 apicompat 转换层源码，或需要自部署多上游 API 网关的团队参考；使用者应清楚其 ToS 风险，作者亦明确禁止商业化运营。注意本仓库 LGPL-3.0 协议对二进制分发的相应义务。
