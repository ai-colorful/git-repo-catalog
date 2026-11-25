# lllyasviel/ControlNet 项目分析

> 星数 31000（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-21
>
> 注：本仓为论文《Adding Conditional Control to Text-to-Image Diffusion Models》（ICCV 2023, arXiv:2302.05543）官方实现，仓库快照对应 **ControlNet 1.0**。当前 gh api 实测星数 34,156、fork 3,008；主分支最后提交 2023-09-09，仓库 pushed_at 2024-02-25，未设 archived，属**事实停更的研究仓**（作者精力已转向 Fooocus 等后续项目，详见第 8 章）。

## 1. 项目速览

一句话定位：ControlNet 官方参考实现——通过"锁定副本 + 可训练副本 + 零卷积"三件套，给预训练扩散模型（Stable Diffusion）增加任意空间条件控制（边缘/深度/姿态/分割等）的小型研究代码库。

| 项目 | 内容 |
|---|---|
| 名称 | lllyasviel/ControlNet |
| 星数 | 31,000（快照）/ 34,156（2026-11-21 gh api 实测） |
| 语言 | Python（约 6.0 万行，509 个文件） |
| 许可证 | Apache-2.0（`LICENSE`） |
| 创建 / 最后提交 | 2023-02-01 / 2023-09-09（main 分支，gh api commits） |
| pushed_at | 2024-02-25（未 archived，但近两年无实质代码提交） |
| Release / Tag | 无（gh api tags、releases 均为空） |
| 贡献者 | 6 人（lllyasviel 183 commits，其余 5 人各 1 commit） |
| 配套论文 | ICCV 2023，作者 Lvmin Zhang / Anyi Rao / Maneesh Agrawala |

## 2. 项目是什么，解决什么问题

大模型微调的两难在于：在小数据集上全量微调会摧毁预训练模型的能力，而不动原模型又学不到新条件。ControlNet 的答案是"结构化复制"（`README.md` 开篇段落）：把扩散 UNet 的一部分权重复制成两份——"locked"（锁定）副本保留原模型能力，"trainable"（可训练）副本学习新条件；两副本之间用"零卷积"（weight 和 bias 都初始化为 0 的 1×1 卷积）连接。训练开始时零卷积输出恒为 0，ControlNet 对原模型完全不产生扰动，因此可以在小数据甚至个人设备上安全训练。

具体到 Stable Diffusion：把 SD UNet 的 12 个编码器 block + 1 个 middle block 复制成可训练副本，每个副本输出经零卷积后以"加法"注入原 UNet 对应层的 skip connection（`README.md` 中 "By repeating the above simple structure 14 times" 一节及架构图 `github_page/sd.png`）。控制信号本身（canny 边缘、深度图、人体骨架等）由一个 4 层小卷积网络编码后加进第一个编码器特征。作者强调该连接方式的算力效率：原 SD 编码器无需存梯度（`README.md` "The original SD encoder does not need to store gradients"）。

仓库同时解决了"工程闭环"问题：自带 7 类条件检测器（annotator/）、9 个预训练模型（权重放在 Hugging Face 外链）、11 个 Gradio 演示应用、一套"像训 pix2pix 一样简单"的训练教程（`docs/train.md`）与 toy 数据集 Fill50K。这使它既是论文 artifact，也是 2023 年可控生成生态的起点。

零卷积"梯度为零学不动"的质疑在 `docs/faq.md` 有数学回答：∂y/∂w = x ≠ 0，一次梯度下降后 w 即非零，零卷积会逐步变成普通卷积。

## 3. 技术栈与依赖

