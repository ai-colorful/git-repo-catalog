# shiyu-coder/Kronos 项目分析

> 星数 31200（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-06

## 1. 项目速览

一句话定位：Kronos 是号称"首个开源金融 K 线基础模型"（A Foundation Model for the Language of Financial Candlesticks），用"K 线 tokenizer + 自回归 Transformer"两阶段框架对 OHLCV 序列做概率化预测，论文已被 AAAI 2026 接收（`README.md` News 部分）。

| 基本事实 | 内容 |
|---|---|
| 名称 / 作者 | shiyu-coder/Kronos（清华系研究团队，论文作者含 Jian Li 等，见 README 引用条目） |
| 星数 / Fork | 31200（快照）；2026-11-06 经 GitHub API 实测已达约 46.9k stars、7.7k forks |
| 主语言 | Python（纯 Python + PyTorch） |
| 许可证 | MIT（`LICENSE`，Copyright (c) 2025 ShiYu） |
| 创建 / 最近 push | 2025-07-01 创建；最近 push 2026-04-13（GitHub API） |
| 论文 | arXiv:2508.02739（2025-08-02 上线），AAAI 2026 接收 |
| 模型权重 | Hugging Face org "NeoQuasar"：mini(4.1M)/small(24.7M)/base(102.3M) 开源，large(499.2M) 未开源（README Model Zoo 表） |
| 仓库规模 | 约 11.1k 行 Python，无打包配置（无 pyproject/setup.py），直接以脚本目录形式分发 |

## 2. 项目是什么，解决什么问题

Kronos 把金融市场 K 线（K-line / candlestick）序列当作一种"语言"来建模：每根 K 线是一个多维连续向量（open/high/low/close/volume/amount），作者认为通用时序基础模型（TSFM）没有针对金融数据的高噪声、多变量耦合特性做专门设计，因此提出两阶段框架（`README.md` Introduction；`model/kronos.py`）：

1. **第一阶段：K 线 tokenizer。** `KronosTokenizer` 是一个 Transformer encoder-decoder 结构的量化器，把连续 OHLCV 向量压缩成"层级离散 token"——即把二进制码拆成高位 s1（粗粒度）与低位 s2（细粒度）两段 token（`model/module.py` 的 `BSQuantizer`/`BinarySphericalQuantizer`，`model/kronos.py:98-113` 的 `z_pre`/`z` 双路解码）。
2. **第二阶段：自回归大 Transformer。** `Kronos` 是 decoder-only 模型，在离散 token 序列上以"预测下一个 token"的方式预训练，输出分布后经 tokenizer 解码回连续 K 线数值，天然支持概率化预测（多次采样取均值）。

这个设计带来几个实际好处：预测是分布而非单点（`T`/`top_p`/`sample_count` 控制采样，`model/kronos.py:389-469`）；同一模型可跨市场、跨时间粒度使用（时间信息由 minute/hour/weekday/day/month 五路 embedding 注入，`model/module.py:536-562`）；输入是统一的 6 通道归一化序列，不依赖具体标的特征工程。

预训练官方声称覆盖全球 45+ 交易所数据（README 首页声明），但**预训练代码与数据未随仓库发布**，仓库发布的是模型权重（HF Hub）+ 微调脚本 + 推理管线。仓库同时提供 Qlib（A股）微调回测全流程、CSV 自定义数据微调流程和 Flask WebUI 演示，属于"研究演示 + 可复用推理组件"形态，而非生产级量化系统。README 明确写有 Disclaimer：微调管线只是演示，"not a production-ready quantitative trading system"（`README.md` 约 221 行）。

## 3. 技术栈与依赖

