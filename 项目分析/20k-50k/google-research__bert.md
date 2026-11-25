# google-research/bert 项目分析

> 星数 41500（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：BERT（Bidirectional Encoder Representations from Transformers）的原始官方 TensorFlow 实现，开启 NLP"预训练+微调"时代的奠基之作（arXiv:1810.04805）。

| 项 | 值 |
|---|---|
| 名称 | google-research/bert |
| 星数 | 41500（快照 2026-10-24；gh api 实测 40030 @2026-11-14） |
| 语言 | Python（TF1.x 风格代码，依赖 `tensorflow >= 1.11.0`，见 `src/requirements.txt`） |
| 许可证 | Apache-2.0（代码与预训练模型同为 Apache 2.0，README FAQ 段明确声明） |
| 仓库状态 | **已归档（archived: true，gh api 实测）**；最后一次代码提交 2020-03-11（gh api commits：'Add links to 24 smaller BERT models'） |
| 规模 | 仅 22 个文件、约 7500 行（Python 代码约 5400 行），是一个"代码即论文附录"式的小仓库 |
| Release | 0 个（gh api 实测），模型 checkpoint 全部走 Google Cloud Storage zip 分发 |

## 2. 项目是什么，解决什么问题

2018 年 10 月，Google AI（Jacob Devlin、Ming-Wei Chang、Kenton Lee、Kristina Toutanova）发布 BERT 论文与本仓库，用"深度双向 Transformer 预训练语言表示"一举刷新 11 项 NLP 任务纪录。README 开篇即给出发售成绩单：SQuAD v1.1 单模型 91.8 F1（当时第一，超第二名 1.6 点）、MultiNLI 86.7（对比 OpenAI GPT 82.2），且这些结果"几乎没有任务特定的网络结构设计"（`src/README.md` Introduction 与结果表）。BERT 的核心创新是第一个**无监督、深度双向**的预训练系统：随机遮住 15% 的词，用深层双向 Transformer 编码器还原它们（MLM 掩码语言模型），再加一个"句子 B 是否紧跟句子 A"的二分类任务（NSP 下一句预测），在 Wikipedia + BookCorpus 上训练 1M 步（`src/README.md` What is BERT? 一节）。

仓库解决的第二个问题（也是它涨到 4 万星的根本原因）是把"用上 BERT"的成本降到几乎为零：Google 一次性放出 BERT-Base（110M 参数，12 层/768 hidden/12 heads）与 BERT-Large（340M 参数，24 层/1024 hidden/16 heads）的 cased/uncased、Whole Word Masking、多语言（104 语言）与中文等十几个预训练 checkpoint（zip 内含 `bert_model.ckpt`、`vocab.txt`、`bert_config.json` 三件套，`src/README.md` Pre-trained models 一节），并提供"按钮式"复现脚本：`run_classifier.py`（GLUE 分类）、`run_squad.py`（问答）、`run_pretraining.py`（自己继续预训练）、`extract_features.py`（抽取 ELMo 式固定向量）。研究者只需下载 checkpoint、换上自己的数据集即可在单卡 GPU 或 Cloud TPU 上数小时内微调出 SOTA 结果。

"预训练一次（贵，4-16 块 Cloud TPU 上 4 天）、微调多次（便宜，单 TPU 最多 1 小时）"的两阶段范式由此确立（`src/README.md`）。这一范式随后统治了整个 NLP 并外溢到 CV、语音，本仓库是那个时代的起点性 artifact。Hugging Face 的 PyTorch 移植（原 pytorch-pretrained-BERT，即 transformers 库中 `bert` 的前身）在 2018 年 11 月即与本仓库 checkpoint 完全互通（README 2018-11-05 更新段落及 FAQ），今天的 transformers 里 `BertModel` 仍是本仓库 `modeling.py` 的逐层移植——本仓库是几乎所有现代预训练框架的"蓝本源代码"。

2020 年 3 月仓库最后一次实质更新（补充 24 个小型 BERT 模型，来自论文 Well-Read Students Learn Better，arXiv:1908.08962），之后进入只读维护；`CONTRIBUTING.md` 明言："BERT needs to maintain permanent compatibility with the pre-trained model files, so we do not plan to make any major changes"——即代码按设计冻结。GitHub 仓库现处于 archived 状态（gh api 实测），属于官方盖章的"历史里程碑、不再演进"仓库。

