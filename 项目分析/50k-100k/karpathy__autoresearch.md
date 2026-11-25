# karpathy/autoresearch 项目分析

> 星数 98461（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：给 AI coding agent 一个真实但小型化的 LLM 预训练环境，让它整夜自主迭代实验（改代码→训练 5 分钟→按指标决定保留或回滚），把"做研究"这件事本身交给 agent —— 这是 Karpathy 2026 年 3 月发布的"自动化研究起源故事"演示仓库。

| 项目 | 值 |
|---|---|
| 名称 | karpathy/autoresearch |
| 星数 | 98461（快照）；GitHub API 2026-10-30 实测 98881 |
| 主语言 | Python（仓库全部代码为 Python + Markdown） |
| 许可证 | README.md 末尾声明 MIT；仓库内无 LICENSE 文件，GitHub API license 字段为 null |
| 最近活跃 | 最后 push/commit 2026-03-26（GitHub API，最新提交 228791f，PR #342 merge） |
| 创建时间 | 2026-03-06（GitHub API）——发布约三周即冲到近 10 万星 |
| 默认分支 | master；无 release、无 tag（GitHub API 返回 0） |
| 规模 | 仅 3 个核心文件：prepare.py（389 行）、train.py（631 行）、program.md（114 行） |

## 2. 项目是什么，解决什么问题

autoresearch 回答的问题是：如果把"LLM 研究员"这个角色本身交给 AI agent，会发生什么？它提供了一个最小但完整的实验闭环（README.md）：一个单 GPU 的 GPT 训练脚本（nanochat 的简化版），固定 5 分钟训练时间预算，以 val_bpb（验证集 bits per byte，越低越好）为唯一指标。agent 循环执行"改 train.py → 训练 → 比较指标 → 改进则保留（advance 分支）、否则 git reset 回滚"，一夜可跑约 100 个实验（每小时约 12 个）。

项目最有意思的是角色倒转（README.md "Design choices" 与 program.md）：人类研究者不再碰 Python 代码，而是编写 `program.md` —— 一份给 agent 的 Markdown 指令（作者称之为"超轻量 skill"），用它定义实验规则、指标、日志格式（results.tsv 五列 TSV）、keep/discard/crash 判据与"永不停止"的自主循环。也就是说，人优化的是"研究组织的代码"，agent 优化的是"模型代码"。作者明确表示仓库自带的 program.md 只是 bare bones 基线，鼓励迭代出更高效的"研究组织"。

这套设计同时是一个精心构造的科学实验台：固定时间预算使任意两次实验（无论改了架构、优化器还是 batch size）都直接可比（README.md）；val_bpb 按字节归一，与词表大小无关，因此架构级改动（如换词表）也能公平比较（prepare.py evaluate_bpb 注释）。仓库附带的 `analysis.ipynb`（11 个单元格，pandas/matplotlib）用于事后分析 results.tsv，绘制 kept 实验的 val_bpb 前沿曲线。

需要强调：这不是一个"框架"或"库"，而是一个可运行的范例 + 一份可编辑的 agent 剧本。它的星数更多来自范式的启发性（Karpathy 效应），代码本身的复用价值集中于 train.py 里那套现代化的单 GPU 训练栈。

## 3. 技术栈与依赖

全部来自 `pyproject.toml`（Python >= 3.10，`.python-version` 为 3.10，uv 管理，uv.lock 锁定）：

- **torch==2.9.1**（cu128 专用 index）：核心框架，模型、训练、`torch.compile`
- **kernels>=0.11.7**（HF kernels）：运行时拉取 FlashAttention-3 内核 —— Hopper（H100）用 `varunneal/flash-attention-3`，非 Hopper 回退 `kernels-community/flash-attn3`（train.py 第 20-24 行）
- **rustbpe>=0.1.0**：Rust 实现的 BPE 分词器训练（prepare.py）
- **tiktoken>=0.11.0**：将 rustbpe 训练出的 merges 封装为 tiktoken Encoding
- **pyarrow>=21.0.0**：读取 HuggingFace parquet 数据分片
- **requests>=2.32.0**：下载数据分片（多进程 + 重试）
- **numpy / pandas / matplotlib**：仅用于 analysis.ipynb 结果分析
- 运行时硬性要求：单张 NVIDIA GPU（README 注明仅在 H100 测试）

## 4. 目录结构与代码组织

仓库是极致的单层平铺结构（tarball 顶层全部文件）：

