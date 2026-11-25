# lllyasviel/Fooocus 项目分析

> 星数 53649（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：Fooocus 是一个以"用户只管写提示词、其余交给软件"为设计哲学的离线开源 Stable Diffusion XL 图像生成器，Gradio 网页界面 + 内置 ComfyUI 后端，从下载到出图不超过 3 次点击。

| 事实 | 内容 |
| --- | --- |
| 仓库 | https://github.com/lllyasviel/Fooocus |
| 星数 | 53649（快照）；gh api 实测 53809 |
| 主语言 | Python（192 个 .py，约 5.2 万行，含 vendored 后端） |
| 许可证 | GPL-3.0（`LICENSE`） |
| 代码版本 | 2.5.5（`fooocus_version.py`，与最新 release v2.5.5 一致） |
| 创建时间 | 2023-08-09 |
| 最近 push | 2025-12-01（gh api `pushed_at`） |
| 最新 release | v2.5.5，2024-08-12（gh api releases） |
| 维护状态 | 官方宣布"受限长期支持（LTS），仅修 bug"（`readme.md` Project Status 节） |

## 2. 项目是什么，解决什么问题

Fooocus 由 ControlNet 作者 lllyasviel（Lvmin Zhang）于 2023 年 8 月发布，自述"基于 Stable Diffusion WebUI 与 ComfyUI 两个代码库的混合体起步"（`readme.md` Thanks 节）。它解决的核心问题是：A1111/ComfyUI 这类工具面向愿意调参的极客，而普通用户只想"输入一句话、得到好图"。Fooocus 把 Midjourney 式的"无参数体验"搬到了本地离线环境（`readme.md`：offline, open source, free, manual tweaking is not needed）。

实现"免调参"的手段是真实算法而非纯 UI 包装：内置 GPT-2 提示词扩写引擎（Fooocus V2 动态风格，`extras/expansion.py`，含 positive 词 logits 偏置），精心调校的采样参数（`presets/default.json` 中 cfg=4.0、dpmpp_2m_sde_gpu、karras）、SDXL 专属的 Negative ADM Guidance 与 SAG 锐度补偿（`readme.md` "Hidden Tricks" 1-14 条），以及 6 份风格模板 JSON 与内置默认风格组合（`sdxl_styles/`）。

工程体验也是卖点：Windows 提供 7z 一键包与 `run.bat`，首次启动自动下载模型，缺模型时回退到 `previous_default_models` 列表避免重复下载（`launch.py` download_models），最低 4GB NVIDIA 显存（借 Windows 虚拟内存交换，`readme.md` Minimal Requirement 表）。

三种预设（General/Realistic/Anime）分别绑定 `run.bat`/`--preset` 与不同默认模型（juggernautXL_v8Rundiffusion 等），2.3.0 起可在浏览器内直接切换预设（`readme.md` Download 节）。

2024 年 8 月后项目进入 LTS 状态：只修 bug，不迁移新模型架构（如 Flux），官方推荐 Flux 需求用户转用同作者的 WebUI Forge 或 ComfyUI（`readme.md` Project Status 节）。

## 3. 技术栈与依赖

