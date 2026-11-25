# chenfei-wu/TaskMatrix 项目分析

> 星数 20000（快照 2026-10-24，gh api 2026-11-21 实测 33885）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：微软研究院出品的 Visual ChatGPT 论文配套实现——用一个单文件 Python 程序把 ChatGPT（LangChain Agent）和 22 个视觉基础模型（扩散/检测/分割/描述）拼成一个可收发图片的 Gradio 聊天机器人；TaskMatrix.AI 论文中 "millions of APIs" 的宏大愿景在此仓库只落在了一个视觉域 demo 上。

| 项目 | 内容 |
|---|---|
| 名称 | chenfei-wu/TaskMatrix（曾用名 microsoft/visual-chatgpt，后改名 TaskMatrix 并转移到作者个人账号；gh api 证实 `microsoft/TaskMatrix` 与 `microsoft/visual-chatgpt` 均重定向至此仓库） |
| 星数 | 快照 20000，2026-11-21 实测 33885 |
| 语言 | Python（单文件为主，Python 100%） |
| 许可证 | MIT（Copyright (c) 2023 Microsoft，`LICENSE.txt` 末尾附加微软免责声明，gh api 显示 NOASSERTION 即因附加条款） |
| 创建时间 | 2023-03-02 |
| main 最后提交 | 2023-06-29（Update LICENSE.txt / README.md） |
| 最后 push | 2024-01-06（gh api pushed_at，此后无任何活动） |
| archived | 否，但实质停更近 3 年；open issues 积压 262 个 |
| release/tag | 0 release、0 tag，共 117 commits、20 contributors |

## 2. 项目是什么，解决什么问题

2023 年 3 月，微软亚研团队（吴晨飞、段楠等）发布 Visual ChatGPT 论文（arXiv:2303.04671），两周内引发爆发式关注；随后 3 月底再发 TaskMatrix.AI 论文（arXiv:2303.16434）。本仓库即 Visual ChatGPT 的官方代码，仓库后来更名为 TaskMatrix 并挂上 TaskMatrix.AI 子目录（`TaskMatrix.AI/README.md`，仅 14 行，描述论文愿景）。

它解决的问题：LLM 是纯文本模型，不能"看"也不能"画"。Visual ChatGPT 的方案是把 ChatGPT 当控制器，把 22 个视觉基础模型（Stable Diffusion、ControlNet、BLIP、InstructPix2Pix、CLIPSeg、SAM、GroundingDINO 等）封装成 LangChain Tool，由 ChatGPT 用 ReAct 格式（Thought/Action/Action Input/Observation，见 `visual_chatgpt.py:62-91` 的 prompt 模板）逐步调用，从而在聊天中完成文生图、图生图、编辑、去背景、扩图、检测分割等操作。

关键设计是"文件名链路追踪"：每张图都是一个真实磁盘文件（`image/xxx.png`），每次工具输出都通过 `get_new_image_name()`（`visual_chatgpt.py:231-244`）生成带 uuid + 操作名 + 出链的新文件名，prompt 中反复强调"严格忠于文件名、绝不伪造"，让 LLM 用字符串引用图像状态而不必真的读像素。

仓库同时塞进了一个姊妹子项目 LowCodeLLM（论文 arXiv:2304.08103）：Planning LLM 生成结构化工作流、用户以低代码方式编辑、Executing LLM 执行。README 自己承认"由于时间限制，只提供最小可行版本"（`LowCodeLLM/README.md:15`）。

愿景与实现的差距需要客观记录：TaskMatrix.AI/ 目录没有任何代码，"Online System: In progress, coming in the near future..."（`TaskMatrix.AI/README.md:8`）至今没有下文；论文里的 millions-of-APIs（任务规划器 + API 检索器 + 数百万外部 API）在仓库中不存在，实际只有 22 个本地视觉模型类。

## 3. 技术栈与依赖

来自 `requirements.txt`（33 项，关键如下）：

