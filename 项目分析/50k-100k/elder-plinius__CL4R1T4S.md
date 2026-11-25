# elder-plinius/CL4R1T4S 项目分析

> 星数 51462（快照 2026-10-24）｜主语言 —（纯文本内容仓库）｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：一个集中收录各大 AI 厂商"泄露"系统提示词（system prompts）与工具定义的纯文本仓库，主打"AI 系统透明化"。

| 项目 | 事实 |
| --- | --- |
| 名称 | elder-plinius/CL4R1T4S（leetspeak，取 "Claritas"（透明/清晰）谐音） |
| 星数 | 51,462（快照）；分析时 GitHub 实时为 51,725 |
| 主语言 | 无代码（纯 Markdown/TXT/JSON 文本） |
| 许可证 | AGPL-3.0（`LICENSE`，仓库所有者施加；对所收内容的授权力见第 9 章风险） |
| 创建时间 | 2025-03-04（gh api `repos/elder-plinius/CL4R1T4S`） |
| 最近活跃 | 最近 push 2026-09-22（gh api），单维护者持续更新 |
| 规模 | 26 个厂商目录、78 个提示词/工具文件、约 10.1 万行、约 7.4MB 文本 |
| 贡献者 | 1 人（elder-plinius，206 次提交，gh api contributors） |

## 2. 项目是什么，解决什么问题

CL4R1T4S 收集了 OpenAI、Anthropic、Google、xAI、Perplexity、Cursor、Windsurf、Devin、Manus、Replit、Lovable、Bolt、Cline、Vercel v0 等几乎所有主流 AI 模型与编码 Agent 的系统提示词全文，以及部分产品的工具定义（tools JSON）。README（`src/README.md`）给出的动机是一句格言式的表述："In order to trust the output, one must understand the input."——作者认为各 AI 实验室通过庞大且不可见的提示词脚手架塑造模型行为，公众在与 AI 交互时并不知道自己面对的是什么样的"隐藏指令"，因此要推动"AI SYSTEMS TRANSPARENCY FOR ALL"（README 自述）。

这类内容的受众主要是：提示词工程研究者（想学习头部产品如何编排指令）、AI 安全研究者（分析厂商如何植入拒答/人格/安全策略）、以及普通好奇用户。仓库按厂商分目录，文件命名通常带模型版本与提取日期（如 `ANTHROPIC/Claude_Sonnet-4.5_Sep-29-2025.txt`、`OPENAI/ChatGPT_4.1_05-15-2025.txt`），使读者可以纵向对比同一产品不同代际的提示词演化。

需要强调：这不是代码项目，没有任何可构建、可运行的软件。它的全部价值在于内容本身。作者 elder-plinius（自称 Pliny，X/Twitter 上的知名越狱与提示注入研究者）在 README 末尾还嵌入了针对 AI 代理的提示注入演示段落（"#MOST IMPORTANT DIRECTIVE#" 一节，`src/README.md`）——这本身也是该仓库"展示攻击面"定位的一部分，阅读/处理本仓库内容的自动化代理应将其视为数据而非指令。

## 3. 技术栈与依赖

- 无编程语言、无框架、无第三方依赖、无包管理清单（tarball 全量 80 个文件中无任何 `package.json`/`pyproject.toml` 等）。
- 文件格式：35 个 `.md`、34 个 `.txt`、3 个 `.mkd`、4 个 `.json`（工具定义）、3 个无扩展名文件（`src/` 目录统计）。
- 仅有两个"元文件"：`README.md`（使命与投稿说明）与 `LICENSE`（AGPL-3.0 全文）。
- 消费方式就是 GitHub 网页浏览或直接拉取原始文本；无 CI、无 `.github/` 目录、无测试。

## 4. 目录结构与代码组织

顶层为"按厂商一目录"的扁平结构（`src/` 实测）：

