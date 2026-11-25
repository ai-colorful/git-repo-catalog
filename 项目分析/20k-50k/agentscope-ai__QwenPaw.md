# agentscope-ai/QwenPaw 项目分析

> 星数 37822（快照 2026-10-24）｜主语言 TypeScript（GitHub 统计）｜档位 20k-50k｜分析日期 2026-11-17

## 1. 项目速览

一句话定位：**QwenPaw（Qwen Personal Agent Workstation，千问个人智能体工作台）是 AgentScope 团队出品的开源个人 AI 助理**——部署在自己机器或云端，通过钉钉/飞书/微信/QQ/Telegram/Discord 等 18 个聊天渠道对话，以 Skills/插件/MCP 扩展能力，数据留在本地。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | QwenPaw（前身名 CoPaw） | `README_zh.md`、`src/qwenpaw/constant.py`（`LEGACY_QA_AGENT_ID = "CoPaw_QA_Agent_0.1beta1"`）、`Makefile`（标题仍为 "CoPaw Test & Coverage Makefile"） |
| 星数 / Fork | 37823 / 3289（gh api，2026-11-17） | gh api repos/agentscope-ai/QwenPaw |
| 语言 | GitHub 标注 TypeScript；实际 Python 后端为主体（.py 3083 个文件，其中 `src/qwenpaw` 1120 个约 38 万行；.ts 1012 + .tsx 1001 + .js 299） | 仓库实测文件统计 |
| 许可证 | Apache-2.0 | `LICENSE` |
| 创建时间 | 2026-02-24，9 个月达成 37.8k 星 | gh api `created_at` |
| 最近推送 | 2026-11-16（分析前一天仍活跃）；当前版本 2.3.1b1，最近正式版 v2.3.0（2026-11-11） | gh api `pushed_at`；`src/qwenpaw/__version__.py`；gh api releases |
| 贡献者 | 约 266 人（contributors 分页统计） | gh api contributors |
| Python 版本 | >=3.11, <3.14 | `pyproject.toml` |

## 2. 项目是什么，解决什么问题

QwenPaw 定位"你的个人 AI 助理"：跑在用户自己的电脑/服务器/Docker 上，不需要第三方托管，通过用户日常已经在用的聊天软件（钉钉、飞书、微信、企业微信、QQ、Telegram、Discord、iMessage、Slack 等）与它对话，让它执行定时任务、处理文档、写代码、搜集信息、推送消息（`README.md` 特性表、`pyproject.toml` 项目描述）。

核心卖点有五块（均见 `README.md` 特性表）：

1. **三层记忆**：工作上下文 + 全量对话历史 + 基于 ReMe（agentscope-ai 生态的记忆库项目，依赖 `reme-ai==0.4.2.0`）的自演化个人知识库，对话与资源沉淀为可读可编辑可检索的 Markdown 记忆。
2. **本地/云端模型双轨**：内置面向 Agent 场景训练的 QwenPaw-Flash 系列模型（2B/4B/9B/27B/35B-A3B）与 QwenPaw Local 运行时（llama.cpp，无需 API Key），同时接 Ollama、LM Studio 及 14+ 云端供应商（`src/qwenpaw/local_models/`、`README.md`）。
3. **安全内建**：内核级 Sandbox、Tool Guard、File Guard、Skill Scanner、Access Policy，危险命令在执行前拦截（`src/qwenpaw/sandbox/`、`src/qwenpaw/security/`）。
4. **多 Agent 与并行**：独立 Agent 各有记忆与技能，运行时子 Agent，ACP（Agent Communication Protocol）跨系统编排（`src/qwenpaw/agents/acp/`）。
5. **全渠道触达**："one instance, all channels"，一套实例同时服务所有聊天平台。

与 Qwen/阿里生态的关系（核实结论）：README_zh"为什么叫 QwenPaw"一节明确写着"QwenPaw 既是 Qwen Personal Agent Workstation（千问个人智能体工作台），也寓意 Qwen 的智识与 Paw 的温度"（`README_zh.md` L485 附近）；由 AgentScope 团队构建（`README.md` "Built By"：agentscope-ai 组织）；依赖阿里系组件随处可见——`pyproject.toml` 依赖 `alibabacloud-tea-openapi` 等阿里云 SDK、`src/qwenpaw/providers/provider_catalog.py` 内含 DashScope/ModelScope/Aliyun Coding Plan 等 provider 分组、部署文档提供阿里云 ECS 一键部署与 ACR 镜像（`README.md`）、渠道含钉钉/元宝/小翼（`plugins/channel/`）、图片资源走 alicdn。可判定为阿里 Qwen 生态的官方开源项目（组织归属与生态痕迹均为仓库内客观证据）。

