# xtekky/gpt4free 项目分析

> 星数 66925（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：gpt4free（包名 `g4f`）是一个把大量第三方 LLM/多模态接口（免费社区端点、需登录账号的服务、OpenAI 兼容网关等）统一封装成一个 OpenAI 风格客户端 + 本地推理 API 服务的 Python 聚合库。

| 项 | 值 |
|---|---|
| 名称 | GPT4Free (g4f)，PyPI 包名 `g4f`（setup.py） |
| 星数 | 66,925（任务快照 2026-10-24） |
| 语言 | Python（另含自研 Go 启动器 g4f-go） |
| 许可证 | GPL-3.0（LICENSE、README 徽章、gh api 元数据一致） |
| 创建时间 | 2023-03-29（gh api） |
| 最近活跃 | push 2026-10-31、release v8.8.9 发布于 2026-10-31（gh api），维护极活跃 |
| 规模 | g4f 包 245 个 .py 约 7.5 万行；etc 测试/示例约 1.6 万行（本地统计） |
| 作者/维护 | @xtekky 创建，@hlohaus 维护（README），最高贡献者 2520 次提交（gh api） |

## 2. 项目是什么，解决什么问题

g4f 的核心命题是"用一个统一接口访问尽可能多的模型服务"。它把各家服务的调用方式（HTTP API、OpenAI 兼容端点、浏览器自动化等）封装成一个个 provider 适配器，对上层暴露两套东西：一是 Python 客户端（`g4f.Client` / `g4f.AsyncClient`，API 形状刻意对齐 OpenAI SDK），二是本地起一个 OpenAI 兼容的 REST 服务（FastAPI 实现），让任何支持自定义 OpenAI base_url 的前端/工具都能直接把 g4f 当后端用（README "What's included"、`g4f/api/__init__.py`）。

项目已经远超早期"逆向网页版 ChatGPT"的单一形态，现在是一个多形态发行：PyPI 包（含 `g4f`/`g4f-mcp`/`g4f-tray` 三个入口，setup.py entry_points）、Docker 全量/精简镜像（docker-compose.yml）、本地 Web GUI（Flask 后端 `g4f/gui/server/`，前端 dist 主要从 g4f.dev 远程加载，`g4f/gui/server/website.py` 中有本地 DIST_DIR 与远程 STATIC_URL 的回退逻辑）、MCP 服务器（`g4f/mcp/`）、以及 Go 写的一键启动器 g4f-go（`g4f-go/README.md`：按平台下载已校验 sha256 的 CPython，再装 g4f[slim]，无需系统 Python）。另有浏览器扩展、Discord bot、WebRTC 远程桌面等附属项目（`projects/`）。

provider 的实际供给分三类：免认证的免费/社区端点（如 Pollinations）、需要用户自带凭证的官方服务（`Provider/needs_auth/` 下 35 个文件，含 Anthropic、Gemini、Groq 等）、以及动态注册的自定义网关（`client/factory.py` 支持传一个 base_url 就地生成 provider 类，还会拉取 g4f.dev 上的在线 providers.json 缓存到本地）。模型层（`g4f/models.py`）维护约 90 个模型定义与别名，把"gpt-4o"这类名字映射到一组可服务的 provider。

需要客观指出：该项目的定位决定了其合规灰度。`LEGAL_NOTICE.md` 明确声明与所列服务商无关联、仅供学习目的、服务商可联系下架；README 亦有 "Security, privacy & takedown policy" 章节。使用其中逆向/免认证端点通常违反对应服务商的服务条款，风险由使用者自担（详见第 9 章）。

## 3. 技术栈与依赖

