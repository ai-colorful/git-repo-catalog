# PatrickJS/awesome-cursorrules 项目分析

> 星数 41500（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：Cursor AI 编辑器的规则文件精选清单（awesome-list），从早期的 `.cursorrules` 单文件提示词集合演进为 Cursor 官方 Project Rules（`.cursor/rules/*.mdc`）格式的按框架分类规则库。

| 项目 | 值 | 出处 |
|---|---|---|
| 名称 | PatrickJS/awesome-cursorrules | GitHub 元数据 |
| 星数 | 41500（快照）；2026-11-14 实查 41158 | 任务参数 / `gh api repos/PatrickJS/awesome-cursorrules` |
| 创建时间 | 2024-09-16 | gh api `created_at` |
| 最近 push | 2026-05-30 | gh api `pushed_at` |
| 主语言 | JavaScript（内容主体为 Markdown/.mdc） | 任务参数 |
| 许可证 | CC0-1.0 | `src/LICENSE` |
| forks / open issues / 贡献者 | 3543 / 121 / 86 | gh api / contributors 分页 Link 头 |
| 内容规模 | 13 个分类、README 收录 214 条规则、`rules/` 下 257 个 .mdc 文件（约 1.0MB 文本） | `src/README.md`、`src/rules/` 实测 |

## 2. 项目是什么，解决什么问题

Cursor 是 AI 优先的代码编辑器，其生成质量高度依赖"项目上下文注入"：同样的模型，告诉他"这是 Next.js 15 + Tailwind 项目、组件用函数式、不要 any"之后，产出会明显更贴合项目。本仓库收录的就是这类"给 AI 的项目规则"：按框架/语言/场景组织、复制即用的规则文件（`src/README.md` "Why Cursor Rules" 一节）。

仓库创建于 2024-09，正值 `.cursorrules`（项目根目录单文件、全局生效）流行的窗口期，两年内涨到 41k 星。随着 Cursor 官方把规则机制升级为 Project Rules（`.cursor/rules/` 目录下带 frontmatter 元数据的 `.mdc` 文件，可按 glob 作用域自动挂载），本仓库已完成整体迁移：README 开头明确声明 "This repository uses the modern `.mdc` Project Rules format"（`src/README.md` 第 4 段），贡献指南要求新规则必须是带 `description/globs/alwaysApply` frontmatter 的 `.mdc`（`src/contributing.md`），还提供了 `scripts/cleanup-legacy-artifacts.mjs` 专门清理 `rules-legacy`、`rules-new`、`.cursorrules` 等迁移残留。可见"仓库是否跟上 Cursor 从 .cursorrules 转向 Rules 的演进"这一问题的答案是：跟上了，且以官方新格式为唯一准入格式。

使用方式极简：在 README 的 13 个分类里挑规则，把对应 `.mdc` 复制进项目的 `.cursor/rules/` 目录即可（`src/README.md` "How to Use"）。每个条目链接指向仓库内 `rules/*.mdc` 文件本体，而非外站，规则内容随仓库分发、可离线取用。

生态位上，它是 awesome 系清单中"AI 编码规则"这一细分品类的头部沉淀。README "Directories" 一节同时指向 CursorList 与 CursorDirectory 两个姊妹目录站（`src/README.md`），后者承担在线浏览/搜索分发，本仓库则承担内容本体与 GitHub 社区协作，形成"清单仓库 + 目录站"的组合。

## 3. 技术栈与依赖

