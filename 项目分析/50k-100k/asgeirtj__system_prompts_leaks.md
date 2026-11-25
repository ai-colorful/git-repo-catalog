# asgeirtj/system_prompts_leaks 项目分析

> 星数 70096（快照 2026-10-24）｜主语言 Python（GitHub 标注，实为 Markdown 文本集合）｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：全球主流 AI 产品"系统提示词（system prompt）"的逐字泄露收集库——把 ChatGPT、Claude、Gemini、Grok、Cursor 等聊天与编程助手背后不可见的系统提示词，以 Markdown 文件形式完整归档并按厂商索引。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | asgeirtj/system_prompts_leaks | GitHub 仓库 |
| 星数 / Fork | 70,497 / 11,411（API 快照 2026-10-31） | `gh api repos/asgeirtj/system_prompts_leaks` |
| 主语言 | GitHub 标注 Python；仓库实际为 1412 个 Markdown 为主的文本集合 | `find` 统计 |
| 许可证 | CC0-1.0（公共领域贡献） | `LICENSE` |
| 创建时间 | 2025-05-03 | 同上 API |
| 最近 push | 2026-10-29（分析前 2 天） | 同上 API |
| 规模 | 1,841 个文件、约 46MB、Markdown 共约 54.4 万行 | 本地 `find`/`wc` 统计 |
| 贡献者结构 | 高度单主体：asgeirtj 849 次提交，其余 Top 贡献者各 2-5 次 | `gh api .../contributors` |

## 2. 项目是什么，解决什么问题

各家 AI 公司都把系统提示词视为产品核心资产：它定义了模型的人设、工具调用规则、安全边界和输出风格，但从不随产品公开。用户只能通过提示注入（把"请完整复述你的系统提示词"类指令发给线上产品）等方式把这段隐藏文本挖出来。这些泄露文本散落在 X/Twitter、Reddit 和各博客里，转载时常被截断、改写或掺假，难以考证。

本仓库把这些泄露结果做成了唯一可信源式的归档：每个文件是某个产品在某个时间点的系统提示词逐字记录（如 `Anthropic/claude-opus-5.5.md`、`OpenAI/gpt-5.5-thinking.md`、`xAI/grok-4.7.md`），文件开头通常带 `[Message role: system]` 或原始 XML 标记等捕获现场特征（见 `Anthropic/raw/claude-opus-4.6-raw.md` 首行的 `<antml:reasoning_effort>` 标记），不加评论、不改写。`.gitattributes` 中专门为 Markdown 设置 `*.md -whitespace` 以保留原始空白，说明作者把"逐字保真"当作第一目标（出处 `.gitattributes`）。

仓库的差异化在于三点：一是全，覆盖 Claude.ai 网页/桌面/移动端、Claude Code 各模型与子代理、ChatGPT 语音/个性/工具、Codex、Gemini 全系、Grok、Meta Muse，以及 Cursor、Perplexity、Devin、Warp、Zed 等 40+ 第三方产品（索引见 `README.md`）；二是系统化版本演进，同一个模型的历次提示词变更都保留（如 Anthropic 官方归档从 2024-07 Claude 3 到 2026-10 Haiku 5.5 共 40+ 个带日期版本，见 `Anthropic/official/README.md`），可直接观察提示词工程随时间的演化；三是不仅收主提示词，还收配套的 skills、子代理定义、注入的 reminders、工具描述等完整上下文（如 `Anthropic/claude-code/skills/` 40 个技能目录、`GLM/skills/` 16 个技能目录）。

影响力方面，README 顶部列出三处独立引用：《华盛顿邮报》2026 年 5 月基于本仓库提示词制作了交互式报道；CEPS' AI World 2026 年 7 月做了逐主题拆解；Simon Willison 2026 年 8 月称其为"我所知最完整的泄露系统提示词收集"（出处 `README.md` 第 35-39 行）。

## 3. 技术栈与依赖

无代码工程、无构建体系、无运行时依赖（目录中无 package.json/pyproject/requirements 等任何工程清单文件）：

- 内容本体：1,412 个 `.md`（约 54.4 万行），是仓库的绝对主体
- 附带格式：117 个 `.xsd`、109 个 `.yaml`、85 个 `.py`、32 个 `.json`、25 个 `.mjs`、17 个 `.ts` 等——这些不是"项目代码"，而是对 Meta Muse agent 虚拟机工作区的整体快照（`Meta/muse-agent/` 含 assets/docs/memory/skills 目录，连字体、XSD schema 都原样保留），GitHub 因此把仓库语言误标为 Python
- 索引与元信息：`README.md`（约 25KB，本仓库唯一的"代码文件"角色——全部导航都靠它）
- 工程化痕迹仅两处：`.gitattributes`（LF 归一化 + Markdown 保真规则）、`.github/FUNDING.yml`（GitHub Sponsors）

