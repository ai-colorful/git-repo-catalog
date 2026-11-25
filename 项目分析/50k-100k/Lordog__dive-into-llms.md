# Lordog/dive-into-llms 项目分析

> 星数 57308（快照 2026-10-24）｜主语言 Jupyter Notebook｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

**《动手学大模型（Dive into LLMs）》系列编程实践教程**——由上海交通大学《自然语言处理前沿技术》（NIS8021）、《人工智能安全技术》（NIS3353）课程讲义拓展而来的公益入门教程，用 11 个章节覆盖大模型从微调部署到安全对齐的完整实践链路。

| 项目 | 内容 |
| --- | --- |
| 名称 | Lordog/dive-into-llms |
| 简介 | 《动手学大模型Dive into LLMs》系列编程实践教程（README.md 自述） |
| 星数 / Fork | 57308 / 7016（gh api，2026-11-03 查询为 58860/7016，持续上涨中） |
| 主语言 | Jupyter Notebook（另含 Markdown、PDF 课件） |
| 许可证 | 无 LICENSE 文件（gh api `license: null`，仓库内亦无） |
| 创建 / 最近推送 | 2024-04-08 / 2025-10-10（gh api pushed_at） |
| 规模 | 128 个文件，84MB 课件与脚本（documents/ 实测），11 个 PDF 讲义共约 63MB |
| 贡献者 | 11 人（gh api contributors），核心为上交大 BCMI 团队师生 |

## 2. 项目是什么，解决什么问题

这是典型的"内容型仓库"：没有可安装的软件产品，产出物是 **11 章"课件 PDF + 图文教程 + 可运行 Notebook"三件套**。它解决的问题是：大模型技术门槛高、资料分散，初学者（尤其课程学生）缺一条"从环境安装到跑通完整实验"的最短路径。README 明确写道"旨在提供大模型相关的入门编程参考……帮助同学们快速入门大模型，更好地开展课程设计或学术研究"（`README.md` 项目动机一节）。

11 章目录（`README.md` 教程目录表 + `documents/chapter1~11/` 实测）：

1. **微调与部署**：Transformers 工具链微调 BERT（虚假新闻分类）+ Gradio Spaces 部署
2. **提示学习与思维链**：通义千问/智谱/OpenAI API 调用、零样本/少样本、CoT/PoT、自洽性
3. **知识编辑**：用 zjunlp/EasyEdit 的 ROME 等方法改写 GPT-2-XL 的知识记忆
4. **数学推理**：蒸馏 DeepSeek-R1 解答微调 Qwen2.5-Math-1.5B，"迷你 R1"全流程
5. **模型水印**：基于 zwhe99/X-SIR 的 KGW/SIR/X-SIR 三种水印嵌入、检测与鲁棒性评估
6. **越狱攻击**：EasyJailbreak 框架、PAIR 攻击法实操
7. **大模型隐写**：GPT-2 生成文本中用霍夫曼/定长编码隐藏信息
8. **多模态模型**：MLLM 两大架构（LLM 作调度器 vs 编码器-LLM-解码器）与搭建训练
9. **GUI 智能体**：Qwen2-VL-7B + OS-Kairos 数据集 + LLaMA-Factory 微调自适应 GUI 智能体
10. **智能体安全**：R-Judge 平台测评开放智能体场景的风险识别能力
11. **RLHF 安全对齐**：trl 库 PPO 算法微调 GPT-2 生成积极影评

选材特点是"**主干技能 + 安全专题并重**"：前 4 章是通用基本功（微调、提示、编辑、推理），后 7 章里水印、越狱、隐写、智能体安全、RLHF 对齐 5 章直接来自安全课程（NIS3353）讲义，这在同类入门教程中差异化明显——多数入门教程止步于微调与 RAG。

定位上它强调"动手"：每章先给目标，再给环境安装命令，再给可复制的参数与代码片段，最后给延伸论文链接。2025/06/06 起还与华为昇腾社区联合推出国产化《大模型开发全流程》系列（含 PPT、实验手册、视频，入口见 `README.md`）。

## 3. 技术栈与依赖

作为教程仓库，技术栈即"被教授的工具栈"（证据：各章 README 与 notebook 的 import 语句）：

