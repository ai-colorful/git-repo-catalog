# dgtlmoon/changedetection.io 项目分析

> 星数 39500（快照 2026-10-24；gh api 实测 2026-11-21 为 35932）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：自托管的网页变更监控服务——给任意网页设一个"Watch"，按计划重新抓取并 diff，发现变化后通过 Apprise 支持的 100+ 渠道（Email/Telegram/Discord/Slack/Webhook/MQTT 等）发通知，并附价格/库存监控、JS 渲染抓取、AI 摘要与条件过滤等能力。

| 项目 | 事实 |
|---|---|
| 名称 | changedetection.io（PyPI 包名即 `changedetection.io`，见 `setup.py`） |
| 星数 | 快照 39500 / gh api 实测 35932（2026-11-21），fork 2265 |
| 主语言 | Python（3.11+，`runtime.txt` 为 python-3.11.5），少量前端 JS |
| 许可证 | Apache-2.0（`LICENSE`） |
| 最近活跃 | gh api `pushed_at` 2026-11-20（前一天仍在提交） |
| 版本 | 0.70.4（`changedetectionio/__init__.py`，2026-11-19 发布） |
| 交付形态 | Docker 镜像（ghcr.io/dgtlmoon/changedetection.io）为主，PyPI/源码为辅（`README-pip.md`） |

## 2. 项目是什么，解决什么问题

changedetection.io 解决"网页内容变了但我不知道"的问题。用户在 Web UI 里添加一个 URL，服务端按设定的时间表重新抓取页面，与上次快照做差分；可配置的变化（正文改动、价格变动、商品补货、关键词出现/消失、JSON 字段变化）会触发通知。核心场景是价格跟踪与补货提醒（README 的 restock/price 专题，`docs/price-tracking.jpeg`）、页面内容编辑监控、以及"页面出现某词就告诉我"的条件触发。

它是典型的"自托管优先"产品：Docker 一行命令起服务，数据全部留在本地（每条 Watch 一个目录、JSON 快照落盘）；同时作者通过同名的托管订阅服务（$8.99/月，README 首屏推广）变现，这是它作为单人项目能长期高强度维护的商业基础。

相比"通知我页面变了"的朴素定位，它的差异化在于：① 抓取端支持纯 HTTP 与真实浏览器（Playwright）双通道，覆盖 JS 渲染页、可执行登录/点击等交互步骤（`browser_steps/`）；② 过滤端提供 CSS/XPath 选择器、JSON/jq 路径、忽略词/触发词/正则提取等多层规则，把"整页变了"收敛为"关心的部分变了"；③ 2025-2026 年新加的 LLM 层（`llm/`）让用户用自然语言声明意图（如"价格低于 50 才通知"），由 LiteLLM 调用各家模型对 diff 做判定与摘要，还支持 Ollama/vLLM 本地模型；④ 新增 MCP 端点（`blueprint/mcp/`）让 Claude/OpenAI 客户端把监控数据当资源查询。

## 3. 技术栈与依赖

