# block/buzz 项目分析

> 星数 37517（快照 2026-10-24）｜主语言 Rust｜档位 20k-50k｜分析日期 2026-11-17

## 1. 项目速览

一句话定位：Buzz 是 Block, Inc.（Jack Dorsey 创立的支付公司，前身为 Square）出品的自托管"人 + AI Agent 同屋办公"团队通信平台，底层完全构建在 Nostr 协议之上——每条消息、反应、工作流步骤、审批、git 事件都是一条签名 Nostr 事件，"hive mind（蜂群思维）"指的是人类与大量 AI agent 通过同一事件日志协同。

| 项 | 值 |
|---|---|
| 名称 | block/buzz |
| 星数 | 37,517（gh api 实测，2026-11-17） |
| 主语言 | Rust（另含 Dart/Flutter、TypeScript） |
| 许可证 | Apache-2.0（`LICENSE`、`Cargo.toml` `workspace.package.license`） |
| 创建时间 | 2026-03-06（gh api），8.5 个月达 37.5k 星 |
| 最近推送 | 2026-11-16（分析前一天，数十次提交，PR 编号已至 #9543） |
| 维护方 | Block, Inc.（README 页脚 "Built by Block, Inc."） |
| 贡献者 | 约 135 人（gh api contributors 分页推算，含匿名） |

## 2. 项目是什么，解决什么问题

当前软件开发协作被拆散在聊天（Slack）、代码托管（GitHub）、CI 面板、issue 工具和一堆机器人胶水代码里；而 AI agent 时代又加上了新问题：agent 只能以"外挂机器人"身份假装是团队成员，缺少独立身份、审计与同等的操作面。Buzz 的回答是：把整个工作区做成一个自托管的 Nostr relay（`README.md`、`VISION.md` "The relay is the workspace"）。

技术上它就是一个实现了 NIP-01 线协议的 Nostr 中继器（`NOSTR.md`："Buzz is a Nostr relay that speaks NIP-29 (relay-based groups) natively"）：每条消息/反应/工作流步骤/canvas 更新/huddle 事件都是一个带 Schnorr 签名的事件，由 `kind` 整数分发（`ARCHITECTURE.md` §2）。新增功能 = 新增 kind 号，旧客户端零破坏。人与 agent 使用同一身份模型（secp256k1 公钥）、同一审计链、同一搜索索引；agent 拥有自己的密钥、自己的频道成员资格，"scoped by identity, not by permission flags"（`README.md`）。

产品形态上是七类 Surface：Home 信息流、Stream（Slake 式实时频道）、Forum（Discourse 式长帖）、DM、Agents 目录、YAML 工作流、Cmd+K 搜索（`VISION.md` Surfaces 表）。特色场景包括"分支即房间"——功能分支自动开频道，NIP-34 patch、CI 结果、评审与合并决定都在同一房间，合并后频道归档为"这段代码为什么存在"的永久记录（`VISION.md`）。

需要澄清的定性：Buzz 虽是"通信平台"，但不是 Twitter/Bluesky 那种公开社交 feed，而是团队协作产品；"去中心化"体现在自托管 relay + 开放协议（第三方 NIP-29 客户端可直接接入，`NOSTR.md`），而非 P2P——架构上明确"no peer-to-peer event exchange, no gossip, no replication"，relay 是唯一事实源（`ARCHITECTURE.md` §1）。多 relay 之间的 mesh 传输基于 iroh，但客户端始终只连一个 relay。

背景事实（可客观陈述）：Block 由 Jack Dorsey 联合创立并任 CEO；Dorsey 长期公开支持比特币与 Nostr 协议（曾出资资助 Nostr 开发），Buzz 选用 Nostr 作为协议底座与这一脉络一致。Buzz 与 Bluesky（Dorsey 曾早期参与后离开）走的是不同协议路线：Nostr 的客户端-中继模型 vs AT Protocol。项目 README 明确"Not blockchain"（`README.md` 末尾）。8.5 个月 37.5k 星的增长背景：大厂 + 高知名度创始人 + AI agent 协作热点题材 + 一键部署（Railway）的传播性，属客观可见因素组合。

## 3. 技术栈与依赖

来自 `Cargo.toml`（workspace）与 `docker-compose.yml`：

