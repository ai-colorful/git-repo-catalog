# HKUDS/nanobot 项目分析

> 星数 49301（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：**nanobot 是一个超轻量、开源自托管的个人 AI agent 框架（Python），可在浏览器 WebUI、终端或聊天应用中通过统一的 agent 内核操作文件、执行 shell、联网搜索、定时任务并保持长期记忆。**

| 基本事实 | 值 |
|---|---|
| 名称 | HKUDS/nanobot（PyPI 包名 `nanobot-ai`，当前 v0.4.0） |
| 星数 | 49301（快照）；gh api 实测 2026-11-04 约 49.8k |
| 语言 | Python（核心）+ TypeScript（WebUI 与终端 TUI） |
| 许可证 | MIT（`LICENSE`，pyproject 中 `license = {text = "MIT"}`） |
| 创建时间 | 2026-02-01（gh api `created_at`） |
| 最近 push | 2026-11-03（分析当日仍有合并，gh api `pushed_at`） |
| 最新 release | v0.4.0（2026-10-28） |
| 代码规模 | Python 核心包约 18.9 万行，测试约 16.6 万行；WebUI/TUI 共 649 个 TS/TSX 文件 |

## 2. 项目是什么，解决什么问题

nanobot 解决的问题是"个人 AI 助手"的部署门槛与可托管性：大多数 LLM 应用要么是一次性脚本，要么是重平台服务。nanobot 把一个常驻的 agent 网关（`nanobot gateway`）作为核心形态——它同时接通 CLI、WebUI（WebSocket）、Telegram、Discord、Slack、飞书、微信、Email、Mattermost、Linear 等 19 个渠道包（`nanobot/channels/` 目录），所有渠道的消息汇入同一条消息总线，由同一个 agent 内核处理（`README.md`、`docs/concepts.md`）。

能力上，模型可通过内置工具读写文件、跑 shell、正则搜索（ripgrep 封装）、联网搜索/抓取、生成图片、cron 定时任务、多 agent 委派（subagent），并通过 MCP 协议接入外部工具服务器。文档解析为内置能力：PDF/Word/Excel/PPT 读取依赖 pypdf、python-docx、openpyxl、python-pptx（`pyproject.toml` dependencies）。

记忆体系是其特色之一："Dream"机制让 agent 定期以独立会话审视并改写工作区中的 `SOUL.md`、`USER.md`、`memory/MEMORY.md` 三个文件，把用户偏好沉淀为长期记忆；改写基于 dulwich 纯 Python git 实现做 diff 审计（`nanobot/agent/memory.py` 中 `MemoryStore._DREAM_CONTENT_PATHS`、`dream_content_diff`）。会话历史存 SQLite（`nanobot/session/manager.py` 引用 `session/sqlite_store.py`），并有自动压缩（autocompact）与上下文预算治理。

安全设计面向"agent 拿着 shell 和网络权限长期运行"的场景：工具边界硬性拦截 SSRF（私网地址访问失败后不封死会话而是注入对话式恢复提示，`nanobot/agent/tools/execution.py` 顶部 `_SSRF_MARKERS`）、URL 目标校验 + DNS 固定（`nanobot/security/network.py`）、工作区路径范围约束（`nanobot/security/workspace_access.py`）、工具沙箱与 bwrap Docker 隔离选项（`nanobot/agent/tools/sandbox.py`、`docker-compose.bwrap.yml`）。

它同时暴露 Python SDK 与 OpenAI 兼容 API（`docs/python-sdk.md`、`docs/openai-api.md`、`nanobot/sdk/`、`nanobot/api/`），可被当作带工具的"个人模型端点"集成进其他系统。项目定位自我对标"小核心、可读源码"——README 的 Why nanobot 一节明确强调 "Small core: readable internals"。

## 3. 技术栈与依赖

