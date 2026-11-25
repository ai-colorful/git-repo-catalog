# ray-project/ray 项目分析

> 星数 40200（快照 2026-10-24）｜主语言 Python (C++)｜档位 20k-50k｜分析日期 2026-11-06

## 1. 项目速览

一句话定位：Ray 是一个统一的分布式 AI 与 Python 应用计算框架——用一个核心运行时（Tasks/Actors/Objects）加一套 AI 库（Data/Train/Tune/Serve/RLlib），让同一份 Python 代码从笔记本无缝扩到集群。

| 项目 | 事实 |
|---|---|
| 仓库 | ray-project/ray（默认分支 master） |
| 星数 | 40200（快照 2026-10-24）；分析时 gh api 显示 44213，仍在快速上涨 |
| 语言 | Python（含 Cython 桥）+ C++ 核心；另有 cpp/、java/ 多语言 API |
| 许可证 | Apache-2.0（`LICENSE`） |
| 版本 | master 开发版 `3.0.0.dev0`（`python/ray/_version.py`），最新发布 ray-2.60.0（2026-11-04，gh api） |
| 最近活跃 | 最近 push 2026-11-05（gh api），月内发布 5 个 release |
| 规模 | 全仓 10582 文件；`python/ray` 下 4148 个 .py 约 133 万行，`src/ray` 下 886 个 .cc/.h 约 25.5 万行 |

## 2. 项目是什么，解决什么问题

单机开发环境（笔记本）无法满足日益算力密集的 ML 工作负载，而传统分布式方案（Spark/MPI/K8s 原生）要么面向批处理数据、要么要求重写应用。Ray 的答案是：提供一套通用进程级分布式运行时，把"并行"藏进 `@ray.remote` 装饰器里，让 Python 函数（Task）和类（Actor）自动在集群中执行，返回值变成集群内可传递的不可变对象（ObjectRef）。这段描述与 `README.md` 中 "Ray is a unified way to scale Python and AI applications from a laptop to a cluster" 一致。

核心抽象只有三个，出自 `README.md`：Tasks（无状态函数）、Actors（有状态 worker 进程）、Objects（跨节点可访问的不可变值）。`ray.init()` 会在本机自动拉起整套集群（GCS Server、Raylet、Plasma 对象存储、Dashboard 等，见 `python/ray/_private/services.py` 中的 `RAYLET_EXECUTABLE`/`GCS_SERVER_EXECUTABLE` 常量与 `python/ray/_private/worker.py` 的 `init()`，其签名含 `num_cpus`/`num_gpus`/`object_store_memory`/`plasma_directory` 等参数）——单机即集群，同一条代码路径扩到多机。

在核心运行时之上，Ray 以库矩阵覆盖 ML 全流程：Data（ETL/流式数据集）、Train（分布式训练）、Tune（超参搜索）、Serve（模型服务）、RLlib（强化学习），以及整合层 AIR（`python/ray/air/`）。这些库全部构建在 Tasks/Actors/Objects 之上，而非各自的通信机制——这是 Ray 与"一堆独立工具"的根本区别。

Ray 的学术界背书也写进了 README：Ownership 论文（NSDI'21，分布式引用计数与 future 系统）和 Exoshuffle 论文（大规模数据 shuffle），分别对应核心运行时与 Ray Data 的设计依据。生态上 K8s 部署由姊妹项目 KubeRay 承担，本仓库内保留联动测试（`.buildkite/kuberay.rayci.yml`）与 autoscaler 的 kuberay node provider（`python/ray/autoscaler/kuberay/`）。

从版本脉络看，master 开发版本已切到 3.0.0.dev0（`python/ray/_version.py`），而发布线仍在 2.x（ray-2.60.0）——大版本切换正在进行中；这一代 Ray 的技术焦点从"分布式 Python 通用计算"明显转向"AI 基础设施"（Data/Serve/LLM），并投入 GPU 直接传输（RDT/NIXL）、Compiled DAG 等面向 LLM 推理场景的底层能力。

## 3. 技术栈与依赖

