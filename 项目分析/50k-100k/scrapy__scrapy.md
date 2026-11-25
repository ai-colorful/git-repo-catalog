# scrapy/scrapy 项目分析

> 星数 64973（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：Python 生态最老牌、最完整的声明式网络爬虫框架，用"引擎 + 调度器 + 下载器 + 中间件管线"的组件化架构把抓取流程标准化。

| 项目 | 事实 |
| --- | --- |
| 名称 | scrapy/scrapy（包名 Scrapy） |
| 版本 | 2.19.0（2026-09-10 发布，HEAD 为下一版本开发中，`scrapy/VERSION`） |
| 星数 | 64,973（快照）；gh api 实测 65,122（2026-10-31） |
| 语言 | Python，要求 >=3.11（classifier 到 3.15，`pyproject.toml`） |
| 许可证 | BSD-3-Clause（`pyproject.toml` license 字段） |
| 创建时间 | 2010-02-22（gh api `created_at`） |
| 最近活跃 | 最近 push 2026-10-30（gh api），活跃度极高 |
| 维护方 | Zyte（原 Scrapinghub）+ 社区（`README.rst`） |
| 规模 | 主包 199 个 .py 约 3.79 万行；tests + tests_typing 约 7.05 万行（实测统计） |

## 2. 项目是什么，解决什么问题

Scrapy 是"从网站提取结构化数据的爬虫框架"（`README.rst`）。它要解决的问题是：写爬虫时真正与业务相关的只有"从哪个 URL 开始、如何解析响应、数据长什么样"，而并发控制、去重、调度优先级、重试、限速、robots.txt、Cookie、代理、压缩解压、编码探测、结果导出等通用工程问题应该由框架统一承担。

框架把一次抓取拆成角色清晰的组件（`docs/topics/architecture.rst` 的数据流描述）：Spider 产出初始 Request → Engine 存入 Scheduler → Scheduler 按优先级取出 Request 交给 Downloader → 响应经 Spider 中间件回到 Spider 回调 → 产出的 Item 经 Item Pipeline 落盘/导出，新 Request 回到 Scheduler 循环。用户只需实现 Spider（`async def start()` 产出请求、回调产出 Item/Request），其余全部可插拔定制。

用户侧的核心抽象是三层：Spider 定义抓什么和怎么解析；中间件（下载器中间件 15 个内置模块 `scrapy/downloadermiddlewares/`、Spider 中间件 9 个 `scrapy/spidermiddlewares/`）做横切处理；Item Pipeline（`scrapy/pipelines/`）做清洗、验证、存储。`scrapy/extensions/` 提供 17 个扩展（自动节流 AutoThrottle、FeedExporter 结果导出、HTTP 缓存、内存监控、telnet 远程控制等）。

项目当前处在 Twisted → asyncio 的迁移后期：`TWISTED_REACTOR` 默认已是 `AsyncioSelectorReactor`（`scrapy/settings/default_settings.py:663`），2.15.0 起还支持 `TWISTED_REACTOR_ENABLED=False` 的"无 reactor"纯 asyncio 模式（`scrapy/utils/reactorless.py`）。HEAD 上旧 Deferred 风格 API 正在系统性替换为 async API（如 `ExecutionEngine.download()` 弃用、改用 `download_async()`，`scrapy/core/engine.py:661-670`），甚至新增了对 `httpx2`/`aiohttp` 这类纯 asyncio HTTP 栈的下载处理器（`scrapy/core/downloader/handlers/_httpx.py`、`_aiohttp.py`）。

## 3. 技术栈与依赖

来源：`pyproject.toml`。

