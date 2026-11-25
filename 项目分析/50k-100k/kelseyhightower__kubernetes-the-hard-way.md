# kelseyhightower/kubernetes-the-hard-way 项目分析

> 星数 50456（快照 2026-10-24）｜主语言 —（纯教程/Shell 文档仓）｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：Kubernetes 社区最著名的"手动从零搭建集群"教程——不带任何自动化脚本，一步步用 openssl、systemd 和官方二进制把一个集群拉起来，专为理解 K8s 内部机制而设计。

| 项目 | 内容 |
|---|---|
| 名称 | kelseyhightower/kubernetes-the-hard-way |
| 星数 / Forks | 50456（快照）/ 15951（gh api 实测 50594 / 15951，2026-11-04） |
| 语言 | 无代码语言（GitHub language 为空）；内容为 Markdown 教程 + Shell 命令 + 配置模板 |
| 许可证 | 代码/配置 Apache-2.0（LICENSE），教程文本 CC BY-NC-SA 4.0（README.md、COPYRIGHT.md）——双许可并存 |
| 最近活跃 | 最后推送 2025-04-10（gh api pushed_at；tarball 内文件 mtime 均为 2025-04-10） |
| 仓库规模 | 37 个文件、约 2059 行（docs/ 13 章 1590 行 + configs/ 113 行 + units/ 133 行 + ca.conf 206 行） |
| 教程锁定版本 | Kubernetes v1.32.3、etcd v3.6.0-rc.3、containerd v2.1.0-beta.0、runc v1.3.0-rc.1、CNI plugins v1.6.2、crictl v1.32.0（downloads-amd64.txt） |

## 2. 项目是什么，解决什么问题

Kubernetes The Hard Way 由 Google 前员工 Kelsey Hightower 于 2016 年 7 月创建（gh api created_at 2016-07-07），回答一个朴素的 问题：用 kubeadm、minikube 等工具一键装好的 Kubernetes，到底在机器上做了什么？本教程故意"绕远路"（README.md："optimized for learning, which means taking the long route"），让你亲手完成证书签发、kubeconfig 生成、etcd 启动、控制面组件注册、CNI 网络配置等每一个环节，做完一遍后对 K8s 各组件如何协作、如何互相认证会有具体而非抽象的认知。

教程目标拓扑为 4 台 Debian 12 (bookworm) 的 ARM64/AMD64 机器：1 台 jumpbox（管理机）、1 台 server（单节点控制面，含 etcd、kube-apiserver、kube-controller-manager、kube-scheduler）、2 台工作节点 node-0/node-1（containerd + kubelet + kube-proxy）（docs/01-prerequisites.md 机器规格表）。明确声明结果"不应被视为生产就绪"（README.md），定位纯粹是学习材料。

当前版本（2025-04 重写版）有一个重大变化：完全去掉了对 GCP 的依赖。早期版本（2016-2021）基于 Google Cloud 的网络路由与实例管理，而 docs/13-cleanup.md 明确写道"Previous versions of this guide made use of GCP resources ... The current version is agnostic"——现在只需任意 4 台同网段虚拟机/物理机，机器信息用一个手工维护的 `machines.txt` 文本数据库描述（docs/03-compute-resources.md），跨云、本地虚拟机均可。这使教程寿命远超任何一家云厂商的 API 生命周期。

教程采用"机器数据库 + 模板渲染"的组织方式：machines.txt 里的 IP/FQDN/主机名/Pod 子网通过 sed 替换注入配置模板（如 docs/09-bootstrapping-kubernetes-workers.md 用 `sed "s|SUBNET|$SUBNET|g"` 生成每个节点的 CNI bridge 配置与 kubelet 配置），用 while read 循环批量 SSH 分发，本身就是一份可复用的集群初始化方法论。

生态位上，它是 K8s 学习路线中的"原理层"经典：无数中文翻译版、二次创作教程（如各种"the hard way 跟做笔记"）都源于此；与它互补的是官方 kubeadm 文档（自动化）和 K8s 源码（实现层）。

## 3. 技术栈与依赖

本仓库自身没有任何构建系统或包管理清单（无 package.json/go.mod 等），"依赖"即教程要求的环境与工具：

