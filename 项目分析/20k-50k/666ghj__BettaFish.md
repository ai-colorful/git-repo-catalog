# 666ghj/BettaFish 项目分析

> 星数 42686（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-13

## 1. 项目速览

一句话定位：从零手写的多 Agent 舆情分析系统（"微舆"），以聊天式提问驱动多个 LLM Agent 并行检索社媒/新闻/私有数据库，经"论坛"辩论协作后生成带 IR 中间表示的交互式 HTML 舆情报告。

| 项目 | 内容 |
|------|------|
| 名称 | 666ghj/BettaFish（微舆） |
| 星数 / Fork | 42686 / 7638（gh api 核实，2026-11-13） |
| 主语言 | Python（174 个 .py 文件，约 5.7 万行） |
| 许可证 | GPL-2.0 |
| 创建时间 | 2024-07-01 |
| 最近推送 | 2026-10-16（gh api 核实） |
| 最新 Release | v3.0.0（2025-12-23） |
| 子模块 | NanmiCoder/MediaCrawler（多平台社媒爬虫，7.1 万星） |
| 贡献者 | 41 人（contributors API Link 头推算） |

## 2. 项目是什么，解决什么问题

BettaFish 定位为"人人可用的舆情分析助手"。传统舆情系统以数据看板为主，用户需要自己解读指标；BettaFish 把交互改成一句话提问，由系统自动完成"全网检索 → 多 Agent 深挖 → 集体讨论 → 结构化报告"的全链路（README.md「项目概述」）。作者称其目标是"始于舆情，而不止于舆情"，即一个可迁移到金融等领域的通用数据分析引擎（README.md）。

系统由四类 Agent 组成（README.md 架构章节，与源码目录一一对应）：

- **Query Agent**（`QueryEngine/`）：调用 Tavily 搜索 API 做国内外新闻广度搜索；
- **Media Agent**（`MediaEngine/`）：多模态搜索（博查 Bocha、Anspire），可返回网页/图片与天气、股票等"模态卡"结构化数据（`MediaEngine/tools/search.py`）；
- **Insight Agent**（`InsightEngine/`）：对用户私有舆情数据库做只读 SQL 挖掘，内置 Qwen 关键词优化中间件与本地情感分析模型（`InsightEngine/tools/keyword_optimizer.py`、`sentiment_analyzer.py`）；
- **Report Agent**（`ReportEngine/`）：汇总三方结果，按模板逐章节生成报告。

三个分析 Agent 并行工作期间，**ForumEngine** 充当"辩论主持人"：监控各 Agent 日志、汇总发言、由主持人 LLM 生成引导发言回灌给各 Agent，形成多轮辩论循环；最后 Report Agent 产出基于 IR（中间表示）渲染的交互式 HTML/PDF 报告。

数据侧配套 **MindSpider** 爬虫系统（`MindSpider/`）：先用 LLM 从当日新闻提取热点话题（`BroadTopicExtraction/`），再按关键词调度 MediaCrawler 子模块抓取 7 个平台（xhs/dy/ks/bili/wb/tieba/zhihu，`MindSpider/DeepSentimentCrawling/platform_crawler.py`）写入 MySQL/PostgreSQL，形成 Insight Agent 的"公域数据池"。此外 `SentimentAnalysisModel/` 收录了 5 套自研情感分析模型（BERT-LoRA、GPT2-LoRA/Adapter、小参数 Qwen 微调、传统 LSTM/SVM/XGBoost）及训练数据集。

项目最大的标签是"从 0 实现、不依赖任何框架"——经全仓库 grep 验证：`langchain/langgraph/llamaindex/crewai/autogen/pydantic_ai/semantic_kernel` 均零匹配，`requirements.txt` 中也没有任何 Agent 框架；Agent 循环、工具分发、状态管理全部为手写 Python（详见第 5 章）。

## 3. 技术栈与依赖

