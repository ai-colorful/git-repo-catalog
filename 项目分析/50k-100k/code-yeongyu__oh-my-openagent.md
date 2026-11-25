# code-yeongyu/oh-my-openagent 项目分析

> 星数 70506（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

OmO 是一个"开箱即用的全能 AI 编码 agent"：一条 `omo` 命令，把多模型路由、多 agent 图编排（mass ulw）、强制验证循环（ultrawork）、git 仓记忆系统（Kibitzer）打包成一个终端应用。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | oh-my-openagent（前身 oh-my-opencode，npm 包 `omo-ai`） | `package.json`、`docs/manifesto.md` |
| 星数 | 70506（2026-10-31 gh api 实测，快照口径 70290） | gh api |
| 创建时间 | 2025-12-03（不到一年冲到 70k 星） | gh api |
| 主语言 | TypeScript 约 120 万行（9350 个 .ts 文件）+ Rust 约 8.3 万行（597 个 .rs） | 源码统计 |
| 许可证 | SUL-1.0 Sustainable Use License（非 OSI：允许内部业务/个人/非商业免费使用与分发，禁止商业转售或竞争性产品） | `LICENSE.md` |
| 最近活跃 | 默认分支 dev，最近 push 2026-10-31（分析当日）；v5.1.52 发布于 2026-10-29 | gh api、`CHANGELOG.md` |
| 官网 | https://omo.dev（Discord 社区运营中） | `README.md` |

## 2. 项目是什么，解决什么问题

OmO 的出发点写在 `docs/manifesto.md`：**"人工接管是系统失败的信号"**——像自动驾驶一样，如果用户需要反复纠正、返工、手把手引导 agent，那是 harness 设计失败，而不是"人机协作"。因此 OmO 的目标是让 agent 一次性把活干完，产出"与资深工程师无法区分的代码"（Indistinguishable Code）。

它解决的实际痛点有四个。第一，**单个模型/单次对话不可靠**：OmO 用 ultrawork 强制 agent 先探索代码库、列计划、逐步证明再动手（`packages/prompts-core/prompts/ultrawork/default.md` 中是长达 2.2 万字符的确定性协议 prompt，按 GPT/GLM/Gemini/Codex 分别调优）。第二，**大任务超出单 agent 能力**：在提示词里加 `mass ulw`，任务被拆成一张 agent DAG 图并行执行，各节点用最适合的模型（`packages/senpi-task/src/dag/`、协议文档 `docs/reference/mass-ulw-protocol.md`，带 WAL 持久化的 17 种日志事件，支持外部 viewer 实时观察）。

第三，**agent 记不住用户**：Kibitzer 记忆系统把记忆存成 git 仓库里的 markdown 文件，由一个便宜的小模型在主 agent 旁持续阅读历史、在重蹈覆辙前"插嘴"提醒（`README.md`；实现核心在 `packages/memory-core/src/`，含 BM25 检索 `recall/bm25*.ts`、reflection/dream/facts/kibitzer 四种 persona `personas/manifest.ts`、CRDT 同步、审计日志）。第四，**模型选择太累**：`/login` 接入 Claude、ChatGPT、Kimi、GLM 订阅，每个模型配专属 system prompt，任务类别自动路由到合适档位的模型（`packages/model-core/src/` 的 fallback chain 与 category routing）。

项目演进路径值得注意：它起源于 OpenCode 插件生态（`package.json` 仍叫 `oh-my-opencode`，依赖 `@opencode-ai/plugin`），后来 fork 了 badlogic/pi-mono 为自己的引擎 "senpi"（依赖 `@code-yeongyu/senpi 2026.10.30-1`），现在是一个独立的终端 agent 产品，同时保留对 OpenCode/Codex（LazyCodex）的兼容迁移层（`packages/omo-codex/`、`omo setup` 命令）。5.0 版本起主打 "OmO ❤️ Pi"（senpi 运行时）。

## 3. 技术栈与依赖

