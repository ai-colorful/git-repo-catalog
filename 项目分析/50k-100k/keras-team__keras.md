# keras-team/keras 项目分析

> 星数 64378（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

Keras 3 是一个多后端深度学习框架：用一套统一的高级 API 编写模型，底层可运行在 JAX、TensorFlow、PyTorch 之上（另有 OpenVINO 纯推理后端及 mlx/paddle 等可插拔后端）。

| 项 | 值 |
|---|---|
| 名称 | keras-team/keras（PyPI 包名 `keras`） |
| 星数 | 64378（快照 2026-10-24） |
| 主语言 | Python（要求 >=3.11，见 `pyproject.toml`） |
| 许可证 | Apache-2.0（`LICENSE`） |
| 版本 | 3.16.0（`keras/src/version.py`；最近 GitHub release v3.15.1，2026-07-29） |
| 创建时间 | 2015-03-28（gh api） |
| 最近 push | 2026-10-30（gh api，分析前一天仍有多次提交） |
| 默认分支 | master |
| 代码量 | keras/src 非测试代码约 18.2 万行，测试代码约 12.9 万行（本地统计） |

## 2. 项目是什么，解决什么问题

Keras 是深度学习领域最早流行的高级建模 API（"Deep Learning for Humans"，`README.md`）。2015 年创建，Keras 2 时代作为 `tf.keras` 内置于 TensorFlow；2023 年发布的 Keras 3 把它改造成后端无关的独立框架：同一份建模代码可以选择 JAX（研究与 TPU 扩展性强）、TensorFlow（生产生态完整）或 PyTorch（训练生态主流）作为执行引擎，另支持 OpenVINO 仅做推理（`README.md` Backend Compatibility Table）。

它解决的核心问题是**框架锁定**：用户按 Keras API 写 `Layer`/`Model`/损失/指标，后端通过环境变量 `KERAS_BACKEND` 或 `~/.keras/keras.json` 在 import 前一次性确定，import 后不可切换（`README.md`"Configuring your backend"）。自定义组件（层、损失、指标、train_step）天然跨后端复用，Keras 模型还能嵌进原生的 PyTorch Module 或 JAX 函数中训练（`README.md`"Why use Keras 3?"）。

Keras 3 同时是一个"全功能训练库"：提供 `keras.ops` 函数命名空间（约 350+ 个算子，`keras/src/ops/numpy.py` 等 7 个命名空间共 358 个 `@keras_export`）、函数式/子类化两种建模方式、`fit/evaluate/predict` 训练循环与回调、数据适配层（可喂 numpy、`tf.data.Dataset`、torch `DataLoader` 等，`keras/src/trainers/data_adapters/`）、分布式训练（`keras/src/distribution/`）、模型保存（`.keras` 格式）与导出（ONNX/LiteRT/SavedModel，`keras/src/export/`）。

近版本还在向 LLM 时代的工程需求扩展：量化模块（GPTQ、AWQ、float8、校准，`keras/src/quantizers/`）、知识蒸馏（`keras/src/distillation/`）、Flax NNX 支持（`keras/src/backend/jax/layer.py` 中 `is_nnx_enabled()` 分支）。Keras 2 代码兼容由 `keras/src/legacy/` 与 `keras/api/_tf_keras/` 提供。

## 3. 技术栈与依赖

- **语言/构建**：Python（>=3.11），setuptools 打包；`pyproject.toml` 将 `keras/api` 重映射为包根、`keras/src` 为内部实现（`[tool.setuptools.package-dir]`）。
- **核心依赖**（`pyproject.toml` dependencies）：`numpy`（数值基础）、`absl-py`、`rich`（终端输出）、`namex`（延迟导入/API 生成）、`h5py`（H5 权重）、`optree`（树结构遍历）、`ml-dtypes`（bfloat16 等扩展 dtype）、`packaging`。
- **后端按需安装**：`tensorflow`/`jax`(+`flax`)/`torch`(+`torch-xla`)/`openvino`（`requirements.txt`）；GPU/TPU 有独立 requirements 文件（`requirements-{backend}-{cuda,tpu}.txt`）。
- **测试/质量**：pytest、pytest-cov、pytest-xdist、coverage；ruff（line-length 80）；pre-commit（`requirements-common.txt`、`pyproject.toml` `[tool.ruff]`）。

## 4. 目录结构与代码组织

顶层是经典的"单包、双层目录"结构：