- **Web 框架**：Flask 3.x 全家桶——flask-login（认证）、flask_wtf（CSRF/表单）、flask_restful（REST API）、flask-cors（Chrome 扩展跨域）、Flask-Babel + 自带 `flask_babel_backport.py`（i18n）、flask-compress、flask-paginate（`requirements.txt`）。
- **实时推送**：flask-socketio（threading 模式，eventlet 已被移除，可选 gevent；`realtime/`、`__init__.py` 注释）。
- **抓取**：requests[socks]（基础通道）；Playwright 1.56（Dockerfile 构建期安装，异步 API）；pyppeteer-ng + pyppeteerstealth 与 Selenium 作为备选 fetcher（`content_fetchers/`）。
- **解析/diff**：beautifulsoup4 + lxml（CSS/XPath 过滤）、elementpath（XPath 2.0-3.1）、inscriptis（HTML→文本，锁 2.7.6 防空白 diff）、diff_match_patch（词级 diff）、extruct + price-parser（LD+JSON 价格/库存元数据）、feedparser/feedgen（RSS）、jsonpath-ng、jq。
- **通知**：apprise==2.2.0（统一 100+ 通知渠道）+ paho-mqtt。
- **AI**：litellm（多模型路由）、rank-bm25（大快照 BM25 裁剪）、pydantic（LLMSettings）。
- **规则/插件**：panzi-json-logic（条件规则）、pluggy（插件系统）。
- **存储**：纯文件 JSON + orjson 快速序列化（`store/__init__.py`），无数据库依赖；快照超 20KB 时 Brotli 压缩（`model/Watch.py` 的 `BROTLI_COMPRESS_SIZE_THRESHOLD`）。
- **工程质量链**：pytest/pytest-flask/pytest-xdist、ruff + pre-commit、CodeQL、第三方许可证 CI（`.github/workflows/`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `changedetectionio/` | 单一主包，全部服务端代码（Flask 应用、模型、引擎、API、模板、静态资源、翻译、测试都在包内） |
| `changedetectionio/model/` | 领域模型：Watch.py（1696 行）、Tag/Tags/App、persistence.py（原子写盘 mixin）、PYDANTIC_MIGRATION.md |
| `changedetectionio/processors/` | 变更检测处理器插件：text_json_diff（默认）、restock_diff（单品价格/库存）、image_ssim_diff（截图像素 diff） |
| `changedetectionio/content_fetchers/` | 抓取通道：requests.py、playwright.py、puppeteer.py、webdriver_selenium.py、har_recorder.py、screenshot_handler.py |
| `changedetectionio/diff/` | diff 渲染层：diff_match_patch 词级 diff、tokenizers、通知用的高亮样式与占位符 |
| `changedetectionio/worker.py` + `worker_pool.py` | 异步检查 worker（每 worker 一线程一事件循环）与 UUID 认领/缩放管理 |
| `changedetectionio/notification/` | Apprise 通知封装、Jinja2 模板渲染、邮件样式、日志脱敏 |
| `changedetectionio/llm/` | LLM 意图评估与摘要：client（隔离子进程调用）、evaluator、prompt_builder、bm25_trim |
| `changedetectionio/blueprint/` | Flask 蓝图按功能切分：ui、watchlist、settings、tags、rss、pwa、backups、browser_steps、mcp（含 OAuth）、price_data_follower 等 |
| `changedetectionio/api/` | flask_restful REST API（Watch/Tags/Import/Search/SystemInfo 等），`docs/api-spec.yaml` 为 OpenAPI 规范 |
| `changedetectionio/conditions/` | json-logic 条件规则引擎 + levenshtein/wordcount 内置插件 |
| `changedetectionio/store/` | ChangeDetectionStore：文件型数据存储（每 Watch 一个 UUID 目录） |
| `changedetectionio/tests/` | 227 个测试文件（2.5MB），随包分发 |
| `docs/` | 图片素材、api-spec.yaml、api_v1 文档 |
| 顶层 | Dockerfile（多阶段，含 playwright/opencv 条件安装）、docker-compose.yml（可选 sockpuppetbrowser/selenium 浏览器容器）、setup.py、MANIFEST.in |

组织方式评述：单体 Python 包 + 插件化内部架构。处理器（processor）、抓取器（fetcher）、条件插件（conditions）、通用插件（pluggy hook）四类扩展点都通过注册表/插件管理器接入（`processors/__init__.py`、`pluggy_interface.py`），新类型监控（如 image_ssim_diff）可以整目录加入而不动主干。模型层用 dict 继承（`model/__init__.py` 的 `watch_base(dict)`），作者在 `model/PYDANTIC_MIGRATION.md` 和 Watch.py 模块头注释里明确承认这是技术债，迁移 Pydantic 是"梦想架构"。

## 5. 核心架构与关键实现

### 5.1 检查流水线：预检跳过 → 预处理 → 过滤 → 文本化 → diff（`processors/text_json_diff/processor.py`，716 行）

