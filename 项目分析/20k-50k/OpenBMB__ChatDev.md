# OpenBMB/ChatDev 项目分析

> 星数 26500（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-21
>
> 注：main 分支已是 **ChatDev 2.0（DevAll）**（2026-01-07 发布的零代码多智能体编排平台）；经典 ChatDev 1.0（虚拟软件公司）整体迁移至 `chatdev1.0` 分支维护（README.md News 区）。本文以 main 分支为主体、以 `chatdev1.0` 分支为对照进行分析（两分支源码均已实际下载阅读）。

## 1. 项目速览

一句话定位：清华 OpenBMB 出品的多智能体协作框架——1.0 版以"虚拟软件公司"实现自然语言驱动的软件开发全流程（ACL 2024 论文配套），2.0 版（DevAll）演化为通过 YAML 配置编排任意多智能体工作流的通用平台。

| 项目 | 值 |
| --- | --- |
| 名称 | OpenBMB/ChatDev（2.0 对外名 DevAll，pyproject.toml `name = "DevAll"`） |
| 星数 | 快照 26500（2026-10-24）；gh api 核实 2026-11-21 为 34829（2.0 发布后快速增长） |
| 语言 | Python（后端）+ TypeScript/Vue（前端） |
| 许可证 | Apache-2.0（根目录 LICENSE、pyproject.toml 均确认） |
| 默认分支 | main（2.0）；1.0 遗留分支 `chatdev1.0`；另有 `macnet`、`puppeteer` 论文实验分支 |
| 最近活跃 | 最新提交 2026-10-30，repo pushed_at 2026-11-06（gh api 核实） |
| Forks / Open Issues | 4352 / 129（gh api） |
| 主页 | https://arxiv.org/abs/2307.07924 （论文链接即 homepage） |

## 2. 项目是什么，解决什么问题

ChatDev 起源于 2023-07 清华 NLP 实验室的论文 *Communicative Agents for Software Development*（arXiv:2307.07924，后发表于 ACL 2024），核心思想是：把软件开发建模为一家"虚拟软件公司"，由多个 LLM 智能体扮演 CEO、CTO、程序员、评审员等角色，通过"指导者—助手"双角色对话（instructor/assistant chatting）完成需求分析、编码、测试、文档等整个软件生命周期（README.md Overview 区、chatdev1.0 分支 wiki.md）。

ChatDev 1.0 的技术贡献是三层可定制抽象：ChatChain（阶段流水线）、Phase（会话阶段）、Role（角色 prompt），全部通过 `CompanyConfig/` 下的 JSON 配置即可定制，用户一行命令 `python3 run.py --task [想法]` 即可产出可运行的软件（chatdev1.0 分支 run.py、wiki.md）。它是"沟通型智能体协作（communicative agent collaboration）"范式的奠基项目之一。

2026-01-07 发布的 ChatDev 2.0（DevAll）把方向从"专用软件开发"扩展为"开发一切"：一个零代码多智能体编排平台，用户通过 YAML 定义节点、边、循环与子图，即可搭建数据可视化、3D 生成、深度调研等任意多智能体系统（README.md Overview；yaml_template/design.yaml）。经典 1.0 移入 `chatdev1.0` 分支，且其"虚拟软件公司"流程已被官方复刻为 2.0 的示例工作流（`yaml_instance/ChatDev_v1.yaml`，图中可见 Programmer 节点沿用原 1.0 角色prompt）。

围绕该项目还有一串论文产出线，均落在独立分支：MacNet（DAG 拓扑协作，arXiv:2406.07155，`macnet` 分支）、Puppeteer（RL 可学习编排器，NeurIPS 2025，arXiv:2505.19591，`puppeteer` 分支）、体验式共学习 ECL/IER（arXiv:2312.17025 / 2405.04219，见 README.md News 区）。仓库还附带 MultiAgentEbook 多智能体论文电子书目录（README.md 2024-06-25 条目）。

## 3. 技术栈与依赖

**ChatDev 2.0（main 分支）**（依据 pyproject.toml、frontend/package.json）：
- 后端：Python 3.12（requires-python ">=3.12,<3.13"），依赖管理用 uv（uv.lock）
- Web 框架：FastAPI 0.124.0 + uvicorn + websockets（pyproject.toml dependencies）
- LLM 接入：openai 官方 SDK（Responses API）+ google-genai（Gemini）；MCP 生态 mcp / fastmcp
- 记忆与检索：faiss-cpu（向量检索）、mem0ai（长期记忆，pyproject 注明 >=1.0.9）
- 工具链依赖：pandas、matplotlib、networkx、seaborn、xhtml2pdf、ddgs（搜索）等（服务生成型工作流）
- 前端：Vue 3.5 + Vite + vue-router + vue-i18n + @vue-flow（工作流画布），根 package.json 仅有 kill-port，前端独立于 frontend/package.json

