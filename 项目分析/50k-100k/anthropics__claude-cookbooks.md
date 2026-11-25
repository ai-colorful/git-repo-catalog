# anthropics/claude-cookbooks 项目分析

> 星数 53694（快照 2026-10-24）｜主语言 Jupyter Notebook｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

Anthropic 官方 Claude API 的 cookbook（食谱）示例集：以可复制运行的 Jupyter Notebook + 配套 Python 脚本形式，手把手演示如何用 Claude 构建分类、RAG、工具调用、多模态、Agent 等真实应用。

| 项 | 值 |
| --- | --- |
| 名称 | anthropics/claude-cookbooks（旧名 anthropics/anthropic-cookbook，README 内链仍用旧名，见 README.md） |
| 星数 | 53694（快照）；gh api 实时 54143 |
| 主语言 | Jupyter Notebook（103 个 .ipynb）+ Python（121 个 .py） |
| 许可证 | MIT（LICENSE，Copyright (c) 2023 Anthropic） |
| 创建时间 | 2023-08-15（gh api） |
| 最近推送 | 2026-11-02（gh api pushed_at），高频活跃 |
| 贡献者 | 86 人（gh api contributors 分页推算，authors.yaml 另登记约 121 个作者条目） |
| Release | 0 个（内容型仓库不发包） |
| 仓库体积 | 约 233MB，其中 images/ 占 141MB |

## 2. 项目是什么，解决什么问题

这是 Claude 官方文档之外的"可运行教材"。README.md 开篇定位："provide code and guides designed to help developers build with Claude, offering copy-able code snippets"。它解决的问题很明确：API 文档告诉你接口长什么样，而 cookbook 告诉你**怎么把它拼成一个能用的功能**——每个 notebook 聚焦一个概念，从问题陈述、前置条件到完整可执行代码和输出全部保留（CLAUDE.md 明确规定 "Keep outputs in notebooks (intentional for demonstration)"、"One concept per notebook"）。

收录范围覆盖 Claude API 的全部能力面（目录见仓库顶层）：基础能力（`capabilities/`：分类、RAG、摘要、内容审核、上下文嵌入、知识图谱、text-to-sql）、工具调用（`tool_use/`：计算器工具、客服 Agent、结构化 JSON 抽取、pydantic 工具、工具检索、记忆 cookbook、context engineering）、多模态（`multimodal/`：视觉入门、最佳实践、图表解读、表单转录）、高级技巧（`misc/`：prompt caching、批处理、citations、JSON mode、metaprompt、审核过滤器、admin API、会话记忆压缩）、性能与成本（`cost_optimization/`、`latency_optimization/` 两个端到端调优 notebook）、扩展思考（`extended_thinking/`）、评测（`evals/agentic_search/`）、微调（`finetuning/`）、可观测性（`observability/`：OpenTelemetry tracing、用量成本 API）。

组织方式是"目录即分类 + registry 索引"。每个顶层目录有自己的 README，根部的 `registry.yaml`（100 个条目）是结构化索引：每条含 title/description/path/authors/date/categories，供官方 cookbook 网站渲染；`authors.yaml`（约 121 条）把 GitHub 用户名映射到真名与头像；对应 schema 在 `.github/registry_schema.json`、`.github/authors_schema.json`，校验脚本在 `.github/scripts/verify_registry.py`。

从 registry.yaml 的分类统计看当前重心（按出现次数）：Agent Patterns 29、Tools 26、RAG & Retrieval 19、Claude Managed Agents 19、Integrations 15、Responses 13、Claude Agent SDK 10、Multimodal 9、Evals 9、Observability 6、Skills 5、Cybersecurity 4。Agent 类内容合计已过半，说明仓库已从早期"prompt 技巧集"演进为"Agent 工程教材"。

条目规模实测：103 个 .ipynb、121 个 .py、86 个 .md；registry.yaml 收录 100 个策展条目（每条带分类/作者/日期）；仓库总体积约 233MB，其中配图占六成。notebook 篇幅差异大——短教程 15-20 个单元（如 customer_service_agent.ipynb 15 单元），长教程 20+ 单元并配套子工程目录（如 claude_agent_sdk/ 各章节都带独立 pyproject 与 demo 目录）。

## 3. 技术栈与依赖