- 内容主体：Markdown（README 主清单，367 行 / 51KB）+ 257 个 `.mdc` 规则文件（Cursor Project Rules 格式，YAML frontmatter + Markdown 正文），合计约 1.0MB 文本，平均单个规则约 4KB（`src/rules/` 实测）。
- 工程化栈：Node.js 20 + pnpm 10（CI 中 `corepack prepare pnpm@10.20.0 --activate`，`src/.github/workflows/main.yml`），全部质检脚本为纯 Node ESM 脚本（`scripts/*.mjs`），测试用内置 `node --test`（`src/package.json`）。
- 运行时第三方依赖：无。唯一的"外部"依赖是 awesome-lint，通过 `pnpm dlx awesome-lint@2.3.0` 按需拉取，不进 lockfile（`src/package.json` 的 `check:awesome-list:upstream` 脚本）。
- 语言归类为 JavaScript 的来源即这批 Node 脚本与测试，仓库本身不是可运行的应用。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `README.md` | 主清单：分类目录 + 214 条规则链接 + 使用说明 + 赞助商 |
| `rules/` | 257 个 `.mdc` 规则文件本体（清单的唯一内容源） |
| `scripts/` | 10 个质检/维护脚本 + 9 个对应测试文件（合计 1780 行测试代码） |
| `.github/workflows/main.yml` | CI，7 个 job：PR 信任门槛、README/rule 卫生、issue 模板策略、仓库安全、测试、awesome-lint |
| `.github/ISSUE_TEMPLATE/` | 规则请求模板（`rule_request.yml`）与仓库维护模板 |
| `.github/CODEOWNERS` | workflows、scripts、CODEOWNERS 强制 @PatrickJS 审查 |
| `LICENSE` / `code-of-conduct.md` / `contributing.md` / `pull_request_template.md` | 标准社区文件 |
| `*.png` | Cursor/赞助商 logo（CodeRabbit、Unblocked、Warp） |

组织方式是"内容型 monolith"：单 README + 平铺的 `rules/` 目录，靠文件命名 slug（如 `nextjs15-react19-vercelai-tailwind-cursorrules-prompt-file.mdc`）区分，无子目录分类。配套一套在 awesome 清单中罕见重的自动化质检体系（`scripts/check-repo-hygiene.mjs` 单文件 919 行）。

## 5. 核心架构与关键实现

### 5.1 清单组织与规模统计

13 个内容分类（`src/README.md` "Contents" 与正文 `###` 小节实测条目数）：前端 44、后端/全栈 43、语言专项 40、构建工具 26、测试 17、CSS 10、移动端 9、状态管理 7、游戏图形 6、文档 5、数据库/API 5、部署托管 3、安全 1，共 214 条。条目格式统一为 `- [名称](blob 链接) - 一句话描述`。分类重心明显偏 TypeScript/React 系 Web 开发（仅 Next.js 相关就 13 条），Python FastAPI（4 条）、Laravel、Rails、Go、Elixir 等各有覆盖。

值得注意的失衡：`rules/` 目录有 257 个文件，README 只引用了其中 214 个，43 个文件未挂账（实测比对），包括一批短命名新规则（`react.mdc`、`vue.mdc`、`typescript.mdc`、`python.mdc`、`go.mdc`、`rust.mdc`、`docker.mdc`、`tailwind.mdc` 等）和疑似变体（`beefreeSDK.mdc` 与已收录的 `beefreeSDK-nocode-content-editor-...mdc` 描述不同，非重复文件）。这批文件应是待补录 README 的积压。

### 5.2 `.mdc` 规则文件的通用结构模式

规则文件遵循统一的三段式模式（抽样 `rules/nextjs15-react19-vercelai-tailwind-cursorrules-prompt-file.mdc`、`rules/angular-typescript-cursorrules-prompt-file.mdc`、`rules/react-router-v7.mdc`）：

1. frontmatter 元数据：`description`（给 Cursor 与贡献者的一句话说明）、`globs`（自动挂载的文件模式）、`alwaysApply`（是否全量注入），三者均为 CI 强制字段（`scripts/check-repo-hygiene.mjs` 中 `requiredRuleFrontmatterFields`）；
2. 角色设定：以 "You are an expert ..." 开头声明技术栈与能力边界；
3. 规范注入：分析流程（先分析请求→拆解方案→再写码）、编码风格（函数式、早返回、DRY）、框架专属最佳实践（如 Angular 文件限制 2 层嵌套、Vue 3 强制 Composition API `<script setup>`，见 `rules/vue-claude-stack.mdc`）。

`rules/react-router-v7.mdc` 展示了 glob 作用域的正确用法：