- **核心 pip 依赖极薄**：`python/setup.py` L414-423 只声明 8 项——click、filelock、jsonschema、msgpack、packaging、protobuf、pyyaml、requests。AI 库依赖通过 extras（`ray[data]`、`ray[tune]`、`ray[serve]`…）按需安装（`python/setup.py` 中 SetupType 分支）。
- **C++ 核心**：Bazel 构建（顶层 `BUILD.bazel`、`WORKSPACE`、`.bazelversion`），进程间通信为 gRPC + Protobuf（`src/ray/protobuf/`）与 FlatBuffers（`src/ray/flatbuffers/`，用于 Plasma 协议）。
- **Python/C++ 桥**：Cython，`python/ray/_raylet.pyx`（5543 行）绑定 C++ core worker 库，另有 `.pxd`/`.pyi` 类型桩。
- **对象存储**：自维护的 Plasma 共享内存存储（`src/ray/object_manager/plasma/`），基于 Arrow 序列化格式，支持 mmap 零拷贝与对象溢写（`python/ray/_private/external_storage.py`）。
- **通信协议分层明确**：控制面与 worker 间 RPC 用 Protobuf（`src/ray/protobuf/`），对延迟敏感的 Plasma 存储协议与节点内资源同步用 FlatBuffers（`src/ray/flatbuffers/`、`src/ray/object_manager/plasma/plasma.fbs`），序列化用户数据用 msgpack + cloudpickle（`python/ray/cloudpickle/` 为内嵌 fork）。
- **前端/可观测**：Dashboard 为 Python 常驻服务（`python/ray/dashboard/`），指标经 Prometheus/metrics agent（`python/ray/_private/metrics_agent.py`）。
- **代码质量工具链**：ruff + mypy + pyrefly（`pyproject.toml`、`pyrefly.toml`）、pylintrc、clang-format/clang-tidy、pre-commit、vale（文档 lint）、gofmt 类工具一应俱全（`.pre-commit-config.yaml` 12KB）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `python/ray/` | Python 包主体：核心 API + 全部 AI 库 + autoscaler/dashboard/runtime_env（4148 个 .py） |
| `python/ray/_private/` | 运行时私有实现：worker 主循环、进程管理、GCS 客户端、runtime_env agent、序列化 |
| `python/ray/_raylet.pyx` | Cython 桥，Python 侧与 C++ core worker 的粘合层 |
| `src/ray/` | C++ 核心运行时：gcs/、raylet/、core_worker/、object_manager/（含 plasma/）、rpc/、ray_syncer/ 等 25.5 万行 |
| `cpp/` | 面向用户的 C++ API（`cpp/include/`、示例与测试） |
| `java/` | Java API 与原生执行引擎绑定 |
| `doc/` | Sphinx 文档源（doc/source/cluster/ 下含 Kubernetes 章节与 KubeRay 引导） |
| `ci/`、`.buildkite/` | Buildkite CI 流水线定义（core/data/kuberay/doc 等分组 .rayci.yml） |
| `release/` | 发布前的长周期验证测试（autoscaling、air、jobs、aws/gcp/azure 凭据脚本） |
| `docker/`、`bazel/` | 镜像构建与 Bazel 辅助 |
| `thirdparty/` | 第三方补丁与 protobuf 依赖（`thirdparty/patches/`、`thirdparty/protobuf/`） |
| `.github/` | CODEOWNERS、issue/PR 模板、dependabot 与部分 workflows |
| `python/ray/_private/telemetry/` | 使用统计与 OpenTelemetry 指标（`open_telemetry_metric_recorder.py`、`metric_cardinality.py`） |

组织方式是典型的**多语言 monorepo**：核心运行时（C++）与上层数据/训练/服务库（Python）同仓演进，用一个 Bazel 工作区 + 一个 pip 包统一切分；`python/ray/tests/`、`src/ray/**/tests/` 就地就近放测试，`release/` 单独承载需要真实云环境的发布验证。AI 库之间相对独立（data/serve/train/tune 各自有 BUILD.bazel 与内部 `_internal/`），共享 `ray/dag/`（编译图，Compiled DAG）与 `ray/air/` 公共层。

## 5. 核心架构与关键实现

### 5.1 分层架构：GCS（全局控制面）— Raylet（节点管家）— CoreWorker（进程内库）

Ray 运行时三层职责在源码中清晰可辨：