**ChatDev 1.0（chatdev1.0 分支）**（依据 requirements.txt）：
- openai==1.47.1、tiktoken、Flask==2.3.2 + Flask-SocketIO（可视化器）、faiss-cpu、Wikipedia-API、colorama
- 特色：将 CAMEL 框架（camel-ai）整个 vendor 进仓库 `camel/` 目录并改造（如 camel/model_backend.py 直接 import chatdev.statistics 记账），而非依赖 pip 包

## 4. 目录结构与代码组织

**main 分支（ChatDev 2.0）** 顶层（实际 ls）：

| 目录/文件 | 职责 |
| --- | --- |
| `workflow/` | 图编排核心：graph.py（GraphExecutor 约 860 行）、graph_manager.py、topology_builder.py、cycle_manager.py、executor/（DAG/循环/并行执行器） |
| `entity/` | 数据模型：messages.py（Message 数据类）、graph_config.py、enums.py、configs/ |
| `runtime/node/` | 节点运行时：executor/（agent/human/python/subgraph/literal 等执行器）、agent/（providers/thinking/memory/tool/skills） |
| `runtime/edge/` | 边的条件与处理器 |
| `yaml_template/` + `yaml_instance/` | YAML schema 模板与 43+ 示例工作流（含 subgraphs/） |
| `schema_registry/` | YAML schema 注册与导出（run.py 启动时 ensure_schema_registry_populated） |
| `functions/` | 边条件函数与内置 function-calling 工具实现（function_calling/：web.py、code_executor.py、file.py、weather.py 等） |
| `server/` | FastAPI 应用：routes/ 11 个路由（execute、websocket、sessions、artifacts、batch 等）+ services/ |
| `frontend/` | Vue 3 Web 控制台（WorkflowView、WorkflowWorkbench、BatchRunView 等 7 个页面） |
| `check/` | YAML/工作流静态校验器 |
| `tools/`、`mcp_example/` | 辅助脚本与 MCP server 示例 |
| `docs/user_guide/` | 中英双语用户手册（nodes、modules、execution_logic、web_ui_guide 等） |
| `WareHouse/` | 运行产物输出根目录（run.py 中 `OUTPUT_ROOT = Path("WareHouse")`） |
| `tests/` | pytest 测试（9 个文件，约 2306 行） |

**chatdev1.0 分支** 顶层：`chatdev/`（10 个核心文件共 2254 行）、`camel/`（vendor 的 CAMEL 框架）、`CompanyConfig/`（Default/Art/Human/Incremental 四套配置 + 60+ 历史生成样例）、`WareHouse/`（68 个生成产物目录）、`visualizer/`（Flask 日志可视化）、`run.py`（CLI 入口，142 行）、`wiki.md`。

组织方式评述：单仓库承载"新平台 + 遗留版 + 论文实验分支"，通过分支隔离代际；main 分支内部是"Python 后端 + Vue 前端"的同仓前后端结构，配 compose.yml 双容器编排；1.0 则是典型的单 Python 包 + vendor 第三方库结构。

## 5. 核心架构与关键实现

### 5.1 ChatDev 1.0：ChatChain / Phase / Role 三层抽象（chatdev1.0 分支）

**ChatChain（顶层调度）**：`chatdev/chat_chain.py`（365 行）。构造函数读入三份 JSON（ChatChainConfig.json / PhaseConfig.json / RoleConfig.json），`self.chain = self.config["chain"]` 即阶段流水线；`execute_step()`（chat_chain.py:121）按 phase 名动态 import 类并实例化；`pre_processing()` 里完成"招聘"（按 recruitments 逐角色开会招募）、`post_processing()`（chat_chain.py:256）负责统计代码量、可选 git 提交、把日志移入 `WareHouse/<项目名>_<组织>_<时间戳>/`。