- **核心语言/运行时**：Rust 2021 edition、MSRV 1.88（`rust-toolchain.toml`）、tokio 多线程运行时。
- **Web/WS**：axum 0.8（WebSocket + REST）、tower/tower-http。
- **协议库**：`nostr = 0.44`（rust-nostr，含 nip44/nip98 feature）——确认整个协议层基于既有 rust-nostr 栈而非自研加密；Schnorr 验签在 `crates/buzz-core/src/verification.rs`，调用方用 `spawn_blocking` 包裹。
- **存储**：sqlx（Postgres，钉在 launchbadge 仓库未发布 rev，见 `Cargo.toml` `[patch.crates-io]` 注释三则修复）、Redis 1.0（pub/sub、presence、typing）、S3/MinIO（Blossom 媒体，经 rust-s3，aws-creds 用 fork 补丁支持 EKS Pod Identity）。
- **Mesh**：iroh 1.0.0-rc（n0 的 P2P 传输库）用于 `buzz-relay-mesh` relay 间互联。
- **Agent/工具协议**：rmcp 1.1（官方 MCP Rust SDK，用于 `buzz-dev-mcp`/`buzz-agent`）；ACP（Agent Client Protocol，agentclientprotocol.com）桥接 Goose/Codex/Claude Code（`crates/buzz-acp`）。
- **工作流引擎**：serde_yaml + evalexpr（条件表达式）+ cron。
- **可观测**：tracing + OpenTelemetry OTLP + metrics-exporter-prometheus（`prometheus.yml`）。
- **前端/客户端**：desktop（本仓库 `desktop/` 为已弃用的 0.X 版，Tauri + React + TS + Vite + Biome；1.0 版迁至 block/buzz-app）、mobile（Flutter ≥3.47.9 + Riverpod，`mobile/pubspec.yaml`）、web/admin-web（pnpm workspace + Vite + Playwright，`package.json`、`pnpm-workspace.yaml`）。
- **基础设施**：Postgres 17（events 按月 RANGE 分区）、Redis 7、MinIO（Silo fork）、Prometheus、可选 Caddy TLS（`deploy/compose/`）、Kubernetes（`crates/buzz-backend-kubernetes`、helm chart workflow）。

## 4. 目录结构与代码组织

Rust monorepo（Cargo workspace，34 个成员 crate + 排除在外的 `desktop/src-tauri`，见 `Cargo.toml` members）。crates/ 内 Rust 代码约 64 万行（1245 个 `.rs` 文件，含 desktop）。

| 目录 | 职责 |
|---|---|
| `crates/buzz-relay` | 服务端本体，Axum WS + REST，216k LOC（crates 中最大） |
| `crates/buzz-core` | 零 I/O 基础类型：kind 注册表、NIP-01 filter 匹配、Schnorr 验签（Cargo.toml 显式禁止 tokio/sqlx/redis/axum 依赖） |
| `crates/buzz-db` | Postgres 事件存储，113k LOC |
| `crates/buzz-auth` | NIP-42/NIP-98 认证、14 种 Scope、限流 trait（未实现） |
| `crates/buzz-pubsub` | Redis pub/sub、presence、typing |
| `crates/buzz-search` | Postgres FTS 查询（写入靠 events 表 generated column） |
| `crates/buzz-audit` | SHA-256 哈希链防篡改审计日志 |
| `crates/buzz-workflow` | YAML-as-code 自动化引擎 |
| `crates/buzz-acp` | relay @mention → ACP agent（Goose/Codex/Claude Code）桥，62k LOC |
| `crates/buzz-cli` | agent 优先 CLI（JSON 进/出），42k LOC |
| `crates/buzz-agent` + `buzz-dev-mcp` | 自研 ACP agent + MCP server（shell/文件编辑） |
| `crates/buzz-sdk` / `buzz-ws-client` / `buzz-media` | 类型化事件构建器 / WS 客户端 / Blossom 存储 |
| `crates/buzz-relay-mesh` / `buzz-pair-relay` | relay 间 mesh（iroh）/ 配对 |
| `crates/buzz-push-gateway` | 移动端推送网关（含 iOS PushKit 测试向量） |
| `crates/buzz-admin` / `buzz-deletion` / `buzz-backend-kubernetes` | 运维 CLI / 社区删除生命周期 / K8s 后端 |
| `crates/git-sign-nostr` / `git-credential-nostr` | nostr 签名 git 提交与凭据助手 |
| `crates/sprig`、`Dockerfile.sprig` | "deploy-anywhere" 小发行物（sprout 内部代号相关，`Cargo.toml` repository 字段指向 block/sprout） |
| `mobile/` | Flutter iOS + Android 客户端 |
| `desktop/` | 已弃用的 0.X 桌面端（1.0 在 block/buzz-app） |
| `web/`、`admin-web/` | Web 前端与管理台（pnpm workspace） |
| `migrations/` | 93 个 SQL 迁移（0001→0093），声明式 schema，禁止触发器/函数/外键（`docs/database.md`，经 `AGENTS.md` 引用核实） |
| `deploy/`、`docker-compose*.yml`、helm workflows | 生产 Compose 包 / 开发栈 / K8s |
| `docs/`、`VISION*.md`（9 份） | 协议 NIP 草案（NIP-FI/OA/AR）、数据库规范、愿景文档 |
| `benchmarks/`、`perf/`、`test-fixtures/`、`examples/countdown-bot` | 性能与示例 |

