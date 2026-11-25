# karpathy/nanochat 项目分析

> 星数 58872（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：Andrej Karpathy 的"最小可行全栈 ChatGPT 克隆"——在单机 8XH100 节点上约 1.5 小时、不到 $100 训练出一个达到 GPT-2 能力水平的对话模型，并可立即用 CLI 与之对话。

| 事实 | 值 |
|---|---|
| 名称 | karpathy/nanochat |
| 定位 | "the minimal full-stack ChatGPT clone"（`pyproject.toml` 中 description） |
| 星数 | 58872（快照）；GitHub API 2026-11-03 显示 59755 |
| 主语言 | Python（约 7800 行）+ 4 个 Bash 管线脚本（384 行） |
| 许可证 | MIT（`LICENSE`，Copyright (c) 2025 Andrej Karpathy） |
| 创建 / 最近 push | 2025-10-13 / 2026-09-07（gh api） |
| 贡献者 / Fork | 51 人 / 8404（gh api）；无 release、无 tag |
| 依赖管理 | uv + `uv.lock`（锁定 torch==2.9.1） |

## 2. 项目是什么，解决什么问题

nanochat 是 nanoGPT 的精神后继。nanoGPT 只覆盖预训练，而 nanochat 把 LLM 的完整生命周期——tokenizer 训练、预训练、SFT 微调、强化学习、评测、推理对话——全部装进一个约 8 千行 Python 的单体仓库（`README.md` 自述"single, cohesive, minimal, readable, hackable"）。它的核心卖点是可复现的端到端：`runs/speedrun.sh` 一条命令完成"下载语料 → 训 BPE → 预训练 → CORE 评测 → SFT → 对话评测"全流程（`runs/speedrun.sh`）。

项目的科研主轴是 **"Time-to-GPT-2" 排行榜**：衡量在 8XH100 节点上训练出 DCLM CORE 分数超过 GPT-2（2019 年花了 $43K、168 小时）模型所需的墙钟时间。当前最好成绩 1.65 小时（2026-03-14，commit a825e63），从 2026-01 的 3.04 小时一路优化下来（`README.md` 排行榜表格、`dev/LEADERBOARD.md`）。这借鉴了 modded-nanoGPT 的"游戏化调优"思路，把整个仓库变成一个开放的速度赛马场。

工程上最重要的设计决策是**单一复杂度旋钮**：用户只需指定 `--depth`（Transformer 层数），其余所有超参（宽度、头数、学习率、训练 token 数、batch size、weight decay）都由代码按 scaling law 自动推导。训练一个 d12（GPT-1 量级）约 5 分钟，d24-d26（GPT-2 量级）约 1.5 小时，换一个数字就得到一个 compute-optimal 模型（`README.md` "Research" 一节、`scripts/base_train.py`）。

它同时是一份教学材料和研究基线：`dev/LOG.md` 是逐日实验日志（假设→实验→结论），记录了如"梯度裁剪被实验证明无效并删除"这类决策依据；README 明确了 AI 使用披露政策（PR 需声明 LLM 参与部分）。仓库还自带 `.claude/skills/read-arxiv-paper/` 的 Claude Code skill，用于读 arXiv 论文辅助开发。

## 3. 技术栈与依赖

- **语言/运行时**：Python >=3.10（`.python-version` 固定 3.10），Bash 管线脚本；tokenizer 训练核心 `rustbpe` 为 Rust 实现（`pyproject.toml`、`nanochat/tokenizer.py`）。
- **核心依赖**（`pyproject.toml`）：`torch==2.9.1`（精确锁定，通过 uv extra 区分 cu128/CPU 索引）、`rustbpe`（BPE 训练）、`tiktoken`（BPE 推理）、`pyarrow`（parquet 数据读取）、`wandb`（实验追踪）、`numpy`、`filelock`、`psutil`、`kernels`。
- **分布式**：原生 `torch.distributed` + `torchrun`，无 FSDP/DeepSpeed 等重型框架；面向单机 8 卡。
- **注意力**：Flash Attention 3，不可用时自动回退 PyTorch SDPA（`nanochat/flash_attention.py`）。
- **数据**：预训练语料为 HuggingFace 上的 `karpathy/climbmix-400b-shuffle`（6542 个 parquet shard，按需下载，`nanochat/dataset.py`）；SFT/RL 数据经 `tasks/` 自带的极简 `HubDataset`（pyarrow 直读 HF parquet 导出，替代 `datasets` 库，`tasks/common.py`）。
- **依赖哲学**：刻意不引入 HuggingFace transformers/datasets、torchao 等——FP8 训练用约 150 行自研代码替代 torchao 约 2000 行（`nanochat/fp8.py` 模块注释）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `nanochat/` | 核心库（11 个模块）：模型、优化器、tokenizer、数据、推理引擎、FP8、评测原语 |
| `scripts/` | 每个阶段一个可执行入口：tok_train/tok_eval → base_train/base_eval → chat_sft/chat_rl/chat_eval/chat_cli，加 infer_bench |
| `runs/` | 端到端管线脚本：`speedrun.sh`（复现 GPT-2）、`miniseries.sh`（d12-d26 系列）、`scaling_laws.sh`（FLOPs 预算扫描）、`runcpu.sh`（CPU/MPS） |
| `tasks/` | SFT/评测任务：GSM8K、MMLU、ARC、HumanEval（实为 Python 代码题）、SmolTalk；`common.py` 提供 TaskMixture/TaskSequence 与 HubDataset |
| `tests/` | pytest 测试（6 个文件、54 个测试函数） |
| `dev/` | 研究资产：`LOG.md` 实验日志、`LEADERBOARD.md`、scaling 分析 notebook、数据重打包参考脚本 |
| `.claude/skills/` | 开发者自用的 Claude Code skill（读 arXiv 论文） |

