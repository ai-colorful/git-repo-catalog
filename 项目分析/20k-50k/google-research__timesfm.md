# google-research/timesfm 项目分析

> 星数 26500（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-21
>
> gh api 核实（2026-11-21）：实际星数已增至 35652，master 分支最近 push 2026-11-13，活跃。

## 1. 项目速览

TimesFM 是 Google Research 开源的预训练时间序列基础模型（time-series foundation model），面向零样本（zero-shot）时间序列预测：任意单变量/多变量序列直接喂入即可得到带分位数区间的预测，无需训练。

| 项 | 值 |
|---|---|
| 名称 | google-research/timesfm |
| 星数 | 35652（gh api 2026-11-21；任务快照 26500） |
| 语言 | Python |
| 许可证 | Apache-2.0（源码及 ≤2.5 权重）；TimesFM 3.0 权重为非商业许可（README.md "License notice for pretrained weights"） |
| 默认分支 | master（gh api） |
| 最近活跃 | 2026-11-13 push（gh api） |
| 当前包版本 | timesfm 3.0.2（pyproject.toml） |
| 配套论文 | arXiv:2310.10688，ICML 2024（README.md） |
| 最新检查点 | google/timesfm-3.0-pytorch（Hugging Face Hub） |

## 2. 项目是什么，解决什么问题

传统预测流程要为每类数据（销量、气温、能耗、股价……）单独选特征、调参、训练模型。TimesFM 的思路是把 LLM 领域的"基础模型 + 零样本推理"范式搬到时间序列上：在大量合成与真实序列上预训练一个 decoder-only 的 patch 化 Transformer，推理时把历史序列切成 patch 输入，直接输出未来 patch 的分位数预测（README.md；src/timesfm3/torch/model.py）。

仓库随模型代际演进了三代，同一仓库承载：
- **TimesFM 1.0/2.0**（2024）：200M/500M 检查点，JAX（PAX）+ PyTorch 双实现，代码整体归档在 `v1/`（README.md "Archived Model Versions"；v1/src/timesfm/patched_decoder.py）。
- **TimesFM 2.5**（2025-09）：缩回 200M，上下文拉长到 16k（16384，src/timesfm/timesfm_2p5/timesfm_2p5_base.py:90），可选 30M 连续分位数头，去掉 frequency 指示；torch + flax 双后端（src/timesfm/）。
- **TimesFM 3.0**（2026-08）：原生多变量预测 + 协变量支持（past-only 与 past-and-future），宣称在 fev-bench、TIME、GIFT-Eval 三大基准均排名第一（README.md "Update — August 2026"）。

商业模式上很有代表性：源码 Apache-2.0，3.0 权重改为非商业许可，商业/生产使用引导到 Google Cloud（BigQuery ML、Sheets、Vertex Model Garden，README.md 第 15-23、61-66 行）。仓库还内置了面向 AI agent 的一等公民支持：AGENTS.md + timesfm-forecasting/SKILL.md（Agent Skill 标准）。

## 3. 技术栈与依赖

- 语言：纯 Python（requires-python >=3.10，pyproject.toml）。
- 核心依赖极薄：仅 numpy、huggingface_hub、safetensors（pyproject.toml dependencies）——即"推理所需的框架后端全部做成可选 extra"。
- 可选后端 extras（pyproject.toml [project.optional-dependencies]）：
  - `torch`（torch>=2.0）：TimesFM 3.0 主后端、2.5 后端之一；
  - `mlx`（mlx>=0.32）：Apple Silicon 原生后端，无需 PyTorch；
  - `flax` / `flax-cuda12`：2.5 的 Flax/JAX 后端（optax、einshape、orbax-checkpoint、jaxtyping）；
  - `xreg` / `xreg-cuda12`：协变量回归（jax + scikit-learn，src/timesfm/utils/xreg_lib.py）。