- **操作系统**：Debian 12 (bookworm)，要求 root SSH 访问（docs/01-prerequisites.md、docs/03-compute-resources.md）
- **核心命令行工具**：wget、curl、vim、openssl、git（docs/02-jumpbox.md 安装清单）；工作节点额外装 socat、conntrack、ipset、kmod（docs/09）
- **证书体系**：纯 openssl，配套 206 行的 `ca.conf`（openssl 配置，含 admin/service-accounts/node-0/node-1/kube-proxy/kube-controller-manager/kube-scheduler/kube-api-server 每组件一个 section 及 SAN 扩展段）
- **进程管理**：systemd unit 文件 7 个（units/ 目录），无容器化封装
- **下载的二进制（downloads-amd64.txt / downloads-arm64.txt，amd64 与 arm64 逐行镜像对应）**：
  - Kubernetes v1.32.3 六件套（kube-apiserver/kube-controller-manager/kube-scheduler/kube-proxy/kubelet/kubectl，来自 dl.k8s.io）
  - etcd v3.6.0-rc.3、containerd v2.1.0-beta.0、runc v1.3.0-rc.1、CNI plugins v1.6.2、crictl v1.32.0
  - 注意：etcd 与 containerd/runc 锁的是当时尚未正式发布的 RC/beta 版（写作时点 2025-04 的最新预发布），时效性风险见第 9 章
- **网络方案**：CNI bridge 插件 + host-local IPAM（configs/10-bridge.conf），Pod 网段 10.200.0.0/16，每节点一个 /24；跨节点连通靠服务器上的静态路由（docs/11-pod-network-routes.md 用 `ip route add`）

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| README.md | 教程入口：定位声明、组件版本、13 个实验的索引链接 |
| docs/01-prerequisites.md ~ 13-cleanup.md | 教程正文 13 章，按顺序编号，每章结尾有 Next 链接串成线性路径 |
| ca.conf | openssl CA 及全部组件证书的配置模板（每组件一个 section，含 SAN） |
| configs/ | 各组件配置模板：CNI bridge/loopback、containerd config.toml、加密配置、kube-proxy/kubelet/kube-scheduler 配置、apiserver-to-kubelet RBAC |
| units/ | 7 个 systemd unit 文件：etcd、containerd、kube-apiserver、kube-controller-manager、kube-scheduler、kubelet、kube-proxy |
| downloads-amd64.txt / downloads-arm64.txt | 组件二进制的精确版本下载清单（每架构 11 个 URL） |
| .gitignore | 列出教程过程中生成的证书/kubeconfig 文件（admin-key.pem、worker-0.kubeconfig 等），说明仓库本身就是跳板机上的工作目录 |
| LICENSE / COPYRIGHT.md / CONTRIBUTING.md | Apache-2.0 许可、CC BY-NC-SA 4.0 版权声明、贡献指南 |

组织方式是典型的"文档型单层结构"：没有代码、没有构建、没有测试，13 章 Markdown 线性排列，配置文件按"模板 + 运行时替换"模式被各章节引用。仓库本身就是教程的工作目录（教程第 2 章要求把本仓库 clone 到 jumpbox 上使用），因此 .gitignore 屏蔽的全是教程产物而非开发产物。

## 5. 核心架构与关键实现

虽是内容仓，但教程的"实现"就藏在三个关键载体里：证书模板、systemd unit、配置模板。

**（1）PKI 体系：ca.conf + docs/04-certificate-authority.md。**
ca.conf 为每个组件预定义证书请求 section（grep 统计有 10 个 `^ [` section，包括 kube-api-server 的 SAN 段含 `server.kubernetes.local`、`10.32.0.1` 等备用名）。签发用一个 8 元素数组循环完成（docs/04）：

```bash
certs=(
  "admin" "node-0" "node-1"
  "kube-proxy" "kube-scheduler"
  "kube-controller-manager"
  "kube-api-server"
  "service-accounts"
)
for i in ${certs[*]}; do
  openssl req -new -key "${i}.key" -sha256 \
    -config "ca.conf" -section ${i} -out "${i}.csr"
  openssl x509 -req -days 3653 ... -CA "ca.crt" -CAkey "ca.key" ...
done
```

这是全教程最能体现"K8s 即 PKI"的部分：组件间互信完全建立在 8 张 4096 位 RSA 证书上，证书有效期 3653 天（约 10 年）。