## 3. 技术栈与依赖

- **Python 2/3 + TensorFlow 1.11+**：唯一依赖即 `tensorflow >= 1.11.0`（`src/requirements.txt` 仅 2 行）；README 明确"tested with TensorFlow 1.11.0, Python2 and Python3"。
- **TF1 时代 API**：全程使用 `tf.layers.dense`、`tf.variable_scope`、`tf.get_variable`、`tf.train.Optimizer`、`tf.contrib.tpu.TPUEstimatorSpec`、`tf.gfile`、`tf.flags` 等在 TF2 中已移除或重写的接口。
- **six**：Python2/3 兼容层（`modeling.py`、`tokenization.py`、`run_squad.py` 均 import six）。
- **numpy**：形状/初始化辅助。
- **TFRecord / tf.train.Example**：预训练数据序列化格式（`create_pretraining_data.py`）。
- **TF Hub**：可选加载方式（`run_classifier_with_tfhub.py` 与 `predicting_movie_reviews_with_bert_on_tf_hub.ipynb`）。
- 无任何打包工程文件（无 setup.py/pyproject），不是可 pip 安装的库，而是"脚本集合"，靠直接 `python run_*.py --flags` 运行。

## 4. 目录结构与代码组织

仓库为单层扁平结构，无子目录，所有文件平铺在根目录（gh api git/trees 与 tarball 双重确认）：

| 文件 | 职责 |
|---|---|
| `modeling.py`（986 行） | BERT 模型定义：BertConfig + BertModel + Transformer 编码器全部组件 |
| `optimization.py`（174 行） | 优化器：线性 warmup/衰减 + AdamWeightDecayOptimizer（AdamW 原始实现之一） |
| `tokenization.py`（399 行） | 分词：FullTokenizer = BasicTokenizer（清洗/小写/去重音/中文按字）+ WordpieceTokenizer |
| `create_pretraining_data.py`（469 行） | 预训练数据生成：纯文本 → MLM/NSP 训练实例 → TFRecord |
| `run_pretraining.py`（493 行） | 预训练主脚本：MLM + NSP 双损失，TPUEstimator |
| `run_classifier.py`（981 行） | GLUE 句子/句对分类微调（内置 cola/mnli/mrpc/xnli 四个 Processor） |
| `run_squad.py`（1283 行） | SQuAD 1.1/2.0 问答微调，仓库最大文件，含滑窗、char 对齐、n-best、空答案阈值 |
| `extract_features.py`（419 行） | 抽取各层隐藏向量输出 JSONL（ELMo 式用法） |
| `run_classifier_with_tfhub.py`（314 行） | 从 TF Hub 拉取 BERT module 的微调示例 |
| `predicting_movie_reviews_with_bert_on_tf_hub.ipynb` | Colab 演示：TF Hub 版 IMDB 情感分类 |
| `modeling_test.py` / `tokenization_test.py` / `optimization_test.py` | 单元测试（共约 460 行） |
| `README.md`（1117 行） | 主文档：模型下载、微调/预训练教程、OOM 表、FAQ |
| `multilingual.md`（303 行） | 多语言/中文模型说明与 XNLI 结果 |
| `sample_text.txt`、`__init__.py`、`LICENSE`、`CONTRIBUTING.md`、`.gitignore` | 辅助文件 |

组织方式评述：典型的"论文配套代码"单体扁平结构——没有包抽象、没有依赖注入，脚本间通过 import `modeling/optimization/tokenization` 三个核心模块复用；对外接口就是命令行 flags。这种极简组织反而成了它的传播优势：任何一个脚本都能独立读完整。

## 5. 核心架构与关键实现

### 5.1 Transformer 编码器（`src/modeling.py`）

`BertModel.__init__`（`modeling.py:131-232`）把模型拆成三段：embeddings（词/位置/段类型三项相加 + LayerNorm + dropout）、encoder（N 层 Transformer）、pooler（取 `[CLS]` 首 token 隐状态接一层 tanh dense 输出句向量）。整个实现用 TF1 graph 风格手工展开，无 Keras 对象。自注意力在 `attention_layer`（`modeling.py:558-751`）中实现，mask 处理是经典技巧——把 padding 位置加分项设为 -10000 后过 softmax：

