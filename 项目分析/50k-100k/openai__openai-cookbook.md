# openai/openai-cookbook 项目分析

> 星数 76777（快照 2026-10-24）｜主语言 Jupyter Notebook｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

OpenAI 官方 Cookbook：围绕 OpenAI API 的可运行示例代码与实践指南合集，是 [cookbook.openai.com](https://cookbook.openai.com) 官方站点的源仓库。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | openai/openai-cookbook | 仓库元数据 |
| 星数 | 76777（fork 13026，watch 1010，open issue 343） | 快照 + `gh api` |
| 主语言 | Jupyter Notebook（另有大量 Python/Markdown） | 清单 + 文件统计 |
| 许可证 | MIT（Copyright 2025 OpenAI） | `LICENSE` |
| 创建时间 | 2022-03-11 | `gh api repos/openai/openai-cookbook` |
| 最近推送 | 2026-10-29（分析日前 2 天） | `gh api` 同上 |
| 贡献者 | 约 382 人 | `gh api contributors` 分页计数 |
| 仓库体积 | GitHub 记录约 959MB（含 1838 张图片、66 个 PDF、音频等） | `gh api` size 字段 + tarball 统计 |

## 2. 项目是什么，解决什么问题

Cookbook 官方自述是 "Example code and guides for accomplishing common tasks with the OpenAI API"（`README.md`）。API 文档讲"接口怎么调"，Cookbook 讲"任务怎么做成"：把嵌入检索、微调、函数调用、Agent 编排、评测、语音、图像等常见工程任务写成可从头跑到尾的 notebook 与长文，读者配好 `OPENAI_API_KEY` 环境变量即可复现（`README.md`）。

内容分三类：一是 `examples/` 下的 272 个 Jupyter notebook + 490 个 Python 脚本，覆盖单点任务（如 `examples/Classification_using_embeddings.ipynb` 用嵌入做评论分类）；二是 `articles/` 下的长篇方法论文（如 `articles/techniques_to_improve_reliability.md` 讲提示可靠性技术、`articles/openai-harmony.md` 讲模型通信格式）；三是 `examples/partners/` 下与 AWS（Bedrock/AgentCore）、Databricks、deepsense.ai 等伙伴合著的端到端方案（如 `examples/partners/AWS/policy_to_review_prior_authorization_agents_sdk_agentcore.ipynb` 的医疗预授权审查 Agent）。

作为官方仓库，它还承担"新模型/新 API 的首发教材"角色：GPT-5、GPT-5.1、GPT-5.2 三代模型的 prompting guide 已同时收录在 `examples/gpt-5/`（gpt-5-2_prompting_guide.ipynb 开篇即介绍 GPT-5.2 定位），Agents API/SDK、Responses API、Deep Research API、Codex、MCP、实时语音等新表面都有专章（目录见 `examples/`）。每篇内容通过 `registry.yaml` 注册后自动发布到 cookbook.openai.com。

对该仓库为何能积累 7.6 万星：其一，它是 OpenAI 官方四大代码仓库之一，且从 GPT-3 时代的嵌入教程起就是社区事实上的"第二官方文档"；其二，教程型仓库天然被教程、博客、课程反复引用，星数随 LLM 开发者总量水涨船高；其三，每个新模型发布周期都会带来一波新增内容与曝光（registry 条目按年分布：2022 年 27、2023 年 97、2024 年 63、2025 年 86、2026 年前 10 月 50，来源 `registry.yaml`）。

## 3. 技术栈与依赖

- **主体格式**：Jupyter Notebook（272 个 .ipynb）+ Markdown/MDX（`examples/` 内 139 个 .md、`articles/` 24 个文件），叙事型 notebook 占比高（如 `examples/gpt-5/gpt-5-2_prompting_guide.ipynb` 18 个 cell 中 17 个为 markdown，仅 1 个代码 cell）。
- **示例代码语言**：以 Python 为主（490 个 .py），少量 Node/TypeScript（48 个 .mjs、46 个 .ts、44 个 .js，多在实时语音与 usage-budget 类示例中）。
- **依赖管理**：无统一 requirements，采用"每示例自带依赖"模式，全仓共 16 个 `requirements*.txt`（如 `examples/gpt-5/prompt-optimization-cookbook/requirements.txt`）；官方建议虚拟环境 + 按需安装（`AGENTS.md`、`CONTRIBUTING.md`）。
- **关键第三方库**：`openai` 官方 SDK、`nbformat`（CI 结构校验）、各向量数据库客户端（`examples/vector_databases/` 覆盖 Pinecone、Weaviate、Qdrant、Milvus、Chroma、Elasticsearch、Redis、MongoDB Atlas 等 23 家）、LangChain/LlamaIndex（第三方集成示例）。
- **发布链路**：`registry.yaml`（325 条，含 title/path/slug/description/date/authors/tags）+ `authors.yaml`（141 位作者的自定义署名）驱动静态站点生成；schema 定义在 `.github/registry_schema.json` 与 `.github/authors_schema.json`。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `examples/` | 主体：36 个主题子目录 + 80 余个历史遗留的顶层 notebook |
| `examples/agents_api/` `agents_sdk/` | Agents API（156 文件，含 AWS Lambda MicroVM 沙箱）与 Agents SDK（86 文件）示例 |
| `examples/gpt-5/` `o1/` `o-series/` | 推理模型系列 prompting guide 与参数指南 |
| `examples/partners/` | 与 AWS、Databricks、deepsense.ai 等合著的端到端方案（246 文件） |
| `examples/vector_databases/` | 23 个向量库的检索集成教程 |
| `examples/audio/` `voice_solutions/` | 实时语音、会议智能、语音 Agent 评测（合计 289 文件） |
| `examples/chatgpt/` | ChatGPT 连接器、Actions 库（gpt_actions_library，29 篇）、管理员 API |
| `examples/evals/` | 官方评测方法论（图像生成评测、realtime 评测含 pytest harness） |
| `examples/data/` | 示例数据集（216 文件） |
| `articles/` | 长篇方法论文章（可靠性技术、harmony 格式、gpt-oss 部署指南等） |
| `images/` | 站点与文章共用图示（565 文件） |
| `.github/` | CI workflow、notebook 校验脚本、issue/PR 模板、schema |
| `registry.yaml` / `authors.yaml` | 站点内容注册表与作者署名表 |
| `.codex/skills/docs-editor/` | 仓库自带的 Codex 编辑审校 skill |

组织方式评述：这是"注册表驱动的多作者内容仓"，不是软件项目。扁平 + 主题目录混合：2022-2023 年的教程散在 `examples/` 顶层（面向 notebook 的相对路径链接），2024 年后新内容一律进 `examples/<topic>/` 并要求注册 registry（`CONTRIBUTING.md` 明确规定放置规则）。单一 CODEOWNERS（`.github/CODEOWNERS`：`* @openai/developer-experience`）意味着所有合并由 OpenAI DevEx 团队把关，外部 PR 是内容供给渠道而非协作开发。

## 5. 核心架构与关键实现

**（1）registry.yaml：内容即数据库。** 每条要发布的内容必须在 registry 中登记，字段由 JSON Schema 约束：

```yaml
- title: GPT Actions library - BigQuery
  path: examples/chatgpt/gpt_actions_library/gpt_action_bigquery.ipynb
  slug: gpt-action-bigquery
  date: 2024-07-09
  authors:
    - aaronwilkowitz-openai
  tags:
    - gpt-actions-library
```

325 条注册项中 239 条指向 notebook、56 条指向 md、5 条 mdx（`registry.yaml` 统计）。tag 体系即站点的分类导航：最高频为 embeddings(99)、completions(94)、chatgpt(38)、responses(35)、agents(35)、evals(33)。

**（2）CI 校验脚本。** `.github/scripts/check_notebooks.py` 用 `git diff --name-only base -- '*.ipynb'` 取变更 notebook，逐个用 `nbformat.read(as_version=4)` 验证结构合法性，非法即退出码 1。对应 workflow `.github/workflows/validate-notebooks.yaml` 在 PR 上运行，另设 windows-latest 作业校验 NTFS 路径安全 checkout——这是历史踩坑（含 Windows 非法文件名的示例）后的产物，`validate-usage-budget.yml` 里还有注释说明"其他 Cookbook 示例含 Windows 无法 checkout 的文件名"，因此该示例改用 `git archive` 导出后做三 OS 矩阵测试。

**（3）AI 辅助审校工作流。** 仓库为 AI 编码 agent 写了两份规则：`AGENTS.md`（目录规范、命名规范、优先级 0 审查项、以及"Recent Learnings"沉淀的实操教训，如 uv 虚拟环境继承问题、pytest sys.path 冲突）；`.codex/skills/docs-editor/SKILL.md` 则定义了一个编辑审校 skill，要求编辑 notebook 时"只改 markdown cell，保护代码 cell、输出、执行计数、元数据不被重序列化"。CONTRIBUTING.md 要求 PR 合并前对每个改动文件跑一遍该 skill——官方仓库用自家 Codex 做内容质检的实例。

**（4）评测 harness 级示例。** 部分新示例已超出 notebook 范畴，是完整的小型工程：`examples/gpt-5/prompt-optimization-cookbook/` 含 `run_FailSafeQA.py`、`llm_as_judge.txt` 评分提示词、基线/优化后的结果 CSV 与目录结构；`examples/evals/realtime_evals/` 按带 pytest 的测试布局组织（`AGENTS.md` 的 Recent Learnings 记录了其 `shared/__init__.py` 导入修复过程）。反映 Cookbook 内容正从"教学 notebook"向"可复现评测基准"演进。

## 6. 构建与运行

来自 `CONTRIBUTING.md` 与 `AGENTS.md` 的实际命令（本分析未执行，仅静态摘录）：

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r examples/<topic>/requirements.txt   # 按示例安装依赖
jupyter lab                                        # 或 jupyter notebook 交互运行
python .github/scripts/check_notebooks.py          # 提交前校验 notebook 结构
```

- 运行任何示例需设置 `OPENAI_API_KEY` 环境变量，或在仓库根放 `.env` 文件（`README.md`）。
- 无顶层统一依赖文件、无构建产物；每个示例各自声明依赖，大数据集不进仓库、在 notebook 中说明获取方式（`CONTRIBUTING.md`）。
- 密钥管理规范：只读环境变量，禁止硬编码（`CONTRIBUTING.md`、`AGENTS.md`）。

## 7. 内容规模、分类与收录机制

**规模**：272 个 notebook、490 个 Python 脚本、139 个示例 README/md、24 个 article 文件、16 个 requirements 文件；tarball 803MB，压缩前约 1GB，其中 jpg/png 共 1838 张、PDF 66 个、音频（wav/mp3）84 个——媒体资源是体积主体。

**分类体系**：双层。文件系统层按 API 表面/场景分目录（agents、gpt-5、audio、vector_databases、partners 等 36 个主题目录）；站点层按 registry tags 分类（embeddings、agents、evals、security、fine-tuning、open-models 等约 200 个标签）。

**收录与更新机制**：PR 贡献制 + 人工审校。流程为：issue 讨论 → PR 提交 notebook/article + registry 条目 → 依 `.github/pull_request_template.md` 自评 6 项 rubric（相关性/独特性/拼写/清晰/正确/完整，各 1-4 分，全部 ≥3 才可接受）→ DevEx 团队 review（best-effort，无承诺时限，`CONTRIBUTING.md`）。CI 只做结构性校验（nbformat 合法性、Windows checkout、单个示例的跨 OS 离线测试），不执行 notebook 本体；正确性靠作者"从头到尾跑通、清空执行计数"的自证。

**生态位与姊妹项目**：官方生态中，API 文档（developers.openai.com）讲规范，`openai/openai-python`/`openai-node` 是 SDK，`openai/evals` 是评测框架，Cookbook 是"应用层教程"；`articles/related_resources.md` 主动收录 LangChain、LlamaIndex、Guidance、Semantic Kernel 等第三方资源，定位为生态入口而非竞争者。

## 8. 社区与维护状态

- **活跃度**：最近推送 2026-10-29，分析窗口内每周多个合并（近 10 条提交横跨 10-09 至 10-29，含新示例、暗色模式修复、依赖清理等，`gh api commits`）；2026 年前 10 个月新增 registry 条目 50 个，更新节奏稳定。
- **贡献者结构**：约 382 名贡献者；`authors.yaml` 显示内容作者是混合编制——OpenAI 员工（slug 带 `-openai` 后缀，如 sperry-openai、dwigg-openai）与外部/伙伴作者（如 Databricks 的 HsingTang、deepsense.ai 团队）各占相当比例。
- **治理**：单一 CODEOWNERS 要求每个 PR 都由 `@openai/developer-experience` 批准；issue 模板分 problem report / feature request / get-help 三类；有 stale bot（`.github/workflows/stale.yaml`）。
- **未确认项**：具体月度 PR 合并量、issue 响应时长未统计；release 无（内容仓不打版本号，以 main 分支 + 站点部署为准）。

## 9. 亮点与风险

**亮点**

1. 官方背书 + 新品首发：GPT-5.2、Agents API、Deep Research、Codex 等发布即有配套指南（`examples/gpt-5/`、`examples/agents_api/`），是学习新表面最快的一手材料。
2. registry.yaml 的"内容即配置"设计：分类、署名、发布日期全部结构化，站点生成与内容解耦，多作者协作有序（`registry.yaml`、`.github/registry_schema.json`）。
3. 工程化审校：rubric 自评 + DevEx 统一把关 + nbformat CI + 自带 Codex docs-editor skill，在内容仓里少见地系统化（`.github/`、`.codex/skills/`）。
4. 覆盖广度：从 2022 年嵌入经典教程到 2026 年 Agent 沙箱/评测 harness，形成完整的 LLM 应用工程编年史；23 个向量库集成使其成为 RAG 选型的活地图（`examples/vector_databases/`）。

**风险/局限**

1. 内容腐化不可避免：老教程仍引用已退役模型与旧 API（如 `articles/techniques_to_improve_reliability.md` 使用 gpt-3.5-turbo-instruct，completions 相关条目 94 篇），仓库不做大规模下架，读者需自行分辨时效。
2. 无自动化正确性保障：CI 不执行 notebook，示例能否跑通取决于模型/SDK 版本漂移，343 个 open issue 中相当部分预计是"跑不通"类问题（issue 构成未逐条核实，标注未确认）。
3. 审校无时效承诺："best-effort，无法保证何时或是否合并"（`CONTRIBUTING.md`），外部贡献可能长期滞留。
4. 仓库体积近 1GB，媒体资源直接入仓，clone/tarball 成本高，对低带宽用户不友好。

## 10. 总结评价

openai-cookbook 是 LLM 时代最成功的官方教程仓库之一：它把"如何用 OpenAI API 做成一件事"沉淀为 325 篇结构化注册、持续更新的可运行内容，从嵌入分类一路写到 GPT-5.2 prompting 与 Agent 评测 harness，本身也是一份 LLM 应用工程演进的编年史料。它不是软件，读"源码"的意义在于读范例与工程化内容治理（registry 模式、rubric 审校、AI 辅助编辑 skill 都值得内容型团队借鉴）。适合三类人：正在接 OpenAI API 新特性的开发者（按 tag 找对应教程）、要构建 RAG/Agent/评测系统的工程师（partners 与 evals 目录的端到端方案）、以及运营技术内容仓库的团队（学它的组织与审校机制）。使用时注意甄别条目日期，优先读 2025 年后的内容。