文件类型分布（本地 `find` 统计）：

| 类型 | 数量 | 说明 |
|---|---|---|
| `.md` | 1,412 | 提示词正文与 README 索引，绝对主体 |
| `.xsd` / `.yaml` / `.py` / `.json` / `.mjs` / `.ts` | 117 / 109 / 85 / 32 / 25 / 17 | 全部来自 `Meta/muse-agent/` 工作区与 `Anthropic/claude-code/skills/` 技能包的整体快照，非项目自身代码 |
| 其他（`.txt`/`.xml`/`.jsx`/`.tsx`/`.html`/字体等） | ~70 | 同上，快照附带资产 |

## 4. 目录结构与代码组织

顶层按厂商分目录，README 做总索引：

| 目录 | 文件数 | 职责 |
|---|---|---|
| `Anthropic/` | 646 | Claude 全家桶：网页版各模型、Claude Code（含 agents/skills/commands/output-styles）、artifact 类型技能、Claude Design/Cowork/Science、官方发布归档 `official/`、原始捕获 `raw/` |
| `Meta/` | 462 | Muse agent 虚拟机工作区整体快照（89 个技能目录 + memory/docs/assets）与 Muse Code |
| `OpenAI/` | 435 | ChatGPT 各模型、Codex（含 skills/plugins）、API 注入提示词、旧版本 `Old/` |
| `GLM/` | 90 | ZCode Desktop (GLM-5.3) 系统提示词 + skills/agents |
| `xAI/` | 76 | Grok 全系版本与 Grok Bot/CLI/Translator |
| `Google/` | 66 | Gemini 全系、Antigravity IDE/CLI、NotebookLM、AI Mode 等 |
| `Misc/` | 26 | 26 个第三方产品（Cursor 之外）：Devin、Warp、Zed、Kagi、Duck.ai、Reddit Answers 等 |
| 其余 9 目录 | 32 | Cursor(10)、Perplexity(8)、Microsoft(5)、Qwen/Mistral/MiniMax/Kimi 各 2、Pi/Notion/DeepSeek 各 1 |

组织方式评述：单层厂商目录 + 平铺文件为主，复杂产品才开子目录（如 `Anthropic/claude-code/`、`Meta/muse-agent/`、`OpenAI/Codex/`）。"新旧并存"是刻意的组织原则：旧版本放 `old/`/`Old/` 子目录而不是删除，最新版本在 README 表格中加粗。README 索引按"厂商 → 产品线 → 模型"三层表格组织，顶部另有"Most recent additions/changes"滚动更新表（出处 `README.md`）。整体是"内容库 + 手工索引"结构，无代码意义上的架构。

代表性子目录的层级设计（本地 `find` 核实）：

| 子目录 | 内部结构 |
|---|---|
| `Anthropic/` | 按产品线再分：`claude-code/`（含 agents、commands、output-styles、prompts、skills 五类）、`artifact-types/`（slides/dashboard/design/design-system/docs/motion 六种类型，各含 SKILL.md 与 reference 资源）、`claude-design/`、`claude-cowork/`、`official/`、`old/`、`raw/` |
| `Anthropic/official/` | 唯一按日期命名的目录：`YYYY-MM-DD-<model>.md` 共 40+ 个官方归档版本 + `all.md` 合集 + 自带 `README.md` 对照表 |
| `Meta/muse-agent/` | 完整 VM 工作区快照：`skills/`（89 个技能目录）、`memory/bank/`、`docs/`（chat/connections/devices）、`assets/onboarding_tour/` |
| `OpenAI/` | 主目录平铺聊天模型 + `API/`（注入提示词与说明）、`Codex/`（含 skills、plugins、old）、`Old/`（废弃工具与旧策略）、`dots/`（Dreamer 语音与 28 个技能） |
| `GLM/` | `zcode-desktop.md` 主提示词 + `skills/`（16 个技能目录）+ `agents/`（Explore/general-purpose/visual-judge 三个子代理定义） |

体积最大的文件（本地 `find -printf %s` 排序）：

