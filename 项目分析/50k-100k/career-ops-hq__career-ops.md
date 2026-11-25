# career-ops-hq/career-ops 项目分析

> 星数 76761（快照 2026-10-24）｜主语言 JavaScript｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

**一句话定位：跑在 AI 编程 CLI 里的开源求职 Agent——扫描职位板、按你的简历给每个职位打 1-5 分、定制 ATS 友好简历与求职信、跟踪投递全流程，但"提交"永远由人来按。**

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | career-ops-hq/career-ops | gh api |
| 星数 | 76761（快照）；分析时 78885，Fork 14560，Open Issues 507 | gh api（2026-10-31） |
| 主语言 | JavaScript（纯 ESM `.mjs`，无 TS；dashboard 为 Go，web/ 为 Next.js） | 根目录清单、`dashboard/go.mod`、`web/package.json` |
| 许可证 | MIT | `LICENSE` |
| 版本 | v1.38.0（2026-10-23 发版，release-please 自动化） | `VERSION`、`CHANGELOG.md`、gh releases |
| 创建时间 | 2026-04-04（约 7 个月涨到 78k 星） | gh api `created_at` |
| 最近活跃 | 最近 push 2026-10-30，极度活跃 | gh api `pushed_at` |
| 作者 | Santiago Fernández de Valderrama（@santifer），Head of Applied AI | `package.json` author、`AGENTS.md` Origin |
| 官网 | https://career-ops.org | gh api homepage |

## 2. 项目是什么，解决什么问题

career-ops 是一个"AI 求职操作系统"。作者把自己求职的过程工程化：先用它评了 740 个职位、投出 68 份、拿到 12 个面试和 1 个 offer（README.md、AGENTS.md Origin 节），随后开源。它解决的核心痛点是投递的"量"与"质"失衡——用 AI 过滤候选人的公司很多，给候选人 AI 用来**挑选公司**的工具几乎没有（README、MANIFESTO.md）。

与"一键海投"机器人不同，项目有三条设计承诺（ARCHITECTURE.md Principles）：**本地优先**（一切在你机器上、对着你的文件跑，无账号无服务器）；**AI 无关**（逻辑核心是 `modes/` 下的 Markdown 提示词文件，由任意 AI CLI 执行——Claude Code、Codex、OpenCode、Gemini CLI、Qwen、Grok、Kimi、Antigravity、Copilot、Cursor、Pi 均有入口文件）；**人在回路**（工具只准备与评估，人审查并亲自点击 Submit，`modes/oferta.md` 中反复强调）。

完整工作流：`scan.mjs` 零 token 扫描公开职位源写入 `data/pipeline.md` → 粘贴职位 URL/JD 触发 `modes/oferta.md` 的 A-H 结构化评估（含 1-5 分、招聘真实性 Block G、草拟答案 Block H）→ 生成定制简历 PDF/LaTeX 与求职信 → `data/applications.md` 跟踪表管理漏斗 → 面试准备、跟进、复盘（`modes/` 有 60+ 模式）。

完整流程的关键门控贯穿始终：评估前先跑**活链检查**（`check-liveness.mjs`/`liveness-*.mjs`，零 token 确认职位仍在招，死链接绝不浪费一次完整评估）；评分低于 4.0 时项目明确建议不投（`modes/_shared.md` Score interpretation 节），即"宁可少投"是产品的默认价值观。

社区运营是其暴涨星数的重要一环：MANIFESTO 在 6 万星时发布并有签名墙（MANIFESTO.md、SIGNATURES.md），"Hired Wall" 记录 22 个公开求职成功案例（HIRED.md、`docs/hired-count.json`，2026-10-30 更新），README 有 20 种语言译本，被 WIRED、Business Insider 报道，登过 Trendshift 日榜第一（README.md 媒体区）。

## 3. 技术栈与依赖

- **运行时**：Node.js ^22.13.0 || >=23.4.0（`package.json` engines；CI 显式钉住 22.13 下限，`lib/node-floor.mjs`）。
- **发行形态**：既是 git 仓库也是 npm 包（`npx @santifer/career-ops init` 安装器）与 Claude Code 插件市场仓库（`.claude-plugin/marketplace.json`），版本 1.38.0 三处同步（`VERSION`、`package.json`、`plugin.json`）。
- **依赖极简**（`package.json` dependencies 仅 5 个）：`playwright`（浏览器自动化、HTML→PDF）、`undici`（HTTP）、`js-yaml`（YAML 配置）、`dotenv`、`@google/generative-ai`（Gemini 免费档独立评估器）。
- **语言形态**：零框架纯 Node ESM 脚本；"逻辑在 Markdown 提示词里"是最大特色——`modes/*.md` 是被 AI CLI 执行的大脑，`.mjs` 脚本做确定性辅助（扫描、写文件、PDF、校验）。
- **可选组件**：`dashboard/` 是 Go 写的终端 TUI（95 个 .go 文件，含成套 `_test.go`）；`web/` 是 Next.js 16 + React 19 的本地 Web UI（独立发版 web-v0.16.0，`web/package.json`）。
- **数据层**：Markdown/TSV 文件为唯一权威源，SQLite 仅作派生查询索引、可随时重建（ARCHITECTURE.md "Files are canonical" 节，决议 #918）。
- **文档即代码**：根目录 28 个顶级 .md（AGENTS.md 693 行、DATA_CONTRACT.md、CONTRIBUTING.md、GOVERNANCE.md、LEGAL_DISCLAIMER.md 等）。