```python
# modeling.py:712-716  masked 位置加分 -10000 等价于 softmax 中剔除
adder = (1.0 - tf.cast(attention_mask, tf.float32)) * -10000.0
attention_scores += adder
attention_probs = tf.nn.softmax(attention_scores)
```

`transformer_model`（`modeling.py:754-892`）为标准 post-LN 残差块：attention → dense 投影 + 残差 + LayerNorm → GELU 中间层（3072）→ 降维 + 残差 + LayerNorm；并用"全程保持 2D 张量、避免多余 reshape"的技巧优化 TPU 性能（`modeling.py:819-823` 注释）。前向返回**所有层**的输出（`all_encoder_layers`），为后来"从不同层抽特征"的研究（如BERT 的各层探测）预留了接口。

### 5.2 AdamWeightDecayOptimizer——AdamW 的原始实现（`src/optimization.py`）

这是被后来无数框架照抄的 174 行文件。`create_optimizer`（`optimization.py:25-84`）组合：多项式（线性）学习率衰减 + 线性 warmup + 全局范数 1.0 梯度裁剪 + AdamWeightDecayOptimizer（weight_decay_rate=0.01，排除 LayerNorm 与 bias）。`apply_gradients`（`optimization.py:108-157`）手写 Adam 更新，并在注释中给出了 Why-AdamW 的著名解释：

```python
# optimization.py:139-145
# Just adding the square of the weights to the loss function is *not*
# the correct way of using L2 regularization/weight decay with Adam,
# since that will interact with the m and v parameters in strange ways.
# Instead we want ot decay the weights in a manner that doesn't interact
# with the m/v parameters.
if self._do_use_weight_decay(param_name):
    update += self.weight_decay_rate * param
```

即"解耦权重衰减"（decoupled weight decay）。注释还特别提醒：微调时应沿用此优化器，但 Adam 的 m/v 动量**不**从 checkpoint 加载（`optimization.py:56-58`）。

### 5.3 分词器（`src/tokenization.py`）

`FullTokenizer` = 两级流水线：`BasicTokenizer`（清洗非法字符 → 中文按字切分 `_tokenize_chinese_chars`，`tokenization.py:251-284` 按 CJK 码位区间加空格 → 小写 + NFD 去重音 → 标点两侧切开）+ `WordpieceTokenizer`（贪心最长匹配，`##` 前缀标子词，`tokenization.py:308-359`）。中文按字切分是 2018-11-03 为多语言/中文模型加入的功能（README 更新日志）。`validate_case_matches_checkpoint`（`tokenization.py:28-75`）还会按 checkpoint 目录名启发式校验 `--do_lower_case` 是否与 cased/uncased 模型匹配，防呆设计。

### 5.4 预训练数据生成与双任务（`src/create_pretraining_data.py`、`src/run_pretraining.py`）

`create_instances_from_document`（`create_pretraining_data.py:223-335`）实现 NSP：按真实句子边界组装 A/B 段，50% 概率从其他文档随机采 B；`dupe_factor`（默认 5 次）让每篇文档以不同 mask 组合重复采样；10% 概率生成短序列以缩小预训练-微调长度失配（`short_seq_prob`，注释见 `create_pretraining_data.py:232-241`）。MLM 掩码是经典的 80/10/10：

```python
# create_pretraining_data.py:392-399
# 80% of the time, replace with [MASK]
if rng.random() < 0.8:
    masked_token = "[MASK]"
else:
    # 10% of the time, keep original
    if rng.random() < 0.5:
        masked_token = tokens[index]
    # 10% of the time, replace with random word
```

`--do_whole_word_mask=True` 开启 WWM：把 `##` 开头的 WordPiece 归并到其词首候选集整词掩码（`create_pretraining_data.py:346-363`），训练侧代码零改动。输出为 `tf.train.Example` TFRecord，字段含 input_ids/input_mask/segment_ids/masked_lm_positions/ids/weights 与 next_sentence_labels。`run_pretraining.py` 的 `model_fn_builder`（`run_pretraining.py:109-225`）以 masked LM 交叉熵 + NSP 交叉熵为联合损失，同样走 TPUEstimator。

### 5.5 微调三脚本（`src/run_classifier.py`、`src/run_squad.py`、`src/extract_features.py`）

