# vercel-labs/agent-browser 项目分析

> 星数 45767（快照 2026-10-24）｜主语言 Rust｜档位 20k-50k｜分析日期 2026-11-06

## 1. 项目速览

一句话定位：Vercel Labs 出品的 Rust 原生 CLI，把浏览器交给 AI agent 逐条命令驱动——`snapshot` 把页面压成带 `@eN` 引用的紧凑文本，agent 引用元素执行动作，浏览器常驻后台。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | agent-browser（npm 包同名，v0.41.2） | `package.json`、`cli/Cargo.toml` |
| 星数 / fork / issue | 45767 / 3141 / 490 open | gh api 2026-11-06 |
| 语言 | Rust（单 crate，约 13.0 万行） | `cli/src/` 统计 |
| 许可证 | Apache-2.0 | `LICENSE`、`Cargo.toml` |
| 创建 / 最近 push | 2026-01-11 / 2026-11-05 | gh api |
| 最新 release | v0.41.2（2026-11-03），累计 118 个版本 | `CHANGELOG.md`、gh api releases |
| 主页 | https://agent-browser.dev | gh api homepage |

## 2. 项目是什么，解决什么问题

agent-browser 解决的是"AI agent 如何可靠操作网页"的问题。README（`README.md`）给出的核心循环是：`open` 打开页面 → `snapshot -i` 读取交互元素（每个元素带 `@eN` 引用）→ `click @e1` 等命令作用于引用 → 再次 `snapshot` 验证结果。页面以无障碍树的紧凑文本形式呈现而非截图，让 LLM 以极小的 token 开销"看懂"页面；每次操作都是一条短 shell 命令，浏览器在命令之间保持打开（README "How it works"）。

与传统自动化库（Playwright/Puppeteer 是让开发者写代码的 SDK）不同，agent-browser 的使用界面就是 shell：一条命令一个动作，天然匹配 agent 逐条调用 Bash 工具的工作方式。错误语义也按 agent 需求设计——点击目标被遮挡、隐藏或不可用时直接报错而不是"假成功"，提示 agent 重新快照再试（README Quick start 一节的显式说明）。

技能（skills）体系是其面向 agent 的关键设计：`agent-browser skills get core` 从二进制内嵌数据输出与当前 CLI 版本严格匹配的使用指南，配合 `--ref` 按需加载单个参考文档（`skill-data/core/SKILL.md`）。这解决了第三方技能文档与工具版本漂移的通病；`skills/agent-browser/SKILL.md` 则是一个刻意做薄的发现 stub，只负责把 agent 引导到 `skills get core`（AGENTS.md 明确说明此分层意图）。

工程上它同时提供 CLI 与 MCP 两种形态：`agent-browser mcp` 启动 stdio MCP server（`cli/src/mcp.rs`），且仓库规约强制 CLI 与 MCP 表面同步演进（AGENTS.md "CLI/MCP Parity" 一节）。这使同一个引擎既能服务 Claude Code 等 shell 型 harness，也能服务 MCP 客户端。

增长背景（客观陈述）：项目 2026-01-11 由 Vercel Labs 创建，处于 2026 年 AI agent 工具生态高速扩张期；依托 npm/Homebrew/skills.sh 多渠道分发（README Install 一节）、Vercel 官方背书与高频迭代（10 个月 118 个版本），10 个月达到 45.8k 星。具体增长动因不做臆断。

## 3. 技术栈与依赖

