# wagoodman/dive 项目分析

> 星数 54664（快照 2026-10-24）｜主语言 Go｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：**一个探索 Docker/OCI 镜像每一层内容、并估算镜像"空间浪费"的终端交互工具**，既支持 TUI 逐层浏览，也支持 CI 中作为门禁做 pass/fail 判定。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | wagoodman/dive | — |
| 星数 | 54664（快照）；gh api 实测 54694 | 任务参数 / gh api |
| 语言 | Go（go.mod 声明 go 1.26.8） | `go.mod` |
| 许可证 | MIT（Copyright 2018 Alex Goodman） | `LICENSE`、gh api |
| 创建时间 | 2018-05-13 | gh api |
| 最近提交 | 2026-10-30 "port image loading and analysis to stereoscope (#727)"，pushed_at 2026-11-02 | gh api commits |
| 最新 release | v0.13.1（2025-03-29） | gh api releases |
| 规模 | 非 Go 测试代码约 10135 行，测试代码约 5199 行；125 个 .go 文件 | 本地统计 |
| 贡献者 | 约 59 人（contributors 分页 Link 头 page=59 × per_page=1） | gh api |
| Open issues | 202 | gh api |

## 2. 项目是什么，解决什么问题

Docker 镜像由只读层堆叠而成，`docker images` 只能看到整镜像大小，无法回答"这 500MB 到底是哪一层塞进来的、有多少是重复/被覆盖/删除后仍占空间的文件"。dive 解决的就是这个可观测性问题：它把镜像逐层解开，左侧列出每一层及其 Dockerfile 指令，右侧展示该层叠加后（或聚合到该层为止）的文件树，并用颜色标注文件是新增/修改/删除/未变（`README.md` Basic Features 节）。

第二个核心卖点是"镜像效率"估算：对每个路径统计它出现在多少层里、累计占了多少字节，按"最小必要体积 / 实际累计体积"给出一个 0-1 分数，同时列出浪费空间最大的文件路径（`README.md`；算法实现在 `dive/analysis.go`）。用户由此能发现典型反模式：在多层里复制同一文件、`RUN` 中先解压后删除（删除并不减小镜像）、跨层移动文件等。

第三个入口是 CI 集成：`CI=true dive <image>` 跳过 TUI，按 `.dive-ci` 中三条规则（`lowestEfficiency`、`highestWastedBytes`、`highestUserWastedPercent`）输出 pass/fail 并决定退出码（`README.md` CI Integration 节；规则解析在 `cmd/dive/cli/internal/command/ci/rules.go`）。这让镜像瘦身可以进入流水线门禁，是 dive 在团队场景下区别于单纯查看工具的关键。

项目由 Alex Goodman 独立发起（2018 年），后来他加入 Anchore 后，dive 的工程底座逐步换用 Anchore 系库（clio/stereoscope/bubbly）；最近一次大改版（2026-10，#727）把镜像加载与分析整体迁移到 stereoscope 之上，源码组织也随之重构（gh api commits；`dive/analysis.go` 头部注释直接引用 stereoscope 类型）。README 自述 "This is beta quality"，但多年过去它已是 Docker 生态事实上的经典工具。

## 3. 技术栈与依赖

- 语言：Go 1.26.8（`go.mod`）。
- 镜像解析：`github.com/anchore/stereoscope` —— 统一抽象 docker/podman/containerd/containers-storage/OCI 归档/registry 等镜像来源并构建分层文件树（`internal/source/source.go` 的 import；`go.mod`）。这是 v0.13 后 dive 的地基，dive 自身不再直接解 tar。
- CLI 框架：`spf13/cobra` + `github.com/anchore/clio`（配置/日志/UI 生命周期），配置用 viper 系与 `pelletier/go-toml`、`gopkg.in/yaml.v3`（`cmd/dive/cli/cli.go`、`go.mod`）。
- 交互 TUI：`github.com/awesome-gocui/gocui`（主界面）+ `anchore/bubbly`（布局组件）（`cmd/dive/cli/internal/ui/tui/app/app.go` import）。
- 加载进度 UI：`charmbracelet/bubbletea` + `lipgloss`（`internal/source/source.go` 之外的 `cmd/dive/cli/internal/ui/progress_ui.go`；`go.mod`）。
- 事件与进度：`wagoodman/go-partybus`（事件总线）、`wagoodman/go-progress`（作者自己的库），stereoscope 的事件也发布到同一条 bus（`cmd/dive/cli/cli.go` Initializers、`internal/bus/`）。
- Docker API：`moby/moby/client`、`docker/cli`；registry 访问用 `google/go-containerregistry`（`go.mod`）。
- 测试：`stretchr/testify` + `gkampitakis/go-snaps`（快照测试）、`google/go-cmp`（`go.mod`）。

