# CompVis/stable-diffusion 项目分析

> 星数 73536（快照 2026-10-24）｜主语言 Jupyter Notebook｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

**一句话定位：** Stable Diffusion v1 的官方原始实现与发布仓库——慕尼黑大学 CompVis 实验室基于 Latent Diffusion（LDM）架构的首个开源文生图扩散模型参考代码。

| 项目 | 内容 |
|---|---|
| 名称 | CompVis/stable-diffusion |
| 星数 | 73536（快照），实测 73559（gh api，2026-10-31） |
| 语言 | Jupyter Notebook（4.17MB notebook 字节数占优）+ Python 约 520KB（gh api languages） |
| 许可证 | CreativeML Open RAIL-M（`LICENSE`，GitHub 识别为 NOASSERTION） |
| 创建时间 | 2022-08-10（gh api） |
| 最后活跃 | 主分支最后提交 2022-11-16；仓库 pushed_at 2024-06-18（非主分支内容变化，未确认） |
| 贡献者 | 约 8 人（contributors 分页 last=8，含匿名，gh api） |
| Issues | 620 open / forks 10668（gh api，2026-10-31） |
| Releases | 无（releases API 为空） |

## 2. 项目是什么，解决什么问题

这是 2022 年 8 月 Stability AI 与 Runway 资助、CompVis（Björn Ommer 组）发布的研究项目，对应 CVPR '22 Oral 论文《High-Resolution Image Synthesis with Latent Diffusion Models》（`README.md`、`Stable_Diffusion_v1_Model_Card.md`）。它解决的问题非常明确：把像素空间的扩散模型搬到低维潜空间（512x512 图像压缩为 64x64x4 潜张量，下采样因子 8），从而将训练和推理算力需求降低一个数量级，使"消费级 GPU 上跑文生图"成为可能——README 明确称模型含 860M UNet 与 123M CLIP 文本编码器，10GB 显存即可运行（`README.md`）。

模型以冻结的 CLIP ViT-L/14 文本编码器做条件引导（`ldm/modules/encoders/modules.py` 的 `FrozenCLIPEmbedder`，max_length=77），UNet 在潜空间上以 cross-attention 接收文本嵌入。权重先后在 laion2B-en（256x256）、laion-high-resolution、laion-aesthetics v2 5+（512x512）上训练，发布 sd-v1-1 到 sd-v1-4 四个检查点，并引入 10% 文本条件丢弃以支持 classifier-free guidance（`README.md` "Weights" 一节）。

仓库的代码部分是从作者前作 `CompVis/latent-diffusion` 演化而来的研究代码库：以 `main.py`（PyTorch Lightning 训练入口）+ `ldm/` 库 + 一堆 YAML 配置驱动，通过 OmegaConf 按配置字符串动态实例化模型（`ldm/util.py` 的 `instantiate_from_config`）。推理则走 `scripts/txt2img.py`、`scripts/img2img.py`、`scripts/inpaint.py` 等独立脚本。

它同时是"负责任 AI 开源"的标志性案例：权重采用 CreativeML Open RAIL-M 许可证（`LICENSE`），附带使用限制条款与模型卡；推理脚本内置 NSFW 安全检查（diffusers 的 `StableDiffusionSafetyChecker`）和 invisible-watermark 隐形水印（`scripts/txt2img.py`）。

## 3. 技术栈与依赖

来自 `environment.yaml` 与 `setup.py`：

| 依赖 | 版本/来源 | 用途 |
|---|---|---|
| Python / PyTorch | 3.8.5 / 1.11.0（cudatoolkit 11.3） | 运行时 |
| pytorch-lightning | 1.4.2 | 训练框架（`main.py` Trainer、模块基类） |
| transformers | 4.19.2 | CLIP tokenizer / CLIPTextModel |
| diffusers | 最新 | 仅用其 `StableDiffusionSafetyChecker`（`scripts/txt2img.py`） |
| taming-transformers / openai CLIP | git 依赖（environment.yaml） | VQ 感知损失 / CLIP |
| omegaconf | 2.1.1 | YAML 配置与模型实例化 |
| einops / kornia / torch-fidelity / torchmetrics | 固定版本 | 张量重排 / 图像操作 / FID 评估 |
| invisible-watermark | 最新 | 输出图隐形水印 |
| opencv-python / albumentations / imageio | 固定版本 | 图像 IO 与增广 |

