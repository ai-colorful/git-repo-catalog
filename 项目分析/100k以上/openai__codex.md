# openai/codex 项目分析

> 星数 130596（快照 2026-10-24）｜主语言 Rust｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：OpenAI 官方出品的本地终端编程 agent（Codex CLI），以 Rust 实现核心引擎，可在终端/IDE/桌面端以交互或无人值守方式自主读写代码、执行命令并沙箱化运行。

| 项目 | 值 |
| --- | --- |
| 名称 | openai/codex |
| 星数 | 131,570（分析时 gh api 实测；快照 130,596） |
| Fork | 20,785 |
| 主语言 | Rust（workspace 132 个 crate，约 5,679 个 .rs 文件） |
| 许可证 | Apache-2.0（仓库 LICENSE 与 `codex-rs/Cargo.toml` 均为 Apache-2.0） |
| 创建时间 | 2025-04-13（gh api） |
| 最近 push | 2026-10-29（gh api） |
| 最新 release | rust-v0.166.0（2026-10-29） |
| 默认分支 | main |
| 贡献者 | 约 478 人（contributors API 分页推算） |

## 2. 项目是什么，解决什么问题

Codex 是 OpenAI 的开源编程 agent，README 开篇即定位"Codex CLI is a coding agent from OpenAI that runs locally on your computer"（`README.md`）。它把一个完整的 agent 运行时装进本地终端：用户用自然语言下达任务，agent 规划、调用工具（执行 shell、改文件、搜索网页、连接 MCP 服务）、循环执行直到完成，并可随时请求用户审批敏感操作。

与"补全式"工具不同，Codex 解决的是**任务级自治 + 本地安全执行**的组合问题。其核心卖点是三层防线：模型侧审批策略（approval policy）、进程侧操作系统级沙箱（macOS Seatbelt、Linux Landlock/Seccomp/bubblewrap、Windows 受限令牌，见 `codex-rs/sandboxing/`）、以及网络代理层（`codex-rs/network-proxy` crate）。`docs/sandbox.md` 明确把安全模型指向官方文档，仓库内则落地了全部平台实现。

产品形态覆盖面很广：交互式 TUI（`codex`）、非交互式（`codex exec`，JSONL 输出，见 `codex-rs/exec/src/`）、IDE 集成用的 app-server（`codex-rs/app-server`，VS Code/Cursor 扩展后端）、桌面 App（`codex app`）、以及 TypeScript/Python SDK（`sdk/typescript`、`sdk/python`）。同一 Rust 核心以不同前端复用。

生态方面它是 OpenAI 多 agent 战略的本地端载体：支持子 agent（`codex-rs/core/src/agent/`）、MCP 客户端/服务端（`codex-rs/rmcp-client`、`codex-mcp`）、插件与技能体系（`codex-rs/plugin*`、`codex-rs/skills`）、云任务（`codex-rs/cloud-tasks`）。认证走 ChatGPT 账号（Plus/Pro/Business 计划）或 API key（`README.md`、`docs/authentication.md`）。

## 3. 技术栈与依赖

