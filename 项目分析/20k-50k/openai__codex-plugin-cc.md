# openai/codex-plugin-cc 项目分析

> 星数 26000（快照 2026-10-24，核实日 2026-11-21 实测 35317）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

**一句话定位**：OpenAI 官方出品的 Claude Code 插件——在 Claude Code 工作流里直接调用本地 Codex CLI 做「第二意见」代码评审、把任务委托给 Codex、甚至把整个 Claude 会话移交到 Codex 中继续。

| 事实 | 值 |
|---|---|
| 名称 | openai/codex-plugin-cc（`@openai/codex-plugin-cc` v1.0.6） |
| 简介 | "Use Codex from Claude Code to review code or delegate tasks."（gh api 核实） |
| 主语言 | JavaScript（gh api 核实；纯 ESM 脚本，无编译产物） |
| 许可证 | Apache-2.0（`LICENSE`、`NOTICE`，版权方 OpenAI） |
| 创建时间 | 2026-03-30（gh api） |
| 最近 push/release | 2026-07-08（v1.0.6，gh api） |
| 体量 | 158 KB，57 个文件，核心代码约 5.6k 行 |
| 贡献者 | 13 人（gh api）；open issues 543、forks 2456 |

## 2. 项目是什么，解决什么问题

这是 Anthropic Claude Code 与 OpenAI Codex 两大编码智能体工具之间的**官方官方桥梁**（README.md: "Use Codex from inside Claude Code for code reviews or to delegate tasks"）。它不是用 API 直接拼凑的第三方集成，而是 OpenAI 维护的正式插件，通过 Claude Code 的插件系统（marketplace / commands / agents / hooks / skills）注入斜杠命令，底层封装的是本机安装的 Codex CLI 及其 app-server 进程（README.md "Codex Integration" 一节明确 "wraps the Codex app server... uses the global codex binary"）。

解决的核心痛点有三个。其一，**交叉评审**：Claude 写完代码后用 `/codex:review` 让 Codex 以独立视角做只读评审，或用 `/codex:adversarial-review` 让 Codex 主动挑战设计决策（挑战假设、权衡、竞态、回滚方案），形成"一个模型实现、另一个模型把关"的 workflow。其二，**任务委托/救援**：`/codex:rescue` 把卡住的调试、修复任务经 `codex-rescue` subagent 转发给 Codex 执行（可指定 `--model`、`--effort`，别名 `spark` 映射到 `gpt-5.3-codex-spark`，见 codex-companion.mjs:72）。其三，**会话迁移**：`/codex:transfer` 把当前 Claude Code 会话的 JSONL transcript 通过 Codex 的 external-agent session importer 导入为 Codex 线程，打印 `codex resume <session-id>` 让用户在 Codex App/TUI 里无缝续接上下文（lib/claude-session-transfer.mjs 强制校验源文件必须在 `~/.claude/projects` 下）。

运行模型上它刻意"零依赖"：插件本体不含任何 npm 运行时依赖（package.json 仅有 typescript 与 @types/node 两个 devDependencies），全部逻辑是 Node >=18.18 原生 ESM 脚本，通过 JSON-RPC 驱动用户已有的 `codex` 二进制。认证、模型、`config.toml`（`~/.codex/config.toml` 与项目级 `.codex/config.toml`）全部复用本地 Codex 现状（README FAQ 三连问都在解释这一点）。用量计入用户的 ChatGPT 订阅或 OpenAI API key。

值得注意的还有一个可选的 **Stop 评审门**（review gate）：`/codex:setup --enable-review-gate` 启用后，Claude 每次要结束回合时由 `Stop` hook 拦截，强制对上一轮改动跑一次 Codex 评审，产出 `ALLOW:`/`BLOCK:` 裁决，BLOCK 则阻止停止让 Claude 先修复（stop-review-gate-hook.mjs:79-96）。README 同时警告这会形成 Claude/Codex 循环、快速消耗用量。

## 3. 技术栈与依赖

