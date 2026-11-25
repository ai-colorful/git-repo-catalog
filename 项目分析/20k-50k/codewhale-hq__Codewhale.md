# codewhale-hq/Codewhale 项目分析

> 星数 41173（快照 2026-10-24）｜主语言 Rust｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：用 Rust 编写的开源终端编码 agent——支持任意模型（52 个内置 Provider 描述符 + 本地模型）、可把一个任务拆给多个 agent（sub-agent / Fleet / Workflow 三层机制），主打"模型可以是任何人的，agent 必须是你自己的"。

| 项 | 值 |
| --- | --- |
| 名称 | codewhale-hq/Codewhale（默认分支 main） |
| 星数 | 41,173（gh api 实测 2026-11-14） |
| 语言 | Rust（edition 2024，MSRV 1.89，源码构建锁 1.100.0，见 rust-toolchain.toml） |
| 许可证 | MIT（LICENSE） |
| 创建时间 | 2026-01-19；最近 push 2026-11-13（gh api 实测，昨天仍在推送） |
| 最新发布 | v0.10.2（2026-10-24 GitHub Release）；Cargo.toml/CHANGELOG 已进 0.10.3（2026-11-08，未发 Release） |
| 代码规模 | crates/ 下 1,590 个 .rs、约 167 万行；另有 vendor 分叉与 web/ 站点 |

## 2. 项目是什么，解决什么问题

Codewhale 是一个跑在终端里的编码 agent：给出任务后它读仓库、改文件、跑命令，直到任务完成或需要人介入（README.md "What it does"）。它沿 codex-cli / claude-code 的赛道，但差异化立场非常明确——单二进制、无账号、自带密钥本地保存、全代码可读可审计。

项目前身是 `deepseek-tui`，至今仍兼容读取其配置与会话，且默认 Provider 是 DeepSeek、默认模型 `deepseek-flash`（README "History and license"、docs/PROVIDERS.md）。但架构上刻意做到模型无关：任何 OpenAI 兼容端点、Anthropic 原生协议、本地 Ollama/vLLM/SGLang 都走同一套终端 harness（docs/PROVIDERS.md）。

多 agent 是其核心卖点之一：`/goal` 设定跨轮目标，任务可拆给 sub-agent（单 `agent` 工具为唯一创建面）、可跑带角色与模型的 Fleet 编队、可执行签入仓库的 Workflow 工作流（README "Several agents on one task"、docs/FLEET.md）。

安全模型较完整：双开关体系（Tab 切 Plan/Work/Operate 模式、Shift+Tab 切 Ask/Auto-Review/Full Access 权限），Ask/Auto-Review 下命令进 OS 沙箱（macOS Seatbelt、Linux bubblewrap，Windows 暂无），沙箱内禁止读 `~/.ssh`、`~/.aws`、`.env` 等凭据存储；每次审批、每条网络决策落审计日志（README "How it stays safe"、docs/ARCHITECTURE.md Security 节）。

工程面提供 receipts（会话做过什么：改了哪些文件、跑了哪些命令、谁批的）/ `/undo` 单步回退 / `/restore` 快照回滚三件套（README "Receipts and undo"）。另有 `codewhale exec` 无头模式、`codewhale web` 本地浏览器客户端、Runtime API（HTTP/SSE）与 ACP（Zed 等编辑器接入）——四类前端共享同一引擎（README "Ways to run it"、docs/ARCHITECTURE.md）。

## 3. 技术栈与依赖

