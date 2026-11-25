# bytedance/deer-flow 项目分析

> 星数 84330（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：字节跳动开源的"超级 Agent 运行时（super agent harness）"——单 Lead Agent 编排子 Agent、持久记忆、沙箱执行与可扩展技能，覆盖深度研究到通用任务自动化。

| 事实 | 值 | 出处 |
| --- | --- | --- |
| 名称 | DeerFlow（Deep Exploration and Efficient Research Flow） | `README.md` |
| 星数 / Fork | 84330 / 11885 | 任务快照 / `gh api repos/bytedance/deer-flow` |
| 语言 | Python 3.12+（后端）+ TypeScript/Next.js（前端） | `backend/pyproject.toml`、`frontend/package.json` |
| 许可证 | MIT | `LICENSE` |
| 当前版本 | 2.3.0-dev（HEAD）；v2.2.0 发布于 2026-10-29 | `backend/pyproject.toml`、`CHANGELOG.md` |
| 最近活跃 | push 于 2026-10-30（分析前一天），仓库创建于 2025-05-07 | `gh api repos/bytedance/deer-flow` |
| 重要说明 | HEAD 是 2.0 完全重写版，与 v1（LangGraph planner/researcher 深度研究框架）不共享代码；v1 维护于 `main-1.x` 分支 | `README.md`、`docs/ARCHITECTURE.md` |

## 2. 项目是什么，解决什么问题

DeerFlow 起家是"Deep Research 框架"（v1：planner→researcher→reporter 的 LangGraph 编排），但社区把它用成了数据管道、PPT 生成、仪表盘搭建等通用工具，作者据此判断它本质上不是研究工具而是一个 **harness（Agent 运行时基础设施）**，于是 2.0 从零重写（`README.md` "From Deep Research to Super Agent Harness"、`docs/ARCHITECTURE.md`）。注意：本分析基于 HEAD（2.x），任务描述中的 planner/researcher/coder 状态机属于 v1，已移至 `main-1.x` 分支，HEAD 快照中不存在该编排。

2.0 解决的问题是：给一个 Lead Agent 配齐"真正干活"所需的一切——文件系统（每线程隔离的 workspace/uploads/outputs）、沙箱代码执行、持久记忆、可渐进加载的技能（Skills）、MCP 工具、子 Agent 派发与并发/预算控制。它不再是你自己拼装的框架，而是开箱即用、可整体拆改的运行时（`docs/ARCHITECTURE.md` §1）。

工程形态是一个可自托管的全栈产品：Nginx 统一入口（2026 端口）→ FastAPI Gateway（8000/8001，内嵌 LangGraph 兼容运行时）+ Next.js 聊天前端（3000）+ 可选沙箱 Provisioner（8002）；飞书/Slack/Telegram/Discord/钉钉/企微等 IM 通过 Gateway 桥接到同一个 Agent（`docs/ARCHITECTURE.md` §2，`docker/` 目录）。Gateway 还提供 LangGraph 兼容路由 `/api/langgraph/*`，标准 LangGraph SDK 客户端无需单独的 LangGraph Server 即可直连（`docs/ARCHITECTURE.md` §2）。

用户视角的核心玩法：网页聊天里创建 thread，Agent 通过 `task` 工具把子任务派给子 Agent（内置 `general-purpose` 与 `bash` 两种，支持自定义），在沙箱里跑代码、产出 artifacts，用技能生成 PPT/播客/网页/图表；所有执行都有确定性"回执（receipts）"注入上下文，最终报告可引用已执行动作（`config.example.yaml` verification 段）。

知识增强是另一条主线：`knowledge_search` 文档检索、WeKnora 插件（`examples/deerflow-extension-weknora/`）与 LightRAG 实体关系检索三类只读知识源可在 `tools[]` 中独立开关，共享范围准入（scope admission）、预算与引用渲染契约；检索引用在前端以"提供方 + 文档标题 + 摘录"卡片呈现，子 Agent 完成的检索也能回溯证据（`backend/docs/KNOWLEDGE.md`、`README.md` "Read-only knowledge sources" 节）。长期记忆（DeerMem/Honcho 后端）、会话目标 `/goal`、上下文手动压缩、聊天归档等均是围绕"上下文工程"的产品化功能（`agents/memory/`、`README.md` Core Features）。