- 语言与运行时：Python 3.11+；构建后端 hatchling；入口脚本 `scrapy = scrapy.cmdline:execute`。
- 事件循环：Twisted>=23.8.0（默认跑在 AsyncioSelectorReactor 上）；可选 uvloop extra；测试默认 `--reactor=asyncio`（`pyproject.toml` [tool.pytest.ini_options]）。
- 网络：aiohttp>=3.13.3（必装依赖）、aiodns、aioboto3/boto3（S3）、aioftp（FTP）；可选 `httpx2[http2,socks]` extra。默认 HTTP 下载处理器仍是 Twisted 的 http11（`DOWNLOAD_HANDLERS_BASE`，`scrapy/settings/default_settings.py:321`）。
- 解析：lxml、parsel、cssselect、defusedxml、w3lib、tldextract。
- 数据模型：itemadapter、itemloaders（跨 item 类型适配，支持 dict/dataclass/attrs/pydantic）。
- 基础设施：queuelib（调度队列）、protego（robots.txt）、PyDispatcher（信号）、zope.interface（接口）、service_identity/pyOpenSSL/cryptography（TLS）、`formerly`（新引入的弃用声明辅助库，`scrapy/utils/deprecate.py:10`）。
- 质量工具链：mypy strict（含 twisted 豁免配置）、ruff（约 40 组规则集）、pylint、pre-commit、coverage（branch 模式）、CodSpeed 基准测试。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `scrapy/core/` | 引擎核心：engine.py（1075 行）、scheduler.py、scraper.py、spidermw.py、downloader/ |
| `scrapy/core/downloader/` | 下载器本体（并发 Slot）、TLS 上下文、按协议的 handlers/（http11、http2、_aiohttp、_httpx、s3、ftp、datauri、file） |
| `scrapy/downloadermiddlewares/` | 15 个内置下载器中间件（robotstxt、retry、redirect、cookies、httpproxy、httpcache、offsite 等） |
| `scrapy/spidermiddlewares/` | 9 个内置 Spider 中间件（depth、referer、httperror、urllength、start、stickymeta 等） |
| `scrapy/extensions/` | 17 个扩展（feedexport、httpcache、AutoThrottle、memusage、logstats、remote_control 等） |
| `scrapy/pipelines/` | ItemPipelineManager + 媒体管线（files/images/media） |
| `scrapy/settings/` | 设置系统，default_settings.py 约 670 行全量默认值 |
| `scrapy/spiders/`、`scrapy/commands/`、`scrapy/selector/`、`scrapy/loader/` | Spider 基类、CLI 子命令（17 个）、CSS/XPath 选择器封装、ItemLoader |
| `scrapy/addons.py`、`scrapy/signalmanager.py`、`scrapy/dupefilters.py`、`scrapy/pqueues.py` | 加载器式组件：Addons、信号、去重过滤、优先级队列 |
| `docs/` | 大而全的 Sphinx 文档（含 architecture/asyncio/item-pipeline 等专题）；`NEWS` 指向 `docs/news.rst` |
| `tests/`、`tests_typing/` | pytest 测试与 mypy 类型测试 |
| `sep/` | Scrapy Enhancement Proposals（SEP-001 至 015），设计决策记录 |
| `extras/`、`.github/workflows/` | 发布脚本、CI 工作流（ci、publish、codspeed 等 8 个） |

组织方式评述：单包单体（single package monolith），内部按"框架组件类型"分目录，与文档 architecture 图一一对应；没有子包拆分发版，靠 hatchling sdist include 列表控制打包范围（`pyproject.toml`）。`AGENTS.md` 专门写了给 AI 编码 agent 的贡献注意事项，是仓库里少见的现代细节。

## 5. 核心架构与关键实现

### 5.1 ExecutionEngine：带显式状态机的主循环（`scrapy/core/engine.py`）

引擎是全框架唯一 orchestrator。HEAD 用枚举状态机取代了历史上散落的布尔标志：`EngineState` 定义 CREATED → SPIDER_OPENING → SPIDER_OPEN → STARTING → RUNNING → SPIDER_CLOSING → STOPPING → STOPPED 八个状态及合法迁移表（`engine.py:84-139`），`_transition_to()` 做非法迁移防御。调度主循环：

```python
def _start_scheduled_requests(self) -> None:
    if self._state is not EngineState.RUNNING or self.paused:
        return
    while not self.needs_backout():
        if not self._start_scheduled_request():
            break
    if self.spider_is_idle() and self._slot.close_if_idle:
        self._spider_idle()
```

`_start_scheduled_request()` 从 scheduler 取一个 Request，`_download()` 后用 Deferred 链把响应交给 `_handle_downloader_output` → Scraper，并在 finally 中重新调度 nextcall 实现事件驱动（`engine.py:528-566`）。背压由 `needs_backout()` 统一判断：引擎暂停、正在关闭、或 `downloader.needs_backout()`（并发已满）即停（`engine.py:510-522`）。API 面正在异步化：`start_async`/`stop_async`/`download_async`/`open_spider_async` 均为非弃用入口，旧 Deferred 方法仅转发并抛 `ScrapyDeprecationWarning`。