- **语言**：Rust（edition 2024，`codex-rs/Cargo.toml`），核心 workspace 含 132 个成员 crate。
- **异步运行时**：tokio（全库统一，`Cargo.toml` workspace dependencies）；HTTP 用 reqwest/hyper。
- **TUI**：ratatui 0.30 + crossterm 0.29（`codex-rs/Cargo.toml` L380、L453；`codex-rs/tui/Cargo.toml`）。
- **MCP**：rmcp 3.5.1，固定官方 modelcontextprotocol/rust-sdk 的 git rev（`codex-rs/Cargo.toml` L468）。
- **序列化**：serde/serde_json/serde_yaml 全家桶；配置分层加载（`codex-rs/config` crate）。
- **工程化**：Bazel（`MODULE.bazel`、`BUILD.bazel` 遍布）与 Cargo 双构建体系；just 任务入口（`justfile`）；pnpm 管理 JS 工具链（`package.json`，engines 要求 node>=24）；prettier 做 Markdown/JSON 格式检查。
- **Python/TS**：`sdk/python` 有独立的 build/release workflow（`.github/workflows/python-sdk-*.yml`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `codex-rs/` | Rust 主 workspace，132 个 crate，约 5,679 个 .rs 文件（实测统计） |
| `codex-rs/core/` | agent 引擎：session/turn 循环、模型客户端、工具编排、压缩（21MB 源码，613 个文件） |
| `codex-rs/tui/` | 交互终端 UI，ratatui 实现（25MB，1,167 个文件） |
| `codex-rs/cli/` | `codex` 二进制入口与子命令分发（`cli/src/main.rs`） |
| `codex-rs/exec/` | 非交互模式，事件转 JSONL/人类可读输出 |
| `codex-rs/app-server/` | 面向 IDE 的进程服务器与协议（8.3MB） |
| `codex-rs/sandboxing/` | 跨平台沙箱：Seatbelt/Landlock/Seccomp/bwrap/Windows |
| `codex-rs/tools/` | 工具定义层：ToolSpec、schema、tool_search、code-mode |
| `codex-rs/execpolicy/`、`network-proxy/` | 命令策略检查与网络代理 |
| `codex-rs/agent-*`、`ext/` | 子 agent、消息板、guardian 审查等扩展 crate 群 |
| `codex-cli/` | npm 包 `@openai/codex` 的薄壳安装器（下载平台二进制，`codex-cli/package.json`） |
| `sdk/` | TypeScript / Python SDK |
| `docs/` | 用户文档：config.md、exec.md、sandbox.md、skills.md、slash_commands.md 等 |
| `.github/` | 20+ workflows：rust-ci、bazel、cargo-deny、issue 机器人、多平台 release |

组织方式评述：典型的**巨型多 crate monorepo**。按"协议/客户端/服务端/能力"横向切分 crate（如 app-server-protocol、cloud-tasks-client、codex-backend-openapi-models），crate 间以 workspace path 依赖严格分层；`*_tests.rs` 与源文件同目录的内联测试占 1,620 个文件，另有 59 个 `tests/` 集成目录。旧版 core 代码通过 `legacy_core` 别名在 tui 中引用（`codex-rs/tui/src/lib.rs`），可见大规模重构仍在渐进推进。

## 5. 核心架构与关键实现

### 5.1 agent 核心循环：task → turn（`codex-rs/core/src/tasks/regular.rs`、`codex-rs/core/src/session/turn.rs`）

一个任务（task）由循环执行的"轮次（turn）"构成，每轮把输入喂给模型、执行工具、按需续轮。`regular.rs` L115 起的主循环：

```rust
loop {
    let last_agent_message = run_turn(
        Arc::clone(&sess), Arc::clone(&ctx), next_input,
        &mut mcp_startup_requirements,
        prewarmed_client_session.take(),
        cancellation_token.child_token(),
    ).await?;
    if ctx.terminal_error.lock().await.is_some() { return Ok(last_agent_message); }
    if !sess.input_queue.has_pending_input(&sess.active_turn).await { return Ok(last_agent_message); }
    next_input = Vec::new();
}
```

要点：轮与轮之间有**输入队列**（用户可中途 steer 插话，`session/turn_input.rs` 定义了 SteerSubmission/StartIfIdleSubmission 等协议类型）；`run_turn`（`turn.rs` L176，全文件 3,555 行）先做 pre-sampling 自动压缩（`run_pre_sampling_compact`）、drain 异步 hook 结果，再发起模型请求（`run_sampling_request` L1741）。上下文管理是一等公民：`compact*` 系列 8+ 个模块处理本地/远端（RemoteCompactionSupport::V2）压缩与 token 预算（`core/src/compact_token_budget.rs`）。

### 5.2 工具体系：ToolSpec → ToolRouter → handlers（`codex-rs/tools/`、`codex-rs/core/src/tools/`）

