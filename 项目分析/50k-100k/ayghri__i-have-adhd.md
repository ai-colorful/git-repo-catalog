# ayghri/i-have-adhd 项目分析

> 星数 58439（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：一个安装到 AI 编码助手（Claude Code、Codex、Cursor、OpenCode、Gemini CLI 等）里的"技能/插件"，让模型输出变成 ADHD 友好风格——行动优先、步骤编号、不绕弯子。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | ayghri/i-have-adhd | gh api |
| 星数 | 58439（快照 2026-10-24）；2026-11-02 实测 59806，仍在增长 | gh api |
| 语言 | Python（测试/评测工具链）；核心产物为 Markdown 提示词，另有 TypeScript/JS 适配层 | 源码统计 |
| 许可证 | MIT（Copyright 2026 Ayoub Ghriss） | `LICENSE` |
| 创建时间 | 2026-05-13 | gh api |
| 最近 push | 2026-10-19；最新 release v0.4.1（2026-10-19） | gh api |
| 贡献者 | 30 人 | gh api contributors |
| Fork | 3477 | gh api |

## 2. 项目是什么，解决什么问题

LLM 编码助手的默认回答风格是"先寒暄、再分析、把答案埋在中间、以 Hope this helps 收尾"。本项目的判断是：这种结构对注意力稀缺的读者尤其不友好——真正要动手的信息被稀释了。它把自己定义为"a skill to stop your coding agent from burying the answer"（`README.md`，与 gh api description 一致）。

核心产物是一份约 140 行的 Markdown 提示词 `skills/i-have-adhd/SKILL.md`，内容为 10 条输出规则：首行即下一步动作、多步任务编号、结尾只留一个两分钟内可完成的动作、抑制跑题岔路、每轮重述当前进度、给具体时间估计（"约 15 分钟"而非"要花点功夫"）、让已完成的工作可见、错误语气就事论事、列表上限 5 项、禁止开场白/复述/客套收尾。规则建立在 5 条"ADHD 如何改变阅读"的事实假设上（工作记忆小、启动最难、时间估计失真等，`SKILL.md` "What ADHD changes about reading" 一节）。

设计上它不是单发提示词而是**会话级持久模式**：规则声明"适用于本次会话每一轮回复，不因话题切换失效"，只能用 "stop adhd mode" 关闭（`SKILL.md` Persistence 节）。安装即用 `/i-have-adhd` 激活，或通过 always-on 钩子每轮自动注入。

规则还自带逃生舱："When to break the rules" 一节规定解释类请求可展开、破坏性操作前必须确认（安全优先于简洁）、debug 死循环三连败后停止迭代并质疑假设、系统提示词优先于本技能等 6 种例外（`SKILL.md`）。文末附"发送前自检"清单：删掉宣布式开头、"还有别的吗"式结尾、by-the-way 插叙和空洞修饰词。

作者自述规则改写自 Ramsay 与 Rostain 的《The Adult ADHD Tool Kit》——"适配 LLM 应该如何回答，而非人应该如何安排一天"（`README.md` Credits 节）。它不是医疗工具，"ADHD" 是对输出形状的隐喻。

## 3. 技术栈与依赖

