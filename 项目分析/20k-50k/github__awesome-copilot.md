# github/awesome-copilot 项目分析

> 星数 41,129（快照 2026-11-14，gh api 核实）｜主语言 JavaScript（工具链）+ Markdown（内容）｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：GitHub 官方运营的 Copilot 定制资源集合——社区贡献的 agents/instructions/skills/plugins/hooks/workflows，经工业化自动化管线审核后，以 Copilot CLI 插件市场 + 官网分发。

| 项目 | 值 |
|------|-----|
| 名称 | github/awesome-copilot |
| 星数 / Forks | 41,129 / 5,354（gh api 2026-11-14） |
| 语言 | JavaScript（约 5,400 行 Node 工具链脚本）+ Markdown 内容主体 |
| 许可证 | MIT（LICENSE） |
| 创建时间 | 2025-06-11（gh api） |
| 最近推送 | 2026-11-13，即分析当天前一天仍在高频合并（gh api） |
| 贡献者 | 约 421 名具名 git 贡献者（gh api 分页核实），.all-contributorsrc 登记 499 人 |
| 官网 | awesome-copilot.github.com（homepage 字段） |

## 2. 项目是什么，解决什么问题

这是"awesome 清单"的进化形态。2025 年 6 月创建时它是 Copilot 自定义提示词的社区列表，到今天（快照）已经成长为 GitHub Copilot 定制生态的**官方集散地**：收集社区编写的智能体定义、编码规范指令、Agent Skills、插件包、生命周期钩子和 agentic workflow，统一格式、统一校验、统一分发。README（`README.md`）只保留安装入口与贡献者表，全部目录由脚本自动生成。

它解决三个真实痛点：其一，Copilot 的定制能力（`.agent.md`、`.instructions.md`、SKILL.md、hooks）散落在各博客与 gist，用户无处发现；其二，提示词质量参差且不可验证，需要有人审；其三，插件生态缺少分发渠道——本仓库直接被注册为 Copilot CLI/VS Code 里的默认插件市场，用户一条命令安装（`README.md` "Install a Plugin" 章节：`copilot plugin install <plugin-name>@awesome-copilot`）。

内容覆盖面极广：从 Azure/AWS 架构师、.NET 升级、WCAG 无障碍审计这类工程领域，到 Arch Linux、Adobe Creative Cloud 等长尾场景（`agents/` 目录文件名抽样可见）。与 anthropics/skills 只收 Agent Skills 不同，本仓收六类资源并把它们组合成插件；与 wshobson/agents 的单人生成器模式不同，这里是多人审核 + 机器校验的工业化流水线。

GitHub 官方身份是这个仓库区别于所有第三方集合的根本：它自带分发渠道（CLI 预注册市场）、aka.ms 短链安装按钮、配套教学网站与 Learning Hub，并且仓库自身的运维大量使用 Copilot agentic workflows 自举（`.github/workflows/` 中 60 余个 workflow，其中大量 `*.lock.yml` 为 gh-aw 编译产物）。

## 3. 技术栈与依赖

内容主体是 Markdown + YAML frontmatter；工程部分是纯 Node.js（无框架）：

- **工具链**（`package.json`）：依赖仅 `js-yaml`、`vfile`、`vfile-matter`（frontmatter 解析）与 `ajv`（JSON Schema 校验）、`@microsoft/vally`（微软官方校验器）、`all-contributors-cli`。零重型依赖。
- **官网**（`website/package.json`）：Astro 7 + React 19 + `@primer/react-brand`（GitHub Primer 设计系统）+ shiki 代码高亮 + choices.js 全文搜索过滤；含 6 种语言内容目录（zh-cn/ja-jp/ko-kr/pt-br/es-es/en）与 `llms.txt` 机器可读清单（`website/src/pages/llms.txt.ts`）。
- **CI/自动化**：GitHub Actions 60+ workflow + gh-aw（GitHub Agentic Workflows，`mcp.json` 中配置 `gh aw mcp-server`）；仓库级 MCP 配置（`mcp.json`）与 `.vscode/` 配置让贡献者用 Copilot 开发本仓本身。
- **其他工程文件**：`.editorconfig`、`.codespellrc`（拼写检查配置）、`.gitattributes`、`SECURITY.md`、`SUPPORT.md`、`CODE_OF_CONDUCT.md`，治理文件齐备度接近 GitHub 官方一线产品仓。
- **测试**：Node 内置 `node:test`（如 `.github/scripts/catalog/validate-plugins.test.mjs`）。