| 目录 | 职责 |
|---|---|
| `keras/api/` | 自动生成的公共 API 门面（文件头标注"DO NOT EDIT"，由 `api_gen.py` + `shell/api_gen.sh` 生成），`keras/api/_tf_keras/` 为 tf.keras 兼容入口 |
| `keras/src/` | 全部真实实现（约 18.2 万行非测试代码），模块与源码同目录放 `*_test.py` |
| `keras/src/backend/` | 多后端抽象核心：`common/`（后端无关 Variable/KerasTensor/dtype/StatelessScope 等）+ `jax/`、`torch/`、`tensorflow/`、`numpy/` 四个内置后端目录 |
| `keras/src/ops/` | `keras.ops` 算子命名空间：numpy/math/nn/linalg/core/image/einops 七个模块 + `operation.py`（Operation 基类）、`function.py`（图容器） |
| `keras/src/layers/` | 内置层库，按 activations/attention/convolutional/core/merging/normalization/pooling/preprocessing/regularization/reshaping/rnn 分类 |
| `keras/src/models/` | `Model`、`Functional`、`Sequential`、模型克隆 |
| `keras/src/trainers/` | `BaseTrainer`（compile/fit/evaluate/predict）、EpochIterator、数据适配器 |
| `keras/src/optimizers|losses|metrics|callbacks|initializers|regularizers|constraints/` | 训练组件库 |
| `keras/src/saving/` | `.keras` 序列化格式（saving_lib/saving_api/serialization_lib） |
| `keras/src/applications/` | 预训练视觉模型（ResNet/VGG/EfficientNet/ConvNeXt 等，`applications/` 下 19 个文件） |
| `keras/src/quantizers/`、`distillation/`、`distribution/`、`export/`、`legacy/` | 量化、蒸馏、分布式、模型导出、Keras 2 兼容 |
| `integration_tests/`、`benchmarks/`、`examples/`、`shell/` | 跨后端集成测试、性能基准、示例脚本、维护脚本（api_gen.sh/format.sh） |

组织方式评述：这是"门面 + 实现"的严格两层——用户永远 import `keras.api` 的符号（`keras/__init__.py` 转发），内部代码走 `keras.src.*`；公共 API 共 1146 处 `@keras_export` 标注（全仓统计），由脚本统一生成 `keras/api`，避免手写门面漂移。

## 5. 核心架构与关键实现

### 5.1 多后端调度：一处 if-elif，两套角色

后端在 import 时一次性决定，`keras/src/backend/__init__.py` 直接做命名空间级 dispatch（内置后端 `from keras.src.backend.<b> import *`），可插拔后端（openvino/mlx/paddle，`backend/config.py` 的 `PLUGGABLE_BACKENDS`）则按 `keras_<backend>` 包名动态 import（`keras/src/utils/module_utils.py` 的 `get_pluggable_backend_module()`）。四个内置后端各约 6500~12700 行实现（本地统计：tensorflow 12638、torch 10777、jax 8856、numpy 6499 行）。

后端目录内再细分为同类角色文件：`layer.py`（层基类混入）、`trainer.py`（训练循环）、`ops/`（算子实现）、`distribution_lib.py`。例如 torch 后端用 `TorchLayer(torch.nn.Module)` 把 Keras Variable 映射进 `torch.nn.ParameterDict` 以打通 PyTorch 原生训练（`keras/src/backend/torch/layer.py`）。

### 5.2 算子体系：Operation = call + compute_output_spec

`keras.ops` 的每个函数都是 `Operation` 子类的实例：`call()` 委托给当前后端的同名算子，`compute_output_spec()` 用符号张量做形状/dtype 推断。这是"符号建图"与"即时执行"统一的枢纽：

```python
# keras/src/ops/operation.py
def __call__(self, *args, **kwargs):
    if any_symbolic_tensors(args, kwargs):
        return self.symbolic_call(*args, **kwargs)   # 建图：Node 挂到 _inbound_nodes
    return self._dispatch_call(*args, **kwargs)      # 真执行（可包 quantized_call/remat）
```

`symbolic_call` 里 `Node(operation=self, call_args, call_kwargs, outputs)` 自动接线输入输出，把函数式模型串成 DAG（`keras/src/ops/operation.py`、`keras/src/ops/node.py`）。典型算子如 `Rot90`（`keras/src/ops/numpy.py`）：`call` 一行委托 `backend.ops.numpy.rot90(...)`，`compute_output_spec` 校验维度并算输出形状。

### 5.3 Layer/Model 类层次

类层次是理解 Keras 3 的钥匙（各文件行数本地统计）：

