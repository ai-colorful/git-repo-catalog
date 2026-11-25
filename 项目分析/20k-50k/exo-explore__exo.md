# exo-explore/exo 项目分析

> 星数 35000（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：把家里/工作室的日常设备（Mac 为主）自动组建成一个 AI 推理集群，让单机装不下的大模型（DeepSeek 671B、Kimi K2 等）跑在多台设备上。

| 项 | 值 |
|---|---|
| 名称 | exo-explore/exo |
| 星数 | 35000（快照 2026-10-24；gh api 查询时已约 48k，增长极快） |
| 语言 | Python（核心）+ Rust（组网层）+ Swift（macOS 应用）+ Svelte（仪表盘） |
| 许可证 | Apache-2.0（`LICENSE`，README 徽章一致） |
| 当前版本 | 0.3.70（`pyproject.toml`），要求 Python 3.13.* |
| 创建时间 | 2024-06-24（gh api） |
| 最近推送 | 2026-11-04（gh api，分析前一天仍在提交，非常活跃） |
| Fork / Open issues | 3592 / 461（gh api 2026-11-05） |

## 2. 项目是什么，解决什么问题

exo 把同一局域网（或雷电线缆直连）的多台设备组成一个"透明"的 AI 集群：设备自动发现彼此、自动协商模型放置，对外在每个节点暴露统一的 HTTP API 与管理仪表盘（`http://localhost:52415`，`README.md`）。用户用任意 OpenAI/Claude/Ollama 客户端即可调用跑在整机群上的模型，数据不出内网，契合"隐私 + 本地"的卖点。

解决的核心问题是**单设备显存/内存不够**：模型按层（流水线）或按权重矩阵（张量）切分到多台设备，协同完成 prefill 与 decode。README 展示了 4 × 512GB M3 Ultra Mac Studio 跑 DeepSeek v3.1 (8-bit)、Qwen3-235B、Kimi K2 Thinking 的实测基准（引用 Jeff Geerling 的评测），并宣称 Tensor Parallel 在 2 台设备约 1.8x、4 台约 3.2x 加速（`README.md`）。

项目当前 HEAD 已与早期版本（层切分 + 自研 net.stack）完全不同：现在是事件溯源（event-sourcing）架构的分布式系统——每台设备运行相同的节点进程（`src/exo/main.py`），通过选举产生 Master 协调者，Worker 承载推理引擎，全部集群状态以命令→事件→状态副本的方式经 Zenoh 发布订阅复制。组网与成员管理等最影响稳定性的部分下沉到 Rust（`rust/networking/`），Python 侧通过 `exo_rs` PyO3 绑定调用（`src/exo/routing/membership.py`）。

差异化亮点是对 **RDMA over Thunderbolt 5** 的"day-0"支持（macOS 26.2+，`README.md`"Enabling RDMA on macOS"）：设备间延迟降低约 99%，使"加设备反而更快"成为可能。拓扑感知的自动并行会实时读取设备内存、网络链路（Socket/RDMA）构建图，在环上选最优放置（`src/exo/shared/topology.py`、`src/exo/master/placement.py`）。

## 3. 技术栈与依赖

