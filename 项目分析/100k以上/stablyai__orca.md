# stablyai/orca 项目分析

> 星数 103459（快照 2026-10-24）｜主语言 TypeScript｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：Orca 是一个开源（MIT）的 Electron 桌面 AI agent 编排器——把 Claude Code、Codex、Grok、OpenCode 等几十个 CLI 编码 agent 并行跑在相互隔离的 git worktree 里，统一管理会话、终端、diff 审查，并配移动端伴侣 App 与远程运行时。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | stablyai/orca | gh api 元数据 |
| 星数 | 103459（快照）；查询时 110166 | gh api |
| 主语言 | TypeScript（src 下 33,327 个 .ts/.tsx 文件） | find 统计 |
| 许可证 | MIT | `LICENSE`、`README.md` |
| 默认分支 | main，最近 push 2026-10-30 | gh api |
| 最新 release | v1.4.244（2026-10-29），几乎每日发版 | gh api releases |
| 产品形态 | 桌面（mac/win/linux）+ 移动伴侣 + 远程 runtime + CLI | `README.md` |

## 2. 项目是什么，解决什么问题

Orca 自称"The AI Orchestrator for 100x builders"（`README.md`）。它解决的问题非常具体：当一个开发者同时驱动多个 AI 编码 agent 时，会话散落在各个终端里、没有隔离、没有统一的完成通知与审查入口。Orca 把这些 agent 全部装进一个桌面 IDE：每个 agent 在自己的 git worktree 中工作，互不踩踏，结果可以对比后合并赢家（`README.md` "Parallel Worktrees" 一节）。

它对 agent 的接入是"终端即一切"哲学：任何能在终端里跑的 CLI agent 都能被 Orca 托管（`README.md`："if it runs in a terminal, it runs in Orca"）。在此之上，Orca 通过两套机制拿到结构化的会话状态：一是各 agent 的 hook 事件经中继归一化（`src/shared/agent-hook-relay.ts`、`src/shared/agent-hook-listener.ts`），二是 Agent Client Protocol（ACP，`src/main/acp/`），从而在 UI 里呈现回合状态、权限请求、用量与限流等。

围绕编排器本体，仓库还包含一整套外围系统：`cloud/` 是让手机与桌面配对的 relay（双方各开一条出站 WebSocket，由 relay 拼接帧，见 `cloud/README.md`）；`mobile/` 是 Expo/React Native 伴侣 App；`src/cli/` 提供 `orca worktree create`、`snapshot`、`click` 等 CLI，反过来让 agent 也能脚本化驱动 Orca；`tests/`、`config/` 则承载了规模惊人的质量门禁。

README 列出的九大特性（`README.md` Features 表）可归纳为四组：编排（Parallel Worktrees、Mobile Companion）、开发底座（Terminal Splits——"Ghostty 级"WebGL 终端与无限分屏、Drag Files to Agents + Monaco 编辑器）、审查链路（GitHub & Linear 原生集成、Annotate AI Diffs——在 diff 行上留评论直接回投给 agent）、自动化与远程（Design Mode——点选真实 Chromium 页面元素即注入 agent 上下文、SSH Worktrees、Orca CLI）。"Also in the box" 里还有 Computer Use（让 agent 操作桌面应用）与用量/限流追踪。

从 README 与功能面判断，Orca 的商业逻辑是开源桌面编排器获客、公司侧提供云 relay 与移动推送等配套服务（`cloud/README.md` 明确 push/connector 的部署由私有仓 `stablyai/orca-cloud` 运维），属于典型的 open-core。

项目由 YC-backed 公司 Stably 主导（topic 含 `yc-backed`，gh api），仓库 2026-03-17 创建，约 7 个月冲到 10 万+ 星，release 节奏接近每日一发（v1.4.239→v1.4.244 共 5 天，gh api releases）。README 明说"we ship daily, so this list is perpetually behind"。

## 3. 技术栈与依赖