## 4. 目录结构与代码组织

| 目录 | 条目规模（真实文件统计） | 职责 |
|------|------|------|
| `agents/` | 230 个 `.agent.md`，42,848 行 | 智能体定义（frontmatter 声明 tools/MCP） |
| `instructions/` | 195 个 `.instructions.md`，58,663 行 | 按 glob 应用的编码规范 |
| `skills/` | 463 个顶层目录、478 个 SKILL.md，共 95,374 行 | Agent Skills（SKILL.md + 脚本资产 py/mjs/sh/ps1） |
| `plugins/` | 113 个本地插件（`plugin.json`）+ `external.json` 外部收录 | 主题化插件包，marketplace 共 220 条（`.github/plugin/marketplace.json`） |
| `extensions/` | 41 个 canvas 扩展（`extension.mjs` + assets） | Copilot Canvas 可视化扩展源码 |
| `hooks/` | 8 个 hook 目录（README.md + hooks.json） | 事件钩子（如 secrets-scanner、tool-guardian） |
| `workflows/` | 8 个 `.md` | gh-aw 格式的仓库自动化（relevance-check 等） |
| `cookbook/` | copilot-sdk 食谱 | 可复制的 API 用法示例 |
| `website/` | 813 文件、408 个内容 md | Astro 官网 + 多语言 Learning Hub（22 篇） |
| `.github/scripts/catalog/` | 5,391 行 JS | 生成/校验/脚手架工具链（带测试） |
| `.github/workflows/` | 60+ 个 | CI 与 agentic 运维 |

条目的分类体系靠三处约定支撑：目录即类型（`agents/` vs `instructions/` vs `skills/`，文件扩展名同样编码类型：`.agent.md`、`.instructions.md`）、frontmatter 即元数据（agent 的 `tools`/`mcp-servers`、instruction 的 `applyTo` glob、skill 的 `name`/`description`）、插件即分类聚合（113 个 `plugin.json` 把相关 agent+skill 打包成 azure-cloud-development、accessibility-kanban 这类主题包，`plugins/azure-cloud-development/plugin.json`）。skills 是最大的门类且结构最复杂：单 skill 可以是纯指令（`skills/` 下单个 SKILL.md），也可以带可执行资产与子技能嵌套（478 个 SKILL.md 多于 463 个顶层目录即来自嵌套）。

组织方式评述：**扁平目录 + 命名约定 + frontmatter 元数据**。没有分类子目录——全部条目平铺在类型目录里，靠 kebab-case 命名（强制校验：`validate-skills.mjs` 要求 name 与目录名一致）和 frontmatter 中的 description 提供元信息；发现性交给官网搜索和 `docs/README.<type>.md` 自动生成的表格。这是与 anthropics/skills 一致的"平铺 + 元数据"路线，在条目过千后仍可维护，因为 README/官网/市场三处视图全部由脚本生成，人工只维护内容本体。

## 5. 核心架构与关键实现

（内容型仓库，本章分析其自动化工具链的核心实现）

**(1) 生成式 README 与市场**。`package.json` 的 build 命令串起两个核心脚本：`node ./.github/scripts/catalog/update-readme.mjs`（890 行）扫描各目录 frontmatter，重写 `README.md` 表格与 `docs/README.agents.md` 等 6 个分类页，每条目附 aka.ms 安装徽章（VS Code / VS Code Insiders 一键安装链接）；`generate-marketplace.mjs`（132 行）合并本地 `plugin.json` 与 `external.json`，产出 `.github/plugin/marketplace.json`（220 个插件）。