- **语言/运行时**：纯 JavaScript ESM（`.mjs`），Node >= 18.18（package.json engines）。零运行时依赖，只用 node: 内置模块（child_process/net/fs/crypto/readline 等）。
- **类型**：TypeScript 仅作类型检查（`tsconfig.app-server.json`，`allowJs+checkJs+noEmit`）；类型有两处来源：手写的 `lib/app-server-protocol.d.ts`（75 行）+ 构建时由 `codex app-server generate-ts` 生成的 `.generated/app-server-types`（package.json prebuild 脚本）。
- **底层协议**：JSON-RPC over JSONL（stdio 或 unix socket/named pipe），对接 Codex app-server 的方法包括 `thread/start`、`turn/start`、`review/start`、`turn/interrupt`、`thread/compact/start` 与 `externalAgentConfig/import`（app-server-broker.mjs:12、lib/codex.mjs:733/1026/1070）。
- **测试**：Node 内置 test runner（`node --test tests/*.test.mjs`）。
- **CI**：GitHub Actions，PR 触发，Node 22 + `npm ci` + 全局安装 `@openai/codex` + `npm test` + `npm run build`（.github/workflows/pull-request-ci.yml）。
- **上游依赖（外部）**：本机 `codex` CLI（`npm install -g @openai/codex`），插件通过 `getCodexAvailability`/`getCodexAuthStatus` 探测其存在与登录态（lib/codex.mjs）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `.claude-plugin/marketplace.json` | Claude Code 插件市场清单，声明 `openai-codex` 市场与唯一的 `codex` 插件 |
| `plugins/codex/.claude-plugin/plugin.json` | 插件清单（name/version/author） |
| `plugins/codex/commands/*.md` | 8 个斜杠命令定义（review、adversarial-review、setup、rescue、transfer、status、result、cancel），frontmatter 声明参数提示、allowed-tools |
| `plugins/codex/agents/codex-rescue.md` | `codex-rescue` subagent 定义——"thin forwarding wrapper"，只做一次 `task` 转发 |
| `plugins/codex/hooks/hooks.json` | 注册 SessionStart/SessionEnd（5s 超时）与 Stop（900s 超时）三个 hook |
| `plugins/codex/scripts/codex-companion.mjs` | 主 CLI 入口（1073 行）：setup/review/adversarial-review/task/transfer/status/result/cancel 子命令 |
| `plugins/codex/scripts/app-server-broker.mjs` | 常驻 broker 进程：多调用方共享一个 app-server 实例 |
| `plugins/codex/scripts/lib/codex.mjs` | 核心库（1219 行）：app-server 编排、线程/turn 捕获、评审执行、会话导入 |
| `plugins/codex/scripts/lib/app-server.mjs` | JSON-RPC 客户端（354 行）：stdio/socket 双传输、broker 会话复用 |
| `plugins/codex/scripts/lib/state.mjs` | 每工作区状态持久化：state.json + jobs 目录（上限 50 个 job） |
| `plugins/codex/scripts/lib/tracked-jobs.mjs` | 后台 job 记录、日志文件、进度事件 |
| `plugins/codex/scripts/lib/git.mjs` | 评审目标解析：working-tree vs 分支 diff、收集评审上下文 |
| `plugins/codex/scripts/lib/render.mjs` | 各子命令输出渲染（465 行） |
| `plugins/codex/scripts/lib/claude-session-transfer.mjs` | Claude transcript 路径解析与安全校验（44 行） |
| `plugins/codex/skills/` | 3 个内部 skill：codex-cli-runtime（companion 调用契约）、codex-result-handling、gpt-5-4-prompting（含提示词反模式/配方参考文档） |
| `plugins/codex/prompts/` | 评审提示词模板：adversarial-review.md、stop-review-gate.md |
| `plugins/codex/schemas/review-output.schema.json` | 评审输出的 JSON Schema（verdict: approve/needs-attention + findings 数组） |
| `scripts/bump-version.mjs` | 跨 package.json/manifests 的版本号同步工具（含 --check 模式） |
| `tests/` | 9 个测试文件共约 3.7k 行（含 658 行的 fake-codex 假二进制夹具） |

组织方式评述：单体单包仓库，但结构严格按 Claude Code 插件规范分层——"声明层"（commands/agents/hooks/skills 的 markdown+json）与"执行层"（scripts/ 的 Node 脚本）分离。所有命令 markdown 最终都收敛到唯一的 `codex-companion.mjs` CLI，AI 侧只负责编排与转发，确定性逻辑全部在脚本内，这是"让 agent 做薄壳、让脚本做重活"的典型设计（review.md 明确 "Your only job is to run the review and return Codex's output verbatim"）。

## 5. 核心架构与关键实现

### 5.1 app-server 客户端与 broker 共享机制（lib/app-server.mjs + app-server-broker.mjs）

