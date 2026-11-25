# FoundationAgents/OpenManus 项目分析

> 星数 58763（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：MetaGPT 团队 3 小时内做出的开源通用 Agent 框架——对 Manus 产品（邀请码制）的开放复刻，以 ReAct + 工具调用 + MCP 为核心，"无堡垒，纯粹的开阔地"。

| 项 | 事实 |
| --- | --- |
| 名称 | FoundationAgents/OpenManus（前身 mannaandpoem/OpenManus，已迁至 org） |
| 星数 | 58,858（gh api 实时，快照 58,763）；fork 10,224；open issues 443 |
| 语言/环境 | Python，要求 >= 3.12（`setup.py`） |
| 许可证 | MIT |
| 创建时间 | 2025-03-06 |
| 最近活跃 | main 分支最近提交 2026-08-16（PR #1398 browser-use-cli-3），仓库 push 至 2026-11-02 |
| 版本 | v0.1.0 / v0.2.0 / v0.3.0，三个 tag 均发布于 2025-04-10（gh api tags/releases） |
| 贡献者 | 54 人（gh api contributors 分页统计） |
| 规模 | 90 个 .py 文件，约 12,302 行 Python（tarball 统计） |

## 2. 项目是什么，解决什么问题

2025 年 3 月 Manus 产品一码难求，MetaGPT 社区的 @Xinbing Liang、@Jinyu Xiang 等五位核心成员在 3 小时内搭出 OpenManus 原型并开源（`README.md`），宣言"No fortress, purely open ground"（仓库 description）。它回答的问题是：一个"给个想法就能自己拆解、调工具、干到底"的通用 agent，其最小可用内核长什么样。

OpenManus 不是单一产品而是一个框架：`main.py` 提供单 agent 交互（输入想法 → Manus 自主循环执行），`run_flow.py` 提供多 agent 的计划-执行编排（README 自述为 "unstable multi-agent version"），`run_mcp.py` / `run_mcp_server.py` 分别消费和对外提供 MCP 工具服务，`sandbox_main.py` 面向沙箱隔离场景（顶层目录）。能力覆盖编程、信息检索、文件处理、网页浏览乃至图表可视化（`app/prompt/manus.py` 的 SYSTEM_PROMPT）。

设计上它是典型的 ReAct 架构：LLM 在每一步"思考"（选择工具与参数）→"行动"（执行工具）→ 把观察结果写回记忆，直到 LLM 调用 `terminate` 特殊工具或步数耗尽。这与 Manus 官方公开的架构叙事一致，因而成为学界/工程界研究 agent 内核最常用的参考实现之一（README 引用其 Zenodo DOI，引用条目自述为 "An open-source framework for building general AI agents"）。

项目的另一条主线是生态化：内置 MCP 双向支持（既作 MCP 客户端聚合远程工具，也能把自己暴露为 MCP server，`app/mcp/server.py`、`run_mcp_server.py`）、浏览器自动化默认接入 browser-use CLI 3.0（`app/agent/manus.py` 的 `_BROWSER_USE_COMMAND`）、另有 A2A 协议服务示例（`protocol/a2a/`）与姊妹项目 OpenManus-RL（RL 训练 agent，README 链接）。

## 3. 技术栈与依赖

