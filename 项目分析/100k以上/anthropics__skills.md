# anthropics/skills 项目分析

> 星数 183206（快照 2026-10-24）｜主语言 Python｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

Anthropic 官方 Agent Skills 仓库：以「SKILL.md + 辅助脚本/资源」的形式，教 Claude 完成特定任务的可复用技能集合，同时是 Agent Skills 开放规范（agentskills.io）的官方参考实现。

| 项目 | 值 |
|---|---|
| 名称 | anthropics/skills |
| 星数 | 183,206（快照）/ 184,755（分析日 gh api 实测） |
| 主语言 | Python（72 个 .py 共约 17,960 行，另有 138 个 .md） |
| 许可证 | 仓库级无统一 License：示例技能 Apache 2.0，文档技能（docx/pdf/pptx/xlsx）为专有 source-available（各技能目录 LICENSE.txt，`skills/docx/LICENSE.txt`） |
| 创建时间 | 2025-09-22（`gh api repos/anthropics/skills`） |
| 最近 push | 2026-10-28（分析前一天，高度活跃） |
| 贡献者 | 16 人（gh api contributors）；无 GitHub Releases |

## 2. 项目是什么，解决什么问题

Skill 是「文件夹形式的知识包」：一个 `SKILL.md`（YAML frontmatter 元数据 + Markdown 指令）加上可选的 `scripts/`（可执行脚本）、`references/`（按需加载的参考文档）、`assets/`（模板、字体等资源）。Claude 运行时只常驻每个技能的元数据（name + description，约 100 词），触发时才加载 SKILL.md 正文，需要时再读捆绑资源——即官方所称的三级「渐进式披露」（`skills/skill-creator/SKILL.md`），解决的是通用模型做专业任务时缺乏领域流程与工具的问题。

仓库包含 20 个技能（`skills/` 下 20 个目录，均含 SKILL.md），分两类：一是驱动 Claude.ai「文档能力」的生产级文档技能 docx/pdf/pptx/xlsx（README 明言 source-available，供开发者参考生产级技能写法）；二是展示技能模式的示例技能，覆盖创意设计（canvas-design、algorithmic-art、theme-factory）、开发工具（mcp-builder、webapp-testing、web-artifacts-builder）、企业沟通（internal-comms、brand-guidelines）与元技能 skill-creator。

另一条主线是标准化：`spec/agent-skills-spec.md` 只有一句话指向 https://agentskills.io/specification ，说明规范已独立为跨厂商开放标准；仓库自身通过 `.claude-plugin/marketplace.json` 注册为 Claude Code 插件市场（README 提供 `/plugin marketplace add anthropics/skills` 用法），用户也可在 Claude.ai 上传技能或经 Skills API 调用。

从工程视角看，本仓库回答了一个新问题：**如何把提示词工程交付成可测试、可分发、可版本化的「软件制品」**。skill-creator 技能给出了完整方法论：起草 → 评测（run_eval.py 测触发准确率）→ 量化基准 → 迭代 → 打包（`skills/skill-creator/scripts/`），把 prompt 当代码维护。

需要澄清「183k 星」的构成：这是 Anthropic 官方仓库 + Skills 概念首发地 + Agent Skills 标准的 reference 实现，星数更多反映生态关注度而非代码规模——全仓库仅 446 个文件、约 5.4MB。

## 3. 技术栈与依赖

- **载体**：纯 Markdown + YAML frontmatter（138 个 .md），无构建系统、无 package.json/pyproject.toml（顶层无工程清单文件）。
- **Python**：72 个脚本（约 1.8 万行），核心依赖为标准库 + `defusedxml`（安全解析 XML，`skills/docx/scripts/merge_runs.py`）；mcp-builder 技能声明 `anthropic>=0.39.0`、`mcp>=1.1.0`（`skills/mcp-builder/scripts/requirements.txt`）。
- **外部工具链**（由 SKILL.md 指导模型调用，非仓库自带）：docx-js（npm，生成 .docx）、pandoc（读取）、LibreOffice/soffice（格式转换与渲染验证）、pdftoppm/Poppler（转图检查）、Playwright（webapp-testing）。
- **数据资源**：117 个 .xsd——完整的 ISO/IEC 29500-4:2016、ECMA OPC、Microsoft 扩展 Office XML Schema（`skills/docx/scripts/office/schemas/`）；54 个 .ttf 开源字体（SIL OFL，`skills/canvas-design/canvas-fonts/`）。
- **运行环境假设**：技能在 Claude Code/Claude.ai/API 沙箱中执行，依赖模型宿主预装的工具，而非本地安装。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `skills/` | 20 个自包含技能，每目录一个 SKILL.md + 可选 scripts/references/assets |
| `skills/docx,pptx,xlsx,pdf/` | 生产级文档技能（专有许可），docx 61 文件、pptx 56、xlsx 53 |
| `skills/claude-api/` | Claude API/SDK 参考（96 文件），按语言分子目录（python/go/java/php/ruby/csharp/curl）+ shared/ 34 篇共享文档 |
| `skills/skill-creator/` | 元技能：创建/评测/打包技能，含 agents/（analyzer、comparator、grader 三个子代理定义）与 eval-viewer/ |
| `skills/canvas-design/` | 视觉设计技能，捆绑 54 个字体 |
| `skills/mcp-builder/` | MCP 服务器开发指南 + connections.py/evaluation.py |
| `.claude-plugin/marketplace.json` | Claude Code 插件市场清单：5 个插件（document-skills、example-skills、claude-api、academy-guide、discernment-nudge） |
| `spec/` | Agent Skills 规范入口（指向 agentskills.io） |
| `template/` | 最小 SKILL.md 模板（9 行） |
| `THIRD_PARTY_NOTICES.md` | 46KB 第三方许可声明（imageio 等 BSD 组件） |

