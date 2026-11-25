# addyosmani/agent-skills 项目分析

> 星数 109842（快照 2026-10-24）｜主语言 JavaScript｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：为 AI 编码智能体（Claude Code、Codex、Cursor、Gemini CLI 等）提供的一套覆盖"定义 → 计划 → 构建 → 验证 → 评审 → 发布"完整软件工程生命周期的 **Markdown 技能（Skills）集合**，并配套一套零依赖的 Node.js 校验与评测工具链。仓库由 Chrome 团队工程师 Addy Osmani 创建，是 2026 年 "agent skills" 生态中最具代表性的内容型项目之一。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | addyosmani/agent-skills | README.md |
| 星数 | 109842（快照 2026-10-24） | 任务参数 |
| 主语言 | JavaScript（内容主体实为 Markdown；工具链为 Node.js） | 仓库文件统计 |
| 许可证 | MIT（Copyright (c) 2025 Addy Osmani） | LICENSE |
| 创建 / 最近推送 | 2026-02-15 / 2026-10-29 | gh api repos/addyosmani/agent-skills |
| 最新版本 | 0.6.14（tag 与 plugin.json 一致，2026-10-29 发布） | plugin.json、gh api tags |
| 规模 | 222 个文件；118 个 Markdown 共约 14461 行；25 个技能目录 31 个 md 共约 8530 行 | find/wc 实测 |
| 社区 | 贡献者 30 人、fork 11688、open issues 145、watchers 547 | gh api |
| 主页 | https://skills.addy.ie | plugin.json |

## 2. 项目是什么，解决什么问题

AI 编码智能体的默认行为是"走最短路径"：跳过写规格、跳过测试、跳过安全评审。本项目把这些工程纪律封装成 25 个结构化的"技能"——每个技能是一个带 YAML frontmatter 的 `SKILL.md` 工作流文档，包含步骤、检查点、退出条件和证据要求，智能体在对应场景下按其执行，从而把资深工程师的做事方式一致地注入到每一次开发会话中（README.md "Why Agent Skills?"）。

内容取材大量来自 Google 工程文化：API 设计中的 Hyrum's Law、测试中的 Beyoncé Rule 与测试金字塔（80/15/5）、代码评审中的变更规模控制（约 100 行）与评审速度规范、简化中的 Chesterton's Fence、Git 工作流中的 trunk-based 开发、CI/CD 中的 Shift Left 与特性开关等（README.md）。它不是泛泛的提示词合集，而是"过程而非散文"：技能是智能体要执行的工作流，而不是供阅读的参考文档。

25 个技能按生命周期六阶段组织（skills/ 目录实测 25 个子目录）：**Define**（interview-me、idea-refine、spec-driven-development、constraint-driven-development）、**Plan**（planning-and-task-breakdown）、**Build**（incremental-implementation、test-driven-development、context-engineering、source-driven-development、doubt-driven-development、frontend-ui-engineering、api-and-interface-design）、**Verify**（browser-testing-with-devtools、debugging-and-error-recovery）、**Review**（code-review-and-quality、code-simplification、security-and-hardening、performance-optimization）、**Ship**（git-workflow-and-versioning、ci-cd-and-automation、deprecation-and-migration、documentation-and-adrs、observability-and-instrumentation、shipping-and-launch），外加一个路由用的元技能 using-agent-skills（CLAUDE.md "Skills by Phase"）。

除技能外还提供三类配套资产：4 个评审人设（agents/：code-reviewer、test-engineer、security-auditor、web-performance-auditor）、7 份共享检查清单（references/：testing、security、performance、accessibility、observability、definition-of-done、orchestration-patterns）、以及 9 个映射到开发流程的斜杠命令（/spec、/plan、/build、/test、/constraints、/review、/webperf、/code-simplify、/ship，README.md "Commands"）。

分发上强调多宿主可移植：既可用 `npx skills add addyosmani/agent-skills`（vercel-labs/skills CLI，宣称支持 70+ 智能体），也可作为 Claude Code / Codex / Gemini CLI / Antigravity / Copilot CLI / Command Code 的原生插件安装；Cursor、Windsurf、OpenCode、Kiro 则按各自约定复制 `skills/` 核心目录（README.md "Quick Start" 与 docs/ 下 17 篇文档）。命令层还提供 `/build auto` 模式：规格存在后自动生成计划并连续实现全部任务，一次批准、每任务仍独立测试与提交（README.md "Commands"）。

单技能体量实测（skills/ 各 SKILL.md）：最大的三个为 constraint-driven-development（21.1KB/311 行）、code-review-and-quality（21.0KB/398 行）、doubt-driven-development（16.6KB/243 行）；最小的 idea-refine 9.4KB/195 行。5 个技能带自有 references/ 或示例脚本，其余为单 SKILL.md 结构。

