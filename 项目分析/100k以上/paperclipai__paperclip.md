# paperclipai/paperclip 项目分析

> 星数 106457（快照 2026-10-24）｜主语言 TypeScript｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：**AI Agent 公司的"控制平面"** —— 一个 Node.js 服务器 + React UI 的自托管应用，把多个异构 AI Agent（Claude Code、Codex、Cursor、OpenClaw 等）组织成一家有目标、组织架构、预算、审批与审计的"公司"并统一编排。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | paperclipai/paperclip | gh api |
| 定位 | "The app people use to manage AI agents for work" | README.md |
| 星数 | 106457（快照）；分析当日 gh api 实测 108,635 | gh api repos/paperclipai/paperclip |
| 主语言 | TypeScript（含少量 Rust 原生 runner） | package.json / packages/paperclip-runner/runner/Cargo.toml |
| 许可证 | MIT（© 2026 Paperclip Labs, Inc） | LICENSE、README License 章节 |
| 创建/最近推送 | 2026-03-02 创建；2026-10-29 push（前一天） | gh api |
| 默认分支 | master | gh api |
| 贡献者 | 233 人（contributors 分页 Link 头 last=233） | gh api contributors |
| 发布节奏 | CalVer 周更（v2026.1023.0、v2026.1012.0…） | gh api releases |

## 2. 项目是什么，解决什么问题

Paperclip 的核心命题来自 README.md："**If OpenClaw is an employee, Paperclip is the company**"。它不做 Agent 本身，而是做"运行一家由 Agent 组成的公司"所需的一切基础设施：任务工单、组织架构、心跳调度、预算治理、审批门禁与审计日志。用户定义一个公司目标（如"做到 $1M MRR 的 AI 笔记应用"），招聘不同 harness 的 Agent 担任 CEO/CTO/工程师，然后从仪表盘审批策略、设置预算、按 go 启动并持续监督。

它解决的问题在 README "Problems Paperclip solves" 一节列得很实：几十个 Claude Code 终端标签页失控、跨重启丢会话、Agent 缺乏目标上下文、失控循环烧掉数百美元 token、周期性工作靠人肉触发等。对应解法分别是：票务化任务与线程化会话持久化、目标从公司→项目→任务的层级传导、花费追踪与预算硬停、Routines 定时生成带历史记录的任务。

doc/PRODUCT.md 给出了更正式的领域模型：**Company 是一等对象**，含 goal、employees（每个员工都是 Agent）、org 结构、收支与任务层级；每个 Agent 有 adapter 类型与配置（OpenClaw 用 SOUL.md/HEARTBEAT.md，Claude Code 用 CLAUDE.md，格式由 adapter 决定）、角色汇报线与能力描述。README "Why Paperclip is special" 强调了几个工程上较真的点：原子任务认领（单一 assignee + 执行锁防竞争）、持久工作上下文与跨 run 会话恢复、运行时技能注入、带回滚的治理（配置变更版本化）、以及公司级数据边界（单部署多组织）。

四个支柱（README "The four pillars"）概括了产品面：Agentic Task Manager（任务+审批+验收）、Org Chart for Agents（角色/权限/边界）、Agent Employee Training（Skill Studio、评测、绩效）、Agentic OS（跨厂商运行时、沙箱、MCP、SSO/成本控制）。

## 3. 技术栈与依赖

- **运行时与工程化**：Node.js ≥ 24.11（engines，package.json）、pnpm 9.15 workspaces + Turborepo 2.11（turbo.json）、TypeScript ^7.0.2、Vitest 5、Playwright、esbuild。
- **服务端**（server/package.json，72 个直接依赖）：
  - ORM/DB：Drizzle（packages/db，drizzle.config.ts）+ PostgreSQL；本地开发用嵌入式 Postgres（embedded-postgres@18.1.0-beta.16，带 patch）。
  - 模型接入：Vercel AI SDK 全家桶（@ai-sdk/anthropic、openai、google、amazon-bedrock、xai）+ @openrouter/ai-sdk-provider。
  - Agent 接入：@agentclientprotocol/claude-agent-acp、codex-acp（ACP 协议）、acpx；MCP 官方 SDK（@modelcontextprotocol/sdk）。
  - 消息渠道：@chat-adapter/slack、discord、telegram、teams、github（均有 pnpm patch）。
  - 可观测/密钥/存储：OpenTelemetry（可选 peer）、Sentry 10.71（可选 peer）、@1password/sdk、@aws-sdk/client-s3。
