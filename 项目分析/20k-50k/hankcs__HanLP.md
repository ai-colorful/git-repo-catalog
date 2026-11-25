# hankcs/HanLP 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Java（master 分支现为 Python，gh api 复核 36536 stars / language=Python）｜档位 20k-50k｜分析日期 2026-11-17

## 1. 项目速览

一句话定位：面向中文与多语言的自然语言处理工具包，1.x 是纯 Java 传统 ML 实现，2.x 基于 PyTorch/Transformer 全面重写为 Python 库，一个模型同时输出分词、词性、NER、依存、成分句法、语义角色、语义依存、AMR 等全部标注。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | hankcs/HanLP | gh api |
| 星数 | 36536（2026-11-17 gh api 核实；任务快照 39500） | gh api |
| 主语言 | master=Python（483 个 .py，57928 行）；Java 完整版在 1.x/portable 分支 | src/ 统计 |
| 许可证 | 代码 Apache-2.0；模型 CC BY-NC-SA 4.0（非商用） | LICENSE、README.md L185-191 |
| 版本 | 2.1.7（master）；1.x 分支最新 release v1.8.6 | hanlp/version.py、gh api releases |
| 最近活跃 | pushed 2026-11-07（活跃） | gh api |
| 分支 | master（2.x）、1.x（Java）、portable（Java 精简）、dev、doc-zh、doc-ja | gh api branches |

## 2. 项目是什么，解决什么问题

HanLP 由何晗（hankcs）于 2014 年创建（gh api created_at=2014-10-09），最初是一套"生产级"中文 NLP Java 工具包：基于词典 Trie、感知机、CRF、维特比等传统算法实现分词/词性/NER/依存句法/关键词/摘要/短语提取，追求"开箱即用 + 高速低内存"。这套实现至今完整保留在 `1.x` 分支与 `portable` 分支（1.x 分支含 pom.xml 与 `src/main/java/com/hankcs/hanlp/` 全套包结构，gh api git/trees 核实）。

2019 年底作者开始 2.x 重写（src 各文件日期注释从 2019-12-28 起密集出现），理念从"算法库"变为"深度学习模型库"：以 PyTorch 为主后端（TensorFlow 2.x 作兼容 extras，且 setup.py L38 注明 "TF is deprecated in Transformers"），用 Transformer 编码器 + 多任务学习把分词、词性、词元化、特征抽取、NER、依存、成分句法、语义角色、语义依存、AMR 十个任务合在一个模型里，依托 UD 2.10 与 OntoNotes5 支持 130 种语言（README.md L33-36）。

2.x 的显著特点是"可复现性承诺"：README L146-148 明确"我们承诺分数表中每个数字可复现，任何复现问题按最高优先级 fatal bug 处理"。配套 `docs/`（含 tok/pos/ner/dep/sdp/con/srl 各任务标注规范 annotations/）与 EMNLP 2021 论文《The Stem Cell Hypothesis》（README L162-179，作者 He & Choi，即作者在 UGA 的博士工作）。

## 3. 技术栈与依赖

- Python 3.6+（setup.py L82），PyTorch（torch>=1.6.0）、HuggingFace transformers>=4.1.1、sentencepiece（setup.py L69-80）
- 自拆分的插件包：hanlp-common（常量/反射/Document 结构）、hanlp-trie（分词双字典）、hanlp-downloader（模型下载），作为独立 PyPI 包依赖（setup.py L76-78；plugins/ 目录即其源码）
- extras：`amr`（penman/networkx/perin-parser）、`fasttext`（fasttext-wheel）、`tf`（tensorflow 2.x + tf-keras，维护旧 TF 模型兼容，setup.py L25-40）
- Java 侧：1.x 分支 Maven（pom.xml）；master 仅剩 RESTful Java 客户端插件（plugins/hanlp_restful_java，1141 行）
- 训练指标：fasttext 分类器、word2vec/glove 词向量、RNN 语言模型（hanlp/pretrained/ 目录清单）

## 4. 目录结构与代码组织

