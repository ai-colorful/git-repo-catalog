# sansan0/TrendRadar 项目分析

> 星数 63137（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：一个"最快 30 秒部署"的中文热点监控助手——聚合多平台热榜与 RSS，用关键词或 AI 筛选出你关心的新闻，再推送到微信/飞书/钉钉/Telegram 等 9 类渠道（README.md 开头；GitHub 仓库描述）。

| 项目 | 事实 |
|---|---|
| 名称 | sansan0/TrendRadar |
| 星数 / Fork | 63,137 / 24,911（gh api，2026-10-31） |
| 主语言 | Python（要求 >=3.12，`pyproject.toml`） |
| 许可证 | GPL-3.0（`LICENSE`、README.md「许可证」章节） |
| 当前版本 | v6.11.0（`version` 文件、`pyproject.toml`）；MCP 服务协议版本 4.1.1（`version_mcp`） |
| 创建 / 最近推送 | 2025-04-28 / 2026-10-29（gh api，分析前两天仍在提交） |
| 默认分支 | master（无 GitHub Release，版本靠 README 更新日志与 `version` 文件管理） |
| 代码规模 | trendradar + mcp_server + tests 共约 3.45 万行 Python（wc -l 统计） |

## 2. 项目是什么，解决什么问题

TrendRadar 解决的是"信息过载"：用户只想追踪特定话题（如某个公司、某类政策），却要每天手动刷微博、知乎、B 站、抖音等十几个热榜。项目的做法是把"聚合 → 筛选 → 推送"整条链路自动化（README.md「核心功能」章节）。

