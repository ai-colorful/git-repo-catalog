# coreyhaines31/marketingskills 项目分析

> 星数 56182（快照 2026-10-24）｜主语言 JavaScript｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：为 AI 编程代理（Claude Code、Codex、Cursor 等）提供的**营销领域 Agent Skills 库**——53 个结构化 Markdown 技能包覆盖 CRO、文案、SEO、投放、增长全链路，配套 66 个零依赖 CLI 工具与完整的版本化发布工程。

| 项 | 值 |
|---|---|
| 名称 | coreyhaines31/marketingskills |
| 星数 | 56,182（快照）；分析时 58,068（gh api，2026-11-03） |
| 主语言 | JavaScript（实际主体为 Markdown 技能内容 + Node.js CLI 工具） |
| 许可证 | MIT（`LICENSE`，gh api 确认） |
| 创建时间 | 2026-01-15（gh api `created_at`） |
| 最近 push | 2026-10-31（gh api `pushed_at`） |
| Forks / Open issues | 8,533 / 73（gh api） |
| 当前仓库版本 | v2.14.0（`.claude-plugin/plugin.json`） |
| 主页 | https://marketing-skills.com |

## 2. 项目是什么，解决什么问题

这是 Corey Haines（SaaS 营销社区 Swipe Files 创始人，`README.md` 自述）开源的"营销技能包"集合，遵循 Agent Skills 规范（agentskills.io，见 `AGENTS.md`）。每个技能是一个带 YAML frontmatter 的 `SKILL.md`：frontmatter 的 `name` + `description`（含大量触发短语）让 AI 代理在对话中自动识别营销任务并加载对应知识，正文则是角色设定 + 分析框架 + 工作流指令。例如 `skills/cro/SKILL.md` 定义了从价值主张清晰度、标题有效性、CTA 层级到视觉层次的完整 CRO 分析框架。

53 个技能覆盖营销全流程（`skills/` 目录，53 个 `SKILL.md`）：SEO 与内容（seo-audit、ai-seo、programmatic-seo、schema）、CRO（cro、signup、onboarding、popups、paywalls）、内容与文案（copywriting、cold-email、emails、social、video）、投放与度量（ads、ad-creative、ab-testing、analytics）、增长与留存（referrals、churn-prevention、lead-magnets）等。技能间有显式依赖设计：`product-marketing` 是基石技能，负责生成 `.agents/product-marketing.md` 产品上下文文档，其余所有技能先读它再动手（`README.md` 架构图、`skills/product-marketing/SKILL.md` Step 1）。特色技能如 `marketing-council`（模拟 12 位知名营销人的顾问团辩论，`skills/marketing-council/SKILL.md`）和 v2.14 新增的 `reverse-engineer-marketing`（九阶段竞品增长拆解，`VERSIONS.md` 2026-10-30 条目）。

除技能内容外，仓库还解决"技能如何落地到真实数据"的问题：`tools/clis/` 提供 66 个零依赖 Node.js CLI（GA4、Ahrefs、Semrush、Mailchimp、Meta Ads 等营销 API 的命令行封装），`tools/integrations/` 提供 110 篇工具接入指南，`tools/REGISTRY.md` 按类别索引，`tools/SETUP.md` 定义了让代理引导用户安全连接营销账号（只读起步、凭据不进聊天）的工作流。

商业模式值得注意：库本身免费 MIT，但通过 "Verified Partners" 赞助计划变现——付费赞助的工具集成被标记 ◆ 并在 `tools/PARTNERS.md` 中明确治理边界（赞助不影响技能推荐中立性），`partners.json` 是唯一数据源，由 `scripts/sync-partners.mjs` 同步到 README 与 REGISTRY。同时 README 导流到作者的 Conversion Factory、Swipe Files、AI Marketing Training 等付费产品。

