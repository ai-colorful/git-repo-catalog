# harness/harness 项目分析

> 星数 38905（快照 2026-10-24）｜主语言 Go｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：Harness Open Source 是 Harness 公司开源的一体化 DevOps 平台——代码托管（SCM）+ CI/CD 流水线 + 云开发环境（Gitspaces）+ 制品仓库（Artifact Registry）四合一，本质是原 Gitness 代码托管平台与 Drone CI 的合体演进（README 明确"下一代 Drone"）。

| 项目 | 内容 |
|---|---|
| 名称 | harness/harness（Go module 名仍为 `github.com/harness/gitness`，见 `src/go.mod`） |
| 星数 | 38905（gh api 核实，2026-11-14） |
| 语言 | Go（后端）+ React/TypeScript（前端，`src/web/package.json`） |
| 许可证 | Apache-2.0（`src/LICENSE`） |
| 最近活跃 | 最近推送 2026-11-12（gh api），发版约每月一次（v2.27.0 2026-01-16 → v2.28.2 2026-04-20） |
| 贡献者 | 约 95 人（gh api contributors anon=true，Link 头 page=95，估算） |

## 2. 项目是什么，解决什么问题

Gitea/GitLab 类项目解决代码托管，Drone/Jenkins 解决 CI，但团队通常要再自行拼装制品仓库与开发环境。Harness Open Source 把这四件事收进一个 Go 单体二进制：Git 代码托管（web UI + HTTP/SSH git 协议）、`.harness/*.yaml` 流水线 CI、基于 devcontainer 的云端开发环境 Gitspaces、基于 CNCF distribution 的 OCI/Docker 制品仓库。

README "Where is Drone?" 一节给出官方定位：Drone 专注 CI，Harness 在其基础上增加代码托管、开发者环境与制品仓库；目标是最终与 Drone pipeline 能力完全对齐（目前以 `drone` 分支快照保留 Drone 主线）。go.mod 中大量 drone 系依赖（drone-go、drone-yaml、go-scm、runner-go、drone-runner-docker、go-convert）直接印证这一继承关系。

代码托管部分即原 Gitness（go.mod 模块名、`cmd/gitness/main.go` 中应用名 `"gitness"`、"Gitness Open source edition" 描述均可证）。2023 年 Gitness 开源后星数快速增长，2024 年并入 Harness 大平台开源版，Drone 作者 Brad Rydzewski 再创业的演进路线。

面向用户：想要"一个容器跑完代码托管+CI+制品库"的中小团队/私有化部署场景；面向读者：想研究 SCM 服务端、CI 调度引擎、容器编排式开发环境实现的工程师。

## 3. 技术栈与依赖

- 后端：Go 1.26.10（`go.mod`），HTTP 框架 go-chi/chi v5，路由分发见 `src/app/router/router.go`
- 依赖注入：google/wire（各模块 `wire.go`/`wire_gen.go`，如 `src/cmd/gitness/wire_gen.go`）
- 存储：PostgreSQL（jackc/pgx/v5、lib/pq）与 SQLite 双支持 + jmoiron/sqlx + golang-migrate 迁移（`src/app/store/database/migrate/{postgres,sqlite}/`）；Redis（go-redis v8）+ redsync 分布式锁/队列
- Git 实现：**封装 git CLI 而非纯 Go 重写**——`src/git/command/command.go:149` 用 `exec.CommandContext(ctx, GitExecutable, args...)`；运行镜像也基于 `alpine/git:2.49.1`（`src/Dockerfile` 最后一个 FROM）
- Git over SSH：gliderlabs/ssh（`src/ssh/server.go`）
- CI 引擎：drone-runner-docker（engine/engine2 双编译器，`src/app/pipeline/runner/runner.go`）、drone/runner-go、drone-yaml YAML 解析、go-scm SCM 抽象
- 制品仓库：distribution/distribution v3（Docker Registry 官方实现库）
- 容器编排（Gitspaces）：docker/docker client，devcontainer 规范实现（`src/app/gitspace/orchestrator/`）
- 其他：kingpin CLI、zerolog、go-jsonnet、kin-openapi（OpenAPI 文档）、Masterminds/squirrel SQL 构造器
- 前端：React + TypeScript + webpack + jest + cypress，`web/package.json` 名为 "codeui"，含主站/cde-gitness/ar（制品库）三个构建目标

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `cmd/gitness/` | 入口：kingpin CLI（server/migrate/user/users/account/swagger），wire 装配 |
| `app/` | 服务端主体（12MB）：api（controller/handler/middleware/openapi）、auth、pipeline、gitspace、services（约 45 个业务服务）、store（数据库层）、router、events（进程内事件总线） |
| `git/` | Git 抽象层：command（CLI 封装）、hook（server 端钩子）、storage、diff、merge、sharedrepo 等子包 |
| `registry/` | 制品仓库模块（6.6MB）：独立 app/api、store/migrations、gc、job、services |
| `web/` | React 前端（18MB，最大目录） |
| `types/`、`store/`、`errors/`、`events/`、`pubsub/`、`livelog/`、`lock/` | 跨模块共享：领域类型、存储接口、错误、事件、发布订阅、实时日志、锁 |
| `cli/`、`client/` | 客户端与 API SDK |
| `infraprovider/`、`ssh/`、`crypto/`、`encrypt/`、`secret/` | 基础设施供给、SSH 服务、加密与密钥 |
| `charts/` | Helm 部署图表 |
| `tests/` | 负载测试（`tests/load/`） |

