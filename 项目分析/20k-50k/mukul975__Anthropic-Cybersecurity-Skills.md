# mukul975/Anthropic-Cybersecurity-Skills 项目分析

> 星数 36475（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

**面向 AI Agent 的网络安全技能库（内容型仓库）**——818 个符合 agentskills.io 标准的结构化安全技能，映射 MITRE ATT&CK、NIST CSF 2.0、MITRE ATLAS、D3FEND、NIST AI RMF、MITRE F3 六大框架，供 Claude Code、Copilot、Codex CLI、Cursor 等 agent 平台直接加载。**注意：虽然仓库名冠以 "Anthropic"，但这是个人开发者 Mahipal Jangra（GitHub: mukul975）的社区项目，README 明确声明与 Anthropic PBC 无任何隶属关系**。

| 项目 | 值 | 出处 |
|---|---|---|
| 名称/作者 | mukul975/Anthropic-Cybersecurity-Skills，个人项目（Mahipal Jangra） | gh api + CITATION.cff |
| 星数/forks | 36475 / 4463 | gh api（2026-11-21） |
| 语言 | Python（来自 skills 内 1103 个辅助脚本，内容主体为 Markdown） | gh api + tarball 统计 |
| 许可证 | Apache-2.0（顶层 LICENSE + 816 个技能各自 LICENSE） | LICENSE、SCOPE.md |
| 创建时间 | 2026-02-25，半年涨到 36.5k 星 | gh api |
| 最近活跃 | 最后推送 2026-08-31，**已停更约 3 个月**；最后 release v1.3.0（2026-06-22） | gh api commits/releases |
| 技能规模 | 818 个技能（实测），GitHub 描述"817"为过期值 | SKILL.md 计数、index.json |

## 2. 项目是什么，解决什么问题

这是一个**纯内容库**：818 个技能目录，每个目录一个 `SKILL.md`（YAML frontmatter + Markdown 操作手册），外加可选的 `references/`（API 参考、标准映射、工作流深读）、`scripts/`（辅助脚本）、`assets/`（报告模板）。它不实现任何 agent 运行时，做的事情是"把资深安全从业者的操作手册翻译成 AI agent 能发现、能执行的结构化知识"。README 的定位原话是："Existing security tool repos give you wordlists, payloads, or exploit code. None of them give an AI agent the structured decision-making workflow a senior analyst follows"（README.md "Why this exists"）。

每个技能遵循统一四段式结构：`When to Use`（触发条件）→ `Prerequisites`（工具与环境）→ `Workflow`（带真实命令的分步执行）→ `Verification`（结果验证）。README 给出的消费模型是"渐进披露"：agent 先扫全部 frontmatter（每个约 30 token），命中后再加载正文（实测平均每文件 9.9KB，约 2.5-3k token，README 宣称 500-2000 token 偏乐观）。所有 818 个技能的 `domain` 字段统一为 `cybersecurity`，细分靠 `subdomain`（实测 52 个原始取值，经 `tools/validate-skill.py` 的别名表归并为 34 个域，如 `red-team`→`red-teaming`、`security-operations`→`soc-operations`）。

**与 Anthropic 官方关系的客观核实**：README 顶部警告框和页脚各声明一次——"⚠️ Community Project — This is an independent, community-created project. Not affiliated with Anthropic PBC."；CITATION.cff 作者为 "Mahipal"（mukuljangra5@gmail.com，alias mukul975），README 引用条目写全名 "Jangra, Mahipal"，主页域名 mahipal.engineer（作者个人站）。仓库名冠 "Anthropic" 是明显的搜索流量/联想策略，但项目做了透明披露，属于"擦边命名 + 明确免责"的组合。作者同时在 README 推广 GARS-2026 全球调研（SRH Berlin 指导）和 Casky.ai playground，有明显的个人品牌运营成分。

