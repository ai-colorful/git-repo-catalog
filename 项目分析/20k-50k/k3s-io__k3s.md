# k3s-io/k3s 项目分析

> 星数 29000（快照 2026-10-24）｜主语言 Go｜档位 20k-50k｜分析日期 2026-11-21
> （gh api 实测 2026-11-21：34,473 stars / 2,790 forks / 122 open issues，本文以快照值为准）

## 1. 项目速览

一句话定位：**单二进制、CNCF 认证合规的轻量级 Kubernetes 发行版**——把 kube-apiserver/kubelet 等控制面与节点组件合并进一个 <100MB 的 Go 进程，默认用内嵌 SQLite（经 kine shim）替代 etcd。

| 事实 | 值 | 出处 |
|---|---|---|
| 仓库 | k3s-io/k3s，默认分支 main | gh api |
| 星数/创建 | 29,000（快照）/ 2018-05-31 创建 | 任务参数 / gh api |
| 语言/许可证 | Go 1.26.10（`go.mod`）｜Apache-2.0（`LICENSE`） | 源码 |
| 最近推送 | 2026-11-20（分析前一日，极活跃） | gh api |
| 当前版本线 | v1.37.2+k3s1（对应上游 K8s v1.37.2），同时维护 v1.34–v1.37 四条 minor | `channel.yaml`、gh api releases |
| 主导方 | SUSE/Rancher（16 名核心维护者邮箱几乎全部为 @suse.com） | `MAINTAINERS` |
| 治理 | CNCF 项目（CLOMonitor 注册于 cncf/k3s，conformance 认证） | `.clomonitor.yml`、`README.md` |

## 2. 项目是什么，解决什么问题

K3s 官方自我定位是"发行版（distribution）而非 fork"：不修改 K8s 核心语义，只维护"远少于 1000 行"的上游补丁（`README.md` "Is this a fork?" 一节），目标是 Edge/IoT/CI/ARM 等跑不起完整 K8s 集群运维的场景。它的"轻"来自两条路径（`README.md` "How is this lightweight"）：

1. **多组件合一进程**：apiserver、scheduler、controller-manager、kubelet、kube-proxy、内嵌 CCM 等不再是独立 systemd 服务，而是同一个 Go 进程内的 goroutine/子模块，省去每组件重复的运行时与连接开销。
2. **裁剪瘦身**：移除 in-tree 存储驱动与 in-tree cloud provider（两者有 CSI/CCM 出树替代且不影响 conformance），换得更小的二进制。

作为发行版它还捆绑并自动部署一整套周边：containerd+runc（容器运行时）、flannel（CNI）、CoreDNS、Traefik（ingress）、klipper-lb（内嵌 Service LB）、kube-router 网络策略控制器、helm-controller（CRD 驱动部署 Helm chart）、kine（datastore shim）、local-path-provisioner（本地存储）（清单见 `README.md` "K3s bundles..." 与 `scripts/airgap/image-list.txt`）。这些均可通过 `--disable` 关闭或换用（`pkg/cli/server/server.go` 的 Skips/Disables 逻辑）。

另一个关键设计是**节点连接面收窄**：worker 节点不对外暴露 kubelet API 端口，agent 通过 websocket 隧道把流量回传到控制面（`README.md` 第 5 条；实现 `pkg/agent/tunnel/tunnel.go`、`pkg/daemons/control/tunnel.go`）。

安装体验被压缩到一条命令：`curl -sfL https://get.k3s.io | sh -`（根目录 `install.sh`，写 systemd/openrc 服务并生成 kubeconfig）。

## 3. 技术栈与依赖

