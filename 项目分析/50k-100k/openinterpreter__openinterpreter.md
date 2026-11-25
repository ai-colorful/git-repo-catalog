# openinterpreter/openinterpreter 项目分析

> 星数 68746（快照 2026-10-24）｜主语言 Rust｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：Open Interpreter 已从著名的 Python"自然语言驱动计算机"项目转型为 OpenAI Codex 的 Rust 发行版 fork——一个为 Kimi K3、GLM、DeepSeek 等低成本开源模型专门优化 harness（提示词/工具/协议塑形）的终端编码 agent。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 仓库 | openinterpreter/openinterpreter（历史名 open-interpreter，现已重定向） | gh api |
| 版本 | 0.0.43（RELEASE_NOTES），最新 release rust-v0.0.56（2026-10-07） | RELEASE_NOTES.md / gh api releases |
| 主语言 | Rust（约 4813 个 .rs 文件、约 193 万行，不含 vendor） | 源码统计 |
| 许可证 | Apache-2.0 | LICENSE / gh api |
| 最近活跃 | push 2026-10-11，release 每周多个 | gh api |
| 形态 | 终端 TUI/CLI agent + app-server/exec 协议 + ACP 编辑器接入 | README.md |

## 2. 项目是什么，解决什么问题

历史背景：原 open-interpreter 是 Python 写的"让 LLM 在本地跑代码、操作电脑"的明星项目（星数主体在 Python 时代积累）。README 明确说明现状："This is the new Rust version of Open Interpreter, based on Codex. Looking for the original Python project? It lives on as a community-maintained fork at endolith/open-interpreter"（README.md 末尾注记）。即官方已把仓库改造为 Codex fork，Python 版移交社区。

当前产品：一个跑在终端里的编码 agent（命令 `i` / `interpreter`），直接复用 OpenAI Codex 的 Rust 内核（codex-rs），在其上叠加差异化能力——"emulating the agent harness that gets the best performance out of low-cost models"（README.md）。它发现各家模型在其"原生 agent CLI"（Claude Code、Kimi Code、Qwen Code 等）里表现更好，于是把这些 harness 的提示词、工具 schema、消息转换、响应处理逐一在 Rust 里复刻，让用户用一个二进制就能以"目标模型最熟悉的口吻"驱动任意 provider（README.md Harness Emulation 节；docs/harness.md）。

同时强调可移植与兼容：兼容 Codex exec 协议与 Codex SDK（README 提供一行 `codexPathOverride: "interpreter"` 的切换示例）、兼容 Agent Client Protocol（`interpreter acp`）、复用共享的 `AGENTS.md` 与 `.agents/skills` 目录，产品私有状态收敛在 `~/.openinterpreter`（README.md Portable by default 节；docs/portability.md）。

内置 Computer Use：随仓库附带 QA skill，可通过 vercel agent-browser 驱动真实浏览器、通过 trycua/cua 操作原生应用来测试界面（README.md Computer Use 节）。

品牌工程化：fork 未粗暴全局改名，而是通过 `codex-rs/product-info/src/lib.rs` 集中管理产品身份（`Product::current()` 选发行版、`display_name()`/`command_name()` 出名），`codex-package.json` 的 `variant: "open-interpreter"` 是安装包契约，`CODEX_*` 环境变量保留为兼容别名（FORK_BRANDING.md）。

## 3. 技术栈与依赖

- 语言：Rust（ editions/toolchain 见 codex-rs/rust-toolchain.toml），workspace 成员 120+ crate（codex-rs/Cargo.toml）。
- 构建/任务：Cargo + Bazel（MODULE.bazel、BUILD.bazel、.bazelrc）双构建体系；justfile 提供任务入口；pnpm monorepo 管理文档与维护脚本（package.json：`codex-monorepo`，node>=22）。
- 关键 Rust 依赖方向：tokio 异步、serde/serde_json、rmcp-client（MCP 客户端）、codex-api 自研三协议传输层（Responses/Chat/Messages）、v8/rusty-v8（code-mode，见 codex-rs/code-mode* 与 rusty-v8-release.yml）。
- 文档站：docs-site/ + docs.json（多语言 README_ZH/ES/JA）。
- Python 仅用于工具脚本（scripts/write_provider_catalog.py 生成 provider 目录；ruff.toml 约束）。