**防御性定位核实**：README 开篇即声明库内含"offensive and dual-use techniques（red-team C2、phishing simulation、exploitation）"，用途限定为授权测试；SCOPE.md "Offensive and dual-use content is in scope" 一节给出理由："A defender's agent that has never seen the offensive procedure detects it badly."。实测进攻面占比很小：目录名以 exploiting-（34）/attacking-（2）/abusing-（2）等攻击动词开头的约 40 个（5%）；subdomain 归属 red-teaming(35)/penetration-testing(21+2) 等明确进攻域的共 58 个（7%）。其余 90%+ 是 detecting（96）、analyzing（76）、hunting（37）、performing（172）、implementing（168）等防御/工程动词。全部 818 个 SKILL.md 中 1008 行出现 "authorized" 字样，进攻类技能描述普遍带"Use during authorized red-team engagements"限定语。本分析只做结构核验，不复述任何攻击程序内容。

## 3. 技术栈与依赖

- **内容主体**：Markdown（2588 个 .md）+ YAML frontmatter，遵循 agentskills.io 开放标准（仓库带 `tools/agentskills-skill.schema.json` schema）。
- **辅助脚本**：1103 个 Python 文件（809 个技能带 `scripts/agent.py`），个别 ps1（2）/yara（1）；GitHub 语言统计因此显示 Python。脚本均为技能配套的本地工具封装，本仓库无统一 Python 依赖清单（CI 只装 `pyyaml`，见 validate-skills.yml）。
- **工具链（tools/）**：自研校验脚本 7 个——`validate-skill.py`（frontmatter 字段/别名校验）、`validate-agentskills.py`（标准符合性）、`generate-index.py`（生成 index.json）、`lint-descriptions.py`、`detect-collisions.py`、`skill_frontmatter.py`（PyYAML 解析器）、`update-contributors.py`。
- **框架映射数据**：纯静态 Markdown/JSON 表（mappings/、docs/），校验时使用官方 `mitreattack-python` 库与 F3 上游 STIX bundle（README 声明）。
- **Claude 插件打包**：`.claude-plugin/plugin.json` + `marketplace.json`（v1.3.0），使仓库可作为 Claude Code 插件市场安装。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `skills/` | 818 个技能目录，扁平结构，不打域子目录；分类只存在于 frontmatter `subdomain` |
| `mappings/` | 框架映射产物：ATT&CK Navigator 层（v4.5 格式/ATT&CK v14）、mitre-attack/coverage-summary.md、nist-csf/csf-alignment.md、owasp/README.md |
| `docs/` | mitre-f3-mapping.md（F3 v1.1 映射 schema，8 tactics、123 techniques） |
| `tools/` | 7 个校验/生成脚本 + schema + lint-baseline.json/collision-allowlist.json 两个"债登记"文件 |
| `.github/workflows/` | 4 个 CI：validate-skills（5 道门禁）、update-index、update-contributors、sync-marketplace-version |
| `.claude-plugin/` | Claude Code 插件市场清单 |
| `index.json` | 生成的全量技能索引（818 条目，generated_at 2026-08-24） |
| `README/SCOPE/CONTRIBUTING/AGENTS.md` | 文档四件套；AGENTS.md 与 .github/copilot-instructions.md 面向"AI agent 协作者" |

技能域分布实测（frontmatter `subdomain`，前 12 名，别名归并前原始值）：cloud-security 66、threat-hunting 58、threat-intelligence 52、network-security 43、web-application-security 42、digital-forensics 41、malware-analysis 39、identity-access-management 37、soc-operations 35、red-teaming 33、container-security 33、ot-ics-security 28。最薄的域恰是 README 自己点名的：purple-team 1、data-protection 1、blockchain/wireless/privacy 各 2——作者在 CONTRIBUTING.md 里公开邀请社区往这些域补内容。

| 技能目录内文件 | 覆盖数（/818） | 说明 |
|---|---|---|
| SKILL.md | 818 | 技能本体，全部合规 |
| LICENSE | 816 | 逐技能许可 |
| references/api-reference.md | 811 | 工具/接口深读 |
| scripts/agent.py | 809 | 模板化辅助脚本 |
| assets/（模板/清单） | 421 | 报告模板等 |
| references/standards.md | 352 | 框架映射详表（带 Rationale） |

