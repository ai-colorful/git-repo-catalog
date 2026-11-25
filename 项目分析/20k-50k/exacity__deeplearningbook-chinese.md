# exacity/deeplearningbook-chinese 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 TeX｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：Ian Goodfellow 等人《Deep Learning》（花书）的社区中文全译本，以 LaTeX 源码开源、可自行编译 PDF，是中文深度学习圈的事实标准译本，后经人民邮电出版社正式出版为纸质书。

| 项目 | 内容 |
| ---- | ---- |
| 名称 | exacity/deeplearningbook-chinese |
| 星数 | 39500（快照）；gh api 实测 37995，forks 9158 |
| 语言 | TeX（LaTeX） |
| 许可证 | 无 LICENSE 文件（`src/` 顶层无任何 license 文件，README 声明"可用于学习研究目的，不得用于任何商业行为"） |
| 创建时间 | 2016-12-07（gh api `created_at`） |
| 最近提交 | 2019-12-03（gh api `pushed_at`），此后停更，未归档 |
| 翻译周期 | 2016-12-08 首个 release → 2017-03-15 v0.5-beta 定稿，约 3 个月完成全书翻译与两轮校对 |
| 状态 | 已完结停更（PDF 因版权不再更新，纸质版为最终版） |

## 2. 项目是什么，解决什么问题

《Deep Learning》是深度学习领域的经典教材（俗称"花书"），原书官方免费开放 PDF 但只有英文版。2016 年底，四位在读学生（赵申剑、黎彧君、李凯、傅天凡）发起中文翻译，选择了一条在当时国内技术书圈相当少见的路线：**翻译草稿直接开源在 GitHub，边译边接受全网读者校对，最后连同纸质出版**。仓库本体就是全书的 LaTeX 源码——20 个章节目录，每章一个 `.tex` 译文文件，外加术语表、符号表、致谢与编译脚本（`src/dlbook_cn.tex`、`src/Makefile`）。

它解决的问题非常直接：让中文读者能尽早、免费、可纠错地读到花书。README 明确说"深度学习发展太快……我们希望大家更早地学到相关的知识"，并把自己类比为原书作者"先阅读后付费"的开放策略（`src/README.md` 出版及开源原因一节）。翻译质量策略上，它没有走"机翻+人工润色"捷径，而是"直译初版 → 公开纠错 → 粗校 → 精校"的迭代发布，每个阶段对应一个 release（v0.1-alpha 直译版 → v0.5-beta 精校版，gh api releases 列表）。

最终成果有三条分发渠道：GitHub Release 上的编译 PDF（v0.5-beta，约 570 页）；GitHub Pages 在线网页版（Jekyll 站点 `src/docs/`，逐章 Markdown，便于搜索和引用）；以及人民邮电出版社的正式纸质书。电子开源与纸质出版并行而非对立——README 明确希望读者看完电子版后支持纸质正版，四位译者还把约 2 万元稿费以全体贡献者名义捐给了杉树公益（`src/donation.pdf` 为捐赠凭证，`src/acknowledgments_github.md` 记录了逐日贡献者名单）。

需要注意的版权边界：仓库没有开源许可证；因版权原因原书图片和 bib 文献库不上传（README"注意"一节、`.gitignore` 排除 `*/figures` 与 `*.pdf`），编译出的 PDF 带红色页眉"仅供学习使用，不得用于商业目的"及 DRAFT 水印（`src/dlbook_cn.tex` 第 92-103 行）。翻译本身获得了原书作者协助——`src/acknowledgments.tex` 开篇写明"原作者和我们分享了书中的原图和参考文献库"——且最终由人邮社正式出版，属于走得通的开源翻译先例，而非盗译。

## 3. 技术栈与依赖

