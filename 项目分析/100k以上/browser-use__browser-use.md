# browser-use/browser-use 项目分析

> 星数 118954（快照 2026-10-24）｜主语言 Python｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：最流行的开源浏览器 Agent 库——让 LLM 像人一样操控真实 Chrome 浏览器完成网页任务。

| 项 | 值 |
| --- | --- |
| 名称 | browser-use/browser-use |
| 版本 | 0.13.14（pyproject.toml，与最新 release 一致） |
| 星数 | ~119k（gh api 2026-10-30），forks ~13.3k，open issues 622 |
| 语言 | Python（>=3.11），纯代码项目，178 个 .py 文件约 6.9 万行 |
| 许可证 | MIT（LICENSE，Copyright 2024 Gregor Zunic） |
| 创建 / 最近 push | 2024-10-31 / 2026-10-27（gh api） |
| 最新 release | 0.13.14，2026-10-26 发布（gh api） |
| 主页 | https://browser-use.com，文档 docs.browser-use.com |

## 2. 项目是什么，解决什么问题

Browser Use 解决的核心问题是：LLM 天生不会"上网"。它把浏览器当前状态（可交互元素、页面结构、截图）序列化成 LLM 可读的文本，把 LLM 输出的决策（点击哪个元素、输入什么、滚动、导航）翻译成对真实浏览器的操作，形成"观察→决策→执行→反馈"的闭环，让用户只需用自然语言描述任务（如"找空位预约驾照考试"），agent 就能自主完成（README 顶部示例与 demo）。

它刻意压低了使用门槛：`Agent(task="...", llm=ChatOpenAI(...))` 十几行代码即可跑起来，系统提示词、浏览器会话、错误重试、上下文管理全部由库托管（README Path 3 快速上手）。同时保留深度定制口子：`Tools` 装饰器注册自定义动作、`extend_system_message`/`override_system_message` 定制提示词、结构化输出、历史保存与免模型重放（`save_history`/`load_and_rerun`/`detect_variables`，README FAQ）。

一个值得注意的事实：该项目早期基于 Playwright，但当前 main 分支已转向直接使用 Chrome DevTools Protocol（CDP）——依赖里有 `cdp-use==1.4.5` 而**没有** playwright（pyproject.toml），鼠标点击直接调 `Input.dispatchMouseEvent`（browser_use/actor/mouse.py:79）。虽然 GitHub topics 里仍挂着 playwright 标签，那只是历史包袱。

商业模式上，开源库是获客入口：README 明确给出三条路径——全托管 Cloud API（$0.02/浏览器小时的隐身浏览器 + CAPTCHA 处理）、CLI（给 Claude Code/Codex 等编码 agent 提供浏览器控制）、Python 本地库。开源代码里大量为云服务预留的接口（`Browser(use_cloud=True)`、browser/cloud/ 目录、browser-use-sdk 依赖）体现了这种 open-core 设计。

## 3. 技术栈与依赖

- 语言与运行时：Python >=3.11，异步优先（asyncio + anyio），类型标注覆盖完整并配 pyright basic 检查（pyproject.toml）。
- 浏览器控制：`cdp-use`（CDP WebSocket 客户端），替代 Playwright；浏览器进程发现/配置见 browser_use/browser/chrome.py。
- 数据模型：pydantic v2 全家桶（pydantic + pydantic-settings），几乎所有视图对象（AgentOutput、BrowserProfile、事件等）都是 BaseModel。
- 事件总线：`bubus`（同公司自研，BrowserSession 的 ResilientEventBus 基于它，browser_use/browser/session.py:113）。
- LLM 接入：直接依赖 openai、anthropic、google-genai、groq、ollama SDK，llm/ 目录另有 azure、aws bedrock、cerebras、deepseek、mistral、openrouter、vercel、litellm、oci_raw 等 15+ 适配器（browser_use/llm/ 目录）。
- 其他关键库：mcp（MCP 协议支持）、posthog（遥测）、rich/click/InquirerPy（CLI）、markdownify/pypdf/reportlab/python-docx（页面文本与文件产出）、pillow（截图处理）、cloudpickle、browser-use-sdk + browser-harness（云浏览器与 CLI 联动）。
- 工程链：hatchling 构建，uv 管理依赖，ruff（tab 缩进、130 列）+ pyright + codespell + pre-commit，pytest + pytest-asyncio + pytest-xdist。