| 路径 | 职责 |
|---|---|
| `prepare.py` | 固定常量、一次性数据准备（下载分片 + 训练 BPE 分词器）、运行时工具（dataloader、evaluate_bpb）。**agent 不可修改** |
| `train.py` | GPT 模型、MuonAdamW 优化器、训练循环。**agent 唯一可修改的文件** |
| `program.md` | agent 指令剧本（实验循环、日志格式、keep/discard 规则）。**人类迭代的文件** |
| `analysis.ipynb` | results.tsv 的 pandas/matplotlib 事后分析（kept 曲线、top hits、汇总统计） |
| `pyproject.toml` / `uv.lock` | 依赖声明与锁文件 |
| `README.md` | 项目说明、快速开始、小算力调参指南、notable forks 列表 |
| `progress.png` | 题图（实验进度曲线） |
| `.gitignore` / `.python-version` | 忽略 results.tsv、results/、queue/、dev/ 等实验产物 |

组织方式评述：这是"三个文件定乾坤"的刻意设计——prepare.py（只读的裁判）、train.py（唯一变量）、program.md（人类与 agent 的接口契约）。无包结构、无 CLI 参数（train.py 超参数直接改源码顶部常量，train.py 第 429-451 行注释"edit these directly, no CLI flags needed"）。`.gitignore` 里的 results/、queue/ 暗示作者自己在跑多 agent 并行实验的变体，但相关脚本未入库。

## 5. 核心架构与关键实现

### 5.1 固定评测协议（prepare.py）

prepare.py 顶部锁定三个常量（第 30-32 行）：`MAX_SEQ_LEN=2048`、`TIME_BUDGET=300` 秒、`EVAL_TOKENS=40*524288`。数据来自 HuggingFace `karpathy/climbmix-400b-shuffle`（共 6542 个 parquet 分片，默认只下载 10 个训练分片，第 06542 号分片被固定为验证集，第 41-44 行）。指标 `evaluate_bpb`（第 344-365 行）把每个 token 的交叉熵（nats）按其 UTF-8 字节长度加权求和后换算为 bits/byte，特殊 token 权重为 0 被排除：

```python
@torch.no_grad()
def evaluate_bpb(model, tokenizer, batch_size):
    token_bytes = get_token_bytes(device="cuda")
    val_loader = make_dataloader(tokenizer, batch_size, MAX_SEQ_LEN, "val")
    steps = EVAL_TOKENS // (batch_size * MAX_SEQ_LEN)
    total_nats = 0.0
    total_bytes = 0
    for _ in range(steps):
        x, y, _ = next(val_loader)
        loss_flat = model(x, y, reduction='none').view(-1)
        y_flat = y.view(-1)
        nbytes = token_bytes[y_flat]
        mask = nbytes > 0
        total_nats += (loss_flat * mask).sum().item()
        total_bytes += nbytes.sum().item()
    return total_nats / (math.log(2) * total_bytes)
```

dataloader（`make_dataloader`，第 276-337 行）是 BOS 对齐的 best-fit 装箱：每行以 BOS 开头，优先放入能整段装下的最长文档，装不下时裁剪最短文档精确填满，实现 100% 利用率（无 padding），预分配 pin_memory CPU 缓冲 + 非阻塞拷贝到 GPU。分词器用 rustbpe 训练（GPT-4 风格 split pattern，数字按 1-2 位切分），导出 tiktoken pickle，并预生成 `token_bytes.pt` 查表供 BPB 计算（第 141-196 行）。

### 5.2 现代化 GPT 训练栈（train.py）

模型是 2025 年代技巧的集合体（train.py 第 32-291 行）：pre-norm RMSNorm、RoPE、QK-norm（对 q/k 做 RMSNorm，第 91 行）、ReluSquared 激活 MLP（`F.relu(x).square()`，第 107 行）、"SSSL" 滑窗注意力模式（3 层半窗 + 1 层全窗，最后一层强制全窗，第 195-206 行）、logit softcap 15（第 282-285 行）。两个特色设计：

- **Value Embeddings（ResFormer 风格）**：隔层放置 `nn.Embedding(vocab, kv_dim)`，通过输入门控 `2*sigmoid(ve_gate(x[...:32]))` 混入 V（第 84-87 行）；门控零初始化使 sigmoid(0)*2=1 保持中性（init_weights 第 170-173 行）。
- **残差缩放标量**：每层可学习的 `resid_lambdas`（初始 1.0）与 `x0_lambdas`（初始 0.1），每层前 `x = resid_lambdas[i]*x + x0_lambdas[i]*x0`（forward 第 277 行），把"从第 0 层残差直连"变成可学习参数。