## 3. 技术栈与依赖

- 后端：Python 3.12+，LangGraph + LangChain（agent 编排）、FastAPI + Starlette（Gateway，`starlette>=1.3.1,<2` 且因安全边界显式 pin，`backend/pyproject.toml`）、uvicorn、SSE（sse-starlette）、WebSocket。
- 核心自研双包：`deerflow-harness`（`backend/packages/harness/`，agent 框架本体）+ `deerflow-extension-api`（`backend/packages/extension-api/`，插件公开契约），应用层 `backend/app/` 依赖 harness，反向禁止（见第 5 节）。
- 沙箱：`SandboxProvider` 抽象，内置 LocalSandboxProvider（本地开发）、AioSandboxProvider（Docker/Apple Container）、E2B（`e2b-code-interpreter`）、BoxLite/Tenki 等（`backend/packages/harness/deerflow/sandbox/`、`config.example.yaml` sandbox 段）。
- 模型适配：OpenAI 兼容之外自带 claude/codex/deepseek/minimax/stepfun/mimo/vllm/mindie 等补丁式 Provider（`backend/packages/harness/deerflow/models/`），README 推荐豆包 Seed-2.0-Code、DeepSeek v3.2、Kimi 2.5。
- IM/渠道 SDK：lark-oapi、slack-sdk、python-telegram-bot、dingtalk-stream、wecom-aibot-python-sdk、discord.py（可选）（`backend/pyproject.toml`）。
- 可观测：LangSmith/Langfuse/Monocle 三套 tracing 开关（`backend/pyproject.toml` dev 组、`README.md` Advanced 节）。
- 前端：Next.js 16 / React 19 / TypeScript / Tailwind v4 / Radix UI，`@langchain/langgraph-sdk` 驱动流式编排，TanStack Query（`docs/ARCHITECTURE.md` §4、`frontend/package.json`）。
- 测试：pytest（+hypothesis/pytest-asyncio）、rstest、Playwright；构建工具 uv（后端）、pnpm（前端）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `backend/packages/harness/deerflow/` | Agent 框架本体：agents、subagents、runtime、tools、skills、sandbox、models、mcp、memory、authz、knowledge、tracing 等 30+ 子模块 |
| `backend/packages/extension-api/` | `deerflow_extension_api`：主机无关的插件公开契约（稳定 API 面） |
| `backend/app/` | 未发布的应用层：FastAPI Gateway（`gateway/`，40+ 路由模块）+ IM 渠道接入（`channels/`）、调度器、子 Agent 批处理 |
| `frontend/` | Next.js 工作台：threads 聊天、agents 画廊、Capability Center（插件/技能管理）、showcase 只读演示 |
| `skills/public/` | 23 个内置技能包（SKILL.md）：deep-research、ppt-generation、podcast-generation、data-analysis、skill-creator 等 |
| `examples/` | 14 个 `deerflow-extension-*` 插件示例（agent-teams、weknora、lightrag、workflows 等） |
| `docs/`、`backend/docs/` | 分层架构文档：顶层 `docs/ARCHITECTURE.md`，backend 下 API/AUTH/GUARDRAILS/KNOWLEDGE/MEMORY/LOOP_DETECTION 等专题 |
| `docker/`、`deploy/` | docker-compose 三套形态（dev/cli-auth/dood）+ nginx 配置 + Helm chart |
| `tests/`（顶层）、`backend/tests/`、`frontend/tests/` | e2e 与各端测试 |
| `contracts/`、`plans/`、`scripts/` | 契约文件、开发计划、构建/多实例脚本 |

组织方式评述：典型的 **monorepo 多包分层**。关键设计是 Harness/App 严格单向依赖——App 可 import `deerflow`，harness 永不 import `app`，由 CI 测试 `backend/tests/test_harness_boundary.py` 强制（`docs/ARCHITECTURE.md` §3）。harness 可独立发布为 `deerflow-harness` 包，插件只依赖 extension-api 契约，形成"框架 / 应用 / 插件"三层。

## 5. 核心架构与关键实现

### 5.1 Lead Agent 组装与中间件链（编排核心）

