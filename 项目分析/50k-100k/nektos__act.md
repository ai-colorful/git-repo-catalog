# nektos/act 项目分析

> 星数 72435（快照 2026-10-24）｜主语言 Go｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：`act` 是一个用 Go 编写的 CLI 工具，让你直接在自己的机器上通过 Docker 运行 GitHub Actions 工作流——"Think globally, `act` locally"，免去每次验证 `.github/workflows/` 改动都要 commit/push 的循环。

| 项目 | 值 | 出处 |
| --- | --- | --- |
| 名称 | nektos/act | 仓库元数据 |
| 星数 | 72435（watchers 72499） | 任务参数 / gh api |
| 主语言 | Go（go.mod 声明 go 1.26.10） | `go.mod` |
| 许可证 | MIT | gh api `.license.spdx_id` |
| 创建时间 | 2019-01-02 | gh api |
| 最近 push | 2026-10-28（分析时 3 天前） | gh api |
| 最新版本 | v0.2.92（2026-10-28 发布） | gh api releases |
| 贡献者 | 204 人 | gh api contributors |
| 默认分支 | main | gh api |

## 2. 项目是什么，解决什么问题

GitHub Actions 的调试回路天然很慢：改一行 workflow YAML 就要 commit、push、等待 runner 分配，再看日志。act 把这个回路搬回本地——读取项目里的 `.github/workflows/`，按事件（默认 `push`）选出要跑的 job，用 Docker API 拉取或构建所需镜像，按 `needs` 依赖确定执行顺序，然后为每个 job/step 起容器执行，环境变量与文件系统布局尽量对齐 GitHub 托管 runner（README.md "How Does It Work?" 一节，`src/README.md`）。

两大使用场景（`src/README.md`）：一是 **Fast Feedback**，本地快速验证 workflow 与内嵌 action 的改动；二是 **Local Task Runner**，把 `.github/workflows/` 当成替代 Makefile 的任务定义——工作流语法比 Make 更结构化，还自带矩阵、依赖编排。

act 并不模拟 Actions，而是尽量**重新实现**其运行时语义：它内置了 GitHub Runner 的不少协议面——artifact 上传下载 API（`pkg/artifacts/`，v4 协议）、actions cache API（`pkg/artifactcache/`，`ACTIONS_RUNTIME_TOKEN` 签发见 `pkg/runner/run_context.go`）、`GITHUB_ENV`/`GITHUB_STATE` 环境文件、`::set-output::` 等工作流命令（`pkg/runner/command.go`）。这使大量真实世界的 action（actions/checkout、setup-node 等）无需修改即可本地运行。

它也是生态的地基：Gitea 的 act_runner、VS Code 的 GitHub Local Actions 扩展（README 有推荐位）都构建在 act 之上；act 定义的 `HostEnvironment` 抽象（`pkg/container/host_environment.go`）让"不开 Docker 直接在宿主机跑"成为可能。

## 3. 技术栈与依赖

Go 单体二进制，依赖选型务实（出处 `go.mod`）：

- **CLI**：`spf13/cobra` + `spf13/pflag`（命令与约百个 flag，见 `cmd/root.go`），`AlecAivazis/survey/v2`（交互式选择平台/secret），`sirupsen/logrus`（日志）。
- **容器**：`moby/moby/client` + `docker/cli` + `docker/go-connections`，`opencontainers/image-spec`、`opencontainers/selinux`——直接走 Docker API 管理镜像/容器/网络。
- **Git**：`go-git/go-git v5` + `go-billy`——远程 action 与 reusable workflow 的拉取与缓存（`pkg/runner/action_cache.go`）。
- **表达式引擎**：`rhysd/actionlint v1.7.7`——关键复用决策：不自己写 `${{ }}` 表达式解析器，而是复用 actionlint 的 AST（`pkg/exprparser/interpreter.go` 的 `evaluateNode(actionlint.ExprNode)`）。
- **存储**：`bolthold`/`bbolt`（artifact cache 的 KV 存储，`pkg/artifactcache/storage.go`），`julienschmidt/httprouter`（本地 artifact/cache HTTP 服务）。
- **其他**：`gopkg.in/yaml.v3`（workflow 解析）、`creack/pty`、`joho/godotenv`（`.env` 文件）、`sabhiram/go-gitignore` + `moby/patternmatcher`（路径过滤）、`adrg/xdg`（配置目录）。