- 语言/运行时：TypeScript；Node 24（服务端）、Node 18（兼容启动器/CLI/WSL guest，见 `.github/CONTRIBUTING.md`）；Bun 参与 typecheck 与测试执行（`config/.bun-version`、`pnpm test` 说明）。
- 桌面框架：Electron 43.8.1 + electron-vite 6（`electron.vite.config.ts`、`package.json` devDependencies）。
- UI：React 19.2 + zustand 5 + shadcn 体系 + lucide-react + tiptap 3（富文本）+ Monaco Editor + react-markdown/mermaid（`package.json`）。
- 终端：xterm 6.1 beta 系列（webgl/search/ligatures 等 7 个 addon）+ node-pty 1.1，两者都带本地补丁（`pnpm-workspace.yaml` patchedDependencies：xterm 6 个补丁 + node-pty 补丁）。
- 通信与集成：ws、ssh2（SSH worktree）、@anthropic-ai/claude-agent-sdk、@linear/sdk、zod、i18next/react-i18next（11 个 locale：de/en/es/fr/ja/ko/pt/ru/tr/zh/zh-Hant，`src/renderer/src/i18n/locales/`）。
- 原生层：node-pty、@parcel/watcher、自研 pnpm 子包 `native/windows-registry`、`native/windows-security`，以及 Swift/Rust/Win32 原生组件（`native/computer-use-{macos,windows,linux}`、`native/keyboard-layout-macos` 等）。
- 包管理：pnpm workspace（根 + `mobile/`、`cloud/` 两个独立 lockfile 工作区，`pnpm-workspace.yaml`）。
- 测试：Vitest 5（打补丁）+ Playwright（electron-headless/headful 双 project，`tests/playwright.config.ts`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `src/main/` | Electron 主进程：agent 适配器（claude/codex/grok/cursor/copilot/droid/gemini/kimi/kiro/opencode/pi/qwen-code/zcode/muse…约 30+ 个 provider 目录）、worktree 编排、PTY、WSL、SSH、更新器、凭证（15,339 个文件） |
| `src/renderer/` | React 渲染层：app-shell、components（shadcn ui）、store、i18n、ssh、web |
| `src/shared/` | 主/渲染进程共享逻辑：agent 会话模型、hook 监听与归一化（`agent-hook-listener/`）、类型契约 |
| `src/cli/` | `orca` CLI：worktree/automation/computer/terminal 等命令组 |
| `src/packages/process-host/` | workspace 子包（`@orca/process-host`） |
| `src/wsl-guest/` | WSL 发行版内运行的 guest 程序 |
| `mobile/` | Expo/React Native 伴侣 App（独立 pnpm 工作区、fastlane） |
| `cloud/` | 移动配对 relay：relay-contract 契约包、relay director/cell、push 网关、Slack connector（独立工作区） |
| `native/` | 各平台原生组件：computer-use 三平台实现、windows-registry/security、wayland-clipboard 等 |
| `tests/` | Playwright e2e（802 个 spec）+ 压测工具（`tests/tools/`） |
| `config/` | 全部构建/lint/测试配置与 80+ 个自研门禁脚本（`config/scripts/`） |
| `docs/` | STYLEGUIDE、agent 参考文档、docs 站点（`docs/site/`）、schemas |
| `skills/`、`skill-guides/`、`skill-stubs/` | 随应用分发的 agent skill 包与生成清单 |
| `examples/`、`Casks/`、`resources/` | 示例、Homebrew cask、图标/引导素材 |

组织方式评述：这是一个"根 Electron monorepo + 两个独立工作区（mobile、cloud）+ 原生子包"的多包结构。代码风格极度"单文件单职责"——`src/main` 顶层平铺上万个以领域命名的小模块（如 `worktree-create-preparation-claim-registry.ts`），并禁止 `helpers/utils` 类模糊命名（`AGENTS.md` Code Style 一节）。依赖预算刻意收敛：主包 dependencies 仅 27 个、devDependencies 123 个（`package.json`），自研代码远多于第三方引入。仓库还带有强烈的"agent 开发 agent"自举色彩：根目录 `AGENTS.md`（4.5KB 开发纪律）、`.mcp.json`、`opencode.json`、`.claude/settings.json`、`docs/agents-md-refs/` 参考文档与 `config/scripts/sync-anti-slop-plugin.mjs`（反 AI 垃圾代码 lint 插件）共同构成 AI 协作规范。

## 5. 核心架构与关键实现

### 5.1 Agent 接入层：hook 归一化 + ACP 方言

Orca 不重新实现 agent，而是把异构 agent 的事件流收敛为统一会话状态。hook 路径上，`src/shared/agent-hook-listener.ts` 的 `normalizeHookPayload` 先用 `parseHookEnvelope` 校验带环境密签的信封（防伪造），再按 source（claude/codex/grok/opencode/amp…）路由到 `src/shared/agent-hook-listener/providers/` 下的 per-provider 事件归一化；同一目录内有 20+ 个针对性测试（如 `agent-hook-listener-codex-approval-ownership.test.ts`）。另一条路径是 ACP：`src/main/acp/` 内含连接管理、历史日志重放（`acp-history-catch-up.ts`）、压缩回合、权限协商，并按 agent 提供"方言"（`src/main/acp/acp-dialects/grok-dialect.ts`、`antigravity-dialect.ts`）。

### 5.2 Worktree 编排流水线