- **语言**：TypeScript（主体，bun 运行时）+ Rust（桌面控制/原生栈，`Cargo.toml` workspace 12 个 crate）。
- **运行时与构建**：Bun 1.4.4（`bun.lock`、`bunfig.toml`、CI 中 `oven-sh/setup-bun`）；npm 发布时可选 12 个平台的预编译二进制 optionalDependencies（`package.json`，darwin/linux/windows × x64/arm64/musl/baseline）。
- **TUI**：`@opentui/solid` + `@opentui/core`（Solid 风格终端 UI），`node-pty`。
- **引擎**：senpi（pi-mono 的 fork，自维护），MCP 协议 `@modelcontextprotocol/sdk`，zod v4 校验。
- **Rust 侧**：tokio、objc2/ScreenCaptureKit（macOS）、windows-sys/uiautomation（Windows）、x11rb/zbus/atspi（Linux X11/Wayland）、enigo（键鼠注入）、xcap/arboard（截屏/剪贴板）——即 computer use 的全平台原生后端（`Cargo.toml`）。
- **质量工具**：oxlint、knip（死代码检查）、typescript 7（native preview）、proptest（Rust 属性测试）、PostHog 遥测（可关）。

## 4. 目录结构与代码组织

顶层为 bun workspace monorepo（约 50 个包）+ Rust cargo workspace（12 crates）双结构：

| 目录 | 职责 |
| --- | --- |
| `packages/omo-opencode/` | 主插件/CLI 主体：agents 定义、hooks、tools、config、openclaw 集成 |
| `packages/omo-native/` | npm 包 `omo-ai`：standalone `omo` 命令（内嵌 senpi 引擎与 OmO 扩展） |
| `packages/omo-senpi/` | senpi 引擎插件：TUI 组件、task DAG RPC 桥、技能同步 |
| `packages/omo-codex/` | LazyCodex：面向 OpenAI Codex CLI 的安装/兼容层 |
| `packages/senpi-*`（7 个） | 桌面协议/引擎/服务的 TS 侧封装 |
| `packages/memory-core/` | Kibitzer 记忆系统（git 仓存储、BM25 召回、persona、CRDT、审计） |
| `packages/model-core/` | 模型能力快照、fallback 链、上下文溢出处理、类别路由 |
| `packages/lsp-core`、`lsp-daemon`、`ast-grep-mcp`、`git-bash-mcp` | LSP/AST/git 结构化工具（MCP 服务） |
| `packages/rules-engine`、`prompts-core`、`skills-loader-core`、`agents-md-core` | 规则/提示词/技能加载 |
| `packages/shared-skills/` | 18 个内置技能（browser、frontend、data-scientist、ulw-plan/execute/research、visual-qa 等） |
| `packages/web/` | 官网与 Web 界面（有 e2e） |
| `crates/senpi-desktop-*` | Rust：桌面控制引擎，macOS/Win32/X11/Wayland/AT-SPI 五种后端 + 安全层 `senpi-desktop-safety` |
| `crates/omo-engine-trust-helper` | 引擎信任辅助（macOS Security.framework / Windows 注册表，`crates/omo-engine-trust-helper/src/`） |
| `docs/` | 48 篇文档：guide/reference/manifesto/troubleshooting，含中文术语表 `i18n-glossary.md` |
| `.agents/`、`.claude/` | 仓库自用 agent 技能（dogfooding：用 OmO 开发 OmO，如 codex-qa、security-research 技能） |
| `.github/workflows/` | 31 个 CI/CD workflow |

组织评述：包粒度极细（连 comment-checker、hashline、tmux 都独立成包），按"一个能力一个 core 包"划分，主入口 `omo-opencode` 组装；同名的 TS 包与 Rust crate（senpi-desktop-engine）成对出现，TS 通过 daemon/RPC 调 Rust 引擎。

## 5. 核心架构与关键实现

**1) ultrawork 确定性协议**（`packages/prompts-core/prompts/ultrawork/default.md`）
核心思想：把"先想清楚再动手"写成机器可执行的强制协议，未达 100% 确定禁止写代码，并预置了 explore/librarian/oracle 三类委派调用模板：

```
**YOU MUST NOT START ANY IMPLEMENTATION UNTIL YOU ARE 100% CERTAIN.**
**IF YOU ARE NOT 100% CERTAIN:**
1. THINK DEEPLY - What is the user's TRUE intent?
2. EXPLORE THOROUGHLY - Fire explore/librarian agents ...
3. CONSULT SPECIALISTS - Delegate: Oracle (架构/调试), Artistry (非常规)
**ONLY AFTER YOU HAVE ... THEN AND ONLY THEN MAY YOU BEGIN IMPLEMENTATION.**
```