组织方式评述：典型的"按依赖分层的多 crate 单体"——`ARCHITECTURE.md` §1 明确关键原则：只有 `buzz-relay` 允许编排全部子系统，子系统间互不调用（唯一例外 `buzz-audit`→`buzz-db` 的社区写护栏）；跨子系统协调一律经 relay。边界纪律写进文档且有 conformance crate（`buzz-conformance`）与 CI 检查保障。

## 5. 核心架构与关键实现

### 5.1 事件管线（buzz-relay）

服务端核心是 11 步事件管线（`ARCHITECTURE.md` §4，实现在 `crates/buzz-relay/src/handlers/event.rs`，4472 行）：AUTH 检查 → pubkey 匹配 → 拒绝 kind 22242 → 短暂事件分流（20000–29999 不落库）→ `spawn_blocking` 验签 → 频道成员检查 → Postgres 幂等插入（`ON CONFLICT DO NOTHING`，同步生成 `search_tsv`）→ Redis 发布 → 三层扇出 → 审计（fire-and-forget）→ 工作流触发（fire-and-forget）。

验签是 CPU 热点，正确地移出 async 上下文（`crates/buzz-relay/src/handlers/event.rs:1022`）：

```rust
let verify_result =
    tokio::task::spawn_blocking(move || verify_event(&event_clone)).await;
```

三层扇出（`crates/buzz-relay/src/subscription.rs`，DashMap）：`(channel_id, kind)` 精确索引 O(1) → 频道通配索引 → 全局订阅线性扫描兜底；且**全局订阅被显式排除在频道事件之外**——私有频道不会经全局 filter 泄露，这是写进架构文档的安全边界。

### 5.2 kind 注册表与协议扩展模型

`crates/buzz-core/src/kind.rs`（1246 行）是全部 127 个 kind 的唯一事实源，标准 Nostr kind（0/1/7/1059/22242…）+ 40000–49999 自定义段（40002 富文本 v2、43001 agent 任务、46001–46012 工作流执行等）。整个产品的功能扩展都靠加常量实现，是"协议即产品"的直观体现。同时实现了标准 NIP-29 群组管理事件（9000/9001/9002/9007/9021/9022/39000 系列，`NOSTR.md` 表格逐项标注兼容状态）与自有草案 NIP（NIP-FI 联邦身份、NIP-OA agent 归属、NIP-AR 同步，位于 `docs/nips/`）。

### 5.3 人机同屋的 agent 接入层（buzz-acp / buzz-agent / buzz-cli）

这是产品差异化所在。`crates/buzz-acp`：连接 relay（NIP-42 认证），监听 @mention 事件，按频道排队（每频道最多一个 prompt 在途、后续批量合并），经 stdio JSON-RPC（ACP）驱动 1–32 个 agent 子进程（Goose/Codex/Claude Code），带崩溃自动重启（`ARCHITECTURE.md` §buzz-acp，模块 LOC 表：relay.rs 3143 行、queue.rs 2565 行等）。`buzz-agent` 是自研 ACP agent（上下文满了自动摘要历史继续），`buzz-dev-mcp` 给任意 MCP 客户端一个受控 shell + 文件编辑器（进程组击杀、有界输出，`VISION_AGENT.md`）。`buzz-cli` 面向 LLM 工具调用设计（JSON 进/出，`README.md`）。

### 5.4 可靠性细节抽样

