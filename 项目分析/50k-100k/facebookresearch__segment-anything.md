# facebookresearch/segment-anything 项目分析

> 星数 55051（快照 2026-10-24）｜主语言 Jupyter Notebook｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

Meta AI Research（FAIR）发布的 **Segment Anything Model（SAM）**：提示式图像分割基础模型，输入点/框/粗掩码等提示即可高零样本质量输出物体掩码，也能自动为整图生成所有物体的掩码。2023 年 4 月发布即成为视觉领域"ChatGPT 时刻"级事件，是分割任务的事实标准底座模型。

| 项目 | 值 | 出处 |
| --- | --- | --- |
| 名称 | segment-anything | README.md |
| 星数 / Fork | 55051 / ~6.4k | gh api 快照（55095 @ 2026-11-03） |
| 主语言 | Jupyter Notebook（示例）+ Python（核心包） | 仓库语言统计 |
| 许可证 | Apache 2.0（代码）；SA-1B 数据集另有研究许可 | LICENSE、README「License/Dataset」 |
| 创建 / 最近推送 | 2023-03-23 / **2024-09-18** | gh api repos/facebookresearch/segment-anything |
| 版本管理 | 无 git tag、无 GitHub Release，pip 安装直接走 main 分支 | gh api（releases=0, tags=0） |
| 仓库形态 | 小型研究仓：核心包约 1200 行 Python + 3 个示例 notebook + 1 个 React 网页 demo | 目录清单 |

## 2. 项目是什么，解决什么问题

SAM 解决的核心问题是**可提示的通用分割（promptable segmentation）**：传统分割模型绑定特定类别与数据集，而 SAM 训练在 1100 万张图、11 亿掩码的 SA-1B 数据集上（README「Segment Anything」节），目标是给定任意提示——前/背景点、框、粗掩码——就分割出"任何物体"，且在未见过的任务和域上有强零样本表现。项目论文为 arXiv:2304.02643（README BibTeX 节）。

工程上它刻意做了"**重编码器 + 轻解码器**"的解耦：重型 ViT 图像编码器只对每张图跑一次，之后任意次提示交互都只经过轻量的提示编码器与掩码解码器，因此交互式标注（点几下就出 mask）可以做到实时。这一设计也是它能导出 ONNX 跑进浏览器的前提（README「ONNX Export」节）。

模型输入输出约定清晰：图像统一缩放到最长边 1024、补边成方形、按 ImageNet 均值方差归一化（`modeling/sam.py` 的 `preprocess`）；输出除二值掩码外还带 `iou_predictions`（模型自估质量分）和 256×256 的 `low_res_logits`——后者可以回喂给下一次预测作为 mask 提示，形成迭代式精修闭环（`predictor.py` 的 `predict`/`predict_torch` 文档注释）。

仓库提供三层使用方式：`SamPredictor`（交互式提示预测）、`SamAutomaticMaskGenerator`（整图自动生成全部掩码）、以及命令行脚本与浏览器 demo。三个规模的 checkpoint（ViT-B/L/H）托管在 dl.fbaipublicfiles.com，注册表里 `default` 即 ViT-H（`segment_anything/build_sam.py`）。

值得注意的是，README 顶部第一屏已经是 **SAM 2**（segment-anything-2，2024-08 发布，扩展到视频）的宣传位，本仓库实质进入维护期；最后一次推送为 2024-09-18（gh api `pushed_at`），此后无更新，属"被自家后继者接班"的停滞。

## 3. 技术栈与依赖