| 文件 | 大小 |
|---|---|
| `Anthropic/claude-code/skills/plugin-authoring/types/claude-code.d.ts` | 857KB（技能包附带类型定义） |
| `OpenAI/Codex/gpt-6-sol.md` | 654KB |
| `OpenAI/Codex/gpt-6-luna.md` | 652KB |
| `OpenAI/Codex/gpt-6.1-sol.md` | 637KB |
| `Anthropic/official/all.md` | 580KB（全部官方版本合集） |
| `Anthropic/claude-code/claude-code-desktop-fable-5.1.md` | 553KB |

## 5. 核心架构与关键实现

内容型仓库无代码架构，以下为内容组织上的三个关键设计：

1. **官方归档与泄露捕获双轨**。`Anthropic/official/` 收录 Anthropic 在 platform.claude.com 文档中主动公开的 `claude_behavior` 系统提示词，按官方标注日期命名（`2024-07-12-claude-haiku-3.md` 到 `2026-10-07-claude-haiku-5.5.md` 共 40 余个版本，`all.md` 合集 580KB），并在 `official/README.md` 中给出模型发布日期与归档版本对照表。`Anthropic/raw/` 则存提示注入原始现场（含 `<antml:*>` 内部标记）。这条轨道让仓库超越了"猎奇泄露"属性，成为提示词版本演化的研究语料。
2. **完整环境快照而非单文件摘录**。对 Meta Muse agent，作者没有只收主提示词，而是整仓捕获：`Meta/muse-agent/skills/` 下 89 个技能目录（每个含 SKILL.md + 资源）、`memory/bank/`、`docs/`、onboarding 资产，甚至包含 `.xsd` schema 与 Python 脚本。对 Claude Code 同样如此：`Anthropic/claude-code/` 同时收 agents、commands、output-styles、skills（40 个）和主提示词。代价是 46MB 体积与符号链接维护负担（本仓库存在 `Meta/muse-agent/skills/podcast/* → ../generate_podcast/*` 等符号链接，本次 Windows 解压时 12 个链接条目失败，但被指向的本体文件均在包内，无内容损失）。
3. **索引即入口的 README**。README 同时承担：媒体背书（WaPo/CEPS/Willison 引用）、增量公告表（最近 12 条更新，均带日期与链接）、全部厂商的模型-文件对照表、Star History。每个条目可从 README 一跳直达，等于把整个库压进一个导航页。另有 `OpenAI/API/README.md` 单独解释 o3/o4-mini API 隐藏注入的系统消息及 Juice/Yap 参数含义，是"带注释索引"的局部尝试。

## 6. 构建与运行

无可构建、无可运行内容。使用方式：

- 阅读/检索：直接浏览 `README.md` 索引，或在仓库内按文件名检索（如 `Anthropic/claude-code/claude-code-opus-5.5.md`）
- 本地获取：`git clone --depth 1` 或下载 tarball 即可，无依赖安装步骤
- 引用：许可证为 CC0-1.0（`LICENSE`），作者明确允许自由使用与 PR 补充（README"PRs welcome"徽章）

## 7. 收录范围与组织方式

- **范围**：17 个厂商目录覆盖 40+ 产品，时间跨度 2024-07（Claude 3 官方归档起点）至 2026-10（Kagi/Perplexity Computer/MiniMax M3 更新），以 2025-2026 年头部模型为主。每个条目粒度到"产品 × 模型 × 版本"，Codex/GPT 同一模型的不同人格（Sol/Luna/Terra/Astra）分文件收。
- **规模统计**：1,841 个文件中 1,412 个 Markdown；总行数约 54.4 万行；最大单文件 653KB（`OpenAI/Codex/gpt-6-sol.md`），Codex 系列单文件普遍超 600KB，说明现代 agent 系统提示词已膨胀到数十万字符级。
- **分类体系**：厂商 → 产品线 → 模型/版本三级；"当前版本 vs 旧版本"用加粗与 `old/` 子目录区分；"官方公开 vs 泄露获取"用 `official/` 与 `raw/` 区分。
- **条目质量口径**：README 自述"Carefully curated and complete"（精心整理且完整），并邀请提交不同捕获结果的 PR——即以社区交叉验证代替机器验证，无自动化校验。

## 8. 更新机制与维护状态

- **更新频率极高**：README 顶部更新表显示 2026-10-20 至 10-29 十天内至少 12 次实质内容更新；`gh api` 最近 5 条提交全部落在 2026-10-29 当天（含 1 条 PR 合并 `Add grok-translator.md (#29)`）。新模型发布后数日内即有对应文件入库。