- **核心产物无运行时**：`skills/i-have-adhd/SKILL.md` 是纯 Markdown 提示词（源文件 142 行，含 YAML frontmatter）。
- **Python**：测试与评测工具链。标准库实现（`unittest`、`json`、`subprocess`），无第三方运行时依赖；评测通过调起外部 Agent CLI（如 Claude Code）完成（`scripts/run_evals.py` 顶部 import 仅为标准库）。
- **TypeScript/JavaScript**：平台适配层。Pi/OMP 扩展依赖 `@earendil-works/pi-coding-agent` 的 ExtensionAPI（`extensions/i-have-adhd.ts:4-8`）；Claude Code/Codex 的 always-on 钩子用 Node 实现，另附 sh 与 PowerShell 两个降级实现（`hooks/always-on.mjs`、`hooks/always-on.sh`、`hooks/always-on.ps1`）。
- **运行载体**：Claude Code、Codex、Cursor、OpenCode（V1/V2）、Pi、OMP、Qwen、Kimi、Gemini CLI、Grok，各有一份清单/入口文件（`AGENTS.md` "Runtime entry points" 表）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `skills/i-have-adhd/SKILL.md` | 规范源头：10 条规则的完整正文 |
| `.cursor/skills/i-have-adhd/SKILL.md` | Cursor 用的镜像副本，CI 强制同步 |
| `.claude-plugin/`、`.codex-plugin/`、`.agents/plugins/` | Claude Code / Codex 插件清单与 marketplace 元数据 |
| `hooks/` | `hooks.json`（SessionStart 声明）+ `always-on.mjs/.sh/.ps1` 三实现 |
| `extensions/` | Pi/OMP 原生扩展（状态机 + 上下文兼容垫片） |
| `.opencode/` | OpenCode 自动发现的插件与命令入口 |
| `qwen-extension.json`、`kimi.plugin.json`、`gemini-extension.json`、`GEMINI.md`、`plugin.json` | 其余运行时清单 |
| `evals/` | 14 条用例 `cases.jsonl`、盲评 rubric、已发布的评测结果 `RESULTS.md` |
| `scripts/` | Python 评测编排与判分（run_evals.py 551 行、judge.py 334 行等，共约 1480 行） |
| `tests/` | 7 个 Python unittest 文件（约 1630 行）+ 1 个 OpenCode 驱动 mjs |
| `.github/readme/`、`.github/install/` | README 与 INSTALL 的 10-11 种语言翻译 |
| `AGENTS.md`、`CONTRIBUTING.md` | 面向（人类与 AI）贡献者的地图与流程 |

组织方式：单仓多运行时适配。产品本体只有一个文件，其余全部是"让这一个文件在尽可能多的 Agent 运行时里正确生效"的适配、验证与文档。`AGENTS.md` 明确了单一源头规则：改行为先改 `skills/i-have-adhd/SKILL.md`，再同步 `.cursor` 镜像；各清单文件的版本号须对齐。

## 5. 核心架构与关键实现

### 5.1 规则本体与"持久化"设计（`skills/i-have-adhd/SKILL.md`）

区别于一次性风格提示词，它把"持续生效"写进规则本身：`Persistence` 节声明规则不随话题切换失效，关闭短语是硬编码的 "stop adhd mode" / "normal mode"。frontmatter 里 `disable-model-invocation: true`（`SKILL.md:5`）——只允许用户显式唤起，防止模型自主触发。规则 9（列表上限 5 项）特别注明"只约束展示，不得限制分析、搜索与候选生成的完整性"（`SKILL.md:105-107`），这是对"简洁规则误伤内容"这一常见失败的防御。

### 5.2 Claude Code/Codex 的 always-on 钩子（`hooks/always-on.mjs`）

SessionStart 钩子（`hooks/hooks.json` 声明，matcher 覆盖 startup/resume/clear/compact）读取 opt-in 标记文件 `$CLAUDE_CONFIG_DIR/.i-have-adhd-always`，存在则把剥掉 frontmatter 的规则全文注入 stdout。两个细节见 `hooks/always-on.mjs:15-44`：SKILL.md 路径相对脚本自身位置解析而非信任环境变量；任何异常 `exit(0)`，绝不阻塞会话启动。同一逻辑有 sh（awk 双遍剥 frontmatter，`hooks/always-on.sh:22-38`）与 PowerShell 两个等价实现，三个实现的剥离正则保持一致（`tests/test_always_on_hooks.py` 专门测这个 parity）。

### 5.3 Pi/OMP 扩展的状态机（`extensions/i-have-adhd.ts`，240 行）

