# langchain-ai/langgraph 项目分析

> 星数 23000（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-06

## 1. 项目速览

一句话定位：LangChain 官方的低层 agent 编排框架（"Low-level orchestration framework for building stateful agents"，`README.md`），把有状态、可长时运行、可中断恢复的 agent 建成图（状态机）执行。

| 项目 | 事实 |
| --- | --- |
| 名称 | langchain-ai/langgraph |
| 星数 | 23000（快照）；forks 7661、open issues 461（gh api，2026-11-06） |
| 主语言 | Python（>=3.10，见 `libs/langgraph/pyproject.toml`） |
| 许可证 | MIT（`LICENSE`、`libs/*/LICENSE`、README 徽章） |
| 最近活跃 | 最近 push 2026-11-05（gh api），高度活跃 |
| 当前版本 | langgraph 1.2.20（`libs/langgraph/pyproject.toml`） |
| 贡献者 | 约 290+（contributors API 翻页至 294 页/页 1 条，含匿名，精确值未确认） |

## 2. 项目是什么，解决什么问题

LangGraph 解决的是"LLM agent 跑得久、会中断、要恢复"的工程问题。传统链式调用（LangChain 的 LCEL）一旦进程失败或需要人工介入，执行状态就丢了。LangGraph 把应用建模为**共享状态图**：节点读取共享 state、返回部分更新，边/条件决定下一步；框架负责把每一步的执行结果持久化成 checkpoint，从而支持失败后自动续跑、人工审批后恢复、回放到历史任意时刻（README "Why use LangGraph" 一节：durable execution、human-in-the-loop、memory、LangSmith 调试、生产部署五项能力）。

底层执行模型借鉴 Google **Pregel / Bulk Synchronous Parallel** 与 Apache Beam，公开接口借鉴 NetworkX（`README.md` Acknowledgements；`libs/langgraph/langgraph/pregel/main.py:456` 类文档完整阐述了 Plan/Execution/Update 三相超步语义）。

项目定位正在**主动收缩为运行时**：`AGENTS.md` 明确写道 "LangGraph is a runtime……moving away from being a friendly SDK for building agents directly"，agent 开发应上移到一层 harness（Deep Agents，独立仓库 langchain-ai/deepagents）；`prebuilt` 等 agent 级包只做维护不再加功能。这是理解其演进方向的关键文件。

与母项目的关系：LangGraph 由 LangChain Inc 开发（README），依赖 `langchain-core`（`libs/langgraph/pyproject.toml`）但可脱离 LangChain 单独使用；JS/TS 实现在独立仓库 langchain-ai/langgraphjs（`libs/sdk-js/README.md` 注明本包已迁移），Python/JS 行为要求同步变更（`AGENTS.md`）。

商业化闭环清晰：开源核心（MIT）+ LangSmith（观测/评测）+ LangSmith Deployment（原 LangGraph Platform，托管部署）+ LangSmith Studio（可视化原型），README 与 `libs/cli/README.md` 反复指向 LangSmith；`libs/cli` 提供本地 `dev/build/up` 与 Docker 化，是自托管与云平台的共用入口。README 还点名 Klarna、Replit、Elastic 等为其生产用户。

## 3. 技术栈与依赖

- 语言：Python >=3.10（支持 CPython/PyPy，分类器见 `libs/langgraph/pyproject.toml`）；另含少量 JS 示例（`libs/cli/js-examples/`）
- 通道抽象是状态聚合的基石（`libs/langgraph/langgraph/channels/`）：`last_value`（默认，存最后写入）、`topic`（PubSub 多值）、`binop`（BinaryOperatorAggregate，reducer 聚合）、`any_value`、`ephemeral_value`（step 内可见、不持久）、`delta`（增量通道，配合 checkpoint 增量存储）、`named_barrier_value`（命名屏障，用于 join）、`untracked_value`（不参与版本追踪）
- 核心运行时依赖（`libs/langgraph/pyproject.toml`）：`langchain-core` >=1.4.7（回调/Runnable 协议）、`langgraph-checkpoint` >=4.3.0、`langgraph-sdk`、`langgraph-prebuilt`、`pydantic` >=2.7.4（state schema 校验）、`xxhash`（channel 版本号）
- 持久化实现：psycopg（postgres）、aiosqlite/sqlite3（见 `libs/checkpoint-postgres`、`libs/checkpoint-sqlite` 的依赖）
- 工程设施：hatchling 打包、uv 管理 monorepo（根 `Makefile` 逐包 `uv pip install -e` / `uv lock`）、pytest 系测试、GitHub Actions CI、trusted publishing 发布（`.github/workflows/release.yml`）