组织方式是**扁平单体内容库 + 软件级 CI 门禁**：`skills/` 不嵌套（AGENTS.md 明令"Do not nest skills by domain"，agent 靠扫 `skills/*/SKILL.md` 发现）；工程治理全部押在 tools/ + CI 上，而不是目录层级上。单个技能内部结构高度一致（SKILL.md + LICENSE + references/ + scripts/，SCOPE.md 统计：810 带 api-reference.md、421 带 assets/）。

## 5. 核心架构与关键实现

### 5.1 技能格式契约（skills/*/SKILL.md）

frontmatter 必填 8 字段（`name/description/domain/subdomain/tags/version/author/license`，REQUIRED_FIELDS 定义于 `tools/validate-skill.py`），映射字段为可选列表：实测覆盖率——`mitre_attack` 807/818、`nist_csf` 805、`d3fend_techniques` 139、`nist_ai_rmf` 97、`mitre_f3` 94、`atlas_techniques` 93，与 README 宣称值（805/804/139/97/94/93）基本吻合、个别差 1-2（微漂移）。示例（`skills/performing-memory-forensics-with-volatility3/SKILL.md`）：

```yaml
name: performing-memory-forensics-with-volatility3
domain: cybersecurity
subdomain: digital-forensics
nist_csf: [RS.AN-03, DE.AE-02, RS.MA-01]
mitre_attack: [T1005, T1074, T1119, T1070, T1059]
```

### 5.2 框架映射质量

映射不是贴标签，而是**带理由的结构化表**：有 standards.md 的 352 个技能里，每条 ATT&CK/CSF 映射附 Rationale 列（如 `skills/abusing-dpapi-for-credential-access/references/standards.md` 中 T1555.004 与 DPAPI 机制的对应关系）。F3 映射文档（docs/mitre-f3-mapping.md）解释了为何 F3 与 ATT&CK 分开存 frontmatter："mitre_attack answers how the adversary got in, mitre_f3 answers how it turned into money"，并声明 123 个 F3 v1.1 technique ID 已对上游 STIX 校验。**发现的内部版本漂移**：mappings/README.md 的 Navigator 层仍是 Enterprise ATT&CK v14（218 techniques），而 README 主文宣称 v19.1（290 techniques、含 TA0112 新战术）——映射产物落后于正文声明，停更后无人收敛。

### 5.3 CI 质量门禁（.github/workflows/validate-skills.yml）

内容库罕见地做了"软件级"质量工程，每次对 skills/ 的 push/PR 跑 6 个 gate：frontmatter 校验（单一真源 validate-skill.py）→ agentskills.io 严格符合性 → index.json 新鲜度 → 描述 lint（baseline 棘轮：`lint-baseline.json` 里的存量违规只许降不许升）→ 近重复描述检测（`detect-collisions.py --max-unreviewed 55`）→ 重名检查。最具特色的是**反模式守卫**——CI 里 grep 检测并拒绝手写正则 YAML 解析器，注释记录了事故："that bug shipped 604/817 broken descriptions before it was caught"（三个手写解析器曾把 604 个多行描述截断成一行，AGENTS.md 亦复述此教训）。

### 5.4 面向 agent 的维护面（AGENTS.md）

这是少数把"贡献者"预设为 AI agent 的仓库：AGENTS.md 规定"只有 name/description 在发现期加载""描述里的负面触发词（Do not use for X — use other-skill）必须遵守"、改 SKILL.md 后必须本地跑全部 5 个校验命令。配合 copilot-instructions.md，仓库把自己当成"人 + agent 混合协作"的工程对象来治理。

## 6. 构建与运行

无构建；使用与校验命令均来自 README.md 与 AGENTS.md：

