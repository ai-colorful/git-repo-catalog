# anthropics/claude-plugins-official 项目分析

> 星数 46215（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-06

## 1. 项目速览

**Claude Code 官方插件目录（官方"应用商店"）**——Anthropic 管理的精选插件市场清单仓库，是 Claude Code `/plugin > Discover` 界面的数据源，收录 315 个插件条目并配套一整套全自动的 SHA 固定、每日跟进、AI 安全审查与校验流水线。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | anthropics/claude-plugins-official | gh api |
| 星数 | 46216（分析时点）｜forks 4807｜watchers 254 | gh api |
| 语言 | Python（GitHub 统计口径；实际以 JSON/Markdown/Python 工具脚本为主） | gh api |
| 许可证 | Apache-2.0（顶层 LICENSE） | LICENSE |
| 创建时间 | 2025-11-20，一年内涨至 46.2k 星 | gh api |
| 最近推送 | 2026-11-05（分析前一天仍在推送） | gh api |
| 默认分支 | main，tarball 共 840 个条目、7.7MB | 解包统计 |
| 贡献者 | 约 41 人（contributors API 分页 42 页，anon=true） | gh api |

## 2. 项目是什么，解决什么问题

Claude Code 的插件系统（skills/agents/commands/hooks/MCP 四类扩展加清单文件）需要一个统一的分发入口。本仓库就是这个入口：`.claude-plugin/marketplace.json`（4431 行）以标准 schema 声明每个插件的元数据与来源，用户通过 `/plugin install {name}@claude-plugins-official` 安装（README.md "Installation" 节）。它本身不实现插件运行时——运行时在 Claude Code 本体，仓库只做"目录 + 实体托管 + 质量门禁"。

仓库分两类内容：`/plugins` 是 Anthropic 内部开发维护的插件（40 个目录），`/external_plugins` 是 vendored 进来的第三方官方插件（14 个目录：github、gitlab、linear、asana、firebase、playwright、serena、telegram、terraform、context7 等，见目录实测）。其余绝大多数条目不在本仓库落盘，而是通过 `source` 字段远程引用上游 GitHub 仓库并**固定到具体 commit SHA**——这使目录成为一个"供应链账本"：每个外部插件的每个版本都有精确锚点。

收录是审核制的。README "Contributing" 节明确：外部插件必须满足质量与安全标准，新插件只能通过 `clau.de/plugin-directory-submission` 表单提交；`close-external-prs.yml` 会自动关闭非成员的普通 PR（唯一例外是"给已收录插件补 marketplace.json 条目"这类 in-scope 贡献，由 `external-pr-scope.js` 判定）。也就是说社区不能直接 PR 进目录，PR 通道只服务于 Anthropic 成员和自动化 bump。

星数一年冲到 46.2k 的原因不在代码量（工具脚本合计不到 900 行），而在于生态位：它是 Claude Code 用户的默认插件发现页，几乎所有活跃用户都会间接"访问"它；加上大厂云集的目录本身（AWS、Google、Microsoft、GitHub、Atlassian、Datadog、Cloudflare、Databricks、SAP、Adobe、AMD 等 233/315 条目有 author 字段）具备很强的"官方背书清单"传播属性。

## 3. 技术栈与依赖

- **无传统构建体系**：没有 package.json/pyproject.toml，无第三方运行时依赖。仓库本体是 JSON 清单 + Markdown 插件内容 + 少量脚本。
- **Python**（GitHub 语言统计来源）：`.github/scripts/discover_bumps.py`（229 行，仅标准库）驱动每日 SHA 跟进；`plugins/claude-security` 内嵌一套 Python 脚本库（`scripts/lib/` 下 finding/cwe/sarif/git/secret/strictjson 等 15 个模块）作为插件自身功能。
- **TypeScript/JavaScript**：`validate-frontmatter.ts`（277 行）校验插件 frontmatter，`external-pr-scope.js`（156 行）在外部 PR 管道中判定贡献范围，`plugins/claude-security/workflows/scan.js` 是插件内编排脚本。
- **GitHub Actions**：9 个工作流（`.github/workflows/`），配合 `claude plugin validate` CLI 与 Claude AI 安全审查。
- 插件生态本身的"依赖"是 Claude Code 的 marketplace schema（`$schema: https://anthropic.com/claude-code/marketplace.schema.json`，marketplace.json 第 2 行）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `.claude-plugin/marketplace.json` | 核心：315 个插件条目的市场清单（name/description/author/category/source/homepage 等），含 10 条 `renames` 改名迁移映射 |
| `plugins/` | Anthropic 内部插件实体，40 个目录（12 个 LSP 插件 + code-review、skill-creator、plugin-dev、hookify、claude-security 等） |
| `external_plugins/` | vendored 第三方插件，14 个目录，通常只有 plugin.json + .mcp.json (+commands) |
| `.github/scripts/` | discover_bumps.py、validate-frontmatter.ts、external-pr-scope.js 三个工具脚本 |
| `.github/workflows/` | 9 个 CI 工作流：bump/scan/validate/mcp-url/license/external-pr 管道 |
| `.github/policy/` | AI 安全审查的 prompt.md（140 行）与 verdict schema.json（52 行） |
| `.github/bump-tracking.json` | 14 个"只跟踪 release"的插件名单（releases-only） |
| `README.md` / `LICENSE` | 目录说明、Apache-2.0 |

