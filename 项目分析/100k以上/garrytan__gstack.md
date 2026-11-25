# garrytan/gstack 项目分析

> 星数 137363（快照 2026-10-24）｜主语言 TypeScript｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：Y Combinator CEO Garry Tan 开源的"AI 软件工厂"——把 Claude Code（及其他编码 agent）变成一支虚拟工程团队（CEO/架构师/设计师/评审/QA/安全官/发布工程师），由约 70 个斜杠命令技能加一个自研无头浏览器守护进程组成。

| 事实 | 值 | 出处 |
|---|---|---|
| 名称 | gstack（"Garry's Stack"） | package.json |
| 星数 | 137,363｜fork 20,496 | 任务清单 + gh api |
| 主语言 | TypeScript（Bun 运行时）+ 大量 Markdown 提示词 | package.json、目录统计 |
| 许可证 | MIT（Copyright (c) 2026 Garry Tan） | LICENSE |
| 版本 | 1.91.324.0 | VERSION、package.json |
| 创建时间 | 2026-03-11（约 7 个月涨到 137k 星） | gh api created_at |
| 最近 push | 2026-10-29（分析前一天，极活跃） | gh api pushed_at |
| 贡献者 | GitHub API 仅 14 人（近乎个人 + AI agent 开发） | gh api contributors |
| 仓库规模 | 顶层 82 个目录、4370 个文件、TS 代码约 61.4 万行 | tarball 实测 |

## 2. 项目是什么，解决什么问题

gstack 的核心主张写在 README.md 第 3 行："turns Claude Code (and other coding agents) into a virtual engineering team"。它解决的问题是：直接面对空白提示框使用 AI 编码 agent 时，产出质量取决于用户会不会提问、会不会验收。gstack 把一个成熟工程组织的流程——产品思考（/office-hours）、计划评审（/plan-ceo-review、/plan-eng-review）、实现后评审（/review）、真浏览器 QA（/qa）、安全审计（/cso）、发布（/ship、/land-and-deploy）——固化成 70 个 SKILL.md 技能，让 agent 按角色与清单行事（README.md "The sprint" 一节：Think → Plan → Build → Review → Test → Ship → Reflect）。

第二个构成物是"眼睛"。AI agent 需要操作真实浏览器做 QA 与数据抓取。ARCHITECTURE.md 阐述了双引擎设计：优先驱动用户自己的 Aside 浏览器（macOS 15+，真实登录态），否则回落到 gstack 自带的常驻无头 Chromium 守护进程——持久化 tabs/cookies、命令延迟约 100-200ms，通过编译出的 CLI（`$B`）以 localhost HTTP + CDP 通信。

第三个层面是治理与安全，这在同类"提示词合集"项目里很少见：默认关闭遥测、每次离机发送写"出站回执"（README Privacy 一节 + bin/gstack-egress）、/careful//freeze 拦截危险命令、review 有"gate integrity"审计（每个被削弱的测试必须上报到人，README /review 条目）。CHANGELOG 中大量条目在讨论 token 预算校准、评审深度分级等成本与质量权衡，说明它在把"跑 agent"本身当作一个有预算、有门禁的工程系统来管理。

作者背书是项目爆发的重要原因：Garry Tan 以 YC 总裁身份公开自己日常使用的全部工作流（docs/why-gstack.md 自述"过去 60 天 3 个生产服务、40+ 特性，兼职完成"），并声称这是"我每天在用的开源软件工厂"。README 末尾还借此为 YC 招聘（ycombinator.com/software）。

## 3. 技术栈与依赖

