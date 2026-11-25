# stanfordnlp/dspy 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：斯坦福 NLP 出品的 LLM 编程框架——用声明式 Signature + 组合式 Python 模块替代手写 prompt，并用"提示编译器"（优化器）自动优化提示与权重。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | stanfordnlp/dspy（原 dspy-ai） | gh api repos/stanfordnlp/dspy |
| 星数 / fork | 39,036（gh api 2026-11-14）/ 3,512 | 同上 |
| 语言 | Python，主包 268 个 .py 约 8.1 万行（含 vendored lm15 81 个文件） | src/dspy/ 统计 |
| 许可证 | MIT | src/LICENSE |
| 版本 | 3.5.0（2026-11-10 发布） | src/pyproject.toml；gh api releases |
| 最近推送 | 2026-11-13（分析前一天仍在提交） | gh api pushed_at |
| 创建时间 | 2023-01-09 | gh api created_at |
| 作者 | Omar Khattab（斯坦福），导师 Chris Potts / Matei Zaharia / Heather Miller | src/pyproject.toml authors；src/docs/docs/roadmap.md |

## 2. 项目是什么，解决什么问题

DSPy（Declarative Self-improving Python）的核心主张是"programming—not prompting"（src/README.md）：手写 prompt 是脆弱的字符串工程，应改为像写 PyTorch 一样写组合式 Python 代码，把"提示词该长什么样"交给编译器自动优化。与 LangChain 类编排框架的差别在于：LangChain 解决"如何把 LM 调用串起来"，DSPy 解决"串起来之后如何让质量可优化、可度量"。

三件套抽象构成它的编程模型（src/docs/docs/roadmap.md 明确表述为"declarative signatures, define-by-run modules, and optimizers"）：

- **Signature**：用 Pydantic 模型/类 docstring 声明任务的输入输出契约，如 `class QA(dspy.Signature): question: str = dspy.InputField(); answer: str = dspy.OutputField()`，docstring 自动成为指令（src/dspy/signatures/signature.py，873 行，`SignatureMeta` 元类实现）。
- **Module**：define-by-run 的程序单元，子类实现 `forward()`，内部组合 Predict/ChainOfThought 等子模块；框架自动枚举所有预测器（`named_predictors()`）、支持 deepcopy/reset_copy（src/dspy/primitives/module.py）。
- **Teleprompter（优化器）**：拿"程序 + 训练集 + 指标"，自动搜索每个预测器的 few-shot 示例（demos）与指令（instructions），产物是一个"编译后"的程序。这是 DSPy 区别于其他框架的核心卖点。

研究血统深厚：README 列出 9 篇论文，从 Demonstrate-Search-Predict（2022-12）到 DSPy 编译器（ICLR 2024）、MIPRO（2024-06）再到 GEPA（2025-07，标题即为"反思式提示进化可胜过强化学习"）（src/README.md）。STORM（维基百科式长文写作）等知名系统也出自同一团队。

2026 年的 DSPy 3.x 已明显"工程化"：3.5 版本（2026-11-10）做了大手术——废弃 LiteLLM 运行时依赖，改为内置 native lm15 引擎（vendored 自 cmpnd-ai/lm15-python），并移除 OpenAI 风格 `messages=` 字典与 `forward()` LM 接口（src/dspy/clients/base_lm.py docstring、src/README.md、src/docs/docs/community/normalized-lm-api-migration.md 迁移指南）。

## 3. 技术栈与依赖

