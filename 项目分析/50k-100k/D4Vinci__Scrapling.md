# D4Vinci/Scrapling 项目分析

> 星数 89309（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：一个"HTTP 请求 + 隐身浏览器 + 强力解析 + 并发爬虫"四合一的 Python 网页抓取框架，招牌能力是网站改版后仍能自动"重定位"页面元素的自适应解析器。

| 项目 | 事实 |
| --- | --- |
| 名称 | Scrapling（作者 Karim Shoair / D4Vinci） |
| 星数 / Fork | 89,309 / 9,370（gh api，2026-10-31） |
| 语言 | Python，要求 >=3.10（pyproject.toml） |
| 许可证 | BSD-3-Clause（LICENSE） |
| 当前版本 | 0.5（pyproject.toml、scrapling/__init__.py；release v0.5 发布于 2026-10-17） |
| 最近活跃 | 最近 push 2026-10-29，仓库创建于 2024-10-13（gh api） |
| 代码规模 | 主包约 16,108 行 Python；测试 99 个文件约 25,529 行 |

## 2. 项目是什么，解决什么问题

Scrapling 是一个完整的 Python 网页抓取框架（README.md："Scrapling is a Python Web Scraping framework that brings fast HTTP requests, stealth browsers, powerful parsing, and concurrent crawling into one library"）。它把抓取工作流拆成四层，每层都可以独立使用：

1. **抓取层（fetchers）**：`Fetcher` 基于 curl_cffi 做 TLS 指纹伪装、HTTP/3 的高速 HTTP 请求；`StealthyFetcher` 基于 Patchright（Playwright 的反检测分支）驱动隐身浏览器，并内置 Cloudflare Turnstile/Interstitial 自动求解（README.md "Advanced Websites Fetching" 一节）。
2. **解析层（parser）**：`Selector` 类封装 lxml，支持 CSS/XPath/文本/正则查找、BS4 风格的 `find_all`、链式选择，以及独门的**自适应元素追踪**——网站改版后用相似度算法在整棵 DOM 里重新找回目标元素（parser.py）。
3. **爬虫层（spiders）**：Scrapy 风格的 `Spider` 基类，async 并发、AutoThrottle、断点续爬（crawldir）、robots.txt 合规、内置 CrawlSpider/SitemapSpider/CSVFeedSpider/ShopifySpider 等模板（scrapling/spiders/）。
4. **AI/终端层**：MCP Server 让 Claude/Cursor 等 agent 直接调用浏览器抓取，`page.markdown()` 输出 RAG 就绪的干净 Markdown，另有 CLI 与 IPython 交互式抓取 shell（scrapling/core/ai/server.py、scrapling/cli.py）。

它解决的问题很明确：现代网页抓取的最大痛点不是"取回 HTML"，而是**反爬对抗**（TLS 指纹、Cloudflare 盾）和**页面结构变化导致选择器失效**。前者靠 curl_cffi 伪装 + Patchright 隐身浏览器 + Cloudflare 自动点击求解；后者靠 v0.1 起就主打的自适应元素重定位——这是它与 Scrapy、BeautifulSoup、Playwright 原生方案的最大差异点。

项目迭代很快：v0.3 引入会话式架构与 AI 集成，v0.5（2026-10）删除了普通 Playwright 后端、全面转向 Patchright 隐身后端，并把浏览器后台 API 捕获改为 `session.network.search()` 形式（docs/migrating-to-v0.5.md）。

## 3. 技术栈与依赖