- **核心**：Python 3.10+，PyTorch >= 2.0（`requirements.txt`）。注意力用 `F.scaled_dot_product_attention`（`model/module.py:345`），依赖较新 PyTorch。
- **关键第三方库**（`requirements.txt`）：`einops`（张量重排）、`huggingface_hub`（模型通过 `PyTorchModelHubMixin` 直接挂 HF Hub 下载，`model/kronos.py:13`）、`pandas/numpy`（数据管线）、`matplotlib`（绘图）、`safetensors`（权重格式）、`tqdm`。
- **微调可选依赖**：`pyqlib`（微软 Qlib，A股数据与回测，`finetune/qlib_test.py:1-15`）、`comet_ml`（实验跟踪，`finetune/config.py:75-84`）、`torchrun` 分布式训练（DDP）。
- **WebUI**：Flask + flask-cors + plotly（`webui/requirements.txt`），独立于主 requirements。
- **数据获取示例**：`akshare`（国内行情接口，`examples/prediction_new.py:16`）。
- 无打包/安装配置：所有脚本靠 `sys.path.append("../")` 这种相对路径 hack 导入模块（如 `model/kronos.py:9`、`finetune/train_tokenizer.py:18`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `model/` | 核心库：`module.py`（基础组件：BSQ 量化器、RoPE、RMSNorm、SwiGLU FFN、层级 embedding、双头输出）+ `kronos.py`（KronosTokenizer、Kronos 主干、KronosPredictor 推理封装、自回归采样循环），共约 1230 行，是全仓库的价值核心 |
| `finetune/` | 基于 Qlib 的 A 股微调管线：config、数据集（pickle 滑窗样本）、tokenizer/predictor 训练脚本（DDP）、top-K 回测（`qlib_test.py`） |
| `finetune_csv/` | CSV 自定义数据微调管线：YAML 配置驱动（`configs/config_ali09988_candle-5min.yaml`），附带港股 09988 五分钟 K 线示例数据与中英双语 README |
| `examples/` | 推理示例：官方极简示例、无成交量预测、批量预测、A股日线/akshare 数据脚本、社区贡献的 1332 行 `prediction_new.py` 与 1624 行 GUI 版本、`yuce/` 社区回测产物 |
| `webui/` | Flask + Plotly 网页演示：模型选择（mini/small/base）、上传数据、出预测图，`app.py` 708 行 |
| `tests/` | pytest 回归测试：固定 HF revision 权重 + 固定随机种子，对比预存输出 CSV |
| `figures/` | README 用图（架构总览、预测示例、回测示例） |

组织方式评述：单仓库脚本式布局，没有 Python 包结构；核心代码高度集中在 `model/` 两个文件里，其余都是围绕它的使用示例与训练脚本。`finetune/`（Qlib 专用）与 `finetune_csv/`（通用 CSV）两条微调路线并存，后者是后加的更易用版本。examples 中混入了社区贡献的大型脚本与输出产物（`examples/yuce/`、`webui/prediction_results/` 下几十个 JSON），工程整洁度一般。

## 5. 核心架构与关键实现

### 5.1 层级离散化：BSQ tokenizer（`model/module.py`, `model/kronos.py`）

量化器直接采用 Binary Spherical Quantization 官方实现（`model/module.py:39-50` 注释标明来自 arXiv:2406.07548）：先把输入 L2 归一化到球面，再按符号二值化得到 K 维 ±1 码，用熵正则（per-sample 熵 + codebook 熵，可微近似）保证码本利用均匀。`KronosTokenizer` 的前向把 s1_bits 与 s2_bits 拆成"粗/细"两组 token（`model/module.py:225-254` `BSQuantizer.forward(half=...)`），推理时只取这两组索引（`model/kronos.py:398` `tokenizer.encode(x, half=True)`）：

```python
# model/kronos.py:398  推理入口：历史窗口编码为 (s1, s2) 两路索引
x_token = tokenizer.encode(x, half=True)
```

tokenizer 是带 `PyTorchModelHubMixin` 的 nn.Module，`from_pretrained("NeoQuasar/Kronos-Tokenizer-base")` 即可拉取权重（`model/kronos.py:13`）。

### 5.2 层级 token 的自回归建模（`model/kronos.py:180-328`, `model/module.py:400-513`）

主干是 pre-LN RMSNorm + RoPE + SwiGLU 的标准 decoder-only Transformer（`TransformerBlock`，`model/module.py:465-483`）。每步预测分两级：

1. 主干输出 s1 logits，采样得到粗 token；
2. `DependencyAwareLayer` 用交叉注意力把采样出的 s1 embedding 作为 query 条件注入，再由 `DualHead.cond_forward` 输出 s2 logits（`model/kronos.py:265-276`）：

```python
# model/kronos.py:270-275  s2 以采样后的 s1 为条件（hierarchical autoregression）
s1_probs = F.softmax(s1_logits.detach(), dim=-1)
sample_s1_ids = torch.multinomial(s1_probs.view(-1, self.s1_vocab_size), 1).view(s1_ids.shape)
sibling_embed = self.embedding.emb_s1(sample_s1_ids)
x2 = self.dep_layer(x, sibling_embed, key_padding_mask=padding_mask)
s2_logits = self.head.cond_forward(x2)
```

