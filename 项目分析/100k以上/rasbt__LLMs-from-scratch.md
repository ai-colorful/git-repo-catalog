# rasbt/LLMs-from-scratch 项目分析

> 星数 107147（快照 2026-10-24）｜主语言 Jupyter Notebook｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：Sebastian Raschka 所著 Manning 图书《Build a Large Language Model (From Scratch)》的官方配套代码仓库——用 PyTorch 从零手写一个类 GPT 大模型，是 GitHub 上最知名的 LLM 教学型仓库。

| 项目 | 值 | 出处 |
|---|---|---|
| 名称 | rasbt/LLMs-from-scratch | gh api |
| 星数 | 107,513（实时 API，快照 107,147） | gh api repos/rasbt/LLMs-from-scratch |
| Fork 数 | 16,577 | gh api |
| 主语言 | Jupyter Notebook（68 个 .ipynb + 150 个 .py） | 本地统计 |
| 许可证 | Apache-2.0（`LICENSE.txt`，Copyright 2023-2026 Sebastian Raschka；GitHub API 识别为 NOASSERTION，以 LICENSE.txt 为准） | LICENSE.txt |
| 创建时间 | 2023-07-23 | gh api |
| 最近 push | 2026-10-02（分析前约 4 周，高度活跃） | gh api |
| 默认分支 | main | gh api |
| 图书信息 | Manning 出版，ISBN 9781633437166，2024-09-12 发布 | CITATION.cff |

## 2. 项目是什么，解决什么问题

这是《Build a Large Language Model (From Scratch)》一书的官方代码仓库（README.md 开篇即声明），核心主张是：不依赖任何现成 LLM 库，仅用 PyTorch 基础组件，从 tokenization 一路写到预训练、指令微调，让读者"从内到外"理解 LLM 的工作原理。作者明确说明这种教学路径与大厂构建基础模型的流程同构，只是规模缩小到普通笔记本可跑的程度。

内容主线共 7 章 + 5 个附录（README.md 目录表）：Ch1 概念综述（无代码）→ Ch2 文本数据处理（BPE 分词、滑窗 dataloader）→ Ch3 注意力机制（因果掩码多头注意力）→ Ch4 从零实现 GPT 模型 → Ch5 无标注数据预训练（含加载 OpenAI GPT-2 官方权重）→ Ch6 垃圾短信分类微调 → Ch7 指令微调（含 Ollama/OpenAI 评测）。附录覆盖 PyTorch 入门（A）、参考书目（B）、习题解答（C）、训练循环增强（D）、LoRA 参数高效微调（E）。

与图书的静态内容不同，这个仓库的生命力很大程度来自持续膨胀的 bonus material（README.md "Bonus Material" 一节）：图书出版后，作者不断把业界新架构以"从零复现"的形式加进对应章节——KV Cache、GQA、MLA、SWA、Gated DeltaNet、DeepSeek Sparse Attention、MoE，以及 Llama 3.2、Qwen3（含 MoE）、Gemma 3/4、Olmo 3、Qwen3.5、Tiny Aya 等完整模型的从零实现 notebook。这让仓库从"书的配套"演化成了持续跟进 SOTA 架构的活教材。

仓库还绑定了姊妹项目生态：`reasoning-from-scratch/` 目录是续作《Build A Reasoning Model (From Scratch)》仓库的 git submodule（`.gitmodules` 声明，tarball 中为空目录属正常现象）；另有 Manning 官方 17 小时 15 分钟配套视频课程（README.md "Video Course" 一节）。

约束与定位非常明确：README.md 声明"由于仓库对应纸质书内容，不接受修改主章节代码的贡献"，以保证与印刷版一致；更新主要通过作者本人添加 bonus material 推进。

## 3. 技术栈与依赖

核心依赖（`pyproject.toml`，项目版本 1.0.18）：

- **Python >=3.10,<3.15**；**PyTorch >=2.2.2**（按 darwin-x86_64/arm64、linux、win32 分平台条件声明；darwin x86_64 限制 <2.6）
- **tiktoken >=0.5.1**：GPT-2 BPE 分词（教学代码中 `tiktoken.get_encoding("gpt2")`，见 `ch04/01_main-chapter-code/gpt.py`）
- **TensorFlow >=2.16/2.18**（仅 python <3.13 且分平台）：唯一用途是读取 OpenAI 官方 GPT-2 的 TF checkpoint 以加载预训练权重
- **jupyterlab、matplotlib、pandas、numpy、tqdm、pytest**：教学与运行环境
- dev 组：build、twine（PyPI 打包发布）、tokenizers、safetensors；bonus 组：chainlit（聊天 UI）、openai、transformers、huggingface_hub、blobfile、sentencepiece、scikit-learn、thop 等