- **语言**：Python 3.13（uv 管理工作区，`pyproject.toml` [tool.uv.workspace] 含 `rust/exo_rs`、`bench`、`tools`）；Rust（tokio + zenoh + netwatcher + blake3，`rust/networking/Cargo.toml`）；Swift 72 个文件约 10.6k 行（`app/EXO/`）；Svelte 5 + SvelteKit + Tailwind 4 + d3（`dashboard/package.json`）。
- **组网**：Zenoh（含 storage-manager 插件做事件副本、liveliness token 做成员存活探测）——`rust/networking/Cargo.toml`、`src/exo/routing/topics.py`。
- **推理后端**：MLX（固定 git rev 引入，`pyproject.toml` [tool.uv.sources]）、mlx-lm、mlx-vlm、mflux（图像生成）；Linux 走 mlx-cpu / mlx-cuda12 / mlx-cuda13 extra。torch 仅作为音频/部分依赖引入（`pyproject.toml`）。
- **API 框架**：FastAPI + hypercorn + pydantic v2 + msgspec（`pyproject.toml`、`src/exo/api/main.py`）。
- **图/拓扑计算**：rustworkx（`src/exo/shared/topology.py` 用 `rx.PyDiGraph` 存节点与链路）。
- **硬件信息采集**：psutil + macmon（Apple Silicon GPU/功耗/温度，fork 固定 rev，`README.md`）、`src/exo/utils/info_gatherer/`（info_gatherer.py、macmon.py、net_profile.py、system_info.py）。
- **工程链**：uv、Nix flake（`flake.nix` + Cachix 二进制缓存）、basedpyright（strict、failOnWarnings）、ruff、pytest/pytest-asyncio、Playwright（仪表盘 e2e）、PyInstaller（`packaging/pyinstaller/exo.spec`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `src/exo/` | Python 主包，全部业务逻辑（约 93k 行 / 484 个 .py） |
| `src/exo/master/` | 选中 Master 后的协调者：placement 调度、命令→事件转换（`main.py`、`placement.py`、`placement_utils.py`） |
| `src/exo/worker/` | 推理执行：`engines/mlx/`（LLM/视觉引擎、sharding、模型族）、`engines/image/`（mflux）、`runner/`（supervisor、llm_inference）、`disaggregated/`（prefill/decode 分离协议） |
| `src/exo/routing/` | Zenoh 之上的 TopicRouter/EventRouter/membership（`router.py`、`event_router.py`、`membership.py`） |
| `src/exo/shared/` | 事件/命令/状态类型（`types/`）、选举 `election.py`、拓扑 `topology.py`、常量 |
| `src/exo/api/` | HTTP API 与四种协议适配器（`adapters/`：chat_completions、claude、ollama、responses） |
| `src/exo/download/` | HuggingFace 模型下载协调与本地存储（coordinator.py、store.py、transfer.py） |
| `src/exo/utils/` | channels、task group、info_gatherer（硬件/网络画像）等基础设施 |
| `rust/networking/` | Rust 组网层：discovery、membership、swarm（约 5.2k 行 .rs） |
| `rust/exo_rs/` | PyO3 绑定包（`exo_rs.pyi` 提供类型桩） |
| `app/EXO/` | macOS 菜单栏应用（Swift，自动更新、雷电网卡配置服务 io.exo.networksetup） |
| `dashboard/` | SvelteKit 管理台（routes：downloads、integrations、traces 等），构建后静态文件由 API 服务 |
| `bench/` | exo-bench 压测工具（`bench/exo_bench.py`） |
| `tests/` | 跨节点集成测试（1/2/4 节点、resilience、dashboard Playwright） |
| `nix/`、`packaging/`、`scripts/`、`docs/` | Nix 打包、DMG/PyInstaller、脚本与文档（`docs/api.md`） |

组织评述：单仓多语言 monorepo，Python uv workspace 把 Rust 绑定纳入同一依赖图；每个模块自带 `tests/` 子目录，类型与测试覆盖为"内聚式"。注意：任务背景提示的 `exo/inference/` 布局属于旧版（0.1/0.2 时代），当前 HEAD 已重构为 master/worker + Zenoh 架构，本文以真实 HEAD 文件为准。

## 5. 核心架构与关键实现

### 5.1 组网与成员管理：Zenoh + Rust + SWIM 风格状态机

设备发现完全自动：Rust 层用 Zenoh 的组播/点对点会话发现同网段节点，用 liveliness token 判断存活（`rust/networking/src/discovery.rs`、`src/exo/routing/membership.py` 模块注释）。成员状态机为 Alive/Suspect/Gone/Resuming，四个定时器都定义在 Rust 侧并导出给 Python（`src/exo/routing/membership.py:40-50`）：

```python
GRACE: Final = timedelta(seconds=exo_rs.GRACE_SECONDS)
"""How long a node is Suspect before it is Gone"""
WEDGE_TIMEOUT: Final = timedelta(seconds=exo_rs.WEDGE_TIMEOUT_SECONDS)
"""How long a node whose transport is up may go without a report before it is Suspect(Wedged)"""
SPLIT_HOLD: Final = timedelta(seconds=exo_rs.SPLIT_HOLD_SECONDS)
"""How long the side of a split without a majority holds before it takes over the lost nodes' work"""
```

还有 quorum 判定（Majority/Half/Minority，`src/exo/routing/membership.py:36-38`）与"事件驱动的 forget"：协调者移除节点时（NodeTimedOut 事件）各节点同步告知 Rust 层停止计数（`membership.py:224-233`）。集群隔离用 namespace（默认=版本号，`EXO_ZENOH_NAMESPACE`，`README.md`"Cluster name"）。

### 5.2 选举与事件溯源：单 Master 协调

每个节点运行 Election（`src/exo/shared/election.py`），投票按 `(seniority, commands_seen, node_id)` 排序，Master 每 5s 心跳、静默 30s 触发改选（`election.py:24-27` 常量）。亮点是**保守改选**：若多数候选者仍在追随现任 Master，则现任连任，避免"睡醒的旧 Master 回来"造成集群二次重置（`election.py` 的 `choose_master` docstring，`src/exo/main.py:78-82` 注释指出所有节点都参与选举）。Master（`src/exo/master/main.py:209`）跑三个任务循环：事件处理器、命令处理器、placement 计划器（`Master.run`，`main.py:255-267`）；命令做幂等去重（`_first_time`），文本生成命令会区分 prefill-only 与 decode 实例集合（`main.py:305-320` 附近）。集群状态 = 命令 → 事件流 → 各节点本地重放，EventRouter 处理全局/本地事件与乱序缓冲（`src/exo/routing/event_router.py`）。

