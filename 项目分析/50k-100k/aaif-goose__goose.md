# aaif-goose/goose 项目分析

> 星数 56143（快照 2026-10-24）｜主语言 Rust｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：goose 是一个开源通用 AI agent（"桌面应用 + CLI + API" 三位一体），能装扩展、执行命令、编辑文件、跑测试，可接 15+ 家模型供应商，通过 MCP 协议连接 70+ 扩展（出自根 `README.md`）。

| 项目 | 事实 |
| --- | --- |
| 仓库 | aaif-goose/goose（现属 Agentic AI Foundation / Linux Foundation，README 与 GOVERNANCE.md 均有注明） |
| 星数 | 56143（gh api 快照） |
| 语言 | Rust（核心）+ TypeScript（UI） |
| 许可证 | Apache-2.0（根 LICENSE、Cargo.toml workspace.license） |
| 版本 | v1.57.0（Cargo.toml workspace version 与最新 release 一致） |
| 最近活跃 | 最近 push 2026-11-02，最新 release v1.57.0 发布于 2026-10-28（gh api） |
| 贡献者 | 451 人（gh api contributors 分页统计） |

## 2. 项目是什么，解决什么问题

goose 起源于 Block（Square/Cash App 母公司）开源的 AI 编程 agent，现已捐入 Linux Foundation 旗下的 Agentic AI Foundation（AAIF），成为中立治理的开源项目（`GOVERNANCE.md`、`README.md`）。它解决的不是一个"代码补全"问题，而是"让 LLM 真正在你机器上干活"的问题：安装依赖、执行/编辑/测试代码、操作文件系统、调用外部工具，覆盖研究、写作、自动化、数据分析等通用任务（`README.md`）。

与 IDE 插件形态的产品不同，goose 把 agent 做成了一个可嵌入的运行时：同一个 Rust 核心（`crates/goose`）向上支撑四种使用面——终端 CLI（`crates/goose-cli`）、Electron 桌面应用（`ui/desktop`）、ACP 协议服务端（stdio 与 HTTP/WebSocket，`crates/goose/src/acp`），以及带 uniffi 绑定的 SDK（`crates/goose-sdk`，官方称 GDK）。用户可以在命令行里 `goose` 一把梭，也可以在桌面 App 里用，还能把自己的应用通过 SDK/ACP 接到同一个 agent 内核上。

扩展体系是其核心卖点：一切能力皆"extension"，统一走 Model Context Protocol（MCP）标准。项目内置了一批 MCP server（memory、computer controller、tutorial 等，`crates/goose-mcp/src/lib.rs`），同时支持用户以 stdio 命令、Streamable HTTP、平台内置等多种方式挂接外部 MCP 服务，README 宣称兼容 70+ 扩展。

供应商侧做得很宽：`crates/goose/src/providers/` 下有 40+ 个 provider 实现/适配文件，除 Anthropic、OpenAI、Google、Azure、Bedrock、Ollama、OpenRouter 等标准 API 外，还通过 ACP（Agent Client Protocol）接入 Claude Code、Codex、Gemini CLI、GitHub Copilot 等"订阅制"CLI 后端（`claude_acp.rs`、`codex_acp.rs`、`gemini_cli.rs`、`copilot_acp.rs` 等），甚至包含基于 candle 的本地推理（`crates/goose-local-inference`）。

## 3. 技术栈与依赖

