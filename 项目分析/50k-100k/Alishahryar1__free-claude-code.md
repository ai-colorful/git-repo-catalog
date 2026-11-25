# Alishahryar1/free-claude-code 项目分析

> 星数 62794（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：一个本地运行的 AI 编码代理网关（local proxy），把 Claude Code、Codex 等 12 种编码 agent 统一接入 61 家大模型提供商（免费/付费/订阅/本地），内置模型路由、故障回退与限额控制。

| 项目 | 值 |
| --- | --- |
| 名称 | Alishahryar1/free-claude-code（FCC） |
| 星数 | 62794（快照）；gh api 实测 65263（2026-10-31），10.2k forks |
| 语言 | Python（锁定 3.14.10，uv 管理） |
| 许可证 | AGPL-3.0-only（`LICENSE`，作者 Ali Khokhar） |
| 创建 / 最近 push | 2026-01-28 / 2026-10-31（当天仍在活跃提交） |
| 最新 release | v6.18.5（2026-10-29，累计 100+ 个 release） |
| 代码规模 | 主包约 7.5 万行 Python（`src/free_claude_code/`） |

## 2. 项目是什么，解决什么问题

FCC 解决的是"工具与模型解耦"问题：Claude Code、Codex、OpenCode、Cline、Aider 等编码 agent 各自绑定自家的 API 端点和认证方式，用户想换模型（尤其是免费额度模型或本地模型）往往无从下手。FCC 在本地起一个代理服务器（默认 `fcc-server`），同时伪装成 Anthropic Messages、OpenAI Responses、Gemini 三种 API 方言（`src/free_claude_code/api/routes.py` 暴露 `/v1/messages`、`/v1/responses`、`/v1beta/models`），于是任何 agent 只需把 base URL 指向 FCC 即可任意切换上游模型。

接入方式刻意做成了"零配置"：`fcc-claude` 等启动器（`src/free_claude_code/harnesses/claude.py`）通过注入 `ANTHROPIC_BASE_URL` 与 `ANTHROPIC_AUTH_TOKEN` 环境变量把官方 CLI 指到本地代理，并代办 onboarding、模型目录注入（让 `/model` 选择器里直接出现 FCC 模型）。Desktop 端用 pystray 提供托盘图标，另有 VS Code/JetBrains/Claude Desktop/Codex App 等集成适配（`src/free_claude_code/harnesses/`）。

模型侧由 `ModelRouter`（`src/free_claude_code/application/routing.py`）承接：Claude Code 的四档模型槽位（Fable/Opus/Sonnet/Haiku）可分别映射到任意 `provider/model`，并支持按顺序回退的 Fallback Models 列表与每档独立的 reasoning effort 覆盖。

围绕核心代理还有三层增值能力：可靠性层（重试、并发准入、多 key 回退，见第 5 章）、消息层（Discord/Telegram 机器人远程驱动会话，支持语音转写，`src/free_claude_code/messaging/`，约 9100 行）、浏览器 Code 会话（`src/free_claude_code/application/code_sessions/`，在 Admin UI 中原生运行 Codex 会话）。

README 明确声明项目独立、与 Anthropic 无关联，并强调"FCC 遵循提供商 ToS，不再被允许时移除对应集成"（README Features 节），对 Kimi Code、QwenCloud Coding Plan 等订阅型接入均引用了官方使用条款（README Provider-specific setup 节）。这是一个定位合规的中立网关，而非破解工具。

## 3. 技术栈与依赖