- **慢客户端治理**：发送缓冲满 3 次即断连（grace counter，`ARCHITECTURE.md` §3 Step 4）；多节点扇出经 16 个按 (community, topic) 哈希的投递 worker 保序，积压只丢弃并让客户端 `CLOSED "resync"` 重同步，带 Prometheus 计数（`handlers/multinode_fanout.rs`，§buzz-pubsub）。
- **审计哈希链**：每条覆盖 seq/时间戳/event_id/actor/canonical JSON(BTreeMap 定序)/prev_hash，单写者用 `pg_advisory_xact_lock` 保证链不断裂，panic-safe（§buzz-audit）。
- **SSRF 防护**：`buzz-core` 的枚举拒绝式地址黑名单（IPv4/IPv6 各十几类 + 内嵌地址递归检查），应用于 webhook、JWKS、桌面端链接预览（§7）。
- **语音 huddle**：relay 内置 WebSocket Opus 帧转发（8 字节 v2 头 + 不透明 payload，软上限 25 人，`src/audio/`，§Huddle），无外部 SFU。

诚实度难得：`ARCHITECTURE.md` §9 明列 6 条"verified gaps"——限流只有 trait 无实现、sqlx 无编译期校验、审批门未端到端接通（WF-08）、部分 workflow action 返回 NotImplemented 等，且 README 用"✅/🚧/💭"三列区分现状（`README.md` "Works today" 表）。

## 6. 构建与运行

来自 `README.md` Quick start 与 `Justfile`：

```bash
# 首次：Docker + Hermit（或 Rust 1.88+/Node 24+/pnpm 10+/just）
git clone https://github.com/block/buzz.git && cd buzz
. ./bin/activate-hermit      # 钉住的工具链
just setup && just build     # setup 自动 bootstrap：.env、工具下载、Docker 服务 + 迁移

just relay      # 启 relay（ws://localhost:3000），自动拉起 Docker 服务
just check      # fmt/clippy/desktop/mobile/安全审查/文件大小等全套检查
just test-unit  # 单元测试（无需基础设施）
just test       # 全套（按需起服务）
just ci         # CI 等价命令
```

生产部署：`deploy/compose/`（Postgres/Redis/MinIO/可选 Caddy TLS）、Railway 一键部署按钮（README）、Kubernetes（helm chart + `buzz-backend-kubernetes`）。agent 用法：设置 `BUZZ_PRIVATE_KEY` 后用 `crates/buzz-cli`（`README.md`）。

## 7. 测试与工程质量

- **规模**：crates/ 内 `#[test]`/`#[tokio::test]` 共 8,414 处；`*/tests/` 目录文件 76 个；`crates/buzz-test-client` 提供 134 个 `#[ignore]` e2e 测试（27 协议 + 7/18 媒体 + 15 Nostr 互操作：NIP-50/NIP-10/NIP-17），需运行中 relay（`ARCHITECTURE.md` §buzz-test-client）。
- **测试基建**：nextest（`.config/nextest.toml`）、proptest 属性测试（workspace 依赖）、Playwright 多配置（desktop/web/admin-web 各有 config）、`TESTING.md` 专述"多 agent E2E 测试套件"、`Dockerfile.loadgen` + `benchmark-harbor.yml` 负载生成。
- **CI**：`.github/workflows/` 下 28 个 workflow——拆分式 `_ci-rust/_ci-relay/_ci-clients/_ci-desktop/_ci-security`，另有 `codex-security-review.yml`（用 OpenAI Codex 做 AI 安全评审，配套 `.github/scripts/codex-security-review.js`）、四个平台的 canary 构建（含 signed-macos）、mesh 生命周期测试、helm chart 发布。
- **质量观察**：模块边界纪律强（零 I/O core crate + 依赖禁令）；关键安全性质有回归测试锁定（`VISION_AGENT.md`："Key safety properties have regression tests that lock them down"）；依赖治理严格（sqlx/aws-creds 补丁注释详尽、Renovate + `deny.toml` + `scripts/check-sqlx-pin.py` 钉版检查）；AGENTS.md 专门为 AI agent 贡献者写了一份贡献指南（PR 前要求 agent 评审 + agent 实测 + 人类复核三道工序），是一个"AI 大规模参与开发"的真实样本——最近提交多为 agent 风格的小步 perf/fix PR（CHANGELOG v0.5.30 一版即 20+ 条）。

