# Yeachan-Heo/oh-my-claudecode 项目分析

> 星数 40810（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：Claude Code 的"全家桶式"多 agent 编排框架——以官方插件形态（agents + skills + commands + hooks + MCP）注入 Claude Code，提供 Team 多智能体协作、19 个专业化子 agent、85 个技能与 tmux 多 CLI 工人编排，口号是"Don't learn Claude Code. Just use OMC."（README.md）。

| 事实 | 值 | 出处 |
| --- | --- | --- |
| 名称 | oh-my-claudecode（npm 包名 `oh-my-claude-sisyphus`，v5.8.6） | package.json、README.md |
| 星数 / fork | 40810 / 3645（2026-11-14 gh api） | gh api repos/Yeachan-Heo/oh-my-claudecode |
| 语言 | TypeScript（ESM，Node 20-26） | package.json |
| 许可证 | MIT（Copyright 2025 Yeachan Heo） | LICENSE |
| 创建时间 | 2026-01-09，10 个月达 40.8k 星 | gh api |
| 最近活跃 | pushed 2026-11-13（分析前一天仍在推送）；v5.8.0→v5.8.4 三天内连发 5 版 | gh api、GitHub Releases |
| 贡献者 | 约 167 人（contributors 分页计） | gh api contributors |

## 2. 项目是什么，解决什么问题

oh-my-claudecode（社区简称 OMC）解决的是 Claude Code 单 agent 上下文有限、无内建团队协作的问题。它不是独立的 agent 运行时，而是寄生在 Claude Code 之上的编排层：用户照常使用 Claude Code，OMC 通过插件机制把" orchestrator（主控 agent）→ 专业子 agent → 工人进程"的三层结构注入会话（README.md "Why oh-my-claudecode?"）。灵感来自 oh-my-opencode，另有姊妹项目 oh-my-codex（面向 Codex CLI）和更轻量的 gajae-code（README.md 首页链接区）。

核心场景一：**Team 模式（作者推荐的规范面）**。`/team 3:executor "fix all TypeScript errors"` 把任务分解为共享任务清单，派发给 N 个具名 teammate，按五阶段流水线推进：`team-plan → team-prd → team-exec → team-verify → team-fix(循环)`（skills/team/SKILL.md、README.md Team Mode）。可选用 Claude Code 2.1.178+ 的实验性原生 agent teams（`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`），OMC 保留自研的 tmux/CLI 工人编排作为兼容路径。

核心场景二：**多 CLI 混合编排**。`omc team 2:codex "review auth flow"` 会在 tmux 分屏里拉起真实的 Codex/Gemini/Antigravity/Grok/Cursor/Claude CLI 进程作为工人，按需拉起、完工即退（README.md "tmux CLI Workers"）。这使 OMC 超出了"Claude Code 插件"范畴，成为跨厂商 CLI agent 的调度器。

核心场景三：**自治执行与持久化**。`/autopilot`、`/execute`、`/ralph`、`/verify` 等 85 个技能覆盖从苏格拉底式需求访谈（/deep-interview）到证据化验收（tests/build/lint/typecheck 全绿才算完成）的完整生命周期；HUD 状态栏实时展示编排指标（README.md Features、src/hud/）。

工程上的显著特征是**把"编排"做成软件而非纯提示词**：hooks.json 注册了 11 类 Claude Code 生命周期事件（UserPromptSubmit/SessionStart/PreToolUse/PostToolUse/SubagentStart/Stop/PreCompact/Stop/SessionEnd 等），由 40+ 个 Node 脚本执行关键词检测、技能注入、git 护栏、交付物校验、预算守卫等确定性逻辑（hooks/hooks.json）；再叠加 MCP 服务器暴露 18+ 个 LSP/AST/REPL 工具（src/mcp/omc-tools-server.ts）。提示词只负责意图与决策，硬约束交给 hook 与脚本——这是它区别于多数"纯 markdown agent 集"项目的关键。

## 3. 技术栈与依赖

- 语言/运行时：TypeScript 5.7（strict，tsconfig.json），ESM，Node 20-26（package.json engines）
- 关键依赖（package.json dependencies）：
  - `@anthropic-ai/claude-agent-sdk`：以 SDK 形态暴露 `createOmcSession()` 等编程接口，并用 `createSdkMcpServer` 注册进程内 MCP 工具（src/index.ts、src/mcp/omc-tools-server.ts）
  - `@modelcontextprotocol/sdk`：独立 MCP 服务器（.mcp.json 指向 bridge/mcp-server.cjs，由 esbuild 打包，scripts/build-mcp-server.mjs）
  - `better-sqlite3`：会话/项目记忆的本地存储（package.json、README 已知告警 #2913）
  - `@ast-grep/napi`：AST 结构化检索工具（src/tools/ast-tools.ts）
  - `vscode-languageserver-protocol`：LSP 工具实现（12 个 LSP 工具，src/tools/lsp-tools.ts）
  - `ajv`/`zod`：schema 校验；`commander`：CLI；`tsx`：脚本直跑