- **GCS Server**（`src/ray/gcs/gcs_server.cc`，1386 行）是全局元数据与状态服务，`GcsServer::Start()` 中注册了一组 gRPC 服务：NodeInfo、NodeResourceInfo、JobInfo、ActorInfo、PlacementGroupInfo、ControlPlanePubSub、ObservabilityPubSub（L565-608），内部按 job/actor/placement group 拆分管理器（`gcs_job_manager.cc`、`gcs_actor/`、`gcs_placement_group_manager.cc`），并有 `gcs_health_check_manager.cc` 做节点存活检测。
- **Raylet**（`src/ray/raylet/node_manager.cc`，3839 行）是每节点守护进程：管理本地 worker 池（`worker_pool` 相关）、执行任务租约（lease）、资源上报与生命周期（`agent_manager.cc` 管 runtime env/dashboard agent）。节点级调度逻辑在 `src/ray/raylet/scheduling/`。
- **CoreWorker**（`src/ray/core_worker/core_worker.cc`，5265 行）嵌入每个用户进程，负责任务提交/执行、对象引用解析（`future_resolver.cc`）、actor 管理（`actor_management/`）与新生代特性 experimental mutable object（`experimental_mutable_object_manager.cc`）。

Python 侧入口是 `python/ray/_private/worker.py`（3968 行）：`worker.py` 的 `Worker` 类持有 `core_worker` 引用，`main_loop()`（L1057）是任务 worker 的事件循环；`python/ray/_private/workers/` 下有 default_worker 等入口。

一条任务的完整链路可从代码推断：用户进程的 CoreWorker 提交任务 → 按租约（lease）机制向 Raylet 申请执行槽（`src/ray/raylet/lease_dependency_manager.cc`、`local_lease_manager.cc`）→ Raylet 依赖就绪后从 worker 池取 worker 派发 → 任务执行中产生的对象经 Plasma 存储，引用计数由所有者（owner）维护——这正是 Ownership 论文的实现落点；节点资源变化经 `ray_syncer` 与 GCS 汇聚后反馈给 autoscaler（`python/ray/autoscaler/`）与 Dashboard 状态聚合（`python/ray/dashboard/state_aggregator.py`）。

### 5.2 两级调度与可插拔策略

Raylet 的调度器按"先本地、后集群"组织：`src/ray/raylet/scheduling/cluster_resource_scheduler.cc` 在构造时同时持有 `LocalResourceManager`（本节点资源账本）和 `ClusterResourceManager`（全集群资源视图），并用 `CompositeSchedulingPolicy` 组合多种策略（L92-100）：

```cpp
scheduling_policy_ =
    std::make_unique<raylet_scheduling_policy::CompositeSchedulingPolicy>(
        local_node_id_, *cluster_resource_manager_,
        [this](auto node_id) { return this->NodeAvailable(node_id); });
```

`src/ray/raylet/scheduling/policy/` 下策略族完整：hybrid（默认，优先本地再抢占）、spread（均匀散开）、random、node_affinity、node_label（K8s 风格标签调度），以及 placement group 专用的 BundleSpread/BundleStrictSpread/TopologyBundle（ gang 调度，支持 `.buildkite` 同款 STRICT_PACK 等）。集群资源视图靠 ray_syncer（`src/ray/ray_syncer/`）与 GCS 资源 pull 模型（`src/ray/gcs/gcs_resource_load_puller.cc`）保持同步。

### 5.3 对象存储：Plasma 共享内存与零拷贝

`src/ray/object_manager/plasma/` 是完整的共享内存对象存储实现：`object_store.h`/`object_lifecycle_manager.h`（Create/Seal/Abort/Delete 生命周期接口，头文件注释明确返回 PlasmaError::OutOfMemory 等错误码）、`dlmalloc.cc`+`malloc.cc` 负责内存分配、`shared_memory.cc` 管理 mmap 文件、`eviction_policy.cc` 做 LRU 驱逐、`create_request_queue.cc`/`get_request_queue.cc` 处理并发请求。对象读写走 FlatBuffers 协议（`plasma.fbs`）。因为对象在共享内存中，同节点 worker 间传输是纯指针共享（零拷贝）；跨节点由 `src/ray/object_manager/` 的 pull/push manager（`pull_manager.cc`、`push_manager.cc`）与 `object_directory`（所有权式目录 `ownership_object_directory.cc`）负责，写满则溢写到外部存储（`python/ray/_private/external_storage.py`）。

