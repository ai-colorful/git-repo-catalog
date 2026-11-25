# ComposioHQ/awesome-claude-skills 项目分析

> 星数 77748（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：Claude Skills（Anthropic 2025 年 10 月推出的开放技能包标准）的精选清单 + 官方示例技能合集，是 Composio 围绕 Skills 生态打造的核心获客与流量枢纽。

| 项 | 值 |
|---|---|
| 仓库 | ComposioHQ/awesome-claude-skills |
| 星数 / Fork | 77748（API 实时 78349）/ 9438（gh api 元数据） |
| 主语言 | Python（仓库实际以 Markdown 为主体，Python 仅是少量技能内嵌脚本） |
| 许可证 | README 声称 Apache 2.0，但根目录无 LICENSE 文件，GitHub API license=null（见 README.md:497、tarball 根目录） |
| 创建时间 | 2025-10-17（与 Anthropic 发布 Skills 同月，gh api `created_at`） |
| 最近活跃 | master 最近提交 2026-07-24（`Update README.md`，gh api commits），pushed_at 2026-09-18 |
| 默认分支 | master（gh api） |
| Open PR / Issue | 1694 个 open PR、184 个 open issue（gh api search，2026-10-31） |
| 规模 | 1142 个文件、864 个 SKILL.md、README 503 行 / 184 个列表条目（tarball 统计） |

## 2. 项目是什么，解决什么问题

Claude Skills 是 Anthropic 于 2025 年 10 月发布、2025 年 12 月开源的"技能包"标准：每个技能是一个含 `SKILL.md`（YAML frontmatter + Markdown 指令）的文件夹，可选捆绑脚本、参考文件与资源（README.md:99-105）。本仓库解决的问题是：Skills 生态爆发后缺乏统一入口，用户不知道有哪些可用技能、哪些技能质量可信、如何跨平台（Claude.ai / Claude Code / API / Codex / Cursor / Gemini CLI）使用（README.md:36、101）。

仓库实际承担三重角色。其一是经典 awesome 清单：README 按 11 个分类（Document Processing、Development & Code Tools、Data & Analysis、Business & Marketing、Communication & Writing、Creative & Media、Productivity & Organization、Collaboration & PM、Security & Systems、Assistive Technology、App Automation via Composio）收录外部社区技能链接并附一句话点评（README.md:107-339）。其二是技能本体仓库：顶层 31 个目录是可直接安装的实体技能（如 `skill-creator/`、`mcp-builder/`、`canvas-design/`，其中 `document-skills/docx|pdf|pptx|xlsx` 为 Anthropic 官方技能的复制，docx/SKILL.md frontmatter 标注 Proprietary 许可）。其三是 Composio 商业生态的技能分发渠道：`composio-skills/` 下 832 个 `*-automation/` 目录全部依赖 Composio 的 Rube MCP 网关（frontmatter 声明 `requires: mcp: [rube]`，见 composio-skills/slackbot-automation/SKILL.md）。

README 对 Skills 的定位解释质量较高：明确区分了 Skills（行为层）、MCP（接入层）、Tools（动作层）三者关系，并说明渐进式加载机制——会话开始只加载约 100 token 的名称与描述，正文（通常 <5000 token）按需加载，这是单 agent 可挂载数百技能的关键（README.md:105-106）。这段科普也被社区广泛引用，是 README 本身成为链接 magnets 的原因之一。

增长故事：创建于 Skills 发布同月，12.5 个月冲到 77k+ 星。机制见第 7、8 章——一个自动化 PR 门禁 + "上榜者自发传播"的飞轮。

## 3. 技术栈与依赖

作为内容型仓库，"技术栈"由三部分组成（文件类型统计来自 tarball：892 md、78 xsd、62 py、54 ttf、2 sh）：

- **Markdown 技能包**：主体。864 个 SKILL.md 遵循统一格式——`name`/`description` 必填 frontmatter，正文按 When to Use / Instructions / Examples 组织（规范见 CONTRIBUTING.md:32-94 与 README.md:389-435）。
- **CI 校验脚本**：`.github/workflows/label-ready-skill.yml`（159 行）内嵌 Node.js 脚本做 PR 校验，依赖 `gh` CLI 与 GitHub API（workflow 第 15-50 行）。
- **技能内嵌代码**：62 个 Python 脚本（document-skills 32 个 OOXML 处理、slack-gif-creator 20 个、webapp-testing 4 个 Playwright 测试等）、2 个 Bash 脚本（artifacts-builder/scripts/）、78 个 XSD（document-skills/docx 与 pptx 各 39 个，用于 OOXML schema 校验）、54 个 TTF 字体（canvas-design/canvas-fonts/，供视觉设计技能离线使用）。
- **运行时依赖**：技能本身零安装依赖，按需调用宿主环境的 Python/Playwright/yt-dlp 等；connect-apps 技能依赖 Composio API key 与 Rube MCP 端点（connect-apps/SKILL.md:20-30）。

## 4. 目录结构与代码组织

顶层 33 个条目的职责划分：