- 语言/运行时：纯 Python，要求 `>=3.10, <3.16`（src/pyproject.toml）。
- 核心依赖（src/pyproject.toml dependencies）：pydantic>=2.11（Signature 即 BaseModel）、orjson、json-repair（容错解析 LM 输出）、diskcache + cachetools（请求缓存）、anyio（异步）、cloudpickle、gepa>=0.1.4,<0.2.0（GEPA 优化器核心包，Python<3.15 才装）。
- 可选 extra：openai、anthropic、weaviate（检索）、mcp、deno（沙箱代码解释器运行时）、langchain、optuna（MIPROv2 用）、numpy、litellm（兼容层，且精确排除了 12 个有 eager MCP import bug 的版本，见 dependencies 注释与 src/dspy/clients/engines/litellm_engine.py）。
- 构建/工具链：setuptools>=77 打包；uv 管理环境（uv.lock，required-version>=0.13.4）；ruff（lint+format，120 列）；ty 类型检查（渐进覆盖 dspy/utils/ 子集）；pytest + pytest-asyncio + pytest-xdist。
- 文档站：mkdocs（src/docs/mkdocs.yml，部署于 dspy.ai）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| src/dspy/signatures/ | Signature 声明式签名（field.py、signature.py） |
| src/dspy/primitives/ | Module/Prediction/Example 基元 + 代码解释器沙箱（python_interpreter.py、runner.js、sandbox_bundle.py） |
| src/dspy/predict/ | 内置模块：Predict、ChainOfThought、ReAct(2)、RLM、BestOfN、Refine、KNN、Parallel、multi_chain_comparison、majority 聚合；flex/（3.5 新抽象 dspy.Flex） |
| src/dspy/adapters/ | 消息格式适配：ChatAdapter、JSONAdapter、XMLAdapter、TwoStepAdapter、BAMLAdapter；types/ 内含 image/audio/tool/citation/document/reasoning/bounding_box 等结构类型；decision.py 为 3.5 新增"决策请求"（Noul/Choice/Score 概率化输出） |
| src/dspy/teleprompt/ | 优化器体系（16 个导出）：BootstrapFewShot 系列、MIPROv2、GEPA/、SIMBA、COPRO、BetterTogether、Ensemble、AvatarOptimizer、InferRules 等 |
| src/dspy/propose/ | MIPROv2 的指令提议器（GroundedProposer） |
| src/dspy/clients/ | LM 抽象层：base_lm.py、lm.py、engines/（lm15_engine、litellm_engine、dummy_engine、streaming）、cache、embedding、provider、model_metadata/（快照自 LiteLLM 元数据，见提交 #10878） |
| src/dspy/_vendor/lm15/ | vendored 原生 LM 引擎（81 个文件），git-subtree 同步自 cmpnd-ai/lm15-python v2.2.0，scripts/update_vendored_lm15.py 维护，provenance 记录 12 个补丁 |
| src/dspy/evaluate/ | Evaluate 并行评估器、metrics、auto_evaluation |
| src/dspy/retrievers/ | ColBERTv2（dsp/colbertv2.py）、WeaviateRM、DatabricksRM、Embeddings |
| src/dspy/dsp/ | v1 时代 DSP 层的残余，现仅剩 colbertv2 与 utils |
| src/dspy/streaming/ | streamify 流式包装、监听器 |
| src/dspy/utils/ | mcp.py（MCP 工具转 dspy.Tool）、asyncify/syncify、saving/loading、usage_tracker、callback |
| src/tests/ | 167 个 .py、5.4 万行，目录结构与主包镜像 |
| src/docs/ | mkdocs 文档站源码 |
| src/scripts/ | lm15 subtree 更新、模型元数据刷新、bootstrap |

组织方式：单包单仓（非 monorepo），主包 + 镜像式测试树 + 内嵌文档站；experimental/ 目录在 3.5 已清空（仅剩 __init__.py），说明 v2 时代实验特性已收敛或移除。

## 5. 核心架构与关键实现

### 5.1 Signature：Pydantic 元类实现的声明式契约

`Signature` 直接继承 `BaseModel`，由 `SignatureMeta` 元类处理：类 docstring 变成默认指令，类属性注解变成输入/输出字段（src/dspy/signatures/signature.py:41, 271）。因此签名可以被程序化变换——`with_instructions()` / `append_instructions()` 返回同字段新签名类，这正是优化器搜索指令空间的操作面（signature.py:276-322）。运行时还有 `dump_state/load_state`，让编译后的程序可序列化保存（dspy/predict/predict.py:74-120，含 `allow_unsafe_lm_state` 安全加载开关，过滤 api_base 等不安全键）。

### 5.2 BootstrapFewShot：最小编译器，理解优化器的钥匙

