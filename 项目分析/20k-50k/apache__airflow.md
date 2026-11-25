# apache/airflow 项目分析

> 星数 41000（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：Apache Airflow 是以 Python 代码定义工作流（DAG）、由调度器按依赖关系编排、在可插拔执行器（Local/Celery/Kubernetes 等）上运行并配全功能 Web UI 的工作流编排平台。

| 项目 | 事实 |
|---|---|
| 名称 | apache/airflow（Apache 软件基金会顶级项目） |
| 星数 | 41000（快照 2026-10-24）；gh api 实测 2026-11-04 已 47360，仍在快速增长 |
| 语言 | Python（核心）+ TypeScript（React UI）+ 少量 Go/Java/Scala SDK |
| 许可证 | Apache-2.0（LICENSE，gh api license: Apache-2.0） |
| 版本 | main 分支版本号 3.5.0（airflow-core/pyproject.toml）；最新 release 3.3.2（2026-09-17，RELEASE_NOTES.rst） |
| 最近活跃 | repo pushed_at 2026-11-04（昨天），极活跃；贡献者约 4767 人（gh api contributors 分页统计，含匿名） |
| 规模 | 全仓 Python 文件 7787 个，核心包 airflow-core 773 个，测试文件 2126 个（实测统计） |

## 2. 项目是什么，解决什么问题

Airflow 解决的核心问题是：把散落在 crontab、脚本、 notebooks 里的数据/工程任务，变成**用代码定义、可版本化、可测试、可协作**的 DAG（有向无环图），由平台统一负责调度、重试、依赖管理和监控（README.md："When workflows are defined as code, they become more maintainable, versionable, testable, and collaborative"）。

一个 DAG 就是一个 Python 文件：用 `@dag` / `@task` 装饰器（TaskFlow API）或传统 operator 声明任务及依赖，Airflow 将其序列化进元数据库。`DagProcessor` 子进程持续解析 DAG 文件（`airflow-core/src/airflow/dag_processing/dagbag.py` 的 `DagBag.process_file`），`SchedulerJobRunner`（`airflow-core/src/airflow/jobs/scheduler_job_runner.py`，4694 行）按调度时间表与依赖关系把就绪任务推入 Executor，Executor 在 worker 上真正执行任务实例，执行结果经事件回流由调度器更新状态。Timetables 体系（`airflow-core/src/airflow/timetables/`）把"何时触发"抽象为插件化对象，支持 cron、事件驱动、资产（Data Assets）触发。

Airflow 3.x（HEAD 已是 3.5.0 开发版）带来架构级重构：核心拆成多个独立发布的 Python 包——`airflow-core`（调度/模型/）、`task-sdk`（任务运行侧 SDK，任务不再直接依赖核心）、`airflow-ctl`（走 API 的客户端 CLI）；所有组件不再直连元数据库，而是通过 FastAPI 实现的 Execution API（`airflow-core/src/airflow/api_fastapi/execution_api/`）通信，为多语言 SDK（go-sdk/、java-sdk/、ts-sdk/ 目录均已出现）和边缘执行铺路。

生态是其护城河：仓库内自带 84 个 provider 包（`providers/` 下每个子目录是独立 PyPI 发行版，如 providers/amazon/），覆盖 AWS/GCP/Azure/数据库/K8s/消息队列/AI 服务（anthropic、openai 等）等集成，provider 通过 `provider.yaml` 声明版本、hooks/operators/sensors 清单。

## 3. 技术栈与依赖

- 语言：Python >=3.11（`requires-python = ">=3.11,!=3.15"`，airflow-core/pyproject.toml），支持 3.11-3.14。
- ORM/迁移：SQLAlchemy >=2.0.48 + Alembic（155 个迁移版本，`airflow-core/src/airflow/migrations/versions/`），支持 Postgres/MySQL/SQLite。
- API：FastAPI + Gunicorn（`api_fastapi/gunicorn_app.py`）；旧 Flask Airflow API 已由新 API 取代。
- 前端：React + Vite + TanStack Query/Table/Virtual + Vitest/Playwright（`airflow-core/src/airflow/ui/package.json`）。
- 构建：uv workspace 多包 monorepo（根 pyproject.toml `[tool.uv.workspace]`，成员含 airflow-core/task-sdk/各 provider 等 100+ 包）；Breeze（`dev/breeze`）为开发者 CLI。
- 部署：Dockerfile（官方镜像）、Helm Chart（chart/，含 api-server、dag-processor、scheduler、triggerer、flower 等 12+ 模板组件）。
- 执行：标准库 multiprocessing（LocalExecutor）；Celery/K8s executor 由 providers/celery、providers/cncf 提供。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| airflow-core/ | 核心包：调度器、模型（Dag/DagRun/TaskInstance）、api_fastapi、dag_processing、executors、timetables、UI |
| task-sdk/ | 任务运行侧 SDK：TaskFlow 定义（sdk/definitions/）、任务执行 API 客户端、XCom |
| airflow-ctl/ | Airflow 3 CLI 客户端（通过 REST API 与服务端交互） |
| providers/ | 84 个 provider 发行包，每个含 src/、tests/、docs/、provider.yaml |
| shared/ | 供 airflow-core/task-sdk 共享的模块（configuration、serialization、secrets_masker 等 14 个），经符号链接挂入 `_shared/` |
| chart/ | 官方 Helm Chart（K8s 部署） |
| go-sdk/、java-sdk/、ts-sdk/ | 多语言 SDK（Airflow 3 新增） |
| dev/、scripts/、generated/ | Breeze 工具、CI 脚本、生成产物（constraints、镜像清单） |
| docs/、performance/ | 各发行版文档源、性能测试 |