爆发式增长背景：仓库 2026-01-15 创建，不到 10 个月破 56k 星。它精准踩中了 "Agent Skills" 生态爆发期——通过 `npx skills add`（vercel-labs CLI）、Claude Code 插件市场（`.claude-plugin/marketplace.json`）、Codex 插件（`.codex-plugin/`）、Claude Web/Perplexity ZIP 等 9 种安装方式全渠道分发（`README.md` Installation 章节），加上作者在 SaaS 营销圈的自带流量。

## 3. 技术栈与依赖

- **核心内容**：Markdown（约 5.2MB，`skills/` 目录）——53 个 SKILL.md 共 16,757 行，含 references 后总计约 57,814 行营销知识文本（实测 `wc -l`）。
- **工具链**：Node.js 18+，零第三方依赖（`AGENTS.md` 明确要求 CLI 工具零依赖，仅用内置 `fetch`；`tools/clis/ga4.js` 实测只用原生 API）。
- **打包脚本**：Python 3（`scripts/package_skills.py`，标准库实现确定性 ZIP 打包）。
- **工程脚本**：Node ESM（`scripts/check-versions.mjs`、`scripts/sync-partners.mjs`）。
- **测试**：Node 内置 test runner（`node --test tests/clis/*.test.cjs`）、Bash 回归脚本、Python unittest。
- **CI**：GitHub Actions × 6（`.github/workflows/`）。
- 关键点：整个仓库无 `package.json`、无 npm 依赖，刻意保持"内容仓库 + 零依赖工具"形态，用户 clone 即用。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `skills/<name>/` | 53 个技能包：SKILL.md（必需）+ references/*.md（282 个深读文档）+ evals/evals.json（469 条评估用例）+ assets（63 个模板） |
| `tools/clis/` | 66 个零依赖营销 API CLI（ga4.js、ahrefs.js、semrush.js 等） |
| `tools/integrations/` | 110 篇第三方工具接入指南（含赞助伙伴披露头） |
| `tools/REGISTRY.md` / `PARTNERS.md` / `SETUP.md` | 工具索引 / 赞助治理 / 凭据连接工作流 |
| `scripts/` | 版本一致性检查、伙伴同步、多平台 ZIP 打包 |
| `tests/` | 147 个 CLI 契约测试 + 验证器回归 fixtures + 打包测试 |
| `.claude-plugin/`、`.codex-plugin/`、`.agents/plugins/` | 三大代理平台的插件/marketplace 清单 |
| `.github/` | 6 个 CI 工作流、PR 模板（新技能/改技能/改文档三类）、issue 模板 |
| `VERSIONS.md` | 158KB：全部技能版本表 + 99 个版本块变更日志 |
| `validate-skills.sh` | 技能规范校验器（frontmatter、命名、描述长度） |

组织方式评述：单体内容仓库（非 monorepo、无构建步骤），"内容为产品、脚本为保障"。技能目录结构高度统一（SKILL.md + references + evals），靠 CI 脚本而非框架强制约束。规范要求 references 下每个文件必须被 SKILL.md 或其他 reference 链接（无孤儿文件，`AGENTS.md` Versioning 节）。

## 5. 核心架构与关键实现

**（1）技能即提示词的三层知识架构**。每个技能是"SKILL.md 主指令（代理按需加载）→ references/ 深读文档（按路由表加载）→ evals/ 评估基线"。以 `skills/cro/SKILL.md` 的 Reference Routing 表为例：表单弃单加载 `references/form.md`，实验建议加载 `references/experiments.md`。这种渐进式加载控制了上下文窗口消耗，是 Agent Skills 规范的标准实践，本仓库是其中内容最重的实现之一。

**（2）双层版本体系与 CI 强制校验**（`scripts/check-versions.mjs`）。仓库版本（plugin.json = marketplace.json = codex plugin.json = VERSIONS.md 最新 `### x.y.z` 块，四者必须一致）+ 每技能 `metadata.version`。`--base origin/main` 模式下：改了技能文件但没 bump 版本、VERSIONS.md 表与 SKILL.md 不一致、references 存在孤儿文件，CI 直接失败。这是内容仓库少见的"把文档当代码管"的设计：

```js
// scripts/check-versions.mjs
const skillVersion = (text) => frontmatter(text)
  .match(/^metadata:\s*\n(?:[ \t]+.*\n)*?[ \t]+version:\s*["']?([\d.]+)/m)?.[1];
const newer = (a, b) => {
  const [x, y] = [a, b].map((v) => v.split(".").map(Number));
  for (let i = 0; i < 3; i++) if (x[i] !== y[i]) return x[i] > y[i];
  return false;
};
```

**（3）多平台确定性发布流水线**（`.github/workflows/release.yml` + `scripts/package_skills.py`）。监测到 plugin.json 版本变更即自动打 tag、建 GitHub Release，发布说明直接从 VERSIONS.md 对应版本块提取。打包脚本从冻结的 git revision 构建单一技能 ZIP，为不同宿主生成变体：Claude Web 版替换为经过人工审核的短描述（`scripts/claude-web-descriptions.json`），Perplexity 版要求 SKILL.md 位于压缩包根、≤100 文件、≤20MB（脚本内 `PERPLEXITY_MAX_*` 常量，注释引用官方帮助文档并注明核查日期 2026-10-27），并计算 sha256。

**（4）零依赖 CLI 的契约测试**。66 个 CLI 各配 mock fetch 的契约测试（`tests/clis/` 147 个 `.test.cjs`），CI 中 `node --test` 运行（`.github/workflows/regression-tests.yml` cli-contract-tests job）。CLI 自带 `--dry-run` 模式预览请求不发真实调用（`tools/clis/ga4.js` 的 `args['dry-run']` 分支），配合 `node --check` 语法检查构成轻量质量闭环。

## 6. 构建与运行

本仓库无构建步骤（内容即产品，`AGENTS.md`："Skills are content-only (no build step)"）。来自 README/AGENTS.md 的实际可用命令：

```bash
# 安装（九选一，最常用三种）
npx skills add coreyhaines31/marketingskills            # CLI 全量安装
npx skills add coreyhaines31/marketingskills --skill cro copywriting  # 按技能安装
claude plugin marketplace add coreyhaines31/marketingskills   # Claude Code 插件
claude plugin install marketing-skills@marketingskills

# 贡献/维护
node scripts/check-versions.mjs --base origin/main   # 版本一致性（CI 同款）
node scripts/sync-partners.mjs                       # partners.json → README/REGISTRY
python3 scripts/package_skills.py ...                # 打包发布 ZIP
bash validate-skills.sh                              # 技能规范校验

# 测试
node --test tests/clis/*.test.cjs     # CLI 契约测试（mock fetch，无真实调用）
bash tests/run-regression.sh          # 校验器回归测试
node --check tools/clis/<name>.js     # CLI 语法检查
```

## 7. 测试与工程质量

- **测试规模**：tests/ 下 147 个测试文件（实测 `find tests -type f | wc -l`），以 CLI 契约测试为主；另有 validate-skills.sh 回归 fixtures（`tests/fixtures/invalid/`，注释标明源自 issue #450/PR #451 的假阴性修复）与 `tests/test_skill_packaging.py` 打包测试。
- **469 条技能评估用例**：53 个技能全部带 `evals/evals.json`（实测汇总），每条含 prompt、expected_output 与逐条 assertions（如 cro 的"Checks for product-marketing.md"），构成人工评估代理行为的基准——这是对"提示词内容"做质量保障的少见认真做法。
- **CI**：6 个工作流覆盖技能校验（按变更技能增量触发）、版本一致性、回归测试、自动 release、打包、伙伴同步；PR 模板区分新技能/技能更新/文档三类（`.github/PULL_REQUEST_TEMPLATE/`）。
- **质量观察**：AGENTS.md 写得极其详尽（版本规则、命名约束、spec 逐字段表格），VERSIONS.md 维护着 99 个版本块的完整变更记录（2026-01-27 至 2026-10-30，约每周 2+ 个版本），工程纪律在内容型仓库中属顶级水平。

## 8. 社区与维护状态

- **活跃度**：极高。gh api 显示最近 push 2026-10-31（分析前 3 天）；VERSIONS.md 最近一个版本 2.14.0 发布于 2026-10-30，多数技能的最后更新日期在 2026-10 月内（VERSIONS.md 版本表）。
- **节奏**：9 个月 99 个版本块，从 v1.x 到 v2.14.0，其中 v2.0 做过 17 个技能重命名与 page-cro/form-cro 合并（README 迁移指南）。
- **社区设施**：issue 模板含专门的 skill-request 表单（`.github/ISSUE_TEMPLATE/skill-request.yml`）、FUNDING.yml、分类 PR 模板；回归测试注释中出现 issue #450，说明外部 PR 在参与修复。
- **贡献者结构**：核心显然是作者本人主导（README 第一人称、partner 计划），外部贡献规模未确认（tarball 无 git 历史，未统计 contributors）。
- **商业化**：Verified Partners 赞助计划已有 4 个活跃伙伴（`partners.json`：Converly、Ploy、AIsa、Omnia），有正式治理文档 `tools/PARTNERS.md`。

## 9. 亮点与风险

**亮点**
1. 把"营销专业知识"工程化为可版本化、可测试、可分发的 Agent Skills，是"内容即代码"的标杆实践（双层版本 + CI 强制 + 469 条 evals）。
2. 渠道覆盖最全的 Skills 发行：npx CLI、Claude Code/Codex 插件市场、Claude Web、Perplexity 各自的打包变体一次流水线产出（`scripts/package_skills.py`）。
3. 技能间有真实的架构设计：product-marketing 上下文前置 + 技能互相引用路由（README 架构图），而非孤立的提示词堆砌。
4. 商业模式透明：赞助伙伴有治理文档与披露机制，中立性承诺写进规则（`tools/PARTNERS.md`）。
5. 配套 66 个零依赖 CLI + 110 篇接入指南，让技能能触达真实营销数据而非空谈。

**风险/局限**
1. 主体是英文营销知识文本，内容质量依赖作者个人经验与判断，多数框架性建议（如 CRO 清单）无法被自动测试验证，evals 的 assertions 也只能靠人工评估。
2. 强绑定 Agent Skills 生态——若规范或主流代理的技能机制更迭（仓库已历经 v1→v2 破坏性迁移），维护成本会转嫁给用户（README 的 v2 升级清理步骤较繁琐）。
3. 66 个 CLI 各自直连第三方 API，token 传递、端点变更等维护面很大；测试虽为契约测试但无法覆盖上游 API 的实际行为变化。
4. 无 package.json、无统一 lint，Python/Node/Bash 三种脚本混合，对纯内容贡献者有隐性门槛（AGENTS.md 需通读才能合规提交）。

## 10. 总结评价

这是 2026 年 Agent Skills 生态爆发期最具代表性的"垂直领域技能库"：它证明了 LLM 时代的领域知识可以像软件一样被版本化、测试、发布和商业化——53 个技能、282 篇参考文档、469 条评估用例、66 个零依赖 CLI、99 个版本块的变更日志，全部由轻量 CI 纪律维系在一个零依赖的单体仓库里。不足 10 个月 56k+ 星的增长由三件事驱动：踩中 Claude Code 技能生态的时间窗口、作者 Corey Haines 在 SaaS 营销社区的既有影响力、以及九种安装方式的全渠道分发。它不适合当代码库研读（代码部分是有意的极简），但其仓库工程（版本治理、多平台打包、内容质量保障）对任何想发布提示词/技能/文档型产品的人都是值得通读的范本；对目标用户——用 Claude Code/Cursor 等代理做增长的 SaaS 创始人和技术营销人——则是即装即用的实战弹药库。