### 5.2 Scheduler 与去重（`scrapy/core/scheduler.py`、`scrapy/dupefilters.py`、`scrapy/pqueues.py`）

`BaseScheduler` 用元类 `BaseSchedulerMeta` 做鸭子类型接口校验（必须实现 `has_pending_requests`/`enqueue_request`/`next_request`，`scheduler.py:24-40`），这是把"内存队列"换成 Redis 等分布式队列的官方扩展点。默认 `Scheduler` 组合三件事：优先级队列（内存 `ScrapyPriorityQueue` 或按 `JOBS_DIR` 落盘的 queuelib 磁盘队列，支持断点续爬）、`RFPDupeFilter`（指纹持久化到文件）、可选的 `DownloaderAwarePriorityQueue`（按下载槽位忙闲调整出队顺序，`pqueues.py:329`）。HEAD 新增 `supports_skip_dupefilter_once` 协议：允许单次跳过去重而不把 `dont_filter` 传染给派生请求（`engine.py:69-81`）。

### 5.3 Downloader：并发 Slot + 协议 handlers + 中间件链（`scrapy/core/downloader/__init__.py`）

下载器按"总并发 + 每域名/IP 并发"双维度限流：`total_concurrency = CONCURRENT_REQUESTS`（默认 16），`needs_backout()` 在 `len(self.active) >= total_concurrency` 时返回 True（`__init__.py:287-289`）。Scheme 分发到 `DOWNLOAD_HANDLERS_BASE` 注册的 handler，默认 http/https 走 Twisted http11，同时仓库内已内置 `_aiohttp.py`、`_httpx.py`、`_aioftp.py` 三个 asyncio 原生实现（`scrapy/core/downloader/handlers/`），迁移路径清晰。中间件链由通用 `MiddlewareManager` 基类驱动（`scrapy/middleware.py:32`），用 `_process_chain` 把 `process_request`/`process_response`/`process_exception` 串成 async 链；内置 15 个中间件按 50-850 的优先级排布（`default_settings.py:351`）。

### 5.4 Scraper 与 Item Pipeline（`scrapy/core/scraper.py`、`scrapy/pipelines/__init__.py`)

Scraper 是引擎与 Spider 之间的缓冲带：响应进入有界队列（`RESPONSE_MAX_ACTIVE_SIZE = 5_000_000` 字节，`default_settings.py:611`），超限即背压，防止 Spider 消费速度慢于下载速度导致内存膨胀（`scraper.py:131` 附近的 `needs_backout`）。Item 侧 `ItemPipelineManager` 继承 `MiddlewareManager`，按 `ITEM_PIPELINES` 优先级链式调用 `process_item`，HEAD 已把非弃用入口改为 `process_item_async`（`pipelines/__init__.py:48-73`）。`ExtensionManager`（`scrapy/extension.py:12`）同样基于 MiddlewareManager，扩展在 `EXTENSIONS_BASE` 默认启用 12 个（AutoThrottle、FeedExporter、CoreStats 等）。2.13 引入的 `AddonManager`（`scrapy/addons.py`）让"改设置"也变成可分发的组件（`ADDONS` 设置项）。

## 6. 构建与运行

来源：`pyproject.toml`、`tox.ini`、`AGENTS.md`、`README.rst`。

```bash
pip install scrapy                 # 用户安装
# 开发（未运行验证，仅摘自项目文件）
tox -e py                          # 全量 pytest（tox.ini envlist）
tox -e py -- tests/test_foo.py     # 单文件（AGENTS.md 推荐写法）
tox -e mypy && tox -e pylint       # 类型与静态检查
tox -e docs / docs-tests           # 文档构建与文档测试
pre-commit run --all-files         # ruff/pylint 等钩子
pytest --reactor=asyncio           # 测试默认 asyncio reactor（pyproject addopts）
```

CLI 日常使用（`scrapy/commands/` 17 个子命令）：`scrapy startproject / genspider / crawl / shell / fetch / parse`。

## 7. 测试与工程质量