组织方式是"**单清单 + 薄实体 + 远程引用**"的混合体：内部/vendored 插件直接落盘（53 条 `source` 为本地字符串路径），外部插件 163 条整仓 `url` 引用、99 条 `git-subdir` 子目录引用，全部带 SHA pin。这不是 monorepo，更像"账本仓库"——真正的插件代码分散在 260+ 个上游 GitHub 仓库中（source 域名统计：github.com 262 条）。

插件条目内部的组件形态（对 54 个本地目录实测统计）：最常见的组合是 mcp+manifest（10 个，纯 MCP 服务器桥接型，如 external_plugins/github）、skills+manifest（7 个）、commands+manifest（4 个）；12 个 LSP 插件目录只有 LICENSE+README，真实定义以 `strict:false` + `lspServers` 字段内联在 marketplace.json 中（如 `.claude-plugin/marketplace.json` 的 typescript-lsp 条目，声明 `typescript-language-server` 命令与扩展名映射）。

## 5. 核心架构与关键实现

### 5.1 marketplace.json：目录账本与三种来源模式

清单 schema 支持三种 `source`：本地路径（`"./plugins/code-review"`）、远程整仓（`{"source":"url","url":...,"sha":...}`）、远程子目录（`{"source":"git-subdir","url":...,"path":...,"ref":...,"sha":...}`）。另有两个重要变体（README "Skill-bundle plugins" / "Plugin names are immutable" 节）：

- **skill-bundle**（`strict:false` + `skills` 数组）：上游仓库只有 SKILL.md 没有 plugin.json 时，清单直接声明 skills 路径（相对 source.path），注册为 `<plugin-name>:<skill-name>`。amd-skills、box 等 14 个条目使用此模式。
- **renames 迁移**：`name` 是不可变 slug，改名会使用户安装报 `plugin-not-found`；顶层 `"renames": {"old": "new"}` 让加载器在下次同步时透明改写。现存 10 条记录（如 `"vals": "valtown"`、`"wordpress.com": "build-with-wordpress"`）。

### 5.2 discover_bumps.py：每日 SHA 跟进算法（`.github/scripts/discover_bumps.py`）

遍历所有带 `sha` 的条目，用 `gh api` 查上游最新 commit（子目录来源用 `commits?path=` 过滤），找出"上游已前进"的条目。两个值得注意的设计：

```python
# Oldest-pinned first so nothing starves under the cap.
bumps.sort(key=lambda b: -b["age_days"])
emitted = bumps[: args.max]
```

- **防饥饿轮换**：每晚最多 bump 60 个（`bump-plugin-shas.yml` 注释记载 2026-09 从 30 提到 60，因约 290 个外部条目、每天约 15 个上游移动，30 的上限导致晚字母条目数天得不到更新），按"固定时间最久"优先排序。
- **死上游处理**：404 与 force-push（"No commit found for SHA"）统一视为条目死亡，汇总上报而不中断。

### 5.3 AI 安全审查管道（`.github/workflows/scan-plugins.yml` + `.github/policy/prompt.md`）

这是仓库最有特色的部分：required check `scan` 把变更插件的全量文件交给 Claude，按 Anthropic Software Directory Policy 和 AUP 审查。prompt.md 的审查标准相当细：恶意代码、提示注入（包括 skill 文本里"ignore other instructions"类胁迫指令）、**跨服务凭证外传**（判定规则：看凭证属于哪个服务而非发往哪——读 `ANTHROPIC_AUTH_TOKEN` 发往非 Anthropic 端点即违规）、宽域 hook（无项目相关性门槛的 UserPromptSubmit/PreToolUse hook 直接 fail）、未披露遥测（默认开启且未在 README 披露 opt-out 即 fail）。工程上配套：verdict 按 `(plugin, sha)` 缓存 30 天、以 policy hash 为缓存键（改 prompt 即全量重扫）；`revert-failed-bumps.yml` 在扫描失败时自动把失败条目从 bump PR 摘除回滚，"一个坏上游不能阻塞其他 bump"。

### 5.4 内部插件纵深样本：claude-security（`plugins/claude-security/`）

最复杂的内部插件（v0.12.0），展示了一个"产品级"插件的全貌：8 个 agent 定义（`agents/` 下 scan-loader、scan-researcher、scan-redactor、scan-verifier、patch-generator、patch-verifier 等多智能体分工）、hooks（`hooks/hooks.json` + py/sh 双实现）、一个 Python 工具库（`scripts/lib/`：SARIF 报告、CWE 分类、finding 模型、git/secret 处理）、skill 定义（`skills/claude-security/SKILL.md` + jobs/ + specs/）。与之对照，`plugins/example-plugin` 是官方参考实现（manifest + .mcp.json + 1 个 command + 2 个 skill，共 7 个文件）。