时间信息通过 `TemporalEmbedding`（minute/hour/weekday/day/month 五路相加，`model/module.py:536-562`）加到 token embedding 上，这是"多时间粒度统一建模"的落点。

### 5.3 自回归推理循环与概率化预测（`model/kronos.py:389-469`）

`auto_regressive_inference` 维护长度 `max_context` 的滚动 token 缓冲区（超出后 `torch.roll` 滑窗），逐步生成 `pred_len` 步；`sample_count` 条并行采样路径在最后统一解码并取均值作为点预测（`model/kronos.py:464-467`）。输入在 `KronosPredictor.predict` 里按"仅用回看窗口统计量"做 z-score 归一化并裁剪到 ±5（`model/kronos.py:544-547`），预测后逆归一化。`predict_batch` 支持多标的 GPU 并行（`model/kronos.py:562-661`）。

### 5.4 微调管线（`finetune/`, `finetune_csv/`）

- tokenizer 微调损失 = s1 粗码与全码两路 MSE 重建损失 + BSQ 熵/commit 损失（`finetune/train_tokenizer.py:137-144`）；predictor 微调损失 = s1/s2 双头交叉熵均值（`finetune/train_predictor.py:109`，loss 定义在 `model/module.py:494-507`）。
- `finetune/dataset.py` 的 `QlibDataset` 用滑窗切样本，归一化统计量只取回看窗口，明确注释"prevent future data leakage"（`finetune/dataset.py:107-117`）——这是数据泄漏防护上做对了的细节。
- `finetune/qlib_test.py` 用 Qlib 的 `TopkDropoutStrategy` 做简单 top-K 换手回测并输出风险分析。
- `finetune_csv/` 提供另一条不依赖 Qlib 的路线：YAML 配置 + CSV 输入（列要求见 `finetune_csv/README.md`），附港股价差数据样例。

## 6. 构建与运行

无构建步骤，直接 `pip install -r requirements.txt`（`README.md:92`）。

```shell
# 推理（官方最小示例，依赖 HF Hub 自动下载权重）
python examples/prediction_example.py          # 需要 ./data/XSHG_5min_600977.csv
python examples/prediction_wo_vol_example.py   # 无成交量输入
python examples/prediction_batch_example.py    # 多标的批量

# Qlib 微调（需先 pip install pyqlib 并准备本地 Qlib 数据）
python finetune/qlib_data_preprocess.py
torchrun --standalone --nproc_per_node=NUM_GPUS finetune/train_tokenizer.py
torchrun --standalone --nproc_per_node=NUM_GPUS finetune/train_predictor.py
python finetune/qlib_test.py --device cuda:0

# CSV 自定义微调
# 编辑 finetune_csv/configs/*.yaml 后：
python finetune_csv/train_sequential.py

# WebUI 演示
python webui/run.py        # 或 bash webui/start.sh

# 回归测试（会从 HF 下载 pinned revision 权重，CPU 即可）
pytest tests/test_kronos_regression.py
```

关键提示（README:99）：small/base 的 `max_context=512`，lookback 不要超限，Predictor 会自动截断。

## 7. 测试与工程质量

- **测试**：仅一个测试文件 `tests/test_kronos_regression.py`（140 行），风格是"黄金输出回归测试"：固定 HF 模型 revision（MODEL_REVISION/TOKENIZER_REVISION 写死在测试里）、固定 seed、`top_k=1` 贪心采样，对比仓库内预存的 `tests/data/regression_output_512.csv`/`_256.csv`（相对误差 1e-5），另有一个 MSE 数量级 sanity check。这能有效防推理回归，但不覆盖模块单元行为，也没有微调代码的测试。
- **CI**：tarball 中无 `.github/` 目录，无 GitHub Actions/issue 模板/release 流程配置（未确认线上是否另行配置）。
- **代码质量**：核心 `model/` 两个文件注释较完整、结构清晰；但存在不少工程毛刺——`sys.path.append("../")` 导入 hack、`model/kronos.py:127` 注释里"Assuming x is a tuple"的半成品式说明、examples 混入社区巨型脚本与结果产物、`finetune/config.py:79` 留有占位 API key 字段。README 主动声明 `finetune/` 中大量注释由 Gemini 2.5 Pro 生成、可能有误（README:308），态度坦诚。
- **依赖管理**：主 requirements 混用钉版本与不钉版本（`numpy` 未钉、`pandas==2.2.2`），webui 单独一份 requirements，无 lockfile。

