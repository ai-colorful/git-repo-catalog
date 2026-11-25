# emilkowalski/skills 项目分析

> 星数 49146（快照 2026-10-24）｜主语言 Markdown｜档位 20k-50k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：Emil Kowalski（Sonner/Vaul 作者，前 Vercel/Linear）把个人 UI 设计与动效专家经验封装成的 **AI Agent Skills 集合**，让 Claude 等 AI 编码助手按他的"品味"来写界面和动画。

| 项目 | 事实 |
| --- | --- |
| 名称 | emilkowalski/skills（"Skills for Designers and Engineers"） |
| 星数 | 49146（快照 2026-10-24）；分析日 gh api 实测已达 53.4k |
| 语言 | Markdown（无任何代码） |
| 许可证 | MIT（`LICENSE`，Copyright (c) 2026 Emil Kowalski） |
| 创建时间 | 2026-03-16（gh api `repos/emilkowalski/skills`），7 个多月涨至 53k+ |
| 最近活跃 | pushed_at 2026-11-04（分析当天仍有提交），活跃度极高 |
| 规模 | 33 个文件、29 个 Markdown、约 5560 行、406KB（tarball 实测） |
| Release | 0 个 release、0 个 tag（直接消费 HEAD） |

## 2. 项目是什么，解决什么问题

这是典型的"内容型仓库"，但内容不是链接清单，而是 **17 个可安装的 Agent Skill**。每个 skill 是一个目录，含 `SKILL.md`（YAML frontmatter `name` + `description`，符合 Anthropic Agent Skills 规范）加若干支撑文档，全部为提示词/知识文本，无一行可执行代码（tarball 中无 package.json、无脚本）。

解决的问题在 README（`README.md`）里说得直白："Agents don't have great taste"——AI 会给入场动画用 `ease-in`（应为 `ease-out`）、用实线边框代替半透明阴影、给高频操作加动画。这些小错叠加起来让界面"还行但不惊艳"。本项目把作者多年 Vercel/Linear 的一线判断浓缩成显式规则表，供 agent 在写 UI 时直接遵循，是作者文章《Agents with Taste》的工程化落地。

项目本质是**领域专家知识的产品化**：作者在 README 中明确说这些 skill 是"domain-expertise 的副产品"，主张 AI 放大而非替代专家经验。每个 skill 都不是泛泛的"写好动画"，而是带精确值的规定（如 `cubic-bezier(0.23, 1, 0.32, 1)` 的自定义 ease-out、`scale(0.9–0.97)` 起始而非 `scale(0)`、按钮按下 `scale(0.97)`），并反复强调"不许自造数值，一律查表"（`skills/animate/SKILL.md` Hard Rules）。

skill 之间形成了**分工与路由拓扑**：建造（animate / animate-expo / choreograph）、审查（review-animations）、审计（improve-animations）、找机会（find-animation-opportunities）、做减法（remove-animations）、性能（animation-performance）、词汇反查（animation-vocabulary），彼此在 description 中互相指路（如 animate 的描述结尾"for critiquing existing motion use review-animations"，见 `skills/animate/SKILL.md:3`），构成一个完整的动效工作流体系。

## 3. 技术栈与依赖

无技术栈依赖——整仓为纯 Markdown 文本（`tar -tzf src.tar.gz` 全量核对，仅 1 个 LICENSE、2 个根级 cheatsheet、29 个 .md、1 个空 `.pl`）。

- **Skill 格式**：Anthropic Agent Skills 规范（`SKILL.md` + YAML frontmatter `name`/`description`）；3 个 skill 追加 `disable-model-invocation: true` 字段声明"仅手动触发"（`skills/pick-ui-library/`、`skills/prototype/`、`skills/review-animations/` 的 frontmatter）。
- **分发渠道**：`npx skills@latest add emilkowalski/skills`（`README.md`，走 skills.sh 注册表），面向 Claude Code 等 agent 运行时。
- **引用的技术生态**：CSS（cubic-bezier、`@starting-style`、WAAPI）、Motion（motion.dev）、React Native/Reanimated/Expo、Sonner（作者自己的 toast 库，`skills/ask-sonner/` 整个 skill 就是其使用手册）、Swift 6（`skills/write-swift/`）。
- **语言统计**：`.gitattributes` 特意设 `skills/**/*.md linguist-detectable` 让 skills 内容计入语言统计（该文件仅 2 行）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `README.md` | 项目介绍、安装命令、17 个 skill 的参考目录（每个一句定位） |
| `LICENSE` | MIT |
| `ui-cheatsheet.md` / `performance-cheatsheet.md` | 根级速查表：11+7 条"问题→解法"对照（如"数字抖动→`font-variant-numeric: tabular-nums`"） |
| `skills/<name>/SKILL.md` | 17 个 skill 的主文件，frontmatter + 指令正文 |
| `skills/<name>/*.md`（其他） | 支撑文档：RECIPES / STANDARDS / AUDIT / PLAN-TEMPLATE / CATALOG / PICKER / API 共 8 个 |
| `.pl` | 0 字节空文件，用途未确认（疑为占位） |
| `.gitattributes` / `.gitignore` | 仅 linguist 标记和 `.DS_Store` |

