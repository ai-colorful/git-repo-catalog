# Kong/kong 项目分析

> 星数 42200（快照 2026-10-24）｜主语言 Lua｜档位 20k-50k｜分析日期 2026-11-06

## 1. 项目速览

Kong Gateway 是构建在 OpenResty/Nginx 之上的云原生 API 网关，现定位为 "API 𖧧 LLM 𖧧 MCP Gateway"（README.md），通过插件体系提供代理、路由、负载均衡、认证、限流与 AI 流量治理能力。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | Kong / Kong Gateway | README.md |
| 星数 | 42200（快照）/ 44359（分析时实测） | gh api |
| 主语言 | Lua（OpenResty + Nginx 1.27.1.2） | kong/meta.lua `_DEPENDENCIES.nginx` |
| 版本 | 开发中 3.10.0，最新 release 3.9.4（2026-10-12） | kong/meta.lua、CHANGELOG.md、gh api releases/latest |
| 许可证 | Apache-2.0（CE 核心） | LICENSE |
| 最近活跃 | master push 2026-10-19，335 名贡献者，5238 forks | gh api、tarball 时间戳 |

## 2. 项目是什么，解决什么问题

Kong 把微服务/API 乃至 LLM、MCP 流量的横切关注点（认证、限流、可观测、日志、AI 语义安全）从业务代码中剥离，收敛到网关层以"Route/Service + 插件"的模型统一编排（README.md Features 节）。请求经 8000/8443 代理端口进入，配置与管理走 8001/8444 Admin API 与 Kong Manager（kong.conf.default `proxy_listen`/`admin_listen`）。

数据面与控制面可分离为三种部署形态（kong/conf_loader/、kong/clustering/）：
- **DB-less 声明式模式**：`database = off` 时以 YAML/JSON 声明式配置直接加载进内存（kong/db/declarative/init.lua），无数据库、适合 Kubernetes/不可变基础设施；
- **Postgres 模式**：`database = postgres`，实体存关系库，Admin API 写入后经版本号+轮询增量同步到各 worker（kong/db/init.lua、kong/runloop/handler.lua 的 router version 机制）。3.x 已移除 Cassandra，strategies 目录只剩 `off/` 与 `postgres/`（kong/db/strategies/）；
- **Hybrid 混合模式**：控制面（CP）与数据面（DP）分离，DP 经 TLS/RPC 从 CP 拉取配置（kong/clustering/control_plane.lua、data_plane.lua）。

AI Gateway 是近年重心：对 LLM 请求做代理、prompt 治理、请求/响应变换与流式（SSE）语义解析，支持 OpenAI、Anthropic、Azure、Bedrock、Cohere、Gemini、HuggingFace、Llama2、Mistral 共 9 类 provider（kong/llm/drivers/），并有 ai-prompt-guard、ai-request-transformer 等 7 个 ai-* 插件（kong/plugins/、kong/constants.lua）。

商业边界：本仓库为 Apache-2.0 的 Kong Gateway CE（含 45 个内置插件，kong/constants.lua）；RBAC、OIDC、Kong Manager 完整版、Vitals、开发者门户等属 Kong Enterprise/Konnect 闭源或 SaaS 产品（README.md "Konnect Cloud" 节明确其为商业订阅）。

## 3. 技术栈与依赖

- **运行时**：OpenResty（Nginx 1.27.1.2 + LuaJIT），由 kong/templates/nginx_kong.lua 程序化生成 nginx.conf（611 行模板，含 `init_by_lua_block`/`access_by_lua_block` 等挂载点与 20+ 个 `lua_shared_dict`）。
- **核心 Lua 依赖**（kong-latest.rockspec）：lua-resty-http（上游请求）、lua-resty-healthcheck（主动健康检查）、lua-resty-ipmatcher、lua-resty-openssl、kong-pgmoon（Postgres 驱动）、lyaml（YAML 声明式配置）、lua-protobuf（gRPC/protobuf）、lua-messagepack、luaxxhash、penlight 等。
- **构建系统**：Bazel（MODULE.bazel、WORKSPACE、build/），Makefile 是薄封装；含 Rust 工具链用于 ngx_wasm 模块相关构建（Makefile `install-rust-toolchain`）。
- **扩展机制**：外部插件服务器（Go/Python 进程经 RPC 挂接，kong/runloop/plugin_servers/）与 Wasm 过滤器（kong/runloop/wasm.lua）。
- **测试框架**：busted（单元/集成 spec）+ Perl prove（t/ 目录 .t 用例）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| kong/ | 核心库：init.lua（phase 钩子入口）、runloop/（请求管线）、router/、db/、pdk/、llm/、plugins/、api/、admin_gui/、clustering/、conf_loader/、templates/ |
| kong/plugins/ | 45 个 CE 内置插件，每个含 schema.lua + handler.lua（+ migrations/） |
| kong/pdk/ | 插件开发工具箱：request/response/service/client/log/router/nginx/tracing/vault 等 16+ 模块 |
| kong/llm/ | AI Gateway 内核：drivers/（9 provider）、adapters/（bedrock/gemini）、plugin/（ai 插件公共基座与 SSE shared-filters）、schemas/ |
| kong/db/ | DAO 层：schema/（实体校验）、strategies/{off,postgres}、declarative/（声明式配置导入导出）、migrations/ |
| kong/cmd/ | CLI：start/stop/reload/restart/health/config/migrations/hybrid/vault |
| kong/api/、kong/admin_gui/ | Admin REST API 与 Kong Manager 入口 |
| spec/ | busted 测试：01-unit、02-integration、03-plugins、04-perf、05-migration、06-third-party |
| t/ | prove 集成测试（124 个 .t） |
| autodoc/、changelog/、build/ | 文档自动生成、changelog 片段（unreleased 聚合机制）、Bazel 构建 |

