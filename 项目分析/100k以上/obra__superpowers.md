# obra/superpowers 项目分析

> 星数 304065（快照 2026-10-24）｜主语言 Shell｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：一套安装进 Claude Code / Codex / Cursor 等 15+ 编码 Agent 的"技能包 + 开发方法论"，用 Markdown 写的流程约束（skills）+ Shell/JS 引导钩子，强制 AI 编码助手走"头脑风暴 → 设计 → 计划 → 子代理执行 → TDD → 评审"的完整软件工程流程。

| 项目 | 内容 |
|------|------|
| 名称 | obra/superpowers |
| 星数 | 304065（快照） |
| 主语言 | Shell（实际为 Markdown skills + Shell hooks + 少量 JS/TS 适配层的混合体） |
| 许可证 | MIT（`LICENSE`，`package.json`） |
| 版本 | v7.0.1（2026-10-21，`RELEASE-NOTES.md` 首条） |
| 创建时间 | 2025-10-09（gh api） |
| 最近 push | 2026-10-23（gh api），极高活跃度 |
| 贡献者 | 约 37 人（gh api contributors 首页计数） |
| 仓库规模 | 231 个文件，源码约 2.0MB（tarball 解压后） |
| Fork / Open Issues | 27415 / 369（gh api） |

## 2. 项目是什么，解决什么问题

Superpowers 解决的问题是：编码 Agent（Claude Code 等）能力很强但行为散漫——不问需求就动手写码、跳过测试、不系统调试、干到一半偏离计划。作者 Jesse Vincent（obra）把它做成一个可安装的"插件"：安装后，Agent 每次会话启动即被注入一份 `using-superpowers` 引导文（`skills/using-superpowers/SKILL.md`），其中用极强的措辞（`<EXTREMELY_IMPORTANT>`、Red Flags 反合理化对照表）规定"只要有 1% 可能适用的技能就必须先调用"，从而让后续一系列流程技能被自动触发，而不是靠用户手动 prompt。

核心方法论是一条流水线（`README.md` "The Basic Workflow"）：**brainstorming**（先逼问出真实需求并产出设计文档）→ **using-git-worktrees**（隔离工作区）→ **writing-plans**（把工作拆成 2-5 分钟粒度、含精确文件路径与验证步骤的任务）→ **subagent-driven-development** 或 **executing-plans**（逐任务派发全新子代理实现 + 双重评审，或单上下文内联执行）→ **test-driven-development**（强制 RED-GREEN-REFACTOR，先写失败测试）→ **requesting/receiving-code-review**（按严重级别阻塞推进）→ **finishing-a-development-branch**（合并/PR/丢弃决策与清理）。

项目自我定位是"技能即代码"（AGENTS.md："Skills are not prose — they are code that shapes agent behavior"）。技能内容经过大量真实会话压测打磨，仓库甚至以 94% 的 PR 拒绝率著称（`AGENTS.md` 开篇），明确拒绝任何未经验证就"重排版"技能内容的贡献。这是它区别于一般 prompt 集合仓库的关键：每一句措辞都是行为工程，改动需要 eval 证据。

另一个显著特征是"零依赖、全平台适配"。同一份 `skills/` 目录被十几套 harness 共享（`README.md` 安装章节列出 Claude Code、Codex App/CLI、Cursor、Devin、Gemini CLI、Copilot CLI、Grok、Kimi、OpenCode、Pi、Qwen、Hermes、Muse、Antigravity、Factory Droid），每家靠仓库里一个专属适配目录接入。

## 3. 技术栈与依赖

