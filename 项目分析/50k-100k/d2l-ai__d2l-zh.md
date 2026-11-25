# d2l-ai/d2l-zh 项目分析

> 星数 82052（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：《动手学深度学习》（Dive into Deep Learning，D2L.ai）中文版全书开源书稿——"文本 + 数学 + 可运行代码"三位一体的深度学习交互式教材。

| 项目 | 内容 |
|---|---|
| 名称 | d2l-ai/d2l-zh（《动手学深度学习》第二版中文版） |
| 星数 | 82052（任务快照）/ 82181（GitHub API 实查 2026-10-31） |
| 主语言 | Python（正文为 markdown，配套代码为 Python） |
| 许可证 | 内容 CC-BY-SA-4.0 + 代码 MIT-0（`config.ini` 第 9 行声明）；仓库顶层 `LICENSE` 文件为 Apache-2.0（GitHub 识别值），三者并存 |
| 版本 | 2.0.0（`config.ini`、`d2l/__init__.py`） |
| 创建时间 | 2017-08-23（gh api） |
| 最近活跃 | master 最后提交 2023-08-18（gh api commits/master），仓库最后推送 2024-07-30（gh api pushed_at），此后停滞 |
| 作者 | Aston Zhang、Zachary C. Lipton、Mu Li（李沐）、Alexander J. Smola（`config.ini`） |
| 官网 | zh.d2l.ai（HTML/PDF/Colab 全套由本仓库 CI 构建发布） |

## 2. 项目是什么，解决什么问题

本项目是深度学习入门经典教材《动手学深度学习》的中文版完整书稿仓库。核心理念（`README.md`）："理解深度学习的最佳方法是学以致用"——把概念、数学原理与可运行代码放在同一处，读者可以直接修改代码、观察结果、获取经验。它解决的问题很明确：深度学习资料要么只有数学（纸上谈兵），要么只有代码（知其然不知其所以然），本书把两者缝合成一套可实践、可免费获取的统一资源。

全书共 20 个 `chapter_*` 目录、约 121 节正文，从线性回归一路讲到注意力机制与 Transformer（`chapter_attention-mechanisms/transformer.md` 确认含 Transformer 一节）、优化算法、计算机视觉和 NLP 应用。每节都是"讲解 → 带注释的代码 → 运行输出 → 练习"的循环，代码即教材的一部分，而非附属品。

本书的影响力有直接背书（`README.md`）：韩家炜、周志华、张潼等学者及黄仁勋、余凯等业界人物公开推荐；被全球多所大学用作教材（README 中附采用院校地图）；配套伯克利 STAT-157 课程（`README.md` 中英文教学资源小节）。中文版由 AWS 翻译团队完成（`config.ini` `[translation]` 段：`origin_repo = d2l-ai/d2l-en`，`translator = aws`）。

值得一提的是仓库保留了 139 个 `*_origin.md` 文件——英文原文与中文译文一一并存于同一目录（如 `chapter_preliminaries/linear-algebra.md` 与 `linear-algebra_origin.md`），这是翻译工作流的痕迹：译文基于英文原稿对照翻译、审校。

作为内容型仓库，它并非"无代码书稿"：`d2l/` 目录是一套约 9400 行的四框架配套 Python 库（详见第 5 章），CI 会把每节 markdown 当作 Jupyter notebook 在 GPU 上真实执行，输出写入最终网页（`INFO.md`："markdown 提交前清除 output，服务器上重新执行生成结果，限制 20min"）。也就是说，网页版里每个代码块的结果都是构建时真跑出来的。

## 3. 技术栈与依赖

