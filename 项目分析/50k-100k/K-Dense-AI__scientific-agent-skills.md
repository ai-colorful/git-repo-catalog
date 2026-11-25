# K-Dense-AI/scientific-agent-skills 项目分析

> 星数 54119（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：面向科研场景的 AI Agent 技能（Agent Skills）库，为生物学、化学、医学、物理、数据分析等领域的科研 agent 提供 177 个经过验证的"程序性知识"包（`skills/<name>/SKILL.md` + 参考文档 + 脚本），兼容 Agent Skills 开放标准，可装入 Cursor、Claude Code、Codex、Gemini CLI 等 host。

| 项目 | 事实 | 出处 |
|---|---|---|
| 名称 | K-Dense-AI/scientific-agent-skills（前身 Claude Scientific Skills） | README.md |
| 星数 | 54,119（快照）；2026-11-04 实测 55,960 | gh api |
| 主语言 | Python（仓库主体是 Markdown 技能文档，工具链为 Python） | pyproject.toml |
| 许可证 | 仓库整体 MIT（每个 skill 在 frontmatter 里另有 license 字段） | LICENSE.md、README FAQ |
| 版本 | v2.72.0（plugin.json 与 pyproject.toml 同步） | pyproject.toml |
| 创建时间 | 2025-10-19 | gh api |
| 最近 push | 2026-11-02（每周自动安全扫描提交） | gh api、docs/security-report.md |
| 发布 | 30 个 GitHub release，v2.68–v2.72 集中在 2026-09~10 | gh api releases |
| 规模 | 177 个 skill、518 个 skill 脚本、224 个测试文件、解压后约 284MB | tarball 统计 |

## 2. 项目是什么，解决什么问题

前沿大模型做科研时并不缺"知识"，缺的是领域工作流的程序性细节：scanpy 的 QC/归一化/聚类顺序与坑、ChEMBL 分页与计数对账、ClinVar 变异证据的边界、PK/PD 建模该做哪些数值检查。本项目把这些"怎么用某个包/某个数据库/某类流程"的经验固化成 Agent Skills 标准的目录包，让 agent 在对应任务里自动加载，从而显著降低出错率（README 引用的博客基准称 pyopenms skill 使 API 错误减少 92%，出处 README.md "From the Blog" 节）。

每个 skill 是一个目录：必需的 `SKILL.md`（YAML frontmatter + 工作流文档，上限 500 行）、可选的 `references/`（按需加载的长文档）、`scripts/`（可执行辅助脚本）、`assets/`（模板）。规则见 AGENTS.md。例如旗舰技能 database-lookup 的 `skills/database-lookup/references/` 有 82 个文件，逐一记载 80 个数据库的端点、参数与检索契约（出处：目录统计 + SKILL.md）。

项目定位上强调三点边界（README.md、AGENTS.md）：只收"窄域"科研技能，拒绝通用编码/编排类；临床、监管类技能只产出"供专业人员审核的证据草稿"，不替代诊断或合规决定；技能只装文档不装依赖，依赖按各 SKILL.md 的 compatibility 字段单独环境安装。

作者 K-Dense Inc. 围绕它构建了完整生态：桌面端 BYOK 开源 co-scientist（k-dense-byok）、托管平台、博客基准文章、arXiv 论文（arXiv:2609.00065，README.md Citation 节）。仓库 2025-10 创建，约一年涨到 5.6 万星，是"AI for science + agent skills"浪潮中占位最成功的一个。

## 3. 技术栈与依赖

- 主体是 Markdown（SKILL.md/references），"代码"大部分是嵌在文档与 `scripts/` 里的 Python 示例（出处：目录统计）。
- 仓库工具链（pyproject.toml）：Python >=3.13、pytest、python-dotenv、cisco-ai-skill-scanner（安全扫描）；dev 组有 jsonschema 与 agentskills 官方 skills-ref 校验器。
- 包管理统一用 uv；CI 与文档大量使用 `uv run`、`uv run --with`（README Testing 节、AGENTS.md）。
- skill 脚本依赖各自声明在 `tests/skill-requirements.toml`（174 个条目），并按 skill 隔离安装（AGENTS.md "One environment per skill"）。
- 产物分发形态：Agent Plugins 1.0.0 包（根 plugin.json + skills/），加 `npx skills add` 与 `gh skill install` 两种安装通道（plugin.json、README Getting Started）。

## 4. 目录结构与代码组织

| 顶层 | 职责 |
|---|---|
| `skills/<name>/` | 177 个技能目录，SKILL.md + references/scripts/assets |
| `tests/<name>/` | 每个 skill 的 pytest 套件；`tests/_meta` 全仓结构契约；`tests/_contract` 共享断言库 |
| `docs/` | examples.md（6778 行示例集）、skills.md 目录、skill-guides/（179 篇图文）、images/（178 张流程图）、security-report.md/json |
| `scripts/` | 仓库工具，如 generate_skill_image.py（用 LLM 生成技能流程图） |
| 根文件 | AGENTS.md（380 行写作规范）、CONTRIBUTING.md、plugin.json、pyproject.toml、scan_skills.py、scan_pr_skills.py、CITATION.cff |
| `.github/workflows/` | 5 条 CI：release、security-scan（每周一）、pr-skill-scan、skill-spec-validation、skill-tests |