工程化配置三套并存，服务不同读者群：

- `pyproject.toml` + uv：官方主力方案，CI 全部基于 uv；同时 `[tool.setuptools] package-dir = {"" = "pkg"}` 声明了可发布的 PyPI 包
- `pixi.toml`：conda-forge 渠道的替代方案（含 tests 特性环境，引入 nbval），README 的 setup 目录提供安装指引
- `requirements.txt`：传统 pip 入口

代码风格：ruff，line-length 140，选中 E4/E7/E9/F 规则（`pyproject.toml` [tool.ruff]）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `ch01/`–`ch07/` | 按书章节组织的代码；每章内部用编号前缀分子目录：`01_main-chapter-code/`（正文代码）、`02_bonus_*`、`03_*`…（附加材料） |
| `appendix-A/`–`appendix-E/` | 附录代码：PyTorch 入门（含 DDP 多卡脚本）、参考文献、习题解答索引、训练循环增强、LoRA |
| `setup/` | Python 环境安装指南、库安装 notebook、Docker 环境、AWS SageMaker 可选内容 |
| `pkg/llms_from_scratch/` | 可 `pip install llms-from-scratch` 的 PyPI 包，把各章核心代码整理成可 import 的模块 |
| `reasoning-from-scratch/` | 续作仓库的 git submodule（tarball 中为空，见 `.gitmodules`） |
| `.github/` | 15 个 workflow、issue 模板、链接检查脚本（`scripts/check_links.py` 及其测试） |
| `troubleshooting.md` | 常见问题手册（图片不渲染、fork 更新策略、CUDA/设备差异等） |
| `conftest.py` | 根级 pytest 配置：为链接检查请求注入超时与重试逻辑 |

组织方式评述：这是典型的"内容型 monorepo"——没有统一的 src 布局，而是"章节目录 × 编号子目录"的扁平教学结构；代码存在三种形态（notebook 主线、章节内 `*.py` 汇总脚本、pkg 包模块），三者内容同源但各自独立维护。规模统计：全仓 68 个 .ipynb、150 个 .py；ch05 最大（30 ipynb + 41 py、19 个子目录），ch04 次之（32 个 py，含大量注意力变体实现）。

## 5. 核心架构与关键实现

**（1）从零 GPT 主线（`ch04/01_main-chapter-code/gpt.py`）**。单文件按章分段收集 Ch2-4 全部构件：`GPTDatasetV1` 滑窗切分文本、`MultiHeadAttention`（`register_buffer` 缓存上三角因果掩码）、`TransformerBlock`、`LayerNorm`，最终拼成 `GPTModel`：

```python
class GPTModel(nn.Module):
    def __init__(self, cfg):
        super().__init__()
        self.tok_emb = nn.Embedding(cfg["vocab_size"], cfg["emb_dim"])
        self.pos_emb = nn.Embedding(cfg["context_length"], cfg["emb_dim"])
        self.drop_emb = nn.Dropout(cfg["drop_rate"])
        self.trf_blocks = nn.Sequential(
            *[TransformerBlock(cfg) for _ in range(cfg["n_layers"])])
        self.final_norm = LayerNorm(cfg["emb_dim"])
        self.out_head = nn.Linear(cfg["emb_dim"], cfg["vocab_size"], bias=False)
```

默认配置即 GPT-2 124M（vocab 50257 / context 1024 / emb 768 / 12 头 / 12 层，同文件 `GPT_CONFIG_124M`），配合 `generate_text_simple` 的自回归生成循环。教学取向明显：优先可读性（显式 view 拆头、逐行注释），不追求 fused kernel。