- **正文格式**：d2lbook 扩展 Markdown。在标准 Markdown 上增加了自有语义标记（真实样例见 `chapter_linear-networks/linear-regression.md`）：`:label:` 章节锚点、`:eqlabel:` 公式编号、`:numref:`/`:cite:` 交叉引用、`:begin_tab:`/`:end_tab:` 多框架代码页签。
- **构建系统**：d2lbook（李沐自研的书稿构建工具，配置入口 `config.ini`），底层为 Sphinx + XeLaTeX；Sphinx 主题为 `mli/mx-theme`，构建脚本子模块为 `d2l-ai/utils`（`.gitmodules`）。
- **d2l 配套库**（`setup.py`）：纯 Python，仅依赖 numpy==1.21.5、matplotlib==3.5.1、pandas==1.2.4、requests==2.25.1、jupyter==1.0.0，刻意保持轻量；运行时按需导入所选框架（torch/tf/mxnet/paddle）。
- **四框架支持**：`config.ini` `tabs = mxnet, pytorch, tensorflow, paddle`，每个框架一份代码实现 + 一套 alias 映射规则。
- **CI 基础设施**：GitHub Actions + AWS Batch（GPU）+ boto3（`ci/submit-job.py`），构建产物部署至 S3（`config.ini` `[deploy]` 有 s3://d2l-webdata 地址）并发布到 zh.d2l.ai。
- **PDF 排版**：XeLaTeX + ctex + 思源宋体/黑体 + Source Serif/Sans/Code Pro（`config.ini` 中 `latex_elements` 内联了完整的中文排版 preamble；字体安装步骤见 `INFO.md`）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `chapter_introduction/` ~ `chapter_appendix-tools-for-deep-learning/` | 全书 20 章书稿正文（markdown），每章一目录，`index.md` 为章首页 |
| `chapter_*/*_origin.md` | 139 个英文原文对照文件（翻译工作流产物） |
| `d2l/` | 四框架配套 Python 库：`torch.py`（2666 行）、`paddle.py`（2680 行）、`mxnet.py`（2551 行）、`tensorflow.py`（1477 行）、`__init__.py`（多入口说明） |
| `config.ini` | d2lbook 构建主配置：书名、框架 tabs、Sphinx/LaTeX 样式、Colab/SageMaker/课件配套仓库映射 |
| `img/` | 247 个插图源文件（171 svg 矢量图 + 52 png + 1 pdf） |
| `graffle/` | OmniGraffle 绘图源文件（书中插图的"源码"） |
| `static/` | 品牌 logo/favicon、LaTeX 后处理脚本 `post_latex/main.py`、构建缓存脚本 |
| `ci/` | AWS Batch 任务提交器 `submit-job.py` 与 Docker 构建定义 |
| `.github/` | Actions 工作流（ci.yml 等）与 8 个 workflow_scripts 构建脚本 |
| `d2l.bib` | 全书参考文献库（61KB） |
| `STYLE_GUIDE.md` / `TERMINOLOGY.md` / `INFO.md` | 行文样式规范、中英术语对照表、构建说明 |
| `contrib/` | 社区贡献章节草稿（如推荐系统章节草稿 `contrib/chapter_recommender-systems/`） |

组织方式评述：这是典型的"书稿单体仓"——内容（章节）与代码（d2l 库）同仓共演进，保证书里 `from d2l import torch as d2l` 调用的每个函数都能在仓库内找到实现；多语言版本不在此仓（英文主仓为 d2l-ai/d2l-en，中文版以 d2l-en 为上游翻译而来）；Colab/SageMaker/课件各自拆分为独立仓库，通过 `config.ini` 的 `[colab]`、`[sagemaker]`、`[slides]` 段声明映射关系。

## 5. 核心架构与关键实现

### 5.1 四框架统一的 d2l 配套库

`d2l/` 下四个框架文件是同一套函数 API 的四份平行实现（torch 107 个函数、mxnet 108、paddle 108、tf 60），涵盖绘图（`plot`/`set_figsize`）、数据加载（Fashion-MNIST、Time Machine 语料、机器翻译数据）、训练循环、模型组件（`Residual`、`RNNModelScratch`、`EncoderDecoder` 等）。最有辨识度的设计是**函数名带章节号**——`train_ch3`（softmax 回归训练循环）、`train_ch6`（GPU 训练）、`train_ch8`（RNN 训练），函数与教材章节一一对应，读者顺着书读就能顺着代码找到实现（`d2l/torch.py:261,473,774`）。

```python
# d2l/torch.py:370-392（节选）——数据集托管：内置下载器 + sha1 校验
def download(name, cache_dir=os.path.join('..', 'data')):
    """下载一个DATA_HUB中的文件，返回本地文件名"""
    assert name in DATA_HUB, f"{name} 不存在于 {DATA_HUB}"
    url, sha1_hash = DATA_HUB[name]
    ...
    if os.path.exists(fname):
        sha1 = hashlib.sha1()
        ...
        if sha1.hexdigest() == sha1_hash:
            return fname  # 命中缓存
```