- **核心语言**：Rust（edition 2021），异步运行时 tokio；单一 crate，无 workspace（`cli/Cargo.toml`）。
- **CDP 通信**：`tokio-tungstenite`（WebSocket）+ `rustls`/`rustls-native-certs`（TLS），直连 Chrome DevTools Protocol，未依赖任何 Playwright/Puppeteer 运行时（依赖清单可证）。
- **安全**：`aes-gcm`（auth vault 加密，`cli/src/native/auth.rs`）、`sha2`/`hmac`；`rust-embed` 把技能文档与 `axe.min.js` 嵌入二进制（`cli/src/native/a11y/`）。
- **其他**：`reqwest`（provider 云端 API/Chrome CVE feed）、`image`（截图处理，release/dev profile 均单独调优 `Cargo.toml`）、`similar`（diff）、`psl`（公域后缀）。
- **外围 Node 侧**：pnpm monorepo，Node >=22；`bin/agent-browser.js` 做平台二进制选择（macOS arm64/x64、Linux x64/arm64 glibc+musl、Windows x64 共 7 个目标，README Platforms）；`packages/dashboard`（Next.js + shadcn/ui 观测面板）、`docs/`（Next.js 文档站）。
- **浏览器**：默认 Chrome for Testing（`install` 命令下载），可附加运行中的 Chrome（`--cdp`）、Lightpanda/Obscura 引擎（`--engine`）、云端 provider（Browserbase、Browserless、Browser Use、Kernel、AgentCore，`cli/src/native/providers.rs`），以及 WebDriver 桥驱动 iOS 模拟器 Mobile Safari（`cli/src/native/webdriver/`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `cli/` | 全部 Rust 源码（单 crate `agent-browser`，62 个 native 模块文件） |
| `cli/src/native/` | daemon、CDP 客户端、actions、snapshot、browser、provider、流式观测等核心 |
| `cli/src/native/cdp/` | CDP 协议层：chrome/lightpanda/obscura 三引擎、发现、Windows 进程管理 |
| `cli/src/native/webdriver/` | WebDriver/Appium 后端：iOS 模拟器 Mobile Safari |
| `cli/src/native/flow/` | 录制流（record once → HTTP 重放首方 API） |
| `cli/src/native/stream/` | WebSocket 实时流 + dashboard 服务 |
| `cli/src/mcp.rs` | stdio MCP server（6651 行，160 个工具） |
| `skill-data/` | 内嵌技能内容（core + electron/slack/webmcp-gen 等专项技能） |
| `skills/agent-browser/` | `npx skills add` 安装的薄发现 stub |
| `packages/` | `dashboard`（观测面板）、`@agent-browser/eve`（eve agent 扩展）、`@agent-browser/sandbox`（Vercel Sandbox） |
| `docs/` | Geistdocs（Next.js）文档站源码，MDX 页面在 `docs/content/docs/` |
| `evals/` | Bun + Vercel AI Gateway 的技能评测（5 类用例） |
| `test/` | Node launcher 测试；`benchmarks/` CLI vs MCP 上下文占用基准 |

组织方式评述：Node 壳（分发/文档/面板）+ Rust 核心（单 crate 大模块文件）的混合 monorepo。单 crate 内按功能拆文件但颗粒度大（`actions.rs` 22418 行、`commands.rs` 7335 行、`e2e_tests.rs` 15713 行），靠模块约定（AGENTS.md）而非 workspace 强制边界。

## 5. 核心架构与关键实现

### 5.1 常驻 daemon 与命令管线（cli/src/native/daemon.rs，1048 行）

首条命令 spawn 一个后台 daemon；Unix 上监听 Unix domain socket，Windows 上用 TCP 并写 `.port` 文件（daemon.rs 第 80-97 行注释与实现）。每个 `--session` 一个独立 daemon + 浏览器；默认空闲 1 小时自动保存状态并退出（`cli/src/session_settings/` 的 `DEFAULT_IDLE_TIMEOUT_MS`）。CLI 前端（`cli/src/main.rs`）每次执行只做：解析命令 → `send_command` 发 JSON 给 daemon → 打印响应，并带 `send_command_with_respawn`（main.rs 第 1894 行）处理 daemon 意外退出后的重生成。这套设计让"每步一条短 shell 命令"的 agent 工作流没有重复的浏览器启动开销。

### 5.2 snapshot 与 durable refs（cli/src/native/snapshot.rs，element.rs）