- **核心语言**：Rust（edition 2021，rust-version 1.94.1，workspace 共 15 个 crate，根 `Cargo.toml`）。
- **异步运行时**：tokio + tokio-util（CancellationToken）、futures、async-trait。
- **MCP 实现**：rmcp 3.4.1（官方 Rust MCP SDK，workspace 依赖）；Agent Client Protocol 相关：agent-client-protocol 2.2.0。
- **HTTP 服务**：axum 0.8 + axum-server（ACP over HTTP/WS、gateway）、reqwest 0.13、rustls。
- **可观测性**：tracing 全家桶 + OpenTelemetry 0.32（otlp 导出，`crates/goose/src/otel/`）+ PostHog（`posthog.rs`）。
- **本地推理**：candle-core/candle-nn 0.11（`crates/goose-local-inference`）；tree-sitter（多语言解析，workspace 依赖，供代码工具使用）。
- **P2P**：iroh 1.2（`goose roam` 命令与 `crates/goose-roaming`，点对点共享 agent）。
- **SDK 绑定**：uniffi（`crates/goose-sdk/Cargo.toml`，生成 Python/Kotlin 绑定）。
- **桌面端**：Electron + electron-forge + vite + React（`ui/desktop/package.json`），Playwright e2e + vitest；pnpm workspace（`ui/pnpm-workspace.yaml`）。
- **构建/质量工具**：Justfile、clippy（workspace lints）、cargo-deny、cargo-machete、release-plz（`release-plz.toml`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `crates/goose` | 核心库：agent 状态机、扩展管理、MCP 客户端、session、ACP 服务端、providers、recipe/skills、gateway 等 |
| `crates/goose-agent` | 纯粹的 agent 状态机引擎（machine/operation/effect 抽象，可独立测试，含 wasm32 支持） |
| `crates/goose-cli` | CLI 入口（clap），configure/doctor/serve/acp/roam/session/schedule/recipe/skills 等子命令 |
| `crates/goose-sdk` | GDK（Goose Development Kit），uniffi 绑定生成 Python/Kotlin SDK |
| `crates/goose-provider-types` | 会话/消息/GooseMode 等跨 crate 共享类型 |
| `crates/goose-providers` | provider 基础设施（声明式 provider、api_client、决策逻辑） |
| `crates/goose-mcp` | 内置 MCP server：memory、computercontroller、tutorial、autovisualiser、peekaboo(macOS) |
| `crates/goose-context-management` / `goose-download-manager` / `goose-local-inference` / `goose-roaming` / `goose-acp-macros` / `goose-test*` | 上下文压缩、下载管理、candle 本地推理、iroh P2P、宏、测试设施 |
| `ui/desktop` | Electron 桌面应用（React + vite，内含 `built-in-extensions.json` 扩展清单） |
| `ui/goose-acp` / `goose-acp-client` / `goose-binary` | npm 分发层：安装/解析 goose 可执行文件、ACP 客户端库 |
| `documentation/` | docusaurus 文档站源码 |
| `buzz/` | Node 脚本（GitHub 自动化，配合 `buzz-automation.yml` workflow） |
| `evals/` / `examples/` / `workflow_recipes/` / `services/` / `oidc-proxy/` | 评测、示例、recipe 配方、辅助服务 |
| `scripts/`、`.github/workflows/` | 辅助脚本；39 个 CI workflow |

组织方式评述：典型的"Rust workspace 多 crate + pnpm workspace 多包"混合 monorepo。分层清晰——`goose-agent` 只定义引擎抽象、不依赖具体扩展，`goose` crate 做胶水与业务，`goose-cli`/`ui` 是薄壳。约 600 个 Rust 文件（本仓统计），核心 crate 单文件偏大（`services.rs` 2980 行、`extension_manager/mod.rs` 2732 行）。

## 5. 核心架构与关键实现

### 5.1 Agent 状态机引擎（goose-agent）

引擎被刻意做成了一个通用库（`crates/goose-agent/src/machine.rs`）：一轮对话由一组 `Step` 组成，`Step` 要么是 `Operation`（工具执行、审批、压缩等），要么是 `Inference`（LLM 调用）。`step()` 按序尝试每个步骤：未命中的返回 `NotApplicable` 继续向下，命中的返回 effects 列表交给 `EffectHandler` 应用，全部未命中则 `Yielded`。取消通过 `CancellationToken` 实现 `tokio::select!` 竞速，中断时为所有未应答的 tool request 合成错误响应，保证会话状态一致：

```rust
// crates/goose-agent/src/machine.rs
let result = tokio::select! {
    biased;
    result = execution => Some(result),
    _ = self.cancel.cancelled() => None,
};
...
_ => {
    *self.interrupted_step.lock().unwrap() = Some(index);
    return Ok(empty_result(RunStatus::Cancelled));
}
```

### 5.2 主循环的操作链（goose crate）

真正决定 goose 行为的是 `crates/goose/src/agents/state_machine_builder.rs` 里 `build_state_machine()` 拼装的操作链，每轮按优先级依次尝试：`ToolApproval`（敏感工具审批）→ `Skill` → `Recipe` → `ToolExecution`（执行待处理工具调用）→ `RemainingToolCalls` → `Subagent`（并发上限默认 5）→ `Steer`（转向插话）→ `BangShell`（!shell 直通）→ 上下文压缩 `Compaction`/`ToolPairCompaction`（若 provider 不自管上下文）→ `Doctor`/`Project`/`Retry`/`EmptyResponse`/`StopHook`/`ExitOnError`/`MaxTurns`，最后是 `InferenceRunner` 兜底调用 LLM。这本质是一个"事件优先于推理"的循环：会话里有待办（工具结果、审批、转向消息）就先处理，都处理完才发起新一轮推理。

