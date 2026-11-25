# The-Vibe-Company/quivr 项目分析

> 星数 39647（快照 2026-10-24）｜主语言 Go（gh api 现语言；2023-2025 年历史主语言为 Python）｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：开源的持续内容流搜索引擎——把不断到达的文档流（新闻、邮件、RSS、帖子）持久化存储、秒级可检索，并对关注的主题持续产生告警；这是 Quivr 2（Go 重写版），与早期 Python RAG"第二大脑"已是两个形态完全不同的产品。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | The-Vibe-Company/quivr（QuivrHQ/quivr 302 重定向至此） | gh api repos/The-Vibe-Company/quivr |
| 星数 / Fork | 39647 / 3729（2026-11-09 仍活跃） | gh api |
| 创建时间 | 2023-05-12（Python 时代建仓） | gh api |
| 语言 | Go 1.27.3（go.mod）；插件含 Python | go.mod |
| 许可证 | MIT（Copyright (c) 2026 The Vibe Company） | LICENSE |
| 最新版本 | v2.0.2（2026-10-30）；v2.0.0（2026-10-29） | CHANGELOG.md、gh releases |
| 最近推送 | 2026-11-09 | gh api |
| 现存 tag | 仅 v2.0.0-alpha.1 起共 10 个，旧版 tag 已清理 | gh api tags |
| 贡献者 | 约 117 人（含 Python 时代） | gh api contributors 分页 |

## 2. 项目是什么，解决什么问题

Quivr 2 是一个无自带 UI 的引擎（engine）：应用通过 HTTP API（`/v0`，OpenAPI 契约 91 个端点，contracts/http/v0/openapi.yaml）向它发送文档，或让它按计划从 RSS、Microsoft 365 邮箱、X 列表等来源采集。它保存每个文档的每个版本（更正产生新版本而非覆盖，撤回立即离开搜索并通知已捕获的告警），使新文档在数秒内可检索，并对照用户保存的告警持续判match（README.md）。

解决的三个核心问题（README.md、docs/dated/design/quivr-v2-architecture-overview.md）：
1. **持久摄取**：先确认入库（Ingestion Receipt）再后台处理——故障只延迟处理、绝不丢已确认内容；搜索索引永远可从存储重建（PostgreSQL + 对象存储是持久真相，Weaviate 只是可重建的搜索投影）。
2. **混合检索**：关键词（BM25）、语义（向量）或两者融合，每条结果指向精确段落、文档与版本，可引用（citation）。
3. **持续监控**：保存布尔关键词树或自然语言描述的告警，新文档命中即记录 Match（含理由），经签名 webhook 投递。

关键断裂点必须如实记录：**本仓库历史上是 QuivrHQ 的 Python RAG "第二大脑"产品（2023 年爆红，39.6k 星主要由此积累）；2026 年 Quivr 团队成立 The Vibe Company（AI 机构），将仓库随组织转移，用 Go 从零重写为"持续内容流搜索引擎"（Quivr 2）并于 2026-10-29 发布 2.0.0**。旧 Python 代码不存于独立仓库，保留在本仓库 git 历史中（856 个分支里有大量 brain-management-*、backend-python-clean 等旧时代分支）；QuivrHQ 组织仍存（8 个小型公开仓库：slackbot、LeJuge、multimodal-rag 等）。GitHub topics 仍保留旧时代的 react/chatbot/groq 等，未随转型更新。设计阶段记录见 docs/dated/design/（法语，2026-09-03 定稿）；AGENTS.md 明确"The Vibe Company 由 Quivr 团队创立"（README.md）。

商业模式：引擎核心 MIT 开源；AGENTS.md 规定核心不得含任何业务规则（配额/计费/套餐），商业层在其上层产品（quivr.app / The Vibe Company 服务）实现。第一方插件中 `described` 类告警默认把文本发往 TypeSafe（团队的 Jev 分类服务），本地向量检查为免费替代（plugins/alerts/README.md）；`jev.rerank` 重排插件按次计费（plugins/core-retrieve/README.md 的 deep profile：3 秒、1 美分预算）。

## 3. 技术栈与依赖

