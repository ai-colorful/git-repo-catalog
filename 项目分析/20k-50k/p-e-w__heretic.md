# p-e-w/heretic 项目分析

> 星数 40764（快照 2026-11-17，gh api 实测）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-17

## 1. 项目速览

**一句话定位**：Heretic 是一个"无需训练"（training-free）的语言模型行为修改工具——用 Optuna 贝叶斯优化 + LoRA 权重修改，在消费级 GPU 上几十分钟内去除模型审查、抑制 AI 腔（slop）、降低谄媚度等，产出可直接发布到 Hugging Face 的修改后模型。

| 项 | 值 | 出处 |
|---|---|---|
| 名称/作者 | heretic-llm（PyPI 包名），作者 Philipp Emanuel Weidmann（p-e-w） | `pyproject.toml`、README.md |
| 星数/分叉 | 40,764 / 4,688 | gh api 2026-11-17 |
| 语言 | Python 3.10+（约 8300 行核心+测试代码） | `pyproject.toml`、`wc -l` 统计 |
| 许可证 | AGPL-3.0-or-later | `LICENSE`、`pyproject.toml` |
| 创建/最近推送 | 2025-09-21 / 2026-11-16（分析前一天仍在推送） | gh api |
| 最近 release | v2.1.0（2026-11-08）；本地版本号 2.2.0.dev0 | gh api、`pyproject.toml` |
| 主页 | https://heretic-project.org | `pyproject.toml` |

## 2. 项目是什么，解决什么问题

Heretic 解决的问题是：大语言模型的系统级行为（拒绝回答、AI 套话、谄媚、冗长等）通常只能在训练阶段改变，普通用户没有算力做微调。Heretic 用一组数学技术——对比示例 prompt + 打分器 + 贝叶斯超参优化 + LoRA 权重手术——绕过训练环节，直接"外科手术式"地改掉这些行为。README 明确宣称"不需要理解模型内部原理"，一条命令 `heretic Qwen/Qwen3-4B-Instruct-2507` 即可在 RTX 3090 上约 30-40 分钟完成去审查（README.md "Usage" 节）。

与传统的"abliteration"（方向消融去审查）手工脚本相比，Heretic 的差异化在于**把行为修改变成一个自动化的多目标优化问题**：给模型装上 LoRA 适配器，让 Optuna 的 TPE 采样器反复尝试不同修改参数，每个 trial 用打分器插件给模型打分（如拒绝率、KL 散度），最后从 Pareto 前沿中挑出"行为改变最大、能力损失最小"的方案（`src/heretic/main.py` 的 `objective()` 函数，默认 100 trials）。社区已用它在 HF 上发布了数千个带 `heretic` 标签的模型（README.md）。

默认功能是去审查（refusal suppression），同时官方提供 8 个配置预设：去 slop、去幽默、去谄媚、情绪对齐调整、缩短回答、PIQA 能力保持、BERT 分类器打分等（仓库根 `config.noslop.toml`、`config.nosyco.toml`、`config.emotions.toml`、`config.terse.toml` 等）。以 `config.noslop.toml` 为例，其"抑制 AI 套话"的实现方式是把 "Eldoria、ethereal、velvet、moonlit" 等典型 AI 写作特征词列入 `KeywordRate` 打分器的 `keyword_markers`，让优化器最小化出现率——即所有功能共用同一套优化框架，只是换了打分目标。

需要注意概念澄清：Heretic **不是**推理时激活干预（activation steering / repE）工具，它最终**生成修改后的权重文件**（LoRA 合并进底模导出 safetensors）；前向 hook 只在分析阶段用于捕捉各层模块的输入输出张量（`src/heretic/model.py` 的 `get_module_io()`）。修改结果是持久化的模型，而非运行时拦截。

## 3. 技术栈与依赖

全部来自 `pyproject.toml`（Python 3.10+，uv 构建，依赖锁定在 `uv.lock`）：

- **深度学习**：`torch`（版本刻意不锁定）、`transformers~=5.18`、`peft~=0.21`（LoRA）、`accelerate~=1.15`（device_map）、`bitsandbytes~=0.50`（bnb_4bit 在线量化支持）
- **优化**：`optuna~=5.0`（TPESampler 多变量多目标采样 + JournalStorage 断点续跑）
- **评估**：`lm-eval[hf]~=0.4`（内建 benchmark 跑 MMLU/PIQA 等）、`datasets~=5.0`（加载 mlabonne/harmless_alpaca、harmful_behaviors 等评估集）
- **CLI/交互**：`rich`、`questionary`（交互菜单）、`IPython`（可选 in-context shell）、`pydantic-settings`（TOML 配置校验）
- **开发工具**：`ruff`（格式+lint）、`ty`（类型检查，CI 中 error-on-warning）

