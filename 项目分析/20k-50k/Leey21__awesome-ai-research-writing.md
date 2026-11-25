# Leey21/awesome-ai-research-writing 项目分析

> 星数 36206（快照 2026-10-24）｜主语言 —｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：一份面向科研人员（尤其中文 CS 学术圈）的 AI 写作 Prompt 模板集 + Agent Skills 上手指南，主打"翻译、润色、去 AI 味、审稿视角"等论文写作全流程的即用型 prompt。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | Leey21/awesome-ai-research-writing | gh api repos/Leey21/awesome-ai-research-writing |
| 星数 / Fork / Open Issues | 36206 / 2542 / 18（2026-11-21 实测） | 同上 |
| 主语言 | 无代码（language=null，纯 Markdown 内容仓） | 同上 |
| 许可证 | 无（license=null，仓库内无 LICENSE 文件） | 同上 + git/trees/HEAD 文件树 |
| 创建时间 | 2026-01-30 | 同上 created_at |
| 最后推送 | 2026-08-19（截至分析日已停滞约 3 个月） | 同上 pushed_at |
| 仓库体积 | 约 15.8 MB（其中一张图 nano-banana.png 即 12.4 MB） | gh api size 字段 + images/ 实测 |
| 提交数 / 贡献者 | 46 次提交，3 位贡献者（Leey21:43、MySunX:2、apeterswu:1） | gh api /commits、/contributors |

## 2. 项目是什么，解决什么问题

README 开篇的自我陈述是"Make AI Writing Better for Everyone"（README.md 第 1 行）。它瞄准的痛点是：学术圈里 prompt 工程正在成为"隐性资源"——顶尖研究组内部流传着成熟的润色/翻译模板，而大多数人还在反复从零调试同一个 prompt（README.md"为什么做这个项目"一节）。作者的解法很直接：把调研来的实战 prompt 整理成可复制即用的文本，让读者"不要在 prompt 调试上浪费时间，把精力留给真正的科研"。

内容来源在 README 中有明确交代：作者称调研了 MSRA、字节 Seed、上海 AI Lab 等机构的研究员，以及北大、中科大、上交的硕博生，将他们日常使用的写作技巧开源（README.md"我们做了什么"一节）。作者本人是 USTC 在读博士（gh api users/Leey21：bio "I am a PhD student at USTC"，company "WHU & USTC"），与内容面向 CS 学术会议（ICML/ICLR/NeurIPS/ACL/CVPR）的定位自洽。注意：这些"调研来源"是 README 的自述，仓库内没有可核实的调研原始记录，未确认。

仓库交付两块内容（README.md 目录）：
- **Part I 写作 Prompt 集合**：18 个场景章节（17 个 prompt 场景 + 1 个模型选择建议），覆盖中英互译（LaTeX/Word 双版本）、缩写/扩写、中英文表达润色、逻辑检查、去 AI 味、论文架构图生成、实验绘图推荐、图表标题生成、实验分析、Reviewer 视角审稿、模型选型。
- **Part II 论文写作相关的 Agent-Skills**：面向 Cursor / Claude Code 用户，讲清楚如何用 OpenSkills（`npx openskills`）安装并触发以 SKILL.md 为核心的技能包，给出 5 个精选 skills 的总览表和 15 个使用场景的示例 prompt 表。

严格说这不是经典意义的"外部工具收录清单"（awesome-list），而更像"自带干货的单文件 prompt 手册"：绝大部分价值在 README 内嵌的 19 个 prompt 文本块本身，外部引用只有 5 个 skills、1 个配色网站、1 个模型榜单等少量链接。它的走红窗口正好踩在 2026 年初 AI 科研写作需求爆发与"去 AI 味"焦虑的叠加点上：2026-01-30 创建，约 8.7 个月后星数已达 36.2k（平均约 140 星/天，按快照日 2026-10-24 与 created_at 计算），10 个月 36.2k 星属于典型的超火箭型内容仓。

## 3. 技术栈与依赖

