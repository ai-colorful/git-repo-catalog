# QuantumNous/new-api 项目分析

> 星数 50572（快照 2026-10-24，gh api 实时 51284）｜主语言 Go｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：自托管 AI 网关 / LLM API 分发系统——把 40+ 上游模型服务（OpenAI、Claude、Gemini、Bedrock、Vertex 等）聚合成统一 API，附带用户/令牌管理、路由、计费与 Web 控制台。

| 项目 | 值 | 出处 |
| --- | --- | --- |
| 名称 | QuantumNous/new-api（原 Calcium-Ion/new-api，org 已迁移；README 徽章仍指向旧名） | `gh api repos/QuantumNous/new-api`、`README.md` |
| 星数 | 51,284（2026-11-04 实时） | gh api |
| 主语言 | Go（约 6.2 万行，不含前端）+ React 19 / TypeScript 前端（`web/` 约 1454 个文件） | 仓库统计、`go.mod`、`web/package.json` |
| 许可证 | AGPL-3.0（基于 MIT 的 One API 二次开发） | `LICENSE`、`README.md` L333 |
| 默认分支 / 最近 push | main / 2026-11-03（分析前一日，极度活跃） | gh api |
| 最近 release | v1.0.0-rc.47（2026-10-25），rc.43→rc.47 约两周内连发 | gh api releases |
| 规模 | 仓库约 93MB，3089 个文件（tarball HEAD） | tarball 统计 |

## 2. 项目是什么，解决什么问题

new-api 是在 songquanpeng/one-api（MIT）基础上二次开发的 AI 网关（`README.md` L309、L333："developed based on One API"）。它解决的核心问题是：团队或服务商手里有多个上游模型渠道（官方 API、云厂商托管、第三方兼容服务），需要在一个入口对外提供 OpenAI/Claude/Gemini 兼容接口，同时做渠道调度、访问控制、用量统计和成本核算。

与原版 one-api 相比，new-api 的明显演进方向有三条（均出自源码证据）：一是协议转换能力被抽成独立模块 `relaykit/`，在 OpenAI Chat Completions、OpenAI Responses、Anthropic Messages、Gemini generateContent 四种协议间两两互转，并给出 good/fair/discouraged 质量分级与多跳选路（`relaykit/README.md`）；二是计费体系大幅强化，支持基于表达式的分段/条件计费（`pkg/billingexpr/`）；三是扩展机制引入 JavaScript 任务插件与插件市场（`pkg/jsplugin/`、`plugins/tasks/`，README 截图表含 Plugin marketplace）。

典型的使用场景（README "Project Description"）：团队共享已授权的模型访问、不改客户端切换供应商、运营私有多模型服务。项目自带 Web 控制台（渠道管理、模型广场、用量日志、定价、插件市场），前端由 `go:embed web/dist` 打进单二进制（`main.go` L48-51）。

需要说明的是，README 用 IMPORTANT/WARNING 大段强调合规边界：仅用于合法授权场景，要求用户合法取得上游 key、遵守上游 ToS，对外提供生成式 AI 服务需完成备案、内容安全、实名、日志留存、税务等义务（`README.md` L45-57）。这从侧面说明该项目的典型用法（API 分发/转售）确实贴近上游服务条款的灰色地带，项目方选择以"免责声明 + 功能开关"应对（详见第 9 章）。

## 3. 技术栈与依赖

后端（`go.mod`，Go 1.26.10，三个 Go module 组成多模块工作区）：

- Web 框架：Gin v1.9.1（+ cors/gzip/static 中间件）
- ORM：GORM v2，同时支持 SQLite（glebarez 纯 Go 驱动）/ MySQL / PostgreSQL；日志库另支持 ClickHouse（`gorm.io/driver/clickhouse`、`model/clickhouse_log_test.go`）
- 缓存：go-redis v8 + 本地内存缓存（大量 `_cache.go` 文件，如 `model/channel_cache.go`、`model/token_cache.go`）
- 鉴权与授权：JWT、WebAuthn/Passkey、TOTP、OAuth/OIDC（`coreos/go-oidc`）、Casbin v2（`service/authz/`、`model/casbin_rule.go`）
- 计费表达式引擎：expr-lang/expr（`pkg/billingexpr/`）
- 支付：epay（`Calcium-Ion/go-epay`）、Stripe（`stripe-go/v81`）
- AWS SDK v2（Bedrock Runtime）、音频/图片解析（wav/mp3/flac/mp4 等，用于多媒体 token 计量）
- 可观测：Prometheus client、Grafana Pyroscope、pprof（`main.go` import）
- 子模块：`relaykit/`（协议转换，仅依赖 uuid/lo/gjson/sjson）、`tokenkit/`（token 计数估算），均通过 replace 指向仓内目录（`go.mod` 尾部）