- 语言/运行时：Python 3.10（README 明确要求；AMD/Mac 亦走同一代码路径）。
- UI：Gradio 3.41.2 锁版本（`requirements_versions.txt`），并通过 `modules/gradio_hijack.py`（483 行）猴补 Gradio 内部实现以定制外观与行为。
- 后端：`ldm_patched/`（143 文件，3.3MB）是 ComfyUI 推理后端的 vendored 分支——目录名即 ComfyUI 内部包名，`ldm_patched/licenses-3rd/comfyui` 亦有署名；k_diffusion、ldm、taesd、unipc、contrib（节点式算子）俱全。
- 模型与算法库：torch 2.1.0（`launch.py` prepare_environment）、transformers 4.42.4（GPT-2 扩写）、safetensors、onnxruntime + rembg + groundingdino-py + segment_anything（inpaint/enhance 的自动遮罩链路）、timm（BLIP/WD14 反推）、opencv-contrib-headless。
- 打包与分发：`environment.yaml`（conda）、`requirements_versions.txt`（pip 锁版本）、`Dockerfile` + `docker-compose.yml`、`build_launcher.py`（生成 Windows 批处理启动器）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `webui.py`（1128 行） | Gradio 界面搭建 + `generate_clicked` 生成主流程 |
| `entry_with_update.py` | 启动入口：pygit2 拉取远端做 fast-forward 自更新，再转 launch |
| `launch.py` | 环境准备（装 torch/依赖）→ 预设模型下载 → 哈希缓存 → `from webui import *` |
| `args_manager.py`（61 行） | argparse 命令行参数（60+ 个 flag 见 README） |
| `modules/`（30 文件，约 7550 行） | 自研封装层：config、core、default_pipeline、async_worker、patch/hijack 系列 |
| `ldm_patched/`（143 文件） | vendored ComfyUI 推理后端（采样器、模型加载、LoRA patcher） |
| `extras/`（53 文件） | 自研算法 + vendored 第三方（BLIP、GroundingDINO、facexlib、sam、safety_checker） |
| `models/` | 19 个子目录（checkpoints/loras/vae/inpaint/prompt_expansion 等），空目录 + README 占位 |
| `presets/` | 8 份预设 JSON：default/anime/realistic/lcm/lightning/playground_v2.5/pony_v6/sai |
| `sdxl_styles/` | 6 份风格模板 JSON（含数千条缩略图 samples/） |
| `wildcards/` | 通配符词表 txt（`__color__` 类语法） |
| `tests/` | 2 个测试文件（unittest） |
| `.github/workflows/` | 仅 build_container.yml（构建推送 ghcr 镜像） |
| `language/`、`css/`、`javascript/` | i18n JSON、样式、前端脚本 |

组织方式评述：单仓单体应用。"自研模块（modules/extras）+ vendored 后端（ldm_patched）"双轨：好处是用户零安装复杂度、行为完全可控（官方说"near free of programmatic issues"依赖此封闭性）；代价是每次升级 Gradio/ComfyUI 都要手动搬代码，依赖在 requirements 中全部锁死。

## 5. 核心架构与关键实现

### 5.1 启动管线（entry → launch → webui）

`entry_with_update.py` 用 pygit2 对自身仓库做 fetch + merge_analysis，可 fast-forward 则硬重置到远端（用户无需 git 即可自更新）；随后 `launch.py` 按序执行：`prepare_environment()` 安装 torch 2.1.0 与 `requirements_versions.txt` → `build_launcher()` → `ini_args()` 解析命令行 → `download_models()` 按预设 JSON 的 `checkpoint_downloads` 下载模型，且当默认模型缺失时遍历 `previous_default_models` 找已有旧模型顶替（`launch.py:103-142`）→ `init_cache()` 建立模型哈希缓存（`modules/hash_cache.py`）→ 末行 `from webui import *` 触发 UI 构建。Gradio 端口默认 7865（`launch.py:14`）。

### 5.2 配置与默认参数哲学（分层解析）

参数解析在 `modules/config.py`（997 行）完成，优先级链清晰可考：

1. 代码内置 default（`get_config_item_or_set_default` 的入参）；
2. `presets/default.json` 先加载（`config.py:35-40`）；
3. 用户 `config.txt` 覆盖（`config.py:39-41`，首次启动自动生成）；
4. `--preset anime/realistic` 再覆盖（`config.py:132` `config_dict.update(try_get_preset_content(preset))`）；
5. 同名环境变量最高优先（`get_config_item_or_set_default` 首段 `v = os.getenv(key)`）。

```python
# modules/config.py get_config_item_or_set_default（节选）
v = os.getenv(key)                       # 1. 环境变量
if v is not None:
    v = try_eval_env_var(v, expected_type)
    config_dict[key] = v
if key not in config_dict:               # 2. preset/config.txt 已注入
    config_dict[key] = default_value     # 3. 代码默认值
    return default_value
```

每份预设 JSON 同时声明默认模型、LoRA 槽位、cfg、采样器、分辨率与下载 URL（`presets/default.json`），实现"预设即配置即安装清单"的合一设计。

