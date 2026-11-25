# searxng/searxng 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：SearXNG 是一个免费开源的互联网元搜索引擎（metasearch engine），把数十个搜索服务与数据库的结果聚合到统一界面，用户不被追踪、不被画像（`README.rst`）。

| 项 | 值 | 出处 |
|---|---|---|
| 仓库 | searxng/searxng | GitHub |
| 星数 | 39500（快照） | 任务参数 |
| 主语言 | Python（358 个 .py，约 5.2 万行，`searx/` 内统计） | 源码统计 |
| 许可证 | AGPL-3.0-or-later | `LICENSE`、`setup.py` |
| 创建时间 | 2021-04-12（自 searx 项目分叉） | gh api + `CHANGELOG.rst` |
| 最近 push | 2026-11-13（分析前一天仍在合 PR） | gh api |
| Forks / 贡献者 | 3612 forks / 292 位贡献者 | gh api contributors 分页求和 |
| 发布模式 | 滚动发布，版本号基于 git 提交（PR #229 起），无传统 CHANGELOG | `CHANGELOG.rst` |
| 运行形态 | 自托管 Flask 应用 + Granian/uWSGI，公共实例池 searx.space | `docs/index.rst`、`container/Dockerfile` |

## 2. 项目是什么，解决什么问题

SearXNG 解决的核心问题是"搜索隐私与搜索多样性"。商业搜索引擎会记录用户的查询、构建画像；而 SearXNG 把用户的查询转发给多个搜索引擎，将返回结果在服务端去重、排序后统一呈现，上游搜索引擎看到的是自托管实例的 IP 而非用户 IP（`README.rst`、`docs/index.rst`）。项目自述"Users are neither tracked nor profiled"。

项目 2021 年年中从老项目 searx 分叉而来（`CHANGELOG.rst` 明确记载），此后采用滚动发布：master 分支即发布版，版本号由 git 提交生成，不存在里程碑式 release。这使其迭代极快——分析时最近 5 个提交均在两天内（gh api commits，2026-11-12/13），当前 PR 编号已到 #6943。

功能面上它远不止"网页搜索聚合"：settings.yml 中 353 个引擎条目覆盖 general/images/videos/news/it/music/packages/science/onions（Tor 暗网）等 20+ 分类（`searx/settings.yml` 统计），还内置计算器、单位换算、货币汇率、随机数/统计等"答案引擎"（`searx/answerers/`、`searx/engines/currency_convert.py`），以及天气数据模型（`searx/weather.py`）。查询支持 `!bang` 语法直接跳转到指定引擎（`searx/external_bang.py`、`searx/data/external_bangs.json`）。

生态位上，SearXNG 是自托管搜索事实上的标准：官方实例索引 searx.space 列出约 70 个"维护良好"的公共实例并有 uptime 监控（`docs/index.rst` 第 14、26 行），大量个人/组织自建私有实例供自己或小团体使用。它也常被其他项目当作"搜索 API 后端"集成（`/search` 支持输出 JSON/RSS 格式，`searx/webapp.py` route `format` 参数）。

与 Google 等源的反爬对抗是本项目最具工程特色的暗线：默认所有出站请求通过 curl_cffi 模拟 Chrome 的 TLS 指纹（`searx/network/client.py` 第 19 行 `DEFAULT_IMPERSONATE = "chrome"`）；Google 网页版因需要执行 JS 无法直接抓取，改用"Nokia 用户代理请求 XML 版面"的方案（`searx/engines/google.py` 模块 docstring，引用 issue #6359）。这些手段在代码与注释中均有客观记录。

## 3. 技术栈与依赖