前端（`web/package.json`）：React 19 + TypeScript，构建用 Rsbuild（非 webpack/vite），TanStack Router/Query/Table/Virtual、Zustand、Base UI、Tailwind 4、VChart 图表、i18next、katex/marked/dompurify；包管理器为 Bun，测试 vitest，lint 用 oxlint（`web/package.json` scripts）。

其他：Electron 桌面封装（`electron/`）、moejs JS 插件运行时（`Calcium-Ion/moejs`，`pkg/jsplugin/`）。

## 4. 目录结构与代码组织

顶层目录（`ls src`，Go 文件约 62,119 行，305 个 `_test.go`）：

| 目录 | 职责 | 规模（Go 文件） |
| --- | --- | --- |
| `relay/` | 中转核心：各协议 handler + 40 个渠道适配器子目录 | 255 |
| `relay/channel/` | 40 个上游适配器（openai、claude、gemini、aws、vertex、ali、zhipu、xunfei……含 `advancedcustom/` 自定义渠道与 `task/` 任务渠道） | 见 `ls relay/channel/` |
| `controller/` | HTTP 控制器（渠道、用户、计费、OAuth、插件等 API） | 135 |
| `model/` | GORM 数据模型与缓存（channel/token/user/log/quota_reserve 等） | 115 |
| `service/` | 业务服务层（含 `service/authz/` Casbin 授权） | 114 |
| `middleware/` | Gin 中间件：auth、distributor（渠道分发）、rate-limit、i18n、audit 等 | 43 |
| `relaykit/`、`tokenkit/` | 独立 Go 子模块：协议转换、token 计量 | 134+5 |
| `setting/`、`constant/`、`common/`、`types/`、`dto/` | 配置项、常量、工具、类型定义 | 67/15/71/4/8 |
| `pkg/` | billingexpr、jsplugin、cachex、wsmanager、perf_metrics、ionet | 45 |
| `plugins/` | JS 任务插件本体（`plugins/tasks/`） | 16 |
| `web/` | React 控制台前端 | 1454 文件 |
| `docs/` | 安装、渠道、plugin-api、openapi 文档 | 19 |
| `electron/`、`bin/`、`e2e/`、`i18n/`、`oauth/` | 桌面壳、脚本、端到端测试、后端 i18n、OAuth 提供方 | 少量 |

组织方式评述：单体 Go 服务 + 两个仓内独立 Go module（relaykit/tokenkit 可被外部网关复用，各有自己的 go.mod 与 README）+ 独立前端目录。模块边界清晰：relaykit 明确声明"HTTP 收发、SSE、渠道选择、鉴权和计费由宿主完成"（`relaykit/README.md`），是近年少见的"从单体里长出库"的干净拆分。另有 `AGENTS.md`/`CLAUDE.md`/`.agents/`（含 shadcn-ui、i18n 等 agent skill），说明项目已把 AI 编码代理纳入工程规范。

## 5. 核心架构与关键实现

### 5.1 请求主链路：router → middleware.Distribute → controller.Relay → channel adaptor

每个 API 请求先过 `middleware/auth.go`（token/会话鉴权）与 `middleware/distributor.go` 的 `Distribute()`：解析请求体中的 model，套上渠道过滤约束（请求路径、任务插件身份、令牌模型限制、用户分组），再选中具体渠道（`middleware/distributor.go` L30-60）。随后进入 `controller/relay.go: Relay(c, relayFormat)`（L72）按协议分发到 `relay/` 下对应 handler（`chat_completions_via_responses.go`、`claude_handler.go`、`gemini_handler.go`、`responses_websocket.go` 等），handler 通过工厂取得适配器：

```go
// relay/relay_adaptor.go L52
func GetAdaptor(apiType int) channel.Adaptor {
    switch apiType {
    case constant.APITypeAli:
        return &ali.Adaptor{}
    case constant.APITypeAnthropic:
        return &claude.Adaptor{}
    case constant.APITypeAws:
        return &aws.Adaptor{}
    ...
```

40 个适配器目录（`relay/channel/`）各自实现统一的 `channel.Adaptor` 接口（`relay/channel/adapter.go`），负责 URL 拼装、鉴权头、请求/响应转换；AWS Bedrock、Vertex AI 等云厂商走各自 SDK/签名。`relaykit` 负责四种协议 DTO 与流式事件互转，README 给出支持矩阵与质量分级，Claude↔Gemini 之间标记为 discouraged（需经 OpenAI Chat 两跳）。

### 5.2 计费与配额：预扣 + Redis Lua 原子结算

