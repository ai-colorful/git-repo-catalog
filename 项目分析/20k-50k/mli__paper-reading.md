# mli/paper-reading 项目分析

> 星数 27,000（派发快照 2026-10-24）｜gh api 实测 34,139（2026-11-21）｜主语言 无（纯 Markdown 内容仓，GitHub language=null）｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：李沐（Mu Li）"深度学习论文逐段精读"视频系列的官方文字索引仓——整个仓库就是一份用两张 Markdown 表格维护的"已播节目单 + 选读论文清单"。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 仓库 | mli/paper-reading | gh api repos/mli/paper-reading |
| 创建时间 | 2021-10-22 | gh api created_at |
| 星数 | 34,139（2026-11-21 实测；forks 2,808；open issues+PR 2） | gh api |
| 主语言 | null（无代码，README.md 一个文件） | gh api language |
| 许可证 | Apache-2.0（LICENSE 文件） | src/LICENSE |
| 最近活跃 | 最后提交 2025-03-22（"add sora video; add sora, hunyuan, movie gen paper links (#268)"），此后停更约 20 个月 | gh api commits |
| 总提交/贡献者 | 163 次提交、9 名贡献者、无 release/tag | gh api commits/contributors/releases/tags |
| 仓库体积 | 约 9 MB（README.md 243 行 + imgs/ 封面图约 58 张 jpg 共 7.8MB） | tarball 实测 |

## 2. 项目是什么，解决什么问题

这是 B 站"跟李沐学AI"论文精读系列的配套索引。作者从 2021 年 10 月起在 B 站/YouTube 上以"逐段精读"的方式讲深度学习经典与新论文，本仓库（README.md，src/README.md）承担三个职责：一是节目单——上半部"录制完成的论文"表按时间倒序列出全部 60 期视频，每期带日期、标题（多数附论文原文链接）、封面缩略图、时长和视频链接（README.md L7-L66）；二是选读清单——下半部"所有论文"按 12 个方向给出 93 篇计划内论文，用 ✅ 标记录制状态，条目按年份排序（README.md L79-L241）；三是社区入口——文末说明选文原则并邀请读者在 GitHub Discussions 里"提供建（点）议（歌）"（README.md L73）。

它解决的核心问题不是代码问题，而是"发现与导航"：论文精读视频分散在 B 站、YouTube、知乎多个平台，观众需要一个权威、可收藏、能实时显示播放量的总目录。README 用 shields.io 动态徽章把各平台的实时播放数直接渲染进表格（详见第 5 章），使这份索引本身就是"活"的。

选文原则在 README 中有明确表述（README.md L73）：10 年内深度学习里有影响力的文章（必读）或近期有意思的文章，并偏向此前直播课（d2l.ai 课程，L73 外链）中没讲过的内容。

需要指出：任务参数中将作者描述为"Amazon 资深科学家/BSLab"，此信息不出自仓库本身；仓内可客观核实的只有 gh api 显示作者 GitHub 资料为 Mu Li、company 字段为 Boson AI、粉丝 23,929，以及 README 中精读的《参数服务器》OSDI'14 论文作者即 Li Mu 本人（README.md L38 链接 osdi14-paper-li_mu.pdf）。作者履历细节属仓外信息，本报告不作展开。

## 3. 技术栈与依赖

无任何代码工程。仓库的"技术栈"全部是文档与外链协议层面的（出处均为 src/README.md）：

- Markdown + GitHub 表格语法：全部内容载体为单个 README.md（243 行）。
- shields.io dynamic JSON badge：B 站播放数徽章通过 `img.shields.io/badge/dynamic/json?...query=data.stat.view&url=...api.bilibili.com/x/web-interface/view?bvid=BVxxx` 直接查询 B 站开放 API 实时渲染播放数（如 L7、L13 等全表 60 处）；知乎视频播放数用 `zhihu.com/api/v4/zvideos/<id>` 的 play_count（L43、L46、L48 等 3 处）；YouTube 用 shields.io 的 youtube/views 徽章（47 个去重链接）。
- Semantic Scholar API：下半部清单的引用数徽章 `query=citationCount&url=...api.semanticscholar.org/graph/v1/paper/<id>`（L84 起，93 条）。README 明确解释选它的原因——"提供 API 可以自动获取，不用手动更新"（L77）。
- 图片资源：imgs/ 下 58 张 jpg 封面（约 7.8MB），另有 1 张 webp（llama3-training-infra.webp，L9 引用）。

没有任何 package.json/CI/构建配置，也不依赖 GitHub Actions——所有"动态数据"都由第三方徽章服务在外链时实时抓取。

