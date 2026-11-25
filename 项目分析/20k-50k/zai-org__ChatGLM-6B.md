# zai-org/ChatGLM-6B 项目分析

> 星数 40848（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：清华大学 KEG 实验室与智谱 AI 联合开源的 62 亿参数中英双语对话语言模型（基于 GLM 架构），是中文开源大模型浪潮的早期标杆项目——以 INT4 量化后 6GB 显存即可本地部署为核心卖点。

| 项目 | 内容 |
| --- | --- |
| 名称 | zai-org/ChatGLM-6B（原 THUDM/ChatGLM-6B，组织后迁移/更名至 zai-org） |
| 星数 | 40848（gh api 核实，快照 2026-10-24） |
| 语言 | Python |
| 许可证 | 代码 Apache-2.0（`LICENSE`）；模型权重单独采用 ChatGLM-6B 模型协议（`MODEL_LICENSE`），登记后允许免费商用 |
| 创建时间 | 2023-03-13（gh api） |
| 最近推送 | 2024-06-27（gh api 核实，此后停更） |
| 仓库规模 | 74 个文件条目，Python 代码约 5700 行（含 ptuning 目录），外加 README/FAQ 等文档与示例截图 |

## 2. 项目是什么，解决什么问题

ChatGLM-6B 是一个开源的、支持中英双语问答与对话的对话语言模型，基于 General Language Model (GLM) 架构，具有 62 亿参数，经过约 1T 标识符的中英双语预训练，辅以监督微调、反馈自助、人类反馈强化学习等对齐技术（`README.md` 介绍节）。它解决的痛点是 2023 年初"ChatGPT 封闭、开源界缺少能在中国消费级显卡上本地跑起来的中文对话模型"这一空白：官方给出的硬件需求表（`README.md` 硬件需求节）显示 FP16 推理需 13GB 显存，INT8 降至 8GB，INT4 仅需 6GB，意味着一张 RTX 3060 即可本地部署，INT4 量化下最低 7GB 显存还能启动 P-Tuning v2 微调。

需要特别说明仓库的形态：**模型权重与模型实现代码均不在本仓库内**。权重发布在 Hugging Face（THUDM/chatglm-6b 及 chatglm-6b-int4/int8 等量化版），模型实现（tokenizer、GLM 建模代码、`chat`/`stream_chat`/`quantize` 方法）随 HF 仓库以 `trust_remote_code=True` 方式动态加载——本仓库所有 demo 与微调脚本都是这一模式（如 `cli_demo.py` 第 7-8 行、`web_demo.py` 第 5-6 行、`utils.py` 第 41 行）。因此 GitHub 仓库实际是"驾驶舱"：四种使用入口（代码调用、Gradio/Streamlit 网页、CLI、FastAPI API）+ 微调工具链 + 详尽的中英双语文档。

2023 年 6 月起团队陆续发布升级版 ChatGLM2-6B（2K→32K 上下文、Multi-Query Attention、推理提速 42%）与 ChatGLM3-6B，2024 年 5-6 月发布 GLM-4-9B 系列；本仓库 README 顶部已放置 GLM-4 引导区（`README.md` 第 15-29 行），并在引用节挂出 ChatGLM 系列综述论文 arXiv:2406.12793。ChatGLM-6B 初代于 2024-06-27 后停止维护，属于"功成身退"的历史版本（停更状态经 gh api pushed_at 核实）。

历史地位客观陈述：`improve/README.md` 记载，模型 2023-03-14 发布后，至 4 月 22 日 GitHub 星数达 2 万、各渠道累计下载量超 100 万、连续 12 天居 Hugging Face 全球大模型下载榜第一。它是中国大模型开源生态的第一波引爆点，直接催生了 langchain-ChatGLM、chatgpt_academic 等一批下游项目（`PROJECT.md` 收录 20 余项）。

## 3. 技术栈与依赖

- 语言：纯 Python（脚本型仓库，无 setup.py/pyproject.toml，不可 pip 安装，按脚本运行）。
- 依赖清单（`requirements.txt`，仅 8 项）：
  - `transformers==4.27.1`：模型加载与推理骨架（README 称不低于 4.23.1 即可）；
  - `torch>=1.10`：底层张量计算；
  - `cpm_kernels`：提供 INT4/INT8 量化算子（GPU 量化 kernel，Mac 上不可用，只能 CPU 量化推理，见 `README.md` Mac 部署节）；
  - `gradio` + `mdtex2html`：网页 demo 与 Markdown/LaTeX 渲染；
  - `sentencepiece`、`protobuf`：tokenizer；
  - `accelerate`：多卡 device_map 分发（`utils.py`）。