全部内容面向 Python（README："the code examples are primarily written in Python"），工程化管理（`pyproject.toml`）：

- 运行要求：Python >=3.11,<3.13；包管理用 uv（`uv.lock`、`uv.toml`），构建后端 hatchling，但 wheel 目标只是占位包 `anthropic_cookbook/`（目录内仅 `__init__.py`，pyproject 注释 "Dummy package for build system"）
- 核心依赖：`anthropic>=2.0.0`（官方 SDK）、`claude-agent-sdk>=0.2.168`（Agent SDK 章节）、`voyageai>=0.3.5`（嵌入）、`pymongo`、`pyjwt`、`numpy/pandas/matplotlib/networkx/rich/requests/python-dotenv`
- 开发链：`ruff`（lint+format，原生支持 .ipynb，见 pyproject `[tool.ruff]` extend-include）、`pytest`+`nbval`+`pytest-cov`、`tox`+`tox-uv`、`pre-commit`、`detect-secrets`（`scripts/detect-secrets/`）
- CI 侧：GitHub Actions 9 个 workflow（`.github/workflows/`），其中 3 个用 `anthropics/claude-code-action` 让 Claude 自己审 PR，认证用 Workload Identity Federation（OIDC）而非静态 API key（`claude-pr-review.yml` 注释明确）

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `capabilities/` | 基础能力：classification、RAG、summarization、content_moderation、contextual-embeddings、knowledge_graph、text_to_sql（7 nb） |
| `tool_use/` | 工具调用专题（14 nb）：customer_service_agent、memory_cookbook、threat_intel_enrichment_agent、context_engineering/、tool_search_with_embeddings 等 |
| `managed_agents/` | Claude Managed Agents 教程群（20 nb，CMA_* 系列）：数据分析 Agent、Slack bot、SRE 事件响应、prompt 版本回滚、安全审查、人在回路门控等，含 slack/sentry/linear/mongodb 集成子目录 |
| `claude_agent_sdk/` | Agent SDK 编号教程 00-08（10 nb）：一句话研究 Agent、幕僚长 Agent、可观测性、SRE、从 OpenAI Agents SDK 迁移、漏洞检测、托管、动态工作流 |
| `multimodal/` | 视觉能力（6 nb），配套 images/ 141MB（reading_charts_graphs 子目录独占 101MB） |
| `misc/` | 高级技巧杂项（15 nb）：prompt_caching、batch_processing、using_citations、how_to_enable_json_mode、metaprompt、session_memory_compaction 等 |
| `patterns/agents/` | 经典 Agent 模式（5 nb）：orchestrator_workers、evaluator_optimizer、async_multi_agent_orchestration 等 |
| `third_party/` | 生态集成：Pinecone、MongoDB、VoyageAI、LlamaIndex、Deepgram、ElevenLabs、WolframAlpha（12 nb） |
| `cost_optimization/`、`latency_optimization/` | 成本/延迟调优各 1 个长 notebook |
| `evals/`、`tool_evaluation/`、`extended_thinking/`、`finetuning/`、`observability/`、`skills/`、`coding/`、`fable_5_fallback_billing/` | 各 1-2 个专题 nb |
| `tests/`、`scripts/`、`.github/`、`.claude/` | 质量工程与自动化（见第 7 章） |
| `registry.yaml`、`authors.yaml` | 面向 cookbook 网站的结构化索引与作者登记 |
| `images/` | notebook 配图（141MB） |

组织评述：这是"内容 monorepo"——没有应用入口，全部产出即仓库本身；代码组织围绕教学单元而非软件模块。目录即分类、文件名即课题、registry.yaml 做机器可读总索引，三层一致。

## 5. 核心架构与关键实现

内容型仓库无传统"架构"，核心资产是代表性 notebook 与贯穿其中的工程范式：

**1）Managed Agents 教程系列（`managed_agents/CMA_iterate_fix_failing_tests.ipynb`，20 个 notebook 的入口篇）**。设计是"做-观察-修"闭环：给 agent 上传一个埋了两个 bug 的 `calc.py` 包，让它自己跑测试、读失败输出、修复。首个代码单元即展示当前推荐范式——