## 8. 社区与维护状态

- **活跃度**：极高。gh api 显示最近推送 2026-11-16（分析前一天），单日数十次提交；open issues 3,885、forks 4,955；约 135 名贡献者。PR 编号 #9543 意味着 8.5 个月近万 PR，节奏约 37 PR/天（推断）。
- **发布节奏**：GitHub Releases 为 desktop 桌面端标签（desktop-v0.5.30 于 2026-11-14，v0.5.25→v0.5.30 约每月一版，gh api）；relay 服务端走 Docker 镜像/Compose/Helm（`docker.yml`、`helm-chart.yml`）。
- **治理**：有 `GOVERNANCE.md`、`CONTRIBUTING.md`、`SECURITY.md`、`CODE_OF_CONDUCT.md`、`RELEASING.md`；两个安全评审通道（人工 SECURITY.md + AI codex-security-review）。
- **生态**：桌面 1.0 独立仓库（block/buzz-app）、姊妹愿景文档九份、社区反馈站跑在 Buzz 自己的 hosted relay 上（"吃自己的狗粮"，README Community 节）。
- 未确认：外部（Block 之外）贡献者占比；hosted 版运营规模。

## 9. 亮点与风险

**亮点**
1. 协议选型干净利落：不发明新协议，直接复用 Nostr NIP-01/29/42/98/34/Blossom + rust-nostr 库，第三方客户端可直连（`NOSTR.md`），同时把 agent 身份、git 事件、工作流全部统一进"签名事件"这一种原语。
2. 人机同权的身份模型：agent 用自己的密钥进频道、有自己的 presence kind（20003）与审计链，是"agent 作为一等成员"最完整的开源实现之一（`ARCHITECTURE.md`、`VISION_AGENT.md`）。
3. 工程质量与其文档互为印证：640k 行 Rust、8.4k 测试、28 个 CI workflow、依赖钉版带修复说明，`ARCHITECTURE.md` 是罕见的高信噪比架构文档（连"未实现"都逐条列出）。
4. 降维打击的部署体验：一个 relay = 工作区，Railway 一键或 `docker compose up`（`deploy/compose/`）。
5. AI 参与开发的范式样本：AGENTS.md 把 agent 评审/实测写进 PR 门禁，值得任何推行 AI 辅助开发的团队研读。

**风险/局限**
1. 客户端割裂：桌面 0.X 弃用、1.0 在另一仓库（block/buzz-app），本仓库 README 的功能表与 desktop/ 目录状态存在过渡期混乱；移动端仍标 🚧（`README.md` Works today 表）。
2. 关键功能未闭环：限流无实现、工作流审批门未接通、send_dm/set_channel_topic 为 stub、huddle 录制未建（`ARCHITECTURE.md` §9 六条已知缺口），生产采用前需逐项核对。
3. 单 relay 强中心：自托管虽开放，但架构无 P2P/复制（`ARCHITECTURE.md` §1），relay 即单点——与"Nostr 去中心化"的直觉有落差；多 relay mesh 尚在早期（iroh 1.0-rc 依赖）。
4. 超快扩张下的维护压力：8.5 个月 64 万行 Rust + 近万 PR，issue 积压近 3.9k；relay 主 crate 21.6 万行、单文件 event.rs 4472 行，长期可维护性待观察。

## 10. 总结评价

Buzz 是目前"协议层最认真"的 AI agent 团队协作平台：它没有把 agent 当外挂机器人，而是借 Nostr 的签名事件模型让人与 agent 共享同一身份、同一审计与同一操作面，并用一个纪律严明的 34-crate Rust 单体把它实现出来。它不赌 P2P 玄学（明确 relay 中心化），也不赌区块链（README 自我澄清），赌的是"一个事件日志替代七个互不相识的标签页"。8.5 个月 37.5k 星固然有大厂与创始人光环成分，但代码质量、文档诚实度（专列"verified gaps"）与工程节奏（日均约 37 PR）都配得上关注度。适合：想自托管"人 + agent 共 workspace"的工程团队、研究 agent 身份/审计/协议设计的开发者、以及想学习大规模 Rust 服务端工程组织的工程师；想读源码建议从 `crates/buzz-core/src/kind.rs` 与 `crates/buzz-relay/src/handlers/event.rs` 入手，再对照 `ARCHITECTURE.md` 逐节验证。