274 行的 `BootstrapFewShot`（src/dspy/teleprompt/bootstrap.py）展示"提示编译"的本质：teacher 程序在训练集上跑，`dspy.context(trace=[])` 捕获每个预测器的调用轨迹；metric 通过的轨迹被转成 `dspy.Example(augmented=True, **inputs, **outputs)` 塞进 student 对应预测器的 `demos`（bootstrap.py:87-97, 185-259）。多轮 bootstrap 用 `lm.copy(rollout_id=round_idx, temperature=1.0)` 绕过缓存采样多样性（bootstrap.py:194）。所有更强的优化器都构建在这个 trace→demo 机制上。

### 5.3 MIPROv2 与 GEPA：两级指令搜索

- **MIPROv2**（src/dspy/teleprompt/mipro_optimizer_v2.py，866 行）：用 Optuna 贝叶斯搜索联合优化"指令 + few-shot 组合"；指令候选由 `dspy/propose/grounded_proposer.py` 从数据集摘要+程序结构中生成；`auto="light|medium|heavy"` 预设轮数与验证集大小（mipro_optimizer_v2.py:52-56）。
- **GEPA**（src/dspy/teleprompt/gepa/，约 2200 行）：外部 `gepa` 包的适配层。`GEPAFeedbackMetric` 协议要求 metric 可返回 `dspy.Prediction(score, feedback)`——GEPA 的"反思式变异"用自然语言反馈驱动 LLM 重写指令，配合帕累托前沿选择候选（gepa/gepa.py:27-90, DspyGEPAResult 字段含 objective_pareto_front）。若未安装 gepa 包，提供占位类保持 API 兼容（src/dspy/teleprompt/__init__.py）。

### 5.4 LM 层：engine 合同 + vendored lm15

3.5 的 LM 层是"单请求合同 + 框架层横切能力"的设计：engine 只需实现 `complete(Request) -> Response`（可选 stream 与异步副本），而缓存、重试、候选 fan-out、usage 记账、回调、历史都由 DSPy LM 层统一持有（src/dspy/clients/base_lm.py:62-82 docstring）。`dspy.LM` 按模型字符串自动选择内置引擎（`engine: Engine | Literal["auto","lm15","litellm"]`，base_lm.py:135），litellm 降级为可选兼容路径。请求/响应对象来自 vendored `dspy.lm15`（src/dspy/_vendor/lm15/，UPDATING.md 说明 git-subtree 更新流程与 12 个下游补丁）。这一层还支持模块级 decision 请求（Noul/Choice/Score 概率证据，src/dspy/adapters/decision.py）。

值得注意的新模块：`dspy.RLM`（src/dspy/predict/rlm.py）实现 Recursive Language Models——LM 把长上下文当外部环境，写 Python 代码检查/分解上下文并递归调用子 LM（引用 Zhang, Kraska, Khattab 2025 论文，rlm.py:1-9），配套沙箱基础设施在 dspy/primitives/（Deno 运行 runner.js）。

## 6. 构建与运行

```bash
pip install dspy                                  # PyPI 安装
pip install "dspy[openai,anthropic,optuna]"       # 带可选 provider
# 开发环境（uv）
uv sync
uv run pytest -vv -m 'not extra and not deno' -n auto --dist worksteal tests/   # CI 同款测试命令
uv run pytest tests/ -m 'extra or deno' --extra --deno -n auto                    # 扩展测试
uv run pytest -m llm_call --llm_call tests/                                       # 真实 LLM 调用测试
uv run ruff format . && uv run ruff check .       # lint（.github/workflows/precommits_check.yml）
```

来源：src/pyproject.toml、src/.github/workflows/run_tests.yml、CONTRIBUTING.md。

## 7. 测试与工程质量

- 规模：167 个测试文件、约 5.45 万行，与主包 1:0.67 行数比相当高；按子域分布：clients 28、teleprompt 18、adapters 16、predict 16、utils 15、primitives 12 等（src/tests/ 统计）。
- 风格：单测为主、mock LM 优先；tests/README.md 明确"端到端质量评测不在本仓，参考 LangProBe"。含 reliability/（可靠性专项）、typing/（ty 类型契约测试）、docs/（文档代码示例测试）。
- CI（src/.github/workflows/）：run_tests.yml 矩阵 Python 3.10–3.15（3.15 允许失败），xdist worksteal 并行；precommits_check.yml 跑 ruff；dependency-range.yml 做依赖区间兼容；build_and_release.yml 发 PyPI；vendored_lm15.yml 守护 subtree 同步；docs-push.yml 部署文档。
- 质量观察：分支覆盖率配置、`ty` 渐进式类型检查、明显的供应链卫生意识——certifi 钉版注释"distrust the compromised emSign CA"（pyproject.toml:28）、精确排除 12 个坏版本 LiteLLM（pyproject.toml:53）、Predict.load_state 默认剥离不安全 LM 键（predict.py:24-42）。