- **核心语言/运行时**：Python 3.11+，asyncio 全异步；构建用 hatchling（含自定义构建钩子 `hatch_build.py`）。
- **关键第三方库**（`pyproject.toml`）：
  - LLM SDK：`anthropic`、`openai`、`boto3`（Bedrock extra）、`mcp>=1.26`（官方 MCP SDK）
  - 网络：`httpx[socks]`、`aiohttp`、`websockets`
  - 数据/校验：`pydantic v2` + `pydantic-settings`
  - CLI/交互：`typer`、`rich`、`prompt-toolkit`、`questionary`
  - 工具链支持：`tiktoken`（token 估算）、`json-repair`（修复模型 JSON）、`jinja2`（提示词模板）、`dulwich`（纯 Python git，Dream diff 用）、`rapidfuzz`（模糊匹配）、`croniter`（定时）
  - 文档解析：`pypdf`、`python-docx`、`openpyxl`、`python-pptx`
  - 可观测：`loguru`，可选 `langfuse` extra
- **前端**：WebUI 为 React + TypeScript + Vite + Radix UI，vitest + Playwright 测试（`webui/package.json`）；TUI 为 TypeScript 客户端，基于 OpenTUI 的全屏 retained 布局，Bun 构建（`tui/README.md`）。
- **部署**：Docker（`Dockerfile`、`docker-compose.yml`、rootless bwrap 变体）、Render 配置（`render.yaml`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `nanobot/agent/` | agent 内核：AgentLoop（渠道侧轮次编排）、AgentRunner（模型侧工具循环）、记忆/Dream、子代理、hooks、自动压缩（`loop.py` 2730 行、`runner.py` 1418 行） |
| `nanobot/agent/tools/` | 工具体系：Tool 基类/Schema、注册表、36 个工具文件（filesystem、shell、rg、web、mcp、cron、image_generation、subagent、sandbox 等） |
| `nanobot/providers/` | LLM 提供方抽象与 47 个 ProviderSpec 注册表；OpenAI 兼容通用实现 + Anthropic/Azure/Bedrock/Codex/Copilot/Grok 专用路径、fallback 链 |
| `nanobot/channels/` | 19 个自包含渠道包（telegram、discord、slack、feishu、weixin、wecom、dingtalk、qq、email、matrix、msteams、signal、whatsapp、linear、mattermost、napcat、mochat、websocket、whatsapp_agent），插件式发现（`manager.py`） |
| `nanobot/session/` | 会话存储（SQLite）、历史可见性、摘要检查点 |
| `nanobot/bus/` | 消息总线：InboundMessage/OutboundMessage 事件与队列 |
| `nanobot/gateway/`、`nanobot/cron/`、`nanobot/goal/`、`nanobot/triggers/` | 常驻网关、定时任务、长程目标、触发器 |
| `nanobot/security/` | SSRF 校验/DNS 固定、工作区访问策略 |
| `nanobot/skills/`、`nanobot/templates/` | 内置技能（weather、github、tmux、summarize 等 11 个）与工作区模板（AGENTS.md、SOUL.md、USER.md、HEARTBEAT.md） |
| `webui/`、`tui/` | 两个独立 TypeScript 前端（React WebUI；OpenTUI 终端客户端），产物打包进 wheel（`nanobot/web/dist/`） |
| `tests/` | 与 `nanobot/` 同构的顶层测试树 + 渠道内共置测试 |
| `docs/` | 31 篇 Markdown 文档 + guides/，架构文档直接映射到源码文件 |
| `scripts/`、`.github/workflows/` | 安装脚本（install.sh/ps1）、CI 范围裁剪脚本等；6 个 workflow |

组织方式评述：单体多包仓库（monorepo 式但单一 Python 包），Python 内核 + 两个 TS 前端并存；渠道与工具均为插件化自包含包，扩展点有明确指南（`docs/channel-package-guide.md`、`docs/my-tool.md`）。渠道测试共置在 `nanobot/channels/*/tests/` 并纳入 pytest 与覆盖率统计（`pyproject.toml` 的 `testpaths`/`omit` 配置）。

## 5. 核心架构与关键实现

### 5.1 双层循环：AgentLoop 与 AgentRunner

