# kubernetes/kubernetes 项目分析

> 星数 128578（快照 2026-10-24）｜主语言 Go｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：Kubernetes 是容器编排领域的事实标准系统—— declarative（声明式）的分布式控制平面，负责跨主机的容器化应用的部署、维护与弹性伸缩。

| 项目 | 内容 |
| --- | --- |
| 名称 | kubernetes/kubernetes |
| 星数 | 128578（快照 2026-10-24） |
| 主语言 | Go（go.mod 声明 go 1.27.0） |
| 许可证 | Apache-2.0（`LICENSE`） |
| 默认分支 | master（gh api 确认） |
| 最近 push | 2026-10-29（gh api，即分析前一天） |
| 最新 release | v1.38.0-alpha.3（2026-10-28，`CHANGELOG/CHANGELOG-1.38.md` 对应） |
| 代码规模 | pkg+cmd 非测试 Go 代码约 69.1 万行，staging 约 141 万行（find+wc 实测） |

## 2. 项目是什么，解决什么问题

Kubernetes（K8s）是一个开源的容器编排系统，源自 Google 内部 Borg 十五年生产实践的经验总结（`README.md` 开篇即明确这一传承），现托管于 CNCF。它要解决的核心问题是：当应用被拆成成百上千个跨主机的容器实例后，如何让它们的部署、故障恢复、伸缩、服务发现、配置与存储挂载全部自动化，而不依赖人工操作。

K8s 的核心设计是**声明式 API + 控制循环**：用户通过 YAML/JSON 向 API Server 提交期望状态（如"维持 3 个副本"），各个控制器持续对比实际状态与期望状态并做调谐（reconcile）。仓库本身就是这一模型的完整实现——`pkg/registry/` 下按 API 组组织的资源存储、`pkg/controller/`（585 个 Go 文件）下的控制器实现、`pkg/scheduler/`（249 个文件）下的调度器，全部围绕这一循环工作。

仓库内包含一整套可独立运行的二进制：控制平面的 kube-apiserver / kube-controller-manager / kube-scheduler，节点侧的 kubelet / kube-proxy，以及部署工具 kubeadm 和客户端 kubectl（`cmd/` 目录）。apiserver 是唯一与 etcd 交互的组件，其余组件全部通过 watch apiserver 协作，这是理解 K8s 架构的关键。

需要特别说明的是该仓库的双重角色：它既是各组件的**实现载体**，也是 `k8s.io/*` 全家桶库（client-go、apimachinery、apiserver 框架等）的**源码母仓**——`staging/src/k8s.io/` 下的 33 个模块随发版发布为独立 Go 库供生态使用。同时 README 明确声明：将 `k8s.io/kubernetes` 模块本身作为库引用是不受支持的用法。

## 3. 技术栈与依赖

- **语言**：纯 Go（`go.mod`：go 1.27.0，godebug default=go1.27）。
- **存储**：etcd 客户端 `go.etcd.io/etcd/client/v3 v3.7.3`（go.mod），apiserver 所有资源持久化走 `staging/src/k8s.io/apiserver/pkg/storage/etcd3/`。
- **通信**：gRPC v1.85.0（kubelet 与 CRI 运行时、apiserver 内部）；HTTP 框架用标准 net/http + `github.com/emicklei/go-restful/v3`（go.mod）。
- **可观测**：Prometheus 客户端（prometheus/client_golang）+ OpenTelemetry v1.47.0 全链路 tracing（go.mod 中 otelrestful/otelhttp 插桩）。
- **策略/校验**：CEL（cel-go v0.33.1）用于准入策略、CRD 校验规则；`sigs.k8s.io/structured-merge-diff/v7` 支撑字段级合并（server-side apply）。
- **依赖管理**：vendor 目录完整入库（`vendor/` 202 个模块、4364 个 Go 文件），go.mod 头注释明确"generated file, do not edit"，依赖变更必须走 `hack/pin-dependency.sh` + `hack/update-vendor.sh`（也是 `AGENTS.md` 的硬性约束）。
- **代码生成**：大量依赖 code-generator（`staging/src/k8s.io/code-generator`）生成 clientset/informer/lister 与 zz_generated 深拷贝/转换/protobuf 代码，`pkg/generated/` 即生成产物。