## 3. 技术栈与依赖

- **内容层**：纯 Markdown + YAML frontmatter（25 个 SKILL.md，单文件 9.4KB–21KB、192–398 行，实测 skills/）。
- **工具链**：Node.js（`node --test` 内置测试器，无任何第三方运行时依赖；run-evals.js 头注释明确 "Zero dependencies"，scripts/）。
- **脚本层**：13 个顶层 JS 脚本 + scripts/lib/ 共享库（skill-lint.js、execution-status.js、skill-lint-test.js，实测 scripts/lib/）。
- **钩子**：Bash 脚本 + jq（hooks/session-start.sh、sdd-cache-pre/post.sh、simplify-ignore.sh）。
- **CI**：GitHub Actions，Node 24，ubuntu/macos/windows 三平台矩阵（.github/workflows/test-plugin-install.yml）。
- **外部工具依赖（仅 CI/评测用）**：Claude Code CLI（`claude plugin validate`、Tier 3 行为评测用 headless claude，scripts/run-evals.js）。
- 无 package.json、无第三方 npm 依赖，整个工具链靠 Node 标准库运行（scripts/ 目录实测）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 | 出处 |
|---|---|---|
| `skills/` | 核心资产：25 个技能，每个一个目录 + SKILL.md（个别含自有 references/、examples、scripts，如 idea-refine） | skills/ 实测 |
| `agents/` | 4 个评审人设（子代理定义） | agents/ |
| `references/` | 7 份仓库级共享检查清单 | references/ |
| `commands/`、`.claude/commands/`、`.gemini/commands/` | 同一套 9 个斜杠命令的三种宿主适配（TOML/Markdown） | 各目录实测，各 9 个文件 |
| `.claude-plugin/`、`plugin.json`、`.codex-plugin/`、`.agents/plugins/` | Claude Code marketplace 清单与通用插件清单（版本 0.6.14） | .claude-plugin/marketplace.json |
| `hooks/` | 会话启动与 SDD 缓存、简化忽略等生命周期钩子（含 bash 测试） | hooks/ |
| `scripts/` | 校验器与评测运行器（15 个 JS 文件，含 7 个 `*-test.js`） | scripts/ |
| `evals/` | 25 份触发/路由评测用例（cases/）、行为评测场景（plugin/ 7 个）、fixtures、README 与 skill-impact.md 拒绝台账 | evals/ |
| `docs/` | 17 篇文档：skill-anatomy 格式规范、各宿主安装指南、采用指南、竞品对比等 | docs/ |
| `AGENTS.md`、`CLAUDE.md`、`CONTRIBUTING.md` | 面向"在本仓库干活的智能体"的自我配置与贡献规则 | 顶层 |

组织方式评述：典型的"**共享核心 + 宿主适配器**"单仓结构。可移植核心（skills/、agents/、references/）保持宿主无关；各宿主的原生发现路径（.claude/、.gemini/、.codex-plugin/ 等）作为薄适配层，并配有 `validate-commands.js` 保证三个命令目录的内容一致性（README.md "Project Structure" 表、.github/workflows/test-plugin-install.yml）。`.gitattributes` 强制 LF（.gitattributes）。注意 `.opencode/skills` 是指向 `../skills/` 的符号链接（tarball 解压时在 Windows 上会失败，属已知平台兼容细节）。

## 5. 核心架构与关键实现

### 5.1 SKILL.md 格式规范（内容架构）

每个技能固定遵循同一解剖结构（docs/skill-anatomy.md）：frontmatter 仅允许 Agent Skills 规范的 6 个顶层键（name、description、license、compatibility、metadata、allowed-tools），description 必须以第三人称写"做什么"并附"Use when..."触发条件（会被注入系统提示用于技能发现）；正文推荐包含 Overview、When to Use、Process、Common Rationalizations（反合理化表）、Red Flags、Verification 六节。关键设计有三：

- **反合理化表**：把智能体跳步骤的常见借口与反驳逐条列出。以 test-driven-development/SKILL.md 第 363-373 行为例："I'll write tests after the code works" → "You won't. And tests written after the fact test implementation, not behavior."
- **验证不可协商**：每个技能以证据要求收尾（测试通过、构建输出、运行时数据），不接受"看起来对"。
- **渐进披露**：SKILL.md 是入口，支撑材料按需加载以控制 token；lint 规则对 SKILL.md 施加 500 行上下文预算警告（scripts/lib/skill-lint.js 第 37 行）。

### 5.2 技能 lint 规则库（scripts/lib/skill-lint.js）