## 4. 目录结构与代码组织

顶层（本仓库为标准 Go 单模块布局，`go.mod` 在根）：

| 目录 | 职责 |
|---|---|
| `cmd/dive/` | `main.go` 入口 + `cli/` 全部 CLI 实现 |
| `cmd/dive/cli/internal/command/` | root/run/build 三个命令；子包 `ci/`（规则评估）、`export/`（JSON 导出） |
| `cmd/dive/cli/internal/options/` | 11 个文件拆分的应用配置（分析源、CI、导出、TUI 偏好等） |
| `cmd/dive/cli/internal/ui/` | UI 分发：progress UI（bubbletea）、line UI、no UI，及 explore 桥接 |
| `cmd/dive/cli/internal/ui/tui/` | gocui 主界面：`app/`（事件循环、controller、截图/extract）、`view/`（layer/filetree/details/filter/status 等 11 个视图）、`filetree/`（13 个文件：树渲染、比较、折叠）、`layout/`、`viewmodel/`、`key/`、`format/` |
| `cmd/dive/cli/internal/ui/handler/` | partybus 事件 → 进度条映射（13 个文件，快照测试覆盖） |
| `dive/` | **核心分析库**（2 个文件）：`analysis.go` 定义 `Analysis/Layer/Inefficiency` 与效率算法 |
| `internal/source/` | 镜像来源解析：`source.go`（9 种来源 + scheme 解析）、`build.go`（dive build）、`docker.go`、`podman.go` |
| `internal/layertree/` | 跨层文件树遍历与 tar header 查询的共享助手 |
| `internal/bus/` | 事件总线与 payload/parser |
| `internal/imagefixture/` | 测试镜像 fixture 生成 |
| `.data/` | 预构建的测试镜像 tar（gzip/zstd/estargz/oci/kaniko/未压缩等变体）与 demo 图 |
| `.github/` | workflows（validations、release）、dependabot、issue 模板 |

组织方式评述：单体单模块、按 `cmd/` + `internal/` 分层，且刻意把可复用的"分析内核"（`dive/` 包，约 250 行）与"应用外壳"（`cmd/dive/cli`）分离——内核只依赖 stereoscope 的 `image.Image`，外壳负责取镜像、跑 UI/CI/导出。`internal/` 严格私有，公共 API 只有 `dive` 包，边界清晰，是 Go 项目里教科书式的"小内核 + 厚壳"结构。

## 5. 核心架构与关键实现

### 5.1 镜像来源解析（`internal/source/source.go`）

定义 9 种来源常量与别名，`Parse()` 把用户输入拆成 (source, ref)：支持 `<source>://<ref>` 全量 scheme；对文件型来源（docker-archive/oci-archive/oci-dir）额外支持 `<source>:<path>` 单冒号形式，其余单冒号（如 `docker:dind`）按"镜像名:tag"处理（`internal/source/source.go` L29-L94）。`Auto` 来源按"文件 → 守护进程/存储 → registry 拉取"顺序探测并在结果中报告实际来源（同文件 L39-L43 注释）。`Cleanup()` 用全局 TempDir 生成器 + WaitGroup 保证中断时也能清掉临时目录（L101-L121）。具体连接 docker/podman 的逻辑分别在 `docker.go`（尊重 `DOCKER_HOST`/context）与 `podman.go`（Linux 直读本地 store，macOS 走 podman machine socket）。

### 5.2 效率评分算法（`dive/analysis.go`）

这是 dive 的灵魂，核心函数 `imageEfficiency()`（L142-L214）：

```go
// imageEfficiency scores the image and lists paths found in more than one layer. Loosely:
//  1. files duplicated across layers discount the score, weighted by file size
//  2. removed files discount the score, weighted by the original file size
```

实现要点：对每一层调用 `layertree.Walk` 遍历该层"自身树"的叶子节点；普通文件按 tar header 大小记一次出现；whiteout（`.wh.` 前缀）视为对下层同名路径的一次出现，大小取自下层 squash 树（目录按整棵子树求和，普通文件记 0——它的字节已在上层出现时计过）；opaque whiteout（`.wh..wh..opq`）直接跳过，不把被隐藏的下层文件当删除（`dive/analysis.go` L142-L188、`internal/layertree/layertree.go` L49-L53）。最终：

