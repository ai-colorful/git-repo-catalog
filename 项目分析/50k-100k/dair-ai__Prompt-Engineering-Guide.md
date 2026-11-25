# dair-ai/Prompt-Engineering-Guide 项目分析

> 星数 79460（快照 2026-10-24）｜主语言 MDX｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：全球最知名的提示工程（Prompt Engineering）系统性免费教程，即 promptingguide.ai 网站的源仓库，覆盖技术、论文、模型、风险与多语言翻译。

| 项目 | 内容 |
|---|---|
| 名称 | dair-ai/Prompt-Engineering-Guide |
| 星数 | 79,460（快照 2026-10-24） |
| 主语言 | MDX（内容型仓库，非传统代码项目） |
| 许可证 | MIT（LICENSE.md，Copyright 2022 DAIR.AI） |
| 创建时间 | 2022-12-16（gh api repos 元数据） |
| 最近推送 | 2026-03-11（gh api `pushed_at`，与解压文件 mtime 2026-03-12 一致） |
| Fork / Watch / 贡献者 | 8,744 / 748 / 约 211（contributors 分页推算） |
| 官网 | https://www.promptingguide.ai/ |
| Release | 0（无发布版本，内容持续滚动更新） |

## 2. 项目是什么，解决什么问题

ChatGPT 爆发前夕（2022 年 12 月），DAIR.AI 的 Elvis Saravia 创建了这份指南，系统回答一个当时无人认真回答的问题：如何有效地与大语言模型"对话"。它不是链接合集，而是有完整知识体系的教程：从提示基础（指令、上下文、角色、few-shot 示例）到 CoT、ToT、ReAct、RAG、Automatic Prompt Engineer 等十余种经典技术，每篇都给出论文出处、可复现的 prompt 示例和真实输出（`pages/techniques/cot.en.mdx` 中"奇数求和"示例即 CoT 论文 Wei et al. 2022 的经典复现）。

内容按学习路径组织为 15 个板块：Introduction（基础要素）、Techniques（19 篇技术）、AI Agents（2025-2026 年新增：agent 组件、context engineering、deep agents）、Applications（函数调用、生成数据、编码）、Prompt Hub（分类/抽取/问答等 12 类现成 prompt）、Models（GPT-4、Llama、Gemini、Mistral、Mixtral、Phi-2 等）、Risks（对抗攻击、幻觉、偏见）、Research（Groq、Infini-attention、ThoughtSculpt 等论文解读）、Papers、Tools、Notebooks、Datasets、Readings（板块导航见 `pages/_meta.en.json`）。

除教程文本外，仓库还内置配套实践资产：`notebooks/` 下 12 个 Jupyter Notebook（pe-lecture、pe-pal、pe-rag、react.ipynb、gemini-context-caching 等），`lecture/` 下 1 小时视频课的 PDF 讲稿，形成"读教程 → 跑 Notebook → 看讲课"的完整学习闭环（README.md "Lecture" 一节）。

它是提示工程领域的"事实标准教材"：2023-02-21 登顶 Hacker News（README.md Announcements），被 WSJ、Forbes 报道，2024 年 1 月宣布学习者突破 300 万（README.md），被大量 LLM 课程与论文引用，并提供 BibTeX 引用条目（CITATION.cff）。

## 3. 技术栈与依赖

- **文档站框架**：Nextra 2.13.2 + nextra-theme-docs 2.13.2，基于 Next.js 13.5、React 18（package.json）。package name 仍是 `nextra-docs-template`，repository 字段也指向 shuding/nextra-docs-template——即整个站是在 Nextra 官方文档模板上改造的，工程 footprint 极小。
- **内容格式**：MDX（Markdown + JSX），单语种页面通过 `.en.mdx` 后缀区分；KaTeX 渲染数学公式（next.config.js 中 `latex: true`）。
- **多语言机制**：next.config.js 配置 14 个 locale（en/zh/jp/pt/tr/es/it/fr/kr/ca/fi/ru/de/ar），`middleware.js` 一行 `export { locales as middleware } from 'nextra/locales'` 实现 locale 路由；阿拉伯语因 RTL 排版单独放在 `ar-pages/` 目录而非 locale 文件后缀。
- **自定义组件**（components/）：CopyPageDropdown（把当前页导出为 Markdown 复制给 LLM 使用，配合 pre.tsx 定制代码块，是 AI 时代的典型功能）、CourseCard/AnnouncementBar（课程与广告导流）、Screenshot、counters 等。
- **辅助库**：FontAwesome 图标、@svgr/webpack（svg 组件化）、@vercel/analytics（站点统计）、@napi-rs/simple-git（Nextra 显示页面更新时间用）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| pages/ | 文档站全部 MDX 内容，984 个 .mdx 文件覆盖 14 种语言（en 版 134 个），约 6.2 万行 / 4.3MB 文本 |
| ar-pages/ | 阿拉伯语专用镜像目录（115 个 mdx，RTL 布局） |
| guides/ | 早期纯 Markdown 版指南（9 个 md，约 1,934 行），现由网站版取代，仍保留 |
| notebooks/ | 12 个配套 Jupyter Notebook（OpenAI/LangChain 实操） |
| lecture/ | 提示工程 1 小时视频课讲稿 PDF（3.1MB） |
| img/ | 教程插图，68MB，是仓库体积的主要来源 |
| components/ | React 自定义组件（约 20 个 tsx/jsx） |
| .github/ | FUNDING.yml + claude.yml / claude-code-review.yml 两个 AI 辅助工作流 |
| 根 | package.json、next.config.js、theme.config.tsx、tsconfig.json、CLAUDE.md、CITATION.cff |