`setup.py` 极简（包名 latent-diffusion 0.0.1，仅 torch/numpy/tqdm 硬依赖），实际依赖以 conda environment.yaml 为准。无 lock 文件，版本固定风格是 2022 年典型的"研究快照"式依赖管理。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `main.py` | 训练/评估统一入口（741 行），argparse + Lightning Trainer + 检查点回调 |
| `ldm/` | 核心库（34 个 .py）：模型、模块、采样器、数据、损失 |
| `ldm/models/diffusion/` | `ddpm.py`(1445 行，含 `LatentDiffusion`)、`ddim.py`、`plms.py`、`classifier.py`、`dpm_solver/` |
| `ldm/models/autoencoder.py` | `VQModel` / `AutoencoderKL`（285 行起）第一潜空间模型 |
| `ldm/modules/diffusionmodules/` | `openaimodel.py`(961 行，`UNetModel`)、`model.py`(835 行，AE 的 Encoder/Decoder) |
| `ldm/modules/attention.py` | `CrossAttention`、`SpatialTransformer`（cross-attn 条件注入） |
| `ldm/modules/encoders/modules.py` | 各类条件编码器，关键是 `FrozenCLIPEmbedder` |
| `ldm/modules/losses/` | 感知压缩损失（contperceptual / vqperceptual） |
| `ldm/data/` | 数据集抽象（imagenet / lsun / base 迭代式数据集） |
| `configs/` | 训练/推理 YAML：`stable-diffusion/v1-inference.yaml` 及 autoencoder、latent-diffusion、检索增强等 |
| `models/` | 各任务/第一阶段的检查点配套 config（kl-f4~f32、vq-f4~f16、text2img256、inpainting_big 等） |
| `scripts/` | 推理与工具脚本（txt2img/img2img/inpaint/knn2img/sample_diffusion）+ 下载脚本 + 唯一测试 |
| `notebook_helpers.py` + `scripts/latent_imagenet_diffusion.ipynb` | Jupyter 演示（该 notebook 4.17MB，是 GitHub 语言统计的主导项） |
| `assets/` / `data/` | 示例图与字体、示例条件图 |

组织方式评述：典型"单仓研究库"——一个 Python 包 + YAML 驱动的多任务模型族（SD、LDM 各种任务、超分、检索增强复用同一套 `LatentDiffusion` 基类，通过配置切换），没有 monorepo/多包结构，没有 docs 站，README 即全部文档。

## 5. 核心架构与关键实现

### 5.1 LatentDiffusion：三段式主类（`ldm/models/diffusion/ddpm.py`）

`DDPM`(L44) 是标准 DDPM 基类；`LatentDiffusion`(L424) 继承并组合三个组件：冻结的 `first_stage_model`（AutoencoderKL）、冻结的 `cond_stage_model`（FrozenCLIPEmbedder）、和 `DiffusionWrapper`(L1395) 包裹的 UNet。冻结通过禁用 train 方法 + requires_grad=False 实现（`instantiate_first_stage`/`instantiate_cond_stage`）；潜空间缩放因子 0.18215 固定在配置中（`configs/stable-diffusion/v1-inference.yaml`），也支持首批数据自动按 std 重标定的 `scale_by_std` 路径。训练损失（`p_losses`，L1012）即 eps 预测的 simple loss + VLB 加权：

```python
def p_losses(self, x_start, cond, t, noise=None):
    noise = default(noise, lambda: torch.randn_like(x_start))
    x_noisy = self.q_sample(x_start=x_start, t=t, noise=noise)
    model_output = self.apply_model(x_noisy, t, cond)
    ...
    loss_simple = self.get_loss(model_output, target, mean=False).mean([1, 2, 3])
    loss = loss_simple / torch.exp(logvar_t) + logvar_t
    loss = self.l_simple_weight * loss.mean()
    loss_vlb = (self.lvlb_weights[t] * loss_vlb).mean()
    loss += (self.original_elbo_weight * loss_vlb)
```