- **核心框架**：HuggingFace Transformers（ch1/ch4/ch7/ch11）、Datasets、PyTorch
- **训练/推理**：LLaMA-Factory（ch9，SFT Qwen2-VL）、trl 0.9.4 PPOTrainer（ch11）、vllm（ch4，推理评测）、EasyEdit（ch3）、X-SIR（ch5）、EasyJailbreak（ch6）
- **国产模型 API**：dashscope（通义千问，ch2）、智谱 AI（ch2）
- **部署/观测**：Gradio Spaces（ch1）、WandB（ch11）、TensorBoard（ch4）
- **版本约束**：仅 ch11 提供 `documents/chapter11/requirements.txt`（torch==2.6.0、transformers==4.51.3、trl==0.9.4、datasets==3.5.0、vllm 未列版本）；其余章节依赖版本靠 README 文字说明
- **硬件门槛**：ch4 需 40GB 显存 GPU + 50GB 磁盘（`documents/chapter4/README.md`）；ch11 参考配置为单卡 A800-80GB 占用约 10GB、35 分钟（`documents/chapter11/README.md`）

依赖整体面向 2024-2025 年主流开源生态，国产模型与国内镜像（hf-mirror、清华 pypi 源）的适配是刻意设计，面向国内读者。

## 4. 目录结构与代码组织

| 路径 | 职责 |
| --- | --- |
| `README.md` | 总入口：动机、11 章目录表（每章三链接）、昇腾课程推广、贡献者名单 |
| `documents/chapter1~11/` | 每章一个目录：README.md（图文教程）+ 讲义 PDF + 实操 .ipynb + assets/ 配图 |
| `documents/chapter11/requirements.txt` | 全仓唯一依赖清单（trl PPO 实验） |
| `documents/chapter11/figs/` | PPO 流程图等实验配图 |
| `pics/icon/` | README 用的 14 张宣传图/图标 |
| `.gitignore` | 10 字节，忽略 `.DS_Store`（但 chapter9 实际仍混入一个 `.DS_Store`） |

组织方式是**扁平的章节式内容仓**：一章一目录、命名统一（`chapterN/`），无代码工程结构、无包管理、无构建系统。7 个章节的 .ipynb 实为**纯 Markdown 教程本**（0 个代码 cell，实测：ch1/2/3/5/6/8/10），内容与该章 README.md 基本同源，只是以 Notebook 形式再发一份；真正的可执行代码集中在 4 个 notebook：ch4 `sft_math.ipynb`（155 行代码）、ch7 `llm_stega.ipynb`（190 行）、ch11 `RLHF.ipynb`（196 行）、ch9 `GUIagent.ipynb`（94 行）。ch3/5/6 则是"教读者用外部工具包（EasyEdit/X-SIR/EasyJailbreak）"，代码留在上游仓库。

逐章明细（文件均实测存在于对应 `documents/chapterN/`）：

| 章 | 主题 | 课件 PDF | Notebook 形态 | 依赖的外部项目 |
| --- | --- | --- | --- | --- |
| 1 | 微调与部署 | dive-into-llm.pdf（11MB） | 纯 Markdown 教程本 | Transformers、Gradio、Kaggle 数据 |
| 2 | 提示学习与思维链 | dive-into-prompting.pdf（8.5MB） | 纯 Markdown 教程本 | dashscope/智谱/OpenAI API |
| 3 | 知识编辑 | dive_edit_0410.pdf（3.3MB） | 纯 Markdown 教程本 | zjunlp/EasyEdit（ROME/MEND） |
| 4 | 数学推理 | math.pdf（4.1MB） | **155 行可执行代码** | DeepMath-103K、Qwen2.5-Math-1.5B、vllm |
| 5 | 模型水印 | watermark.pdf（3.0MB） | 纯 Markdown 教程本 | zwhe99/X-SIR（KGW/SIR/X-SIR） |
| 6 | 越狱攻击 | dive-Jailbreak.pdf（7.9MB） | 纯 Markdown 教程本 | EasyJailbreak（PAIR 等 11 种攻击） |
| 7 | 大模型隐写 | stega.pdf（0.5MB） | **190 行可执行代码** | 自实现霍夫曼/FLC + GPT-2 |
| 8 | 多模态模型 | mllms.pdf（8.7MB） | 纯 Markdown 教程本 | NExT-GPT 等架构综述 |
| 9 | GUI 智能体 | GUIagent.pdf（6.5MB） | **94 行数据预处理代码** | Qwen2-VL-7B、OS-Kairos、LLaMA-Factory |
| 10 | 智能体安全 | dive-into-safety.pdf（7.7MB） | 纯 Markdown 教程本 | R-Judge、Auto-UI、AIOS、OS-Copilot |
| 11 | RLHF 对齐 | RLHF.pdf（2.1MB） | **196 行可执行代码** | trl（PPOTrainer）、IMDB、wandb |

## 5. 核心架构与关键实现

以 4 个自含代码 notebook 为核心逐一说明（均为静态阅读，未运行）：

