# usememos/memos 项目分析

> 星数 64015（快照 2026-10-24）｜主语言 Go｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：开源自托管的轻量笔记/备忘录应用——把日常想法以 Markdown 时间线形式记录在自己的服务器上，"flomo 的可自托管替代品"。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | usememos/memos | GitHub |
| 星数 | 64015（快照），gh api 实时 64192 | 任务参数 / `gh api repos/usememos/memos` |
| 主语言 | Go（后端）+ TypeScript（前端），550 个 .go 文件、634 个 .ts/.tsx | 源码统计 |
| 许可证 | MIT | `LICENSE` |
| 创建时间 | 2021-12-08 | `gh api` |
| 最近 push | 2026-10-31（分析当日仍活跃） | `gh api` |
| 最新 release | v0.31.0（2026-09-19），新版本号方案 `YY.MM.N` | `gh api`、`README.md` |
| 贡献者 | 约 403 人（contributors 分页 last=404 推断） | `gh api` Link header |
| 官网 | https://usememos.com | `README.md` |

## 2. 项目是什么，解决什么问题

Memos 是一个部署在自己服务器上的"个人想法时间线"：一句话也能成为一条 memo，长文则用 Markdown 组织。每条 memo 归属于写入当天，标签（tag）把跨日期的想法串起来（`README.md` "About Memos" 节）。

它解决的核心问题是"数据主权 + 低门槛记录"：所有数据存放在自己的服务器（Docker 部署下所有内容集中在 `~/.memos` 一个目录，`README.md` Quick Start），同时分享粒度可控——单条 memo 可设为 PUBLIC / PROTECTED / SPACE / PRIVATE（`store/memo.go` 的 `Visibility` 定义），还有 Space（多人共享空间）但每人保留自己的 memo。

工程形态是单二进制 + 单数据目录：默认 SQLite（纯 Go 实现，免 CGO），可切换 MySQL/PostgreSQL（`go.mod` 依赖 `modernc.org/sqlite`、`go-sql-driver/mysql`、`lib/pq`；`cmd/memos/main.go` 默认 `driver: sqlite`）。默认端口 8081（配置），Docker 镜像对外 5230（`README.md`）。

与早期版本相比，当前快照已演进为一个 API 优先的产品：全部接口由 protobuf 定义（`proto/api/*.proto`），前端走 Connect RPC，官方提供 API 文档、Web Clipper 浏览器扩展、Telegram bot（Memogram）与 MCP server 接入 AI 工具（`README.md` "Around Memos" 节），并内置 AI 能力（语音转写、AI provider，见 `server/api/ai_service.go`、`provider/ai`）。

仓库对 AI 协作开发也做了工程化：根目录 `AGENTS.md`（AI 编码 agent 的仓库说明）、`CONTEXT.md`、`.agents/skills/release/`，这在大型开源仓库中比较少见。

## 3. 技术栈与依赖

后端（`go.mod`，Go 1.27.2）：

| 依赖 | 用途 |
| --- | --- |
| `connectrpc.com/connect` | Connect RPC（gRPC 兼容的 API 协议层） |
| `grpc-ecosystem/grpc-gateway/v2` | HTTP/JSON 兼容网关 |
| `google.golang.org/grpc` + `protobuf` | API 传输与序列化 |
| `modernc.org/sqlite` / `go-sql-driver/mysql` / `lib/pq` | 三种数据库驱动（SQLite 为纯 Go 无 CGO） |
| `google/cel-go` | 用户过滤表达式（CEL）编译执行 |
| `modelcontextprotocol/go-sdk` | 内置 MCP server |
| `openai/openai-go/v3`、`google.golang.org/genai` | AI 提供方（转写/文本） |
| `aws/aws-sdk-go-v2`（S3）、`johannesboyne/gofakes3` | 附件对象存储及本地 S3 仿真 |
| `yuin/goldmark` | Markdown 基础引擎（在其上自建 memos 语法扩展） |
| `golang-jwt/jwt/v5`、`x/oauth2` | 认证（JWT、OAuth/SSO） |
| `spf13/cobra` + `viper`、`joho/godotenv` | CLI 与配置 |
| `testcontainers-go`（mysql/postgres 模块） | 集成测试 |

前端（`web/package.json`，pnpm 11 / Node >= 24）：

| 依赖 | 用途 |
| --- | --- |
| React 19 + react-router-dom 7 | UI 与路由 |
| `@connectrpc/connect` + `@bufbuild/protobuf` | 与后端 Connect API 通信 |
| TanStack React Query v5 | 数据获取/缓存 |
| Tailwind CSS v4 + `@base-ui/react` | 样式与基础组件 |
| Vite 8（rolldown/babel-react-compiler） | 构建 |
| Biome + tsc | Lint/格式化（`web/package.json` scripts） |
| Vitest + Testing Library | 前端测试 |
| i18next、katex、leaflet/maplibre、exifr、dompurify | i18n、公式、地图位置、EXIF、XSS 清洗 |
| `@usememos/editor`（独立包，26.10.30） | 官方维护的编辑器组件 |