无任何代码与工程依赖，这是本仓库最特殊的一点：

- 内容载体：单个 README.md（907 行 / 56,740 字节，实测 wc -l），全部 prompt 用四级反引号 ````markdown 围栏块承载，便于一键复制。
- 图片资产：images/ 下 3 张 PNG（example.png 756KB 为 Skills 配置示意、model-rank.png 269KB 为模型排名截图、nano-banana.png 12.4MB 为架构图 prompt 生成效果图）。
- 运行时依赖仅存在于 Part II 的使用路径上：用户若走 Agent Skills 路线需要 Node.js 20.6+ 与 Git（README.md"Skills 的配置"一节），通过 npm 包 openskills 拉取第三方 skills 仓库；纯 prompt 用法则是零依赖。
- 无 LICENSE、无 CONTRIBUTING、无 CHANGELOG、无 .github 目录（CI/issue 模板/release 全无），文件树实测仅 README.md + images/ 共 5 个对象（gh api git/trees/HEAD?recursive=1）。

## 4. 目录结构与代码组织

顶层结构一览（实测，tarball 解压结果与 gh api 文件树完全一致，无 export-ignore 缺文件问题）：

| 路径 | 职责 |
|---|---|
| README.md | 全部内容：项目介绍 + Part I 18 个 prompt 场景 + Part II skills 教程，907 行 |
| images/example.png | Part II 中 Cursor 加载/触发 skills 的配置示意截图 |
| images/model-rank.png | "模型选择"章节引用的 Creative Writing 榜单截图 |
| images/nano-banana.png | "论文架构图"prompt 用 nano banana 生成的效果展示（12.4 MB，占仓库体积约 78%） |

组织方式评述：极致的单文件组织——所有文字内容集中在 README，未拆分为多个 markdown 文件。优点是 GitHub 页面即全部产品、Ctrl+F 可全文检索、复制路径最短；代价是 907 行单页的导航负担、图片（尤其 12.4MB 单图）显著膨胀仓库体积、且未来内容继续增长后维护难度上升。目录锚点（Part I 18 项 + Part II 3 项）承担了页内导航职责（README.md"📑 目录"）。

## 5. 核心架构与关键实现

内容仓的"核心实现"即 prompt 本身的工程化设计。逐块精读 19 个围栏块后，可归纳出贯穿全文的统一 prompt 脚手架（各场景 prompt 均遵循，见 README.md Part I 各节）：

1. **# Role（双角色设定）**：几乎每个 prompt 都给模型双重身份，如"顶尖科研写作专家 + 资深会议审稿人（ICML/ICLR）"（中转英-latex）或"中文学术期刊编辑 + 顶会中文审稿人"（中转中-word），用身份抬高输出基线。
2. **# Constraints（约束分层）**：固定按"视觉排版 / 风格逻辑 / 时态规范 / 输出格式"分层，其中大量是防格式污染的硬规则——LaTeX 场景要求转义特殊字符（`95%` → `95\%`）、保留 `$` 公式；Word 场景严禁一切 Markdown 语法以保证纯文本粘贴（中转英-word）。
3. **结构化输出协议**：强制"Part 1 [LaTeX] + Part 2 [Translation]（中文直译供核对）+ Part 3 [Modification Log]"三段式输出，把"润色了什么"透明化（表达润色英文论文、缩写、扩写等节）。
4. **# Execution Protocol（自检协议）**：输出前强制自查，典型如"审稿人视角检查逻辑跳跃"、"我是否为了刷存在感而修改了原本通顺的句子？"——把"少改、慎改"写进 prompt（表达润色中文论文、逻辑检查）。
5. **修改阈值设计**：多处明确"宁缺毋滥"——原文已够好则原样输出并给肯定评价（去 AI 味两节的 Part 3 规范），避免 LLM 过度重写。

