# deepseek-ai/DeepSeek-R1 项目分析

> 星数 91942（快照 2026-10-24）｜主语言 —（无代码）｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：DeepSeek 第一代推理模型（DeepSeek-R1 / R1-Zero 及 6 个蒸馏版）的官方发布仓库——一个"模型卡"型仓库，不含训练/推理代码，权重全部经 Hugging Face 分发。

| 项目 | 内容 |
|---|---|
| 名称 | deepseek-ai/DeepSeek-R1 |
| 星数 | 91942（快照 2026-10-24；gh api 实测 91987） |
| Fork 数 | 11661（gh api） |
| 语言 | 无代码（仅 README/LICENSE/PDF/图片/1 个 workflow，共 5 个文件） |
| 许可证 | MIT（LICENSE，代码与模型权重均 MIT；部分蒸馏版受上游约束） |
| 创建时间 | 2025-01-20（gh api） |
| 最近活跃 | main 分支最后提交 2025-04-09；release v1.0.0 发布于 2025-06-27（gh api） |
| 维护状态 | 发布后进入冻结状态，基本不再更新（见第 8 章） |

## 2. 项目是什么，解决什么问题

DeepSeek-R1 仓库是 DeepSeek-AI 于 2025 年 1 月发布的推理模型（reasoning model）官方落地页。它解决的问题是：在 OpenAI o1 独占"长思考"推理能力、且不开放权重与蒸馏权限的背景下，提供第一个在数学/代码/通用推理上与 o1 相当、又完全开放权重的开源替代（README.md 第 1 节："DeepSeek-R1 achieves performance comparable to OpenAI-o1 across math, code, and reasoning tasks"）。

仓库核心交付物是一套 8 个模型权重，全部托管在 Hugging Face（README.md 第 3 节两张下载表）：

- **DeepSeek-R1-Zero（671B 总参/37B 激活，MoE，128K 上下文）**：直接在 DeepSeek-V3-Base 上做大规模强化学习，不经任何监督微调（SFT）。README 明确称其为"首个开源研究验证 LLM 推理能力可纯靠 RL 激发"，模型自发涌现自我验证、反思、长思维链等行为（README.md 第 2 节 Model Summary）。代价是可读性差、 endless repetition、语言混杂。
- **DeepSeek-R1（671B/37B，128K）**：在 Zero 流程前加入冷启动数据，采用"两阶段 RL + 两阶段 SFT"流水线，兼顾推理能力与人类偏好对齐（README.md 第 2 节）。
- **6 个蒸馏小模型**：用 DeepSeek-R1 生成的 80 万条样本微调 Qwen2.5 与 Llama 系列得到，覆盖 1.5B/7B/8B/14B/32B/70B 六个规模（README.md 第 3 节 Distill 表）。其中 R1-Distill-Qwen-32B 在多个基准上超过 o1-mini。

仓库同时是一份"使用说明书"：README 第 6 节给出 temperature 0.5-0.7（推荐 0.6）、禁止 system prompt、数学题要求 `\boxed{}` 收尾、强制以 `<think>\n` 开头等复现性关键设置，并公开了官网在用的文件上传与联网搜索两套官方 prompt 模板（中英双语全文）。这些细节使第三方能以正确姿势复现评测结果，是该仓库区别于普通公告页的实质内容。

配套的 `DeepSeek_R1.pdf`（1.3MB）即技术报告 "DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning"（arXiv 2501.12948，README.md 第 8 节 Citation；PDF 本身无文本层可校验，以 README 引用为准）。

需要强调：GitHub 上没有权重、没有训练代码、没有推理代码。训练细节（GRPO、RL 奖励设计等）在论文里，模型架构沿用 DeepSeek-V3（README 第 3 节明确指向 DeepSeek-V3 仓库），推理框架支持由 vLLM/SGLang 等第三方项目承接。

## 3. 技术栈与依赖

本仓库自身零依赖、零构建：无 package.json / pyproject.toml / requirements 等任何工程清单（顶层仅 5 个文件，`tar -tzf` 全量确认）。围绕它的真实"技术栈"是模型生态：

| 组件 | 用途 | 出处 |
|---|---|---|
| Hugging Face Hub | 8 个模型权重的唯一分发渠道 | README.md 第 3 节 |
| DeepSeek-V3 架构 | R1/R1-Zero 的基座（MoE 671B/37B），本地运行文档指向 V3 仓库 | README.md 第 3、6 节 |
| vLLM | 蒸馏模型推荐 serving 方式，给出完整启动命令 | README.md 第 6 节 |
| SGLang | 备选 serving，`--trust-remote-code --tp 2` | README.md 第 6 节 |
| Qwen2.5 / Llama 3.x | 蒸馏小模型的底座 | README.md 第 3、7 节 |
| GitHub Actions + actions/stale@v9 | 唯一的自动化：每日清理 stale issue | .github/workflows/stale.yml |