**（2）GPT→现代架构改造线（`ch05/07_gpt_to_llama/`）**。通过 `converting-gpt-to-llama2.ipynb` → `converting-llama2-to-llama3.ipynb` → `standalone-llama32.ipynb` 三步递进，逐个引入 RMSNorm、SwiGLU、RoPE 等与 GPT-2 的差异点；`tests/tests_rope_and_parts.py` 对 RoPE 等部件做单测。该模式随后被复制到 Qwen3（`ch05/11_qwen3/`，含 dense 与 MoE 的 standalone notebook）、Gemma 3/4、Olmo 3、Tiny Aya、Qwen3.5（`ch05/12_`–`17_`），形成"以 GPT-2 为基线、增量演进到当代架构"的教学方法论。

**（3）PyPI 包（`pkg/llms_from_scratch/`）**。把章节代码提炼为可导入模块（`ch02.py`…`ch07.py`、`appendix_a/d/e.py`、`llama3.py`、`qwen3.py`、`generate.py`、`utils.py`），并带 `kv_cache/`、`kv_cache_batched/` 两个子包（gpt2/llama3/qwen3 三套 KV cache 实现）。这是仓库从"书配套"走向"可复用库"的桥梁，包版本随主仓同步（1.0.18）。

**（4）权重加载与评测配套（`ch05/02_alternative_weight_loading/`、`ch07/`）**。Ch5 提供 TF checkpoint / safetensors / HuggingFace 多来源的 GPT-2 权重加载路径及内存优化加载；Ch7 提供指令微调数据生成、`ollama_evaluate.py`（用本地 Ollama 模型当裁判）与 DPO 偏好对齐（`ch07/04_preference-tuning-with-dpo/dpo-from-scratch.ipynb`），把教学闭环延伸到对齐阶段。

## 6. 构建与运行

来自 `setup/README.md` 指引与 CI 脚本的实际命令：

```bash
# 方式一：uv（官方推荐，CI 同款）
git clone --depth 1 https://github.com/rasbt/LLMs-from-scratch.git
cd LLMs-from-scratch
uv sync                              # 依据 pyproject.toml 建环境
# 方式二：pip
pip install -r requirements.txt      # 或 pip install llms-from-scratch 直接用包
# 方式三：pixi/conda（pixi.toml）
```

运行以 JupyterLab 为主：`jupyter lab` 后打开如 `ch04/01_main-chapter-code/ch04.ipynb` 顺序执行；每章 `01_main-chapter-code/` 下的汇总脚本（如 `gpt.py`、`ch05/01_main-chapter-code/gpt_train.py`、`ch06/01_main-chapter-code/gpt_class_finetune.py`）可独立 `python` 运行。README 明确"主章代码设计为普通笔记本在合理时间内可跑"，GPU 可用时自动启用。测试命令（CI 同款）：`pytest ch04/01_main-chapter-code/tests.py`、`pytest pkg/llms_from_scratch/tests/`、`pytest --nbval ch02/01_main-chapter-code/dataloader.ipynb`。

## 7. 内容规模与组织体系（内容型特例）

**收录范围与分类体系**：内容按"章节 → 编号子目录"两级组织，编号即阅读顺序（01 主线、02+ 附加），这个约定贯穿全部章节，可预期性极强。README.md 的目录表是唯一导航入口，为每章列出"Main Code（快速访问）"与"All Code + Supplementary"两档链接；Bonus Material 一节再按章聚合所有扩展专题。

**条目规模**：68 个 notebook + 150 个 Python 脚本；ch05 一个章节就有 19 个子目录，覆盖预训练调参、Gutenberg 语料预训练、学习率调度、训练提速、UI 界面（Chainlit）、7 个当代 LLM 架构复现等专题。习题解答按章内 `exercise-solutions.ipynb` 与 `appendix-C/` 双通道提供。

**与书的同步约束**：主章节 notebook 与印刷书严格一致（README "Questions, Feedback, and Contributing" 一节明示不接受改动主线代码的贡献），演进性内容全部追加在编号子目录中——这是内容型仓库里少见的"冻结主线、外挂更新"双轨制，既保图书读者体验，又不阻碍内容生长。

**姊妹项目与生态位**：上游是 Manning 图书 + 17 小时视频课程 + 170 页免费自测 PDF 的完整产品矩阵；下游接续作 `rasbt/reasoning-from-scratch`（本仓以 submodule 挂载，内容含 Qwen3 源码走读、GRPO 强化学习、MMLU/MATH-500 评测等）。在"LLM 教学"生态位中，它占据"从零实现"这一最硬核也最受欢迎的细分，与 Karpathy 的 nanoGPT（极简研究向）形成鲜明互补。

