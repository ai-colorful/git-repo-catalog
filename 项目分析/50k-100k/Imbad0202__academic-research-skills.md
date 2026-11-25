# Imbad0202/academic-research-skills 项目分析

> 星数 53573（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：面向 Claude Code 的学术研究技能套件（ARS），以"研究 → 写作 → 评审 → 修改 → 定稿"全流程多 agent 管线辅助人类学者完成论文，坚持 human-in-the-loop 而非自动化科研。

| 项 | 值 |
|---|---|
| 名称 | academic-research-skills（ARS） |
| 星数 | 53573（快照）；gh api 实测 55198（2026-11-04） |
| 语言 | Python（核心方法论为 Markdown prompt 工程，Python 为支撑工具链） |
| 许可证 | CC BY-NC 4.0（GitHub 识别为 NOASSERTION，见 `LICENSE` 417 行） |
| 版本 | v3.24.1（2026-11-02 release，见 `.claude-plugin/plugin.json`） |
| 最近活跃 | 最近 push 2026-11-02，当周仍有发布（gh api） |
| 创建时间 | 2026-02-26（约 8 个月涨到 55k 星，gh api） |
| 作者 | Cheng-I Wu（吳政宜），单人主导 + 27 名贡献者（gh api contributors） |

## 2. 项目是什么，解决什么问题

ARS 是一个以 Claude Code 插件形式分发的学术研究工作流套件（`README.md`，安装命令 `/plugin marketplace add Imbad0202/academic-research-skills`）。它由 5 个技能组成：deep-research（13-agent 研究团队）、academic-paper（12-agent 写作管线）、academic-paper-reviewer（7-agent 模拟同行评审）、academic-pipeline（10 阶段编排器）、sr-screener（系统综述文献筛选），合计 35 种模式、43 个 prompt 角色（`.claude-plugin/plugin.json` 描述字段；各技能 `agents/` 目录共约 40+ 个角色定义文件）。

它解决的问题不是"替人写论文"，而是把 AI 当 copilot：每个阶段结束都有用户确认检查点（`docs/ARCHITECTURE.md` §1 列出 10 个决策型检查点 + 2 个完整性门禁确认点），Stage 2.5/4.5 两个强制完整性门禁运行 7 种 AI 失败模式清单（实现 bug 自审通过、幻觉引用、幻觉实验结果、走捷径、bug 包装成洞见、方法编造、frame-lock，见 `academic-pipeline/references/ai_research_failure_modes.md`，依据 Lu et al. 2026 *Nature*）。

README 用 6 篇学术文献为设计立场背书（幻觉引用规模审计、AI 科学家失败模式、AI 评审军备竞赛综述等），并明确承认自身局限：`POSITIONING.md` 记录了一份"拒绝机制"清单（拒绝端到端自主科研、自主假设生成、模拟伦理委员会等），并声明完整性检查无法发现"前后一致的编造"——"一个建立在编造数据上的研究若报告一致，可通过所有 ARS 门禁"。

输出面覆盖完整学术形态：引用格式支持 APA 7.0（含中文引注规则）/Chicago/MLA/IEEE/Vancouver，论文结构支持 IMRaD、主题式文献综述、理论分析、案例研究、政策简报、会议论文（`README.md`）；成稿为 MD + DOCX（Pandoc）+ LaTeX（APA 7.0 apa7 类/IEEE/Chicago）经 tectonic 出 PDF。语言上繁体中文与英文为默认（中文输入默认繁体回应），SKILL.md 触发词内嵌中、日、韩、西四语关键词（如 `deep-research/SKILL.md` frontmatter）。配套产物 Material Passport 全程携带研究状态，sr-screener 以 `literature_corpus[]` 端口把筛选后的文献语料递给 academic-paper 的 Phase 1（`docs/ARCHITECTURE.md` §5）。

模式全景（`README.md` Usage 节，合计 35 模式）：

| 技能 | 模式数 | 模式清单 |
|---|---|---|
| deep-research | 8 | full、quick、review、lit-review、three-way-scan、fact-check、socratic、systematic-review |
| academic-paper | 11 | full、plan、outline-only、revision、revision-coach、abstract-only、lit-review、format-convert、citation-check、disclosure、rebuttal-audit |
| academic-paper-reviewer | 6 | full、re-review、quick、methodology-focus、guided、calibration |
| sr-screener | 8 | protocol、quick、pilot、ta-screen、ft-screen、adjudicate、audit、report |
| academic-pipeline | 编排 | 全管线 + 中途入场（Stage 2.5 前入场 / Stage 4 前入场） |