**Phase（阶段/会话单元）**：`chatdev/phase.py`（652 行）定义 12 个 SimplePhase（DemandAnalysis、LanguageChoose、Coding、ArtDesign、CodeComplete、CodeReviewComment/Modification、TestErrorSummary/Modification、EnvironmentDoc、Manual 等，见 grep class 结果）；`chatdev/composed_phase.py` 定义 ComposedPhase（CodeCompleteAll、CodeReview、Test、HumanAgentInteraction），通过 `cycleNum` 循环执行子 Phase，`break_cycle()` 决定何时退出循环。Phase 执行骨架（phase.py:278 `execute`）：先 `update_phase_env` 从全局环境取上下文，再 `chatting` 进行双角色对话，最后 `update_chat_env` 写回结论。

**Role（角色定义）**：`CompanyConfig/Default/RoleConfig.json` 中 9 个角色的 prompt 文本——Chief Executive Officer、Counselor、Chief Human Resource Officer、Chief Product Officer、Chief Technology Officer、Programmer、Code Reviewer、Software Test Engineer、Chief Creative Officer（实际解析 JSON 确认）；CEO 的 prompt 以 "You are Chief Executive Officer... decision-maker..." 开头。Default/ChatChainConfig.json 的 `chain` 数组即流水线：DemandAnalysis → LanguageChoose → Coding → CodeCompleteAll(循环10) → CodeReview(循环3) → Test(循环3) → EnvironmentDoc → Manual（实际解析确认）。

**通信机制**：双 agent 角色扮演对话由 vendor 的 `camel/agents/role_playing.py` 的 RolePlaying 类承担（instructor/assistant 互为 user/assistant 轮流发言），每个 Phase 配置 assistant_role_name / user_role_name / phase_prompt 三元组（PhaseConfig.json，实际解析确认）。

**LLM 后端**：`camel/typing.py` 的 ModelType 枚举列出 GPT_3_5_TURBO/GPT_4/GPT_4_TURBO/GPT_4O/GPT_4O_MINI/STUB 等（实际源码）；`camel/model_backend.py` 的 OpenAIModel 支持 `BASE_URL` 环境变量指向任意 OpenAI 兼容端点；run.py 的 `--model` 参数暴露 GPT_3_5_TURBO/GPT_4/GPT_4_TURBO/GPT_4O/GPT_4O_MINI 五个选项（run.py:81-99）。

**可视化**：`visualizer/app.py` 是一个 Flask 服务（index/chain_visualizer/replay 页面），运行时通过 `send_msg()` POST 到 127.0.0.1 的 `/send_message` 收集消息，实现 Online log 模式与回放（实际源码确认）。

**产物组织**：WareHouse 目录以 `<项目名>_<组织名>_<时间戳>` 建目录，ChatEnv（chatdev/chat_env.py）维护 codes/requirements/manuals 并 `write_meta()` 写元信息；`CompanyConfig/` 下还保存了 60+ 历史生成样例（如 Calculator_THUNLP_20230628184918）作为 showcase（目录 ls 确认）。

### 5.2 ChatDev 2.0（DevAll）：YAML 图编排运行时（main 分支）

**设计即配置**：`yaml_template/design.yaml`（10KB）是完整的 YAML schema 注释模板，核心概念：`graph.nodes`（类型 model/agent/human/subgraph/python/passthrough/literal/loop_counter）、节点 config 支持 variant（model 配置含 provider/base_url/api_key/tooling/thinking/memories/retry；agent 配置同理）、`vars` 变量与 `${API_KEY}` 环境变量插值（实际源码）。

**执行引擎**：`workflow/graph.py` 的 GraphExecutor（约 860 行）是核心。`_execute_node()`（graph.py:549）展示单节点执行全流程：资源守卫（resource_manager）→ 记录日志 → 动态边配置检查 → 调 executor → 输出包装为 Message → context_window 上下文清理 → 逐输出消息沿 outgoing edges 分发（`_process_edge_output`）。执行策略分层在 `workflow/executor/`：dag_executor.py（DAG 调度）、cycle_executor.py（循环）、parallel_executor.py（并行）、dynamic_edge_executor.py（条件边）、resource_manager.py。

**Agent 节点与思考/记忆**：`runtime/node/executor/agent_executor.py` 的 AgentNodeExecutor.execute 编排完整智能体流水线——ProviderRegistry 取 provider → pre_gen_thinking → 记忆检索 → 调模型 → post_gen_thinking → 记忆写回（agent_executor.py:46-160）。思考策略 `runtime/node/agent/thinking/self_reflection.py`；记忆体系 `runtime/node/agent/memory/`：simple_memory、file_memory、blackboard_memory（黑板）、mem0_memory、embedding.py（faiss 向量检索）。LLM Provider 注册在 `runtime/node/agent/providers/builtin_providers.py`：内置 "openai"（官方 SDK Responses API）与 "gemini"（google-genai）两个 provider；openai_provider.py 模型名为自由字符串（YAML `name` 字段），兼容任意 OpenAI 兼容端点，且会剥离 DeepSeek-R1/MiniMax 等推理模型的 `<think>` 块（openai_provider.py:400 注释）。