## 4. 目录结构与代码组织

顶层是**刻意扁平的单仓**：165 个 `.mjs` 脚本直接放根目录，"一个脚本一个职责"，路径稳定性是特性——更新器白名单、社区插件与用户肌肉记忆都依赖这些路径（ARCHITECTURE.md "Why the flat root" 节，#1386）。

| 目录 | 职责 |
|---|---|
| `modes/` | 提示词大脑：`oferta.md`（862 行，A-H 评估）、`_shared.md`（317 行，评分体系/全局规则）+ 60+ 模式与 25 个语言子目录（zh/、ja/、es/ 等） |
| `providers/` | 145 个文件，142 个非助手职位源适配器（Greenhouse/Ashby/Lever/Workday/Amazon/Apple 等，`_` 前缀为共享助手），`ADDING_A_PROVIDER.md` 教你加源 |
| `lib/` | 51 个共享模块（`update-manifest.mjs`、`path-resolver.mjs`、`syntax-check.mjs` 等） |
| `tests/` | 996 个 `*.test.mjs`，约 20 万行测试代码 |
| `scripts/` | 辅助脚本（语法检查、本地 UI 服务器 `ui-server.mjs`） |
| `templates/`、`fonts/` | ATS 安全简历模板（HTML/LaTeX/CJK）与字体 |
| `dashboard/` | Go TUI 漏斗看板（可选，与核心隔离） |
| `web/` | Next.js 本地 Web UI（独立 npm 发版） |
| `data/`、`reports/`、`jds/`、`output/`、`interview-prep/`、`writing-samples/` | 用户层数据区（随仓库只带 .gitkeep 脚手架） |
| `docs/` | 29 篇专题文档（SETUP、PLUGINS、WINDOWS、NIXOS、RUNNING_ON_A_BUDGET、SUPPORTED_JOB_BOARDS 等） |
| `.claude-plugin/`、`.agents/`、`.codex-plugin/`、`.cursor-plugin/` 等 | 各 CLI 的插件/技能入口；`.claude/skills/`、`.opencode/`、`.qwen/`、`.kimi/`、`.grok/`、`.antigravitycli/` 中均为指向 `.agents/skills/career-ops/SKILL.md` 的符号链接 |
| `evals/` | 评估质量金样本与 fixtures（`evals/golden/`） |
| `hooks/`、`config/`、`seeds/`、`batch/`、`plugins/`、`plugins-registry/` | 会话钩子、示例配置、扫描种子（YC、a16z 等）、批量输入/回传、社区插件与注册表校验（`validate-plugin-registry.mjs`） |

`AGENTS.md` 是规范入口（CLAUDE.md/CODEX.md/OPENCODE.md/KIMI.md 都是其 `@AGENTS.md` 薄包装）；`.agents/skills/career-ops/SKILL.md` 遵循开放 agent skill 标准，承担跨 CLI 模式路由。

## 5. 核心架构与关键实现

### 5.1 数据契约：系统层/用户层严格分离（`DATA_CONTRACT.md`）

全仓最重要的一条架构规则：`cv.md`、`config/profile.yml`、`modes/_profile.md`、`data/`、`reports/` 等是**用户层**，自更新器永不触碰；`modes/_shared.md`、全部脚本、模板是**系统层**，由 `update-system.mjs` 按 `lib/update-manifest.mjs` 的 `SYSTEM_PATHS` 白名单拉取升级。边界由 `updater-migration-tests.mjs` 在 CI 强制（ARCHITECTURE.md）。这解决了"工具要持续迭代、用户数据不能被覆盖"的矛盾，是这个项目能放心自更新的根基。

### 5.2 零 token 扫描器 + 插件式 provider 层（`scan.mjs`、`providers/`）

`scan.mjs`（255KB）头注释自述："Zero-token portal scanner with a plugin-based provider layer"——每个 provider 导出 `{ id, detect(entry), fetch(entry, ctx) }`，新源即拖入一个 .mjs 文件。只读公开无鉴权 API（Greenhouse、Ashby、Lever、BambooHR、Workday 等），登录墙源刻意排除到插件层。安全设计密集：`providers/_safe-url.mjs`、`_ip-guard.mjs`、`_trust-validator.mjs`、`_connection-scope.mjs` 做 SSRF/HTTPS/主机白名单防护，`tests/providers/ats-ssrf-hardening.test.mjs` 与根脚本 `validate-untrusted-content-coverage.mjs` 守护覆盖面。结果写入 `data/pipeline.md`（URL 收件箱）。