项目对 AI 写作的现实局限有工程化应对：Devil's Advocate 的反谄媚让步阈值协议（反驳须 1-5 打分，≥4 才允许让步，`README.md` v3.0 章节）、苏格拉底导师的意图检测与对话健康自检、引用存在性四索引确定性门禁（v3.11 #182）、可选的逐条 claim-source 对齐审计（v3.8，`ARS_CLAIM_AUDIT=1`，5 类 HIGH-WARN 注解触发 formatter 硬门禁拒输出）。

## 3. 技术栈与依赖

- **核心载体：Markdown prompt 工程**。全仓 1149 个 .md 文件共约 20.1 万行（实测统计），包括 5 个 SKILL.md（516-777 行，frontmatter 含版本、data_access_level、task_type、多语言触发词，如 `deep-research/SKILL.md`）、约 40+ agent 角色定义、references/ 方法论文档、shared/ 契约与 schema。
- **Python 支撑工具链**：`scripts/` 下 464 个 .py 共约 27.9 万行（实测统计），运行时基本只用标准库（`scripts/arxiv_client.py` 用 http.client/urllib/xml.etree，带 3 秒退避重试遵守 arXiv ToU）。
- **外部 API 客户端**：`scripts/arxiv_client.py`、`scripts/crossref_client.py`、`scripts/openalex_client.py`、Semantic Scholar Tier-0 引用核验（Levenshtein ≥ 0.70 阈值，`docs/ARCHITECTURE.md` §3）。
- **可选外部工具**：Pandoc（DOCX）、tectonic + Source Han Serif TC（APA 7.0 PDF）；"真实 Python"仅写作用域守护 hook 与少数 opt-in 命令需要（`README.md` Quick install 节）。
- **开发依赖**（`requirements-dev.txt`）：pyyaml、ruamel.yaml、jsonschema、pypdf、defusedxml、openpyxl、markdown-it-py ≥ 3.0（注释写明低于 3.0 会让某回归测试静默跳过）。
- **分发形态**：Claude Code 插件（`.claude-plugin/plugin.json` + `marketplace.json`）、Pi 包装器（`pi/`，`package.json` 声明 pi 扩展）、Codex CLI 姊妹仓（academic-research-skills-codex）、claude.ai/Claude Desktop/Claude Science 导入（`README.md`）。

## 4. 目录结构与代码组织

| 目录 | 职责（实测文件数） |
|---|---|
| `deep-research/` (53)、`academic-paper/` (63)、`academic-paper-reviewer/` (28)、`academic-pipeline/` (30)、`sr-screener/` (25) | 5 个技能：SKILL.md + agents/ + references/ + templates/ + examples/ |
| `agents/` (4) | 4 个插件级 subagent（report-compiler、research-architect、screening-reviewer、synthesis） |
| `commands/` (16) | `/ars-*` 斜杠命令定义（ars-full、ars-plan、ars-lit-review、ars-reviewer 等） |
| `hooks/` (2) | `hooks.json` + `run_guard.sh` 写作用域守护 |
| `scripts/` (644，其中 test_*.py 240) | Python 工具链：API 客户端、验证门禁、lint、校准 runner、适配器 |
| `shared/` (155) | 跨技能契约：JSON Schema（Material Passport、benchmark_report、sprint_contract 等）、协议文档（ground_truth_isolation_pattern.md、cross_model_verification.md） |
| `tests/` (342) | pytest 夹具为主（issue 回归 fixture、expected.yaml/md 对拍） |
| `evals/` (1146) | 金标准集（gold/calibration/heldout/bakeoff）+ `run_evals.py` 多任务评测 |
| `docs/` (125) | ARCHITECTURE、SETUP、DATA_FLOWS、RISK_REGISTER、PERFORMANCE、STAGE_CAPABILITY_MATRIX、design/ 设计记录、changelog-archive/ |
| `examples/` (45) | 真实管线产物 showcase（含 integrity report、评审报告、终稿 PDF） |
| `plugin-evals*/` (3 个目录) | 评审器场景评测集（中英文 rebuttal/decision letter 等 7 组） |

