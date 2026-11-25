# AUTOMATIC1111/stable-diffusion-webui 项目分析

> 星数 165122（快照 2026-10-24）｜主语言 Python｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：Stable Diffusion 官方生态之外最流行的本地浏览器图形界面，基于 Gradio，让普通用户零代码跑文生图/图生图，并通过脚本与扩展体系成为 SD 社区事实上的标准宿主。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | stable-diffusion-webui | README.md |
| 星数 | 165179（gh api，2026-10-30） | GitHub API |
| 语言 | Python（约 43.6k 行）+ JavaScript（约 3.7k 行） | `find ... wc -l` 统计 |
| 许可证 | AGPL-3.0 | LICENSE.txt、gh api |
| 最近活跃 | master 最后提交 2024-07-27（v1.10.1）；仓库最近 push 2026-03-02（非 master 分支） | gh api commits/releases、tarball 文件 mtime |
| 最新版本 | v1.10.1（release 发布于 2025-02-09） | CHANGELOG.md、gh api releases |
| 贡献者 | 430 名（gh api contributors 分页推算） | GitHub API |

## 2. 项目是什么，解决什么问题

Stable Diffusion 的官方发布形态只有 Python 推理脚本和命令行，对绝大多数用户门槛过高。本仓库把"加载模型→写提示词→采样出图→后处理→存图"的完整流程封装成一个浏览器里的多标签界面：txt2img、img2img、Extras（放大与人脸修复）、PNG Info、Checkpoint Merger、Train 等标签开箱即用（README.md "Features" 一节，`modules/ui.py` 的 `create_ui()`）。

它解决的第二层问题是**参数可复现与可流通**：每次生成把全部参数（模型、采样器、种子、提示词等）写进 PNG 文件块或 JPEG EXIF（`modules/images.py`、`modules/infotext_utils.py`），把图片拖回 UI 即可完整还原生成参数——这一机制催生了社区"发图即发参数"的分享文化，是它远超同类工具流行度的直接原因之一。

第三层是**可扩展性**：仓库内置脚本（`scripts/` 下的 xyz_grid、prompt_matrix、outpainting 等 12 个）与扩展（`extensions/` 目录 + `extensions-builtin/` 内置 11 个）构成插件体系，第三方可以不动主代码就注入 UI 控件、生成回调、模型类型与 API 路由。Lora、ControlNet 等社区创新大多通过该体系首发或快速跟进。

覆盖面也刻意做宽：NVidia/AMD/Apple Silicon/Intel/Ascend NPU 多硬件路径（README.md "Installation and Running"；`modules/xpu_specific.py`、`modules/npu_specific.py`、`modules/mac_specific.py`），并提供 `--api` 独立 API 模式（`webui.py` 的 `api_only()`）供集成调用。

## 3. 技术栈与依赖