## 4. 目录结构与代码组织

顶层（tarball 根）：

| 目录/文件 | 职责 |
| --- | --- |
| codex-rs/ | Rust 主工程，120+ crate 的工作区（核心全部逻辑） |
| codex-rs/core | agent 内核：会话、turn 循环、harness、工具、审批（18M 源码） |
| codex-rs/tui | 终端交互界面（22M 源码，最大 crate） |
| codex-rs/protocol | 消息/事件协议（约 2.8 万行，protocol/src 共 27830 行） |
| codex-rs/sandboxing、linux-sandbox、windows-sandbox-rs、windows-sandbox-service、mxc-sandbox、bwrap、network-proxy | 执行沙箱与网络代理族 |
| codex-rs/app-server、acp-server、mcp-server、exec-server | 对外服务面（Codex 协议/ACP/MCP） |
| codex-rs/skills、hooks、memories、cloud-tasks、execpolicy 等 | 功能子系统 |
| docs/ | 产品文档源（harness.md、sandbox 相关、config-reference 等） |
| scripts/ | 安装器、发布打包（build-interpreter-release.sh）、provider 目录生成 |
| .github/workflows | 大量 CI：rust-ci、rust-release*、bazel、python sdk 遗留等 |
| docs-site、logo、sdk、tools | 站点资产、Python SDK 遗留、维护工具 |

组织方式评述：这是典型的"上游大型 monorepo 整体继承"——完整保留 Codex 的 crate 划分与内部命名（codex-core、codex-tui 等按 FORK_BRANDING.md 要求不改名），Open Interpreter 自身的增量集中在 harness 目录、product-info 品牌、provider 目录、安装器/发布脚本，改动面克制、利于跟随上游合并。

## 5. 核心架构与关键实现

### 5.1 Harness 模拟层（本 fork 的核心增量）

`codex-rs/core/src/harness/`（共 20153 行）实现 13 种 harness：claude_code（4870 行）、kimi_cli（2972 行）、zcode（2729 行）、terminus_2、mini_swe_agent、opencode、swe_agent、deepseek_tui、minimal、kimi_code、qwen_code、little_coder、pi。每种 harness 提供 `build_request` 等 `build_*` 函数，把内核中立的 Prompt/ResponseItem 塑形成目标模型熟悉的请求。

路由由 `codex-rs/core/src/harness/routing.rs` 决定，按 (wire_api, harness) 二维映射到 `StreamTransportRoute`：

```rust
pub(crate) enum StreamTransportRoute {
    ResponsesApi,
    ChatCompletionsCompat,
    ChatHarness(ChatHarnessRoute),
    MessagesHarness(MessagesHarnessRoute),
    /// claude-code shaping carried over the Responses wire (`/responses`).
    ClaudeCodeResponses(ClaudeCodeProfileRoute),
    /// claude-code shaping carried over the chat-completions wire
    ClaudeCodeChat(ClaudeCodeProfileRoute),
}
```

claude_code.rs 是最彻底的模拟：完整构造 Anthropic Messages 请求（AnthropicMessage/AnthropicThinkingConfig/AnthropicCacheControl 等），并伪装客户端指纹——`CLAUDE_CODE_USER_AGENT = "claude-cli/2.1.158 (external, sdk-cli)"`、`CLAUDE_CODE_STARTUP_HEAD_USER_AGENT = "Bun/1.3.14"`、成套 beta header 与 billing header 常量（codex-rs/core/src/harness/claude_code.rs 头部）。harness 还带各自的 prompt 资产文件（如 kimi_code_system_prompt.md、zcode_tools.json、claude_code_prompt.rs）。文档明确"不 shell out 到真实外部 CLI，全部在 Rust 内复刻"（docs/harness.md）。

### 5.2 语言模型主循环