增长背景（客观陈述，不臆断）：2026-02-24 创建，9 个月 37.8k 星（日均约 140 星，属超高速增长），同期上了 trendshift 趋势榜徽章（`README.md`）；README 有英/中/日/俄/越 5 个语言版本；桌面安装包、Docker、pip、云平台免费托管多入口降低使用门槛；对中文 IM（钉钉/微信/QQ/飞书）的完整覆盖在同类"个人助理"开源项目中少见，且背靠 AgentScope（阿里开源多智能体框架）既有社区。

## 3. 技术栈与依赖

- **后端**：Python 3.11-3.14，FastAPI + uvicorn（`src/qwenpaw/app/app_services/__init__.py` 提及 FastAPI lifespan），异步 httpx/websockets，apscheduler 定时，playwright 浏览器自动化，mcp>=1.28（MCP 协议）（`pyproject.toml`）。
- **Agent 框架**：核心 agent 循环直接复用自家 `agentscope==2.0.10` 框架（`pyproject.toml`）：`agentscope.tool.Toolkit`、`agentscope.agent.ReActConfig`、`agentscope.middleware.MiddlewareBase`（`src/qwenpaw/runtime/builder.py`）。
- **记忆**：ReMe（`reme-ai==0.4.2.0`）+ ONNX/transformers 嵌入模型（`pyproject.toml`、`src/qwenpaw/agents/memory/`）。
- **本地模型运行时**：llama.cpp（`src/qwenpaw/local_models/llamacpp.py`），模型下载管理接 modelscope/huggingface_hub。
- **前端/桌面**：React 18 + TypeScript + Vite + Ant Design（含 @ant-design/x-markdown）+ Monaco Editor + Lexical；桌面端 Tauri（`console/package.json`、`console/src-tauri/`）；控制台组件用自家 npm 包 `@agentscope-ai/design`、`@agentscope-ai/chat`。
- **打包分发**：PyPI（`qwenpaw`）、Docker Hub（`agentscope/qwenpaw`）、Tauri 桌面安装器、install.sh/install.ps1 脚本（`README.md`）。

## 4. 目录结构与代码组织

顶层目录（快照 HEAD，7841 个条目）：

| 目录 | 职责 |
| --- | --- |
| `src/qwenpaw/` | Python 后端主体（38 万行），约 38 个子模块 |
| `console/` | React/TS 控制台 + Tauri 桌面端（1581 个 ts/tsx） |
| `plugins/channel/` | 18 个聊天渠道适配器（独立插件包） |
| `plugins/apps/` | 内置应用插件：agent-kanban、qwenpaw-creator、qwenpaw-data |
| `plugins/bundle/` | 捆绑插件：chrome、cloudpaw、computer-use、qwenpaw-openui、qwenpaw-pet 等 |
| `packages/qwenpawmail-mcp/` | 独立子包：邮件 MCP server |
| `tests/` | 1002 个 Python 测试文件（unit/integration/contract/e2e/channel_support/typing） |
| `e2e/` | Playwright 端到端测试套件 |
| `.github/workflows/` | 44 个 CI workflow |
| `deploy/` | Dockerfile、entrypoint、依赖维护文档 |
| `docs/`、`website/` | 文档与官网 |
| `design.md` | UI 设计语言规范（69KB，橙色品牌 `#FF7F16`） |

`src/qwenpaw/` 关键子模块：`runtime/`（请求编排）、`loop/`（循环门控）、`agents/`（agent 构建与工具）、`plugins/`（插件框架）、`channel_protocol/`（渠道协议）、`providers/`（模型接入）、`sandbox/`+`security/`（安全）、`memory/`（记忆）、`hub/`（多用户托管）、`market/`（市场）、`cli/`、`app/`（FastAPI 服务层）。

组织方式评述：单体仓库 + 多发布物（PyPI 包 + npm/桌面 + 子包），渠道与能力全部插件化（v2.3.0 起"channels as plugins"），插件用声明式 manifest（plugin.json/channel.json）+ 独立 Python 包 + requirements.txt，主程序与插件通过 `channel_protocol/` 的进程隔离协议通信（`plugins/channel/telegram/plugin.json` 中 `process_mode: "runner_process"`）。

## 5. 核心架构与关键实现

### 5.1 8 阶段请求编排（runtime 层）

每个请求由 per-workspace 的 `Runtime` 编排，固定 8 个 hook 阶段，插件/模式只能通过 `LifecycleHook` 挂载，agent 触碰代码只有 build+execute 两步（`src/qwenpaw/runtime/runtime.py` 模块注释）：

```python
# src/qwenpaw/runtime/phases.py（节选）
class Phase(str, Enum):
    PRE_DISPATCH = "pre_dispatch"
    POST_DISPATCH = "post_dispatch"
    PRE_AGENT_BUILD = "pre_agent_build"
    POST_AGENT_BUILD = "post_agent_build"
    PRE_EXECUTE = "pre_execute"
    POST_RESPONSE = "post_response"
    ON_ERROR = "on_error"
    FINALLY = "finally"
```