| 层 | 技术 | 出处 |
| --- | --- | --- |
| 引擎主语言 | Go 1.27.3，单二进制 `quivr` | go.mod、cmd/quivr/main.go |
| 持久工作流 | Temporal SDK v1.50（摄取/连接器/回填/重建工作流） | go.mod、internal/orchestration/temporal/ |
| 持久存储 | PostgreSQL（pgx/v5，96 个迁移文件）+ S3 兼容对象存储（aws-sdk-go-v2，本地用 SeaweedFS） | migrations/、internal/adapters/{postgres,s3}/、deploy/seaweed/ |
| 搜索投影 | Weaviate 1.39.14（BM25F + 向量 + hybrid），TEI（Text Embeddings Inference）承载 E5 嵌入 | internal/adapters/weaviate/、internal/adapters/tei/、CHANGELOG |
| 契约与代码生成 | OpenAPI + oapi-codegen（生成的 Go client 在 client/）；JSON Schema 校验 | contracts/http/v0/openapi.yaml、client/ |
| 插件协议 | 语言中立 HTTP 协议 + HS256 签名（ADR 0007），Go/Python 两套 SDK（sdks/） | contracts/plugins/v0/、sdks/ |
| 可观测 | OpenTelemetry 全套（trace+metric，grpc/http exporter） | go.mod、internal/telemetry/ |
| Agent 接入 | MCP（modelcontextprotocol/go-sdk，stdio） | internal/online/mcp.go |
| 第一方插件 | Go：core.ingest、core.retrieve、rss、m365-mail、x-list、object-storage-archive；Python：alerts、jev-rerank、newsml-g2、push-source、hosted-embed | plugins/*/quivr-plugin.yaml |
| 演示前端 | React 19 + Vite 8 + TypeScript（quivr-search 监控面板） | quivr-search/package.json |

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| cmd/ | `quivr` 单二进制（api/worker/migrate/storage/journal + search/records/apply/mcp + plugin 工具）与 quivr-autoscaler（cmd/quivr/main.go、internal/app/run.go） |
| internal/ | 引擎全部实现，一个领域一个 Go 包（37 个包，881 个 .go 文件共约 21.4 万行） |
| contracts/ | HTTP API（OpenAPI）与插件协议契约 + 双向往返校验（checks/） |
| plugins/ | 12 个第一方插件（连接器、规范化、摄取、检索、告警） |
| sdks/ | Go（quivrplugin）与 Python 插件 SDK |
| migrations/ | 96 个 PostgreSQL 迁移（001_corpora.sql 起，legacy.json 标记历史风险） |
| internal/adapters/ | postgres/s3/weaviate/tei/pluginhttp 五个基础设施适配器 |
| deploy/ | Compose、Railway、single-machine、Modal、MLflow、vector-host |
| quivr-search/ | 基于公开 API 的演示监控面板（法语界面，Fil/Alertes/Sources/Admin 四标签） |
| docs-site/ | 文档站源码（MDX）；docs/ 贡献者文档、ADR、冻结的 dated 记录 |
| tests/ | acceptance（真实栈端到端）、plugin-contract、fakes、load、measurement |

组织方式评述：严格单二进制 + 单模块（module github.com/The-Vibe-Company/quivr）的 Go 单体，但 internal/ 内按领域垂直切包（adapters 237 文件最大、transport 85、plugins 93），插件则是独立 module（各自 go.mod，如 plugins/core-retrieve/go.mod），通过 HTTP 契约与引擎解耦。规模上这是一个远超典型开源项目成熟度的工程（21 万行 Go、8.4 万行测试代码）。

## 5. 核心架构与关键实现

### 5.1 持久摄取管线（Temporal 工作流 + Pipeline Plan 钉扎）

每个提交先获得不可变 Ingestion Receipt（恰好一次解析为 created/duplicate/withdrawal applied/conflict，CONTEXT.md），随后由 Temporal 工作流在后台处理。工作负载分 live/bulk 两个队列类（internal/workqueue/queue.go），worker 崩溃或插件排空不会把工作钉在死版本上：

```go
// internal/orchestration/temporal/ingestion.go
const stepHeartbeatTimeout = 10 * time.Second
// Pinner pins each piece of work to the Pipeline Plan it started on (Spec 5):
// every activity of the work, retries and restarts included, resolves its
// plugins in that plan.
type Pinner interface {
    Pin(ctx context.Context, kind, org, id string) (context.Context, error)
    Release(ctx context.Context, kind, org, id string) error
}
```

不可变 Pipeline Plan 把每个角色（normalizer、alert rule、connector、ingestion/retrieval 插件）映射到具体插件注册；激活/回滚返回 202 + Operation 异步完成（docs/adr/0009、CHANGELOG 2.0.0-alpha.7）。强制贡献无法安全完成时进入 Quarantine 持留，待运维换新 plan 重处理（internal/quarantine/、CONTEXT.md）。

### 5.2 混合检索（Weaviate BM25 + 向量，检索插件多轮协议）

检索被做成协议：引擎在索引中找候选，检索插件最多三轮决定最终排序（CONTEXT.md "Retrieval Round"）。默认第一方插件 core.retrieve 的策略（plugins/core-retrieve/README.md）：

- `lexical`：Weaviate `bm25` + group_by record；`semantic`：`near_vector` 在每个 served space；`hybrid`：Weaviate hybrid，alpha 默认 0.5，融合可选 relative score 或 RRF（常数 60）；
- 关键词分析器在摄取时写入字段的独立分析副本（`folded` 语言无关折叠、`french_light` 法语轻词干，internal/keywords/keywords.go）；
- 无 served space 时 hybrid 自动降级 bm25 并在 `retrieval_profile.degraded` 报 `vectors_unavailable`（core-retrieve README）；
- 嵌入由 core.ingest（multilingual-e5-small 经 TEI）产生，向量空间可评估/晋升（Space Promotion）而不停机（CONTEXT.md）；
- `jev.rerank` 插件可对 hybrid 排序重排，不可用时回退未重排结果（plugins/jev-rerank/quivr-plugin.yaml）。

### 5.3 告警与监控（saved query → subscription → match → delivery）

internal/monitoring/（24 文件）实现完整链条：Saved Query（不可变版本）→ Subscription（钉住一个 query 版本与投递策略）→ Match（幂等判定，渐进富化后重判收敛于同一 match）→ Delivery/Attempt（传输至少一次，与判定分离）→ 签名 webhook 重试（delivery.go、retry.go、webhook.go）。判定逻辑在插件侧：

```json
{"kind": "keywords", "match": {"all": [
  {"term": "Acme"},
  {"any": [{"term": "grève"}, {"term": "strike"}]},
  {"not": {"term": "sport"}}]}}
```

（plugins/alerts/README.md；布尔树最多嵌套 6 层、每组 1-64 节点；另有 `meaning` 本地向量检查与 `described`（Jev/TypeSafe）两类。）

### 5.4 插件体系与 MCP

插件是独立部署的 HTTP 服务，manifest（quivr-plugin.yaml）声明身份、兼容区间（engine/plugin_api 语义版本范围）、contributions（connector/normalizer/ingestion/retrieval/subscription 六类）与配置 schema；引擎用 Contract Runner 以引擎自身校验+规范 fixture 认证插件（internal/plugins/、contracts/plugins/v0/、AGENTS.md）。调用以幂等键重试且必须收敛（Plugin Invocation，CONTEXT.md）。`quivr mcp` 用官方 Go MCP SDK 在 stdio 上暴露 5 个只读/摄取工具（list_corpora/search/read_record/ingest_text/read_receipt），分 read/ingest 两种 profile（internal/online/mcp.go）。

## 6. 构建与运行

来自根 README.md 与 Makefile（未实际运行，仅静态核实）：

```bash
git clone https://github.com/The-Vibe-Company/quivr.git && cd quivr
make dev                 # Docker Compose 启动全栈（首次下载约 1GB，含 958MB E5 模型）
eval "$(make -s env)"    # 指向本地 API
make demo                # 附加 quivr-search 演示面板 http://127.0.0.1:5183
make check               # ~2 分钟静态检查+单测，无需 Docker（提交前要求）
make verify              # 在隔离栈上跑端到端旅程（linux/amd64 only）
make test                # test-go + test-python + test-eval + 两套 SDK 测试
make measure             # 检索基线测量（24 条 CC0 查询，docs/dated/evidence/the-661-*）
```

引擎命令：`quivr api|worker|migrate|storage|journal`，CLI 在线命令 `quivr search|records|apply|mcp`（internal/app/run.go、cmd/quivr/main.go）。平台支持：linux/amd64 与 macOS Apple Silicon（docs/quivr-v2-remaining-limits.md）。

## 7. 测试与工程质量

- 407 个 Go 测试文件共约 8.4 万行（占 Go 代码 39%），另有 17 个 Python 测试文件；分层为单测（包内）→ tests/plugin-contract（契约）→ tests/acceptance（真实栈，自带 README 说明顺序与负载敏感性）→ tests/load / measurement（docs/quivr-v2-remaining-limits.md）。
- 测试纪律写进 AGENTS.md：每个契约一个 owner test、禁止 wall-clock 等待、每层有耗时预算、每个 flaky 必须提交带根因的 ticket。
- CI：8 个 workflow（.github/workflows/）——verify.yml、release-images.yml（签名镜像）、release-please.yml、security-nightly.yml、measure-upgrade.yml（升级测量）等；发布走 release-please（release-please-config.json），镜像签名 + SBOM 类核验（SECURITY.md 提到"verify release inventories"）。
- 文档治理罕见地严格：docs/inventory.toml 声明"活文档"清单，`make docs` 校验行预算；ADR 与 dated 文档冻结不可改，只能以新文档取代（AGENTS.md、docs/adr/）。
- 质量观察：依赖面极小而精（go.mod 仅 9 个直接依赖组）；客户信息用加盐哈希 denylist 阻止入库（scripts/denylist.py，AGENTS.md）；错误处理、TLS、幂等性在注释中普遍引用 ticket 编号（THE-xxx）。

## 8. 社区与维护状态

- 活跃度极高但结构特殊：最近推送 2026-11-09（gh api）；v2.0.0→2.0.2 三连发于 2026-10-29/30；PR 编号已达 #3976（CHANGELOG）。
- 贡献者约 117 人，但其中绝大多数属 Python 时代（2023-2025）；Go 重写期的工作由 The Vibe Company 的 AI agent 舰队完成——AGENTS.md 描述了完整的"agent 流水线"：Linear 为共享数据库、多个 agent（Claude Code/Codex/Conductor）并行领票、仅协调者可合并、Vercel 私有仪表盘展示进度。这是本仓库最独特的维护模式。
- issue 极少（open_issues 16，gh api），issues 几乎不经社区流入；外部贡献路径存在于 CONTRIBUTING（README 指向 AGENTS.md 与 docs/start/contributor.md），但实际门槛极高（make check + Commitizen + 契约测试）。
- release 节奏：2.0.0-alpha.1→2.0.2 共 10 个 tag，alpha 期约 2 个月（2026-08 前后至 10-29），节奏由 release-please 自动化。

## 9. 亮点与风险

亮点：
1. **持久优先的架构自洽**：PostgreSQL+S3 为唯一真相、索引一律可重建、摄取回执恰好一次、Pipeline Plan 不可变钉扎——为"内容流不断到达"这一场景做的端到端一致性设计（CONTEXT.md、internal/）。
2. **语言中立的插件协议**：六类 contribution、语义版本兼容区间、HS256 签名、契约 runner + 规范 fixture，第一方插件自己就是示范（Go 与 Python 混用）（contracts/plugins/v0/、sdks/）。
3. **工程纪律罕见地高**：领域词汇表、冻结 ADR、声明式文档清单、测试所有权规则、denylist 哈希防泄漏（AGENTS.md、CONTEXT.md、docs/adr/）。
4. **MCP 一等公民**：`quivr mcp` 让 AI agent 以引用（精确段落+版本）的方式消费搜索结果，契合 agent 时代检索需求（internal/online/mcp.go）。
5. **透明的能力边界**：docs/quivr-v2-remaining-limits.md 公开列出全部已知技术限制并逐条挂 ticket，而不是隐藏。

风险/局限：
1. **品牌与星数严重错位**：39.6k 星几乎全部来自 Python RAG 时代，当前仓库是另一个产品；GitHub topics（react/chatbot/rag 等）未更新，新访客极易误解（gh api、README.md）。
2. **生态断层**：旧 Python 版的集成、教程、社区内容全部失效；仓库内无旧版迁移指南，2.0 alpha 间甚至不支持原地升级（CHANGELOG BREAKING CHANGES）。
3. **部署复杂**：完整栈需 PostgreSQL+对象存储+Weaviate+Temporal+TEI 五类依赖；引擎自身无 UI，需要上层应用（deploy/、README.md）。
4. **平台与版本前沿**：Go 1.27.3、仅承诺 linux/amd64 与 macOS arm64；2.0 刚发布两周，`/v0` API 的跨 2.x 兼容承诺尚待时间检验（go.mod、docs/quivr-v2-remaining-limits.md）。

## 10. 总结评价

Quivr 是一次教科书级的产品转型：同一仓库从 2023 年的 Python RAG"第二大脑"（星数来源）彻底重写为 2026 年的 Go 持久内容流搜索引擎，断裂点清晰可辨（建仓 2023-05、tag 全部从 2.0.0-alpha 起、LICENSE 版权人 2026 The Vibe Company、旧代码只在 git 历史中）。转型后的代码库以 21 万行 Go、39% 测试占比、冻结的决策记录和 agent 舰队开发流程，呈现出远高于社区项目均值、更接近一流基础设施团队的水准；其"存储为真相、索引可重建、插件语言中立、告警幂等投递"的架构对构建监控/情报类系统的工程师有很高的参考价值。适合谁：需要自建持续情报/监控管道（新闻监测、合规告警、邮箱监听）的后端与平台工程师直接可用；对检索引擎、Temporal 工作流、插件协议设计感兴趣的工程师值得通读 internal/ 与 contracts/；而只想找"开箱即用的 RAG 聊天机器人"的用户应另寻项目——那已经是这个仓库的前世。