### 5.3 拓扑感知放置与切分调度

放置算法（`src/exo/master/placement.py` + `placement_utils.py`）：先在拓扑有向图上找环（模型 ring 放置依赖网络成环），再按内存过滤（`filter_cycles_by_memory`，`placement_utils.py:21-37`）、取最小环（`get_smallest_cycles`），层分配用最大余数法按各节点可用内存比例切层（`allocate_layers_proportionally`，`placement_utils.py:45-60`）。两种实例形态（`placement.py:55-58`）：

```python
INSTANCE_META_BACKENDS: dict[InstanceMeta, list[Backend]] = {
    InstanceMeta.MlxRing: [Backend.MlxMetal, Backend.MlxCuda, Backend.MlxCpu],
    InstanceMeta.MlxJaccl: [Backend.MlxMetal],
}
```

MlxRing 是环形流水线（TCP/RDMA 均可），MlxJaccl 是基于 Apple JACCL 集合通信的张量并行（仅 Metal/RDMA），后者需要计算各节点 coordinator 矩阵（`get_mlx_jaccl_coordinators`、`get_mlx_jaccl_devices_matrix`，`placement.py:7-15`）。流水线切分实现在 `src/exo/worker/engines/mlx/sharding/pipeline.py`（send/recv 激活、`flush_prefill_sends` 批量异步发送）；张量并行通过 monkey-patch 模型 `__call__` 注入分布式同步依赖（`sharding/tensor.py:8-38` 的 `patch_tensor_model`），并提供 `all_to_sharded_linear` 等抽象由各模型族实现。设备画像（内存、雷电路径、RDMA 状态、时延带宽）由 `src/exo/utils/info_gatherer/` 持续采集供放置决策。

### 5.4 推理引擎、模型族与 P/D 分离

MLX 引擎（`src/exo/worker/engines/mlx/`）内含按模型族组织的实现：`models/` 下有 deepseek_v3/v4/v41、kimi、qwen3、llama、gemma、glm、gpt_oss、minimax、nemotron_h、step 等 16+ 族（`models/registry.py`、`spec_registry.py` 提供放置规则）。进阶特性是 **prefill/decode 分离**：`src/exo/worker/disaggregated/protocol.py` 用 msgspec 定义二进制 KV-Cache 分块传输协议（KVChunk 含 layer_idx/num_tokens/keys/values，压缩由 zstandard 支撑），Master 侧维护 prefill→decode 实例链接（`master/main.py` 的 instance_links）。图像生成走独立引擎 `worker/engines/image/`（mflux，需 `EXO_ENABLE_IMAGE_MODELS=true`）。API 层 2734 行（`src/exo/api/main.py`）实现四协议：`/v1/chat/completions`、`/v1/messages`（Claude）、`/v1/responses`、`/ollama/api/chat`，适配器在 `src/exo/api/adapters/`，外加 `/instance/previews` 放置预览与 `/models/add` 自定义模型（`README.md`"Using the API"）。

## 6. 构建与运行

来自 `README.md` 与 `AGENTS.md`（均已在仓库内核对）：

```bash
# 从源码运行（macOS/Linux；Linux 目前仅 CPU）
cd dashboard && npm install && npm run build   # 先构建仪表盘
uv sync --extra mlx          # macOS；Linux: --extra mlx-cpu / mlx-cuda12 / mlx-cuda13
uv run exo                    # 启动 API + 仪表盘 http://localhost:52415
uv run exo --no-worker        # 只做协调不跑推理（弱节点）
uv run exo --namespace my-cluster

# 测试 / 质量门禁
uv run pytest                 # 默认排除 slow 标记
uv run pytest -m ""           # 全量
uv run basedpyright           # strict 类型检查，0 错误才允许提交（AGENTS.md 明确要求）
uv run ruff check

# 基准
uv run bench/exo_bench.py --model Llama-3.2-1B-Instruct-4bit --pp 128,256 --tg 128

# Nix 一键运行 / macOS 应用
nix run .#exo
brew install --cask exo
```

RDMA 需 macOS 26.2+，在恢复模式执行 `rdma_ctl enable`（`README.md`）。

## 7. 测试与工程质量