## 4. 目录结构与代码组织

顶层目录（`ls src`）：

| 目录 | 职责 |
| --- | --- |
| `cmd/memos/` | 程序入口，cobra/viper 配置、日志（`main.go`、`config.go`） |
| `server/` | HTTP 服务组装：`server.go` 建 mux；`server/api/` 为 Connect/gRPC 服务实现（约 100 个文件）；`server/mcp/` MCP server；`server/fileserver/` 附件服务；`server/frontend/` 前端静态资源 |
| `core/` | 业务域模块：`user`、`memoexport`、`memopayload`、`notification`、`usagelimit`、`identityprovider`、`configuration`、`access` |
| `store/` | 数据层：`Driver` 接口 + 三种实现（`store/db/{sqlite,mysql,postgres}`）、进程内缓存（`store/cache`）、迁移（`store/migration`）、种子数据（`store/seed`） |
| `proto/` | `api/` 11 个公开 API proto；`store/` 内部存储 proto；`gen/` 生成代码（Go/OpenAPI/TS） |
| `markdown/` | 自研 Markdown 引擎：AST（`ast/`）、memos 扩展语法（`extensions/`、`parser/`）、渲染与派生 payload（`markdown/doc.go` 注释明确分层约束） |
| `filter/` | CEL 过滤器引擎：解析→IR→方言无关条件树→SQL 渲染（`filter/engine.go`、`filter/render.go`） |
| `provider/` | 外部集成：`ai/`（AI 提供方）、`idp/`（身份提供方）、`storage/`（S3 等） |
| `internal/` | 横向基础设施：`httphandler`、`ratelimit`、`clientip`、`linkmeta`、`webhook`、`email`、`profile`、`imagelimit`、`motionphoto` 等 |
| `web/` | React 前端（`src/{components,pages,layouts,router,hooks,contexts,themes,locales}`，`tests/`） |
| `.github/` | CI workflows、issue 模板、复用的 setup actions |

组织方式评述：单体 Go 模块（`module github.com/usememos/memos`）+ 前端子目录，非 monorepo 工具链。分层清晰且在代码里显式声明约束：`markdown/doc.go` 注明 markdown 层"不得 import store/core/server"；API 面由 proto 单一事实源驱动，生成 Go/TS 双侧代码（`AGENTS.md` Working Rules：禁止手改生成物，改 `.proto` 后 `buf generate`）。业务编排集中在 `server/api`（薄网关风格，委托给 store/core），`core/` 承载独立领域逻辑。

## 5. 核心架构与关键实现

### 5.1 服务组装与 API 组织（server/server.go、server/api/api.go）

单进程多角色：`server/server.go` 的 `NewServer` 用标准库 `http.ServeMux` 组装 `/healthz`（带数据库 ping，2 秒超时）、API handler、附件 fileserver、前端静态服务，外层套 Recover → clientIP → CORS 中间件，并启动后台任务（`backgroundJobs`）。细节可看出成熟度：`readHeaderTimeout=15s` 防慢连接占满 worker，SSE/上传路径不受写超时影响（`server/server.go` 常量注释）。

API 采用"双协议单实现"：`proto/api/` 定义 8 个服务（Instance/Auth/User/Memo/Space/Attachment/AI/IdentityProvider，见 `server/api/api.go` 中 APIService 内嵌的 Unimplemented 列表）。gRPC 侧由 `APIService` 直接实现；Connect 侧 `server/api/connect_services.go` 的 `ConnectServiceHandler` 逐方法委托给 gRPC 实现；HTTP/JSON 兼容由 grpc-gateway 提供（`server/api/http_compat_test.go` 验证双协议一致性）。请求体按 procedure 分级限额：普通 16MB、内联文件（附件/转写）256MB（`api.go` 的 `requestBodyLimit`）。鉴权白名单集中在 `server/api/acl_config.go`（`AGENTS.md` 要求新增端点必须登记）。另有 `server/api/sse_hub.go` 提供 SSE 实时推送、`server/mcp/` 把 MCP 工具调用经同一套 API 路由转发（`api.go` 注释：MCP 与 API 限额保持同步）。

### 5.2 Store 层与迁移（store/driver.go、store/store.go、store/migrator.go）

