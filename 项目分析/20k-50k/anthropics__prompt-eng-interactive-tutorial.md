# anthropics/prompt-eng-interactive-tutorial 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Jupyter Notebook（Python）｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

Anthropic 官方出品的 Claude 提示词工程互动教程：以 41 个 Jupyter Notebook 组成的 9 章 + 附录课程，配自动判分练习、提示与参考答案，是学习 Claude prompt engineering 事实上的官方入门教材。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | anthropics/prompt-eng-interactive-tutorial | gh api |
| 描述 | Anthropic's Interactive Prompt Engineering Tutorial | gh api |
| 星数 | 39280（live，2026-11-14）／快照 39500 | gh api |
| Fork | 4353 | gh api |
| 内容 | 51 个文件，3 套变体共 41 个 .ipynb | 解压统计 |
| 主语言 | Python（Jupyter Notebook；无 TypeScript 版本） | 实际文件 |
| 许可证 | 仓库级无 LICENSE（license=null）；仅 `AmazonBedrock/` 目录含 MIT No Attribution | gh api + `AmazonBedrock/LICENSE` |
| 默认分支 | master，最后内容提交 2024-04-08 | gh api commits |
| 最近推送 | 2026-08-28（仅为侧分支依赖安全刷新） | gh api branches/commits |
| 贡献者 | 2 人（Maggie Vo、Jawhny Cooke） | gh api contributors |
| Release | 0 | gh api releases |

## 2. 项目是什么，解决什么问题

这是 Anthropic 官方教程仓库（`README.md`），目标是让读者"系统掌握 Claude 提示词工程"：掌握好 prompt 的基本结构、识别常见失败模式并学会'80/20'技巧、理解 Claude 的强弱项、为常见场景从零构建强 prompt。课程按 Beginner（1-3 章：基本结构/清晰直接/角色分配）、Intermediate（4-7 章：数据与指令分离/输出格式与代答/思维链/少样本示例）、Advanced（8-9 章：避免幻觉/从零构建复杂 prompt）分阶，另有附录（提示链、工具使用、检索）。

与其他 prompt 教程最大的区别是**可动手**：每个 notebook 都接真实 API，每章末尾有带自动判分的练习（`grade_exercise()` 正则校验 Claude 回答并打印 True/False），每课底部留 "Example Playground" 自由实验区，另有外部 Google Sheets 答案册（`README.md`）。教程刻意使用最小最便宜的 Claude 3 Haiku、temperature 0，降低学习门槛（`README.md`、`Anthropic 1P/00_Tutorial_How-To.ipynb`）。

仓库同时存在三个平行变体：直连 API 的 `Anthropic 1P/`、走 `AnthropicBedrock` SDK 的 `AmazonBedrock/anthropic/`、走 boto3 原生 `invoke_model` 的 `AmazonBedrock/boto3/`。Bedrock 版由 Anthropic 与 AWS 合作维护（`AmazonBedrock/CONTRIBUTING.md` 为 Amazon 标准模板，配套 SageMaker CloudFormation 模板一键拉起 workshop 环境，`AmazonBedrock/cloudformation/workshop-v1-final-cfn.yml`）。

README 自己承认 Google Sheets 版（Claude for Sheets 扩展）"更易用，推荐使用"——GitHub 仓库实际上是教程的第二载体，这也是它作为"官方背书物件"而非高频迭代工具仍能持续吸星的背景。

## 3. 技术栈与依赖