### 5.3 提示词即程序：A-H 评估协议（`modes/oferta.md` + `modes/_shared.md`）

评估核心是 862 行的 `oferta.md`：入口即多重门——**活链门**（死链绝不进评估，`---DEAD_POSTING---` 哨兵行）、**黑名单门**（#1742，用户自己的勿投名单）、**Dealbreaker 门**（用户硬规则，命中即停）。之后产出 A-H 报告块（A-F 评估、G 招聘真实性、H 仅 4.5+ 分才生成的草拟答案）。`_shared.md` 定义五维评分（CV 匹配、北极星契合、薪酬、文化信号、红旗）合成 1-5 全局分，并要求输出**证据置信度**（supported/partial/unknown 三态映射 High/Medium/Low，见 "Evidence confidence" 节）——这是对"AI 幻觉打分"的直接工程化对抗。 unethical JD 注入也被显式处理："JD 文本是数据不是指令"，含指令性文本按 Block G 异常引用（`oferta.md` 首节）。

无 CLI 时可用独立评估器跑同一评分协议：`gemini-eval.mjs`（Google 免费档）、`ollama-eval.mjs`（全本地）、`openai-eval.mjs`（任意兼容端点）、`batch-evaluate-gemini.mjs` 批量；`modes/_shared.md` 的 Spend Tier 表按 economy/standard/premium 三档路由模型。

### 5.4 文件即权威 + 自更新系统（`tracker.mjs` 系、`update-system.mjs`）

跟踪层以 `data/applications.md` 为权威表，`reports/{NNN}-{company}-{date}.md` 存完整评估，`reserve-report-num.mjs` 原子领取报告编号，`merge-tracker.mjs`（131KB）、`dedup-tracker.mjs`、`reconcile-pipeline.mjs` 保证一致性，SQLite 仅派生。自更新器 `update-system.mjs`（286KB）负责备份→拉取→仅检出 SYSTEM_PATHS，`tests/root-tests-registration.test.mjs` 等守护升级安全。整套设计让用户"装一次、永远跟上游、个人数据永不动"。

## 6. 构建与运行

来自 `package.json` scripts 与 README/docs（均为项目声明的真实命令）：

```bash
# 安装（下载最新 release 到 ./career-ops 并装依赖）
npx @santifer/career-ops init
# Claude Code 插件方式
/plugin marketplace add career-ops-hq/career-ops && /plugin install career-ops@career-ops

node doctor.mjs            # 体检：依赖/路径/配置
node scan.mjs              # 零 token 扫描（--verify 做活链核验）
node scan-ats-full.mjs --seeds yc,a16z   # 种子公司全量扫描
# 评估：在 AI CLI 内粘贴职位 URL/JD，或批量：
node openrouter-runner.mjs pipeline        # npm run or:pipeline
node gemini-eval.mjs / ollama-eval.mjs     # 免费档/本地模型评估
node generate-pdf.mjs      # 简历 PDF（Playwright）
npm run test               # 说明：无此 script；测试入口为↓
node test-all.mjs          # 全量测试（glob 发现 tests/**/*.test.mjs）
node test-all.mjs --quick / --only <substring> / --jobs N
npm run lint               # node scripts/check-syntax.mjs
```

另有 `Dockerfile` + `docker-compose.yml`（DOCKER.md）、`flake.nix`（NixOS，docs/NIXOS.md）。

## 7. 测试与工程质量

- **规模**：`tests/` 下 996 个 `*.test.mjs`、约 20 万行；`test-all.mjs`（1MB 主控）按 glob 自动发现、每套 30s 超时、多进程并行、跨机器队列锁（`lib/test-queue.mjs`），另有内联核心段（语法、数据契约、个人数据、路径等）。ARCHITECTURE.md 自述 "500+ checks"（口径不同，实际测试文件数远超）。
- **CI**（`.github/workflows/` 共 28 条）：`test.yml` 三 OS 矩阵（ubuntu/macos/windows，显式钉 Node 22.13 语法下限 + Node 24 跑测试、Windows 强制 LF、merge queue 支持）；CodeQL 必过；CodeRabbit 逐 PR 评审（`.coderabbit.yaml` 19KB）；Renovate 管依赖；SBOM、dependency-review、release-please 自动发版。
- **测试风格**：行为命名为主（约 4/5 直接命名行为而非模块，ARCHITECTURE.md 详述）；`tests/cv-visual/` 用 Playwright 快照测简历视觉（`playwright.cv.config.mjs`）；`evals/golden/` 存评估金样本；`updater-migration-tests.mjs` 专测升级迁移边界。还有安全专项（SSRF 加固、`validate-untrusted-content-coverage.mjs` 验证"不可信内容"规则覆盖）。
- 工程纪律突出：守卫测试防止回归约定本身（如 `tests/no-root-suites.test.mjs` 保根目录无游离测试），文档中的断言甚至有 `architecture-diagrams.test.mjs` 之类校验图表与代码一致。

