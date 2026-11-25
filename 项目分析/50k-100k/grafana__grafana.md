# grafana/grafana 项目分析

> 星数 77451（快照 2026-10-24）｜主语言 TypeScript（Go + TypeScript 双栈）｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：开源的可观测性与数据可视化平台，对接任意数据源，提供仪表盘、告警与探索式查询。

| 事实 | 值 | 出处 |
| --- | --- | --- |
| 名称/定位 | Grafana，"The open-source platform for monitoring and observability" | `README.md` |
| 星数 | 77451（快照）；分析当日 API 实测 77585 | 任务参数 / gh api |
| 主语言 | TypeScript（前端）+ Go（后端） | `package.json`、`go.mod` |
| 许可证 | AGPL-3.0-only（附 Apache-2.0 例外说明） | `LICENSE`、`LICENSING.md` |
| 版本 | 13.4.0-pre（main 分支），最新 release v13.3.0（2026-10-27） | `package.json`、`CHANGELOG.md` |
| 最近活跃 | push 时间 2026-10-30（分析前一天），open issues 3354，forks 14885 | gh api repos/grafana/grafana |
| 创建时间 | 2013-12-11 | gh api |

## 2. 项目是什么，解决什么问题

Grafana 解决的核心问题是：监控指标、日志、链路等数据散落在各种异构存储（Prometheus、Loki、Elasticsearch、SQL 数据库、云厂商服务……）里，用户需要统一的地方做查询、可视化、告警和分享。Grafana 通过"数据源插件协议"把后端任意存储抽象成统一的查询/帧（DataFrame）接口，前端再用统一的面板体系渲染，从而实现"数据存在哪里不重要，都能画在一张图里"（`README.md`）。

产品能力上分四块：可视化面板（客户端渲染的高性能图表，见 `public/app/plugins/panel/` 下 30 种面板）；动态仪表盘（模板变量、可在顶部下拉切换，由 dashboard schema 驱动，见 `kinds/dashboard/`）；告警（`pkg/services/ngalert/`，540 个 Go 文件的多数据源告警引擎）；以及 Explore 探索式查询（metrics/logs 拆分视图、即席查询，前端 `public/app/features/` 各模块）。

架构上是典型的"Go 单体后端 + React SPA 前端 + 插件生态"。后端把每个业务域做成独立 service（共 78 个，见 `pkg/services/`），通过依赖注入组装；HTTP 层在 `pkg/api/`（`pkg/api/http_server.go` 约 1120 行）。值得注意的现状：**大部分核心数据源实现已迁出主仓库**——`pkg/tsdb/` 现在只剩 azuremonitor、graphite、grafanads（内置测试）、grafana-testdata-datasource 四个实现，前端 `public/app/plugins/datasource/` 也只剩 7 个目录（无 prometheus/loki/tempo 等），说明 Prometheus、Loki 等数据源已拆分到独立仓库、以插件形式安装（据目录结构推断，`pkg/tsdb/`、`public/app/plugins/datasource/`）。

另一个重要趋势是"App Platform"改造：仓库正把各业务域迁移到 Kubernetes 风格的 APIServer 之上。`apps/` 下 24 个应用（dashboard、folder、iam、alerting 等）各自是独立 Go module，带 `kinds/`（CUE 定义）与 `v0alpha1/v1beta1/v2` 多版本 API（`apps/dashboard/`、`go.work`）；连数据源服务本身也开始依赖 `k8s.io/apimachinery` 与 `pkg/apis/datasource/v0alpha1`（`pkg/services/datasources/service/datasource.go`）。

## 3. 技术栈与依赖