## 4. 目录结构与代码组织

标准 Go 单模块布局（`go.mod` module `github.com/nektos/act`），约 19.4k 行非测试 Go 代码（含测试约 32.2k 行）：

| 目录 | 职责 |
| --- | --- |
| `main.go` | 入口；三级版本解析（ldflags → buildinfo → 内嵌 VERSION 文件） |
| `cmd/` | cobra CLI：`root.go`（主命令）、`actrc.go`（默认配置文件）、`graph.go`（依赖图可视化）、`list.go`、`platforms.go`、`secrets.go`、`input.go` |
| `pkg/model/` | workflow/action 的领域模型与 YAML 解析：`workflow.go`（762 行）、`planner.go`（Plan/Stage/Run 与依赖分阶段）、`anchors.go`（YAML 锚点）、`github_context.go` |
| `pkg/runner/` | 执行编排核心：`runner.go`、`run_context.go`（1250 行，最大文件）、`job_executor.go`、`step*.go`（四类 step）、`action*.go`（action 解析与缓存）、`expression.go`、`logger.go`、`testdata/`（大量迷你工作流 fixture） |
| `pkg/container/` | Docker 与宿主机执行抽象：`container_types.go`（`Container`/`ExecutionsEnvironment` 接口）、`docker_run.go`（977 行）、`docker_cli.go`（1162 行）、`host_environment.go`、`docker_options_policy.go`（安全策略） |
| `pkg/exprparser/` | GitHub 表达式求值器，基于 actionlint AST（`interpreter.go`、`functions.go`） |
| `pkg/artifacts/` | 本地 artifact 服务（v4 API + protobuf，`server.go`、`artifacts_v4.go`） |
| `pkg/artifactcache/` | actions cache 服务（`handler.go` + bbolt `storage.go`） |
| `pkg/common/` | Executor 组合子库（`executor.go`）等公共设施 |
| `pkg/schema/` | workflow/action 的 JSON Schema 校验（`workflow_schema.json`、`action_schema.json`） |
| `pkg/filecollector/`、`pkg/workflowpattern/`、`pkg/gh/`、`pkg/lookpath/` | 文件收集（gitignore 感知）、路径模式、GitHub API 助手、vendor 的 LookPath |
| `.github/` | CI/发布/安全工作流；`docs/` 仅存图片，正文文档在独立站点 nektosact.com |

组织方式评述：按"解析（model）→ 编排（runner）→ 执行（container）"三层清晰分包，接口边界（`WorkflowPlanner`、`Runner`、`Container`、`ActionCache`、`stepFactory`）划分明确，是教科书式的 Go CLI 工程结构。

## 5. 核心架构与关键实现

### 5.1 Executor 组合子：一切皆 `func(ctx) error`

整个执行引擎建立在一个极简抽象上（`pkg/common/executor.go`）：

```go
// Executor define contract for the steps of a workflow
type Executor func(ctx context.Context) error

// Then runs another executor if this executor succeeds
func (e Executor) Then(then Executor) Executor {
    return func(ctx context.Context) error {
        err := e(ctx)
        if err != nil {
            switch err.(type) {
            case Warning:
                Logger(ctx).Warning(err.Error())
            default:
                return err
            }
        }
        ...
        return then(ctx)
    }
}
```

`Then`/`OnError`/`If`/`Finally` 等组合子把 job 编排、step 顺序、容器启停、日志注入全部声明式地拼起来；`NewParallelExecutor`（同文件）用 worker channel 池实现受控并发（job 级并发默认 = CPU 核数，`runner.go` 的 `GetConcurrentJobs`）。

### 5.2 Planner：从 needs 依赖到分阶段执行计划