数据侧，它不自己爬各平台页面，而是调用开源项目 [newsnow](https://github.com/ourongxing/newsnow) 的公共 API 一次拿到十余个中文平台的热榜（`trendradar/crawler/fetcher.py` 的 `DEFAULT_API_URL = "https://newsnow.busiyi.world/api/s"`），同时支持用户自建 newsnow 实例替换 `api_url`（`config/config.yaml` platforms 段）。v4.5.0 起还支持任意 RSS/Atom/JSON Feed 订阅源（`trendradar/crawler/rss/`）。

筛选侧提供两套机制：一是纯本地关键词规则 DSL（`config/frequency_words.txt`，支持正则、必选词、过滤词、别名，详见第 5 章）；二是 v6.5.0 新增的 AI 智能筛选——用户用自然语言写兴趣（`config/ai_interests.txt`），AI 自动提取标签并对新闻打分分类（`trendradar/ai/filter_pipeline.py`）。

推送侧是项目最"重"的部分：飞书、钉钉、企业微信、个人微信、Telegram、邮件、ntfy、Bark、Slack、通用 Webhook 共 10 个发送函数（`trendradar/notification/senders.py`），每个渠道都要适配各自的卡片格式与字节上限，配套的拆分器 `notification/splitter.py` 有 1871 行。

部署形态是星数暴涨的关键：官方主推 **fork 仓库 → 配 GitHub Secrets → GitHub Actions 每小时定时跑**（`.github/workflows/crawler.yml`），零服务器；另有 Docker、本地 uv、Cloudflare Pages 三种方式（docker/、setup-windows.bat、README.md「快速开始」）。Actions 版内置"7 天试用期"签到机制——连续 7 天不手动 Check In 就自动停跑，作者以此控制公共资源滥用（crawler.yml 头部注释）。

## 3. 技术栈与依赖

来自 `pyproject.toml`（全部锁死精确版本，另有 `uv.lock`）：

- **语言与运行时**：Python >= 3.12；构建后端 hatchling，入口 `trendradar = trendradar.__main__:main` 与 `trendradar-mcp = mcp_server.server:run_server`。
- **requests 2.33.0**：HTTP 抓取（热榜 API + RSS + 推送 Webhook 的唯一 HTTP 客户端）。
- **feedparser 6.0.12**：RSS/Atom 解析（配合自研 JSON Feed 解析，`trendradar/crawler/rss/parser.py`）。
- **fastmcp 2.12.5 + websockets 13.1**：MCP（Model Context Protocol）服务器，支持 stdio 与 HTTP 两种传输（`mcp_server/server.py`）。
- **litellm 1.82.6**：统一 LLM 调用层，AI 分析/筛选/翻译都走 `from litellm import completion`（`trendradar/ai/client.py:12,104`），用户可配任意 OpenAI 兼容供应商。
- **boto3 1.42.76**：可选的 S3 兼容远程存储（Cloudflare R2/OSS/COS/MinIO，`trendradar/storage/remote.py`）。
- **PyYAML / pytz / tenacity / json-repair**：配置、时区、重试、LLM 输出容错修复。

无 Web 框架依赖——HTML 报告是纯静态生成（`trendradar/report/html.py`），本地网页服务器只是简单 HTTP 静态服务（`start-http.sh`）。这个"重标准库 + 少依赖"取向与作者"轻量、易部署"的自我定位一致（README.md 第 53 行）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `trendradar/` | 核心主程序包（约 1.5 万行） |
| ├ `__main__.py` | CLI 入口 + NewsAnalyzer 主流水线（1717 行） |
| ├ `crawler/` | 数据抓取：newsnow 热榜 `fetcher.py` + RSS `rss/` |
| ├ `core/` | 关键词频率 `frequency.py`、权重统计 `analyzer.py`、配置加载 `config.py`、调度 `scheduler.py`、版本检查 `cdn.py` |
| ├ `ai/` | LLM 客户端、AI 筛选（filter/filter_pipeline）、AI 分析、AI 翻译、提示词加载 |
| ├ `notification/` | dispatcher（多账号调度）、senders（10 渠道）、splitter（按渠道字节上限拆分）、renderer/formatters |
| ├ `report/` | HTML 报告生成（html.py 3221 行，含暗色模式/搜索/导出） |
| ├ `storage/` | StorageManager + 本地 SQLite / S3 兼容远程双后端，按天分库（output/news/2025-12-21.db） |
| └ `commands/` | doctor 体检、test-notification、status、version 子命令 |
| `mcp_server/` | FastMCP 服务器（约 1.1 万行）：data_query/analytics/search/config_mgmt/notification/storage_sync 等 8 组工具 |
| `config/` | config.yaml（v2.4.0）、timeline.yaml（时段调度）、frequency_words.txt、AI 提示词；中英双套 |
| `tests/` | 8 个测试文件共 761 行 |
| `.github/workflows/` | crawler.yml（定时抓取）、clean-crawler.yml、docker.yml（镜像发布）、issue-guard.yml |
| `index.html` + `docs/` | 可视化配置编辑器（浏览器打开即用）与官网 |
| `docker/` | Dockerfile、Dockerfile.mcp、compose 文件、manage.py |

组织方式评述：单体单包（monorepo 式单 Python 包 + 独立 mcp_server 包），无内部服务拆分。模块边界清晰（抓取/筛选/AI/推送/存储/报告各一层），v6.10.0 起作者在持续做"模块拆分重构"（README.md 更新日志），但仍有明显的巨型文件：report/html.py 3221 行、mcp_server/tools/analytics.py 2639 行、splitter.py 1871 行。

## 5. 核心架构与关键实现

### 5.1 数据源适配层（`trendradar/crawler/fetcher.py`）

`DataFetcher.crawl_websites()` 逐平台串行请求 newsnow API，带抖动重试与限速：

```python
# fetcher.py:232-235  请求间隔加随机抖动，防被限流
if i < len(ids_list) - 1:
    actual_interval = request_interval + random.randint(-10, 20)
    actual_interval = max(50, actual_interval)
    time.sleep(actual_interval / 1000)
```

值得注意的安全设计：`_check_domain_safety()`（fetcher.py:51-84）校验 API 返回的每条链接必须为 HTTPS 且域名与配置的 `expected_domain`（含子域）匹配，并用标准库 `urlparse` 解析主机名、显式防 `https://baidu.com@evil.com` 这类 userinfo 绕过——这是 v6.9.0 因"链接劫持/数据篡改"风险专门加的（fetcher.py 注释；README.md 更新日志）。不通过则整平台数据丢弃。标题按 `{title: {ranks: [名次...], url, mobileUrl}}` 归并，同一标题在榜上多个位置的名次都保留，供后续权重计算。

RSS 侧（`crawler/rss/`）：`RSSFetcher` 用 requests.Session 抓取，`RSSParser` 同时支持 RSS/Atom 与 JSON Feed（`parser.py:79` `_is_json_feed`），v6.7.0 起 guid > url 的去重优先级解决"同文异 URL 重复入库"。

### 5.2 关键词过滤引擎（`trendradar/core/frequency.py` + `core/analyzer.py`）

`frequency.py` 实现了一个小而全的规则 DSL（模块 docstring 与 `_parse_word()`，frequency.py:20-72）：

- 普通词组、`+必须词`、`!过滤词`、`@数量上限`；
- `/正则/` 语法（`re.match(r'^/(.+)/[a-z]*$', word_config)`，编译失败降级为普通词）；
- `=> 别名` 显示名、`[组别名]` 组命名；
- `[GLOBAL_FILTER]` 全局排除区（`config/frequency_words.txt`）。

匹配后由 `core/analyzer.py` 的 `calculate_news_weight()` 排序：排名分（前 10 名线性衰减 `11 - min(rank, 10)`，归一化到 0-100）× 频次 × 热度三因子加权（analyzer.py:19-50），让"多平台同时上榜、排名靠前"的新闻置顶。

### 5.3 推送渠道层（`trendradar/notification/`）

三层结构：`dispatcher.py` 的 `NotificationDispatcher` 统一入口（支持每渠道最多 3 个账号、AI 翻译回填，dispatcher.py:43-78）→ `formatters.py`/`renderer.py` 按渠道渲染 → `senders.py` 10 个 `send_to_xxx()`（飞书 feishu.py:96 / 钉钉:230 / 企业微信:346 / Telegram:473 / 邮件:588 / ntfy:747 / Bark:938 / Slack:1100 / 通用 Webhook:1206）。消息超长时由 `splitter.py` 按各渠道字节上限拆批（飞书 30KB、钉钉 20KB 等，README.md mcp-v4.0.0 日志），`batch.py` 负责跨渠道分批发送的节奏控制。

### 5.4 主流程与调度（`trendradar/__main__.py` + `core/scheduler.py`）

主流程（`NewsAnalyzer.run()`，__main__.py:1611-1639）：初始化配置 → 取模式策略 → `_crawl_data()` 抓热榜 → `_crawl_rss_data()` 抓 RSS → `_execute_mode_strategy()` 合并、筛选、存储、推送 → finally 清理。三种推送模式为策略字典（__main__.py:34-53）：`incremental`（只推新增）、`current`（当前榜单匹配）、`daily`（全天汇总），配合 `config/timeline.yaml` 的时段预设（morning_evening/office_hours/night_owl 等）实现"什么时间推什么内容、是否做 AI 分析"。存储层 `StorageManager`（`storage/manager.py:19`）按天落 SQLite；配置了远程存储时走"下载当天库 → 合并 → 回传"（`storage/remote.py` 模块 docstring），这是 GitHub Actions 沙箱无持久盘时的状态延续方案。

## 6. 构建与运行

以下命令均来自 README.md、`pyproject.toml` 与 `docker/docker-compose.yml`（按规范未实际执行）：

```bash
# 本地（uv 方式，README「本地部署」）
uv sync && uv run trendradar            # 单次运行
uv run trendradar --doctor              # 环境与配置体检
uv run trendradar --test-notification   # 测试推送渠道
uv run trendradar --show-schedule       # 查看当前时段调度状态

# Docker（docker/docker-compose.yml：挂载 config/ 与 output/，env 注入 webhook 密钥）
cd docker && docker compose up -d

# GitHub Actions 部署：fork 后在 Settings>Secrets 配置 webhook，crawler.yml cron 每小时第 33 分钟触发
# MCP 服务器：trendradar-mcp（stdio 模式）或 HTTP 模式，见 README「MCP 客户端」

# 测试（tests/ 目录，未声明 test 框架，文件内为 pytest 风格断言）
python -m pytest tests/
```

配置通过 `config/config.yaml` + `frequency_words.txt` 完成，官方提供网页可视化编辑器（`index.html`，README 多处链接 sansan0.github.io/TrendRadar）。

## 7. 测试与工程质量

- **测试规模**：8 个文件共 761 行（`tests/`），均为小而聚焦的单元/集成测试：关键词匹配（test_frequency.py）、时间解析（test_time.py）、配置序列化（test_config_serialization.py）、AI 响应解析与额外参数（test_ai_analysis_response.py、test_ai_extra_params.py）、MCP stdio 连通（test_mcp_stdio.py，191 行）、RSS 计数（test_rss_total_count.py）、存储同步（test_storage_sync.py）。相对 3.4 万行源码，覆盖率偏低，且无 CI 工作流跑测试（.github/workflows/ 下只有 crawler/docker/issue-guard）。
- **代码质量观察**：中文 docstring 覆盖率高、模块头注释统一说明职责；防御性编码普遍（重试、抖动、域名校验、空标题兜底——v6.7.0 日志）；弱点是少数巨型文件（report/html.py 3221 行、splitter.py 1871 行）与 `print()` 直出日志（无 logging 分级）。
- **依赖管理**：全部依赖精确锁定 + uv.lock，但零版本区间意味着升级完全靠手动。

## 8. 社区与维护状态

- **活跃度**：创建于 2025-04-28，一年半到 63k 星；最近推送 2026-10-29（分析前 2 天），gh api 确认。34 个 open issue，208 watchers。
- **发版节奏**：不用 GitHub Release，以 README 更新日志为准：v6.0.0（2026/02）→ v6.5.0（03）→ v6.6.0（03/28）→ v6.7.0（05/15）→ v6.8.0（05/23）→ v6.9.0（06/02）→ v6.10.0（06/19）→ v6.11.0（10/13），平均每月一个功能版本（README.md「更新日志」）。
- **贡献者结构**：contributors API 显示约 10 位贡献者——典型的单人主导项目（作者 sansan0），fork 数 2.4 万说明大量用户是"fork 即部署"而非协作开发。
- **推广路径**：README 致谢区记录了小众软件、LinuxDo、阮一峰科技爱好者周刊的推荐（README.md:101-106），配合 Trendshift 徽章；数据源 newsnow 作者公开许可其公共 API 使用（README.md:91-97）。

## 9. 亮点与风险

**亮点**

1. 极低使用门槛的部署设计：fork + Secrets + GitHub Actions 即可运行，"30 秒部署"名副其实；Actions 试用版 7 天签到制是少见的资源自觉设计（`.github/workflows/crawler.yml` 头部注释）。
2. 数据安全的细节意识：API 返回链接的 HTTPS + 域名白名单校验并防 userinfo 绕过（`crawler/fetcher.py:51-84`），提示词注入面与数据源可信度都有明确处理。
3. 关键词 DSL 表达力强且零门槛：正则/必选/过滤/别名/分组在一个 txt 里解决（`core/frequency.py`），还配了可视化编辑器兜底。
4. 渠道适配完成度罕见：10 个推送渠道 + 按渠道字节上限智能拆分 + 每渠道 3 账号 + AI 翻译回填，中文推送生态（个人微信、Bark、ntfy）覆盖比同类项目全。
5. MCP 双端集成：既能作为 MCP 服务器被 Claude/Cline 等查询数据，也能把 AI 生成内容直接推到 9 个渠道（`mcp_server/`，README mcp-v4.0.0 日志）。

**风险/局限**

1. 数据源单点依赖：默认热榜来自 newsnow 作者善意维护的公共 API（README.md:91-97 明说"基于善意和信任"），该服务限流或下线会直接影响所有默认配置用户；项目已提供自建 newsnow 出路。
2. GPL-3.0 对二次分发的商业闭源使用不友好；且 README 更新日志提示配置文件多次 breaking change（v6.0.0 迁移说明），升级需跟版。
3. 测试薄弱：761 行测试对 3.4 万行代码，核心推送/拆分逻辑（splitter.py、senders.py）基本无自动化回归保障。
4. 个人微信等渠道依赖第三方 hook 方案的稳定性存疑（README 将个人微信与企业微信并列，具体实现路径未在本次抽样中确认——标注：未确认）。

## 10. 总结评价

TrendRadar 是 2025-2026 年"AI + 信息聚合"浪潮里跑出来的现象级中文工具：它没有技术上的深水区，胜在对目标用户（不想写代码又想自定义监控的中文用户）的需求拆解极准——部署够快、关键词够灵活、渠道够全、AI 功能踩点够新，再配上高质量 README 和可视化编辑器，把"fork 即用"做到了极致，63k 星是产品力与传播力共同的结果。代码本身是务实的单体工程，模块划分清楚、防御性细节（域名校验、拆批、去重）用心，但测试与日志体系明显跟不上体量。适合两类人：想要私有化热点监控/舆情简报的个人与团队直接部署使用；以及想把"多渠道推送适配""AI 筛选流水线""MCP 服务器封装"当作参考实现的 Python 开发者读它的 notification/、ai/、mcp_server/ 三个目录。