三个脚本共享同一模式：flags 定义 → DataProcessor 读 TSV/JSON 生成 InputExample → 转 InputFeature → `model_fn_builder` 构造 TPUEstimatorSpec → `tf.train.init_from_checkpoint` 按变量名映射加载预训练权重（`run_classifier.py:647-661`，借助 `modeling.get_assignment_map_from_checkpoint` 只恢复 encoder 部分、随机初始化任务头）。`run_classifier.py` 内置 `{"cola","mnli","mrpc","xnli"}` 四个 Processor（`run_classifier.py:786-791`）作为自定义任务的模板。`run_squad.py` 最复杂：长文档按 `doc_stride=128` 滑窗切块、char 级答案标签对齐到 token、start/end 双 span 预测、n-best 输出，以及 SQuAD 2.0 的"空答案概率差阈值"后处理（`null_score_diff_threshold`，README SQuAD 2.0 一节）。`extract_features.py` 按 `--layers=-1,-2,-3,-4` 输出多层激活到 JSONL，是"BERT 当静态特征提取器"的官方用法。

## 6. 构建与运行

无构建环节，直接跑脚本（命令均来自 `src/README.md`，需 TF 1.11 环境，现代环境无法运行）：

```shell
# 微调 GLUE/MRPC（README.md:440-453）
python run_classifier.py \
  --task_name=MRPC --do_train=true --do_eval=true \
  --data_dir=$GLUE_DIR/MRPC \
  --vocab_file=$BERT_BASE_DIR/vocab.txt \
  --bert_config_file=$BERT_BASE_DIR/bert_config.json \
  --init_checkpoint=$BERT_BASE_DIR/bert_model.ckpt \
  --max_seq_length=128 --train_batch_size=32 \
  --learning_rate=2e-5 --num_train_epochs=3.0 --output_dir=/tmp/mrpc_output/

# SQuAD（README.md:527-541，GPU 上 BERT-Base 约 88.5 F1）
python run_squad.py --do_train=True --train_file=$SQUAD_DIR/train-v1.1.json \
  --do_predict=True --predict_file=$SQUAD_DIR/dev-v1.1.json \
  --train_batch_size=12 --learning_rate=3e-5 --num_train_epochs=2.0 \
  --max_seq_length=384 --doc_stride=128 --output_dir=/tmp/squad_base/

# 预训练数据生成 + 预训练（README.md:908-942）
python create_pretraining_data.py --input_file=./sample_text.txt \
  --output_file=/tmp/tf_examples.tfrecord --vocab_file=$BERT_BASE_DIR/vocab.txt \
  --max_seq_length=128 --max_predictions_per_seq=20 --masked_lm_prob=0.15 \
  --random_seed=12345 --dupe_factor=5
python run_pretraining.py --input_file=/tmp/tf_examples.tfrecord ... --do_train=True

# 单测（tf.test 风格，无 requirements 之外的测试依赖）
python modeling_test.py && python tokenization_test.py && python optimization_test.py
```

TPU 用户加 `--use_tpu=True --tpu_name=$TPU_NAME`，且输入输出须在 GCS（README Fine-tuning with Cloud TPUs 一节）。

## 7. 测试与工程质量

- 测试规模小而精：3 个测试文件共约 460 行、14 个用例（`modeling_test.py` 覆盖默认配置前向与 config JSON 往返；`tokenization_test.py` 覆盖全流程分词、中文、大小写、WordPiece、ids 转换、空白/控制符/标点判定；`optimization_test.py` 仅 1 个 `test_adam`，48 行）。全部基于 `tf.test.TestCase`，无 CI 配置（仓库文件树中不存在 `.github/` 目录，gh api 确认），无 lint/格式化配置。
- 代码质量观感：注释密度极高，几乎每个函数都有完整 Args/Returns docstring，且大量注释解释"为什么"（如 WWM 不改训练代码的原因、-10000 mask 的等价性、AdamW 与 L2 的区别）；命名统一 2 空格缩进的 Google Python 风格。工程化程度低是刻意选择——`CONTRIBUTING.md` 声明不做大改以永久兼容预训练 checkpoint，仅接受重构/文档类小补丁。

## 8. 社区与维护状态