计费入口在 `relay/request_billing.go: PrepareRequestBilling`：先按需构建 token 计量元数据（敏感词命中则本地拒绝），估算 prompt tokens，再经 `relay/helper/price.go: ModelPriceHelper` 算价——其中 `HandleGroupRatio` 处理用户分组倍率与分组专属倍率（L48-72）。定价不止"倍率×单价"一种：`pkg/billingexpr` 允许管理员为模型写计费表达式，测试用例展示了分段+固定价+参数开关的组合：

```go
// pkg/billingexpr/billingexpr_test.go L17
const expression = `(len <= 32000 ? tier("short", fixed(0.01)) : tier("long", p * 2 + c * 8))
    * (param("fast") == true ? 2 : 1)`
```

配额扣减用 Redis Lua 脚本保证原子性，且带乐观校验（用户 Id/CacheSchema 不匹配即返回 -1 回退查库）：

```go
// model/quota_reserve.go
const userQuotaReserveScript = `
if tonumber(redis.call('HGET', KEYS[1], 'Id') or '0') ~= tonumber(ARGV[2])
  or tonumber(redis.call('HGET', KEYS[1], 'CacheSchema') or '0') ~= tonumber(ARGV[3]) ...
  return -1
end
local quota = tonumber(redis.call('HGET', KEYS[1], 'Quota'))
if quota == nil or quota < tonumber(ARGV[1]) then return 0 end
redis.call('HINCRBY', KEYS[1], 'Quota', -tonumber(ARGV[1]))
return 1`
```

即"预扣预留 → 请求结束按真实 usage 结算/退款"的两段式计费，渠道重试时保留 billing session 与价格快照（`request_billing.go` 注释 L20-22）。令牌（API key）侧有独立额度字段 RemainQuota/UsedQuota 与对应脚本（`model/quota_reserve.go`）。

### 5.3 token 计量：tokenkit 独立模块与"诚实的不确定性"

`tokenkit/` 把"不调上游 API 就估算 token"拆成可复用库：GPT/o 系列用 tiktoken 精确计数，Claude/Gemini 按字符类别加权估算，图片按各家公开规则（OpenAI tile、Claude 两档分辨率、Gemini media resolution）计算。README 附了用真实 Claude Code/Codex 会话校准的误差表（prompt 偏差 +0.3%~+0.9%，平均误差 3%~4%），并明确写出已知偏差来源（`tokenkit/README.md`）。这种把估算误差摆在明面上的做法在同类项目里少见。

### 5.4 扩展：JS 任务插件与合规开关

异步任务类渠道（如 Suno/视频类）通过 JavaScript 插件接入：`pkg/jsplugin/engine.go` 用 moejs 运行插件脚本，插件以声明式 `buildSubmitRequest`/轮询函数描述上游交互，`e2e/doc_parse_test.go` 用内嵌 SQLite + httptest 起完整 Gin 服务验证插件全链路。支付、兑换码、订阅、邀请奖励等功能被 `controller/payment_compliance.go: requirePaymentCompliance` 门控——管理员必须在控制台会话（不允许 API token）中确认合规条款后才能启用（L20-45）。

## 6. 构建与运行

```bash
# 本地开发（makefile）
make build-web   # cd web && bun install && bun run build
make start-api   # go run main.go（go:embed 需要 web/dist 存在）
make dev         # docker compose -f docker-compose.dev.yml + 前端 dev server (5173)
make test        # 根模块 + relaykit + tokenkit 三模块测试（CI 同款）

# Docker 部署
docker-compose up -d   # 默认 Postgres，可切 MySQL（docker-compose.yml）

# 前端独立命令
cd web && bun run typecheck / lint / test   # rsbuild + oxlint + vitest
```

二进制内嵌前端（`main.go` `//go:embed web/dist`），默认端口 3000；另有 `new-api.service`（systemd）与 Heroku 配置（`go.mod` heroku 注记）。CI 要求三个 Go module 分别 vet/build/test（`.github/workflows/ci.yml`，actions 全部按 SHA pin）。

## 7. 测试与工程质量

- 测试规模：305 个 Go `_test.go` 文件，几乎每个核心目录都有（`controller/channel_*_test.go`、`relay/channel/*/api_request_test.go`、`model/quota_reserve_test.go`、`pkg/billingexpr/` 含 testdata）。 testify + 表驱动风格；`e2e/doc_parse_test.go` 是带真实 SQLite、httptest、插件运行时的端到端测试。
- CI（`.github/workflows/ci.yml`）：后端（vet + build + make test，三个 module 独立跑，GOWORK=off）与前端（typecheck + vitest）两个 job，PR 打开/同步/合并均触发；另有 docker-build、dependabot。测试不依赖真实上游（httptest/mock）。
- 质量观察：错误码与 i18n 全链路贯穿（`i18n/` + `nicksnyder/go-i18n`）；大文件偏多（controller/model 单文件数百行常见），但边界（relaykit/tokenkit）有独立 go.mod 和 README 强约束；前端有 knip 查死代码、版权头脚本（`web/package.json`）。文档有中文注释与英文混合（如 `docker-compose.yml`、`middleware` 部分中文注释），README 有 5 语言版本与翻译术语表（`docs/translation-glossary*.md`）。