插件通过 JSON-RPC/JSONL 与 `codex app-server` 通信（lib/app-server.mjs:57 的 `AppServerClientBase`，stdio 与 socket 双传输）。关键设计是 **broker 进程**：首次调用时 `ensureBrokerSession`（lib/broker-lifecycle.mjs）拉起一个常驻 `app-server-broker.mjs serve --endpoint <socket>` 进程并写入端点与 pid 文件；后续调用通过 `CODEX_COMPANION_APP_SERVER_ENDPOINT` 环境变量直连同一 broker，避免每次斜杠命令都冷启动一个 app-server。broker 内部用 `activeRequestSocket/activeStreamSocket` 做单写者仲裁，`turn/completed` 通知即回收流式所有权（app-server-broker.mjs:84-100），冲突时以 RPC 错误码 `-32001`（`BROKER_BUSY_RPC_CODE`，app-server.mjs:23）拒绝并发请求。

### 5.2 评审执行与 turn 捕获（lib/codex.mjs）

`runAppServerReview`（codex.mjs:1002-1056）是评审核心：创建 `sandbox: "read-only"`、`approvalPolicy: "never"`、`ephemeral: true` 的临时线程，再发 `review/start` 请求（delivery 为 inline 或 detached），由 `captureTurn` 缓冲全部通知直到 turn 完成，回收 `reviewText`/`reasoningSummary`/错误与 stderr。线程以 `serviceName: "claude_code_codex_plugin"` 标记归属（codex.mjs:47）。评审结果可套用 `schemas/review-output.schema.json`（verdict/severity/confidence 字段）做结构化解析。而 `runAppServerTurn`（codex.mjs:1095+）服务于 rescue 任务，默认同样只读，仅在用户显式 `--write` 时放开（agents/codex-rescue.md:34）。

### 5.3 后台 job 状态机（lib/state.mjs + lib/tracked-jobs.mjs + job-control.mjs）

每个工作区独立一份状态：目录 = `CLAUDE_PLUGIN_DATA/state/<slug>-<sha256(规范路径)前16位>`，无该环境变量时回退到系统临时目录（state.mjs:29-44）。job 记录含 status（queued/running/…）、sessionId、进度 phase/threadId/turnId，日志逐行追加到 jobs 目录下的独立文件；`MAX_JOBS = 50` 自动裁剪（state.mjs:13）。`status/result/cancel` 默认按当前 Claude 会话 ID 过滤（stop-review-gate-hook.mjs:40-46 同样过滤），保证多会话并行时不互相踩踏。

### 5.4 声明式命令层与 Stop 评审门（commands/*.md + stop-review-gate-hook.mjs）

命令 markdown 是真正的"编排程序"：以 review.md 为例，frontmatter 约束 `allowed-tools: Read, Glob, Grep, Bash(node:*), Bash(git:*), AskUserQuestion`，正文规定 Claude 先用 git 命令估算变更规模、决定前台/后台、再以 `Bash(run_in_background: true)` 分离执行，并要求"逐字返回 Codex 输出、禁止修复任何问题"。Stop 门则把评审结果协议化为首行 `ALLOW:`/`BLOCK:`（stop-review-gate-hook.mjs:79-96），BLOCK 时输出 `{decision:"block", reason}` 让 Claude Code 阻止回合结束。

## 6. 构建与运行

```bash
# 安装插件（在 Claude Code 内）
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
/codex:setup          # 检查 Codex 就绪，可代装 npm install -g @openai/codex

# 本地开发
npm ci                                        # 安装 devDependencies（仅 TS 类型检查用）
npm test                                      # node --test tests/*.test.mjs
npm run build                                 # 生成 app-server TS 类型并 tsc --noEmit 类型检查
npm run bump-version / check-version          # 版本号跨文件同步/校验
```

前置条件：Node >= 18.18、ChatGPT 订阅（含 Free）或 OpenAI API key、本机 `codex` CLI（可由 setup 代装）。评审常用形态：`/codex:review --base main`、`/codex:rescue --model gpt-5.4-mini --effort high ...`（README.md）。

## 7. 测试与工程质量

- **测试规模**：9 个测试文件共 3715 行，其中 runtime.test.mjs 独占 2259 行，覆盖 setup/review/task/transfer/stop-gate/broker 的端到端行为；commands.test.mjs 校验命令 markdown 的 frontmatter 契约；另有 broker-endpoint/git/process/render/state/bump-version 单元测试。
- **测试风格**：突出亮点是 `tests/fake-codex-fixture.mjs`（658 行）——一个假 `codex` 二进制夹具，注入 PATH 后即可在 CI 无 ChatGPT 账号的情况下全链路跑通 companion 脚本与 broker（runtime.test.mjs:15-33 的 `installFakeCodex`），属于把外部依赖彻底虚拟化的成熟做法。
- **CI**：仅 PR 触发的单一 workflow（pull-request-ci.yml），10 分钟超时，Node 22，测试+构建两步；未见 main 分支保护性定时任务或发布自动化（release 应为手动，未确认）。
- **代码质量观察**：零依赖、纯 ESM、单入口收敛；tsconfig 用 `strict:false` 的宽松类型检查，类型安全是辅助而非防线；v1.0.3-v1.0.6 的 release notes 显示团队持续修复 Windows 兼容（ENOENT、Git Bash SHELL）、会话隔离、命令注入（v1.0.6 "Remove shell expansion for git commands"）等真实使用问题。