硬件要求：能跑目标模型的 GPU；可用 `quantization = "bnb_4bit"` 把加载显存压到典型量化模型水平（README.md 脚注 1）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `src/heretic/main.py`（1568 行） | CLI 入口 `heretic` 命令：批量大小自动测试、CoT 前缀探测、Optuna trial 循环、导出/上传/聊天/benchmark 交互菜单 |
| `src/heretic/model.py`（1013 行） | `Model` 封装：加载（含 bnb 量化）、`apply_lora`、`get_residuals/get_module_io`（hook 捕获激活）、`get_logits`、合并导出；兼容 dense/MoE/多模态/多 residual stream 架构 |
| `src/heretic/config.py`（646 行） | pydantic-settings 配置体系，全部 CLI/TOML 选项及文档 |
| `src/heretic/modifier.py` + `modifiers/` | Modifier 插件接口 + 两个实现：`ara.py`（默认）、`abliteration.py` |
| `src/heretic/scorer.py` + `scorers/` | Scorer 插件接口 + 5 个实现：`keyword_rate.py`、`kl_divergence.py`、`bert_classifier.py`、`benchmark_score.py`、`response_length.py` |
| `src/heretic/plugin.py`（405 行） | 插件加载机制：支持 import 路径或 `path/to/plugin.py:ClassName` 文件路径，`Context` 提供响应/logits/残差缓存 |
| `src/heretic/evaluator.py`（239 行） | 汇集各 scorer 得分，映射为 Optuna 多目标方向 |
| `src/heretic/reproduce.py`（385 行） | reproduce.json 清单生成/校验/环境比对/`--collect-reproducibles` 收集 |
| `config.*.toml`（8 个） | 预设：default（去审查）、noslop、nohumor、nosyco、emotions、terse、piqa、classifier |
| `tests/` | 6 个单元测试 + 11 个真实模型复现测试目录（SHA256SUMS 校验）+ `run_tests.py` |

组织方式评述：单包单体（src layout），代码极精简（全项目 Python 约 8300 行），核心抽象只有两个插件接口（Modifier/Scorer），扩展新"修改目标"只需写一个 Scorer。README 强调代码由作者手写而非 vibe-coded。

## 5. 核心架构与关键实现

### 5.1 ARA（Arbitrary-Rank Ablation，默认修改器）— `src/heretic/modifiers/ara.py`

这是为 Heretic 专门开发的方法（README 称 "Weidmann 2026"）。流程：先用 forward hook 记录好/坏两组 prompt（默认各 400 条，来自 `mlabonne/harmless_alpaca` 与 `mlabonne/harmful_behaviors`）在各层各模块上的输入输出，然后给指定层区间装 rank-50 LoRA，用 L-BFGS 直接对 LoRA 的 A/B 权重做梯度优化，损失函数（`ara_loss`，`ara.py` L125-160）：

```python
# 好 prompt 的输出应尽量不变
preserve_good_behavior = ((new_good_output - good_output) ** 2).mean()
steer_bad_behavior = (
    # 把坏 prompt 的输出拉近好 prompt 的输出（kNN 距离）
    mean_distances_to_knn(new_bad_output, good_output, parameters.neighbor_count).mean()
    # 同时推离原坏输出——过度校正以突破更复杂的拒绝机制
    + parameters.overcorrect_relative_weight
    * -mean_distances_to_knn(new_bad_output, bad_output, parameters.neighbor_count).mean()
)
```

优化的是 LoRA 低秩增量 `W_base + B@A`（`ara.py` L324-341 的注释解释了只重算低秩项以避免物化稠密矩阵的显存技巧），可选保留行范数（MPOA 思想）。层区间 `start_layer_index/end_layer_index`、损失权重、邻居数 6 个超参全部交给 Optuna 搜索（`suggest_parameters`，L206-252）。

### 5.2 方向消融（abliteration）— `src/heretic/modifiers/abliteration.py`

