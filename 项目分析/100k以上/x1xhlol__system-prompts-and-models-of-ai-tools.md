# x1xhlol/system-prompts-and-models-of-ai-tools 项目分析

> 星数 144210（快照 2026-10-24）｜主语言 —（纯文本收集仓库）｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：互联网上规模最大、更新最勤的"各家商业 AI 工具系统提示词泄露合集"，被称为 AI 圈的提示词情报库。

| 项 | 值 | 出处 |
|---|---|---|
| 仓库 | x1xhlol/system-prompts-and-models-of-ai-tools | gh api |
| 星数 | 144210（快照），forks 34700 | 任务参数 / gh api |
| 创建时间 | 2025-03-05 | gh api |
| 最近 push | 2026-10-12 | gh api |
| 主语言 | 无代码，94 个 .txt + 17 个 .json 纯文本 | `find` 统计 |
| 许可证 | GPL-3.0（`LICENSE.md`，完整 GPLv3 文本） | LICENSE.md |
| 规模 | 122 个文件，提示词类文本约 1.97MB / 3.7 万行 | find/wc 统计 |
| 默认分支 | main | gh api |

## 2. 项目是什么，解决什么问题

各家头部 AI 产品（Cursor、Claude Code、Windsurf、Manus、Devin、v0 等）的系统提示词都是保密的，但它们会随产品分发给客户端或模型端，因此可以被逐字提取出来。本仓库由作者 x1xhlol（网名 Lucknite）持续收集这些提取结果，以原始文本形式公开，让任何人都能读到"这些产品到底是怎么写提示词的"（`README.md`、各目录 `Prompt.txt`）。

它解决的实际问题有三层。第一是学习参考：工程师可以直接研究一线产品如何组织角色设定、工具定义、安全条款与输出格式，例如 Cursor 的 Agent Prompt 演进了 v1.0 → v1.2 → 2.0 多个版本（`Cursor Prompts/` 目录下按版本命名的文件）。第二是工具定义（tool schema）情报：多个目录附有 `Tools.json`，完整记录产品暴露给模型的工具名、描述与输入 schema，如 `Anthropic/Claude Code/Tools.json`、`Replit/Tools.json`、`NotionAi/tools.json`。第三是行业情报：从提示词的措辞、模型名称（如 VSCode Agent 目录按 `gpt-5.txt`、`gemini-2.5-pro.txt`、`claude-sonnet-4.txt` 分文件）可以推断各家产品背后的模型选型与更新节奏。

仓库自带明显的作者商业化色彩：README 顶部是"LeaksLab Discord"徽章和 Trendshift 徽章，中部推广作者自己的提示词安全审计服务 ZeroLeaks，并附 BTC/LTC/ETH 打赏地址、Patreon/Ko-fi 链接和赞助商招募邮箱（`README.md`）。作者还借着这个仓库做起了"你的提示词安全吗"的生意——README 明确警告 AI 初创公司"暴露的提示词很容易成为攻击目标"。

## 3. 技术栈与依赖

内容型仓库，无任何代码、构建系统或第三方依赖。文件构成（find 统计）：

- 94 个 `.txt`：系统提示词正文（约 1.97MB），部分带原始对话标记（如 `Cursor Prompts/Agent Prompt 2.0.txt` 开头保留 `<|im_start|>system`）
- 17 个 `.json`：工具定义（name/description/input_schema 结构），如 `Anthropic/Claude Code/Tools.json`
- 2 个 `.yaml`：`Amp/claude-4-sonnet.yaml`、`Amp/gpt-5.yaml`，按模型分版的 Amp 系统提示词
- 3 个 `.md`：根 README、根 LICENSE、`Amp/README.md`（记录如何用 Alt+点击查看 Thread YAML 提取提示词的操作步骤）
- 4 个 `.png`：`assets/` 下赞助商 logo（latitude、tembo 明暗两版），README 未展示但留在仓库内
- `.github/` 仅 `FUNDING.yml`（patreon: lucknite / github: x1xhlol / ko_fi: lucknite），无 CI、无 issue 模板、无 CONTRIBUTING

"技术栈"就是 Markdown + 原始文本 + git 本身。文件命名即元数据：日期（`Agent Prompt 2025-09-03.txt`）、版本号（`Agent Prompt 2.0.txt`）、底层模型（`Agent Prompt (gpt.5.6.sol) 2026-09-11.txt`、`VSCode Agent/gpt-5.txt`）全部编码进文件名，仓库内不存在任何索引文件将其汇总成清单。

## 4. 目录结构与代码组织

顶层 33 个工具目录 + assets，目录名即产品名，代表产品如下（find 列举）：

