# tinyhumansai/openhuman 项目分析

> 星数 42683（快照 2026-11-14，gh api 核实）｜主语言 Rust｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：OpenHuman 是一个"库优先"的 Rust agent harness（代理运行框架），同时以桌面应用形态交付普通用户，以 Rust 库形态交付开发者；核心卖点是单进程内高密度托管大量 agent——"500 agents on a $10 VPS"。

| 项 | 值 |
| --- | --- |
| 名称 | tinyhumansai/openhuman |
| 星数 | 42683（2026-11-14，gh api） |
| 主语言 | Rust（工作区 edition 2021，工具链锁定 1.99.1，见 `rust-toolchain.toml`） |
| 许可证 | GPL-3.0-only（`Cargo.toml` `[workspace.package].license` 与根目录 `LICENSE`） |
| 创建时间 | 2026-02-18（gh api）；最近 push 2026-11-13（分析前一天，仍在活跃开发） |
| 当前版本 | workspace v0.64.20（根 `Cargo.toml`） |
| 规模 | 约 4313 个 .rs 文件（其中约 2065 个为测试文件）；`Cargo.lock` 1145 个包 |
| 形态 | Tauri 桌面应用 + Rust 库（`openhuman-embed`）+ CLI/TUI + headless 服务器 |
| 官方基准 | 独立仓库 github.com/tinyhumansai/openhuman-benchmarks（README 徽章与 `docs/gitbooks/en/developing/performance.md` 引用） |

## 2. 项目是什么，解决什么问题

OpenHuman 定位为"the fastest, cheapest, most efficient open-source agent harness"（`README.md`），对标 Claude Code / Codex / OpenClaw / Hermes Agent 等单 agent harness。它解决的核心问题是 agent 的运行成本与规模：现有 harness 大多"每个 agent 一个重进程、每次调用重发大 prompt"，跑不了几百个并发 agent；OpenHuman 把整个 harness 编译为单进程 Rust 库，一个进程内可承载数百个 agent，每个 agent 可有独立的模型、工具、记忆与沙箱（`README.md` "For developers"、`docs/gitbooks/en/developing/performance.md`）。

产品形态是双轨的：对普通用户是完整桌面应用（Tauri 壳 + Web 前端），内置记忆、语音、浏览器/桌面控制、14 种消息渠道、119 个 OAuth 集成（Composio）、工作流、MCP/skills、审批门与沙箱（`README.md` "For users" 表格）；对开发者是类型化 Rust 库 API——`Harness::builder().provider(...).workspace(...).access(...).build()` 即可把 agent 当函数调用（`README.md` 示例代码，实现在 `crates/openhuman-embed/src/harness/`）。

降本的另一半在 token 侧。工具输出在进入模型前经过 TokenJuice 压缩路由（基于 Recursive Language Models 论文，arXiv 2512.24601）：按 JSON/Diff/HTML/Search/Code/Log/PlainText 七类内容选择专用压缩器，有损压缩的大结果卸载到 CCR 缓存并追加 `⟦tj:<hash>⟧` 恢复标记，保证"nothing is thrown away"（`docs/gitbooks/en/features/token-compression.md`）。官方基准称每次任务比中位数少用 2.6 倍 token（`README.md` Benchmarks 表）。

工具发现由 Jev 承担：一个小模型在 1215 个工具中选出正确的，top 命中率 86.8%（关键词搜索为 70.5%）（`README.md` "Major innovations"）。所有功能（搜索、文档、语音等）通过统一 Rust bus 以可加载 cdylib 模块方式按需挂载，编译期注册 11 个模块名（tinycomputer、tinysearch、tinydocs、tinywallet、tinyjuice、tinyvoice、tinymcp、tinyconnectors、tinybox、tinychannels、tinyhosts），带 ABI/manifest/依赖/digest 准入检查（`docs/gitbooks/en/developing/performance.md` 与 `docs/gitbooks/en/developing/loadable-modules.md`）。

增长背景（客观陈述）：仓库 2026-02-18 创建，9 个月到 42.7k 星；README 自述上线一周内连续 9 天居 GitHub 趋势榜第一，并带 Trendshift 与 Product Hunt top-post 徽章（`README.md` 第 47-48 行及徽章区）。浏览器/桌面控制、语音、多渠道等能力使它同时吸引普通用户与开发者两类人群。

