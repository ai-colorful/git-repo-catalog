# gsd-build/get-shit-done 项目分析

> 星数 64238（快照 2026-10-24）｜主语言 JavaScript｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

GSD（Get Shit Done）是一个面向 Claude Code 等 AI 编程运行时的**元提示、上下文工程与规格驱动开发系统**：用一套 Markdown 提示工程 + Node.js 工具链，把"vibecoding"变成可控的 规划 → 执行 → 验证 流水线，核心卖点是对抗长上下文对话中的质量劣化（context rot）。

| 项 | 值 | 出处 |
|---|---|---|
| npm 包名 | `get-shit-done-cc`（v1.50.0-canary.0） | `package.json` |
| 作者 | TÂCHES（独立开发者，自称"不写代码，Claude Code 写"） | `README.zh-CN.md` |
| 语言 | JavaScript（CommonJS 工具链）+ TypeScript（SDK） | `package.json`、`sdk/package.json` |
| 许可证 | MIT | `LICENSE`、gh api |
| 创建时间 | 2025-12-14 | gh api |
| 最近 push | 2026-05-31，**仓库已归档**，README 指向继任项目 open-gsd/gsd-core | `README.md`、gh api |
| 运行时支持 | Claude Code、OpenCode、Gemini CLI、Kilo、Codex、Copilot、Cursor、Windsurf、Antigravity、Augment、Trae、CodeBuddy、Cline 等 15+ | `docs/ARCHITECTURE.md` |

**重要事实：本仓库已于 2026 年中归档，HEAD 上 `README.md` 仅剩一页迁移声明**（"GSD Has Moved → https://github.com/open-gsd/gsd-core"，见 `README.md`），完整功能介绍保留在各语言版 README（`README.zh-CN.md` 等）与 `docs/` 中。

## 2. 项目是什么，解决什么问题

GSD 解决的核心问题是 **context rot**：随着 AI 编程会话上下文窗口被填满，模型输出质量逐步劣化（`README.zh-CN.md`："它解决的是 context rot"）。作者 TÂCHES 是不写传统代码的独立开发者，认为既有规格驱动工具（BMAD、Speckit、Taskmaster）要么套用企业仪式（冲刺、故事点、Jira 流程），要么缺乏对"到底在构建什么"的整体理解，因此做了一套"复杂性在系统内部、不在你的工作流里"的系统（`README.zh-CN.md` "我为什么做这个"）。

它的用法是一条主循环命令流：`/gsd-new-project` 初始化（提问→并行研究→需求梳理→路线图），然后每个阶段走 `/gsd-discuss-phase`（收集用户偏好，产出 CONTEXT.md）→ `/gsd-plan-phase`（研究+拆成 2-3 份原子计划，XML 结构）→ `/gsd-execute-phase`（按依赖分 wave 并行执行，每份计划用全新 200K 上下文，每任务原子提交）→ `/gsd-verify-work`（人工验收，失败自动生成修复计划）→ `/gsd-ship`（建 PR），循环至里程碑完成（`README.zh-CN.md` "它是怎么工作的"）。

实现上是典型的"提示即程序"架构：`commands/gsd/*.md` 是用户入口（67 个），`get-shit-done/workflows/*.md` 是编排逻辑（91 个），`agents/gsd-*.md` 是专职子代理定义（33 个），`get-shit-done/references/` 存放共享知识文档——Claude 在每一步按指令读取对应 Markdown 并 spawn 子代理（`docs/ARCHITECTURE.md` System Overview）。真正的"引擎"是两个：CJS 工具 `gsd-tools.cjs`（含 `bin/lib/` 75 个领域模块，约 3 万行）负责解析/校验/更新 `.planning/` 状态文件；`@gsd-build/sdk`（TypeScript，`sdk/src/` 299 个文件）提供编程接口并逐步接管 CJS 路由（`docs/adr/0012-command-routing-hub.md`）。

所有状态落盘在项目 `.planning/` 目录的人读 Markdown/JSON 中（PROJECT/REQUIREMENTS/ROADMAP/STATE.md 等，`docs/ARCHITECTURE.md` File System Layout），不依赖数据库或服务端，`/clear` 后状态不丢，人与代理都能读。