| 层 | 技术 | 出处 |
| --- | --- | --- |
| 后端 | Go 1.27.3，google/wire 依赖注入（`pkg/server/wire_gen.go`），grafana-plugin-sdk-go 插件 SDK | `go.mod`、`go.work` |
| 前端 | React + TypeScript，pnpm workspace（deps 157 / devDeps 158） | `package.json`、`pnpm-workspace.yaml` |
| 构建 | 前端 rspack + nx（`scripts/rspack/rspack.prod.ts`）；后端 go build + Makefile；wire 代码生成 | `package.json`、`Makefile` |
| 前端公共库 | packages/ 下 18 个可复用包：`@grafana/ui`（组件库，1057 个 ts 文件）、`@grafana/data`、`@grafana/schema`、`@grafana/runtime`、`@grafana/sql`、`@grafana/alerting` 等 | `packages/` |
| 存储 | 默认 SQLite（`pkg/util/sqlite` 独立 module），支持 MySQL/PostgreSQL | `go.work`、`conf/defaults.ini` |
| e2e | Playwright（242 个 spec） | `playwright.config.ts`、`e2e-playwright/` |
| 其他 | CUE（kind 定义 `kinds/`、`cue.mod/`）、NATS（模块间通信，`pkg/server/nats_module_test.go`）、OpenFeature（`pkg/services/featuremgmt/openfeature.go`） | 各对应文件 |