- 构建：setuptools（新版）；v1 归档目录用 poetry（v1/pyproject.toml，version 1.3.0）。
- 锁定文件：requirements.txt 由 `uv pip compile` 生成。
- 模型分发：Hugging Face Hub `PyTorchModelHubMixin`（src/timesfm3/torch/model.py:30,37），仓库内不含权重。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `src/timesfm3/` | TimesFM 3.0（当前主版本）：共享基类 + `torch/`（主后端）+ `mlx/`（Apple Silicon） |
| `src/timesfm/` | TimesFM 2.5：`timesfm_2p5/`（base/flax/torch 三件套）+ `flax/`、`torch/` 公共层 + `utils/xreg_lib.py` 协变量 |
| `v1/` | 归档的 1.0/2.0 代码：PAX/JAX patched_decoder、PyTorch 移植、LoRA/DoRA 微调、时间基准实验 |
| `tests/` | 顶层 pytest 测试（21 个文件，主要覆盖 2.5 与公共逻辑） |
| `timesfm-forecasting/` | 一方 Agent Skill（SKILL.md）+ 示例（异常检测/协变量/微调/全球气温）+ references 文档 |
| `timesfm3-usage/` | 3.0 基准复现：fev_bench、gift_eval、time_bench + notebooks |
| `.github/workflows/` | main.yml（构建检查）、manual_publish.yml（人工触发发 PyPI） |
| AGENTS.md | agent 开发入口：源码位置说明 + 测试命令 |

组织评述：典型"多代模型共仓"结构——当前版本在 `src/` 布局的标准包里，历史版本整体冻结在 `v1/` 不再维护，代际之间通过 pip 版本区分（`pip install timesfm==1.3.0` 可装旧版，README.md 第 30-32 行）。两个顶层包 `timesfm`（2.5）与 `timesfm3`（3.0）并存，且 `timesfm/__init__.py` 会再导出 `timesfm3` 的符号（src/timesfm/__init__.py:40-44）。注意：当前源码中 PyTorch 支持是单包的可选 extra（`pip install timesfm[torch]`），仓库内没有独立的 timesfm-torch 包定义；早期 PyPI 时代是否拆分过独立 `timesfm-torch` 分发，本快照中未见证据（未确认）。

## 5. 核心架构与关键实现

### 5.1 TimesFM 3.0 模型（PyTorch 主后端）

`src/timesfm3/torch/model.py`（649 行）定义 `TimesFM3Torch`（继承 nn.Module + PyTorchModelHubMixin，仅推理）：patch 化 decoder-only 架构，`input_patch_len=32`、`output_patch_len=64`，默认 20 层、model_dims=1280、16 头、RMSNorm、序列维 RoPE、relu FFN（model.py:59-106；README 推算总参数约 330M）。输入 patch 先过 ResidualBlock 升维，输出头是 `Linear(1280, output_patch_len * num_quantiles)`，一次出 64 步 × 9 分位数（model.py:146-159）。多变量通过形状 `(b, v, n, d)` 的"混合 Transformer"处理：

```python
class MixingTransformer(nn.Module):
  """Transformer with sequential sequence and variate attention."""
  # src/timesfm3/torch/transformer.py:420-420
```

即每层先做序列内因果注意力、再做跨变量（variate）注意力、最后 FFN（transformer.py:420-571），单变量时走快速路径跳过 variate attention（transformer.py:541-546）。

### 5.2 推理管线： stitching + RevIN 精炼

`src/timesfm3/torch/timesfm3_forecaster.py`（825 行）是对外的 `TimesFM3Forecaster`/`TimesFM3Evaluator` 封装，默认检查点 `google/timesfm-3.0-pytorch`，全局上下文上限 15360（timesfm3_forecaster.py:29）。长预测通过 stitching 拼接多个输出 patch（model.py:141-144），两个特色实现：
- **迭代 CPM-RevIN 精炼**（`src/timesfm3/torch/cpm_revin_refine.py`）：对被 CPM 掩码（预测区间）的 patch，用模型自身已估出的值迭代更新 RevIN 归一化统计量，缓解长程分布漂移（cpm_revin_refine.py:29-45）；
- **翻转不变性**：`force_flip_invariance` 保证 TimesFM(aX+b)=a·TimesFM(x)+b 对 a<0 也成立（src/timesfm/configs.py:46-48）。