## 4. 目录结构与代码组织

单包（非 monorepo），主包 `browser_use/` 按职责分 18 个子模块（约 6.9 万行）：

| 目录 | 行数 | 职责 |
| --- | --- | --- |
| browser/ | 19,269 | 浏览器会话、CDP 连接、14 个 watchdog、云浏览器、进程管理 |
| llm/ | 10,036 | 15+ 家 LLM 供应商的统一 ChatModel 适配层 |
| beta/ | 6,882 | beta 版 hosted agent 封装（service.py 明确标注 beta） |
| agent/ | 7,784 | agent 主循环、消息管理、提示词、judge、变量检测 |
| dom/ | 6,188 | CDP DOMSnapshot 抓取、裁剪、序列化成编号元素列表 |
| tools/ | 3,603 | 动作注册表（Registry）与内置浏览器动作 |
| integrations/ | 2,558 | 外部集成（MCP、toolsets-for-claude、langchain 等） |
| actor/ | 2,377 | 底层原子操作：鼠标/键盘/元素点击（CDP Input 域） |
| mcp/ | 2,328 | 作为 MCP server 暴露浏览器能力 |
| filesystem/、tokens/、sandbox/、skills/、sync/、telemetry/ | 各 0.3-1.1k | agent 文件系统、token 计费、沙箱、技能、同步封装、遥测 |

顶层另有 examples/（112 个示例脚本，覆盖 getting_started 到 use-cases）、tests/、docker/、skills/、static/。组织方式评述：按"层"分包（agent 编排层 → tools 动作层 → browser 会话层 → dom 观察层 → actor 执行层），层间依赖方向清晰；单文件普遍偏大（session.py 4197 行、agent/service.py 4188 行），属于"单类巨文件"风格。

## 5. 核心架构与关键实现

### 5.1 Agent 主循环（browser_use/agent/service.py，4188 行）

`Agent.run(max_steps=500)` 驱动 while 循环（service.py:2620），每步检查暂停/停止标志与连续失败次数（`consecutive_failures >= max_failures` 即终止，service.py:2630）。单步 `step()` 分四阶段（service.py:1036-1086）：

```python
async def step(self, step_info: AgentStepInfo | None = None) -> None:
    browser_state_summary = None
    try:
        if self.browser_session:
            captcha_wait = await self.browser_session.wait_if_captcha_solving()  # Phase 0
        # Phase 1: 准备上下文（浏览器状态 + 消息管理）
        browser_state_summary = await self._prepare_context(step_info)
        # Phase 2: LLM 决策 + 执行动作
        await self._get_next_action(browser_state_summary)
        await self._execute_actions()
        # Phase 3: 后处理（记忆、历史、遥测）
        await self._post_process()
    except Exception as e:
        await self._handle_step_error(e)   # 统一错误处理：重试/失败计数
    finally:
        await self._finalize(browser_state_summary)
```

上下文准备阶段（service.py:1088-1161）有多个"提示词工程"干预点：步数预算警告（70% 时注入 `BUDGET WARNING: You have used x/y steps`，service.py:1549-1566）、循环检测 nudge、探索 nudge、重规划 nudge、最后一步强制只留 done 工具（service.py:1573）。LLM 调用用 `asyncio.wait_for(self._get_model_output_with_retry(...), timeout=llm_timeout)` 包裹（service.py:1185），带重试与限流退避。

### 5.2 DOM 观察层（browser_use/dom/）

观察不靠截图 OCR 也不靠 HTML 字符串，而是走 CDP 的 `DOMSnapshot.captureSnapshot` 一次性拿到整页布局数据（browser_use/dom/enhanced_snapshot.py:1-30）。enhanced_snapshot.py 只保留 10 个"真正被代码用到"的计算样式（display/visibility/opacity/cursor/pointer-events 等，REQUIRED_COMPUTED_STYLES），既防 Chrome 在重型页面上崩溃又提速；同时识别 password/file 等敏感输入并在离开快照前脱敏（enhanced_snapshot.py `_is_sensitive_input`）。

