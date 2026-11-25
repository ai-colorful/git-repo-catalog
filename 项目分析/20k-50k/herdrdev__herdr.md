# herdrdev/herdr 项目分析

> 星数 48082（快照 2026-10-24）｜主语言 Rust｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：**AI 编码 agent 的终端工作区运行时**——一个 tmux 式的 C/S 终端复用器，但把"识别、调度、等待编码 agent"做成了原生能力（官方自述 "the runtime your coding agents live on"，`README.md`）。

| 项目 | 事实 |
|---|---|
| 名称 | herdrdev/herdr，当前版本 0.9.5（`Cargo.toml`） |
| 星数 | 48,082（gh api 核实，快照 2026-10-24） |
| 主语言 | Rust（工具链 1.96.1，`rust-toolchain.toml`）+ 少量 vendored Zig |
| 许可证 | Apache-2.0 |
| 创建时间 | 2026-03-27（v0.1.0 同日发布，`CHANGELOG.md`） |
| 最近推送 | 2026-11-03（gh api 核实） |
| 其他 | 3,862 forks、428 open issues、89 contributors、26 open PRs（gh api） |

## 2. 项目是什么，解决什么问题

herdr 解决的核心痛点是：**同时运行多个 AI 编码 agent（Claude Code、Codex、Cursor、Gemini CLI 等）时，人无法知道哪个 agent 在干活、哪个卡住了等人**。传统做法是开一堆 tmux 窗格来回切；herdr 把每个窗格里的 agent 状态实时识别出来（working / blocked / idle），哪个 agent 停下来要答案，界面会直接告诉你（`README.md`、`src/detect/mod.rs` 模块注释）。

它保持了 tmux 的核心体验：服务端持有 PTY，客户端断开（detach / SSH 掉线）后窗格照常运行，重连后恢复布局，甚至能恢复受支持的 agent 会话本体（`src/server/headless.rs` 模块注释、`src/persist/restore.rs`）。在此基础上做了 tmux 不做的事：

- **agent 原生接口**：102 个 Socket API 方法（`src/api/schema.rs`），agent 可以互相 spawn 窗格、读对方输出、`agent.wait` 等到对方真正 blocked 才返回——为多 agent 协作编排提供原语。
- **不替代、只承载**：herdr 不包装任何 agent，只是"拥有它们的终端"。通过 `integration install` 给各家 agent CLI 安装 hook，让 agent 主动上报状态（`src/integration/targets.rs`）。
- **单二进制无 Electron**：纯 Rust + ratatui TUI，跑在任意已有终端里，支持 macOS / Linux / Windows（ConPTY）。

中文 README（`README.zh-CN.md`）与 Homebrew / mise / 一键安装脚本（`distribution/install.sh`）等分发渠道齐全。

## 3. 技术栈与依赖

| 组件 | 选型与用途 |
|---|---|
| 语言 | Rust 2021 edition（主 crate 约 291K 行，`Cargo.toml` + 实测） |
| TUI 渲染 | ratatui 0.30 + crossterm 0.29 |
| 终端解析 | vendored **libghostty-vt**（Ghostty 的 Zig 终端仿真核心，757 个 .zig 文件）+ 自写 Rust FFI 绑定 crate `crates/ghostty-vt`（12K 行）；对子进程暴露 TERM=xterm-256color |
| PTY | portable-pty 0.9（vendor 补丁版，`vendor/portable-pty`），Linux/macOS 另有自研 actor/backend（`src/pty/`） |
| IPC | interprocess 2.4（Unix domain socket / Windows named pipe），tokio 1 多线程运行时 |
| CLI | clap 4.5 + 自定义子命令分发（`src/cli.rs`） |
| 平台层 | zbus 5.19（Linux logind 关机延迟抑制）、windows-sys / WMI / Notification Center（`Cargo.toml`） |
| Schema | schemars 1.2 生成 `docs/next/api/herdr-api.schema.json` |
| 构建 | cargo workspace（主 crate + ghostty-vt）+ justfile + Nix flake |

## 4. 目录结构与代码组织