## 3. 技术栈与依赖

- 语言/运行时：Rust（workspace resolver 2，7 个成员 crate；async 基于 Tokio，`linux_fleet` 示例显式 `worker_threads(2)`，见 `crates/openhuman-embed/examples/linux_fleet.rs`）。
- 桌面壳：Tauri（`crates/openhuman-app` 为独立 Cargo 世界，刻意排除出主 workspace 以避免 GTK/WebKit/Tauri 进入根构建，见根 `Cargo.toml` 注释）。
- 前端/打包：pnpm workspace（`pnpm-workspace.yaml`）、React/TS 桌面前端（`app/`，含 Playwright e2e 配置）、`packages/` 下 deb/arch/homebrew/npm 打包。
- 后端传输：JSON-RPC/Socket.IO 服务器与 axum HTTP（`crates/openhuman-rpc`、`crates/openhuman-core/src/core/`），含 OpenAI 兼容 `/v1` 路由（`crates/openhuman-core/Cargo.toml` features 注释）。
- 存储：SQLite（rusqlite 0.40，`rust-toolchain.toml` 注释解释了为何需要 Rust >= 1.96）。
- 模块体系：约 22 个 vendored 子系统作为 git submodule（`vendor/` 下 tinyagents、tinybus、tinystoragedrivers、tinyjuice、tinymcp、tinymemory 等），通过 `[patch]` 表把多个 git 源统一到单一 Rust 类型标识（根 `Cargo.toml` 50-134 行）。注意：tarball 中这些 submodule 目录为空，内容需 `git submodule update --init --recursive`（`README.md` 贡献说明）；agent loop 实体（tinyagents-harness 等）在子模块里。
- 测试：cargo nextest（每测试独立进程，根 `Cargo.toml` 157-163 行注释）、wiremock（示例内 loopback mock）、Playwright（UI e2e）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `crates/openhuman-core` | 业务核心库（包名 `openhuman`）：agent、记忆、工具、渠道、推理、沙箱、安全、JSON-RPC 分发等约 40 个领域模块（`crates/openhuman-core/src/lib.rs` 列表） |
| `crates/openhuman-embed` | 给第三方宿主的类型化库门面：`Harness`/`Runtime`/`AgentSpec`、Access/Provider/Workspace 输入类型、约 26 个示例 |
| `crates/openhuman-rpc` | JSON-RPC 信封、客户端/服务器、磁盘 session store |
| `crates/openhuman-tinyhumans` | SDK 后端传输 + 登录/会话归属（根 `Cargo.toml` 注释） |
| `crates/openhuman-cli` / `openhuman-tui` | CLI 二进制与终端客户端 |
| `crates/openhuman-app` | Tauri 桌面壳（独立 workspace） |
| `vendor/` | 22 个 vendored 子系统 git submodule（harness loop、bus、存储驱动、压缩器等） |
| `app/` | 桌面前端（React/TS） |
| `docs/` | gitbook 文档源、架构/性能/嵌入文档、`docs/plans/` 迁移计划与 `docs/specs/` |
| `examples/`、`tests/`、`e2e/` | 根集成测试（约 20+ 个 `*_e2e.rs`）、Docker 化 e2e 环境 |
| `scripts/` | CI 脚本（crate 链检查、产品 features 清单、依赖地板棘轮 kernel-floor.sh）、安装脚本 |
| `worktrees/` | 排除出 workspace 的临时工作树（根 `Cargo.toml` exclude） |

组织评述：这是"多 crate workspace + 大量 vendored submodule"的强分层单体。crate 依赖链严格单向（core -> embed -> tinyhumans -> rpc -> app/cli/tui），由 `scripts/ci/check-crate-chain.mjs` 在 CI 强制（根 `Cargo.toml` 注释）。核心思想是"host 只需点名 openhuman-rpc/embed 一个 crate，harness 能力由 vendored crate 组合供给"。`crates/openhuman-core/src/agent/README.md` 自述其代码多为"把只有 OpenHuman 知道的东西交给 TinyAgents 的适配器"——通用 agent loop 被有意上移到可复用的 vendored 库。

## 5. 核心架构与关键实现

### 5.1 Agent 回合的两条路径与分层（`crates/openhuman-core/src/agent/README.md`）

