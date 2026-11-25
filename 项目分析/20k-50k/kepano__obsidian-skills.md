# kepano/obsidian-skills 项目分析

> 星数 49966（快照 2026-10-24）｜主语言 —（纯 Markdown/JSON 内容仓库）｜档位 20k-50k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：Obsidian 官方（第一方）AI Agent Skills 集合——用 6 份 SKILL.md 教会任何兼容 Agent Skills 规范的编码智能体正确读写 Obsidian 生态的笔记与开放格式。

| 项目 | 内容 |
|------|------|
| 名称 | kepano/obsidian-skills |
| 作者 | Steph Ango（@kepano，Obsidian CEO）——`.claude-plugin/marketplace.json` |
| 星数 | 49966（快照），gh api 实时约 50695 |
| 语言 | 无代码；全部为 Markdown + 2 个 JSON 清单 |
| 许可证 | MIT（Copyright (c) 2026 Steph Ango，`LICENSE`） |
| 创建 / 最近推送 | 2026-01-02 / 2026-09-15（gh api） |
| 规模 | 15 个文件、合计约 1991 行（`wc -l` 统计） |
| 版本 | 1.0.1（`plugin.json`），无 git tag、无 GitHub Release |

## 2. 项目是什么，解决什么问题

LLM 编码智能体（Claude Code、Codex、OpenCode 等）默认只熟悉标准 CommonMark/GFM 语法，面对 Obsidian 生态的私有扩展（wikilinks、callouts、`.base`、`.canvas`）时经常生成无法渲染或语法错误的文件。本仓库按 [Agent Skills 规范](https://agentskills.io/specification)（`README.md`）把 Obsidian 的格式知识打包成可被智能体按需加载的"技能包"，让智能体直接产出正确的 vault 文件。

每个技能是一个目录，内含一个带 YAML frontmatter（`name` + `description`）的 `SKILL.md`，`description` 字段写明触发场景（如"Use when working with .base files…"，`skills/obsidian-bases/SKILL.md`），智能体据此判断何时加载该技能；`references/` 子目录存放按需加载的细节参考，避免一次性注入过多 token——这是 Anthropic 提出的渐进式披露（progressive disclosure）模式的标准实现。

6 个技能覆盖三个层面（`README.md` 技能表）：
- **格式层**：`obsidian-markdown`（Obsidian 风味 Markdown：wikilinks/embeds/callouts/properties）、`obsidian-bases`（`.base` 数据库视图语法）、`json-canvas`（JSON Canvas 1.0 规范）；
- **工具层**：`obsidian-cli`（驱动运行中的 Obsidian 实例做读写、搜索、插件调试）、`defuddle`（网页抽正文转 Markdown，作者自家的另一个开源项目）、`knap`（模板 + JSON/CSV 批量生成笔记）；
- **工作流层**：技能之间可以串联，例如 defuddle 抓网页 → knap 套模板 → obsidian-cli 写入 vault（`skills/knap/SKILL.md` 的 "Defuddle pipeline" 一节给出了完整管道命令）。

本质上这不是传统意义的"软件项目"，而是一份被精心工程化的**知识文档仓库**：它的"代码"就是提示词本身，星数反映的是 Obsidian 数千万用户与 Agent Skills 生态交汇后的刚需。

## 3. 技术栈与依赖

- **载体格式**：纯 Markdown（SKILL.md + references）与 JSON（插件清单），无任何编译型代码（全仓库 15 个文件清单）。
- **规范依赖**：Agent Skills 规范（agentskills.io）、JSON Canvas Spec 1.0（`skills/json-canvas/SKILL.md` 引用 jsoncanvas.org）。
- **运行时依赖（可选，由技能指导智能体安装）**：`obsidian` CLI（随 Obsidian 1.10+ 分发，`skills/obsidian-cli/SKILL.md`）、`npm install -g defuddle`、`npm install -g knap`（需 Node.js 20+，`skills/defuddle/SKILL.md` 与 `skills/knap/SKILL.md`）。
- **分发依赖**：Claude Code 的 plugin marketplace 机制（`.claude-plugin/marketplace.json` 定义了一个名为 `obsidian` 的插件）、`npx skills add` 通用安装器（`README.md`）。
- 无 package.json、无 CI、无构建脚本——仓库本身零依赖，所有"依赖"都在文档中描述为智能体的运行时可选能力。

## 4. 目录结构与代码组织

| 路径 | 职责 |
|------|------|
| `README.md` | 定位说明 + 三种安装方式 + 6 技能一览表 |
| `LICENSE` | MIT，2026 |
| `.claude-plugin/plugin.json` | 聚合插件清单：名称/版本/触发描述 |
| `.claude-plugin/marketplace.json` | Claude Code marketplace 发布清单 |
| `skills/obsidian-markdown/` | 主力技能：SKILL.md（196 行）+ references/（CALLOUTS/EMBEDS/PROPERTIES 共 189 行） |
| `skills/obsidian-bases/` | 最大技能：SKILL.md（499 行）+ references/FUNCTIONS_REFERENCE.md（173 行） |
| `skills/json-canvas/` | SKILL.md（244 行）+ references/EXAMPLES.md（329 行完整示例） |
| `skills/obsidian-cli/` | 单文件技能（106 行），CLI 命令速查 + 插件开发调试循环 |
| `skills/defuddle/` | 单文件技能（41 行），最精简 |
| `skills/knap/` | 单文件技能（102 行），模板渲染与批量生成 |

组织方式评述：扁平的单层技能目录，每个技能自包含（SKILL.md 永远可独立阅读，细节放 references/）。没有 monorepo 工具链，也没有代码复用问题。值得注意的是 `plugin.json` 把 6 个技能聚合为一个名为 `obsidian` 的插件（而非每技能一个插件），一次安装全部生效。

## 5. 核心架构与关键实现

作为内容型仓库，其"核心实现"是 SKILL.md 的写作工程。三个值得拆解的设计：

**（1）渐进式披露的分层文档结构。** 每个 SKILL.md 控制在智能体单次可承受的篇幅内，细节外移。以最大的 `skills/obsidian-bases/SKILL.md`（499 行）为例：正文只保留"高频函数表 + 三个完整示例 + 故障排查"，完整的 Date/String/Number/List/RegExp 函数签名放到 `references/FUNCTIONS_REFERENCE.md`，并明确引导："For the complete reference of all types… see FUNCTIONS_REFERENCE.md"。`obsidian-markdown` 同样把 13 种 callout 类型、全部 embed 语法拆到 CALLOUTS.md/EMBEDS.md。

**（2）把"踩坑经验"写成硬性规则。** 文档价值不在罗列语法，而在作者吃过的亏。例如 `obsidian-bases/SKILL.md` 用专门小节强调 Duration 类型不直接支持 `.round()`，必须先取 `.days` 字段（附 WRONG/CORRECT 对照代码）；`json-canvas/SKILL.md` 提醒 JSON 字符串里要用 `\n` 而非字面 `\\n`（否则 Obsidian 渲染出反斜杠）；每条工作流末尾都附"Validate"步骤（如 json-canvas 的 8 项校验清单），把验证内建进智能体的执行流程。

**（3）技能间管道设计。** `skills/knap/SKILL.md` 末尾给出端到管道：
```bash
defuddle parse https://example.com/article --md --json \
  | knap render template.md --data - -o note.md
```
抓取、渲染、（经 obsidian-cli）落盘三个技能形成可组合的自动化链路，这正是"技能集合"优于"单技能"的地方。

**（4）触发描述的精心打磨。** 每个 frontmatter 的 `description` 都是经过 PR 迭代的触发器——最近一次合并的 PR #143 就是"shorten skill description"（提交历史），defuddle 也有专门的 "docs: shorten defuddle skill description" 提交，说明维护者把 description 当作影响召回率的接口来维护。

## 6. 构建与运行

无构建过程。安装即"让智能体看到这些文件"（`README.md`）：

```bash
# 方式一：Claude Code marketplace
/plugin marketplace add kepano/obsidian-skills
/plugin install obsidian@obsidian-skills

# 方式二：通用 skills 安装器
npx skills add https://github.com/kepano/obsidian-skills

# 方式三：手动（Codex / OpenCode）
# Codex: 复制 skills/ 到 ~/.codex/skills
# OpenCode: git clone 到 ~/.opencode/skills/obsidian-skills（须保留完整目录结构）
```

另注意：`obsidian-cli` 技能要求本机 Obsidian 应用处于运行状态（CLI 通过它与活动实例通信）；`defuddle`/`knap` 需 Node.js 20+ 并 `npm install -g`。

## 7. 收录内容与质量

- **条目规模**：6 个技能、10 份文档、约 1991 行；无死链引用（README 技能表与目录一一对应，已核对）。
- **质量机制**：无自动化测试（内容仓库，SKILL.md 的正确性无法单测），质量控制依赖 PR review——贡献历史显示外部贡献以"补充示例/精简描述"类文档 PR 为主（PR #59 补 base embed 示例、PR #143 精简技能描述）。
- **与官方文档的关系**：SKILL.md 大量引用 help.obsidian.md 的官方页面作为延伸阅读，技能内容与官方文档同源（作者即 Obsidian 方），因此准确性有第一方背书。
- **版本管理**：仅靠 `plugin.json`/`marketplace.json` 中的 1.0.1 版本号，无 CHANGELOG、无 tag，用户无法追溯技能内容变更史——对纯内容仓库是可接受的取舍，但属于明显缺口。

## 8. 社区与维护状态

- **活跃度**：2026-01-02 创建，最近推送 2026-09-15（gh api）；提交历史显示 1 月（创建）→ 6 月（bases 生态更新）→ 9 月（新增 knap 技能、描述打磨）三个波峰，与 Agent Skills 生态的产品节奏（Claude Skills 规范开放、knap 发布）吻合。
- **贡献者**：16 人（gh api contributors），但提交以 kepano 本人为主，外部贡献均为小型文档 PR。
- **发布节奏**：无 GitHub Release、无 tag；版本号 1.0.1 表明作者视其为稳定的基础设施而非快速迭代产品。
- **治理信号**：README 同时给出 Claude Code/Codex/OpenCode 三家安装路径，`.claude-plugin/` 采用通用 marketplace 清单格式，刻意保持"规范中立、不绑定 Anthropic"的姿态。
- 最近 7 周（2026-09-15 之后）无新提交，属正常缓释而非弃坑（内容仓库更新频率天然低于代码仓库）。

## 9. 亮点与风险

**亮点**
1. **第一方权威性**：作者为 Obsidian CEO，技能内容与 Obsidian 官方文档同源，是生态内事实上的标准 Obsidian 技能包。
2. **教科书级的 SKILL.md 写法**：渐进式披露、frontmatter 触发描述、references 分层、Validate 清单——这个仓库本身就是"如何写好一个 agent skill"的参考实现，值得所有技能作者模仿。
3. **开放格式组合拳**：围绕 Markdown/Bases/JSON Canvas 三个开放格式构建，跨客户端可用，不锁定任何单一 Agent 产品。
4. **可组合的工作流**：defuddle→knap→obsidian-cli 管道把"网页到笔记的自动化"变成了三条命令。
5. **极小的维护面**：1991 行纯文本承载近 5 万星，维护成本几乎为零。

**风险 / 局限**
1. **时效耦合**：Bases 函数表、obsidian-cli 命令等均锚定当前 Obsidian 版本，产品迭代后若无更新会静默过时（无 CI 校验文档与产品一致性）。
2. **无版本化发布**：缺 tag/CHANGELOG，依赖方难以锁定版本、难以 diff 升级变化。
3. **平台绑定 `--copy`/`silent` 等 CLI 行为**：obsidian-cli 技能强依赖 Obsidian 桌面端运行中实例，无头/服务器场景不可用（`skills/obsidian-cli/SKILL.md` 自述 "Requires Obsidian to be open"）。
4. **内容仓库的可验证性天花板**：无测试即无法防止回归，一条写错的公式示例只能靠用户报 issue 发现。

## 10. 总结评价

这是"内容型仓库也能成为顶级开源项目"的典型案例：Obsidian 创始人用不到 2000 行 Markdown，精准切中了 AI 编码智能体与个人知识管理两大趋势的交点，凭借第一方权威与开放格式中立性成为 Obsidian × Agent 生态的事实标准技能包。它没有代码可读，但每一份 SKILL.md 都是 agent skills 工程的优秀范文——**适合所有 Obsidian 重度用户直接安装使用，适合所有想编写自己 agent skill 的开发者当作模板研读**；对研究"提示词即软件"这一新形态的工程师而言，这个仓库的目录结构、触发描述写法与分层文档策略本身就是值得精读的架构。