`Envelope` 是 SSE 状态机，`AgentBuilder` 按请求组装 agent（Toolkit 装载工具、ReActConfig 配置 ReAct 循环、Middleware 挂中间件），`AgentExecutor` 用 heartbeat 包裹回复流（`src/qwenpaw/runtime/{runtime,builder,executor,envelope}.py`）。与 agent 内循环正交的还有 `loop/gates/` 一组门控（budget、iteration、limits、doom_loop、completion、rubric，`src/qwenpaw/loop/gates/`），用于防死循环与预算控制。

### 5.2 渠道即插件（18 个聊天平台适配器）

`plugins/channel/` 下 18 个适配器：azure_bot、dingtalk、discord、feishu、imessage、matrix、mattermost、mqtt、onebot、qq、sip、slack、telegram、voice、wechat、wecom、xiaoyi、yuanbao。每个适配器是标准插件包，manifest 描述入口与配置 schema（`plugins/channel/telegram/plugin.json` + `channel.json`）：

```json
{
  "schema_version": 2,
  "channel_key": "telegram",
  "process_mode": "runner_process",
  "entrypoint": { "scope": "runner",
    "module": "qwenpaw_channel_telegram.driver",
    "qualname": "TelegramDriver" },
  "requirements_file": "requirements.txt"
}
```

渠道以独立 runner 进程运行，宿主通过 `src/qwenpaw/channel_protocol/`（canonical 消息规范化、生命周期、outbound 预算、回执、可靠性重试等 20+ 模块）与其 RPC 通信；这是"easily extensible"承诺的落点之一——新增平台只需按协议实现 Driver 并提交插件包，不需动宿主代码。

### 5.3 模型接入层（38 个 provider 条目，Qwen 双轨）

`src/qwenpaw/providers/provider_catalog.py` 定义 38 个 provider 条目，含 DashScope、ModelScope、Aliyun Coding/Token Plan、Zhipu 等分组。设计上 DashScope provider 继承 OpenAIProvider 复用兼容层，仅重写模型实例构造为原生 `DashScopeChatModel`（`src/qwenpaw/providers/dashscope_provider.py` 模块注释），Qwen3 系列的上下文窗口与截断策略有专门条目（`src/qwenpaw/providers/context_windows.py`：qwen3.8/qwen3-max 等达 1M token）。本地侧 `src/qwenpaw/local_models/llamacpp.py` 提供 QwenPaw Local 运行时（免 API Key 跑 QwenPaw-Flash），另有 `local_decision_model.py`（Piccolo 本地小模型）做大模型/小模型协同的 Advisor Mode（`README.md` News v2.3.0）。

### 5.4 安全纵深（沙箱 + 双 Guard + 技能扫描）

- **沙箱**：按平台分文件实现——`linux_sandbox.py`、`bubblewrap_sandbox.py`、`macos_sandbox.py`、`windows_appcontainer_sandbox.py`、`windows_elevated/unelevated_sandbox.py`（`src/qwenpaw/sandbox/`），跨三平台真实落地而非仅文档声明。
- **Tool Guard**：`src/qwenpaw/security/tool_guard/`（engine、approval、safety_checks、execution_level）在工具执行前做策略判定；运行时层再挂 `src/qwenpaw/runtime/tool_guard.py`。
- **Skill Scanner**：`src/qwenpaw/security/skill_scanner/` 对安装的 Skill 做扫描，配合 `plugins/check.py`、`governance/`、`plugins/execution/`（进程树、RPC、diagnostics）构成插件供应链防线。
- 辅以 Skill 系统的 Pool/Registry/Store 分层（`src/qwenpaw/agents/skill_system/`，内置 49 个 SKILL.md，中英成对，Anthropic 风格 frontmatter，见 `src/qwenpaw/agents/skills/cron-en/SKILL.md`）。

## 6. 构建与运行

```bash
# 安装（三选一，README.md Quick Start）
pip install qwenpaw && qwenpaw init --defaults && qwenpaw app   # 打开 http://127.0.0.1:8088/
curl -fsSL https://qwenpaw.agentscope.io/install.sh | bash       # 脚本一键装（自动装 uv）
docker run -p 127.0.0.1:8088:8088 agentscope/qwenpaw:latest      # Docker

# 开发（Makefile）
make install-dev          # pip install -e .
make test                 # 全量 pytest tests/
make test-unit|test-contract|test-integration|test-channel
make quick                # 快速冒烟（隔离工作目录）
make coverage-full        # 覆盖率

# 前端（console/package.json）
npm run dev|build|lint|test:run
```

## 7. 测试与工程质量