## 8. 社区与维护状态

- 活跃度：最近 push 2026-11-03（分析前一日），release 节奏约每 3-5 天一个（v1.0.0-rc.43~rc.47，2026-10-10 至 10-25，gh api releases）。项目正在 rc 冲刺 1.0。
- 贡献者：gh api contributors 首页 30 人（分页上限，实际更多，精确数未确认）。主仓库在 QuantumNous org 下，另有 AtomGit 镜像（README 徽章）、alipay/阿里云/北大 BDA 等"Trusted Partners"展示（`README.md`）。
- 治理痕迹：`.github/ISSUE_TEMPLATE` 中英双语模板、`SECURITY.md`、`CODE_OF_CONDUCT.md`、`FUNDING.yml`；`AGENTS.md` 详细到"后端错误消息必须走 i18n key"级别的工程约定，且 `.agents/` 内置 agent skills（i18n-translate、shadcn-ui、vercel-react-best-practices），属重度 AI 辅助开发的开源项目。
- 生态：Cherry Studio、AionUi、北大 BDA、阿里云、io.net 列为合作伙伴（`README.md` L100-130，徽章为准）；`docs/images/` 与 `THIRD-PARTY-LICENSES.md` 佐证。

## 9. 亮点与风险

亮点：

1. relaykit/tokenkit 两个子模块的拆分质量高：依赖极简、边界明确、自带质量分级与校准数据，可直接被第三方网关复用（`relaykit/README.md`、`tokenkit/README.md`）。
2. 计费体系完整度罕见：预扣-结算两段式 + Redis Lua 原子脚本（`model/quota_reserve.go`）+ expr 表达式计费（`pkg/billingexpr`）+ 分组/令牌/渠道多层倍率（`relay/helper/price.go`），足以支撑真实运营。
3. 协议覆盖广且诚实：四大协议互转矩阵标明 good/fair/discouraged，转换损失以诊断返回而非静默吞掉（`relaykit/README.md`、`relay/common/conversion_diagnostics.go`）。
4. 工程化程度高：三模块 CI、SHA-pinned actions、305 个测试文件、端到端插件测试、双语 issue 模板、内置 agent 工作流约定。
5. 合规姿态主动：支付/兑换功能默认关闭需管理员确认合规（`controller/payment_compliance.go`），README 明确法律责任转移条款。

风险/局限：

1. 合规边界风险（如实记录）：项目本质是 API 分发/转售基础设施，README WARNING 段明确要求运营者自行完成备案、实名、内容安全、税务与"上游授权"义务（`README.md` L45-57）；用于转售共享 key 的场景普遍触碰上游供应商 ToS，法律责任完全由部署者承担。
2. AGPL-3.0 传染性：基于 MIT 的 one-api 二次开发后整体转 AGPL-3.0（`README.md` L333、`LICENSE`），对商用分发与 SaaS 化改造有强制开源义务。
3. 复杂度门槛：数据库三选一 + 可选 ClickHouse 日志库 + Redis + 支付/插件/多倍率体系，配置面大；单文件体量偏大（如 `controller/`、`model/channel.go`），新人上手成本不低。
4. token 估算误差：非 tiktoken 模型靠加权估算，README 自述平均误差 3%-5%（`tokenkit/README.md`），对按 token 精确对账的运营方是固有误差源。

## 10. 总结评价

new-api 是国内开源 AI 网关事实上的头部方案之一：在 one-api 的骨架上，把协议转换、token 计量拆成了两个可以独立评估质量的库，把计费做到了"能真金白银运营"的深度，并用高频 release、三模块 CI 和 300+ 测试文件维持着不低的工程水准。它适合作为团队级模型接入网关自托管，也适合想研究"LLM 网关如何做渠道调度与计费"的工程师通读源码——`middleware/distributor.go` → `controller/relay.go` → `relay/relay_adaptor.go` → `model/quota_reserve.go` 这条链路基本就是该领域的一份实现教材。但要清楚两点：把它用于对外 API 转售时，上游 ToS 与属地监管的合规责任全部落在运营者；AGPL-3.0 意味着任何修改分发都必须开源。