- **Python 3.8.5 + PyTorch 1.12.1 + CUDA 11.3**（`environment.yaml`），环境用 conda 管理，无 requirements.txt / pyproject.toml。
- **pytorch-lightning 1.5.0**：训练框架（`tutorial_train.py`）；**gradio 3.16.2**：全部演示 UI。
- **opencv-contrib-python**：图像处理与 Canny/M-LSD 等检测；**einops**：张量重排；**omegaconf**：YAML 配置实例化（`cldm/model.py` 的 `create_model`）。
- **transformers 4.19.2**：FrozenCLIPEmbedder 文本编码（`models/cldm_v15.yaml` 的 cond_stage_config）。
- 无 Lint/格式化/类型检查配置；依赖列表整体停留在 2023 年初版本，属学术仓快照式 pinned。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `cldm/` | 核心增量代码：ControlNet 架构（`cldm.py`）、魔改 DDIM 采样器（`ddim_hacked.py`）、monkey-patch 显存/CLIP 补丁（`hack.py`）、模型加载（`model.py`）、日志（`logger.py`） |
| `ldm/` | 从 CompVis latent-diffusion 整体拷入的底座代码（ddpm.py 1797 行、attention.py、autoencoder.py 等），未做包化 |
| `annotator/` | 7 类条件检测器：canny、hed、midas（深度）、mlsd（直线）、openpose（姿态）、uniformer（分割，内嵌整套 mmcv/mmseg 源码）、ckpts（空，权重走 HF 外链，仅有 `ckpts.txt` 占位） |
| `models/` | 仅 2 个 YAML 配置（`cldm_v15.yaml`、`cldm_v21.yaml`），模型权重不在仓内 |
| `gradio_*.py`（11 个） | 9 个"条件→图像"演示 + 交互式涂鸦版 + `gradio_annotator.py` 自动标注工具，均为独立可运行脚本 |
| `tool_add_control.py` 等 | 把任意 SD checkpoint 转成 ControlNet 初始权重的工具（另有 sd21 版与 `tool_transfer_control.py`） |
| `tutorial_train.py` / `tutorial_dataset.py` | 训练入口与 Dataset 示例（各约 30/40 行） |
| `docs/` | train.md（训练教程）、faq.md（零卷积原理）、low_vram.md、annotator.md |
| `github_page/`、`test_imgs/` | README 配图（52 个文件）与 24 张测试图 |

组织方式评述：典型"论文 artifact 单体"——核心增量（cldm/，不足千行）极小，其余是大段拷贝的底座（ldm/、annotator/uniformer 内嵌 mmcv/mmseg）和复制粘贴式的 per-condition Gradio 脚本（11 个脚本结构雷同，仅检测器与 UI 参数不同）。没有包化、没有 CLI、没有插件抽象，一切以"可读、可改、可复现论文"为目标。任务侧提到的 `train.py`、`gradio_pabeltoimage`、`training/` 目录在本快照中不存在——实际训练入口是 `tutorial_train.py` 系列，`training/fill50k` 数据集从 Hugging Face 下载后按 `docs/train.md` 约定路径放置。

## 5. 核心架构与关键实现

### 5.1 ControlNet 模块与零卷积（`cldm/cldm.py`）

`ControlNet(nn.Module)`（cldm/cldm.py:48）是可训练副本。其 `self.input_blocks` 按 UNet 编码器同构搭建（channel_mult `[1,2,4,4]`、ResBlock+SpatialTransformer+Downsample，见 `models/cldm_v15.yaml`），每个 block 后跟一个零卷积；条件图经 `input_hint_block`（cldm/cldm.py:147-163，16→16→32→32→96→96→256→model_channels 的 4 层卷积，末端也是零卷积）编码成 `guided_hint`，加到第一个编码器特征上：

```python
def forward(self, x, hint, timesteps, context, **kwargs):
    ...
    guided_hint = self.input_hint_block(hint, emb, context)
    for module, zero_conv in zip(self.input_blocks, self.zero_convs):
        h = module(h, emb, context)
        if guided_hint is not None:
            h += guided_hint
            guided_hint = None
        outs.append(zero_conv(h, emb, context))
    h = self.middle_block(h, emb, context)
    outs.append(self.middle_block_out(h, emb, context))
    return outs
```

零卷积本身是 `make_zero_conv`（cldm/cldm.py:281-282）：1×1 卷积套 `zero_module`；而 `zero_module` 在 `ldm/modules/diffusionmodules/util.py:177-183`，实现只有三行——`for p in module.parameters(): p.detach().zero_()`。整层 13 个输出（12 编码器 + 1 middle）经零卷积返回。

### 5.2 锁定副本与控制注入（`ControlledUnetModel` + `ControlLDM`）

`ControlledUnetModel(UNetModel)`（cldm/cldm.py:22-45）继承 SD 原版 UNet，唯一改动是 forward 签名多了 `control`：编码器全程包在 `torch.no_grad()` 里（"locked"的落点，与论文中不存梯度的说法对应），skip connection 处把控制信号加上去：