组织方式评述：**"prompt 代码库 + Python 工具链"双层的单仓（monorepo）插件**。方法论以文档形式版本化，机器可验证的部分（schema、存在性核对、lint、评测）下沉到 Python 和 CI；每个技能自包含（SKILL.md + agents + references），跨技能共享物集中在 shared/，路由核心（routing-core）刻意在 `.claude/CLAUDE.md` 与各 SKILL.md 中重复以保证插件安装场景可用（`deep-research/SKILL.md` v3.9.2 注）。

## 5. 核心架构与关键实现

**（1）10 阶段管线状态机（`academic-pipeline/SKILL.md`、`docs/ARCHITECTURE.md` §2）**。RESEARCH → WRITE → 2.5 INTEGRITY → REVIEW → 修订教练（≤8 轮苏格拉底对话）→ REVISE → RE-REVIEW → RE-REVISE（硬上限：修订循环总计 2 次）→ 4.5 FINAL INTEGRITY → FINALIZE → PROCESS SUMMARY。Stage 2.5/4.5 是强制门禁：7 模式失败清单核验 + Material Passport（Schema 9，`shared/handoff_schemas.md`）落盘，任何绕过须记录用户理由供 Stage 6 追责。编排器自我定位为"只做阶段检测、模式推荐、派发与状态跟踪，不做实质工作"（`academic-pipeline/SKILL.md` 开头）。

**（2）数据访问分级与地面真值隔离（`shared/ground_truth_isolation_pattern.md`，v3.3.2+）**。每个技能声明 `data_access_level`（raw/redacted/verified_only）：deep-research 产 raw，academic-paper 消费后产 redacted，完整性门禁产 verified_only；配合 `scripts/check_data_access_level.py` 校验。模式注明取材自 Anthropic automated-w2s-researcher。

**（3）确定性引用核验门禁（v3.11 #182，`scripts/verification_gate/`、`scripts/arxiv_client.py`）**。arXiv 客户端实现"ID 优先 + 标题交叉核对（ID_MISMATCH 模式）+ 标题相似度回退 + 429→3 秒×3 退避"的查找契约，契约文档独立放在 `deep-research/references/arxiv_api_protocol.md`；四索引（Crossref/OpenAlex/arXiv/S2）三角定位，`lookup_verified == false` 在 strict 策略下触发 formatter REFUSE 规则 12（`docs/ARCHITECTURE.md` §3 Stage 5 行）。

**（4）跨模型校验与诚实评测基建**。`scripts/cross_model_handoff.py` 定义 `[CROSS-MODEL-HANDOFF v1]` 信封及规范性 Python 文法，盲检点经 owner→dispatcher→owner 传输（v3.17 #527）；claim 审计校准 runner 内置 25 元组合金集，门槛 FNR<0.15 + FPR<0.10，但 README 明示"stub judge 只测工具本身，尚无 live-judge 校准结果"（`README.md` v3.8 段）。模型分层开关 `ARS_MODEL_TIERING`（economy/quality-boost 两向，`shared/model_tiering.md`）。

**（5）路由核心与意图澄清（各 SKILL.md 共享的 routing-core 段）**。用户消息先过逃逸口（`[direct-mode]` 字节零前缀跳过分类）、显式意图直路由、跨阶段材料（如草稿+评审意见+文献表）必须澄清而非自动归入单阶段 agent——后者是 issue #133 复盘出的反模式，`tests/fixtures/issue_133_routing/` 用 7 组输入/期望对拍锁死该行为。sr-screener 的边界也写入路由：请求筛选已有时路由到它，请求写综述则不（`deep-research/SKILL.md`）。