## 4. 目录结构与代码组织

顶层结构（HEAD 快照，共 5 个文件）：

| 路径 | 职责 |
|---|---|
| `README.md`（278 行） | 唯一核心文件：模型介绍、下载表、评测表、运行方式、使用建议、官方 prompt、许可证与引用 |
| `LICENSE` | MIT 全文（Copyright (c) 2023 DeepSeek） |
| `DeepSeek_R1.pdf`（1.3MB） | 技术报告/arXiv 论文 2501.12948 |
| `figures/benchmark.jpg`（777KB） | R1 与 o1 等基线的基准对比图（README 内嵌） |
| `.github/workflows/stale.yml` | 每日 stale issue 自动标记与关闭 |

组织评述：这是典型的"发布型仓库"（announcement repo）形态——单页 README 承担官网落地页、模型卡、评测报告和使用文档四重角色，没有目录层级可言。与之配套的工程在兄弟仓库：训练/推理在 DeepSeek-V3（含 inference 代码），日常交互在 chat.deepseek.com 与 platform.deepseek.com API（README 第 5 节）。没有 CONTRIBUTING、CHANGELOG、issue 模板和 CI 测试（`tar -tzf` 全量清单确认）。

## 5. 核心架构与关键实现

无代码，改为拆解 README 中最具信息量的三块内容。

**1) 训练流水线设计（README.md 第 2 节）**：R1-Zero 证明"纯 RL、无 SFT 冷启动"即可激发推理；R1 则是工程化版本——两个 RL 阶段（发现更优推理模式 + 人类偏好对齐）与两个 SFT 阶段（推理/非推理能力种子）交替。这一流水线描述是全仓库最有复用价值的部分，后来被大量开源 RL 复现项目引用。

**2) 蒸馏版本表与权重分发（README.md 第 3 节）**：蒸馏版本表给出每个小模型与底座的精确对应（如 Distill-Qwen-7B ← Qwen2.5-Math-7B，Distill-Llama-70B ← Llama-3.3-70B-Instruct），并提醒"我们微调时改了 config 和 tokenizer，请用我们的设置运行"（README 第 3 节原文），提示直接用底座配置会出错。

**3) 使用建议与官方 prompt（README.md 第 6 节）**：四条硬性建议（temperature 0.5-0.7、无 system prompt、数学题 boxed 指令、多轮平均评测）+ 强制 `<think>\n` 开头防跳过思考；随后完整贴出官网在用的 `file_template` 与中英文 `search_answer_zh/en_template`。这段约 60 行，是第三方复现 DeepSeek 官方产品体验的直接素材。

评测口径也写得很严格（README 第 4 节）：最大生成长度 32768、temperature 0.6、top-p 0.95、每题 64 条采样估 pass@1——直接框定了社区后续评测该模型时应采用的协议。

## 6. 构建与运行

仓库本身无需构建。运行模型（README.md 第 6 节原文命令）：

```shell
# 671B 主模型：Transformers 尚不直接支持，参考 DeepSeek-V3 仓库
# （README 原文："Hugging Face's Transformers has not been directly supported yet."）

# 蒸馏模型（以 32B 为例），vLLM：
vllm serve deepseek-ai/DeepSeek-R1-Distill-Qwen-32B --tensor-parallel-size 2 --max-model-len 32768 --enforce-eager

# SGLang：
python3 -m sglang.launch_server --model deepseek-ai/DeepSeek-R1-Distill-Qwen-32B --trust-remote-code --tp 2
```

评测/使用配置要点（README 第 4、6 节）：temperature=0.6、top-p=0.95、最大生成 32768 token、不设 system prompt、强制 `<think>\n` 起始。

## 7. 评测基准与仓库工程化（内容型仓库特例：替代"测试与工程质量"）