所有数据集通过 `DATA_HUB` 字典注册（URL + sha1，源站 `d2l-data.s3-accelerate.amazonaws.com`，`d2l/torch.py:367-368`），保证教学数据可复现且不被上游静默变更。此外每个函数的 docstring 还带 `Defined in :numref:`sec_xxx`` 反向链接到书中的定义小节，代码与正文双向索引。

### 5.2 d2lbook 的"一份书稿、四个框架"机制

这是全仓库最核心的工程设计。作者只需用 `d2l.*` 这个抽象层写一遍代码，构建时 d2lbook 依据 `config.ini` 中每个 `[library-*]` 段的别名规则，把书稿中的 d2l 调用和 NumPy 风格 API 自动转写为目标框架的等价代码（`config.ini:126-207`）：

```ini
# config.ini [library-pytorch]（节选）
simple_alias = ones, zeros, tensor, arange, ..., concat -> cat, stack, abs, eye
fluent_alias = numpy -> detach().numpy, size -> numel, reshape, to,
               reduce_sum -> sum, argmax, astype -> type, transpose -> t
alias = nn_Module = nn.Module
```

`simple_alias` 处理直接改名（`concat -> cat`），`fluent_alias` 处理方法链式调用（`numpy -> detach().numpy`），正则 `reverse_alias` 处理反向替换。正文中仍存在实现差异的地方则用 `:begin_tab:` 页签手工分支（`chapter_linear-networks/linear-regression.md:510-518` 实测含 mxnet/pytorch/tensorflow 三个页签）。这套机制让四框架版本的成本被压到"一份手写 + 三份自动转写 + 少量手工页签"。

### 5.3 "书即代码"的重型 CI

`.github/workflows/ci.yml`（143 行）定义了五阶段流水线：四个 GPU job（build_torch/tf/mxnet/paddle）各自把对应框架的书稿 notebook 提交到 AWS Batch 真实执行，全部通过后 build_and_deploy 生成 HTML + 四个 PDF 并发布，最后自动在 PR 上评论 preview.d2l.ai 预览链接（`ci.yml:135-143`）。任务派发器 `ci/submit-job.py` 用 boto3 向 `D2L-CI-CPU`/GPU 队列提交预定义的 job definition，支持取消时终止远端 Batch 任务（`ci.yml:38-42`）。用 AWS Batch 而非 GitHub 托管 runner，是因为全书 notebook 全框架执行远超 Actions 的时限与算力配额——这是"书即代码"模式的必要代价，也是大多数开源书项目做不到的工程投入。PDF 构建还有专门的 LaTeX 后处理（`static/post_latex/main.py`，如把"小结/练习"小节从编号体系中摘除）。

## 6. 构建与运行

以下命令均来自 `INFO.md` 与 `.github/workflow_scripts/`（本分析未实际运行）：

```bash
# 安装 d2l 配套库（读者使用）
pip install d2l                    # 或 pip install git+https://github.com/d2l-ai/d2l-zh@release
# 书内用法：from d2l import torch as d2l

# 本地构建 HTML（需 GPU；build/ 为子模块 mx-theme + utils，需先 git submodule update）
conda env update -f build/env.yml
source activate d2l-zh-build
make html                          # 全量执行 notebook
make html EVAL=0                   # 跳过 notebook 执行（无输出结果）

# 构建 PDF（需 texlive-full、librsvg2-bin、思源字体，安装步骤见 INFO.md）
make pdf

# CI 侧按框架构建（.github/workflow_scripts/build_pytorch.sh 等）
./.github/workflow_scripts/build_pytorch.sh <repo> <branch> <cache_dir>
```

阅读门槛最低的方式是直接访问 zh.d2l.ai，或用配套 Colab 仓库（`d2l-ai/d2l-zh-pytorch-colab` 等，见 `config.ini` `[colab]` 段）零配置运行。

## 7. 内容质量与工程化 CI（测试与工程质量）

本项目没有传统意义的单元测试目录，"测试"体现为两层机制：