`pkg/model/planner.go` 的 `createStages` 是调度的心脏：先沿 `needs` 闭包收集全部依赖 job，再循环构建 Stage——每一轮把"依赖已全部入图"的 job 放入当前 Stage（拓扑分层），同 Stage 内的 job 由 `runner.go` 的 `NewPlanExecutor` 用 `NewParallelExecutor` 并行跑；无法分层时报依赖环错误。矩阵策略在进入 Stage 前展开（`job.GetMatrixes()`，`pkg/model/workflow.go`），支持 `--matrix` 过滤（`selectMatrixes`，`pkg/runner/runner.go`）。

### 5.3 RunContext 与 job 生命周期：pre → main → post

`pkg/runner/run_context.go`（1250 行）是单个 job 的运行时上下文：持有 Config、矩阵、`StepResults`、表达式求值器，负责 job 容器（或宿主环境）的创建/挂载卷/服务容器健康等待（`waitForServiceContainers`）。`pkg/runner/job_executor.go` 则把每个 step 展开为 pre/main/post 三段 executor，post 段用 `Finally` 链**逆序**执行——这精确复刻了 GitHub Actions 的 `post:` 语义：

```go
postExec := useStepLogger(rc, stepModel, stepStagePost, step.post()...)
if postExecutor != nil {
    // run the post executor in reverse order
    postExecutor = postExec.Finally(postExecutor)
} else {
    postExecutor = postExec
}
```

step 分派见 `pkg/runner/step_factory.go`：`StepTypeRun`（shell）、`UsesActionLocal`/`UsesActionRemote`（本地/远程 action，可递归展开 composite）、`UsesDockerURL`（Docker action），四类各有实现文件。

### 5.4 双执行环境：Docker 容器 or 宿主机

`pkg/container/container_types.go` 定义了 `Container` 接口（Create/Pull/Start/Exec/Copy/UpdateFromEnv…），`docker_run.go`/`docker_cli.go` 给出 Docker 实现；`host_environment.go` 的 `HostEnvironment` 实现同一 `ExecutionsEnvironment` 契约，配合 `--platform=-self-hosted` 即可在无 Docker 环境下跑（这也是 Gitea act_runner 等下游复用的接缝）。值得注意的安全设计：`docker_options_policy.go` 引入 `--container-options-policy=auto|restrict|allow`，对**来自 workflow 文件**的 `options:` 默认拒绝可触及宿主文件/设备/命名空间的参数（`NewContainerInput.RestrictOptions`，`container_types.go` 注释明确"当 Options 来自 workflow 而非运维者时设置"）。

### 5.5 远程 action 缓存：裸 git 仓库

`pkg/runner/action_cache.go` 的 `GoGitActionCache` 把每个远程 action 克隆成 `safeFilename(cacheDir)+".git"` 的裸仓库，命中即 `PlainOpen`，拉取时用随机临时分支避免锁冲突，再按 sha 输出 tar 流给容器拷贝；另有 offline 模式实现（`action_cache_offline_mode.go`）。reusable workflow（`reusable_workflow.go`）复用同一缓存机制。

## 6. 构建与运行

来自 `src/README.md` 与 `Makefile`（未实际运行，仅静态阅读）：

```bash
make build      # 构建
make install    # 构建并安装到 $GOPATH/bin
make test       # go test ./... + 用 act 跑自身测试工作流（需要 Docker）
make test-fast  # go test -short ./...（免 Docker）
make lint       # golangci-lint 等静态检查
```

日常使用：`act push` / `act -j <job>` / `act -l`（列出 job）/ `act --graph`（依赖图），配置可写入 `~/.actrc`（`cmd/actrc.go`）。安装另有 `install.sh` 与 chocolatey 包描述（`act-cli.nuspec`）。

## 7. 测试与工程质量

