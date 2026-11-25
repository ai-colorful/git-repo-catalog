# pocketbase/pocketbase 项目分析

> 星数 61560（快照 2026-10-24）｜主语言 Go｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：把 SQLite + REST API + 实时订阅 + 文件管理 + 管理后台打包进单个可执行文件的开源后端（"Backend in 1 file"）。

| 项目 | 事实 |
|---|---|
| 名称 | pocketbase/pocketbase |
| 星数 | 61611（gh api 实时值，快照 61560） |
| 主语言 | Go（`go 1.27`，见 `go.mod`） |
| 许可证 | MIT（`LICENSE.md`，Copyright 2022-present Gani Georgiev） |
| 最新 release | v0.40.6（2026-10-23，gh api） |
| 最近 push | 2026-10-26（gh api） |
| 仓库状态 | **GitHub 仓库已 archived**，迁移至 ganigeorgiev.com 自托管（README 顶部声明 + CHANGELOG.md v0.40.6） |
| 默认分支 | master（gh api） |

## 2. 项目是什么，解决什么问题

PocketBase 的目标是把"起一个后端"的成本压缩到零：下载一个可执行文件，运行 `./pocketbase serve`，即得到一个带嵌入式 SQLite 数据库、REST-ish API、文件上传、用户/认证体系、实时订阅（SSE）和管理后台 UI 的完整后端（`README.md` Overview 部分）。它直接瞄准 Firebase/Supabase 这类 BaaS 在自托管小项目场景的替代位。

它有两种使用形态。第一种是独立应用：官方预编译二进制基于 `examples/base/main.go` 构建，默认启用 JS VM 插件，用户把业务逻辑写在 `pb_hooks/*.pb.js` 里即可扩展。第二种是 Go 框架/toolkit：`pocketbase.New()` 返回的实例实现了 `core.App` 接口，用户在 `OnServe()` 等 hook 里注册自己的路由和业务逻辑，最终仍编译出单个静态可执行文件（README "Use as a Go framework/toolkit" 一节示例代码）。

数据模型是"动态 schema"：开发者不写建表 SQL，而是通过管理后台或 API 定义 Collection（base/auth/view 三类，见 `core/collection_model.go` 顶部常量），每个 Collection 由一组 Field 定义（14 种字段类型），PocketBase 负责把 Collection 定义同步成真实的 SQLite 表结构（`core/collection_record_table_sync.go`）。记录（Record）本身不对应固定 Go 结构体，而是"字段名→值"的动态容器（`core/record_model.go`）。

工程上最有意思的决策是"零外部 web 框架 + 纯 Go SQLite 驱动"：HTTP 路由、hook 链、搜索过滤 DSL、SSE broker 全部自己实现（`tools/` 目录），SQLite 使用 `modernc.org/sqlite`（CGO-free），因此 `CGO_ENABLED=0 go build` 即可交叉编译到几乎所有平台（`.goreleaser.yaml` 的构建矩阵、README 支持平台表）。

需要注意的是，项目在 GitHub 上已归档：README 顶部标注 "The repo has moved to https://ganigeorgiev.com/pocketbase/pocketbase"，v0.40.6 的 CHANGELOG 写明"这很可能是 GitHub 上最后一个 release"，后续发布将走作者自建站点。

## 3. 技术栈与依赖

- 语言：Go 1.27（`go.mod`）；管理后台 UI 为原生 JavaScript + Vite（无 Vue/React/Svelte 框架依赖，`ui/package.json`）
- 数据库：`modernc.org/sqlite` v1.60.1 —— 纯 Go（CGO-free）SQLite 驱动，是"单文件、可交叉编译"的基石（`go.mod`）
- 查询构建：`github.com/pocketbase/dbx`（自维护 fork，dbx.Builder 贯穿 `core/base.go`）
- 过滤 DSL：`github.com/ganigeorgiev/fexpr` v0.6.0（作者自研表达式解析器，`tools/search/filter.go:10`）
- JS 运行时：`github.com/dop251/goja` + `goja_nodejs`（JS 扩展插件，`plugins/jsvm/jsvm.go`）
- CLI：`spf13/cobra`（`pocketbase.go` 中 RootCmd）
- 校验：`pocketbase/ozzo-validation/v4` fork；JWT：`golang-jwt/jwt/v5`；图片处理：`disintegration/imaging` + `golang.org/x/image`；邮件：`domodwyer/mailyak/v3`；文件类型探测：`gabriel-vasile/mimetype`；热重载：`fsnotify`（均见 `go.mod`）
- 构建/发布：GoReleaser（`.goreleaser.yaml`）+ GitHub Actions（`.github/workflows/release.yaml`）
- 代码规模：非测试 Go 代码约 5.94 万行（272 个文件），含测试约 12.9 万行（`find` 统计）