- **前端**（ui/）：React 19（overrides 统一到 ^19.2.8）+ Vite + Tailwind v4（无 tailwind 配置文件，token 全在 ui/src/index.css 的 CSS 自定义属性，见 DESIGN.md）+ shadcn 体系 + Storybook。
- **CLI**（cli/）：esbuild 打包，npm 包名 `paperclipai`，提供 `onboard`/`test-drive`/`configure` 等命令（README Quickstart）。
- **原生 Runner**：packages/paperclip-runner/ 下 TS 层 + Rust crate runner-core（runner/Cargo.toml），自托管默认启用，可用 `PAPERCLIP_RUNNER_BINARY` 指向预编译产物（README 源码开发一节）。

## 4. 目录结构与代码组织

pnpm-workspace.yaml 声明 `packages/*`、`packages/adapters/*`、`packages/plugins/*`、`server`、`ui`、`cli`，是典型的 **monorepo 多包** 结构：

| 目录 | 职责 |
| --- | --- |
| server/ | 核心控制面服务（3094 个 .ts，自测规模最大），含 http/ 路由（server/src/routes 下 101 个文件）、services/（数百个服务文件）、modules/（严格分层的特性模块） |
| ui/ | React 前端（2400 个 .ts/.tsx），src/ 下 features/、pages/、components/、adapters/、plugins/、i18n/ |
| cli/ | `paperclipai` npm 包，安装引导与实例管理 |
| packages/db/ | Drizzle schema 与迁移；src/schema/ 共 177 张表定义、约 12,138 行 |
| packages/adapters/ | 12 个适配器包：claude-local、codex-local、cursor-local/cloud、gemini-local、grok-local、kimi-local、opencode-local、pi-local、openclaw-gateway、hermes、hermes-gateway |
| packages/paperclip-runner/ | TS+Rust 原生执行器、协议（protocol/）、评测（evals） |
| packages/shared/、skills-catalog/、teams-catalog/、mcp-server/、plugins/ | 共享类型、内置技能目录、团队模板、MCP server、插件 |
| skills/、rules/ | 内置 Agent 技能（paperclip、board、slack 等）与规则 |
| tests/ | e2e、release-smoke、runner-e2e、storybook-visual、lifecycle-baseline 等专项套件 |
| doc/ 与 docs/ | 工程文档（PRODUCT.md、TASKS.md、AGENT-LIFECYCLE.md 等 40+）与 Mintlify 用户文档站 |
| .github/workflows/ | 24 个 CI 工作流 |

组织方式评述：按"工作面包 + 特性模块内三层"组织；适配器以独立包形式贡献 cli/server/ui 三端代码（如 packages/adapters/claude-local/src/ 下分 cli/、server/、ui/），是本仓库最有特色的横向切分。

## 5. 核心架构与关键实现

### 5.1 特性模块的三层架构与机器强制边界

server/src/modules/README.md 规定每个模块必须是 `domain → application → adapters` 单向依赖：domain 只收发纯数据、禁止读时钟（`now` 由调用方传入）、禁止引 Drizzle/IO；application 持端口调用 domain；adapters 落 Postgres 与进程控制。违规由 `pnpm check:module-boundaries`（package.json 脚本，实现在 scripts/check-module-boundaries.mjs）在 CI 拦截。这是把六边形架构写成了可机检的规范。

### 5.2 心跳执行的纯函数状态机（wake-queue）

Agent 唤醒/排队/合并的决策被抽成无副作用决策函数，见 `server/src/modules/wake-queue/domain/policy.ts`：