| 目录/文件 | 职责 |
|---|---|
| `README.md`（503 行） | 清单主体：分类索引、Skills 科普、上手指南、技能创作模板 |
| `CONTRIBUTING.md` | 贡献规范：7 条质量要求、SKILL.md 模板、PR 流程、署名规则 |
| `skill-creator/` | 元技能：教 Claude 创建新技能（含 `skills` 分级说明，3 个 py 脚本） |
| `mcp-builder/` | 教 agent 构建高质量 MCP server（Python/TypeScript 双栈） |
| `document-skills/` | docx/pdf/pptx/xlsx 四个文档技能（Anthropic 官方复制，含 OOXML schema 与处理脚本） |
| `connect/`、`connect-apps/`、`connect-apps-plugin/` | Composio 打通 1000+ SaaS 的接入技能与 Claude Code 插件（plugin.json + setup 命令） |
| `composio-skills/` | 832 个 SaaS 自动化技能（每目录仅一个 SKILL.md，指向 Rube MCP） |
| `template-skill/` | 空白技能模板（SKILL.md 仅 4 行） |
| 其余 24 个目录 | 具体场景技能：canvas-design、webapp-testing、slack-gif-creator、lead-research-assistant、twitter-algorithm-optimizer 等 |
| `.github/workflows/label-ready-skill.yml` | 唯一 CI：PR 自动校验与打标 |

组织方式评述：单体仓库双轨制——README 层是"外部收录"（113 个外链），目录层是"自有技能"（31 个实体目录 + 832 个生成的自动化技能）。`composio-skills/` 的 832 个目录结构高度模板化（每个仅一个 SKILL.md、同一套 RUBE_SEARCH_TOOLS → RUBE_MANAGE_CONNECTIONS → 执行 的三步模式），明显是脚本批量生成的，与手写的社区技能形成鲜明对比。

## 5. 核心架构与关键实现

内容型仓库，以下三个"实现"是仓库的核心资产。

**（1）自动化 PR 门禁（.github/workflows/label-ready-skill.yml）**

这是仓库最重要的工程实现：一个 pull_request_target 触发的 Node 脚本，对"只加一条技能条目"的 PR 做六项自动校验，全部通过自动打 `ready-to-merge` 标签。关键逻辑（workflow 第 45-149 行）：

```javascript
// 1. 只允许改 README.md
if (changed.length === 0) fail('no changed files reported');
for (const f of changed) {
  if (f !== 'README.md') fail(`disallowed file changed: ${f}`);
}
// 2. 改动只能落在 ## Skills 与 ## Getting Started 之间
if (bPre !== hPre) fail('changes detected before the Skills section');
if (bPost !== hPost) fail('changes detected after the Getting Started section');
// 5. 新增条目必须外链，且禁止指向 composio.dev / anthropic.com
if (host.endsWith('composio.dev') || host.endsWith('anthropic.com')) {
  fail(`bullet "${b.name}" links to internal host ${host}`);
}
// 6. 封杀 crypto/web3/nft 关键词
const blocked = /\b(crypto|cryptocurrency|web3|blockchain|nft|defi|...)\b/i;
// 7. 新增条目必须符合所在分类的字母序
if (prev && ci(prev.name).localeCompare(ci(items[i].name)) > 0) {
  fail(`"${items[i].name}" placed after "${prev.name}" ...`);
}
```

设计意图清晰：把人工审核压缩到几乎为零（合格 PR 直接可合并），同时防住垃圾链接（crypto）与自我推广（禁内链/禁自有 host）。

**（2）标准化技能模板与元技能**

`skill-creator/SKILL.md` 是技能的"技能"：定义了 SKILL.md + scripts/references/assets 的标准解剖结构（skill-creator/SKILL.md:18-30），并提供打包脚本。`template-skill/` 与 CONTRIBUTING.md:32-94 的模板共同构成新人低门槛入口——复制模板、填 frontmatter、提 PR。

**（3）文档技能的 OOXML 工程实现**

`document-skills/docx/` 是仓库中工程质量最高的部分：SKILL.md 教 Claude 把 .docx 当 ZIP 处理，附 `docx-js.md`、`ooxml.md` 两份参考文档、`scripts/` 内 32 个 Python 脚本、39 个 XSD schema 用于修改后校验 XML 合法性。这套东西是 Anthropic 官方 `anthropics/skills` 仓库的复制（SKILL.md frontmatter 标注 Proprietary license），本仓库将其作为"标杆技能"展示。

## 6. 构建与运行

无构建系统（无 package.json/pyproject）。使用方式即"把目录装进 agent"（README.md:341-367）：

```bash
# Claude Code：把技能目录放进配置目录
mkdir -p ~/.config/claude-code/skills/
cp -r skill-name ~/.config/claude-code/skills/
head ~/.config/claude-code/skills/skill-name/SKILL.md   # 验证 frontmatter
claude                                                   # 启动后按需自动激活

# connect-apps 插件（Composio 生态）
claude --plugin-dir ./connect-apps-plugin
/connect-apps:setup      # 粘贴 Composio API key
```

技能创作流程：复制 `template-skill/` → 按 CONTRIBUTING.md 模板填写 → 提 PR（CI 自动校验）。