```python
# train.py forward 主干（第 273-280 行）
x = self.transformer.wte(idx)
x = norm(x)
x0 = x
for i, block in enumerate(self.transformer.h):
    x = self.resid_lambdas[i] * x + self.x0_lambdas[i] * x0
    ve = self.value_embeds[str(i)](idx) if str(i) in self.value_embeds else None
    x = block(x, ve, cos_sin, self.window_sizes[i])
x = norm(x)
```

### 5.3 MuonAdamW 融合优化器（train.py 第 293-426 行）

单文件实现 Muon + AdamW 混合优化器：2D 矩阵参数走 Muon 路径——Nesterov 动量后做 **Polar Express 多项式正交化**（硬编码 5 组系数代替 Newton-Schulz 迭代，第 297-336 行），再叠加 **NorMuon 二阶动量方差缩减**与 **cautious weight decay**（仅更新 `g*p>=0` 的位置，第 350-353 行）；同形状参数堆叠成 3D 张量一次性更新（`torch.stack`）。embedding/unembedding/标量走 `torch.compile(fullgraph=True)` 融合的 AdamW（第 305-314 行）。学习率按参数组划分（EMBEDDING_LR=0.6、UNEMBEDDING_LR=0.004、MATRIX_LR=0.04、SCALAR_LR=0.5），并按 `1/sqrt(dmodel/768)` 整体缩放（第 248-249 行）。所有调度基于**训练时间进度**而非步数：warmdown 占后半程 50%、Muon 动量 300 步内从 0.85 升至 0.95、权重随进度线性衰减（第 518-532 行）。

### 5.4 工程细节与实验循环协议

train.py 的训练循环有几处讲究的细节：前 10 步不计入时间预算（排除 torch.compile 编译时间，第 603 行）；loss 为 NaN 或 >100 时立即 `exit(1)` 快速失败（第 570-572 行）；第 0 步后 `gc.freeze(); gc.disable()` 消除 Python GC 约 500ms 的停顿（第 593-596 行注释）；meta device 初始化再 `to_empty` 落显存（第 482-485 行）。结束时打印 9 行 summary（val_bpb/training_seconds/peak_vram_mb/mfu_percent 等），与 program.md 中 agent 的 `grep "^val_bpb:" run.log` 提取协议精确对应。program.md 则定义了完整循环：建分支 `autoresearch/<tag>` → 首跑建 baseline → 每轮 commit 后跑 5 分钟 → 改进则 advance、否则 reset → 追加 results.tsv（不 commit）→ 超过 10 分钟视为失败 → "NEVER STOP"。

## 6. 构建与运行

来自 README.md "Quick start"（要求单张 NVIDIA GPU，H100 测试通过）：

```bash
# 1. 安装 uv
curl -LsSf https://astral.sh/uv/install.sh | sh
# 2. 安装依赖（torch 2.9.1 cu128）
uv sync
# 3. 一次性数据准备：下载数据分片 + 训练分词器（约 2 分钟）
uv run prepare.py            # 可加 --num-shards 8 仅下载测试
# 4. 手动跑一次 5 分钟训练实验
uv run train.py
```

进入自主实验模式：在仓库根目录启动任意 coding agent，提示词如 "have a look at program.md and let's kick off a new experiment!"。agent 侧的标准命令（program.md）：`git checkout -b autoresearch/<tag>`、`uv run train.py > run.log 2>&1`、`grep "^val_bpb:\|^peak_vram_mb:" run.log`。数据与分词器缓存在 `~/.cache/autoresearch/`（prepare.py 第 38 行）。

## 7. 测试与工程质量

- **测试**：仓库无任何测试目录、测试框架或测试文件（tarball 顶层与 pyproject.toml 均无 pytest 配置）。唯一的自检是 prepare.py 第 199-203 行的分词器 roundtrip 断言（"Hello world! Numbers: 123. Unicode: 你好" 编解码一致）。
- **CI**：tarball 中无 `.github/` 目录，无 CI 工作流（未确认 GitHub 侧是否曾在其他分支配置过）。
- **代码质量观察**：两份源码都遵循 Karpathy 一贯的"教学式单文件"风格——常量集中在文件顶部、分节注释横幅、几乎无抽象层。风险控制更多交给协议而非代码：program.md 的 crash 处理规则、10 分钟超时 kill、results.tsv 的 0.000000/0.0 崩溃记账约定。对普通软件工程标准而言这不算健壮，但对"agent 沙盒实验"场景是合理的最小化设计。