组织方式评述：单仓单站、以内容为中心。没有代码架构可谈，工程复杂度全部集中在 Nextra 约定（_meta.json 导航 + locale 后缀）与 theme.config.tsx 站点配置上。翻译采用"同一目录内语言后缀文件"策略（techniques/cot.en.mdx 与 cot.zh.mdx 并列），只有 ar 因 RTL 特殊处理拆出独立目录。68MB 的 img/ 直接入库而非 CDN 托管，是唯一的仓库膨胀点。

## 5. 核心架构与关键实现

内容型仓库无传统"核心模块"，取页面路由/多语言/站点配置三个关键机制分析：

**（1）多语言内容组织**。每个内容页以语言后缀并列存在，如 `pages/techniques/` 下 cot 有 .en/.zh/.jp/.pt/.tr/.es/.it/.fr/.kr/.ca/.fi/.ru/.de/.ar 共 13 个翻译版本 + 1 个主文件；导航由每目录的 `_meta.<locale>.json` 提供（pages/_meta.en.json 定义 15 个顶层板块与 Courses 下拉菜单）。英文是唯一"主版本"：新内容（如 techniques/meta-prompting.en.mdx、整个 pages/agents/ 7 篇）先出英文，翻译滞后跟进——agents 板块目前仅有 .en.mdx。

**（2）AI 时代的读站功能**（components/CopyPageDropdown.tsx）。`getPagePath()` 把 Next 路由路径反推回 `.en.mdx` 源文件路径，提供"复制页面为 Markdown"入口，方便用户把教程粘贴给 ChatGPT/Claude；theme.config.tsx 的 `main` 包装器控制该按钮只在英文非首页显示。这是 2025 年后文档站适应"LLM 读者"的代表性设计。

**（3）站点配置即产品**（theme.config.tsx）。配置中可见完整商业化链路：导航栏紫色 "✨ Services" 按钮链接 `/services`，页脚 SerpAPI 赞助 logo 与 gitTimestamp 并排展示，`chat` 指向 Discord，导航 Courses 菜单（pages/_meta.en.json）挂 6 门 DAIR.AI Academy 付费课。开源内容 + 付费课程的漏斗架构全部由这一个配置文件承载。

**（4）CI 即 AI 协作者**（.github/workflows/claude.yml、claude-code-review.yml）。两个工作流分别是"@claude 评论触发 Claude Code 处理 issue/PR"与"PR 打开时自动 Claude 审查"（anthropics/claude-code-action@v1），说明维护者用 AI agent 处理 290+ open issues 的翻译 PR 流。根目录 CLAUDE.md 则是给 agent 的项目守则（notes 放 Obsidian vault、禁止主动 push）。

## 6. 构建与运行

来自 README.md "Running the guide locally" 一节与 package.json scripts：

```bash
# 要求 Node >= 18，安装依赖
pnpm i next react react-dom nextra nextra-theme-docs
# 本地开发（http://localhost:3000/）
pnpm dev
# 生产构建 / 启动
pnpm build && pnpm start
```

无测试命令、无 lint 配置（package.json 仅 dev/build/start 三个脚本）。仓库同时存在 package-lock.json 与 pnpm-lock.yaml 两份锁文件。

## 7. 内容规模与更新机制（内容型特例）