- **核心 Web**：Flask 2.3 + flask-socketio/eventlet（主控台实时日志）、Streamlit 1.28（三个引擎各自的独立界面）、FastAPI/uvicorn 亦在依赖中（`requirements.txt`）。
- **LLM 接入**：仅依赖 `openai` SDK 直连任意 OpenAI 兼容端点，配合自研 `utils/retry_helper.py` 重试装饰器；每个 Agent 独立配置 API Key/Base URL/模型（`.env.example` 推荐 Insight=kimi-k2、Media/Report=gemini-2.5-pro、Query/MindSpider=deepseek-chat、论坛主持人=qwen-plus）。
- **搜索 API**：tavily-python（QueryEngine）、博查/Anspire（MediaEngine）。
- **数据库**：SQLAlchemy 2.0 异步 + asyncpg/aiomysql（Insight 只读查询）、motor/redis/aiosqlite 也在依赖中；生产部署用 PostgreSQL 15（`docker-compose.yml`）。
- **爬虫**：playwright + httpx + bs4/lxml/parsel，实际多平台抓取委托给 MediaCrawler 子模块。
- **机器学习**：torch + transformers + sentence-transformers + scikit-learn + xgboost（本地情感分析模型与 `SentimentAnalysisModel/` 训练脚本）。
- **报告渲染**：WeasyPrint（PDF）、自研 HTML 渲染器；中文字体 SourceHanSerifSC 随仓库附带（`ReportEngine/renderers/assets/fonts/`）。
- **可视化**：plotly、matplotlib、wordcloud、jieba。
- **部署**：Dockerfile（python:3.11-slim）+ docker-compose（应用 + postgres 双服务）；requirements 注释表明主要面向 Windows + Conda 环境。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|------|------|
| `app.py`（1350 行） | Flask 主控台：进程编排、配置 API、日志流、论坛日志接口、`/api/search` 分发 |
| `QueryEngine/` | Tavily 新闻广度搜索 Agent（agent/llms/nodes/tools/state/prompts 七件套） |
| `MediaEngine/` | 多模态搜索 Agent（博查+Anspire 工具集） |
| `InsightEngine/` | 私有数据库挖掘 Agent（只读 SQL + 关键词优化 + 情感分析工具） |
| `ForumEngine/` | 日志监控器 + LLM 论坛主持人（monitor.py 858 行、llm_host.py 262 行） |
| `ReportEngine/` | 报告生成 Agent：IR schema/校验、章节装订器、HTML/PDF 渲染器（agent.py 1547 行、html_renderer.py 6536 行） |
| `MindSpider/` | 话题提取 + 深度爬取（MediaCrawler 子模块） + 数据库 schema/ORM |
| `SentimentAnalysisModel/` | 5 套情感模型训练/预测脚本与数据集（含 32MB LSTM 权重、19MB×3 微博数据集） |
| `SingleEngineApp/` | 三个引擎的 Streamlit 单独运行入口 |
| `utils/` | forum_reader（读论坛发言）、retry_helper、github_issues |
| `tests/` | pytest 测试（10 个文件，2605 行） |
| `templates/` + `static/` | Flask 前端（单文件 index.html 5.3MB）与报告示例 |

组织方式评述：单体多引擎结构，四个 Engine 内部布局刻意保持一致（agent.py + nodes + tools + state + prompts），像同一模板的四个实例，降低了阅读成本；但模块间通过 `sys.path.append` 硬编码路径互导（如 `ForumEngine/llm_host.py` 第 10-28 行），包边界并不严格。仓库约 240MB，主要是随仓附带的数据集与模型权重。

## 5. 核心架构与关键实现

### 5.1 手写 Agent 编排（"不依赖框架"的实证）

以 QueryEngine 为例，Agent 是一个普通 Python 类，流程为"生成报告结构 → 按段落初步搜索+总结 → 反思循环 → 汇总报告"（`QueryEngine/agent.py` `DeepSearchAgent.research()`）：

```python
def _reflection_loop(self, paragraph_index: int):
    paragraph = self.state.paragraphs[paragraph_index]
    for reflection_i in range(self.config.MAX_REFLECTIONS):
        reflection_output = self.reflection_node.run(reflection_input)
        search_query = reflection_output["search_query"]
        search_tool = reflection_output.get("search_tool", "basic_search_news")
        search_response = self.execute_search_tool(search_tool, search_query, **search_kwargs)
```

工具分发 `execute_search_tool()` 是手写 if/else 映射（`QueryEngine/agent.py:100`），LLM 调用是 `openai` SDK 的薄封装——`LLMClient.invoke()` 只做消息拼装、时间前缀注入与重试（`QueryEngine/llms/base.py`），状态用纯 dataclass（`QueryEngine/state/state.py`）。全仓库 grep 无任何 Agent 框架导入，声明属实。

### 5.2 ForumEngine：日志文件作为 Agent 间消息总线

这是项目最有辨识度的设计。三个 Agent 各自用 loguru 写 `logs/{insight,media,query}.log`，`LogMonitor` 线程轮询增量 tail 这三个文件，用正则匹配 SummaryNode 输出标记（`ForumEngine/monitor.py` 的 `target_node_patterns`）捕获各 Agent 的"发言"进入缓冲区；每积累 5 条发言就调用主持人 LLM 生成引导，写入 `logs/forum.log`：