```ts
export function decideWakeAdmission(facts: WakeAdmissionFacts): WakeAdmissionDecision {
  if (
    facts.allowRunCoalescing !== false &&
    facts.sameDurableActor !== false &&
    facts.isSameExecutionAgent &&
    !facts.shouldDeferFollowupWake &&
    !facts.shouldQueueFollowupForRunningWake &&
    facts.availableActiveExecutionRunPresent
  ) {
    return { kind: "coalesce" };
  }
  if (facts.availableActiveExecutionRunPresent) {
    return { kind: "defer" };
  }
  return { kind: "proceed" };
}
```

文件头注释明确四类决策点（admission、pre-drain、per-wake、release-recovery），调用方读库打包成 facts，策略文件"只分支、不查库不看时钟"。配合 Postgres 适配器（modules/wake-queue/adapters/postgres.ts）实现 DB 背书的延迟唤醒队列与执行锁，是"原子任务认领"承诺的落地处。run-dispatch/、agent-lifecycle/、active-run-watchdog/、external-agents/ 同构。

### 5.3 数据模型：177 张表的公司操作系统

packages/db/src/schema/ 按领域拆 177 个文件（约 1.2 万行）：companies、agents、agent_wakeup_requests、agent_task_sessions、budget_policies、budget_incidents、approvals、activity_log、company_secret_*、chat_*（slack/telegram/discord/teams/voice）等，覆盖组织、任务、预算、治理、密钥、渠道全域，且配有大量迁移安全测试（packages/db/src/check-migration-safety.test.ts 等）。

### 5.4 适配器体系与原生 Runner

每个适配器包导出类型标识、模型目录与配置文档（packages/adapters/claude-local/src/index.ts：`type = "claude_local"`、模型列表、`agentConfigurationDoc`），服务端通过 adapter-registry-bootstrap.ts 注册（server/src/services/）。README 的 ASCII 架构图把服务端划分为 Identity、Work & Tasks、Heartbeat、Governance、Budget、Plugins 等 12 个子系统，与 schema/服务和目录一一对应。Runner 侧 packages/paperclip-runner/runner/ 为 Rust 实现（crates/runner-core），负责本地执行的进程管理与恢复，TS 层提供 protocol/ 与 eval-session CLI。

## 6. 构建与运行

来自 README Development 一节与 package.json scripts（均未实际运行，仅静态核对）：

```bash
# 一键安装（用户侧）
npx paperclipai@latest onboard --yes        # 默认本地信任回环模式
npx paperclipai test-drive --harness codex  # 隔离试用实例

# 源码开发
git clone https://github.com/paperclipai/paperclip.git && cd paperclip
pnpm install            # 需 Node 24.11+、pnpm 9.15+；Rust 工具链（原生 runner）
pnpm dev                # API+UI watch 模式，http://localhost:3100，自动内嵌 Postgres
pnpm build / pnpm typecheck
pnpm test               # Vitest（默认廉价档，不含 Playwright）
pnpm test:e2e           # Playwright 浏览器套件
pnpm db:generate / pnpm db:migrate
```

生产可用自有 Postgres + Docker 部署（Dockerfile、docker/、docs Docker 指南）；远程访问建议认证模式 + Tailscale/LAN 绑定（README FAQ）。

## 7. 测试与工程质量

- **规模**：全仓约 7,175 个 .ts + 1,963 个 .tsx，其中 `*.test.ts|*.spec.ts` 约 2,917 个——测试文件占比约三分之一（find 统计）。
- **分层测试**：Vitest 单测（server/、ui/、cli/ 各有 vitest.config.ts）、Playwright e2e（tests/e2e 多套 config：multiuser-authenticated、company-context）、runner-acceptance、release-smoke、storybook-visual 视觉回归（tests/storybook-visual/）、promptfoo 评测（evals/promptfoo，`evals:smoke`）、lifecycle-baseline。
- **CI**：.github/workflows/ 24 条流水线，含 pr.yml、ci-profile.yml（按改动剖面路由测试）、runner-chaos-evals.yml、runner-live-evals.yml、release-verify.yml 等，发布前还有 test:release-registry 这类 40+ 文件的注册表测试（package.json）。
- **质量护栏脚本**：check:module-boundaries、check:no-git-push、check:forbidden-tokens、check:token-gates、check:node-version（package.json scripts），为"Agent 大量参与开发"设计的机器约束。DESIGN.md 第 8 条"Agent-modifiable by design"直言 UI 体系必须能被 Agent 按指令安全修改。
- 观察到个别测试文件与实现同目录混放（如 server/src/ 下 embedded-postgres-owner.test.ts），是有意采用的就近测试风格。