- **语言/构建**：纯 Python（>=3.10），setuptools 构建，静态版本号 0.5（pyproject.toml）。
- **核心依赖（装包即有）**：lxml（HTML 解析）、cssselect（CSS 选择器解析）、orjson（快速 JSON）、tld（域名提取）、w3lib（URL/实体处理）（pyproject.toml `dependencies`）。
- **fetchers 可选依赖**：curl_cffi（浏览器 TLS 模拟 HTTP 客户端）、playwright + patchright（浏览器自动化与隐身版）、browserforge + apify-fingerprint-datapoints（请求头/指纹生成）、msgspec、anyio、protego（robots.txt 解析）、click（CLI）。
- **extras 分组**：`rag`（markdownify 转 Markdown）、`ai`（mcp 官方 SDK）、`shell`（IPython）、`all`（pyproject.toml `[project.optional-dependencies]`）。
- **类型与质量工具**：mypy + pyright 双静态检查、ruff、bandit（.bandit.yml）、pre-commit（.pre-commit-config.yaml）。
- **文档**：zensical*.yml（9 种语言的文档站配置）+ .readthedocs.yaml，文档站 scrapling.readthedocs.io。
- **分发**：PyPI 包 `scrapling`，入口点 `scrapling`（CLI）与 `scrapling-mcp`（MCP server）；Docker 镜像 pyd4vinci/scrapling 由 CI 自动构建（Dockerfile、.github/workflows/docker-build.yml）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| scrapling/parser.py | 核心解析器 `Selector`/`Selectors`，自适应追踪逻辑（1390 行） |
| scrapling/engines/static.py | Fetcher/AsyncFetcher/FetcherSession，curl_cffi HTTP 引擎（805 行） |
| scrapling/engines/_browsers/ | Patchright 隐身浏览器引擎：_stealth.py（Cloudflare 求解）、_base.py、_network.py（后台 API 捕获）、_page.py 等 |
| scrapling/engines/toolbelt/ | 辅助件：convertor.py（响应→Response 工厂）、fingerprints.py（browserforge 指纹）、proxy_rotation.py、ad_domains.py（3500+ 广告域名表，3537 行） |
| scrapling/fetchers/ | requests.py / stealth_chrome.py 薄门面，统一 lazy import 出口 |
| scrapling/spiders/ | 爬虫框架：engine.py（CrawlerEngine）、scheduler、throttle（AutoThrottle）、checkpoint（断点续爬）、cache、robotstxt、links、templates/（5 种爬虫模板） |
| scrapling/core/ | 基础设施：storage.py（SQLite 自适应存储）、translator.py（CSS→XPath，改编自 Parsel）、custom_types.py、mixins.py（选择器生成）、shell.py（IPython shell）、ai/（MCP server，server.py 1160 行） |
| scrapling/integrations/scrapy.py | `scrapling_response` 装饰器，让 Scrapy 回调用 Scrapling 解析 |
| scrapling/cli.py | click CLI：install / extract / shell / mcp |
| tests/ | 99 个测试文件，按 parser/fetchers/spiders/core/ai/cli 分层 |
| docs/ | 多语言文档站源码 + 9 份翻译版 README |
| agent-skill/ | 面向编码 agent 的 Skill 包（含 zip 分发） |

组织方式评述：单包（single package）结构，`core`/`engines`/`fetchers`/`spiders` 四层职责清晰；所有公共出口（`scrapling/__init__.py`、`fetchers/__init__.py`）都用 `__getattr__` 做 lazy import，保证"只装解析器"的最小安装不被拖累（scrapling/__init__.py `_LAZY_IMPORTS`）。

## 5. 核心架构与关键实现

### 5.1 自适应元素追踪（parser.py）

这是项目的招牌。`Selector` 不继承 lxml 的 HtmlElement（docstring 解释了原因：lxml 元素不可 pickle，见 parser.py L106-112），而是包装 root 元素。开启 `adaptive=True` 后：

- **保存**：`save()` 把元素序列化成 dict（tag/text/attributes/path/parent/siblings），存入 SQLite（默认 `scrapling/elements_storage.db`，按网站主域名分键，见 core/storage.py `SQLiteStorageSystem` 与 `_get_base_url`）。
- **重定位**：选择器落空时 `relocate()` 遍历整棵 DOM，对每个候选元素算相似度分（parser.py L522-569、L814-879）：

```python
score += 1 if original["tag"] == data["tag"] else 0
if original["text"]:
    score += SequenceMatcher(None, original["text"], data.get("text") or "").ratio()
score += self.__calculate_dict_diff(original["attributes"], data["attributes"])
for attrib in ("class", "id", "href", "src"):
    if original["attributes"].get(attrib):
        score += SequenceMatcher(...).ratio()
score += SequenceMatcher(None, original["path"], data["path"]).ratio()
# 再累计 parent_name/parent_attribs/parent_text/siblings，最后
return round((score / checks) * 100, 2)
```

得分用 difflib 的 `SequenceMatcher` 对文本、属性键值、DOM 路径、父节点、兄弟节点逐项加权平均，取全页最高分且不低于阈值（默认 40%）的元素集合作为重定位结果。同类的 `find_similar()` 用 `__are_alike()` 按属性相似度找同构元素（parser.py L979-1020）。

- **容错设计**：存储类必须被 `lru_cache` 包裹（parser.py L183-187 强制校验），SQLite 连接带 `RLock` 保证线程安全（core/storage.py 头部）。

### 5.2 抓取引擎（engines/static.py 与 engines/_browsers/）

