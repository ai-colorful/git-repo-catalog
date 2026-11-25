# FoundationAgents/MetaGPT 项目分析

> 星数 70993（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：MetaGPT 是"SOP 驱动的多智能体框架"——把软件公司的标准作业流程（产品经理→架构师→工程师→QA）物化为 LLM 角色协作流水线，`Code = SOP(Team)`，输入一行需求即可产出 PRD、设计文档和可运行代码仓库。

| 事实 | 值 | 出处 |
|---|---|---|
| 名称 | FoundationAgents/MetaGPT（原 geekan/MetaGPT，org 已迁移；README 与 setup.py 内部链接仍指向 geekan） | `gh api repos/FoundationAgents/MetaGPT`、`setup.py` |
| 星数 / forks | 71097 / 9071（gh api 2026-10-31 实测；清单快照 70993） | 同上 |
| 语言 / 许可证 | Python（≥3.9,<3.12）/ MIT | `setup.py`、`LICENSE` |
| 版本 | setup.py version=1.0.0；GitHub 最新 release 仍为 v0.8.2（2025-03-09） | `setup.py`、gh api releases |
| 最近推送 | 2026-01-21（距今约 9 个月） | gh api pushed_at |
| 贡献者 | 约 116 人（contributors 分页推算） | gh api |
| 商业化衍生 | MGX（mgx.dev，2025-02 上线）、atoms.dev | `README.md` News 节 |

## 2. 项目是什么，解决什么问题

MetaGPT 要解决的核心问题是：单个 LLM 直接生成复杂软件时输出混乱、无分工、无质量约束。它借鉴人类软件公司的分工与 SOP，把"输入一句话需求 → 输出用户故事/竞品分析/PRD/系统设计/接口定义/代码"拆成多个角色依次接力，每个角色只做职责内的事，上一角色的结构化产出（如 PRD 文档）是下一角色（如架构师）的输入，从而用流程约束换输出质量（`README.md`、`metagpt/software_company.py`）。

框架层则把这套"角色协作"抽象成通用原语：Role（角色，订阅感兴趣的消息并执行动作）、Action（动作，封装一次 LLM 调用及结构化输出）、Message（带路由元数据的消息）、Environment（消息总线 + 角色容器）、Team（组队、预算控制、回合驱动）。基于这套原语，仓库内不止软件公司一种玩法，还内置了狼人杀（`metagpt/ext/werewolf/`）、斯坦福小镇（`metagpt/ext/stanford_town/`）、辩论（`examples/debate.py`）、研究agent（`examples/research.py`）等场景（`metagpt/environment/` 下有 software/mgx/android/minecraft/stanford_town/werewolf 多个环境实现）。

当前版本（tarball 快照，2026-01 HEAD）实际上是"双代架构并存"：经典固定 SOP 流水线（`WritePRD`→`WriteDesign`→`WriteCode` 动作链）仍在，但软件公司默认阵容已切换到新一代 RoleZero 角色——由 LLM 动态决策、调用工具、维护计划的 agent（`metagpt/software_company.py` 中 hire 的是 TeamLeader/ProductManager/Architect/Engineer2/DataAnalyst，ProductManager 等 roles 继承 `metagpt/roles/di/role_zero.py` 的 RoleZero）。这与官方把框架研究重心转向 MGX 商业产品的路径一致。

学术属性很强：仓库内置多篇论文的官方实现——AFlow（ICLR 2025 oral，自动化 agentic workflow 搜索，`metagpt/ext/aflow/`）、SPO/AOT（2025-02，`metagpt/ext/spo/`）、SELA（`metagpt/ext/sela/`）等（`README.md` News 节、`metagpt/ext/`）。

## 3. 技术栈与依赖

- **语言**：Python 3.9–3.11（`setup.py`: `python_requires=">=3.9, <3.12"`）；核心全用 **pydantic v2** 建模（Role/Action/Message/Team 均为 BaseModel，天然获得序列化/恢复能力，`metagpt/team.py` 的 serialize/deserialize 即依赖此）。
- **LLM 接入**：自研 provider 层，`LLMType` 枚举 26 种 API（openai/azure/anthropic/gemini/ollama/qianfan/dashscope/zhipuai/spark/ark/bedrock/deepseek/moonshot/mistral 等，`metagpt/configs/llm_config.py`），`metagpt/provider/` 下 20+ 适配文件经 `llm_provider_registry.py` 注册。
- **关键第三方库**（`requirements.txt`）：openai 1.x、anthropic、tiktoken、tenacity（重试）、loguru（日志）、typer（CLI）、fire、lancedb/faiss_cpu/qdrant-client（向量记忆）、redis、gitpython（工程落盘归档）、playwright/selenium（浏览器工具）、tree_sitter（代码解析）、httpx、nbclient（Jupyter 执行）。
- **RAG**（optional extra，`setup.py`）：llama-index-core 0.10.x 全家桶 + BM25/FAISS/Elasticsearch/Chroma 向量库。
- **工具链**：ruff + pre-commit（`ruff.toml`、`.pre-commit-config.yaml`）、pytest + coverage（`pytest.ini`）、Docker（`Dockerfile`）、mermaid-cli（文档渲染需 node/pnpm，`setup.py` InstallMermaidCLI）。