Lead Agent 不是手写状态机，而是 LangChain `create_agent` + 约 59 个 middleware 文件（`backend/packages/harness/deerflow/agents/middlewares/`）拼装出的运行时。入口 `make_lead_agent`（`backend/packages/harness/deerflow/agents/lead_agent/agent.py`，1446 行）在每次装配时：解析用户身份/模型/交互策略 → 装配工具（内置 + 配置 + MCP）→ 叠加中间件链 → 编译图并产出可观测的 assembly descriptor。官方文档列出的链路顺序：ThreadData（线程数据目录）→ Uploads → Sandbox → Summarization → Title → Todo → ViewImage → Clarification（`docs/ARCHITECTURE.md` "Agent runtime path"）。中间件覆盖安全（input_sanitization、pii_redaction、loop_detection、safety_finish_reason）、上下文工程（summarization、durable_context、system_message_coalescing）、治理（token_budget、subagent_limit、receipt_verification）三类，是整个框架最重的实现层。

关键片段（`agents/factory.py`，纯参数工厂作为 SDK 级入口）：

```python
def create_deerflow_agent(...):
    """Pure-argument factory ... sits between the raw
    ``langchain.agents.create_agent`` primitive and the config-driven
    ``make_lead_agent`` application factory."""
```

### 5.2 子 Agent 执行引擎与容量治理

`backend/packages/harness/deerflow/subagents/`（约 20 个模块）实现派发治理：`task` 工具（`tools/builtins/task_tool.py`）把子任务交给 `SubagentExecutor`（`executor.py`），在隔离事件循环上运行子 Agent 图，带容量控制（`capacity.py`：默认 `max_running=3`、`max_queued=64`、admission=queue）、每 run 总派发上限（默认 6，1-50）、子 Agent token 预算（硬停时强制 `finish_reason=stop` 并标记 `subagent_stop_reason=token_capped`）、turn 预算换算 LangGraph step 上限（`turn_budget.py`）、验收标准检查（`acceptance_checks.py`）与结果状态契约（`status_contract.py`）。另有持久化批处理 `subagent_batches`（默认关闭，lease/重试/结果上限齐备）（`config.example.yaml` subagent_runtime/subagents/subagent_batches 段）。内置子 Agent 仅两种：`general-purpose`（150 turns）与 `bash`（60 turns），自定义 Agent 走 config 声明 prompt/tools/skills 白名单（`subagents/builtins/`、`config.example.yaml` custom_agents 注释段）。

### 5.3 技能（Skills）系统

`backend/packages/harness/deerflow/skills/` 约 22 个模块实现 Claude 风格 Agent Skills：SKILL.md + frontmatter 解析校验（`parser.py`、`validation.py`）、渐进式发现/激活（`skill_activation_middleware.py`，任务需要时才加载以省上下文）、`.skill` 包安装/导入/导出（`installer.py`、`export.py`）、安全静态扫描（`security_scanner.py`、`skillscan/`）、命名冲突诊断与 slash 命令。内置 23 个公开技能（`skills/public/`），含 skill-creator 元技能；技能包目录即包边界，嵌套 SKILL.md 不重复注册（`README.md` Skills & Tools 节）。

### 5.4 运行时与状态

`backend/packages/harness/deerflow/runtime/`：`ThreadState` 扩展 LangGraph AgentState（sandbox/artifacts/thread_data/title/todos/viewed_images），每线程数据隔离在 `.deer-flow/threads/{thread_id}/`；checkpoint 支持 sqlite/postgres 后端与 channel 模式冻结（`checkpoint_mode.py`、`checkpointer/`）；SSE 流桥 `stream_bridge/` 对子 Agent 发命名空间事件 `values|<ns>`，兼容 LangGraph Platform 风格（`docs/ARCHITECTURE.md` §3）。`backend/langgraph.json` 把 `deerflow.agents:make_lead_agent` 注册为 `lead_agent` 图，可直连 LangGraph Studio。