| 依赖 | 版本 | 用途 |
|---|---|---|
| Flask / flask-babel / jinja2 | 3.1.3 / 4.0 / 3.1.6 | Web 框架、i18n、模板（`requirements.txt`） |
| lxml | 6.1.3 | HTML/XPath 解析——全部 scraping 类引擎的基础 |
| curl_cffi | 0.17.0 | 出站 HTTP 客户端，支持浏览器 TLS 指纹模拟（impersonate） |
| valkey | 6.1.1 | Redis 分支客户端：限流计数器、缓存（`searx/valkeydb.py`、`valkeylib.py`） |
| granian | 2.8.7 | Rust 实现的 HTTP 服务器，官方文档称将取代 uWSGI（`docs/admin/installation-granian.rst`） |
| msgspec | 0.22.0 | 强类型结果结构（`searx/result_types/`）与 EngineAbout |
| typer | 0.29.0 | 数据维护 CLI（`searx/data/__main__.py`） |
| whitenoise | 6.12.0 | 静态文件直挂 Flask |
| babel / python-dateutil / pyyaml / pygments / markdown-it-py / isodate | — | i18n、日期、配置、代码高亮、Markdown 信息页、时长解析 |

前端：`client/simple/` 是 TypeScript + Vite 的"simple"主题源码（biome 做 lint），构建产物进 `searx/static/`（`client/simple/package.json`、`vite.config.ts`）。仓库根部的 `go.mod` 仅用于开发工具链（shfmt），非运行时依赖（`go.mod` 第 3 行 `tool mvdan.cc/sh/v3/cmd/shfmt`）。

Python 版本：`setup.py` 声明 `python_requires>=3.10`，CI 矩阵测 3.11-3.14（`.github/workflows/integration.yml`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `searx/` | 主 Python 包（约 5.2 万行），全部服务端逻辑 |
| `searx/engines/` | 254 个引擎适配器文件（`ls searx/engines/*.py | wc -l`） |
| `searx/search/processors/` | 引擎执行器：online/offline/currency/dictionary/url_search 五类 |
| `searx/enginelib/` | Engine 基类、EngineTraits（语言映射）、EngineCache |
| `searx/network/` | curl_cffi 封装、连接池、代理、错误处理 |
| `searx/plugins/` | 插件体系与 13 个内置插件 |
| `searx/botdetection/` | 反 bot 检测（IP 限流、UA/Header 校验、link_token） |
| `searx/result_types/` | msgspec 强类型结果模型（image/video/paper/answer/code…） |
| `searx/templates/`、`searx/static/` | Jinja2 模板（simple 主题）与构建后静态资源 |
| `searx/translations/` | 61 个语言翻译（Weblate 维护） |
| `searx/data/` | 预生成数据：external_bangs.json、engine_traits.json、currencies.json、fasttext 语言模型 lid.176.ftz 等 |
| `client/simple/` | simple 主题 TS/Vite 前端源码 |
| `container/` | Dockerfile 与 compose（基于 searxng/base 构建镜像） |
| `docs/` | Sphinx 文档（admin/dev/user 三视角，发布到 docs.searxng.org） |
| `tests/` | unit/ 与 robot/（Selenium E2E） |
| `utils/` | `searxng.sh` 管理脚本与 Makefile 的函数库 |
| `.github/workflows/` | 7 个 CI 工作流：integration/container/documentation/data-update/l10n/sync/ai-policy |
| `searxng_extra/` | setup.py 等打包附属 |

组织评述：单体单包应用（single-package monolith），打包为一个 `searxng` PyPI 包（`setup.py`）。模块边界清晰：引擎适配器（engines）与执行框架（search/processors）、结果模型（result_types/results.py）、网络层（network）严格分层，是"适配器模式 + 管线式聚合"的教科书式布局。

## 5. 核心架构与关键实现

### 5.1 引擎适配器体系（254 个文件，353 个配置条目）

每个引擎是一个模块，实现 `request()` 与 `response()` 并声明元数据；统一由 `searx/enginelib/__init__.py` 的 `Engine` 基类（第 228 行）与 `EngineTraits`（`searx/enginelib/traits.py`，负责每个引擎的语言/地区参数映射）驱动。settings.yml 注册 353 条引擎（默认启用 123、默认禁用 230），分类统计：general 55、images 28、it 23、videos 23、news 20、web 16 等（`searx/settings.yml` 解析统计）。覆盖 Google（web/images/news/videos/scholar/play/cse 七个变体，`searx/engines/google*.py`）、Bing 四变体、DuckDuckGo 四文件、Wikipedia、Brave、百度、Qwant 等。