- **内容规模**：pages/ 共 984 个 MDX、约 61,972 行、4.34MB 文本；英文主版本 134 个页面；guides/ 旧版 9 篇约 1,934 行；12 个 Notebook；论文页（pages/papers.en.mdx，451 行）收录 421 个 arXiv 链接，工具页收录 64 个工具链接。
- **技术条目体系**：Techniques 板块 19 篇（zeroshot/fewshot/cot/consistency/knowledge/prompt_chaining/tot/rag/art/ape/activeprompt/dsp/pal/react/multimodalcot/graph/reflexion/meta-prompting，目录列表见 pages/techniques/），基本覆盖 2022-2024 所有高被引提示技术论文，每篇附论文链接、prompt 示例与输出。
- **翻译流程**：社区 PR 翻译驱动（README 结尾"Feel free to open a PR"；README 宣称 13 种语言，next.config.js 实配 14 个 locale，README 略滞后）。英文先行、翻译跟进的不对称明显：agents 与 meta-prompting 等新内容尚无译文。
- **更新机制**：无 Release、无版本号，内容直接滚动进 main；最近一次推送 2026-03-11。CI 无构建校验（仅两个 Claude Code 工作流），翻译质量依赖人工 review。
- **商业化**：内容 MIT 免费 + DAIR.AI Academy 付费课程（页面内嵌课程卡片与折扣码 PROMPTING20）+ 企业培训/咨询服务（pages/services.*.mdx）+ SerpAPI 赞助。

## 8. 社区与维护状态

- **活跃度**：创建于 2022-12-16，3 年多持续更新，最近推送 2026-03-11（gh api）。2019 篇结构化内容 + 高频翻译 PR，open issues 293（多为翻译请求与内容勘误，未确认具体分布）。
- **贡献者**：约 211 人（contributors API 分页推算），其中大量为单次翻译贡献者——14 语言版本天然扩大了贡献者基盘。核心作者是 Elvis Saravia（omarsar，FUNDING.yml），DAIR.AI 团队主导。
- **生态位**：姊妹项目包括 dair-ai/ML-YouTube-Courses、dair-ai/ml-visuals、dair-ai/ML-Interviews 等（DAIR.AI 教育矩阵，未在本仓库内确认）；官方 Discord、Newsletter（nlpnews.substack.com）、YouTube 构成分发渠道（README.md）。
- **荣誉**：HN #1（2023-02-21）、WSJ/Forbes 报道、300 万学习者（README.md Announcements，数据为官方自述）。

## 9. 亮点与风险

**亮点**

1. 提示工程领域最早、最系统的免费教程，内容以论文为锚（CoT/ToT/ReAct 等均给出 arXiv 出处与可复现示例），权威性使其成为课程与论文的标准引用对象。
2. 14 种语言翻译 + 每语言独立导航元数据，触达非英语学习者的广度在同类仓库中罕见，是多语言内容仓库的范本。
3. 学习闭环完整：教程 + 12 个可运行 Notebook + 视频课讲稿 + Prompt Hub 现成模板，从读到练无缝衔接。
4. 工程上极度克制（Nextra 模板 + 约定式路由），单人即可维护百万级读者的站点，是"内容仓库最小工程 footprint"的典范；CopyPageDropdown 等细节显示对 LLM 时代阅读习惯的敏锐。
5. 用 Claude Code 工作流管理翻译 PR 与审查，是大型内容仓 AI 辅助维护的先行实践。

**风险/局限**

1. 内容时效性：Models 板块仍以 GPT-4、Llama 3、Mixtral、Phi-2 为主（README.md Guides 列表），2026 年的前沿模型覆盖明显滞后；2026-03 后更新停滞（快照日 2026-10 距最后推送已 7 个月）。
2. 翻译不同步：英文新内容（agents、meta-prompting）长期无译文，各语言版本深度参差，阿拉伯语被迫走独立目录的特例也增加了维护成本。
3. 无构建/链接校验 CI：421 个论文链接等外链随时间腐化无人检测；68MB 图片直接入库拖慢 clone 与 tarball 下载。
4. 商业导流密度上升（课程卡片、折扣码、Services 按钮、赞助 logo 同屏），社区观感上免费内容与付费漏斗的边界可能逐渐模糊。

## 10. 总结评价

这是"内容即产品"的开源标杆：一家小团队凭借一份以论文为锚、社区翻译驱动的高质量教程，在 LLM 浪潮起点卡位成功，把 promptingguide.ai 做成了提示工程领域的默认入口，79k 星是内容权威性、多语言覆盖与先发优势叠加的结果，而非代码价值。代码部分（Nextra 站点）本身没有阅读价值，值得看的是它的内容组织方式（locale 后缀 + _meta.json 的多语言约定）、AI 时代的文档设计（CopyPageDropdown）与"免费内容 + 付费课程"的运营架构。适合三类人：学提示工程的所有初学者（直接读站）、想建多语言文档站的人（抄它的 Nextra 组织方式）、运营开源教育内容的团队（学它的漏斗设计与 AI 辅助维护）。