```python
import os
from pathlib import Path
from anthropic import Anthropic

MODEL = os.environ.get("COOKBOOK_MODEL", "claude-sonnet-5-5")
client = Anthropic()
FIXTURE = Path("example_data") / "iterate"
```

配套 `managed_agents/utilities.py`、`log_review_utils.py` 与 `cma-mcp/`、`self_hosted_sandboxes/` 等子工程。生产篇 `CMA_operate_in_production.ipynb` 覆盖 vault 凭据、webhook 人在回路、地域固定、资源生命周期。

**2）工具调用与记忆（`tool_use/`）**。`memory_cookbook.ipynb` + `memory_tool.py` 演示客户端记忆工具；`tool_search_with_embeddings.ipynb` 解决工具过多时的检索式选择；`context_engineering/context_engineering_tools.ipynb` 配 `research_corpus.py` 演示上下文工程。`customer_service_agent.ipynb` 是最经典的客户端工具完整案例（查客户、查订单、退单）。

**3）性能调优方法论（`cost_optimization/cost_optimization.ipynb`、`latency_optimization/latency_optimization.ipynb`）**。README 描述的方法是：在同一个 agent 上做 checklist 式实验，度量"每任务成本×通过率"找 Pareto 最优配置，度量 TTFT/任务时长/准确率选最快且过质量线的配置——把调优做成了可复现实验。

**4）质量工程基础设施（`tests/notebook_tests/`，1003 行 Python）**。`test_notebooks.py`（431 行）对每个 notebook 做：JSON/nbformat 合法性、kernelspec 存在且为 Python kernel、全部代码单元已执行、按序执行、无 error 输出、无硬编码密钥、pip 依赖可检测；`--execute-notebooks` 开关可真实执行（pytest markers：slow / integration / mutates_state，见 pyproject）。这套测试让 103 个 notebook 的"文档即代码"成为可 CI 的资产。

## 6. 构建与运行

来自 CLAUDE.md、CONTRIBUTING.md、Makefile（真实可用命令）：

```bash
# 环境搭建（uv 优先）
uv sync --all-extras          # 安装全部依赖
uv run pre-commit install     # 钩子
cp .env.example .env          # 配置 ANTHROPIC_API_KEY

# 质量检查
make check                    # ruff format --check + lint
make test                     # pytest（单元/结构测试）
make test-notebooks           # notebook 结构测试（快，无 API 调用）
make test-notebooks-exec      # notebook 真实执行测试（慢，需 API key）
make test-notebooks-tox       # tox 隔离环境
make test-notebooks NOTEBOOK=tool_use/calculator_tool.ipynb   # 单个

# 读者视角：直接在 Jupyter 中打开对应 .ipynb 顺序执行即可
# 例如 multimodal/getting_started_with_vision.ipynb
```

## 7. 内容质量保障与更新机制（内容型特例）

**更新机制是"registry 驱动 + Claude 自审"的双重流水线**：

- 新增 recipe 走 PR：`.claude/commands/add-registry.md` 提供"登记新条目"的标准命令；`verify-authors.yml` 强制 authors.yaml 与 registry.yaml 一致且按字母排序（`scripts/validate_authors_sorted.py --fix`，Makefile `sort-authors`）；`.github/scripts/verify_registry.py` 按 schema 校验索引
- CI 按改动增量触发：`notebook-tests.yml` 只对 PR 中变更的 .ipynb 运行（git diff 取变更列表）；`notebook-quality.yml` 对 notebook 跑 ruff；`links.yml` 用 lychee 校验外链（`lychee.toml` 在根部）
- Claude 审查 Claude 的内容：`claude-pr-review.yml`、`claude-link-review.yml`、`claude-model-check.yml` 三个 workflow 用 claude-code-action 审查 PR；`.claude/skills/cookbook-audit/` 内置打分 rubric（style_guide.md 定义了"问题导向引言 + 终结性/使能学习目标（TLO/ELO）"的教学模板）和 `validate_notebook.py` 自动检查脚本；`.claude/agents/code-reviewer.md` 定义专职审查 agent
- 密钥卫生：`scripts/detect-secrets/` 自定义插件 + baseline，测试层再做一遍 `validate_no_hardcoded_secrets`

这套机制的水平显著高于一般内容仓库：文档仓库有结构测试、执行测试、密钥扫描、链接检查、AI 审查五层防线。