经典 Arditi et al. 2024 方法的工程化增强版：逐层计算好/坏 prompt 残差流均值之差并归一化得到"拒绝方向"，然后以闭式解写入 LoRA：`delta W = -lambda * v * (v^T W)`（L371-416）。增强点：投影消融（把方向中与好方向共线的分量正交化掉，L193-214）、行范数保持（FULL 模式先归一化再加 delta 后重归一化，用 `torch.svd_lowrank` 截断到 rank-3 近似，L399-452）、每组件可学习的"消融强度钟形分布"（max_weight 及其在层栈上的位置/宽度/底值，MLP 可整段关闭——注释 L266-270 引 issue #202 说明消融 MLP 往往弊大于利）。

### 5.3 优化闭环与插件体系 — `main.py` + `plugin.py` + `evaluator.py`

`run()` 的主循环：每个 Optuna trial 中 `modifier.reset_model()`（LoRA 清零）→ `modify_model(parameters)` → `evaluator.get_scores()`，得分回填 TPE 采样器；结束时按 Pareto 前沿交互式选择 trial，再提供保存/上传 HF/聊天/lm-eval benchmark 四个动作。Scorer 与 Modifier 都是插件（`plugin.py` 的 `load_plugin` 支持任意 `.py` 文件路径），用户不改包代码即可定义新行为目标——这是"改 slop/幽默/谄媚"功能分层的基础。

### 5.4 复现性系统（Grimoire）— `reproduce.py` + `main.py`

上传 HF 时可附带约 9KB 的 `reproduce.json`（v4 格式：完整 settings、trial 参数、包版本、系统信息、权重 SHA256）；他人运行 `heretic --reproduce xxx.json` 时无需重做多小时的优化计算，几分钟即可重建模型并校验权重哈希（`main.py` L1093-1116 哈希核对逻辑）。README 称此构成一个"分布式模型库"（Heretic Grimoire），`tests/deepseek-v4/SHA256SUMS.*` 等文件表明 CI 中就用同一机制做逐字节复现测试。

## 6. 构建与运行

来自 README.md 与 `pyproject.toml`：

```sh
# 安装（需 Python 3.10+ 与配套硬件的 PyTorch 2.6+）
pip install -U heretic-llm
# 默认即去审查
heretic Qwen/Qwen3-4B-Instruct-2507
# 换功能：把对应 config 复制为工作目录 config.toml（或设 HERETIC_CONFIG_FILE）
# 复现他人模型 / 收集 Grimoire 清单
heretic --reproduce path/to/reproduce.json
heretic --collect-reproducibles path/to/target/folder/
```

开发环境（CI 同款，`.github/workflows/ci.yml`）：`uv sync --all-extras --dev`；检查：`uv run ruff format --check .`、`uv run ruff check --extend-select I .`、`uv run ty check --error-on-warning .`；单元测试：`uv run python -m unittest discover -s tests -p 'test_*.py'`；构建：`uv build`。

## 7. 测试与工程质量

- **单元测试**：6 个 `tests/test_*.py`（unittest 风格，共约 790 行），覆盖 ARA 低秩等价性、chat 模板 CoT 处理、配置校验、KeywordRate 归一化（弯引号/星号强调/空响应判拒绝防"刷空串"）、插件加载、多 residual stream 折叠。
- **复现测试（亮点）**：`tests/run_tests.py` 遍历 11 个真实模型目录（deepseek-v4、gemma-3、gemma-4e、gpt-oss、minicpm5、mistral-3、qwen2.5、qwen3.5-moe、seed-oss），跑完完整流程后把导出权重 SHA256 与 `SHA256SUMS.linux/.windows/.ci*` 比对——等价于端到端回归测试，且 CI（`ci.yml`）真的在跑它。这解释了 abliteration.py L436-439 里在 `svd_lowrank` 前显式重播种的原因（随机 SVD 会破坏逐字节复现）。
- **CI**：GitHub Actions，Python 3.10-3.15 六版本矩阵，ruff + ty（error-on-warning）+ unittest + 复现测试 + uv build 构建校验（`.github/workflows/ci.yml`）。
- 代码质量观察：注释密度高且多为"为什么"注释（显存技巧、采样器限制、issue 引用），存在少量 `cast`/`FIXME`（如 `abliteration.py` L362-368 承认 PEFT 量化路径下模块类型假定未验证）。