`run_changedetection()` 是核心入口。第一步用"双哈希"跳过昂贵工作：原始文档 MD5 + 有效过滤配置哈希（合并 watch/标签/全局三层规则后取 `get_filter_config_hash()`），两者都没变且 watch 未被编辑就直接抛 `checksumFromPreviousCheckWasTheSame`（源码 493-560 行区间）。这样全局规则变更也会正确失效缓存，而不是只看页面是否变化：

```python
if (not force_reprocess and
    not watch.was_edited and
    self.last_raw_content_checksum and
    self.last_raw_content_checksum == current_raw_document_checksum and
    watch.get('last_filter_config_hash') and
    watch.get('last_filter_config_hash') == current_filter_config_hash):
    raise checksumFromPreviousCheckWasTheSame()
```

之后依次：`guess_stream_type` 识别 HTML/JSON/PDF/RSS → `preprocess()`（RSS 转 HTML 阅读模式、PDF 用 pdftotext 转换、JSON 排版美化、混淆 workaround）→ `filter_to_text()`（先减性选择器去掉广告/导航，再 include 过滤器支持 CSS/XPath/`json:`/`jq:` 前缀，最后 inscriptis HTML→文本）→ 触发词/忽略词/正则提取规则 → 与历史快照 diff。过滤规则按 watch → tag → global 三层合并去重（`FilterConfig._get_merged_rules`），是配置继承的核心。

### 5.2 diff 渲染（`diff/__init__.py`，521 行）

行级差异用 difflib，行内词级差异用 diff_match_patch 的 `diff_linesToChars` 把词元当字符做 diff，配合 `tokenizers/` 的 `words_and_html` 分词器。产出用"占位符"中转（`@removed_PLACEMARKER_OPEN` 等）而非直接嵌 HTML，因为 Gmail/Outlook 会剥掉 `<style>` 和 `<ins>/<del>` 标签（文件头注释），最终由 `notification/handler.py` 的 `apply_service_tweaks()` 按通知渠道替换为内联样式的 `<span>`：

```python
REMOVED_STYLE = "background-color: #fadad7; color: #b30000;"
ADDED_STYLE = "background-color: #eaf2c2; color: #406619;"
# 行内更细粒度的"行内变化"高亮
REMOVED_INNER_STYLE = "background-color: #ff867a; color: #111;"
```

同时提供 `extract_changed_from/to()` 抽出旧值/新值（如旧价格→新价格），供通知模板 `{{diff_changed_to}}` 使用。

### 5.3 异步 worker 池（`worker_pool.py` 622 行 + `worker.py` 852 行）

每个 worker 是"一个原生线程 + 独立 asyncio 事件循环"，在循环里消费优先级队列（`queue_handlers.py` 的 RecheckPriorityQueue，调度器按 watch 的时间表投递）。关键设计：取到任务后立刻 `claim_uuid_for_processing()` 认领 UUID 防止重复检查（竞态有注释专门说明）；重复投递的任务若 `last_checked >= scheduled_at` 则直接丢弃；worker 有 `WORKER_MAX_JOBS`/`WORKER_MAX_RUNTIME` 自愈式重启策略，防内存泄漏与僵尸状态：

```python
max_jobs = int(os.getenv("WORKER_MAX_JOBS", "10"))
max_runtime_seconds = int(os.getenv("WORKER_MAX_RUNTIME", "3600"))
...
uuid = queued_item_data.item.get('uuid')
if not worker_pool.claim_uuid_for_processing(uuid, worker_id):
    # Already being processed - re-queue and continue
    deferred_priority = max(1000, queued_item_data.priority * 10)
```

worker 数默认 10（`FETCH_WORKERS` 环境变量可覆盖 `flask_app.py:1586`），queue 取任务走 ThreadPoolExecutor 阻塞等待避免轮询。通知走独立队列，由 `notification_service.py` 统一消费。

### 5.4 抓取通道与 Playwright 集成（`content_fetchers/`）