- **运行时**：Bun ≥1.4.2（engines 字段，package.json）；Windows 额外要求 Node.js（Bun 在 Windows 有 Playwright pipe 传输 bug，README Platform notes 引用 bun#4253）。选 Bun 的理由见 ARCHITECTURE.md：`bun build --compile` 产单文件可执行（约 58MB）、内置 SQLite（直接读 Chromium cookie 库）、原生 TS、内置 HTTP 服务。
- **核心依赖**（package.json）：playwright ^1.62.1（带本地补丁 patches/playwright-core@1.62.1.patch）、@anthropic-ai/sdk（模型调用）、sharp（图像）、marked + html-to-docx（/make-pdf 的 md→PDF/docx 链路）、xterm（浏览器侧边栏终端）、@ngrok/ngrok（/pair-agent 远程隧道）、@huggingface/transformers。
- **提示词层**：纯 Markdown + YAML frontmatter（office-hours/SKILL.md 含 allowed-tools、triggers、gbrain 上下文查询等声明式配置），SKILL.md 由 SKILL.md.tmpl 经 `bun run gen:skill-docs` 生成（office-hours/SKILL.md 头部注释）。
- **类型**：TypeScript 7.0.2（devDependencies），Prettier 格式化。
- **后端碎片**：supabase/（functions + migrations，GBrain 相关配置）、extension/（Chrome 扩展，配合浏览器守护进程的 inspector/侧边栏）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| office-hours/、review/、ship/、qa/、plan-*/、design-*/ 等 70 个技能目录 | 每个斜杠命令一个目录：SKILL.md（生成物）+ SKILL.md.tmpl（源）+ sections/（共享前导、分步指令）+ specialists/、references/、templates/ |
| browse/ | 回落浏览器引擎：src/server.ts（2223 行，Bun.serve）、src/browser-manager.ts（2485 行，CDP/Playwright）、src/routes/table.ts（约 30 条路由表）、cookie 导入系列模块 |
| lib/ | 数百个 TS 支撑模块（autoplan-*、review-*、ship-budget、aside-render、banned-terms、code-intelligence/ 等），技能的"可执行后端" |
| bin/ | 约 100+ 个 `gstack-*` 独立 CLI（config、doctor、egress、review-log、ship-budget、brain-sync 等） |
| make-pdf/、design/、diagram | 独立编译二进制的文档/设计工具链 |
| test/ | 1276 个测试文件（bun test） |
| scripts/ | 构建、分片测试、eval、发布自动化（build.sh、test-free-shards.ts、test-paid-shards.ts 等） |
| .github/workflows/ | 31 个工作流：quality-gate、evals、scorecard、osv-scanner、weekly-release、windows-setup-e2e 等 |
| docs/（66 个文件） | skills.md 深入讲解、install/privacy/troubleshooting/captain 等专题 |
| 根文档 | ARCHITECTURE.md（设计决策）、ETHOS.md（构建哲学）、BROWSER.md、CLAUDE.md、AGENTS.md、SECURITY.md、CONTRIBUTING.md、CHANGELOG.md |
| extension/、supabase/、canary/、captain/、capy/、openclaw/、codex/ | 浏览器扩展、GBrain 后端、多 agent 协作（Captain 车队控制）与宿主适配 |

组织方式评述：这是一个"Markdown 技能即产品 + TypeScript 引擎做底座"的单体仓库。约 70 个技能目录彼此独立、共享 sections 前导与 lib/ 后端；无 monorepo 工作区划分，但技能与代码通过生成脚本（gen-skill-docs.ts）、manifest 审计（scripts/prompt-audit-manifest.ts）与 drift 检查（README /write-skill 条目）强约束一致性。规模上已远超"提示词合集"——TS 代码 61 万行、测试文件 1276 个。

## 5. 核心架构与关键实现

### 5.1 双引擎浏览器层（browse/ + scripts/resolvers/aside.ts）

ARCHITECTURE.md 给出核心洞察："an AI agent wants to be in *your* browser"。技能在 BROWSER SETUP 步骤先探测 Aside（`command -v aside`），READY 则走 `aside repl` 沙箱脚本（一次性会话、退出码恒 0、以 GSTACK_STEP_OK 哨兵行为准），否则回落 `$B`。回落引擎是常驻守护进程：CLI 读状态文件→POST /command 到 localhost→Bun.serve 分发→CDP 驱动 Chromium（30 分钟空闲超时）。路由表带统一鉴权门（browse/src/routes/table.ts 定义 AuthKind：root-bearer/scoped/extension-origin 等），并由 ratchet 测试钉死（文件头注释）。