对外入口在 `crates/goose/src/agents/services.rs`（2980 行）的 `StateMachineServices::reply()`：接收用户消息 → 处理 elicitation 应答 → 获取 turn guard（防并发）→ 持久化消息 → 异步生成会话命名 → 返回 `BoxStream<AgentEvent>` 事件流。权限模式定义在 `crates/goose-provider-types/src/goose_mode.rs`：`Auto / Approve / SmartApprove / Chat` 四档，从全自动到纯聊天。

### 5.3 扩展（MCP）体系

扩展配置是 serde tagged 枚举 `ExtensionConfig`（`crates/goose/src/agents/extension.rs`），共四类：

- `stdio`：以子进程方式拉起 MCP server（cmd/args/envs）；
- `builtin`：进程内直连 goose 自带的 MCP server（`crates/goose-mcp/src/lib.rs` 的 `BUILTIN_EXTENSIONS` 注册表，经 DuplexStream 内存管道通信，避免进程开销）；
- `platform`：直接跑在 agent 进程内的平台扩展（`crates/goose/src/agents/platform_extensions/`：developer（edit/shell/tree，见其 `shell.rs`/`edit.rs`）、todo、summarize、scheduler、code_execution、javascript 等），拥有直接访问 agent 的特权；
- `streamable_http`：远程 HTTP MCP server。

`ExtensionManager`（`extension_manager/mod.rs`，2732 行）负责配置解析、启用/禁用、工具清单汇聚；每轮通过 `ExtensionLease`（`extension_manager/lease.rs`，879 行）按会话"租约"锁定扩展集合，工具重名检测、`get_tool_owner` 归属、`extension_malware_check.rs` 安全扫描都在这一层。审批链路由 `tool_confirmation_coordinator.rs` 与 `ToolApprovalOperation` 协同完成。

### 5.4 CLI / Desktop / API 三层分发

- CLI（`crates/goose-cli/src/cli.rs`）：`goose acp` 把 goose 变成 stdio 上的 ACP agent 服务器；`goose serve` 提供 HTTP/WebSocket ACP 服务（默认 127.0.0.1:3284，支持 TLS 与 `GOOSE_SERVER__SECRET_KEY` 认证）；另有 `mcp`（单独跑内置 server）、`roam`（iroh P2P）、`schedule`、`recipe` 等子命令。
- 桌面端（`ui/desktop`）：Electron 主进程通过 `gooseServe.ts`/`gooseServeLeaseRegistry.ts` 拉起并管理 goose 二进制（ACP stdio），渲染层为 React 聊天界面（`src/components/`），`built-in-extensions.json` 提供可一键安装的扩展清单。
- API/SDK（`crates/goose-sdk`）：导出 GDK，经 uniffi 产出 Python/Kotlin 绑定（CI 有 `python-sdk-wheels.yml`、`maven-sdk.yml` 专门发 wheels/Maven）。
- 其他接入面：`crates/goose/src/gateway/`（含 `telegram.rs`，Telegram bot 网关）、`crates/goose-roaming`（iroh P2P 远程连接）。

## 6. 构建与运行

来自根 `Justfile`、`README.md` 与 `ui/desktop/package.json`：

```bash
# CLI 发布构建
cargo build --release -p goose-cli --bin goose      # Justfile release-binary
# 风格检查（fmt + clippy + UI lint）
just check-everything
# Rust 全量测试
cargo test --all                                    # CI: rust-build-and-test job
# 桌面端
cd ui && pnpm install
cd desktop && pnpm run start-gui                    # electron-forge start
pnpm run test-e2e                                   # Playwright
# ACP 服务
goose serve --host 127.0.0.1 --port 3284            # HTTP/WS ACP
goose acp --with-builtin developer,tutorial          # stdio ACP
```

## 7. 测试与工程质量