- **Markdown（112 个 .md 文件）**：主体资产。每个 skill 是 `skills/<name>/SKILL.md`（YAML frontmatter: name/description + 正文指令），配套 `references/` 按需加载的参考文档与 `scripts/` 辅助脚本。
- **Bash（50 个 .sh + 扩展名脚本，非测试 shell 约 1785 行）**：SessionStart 钩子、worktree 管理、评审包生成、版本同步等运行时逻辑。
- **JavaScript/TypeScript（11 js + 4 mjs + 2 ts）**：OpenCode V1/V2 双兼容插件（`.opencode/plugins/superpowers.js`，内含手写零依赖 frontmatter 解析器）、Pi 扩展（`.pi/extensions/superpowers.ts`）、brainstorming 可视化伴侣的本地服务器（`skills/brainstorming/scripts/server.cjs`，手写 RFC6455 WebSocket 实现）。
- **Node 内置模块 + Bash 为唯一"运行时依赖"**：`package.json` 无任何 dependencies；`AGENTS.md` 明文规定不接受引入第三方依赖的 PR（"zero-dependency plugin by design"）。
- **工程工具**：pre-commit 仅覆盖 `evals/` Python 的 ruff/ty 检查（`.pre-commit-config.yaml`）；`.version-bump.json` + `scripts/bump-version.sh` 统一 17 处清单中的版本号。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|------|------|
| `skills/` | 15 个技能包，核心资产（见第 5 章） |
| `hooks/` | `hooks.json`（SessionStart 触发配置）、`session-start`（bash 引导注入）、`run-hook.cmd`（cmd/bash 双语 polyglot 包装器，解决 Windows 下找 Git Bash） |
| `.claude-plugin/` `.codex-plugin/` `.cursor-plugin/` `.devin-plugin/` `.kimi-plugin/` `.muse-plugin/` `.antigravity-plugin/` `.hermes-plugin/` | 各 harness 的插件清单（plugin.json/yaml） |
| `.opencode/` `.pi/` | OpenCode 的 JS 插件实现（含 V1/V2 双协议）；Pi 的 TS 扩展 |
| `tests/` | 18 个子目录、约 70 个文件的插件基础设施测试（bash 集成测试 + node:test 单测 + 提示词样本） |
| `scripts/` | bump-version / lint-shell / package-codex-plugin / sync-to-codex-plugin / render-antigravity-logo |
| `docs/` | 各 harness 详细安装文档、`porting-to-a-new-harness.md`、Windows 说明、`superpowers/specs/` 设计规格 |
| `.github/` | issue/PR 模板、FUNDING；**无 workflows（未发现 CI 配置）** |
| `AGENTS.md` | 贡献者（尤其是 AI agent）行为规范，94% PR 拒绝率的成因说明 |
| `RELEASE-NOTES.md` | 112KB，56 个版本的完整发布说明 |

组织方式评述：这是典型的"单仓库多分发"结构——一份 skills 源，N 份 harness 清单。没有 monorepo 构建系统，没有包管理器锁文件（brainstorm-server 测试目录内有局部 package-lock），版本一致性靠 `scripts/bump-version.sh` 扫描同步。skills 之间通过 `superpowers:<skill-name>` 命名空间互相调用，形成一张技能依赖网（如 sdd 调 using-git-worktrees、finishing-a-development-branch）。

## 5. 核心架构与关键实现

### 5.1 引导注入机制（框架的"启动器"）

`hooks/hooks.json` 在 SessionStart（matcher `startup|clear|compact`）触发 `hooks/session-start`。该 bash 脚本把 `skills/using-superpowers/SKILL.md` 全文 JSON 转义后包进 `<EXTREMELY_IMPORTANT>` 标签注入上下文，且按平台输出不同 JSON 形状（`hooks/session-start` 第 71-83 行）：

```bash
if [ -n "${CURSOR_PLUGIN_ROOT:-}" ]; then
  emit_json '{\n  "additional_context": "%s"\n}\n' "$session_context"
elif [ -n "${CLAUDE_PLUGIN_ROOT:-}" ] && [ -z "${COPILOT_CLI:-}" ] && ...
  emit_json '{\n  "hookSpecificOutput": {\n  "hookEventName": "SessionStart", ... }\n}\n' ...
else  # Copilot CLI 或未知平台 — SDK 标准格式
  emit_json '{\n  "additionalContext": "%s"\n}\n' "$session_context"
fi
```

工程细节非常较真：不用 `dirname`（Claude Code 启动钩子可能带空 PATH，见脚本注释 anthropics/claude-code#43127）、用 bash 内建 `$(<file)` 替代 cat、用 `printf | cat` 吸收 Windows Git Bash 的 EPIPE（issue #1612）。matcher 含 `compact` 意味着压缩后重新注入——解决长会话技能"失忆"。Windows 侧由 `hooks/run-hook.cmd` 这个 cmd/bash polyglot 包装：cmd 段在标准路径和 PATH 中查找 bash.exe 并显式排除 WSL 启动器。

### 5.2 subagent-driven-development（方法论的发动机）

`skills/subagent-driven-development/SKILL.md`（约 337 行）是最复杂的技能：每任务派发一个**全新**实现者子代理（上下文隔离），随后派发评审者做"规格符合性 + 代码质量"双裁决，全部任务结束后再做一次全分支终审。关键设计：

