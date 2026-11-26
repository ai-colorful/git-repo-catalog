# nicolargo/glances 项目分析

> 星数 28500（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：Glances 是一个跨平台"一眼看清系统"的监控工具——单个命令在终端 TUI / Web UI / REST API / MCP 四种形态下展示 CPU、内存、磁盘、网络、容器、传感器等全部指标，并可导出到 24 种后端。

| 项 | 值 |
|---|---|
| 名称 | nicolargo/glances |
| 星数 | 33,998（gh api 2026-11-21 实测；快照 28,500 @ 2026-10-24，一个月净增约 5,500 星） |
| 语言 | Python（含 Vue 3 WebUI 前端） |
| 许可证 | LGPL-3.0（`COPYING`、`pyproject.toml` `license = "LGPL-3.0-only"`、`LICENSES/LGPL-3.0-only.txt`） |
| 创建时间 | 2011-12-04（gh api 核实，实际早于常见的"2014"说法） |
| 最近活跃 | 最近 push 2026-11-21（分析当天） |
| 当前版本 | 稳定 4.5.9（`support/glancesv4` 分支）；develop 分支为 5.0.0b2 完全重写版（`glances/__init__.py`） |
| 默认分支 | develop |
| 贡献者 | 251 人（gh api contributors 分页合计）；fork 1,841；watch 490；open issues 105 |

## 2. 项目是什么，解决什么问题

Linux 上有 top/htop，Windows 有任务管理器，macOS 有活动监视器——但它们都只覆盖单平台、单形态。Glances 用 Python + psutil 把这些能力统一到一个工具里：一条 `glances` 命令，在 GNU/Linux、BSD、macOS、Windows（以及 WSL）上以 curses 终端界面呈现全部关键指标（README.rst、`glances/globals.py` 的平台常量）。

它解决的核心问题是"监控的统一入口"：同一份采集数据同时供给四种消费形态——Curses TUI（`glances/outputs/glances_curses.py`）、Vue 3 Web UI（`glances/outputs/static/`）、REST API（`glances/routes.py` 挂载于 `/api/5`）、MCP 服务器（`glances/outputs/glances_mcp.py`，可让 Claude 等 LLM 直接查询系统状态）。此外还有 24 个导出模块把指标推送/拉取到时序数据库与消息队列，使其可以嵌入 Prometheus/Grafana 等正经监控体系（`glances/exports/`）。

另一个核心问题是阈值告警。每个插件字段都声明 `careful/warning/critical` 三级阈值与 `watch_direction`（如 `glances/plugins/cpu/model.py` 中 `total` 字段 50/70/90），由告警引擎（`glances/alerts.py`）聚合为全局告警，并可触发 action（`glances/actions/`，如阈值命中时执行脚本）。

项目当前正处于代际切换点：develop 分支的 Glances 5 是对采集引擎、REST API、TUI、Web UI 的完全重写（`glances-v4-vs-v5.md` 自述），稳定线 4.5.x 在 `support/glancesv4` 分支继续维护。重写动机是架构债：v4 的按需采集 + XML-RPC + SNMP fallback 被 v5 的 asyncio 调度器 + 纯 REST 取代，同时按 2026-10-17 安全审计结果加固（默认绑定 127.0.0.1、PBKDF2-SHA256 60 万轮口令哈希、修复 CVE-2026-33641 配置项命令注入等，见 `glances-v4-vs-v5.md` 安全章节）。

## 3. 技术栈与依赖

- 语言与运行时：Python ≥3.11（`pyproject.toml` classifiers 支持 3.11–3.15），WebUI 为 Vue 3 + webpack（`glances/outputs/static/package.json`）。
- 硬依赖（`pyproject.toml [project].dependencies`）：仅 5 个——`psutil>=5.6.7`（一切系统指标之源）、`requests`、`jinja2`（fetch 模板）、`windows-curses`（仅 Windows）、`shtab`（仅非 Windows 补全）。
- 分层可选依赖（`[project.optional-dependencies]`，12 组 extras）：
  - `web`：fastapi、uvicorn、PyJWT、msgpack、starlette（CVE 下限版本钉住）；
  - `containers`：docker、podman、pylxd、python-dateutil；
  - `export`：19 个后端客户端库（influxdb-client、prometheus_client、kafka-python、paho-mqtt、pika、pymongo、elasticsearch、clickhouse-connect、bernhard(Riemann)、potsdb、statsd、pyzmq、nats-py、psycopg、pycouchdb、graphitesender、influxdb v1/v3 等）；
  - `gpu`（nvidia-ml-py）、`graph`（pygal）、`mcp`（mcp>=1.23.0）、`raid`（pymdstat）、`smart`（pySMART）、`sensors`（batinfo，仅 Linux）、`browser`（zeroconf）、`action`（chevron）。