- **语言/运行时**：Go 1.26.10（`go.mod`），CGO 用于内嵌 SQLite（`scripts/build` 中 `SQLITE_TAGS` + 一组 `SQLITE_CGO_CFLAGS`，注释注明与 kine 保持同步）。
- **上游 Kubernetes**：`k8s.io/kubernetes => github.com/k3s-io/kubernetes v1.37.2-k3s1`——k3s 维护的 K8s fork，全部 staging 子模块也指向该 fork（`go.mod` replace 块）。这是"跟随上游 + 少量补丁"模式的载体。
- **k3s fork 生态**（go.mod replace）：etcd `v3.7.1-k3s3`（k3s-io/etcd）、containerd `v2.3.7-k3s1`（k3s-io/containerd）、kube-router `v2.11.1-k3s1`、cri-dockerd `v0.3.19-k3s5`、cadvisor、cri-tools 均有 k3s 后缀版本；runc 用上游 `v1.4.2`。
- **核心 shim/组件库**：kine `v0.17.5`（etcd API 兼容的数据store shim）、flannel `v0.28.10`（以库形式进程内运行）、helm-controller `v0.17.9`、k3s-io/api（K3s CRD 类型）。
- **其他**：`gorilla/websocket`（agent 隧道）、`minio/minio-go`（etcd 快照上传 S3）、`libp2p`（VPN/隧道辅助，`pkg/vpn`）、`urfave/cli/v2`（命令行）、`contour/termenv` 等常规件。
- **捆绑二进制**（非 Go 库，打包进产物）：containerd/shim/runc、CNI 插件（`v1.9.1-k3s1`）、k3s-root `v0.15.4`（iptables/socat 等 host 工具，`scripts/version.sh` 67 行起，带四架构 SHA256）。
- **构建链**：Docker buildx 多阶段（`Dockerfile`，golang:1.26.10-alpine + tonistiigi/xx 交叉编译），Makefile 以 dapper 风格在容器内出产物。

## 4. 目录结构与代码组织

tarball（HEAD，2026-11-21）共 903 个文件，无 vendor 目录（`gh api contents/vendor` 404），依赖全部走 Go module 下载，仓库本体很精炼。

| 目录 | 职责 |
|---|---|
| `cmd/` | 14 个薄入口（`cmd/k3s/` 为真正的多合一入口，含 linux/windows 分支；containerd/ctr/kubectl 等为打包符号链接用的入口） |
| `pkg/cli/` | urfave/cli 命令定义与 flag 组装（server/agent/cert/etcdsnapshot/secretsencrypt 等） |
| `pkg/daemons/` | 组件启动编排：`control/`（控制面四组件）、`agent/`（kubelet/kube-proxy）、`executor/`（Executor 接口） |
| `pkg/executor/embed/` | 默认 executor：把 K8s 各组件当库调用的进程内实现（`etcd/` 子包为内嵌 etcd 启动） |
| `pkg/cluster/` | 集群/datastore 编排：bootstrap、kine 启动、managed driver 注册、etcd proxy |
| `pkg/etcd/` | 内嵌 etcd 管理（成员控制器、快照、S3、sqlite→etcd 迁移） |
| `pkg/agent/` | agent 侧：containerd、CNI/flannel、netpol、tunnel、loadbalancer、proxy |
| `pkg/deploy/`、`manifests/` | 自动部署到集群的 YAML（coredns/traefik/local-storage/ccm/metrics-server） |
| `pkg/static/`、`pkg/data/` | go:embed 打包的捆绑二进制与 manifest，运行时解包 |
| `scripts/` | build/package-cli/package-airgap/version.sh 等全部构建产物化脚本 |
| `tests/` | integration（20+ 场景）、e2e（vagrant+ansible）、install、perf、docker、mock |
| `.github/workflows/` | 20 个 CI workflow（unitcoverage/integration/e2e/airgap/trivy/codeql/govulncheck/scorecard 等） |
| `updatecli/` | 依赖版本自动更新流水线（含 updatecli_k8s.yaml 跟上游 K8s） |

组织方式评述：单体 Go 仓库 + 薄 cmd + 厚 pkg（约 40 个子包），没有 monorepo 多模块；与上游的耦合被集中到 go.mod replace 和 `pkg/executor/embed` 一处，升级 K8s minor 主要改这两个点。

## 5. 核心架构与关键实现

### 5.1 单进程合并：Executor 抽象与 embed 实现

K8s 组件"合体"的关键接口是 `pkg/daemons/executor/executor.go`（32-51 行）：定义了 `APIServer/Scheduler/ControllerManager/CloudControllerManager/Kubelet/KubeProxy/ETCD/Containerd/CNI...` 一套启动函数。默认实现 `pkg/executor/embed/embed.go`（构建 tag `!no_embedded_executor`）直接 import 上游组件的 app 包：