组织方式评述：典型的"库 + 阶段脚本"单体结构，无包管理分发意图（version 0.1.0，从不发 release）。`scripts/` 之间允许横向 import（如 `base_train.py` 复用 `base_eval.evaluate_core`、`chat_sft.py` 复用 `chat_eval.run_chat_eval`），阶段间通过 `~/.cache/nanochat` 下的 checkpoint 目录传递产物（`nanochat/checkpoint_manager.py`、`nanochat/common.py:get_base_dir`）。没有配置框架，全部用 argparse 平铺参数。

## 5. 核心架构与关键实现

### 5.1 模型：现代技巧集大成的 GPT（`nanochat/gpt.py`，555 行）

模型汇聚了 2024-2026 年 modded-nanoGPT 社区验证过的效率技巧：RoPE（无位置嵌入）、QK-Norm、untied embedding/lm_head、relu² MLP、GQA、滑动窗口注意力（默认 pattern `"SSSL"`，短窗为 1/4 上下文且按 FA3 tile 128 对齐）、logit softcap=15、ResFormer 式 value embedding（隔层注入，带输入相关 sigmoid 门控）、逐层残差标量 `resid_lambdas`/`x0_lambdas`、"smear" 门控（把前一 token 嵌入混入当前 token，廉价 bigram 信息）、"backout"（最终 norm 前减去中层残差流，去除低层特征）。全部初始化集中在 `init_weights()` 一个函数里并使用 meta device 两段式构建（先 `__init__` 造形状、再 `to_empty + init_weights`），可读性极强。

```python
# nanochat/gpt.py —— 前向主干：逐层残差缩放 + x0 回灌 + value embedding
x0 = x
backout_layer = n_layer // 2
for i, block in enumerate(self.transformer.h):
    x = self.resid_lambdas[i] * x + self.x0_lambdas[i] * x0
    ve = self.value_embeds[str(i)](idx).to(x.dtype) if str(i) in self.value_embeds else None
    x = block(x, ve, cos_sin, self.window_sizes[i], kv_cache)
    if i == backout_layer:
        x_backout = x
```

精度管理不用 `torch.amp.autocast`，而是全局 `COMPUTE_DTYPE`（按硬件自动检测，`nanochat/common.py`）+ 自定义 `Linear.forward` 中显式 `weight.to(dtype=x.dtype)`：主权重保持 fp32 供优化器使用，矩阵乘运行在 bf16/fp8（`nanochat/gpt.py:45-50`）。

### 5.2 优化器：MuonAdamW 融合实现（`nanochat/optim.py`，459 行）

单个类同时管理两种优化器：矩阵参数走 Muon（Polar Express 正交化替代 Newton-Schulz，附 NorMuon 方差缩减与 cautious weight decay），embedding/标量走 AdamW；单卡时跳过通信原语，多卡时在优化器内部做梯度同步——**刻意不使用 DDP 包装**（`dev/LOG.md` 记录了由此发现的梯度裁剪 bug 并最终删除全部裁剪代码）。两个 `@torch.compile(fullgraph=True)` 融合核消除 Python 开销，AdamW 步进函数还处理了 bf16 参数在 MPS 上的精度问题（`nanochat/optim.py:23-63`）。

### 5.3 训练管线与"深度旋钮"的自动推导（`scripts/base_train.py`，604 行）