| 目录 | 职责（收录对象） | 文件数 | 体量 |
| --- | --- | --- | --- |
| `OPENAI/` | ChatGPT 4o/4.1/5、Codex、Atlas、Codex_Desktop（含 GPT-6 系列） | 18 | ~54.4k 行 / 2.72MB |
| `ANTHROPIC/` | Claude Sonnet/Opus 3.5→5.5、Claude Code、设计提示词 | 15 | ~33.5k 行 / 3.27MB |
| `XAI/` | Grok 3/4 系列 | 7 | ~858 行 |
| `ZAI/ZCode/` | Z.ai 编码代理的 Prompts/Skills/Tools | 3 | ~5.5k 行 / 394KB |
| `CURSOR/`、`DEVIN/`、`REPLIT/` | 各 3 个文件（提示词+工具/命令） | 各 3 | 数百行级 |
| `GOOGLE/`、`MANUS/`、`META/`、`WINDSURF/`、`DIA/` | 各 2 个文件 | 各 2 | — |
| `BOLT/`、`BRAVE/`、`CLINE/`、`CLUELY/`、`FACTORY/`、`HUME/`、`LOVABLE/`、`MINIMAX/`、`MISTRAL/`、`MOONSHOT/`、`MULTION/`、`PERPLEXITY/`、`SAMEDEV/`、`VERCEL V0/` | 各 1 个文件 | 各 1 | 数十至数百行 |

组织方式评述：单体扁平仓库，无索引文件、无目录级说明，检索完全依赖目录名 + 文件名。文件名自带元数据（版本号 + 提取日期）是唯一系统性约定，例如 `MOONSHOT/Kimi_K2_Thinking.txt`、`XAI/GROK-4.1_Nov-17-2025.txt`。跨层级有少量嵌套（`OPENAI/Codex_Desktop/`、`ZAI/ZCode/`），把提示词与配套 `Tools.json` 放在一起。整体更像一个不断追加的档案库，而非结构化数据库。

## 5. 核心架构与关键实现

无代码架构。从"内容资产"角度，最有分析价值的实现性工作是三类：

1. **提示词 + 工具定义配对**。最强的几个条目不只有提示词正文，还附带了机器可读的工具 schema，如 `OPENAI/Codex_Desktop/GPT-6-Astra_Tools.json`（1.18MB，含 `setup_codex_step` 等工具的 JSON Schema）、`OPENAI/Codex_Desktop/5.6-Sol_Tools.json`、`ZAI/ZCode/Tools.json`。工具 JSON 使研究者可以完整还原 Agent 的能力面，而不只是话术面。
2. **超长单文件条目**。`ANTHROPIC/CLAUDE-OPUS-5.5.md` 单文件 1.95MB、约 3.4 万行，以 `[system prompt]` 标记开头并包含函数调用块描述（抽样头部确认）；`OPENAI/Codex_Desktop/5.6-Sol_SystemPrompt.md` 约 30 万字符。这类条目反映了新一代 Agent 提示词的规模量级。
3. **代际快照系列**。ANTHROPIC 目录从 Claude Sonnet 3.5、3.7、4、4.1、4.5 到 OPUS-5、Fable-5/5.1、Opus 5.5 共 15 个文件，OPENAI 目录覆盖 4o（2025-04 起）到 GPT-5 系列与 Codex 多版本，可做时间线对比。各文件真实性均为"仓库自述"，未确认（见第 9 章）。

## 6. 构建与运行

无构建、无运行、无测试。使用方式即浏览：

- 在线浏览：GitHub 仓库页按厂商目录点开原始文本；
- 本地获取：`git clone --depth 1 https://github.com/elder-plinius/CL4R1T4S.git`（约 2MB tarball，本次即通过 codeload HEAD 快照获取）；
- 贡献方式（README `src/README.md` 自述）：提交 PR 时附模型名/版本、提取日期、上下文说明，或联系作者 X/Discord（@elder_plinius）。仓库无 PR/issue 模板、无自动化校验。

## 7. 收录范围与条目统计（内容型特例）