组织方式评述：典型的 **uv workspace 多包 monorepo**，从 Airflow 1.x 的单包演进而来。每个 provider、每个核心发行版都是独立 pyproject + 独立版本号，由 release manager 统一协调（providers/PROVIDER_RELEASES.rst）；`shared/` + 符号链接（`airflow/_shared/* -> shared/*`）解决跨发行版代码复用——这是 tarball 中 31 个符号链接条目的来源，本体均在 shared/ 下，解压排除链接不损失内容。

## 5. 核心架构与关键实现

### 5.1 调度器主循环（多执行器 + 定时器驱动）

`_run_scheduler_loop`（airflow-core/src/airflow/jobs/scheduler_job_runner.py:1987）每轮迭代做四件事：`_do_scheduling` 出队任务 → 逐个 `executor.heartbeat()` 驱动执行器 → `_process_executor_events` 消费执行结果 → 处理过期 Deadline/回调；周边用 `EventScheduler` 挂了十余个定时巡检（孤儿任务、trigger 超时、卡死 queued 任务、指标上报等）。空闲时按需 sleep 降 CPU（`idle_in_this_run` 分支）。关键片段：

```python
for loop_count in itertools.count(start=1):
    with create_session() as session:
        num_queued_tis = self._do_scheduling(session)
        session.expunge_all()
    for executor in self.executors:
        executor.heartbeat()
    with create_session() as session:
        for executor in self.executors:
            num_finished_events += self._process_executor_events(executor=executor, session=session)
```

值得注意 `self.executors` 是列表：单调度器可同时挂多个执行器（executor_loader.py 按 team 分组加载），并支持调度器 HA——所有状态推进用 `with_row_locks(..., skip_locked=True)` 抢行锁（scheduler_job_runner.py:819/1619/1649），多副本不会重复调度。

`_do_scheduling`（:2189）分三步：按 `next_dagrun_create_after` 批量建 DagRun（默认每轮仅 10 个，防创建挤占调度）、取"最久未检查"的 20 个 running DagRun 推进状态、最后在 Pool 行锁临界区把 TIs 批量置 queued 并发给执行器。注释明确解释了这些上限的吞吐权衡（大 DAG >500 任务时增大反会降吞吐）。

### 5.2 执行器抽象与 LocalExecutor

`BaseExecutor`（airflow-core/src/airflow/executors/base_executor.py:201，963 行）定义接口：`queue()` 收工单、`heartbeat()` 驱动异步执行、`sync()` 收事件。任务以 `ExecutorWorkload` 对象传递。Airflow 3 中 Celery/K8s 执行器已外移为 provider（executor_loader.py 仍按 CELERY_EXECUTOR/KUBERNETES_EXECUTOR 常量加载），核心只剩 LocalExecutor（local_executor.py:121）：基于 multiprocessing 简单队列 + 常驻 worker 进程池执行任务，`is_mp_using_fork` 适配 Linux fork 与 Windows spawn 两种启动方式。

### 5.3 DAG 解析与"包"（Bundle）体系

DAG 解析从调度器中独立成 `dag_processing` 模块：`DagFileProcessorManager`（dag_processing/manager.py:198）维护文件队列与 run_count 统计，子进程里 `DagBag.process_file`（dagbag.py:289）import 模块提取 DAG 并序列化入库；解析结果经 `DagFileProcessorAgent` 被 `SchedulerJobRunner` 收割（`_run_scheduler_loop` 文档串第一步）。`bundles/`（local/manager/manifest + provider.yaml）是 3.x 新概念：DAG 文件来源抽象为版本化 bundle，调度器按 bundle-team 关联解析/执行（主循环开头 `self._dag_id_to_team_name = {}` 每轮刷新），支持 Git/对象存储等远程源。

### 5.4 TaskFlow API 与 Task SDK

TaskFlow 的实现在 task-sdk：`@task`/`@dag` 动态生成的装饰器族（`task-sdk/src/airflow/sdk/definitions/decorators/__init__.pyi` 的 TaskDecoratorCollection：task/task_group/dag/setup/teardown/condition），核心逻辑在 `sdk/definitions/_internal/decorators.py`，依赖展开（`.expand()` 动态映射）在 `_internal/expandinput.py`；XCom 数据依赖通过 `xcom_arg.py` 的 XComArg 在解析期建边。任务运行期通过 `sdk/api/` 客户端调用 Execution API 写日志/XCom/心跳，与核心解耦——AIP-44 之后"任务侧不再直连数据库"的落点。