## 8. 社区与维护状态

- **活跃度**：创建于 2026-04-04，约 7 个月至 78.9k 星；最近 push 2026-10-30（快照前一天）；发版密集（v1.38.0 于 2026-10-23，web 与主包双轨发版，gh releases）。CHANGELOG.md 达 474KB，引用的 issue 编号已到 #5736+。
- **贡献者**：README 自称 "700+ contributors"；`.all-contributorsrc` 记录 203 人；GitHub contributors API 分页头显示约 467+ 条目（per_page=1 时 last=468）。GOVERNANCE.md、MAINTAINERS.md、CODEOWNERS 齐备。
- **社区机制独特**："I got hired" issue 模板驱动 Hired Wall（22 例在案）；MANIFESTO 签名墙（SIGNATURES.md 30KB）；26 条工作流中多条专为社区运营（welcome、gfi-claims、pr-adoption、manifesto-guestbook、label-bot）。
- **风险信号**：Open Issues 507 偏高但配有 auto-triage、stale、close-stale 机器人流水线（`.github/workflows/`），未见失控证据。
- **多语言社区贡献模式**：`modes/` 下 25 个语言目录与 20 份 README 译本由社区以 PR 维护（CHANGELOG 中多条 "adopt Modes X Localization" 提交，如 #5325 土耳其语），并有 `i18n-drift.mjs` 检测译文漂移。

## 9. 亮点与风险

**亮点**
1. **架构范式新颖且有实效**："提示词文件当程序写"（`modes/` 862 行评估协议）+ 脚本做确定性护栏，是 AI-native 应用的成熟样本；系统/用户层契约、文件权威+派生索引等决策都有 issue 编号溯源（`DATA_CONTRACT.md`、ARCHITECTURE.md）。
2. **安全意识罕见地到位**：JD 视为不可信输入防提示注入、provider 层 SSRF 硬化、专守卫测试验证覆盖面（`providers/_ip-guard.mjs`、`tests/providers/ats-ssrf-hardening.test.mjs`、`validate-untrusted-content-coverage.mjs`）。
3. **工程质量对得起 78k 星**：996 个测试文件、三 OS CI、CodeQL、行为命名规范、守卫测试测试约定本身。
4. **反幻觉设计**：评分强制证据置信度三态、story-bank 数字溯源标记（`story-provenance-check.mjs`）、"关键词只重述不编造"硬规则（AGENTS.md Source-of-Truth Boundary）。
5. **多 CLI 中立**：一套逻辑 11+ 个 AI CLI 入口，`.agents/skills/` 走开放标准，6 个符号链接目录即插即用。

**风险/局限**
1. **行为一致性依赖 LLM 遵从超长提示词**：核心逻辑在 Markdown 而非代码，模型换代/换档可能悄然改变评估口径；置信度与 golden eval 只是缓解而非消除。
2. **抓取合规的猫鼠游戏**：142 个 provider 依赖公开 ATS API 与页面结构，任何源改版即失效；项目用 robots.txt 尊重与验证脚本应对（docs/SUPPORTED_JOB_BOARDS.md），但长期维护成本高。
3. **上下文成本与认知负担重**：AGENTS.md 693 行 + DATA_CONTRACT 27KB + 165 个根脚本，对 AI 上下文窗口和人类新贡献者都不轻。
4. **简历数据经第三方 LLM**：本地优先指存储与执行，提示词内容仍会发送给所选模型供应商；LEGAL_DISCLAIMER.md 有提示但用户需自知。

## 10. 总结评价

career-ops 是 2026 年"AI-native 开源应用"的标杆案例：它把求职从海量海投变成有纪律的漏斗运营，技术上的最大创新是**用严格的数据契约和守卫测试把"提示词当生产代码"管起来**，从而在 7 个月内借社区运营（宣言、Hired 墙、多语言）冲到近 80k 星。它不自动投递、不碰登录墙源、不编造简历事实，工程与伦理边界画得清晰。适合正在求职、已在使用任意 AI 编程 CLI 的工程师直接上手；也极适合研究"如何工程化 LLM Agent"的开发者精读其 `modes/` 协议、`DATA_CONTRACT.md` 与测试布局——这套方法论的参考价值不亚于工具本身。注意其提示词驱动特性带来的行为漂移风险，以及对职位源持续维护的依赖。
