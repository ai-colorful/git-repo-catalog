# bmad-code-org/BMAD-METHOD 项目分析

> 星数 54,932（快照 2026-10-24｜复核 2026-11-04）｜主语言 Python｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：BMad Method（Breakthrough Method for Agile Ai Driven Development）是一个"敏捷 AI 驱动开发"方法框架——用一套可安装到任意 AI 编码工具的提示词技能（skills）体系，把产品构思、规划、架构、实现、评审、回顾组织成结构化的多 agent 协作流程。

| 事实 | 值 | 出处 |
| --- | --- | --- |
| 名称 | bmad-code-org/BMAD-METHOD | GitHub API |
| 星数 | 54,932（任务快照 54,479） | GitHub API 2026-11-04 |
| Fork | 6,135；Open Issues 98 | GitHub API |
| 创建时间 | 2025-04-13（约 19 个月涨到 5.4 万星） | GitHub API |
| 最近推送 | 2026-11-03（分析前一天） | GitHub API |
| 默认分支 | main（当前为 V7 preview alpha） | GitHub API、README.md |
| 许可证 | MIT（LICENSE，BMad Code, LLC 版权） | LICENSE |
| 语言形态 | 提示词文档为主（358 个 .md）+ Python 工具（96 个 .py） | 源码统计 |
| 最新 release | v6.12.4（2026-10-14），稳定线在 V6.12 分支 | GitHub API releases |

## 2. 项目是什么，解决什么问题

AI 编码助手擅长实现，但常把"没说出口的假设"直接变成代码。BMad 的解法是把敏捷软件交付的全流程做成结构化提示词工作流：该建什么（产品）、怎么组织（架构）、怎么变化（迭代），每一步的决策都被显式化并作为上下文传递下去（README.md "Why BMad" 一节）。它不替代人的判断，而是用专门的 agent 视角（产品、架构、UX、开发、测试）和引导式工作流，让"过程规模与工作规模匹配"：小改动直接进 Build，大计划走完整规划路径（README.md；docs/plan/choose-a-planning-path.md）。

需要特别指出：这不是传统意义的"代码库"——仓库树本身就是分发包。`pyproject.toml` 开头写明"this is not a distributable package: the repository tree itself is what `npx skills add bmad-code-org/BMAD-METHOD` and the plugin marketplaces install"。安装方式是 `npx skills add bmad-code-org/BMAD-METHOD --skill '*'`，或通过 Claude Code / Codex 的插件市场（README.md Start Building）。安装后技能落入用户工具的技能目录（Claude Code 为 `.claude/skills/`，Cursor/Codex 等为 `.agents/skills/`，见 docs/reference/skills-and-agents.md）。

仓库当前 HEAD 是 **V7 preview alpha**（发布 ETA 2026 年 11 月中，README.md），稳定版 v6.12 维护在单独的 V6.12 分支、走旧的 npm 安装器（`npx bmad-method install`）。V7 的核心变化：从"npm 安装器复制 agent 定义文件"转为"原生 skills + uv 运行 Python 辅助脚本 + Jinja2 渲染工作流快照"。

体系分三块（docs/reference/skills-and-agents.md）：
- **六个具名 agent**：Mary 分析师、John 产品经理、Sofia 产品负责人、Winston 架构师、Amelia 开发者、Sally UX 设计师（roster 见 skills/bmod-method/roster.toml）；
- **核心技能**：`bmad` 枢纽（问答/安装/迁移/倡议管理）、`bmad-build`（端到端实现）、`bmad-review`、`bmad-code-review`、`bmad-retrospective`、`bmad-prd`、`bmad-architecture` 等 33 个技能目录；
- **生态模块**：Test Architect、Creative Intelligence Suite、Loop（无人值守跑整个 epic）、Game Dev Studio、Builder（README.md BMad Ecosystem 表）。

## 3. 技术栈与依赖