### 5.2 UNetModel + SpatialTransformer（`ldm/modules/diffusionmodules/openaimodel.py`、`ldm/modules/attention.py`）

UNet 实现源自 OpenAI ADM 代码库（README "Comments" 致谢）。v1 配置：model_channels=320，channel_mult=[1,2,4,4]，num_heads=8，attention_resolutions=[4,2,1]，`use_spatial_transformer: True`，context_dim=768（`configs/stable-diffusion/v1-inference.yaml` L29-44）。文本条件注入点在 `attention.py` 的 `CrossAttention`(L152)：query 来自图像特征，key/value 来自 CLIP 嵌入，经 `BasicTransformerBlock`(L196) 组成 `SpatialTransformer`(L218) 嵌入 UNet 中层；`zero_module` 工具将注意力输出层零初始化，保证训练起步稳定。

### 5.3 采样器与 classifier-free guidance（`ldm/models/diffusion/ddim.py`、`plms.py`、`dpm_solver/`）

提供 DDIM、PLMS（默认，Katherine Crowson 实现，见 README）、DPM-Solver（2022-11-06 合入 LuChengTHU 的 PR #440，见 git log）三种采样器。CFG 的经典双批写法（`ddim.py` p_sample_ddim）：

```python
x_in = torch.cat([x] * 2)
t_in = torch.cat([t] * 2)
c_in = torch.cat([unconditional_conditioning, c])
e_t_uncond, e_t = self.model.apply_model(x_in, t_in, c_in).chunk(2)
e_t = e_t_uncond + unconditional_guidance_scale * (e_t - e_t_uncond)
```

`DDIMSampler.stochastic_encode`/`decode`（L207/223）还支撑了 img2img 的 SDEdit 式加噪-去噪流程（`scripts/img2img.py` 的 `--strength`）。

### 5.4 条件编码器与推理管线（`ldm/modules/encoders/modules.py`、`scripts/txt2img.py`）

```python
class FrozenCLIPEmbedder(AbstractEncoder):
    def __init__(self, version="openai/clip-vit-large-patch14", device="cuda", max_length=77):
        self.tokenizer = CLIPTokenizer.from_pretrained(version)
        self.transformer = CLIPTextModel.from_pretrained(version)
        self.freeze()
    def forward(self, text):
        ...
        z = outputs.last_hidden_state
        return z
```

`txt2img.py` 完整演示了"负责任生成"链路：`load_model_from_config` 从 ckpt + YAML 重建模型 → 采样 → `check_safety`（L88，diffusers SafetyChecker 判 NSFW 并置黑图）→ `put_watermark`（L69，invisible-watermark）→ 保存。安全检查可通过 `--no-half` 等参数外的环境组合关闭（脚本内无开关，始终执行，L317）。

## 6. 构建与运行

环境（`README.md`、`environment.yaml`）：

```bash
conda env create -f environment.yaml
conda activate ldm
# 或从 latent-diffusion 旧环境升级：
pip install transformers==4.19.2 diffusers invisible-watermark
pip install -e .
```

权重：需从 HuggingFace CompVis 组织下载 `stable-diffusion-v1-*-original` 并软链到 `models/ldm/stable-diffusion-v1/model.ckpt`（README "Weights"）。生成：

```bash
python scripts/txt2img.py --prompt "a photograph of an astronaut riding a horse" --plms
python scripts/img2img.py --prompt "A fantasy landscape, trending on artstation" --init-img <img> --strength 0.8
python scripts/inpaint.py --prompt ... --init-img ... --mask-img ...
```

常用参数：`--scale 7.5`（CFG）、`--ddim_steps 50`、`--H/--W 512`、`--seed`、`--precision autocast`（`scripts/txt2img.py` argparse）。训练入口为 `python main.py --base configs/... --train`（`main.py` get_parser），仓库未提供可直接复现 SD v1 训练的数据管道配置（训练配置针对 laion 数据集需自备，未确认端到端可跑）。

## 7. 测试与工程质量