`codex-rs/core/src/session/turn.rs` 的 `run_turn`（3089 行文件）是采样循环，模块注释即架构说明："at each sampling request, the model replies with either requested function calls or an assistant message; if the model requests a function call, we execute it and send the output back to the model in the next sampling request"（turn.rs:149-162）。循环体内做：pending input 排水（turn 中途插入的用户消息）、step context 捕获（上下文+工具清单+MCP servers）、pre-sampling compaction（上下文接近阈值先压缩，compact.rs/compact_remote_v2.rs，失败重试"once per model step, so ineffective compaction cannot loop"）、流式响应经 stream_events_utils 解析分派工具调用或落史。外围由 thread_manager.rs（2595 行）管理会话生命周期。

### 5.3 执行环境沙箱（跨平台纵深）

`codex-rs/sandboxing/src/lib.rs` 导出统一 `SandboxManager`/`SandboxExecRequest` 抽象，平台后端分派（manager.rs `get_platform_sandbox`）：

- macOS：Seatbelt（.sbpl 策略，含 seatbelt_base_policy.sbpl、seatbelt_network_policy.sbpl，另有 seatbelt_daemon）；
- Linux：Landlock + 可选系统 bubblewrap（bwrap.rs、landlock.rs、find_system_bwrap_in_path），另有独立 linux-sandbox 可执行 crate；
- Windows：restricted token（windows.rs、windows-sandbox-rs/-service crate）与 MXC 沙箱（windows_mxc.rs、mxc-sandbox）。
- 网络：network-proxy crate 提供 MITM 代理（certs.rs、authorization_path.rs），实现网络级沙箱与域名白名单。

应用侧的 `codex-rs/core/src/exec_env.rs` 负责按 `ShellEnvironmentPolicy` 构造子进程环境（env_clear 后按白名单注入，避免变量泄漏），注入 `CODEX_SESSION_ID`/`CODEX_THREAD_ID`/`CODEX_VERSION` 等会话标识。审批与策略在 core/src/exec_policy/（executable_identity.rs、model_policy.rs）与 permissions 体系。

### 5.4 消息协议与多端接入

`codex-rs/protocol/src/protocol.rs` 定义核心事件枚举 `EventMsg`（Error/Warning/TurnStarted/TurnComplete/ContextCompacted/ModelReroute/Realtime* 等，serde tag=type 蛇形命名，并 `alias = "task_started"` 兼容 v1 wire 格式），协议 crate 共约 2.8 万行。对外三张面孔（README.md + docs/）：Codex app-server/exec 协议（app-server*、exec-server* crate，可被 Codex SDK 直接接管）、ACP（acp-server，接 Zed 等编辑器）、MCP（mcp-server + rmcp-client 双向）。

## 6. 构建与运行

```bash
# 安装（官方）
curl -fsSL https://www.openinterpreter.com/install | sh      # macOS/Linux
irm https://www.openinterpreter.com/install.ps1 | iex        # Windows
# 启动：终端输入 i 或 interpreter

# 源码构建（codex-rs 工作区）
cargo build --manifest-path codex-rs/Cargo.toml
cargo test  --manifest-path codex-rs/Cargo.toml   # CI 另用 nextest（rust-ci-full-nextest-platform.yml）
just / bazel build                                 # 备选任务体系（justfile、MODULE.bazel）

# 常用交互命令（README.md / docs/harness.md）
/model          # 切换 provider/model
/harness        # 切换 harness（claude-code、kimi-code、qwen-code、deepseek-tui…）
interpreter acp # 作为 ACP agent 接入编辑器
interpreter --chat-completions  # 走 OpenAI 兼容 Chat Completions

# 发布打包（fork 维护者）
scripts/build-interpreter-release.sh + .github/workflows/rust-release.yml
```

## 7. 测试与工程质量