## 8. 社区与维护状态（含生态位）

- **活跃度**：gh api pushed_at 2026-11-02（分析前两天仍在推送）；377 个 open issue；fork 6545。registry.yaml 中最新条目日期 2026-04-08/04-10（Managed Agents 系列），说明索引持续追加
- **贡献者结构**：86 位 GitHub 贡献者（gh api 分页推算），authors.yaml 登记约 121 位作者（含大量 `-ant` 后缀的 Anthropic 内部员工账号，如 charmaine、pauly-ant、markn-ant），官方主导、社区辅助；PR 流程见 CONTRIBUTING.md（要求 conventional commits、分支命名 `<username>/<feature>`）
- **节奏**：0 个 release——内容仓库以"main 即最新版"运行；模型别名策略（CLAUDE.md 规定只用非日期别名 `claude-opus-5-5`/`claude-sonnet-5-5` 等）让内容随模型滚动保持不过期
- **生态位与姊妹项目**：同在 anthropics 组织下的 `courses`（README 引导新手先学 anthropic_api_fundamentals 课程）、`claude-agent-sdk`（本仓库 `claude_agent_sdk/` 章节为其配套教程）；third_party/ 与 Pinecone、MongoDB、VoyageAI、LlamaIndex、Deepgram、ElevenLabs、WolframAlpha 互为生态引流；README 还指向 aws-samples/anthropic-on-aws

**为什么能涨到 5 万+ 星**：其一，它是 Claude 生态的"官方第一入口"——从文档、课程到 Discord 全链路引流至此，每个新模型/新 API（Managed Agents、Agent SDK、Skills、prompt caching）发布都会带来一波学习流量；其二，内容本身可运行、带输出、按概念切分，实操价值远超普通 awesome 清单；其三，Anthropic 内部作者持续高频供给（registry 近期条目几乎全是内部作者），保证内容与产品同频。

## 9. 亮点与风险

**亮点**

1. 官方出品、覆盖完整且持续更新：103 个 notebook 覆盖从 prompt caching 到 Managed Agents 的全部 API 面，2026-11 仍在推送（gh api）
2. "文档即代码"做到了工程级：结构测试+执行测试+密钥扫描+外链检查+ruff 五层 CI（`.github/workflows/`、`tests/notebook_tests/`）
3. registry.yaml 机器可读索引 + schema 校验 + 作者登记，内容可直接驱动官网渲染（`.github/registry_schema.json`）
4. 用 Claude 审查 Claude 内容的自指式质量闭环（`.claude/skills/cookbook-audit/`、`claude-pr-review.yml`），且认证用 WIF/OIDC 免静态密钥
5. 教学设计有方法论：cookbook-audit style_guide.md 的 TLO/ELO 模板、一 notebook 一概念、保留输出

**风险/局限**

1. 仓库体积 233MB（images/ 141MB，reading_charts_graphs 独占 101MB），clone/浏览成本高
2. 依赖绑定 Anthropic API 且模型别名随版本滚动（claude-opus-5-5 等），旧输出与最新模型行为可能不一致，读者需自备 API key 且产生真实费用
3. README.md 内链大量使用旧仓库名 `anthropics/anthropic-cookbook`（依赖 GitHub 重定向），存在链接维护债务（`links.yml` 外链检查不覆盖仓内旧名）
4. 无版本化发布（0 release），引用具体内容时只能锚定 commit，教学代码随 main 漂移

## 10. 总结评价

这是官方文档与真实工程之间缺失的那一环：Anthropic 把"如何用 Claude 做成一件事"沉淀成一百多个可运行、可测试、可索引的 notebook，并用一套罕见于内容仓库的工程体系（结构/执行测试、registry 索引、Claude 自审、密钥扫描）保证它持续不过期。它不教你读源码——它本身就是"源码级教材"。适合三类人：正在接 Claude API 的工程师（直接抄 `tool_use/`、`managed_agents/` 的完整范式）、想系统学 Agent 工程的学习者（按 `claude_agent_sdk/00-08` 编号路径走）、以及任何想给自家 API 做 cookbook 的平台团队（抄它的质量工程与 registry 机制）。对研究者而言，`cost_optimization/` 与 `latency_optimization/` 的 Pareto 调优实验设计也值得单独一读。