## 8. 社区与维护状态

- **生命周期极短而爆发力极强**：创建于 2026-03-06，最后一次 push 2026-03-26（GitHub API commits，228791f "Merge pull request #342"），三周内完成全部开发后停更；快照星数 98461（2026-10-24），API 实测 98881（2026-10-30）。
- **贡献者结构**：默认分支 contributors API 返回 9 人；最近 5 条提交中除作者外可见外部 PR（kaizen-38 的 bug fix、indianspeedster 的 fork 列表更新），属于"作者主导 + 小量外部 PR"模式。
- **Issue 积压**：195 个 open issues 无人处理（GitHub API），与停更状态一致；13634 个 fork。
- **生态分叉**：README "Notable forks" 列出 4 个平台移植——miolini/autoresearch-macos、trevin-creator/autoresearch-mlx（均 macOS）、jsegov/autoresearch-win-rtx（Windows）、andyluo7/autoresearch（AMD），并附小算力（MacBook 级）调参指南（README "Platform support" 一节，含 TinyStories 数据集、降 vocab、降 MAX_SEQ_LEN 等 7 条建议）。
- **Release**：无 release、无 tag（GitHub API 返回 0），无版本管理，属于一次性发布型项目。

## 9. 亮点与风险

**亮点**

1. **范式创新**："人类写 program.md（研究组织代码）、agent 写 train.py（模型代码）"的角色倒转，是当前最清晰的"AI 自主科研"最小可行演示（README.md、program.md）。
2. **实验方法论严谨**：固定 5 分钟时间预算 + 词表无关的 val_bpb，使任意改动都可比、可积累；git 分支 advance/reset 的 keep-or-discard 协议天然可审计（program.md）。
3. **训练栈本身先进**：Muon（Polar Express + NorMuon + cautious WD）、value embeddings 门控、QK-norm、ReluSquared、SSSL 滑窗 FA3、时间驱动 LR 调度——train.py 单文件即是当代单 GPU 预训练的参考实现。
4. **工程细节成熟**：best-fit packing 100% 利用率 dataloader、编译时间与训练时间分离记账、GC 停顿治理、NaN/爆炸快速失败（train.py）。
5. **教学与传播价值**：三个文件即可读懂全貌，配套 notebook 分析、小算力调参指南与 fork 生态，上手门槛极低。

**风险/局限**

1. **平台强绑定**：仅支持单张 NVIDIA GPU，MFU 计算硬编码 H100 峰值 989.5e12 FLOPs（train.py 第 463 行）；CPU/MPS/多卡明确不支持，跨平台依赖社区 fork（README.md）。
2. **许可证瑕疵**：README 声明 MIT 但仓库无 LICENSE 文件（GitHub API license=null），商用前需注意。
3. **无测试、无 CI、已停更**：三周 burst 后冻结，195 个 open issues 无响应；升级 torch/依赖可能破坏 FA3 内核拉取路径。
4. **结果不可跨平台比较（by design）**：固定墙钟时间意味着不同硬件得到不同的最优模型；依赖外部 HF 数据集（climbmix-400b-shuffle）与 `~/.cache` 的可用性。

## 10. 总结评价

autoresearch 用三个文件完成了一次概念验证：把研究流程本身写成一份 Markdown 剧本，让 agent 在固定规则（5 分钟预算、val_bpb 裁判、keep-or-discard 分支协议）下整夜自主进化一个真实的 GPT 训练代码库。它既不是库也不是框架，而是一份可运行、可 fork、可迭代的研究组织蓝图——训练栈部分（MuonAdamW、value embeddings、best-fit packing 等）有真实的参考实现价值，而更大的价值在于它把"AI 做研究"从愿景压缩成了一个周末就能复现的实验。适合三类人：想亲眼验证 agent 自主科研闭环的实践者（一张 GPU 即可）、想读一份现代单 GPU 预训练范本的研究者（train.py），以及研究"如何给 agent 写指令"的 prompt/组织设计者（program.md 是比任何教程都诚实的样本）。不适合需要生产级训练框架、多卡分布式或长期维护支持的用户——那是它的父项目 nanochat 的领地。