## 4. 目录结构与代码组织

uv monorepo，每个库独立 `pyproject.toml` + 独立版本 + 独立发布（release.yml 按包分发）。依赖方向（`AGENTS.md` 依赖图）：checkpoint → {postgres, sqlite, prebuilt, langgraph}，sdk-py → {langgraph, cli}，sdk-js 独立。

| 目录 | 职责 | 版本 |
| --- | --- | --- |
| `libs/langgraph/` | 核心框架：StateGraph、Pregel 引擎、channels、stream、Functional API | 1.2.20 |
| `libs/checkpoint/` | checkpointer 基类、serde、InMemorySaver、BaseStore 长期记忆、BaseCache | 4.3.1 |
| `libs/checkpoint-postgres/` | Postgres 持久化实现 | 3.2.0 |
| `libs/checkpoint-sqlite/` | SQLite 持久化实现 | 3.1.3 |
| `libs/checkpoint-conformance/` | checkpointer 合规测试套件 | 0.0.2 |
| `libs/prebuilt/` | create_react_agent、ToolNode（仅维护模式，AGENTS.md） | 1.1.1 |
| `libs/sdk-py/` | LangGraph Server REST API 的 Python SDK（附 `CHANGELOG.md`、`MIGRATION.md`、integration 测试） | 动态版本 |
| `libs/cli/` | langgraph CLI：本地开发/构建/Docker 运行 | 动态版本 |
| `libs/sdk-js/` | 仅存 README 占位，JS SDK 已迁至 langgraphjs 仓库 | — |
| `examples/` | 20+ 教程 notebook（multi_agent、human_in_the_loop、llm-compiler、plan-and-execute、reflexion 等） | — |
| `docs/` | 仅重定向脚本与 llms.txt，正文文档在 docs.langchain.com | — |
| `.github/workflows/` | 17 个 workflow：ci/test matrix/integration/bench/release 等 | — |

核心库内部（`libs/langgraph/langgraph/`，79 个非测试 py 文件）：

| 子目录 | 职责 |
| --- | --- |
| `graph/` | StateGraph 构建器、节点/分支（`state.py`、`_node.py`、`_branch.py`） |
| `pregel/` | 执行引擎，21 个模块（`main.py`、`_loop.py`、`_algo.py`、`_runner.py` 等） |
| `channels/` | 8 种通道实现（见 5.1） |
| `func/` | Functional API（`task`/`entrypoint`） |
| `stream/` | 流式输出（`run_stream.py`、`mux.py`、`transformers.py`） |
| `types.py` | Command/Interrupt/Send/StateSnapshot/RetryPolicy 等公共类型（1108 行） |
| `managed/`、`_internal/`、`cache` 相关 | 托管值、私有工具、节点缓存协议 |

## 5. 核心架构与关键实现

### 5.1 StateGraph：状态机构建器（`libs/langgraph/langgraph/graph/state.py`，1978 行）

`StateGraph` 是 builder 模式：节点签名为 `State -> Partial<State>`，state 字段用 `Annotated` 标注 reducer 函数聚合多节点的并发写入（`state.py:131` 类文档）。`add_node` 有 5 个重载（`state.py:376-667`），`add_edge`/`add_conditional_edges`/`set_entry_point` 构图，`compile()`（`state.py:1177`）把图编译成 `CompiledStateGraph`（`state.py:1404`，继承 Pregel），把每个节点包装为 `PregelNode`（`state.py:1509`）。reducer 语义由 `_get_channels`/`_is_field_binop`（`state.py:1815-1925`）从类型注解反射生成 channels——这是"注解即编排"的核心：`Annotated[list, reducer]` 字段自动对应 `BinaryOperatorAggregate` 通道。

### 5.2 Pregel 执行引擎：超步迭代（`libs/langgraph/langgraph/pregel/main.py`，4557 行 + `_loop.py` 2202 行 + `_algo.py` 1465 行）

执行遵循 BSP 三相循环（`main.py:456` 类文档原文）：**Plan** 选出订阅了上一步被更新通道的 actors（任务）；**Execution** 并行执行所有选中 actor，期间通道更新对彼此不可见（隔离写）；**Update** 批量落盘写入。单次超步由 `PregelLoop.tick()`（`_loop.py:652`）驱动：检查步数上限（`out_of_steps`）、准备任务并发出 checkpoint、处理 interrupt_before 断点、返回是否继续；`after_tick()` 汇总所有任务 writes、处理 DeltaChannel 快照与恢复重放。任务生成在 `_algo.py` 的 `prepare_next_tasks`（`_algo.py:349`）与 `apply_writes`（`_algo.py:234`，按通道版本号增量更新）；`Send`（`types.py:748`）实现 map-reduce 式动态派发（`prepare_push_task_send`，`_algo.py:938`）。引擎还内置重试（`_retry.py`、`RetryPolicy` in `types.py:428`）、节点级缓存（`langgraph/cache/base`）、超时（`TimeoutPolicy`）。