## 4. 目录结构与代码组织

单包 monorepo（pip 包 `metagpt` + examples + docs），顶层干净：

| 目录 | 职责 |
|---|---|
| `metagpt/roles/` | 角色实现：经典 SOP 代（product_manager/architect/engineer/project_manager/qa_engineer/researcher/teacher 等 17 文件）+ `di/` 新代（RoleZero 基类及 TeamLeader/DataInterpreter/Engineer2/SWEAgent） |
| `metagpt/actions/` | 动作实现：write_prd/design_api/write_code/write_test/write_code_review 等 SOP 动作 + `action_node.py`（876 行，结构化输出核心）+ `di/` |
| `metagpt/environment/` | 消息总线与多环境：`base_env.py` 通用环境 + software/mgx/werewolf/stanford_town/android/minecraft 子环境 |
| `metagpt/team.py` / `software_company.py` / `startup.py` | Team 编排、软件公司入口（CLI entry `metagpt=metagpt.software_company:app`） |
| `metagpt/schema.py` | Message/Task/Plan/Documents 等核心数据结构（976 行） |
| `metagpt/tools/` | 工具注册表 + `libs/` 32 个内置工具（browser/editor/shell/git/deployer/linter/web_scraping 等） |
| `metagpt/provider/` | LLM 适配层（20+ provider） |
| `metagpt/memory/` | 短期/长期记忆（lancedb/faiss/qdrant 存储） |
| `metagpt/exp_pool/` | 经验池：exp_cache 装饰器 + scorer/perfect_judge（agent 自我改进） |
| `metagpt/strategy/` | Planner（plan-and-act 策略） |
| `metagpt/ext/` | 论文扩展：aflow/sela/spo/cr/android_assistant/stanford_town/werewolf |
| `metagpt/rag/`、`metagpt/learn/`、`metagpt/configs/`、`metagpt/utils/` | RAG、文生图/TTS 等小技能、配置模型、工具函数 |
| `tests/` / `examples/` / `docs/` / `config/` | 309 个测试文件 / 71 个示例 / 多语言文档 / config2.example.yaml |

规模：`metagpt/` 下 509 个 Python 文件。组织方式是单包分层：base 原语 → roles/actions 编排 → environment 总线 → ext 场景/论文，模块边界清晰，RFC 编号（113/116/135）注释在代码中引用设计文档（`metagpt/roles/role.py` 文件头）。

## 5. 核心架构与关键实现

### 5.1 消息总线与订阅机制（Environment + Message 路由）

消息是唯一的协作媒介。`Message` 携带三个路由字段：`cause_by`（产生它的 Action 类名字符串，默认 UserRequirement）、`sent_from`、`send_to`（地址集合，默认广播全体，`metagpt/schema.py:232-241`）。Environment.publish_message 按角色的 addresses 集合定向分发，无人认领则告警并只进历史：

```python
# metagpt/environment/base_env.py:175-193
def publish_message(self, message: Message, peekable: bool = True) -> bool:
    found = False
    for role, addrs in self.member_addrs.items():
        if is_send_to(message, addrs):
            role.put_message(message)
            found = True
    if not found:
        logger.warning(f"Message no recipients: {message.dump()}")
    self.history.add(message)  # For debug
```

订阅在 Role 侧完成：`_watch(actions)` 把感兴趣的 Action 类名存入 `rc.watch`（`metagpt/roles/role.py:284`），`_observe()` 从私有 msg_buffer 取新消息，仅保留 `cause_by in rc.watch` 或角色名命中 `send_to` 的消息进记忆（`role.py:415-436`）。这套"按 Action 类型订阅"的发布-订阅设计是理解所有 SOP 流水线的钥匙：Engineer `_watch([WriteTasks, SummarizeCode, WriteCode, WriteCodeReview, FixBug, ...])`（`metagpt/roles/engineer.py:106`），PRD 一发布即被 Architect 消费。

