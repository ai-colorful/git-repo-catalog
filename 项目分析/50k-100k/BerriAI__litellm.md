# BerriAI/litellm 项目分析

> 星数 61899（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：LiteLLM 是统一 100+ LLM 提供商的 OpenAI 格式接口，双形态交付——Python SDK 与可自托管的 AI Gateway（代理服务器），README 自述 "Open Source AI Gateway for 100+ LLMs"（`README.md`）。

| 项 | 值 |
|---|---|
| 名称 | BerriAI/litellm |
| 星数 | 61899（快照），查询时已 62557（gh api，2026-11-02） |
| 主语言 | Python（另含 Rust 核心子项目） |
| 许可证 | MIT + enterprise/ 目录专有授权（`LICENSE` 首部声明） |
| 版本 | HEAD 为 v1.109.0（`pyproject.toml`），最新 release v1.106.6（2026-10-28） |
| 最近 push | 2026-11-02（gh api，分析前一天） |
| Forks / Open Issues | 12928 / 5610（gh api） |
| 背景 | Y Combinator W23（`README.md` 徽章） |

## 2. 项目是什么，解决什么问题

调用多家 LLM 时，每家 SDK、鉴权方式、请求格式、错误类型各不相同。LiteLLM 用一层统一抽象解决：所有提供商的请求/响应都转换到 OpenAI 格式，应用代码只写一份，通过改模型字符串（如 `anthropic/claude-...`、`bedrock/...`）即可切换供应商（`README.md`、`litellm/llms/` 下 164 个提供商目录）。

项目分两层交付（`ARCHITECTURE.md`）：

- **SDK（`litellm/`）**：Python 库，`litellm.completion()/acompletion()` 直连各提供商，内置路由、重试、fallback、成本计算，可嵌入应用进程内使用。
- **AI Gateway（`litellm/proxy/`）**：FastAPI 代理服务，包装 SDK，叠加 API 虚拟密钥、按团队/用户的预算与限流、guardrails、负载均衡、花费追踪与管理 UI，作为团队/组织的集中式 LLM 网关部署。

关键设计是**代理完全复用 SDK**：`ARCHITECTURE.md` 明确 "The LiteLLM AI Gateway (Proxy) uses the LiteLLM SDK internally for all LLM calls"，客户端（OpenAI SDK / Anthropic SDK / 任意 HTTP）打到 proxy，proxy 经鉴权后调用 `litellm.acompletion()`，由 SDK 完成提供商转换。

商业模型是开源核心 + 企业版：`enterprise/` 目录与 `enterprise/LICENSE` 单独授权（密钥管理高级功能、SSO 等），主仓库仍 MIT（`LICENSE`）。

## 3. 技术栈与依赖

- **语言**：Python >=3.10,<3.16（`pyproject.toml`）；新增 Rust 核心 `litellm-rust/`（edition 2024, rust-version 1.88，`litellm-rust/Cargo.toml`）。
- **Web 框架**：FastAPI + uvicorn/gunicorn/granian（proxy extra，`pyproject.toml` `[project.optional-dependencies].proxy`）。
- **HTTP/模型**：httpx[http2]、openai>=2.20（直接复用 OpenAI 官方 SDK）、aiohttp、pydantic v2、tiktoken + tokenizers（token 计数）、boto3（Bedrock/AWS）。
- **存储**：PostgreSQL（Prisma，`schema.prisma`）+ Redis 双缓存（DualCache）；ClickHouse 支出存储在 Rust crate 中（`litellm-rust/crates/spend-clickhouse`）。
- **可观测**：OpenTelemetry 三件套锁版本（`pyproject.toml` 注释）、Prometheus（`prometheus.yml`）。
- **工程链**：uv（`uv.lock` 冻结安装）、Ruff（`ruff.toml`/`ruff-strict.toml`）、basedpyright（`pyrightconfig.json`）、Makefile 驱动 lint/test。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `litellm/` | SDK 核心，3020 个 .py、约 103.5 万行（实测统计） |
| `litellm/llms/` | 164 个提供商实现目录、1122 个 .py；`base_llm/` 为能力域基类 |
| `litellm/proxy/` | 网关服务，840 个 .py；`proxy_server.py` 为 FastAPI 应用 |
| `litellm/router*.py` | 路由器（host/core 分离，见第 5 节） |
| `litellm/router_strategy/` | 17 个路由策略实现（least_busy、lowest_cost、adaptive_router 等） |
| `litellm-rust/` | Rust 重写核心：70+ crates（gateway/router/inference/cache/secrets/token-counter 等） |
| `enterprise/` | 商业授权代码；`litellm/proxy/enterprise` 是指向它的符号链接 |
| `tests/` | 5425 个 .py，16 个子目录（unit/integration/e2e/benchmarks 等） |
| `ui/`、`docs/`、`cookbook/`、`helm/`、`terraform/`、`docker/` | 管理界面、文档、示例、K8s/Terraform 部署、容器 |