- **Agent 编排**：langchain==0.0.101（版本钉死在 2023-03 的早期 API，`initialize_agent`/`Tool`/`ConversationBufferMemory` 均为已废弃的旧路径，见 `visual_chatgpt.py:29-32`）；openai（OpenAI completion API）
- **扩散模型**：diffusers（StableDiffusionPipeline、Inpaint、InstructPix2Pix、ControlNet 管线）、accelerate、safetensors
- **视觉模型**：transformers（BLIP caption/VQA、CLIPSeg、UperNet 语义分割）、controlnet-aux（Openpose/MLSD/HED 检测器）、timm、opencv-python
- **检测/分割（git 直装）**：GroundingDINO 与 segment-anything 不在 requirements 中，需 `pip install git+...`（`README.md:67-68`）
- **UI**：gradio（Blocks 界面）、streamlit（LowCodeLLM）
- **底座**：torch==1.13.1 / torchvision==0.14.1（CUDA 时代 2023 上半年）

## 4. 目录结构与代码组织

全仓仅 36 个文件，是典型的"论文 demo"体量：

| 路径 | 职责 |
|---|---|
| `visual_chatgpt.py` | 全部核心：1583 行单文件，26 个类（22 个视觉模型类 + 4 个模板类 + ConversationBot + 工具函数） |
| `requirements.txt` | 依赖清单（钉死 langchain==0.0.101、torch==1.13.1） |
| `README.md` | 快速开始、22 个模型的 GPU 显存表（4-9GB 不等）、中英双语支持说明 |
| `LICENSE.txt` | MIT + 微软附加免责声明 |
| `TaskMatrix.AI/` | 仅 1 个 14 行 README.md，零代码 |
| `LowCodeLLM/` | 姊妹项目：375 行 Python（planningLLM.py 104 行最大）+ Dockerfile + Flask/HTML 前端 + 3 个 pytest 文件 |
| `assets/` | 8 个演示 gif/png/mp4 |
| `.github/`、CI 配置 | 不存在（无任何 CI） |

组织方式评述：极端单体。所有逻辑压在一个 1583 行的文件里，靠"一个类 = 一个工具"的约定维持可读性；没有包结构、没有模块拆分、没有抽象层。对研究 demo 而言可行，但对号称 "AI 生态" 的项目来说，工程骨架与论文愿景完全不成比例。

## 5. 核心架构与关键实现

### 5.1 ConversationBot：globals() 即注册表（visual_chatgpt.py:1465-1516）

模型注册不用任何显式 registry，直接反射模块全局命名空间：

```python
self.models = {}
for class_name, device in load_dict.items():
    self.models[class_name] = globals()[class_name](device=device)

# 模板类自动发现与依赖注入
for class_name, module in globals().items():
    if getattr(module, 'template_model', False):
        template_required_names = {k for k in inspect.signature(module.__init__).parameters.keys() if k!='self'}
        loaded_names = set([type(e).__name__ for e in self.models.values()])
        if template_required_names.issubset(loaded_names):
            self.models[class_name] = globals()[class_name](
                **{name: self.models[name] for name in template_required_names})
```

命令行 `--load "Text2Image_cuda:0,ImageCaptioning_cpu"`（`visual_chatgpt.py:1559-1562`）解析成 `{类名: 设备}` 字典；ImageCaptioning 是强制项（1469-1470 硬校验，因为上传图片必须先自动打标）。工具收集同样反射：遍历每个实例的 `inference*` 方法，用 `@prompts` 装饰器挂上的 name/description 元数据（149 行定义的装饰器）构造 LangChain `Tool`（1488-1493）。`template_model = True` 类属性 + `inspect.signature` 依赖注入是全仓最精巧的设计：模板类只要构造函数参数名写对，就能在所需基础模型已加载时自动组合注册，无需额外配置。

### 5.2 Agent 会话与记忆（visual_chatgpt.py:1497-1526）

LangChain `conversational-react-description` agent + `ConversationBufferMemory`。会话历史用按词数截断的朴素方案：

```python
def cut_dialogue_history(history_memory, keep_last_n_words=500):
    ...
    paragraphs = history_memory.split('\n')
    last_n_tokens = n_tokens
    while last_n_tokens >= keep_last_n_words:
        last_n_tokens -= len(paragraphs[0].split(' '))
        paragraphs = paragraphs[1:]
```

（`visual_chatgpt.py:215-228`）`run_text` 把 agent 输出里的 `image/*.png` 正则替换成 Gradio 的 `![](file=...)` 内联图（1522）。`run_image`（1528-1552）上传图片先缩放到 64 倍数、≤512px，用 BLIP 自动打标，再把"文件名 + 描述"以固定话术注入 memory，防止 LLM 凭空想象图像内容。中英双语通过三段 prompt 常量切换（48-136 行，中文版要求"工具参数用英文、聊天回复用中文"）。