组织方式：**扁平的"每 skill 一目录"**，无 monorepo 工具、无构建系统。17 个 skill 大小差异大——最大的 `emil-design-eng`（676 行/1 文件）到最小的 `pick-ui-library`（85 行）。支撑文档按需拆分：主文件管"决策顺序与硬规则"，长内容外置成查表文档（如 animate 的精确配方进 `RECIPES.md`，审查的精确值进 `STANDARDS.md`），主文件用相对引用衔接。

## 5. 核心架构与关键实现

### 5.1 统一的 skill 内部结构（提示词工程范式）

几乎所有 skill 遵循同一骨架（见 `skills/animate/SKILL.md`、`skills/animate-expo/SKILL.md`、`skills/break-ui/SKILL.md` 等）：**Initial Response**（被调用时先说什么）→ **Operating Posture**（角色定位与失败模式）→ **Hard Rules**（不可违反的规则）→ 决策序列 → **Output**（输出格式要求）→ **Tone**。例如 animate 的 Hard Rules 第 2 条：

```markdown
2. **No approximated values.** Every curve, duration, and spring config
   comes from the tables below. Never invent `cubic-bezier(0.4, 0, 0.2, 1)`
   because it looks familiar.
```

这是把"专家品味"防降级的手段：禁止 agent 用训练记忆里的近似值，强制走查表。多个 skill 还强制输出格式——`emil-design-eng` 的 "Review Format (Required)" 规定审查结果必须输出 `| Before | After | Why |` 三列表格并明示"禁止逐行 Before/After 清单"（`skills/emil-design-eng/SKILL.md:40-60`）。

### 5.2 动画决策流水线（animate 系列）

`skills/animate/SKILL.md` 的 Build Sequence 是全项目核心：①该不该动（按使用频率分级，"100+ 次/天：No animation. Ever."，以 Raycast 无开关动画为例证）→ ②目的何在（feedback/空间一致性/状态指示等六类，说不出目的就停）→ ③选工具（"cheapest that works"：CSS transition → `@starting-style` → CSS animation → WAAPI → Motion，按需递进）→ ④选属性（只准 `transform`/`opacity`，`clip-path` 为"批准的第四个"，手写组件需求则转 `pick-ui-library`）→ ⑤曲线/时长/弹簧。同一套决策表在 `review-animations/STANDARDS.md`、`improve-animations/AUDIT.md`、`animate-expo`、`emil-design-eng` 中按角色重复出现，保证建造与审查口径一致。

### 5.3 "计划-执行分离"的审计设计（improve-animations）

`skills/improve-animations/` 拆成 3 个文件：SKILL.md（审计流程）+ AUDIT.md（8 类审计项及精确目标值）+ PLAN-TEMPLATE.md（实施计划模板）。模板开篇写明设计意图："The executor may be a less capable model with zero context and zero taste — the plan must contain everything, exactly"，要求计划引用 `path/to/file.tsx:123` 并附当前代码原文，即**高级模型产计划、廉价模型照单执行**的多模型协作模式。

### 5.4 极端用例目录与固定测试装置（break-ui / prototype）

`skills/break-ui/CATALOG.md` 按"人物名/邮箱/数字/文本"分组收录真实可产生的破坏性数据（如波兰语连字符长名 `Aleksandra Wiśniewska-Kowalczyk` 会撑爆单行、`Jo` 让首字母缩写逻辑只出 `J`、越南语叠声调被 `overflow: hidden` 裁切）；`skills/prototype/PICKER.md` 把变体切换器定义为"非设计决策、逐字复制"的固定深色玻璃胶囊组件，保证原型评审时装置本身不干扰判断。

## 6. 构建与运行

无构建、无测试、无 CI（tarball 全量核对无 `.github/`、无 package.json/Makefile）。实际使用方式（来自 `README.md`）：

```bash
npx skills@latest add emilkowalski/skills
```

安装后各 skill 由 agent 运行时（如 Claude Code）按 description 自动触发，或用户手动点名调用（3 个 `disable-model-invocation: true` 的 skill 只能手动）。无本地预览需求；上游仓库无 release/tag，`npx skills` 每次取最新 HEAD。