这是仓库的灵魂。`--depth` 进入后按序推导：模型维度 `= depth × 64` 向上取整到 head_dim=128 的倍数 → 用 scaling law 定训练 token 数（`target_tokens = ratio × (transformer_matrices + lm_head 参数)`）→ 按 Power Lines 论文（arXiv:2505.13738）`B ∝ D^0.383` 定总 batch size（取 2 的幂）→ 学习率按 `√(B/B_ref)` 缩放 → weight decay 按 T_epoch 框架（arXiv:2405.13698）缩放。所有规律以 d12 为参考模型标定，muP 式外推到其他深度（`scripts/base_train.py:259-320`）。

```python
# scripts/base_train.py —— 从深度自动推导 compute-optimal 超参
d12_ref = build_model_meta(12)                       # 参考模型，超参在此标定
D_REF = args.target_param_data_ratio * get_scaling_params(d12_ref)
B_REF = 2**19                                        # d12 最优 batch ≈ 524,288 tokens
batch_size_ratio = target_tokens / D_REF
predicted_batch_size = B_REF * batch_size_ratio ** 0.383   # Power Lines 论文
total_batch_size = 2 ** round(math.log2(predicted_batch_size))
batch_lr_scale = batch_ratio ** 0.5                  # AdamW/Muon: η ∝ √(B/B_ref)
```

FP8 训练为可选项（`--fp8`，仅 H100+）：自研 `Float8Linear` 过滤掉维度不合 16 的层，评测时用 context manager 换回 bf16 的 `Linear`（`scripts/base_train.py`、`nanochat/fp8.py`）。

### 5.4 全流程脚本与后训练（`runs/speedrun.sh`、`scripts/chat_sft.py`、`scripts/chat_rl.py`）

`speedrun.sh`（79 行）是完整复现入口：下载 8 个 shard 先训 tokenizer（vocab 2^15=32768），后台并行下载 170 个 shard，预训练 d24（`--target-param-data-ratio=8` 轻度过训），CORE 评测，SFT，对话评测。SFT（`scripts/chat_sft.py`）用 SmolTalk+GSM8K+MMLU 混合教对话模板与工具调用；tokenizer 内置 9 个特殊 token（`<|user_start|>`、`<|python_start|>` 等，`nanochat/tokenizer.py`）。RL（`scripts/chat_rl.py`）是"打了引号的 GRPO"——删掉信任域与 KL 正则、去掉 PPO clip、用 DAPO 式 token 级归一化，实际近似 REINFORCE，只在 GSM8K 上做可验证奖励。推理侧 `nanochat/engine.py` 提供带 KV cache 的引擎（滑窗层只读窗口内 cache）与计算器工具；`nanochat/execution.py` 用"子进程 + rlimit 内存限制 + 禁危险函数"的轻量沙箱执行模型生成的 Python（文档自述非真正安全沙箱）。数据加载采用 BOS 对齐 bestfit 打包：每行以 BOS 开头、文档按 best-fit 装箱、无 padding 但约 35% token 被截断（`nanochat/dataloader.py`）。

## 6. 构建与运行

```bash
# 安装（uv 管理，二选一）
uv sync --extra gpu          # CUDA (A100/H100)
uv sync --extra cpu          # CPU / MPS
source .venv/bin/activate

# 完整流程：8XH100 节点约 1.5 小时训练出 GPT-2 级模型并评测
bash runs/speedrun.sh

# 与自己的模型对话
python -m scripts.chat_cli            # 交互式
python -m scripts.chat_cli -p "Why is the sky blue?"

# 研究迭代：5 分钟级小模型（d12），关闭中间采样/保存
OMP_NUM_THREADS=1 torchrun --standalone --nproc_per_node=8 -m scripts.base_train -- \
    --depth=12 --run="d12" --core-metric-every=999999 --sample-every=-1 --save-every=-1

# 其他入口
torchrun --standalone --nproc_per_node=8 -m scripts.chat_sft    # SFT
python -m scripts.chat_rl                                       # RL (GSM8K)
bash runs/miniseries.sh                                         # d12-d26 系列
bash runs/runcpu.sh                                             # CPU/MPS 缩小版

# 测试
uv sync --extra gpu --group dev && pytest -m "not slow"
```

（来源：`README.md` Getting started/Research 节、`pyproject.toml` pytest 配置）

## 7. 测试与工程质量