## 6. 构建与运行

本仓库无构建步骤。实际可用的操作（均出自项目文档/工作流）：

```bash
# 安装插件（用户侧，Claude Code 内）
/plugin install {plugin-name}@claude-plugins-official
/plugin   # 或在 Discover 界面浏览

# 每日 SHA 跟进（维护侧，discover_bumps.py 顶部用法）
python .github/scripts/discover_bumps.py [--plugin NAME] [--max N] [--dry-run]

# 插件校验（CI 中使用，validate-plugins.yml）
claude plugin validate
```

CI 侧的检查矩阵：`validate`（manifest 校验）、`scan`（AI 安全审查）、`check`（MCP URL 存活，仅限本地 vendored 插件）、license 校验、frontmatter 校验（`.github/workflows/` 各文件头注释）。

## 7. 测试与工程质量

- 无单元测试套件（目录型仓库，测试主体是 CI 校验管道）。校验逻辑集中在 `validate-frontmatter.ts`（277 行）与 9 个工作流。
- 工程质量观察：`.github/workflows/` 内注释质量极高，多处记录真实踩坑——`validate-plugins.yml` 详述 glob `*` 不跨 `/` 导致 vendored 插件路径漏匹配（PR #5416）、GITHUB_TOKEN 触发的 PR 不跑 `on:pull_request` 的递归防护需要手动 dispatch required check、policy-only PR 会让 required check 永远挂起的路径补偿等。这些注释本身是 GitHub Actions 工程实践的好教材。
- 安全设计成熟：审查缓存防重复烧钱、并发控制（scan 按 ref 串行化）、失败条目自动隔离回滚、外部 PR 面收窄到"只允许补条目"。

## 8. 社区与维护状态

- **极度活跃且高度自动化**：最近 5 个 commit 全部是 nightly bump PR（`bump(postman) #7010`、`bump(zapier) #6873` 等，gh api commits）；PR 编号已到 7000+，一年内平均每天约 19 个 PR。
- **贡献者约 41 人**（含自动化 bot），核心是 Anthropic 团队；外部贡献被结构性限制（PR 自动关闭 + 表单制收录），这是产品决策而非疏于维护。
- open_issues 1282（含 PR）。无 release/tag 发布节奏——main 分支即生产目录，由 Claude Code 客户端直接拉取。
- 每日推送由 `bump-plugin-shas.yml` 的 cron（07:23 UTC）驱动，人工介入体现在收录审核与 bump PR 的合并。

## 9. 亮点与风险

**亮点**
1. **开源的官方插件供应链流水线**：SHA pin → 每日跟进 → AI 安全审查 → 失败自动回滚 → per-entry bump PR 隔离故障，闭环完整，可直接借鉴到任何"聚合第三方内容"的场景。
2. **`policy/prompt.md` 是一份高质量 AI 安全审查提示词范本**：从恶意代码到跨服务凭证外传判定、宽域 hook、未披露遥测，规则具体到可执行（`.github/policy/prompt.md`）。
3. **插件规范设计清晰**：manifest schema、skill-bundle 兜底、LSP 内联声明、renames 不可变 slug 迁移机制，覆盖了目录运营的真实边角（README + marketplace.json）。
4. **CI 注释即文档**：踩坑记录（glob 跨目录、required check 挂起、GITHUB_TOKEN 递归）对维护类似仓库的人有直接价值。

**风险/局限**
1. **信任模型依赖快照审查**：外部插件按 SHA pin，bump 时只做 `claude plugin validate` + AI 扫描；且审查管道自己也承认"git 来源会 clone 整个仓库到用户磁盘"（policy/prompt.md 第 17-24 行），上游非加载面的隐藏代码仍在攻击面上。
2. **星数与代码价值错位**：46k 星来自生态入口效应，本体工具代码约 900 行；把它当"代码项目"评估会失真。
3. **社区贡献通道收窄**：新收录只能走表单，PR 自动关闭机制对 fork 驱动的贡献者不友好（`close-external-prs.yml`）。
4. **单一清单文件扩展性**：315 条、4431 行的 marketplace.json 是单点，全部 bump/校验/审查围绕它串行轮换（cap 60/晚）。

## 10. 总结评价

这是"AI 编辑器官方插件市场"的运营中枢：表面是一份 JSON 目录，实质是一套用 GitHub Actions + Claude 自身搭起来的第三方插件供应链治理系统——收录靠表单与成员制，更新靠每日 SHA 跟进，安全靠开源的 AI 审查提示词，故障靠自动回滚隔离。它不适合"读代码学架构"，但非常适合三类人：(1) 想给自己的 AI 工具做官方插件市场的平台团队（抄它的流水线与 policy prompt）；(2) 想发布 Claude Code 插件的开发者（照 example-plugin 的结构与收录标准准备）；(3) 研究智能体安全边界的人（prompt.md 对凭证外传、hook 宽域、提示注入的判定规则是少见的实操级材料）。一年 46.2k 星是 Claude Code 生态爆发在 GitHub 上的直接投影。