组织方式：单体多包（非 monorepo 多语言混合，前端独立于 web/ 但同仓）。分层清晰：cmd → router → api(handler/controller) → services → store，横切能力（事件、锁、日志）下沉为顶层公共包；registry 与 app 内 gitspace 相对自治，各自带独立的 controller/store。约 2886 个 Go 文件（不含 web/）。

## 5. 核心架构与关键实现

### 5.1 Git 服务层：CLI 封装 + 三路由分发

与 Gitea 一致采用"git CLI + 裸仓库"路线而非 go-git 纯 Go 实现。`src/git/command/command.go` 统一封装进程执行、环境注入、错误解析（stderr 转 error），供 branch/commit/diff/merge 等 60 余个操作文件调用。服务端钩子通过 `git/hook/` 生成 githook 脚本回调 HTTP API（`src/app/githook/`）实现引用更新校验与 pre-receive 处理（`src/git/pre_receive_pre_processor.go`）。

入口路由按流量分三类，`src/app/router/router.go` 依次匹配：

```go
type Router struct {
    routers []Interface
}
func (r *Router) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ...
    for _, router := range r.routers {
        if ok := router.IsEligibleTraffic(req); ok {
            router.Handle(w, req)
            return
        }
    }
}
```

APIRouter 挂 `/api`（`app/router/api_router.go:27`），GitRouter 识别 `/git` 前缀或 `<space>/<repo>.git` 形式路径（`app/router/git_router.go:26-60`），WebRouter 服务 React SPA。SSH 走 `ssh/` 独立端口（容器内 3022）。

### 5.2 CI 引擎：Drone 管理器 + Docker runner 内嵌

流水线模块 `app/pipeline/`：triggerer 按 DAG 展开步骤（`triggerer/trigger.go`、`triggerer/dag/`），scheduler 管理执行队列与取消（`scheduler/scheduler.go`），manager 作为执行管理器向 agent 发放 72 小时临时 JWT（`manager/manager.go` 中 `pipelineJWTLifetime = 72 * time.Hour`）。runner 直接复用 Drone 官方 runner：

```go
import (
    "github.com/drone-runners/drone-runner-docker/engine"
    compiler2 "github.com/drone-runners/drone-runner-docker/engine2/compiler"
    ...
)
```

（`src/app/pipeline/runner/runner.go`）

即 CI 执行器就是 drone-runner-docker 本体，YAML 兼容性由 drone-yaml/go-convert 保证；webhook 事件驱动构建（`app/services/webhook/handler_check.go` 联动 status check）。这与 Drone CI 的差异主要是把"SCM 触发源"从外部插件换成内置仓库事件。

### 5.3 Gitspaces：嵌入式 Docker 容器编排

`app/gitspace/orchestrator/container/embedded_docker_container_orchestrator.go` 定义 `EmbeddedDockerOrchestrator`，把 devcontainer 生命周期拆为 step 序列（start/stop/resume），调用 docker client 拉起容器、执行 devcontainer 特性脚本（`orchestrator/devcontainer/exec.go`）、注入 IDE（`orchestrator/ide/`）。基础设施供给由 `infraprovider/` 抽象（当前主要是本机 Docker）。

### 5.4 制品仓库：distribution v3 内嵌

`registry/` 将 CNCF distribution（Docker Registry v3）作为库内嵌，自建 metadata API（`registry/app/api/controller/metadata/`，63 个文件）、GC（`registry/gc/`）、独立迁移（`registry/app/store/migrations/`），支持 OCI 镜像与通用制品，含生命周期策略与用量统计（`app/services/usage/`）。

### 5.5 平台支撑