### 5.3 Checkpoint 持久化与 time-travel（`libs/checkpoint/langgraph/checkpoint/base/__init__.py`，889 行）

`BaseCheckpointSaver`（`base/__init__.py:199`）定义抽象接口：`get_tuple`/`list`（读历史）、`put`/`put_writes`（写检查点/待定写）、`delete_thread`、`copy_thread`、`prune`。`Checkpoint` 结构（`base/__init__.py:93`）记录 channel_versions（通道版本表）+ channel_values，使引擎能判断"哪些通道变了→哪些任务要跑"。序列化层 `checkpoint/serde/`（jsonplus、msgpack、encrypted 加密选项）。实现方有 Memory/Postgres/SQLite 三个 saver，另有 `langgraph/store/base`（跨 thread 的长期记忆 Store）与 `langgraph/checkpoint/redis` 缓存。time-travel 能力由 checkpoint 历史链支撑：`list` 返回历史快照，`StateSnapshot`（`types.py:721`）暴露 next/ interruptions 元数据，`_internal/_replay.py` 处理按 checkpoint_id 重放。

### 5.4 Command / interrupt：人机协同与恢复（`libs/langgraph/langgraph/types.py:843-1000`）

`interrupt(value)` 在节点内抛出 `GraphInterrupt` 暂停图执行，value 透传给客户端；恢复必须用 `Command(resume=...)`，图从节点**开头重新执行**已执行过的逻辑，interrupt 以"节点内首次调用返回恢复值、后续调用真抛异常"的方式幂等化（`types.py:913` 函数文档）。同一节点内多个 interrupt 调用按出现顺序与恢复值匹配，恢复值作用域限定在执行该节点的任务内（函数文档）。`Command`（`types.py:843`）四字段：`update`（状态更新）、`goto`（节点名/Send，动态跳转）、`resume`（恢复值）、`graph`（PARENT 表示发给父图，支持子图通信）。interrupt 依赖 checkpointer（文档明示），恢复匹配按 interrupt_id（`types.py:660`）。这套机制与 `StateSnapshot.interrupts`、`PregelLoop` 的 `CONFIG_KEY_RESUMING`（`_internal/_constants.py`）一起构成 human-in-the-loop 完整闭环。

### 5.5 Functional API 与 prebuilt

`task`/`entrypoint`（`libs/langgraph/langgraph/func/__init__.py:110,262`）提供"写普通函数也能获得持久化"的入口：`task` 把函数变成可恢复单元，`entrypoint` 是带输入输出的图入口，底层同样落到 Pregel——即两种 API（Graph API / Functional API）共用一个运行时。`create_react_agent`（`libs/prebuilt/langgraph/prebuilt/chat_agent_executor.py:278`）+ `ToolNode`（`tool_node.py:630`，2064 行）是最常用的 ReAct 模板，含工具调用校验（`tool_validator.py`）、ToolRuntime 注入（`tool_node.py:1697`）与中断处理（`interrupt.py`）。

## 6. 构建与运行

```bash
pip install -U langgraph            # 用户安装（README）
# 开发（monorepo，根 Makefile + AGENTS.md）：
make install                         # uv venv + 逐包 uv pip install -e
make lint / make format              # 逐包跑 ruff 等
make test                            # 逐包 pytest
TEST=path/to/test.py make test       # 指定测试文件（AGENTS.md）
uv add "langgraph-cli[inmem]"        # CLI + 内存态本地 server（libs/cli/README.md）
langgraph dev / build / up           # 本地开发、Docker 构建、运行
```

## 7. 测试与工程质量

