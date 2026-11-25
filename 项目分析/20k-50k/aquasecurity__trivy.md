# aquasecurity/trivy 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Go｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：Trivy 是容器/K8s/代码仓库/云环境的"全能型"开源安全扫描器，一套二进制同时覆盖漏洞（CVE）、IaC 错误配置、敏感信息（secrets）、SBOM 与许可证五大扫描能力。

| 项目 | 值 |
|---|---|
| 名称 | aquasecurity/trivy（Aqua Security 出品） |
| 星数 | 39,321（gh api 实测 2026-11-14） |
| 主语言 | Go（go.mod 声明 `go 1.27.0`，src/go.mod） |
| 许可证 | Apache-2.0 |
| 创建时间 | 2019-04-11（gh api） |
| 最近 push | 2026-11-13（gh api，极活跃） |
| 最新 release | v0.76.2（2026-11-05，gh api releases） |
| 规模 | 1,656 个 .go 文件；非测试约 13.8 万行、测试约 15.5 万行 |
| 贡献者 | 约 441 人（contributors 分页 Link 头推算） |

## 2. 项目是什么，解决什么问题

Trivy 解决的是"安全扫描工具碎片化"问题：过去扫漏洞用 Grype/Clair、扫 IaC 用 tfsec/checkov、查 secrets 用 trufflehog、生成 SBOM 用 syft，各工具输入输出互不兼容。Trivy 把这些能力收进单一 CLI，用一个命令 `trivy <target> [--scanners ...] <subject>` 完成（src/README.md）。目标是两大维度：**扫描目标**（Container Image / Filesystem / Git Repository / VM 镜像 / Kubernetes）与**扫描器**（OS 包与语言依赖 SBOM、CVE、IaC misconfig、secrets、许可证），任意组合。

项目由 Teppei Fukuda（knqyf263，作者曾在 Aqua Security）创建，2019 年以 aquasecurity 组织名义发布；今天 knqyf263/trivy 仅是同名镜像 fork（gh api 实测 1 star）。Trivy 已成为 Aqua 商业版 CNAPP 的开源底座，同时通过 trivy-action、trivy-operator、VS Code 插件等外围项目渗透进 GitHub Actions、K8s 准入控制等场景（src/README.md Ecosystem 部分）。

一个关键的产品决策是把**漏洞数据库与检测逻辑分离**：漏洞库（trivy-db）、Java 依赖库（trivy-java-db）、Rego 检查集（trivy-checks）都是独立仓库，以 OCI 制品分发，CLI 运行时拉取缓存。这使扫描器本体无需发版即可更新漏洞数据（src/pkg/db/db.go、src/pkg/javadb/client.go）。

Trivy 还支持 client/server 分离模式（`trivy server` + `trivy client`，src/pkg/commands/app.go L620/L668），适合企业集中缓存 DB、多节点共享；以及 Wasm 插件系统（`trivy module`）允许第三方在扫描管线中注入自定义逻辑（src/pkg/module/）。

## 3. 技术栈与依赖

- **语言**：Go 1.27（src/go.mod），单二进制发布，goreleaser 构建。
- **关键依赖**（src/go.mod，共 493 行依赖声明）：
  - `open-policy-agent/opa v1.21.2`：misconfig 扫描的 Rego 策略引擎。
  - `go.etcd.io/bbolt`：本地漏洞库存储（bolt KV）。
  - `containerd/containerd/v2`、`google/go-containerregistry`：镜像拉取与 OCI 操作。
  - `helm.sh/helm/v4`、`k8s.io/client-go`：K8s/Helm 解析与集群 API。
  - `tetratelabs/wazero`：纯 Go Wasm 运行时，跑插件模块（src/pkg/module/module.go）。
  - `go-git/go-git`：远程 git 仓库扫描。
  - `licenseclassifier/v2`：许可证文本识别；`CycloneDX/cyclonedx-go`：SBOM 输出。
  - `modernc.org/sqlite`：无 CGO 的 sqlite 驱动（RPM DB / Java DB 检索，src/cmd/trivy/main.go 顶部注释）。
  - 兄弟仓库：`aquasecurity/trivy-db`、`trivy-java-db`、`trivy-checks`、`trivy-kubernetes`（均 gh api 可查）。
