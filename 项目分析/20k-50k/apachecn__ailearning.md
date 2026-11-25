# apachecn/ailearning 项目分析

> 星数 36500（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-13

## 1. 项目速览

一句话定位：ApacheCN 出品的中文 AI 入门一站式教程仓库，以《机器学习实战》读书笔记为核心，聚合数据分析、机器学习、深度学习、NLP、线性代数等板块的笔记/译文与配套示例代码，并提供 GitBook 在线阅读站点。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | apachecn/AiLearning | GitHub |
| 星数 | 36500（快照）；2026-11 实查 API 为 42630，仍在自然增长 | 任务参数 / `gh api repos/apachecn/ailearning` |
| Fork | 11486 | 同上 |
| 主语言 | Python（内容以 Markdown 为主，代码为示例性质） | GitHub / 本仓统计 |
| 许可证 | CC BY-NC-SA 4.0（GitHub 标记 NOASSERTION） | `LICENSE` |
| 创建时间 | 2017-02-25 | `gh api` created_at |
| 最近 push | 2024-11-12（tarball 内全部文件 mtime 一致为 2024-11-13 00:21，即 HEAD 提交时间） | `gh api` pushed_at / tarball mtime |
| 在线阅读 | https://ailearning.apachecn.org（v1 旧版 https://alv1.apachecn.org） | `README.md` |
| 规模 | 299 个 .md（约 8.99 万行）、138 个 .py（约 2.5 万行），仓库约 53MB（大头为图片资源 `docs/ml/img` 6MB） | 本地统计 |
| 维护状态 | 官方"封板"（见第 9 章），内容冻结约在 2020-2024 年 | `docs/old_README.md` |

## 2. 项目是什么，解决什么问题

AiLearning 是国内开源翻译/知识分享社区 ApacheCN 的旗舰教程仓库，2017 年 2 月创建（`gh api` created_at）。它解决的痛点在 `README.md` 里写得很直白：中文环境里缺少一本"以程序员 Coding 角度"讲机器学习的入门材料——吴恩达的课程数学推导多、国内培训课贵且干货少，而《机器学习实战》（Machine Learning in Action）恰好是"理论+推导=加减乘除+循环"式的代码入门书，但需要有人把它啃透、补全注释、调通代码并翻译成中文。

于是这个仓库以《机器学习实战》逐章读书笔记（`docs/ml/1.md` ~ `docs/ml/15.md`，共 16 章）为骨架，每个算法章节配"概述 → 场景 → 原理 → 逐步实现代码 → sklearn demo"的结构（例见 `docs/ml/2.md` KNN 一章，567 行、32 个代码块），并在 `src/py2.x` 与 `src/py3.x` 下提供逐章可运行的示例代码。随后不断向外扩张：数据分析（Python/Numpy/Pandas/Matplotlib/Scipy）、MIT 18.06 线性代数笔记、深度学习手写实现、PyTorch（莫烦教程）、TensorFlow 2.x（《Sklearn 与 TensorFlow 机器学习实用指南》第 2 版节选）、NLP（NLTK 第二版译文 + 百度公开课笔记），形成"入门路线图"式的一站式仓库。

需要强调的是：这不是一个软件项目，而是"内容型仓库"——主体是 Markdown 笔记与译文（docs/ 约 9 万行），Python 代码是教材配套示例（src/ 约 2.5 万行），站点用 GitBook 渲染、ApacheCN 域名对外发布，另有 B 站配套教学/讨论视频（`README.md` 视频章节）。

内容的聚合性质非常明确：`docs/da/README.md` 注明数据分析笔记来自李金（lijin-THU/notes-python），`docs/linalg/README.md` 注明 MIT 18.06 笔记来自子实（zlotus），`docs/nlp/README.md` 是 Python 文档协作翻译小组的 NLTK 第二版译文，`docs/tf2/README.md` 明言"节选自《Sklearn 与 TensorFlow 机器学习实用指南第二版》"（apachecn/hands-on-ml-2e-zh）。ApacheCN 的角色是把这些分散的中文优质内容收拢、补课、统一站点化发布。

## 3. 技术栈与依赖