- **语言**：Rust edition 2024；workspace 46 个成员 crate + 3 个 tests/portable-* 集成 crate（Cargo.toml）
- **异步运行时**：tokio 1.50（多特性全开，Cargo.toml）
- **TUI**：自维护分叉 vendor/codewhale-ratatui（"Codewhale Ratatui"，16 主题、海洋动效组件库，带独立 README/DESIGN/BENCHMARKS）；并 patch 上游 unicode-width（CJK 宽度修正 #4479）与 crossterm（Windows Terminal 粘贴丢 emoji 修复）
- **HTTP/API**：axum 0.8（Runtime API，含 ws）、reqwest 0.13 + rustls（网络层）、tower-http
- **持久化**：rusqlite 0.41 bundled（crates/state 会话/线程/任务存储）；gix 0.90（git 操作）
- **脚本扩展**：rquickjs 0.14（Workflow 的 QuickJS VM，crates/workflow-js，刻意单线程、经 channel 与多线程引擎桥接，Cargo.toml 注释）
- **其他**：mimalloc 分配器（rusty_alloc 为可选替代）、jsonschema、yaml-rust2、clap 4
- **周边**：web/（Astro 类站点）、telemetry-ingest/（遥测接收端）、extensions/（TS mods，含 vscode 目录）、npm/ 与 .winget/（分发）、.cnb.yml 与 sync-cnb/sync-gitcode CI（CNB/GitCode 镜像同步）

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| crates/engine | 引擎核心，约 67.5 万行：turn loop、client、tools/、subagent/、compaction/purge、hooks |
| crates/tui | 终端运行时（约 45 万行）：ratatui UI、runtime_api.rs、task_manager.rs；正按 docs/design/TUI_DECONSTRUCTION.md 拆分 |
| crates/config | 配置加载、ProviderKind 枚举、52 条 provider_descriptors.json |
| crates/execpolicy | 工具执行审批/沙箱策略引擎（canonical action 体系） |
| crates/sandbox | Seatbelt/bwrap/OpenSandbox 后端、read_guard、进程加固、shell 调度 |
| crates/state、crates/memory | SQLite 线程/会话持久化；带溯源的本地记忆库 |
| crates/agent-contract | FleetRole、ChildAuthority/ChildGrant、worker profile 契约类型 |
| crates/workflow、workflow-js | Workflow 类型化 IR + QuickJS 脚本层 |
| crates/lane | Lane 运行时：Fleet/Workflow 的可附着持久实例（lane list/attach/logs） |
| crates/mcp、skills、plugins、skill-install、hooks | MCP 客户端+stdio server、技能体系、插件市场、生命周期 outbox |
| crates/app-server(-protocol) | HTTP/SSE + JSON-RPC 无头服务，约 11.5 万行 |
| crates/lsp、repl | 语言服务器客户端（rust-analyzer/gopls/pyright 等默认表）；Python REPL/RLM 运行时 |
| crates/cli | `codewhale` 可执行入口，29 行数最少的核心 crate，仅命令面 |
| vendor/codewhale-ratatui | 自维护 ratatui 分叉（独立版本化） |
| docs/ | 116 篇文档 + 18 种语言的 README/文档翻译 |
| fleets/ | 示例 roster（stopship.toml）；web/、deploy/、packaging/、telemetry-ingest/ |

组织方式评述：典型 Rust workspace 多 crate 架构，但重心极不均衡——engine+tui 两 crate 占 112 万行（约 2/3），ARCHITECTURE.md 开篇即承认"crates/tui 仍是活运行时，其余 crate 是增量拆出的"，拆分以"agent 工作分区"为单位推进并用 `scripts/check-command-crate-boundaries.py` 强制边界（防 runtime 反向依赖 UI）。这是"单体演进中"的真实中间态。

## 5. 核心架构与关键实现

### 5.1 一引擎多前端的 turn loop

所有前端（TUI/exec/web/Runtime API/ACP）收敛到 `Engine::run_turn`（docs/ARCHITECTURE.md）。外层循环在 `crates/engine/src/core/engine/turn_loop.rs`（8,949 行），私有阶段拆为 preparation（请求准备）、model_step（模型派发与准入）、continuation（有序续跑阶梯）、inline_repl、tool_batch 五相；每轮的簿记集中在 `TurnLoopProgress`（步预算、流重试预算、图像拒收恢复、fleet 拒绝守卫、goal 续跑计数等约 20 个状态字段，turn_loop.rs:44-69）。设计上"重试、终止、立即返回"三路分离（turn_loop.rs 模块注释）。

### 5.2 模型无关的 Provider 抽象

三线并存：`ProviderKind` 枚举（crates/config/src/provider_kind.rs，仅保留"不同 wire"的变体，40+ 厂商含大量 serde alias 兼容旧拼写）；52 条编译期打包的 provider_descriptors.json（wire/base_url/key env/aliases，模型列表可来自运行时 `GET /v1/models`）；以及 `[providers.<name>]` 自定义表（任意 OpenAI 兼容端点即一等公民）。HTTP 适配层在 `crates/engine/src/client.rs`（7,610 行）：OpenAI 兼容/Anthropic/Responses 三种 wire、DeepSeek 边界处理、重试与流式；`llm_client/` 提供 `LlmClient` trait、RetryConfig 与错误分类。默认 DeepSeek beta 端点（ARCHITECTURE.md "DeepSeek API Endpoints"）。

### 5.3 多 agent 拆分：agent 工具 + ChildAuthority + Fleet + Lane

子代理运行时约 3.6 万行（crates/engine/src/tools/subagent/）。模型面只暴露一个 `agent` 工具（创建），协调类工具（agents/list、message、wait 等）已退役为仅重放历史用（mod.rs 模块注释、ARCHITECTURE.md）。子代理不另起引擎：捕获父级权限投影到同一 Engine——核心是 `ChildAuthority::capture`（crates/engine/src/tools/subagent/engine.rs:105-140），对 shell 策略取父子最小值（`ShellPolicy::min_with`）、工具集取交集（`intersect_explicit_tool_scope`），再由 `ChildGrant::resolve` 依据 FleetRole（角色）、深度余量解析出可委托权限：