- 测试规模可观：`tests/` 下 1002 个 Python 测试文件（unit 786、integration 187、contract 13、channel_support 7、e2e 3），另有独立 Playwright `e2e/` 套件与前端 vitest（`console/package.json` 有 coverage 配置）。
- CI 密度高：`.github/workflows/` 44 个 workflow，覆盖主测试、nightly、e2e smoke/integration、前端测试、CodeQL、桌面端构建/发布、渠道/插件发布守护、PR 欢迎与防刷（pr-spam-gate）、release 看门狗等——发布工程化程度在开源个人项目中罕见。
- 工程约束：pre-commit（black、flake8 等，`.pre-commit-config.yaml`）、`.flake8`、类型存根测试目录 `tests/typing/`。
- 观察点：`pyproject.toml` 对 anyio/textual 等依赖有细致的上界锁定与注释（引用具体 issue 编号），显示维护者对运行时兼容问题的追踪习惯。

## 8. 社区与维护状态

- 活跃度：创建于 2026-02-24，最后一次 push 为 2026-11-16（分析前一天），提交强度持续（gh api `pushed_at`）。
- 发布节奏：正式版约每 3-6 周一个 minor（v2.1.0 2026-08-13 → v2.2.0 2026-09-03 → v2.2.2 2026-10-15 → v2.3.0 2026-11-11），minor 前有 3-4 个 beta（gh api releases）。
- 贡献者：约 266 人（gh api contributors 分页），有 first-time-contributor 欢迎机器人与 PR 欢迎流程（`.github/workflows/first-time-contributor-welcome.yml`、`issue-welcome.yml`）。
- 生态出口：PyPI、Docker Hub、阿里云 ACR、Tauri 桌面安装器、AgentScope Platform 免费云托管、ModelScope Studio（`README.md`）。
- 社区渠道：Discord、X、钉钉群、小红书（`README.md` Contact Us）。
- 开放问题 1173 个（gh api），对 9 个月新仓库属于正常偏高位，未确认响应时长。

## 9. 亮点与风险

亮点：

1. **渠道覆盖是同类项目中最全的**：18 个适配器且全部插件化、进程隔离，含 iMessage/企业微信/腾讯元宝/电信小翼等稀缺渠道（`plugins/channel/`）。
2. **安全实现扎实且可核验**：三平台真实沙箱实现 + Tool Guard/Skill Scanner 双层拦截 + 插件供应链治理（`src/qwenpaw/sandbox/`、`security/`、`plugins/execution/`）。
3. **模型接入双轨完整**：38 个云端 provider 条目 + 免 Key 本地 llama.cpp 运行时 + 本地小模型协同（`providers/`、`local_models/`）。
4. **工程化程度高**：44 个 CI workflow、1002 个测试文件、多语言 README、发布看门狗。
5. **架构清晰**：8 阶段 hook 编排 + 协议化渠道 + manifest 驱动插件，宿主与能力解耦（`runtime/phases.py`、`channel_protocol/`）。

风险/局限：

1. **体积与复杂度大**：后端 38 万行 Python + React/Tauri 前端，个人部署排障成本高；1173 个开放 issue 积压。
2. **深绑定阿里生态**：遥测（`qwenpaw init` 收集匿名信息，README Telemetry 节）、ACR/阿里云一键部署等对非阿里用户是多余面；部分依赖（alibabacloud SDK）较重。
3. **发布节奏快、接口变动大**：v2.3.0 刚完成插件系统重构（channels as plugins），插件 SDK 与版本兼容层（`src/qwenpaw/_version_compat.py` 明确写"上界检查暂被禁用"）仍处于过渡态。
4. **"QwenPaw-Flash 专项训练"的效果无仓库内证据**，README 宣称与实际质量需使用者自行验证（未确认）。

## 10. 总结评价

QwenPaw 是 2026 年个人 AI 助理赛道里增长最快的开源项目之一：它把"能装、能连聊天软件、能扩展、数据不出门"四件事做成了一条完整产品线——Python 后端（基于自家 AgentScope 2.0 框架）负责 agent 编排与安全沙箱，React/Tauri 前端负责控制台与桌面体验，18 个渠道适配器和 38 个模型 provider 条目撑起"一个实例、所有入口"的承诺。9 个月 37.8k 星的背后是阿里系团队的持续高强度投入与中文 IM 场景的差异化，而非偶然爆款。适合想自托管个人助理的进阶用户（尤其钉钉/微信/飞书用户）与安全敏感的本地模型用户；对想研究 agent 运行时设计（hook 生命周期、协议化渠道、插件进程隔离）的工程师，`src/qwenpaw/runtime/`、`channel_protocol/`、`sandbox/` 是很好的活教材；但小团队直接改造需评估其体量与快速变动的插件接口。
