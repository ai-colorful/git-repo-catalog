# mattpocock/skills 项目分析

> 星数 301533（快照 2026-10-24）｜主语言 Shell（实际主体为 Markdown）｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：Matt Pocock（Total TypeScript 作者）日常使用的 AI 编码 agent skills 集合，主打"真实工程而非氛围编码"，以"盘问（grilling）— 规格 — 工单 — TDD — 评审"的小型可组合技能链对抗 GSD/BMAD/Spec-Kit 类重流程框架。

| 项 | 值 |
| --- | --- |
| 名称 | mattpocock/skills |
| 星数 | 306,388（gh api 实时，2026-10-30） |
| Fork | 25,774（gh api） |
| 语言 | GitHub 标注 Shell，实际 95% 以上内容为 Markdown（仅 3 个脚本：2 个 bash + 1 个 node） |
| 许可证 | MIT（`LICENSE`，Copyright (c) 2026 Matt Pocock） |
| 创建时间 | 2026-02-03（gh api） |
| 最近推送 | 2026-10-29（gh api，截至分析日仅隔 1 天） |
| 最新 release | v1.3.2（2026-10-29 发布，changesets 驱动） |
| 贡献者 | 4 人（gh api contributors），实质单维护者 + 少量外部 PR |
| 规模 | 38 个 skill（27 个随插件发布），SKILL.md 合计 2,713 行 |

## 2. 项目是什么，解决什么问题

这是 Matt Pocock 把自己 `.agents` 目录里的日常 agent skills 开源出来的仓库（`README.md` 标题 "Skills For Real Engineers"）。它不是框架，而是一组离散的 Markdown 指令文件（SKILL.md），每个只解决 agent 编码中的一个具体失败模式，用户按需组合进 Claude Code、Codex、Copilot、Gemini CLI 等任意 harness。

`README.md` 用四个失败模式组织整个叙事，每个对应一组技能：

1. **"Agent 没做我想要的"**（沟通错位）→ `/grill-me`、`/grill-with-docs`：开工前让 agent 反过来盘问用户，把设计树的每个分支问清楚（README.md #1 节）。
2. **"Agent 太啰嗦"**（缺乏共享语言）→ `GLOSSARY.md` + `domain-modeling` 技能：为项目建立领域词汇表和 ADR，agent 用 1 个词代替 20 个词（README.md #2 节，引用 Eric Evans《DDD》）。
3. **"代码跑不起来"**（反馈回路缺失）→ `/tdd`（红绿重构循环）、`diagnosing-bugs`（门控式调试循环）（README.md #3 节）。
4. **"我们造了一个泥球"**（熵增失控）→ `codebase-design`、`/improve-codebase-architecture`：每天投资设计，用 Ousterhout 的"深模块"概念扫描重构候选（README.md #4 节）。

与 Spec-Kit、BMAD 等的区别在 README 开篇说得很直白：那些方法"接管流程"，出错时难以修复；这套 skills 刻意保持小、易改、可组合（"Hack around with them. Make them your own."）。

仓库本身也"吃自己的狗粮"：根目录的 `CLAUDE.md` 是它自己的 agent 约定文件（定义 bucket 规则、文档同步义务、禁用 em-dash 等），`GLOSSARY.md`、`SCOPE.md`、`.agents/adr/` 正是它推荐用户项目采用的结构。CHANGELOG（374 行，版本 1.3.2）显示这些技能在真实使用中持续被打磨，几乎每个改动都关联一个真实 issue 编号（#262 到 #1341，9 个月内 1300+ issue/PR 编号），这是"提示词也需要工程化迭代"的最好证据。

## 3. 技术栈与依赖