- **LaTeX/XeLaTeX**：文档类为 `ctexbook`（中文排版），引擎 XeLaTeX，中文配置 Adobe Song Std / Adobe Heiti Std / Adobe Fangsong 三套字体（`src/dlbook_cn.tex` 第 2-8 行）。
- **编译辅助工具链**：bibtex（文献）、texindy + makeidx/imakeidx（索引）、makeglossaries + glossaries 宏包（术语表）（`src/Makefile`）。
- **版式宏包**：amsmath/amsthm（公式与定理）、booktabs/multirow/colortbl（表格）、hyperref/bookmark（PDF 书签跳转）、algorithm/algorithmic（算法环境，重命名为"算法"）、draftwatermark/background（水印与页眉声明）、natbib + breakcites（引用）、tocstyle/tocbibind（目录）（`src/dlbook_cn.tex`）。
- **Python（辅助脚本）**：`src/scripts/parse_anno.py`（从 Hypothes.is API 拉取网页版批注并整理为 annotations.txt）、`src/scripts/parse_gls.py`（展开 `\gls` 术语宏，供 Markdown 转换用）。
- **Bash + sed**：`src/scripts/convert2md.sh` 用一条 sed 管道流水线把每章 tex 批量转成 Jekyll `_posts` 文章。
- **Jekyll/GitHub Pages**：`src/docs/` 为完整 Jekyll 站点（`_config.yml`、`Gemfile`、`_layouts`、`_sass`），20 章各对应一篇 `docs/_posts/2016-12-XX-Chapter*.md`。
- 无 package.json/pyproject 等依赖清单；全部依赖为 TeX Live 发行版自带工具，构建零第三方包管理。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| ---- | ---- |
| `Chapter1/` … `Chapter20/` | 每章一个目录：主译文 `.tex`、`annotations.txt`（Hypothes.is 校对批注导出，17/20 章有）、`README.md`（网页版笔记链接） |
| `dlbook_cn.tex` | 主文件：文档类、字体、宏包、版面（7in×9in）、水印、全书组装 |
| `applied_math_and_machine_learning_basics.tex` / `deep_networks_modern_practices.tex` / `deep_learning_research.tex` | 原书三个部分的分部引言文件，按原书结构 input 各章（第一部分 Ch2-5、第二部分 Ch6-12、第三部分 Ch13-20） |
| `terminology.tex` | 886 条术语中英对照表（glossaries 条目），是全书术语一致性的单一事实来源 |
| `math_symbol.tex` / `notation.tex` | 258 个数学符号宏定义 + 原书符号说明页译文 |
| `acknowledgments.tex` / `acknowledgments_github.md` | 书面致谢（真名）与 GitHub 网友逐日致谢流水 |
| `Makefile` | 两行：`all` 完整编译链，`clean` 清理中间产物 |
| `scripts/` | parse_anno.py、parse_gls.py、convert2md.sh 三个辅助脚本 |
| `docs/` | Jekyll 网站源码（GitHub Pages 在线阅读版） |
| `donation.pdf` | 稿费捐赠凭证（杉树公益，4 名贵州高中生 3 年生活费） |

组织方式评述：这是典型的"原书结构镜像"组织——目录划分、分部引言、符号表、致谢、术语表全部 1:1 对应原书 LaTeX 工程（`src/dlbook_cn.tex` 中的注释如 `% From en book -B` 表明直接参考了原书源码骨架）。单文件单章、`\input` 汇聚，任何人认领一章即可并行翻译，是内容型翻译项目最朴素的 monorepo 形态。

## 5. 翻译组织与排版工程的关键设计

**术语系统是全书翻译的核心工程**（`src/terminology.tex`，886 条）。每个术语定义为 glossaries 条目，正文中用 `\gls{ML}`、`\gls{cost_function}` 等宏引用，编译时首次出现自动展开为"机器学习（machine learning）"格式（自定义 `\firstgls` 命令，`src/dlbook_cn.tex` 第 78-81 行）。这样改动一处术语全书生效，解决了多人分工翻译最大的术语漂移问题。网页版转换前也先用 `parse_gls.py` 展开这些宏。

**短行分行排版**：全书 tex 源码保持一句一行的写法（如 `src/Chapter5/machine_learning_basics.tex` 通篇每句独立成行），与原书 LaTeX 源码风格一致，这让 git diff 粒度落在句子级，校对 PR 与 issue 定位极其精确，也是 408 条/章级别的 Hypothes.is 批注能高效合并进正文的基础。