Python 端 `ray.put()` 落到 `python/ray/_private/worker.py` 的 `put_object()`（L806）：序列化上下文负责把值写入 plasma，注释明确 plasma 满时按指数退避重试 `DEFAULT_PUT_OBJECT_RETRIES` 次；新版还支持 `tensor_transport`（NIXL 等单侧传输，用于 RDT 直接传 GPU 张量，L806-845）。

### 5.4 AI 库矩阵的代码体量

实测各库规模（`find python/ray/<lib> -name '*.py' | wc -l` 及行数）：

| 库 | .py 文件数 | 行数 | 代表文件 |
|---|---|---|---|
| data | 741 | 296,685 | `python/ray/data/dataset.py`（9064 行） |
| rllib | 932 | 214,709 | `python/ray/rllib/algorithms/` |
| serve | 333 | 191,630 | `python/ray/serve/deployment.py`（685 行）、`_private/` 下 controller/proxy |
| train | 327 | 70,895 | `python/ray/train/base_trainer.py`、`data_parallel_trainer.py` |
| tune | 230 | 59,717 | `python/ray/tune/execution/`、`analysis/` |

另有 `python/ray/dag/`（Compiled DAG，把 actor 拓扑编译成通信图，`compiled_dag_node.py`）、`python/ray/llm/`（新增 LLM 服务辅助层）、`python/ray/workflow/`（持久化工作流，仅剩 `__init__.py`，已被标记弃用）。可见 Ray 的重心已明显向 Data/Serve/LLM 倾斜；同时 `python/ray/experimental/` 下承载 mutable objects、RDT 张量传输（`ray/experimental/rdt/`）等下一代特性，成熟后下沉进核心。

## 6. 构建与运行

- **用户安装**：`pip install ray`（README.md），或按库 `pip install "ray[data,serve]"`。
- **源码构建 wheel**：`python/setup.py` 的 `build()` 要求 Python 3.10+（`SUPPORTED_PYTHONS` 校验）并自动探测 bazel/bazelisk（`_find_bazel_bin()` L440）；顶层 `build-wheel.sh` 一键打 wheel，`build-docker.sh`/`build-image.sh` 构建镜像。
- **运行时环境隔离**：runtime_env 子系统独立成服务——`python/ray/_private/runtime_env/agent/`（`main.py`、`runtime_env_agent.py`）是每节点常驻 agent，支持 pip/conda/py_modules/java_jars/image_uri/nsight 等插件（同目录 `pip.py`、`conda.py`、`java_jars.py`、`nsight.py`），Raylet 经 `runtime_env_agent_client.cc` 与其交互。
- **C++ 单独构建**：Bazel 工作区（`.bazelrc` 15KB 配置 + `.bazelversion` 锁版本）。
- **运行**：`ray.init()` 单机自动拉起集群；`ray start --head` / `ray up`（autoscaler CLI，`doc/source/cluster/`）起多机；K8s 走 KubeRay。
- **测试**：pytest（`pytest.ini`：全局单测超时 180s、warning 即 error、asyncio 自动模式）；Bazel 测试目标遍布 `src/ray/**/BUILD.bazel`。

## 7. 测试与工程质量

- **规模**：Python 侧 1403 个 `test_*.py`，C++ 侧 192 个 `*_test.cc`（find 统计），另有 `release/` 下需要真实云环境的发布验证套件（autoscaling_tests、air_tests、cluster_tests 等）。
- **CI**：Buildkite 流水线按组件分文件（`.buildkite/core.rayci.yml`、`data.rayci.yml`、`doc.rayci.yml`、`kuberay.rayci.yml`、`dependencies.rayci.yml` 等），用 Wanda 镜像构建语法（`ci/docker/*.wanda.yaml`），支持多 Python 版本矩阵（manylinux x86_64、macOS `macos.rayci.yml`），还包含 K8s 混沌测试（kuberay.rayci.yml 中 "chaos {{matrix.workload}} under {{matrix.fault}}"）。
- **质量观察**：linter 链完整（ruff 基线规则锁定版本避免规则漂移，`pyproject.toml` L32 注释明确说明）；mypy 对生成 proto 模块做了定向豁免（`pyproject.toml` overrides）；pytest 把所有 warning 视为 error 逼出腐化。仓库还带 AI 协作配置（`AGENTS.md`、`.claude/`、`.cursor/`、`.gemini/`），2025-2026 年代特征的体现。C++ 侧测试与实现同目录就近放置（如 `src/ray/object_manager/plasma/tests/`），大量 `fake_*.h` 接口替身说明对可测性有明确投入。