| 目录 | 内容 |
|---|---|
| `Anthropic/` | 10 文件：Claude Code（Prompt+Tools）、Claude Code 2.0、Sonnet 4.5/4.6/5、Claude Fable 5、Claude for Chrome |
| `Cursor Prompts/` | 7 文件：Agent Prompt v1.0/v1.2/2.0/按日期版、Agent Tools、Chat Prompt |
| `Open Source prompts/` | 18 文件：Bolt、Cline、RooCode、Codex CLI、Gemini CLI、Qwen Code、OpenCode（按后端模型再分 8 份）、Kilo Code、MiMo Code、Hermes、Lumo |
| `VSCode Agent/` | 9 文件：按底层模型分文件（gpt-4o/gpt-5/gemini-2.5-pro/claude-sonnet-4 等）+ tab 补全提示词 |
| `Manus Agent Tools & Prompt/` | Prompt + Agent loop + Modules + tools.json 四件套 |
| `Xcode/` | 6 文件：按功能拆（System/DocumentAction/ExplainAction/MessageAction/PlaygroundAction/PreviewAction），是"一条产品线拆成多个功能提示词"的典型 |
| `Poke/` | 7 文件：`Poke agent.txt` + `Poke_p1`~`p6`，按页/分段拆分收录 |
| `Google/` | Antigravity（Fast Prompt、planning-mode）+ Gemini（AI Studio vibe-coder，1644 行，为全仓最大单文件） |
| `Devin AI/` | `Prompt.txt` + `DeepWiki Prompt.txt`（连带其文档问答产品一并收录） |
| 其余 | Windsurf（Prompt/Tools Wave 11）、Replit、Lovable、v0、Same.dev、Perplexity、NotionAi、Warp.dev、Trae、Kiro、Qoder、Augment Code、Comet Assistant、Cluely、Emergent、Orchids.app、Leap.new、Traycer AI、CodeBuddy Prompts、Junie、dia、Z.ai Code |

组织方式评述：平铺式单层目录，无索引文件、无分类元数据、无自动化脚本；一个工具一个目录，目录内以 `Prompt.txt` + `Tools.json` 为惯例命名（Anthropic 的 Claude Code 与 Claude for Chrome、Replit、v0、NotionAi、Traycer AI、Windsurf 均遵循此对），版本演进靠文件名内嵌日期或版本号（如 `Agent Prompt 2025-09-03.txt`、`Agent Prompt (gpt.5.6.sol) 2026-09-11.txt`）。粒度策略不一：Manus 按 Prompt/Agent loop/Modules/tools.json 四件套拆分，Xcode 按功能动作拆 6 份，Poke 按 p1~p6 分段，Amp 则按底层模型分 yaml——反映出各产品提示词的天然结构差异，作者未强行统一。检索全靠 GitHub 目录浏览或全文搜索。这种"零加工"结构让提交成本极低，是作者能高频更新的原因之一。

## 5. 核心架构与关键实现

无代码架构，以下剖析三份代表性内容：

**Claude Code 系统提示词与工具定义**（`Anthropic/Claude Code/Prompt.txt`、`Tools.json`）：Prompt.txt 开头即"You are an interactive CLI tool that helps users with software engineering tasks"，随后是防御性安全限定条款（拒绝恶意代码）、URL 生成约束、`/help` 反馈引导。Tools.json 200+ 行 JSON，每个工具含 name、description（含"何时不用该工具"的反例段落）与完整 input_schema，Task 工具的描述中还内嵌了 4 个使用示例，是研究"工具描述工程质量"的完整样本。

**Cursor Agent Prompt 的版本演进**（`Cursor Prompts/`）：v1.0 → v1.2 → 2.0 → 2025-08-07 CLI 版 → 2025-09-03 版共 5 个提示词快照保留在同一目录，`Agent Prompt 2.0.txt` 开头保留了 `<|im_start|>system` 原始标记和"Knowledge cutoff: 2024-06"字样，说明文件是未经清洗的原始提取。772 行的规模使其成为观察产品提示词如何随功能迭代膨胀的少数公开样本。

**Amp 的提取方法论**（`Amp/README.md`）：全仓库唯一一处"提取教程"——登录 VSCode 版 Amp，发一条查询，按住 Alt/Option 点击 workspace 按钮即可查看 Thread YAML，从而拿到原始系统提示词；README 还说明了如何改 VSCode 用户设置切换到 GPT-5 调优版提示词再提取一次。这解释了 `claude-4-sonnet.yaml` / `gpt-5.yaml` 按模型分版的来源，也说明本仓库的原料并非厂商主动公开。

## 6. 构建与运行

无构建、无测试、无运行时。使用方式即浏览：

- 网页浏览：`https://github.com/x1xhlol/system-prompts-and-models-of-ai-tools/tree/main/<工具目录>`
- 本地检索：`git clone --depth 1` 后 `grep -ri "<关键词>" .`
- DeepWiki 链接（README 徽章）提供 AI 辅助问答入口
- 克隆体积约 1.7MB（gh api size=1725KB），几乎无获取成本