**译者署名直接写进章节文件头**：每章 tex 第二行有 `% Translator: Yujun Li` 等注释（grep 全部 20 章：Shenjian Zhao 6+1 章、Yujun Li 6 章、Kai Li 3 章、Tianfan Fu 4+1 章），与 README 的分工表互相印证，第 12 章两人合译并在注释中标明节次界线（`% Translator: Tianfan Fu: 12.1~12.3 Shenjian Zhao: 12.4~12.5`）。

**编译产物内嵌合规声明**：`src/dlbook_cn.tex` 第 92-103 行用 draftwatermark 加 DRAFT 水印、background 宏包在每页页眉固定输出红色"仅供学习使用，不得用于商业目的。https://github.com/exacity/deeplearningbook-chinese"，把使用边界写进了每一个 PDF 页面，而非只写在 README。

**网页版自动化**：`src/scripts/convert2md.sh` 对 20 章循环执行"python 展开术语 → sed 连环替换（章节标题转 Jekyll front-matter、`\cite` 转 `{cite?}` 占位、剔除 label）"生成 `_posts`。转换产物不手工维护（README 声明"原则是不直接修改 md 文件"），保证 LaTeX 源码是唯一上游。

## 6. 构建与运行

来自 `src/Makefile`（构建一本完整 PDF 的官方命令）：

```bash
xelatex dlbook_cn.tex && bibtex dlbook_cn.aux && texindy dlbook_cn.idx \
  && makeglossaries dlbook_cn && xelatex dlbook_cn.tex && xelatex dlbook_cn.tex
```

- 依赖：TeX Live（含 ctex、glossaries、xindy）+ Adobe 中文字体三套；`make clean` 清理 aux/log/toc 等 11 类中间产物。
- 注意：仓库不含图片（`*/figures` 被 `.gitignore` 排除）与 bib 文献库（版权原因），直接编译只能得到无图版；成品 PDF 以官方 Release `dlbook_cn_v0.5-beta.pdf` 为准。
- 网页版：`docs/` 为标准 Jekyll 站点，`jekyll serve` 本地预览，线上即 GitHub Pages（`docs/_config.yml` 中 `url: https://exacity.github.io/deeplearningbook-chinese`）。
- 批注再生成：`parse_anno.py` 读取 Hypothes.is API 的 raw JSON（脚本头注释给出了按章节 URI 拉取的示例 URL）重写 annotations.txt。

## 7. 翻译流程与校对机制（测试与工程质量的内容型对应）

本项目没有测试套件，"质量保障"体现在三层的校对流水线上（`src/README.md` 致谢一节的表格 + `src/acknowledgments.tex` 逐章致谢）：

1. **负责人（4 位译者）**：按章节分工包干初翻（分工见第 5 节），且四人之间互相交叉校对一遍（`src/acknowledgments.tex`："我们四位译者再次进行了校对并且相互之间也校对了一遍"）。
2. **简单阅读校对**：不通顺、难理解处提意见，如第一章的刘畅、许丁杰等。
3. **中英对比校对**：逐段对照原文排除少翻错翻，如第二章的 @badpoem、第九章的 @zhiding。

批注的采集方式值得一提：校对者不在 GitHub 上逐行评论，而是在网页版（GitHub Pages）上用 **Hypothes.is** 公开批注，`scripts/parse_anno.py` 定期从 Hypothes.is API 拉取全部批注导出为各章 `annotations.txt`（实测 17 章共约 2018 条批注，第一章 408 条最多，每条含批注人、建议文本、原句上下文和时间戳，`src/Chapter1/annotations.txt` 首条时间 2017-01-09）。这套"网页批注 → 脚本同步 → tex 落地"的流程把非技术读者的参与门槛降到了划词评论的程度，是本项目社区贡献规模远超一般翻译仓库的关键设计。

发版节奏即质量里程碑：v0.2-alpha"整书全部章节完成"（2016-12-14，翻译仅用 5 天）→ v0.3-alpha"文字信息翻译完成"（2016-12-28）→ v0.4-alpha"完成粗略校对"（2017-01-11）→ v0.5-beta"完成精细校对及语句调整"（2017-03-15，最终版，gh api releases）。遗留 TODO 仅剩"排版"一项（`src/README.md` TODO 节），已知缺陷如实写在 README 开头："至少 90% 的内容是可读的，并且是准确的"，引用处保留 `{cite?}` 占位（图片与文献库未上传所致）。