- 语言与运行时：Python 3.12+，asyncio 全异步；90 个 py 文件约 1.2 万行，属"小内核"项目。
- 数据与校验：pydantic v2（~2.10）贯穿全部实体——Agent、Message、Tool、配置均为 BaseModel（`app/schema.py`、`app/agent/base.py`）。
- LLM 接入：openai SDK（~1.66）+ tenacity 重试 + tiktoken 计数（`app/llm.py`）；支持 OpenAI 兼容端点，配置示例覆盖 Anthropic/Azure/Google/Ollama/PPIO 等（`config/config.example-model-*.toml`）。
- Agent 协议：mcp SDK（~1.5）实现 SSE/stdio 双传输客户端（`app/tool/mcp.py`）；A2A SDK（`protocol/a2a/app/main.py` 引用 `a2a.server.apps`）。
- 执行与隔离：docker SDK 实现 DockerSandbox（`app/sandbox/core/sandbox.py`）；Daytona 云沙箱（`app/daytona/sandbox.py`）。
- 工具生态：playwright + browsergym（浏览器任务环境）、crawl4ai（爬取）、googlesearch-python / baidusearch / duckduckgo_search（多引擎搜索后端，`app/tool/search/`）、pillow（图像）。
- 服务与工程：fastapi/uvicorn（A2A 与 MCP server）、loguru/structlog 双日志、pytest + pytest-asyncio（`requirements.txt`）。
- 前端点缀：图表可视化工具带一个 TypeScript + ECharts 的小前端（`app/tool/chart_visualization/src/chartVisualize.ts`），是该纯 Python 仓库中唯一的非 Python 实现。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `main.py` / `run_flow.py` / `run_mcp.py` / `run_mcp_server.py` / `sandbox_main.py` | 五个顶层入口：单 agent / 多 agent 编排 / MCP 消费 / MCP 服务 / 沙箱 |
| `app/agent/` | Agent 类层次：base → react → toolcall → manus 及各特化 agent |
| `app/tool/` | 15 个工具模块 + 基类/集合 + MCP 工具桥 + search 多引擎后端 + 沙箱工具 |
| `app/flow/` | 编排层：BaseFlow、PlanningFlow（442 行）、FlowFactory |
| `app/llm.py` | 766 行 LLM 抽象：消息格式化、token 计数与限额、重试 |
| `app/schema.py` / `app/config.py` / `app/exceptions.py` | 消息/状态模型、TOML 配置单例、异常体系 |
| `app/prompt/` | 各 agent 的系统提示词与下一步提示词（纯字符串模块） |
| `app/sandbox/` | Docker 沙箱：core/sandbox.py、manager.py、terminal.py、client.py 全局单例 |
| `app/daytona/` | Daytona 云沙箱接入 |
| `app/mcp/` | 把内置工具暴露为 MCP server（`run_mcp_server.py` 使用） |
| `protocol/a2a/` | A2A（agent-to-agent）协议服务示例 |
| `config/` | config.toml 模板 + 6 个厂商模型示例 + mcp.example.json |
| `tests/` / `examples/` / `assets/` / `workspace/` | 测试、用例展示、图片、agent 工作区 |

组织方式评述：单包（single package）单体，无 monorepo/子包拆分；按"agent/tool/flow/prompt/sandbox"四象限垂直切分，一个概念一个文件，层次短平快——是典型的"3 小时原型长成的框架"，靠目录约定而非抽象层维持秩序。

## 5. 核心架构与关键实现

### 5.1 Agent 类层次：三层继承 + 特化

```
BaseAgent (app/agent/base.py)          状态机 + 记忆 + max_steps 主循环 + 卡死检测
└── ReActAgent (app/agent/react.py)    抽象 think()/act()，step = think → act
    └── ToolCallAgent (app/agent/toolcall.py)   LLM 工具调用闭环
        ├── Manus (app/agent/manus.py)          旗舰通用 agent，挂载 MCP
        ├── MCPAgent → BrowserAgent (app/agent/mcp.py, browser.py)
        ├── SWEAgent (app/agent/swe.py)         bash + str_replace_editor
        ├── DataAnalysis (app/agent/data_analysis.py)
        └── SandboxManus (app/agent/sandbox_agent.py)
```

`BaseAgent`（pydantic BaseModel + ABC，`app/agent/base.py`）的主循环把执行做成有界步数的状态机，并用"最近 assistant 消息重复 ≥2 次"判断卡死、注入换策略提示（`is_stuck`/`handle_stuck_state`，L163-186）：

```python
while (self.current_step < self.max_steps and self.state != AgentState.FINISHED):
    self.current_step += 1
    step_result = await self.step()
    if self.is_stuck():
        self.handle_stuck_state()
```

`ToolCallAgent.think()`（`app/agent/toolcall.py` L39-129）以 `tool_choice` 三模式（none/auto/required，`app/schema.py` ToolChoice）调 `llm.ask_tool`，捕获 `TokenLimitExceeded` 时直接置 FINISHED 而非重试；`act()` 逐个执行 tool_calls，`max_observe` 截断观察（Manus 设 10000，`app/agent/manus.py` L50），工具返回的 `base64_image` 会追加为 user 图像消息回注记忆（L162-168）。`Terminate` 等注册在 `special_tool_names` 中的工具执行成功即把状态切到 FINISHED（`_handle_special_tool`）。

### 5.2 工具体系：pydantic 工具 + 名称去重集合 + MCP 桥