快照走 CDP `Accessibility.getFullAXTree` 拿完整无障碍树，按 `INTERACTIVE_ROLES`/`CONTENT_ROLES`/表格角色分类过滤（snapshot.rs 第 107-130 行），构建 `RefMap` 把每个元素登记为 `@eN` 引用并锚定 CDP `backendNodeId`（`cli/src/native/element.rs` `RefEntry`）。亮点是"文档身份跟踪"：`RefMap.observe_document` 记录 (session, frame, loaderId)，同文档更新时 ref 保持稳定，文档被替换（iframe 重载、导航）时强制失效——snapshot.rs 顶部回归测试 `durable_refs_invalidate_replaced_iframe_document` 即为此设计。iframe 按同源/跨进程解析各自 effective CDP session（`resolve_ax_session`），选择器模式可用 `DOM.describeNode` 子树匹配把快照范围缩到局部元素。

### 5.3 CLI/MCP 双形态与命令面（cli/src/mcp.rs，commands.rs）

MCP server 是手写 stdio JSON-RPC（无 MCP SDK 依赖，mcp.rs 模块头注释），支持 2024-11-05 至 2025-11-25 四个协议版本，工具列表按 64 个分页。关键取舍：工具调用不重新实现，而是"委托给当前二进制的 `--json` 模式"（mcp.rs 第 1-8 行），从机制上保证 MCP 行为与 CLI 一致；工具常量共 160 个（`agent_browser_open` 到 `agent_browser_skills_get`）。CLI 侧 `cli/src/commands.rs` 手写解析器（含 `ParseError` 分类型报错），flag 强制 kebab-case（AGENTS.md 规约）。仓库还内置 `parity_tests.rs` 用测试锁死两个表面的对齐。

### 5.4 技能分发与安全边界

技能内容用 `rust-embed` 打进二进制，`skills get core --ref <name>` 按需输出单个参考（CHANGELOG 0.41.2 #2310），`evals/cases/` 用 LLM 评测技能加载正确性与上下文占用（context-footprint）。安全面包括：AES-256-GCM auth vault（`native/auth.rs`）、域白名单 `DomainFilter`（`native/network.rs`）、动作确认策略 `ActionPolicy`/`ConfirmActions`（`native/policy.rs`）、WebMCP 页面工具元数据按不可信数据处理（`skill-data/core/SKILL.md` trust-boundaries 一节）、Chrome 过期 CVE 预警（`cli/src/browser_security.rs` 拉 Google release feed）。

## 6. 构建与运行

```bash
# 安装（用户）
npm install -g agent-browser && agent-browser install   # 首次下载 Chrome for Testing
agent-browser open example.com && agent-browser snapshot -i && agent-browser click @e1

# 从源码构建（README Development）
pnpm install && pnpm build:source    # 构建面板 + 原生 CLI
pnpm link --global

# Rust 检查与测试（README / AGENTS.md）
cd cli
cargo fmt -- --check
cargo clippy
cargo test                                        # 单元测试，不需要 Chrome
cargo test e2e -- --ignored --test-threads=1      # 真实 headless Chrome，必须串行
```

发布：手动单 PR 流程（bump 版本 → `pnpm version:sync` → CHANGELOG 写在 release 标记之间），合并后 CI 检测 npm 版本差异自动构建 7 平台二进制并 publish（AGENTS.md "Releasing"，`package.json` build:all-platforms / release 脚本）。

## 7. 测试与工程质量