- 语言/运行时：Python 3.10.6（README 明确新版 Python 不被 torch 支持），前端为 Gradio 自动生成 + 原生 JS 增强。
- UI 框架：**gradio==3.41.2**（`requirements_versions.txt` 精确锁死，这是全项目最关键的锁定依赖，3.x API 与 4.x 不兼容）。
- 推理栈：torch（不锁版，按硬件自动选择源）+ transformers==4.30.2 + open-clip-torch + pytorch_lightning==1.9.4 + safetensors（`requirements_versions.txt`）。
- 采样数学：k-diffusion（`modules/launch_utils.py:414` 以固定 commit git_clone 引入）、torchsde/torchdiffeq；自研 samplers 封装在 `modules/sd_samplers_*.py`。
- 后处理/放大：spandrel + spandrel-extra-arches 作为通用超分模型加载器（README.md Credits；`modules/upscaler_utils.py`），GFPGAN/CodeFormer 人脸修复。
- Web 服务：FastAPI 0.94.0 承载 Gradio 与 `/sdapi/v1/*` REST API（`modules/api/api.py`）。
- 工程：ruff（pyproject.toml，target py39）、eslint（package.json）、pytest（requirements-test.txt）。
- 安装器本身用 Python 写（`launch.py` → `modules/launch_utils.py`，482 行），负责建 venv、装依赖并 git_clone 5 个上游推理仓库（assets、stable-diffusion-stability-ai、generative-models/SDXL、k-diffusion、BLIP，均锁定 commit，`launch_utils.py:411-415`）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `webui.py` / `launch.py` | 顶层入口：launch.py 先装环境，webui.py 初始化并启动 Gradio+API |
| `modules/` | 全部核心逻辑，约 150 个平铺模块（见第 5 章） |
| `modules/api/` | FastAPI REST 接口（api.py 928 行 + models.py） |
| `modules/hypernetworks/`、`modules/textual_inversion/` | 两类微调网络实现与训练 UI |
| `modules/models/` | 上游模型结构适配（diffusion/、sd3/） |
| `scripts/` | 内置用户脚本（xyz_grid、loopback、outpainting_mk_2 等 12 个） |
| `extensions-builtin/` | 内置扩展（Lora、SwinIR、LDSR、ScuNET、canvas-zoom-and-pan、mobile、soft-inpainting 等 11 个） |
| `extensions/` | 用户扩展安装位（仓库只放 "put extensions here.txt"） |
| `javascript/`、`script.js`、`style.css` | 前端增强脚本（约 3.7k 行）与全局样式 |
| `configs/` | 各模型推理配置 yaml（v1、SDXL inpaint、sd3 等） |
| `models/`、`embeddings/`、`textual_inversion_templates/`、`localizations/` | 模型/嵌入/模板/本地化的运行时目录占位 |
| `test/` | pytest 端到端测试（310 行） |
| `.github/workflows/` | run_tests.yaml（CI 测试）、on_pull_request.yaml（ruff+eslint）、warns_merge_master.yml |

组织方式评述：**单仓单包、`modules/` 平铺大目录**风格——没有内部包层级，模块间通过 `modules.shared` 全局单例（`modules/shared.py`）和 `import modules.xxx` 互相引用；插件化只对外（extensions），对内是传统扁平结构。CHANGELOG.md 达 1085 行、97KB，记录了逐版本细粒度变更。

## 5. 核心架构与关键实现

### 5.1 双层扩展体系：Script（生成时插件）与 Extension（完整包）

最小层是 **Script**（`modules/scripts.py:52` `class Script`）：一个定义了 `title()/ui()/run()/process()/process_batch()/postprocess_image()` 等生命周期方法的类。`load_scripts()`（`scripts.py:487`）从 `scripts/` 和 `modules/processing_scripts/` 扫描 .py，用 `modules/script_loading.py:10` 的 `importlib.util.spec_from_file_location` 动态 exec 加载；扩展目录里的脚本同样被收集，`AlwaysVisible` 标记的脚本常驻管线。加载时还会做回调的**拓扑排序**（`scripts.py:12` `topological_sort`），让扩展能声明 callback 执行先后（`modules/extensions.py:31` `CallbackOrderInfo`，metadata.ini 的 before/after requires）。

上层是 **Extension**（`modules/extensions.py:105` `class Extension`）：一个 git 仓库目录，含 `metadata.ini` 声明 canonical_name 与依赖要求；`active()`（`extensions.py:21`）根据 `--disable-all-extensions` 等开关决定启用集合。回调中枢在 `modules/script_callbacks.py`（613 行，注册/触发 before_ui、app_started、model_loaded 等数十种事件）。这套"Script 管线钩子 + Extension 包 + 回调排序"是本项目最核心的可扩展性设计。

```python
# modules/scripts.py:47-48 及 load_scripts()
scripts_list = list_scripts("scripts", ".py") + \
    list_scripts("modules/processing_scripts", ".py", include_extensions=False)
```

### 5.2 生成管线：process_images_inner 主循环

txt2img 与 img2img 共用一个主循环 `modules/processing.py:863` `process_images_inner(p)`（`processing.py:819` 的 `process_images` 只是包了队列锁）。流程：GC→种子处理→`p.scripts.process(p)`（脚本预处理）→进入 `torch.no_grad()` 后按 `n_iter` 迭代：每轮先 `sd_models.reload_model_weights()`（支持 refiner 换模型）→构造 `rng.ImageRNG`→`before_process_batch`→`parse_extra_network_prompts()` 解析 `<lora:...>` 等关键词→`extra_networks.activate(p, ...)` 激活 Lora/Hypernet→`process_batch`→`setup_conds()`（带缓存的条件编码，`processing.py:466` `get_conds_with_caching`）→`p.sample(...)` 采样→`post_sample` 钩子→VAE 解码→`postprocess_image` 等图像级钩子（`processing.py:932-1002`）。