**工具层**：`functions/function_calling/` 实现内置工具（execute_code、web_search、文件读写、uv 环境管理、天气、视频等）；YAML tooling 支持 function / mcp_remote / mcp_local 三种接入（yaml_template/design.yaml tools 列表实测 25 个工具名）。人机协同由 human_executor.py + server/routes/websocket.py 支持人工介入节点。

**1.0 → 2.0 的对应关系**：`yaml_instance/ChatDev_v1.yaml` 把 1.0 虚拟软件公司复刻为 2.0 图——Code Complete All Phase Loop Counter（max_iterations: 5 对应原 cycleNum 10）、Test Phase Loop Counter（3 对应 3）、Programmer 等 agent 节点直接复用 1.0 角色文案（实际源码片段确认）。yaml_instance/ 还有 react.yaml、reflexion_product.yaml、deep_research_v1.yaml、blender_3d_builder_hub.yaml 等 40+ 场景示例与 3 个子图（subgraphs/）。

**与 MetaGPT 等同类框架的差异化（客观描述，基于两项目公开 README/论文定位）**：两者同属"虚拟软件公司"SOP 范式；ChatDev 1.0 的实现特点是完全配置驱动（JSON 定义 chain/phase/role）+ 双角色聊天驱动（CAMEL RolePlaying），MetaGPT 则强调角色/协议编程与结构化文档产物（PRD、设计文档）。2.0 转向通用图编排平台后，定位更接近 LangGraph/Dify 一类工作流引擎，差异点在于"零代码 YAML + 内置记忆/思考/工具全家桶 + 自家论文（MacNet/Puppeteer）的编排算法"。该对比依据双方 README 自述，未做基准测试验证。

## 6. 构建与运行

**ChatDev 2.0（main 分支，README.md Quick Start + Makefile 实测摘录）**：

```bash
uv sync                      # 后端依赖（Python 3.12 + uv）
cd frontend && npm install   # 前端依赖
cp .env.example .env         # 配置 API_KEY / BASE_URL
make dev                     # 后端 6400 + 前端 5173（Vite）
# 或手动：
uv run python server_main.py --port 6400 --reload
cd frontend && VITE_API_BASE_URL=http://localhost:6400 npm run dev
```

- CLI 直跑工作流：`uv run run.py --path yaml_instance/xxx.yaml --name <project>`（run.py 参数定义）
- Docker：`compose.yml` 提供 backend（Dockerfile target: runtime，端口 6400）+ frontend 双服务
- 校验：`make` 系列含 validate（tools/validate_all_yamls.py）；CI 亦跑同一校验

**ChatDev 1.0（chatdev1.0 分支，wiki.md Quick Start + run.py）**：

```bash
pip install -r requirements.txt
export OPENAI_API_KEY=...    # 可选 BASE_URL 指向兼容端点
python3 run.py --task "开发一个五子棋游戏" --name Gomoku --config Default --model GPT_4O
# 产物输出至 WareHouse/Gomoku_<org>_<时间戳>/
```

配置定制：修改 `CompanyConfig/<config>/ChatChainConfig.json`（chain/recruitments）、PhaseConfig.json（会话 prompt）、RoleConfig.json（角色人设），支持 Art/Human/Incremental 等预置组织（run.py get_config）。

## 7. 测试与工程质量

- **2.0 测试**：tests/ 共 9 个 pytest 文件约 2306 行（wc 实测），主题明显偏安全与稳定性：test_code_execution_security.py、test_local_tool_security.py、test_attachment_boundary_security.py、test_session_workspace_security.py、test_server_main_reload.py、test_mem0_memory.py 等（目录 ls 确认）。pytest 配置在 pyproject.toml（pythonpath=["."]，testpaths=["tests"]）。
- **CI**：`.github/workflows/` 仅一个 validate-yamls.yml（YAML 校验），无多平台矩阵测试（ls 确认）。测试覆盖面窄于代码量（198 个 py 文件 31302 行，find/wc 实测）。
- **1.0 测试**：无正式测试套件，仅 camel/typing.py 中保留 STUB 模型用于调试。
- **代码质量观察**：2.0 代码分层清晰（entity/workflow/runtime/server 四层），注释质量高（如 server_main.py 对 uvicorn StatReload 局限的处理说明、引用 issue #569）；server/routes 与 services 职责分明。1.0 代码高度配置化但缺少类型注解。2.0 pyproject 对 click 8.3 有兼容性 pin 注释，uv.lock 完整，工程化程度较高。