```
Claude Code → $B snapshot -i → CLI → POST /command → Server(Bun.serve)
            → CDP → Chromium(headless, 持久 tabs/cookies)
```

### 5.2 技能组合与生成机制

每个技能是"YAML 声明 + 模板生成 + 分段加载"。以 office-hours/SKILL.md（683 行）为例：frontmatter 声明 allowed-tools、triggers 与 gbrain 上下文查询（按 repo_slug 拉取历史 office-hours 会话、design 文档、eureka 记录注入上下文）；正文从共享 preamble（带指纹 id `4e82fa4c839f`，"已在上下文就不重复读"——一种手写的上下文去重协议）与 sections/ 组装。review/SKILL.md 达 958 行、ship/SKILL.md 1011 行，内含 review-army（多评审员编队）、gate-integrity（测试弱化上报）等子流程，复杂度接近一个小型工作流引擎的规格说明。

### 5.3 预算与门禁系统（lib/ship-budget.ts、lib/review-depth.ts）

CHANGELOG.md（1.91.324.0，2026-10-29）记录了 /ship 的 token 预算校准：`gstack-ship-budget check` 用本次运行实测的"步进因子"（最近三次 actual/prior 最大比 × 1.5）替换先验投影，解决"29.28M 上限拒绝实际只花 0.23-1.02M 的评审"的问题；评审深度按变更行数分级（≤150 行的修复范围走 scoped pass：fix-check + Red Team + 原生对抗，否则全编队），`lib/review-depth.ts` 输出 `REVIEW_PLAN:`/`FIX_RANGE:` 机器可读行。test/ship-budget-projection.test.ts 直接回放 GSTA-2、GSTA-152 两个真实事故的数字做回归。

这种"机器可读状态行 + 技能解析"是贯穿全仓库的协议风格：技能之间不靠自由文本交接，而是约定 `KEY: value` 行（office-hours/SKILL.md 要求读取 `gstack-skill-start` 回显的 STATUS 行并据此决策）、跨技能产物落盘到 `$GSTACK_STATE_ROOT/projects/<slug>/`（README /spec 条目），使 /office-hours 的设计文档能被 /plan-ceo-review 消费、/plan-eng-review 的测试计划能被 /qa 拾取（README "The sprint"：each skill feeds into the next）。

### 5.4 多 agent 协作与记忆（captain/、/pair-agent、GBrain）

gstack 不只服务单个 agent：/captain（Crew Controller，README Power tools 表）用一个控制器跑多条"车道"，车道只发 READY 块、永不自行合并，合并由"merge trains"（stamped on call、验证后以期望 head 合并）完成，并带每日 flaky 修复；/pair-agent 把 gstack 自己的浏览器分享给任意能 curl 的 agent（scoped token、tab 隔离、速率限制、ngrok 隧道，browse/src/routes/pairing.ts）。记忆层是姊妹项目 GBrain（github.com/garrytan/gbrain）：/setup-gbrain 四条路径从零到可用（PGLite 本地 / Supabase 现有实例 / Management API 自动开新项目），frontmatter 里的 gbrain.context_queries（office-hours/SKILL.md）声明每个技能启动时注入哪些历史会话与设计文档——"learnings compound across sessions"是它区别于一次性 prompt 的关键设计（README /learn 条目）。

### 5.5 安装与多宿主适配（setup 脚本）

setup（约数千行 bash，头部有针对 macOS bash 5.2 heredoc 管道死锁的 BASH_COMPAT=50 规避）负责：构建二进制、注册技能到宿主、装 Chromium。支持 12 种宿主分级（Claude Code=full 认证级；Codex/OpenCode/Cursor/Copilot 等=experimental 仅建议级安全；slate/gbrain=instruction-only），--team 模式让仓库通过 hook 自动给团队成员装 gstack（README Step 2）。Windows 有 setup.ps1 包装与 Smart App Control 已知问题（README 引用 issue #2595：本机编译的未签名二进制被拒）。