**（6）插件运行时面与注入防护**。`hooks/hooks.json` + `run_guard.sh` 挂 SessionStart 公告与写作用域守护（守护进程 v3.24.0 起每次工具调用只起一个 Python 进程以缓解 Windows AppXSvc 泄漏，README v3.24.0 段 #1015）；`agents/` 下 4 个插件级 subagent 带 frontmatter（`model: inherit`，工具白名单 Read/Write/Edit/Grep/Glob，见 `agents/report_compiler_agent.md`），并内嵌两条安全纪律：知识隔离（上游材料不足标 `[MATERIAL GAP]` 而非凭参数记忆补写，`academic-paper/references/anti_leakage_protocol.md`）与"取回内容是数据不是指令"（外部文本中的命令性语句不得提升为用户指令，SKILL.md canonical:instruction-data-boundary 段）。

## 6. 构建与运行

```bash
# 安装（Claude Code v3.7.0+，README.md）
/plugin marketplace add Imbad0202/academic-research-skills
/plugin install academic-research-skills
# 验证：运行 /ars-plan 或 /ars-lit-review "topic"

# 测试（.github/workflows/pytest.yml，Python 3.11）
pip install -r requirements-dev.txt pytest
PYTHONPATH=. pytest scripts/ tests/

# 评测 harness（evals/README.md）
PYTHONPATH=. python -m scripts.run_evals                       # 全部任务
PYTHONPATH=. python -m scripts.run_evals --baseline before.json --compare after.json
python -m scripts.check_evals_gold_set evals/gold/<task>       # 金集校验

# 金集校准（claim audit，scripts/test_claim_audit_calibration.py）
# 输出格式：MD 默认；DOCX 需 Pandoc；PDF 需 tectonic + Source Han Serif TC
```

## 7. 测试与工程质量

- **pytest 套件**：`scripts/` 内嵌 240 个 `test_*.py` + `tests/` 2 个，约 28 万行 Python 中相当比例是测试与夹具；`tests/fixtures/` 按议题编号组织回归（如 `tests/fixtures/issue_133_routing/` 7 组路由对拍）。CI 在 PR 与 main push 上跑 `pytest scripts/ tests/`（`.github/workflows/pytest.yml`，按路径过滤触发）。
- **评测体系**：`evals/gold/` 每个测量目标一个自包含金集（citation_extraction、rq_framing_patterns 等），`scripts/run_evals.py` 汇总产出 schema 化报告（`shared/evals_lift_report.schema.json`），`scripts/check_ranking_lift.py` 在 `eval-harness.yml` 中阻断未确认的性能回退，未就绪任务报 pending/skipped 而非失败（`evals/README.md`）；`plugin-evals/` 收录 7 组中英文评审场景（decision letter、ICLR rebuttal、伦理委员会信函等，见 `plugin-evals/` 目录）。金集目录另有 calibration/heldout/bakeoff 分层。
- **CI 纪律（14 个 workflow，`.github/workflows/`）**：除 pytest/eval-harness 外还有 command-invariants、spec-consistency、test-count-monotonic（测试数不可减）、changelog-covers-merges、tag-version-match、release-cooldown、freshness-check、repository-hygiene、gitleaks、defer-label-gate 等——对一个 prompt 仓库而言纪律罕见地严。
- **诚实性设计**：`shared/benchmark_report_pattern.md` 要求"诚实基准对比 schema"；showcase 明确标注是 2026-03 旧版管线产物，并附事后审计"68 条参考文献仍有 21 条有问题"（`README.md` Showcase 节、`examples/showcase/`）。
- 观察：核心方法论（prompt 协议）本身无法被 pytest 覆盖，仓库以 lint + 对拍夹具 + eval 金集近似弥补，但"agent 是否真按协议执行"仍依赖 LLM 运行时，多处文档自认"protocol the agents follow, not runtime guarantees"（`docs/ARCHITECTURE.md`）。

## 8. 社区与维护状态