- **语言**：Python（核心包 + 脚本）、TypeScript/React（demo）、Jupyter Notebook（示例，也是 GitHub 语言统计的主语言）。
- **核心依赖**：PyTorch（`pytorch>=1.7`）、torchvision（README「Installation」节）；`setup.py` 的 `install_requires=[]`，几乎零硬依赖。
- **可选依赖**（`setup.py` extras）：`opencv-python`、`pycocotools`（COCO RLE）、`matplotlib`（可视化）、`onnx` + `onnxruntime`（模型导出）。
- **demo 端**：`onnxruntime-web`（WASM 多线程 + SIMD）、`npyjs`（加载 .npy 图像嵌入）、React 18 + webpack + tailwind（`demo/package.json`）。
- **代码质量工具**：black==23、isort==5.12、flake8、mypy（`linter.sh`、`setup.cfg`、`.flake8`）。
- 模型骨干来自 detectron2 的 ViTDet 与 mvitv2 的分解式相对位置编码（`segment_anything/modeling/image_encoder.py` 头部注释与 `add_decomposed_rel_pos` 引用）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `segment_anything/` | 核心 Python 包，全部模型与推理逻辑 |
| `segment_anything/modeling/` | 模型定义：image_encoder / prompt_encoder / mask_decoder / transformer / sam |
| `segment_anything/utils/` | amg（自动掩码生成工具函数）、onnx（导出包装模型）、transforms（ResizeLongestSide） |
| `scripts/` | `amg.py`（整图自动分割 CLI）、`export_onnx_model.py`（ONNX 导出 CLI） |
| `notebooks/` | 3 个官方示例 notebook（提示预测 / 自动画 mask / ONNX） |
| `demo/` | 纯前端 React demo：浏览器里跑 ONNX 掩码解码（`demo/README.md`） |
| `assets/` | README 与 demo 用的图片素材 |
| `setup.py` / `setup.cfg` / `.flake8` / `linter.sh` | 打包与 lint 配置 |

组织方式是典型的 **FAIR 研究仓单包结构**：一个可 pip 安装的扁平包（`packages=find_packages(exclude="notebooks")`，`setup.py`），modeling 层只依赖 PyTorch，numpy/CV 类工具隔离在 utils，脚本与 notebook 只做入口。没有配置系统、没有训练代码、没有数据引擎代码——论文中的数据引擎与训练并不在本仓库开源，这是一个纯推理/应用仓库。

`segment_anything/` 内部文件一览（行数为 wc -l 统计）：

| 文件 | 行数 | 内容 |
| --- | --- | --- |
| `modeling/image_encoder.py` | 396 | ViTDet 式 ViT、窗口/全局注意力、相对位置编码、PatchEmbed |
| `utils/amg.py` | ~440 | MaskData 容器、RLE 编解码、稳定分、裁剪与去重工具 |
| `modeling/transformer.py` | 241 | TwoWayTransformer 及其注意力块 |
| `predictor.py` | 269 | SamPredictor：set_image 缓存嵌入 + predict 交互预测 |
| `automatic_mask_generator.py` | 372 | 整图自动掩码生成与过滤流水线 |
| `modeling/prompt_encoder.py` | 215 | 点/框/掩码提示编码 + 随机傅里叶位置编码 |
| `modeling/mask_decoder.py` | 177 | 双向 transformer 解码、动态卷积出掩码、IoU 头 |
| `modeling/sam.py` | 175 | 顶层容器、预处理归一化补边、掩码上采样后处理 |
| `build_sam.py` | 107 | ViT-B/L/H 三个规模的超参注册表 |
| `utils/onnx.py` | 145 | ONNX 导出专用包装模型（无控制流化） |
| `utils/transforms.py` | ~85 | ResizeLongestSide 图像/坐标/框变换 |
| `modeling/common.py` | ~55 | LayerNorm2d、MLPBlock 基础件 |

## 5. 核心架构与关键实现

SAM 由三大组件构成（`segment_anything/modeling/sam.py` 的 `Sam` 类聚合三者为成员）：

**（1）图像编码器 ImageEncoderViT（`segment_anything/modeling/image_encoder.py`）**
从 ViTDet 轻改而来（文件头注释注明出处）。输入 1024×1024、patch 16，得到 64×64 token 网格；关键工程点是 **MAE 式窗口注意力**：绝大多数 block 只做 14×14 窗口内注意力，仅少数 block（由 `global_attn_indexes` 指定，ViT-H 为第 7/15/23/31 层，见 `build_sam.py`）做全局注意力，大幅降低 1024 分辨率下 self-attention 的开销；配合分解式相对位置编码（`add_decomposed_rel_pos`，einsum 实现来自 mvitv2）。末端 `neck` 用 1×1 + 3×3 卷积把 embed_dim 压到 256 通道（`image_encoder.py` L88-104），供解码器消费。

**（2）提示编码器 PromptEncoder（`segment_anything/modeling/prompt_encoder.py`）**
点/框走稀疏路径：坐标先经随机傅里叶位置编码 `PositionEmbeddingRandom`（L171-192，固定高斯矩阵 + sin/cos），再按标签加上 4 个可学习 embedding（正点/负点/框两角，L45-47）；掩码走稠密路径：3 层卷积下采样 4 倍到 embedding 尺寸（L50-59）；无掩码输入时用 `no_mask_embed` 广播填充（L163-166）。设计目标是提示与图像嵌入在同一 256 维空间对齐。

