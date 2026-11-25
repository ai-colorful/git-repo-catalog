# abi/screenshot-to-code 项目分析

> 星数 80659（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：用多模态 LLM 把截图、设计稿、Figma 乃至屏幕录像直接转换成可运行的单文件前端代码（HTML/Tailwind、React、Vue、Bootstrap、Ionic），并通过"多变体并行生成 + 实时预览 + 选中元素局部改"形成完整的视觉→代码工作台。

| 项 | 值 |
|---|---|
| 名称 | abi/screenshot-to-code |
| 星数 | 80,659（快照）/ 80,917（分析时 gh api 实测） |
| 主语言 | Python（后端）+ TypeScript/React（前端） |
| 许可证 | MIT（LICENSE，Copyright 2023 Abi Raja） |
| 创建 / 最近 push | 2023-11-14 / 2026-10-27（gh api） |
| 贡献者 / Release | 25 人 / 无正式 release（gh api，releases 数为 0） |
| 官网 | https://screenshottocode.com（README 第 3 行，托管版即商业产品） |

## 2. 项目是什么，解决什么问题

手写一个网页还原设计稿既慢又琐碎。screenshot-to-code 把这件事变成"喂一张图"：用户上传截图/粘贴 URL/输入文字描述，甚至录制一段屏幕操作视频，后端调用多模态大模型生成一个单文件 HTML 页面（内联 CSS/JS），前端用 iframe 实时渲染，用户可对比原图与成品、在多个模型变体之间切换，再通过对话式修改或直接点选页面元素做局部调整（README 开头及 Examples 节）。

它本质上是一个"AI 前端生成工作台"而非一次性脚本。项目从 2023 年的"单次 LLM 调用生成 HTML"演化为如今的 agent 化架构：模型拥有 create_file/edit_file、图片生成/抠图/编辑、资产提取、截图自检等一整套工具，在循环中多轮调用直到产出并验证页面（`backend/agent/engine.py`、`backend/agent/tools/runtime.py`）。

多模型"赛马"是核心产品理念：一次生成默认并行跑 4 个变体（`NUM_VARIANTS = 4`，`backend/config.py`），按用户可用的 API key 自动挑选模型组合（OpenAI GPT-5.x/6.1、Anthropic Claude Opus 4.8/5/5.5、Gemini 3.x，见 `backend/llm.py` 与 `backend/routes/model_choice_sets.py`），用户对比后择优继续迭代。

代码产出的目标是"能直接在浏览器里跑的单 HTML 文件"：React/Vue 等栈通过 CDN 脚本（react UMD + Babel standalone、Vue global build）在页面内编译运行，Tailwind/Bootstrap/Ionic 走 CDN，见系统提示词中的各栈注入指令（`backend/prompts/system_prompt.py`）。

支持 6 种输出栈：HTML+Tailwind、HTML+CSS、React+Tailwind、Vue+Tailwind、Bootstrap、Ionic+Tailwind（README "Supported stacks" 节）。

## 3. 技术栈与依赖

后端（`backend/pyproject.toml`，Poetry 管理，Python ^3.10）：
- FastAPI + uvicorn + websockets：HTTP 与 WebSocket 服务
- openai / anthropic / google-genai：三家模型官方 SDK
- playwright：无头 Chromium，用于 agent 的 screenshot_preview 自检工具
- pillow + pillow-heif：图像解码/裁剪（资产提取预处理，`backend/asset_extraction.py`）
- moviepy：视频模式帧处理；langfuse：LLM 观测；pydantic、httpx、beautifulsoup4