组织评述：单仓单体（非 monorepo），Lua 模块按"框架层（runloop/router/db）— PDK 层 — 插件层"清晰分层，插件目录结构高度统一（schema + handler + migrations），便于第三方按约定扩展。

## 5. 核心架构与关键实现

### 5.1 OpenResty phase 钩子全景（kong/init.lua，2008 行）

Kong 把自身注册为 Nginx 各阶段的 Lua 处理器：`init`/`init_worker`（启动与 worker 初始化）、`ssl_certificate`/`ssl_client_hello`（证书动态选择）、`preread`（L4 流子系统）、`rewrite`、`access`、`balancer`、`response`、`header_filter`、`body_filter`、`log`、`handle_error`（函数清单见 kong/init.lua:636-1991）。每个 phase 内部再挂全局插件迭代器或按 Route/Service/Consumer 收集的插件迭代器（`execute_global_plugins_iterator`/`execute_collecting_plugins_iterator`/`execute_collected_plugins_iterator`，kong/init.lua:290-441）。

### 5.2 请求管线与路由（kong/runloop/handler.lua，1677 行）

HTTP 子系统 access 阶段的核心流程（kong/runloop/handler.lua:1195-1245）：

```lua
-- routing request
local router = get_updated_router()
local match_t = router:exec(ctx)
if not match_t then
  return kong.response.error(404, "no Route matched with those values")
end
ctx.workspace = match_t.route and match_t.route.ws_id
...
```

随后 `balancer_prepare` + `balancer_execute`（kong/runloop/handler.lua:860，转发到 kong/runloop/balancer/）选择上游节点并支持重试（upstream_retry.lua）。路由器按版本号惰性重建：`get_updated_router()` 在 `worker_consistency = strict` 时每次请求检查版本，否则由 worker 定时轮询（kong/runloop/handler.lua:470-505），保证 DB 模式下最终一致、DB-less 下零开销。

### 5.3 路由匹配三 flavor（kong/router/init.lua）

`router_flavor` 可选 `traditional`（默认实现，基于路径前缀树+正则，kong/router/traditional.lua 1833 行）、`expressions`（ATC 表达式路由，字段级匹配编译为表达式，kong/router/expressions.lua + atc.lua）、`traditional_compatible`（kong.conf.default:1713 的出厂默认，行为兼容旧版）。同一套 `Route` schema 支持十种协议：http/https/tcp/tls/udp/tls_passthrough/grpc/grpcs 等，L4 协议走 stream 子系统的 preread 管线（kong/constants.lua `protocols_with_subsystem`、kong/runloop/handler.lua:1100-1133 的 tls_passthrough 双层代理逻辑）。

### 5.4 插件体系与优先级排序