- 主语言 Python 3.10+（README Requirements；CI 同时测 3.10 与 3.15，`.github/workflows/unittest.yml`）。
- HTTP 栈双轨：优先 `curl_cffi`（TLS 指纹伪装能力，`g4f/requests/__init__.py` 中按可用性回退到 `aiohttp`），另有 CDP 浏览器会话（`requests/cdp.py`、`cdp_browser.py`）供需要浏览器自动化的 provider 使用。
- 服务端：FastAPI + uvicorn + websockets（推理 API），Flask[async] + werkzeug（GUI 后端），`a2wsgi` 把 Flask 挂进 FastAPI（`api/__init__.py` 第 733 行附近）。
- 基础依赖：requests、aiohttp、pycryptodome、brotli、browser_cookie3（读浏览器 cookie）、ddgs（网页搜索）、beautifulsoup4、pillow/cairosvg（图像）、markitdown（文件解析）、PyYAML（config.yaml 路由）等（setup.py EXTRA_REQUIRE 按 all/slim/api/gui/image/search/local/files/tray 分组，支持按需安装）。
- 辅语言：Go 1.x（g4f-go，自带 go.mod 与独立测试）。
- 打包：setuptools（setup.py），PyPI 发布走 `.github/workflows/publish-to-pypi.yml`；版本号由环境变量 `G4F_VERSION` 注入 Docker 构建。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `g4f/` | 主包，约 245 个 .py / 7.5 万行 |
| `g4f/providers/`（小写） | 框架层：基类、路由、重试、类型、缓存、config 路由 |
| `g4f/Provider/`（大写） | 各服务适配器实现，161 个类 / 124 个适配器文件，11 个子目录 |
| `g4f/api/` | FastAPI 推理服务（`__init__.py` 单文件 3057 行） |
| `g4f/client/` | Python 客户端（Client/AsyncClient、Pydantic 模型、工厂） |
| `g4f/requests/` | 统一传输层（curl_cffi/aiohttp/CDP） |
| `g4f/gui/` | 本地 GUI 的 Flask 后端（前端 dist 不入库，远程加载） |
| `g4f/mcp/` | MCP 服务器（g4f-mcp 入口，含 sandbox、browser_dom 工具） |
| `g4f/tools/` | 媒体渲染、鉴权（AuthManager）、run_tools、web search 等横切能力 |
| `g4f/models.py` | 模型注册表（ModelRegistry + Model/ImageModel/AudioModel/VideoModel/VisionModel） |
| `g4f-go/` | Go 启动器（下载 CPython + 装 g4f，独立测试与 Makefile） |
| `etc/` | unittest（44 个测试文件）、testing（19 个集成脚本）、examples（15 个示例）、tool |
| `projects/` | browser-extension、discord-bot、remote-desktop、windows、android 等附属 |
| `docker/` | Dockerfile 全量/slim/armv7/sandbox-desktop（VNC 桌面） |

组织方式评述：单仓多形态 monorepo。核心库与附属项目同仓，框架层与适配器层用大小写目录名区分（`providers/` vs `Provider/`），这一点初见不直观但职责边界清晰：框架层不知道具体服务，适配器层只依赖框架层。api/`__init__.py` 3000+ 行、backend_api.py 1300 行属于明显的"大文件"风格，迭代速度快于重构速度。

## 5. 核心架构与关键实现

### 5.1 Provider 抽象（g4f/providers/base_provider.py，615 行）

类层次为 `AbstractProvider` → `AsyncProvider`（一次性返回）/ `AsyncGeneratorProvider`（流式生成，`supports_stream=True`），配三个功能 Mixin：`ProviderModelMixin`（模型列表与别名）、`RaiseErrorMixin`（上游错误归一化）、`AuthFileMixin`/`AsyncAuthedProvider`（凭证持久化）。设计上用"方法签名即能力声明"：`get_parameters()` 通过 `inspect.signature` 内省 `create_async_generator` 的参数，只放行 `SAFE_PARAMETERS` 白名单（temperature、media、tools、reasoning_effort 等 36 个，第 35-71 行），API 层据此自动生成各 provider 的可用参数说明，避免文档与实现漂移。同步/异步互操作由 `get_provider_method`/`get_async_provider_method`（第 138-175 行）做适配：异步生成器可被 `to_sync_generator` 拉平为同步迭代。

### 5.2 路由与容错（g4f/providers/provider_router.py、retry_provider.py、any_provider.py）

`ProviderRouter` 负责为模型请求挑选 provider：读取全部 provider 元数据 + 磁盘缓存的模型列表（`_models_cache`/`_lately_used` 持久化为 JSON，第 100-146 行），`match_model` 支持别名归一化匹配（大小写/连字符不敏感）。容错由 `retry_provider.py` 承担：`IterListProvider` 按顺序遍历候选 provider、`RetryProvider` 单点重试、`RotatedProvider` 轮换，且实现了 `ProviderCircuitBreaker` 熔断器（第 173 行）。`any_provider.py` 的 `DefaultProvider.create_async_generator` 是"自动模式"总入口（第 394 行起）：无模型名时按请求内容分流——检测到音频 media 走 PollinationsAudio/OpenAIFM，有图片走默认视觉 provider 组，否则用默认文本 provider 列表并叠加已配置 key 的 provider 参与 `rank()` 排序；`agent:` 前缀模型路由到 `AgentTools`，config.yaml 中定义的规则路由到 `ConfigModelProvider`。

