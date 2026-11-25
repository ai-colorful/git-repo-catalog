# jakevdp/PythonDataScienceHandbook 项目分析

> 星数 50195（快照 2026-10-24）｜主语言 Jupyter Notebook｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：O'Reilly 畅销书《Python Data Science Handbook》（Python 数据科学手册）的**全文免费开源版**，以 Jupyter Notebook 形式完整呈现 IPython、NumPy、Pandas、Matplotlib、Scikit-Learn 五大件入门（README.md）。

| 项目 | 事实 |
|---|---|
| 名称 | jakevdp/PythonDataScienceHandbook |
| 简介 | "Python Data Science Handbook: full text in Jupyter Notebooks"（GitHub API `description`） |
| 星数 / Fork | 50195 / 19157（GitHub API，2026-11-04 查询） |
| 主语言 | Jupyter Notebook（全仓 136 个 .ipynb） |
| 许可证 | 双许可：代码 MIT（LICENSE-CODE），正文文本 CC-BY-NC-ND 3.0（LICENSE-TEXT） |
| 创建时间 | 2016-08-10（GitHub API `created_at`） |
| 默认分支 | master（GitHub API） |
| 最后实质提交 | 2023-05-05 "Add source material from second edition"（API `commits`） |
| 最后推送 | 2024-06-26（API `pushed_at`，各分支 tip 均早于此，应为 Dependabot 分支重建类推送，非内容更新） |
| 主页 | https://jakevdp.github.io/PythonDataScienceHandbook （仓库内 website/ 目录即其 Pelican 源） |

## 2. 项目是什么，解决什么问题

这是 Jake VanderPlas（NumPy/Altair/scikit-learn 生态的核心开发者之一）2016 年出版的纸质书《Python Data Science Handbook》的完整电子版仓库：全书正文与全部可运行代码以 Jupyter Notebook 形式免费公开，作者在 README 中同时提供了在线阅读站、Colab 一键运行与 Binder 一键运行三种入口（README.md 顶部的两个徽章及 "How to Use this Book" 一节）。

它解决的问题是：Python 数据科学入门者长期面临"文档碎片化"——NumPy、Pandas、Matplotlib、scikit-learn 各有独立文档，但缺少一份把"数组的思维、表格数据的处理、可视化的习惯、机器学习的基本流程"串成一条线的系统性教材。本书按五大库分六章，从 IPython 的使用技巧一路讲到无监督学习，是中文技术社区公认的"数据科学四大件"标准入门读物之一。

章节分布（章号与标题定义见 tools/generate_contents.py 的 CHAPTERS，节数为实测 `ls` 计数）：

| 章 | 主题 | 节数（章首页除外） |
|---|---|---|
| 01 | IPython: Beyond Normal Python | 8 |
| 02 | NumPy | 9 |
| 03 | Pandas | 13 |
| 04 | Matplotlib | 15 |
| 05 | Machine Learning | 15 |
| 06 | Figure Code（书籍插图生成代码汇总） | 1 |

每章以 `0X.00` 为章首介绍、`0X.01` 起为正文小节，机器学习章覆盖朴素贝叶斯、线性回归、SVM、随机森林、PCA、流形学习、K-Means、高斯混合、KDE 直至图像特征（notebooks_v1/ 目录清单实测）。

内容型仓库的特征在本项目上非常典型：代码不是"软件产品"而是"教材示例"，仓库的价值几乎全部在 notebooks/ 目录的正文与可执行单元里；tools/ 与 website/ 只是围绕书籍的维护脚本与静态站生成器。README 明确说明正文以 Python 3.5 测试（README.md "Software" 一节），requirements.txt 也把依赖钉死在 2016 年的版本上，这决定了它的时效性边界（见第 7 章）。

姊妹项目与生态位：README 中指向作者的免费姊妹书《A Whirlwind Tour of Python》（jakevdp/WhirlwindTourOfPython）作为语言入门前置。在"免费数据科学教材"这一生态位中，本项目与 fast.ai、Kaggle Learn 等并列为引用率最高的资源之一，其长尾星数增长（2016 年创建至今持续有人标注 star）主要来自教材的"长销"属性而非版本迭代。