## 7. 收录范围、组织与更新机制（内容型仓库分析）

- **收录范围**：17 个 skill 全部为作者个人经验的自产内容（非聚合第三方），主题覆盖 Web 动画（6 个）、React Native/Expo 动画（1 个）、设计哲学（2 个）、移动 Web 适配（1 个）、UI 压测（1 个）、库选型（1 个）、原型（1 个）、Sonner 使用（1 个）、Swift 编码（1 个）、动效词汇（1 个）、动效增删审（3 个）——Swift 是唯一的非 UI 主题。
- **条目规模**：总计约 5560 行 Markdown；单 skill 85–676 行（各目录行数见第 4 章实测清单）。
- **分类体系**：不靠目录分类，靠 description 里的**触发词与互相路由**实现检索（如 choreograph 的 description 列出 stagger/cascade/"feels robotic" 等十余个触发词，`skills/choreograph/SKILL.md:3`）。
- **更新机制**：个人直更（贡献者仅 3 人，gh api contributors），无 PR/自动化流水线、无 release；配套分发为 skills.sh 注册表 + aiforui.dev Newsletter 订阅（README）。
- **姊妹项目与生态位**：作者是 Sonner（React toast 库）与 Vaul 的作者，`ask-sonner` 直接内嵌其官方手册（`skills/ask-sonner/API.md` 为 Toaster 全 props 表）；本项目定位是"agent 时代的付费咨询替代品"，与 awesome 类聚合清单完全不同——它卖的是单一作者的判断力。

## 8. 社区与维护状态

- 活跃度：created 2026-03-16 → pushed 2026-11-04（分析当天），gh api 实测星数已从快照 49146 涨至 53444，增速在 7 个月达 5 万+，属于 2026 年 AI skills 浪潮中的头部个人仓库。
- 维护结构：单人主导（contributors=3，未确认其余两人贡献内容；issues 仅 1 个开放），无 CI、无测试、无 release 流程——对纯内容仓库属常见做法。
- 社区消费：3111 forks（gh api）表明大量用户 fork 后自改自用；skills.sh 徽章（README）显示其按安装量排名的注册表生态。

## 9. 亮点与风险

亮点：

1. **专家知识的高保真封装**：所有关键值都显式成表并禁止 agent 自造近似值（`skills/animate/SKILL.md` Hard Rules），直击"agent 品味差"的真实痛点。
2. **skill 间路由拓扑设计精良**：description 互相指路 + 触发词枚举，让 17 个 skill 各司其职不互相抢活（各 SKILL.md frontmatter）。
3. **建造/审查/审计/减法四视角闭环**：animate、review-animations、improve-animations、remove-animations + find-animation-opportunities 覆盖动效全生命周期，口径统一（同一频率表五处复现）。
4. **计划-执行分离**（`improve-animations/PLAN-TEMPLATE.md`）适配多模型协作现实，计划自包含到"零品味模型也能执行"。
5. **可复用的 skill 写作范式**：Initial Response/Posture/Hard Rules/Output/Tone 结构是给其他领域专家做 skill 的模板。

风险/局限：

1. **单一信源无对冲**：全部内容是个人观点的规则化（部分断言较绝对，如"键盘操作永不动画"），与其他专家共识可能冲突，且无社区评审机制。
2. **与具体 agent 运行时耦合**：依赖 skills.sh/npx 分发与 Claude 风格的 skill 触发机制，其他生态（如 Cursor rules、Copilot instructions）需自行转换。
3. **规则难以随技术演进自动保鲜**：无 CI 校验内容时效，文中涉及 Motion 版本等事实（如 "motion 14.2+ hardware accelerated"）需人工维护。
4. **中文/国际化场景覆盖有限**：内容默认英文界面语境（break-ui 的 CATALOG 虽含 CJK 用例，但整体面向英文 UI）。

## 10. 总结评价

这是 2026 年 "AI skills" 浪潮中质量最高、定位最清晰的个人仓库之一：它不聚合链接，而是把一位顶级设计工程师的判断力写成 17 份可安装、互相路由、口径一致的 agent 指令集，用 5560 行 Markdown 换来了 5 万+ 星，验证了"领域专家知识产品化"这条路。适合两类人：一是做 Web/RN 界面与动效、想让 AI 产出达到资深设计工程水准的开发者（直接 `npx skills@latest add` 安装）；二是想学习"如何把个人专长写成高质量 agent skill"的从业者——其统一骨架、查表强制、输出格式约束、计划-执行分离等手法本身就是最佳范本。读"源码"（即这些 Markdown）的成本极低，收益却超出前端范畴。