### 5.3 config.yaml 动态路由（g4f/providers/config_provider.py）

内置了一个小型条件表达式引擎：`_tokenize`/`_parse_or/_parse_and/_parse_not/_parse_comparison` 手写递归下降解析器（第 200-301 行），支持 `quota < 10 and not needs_auth` 这类布尔条件，配合 `QuotaCache` 与 `ErrorCounter`（带错误计数熔断）实现"按各 provider 实时配额/健康度动态选路"。这让运维者不改代码就能重排 provider 优先级（docs/config-yaml-routing.md 有专文）。

### 5.4 OpenAI 兼容 API（g4f/api/__init__.py，3057 行）

`create_app()` 构建单 FastAPI 应用，路由对齐三家协议：`/v1/chat/completions`、`/v1/responses`（OpenAI Responses API）、`/v1/messages`（Anthropic Messages 格式）、`/v1/images/generations`，且每条都有 `/api/{provider:path}/...` 变体可强制指定 provider（第 1133-1138、1351-1353、1536-1538 行）。另有 `/api/{provider}/quota`、`/api/{provider}/health`（仅 GET 探活不上游对话，`base_provider.py` 第 361-377 行的 `get_health` 注释明确此约束）、`/screenshot` 截图服务等运维端点。请求管道含日志中间件、鉴权中间件（`authorization`，第 862 行）与流式 tee。

### 5.5 客户端工厂与动态 provider（g4f/client/factory.py，172 行）

`ClientFactory.create_provider` 把"字符串 provider 名"解析成类：内置名走 `ProviderUtils.convert`；URL 直接 `create_custom_provider` 用 `type(name, (OpenaiTemplate,), attrs)` 动态造类（第 63 行）；`custom:` 前缀映射到 g4f.space 托管网关；未知名字从 g4f.dev 的 providers.json 在线清单加载并按日期缓存到本地（第 107-121 行）。这是项目保持"新服务上线不用等发版"的关键机制，但也意味着运行时会引入远程配置（见风险）。

## 6. 构建与运行

来自 README、setup.py 与 CI（未实际运行验证）：

```bash
# pip 安装
pip install g4f                      # 最小安装（requests/aiohttp/pycryptodome/brotli）
pip install -U g4f[all]              # 全量

# 源码运行
pip install -r requirements.txt
g4f gui                              # 本地 Web GUI
python -m g4f api --port 8080        # 或 g4f api，起 OpenAI 兼容服务
g4f client "What is gpt4free?"       # CLI 单轮对话

# Docker（README 推荐）
docker compose up                    # hlohaus789/g4f:latest，含 GUI+API
# slim 镜像支持 x86_64/arm64（docker/Dockerfile-slim）

# 测试（CI 同款，.github/workflows/unittest.yml）
pip install -r requirements-min.txt
python -m etc.unittest               # 标准库 unittest 聚合入口，15 分钟超时

# Python 客户端用法（README 示例）
from g4f.client import Client
client = Client()
client.chat.completions.create(model="gpt-4o", messages=[{"role":"user","content":"Hello"}])
```

## 7. 测试与工程质量

- 测试分两层：`etc/unittest/` 44 个文件（标准库 unittest，`__main__.py` 手工聚合 import， CI 在 Python 3.10 最小依赖与 3.15 全量依赖双环境各跑一遍），`etc/testing/` 19 个需真实网络的集成脚本（test_api.py、test_providers.py、test_gui.py 等）。合计约 1.6 万行（含 examples/tool）。
- 覆盖风格：以 API 路由、路由决策（test_provider_routing.py、test_api_route.py）、重试/熔断（test_auth_retry.py、retry_provider.py）、标准化（test_reasoning_standardization.py）为主；部分真实 provider 测试被注释停用（`__main__.py` 中 test_qwen_stream、test_cdp_parallel 等），反映上游易变导致集成测试难稳定的现实。
- CI 完整度尚可：unittest.yml（PR + main）、codeql.yml（代码扫描）、publish-to-pypi.yml、build-packages.yml（打包）、close-inactive-issues.yml（过期 issue 自动关闭）；`copilot.yml.disabeld`（文件名拼写如此）为停用的 Copilot 工作流。
- 代码质量观察：mixin 组合与类属性配置风格一致；存在明显拼写遗留（`CopilotLeagcy.py`、"disabeld"、"interference API"），setup.py 自报 `Development Status :: 2 - Pre-Alpha`，与 v8.x 的发布节奏形成反差——工程上是"快速迭代、以可用性优先"的社区风格，而非企业级规范。