配置系统极重：`conf/defaults.ini` 单文件 2770 行；feature toggle 共 259 个（`pkg/services/featuremgmt/toggles_gen.go`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `pkg/` | Go 后端主体：api（HTTP 层）、server（生命周期/DI）、services/（78 个业务服务）、tsdb（数据源查询后端）、plugins（插件模型/加载/签名）、expr（服务端表达式引擎）、infra、storage |
| `apps/` | 24 个 App Platform 应用，独立 Go module，k8s-apiserver 风格（dashboard、folder、iam、alerting/*、provisioning 等） |
| `public/app/` | React 前端：features/ 61 个功能模块、plugins/（panel 30 种 + datasource 7 种）、extensions/、store/ |
| `packages/` | 18 个 @grafana/* 可独立发布的公共库（ui/data/schema/runtime/sql/alerting…） |
| `kinds/` | 核心实体的 CUE schema（dashboard 等），多 API 版本 |
| `e2e-playwright/` | Playwright e2e：242 个 spec，按 suite 分目录（alerting、dashboards、cloud-plugins…） |
| `conf/` | defaults.ini（2770 行）与样例配置 |
| `devenv/` | 开发环境数据源/仪表盘样例 |
| `docs/`、`contribute/`、`.changelog-archive/` | 文档、贡献指南、历史 CHANGELOG |
| `grafana-mixin/` | 自监控用的 Prometheus mixin |

组织方式评述：这是一个"单体 + 多 module"的混合 monorepo。后端用 go.work 把 24 个 apps 与若干 pkg 子模块（pkg/plugins、pkg/apiserver、pkg/storage/unified/* 等）拆成独立 Go module 以控制编译边界；前端用 pnpm workspace 把可复用库抽成独立包。主仓库内部又通过 service 接口（`pkg/services/*` 每个服务一个目录，接口+impl+mock 同置）保持模块化，本质仍是单个可执行文件的单体架构（`pkg/cmd/grafana/`，未逐行确认入口，见 pkg/cmd 目录）。

## 5. 核心架构与关键实现

### 5.1 后端服务分层与生命周期（pkg/server + pkg/services）

Grafana 后端是一个"服务注册 + Wire 注入"的生命周期容器。`pkg/server/server.go` 中 `Server` 负责 Init/Run/Shutdown 一组实现 `registry.BackgroundService` 的服务；更细粒度地，`pkg/server/module_server.go` 把进程拆成可独立运行的模块（Core、FrontendServer、SwaggerServer、SearchServerDistributor、GRPCServer、NATS 模块等），支持横向扩展场景（如单独跑搜索分发器）。依赖图用 google/wire 生成（`pkg/server/wire_gen.go`，注入 34 处 Provide/New 服务）。业务逻辑全部下沉到 `pkg/services/` 的 78 个服务，各自暴露接口（如 `pkg/services/dashboards/dashboard.go` 的 DashboardService），服务之间只依赖接口，便于 mock（每个服务目录自带 *_test 与 mock 文件）。

### 5.2 数据源插件协议（pkg/tsdb + grafana-plugin-sdk-go）

数据源后端实现 SDK 的 `backend.QueryDataHandler` 接口。以仍留在主仓库的 graphite 为例（`pkg/tsdb/graphite/graphite.go:84`）：

```go
func (s *Service) QueryData(ctx context.Context, req *backend.QueryDataRequest) (*backend.QueryDataResponse, error) {
    if len(req.Queries) == 0 {
        return nil, fmt.Errorf("query contains no queries")
    }
    // get datasource info from context
    dsInfo, err := s.getDSInfo(ctx, req.PluginContext)
    if err != nil {
        return nil, err
    }
    return s.RunQuery(ctx, req, dsInfo)
}
```

要点：请求按 query 并行打包、`PluginContext` 携带数据源实例配置与用户身份、错误区分 `DownstreamError/PluginError`（`pkg/tsdb/graphite/graphite.go`）。内置数据源走进程内 Service 直调；外部插件则通过 `pkg/plugins/backendplugin/` 的进程管理（gRPC）对接。同一协议栈上还有 `CallResource`（代理资源请求）与健康检查（`pkg/tsdb/graphite/healthcheck.go`）。多数据源混合查询由前端 mixed 数据源 + 后端 `pkg/expr` 表达式引擎（`pkg/expr/graph.go` 构建 DAG）完成。

### 5.3 插件体系与签名（pkg/plugins + pkg/services/pluginsintegration）

插件统一由 `plugin.json` 描述（模型在 `pkg/plugins/models.go`，含 Dependencies、Extensions V1/V2 声明式扩展点）。加载管线在 `pkg/plugins/manager/`：sources 发现 → loader 读取 → registry 登记 → signature 验签 → installer 安装。签名机制是安全核心（`pkg/plugins/manager/signature/manifest.go`）：插件包内 MANIFEST.txt 记录文件哈希，用 Grafana 私钥分级签名（official/commercial/community），校验时对 ID、版本、每个文件逐一比对，未覆盖文件会告警。`pkg/services/pluginsintegration/`（30 个子目录）把插件系统接入 HTTP、数据源代理（clientmiddleware）、Angular 淘汰检测（angulardetectorsprovider/angularinspector）与访问控制。

### 5.4 前端插件加载（public/app/features/plugins）

前端用 SystemJS 在运行时加载插件 JS（`public/app/features/plugins/loader/systemjs.ts`）：

```ts
import 'systemjs/dist/system';
// Add ability to load plugins bundled as AMD format
import 'systemjs/dist/extras/amd';
// Add named register for on demand dependency loading
import 'systemjs/dist/extras/named-register.js';

export const SystemJS = window.System;
```

pluginPreloader 预取插件元信息，loader 通过 sharedDependencies 把 `@grafana/*` 作为共享依赖注入插件沙箱（`sharedDependencies.ts`），另有 `sandbox/` 目录做插件隔离。面板侧 `public/app/plugins/panel/` 提供 timeseries、barchart、table、geomap、logs、traces、nodeGraph、canvas 等 30 种面板；数据源侧的注册与解析在 `public/app/features/plugins/datasource_srv.ts`。

## 6. 构建与运行

来自 `contribute/developer-guide.md` 与 `package.json`（未实际运行验证）：

```bash
# 前端开发（rspack dev server）
pnpm install
pnpm dev            # nx exec -- rspack build --config scripts/rspack/rspack.dev.ts

# 后端
make backend        # 或 go run .（pkg/cmd 入口）

# 生产构建
pnpm build          # NODE_ENV=production rspack，8GB 堆
# 测试
pnpm test           # jest --watch
go test ./pkg/...
pnpm e2e            # playwright（e2e-playwright/）
# Docker
docker build -t grafana .   # developer-guide "Build a Docker image"
```

工具链经 `mise.toml` 固定版本，git hooks 用 lefthook（`lefthook.yml`）。

## 7. 测试与工程质量

- 测试规模：Go 测试文件 2321 个（pkg/ 内），前端 jest 测试 2379 个（public/app）+ 548 个（packages/），Playwright e2e spec 242 个（`find` 统计）。
- CI：`.github/workflows/` 下约 40+ 工作流，覆盖 backend-unit-tests、backend-code-checks、build-go-matrix、check-frontend-test-coverage、deploy-pr-preview、create-release-tag 等；另有 auto-triager、backport-workflow 等自动化（目录列表）。
- 质量观察：服务层普遍接口化并自带 mock；feature toggle 体系（toggles_gen.go 生成式管理 259 个开关）与 OpenFeature 集成便于灰度；eslint suppressions 与 knip（未用依赖检查）等治理工具齐备（`eslint-suppressions.json`、`knip.config.ts`）。e2e 按 suite 组织并有 journey-tracking 目录，投入明显。

## 8. 社区与维护状态

- 活跃度：push 时间 2026-10-30，即分析前一天仍在提交（gh api）。CHANGELOG 最新版本 13.3.0 发布于 2026-10-27。
- Release 节奏：同日维护 5 条分支——v13.3.0、v13.2.5、v13.1.9、v13.0.12、v12.4.14 均于 2026-10-27 发布（gh api releases），当日批次以安全修复为主：CHANGELOG 中 13.3.0 一节列出 10 个 CVE 修复，说明有月度安全批量发布机制。
- 贡献者：GitHub contributors API 分页统计 372 人（API 有缓存，前端页面显示数通常更高，未确认精确值）。
- 治理：仓库有正式的 `GOVERNANCE.md`、`MAINTAINERS.md`、`WORKFLOW.md`、`ROADMAP.md`、`AI_POLICY.md` 与 `HALL_OF_FAME.md`，商业化由 Grafana Labs 承担；i18n 走 Crowdin（`crowdin.yml`）。

## 9. 亮点与风险

亮点：
1. 插件协议设计成熟：plugin.json 声明模型 + SDK 接口（QueryData/CallResource/HealthCheck）+ gRPC 进程外插件 + 签名验证，形成完整生态闭环（`pkg/plugins/`、`pkg/tsdb/`）。
2. 大规模服务化单体仍保持清晰边界：78 个服务接口化 + wire 显式依赖图 + go.work 多 module 编译隔离，5800 个 Go 文件没有失控成泥球（`pkg/services/`、`go.work`）。
3. 工程化水位高：40+ CI 工作流、三类测试共 5000+ 文件、feature toggle 生成式管理、Storybook 组件库独立发布。
4. 演进路径明确：App Platform（apps/ 的 k8s-apiserver 化 + CUE kinds）展示了一条单体渐进迁移到统一 APIServer 的可操作路径（`apps/dashboard/`、`kinds/`）。

风险/局限：
1. 巨型仓库复杂度高：核心数据源已大量外移，主仓库内 tsdb 只剩 4 个实现，新贡献者需要同时理解主仓库与多个外部数据源仓库的关系。
2. AGPL-3.0 对二次分发的传染性（`LICENSE`），商用集成需走 Grafana Labs 授权或注意例外条款（`LICENSING.md`）。
3. 双栈构建链重：pnpm+nx+rspack+jest+playwright 与 go.work+wire+CUE 叠加，本地全量构建成本大（`package.json` build 脚本需 8GB 堆）。
4. 大量 feature toggle 与 Angular 兼容层并存，长期维护负担可见（259 个 toggle；pluginsintegration 内整套 angular 检测器）。

## 10. 总结评价

Grafana 是可观测性领域事实上的标准前端，其价值不在单个算法而在"数据源抽象 + 插件生态 + 仪表盘模型"这套协议设计；代码库是学习大型 Go 服务化单体与 React 插件化前端的一流教材。适合 SRE/平台工程师直接使用，适合后端开发者研读 `pkg/server`+`pkg/services` 的服务组装方式、插件开发者研读 `pkg/plugins` 与 `public/app/features/plugins` 的双层插件机制，以及任何在维护大型 monorepo 的团队参考其 CI 与模块边界治理。