- `Operation`（`keras/src/ops/operation.py`，449 行）——统一调用协议（符号/实调/量化/remat 分派）。
- `Layer(BackendLayer, Operation)`（`keras/src/layers/layer.py`，2403 行）——在 Operation 上叠加 `add_weight()`/`build()` 延迟构状态、autocast、变量跟踪、量化策略；`BackendLayer` 来自后端目录（torch 下是 `TorchLayer`，jax 下 NNX 模式是 `nnx.Module`）。
- `Model(Trainer, Layer)`（`keras/src/models/model.py`，1308 行）——`Trainer` 按后端选择（`keras/src/backend/tensorflow|torch|jax/trainer.py`）。
- `Functional(Function, Model)`（`keras/src/models/functional.py`，1001 行）——函数式图模型，`Function`（`keras/src/ops/function.py`）是持有 Node DAG 的图容器。

`Layer` 通过 `build(input_shape)` 在首次调用时按输入形状创建权重（`keras/src/layers/core/dense.py` 的 `Dense.build` 中 `self.add_weight(name="kernel", ...)`），配合 `KerasTensor`（`keras/src/backend/common/keras_tensor.py`）实现"无数据先建模型"。

### 5.4 训练循环与数据适配

`BaseTrainer`（`keras/src/trainers/trainer.py`，1250 行）定义 `compile/fit/evaluate/predict` 与 loss/metrics 组装（`compile_utils.py`）；每个后端的 `Trainer` 实现自己的 `train_step`/`fit`——jax 版用 `jax.jit`（或 flax NNX 的 `nnx.jit`，见 `keras/src/backend/jax/trainer.py` 头部），torch 版走 autograd，tensorflow 版走 `tf.GradientTape`。数据侧 `data_adapters/`（array/generator/py_dataset/tf_dataset/torch_dataloader/grain 六种适配器）把任意数据源统一为 epoch 迭代协议（`epoch_iterator.py`）。

### 5.5 保存与导出

`keras/src/saving/saving_lib.py` 实现基于 zip+JSON 的 `.keras` 格式（权重走 H5 或 zip 内嵌），`export/` 提供 ONNX（onnxruntime/onnxscript 验证，见 `requirements-common.txt` 注释）、LiteRT、SavedModel、OpenVINO、torch.export 多目标导出（`keras/src/export/` 下 onnx.py/litert.py/torch.py 等）。

## 6. 构建与运行

来自 `README.md` 与 `CONTRIBUTING.md` 的实际命令（本次未执行，仅静态核对）：

```bash
pip install keras --upgrade                 # 用户安装
pip install -r requirements.txt             # 开发依赖（CPU 版 TF/JAX/Torch/OpenVINO）
python pip_build.py --install               # 本地源码安装（根目录执行）
./shell/api_gen.sh                          # 更新公共 API 后重新生成 keras/api

# 测试（pytest，配置在 pyproject.toml [tool.pytest.ini_options]）
pytest keras/src/losses/losses_test.py                              # 单文件
SKIP_APPLICATIONS_TESTS=True pytest keras                            # 全量（跳过重型应用测试）
KERAS_BACKEND=jax SKIP_APPLICATIONS_TESTS=True pytest keras          # 指定后端跑全量
KERAS_USE_BACKEND_AGNOSTIC_OPS=1 pytest keras/src/ops/numpy_test.py  # 后端无关算子路径
```

## 7. 测试与工程质量

- **规模**：294 个 `*_test.py`、约 12.9 万行测试代码，与被测模块同目录放置；另有 `integration_tests/`（11 个跨后端端到端脚本：numerical_test、tf/torch 自定义 fit、分布式训练、export 等）与 `benchmarks/`（layer/model/torch_ctl 三类基准）。测试/源码行比约 0.7:1。
- **CI**：GitHub Actions 覆盖 CPU/GPU/TPU 三档硬件与 nightly（`.github/workflows/` 下 cpu_tests.yml、gpu_tests.yml、tpu_tests.yml、nightly.yml），并有 publish-to-pypi.yml（发布）、scorecard.yml/zizmor.yml（安全加固）、issue 自动分诊（gemini-automated-issue-triage.yml）。CI 按 `workflows/config/{jax,torch,tensorflow,numpy}/keras.json` 组合各后端配置矩阵（目录实测存在）。
- **质量机制**：pytest `filterwarnings=error`（警告即失败，`pyproject.toml`）；ruff 检查 + pre-commit；codecov.yml 接入覆盖率；公共 API 全部经 `@keras_export` 生成，保证门面一致性。