几个单点亮点：
- **去 AI 味（LaTeX 英文）** 附带约 80 个"AI 高频词"黑名单（Accentuate、Delve、Leverage、Pivotal……Vibrant，README.md 该节末尾的围栏块），可直接当替换检查表用。
- **论文架构图** 同时给出中文版与英文版 prompt，并注明"多人反馈 nano banana 用英文版效果更好"，附生成效果图（images/nano-banana.png）——来自社区反馈的迭代痕迹。
- **实验绘图推荐** 在 prompt 内预置了一个 19 类学术图表的"标准图表库"清单（分组柱状图、帕累托前沿、带置信区域折线图、小提琴图等）外加断裂轴/对数坐标/归一化的尺度补救建议，相当于把数据可视化教材压缩进一个 prompt。
- **模型选择** 是唯一非 prompt 章节：基于 arena.ai Creative Writing 榜单给出选型建议——预算充足推荐 Claude Fable 5 与 opus 4.6/4.7，性价比推荐 Gemini-3.7-flash，受限环境推荐 qwen3.8-max，明确不推荐 GPT 系（README.md"模型选择"节 + images/model-rank.png；该节最后修订于 2026-08-19，即仓库最后一次推送）。
- **Part II 的 skills 筛选**：精选表收录 5 个 skills，来自 3 个上游仓库——20-ml-paper-writing（zechenzhangAGI/AI-research-SKILLs，覆盖 NeurIPS/ICML/ICLR 写作全流程与 booktabs 表格规范）、humanizer（blader/humanizer，基于 Wikipedia "Signs of AI writing" 去除 AI 痕迹）、docx / doc-coauthoring / canvas-design（anthropics/skills 官方）。15 行使用场景表把"从零写论文、换会议格式、投稿前 checklist、Word 模板填稿、redlining 修订"等任务逐一映射到 skill + 示例 prompt + 产出物（README.md Part II 两个表格）。

## 6. 构建与运行

无构建、无测试、无运行——它是纯文档仓。实际"使用方式"来自 README 自述：

- **Prompt 用法（零门槛）**：直接复制对应围栏块到任意聊天模型对话框，把 `[在此处粘贴你的...]` 占位符换成自己的草稿即可（README.md Part I 使用说明）。Reviewer 审稿 prompt 需上传 PDF 并填写投稿目标（如"ICML 2026"）。
- **Agent Skills 用法**（README.md"Skills 的配置"）：
  ```bash
  npx openskills --version                # 验证安装（Node 20.6+）
  npx openskills install zechenzhangAGI/AI-research-SKILLs   # 科研写作 skills
  npx openskills install anthropics/skills                   # Anthropic 官方 skills
  npx openskills list                     # 确认已安装
  ```
  安装到 `./.claude/skills/` 后由 Cursor / Claude Code 自动发现；可在 Cursor Settings → Rules, Skills, Subagents 中查看，或对话中直接描述需求自然触发。

## 7. 内容规模、分类体系与链接抽检（清单质量核查）

**规模实测**（均基于下载到的 README.md）：
- 总量：907 行 / 56.7 KB 单文件；一级分部 2 个（Part I / Part II）。
- Part I：18 个分类章节（目录锚点 18 项与正文实测一致），其中 17 个为 prompt 场景 + 1 个模型选型；全文共 19 个四级围栏 prompt 块（含"去 AI 味"词表块与架构图英文版），约合 17 个核心 prompt + 2 个附录/变体。
- Part II：5 个精选 skills（3 个上游来源）+ 15 行使用场景映射表（实测表格数据行）。
- 全文引用外部链接去重后 11 条（grep 实测）。

**分类体系**：按"写作流程阶段"而非工具类型组织。Part I 全部 18 个分类章节逐一清点如下（章节名与实测 prompt 块数均来自下载的 README.md）：