```python
self.monitored_logs = {
    'insight': self.log_dir / 'insight.log',
    'media': self.log_dir / 'media.log',
    'query': self.log_dir / 'query.log'
}
self.agent_speeches_buffer = []
self.host_speech_threshold = 5  # 每5条agent发言触发一次主持人发言
```

反向通道是 `utils/forum_reader.py` 的 `get_latest_host_speech()`——各引擎的 SummaryNode 在生成总结前读取 forum.log 中最新一条 `[HOST]` 发言，把主持人的辩论引导注入下一轮 prompt（`InsightEngine/nodes/summary_node.py:27`）。整套协作是"黑板模式"的文件系统实现：零消息中间件、零 RPC，但也将多 Agent 协作耦合到了单机单目录的日志文件上，且主持人质量高度依赖日志格式的稳定性（monitor.py 里保留了旧格式兼容与错误块跳过逻辑）。

### 5.3 ReportEngine：报告 IR 中间表示

报告不走"一次生成一大段 Markdown"，而是编译器式流水线（`ReportEngine/agent.py` 1547 行）：模板选择 → 文档布局 → 字数预算 → 逐章节生成 JSON。章节 JSON 必须通过自研 IR 契约校验（`ReportEngine/ir/schema.py`，定义 17 种块类型与 13 种行内标记，`IR_VERSION = "1.0"`），再由 `DocumentComposer` 装订为整本文档（补齐章节 ID、去重锚点、注入元数据，`ReportEngine/core/stitcher.py`），最终由 6536 行的 `renderers/html_renderer.py` 渲染成交互式 HTML，PDF 走 WeasyPrint。生成失败有章节级校验重试与图表修复 API（`ReportEngine/utils/chart_repair_api.py`）。SSE 流式接口含心跳、断线补发、任务取消（`ReportEngine/flask_interface.py`）。

### 5.4 进程编排与数据接入

`app.py` 是一个"驾驶舱"：用 subprocess 启动三个 Streamlit 子应用（端口 8501/8502/8503）、ForumEngine 监控线程与 ReportEngine，提供健康检查、优雅退出、并发清理（`initialize_system_components()`、`cleanup_processes_concurrent()`）；`/api/search` 把用户提问 POST 给各 Streamlit 应用转发执行（`app.py:1152`）。数据侧 `MindSpider/DeepSentimentCrawling/platform_crawler.py` 通过 subprocess + 配置文件注入调用 MediaCrawler 抓取 7 个平台，写入由 `MindSpider/schema/` 定义的 ORM 表；Insight Agent 通过 SQLAlchemy 异步只读引擎访问该库（`InsightEngine/utils/db.py`）。

## 6. 构建与运行

Docker 方式（README.md「快速开始」）：

```bash
cp .env.example .env   # 配置 7 组 LLM Key + 数据库
docker compose up -d   # bettafish + postgres:15，端口 5000/8501-8503
```

源码方式（Windows + Conda 为主要目标环境）：

```bash
conda create -n bettafish python=3.11 && conda activate bettafish
pip install -r requirements.txt        # PDF 功能需先装 WeasyPrint 系统依赖
playwright install chromium            # 爬虫浏览器驱动
git submodule update --init --recursive   # MediaCrawler 子模块（MindSpider 必需）
python app.py                          # 启动主控台，自动初始化数据库与子进程
```

测试：`python tests/run_tests.py`（或 pytest，`tests/run_tests.py`）。数据库无需手工建表，`app.py` 启动时自动检测初始化（README.md）。

## 7. 测试与工程质量

- 测试规模：`tests/` 共 10 个 pytest 文件、2605 行，另有 2 个工具级测试在 `ReportEngine/utils/` 下。覆盖集中在基础设施：论坛监控器（`test_monitor.py`）、报告脱敏（`test_report_engine_sanitization.py`）、导出路径、重试逻辑、星标抓取脚本等；四个 Agent 的核心检索/生成流程没有自动化测试。
- CI：`.github/workflows/docker_ci.yml`（打 v* 标签时构建发布 ghcr 镜像）与 `update-star-history.yml`（定时更新星标历史图）。**没有测试/ lint 的 CI**，测试只靠本地手动跑。
- 代码质量观察：模块模板化程度高、中文注释密度大，可读性好；但存在大量 `sys.path.append` 式导入 hack、单文件偏大（html_renderer.py 6536 行、app.py 1350 行、templates/index.html 5.3MB 单文件前端）；`LICENSE` 为 GPL-2.0，附带 `THIRD_PARTY_NOTICES.md` 与双语文档/贡献指南，工程规范性意识较好。