## 8. 社区与维护状态

- **活跃度**：极高。gh api 显示最近 push 2026-10-30，最近 5 次提交集中在同一天（如 "Support remote paths in ModelCheckpoint #24092"、"Blackman #24107"），PR 编号已到 24000+。
- **发布节奏**：PyPI/GitHub release 保持月级节奏（v3.14.1 2026-05、v3.15.0 2026-06、v3.15.1 2026-07；master `version.py` 已是 3.16.0，说明 master 领先于最近一次 GitHub release）。
- **贡献者**：contributors 分页末页 1597（per_page=1，gh api Link 头），即约 1600 名贡献者；Keras 团队（Google）主导，`pyproject.toml` 作者邮箱为 keras-users@googlegroups.com。
- **社区治理**：`CONTRIBUTING.md`（1.3 万字）规定 PR 流程与贡献条款（pr-contributor-terms.yml 强制勾选）；issue 由 Gemini 自动分诊工作流处理（`.github/workflows/gemini-automated-issue-triage.yml`）。
- 开放 issue 约 271（gh api，快照日）。

## 9. 亮点与风险

**亮点**

1. **多后端抽象的教科书实现**：`Operation`/`KerasTensor`/`compute_output_spec` 三件套把"符号建图"从 TF 专属能力抽象为后端无关层（`keras/src/ops/operation.py`、`backend/common/`），一套 `keras.ops` API 下挂四个后端实现，且支持 openvino/mlx/paddle 外挂。
2. **工程化成熟度罕见**：12.9 万行测试、CPU/GPU/TPU 三档 CI、警告即失败、自动生成的公共 API 门面（1146 个导出）、安全 scorecard——适合作为大型 Python 框架的工程范本。
3. **生态桥接做得实**：TorchLayer 继承 `torch.nn.Module`、JAX 侧支持 flax NNX、数据侧同时吃 `tf.data` 与 `DataLoader`，"嵌入原生框架训练"是可用特性而非口号（`backend/torch/layer.py`、`backend/jax/trainer.py`）。
4. **持续演进**：从 Keras 2 到 3 的迁移层（`legacy/`、`_tf_keras`）、LLM 时代的量化（GPTQ/AWQ/float8）与蒸馏模块均已落地（`keras/src/quantizers/`）。
5. **文档内嵌**：核心函数的 docstring 即教程（含 doctest 示例），`api_gen.py` 生成的 API 文档与源码同步。

**风险/局限**

1. **同算子多实现的一致性负担**：`keras.ops` 每个函数要在 TF/Torch/JAX/NumPy 四处各写一份（再加可插拔后端），数值一致性靠 `integration_tests/numerical_test.py` 兜底；x64 支持目前仅 jax/numpy/torch（`backend/config.py` 的 `_X64_BACKENDS` 注释"TODO: Add mlx and paddle"）。
2. **双层包结构的理解成本**：`keras/api`（生成代码）与 `keras/src`（实现）分离 + namex 延迟导入，新人读源码需先掌握 `@keras_export` 机制；直接使用 `keras.src.*` 无兼容性保证。
3. **平台支持面**：Windows 官方建议 WSL2（`README.md`），OpenVINO 后端仅支持 `model.predict()` 推理。
4. **依赖后端版本敏感**：后端最低版本要求较新（PyTorch >=2.6.0 等，`README.md`），TF/JAX/Torch 三方大版本对齐是长期维护成本（`requirements.txt` 中多处 pin 与注释可证）。

## 10. 总结评价

Keras 是深度学习高级 API 的事实标准之一，Keras 3 通过"统一算子命名空间 + 后端角色文件（layer/trainer/ops/distribution）+ 符号/即时双模式"这套干净的多后端架构，把 11 年积累的建模 API 平移到了 JAX/TF/Torch 三个引擎上，同时以极其规范的测试、CI 与 API 生成流程维持着月级发布节奏——6.4 万星与近 2 万 fork 与其工程质量是相称的。它适合：想快速建模而不关心底层框架的应用研究者、需要"一次编写多框架运行"的框架选型者，以及任何想学习"如何为大型数值库设计后端抽象层"的工程师（推荐阅读顺序：`keras/src/ops/operation.py` → `keras/src/backend/__init__.py` 与 `backend/common/` → `keras/src/layers/layer.py` → `keras/src/trainers/trainer.py` 与任一后端 `trainer.py`）。