组织方式：单一 monorepo 内"SDK + 网关 + Rust 核心 + 企业代码 + 部署资产"并存；Python 侧以能力域切包（`chat/`、`embedding/`、`responses/`、`rerank/`… 见 `litellm/llms/base_llm/` 子目录），提供商按目录隔离，新增提供商即加一个子目录，扩展模式高度模板化。

## 5. 核心架构与关键实现

### 5.1 Provider 统一抽象

每个提供商实现一组转换函数，基类约定在 `litellm/llms/base_llm/`（按 chat、embedding、responses、rerank、ocr、videos 等能力域分目录）。以 Anthropic 为例（`litellm/llms/anthropic/chat/transformation.py`）：

- `class AnthropicConfig(AnthropicModelInfo, BaseConfig)`（L532）
- `transform_request()`（L2348）：OpenAI 格式请求 → Anthropic Messages API 格式
- `transform_response()`（L3219）：提供商响应 → 统一 ModelResponse
- `map_openai_params()`（L1899）：OpenAI 参数名 → 提供商参数名

实际 HTTP 由统一 handler 执行：`llms/custom_httpx/llm_http_handler.py` 的 `BaseLLMHTTPHandler.completion()`（`ARCHITECTURE.md` 请求时序图）。因此接入新提供商只需写一个 `transformation.py`，传输、流式、重试逻辑全部复用。入口分发在 `litellm/main.py` 的 `completion()`（L5557 起，全文件 9650 行）。

### 5.2 Router：负载均衡 / 重试 / 降级

Router 近期重构为 **host/core 分离**（`litellm/router.py` 模块 docstring 明示）：

```python
"""
litellm.Router: load balancing, retries, fallbacks and cooldowns across deployments.

``Router`` holds no routing logic. It builds its configuration (``litellm/router_config.py``),
picks one ``RouterCore`` at construction (``litellm/router_core.py``) and delegates every
member to it ...
"""
```

- `litellm/router.py`（14KB）：仅做委托的外壳，另含 `router_python_host.py` 支持 Rust Core 注入。
- `litellm/router_python_core.py`（17669 行，846KB）：`class PythonCore`（L1329）持有全部路由逻辑——部署注册表、选择、重试、fallback、冷却、用量记账。
- **路由策略**：`RoutingStrategy` 枚举（`litellm/types/router.py` L1283-1288：least-busy、latency-based-routing、usage-based-routing(-v2)），策略实现目录 `litellm/router_strategy/` 含 17 个文件，包括新的 `adaptive_router/`、`auto_router/`、`complexity_router/`、`quality_router/`、`lar1_routing.py`（按请求复杂度/质量自适应选型）。
- **冷却与容错**：`litellm/router_utils/cooldown_cache.py`（`class CooldownCache` L41）+ `cooldown_handlers.py`，配合 `RetryPolicy`/`AllowedFailsPolicy`（`litellm/types/router.py`）在失败/429 时把部署拉入冷却队列。
- **流中重试**：`FallbackAwareStreamWrapper`/`_mid_stream_retry_trigger`（`router_python_core.py` L964-1218）支持流式首块未到时的静默换部署重试，首块后不再重试，避免重复输出。

### 5.3 代理服务（AI Gateway）请求链路