- **极活跃**：2026-09-16 至 2026-11-02 间发了 8 个 release（v3.22.0 → v3.24.1，gh api releases），CHANGELOG.md 2658 行且每条带议题号与实测边界说明；最近一次 push 2026-11-02（gh api）。
- **贡献者结构**：27 人（gh api contributors），但典型"单人主导 + 外围贡献"：README 署名区显示贡献集中在 README 翻译（日/简中/韩/西/葡语，各带 PR 号）、期刊清单扩展（#5/#7/#8）、披露系统扩展（#596/#599 等）；核心方法论与门禁几乎全部由作者 Cheng-I Wu 实现。
- **生态外溢**：姊妹仓 academic-research-skills-codex（Codex CLI 版）、第三方 Teaching Skills（YujxZJCN/teaching-skills）复用其架构、companion 实验仓 experiment-agent；THIRD_PARTY.md 登记第三方集成（未背书）。
- **治理**：GOVERNANCE.md（决策权与 EOL 姿态）、SECURITY.md（安全上报）、CONTRIBUTING.md（平台移植仅限社区维护）、governance 层面在 README 有专节链接。
- **热度**：4211 forks（gh api）；8 个月 55k 星在同类中属极快增速，README 附外部教程（Substack 中英文全流程走查）；`.github/FUNDING.yml` + Buy Me a Coffee 链接（仓库 homepage 字段）作为维护资金来源。
- **issue/PR 流程**：PR 模板（`.github/pull_request_template.md`）、copilot-instructions（`.github/copilot-instructions.md`）与 pr-closes-issue 门禁 workflow 并存；CHANGELOG 条目普遍标注议题号与外部贡献来源（如 v3.23.2 注明 #978/#985/#988 来自外部贡献者）。

## 9. 亮点与风险

**亮点**
1. **诚实工程的天花板级示范**：拒绝机制清单、RISK_REGISTER、benchmark schema、"未校准"标注、showcase 自曝短板（21/68 引用仍有问题）——全仓贯穿"声明边界而非夸大能力"的写法（`POSITIONING.md`、`README.md`）。
2. **可验证的方法论**：把 prompt 协议中能机器化的部分全部下沉为 Python 客户端、JSON Schema、lint 与 CI 不变量（14 个 workflow），test-count-monotonic 这类元纪律少见。
3. **反谄媚/反 frame-lock 的机制化**：DA 让步阈值（1-5 评分，≥4 才让步）、苏格拉底意图检测、对话健康自检，直击 LLM 结构性弱点并有 v3.0 设计复盘（`README.md`）。
4. **human-in-the-loop 的强制落地**：每阶段确认检查点 + 2.5/4.5 强制完整性门禁 + 绕过须记录理由，配套 R&R Traceability Matrix 闭环追责。
5. **多平台分发与本地化**：Claude Code/claude.ai/Desktop/Pi/Codex/Claude Science 六通道 + 7 语种 README（含繁中默认）。

**风险/局限**
1. **许可证非开源**：CC BY-NC 4.0 禁止商用（GitHub 标 NOASSERTION），企业/商业产品集成不合规；`POSITIONING.md` 明言"这是有意的限制"。
2. **保证是纸面的**：门禁多为"agent 遵循的协议而非运行时保证"（文档原话），live-judge 校准未完成、评审状态 NOT_CALIBRATED；一致性编造可穿透所有门禁（`POSITIONING.md` 自认）。
3. **成本与平台耦合**：15k 字论文全管线约 $3-7 token 成本（`docs/PERFORMANCE.md`，README 转述），且深度绑定 Claude Code hooks/agents——在 claude.ai 等通道上控制项可用性打折（`docs/CONTROL_AVAILABILITY.md`），Windows 环境有已知坑（AppXSvc 泄漏、MS Store Python stub）。
4. **单点风险与体量**：方法论核心单人维护，2.3 万文件级联更新依赖繁琐的版本纪律；对上游学术文献的引用（2026 年多篇 arXiv/Nature）真实性与时效性未独立核验（本次分析未验证）。

## 10. 总结评价

ARS 是当前"AI 辅助学术写作"赛道中工程纪律与自我诚实度最高的项目之一：它没有去做"一键生成论文"的演示型产品，而是把同行评审、完整性核验、反谄媚机制、数据分级、评测金集做成了一套可审计的 prompt 操作系统，并诚实地标注了每一处"这是协议不是保证"。55k 星（8 个月）反映的正是学术界对这类"既要用 AI 又怕 AI 不可靠"工具的真实需求。适合学术作者（尤其社科/教育/信息系统方向，繁中/英双语用户）作为 Claude Code 插件直接使用，也极适合 AI agent 工程师阅读——读它的价值不在某个具体实现，而在"如何为 LLM 行为设计可检验的契约、门禁与边界文档"这一整套范式。想商用集成或期待全自动化产出的人则应绕行：许可证与设计立场都明确把你拒之门外。