校验规则的"单一事实来源"，CLI validate-skills.js 只是薄封装（其文件头注释自述此设计）。错误级检查包括：SKILL.md 存在、frontmatter 含 name/description、name 与目录名一致、目录名 kebab-case、description ≤1024 字符且含正向触发词（正则 `/\buse (this )?when\b|\buse (before|after|during)\b/i`，并显式剔除"Do not use when..."类否定表述）、五个必备小节存在、子目录非空；警告级包括跨技能引用有效性。豁免清单硬编码在库内而非技能 frontmatter，注释明言"防止贡献者通过改自己的技能文件绕过校验"（skill-lint.js 第 83-89 行）。

### 5.3 评测运行器（scripts/run-evals.js）

两级评测体系（文件头注释与 evals/README.md）：**Tier 2**（默认、确定性、CI 安全）对 evals/cases/ 下 25 份用例做触发排序——每条正向 prompt 须把目标技能排进 top_k（默认 3），每条负向 prompt 不得排到第 1；另做描述"路由碰撞"检测（两技能 description 余弦相似度 >0.5 警告、>0.75 报错，run-evals.js 第 59-60 行）和覆盖率/schema 校验。**Tier 3**（可选、耗 token、不进 CI）用 headless claude 在一次性工作区里真实执行行为评测（执行超时 15 分钟、评分超时 5 分钟，预授权 Read/Glob/Grep/Edit/Write/Bash/WebFetch/WebSearch 工具）。每个技能必须配 ≥3 正向、≥2 负向触发与 ≥1 个行为评测（CONTRIBUTING.md "Structure"）。

### 5.4 斜杠命令与钩子（薄适配示例）

9 个命令在 `.claude/commands/*.md`、`.gemini/commands/*.toml`、`commands/*.toml` 三处保持语义同步，由 `validate-commands.js` 在 CI 中校验一致性。hooks/session-start.sh（16 行）演示了钩子契约：仅当宿主没有原生技能路由时才接线，用 jq 构造标准 SessionStart 输出信封 `{"hookSpecificOutput":{"hookEventName":"SessionStart",...}}`，缺 jq 时降级为提示而非报错。

## 6. 构建与运行

本仓库无可构建产物（CLAUDE.md 明言 "npm test — Not applicable (this is a documentation project)"）。实际可用的命令（均出自 .github/workflows/test-plugin-install.yml 与 CLAUDE.md）：

```bash
# 校验技能格式（错误级规则）
node scripts/validate-skills.js
# 触发/路由评测（CI 门槛：rank1 命中率 ≥95%）
node scripts/run-evals.js --min-rank1 95
# 行为评测（可选，调用本机 headless claude）
node scripts/run-evals.js --behavioral <skill> [--dry-run]
# 各专项校验
node scripts/validate-versions.js          # 清单版本一致性
node scripts/validate-commands.js          # 三套命令目录一致性
node scripts/validate-reference-links.js   # 技能内 references/ 链接有效性
node scripts/validate-artifact-paths.js    # spec/plan/todo 工件路径
# 单元测试（Node 内置 test runner）
node --test scripts/lib/skill-lint-test.js scripts/run-evals-test.js ...
bash hooks/sdd-cache-test.sh && bash hooks/simplify-ignore-test.sh
# 安装（作为 Claude Code 插件）
/plugin marketplace add addyosmani/agent-skills
/plugin install agent-skills@addy-agent-skills
```

## 7. 测试与工程质量

- **测试规模**：scripts/ 下 7 个 `*-test.js`（run-evals-test.js 达 41KB）+ hooks/ 下 3 个 bash 测试脚本（simplify-ignore-test.sh 22KB），全部进 CI（实测目录与 workflow）。
- **行为评测场景**：evals/plugin/ 下 7 个真实场景（各含 prompt.md + graders/），包括"贴入函数的 bug 修复须先写失败测试"（tdd-fires-on-pasted-bug）、"code-review 对正常提交须保持安静"（code-review-stays-quiet）、"简化技能不得在性能任务上误触发"（simplify-stays-quiet-on-make-it-fast）等，兼顾正反两向行为（evals/plugin/ 实测）。
- **CI 设计**：6 个 job——技能内容校验、命令一致性、macOS/Windows 双平台矩阵、钩子回归、`claude plugin validate .` 清单校验，以及最不寻常的 **test-install**：真的在 CI 里 `npm i -g @anthropic-ai/claude-code`、注册 marketplace 并完成一次插件安装（.github/workflows/test-plugin-install.yml）。
- **质量机制**：评测有"rank-1 棘轮"（--min-rank1 95，路由质量不得低于基线）；CONTRIBUTING.md 维护前置检查（查目录、查在途 PR、查 evals/skill-impact.md 拒绝台账、证明覆盖空缺），并要求剥离模型特定 workaround；跨技能内容重复被 lint 与碰撞检测双重抑制。
- **工程质量观察**：对"内容仓库"而言工程化程度罕见地高——规则可单测、评测可复现、发布节奏稳定（0.6.7→0.6.14 连续小版本，gh api tags）。局限是评测的"确定性部分"本质是字符串相似度排序，并非端到端效果证明；真正的行为评测（Tier 3）不在 CI 内运行。