```go
score := 1.0
if discovered != 0 {
    score = float64(minimum) / float64(discovered)
}
```

即"每条路径只保留最小一次出现的体积之和 / 所有层累计体积之和"（`dive/analysis.go` L190-L214）。出现在多层的路径记为 `Inefficiency`，按浪费体积降序输出。`layerInfo()` 把层与镜像 history 对齐（跳过 `empty_layer` 的元数据指令、剥离 `/bin/sh -c` 前缀，缺失记 "(missing)"）（L102-L126）。算法刻意只用 tar header 元数据、不解压文件内容，因此分析很快。

### 5.3 主流程与事件总线（`cmd/dive/cli/internal/command/run.go`）

`run()` 是所有命令的公共骨架：`source.Parse` → `load`（发 partybus 任务事件）→ `analyze` → 按 `--json`/`CI=true`/默认三分支走向导出、评估或 TUI（L63-L98）。镜像临时目录用 `defer img.Cleanup()` + 进程级 `source.Cleanup()` 双保险（L72-L75）。错误经 `escapedError` 包装转义后再打印，因为加载错误可能携带 registry/daemon 返回的任意控制字符（L46-L61）。CLI 装配在 `cmd/dive/cli/cli.go`：clio 统一配置/日志，`stereoscope.SetBus(state.Bus)` 让底层拉镜像进度直接驱动 UI（L29-L39）。

### 5.4 交互式 TUI（`cmd/dive/cli/internal/ui/tui/`）

主界面基于 gocui：`app/app.go` 的 `Run()` 初始化 GUI、注册 Ctrl+Z 与 ctx 取消联动，`controller.go`（252 行）管理视图切换与按键分发；`view/` 拆出 layer、filetree、image_details、layer_details、filter、status 等视图；`filetree/` 子包负责树比较与折叠渲染（目录结构见第 4 节）。两层 UI 分工明确：镜像加载期用 bubbletea 写的 progress UI（`internal/ui/progress_ui.go`），进入分析结果后切换到 gocui TUI；`dive build` 等需要与子进程共享终端的场景强制 line UI（`run.go` L36-L44）。键位与 UI 均可通过 `~/.config/dive/*.yaml` 配置（`README.md` UI Configuration 节）。值得注意的是 `app.go` L33-L36 保留了作者自认的 hack：为规避容器内 termbox 初始化偶发阻塞，硬编码 `time.Sleep(100ms)`，注释标注 "todo: remove this hack"。

## 6. 构建与运行

来自 `README.md` 与 `Taskfile.yaml`/`Makefile`：

```bash
# 安装（各发行版包管理器均收录，另有 brew/snap/choco/scoop/winget）
go install github.com/wagoodman/dive@latest        # 需 Go 1.26+
# linux 需要 containers-storage 支持时：
go build -tags containers_image_openpgp,exclude_graphdriver_btrfs ./cmd/dive

# 本地开发（Makefile 是 binny/task 的 shim，实际命令在 Taskfile.yaml）
task build          # 构建
task test           # = task unit + task cli
task lint           # gofmt + golangci-lint（.golangci.yaml）

# 使用
dive <image-tag>                          # 交互分析
dive build -t <tag> .                     # 构建后直接分析
CI=true dive <image>                      # CI 门禁（读 .dive-ci 规则）
dive <image> --json out.json              # 跳过 UI 导出 JSON
dive oci-dir:./layout --source auto      # 9 种来源任选
```

发布走 `.goreleaser.yaml` + `.github/workflows/release.yaml`，Docker 镜像基于 alpine:3.21 打包静态 docker CLI（`Dockerfile`）。

## 7. 测试与工程质量

- 规模：36 个 `*_test.go`，约 5199 行测试代码对约 10135 行实现（约 1:2），核心包 `dive/analysis_test.go`、`internal/layertree/layertree_test.go`、`internal/source/source_test.go`（501 行）均有针对性单测。
- 风格分三层：纯单测；快照测试（go-snaps，`cmd/dive/cli/internal/ui/handler/__snapshots__/`）；以及 CLI 级 golden 测试——`cmd/dive/cli/` 下 7 个测试直接跑整条 CLI 并对比 `testdata/snapshots/*.snap`，配合 `.data/` 里 10 余个预构建的多压缩格式镜像 tar 做真实数据驱动（目录清单见第 4 节）。
- 质量门禁：`Taskfile.yaml` 单测设 25% 覆盖率阈值（`.github/scripts/coverage.py` 校验）；golangci-lint（`.golangci.yaml`）；CI workflow `validations.yaml` + release workflow `release.yaml`，dependabot 自动升依赖，`.bouncer.yaml`/`.binny.yaml` 是 Anchore 系发布与工具链锁版本设施。
- 观察：代码注释密度高且都在解释"为什么"（如 efficiency 算法的 whiteout 计分规则），README 与代码中标注实验性行为（"experimental metric"）的方式诚实。