## 8. 社区与维护状态

- **活跃度**：仓库 2025-07-01 创建，一年内冲到 3 万+ 星（快照 31200；2026-11-06 GitHub API 实测约 46.9k，增速极快）。最近 push 2026-04-13，为合并多个社区 PR（#243 批量维度训练修复、#232 top-k 采样 bug 修复等，GitHub commits API）——说明社区贡献在被合并，但近半年（2026-05 之后）无新 push（未确认是否暂停维护）。
- **贡献者**：18 人（GitHub contributors API），核心仍是原作者；examples/ 里有明显社区贡献痕迹（`prediction_new.py`、`prediction_new_GUI.py`、`examples/yuce/`）。
- **Release**：0 个正式 release/tag（GitHub releases API），版本演进只体现在 HF 权重仓库与 git 历史。
- **生态**：HF org "NeoQuasar" 下 4 个模型权重 + 2 个 tokenizer；在线 demo（shiyu-coder.github.io/Kronos-demo，BTC/USDT 24h 预测）；README 通过 zdoc.app 自动维护 8 种语言翻译。
- **Issue**：297 个 open issue（API），相对代码体量偏多，反映使用者众但维护带宽有限。

## 9. 亮点与风险

**亮点**

1. 架构主张清晰且实现完整：BSQ 层级离散化 + 双级自回归（s1 条件化 s2）在 `model/` 约 1200 行内完整落地，量化器直接复用 BSQ 官方实现并注明出处（`model/module.py:39-50`），可信度高。
2. 概率化预测是该模型区别于点预测 TSFM 的实质卖点：温度/top-p/多路径采样平均（`model/kronos.py:389-469`），金融场景下比单点回归更合理。
3. 开箱即用程度高：权重挂 HF Hub，`KronosPredictor` 自动处理归一化/截断/设备选择/批量预测，几行代码出结果；另有 WebUI 与 live demo。
4. 训练材料相对完备（对"推理-only"论文仓而言是加分项）：tokenizer 与 predictor 两段微调脚本、Qlib 与 CSV 两条数据路线、回测脚本齐备，且数据集实现明确防未来泄漏（`finetune/dataset.py:107-117`）。
5. 学术背书：arXiv:2508.02739 + AAAI 2026 接收（README News），论文-代码可互相对照。

**风险/局限**

1. **预训练不可复现**：预训练代码、数据构成（45+ 交易所的具体清单）、训练配置均未公开，仓库只有微调与推理；Kronos-large（499M）权重也未开源，"基础模型"的根基部分不可复现。
2. 回测与评测参考价值有限：`finetune/qlib_test.py` 的 top-K 换手策略是作者自己声明的教学演示（README Disclaimer），未计入交易成本/滑 crowding 等真实因素；examples 中的社区回测产物（`examples/yuce/`）来源与口径不明。**本分析为客观技术描述，不构成任何投资建议；模型预测输出不等于投资建议。**
3. 工程化程度低：无包结构、无 CI、无版本 release、依赖未完全钉死；测试只有一条推理回归链路。
4. 采样型输出的稳定性问题：自回归 + 随机采样意味着同参数下结果有方差，回归测试用 `top_k=1` 绕开了这一点（`tests/test_kronos_regression.py:76`），实际使用中 `sample_count` 的均值收敛性需要用户自行验证。

## 10. 总结评价

Kronos 是"金融 K 线版 GPT"这一想法目前最有代表性的开源实现：把 OHLCV 视作语言、用 BSQ tokenizer 离散化、用 decoder-only Transformer 做层级自回归预测，架构叙事完整、核心代码质量不错、HF 权重即下即用，论文与代码互为支撑，一年内 3 万+ 星的热度有真实的技术与社区基础。它的边界同样清楚：预训练过程不开源导致"基础模型"能力不可复现，评测仅到教学级回测为止，工程化（打包、CI、测试覆盖）仍是研究仓库水准。它适合量化研究者作为预测组件快速实验与微调（尤其配合 `finetune_csv/` 接自有数据），适合 LLM/时序建模方向的研究者阅读 `model/` 两个文件学习"离散化 + 层级自回归"建模范式；但任何把它当作可直接实盘的量化系统的期待都超出了项目自身的定位，产出信号需自行接组合优化与风控，且不应将预测结果等同于投资依据。