数据访问收敛到一个巨型 `Driver` 接口（`store/driver.go`：Attachment/Memo/Space/Relation/InstanceSetting/User 等约 50 个方法），三种数据库各自实现于 `store/db/{sqlite,mysql,postgres}`。`Store` 结构体（`store/store.go`）在驱动之上叠加：进程内 TTL 缓存（默认 10 分钟、上限 1000 条，分别缓存 instance settings、users、user settings）、按 generation 失效的对象存储客户端缓存、以及每域一把互斥锁（`userCreateMu`、`refreshTokenMu` 等）串行化敏感写路径。

Memo 领域模型（`store/memo.go`）：系统 ID（int32）+ 用户自定义 UID（`shortuuid`）；内容派生数据（tags、properties、snippet）不在表列里，而是存进 `Payload *storepb.MemoPayload`（proto 序列化 JSON）。这些派生数据由 `core/memopayload/runner.go` 维护：定义 `Version = 1`，升级后启动时 `Backfill` 按 100 条/批重建全部 memo 的 payload，只有整轮成功才记录版本号，中断可重入且"边服务流量边跑"。这套"派生 payload + 版本化回填"是该仓库最有特色的设计之一。

迁移系统按驱动分目录（`store/migration/{sqlite,mysql,postgres}`），SQLite 下为版本目录（如 `26.10/`）+ `LATEST.sql` 基线（`store/migration/sqlite/`），配合 `store/schema_version.go`、`store/migrator.go` 执行；`AGENTS.md` 明确要求改 schema 必须三驱动同步并更新各自 `LATEST.sql`。

### 5.3 CEL 过滤引擎（filter/）

`filter/engine.go` 把用户在 UI 上写的过滤表达式（CEL 方言）编译为方言无关的条件树（IR），再由 `filter/render.go` 渲染成各数据库的 SQL 谓词——避免把用户输入拼 SQL，也避免为三种数据库手写查询。防护措施明确：表达式长度上限 8192 code points（`maxFilterExpressionSize`），`now` 变量在单次编译中冻结保证确定性，`filter/schema.go` 定义允许的字段集。`server/api/memo_service_filter.go` 将其接入 memo 列表查询，`FindMemo.Filters`（`store/memo.go`）以字符串表达式存储、入库前编译。

### 5.4 Memo 模型与 API 面（proto/api/memo_service.proto）

`Memo` 资源遵循 Google AIP 风格：资源名 `memos/{memo}`（name 为 IDENTIFIER，用户可自定义 UID）、`creator` 引用 `users/{user}`、可选 `space` 引用；字段级 `field_behavior` 注解标注 REQUIRED/OPTIONAL/OUTPUT_ONLY（`proto/api/memo_service.proto:203` 起）。派生字段（tags、property、snippet、parent）全部 OUTPUT_ONLY，由服务端从 Markdown 内容计算；`Property` 记录 has_link/has_code/has_task_list 等布尔特征供前端筛选。关联能力完整：attachments、双向 relations（REFERENCE 可设、COMMENT 只读）、reactions、shares，以及独立的评论接口（`CreateMemoCommentRequest`）。

### 5.5 自研 Markdown 引擎（markdown/）

在 goldmark 之上建了完整的 in-house 管线：`markdown/ast` 自有 AST、`markdown/extensions`+`markdown/parser` 实现 memos 专属语法（标签、@提及、数学公式、GFM 扩展），`markdown/renderer` 输出，最终产出 memo 的派生 payload（tags/properties/snippet，`markdown/doc.go`）。这是把"产品语法"与通用 Markdown 解耦的典型做法，也让 payload 重建（5.2）成为可能。

## 6. 构建与运行

后端（来源：`go.mod`、`AGENTS.md`、`.github/workflows/backend-tests.yml`）：

```bash
go build ./...            # 编译
go test ./...             # 测试（CI 按 test-group 矩阵分片）
golangci-lint run         # 静态检查（CI Static Checks job）
go run ./cmd/memos        # 本地启动（默认 driver=sqlite，端口 8081）
```

前端（来源：`web/package.json` scripts）：

```bash
pnpm install
pnpm dev                  # vite 开发服务
pnpm release              # 构建产物输出到 ../server/frontend/dist（内嵌进二进制）
pnpm lint                 # tsc --noEmit + biome check
pnpm test                 # vitest run
```

proto 生成（`AGENTS.md`）：改 `.proto` 后 `buf generate`（生成 `proto/gen/` 与 `web/src/types/proto/`）。

生产运行（`README.md` Quick Start）：

```bash
docker run -d --name memos -p 5230:5230 -v ~/.memos:/var/opt/memos neosmemo/memos:stable
```

## 7. 测试与工程质量