```yaml
---
description: React Router v7 rules for framework mode, data routers, loaders, actions, route modules, and progressive enhancement
globs: ["app/routes/**/*", "src/routes/**/*", "routes/**/*", "react-router.config.*", "vite.config.*", "**/*.tsx", "**/*.ts"]
alwaysApply: false
---
```

### 5.3 质检脚本与提示词安全检查

核心是 `scripts/check-repo-hygiene.mjs`（919 行），支持 `--only readme,rules,issues,security` 按"关注面"组合复用，各 `check-*.mjs` 入口都是它的薄封装。其中提示词安全检查（L298-311）针对规则文件内容做三类扫描，在 AI 内容仓库里属于前瞻性设计：

```js
function checkPromptSafety(candidateFiles) {
  for (const file of candidateFiles) {
    ...
    const content = readFileSync(fullPath, "utf8");
    checkPromptUnicode(normalizedFile, content);           // Unicode 隐写/不可见字符注入
    checkPromptUnsafeDeveloperCommands(normalizedFile, content); // 危险开发命令
    checkPromptCredentialExfiltration(normalizedFile, content);  // 凭证外泄话术
  }
}
```

`scripts/check-awesome-list.mjs`（201 行）校验 README 条目链接必须指向本仓库 `rules/` 下的真实文件（`repo-config.mjs` 提供 blob/raw 前缀），防止清单与文件本体脱节。

## 6. 构建与运行

内容仓库无构建产物，全部命令来自 `src/package.json` 与 CI：

```bash
pnpm test                        # node --test scripts/*.test.mjs（9 个测试文件，1780 行）
pnpm run check:awesome-list      # 自研清单校验（链接/分类/命名）
pnpm run check:awesome-list:upstream  # pnpm dlx awesome-lint@2.3.0（社区标准 lint）
pnpm run check:readme-hygiene    # README 卫生
pnpm run check:rule-hygiene      # 规则文件 frontmatter 与内容卫生
pnpm run check:repo-security     # 仓库安全 + 提示词安全（check:prompt-safety 别名）
pnpm run links:relative          # README 链接格式互转（--to absolute/--to relative）
pnpm run cleanup:legacy          # 清理 .cursorrules/rules-legacy 等迁移残留
```

## 7. 质量保障与自动化检查

测试规模与风格：9 个 `node:test` 测试文件共 1780 行（`scripts/check-repo-security.test.mjs` 604 行、`check-repo-hygiene.test.mjs` 403 行最大），覆盖质检脚本自身的行为，属"清单质量工具的自测试"，在 awesome 系仓库中几乎没有同类。

CI 设计（`src/.github/workflows/main.yml`）有三个亮点：其一，`pull-request-trust` job 用 GitHub API 校验 PR 作者账号注册满 30 天才放行（阈值与白名单走 `PR_AUTHOR_MINIMUM_AGE_DAYS`/`PR_AUTHOR_AGE_ALLOWLIST` 仓库变量，默认白名单含 dependabot 与 Copilot），专门抵御刷 PR 类滥用；其二，所有针对 PR 内容跑的检查都从 base 分支 checkout 到 `.trusted-base` 目录执行脚本（"trusted-base" 模式），脚本本体永远来自可信分支，PR 只贡献被检数据，规避 `pull_request_target` 的脚本注入风险；其三，actions 与 pnpm 版本均按 commit SHA 锁定。CODEOWNERS 再兜底：改 workflows/scripts 必须过维护者。

## 8. 社区与维护状态