- 锁文件：`requirements.txt` / `all-requirements.txt` / `docker-requirements.txt` / `dev-requirements.txt` 均由 uv export 生成、带全量 hash（`Makefile` 的 `requirements-*` 目标），走 uv 工作流（`[tool.uv.workspace]`）。
- 开发工具链：ruff（lint+format，`[tool.ruff]` 行宽 120、mccabe 复杂度上限 21）、pytest/pytest-asyncio、playwright（WebUI 浏览器测试）、semgrep、pre-commit、sphinx（`dev-requirements.txt` / `pyproject.toml`）。
- 打包分发：setuptools + PyPI、Snap（`snap/snapcraft.yaml`）、Docker（`docker-files/alpine.Dockerfile`、`ubuntu.Dockerfile` + `docker-compose/`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `glances/` | 主包，约 39,900 行 Python、235 个 .py 文件（实测统计） |
| `glances/plugins/` | 34 个监控插件 + `plugin/` 插件基类目录（清单见第 5 章） |
| `glances/exports/` | 24 个导出后端 + `export_base.py` 统一基类 |
| `glances/outputs/` | 三种 UI 输出：curses 系列、stdout、`static/`（Vue 3 WebUI，约 1.2MB）、`glances_mcp.py`（MCP 服务器）、`restful_doc.py` |
| `glances/actions/`、`glances/amps/` | 阈值触发动作；应用监控程序（AMP）插件框架 |
| `tests/`、`tests-data/` | 166 个 .py 测试文件 + 固定数据（含伪 sysfs 树） |
| `conf/` | 主配置 `glances.conf`、Grafana 仪表盘 JSON、fetch 模板、主题 CSS |
| `docs/` | Sphinx 文档 + man page（`docs/man/glances.1`） |
| `.github/workflows/` | ci/quality/test/webui/build/build_docker/cyber/inactive_issues/needs_contributor 共 9 个 workflow |
| 顶层文件 | `pyproject.toml`（唯一工程清单）、`Makefile`（uv 工作流）、`glances-v4-vs-v5.md`（迁移指南）、`CLAUDE.md` |

组织方式评述：单包单体（single package monolith），但内部是严格的三层流水线——采集（plugins）→ 存储/聚合（stats_store/history/alerts）→ 消费（outputs/exports/routes），依赖方向单向。所有插件/导出模块遵循统一文件布局：`<name>/model.py`（实现）+ `<name>/render_curses.py`（TUI 渲染），渲染逻辑与数据逻辑分离（如 `glances/plugins/cpu/render_curses.py`、`glances/plugins/containers/render_curses.py`）。

## 5. 核心架构与关键实现

### 5.1 插件体系：34 个监控插件 + 泛型异步基类

`glances/plugins/` 下 34 个监控插件：cpu、percpu、mem、memswap、load、uptime、system、version、now、quicklook、diskio、fs、network、ip、connections、ports、irq、processcount、processlist、programlist、containers（引擎子分 docker/podman/lxd，见 `glances/plugins/containers/engines/`）、gpu、npu、vms、cloud、sensors、smart、raid、wifi、folders、amps、core、mpp、psutilversion（目录清单实测）。

v5 插件基类 `glances/plugins/plugin/base.py`（1,351 行）是典型的"重基类"设计——基类承担流水线全部逻辑，具体插件只实现 `_grab_stats()` 并声明元数据：

```python
class GlancesPluginBase(Generic[T], ABC):
    plugin_name: ClassVar[str] = ""
    IS_COLLECTION: ClassVar[bool] = False   # False=标量插件(cpu/mem)，True=集合插件(fs/network)
    DEPENDS_ON: ClassVar[tuple[str, ...]] = ()   # 插件间依赖声明
    EMITS_ALERTS: ClassVar[bool] = True      # 是否驱动告警管线
    DISPLAY_IN_TUI: ClassVar[bool] = True
```