所有调用方（web_chat、渠道、cron、flows、MCP、语音、`agent.chat` RPC）经两条路进入模型回合：有态路径 `OpenHumanSessionHost`（每会话一份，从 transcript 恢复），无态路径 `agent.run_turn` 原生 bus 处理器（调用方自带历史，见 `crates/openhuman-core/src/agent/bus.rs`）。回合的执行分层为：

```text
OpenHumanSessionHost -> OpenHumanSessionDriver::execute
  -> TurnSpec::run_hosted（agent/tinyagents 适配缝）
  -> tinyagents-runtime::Session::turn
  -> AgentHarness loop（vendor/tinyagents 子模块）
       model call -> tool calls -> middleware -> results
  -> OpenHumanTranscriptCodec 落盘 commit -> after_commit（记忆/BUS）
```

宿主不实现 loop 本身，只通过 hook 注入 prompt、记忆包、工具与策略；事件经 `OpenhumanEventBridge` 桥接为 AgentProgress。子 agent 委派由 `agent/orchestration/`（`running_subagents.rs`、`background_completions.rs`、`spawn_parallel/`）承接，`spawn_subagent` 是常异步工具，父 agent 可用 `task_key` 续谈已完成 worker（`crates/openhuman-core/src/agent/orchestration/README.md`）。

### 5.2 库门面与 harness 抽象（`crates/openhuman-embed/src/harness/README.md`、`builder.rs`）

`Harness` 是"一个 Runtime + 恰好一个 id 为 harness 的 Agent"的一步式门面；需要多 agent 时直接用 `Runtime::agent(AgentSpec)` 添加（示例 `two_agents.rs`、`linux_fleet.rs` 即一 Runtime 挂 N 个 AgentSpec）。关键安全设计是 `Access` 类型同时携带两个独立机制——自治级别（驱动 SecurityPolicy）与回合来源（审批门读取的 task-local `AgentTurnOrigin`），`Access::readonly()/supervised()/full()` 三档预置防止"只设级别导致模型在拒绝叙述中打转"的陷阱（`crates/openhuman-embed/src/harness/README.md` Access 表）。`Harness::core()` 刻意只暴露 `config()/auth()`，防止宿主绕过 provider 路由与访问级别发起回合。

### 5.3 Fleet 密度：单进程多 agent 的资源模型（`docs/gitbooks/en/developing/performance.md`、`crates/openhuman-embed/examples/linux_fleet.rs`）

密度来自"固定引导成本（分配器预热、代码分页、注册表）每进程只付一次，进程内所有 agent 共享"这一决策。两次实测：

- macOS/Apple Silicon + mock 推理（200ms 固定延迟，`rss-bench` feature）：500 agent 边际 RSS 1770 KiB/agent（约 1.73 MiB），空闲 CPU 恒定 3ms/10s，线程每 agent 增约 0.35（`performance.md` 第一张表）。
- Linux x86_64、真实 cgroup 隔离（memory.max=2GiB、cpu.max=2 vCPU、release 版 `linux_fleet` 示例、loopback HTTP mock、两个 Tokio worker）：500 并发 agent 边际 RSS 3.355 MiB/agent、cgroup 峰值 1760 MiB、fleet 总墙钟 20.8s、回合 p95 19.5s（`performance.md` "Linux runtime-owned agents (Medulla integration)" 表，2026-10-10 测于 commit b783b39e78）。关 swap 的对照跑为 3.541 MiB/agent、峰值 1884 MiB。

文档同时明确写出测量局限：mock 推理测的是编排开销而非真实模型延迟；macOS RSS 高估共享页；"Thousands of agents on one box is a goal, not a measured result"；复现脚本在独立的 openhuman-benchmarks 仓库 `profile/scripts/`（含 `library-fleet.sh --budget-mib 2048` 的 2GB 预算门）。另有 `library-instances.sh` 对照：每 agent 独立进程固定 47.8-48.2 MiB，同内存预算下单进程模型密度约为每进程模型的 25 倍。

### 5.4 Token 压缩管线（`docs/gitbooks/en/features/token-compression.md`、`vendor/tinyjuice/`）