- **HTTP 引擎**：`_ConfigurationLogic` → `_SyncSessionLogic`/`_ASyncSessionLogic` → `FetcherSession` 三层结构；`_make_request` 统一走 curl_cffi，支持 `impersonate`（TLS 指纹伪装）、`http3`、`stealthy_headers`（用 browserforge 生成真实浏览器头，engines/static.py L211-298、toolbelt/fingerprints.py）。响应经 toolbelt/convertor.py `ResponseFactory.from_http_request` 包装成带 `Selector` 能力的 `Response`。
- **浏览器引擎**：v0.5 起只有 Patchright 一个后端（docs/migrating-to-v0.5.md "The plain Playwright browser backend has been removed"）。`StealthySession.fetch` 支持等待选择器、networkidle、广告域名拦截（toolbelt/ad_domains.py 内置 ~3500 域名）、DoH 防 DNS 泄漏、CDP 远程浏览器（engines/_browsers/_stealth.py、_validators.py）。
- **Cloudflare 求解器**（_stealth.py L114-230）：先轮询检测挑战类型（`embedded`/`non-interactive`/turnstile），embedded 型进 iframe 点复选框，turnstile 型计算 widget 的 bounding_box 后用 `page.mouse.click(x+26~28, y+25~27, delay=randint(100,200))` 模拟人手点击，失败递归重试至上限。全部带 `pragma: no cover`（逻辑依赖真实盾页）。
- **后台 API 捕获**：`record_requests=True` 时 `_network.py` 持久化页面产生的所有响应，之后 `session.network.search()` 可检索历史 API 数据——针对"数据在前端 API 里"的现代站点。

### 5.3 爬虫框架（spiders/）

`CrawlerEngine` 用 anyio 的 task group + 内存对象流编排并发（spiders/engine.py L1-60），组件各司其职：`Scheduler`（URL 去重与 BFS/策略）、`AutoThrottle`（按网站响应速度自动调延迟、阻塞时翻倍并尊重 Retry-After，spiders/throttle.py）、`CheckpointManager`（Ctrl+C 存 pending 请求与 seen URL，重启续爬）、`ResponseCacheManager`（开发模式缓存响应）、`RobotsTxtManager`（protego 解析，可配置 compliance）。`Spider` 基类还内置了按日志级别计数的 `LogCounterHandler`（spiders/spider.py L17-50）。五个模板（CrawlSpider/SitemapSpider/XMLFeed/CSVFeed/ShopifySpider）都在 spiders/templates/。

### 5.4 选择器层兼容性设计（core/translator.py、core/mixins.py）

CSS→XPath 转译器改编自 Parsel，目的写在文件头：让 Scrapling 直接支持 `::text`、`::attr()` 伪元素，Scrapy 用户零学习成本；对含逗号的组合选择器会先 split 再分别保存自适应数据（parser.py L600-620）。`SelectorsGeneration` mixin 能为任意元素反向生成 CSS/XPath 选择器（core/mixins.py L66-87），支撑"自动选择器生成"卖点。

## 6. 构建与运行

```bash
# 最小安装（仅解析器）
pip install scrapling
# 完整安装（fetchers + 浏览器）
pip install "scrapling[fetchers]" && scrapling install      # 下载浏览器与系统依赖
pip install "scrapling[all]"                                 # MCP/rag/shell 全量
# CLI 用法
scrapling shell                                              # IPython 交互 shell
scrapling extract get 'https://quotes.toscrape.com/' quotes.md
scrapling-mcp                                                # 启动 MCP server
# Docker
docker pull pyd4vinci/scrapling
```
（来源：README.md Installation/CLI 章节、pyproject.toml `[project.scripts]`）

```bash
# 测试（tox.ini）
pip install tox && tox                       # py310-313 + pre-commit
pytest -m "not asyncio and not browser"      # 常规测试（可并行）
pytest -m browser                            # 真实浏览器测试（不并行）
```

## 7. 测试与工程质量