类型参数 `T` 区分两种插件形态（`GlancesPluginBase[dict]` 标量 / `GlancesPluginBase[list]` 集合，见模块 docstring）。每个字段在 `fields_description` 中声明 description/unit/history/watched/prominent/default_thresholds 等模式信息（`glances/plugins/cpu/model.py`），基类据此统一实现历史记录、阈值分级（`glances/plugins/plugin/thresholds.py`）、API 路径、导出过滤（`exportable=False` 的内部字段不导出）。

注册机制是约定式自动发现（`glances/main.py:discover_plugin_classes()`，约 907 行起）：`pkgutil.iter_modules` 遍历 `glances.plugins` 包，导入每个子包的 `model.py`，取 `PluginModel` 类并校验其继承 `GlancesPluginBase`；配置 `[<plugin>] disable=true` 的插件根本不实例化（因其 `__init__` 有副作用，如 ports 插件启动后台线程）。导出模块用同样机制发现（`main.py` 约 1,081 行）。注意：v5 移除了 v4 的外部插件目录（`-P/--plugins`，见 NEWS.rst 破坏性变更列表）。

### 5.2 多 UI 形态：一个存储，四种消费

v5 的核心架构决策是共享统计存储 `glances/stats_store.py`（全文 97 行，注释即设计文档）：每个插件一个 asyncio.Task 按自己的 `refresh` 频率采集（`glances/scheduler.py`，一插件一 Task + 空闲降频），写入共享 `StatsStore`；写侧用 `asyncio.Lock` 串行化，读侧完全无锁——依据是"存储永远整体替换顶层键、从不原地改子结构"+"CPython GIL 下 dict 单键读写是原子字节码"，并提供单调 `revision` 计数器供 TUI 启动追赶期轮询。TUI、REST API、导出器、MCP 服务器四个消费者都从这个 store 读，互不耦合。

REST/Web 侧用 FastAPI（`glances/webserver.py:build_app()` 构建应用，`glances/routes.py` 定义 `APIRouter(prefix="/api/5")`）：路由表覆盖 `/api/5/all`、`/api/5/<plugin>/{info,limits,history,top}`、`/api/5/<plugin>/<field>/<pk_value>` 等细粒度查询，另有 JWT 认证（`/api/5/token`）与 MessagePack 编码端点（`/api/5/all/msgpack`）。需要修正一个常见旧印象：Web 后端在此版本已是 FastAPI 而非 Bottle（`pyproject.toml` classifier `Framework :: FastAPI`）；Bottle 属于 v4 稳定线，未在本次源码范围内，未确认。TUI 则在 `glances/outputs/glances_curses.py`（2,703 行）中，配合每插件的 `render_curses.py`。

MCP 服务器（`glances/outputs/glances_mcp.py`，33 个函数）把插件查询封装为 MCP tools/resources，兼容 MCP Python SDK 1.x（FastMCP）与 2.x（MCPServer）——这是同类工具中少见的"让 LLM 读系统状态"入口。

### 5.3 导出层：24 个后端共享一个基类

`glances/exports/` 实测 24 个模块：cassandra、clickhouse、couchdb、csv、duckdb、elasticsearch、graph(pygal 本地图)、graphite、influxdb（v1/v2/v3 三代独立模块）、json、kafka、mongodb、mqtt、nats、opentsdb、prometheus、rabbitmq(bernhard)、restful、riemann、statsd、timescaledb、zeromq。

统一抽象 `glances/exports/export_base.py`：三类职责——配置读取（`load_conf()`）、线上格式兼容（`build_export()`/`normalize_for_influxdb()` 从 v4 原样移植，保证用户仪表盘字段名不变）、payload 准备（注入 key 字段、合并插件配置为 limits）。一个值得注意的工程决策写在了 docstring 里：`update()`/`export()` 刻意保持同步（所有后端客户端都是阻塞的），异步边界放在调度器的 `_export_loop()` 里用 `asyncio.to_thread` 包裹——比一个"全 async 但处处阻塞"的假异步更诚实。导出频率被钳制为不低于全局采集频率（`resolve_export_refresh()`），避免导出重复点位。

### 5.4 统计聚合与历史