- **构建工具**：Mage（src/magefiles/magefile.go）统一封装 lint/test/docs/protoc；goreleaser + release-please 发版。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| cmd/trivy/ | 入口 main.go（插件模式、优雅退出、错误分类） |
| pkg/commands/ | cobra CLI 定义与各 target 的 Runner（app.go 定义 image/fs/rootfs/repo/config/client/server/k8s/vm/sbom 等 15+ 命令） |
| pkg/fanal/ | 文件系统/镜像分析层：analyzer（21 类）、handler、artifact（image/daemon/oci/archive）、walker、applier |
| pkg/scan/ | 扫描编排服务层（local/service.go 组装各扫描器） |
| pkg/vulnerability/、pkg/detector/ | 漏洞结果填充 + OS 包检测器（支持 alma/alpine/amazon/azure/bottlerocket/chainguard/debian/oracle/photon/redhat/rocky/suse/ubuntu/wolfi 等 20 种发行版，src/pkg/detector/ospkg/） |
| pkg/misconf/ + pkg/iac/ | IaC 扫描：misconf 做路由，iac/ 是完整的策略扫描框架（rego/、scanners/、providers/、adapters/、severity/） |
| pkg/fanal/secret/ | secrets 检测（约 70 条内置规则，src/pkg/fanal/secret/builtin-rules.go） |
| pkg/db/、pkg/javadb/ | 漏洞库/Java 库的下载、校验与本地缓存 |
| pkg/sbom/、pkg/licensing/、pkg/vex/ | SBOM 解析生成、许可证扫描、VEX 情报（CSAF/OpenVEX/CycloneDX） |
| pkg/report/ | 输出格式：table/json/sarif/cyclonedx/spdx/github(asff)/模板 |
| pkg/module/ | Wasm 插件系统（OCI 分发 + wazero 沙箱） |
| pkg/iac/ | 内嵌 IaC 扫描框架（源自 defsec 血统），Rego 引擎封装在 pkg/iac/rego/ |
| internal/ | 仅测试辅助（dbtest/gittest/registrytest 等） |
| docs/ | mkdocs 文档站源码，194 个 .md 文件 |
| integration/、e2e/ | 集成与端到端测试 |
| helm/、ci/、contrib/ | Helm chart、CI 脚本、模板贡献（GitLab/HTML/ASFF 模板） |

组织方式评述：单体 Go 应用 + "pkg/ 为主、internal 仅测试"的经典布局。特化程度高——`pkg/iac` 实质上是把原 aquasecurity/defsec（tfsec 的引擎）整个搬入主线，形成"库中库"；`pkg/fanal` 同样是自成一体的分析子系统（旧仓库 fanal 已并入）。这种"大 pkg/ 多子系统"结构让各扫描器边界清晰，但目录深度达 5-6 层。

## 5. 核心架构与关键实现

### 5.1 扫描目标抽象与编排（pkg/commands/artifact/run.go）

CLI 层用 `TargetKind` 映射到具体扫描函数，然后统一走"扫描 → 过滤 → 报告"三段式：

```go
scans := map[TargetKind]func(context.Context, flag.Options) (types.Report, error){
    TargetContainerImage: r.ScanImage,
    TargetFilesystem:     r.ScanFilesystem,
    TargetRootfs:         r.ScanRootfs,
    TargetRepository:     r.ScanRepository,
    TargetSBOM:           r.ScanSBOM,
    TargetVM:             r.ScanVM,
}
// 1. Scan the artifact  2. Filter the results  3. Report
```

（src/pkg/commands/artifact/run.go L431-455）

每个 target 还区分 standalone（本地拉 DB）与 client/server（连远程扫描服务）两种服务形态，见 src/pkg/commands/artifact/scanner.go 的 `imageStandaloneScanService` / `imageRemoteScanService` 等成对函数（L38-243）。插件钩子 `extension.PreRun/PostRun` 在命令前后注入（run.go L399-408）。

### 5.2 fanal 分析层（pkg/fanal/）

fanal 负责把"目标"变成结构化的 `ArtifactDetail`（OS 包、语言依赖、配置文件、secrets、许可证等）。设计为三层流水线：

1. **artifact**：定义统一接口，镜像由 docker daemon/registry/OCI archive/tar 多源接入（src/pkg/fanal/image/），VM 镜像与 git repo 各有实现。
2. **analyzer**：约 83 种分析器类型（src/pkg/fanal/analyzer/const.go），按 OS 包（apk/dpkg/rpm）、语言生态（language/ 下 c、conda、dart、dotnet、elixir、golang、java、julia、nodejs、php、python、ruby、rust、swift 共 14 个目录，src/pkg/fanal/analyzer/language/）、配置、加密资产、许可证等分组，通过空导入注册机制组装（`_ "pkg/fanal/analyzer/all"`，src/pkg/scan/local/service.go L35）。
3. **handler + applier**：对分析出的文件做后处理（unpackaged 处理等，src/pkg/fanal/handler/），最终 `applier.ApplyLayers` 把分层 blob 合成 `ArtifactDetail` 交给扫描服务（src/pkg/scan/local/service.go L59-66）。