- 单元测试 1539 个 `#[test]`，e2e 175 个 `#[tokio::test]`（全部 `#[ignore]`，需真实 Chrome，`cli/src/native/e2e_tests.rs` 15713 行，AGENTS.md 列明覆盖面：导航/快照/表单/cookie/tab/域过滤/state 等）。另有 `parity_tests.rs`（CLI/MCP 对齐）与 Obscura 适配器专项回归（AGENTS.md）。
- CI（`.github/workflows/ci.yml`）：version-sync、launcher、docs（production/preview 双矩阵 + 路由契约测试）、rust（fmt + `clippy -D warnings` + `cargo test --profile ci`）、dashboard、sandbox/eve 包，共 8+ job。
- 质量观察：clippy 以 `-D warnings` 门槛执行；release profile 全量 LTO + strip，并为截图路径的 `image`/`png` crate 单独开优化（`cli/Cargo.toml` 注释"Keep image processing real-time"）；文档站有独立测试与路由契约 fixture。整体工程纪律明显高于同体量 0.x 项目平均水平。
- `evals/` 是少见的"技能层"评测设施：用 Claude/Codex 实测 agent 能否正确加载技能并产出正确命令，还能量化 CLI 与 MCP 的上下文占用（`evals/README.md`）。

## 8. 社区与维护状态

- 活跃度极高：10 个月 118 个版本（平均约 2.5 天一版），最新 release v0.41.2 发布于 2026-11-03，最近提交 2026-11-05（gh api）。
- 贡献者 30 人（gh api contributors），CHANGELOG 每版列出贡献者名单，既有 Vercel 员工（如 @ctate）也有外部贡献者（@Railly、@lahfir 等，`CHANGELOG.md` 0.41.2 节）。
- 490 个 open issue，fork 3141；issue 模板为 YAML 表单（`.github/ISSUE_TEMPLATE/`）。
- 发布完全由维护者主导（AGENTS.md："Releases are manual, single-PR affairs"），changelog 口径统一。

## 9. 亮点与风险

**亮点**
1. 工作流为 LLM 原生设计：无障碍树文本 + `@eN` 引用 + 失败即报错，比截图驱动省 token 且更可判定（README、`skill-data/core/SKILL.md`）。
2. daemon + 常驻浏览器 + durable refs 三件套，兼顾每步短命令与元素引用稳定性（daemon.rs、snapshot.rs）。
3. 技能与二进制同版本分发（rust-embed），配 `--ref` 渐进加载与 LLM eval 基础设施，系统性解决 agent 技能漂移与上下文成本（skill-data/、evals/）。
4. CLI/MCP 双表面由架构与测试强制同步，MCP 零 SDK 手写实现、行为委托 CLI（mcp.rs、parity_tests.rs）。
5. 引擎/托管抽象宽：Chrome for Testing、CDP 附加、Lightpanda/Obscura、五家云 provider、iOS Safari（webdriver/、providers.rs）。

**风险/局限**
1. 0.x 阶段迭代极快（118 版），CLI 行为与技能内容可能频繁变动，依赖它的 harness 需跟随升级（CHANGELOG）。
2. 超大单文件（`actions.rs` 2.2 万行、`e2e_tests.rs` 1.6 万行）对社区贡献与审查成本不友好。
3. 功能面扩张快（flow、webmcp、dashboard、多引擎、多 provider），复杂度集中在少数维护者手上；490 个 open issue 积压。
4. 强绑定 Chrome 系 CDP 语义；Firefox/WebKit 主流引擎仅经 WebDriver/iOS 桥部分覆盖（webdriver/ 目录范围）。

## 10. 总结评价

agent-browser 是 2026 年 agent 基础设施竞赛中完成度最高的浏览器入口之一：它把"给 LLM 一个浏览器"拆成了 daemon 常驻、无障碍树快照、`@eN` 引用、版本对齐技能、CLI/MCP 双表面这一整套自洽设计，并用 1539 个单元测试、175 个真实 Chrome e2e 和专门的 LLM eval 支撑其工程质量，10 个月 45.8k 星与 Vercel 的发布纪律相互印证。适合直接给 Claude Code 等 shell 型 agent 装上做网页操作、抓取与回归验证；对想做同类 agent 工具的团队，`cli/src/native/snapshot.rs` 的 durable refs、`mcp.rs` 的"委托 CLI 而非重实现"与 `evals/` 的技能评测方法都值得读源码借鉴。若你需要一个稳定的长期集成件，建议锁版本并关注其 0.x 变更节奏。