引擎按执行类型分五类处理器（`searx/search/processors/__init__.py` 第 24-28 行）：`online`（常规 HTTP 抓取）、`offline`（本地数据，如货币换算）、`online_currency`、`online_dictionary`、`online_url_search`。Google 网页引擎的对抗细节见 `searx/engines/google.py` 头注释："This implementation uses Nokia user agents to request an XML layout from Google. The normal web version requires executing JavaScript"。

### 5.2 并发聚合与去重管线

`searx/search/__init__.py` 是编排中枢：`_get_requests()` 为每个被选引擎生成请求参数并取所有引擎 timeout 的最大值，`search_multiple_requests()` 对每个引擎开独立线程并发请求（第 145-150 行 `threading.Thread` + `copy_current_request_context`），超时后 join 带超时回收。

去重与排序在 `searx/results.py`：`ResultContainer` 以 URL 为键归并结果，`merge_two_main_results()`（第 334 行）保留更长的 title/content、合并引擎来源集合、URL scheme 升级为 https；打分函数 `calculate_score()`（第 17 行）按 `weight / position` 累加——同一结果被越多引擎返回、排名越靠前，最终得分越高，这正是元搜索"多源印证"的核心算法。结果类型用 msgspec 结构强类型化（`searx/result_types/`：image、video、paper、answer、code、file、keyvalue）。

### 5.3 反 bot 限流与查询解析

`searx/limiter.py` + `searx/limiter.toml` + `searx/botdetection/`（11 个模块）构成对公共实例的防护：Header 合法性检查（http_accept、http_user_agent 等）、IP 滑动窗口限流（`valkeylib.py` 的 `incr_sliding_window`、Lua 脚本保证原子性）、link_token 挑战、可信代理解析。查询解析在 `searx/query.py` 的 `QueryPartParser` 体系：把 `!bang`（引擎选择）、`:category`（分类）、`timeout` 等从原始查询串剥离。

### 5.4 插件与主题

插件基类提供三个钩子（`searx/plugins/_core.py` 第 141/149/169 行）：`pre_search`、`on_result`、`post_search`，由 `PluginStorage` 按 settings 启用。内置 13 个插件（`searx/plugins/`）：tracker_url_remover（去跟踪参数）、hostnames（结果域名过滤/升降权）、ahmia_filter（暗网内容过滤）、calculator、unit_converter、tor_check、oa_doi_rewrite 等。主题只剩 simple 一个（`searx/templates/simple/`，旧主题已移除），前端源码在 `client/simple/`（TS+Vite），另有 favicon 代理子系统（`searx/favicons/`，5 种解析器）。

另有两处值得单独指出的细节：一是 `searx/data/engine_traits.json` 集中存放全部引擎的语言映射数据，由 data-update CI 定时从上游页面重新抓取生成，使"语言支持漂移"问题被工程化消解；二是 `searx/metrics/` + `searx/openmetrics.py` 为每个引擎记录成功/失败/耗时计数并暴露在 `/metrics`，运维可据此发现"哪个引擎又坏了"，是对抗性维护不可或缺的可观测层。

## 6. 构建与运行

```bash
# 完整工具链（Makefile，实为 utils/searxng.sh 封装）
make all          # clean + install（Makefile 第 4 行）
make run          # 安装并启动开发实例（0.0.0.0:8888）
make test         # yamllint black pyright pylint unit robot rst shell shfmt
make docs         # Sphinx HTML 文档
make themes       # 构建 simple 主题（client/simple Vite 构建）
make container    # 构建 Docker 镜像

# 生产运行：容器（推荐，Granian 服务器）
docker compose -f container/docker-compose.yml up   # container/docker-compose.yml
# searxng-run = searx.webapp:run （setup.py console_scripts 入口）
```

配置：`searx/settings.yml`（默认值）+ `/etc/searxng/settings.yml`（部署覆盖），加载逻辑在 `searx/settings_loader.py`，默认值 schema 在 `searx/settings_defaults.py`。

## 7. 测试与工程质量