- 规模：Rust 侧约 4070 处 `#[test]/#[tokio::test]`（grep 统计），600 个 .rs 文件；UI 侧 120 个测试文件（vitest 单测 + Playwright e2e）。状态机、压缩生命周期、取消语义都有专门测试目录（`crates/goose/src/agents/state_machine/tests/`）。
- CI（`.github/workflows/ci.yml`）按路径变更分发的 25+ 个 job：多平台构建测试、MSRV 检查、Windows 构建、TLS/roaming/wasm 特性矩阵、schema 校验、MCP conformance、model-toolcall conformance、GDK Kotlin 冒烟测试、quarantine 隔离测试。另有 `cargo-deny`（依赖合规）、`cargo-machete`（无用依赖）、scorecard（供应链安全）等独立 workflow，共 39 个 workflow 文件。
- 质量观察：workspace 统一 clippy lints（`-D warnings`）；依赖全部收敛到 `[workspace.dependencies]`；有 `goose-self-test.yaml`、evals/ 目录承载 agent 行为评测。整体工程质量在同类开源 agent 项目中属第一梯队。

## 8. 社区与维护状态

- 治理：已入 AAIF（Linux Foundation），有正式 `GOVERNANCE.md`、`MAINTAINERS.md`、`SECURITY.md`、`CODE_OF_CONDUCT.md`；Linux Foundation Insights 健康分徽章（README）。
- 活跃度：最近 push 2026-11-02（gh api）；release 节奏极快——v1.57.0 于 2026-10-28 发布，仓库内有 `patch-release.yaml`/`minor-release.yaml`/`release-plz.toml` 支撑的自动化版本流水线（版本号已迭代到 1.57）。
- 贡献者：451 人（gh api 分页）；有 CODEOWNERS、PR 模板、issue 模板、stale/take 等 community workflow。
- 商业背景：README 中作者邮箱 `ai-oss-tools@block.xyz`（Cargo.toml workspace authors），表明 Block 仍是主力维护方，但项目已置于中立基金会治理下。

## 9. 亮点与风险

**亮点**

1. 状态机引擎与业务解耦设计出色：`goose-agent` 只含抽象（Operation/Effect/Emitter），19 个具体操作在 `state_machine_builder.rs` 中声明式拼装，取消、审批、重试、压缩都成为可独立测试的一等公民。
2. MCP-first 的扩展架构完整：stdio/HTTP/builtin/platform 四类扩展统一抽象，租约机制（ExtensionLease）保证会话级隔离，内置 server 走内存管道零进程开销。
3. 分发面极广且相互复用同一内核：CLI、Electron 桌面、ACP stdio/HTTP 服务、uniffi SDK、Telegram 网关、iroh P2P，一套 `crates/goose` 打天下。
4. 供应商覆盖罕见地宽（40+ 适配文件），尤其 ACP 路线让用户能复用 Claude Code/Codex/Gemini 订阅而非单纯 API key。
5. 工程化程度高：4000+ Rust 测试、25+ CI job、conformance/quarantine/MSRV 矩阵、自动化 release。

**风险/局限**

1. 体量大：tarball 315MB、仓库约 1GB，15 crates + 5 UI 包 + 文档站 + evals 全在一个 monorepo，二次开发上手成本高；核心文件（services.rs 2980 行）偏重。
2. 历史命名混乱：从 block/goose 迁移到 aaif-goose/goose，文档域名也换到 goose-docs.ai，网上旧教程链接与仓库引用可能失效（本次下载即因清单记录的旧名而需先核实）。
3. 安全面广：platform 扩展可直接访问 agent 进程、shell/code_execution 能力天然高危，虽有 GooseMode 四档与 malware check，但依赖用户正确配置。
4. Electron 桌面端资源占用偏重，且 `built-in-extensions.json` 与 Rust 侧内置扩展存在两处维护点。

## 10. 总结评价

goose 是目前开源 AI agent 领域工程质量与功能完整度都在第一梯队的项目：以 Rust 状态机为核心、MCP 为扩展标准、ACP 为多端分发协议，把"一个 agent 内核、多种交付形态"做到了教科书级别的清晰分层，加上 Block 输血与 Linux Foundation 中立治理，1.57 个版本的快速迭代和 451 名贡献者证明了其可持续性。适合两类人：想在自己的产品里嵌入一个可靠 agent 运行时的开发者（走 SDK/ACP），以及想研究"生产级 agent 主循环该怎么设计"的工程师（直接读 `crates/goose-agent/src/machine.rs` 与 `crates/goose/src/agents/state_machine_builder.rs`，这条 19 步操作链是全仓最精华的部分）。