v4 的中心聚合器 `glances_stats` 在 v5 中被拆解为三件套（v4 文件不在本快照，此对照来自 `glances-v4-vs-v5.md` 与本分支实际文件）：`StatsStore`（最新值 KV 存储）、`glances/history.py` 的 `HistoryStore`（服务端列式历史，默认 1200 点，以 `{timestamps, series}` 结构经 `/api/5/<plugin>/history` 输出）、`glances/alerts.py` + `glances/alerts_incidents.py`（新告警引擎，`EMITS_ALERTS=False` 的插件不进入告警管线但保留 TUI 配色）。阈值分级计算独立成 `glances/plugins/plugin/thresholds.py`（`compute_level` / `compute_level_categorical`），插件声明式阈值无需自己写比较逻辑。

### 5.5 跨平台适配

平台常量集中在 `glances/globals.py:73-79`：`BSD/LINUX/MACOS/SUNOS/WINDOWS/WSL`，并按平台选择 multiprocessing fork 上下文（POSIX 用 fork，其余默认）。psutil 差异处理策略是"能声明的走依赖标记，能隔离的走引擎子类"：`windows-curses`/`shtab`/`batinfo` 用 `platform_system` 条件依赖（`pyproject.toml`）；容器监控按运行时而非 OS 分引擎（docker/podman/lxd）；wifi 插件在 Linux 直接读 nl80211 所以 `wifi` extra 为空（`pyproject.toml` 注释）；system 插件对 BSD/SunOS 的 ostype 特判（`glances/plugins/system/model.py:106`）。CI 用四平台矩阵（Linux/Windows/macOS/FreeBSD-QEMU，见第 7 章）兜底回归。

## 6. 构建与运行

来自 `pyproject.toml`、`Makefile`、`.github/workflows/test.yml` 的实际命令：

```bash
# 安装（最小：仅硬依赖 + TUI）
pip install glances
# 全量可选功能
pip install "glances[all]"

# 运行
glances            # TUI
glances -w         # Web/REST 服务器（端口 61208，默认绑定 127.0.0.1）
glances -c <host>  # TUI 连远程服务器（v5 经 REST API，XML-RPC 已移除）

# 开发（uv 工作流，Makefile）
make venv-dev          # 建虚拟环境
make test              # 全部单测（pytest tests/）
make test-core / test-plugins / test-api / test-webui / test-js
make test-export-csv   # 导出接口测试（另有 json/timescaledb 等目标）
make requirements      # uv export 重新生成 4 个 requirements 锁文件

# CI 内的实际测试命令（.github/workflows/test.yml）
python -m pip install ".[all]" pytest pytest-asyncio docutils playwright
python -m playwright install --with-deps chromium
python -m pytest tests/
```

不运行构建（按规范仅静态阅读）；以上均来自工程文件原文。

## 7. 测试与工程质量

- 测试规模：`tests/` 166 个 .py 文件（实测），v5 文档自述"161 test files"（`glances-v4-vs-v5.md`）。覆盖单元（插件/配置/API）、导出接口测试（`tests/test_export_*.py` + `.sh`，真连 CSV/TimescaleDB 等后端）、JS 单测（`tests/test_webui_js.py`）与 Playwright 真浏览器测试（`tests/test_webui_browser*.py`），MCP 测试说明见 `tests/HOW_TO_TEST_MCP.md`。pytest-asyncio `asyncio_mode=auto`（`pyproject.toml`）。
- CI 矩阵（`.github/workflows/test.yml`）：Ubuntu 24.04 × Python 3.11/3.12/3.13/3.14/3.15 全量测试；Windows 2025 × 3.14；macOS 15 × 3.14；FreeBSD 通过 `vmactions/freebsd-vm`（QEMU/KVM 虚机）跑全测试——四操作系统测试在个人项目中罕见。`ci.yml` 串联 quality（ruff + semgrep + `cyber.yml` 安全扫描）→ test → webui → build（PyPI 构建 + attestation）→ build_docker；另有 `inactive_issues.yml`、`needs_contributor.yml` 两个 issue 治理机器人 workflow。
- 质量观察：全仓库 SPDX 许可证头注释；ruff 强制 lint；锁文件带全量 hash；可选依赖钉 CVE 下限版本（如 `starlette>=0.49.1` 注明 CVE-2025-62727）；导出器做了 TLS 校验、CSV 公式注入防护、Jinja2 沙箱渲染（`glances-v4-vs-v5.md` 安全章节，均有对应源码模块）。`sonar-project.properties`、Codacity/Scrutinizer 徽章（README.rst）表明长期质量监控习惯。