- **台账（ledger）对抗压缩**：进度写进 `<repo-root>/.superpowers/sdd/<plan>/progress.md`（git-ignore 的 scratch 区），明确写道"压缩后相信台账与 git log，而不是你自己的记忆"，因为真实会话中出现过丢失上下文后把已完成任务全部重跑的事故（SKILL.md "Setup" 节）。
- **Rulings, not stalls**：控制器不许因歧义停下来问人，必须裁决并在台账记录 `Ruling: <决定> — <理由> — <错了的代价>`；只有四类情况允许停下（不可逆/破坏性操作、安全敏感、worktree 外副作用、计划彻底崩坏）。
- **模型分层**：机械任务用便宜模型、终审用最强模型、fix 循环第 4-5 轮升级一档，并要求派发时显式指定模型防止继承会话模型（"Model Selection" 节）。
- **上下文预算纪律**：任务书走 `scripts/task-brief` 落盘为文件而非贴进 prompt（引用了"一次派发 42k 字符、99% 是历史粘贴"的真实反例）；diff 走 `scripts/review-package` 生成文件给评审者。
- **反合理化表**收尾，与 using-superpowers 风格一致。

为在 Claude Code 压缩截断（20,000 字符）下幸存，v7 把 fix-loop、final-review、preflight、resuming 等拆进 `references/` 按步读取（`RELEASE-NOTES.md` v7.0.1，#2565），并由 `tests/skill-size/test-skill-size.sh` 强制任何 SKILL.md < 20000 字节。

### 5.3 brainstorming（入口技能）

`skills/brainstorming/SKILL.md` 规定实现前必须：先读现有项目上下文（"Look Before You Ask"，不许问项目已能回答的问题）→ 列 2-3 个方案让用户选 → 写设计文档并获批 → 才调用 writing-plans。v7.0.1 专门针对"问题太套路化"的社区反馈重调（`RELEASE-NOTES.md` v7.0.1：改动前 24/24 次运行跳过设计文档，改后仅 3/24）。其可选的"可视化伴侣"是一个零依赖 Node 服务器（`skills/brainstorming/scripts/server.cjs`，手写 WebSocket 帧编解码），README 末尾披露了它唯一的遥测：加载官网 logo 时带上版本号，可用 `SUPERPOWERS_DISABLE_TELEMETRY` 关闭。

### 5.4 多 harness 适配层

OpenCode 插件（`.opencode/plugins/superpowers.js`）同时兼容 V1（命名导出 + `experimental.chat.messages.transform` 注入引导）与 V2（`ctx.skill.transform` 原生注册技能），自带一个手写 frontmatter 解析器以避免依赖 skills-core；根目录 `index.js` 仅为满足 OpenCode V2"目录形式插件必须有 index 入口"的要求做 re-export。Pi 扩展（`.pi/extensions/superpowers.ts`）在 `session_start` 与 `session_compact` 事件重注引导。`docs/porting-to-a-new-harness.md` 把接入流程文档化，`AGENTS.md` 规定新 harness 必须附"发一句 Let's make a react todo list 会自动触发 brainstorming"的完整会话转录作为验收测试。

## 6. 构建与运行

本项目无构建产物，"运行"即安装进某个 harness（`README.md`）：

```bash
# Claude Code 官方市场
/plugin install superpowers@claude-plugins-official
# 或自建市场
/plugin marketplace add obra/superpowers-marketplace
/plugin install superpowers@superpowers-marketplace
# 其他 harness 各有命令，如 Gemini CLI:
gemini extensions install https://github.com/obra/superpowers
```

测试与维护命令（来自 `README.md` Contributing、`tests/` 与 `scripts/`）：

```bash
npm test                                   # package.json 未定义 test 脚本时由各 run-*.sh 承担（未确认 npm test 实际可用）
tests/claude-code/run-skill-tests.sh       # 需本机安装 claude CLI 的行为集成测试（每文件 900s 预算）
tests/skill-size/test-skill-size.sh        # SKILL.md 20KB 上限守卫
scripts/lint-shell.sh                      # shell 语法检查
scripts/bump-version.sh                    # 跨 17 处清单同步版本号
```

技能行为 eval 依赖外部仓库 prime-radiant-inc/superpowers-evals（clone 到 `evals/`，由 "Quorum" CLI 驱动真实编码 Agent 过 Gauntlet QA 场景，见 `AGENTS.md` Eval harness 节）。

## 7. 测试与工程质量

- **测试规模**：`tests/` 下 18 个子目录约 70 个文件、49 个 .sh/.test.js。覆盖各 harness 清单一致性（codex/devin/kimi/hermes/antigravity/pi/opencode 各有 test-*plugin*.sh）、hooks 行为、`explicit-skill-requests/prompts/` 的 14 个真实用户话术样本（验证"请用 brainstorming"、"skip formalities"等表述能否正确触发技能）、sdd 脚本与 worktree 路径策略等。
- **两层测试观**：插件基础设施测试在仓内 `tests/`；技能行为 eval 在外部 `superpowers-evals` 仓库做跨会话压测。技能改动须附 before/after eval 结果才能合入（`AGENTS.md`）。
- **brainstorm-server** 有 12 个文件的 node:test 单测（auth/lifecycle/ws-protocol/browser-launcher 等），对零依赖手写 WebSocket 服务器覆盖相当认真。
- **CI 缺位**：`.github/` 下只有模板与 FUNDING，未发现 workflows——静态检查靠 pre-commit（但只管 evals/ 的 Python），推测测试主要靠维护者本地与人工审查（未确认，无 CI 日志可查）。
- **工程质量观感**：shell 脚本普遍 `set -euo pipefail`、注释引用具体 issue 编号与踩坑原因；`tests/skill-size` 这类"给 prompt 文件设字节上限"的测试体现了对 LLM 上下文机制的量化理解。风格统一、防御性极强。