## 7. 收录范围与组织方式

收录范围覆盖五大类产品（据目录名归纳）：编程 Agent/IDE（Cursor、Windsurf、Claude Code、VSCode Agent、Xcode、Trae、Kiro、Qoder、Amp、Augment Code、Junie、CodeBuddy、Z.ai Code、Traycer）、应用生成器（v0、Lovable、Same.dev、Replit、Bolt、Orchids.app、Leap.new、Emergent）、通用 Agent（Manus、Devin AI、Perplexity、NotionAi、Poke、Cluely、Comet Assistant、dia、Warp.dev）、浏览器助手（Comet、Claude for Chrome）以及开源工具提示词（Cline、RooCode、OpenCode、Qwen Code、Gemini CLI、Codex CLI 等 18 文件）。共 33 个工具目录、122 个文件，提示词正文约 3.7 万行。

组织方式的特点：按产品平铺、文件即快照、不做任何二次加工或评论；同一产品多版本并存（Cursor 5 份、OpenCode 按后端模型 8 份），文件名内嵌日期（2025-08、2025-09、2026-09）承担版本管理职责。无 CONTRIBUTING、无条目模板、无 PR 自动化——`.github/` 下只有 FUNDING.yml，说明收录完全由作者单人手工驱动。

## 8. 更新机制与维护状态

- 最近 push 2026-10-12（gh api），与 tarball 内全部文件 mtime（2026-10-12 23:19）一致，README 自述"Latest Update: 12/07/2026"，三处证据互相印证，仓库处于活跃更新状态
- 文件名日期显示 2026-09-11 有一批开源工具提示词集中收录（9 个文件），说明存在"批量补货"式更新节奏
- 163 个 open issues（gh api），主要用途推测为提交新泄露线索（无 issue 模板佐证，未确认具体形态）
- 单人维护：贡献结构未确认（tarball 无 git 历史），但 README 全程第一人称、打赏账户统一为 lucknite/x1xhlol，姊妹动作是自有服务 ZeroLeaks 和 LeaksLab Discord 社区（README）

## 9. 亮点与风险

**亮点**

1. 稀缺性独一无二：33 个商业产品的原始系统提示词 + 工具 schema 一站式可得，多数内容厂商从未公开，是研究提示词工程事实标准的第一手素材（各目录 Prompt.txt/Tools.json）
2. 保留原始提取形态：不清洗、不改写，连 `<|im_start|>system` 标记和对话包装都保留（`Cursor Prompts/Agent Prompt 2.0.txt`），可信度和研究价值高于转述型文章
3. 版本快照意识：同一产品多版本并存（Cursor 5 版、Claude Code 1.0/2.0），能观察提示词随产品迭代的演化（`Cursor Prompts/`、`Anthropic/`）
4. 维护活跃且成本极低：单人手工更新仍能保持月度级 push（gh api: 2026-10-12）
5. 生态背书强：34.7k forks、Trendshift 徽章、DeepWiki 收录（README、gh api）

**风险与局限**

1. 版权与法律风险（本仓库最核心的问题）：所有提示词均为各厂商专有内容，未获授权发布。仓库自标 GPL-3.0（`LICENSE.md`）在法律上站不住脚——作者无权对自己不拥有版权的泄露文本授权。收录的 Anthropic/Google/OpenAI 提示词通常带有"不得重分发"条款，DMCA 下架风险持续存在
2. 获取手段灰色：`Amp/README.md` 公开了绕过产品 UI 的提取步骤，本质属于对厂商系统的未授权提取，仓库名字面也自称 "Leaks"，存在被认定为协助违反服务条款的争议
3. 真实性无法验证：无来源说明、无提取时间元数据、无与实际产品的比对证据，个别文件可能是过期、篡改或坊间流传版本（全仓库无 README 级别的溯源声明，README.md 仅 63 行）
4. 组织粗糙：无索引、无分类、无条目元数据，33 个目录的发现成本随规模增长而上升；README 已从早期的工具索引表退化为打赏/广告页，导航价值很低

## 10. 总结评价

这是一个"情报库"而非"工程库"：它把散落在各商业 AI 产品里的系统提示词和工具定义集中成 122 个原始文本文件，用最低的组织成本换来了最高的覆盖度与更新频率，并以 144k 星成为 GitHub 上该领域事实上的唯一入口。它对提示词工程师、Agent 产品研究者和安全研究者价值极高——可以直接对标一线产品如何写角色设定、工具描述和安全条款；但它同时踩在版权与灰色获取的线上，自标 GPL-3.0 并不能改变内容本身未经授权的事实，引用其中任何文本都应意识到法律与伦理边界。适合把它当参考资料库查阅，不适合当可复分发的内容源使用；读它的正确姿势是"学习写法"而非"搬运文本"。