## 3. 技术栈与依赖

- **内容载体**：Jupyter Notebook（.ipynb，nbformat v4），共 136 个（全仓统计）；正文内核标注 Python 3（抽样 `notebooks/05.08-Random-Forests.ipynb` metadata.kernelspec）。
- **教学依赖库**（requirements.txt，2016 年钉版）：numpy==1.11.1、pandas==0.18.1、scipy==0.17.1、scikit-learn==0.17.1、scikit-image==0.12.3、pillow==3.4.2、matplotlib==1.5.1、seaborn==0.7.0，另含 jupyter、notebook、line_profiler、memory_profiler、numexpr、pandas-datareader、netcdf4。
- **环境管理**：environment.yml 用 conda（`python=3.5`，conda-forge 渠道，pip 安装 requirements.txt）。
- **维护工具链**：tools/ 下 4 个脚本基于 `nbformat` 批量改写笔记本（见第 5 章）。
- **建站工具链**：website/ 基于 Pelican 静态站生成器（pelicanconf.py），依赖两个 git submodule：danielfrg/pelican-ipynb 与 getpelican/pelican-plugins（.gitmodules）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| notebooks/ | 当前工作版正文：6 章 + 前言 + 图代码章共 65 个章节 notebook，另含 data/（5 个教学用 CSV）、figures/（书籍插图 PNG）、helpers_05_08.py（随机森林章交互可视化辅助模块）、Untitled.ipynb（0 单元的遗留空文件） |
| notebooks_v1/ | 第一版完整存档（69 个 .ipynb），比 notebooks/ 多出 Index.ipynb（目录页）与 04.13-Geographic-Data-With-Basemap.ipynb（因 basemap 弃用被当前版移除） |
| notebooks_v2/ | 第二版素材起步占位，仅一个 113 字节的 data.csv |
| tools/ | 4 个 nbformat 维护脚本：generate_contents.py、add_navigation.py、add_book_info.py、fix_kernelspec.py |
| website/ | Pelican 建站源（pelicanconf.py、publishconf.py、copy_notebooks.py、fabfile.py、Makefile、theme/），用于生成 GitHub Pages 在线阅读版 |
| requirements.txt / environment.yml | Python 3.5 时代的钉版环境文件 |

附：notebooks/data/ 内置的教学数据集共 5 个 CSV——births.csv（出生统计）、president_heights.csv（美国总统身高）、state-abbrevs.csv、state-areas.csv、state-population.csv（美国州人口合并示例），全部为公开小数据集，保证读者离线可复跑全部示例（目录清单实测）。
| LICENSE-CODE / LICENSE-TEXT | MIT（代码）与 CC-BY-NC-ND 3.0（正文）双许可 |

组织方式评述：典型的"内容仓库"扁平结构——没有包管理器、没有构建系统，正文按 `章.节-标题.ipynb` 命名（如 03.08-Aggregation-and-Grouping.ipynb），文件名即目录。值得注意的冗余：notebooks/ 与 notebooks_v1/ 内容 95% 重复（16MB vs 22MB），前者是"当前展示版"（去掉 Index 与 Basemap 章、经 fix_kernelspec 处理），后者是完整快照；仓库内还遗留一个 0 单元的 Untitled.ipynb（两目录均有），属作者个人工作流的痕迹。仓库没有 .github/ 目录，无 CI、无 issue 模板。

## 5. 核心架构与关键实现

作为内容型仓库，"核心实现"体现在三处：章节体系、笔记本自动化维护、在线站生成。

**(1) 章节体系与规模**（tools/generate_contents.py 中 CHAPTERS 映射为权威定义）：00 Preface、01 IPython: Beyond Normal Python（8 节）、02 NumPy（10 节）、03 Pandas（14 节）、04 Matplotlib（16 节）、05 Machine Learning（16 节）、06 Figure Code。对 notebooks_v1 全部 69 个 .ipynb 做静态统计（本次分析实测）：markdown 单元 1853 个、代码单元 1181 个，合计 3034 个单元；篇幅最大的是 03.11-Working-with-Time-Series.ipynb（114 单元）与 03.05-Hierarchical-Indexing.ipynb（108 单元）。