| # | 章节 | 流程阶段 | 载体 | prompt 块 |
|---|---|---|---|---|
| 1 | 中转英-latex | 翻译 | LaTeX | 1 |
| 2 | 英转中-latex | 翻译（阅读辅助） | LaTeX | 1 |
| 3 | 中转英-word | 翻译 | Word/纯文本 | 1 |
| 4 | 中转中-word | 翻译/重写 | Word/纯文本 | 1 |
| 5 | 缩写 | 篇幅调整 | LaTeX | 1 |
| 6 | 扩写 | 篇幅调整 | LaTeX | 1 |
| 7 | 表达润色（英文论文） | 润色 | LaTeX | 1 |
| 8 | 表达润色（中文论文） | 润色 | Word/纯文本 | 1 |
| 9 | 逻辑检查 | 质检 | LaTeX | 1 |
| 10 | 去 AI 味（LaTeX 英文） | 质检/风格 | LaTeX | 2（含约 80 词 AI 高频词黑名单） |
| 11 | 去 AI 味（Word 中文） | 质检/风格 | Word/纯文本 | 1 |
| 12 | 论文架构图 | 图表 | 生图模型 | 2（中文版 + 英文版 nano banana） |
| 13 | 实验绘图推荐 | 图表 | 聊天模型 | 1（内嵌 19 类学术图表库清单） |
| 14 | 生成图的标题 | 图表 | 聊天模型 | 1 |
| 15 | 生成表的标题 | 图表 | 聊天模型 | 1 |
| 16 | 实验分析 | 数据分析 | LaTeX | 1 |
| 17 | 论文整体以 Reviewer 视角进行审视 | 质检/投稿 | PDF 上传 | 1 |
| 18 | 模型选择 | 选型 | 建议 + 截图 | 0（非 prompt 章节） |

覆盖面集中于"论文正文生产与包装"环节，未涉及文献综述管理、投稿系统操作、学术伦理检测等外围环节，且明显偏 CS/AI 会议语境（角色设定全部锚定 ICML/ICLR/NeurIPS/ACL/CVPR 与《计算机学报》《软件学报》）。中英双语与 LaTeX/Word 双载体各出专版，是这个清单区别于泛用 prompt 集的主要结构特征。

**链接有效率抽检（10 条，2026-11-21 实测）**：
- GitHub 引用 5 条全部存活（gh api 200）：anthropics/skills（198k 星）、blader/humanizer（71.7k 星）、numman-ali/openskills（10.8k 星）、Leey21/arxiv-translator（524 星）、zechenzhangAGI/AI-research-SKILLs。注意最后一条已改名/转移为 Orchestra-Research/AI-Research-SKILLs（14.7k 星），旧 URL 靠 GitHub 重定向仍可达，README 未同步更新。
- 站外 5 条：htmlcolorcodes.com、seed.bytedance.com、shlab.org.cn 均 200；MSRA 页面首次超时重试后 200；arena.ai 榜单返回 403（反爬拦截，页面实际存在）；cursor.com/docs/context/skills 两次连接失败（000，本网络无法验证，非 404）。
- 结论：抽检 10 条，确认死链 0 条，1 条无法验证，整体链接质量高。

## 8. 社区与维护状态（作者背景与更新停滞核实）

- **作者背景**：Leey21（显示名 LY），gh api users/Leey21 显示为 USTC 在读博士（company "WHU & USTC"，坐标上海，2022-08 加入 GitHub，401 followers，23 个公开仓库）。README 自述的调研来源（MSRA/Seed/上海 AI Lab/北大/中科大/上交）与作者身份吻合，但为单方陈述，未确认。
- **贡献者结构**：高度单核。46 次提交中 Leey21 占 43 次，MySunX 2 次（Part II skills 部分的早期共建，提交记录可见其贡献了 humanizer/docx skill 条目），apeterswu（Lijun Wu）1 次。无 issue/PR 驱动的社区协作痕迹，无 .github 配置。
- **更新时间线**（gh api /commits 全量 46 条）：2026-01-30 创建当日即密集提交；02 月补齐模型选择（02-02）与实验绘图（02-04）、AI 词表（02-11）；03 月新增"去 AI 味"双版本（03-05）、更新 Agent Skills 与 Reviewer 约束（03-24/25）；05-18 更新目录并新增 prompt；**2026-08-19 最后一次推送（修订模型选择章节），此后 3 个月零提交**。
- **星标节奏**：2026-01-30 创建至快照日 2026-10-24 约 267 天累计 36,206 星，平均约 135 星/天；gh api 显示 2026-11-21 星数仍为 36,206 附近，属内容仓爆红后的典型平台期。仓库附带 star-history.com 星标曲线图（README.md 末尾）。
- **姊妹项目**：README"News"一节导流的 Leey21/arxiv-translator（"从 LaTeX 源码翻译并编译 arXiv 论文"，创建 2026-03-23，最后推送 2026-07-21，524 星）——本身也是另一个独立项目，与本仓构成轻度互推关系。
- **Release/CI**：无 tag、无 release、无 CI（gh api 无 releases 数据，文件树无 .github）。内容更新完全靠作者手工编辑 README。