同一协议为不同模型家族维护分版本 prompt（gpt.md 15.6KB、glm.md 12KB、gemini.md 20.9KB），配合 `packages/prompts-core/prompts/atlas|prometheus|mode/` 形成编排层提示词体系。

**2) 多 agent DAG 编排（mass ulw）**（`docs/reference/mass-ulw-protocol.md`、`packages/senpi-task/src/dag/`）
运行轨迹以事件账本形式持久化：`omo.dag.event`（顺序化+WAL 持久化）、heartbeat、activity、updated 四个推送通道，17 种日志事件类型，schemaVersion=1 强制版本协商；配套外部 viewer（README 中的 omo-herdr-dag）可只凭该文档实现。这使长任务重启后能从账本恢复，也是 README 所说"重启的会话从停下的地方继续"的机制基础。

**3) Kibitzer 记忆系统**（`packages/memory-core/src/`）
约 30 个子模块：`git/`（记忆即 git 仓）、`recall/bm25-index.ts`（BM25 全文召回，含英文词干化 `english-stem.ts`）、`personas/`（reflection/dream/facts/kibitzer 四种小模型 persona，prompt 资产由 `manifest.ts` 集中登记防改名失联）、`crdt/` 与 `journal/`（并发与账本）、`receipts/`（执行凭证，CHANGELOG 5.1.52 提到 claim receipts 防止测试造假）。记忆支持 frontmatter `expires:` 过期注入（CHANGELOG #11458），kibitzer 记录与引擎共享同一套密钥脱敏（#11556）。

**4) 模型路由与兜底**（`packages/model-core/src/`）
`fallback-chain-from-models.ts` + `category-model-requirements.ts` 按任务类别（如需要强逻辑/强文笔/便宜批量）维护模型链与降级策略；`context-overflow.ts` 处理上下文溢出；`model-capabilities/` 有随仓库快照的能力表（脚本 `script/build-model-capabilities.ts` 定期重建，CI 有 `refresh-model-capabilities.yml`）。CHANGELOG 中"Kimi K3.1 Pro 成为 Kimi 链默认模型"（#11451）即体现在此层。

## 6. 构建与运行

来自 `package.json` scripts 与 README（本项目要求静态阅读，未实际运行）：

```bash
# 安装（用户）
curl -fsSL https://get.omo.dev/install.sh | bash   # 或 irm .../install.ps1 | iex
bun add -g omo-ai                                   # 或 npm i -g omo-ai

# 开发（仓库内）
bun install
bun run build            # script/build.ts
bun run build:all        # 构建 + 平台二进制（script/build-binaries.ts）
bun run typecheck        # bun check 全 workspace + script + packages
bun test --timeout 20000 # 全量测试（bun test）
bun run test:fast        # script/test-fast.ts
bun run test:codex       # LazyCodex 兼容层专项
bun run test:desktop     # Rust/TS 桌面栈联动测试
bun run lint             # oxlint（另有 lint:ratchet 基线棘轮）
bun run knip:check       # 死代码检查
```

## 7. 测试与工程质量

- 测试规模惊人：**4009 个 `*.test.ts`，约 70.9 万行测试代码**（与 120 万行源码近 1:1.7），Rust 侧用 proptest（`Cargo.toml`）。文件级证据：几乎每个源文件旁都有同名 `.test.ts`（如 `packages/memory-core/src/recall/bm25.test.ts`）。
- 风格：单元为主 + 契约测试（protocol/schema）+ e2e（`packages/web/e2e/`，自研 omowright 驱动）+ 仓库自用 QA 技能（`.agents/skills/codex-qa/`、`opencode-qa/`，含 TUI 冒烟、LSP e2e、hermetic home 测试脚本 `test-hermetic-home.ts`）。
- CI：31 个 workflow（`.github/workflows/`），`ci.yml` 支持 merge group 与 lint-ratchet（只允许基线下降被修复，不允许恶化）；专项流水线覆盖 desktop 各平台 QA、windows-flake-soak、macos-signing-canary、installer-mirror 等。
- 工程纪律突出：`CHANGELOG.md` 每条变更都带 PR/issue 编号与实测数据（如 "LSP 会话 4 进程降为 2，292MiB vs 597MiB"，#11766）；knip + lint ratchet + `check:copy` 文案校验；CHANGELOG/文档拼写错误会被 CI 拦截。