## 8. 社区与维护状态

- **Release 节奏**（gh api releases 实测）：v1.1.5（2024-08-28）→ v1.1.6（2024-11-12）→ 1.x 冻结，全力开发 2.0 → v2.0.0（2026-01-07）→ v2.1.0（2026-01-22）→ v2.2.0（2026-03-23）。2.0 发布后 3 个月内连发两版，节奏明显加快。
- **活跃度**：最新提交 2026-10-30（gh api commits），pushed_at 2026-11-06；近期提交来自 PR #700/#701（websocket 修复与本地工具安全加固），有社区 PR 参与（提交标题可见贡献者 NA-Wen）。
- **贡献者**：contributors API 返回 20（含匿名，未确认精确非匿名数）。项目由清华 NLP 实验室 / OpenBMB 社区主导，核心成员以 THUNLP 为主（生成样例目录后缀 *_THUNLP_* 与 README Developers 区佐证）。1.x 时期 SaaS 版曾由面壁智能（modelbest.cn）运营（README 2023-11-15 News）。
- **多分支科研产出**：macnet、puppeteer 等论文分支与主仓并行维护；NeurIPS 2025 论文（arXiv:2505.19591）实现明确落在 puppeteer 分支（README News）。
- Open Issues 129、Forks 4352（gh api）；星数自快照 26500 增至 34829（2026-11-21 核实），2.0 发布为主要增长引擎（未确认归因，仅时间相关性）。

## 9. 亮点与风险

**亮点**
1. 学术范式级贡献：ACL 2024 的沟通型智能体协作范式 + NeurIPS 2025 编排算法，论文与代码一一对应（论文分支/主分支均可查证），代码可直接复现论文（src10/chatdev/、puppeteer 分支声明）。
2. 全程配置驱动的可定制性：1.0 用三份 JSON 定义公司组织架构与全流程，2.0 用单一 YAML schema 覆盖节点/边/循环/记忆/工具/重试，均无需改代码（yaml_template/design.yaml、CompanyConfig/Default/*.json）。
3. 平滑的代际迁移设计：1.0 保留独立分支且有完整 wiki，1.0 流程又被复刻为 2.0 示例 ChatDev_v1.yaml，老用户有清晰迁移路径。
4. 现代工程栈与安全意识：uv + FastAPI + Vue-flow 的 2.0 技术栈新，MCP 原生支持（mcp_local/mcp_remote），测试聚焦执行沙箱/附件/会话隔离等安全面（tests/ 文件名）。
5. 中英双语完整文档：docs/user_guide/{en,zh} 内容对齐，README 亦双语。

**风险/局限**
1. 代际割裂感：main 分支 README 全部围绕 2.0，访问者若搜"ChatChain/CEO/CTO"等 1.0 概念会找不到对应代码，教程与社区资料多基于 1.0，学习曲线变陡。
2. 贡献者集中：约 20 名贡献者、核心来自同一实验室，bus factor 偏低（gh api contributors 数据）。
3. 测试覆盖薄：31302 行 Python 仅 2306 行测试且 CI 只校验 YAML，graph.py 等 800+ 行核心模块无直接单测。
4. 竞争激烈且定位漂移：2.0 转通用编排平台后直面 LangGraph/Dify/AutoGen 等成熟项目，差异化优势（论文算法）尚待生态验证（客观判断，非实测）。

## 10. 总结评价

ChatDev 是"学术范式中试—产品化转型"的典型样本：1.0 用极简的三层 JSON 配置把"LLM 虚拟软件公司"做成了多智能体领域引用量最高的开源实现之一；2.0 则以 DevAll 之名把同一套思想抽象为 YAML 图编排平台，工程栈大幅现代化，但社区与文档生态仍在向新形态迁移的阵痛期。适合两类人：想研究多智能体协作范式与编排算法（ChatChain/Phase/Role、MacNet、Puppeteer）的研究者，直接读 chatdev1.0 分支的 chatdev/ 包（2254 行即可读完全貌）与论文分支；想用零代码方式搭带记忆/反思/工具调用的多智能体工作流的工程团队，用 main 分支 + Web 控制台。若只是想"一句话生成软件"，1.0 分支的 `python3 run.py --task ...` 仍是最直接的入口。