```go
// pkg/executor/embed/embed.go
apiapp "k8s.io/kubernetes/cmd/kube-apiserver/app"
cmapp  "k8s.io/kubernetes/cmd/kube-controller-manager/app"
proxy  "k8s.io/kubernetes/cmd/kube-proxy/app"
sapp   "k8s.io/kubernetes/cmd/kube-scheduler/app"
kubelet "k8s.io/kubernetes/cmd/kubelet/app"
```

以 kubelet 为例（`pkg/executor/embed/embed.go` Kubelet 方法）：`kubelet.NewKubeletCommand(...)` 构造 cobra 命令后在 goroutine 里 `command.ExecuteContext(ctx)`，即上游 kubelet 代码作为库运行在同一进程。apiserver 还额外通过 `APIServerHandlers()` 把其 http.Handler 暴露出来，由 k3s 的 supervisor listener 复用（`pkg/daemons/control/server.go` 的 `waitForAPIServerHandlers`）。

控制面启动顺序在 `pkg/daemons/control/server.go` 的 `Server()`：先 `cfg.Cluster.Start`（datastore），再依次 apiserver → scheduler（等 CCM 摘除 node taint 后启动）→ controller-manager → cloud-controller-manager，全部可经 `--disable-*` 关闭。agent 侧 `pkg/daemons/agent/agent.go` 以同样方式调 `executor.KubeProxy`（62 行）与 `executor.Kubelet`（83 行）。

**例外**：containerd 不进程内运行，而是子进程——`pkg/agent/containerd/containerd.go:105` 用 `exec.CommandContext` 拉起容器内打包的 containerd 二进制；k3s 只做 CRI 等待与配置生成。这个分层（控制面进程内、运行时子进程）是内存与可靠性之间的折中。

### 5.2 数据存储：kine shim 之上的三种后端

apiserver 永远以 etcd3 协议连接存储（`pkg/daemons/control/server.go` `setupStorageBackend`：硬编码 `argsMap["storage-backend"] = "etcd3"`）。真正的后端由 `pkg/cluster/cluster.go` `startStorage` 决定：调用 kine 的 `endpoint.Listen(ctx, c.config.Datastore)`，在本地 unix socket 上提供一个 etcd 兼容 gRPC 端点：

- **默认**：`--datastore-endpoint` 为空时 kine 用内嵌 SQLite（单写者，因此 `etcdConfig.LeaderElect=false`，`NoLeaderElect` 由此推导，`pkg/cluster/cluster.go` ~213 行注释）；
- **外部数据库**：`--datastore-endpoint` 指向 MySQL/Postgres/外部 etcd 时 kine 翻译到对应驱动；
- **内嵌 etcd**：注册为 managed driver（`pkg/cluster/etcd.go` 的 `managed.RegisterDriver(etcd.NewETCD())`），由 `pkg/etcd/etcd.go` 负责成员管理、快照（本地 cron + S3，`pkg/etcd/s3/`、`pkg/etcd/snapshot/`）。还支持从 sqlite 数据迁移到 etcd（`pkg/etcd/etcd.go` ~899 行 "Migrating content from sqlite to etcd"）。

```go
// pkg/cluster/cluster.go startStorage（节选）
etcdConfig, err := endpoint.Listen(ctx, c.config.Datastore)
...
c.config.Datastore.Endpoint = strings.Join(etcdConfig.Endpoints, ",")
c.config.NoLeaderElect = !etcdConfig.LeaderElect
```

这套抽象让"默认单节点零依赖、可平滑升级到 HA"成为可能，是 K3s 与同类发行版最大的差异点。

### 5.3 打包与分发：一切塞进一个二进制

`scripts/package-cli` 是产物化的核心：先为 containerd/crictl/kubectl/k3s-agent 等生成指向 k3s 的符号链接，把 `bin/`+`etc/` 打成 tar 并 zstd 压缩（`--sort=name --mtime=@0` 保证可复现），拷入 `pkg/data/embed/`，最后 `go build` 时以 `go:embed` 打进 k3s 单文件（`pkg/data/data.go:9`、`pkg/static/stage.go:15`）。运行时再把捆绑二进制解包到数据目录（`pkg/static/stage.go` 的 `Stage()`）。`scripts/build` 编译所需的 containerd/runc/CNI 二进制并注入全部组件版本号 ldflags（`scripts/version.sh` 从 go.mod 反查各 fork 版本）。