```rust
// crates/engine/src/tools/subagent/engine.rs
let mut effective = runtime.worker_profile.clone();
effective.shell = effective.shell.min_with(parent_shell);
let scope = intersect_explicit_tool_scope(&effective.tools, explicit_allowed_tools);
let grant = ChildGrant::resolve(
    &agent_type, &effective, scope, !runtime.would_exceed_depth(),
);
```

更持久的形态是 Fleet：roster 定义在 `fleets/<name>.toml`，`codewhale fleet run tasks.json --max-workers N` 起 run；Fleet 本身只做"编成与成员选择"，不执行不授权——协调者实际拉起 headless `codewhale exec` 子进程，运行记录入 `.codewhale/fleet.jsonl` 台账，可 interrupt/restart/resume（docs/FLEET.md）。其上再有 Workflow（crates/workflow 类型化 IR + QuickJS VM）与 Lane（可 attach 的持久运行实例，`codewhale lane attach`）。三层各司其职：agent 工具=临时并发，Fleet=稳定身份的持久编队，Workflow=可签入仓库的确定性编排。

### 5.4 沙箱与审批

审批策略与沙箱机制分层：crates/execpolicy 以"canonical action"归一化命令意图做审批决策；crates/sandbox 负责平台包装——seatbelt.rs 动态生成 macOS sandbox-exec profile（含 restrict 档与 Full Access 强制走沙箱的例外路径），bwrap.rs 生成 Linux bubblewrap 参数，opensandbox.rs 可把执行路由到远端（如阿里 OpenSandbox）HTTP 后端，read_guard.rs 统一拦凭据路径读取，process_hardening.rs 是 TUI 进程自身的 Linux 加固（纵深防御，非子命令沙箱）。Windows 的 helper 契约已写（windows.rs）但"未接入执行路径前不宣传"（ARCHITECTURE.md Security）。每次审批写入 `~/.codewhale/sessions/<id>/approval_receipts.jsonl`，安全事件入 append-only `audit.log`。

### 5.5 上下文管理

长对话用 compaction（crates/engine/src/core/engine/compaction.rs，1,514 行，分自动/手动两路，事件语义不同）；外科手术式裁剪走 purge（crates/engine/src/purge.rs，agent 主动删除/改写消息）；递归上下文走 RLM——持久本地 Python REPL 子进程（环境擦洗），变量与语义 helper 跨轮保留（ARCHITECTURE.md rlm 条目）；另有 LSP 后编辑诊断注入（core/engine/lsp_hooks.rs：File 写/改/补丁成功后收集诊断，下个 API 请求前以合成 user 消息注入，crates/lsp/registry.rs 默认映射 rust-analyzer/gopls/pyright/clangd 等 8 类服务器）。

## 6. 构建与运行

```bash
# 运行（安装脚本，macOS/Linux）
curl -fsSL https://codewhale.net/install.sh | sh
# Windows / npm / 源码
winget install --id HunterBown.CodeWhale --exact
npm install -g codewhale
cargo install codewhale-cli --locked      # crates/cli 是发布 crate 名
# 从源码构建（默认成员只有 crates/cli，见 Cargo.toml default-members）
cargo build --profile dist               # 发布产物用 dist profile（fat LTO）
cargo test                               # 测试用 .config/nextest.toml（nextest）
# 运行形态
codewhale                                # 交互 TUI
codewhale exec "…"                       # 无头单任务
codewhale web                            # 本地浏览器客户端
codewhale app-server --http              # Runtime API
codewhale serve --acp                    # ACP（Zed 等）
```

常用配置：`~/.codewhale/config.toml`（主配置）、`config.example.toml`（仓库内 11 万行带注释样例）、`~/.codewhale/mcp.json`、技能目录 `~/.codewhale/skills/`。

## 7. 测试与工程质量

- **测试规模**：源码内 `#[test]` 约 1.6 万个 + `#[tokio::test]` 约 3,175 个，合计近 1.9 万（grep 计数）；单 subagent 模块就带 tests.rs 2.87 万行、budget_handback/delivery/lifecycle/limits 各自独立测试文件；另有 tests/portable-session、portable-config、portable-core-navigation 三个独立集成 crate
- **CI**：.github/workflows/ 下 28 个工作流——ci.yml、codeql.yml、cargo-deny.yml、security-audit.yml、nightly.yml、release-artifacts.yml、release-parity.yml、ohos.yml（HarmonyOS）、windows-focus.yml、triage.yml、spam-lockdown.yml 等；`.config/nextest.toml` 统一测试配置；Cargo.toml 设 `warnings = "deny"` 并由 CI 的 `RUSTFLAGS: -Dwarnings` 强制
- **工程细节**：dev profile 只留行表调试信息（解决 227MB debug 二进制与 260GB target 增量问题，Cargo.toml 注释 #4991/#5246）；TUI panic 用 catch_unwind 监督使单个工具 panic 不炸整个会话（dist profile 无 panic=abort 的注释）；schema_version 门禁防止新旧二进制读写损坏状态（ARCHITECTURE.md "Durable Schema Gates"）；CHANGELOG 达 1MB、严格 Keep-a-Changelog 格式