## 4. 目录结构与代码组织

单仓库单 Go module，无 monorepo/多包；按"core → apis → tools → plugins"分层，依赖方向单一（apis 依赖 core，core 只依赖 tools）。

| 目录 | 职责 |
|---|---|
| `core/` | 骨架：App 接口与 BaseApp 实现、Collection/Record 模型、14 种 Field、系统模型（authOrigin/externalAuth/MFA/OTP/log/settings）、迁移、备份。139 文件 / 4.8 万行（`core/app.go` 顶部包注释自称 "the backbone of PocketBase"） |
| `apis/` | HTTP 层：路由注册、默认中间件、records CRUD、认证全套端点、realtime、backup、batch、SQL 控制台。78 文件 / 3 万行 |
| `tools/` | 自研工具箱：hook 链、router、search(fexpr)、subscriptions(SSE broker)、filesystem(local/S3)、mailer、cron、store、security、template 等 21 个子包，185 文件 / 3.4 万行 |
| `plugins/` | 可选插件：jsvm（JS 扩展）、migratecmd（迁移脚手架）、ghupdate/update（自更新） |
| `ui/` | 管理后台（"PocketBase Superuser UI"），Vite + 原生 JS，构建产物经 `ui/embed.go` 嵌入二进制，挂载在 `/_/` 路径（`apis/serve.go`） |
| `cmd/` | serve/superuser 两个系统子命令 |
| `migrations/` | 系统 migrations（含 v0.23 大版本迁移 `1717233556_v0.23_migrate.go`） |
| `examples/base/` | 预编译发行版的入口 `main.go` |
| `forms/`、`mails/`、`tests/` | 请求表单校验、邮件模板、测试基础设施 |

组织方式评述：核心库与发行版入口分离（`examples/base`），admin UI 以 embed 方式随二进制分发；测试文件与源码同目录（`*_test.go`，189 个），公共测试设施单独放 `tests/`。

## 5. 核心架构与关键实现

### 5.1 App 抽象与全 hook 驱动（`core/app.go`、`core/base.go`）

`App` 接口约 1581 行，覆盖引导（Bootstrap/ClearBootstrap）、双数据库句柄、设置、cron、SSE broker、邮件客户端、文件系统，以及一整套 `OnXxx()` hook 注册点（OnBootstrap、OnServe、OnModelValidate/Create/Update/Delete 全生命周期、OnCollectionXxx、OnRecordXxx、OnRealtimeXxx、OnTerminate 等）。`BaseApp`（`core/base.go`）是唯一实现，用户结构体通过嵌入复用（`pocketbase.go:29` 的 `var _ core.App = (*PocketBase)(nil)`）。

双库设计：`data.db`（业务数据，区分 ConcurrentDB/NonconcurrentDB 两个连接池）与 `auxiliary.db`（日志等辅助数据），事务内两者指向同一个 `*dbx.TX`（`core/base.go:525-592` 注释）。事务采用"替换 App"模式——`RunInTransaction` 内部创建一个绑定 tx 的 `txApp` 传给回调，天然支持嵌套（`core/db_tx.go:14-34`）：

```go
func (app *BaseApp) RunInTransaction(fn func(txApp App) error) error {
	return app.runInTransaction(app.NonconcurrentDB(), fn, false)
}
...
case *dbx.DB:
	var txApp *BaseApp
	txErr := txOrDB.Transactional(func(tx *dbx.Tx) error {
		txApp = app.createTxApp(tx, isForAuxDB)
		return fn(txApp)
```

### 5.2 Collection/Record 动态模型（`core/collection_model.go`、`core/record_model.go`）

Collection 本身也是存库的 Model（`_collections` 表），分 base/auth/view 三类。关键机制是"hook 桥接"：`registerCollectionHooks()` 把泛化的 OnModelCreate 等事件在运行期转换为 OnCollectionCreate 等具体事件（`core/collection_model.go:30-80`，桥接 handler 统一挂 `Priority: -99` 保证先于用户逻辑执行），Record 侧同理（`core/record_model.go` 的 `systemHookIdRecord`）。Collection 变更落库时由 `core/collection_record_table_sync.go` 把 Fields 同步为 SQLite 表结构。