- **内容主体**：Markdown（299 个 .md，约 8.99 万行）+ 图片资源（`docs/ml/img/` 6MB 等）。
- **站点构建**：GitBook 3.2.3（`old/book.json` 配置了 theme-apachecn 自定义主题、search-pro、copy-code-button 等约 30 个插件；`docs/tf2/README.md` 内含 `gitbook fetch 3.2.3` 编译说明）；当前站点以 httpd:2.4 镜像静态托管（`Dockerfile`：`FROM httpd:2.4; COPY ./ /usr/local/apache2/htdocs/`）。
- **示例代码**：纯 Python。依赖极少，老版依赖清单在 `old/requirements.txt`：numpy、pandas、sklearn、keras==2.3.1、tensorflow==2.0.0、keras-contrib——全部是 2019-2020 年代的版本。
- **发布渠道**：Docker（`docker pull apachecn0/ailearning`）、PyPI（`pip install apachecn-ailearning`）、NPM 多渠道分发（`docs/old_README.md` 下载章节）。
- 无测试框架、无 CI 工作流（`.github/` 下仅有两个 issue 模板），无 package.json/pyproject 等现代工程清单。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `docs/` | 站点正文与全部教程笔记（GitBook 根） |
| `docs/da/` | 数据分析 155 篇（Python 工具/基础/Numpy/Scipy/Pandas/Matplotlib/Cython/Theano，52k 行） |
| `docs/linalg/` | MIT 18.06 线性代数笔记 37 篇（35 讲，2.9k 行） |
| `docs/ml/` | 《机器学习实战》笔记 16 章 + 附录（7.4k 行），是仓库的招牌部分 |
| `docs/nlp/` + `docs/nlp_old/` | NLTK 第二版译文 16 篇（12k 行）及旧版 NLP 笔记 |
| `docs/pytorch/` | 莫烦 PyTorch 教程 28 篇（2.1k 行） |
| `docs/tf2/` | 《Sklearn 与 TF 机器学习实用指南 2e》节选 12 篇（9.2k 行） |
| `docs/misc/`、`docs/faq/`、`docs/report/` | AI 学习路线、常用函数说明、复习版、直播问答等杂项 |
| `src/py2.x/` | 《机器学习实战》Python2 原版示例（ml 16 章 40 个 .py + dl 手写神经网络 10 个 .py） |
| `src/py3.x/` | 同一套代码的 py3 移植（ml 17 章 54 个 .py + dl 10 个 .py + tensorflow2.x 实战 12 个 .py） |
| `src/tool/`、`src/config/` | 辅助脚本（libsvm 格式转换、决策树信息增益可视化等）与包配置 |
| `tutorials/` | 较新的项目式示例：推荐系统 demo、Keras 命名实体标注等 5 个 .py |
| `old/` | v1 旧站点遗留：GitBook `book.json`、`requirements.txt`、启动脚本 |
| `NAV.md` | ApacheCN 全家族站点导航（196 行、187 条链接） |

组织方式是典型的"内容单仓 + 站点根"：一个仓库同时承载 GitBook 站点源文件、示例代码和社区导航，`SUMMARY.md`（400 行）定义站点目录树。py2/py3 两套平行目录是历史包袱的活化石——同一章代码各存一份（如 `src/py2.x/ml/2.KNN/kNN.py` 与 `src/py3.x/ml/2.KNN/kNN.py` diff 约 179 行，主要差异是 `from __future__ import print_function` 与 print 写法）。

## 5. 核心架构与关键实现

**（1）《机器学习实战》笔记 + 双版本示例代码（仓库核心资产）**
`docs/ml/2.md` 是标准范式：先"一句话总结：近朱者赤近墨者黑"式的通俗概述，再给电影分类场景、数学原理，然后逐行讲解代码。对应代码 `src/py3.x/ml/2.KNN/kNN.py`（325 行）保留了原书作者署名头：

```python
'''
Created on Sep 16, 2010
Update  on 2017-05-18
Author: Peter Harrington/羊三/小瑶
GitHub: https://github.com/apachecn/AiLearning
'''
```

即原书代码（Peter Harrington, 2010）由社区成员在 2017 年补充中文 Docstring、逐行注释与多种实现对比（`classify0()` 内给出两种距离计算写法），每章另配 sklearn 对照 demo（如 `src/py3.x/ml/2.KNN/sklearn-knn-demo.py`）。SVM 一章代码量最大：`src/py3.x/ml/6.SVM/` 下 simple/complete/Non-Kernel 三个版本合计 1177 行，含完整 SMO 算法推导注释。