## 8. 社区与维护状态

- 活跃度（gh api 2026-11-21）：最近 push 即分析当天；open issues 105（控制得很好）；watch 490。
- 发版节奏（gh api releases）：稳定线约每月一版——v4.5.0（2026-02-08）→ v4.5.1/4.5.2/4.5.3/4.5.4/4.5.5/4.5.6/4.5.7（2026-09-26）/4.5.8（2026-10-11）/4.5.9（2026-11-07），10 个月 10 个版本；develop 线已发 5.0.0b1/b2 预发布（beta 不移动 Docker `latest` 标签，NEWS.rst）。
- 贡献者结构（gh api contributors）：共 251 人。头部：nicolargo 5,628 次提交（绝对主导）、foralesergi 506、RazCrimson 232、claude（AI 助手）226、renovate[bot] 165。单人长期维护 + 中等规模社区贡献 + AI/机器人辅助的混合模式；`CLAUDE.md`、`glances-v4-vs-v5.md` 中大量引用设计文档（MEMORY.md、architecture §x）说明 v5 重写由 AI 深度参与完成。
- 治理：`CONTRIBUTING.md` 规范 issue 用途（不支持个人支持请求、走 Google Group）；`.github/workflows/needs_contributor.yml` 自动标记需要贡献者的 issue；GitHub Sponsors 赞助（README 徽章）；bounty 机制未见证据（未确认）。
- 生态位：Docker Hub 官方镜像（`build_docker.yml`）、Snap 包、PyPI 月下载徽章；Grafana 仪表盘模板随仓库分发（`conf/glances-grafana-*.json`）。

## 9. 亮点与风险

亮点：
1. 架构教科书级的三层解耦：34 插件 → 无锁共享存储 → 四消费形态（TUI/Web/API/MCP），v5 重写有完整设计文档与迁移对照（`glances-v4-vs-v5.md`），且有量化性能数据（TUI 内存 -17%）。
2. 依赖分层克制：硬依赖仅 5 个（psutil 为主），19 个导出后端全部收进可选 extras，锁文件全 hash 钉版。
3. 四操作系统测试矩阵（含 FreeBSD QEMU）+ Playwright 浏览器级 WebUI 测试 + 导出后端真实接口测试，质量投入在同类 CLI 工具中属第一梯队。
4. 与时俱进的安全基线：2026-10 安全审计、CVE-2026-33641 修复、默认回环绑定、PBKDF2 60 万轮哈希——监控工具自带 Web 服务，这类加固直接决定其能否安全部署。
5. 积极拥抱新形态：MCP 服务器让 LLM 能查询系统状态，兼容 MCP SDK 1.x/2.x（`glances/outputs/glances_mcp.py`）。

风险/局限：
1. 代际切换期阵痛：develop 分支 5.0.0b2 API 与 v4 完全不兼容（口令哈希、阈值键名、REST 路径、模板全变），生态客户端需二选一迁移；`glances` 命令装的就是 beta（NEWS.rst 破坏性变更）。
2. v5 移除外部插件目录（`-P/--plugins`），第三方扩展只能进主干，插件生态开放性下降。
3. 单点维护风险仍高：头部贡献者一人占 5,628 次提交，251 名贡献者中绝大多数是小修补；v5 这种量级的重写实质依赖作者个人 + AI 协作。
4. 无锁读取的正确性论证依赖 CPython GIL 假设（`glances/stats_store.py` docstring 自认），若 GIL 移除需回补锁——self-aware 但属于长期技术债。

## 10. 总结评价

Glances 是"单人主导项目如何长期演进"的范本：2011 年创建、连续维护 15 年、约每月一个稳定版，并在第 14 年敢于推倒重写核心引擎（v5）而非躺在稳定 API 上吃老本。它的源码值得三类人阅读：想学 Python 插件式架构的人（泛型基类 + 约定式发现 + 声明式字段模式，每层都有可引用的设计文档）；想做跨平台系统工具的人（psutil 差异隔离、四平台 CI 矩阵）；以及想理解"工具如何同时服务人类（TUI/Web）与机器（API/导出/MCP）"的人。对使用者，它是服务器巡检与轻量监控的标准答案之一；对贡献者，当前正处于 v5 迁移窗口，是深度参与架构定型的最佳时机。