- **维护状态：官方冻结 + 已归档**。最后实质提交 2020-03-11（gh api commits）；`pushed_at` 显示 2024-07-23（归档动作）；GitHub `archived: true`（gh api 实测）。open issues 879 个且无人处理（归档仓库）。
- **贡献者结构**：核心作者即论文四位作者（README Contact information：Jacob Devlin、Ming-Wei Chang、Kenton Lee 等）；contributors 接口返回多页外部贡献者，但实际代码改动极少——2019-10 至 2020-03 间主要提交是补文档与小型模型链接（gh api commits 样本）。
- **发布节奏**：无 GitHub release（gh api 实测 0 个），模型以 GCS zip 形式发布（`storage.googleapis.com/bert_models/...`），更新公告直接写进 README 顶部的 "New ..." 时间线段落（2018-10 至 2020-03 共 8 条公告）。
- **现状提示**：README 引用的 Cloud TPU/Colab 链接与当年价格为 2018 年语境；TF1.x 代码在 TF2 环境下无法运行（`tf.contrib` 已删除）。社区实际使用均迁移到 Hugging Face transformers 等下游实现，本仓库现在的价值是文献/源码考古。

## 9. 亮点与风险

**亮点**
1. 历史奠基价值：开启"预训练+微调"范式的官方实现，transformers 库中 `BertModel` 的直接蓝本（README 2018-11-05 段确认 HF 移植与 checkpoint 互通），NLP 引用量最高级别论文（arXiv:1810.04805）的配套源码。
2. 代码极简自洽：22 个文件约 7500 行完整覆盖 模型/优化器/分词/预训练数据/四种任务脚本，每个文件可独立读完，是学习 Transformer 编码器工程实现的最佳教材之一（`modeling.py`/`optimization.py`/`create_pretraining_data.py`）。
3. 细节注释即文档：-10000 attention mask、AdamW 解耦权重衰减的论证、WWM 分组逻辑、position embedding 切片省内存等关键决策都有就地注释说明出处与理由。
4. 完整可复现承诺：README 提供 GLUE/MRPC/SQuAD 1.1/2.0 的逐条命令、预期输出数值（如 MRPC 84.55%）、Titan X 12GB 下的 batch size 内存对照表，以及单 TPU 微调时长承诺。
5. 中文/多语言关怀：`tokenization.py` 内置 CJK 按字切分，`multilingual.md` 给出 104 语言模型与 XNLI 零样本迁移（英→中 63.8%）等诚实的数据分析。

**风险/局限**
1. **不可运行现状**：TF1.x + `tf.contrib` + Python2 时代代码，TF2 已删除相关 API，`requirements.txt` 指定的 tensorflow>=1.11 无法在现代环境安装运行；只能作为静态阅读资料。
2. 官方已归档：879 个 open issues 无响应，模型下载链接指向 GCS（长期有效性依赖 Google Storage 桶存续），文档中的 Colab/TPU 教程链接部分已失效或过时。
3. 功能边界：不含 WordPiece 词表训练代码（原实现是 C++ 内部库，README 明说不发布）、GPU 仅支持单卡、无 gradient accumulation/checkpointing（README OOM 一节自认 "not implemented in the current release"），BERT-Large 在 12-16GB GPU 上无法复现论文结果。
4. 预训练数据不可得：Wikipedia+BookCorpus 预处理语料不发布，BookCorpus 本身已下架（README Pre-training data 一节），完整复现预训练几乎不可能。

## 10. 总结评价

google-research/bert 是"一篇论文 + 一座里程碑"型仓库：它在 2018 年把 NLP 推入预训练时代，自身则以冻结的 TF1.x 形态成为考古层——GitHub 已归档、2020 年 3 月后不再改动、代码已不可运行，但它的价值不因停更而降低，反而因其"原始出处"身份而上升：读懂 `modeling.py` 的 986 行就等于读懂了 transformers 库 BertModel 的全部血统，读懂 `optimization.py` 的 174 行就理解了 AdamW 为何成为预训练标配，读懂 `create_pretraining_data.py` 就掌握了 MLM/NSP 的工程细节。它不适合作为今天的生产工具（请用 transformers），但适合三类读者：想理解 Transformer 编码器底层实现的工程师、复现/研究预训练范式的学生、以及研究 NLP 技术史的人。四万星不是给它的功能的，是给它开启的那个时代的。