组织方式：**无 monorepo 工程，而是「多自包含包」集合**。每个技能目录必须独立可用（可单独 zip 上传到 Claude.ai），因此 docx/pptx/xlsx 三者各自完整复制了一份 office 校验器与 schema（经 md5 证实 `skills/{docx,xlsx,pptx}/scripts/office/validators/base.py` 三份完全相同，875 行），属于有意为之的重复而非疏忽。

## 5. 核心架构与关键实现

### 5.1 SKILL.md 协议（技能即接口）

所有技能共享统一契约：frontmatter 仅强制 `name` 与 `description` 两字段（README「Creating a Basic Skill」），description 是**唯一的触发机制**——skill-creator 明确建议写得「pushy」一点以防欠触发（`skills/skill-creator/SKILL.md`）。正文控制在 500 行内，大参考按域拆分到 references/。claude-api 的 SKILL.md（616 行）展示了协议上限的用法：TRIGGER/SKIP 显式触发规则、针对「训练先验过期」的 API 漂移对照表、`/claude-api migrate` 等子命令表（`skills/claude-api/SKILL.md`）。

### 5.2 docx 技能：生产级 OOXML 处理管线

docx 技能按任务三分法组织（`skills/docx/SKILL.md`）：新建用 docx-js 脚本、编辑走 unzip→改 `word/document.xml`→zip、读取用 pandoc。最有价值的实现是针对「模型编辑 XML」特有障碍的工具：

- `scripts/merge_runs.py`：Word 会把一段文本拆散到大量 `<w:r>` run（修订 ID、拼写检查标记），导致查找替换失效。该脚本合并相邻同格式 run、剥离 rsid/proofErr 标记，且处理 `xml:space="preserve"` 的可见文本与文件字节差异（docstring 有完整论证，约 200 行）。
- `scripts/office/validate.py`：基于捆绑的 ISO 29500 XSD 做全量校验，`--auto-repair` 修 paraId 超限、缺 xml:space 等常见问题；`--author` 模式检查修订（redlining）中每个改动都被 `<w:ins>/<w:del>` 包裹。
- `scripts/comment.py`：Word 批注需六个交叉引用的 XML 文件，脚本自动生成 comments*.xml、关系与 content-type 并打印需手工放入 document.xml 的锚点标记。
- `scripts/accept_changes.py` + SKILL.md 中对 pandoc、LibreOffice 接受修订时「删除段落标记不合并」行为差异的逐项记录——这类踩坑笔记是仓库最有含金量的部分。

### 5.3 skill-creator：技能的评测与打包工具链

把 prompt 当软件维护的完整闭环（`skills/skill-creator/`）：`scripts/run_eval.py` 用 ProcessPoolExecutor 并发跑触发评测（判断某 description 是否让 Claude 在给定 query 下正确触发）；`eval-viewer/generate_review.py`（471 行）生成 HTML 评审页；`scripts/package_skill.py` 打包 .skill 分发文件；`agents/grader.md` 等定义评分子代理。SKILL.md 还包含面向非技术用户的沟通风格指引。

### 5.4 claude-api 技能：多语言 SDK 知识库

616 行 SKILL.md + 按语言目录（python/go/java 等各含 tool-use.md、streaming.md、files-api.md 等）+ shared/ 34 篇（managed-agents 系列、model-migration、prompt-audit、live-sources）。特色是强反幻觉约束：「Never guess SDK usage」，未在文档中出现的绑定必须先 WebFetch 官方 SDK 仓库；两个迁移诊断脚本 `shared/preserved-thinking-migration/prefix_diff.py`（1532 行）与 `drop_block_probe.py`（1066 行）是仓库最大的单文件 Python 程序。