事件驱动：`app/events/` + `events/` 提供进程内类型化事件总线（每域一个 reporter/consumer），SSE 推送（`app/sse/`）+ livelog 流式日志。AI 方面 OSS 部分仅有 `app/services/aitaskevent/` 与 `app/events/aitask/`（AI 任务事件管道，为 Harness 商业版 AI 能力留的接口，未见完整开源 agent 框架）。权限模型为 space/repo 层级 RBAC（`app/auth/`、`app/services/publicaccess/`），另含 webhook、protection 规则、merge queue、automerge 等托管级特性（`app/services/`）。

## 6. 构建与运行

来自 `README.md`、`Makefile`、`web/package.json`：

```bash
# 前端
cd web && yarn install && yarn build

# 后端（需 protoc v3.21.11、protoc-gen-go 等，先 make dep && make tools）
make build        # go build -o ./gitness ./cmd/gitness

# Docker 一键运行（README）
docker run -d -p 3000:3000 -p 3022:3022 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /tmp/harness:/data --restart always harness/harness
```

前端测试 `yarn test`（jest）/ cypress；Go 测试 `go test ./...`（Makefile 有 test 目标）；注意 CI 引擎依赖挂载 docker.sock。

## 7. 测试与工程质量

- Go 测试：333 个 `*_test.go`（全仓统计），覆盖 git 层（如 `git/repo_create_test.go`、`git/branch_last_page_test.go`）、store 层、webhook 等；另有 `tests/load/` 负载测试目录
- 前端：jest 单测 + cypress e2e（`web/package.json` scripts）
- CI：`.github/workflows/` 仅 `ci-lint.yml`（Go 与 TypeScript 分开 lint），未见公开的测试/发布 workflow（发版走 Harness 内部流水线，未确认）
- 质量观察：wire 显式依赖装配、接口与实现分离（store/services 均先定义 interface）、错误码体系（`errors/`）、migrations 与 registry 迁移分离；CONTRIBUTING.md 要求 CLA 签署与 pre-commit 钩子

## 8. 社区与维护状态

- 活跃度：gh api 显示最近推送 2026-11-12（两天前），持续高频开发
- 发版节奏：GitHub Releases 近半年 v2.27.0（2026-01-16）→ v2.27.1（02-25）→ v2.27.2（03-11）→ v2.28.0（03-27）→ v2.28.1（03-30）→ v2.28.2（04-20），约每月一版
- 贡献者：约 95 名（含匿名，gh api Link 头估算）；核心由 Harness 公司员工主导（Copyright 头均为 "Harness, Inc."，CONTRIBUTING 要求 CLA）
- 商业背景：仓库背后是 Harness 商业 DevOps 平台，开源版为其一体化平台的自管形态；issue 133 open
- 社区渠道：Harness Community forum（CONTRIBUTING.md）；文档站 developer.harness.io

## 9. 亮点与风险

**亮点**
1. 四合一平台单个 Go 二进制交付（代码托管+CI+Gitspaces+制品库），部署运维成本低（`Dockerfile`、README）
2. Drone CI 血统的成熟执行引擎：直接复用 drone-runner-docker 与 drone-yaml，YAML 兼容 Drone 生态（`app/pipeline/runner/runner.go`、go.mod）
3. Git 层抽象完整：CLI 封装、githook 回调、diff/merge/sharedrepo 独立子包，边界清晰（`git/` 目录）
4. 工程化程度高：wire DI、类型化事件总线、双数据库迁移、OpenAPI 契约、前端三构建目标
5. Gitspaces 把 devcontainer 规范服务化，是同类自托管方案中少见的完整实现（`app/gitspace/`）

**风险/局限**
1. 公司主导 + CLA 门槛：社区贡献者话语权有限，路线图跟随 Harness 商业利益（CONTRIBUTING.md）
2. 命名混乱成本：仓库名 harness、模块名 gitness、UI 名 codeui，历史包袱影响检索与理解
3. Git 走 CLI 子进程：高并发下进程开销与版本依赖（镜像钉死 alpine/git 2.49.1）是天然瓶颈
4. OSS 版 AI/高级能力仅为事件接口（`app/services/aitaskevent/`），与商业版能力边界需自行评估

## 10. 总结评价

harness/harness 是"Drone 作者团队把 Gitness 与 Drone 合体"的一体化开源 DevOps 平台，38905 星、月度发版、持续活跃，代码质量与分层设计在同类自托管平台中属第一梯队。它适合中小团队私有化部署"一站式"研发平台，也适合研究 SCM 服务端架构、Drone 式 CI 调度和 devcontainer 云开发环境实现的人精读源码；对比 Gitea（轻量代码托管+插件 CI）它重而全，对比 GitLab（开源核心+商业高级）它的开源边界更慷慨但治理更公司化。