`litellm/proxy/proxy_server.py`（22264 行）是 FastAPI 应用本体；路由按域拆成 68 处 `include_router()`（grep 实测）：key_management、team、organization、customer、batches、files、passthrough、anthropic_endpoints、agent_endpoints、health 等。核心链路（`ARCHITECTURE.md` 时序图 + 源码）：

1. `proxy_server.py` 端点 → `proxy/auth/user_api_key_auth.py` 鉴权（查 API key，DualCache 未命中回源 PostgreSQL）
2. `proxy/litellm_pre_call_utils.py` 注入元数据 → `proxy/route_llm_request.py`（42KB）决定走 Router 还是直通
3. Router → `litellm.acompletion()` → 提供商 handler
4. 响应经 `litellm/cost_calculator.py` 计价，写入 `response._hidden_params["response_cost"]`，并以 `x-litellm-response-cost` 响应头返回
5. 异步日志钩子 → `proxy/db/db_spend_update_writer.py` 批量落 PostgreSQL

即网关的差异化价值全在横切面：鉴权、限流（`proxy/hooks/` 并发/TPM/RPM 限制）、预算、guardrails（`litellm/guardrails.py` 与 `proxy/` hooks）、花费归因。

### 5.4 Rust 核心（进行中的重写）

`litellm-rust/` 为 70+ crate 的 workspace（`litellm-rust/Cargo.toml`）：`gateway/`、`router-core/`、`inference-chat/messages/responses/ocr/transcription`、`cache-redis/s3/gcs/…`、`secrets-aws/azure/hashicorp/…`、`token-counter-*`。Python 侧通过 `litellm/rust_bridge/`、`router_python_host.py` 的 `python_host.invoke()` 桥接（`litellm/router.py` import 可见），并配套 `tests/rust-python-harness/`、`tests/test_litellm_rust/` 做双轨一致性测试。Rust 侧决策记录见 `litellm-rust/docs/adrs/`（`ARCHITECTURE.md` 引用）。

## 6. 构建与运行

```bash
# SDK 安装
pip install litellm            # SDK
pip install "litellm[proxy]"   # 网关（gunicorn/uvicorn 等 extra）

# CLI 入口（pyproject.toml [project.scripts]）
litellm --model gpt-4o         # 单模型起代理
litellm --config config.yaml --port 4000   # Dockerfile CMD 默认 --port 4000
lite                           # CLI（litellm_cli.main:cli）

# 容器化
docker compose up              # docker-compose.yml；另有 hardened/liteadmin 变体
# Dockerfile: uv sync --frozen 安装依赖, ENTRYPOINT docker/prod_entrypoint.sh

# 本地开发（Makefile）
make install-dev && make test-unit-llms      # 按域拆分的单元测试目标
make lint                     # Ruff + basedpyright(strict) + 循环导入/导入安全检查
make format && make check     # 格式化、CI 等价 lint（针对 staged/diff）
```

数据库：PostgreSQL（`schema.prisma`，迁移在 `migrations/`），Redis 可选（双缓存）。Helm chart（`helm/`）与 Terraform module（`terraform/`）用于生产部署。

## 7. 测试与工程质量

- **规模**：`tests/` 5425 个 .py、16 个子目录（实测）。分层清晰：`unit/`（provider 单测）、`integration/`（真实 API，VCR 录制）、`e2e/` + `e2e_harness/`（网关端到端）、`benchmarks/`、`audio_tests/`、`documentation_tests/`（文档一致性）、`windows_tests/`、`code_coverage_tests/`。
- **录制回放**：VCR 测试体系（`tests/VCR_TESTING.md`、`_vcr_conftest_common.py`、`_openai_record_replay_proxy.py`），CI 有 `e2e_record_replay.yml` 工作流。
- **框架**：pytest 9.0.3 + pytest-asyncio/xdist/rerunfailures/mock + respx（`pyproject.toml` dev 依赖）。
- **CI**：`.github/workflows/` 30+ 条流水线——CodeQL、OSV 扫描、OpenSSF Scorecard、mutation-test、`test-e2e-redis-chaos.yml`（Redis 混沌）、`check-schema-sync.yml`（Prisma schema 同步）、`cost-map-guard.yml`（价格表守卫）、conventional-commits、issue 自动分类。
- **质量门禁**：Makefile 提供 `lint-ruff-strict`（严格规则计数基线门禁）、`lint-basedpyright`（per-rule 错误数门禁）、`lint-openapi-docs`（OpenAPI 文档覆盖门禁）、`check-circular-imports`——对超大代码库用"计数基线防劣化"策略。
- 观察：3020 文件 103 万行的 Python 代码库保持 strict 类型检查与 30+ 条 CI 门禁，工程化程度在开源 Python 项目中属第一梯队。