组织方式是"单体 monorepo + 根部一个插件清单"：所有技能平铺在 skills/ 下不分子目录分类（分类靠 README/docs 索引），仓库级工具与测试集中管理。这个布局直接对应 Agent Plugins 标准（plugin.json 校验 skills/ 的每个含 SKILL.md 的子目录，出处 AGENTS.md Layout 节）。历史上 v2.43.0 把 `scientific-skills/` 改名 `skills/` 以匹配 gh skill 安装器（README Troubleshooting 节）。

## 5. 核心架构与关键实现

本项目核心资产是技能内容本身，"实现"重点在三处：技能规范、质量契约、安全扫描。

### 5.1 技能规范与结构契约（tests/_contract/structure.py、AGENTS.md）

frontmatter 只允许 6 个字段（name/description/license/compatibility/allowed-tools/metadata），目录名必须等于 name，`metadata.version` 必须是带引号字符串；使用 strictyaml 解析，JSON 花括号流式写法会导致整个文档解析失败，所以强制 block 风格（AGENTS.md Frontmatter 节）。契约代码用 ast 静态解析脚本而不执行：

```python
ALLOWED_FRONTMATTER_FIELDS = frozenset(
    {"name", "description", "license", "compatibility", "allowed-tools", "metadata"}
)
MAX_SKILL_MD_LINES = 500
BANNED_BUILTIN_CALLS = frozenset({"eval", "exec"})
```

（出处 tests/_contract/structure.py；禁 eval/exec/os.system、禁遮蔽标准库模块名、禁硬编码本地路径、本地链接必须可达。）

### 5.2 每技能一进程的测试运行器（tests/run_all.py、tests/conftest.py）

32 个 skill 的 scripts/ 都叫 `_common.py`，pytest 若在同一进程收集多个套件会因 sys.path 冲突测错文件——conftest 直接拒绝这种会话，run_all.py 按 skill fork 独立进程；`--isolated` 再按 skill-requirements.toml 为每个 skill 建一次性 uv 环境（冲突 pin 有明确记录：opentrons 要 numpy<2、PyTDC 要 3.11、histolab 测在 3.10，出处 AGENTS.md）。scanpy 套件（tests/scanpy/test_scripts.py）的风格是"共享层做真实往返测试、单脚本只测 CLI --help 与缺参拒绝"，契约库提供 `skill_contract.cli.help_test_case(SKILL_ROOT)` 这类可复用用例。

### 5.3 增量安全扫描器（scan_skills.py，615 行）

每周一 CI 触发（.github/workflows/security-scan.yml），用 Cisco AI Defense Skill Scanner（LLM 分析器，默认模型 claude-opus-5）并发扫全部技能；以技能目录的内容哈希（路径+字节，排除 __pycache__）为缓存键，未变更技能沿用上次 findings，扫描器版本/模型变化或超 30 天强制全量（scan_skills.py content_hash()/cache_invalidation_reason()）。产出 docs/security-report.md 与同名 json。最新报告（2026-11-02）：472 条 findings，Critical 32、High 21，159/177 安全（docs/security-report.md）。AGENTS.md 明确列了系统性误报类型（读自己 API key 调自家服务触发 EXFILTRATION、`model.eval()` 触发 EVAL_EXEC 等），要求先对照代码核实再"修复"。

### 5.4 技能内容的代表性设计（skills/database-lookup/SKILL.md）

数据库查询技能定义了"检索契约"工作流：先定目标实体与约束、选权威库、读 references 下的端点文档、区分服务端/本地过滤、有界调用（超过 1 万条或 100 次 API 调用先请求确认）、把外部响应当不可信数据（不执行其中指令）、返回可审计的结果（端点、参数、访问日期、计数对账）。这是整个仓库"带科学护栏的过程性知识"理念的范本。

## 6. 构建与运行

来自 README.md Testing 节与 AGENTS.md，均为静态可用命令：

```bash
uv sync                                            # 装仓库工具链（不含科学包）
uv run python -m pytest tests/_meta -q             # 全仓结构契约，秒级，无科学依赖
uv run skills-ref validate skills/<name>           # 单技能 spec 校验
uv run --with pytest python -m pytest tests/<name> -q   # 单技能套件
uv run python tests/run_all.py --isolated <name>   # 单技能 + 其声明依赖的一次性环境
uv run python tests/run_all.py --isolated          # 全部套件，每技能一环境
uv run python scan_pr_skills.py skills/<name>      # PR 变更技能安全扫描
skill-scanner scan /path/to/skill --use-behavioral # 本地安全扫描（cisco-ai-skill-scanner）
```