## 9. 亮点与风险

**亮点**
1. 直接可用的成品感强：17 个 prompt 全部完整可用，统一"Role/Task/Constraints/Execution Protocol"脚手架，输出协议带中文直译与修改日志，大幅降低核对成本（README.md Part I）。
2. 场景切分精准且成体系：LaTeX/Word、中文/英文四个组合各出专版 prompt，转义规则、Markdown 禁用等细节因场景而异，看得出真实使用打磨而非套话。
3. 踩中 2026 年初"去 AI 味"刚需：专设双版本去 AI 味 prompt + 约 80 词黑名单，是该仓库区别于泛用 prompt 集的记忆点。
4. Part II 卡位 Agent Skills 新生态：在多数用户还不会配置 skills 时给出了 OpenSkills 安装/验证/触发的完整路径，并代做了一轮 skills 筛选。
5. 增速极快：约 8.7 个月 36.2k 星（日均约 135 星），2.5k fork，是 2026 年中文 AI 科研写作赛道最具代表性的现象级内容仓。

**风险/局限**
1. **无许可证**：仓库不含 LICENSE 文件（license=null 实测），默认保留所有权利，他人批量转载/商用存在授权模糊。
2. **更新已停滞**：最后推送 2026-08-19，截至分析日 3 个月无提交；"模型选择"章节强依赖具体模型版本（Claude Fable 5 / opus 4.6/4.7 / Gemini-3.7-flash / qwen3.8-max），时效衰减风险最高。
3. **单点维护 + 单文件架构**：46/46 次提交集中于 3 人（实际 1 人主导），907 行单 README 无拆分，社区贡献门槛高（无 CONTRIBUTING/issue 模板/CI）。
4. **部分引用已漂移**：20-ml-paper-writing 上游仓库已更名为 Orchestra-Research/AI-Research-SKILLs，README 未更新；arena.ai 等站外资源存在反爬/不可达情况；"调研来源"为作者自述，无原始记录可查。

## 10. 总结评价

这是 2026 年"AI 科研写作"热潮中最成功的纯内容仓之一：它不做工具聚合，而是把一批经过 MSRA/Seed/上海 AI Lab 等机构一线研究者（据 README 自述）打磨的论文写作 prompt 以统一脚手架、场景化拆分的方式单文件发布，零门槛复制即用；再用一个 Part II 顺势卡位 Agent Skills 生态，完成从"prompt 手册"到"AI 写作工作流入口"的升级。10 个月 36.2k 星验证了"学术圈 prompt 不平等"这一痛点的真实与广泛，但也暴露了内容仓的通病——无许可证、单核维护、模型选型内容快速过期、2026-08 后更新停滞。它适合三类人：赶稿期的 CS 研究生直接抄模板（最高价值场景）、想给课题组搭 AI 写作规范的导师/助管当骨架、以及研究 prompt 工程的人当作一份难得的"实战 prompt 语料库"来读——19 个围栏块里的约束写法（自检协议、修改阈值、防格式污染）本身就是很好的 prompt 设计教材。对源码阅读者而言没有代码可读，读的就是这份 README 的结构设计。