Record 是非固定结构体：`Record` 持有 `collection *Collection` 和 `data/expand/originalData` 等 `store.Store` 动态容器（`core/record_model.go:41-53`），字段校验完全由该 Collection 的 Fields 列表驱动。Field 体系为接口 + 14 个实现（`core/field.go` 的 `Field` 接口与 `Fields` 注册表；`core/field_text.go`、`field_file.go`、`field_relation.go`、`field_geo_point.go` 等），新增字段类型只需实现接口并注册。

### 5.3 API 路由组织（`apis/base.go`、`apis/record_crud.go`、`apis/record_auth.go`）

路由为自研类型化 router（`tools/router/router.go`，泛型 `RouterGroup[*core.RequestEvent]`，配合 Go 原生 `r.PathValue`）。`apis/base.go:16-49` 的 `NewRouter` 集中装配：先绑定 7 个默认中间件（activityLogger、panicRecover、rateLimit、loadAuthToken、superuserIPsWhitelist、securityHeaders、bodyLimit），再以 `/api` 为根挂 12 个 bind 函数（settings/collections/recordCrud/recordAuth/logs/backup/cron/files/batch/realtime/health/sql）。核心 CRUD 只有一组五条路由（`apis/record_crud.go:28-33`）：

```go
subGroup := rg.Group("/collections/{collection}/records").Unbind(DefaultRateLimitMiddlewareId)
subGroup.GET("", recordsList)
subGroup.GET("/{id}", recordView)
subGroup.POST("", recordCreate(true, nil)).Bind(dynamicCollectionBodyLimit(""))
subGroup.PATCH("/{id}", recordUpdate(true, nil)).Bind(dynamicCollectionBodyLimit(""))
subGroup.DELETE("/{id}", recordDelete(true, nil))
```

认证端点全部挂在 `/api/collections/{collection}/` 前缀下（password/oauth2/otp 三种登录 + 密码重置/验证/换邮箱的 request-confirm 对 + superuser impersonate），每个端点独立绑定按 action 命名的限流（`apis/record_auth.go:17-69`）。列表查询的 filter/sort 语法由 `tools/search/filter.go` 基于 fexpr 解析并缓存（`parsedFilterData` store）。

### 5.4 实时订阅与 JS 扩展（`apis/realtime.go`、`tools/subscriptions/`、`plugins/jsvm/`）

Realtime 走 SSE：`realtimeConnect` 设置 `text/event-stream` 头与 `X-Accel-Buffering: no`，先发 `PB_CONNECT` 消息，连接空闲超时 5 分钟、最长 30 分钟，可被 `OnRealtimeConnectRequest` hook 改写（`apis/realtime.go:43-110`）。连接统一注册进 `tools/subscriptions/broker.go` 的内存 Broker（`Broker struct { store *store.Store[string, Client] }`），记录变更事件经 `bindRealtimeEvents` 广播给订阅者。

JS 扩展插件（预编译版默认启用）：`plugins/jsvm/jsvm.go` 用 goja 跑 `pb_hooks/*.pb.js`，通过 fsnotify 监听文件变化热重启（`HooksWatch`），并预热一个 runtime 池复用实例（`plugins/jsvm/pool.go`）；`plugins/jsvm/binds.go`（1274 行）把 `$app`、`$os` 等 API 桥接给 JS。

## 6. 构建与运行

```bash
# 作为发行版构建（与官方预编译产物等价）
cd examples/base
CGO_ENABLED=0 go build          # 交叉编译：GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build
./base serve

# 作为 Go 库使用
go mod init myapp && go mod tidy   # main.go 见 README "Use as a Go framework/toolkit"
go run main.go serve

# 仓库内常用命令（Makefile）
make test        # go test ./... -v --cover
make lint        # golangci-lint run -c ./golangci.yml ./...
make jstypes     # 生成 JSVM 的 types.d.ts

# 管理后台 UI 开发/构建（ui/）
npm --prefix=./ui ci && npm --prefix=./ui run build   # 产物 dist/ 由 ui/embed.go 嵌入
```

来源：`README.md`、`Makefile`、`.github/workflows/release.yaml`、`.goreleaser.yaml`。

## 7. 测试与工程质量