序列化器（browser_use/dom/serializer/serializer.py，`DOMTreeSerializer`）做四件事：构建 SimplifiedNode 树 → 边界盒过滤（`_apply_bounding_box_filtering`，viewport_threshold 裁掉视口外节点，service.py 默认 1000）→ 按 paint order 与可交互性判定（ClickableElementDetector）→ 给每个可交互元素分配稳定递增的 `selector_index`，输出形如 `*[12]<button ...>` 的文本树。编号即动作接口：LLM 只需输出 `click(12)`，执行层按 backend_node_id 定位元素（`_reserve_backend_node_ids`）。跨域 iframe 支持（默认 max_iframes=100、深度 5）与视口外隐藏可交互元素提示也在这层处理（browser_use/dom/service.py:45-120）。

### 5.3 浏览器会话与 watchdog 架构（browser_use/browser/session.py，4197 行）

`BrowserSession` 是 pydantic 模型，采用"事件驱动 + 命令直调"双层架构（类 docstring，session.py:141-161）：高层用事件总线（bubus）广播 NavigateToUrlEvent、TabCreatedEvent、FileDownloadedEvent 等；底层直接发 CDP 命令。最有特色的机制是 14 个 **watchdog**（browser_use/browser/watchdogs/：captcha、crash、dom、downloads、popups、permissions、security、storage_state、har_recording、recording、screenshot、aboutblank、local_browser、default_action）。BaseWatchdog 按约定自动注册事件处理器——方法名即 `on_<事件类型名>`（browser_use/browser/watchdog_base.py:1-40），例如 DOMWatchdog.on_BrowserStateRequestEvent 负责组装浏览器状态（dom_watchdog.py:244）。崩溃恢复、下载拦截、CAPTCHA 等待都被解耦成独立组件。本地 Chrome 自动发现与 profile 查找在 browser_use/browser/chrome.py（find_chrome_executable、list_chrome_profiles）。

### 5.4 动作系统与 LLM 抽象（browser_use/tools/、browser_use/llm/）

`Tools` + `Registry`（tools/registry/service.py:291 的 `@action` 装饰器）把 Python 函数注册为带 pydantic 参数模型的结构化工具，运行时动态拼出 `ActionModelUnion`（RootModel，registry/service.py:581）作为 LLM 的 function-calling schema；内置动作集中在 tools/service.py（navigate/click/scroll/extract/save_as_pdf/done 等，且支持按 URL 过滤的页面专属动作，`get_prompt_description`）。LLM 侧 `BaseChatModel` Protocol 只有一个方法签名（llm/base.py:33-55）：`ainvoke(messages, output_format: type[T]) -> ChatInvokeCompletion[T]`——用泛型把"结构化输出"下沉到每家适配器，agent 主循环因此与供应商完全解耦。

## 6. 构建与运行

```bash
# 安装（README Path 3）
uv add browser-use                 # 或 pip install browser-use，Python >= 3.11

# 最小运行（README agent.py 示例）
uv run agent.py                    # Agent(task=..., llm=ChatOpenAI(...)).run()

# CLI（pyproject.toml [project.scripts]，四个等价入口）
browser-use          # 等价别名：bu / browseruse / browser
browser-use skill install   # 给编码 agent 注册技能（README Path 2）

# 开发（pyproject.toml dev-dependencies + Makefile 不存在，命令来自 CI）
uv sync --all-groups               # 安装全部 dev 依赖
uv run ruff check --fix .          # lint（.github/workflows/lint.yml）
uv run pyright                     # 类型检查
uv run pytest                      # 测试；单文件跑：pytest tests/ci/test_browser_session.py
uv run pytest -m "not slow"        # 排除慢速标记（pyproject [tool.pytest.ini_options]）

# Docker
docker build -t browser-use .      # 根目录 Dockerfile / Dockerfile.fast
```

## 7. 测试与工程质量