工具层三分合一：`get_available_tools()`（`tools/tools.py`）合并内置工具（`present_file`、`ask_clarification`、`review_skill_package` 等，`tools/builtins/` 13 个）、config.yaml 声明的配置工具（bash/read_file/write_file/str_replace/ls、web 搜索抓取、knowledge_search）与 MCP 工具（带来源标签与缓存）；`task`/`batch_task` 作为子 Agent 派发工具按运行时容量条件注入，另支持 ACP agent 调用与插件工具（`tools/tools.py` BUILTIN_TOOLS/SUBAGENT_TOOLS 及装配逻辑）。Gateway 侧 40+ REST 路由（threads、runs、agents、skills、memory、mcp、channels、github_webhooks 等）对齐产品功能面（`backend/app/gateway/routers/`）。

源码阅读路线（关键文件索引）：

| 想理解什么 | 读哪里 |
| --- | --- |
| Lead Agent 怎么组装 | `backend/packages/harness/deerflow/agents/lead_agent/agent.py`（`_assemble_lead_agent`）、`agents/factory.py` |
| 中间件链 | `backend/packages/harness/deerflow/agents/middlewares/`（59 个文件） |
| 子 Agent 派发与预算 | `backend/packages/harness/deerflow/tools/builtins/task_tool.py`、`subagents/executor.py`、`subagents/capacity.py` |
| 工具装配 | `backend/packages/harness/deerflow/tools/tools.py`、`tools/builtins/` |
| 技能生命周期 | `backend/packages/harness/deerflow/skills/`（parser/validation/installer/security_scanner） |
| 沙箱抽象 | `backend/packages/harness/deerflow/sandbox/sandbox_provider.py`、`sandbox/security.py` |
| 运行入口 | `backend/app/gateway/app.py`、`backend/langgraph.json`、根 `Makefile` |

## 6. 构建与运行

来自 `Makefile`、`backend/pyproject.toml`、`README.md`、`Install.md`：

```bash
# 一键安装（setup wizard，后端 uv + 前端 pnpm）
make setup            # backend/scripts + scripts/setup_wizard.py
make dev              # 开发模式：check.py 预检 + scripts/serve.sh --dev
make start            # 生产模式（SKIP_FRONTEND_BUILD=1 复用前端构建）
make dev-daemon       # 后台守护模式
make docker-init / docker-start / docker-logs   # Docker 形态（nginx 2026 端口统一入口）
make config-upgrade   # config.yaml 模式升级（config_version: 64）

# 后端测试与静态检查（backend/Makefile、pyproject dev 组）
make check            # 顶层聚合检查
make test-config-version

# 前端（frontend/package.json）
pnpm run dev / build / test(rstest) / test:e2e(playwright) / typecheck
```

配置入口：根 `config.yaml`（模板 `config.example.yaml`，模型/工具/沙箱/子 Agent/预算全在其中），环境变量可覆盖任意字段（`api_key: $OPENAI_API_KEY`）。安全默认值值得注意：`sandbox.allow_host_bash: false`，默认 LocalSandboxProvider 不提供 shell 隔离（`config.example.yaml` sandbox 段）。

## 7. 测试与工程质量

- 规模：`backend/tests` 1392 个 Python 测试文件、`frontend/tests` 455 个文件（find 统计），覆盖单元/契约/回放 e2e。
- CI：`.github/workflows/` 19 个工作流——backend-unit-tests、backend-blocking-io-tests（专查阻塞 IO）、backend-windows-live-policy、frontend-unit-tests、e2e-tests、replay-e2e、lint-check、extension-package-check、skill-review-ci、nightly、verify-versions、sandbox 镜像 smoke 等；harness 边界由 `backend/tests/test_harness_boundary.py` 在 CI 强制。
- 质量观察：工程纪律极高。pyproject 依赖注释解释每个直接依赖的存在理由；CHANGELOG 每条修复带 issue 号与安全影响描述（如 memory 凭据掩码修复 #7665）；安全敏感路径（CSRF、auth、loopback bind、secret masking）均有 pin 测试；文档密度罕见（backend/docs 下 25+ 专题文档，每个模块目录带 AGENTS.md 面向 AI 编码者）。ruff + eslint + tsc 全量静态检查（`backend/pyproject.toml`、`frontend/package.json`）。

## 8. 社区与维护状态