## 4. 目录结构与代码组织

顶层目录（tarball 全量）：

| 路径 | 职责 |
| --- | --- |
| README.md | 唯一内容文件（243 行）：上半"录制完成的论文"表 60 期 + 下半"所有论文"12 分类 93 篇 |
| LICENSE | Apache-2.0 全文 |
| .gitignore | 仅 3 行：`imgs/*.png`、`.DS_Store`、`convert.sh` |
| imgs/ | 58 张视频封面 jpg（每期一张，200px 宽内联显示） |

内容组织方式评述：

- 双视图结构：上半部是"播出记录"（倒序时间线，适合追更），下半部是"课程大纲"（按知识体系分方向，适合系统学习）。同一论文在两表中交叉出现，✅ 与上半表条目一一对应。
- 分类体系（README.md L79-L241 共 12 个 `###` 小节）：CV-CNN（7 篇）、CV-Transformer（4）、生成模型（17）、目标检测（10）、对比学习（13）、视频理解（11）、多模态（11）、NLP-Transformer（6）、系统（5）、图神经网络（1）、优化算法（3）、新领域应用（6）。下半清单统计 93 篇、其中 62 条带 ✅。
- 与 LLM 时代同步：2022 年下半年起重点转向 NLP/大模型（InstructGPT L18、Anthropic LLM L15、GPT-4 L13、Llama 3.1 五集连讲 L8-L12），2025 年初补 Sora（L7）。
- 单文件即架构：没有分目录/分文件的内容拆分，更新 = 直接编辑一张表插入新行。这种"极简单文件"正是它能被 2,808 人 fork、被广泛收藏转贴的原因之一。

## 5. 核心架构与关键实现

内容仓无代码，"关键实现"是 README 里三处巧妙的外链设计（均出自 src/README.md）：

1. 实时播放数徽章。以 L7 的 Sora 一期为例：

```markdown
[![bilibili](https://img.shields.io/badge/dynamic/json?label=views&style=social&logo=bilibili
&query=data.stat.view&url=https%3A%2F%2Fapi.bilibili.com%2Fx%2Fweb-interface%2Fview
%3Fbvid%3DBV1VdcxesEAt)](https://www.bilibili.com/video/BV1VdcxesEAt)
```

shields.io 拉取 B 站 API 的 JSON、取 `data.stat.view` 字段渲染成徽章图，GitHub 渲染 README 时每次都是实时数据。这让一张静态表格同时具备"节目单"和"热度榜"两种读法。多数期数同时挂 B 站 + YouTube 双徽章，早期还有知乎徽章。

2. 引用数自动化。下半清单的引用徽章走 Semantic Scholar API（L84 起），且 L77 注明理由：API 可自动获取、免手动更新——这是作者刻意把"人工维护成本"外移到自动化服务的两处设计之一。

3. 计划-完成双表联动。"所有论文"各表第一列"已录制"用 ✅，录制完成后既更新下半表的 ✅，也在上半表新增行，并配一张新封面图入 imgs/（文件名与论文对应，如 bert.jpg、alphafold_2.jpg）。图片按期入库、`.gitignore` 忽略 `imgs/*.png` 和 `convert.sh`（src/.gitignore），暗示作者本地有 PNG→JPG 的封面转换脚本但未纳入仓库。

值得注意的失真点：README L75 自述"总论文数 67，录制完成数 32"，但下半清单实际已有 93 篇、62 处 ✅，上半表已有 60 期——这两行计数文字明显停留在 2022 年初的状态，此后只加行未更新汇总数字（源码内可直接核对）。

## 6. 构建与运行

无构建、无测试、无运行概念：内容消费方式就是打开 GitHub 仓库页阅读 README，或点击表内 B 站/YouTube 链接观看视频。对贡献者而言，"运行"仅指编辑 Markdown 表格并附 imgs/ 封面图；封面图预处理依赖作者本地未入库的 convert.sh（src/.gitignore），仓库内不存在任何可执行脚本。

## 7. 内容质量与更新机制（特例：测试与工程质量）