- 微调额外依赖（`ptuning/README.md`）：`rouge_chinese`、`nltk`、`jieba`、`datasets`；全参数微调另需 DeepSpeed（`ptuning/deepspeed.json` 配置 ZeRO stage 2）。
- Streamlit demo（`web_demo2.py`）额外需要 `streamlit`、`streamlit_chat`；视觉 demo（`cli_demo_vision.py`）需 SwissArmyTransformer 与 torchvision（`README.md` VisualGLM 节）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `web_demo.py` | Gradio 网页对话 demo（默认入口，打字机流式效果） |
| `web_demo2.py` / `web_demo_old.py` | Streamlit 版 demo / 旧版非流式 Gradio demo |
| `cli_demo.py` | 命令行交互 demo（clear 清空、Ctrl+C 截断生成） |
| `api.py` | FastAPI 单接口部署（POST /，默认 8000 端口） |
| `cli_demo_vision.py` / `web_demo_vision.py` | VisualGLM-6B 多模态（图像理解）demo |
| `utils.py` | 多卡部署：device_map 自动切分 + `load_model_on_gpus` |
| `ptuning/` | P-Tuning v2 微调全套：训练脚本、自维护 Trainer、评测、deepspeed 配置、微调后 web demo |
| `README.md` / `README_en.md` / `UPDATE.md` / `FAQ.md` / `PROJECT.md` | 中英文档、更新日志、常见问题、生态项目收录 |
| `improve/` | Badcase 反馈计划说明 + 示例数据 `data_sample.jsonl`（99 行 jsonl） |
| `examples/` / `limitations/` / `resources/` | 能力示例截图、已知局限截图、生态图片素材 |
| `LICENSE` / `MODEL_LICENSE` | 代码 Apache-2.0；权重专用模型协议 |
| `.github/ISSUE_TEMPLATE/` | bug_report.yaml、feature_request.yml、config.yml（仅此三项，无 CI） |

组织方式评述：单层扁平脚本 + 一个微调子目录，没有任何包结构与测试目录，是典型的"模型配套仓库"形态。代码极薄（顶层 8 个脚本共约 570 行），重心在文档与使用体验；唯一成规模的代码是 `ptuning/trainer.py`（3830 行，从 HuggingFace transformers 的 Trainer 改造而来，`ptuning/trainer.py` 第 320 行起增加 `save_prefixencoder` 参数）。

## 5. 核心架构与关键实现

### 5.1 远程模型加载 + 量化推理路径

本仓库一切推理入口都遵循同一模式：`AutoModel.from_pretrained("THUDM/chatglm-6b", trust_remote_code=True).half().cuda()`，再按显存选择 `.quantize(4/8)`（`README.md` 模型量化节）。量化方法与 GLM 模型类都在 HF 侧的远程代码中，仓库侧不包含实现——这是一种"GitHub 做入口、HF 做实现+权重"的双仓联动设计。GLM 主干为 28 层 Transformer（`utils.py` 第 13 行 `num_trans_layers = 28`），加上 word_embeddings、final_layernorm、lm_head 共 30 个可切分单元；对话历史通过多轮拼接传入 `model.chat/stream_chat(tokenizer, query, history=...)`。流式接口 `stream_chat` 于 2023-03-19 加入（`UPDATE.md`），是所有 demo 打字机效果的基础。

### 5.2 多卡自动切分（utils.py）

`utils.py` 的 `auto_configure_device_map` 将 30 个单元均匀摊到 num_gpus 张卡，并把 embedding/final_layernorm/lm_head 固定在 0 号卡，注释中明确解释了动机：Linux 下 `model.device` 取 lm_head 所在设备而 Windows 取 word_embeddings 所在设备，不固定会导致 `stream_chat` 时 input_ids 与权重不在同一 device 而报错（`utils.py` 第 16-22 行）。再用 `accelerate.dispatch_model` 按 map 分发（`utils.py` 第 50 行）。这是当时缺乏成熟模型并行方案下的一段务实补丁，被 README 直接推荐给"每张卡都不够大"的用户。

### 5.3 P-Tuning v2 微调链路（ptuning/）

`ptuning/main.py` 是标准 HF Seq2Seq 微调脚本的三处关键改造：

1. 注入前缀参数：`config.pre_seq_len` 与 `config.prefix_projection` 写入 config（`ptuning/main.py` 第 107-109 行），由 HF 远程模型代码据此构建 `transformer.prefix_encoder`；
2. 量化训练：`--quantization_bit 4` 时先 `model.quantize(...)`，再把 prefix_encoder 单独转回 FP32 以保证可训练精度（`ptuning/main.py` 第 126-132 行），冻结主干、只训 0.1% 参数，配合 Gradient Checkpoint 达到 6.7GB 显存即可微调（`ptuning/README.md`）；
3. 多轮模板：训练数据按 `[Round N]\n问：{}\n答：{}` 格式拼接历史（`ptuning/main.py` 第 170-171 行），标签侧通过定位 bos（gMask）位置把 prompt 部分置 -100 只对回答计算损失（第 219-221 行）。