MLX 后端（`src/timesfm3/mlx/`）逐文件镜像 torch 结构（model/transformer/dense/normalization…），并有数值一致性测试 `mlx/model_parity_test.py`（README 给出 1e-6 量级的对齐误差）。

### 5.3 可选后端的懒加载

两个包的 `__init__.py` 都用 PEP 562 `__getattr__` 延迟导入后端，缺 torch/flax 时给出带安装提示的 AttributeError 而非裸 ImportError（src/timesfm/__init__.py:54-77，引用了 issue #318/#287）。这使 `import timesfm3` 或 `import timesfm3.mlx` 完全不依赖 PyTorch（src/timesfm3/__init__.py:15-21）。

### 5.4 协变量（XReg）与 2.5 的 ForecastConfig

协变量走"上下文内线性回归"路线：`src/timesfm/utils/xreg_lib.py` 用 JAX + sklearn 对上下文窗口做带类别编码的线性回归，再把残差交给模型，模式有 `timesfm + xreg` 与 `xreg + timesfm` 两种（xreg_lib.py:23-27）。2.5 的 `ForecastConfig`（src/timesfm/configs.py:26-69）集中暴露了 normalize/quantile head/flip invariance/nonnegative/quantile crossing 等推理开关，是理解 2.5 行为的钥匙。

## 6. 构建与运行

```bash
# 安装（PyPI）
pip install timesfm[torch]     # PyTorch 后端
pip install timesfm[mlx]       # Apple Silicon, MLX 后端
pip install timesfm[flax]      # 2.5 Flax 后端（CPU 可用）

# 本地开发（README.md Install 节）
uv venv && source .venv/bin/activate
uv pip install -e .[torch]

# 测试（AGENTS.md）
pip install -e ".[torch]" pytest
pytest src/timesfm3/                                # 3.0；MLX 测试缺 mlx 时自动跳过
pytest tests/ --ignore=tests/test_model_loading.py  # 2.5；model_loading 需 flax extra

# 最小用法（README.md）
from timesfm3 import TimesFM3Evaluator, ModelConfig
config = ModelConfig(checkpoint_path="google/timesfm-3.0-pytorch", device="cuda")
forecaster = TimesFM3Evaluator(config)
outputs = forecaster.predict_batch([...], horizon=12, return_quantiles=True)
```

CI（.github/workflows/main.yml）只做包构建检查（Python 3.11 + uv + `python -m build`），不在 CI 里跑测试；PyPI 发布走 manual_publish.yml 人工触发，经 Google 内部 "OSS Exit Gate"（GCP Artifact Registry 暂存）后再上 PyPI。

## 7. 测试与工程质量

- 测试规模：顶层 `tests/` 21 个文件 + `src/timesfm3/` 内嵌 `*_test.py`（torch/evaluator/transformer/primitives/parity 等），合计约 240 个测试类/函数（grep 统计）；另有 v1/tests/ 覆盖归档代码。
- 风格：pytest，单元级为主——配置不可变性与默认值防漂移（tests/test_configs.py 文档字符串明说是防"silent default-value drift"）、层等价性（tests/test_torch_layers.py）、数值路径（输出 patch 聚合、stitching、归一化统计分配）、后端缺失时的错误行为（tests/test_optional_backend_errors.py）。MLX 与 torch 的数值对齐测试（src/timesfm3/mlx/model_parity_test.py，435 行）少见且扎实。
- 质量观察：类型注解完整（py.typed）、ruff 约束（line-length 88、indent-width 2，pyproject.toml:60-62）、错误信息带安装指引、引用 issue 号回溯。短板：CI 不执行测试（仅构建），代码内测试需开发者本地跑；中文/英文注释混杂度低但部分 TODO 未清（src/timesfm/configs.py:41 `TODO(siriuz42):implement it`）。