四类 fetcher 共享 `base.py` 的 Fetcher 抽象（截图、xpath_data、browser steps 接口）。`playwright.py`（545 行）是功能最强的通道：整页截图按 `SCREENSHOT_SIZE_STITCH_THRESHOLD` 分块滚动拼接（防 GPU 溢出、限总高）、注入 `res/` 里的 JS 做 Visual Selector 的 XPath 元素数据采集、库存状态 JS 探测、favicon 提取；支持通过 `PLAYWRIGHT_DRIVER_URL` 连接外部 sockpuppetbrowser 容器（`docker-compose.yml` 提供现成编排），也支持 per-watch 代理。Browser Steps（`browser_steps/` + `model/browser_steps.py`）允许在检查前执行"填输入框→点按钮"序列以处理登录/翻页。图片型监控由 `processors/image_ssim_diff/` 用 OpenCV（可选依赖，ARM 上跳过）做像素 diff。

### 5.5 价格/库存监控（`processors/restock_diff/`）

针对单品页的专用处理器：用 extruct 解析 schema.org/LD+JSON 元数据抽价格与 availability（`get_itemprop_availability`），大页内存优化用"纯 Python 提取优先 + 子进程跑 extruct 兜底"的混合模式（`extract_itemprop_availability_safe`），价格去重归一（`_deduplicate_prices` 处理 `$159/159/159.00` 多形态），支持上限/下限/百分比变化通知条件，价格历史可出趋势图（`blueprint/price_data_follower/`）。这是它区别于通用 diff 工具的最大特色场景。

### 5.6 LLM 意图层与 MCP（`llm/`、`blueprint/mcp/`）

`llm/evaluator.py` 对每次检测出的 diff 按用户声明的意图（自然语言）做"是否值得通知"的判定与摘要，经 `llm/client.py` 的 `completion_isolated()` 在子进程里调用 litellm（隔离网络/日志副作用），超长快照先用 `bm25_trim.py`（rank-bm25）按相关性裁剪。MCP 蓝图实现 `/mcp` 服务端（`mcp-types` 轻量协议类型 + authlib OAuth + token 管理），让 AI 客户端直接查询监控数据。

## 6. 构建与运行

```bash
# Docker（官方推荐，docker-compose.yml）
docker run -d --restart=always -p 5000:5000 \
  -v datastore-volume:/datastore --name changedetection ghcr.io/dgtlmoon/changedetection.io
# 界面：http://localhost:5000

# pip 安装（README-pip.md）
pip install changedetection.io
changedetection.io -d /path/to/data -p 5000

# 源码运行
pip install -r requirements.txt
python changedetection.py   # 仅是 changedetectionio.main() 的薄壳

# 测试（CONTRIBUTING.md）
pip install -r requirements-dev.txt
pytest changedetectionio/tests/test_notification.py   # 单文件
pytest changedetectionio/tests/                        # 全量

# 代码检查（.pre-commit-config.yaml / requirements-dev.txt）
ruff check .
```

JS 渲染监控需另起浏览器容器（docker-compose.yml 中注释给出的 sockpuppetbrowser/selenium 段落），或用官方 hosted 版。

## 7. 测试与工程质量

- **测试规模**：`changedetectionio/tests/` 227 个测试文件（约 2.5MB），按域分目录（apprise/、fetchers/、restock/、llm/、smtp/、proxy_list/、custom_browser_url/、plugins/ 等），pytest + pytest-flask（live server）+ pytest-mock + pytest-xdist 并行（`pytest.ini`、`requirements.txt`）。风格以"起真实 Flask live server + 内置 HTTP 测试服务器"的黑盒集成测试为主，覆盖 API 安全（test_api_security.py）、CSRF、认证、代理、通知模板等端到端行为。pytest 随主依赖安装（注释写明：方便用户在支持请求中复现）。
- **CI**：`.github/workflows/` 有 test-only.yml（README 徽章对应的主测试流）、containers.yml（多架构镜像）、pypi-release.yml、codeql-analysis.yml、third-party-licenses.yml（注释显示作者在意 license 合规，如特意选 rapidfuzz MIT 而非 GPL 的 levenshtein，见 `requirements.txt` 注释）。
- **代码质量**：ruff（`.ruff.toml`）+ pre-commit；依赖 pin 极其细致，requirements.txt 里有大量"为什么锁这个版本"的注释（litellm/jsonschema 冲突链、ARM 轮子、inscriptis 2.7.6 空白 diff 等），可读性在开源项目里属上乘。