**（3）掩码解码器 MaskDecoder + TwoWayTransformer（`modeling/mask_decoder.py`、`modeling/transformer.py`）**
解码器只有 2 层双向 transformer（8 头、mlp 2048）：每个 `TwoWayAttentionBlock` 依次做 token 自注意力 → token→图像交叉注意力 → token MLP → 图像→token 交叉注意力（`transformer.py` L151-182），让提示 token 与图像嵌入互相更新。输出侧的设计最巧：4 个 mask token（1 个歧义单掩码 + 3 个多掩码档位，`mask_decoder.py` L50-51）经超网络 MLP 生成 256×1 的动态卷积核，与上采样后的图像嵌入做逐点乘（L139-144 `hyper_in @ upscaled_embedding`），即**用 token 动态生成卷积权重来"画"出掩码**；同时 iou_token 经 MLP 预测每个掩码的质量分供挑选（L67-69、L147）。

**（4）自动掩码生成 SamAutomaticMaskGenerator（`segment_anything/automatic_mask_generator.py`、`utils/amg.py`）**
"分割一切"模式不需要用户提示：在 32×32 的均匀点网格上批量采样提示（`points_per_side=32`），每个点输出 3 档掩码，再经过一套纯张量流水线过滤：预测 IoU 阈值 0.88、稳定分阈值 0.95（对 logits 加 ±1 扰动后阈值化，比较前后面积变化率，`utils/amg.py` 的 `calculate_stability_score`）、去重 NMS 0.7、可选多级裁剪（`crop_n_layers`）提升小物体召回、最后用 `remove_small_regions` 修补空洞并支持输出 COCO RLE（`automatic_mask_generator.py` L24-45 默认参数）。支撑它的 `utils/amg.py`（约 440 行）是仓库里最长的文件，实现了 `MaskData` 批处理容器与一批 RLE/框运算工具——这是全仓唯一"研究之外"的工程密集区。

**（5）ONNX 导出与浏览器推理（`segment_anything/utils/onnx.py`、`scripts/export_onnx_model.py`）**
`SamOnnxModel` 把提示编码器 + 掩码解码器 + 后处理打包为可 trace 的模型，专门做了两件事以适配导出：把分支逻辑改成无控制流的张量运算（如 `select_masks` 用 `score + (num_points-2.5)*reweight` 的加权技巧代替 if，`onnx.py` L92-105）；掩码插值的输出尺寸改为张量参数。导出脚本默认 opset 17，支持动态量化（QUInt8）、tanh 近似 GELU 等选项（`export_onnx_model.py` L52-84）。浏览器 demo 用 `onnxruntime-web` 的 WASM 多线程跑这个模型，图像嵌入则由 Python 预计算为 .npy 加载（`demo/README.md`）——即"重编码器离线算、轻解码器在线跑"的完整示范。

## 6. 构建与运行

安装（README「Installation」节）：

```bash
pip install git+https://github.com/facebookresearch/segment-anything.git
# 或本地：git clone ... && cd segment-anything && pip install -e .
pip install opencv-python pycocotools matplotlib onnxruntime onnx   # 可选依赖
```

使用（README「Getting Started」节）：

```python
from segment_anything import SamPredictor, sam_model_registry
sam = sam_model_registry["vit_h"](checkpoint="sam_vit_h_4b8939.pth")
predictor = SamPredictor(sam)
predictor.set_image(image)
masks, _, _ = predictor.predict(input_prompts)
```

命令行与其他：

```bash
python scripts/amg.py --checkpoint <ckpt> --model-type <type> --input <img> --output <dir>   # 自动画 mask
python scripts/export_onnx_model.py --checkpoint <ckpt> --model-type <type> --output <onnx>  # 导出 ONNX
cd demo && yarn && yarn start   # 浏览器 demo（localhost:8081，demo/README.md）
```

checkpoint：ViT-H `sam_vit_h_4b8939.pth`（default）、ViT-L `sam_vit_l_0b3195.pth`、ViT-B `sam_vit_b_01ec64.pth`（README「Model Checkpoints」节）。

## 7. 测试与工程质量