回合驱动与并发在 Team.run：每轮先检查 `env.is_idle`（所有角色缓冲区无待处理消息则提前结束），然后对非 idle 角色并发 `asyncio.gather(*role.run())`；预算耗尽抛 NoMoneyException 终止（`metagpt/team.py:123-137`、`base_env.py:197-210`）。

### 5.2 Role：Think-Act 循环与三种反应模式

`Role`（`metagpt/roles/role.py`，592 行）是 pydantic 模型，生命周期为 `_observe → _think → _act` 循环。`_think` 决定下一步执行哪个 Action：单动作角色直接执行；多动作角色有三种模式（`RoleReactMode`，role.py:340-380）——`REACT`（LLM 看 STATE_TEMPLATE 动态选下一个动作编号，可自行决定 -1 终止）、`BY_ORDER`（按 set_actions 声明顺序依次执行，经典 SOP 用）、`PLAN_AND_ACT`（Planner 先生成计划再逐任务执行，role.py:459-477）。`_act` 统一调用 `self.rc.todo.run(self.rc.history)` 并把结果包成 AIMessage 回写记忆、经 publish_message 广播（role.py:381-398）。支持 serialize/deserialize 断点恢复（`latest_observed_msg` + recovered 标记，role.py:417-422）。

### 5.3 ActionNode：结构化输出的工程化

SOP 动作能稳定接力，靠的是 `ActionNode`（`metagpt/actions/action_node.py`，876 行）：以树形节点声明字段（key/type/desc/example），运行时动态生成 pydantic 模型作为 LLM 的 JSON/Markdown 输出 schema，输出不合法则携带错误信息自动重试修复。PRD、API 设计、任务拆分等动作都基于它（`write_prd_an.py`、`design_api_an.py`、`project_management_an.py`），`instruct_content` 字段承载结构化结果并可在 Message 中序列化还原（`schema.py:252-274`）。这是 MetaGPT 相对早期"裸 prompt 多智能体"项目最大的工程差异化。

### 5.4 RoleZero：新一代动态 agent

`RoleZero`（`metagpt/roles/di/role_zero.py`，491 行）把角色从"固定动作链"升级为"指令 + 工具 + 动态决策"：`_think` 组装计划状态（Planner）、BM25 推荐的工具 schema、经验池检索结果、角色 instruction 为 system prompt，让 LLM 输出 JSON 命令序列；`_act` 解析并执行命令（Editor/Browser/Terminal 内置为属性，special_tool_commands 含 `Plan.finish_current_task`、`end`、`RoleZero.ask_human`，role_zero.py:69-92）。LLM 调用套了 `@exp_cache` 装饰器，把成功经验写入经验池供后续检索复用（role_zero.py:268-277、`metagpt/exp_pool/decorator.py`）。`use_fixed_sop=True` 可退回经典 SOP 行为（`product_manager.py:37-44`），实现平滑共存。

## 6. 构建与运行

```bash
# 安装（Python 3.9-3.11）
pip install --upgrade metagpt
# 需 node + pnpm（mermaid 渲染）
metagpt --init-config          # 生成 ~/.metagpt/config2.yaml，填 api_key
metagpt "Create a 2048 game"   # CLI 运行，产物在 ./workspace

# 库用法（README.md Usage 节）
python -c "from metagpt.software_company import generate_repo; generate_repo('Create a 2048 game')"

# 源码开发
git clone https://github.com/FoundationAgents/MetaGPT && cd MetaGPT
pip install -e .        # 或 pip install -e .[test] 带测试依赖
pytest                  # 单测；CI 设 ALLOW_OPENAI_API_CALL=0 防真实调用（.github/workflows/unittest.yaml）

# Docker
docker build -t metagpt . && docker run -it --rm metagpt  # Dockerfile
```

配置体系：`~/.metagpt/config2.yaml`（llm/mermaid/redis 等），对应模型 `metagpt/config2.py` 与示例 `config/config2.example.yaml`。

## 7. 测试与工程质量