| 目录 | 职责 | 规模（行） |
|---|---|---|
| `src/server/` | 后台服务端：事件循环、客户端接入、帧流推送、关机/handoff | 30K |
| `src/client/` | TUI 客户端：渲染、输入、attach 生命周期 | 67K |
| `src/app/` | AppState 纯数据层 + 动作 + 输入路由 | 39K |
| `src/api/` | Socket JSON API：schema、event hub、订阅 | 9.5K |
| `src/cli/` | agent/pane/workspace/tab/plugin/integration 等子命令 | 11K |
| `src/detect/` | 屏幕检测引擎 + 22 个 agent 清单 | 5.7K |
| `src/pane/` | 窗格状态、OSC 追踪、光标、kitty 键盘协议 | 13K |
| `src/integration/` | 给各家 agent CLI 装/卸 hook | 12K |
| `src/persist/` | 会话快照 v3、原子写、恢复与 agent resume | 5.2K |
| `src/protocol/` | 客户端二进制协议、surface delta 帧差量 | 8.9K |
| `src/platform/` | linux/macos/windows 平台隔离层 | 18K |
| `src/pty/` `src/remote/` `src/terminal/` `src/workspace/` | PTY actor、SSH 远程机、终端运行时、工作区/git worktree | 18K |
| `crates/ghostty-vt/` `vendor/` | ghostty FFI 绑定 + vendored libghostty-vt/portable-pty | 12K(RS)+Zig |

组织方式评述：单仓库单主 crate（非多 crate 拆分，仅 ghostty-vt 独立成 crate），靠 19 个顶层模块目录划分职责。`AGENTS.md`（31KB）写明架构纪律：AppState 纯数据可脱离 PTY 测试、render 纯函数只画不改、平台代码隔离在 `platform/<os>.rs`、检测器只读屏幕快照不碰解析器状态、渲染/PTY/检测是"乘法级"性能路径需按 1 与 15+ 窗格做规模基准。

## 5. 核心架构与关键实现

### 5.1 C/S 分离与无头服务端（tmux 内核）

服务端不进 raw mode、不读 stdin，渲染到内存里的虚拟 ratatui Buffer，在两个 socket 上监听：`herdr.sock`（JSON API）与 `herdr-client.sock`（二进制帧协议），把每帧 surface delta 推给客户端，客户端断开后继续跑（`src/server/headless.rs` 模块注释）。掉线/重启后 `src/persist/` 的 SessionSnapshot（version=3）原子落盘并恢复布局、ANSI 滚动缓冲（`src/persist/snapshot.rs`、`src/persist/restore.rs`）。

### 5.2 agent 状态检测：清单驱动的屏幕规则引擎

核心创意所在。检测器周期性读窗格底部缓冲文本 + OSC title/progress，按优先级规则匹配出 `AgentState::{Idle, Working, Blocked, Unknown}`（`src/detect/mod.rs`）。每个 agent 一个 TOML 清单，如 Claude 的规则（`src/detect/manifests/claude.toml`）：

```toml
[[rules]]
id = "live_blocked_form"
state = "blocked"
priority = 980
region = "after_last_horizontal_rule"
visible_blocker = true
contains = ["esc to cancel"]
any = [ { contains = ["enter to confirm"] }, ... ]
```

规则支持 region（只看提示框/横线以下等区域）、`any/all/not` 布尔门、正则（含 Braille 转轮、半圆 busy 指示符），注释里还引用了真实 issue 编号校准（#4573、#2650）。清单可远程热更新（`src/detect/manifest_update.rs`），共 22 个：claude、codex、gemini、cursor、droid、grok、qwen、kimi、opencode、github-copilot、devin、antigravity、cline、amp、kiro、letta、pi、hermes、kilo、muse、maki、qodercli（`src/detect/manifests/`）。

### 5.3 双向集成：hook 上报 + 自我恢复

检测是"读屏"，另一条腿是"上报"：`integration install <agent>` 会改写 Claude 的 settings hooks、Codex 的 config hook、Cursor/Grok/Droid 等各自配置（`src/integration/targets.rs`、`src/integration/config_edit.rs`），在会话/轮次事件时回调 herdr。恢复侧，agent 自己报告 resume 命令（如 `claude --continue`），herdr 存为去重键，机器重启后在新窗格原目录重启会话（`src/agent_resume.rs` 的 `ReportedAgentResume::plan()`）。

### 5.4 面向 agent 的 API 与 skill

102 个方法分 workspace/tab/pane/agent/worktree/plugin/events 命名空间（`src/api/schema.rs`），关键编排原语：`agent.start/prompt/wait`、`pane.wait_for_output`、`events.subscribe/wait`。仓库自带 `skills/herdr/SKILL.md`，教 coding agent 先校验 `HERDR_ENV=1`、用 `herdr --help` 自发现语法、读 JSON 返回而非猜测——这是它"agent 运行时"定位的落点：**agent 既是被管理的对象，也是操作 herdr 的用户**。

## 6. 构建与运行