## 8. 社区与维护状态

- 维护方：google-research org，核心作者即论文团队——pyproject.toml 列名 Rajat Sen、Yichen Zhou、Abhimanyu Das、Petros Mol、Ayush Jain、Michael Chertushkin（全 google.com 邮箱）；贡献榜第一 rajatsen91（151 commits）、第二 siriuz42（78，与 configs.py TODO 署名对应，即 Yichen Zhou），共 50 名贡献者（gh api contributors）。
- 发版节奏（gh api releases）：v1.2.1（2024-10）→ v1.2.6（2024-12）→ 长期沉寂 → v2.0.1（2026-06）→ v2.0.2（2026-07）→ v3.0.0（2026-08）→ 源码版本 3.0.2。呈"随模型代际脉冲式发版"模式，2025 年下半年至 2026 年为高强度活跃期，最近 10 个 commit 全部落在 2026-11-13。
- 社区参与：社区贡献真实融入——README 致谢 @kashif（HF transformers 维护者）、@darkpowerxo、@borealBytes；Agent Skill 与部分单测由社区作者署名（timesfm-forecasting/SKILL.md author: Clayton Young @borealBytes）。open issues 247（gh api），作为 35k 星的研究仓库属正常水位。
- 未确认：Google 内部投入人力规模、issue 平均响应时长。

## 9. 亮点与风险

**亮点**
1. 架构演进清晰可追溯：v1 → 2.5 → 3.0 三代代码同仓分层归档，每代都有对应论文/博客，是读"时序基础模型如何落地"的完整标本。
2. 多后端工程化扎实：torch/flax/mlx 三后端，MLX 与 torch 数值对齐到 1e-6 量级并有 parity 测试；可选依赖懒加载处理周到（带安装提示、引用 issue）。
3. 基准可复现：fev_bench/GIFT-Eval/TIME 三套评测的 runner 与结果 CSV 全部随仓公开（timesfm3-usage/benchmarks/）。
4. 对 agent 生态有一等支持：AGENTS.md + 可直接安装的 Agent Skill + 预检脚本，属 2026 年研究仓库的先进行列。
5. 基座依赖极轻（numpy/hf_hub/safetensors），装包即用。

**风险/局限**
1. 许可证分裂：3.0 权重非商业许可，自托管商用需走 Google Cloud 付费通道，升级前务必核对（README.md License notice）。
2. CI 只构建不跑测试，回归风险靠本地自律；测试虽多但未进流水线。
3. 多代共仓且 v1 用 poetry、新版用 setuptools/uv，构建体系割裂，旧目录文档（v1/README、TROUBLESHOOTING）可能误导新用户。
4. "Rank #1" 基准结论均出自官方 runner 与官方博客，独立第三方复现需自行验证。

## 10. 总结评价

TimesFM 是时间序列基础模型方向工程完成度最高的开源代表之一：Google Research 论文团队亲自维护、三代模型代码完整沉淀、三后端（torch/flax/mlx）覆盖从服务器到 Apple Silicon、基准 runner 全公开，且已进入 BigQuery ML/Sheets 等 Google 一方产品。它不是最好的"学习 PAX/JAX 建模"的教材（那是 v1/ 的历史遗产），而是了解"基础模型如何做推理工程与产品化"的一手材料。适合：需要零样本预测能力的工程团队（直接 pip 装 2.5/3.0）、时序基础模型研究者（读 MixingTransformer/CPM-RevIN/stitching 实现）、以及想给自家模型配 agent skill 的维护者（抄 timesfm-forecasting/ 作业）。商用前务必确认 3.0 权重许可边界。