`src/main/worktree-create-preparation.ts`（303 行）实现了创建 worktree 的分阶段准备：base 预取（`worktree-create-base-prefetch.ts`）、claim registry 防并发重复（`worktree-create-preparation-claim-registry.ts`）、过期清理与取消。删除侧同样严防死守：`worktree-removal-safety.ts`、`worktree-removal-home-guard.ts`（防误删 home）、`worktree-removal-authority.test.ts`。目录里还支持非 git 的"folder workspace"（`AGENTS.md` First Class Citizens 明确要求两种形态平等对待）。

### 5.3 守护进程与终端持久化

会话由独立的 node runtime daemon（orcad，`pnpm build:orcad` 产出带 prebuilds 的独立二进制，`src/main/orcad/`）承载：桌面 App 退出重启后终端 scrollback 与会话可恢复（`src/main/daemon/cold-restore-payload-cache.ts`、`terminal-scrollback-snapshots.ts`）。daemon 同时是远程/SSH host 与 serve 模式的基础（`src/main/daemon/AGENTS.md`、`serve-update-handoff.ts`）。这与 `AGENTS.md` 中"客户端与服务端独立更新、保持混合版本兼容"的远程用例要求呼应。

### 5.4 移动配对 relay

`cloud/README.md` 描述了完整的安全模型：手机与桌面互不直连，各自向 relay cell 开出站 WebSocket；director 负责 host→cell 分配与迁移；认证基于桌面侧 X25519 密钥的加密挑战；推送走独立的 push 网关（APNs/FCM，桌面先通过挑战换取 24 小时会话再注册 token）。代码即契约：`cloud/packages/relay-contract` 是三端共享的线协议实现。

### 5.5 账号、用量与权限面

Orca 直接管理各 agent 的登录态与限额：`src/main/claude-accounts/`、`codex-accounts/`、`grok-accounts/`、`cursor-accounts/` 支持"热切换账号"（README Account switcher）；`src/main/claude-usage/`、`codex-usage/`、`opencode-usage/`、`muse-usage/` 追踪用量与限流重置；`src/main/rate-limits/` 与 `src/main/credentials/`（含 `protected-secret-persistence.ts`、`credential-file-protection.ts`）处理凭证安全落盘。工作区信任模型独立成体系：`agent-workspace-trust.ts`、`execution-host-workspace-trust.ts` 在 spawn 前做信任判定。

## 6. 构建与运行

（来源：`package.json` scripts、`AGENTS.md`、`.github/CONTRIBUTING.md`）

```bash
# 环境：Node 24 + pnpm（见 packageManager）+ Bun（config/.bun-version）+ 平台 C 工具链
pnpm install
pnpm -C mobile install --frozen-lockfile   # mobile 是独立工作区，需单独装

pnpm dev          # Electron 开发模式（worktree 下自动独立 profile/端口）
pnpm dev:web      # 纯 Web 渲染层开发
pnpm tc           # 并行 typecheck（node/cli/web 三个 tsconfig project）
pnpm test         # Vitest（Bun 运行，node-pty/SQLite 等运行时契约用 Node worker）
pnpm test:node    # 全部在 Node 上跑
pnpm test:e2e     # Playwright electron-headless
pnpm build:mac    # / build:win / build:linux 本地打包
pnpm build:release:parallel   # 发布构建（含 wsl-guest、cli、native、mobile-web）
```

## 7. 测试与工程质量

- 规模：`src/` 内 13,791 个 `*.test.ts`/`*.spec.ts`（与源码同目录就近放置），`tests/e2e/` 802 个 Playwright spec（find 统计）。
- e2e 覆盖面罕见：terminal 渲染 golden（emoji 表格、scrollback 恢复）、IME（ibus/hangul）、Windows crash survival、WSL e2e、macOS/Windows 更新存活、remote bulk-open 冻结复现（`package.json` scripts `test:e2e:*`、`tests/e2e/`）。
- 测试纪律同样制度化为门禁：`check:flaky-test-pattern-ratchet`（禁止 flaky 写法）、`check:test-file-runners`（测试运行器一致性）、ast-grep 专项 `lint:ast-grep:flaky-tests`；`tests/AGENTS.md` 专门规范测试写法，AGENTS.md 明确"避免低价值测试"并链接反模式文档（`docs/agents-md-refs/test-antipatterns.md`）。
- 性能当作契约：`test:perf:contracts`、约 25 个 `bench:*` 基准（idle-cpu、startup、daemon-coldstart、typing latency 等），配 `check-terminal-perf-report-budgets` 预算门禁。
- Ratchet 门禁：约 20 个 `check:*-ratchet` 脚本（max-lines、ts-nocheck、flaky-test-pattern、daemon-blocking-call、id-only-store-write、zustand-selector-fanout 等），防止指标回退（`package.json` scripts）。
- Lint 矩阵：oxlint 多套配置（performance-audit/code-quality/anti-slop/design-system/dead-classes）+ ast-grep + react-doctor + knip 死代码 + import boundary 检查。
- 供应链：`minimumReleaseAge: 4320`（依赖发布 3 天后才能进，`pnpm-workspace.yaml`）、`check-dependency-security`、overrides 顶掉漏洞版本、13 个 patchedDependencies。
- CI：`.github/workflows/` 84 个 workflow，覆盖三平台 e2e、WSL、terminal-perf、各平台 release 与 mobile 发布。