- 语言与运行时：Python 3.14（`pyproject.toml` 锁定 `==3.14.10`），uv 管理依赖（`uv.lock` 418KB，全部锁定）。
- Web 层：FastAPI + uvicorn + Starlette（`pyproject.toml`）；SSE 流式响应用 hand-rolled AsyncIterator 封装。
- HTTP 客户端：httpx 与 httpx2 并存（`src/free_claude_code/providers/failure_policy.py` 同时 import 两者，用于兼容不同上游 SDK 栈），另用 aiohttp、openai 官方 SDK、github-copilot-sdk。
- 数据：pydantic v2（Settings 模型约 169 个字段，`src/free_claude_code/config/settings.py`）+ SQLite（SQLAlchemy 仅作连接辅助，自写 schema 迁移 `src/free_claude_code/runtime/sqlite_migrations/`）。
- 消息与语音：discord.py、python-telegram-bot、grpcio + nvidia-riva-client（NIM 语音转写）、本地 Whisper 走 torch/transformers（可选 extra `voice_local`）。
- 工程链：hatchling + hatch-vcs（版本号来自 git tag）、ruff（lint+format）、ty 类型检查器、pytest 系列。
- 发布：GitHub Actions post-merge 自动发布到 PyPI（`.github/workflows/post-merge.yml`）。

## 4. 目录结构与代码组织

顶层是标准的 `src/` 布局单包仓库（无 monorepo/多包）：

| 目录 | 职责 |
| --- | --- |
| `src/free_claude_code/api/` | FastAPI 适配层：路由、handlers、Admin UI 静态资源（约 4900 行） |
| `src/free_claude_code/application/` | 用例层：模型路由、执行编排、code_sessions、web_tools（约 5100 行） |
| `src/free_claude_code/core/` | 领域核心：三套 API 方言模型/流协议/转换（约 14400 行） |
| `src/free_claude_code/providers/` | 61 家提供商适配 + 准入/重试/失败分类（约 22100 行，最大模块） |
| `src/free_claude_code/harnesses/` | 12 种 agent 的环境注入与集成（claude.py、codex.py 等） |
| `src/free_claude_code/messaging/` | Discord/Telegram 平台层：命令分发、转写、渲染（约 9100 行） |
| `src/free_claude_code/runtime/` | 进程装配、生命周期、SQLite 存储、codex 协议（约 7600 行） |
| `src/free_claude_code/config/` | Settings、凭据、provider 目录、环境迁移（约 5400 行） |
| `src/free_claude_code/cli/` | 入口命令、桌面托盘、doctor、uvicorn 封装（约 4000 行） |
| `tests/` `e2e/` `smoke/` | 单元/契约 448 文件、Playwright e2e 30 文件、smoke 72 文件 |
| `scripts/` | 三平台安装/卸载/更新脚本、CI 分片脚本、发布校验 |

组织方式评述：分层清晰（api→application→core，providers 自成插件域），核心域与外部适配严格分离——`providers/base.py` 用 ABC 定义 Provider 接口，新增提供商只需实现 `stream_messages`/`stream_responses`/`stream_gemini`。Admin UI 是内嵌的原生 JS/CSS（`api/admin_static/`，含 61 家提供商图标与来源声明 `SOURCES.md`），未引入前端框架。

## 5. 核心架构与关键实现

### 5.1 三方言网关与请求编排

`src/free_claude_code/api/app.py` 的 `create_app()` 是纯工厂：挂四个中间件（请求关联 ID、生命周期、结果统计、Admin no-store），依次 include admin、code_sessions、主代理三组路由。`routes.py` 的 `/v1/messages` 入口把请求交给 `MessagesHandler`（`api/handlers/messages.py`），后者组装 `ProviderExecutor`（进度超时、token 计数、原始 payload 日志开关）与 `ModelRouter`，流式与非流式统一走 `anthropic_sse_streaming_response`。错误映射做得很细：`app.py` 按 API 方言返回对应格式的错误 payload（Anthropic/Gemini/OpenAI 各有一套 `*_error_payload`）。

### 5.2 可靠性引擎：准入与失败分类

`src/free_claude_code/providers/admission.py`（1049 行）实现了"提供商自持准入"：并发容量限制（`admission_capacity.py`）、协调式重试（上游瞬态错误最多 10 次尝试、指数退避 2s→60s 带抖动）、probe 预算控制（`ProviderRecoveryExhausted`）。`failure_policy.py` 把各 SDK 异常分类为鉴权失败/配额/限流/上下文超窗等十种 `FailureKind`，供回退决策使用。README 宣称的"自动模型/多 key 回退"即由这层与 `ModelRouter` 的 fallback 列表协同完成。

