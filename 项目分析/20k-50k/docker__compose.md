# docker/compose 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Go｜档位 20k-50k｜分析日期 2026-11-14
>
> gh api 复核（2026-11-14）：38824 stars，Go，Apache-2.0，最近 push 2026-11-13，open issues 119，forks 5943。

## 1. 项目速览

一句话定位：Docker 官方的多容器应用编排工具 Compose v2+ 的主仓库——用 Go 重写、以 Docker CLI 插件形态发布的 `docker compose`，是 compose-spec 规范的参考实现。

| 事实 | 值 | 出处 |
|---|---|---|
| 名称 | docker/compose（Go 模块 `github.com/docker/compose/v5`） | `go.mod` |
| 星数 | 38824（2026-11-14 gh api；快照 39500） | gh api |
| 语言 | Go 1.26 | `go.mod` |
| 许可证 | Apache-2.0 | `LICENSE` |
| 当前版本线 | v5.x（最新 v5.7.1，2026-11-11 发布） | gh api releases |
| 最近活跃 | push 2026-11-13，约每月发版 | gh api |
| 规模 | 434 个 Go 文件，136 个单测文件 + 58 个 e2e 测试文件 | 源码统计 |
| 贡献者 | 约 295 人（含匿名） | gh api contributors |

## 2. 项目是什么，解决什么问题

Compose 解决"一个应用 = 一组容器"的编排问题：用户在 `compose.yaml` 里声明式定义各服务（镜像/构建、端口、卷、依赖、健康检查），然后 `docker compose up` 一条命令拉起整个应用。README 给出的三步法是 Dockerfile → compose.yaml → `up`（`README.md` Quick Start）。

这个仓库是 Compose 的第二代实现。第一代是 Python 编写的独立二进制（docker-compose v1），维护于 `v1` 分支并已 EOL（`README.md` Legacy 一节）。v2 起用 Go 重写，编译为 Docker CLI 插件二进制放入 `cli-plugins` 目录，从而作为 `docker compose` 子命令原生运行（`cmd/main.go` 中 `plugin.Run` + `metadata.Metadata{Vendor: "Docker Inc."}`）；同时支持 standalone 模式，独立运行时把旧式参数经 `compatibility.Convert` 转换以尽量模仿 v1 行为（`cmd/main.go` 的 `RunningStandalone` 分支、`cmd/compatibility/convert.go`）。

文件格式由 compose-spec.io 规范定义，解析/插值/合并交给独立的 `compose-spec/compose-go/v2` 库（v2.16.2，`go.mod`），本仓库专注运行时编排。Compose 也是 Docker Desktop 的内置组件，并有一个 `desktop-module/` 目录存放其 Desktop 集成元数据。

除传统容器编排外，v2 近年把边界向外推：`docs/extension.md` 定义了 `provider` 扩展模型——一个服务可以把实现委托给外部工具管理的云资源（云数据库、沙箱等），为此配套了独立的 `relay/` 模块（compose-relay 网关，把 provider 资源映射回 `http://<service>:<port>` 的 compose 原生地址，见 `relay/README.md`）；`pkg/bridge` 则提供 Swarm 遗产配置向 compose 的转换（`docker compose bridge convert`，`cmd/compose/compose.go` 命令表）。

## 3. 技术栈与依赖

- 语言/运行时：Go 1.26（`go.mod`），无框架，纯 cobra CLI + 官方 Docker 客户端库。
- 关键依赖（均见 `go.mod`）：
  - `github.com/compose-spec/compose-go/v2`——compose-spec 文件解析、环境插值、多文件合并、profile 选择，即 compose 项目模型的核心。
  - `github.com/docker/cli`——CLI 插件框架（`cli-plugins/plugin.Run`）、docker context/配置。
  - `github.com/moby/moby/api` + `client`——Docker Engine API 客户端；`github.com/containerd/containerd/v2`——容器原语。
  - `github.com/moby/buildkit`——镜像构建（buildx/bake 协同，见第 5 节）。
  - `github.com/spf13/cobra` + `pflag`——命令行框架。
  - `github.com/tilt-dev/fsnotify` + `fsnotify/fsevents`——watch 模式的文件事件（darwin 用 FSEvents）。
  - OpenTelemetry 全家桶（otel/otlp-grpc）——`internal/tracing` 统一插桩，`cmd/cmdtrace` 在命令入口启用。
  - `go.uber.org/mock`（mock 生成于 `pkg/mocks`）、`gotest.tools/v3`——测试。