- **主体**：纯 Markdown。每个 skill 是一个目录，含 `SKILL.md`（YAML frontmatter：`name`、`description`、可选 `disable-model-invocation: true`）+ 辅助文档（如 `skills/engineering/tdd/tests.md`、`mocking.md`，`skills/engineering/codebase-design/DEEPENING.md`）。
- **多 harness 适配**：`agents/openai.yaml`（每个 user-invoked skill 配 `policy.allow_implicit_invocation: false`，见 `.agents/invocation.md`）。
- **分发**：Claude Code 插件（`.claude-plugin/plugin.json` 列出 27 个技能路径 + `marketplace.json` 自建 marketplace）；skills.sh / `npx skills` 安装器；Gemini CLI 按 bucket 安装（`README.md` 安装节）。
- **版本与发布**：Node 22 + npm，`@changesets/cli`（`package.json` devDependencies 唯一依赖），`scripts/sync-plugin-version.mjs` 把 package.json 版本同步进 plugin.json（带 `--check` 校验模式）。
- **脚本**：`scripts/link-skills.sh`（bash，把技能 symlink 进 `~/.claude/skills`、`~/.agents/skills`，注释明确"dev-only，非受支持安装器"）、`scripts/list-skills.sh`。
- 无第三方运行时依赖、无框架——刻意保持零依赖。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `skills/engineering/` | 20 个日常代码类技能（核心，全部随插件发布） |
| `skills/productivity/` | 7 个非代码工作流技能（随插件发布） |
| `skills/in-progress/` | 7 个 beta 技能（公开求反馈，不进插件） |
| `skills/misc/` | 4 个留存但不推广的技能 |
| `skills/deprecated/` | 仅一个 README.md，已淘汰技能（tarball 中为空） |
| `docs/<bucket>/` | 每个已推广技能的人工文档页，发布到 `aihero.dev/skills-<name>`（28 个 md） |
| `.claude-plugin/` | plugin.json（27 技能清单）+ marketplace.json |
| `.agents/` | 仓库自身的 agent 约定：2 篇 ADR、invocation.md、writing-docs.md、install-block.md |
| `.out-of-scope/` | 9 篇"明确拒绝的设计"记录（如 new-skills.md、subagent-recursion.md） |
| `.github/workflows/` | release.yml、needs-info.yml、triage-label.yml |
| `scripts/` | link-skills.sh、list-skills.sh、sync-plugin-version.mjs |
| 根 | README.md（258 行，核心叙事）、CLAUDE.md、SCOPE.md、GLOSSARY.md、CHANGELOG.md、package.json |

组织方式评述：**单体内容仓库 + 插件清单驱动**。核心纪律在 `CLAUDE.md` 中写死：promotion 的技能必须同时出现在根 README 和 plugin.json 的 `skills` 数组里；非推广 bucket 禁止出现在两者中；每改一个技能必须同步 docs 页和 `ask-matt` 路由器。这是一套靠约定 + 交叉引用（而非代码）维持一致性的"文档即代码"治理。

## 5. 核心架构与关键实现

### 5.1 grilling：整座大厦的原语（`skills/productivity/grilling/SKILL.md`，31 行）

最短也最核心的技能，被 `grill-me`、`grill-with-docs`、`triage`、`wayfinder`、`improve-codebase-architecture` 复用。算法化地把访谈建模为"设计树 + 前沿（frontier）"：

> Work the tree in **rounds**. The **frontier** is every decision whose prerequisites are already settled... Ask the whole frontier in one round... Each round the user answers reshapes the tree: settled decisions push the frontier outward.

三条关键规则：事实由 agent 派 fresh-context 子代理自查（"Finding _facts_ is your job, never the user's"）；决策必须交给用户；前沿为空（无隐含假设）才算结束。问题措辞还要求"答 yes 即接受推荐答案"（CHANGELOG #1193 记录了这一改进的来由）。

### 5.2 组合式调度而非复制粘贴（`skills/engineering/grill-with-docs/SKILL.md`，全文仅 3 行正文）

> Call the Skill tool twice, for "grilling" and "domain-modeling".

这是整个仓库最重要的架构决定：user-invoked 技能是编排层，model-invoked 技能是可复用纪律层（README.md "Reference" 一节明确"用户技能可以调用模型技能，但绝不调用另一个用户技能"）。技能间通过 harness 的 Skill 工具组合，每个技能保持单一职责。