- **载体**：Jupyter Notebook（41 个 .ipynb），纯 Python，无 TypeScript/JS 版本（按实际文件统计，任务提示中的"Python/TS 双语"在本仓库不成立）。
- **直连版依赖**：仅 `anthropic` SDK（notebook 内 `!pip install anthropic`，见 `Anthropic 1P/00_Tutorial_How-To.ipynb` cell 2）。
- **Bedrock 版依赖**（`AmazonBedrock/requirements.txt`）：`awscli==1.32.74`、`boto3==1.34.74`、`botocore==1.34.74`、`anthropic==0.21.3`、`pickleshare==0.7.5`——全部钉死在 2024 年 4 月的版本。
- **模型**：`claude-3-haiku-20240307`，temperature 0.0，max_tokens 2000（`Anthropic 1P/00_Tutorial_How-To.ipynb` cell 4/7）。
- **跨 notebook 状态**：用 IPython 魔法 `%store API_KEY / %store MODEL_NAME` 在 13 个 notebook 间共享变量（cell 4）。
- **无构建系统、无测试框架、无 CI 配置**——这是纯内容仓库，工程面仅一个 CloudFormation 模板（67 行，`AmazonBedrock/cloudformation/workshop-v1-final-cfn.yml`）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `README.md` | 总课程介绍、目录（9 章 + 附录）、答案册链接 |
| `Anthropic 1P/` | 直连 API 版：13 个 notebook（00-09 章 + 10.1-10.3 附录）+ `hints.py` |
| `AmazonBedrock/anthropic/` | Bedrock + Anthropic SDK 版：14 个 notebook（多出 10_3 实证评估附录）+ 空格命名异常的 `07_Using_Examples _Few-Shot_Prompting.ipynb` |
| `AmazonBedrock/boto3/` | Bedrock + boto3 原生版：14 个 notebook |
| `AmazonBedrock/utils/` | `__init__.py` + `hints.py`（包化，供两套 Bedrock notebook `from utils import hints`） |
| `AmazonBedrock/cloudformation/` | SageMaker workshop 一键部署模板（KMS 加密、IAM 角色、克隆仓库） |
| `AmazonBedrock/CONTRIBUTING.md`、`LICENSE` | Amazon 标准贡献指南 + MIT-0 许可证（仅覆盖 Bedrock 版） |
| `AmazonBedrock/README.md` | Bedrock 版课程说明（比根 README 多一条"实证评估"附录） |
| `.gitignore` | 仅忽略 `.DS_Store` |

组织方式评述：三套变体是**内容复制 + 手工同步**，非代码复用式 monorepo。三个 `hints.py` 逐字节一致（diff 验证 IDENTICAL），课程正文在三套间同样近乎复制——约 1.08MB notebook 源文本中 1P 版占 353KB，重复率约 2/3。命名不规范：目录名含空格（`Anthropic 1P`），Bedrock anthropic 版第 7 章文件名有游离空格，1P 附录用 `10.1` 而 Bedrock 用 `10_1`。

## 5. 核心架构与关键实现

内容型仓库，"核心实现"即互动练习机制，全部在 notebook 与 `hints.py` 内：

**(1) 每章统一骨架**（`Anthropic 1P/01_Basic_Prompt_Structure.ipynb` outline，32 cell）：Setup（装 SDK + `%store` 取 key）→ Lesson（可运行示例）→ Exercises（占位 `PROMPT = "[Replace this text]"`）→ Example Playground。全 13 个 notebook 共 346 cell（169 code + 177 md），约 353KB 源文本。

**(2) 自动判分器**：每个练习 cell 内嵌正则判分，直接对 Claude 的真实回答打分，打印 `This exercise has been correctly solved: True/False`：

```python
# Anthropic 1P/01_Basic_Prompt_Structure.ipynb cell 17
def grade_exercise(text):
    pattern = re.compile(r'^(?=.*1)(?=.*2)(?=.*3).*$', re.DOTALL)
    return bool(pattern.match(text))
print("This exercise has been correctly solved:", grade_exercise(response))
```

判分风格是"关键词/格式断言"（如查 "hola"、"brown"、`<haiku>` 标签、≥800 词），宽松且面向教学反馈，非严格测试。

**(3) 提示与答案库**（`Anthropic 1P/hints.py`，12.7KB）：为 19 道练习各备一条 `exercise_X_Y_hint`（如 6.1 邮件分类题给出 4 步拆解思路），并为 5 道难题附完整参考解（`exercise_6_1_solution`、`7_1_solution`、`9_1_solution`、`9_2_solution`、`10_2_1_solution`）。9.1 税务 chatbot 参考解内嵌真实美国税法条文摘录 + `<quotes>`→`<answer>` 两段式结构，是全仓库最有含金量的 prompt 模板。练习计时：第 1 章 2 题、第 2 章 3 题、第 3 章 1 题、第 4 章 3 题、第 5 章 3 题、第 6 章 2 题、第 7 章 1 题、第 8 章 2 题、第 9 章 2 题、附录 10.2 一题，合计 19 题（各 notebook "### Exercise" 标题统计）。