**(2) 笔记本自动化维护**（tools/）：四个脚本都以 `generate_contents.py` 的正则 `(\d\d)\.(\d\d)-(.*)\.ipynb` 枚举章节文件，再用 nbformat 批量注入。例如 add_navigation.py 在每章首尾插入"上一篇/目录/下一篇"导航条并附 Colab 徽章：

```python
COLAB_LINK = """
<a href="https://colab.research.google.com/github/jakevdp/
PythonDataScienceHandbook/blob/master/notebooks/{notebook_filename}">
<img align="left" src=".../colab-badge.svg" alt="Open in Colab"></a>
"""
```
（tools/add_navigation.py，节选有折行）

这套机制说明作者把笔记本当"可再生的发布产物"维护：目录页、导航、书籍信息横幅全部脚本生成，改完正文重跑脚本即可（tools/README.md）。

**(3) 在线阅读站**（website/）：`copy_notebooks.py` 把 notebooks/ 拷入 Pelican content 并改写链接，`make html` 生成静态页，`make publish-to-github` 部署到 gh-pages（website/README.md）。网站与仓库通过分支 gh-pages（tip 2018-08）和 website 分离，README 徽章同时挂 Binder 与 Colab，让读者零安装运行。

## 6. 构建与运行

本书无"构建"概念，运行方式即执行笔记本（均出自 README.md）：

```bash
# 方式一：conda 创建与书一致的环境（Python 3.5 + 2016 版依赖）
conda install --file requirements.txt
conda create -n PDSH python=3.5 --file requirements.txt

# 方式二：Colab 零安装运行
# 打开 README 徽章 https://colab.research.google.com/github/jakevdp/
#   PythonDataScienceHandbook/blob/master/notebooks/Index.ipynb

# 方式三：Binder 一键启动（README 顶部徽章）

# 本地建站（website/README.md）
git submodule update --init --recursive && cd website
python copy_notebooks.py && make html && make serve
```

工具脚本用法：在 tools/ 下执行 `python add_navigation.py` / `python add_book_info.py` / `python fix_kernelspec.py`（tools/README.md）。注意：这些脚本 import 同目录的 generate_contents.py，只能在 tools/ 目录内运行。

## 7. 内容维护与工程质量

- **更新机制**：无 PR 自动化、无 CI（无 .github/ 目录，本次实测），维护全靠作者手工提交。历史上引入过两次外部自动化：Dependabot（分支 dependabot/pip/scipy-1.10.0 与 dependabot/pip/pillow-10.0.1，仅动 requirements.txt，未合并）。
- **正文质量**：单元结构规整（章节标题-讲解-代码-图），导航与书籍横幅脚本化保证一致性；但存在 Untitled.ipynb 空文件遗留、notebooks/ 与 notebooks_v1/ 双份冗余等仓库卫生问题（实测 `ls` 对比）。
- **时效性（重要）**：master 最后一次内容提交为 2023-05-05（"Add source material from second edition"，API commits）；正文基于 Python 3.5 / pandas 0.18 / matplotlib 1.5 / seaborn 0.7 编写（README.md、requirements.txt）。在新版环境下直接"Run All"会有报错点：如 `%matplotlib inline` 风格字符串 `seaborn-whitegrid` 在 matplotlib 新版需改为 `seaborn-v0_8-whitegrid`，抽样 notebook `notebooks/05.08-Random-Forests.ipynb` 代码单元即用旧风格名；master 最后一笔内容前的 2018 年还修过一次 `pd.datetools.timedelta` 弃用（API commits 2018-11-30）。仓库未随 2022 年后 NumPy 2.0 / pandas 2.x 更新正文。
- **第二版去向**：v2、notebooks-v2 分支（tip 2023-05 "Add v2 notebooks"）与 notebooks_v2/ 占位目录表明作者确有第二版计划，但本仓库主线未跟进，第二版正文不在本仓库（基于分支与提交信息推断，仓库外状态未确认）。

## 8. 社区与维护状态