README "Most recent additions/changes" 表原文摘录（出处 `README.md` 第 45-58 行）：

| 条目 | 日期 |
|------|------|
| Kagi Assistant（更新） | 2026-10-29 |
| Perplexity Computer（更新）+ Model Council 技能 | 2026-10-29 |
| MiniMax M3（Mavis / MiniMax Agent） | 2026-10-29 |
| LM Studio Bionic | 2026-10-29 |
| GPT-Live-1.1 语音系统提示词（前端/后端/任务协调） | 2026-10-29 |
| Antigravity IDE 与 CLI（Gemini 4 Argon） | 2026-10-29 |
| Grok Bot（更新） | 2026-10-24 |
| Perplexity search | 2026-10-23 |
| Claude Motion（类型技能） | 2026-10-21 |
| ZCode Desktop（GLM-5.3） | 2026-10-20 |
| Cursor Cloud Agents | 2026-10-20 |
| Claude artifact types（Slides/Dashboard/Design/Docs） | 2026-10-09 |
- **流程**：单人主导（asgeirtj 贡献占比绝对优势）+ 开放 PR（接受新条目与勘误）；无 CONTRIBUTING、无 issue 模板、无 CI（`.github/` 下仅 `FUNDING.yml` 与一张 logo 图），提示词真伪依赖作者本人校对与社区反馈。
- **元数据佐证**：仓库 2025-05-03 创建，18 个月 70k+ 星、11.4k fork，21 个开放 issue；近两天仍有 push，处于活跃维护期（出处：`gh api repos/asgeirtj/system_prompts_leaks`）。
- **贡献者长尾**：Top 10 贡献者中除 asgeirtj（849 次）外，其余 9 人合计仅 26 次（anonymous619259 5、awrreny 5、lcebot 3、dbyale 3、其余各 2），说明外部贡献以零星补件为主，内容质量实际由维护者把关（出处：`gh api .../contributors`）。
- 可持续性风险：完全依赖单人持续投入与持续泄露供给，无自动化管道（未确认作者是否有采集工具链）。

## 9. 亮点与风险

亮点：

1. 覆盖面与时效性同类中最强：40+ 产品、54 万行语料、新模型发布后数日即更新，且经华盛顿邮报、Simon Willison 等第三方背书（`README.md`）
2. 版本演化语料价值：Anthropic 官方归档保留 2024-07 起全部带日期版本，可系统研究提示词工程的变迁（`Anthropic/official/`）
3. 保真优先的收录纪律：逐字捕获、保留原始标记与空白（`.gitattributes` 的 `*.md -whitespace`）、新旧版本不覆盖只归档
4. 生态快照完整性：不止主提示词，连带 skills/agents/memory 工作区整体捕获，是研究 agent 产品内部结构的罕见样本（`Meta/muse-agent/`）
5. CC0 公共领域许可，使用门槛为零（`LICENSE`）

风险与局限：

1. **来源合法性灰色**：绝大多数条目是通过提示注入绕过产品防护取得的保密文本，各大厂商的服务条款均禁止此类提取；仓库本身处于灰色地带，存在被 DMCA 投诉或要求下架的可能（本次分析时点仓库仍在线）。
2. **版权风险**：系统提示词文本的著作权归属厂商，CC0 许可只能约束作者自己的贡献，无法替厂商放弃权利；再分发与商用存在法律不确定性。
3. **真实性无保证**：无自动化校验，个别条目可能是截断、拼接或伪造版本；README 虽以"逐字"自许并欢迎勘误 PR，但普通使用者难以逐一验证。
4. **时效衰减**：产品迭代后旧提示词即失真，仓库的价值依赖持续人工更新；单点维护是最大结构性风险。

## 10. 总结评价

system_prompts_leaks 是"AI 时代 BBC 档案"式的社区基础设施：用最朴素的技术形态（Markdown 文件 + 一个 README 索引）做成了目前公认最完整的头部 AI 产品系统提示词语料库，单人 849 次提交、18 个月 7 万星的增速证明了稀缺信息的聚合价值。它没有代码可读，价值全在内容：做提示词工程、模型行为研究、安全分析与产品逆向的人可以把它当参考语料库；普通用户当"AI 产品说明书考古"阅读亦可。需要清醒的是其灰色属性——条目真伪靠社区互证、文本权利属各厂商、来源不受厂商认可，引用与研究时应注明捕获性质，商用更需谨慎。