工具结果统一过 TokenJuice 路由：2KB 尺寸门 -> 七类内容检测（无热路径正则）-> 每类专用压缩器（SmartCrusher 表格化 JSON、Code 保留签名/imports/错误标记、Diff 折叠 lockfile hunk 等）-> 压缩失败/变大则回退透传 -> 有损且 >= 500 token 的原件卸载 CCR 缓存 -> 追加 `⟦tj:<hash>⟧` 恢复标记 -> 按 model/compressor 记账节省。配合 microcompact 中间件只保留最近 5 条工具结果（`crates/openhuman-core/src/agent/context/mod.rs` 的 `DEFAULT_KEEP_RECENT_TOOL_RESULTS = 5`）与 0.90 压缩阈值（`agent/tinyagents/summarize.rs`，见 `agent/context/README.md`），构成"少读 token"的完整闭环。

### 5.5 会话持久与模块挂载

会话与运行账本：TinyAgents session ledger 存 `agent_runs`/`run_events`，OpenHuman 投影为产品 AgentRun 视图；会话租约在 `agent/session_lease.rs`，transcript 恢复在 `agent/session_host/`（codec、resume、orphaned head 等均有专属测试文件）。模块挂载：11 个 cdylib 模块经 `*-bus` 契约 crate 接入统一 bus，加载走 tinybus `attach_linked` 准入（哈希宿主可执行文件，根 `Cargo.toml` 157-163 行注释解释了为何为 sha2 单独开优化）。

## 6. 构建与运行

来自 `README.md` 贡献节与 `INSTALL.md`/`docs/gitbooks/en/developing/`：

```bash
# 开发环境依赖：Git、Node.js 24+、pnpm 10.10.0、Rust 1.99.1、CMake、Ninja、ripgrep
git submodule update --init --recursive && pnpm install
pnpm dev            # UI 开发
pnpm dev:app        # 桌面应用
pnpm typecheck && pnpm format:check && cargo check --manifest-path Cargo.toml

# 桌面安装（终端一键）
curl -fsSL https://raw.githubusercontent.com/tinyhumansai/openhuman/main/scripts/install.sh | bash

# 嵌入库（无头宿主最小配方）
cargo build --no-default-features --features "skills,flows"   # library-minimal
# 纯精简：--no-default-features（features 全表见 crates/openhuman-core/Cargo.toml [features]）

# fleet 实测（在 openhuman-benchmarks 仓库）
cargo run --release --example linux_fleet -- 500   # （示例位于 crates/openhuman-embed/examples/linux_fleet.rs）
./profile/scripts/library-fleet.sh --agents "50,100,500" --target 1000 --budget-mib 2048
```

## 7. 测试与工程质量

- 测试规模：约 4313 个 .rs 文件中约 2065 个为测试文件（`*_tests.rs` 命名约定内嵌于源码树）+ 根 `tests/` 下约 20 个集成/e2e 文件（`agent_harness_e2e.rs`、`cli_agent_chat_headless_e2e.rs`、`cost_report_e2e.rs` 等）+ Playwright UI e2e（`app/playwright.config.ts`）+ Docker 化 e2e（`e2e/docker-compose.yml`）。风格以同目录并列测试文件为主，覆盖面极广（会话恢复、编排取消、租约、观测错误分类等均有专测）。
- CI：`.github/workflows/` 下 26 个工作流——ci-gate/ci-full/ci-lanes 分层、e2e-agent-review/e2e-playwright、security-browser-egress/security-native、release-packages/release-production、pr-quality 等。nextest 每测试独立进程（根 `Cargo.toml` 注释连带解释了 sha2 的 dev 优化）。
- 工程纪律：依赖地板"棘轮"（`scripts/kernel-floor.sh` + `kernel-floor.limits`，依赖图只许降不许升，升需书面理由，`performance.md`）；crate 依赖链 CI 强检（`scripts/ci/check-crate-chain.mjs`）；产品 features 清单固定于 `scripts/ci/product-features.txt`；release profile 精细控制 DWARF/strip（根 `Cargo.toml` [profile.release]）。
- 观察到的高标准信号：错误分类观测有 20+ 个专项测试文件（`crates/openhuman-core/src/core/`）；agent 模块 README 对"已移除文件的职责去向"逐一交接（`agent/context/README.md`），显示持续的架构收敛。

## 8. 社区与维护状态

