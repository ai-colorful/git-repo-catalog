# langfuse/langfuse 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-17

## 1. 项目速览

一句话定位：开源 LLM 工程平台——以 OpenTelemetry 为底座的 LLM 可观测性（trace/评估/指标）+ 提示管理 + 数据集实验 + AI 网关，可云用可自托管。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | langfuse/langfuse | gh api |
| 星数 | 39,500（任务快照）；gh api 核实时 37,500 | gh api repos/langfuse/langfuse |
| 主语言 | TypeScript（含两个 Rust 子服务） | gh api / ai-gateway/Cargo.toml |
| 许可证 | MIT（核心）+ EE 目录商业例外（GitHub 识别为 NOASSERTION） | LICENSE、ee/LICENSE |
| 当前版本 | v4.81.0（2026-11-16 发布） | package.json、gh api releases |
| 创建时间 | 2023-05-18 | gh api |
| 最近 push | 2026-11-16（分析前一天） | gh api |
| 背景 | YC W23；2026 年 1 月起属 ClickHouse 公司 | README.md 徽章与招聘段 |

## 2. 项目是什么，解决什么问题

Langfuse 定位为 "AI engineering platform"，解决 LLM/Agent 应用在生产环境中的可观测与迭代问题。应用方通过 SDK 或 OpenTelemetry 端点把每次 LLM 调用、检索、工具调用、Agent 步骤上报为 trace/observation（generation 是 observation 的一种），平台负责存储、检索、可视化、成本核算与质量评估（README.md「Core Features」节）。

核心能力六件套（均可在源码定位）：
- **可观测性**：trace/observation/score 数据模型，基于 OTLP 的摄取端点（`web/src/pages/api/public/otel/`、`packages/shared/src/server/otel/OtelIngestionProcessor.ts`）。
- **提示管理**：提示按 `(projectId, name, version)` 版本化，支持 label（如 production）、tags、commit message（`packages/shared/prisma/schema.prisma:906` model Prompt），配套 Playground（`web/src/features/playground/`）。
- **评估**：LLM-as-judge、代码评估器（AWS Lambda/本地执行）、decision model 评估器三类执行路径（`packages/shared/src/server/evals/`），由 `worker/src/queues/evalQueue.ts` 驱动。
- **数据集与实验**：datasets/experiments 公共 API 与前端（`web/src/pages/api/public/datasets/`、`web/src/features/experiments/`）。
- **AI Gateway（beta）**：独立 Rust 服务，一个端点代理 OpenAI/Anthropic 原生 API，集中管理 provider key、按 key/模型核算花费，并把每次调用以 OTLP 写回 Langfuse（`ai-gateway/README.md`）。
- **指标与仪表盘**：ClickHouse 上的自定义 dashboard、metrics API、告警（`web/src/features/metrics/`、`web/src/features/dashboard/`）。

商业化形态：Langfuse Cloud（托管）+ 自托管（MIT 核心）。EE 目录内的功能（SSO、审计日志、admin API、计费、SFDC 同步、UI 定制、verified domains 等）仅限企业版/云（`web/src/ee/features/`、`ee/README.md`）。2026 年 1 月起项目并入 ClickHouse 公司（README 招聘段），banner 也改为 "Langfuse by ClickHouse"。

## 3. 技术栈与依赖

- **语言**：TypeScript 主体（Node 24）；`ai-gateway/` 与 `packages/native/` 为 Rust（前者独立服务，后者是 napi-rs 原生插件）。
- **Web 全栈**：Next.js（16.x，App Router + Pages API 并存：`web/src/app/` 与 `web/src/pages/api/`），tRPC 用于内部管理接口（`web/src/server/api/routers/`），React + Tailwind + Radix（pnpm-workspace.yaml patches 节）。
- **数据层**：Postgres 17 + Prisma（元数据/配置，486 个迁移，`packages/shared/prisma/migrations/`）；ClickHouse 26.3（观测数据，112 个迁移文件，`packages/shared/clickhouse/migrations/canonical/`）；Redis 7（BullMQ 队列）；S3/MinIO（事件原始 blob）（`docker-compose.yml`）。
- **消息**：BullMQ + Redis，队列定义集中在 `worker/src/queues/`（约 30 个队列）。
- **观测自身**：OpenTelemetry 全链路（`@opentelemetry/*` 依赖遍布 web/worker；`worker/src/instrumentation.ts`）。
- **工具链**：pnpm 12 + Turborepo 2.11、TypeScript 7（原生编译器，catalog 中以 ts6 兼容包给 Storybook 等）、oxlint/oxfmt、knip、husky、release-it（根 package.json、pnpm-workspace.yaml）。
- **供应链治理**：依赖 5 天发布隔离（minimumReleaseAge: 7200）、trustPolicy no-downgrade、数十条带 CVE 编号的 override（pnpm-workspace.yaml overrides 节）。