每个插件 handler 声明静态 `PRIORITY` 数值，跨插件按其排序执行：如 correlation-id 100001 > bot-detection 2500 > cors 2000 > key-auth 1250 > basic-auth 1100 > acl 950 > ai-proxy 770 > 日志类 6（kong/plugins/*/handler.lua）。运行期 kong/runloop/plugins_iterator.lua（646 行）把 phase 分为 collecting（access/preread，按 Route/Service/Consumer 三级叠加解析出该请求生效的插件集）与 non-collecting/downstream（rewrite、header_filter 等，复用 access 阶段已收集的结果），并区分 http/stream 子系统（kong/runloop/plugins_iterator.lua:25-52）。插件可用 Lua 本地开发（实现 PDK 接口），也可经 plugin_servers/ 以 Go/Python 外部进程方式接入。

### 5.5 PDK 与 AI Gateway（kong/pdk/、kong/llm/）

PDK 是插件与运行环境间的唯一契约层：`kong.request/response/service/client/log/router/nginx/ctx/tracing/vault/telemetry/cluster` 等（kong/pdk/init.lua），隔离了对 ngx API 的直接访问，便于测试与跨引擎移植。AI 侧的 provider 差异被收敛到 kong/llm/drivers/*.lua（9 个 provider 各自实现请求/响应/SSE 流适配），公共逻辑在 kong/llm/plugin/base.lua 与 shared-filters/（SSE chunk 解析、JSON 归一化、buffering），ai-proxy 等插件只需组合这些能力（kong/plugins/ai-proxy/handler.lua 仅 19 行，核心在其 schema.lua 与基座）。

## 6. 构建与运行

```bash
# 开发环境（Bazel 驱动，拉起 venv + OpenResty 构建）
make dev                # Makefile: install-rust-toolchain build-venv install-dev-rocks

# 测试
make test               # busted 单元
make test-integration   # prove 集成
make lint               # 代码检查

# 运行（生产通常用官方 Docker 镜像或 KONG_DATABASE=postgres compose）
bin/kong start          # kong/cmd/start.lua，读 kong.conf
bin/kong reload         # 平滑重载
echo "database = off\ndeclarative_config = kong.yml" >> kong.conf  # DB-less
```

DB-less 下 `bin/kong config db_import`/`kong config parse` 处理声明式文件（kong/cmd/config.lua）；Hybrid 模式经 `kong hybrid` 生成证书（kong/cmd/hybrid.lua）。

## 7. 测试与工程质量

- 单元/插件 spec 共 472 个 busted 文件，覆盖 conf_loader、router、balancer、PDK 各 phase（spec/01-unit/08-router_spec.lua、09-balancer/ 等）；集成用例 124 个 .t（t/）；另有 04-perf 与 06-third-party 专项目录（spec/）。
- CI 共 25 个 workflow（.github/workflows/）：build_and_test.yml（构建+测试矩阵）、autodocs.yml（文档自动生成）、changelog-validation.yml（强制每个 PR 带 changelog 片段）、labeler-v2/label-schema（打标）、upgrade-tests.yml（升级路径测试）、backport 自动化（backport-v2.yml、cherry-picks-v2.yml）。
- 质量观察：changelog 采用 unreleased 片段聚合机制（changelog/unreleased/），追溯规范好；kong/constants.lua、meta.lua 等带版本断言测试；CE 插件均有独立 spec。文档注释采用 LDoc（kong/pdk/init.lua 内嵌 @usage 示例）。

## 8. 社区与维护状态

- 活跃度：master 最近 push 2026-10-19（gh api），tarball 快照同日；最新 release 3.9.4 于 2026-10-12 发布，3.x 系列保持每月级补丁节奏（CHANGELOG.md TOC：3.9.0→3.9.4 连续小版本）。
- 贡献者 335 人（gh api contributors 分页统计），由 Kong Inc. 主导开发，社区 PR 经 labeler/assignee 机器人流程管理（.github/workflows/auto-assignee.yml、label-community-pr.yml）。
- 议题 228 个开放（gh api），issue 模板与 SECURITY.md、CODE_OF_CONDUCT.md、COMMUNITY_PLEDGE.md 齐备。
- 文档与商业化投入（docs.konghq.com、developer.konghq.com AI 文档）由 Kong Inc. 维护，开源核心与闭源产品双轨并行（README.md）。

## 9. 亮点与风险

**亮点**
1. 全 phase 覆盖的插件运行时：从 TLS 握手（ssl_certificate）到 L4 preread 再到 body_filter/log，任何横切逻辑都有确定挂点（kong/init.lua、kong/runloop/handler.lua）。
2. 路由器版本化热更新设计：worker 间以版本号对齐、strict/eventual 一致性可选，兼顾正确性与性能（kong/runloop/handler.lua:470-505）。
3. DB-less 声明式 + Hybrid 模式补齐云原生部署形态，配置即文件、可 GitOps（kong/db/declarative/、kong/clustering/）。
4. AI Gateway 架构干净：provider driver 与插件基座分离，SSE 流式解析做成 shared-filters 复用（kong/llm/）。
5. PDK 抽象层使插件不直接触碰 ngx API，外部插件（Go/Python）与 Wasm 过滤器扩展通道并存。

**风险/局限**
1. 数据库仅剩 Postgres（strategies/ 目录），且声明式模式功能子集化，传统 DB 模式下 Admin API 写入与路由重建的延迟仍需调优。
2. 商业边界持续上移：管理 UI 完整版、RBAC、部分认证插件（如 OIDC）不在 CE 仓库，开源用户能力预期需对齐官方产品矩阵（README.md Konnect 节）。
3. 构建 Bazel + Rust 工具链门槛高，本地从源码构建 OpenResty 对贡献者不友好（Makefile、MODULE.bazel）。
4. Lua 生态的内存/调试限制使其复杂插件（如 oauth2、proxy-cache）排障成本高于 Native 代理（如 Envoy/APISIX 对比视角，属通用观察，未在仓库内验证）。

## 10. 总结评价

Kong 是 API 网关领域的标杆项目：以 OpenResty 为底座，用规整的 phase 钩子 + 静态优先级插件体系 + PDK 契约层构建出高扩展性网关内核，并率先把 LLM/MCP 流量治理产品化。代码分层清晰、注释与测试充分，是学习"如何在 Nginx 上构建可编程代理"的最佳样本之一。适合负责 API 基础设施的平台工程师选用，适合网关/中间件开发者精读 kong/init.lua、runloop/handler.lua、router/ 与 plugins_iterator.lua 四处源码；需要企业级治理（RBAC、多租户管理界面）或 SaaS 控制面的团队则应评估 Kong Enterprise/Konnect。