## 6. 构建与运行

用户侧（README.md / INSTALLING.md）：

```bash
pip install "apache-airflow[postgres]"          # 或 pip install apache-airflow
pip install "apache-airflow[celery,cncf.kubernetes]"
airflow standalone                              # 一键起全套（含示例）
airflow db migrate && airflow users create ...  # 初始化
airflow scheduler | airflow webserver | airflow api-server | airflow triggerer
```

Docker/Helm：官方镜像 `apache/airflow`（Dockerfile），K8s 用 `chart/`（Helm Chart，模板含 api-server/dag-processor/scheduler/jobs）。

开发者侧（Breeze，README.md、BREEZE.rst）：

```bash
./dev/breeze/build-image-prod.sh                # 构建镜像
breeze shell                                    # 进入开发容器
breeze testing tests --test-type All            # 跑测试
pre-commit run --all-files                      # 静态检查（prek/pre-commit）
```

## 7. 测试与工程质量

- 规模：全仓 2126 个 `test_*.py`（实测），分属 airflow-core/tests、task-sdk/tests、chart/tests（helm 单元测试）、airflow-e2e-tests/、task-sdk-integration-tests/、kubernetes-tests 等独立测试包；provider 各自带 tests/（providers/amazon/tests/ 等）。
- CI：`.github/workflows/` 66 个 workflow（gh api 实测）：ci-amd.yml/ci-arm.yml 双架构测试矩阵、helm 测试、e2e、codeql、dependency-review、flaky 测试报表（e2e-flaky-tests-report.yml）、自动 backport（automatic-backport.yml）、constraints 每日生成（generate-constraints.yml）。测试约束通过 uv.lock + constraints 体系严格控制版本。
- 质量观察：核心代码 type hints 全覆盖（py.typed 标记）、有 JOB_LIFECYCLE.md 等架构文档、性能测试目录 performance/、codecov 配置。4694 行的 scheduler_job_runner.py 单文件偏大，但内部方法文档注释详尽，且大量行级注释解释调度权衡，可读性尚可。

## 8. 社区与维护状态

- ASF 顶级项目，治理文件齐全（GOVERNANCE.md、COMMITTERS.rst、PROVIDERS.rst、Voting Policy 见 README 目录）；有专门的品牌/下线流程文档（INTHEWILD.md、SUSPENDING_AND_REMOVING_PROVIDERS.rst）。
- 发版节奏极快：RELEASE_NOTES.rst 显示 3.3.2 于 2026-09-17 发布，3.4.0 已在 unreleased 区（含 XCom v2 表迁移），main 版本号 3.5.0；README 徽章显示 v3-2-test 分支并行维护。
- 贡献者约 4767 人（gh api 分页，含匿名贡献者），repo push 至今活跃（2026-11-04）；open issues 1938（gh api）。
- README 新增 "Agent-assisted contribution (apache-magpie)" 章节，说明已接纳 AI 辅助贡献流程——活跃度与流程成熟度的侧面印证。

## 9. 亮点与风险

亮点：
1. 调度器设计成熟：行锁 HA、批量上限防抖、空闲降载、Deadline/回调管线，全部有源码可证（scheduler_job_runner.py）。
2. 多包 monorepo + provider 生态（84 个）是数据工程事实标准之一；provider.yaml 清单让第三方集成完全工程化。
3. Airflow 3 重构方向清晰：task-sdk 与核心解耦、FastAPI Execution API、多语言 SDK（go/ts/java）为下一阶段铺路。
4. 工程基建顶级：66 个 CI workflow、双架构矩阵、flaky 跟踪、每日 constraints、pre-commit/prek。
5. TaskFlow API + Timetables + Assets 让"工作流即代码"体验显著优于 1.x 的 operator 拼接。

风险/局限：
1. 巨型 monorepo 上手成本高：100+ workspace 包、Breeze/Docker 工具链复杂，新人贡献曲线陡。
2. 调度器与元数据库强耦合，超大 DAG（千级任务/30 亿行 XCom 量级迁移在 3.4.0 说明中提及）仍需 DBA 级运维。
3. 版本演进快、API 面大：3.x 大重构带来兼容成本（XCom v2 迁移、降级受限的警示写在 RELEASE_NOTES.rst）。
4. LocalExecutor 仅适合小规模；真实生产依赖 Celery/K8s provider，核心仓库内看不到完整执行路径。

## 10. 总结评价

Apache Airflow 是工作流编排领域定义范式级别的项目：把 DAG-as-code、调度器/执行器/触发器分离、插件化集成体系做成了行业标准，4 万+星来自其不可替代的数据工程生态位。当前 HEAD（3.5.0 开发版）正处在继往开来的节点——task-sdk/Execution API/多语言 SDK 让它从"Python 平台"走向"编排协议"。适合数据平台工程师生产使用，也适合想学大规模调度系统设计（并发控制、行锁 HA、状态机推进）的工程师精读 scheduler_job_runner.py 与 base_executor.py；但直接改造 monorepo 对新手不友好，建议从某个 provider 包入手。