配套自维护 Trainer（`ptuning/trainer.py`，3830 行）在保存时按 `save_prefixencoder` 过滤出 requires_grad 参数，只落盘 PrefixEncoder 权重（`ptuning/trainer.py` 第 2830-2836 行），推理时用 `--ptuning_checkpoint` 把前缀权重灌回原模型（`ptuning/main.py` 第 113-122 行）。评测指标为 jieba 分词后的中文 Rouge-1/2/l 与 BLEU-4（`ptuning/main.py` 第 307-337 行）。训练入口 `train.sh`：PRE_SEQ_LEN=128、LR=2e-2、等效 batch 16；全参数微调走 `ds_train_finetune.sh`（DeepSpeed 4 卡、ZeRO-2，见 `ptuning/deepspeed.json`）。

### 5.4 服务化与多模态入口

`api.py` 用 FastAPI 暴露单个 POST 端点，入参 prompt/history/max_length/top_p/temperature，默认 max_length=2048、top_p=0.7、temperature=0.95，每次请求后 `torch_gc()` 清理显存（`api.py` 第 11-15、32-48 行）。`web_demo.py` 通过重写 `gr.Chatbot.postprocess` + `mdtex2html` 实现 Markdown/LaTeX 渲染，`parse_text` 处理代码块转义（其实现注明 copy 自 ChuanhuChatGPT，`web_demo.py` 第 26-56 行）。视觉 demo 加载 THUDM/visualglm-6b，`stream_chat` 签名变为 `(tokenizer, image_path, query, history)`（`cli_demo_vision.py` 第 44 行），即以图片路径为首参的多模态对话。

## 6. 构建与运行

无构建环节（非安装包），实际使用命令均来自 README/ptuning 文档：

```bash
pip install -r requirements.txt          # transformers 推荐 4.27.1

# 代码调用（自动从 HF 下载实现+权重）
python -c "from transformers import AutoTokenizer, AutoModel; \
  model = AutoModel.from_pretrained('THUDM/chatglm-6b', trust_remote_code=True).half().cuda(); \
  print(model.chat(AutoTokenizer.from_pretrained('THUDM/chatglm-6b', trust_remote_code=True), '你好', history=[]))"

# 量化加载 / 直接加载量化版
model = AutoModel.from_pretrained("THUDM/chatglm-6b", trust_remote_code=True).quantize(8).half().cuda()
model = AutoModel.from_pretrained("THUDM/chatglm-6b-int4", trust_remote_code=True).half().cuda()

python web_demo.py        # Gradio 网页版
python cli_demo.py        # 命令行版（clear/stop 控制字）
python api.py             # FastAPI，POST http://127.0.0.1:8000
python web_demo2.py       # Streamlit 版

# 多卡部署
python -c "from utils import load_model_on_gpus; m = load_model_on_gpus('THUDM/chatglm-6b', num_gpus=2)"

# P-Tuning v2 微调（ADGEN 数据集，INT4 下最低 6.7GB 显存）
cd ptuning && pip install rouge_chinese nltk jieba datasets && bash train.sh
# 全参数微调：bash ds_train_finetune.sh（DeepSpeed）
```

Mac 部署需 PyTorch-Nightly 与 MPS 后端，量化 kernel 为 CUDA 实现，Mac 上只能 CPU 推理且需另装 OpenMP（`README.md` Mac 部署节、`FAQ.md` Q1）。

## 7. 测试与工程质量

- 测试：仓库内无任何测试目录、测试文件或测试框架配置（全 74 个文件条目中无 test 相关内容，`.gitignore` 含 pytest/coverage 条目但仅是模板残留）。工程质量重心完全在"可运行性"而非可测试性。
- CI：`.github/` 下仅有 ISSUE_TEMPLATE 三个文件，无 workflows 目录，即无任何 CI/CD（基于源码树确认）。
- 代码质量观察：脚本短小、注释充分（尤其中文注释解释 Windows/Linux device 差异等踩坑点）；存在少量重复（`parse_text` 在三个 demo 文件间复制粘贴）；`requirements.txt` 将 transformers 钉在 4.27.1，对今天的环境有兼容性老化问题；微调侧 `trainer.py` 整体复制 HF Trainer 再改造，升级成本高。文档质量是本仓库最强项：中英双语对照、硬件需求量化表、局限性章节主动展示失败案例截图（`limitations/`）。