```python
# src/free_claude_code/providers/base.py（节选）
class BaseProvider(ABC):
    @abstractmethod
    async def list_model_infos(self) -> frozenset[ProviderModelInfo]: ...
    @abstractmethod
    def stream_messages(self, request: MessagesRequest, ...) -> AsyncIterator[str]:
        """Validate the request before yielding a response in Anthropic SSE format."""
    @abstractmethod
    def stream_responses(self, request: OpenAIResponsesRequest, ...) -> AsyncIterator[str]:
        """Validate the request before yielding OpenAI Responses SSE events."""
```

### 5.3 模型路由与 Claude Code 槽位映射

`src/free_claude_code/application/routing.py` 定义 `_ROUTE_SETTINGS = (("fable", ...), ("opus", ...), ("haiku", ...), ("sonnet", ...))`，把 Claude Code 的四个模型档位解析成 `ProviderModelTarget`（provider_id + provider_model + reasoning 策略）；网关侧用 `gateway_model_ids.py` 编造可解码的模型 ID 注入 agent 的模型选择器。请求在进入 provider 前统一做 reasoning effort 解析（Inherit/From client/固定档位，`application/reasoning.py`）。

### 5.4 安全边界与凭据处理

Admin 管理面被硬性限制在本机：`src/free_claude_code/api/admin_security.py` 校验请求 Host 与 Origin 必须是 loopback（localhost/127.0.0.0/8、::1），非本地来源直接拒绝；代理本身可选用 bearer token（README "Proxy Authentication"）。凭据一律用 pydantic `SecretStr` 存储，`config/api_keys.py` 把有序多 key 规范化为去重元组并支持按编号选取；会话与消息数据落在 `~/.fcc/` 的 SQLite 中，`runtime/sqlite_database.py` 含符号链接/重定向防护与 schema 迁移。OAuth 类接入（ChatGPT 订阅、GitHub Copilot）走官方 SDK 完成设备授权，凭据不经过第三方。

## 6. 构建与运行

来自 `pyproject.toml`、`CONTRIBUTING.md` 与 README：

```bash
# 用户安装（三平台脚本，装 uv + Python 3.14.10 + FCC 本体）
curl -fsSL "https://raw.githubusercontent.com/Alishahryar1/free-claude-code/main/scripts/install.sh" | sh

# 开发运行（uv 自动装指定版本 Python；勿用全局解释器）
uv run fcc-server          # 启动代理 + Admin UI
uv run fcc-claude          # 以代理环境启动 Claude Code
uv run fcc-doctor          # 诊断环境

# 质量检查（PR 前必须全绿）
./scripts/ci.sh            # ruff format/check + ty + pytest + playwright

# 测试
uv run --locked pytest     # 单元/契约（tests/）
uv run --locked pytest e2e # Playwright 浏览器端到端
```

更新与卸载分别提供 `fcc-update` 命令与三平台 uninstall 脚本（`scripts/`）。

## 7. 测试与工程质量

- 测试规模：`tests/` 448 个文件、约 4400 个测试函数（grep 统计），按 api/application/core/providers/messaging/harnesses 分域镜像源码结构，另有 `contracts/`（确定性契约检查，marker 见 `pyproject.toml`）；`e2e/` 30 个 Playwright 文件覆盖 Admin UI；`smoke/` 72 个文件做真服务可选冒烟（`live`/`provider` marker）。
- CI（`.github/workflows/tests.yml`）：三平台（Linux/Windows/macOS）pytest 分片 3/3/2 + Playwright 分片 3/3/3，按 least_duration 智能分组，`worksteal` 负载均衡；所有 action 固定到 commit SHA；启用 uv 的 malware-check。
- 强约束：CI 首个 job 就禁止 `# type: ignore`、`# ty: ignore` 与 `from __future__ import annotations`（tests.yml 第 34-39 行），强制修根因而非压制告警——在开源项目里极为少见。
- 静态质量：ruff 全家桶规则集（E/W/F/I/UP/B/C4/SIM/PERF/RUF）+ ty 全量类型检查；提交规范要求 `patch:/minor:/major:` 前缀驱动自动发版（CONTRIBUTING.md）。