- **规模**：模块内单测 197 个 test_*.py、约 4 万行测试代码（`src/exo/*/tests/` 统计）；跨节点集成测试独立在 `tests/`：`test_1node.py`、`test_2node.py`、`test_4node.py`、`test_resilience.py`（故障注入/恢复）与 `test_dashboard.py`（Playwright），并有自研测试框架 `tests/framework.py`。旧测试归档于 `tmp/old_tests/`。
- **CI**：`.github/workflows/pipeline.yml` 在 macOS/Linux 矩阵上构建全部 Nix outputs、`nix flake check`、macOS 跑 pytest；`.github/workflows/build-app.yml` 在 tag 时用 Xcode 26.2 打 DMG 并发 release。
- **代码质量**：basedpyright strict 且 failOnWarnings，`pyproject.toml:127-152` 将 reportAny/UnknownType 等全部设为 error——对 9 万行动态语言代码是极高的门禁；ruff 启用 I/N/B/A/PIE/SIM。`AGENTS.md`、`.clauderules`、`.cursorrules` 表明项目深度使用 AI 结对开发并为其写了严格守则。类型系统上用 pydantic FrozenModel + 大量 Literal/Protocol（如 `membership.py` 的 Protocol 接口），风格相当函数式。
- 观察：注释质量异常高（解释设计动机而非复述代码），如 `choose_master` 对改选策略的论述；`router.py` 顶部坦承 TopicRouter 当前无法防回环的局限——工程透明度好。

## 8. 社区与维护状态

- **活跃度**：gh api 显示最近推送 2026-11-04（分析前一天），461 个 open issues、3592 forks；README 大量引用 2025-2026 年的实测与推文（RDMA day-0 支持时间为 2026 年，`README.md:20`），处于高强度迭代期。具体 commit 频率与贡献者名单 tarball 无 git 历史，标注"未确认"（可由 gh api contributors 进一步核实）。
- **发布**：版本双轨——macOS 应用用 release 号（如 1.0.71）+ Homebrew cask + 自动更新（`README.md`"Updating"），源码版本 0.3.70；两者默认不互通（集群名=版本号），文档明确指导混跑需统一 namespace。
- **生态位**：官方 Discord 社区、bench 提供公开基准、与 MLX 生态（ml-explore/mlx 固定 rev）深度绑定；被 Jeff Geerling 等硬件博主实测报道（README 引用）。

## 9. 亮点与风险

**亮点**
1. 端到端自动化程度极高：零配置发现、自动选举、拓扑感知自动放置，用户只需"装上、连网"（`src/exo/main.py`、`rust/networking/`）。
2. 工程质量在同类开源项目中罕见：strict 类型门禁、40k 行测试、Rust 承载最易崩的组网逻辑、事件溯源使全集群状态可审计可恢复（`pyproject.toml`、`src/exo/routing/`）。
3. 对 Apple 生态的深度调优是差异化壁垒：TB5 RDMA day-0、JACCL 张量并行、macmon 硬件画像、Swift 菜单栏应用（`app/EXO/`、`src/exo/master/placement.py`）。
4. 四协议 API 兼容层 + P/D 分离 + 图像模型，实用面广（`src/exo/api/adapters/`、`src/exo/worker/disaggregated/`）。

**风险/局限**
1. 平台偏向严重：完整能力（GPU 推理、RDMA、Tensor Parallel）仅限 Apple Silicon；Linux 仅 CPU（`README.md`"Hardware Accelerator Support"），NVIDIA 用户价值有限。
2. 强版本耦合：集群所有设备必须同版本/同 namespace 才能组网（`README.md`"Adding another Mac"），升级需全集群同步，运维摩擦大。
3. 架构仍在剧烈重构期（旧版 inference 布局已整体废弃，旧测试进 `tmp/`），文档 API 面尚未完全稳定，跟进成本高；open issues 461 也侧面反映边跑边修。
4. MLX/torch 依赖用 git rev 固定 + 多 CUDA extra 互斥（`pyproject.toml` [tool.uv] conflicts），依赖解析对新人不友好，且 Python 锁死 3.13。

## 10. 总结评价

exo 已经从早期的"设备组网跑层切分"玩具，进化成一个工程质量对标分布式数据库水准的个人 AI 集群操作系统：Zenoh/Rust 组网、选举协调、事件溯源、拓扑感知放置、RDMA 张量并行一应俱全，是"在家用 Mac 跑 671B"这一细分场景里目前完成度最高的开源方案。它适合想在不买服务器的前提下聚合多台 Mac 算力跑大模型的个人/小团队，也非常适合研究分布式推理调度的开发者精读——`src/exo/master/placement.py`、`src/exo/shared/election.py`、`src/exo/routing/membership.py` 与 `rust/networking/src/membership.rs` 是理解其设计精髓的最佳入口；但若你只关心 Linux/Windows 上的 GPU 集群，或需要一个稳定版本做生产集成，当前时点仍需观望。