- 构建：tsc 编译 + 7 个 esbuild 打包脚本（cli/mcp-server/team-server/skill-bridge/runtime-cli/claude-md-coordinator 等），外加 prompt-ssot、inventory-graph 等 10 余个代码生成/校验脚本（package.json scripts，共 92 个构建脚本于 scripts/）
- 测试：Vitest 4（vitest.config.ts，排除 tests/perf），含 fast-check 属性测试

## 4. 目录结构与代码组织

| 目录 | 职责 | 出处 |
| --- | --- | --- |
| `.claude-plugin/` | Claude Code 插件清单：plugin.json（注册 85 skills + commands + mcpServers）、marketplace.json | .claude-plugin/ |
| `agents/` | 19 个子 agent 的提示词定义（markdown + frontmatter，带 model 路由与 disallowedTools） | agents/architect.md 等 |
| `skills/` | 85 个技能目录（SKILL.md），从 plan/execute/verify 到 ralph/autopilot/teach/wiki | skills/ |
| `commands/` | 21 个斜杠命令（omc-setup、hud、remember 等） | commands/ |
| `hooks/` | hooks.json（11 类事件）+ jev-compaction.mjs | hooks/ |
| `src/team/` | Team 编排核心，171 个文件；tmux 会话、工人契约、邮箱通信、合并编排、恢复 saga | src/team/ |
| `src/mcp/` | MCP 服务器：omc-tools-server（18+ 工具）、team-server、standalone-server | src/mcp/ |
| `src/agents/` | agent 定义的 TS 层（definitions.ts 汇总、prompt-ssot 单一事实源） | src/agents/ |
| `src/cli/` | `omc` 子命令实现（ask/team/autoresearch/checkpoint/graph 等） | src/cli/ |
| `src/tools/` | LSP/AST/python-repl/skills/state/notepad/memory/trace/wiki 工具 | src/tools/ |
| `src/hud/` | 状态栏 HUD（agent-kind、mission-board、background-tasks 等） | src/hud/ |
| `src/{workflow,goal-workflows,planning,graph,interop,...}` | 工作流引擎、目标工作流、规划、依赖图、互操作等约 35 个模块 | src/ |
| `scripts/` | 92 个构建/生成/校验脚本 | scripts/ |
| `docs/` | 65 个文档（REFERENCE、MIGRATION、ADR、TEAM-WORKTREE-MODE 等） | docs/ |
| `tests/` | 顶层测试（75 个文件，约 2.2 万行；另有 806 个内嵌 `__tests__` 目录于 src/ 内） | tests/ |

组织方式评述：单仓单包（非 monorepo），但内部按"提示词资产（agents/skills/commands/hooks）+ TS 运行时（src/）+ 构建产物（bridge/*.cjs、dist/）"三层切分。规模罕见地大：src/ 下 1422 个 TS 文件、约 47.9 万行；最大单文件 runtime-v2.ts 达 332KB、tmux-session.ts 202KB（src/team/ 实测）。

## 5. 核心架构与关键实现

### 5.1 与 Claude Code 的五路集成（插件机制为主）

OMC 不是通过单一机制接入，而是同时占用 Claude Code 的全部扩展面（.claude-plugin/plugin.json、hooks/hooks.json、.mcp.json）：

1. **Skills/Commands/Agents**：plugin.json 注册 85 个 skills、commands/ 目录、agents/ 的 19 个子 agent——纯提示词面；
2. **Hooks**：11 类生命周期事件挂 40+ 个确定性 Node 脚本，例如 PreToolUse 的 `pre-tool-enforcer.mjs`、Stop 的 `persistent-mode.mjs`（防半途而废）与 `budget-guard.mjs`（预算守卫）、SubagentStop 的 `verify-deliverables.mjs`（交付物校验）；
3. **MCP**：`.mcp.json` 声明 `t` 服务器（bridge/mcp-server.cjs），esbuild 打包为单文件分发给插件缓存（scripts/build-mcp-server.mjs）；
4. **npm CLI**：`omc`/`oh-my-claudecode`/`omc-cli` 三个 bin（package.json bin），覆盖安装（omc setup）、终端版 team、跨 CLI 咨询（omc ask）；
5. **Agent SDK 库**：包导出 `createOmcSession()`、prompt 扩展与 `./team` 子路径入口，供 Node 程序编程集成（src/index.ts、src/team/index.ts）。