增长动因：它踩中了 2025-2026 年 Claude Code 重度用户最大痛点（长任务质量崩塌），以"零仪式感"的定位区别于企业化工具，配合 5 语种 README、Discord 社区与高频发布快速传播；README 宣称被 Amazon、Google、Shopify、Webflow 工程师采用（`README.zh-CN.md`，真实性未确认）。

## 3. 技术栈与依赖

- **运行环境**：Node.js >= 22（`package.json` engines）。
- **运行时依赖极简**：仅 `@anthropic-ai/claude-agent-sdk`（SDK 编程接口底座）、`ws`（WebSocket）；可选依赖 `fallow`（`package.json`）。主体能力靠 Markdown 提示而非库。
- **双栈代码**：
  - 根层 CommonJS：安装器 `bin/install.js`（11,376 行）、CLI 工具 `get-shit-done/bin/gsd-tools.cjs` + `bin/lib/*.cjs`（75 模块，约 30,376 行）、hooks（`hooks/*.js|sh`，14 个）。
  - SDK 层严格 TypeScript ESM/NodeNext（`AGENTS.md` Coding Style；`sdk/tsconfig.json`），构建产物 `sdk/dist/`。
- **测试**：根层 Node 内置 `node:test` + `node:assert/strict`（明确禁用 Jest/Mocha/Chai，`AGENTS.md`）；SDK 用 Vitest（`sdk/vitest.config.ts`）；覆盖率用 `c8`，对 `get-shit-done/bin/lib/*.cjs` 强制 70% 行覆盖（`package.json` test:coverage）。
- **发布**：Changesets 流程（`.changeset/` 数十个变更条目、`scripts/changeset/` 自研 lint/render 脚本）。
- **语言占比说明**：GitHub 标记 JavaScript，但仓库同时包含大量 Markdown（提示资产）与 TypeScript（SDK）；按行数三者量级相当。

## 4. 目录结构与代码组织

| 目录 | 职责 | 规模 |
|---|---|---|
| `commands/gsd/` | 用户命令入口（斜杠命令/skill 定义） | 67 个 .md |
| `get-shit-done/workflows/` | 编排逻辑（命令的执行剧本） | 91 个 .md（含 `discuss-phase/modes/` 等子目录） |
| `get-shit-done/references/` | 工作流/代理共享的知识文档（gates、model-profiles、context-budget 等） | 41 个（`docs/INVENTORY.md`） |
| `get-shit-done/templates/` | 规划工件模板（project/roadmap/UAT/UI-SPEC 等） | 20+ 个 |
| `get-shit-done/bin/` | CJS 引擎：`gsd-tools.cjs` + `lib/` 75 个领域模块 | 约 3 万行 |
| `agents/` | 子代理角色定义（planner/executor/verifier/researcher/checker/debugger 等） | 33 个 |
| `hooks/` | 运行时钩子：statusline、上下文监控、prompt-injection 扫描、更新检查等 | 14 个文件 |
| `bin/` | `install.js` 多运行时安装器 + `gsd-sdk.js` 入口 | install.js 11,376 行 |
| `sdk/` | TypeScript SDK（`@gsd-build/sdk`）：query 分发、事件流、transport、phase-runner | `sdk/src/` 299 个 .ts |
| `tests/` | 根层集成/单元/安装/安全测试 | 552 个 .test.cjs |
| `docs/` | 架构、用户指南、配置、ADR（18 个）、发布说明 | 44 项 |
| `scripts/` | 自研 lint（description/skill-deps/no-source-grep/secret-scan/base64-scan/prompt-injection-scan 等 27 个） | — |
| `.changeset/`、`.github/`、`.plans/`、`.out-of-scope/` | 变更管理、CI/模板、项目自身规划（自举）、明确不做范围 | — |

组织方式评述：单仓多包（npm 根包 + `sdk/` 子包），"提示资产 + 双语言引擎"分层；用 `docs/INVENTORY.md` 与多条 lint 脚本强制各层清单/文案/测试数量一致性，工程纪律罕见地严格。