- **规模**：99 个测试文件、约 25,529 行测试代码，测试代码量超过主包（tests/ 目录 wc -l 统计）。分层清晰：tests/parser（test_adaptive.py、test_find_similar_advanced.py 等自适应专项）、tests/fetchers（test_cloudflare_solver.py、test_shadow_*.py 等 30 个文件）、tests/spiders（test_engine.py、test_checkpoint.py、test_throttle.py 等 20 个文件）、tests/ai、tests/cli。
- **框架与风格**：pytest + pytest-asyncio（strict 模式），自定义 `browser`/`asyncio` marker 区分重测试；`--doctest-modules` 把文档字符串也纳入测试（pytest.ini）。tox 按 marker 分三波跑：浏览器测试不并行、asyncio 不并行、其余 `-n auto` 并行（tox.ini）。
- **CI**：4 条 workflow——tests.yml（多 OS 矩阵，60 分钟超时）、code-quality.yml、docker-build.yml、release-and-publish.yml（.github/workflows/）。
- **质量观察**：README 声称 92% 覆盖率与全量类型标注；源码确有完整 type hints，pyproject 配置了 mypy（check_untyped_defs）与 pyright（basic 模式）；ruff + bandit + pre-commit 齐备。工程化程度在同类开源抓取库里属第一梯队。

## 8. 社区与维护状态

- **活跃度**：仓库 2024-10-13 创建，两年内冲到 89k star；最近 push 2026-10-29（分析前 2 天），22 个 open issue，9,370 fork（gh api，2026-10-31）。
- **Release 节奏**：v0.5（2026-10-17）、v0.4.15（2026-08-23）、v0.4.14（2026-08-10），docs/changelog.md 含 41 个版本小节，迭代密集且每次都有迁移文档（docs/migrating-to-v0.5.md）。
- **贡献者结构**：作者 Karim Shoair 一人主导（pyproject authors/maintainers 均为单人；README 落款 "Designed & crafted with ❤️ by Karim Shoair"）；贡献者数量未确认（tarball 无 git 历史，未做 gh api contributors 查询）。
- **商业化信号**：README 有 9 个 Platinum Sponsors（DataImpulse、NodeMaven 等代理服务商），配有 Discord 社区、X 账号、Trendshift/Star-History 徽章，是典型的"开源核心 + 赞助变现"运营模式。

## 9. 亮点与风险

**亮点**
1. 自适应元素追踪是差异化护城河：元素改版后按 tag/文本/属性/DOM 路径/父子结构的 SequenceMatcher 相似度自动重定位（parser.py L522-879），同类库无此能力。
2. 反爬对抗成体系：curl_cffi TLS 指纹伪装 + browserforge 指纹头 + Patchright 隐身浏览器 + 真实可用的 Cloudflare Turnstile 自动点击求解（engines/_browsers/_stealth.py），覆盖 HTTP 与浏览器两条链路。
3. 工程质量第一梯队：2.5 万行测试超过主包、双静态类型检查、4 条 CI、多语言文档站、每个大版本附迁移指南。
4. API 兼容策略聪明：Parsel 系 `::text/::attr()` 伪元素 + BS4 风格 find_all + `scrapling_response` 装饰器直接接入存量 Scrapy 项目（core/translator.py、integrations/scrapy.py）。
5. 抓住 AI 时代入口：MCP server、Agent Skill、`page.markdown()` RAG 输出、ContentSpider 整站转 Markdown 语料（core/ai/server.py、agent-skill/），在抓取库里最早一批拥抱 agent 生态。

**风险/局限**
1. 大版本破坏性变更频繁：v0.5 删除 DynamicFetcher、capture_xhr 等整套旧 API（docs/migrating-to-v0.5.md），生产用户升级成本高；版本号仍停在 0.x（Development Status :: 4 - Beta）。
2. 单人维护风险：核心逻辑几乎全部出自一位作者，bus factor 为 1。
3. Cloudflare 求解器依赖坐标点击与页面结构猜测（`#cf_turnstile` 等硬编码选择器，_stealth.py L182-186），盾牌升级即失效，且此类逻辑标注 `pragma: no cover` 无法被测试保护。
4. 反爬对抗的合规灰色地带：README 虽有免责声明，但自动绕过 CAPTCHA/Cloudflare 的能力与目标站服务条款存在天然冲突，商用需自担法律风险。

## 10. 总结评价

Scrapling 是近两年 Python 抓取生态里最亮眼的新框架：它没有重新发明解析器，而是把"自适应元素追踪 + 成体系的反隐身对抗 + Scrapy 风格爬虫框架 + MCP/agent 入口"打包成一条完整工作流，两年 89k star 的增速证明了市场对这种一体化定位的需求。工程质量（测试量超主包、类型全覆盖、迁移文档齐全）远超典型个人项目，但 0.x 版本号下的破坏性迭代和单人维护是长期使用的现实约束。适合需要对抗反爬、页面频繁改版的数据采集工程师直接上生产（锁版本），适合想研究反检测浏览器工程化、相似度算法落地、现代 Python 库工程实践的开发者通读源码——parser.py 与 engines/_browsers/ 是全库最值得精读的两处。