- 测试规模：后端 210 个 `*_test.go`，前端 192 个 `*.test.*`（源码统计）。覆盖面不止单元测试：`server/api` 下有大量行为测试（鉴权、限流、EXIF 嗅探、webhook、上传传输等），`store/db` 与 `server/test` 用 testcontainers 起 MySQL/PostgreSQL 做真库集成测试（`go.mod` testcontainers 依赖）。
- CI（`.github/workflows/`，共 8 个 workflow）：`backend-tests.yml`（静态检查：go.mod tidy 校验、go fix、golangci-lint + 测试矩阵分片）、`frontend-tests.yml`、`proto-linter.yml`（buf lint）、`release.yml`、`build-canary-image.yml`（canary Docker 构建）、`demo-deploy.yml`、`upgrade-smoke.yml`（升级冒烟——对自托管产品是少见的认真做法）、`stale.yml`。
- 代码质量观察：单文件体量克制（核心文件多在 500-800 行内）；错误处理统一 `errors.Wrap`；并发保护显式（`store/store.go` 锁字段注释清楚）；资源限额普遍内建（请求体分级限额、CEL 表达式长度、rate limiter 可选注入 `server/api/api.go`）；文档自持（`store/migration/README.md`、`filter/README.md`、`server/mcp/README.md`、根 `AGENTS.md`/`CONTEXT.md`/`SECURITY.md`）。

## 8. 社区与维护状态

- 活跃度：gh api 显示 pushed_at 为 2026-10-31（分析当日），open issues 仅 59，说明 issue 消化很快；fork 4922。
- Release 节奏：最新 v0.31.0（2026-09-19）；README 宣布新版本号方案 `YY.MM.N`（如 26.10），Docker 标签分 `stable`（跟随稳定版）、`26.10` 月度标签、`canary`（开发构建），并有 `upgrade-smoke.yml` CI 与 `store/migration/README.md` 升级要求（从 v0.31.0 之前升级需先过 v0.31.0）——月度级发布节奏。
- 贡献者结构：约 403 名贡献者（gh api 分页推断）；README 显示有商业化信号（CodeRabbit、TestMu AI 赞助横幅，GitHub Sponsors），Web Clipper 为独立发布到 Chrome/Firefox 商店的官方周边。
- 仓库本身已为 AI 协作开发做治理（`AGENTS.md`、`CONTEXT.md`、`.agents/skills/release/`、CodeRabbit 评审），维护方式现代化。 CONTRIBUTORS 具体分布未逐个确认。

## 9. 亮点与风险

亮点：

1. 架构纪律性强：proto 单一事实源 + 生成 Go/TS 双端代码 + `markdown/doc.go` 等处显式分层禁令 + ACL 白名单集中登记，跨协议（gRPC/Connect/HTTP JSON/MCP）共享一套实现（`server/api/`）。
2. 派生 payload 设计：内容派生数据（tags/property/snippet）版本化回填、幂等可重入、可在线重建（`core/memopayload/runner.go`），升级演进成本低。
3. 三数据库平权 + 纯 Go SQLite：默认零依赖单二进制（modernc.org/sqlite 免 CGO），MySQL/PG 用 testcontainers 真库测试保障（`go.mod`、`.github/workflows/backend-tests.yml`）。
4. 测试与 CI 成熟：210+192 个测试文件、协议一致性测试（`http_compat_test.go`）、升级冒烟流水线（`upgrade-smoke.yml`）。
5. 生态外延完整：Web Clipper、Telegram bot、MCP server、官方编辑器独立包，形成"产品 + API + 工具"的完整面。

风险/局限：

1. 演进激进、升级摩擦：模型已从旧版"rowStatus+content 表"演进到 Space/CEL/payload 架构，README 明示跨 v0.31.0 需要中间版本迁移；对已部署用户是持续成本。
2. `store/driver.go` 巨型接口（约 50 方法）：三种数据库各实现一遍，新增存储功能需三处同步（AGENTS.md 也承认此约束），长期是维护负担。
3. 派生 payload 存 proto JSON：强 schema 换取灵活性，但直接 SQL 查询 tags/property 需绕道（CEL 编译为 SQL 谓词缓解了主要场景）。
4. 单体 `server/api` 目录近百文件：memo 相关实现散布 `memo_*.go`/`user_service_memo_*.go` 多个文件，新人定位成本偏高。

## 10. 总结评价

Memos 是"自托管轻笔记"品类中工程质量最高的项目之一：产品面简单克制（一条时间线 + 标签 + 分享），技术面却按大型系统标准建设——proto 驱动的多协议 API、三数据库驱动、版本化迁移与 payload 回填、CEL 过滤、真库集成测试与升级冒烟 CI。64k 星与当日仍在 push 的活跃度印证了其实用性与健康维护。适合作为个人/小团队的私有 flomo 替代品使用（Docker 一行起跑、数据单目录备份）；对 Go 后端工程师，它是学习"单二进制全栈产品如何做协议层设计与数据层抽象"的优秀范本，重点可读 `server/api/api.go`、`store/driver.go`、`filter/engine.go` 与 `core/memopayload/runner.go`。