前端（`frontend/package.json`，pnpm + Vite 6 + React 18 + TS）：
- zustand：状态管理（`frontend/src/store/app-store.ts`、`project-store.ts`）
- CodeMirror 6：代码面板；radix-ui + tailwindcss：UI 组件体系（shadcn 风格，`frontend/components.json`）
- react-dropzone（上传）、webm-duration-fix + html2canvas（录屏与导图）、puppeteer 仅用于前端 E2E（`src/tests/qa.test.ts`）
- 测试：后端 pytest + pytest-asyncio + pyright；前端 jest（devDependencies 中另有 vitest 但脚本走 jest）

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `backend/agent/` | Agent 引擎、Provider 会话抽象、工具运行时与定义 |
| `backend/routes/` | FastAPI 路由：generate_code(WS)、evals、export、design_systems 等 10 个模块 |
| `backend/prompts/` | 提示词构建管线（create/update 两族、栈指令、设计系统） |
| `backend/preview_screenshot/` | Playwright 截图后端与可用性探测 |
| `backend/asset_extraction.py` | Gemini 驱动的截图资产提取（定位+裁剪原图元素） |
| `backend/image_generation/` | Replicate 图像生成/编辑/去背 |
| `backend/evals/` + `run_evals.py` | 评测框架（批量跑模型、对比、打分） |
| `backend/fs_logging/` | 每次 agent 运行的 JSON 落盘记录（回放/报告） |
| `backend/tests/` | 40 个测试文件、266 个测试函数 |
| `frontend/src/components/` | 按功能域分组：preview、variants、select-and-edit、evals、recording 等 |
| `frontend/src/store/` | zustand：app-store（全局 UI）+ project-store（commits/variants 历史） |
| `design-docs/`、`blog/` | 7 篇内部设计文档与 1 篇模型评测博客 |
| `AGENTS.md`（=CLAUDE.md 符号链接） | 面向 AI 编码代理的贡献规范 |

组织方式：典型的双包结构（根 `package.json` 以 pnpm workspaces 聚合 frontend/backend，`docker-compose.yml` 编排两个容器）。后端按"管线-引擎-工具-提供商"分层，前端按功能域切分组件，均为扁平单仓，无子包发布。

## 5. 核心架构与关键实现

### 5.1 WebSocket 中间件管线（视觉→代码的请求主链路）

前端唯一的关键通道是 `/generate-code` WebSocket。后端用中间件链组织一次生成请求（`backend/routes/generate_code.py` 第 887-901 行）：

```python
@router.websocket("/generate-code")
async def stream_code(websocket: WebSocket):
    pipeline = Pipeline()
    pipeline.use(WebSocketSetupMiddleware())
    pipeline.use(ParameterExtractionMiddleware())
    pipeline.use(StatusBroadcastMiddleware())
    pipeline.use(PromptCreationMiddleware())
    pipeline.use(CodeGenerationMiddleware())
    pipeline.use(PostProcessingMiddleware())
    await pipeline.execute(websocket)
```

每个中间件向 `PipelineContext`（第 86-108 行）填充状态：参数校验 → 按 key 组合挑选变体模型（`ModelSelectionStage._get_variant_models`，第 451-499 行）→ 组装提示词 → 并行生成。消息协议为 11 种类型（`chunk/setCode/status/thinking/assistant/toolStart/toolResult/variantComplete/variantError/variantCount/variantModels`，第 42-55 行），前端在 `frontend/src/generateCode.ts` 中一一映射为回调。

### 5.2 AgentEngine：工具循环 + 流式代码回传

每个变体由一个 `AgentEngine` 驱动（`backend/agent/engine.py`）。`_run_with_session`（第 218-327 行）最多 30 轮工具调用，每轮通过 `session.stream_turn` 拿到流式事件（assistant/thinking/tool_call_delta），并对 `create_file` 工具做特殊的"流式预览"——把工具参数里的 HTML 增量按 ≥40 字符节流推给前端 `setCode`（`_handle_streamed_tool_delta`，第 208-216 行），让用户在模型写代码时就能看到页面逐步成形。

两层防护值得注意：单变体花费上限 `GENERATION_MAX_COST_USD = 3.0` 美元，超支抛 `BudgetExceededError`（第 270-276 行，`backend/config.py`）；空产出抛 `EmptyOutputError` 防止 eval 假绿（第 25-36 行注释详细记录了动机）。