## 8. 社区与维护状态

- 活跃度：创建于 2023-03-13，最后推送 2024-06-27（gh api 核实），之后 16 个月无任何更新，处于停更但未归档（archived=false）状态。
- Release 节奏：GitHub 侧无 release/tag 发布记录（源码树未见相关配置，未在 gh api 中单独核实 releases）；模型版本走 HF 仓库 revision（v1.0、v1.1 等，`README.md` 第 162 行提及 revision="v1.1.0"）。
- 社区热度：40848 星、5078 forks、609 个开放 issue（gh api）——开放 issue 数量偏大且无人处理，与停更状态一致。
- 贡献者结构：文档中致谢了多位社区贡献者（stream_chat、多卡部署、API 部署、web 界面优化分别由社区 PR 贡献，见 `UPDATE.md`），说明早期接受 PR；Badcase 反馈计划（`improve/README.md`）尝试用众包数据驱动模型迭代，是当时少见的社区共创机制。
- 演进去向：系列后续在独立仓库演进——ChatGLM2-6B、ChatGLM3-6B、GLM-4（README 更新信息节），本仓库顶部已放置 GLM-4 引导与智谱 AI 开放平台导流（`README.md` 第 15-29 行）。组织层面 THUDM 已更名/迁移为 zai-org（仓库现归属 zai-org，gh api 核实），仓库内部链接仍保留 THUDM 旧域名。

## 9. 亮点与风险

亮点：

1. 消费级部署定位精准：INT4 量化 6GB 显存推理、7GB 微调的硬指标（`README.md` 硬件需求表）在 2023 年 3 月是中文开源模型中最低门槛，直接引爆本地部署生态。
2. 文档即产品：中英双语 README、量化对比表、局限性章节附真实失败截图、FAQ 覆盖 Mac OpenMP 等长尾问题，文档完整度远超同期开源模型仓库。
3. 使用入口全覆盖：代码调用、Gradio/Streamlit 网页、CLI、FastAPI、VisualGLM 多模态五种入口，均为百行内小脚本，易于阅读与魔改。
4. 微调工具链完整：P-Tuning v2（INT4 量化训练 + 只保存 PrefixEncoder）与 DeepSpeed 全参数微调双路径，配套中文 Rouge/BLEU 评测，形成"能跑→能改→能训"闭环。
5. 生态辐射力强：`PROJECT.md` 收录的 langchain-ChatGLM、chatgpt_academic、ChatGLM-Efficient-Tuning 等下游项目本身也都是千星级仓库，验证了其作为中文 LLM 生态"母项目"的地位。

风险/局限：

1. 已停止维护：2024-06-27 后无推送，transformers 钉在 4.27.1、依赖 cpm_kernels 等老库，在新环境（transformers 4.40+、torch 2.x）下复现有兼容成本；609 个未处理 issue。
2. 模型实现不在仓库内：`trust_remote_code=True` 动态加载 HF 侧代码，供应链上等同于执行远端代码，且仓库代码与 HF 代码版本耦合（README 需提示用 revision 固定版本）。
3. 模型能力局限明确：6B 容量导致事实性错误、数学/编程弱、英文夹杂、易被误导（`README.md` 局限性节，官方自附截图佐证）；上下文训练长度 2048，超出后性能下降。
4. 双许可证门槛：代码 Apache-2.0 但权重走自定义 MODEL_LICENSE（禁军事/非法用途、可撤销），商用需登记问卷，与纯 Apache 项目不同，集成前需法务确认。

## 10. 总结评价

ChatGLM-6B 是中文大模型开源运动的标志性起点：它未必是技术上最强的 6B 模型，但它用"INT4 量化 + 消费级显卡 + 中英双语 + 全套本地 demo/微调工具链"的组合拳，第一次让中文开发者真正普遍地拥有了"在自己电脑上跑/改/训大模型"的能力，并以此在两个月内收获 2 万星、孵化出庞大的下游生态。今天它的实用价值已被 ChatGLM2/3、GLM-4-9B 及各家后继开源模型取代，仓库本身自 2024 年年中停更，更适合作为三方面对象来读：想理解早期开源 LLM 项目如何做工程配套（入口脚本、量化部署、P-Tuning 微调、文档运营）的开发者；做中文开源模型史与生态演化研究的研究者；以及仍在维护老项目、需要迁移到 GLM 系列后继模型或官方 API 的存量用户。源码层面值得细读的是 `utils.py` 的多卡切分与 `ptuning/` 的量化微调改造——它们是"有限资源下把大模型用起来"这一命题的经典答案。