## 8. 社区与维护状态

- 活跃度：创建于 2026-01-28，9 个月到 6.5 万星；gh api 显示最近 push 为 2026-10-31（分析当天），open issues 330。
- 发布节奏：release 总数超过 100（分页 API 返回满 100），最新 v6.18.5 发布于 2026-10-29；仅 10 月 25-29 五天就发了 5 个版本（v6.18.1-v6.18.5），主干合并即自动发 PyPI（post-merge.yml）。
- 贡献者结构：30 名贡献者（gh api contributors），头部贡献者 1016 次提交，属于典型的强主导者单核模式；CONTRIBUTING.md 明确文档 PR 保留给作者、禁止 Docker 集成 PR，PR 面收得很紧。
- 版本号已到 6.x 主线，迭代速度远超常规开源节奏（未确认是否有团队/机构支持）。

## 9. 亮点与风险

亮点：

1. 工程质量顶级：4400+ 测试、三平台分片 CI、禁止类型压制、SHA 固定 action、uv 锁定依赖，测试金字塔完整（单元→契约→e2e→smoke）。
2. 架构干净：三 API 方言、12 agent、61 提供商全部通过接口解耦（`providers/base.py`），核心域无外部 SDK 泄漏，扩展点明确。
3. 可靠性设计认真：准入并发、协调重试、失败分类、多级回退是代理类工具少见的成体系实现（`providers/admission.py`、`failure_policy.py`）。
4. 合规姿态明确：AGPL-3.0-only、声明独立性、主动引用各提供商 ToS 并承诺下架被禁集成（README）。
5. 安全边界内建：Admin 限 loopback、SecretStr 凭据、可选代理鉴权、SQLite 防符号链接迁移。

风险与局限：

1. 依赖订阅账号的接入（ChatGPT/Copilot OAuth、各家 Coding Plan）本质上让个人订阅承载 agent 流量，是否持续被提供商允许由对方决定，免费额度与条款随时可能变化（README 亦有声明）；用户应自行确认所购订阅的适用范围，本分析不构成使用建议。
2. Python 精确锁定 3.14.10（`pyproject.toml`），升级窗口窄，对新平台/发行版兼容性构成硬约束。
3. 单核维护：约 30 位贡献者中头部一人占 1016 次提交，文档 PR 与发版权限集中，总线风险存在。
4. 项目 9 个月即 6.x 版本、日更级发布，API/配置面（61 提供商 × 12 agent 矩阵）的长期维护成本高，冷门提供商适配可能出现测试覆盖外的回归（未确认）。

## 10. 总结评价

free-claude-code 是"AI 编码代理网关"这一新品类里目前工程质量最高的代表之一：它把"换模型"这件原本需要改配置、破解环境变量的事，做成了一个有准入控制、有失败分类、有三平台安装器和浏览器会话的完整产品；其测试与 CI 水准（4400+ 用例、禁 type-ignore、三平台分片）即使放到头部开源项目中也属第一梯队。星数一年内冲到 6.5 万既有品类红利的成分，也有真实产品力支撑。适合两类人：想在自己的 Claude Code/Codex 工作流里自由切换免费与本地模型的终端用户；以及研究代理网关如何做协议转换、流式可靠性与凭据治理的工程师——`providers/` 与 `core/` 的分层是很好的范本。源码阅读建议从 `api/app.py` → `api/routes.py` → `application/routing.py` → `providers/base.py` 这条主干进入。