```bash
# 安装（用户侧，任选其一）
npx skills add mukul975/Anthropic-Cybersecurity-Skills   # agentskills.io CLI
git clone https://github.com/mukul975/Anthropic-Cybersecurity-Skills.git

# 作为 Claude Code 插件安装（.claude-plugin/marketplace.json）
# /plugin marketplace add mukul975/Anthropic-Cybersecurity-Skills

# 校验（贡献者/维护侧，AGENTS.md）
pip install pyyaml
python tools/validate-skill.py --all
python tools/validate-agentskills.py --strict
python tools/generate-index.py          # 重新生成 index.json，禁止手改
python tools/lint-descriptions.py --all
python tools/detect-collisions.py
```

**跨平台适配声明的核实**：README "Compatible platforms" 列出 26+ 平台，分四组——AI 代码助手（Claude Code、Copilot、Cursor、Windsurf、Cline、Aider、Continue、Roo Code、Amazon Q、Tabnine、Cody、JetBrains AI）、CLI agent（Codex CLI、Gemini CLI）、自主 agent（Devin、Replit Agent、SWE-agent、OpenHands）、框架/SDK（LangChain、CrewAI、AutoGen、Semantic Kernel、Haystack、Vercel AI SDK、"Any MCP-compatible agent"）。**仓库内没有任何针对单一平台的适配代码**——兼容性完全建立在"SKILL.md 格式与 agentskills.io 标准一致"这一前提上，唯一落地的专属适配是 `.claude-plugin/` 的 Claude Code 插件清单。即"平台兼容"是格式声明而非工程实现，核实为轻微营销夸大但无虚假（frontmatter 确为标准格式）。

## 7. 测试与工程质量

无单元测试框架（tools/ 脚本无 pytest），质量保证完全由 CI 内容门禁承担——这对内容型仓库是合理取舍。工程质量观察：(1) 五道门禁 + 两个棘轮（lint-baseline 只降不升、collision 上限只降不升）体现"存量债冻结、增量零容忍"的思路；(2) index.json 为生成物且 CI 强制新鲜度，杜绝手改漂移；(3) 文档自洽度高，SCOPE.md/CONTRIBUTING.md/AGENTS.md 三者交叉引用、口径一致；(4) 明显的过度声明一处：README 称 ATT&CK 映射"documented in each skill's references/standards.md"，实测仅 352/818 技能有该文件（多数映射只在 frontmatter）。另存在 6 个 SKILL.es.md 西语翻译与 README.fr.md，多语言化刚起步。

综合评价：以"内容仓库的工程标准"衡量属于上游水准；以"安全内容的正确性审计"衡量则无法由 CI 完成——门禁能保证格式与去重，保证不了 818 份手册里每条命令仍然安全有效，这也是内容型项目停更后的典型衰减路径。

## 8. 社区与维护状态

- **节奏**：release 时间线如下（gh api verified），此后约 3 个月零推送。

| Release | 日期 | 要点 |
|---|---|---|
| v1.0.0 | 2026-03-11 | 734 技能 · 26 域 · ATT&CK + NIST CSF 映射 · Navigator 层 |
| v1.1.0 | 2026-03-21 | 753 技能 |
| v1.2.0 | 2026-04-06 | 五框架覆盖（补 ATLAS/D3FEND/AI RMF） |
| v1.3.0 | 2026-06-22 | 六框架（新增 MITRE F3），插件打包版本同步 1.3.0 |

- 最后提交：2026-08-24 一批自动化（index.json/CI/贡献者墙），2026-08-31 "chore: update contributor wall" 为绝笔；index.json 的 generated_at（2026-08-24）与之互证。
- **贡献者**：15 人，头部是维护者本人 + valorisa（18 次）/juliosuas（13 次），其余多为 1-2 次贡献（README contributors 区块）——典型单维护者模式。
- **停更前的自述信号**：SCOPE.md/CONTRIBUTING.md 中作者坦承"review queue is longer than it should be — some pull requests have been waiting months"、"I do that review myself"。单点评审瓶颈在停更前已显性化；目前 77 个 open issues 无人处理（gh api）。
- **社区采纳**：被 VoltAgent/awesome-agent-skills、ottosulin/awesome-ai-security 等 awesome 列表收录（README "Featured in" 表），并进入 SkillsLLM 等 skills 目录站。