**(2) 校验器**。`validate-skills.mjs`（172 行）检查 SKILL.md：frontmatter 可解析、name 合法、name 与目录名一致、description 非空且长度达标、frontmatter 中声明的资产文件真实存在（第 85-95 行逐个 `fs.access` 验证）；`validate-plugins.mjs`（522 行）校验 plugin.json 对 agent-plugins.org schema 的符合性与引用完整性。CI 中对应 `validate-skills.yml` / `validate-plugins.yml` / `validate-readme.yml`，PR 未过校验无法合并。内容写入规范由 `.github/copilot-instructions.md` 逐条列出（文件名 kebab-case、description 非空、agent 声明 agent/ask/Plan 之一），使 AI 评审与人工评审使用同一张检查表。

**(2b) 脚手架**。`create-skill.mjs`（222 行）与 `create-plugin.mjs`（196 行）让贡献者 `npm run skill:create` 生成合规骨架，把"格式正确"从评审负担变成起点状态；`materialize-plugins.mjs`（208 行）+ `clean-materialized-plugins.mjs`（332 行）负责把插件引用的 agents/skills 物化到 `.github/plugin/` 下供 CLI 消费，`extension-plugin-ownership.mjs` 保证 extensions 与其 plugin.json 一一对应。

**(3) 外部插件供应链锁定**。第三方插件不允许直接 PR（`CONTRIBUTING.md` "What We Don't Accept" 明确禁止），必须走 issue 提交，批准后由维护者写入 `plugins/external.json`，每条带 `source.ref` + `source.sha` 双重 pin（如 accessibility-agents 条目 pin 到 `copilot-plugin-v7.1.0-aaf1cfb` / sha `44594ff1...`），防止上游仓库被篡改后毒化市场。

**(4) 六个月强制复审**。`CONTRIBUTING.md` "Review cadence" 章节定义了完整标签状态机（`awaiting-review → ready-for-review → approved/rejected → re-review-due → removed`），nightly workflow 扫描关闭满 6 个月的 approved issue 打上 `re-review-due`，维护者用 `/re-review-keep`、`/re-review-needs-changes`、`/re-review-remove` 三个 slash 命令续期或下架——内容会过时是被正面管理的常态，而非放任。

**(5) hooks 与 agentic workflows**。`hooks/` 收录 8 个事件钩子（governance-audit、secrets-scanner、tool-guardian、fix-broken-links、dependency-license-checker 等），每个是 README.md（frontmatter 元数据）+ `hooks.json` 配置对的文件夹；`workflows/` 收录 8 个 gh-aw 格式的仓库自动化 workflow（`.md` 文件内嵌 `on: slash_command`、`engine: copilot`、`safe-outputs` 等 frontmatter，如 `workflows/relevance-check.md` 实现"检查 issue/PR 是否仍与项目相关"的斜杠命令），由 GitHub Actions 以编译后的 `.lock.yml` 执行。

**(6) 自举运维**。仓库自己也是 Copilot 定制的示范田：`.github/copilot-instructions.md` 定义 AI 代码评审规则，`.github/agents/` 有仓库专用 agent，`.github/extensions/` 有三个自研 GitHub App 扩展（duplicate-review-board、external-plugin-review-board、open-pr-board，各带 `extension.mjs` + Node 测试），60+ workflow 中大量 `*.lock.yml` 是 gh-aw 把 markdown workflow 编译后的产物（如 `pr-duplicate-check.lock.yml`、`resource-staleness-report.lock.yml`）。

## 6. 构建与运行

来自 `package.json` scripts 与 `AGENTS.md`（均为实际可用命令）：

```bash
npm ci                        # 安装依赖
npm run build                 # 生成分类 README + .github/plugin/marketplace.json
npm run skill:validate        # 校验 skills/ 全部 SKILL.md
npm run skill:create -- --name <name> --description "..."   # 脚手架新建 skill
npm run plugin:validate       # 校验 plugin.json
npm run plugin:create -- --name <name>                      # 脚手架新建插件
npm run contributors:generate # 重新生成贡献者表
npm run website:build         # 构建 Astro 官网（先跑 website:data）
npm run website:a11y          # 官网构建 + 无障碍审计
```

运行时使用：`copilot plugin install <plugin-name>@awesome-copilot`（CLI 已预注册该市场）；单文件使用则直接下载 `.agent.md`/`.instructions.md` 放入仓库；VS Code 内通过生成的 aka.ms 徽章链接一键安装（`docs/README.agents.md` 每行附 Install 徽章）。