官方架构文档（`docs/architecture.md`）与源码一致地把运行时拆为两层：

- **AgentLoop**（`nanobot/agent/loop.py`，`class AgentLoop` @ 209 行起）：面向渠道的轮次编排——接收 InboundMessage、决定会话与工作区范围、构建上下文、挂接 hooks、发布 OutboundMessage；`from_config()`（464 行）为组合根，MCP 连接生命周期由应用持有而非 loop 持有。
- **AgentRunner**（`nanobot/agent/runner.py`，`async def run` @ 256、`_run_core` @ 356）：面向模型的循环——发请求、流式增量/推理块处理、执行工具调用并回灌结果，直到产出最终回答或触发运行时上限。

`_run_core` 主循环的骨架（`nanobot/agent/runner.py:356` 起）：

```python
iteration = -1
followup_requested = False
while (
    iteration + 1 < spec.max_iterations
    or followup_requested
    or await self._continuation_prompt(spec) is not None
):
    iteration += 1
    # 会话收件箱在每次模型调用前截取有限快照，吸收运行中到达的新消息
    drained_before_request = await self._try_drain_injections(
        spec, messages, phase="before model call",
    )
    ...
    response, raw_usage = await self._request_model(...)
    await self.context_governor.accept_request(...)
```

循环内含多类韧性机制：空响应重试（`_MAX_EMPTY_RETRIES=2`）、长度截断恢复（`length_recovery_parts` 分段续写）、工具迭代上限后的"终局注记"（由 `session/turn_continuation.py` 保守，模型历史回放稳定但展示隐藏）、以及 hook 体系（`before_run/before_iteration/after_run/on_error`，`nanobot/agent/hook.py`）。

### 5.2 工具体系：Tool ABC + ToolRegistry + 分批并发执行

工具基类是极简 ABC（`nanobot/agent/tools/base.py:212` `class Tool`）：`name/description/parameters` 三个抽象属性 + `execute(**kwargs)`；`read_only`/`concurrency_safe`/`exclusive` 三个布尔属性声明并发语义；`cast_params()` 按 JSON Schema 做"安全转型"（仅当声明类型要求 object/array 时解码嵌套 JSON 字符串，不做分支推断）。`@tool_parameters` 类装饰器可从声明生成 schema（`nanobot/agent/memory.py:1` 注释可见其用途）。

执行层（`nanobot/agent/tools/execution.py`）按并发语义分批：`_partition_tool_batches` 把一批 tool call 切成"并发安全的只读批"（`asyncio.gather` 并行）与"必须独占的写批"（串行），结果保持稳定顺序；同一批内对重复 `read_file` 做去重索引（`read_results()` @ cache）。SSRF 触发时不中止回合，而是追加一段不可绕过的对话式提示（`_SSRF_BOUNDARY_NOTE`），工作区越界同理（`_WORKSPACE_VIOLATION_MARKERS`）。

注册表（`nanobot/agent/tools/registry.py`）保证定义顺序稳定以利用提示缓存：内置工具排前缀、MCP 工具排序后追加（`get_definitions()` 注释明确说明 cache-friendly 意图）。

### 5.3 MCP 集成与安全边界

`nanobot/agent/tools/mcp.py`（1724 行）把 MCP 服务器封装为一等工具源：支持 stdio/HTTP 传输、OAuth（`mcp_oauth.py` 889 行）、断线重连（对"瞬态流失败 + 可重放操作"做重试）、以及 `nanobot/security/network.py` 提供的 URL 目标校验（`validate_url_target`，拒绝私网/回环地址）与 DNS 固定（`PinnedDNSAsyncTransport`，防止解析切换后的 DNS rebinding）。MCP 工具经同一个 `ToolRegistry` 与内置工具无差别参与调度。

### 5.4 记忆（Dream）与上下文治理