### 5.3 tdd：把 TDD 纪律编码为可执行约束（`skills/engineering/tdd/SKILL.md`）

亮点是 **seam（接缝）概念**："Test only at pre-agreed seams"——写测试前先与用户确认在哪些公共边界测，每个 seam 标注"能抓到什么/放过什么"。反模式清单具体到可判定：implementation-coupled（mock 内部协作对象）、tautological（`expect(add(a,b)).toBe(a+b)`）、horizontal slicing（先写全部测试再写实现，改为垂直切片 tracer bullet）。并显式声明"重构不属于红绿循环，归 code-review 阶段"——把 XP 实践拆成了技能间的流水线。

### 5.4 implement-spec：并发子代理任务图（`skills/engineering/implement-spec/SKILL.md`）

把"工单不是步骤列表，是带阻塞边的任务图（task graph），永远存在一个 ready 的前沿"作为调度原则：每个工单派一个 fresh-context 实现者子代理，各自在独立 worktree 分支上工作、后台并发跑，子代理间只传"context pointers"（指向 spec/工单/研究笔记的指针）而非复制信息；最终在单条 integration branch 上汇合。这是把 LLM 编排中的 fan-out/fan-in 模式写成了给 agent 读的说明书。

### 5.5 双轴代码评审（`skills/engineering/code-review/SKILL.md`，89 行）

对固定基点以来的 diff 跑两个并行子代理：Standards 轴（对照仓库编码规范 + Fowler 坏味道基线）与 Spec 轴（忠实实现原 spec？），互相隔离避免污染（CHANGELOG #1208 记录了其演变为前台运行并消费报告的过程）。

## 6. 构建与运行

面向使用者（来自 `README.md` 安装节）：

```bash
# Claude Code（官方 marketplace，自动更新）
claude plugin install mattpocock-skills@claude-plugins-official
# Codex
codex plugin marketplace add mattpocock/skills
codex plugin add mattpocock-skills@mattpocock
# 任意其他 agent / 可编辑文件方式
npx skills@latest add mattpocock/skills -a <agent>
# 安装后在每个仓库跑一次
/setup-matt-pocock-skills
```

面向维护者（来自 `package.json` 与 `CLAUDE.md`）：

```bash
npm ci                      # 仅装 @changesets/cli
npm run changeset           # 记录变更
npm run version             # changeset version + 同步 plugin.json 版本
npm run check-plugin-version  # 校验两处版本一致
claude plugin validate . --strict   # 触碰插件清单后校验（CLAUDE.md）
bash scripts/link-skills.sh # dev-only：symlink 安装到本机 harness
```

无构建产物、无运行时——"构建"就是版本号同步与清单一致性校验。

## 7. 测试与工程质量

- **无自动化测试**：`.github/workflows/` 仅 3 个工作流（release.yml、needs-info.yml、triage-label.yml），没有 CI 测试。技能本身是提示词，"测试"即真实使用；`needs-info.yml` 头部注释写着 14 天无回复自动关闭 needs-info issue，`triage-label.yml` 自动打标签——issue 治理自动化替代了传统 CI。
- **质量抓手是约定与交叉引用**：`CLAUDE.md` 规定 promoted 技能三处强制同步（README、plugin.json、docs 页），`ask-matt` 路由器不同步即"说谎的路由器"；`sync-plugin-version.mjs --check` 提供版本漂移检测。
- **工程化痕迹密集**：2 篇 ADR（`.agents/adr/0002-ship-as-a-claude-code-plugin.md` 记录为何以插件形式发布）、9 篇 out-of-scope 拒绝记录、changesets 全量变更日志。对纯内容仓库而言，这套治理水准罕见地高。
- 明确的写作规范：全仓库禁用 em-dash（`CLAUDE.md`）；所有派子代理的技能必须声明 fresh context window（`CLAUDE.md` 末节，针对 Codex 默认 fork 父对话的问题）。

## 8. 社区与维护状态