- 活跃度：push_at 2026-10-30（分析前一天）；v2.0.0 发布 2026-06-25，v2.1.0 2026-09-24，v2.2.0 2026-10-29——该里程碑合并 **1068 个 PR**（`gh api releases`、`CHANGELOG.md`）。当前处于 2.3.0 开发期（`backend/pyproject.toml` version 2.3.0-dev）。
- 贡献者：GitHub contributors API 返回 55 人；issue 开放 991 个（gh api，未确认其中 bug/feature 构成）。字节主导开发 + 社区贡献并存，CHANGELOG 显示大量带编号的社区 PR。
- 生态位：README 标注 2026-02-28 曾登顶 GitHub Trending；姊妹项目 LLM Space（调试/回放/基准工具）；深度绑定 BytePlus InfoQuest 搜索与火山引擎 Coding Plan 生态（`README.md`）。
- 版本迁移沟通清晰：v1 深度研究框架移至 `main-1.x` 分支继续接受贡献（`README.md` NOTE）。

## 9. 亮点与风险

亮点：

1. **Harness/App/契约三层架构 + CI 强制单向依赖**，框架可独立发布（`deerflow-harness`），插件面收敛到 `deerflow-extension-api`（`docs/ARCHITECTURE.md` §3、`backend/pyproject.toml`）。
2. **中间件即运行时**：59 个 middleware 把上下文工程、安全、治理做成可组合层，是 LangChain middleware 模式目前最完整的工业级实践之一（`backend/packages/harness/deerflow/agents/middlewares/`）。
3. **子 Agent 治理完备**：并发容量、每 run 上限、turn/token 双预算、验收标准、状态契约、持久批处理，直接对齐生产成本失控风险（`subagents/`、`config.example.yaml`）。
4. **安全默认值保守**：host bash 默认禁用、Starlette 版本 pin 因安全边界、回执防幻觉机制、PII 脱敏中间件（`config.example.yaml`、`backend/pyproject.toml`、`middlewares/`）。
5. **测试与文档密度罕见**：后端近 1400 个测试文件、19 条 CI 流水线、每个模块带 AGENTS.md（`backend/tests/`、`.github/workflows/`）。

另有一个常被忽略的交付面：harness 内置 TUI（`backend/packages/harness/deerflow/tui/`），加上 GitHub webhook 渠道（`routers/github_webhooks.py`，来自外部评论者的 webhook 消息被标记为不可信、禁用管理类工具，`agents/lead_agent/agent.py` `_WEBHOOK_CHANNELS`），说明同一 Agent 内核同时面向 Web、IM、终端与代码托管平台四种交互入口。

风险/局限：

1. **复杂度门槛高**：单仓库双语言 + 三服务 + 64 版 config schema，二开与运维成本不低；`config.example.yaml` 即 2000+ 行注释。
2. **版本断层**：v1（planner/researcher 编排）与 2.0 完全不兼容，教程与生态资产分裂在 `main-1.x` 与 main 两条线上（`README.md`）。
3. **生态绑定倾向**：README 大幅导向 BytePlus InfoQuest、火山引擎 Coding Plan、Volces 容器镜像，中立自托管体验需自行替换（`README.md`、`config.example.yaml`）。
4. **多租户安全需自查**：authz/授权默认关闭时所有登录用户可用管理能力（CHANGELOG #7665 暗示的暴露面），生产部署需显式配置（`config.example.yaml`、`backend/docs/AUTH_DESIGN.md`）。

## 10. 总结评价

DeerFlow 是当前开源 Agent 基础设施里工程完成度最高的项目之一：从"深度研究框架"进化为"超级 Agent 运行时"，把模型调用之外的所有脏活（沙箱、文件系统、记忆、技能、子 Agent 治理、IM 渠道、观测）做成了一层可拆装的 harness，并以罕见的测试/文档/CI 纪律维护。它适合三类人：想自托管一个能干活的通用 Agent 产品的团队（开箱即用 + Docker/Helm）；做 Agent 框架研究的工程师（middleware 链、子 Agent 容量治理、回执防幻觉都是可直接抄的设计）；以及想给字节系模型生态找落地场景的开发者。想读源码建议从 `docs/ARCHITECTURE.md` 入手，再按 `agents/lead_agent/agent.py` → `subagents/executor.py` → `skills/` 的顺序读；想找 v1 的 planner/researcher 编排则去 `main-1.x` 分支。