- 构建：`docker-bake.hcl` + Makefile，CI 产物带 SBOM 与 provenance（`.github/workflows/ci.yml`）。
- 子模块：`relay/` 有独立 `go.mod`（compose-relay 网关）。

## 4. 目录结构与代码组织

| 目录 | 职责 | 出处 |
|---|---|---|
| `cmd/main.go` | 入口：以 CLI 插件运行，standalone 时走 v1 参数兼容层 | |
| `cmd/compose/` | 36 个子命令的 cobra 定义与 flag→Options 映射（up/down/build/run/exec/watch/bridge/alpha 等） | `cmd/compose/compose.go` 的 `AddCommand` 列表 |
| `cmd/formatter`、`cmd/display`、`cmd/prompt` | 输出格式化（progress/log）、交互菜单（`--menu`）、确认提示 | |
| `cmd/compatibility` | standalone 模式下 v1 参数转换 | `cmd/compatibility/convert.go` |
| `pkg/api/` | 对外 API 接口层：`Compose` 总接口及 Options/事件类型（921 行） | `pkg/api/api.go` |
| `pkg/compose/` | 编排引擎实现（composeService），up/create/start/down/build/watch 全在这里，约 60 个文件 | |
| `pkg/watch/` | 文件监视器平台适配（darwin FSEvents / naive walk / windows）与防抖 | `pkg/watch/notify.go`、`debounce.go` |
| `pkg/remote/` | Git/OCI 远程 compose 文件加载 | |
| `pkg/dryrun/` | dry-run 模式：包装 Docker client 拦截写操作 | `pkg/dryrun/dryrunclient.go` |
| `pkg/bridge/` | Swarm→compose 配置转换（transformers 表驱动） | `pkg/bridge/convert.go` |
| `pkg/e2e/` | 58 个端到端测试（需真实 daemon） | |
| `internal/depgraph/` | 服务依赖图构建与按（逆）依赖序并行遍历 | `internal/depgraph/depgraph.go` |
| `internal/engine/` | 对 daemon 能力的统一抽象 `Backend` 接口（APIClient/MaxConcurrency/Containers） | `internal/engine/engine.go` |
| `internal/projectloader/` | LoadProject（从文件）/ Generate（从现有容器反推） | `internal/projectloader/loader.go` |
| `internal/sync/` | watch 同步的 tar 打包传输实现 | `internal/sync/tar.go` |
| `internal/progress`、`logstream`、`tracing`、`concurrency`、`containerops` 等 | 进度渲染、日志汇聚、OTel 插桩、并发限流原语等支撑层 | |
| `relay/` | 独立 Go 模块 compose-relay：provider 托管资源的网络网关 | `relay/README.md` |
| `docs/reference/` | 93 份 cobra 生成的命令参考文档 | |
| `docs/extension.md`、`docs/sdk.md` | provider 扩展模型、Go SDK 用法 | |

组织评述：清晰的四层——`cmd`（交互层）→ `pkg/api`（契约层）→ `pkg/compose`（业务层）→ `internal/*`（支撑层）。`pkg/api` 接口使 Compose 可作为 Go SDK 被 Docker Desktop 等宿主嵌入（`docs/sdk.md`）；`internal` 与 `pkg` 的划分暗示 `pkg` 是对外稳定面。多模块（主模块 + relay）+ bake 声明式构建，是典型的现代化 Go 单仓多件套。

## 5. 核心架构与关键实现

### 5.1 接口契约层与 composeService

`pkg/api/api.go:102` 把能力切成六个正交接口组成 `Compose`：`Lifecycle`（up/down/stop/scale 等生命周期）、`Inspector`（ps/logs/top 只读）、`ImageManager`（build/pull/push/publish）、`Runner`（run/exec/attach/cp）、`Watcher`（开发循环）、`ProjectLoader`。实现集中在 `pkg/compose/composeService`（`pkg/compose/compose.go:208`），持有 `dockerCli`、事件总线 `events`、dry-run 客户端等；SDK 用户可注入自定义 IO 流（`outStream/errStream/inStream`）。命令层（`cmd/compose/*`）只做参数解析并调用接口，实现了 UI 与编排逻辑完全分离。

### 5.2 Plan/Executor 编排引擎（近期重构的重点）

up 的执行已从"命令式过程调用"重构为"先规划、后执行"的声明式引擎（源码注释引用 epic #14081，`pkg/compose/up.go:98` 附近的 `upDetached` 注释）：