**（2）深度学习手写实现（`src/py3.x/dl/`）**
10 个 .py 从零实现 perceptron、linear_unit、bp、cnn、rnn、lstm 等，不依赖框架，与 `docs/misc/` 下的反向传递/CNN/RNN/LSTM 原理笔记互为配套，是"用几百行 numpy 讲清 DL"的极简路线。

**（3）GitBook 站点体系**
`SUMMARY.md` 定义目录树，`old/book.json`（v1）与 `docs/tf2/README.md` 的编译说明显示站点用 GitBook 3.2.3 + ApacheCN 自定义主题（`theme-apachecn` git 插件）构建，加 Google Analytics（GA token 在 book.json）与百度统计。现在的分发已经转向 Docker/PyPI/NPM 三渠道打包静态站点（`docs/old_README.md`）。

**（4）TensorFlow 2.x 实战（`src/py3.x/tensorflow2.x/`）**
README 宣称的 4 个实战项目对应代码：电影情感分类、汽车燃油效率回归、过拟合优化、古诗词生成（`text_PoetryModel.py`），并附带一个完整的 zh-NER-keras 中文命名实体识别项目（BiLSTM+CRF，`bilsm_crf_model.py`），是仓库中最"新"的代码块（依赖 keras 2.3.1 / TF 2.0，见 `old/requirements.txt`）。

## 6. 构建与运行

仓库无构建/测试体系，"运行"分两类：

```bash
# 示例代码（老代码，官方 README 明确 ML 部分建议 Python 2.7.x；DL/TF2 部分用 3.6.x+）
cd src/py3.x/ml/2.KNN && python kNN.py        # 或 py2.x
pip install numpy pandas scikit-learn          # DL/TF2 另需 old/requirements.txt 所列版本

# 在线站点（当前静态托管方案）
docker pull apachecn0/ailearning
docker run -tid -p <port>:80 apachecn0/ailearning   # 访问 http://localhost:{port}

# 本地构建 v1 GitBook 站点（old/ 与 docs/tf2/README.md 记载）
npm install -g gitbook-cli
gitbook fetch 3.2.3
gitbook install && gitbook build

# PyPI 方式
pip install apachecn-ailearning && apachecn-ailearning <port>
```

数据集不在仓库内，统一由 `apachecn/data` 仓库及百度云分发（`README.md` "统一数据地址"）。

## 7. 内容体系与规模统计（特例）

- **教程章节组织**：`SUMMARY.md` 定义八大板块——数据分析（da，155 篇）、线性代数（35 讲）、机器学习（16 章）、PyTorch（26 节）、自然语言处理（12 章 + 索引）、TensorFlow 2.x（10 章）、贡献者页；README 另规划了深度学习入门、NLP 场景、Graph 图计算、知识图谱等扩展篇（部分只有链接清单，如 Graph/KG 板块为纯外链推荐，`README.md` 末段）。
- **规模**：全仓 299 个 .md 约 8.99 万行、138 个 .py 约 2.5 万行；各板块行数：da 51968 / nlp 12102 / tf2 9224 / ml 7412 / linalg 2890 / pytorch 2121。ml 代码 94 个 .py（py2 40 + py3 54），dl 双份各 10 个。
- **覆盖主题**：Python 基础与科学计算全家桶 → 经典 ML 全算法（KNN/决策树/朴素贝叶斯/Logistic/SVM/集成/回归/树回归/KMeans/Apriori/FP-growth/PCA/SVD/MapReduce/推荐系统）→ 手写 DL → TF2/PyTorch → NLP 工程场景（分词/文本分类/情感分析/NER/机器翻译/问答/摘要）。覆盖广度是它区别于单书笔记仓库的最大特征。
- **内容来源三类**：① 原创读书笔记（docs/ml，社区认领章节、README 表格逐章标注负责人与 QQ 号）；② 外部优质内容收拢（da/李金、linalg/子实、pytorch/莫烦）；③ ApacheCN 自产译文（nlp/NLTK、tf2/hands-on-ml-2e-zh）。

## 8. 代码示例质量与年代（特例）