- 测试规模：仅 1 个测试文件 `scripts/tests/test_watermark.py`（17 行，pytest 风格，验证水印可检测性）。无单元测试体系，模型正确性靠"跑脚本看图"。
- CI：无。仓库没有 `.github/` 目录（tarball 实测），无 workflow、无 issue 模板、无 release 流程。
- 代码质量观察：配置驱动架构（OmegaConf target 字符串反射实例化，`ldm/util.py`）灵活但可读性一般；`ddpm.py` 单文件 1445 行承载 DDPM+LDM+多任务，是典型研究代码形态；docstring 与注释较充分，关键论文公式处有出处标注。依赖已严重过时（pytorch 1.11 / lightning 1.4.2 / transformers 4.19.2），在当前环境难以直接安装运行。

## 8. 社区与维护状态

- 时间线：2022-08-10 建仓，08-22 以 Open RAIL-M 许可证发布（git log "Release under CreativeML Open RAIL M License"），2022-10-28 合入 DPM-Solver，**主分支最后一次提交 2022-11-16**（"Update sampler.py"，dpm_solver 稳定性修正）。此后主分支再无代码更新；pushed_at 显示 2024-06-18 有仓库写入事件，但主分支 HEAD 未变（具体内容未确认）。
- 功能演进事实性外移：README 自己声明"diffusers 集成预期会有更活跃的社区开发"（README "Diffusers Integration"），后续生态（SD v2、SDXL）由 Stability AI 的 generative-models 与 HuggingFace diffusers 承接，本仓库实际进入冻结归档状态（未标记 archived）。
- 社区规模：约 8 名贡献者（含匿名，contributors API）；10668 forks、73559 stars、620 个 open issues（gh api 2026-10-31）——issue 区长期无维护，多为使用环境问题。无 GitHub Releases，权重分发完全走 HuggingFace。

## 9. 亮点与风险

**亮点：**
1. 历史地位无可替代：第一个开源的 SOTA 级文生图模型，引爆了整个生成式 AI 生态（Stable Diffusion 系、ComfyUI、LoRA 等均由此起步）。
2. 架构教科书级清晰：`LatentDiffusion` 三组件（AE + 条件编码器 + UNet）解耦、YAML 配置驱动，是理解潜扩散模型最直接的可读实现。
3. 教程价值完整：CFG 双批引导（`ddim.py`）、img2img 的 SDEdit 机制、inpainting 掩码合并（`ddim.py` p_sample_loop 的 mask 逻辑）都有最小实现。
4. 负责任发布范式：Open RAIL-M 许可证 + 模型卡 + NSFW 检查器 + 隐形水印（`LICENSE`、`Stable_Diffusion_v1_Model_Card.md`、`scripts/txt2img.py`），成为后续模型发布的模板。

**风险/局限：**
1. 代码已实质停滞三年（主分支 2022-11 后无提交），依赖栈（torch 1.11、lightning 1.4.2、transformers 4.19.2）在新环境装不上，按原样复跑成本高。
2. 权重与代码分离且无 release 管理，权重获取依赖 HuggingFace 门控；Open RAIL-M 含使用限制条款，GitHub 标为 NOASSERTION，商用需自行合规审查。
3. 仅一个水印测试，无 CI 无单测，`main.py` 训练路径对 SD v1 数据集不可直接复现，作为训练框架使用价值有限。
4. 620 个 open issues 无人处理，遇到问题基本只能靠社区 fork 与 diffusers 文档。

## 10. 总结评价

这是生成式 AI 史上最具影响力的仓库之一：它不是最好的工程（依赖老旧、无 CI、已冻结），而是"潜空间扩散 + CLIP 文本条件"这一技术路线的开源原点。今天的读者不建议把它当生产工具——推理用 diffusers/ComfyUI，训练用后续 SD 代码库；但作为学习材料，`ldm/` 下约 5000 行核心代码（ddpm.py + openaimodel.py + attention.py + ddim.py）仍是理解扩散模型从训练损失到 CFG 采样全链路的最佳读物。适合 AI 研究者阅读源码理解 LDM 原理、工程者考古借鉴其配置驱动架构，不适合追求新特性或维护支持的使用者。