每个工具 = 一个 pydantic 模型：`BaseTool`（`app/tool/base.py` L78-137）声明 name/description/parameters，`to_param()` 直接产出 OpenAI function-calling schema，`execute()` 为唯一抽象方法；结果统一为 `ToolResult(output, error, base64_image, system)`，可 `__add__` 合并。`ToolCollection`（`app/tool/tool_collection.py`）维护 name→tool 映射，`add_tool` 对重名跳过并告警——这正是动态挂载 MCP 工具时的防撞机制。

MCP 双向桥是工程上最值得读的部分：`MCPClients` 继承 ToolCollection（`app/tool/mcp.py` L49），支持 SSE 与 stdio 两种传输；远程工具被包成本地 `MCPClientTool`（同文件 L14），工具名经 `_sanitize_tool_name` 清洗并按 server_id 加前缀避免冲突。`Manus.initialize_mcp_servers`（`app/agent/manus.py` L83-127）默认通过 `uvx browser-use --cli-mcp` 把 browser-use CLI 3.0 挂为内置 MCP server（环境变量 `OPENMANUS_DISABLE_BROWSER_USE=1` 可关闭），server instructions 会作为 system 消息注入记忆（L159-171）。

### 5.3 Flow 编排：PlanningFlow 的计划-执行循环

`BaseFlow`（`app/flow/base.py`）统一持有 `agents` 字典与 `primary_agent`；`FlowFactory`（`app/flow/flow_factory.py`）目前只注册 `PLANNING` 一种。`PlanningFlow`（`app/flow/planning.py`，442 行）是唯一的实现：先用 LLM + `PlanningTool` 生成初始计划（多 agent 时在系统提示中注入各 agent 描述，要求步骤以 `[agent_name]` 指派执行者，L134-180），然后进入主循环：

```python
while True:
    self.current_step_index, step_info = await self._get_current_step_info()
    if self.current_step_index is None:          # 计划完成
        result += await self._finalize_plan(); break
    executor = self.get_executor(step_info.get("type"))
    step_result = await self._execute_step(executor, step_info)
    if executor.state == AgentState.FINISHED: break
```

计划步骤有四态（not_started/in_progress/completed/blocked，`PlanStepStatus`），执行失败可置 BLOCKED 而非整体崩溃；`run_flow.py` 整体套 3600 秒 asyncio 超时。

### 5.4 LLM 层：token 预算与重试

`app/llm.py`（766 行）是体量最大的单文件：`TokenCounter` 用 tiktoken 精确计数文本/图像（按分辨率算 high-detail 图像 token，L95 起）/tool_call；`LLM.check_token_limit` 在每次请求前估算，超限抛 `TokenLimitExceeded` 且 tenacity 重试明确排除该异常（`@retry(...)` L354-360）——保证"预算耗尽"不被无限重试烧钱。`ask/ask_with_images/ask_tool` 三个入口分别对应纯文本、多模态、工具调用三种调用面。

## 6. 构建与运行

来自 `README.md` 与 `config/config.example.toml`：

```bash
# 安装（推荐 uv；conda 亦可，需 Python 3.12）
uv venv --python 3.12 && source .venv/bin/activate
uv pip install -r requirements.txt

# 配置：复制模板并填 API key（LLM 端点为 OpenAI 兼容格式）
cp config/config.example.toml config/config.toml
#   [llm] model/base_url/api_key/max_tokens/temperature；[llm.vision] 可选
#   [runflow] use_data_analysis_agent = true 启用数据分析 agent

# 运行
python main.py        # 单 Manus agent（可 --prompt "..." 直接传参，main.py）
python run_flow.py    # PlanningFlow 多 agent 编排（1 小时超时）
python run_mcp.py     # MCP 工具版 agent
python run_mcp_server.py  # 把内置工具暴露为 MCP server

# 浏览器自动化（默认 MCP server）
uvx browser-use --cli-mcp    # OPENMANUS_DISABLE_BROWSER_USE=1 关闭
playwright install           # 仅 BrowserGym 需要

# 质量工具
pre-commit run --all-files
```

容器化：`Dockerfile` 基于 python:3.12-slim，用 uv 安装依赖后 `CMD ["bash"]`（未定义默认 agent 入口）。

## 7. 测试与工程质量