- 测试规模：tests/ 下 126 个 .py 文件，分 ci/（主体，覆盖 browser、interactions、models、security、infrastructure 等子目录）、agent_tasks（YAML 任务定义）、mind2web_data（基准数据）、scripts。既有纯单元测试也有大量真浏览器集成测试（pytest 标记 slow/integration/unit，pyproject.toml）。
- 测试基建：pytest-asyncio auto 模式、300 秒超时、pytest-xdist 按文件分组分发（`--dist=loadscope`）；.github/workflows/test.yaml 用脚本把每个测试文件生成独立 CI matrix job（避免慢测试互相拖累）。
- CI 配置：lint.yml（ruff）、test.yaml（按文件矩阵）、publish.yml（PyPI 发布）、docker.yml、eval-on-pr.yml（PR 上跑 benchmark）、stale-bot.yml、claude.yml（Claude 辅助开发）。
- 质量观察：类型标注与 docstring 覆盖率高，ruff 规则含 ASYNC 专项（异步超时治理）；但也有债务信号——DomService docstring 自注 TODO "currently we start a new websocket connection PER STEP"（dom/service.py:48-49），session.py/agent service.py 均超 4000 行。

## 8. 社区与维护状态

- 活跃度极高：最近 push 2026-10-27，最新 release 0.13.14 发布于 2026-10-26（gh api），与代码内版本号完全一致，说明"发版即主干"的紧凑节奏。
- 背景：2024-10-31 创建，由 Magnus Müller 与 Gregor Žunič（苏黎世）发起（README Citation 块），不到两年到 119k 星，是 AI agent 浪潮中增长最快的 Python 库之一。公司已商业化（Browser Use Cloud、BU2 专有模型，README/FAQ）。
- 贡献者结构：单人/小团队主导 + 公司化运营的形态（pyproject 作者仅列 Gregor Zunic；CLAUDE.md、claude.yml 说明重度使用 AI 辅助开发）；社区 PR 规模未确认（tarball 无 git 历史）。
- 生态位：README 列出 7 个姊妹仓库（browser-harness、browser-harness-js、browser-use-pi、sdk、video-use、macos-harness、benchmark），围绕同一浏览器基础设施形成产品矩阵。

## 9. 亮点与风险

亮点：
1. 观察表示设计出色：CDP DOMSnapshot + 计算样式白名单 + paint order + 边界盒过滤，把任意网页压缩成 LLM 可靠消费的编号元素树，这是其效果领先的关键（browser_use/dom/）。
2. 工程化程度远超同类：事件总线 + watchdog 解耦横切关注点（崩溃/下载/CAPTCHA/录制），步级预算警告、循环检测 nudge、消息压缩、token 计费等生产细节齐全（browser_use/browser/watchdogs/、agent/service.py）。
3. LLM 无关性干净：单一 `ainvoke(messages, output_format: T)` 泛型协议 + 15 家适配器，换模型零改动（browser_use/llm/base.py）。
4. 免模型重放：历史保存后 `load_and_rerun` + `detect_variables` 可用变量替换重放点击流，大幅降成本（README FAQ、examples/features/rerun_history.py）。
5. 测试与 CI 矩阵认真，112 个示例几乎覆盖所有功能点。

风险/局限：
1. 商业化引力明显：代码中大量云服务钩子（telemetry 用 posthog 上报、cloud 目录、browser-use-sdk 强依赖），纯离线用户需自行甄别遥测配置。
2. 核心文件巨型化（session.py/agent service.py 各约 4200 行），且 DomService 自认存在"每步新建 WebSocket"的性能债（dom/service.py TODO）。
3. LLM 驱动执行不可控：真实浏览器上点击、表单提交等动作有误操作风险（README 对重放误提交也有明确警告），生产使用需沙箱/监控配套。
4. 迭代极快、破坏性变更频繁：v0.x 阶段 API（如从 Playwright 迁到 CDP）历史上多次重构，升级需谨慎读 changelog。

## 10. 总结评价

Browser Use 是"LLM 操控浏览器"赛道的定义性项目：它把 browser agent 所需的全套难题——页面状态序列化、动作空间定义、上下文管理、错误恢复、多模型适配——做成了一个开箱即用且工程质量上乘的 Python 库，并借 open-core 模式成功商业化。代码读起来虽然单文件偏大，但分层清晰（dom 观察 → actor 执行 → browser 会话 → tools 动作 → agent 编排），尤其推荐精读 dom/serializer/ 与 browser/watchdogs/，前者是"如何把网页讲给 LLM 听"的最佳实践，后者是"如何用事件架构组织复杂浏览器自动化"的范本。适合需要构建网页自动化 agent 的工程师直接使用，也适合研究 agent 系统设计的开发者作参考实现；对纯离线/成本敏感场景，需评估其云服务倾向与 v0.x 的 API 稳定性。