## 8. 社区与维护状态

- **活跃度**：创建于 2025-09-21，14 个月冲到 40.8k 星（trendshift #1 Repository of the Day 徽章，README.md L23）；最近推送 2026-11-16（分析前一天），open issues 65（gh api）。
- **Release 节奏**：v1.1.0（2025-12）→ v1.2.0（2026-02）→ v1.3.0（2026-05）→ v1.4.0（2026-06）→ v2.0.0（2026-11-01，插件系统重构，附 `docs/migration_v1_to_v2.md` 迁移指南）→ v2.1.0（2026-11-08），大约每 1-2 月一个版本。
- **贡献者结构**：41 名贡献者，但高度单维护者——p-e-w 128 次提交居首（约为第二名的 5 倍多），其后为 dependabot、Vinay-Umrethe（BERT 打分器作者，PR #500）等；README 提到 PR #53（约 350 条评论/7 个月）与 #191（200+ 评论）显示审稿极严。
- **生态**：HF 上数千个 `heretic` 标签模型；第三方独立评测（DreamFast 的三向/23 变体 abliteration 对比）结论"heretic 最精准、综合最优"（README.md 脚注 2、3）；Discord/Matrix 社区。
- **作者背景**（gh api 核实）：Philipp Emanuel Weidmann，老牌开源开发者，代表作为 maybe（6.3k 星，"假设式"命令预览）、finalterm（3.8k 星终端）、argos、ternimal，个人主页 worldwidemann.com。

## 9. 亮点与风险

**亮点**

1. 技术路线独特：把模型行为修改工程化为"插件打分 + Optuna 多目标优化 + LoRA 手术"，一次框架投入换来无限行为目标（换 Scorer 即换功能），远超一次性 abliteration 脚本。
2. 复现性做到工业级：reproduce.json + 哈希校验 + 随机 SVD 重播种 + CI 内 11 个真实模型的逐字节复现回归测试，在个人项目里罕见。
3. 可用性打磨到位：自动 batch size 基准、CoT 块（`<think>`/gpt-oss channel）自动探测、bnb_4bit 显存压缩、中断续跑（JournalStorage）、出错不丢优化结果的 action 循环。
4. 代码与工程文化：约 8300 行的精简单体，高密度"为什么"注释，严格人工 review 的贡献流程。

**风险/局限**

1. **滥用与合规风险（本项目的核心争议）**：项目主功能即绕过模型的安全对齐（去审查），HF 上已存在大量现成"已去审查"模型；对上游模型厂商的许可约束、各地内容法规的责任完全转嫁给使用者。README/LICENSE 无任何使用场景限制条款。
2. **打分器可被规避且效果依赖关键词表**：默认拒绝检测是关键词匹配（`keyword_rate.py` 的 REFUSAL_MARKERS），模型换措辞即可逃逸；去 slop/谄媚等目标本质是拟合特定词表，泛化性有限（BERT 分类器打分器是改进方向但为可选项）。
3. **单维护者 + AGPL-3.0**：核心决策集中于一人，Bus factor 低；AGPL 对想集成进闭源产品的团队是硬约束（CLI 工具场景影响较小）。
4. **能力损失与算力门槛**：行为修改必然伴随能力回退风险（KL 散度目标只压制而非消除；README 也承认 bnb_4bit 有精度代价）；完整流程需本机拥有能跑目标模型的 GPU，100 trials 在大模型上耗时可观。

## 10. 总结评价

Heretic 是"模型行为修改"方向目前工程质量最高的开源实现：它没有停留在 abliteration 论文的演示脚本层面，而是围绕同一套数学内核（方向消融 + 自研 ARA）建出了插件化打分、多目标贝叶斯优化、断点续跑、逐字节复现、一键发布 HF 的完整产品闭环，加上作者 p-e-w 数十年开源工程功底（maybe/finalterm 作者）和近乎手写代码的严格主义，使其在 14 个月内成为 40k 星的现象级工具。想研究表示工程/模型对齐机制的人，`modifiers/ara.py` 与 `abliteration.py` 是极好的精读材料（hook 捕获激活 → L-BFGS/闭式解写 LoRA 的完整链路）；想自己改模型行为的玩家，只需 GPU + 一条命令；而将其引入产品链路的团队则需同时评估 AGPL、滥用责任与关键词打分的可靠性边界。