## 8. 社区与维护状态

- **活跃区间**：2016-12-07 创建，2016-12 至 2017-03 为高强度翻译期（7 个 release 密集发布），2017-04 起转入纠错维护，最后 push 2019-12-03（gh api），此后停更至今；未归档，issue 仍有 70 个开放（gh api `open_issues_count`）。
- **贡献者结构**：核心 4 人（四位译者，exacity 团队）；GitHub contributors API 分页统计约 20 人有提交记录；另有一大批不经 git 而经 issue/批注贡献的网友——`src/acknowledgments_github.md` 按日记录了 2016-12-08 起的数十位贡献者及各自贡献点（如 @fairmiracle 连续多日反馈公式错误，@huangpingchun 贡献第五章大量建议），README 致谢名单再列约 90 人。
- **维护模式**：PDF 冻结在 v0.5-beta（README："出于版权原因，我们不再更新此初版 PDF 文件……但源码会一直更新"），后期的 issue/PR 只是修 tex 错漏，无新功能、无 CI（仓库内无 `.github/` workflows，tarball 中未发现任何 CI 配置）。
- **生态位**：纸质版《深度学习：精装版》由人民邮电出版社 2017 年出版（异步社区发售，`src/README.md` 给出链接），四人署名即四位译者；本仓库由此成为"开源草稿 → 社区校对 → 正式出版"完整闭环的中文技术翻译标杆案例。

## 9. 亮点与风险

**亮点**

1. 工程化的翻译流水线：glossaries 术语表（886 条）+ 一句一行源码 + Hypothes.is 批注同步脚本（`src/terminology.tex`、`src/scripts/parse_anno.py`），把众包校对做成了可复现的流程。
2. 三个月完成全书约 2.3 万行 tex 译文（20 章正文合计，`wc -l` 实测）并两轮校对，且以 release 分阶段交付质量承诺，节奏管理极佳。
3. 电子开源与纸质出版双赢：开源版带合规水印、不传图片和 bib、PDF 停更以纸质版为准（`src/dlbook_cn.tex`、`src/README.md`），既满足了免费阅读需求又守住了出版授权边界，稿费还以全体贡献者名义捐出（`src/donation.pdf`）。
4. 多形态分发覆盖全场景：Release PDF / GitHub Pages 在线版（可被 Hypothes.is 划词批注）/ 纸质书。
5. 术语一致性靠宏机制而非人肉约定，任何术语修订一处生效全书。

**风险/局限**

1. 无开源许可证：README 仅声明"可用于学习研究目的，不得用于任何商业行为"，法律上是"保留所有权利"的默认状态，二次分发与改编存在灰色地带。
2. 停更六年+（最后 push 2019-12-03），issue 区 70 个开放问题无人处理，仓库发现的翻译错漏只能靠自己打补丁；且 2017 年后的深度学习进展（Transformer 等）不在原书范围内。
3. 仓库无法独立编译出完整书（缺图、缺 bib），`{cite?}` 与 `图?` 占位散布译文，对想复刻编译流程的用户是硬伤。
4. 依赖 Adobe 商业中文字体与 texindy/makeglossaries 全链路，Linux 发行版默认 TeX Live 环境下编译门槛不低（未在本机验证，"未确认"实际编译成功率）。

## 10. 总结评价

这是中文技术翻译史上教科书级的项目：四位学生译者用三个月、以"开源草稿 + Hypothes.is 众包校对 + glossaries 术语工程"的组合，把 2.3 万行 LaTeX 花书译文做到了可直接付梓的质量，最终经人民邮电出版社正式出版，并将稿费捐作公益。它证明了众包翻译可以同时满足免费、可纠错、可出版三重目标，其流程设计（网页批注脚本化回收、句子级 diff、术语宏单一来源）至今仍值得任何翻译型内容仓库照抄。它适合想读花书的中文学习者（直接下 v0.5-beta PDF 或在线网页版），也适合研究开源协作与内容型仓库运营的人精读其 README、致谢文件与三个辅助脚本；但对想 fork 后自行编译完整 PDF 的用户，因缺图缺 bib 需自行补齐素材。作为"花书中文译本的事实标准"，其历史地位在可预见的将来都难以被替代。