## 8. 社区与维护状态

- **单人主导**：gh api contributors 共 178 人，但 dgtlmoon 本人 2389 次贡献，第二名是 dependabot（114），其余个人最高 29 次——典型的"单一维护者 + 机器人 + 少量外围贡献者"结构。
- **发版节奏**：gh api releases——0.70.4（2026-11-19）、0.70.3（11-09）、0.70.2/0.70.1（11-05）、0.60.10（10-26）、0.60.9（10-12）、0.60.8（09-28）、0.60.7（09-17），近两个月 8 个 release，约每 1-2 周一版，持续多年无断档。
- **活跃度**：pushed_at 2026-11-20；open issues 393。README 提供 BTC 捐赠（docs/btc-support.png）与 $8.99/月托管订阅，商业闭环支撑全职维护。
- 版本号从 0.60 跳到 0.70 属大版本推进（作者注释：semver 不用 0.01 形式），最近的 0.70.x 连发与 MCP/LiveLLM 特性期吻合（未逐条核对 changelog，推断自 release 频率与 README 新特性宣传）。

## 9. 亮点与风险

**亮点**
1. 检查流水线的工程精度：双哈希预检跳过（内容+过滤配置）、三层规则合并、多格式预处理（RSS/PDF/JSON），把"页面 diff"这个看似简单的需求做成了可靠系统（`processors/text_json_diff/processor.py`）。
2. 扩展点设计清晰：processor/fetcher/conditions/pluggy 四类插件机制并存且互不纠缠，image_ssim_diff、restock_diff 都是后来以插件形态加入的成功案例（`processors/`、`pluggy_interface.py`）。
3. 通知生态靠 Apprise 一步到位覆盖 100+ 渠道，diff 高亮为邮件客户端兼容（Gmail 剥标签）做了大量细节处理（`diff/__init__.py`、`notification/handler.py`）。
4. 文件存储零依赖：每 Watch 一个 UUID 目录 + JSON 原子写 + orjson/Brotli 优化，备份=拷目录，用户自托管心智负担极低（`store/`、`model/persistence.py`）。
5. 测试规模与 CI 完整度远超同体量单人项目（227 个测试文件、CodeQL、许可证合规 CI）。

**风险/局限**
1. 核心模型继承 dict（`model/Watch.py` 模块头自述技术债），类型安全差，Pydantic 迁移（PYDANTIC_MIGRATION.md）尚未落地，改动成本会随规模上升。
2. 单维护者 bus factor：2389/约2700+ 的贡献占比意味着作者停更项目即停摆（gh api contributors 数据）。
3. 大量精细的版本 pin（cryptography==50.0.2、werkzeug==3.1.9 等）降低了升级风险但也抬高了依赖维护负担，requirements.txt 注释里已出现多条依赖冲突史。
4. 高级功能（JS 渲染、Visual Selector）依赖外部浏览器容器或订阅服务，纯 HTTP 模式下对重 JS 页面无能为力；LLM 评估每次检查有成本，需用户自备 key。

## 10. 总结评价

changedetection.io 是"自托管网页变更监控"事实上的开源标杆：用 Flask + 文件存储保持了极低的部署门槛，用插件化的 processor/fetcher/条件规则体系承载了价格监控、截图对比、AI 摘要、MCP 等持续膨胀的需求，测试和发布纪律支撑它以单人项目之力做到了近 4 万星与多年高频发版。它适合三类人：想自建价格/补货/内容监控的普通用户（Docker 即用）；做竞品价格跟踪或运营巡检的小团队（REST API + Apprise 通知可编排进工作流）；以及想学习"如何把一个简单需求做成高可靠产品"的 Python 开发者——它的检查流水线、diff 渲染兼容层、异步 worker 池和依赖 pin 注释都值得精读。短板（dict 模型、单维护者）真实存在，但作者的商业化路径证明了这个模式在其定位下的可持续性。