- **主体是提示词工程**：358 个 Markdown 文件承载技能、工作流、步骤、参考文档；86 个 TOML 承载模块清单与定制值（源码统计）。
- **Python 3.11+（uv 管理）**：所有随附脚本用 PEP 723 内联声明依赖（如 `skills/bmad/scripts/render_skill.py` 头部 `# dependencies = ["jinja2>=3.1"]`），`uv run` 直接执行，无需安装包。核心依赖只有 jinja2（渲染）、pyyaml/ruamel.yaml（配置工具，dev 组）。
- **工程工具链**（pyproject.toml）：pytest + pytest-xdist（`-n auto` 并行）、ruff、pre-commit、rumdl（Markdown lint）、yamlfix；ruff target py311、行宽 120。
- **文档站**：Astro + Starlight（docs-site/package.json，`@astrojs/starlight ^0.42.1`），构建脚本带链接校验、侧边栏顺序校验（docs-site/scripts/）。
- **运行时依赖倒挂设计**：被安装进用户项目的脚本不依赖本仓库 pyproject，各自内联依赖声明，`tools/tests/test_python_floor.py` 强制所有脚本声明同一 Python 下限（pyproject.toml 注释）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `skills/` | 33 个技能目录，每个含 SKILL.md、bmod.toml、customize.toml，多数含 workflow.md、step-*.md、references/、scripts/、evals/ |
| `skills/bmad/` | 枢纽技能：setup/migrate/initiative + `scripts/render_skill.py` 等运行时核心 |
| `skills/bmod-method/`、`bmod-core-tools/`、`bmod-toolsmith/` | 模块记录（module record）：bmod.toml、roster.toml、retired.toml、migration-*.toml |
| `tools/` | 仓库自身开发工具：validate_skills.py、validate_file_refs.py、dogfood.py、quality.py、tests/（7 个测试文件） |
| `docs/` | 文档站内容源（Markdown，54 个文件，按 start/plan/build/customize/existing-codebases/toolsmith/reference 分章） |
| `docs-site/` | Astro + Starlight 文档站工程（含 .nvmrc、测试、链接校验脚本） |
| `.github/` | CI（quality/docs/discord/coderabbit-review/retarget-pr 五个 workflow）、issue 模板 |

组织方式评述：单体仓库即产品。没有 src/、没有 npm 包体——技能源文件直接放在仓库里，安装动作就是"把目录复制/链接到用户工具的技能目录"。模块化靠 `bmod.toml` 清单（`[skill] bmod = "bmod-method"` + `source`，如 skills/bmad-build/bmod.toml）声明归属；下线技能由各模块的 `retired.toml` 登记并由 setup 清理（CHANGELOG Unreleased 节）。仓库自身开发用 `AGENTS.md` 约束（Conventional Commits、PR 走 dev 分支、main 只按 tools/release.md 发布）。

## 5. 核心架构与关键实现

### 5.1 技能入口与"渲染快照"机制

每个 SKILL.md 带 frontmatter（name + description 触发条件），正文是给模型的指令。核心技能入口不直接跑工作流，而是先渲染出一个不可变快照——`skills/bmad-build/SKILL.md` 第 11-19 行：

```bash
uv run --no-cache "{project-root}/_bmad/scripts/render_skill.py" \
  --project-root "{project-root}" --skill "{skill-root}"
```

`render_skill.py`（skills/bmad/scripts/render_skill.py，约 500 行）用 Jinja2 把技能源 + 三层定制值合成最终工作流文本：校验参数（`--set workflow.route=oneshot` 必须命中默认值中已声明的路径，`--set` 重复/冲突直接报错）、做 TOML 结构合并、sha256 指纹，任何不安全情况抛 `RenderError` 并输出 `HALT: <reason>` 让 agent 停下。这解决了提示词框架最大的痛点：**运行时可变配置不靠模型"记住"，而是渲染期确定性注入**。

定制合并三层：技能自带的 `customize.toml`（默认，更新时覆盖）→ 团队 `_bmad/custom/<skill>.toml` → 个人 `<skill>.user.toml`；规则是标量替换、表深合并、按 id/code 键的表数组替换同名项、其余数组追加（skills/bmad-agent-pm/SKILL.md 第 27-33 行明文规定，回退逻辑与 resolver 一致）。

### 5.2 bmad-build：步文件工作流编排

`skills/bmad-build/workflow.md` 是编排主干：路由参数 `route`（oneshot 本会话直接实现 / full 写计划后交给子代理 / auto 按情况选）、`review`（none/quick/thorough/auto），Jinja2 条件在渲染期裁剪分支。执行纪律写得很硬（workflow.md 第 57-80 行）："NEVER load multiple step files simultaneously"、"ALWAYS halt at checkpoints and wait for human input"。步骤文件链：step-01-clarify-and-route（意图/工单树/已有计划三路解析，接 `tickets.py find/next`）→ step-02-plan → step-03-implement → step-04-review → step-05-present，另有 step-oneshot 捷径。