## 4. 目录结构与代码组织

pnpm + turbo monorepo，固定版本组 `[langfuse, web, worker]` 由 release-it 同步发版（pnpm-workspace.yaml versioning）。

| 目录 | 职责 | 证据 |
| --- | --- | --- |
| `web/` | Next.js 全栈：管理 UI + tRPC + 公共 REST API + OTel 端点 | web/package.json、web/src/pages/api/public/（35 个资源组） |
| `worker/` | 异步摄取/评估/集成 worker（BullMQ 消费者） | worker/src/queues/、worker/src/services/ |
| `packages/shared/` | Prisma/ClickHouse 客户端、摄取批处理、evals 执行、队列类型定义 | packages/shared/src/server/ |
| `packages/native/` | Rust napi-rs 插件：ClickHouse Native 编码、Topics 聚类（UMAP+HDBSCAN） | packages/native/README.md |
| `ai-gateway/` | 独立 Rust 网关（/openai/v1、/anthropic/v1 两命名空间） | ai-gateway/README.md |
| `ee/`、`web/src/ee/`、`worker/src/ee/` | 企业版功能（SSO、审计日志、计费等） | ee/README.md、web/src/ee/features/ |
| `fern/` | OpenAPI 规范与 API 文档生成配置（Fern） | 根 package.json openapi:export 脚本 |
| `specs/`、`.agents/`、`skills-lock.json` | 内部规格文档与 agent 协作配置（AGENTS.md、SKILL.md 体系） | 顶层 ls |
| `docker-compose*.yml`（6 份） | 生产/开发/redis-cluster/azure 变体部署编排 | 顶层 ls |

组织评述：典型的"前后端分离于同一 monorepo"形态——web 即 API 网关（接收写入与查询请求），worker 承担全部重计算；shared 包是两者契约层。代码量（含生成物）：web/src 约 96 万行、packages/shared 约 21 万行、worker 约 18 万行、ai-gateway 约 2.3 万行 Rust（find+wc 统计）。业务按 feature 垂直切分（web/src/features/ 下 60+ 个 feature 目录），架构纪律由 `.agents/ARCHITECTURE_PRINCIPLES.md` 与 dependency-cruiser/knip 等工具约束。

## 5. 核心架构与关键实现

### 5.1 摄取管线（trace/span 数据流）

写入路径两级解耦：web 接收 → S3 落盘原始事件 → Redis BullMQ → worker 消费写入 ClickHouse。

1. 公共入口 `web/src/pages/api/public/ingestion.ts`（legacy）与 `web/src/pages/api/public/otel/`（OTLP）：zod 校验、限流（RateLimitService）、鉴权、SDK 归因（ingestionAttribution）后交给 `processEventBatch`（`packages/shared/src/server/ingestion/processEventBatch.ts`）。
2. OTel 路径由 `packages/shared/src/server/otel/OtelIngestionProcessor.ts` 把 OTLP span 映射为 Langfuse trace/observation 模型（ObservationTypeMapperRegistry 识别各家框架 span），入 `otelIngestionQueue`。
3. worker 侧 `worker/src/queues/ingestionQueue.ts` 消费，经 `worker/src/services/IngestionService/` 与 `worker/src/services/ClickhouseWriter/`（含 writeStrategies 分策略写、S3 slowdown 降级、DLQ `worker/src/services/dlq/`）批量写入 ClickHouse；原始事件同时进 event log 表供回放（refill/replay 脚本在 worker/package.json scripts）。

ClickHouse 表设计（`packages/shared/clickhouse/migrations/canonical/0001_traces.up.sql`）：

```sql
CREATE TABLE traces {CLICKHOUSE_CLUSTER_CLAUSE} (
    `id` String, `timestamp` DateTime64(3), `name` String,
    `project_id` String, `tags` Array(String),
    `input` Nullable(String) CODEC(ZSTD(3)),
    `output` Nullable(String) CODEC(ZSTD(3)),
    `event_ts` DateTime64(3), `is_deleted` UInt8,
    INDEX idx_id id TYPE bloom_filter(0.001) GRANULARITY 1
) ENGINE = ReplacingMergeTree(event_ts, is_deleted)
Partition by toYYYYMM(timestamp)
PRIMARY KEY (project_id, toDate(timestamp));
```