- **规模**：`tests/` 309 个 Python 文件，按 metagpt 包结构镜像组织（actions/roles/provider/memory/serialize_deserialize 等），另含 `tests/mock/` 与 rsp_cache 缓存机制（CI 上传 `tests/data/rsp_cache_new.json`，`unittest.yaml`）。
- **风格**：pytest + pytest-asyncio，`pytest.ini` 开启 `--doctest-modules` 和覆盖率，但 ignore 列表长达数十条——凡是依赖真实 LLM 响应的测试被排除（test_write_prd.py、test_action_node.py 等），说明大量测试是"结构/序列化/mock 可测，端到端不可测"的状态。CI 用 `ALLOW_OPENAI_API_CALL=0` 强制兜底（`.github/workflows/unittest.yaml`）。
- **CI**：5 个 workflow——unittest（仅 py3.9 矩阵，3.10/3.11 被注释）、fulltest、pre-commit、build-package、stale（`.github/workflows/`）。lint 用 ruff + pylint 双轨。
- **观察**：模块普遍带文件头注释记录 RFC 演进与修改人（如 role.py 头部 5 条 Modified By 记录），协作规范程度高；但 requirements.txt 中有大量注释掉的废弃依赖（Django/pygame/pymilvus 等），反映多场景探索后收敛留下的痕迹。

## 8. 社区与维护状态

- **活跃度**：创建于 2023-06-30，2023-2024 年是多智能体领域最热项目之一；gh api 显示最后一次 push 为 **2026-01-21**，tarball 内所有文件 mtime 均为 2026-01-21（即 HEAD 快照日期）。此后约 9 个月（截至 2026-10-31）无提交，维护已明显放缓——主要精力转向商业产品 MGX/atoms.dev（`README.md` News 节最后一条为 2025-03-10）。
- **Release 节奏**：GitHub release 停在 v0.8.2（2025-03-09），而 `setup.py` 版本已是 1.0.0，说明 1.0 只走了 PyPI/repo、未走 GitHub release 流程；v0.8 之前基本维持月级节奏（v0.8.0 2024-03、v0.8.1 2024-04）。
- **社区**：约 116 名贡献者（gh api 分页推算），134 个 open issue；有 Discord 社区徽章（`README.md`）。核心作者 Alexander Wu（deepwisdom.ai）。组织已由个人账号 geekan 迁至 FoundationAgents，但 README/setup.py 大量内链未更新，官方文档站为 docs.deepwisdom.ai。

## 9. 亮点与风险

**亮点**
1. 按 Action 类型的发布-订阅消息总线设计干净（`environment/base_env.py` + `roles/role.py`），RFC 驱动的演进痕迹可考，是学习多智能体编排的一流范本。
2. ActionNode 结构化输出层把"LLM 输出可靠接力"工程化，远早于后来的 structured output API（`actions/action_node.py`）。
3. 双代架构平滑共存：经典 SOP 流水线（教学友好）与 RoleZero 动态 agent（能力更强）用 `use_fixed_sop` 一个开关切换（`roles/di/role_zero.py:98`）。
4. 场景与研究复现丰富：软件公司之外有狼人杀、小镇、SWE benchmark，内置 AFlow/SPO/SELA 等论文官方实现（`metagpt/ext/`），学术可信度高。
5. 工程配套完整：26 种 LLM provider、经验池自我改进、断点恢复、预算控制、Docker、RAG extras。

**风险/局限**
1. 维护近乎停滞：最后推送 2026-01，最新 release 近一年半前，open issue 无人清理；选型需预期无修复响应。
2. 依赖面极重：requirements 直装约 60+ 包且强 pin 版本（如 numpy~=1.26.4、semantic-kernel 0.4.3.dev0 dev 版），Python <3.12 限制，与新版生态共存困难。
3. 效果强依赖 GPT-4 级模型与多轮调用，成本高（官方默认 investment=3.0 美元预算机制即是证据，`software_company.py`）；复杂项目产出质量仍不稳定。
4. 1.0.0（MGX 架构）与文档/教程（docs.deepwisdom.ai 大多对应 0.8）存在代差，新用户易被旧文档误导；org 迁移后内链未改也可能产生 404。

## 10. 总结评价

MetaGPT 是"多智能体框架"赛道的定义级项目之一：它把 LLM 协作从"聊天轮次"改造成"带路由元数据的消息总线 + 可订阅角色 + 结构化动作输出"，其 Role/Action/Environment/Message 抽象与 `Code = SOP(Team)` 的理念影响了后续大量 agent 框架。代码组织清晰、注释与设计文档意识出色，7 万+ 星实至名归；但截至分析日，项目作为开源框架已进入低维护期，重心明显移向商业产品 MGX。适合三类人：想理解多智能体系统内部机制而读源码的工程师（重点读 `roles/role.py`、`environment/base_env.py`、`actions/action_node.py` 三个文件即可掌握骨架）、做 agent 编排研究需要论文复现基座的研究者（`metagpt/ext/aflow` 等）、以及需要离线/私有化部署固定 SOP 流水线且能自行维护依赖的团队。作为生产依赖引入则需谨慎评估停滞风险。