- 活跃度：创建于 2024-09-16，最近 push 2026-05-30（gh api）。tarball 无提交历史，按 tar 内文件 mtime（全部为 2026-05-31）与 `pushed_at` 互相印证，最近一次更新在分析日（2026-11-14）前约 5 个半月；2024-2025 为高速增长期（两年 41k 星、3.5k fork），近期节奏明显放缓（推断自 push 间隔与 121 个 open issues 的积压，未确认逐月分布）。
- 贡献者结构：86 人（contributors 分页 Link 头 last=86），典型 awesome 模式——长尾贡献者提规则 PR，维护者 PatrickJS 把关；`check-pr-author.mjs` 的账号年龄门槛说明低质量 PR 流量已大到需要设闸。
- 商业化：README 顶部赞助位为 CodeRabbit、Unblocked、Warp 三家 AI 开发工具商（`src/README.md`），`.github/FUNDING.yml` 开启 GitHub Sponsors（patrickjs）。赞助商与仓库受众（Cursor/AI 编码用户）高度重合，是内容型仓库变现的典型样本。
- Issue 治理：`rule_request.yml` 模板明确"仓库只收规则内容与维护问题，不受理 Cursor 账号/订阅/破解类求助"（`src/.github/ISSUE_TEMPLATE/rule_request.yml`）。

## 9. 亮点与风险

**亮点**

1. awesome 清单里罕见的工程化强度：919 行自研校验 + 1780 行脚本自测试 + 7 job CI + awesome-lint 双保险，清单的"214 条链接全部指向真实存在的文件"由机器保证（实测无一失效）。
2. 提示词供应链安全意识超前：对规则文件本身做 Unicode 隐写、危险命令、凭证外泄扫描，并以 trusted-base 模式隔离不可信 PR——规则文件是要被喂给 AI 的，投毒面比普通清单更敏感，这套检查直击要害。
3. 完整跟进了 Cursor 官方格式演进：`.cursorrules` → Project Rules `.mdc` 的迁移已收口，frontmatter（description/globs/alwaysApply）成为强制规范，贡献指南与清理脚本配套（`src/contributing.md`、`scripts/cleanup-legacy-artifacts.mjs`）。
4. 分类体系贴合真实开发栈：13 类 214 条覆盖前端/后端/移动/测试/安全，热门框架（Next.js、React、Vue、FastAPI）有多条不同组合的规则可选。
5. 部分条目已出现跨工具形态（如 `rules/vue-claude-stack.mdc` 同时产出 Cursor 规则 + CLAUDE.md + Copilot instructions），开始回应"一个项目喂多个 AI 工具"的现实需求。

**风险/局限**

1. 清单与文件本体不同步：43 个 `rules/*.mdc` 未列入 README（含整批短命名通用规则），收录进度滞后于文件积压。
2. 规则内容同质化与模板化：大量文件共享 "You are an expert ..." + 分析流程 + 编码风格的骨架，单条规则的增量价值参差；版本锚点（Angular 18、Next.js 15）会随框架迭代过时，时效性完全依赖社区 PR。
3. 仓库名仍是 `cursorrules` 而内容已全面转向 `.mdc`，对新人有轻微误导（历史品牌资产与命名惯性的权衡）。
4. 更新动能衰减：最近 push 距今 5 个多月、121 个 issue 积压；Cursor 官方自带 Rules/Memories 机制与 cursor.directory 等目录站分流后，静态清单的边际价值在下降（趋势推断，未确认）。

## 10. 总结评价

这个仓库是"AI 编码规则文件热潮"的完整标本：2024 年 Cursor 崛起带火了 `.cursorrules` 提示词共享，本仓库两年攒下 41k 星成为该品类的事实入口；随后官方把机制升级为 Project Rules，它又完成了一次教科书级的格式迁移，并用远超 awesome 均值的工程化手段（账号年龄门槛、trusted-base CI、提示词投毒扫描）守住内容质量。热潮的下半场也能从它身上读出：push 节奏放缓、issue 积压、43 个文件待挂账，说明当 AI 工具原生规则机制成熟、目录站承接分发后，静态清单正退居"参考资料库"。适合三类人：想给 Cursor 配项目规则的工程师直接按框架取用；想做自己的 AI 规则仓库的人把它的质检脚本与 CI 当模板；研究"提示词即依赖"生态的人把它当作规则文件结构模式的样本库（frontmatter 三件套 + 角色设定 + 规范注入）。它不是代码项目，但它是理解 2024-2026 年 AI 编码工作流沉淀方式的一手材料。