**(4) 长文档真实感**：第 8 章以 3 个约 33KB 的内嵌长文档（含"干扰信息"文档、招股书）做幻觉演示与练习（`Anthropic 1P/08_Avoiding_Hallucinations.ipynb` cell 8/10/18），第 9 章 69KB 覆盖 chatbot/法律/金融/编码四个行业案例——是两套 Bedrock 变体之外真正的差异化内容。

**(5) Bedrock 双通道**：同一课程两条接入路径——`AnthropicBedrock(aws_region=...)` 走 SDK（`AmazonBedrock/anthropic/01_Basic_Prompt_Structure.ipynb` cell 1），或 `boto3.client('bedrock-runtime')` + 手工 `json.dumps` 请求体走原生（`AmazonBedrock/boto3/01_...ipynb` cell 1）。Bedrock 版故意保留 2 个"错误格式必报错"的教学 cell（前一 markdown cell 明确标注 ⚠️ 预期行为），初读像 bug、实为设计。

## 6. 构建与运行

无构建、无测试。运行路径（来自 `Anthropic 1P/00_Tutorial_How-To.ipynb`）：

```bash
pip install anthropic        # 或 Bedrock 版: pip install -r AmazonBedrock/requirements.txt
jupyter notebook             # 从 00_Tutorial_How-To.ipynb 开始按章序运行
```

notebook 内设置 `API_KEY`、`MODEL_NAME = "claude-3-haiku-20240307"` 后 `%store` 共享。Bedrock workshop 路径：用 CloudFormation 控制台部署 `AmazonBedrock/cloudformation/workshop-v1-final-cfn.yml`（67 行）在 SageMaker 上拉起带 Bedrock 权限的 notebook 实例；模板默认 repo URL 实际指向 AWS 的姊妹仓库 `aws-samples/prompt-engineering-with-anthropic-claude-v-3.git`，体现其 workshop 定位。不想跑 API 的用户被引导到 Google Sheets 版答案册（`README.md`）。

## 7. 教程内容与互动练习设计（内容型特例）

**章节主题统计**（41 notebook 实测，以 `Anthropic 1P/` 文件名为准）：

| 章 | notebook | 主题 | 练习 | 备注 |
|---|---|---|---|---|
| 00 | 00_Tutorial_How-To | 环境配置、get_completion 助手 | 0 | `%store` 共享 key/模型名 |
| 1 | 01_Basic_Prompt_Structure | Messages API 格式、user/assistant/system 三段结构 | 2 | 判分器查 "1,2,3" 等 |
| 2 | 02_Being_Clear_and_Direct | 清晰直接表达 | 3 | 查 "hola"、≥800 词等 |
| 3 | 03_Assigning_Roles_Role_Prompting | 角色分配 | 1 | 数学题 + 专家角色 |
| 4 | 04_Separating_Data_and_Instructions | 数据/指令分离、XML 标签分隔符 | 3 | 查 "brown"、"haiku"、"pig" |
| 5 | 05_Formatting_Output_and_Speaking_for_Claude | 输出格式、代 Claude 说话（prefill） | 3 | 查 "Warrior"、`<haiku>` 标签 |
| 6 | 06_Precognition_Thinking_Step_by_Step | 思维链（先想后答） | 2 | 邮件分类实战 |
| 7 | 07_Using_Examples_Few-Shot_Prompting | 少样本示例 | 1 | 邮件格式化 |
| 8 | 08_Avoiding_Hallucinations | 幻觉避免：给出口 + 引用提取 | 2 | 内嵌 3 个 ~33KB 长文档，全仓最大章（170KB） |
| 9 | 09_Complex_Prompts_from_Scratch | 行业实战：chatbot/法律/金融/编码 | 2 | 税法 chatbot + Socratic Codebot |
| 附 10.1 | 10.1_Appendix_Chaining Prompts | 提示链 | 0 | — |
| 附 10.2 | 10.2_Appendix_Tool Use | 工具使用（JSONSchema） | 1 | SQL 工具编排 |
| 附 10.3 | 10.3_Appendix_Search & Retrieval | 检索/RAG | 0 | 1156 字节占位页，跳转 anthropic-cookbook |

Bedrock 两套变体比 1P 多一个实内容附录 10_3 实证性能评估（10KB，`AmazonBedrock/anthropic/`），检索附录（其 10_4）同为占位页。主题与 Anthropic 官方文档 prompt engineering 指南一一对应，官方文档本身就是该课程的结构化"文档版"。