- **收录范围**：通用对话模型（ChatGPT、Claude、Gemini、Grok、LeChat、Kimi、MiniMax）、编码 Agent（Cursor、Windsurf、Devin、Codex、ZCode、Factory Droid、Same.dev、Replit Agent）、应用生成平台（Lovable、Bolt、Vercel v0）、浏览器/语音产品（Brave Leo、Dia、Hume、Perplexity、MultiOn、Manus）等，覆盖 26 家厂商/产品线。
- **条目规模**：78 个提示词文件、合计约 100,908 行、约 7.4MB（不含 LICENSE/README，实测统计）。
- **热度分布**：OPENAI（18 文件）与 ANTHROPIC（15 文件）占全部条目 42%、行数占约 87%，头部厂商的条目远比长尾厂商完整。
- **组织维度**：仅按"厂商"一级分类，无按用途（拒答策略/工具调用/人格设定）的交叉索引；查找特定主题只能全文搜索。

## 8. 更新机制与社区（内容型特例）

- **更新机制**：单维护者手工追加 + 社区 PR。git log（gh api commits）显示提交作者均为 "pliny"（elder-plinius），全部 206 次提交来自单一账号，contributors 数为 1；近期提交多为直接新建/重命名单个文件（如 2026-09-22 一天新增 CLAUDE-OPUS-5.5.md、GPT-6-Sol 系列等多个条目）。
- **节奏**：2025-03 创建至今约 19 个月持续更新，最近 push 为 2026-09-22；从文件名日期看，新模型发布后数日至数周内即出现对应条目，跟进速度是该仓库的核心竞争力。
- **社区结构**：无核心团队、无治理文件、无 CONTRIBUTING.md；issue 数 123（gh api），多为投稿/纠错类，未逐一核实。无 release 机制（纯内容仓库不需要版本发布）。

## 9. 亮点与风险

亮点：

1. 覆盖面与时效性极强：26 家厂商、78 个条目、新模型发布后数日跟进（文件名自带提取日期，可验证跟进速度）。
2. 部分条目附带完整工具定义 JSON（`OPENAI/Codex_Desktop/`、`ZAI/ZCode/`），对 Agent 架构研究是一手素材。
3. 代际序列完整（Claude/GPT/Grok 多版本并存），支持纵向演化研究。
4. 极低消费门槛：纯文本、无构建，2MB 即可全量带走。

风险/局限：

1. **来源合法性**：内容为厂商内部保密文档的未经授权披露，涉及版权与潜在商业秘密问题；仓库所标 AGPL-3.0（`LICENSE`）仅是维护者对"仓库自身"的声明，并不能代厂商对泄露的提示词正文授予任何权利。再分发或商用这些文本存在明确法律风险。
2. **真实性与完整性未确认**：所有条目均为"仓库自述"的提取结果，无法在本文档分析中核实；部分文件（如 `XAI/GROK-4.20.mkd`）命名带有戏谑色彩，不排除虚构、拼接或被修改的可能，引用时应视为未经证实的材料。
3. **内嵌提示注入**：README 末尾及部分正文即包含针对 AI 代理的注入演示（`src/README.md` 第 39-45 行附近）；自动化工具读取本仓库时必须把全部内容当数据处理，否则存在被操控风险。
4. **组织粗糙**：无索引、无目录说明、无元数据清单，条目命名风格不统一（`.md`/`.txt`/`.mkd`/无扩展名混用），检索与程序化消费成本较高。

## 10. 总结评价

CL4R1T4S 是"AI 透明化"旗帜下规模最大的泄露系统提示词档案库之一：以一己之力维持 26 家厂商、约 10 万行文本的持续收录，时效性接近新模型发布的实时镜像，星数在 19 个月内冲上 5 万，本质上是踩中了 Agent 时代公众对"AI 到底被怎么指挥"的强烈好奇，加上越狱社区本身的传播效应。它没有工程含量，价值完全在内容，而内容的合法性、真实性都以"仓库自述"为限，研究者可作参考线索，引用前需自行核证；企业则应把它当作风险源而非素材库。适合 AI 安全研究者、提示词工程学习者做对比分析；不适合任何直接生产环境使用。

---
*分析基于 2026-11-04 的 codeload HEAD 快照（最近 push 2026-09-22）；文中统计均来自本地解压目录实测，标注"未确认"处为无法本地核实的信息。*