## 8. 社区与维护状态

- 活跃度：创建于 2024-07，最近推送 2026-10-16（gh api），维护持续。Release 节奏密集：v1.1.0（2025-11-04）→ v1.2.0 → v2.0.0（2025-11-28）→ v2.1.0 → v3.0.0（2025-12-23），两个月内 5 个版本，且 Docker CI 覆盖 tag 发布（gh api releases）。
- 社区结构：41 位贡献者（contributors API），主体推测为作者主导 + 社区 PR；README 显示有赞助商（LLM API 中转商）与活跃的 linux.do 社区讨论帖，README 徽章含 Trendshift。issue 管理有配套 `utils/github_issues.py` 统一生成错误上报链接。
- 多语言：中英双语 README/CONTRIBUTING/CODE_OF_CONDUCT 齐备，国际化意识好。
- 以上均有仓库内文件或 gh api 佐证；无 CHANGELOG 文件（版本变更详情需看 Release 说明），此点未确认。

## 9. 亮点与风险

**亮点**

1. "从 0 实现"名副其实：无任何 Agent 框架依赖，Agent 循环、工具调用、状态、重试全部手写，是学习多 Agent 系统内部机制的高质量参考实现（全仓库 grep + `QueryEngine/agent.py` 佐证）。
2. ForumEngine 的"日志黑板 + LLM 主持人"协作机制新颖且实现轻量，为多 Agent 异步协作提供了不依赖消息队列的工程范式（`ForumEngine/monitor.py`、`utils/forum_reader.py`）。
3. 报告生成引入 IR 中间表示与契约校验，把 LLM 输出的不确定性约束进 17 种块类型的强 Schema，再确定性渲染，是"LLM 生成结构化文档"的成熟做法（`ReportEngine/ir/schema.py`）。
4. 复合分析引擎：LLM Agent 之外融合本地情感模型（5 套自研微调/传统 ML 模型随仓附带权重与数据集）、Qwen 关键词优化中间件，多模型协同而非单 LLM 包打天下（`SentimentAnalysisModel/`、`InsightEngine/tools/`）。
5. 一键部署体验完整：Docker 双服务编排、自动建表、双语文档、示例报告随仓（`docker-compose.yml`、`final_reports/`）。

**风险 / 局限**

1. **数据合规**：舆情数据来自对微博、知乎、小红书、抖音等平台的爬取（MediaCrawler 子模块，`platform_crawler.py`），此类抓取与平台服务条款及个人信息保护法规（如 PIPL）的边界需使用者自行评估，项目将合规责任留给部署者；本文档不构成使用指导。
2. 运行成本高：一次完整分析需 7 组独立 LLM API（4 个 Agent + MindSpider + 主持人 + 关键词优化器）加多轮反思循环与论坛辩论，token 消耗与配置门槛显著（`.env.example`）。
3. 架构单机化：Agent 协作基于同机日志文件，无法水平扩展；大量 `sys.path` hack 与 Windows 优先的路径假设使跨平台/容器化部署存在隐性脆弱点（各 Engine 源文件头部、`requirements.txt` 注释）。
4. 测试与 CI 短板：无 lint/测试 CI，核心 Agent 流程无自动化测试，报告质量依赖所配 LLM 能力（README 自述图表空白时需换更强模型）。

## 10. 总结评价

BettaFish 是目前中文开源社区里完成度最高的多 Agent 舆情分析系统之一：它用约 5.7 万行纯 Python 证明了"不用框架也能把多 Agent 系统做扎实"，论坛协作、IR 报告、复合模型三块设计都有独到之处，且部署文档、示例、双语支持一应俱全，难怪能在一年半内积累 4 万+ 星。代价是重成本运行、单机架构与合规自担。它适合两类人：想直接拿来跑舆情/品牌分析的终端用户（需自备 API Key 并自行评估数据来源合规性），以及想吃透多 Agent 系统工程细节的开发者——后者的最佳路径是依次精读 `QueryEngine/agent.py`（Agent 循环）、`ForumEngine/monitor.py`（协作机制）与 `ReportEngine/ir/schema.py`（结构化输出约束），堪称一套"无框架 Agent 系统"的活教材。