两条量化标准值得注意（workflow.md）：计划"Ready for Development"六条（可执行/有序/可测/无占位/无缺口/无矛盾，AC 一律 Given/When/Then）；计划体量"单用户目标、900–1600 tokens"——下限防歧义、上限防实现 agent 上下文腐烂，且"两条都不是门禁，可被用户覆盖"。评审配 prompt 库 review-prompts/edge-case-hunter.md、verification-gap.md 与 references/claims-check.md、deletion-check.md。

### 5.3 Agent 角色体系

六个 agent 技能结构同构：激活八步（解析 agent 定制块 → 前置步骤 → 入戏 persona → 载入 persistent_facts → 读全局 config → 问候 → 后置步骤 → 菜单分发），见 skills/bmad-agent-pm/SKILL.md。persona 本体可被团队/个人定制层改写（role/identity/communication_style/principles 四键），图标前缀保证会话中角色可辨识。技能描述即触发器，如 bmad-agent-pm 的 description "Use when the user asks to talk to John or requests the product manager"。

`skills/bmod-method/roster.toml` 为 party-mode（多 agent 讨论）登记成员：Mary"像寻宝者一样说话"、John"像侦探审问悬案"——每个 persona 一句话浓缩方法论立场（John 主张 JTBD 优先于填模板，Sofia"早早说不并给出理由"）。这是提示词层面的"角色即观点"设计。

### 5.4 安装器与自举（setup.py）

`skills/bmad/scripts/setup.py` 是运行时安装/体检核心：扫描各技能目录的 `bmod.toml` 清单、处理 retired.toml 清理、读写 `_bmad/config.toml`（团队）与 `_bmad/custom/config.user.toml`（个人，gitignore）、支持 `github:/https:/file:/plugin:` 四类更新源、校验 semver。它一个流程同时完成"安装 + 更新 + 迁移"（CHANGELOG Unreleased 节）。v6 → v7 迁移由 `migrate_v6.py` 承担（含工单树布局迁移、custom 键改名）。

### 5.5 技能评测（少见的设计）

六个技能自带 `evals/`：`triggers.json` 是触发/不触发正反例（如 bmad-shard-pro 的"400 页 PRD 卡住 agent，能切吗"→ should_trigger true；"6 小时会议记录转写"→ false），`cases.json` 定义端到端用例——输入、挂载 fixture（GitHub URL 固定 commit 针脚）、逐条 rubric（skills/bmad-shard-pro/evals/cases.json）。配套 `bmad-eval` 技能提供 baseline/variant/quality/trigger 四种跑法与聚合基准（skills/bmad-eval/scripts/run_evals.py、aggregate_benchmark.py）。给"提示词技能"建回归评测体系，是该仓库工程质量上最有辨识度的一点。

## 6. 构建与运行

安装（用户视角，README.md）：

```bash
# V7 preview：装到支持 skills 的 AI 编码工具
npx skills add bmad-code-org/BMAD-METHOD --skill '*'
# 或插件市场：/plugin marketplace add bmad-code-org/bmad-plugins（Claude Code）
# 稳定版 v6.12（V6.12 分支，npm 安装器）：
npx bmad-method install
```

仓库开发（AGENTS.md、pyproject.toml、.github/workflows/quality.yaml）：

```bash
uv sync --frozen                                  # 装开发依赖
uv run tools/validate_skills.py --strict          # 校验全部技能
uv run --frozen python -B -m pytest               # 全量测试（tools/tests + skills）
uv run --frozen pre-commit run --all-files        # lint + 校验（CI 同款）
cd docs-site && npm ci && npm run build && npm test  # 文档站构建与测试
```

## 7. 测试与工程质量

- **测试规模**：45 个 test_*.py、约 1,071 个测试函数，pytest + xdist 并行（pyproject.toml `addopts = "-q -ra --tb=short -n auto"`）。覆盖 bmad/scripts（setup/roster/render/migrate 等各有对应测试）、toolsmith 全部脚本、eval runner 等（源码统计）。
- **测试哲学有明文边界**：AGENTS.md 规定"自动化测试只断言确定性代码的产物，不为 LLM 输出或静态源码写自动化测试"——提示词部分交给 evals 体系而非单测，界限清晰。
- **CI**（.github/workflows/quality.yaml）：Python 矩阵跑 ubuntu + windows 两平台（fail-fast 关闭，各报各的），核心一步是 pre-commit 全文件运行；文档站独立 job 做 ESLint、Prettier、构建内链接校验、侧边栏校验与 4 个 node 测试脚本。
- **质量设施**：ruff（E/F/I/UP/B 规则集）、rumdl Markdown lint、yamllint/yamlfix、.pre-commit-config.yaml、CodeRabbit 评审（.coderabbit.yaml、coderabbit-review workflow）。一个有趣的细节：`tools/tests/test_python_floor.py` 用测试强制所有分发脚本的 Python 下限一致，把"约定"变成了"回归"。