工具定义在 `tools/src/tool_spec.rs` L22 的 `ToolSpec` 枚举：`Function`（普通函数调用）、`Namespace`（命名空间分组）、`ToolSearch`（延迟检索工具）、`WebSearch`（OpenAI 托管搜索，带 external/indexed web access 开关）、`Freeform`（自定义 schema）。分发由 `core/src/tools/router.rs` 的 `ToolRouter::dispatch_tool_call_with_code_mode_result`（L344）统一入口，经 `orchestrator.rs`、`parallel.rs` 支持并行调用与调用追踪。内置 handler 一览（`core/src/tools/handlers/` 目录）：`apply_patch`（补丁式改文件）、`unified_exec`（exec_command/写 stdin 的统一执行器）、`plan`、`request_user_input`、`mcp`/`mcp_resource`、`multi_agents(_v2)`（子 agent 派生）、`current_time`、`get_context_remaining`、`code_mode` 等。另有 `tool_search` + `tools/src/indirect_namespace_prefixes.rs` 实现"工具太多时按需发现"的降载机制。

### 5.3 跨平台沙箱（`codex-rs/sandboxing/src/manager.rs`）

平台沙箱选择是显式的三叉分支（manager.rs L50）：

```rust
pub fn get_platform_sandbox(windows_sandbox_enabled: bool) -> Option<SandboxType> {
    if cfg!(target_os = "macos") { Some(SandboxType::MacosSeatbelt) }
    else if cfg!(target_os = "linux") { Some(SandboxType::LinuxSeccomp) }
    else if cfg!(target_os = "windows") {
        if windows_sandbox_enabled { Some(SandboxType::WindowsRestrictedToken) } else { None }
    } else { None }
}
```

macOS 用 Seatbelt 并维护多份 SBPL 策略文件（`sandboxing/src/seatbelt_base_policy.sbpl`、`seatbelt_network_policy.sbpl` 等）；Linux 是 Landlock + Seccomp 双栈，外加 bubblewrap 包装（`sandboxing/src/bwrap.rs`）与 PID namespace；Windows 用受限令牌（`windows-sandbox-rs` crate）和 MXC 沙箱服务（`windows-sandbox-service/`）。违规行为可被捕获上报（`violation.rs` 的 SandboxViolationEvent），配合 `execpolicy` crate 做命令级策略判定，`network-proxy` 做域名白名单。审批拒绝→沙箱重试的生命周期在 CHANGELOG Unreleased 条目（#44557）中也有印证。

### 5.4 TUI 与 app-server 解耦（`codex-rs/tui/`、`codex-rs/app-server/`）

TUI 不是直接调用 core，而是作为 app-server 的客户端运行：`tui/src/lib.rs` 顶部组装 `AppServerSession`，支持 InProcessAppServerClient 与 RemoteAppServerClient（可连常驻 daemon，`tui/src/daemon_startup.rs`）。UI 侧按组件域分目录：`chatwidget/`（主对话视图，`chatwidget.rs` 2,149 行）、`bottom_pane/`（输入编辑器 chat_composer、审批浮层 approval_overlay、footer、各类 popup）、`app_backtrack/`（回溯）、`custom_terminal/`。这一架构让 IDE 扩展、桌面 App 与 TUI 共享同一个协议层（`app-server-protocol`）。

## 6. 构建与运行

```bash
# 构建（Rust 核心）
cd codex-rs && cargo build --release
# 运行交互式 TUI（justfile 定义）
just codex            # = cargo run --bin codex
just exec             # = cargo run --bin codex -- exec
# 安装（终端用户）
curl -fsSL https://chatgpt.com/codex/install.sh | sh        # macOS/Linux
npm install -g @openai/codex                                 # npm 薄壳
brew install --cask codex
# 非 Rust 工具链
pnpm install && pnpm format    # JS 侧格式检查（package.json）
```

来源：`README.md`（安装）、`justfile`（cargo run 别名）、`docs/install.md`/`docs/contributing.md`。测试用 nextest（`.github/workflows/rust-ci-full-nextest-platform.yml`）。

## 7. 测试与工程质量