```python
for i, module in enumerate(self.output_blocks):
    if only_mid_control or control is None:
        h = torch.cat([h, hs.pop()], dim=1)
    else:
        h = torch.cat([h, hs.pop() + control.pop()], dim=1)
    h = module(h, emb, context)
```

`ControlLDM(LatentDiffusion)`（cldm/cldm.py:308）把三者组装：`apply_model`（cldm/cldm.py:328-341）先跑 control_model 得到 13 路控制，按 `control_scales` 缩放后注入 UNet；`configure_optimizers`（cldm/cldm.py:416-423）只优化 `control_model`，`sd_locked=True` 时连 SD 解码器也不训——与 `docs/train.md` 中"sd_locked/only_mid_control"两个训练选项直接对应。`low_vram_shift`（cldm/cldm.py:425-435）通过 CPU/GPU 交换模块支撑 8GB 显卡（`docs/low_vram.md`）。

### 5.3 从 SD checkpoint 构建 ControlNet（`tool_add_control.py`）

权重初始化的关键脚本（tool_add_control.py 全文约 50 行）：新建空 ControlLDM 后，把所有 `control_*` 前缀的键映射回 `model.diffusion_*` 从原 SD 权重复制，只有零卷积等新增层保留随机/零初始化——"no layer is trained from scratch"（`README.md`）的落点。`tool_add_control_sd21.py` 是 SD2 版，`tool_transfer_control.py` 支持迁移到任意社区模型（README "Use ControlNet in Any Community Model" 一节）。

### 5.4 采样与 hack 层（`cldm/ddim_hacked.py`、`cldm/hack.py`）

`cldm/ddim_hacked.py` 是从 `ldm/models/diffusion/ddim.py` 拷贝改出的 DDIM 采样器（diff 约 40 行），使采样循环能携带 control 条件。`cldm/hack.py` 集中三处 monkey-patch：关闭 transformers 冗余日志、切片 attention（代码内注释自认 "Stolen from basujindal/stable-diffusion" 的省显存技巧）、以及作者自写的 CLIP forward（支持 >77 token 的 3×75 分段与 clip_skip）。所有 gradio 入口经 `share.py` → `config.py`（`save_memory` 开关）激活这些补丁。

## 6. 构建与运行

无构建系统；运行即"conda 环境 + 下载权重 + python 脚本"（均出自 `README.md` 与 `docs/`）：

```bash
# 环境
conda env create -f environment.yaml && conda activate control

# 从 SD1.5 checkpoint 生成 ControlNet 初始权重（docs/train.md Step 3）
python tool_add_control.py ./models/v1-5-pruned.ckpt ./models/control_sd15_ini.ckpt

# 训练（tutorial_train.py：batch_size=4, lr=1e-5, sd_locked=True）
python tutorial_train.py

# 演示（9 个同类入口之一；权重从 huggingface.co/lllyasviel/ControlNet 下载）
python gradio_canny2image.py
python gradio_annotator.py   # 自动标注工具（docs/annotator.md）

# 低显存：config.py 中 save_memory = True（docs/low_vram.md）
```

模型权重与数据集全部为 Hugging Face 外链（README "Production-Ready Pretrained Models" 节；`annotator/ckpts/` 仅有占位文件），仓内 `models/` 只有 YAML。

## 7. 测试与工程质量

- **无测试框架、无测试目录**：全仓唯一带 test 的文件 `tutorial_dataset_test.py`（206 字节）只是顺序访问数据集并打印样本，非断言式测试。
- **无 CI**：仓库无 `.github/` 目录（tarball 顶层确认），无 workflow、无 issue/PR 模板。
- **代码质量观察**：核心增量代码（cldm/）紧凑可读，注释直白（"Fool!! You forgot..."、"Magic number. IDN why"——guess_mode 的 0.825 阶梯衰减，`gradio_canny2image.py` process 函数内）；但工程化程度低——11 个 gradio 脚本大量复制粘贴，ldm/ 与 annotator/uniformer 为整库拷贝（uniformer 内嵌 mmcv/mmseg 数千行），无包管理、无 Lint。对研究仓而言可接受，对依赖它的生态而言不可复用，这也是后来 diffusers 重写实现的动机之一。

## 8. 社区与维护状态