| 目录 | 职责 | 出处 |
|---|---|---|
| hanlp/ | 2.x 核心 Python 包（50816 行） | hanlp/ |
| hanlp/common/ | 组件基类：Component/TorchComponent/KerasComponent、dataset/vocab/transform 抽象 | hanlp/common/ |
| hanlp/components/ | 各任务实现：mtl（多任务）、parsers（依存+成分）、amr、srl、taggers、tokenizers、classifiers、ner、distillation、eos、sts、lm | hanlp/components/ |
| hanlp/layers/ | 网络层：transformers（编码器封装）、embeddings（含 ContextualWordEmbedding）、crf、cnn、feed_forward | hanlp/layers/ |
| hanlp/pretrained/ | 17 个模型清单模块：模型 ID → 官方 zip URL + docstring 内嵌指标 | hanlp/pretrained/ |
| hanlp/datasets/、transform/、losses/、metrics/、optimizers/、callbacks/、utils/ | 数据集加载器、特征变换、损失、指标、训练回调与工具 | hanlp/ |
| plugins/ | 自拆分子包：hanlp_common、hanlp_trie、hanlp_restful（Python REST 客户端）、hanlp_restful_java、hanlp_restful_golang、hanlp_demo（en/ja/lzh/mul 演示） | plugins/ |
| docs/ | Sphinx 文档：教程、API、各任务标注规范（annotations/tok...srl） | docs/ |
| tests/ | pytest 测试（7 文件 414 行） | tests/ |
| .github/workflows/ | unit-tests.yml（CI + PyPI 自动发布） | .github/workflows/ |

组织方式评述：master 是"单核心包 + 多 PyPI 插件"模式（hanlp 主包复用自己拆出的 hanlp-common/hanlp-trie，plugins/ 同时是插件源码与主仓依赖，耦合巧妙但略绕）；Java 1.x 与 Python 2.x 以分支而非 monorepo 目录共存，portable 分支面向 Android/无数据包场景。

## 5. 核心架构与关键实现

### 5.1 多任务学习框架（2.x 的心脏）

`hanlp/components/mtl/multi_task_learning.py`（3204 行目录核心）：`MultiTaskModel = encoder(ContextualWordEmbeddingModule) + scalar_mixes + decoders(ModuleDict)`——一个 Transformer 编码器服务所有任务解码器；`MultiTaskDataLoader` 按 `tau` 权重对多语料/多任务混合采样（源码 L47-70）。训练时用 toposort 对任务依赖（如 tok→pos→ner 传递）做拓扑排序（import toposort，L23）。这是论文"Stem Cell Hypothesis"的工程落地：同一编码器上共存 10 个任务头。

```python
class MultiTaskModel(torch.nn.Module):
    def __init__(self, encoder, scalar_mixes, decoders, use_raw_hidden_states):
        self.encoder: ContextualWordEmbeddingModule = encoder
        self.scalar_mixes = scalar_mixes
        self.decoders = decoders  # 每个任务一个解码器
```

### 5.2 组件基类体系

`hanlp/common/torch_component.py`：`TorchComponent` 抽象出"建词表→建数据集→建 DataLoader→建模型"四阶段工作流（docstring L31-35 明言是"guideline 而非强制协议"，子类可覆写），config 用 `SerializableDict` 持久化。TF 旧模型走平行的 `keras_component.py`。`hanlp/__init__.py` 的 `load()` 统一入口：模型 ID → `hanlp.pretrained.ALL` 映射 URL → 下载 meta.json → 反射实例化（L34-74，还带 `trust_remote_code` 白名单安全开关）。

### 5.3 分词器体系与词典融合

2.x 分词已全部 Transformer 化（RNN 仅为兼容保留）：`hanlp/components/tokenizers/transformer.py` 的 `TransformerTaggingTokenizer` 继承 TransformerTagger 做 span 标注，特色是双层词典——`dict_force`（强制词典，对输入先做最长匹配切走关键词，优先级高于模型）与 `dict_combine`（合并词典，对模型输出做再合并），解决"新词/领域词覆盖模型预测"的工程痛点（docstring L24-30）。另有 `multi_criteria_cws_transformer.py`（多规范分词）、`tok.py` 的 RNN 版（BMES→词，L26-28）。感知机/CRF/Viterbi 分词则在 1.x 分支（`src/main/java/com/hankcs/hanlp/seg/`，gh api 核实包名）。

### 5.4 模型外置与数据包分发

仓库只含代码不含模型：`plugins/hanlp_common/hanlp_common/constant.py` L14 定义 `HANLP_URL = https://file.hankcs.com/hanlp/`（可环境变量覆盖），`hanlp/pretrained/*.py` 里每个常量都是该域名下的 zip 地址，docstring 内嵌论文引用与指标（如 `mtl.py` 中 electra/ernie-gram/mMiniLMv2/XLM-R/ModernBERT 系列，2025-01 的古典汉语 Kyoto EvaHan 模型）。模型按语料分 open（开源语料训练）/close（闭源语料）两档，商用需注意 close 档与 NC 许可。

## 6. 构建与运行