## 5. 核心架构与关键实现

### 5.1 分层编排：命令 → 工作流 → 子代理 → 文件状态

`docs/ARCHITECTURE.md` 给出五层结构：命令层（`commands/gsd/*.md`，YAML frontmatter + 提示体）→ 工作流层（薄编排器，只做"加载上下文、spawn 代理、收结果、改状态"）→ 代理层（每个子代理全新 200K 上下文）→ CLI 工具层（`gsd-sdk query` / `gsd-tools.cjs`）→ 文件状态层（`.planning/`）。核心设计原则四条：Fresh Context Per Agent、Thin Orchestrators、File-Based State、Absent=Enabled（特性开关缺省即启用，`docs/ARCHITECTURE.md` Design Principles）。

工作流文件本身就是"源代码"。例如 `get-shit-done/workflows/execute-phase.md` 开头：

```xml
<purpose>
Execute all plans in a phase using wave-based parallel execution.
</purpose>
<core_principle>
Orchestrator coordinates, not executes. Each subagent loads the full
execute-plan context. Orchestrator: discover plans → analyze deps →
group waves → spawn agents → handle checkpoints → collect results.
</core_principle>
<runtime_compatibility>
- **Claude Code:** Uses Agent(subagent_type="gsd-executor", ...) — blocks until complete
- **Copilot:** ... Default to sequential inline execution ...
</runtime_compatibility>
```

为控制每回合 token 成本，工作流文件有按 XL/LARGE/DEFAULT 分档的行数预算（1700/1500/1000 行，`tests/workflow-size-budget.test.cjs` 强制），超限就把 per-mode 内容拆进 `workflows/<name>/modes/`，父文件退化为薄分发器（`docs/ARCHITECTURE.md` Progressive disclosure）。v1.40 又引入 6 个命名空间 meta-skill 两级路由，把冷启动技能清单从约 2150 token 压到约 120（`CHANGELOG.md` 1.41.0、issue #2792）。

### 5.2 Wave 并行执行与并行安全

`execute-phase` 把阶段内各 PLAN.md 按依赖关系分组为 wave：同 wave 并行、跨 wave 串行（`README.zh-CN.md` 附 ASCII 图；`docs/ARCHITECTURE.md` Wave Execution Model）。并行写状态的竞态在引擎层有针对性处理——`get-shit-done/bin/lib/state.cjs` 用 `STATE.md.lock` 的 `O_EXCL` 原子创建做互斥，含 10s 陈旧锁检测与带抖动自旋等待，并在 `process.on('exit')` 统一清理持有的锁（`state.cjs` 头部注释，issue #1916）：

```js
// Track all lock files held by this process so they can be removed on exit.
const _heldStateLocks = new Set();
process.on('exit', () => {
  for (const lockPath of _heldStateLocks) {
    try { require('fs').unlinkSync(lockPath); } catch { /* already gone */ }
  }
});
```

上下文监控 hook（`hooks/gsd-context-monitor.js`）在剩余上下文 ≤35%/≤25% 两档向代理注入告警，5 次工具调用去重、升级告警绕过去重（`docs/ARCHITECTURE.md` Context Monitor Thresholds）。

### 5.3 多运行时安装器

`bin/install.js`（11,376 行）负责 15+ 运行时的差异适配：工具名映射（Claude 的 `Bash`→Copilot 的 `execute`）、hook 事件名（`PostToolUse`→Gemini 的 `AfterTool`）、命令格式（Codex 用 `$skill`、Gemini 用 `gsd:` 命名空间 TOML、Cline 写 `.clinerules`）、路径约定改写（`docs/ARCHITECTURE.md` Runtime Abstraction 与 Install Contract Matrix）。另含本地补丁备份（`gsd-local-patches/` 供 `/gsd-update --reapply`）、清单追踪（`gsd-file-manifest.json`）与卸载模式。

### 5.4 供应链与注入防御