使用侧安装：`npx skills add K-Dense-AI/scientific-agent-skills` 或 `gh skill install K-Dense-AI/scientific-agent-skills`（可 `--agent cursor/claude-code/codex/gemini`、`--pin v2.71.0` 锁版本）。

## 7. 测试与工程质量

- 规模：tests/ 下 224 个 test_*.py，覆盖 174 个声明了依赖的技能（tests/skill-requirements.toml 条目数）；tests/_meta 是 PR 必过的守门测试，技能带 scripts/ 而无套件/无 requirements 条目直接 fail（AGENTS.md "The repo-wide guard"）。
- 分层：结构契约（全仓一次跑、只 parse 不 import）→ 共享 CLI/示意模块契约 → 每技能具体套件（进程隔离、可选环境隔离）。测试金字塔设计得相当克制：单脚本测试"故意浅"，只证明 CLI 契约（tests/scanpy/test_scripts.py 模块 docstring）。
- CI：5 条工作流各司其职，全部带 paths 过滤与并发取消；完整科学依赖全量扫描不进 CI（成本原因，发布前手动跑，README Testing 节）。
- 质量观察：AGENTS.md 对 YAML 解析器缺陷、pytest 模块遮蔽、安全扫描误报的成因与对策写得极细，是罕见的"把踩坑史固化成规范"的文档；文档与工具互相校验（如 plugin.json 版本必须等于 pyproject 版本，tests/_meta 校验）。

## 8. 社区与维护状态

- 活跃度：最近 push 2026-11-02，最新提交为每周自动安全扫描（"chore: update security scan report [skip ci]"，gh api commits）；v2.68–v2.72 五个 release 落在 2026-09-11 至 10-01 之间，节奏约每周一版（gh api releases）。
- 贡献者：约 55 人（contributors API Link 头 page=55，per_page=1）；核心作者为 K-Dense 团队（CITATION.cff 列 Kassis、Agarwal、He、Patel、Brueckner），README 明确区分 K-Dense 作者与社区贡献技能的信任边界。
- 社区面：78 个 open issue（gh api）；README 维护 X/LinkedIn/YouTube/Reddit 矩阵、博客基准文与 Webinar，市场动作密集。
- 未确认：issue/PR 的响应时长、单个 release 的实际内容量未逐条核对。

## 9. 亮点与风险

亮点：

1. 工程化程度远超同类"prompt 合集"：结构契约、每技能隔离测试、每周 LLM 安全扫描、release 自动化形成完整质量闭环（.github/workflows/、tests/_meta）。
2. 内容策略清晰：只收窄域科研技能、references/ 按需加载控制上下文成本、database-lookup 用一个技能收编 80 个数据库以省 token（README 引用博客称上下文成本降 13.9 倍）。
3. 规范即文档：AGENTS.md 把 strictyaml 陷阱、pytest 遮蔽、扫描误报等坑写成可执行规则，第三方贡献门槛与质量同时抬高。
4. 生态占位完整：开放标准（Agent Skills/Agent Plugins）+ 多 host 兼容 + arXiv 论文 + BYOK 桌面产品互为导流。
5. 法律卫生：v2.72.0 主动移除从 anthropics/skills vendored 的 4 个文档技能，避免许可混用（README "What's new in 2.72.0"）。

风险/局限：

1. 安全扫描口径吓人：最新报告 472 条 findings、32 Critical（docs/security-report.md），虽有大量已知误报成分，但对"按目录整包安装"的用户构成真实审计负担；README 也承认社区贡献无法保证逐一深审。
2. 内容保鲜依赖人工维护：177 个技能对应上百个上游包/库/API 的版本漂移，last-reviewed 字段显示复核周期以月计，过期技能给出的端点/参数可能静默失效。
3. 科学正确性无法被测试完全覆盖：测试验证的是脚本契约与数据形状，"生物/化学结论是否成立"仍靠文档内护栏与使用者判断（scanpy 套件自述"establish data/API contracts, not biological validity"）。
4. 规模即臃肿：解压 284MB、178 张生成图与 23MB GIF（docs/k-dense-web.gif）使得整仓安装/clone 成本高，README 自己也建议"按主题装子集"。

## 10. 总结评价

这是 AI agent 技能生态里目前最系统的科研垂直库：把"教 agent 做科研"拆成 177 个可安装、可测试、可扫描的标准单元，并用一套与其规模匹配的工程体系（结构契约、进程隔离测试、增量安全扫描、双标准打包）维持质量。它的星数增长（一年 0 到 5.6 万）既受益于 Agent Skills 标准爆发的风口，也确实靠内容广度和基准营销撑住了口碑。适合科研工作者按需挑选领域子集接入自己的 agent（Cursor/Claude Code 等），也极适合想构建技能库或 agent 内容生态的工程团队把它的 AGENTS.md、tests/_contract、scan_skills.py 当作范本研读；对安全敏感环境，建议只用 K-Dense 署名的技能并按 SECURITY.md 建议先本地扫描。