## 8. 社区与维护状态

- **活跃度极高**：创建于 2025-04-13，最近推送 2026-11-03（分析前一日）；v6.12 线 2026-09-03 → 10-14 连发 5 个 patch release，随后全力投入 V7 alpha（GitHub API releases、CHANGELOG.md）。
- **CHANGELOG.md 达 162KB**，每条目是叙述性段落而非流水行，Unreleased 节包含大量设计决策说明（如 elicitation 重建的盲测评审过程），维护投入罕见地重。
- **贡献者**：GitHub API contributors 分页约 164 人；CONTRIBUTORS.md 仅 1.3KB（列核心贡献者）。Issue/PR 模板、行为准则、SECURITY.md、TRADEMARK.md 齐备（.github/）。
- **社区渠道**：Discord、YouTube（@BMadCode）、Buy Me a Coffee 赞助，主打"无付费墙、无封闭社区"（README.md）。
- **治理特征**：仓库由 BMad Code, LLC 商标持有，AGENTS.md 明确 main 分支只按 tools/release.md 发布、PR 走 dev 分支——典型的单一维护方 + 社区贡献结构。

## 9. 亮点与风险

**亮点**
1. 提示词工程的工程化范式：渲染期确定性注入配置 + 三层定制合并 + `HALT` 失败语义，把"模型自觉"换成"机制保证"（skills/bmad/scripts/render_skill.py）。
2. 步文件工作流编排：微文件、按序加载、检查点强制停等，配 900–1600 token 计划体量标准，直接针对 LLM 上下文腐烂设计（skills/bmad-build/workflow.md）。
3. 提示词技能的评测体系：triggers.json 正反例 + rubric 化端到端 cases，可本地复跑（skills/*/evals/）。
4. 仓库即分发包：bmod.toml 清单 + retired.toml 退役登记 + setup 一个流程完成装/更/迁，安装面覆盖 7+ 种 AI 工具（docs/reference/skills-and-agents.md）。
5. 方法论内嵌：具名 agent 各有一句话方法论立场，ego 消融成"角色即观点"，便于团队定制层覆盖（skills/bmod-method/roster.toml）。

**风险/局限**
1. V7 preview 变动剧烈：README 自述"expect breaking changes"， translated 文档已删除、多键改名（CHANGELOG Breaking 节），生产采用应留 V6.12 分支。
2. 提示词即代码的可维护性风险：358 个 md 是"每次运行都全文付费"的上下文，AGENTS.md 已意识到并要求精简，但规模仍在膨胀（33 个技能 + 生态模块）。
3. 强依赖宿主工具能力：需要支持 skills 的 AI 编码工具与 uv/Node 环境；渲染机制假定 `_bmad/` 目录可写，沙箱化环境适配未验证（README.md、SKILL.md）。
4. 文档主站与仓库同步成本高：docs/ 与 docs-site/ 双轨 + 链接校验，多语言已放弃，非英文用户门槛上升（CHANGELOG Breaking 节）。

## 10. 总结评价

BMAD-METHOD 是"AI 时代敏捷交付方法论"目前最完整的开源实现：它把 SCRUM/敏捷的角色与制品翻译成可安装、可定制、可评测的提示词技能体系，并用渲染器、三层定制合并、步文件编排和 eval 框架补上了提示词工程普遍缺失的确定性底座。5.4 万星（19 个月内）源于踩中了"AI 编码缺流程"的真空，而留存价值在于其工程纪律——测试边界、发布治理、变更日志都是同龄项目的上乘水准。适合：想给团队引入结构化 AI 交付流程的实践者直接安装使用；想做 agent 编排/提示词框架的开发者精读 `render_skill.py`、`bmad-build/workflow.md` 与 evals 体系三处——这是本仓库最有复用价值的设计。不宜期待的是传统意义源码阅读体验：绝大部分"代码"是精心打磨的英文提示词文档。