## 8. 社区与维护状态

- 活跃度：最近 push 2026-10-30（gh api）；release v1.4.239→v1.4.244 在 5 天内连续发布（gh api releases），并穿插 `mobile-android-v0.0.60`，呈"每日发版"节奏。
- 贡献者：contributors API 首页满 30（真实总数未确认）；README 挂 contrib.rocks 图；治理以 Stably 公司团队为主（仓库 2026-03 创建即由公司开发）。
- 社区：Discord、X（@orca_build）、微信群（`README.md` Community 一节）；Windows 签名由 SignPath 赞助（README Signed Builds）。
- 待办压力：open issues 7,938（gh api），forks 6,957。docs 站点与 CHANGELOG 以 GitHub releases 为准（README："The changelog is the real feature list"）。

## 9. 亮点与风险

**亮点**

1. Agent 接入层的工程化标杆：hook 信封校验 + per-provider 归一化 + ACP 方言（`src/shared/agent-hook-listener/`、`src/main/acp/`），让 30+ 异构 agent 在同一会话模型下运行，且每个 provider 都有针对性测试。
2. 质量门禁密度极高：ratchet 脚本 + 性能契约 + 25 个基准 + 84 条 CI 流水线 + 3 天依赖冷却期，把"快速每日发版"和"不破坏 main"（`AGENTS.md` 第一条）同时兜住。
3. 全平台纵深真实可查：桌面三平台、WSL guest（`src/wsl-guest/`）、SSH 远程（`src/main/ssh/`）、移动伴侣（`mobile/`）、relay 云（`cloud/`）在仓库中均有对应实现而非 PPT。
4. 反向可编程：`src/cli/` 让 agent 也能驱动 Orca（worktree/snapshot/click/computer-use），形成 agent↔IDE 双向闭环。
5. 多语言与文档自觉：UI 11 个 locale 且有覆盖审计脚本（`verify:localization-coverage`），README 9 种语言。
6. 会话可靠性下过深功夫：退出前的 durable write 阻塞保证（`src/main/quit-path-durable-write-blocking.test.ts`、`quit-teardown-deadline.ts`）、PTY 后代进程终止验证（`src/main/pty-descendant-termination.ts`）、更新重启后的会话保留（`agent-auth-restart-preservation.ts`）——这些是同类工具最容易烂掉的地方。

**风险/局限**

1. 体量与门槛：`src/` 3.3 万个 TS 文件、上万个平铺小模块，外部贡献者理解成本高；AGENTS.md 的严格命名/注释纪律是双刃剑。
2. 重度补丁依赖：xterm 全家 beta + node-pty 等 13 个 patchedDependencies（`pnpm-workspace.yaml`），上游升级维护负担大。
3. 开源边界：push/connector 的部署运维在私有仓 `stablyai/orca-cloud`（`cloud/README.md`），relay 云的自托管完整度取决于文档；README 自述功能列表"永远滞后"。
4. 遥测与账号绑定：内置 posthog-node 遥测与各 agent 账号管理，隐私敏感用户需自行评估（README 提供遥测文档与退出说明）。

## 10. 总结评价

Orca 是 2026 年"AI agent 编排器"赛道里工程完成度最高的开源实现之一：7 个月 10 万+ 星的背后，是"把每个 CLI agent 当作一等公民接入（hook 归一化 + ACP）、用 git worktree 做隔离、用自研 daemon 保证会话比 App 活得久"这条清晰主线，以及一套用 ratchet、性能契约和 84 条流水线堆出来的发布纪律。它适合两类人：想真正并行使用多家编码 agent、并需要移动端与远程监控的重度开发者（直接下载成品）；以及研究"如何工程化地托管异构 agent"的团队（读 `src/shared/agent-hook-listener/`、`src/main/acp/`、`src/main/worktree-*` 与 `config/scripts/` 的门禁脚本，堪称这个细分领域的活教材）。想为其新增 agent 适配或参与核心开发的人，则要有啃下三万文件单体仓库的心理准备。