### 5.3 misconfig 扫描与 Rego/OPA 策略引擎（pkg/misconf/ + pkg/iac/rego/）

misconf 层先按文件类型路由到具体 IaC 扫描器（Terraform、CloudFormation、Dockerfile、Kubernetes、Helm、Ansible、Azure ARM、TerraformPlan JSON/snapshot、通用 JSON/YAML），路由表见 src/pkg/misconf/scanner.go 的 `enablediacTypes`（L36-49）。所有策略统一用 OPA 执行：

- 策略本体来自独立仓库 aquasecurity/trivy-checks（gh api 确认），以依赖形式内嵌，`DisableEmbeddedPolicies` 选项允许外部替换（scanner.go L63-64）。
- Rego 封装在 src/pkg/iac/rego/scanner.go：预编译 `*ast.Module`、自定义 storage.Store、容忍 10 条编译错误的 `DefaultAllowedRegoErrors`（L29-31），并按 cloud provider 过滤结果（`makeSupportedProviders`，L37-44）。

### 5.4 漏洞库分发与本地缓存（pkg/db/ + pkg/javadb/）

漏洞库以 OCI 镜像层格式分发（不是传统的 tar.gz 下载），默认双源：

```go
DefaultGHCRRepository = fmt.Sprintf("%s:%d", "ghcr.io/aquasecurity/trivy-db", db.SchemaVersion)
DefaultGCRRepository  = fmt.Sprintf("%s:%d", "mirror.gcr.io/aquasec/trivy-db", db.SchemaVersion)
dbMediaType = "application/vnd.aquasec.trivy.db.layer.v1.tar+gzip"
```

（src/pkg/db/db.go L24-33，tag 即 schema 版本号实现破坏性升级隔离）

本地缓存为 bbolt 单文件，Init 时以 `ReadOnly: true` 打开（L44-47）。trivy-java-db 同机制（ghcr.io/aquasecurity/trivy-java-db，src/pkg/javadb/client.go L31）。扫描结果由 `pkg/vulnerability.Client.FillInfo` 统一补齐 severity（多数据源优先级 + 自动仲裁）与 primary URL（src/pkg/vulnerability/vulnerability.go L64-176）。

### 5.5 Wasm 模块扩展（pkg/module/）

`trivy module install` 从 OCI registry 拉取 Wasm 模块（media type `application/vnd.module.wasm.content.layer.v1+wasm`，src/pkg/module/command.go L16），wazero 运行时沙箱执行（src/pkg/module/module.go L15）。这是 trivy 区别于多数竞品的扩展点：无需 fork 即可定制扫描行为。

## 6. 构建与运行

```bash
# 构建
go build ./cmd/trivy            # 标准构建；官方用 goreleaser（goreleaser.yml）
mage build:trivy                # mage 封装（src/magefiles/magefile.go）

# 测试
mage test:unit                  # CI 实际使用（.github/workflows/test.yaml L66）
mage test:integration           # 集成测试（需拉取 fixture 镜像，mage test:fixture:container-images）
mage test:k8s                   # K8s 集成测试
mage lint                       # golangci-lint

# 运行
trivy image python:3.4-alpine
trivy fs --scanners vuln,secret,misconfig myproject/
trivy k8s --report summary cluster
trivy server --listen 0.0.0.0:4954
```

Wasm 示例模块、ignore 策略等在 src/examples/ 下。文档站构建用 `mage docs:generate` + mkdocs（src/mkdocs.yml）。

## 7. 测试与工程质量