- **评测透明度**：README 第 4 节两张完整表格（R1 主表 16 项基准 × 5 个基线；蒸馏表 6 基准 × 6 模型），口径、采样参数全部公开。关键数字：AIME 2024 pass@1 79.8（o1-1217 为 79.2）、MATH-500 97.3、Codeforces rating 2029（o1 2061）、LiveCodeBench 65.9（README 第 4 节）。蒸馏表中 Distill-Qwen-32B AIME 72.6 全面超过 o1-mini 的 63.6。
- **可复核性**：`figures/benchmark.jpg` 提供可视化对比，论文 PDF 提供完整实验细节，但仓库内没有评测脚本，数字无法在仓库内独立复跑（需依赖论文与第三方框架）。
- **工程化程度**：仅一条 CI——stale.yml 每日运行 actions/stale@v9，issue 30 天无活动标记 stale、再 14 天关闭，exempt `pinned`/`security` 标签，PR 明确豁免（`days-before-pr-stale: -1`）。配合 45 个 open issue（gh api），说明 issue 区主要作为用户反馈渠道而非开发协作面。
- **文档质量**：单文件 278 行，结构完整（9 节），英文撰写，prompt 模板中英双语，数字与表格排版规范；仅依赖 2 个外部图片链接（借用 DeepSeek-V2 仓库的 logo/badge，README 第 6-28 行）。

## 8. 社区与维护状态

- **活跃度**：main 分支最后提交 2025-04-09（三个社区 PR 合并：#129、#189、#100，gh api commits）；仓库最后一次动静是 2025-06-27 打的 release `v1.0.0`（gh api releases，与 pushed_at 一致）。此后约 18 个月（至分析日）无任何提交——仓库已冻结，作为历史发布页存在。
- **贡献者**：12 人（gh api contributors），主体为 DeepSeek 团队账号，社区贡献以 README 修订类小 PR 为主（如 #189 来自 eladb，均为 patch 分支命名，未确认具体内容规模）。
- **release 节奏**：仅 1 个 release（v1.0.0，2025-06-27），无版本迭代概念——权重更新发生在 Hugging Face 侧而非 GitHub。
- **社区规模**：91.9k 星 / 11.7k fork / 598 watchers / 45 open issues（gh api）。stale bot 自动关闭低活跃 issue，issue 区维护成本低。
- **生态位**：该仓库是 2025 年 1 月 "R1 时刻"（开源模型首次对齐 o1、引发全球关注，此为公开背景，未在仓库内验证）的源点之一；星数曲线与事件强相关，创建一个月内即成为增长最快的 AI 仓库之一（未确认，仓库内无历史数据）。

## 9. 亮点与风险

**亮点**

1. 历史意义明确：首个开源的 o1 级推理模型发布页，R1-Zero 证明纯 RL 可激发推理能力，直接带动了整个开源 RL 推理复现浪潮（README 第 2 节）。
2. 许可证极其宽松：代码与权重均 MIT，明确允许蒸馏训练其他 LLM（README 第 7 节），是当时头部模型中最开放的授权姿态。
3. 蒸馏谱系完整：1.5B→70B 六档小模型覆盖消费级到单机多卡部署，且逐个标注底座与许可证约束（Qwen 系 Apache 2.0、Llama 系沿用 Meta 许可证，README 第 7 节）。
4. 使用协议透明：temperature/采样/评测口径/官方 prompt 全部公开，把"复现性"写进了 README（第 4、6 节）。

**风险与局限**

1. 仓库已停止维护：最后提交 2025-04-09、最后 release 2025-06-27，README 中 "Transformers has not been directly supported yet" 等表述早已过时（transformers 现已支持相关架构），仓库不再修订（gh api / README 第 6 节）。
2. 仓库本身无任何代码：想要训练或推理实现，必须转到 DeepSeek-V3 仓库与论文，本仓库无法独立使用（顶层文件清单）。
3. 671B 主模型部署门槛极高（需 V3 级推理栈），多数用户实际只能用蒸馏模型（README 第 3 节参数表）。
4. 评测数字为官方自报，仓库内无复现脚本，第三方复测依赖论文外的框架与配置（第 4 节与文件清单推论）。

## 10. 总结评价

DeepSeek-R1 是一个"以 README 为全部"的模型发布仓库，但它可能是 2025 年对开源 AI 社区影响最大的单个仓库之一：MIT 授权 + 权重全开 + 蒸馏自由，把 o1 级推理能力从闭源 API 变成了任何人可下载、可微调、可商用的资产。仓库文档本身质量很高——训练流水线、蒸馏谱系、评测口径、官方 prompt 一应俱全——只是它注定只是一张"发布页"：2025 年 6 月后冻结，README 中的部分限制说明已过时。适合三类人使用：需要下载权重的模型用户（直接看第 3、6 节表格和命令）、研究 RL 推理流水线的研究者（看第 2 节与配套论文 PDF）、以及想理解"模型卡/发布型仓库应该怎么写"的工程团队（全文 278 行就是范本）。想读代码的人应该去 DeepSeek-V3 仓库，而不是这里。