- **内容层的"测试即运行"**：CI 把全书 markdown 当 notebook 逐节执行（`INFO.md` 明确要求提交前清除 output、单节执行不超过 20 分钟），任何代码错误、依赖变更导致的运行失败都会直接挡住合并。这相当于对全书代码做了全量集成测试，且输出即文档。
- **翻译质量控制**：`STYLE_GUIDE.md`（中文行文细节，如"时刻t"、中英标点规则）与 `TERMINOLOGY.md`（术语对照表）+ 139 个 `_origin.md` 原文对照，构成翻译审校流程。
- **工程观察**：CI 配置成熟度高（幂等的取消清理、按 commit message 的 `[skip torch]` 等跳过开关、PR 预览评论）；但 `config.ini` 中 Colab/SageMaker 仍固定 `mxnet-cu101==1.7.0`，而 Apache MXNet 已于 2023 年退役，四个框架中 tf 版函数量也明显偏少（60 vs 107），多框架维护成本已现疲态。仓库无 linter/formatter 配置（`.github/` 内未见），代码风格靠 `STYLE_GUIDE.md` 约定。

## 8. 社区与维护状态

- **活跃度**：2017-08 建仓，v0.1（2017-09）到 v2.0.0（2022-12-08）共 16 个 release（gh api releases），前期几乎每月迭代。master 最后提交 2023-08-18，仓库最后推送 2024-07-30（gh api），**此后至今（2026-10）无任何新提交，v2.0.0 就是最终版本**——维护处于事实停滞状态。
- **社区规模**：fork 12469；贡献者 100+（contributors API per_page=100 已封顶，精确总数未确认）。issue 区仍有 127 个 open issues（gh api），无人处理。
- **演进去向**：新版开发重心在英文主仓 d2l-ai/d2l-en（`README.md` 明示"新版将继续用英文编写，并搬回中文版"），中文版停留在第二版快照。生态配套（Colab/SageMaker/slides 各框架独立仓库、courses.d2l.ai）均由同一团队维护。
- **持续性**：内容虽冻结，但 zh.d2l.ai 在线版、PDF、数据集 S3 源仍在线可用（`config.ini` 部署地址与 `d2l/torch.py` 数据源），存量读者不受影响。

## 9. 亮点与风险

**亮点**

1. "文本 + 数学 + 可执行代码 + 真实运行输出"四合一的教材范式，CI 强制全书代码可跑通，质量有硬保障（`INFO.md`、`.github/workflows/ci.yml`）。
2. d2lbook 一稿多框架机制：一份手写书稿经 alias 规则自动转写出 MXNet/PyTorch/TensorFlow/Paddle 四个版本，工程上极省成本（`config.ini` `[library-*]` 段）。
3. d2l 库与教材深度耦合但刻意轻量（仅 numpy/matplotlib/pandas/requests，`setup.py`），函数按章节命名、docstring 反链书中小节，学习路径顺滑。
4. 数据集托管带 sha1 校验（`d2l/torch.py` DATA_HUB），教学可复现性强。
5. 极高的社区信任：学界与业界大佬背书、全球高校采用（`README.md`），12k+ fork。

**风险/局限**

1. **维护停滞**：最后实质提交 2023-08，最后推送 2024-07，issue 无人处理；深度学习领域 2023 年后的大模型/LLM 浪潮完全未覆盖。
2. MXNet 标签已是负资产：Apache MXNet 项目已退役，书中 mxnet 页签与 CI 中的 mxnet 构建链路只剩历史价值（`config.ini`、`ci.yml`）。
3. 重型 CI 依赖 AWS Batch 专属基础设施（`ci/submit-job.py`、自定义 job definition），他人难以复用或自行部署同款构建流程。
4. 依赖版本陈旧（numpy 1.21.5 / jupyter 1.0.0，`setup.py`），在新环境安装可能遇到兼容性问题。

## 10. 总结评价

d2l-zh 是中文技术开源内容的天花板级作品：它把一本深度学习教材变成了"可运行、可校验、可持续构建"的软件工程产物，其"书即代码 + CI 强制执行 + 一稿四框架"的范式在开源书项目中几乎独此一家。8 万星背后是李沐团队的个人影响力、免费开放的内容策略与中文社区对优质 ML 教材的长期饥渴共同作用的结果。如今项目已完成历史使命——v2.0.0 定格于 2022 年底，2023 年后无实质更新，新版演进转移至英文主仓。它仍然值得推荐：对深度学习初学者，这是中文世界最好的入门教材之一；对想学习"如何用软件工程方法维护一本书/一套教程"的作者与教育者，`config.ini` + `d2l/` + `.github/` 这套组合是极有参考价值的范本；但对冲着最新技术（LLM/大模型）而来的读者，需要转向 d2l-en 或其他更新资料。