### 5.2 Team 编排内核（src/team/，171 文件）

最重的模块。两种运行时并存：in-session 原生团队（复用 Claude Code Agent/Task 工具，skills/team/SKILL.md）与 tmux CLI 工人（`omc team`）。后者围绕"契约 + 邮箱 + 哨兵"组织：

- `model-contract.ts`（27KB）：抽象各 CLI（claude/codex/gemini/antigravity/grok/cursor）的启动参数、输出解析与环境变量，`getContract()` 是单一事实源；
- `dispatch-queue.ts`（27KB）+ `allocation-policy.ts`：任务分解入队、按 lane 意图分配；`role-router.ts` 用正则关键词表把任务文本映射到 implementation/verification/review/debug/design/docs 等 lane，再路由到对应角色并给出置信度；
- `mcp-comm.ts`（28KB）+ `leader-inbox.ts`/`conflict-mailbox.ts`：领导与工人经 MCP 信箱通信（v4.4.0 起废除 Codex/Gemini 专用 MCP server，统一走 CLI-first）；
- `git-worktree.ts`（28KB）+ `merge-orchestrator.ts`（35KB）：每个工人独立 worktree/分支，收尾经两轴评审门禁合并到 integration branch（GLOSSARY.md "Integration branch"）；
- 可靠性层：`worker-launch-ack.ts`（146KB，启动确认）、`recovery-saga.ts` + `task-recovery-checkpoint.ts`（崩溃恢复）、`team-owner-epoch.ts`（所有权纪元）、`idle-nudge.ts`（空闲催促）、`sentinel-gate.ts`（就绪哨兵）。