- 规模：全仓 `libs/**/tests/` 共 **219 个测试文件、约 11.9 万行**测试代码（实测 `find libs -path "*/tests/*" -name "*.py" | wc -l` = 219，`wc -l` 合计 118,802）；分布：langgraph 76、sdk-py 71、cli 24、prebuilt 18、postgres/sqlite 各 11、checkpoint 7；核心库测试/源码比例约 1:1（79 源文件）
- 风格：pytest + pytest-xdist 并行 + pytest-repeat（并发稳定性）+ syrupy 快照测试（`libs/langgraph/pyproject.toml` test 组）；postgres/sqlite 各自带 `test_conformance_delta.py` 等基于 `libs/checkpoint-conformance` 合规套件的测试，保证三个 saver 行为一致
- CI：`.github/workflows/ci.yml` 编排；`_test.yml` Python 版本矩阵测试；`_lint.yml`、`_integration_test.yml`、`_sdk_integration_test.yml`、`bench.yml`（基准）、`pr_lint.yml`、`require_issue_link.yml`（强制 PR 关联 issue）、`uv_lock_ugprade.yml`（锁文件自动升级）；release.yml 拆分 build/release 两 stage 隔离权限并走 trusted publishing，工程化程度很高
- 示例与集成验证：`examples/` 收录 20+ notebook（multi_agent、human_in_the_loop、llm-compiler、plan-and-execute、reflexion、rewoo、usaco 等），`libs/cli/` 附 python/js monorepo 示例工程；postgres 测试目录内含 `compose-postgres.yml` 供本地起库
- 代码质量观察：核心引擎类型注解完整（`py.typed`），公共 API 收敛严格（AGENTS.md 明文"一个功能只有一种写法、禁止便利性别名"），引擎与构建器分层清晰；移除公开行为须经正式 deprecation 流程并注明迁移说明（AGENTS.md）

## 8. 社区与维护状态

- 活跃度：最近 push 2026-11-05（gh api，即分析前一天）；open issues 461、forks 7661；核心库自带基准测试目录 `libs/langgraph/bench/`，CI 有 `bench.yml` 常态化跑性能基线
- 维护方：LangChain Inc 主导开发（README），贡献者 290+（API 翻页估算，未确认精确值）；AGENTS.md 出现 "Corridor security analysis" 段落，说明已引入 AI 辅助开发流程
- release 节奏：每包独立版本独立发布，langgraph 已到 1.2.20、checkpoint 4.3.1，patch 号极高说明发布频繁（pyproject.toml）；发版由 `release.yml` 手动触发按包执行
- 生态位：PyPI 徽章显示下载量统计入口（README 徽章）；Deep Agents、langgraph-supervisor/swarm/cua 等 agent 级包被明确降级为"仅维护"（AGENTS.md），项目重心转向运行时正确性

## 9. 亮点与风险

亮点：
1. 执行模型扎实：Pregel/BSP 超步语义 + channel 版本号增量计算，是少数把分布式图计算模型完整落地的 agent 框架（`libs/langgraph/langgraph/pregel/main.py:456`、`_algo.py`）
2. 持久化是一等公民：checkpointer 接口 + 三实现 + 合规测试套件（`libs/checkpoint-conformance/`），interrupt/Command/time-travel 全部建立在同一套 checkpoint 语义上
3. 工程纪律强：monorepo 每包独立版本与发布、conformance 测试、强制 issue 关联、双语言行为同步的成文规范（AGENTS.md）
4. API 收敛哲学：明文"一种功能一种写法、只减不增"（AGENTS.md），在快速演进的 agent 框架中罕见，利于长期稳定性
5. 商业闭环与开源边界清晰：运行时开源（MIT），观测/部署/托管在 LangSmith 侧

风险/局限：
1. 定位收缩使 `prebuilt`（create_react_agent 等）进入仅维护模式，直接用 LangGraph 裸写 agent 的路径被官方弱化，教程生态可能向 Deep Agents 迁移（AGENTS.md）
2. 核心引擎复杂度高：`pregel/main.py` 4557 行、`_loop.py` 2202 行，且含大量 resume/replay 边界处理，社区贡献门槛高
3. 强绑定 langchain-core 与自家 SDK（`libs/langgraph/pyproject.toml`），脱离 LangChain 生态使用时部分能力（回调追踪）受限
4. 文档主体外移至 docs.langchain.com（`docs/` 目录仅剩重定向），仓库内自包含文档变薄；`config_schema` 等弃用周期已排到 v2.0（`graph/state.py` 弃用警告）

## 10. 总结评价

LangGraph 是"agent 编排"赛道事实上的工程化标杆：它不追求上手最快，而是把长时运行 agent 最难的四个问题——状态持久化、失败恢复、人工介入、并发聚合——收敛为一套自洽的图运行时，其 Pregel 引擎、checkpoint 语义与 Command/interrupt 机制值得任何做 agent 基础设施的人精读；当前项目正从"框架"转型为"运行时"，上层易用性交给了 Deep Agents。适合需要可控、可审计、可恢复 agent 流程的后端/平台工程师选用，也适合研究 agent 编排内部机制的读者读源码（建议入口：`libs/langgraph/langgraph/graph/state.py` → `pregel/main.py` → `_loop.py` → `checkpoint/base/__init__.py`）；只想快速搭 agent 原型的新手则更适合其上层的 Deep Agents 或 prebuilt 模板。