## 8. 社区与维护状态

- **极度活跃**：最近 push 2026-11-05（gh api）；release 节奏约每两周一个 minor/patch——2026-10-02 的 2.59.0 到 2026-11-04 的 2.60.0 之间共发了 5 个版本（2.57.1、2.58.1、2.59.0、2.59.1、2.60.0，gh api releases）。
- **贡献者**：GitHub contributors API（仅统计 master 分支 commit）返回 510 位；全社区贡献者（含文档、非默认分支）应更多，具体数未确认。
- **社区治理**：3748 个 open issues、8249 forks（gh api）；README 提供 Discourse/Slack/Meetup 多渠道支持矩阵，自称 "Ray OSS Team" 2 天内响应 issue。
- **商业生态**：README 内置 Anyscale（Ray 商业公司）入口徽章；KubeRay 为独立仓库但 CI 联动（`.buildkite/kuberay.rayci.yml`），autoscaler 保留 kuberay provider 兼容。
- **文档体系**：docs.ray.io 由 Read the Docs 托管（README 徽章与 `.readthedocs.yaml` 7.7KB 配置），文档源即 `doc/source/`，与代码同仓同步演进——文档不落后于代码的机制化保障。

## 9. 亮点与风险

**亮点**
1. 核心依赖极薄（8 个 pip 依赖，`python/setup.py` L414），AI 库按需 extra 安装——对基础设施工具而言罕见的克制。
2. 分层清晰的运行时设计：GCS 控制面 / Raylet 节点面 / CoreWorker 进程面，三层全部有独立可测的 C++ 实现与 fake 实现（如 `src/ray/gcs/fake_gcs_kv_manager.h`），工程可维护性强。
3. 调度策略可插拔且代码解耦（`src/ray/raylet/scheduling/policy/` 每个策略一个 .cc/.h），gang 调度（placement group）与标签调度开箱即用。
4. Plasma 共享内存对象存储实现零拷贝 + 溢写兜底 + 驱逐策略，是分布式系统教学级实现（`src/ray/object_manager/plasma/`）。
5. 测试文化扎实：1600+ 测试文件、混沌测试、发布前长周期云上验证（`release/`）。

**风险/局限**
1. 体量巨大（python/ray 约 133 万行 + C++ 25.5 万行），新手读源码与排查问题成本高；Python/C++/Cython 三层交织，栈追踪跨越语言边界。
2. GCS 历史上是控制面单点，虽有 HA 与 leader 选举（`gcs_leader_gated_handlers.h`），小团队运维仍需谨慎评估。
3. 版本演进快（月度 release、master 已是 3.0.0.dev0），API 迁移与依赖升级成本需纳入评估；workflow 模块已收缩弃用（`python/ray/workflow/` 仅剩壳）。
4. 构建体系复杂（Bazel + Wanda 镜像 + 多平台矩阵），从源码贡献的上手门槛显著高于一般 Python 项目。

## 10. 总结评价

Ray 是过去十年分布式 Python 生态中最重要的开源项目之一：它用"进程级任务抽象 + 所有权式对象模型 + 共享内存对象存储"这套自洽的核心，统一了数据处理、训练、调参与服务四类 ML 工作负载，并凭借极薄的核心依赖和单机即集群的体验做到从原型到生产几乎零改写。源码层面，C++ 运行时（gcs/raylet/core_worker）分层干净、策略可插拔、测试密度高，是学习分布式调度与对象存储实现的优质教材；Python 侧体量庞大但模块边界清晰。适合两类人：需要把 Python/AI 应用扩到多机或上 K8s 的工程团队直接采用；想深入分布式系统实现的工程师精读 `src/ray/raylet/scheduling/`、`src/ray/object_manager/plasma/` 与 `python/ray/_private/worker.py` 三个入口。