角色分工由 19 个 agent 定义承载，模型分层路由是核心卖点：7 个 opus（architect/critic/planner 等重推理）、10 个 sonnet（executor 等）、2 个 haiku（轻量任务），且 architect 声明 `disallowedTools: Write, Edit` 强制只读（agents/*.md frontmatter）。

### 5.3 判断点与 Jev 决策模型（hooks + GLOSSARY）

OMC 把编排中的每个"模糊判断"显式建模为 judgment point（模式/技能触发、意图识别、模型分层路由、循环继续、上下文剪枝、ralph 完成裁决等 8 处），每处都有启发式实现 + Jev 实现双轨：Jev 是 TypeSafe 的结构化决策 API（api.typesafe.ai），先以 shadow 模式并行收集一致率/延迟数据，达标后才翻转 gate（GLOSSARY.md "Jev integration"）。这是把"提示词工程"向"可评测决策系统"演进的少见设计。

### 5.4 Prompt SSOT 与生成式一致性

agent 提示词采用单一事实源构建：`npm run prompt-ssot:build` 从 src/agents/prompt-ssot/ 生成投影，`--check` 在 CI 中校验漂移；inventory-graph、skill-entitlements、prompt-projections 等生成物全部带 --verify 校验脚本（package.json scripts、src/agents/prompt-ssot/）。说明作者把 85 个技能的提示词当"编译型资产"管理，而非手工维护。

## 6. 构建与运行

```bash
# 安装（二选一）
/plugin marketplace add https://github.com/Yeachan-Heo/oh-my-claudecode   # Claude Code 内
/plugin install oh-my-claudecode
npm i -g oh-my-claude-sisyphus@latest                                     # 终端 CLI（注意 npm 包名不同）

# 构建（完整链：contained-fs → entitlements → tsc → 多个 esbuild 打包 → 文档合成）
npm run build
npm run dev:full        # 并行 watch（tsc + cli + mcp + skill-bridge + runtime + team）
npm test                # vitest（排除 tests/perf）
npm run test:coverage
npm run lint            # eslint src
omc setup               # 会话内等价 /omc-setup
omc team 2:codex "review auth module"   # tmux 多 CLI 工人
```

均来自 package.json scripts 与 README.md Quick Start。要求 tmux（CLI team 面）；autopilot 命名 workflow 仅支持 Linux flock（README.md）。

## 7. 测试与工程质量

- 测试规模：src/ 内嵌 806 个 `__tests__` 目录 + tests/ 顶层 75 个文件约 2.2 万行，Vitest；含单元、Windows 集成（cli-detection.windows.integration.test.ts）、行为快照（behavior-snapshot.test.ts）、fast-check 属性测试与 tests/perf 性能基准（src/team/__tests__/、vitest.config.ts）。
- CI：ci.yml（lint/typecheck/test，push+PR+tag），另有 pr-check、release、upgrade-test、auto-label、stale 等 8 个 workflow（.github/workflows/）。
- 工程质量观察：亮点是生成物全部可校验（十余个 --verify 脚本防漂移）、GLOSSARY.md 把术语当"法律"管理；隐患是巨型文件——runtime-v2.ts 332KB、worker-launch-ack.ts 146KB、scaling.ts 98KB，可维护性存疑。

## 8. 社区与维护状态

- 活跃度极高：创建于 2026-01-09，10 个月 40.8k 星；最近 5 个 release 集中在 2026-11-08 至 11-10 三天内（v5.8.0→v5.8.4），最新 v5.8.6，push 到分析前一日（gh api releases/pushed_at）。
- 贡献者约 167 人；README 列出核心维护者（创建者 Yeachan Heo）、大使（sigridjineth）、文档专员与 Top Collaborators（JunghwanNA 65 commits、riftzen-bit 52 等），并有 11 种语言的 README 翻译（README.md）。
- 社区配套：Discord 服务器、GitHub Sponsors、"Featured by OmC Contributors" 板块；配套文档站点 yeachan-heo.github.io/oh-my-claudecode-website（README.md 头部）。
- 版本节奏：CHANGELOG.md 仅保留近 7 版，格式带 PR 编号与统计行（"5 PRs merged | 3 features | 4 fixes"），显示流水线化发布。

## 9. 亮点与风险

亮点：
1. **占用 Claude Code 全部扩展面的编排架构**——hooks 承担硬约束、MCP 承担工具、agents/skills 承担提示词，工程与提示词分层清晰（hooks/hooks.json、.claude-plugin/plugin.json）；
2. **跨 CLI 工人编排**：一份契约抽象 6 家 CLI agent，tmux 面板即算力，按需拉起（src/team/model-contract.ts、README Team Mode）；
3. **可靠性工程罕见地重**：启动确认、恢复 saga、所有权纪元、交付物校验、预算守卫，把 agent 编排当分布式系统做（src/team/）；
4. **判断点双轨 + 影子评测**：启发式与 Jev 决策 API 并行，有 promotion evidence 机制，可量化地替换提示词判断（GLOSSARY.md）；
5. **提示词编译化管理**：SSOT + 投影 + --verify，杜绝提示词与代码漂移（src/agents/prompt-ssot/、package.json）。

风险/局限：
1. **体量与复杂度失控征兆**：47.9 万行 TS、单文件 332KB，新人读源码与贡献门槛极高；版本号已到 v5.8.6 但 CHANGELOG 只留 7 版，历史不可考；
2. **强环境耦合**：tmux、Linux flock（autopilot workflow）、better-sqlite3 原生模块，Windows 原生体验受限（README Requirements、已知告警 #2913）；
3. **依赖 Claude Code 上游快速变化**：如 2.1.178 移除原生 TeamCreate/TeamDelete 迫使 OMC 改用实验环境变量（skills/team/SKILL.md），上游一变即需紧急跟进；
4. **实验性面多**：实验 agent teams 环境变量、外部 CLI（antigravity/grok/cursor）皆非稳定契约，README 大量篇幅在澄清"哪些面是实验/哪些已废弃"，认知负担重。

## 10. 总结评价

oh-my-claudecode 是 2026 年 Claude Code 生态爆发期最具代表性的"重编排"框架：它把 oh-my-zsh 式的开箱即用理念带进 AI 编码工具，用官方插件 + 全生命周期 hooks + MCP 工具 + tmux 多 CLI 工人堆出一个接近分布式系统的 agent 编排栈，10 个月 40.8k 星既是命名红利与生态时机的产物，也与其"把编排做成可测试软件"的工程认真度相称。代价是极端的体量与对上游的深度耦合。适合：想直接获得多 agent 团队能力的 Claude Code 重度用户（装完即用）、研究 agent 编排工程化（判定点/影子评测/恢复 saga）的工程师读源码；不适合：想要轻量极简集成的用户（作者自己的 gajae-code 就是官方认领的"更轻"替代），或需要长期稳定 API 的二次开发者——大量接口仍处实验态。与同类相比，ccstatusline 只做状态栏（OMC 的 HUD 是其超集场景），claude-flow 走独立引擎 + 自有 MCP 路线，OMC 则选择寄生并放大宿主能力，三者代表了编排框架的三种架构取向（对比基于各项目公开定位，未逐行核对外部代码）。