离线（airgap）分发是显式产品线：`scripts/airgap/image-list.txt` 固定 8 个镜像（klipper-helm/klipper-lb/local-path-provisioner/coredns/busybox/traefik/metrics-server/pause），`scripts/package-airgap` 用 `ctr images export` 导出 `k3s-airgap-images-<arch>.tar.zst/.gz` 并生成 sha256（专门有 `.github/workflows/airgap.yaml` 验证）。CI 还有 `scripts/binary_size_check.sh` 守住二进制体积。

### 5.4 网络：进程内 flannel + websocket 隧道 + 双层 LB

- flannel 以 Go 库进程内运行：`pkg/agent/flannel/flannel.go` 直接 import flannel 的 backend（hostgw/vxlan/wireguard/ipsec/extension，36-40 行 blank import 注册），`flannel.Run` 在 agent goroutine 内执行，不再有独立 flanneld 进程；网络策略由 kube-router fork 的 netpol 控制器承担（`pkg/agent/netpol/`）。
- agent 与 server 间的 websocket 隧道（`pkg/agent/tunnel/tunnel.go`，`websocket.Dialer`）承载 kubelet API 等回传流量，服务端 `pkg/daemons/control/tunnel.go` 提供 tunnel handler——这是"worker 不暴露 kubelet 端口"的实现。
- 两层 LB 职责分离：agent 连接 apiserver 用内置的客户端轮询代理 `pkg/agent/loadbalancer`（代码自述"not a proper load-balancer"，仅维护 server 池 failover）；Service 的 LoadBalancer 类型则由以 DaemonSet 运行的 klipper-lb 镜像（`rancher/klipper-lb v0.4.18`）承接，二者不同代码体。

## 6. 构建与运行

```bash
# 容器内构建（dapper 风格，Makefile 默认目标 ci）
make build            # = make binary：docker buildx -f Dockerfile --target=result
make multiarch-binary # amd64/arm64/armv7/riscv64/windows
make image && make airgap   # 镜像与离线包

# 容器外直接 go build（scripts/build）
. ./scripts/version.sh && ./scripts/build
# 关键 build tags：ctrd netcgo osusergo providerless urfave_cli_no_docs
#   sqlite_unlock_notify ... sqlite_preupdate_hook（CGO SQLite）apparmor seccomp [selinux]

# 运行
sudo ./k3s server                 # kubeconfig 写入 /etc/rancher/k3s/k3s.yaml
sudo ./k3s agent --server URL --token TOKEN
sudo ./k3s kubectl get nodes

# 测试
./tests/integration/test-runner.sh <场景>   # tests/integration/ 下 20+ 场景
# e2e：tests/e2e/ vagrant + ansible（e2e_test_playbook.yaml）
```

以上均出自 `Makefile`、`scripts/build`、`scripts/package-cli`、`Dockerfile`、`tests/TESTING.md`。

## 7. 测试与工程质量

- **测试分层完整**：单元测试散布各包（如 `pkg/daemons/control/server_test.go`、`pkg/cluster/bootstrap_test.go`）；`tests/integration/` 覆盖 etcd 快照/恢复、证书轮换、双栈、secrets 加密、flannel 各模式、longhorn 等真实场景；`tests/e2e/` 用 vagrant 多节点验证 rootless、多租户、Tailscale VPN、wasm 等端到端行为；另有 install 脚本测试与 perf。
- **CI 密度大**：20 个 workflow（`.github/workflows/`），含每日 nightly-install、unit coverage 上报（codecov 配置 `.github/.codecov.yml`）、trivy 镜像扫描、codeql、govulncheck、OpenSSF Scorecard（README 徽章可证）。
- **质量护栏**：`.golangci.yml` lint、`scripts/validate` 构建前校验、DCO（`DCO` 文件 + dco.yml）、二进制体积检查、CLOMonitor 合规。
- 观察性细节：构建脚本对产物做 sha256 并强制可复现打包顺序（`scripts/package-cli`），release 产物有 `install.sh.sha256sum` 校验。