- 测试框架 nose2 + cov-core（`requirements-dev.txt`）；单测 `tests/unit/` 约 30 个文件，含按引擎写的测试（`test_engine_brave.py`、`test_engine_github_code.py` 等）与 processors/network/settings 分目录测试；E2E 用 Selenium + splinter 的 `tests/robot/test_webapp.py`。
- CI 七个工作流（`.github/workflows/`）：integration（Python 3.11-3.14 矩阵）、container（镜像构建）、documentation、data-update（定时更新 engine_traits/currencies 等数据文件）、l10n（Weblate 同步）、sync、ai-policy。
- 代码质量：black + pylint + basedpyright（`pyrightconfig.json`）+ yamllint + shfmt 全套强制；类型注解覆盖度高（源码中大量 `t.TYPE_CHECKING`、TypedDict）。
- 观察：引擎测试覆盖不均匀（254 个引擎文件 vs 少量 per-engine 测试），但 data-update 工作流自动刷新 traits 数据弥补了引擎漂移问题。

## 8. 社区与维护状态

- 活跃度极高：gh api 显示最近 push 2026-11-13（分析前一天），最近 5 个提交均出自 11-12/13，PR 编号 #6930-#6943 连续合入；open issues 214。
- 贡献者：292 人（gh api contributors 分页求和）；forks 3612；项目 2021-04 创建，五年持续演进。
- 治理特色（`AI_POLICY.rst`）：明文限制生成式 AI 贡献——必须披露 AI 使用、人类须完全理解每行代码、"AI 永远不能是 PR 主要作者"、AI slop 贡献者会被封禁；称低质 AI 贡献为"extractive contribution"。这在同类项目中相当少见。
- 翻译走 Weblate（translate.codeberg.org，`CONTRIBUTING.rst`），61 种语言（`searx/translations/`）；社区主渠道 Matrix（#searxng:matrix.org）；仓库托管在 GitHub 但翻译与组织基础设施部分依托 Codeberg 生态。
- 发布模式为滚动发布，无版本化 release（`CHANGELOG.rst`），运维通过 Docker 镜像标签跟踪。

## 9. 亮点与风险

**亮点**
1. 超大规模适配器体系：254 个引擎适配器 + 五类处理器抽象，新增一个搜索源只需实现一个模块文件（`searx/engines/`、`searx/search/processors/`）。
2. 真实可查的反爬工程：curl_cffi 默认 Chrome TLS 指纹模拟、Google XML 版面抓取、botdetection 限流三件套，全部有代码与注释佐证（`searx/network/client.py`、`searx/engines/google.py`、`searx/botdetection/`）。
3. 服务端聚合保护用户隐私，且 `/search` 支持 JSON/RSS 输出，可被第三方当搜索 API 消费（`searx/webapp.py`）。
4. 极致活跃的滚动发布与自动化数据维护（data-update CI 每日刷新 traits/汇率），五年无断档。
5. 明确的 AI 贡献政策与成熟的多语言/文档/实例生态（AI_POLICY.rst、61 语言、searx.space 约 70 实例）。

**风险/局限**
1. 对抗性依赖：Google/Bing/DDG 页面结构或反爬策略一变即坏，修复靠社区快速响应（近期提交多为引擎修复，如 `[fix] ddg: skip web-result items without a link` #6930），属于永久性维护负担。
2. 滚动发布无稳定版本点，部署方必须跟随 master，回退与审计不便（`CHANGELOG.rst`）。
3. 并发模型为"每引擎一线程"（`searx/search/__init__.py`），高 QPS 公共实例下线程数可观，需限流与缓存兜底。
4. 自托管门槛：需理解反向代理、Valkey/Redis、limiter 配置才能真正安全公开（`docs/own-instance.rst`、`searx/limiter.toml`）。

## 10. 总结评价

SearXNG 是"元搜索"品类中规模与活跃度俱佳的旗舰开源实现：以 5.2 万行 Python 聚合 250+ 搜索源，用 TLS 指纹模拟等务实手段维持对商业引擎的可用性，同时以 AGPL + 滚动发布 + 严格 AI 政策维持社区纪律。它既是可直接部署的生产级隐私搜索引擎，也是学习"适配器模式 + 并发聚合 + 反爬工程 + 社区治理"的高质量范本。适合关心隐私的用户自建实例、适合后端工程师阅读其引擎适配与结果归并管线（`searx/search/`、`searx/results.py`），也适合想接入中立搜索 API 的开发者；但不适合追求稳定版本号或不愿承担引擎失效维护成本的生产环境。