## 6. 构建与运行

无构建步骤。使用方式（README）：

```bash
# Claude Code 中注册为插件市场
/plugin marketplace add anthropics/skills
# 安装插件组
/plugin install document-skills@anthropic-agent-skills
/plugin install example-skills@anthropic-agent-skills
```

或在 Claude.ai 付费版直接使用内置同款技能，或经 Skills API 上传（README 链接 docs.claude.com Skills Quickstart）。技能内部脚本按需调用，如 docx 校验：

```bash
python scripts/office/validate.py out.docx --original doc.docx
```

`template/SKILL.md` 是创建自有技能的起点（frontmatter 两字段 + 正文）。

## 7. 测试与工程质量

- **无自动化测试**：全仓库无 test_*.py、无 tests/ 目录；技能行为的「测试」被显性化为 skill-creator 的评测工作流（LLM 触发评测 + 人工评审），而非传统单元测试——这是 prompt 资产仓库的合理取舍，但 1.8 万行 Python（含 875 行 XML 校验器）没有测试仍是风险点。
- **无 CI**：tarball 与 gh api 均确认不存在 `.github/` 目录。
- **质量观察**：代码风格统一（类型标注、argparse、defusedxml 防注入）；文档质量极高，SKILL.md 普遍包含「为什么」级别的踩坑解释而非仅操作步骤；已知短板是三份 office 校验器代码完全重复（见第 4 章）。

## 8. 社区与维护状态

- 极度活跃：创建于 2025-09-22，13 个月即 18 万+星；最近 push 2026-10-28（分析前一天），`gh api` 实测 open issues 1,446、forks 21,848。
- 贡献者 16 人（gh api contributors），以 Anthropic 员工为主（marketplace.json 中 owner 为 klazuka@anthropic.com；各技能内容明显出自内部团队），外部 PR 空间有限。
- 无 GitHub Releases、无 CHANGELOG——发布节奏由 Claude.ai/插件市场承担，仓库本身是滚动快照。README 声明技能「仅供演示与教学」，生产行为以线上为准。

## 9. 亮点与风险

**亮点**
1. 定义并实例化了「Agent Skills」这一新品类：SKILL.md 协议 + 渐进式披露 + agentskills.io 开放规范，是当前 agent 上下文工程的事实标准之一。
2. 生产级参考实现：docx/pptx/xlsx 技能真实驱动 Claude 文档功能，其中对 OOXML/修订/批注的工程化处理（merge_runs.py、XSD 校验、六文件批注链）文档与代码双高。
3. 元技能闭环：skill-creator 把技能创建、触发评测、基准、打包做成工具链，「prompt 即软件」方法论的唯一完整开源示范。
4. 强反幻觉设计：claude-api 技能的触发规则、API 漂移表、「禁止凭记忆猜 SDK」约束，可直接借鉴到任何 LLM 参考类技能。
5. 许可与合规清晰：逐目录 LICENSE.txt + 46KB 第三方声明 + OFL 字体随附许可文本。

**风险/局限**
1. 深度绑定 Claude 生态（插件市场、Skills API、宿主预装工具链），脱离 Claude Code/Claude.ai 环境的可移植性有限；跨厂商复用需走 agentskills.io 规范自行适配。
2. 无测试、无 CI、无版本化发布，文档技能为 source-available 专有许可，不能自由修改再分发。
3. 技能效果依赖模型行为（触发准确率、指令遵循），仓库明确声明「可能与线上行为不一致」，且 description 触发机制天然存在误触发/欠触发博弈。
4. 大量模板化重复（三份相同校验器、schema 复制），单一技能更新需人工同步三处。

## 10. 总结评价

本仓库是 2025-2026 年 agent 生态最重要的范式输出之一：它把「教模型做事」从散落的提示词升级为带元数据、资源与工具链的标准制品，并以官方文档技能证明了该模式可承载生产级工作负载。代码总量不大（约 1.8 万行 Python + 138 个 Markdown），但信息密度极高——最有价值的不是某个具体函数，而是 SKILL.md 的写法、渐进式披露的组织方式与 skill-creator 的评测闭环。适合三类人：想给自己的 agent 写高质量技能的开发者（精读 skill-creator 与 docx SKILL.md）、研究 agent 上下文工程/工具调用设计的工程师（读 SKILL.md 协议与 claude-api 的触发设计）、以及想把内部流程固化为团队技能资产的企业用户。想看传统意义上的架构与测试，这里没有；想学「如何为 LLM 写说明书」，这是目前最好的教材。