**（2）控制面进程定义：units/kube-apiserver.service。**
34 行的 unit 文件就是一个可运行的 kube-apiserver 参数清单，暴露了学习重点：`--authorization-mode=Node,RBAC`、`--etcd-servers=http://127.0.0.1:2379`（etcd 走本机明文 HTTP，units/etcd.service 中 etcd 仅监听 127.0.0.1，是简化教学拓扑而非生产实践）、`--encryption-provider-config`（Secret 落盘加密）、`--service-account-issuer=https://server.kubernetes.local:6443`、审计日志参数等。每个参数都能在后续章节找到对应的验证步骤。

**（3）工作节点栈：configs/ + units/kubelet.service。**
kubelet-config.yaml 声明 x509/webhook 双重认证、Webhook 授权、systemd cgroupDriver、maxPods: 16；containerd-config.toml 配置 overlayfs 快照器与 SystemdCgroup=true（与 kubelet 的 cgroup 驱动一致，这是新手最常见的坑，教程直接给对）；10-bridge.conf 模板把每个节点的 Pod 子网（10.200.0.0/24、10.200.1/24）用 sed 注入（docs/09 开头）。

**（4）端到端验证设计：docs/12-smoke-test.md。**
冒烟测试覆盖 6 个能力点：Secret 落盘加密（用 etcdctl + hexdump 验证 `k8s:enc:aescbc:v1:key1` 前缀）、Deployment 创建、port-forward、Logs、Exec、Services（NodePort + iptables）。每一步都给出期望输出样例（如 nginx 响应头 `Server: nginx/1.27.4`，日期 Sun, 06 Apr 2025），读者可逐项 diff 自查——这种"命令 + 真实输出"的写作风格贯穿全部 13 章。

## 6. 构建与运行

没有构建。运行方式即"跟做教程"，核心命令脉络（均出自 docs/）：

```bash
# 0. 准备 4 台 Debian 12 机器（jumpbox/server/node-0/node-1，各 2GB 内存即可）
# 1. jumpbox 上克隆本仓库作为工作目录（docs/02-jumpbox.md）
git clone --depth 1 https://github.com/kelseyhightower/kubernetes-the-hard-way.git
cd kubernetes-the-hard-way

# 2. 按架构下载全部二进制（docs/02-jumpbox.md）
wget -q --show-progress --https-only --timestamping -P downloads \
  -i downloads-$(dpkg --print-architecture).txt

# 3. 依次跟做 docs/03 至 docs/12：机器准备 → CA/证书 → kubeconfig →
#    加密配置 → etcd → 控制面 → 工作节点 → kubectl → Pod 路由 → 冒烟测试
# 4. 验收（docs/12-smoke-test.md）
kubectl create deployment nginx --image=nginx:latest
kubectl get pods -l app=nginx   # 期望 Running
```

共 13 章实验，README.md 提供完整索引；每章末尾有 Next 链接。全程约需数小时，网络带宽需求 500MB+ 二进制下载（docs/02）。

## 7. 教程内容与组织方式（内容型特例）

- **章节体系**：13 章严格线性递进，按"基础设施 → PKI → 认证配置 → 数据面/控制面 → 网络路由 → 验证 → 清理"排列（docs/ 目录）。章节间无分支、无并行路径，是刻意的单一正路设计。
- **条目规模**：docs/ 正文 1590 行、约 200+ 个命令块；外加 ca.conf（206 行 openssl 配置）、9 个配置模板（113 行）、7 个 systemd unit（133 行）、2 份下载清单。全部内容 2059 行——以极小的文本量承载了一个完整集群的搭建知识。
- **写作风格**：每步"目的说明 → 命令 → 真实输出样例 → 原理解释"，关键处有安全/简化权衡的显式声明（如 docs/03 解释为何开 root SSH："Security is a tradeoff, and in this case, we are optimizing for convenience"）。
- **无自动化立场**：全仓库没有一个封装脚本（.sh 文件数量为 0），所有命令都要求读者手动逐条执行——这是与各类"一键脚本化 hard way 衍生品"的本质区别，也是作者刻意的内容设计。

## 8. 更新与维护机制（内容型特例）