- **规模**：codex-rs 内 1,620 个 `*tests*.rs` 文件 + 59 个 `tests/` 目录（对约 5,679 个 .rs 文件，测试文件占比约 28%），单 crate 内如 `core/src/session/` 即有 40+ 测试模块。
- **风格**：源文件旁内联 `*_tests.rs` 为主，协议层有快照/fixture（如 `hooks` 的 write_hooks_schema_fixtures，`package.json` scripts）；支持包（`test-binary-support`、`exec-server/tests/support`）独立成 crate。
- **CI**：`.github/workflows/` 下 rust-ci.yml（PR 快检）、rust-ci-full.yml + nextest 平台矩阵（全量）、bazel.yml、cargo-deny.yml（依赖审计）、codespell、blob-size-policy 等；另有 issue-deduplicator/issue-labeler/issue-translator 等 bot，反映高强度社区流入（open issues 25,555，gh api）。
- **质量观察**：全库禁用 print_stdout/stderr（`core/src/lib.rs` L4 `#![deny(clippy::print_stdout, clippy::print_stderr)]`，TUI 同），输出必须走事件/抽象层；clippy.toml、rustfmt.toml、deny.toml 齐备。

## 8. 社区与维护状态

- **活跃度极高**：最近 push 2026-10-29（即分析前一天）；2026-10-27~29 三天内发布 0.165.0、0.166.0 及 4 个 alpha（gh api releases），固定版本号前缀 `rust-v*`。
- **发布节奏**：主干版本约每周 1-2 个 minor，alpha 通道滚动；CHANGELOG.md 顶部声明"changelog 见 releases 页"，仅保留 Unreleased 段。
- **贡献者结构**：约 478 名贡献者；CHANGELOG 中社区 PR 会被署名致谢（如 Unreleased 中的 @Marvinthebored）；仓库有 CLA.md、CODEOWNERS，issue 自动化机器人齐全。
- **商业背书**：OpenAI 官方维护，安装源 releases.openai.com（README），并有 open-source-fund（`docs/open-source-fund.md`）回馈生态。

## 9. 亮点与风险

**亮点**
1. 生产级跨平台沙箱体系：三大 OS 原生机制 + 违规捕获 + 网络代理，`codex-rs/sandboxing/` 是同类开源项目中最完整的实现之一。
2. 架构分层干净：core 引擎与 TUI/app-server/SDK 前端通过协议 crate 解耦（`app-server-protocol`），一套引擎多端复用。
3. 上下文工程深入：本地/远端两级压缩、token 预算、工具延迟发现（tool_search）等 10+ 模块专门管理长任务上下文。
4. 工程纪律严格：132 crate 按职责拆分、edition 2024、全库 lint 禁直接打印、双构建体系（Cargo+Bazel）、nextest 矩阵 CI。
5. 生态完整：MCP 双向支持、插件/技能体系、子 agent、云任务协同，agent 能力面在持续快速扩张。

**风险/局限**
1. 代码体量巨大（core 单 crate 21MB、turn.rs 3,555 行），阅读与二次开发门槛高，且存在 legacy_core 这类过渡层（`tui/src/lib.rs`），重构中间态会困扰读者。
2. 部分深度能力（远端压缩、guardian 审查、ChatGPT 计划认证）与 OpenAI 服务端耦合，自建后端/其他模型提供方时功能面收窄（`model-provider` crate 支持多 provider，但完整体验绑定官方 API）。
3. open issues 超 2.5 万，issue 治理靠机器人流水线，普通用户的问题响应不可预期。
4. 版本演进极快（周级 release、alpha 滚动），第三方集成需频繁跟进协议变化。

## 10. 总结评价

openai/codex 是当前开源编码 agent 中工程化程度最高的项目之一：以 Rust 把"模型循环 + 工具执行 + OS 级沙箱 + 多端 UI"整合成单一二进制，同时以协议层开放给 IDE、桌面与 SDK 生态。它既是日常可用的生产力工具，也是学习"如何把 LLM agent 做成可靠本地软件"的最佳范本——尤其是沙箱（`codex-rs/sandboxing/`）、工具路由（`codex-rs/core/src/tools/`）与 turn 循环（`codex-rs/core/src/session/`）三块源码。适合想构建编程 agent 的工程师精读、想在终端/CI 里落地 AI 编码的开发者使用；但若只想轻量定制或接入非 OpenAI 模型，需要先评估体量与耦合成本。