- **生命周期时间线**（gh api + README News 节）：2023-02-01 建仓；2023-02-12 社区模型迁移、2023-02-20 Guess Mode、2023-02-26 消融实验讨论、2023-03-03 预计算讨论——两个月内功能密集发布；1.1 版模型转为独立仓 `ControlNet-v1-1-nightly`（README 顶部 News；该仓 gh api 实测 5,158 星、最后 push 2024-08-08）。main 分支最后实质提交 2023-04-23（"Change requested by an important anonymous user"），最后两次提交为 2023-09-09 的 README 更新；仓库 pushed_at 2024-02-25，此后无任何提交。**未设 archived，但代码已两年多无实质维护。**
- **贡献者结构**：6 人中 lllyasviel 占 183 commits（约 97%），其余 5 人各 1 commit（gh api contributors）——典型单人研究仓。
- **Release 节奏**：无 tag、无 release，版本以 README News 与独立 nightly 仓的形式外置。
- **作者后续项目**（gh api users/lllyasviel/repos 按最近 push 排序）：Fooocus（53.9k★）、FramePack（17.4k★）、stable-diffusion-webui-forge（13.1k★）、IC-Light（8.6k★）、Paints-UNDO（4.1k★）等，2024 年后作者产出与维护重心已完全转移，与本仓停更时间线吻合。
- **生态集成（客观事实，gh api 核实）**：diffusers 官方移植位于 `huggingface/diffusers` 的 `src/diffusers/models/controlnets/controlnet.py` 与 `multicontrolnet.py`（附 `examples/controlnet/train_controlnet.py` 训练示例）；ComfyUI 的 ControlNet 支持在 `comfyanonymous/ComfyUI` 的 `comfy/controlnet.py`；A1111 生态的 `Mikubill/sd-webui-controlnet` 扩展（17.8k★，pushed 2024-08-12）则直接消费本仓权重。README 致谢节另列有 HF Spaces、Colab 列表等大量第三方衍生。

## 9. 亮点与风险

**亮点**
1. 零卷积 + 结构复制的设计极简且可证安全：初始扰动为零、锁定副本不存梯度，训练成本接近微调（`cldm/cldm.py`、`docs/faq.md`）。
2. 核心增量极小（cldm/ 不足千行），配合 `tool_add_control.py` 的权重映射脚本，"给任意 SD 加控制"可被任何人复现——这是它成为论文 artifact 标杆的原因。
3. 训练教程质量高：`docs/train.md` 完整给出 Fill50K 流程、"sudden converge"（3k-7k 步）现象、梯度累积策略与 sd_locked/only_mid_control 取舍。
4. 条件检测器全家桶自带（canny/hed/mlsd/midas/openpose/uniformer，`annotator/`），论文到演示的链路完整。
5. 生态影响巨大：diffusers 官方移植、ComfyUI 原生支持、A1111 扩展均以其权重/架构为事实标准（第 8 章核实记录）。

**风险/局限**
1. 停更仓：2023-09 后无代码提交，依赖 pinned 在 2023 年初版本（torch 1.12、gradio 3.16），新环境安装大概率冲突；安全补丁亦无。
2. 工程化程度低：无测试、无 CI、无包化，gradio 脚本复制粘贴，ldm/annotator 为大段拷贝，二次开发成本高。
3. 权重外链依赖 Hugging Face 且不在仓内（`models/` 仅 YAML、`annotator/ckpts/` 为空），离线复现需自行凑齐约 9 个模型 + 7 个检测器权重。
4. 显存优化靠 monkey-patch（`cldm/hack.py` 覆写 attention/CLIP forward），与新版 PyTorch/transformers 兼容性脆弱。

## 10. 总结评价

ControlNet 官方仓是"小代码改变一个领域"的教科书案例：以不到千行的核心增量（`cldm/cldm.py` 的三件套——ControlNet 可训练副本、ControlledUnetModel 锁定注入、ControlLDM 组装）加上零卷积初始化，把可控生成从小众技巧变成 2023-2024 年图像生成的事实标准，其架构被 diffusers、ComfyUI、A1111 全部采纳。但它本身是一个停更的单人研究仓（最后提交 2023-09-09，作者转向 Fooocus/FramePack 后再无维护），工程质量停留在"论文 artifact"水准。今天读它，价值在架构思想与极简实现而非直接使用：想理解可控生成原理、复现论文或写自己的条件注入模块的人，这仍是最值得精读的源码；想在实际生产中用 ControlNet，应直接使用 diffusers 的移植实现或 ComfyUI/A1111 生态，而非本仓。