## 8. 社区与维护状态

- 历史节奏（gh api commits/releases，实测）：2018 年创建，2021-03 发 v0.10.0，2023-07 发 v0.11.0，2024-02 发 v0.12.0，2025-03 连发 v0.13.0/v0.13.1；2025-04 至 2025-05 之间基本只剩 dependabot 依赖 bump；最新功能提交是 2026-10-30 的 "port image loading and analysis to stereoscope (#727)"（为当前 tarball HEAD 的直接来源），pushed_at 2026-11-02。整体呈"长期间歇维护（依赖升级为主）+ 偶发大规模重构"的模式，不是每日活跃型项目。
- 值得注意：2026-10 的 stereoscope 迁移尚未伴随 v0.14 release（最新 release 仍为 2025-03 的 v0.13.1），说明当前 HEAD 领先于已发布版本（未确认是否即将发版）。
- 贡献者约 59 人（分页推算），但代码主线长期由作者一人主导，近期大改版亦为单 PR（#727）。issue 积压 202 个，fork 2010。
- 社区分发极广：deb/rpm/snap/arch/brew/macports/choco/scoop/winget/nix 全覆盖（`README.md` Installation 节），Arch extra 仓库与 Docker Hub/ghcr 官方镜像均有收录。

## 9. 亮点与风险

**亮点**
1. 产品定位极准：把"镜像分层可观测性 + 浪费空间量化"做成了一个只读、零依赖（无需 root、无需改 Dockerfile）的单一 CLI，54k+ 星是刚需验证。
2. 算法简单可解释：效率分 = min/discovered（`dive/analysis.go`），只用 tar 元数据不解压内容，快且结果可人工复核，并可直接映射为 CI 三规则（`command/ci/rules.go`）。
3. 架构边界干净：约 250 行的分析内核（`dive/` 包）与厚应用壳分离，镜像获取全部委托 stereoscope，一次迁移同时获得 9 种来源与多压缩格式（zstd/estargz/OCI）支持。
4. 测试策略成熟：单测 + go-snaps 快照 + CLI golden 三层，用 10 余个真实镜像 tar 做数据驱动，覆盖率有硬阈值。
5. 事件总线（partybus）把加载进度、错误 UI 与主流程解耦，stereoscope 的事件也能无缝接入（`cmd/dive/cli/cli.go` L34-L36）。

**风险/局限**
1. 维护节奏低频且不连续（2025-05 至 2026-10 间几乎只有 dependabot），issue 积压 202；重大变更（stereoscope 迁移）落地后长期未发版。
2. 效率算法是启发式：opaque whiteout 隐藏的下层文件不计为浪费、删除文件只在目录维度计子树大小等取舍都有偏差空间，README 自己标注 experimental，不能当精确账本用。
3. TUI 内残留已知 hack（termbox 初始化前硬编码 sleep，`tui/app/app.go` L33-L36），且 README 仍自称 "beta quality"。
4. 行为对容器生态版本敏感：README 提示旧版 Docker 需 `DOCKER_API_VERSION` 覆盖、Snap 与 apt 版 Docker 冲突（issue #546）等兼容性坑，文档只能缓解不能消除。

## 10. 总结评价

dive 是"小而准"的典范：用一个约 250 行、纯元数据的启发式算法回答了每个 Docker 用户都问过的问题——"这个镜像为什么这么大、白占了多少空间"，再用双栏 TUI 和 CI 门禁把答案变成日常动作与流水线卡点。工程上它是 Go CLI 项目的好教材：分析内核与应用壳分离、事件总线解耦、三层测试加真实镜像 fixture、clio/stereoscope 等基础设施外包给专门的库。主要不确定性来自维护模式：作者主导、间歇性投入，HEAD 与最新 release 之间隔着一次大重构，选型时应优先使用发行版收录的稳定版本而非 main。适合所有做容器交付的团队把它放进 CI；也适合想学 Go 终端应用架构（gocui + bubbletea + partybus + cobra/clio）的开发者精读 `dive/analysis.go`、`internal/source/` 与 `cmd/dive/cli/internal/ui/` 这条主线。