1. `preparePlan` 对比项目期望状态与 daemon 快照（observed state），生成一个有向无环的 `Plan`，节点是原子操作：`OpCreateNetwork/OpCreateVolume/OpCreateContainer/OpStartContainer/OpWaitCondition/OpRunPreStart/OpRunPostStart/OpRunProvider` 等（`pkg/compose/plan.go:38-63`）。
2. `planExecutor` 沿 DAG 并行执行节点——"它不带决策逻辑，所有决策都由 reconciler 在构建 plan 时做出"（`pkg/compose/executor.go` 顶部注释）。并发用 `golang.org/x/sync/semaphore` 约束（`COMPOSE_PARALLEL_LIMIT`，`cmd/compose/compose.go:41`），依赖等待（`service_healthy`/`service_completed_successfully`）每个节点有独立的新鲜超时窗口而非共享绝对期限（`pkg/compose/up.go:78-96` 注释，这是个细致的正确性设计）。
3. 依赖排序的基础设施在 `internal/depgraph/depgraph.go`："builds the dependency graph of a project's services and walks it in (reverse) dependency order, optionally in parallel"。

交互式 up 将 Create 与 Start 两阶段共享同一份 plan 与 daemon 快照（`createdUp` 结构，`pkg/compose/up.go:56`），替代了旧实现"两次快照、第二次看不见第一次结果"的缺陷。`pkg/compose/reconcile.go`/`observed_state.go` 承担状态对比。

### 5.3 watch 开发模式

`develop.watch` 配置驱动的热部署循环，五个动作：`sync`（tar 流式同步文件进容器，`internal/sync/tar.go`）、`rebuild`（触发重建）、`restart`、`sync+restart`、`sync+exec`（动作枚举来自 compose-go 的 `types.WatchAction*`，分发逻辑在 `pkg/compose/watch.go:529-538`）。文件事件由 `pkg/watch` 提供平台适配：macOS 用 FSEvents（`watcher_darwin.go`），其余平台用 fsnotify/naive walk（`watcher_naive.go`），配 `debounce.go` 防抖与 `dockerignore.go` 过滤；rebuild 有专门的 `rebuildScheduler`（`pkg/compose/rebuild_scheduler.go:37`）合并抖动期的多次变更。watch 既可独立运行（根命令表中的 `watchCommand`），也被 `up --watch` 集成。这是 Compose v2.22（2023-11）GA 的特性，如今已是成熟子系统。

### 5.4 构建链：buildx/bake 集成

构建默认走 buildx 的 bake 子命令：`pkg/compose/build_bake.go` 自定义了精简版 bake 配置类型（`bakeConfig/bakeGroup/bakeTarget`，`build_bake.go:80-88`）——注释明确"长期计划是移除对 buildx 的依赖"，即不经 buildx 库直接生成 bake JSON、以 `--allow fs.read/fs.write` 授权本地路径访问（`bakeBuild` 结构 `build_bake.go:124-132`）。旧式经典 builder 作为回退保留在 `pkg/compose/build_classic.go`。

### 5.5 与 Python v1 的架构差异

v1：Python 进程、自管 CLI 入口、直接调 Docker API。v2/v5：Go 编译的单二进制 CLI 插件，由 docker CLI 发现并注入 context/凭证（`cmd/main.go`）；配置解析外包给 compose-go；API 上下文化（`pkg/api/context.go`）+ OTel 全链路插桩（`internal/tracing/tracing.go`）；提供 dry-run（客户端拦截层 `pkg/dryrun/dryrunclient.go`）、从容器反推 compose 文件（`internal/projectloader/generate.go`）等 v1 没有的能力。模块主版本已到 v5（CLI 用户层面命令未变），反映了 Go API 的滚动演进。

## 6. 构建与运行

来自 `Makefile` 与 `BUILDING.md`（未运行验证，静态摘录）：

```bash
make                      # 构建本地插件二进制（BUILDING.md 主路径）
make binary               # 指定目标
make cross                # linux/darwin/windows 交叉编译
make test                 # 单元测试
make relay-test           # relay 子模块测试
make e2e-compose          # 端到端测试（需 Docker daemon）
make lint / relay-lint    # golangci-lint
make validate             # go-mod/headers/docs/mocks 四项一致性校验
make mocks                # 重新生成 pkg/mocks（go.uber.org/mock）
make docs                 # 生成 docs/reference 命令文档
```

产物安装：把二进制放入 `~/.docker/cli-plugins`（或 `/usr/local/lib/docker/cli-plugins` 等系统目录）后即作为 `docker compose` 使用（`README.md`）。Windows/macOS 随 Docker Desktop 内置。

## 7. 测试与工程质量