要点：ReplacingMergeTree 以 event_ts 版本排序、is_deleted 软删；按月分区、(project_id, 日期) 主键适配多租户时间窗口查询；metadata 用 Map + bloom_filter 索引。observations 表同构，附带 usage/cost details（Map(UInt64)/Decimal64(12)）实现逐 token 成本核算（0002_observations.up.sql）。

### 5.2 v4 摄取演进（events 统一事件表）

v4 正在把 legacy traces/observations 写路径迁移到统一 events 模型：`events_full`/`events_core`/`observations_batch_staging`（migrations 0038-0040），worker 内有 `worker/src/features/v4/` 与 `packages/shared/src/server/v4/`。云上可按项目启用 "v4 cutoff" 强制走新 API（`packages/shared/src/server/v4Cutoff.ts`：`if (!params.cloudRegion) return false;` —— 自托管永远不受 cutoff 影响），旧 API 通过 deprecation headers 引导迁移（`web/src/pages/api/public/ingestion.ts` 引入 INGESTION_DEPRECATION）。

### 5.3 评估体系

`worker/src/queues/evalQueue.ts` 按事件（trace-upsert 等）创建 eval job，执行层三路径（`packages/shared/src/server/evals/`）：
- `llmEvaluatorExecution.ts`：LLM-as-judge，经 llm-connections 配置模型，带限流重试与错误分类（classifyEvaluatorLlmError）；
- `codeEvalDispatcher*.ts`：代码评估器派发到 AWS Lambda（microvm 镜像依赖可见于 catalog）或本地执行；
- `decisionModelEvaluatorExecution.ts`：决策型评估器。
采样由 `worker/src/features/evaluation/deterministicSampling.ts` 控制，结果写回 scores 表并产出带 W3C trace id 的评估 trace（evalQueue.ts 中 createW3CTraceId）。

### 5.4 AI Gateway（Rust）

`ai-gateway/src/` 独立服务：`/openai/v1`（responses/chat/completions/embeddings/decisions）与 `/anthropic/v1`（messages）两命名空间；Rust 只中继原生 JSON/SSE 不改写 provider 字节，用有界 capture 层记录请求事实，每个完成的 generation 通过 OTLP 按项目批量上传（ai-gateway/README.md 表格与正文）。网关 key 与 web 侧信任标记通过 `AI_GATEWAY_INSTRUMENTATION_SCOPE_NAME`/`gateway:` key 前缀约定打通（OtelIngestionProcessor.ts 头部常量）。

### 5.5 Rust 原生插件

`packages/native/`（napi-rs）把 CPU 密集逻辑下沉 Rust：ClickHouse Native 格式编码（跳过 JSON 解析开销）、Topics 功能的 UMAP+HDBSCAN 聚类（子进程运行防阻塞，packages/native/README.md）。

## 6. 构建与运行

来自根 package.json 与 README「Self-Host」节：

```bash
# 自托管（5 分钟起步）
git clone --depth=1 https://github.com/langfuse/langfuse.git
cd langfuse && docker compose up
# 产出 6 容器：langfuse-web、langfuse-worker、langfuse-ai-gateway、
# clickhouse:26.3、postgres:17、redis:7、minio（docker-compose.yml）

# 本地开发（根 package.json scripts）
pnpm install
pnpm run dx          # 起 dev 基础设施 + 迁移 + 种子 + turbo watch dev
pnpm dev:web         # 只跑 web
pnpm dev:worker      # 只跑 worker

# 质量门（CONTRIBUTING.md：每个 PR 四检查）
pnpm run lint        # oxlint --max-warnings 0
pnpm tc              # turbo typecheck
pnpm exec knip       # 死代码/死依赖
pnpm run test        # turbo test（vitest）

# 发版（release-it，git tag v${version}，同步 bump web/worker/shared VERSION.ts）
pnpm run release
```

其他部署路径：Kubernetes Helm（首选生产）、Terraform 模板 AWS/Azure/GCP、Render/Railway 社区维护（README.md Self-Host 节）。

## 7. 测试与工程质量