## 8. 社区与维护状态

- **极活跃**：创建于 2025-10-09（与作者博客发布文同步），一年内发布 56 个版本（`RELEASE-NOTES.md` 共 56 条 `## vX.Y.Z`），最近 push 2026-10-23、最新版 v7.0.1（2026-10-21），发版节奏约每周至每两周一次。
- **商业化背景**：README 挂出 Prime Radiant 的企业服务邮箱，项目由 Jesse Vincent 及 Prime Radiant 团队主导；贡献者约 37 人（gh api 首页计数），v7.0.1 致谢名单以 issue 报告与测试反馈为主，代码改动高度集中于核心团队（从致谢结构推断，未逐一验证）。
- **社区运营**：官方 Discord、release 通知邮件列表（`README.md` Community 节）；369 个 open issue 反映庞大的使用基数与反馈量。
- **对 AI 贡献的重炮防御**：`AGENTS.md` 公开 94% PR 拒绝率，要求 PR 必须含人类亲笔摘要、披露模型/插件环境、目标 dev 分支、禁止代写他人 issue 的修复——这在开源界相当罕见，本身就是该项目"如何管理 agent 行为"方法论的自证。

## 9. 亮点与风险

**亮点**
1. 技能即代码的工程化：反合理化表、四停条件、台账恢复、20KB 压缩上限守卫测试——把"prompt 工程"做成了有回归保护的软件工程（`skills/`、`tests/skill-size/`）。
2. 零依赖多 harness 分发：一份 skills + 17 处清单 + polyglot 钩子，覆盖 15+ 主流编码 Agent，适配层本身无第三方依赖（`.claude-plugin/` 等、`hooks/run-hook.cmd`）。
3. 子代理驱动开发的设计深度：上下文隔离、双阶段评审、模型分层调度、rulings-not-stalls 的自治授权模型，是目前公开方法论里最系统的多代理执行方案之一（`skills/subagent-driven-development/SKILL.md`）。
4. eval 驱动的内容演进：v7.0.1 对 brainstorming 的重调附有 24/24 → 3/24 的量化对照，技能措辞改动需压测证据（`RELEASE-NOTES.md`）。
5. 极强的可诊断性：`diagnosing-superpowers` 技能让用户一句"figure out what went wrong"即可让 agent 自查会话转录并产出带行号证据的报告（`skills/diagnosing-superpowers/`、README）。

**风险/局限**
1. 仓库无 CI workflows（`.github/` 下未发现），多 harness 兼容回归依赖人工与本地测试，长期多平台一致性存在隐患（未确认其是否依赖外部 CI）。
2. 强制性措辞（EXTREMELY_IMPORTANT、必须调用技能）对小任务有明显仪式感开销，官方也在 v7.0.1 承认"问题套路化"引发大量投诉；行为效果依赖各 harness 对注入指令的遵从度。
3. 核心技能高度耦合 Claude Code 的机制假设（compaction 20K 截断、Skill 工具、SessionStart 钩子），其他 harness 属于尽力适配，README 亦注明 Hermes 无压缩后钩子等残缺点。
4. 贡献门槛极高（94% 拒绝率、人类摘要强制），社区实质上只能报 bug，演进节奏完全系于 Prime Radiant 一家。

## 10. 总结评价

Superpowers 是"给 AI 编码 agent 立规矩"这个赛道的标杆项目：它不写传统意义的软件，而是把一套经过真实会话锤炼的软件工程方法论编码成可自动触发的 Markdown 技能与跨平台注入基础设施，用测试守卫 prompt、用 eval 驱动措辞演进、用台账对抗上下文压缩。它适合所有重度使用 Claude Code/Codex/Cursor 等工具、希望 agent 按正规流程干活（先设计、必 TDD、强制评审）的开发者直接安装使用；也适合研究 agent 行为工程的人精读——`skills/using-superpowers/SKILL.md`、`skills/subagent-driven-development/SKILL.md` 与 `AGENTS.md` 三份文件本身就是如何"驯化"大模型行为的一流案例教材。