**互动设计三层反馈**：①每题内嵌判分器即时 True/False；②`hints.py` 按需提示（"❓ If you want a hint, run the cell below!"）；③外部 Google Sheet 答案册兜底。加上每章底部 Playground 和 `%store` 跨章续跑，形成完整的自学闭环——这在 2024 年的 LLM 教程里是罕见的完成度。

**示例代码质量**：Python 风格统一、类型标注零散存在（`def get_completion(prompt: str)`），SDK 调用全部走 Messages API 正规姿势；缺点是版本钉死（anthropic 0.21.3）、无异常处理、判分器偏脆弱。三个变体间内容一致性靠人肉保证（hints 三份 diff 全等可证），同步成本高。

**与官方生态的同步关系**：仓库内容已于 2024-04-08 冻结，而 Anthropic 官方文档与 cookbook（如 `10.3` 占位 notebook 直接跳转 `anthropic-cookbook` 的 RAG 示例）持续演进；教程仍教 Claude 3 Haiku，社区 issue "Tutorial using Haiku 3- needs update"（2026-04-14）公开确认过时。即：本仓库是"快照型官方教材"，生态同步靠指向外部链接而非更新本体。

## 8. 品牌效应与增长（gh api 核实时间线）

时间线（gh api commits/branches/issues 实证）：2024-04-02 建仓（Claude 3 系列 3 月发布后的窗口期）；04-06 至 04-08 集中提交全部内容并由 Jawhny Cooke 对齐 aws-samples 姊妹仓、补许可与贡献指南；此后 **master 默认分支 2.5 年零提交**。2026-03-01 dependabot 提 awscli 补丁分支、2026-08-28 `jasonnoll/sec-refresh-aug28-awscli` 分支提"Bump awscli/boto3/botocore (3 alerts)"——所谓最近 push 只是未合并的依赖安全分支。

增长归因：39.2k 星几乎全部来自**Anthropic 品牌背书 + 内容常青**——它是 Claude prompt 工程的"官方第一入口"，被无数 Claude 教程/文章引用；课程内容（清晰、分隔符、角色、CoT、少样本、幻觉）是跨模型代际的常青知识，故内容冻结并不妨碍持续涨星。仅 2 名贡献者、64 个 open issue（历史共 43 个关闭）、0 release、无 CI——是典型"官方发布一次即成公共品"的增长曲线，与同星数级活跃开源项目完全不同。

## 9. 亮点与风险

**亮点**
1. 官方一手教程 + 三层反馈互动设计（判分器/提示/答案册），自学闭环完成度极高（`Anthropic 1P/01_*.ipynb`、`hints.py`）。
2. 三变体覆盖直连 API、Bedrock-SDK、Bedrock-boto3，并配 SageMaker CloudFormation，直接服务企业 workshop 场景（`AmazonBedrock/`）。
3. 第 8/9 章用 33KB 级真实长文档与行业案例（税法 chatbot、Socratic Codebot）做实战，非玩具示例（`08_*.ipynb`、`09_*.ipynb`）。
4. 主题体系与官方文档一一映射，是理解 Anthropic prompt engineering 方法论的"习题版"。

**风险/局限**
1. 内容冻结于 2024-04（Claude 3 Haiku、anthropic 0.21.3），与 2026 年的模型/API 已脱节，社区 issue 明确求更新（"Tutorial using Haiku 3- needs update"）。
2. 仓库级无 LICENSE（仅 `AmazonBedrock/LICENSE` MIT-0），1P 主内容的再分发授权状态不明。
3. 三变体手工复制同步，约 2/3 冗余；文件命名有空格等瑕疵，判分器为脆弱的正则断言。
4. 维护实质停滞（默认分支 2.5 年零提交、0 release、2 贡献者），后续修错依赖外部 PR 自流。

## 10. 总结评价

一个"一次发布、长期吸星"的官方内容型仓库：靠 Anthropic 品牌与完整的三变体互动课程在两周末建成，随后 2.5 年不动仍稳居 39k 星——星数衡量的是官方教程的生态位而非工程活跃度。它仍是学习 Claude（乃至通用 LLM）提示词工程的最佳入门练习册之一，适合 AI 初学者按章刷题、团队培训/workshop 直接部署 Bedrock 版；值得读"源码"的是 `hints.py` 里的参考解与第 8/9 章的长文档实战模板，而非工程代码。使用者需自行换用新版模型并对照最新官方文档校准细节。