- 测试规模：189 个 `*_test.go` 与 272 个源码文件同目录分布，几乎所有 core/apis/tools 包都有对应测试（如 `core/collection_model_test.go`、`apis/record_crud_test.go`、`tools/hook` 等）；测试基础设施集中在 `tests/`（mock app、请求构造、邮件 stub、dynamic_stubs）。
- 测试风格：以 go test 标准库为主的单元/集成测试，覆盖行为细节（如 MFA、OTP、auth origin 等专项测试文件）；无 e2e 浏览器测试（未确认，未见对应目录）。
- CI：`.github/workflows/release.yaml` 在 PR/push 时构建 admin UI → `go test ./...` → GoReleaser（非 tag 时 `--snapshot`），单文件覆盖测试与发布。
- 质量观察：`golangci.yml` 启用 staticcheck、unconvert、prealloc 等 15 个 linter；源码注释极其详尽（App 接口每个方法都有语义说明和坑位提示）；CHANGELOG 记录粒度到单个 issue，安全修复（Go CA、net/http DoS）响应迅速（`CHANGELOG.md` v0.40.4-0.40.6）。

## 8. 社区与维护状态

- 活跃度：最近 push 2026-10-26，最新 release v0.40.6 于 2026-10-23 发布（gh api），截取的 tarball 文件 mtime 为 2026-10-27，处于高活跃状态。
- 仓库迁移：GitHub 仓库已 **archived**，CHANGELOG v0.40.6 明确"这很可能是 GitHub 上最后一个 release，后续更新从 https://ganigeorgiev.com 获取"（`CHANGELOG.md`、`README.md` 顶部、`plugins/update` 插件对应改造）。
- 贡献者：contributors API 分页显示约 45 位（per_page=1 时 last page=45，含匿名），主体由作者 Gani Georgiev 主导——多个 v0.2x 大版本（v0.23 迁移等）均为个人重写（`migrations/1717233556_v0.23_migrate.go`；贡献者结构细节为推断）。
- release 节奏：v0.40.x 系列在 2026-10 内连发多个 patch（CHANGELOG.md），0.x 阶段迭代快、大版本有破坏性变更（v0.23 专用迁移文件佐证）。

## 9. 亮点与风险

**亮点**

1. "单文件后端"体验做到极致：纯 Go SQLite 驱动 + admin UI embed + GoReleaser 多平台矩阵（`.goreleaser.yaml`），`CGO_ENABLED=0` 即可全平台交叉编译。
2. 全 hook 驱动的可扩展内核：从 HTTP 路由、模型 CRUD 到 SSE 消息全部是可绑定/可摘除的泛型 hook 链（`tools/hook/hook.go`，Handler 带 Id/Priority），API 设计对 Go 库用户非常友好。
3. 动态 schema 的完整闭环：Collection 定义 → 校验 → SQLite 表同步 → 记录 CRUD → filter DSL（fexpr）→ realtime 广播，各环节代码路径清晰、测试充分。
4. 自研工具链质量高：类型化 router、fexpr 过滤语言、SSE broker 均零重型依赖，核心代码可读性好。
5. JS 扩展降低门槛：goja + runtime 池 + fsnotify 热重载，让不会 Go 的用户也能写业务钩子（`plugins/jsvm/`）。

**风险/局限**

1. GitHub 仓库已 archived，文档/发布/更新渠道迁往 ganigeorgiev.com，依赖 GitHub 镜像或 forks 衍生的生态存在断链风险（README、CHANGELOG v0.40.6）。
2. v1.0 未达成，官方明示不保证向后兼容，大版本（如 v0.23）需要数据迁移。
3. 单机 SQLite 架构天然不支持水平扩展/多实例写入；缩略图生成的内存占用是作者自认的 OOM 隐患（CHANGELOG v0.40.6）。
4. 强个人主导（bus factor 偏低），仓库迁移后的维护透明度与社区治理方式待观察（未确认）。

## 10. 总结评价

PocketBase 是"小而全"后端赛道的标杆实现：用不到 6 万行非测试 Go 代码，把动态 schema 数据库、认证体系、文件存储、SSE 实时、管理后台和插件机制装进一个零 CGO 的可执行文件，且每一层都开放为可编程 hook——既是一个开箱即用的产品，也是一个值得精读的 Go 库设计范例。适合个人项目、原型、中小规模自托管应用直接使用；适合想学习"如何用标准库+少量依赖构建完整后端框架"的 Go 开发者精读 `core/`（App 抽象与 hook 桥接）和 `apis/base.go`（路由装配）。唯一需要正视的是其 GitHub 仓库已归档迁移，选用时应确认新渠道的发布与维护承诺。