### 5.3 管线封装层（modules/core.py + default_pipeline.py）

`core.StableDiffusionModel`（`modules/core.py:37`）把 ComfyUI 风格的 ModelPatcher 聚合为 unet/vae/clip/clip_vision 四件套，并为 LoRA 预建 key map；`refresh_loras` 通过 `self.unet.clone()` 后 `add_patches` 应用 LoRA——非破坏式 patch，换 LoRA 不重载底模（`core.py:92-119`）。core 顶部直接实例化 ComfyUI 节点算子（`opVAEDecode = VAEDecode()` 等，`core.py:26-34`），说明封装层是把节点图"摊平"成过程式调用。

`default_pipeline.py` 用模块级单例（`model_base`/`model_refiner`/`final_unet`/`final_clip`...）承载全局状态，所有入口函数带 `@torch.no_grad()+@torch.inference_mode()` 装饰器；`refresh_everything`（`default_pipeline.py:234`）是模型/LoRA/CLIP-skip 的统一刷新入口。核心采样在 `process_diffusion`（`default_pipeline.py:336`）：

```python
def process_diffusion(positive_cond, negative_cond, steps, switch, width, height,
                      image_seed, callback, sampler_name, scheduler_name, latent=None,
                      denoise=1.0, tiled=False, cfg_scale=7.0,
                      refiner_swap_method='joint', disable_preview=False):
```

支持 joint/separate/vae 三种 refiner 切换策略，joint 模式即 README "Hidden Tricks" 第 2 条的卖点——base 与 refiner 在同一个 k-sampler 内原生切换、复用采样动量（实现见 `modules/sample_hijack.py` 对 k_diffusion 采样的劫持 + `ldm_patched/k_diffusion/`）；`get_candidate_vae`（`default_pipeline.py:319`）按 karras sigma 指数 0.834 判定 refiner 阶段是否需要 VAE 参与图生图。

### 5.4 异步任务队列与生成编排（modules/async_worker.py + webui.py）

UI 线程 `generate_clicked`（`webui.py:33`）把约 50 个 Gradio 控件按位置顺序打包成 `AsyncTask(args)`（`async_worker.py:10-120` 逐个 `args.pop()` 解包成命名属性），推入 `async_tasks` 队列后循环消费 `task.yields` 实现进度流式渲染；后台 `worker()` 线程（`async_worker.py:168`）串行处理任务，内嵌通配符/数组/内联 LoRA 提示词处理（`modules/util.py` apply_wildcards 等）、NSFW 审查（`extras/censor.py`）、以及 2.5.0 加入的 enhance 流水线（自动检测 + GroundingDINO/SAM 出罩 + 局部重绘放大，`async_worker.py:1031` enhance_upscale，全文 161 处 enhance 引用）。

## 6. 构建与运行

不涉及构建（纯 Python 应用），以下命令均出自 `readme.md` 与工程文件：

```bash
# Linux (conda)
conda env create -f environment.yaml && conda activate fooocus
pip install -r requirements_versions.txt
python entry_with_update.py                # 或 --preset anime / --preset realistic / --listen

# Linux (venv, 需 Python 3.10)
python3 -m venv fooocus_env && source fooocus_env/bin/activate
pip install -r requirements_versions.txt
python entry_with_update.py

# Windows：下载 Fooocus_win64_2-5-0.7z 解压后运行 run.bat / run_anime.bat / run_realistic.bat

# Docker
# 见 docker.md / Dockerfile / docker-compose.yml；CI 自动推 ghcr 镜像（.github/workflows/build_container.yml）

# 测试（仓库自带，仅 2 个文件）
python -m unittest discover -s tests
```

## 7. 测试与工程质量

- 测试：`tests/` 仅 `test_utils.py`、`test_extra_utils.py` 两个文件，unittest 风格，覆盖通配符解析与环境变量求值等纯函数；核心管线（采样、模型加载、enhance）无测试。
- CI：`.github/workflows/` 仅 build_container.yml（push/tag 触发构建 ghcr 镜像），无测试、无 lint 工作流。
- 代码质量观察：大量猴补风格（`gradio_hijack.py`、`sample_hijack.py`、`patch.py` 的 patch_all）、`default_pipeline.py` 全局单例可变状态、`AsyncTask` 依赖位置参数解包（`async_worker.py:33-120`，控件顺序与解包顺序必须严格一致，是明显的脆弱点）、print 式日志（`private_logger.py`）。工程上"锁死一切依赖"换取复现性与稳定，符合其 LTS 定位。