- **无测试目录、无任何测试文件**（全仓文件清单中不存在 tests/ 或 *_test.py；仅 demo/package.json 里的 `test` 脚本实际是启动模型调试服务）。质量保障完全依赖 lint：`linter.sh` 串行执行 isort、black（行宽 100）、flake8（max-complexity 18）、mypy，`CONTRIBUTING.md` 明确要求 PR 过 lint。
- **无 CI**：仓库根目录与 tarball 中均不存在 `.github/`（gh api contents 列表核实），意味着没有自动化测试/构建流水线，与"研究快照开源"的定位一致。
- 代码质量本身很高：全包约 1200 行 Python（wc -l 统计），每个模块都是"一个文件一个概念"，docstring 完整、类型标注齐全，mypy `# type: ignore` 用得克制；`modeling/` 与 `utils/` 的分层使得不装 torch 之外任何东西也能跑核心模型。

## 8. 社区与维护状态

- **活跃度**：创建于 2023-03-23，最后推送 **2024-09-18**（gh api `pushed_at`），距今约两年无提交，**项目已停滞**——这是事实记录而非缺陷：后继项目 segment-anything-2（README 顶部链接）接棒视频分割方向。
- **发布节奏**：无 git tag、无 GitHub Release（gh api releases=0, tags=0），版本号停在 `setup.py` 的 `1.0`，发布方式是"main 分支即发布"。
- **贡献者**：GitHub contributors 端点显示 17 人，核心为 Meta FAIR 团队（README 列出 12 位论文作者 + 26 位致谢贡献者）；另有 CLA 要求（CONTRIBUTING.md），典型企业研究仓模式。
- **社区热度**：约 6.4k fork、596 个 open issue（gh api），周边生态（MobileSAM、FastSAM、MedSAM、Grounded-SAM 等）大量派生——issue 多但官方响应随停滞基本停止（未确认具体响应时长）。

## 9. 亮点与风险

**亮点**
1. 教科书级的架构解耦：重 ViT 编码器一次性算嵌入、轻解码器支持任意次交互与 ONNX 导出，这一"基础模型 + 轻量适配"范式被后续无数视觉模型沿用（`modeling/sam.py`、`utils/onnx.py`）。
2. 极小的代码量承载完整能力：核心包约 1200 行、零硬依赖（`setup.py` install_requires=[]），可读性和可移植性极佳，是学习 ViTDet/窗口注意力/双向 transformer 解码器的最佳样本之一。
3. 从 Python 到浏览器的全链路：官方 ONNX 导出 + 动态量化 + React/WASM demo（`scripts/export_onnx_model.py`、`demo/`），端侧部署路线完整可复现。
4. 10 亿级掩码数据集 SA-1B 与模型同时开源（README「Dataset」节），催生了整个提示式分割研究生态。

**风险/局限**
1. **已停止维护**：两年无提交、无 CI、无测试、无版本发布，bug 只能靠社区 fork 修复；生产使用建议锁定 checkpoint 与 commit。
2. 无训练代码：数据引擎、模型训练、微调均未开源，想微调需自行实现或转用第三方（如 MobileSAM/fine-tune 项目）。
3. 重编码器成本高：ViT-H 在 1024 分辨率下编码开销大，实时视频/端侧场景需改用蒸馏变体；本仓库不含任何加速实现。
4. pip 直装 main 分支且依赖 torch 行为（如 ONNX 导出对 PyTorch 版本敏感，README 已提示"推荐最新稳定版 PyTorch"），未来环境漂移下复现性会逐渐变差。

## 10. 总结评价

SAM 是提示式分割范式的奠基之作，这个仓库用约 1200 行干净、零依赖、注释完备的 PyTorch 代码完整呈现了"重编码器 + 提示编码器 + 双向 transformer 掩码解码器"的架构，并配套 ONNX 导出与浏览器 demo 的端到端示范；它作为产品已经功成身退（由 SAM 2 接棒、仓库自 2024-09 起停滞），但作为**源码教材**的价值反而历久弥新。适合三类人：需要零样本分割能力的应用开发者（直接用 checkpoint + predictor）、想把分割模型部署到端侧的工程师（重点读 `utils/onnx.py` 与 demo）、以及想理解 ViT 变体与 query-based 解码器设计的研究者（精读 `modeling/` 四个文件，每个不超过 400 行）。读到它，也就读到了后续一大半分割基础模型的共同骨架。