- 规模：codex-rs 下 1821 个 *tests*.rs / tests 目录文件，测试普遍与被测代码同目录伴生（如 core/src/exec_policy_tests.rs、harness 各模块测试），另有独立 tests/ 目录（codex-rs/analytics/src/tests/suite/）。源码统计。
- CI：rust-ci.yml / rust-ci-full.yml / postmerge-ci.yml（nextest 多平台矩阵）、cargo-deny（供应链/license 检查）、bazel.yml、codespell、blob-size-policy、readme-translations（多语言 README 校验，scripts/check-doc-locales.mjs）。.github/workflows/ 目录。
- 质量观察：cargo deny.toml、clippy.toml、rustfmt.toml 齐备；测试命名与模块划分非常工整；同时保留大量上游遗留（python-sdk-release.yml 等历史 workflow），fork 技术债可控。FORK_BRANDING.md 提供品牌改动的验证清单（TUI snapshot、installer、rg 审计残留字面量），工程纪律良好。

## 8. 社区与维护状态

- 活跃度：默认分支 main，最近 push 2026-10-11；release rust-v0.0.56（2026-10-07）、0.0.55/0.0.54（2026-09-30 同日两发），节奏高频（gh api releases）。
- 贡献者：contributors 分页约 615 页（per_page=1，含匿名），但其中主体继承自原 open-interpreter/上游 Codex 历史；当前增量开发推测由 Open Interpreter 团队小群体驱动（RELEASE_NOTES 为单品牌口吻，无独立贡献者数据佐证，未确认）。
- 星数构成：68746 星主要为 Python 时代积累（仓库 id 666299222 为原 open-interpreter），转型后延续承载（gh api）。
- 生态：配套 openinterpreter/interpreter-workstation（桌面/无头客户端）、openinterpreter.com 文档站与 Discord 社区（README.md）。

## 9. 亮点与风险

亮点：
1. Harness 模拟是独到产品思路：用约 2 万行 Rust 精确复刻 13 种主流 agent CLI 的提示词/工具/协议面，让廉价开源模型立刻拿到"原生 harness 级"表现，且不依赖外部 CLI（codex-rs/core/src/harness/、docs/harness.md）。
2. 基于成熟的 Codex 内核，沙箱（Seatbelt/Landlock/bwrap/Windows/MXC + 网络代理）、审批、MCP、hooks、compaction 全部开箱即得，工程底座远超同类新项目（codex-rs/sandboxing/、core/src/session/turn.rs）。
3. 兼容性策略激进且务实：Codex SDK 一行覆盖即可替换、ACP/MCP/AGENTS.md 全兼容，用户迁移成本近乎为零（README.md）。
4. 品牌工程化范式可复用：Product 抽象 + package variant 契约 + 审计清单，把"fork 如何不改坏上游"写成了文档（FORK_BRANDING.md）。

风险/局限：
1. 强依赖上游 Codex：协议字段、crate 命名、行为均继承上游，上游大改时合并成本高，FORK_BRANDING.md 自己也承认这是"发行版 fork"而非独立实现。
2. 模拟外部产品指纹（claude-cli User-Agent、beta/billing header 伪造）存在被 provider 风控或条款变化打断的脆弱性（claude_code.rs 常量区）。
3. 版本仍处 0.0.x（rust-v0.0.56），RELEASE_NOTES 自述"维护性发布"，API/行为稳定性有限。
4. 规模庞大（约 193 万行、120+ crate），二次开发上手门槛高；大量遗留 Python SDK/Bazel 体系增加维护面。

## 10. 总结评价

Open Interpreter 是一个"星数为旧项目遗产、代码为全新形态"的特殊样本：原 Python 自动化明星项目被官方改造为 Codex 的 Rust 发行版，用 harness 模拟层（13 种、约 2 万行）换取低成本开源模型的最佳 agent 表现，底座则是 Codex 成熟的沙箱、审批与多协议接入体系。它适合：想在 Kimi/GLM/DeepSeek 等低价模型上获得接近 Claude Code/Codex 体验的终端用户；研究"同一模型在不同 harness 下表现差异"的 agent 工程师（harness 目录是绝佳教材）；以及想学习生产级 Rust agent 内核（沙箱+协议+TUI）的开发者。读源码建议从 docs/harness.md 与 codex-rs/core/src/harness/routing.rs 入口切入，再看 session/turn.rs 的采样循环与 sandboxing/ 的平台分派。