v1.42.1 引入 Package Legitimacy Gate 对抗 slopsquatting（AI 幻觉包名被抢注）：研究代理对每个推荐包跑 `slopcheck` 并写审计表，规划代理对 `[ASSUMED]/[SUS]` 包在 PLAN.md 注入 `checkpoint:human-verify`，执行代理禁止静默换包（`docs/ARCHITECTURE.md` Package Legitimacy Gate）。`get-shit-done/bin/lib/security.cjs` 提供路径穿越防护、提示注入检测、shell 参数校验；`hooks/gsd-prompt-guard.js` 与 `gsd-read-injection-scanner.js` 分别扫描 `.planning/` 写入与 Read 工具输出中的注入模式（advisory）。仓库自身也用 `scripts/secret-scan.sh`、`base64-scan.sh`、`prompt-injection-scan.sh` 做 CI 级自检（`AGENTS.md` Security Tips）。

### 5.5 SDK 化（进行中的架构迁移）

`sdk/src/`（TypeScript）以 `@gsd-build/sdk` 发布，含 query 分发管线（`query-dispatch.ts` + 六值 errorKind 枚举）、事件流（`event-stream.ts`）、milestone/phase runner（`index.ts`、`phase-runner.ts`）。CJS 命令路由器正逐个迁移到 `get-shit-done/bin/lib/command-routing-hub.cjs`：统一 sdk/cjs 模式选择、no-throw 纯结果契约与封闭错误枚举，且明确"SDK 失败不透明回退 CJS"（`docs/adr/0012-command-routing-hub.md`、`docs/ARCHITECTURE.md` Command Routing Hub）。根目录 87KB 的 `CONTEXT.md` 是项目用 GSD 管理自身开发的"机器可 grep 谓词库"（自举/dogfooding 的直接证据）。

## 6. 构建与运行

来自 `package.json` scripts 与 `AGENTS.md`（均未实际运行，仅静态确认）：

```bash
npx get-shit-done-cc@latest          # 用户安装（交互选运行时/全局或本地）
npx get-shit-done-cc --claude --global  # 非交互示例
npm install                          # 开发安装依赖
npm test                             # 先构建 SDK，再跑根层 node:test 套件（scripts/run-tests.cjs）
npm run test:unit | test:integration | test:install | test:security | test:slow
npm run test:coverage                # c8，强制 lib/*.cjs 70% 行覆盖
npm run build:hooks                  # 重建 hooks/dist（原子写：staging + renameSync）
npm run build:sdk                    # cd sdk && npm ci && npm run build（tsc）
cd sdk && npm test                   # Vitest unit + integration 两个 project
node bin/install.js --claude --local # 源码安装到 ./.claude/（贡献者验证用）
```

## 7. 测试与工程质量

- **规模**：根层 552 个 `*.test.cjs`（约 13.2 万行，`tests/` 目录实测）；SDK 132 个单元测试 + 7 个集成测试（`sdk/src/**/*.test.ts` 实测）。测试文件命名带 issue 号（如 `tests/bug-2396-makefile-test-priority.test.cjs`），回归锚定习惯好。
- **分层套件**：unit/integration/install/security/slow 五类，`scripts/run-tests.cjs` 统一调度；install-smoke 在 CI 矩阵中真装真跑（`CHANGELOG.md` 1.41.0 提及 Node-22/Node-24 install-smoke matrix）。
- **自研 lint 体系**（`scripts/`）：描述长度、skill 依赖、禁止在测试中 grep 源码实现细节、测试文件数量门槛、PR 需关联 issue、文档必需项、变更集格式等 10+ 条目——把"提示资产的一致性"也纳入机器检查，是该仓库独有特色。
- **CI**：`.github/workflows/` 20 条流水线（test、pr-gate、install-smoke、security-scan、canary、release、release-sdk、changeset-required、stale、auto-label 等），含 issue 强制链接、未授权 PR 审批自动驳回等治理型 workflow。
- **观察**：CHANGELOG 条目极详尽（含竞态 bug 的完整根因分析，见 1.41.0 atomic writes 条目）；`QUICK-WINS-CONFIRMED-BUGS.md` 公开维护已知 bug 清单。整体工程质量在同类"提示工程"项目中显著偏高。