- 规模：tests/ 下 176 个 test_*.py 共约 7.0 万行，另含 tests_typing/ 3 个 mypy-testing 文件；自带 `tests/mockserver/` 假服务器、mitmproxy 集成（pytest marker `requires_mitmproxy`）、`requires_internet` 标记隔离真实网络测试（`pyproject.toml` markers）。
- 风格：pytest + pytest-twisted，`xfail_strict = true`，warnings 全部按 error 过滤（filterwarnings 配置达 15 条，说明对告警零容忍）；支持 asyncio/none（reactorless）双模式跑测试。
- CI：`.github/workflows/ci.yml` 矩阵覆盖 Python 3.11-3.15、ubuntu/windows、tox 的 pinned 旧依赖环境（`min-extra-deps` 等防止最低版本回归）、CodSpeed 基准（codspeed.yml/codspeed-walltime.yml）、Codecov 上报、dependabot。
- 工程质量观察：mypy strict + ruff + pylint 三重静态检查；`AGENTS.md` 要求贡献者同时跑 pre-commit/mypy/pylint/docs 四道门；兼容性处理细致（Python 3.14 zstd backport、Twisted socket 泄漏 workaround 等见 filterwarnings 注释）。

## 8. 社区与维护状态

- 活跃度：最近 push 2026-10-30（gh api）；open issues 185、forks 12,081（2026-10-31 实测）。
- Release 节奏：近一年 2.14（2026-01）→ 2.15（2026-04）→ 2.16（2026-05）→ 2.17（2026-07）→ 2.18（2026-08）→ 2.19（2026-09）（`docs/news.rst` + gh api releases），约 1-2 月一个 minor，稳定输出。`.github/workflows/publish.yml` 自动化发版。
- 贡献者结构：商业公司 Zyte 主导维护（README 明示），`AUTHORS` 文件收录贡献者；有 CODE_OF_CONDUCT、SECURITY.md、issue/PR 模板，另有 `close-spam-prs.yml` 自动处理垃圾 PR——社区治理成熟。
- 设计治理：`sep/` 目录维护 15 篇 Scrapy Enhancement Proposals，重大架构变更（如 asyncio 迁移）有书面 RFC 流程。

## 9. 亮点与风险

亮点：
1. 架构正交性极强：Engine/Scheduler/Downloader/Spider 回调/Item Pipeline 五段各自可替换，`BaseScheduler` 元类接口、下载 handler 按协议注册等扩展点设计清晰（`scrapy/core/scheduler.py`、`default_settings.py:321`）。
2. Twisted → asyncio 迁移策略稳健：双栈长期并存（`TWISTED_REACTOR_ENABLED`、`is_asyncio_available()`、reactorless 模式），async API 与弃用警告平滑过渡，且已备好 aiohttp/httpx 原生 handler。
3. 工程化天花板：7 万行测试对 3.8 万行源码、mypy strict、CodSpeed 基准、最低依赖版本 pin 测试，在 Python 框架中属第一梯队。
4. 生产级运维能力内建：断点续爬（JOBS_DIR）、AutoThrottle 自适应限速、HTTP 缓存、内存/泄漏监控、feed 导出到 S3/GCS/FTP（`scrapy/extensions/`）。
5. 文档与治理：Sphinx 文档覆盖每个组件与设置项，SEP 流程留档重大决策。

风险/局限：
1. 历史包袱重：弃用路径极长，同一能力常有 sync/async、旧/新两套 API 并存（`engine.py` 中 `download` vs `download_async` 等），初学者和 AI 生成的代码容易踩到 deprecated 路径。
2. 对 Twisted 的耦合仍是现状：默认 http/https handler、Deferred 链（`_start_scheduled_request` 的 addBoth 链）依然 Twisted 风格，纯 asyncio 用户尚需切换 handler 才能摆脱。
3. 单进程架构：横向扩展依赖外部方案（Scrapy Cloud、scrapy-redis 等第三方），核心框架不内置分布式调度。
4. 重型依赖树（lxml/Twisted/cryptography 等 C 扩展），嵌入式或纯 Python 环境安装成本偏高。

## 10. 总结评价

Scrapy 是爬虫领域"框架"这一形态的定义者：16 年持续演进（2010 至今），把抓取流水线的每个环节都做成了可插拔组件，又以极高的工程纪律（70k 行测试、strict 类型、SEP 流程）支撑了从 Twisted 到 asyncio 的整代迁移。它不是最轻的选择——学习曲线和依赖树都不小——但在需要规模化、稳定、可运维的抓取系统时仍是 Python 的事实标准。适合需要工业级并发控制与扩展点的爬虫工程师使用；源码方面，`scrapy/core/engine.py` 的状态机与调度循环、`scrapy/middleware.py` 的组件链抽象是理解"事件驱动爬虫框架如何分层"的最佳教材。