## 7. 收录标准与审核机制（测试与工程质量 → 特例改写）

**收录范围**（`CONTRIBUTING.md` "What We Accept"）：任何帮助开发者高效使用 Copilot 的技术、框架、方法论——语言/框架、最佳实践、架构模式、测试策略、DevOps、无障碍、性能优化；付费服务类内容需先参照官方 Discussion #968 指引。

**负面清单**（"What We Don't Accept"）同样成文：违反 Responsible AI 原则、危害安全、恶意用途、绕过平台政策的内容直接拒收；最具信息量的一条是拒绝"只重复前沿模型已有能力而无实质增益"的泛型提示词（generic TypeScript/HTML 等）——官方明确认识到弱提示词会比模型默认行为更差。

**三道审核门**：内容门（上述负面清单）、质量门（Be specific/Test your content/遵循命名约定）、结构门（frontmatter 格式、资产存在性、schema 校验，全部机器执行）。
- **人工审核按目录分域**（`CODEOWNERS`）：5 个 GitHub 团队分别负责 extensions/plugins/内容类/workshop/核心维护，规则明确"团队拥有路径，禁止把个人加为默认 owner"，评审人池在 `.github/review-routing.yml` 中由机器人自动指派（`review-routing.yml` workflow）。
- **外部插件专门管线**：`external-plugin-intake`（自动校验提交）→ `external-plugin-quality-gates`（质量门）→ `external-plugin-approval-command` / `rereview-command`（维护者 slash 命令）→ 六个月复审闭环。全程 issue+PR 双轨，每步有对应 workflow（`.github/workflows/` 可数出 11 个 external-plugin/pr 质量相关 workflow）。
- **工程纪律**：codespell、行尾检查、贡献者校验、重复资源检测（`duplicate-resource-detector`、`pr-duplicate-check`）、PR 风险扫描（`pr-risk-scan`）、资源过期报告（`resource-staleness-report`）。工具链脚本自身有 4 个 `*.test.mjs` 测试文件（`node:test`），对 5,400 行脚本而言覆盖率偏轻，但校验逻辑有回归测试兜底。
- **评审责任成文**：`CONTRIBUTING.md` "Maintainer review responsibilities" 明确维护者需核对仓库真实性、元数据质量与与现有条目的差异化价值；`docs/maintainers/` 目录存放评审路由与治理文档，评审标准不是口头传统而是仓库内可审计的文档。
- **无 release**：0 个 GitHub Release（gh api），版本语义由每个插件 `plugin.json` 的 version 字段承载，市场按仓库快照分发。

## 8. 社区贡献、维护状态与生态位

- **活跃度极高**：2026-11-13 单日仍有 aaronpowell（GitHub DevRel，近期提交主力）多次合并与 dependabot 升级（gh api commits）。41k 星 / 5.3k fork / 58 open issues，issue 数量相对体量极低，说明吞吐健康。
- **更新节奏**：无版本发布节点，主干持续滚动合并；内容（agents/skills/plugins）与工程（工具链/官网）由同一条主干承载，工程类变更走 dependabot 与官方员工提交，内容类变更走社区 PR。仓库仍按周级频率新增条目（skills 目录内可见近期新增的 canvas 类插件族，如 arcade-canvas、backrooms-canvas 等一整个生成式图形扩展批次）。
- **贡献者结构**：约 421 名具名贡献者（gh api 分页 Link header page=421 核实），`.all-contributorsrc` 登记 499 人含文档/评审类贡献；topics 含 `hacktoberfest`，2025 年活动期批量引入外部贡献是可见的增长杠杆。维护层为 GitHub 员工 + 社区评审团队双轨。
- **增长曲线**：2025-06-11 创建，18 个月到 41k 星（gh api 快照）；关键催化是 README 所述"Awesome Copilot marketplace 已在 Copilot CLI/VS Code 中预注册"——官方产品入口直接导流，这是任何第三方集合都无法复制的分发优势。
- 精确的逐月 star 历史：未确认（规范限制不做额外网络请求）。