```bash
git clone https://github.com/herdrdev/herdr && cd herdr
cargo build --release        # 需先构建 vendored Zig 库：just build-libghostty-vt（需 Zig 工具链）
just test                    # 单元 + 集成测试
just check                   # 格式化、测试与维护性检查（含 windows-lint）
herdr                        # 裸命令启动/附着 TUI；ctrl+b q 分离
herdr agent / pane / workspace / integration / machine   # 子命令组，输出 JSON
curl -fsSL https://herdr.dev/install.sh | sh  # 或 brew install herdr / mise / PowerShell
```
（来源：`README.md`、`justfile`、`skills/herdr/SKILL.md`）

## 7. 测试与工程质量

- 集成测试 `tests/` 34 个文件约 22.5K 行，覆盖 detach/reattach、multi_client、session_restore、live_handoff、host_shutdown、windows_server_security 等真实场景；`#[cfg(test)]` 内联测试遍布 226 个源文件。
- CI（`.github/workflows/ci.yml`）：conventional-commits 校验 + 全平台 check + windows-conpty-package；另有 pr-gate、nix、release、windows-arm64 等 11 个 workflow。
- justfile 提供 9 个基准目标（render-scale、api-fairness、bsp-layout 等），AGENTS.md 要求性能敏感改动附 1/15 窗格规模对比数据——工程质量纪律明显高于同期多数新项目。
- 检测规则注释普遍带 issue 号溯源，vendor 补丁有独立 `.patches.md` 台账（`vendor/libghostty-vt.patches.md`）。

## 8. 社区与维护状态

- 发版极密：CHANGELOG 61 个版本，v0.1.0（2026-03-27）到 v0.9.5（2026-10-14）平均每周 2 个版本，另频繁发 preview tag（gh api releases）。
- 89 名贡献者；CHANGELOG 中外部贡献者 @JJLiebig、@minatoaquaMK2 反复出现在 Windows/恢复修复中，社区贡献已进入核心链路（`CHANGELOG.md`）。
- 议题体量大：428 open issues、26 open PRs，维护者单人主导（AGENTS.md 提及 "Can's workstation"，作者疑似单人主导 + 外部贡献者结构，未最终确认）。
- 增速客观陈述：2026-03-27 创建至快照约 7 个月达 48k 星，平均 ~220 星/天，属极快增速；同期仓库具备 3.8k fork、高密度 issue 互动与持续发版，使用与开发生态迹象真实；tarball 无星史数据，无法从源码进一步核验星数曲线。

## 9. 亮点与风险

**亮点**
1. 卡位精准：第一个把"多 agent 状态感知"做成终端复用器核心能力的项目，检测引擎工程化程度（优先级/区域/布尔门/远程热更）远超 demo 水平。
2. 架构干净：C/S + 纯函数渲染 + 平台隔离层，30 万行仍保持可测性（AppState 不依赖 PTY 即可测试）。
3. 真正跨平台：Windows ConPTY/通知中心/JobObjects/ARM64 CI 是同类 tmux 系工具的短板，这里是一等公民。
4. 对 agent 友好：102 方法 Socket API + 自带 SKILL.md + agent 自报 resume 命令，把 agent 当一等用户设计。
5. 工程纪律：61 版本的 CHANGELOG、conventional commits、性能基准门槛、vendor 补丁台账。

**风险**
- 核心依赖 vendored libghostty-vt（Zig）与补丁版 portable-pty，上游演进需人工跟版，构建链引入 Zig 工具链门槛。
- 屏幕检测天然脆弱：各 agent UI 改版（换图标/文案）即需更新 22 份清单，长期维护成本高，hook 上报与读屏双轨并存正是为此对冲。
- 项目仅 7 个月大、0.x 版本，API 与快照格式（v3）仍在变动；428 open issues 对疑似小核心团队是不小的积压压力。
- 极快增速叠加"AI runtime"热点概念，热度回落后的留存有待观察（客观风险，非臆断）。

## 10. 总结评价

herdr 是 2026 年"AI 编码 agent 基础设施"赛道中完成度最高的终端运行时：把 tmux 验证过的 C/S 复用器内核与 agent 状态检测、hook 集成、Socket API 编排原语组合成单一 Rust 二进制，29 万行代码配得上 48k 星的增速。适合同时挂多个 coding agent 的重度用户、需要编排多 agent 工作流的自动化开发者；对研究"agent 原生工具如何设计状态机与人机接口"的人，`src/detect/` 的清单引擎与 `src/api/schema.rs` 的方法面是极好的读本。