## 8. 社区与维护状态

- **极度活跃**：gh api 显示最近 push 为 2026-10-29（分析前一天）；README FAQ 自述已合并 2,700+ PR。
- **贡献者**：233 人（contributors API 分页 Link 头推算）。
- **发布**：CalVer 周更节奏稳定（v2026.916 → v2026.1001 → v2026.1005 → v2026.1012 → v2026.1023，gh api releases），配套 release.sh/canary/rollback 脚本（package.json）。
- **issue 压力**：open_issues_count 6,595（gh api），对 10 万星项目属高负荷，但issue/PR 流转有 issue-references、commitperclip-review.yml 等 CI 辅助（未验证流转时效）。
- **公司背景**：Paperclip Labs 商业化（paperclip.ing，Cloud waitlist 开放中），在招人（README Contributing 节）。

## 9. 亮点与风险

**亮点**

1. 领域建模罕见地完整：把"管理 Agent"落到组织架构、目标血缘、预算硬停、审批回滚、审计日志等可执行机制，而非提示词工程（doc/PRODUCT.md、README "Why Paperclip is special"）。
2. 工程纪律极强：三层模块边界机检（server/src/modules/README.md + check:module-boundaries）、纯函数状态机（wake-queue/domain/policy.ts）、约 2,900 个测试文件、24 条 CI 流水线。
3. 异构适配生态：12 个官方适配器统一"能收心跳即可入职"接口，且每个适配器同时贡献 CLI/Server/UI 三端能力（packages/adapters/）。
4. 自托管体验好：`npx paperclipai onboard` + 嵌入式 Postgres 免配置起步，Docker 与 Tailscale 路径齐备（README Quickstart）。
5. 文档与设计文档厚度大：71KB 的 DESIGN.md（token 化设计系统、面向 Agent 修改的原则）+ doc/ 下 40+ 工程规范。

**风险/局限**

1. 项目极年轻（2026-03 创建，约 8 个月冲到 10 万星），API/表结构（177 张表）大概率仍在快速演化，升级成本高。
2. 复杂度与依赖负担：server 72 个直接依赖、13 个 pnpm patch（含对 Claude/Codex ACP 包的锁定 override），上游一变就要跟着改（pnpm-workspace.yaml）。
3. 遥测默认开启（README Telemetry 章节，虽提供三种关闭方式），对隐私敏感部署需要显式处理。
4. 资源占用与上手门槛不低：Node 24.11+/Rust 工具链/Postgres 心智模型，6,595 个 open issue 也意味着踩坑时自助成分较高。

## 10. 总结评价

Paperclip 是 2026 年"AI Agent 公司化"赛道的旗舰级开源实现：它把易碎的"一堆终端里跑 Agent"升级为有组织架构、预算、审批与审计的持久化操作系统，工程纪律（分层边界、纯函数决策、海量测试、周更发布）达到商业产品水准，10 万星与其完成度基本相称。适合想认真跑"全天候 Agent 团队"的独立开发者与小团队自托管使用；适合研究 Agent 编排、多租户治理与大型 TypeScript monorepo 工程化的开发者精读源码——建议入口是 server/src/modules/README.md、modules/wake-queue/domain/policy.ts 与 doc/PRODUCT.md。想拿它当轻量脚手架或对依赖膨胀敏感的人则需谨慎。