- 测试规模：`tests/` 仅 5 个文件（4 个 sandbox：test_client/test_sandbox/test_sandbox_manager/test_docker_terminal；1 个 tools：test_browser_use_mcp），pytest + pytest-asyncio。相对 1.2 万行核心代码，agent/flow/llm 层基本无单测覆盖（tarball 内未见对应测试文件）。
- CI（`.github/workflows/`，6 条流水线）：pre-commit（black/isort/autoflake）、build-package、environment-corrupt-check（环境损坏检查，较有特色）、pr-autodiff、stale（自动关闭陈旧 issue）、top-issues；另有 dependabot（`.github/dependabot.yml`）。
- 代码质量观察：pydantic 模型驱动、异常体系清晰（`app/exceptions.py`：TokenLimitExceeded/ToolError 等）；日志用 emoji + loguru，可读性好。瑕疵：`app/tool/base.py` 保留约 70 行注释掉的旧实现（L10-35、L97-145）；`setup.py` 的 console_scripts 指向 `main:main`，而 `main.py` 中 main 是 async 函数且无同步包装，安装后 `openmanus` 命令实际不可用（未确认是否影响发布版）；`protocol/a2a/app/main.py` L13 引入 `app.tool.browser_use_tool`，但该文件在仓库中不存在（HEAD 断链，A2A 示例不可运行）。

## 8. 社区与维护状态

- 活跃度：2025-03-06 创建后爆发式涨星（README 星史图）；gh api 显示 main 最近提交 2026-08-16（browser-use CLI 3.0 集成，PR #1398），仓库 2026-11-02 仍有 push，issue 区 443 个 open，社区热度仍高但主分支近月节奏放缓。
- 版本节奏：v0.1.0→v0.3.0 集中在 2025-04-10，此后未再打 tag——以 main 直发为主（gh api tags/releases）。
- 贡献者结构：54 名贡献者；核心来自 MetaGPT 团队（README 作者段），生态绑定明显：Discord + 飞书社群（`assets/community_group.png`）、HuggingFace Demo 空间（stepfun 支持）、赞助商 PPIO、Zenodo DOI。
- 姊妹项目：OpenManus-RL（UIUC 合作，GRPO 等 RL 调优，README 链接）。

## 9. 亮点与风险

**亮点**

1. 教科书级的 agent 内核分层：状态机 → ReAct → 工具调用三层各 ~200 行，是读源码学 agent 的最佳入口之一（`app/agent/base.py`/`react.py`/`toolcall.py`）。
2. MCP 双向完整支持：既能聚合任意远程 MCP 工具（含默认内置 browser-use），又能反把自己暴露为 MCP server，一仓库打通两种角色（`app/tool/mcp.py`、`app/mcp/server.py`）。
3. 工程细节扎实：tiktoken 精确 token 预算 + 限额异常不重试（`app/llm.py`）、卡死检测注入换策略提示（`app/agent/base.py`）、Docker/Daytona 双沙箱隔离（`app/sandbox/`）。
4. 生态位与社区效应：3 小时原型 + MetaGPT 背书 + 多语言 README（en/zh/ko/ja），赶在 Manus 热点窗口期占据开源生态位，10k fork 形成飞轮。

**风险/局限**

1. 测试严重偏科：核心 agent/flow/llm 无单测，回归靠 pre-commit 与人工（`tests/` 仅 5 文件）。
2. 边缘模块烂尾：A2A 示例 import 断链（`protocol/a2a/app/main.py` → 不存在的 `app/tool/browser_use_tool.py`）、setup.py 入口指向 async 函数无法直用、`app/tool/base.py` 存留大段注释代码。
3. 简化设计的固有局限：卡死检测只做 assistant 文本精确重复匹配；`FlowFactory` 只有 PLANNING 一种 flow；Memory 无摘要/压缩，长任务受 token 预算硬约束。
4. 版本管理松散：一年半无新 tag，main 直发，用户锁定具体行为只能靠 commit。

## 10. 总结评价

OpenManus 用约 1.2 万行 Python 证明了"通用 agent 框架"的最小完备形态：一条 ReAct 主循环、一套 pydantic 工具协议、一个计划编排器、一层 MCP 桥，其余全是生态与社区运营。它不是工程最严谨的 agent 框架（测试薄、边缘模块断链、无正式版本节奏），但可能是被阅读、fork 和二次开发最多的 agent 参考实现——58k 星里很大一部分来自"看懂 Manus 到底怎么运作"的需求。适合三类人：想理解 agent 内核原理的工程师（精读 `app/agent/` 三层与 `app/tool/mcp.py`）、想快速搭自定义 agent 的原型者（复制 Manus 换工具集即可）、以及做 agent 相关研究需要可控 baseline 的学者。想读源码的人应抱着"读内核、跳过边缘"的预期。