## 4. 目录结构与代码组织

顶层目录（实测 `ls`）：

| 目录 | 职责 |
| --- | --- |
| `cmd/` | 26 个命令入口：kube-apiserver、kube-controller-manager、kube-scheduler、kubelet、kube-proxy、kubeadm、kubectl、cloud-controller-manager 及文档/检查类工具 |
| `pkg/` | 主体实现，30 个子目录：kubelet（714 文件）、apis（688）、controller（585）、registry（445）、scheduler（249）、volume、proxy、controlplane、kubectl 等 |
| `staging/src/k8s.io/` | 35 个待发布库的源码（apiserver、client-go、apimachinery、kubectl、kubelet、cri-api 等），go.work 声明 33 个 use 模块 |
| `api/` | OpenAPI/发现文档等 API 描述产物 |
| `build/` | 构建系统（`build/root/Makefile` 是根 Makefile 的符号链接目标） |
| `hack/` | 122 个开发脚本：构建、lint（golangci.yaml）、e2e（ginkgo-e2e.sh）、local-up-cluster.sh 等 |
| `test/` | e2e、e2e_node、integration、conformance、fuzz 等大型测试目录 |
| `vendor/` | 全量 vendored 依赖 |
| `CHANGELOG/` | v1.11~v1.38 全量变更日志 |
| `plugin/`、`cluster/`、`docs/`、`logo/`、`LICENSES/` | 准入插件、云部署脚本、占位 docs（仅 OWNERS）、logo、第三方许可 |

组织方式评述：这是典型的"**单体母仓 + staging 伪多模块**"架构。主模块 `k8s.io/kubernetes` 内部代码与 staging 里的发布库同仓共栖，用 `go.work`（`go.work`）把 33 个 staging 模块拼成可开发的 workspace；发布时 staging 目录整体对外。约束非常明确：staging 代码禁止反向 import 主模块（`AGENTS.md`）。API 类型层与实现层严格分离：`pkg/apis/` 只放内部版本类型，`pkg/registry/<组>/<资源>/storage/` 放存储适配，`pkg/apis/` 下 27 个 API 组与 `pkg/registry/` 下 23 个组一一对应。此外 `docs/` 与 CI 均外置（docs 仅含 OWNERS；`.github/` 无 workflows），文档在 kubernetes/website，CI 在 kubernetes/test-infra（Prow）。

## 5. 核心架构与关键实现

### 5.1 kube-apiserver：聚合链与资源装配（深读）

apiserver 入口 `cmd/kube-apiserver/app/server.go:187` 定义了 `CreateServerChain`：把 **kube-aggregator（聚合 API）→ apiextensions（CRD）→ kube-apiserver（原生资源）** 三层 apiserver 用 Delegation 机制串成一条 handler 委托链，外层聚合器按 APIGroup 把请求路由给内层。`pkg/controlplane/apiserver/server.go` 中的 `Server` 结构与 `completedConfig.New()` 展示了装配方式：底层先构造 `genericapiserver.GenericAPIServer`（来自 `staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go` 的通用框架），再挂载资源配置、RESTOptionsGetter（决定 etcd 存储参数）、informers 与租约型 leader 选举控制器（`pkg/controlplane/controller/leaderelection`）。

每个 API 组的存储通过 registry 模式装配，以核心组为例（`pkg/registry/core/rest/storage_core.go`）：pod、node、service 等 30 余种资源各自在 `pkg/registry/core/<资源>/storage/` 下有 storage 适配器，统一以 `RESTOptionsGetter` 拿到 etcd-backed 的 generic registry store（框架在 `staging/src/k8s.io/apiserver/pkg/registry/generic/`）。apiserver 侧的准入（`pkg/admission/` + `plugin/`）、鉴权（`pkg/auth/`）、流控（APF，`pkg/registry/flowcontrol/`）均以插件形式插入 handler 链。

```go
// pkg/controlplane/apiserver/server.go
type Server struct {
    GenericAPIServer *genericapiserver.GenericAPIServer
    APIResourceConfigSource   serverstorage.APIResourceConfigSource
    RESTOptionsGetter         genericregistry.RESTOptionsGetter
    ClusterAuthenticationInfo clusterauthenticationtrust.ClusterAuthenticationInfo
    VersionedInformers        clientgoinformers.SharedInformerFactory
}
```