## 8. 社区与维护状态

- **活跃度**：2026-03-30 创建，2026-07-08 最后一次 push/release，生命周期约 3 个月内密集迭代；截至核实日 35317 星、2456 fork、543 open issues（gh api），issue 远多于代码量，反映海量使用与反馈。
- **Release 节奏**：7 个 release（v1.0.0 → v1.0.6）：3-30 首发、3-31 v1.0.2、4-08 v1.0.3、4-18 v1.0.4、6-23 v1.0.5（新增 /codex:transfer）、7-08 v1.0.6，平均月度级别，版本号始终锁在 1.0.x（`plugins/codex/CHANGELOG.md` 只有一行占位，真实日志靠 GitHub releases）。
- **贡献者结构**：13 名贡献者，呈"OpenAI 员工主导 + 社区修补"模式：核心版本 bump 由 dkundel-openai 等官方账号执行，外部贡献者（VOIDXAI、xiaolai、zeta987、tmchow、pengyou200902、bryane-oai 等）以小修为主（每条 PR 一行日志，见 release notes）。
- 未确认项：无公开的发布流程文档；CI 无发布 job，推测 release 手动触发。

## 9. 亮点与风险

**亮点**
1. **官方身份 + 零依赖**：OpenAI 维护的 Claude Code 一方插件，纯 Node 内置模块实现（package.json 无任何运行时依赖），安装面极小、审计容易。
2. **broker 复用架构**：常驻 app-server 进程 + socket 复用（lib/broker-lifecycle.mjs、app-server-broker.mjs），把每次斜杠命令的冷启动摊销掉，工程上远优于"每次 spawn CLI"的粗糙集成。
3. **AI 薄壳 + 脚本厚核的分层**：命令/subagent markdown 只做编排与"逐字转发"约束（agents/codex-rescue.md:11 "thin forwarding wrapper"），确定性逻辑全部落脚本，可控、可测（tests/ 契约测试覆盖 markdown 契约本身）。
4. **测试基建扎实**：658 行 fake-codex 夹具让全链路 E2E 无需真实账号/网络，runtime.test.mjs 2259 行覆盖后台任务、broker、stop 门等异步难点。
5. **跨智能体会话迁移**：`/codex:transfer` 用 Codex external-agent importer 实现 Claude→Codex 上下文接管，且路径白名单校验（claude-session-transfer.mjs:33-37）体现了安全意识。

**风险/局限**
1. **强绑定本机 Codex CLI 与其 app-server 私有协议**：`codex app-server generate-ts` 生成的类型与 broker 方法集随上游演进，旧版 Codex 会直接报错（codex.mjs:1072-1076 对 -32601 的兼容提示），升级耦合明显。
2. **Stop 评审门的用量风险**：官方自警告"Claude/Codex 循环、快速消耗限额"（README WARNING），启用需谨慎；hook 超时长达 900 秒。
3. **维护节奏放缓迹象**：最后 release 为 2026-07-08，此后近 5 个月无 push（核实日 2026-11-21），且 543 个 open issues 未消化；1.0.x 长期锁版本，CHANGELOG 文件形同虚设。
4. **平台差异面大**：release notes 中 Windows 相关修复频发（ENOENT、Git Bash SHELL、shell expansion），Windows 用户仍是体验洼地。

## 10. 总结评价

这是 LLM 编码工具竞争与融合交汇处的一个标志性项目：Anthropic 的 Claude Code 通过官方插件系统接纳了 OpenAI 的 Codex 作为"评审员、救援队与承接方"，而实现本身——零依赖 ESM 脚本、JSON-RPC broker、AI 薄壳脚本厚核、假二进制 E2E 测试——是一部高质量的小型集成工程教科书。5.6k 行代码撑起 35k+ 星，说明"官方 bridges between rival agents"的刚需远大于代码体量。适合三类人：想在 Claude Code 里引入 Codex 交叉评审/委托的用户（直接装）、想学习 Claude Code 插件体系完整范式的开发者（读 commands/agents/hooks/skills 的声明层设计）、以及做智能体互操作/多 agent 编排的工程师（读 lib/codex.mjs 与 broker 实现）。需要注意其维护已停滞数月、且深度依赖 Codex CLI 版本演进，生产依赖前先确认上游兼容性。