## 6. 构建与运行

来自 package.json scripts 与 README（按规范未实际执行）：

```bash
# 安装（面向用户，README）
git clone --single-branch --depth 1 https://github.com/garrytan/gstack.git ~/.claude/skills/gstack
cd ~/.claude/skills/gstack && ./setup            # 或 ./setup --host codex / --host auto
./setup --status                                  # 每个安装一行状态 + gstack-doctor 路径

# 团队模式（README Step 2，仓库内执行）
(cd ~/.claude/skills/gstack && ./setup --team) && \
  ~/.claude/skills/gstack/bin/gstack-team-init required

# 开发（package.json）
bun install
bun run build          # bash scripts/build.sh
bun run build:gates    # bun build --compile 编译 make-pdf 与 browse 两个二进制
bun run typecheck      # tsc -p tsconfig.json
bun run test           # scripts/test-free-shards.ts（免费分片）
bun run test:windows   # --windows-only
bun run test:pr        # 付费 gate 的 PR profile（花真钱调模型）
bun run gen:skill-docs # 从 SKILL.md.tmpl 重新生成技能文档
bun run skill:check    # 技能一致性检查
bun run slop           # npx slop-scan 扫"AI slop"
bun run dev            # bun run browse/src/cli.ts（浏览器 CLI 开发模式）
bun run server         # bun run browse/src/server.ts（守护进程服务端）

# 诊断
~/.claude/skills/gstack/bin/gstack-doctor    # 不启动技能、不花钱，逐项 ok/warn/fail
```

使用入口（README Quick start）：`/office-hours` 描述要构建的东西 → `/review` 评审任意分支 → `/qa <staging-url>` 真浏览器回归 → 需要完整 sprint 时 `/autoplan` + `/ship`。

## 7. 测试与工程质量

- **规模**：test/ 目录 1276 个文件，bun test 框架（`import { describe, expect, test } from 'bun:test'`，test/ship-budget-projection.test.ts）。测试按成本分片：test:free（免费、PR 必跑）与 test:gate/test:release（付费 eval 分片，package.json 注释与 CHANGELOG 提到"full $33 gate"）。
- **测试风格**：除单元/集成外，大量"回放真实事故"测试——CHANGELOG 几乎每个条目都写明"哪个测试在哪个 commit 会失败"（如 1.91.324.0："The first 8 tests failed at ad8e12d"），以及 eval 体系（scripts/eval-bg.ts、eval:list/compare/summary/flake-rank）对技能本身做模型评测。
- **CI**：31 个工作流（.github/workflows/），含 quality-gate、evals-marathon、scorecard、osv-scanner、scorecard、windows-setup-e2e、weekly-release、cso-runtime-qualification（为 /cso 构建 linux/amd64+arm64 认证镜像）等，覆盖面在同类项目中罕见。
- **质量观察**：路由有 ratchet 测试防回退（browse/src/routes/table.ts 头注释）；slop-scan 自我扫描"AI slop"；patchedDependencies 显式钉住供应链版本。代码注释密度高、充满对边界条件的解释，工程化程度极高，但也明显是 AI 大批量产出 + 严格门禁的产物。

值得注意的工程质量细节还有：setup 脚本为 bash-3.2 兼容并显式规避 macOS bash 5.2 的 heredoc 管道死锁（setup 头部 BASH_COMPAT=50 注释）；`umask 077` 把新建文件限制为 owner-only；GBrain 同步有独立错误索引文档（docs/gbrain-sync-errors.md）；docs/troubleshooting.md 自称收录"每一条 not run / unavailable 消息"及修复命令——把排错路径本身文档化，这在开源项目里少见。