### 5.2 调度器框架：扩展点插件模型（深读）

`pkg/scheduler/` 的现代架构是"框架 + 插件"：调度一个 Pod 要经过 QueueSort → PreFilter → Filter → PostFilter（抢占）→ Score → Reserve → Permit → PreBind → Bind 等扩展点，每个扩展点由多个可配置插件按序执行，Profile（`pkg/scheduler/profile/`）允许同一调度器内多套插件配置并存。核心执行逻辑在 `pkg/scheduler/framework/runtime/framework.go:1128` 的 `RunFilterPlugins`：串行跑所有 Filter 插件，任一拒绝即放弃该节点，状态上标注来源插件。

```go
// pkg/scheduler/framework/runtime/framework.go:1128
func (f *frameworkImpl) RunFilterPlugins(
    ctx context.Context, state fwk.CycleState,
    pod *v1.Pod, nodeInfo fwk.NodeInfo,
) *fwk.Status {
    for _, pl := range f.filterPlugins {
        if state.GetSkipFilterPlugins().Has(pl.Name()) {
            continue
        }
        if status := f.runFilterPlugin(ctx, pl, state, pod, nodeInfo); !status.IsSuccess() {
            status.SetPlugin(pl.Name())
            return status
        }
    }
    return nil
}
```

主循环在 `pkg/scheduler/schedule_one.go`（含 `schedulePod` 及 PodGroup 批量调度雏形 `schedule_one_podgroup.go`），EventHandlers（`eventhandlers.go`）维护调度队列与缓存（`pkg/scheduler/backend/`）和 informer 的增量一致。

### 5.3 kubelet：节点代理的模块化（概览）

`pkg/kubelet/`（714 个 Go 文件）是最大的实现包。入口 `kubelet.go`，围绕 Pod 的生命周期由 `pod_workers.go` 按上 Pod 级并发 worker 驱动；关键子系统包括：PLEG（`pleg/`，容器运行时事件重放推断）、CRI 运行时接入（`kuberuntime/`）、存活/就绪探测（`prober/`）、资源驱逐（`eviction/`）、cAdvisor 监控（`cadvisor/`）、卷插件（`volume/` 在 pkg 层另有 190 文件）以及容器/镜像 GC（`cm/` 容器管理器）。OS 差异用 `_linux`/`_windows`/`_others` 后缀文件隔离（如 `kubelet_network_linux.go`）。

## 6. 构建与运行

来自 `build/root/Makefile`（根目录 `Makefile` 为其符号链接，tarball 中符号链接解压失败但目标文件存在）与 `hack/` 脚本：

```bash
make all                 # 构建全部本地平台的组件二进制
make test                # 单元测试
make test-integration    # 集成测试（需 etcd，见 hack/install-etcd.sh）
make test-cmd            # kubectl 命令行测试
make verify              # 全量静态校验（license、gofmt、生成代码一致性等）
make lint                # golangci-lint（配置 hack/golangci.yaml）
make release-images      # 构建正式发布镜像（含 conformance）
hack/local-up-cluster.sh # 本地拉起一套单节点集群（开发用）
```

官方构建方式是容器内构建（`build/` 下 build/run via Docker），非 Linux 平台推荐 `build/run.sh make`。本分析未运行构建（规范约束，仅静态阅读）。

## 7. 测试与工程质量

- **规模**：pkg+cmd 下测试代码约 78 万行（1141 个 `_test.go` 文件），**超过**非测试代码的 69 万行，测试/代码比 > 1:1（find+wc 实测）。staging 库同样内嵌大量测试。
- **分层**：`test/integration/`（apiserver 级集成）、`test/e2e/`（集群级，Ginkgo/Ginkgo v2）、`test/e2e_node/`（节点级，独立框架）、`test/conformance/`（一致性保证）、`test/fuzz/`（模糊测试）——五层金字塔齐全（`test/` 目录实测）。
- **静态质量**：golangci-lint 配置入库（`hack/golangci.yaml`），另有自研检查器 `hack/kube-api-linter/`、import-boss（依赖方向检查，`cmd/import-boss/`）、API 规范检查脚本等；`hack/verify-*.sh` 家族保证生成代码、license 头、protobuf 一致性。
- **CI**：仓库内无 GitHub Actions 配置（`.github/` 仅模板与 SECURITY.md），全部 CI 由 kubernetes/test-infra 的 Prow 驱动，PR 需通过数千个 CI job 矩阵——这是该仓库质量体系最显著的外部特征。