`nanobot/agent/memory.py`（1263 行）实现 MemoryStore/MemoryArchiver/Consolidator。Dream 的关键设计：记忆就是工作区里的普通 Markdown 文件，每次 Dream 运行后用 dulwich 对 `SOUL.md`、`USER.md`、`memory/MEMORY.md` 生成 diff（`dream_content_diff()`），供用户在 WebUI 审计"agent 改了什么记忆"。上下文治理（`nanobot/agent/context_governance.py` + `agent/autocompact.py`）在请求超预算时先压缩历史、再以预算错误替换未读的文件读取结果并保存该错误用于回放，保证"重放 = 展示"的一致性（`docs/architecture.md` Context Budgets 一节）。

### 5.5 提供方注册表

`nanobot/providers/registry.py` 以声明式 `ProviderSpec` 元组维护 47 个提供方条目（含关键词、API 前缀、baseURL 提示用于自动路由）；实现上绝大多数走 OpenAI 兼容路径（`openai_compat_provider.py`），Anthropic/Azure/Bedrock/OpenAI Codex/GitHub Copilot/xAI Grok 有专用实现，另有 `fallback_provider.py` 做多模型回退链（`docs/architecture.md` Providers 一节）。

## 6. 构建与运行

来自 README、`docs/quick-start.md` 与 `pyproject.toml`：

```bash
# 安装（稳定版，安装脚本）
curl -fsSL https://raw.githubusercontent.com/HKUDS/nanobot/main/scripts/install.sh | sh   # macOS/Linux
irm https://raw.githubusercontent.com/HKUDS/nanobot/main/scripts/install.ps1 | iex        # Windows

# pip / uv
pip install nanobot-ai            # 或 uv tool install nanobot-ai

# 首次配置与运行
nanobot onboard                   # 生成 ~/.nanobot/config.json 与 workspace
nanobot agent -m "Hello!"         # 单次终端对话
nanobot gateway                   # 常驻网关（WebUI http://127.0.0.1:8765，健康检查 :18790）

# Docker
docker compose up -d              # docker-compose.yml；bwrap 加固变体见 docker-compose.bwrap.yml

# 开发：前端
cd webui && bun install && bun run build     # 产物进 nanobot/web/dist/
bun install --cwd tui && bun run --cwd tui build

# 测试与静态检查
pip install -e ".[dev]"
pytest                            # asyncio_mode=auto，含 nanobot/channels 共置测试
ruff check . && basedpyright      # ruff(E,F,I,N,W) + basedpyright strict
```

## 7. 测试与工程质量

- **测试规模**：`tests/` 树 + 渠道共置测试合计约 8347 个测试函数（`grep -c "def test_|async def test_"` 统计），约 16.6 万行测试代码，测试代码量接近产品代码（18.9 万行），比例健康。
- **风格**：pytest + pytest-asyncio（auto 模式）+ xdist 并行 + coverage 门禁 `fail_under = 75`（`pyproject.toml`）。覆盖从纯单元到 Docker 冒烟（`tests/test_docker_smoke.py`）、安装脚本（`tests/test_install_scripts.py`）、发布候选校验（`tests/test_release_candidate.py`）、OpenAI 兼容 API 集成测试，金字塔形态完整。
- **CI**：`.github/workflows/ci.yml` 按变更路径裁剪执行面（`scripts/ci_scope.py` 计算 python/webui/tui/docker/native 五个维度是否需要跑），多 OS 矩阵（含 Windows，Windows 更新任务超时单独放宽到 30 分钟），另有 native-acceptance、release-candidate、release-approve、tui-release 等 6 个 workflow，发布流程带签名要求（`.github/release-signing-requirements.txt`，依赖 sigstore）。
- **静态质量**：basedpyright `strict` 模式全量检查 `nanobot/`、ruff line-length 100；类型注解密度高（抽查 `runner.py`、`execution.py` 均为完整泛型/ParamSpec 注解）。
- **观察**：核心大文件偏大（`agent/loop.py` 2730 行、`providers/base.py` 1770 行、`tools/mcp.py` 1724 行），但模块边界（loop/runner、工具/执行/注册表、提供方/注册表）切分清楚，`docs/architecture.md` 与源码一一对应，可读性确实如宣传所说较高。