- **年代底色**：示例代码直接继承 2010 年《机器学习实战》原书（`src/py3.x/ml/2.KNN/kNN.py` 头注释），2017 年社区批量补注释移植，DL/TF2 部分停在 2019-2020（keras==2.3.1、tensorflow==2.0.0，`old/requirements.txt`）。
- **Python2/3 兼容性**：README 用表格明确"机器学习实战：仅仅只是学习，请使用 python 2.7.x 版本（3.6.x 只是修改了部分）"——py2.x 目录至今保留大量 py2 print 语句残留（grep `print [^(]` 命中 `src/py2.x/ml/12.FrequentPattemTree/fpGrowth.py`、`11.Apriori/apriori.py`、`15.BigData_MapReduce/pegasos.py` 等 6+ 文件），py3.x 为干净移植版。以今天标准，全仓代码都不能直接跑在新版 numpy/sklearn 上（如 `np.int`、旧版 API），教学价值大于运行价值。
- **质量观感**：注释密度极高（Docstring + 逐行中文注解 + 多实现对比），但无自动化测试、无 lint、无类型标注；`run_example.py` 是一个靠改 import 手动切换 demo 的极简入口。代码是"教材配套练习"而非可复用库。

## 9. 社区生态与维护状态（特例）

- **ApacheCN 生态位**：AiLearning 是 ApacheCN 翻译/知识社区矩阵中的"AI 入门门面"。`NAV.md` 一页收录 187 条本社区站点链接，覆盖编程/前端/后端大数据/工具/数据科学/人工智能/计算机科学/安全等 9 大板块（JavaTPoint、GeeksForGeeks 翻译系列 + 各技术译文集）；README 交叉推荐 ai-roadmap、RecommenderSystems、NLP-with-PyTorch、quant 系列等姊妹仓。仓库运营高度依赖 QQ 群（README 内嵌群号与逐章负责人 QQ）与 B 站视频（教学版/讨论版双课程）。
- **协作模式**：逐章认领 + 千字 2~4 元校对奖励（`docs/old_README.md` 贡献指南）。GitHub 显示 12 位贡献者，前两名 jiangzhonglian（片刻，153 commits）与 wizardforcel（飞龙，78 commits）占了 96%，是典型的"少数核心 + 社区众包"结构（`gh api contributors`）。
- **更新停滞（有明确证据）**：① `docs/old_README.md` 顶部公告"⚠⚠⚠此项目已正式封板⚠⚠⚠"，声明 AI 项目告一段落、转向 Quant 系列项目；② 最近一次 push 为 2024-11-12（gh api），此后约两年零提交；③ README 中 PyTorch 教程仍标注"-- 待更新"；④ open issues 仅 4 个（封板后基本不再响应）。星数却在封板后继续增长（快照 36500 → 2026-11 实查 42630），呈现"内容冻结、热度惯性"状态。

## 10. 总结评价（为什么长期霸榜中文 ML 教程）

AiLearning 的成功是"时点 + 结构 + 运营"三者叠加：2017 年正值国内 AI 学习潮爆发，它以《机器学习实战》这个最适合程序员的入门书为锚点，用极低门槛的中文逐行注释笔记 + 可运行示例代码 + B 站视频 + QQ 群答疑形成完整学习闭环，再横向扩成"数据分析→ML→DL→NLP→数学基础"的一站式路线图仓库，精准踩中了"看不懂吴恩达、买不起培训班"的最大公约数人群；ApacheCN 矩阵站点与 11k+ fork 的二次传播进一步放大了 star 滚雪球效应。代价同样明显：内容聚合而非原创（许可证 CC BY-NC-SA 限制了商用）、代码与依赖停留在 2010/2020 年代、官方已宣布封板不再维护。今天它更像一部"中文机器学习入门的活化石"：算法原理讲解与代码走读仍然好用，但涉及框架与工程实践的部分应转向更新的资料。适合完全零基础的中文学习者当第一本"跟着敲"的入门教材；不适合作为工程参考或源码研究对象（它没有需要阅读的"源码架构"）。对开源社区研究者而言，它是观察中文知识型仓库如何靠内容运营登上 star 榜单的经典样本。

---
*分析基于 HEAD tarball（提交时间 2024-11-13）真实文件与 GitHub API 元数据，所有路径均为仓库内相对路径。*