## 8. 社区与维护状态

- **活跃度**：最近 push 2026-10-29（gh api），即分析前 1 天；处于 1.38 开发周期。open issues 约 3227，forks 约 4.6 万（gh api）。
- **发布节奏**：每年 3 个 minor 版本；当前同时维护 5 条版本线——2026-10-28 一天内同时发布 v1.38.0-alpha.3、v1.37.2、v1.36.6、v1.35.10、v1.34.13（gh api releases），补丁维护窗口长达两年。
- **贡献者**：contributors API 分页累计约 5976 人（含匿名）；实际有 commit 的规模在数千级别，SIG 治理下有严格的 OWNERS 文件链（每个目录都有 `OWNERS`，顶层 `OWNERS_ALIASES`）。
- **流程证据**：`.github/PULL_REQUEST_TEMPLATE.md`、`CONTRIBUTING.md`、`AGENTS.md`（对 AI 协作的硬性规范：生成文件只读、禁止 go mod tidy、PR 须披露 AI 使用情况）；`hack/cherry_pick_pull.sh` 是版本线 cherry-pick 的标准化工具。

## 9. 亮点与风险

**亮点**

1. 架构本身就是教材：声明式 API、控制循环、层级式 apiserver 委托链、框架化调度器，每个设计在代码里都有清晰落点（`pkg/controlplane/apiserver/`、`pkg/scheduler/framework/`）。
2. 工程纪律极端严格：vendor 锁定、生成文件只读、import-boss 依赖方向管控、五层测试体系、测试代码量超过实现代码（第 3/7 章实测数据）。
3. staging 母仓模式兼顾单体开发与库生态发布：33 个 `k8s.io/*` 模块同仓开发、独立发版（`go.work` + `staging/src/k8s.io/`）。
4. 可维护性设计到位：OS 平台文件后缀隔离、OWNERS 分级治理、CHANGELOG 自 1.11 起全量保留、klog 结构化日志与 OpenTelemetry 全链路插桩。
5. 对 AI 时代的协作也有明确规范（`AGENTS.md`），在超大型开源项目中属于先行者。

**风险/局限**

1. 规模门槛极高：仅 pkg+cmd 就 69 万行非测试代码，新人定位一个改动往往要横跨 staging、pkg/apis、pkg/registry 三层。
2. 兼容性包袱重：API 版本迁移、废弃 API（`pkg/apis/extensions/` 等遗留组）、`k8s.io/kubernetes` 不支持作为库引用等约束，都是生态长期演化的代价。
3. 仓库内文档刻意稀薄（`docs/` 仅 OWNERS），一切依赖外置仓库（website、test-infra、community），离线阅读源码时上下文获取成本高。
4. 部分核心路径仍保留 legacy 结构（如 `pkg/api/legacyscheme`、apiserver 内 leader 租约控制器），新旧机制并存增加理解负担。

## 10. 总结评价

kubernetes/kubernetes 是全球工程化水平最高的一档开源项目：128k+ 星的背后不是个人英雄主义，而是 CNCF 治理、SIG 分工、Prow 流水线和 OWNERS 体系共同支撑的"制度型开源"。它既是生产级容器编排的标准实现，也是 Go 大型系统设计的公开教科书——apiserver 的插件化框架、scheduler 的扩展点模型、client-go 的 informer 机制被无数后继项目借鉴。适合云计算基础设施工程师精读其架构分层（尤其 staging/src/k8s.io/apiserver 与 pkg/scheduler/framework），适合平台团队作为参考实现校准自己的控制循环设计；但对只想"拿来看"的初学者，建议从 client-go 和 sample-controller（staging 内）入手，而不是直接扎进主模块。想要自己搭集群做实验，走 kubeadm 或 hack/local-up-cluster.sh 即可。