- **规模**：53 个 `_test.go` 文件、258 个测试函数；`pkg/runner/testdata/` 下有数十个迷你工作流项目（basic、matrix、composite、services、GITHUB_ENV 等场景），属于"真实 YAML fixture + 断言行为"的集成式单测风格，另有 `pkg/runner/linux_shard_test.go` 做 CI 分片。
- **CI**（`.github/workflows/checks.yml`）：lint + 按 shard 并行的 test-linux（QEMU、缓存 fixture 镜像、Codecov、测试计时上传），最后 **"Run act from cli" 用 act 跑自己的 workflow——高度自举的 dogfooding**。
- **安全与供应链**：codeql、govulncheck、zizmor（workflow 静态审计）、scorecard、gitleaks、dependabot 自动合并流水线（`.github/workflows/`），`SECURITY_AUDIT.md` 存在，工程纪律在同类项目中属第一梯队。

## 8. 社区与维护状态

- 非常活跃：最近 push 2026-10-28；v0.2.90/91/92 三个 release 均发布于同一天（gh api releases），说明有紧的补丁节奏与自动化发布（`.github/workflows/release.yml`、`promote.yml`）。
- 204 位贡献者（gh api，per_page=1 取 last page），374 个 open issue，204 watchers——issue 积压不小，符合 7 万星项目的常态。
- 历史：2019-01 创建，v0.2.x 已迭代 90+ 个版本；README 中文档外移到 nektosact.com 独立站点，本体只留 44 行 README。
- 核心维护者结构与 bus factor：未确认（tarball 无 git 历史，未做逐提交统计）。

## 9. 亮点与风险

**亮点**

1. `Executor func(ctx) error` 组合子模型（`pkg/common/executor.go`）——用极小抽象承载了整个 DAG 并发编排，可读性与可测性俱佳。
2. 对 GitHub Actions 运行时语义的高保真复刻：矩阵 include/exclude、reusable workflow、composite action、service 健康检查、GITHUB_ENV/STATE、`::set-output::` 命令、artifact/cache 本地服务端，兼容性是它的核心资产。
3. 战略性复用 actionlint 的表达式 AST（`pkg/exprparser/interpreter.go`），避免自己维护一门表达式语言的解析器。
4. 容器/宿主双执行环境抽象（`ExecutionsEnvironment`），既支持零 Docker 的 `-self-hosted` 模式，也让 act_runner 等下游可嵌入。
5. 近期仍在主动加固：容器 options 安全策略（`docker_options_policy.go`）、gitleaks/zizmor/scorecard 全套供应链防线。

**风险/局限**

1. 兼容性长尾永远追不完：GitHub 持续演进 Actions 语义，act 是逐个特性追赶的重实现，边缘行为差异是已知常态（open issue 374 个）。
2. `pkg/runner/run_context.go` 1250 行、职责繁重（容器生命周期、挂载、上下文构造、缓存令牌），是事实上的"上帝对象"，改动风险集中。
3. 完整体验依赖本机 Docker daemon，且部分镜像需 privileged 模式；默认镜像（catthehacker 系列）是外部项目，版本耦合不受 act 仓库控制。
4. 无 Windows/macOS 原生容器执行（依赖 Docker 语义），宿主模式在非 Linux 平台能力受限（`pkg/container/util_windows.go` 等平台分支较多）——后者为观察，未逐行确认。

## 10. 总结评价

act 是"本地复刻 GitHub Actions 运行时"这一难题的公认最佳开源答案：72435 星、2019 年至今持续高频迭代、204 名贡献者、以自身跑自身 CI 的工程纪律，让它成为 Actions 生态事实上的本地运行时标准。架构上，它展示了如何用一个函数类型（Executor）+ 少量组合子搭建出完整的 DAG 执行引擎，以及如何通过接口（Container/HostEnvironment、ActionCache）保持可嵌入性。适合所有写 GitHub Actions 的开发者日常使用；对想学习"如何用 Go 编排容器化任务""如何兼容一个外部平台的运行时语义"的工程师，`pkg/common/executor.go` → `pkg/model/planner.go` → `pkg/runner/job_executor.go` → `pkg/container/` 这条主线是非常值得一读的源码路径。