## 8. 社区与维护状态

- 活跃度：最后推送 2026-11-13（分析前一天）；最近 10 次提交横跨 11-10 至 11-12，密度高（gh api commits）。
- 发版节奏：约每月一版——3.3.0（2026-08-03）、3.3.1（08-21）、3.4.0b1（09-11）、3.4.0（09-25）、3.4.1（10-12）、3.5.0（11-10）（gh api releases）。
- 贡献者：约 410 人（contributors 分页 last=410 推算）；roadmap.md 自述 2024 年即有 200 贡献者、16 万月下载。
- 组织：Project Lead Omar Khattab（Stanford & Databricks），mentor 包括 Chris Potts、Matei Zaharia、Heather Miller（src/docs/docs/roadmap.md）；有 SECURITY.md、CONTRIBUTING.md，社区渠道 Discord。781 个开放 issue（gh api），开源治理成熟。roadmap.md 本身已标注为 2024-08 的历史文档。

## 9. 亮点与风险

**亮点**
1. 学术界最强的"提示优化"工程化：16 个优化器覆盖从最简 Bootstrap 到 MIPROv2（贝叶斯）、GEPA（反思进化）、SIMBA、BootstrapFinetune（权重微调），研究→库的转化路径清晰（src/dspy/teleprompt/__init__.py）。
2. 3.5 的 LM 层重构架构干净：engine 单请求合同把缓存/重试/流式/记账收敛到框架层，vendored lm15 消除 LiteLLM 重依赖，供应商元数据快照化（src/dspy/clients/base_lm.py、src/dspy/_vendor/）。
3. 全套 Pydantic-native 类型系统：Signature 即 BaseModel，adapters/types/ 支持 image/audio/tool/citation 等结构化 I/O，3.5 又加了概率化 decision 请求（src/dspy/adapters/）。
4. 工程质量高：5.4 万行测试、6 个 workflow 的 CI 矩阵、供应链卫生细节到位。
5. 前沿算法快速跟进：RLM、ReActV2、GEPA、Flex 都是 2025-2026 新成果的落地（src/dspy/predict/）。

**风险/局限**
1. 大版本破坏性迁移频繁：3.5 移除 `messages=`/`forward()`/LiteLLM 默认路径，存量代码需按迁移指南改写（src/docs/docs/community/normalized-lm-api-migration.md）。
2. Python 支持上限因 gepa 声明 `<3.15` 而割裂：3.15 上 GEPA 只有报错占位类（src/dspy/teleprompt/__init__.py）。
3. 优化器的真实收益依赖训练集与指标质量，小样本场景 BootstrapFewShot 可能产生噪声 demos（bootstrap.py 的 trace→demo 机制固有局限；端到端收益官方也指向外部 LangProBe 基准）。
4. vendored lm15 带下游补丁栈（12 个 patch），上游在 cmpnd-ai 组织而非本仓，长期维护归属存在不确定性（src/dspy/_vendor/UPDATING.md、lm15-provenance.txt）。

## 10. 总结评价

DSPy 是"提示工程编译器化"这一方向的代表作与事实标准：它把 LM 程序拆成 Signature（契约）、Module（计算图）、Teleprompter（编译器）三层，用机器可搜索的方式替代手写 prompt，背后有斯坦福 NLP 持续五年的论文序列支撑，2026 年的 3.5 版本又完成了 LM 层的去 LiteLLM 化和全面工程加固。适合：构建多步 LM 管线且关心质量优化的 ML 工程师；做 prompt/agent 优化研究的研究者（MIPRO/GEPA 都可直接复用）。想读懂"提示编译"本质的人，建议按 signature.py → predict.py → bootstrap.py → mipro_optimizer_v2.py → gepa/ 的顺序读源码，这五个文件就是整条编译流水线。