```python
# modules/processing.py:959-961
if not p.disable_extra_networks:
    with devices.autocast():
        extra_networks.activate(p, p.extra_network_data)
```

子类 `StableDiffusionProcessingTxt2Img`（`processing.py:1166`）通过 `sample_hr_pass()`（`processing.py:1364`）实现 Highres Fix 的两段式生成；`StableDiffusionProcessingImg2Img`（`processing.py:1557`）实现重绘蒙版/局部重绘。采样器抽象在 `modules/sd_samplers.py`，k-diffusion 后端在 `sd_samplers_kdiffusion.py`，CFG 去噪器在 `sd_samplers_cfg_denoiser.py`。

### 5.3 模型层 hijack 与状态管理

对上游 Stable Diffusion 的定制不做 fork 而是运行时猴补：`modules/sd_hijack.py` 用 `model_hijack` 替换 UNet/CLIP 的注意力层（`sd_hijack_optimizations.py` 提供 split-attention/xformers/sub-quadratic 三种显存优化）、注入 textual inversion 嵌入词表；`modules/sd_models.py`（1034 行）负责 checkpoint 扫描/加载/热切换，`sd_vae.py`、`sd_unet.py` 分别管理 VAE 与 UNet 替换，`lowvram.py` 做低显存分块卸载。全局配置集中在 `modules/shared.py` 的 `opts`（选项定义拆在 `shared_options.py`），命令行参数在 `modules/cmd_args.py`。

### 5.4 并发与 UI 组织

GPU 是单资源：`modules/call_queue.py:6` 用自研 `FIFOLock` 保证生成任务严格排队，`wrap_gradio_gpu_call` 同时接入进度条（`modules/progress.py`）。UI 由 `modules/ui.py`（1235 行）在 `create_ui()`（`ui.py:253`）里用 gradio Blocks 手工搭建 txt2img/img2img/extras 等接口，各 `ui_*.py` 拆分面板（extra networks、settings、checkpoint merger 等），`ui_loadsave.py` 支持用 ui-config.json 固化默认值。REST API 在 `modules/api/api.py:199` `class Api`，通过 `add_api_route` 挂载 `/sdapi/v1/txt2img` 等端点，UI 与 API 共享同一队列锁。

## 6. 构建与运行

来自 README.md 与仓库脚本（规范要求不实际执行，仅摘录）：

```bash
# Linux 自动安装：装好 python3.10/3.11 与 git 后
wget -q https://raw.githubusercontent.com/AUTOMATIC1111/stable-diffusion-webui/master/webui.sh
./webui.sh          # 首次自动建 venv、拉依赖与 5 个上游仓库；默认 http://127.0.0.1:7860
# Windows：克隆仓库后运行 webui-user.bat

# 仅 API 模式
python launch.py --api --nowebui

# lint / test（CI 同款，见 .github/workflows/）
ruff .              # pyproject.toml 定义规则
npm run lint        # eslint
python launch.py --skip-torch-cuda-test --test-server --do-not-download-clip --no-half --use-cpu all --api-server-stop
python -m pytest -vv test   # 对运行中 127.0.0.1:7860 做端到端断言（pyproject.toml base_url）
```

## 7. 测试与工程质量

- 测试规模很小：`test/` 共 310 行、8 个文件（test_txt2img.py 90 行、test_img2img.py 68 行等），风格是**针对真实运行服务器的端到端冒烟测试**（pytest 经 conftest.py 连 127.0.0.1:7860，CI 里用 `--use-cpu all` + 缓存的空模型跑），而非单元测试。
- CI：`.github/workflows/run_tests.yaml` 在 CPU 上启动服务并 pytest + coverage 报告；`.github/workflows/on_pull_request.yaml` 跑 ruff==0.3.3 与 eslint（Node 18）。lint 配置较宽松：`pyproject.toml` 忽略 E501/C901 等。
- 代码质量观察：43k 行 Python 集中在数百个模块中，存在大量模块级全局态（`shared.opts`、`shared.sd_model`、`model_hijack`），耦合靠 import 顺序约定（`modules/initialize.py:11` 的 `imports()` 手工控制导入次序）；`sd_hijack_*` 系列猴补使代码与上游模型仓库的固定 commit 强绑定。整体可读但可测性一般，与测试规模小互为印证。