```bash
# 安装（2.x）
pip install hanlp            # 或从源码: pip install -e . （含 plugins 依赖）
python -c "import hanlp; HanLP = hanlp.load(hanlp.pretrained.mtl.CLOSE_TOK_POS_NER_SRL_DEP_SDP_CON_ELECTRA_SMALL_ZH)"

# 测试（CI 同款，.github/workflows/unit-tests.yml）
python -m pip install -e plugins/hanlp_trie && python -m pip install -e plugins/hanlp_common && python -m pip install -e .
pytest tests && pytest plugins/hanlp_trie/tests

# Java 1.x（1.x 分支，Maven）
mvn package   # pom.xml；portable 分支无依赖精简版

# RESTful 客户端
pip install hanlp_restful    # 或 Java: com.hankcs.hanlp.restful:hanlp-restful（README L67-73）
```

## 7. 测试与工程质量

- 测试：tests/ 7 个文件共 414 行（test_mtl、test_pipeline、test_rules、test_safe_extraction、test_safe_torch_serialization 等），偏冒烟/回归性质而非高覆盖单元测试；hanlp_trie 插件另有独立 tests 目录（CI 中执行，unit-tests.yml）
- CI：GitHub Actions 三平台矩阵（ubuntu/macos/windows × Python 3.6-3.10），带 HANLP_HOME 数据缓存（unit-tests.yml L23-45）；master push 构建通过后自动 twine 发布 PyPI（L47-58）——CI 即发布流水线，节流且要求 master 始终绿
- 工程质量：类型注解较全（typing 遍布）、docstring 规范（Sphinx 格式）、版本号单点维护（hanlp/version.py）；缺陷是测试规模相对 5.8 万行代码偏薄，主要靠"复现指标"文化兜底

## 8. 社区与维护状态

- 活跃度：最近 push 2026-11-07（gh api），open issues 仅 11——项目长期处于"作者精确控仓"状态
- Release：GitHub release 稀疏（v2.1.1 2025-01、v2.1.0 2024-12、v1.8.6 2024-12、v1.8.5 2024-11、v1.8.4 2023-02，gh api）；实际 PyPI 版本走 CI 自动发版更频繁（version.py=2.1.7）
- 贡献者：37 人（gh api contributors），但主体代码近乎全部出自 hankcs 一人（文件头 Author: hankcs 贯穿全仓），典型单人明星项目
- Java 1.x 维护至 2024-12（v1.8.6 "常规维护"），之后基本冻结，新特性全在 2.x；另有 doc-zh/doc-ja 文档分支与 bbs.hankcs.com 论坛（README L27）

## 9. 亮点与风险

亮点：
1. 十任务一体：MTL 框架用一个 Transformer 服务 tok/pos/lem/fea/ner/srl/dep/con/sdp/amr，130 语种（README L33-36、mtl/ 目录）
2. 学术诚信强：每个模型 docstring 内嵌指标，公开承诺全表可复现（README L146-148），有 EMNLP 2021 论文背书
3. 模型即配置外置：pretrained/*.py 声明式模型清单 + 统一下载/反射加载，零模型进仓保持 repo 轻量（722KB tarball）
4. 双轨平滑：Java 1.x 分支继续可用，Python 2.x 主推，RESTful API 覆盖 Python/Java/Go 三客户端（plugins/）
5. 工程细节扎实：分词双词典融合、trust_remote_code 白名单、多平台 CI

风险/局限：
1. 模型许可 CC BY-NC-SA 4.0——闭源商用需要单独授权，close 语料模型边界更严（README L188-191），开源代码可商用不等于整体可商用
2. 单人项目：核心近乎 hankcs 一人维护，bus factor 低；测试覆盖率相对代码量偏薄
3. 双后端包袱：TF 兼容层（keras_component、*_tf.py、TF_USE_LEGACY_KERAS=1 环境变量，hanlp/__init__.py L8）是长期维护负担
4. 模型托管在 file.hankcs.com 单点，仓库不自带模型，离线/内网场景依赖手动下载（pretrained/*.py）

## 10. 总结评价

HanLP 是中文 NLP 领域从传统算法库成功演进为深度学习框架的少数范本：1.x 以 Java 感知机/CRF 体系赢得口碑，2.x 由作者博士论文工作驱动、以 PyTorch MTL 架构实现十任务全覆盖并公开承诺指标可复现，工程（双词典分词、声明式模型分发、三端客户端）与学术（EMNLP 论文、标注规范文档）双轮驱动。代码可商用（Apache-2.0）但预训练模型 NC 许可是商用落地的关键边界。适合：中文 NLP 生产环境直接使用者、想研究"一个编码器多任务头"工程实现的 NLP 工程师、以及想学习开源项目如何长期单人高质量维护的开发者；读源码首选 hanlp/components/mtl/multi_task_learning.py 与 hanlp/common/torch_component.py 两条主线，Java 传统算法爱好者则看 1.x 分支的 seg/ 与 model/ 包。