**姊妹项目与生态位**。gh api 核实的三个同生态仓库（快照 2026-11-14）：

| 仓库 | 星数 | 创建 | 模式 |
|------|------|------|------|
| anthropics/skills | 194,089 | 2025-09-22 | Anthropic 官方 Agent Skills 仓，只收 skills，含 examples 文档，无市场与网站 |
| wshobson/agents | 42,486 | 2025-07-24 | 单人主导：Markdown 写一次，生成器适配 Claude Code/Codex CLI/Cursor/OpenCode/GitHub Copilot/Pi 七种工具，带 validation 与 evals |
| github/awesome-copilot | 41,129 | 2025-06-11 | GitHub 官方：六类资源 + 插件市场 + 官网 + 多人审核管线 |

生态位差异：anthropics/skills 是"官方能力标准库"，胜在原始权威；wshobson/agents 是"跨工具单点分发"，胜在格式中立与生成器路线；awesome-copilot 是"官方平台商店"，胜在分发渠道与治理深度。规范正在趋同：本仓 `plugin.json` 使用 agent-plugins.org schema，`CONTRIBUTING.md` 明确引用 Claude Code 插件市场 spec，各家插件规范收敛为同一标准；第三方集合星数与本仓相当，证明该生态足以支撑多个头部仓库并存。

## 9. 亮点与风险

1. 官方背书 + 产品内分发渠道：Copilot CLI 预注册市场，一条命令安装（`README.md`），兼有 aka.ms 安装徽章直达 VS Code。
2. 工业化审核管线：机器校验（schema/资产/重复/拼写）+ 分域人工评审 + 六个月强制复审 + 供应链 sha pin，是目前所有提示词集合中最完整的治理设计（`CONTRIBUTING.md` + `.github/workflows/`）。
3. 全自动视图生成：人工只写内容本体，README、分类页、官网数据、marketplace.json、llms.txt 全部由 5,391 行工具链产出，规模扩大不增加文档负担。
4. 覆盖六类资源并允许组合：agent/instruction/skill/hook/workflow/extension 可打包进插件，形成可分发的"能力包"而非孤立提示词。
5. 自举示范：仓库自身运行在 Copilot agent + gh-aw 上，既是集合也是官方最佳实践展示。

**风险/局限**

1. 内容质量本质不可自动验证：提示词有效性取决于具体模型版本与场景，机器校验只保结构与安全底线；README 亦警示用户"安装前自行检查"（`README.md` Contributing 节）。
2. 插件携带可执行资产：skills 内含 py/mjs/sh/ps1 脚本（目录统计 67 py/22 mjs/20 sh/18 ps1），外部插件虽有 sha pin，第三方仓库本体仍是信任面。
3. 运维成本高：60+ workflow、夜间复审、网站多语言，强依赖 GitHub Actions 配额与员工精力；单人/小团队不可复制。
4. 平台绑定深：`.agent.md`、chatmode、Copilot CLI 插件格式均为 GitHub 特有，跨工具复用需自行转换（wshobson/agents 的生成器路线正是针对此缺口的对照方案）。

## 10. 总结评价与使用建议

awesome-copilot 是"官方运营 + 工业化治理"的社区集合范式样本：它把一个 2025 年 6 月的提示词清单，做成了带供应链锁定的插件市场、带复审闭环的内容平台、带官网与多语言教学的生态枢纽。18 个月 41k 星背后，官方产品入口导流是第一推动力，但让 463 个 skills、230 个 agents、113+ 插件保持可用且可发现的，是那条从脚手架、校验、生成到复审的自动化流水线——对想运营大规模内容集合的团队，这个仓库的工具链与流程本身就是最好的源码读物。适合三类人：想深度定制 Copilot 的工程师（直接用）、想研究 AI 内容生态治理的运营者（读 CONTRIBUTING 与 `.github/workflows/`）、想给自家工具做内容市场的平台开发者（读 `plugins/external.json` 的 pin 设计与 marketplace 生成逻辑）。研究其提示词写法则建议直接读 `agents/azure-principal-architect.agent.md` 这类高流量条目——frontmatter 的 tools 声明与"WAF 五支柱逐项评估"式结构化指令是可迁移的写法。