## 8. 社区与维护状态

- **活跃度**：pushed_at 2026-10-29；CHANGELOG 共 564 个版本条目，其中 280 条日期在 2026-10（一个月内），即每天多次版本号推进（CHANGELOG.md 实测）。
- **Release 节奏**：GitHub releases 显示 stable 通道约每周两签（v1.91.318.0 于 2026-10-29、v1.91.242.0 于 2026-10-26），与 README"周一/周四 tagging，间隔≥72h"一致；main 通道可跟随每次合并（docs/releases.md，README 引用）。
- **贡献者结构**：contributors API 仅 14 人、fork 2 万。结合 docs/why-gstack.md（作者自述用 gstack 本身、以 10 个并行 sprint 开发）与 ETHOS.md 的"一人 + AI = 二十人团队"叙事，这是一个"权威个人 + AI 军团"维护、社区以使用/fork 为主的项目；issue 204 个开放（gh api）。内部工单体系（GSTA-、gstack-evals# 编号）显示有私有跟踪系统，社区透出有限。
- **商业化与招聘**：README 末尾明确"无付费层、无候补名单"，同时为 YC 招聘。

## 9. 亮点与风险

**亮点**
1. 把"AI 协作流程"工程化到极致：技能不是提示词片段，而是带 frontmatter、模板生成、drift 检查、共享 preamble 去重的规格化制品（office-hours/SKILL.md、scripts/gen-skill-docs.ts）。
2. 浏览器双引擎设计务实：优先复用用户真实登录态（Aside），回落引擎解决延迟与持久化（ARCHITECTURE.md）。
3. 治理与隐私认真：遥测默认关、出站回执可审计、危险命令硬拦截、评审 gate integrity 强制人审（README、bin/gstack-egress）。
4. 测试与门禁体系成体系：免费/付费分片、真实事故回放测试、31 个 CI 工作流、自我 slop 扫描。
5. 文档密度与质量极高：66 篇 docs + ARCHITECTURE/ETHOS/BROWSER 等根文档，设计决策有完整"为什么"。

**风险/局限**
1. 重度绑定单一生态：核心体验为 Claude Code（full tier），其他宿主均为 experimental 且安全技能"advisory, not blocked"（README 宿主表）；/cso、/design-motion 等依赖 Docker、ffmpeg、原生工具链。
2. 本机编译未签名二进制在 Windows Smart App Control 下被拒（issue #2595，README 已知问题），Windows 路径依赖 WSL 或降安全设置。
3. 质量门禁成本高：完整付费 gate 单次约 $33（CHANGELOG 1.91.322.0），个人用户需要自行理解并配置 eval 预算。
4. 单人 + AI 的超高速演进（月 280 个版本）带来稳定性与可追溯性风险：变更说明虽详尽，但外部贡献者与 API 兼容性承诺不明确（14 名贡献者，未确认是否有语义化版本承诺之外的稳定性保证）。

## 10. 总结评价

gstack 是 2026 年"AI 原生软件开发"最具代表性的开源实践之一：它不是又一个 prompt 合集，而是一套把角色分工、评审门禁、预算计量、安全审计和自研浏览器引擎打包成完整软件工厂的系统，且由 YC CEO 以近乎日更的节奏亲自运营并 dogfood。星数（137k）很大程度上是作者身份与"公开个人工作流"叙事的产物，但源码本身的工程密度配得上关注。适合三类人：想给 Claude Code/其他 agent 建立严格流程的创始人与技术负责人（直接安装使用）；研究"如何把 agent 工作流做成可测试、可门禁的软件系统"的工程师（读 ARCHITECTURE.md + review/ship 技能 + lib/ship-budget.ts）；以及想借鉴其技能生成与 eval 门禁机制的 AI 工具作者（读 scripts/ 与 .github/workflows）。不建议把它当作中立的多宿主框架——它是强个人风格、强 Claude Code 倾向的"言论自由软件"，用它的前提是接受它的一整套观点。