- 内容覆盖（上半表 60 期，README.md L7-L66）：CV 基础（AlexNet/ResNet/ViT/Swin/MAE/DETR）、对比学习综述与 CLIP 改进串讲、视频理解串讲、多模态串讲、生成模型（GAN/DALL·E 2/Sora）、NLP/LLM（Transformer/GPT 三部曲/BERT/GPT-4/InstructGPT/Whisper/Llama 3.1 拆五集）、ML 系统（参数服务器/GPipe/Megatron-LM/ZeRO/Pathways）、科研方法论（如何读论文、《研究的艺术》四讲、研究价值、novelty、大模型时代做科研四思路）以及配套工具 autocut 一期（L22）。
- 信息一致性：表格结构规整、每期必有时长与封面，但存在前述 L75 计数失真；部分早期行字段缺 YouTube 徽章（如 L27、L30、L31），非统一但无碍阅读。
- 更新机制：完全人工 + 社区辅助。选题经 GitHub Discussions 征集（L73"点歌"），最后一次提交即社区 PR #268（2025-03-22，"add sora video; add sora, hunyuan, movie gen paper links"），说明链接补录也接受 PR。无 CI/自动化校验（tarball 中无 .github/ 目录）。
- 工程痕迹：.gitignore 仅 3 行；163 次提交、9 名贡献者（gh api）；无 release/tag——对内容仓属正常。

## 8. 社区与维护状态

- 活跃时间线（gh api）：创建 2021-10-22；高频更新期 2021-10 至 2024-08（上半表几乎每周一期到 2022 年中，之后节奏放缓：2023 年 4 期、2024 年 5 集 Llama 3.1）；最后一次推送 2025-03-22（PR #268），截至分析日（2026-11-21）已停更约 20 个月，任务参数"2025-03 后停滞"经核实属实。
- 星数趋势：派发快照 27,000（2026-10-24），gh api 实测 34,139（2026-11-21）——快照与实测差异较大且方向为增长，原因未确认（可能与快照采集时间或统计口径有关，如实记录两值）。
- 贡献者结构：9 名贡献者（gh api contributors），主体为作者本人；issues+PR 合计仅 2 个 open，讨论主要在 Discussions（数量未确认）。forks 2,808——单文件索引仓的 fork 更多体现收藏/搬运行为，无证据表明存在翻译分支，标注未确认。
- 与作者其他仓库的关系：README 中仅两处仓外关联证据——d2l 直播课链接（L73，指向作者参与的 Dive into Deep Learning 课程 c.d2l.ai/zh-v2）与 autocut 剪视频工具一期（L22，对应作者名下 mli/autocut 仓库）。本仓是其内容生态中的"目录"角色。

## 9. 亮点与风险

亮点：
1. 极简而高效的组织：一个 README 双表结构同时满足"追更"与"系统学习"两种阅读路径（README.md L3-L76 / L71-L241）。
2. 动态徽章工程：B 站/YouTube/知乎播放数与 Semantic Scholar 引用数全部实时渲染，索引"永不过时"的错觉由自动化外链维持（L7-L66、L84-L241）。
3. 内容权威性：逐段精读由作者亲自录制并长期坚持 60 期，覆盖 CV/NLP/生成模型/ML 系统/科研方法五大板块，选文原则公开（L73）。
4. 社区选题机制：Discussions"点歌"让观众参与排片，PR #268 证明社区可实际参与维护。
5. 中文 AI 学习者社区的稀缺品：高质量中文论文精读几乎没有同类体系化替代。

风险/局限：
1. 已实质停更：最后推送 2025-03-22，此后约 20 个月无内容更新（gh api commits），"所有论文"清单中仍有约 31 篇未录（93-62）。
2. 数据失真：L75"总论文数 67，录制完成数 32"为陈旧汇总，与表格实际（93/62/60 期）不符，仓库缺一次"收尾性维护"。
3. 单点依赖：内容与作者个人强绑定，无团队接力；外部依赖（B 站 API、shields.io、Semantic Scholar）任一变更都会让徽章失效。
4. 知乎徽章仅早期 3 处、部分行缺 YouTube 徽章，跨平台覆盖不均匀（L43-L48 与 L27/L30/L31 对比）。

## 10. 总结评价

mli/paper-reading 是"内容型仓库"的教科书案例：没有任何代码，仅凭一张精心维护的双表 README 成为 34k 星的资产——它把个人 IP（李沐的视频系列）、知识组织（12 方向 93 篇的选读大纲）与自动化外链（实时播放数/引用数徽章）压缩进一个 9MB 的仓库，维护成本低到只需"录完一期加一行"。它的价值在停更后依然存在：对中文深度学习学习者，这是从 AlexNet 到 Sora/Llama 3.1 的一条被验证过的学习路径图；对想模仿"内容即仓库"的人，它的徽章机制、双表结构、点歌式选题都是可直接借鉴的设计。适合谁：想系统补论文功课的学生与工程师（按下半清单 + 上半表选看），以及运营技术内容系列、想用 GitHub 做分发索引的作者。不适合期待源码或可复现代码实验的读者——这里没有代码，只有通往代码与视频的路标。