## 8. 社区与维护状态

- **活跃度（gh api 2026-11-21）**：最近 push 为分析前一日；近 30 条 main 提交分布在 2026-10-28 至 2026-11-19 约 3 周内，日均 1+ 提交；open issues 仅 122（对 34k star 仓库极低）。
- **贡献者规模**：~353 名贡献者（gh api contributors 分页推算，含匿名）。
- **发版节奏（gh api releases）**：2026-11-03 同日发布 v1.34.13/v1.35.10/v1.36.6/v1.37.2 四条 minor 的 `+k3s1` 补丁版，RC 在前一周（10-29/10-30）——与 `README.md` "patch 一周内、新 minor 30 天内" 的承诺一致；同时维护 4 条 minor 是当前事实。
- **维护者结构**：`MAINTAINERS` 列出 16 名核心维护者，邮箱几乎全为 @suse.com（Rancher 为 SUSE 品牌），即 SUSE Rancher 团队主导日常开发；`GOVERNANCE.md` 定义了 CNCF 化的治理（价值观、维护者晋升、CNCF Resources、TOC 决策），CNCF 身份由 `.clomonitor.yml`（注册于 cncf/k3s）与 README conformance 链接佐证。商业支持为 SUSE Rancher 的 K3s/Rancher 产品线（客观记录）。
- **路线**：`ROADMAP.md` 说明开发在 main（对应最新 K8s minor）进行，回移植到 `release-[N]`、`release-[N-1]`、`release-[N-2]`；`updatecli_k8s.yaml` 自动跟进上游版本。

## 9. 亮点与风险

**亮点**
1. 单二进制工程范式教科书级：Executor 接口（`pkg/daemons/executor/executor.go`）把"进程内运行上游组件"抽象干净，还留有 `no_embedded_executor` 构建标签可切换传统模式。
2. kine datastore 抽象：sqlite 默认零依赖、外部 DB 与内嵌 etcd 可选、sqlite→etcd 迁移路径明确（`pkg/cluster/cluster.go`、`pkg/etcd/etcd.go`）。
3. "fork + 少补丁"而非硬 fork：所有 K8s 代码经 k3s-io/kubernetes fork 注入，补丁量宣称 <1000 行，随上游节奏滚动（go.mod、README）。
4. 离线安装是一等公民：镜像清单、tar 导出、sha256、专属 CI workflow 全链路（`scripts/airgap/`、`scripts/package-airgap`）。
5. 测试/安全/治理齐备：20 个 CI workflow、OpenSSF Scorecard、CNCF conformance、DCO。

**风险/局限**
1. 与上游 K8s 版本强绑定：每个 minor 都要同步 rebase fork 与 replace 块，升级工程量大（`updatecli_k8s.yaml` 缓解但不能消除）。
2. CGO SQLite 默认后端为单写者，大规模 HA 必须切内嵌/外部 etcd，kine 转换层在超大规模下的性能上限未在仓库内给出数据（未确认）。
3. 控制面组件同进程共享故障域：apiserver goroutine panic 触发的 `signals.RequestShutdown` 路径会影响同进程其他组件（`pkg/executor/embed/embed.go` kubelet 的 recover/Fatal 逻辑可佐证耦合度）。
4. 维护者高度集中于单一厂商（SUSE），社区多元化依赖 CNCF 治理约束（`MAINTAINERS`）。

## 10. 总结评价

K3s 是"把 Kubernetes 做成软件包"的代表作：以发行版纪律跟随上游，用进程内合并与 datastore shim 把部署面压到一条命令、一个二进制，同时保住 CNCF conformance 与 airgap 交付能力。它没有发明新 K8s 功能，而是把运维复杂度折叠进构建与打包工程，这种"减法工程"在 Go 生态里有很高的源码研读价值。适合边缘/IoT/CI/开发环境与需要内嵌 K8s 能力的产品集成；想学习"如何不动上游代码地魔改 K8s"的工程师，`pkg/executor/embed`、`pkg/cluster`、`pkg/agent/tunnel` 三条线是最值得精读的路径。