## 8. 社区与维护状态

- **极度活跃**：最近 push 2026-11-02（分析前一天，gh api）；HEAD 版本号 v1.109.0 已超最新 GitHub release v1.106.6（2026-10-28），说明约每 2-3 天一个 patch 版本，发版节奏极快。
- **社区体量**：open issues 5610、forks 12928（gh api）；issue 侧自动化完善（去重、关键词打标、分类、stale 均有专门 workflow，`.github/workflows/`）。
- **采纳**：README 列出 OSS 采用者 Stripe、Netflix、OpenAI Agents SDK、OpenHands、Google ADK 等。
- **商业化**：YC W23 公司 BerriAI 主导，企业版（`enterprise/`）+ 托管服务并行；贡献者总数 API 分页未取到，标"未确认"，但从 `docs/adrs/`、AGENTS.md、GEMINI.md 等基础设施看有强内部工程团队。

## 9. 亮点与风险

**亮点**

1. 提供商抽象设计出色：`transformation.py` 三函数契约（transform_request/transform_response/map_openai_params）+ 统一 HTTP handler，164 个提供商目录即 164 份可复制的接入模板（`litellm/llms/`）。
2. 路由/容错工程化深：17 种路由策略、冷却缓存、策略化重试（RetryPolicy）、流首块前静默换部署重试，是同类网关中最完整的（`litellm/router_strategy/`、`router_python_core.py`）。
3. Router host/core 分离 + Rust Core 可插拔，为性能演进铺路且保持 Python 兼容（`router.py` docstring、`litellm-rust/`）。
4. 测试与 CI 纪律严明：VCR 录制回放、混沌测试、变异测试、文档/价格表守卫、基于错误计数的渐进式 lint 门禁（`.github/workflows/`、`Makefile`）。
5. 双形态（SDK/网关）共享同一抽象，进程内与集中部署两种集成路径都成立（`ARCHITECTURE.md`）。

**风险 / 局限**

1. 巨型文件集中：`proxy_server.py` 22264 行、`router_python_core.py` 17669 行、`main.py` 9650 行，单文件复杂度已到重构临界点（行数实测；router 拆分正是对此的回应）。
2. 版本节奏极快（2-3 天一版）+ 依赖面广（100+ 提供商 API 漂移），升级时行为变更风险高，需锁定版本。
3. 开源/商业边界在仓库内交织：`enterprise/` 专有授权、部分代理功能企业限定，评估时需逐项核对（`LICENSE`、`enterprise/`）。
4. Rust/Python 双轨过渡期存在两套实现与一致性测试成本（`tests/rust-python-harness/`），架构处于大迁移中。

## 10. 总结评价

LiteLLM 是当前 LLM 网关赛道事实上的开源标杆：以"OpenAI 格式统一 100+ 提供商"的抽象为核心资产，向上用 FastAPI 网关叠加密钥/预算/限流/花费归因等企业能力，向下正在用 Rust 重写性能关键路径。它证明了横切面（路由、容错、成本治理）才是 LLM 基础设施的主要复杂度所在。适合：需要多提供商容灾、成本控制与集中治理的平台/基建团队直接部署；想学习"如何抽象 N 家供应商 API"的工程师读 `litellm/llms/base_llm/` + 任一 `transformation.py` 与 `router_strategy/`；大型代码库工程化治理（基线式 lint 门禁、VCR 测试体系）也值得参考。阅读源码建议避开 `proxy_server.py` 全文，从 `ARCHITECTURE.md` 的时序图入手按链路抽读。