- **维护模式**：单维护者模式。CONTRIBUTING.md 明确"Review and merge duties are managed by @kelseyhightower"，并对新增内容设有"边际价值论证"门槛（举了 PR #208、#282 两个例子），对纯格式性小改动劝退（"consider the tradeoff between maintainer time and community benefit"）。
- **版本演进史**：仓库 2016 年创建，历史上经历了至少两次大改版——早期 GCP 绑定版 → 2021 年前后多节点 etcd/TLS 版 → 2025-04 完全云无关重写版（单机 etcd 明文 HTTP、machines.txt 机器库、K8s v1.32.3）。docs/13-cleanup.md 与 units/etcd.service 的现状均为证据。
- **活跃度**：最后推送 2025-04-10（gh api）。本次快照与最后一次大版本重写间隔约一年半，属于"低频大改"节奏：教程更新以年度为单位整体重写而非滚动小修。无 release（gh api releases 返回 404），无版本号，更新即 master 分支内容。
- **无 CI / 无 .github**：tarball 中不存在 .github 目录，没有任何自动化检查——内容正确性靠作者亲自跑通全流程来保证（文档中的输出样例即跑通痕迹）。
- **版本时效性（如实记录）**：截至分析日（2026-11-04），教程锁定 Kubernetes v1.32.3（2025-03 构建）、etcd/containerd/runc 锁的还是 RC/beta 版（downloads-amd64.txt），已落后 K8s 当前的 1.3x/1.4x 系列约 4-5 个小版本；流程与原理仍适用，但逐条命令的输出细节可能已漂移。

## 9. 亮点与风险

**亮点**

1. 知识密度极高：2059 行文本覆盖 PKI、kubeconfig、etcd、控制面、容器运行时、CNI、路由、加密存储、RBAC 九大主题，无一字冗余。
2. 全程可验证：每个命令都附真实输出样例，读者可以逐字对照排错，这在一万星以上的教程仓库里几乎是唯一一份做到如此彻底的。
3. 云无关设计（2025 版）：machines.txt + 模板替换的做法让教程在任何环境可复现，摆脱了早期 GCP 版的云厂商锁定，显著延长了教程寿命。
4. 显式声明简化点：etcd 明文单节点、root SSH、自签 CA 等处都注明"教学取舍"，避免读者把学习拓扑误当生产方案。
5. 配置模板本身是资产：ca.conf 与 units/ 下的 unit 文件可直接作为生产集群搭建的参考底稿。

**风险/局限**

1. 版本时效：组件锁在 v1.32.3 与多个 RC/beta 版（downloads-amd64.txt），距最后一次更新已逾一年半；containerd v2.1.0-beta.0、etcd v3.6.0-rc.3 与正式版存在差异，照做可能遇到细节不符。
2. 无社区维护通道的实际效力：单维护者 + 高门槛 PR（CONTRIBUTING.md），issue 积压 57 个（gh api open_issues_count），纠错修复可能不及时。
3. 对新手仍有门槛：要求 Linux/网络/TLS 基础，教程不解释 systemd 单元语法、iptables 原理等前置知识。
4. 双许可表述易混淆：README/COPYRIGHT 写 CC BY-NC-SA 4.0（禁商用）而 LICENSE 文件是 Apache-2.0，引用内容时需注意区分教程文本与配置文件（README.md、COPYRIGHT.md、LICENSE）。

## 10. 总结评价（内容型特例）

这是"内容即产品"的典范仓库：作者以个人权威（前 Google 员工、CNCF 名人）背书，用罕见的"全手工 + 全验证输出"风格把 Kubernetes 的黑盒拆解成 2059 行可复现的命令序列，五万星的本质是它成了整个 K8s 生态公认的基础原理第一课。它能长青的秘诀有二：一是内容与任何云厂商、任何自动化工具解耦，二是坚持低频重写而非滚动修补，每版都亲自跑通并留下真实输出。适合谁：想搞懂 kube-apiserver/kubelet/etcd/CNI 如何咬合的运维与后端工程师、CKA 备考者、需要给集群排障但只知 kubeadm 的使用者。不适合谁：想快速搭可用集群的用户（请用 kubeadm/k3s）、想在生产复刻此拓扑的团队。读它的方式不是读代码，而是找 4 台虚拟机从头跟做一遍——这正是"the hard way"的本意。