## 8. 社区与维护状态

- **活跃度**：gh api 显示最近 push 2026-11-13（分析前一天）；2026-09 至 10 连发 5 个 release（v0.9.12→v0.10.2，约每两周一个）；issue 编号已进 #7300+ 区间
- **贡献者**：contributors API 翻页至 249 页（per_page=1，含匿名，约 249 人）；仓库另有 docs/CONTRIBUTORS.md 贡献者墙与 scripts/contributor-wall.py 重算脚本（含 co-authored 与致谢者）；CHANGELOG 持续具名致谢社区 PR（如 luochen211 #7322/#7315-7319、MannXo #7293）
- **社区渠道**：Discord、WeChat 群（README "Contributing"）；18 语言文档翻译与 CONTRIBUTING.id.md 等本地化
- **分发**：install.sh、winget、npm、cargo、Homebrew tap、Scoop、Nix、Termux preview、Docker（docs/INSTALL.md）；.cnb.yml、sync-cnb.yml、sync-gitcode.yml 表明有 CNB/GitCode 中国区镜像同步
- **背景（客观陈述）**：仓库 2026-01-19 创建，约 10 个月达 41.2k 星（约 4.1k 星/月），处于 2026 年终端编码 agent 赛道整体高热期；默认 Provider 为 DeepSeek（项目起源），并有 OrcaRouter 充值返佣活动（README 已自我披露为 referral campaign）。维护者署名 Hunter（winget id HunterBown；vendor README 中为 Hmbown），团队规模未见公开披露，未确认

## 9. 亮点与风险

**亮点**
1. 模型无关架构扎实：ProviderKind + 52 条描述符 + 自定义端点三层抽象，本地模型一等公民，切换模型不改 harness（crates/config/src/provider_kind.rs、crates/engine/src/client.rs）
2. 多 agent 三层设计清晰：临时 sub-agent（单工具入口+权限投影）/ Fleet（稳定身份+台账）/ Workflow（确定性编排）各管一段，ChildAuthority 的"最小权限捕获"实现优雅（engine.rs）
3. 安全可审计：审批策略引擎与沙箱分层，receipts/audit.log/approval_receipts.jsonl 全留痕，`/undo`、`/restore`、崩溃检查点与离线队列完备（docs/RECEIPTS.md、ARCHITECTURE.md）
4. 工程纪律罕见地强：近 1.9 万测试、28 个 CI 工作流、-D warnings、schema 门禁、crate 边界检查脚本，1MB CHANGELOG
5. 文档与本地化投入大：116 篇文档 + 18 语言翻译，docs/ 与代码同步演进

**风险/局限**
1. 体量过重：crates/ 167 万行、engine 单 crate 67.5 万行，TUI 拆分仍在半途（ARCHITECTURE.md 自述），外部贡献者读懂核心成本很高
2. 文档先于发布：README/docs 明言跟随 main 分支、可能描述未发布特性（0.10.3 的权限模型改动即如此），按文档使用会与 Release 行为不一致
3. Windows 无 OS 级命令沙箱（sandbox/windows.rs 未接入），Full Access 模式风险自担；遥测默认开启（可 `codewhale config set telemetry false` 关闭）
4. 默认 Provider/推荐位与商业推广绑定（DeepSeek 默认 + OrcaRouter 返佣），虽披露透明，但中立性存疑

## 10. 总结评价

Codewhale 是 2026 年终端编码 agent 竞赛中一个"工程洁癖型"选手：用单一 MIT Rust 二进制、模型无关与全链路可审计三个硬承诺，对标 codex-cli/claude-code 的闭源体验，又以 Fleet/Workflow/sub-agent 三层多 agent 和 receipts/undo 机制做出了自己的辨识度；10 个月 41k 星与其近 1.9 万测试、双周发版的工程投入相互印证，但 167 万行代码与未完成的 crate 拆分也意味着它已越过"轻松通读"的阶段。适合想用自有密钥/本地模型跑终端 agent 的开发者直接安装使用；想读源码或贡献者，建议从 docs/ARCHITECTURE.md、crates/agent-contract（契约层）与 crates/sandbox、crates/execpolicy（边界清晰的中小 crate）入手，engine/tui 两个巨 crate 宜按 ARCHITECTURE.md 的模块索引定点查阅。与 openhands 相比它无 Web 容器形态、更贴近个人终端工作流；与 codex-cli 相比它的 Provider 矩阵和多 agent 编排明显更广更深。