- 活跃度：最近 push 2026-11-13（分析前一天）；workspace 版本 0.64.20，release 节奏高频——v0.64.4（09-26）、v0.64.7（09-29）、v0.64.10（09-30）、v0.64.14（10-08）、v0.64.15（10-09），近乎每周多发（gh api releases）。
- 贡献者：分页查询上限 100（anon=true 计 100），实际 100+（未确认精确总数）。354 个 open issues（gh api）。
- 社区运营：Discord guild、Reddit r/tinyhumansai、X、Discussions；贡献者有周边与 Discord 特权激励（`README.md` Contributing）；README 有 7 语言版本（`docs/README.*.md`）；根目录 `llms.txt`/`llms-full.txt` 面向 AI agent 的文档索引；`CITATION.cff` 学术引用文件。
- 争议性信号：README 含强营销语气（"number one trending … nine days in a row"）与自办基准；文档将方法学与复现脚本公开在独立仓库以对冲（见第 9 章）。

## 9. 亮点与风险

亮点：

1. 库优先的 fleet 架构经真实测量：500 agent 边际 RSS 1.73-3.36 MiB/agent，2 GiB cgroup 实测放得下 500 并发 agent，且测量文档罕见地诚实列出全部局限（mock 推理、共享宿主、RSS 口径）与复现脚本（`docs/gitbooks/en/developing/performance.md`）。
2. Token 成本工程化：TokenJuice 七类压缩器 + CCR 可恢复原件 + microcompact + 记忆召回预算，构成完整"少读 token"管线（`docs/gitbooks/en/features/token-compression.md`）。
3. 工程纪律强：依赖地板棘轮、crate 链 CI 检查、feature 门控最小构建（纯精简 51 MiB stripped 二进制）、26 个 CI 工作流、约半数文件为测试。
4. harness 抽象干净：`Harness`/`Runtime`/`AgentSpec`/`Access` 类型化 API，审批门 fail-closed 设计（自治级别与回合来源双机制），有 26 个官方嵌入示例含 cgroup fleet 实测脚本（`crates/openhuman-embed/examples/`）。
5. 通用 agent loop 上沉到 vendored `tinyagents-*` 库，宿主侧只剩适配器，边界由 `docs/specs/agent-runtime-upstream-boundary.md` 与迁移计划（`docs/plans/migrate-agent-runtime-to-tinyagents.md`，状态 live）持续收敛。

风险/局限：

1. 构建复杂度高：约 22 个 vendored submodule（含嵌套 vendoring），大量 `[patch]` 表统一 git 源；tarball 中 vendor 目录为空，贡献者必须递归初始化子模块（`README.md`、根 `Cargo.toml`）。
2. 基准数字自测自报：SWE-bench 对比跑由项目方在其自己的 benchmarks 仓库执行（10 个任务的小样本），"fastest/cheapest"结论依赖该单一来源；`performance.md` 明确声明数字不可直接外推到真实网络推理。
3. fleet 声明的边界条件：500-agent 实测关闭了会话双写、本地模型与记忆，且 mock 推理无真实 token 状态；真实负载下每 agent 的 transcript/记忆存储会推高边际成本（`performance.md` 测量条件节）。
4. GPL-3.0-only 对商用嵌入比竞品（Codex Apache-2.0、OpenClaw/Hermes MIT）更严格；项目自评 early beta（README 徽章），API 面在快速演化（v0.64.x 高频发版）。

## 10. 总结评价

OpenHuman 是 2026 年 agent harness 赛道中工程密度最高的开源实现之一：它把"跑 agent"从每进程一个实例的粗放模型改造成单进程共享运行时的库模型，并用 token 压缩、工具检索与小系统提示三条路径压低每次调用的成本；其 fleet 数字有公开方法学与 cgroup 级实测背书，测量文档的诚实程度在同类项目中少见。9 个月 42.7k 星的增长与其"桌面应用即产品、Rust 库即平台"的双轨定位相符——普通用户拿到的是对标 Claude Code 的全功能桌面 agent，开发者拿到的是能把 agent 当函数调用的嵌入运行时。适合三类人：想把 agent 能力嵌进自己产品的 Rust 开发者（读 `crates/openhuman-embed` 与 examples）；研究 agent 运行时资源效率的工程团队（读 `docs/gitbooks/en/developing/performance.md` 并复现 fleet 脚本）；以及评估 agent harness 选型者（先读 openhuman-benchmarks 的方法学再采信数字）。阅读源码的门槛在于庞大的 vendored 体系与 feature 门控，建议从 `crates/openhuman-core/src/agent/README.md` 的分层图进入。