**（1）数学推理蒸馏（`documents/chapter4/sft_math.ipynb`）**：数据侧从 DeepMath-103K 中 `filter(lambda x: x["final_answer"].isdigit())` 筛纯数字答案题，用 `apply_chat_template` 把 question + `<think>\n{r1_solution}\n` 组装成训练样本，再按 R1 回复长度 `<4096` 二次筛选；训练侧用 HF `Trainer`（bf16、per_device_batch 4 × 梯度累积 16、lr 1e-6、cosine 调度、3 epochs），推理评测用 vllm。这是全仓最完整的"数据蒸馏→SFT→评测"闭环。

**（2）LLM 隐写（`documents/chapter7/llm_stega.ipynb`）**：自实现 143 行霍夫曼树（`Node/createNodes/createHuffmanTree`）与 FLC 定长编码，在 GPT-2 逐 token 生成时把 secret bits 映射到候选词表，实现"流式回答里夹带隐蔽信息"；种子固定（`setup_seed(2023)`），主函数同时生成隐写/普通文本对比。全仓唯一不依赖外部工具包、纯算法实现的章节。

**（3）RLHF-PPO（`documents/chapter11/RLHF.ipynb`）**：翻译整合 newfacade 的 PPO 笔记与 trl 官方 gpt2-sentiment 示例：`PPOConfig(model_name="model/gpt2-imdb", learning_rate=1.41e-5, log_with="wandb")`，`AutoModelForCausalLMWithValueHead` 同时加载策略模型与冻结参考模型，用 `distilbert-imdb` 情感分类器输出作奖励，KL 散度约束不偏离参考模型——完整展示 Rollout→Evaluation→Optimization 三步。

**（4）GUI 智能体 SFT（`documents/chapter9/GUIagent.ipynb`）**：94 行数据预处理把 OS-Kairos 轨迹（task/previous_actions/score/teacher_action）拼成 sharegpt 多模态格式，再以 Markdown 形式给出 LLaMA-Factory 的 `dataset_info.json` 注册与 `qwen2_vl.yaml` 推理配置，对应论文 arXiv:2503.16465（自适应人机交互 GUI 智能体）。

课件 PDF 覆盖理论部分（如 ch8 的 MLLM 两种架构图解），Notebook 与 PDF 分工明确："PDF 讲原理、README/Notebook 跑代码"。

## 6. 构建与运行

无构建系统。使用方式即" clone 后按章阅读 + 跑 Notebook"（各章 README 给出的命令汇总）：

```bash
# 通用
conda create -n llm python=3.9 && conda activate llm && pip install transformers  # ch1
jupyter notebook sft_math.ipynb     # ch4（需 40GB 显存 GPU、预留 50GB 磁盘）
jupyter notebook llm_stega.ipynb    # ch7（GPT-2，CPU 亦可跑）
jupyter notebook RLHF.ipynb         # ch11，先 pip install -r documents/chapter11/requirements.txt
export HF_ENDPOINT=https://hf-mirror.com; huggingface-cli download <model>  # ch11 的国内镜像下载方式

# 外部工具包按章安装
git clone https://github.com/zjunlp/EasyEdit.git && pip install -r requirements.txt   # ch3
pip install easyjailbreak                                                             # ch6
git clone https://github.com/zwhe99/X-SIR && pip3 install -r requirements.txt         # ch5
```

注意 ch1 的两个"工程包"（TextClassificationCustom/TextClassification）通过夸克网盘链接分发（`documents/chapter1/README.md`），不在仓库内，可复现性一般；ch9 需自行从 OS-Kairos 仓库下载数据集并填空代码中的路径。

## 7. 收录范围与组织方式

- **条目规模**：11 章、12 个 Markdown（2492 行，实测 `cat README.md documents/chapter*/README.md | wc -l`）、11 个 PDF 讲义、55 张配图、128 个文件。每章体量约 100-250 行图文教程，配 2-17 张图。
- **组织维度**：按"主题章节"线性组织而非 awesome 式链接清单；延伸阅读以"论文/仓库链接 + 一句话点评"内嵌在各章末尾（如 ch2 思维链进阶列 Auto-CoT、Sum-CoT、CRITIC、ReAct 与 promptingguide.ai），单章外链约 5-15 个。
- **三件套一致性**：课件 PDF（原理）→ README（图文实操步骤，含安装命令与代码片段）→ ipynb（可执行版或教程副本）。实测 7 章的 ipynb 是 README 的 Notebook 复刻（纯 Markdown），4 章是真实可跑代码——"是否自含代码"取决于该章依赖的外部工具是否已开源。
- **语言与受众**：全中文，面向有大一编程基础的学生；刻意提供国产大模型 API 与 hf-mirror 镜像方案，降低国内网络门槛（ch2/ch11 README）。