最完整的实现。要点：状态以 custom entry 追加进会话日志，恢复会话时重放最后一条得到 enabled 布尔（`getSavedState`，`extensions/i-have-adhd.ts:81-96`）；规则注入用"标记消息"追踪——只有最新标记算数，后续的 disabled 通知会取消先前的规则集，压缩（compaction）后规则被摘要掉、自动重新注入（`rulesAreInContext` + `syncContext`，第 105-159 行）。`extensions/context-compat.ts` 则是对上游 session manager API 演进的防御性垫片：接口缺失或抛错时返回空列表，宁可多注入一次也不让会话启动崩掉。开关短语在输入层拦截（`pi.on("input")`，第 211-235 行），不消耗模型一轮。

### 5.4 评测与发布门禁（`scripts/run_evals.py`、`scripts/judge.py`、`evals/RESULTS.md`）

项目把"一个提示词有没有用"做成了可复现实验：14 条用例（`evals/cases.jsonl`），baseline（裸提示）与 candidate（注入技能正文）各跑 3 试，由同一模型盲评（A/B/C 标签隐藏条件名，`scripts/judge.py` + `evals/rubric.md` 中 judge:begin/end 标记块）。五个维度加权：正确性 35%、自主性 25%、可执行性 20%、安全性 10%、简洁性 10%。评测器还有一个反污染设计：每次 runner 调用在空临时目录里跑，防止"agent 顺手分析评测仓自身"造成条件间不对称污染（`scripts/run_evals.py:22-35` 的 `_neutral_cwd` docstring）。已发布的一轮结果（`evals/RESULTS.md`，2026-08-02，claude-opus-4-8）：candidate 加权 4.473 vs baseline 4.045，简洁性 +1.143，且正确性/安全性均未倒退；但发布门禁判定 FAILED——因为门禁规则"零阻断性发现"是绝对条款，3 条 blocker 导致即使阻断数从 7 降到 3 也不放行。作者把这个失败连同分析一并发布，并指出"照此写法任何候选永远无法通过"是门禁本身的设计缺陷。

## 6. 构建与运行

无构建步骤（纯脚本仓）。安装入口是往任意 Agent CLI 里粘贴一句话，或按 `INSTALL.md`（27KB）按平台逐个安装/验证/更新/卸载/always-on。验证命令来自 `AGENTS.md` "Verification" 节：

```bash
python3 -m unittest discover -s tests -v   # 全部单元测试
python3 scripts/run_evals.py validate      # 校验用例文件
bun scripts/check_context_compat.ts        # 上下文垫片检查
claude plugin validate .                   # Claude Code 清单校验
```

安装示例（Claude Code，`README.md`）：

```bash
claude plugin marketplace add ayghri/i-have-adhd
claude plugin install i-have-adhd@i-have-adhd
```

评测复现流程与一次记录的成本（$2.67 生成 + $0.92 判评）见 `evals/RESULTS.md` 与 `evals/runners.example.json`。

## 7. 测试与工程质量

- 测试规模：7 个 Python unittest 文件约 1630 行，覆盖钩子三实现的行为一致性（`tests/test_always_on_hooks.py`）、判分器（`tests/test_judge.py` 416 行）、评测编排（`tests/test_run_evals.py` 530 行）、OpenCode 插件（`tests/test_opencode_plugin.py` + `tests/opencode_plugin_driver.mjs`）、安装文档与平台清单的一致性（`tests/test_install_docs.py`、`tests/test_omp_package.py`）。另有一条真实场景评测链路 `tests/test_run_scenario_eval.py`（169 行用例 + `evals/scenarios/` 里的故事脚本）。
- CI（`.github/workflows/`）：4 条流水线。`plugin-load-check.yml` 把插件装进临时 CLAUDE_CONFIG_DIR，跑不到 "enabled" 状态就失败（注释里写明这是为了抓 schema 校验抓不到的加载层损坏，ubuntu + windows 矩阵）；`pi-load-check.yml` 验证 Pi 扩展加载；`cursor-skill-sync.yml` 强制镜像副本同步；`claude.yml` 为 Claude 触发的工作流。
- 质量观察：代码注释密度高且解释"为什么"（如 `_neutral_cwd` 的污染不对称论证）；对失败路径有一贯的"永不阻塞用户"原则；`CONTRIBUTING.md` 与 PR 模板 + `AGENTS.md` 中对 AI agent 发评论的专门约束（AI Agora 标签制，仅限 issue #127）显示作者认真对待"AI 贡献者"这一新型社区成员。