## 8. 社区与维护状态

- **master 已实质冻结**：默认分支最后提交为 2024-07-27 的 "changelog"（gh api commits/master），对应 v1.10.0；此后仅 v1.10.1（release 发布时间 2025-02-09）一个小修复版本。仓库 `pushed_at` 为 2026-03-02，说明 2025-2026 年间仍有分支推送，但 master 两年多无实质更新——与任务提示"维护趋于停滞"一致，如实记录。
- 活跃度对比：165k 星仓库当前 open issues 2510 个（gh api），issue 模板齐全（bug_report.yml/feature_request.yml），但按上面事实，主干消化已停。
- 贡献者 430 名（contributors API 分页推算），CODEOWNERS 存在但主要指向作者 AUTOMATIC1111（CODEOWNERS 文件）。
- release 节奏（gh api）：v1.9.3 2024-04 → v1.9.4 2024-05 → v1.10.0-RC 2024-07 → v1.10.0 2024-07 → v1.10.1 2025-02，之后无新版本。
- 生态位（一般社区共识，源码内无直接证据，未逐项核实）：其停滞催生了 Forge（同为 A1111 作者）等活跃分支；本文档仅陈述 master 冻结事实，不做生态判断。

## 9. 亮点与风险

**亮点**
1. 极致的可扩展架构：Script 生命周期钩子 + Extension metadata.ini 依赖/回调拓扑排序 + 数十种事件回调（`modules/scripts.py`、`modules/script_callbacks.py`），是 Python 项目插件体系的教科书案例。
2. infotext 参数随图存取（PNG chunk/EXIF，`modules/infotext_utils.py`）打通了"图片即配置"，重塑了社区分享方式。
3. 一键安装器自研且可靠（`launch.py`/`modules/launch_utils.py` 全部 git_clone 锁 commit，跨 Win/Linux/macOS/NPU），把"装 CUDA 环境跑大模型"降到了双击一个 bat。
4. 硬件兼容面宽：注意力优化可选 xformers/split-attention/sub-quadratic，低显存路径（`lowvram.py`）让 4GB 卡可用（README.md）。
5. CHANGELOG 逐条到 PR 级（1085 行），历史可追溯性好。

**风险/局限**
1. 维护停滞：master 冻结于 2024-07-27，gradio==3.41.2、transformers==4.30.2 等锁定依赖多年未升级（`requirements_versions.txt`），存在已知老版本组件的安全与兼容隐患；新模型支持（SD3 除外）基本要靠分支或竞品。
2. 全局单例 + 模块平铺 + 运行时猴补（`sd_hijack.py`）使核心难以单元测试，测试仅 310 行端到端冒烟，回归风险高。
3. 单进程 FIFOLock 串行所有 GPU 任务（`modules/call_queue.py`），无多用户隔离，Gradio CORS 需要手工摘除中间件补救（`webui.py:100-104` 注释，RyotaK 安全建议），公网暴露需自担风险。
4. AGPL-3.0 对集成到商业产品的传染性约束需注意（LICENSE.txt）。

## 10. 总结评价

stable-diffusion-webui 是"AI 绘图平民化"的最大功臣：它用约 4.4 万行 Python 把扩散模型推理包装成人人可用的界面，其 Script/Extension 双层插件机制和随图存参数的设计至今仍是同类产品的对标对象。但自 2024 年中 v1.10.1 后 master 已冻结，依赖栈明显老化，继续使用建议锁定 v1.10.1 当作稳定工具，而想读源码的人应把它当作"Gradio 重度定制 + 插件化架构 + PyTorch 工程化"三个方向的高价值教材；若要跟进新模型生态，则需转向其活跃分支或替代品。适合本地部署使用、研究扩展开发模式与回调式架构设计的开发者；不适合作为需要长期演进的二次开发基底。