### 5.3 模板类：组合即工具（visual_chatgpt.py:1184-1282, 1357-1417）

4 个模板类（`InfinityOutPainting`、`ObjectSegmenting`、`ImageEditing`、`BackgroundRemoving`，grep `template_model = True` 共 4 处）。以 `InfinityOutPainting` 为例：组合 ImageCaptioning + Inpainting + VisualQuestionAnswering 三个基础模型，`dowhile()` 循环"裁边 15px → 白底扩边 → inpaint 50 步 → 高斯羽化融合（blend_gt2pt，158 行）"直到目标分辨率，实现任意尺寸无限扩图且零训练。`ImageEditing` 则串起 GroundingDINO（定位框）→ SAM（掩码）→ SD inpaint（重绘）三段管线（1380-1417），这是 README 更新里社区贡献者 jordddan 带来的 2023-05 增强。

### 5.4 模型类统一模式与模型分发（visual_chatgpt.py:246-801 等）

22 个基础模型类全部同一个模式：`__init__` 里 `from_pretrained` 拉权重 + `@prompts(name, description)` 装饰 `inference()` + 返回新文件名。模型分发两条路：

- HuggingFace Hub：`runwayml/stable-diffusion-v1-5`、`Salesforce/blip-*`、`fusing/stable-diffusion-v1-5-controlnet-*`、`timbrooks/instruct-pix2pix`（252-308、349-780 行各处）
- 外链 wget 直下：SAM 权重 `dl.fbaipublicfiles.com/segment_anything/sam_vit_h_4b8939.pth`（`visual_chatgpt.py:807-820`）；GroundingDINO 权重取 GitHub release、配置取 raw.githubusercontent.com（1029-1045 行）。仓库本身不含任何权重。

### 5.5 Gradio 界面（visual_chatgpt.py:1555-1583）

约 30 行：语言单选 → 初始化 agent → 文本框/上传按钮/清除按钮三件套，监听 0.0.0.0:7861。无鉴权、无会话隔离，单用户本地演示定位。

## 6. 构建与运行

来自 `README.md:50-99`（无构建，纯运行）：

```bash
conda create -n visgpt python=3.8 && conda activate visgpt
pip install -r requirements.txt
pip install git+https://github.com/IDEA-Research/GroundingDINO.git
pip install git+https://github.com/facebookresearch/segment-anything.git
export OPENAI_API_KEY={Your_Key}
# CPU 最低配置：仅描述+文生图
python visual_chatgpt.py --load ImageCaptioning_cpu,Text2Image_cpu
# 1x T4 (Colab)
python visual_chatgpt.py --load "ImageCaptioning_cuda:0,Text2Image_cuda:0"
```

LowCodeLLM 走 Docker：`docker build -t lowcode:latest . && docker run -p 8888:8888 --env OPENAIKEY=... lowcode:latest`（`LowCodeLLM/README.md:26-33`）。

## 7. 测试与工程质量

- 主项目 visual_chatgpt.py：零测试、零类型注解、无 CI（仓库无 `.github/` 目录），错误处理靠 print。
- LowCodeLLM 子项目：3 个 pytest 文件共 55 行（`LowCodeLLM/src/test/test_execute.py` 等），用 JSON 用例驱动的冒烟测试。
- 安全响应有痕迹：2023-05 有社区 PR 修 "avoid information exposure through exception" 和升级 flask 2.2.5（gh api commits 记录）；`SECURITY.md` 存在但本项目无发布渠道可跟进。
- 代码质量观察：单文件 1583 行、重复样板多（22 个 ControlNet 变体类结构几乎复制粘贴）；`Image.ANTIALIAS` 等在新版 Pillow 中已移除；langchain==0.0.101 与任何现行版本不兼容。按 2026 年标准，此仓库已无法开箱运行。

## 8. 社区与维护状态