- **测试**：`tests/` 6 个文件共 54 个测试函数，pytest 框架（`pyproject.toml` 配置 testpaths 与 `slow` marker）。覆盖注意力回退路径（FA3 vs SDPA 数值对照，372 行最大测试文件）、KV cache 推理引擎、tokenizer 往返与 chat 渲染、MuonAdamW（标记为需 CUDA，`tests/test_optim.py:16`）、沙箱执行的恶意输入参数化用例（`tests/test_execution.py:41`）、任务切片与 HubDataset。
- **CI**：仓库内**没有** `.github/` 目录（tarball 与 gh 文件树均确认），即无 GitHub Actions——测试靠开发者本地跑，这是个人研究仓库的典型取舍。
- **代码质量观察**：注释密度异常高且多为"解释为什么"（如 FLOPs 估算公式逐项对照 Chinchilla 论文、fp8.py 解释 `torch._scaled_mm` 原理）；关键数值决策都有出处引用（arXiv 编号）；`dev/LOG.md` 以假设→实验→结论格式记录了每个默认超参的来历，是罕见的可审计研究代码库。负面点：无类型标注、无 lint 配置、脚本间横向 import 依赖 `python -m` 的运行方式。

## 8. 社区与维护状态

- **活跃度**：创建于 2025-10-13，最近 push 2026-09-07（gh api），持续活跃约一年；135 个 open issue，51 名贡献者，8404 fork。星数增长极快（一年 6 万），属于 Karpathy 个人品牌 + 教学稀缺题材的双重效应。
- **贡献结构**：核心开发明显由 Karpathy 主导，README 致谢中提到 "repo czar" @svlandeg 负责社区管理；排行榜已有外部贡献（如 @ddudek 的数据集切换，`README.md` 排行榜第 4 行）。 Discussions 是主要交流场所（README 提供 5 个 guide 链接 + Discord 频道）。
- **release 节奏**：无 release、无 tag、无 CHANGELOG——代码演进直接体现在 master 与排行榜 commit 列中。排行榜本身（3.04h → 1.65h，2026-01 至 2026-03）就是事实上的版本历史。
- **研究方向**：`dev/LOG.md` 最近条目（2026 年初）聚焦 Muon 的 Polar Express/NorMuon 改进与 weight decay 深度缩放律；数据集从 fineweb 切换到 NVIDIA ClimbMix。

## 9. 亮点与风险

**亮点**
1. 端到端完备且可复现：一条 `speedrun.sh` 从原始语料到可对话模型，成本 <$100，是公开可得的最低门槛全栈 LLM 基线（`runs/speedrun.sh`）。
2. 单旋钮设计 + scaling law 自动调参：`--depth` 一个整数推导全部超参，代码即论文（`scripts/base_train.py:259-320`）。
3. 研究过程完全透明：`dev/LOG.md` 实验日志 + `dev/LEADERBOARD.md` 方法文档 + Discussions guide，几乎每个默认值可追溯到实验证据。
4. 极限精简的关键组件：FP8 训练 150 行替代 torchao 2000 行、HubDataset 替代 HF datasets、Muon/AdamW 融合单类双优化器——适合逐行学习。
5. 模型侧集成了 modded-nanoGPT 社区近两年几乎所有被验证的效率技巧，且每个都有开关或注释（`nanochat/gpt.py`）。

**风险/局限**
1. 无 CI、无 release：HEAD 可能随时变化，排行榜 commit 与 master 未必对应；外部使用者需自行锁 commit。
2. 硬件假设强：主路径面向 8XH100/A100，CPU/MPS 路径明确"得不到有意义的结果"（`README.md`、`runs/runcpu.sh`）；<80GB 显存需手调 `--device-batch-size`。
3. 规模上限：单机 DDP、无 FSDP，训练目标限定在 ~4e19 FLOPs（d20-d26）；不适合想训更大模型的人。
4. 沙箱与安全边界需知：`execution.py` 自述"not a true security sandbox"，网络未封锁，不能用于执行不可信代码。

## 10. 总结评价

nanochat 是"教育价值与科研价值罕见地统一"的仓库：它把一个 ChatGPT 克隆的完整技术栈压缩进约 8 千行高度注释的 Python，用 scaling law 把复杂度收敛到一个 `--depth` 旋钮，再用排行榜把优化过程变成社区可参与的竞赛。它不是生产框架，而是移动的教科书 + 可复现的强基线——适合三类人：想完整理解 LLM 从 tokenizer 到 RL 每一步的学习者（通读 `nanochat/` + `scripts/` 即可）、有 8XH100 预算想复现/冲击"Time-to-GPT-2"排行榜的研究者、以及需要一个小而干净的最优传输基线来做消融实验的 LLM 效率研究者。对想直接部署生产级对话模型的工程师，它太小；但对想看清现代 LLM 训练全貌的人，目前几乎没有更好的开源标本。