工具运行时 `AgentToolRuntime`（`backend/agent/tools/runtime.py`）实现 9 个工具：`create_file`、`edit_file`（精确字符串替换 + 统一 diff 回传，第 134-257 行）、`generate_images`/`remove_backgrounds`/`edit_images`（Replicate，批量 20 个并发）、`extract_assets`（Gemini）、`screenshot_preview`（Playwright）、`save_assets`、`retrieve_option`（跨变体取代码）。工具是否注册按能力降级：没有 Gemini key 就不提供 extract_assets，没有 Chromium 就不提供 screenshot_preview（`backend/agent/providers/factory.py` 第 31-39 行）。

### 5.3 screenshot_preview：让模型"看见"自己的产出

这是本项目 agent 化的关键差异化设计。`backend/agent/tools/screenshot_preview.py` 用无头 Chromium 把当前 HTML 渲染成桌面+移动端两张全页截图，以多模态字节回传给模型自检（第 32-63 行）：

```python
for viewport in PREVIEW_VIEWPORTS:  # ("desktop", "mobile")
    image_bytes = await capture_preview_screenshot(
        file_state.content, device=viewport, full_page=True)
    ...
    multimodal_parts.append(ToolMultimodalPart(
        display_name=display_name, mime_type="image/png", data=image_bytes))
```

系统提示词要求模型在 create_file/edit_file 之后总是调用它，发现布局问题就用 edit_file 修复（`backend/prompts/system_prompt.py` 第 19 行）。启动时通过 `probe_screenshot_preview` 预热探测（`backend/main.py` 第 34-40 行），Chromium 缺失则静默跳过该工具。

### 5.4 前端实时预览与"点选改"（select-and-edit）

预览即 iframe：`frontend/src/components/preview/PreviewComponent.tsx` 把生成代码写入 `iframe.srcdoc`，经 `useThrottle(code, 200)` 节流（第 37 行、285-293 行），桌面视口固定 1366px 按容器缩放、移动端 375px（第 177-220 行）。另有 `normalizeBabelCdn` 把生成代码里过期的 Babel CDN URL 规范化为可用版本，避免旧产物无法编译（第 289 行，配套逻辑在 `backend/babel_cdn.py` 与 `frontend/src/lib/babelCdn.ts`）。

点选改的实现：进入选择模式后，在 iframe 的 window 上以捕获阶段拦截 click/mousedown/submit 等事件，保证生成页面自身的 React/Bootstrap 处理器不会先触发（`PreviewComponent.tsx` 第 222-249 行注释明确说明利用捕获阶段先于 document 级委托处理器）；选中元素高亮后把 outerHTML 作为定位线索发给后端，系统提示词指导模型在源码中定位对应片段只改该元素（`backend/prompts/system_prompt.py` 第 91-94 行）。

前端状态用 git 隐喻建模：`frontend/src/store/project-store.ts` 维护 `commits`（每次生成/修改为一个 commit）、`head`、每个 commit 下的 variants 及其 agent 事件流，历史版本可回溯切换。

## 6. 构建与运行

来自 README "Getting Started" 与根 `package.json`：

```bash
# 后端（Poetry）
cd backend && echo "OPENAI_API_KEY=sk-..." > .env
poetry install
poetry run playwright install chromium   # 启用 screenshot_preview
poetry run uvicorn main:app --reload --port 7001

# 前端
cd frontend && pnpm install && pnpm dev  # http://localhost:5173

# Docker 一键起（根目录需 .env）
docker-compose up -d --build

# 测试（根 package.json 聚合）
pnpm test        # = frontend jest + backend pytest
pnpm -C frontend run test:qa  # E2E（RUN_E2E=true，puppeteer）
```

前端另需在 `frontend/.env.local` 配置 `VITE_WS_BACKEND_URL`/`VITE_HTTP_BACKEND_URL` 指向后端（README FAQ 节）。运行时至少要 OpenAI/Anthropic/Gemini 三者之一的 key，Gemini+Replicate 为推荐组合（README key 表格）。

## 7. 测试与工程质量