- 贡献者：15 人（GitHub API contributors，分页计数），基本是作者单人著作 + 少量勘误型 PR；fork 19157 多为读者存档而非协作开发。
- 活跃度：创建于 2016-08；2018-08 有过一波建站/Colab 集成的密集提交（API commits）；此后仅零星维护（2018-11 弃用修复、2023-05 加入第二版素材）；open issues 226 个、长期无人关闭（API），多为读者环境适配反馈（issue 内容未逐一核实，量级以 API 为准）。
- Release：无正式 release/tag（branches 列表中无 tag 产物，`branches` API 仅列 8 个分支；release 情况未单独确认，按 API 无 releases 页推断"未确认"）。
- 分支清单（branches API 实测）：master（内容主线，tip 2023-05）、gh-pages（网站产物，tip 2018-08）、website、package-updates（2017-05 尝试去掉 requirements 钉版，未合并）、v2（2021-10）、notebooks-v2（2023-05 "Add v2 notebooks"）、dependabot/pip/pillow-10.0.1（2023-10）与 dependabot/pip/scipy-1.10.0（2023-07）。 Dependabot 分支的依赖升级始终未被合并进 master，requirements.txt 仍是 2016 钉版。
- 总体判断：项目处于"完成并冻结"状态——书已出版且全文免费公开，仓库作为静态读物继续服务读者，作者精力转向第二版（另一仓库/渠道）。

## 9. 亮点与风险

**亮点**

1. O'Reilly 正版纸质书全文免费 + 双许可清晰（代码 MIT、文本 CC-BY-NC-ND），在"知识开放"上做得非常彻底（README.md、LICENSE-*.）。
2. 三种零门槛阅读/运行入口：GitHub 在线站、Colab 徽章、Binder 徽章，全部内置在每章笔记本导航条中（tools/add_navigation.py 生成）。
3. 内容体系极完整：3034 个单元覆盖 IPython→NumPy→Pandas→Matplotlib→Sklearn 全链路，章节文件名即目录，检索友好（第 5 章实测统计）。
4. 作者权威性：Jake VanderPlas 是 NumPy/Altair/scikit-learn 生态活跃开发者，书内容与库设计哲学同源，这是它能成为"事实标准入门书"的根本原因。
5. 笔记本维护脚本化（tools/），把"目录/导航/横幅"当作可重建产物，工程习惯良好。

**风险/局限**

1. 依赖与内容停留在 2016-2023：requirements.txt 钉死 numpy 1.11.1/pandas 0.18.1 等旧版，新版环境直接运行会有风格名、API 弃用等报错，需读者自行适配（requirements.txt、notebooks/05.08 代码单元实测）。
2. 仓库冻结多年（master 内容止于 2023-05，2024-06 推送仅为分支维护），226 个 open issues 无人处理（GitHub API）。
3. 仓库卫生：notebooks/ 与 notebooks_v1/ 大面积重复、Untitled.ipynb 空文件遗留、notebooks_v2/ 只有一个 data.csv 的半成品占位（实测目录对比）。
4. 无 CI 校验：笔记本是否随依赖演化仍可运行没有任何自动保障，"能跑"依赖读者降级环境或人工修（无 .github/ 实测）。

## 10. 总结评价

这是"内容开源"模式的标杆案例：一本 2016 年的 O'Reilly 纸质书以 CC/MIT 双许可全文免费公开为可执行笔记本，凭借内容的系统性、作者在 Python 数据科学生态中的权威地位和 Colab/Binder 的零门槛入口，五年多来持续累积到 5 万星，成为 NumPy/Pandas/Matplotlib/scikit-learn 入门的事实标准教材之一。它今天的角色是一本"冻结的经典旧版教材"：主线内容止于 2023 年，依赖版本停留在 2016，在最新环境下照抄代码会遇到弃用与改名问题——因此它适合（1）初学数据科学、想系统过一遍四大件的读者（建议用 Colab 或按 requirements.txt 建旧环境）；（2）想学习"如何把一本书组织成 notebook 仓库"的作者（tools/ 与 website/ 是很好的范本）；（3）研究教学类开源仓库运营方式的人。若需要新版本语法的对应教程，应另寻第二版或更新教材，而不是期待本仓库更新。