- **时间线（gh api 逐条核实）**：2023-03-02 建仓（微软组织名下）→ 2023-03 论文发布后数周内星数爆发（20k 档快照即来自该时期）→ 2023-05-31 最后一次功能性合并（PR #408，GroundingDINO/SAM）→ 2023-06-29 main 分支最后提交（仅改 LICENSE/README）→ 2024-01-06 最后一次 push（非代码功能）→ 至今零活动，未 archive 但实质弃养，262 个 issue 无人回应。
- **贡献者**：20 人（gh api contributors 分页核实）。头部：chenfei-wu 33 commits、jordddan 17（检测/分割增强）、nanduan 13（通讯作者段楠）、dawnmsg 9、shengming-yin 9（InfinityOutPainting 模板）、BinayakJha 8。典型"1-2 个核心作者 + 少量一次性 PR"的研究仓结构。
- **仓库归宿**：gh api 证实 `microsoft/TaskMatrix` 与 `microsoft/visual-chatgpt` 均重定向到 `chenfei-wu/TaskMatrix`——仓库已从微软组织转移到第一作者个人账号（转移时间未确认），README 中的 clone 地址仍写着 microsoft/TaskMatrix，未更新。
- **作者去向**：gh api users/chenfei-wu 显示 Chenfei Wu 现供职 Tongyi Lab, Alibaba，bio 为 "Qwen's Painter now."，研究方向为视觉生成（联合建模感知与生成）；其个人仓库 2024 年后仅有 fork 活动（Genesis、HunyuanVideo 等），TaskMatrix 是其名下唯一星数上万的原创仓库。即：作者已离开微软并转向阿里通义千问的图像生成方向，此研究线就此断档。
- **星数与维护落差**：33885 星 / 最后功能提交 2023-05 / 0 release——星数记录的是 2023 年 3 月那波 "LLM 编排一切" 热潮的历史地位，而非当前可用性。

## 9. 亮点与风险

**亮点**
1. 用 1583 行单文件把 "LLM 作为控制器 + 领域模型作为工具" 的 agent 范式讲得最清楚的教科书实现之一（`visual_chatgpt.py`），是 2023 年 agent/tool-use 浪潮的源头代码之一。
2. `template_model = True` + `inspect.signature` 的模板依赖注入机制（1477-1484 行）：新增组合工具只需写一个类，加载器自动组装，扩展成本极低。
3. 文件名链路追踪设计（`get_new_image_name`，231-244 行）：用 uuid+操作名串联的文件名序列让纯文本 LLM 可靠地管理图像状态，规避了多模态传递的复杂性。
4. 精细化显存调度：`--load` 参数按"模型_设备"粒度分配 GPU（1559 行），README 附 22 个模型逐个显存实测表（102-127 行），对当时的消费级/Colab 用户非常友好。
5. 社区模板贡献路径真实有效：InfinityOutPainting、ImageEditing（GroundingDINO+SAM）均由外部贡献者以纯代码合入、无需改框架。

**风险/局限**
1. 实质停更近 3 年：最后功能提交 2023-05-31，262 个 open issue 无人处理，仓库未 archive 但已弃养。
2. 依赖全面腐化：langchain==0.0.101（现行 API 不兼容）、torch 1.13.1、`Image.ANTIALIAS`（Pillow 10 移除）、`runwayml/stable-diffusion-v1-5`（该 HF 组织权重已不可用，需替换镜像源）——2026 年照 README 运行必失败。
3. 愿景与实现落差巨大：TaskMatrix.AI 论文的 millions-of-APIs 架构（任务规划器/API 检索/物理域执行）在本仓库零实现，`TaskMatrix.AI/` 目录只有 14 行 README；LowCodeLLM 亦自认"最小可行版本"。
4. 工程面薄弱：无测试无 CI，Gradio 端口 0.0.0.0 无鉴权，OpenAI key 明文环境变量，异常直接 print，不适合任何生产用途。

## 10. 总结评价

TaskMatrix 是 2023 年上半年 "LLM 编排基础模型" 浪潮的标志性遗迹：微软团队用一个 1583 行的单文件演示了 ChatGPT + 22 个视觉模型的可能性，论文愿景宏大（millions of APIs），仓库现实极小（36 个文件），热潮过后三个月即停止功能开发，仓库被转移到已跳槽阿里通义实验室的第一作者名下，至今未 archive 也未维护。它的星数（33885）衡量的是历史影响力——agent-with-tools 范式的启蒙代码之一——而非今天的可用性：依赖钉死在 2023-03，按现状已无法运行。适合两类人读源码：一是想理解 LangChain 早期 agent/Tool/模板注册机制的读者（5.1 节的 globals 反射注册表与依赖注入至今仍是简洁优雅的设计样本）；二是研究 2023 年 LLM 应用爆发期技术叙事的人。想找可用的视觉对话系统，应看其精神后继者（如各类 multimodal agent 框架）而非本仓库。