## 7. 收录范围、规模与自动化治理（测试与工程质量特例）

- **条目规模**：README 共 184 个列表条目（`^- \[` 统计），113 个外部链接、101 个内部 `./` 链接。分类分布：App Automation via Composio 78、Development & Code Tools 32、Creative & Media 10、Productivity 9、Communication 7、Document Processing/Collaboration/Business 各 6、Data & Analysis 5、Security 4、Assistive Technology 1（awk 分类统计）。
- **声称与实际**：README 声称 "1000+ production ready skills"（README.md:36），实际全仓 SKILL.md 共 864 个（832 个为模板化生成的 composio 自动化技能）；"production ready" 无任何验证机制，属营销口径。
- **已知断链问题**：README 中 78 个 `./xxx-automation/` 内部链接指向的目录全部不存在于仓库顶层——这 78 个目录实际位于 `composio-skills/` 下（如 `composio-skills/slack-automation` 类目录），README 未随目录迁移更新，截至快照快照全部 404（逐目录比对验证）。
- **质量治理**：CI 门禁覆盖"格式合规"（字母序、条目格式、改动范围）而非"内容质量"；7 条 CONTRIBUTING 质量要求（真实用例、有示例、跨平台测试等）依赖作者自觉，无人工或自动内容评审。

## 8. 社区与维护状态

- **PR 流水线特征显著**：1694 个 open PR 对 420 个 closed——大量"上榜申请"排队等待合并，这是 awesome 类仓库少见的状态（gh api search，2026-10-31）。184 个 open issue 相对温和。
- **贡献者结构高度头部化**：Prat011（38 次提交）、sohamganatra（11 次）之后即断崖，第 3 名起均为 1-2 次（gh api contributors）。合并节奏：2026-05-22 一天合并多个 PR（`Add overkill skill (#880)`、`Add building-blog skill (#893)`），呈批量审复合并模式（gh api commits）。
- **活跃度走势**：创建于 2025-10（Skills 发布红利期），master 最近提交 2026-07-24（README 调整，应为顶部 banner/utm 更新），pushed_at 2026-09-18。快照前 3 个月 master 无新条目合并但 PR 队列持续膨胀，维护热度在下降（未确认 push 期间是否在其他分支活动）。
- **商业背景**：仓库归 Composio（agent 集成平台）所有，README 顶部 banner、connect-apps 教程、composio-skills 全集、底部 CTA（"Join 20,000+ developers"）构成完整获客漏斗；所有 composio 相关链接带 utm_source=Github 追踪参数（README.md:4-6、62）。

## 9. 亮点与风险

**亮点**

1. CI 门禁是 awesome 类仓库的工程化标杆：六项规则（范围限定/外链强制/关键词封禁/字母序）把清单维护的人工成本降到接近零，值得所有清单类仓库借鉴（label-ready-skill.yml）。
2. README 的 Skills 科普段落（三层架构辨析、渐进式加载量化）质量高于官方多数文档，本身就是传播资产（README.md:99-106）。
3. 内容双轨供给：外部精选 + 864 个可直接安装的实体技能，从"看清单"到"用起来"零跳转。
4. 蹭点精准：创建于 Skills 标准发布同月，一年 77k 星印证了"生态早期抢定义权"的窗口价值。
5. document-skills 收录 Anthropic 官方 OOXML 技能集，含 schema 校验脚本，是仓库内可直接复用的最高价值资产。

**风险/局限**

1. 断链与口径失真：78 个 README 内链 404、"1000+ skills" 实为 864 且其中 96% 是模板化生成物，清单一半以上是 Composio 自有内容，中立性存疑（见第 7 章证据）。
2. 无 LICENSE 文件与 README 声明的 Apache 2.0 矛盾，GitHub 也无法识别许可证，复制/分发技能存在合规模糊（tarball 根目录无 LICENSE）。
3. 维护减速：master 近 3 个月无合并，1694 个 PR 积压，清单的"新鲜度"承诺正在透支（gh api）。
4. composio-skills 的 832 个技能同质化严重且全部导向 Rube MCP 商业端点，对非 Composio 用户是噪音。

## 10. 总结评价

这是一个把 awesome 清单做成增长引擎的教科书案例：内容上，它靠 Skills 标准发布的时间窗口、高质量科普段落和低门槛贡献流程在 12.5 个月内聚集了 77k 星；工程上，它的 PR 自动门禁确实解决了清单仓库"人工审核不可扩展"的老问题；但光环之下，README 声称与仓库实况的差距（断链、虚报规模）、缺失的 LICENSE 和正在积压的 PR 队列说明其增长引擎已从"内容价值"切换到"惯性运转"。适合三类人看：想建设/治理 awesome 清单的维护者（直接读 label-ready-skill.yml，这是全仓最值得抄的 159 行）；想快速上手 Claude Skills 的使用者（读 README 第 2、Getting Started 节 + document-skills）；以及研究开发者增长打法的从业者（把它当一个获客漏斗样本观察）。想逐个读 864 个技能的没必要——composio-skills 是同一模板的 832 次实例化，读通一个即读通全部。