## 8. 社区与维护状态

- 活跃度：2026-05-13 创建，5 个月内涨到 58k+ 星（快照 58439，2026-11-02 实测 59806，日均千星级增长在本档位属罕见速度）；最近 push 2026-10-19，与最新 release v0.4.1 同日，维护持续且节奏快。open issues 83（gh api）。
- 贡献者结构：30 名贡献者（gh api contributors），fork 3477——大量 fork 来自用户改 SKILL.md 自定义（README "Tune it" 一节明示这是预期玩法）。
- 传播证据：README 引用了 YouTube 博主 Kacper Rutkiewicz 对早期版本的拆解视频（`README.md`）；11 种语言的 README 翻译（`.github/readme/`）与 9 种语言的安装文档（`.github/install/`）表明社区/作者在主动做跨语言扩散。
- 发布节奏：当前标签 v0.4.1，未见更早 release（gh api releases 仅返回 1 条），版本主要记录在各清单文件里（如 `package.json` 0.4.1），"未确认"更早的版本历史。

## 9. 亮点与风险

**亮点**

1. 把"提示词"当软件工程做：盲评选 eval、加权 rubric、发布门禁、中性 CWD 防污染、CI 验证插件真实加载（`scripts/`、`evals/`、`.github/workflows/`）——这套方法论对提示词产品有普遍参考价值。
2. 多运行时适配完整且防御性好：三语言钩子实现行为对齐并有 parity 测试；Pi 扩展处理了压缩重注入、会话状态重放、开关短语输入层拦截等真实边缘情况（`extensions/i-have-adhd.ts`、`hooks/`）。
3. 诚实文化：评测门禁 FAILED 的结果连同对门禁自身缺陷的批评一起公开在 `evals/RESULTS.md`。
4. 规则设计自带安全阀与例外条款（破坏性操作确认、debug 死循环检测、harness 优先），避免了大多数"风格提示词"一刀切的通病（`SKILL.md` "When to break the rules"）。
5. 文档与本地化投入罕见地高：27KB 分平台 INSTALL、11 语言 README、面向 AI 贡献者的 AGENTS.md。

**风险/局限**

1. 效果依赖模型行为，无运行时保证：SKILL.md 只是一段注入的文本，不同模型/版本上 10 条规则的遵循度会漂移；项目自己的评测也只覆盖单一模型单次快照。
2. 一源十副本的维护负担：SKILL.md + Cursor 镜像 + 10 份清单/入口文件需人工对齐（`AGENTS.md` 只能靠约定与 CI 兜底），contributor 数仅 30，长期同步风险集中。
3. 发布门禁按当前写法永远无法通过（`evals/RESULTS.md` 自证），评测体系对迭代实际不构成放行信号，价值退化为回归对比。
4. always-on 模式每轮注入约 130 行规则文本，有持续的上下文/token 成本；`disable-model-invocation: true` 也意味着用户必须显式行动才生效。

## 10. 总结评价

这是一个"用一个 Markdown 文件定义产品、用整个仓库保证它在十几个 Agent 运行时里可靠生效"的项目：本体是 10 条反直觉但论证严密的输出规则，周边是罕见地完整的评测、CI、测试与本地化体系。星数增长（5 个月 58k）源于击中了 LLM 输出冗长这一普遍痛点 + 易记的仓库名 + 多平台即插即用 + 视频传播，但留存价值在于其评测方法论与适配层工程。适合三类人：想立即改善自己 Agent 回答可用性的普通用户（装了就用）；研究提示词工程/LLM 输出风格的人（读 `SKILL.md` + `evals/rubric.md` + `evals/RESULTS.md` 这条"规则→度量→门禁"链路）；想给自己的 Claude Code/Pi 扩展写适配层的开发者（读 `extensions/i-have-adhd.ts` 与 `hooks/`）。不建议把它当 ADHD 相关的医疗辅助看待——README 与代码中均无此声明，它只是借了这个隐喻的输出整形工具。