## 8. 更新机制与维护状态

- **贡献者结构**：11 人（gh api contributors），以上海交大 BCMI 实验室张倬胜团队师生为主（README 贡献者名单列出张倬胜、袁童鑫/Lordog、马欣贝、何志威/zwhe99、吴铮等 + 新加坡国立大学费豪）；华为昇腾社区 5 人参与第二系列。多个章节作者正是所教上游项目的作者（如 zwhe99 之于 X-SIR/DeepMath、吴铮之于 OS-Kairos），"作者亲自写教程"保证内容准确度。
- **更新节奏**（gh api commits，2024-04 创建至 2025-10-10 最后推送）：2024 年首发 6 章；2025-06-12 "major update" 新增数学推理/GUI Agent/隐写/对齐等 5 章并宣发昇腾课程；此后以 PR 合入为主（#23 Wuzheng02、#24 LZ-Dong）+ README 微调，2025-07-03 补隐写章，10-10 为最后一次 README 更新。距今约 1 年无实质更新，处于"稳定维护/低频更新"状态，README 徽章自述 Status-building、v0.1.0。
- **社区互动**：PRs-Welcome 徽章 + 免责声明欢迎 Issue/PR（README）；当前 open issues 18 个（gh api），无 CI、无自动化检查——内容仓常见形态。星数曲线（README 内嵌 star-history）显示 2025-06 的课程更新与昇腾合作是其破圈增长节点。
- **质量把关方式**：无代码测试可言（教程内容），质量主要靠"上游项目作者审读自身章节"这一隐性机制 + 团队内 PR 互审（如 #23/#24 合入）；仓库无 CONTRIBUTING.md、无 issue 模板（tarball 顶层实测仅 README/.gitignore/documents/pics）。

## 9. 亮点与风险

**亮点**
1. 课程级体系化：11 章成体系且含 5 个安全专题，同类中文入门教程中罕见（`README.md` 目录表）。
2. "作者即教者"：多章由上游项目作者亲自编写（ch4 用 zwhe99 的 DeepMath-103K、ch5 用其 X-SIR、ch9 用吴铮的 OS-Kairos），内容与工具版本对齐度高。
3. 国产生态友好：通义千问/智谱 API、hf-mirror、清华源、昇腾课程，对国内读者开箱成本极低。
4. 4 个自含 notebook 代码量精炼（94-196 行）且附显存/时长参考值（如 ch11 "A800 占用 10097MiB、35min19s"），实验预期明确。
5. 理论/实践分离清晰：PDF 讲义 + 图文教程 + 代码三件套，可分别用于授课与自学。

**风险/局限**
1. **无 LICENSE**：内容版权状态不明，使用者严格意义上仅可"看"不可再分发/改编（gh api `license: null`，仓库无 LICENSE 文件）。
2. 维护趋缓：最后实质内容更新在 2025-07-03（ch7 隐写），最后提交为 README 微调（2025-10-10，gh api），大模型领域迭代快，部分章节（trl 0.9.4、dashscope 老接口）已有版本漂移风险。
3. 链接漂移与历史遗留：ch8 README 指向 `chapter6/mllms.pdf`、ch9 指向 `chapter8/GUIagent.pdf`、ch10 指向不存在的 `chapter7/agents.pdf`（实际为 chapter10/dive-into-safety.pdf）——章节重编号后课件链接未同步，实测均失效；ch9 目录混入 `.DS_Store`。
4. 部分材料走网盘分发（ch1 工程包夸克网盘链接），且 7 章 ipynb 无可执行代码，"动手"深度不均；环境要求两极（ch7 CPU 可跑，ch4 需 40GB 显存）。

## 10. 总结评价

这是一份"高校课程讲义开源"路线的高星标杆：凭借上交大课程背书、作者即项目作者的准确度、安全专题差异化与对国内生态的适配，在一年多内积累 5.7 万+ 星，是大模型中文入门教程中体系最完整、安全视角最强的之一。它不适合当作工程代码库来审读（多数 ipynb 是教程文本而非程序），但非常适合三类专业人：**想快速入门大模型实践的学生**（按章跑通即可）、**需要现成课程骨架的教师**（PDF + 实验直接借用）、**想了解 LLM 安全议题（水印/越狱/隐写/对齐/智能体风险）全貌的工程师**（ch5-7、10-11 是最成体系的中文动手材料）。使用前注意其无许可证的版权限制与个别失效链接。