- **测试规模**：vitest workspace `["web","worker"]`（vitest.workspace.ts）。worker 234 个 `*.test.ts`、packages/shared 219 个、web `__tests__` 目录 418 个文件（`.clienttest.ts(x)` 命名区分客户端测试）+ 特性目录内 in-source 测试、Playwright e2e（`web/src/__e2e__/` 14 个 spec + `test:e2e`/`test:e2e:server`/`test:e2e:ai-gateway` 三套）。测试密度高，摄取/evals 等核心路径都有单测（如 `packages/shared/src/server/otel/OtelIngestionProcessor.*.test.ts`）。
- **CI**：43 个 workflow（`.github/workflows/`），含 codeql、codespell、licencecheck、CLA、每个 PR 的 preview 部署（`pr-<N>.preview.langfuse.com`，CONTRIBUTING.md）、模型价格审计（model-price-audit）、甚至 AI 审查（claude-review-maintainer-prs、ci-runtime-analyst）。
- **工程纪律**：oxlint 零警告门槛、knip 死代码检查、OpenAPI 导出一致性检查（openapi:check）、依赖 5 天隔离 + CVE override 全部带注释、`.agents/ARCHITECTURE_PRINCIPLES.md` 维护架构原则。工程质量属开源项目第一梯队。

## 8. 社区与维护状态

- **活跃度**：gh api 显示最近 push 2026-11-16（分析前一天），open issues 1182，forks 4241。最新提交（gh api commits）：`chore(topics)...`、`fix(ci)...`、`feat(native): normalize provider usage...`——feature/fix/chore 高频滚动。
- **发版节奏**：周级以上——11 月上半月连发 v4.75.0→v4.81.0（11/04、11/05、11/09、11/10、11/12、11/13、11/16，gh api releases），同时维护 v3.227.1 旧线（11/10 补丁），4.x 主线滚动极快。
- **贡献者**：contributors API 分页累计约 235 人（含匿名）。核心开发以 langfuse 团队为主（ClickHouse 员工），README 招聘段称"6 个月团队翻倍"，属公司主导型开源。
- **多语言 README**：英/中/日/韩四语（README*.md），文档单独仓库 langfuse-docs（CONTRIBUTING.md）。

## 9. 亮点与风险

**亮点**
1. 双写架构成熟：Postgres（事务元数据）+ ClickHouse（分析型观测）+ Redis/S3 解耦缓冲，ReplacingMergeTree + 月分区 + ZSTD 的多租户表设计可直接借鉴（canonical/0001-0003 迁移）。
2. 摄取链路全链路防御：S3 SlowDown 降级、二级队列重定向、DLQ、事件回放脚本、原始事件落盘审计（worker/src/queues/ingestionQueue.ts、services/dlq/）。
3. OTel 优先的集成策略：任何 OTLP 源即可接入，官方 SDK 之外靠 span 属性映射吃下 LangChain/LlamaIndex/OpenAI Agents 等 25+ 生态（README 集成表、ObservationTypeMapper）。
4. 工程化程度极高：43 个 CI workflow、preview 部署、knip/oxlint、供应链隔离，Turborepo + Rust 混合构建示范。
5. 性能敏感路径下沉 Rust（native 编码、聚类）且提供纯 JS 回退路径。

**风险/局限**
1. 自托管运维面宽：6 容器（Postgres+ClickHouse+Redis+MinIO+2 服务）起步，升级需同时跟进三套迁移（Prisma 486 + ClickHouse 112 + background migrations）。
2. 许可非纯 MIT：ee/、web/src/ee/、worker/src/ee/、packages/shared/src/server/ee/ 为商业许可（LICENSE 首段），二次分发需区分目录边界；GitHub 也标 NOASSERTION。
3. 主仓不含 SDK：Python/JS SDK 在独立仓库（langfuse-python/langfuse-js，CONTRIBUTING.md），跨仓库版本对齐需自行关注。
4. 版本演进激进：4.x 半月 7 版 + v3/v4 双 API 并存（cutoff/deprecation 机制），自托管用户跟随升级成本不低。

## 10. 总结评价

Langfuse 是当前开源 LLM 可观测/评估赛道事实上的标杆之一：以 OpenTelemetry 为统一摄取协议，用 ClickHouse 承载海量 trace 分析、Postgres 管理配置元数据、BullMQ/Rust worker 处理异步与重计算，产品形态覆盖云、Docker、K8s、Terraform 全谱系，2026 年被 ClickHouse 收购后数据层同源性更强。对做 LLM 平台工程的团队，其摄取管线容错设计、多租户 ClickHouse 表结构和 monorepo 工程纪律最有参考价值；对使用者，自托管门槛主要是四件套存储组件的运维而非代码。适合 LLM 应用平台团队选用，也适合研究"分析型多租户 SaaS 数据架构"的工程师精读 worker/ 与 packages/shared/。