- 后端：40 个测试文件、266 个测试函数（`backend/tests/` 实测统计），覆盖 agent 引擎、各 Provider 会话、工具运行时、提示词构建、eval 框架等，`pytest.ini` + `pytest-asyncio`；类型检查用 pyright（`backend/pyrightconfig.json`，AGENTS.md 要求"改动文件零新增告警"）。
- 前端：9 个 jest 单测文件，集中在纯逻辑模块（overlays、activity-order、models、prompt-history 等）；E2E 用 puppeteer（`frontend/src/tests/qa.test.ts`，需显式 `RUN_E2E=true`）。
- 无 CI：`.github/` 下只有 FUNDING.yml 和 issue 模板，没有 workflows 目录，测试/类型检查依赖本地与贡献者自觉（对 8 万星项目而言是明显缺口）。
- 观测性做得比同类好：每次 agent 运行落盘 JSON（`backend/fs_logging/agent_runs.py`），配套 `/evals` 前端页面可回放时间线、对比 prompt 报告与 Best-of-N 评分（`frontend/src/components/evals/`）。`AGENTS.md`、`TESTING.md`、`QA.md`、`Evaluation.md` 及 7 篇 `design-docs/` 显示工程过程高度文档化。

## 8. 社区与维护状态

- 活跃度：gh api 显示最近 push 为 2026-10-27（分析日前 4 天），仓库持续高频更新；tarball 内文件 mtime 均为 2026-10-10，与之一致。
- 贡献者：25 人（gh api contributors），实际提交高度集中于作者 Abi Raja 一人（"hosted 分支对接独立 SaaS 代码库"的表述见 `AGENTS.md`，商业化单人主导特征明显）。
- 发布：0 个 GitHub release，无版本化发布流程；README/文档面向"拉 main 自建 + 官网托管版"两种使用方式。
- 社区面板：149 个 open issues（gh api）。README 用大量 issue 锚点回复 FAQ，作者维护参与度高（未能进一步统计回复率，未确认）。

## 9. 亮点与风险

亮点：
1. 完整的 agent 化生成管线：工具循环 + 流式 tool_call_delta 回传 + 预算熔断 + 空产出防护，是"LLM 生成 UI"领域工程质量很高的参考实现（`backend/agent/engine.py`）。
2. screenshot_preview 让模型用真实渲染截图自我校验，形成"生成→目检→修复"闭环，直接针对 LLM 视觉代码生成最常见的布局走样问题（`backend/agent/tools/screenshot_preview.py`、`backend/prompts/system_prompt.py`）。
3. 前端体验细节扎实：200ms 节流 iframe 预览、捕获阶段事件拦截实现可用的点选改、Babel CDN 兼容修复、commit/variant 历史模型（`PreviewComponent.tsx`、`project-store.ts`）。
4. 内建 evals 框架与运行记录回放 UI，模型/提示词迭代有数据支撑（`backend/evals/`、`frontend/src/components/evals/`）。
5. 多 Provider 抽象干净：统一 ProviderSession + 规范化工具定义，按可用 key 自动降级能力（`backend/agent/providers/`）。

风险/局限：
1. 无 CI、无 release，质量完全依赖作者本地 pyright+pytest，协作者门槛高（第 7 章证据）。
2. 生成目标被刻意限定为"CDN 依赖的单文件 HTML"，适合原型/营销页，不适合直接投产的工程化前端（`backend/prompts/system_prompt.py` 各栈指令）。
3. 强依赖付费 API（模型 key + Replicate），自托管成本高；README 也承认 Ollama 本地模型效果差。
4. 后端 CORS 全开、API key 可从前端设置对话框传入后端转发（`backend/main.py` 第 43-49 行、`generate_code.py` 第 302-315 行），公网自托管需自行加固。

## 10. 总结评价

screenshot-to-code 是"截图转代码"赛道的开创者与标杆：它把一次性的多模态 LLM 调用打磨成了带工具循环、视觉自检、多变体赛马和版本历史的完整工作台，约 2.2 万行 Python 后端 + React 前端的代码组织清晰、文档充分，单人支撑 8 万星项目并同步运营商业托管版。想研究"视觉→代码"管线设计、多模型并行变体、或 agent 工具循环工程化的开发者，这个仓库是必读样本；想直接拿它产出可维护的生产代码或为其贡献代码的人，则要注意单文件 CDN 产出的定位限制与无 CI 的协作现状。