## 8. 社区与维护状态

- **活跃度**：极高。gh api 显示最近 8 个提交全部在 2026-11-03（分析前一天到当天），PR 编号已达 #6552 量级；创建于 2026-02-01，约 9 个月达到 ~49.8k 星。
- **贡献者**：94 人（gh api contributors 分页计数）；近期提交集中在少数核心维护者（如 chengyongru）。
- **发布节奏**：v0.2.1（2026-06-01）→ v0.2.2（06-23）→ v0.3.0（07-25）→ v0.3.5（09-15）→ v0.4.0（10-28），约每 4-8 周一个 minor/patch；issue 积压 864 个 open issues（gh api），对一个增长期项目属常见现象。
- **生态与传播**：文档站 nanobot.wiki 提供 10 语言版本；Discord/X/WeChat/飞书多社区入口（`README.md`、`COMMUNICATION.md`）；有安装器、自更新协议（`docs/update-protocol.md`）、发布信任校验（`nanobot/release_trust.py`）等运营级基础设施，说明团队按产品化标准运营。

## 9. 亮点与风险

**亮点**

1. **架构分层教科书级清晰**：渠道侧 AgentLoop / 模型侧 AgentRunner / 声明式 ProviderSpec 注册表 / 自包含渠道包，四个扩展点都有配套文档（`docs/architecture.md`、`docs/channel-package-guide.md`）。
2. **工程质量罕见地高**：8.3k+ 测试、basedpyright strict、75% 覆盖率门禁、按路径裁剪的 CI、带 sigstore 签名的发布流程（`.github/workflows/ci.yml`、`pyproject.toml`）。
3. **安全设计认真**：SSRF 硬边界 + DNS 固定 + 工作区范围绑定 + 可选 bwrap 沙箱，且失败恢复策略考虑了对话连续性（`nanobot/security/`、`agent/tools/execution.py`）。
4. **上下文工程扎实**：autocompact、上下文预算错误替换、长度截断分段续写、终局注记等机制保证"重放即历史"的一致性（`agent/runner.py`、`agent/context_governance.py`）。
5. **Dream 记忆可审计**：长期记忆是普通 Markdown 文件 + git diff 审计，用户可见可改，避免了黑盒记忆库。

**风险/局限**

1. **仍在 0.x Alpha**：pyproject 分类器为 `Development Status :: 3 - Alpha`，工具名/错误文案被明确视为"模型契约"，跨版本兼容压力随功能膨胀而增大。
2. **核心文件持续变大**：`agent/loop.py` 2730 行、`runner.py` 1418 行，`agent/` 目录已 1102KB，"轻量"主要是部署轻量而非代码量轻量。
3. **依赖面广**：核心依赖约 40 个（含 4 个办公文档解析库），供应链与跨平台兼容（Windows/旧 Python）是长期负担，CI 中已有专门为 Windows 放宽超时的迹象。
4. **issue 积压**：864 open issues，社区增速远超维护人力（94 贡献者中提交集中在少数人），响应能力可能成为瓶颈。

## 10. 总结评价

nanobot 是 2026 年增长最快的开源个人 agent 框架之一（9 个月 0→49.8k 星），它用"一个可读的小内核 + 19 个渠道插件 + 47 个模型提供方"的组合证明了自托管个人 agent 可以同时做到功能完整与源码友好；其双层 agent 循环、上下文预算治理、SSRF 硬边界和 Dream 可审计记忆都是同类项目中的高水准实现。风险主要来自自身增长速度：0.x 阶段的契约稳定性、核心文件体量与 issue 积压。

适合谁用：想要一个常驻、多渠道、可自托管、记忆可审计的个人 AI 助手的开发者与进阶用户；需要 OpenAI 兼容 API + 工具调用的集成方。适合谁读源码：正在做 agent 框架的工程师——`nanobot/agent/loop.py`、`agent/runner.py`、`agent/tools/execution.py`、`agent/tools/mcp.py` 四个文件合起来是一份"生产级 agent 循环该长什么样"的参考实现。