- **测试规模**：603 个 `_test.go` 文件，测试代码约 15.5 万行，超过业务代码（13.8 万行）——测试优先的工程文化（find 统计）。
- **分层**：单元测试（各包内，大量 golden file，`mage test:update-golden` 维护）；集成测试 13 个入口文件覆盖 image/fs/k8s/client-server/module/plugin/docker-engine（src/integration/）；e2e 顶层 smoke（src/e2e/）；internal/ 提供假 registry、假 git、假 DB 等测试基建。
- **CI**：20 个 workflow（src/.github/workflows/）——test.yaml 分 lint/docs 校验/unit/integration/k8s/module 多 job；canary.yaml 每次 main push 产出 canary 构建供用户试金丝雀版本（README 有说明）；另有 semantic-pr（PR 标题规范）、triage、stale-issues、spdx-cron 等自动化治理。
- **观察**：错误处理统一 `xerrors.Errorf` 包裹 + `types.ExitError/UserError` 分类退出码（src/cmd/trivy/main.go）；内部大量使用泛型工具库（pkg/x/、samber/lo）。Wasm fixture 与 proto schema 都有专门的生成任务（magefiles/）。

## 8. 社区与维护状态

- **活跃度**：最近 push 2026-11-13；近 6 周（2026-10-01 后）合并 87 个 PR（gh api search 实测）。
- **发版节奏**：release-please 自动化，约每月一个 minor——v0.75.0（2026-10-01）→ v0.76.0（2026-10-14）→ v0.76.1 → v0.76.2（2026-11-05），patch 随 bug 及时跟进（gh api releases，CHANGELOG.md 同步自动生成）。2026-03 至 2026-08 期间 v0.69→v0.74 节奏稳定。
- **贡献者**：约 441 人（contributors API 分页推算）；.github/CODEOWNERS 存在，Aqua 团队主导方向，社区贡献以发行版适配/新生态分析器为主（从 CHANGELOG feature 条目可佐证）。
- **生态**：官方外围含 trivy-action（GitHub Actions）、trivy-operator（K8s 安全运营）、trivy-vscode-extension，与 trivy-db/trivy-checks/trivy-kubernetes 等数据/策略仓库分工（README Ecosystem + gh api）。
- 文档：docs/ 194 个 md，mkdocs 双版本站（latest/dev workflow），文档质量在同类开源安全工具中属第一梯队。

## 9. 亮点与风险

**亮点**
1. 能力全且组合自由：五大扫描器 × 六类目标自由编排，单二进制零依赖安装，取代多工具拼盘（src/README.md、src/pkg/commands/app.go）。
2. 漏洞库 OCI 化分发 + schema 版本 tag + 双镜像源，DB 更新与 CLI 发版解耦，工程上很干净（src/pkg/db/db.go）。
3. 测试文化扎实：测试代码行数超过业务代码，集成测试覆盖 client/server、K8s、Wasm module 等难测路径（src/integration/、magefiles/）。
4. 可扩展性三通道：Wasm module（沙箱）、Rego 自定义策略、ignore/政策文件（pkg/policy/），满足企业定制。
5. 自动化治理成熟：release-please、canary 构建、semantic-pr、golden 更新 CI 强制校验。

**风险/局限**
1. 代码体量大（约 29 万行含测试）且 pkg/iac 是大而全的内嵌框架，外部贡献者理解成本高，目录深度 5-6 层。
2. 扫描准确性强依赖漏洞库与 trivy-checks 的更新节奏，离线/内网环境需自建 DB 镜像源（pkg/db/db.go 依赖 ghcr/GCR）。
3. Rego 策略引擎随 OPA 版本升级存在行为漂移风险（go.mod 中 OPA v1.21.2，pkg/iac/rego 大量底层 API 封装）。
4. 版本号停留在 0.x（v0.76.x），历史上有过 schema 破坏性变更，长期升级需关注 release note（CHANGELOG.md）。

## 10. 总结评价

Trivy 是开源容器/K8s 安全扫描的事实标准之一：以"一个二进制、一组正交的 target×scanner 抽象、数据与代码分离的 OCI 化漏洞库"三个设计支点，把原本需要 Grype+Syft+tfsec+trufflehog 组合的工作收敛到一个工具，同时用 Wasm/Rego 双扩展机制维持了企业级定制能力。代码规模接近 30 万行但模块边界清晰，测试覆盖和文档是同类项目标杆。适合安全工程师作为 CI/CD 与 K8s 集成的默认扫描器；对 Go 开发者而言，pkg/fanal 的分层分析流水线、pkg/db 的 OCI 制品分发模式、pkg/iac 的 Rego 引擎封装是三处最值得精读的实现。竞品对比上，Grype/Syft 只覆盖漏洞与 SBOM，Clair 架构偏重服务端，Snyk 是闭源 SaaS——Trivy 的全能开源形态是其 39k 星的核心原因（此对比为分析性结论，无源码出处）。