## 8. 社区与维护状态

- **极度活跃**：gh api 显示最近 push 为分析当日（2026-10-31）；v5.1.45→v5.1.51 五个 release 集中在 2026-10-24 至 10-28 五天内，平均一天一个 patch 版本（gh api releases）。
- **贡献者**：333 名（gh api contributors 分页推算）；CHANGELOG 中可见社区贡献者署名致谢（@MoerAI、@AgentSMLA 等），外部 PR 有汇总批次（#11555）。
- **议题积压大**：open issues 2011、forks 5823（gh api）——星涨太快，issue 处理压力可见；README 主推应用内 `/bug` 上报（自动附带脱敏日志与回放），并明确列出遥测包含哪些身份字段，GitHub issues 作为兜底。
- **商业化与治理**：个人 side project 起家（README 自述），OpenGateway 赞助；`CLA.md` 存在，license 从 MIT 基座改为 SUL-1.0（禁止商业竞争性使用），Rust crates 子树单独标 MIT（`Cargo.toml` license=MIT）。
- 文档本地化覆盖 10 种语言 README（`README.md` 头部链接），docs 含法律子目录 `docs/legal/`。

## 9. 亮点与风险

**亮点**
1. 工程化程度在开源 agent 项目中属第一梯队：120 万行源码配 71 万行测试、31 条流水线、lint 棘轮、knip、带实测数据的 CHANGELOG（`.github/workflows/`、`CHANGELOG.md`）。
2. ultrawork/mass-ulw 把"方法论"落成可测试的协议与数据结构（DAG 事件账本 + WAL + 外部 viewer 协议），而不是停留在 prompt 玄学（`docs/reference/mass-ulw-protocol.md`）。
3. 记忆系统设计完整：git 仓存储 + BM25 召回 + persona 分工 + 过期/脱敏/审计，是同类项目少见的深度（`packages/memory-core/src/`）。
4. 全平台原生栈自研：Rust 五后端桌面控制（macOS/Windows/X11/Wayland/AT-SPI）+ 12 平台预编译二进制分发，安装体验一条命令（`crates/`、`package.json` optionalDependencies）。
5. dogfooding 文化：仓库自用技能（codex-qa、security-research、github-triage）与自家 CI 共存，说明产品自己吃自己的狗粮（`.agents/skills/`）。

**风险/局限**
1. License 不是 OSI 开源：SUL-1.0 禁止商业转售与竞争性产品使用，企业商用需评估合规（`LICENSE.md`）；open issues 2011 偏高，个人项目主导的维护结构存在单点风险（作者账号故事见 README 的 X 账号被封段落）。
2. 体量与复杂度陡峭：50 个 TS 包 + 12 个 Rust crate、自研引擎 fork（senpi）、自研测试框架（omowright），外部贡献者理解成本高；多套兼容层（OpenCode/Codex/LazyCodex）长期并存增加维护面（`packages/omo-codex/`）。
3. 强 opinionated 设计（固定 agent 人格、固定 prompt 协议、telemetry 默认开）对想自定义的团队不友好，尽管配置面（`docs/reference/configuration.md`、`omo.jsonc`）在补齐。
4. 版本节奏极快（日均 release），下游跟随者（如依赖其 npm 包做集成）需要频繁升级。

## 10. 总结评价

OmO 是 2025 年末至 2026 年 "agent harness 军备竞赛" 中跑得最快的开源项目之一：不到一年从零到 70k 星，靠的是把 Claude Code 类产品的成功要素（强协议 prompt、多模型路由、后台并行 agent、持久记忆）做成一个安装即用、跨三平台、有真实工程纪律的产品，并用 Rust 原生栈补齐了 computer use 这块硬骨头。它不是教学型代码库——代码量与包粒度对读者不友好——但对想研究"生产级 agent harness 长什么样"的工程师，它的 DAG 事件协议、记忆系统与测试金字塔是极好的活教材；对想"今天就用 AI 干完活"的终端用户，它是 Cursor/Claude Code 之外最完整的开源替代之一。使用前请确认 SUL-1.0 许可与你的商业场景兼容。