## 8. 社区与维护状态

- **活跃度**：创建于 2026-02-15，分析日前一天（2026-10-29）仍有合并与发版（gh api commits：chore(release): bump plugin manifests to 0.6.14、Merge #463/#668/#735/#747），8 个多月持续高频迭代。
- **发版节奏**：语义化小版本密集发布，最新 0.6.14 与 plugin.json/marketplace.json 版本严格一致（validate-versions.js 强制）。
- **贡献者结构**：30 名贡献者（gh api），核心三人组为 Addy Osmani（Creator）与 Federico Bartoli、Joan León（Collaborators，README.md "Team"）；open issues 145、fork 11688。
- **内容治理**：被否决的技能提案记录在 evals/skill-impact.md 拒绝台账中，贡献者提案前必须查台账避免重复劳动（CONTRIBUTING.md 前置检查第 3 条）；docs/developer-onboarding.md 以"五层结构 + 验证回路"组织新人导览（CONTRIBUTING.md 开篇）。
- **插件元数据**：Claude Code marketplace 注册名 addy-agent-skills，插件名 agent-skills，版本 0.6.14 与根 plugin.json 双处一致（.claude-plugin/marketplace.json、plugin.json）。
- **生态位**：README 直接与 obra/superpowers、mattpocock/skills 对比（docs/comparison.md），并收录第三方对照实验；通过 vercel-labs/skills CLI 等第三方渠道进入 70+ 智能体的安装生态（README.md）。未确认：各宿主实际安装量无公开数据。

## 9. 亮点与风险

**亮点**

1. 内容方法论扎实：技能被设计成"带反合理化表与证据要求的门控工作流"，直击 LLM 智能体最典型的失效模式（偷懒跳步、事后补测试）（skills/test-driven-development/SKILL.md 第 363-373 行等）。
2. 用软件工程的方式管理提示词：格式规范（docs/skill-anatomy.md）、lint 规则库、确定性评测、路由碰撞检测、拒绝台账，形成可回归的内容管线。
3. 多宿主可移植性：共享核心 + 适配器分层，一份内容覆盖 Claude Code、Codex、Gemini、Antigravity、Cursor 等（README.md "Project Structure"）。
4. 工程文化浓度高：Google SWE 书中的实践被逐条落成可执行步骤而非口号（README.md "Why Agent Skills?"）。
5. CI 真实性强：不但校验内容，还在 CI 里完整走一遍插件安装链路。

**风险 / 局限**

1. 效果依赖模型顺从度：技能本质是提示词约束，没有硬性强制层；"防跳步"只能靠模型自觉 + 人审。
2. 上下文成本：SKILL.md 激活即全文加载，500 行仅为警告级预算，多技能叠加会显著占用 token（skill-lint.js 注释自述）。
3. 可移植性缺口：单独 `npx skills add` 某个技能时拿不到仓库级 references/ 共享清单，官方已在 README 与 issue #361 中承认。
4. 评测的确定性部分（top_k 排序、余弦碰撞）衡量的是"路由命中率"而非真实工程质量提升，Tier 3 行为评测又不在 CI 强制范围。

## 10. 总结评价

agent-skills 是 2026 年 AI 编码智能体浪潮中"提示词工程产品化"的标杆样本：它把资深工程师的工作流写成 25 份可审计、可回归测试的 Markdown 技能，配上罕见完整的校验-评测-发布管线，并在 8 个多月内积累约 11 万星与近 1.2 万 fork——增长动力来自 Addy Osmani 的个人影响力、技能生态爆发的时机窗口，以及"多宿主可移植 + 反合理化内容设计"带来的真实差异化。它适合两类人：想在 Claude Code/Codex 等工具中立即获得工程纪律的实践者（直接安装或按技能挑选），以及想研究"如何用软件工程方法治理提示词资产"的从业者（读 docs/skill-anatomy.md、scripts/lib/skill-lint.js 与 run-evals.js 这条内容质量管线）。若要将其用于严肃生产团队，建议结合自身约束裁剪技能子集，并自建 Tier 3 行为评测验证实际收益。