- 规模：136 个单元测试文件（遍布 `pkg/`、`internal/`，与被测文件同包同目录，表驱动为主）+ 58 个 e2e 测试文件（`pkg/e2e/`，基于 `gotest.tools` 断言与 fixtures 目录），`e2e` 按场景拆分（cascade/healthcheck/scale/watch 等，见 `pkg/e2e/` 文件列表与 `SCENARIO.md`）。
- CI（`.github/workflows/ci.yml`）：构建产物（buildx，附 SBOM+provenance 与校验和）→ 单测+覆盖率（Codecov 徽章，`codecov.yml`）→ e2e 矩阵（mode × channel × store 三维，`ci.yml:189`）。
- 治理：OpenSSF Scorecard 徽章（README）、`zizmor.yml`（GitHub Actions 安全审计）、`scorecards.yml`；还有 `pr-review.yml`/`ai-disclosure-gate.yml`（AI 贡献披露门禁，配 `AI_POLICY.md`）与 `AGENTS.md`——工程治理在同类项目中相当完善。
- 代码质量观察：包注释普遍解释"为什么"（如 executor 的 wait-timeout 设计论证），`pkg/api` 注释详尽到字段级；`gofumpt`/golangci-lint + license header 检查 + mocks/文档一致性校验纳入 `make validate`。

## 8. 社区与维护状态

- 活跃度：gh api 显示最近 push 2026-11-13（分析前一天），open issues 仅 119（相对 38.8k stars 很低），维护极其活跃。
- Release 节奏（gh api，2026-11 核实）：v5.3.1（7-07）→ v5.4.0（8-03）→ v5.5.0（8-17）→ v5.5.1（9-03）→ v5.6.0（10-02）→ v5.6.1（10-21）→ v5.7.0（11-02）→ v5.7.1（11-11），稳定在每月 1-2 个版本，patch 随 hotfix 快速跟进，与"频繁月度发版"的预期一致。
- 贡献者：约 295 人（gh api contributors 含匿名，分页累加）。核心由 Docker Inc. 工程师驱动（`metadata.Metadata{Vendor: "Docker Inc."}`，`cmd/main.go`），社区 PR 有 review 门禁与 AI 披露流程。贡献者结构细节未逐人核实。
- 分发：README 徽章与 release 页面；Linux 单独发二进制，桌面端随 Docker Desktop。

## 9. 亮点与风险

亮点：
1. 契约层设计教科书级——`pkg/api` 六接口拆分 + 可注入 IO/事件，既是 CLI 又是可嵌入 Go SDK（`docs/sdk.md`），Docker Desktop 直接复用。
2. Plan/Executor 编排引擎：声明式 DAG + 有界并发 + 每节点独立等待超时，正确性论证写在代码注释里（`pkg/compose/executor.go`、`up.go`），是学习"编排器如何收敛状态"的优秀范本。
3. watch 开发闭环完整：跨平台文件事件（FSEvents/fsnotify）+ 防抖 + tar 同步 + rebuild 调度，把"容器里的热重载"做成了系统特性。
4. compose-spec 的落地样板：解析外包给 compose-go，本体专注运行时，规范与实现职责分明。
5. 工程治理现代化：SBOM/provenance、Scorecard、zizmor、AI 披露门禁、文档自动生成校验。

风险/局限：
1. 迭代快、重构猛（模块已跳到 v5、up 内核仍在向 plan 引擎迁移，epic #14081 尚在进行中），SDK 消费方会感受到接口漂移。
2. 与 Docker 生态强耦合（buildx/bake、desktop、provider 扩展），脱离 Docker 环境不可用；Swarm 支持实际停滞（README 明言 Mirantis 接手后不再跟进）。
3. 单测以单元/集成为主，e2e 依赖真实 daemon，外部贡献者跑全量验证成本高。
4. 星数与日常使用者相比偏低（38.8k），但作为随 Docker Desktop 亿级分发的内置工具，仓库热度早已不是其影响力的准确度量。

## 10. 总结评价

docker/compose 是"从 Python 原型到工业级 Go 实现"的标杆迁移案例：以 CLI 插件形态融入 docker 生态，用 compose-go 固化规范、用 Plan/Executor 引擎重构编排内核、用 watch 补齐开发闭环，治理与发版纪律（月度节奏）均属一流。适合三类读者：想学 Go 大型 CLI 工程分层（cmd/pkg/api + internal）的工程师；想理解声明式编排、依赖 DAG、状态收敛实现的开发者（直接读 `pkg/compose/plan.go`、`executor.go`、`reconcile.go`）；以及想给 Docker 生态贡献扩展（provider/relay）的集成方。源码注释密度高且解释设计动机，阅读体验显著好于一般 Go 项目。