- **极度活跃**：创建于 2026-02-03，9 个月内涨到 30 万+ 星（gh api 实时 306,388）；最后推送 2026-10-29（分析日前一天）；最新 release v1.3.2 发布于 2026-10-29，即当天发版。issue/PR 编号已到 #1341（`CHANGELOG.md`），平均每天约 5 个 issue/PR 编号消耗，速度惊人。
- **单维护者结构**：contributors API 返回 4 人；CHANGELOG 中几乎全部条目署名 @mattpocock，外部贡献以问题报告和小修复为主（如 @lofi-coding、@richardwhatever、@adamslowe）。205 个 open issue（gh api）主要来自用户使用反馈。
- **分发即增长引擎**：README 声称 6 万人订阅其 newsletter；进入 `claude-plugins-official` 官方插件市场（`README.md` 安装节）让技能随 Claude Code 生态自动分发。星数暴涨是"作者影响力（Total TypeScript/AI Hero）× agent skills 风口 × 高频迭代"三者的乘积。
- 发布节奏：changesets 驱动，push 到 main 即开 Version PR（`.github/workflows/release.yml`），2026-10 内多次 patch 发版（CHANGELOG 密度可证）。

## 9. 亮点与风险

**亮点**

1. **概念即 API**：grilling 的 design tree/frontier、tdd 的 seam、implement-spec 的 task graph，都是把成熟软件工程概念翻译成 agent 可执行的约束，短小精悍（最核心的 grilling 仅 31 行）。
2. **组合而非框架**：user-invoked 编排层 / model-invoked 纪律层的分层，技能间通过 Skill 工具调用复用，避免了 Spec-Kit 类方案的流程锁定，用户可随意拆改。
3. **内容工程化的标杆**：ADR、out-of-scope 拒绝记录、changesets 语义化版本、三处强制同步的一致性纪律、多 harness 适配清单，示范了"提示词仓库如何做工程治理"。
4. **真实使用驱动的迭代**：CHANGELOG 每条改动关联真实 issue（如 #955 让 diagnosing-bugs 在信任红灯前先 diff 一次强制变异），打磨密度是同类仓库最高的。
5. **自我验证**：仓库自身就用 GLOSSARY.md/SCOPE.md/CLAUDE.md/ADR 结构运行，是推荐实践活的参考实现。

**风险与局限**

1. **效果强依赖 harness 行为**：技能本质是自然语言指令，不同模型/ harness 的服从度差异大；作者自己也须为 Codex 的 fork 语义专门打补丁（`CLAUDE.md`、CHANGELOG #1341），提示词的"可移植性"有上限。
2. **无任何自动化回归手段**：一个措辞改动可能悄悄改变技能行为，且没有测试能发现；一致性靠人工遵守 `CLAUDE.md` 约定。
3. **单点维护**：实质一人维护，9 个月 1300+ issue/PR 编号、205 个 open issue，响应可持续性存疑。
4. **星数与实用规模错配**：30 万星里相当部分是符号性关注（star 作为"收藏 AI 时代方法论"的动作），实际采纳量应以插件安装量衡量（未确认，无公开数据）。

## 10. 总结评价

mattpocock/skills 是 agent skills 元年中内容工程化程度最高、迭代最勤的个人技能集合：它不卖框架、不发抽象层，而是把"盘问对齐、共享语言、TDD 反馈回路、深模块设计"四条经典工程原则压缩成 27 个小而锋利的 Markdown 指令，配上一套比多数代码仓库还严谨的版本、清单与文档治理。30 万星固然有作者个人品牌与风口的加持，但 CHANGELOG 里上千条关联真实 issue 的打磨记录说明它确实在被大量真实使用。适合三类人：想把 agent 从"氛围编码"拉回工程正轨的日常使用者（直接装插件）；想写自己的 SKILL.md 的人（读 grilling/tdd/implement-spec 学怎么把纪律写成指令）；研究多代理编排与提示词治理的人（读 CLAUDE.md、ADR 和 implement-spec 的任务图设计）。它不适合想要"一键全自动流程"的人——作者本人就是反对这个的。