## 8. 更新机制、测试与社区维护

**更新机制**：单人主导（作者 raschka）+ 轻量外部贡献。最近 push 2026-10-02；tarball 中大量文件 mtime 晚于 2026-08，说明秋季仍在密集更新（如新增 Gemma 4、Qwen3.5、Olmo 3、DSA、KV sharing 等专题，见 README Bonus Material 与 `ch05/` 子目录）。贡献者共 68 人（gh api contributors），开放 issue 仅 6 个，维护响应极快。

**测试与 CI（对内容型仓库而言异常扎实）**：15 个 GitHub Actions workflow（`.github/workflows/`），覆盖 Linux/macOS/Windows 三平台 uv 测试、pip/pixi 兼容、最新 Python、旧版 PyTorch、PyTorch RC 前瞻、链接检查、拼写检查、ruff lint 与必需检查汇总。测试分三层：

- 各章 `tests.py`：黄金输出对比（如 `ch04/01_main-chapter-code/tests.py` 断言脚本 stdout 与期望张量逐字一致）
- `pkg/llms_from_scratch/tests/`：规范 pytest 单测，含参数化 fixture（`test_ch04.py` 的 `test_gpt_model_variants` 等）
- `--nbval` 对选定 notebook 做执行级验证；全仓测试文件 37 个

**文档质量**：README 导航完备；`troubleshooting.md` 覆盖图片托管、fork 同步、设备差异等真实痛点；每章目录有 README。唯一短板：notebook 图片外链到 sebastianraschka.com，离线环境不渲染（troubleshooting.md 第一条即为此致歉）。

## 9. 亮点与风险

**亮点**

1. 教学设计一流：滑窗 dataloader → 注意力 → GPT → 预训练 → 微调的递进链路完整，代码逐行注释、每章附习题与解答（`ch02/01_main-chapter-code/exercise-solutions.ipynb` 等）
2. "冻结主线 + 外挂 bonus"的双轨内容策略，使 2024 年的图书代码持续跟进 2026 年的 SOTA 架构（Gemma 4、Qwen3.5、DSA 等，`ch05/16_qwen3.5/`、`ch05/17_gemma4/`）
3. 对教学仓库而言罕见的工程质量：三平台 CI × 8 种环境矩阵、三层测试、链接检查、ruff（`.github/workflows/`）
4. 顺手做成了可复用库：`pkg/llms_from_scratch/` 发布 PyPI，KV cache 等实现可直接 import
5. 产品矩阵协同：书 + 视频课 + 自测 PDF + 续作仓库互相导流，README 信息密度高且无冗余

**风险/局限**

1. 教学代码刻意保守（学习式位置嵌入、非 fused 注意力、未用 flash-attention），不能直接当生产代码用——这是定位而非缺陷，但易被误用
2. notebook 图片全部外链作者站点，离线/被墙环境体验受损（troubleshooting.md 明示）
3. 单人维护 + 不接受主线 PR，图书版次之外的内容质量依赖作者个人带宽
4. TensorFlow 依赖只为读旧版 GPT-2 checkpoint 而存在，且 python >=3.13 即不可用，是环境配置中最脆的一环（`pyproject.toml` 平台条件）

## 10. 总结评价

这是"教育型内容仓库"的教科书级范本：一本畅销书 + 一套随时间生长的从零复现代码 + 异常认真的工程化保障，三者叠加造就了 10 万星。它能到这个量级，靠的不是营销而是复利——精准卡位"从零手写 LLM"这一刚需学习路径，且在书出版后两年里持续把 Qwen3、Gemma、Olmo、MoE、KV Cache、DPO 等新东西按同一教学范式补进仓库，让老内容不断获得新的搜索与推荐入口。适合三类人：想把 LLM 原理真正落到代码的学习者（顺书读 notebook）；想教 LLM 课程的教师（章节编号 + 习题体系可直接搬用）；想快速理解现代架构与 GPT-2 差异的工程师（`ch05/07_gpt_to_llama/` 及各 standalone 复现 notebook 是最快的对照材料）。想读"源码"的人则应把 pkg 包和各章汇总 .py 当作精读入口，notebook 作为讲解旁注。