## 8. 社区与维护状态

- 节奏：2023-08 创建后爆发式增长；release 从 v1.x 密集迭代到 v2.5.5（2024-08-12）后不再发版；此后仓库仍有 push（最近 2025-12-01，gh api），对应"仅修 bug 的 LTS"承诺（`readme.md` Project Status）。
- 贡献者：gh api contributors 分页计数约 59 人；README 明确将长期维护归功于 mashb1t。ControlNet 作者 lllyasviel 为发起人。issues 开放数 316、forks 8621（gh api）。
- 生态：README 维护一份 fork 列表（Fooocus-Control、RuinedFooocus、Fooocus-MRE、mashb1t/Fooocus 等），创新活动已转移到 fork 与 WebUI Forge；官方反复警告仿冒网站（fooocus.com 等均为假），侧面反映其品牌流量之大。
- 未来动向：官方称若社区收敛出统一的新一代图像生成方法，Fooocus 可能随之迁移（`readme.md`），但截至快照无动作。

## 9. 亮点与风险

亮点：

1. "默认即最佳"的产品哲学落地为代码资产：预设 JSON 同时是配置与安装清单（`presets/*.json`），GPT-2 提示扩写（`extras/expansion.py`）与 14 条 SDXL 采样技巧（`readme.md` Hidden Tricks）让短提示也能稳定出好图。
2. 单 k-sampler 内原生 refiner swap（`modules/sample_hijack.py` + `default_pipeline.py:process_diffusion`），比 A1111 双采样器方案保留采样动量，质量更连贯，且该思路被反向回馈进 A1111 dev 分支（README 注释）。
3. "自研薄封装 + vendored ComfyUI 后端"的架构取舍得当：拿到节点系统的健壮推理栈（`ldm_patched/`），又免去用户理解节点图的门槛。
4. 面向小白的全链路自动化：pygit2 自更新（`entry_with_update.py`）、模型自动下载与旧模型回退（`launch.py:119-129`）、4GB 显存可跑、多平台启动器生成（`build_launcher.py`）。
5. 自研 inpaint / image prompt 算法（`modules/inpaint_worker.py`、`extras/ip_adapter.py`，配套 1.28GB 专属 control patch），效果官方称优于通用 SDXL 方案。

风险/局限：

1. 架构冻结在 SDXL：官方明确不迁移 Flux/SD3（`readme.md` Project Status），功能演进依赖 fork，直接使用会逐渐落后。
2. Gradio 3.41.2 等依赖全部锁死且被猴补（`modules/gradio_hijack.py`），升级与安全补丁成本高。
3. 测试几乎为零、无测试 CI、全局状态 + 位置参数解包（`async_worker.py` AsyncTask）使外部贡献与重构困难。
4. 2024-08 后无 release，仅零星 bug 修复，属于"事实停更、名义 LTS"；新显卡/新 Python 版本的适配未确认。

## 10. 总结评价

Fooocus 是 2023 年"文生图大众化"浪潮中最成功的极简入口之一：它用 ControlNet 作者的算法品味把 SDXL 调参知识固化成默认值，再用自更新、自动下载、4GB 显存门槛把安装成本压到三连击以内，因此在一年多时间里收获 5 万+ 星。技术上它最有价值的范本是"薄封装层 + vendored ComfyUI 后端"的分层与 `modules/config.py` 的五级配置解析——前者展示了如何把节点系统摊平成过程式管线，后者是"预设覆盖 + 用户配置 + 环境变量"的教科书实现。如今项目处于仅修 bug 的休眠态，不适合作为追新模型的生产平台，但其源码仍是学习"如何为黑盒模型写一个负责任的默认值层"的优质读物；想继续用或改它的人，应转向 README 列出的活跃 fork。