## 9. 亮点与风险

**亮点**
1. **规模与一致性兼得**：818 个技能全部通过同一 frontmatter 契约与命名规范（实测 818/818 有合规 frontmatter、name=目录名），在同类社区技能库中少见。
2. **六框架映射实测兑现**：frontmatter 映射覆盖率与 README 宣称一致（807 ATT&CK / 805 CSF / 139 D3FEND / 97 AI RMF / 94 F3 / 93 ATLAS），且映射带 Rationale；抢先覆盖了 2026-04 才发布的 MITRE F3。
3. **内容库当软件工程做**：5 道 CI 门禁、双棘轮、生成式索引、"禁手写 YAML 解析器"守卫及事故复盘注释，是任何"大量 Markdown 条目"仓库可直接套用的治理模板。
4. **agent-native 治理**：AGENTS.md/copilot-instructions.md 把 AI agent 当一等贡献者写明操作规程，与仓库本身的用途（给 agent 读）形成同构。
5. **命名披露诚实**：蹭 "Anthropic" 冠名但两处明示非关联，配合 Apache-2.0 与逐技能 LICENSE，法律与伦理边界写得清楚。

**风险/局限**
1. **已停更约 3 个月**（最后推送 2026-08-31），77 个 open issues 无人回应；单维护者瓶颈自述在案，复活可能性未知——安全内容（工具版本、ATT&CK v19.1 战术重构、F3 后续版本）会随时间失效。
2. **元数据多口径漂移**：GitHub 描述 817 技能/29 域/20+ 平台，README HEAD 818/34/26+，SCOPE.md 817；mappings/README 停在 ATT&CK v14（218 techniques）与主文 v19.1（290）矛盾。
3. **双刃内容风险**：约 40 个技能（exploiting-/attacking-/abusing- 等）含可操作攻击程序，虽有授权使用限定与披露，仍是该库被平台方或企业安全团队审视时的主要顾虑；下游滥用责任完全由使用者承担。
4. **深度参差**：809 个技能共享模板化 scripts/agent.py，README 宣称"every skill encodes real practitioner workflows"难以逐一验证；部分映射声明（standards.md 仅 352/818）与文案不符。

## 10. 总结评价

这是一个把"安全操作手册 + AI agent 可发现性 + 内容工程治理"三者结合得最好的社区技能库：818 个技能撑起 34 个安全域的广度，六框架映射（尤其抢跑 MITRE F3）提供行业对齐，五道 CI 门禁让纯内容仓库获得了软件仓库的工程质量。它与此前分析的官方仓库 anthropics/claude-plugins-official（46.2k 星）构成一组对照——**官方仓库的价值在"分发渠道与供应链治理"（315 条插件目录、SHA 固定、AI 审查流水线），本体代码不足千行；本仓库的价值在"内容资产本身"（818 份可执行安全手册），靠个人作者的领域写作能力与命名策略半年博得 36.5k 星**；前者是 Anthropic 官方组织账号，后者是冠名"Anthropic"但反复声明非关联的个人项目——两者代表了 Claude skills 生态星力的两种来源：入口税与内容稀缺性。当前最大问题是 2026-08-31 起停更、单维护者模式已力不从心。适合三类人使用：想给安全 agent 塞领域知识的工程团队（直接消费 skills/ 与 index.json）、想建大规模内容库的维护者（抄 tools/ + CI 门禁 + SCOPE.md 治理文书）、研究 agent 技能标准的人（agentskills.io 落地样本）。不建议在未审计的情况下盲目加载全部技能——尤其进攻类条目应按 SCOPE.md 的授权前提逐条评估。