## 8. 社区与维护状态

- 活跃度极高：gh api 显示 push 时间为 2026-10-31（本次分析当天）；release 密集，v8.8.5 至 v8.8.9 五个版本集中在 2026-10-22 至 10-31 九天内，基本两三天一版。
- 贡献者结构：最高产贡献者 2520 次提交（gh api contributors 接口）；README 明示 @xtekky 创建、@hlohaus 接棒维护，赞助入口指向 hlohaus 的 Sponsors。
- 社区渠道：Telegram 频道 + 两个 Discord（News/Support）（README Quick links）；issue 有模板（`.github/ISSUE_TEMPLATE/`）与自动关闭机器人。
- 生态位：官网/文档 g4f.dev，Docker 镜像 hlohaus789/g4f，npm 侧有 gpt4free.js 浏览器客户端（README）；open_issues 仅 9（gh api），结合 close-inactive-issues 机器人说明 issue 清理非常激进。
- Star 走势未做历史核查（未确认），但 66.9k 星 + 13.5k fork 的量级与 2023 年以来持续热度相符。

## 9. 亮点与风险

亮点：
1. 适配器抽象完成度高：签名即能力、参数白名单内省、模板继承（OpenaiTemplate 516 行撑起大批 OpenAI 兼容 provider），新增一个 provider 的边际成本很低（`Provider/template/OpenaiTemplate.py`、`providers/base_provider.py`）。
2. 路由与容错是认真的：候选列表 + 熔断器 + 配额缓存 + YAML 规则路由 + 手写条件表达式引擎，形成一套面向"上游随时会挂"现实的弹性层（`providers/retry_provider.py`、`config_provider.py`）。
3. 多形态分发降低使用门槛：pip/Docker/Windows exe/Go 启动器/MCP/浏览器扩展全都有（setup.py、docker/、g4f-go/、projects/）。
4. 协议兼容面广：同时讲 OpenAI Chat Completions、OpenAI Responses、Anthropic Messages 三种方言，任何现成前端可直接接（`api/__init__.py`）。
5. 测试与 CI 不敷衍：双 Python 版本矩阵、CodeQL、最小依赖可跑（etc/unittest、workflows）。

风险/局限：
1. 合规风险（如实记录）：大量 provider 属于对第三方服务的非官方访问或免费端点聚合，通常违反相应服务商 ToS；`LEGAL_NOTICE.md` 自认"仅供教育目的、与服务商无关联"并留下架通道，使用者需自行承担法律与服务条款风险。本分析不提供任何绕过或滥用指导。
2. 稳定性天然脆弱：免费端点随上游变动频繁失效，项目用重试/熔断缓解但无法根治；部分真实 provider 集成测试被注释停用即是佐证（etc/unittest/__main__.py）。
3. 供应链面较大：运行时从 g4f.dev 拉取 providers.json 动态注册 provider、前端 dist 远程加载、g4f-go 首跑下载 CPython——便利与信任链都依赖 g4f.dev 基础设施（client/factory.py、gui/server/website.py、g4f-go/README.md）。
4. 代码组织债：api/__init__.py 3000+ 行、拼写遗留（CopilotLeagcy 等）、Pre-Alpha 状态声明，读源码需适应"快糙猛"风格。

## 10. 总结评价

gpt4free 是"免费/聚合 LLM 接入"这个赛道的旗帜性项目：66.9k 星、连续三年高强度维护、从单一逆向脚本长成了一套覆盖客户端库、推理网关、GUI、MCP、桌面与移动分发在内的完整产品族。它的架构核心——provider 抽象 + 多级路由容错 + 动态 provider 注册——本身就是一份值得读的"如何面对不可靠上游做工程"的教材；对想研究 API 网关设计、适配器模式在脏环境下的实战形态的开发者，源码价值很高。但它原生带着合规灰度：免认证端点普遍触及服务商 ToS，生产使用应只限于用户自带 key 的官方 API 转接部分（needs_auth 与自定义 base_url 路径），并对远程动态配置保持警惕。适合谁：研究多 provider 聚合架构的工程师、想自建私有 OpenAI 兼容网关（接自己的 key）的用户；不适合：寻求"开箱即用长期稳定免费 GPT"的非技术用户——那正是它风险与维护成本最集中的区域。