## 8. 社区与维护状态

- **活跃度**：2025-12-14 建仓，约 5 个月内从 0 到 64k 星；CHANGELOG 显示 2026-05 一周内连发 v1.42.1/1.42.2/1.42.3 与 v1.43.0-rc1/rc2（gh api releases），节奏极快。最后 push 2026-05-31。
- **归档与迁移**：仓库已标记 archived（gh api），`README.md` 声明开发移至 open-gsd/gsd-core；`docs/discussions/grok-build-support-2026-05.md`（`AGENTS.md` 引用）显示归档前仍在做多运行时同步。归档原因官方仅以迁移声明呈现，未确认。
- **贡献者**：136 人（gh api contributors 分页合计）；PR 治理严格（必须 `Closes #N` 关联 issue，`.github/PULL_REQUEST_TEMPLATE/` + 专用 lint）。
- **社区生态**：Discord、X 账号、5 语种 README；README 附 `$GSD` Solana 代币徽章（`README.zh-CN.md` 徽章行）——社区存在加密代币关联，评价见第 9 节。

## 9. 亮点与风险

**亮点**

1. 把"上下文工程"做成了可复用工程方法：fresh-context 子代理、薄编排器、文件态持久化三件套直击 AI 长任务质量衰减，且全部有文档与测试背书（`docs/ARCHITECTURE.md`、`tests/workflow-size-budget.test.cjs`）。
2. 提示资产管理工程化：工作流行数预算、技能 token 预算两级路由、agent 字符上限、清单 lint——对"提示膨胀"本身设防。
3. 工具链成熟：并行 STATE.md 文件锁、原子写 hooks 构建、c8 70% 覆盖率门禁、132k 行测试，远超同类项目平均水准。
4. 多运行时覆盖最广（15+ 运行时安装矩阵，`docs/ARCHITECTURE.md`），一个系统通吃主流 AI CLI。
5. 安全意识领先：slopsquatting 供应链门、双重注入扫描、secret/base64 CI 自检。
6. 高度自举：用自身工作流开发自身（根 `CONTEXT.md`、`.plans/`），长期真实使用反哺设计。

**风险/局限**

1. **本仓库已归档**，后续演进在 open-gsd/gsd-core；基于本仓库的代码阅读需注意 HEAD 冻结于 v1.50.0-canary（`package.json`）。
2. 与宿主运行时强耦合：Claude Code/Codex 等的 hook、skill、frontmatter 格式变动快，矩阵式适配维护成本高（`docs/ARCHITECTURE.md` Upstream Contract Sources 自述需定期刷新快照）。
3. 推荐使用 `claude --dangerously-skip-permissions` 跳过权限确认（`README.zh-CN.md`），自动化放权存在误操作面；注入扫描均为 advisory 不阻断。
4. README 关联 `$GSD` 加密代币徽章，项目治理与商业动机存在社区争议可能（徽章见 `README.zh-CN.md`，官方动机未确认）。

## 10. 总结评价

GSD 是 2025-2026 年 AI 编程工作流浪潮中工程化程度最高的"提示系统"代表作：它把 context rot 的解法从玄学变成可测试的架构（新上下文子代理 + 文件态 + wave 并行），又用 13 万行测试、自研 lint 矩阵和 20 条 CI 流水线把一堆 Markdown 提示资产管出了严肃软件的纪律，5 个月 64k 星是其解决问题能力的直接投票；代价是体积庞大（安装器 1.1 万行、75 个 CLI 模块、67 命令 91 工作流 33 代理）、与宿主运行时演进强绑定，且原仓库已归档迁移至 open-gsd/gsd-core。适合两类人：想把 Claude Code/Codex 用出工程节奏的重度使用者（直接装来跑项目），以及想研究"如何给 LLM 做上下文工程与多代理编排"的从业者（读 `docs/ARCHITECTURE.md` + `get-shit-done/workflows/execute-phase.md` + `bin/lib/state.cjs` 这条主线，是现成的教科书）。
