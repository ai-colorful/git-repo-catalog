# luongnv89/claude-howto 项目分析

> 星数 46,608（快照 2026-11-06，gh api 实测）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-06

## 1. 项目速览

一句话定位：一份"可视化、示例驱动"的 Claude Code 中文圈之外最流行的社区教程——用 10 个递进式模块、351 张 Mermaid 图和大量可复制粘贴的模板，教用户从会敲 `claude` 到编排 agents/hooks/skills/MCP 的完整工作流。

| 项目 | 值 | 出处 |
|------|-----|------|
| 名称 | luongnv89/claude-howto | gh api |
| 星数 / Fork / Watch | 46,608 / 5,705 / 204 | gh api（2026-11-06） |
| 创建时间 | 2025-11-07 | gh api |
| 最近推送 | 2026-11-03（fix scripts 提交） | gh api |
| 许可证 | MIT | `LICENSE` |
| GitHub 语言标签 | Python（实际主体为 Markdown，Python 仅为工具链） | gh api languages |
| 主页 | http://luongnv.com/claude-howto/（GitHub Pages 静态站） | gh api homepage、`.github/workflows/pages.yml` |
| 版本模型 | 跟随 Claude Code 版本号同步（v2.0.0 → v2.1.245+） | `CHANGELOG.md` |

## 2. 项目是什么，解决什么问题

作者在 README 里把痛点写得非常直白（`README.md` "The Problem" 一节）：官方文档只"描述功能"，不教你怎么把 slash commands、hooks、memory、subagents 组合成真正省时间的工作流；没有清晰的学习路径，用户"什么都 skim 一遍，什么都没掌握"；网上示例太基础，一个 hello-world slash command 帮不上你搭生产级的 code review pipeline。项目自述定位不是"又一份功能参考"，而是结构化、可视化、示例驱动的教程，配套可立即复制进项目的生产级模板。

内容组织为 **10 个编号模块**（`01-slash-commands/` 到 `10-cli/`）：Slash Commands、Memory、Skills、Subagents、MCP、Hooks、Plugins、Checkpoints、Advanced Features、CLI。README 给出两条学习路径：按模块顺序的完整路线（总计约 11-13 小时，`LEARNING-ROADMAP.md`），以及按水平分层的快捷入口（Beginner/Intermediate/Advanced 三档，各配预计时长）。

区别于静态教程的一个特色是**内置自测**：仓库把两个 Claude Code skill（`.claude/skills/lesson-quiz/`、`.claude/skills/self-assessment/`）直接放进仓库，用户在 Claude Code 里运行 `/lesson-quiz hooks` 或 `/self-assessment` 即可对自己的理解程度出题打分。`lesson-quiz/SKILL.md` 写有明确的 guardrails："绝不编造题目"，只允许从 `references/question-bank.md` 题库出题——这是把教程和 AI 助手结合得比较认真的设计。

项目还有一个明确写出来的野心：`docs/ROADMAP-20260401.md`（2026-2027 路线图）提出把仓库从"静态教程"升级为"双层知识库"——服务人类的同时，通过 `agent-manifest.json`、`AGENT-INDEX.md`、`RECIPES.md` 让 AI agent 在执行 Claude Code 任务前先查询本仓库，原话是 "No competitor targets AI agents as a primary audience. This is the moat."（截至本次分析，这些产物尚未出现在仓库中，属规划）。

## 3. 技术栈与依赖

内容型仓库，"技术栈"分两层：

**内容层（主体）**：纯 Markdown + Mermaid 图 + 少量 PNG 截图。无任何运行时框架，读者只需 `git clone` 后阅读，或把模板文件复制到自己的项目。

**工具链层（`scripts/`，Python 为主）**：
- `build_website.py`：静态站生成器，依赖 markdown 3.7 + beautifulsoup4 4.12.3 + jinja2 3.1.4（以 uv `#!/usr/bin/env -S uv run --script` 内联声明，与 `scripts/requirements.txt` 版本一致；文件头注释解释了 pin 在 3.7 的原因——markdown 3.11+ 会错误解析代码片段中的裸 `</...>`）
- `build_epub.py`：EPUB 电子书导出，图预先渲染为 SVG
- `pele_renderer.py` + `pele_render.mjs`：Mermaid→SVG 渲染，通过单一 Node.js（>=20）进程运行 sha256 校验过的 pinned "Pele" bundle，Python 侧仅 stdlib；区分环境故障（`PeleUnavailableError`）与图本身语法错误（`PeleHelperError`/`PeleResult.error`）
- 五个校验器：`check_links.py`、`check_mermaid.py`、`check_cross_references.py`、`check_markdown_rendering.py`、`check_epub_diagrams.py`
- `sync_translations.py`（多语言同步）、`vendor_assets.py`（下载校验 Tailwind CLI/Pele bundle/字体等 vendor 资产，缓存于 `scripts/.vendor-cache/`）

网站端：自托管 Tailwind CSS + Inter 字体 + 浏览器端 Pele 渲染 Mermaid，明确声明"运行时零第三方 CDN 脚本"（`scripts/build_website.py` docstring）。gh api languages 显示 Python 420KB / Shell 88KB / CSS 41KB / Jinja 37KB / JS 30KB / HTML 20KB——Python 字节数大是因为 vendor 测试与模板，教程正文全是 Markdown。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|------|------|
| `01-slash-commands/` … `10-cli/` | 10 个教程模块：模块 README（正文）+ 可复制模板（slash 命令 md、hook 脚本、MCP JSON、skill/插件包） |
| `ja/ ko/ uk/ vi/ zh/` | 五个完整翻译树（日/韩/乌克兰/越南/中文），含各自 README、CATALOG、CHANGELOG |
| `scripts/` | Python 工具链：网站/EPUB 构建、5 个文档校验器、翻译同步、vendor 资产管理、`tests/` pytest 用例 |
| `resources/` | 设计系统（`DESIGN-SYSTEM.md`）、logo/favicon/图标清单（`MANIFEST.txt`） |
| `assets/`、`slides/` | 网站 logo；2 份作者演讲 PDF（`slides/LuongNGUYEN-Coding-with-AI.pdf` 等） |
| `.claude/skills/` | lesson-quiz、self-assessment 两个可直接安装的 Claude Code skill |
| `.github/` | 4 个 workflow（test/docs-check/pages/release）+ 5 个 issue 模板 + `TESTING.md` |
| 顶层 md | `README`（31KB）、`CATALOG.md`（49KB 功能目录）、`INDEX.md`（33KB）、`LEARNING-ROADMAP.md`（29KB）、`QUICK_REFERENCE.md`（20KB 速查卡）、`claude_concepts_guide.md`（53KB）、`STYLE_GUIDE.md`、`CHANGELOG.md`（68KB） |
| 其他 | `docs/ROADMAP-20260401.md` + `TASKS-20260401.md`（内部规划）；`coverage.xml`（误提交的覆盖率报告）；`.idd.yml`（issue 驱动开发配置，自述"文档仓，docs-sync PR 无需关联 issue"） |

组织方式评述：这是典型的"内容 monorepo"——教程、模板、翻译、构建工具、CI 全在一个仓库。模块编号即学习顺序，翻译目录镜像主结构，工具链集中在 `scripts/` 不与内容混杂。`CLAUDE.md` 与 `STYLE_GUIDE.md` 的存在表明内容生成有严格的风格约束（这也是 AI 辅助写作仓的常见配套）。

## 5. 核心架构与关键实现

内容型仓库，"核心实现"体现在三处：

**（1）模块正文 + 模板的双层结构。** 每个模块是一个大而全的单文件教程加一套立即可用的模板。以最重的 `06-hooks/README.md` 为例：单文件 1,688 行，覆盖 31 个 hook 事件、5 种 hook 类型（command/http/prompt/mcp_tool/agent）、matcher 模式、JSON 输入输出等（该 README 目录结构可证），旁边配 11 个真实可跑的 hook 脚本（`06-hooks/security-scan.sh`、`pre-commit.sh`、`context-tracker-tiktoken.py` 等）。10 个模块 README 合计 12,059 行。

**（2）可直接安装的示例资产。** 模板不是片段而是完整构件：`03-skills/` 下 6 个完整 skill 包（如 `code-review-specialist/` 含 SKILL.md + `scripts/analyze-metrics.py` + templates）；`05-mcp/` 4 份 MCP 配置 JSON（database/filesystem/github/multi）；`07-plugins/` 3 个完整插件包（`pr-review/` 含 `.claude-plugin/plugin.json`、3 个 agents、3 个 commands、hook、MCP 配置）。模块内非 md 模板文件 37 个（`find 0*/ -type f ! -name "*.md" ! -name "*.png"` 统计）。

**（3）"文档即软件"的工程化。** 内容按软件工程标准管理：CHANGELOG 严格跟随 Claude Code 上游版本（见第 9 章）；CI 用 pytest 测试构建脚本和校验器（`.github/workflows/test.yml`，矩阵 3.10/3.11/3.12，uv 安装）；链接、交叉引用、Mermaid 语法、渲染效果全部有自动校验器把关。CHANGELOG 中 v2.0.0（2026-02-01）一节甚至公开列出了早期版本的"虚构内容修正"清单——把虚构的 permission mode、hook 事件（`PreCommit`/`PostPush` 等）逐一替换为官方真实行为（`CHANGELOG.md` 1181 行起）。这种"承认错误并系统纠错"的记录在教程仓里相当少见。

## 6. 构建与运行

本项目无需"运行"，以下均来自项目文档与工程文件（`scripts/`、`.github/workflows/`、`README.md`）：

```bash
# 教程使用：clone 后把模块模板复制进自己的项目（QUICK_REFERENCE.md）
git clone https://github.com/luongnv89/claude-howto.git && cd claude-howto
cp 01-slash-commands/*.md .claude/commands/        # 安装全部 slash 命令示例

# 构建静态网站（GitHub Pages 用，scripts/build_website.py 头部）
uv run scripts/build_website.py                    # 输出 site/，支持 --lang vi 等
uv run scripts/build_website.py --output site/ --verbose

# 构建 EPUB / 渲染校验（scripts/ 各脚本 --help）
uv run scripts/build_epub.py
uv run scripts/check_mermaid.py                    # Mermaid 语法校验（Pele 渲染）

# 测试工具链（.github/workflows/test.yml，pytest + uv，Python 3.10-3.12）
cd scripts && pytest tests/
```

网站部署为全自动：`.github/workflows/pages.yml` 监听 `**/*.md` 与构建脚本变更，push 到 main 即重建 GitHub Pages。

## 7. 教程章节组织与规模统计（内容特例）

**规模实测**（tarball 全量统计）：
- 全仓 Markdown **580 个文件**；其中英文主内容 **109 个 / 33,228 行**，五个翻译树合计 471 个 md（ja 102、zh 99、ko 98、uk 96、vi 76）
- 10 个模块 README 合计 **12,059 行**；单模块最大 `06-hooks`（1,688 行）、次大 `07-plugins`（27 个 md，39 个文件）
- **Mermaid 图 351 个代码块**（全部 md 合计），主内容中 71 个文件含 Mermaid
- 图片仅 **29 个**（PNG 实机截图 + SVG logo，翻译目录有复制），截图集中在 01/02 模块（如 `01-slash-commands/pr-slash-command.png`、`02-memory/memory-saved.png`）
- 可复制模板/示例资产：37 个非 md 文件（hook 脚本、MCP JSON、skill/plugin 包内脚本），外加 8 个示例 slash 命令 md、6 个 skill 包、3 个插件包、4 份 MCP 配置
- 辅助学习件：`CATALOG.md` 功能目录（13 大节）、`LEARNING-ROADMAP.md`（分四级路线 + 每级练习）、`QUICK_REFERENCE.md` 速查卡、题库驱动的两个 quiz skill（`lesson-quiz/references/question-bank.md`）

**组织方式**：编号模块即依赖顺序（README 明确"each builds on the last"），每模块自带"概念讲解 → 内部机制图 → 可复制模板 → quiz"。这种"编号课程 + 模板库 + 测验"三位一体是它区别于 awesome 清单和官方文档的核心结构。

## 8. 可视化与 Python 工具链（内容特例）

**可视化实现方式**：不是截图堆砌，而是以 Mermaid 源码为唯一图源，多端渲染——
1. **GitHub 原生渲染**：md 内 Mermaid 代码块在 github.com 直接显示；
2. **静态网站**：`build_website.py` 生成 GitHub Pages 站，浏览器端用 pinned Pele bundle（`pele.global.js`）渲染，自托管 Tailwind + Inter，支持暗色模式与响应式，运行时零 CDN 依赖；
3. **EPUB**：`build_epub.py` 经 `pele_renderer.py` 预渲染为 SVG 嵌入。
`pele_renderer.py` 的设计相当讲究：所有图共享一个 Node 进程批量渲染；用 sha256 校验 vendor bundle；把"环境缺 Node"与"图写错了"分成两类异常，避免误报。README 截图（PNG）只用于展示终端实际交互效果，图解一律用 Mermaid——保证图可版本化、可翻译、可自动校验。

**Python 部分的角色**：纯工程支撑，不参与教程内容本身——网站/EPUB 构建、5 类内容质量校验（链接/交叉引用/Mermaid/渲染/EPUB 图）、翻译同步、vendor 资产管理，合计约 3,965 行（`wc -l scripts/*.py`），配 9 个 pytest 测试文件（`scripts/tests/`）。GitHub 把语言标成 Python 只是工具链字节数所致，容易让人误以为是代码项目。

**交互式学习**：`.claude/skills/lesson-quiz/` 与 `self-assessment/` 是仓库里唯一"可执行"的教学件——在 Claude Code 内运行即可出题评分，题库独立存放且有防编造 guardrail，是 2026 年"仓库内嵌 Claude Code skill"这一新兴形态的范例。

## 9. 社区、更新节奏与增长背景（内容特例）

**更新节奏（有据可查）**：`CHANGELOG.md` 共 1,227 行、20 个版本条目，从 v2.0.0（2026-02-01）到 v2.1.245（2026-08-25），几乎每个 Claude Code 上游版本（v2.1.126/131/138/143/145/150/160/212/217/220/245）都有对应 sync 条目，且每条注明上游 changelog 来源（`https://code.claude.com/docs/en/changelog`）。最近的 README 徽章已标 v2.1.310（2026-10，`README.md` 15/105 行），说明 sync 仍在继续但 CHANGELOG 未逐版补录（CHANGELOG 顶部自己承认 v2.1.233/235 两次 sync 漏记）。GitHub Releases 只发到 v2.1.160（2026-06-02），之后改为纯 commit 推进。最近三次提交（2026-11-02/03）均为 scripts 修复，gh api 可证。

**社区结构**：29 名贡献者，作者 luongnv89 以 210 次贡献占绝对主导（约 78%），第二贡献者 edocltd 20 次；其余多为翻译与脚本小修（xiaolai、lzw-git-all、jeffreyyjp 等与 zh/ja/ko 翻译树对应）。5 个语言树的存在本身就是社区需求驱动的证据——韩语落地页链接是 2026-11-02 才加的（提交 #232）。52 个 open issues。文档基建完善：5 个 issue 模板、`CONTRIBUTING.md`（11KB）、`STYLE_GUIDE.md`（17.6KB）、`SECURITY.md`。

**增长背景（客观陈述）**：仓库 2025-11-07 创建，到 2026-11-06 达 46,608 星，约 12 个月折合日均 ~127 星，对一个文档教程仓属于极端速度（同期典型热门文档仓年增星在数千量级）。可验证的配套事实：5,705 fork、204 watch、README 内嵌 star-history 曲线与自标的 "#1 Trending" 徽章（`README.md` 7-10 行，自述性、未经独立验证）；增长窗口与 Claude Code 2025-2026 的爆发式普及期完全重合，且该仓是"随上游版本逐周同步"的少数社区教程之一，翻译树覆盖中日韩越乌五个非英语市场，这些都是可从仓库内容直接验证的、与高速增长自洽的因素。星数真实性（是否含刷星）无法从仓库内部验证，本报告不做臆断。

## 10. 亮点与风险

**亮点**：
1. 示例质量高且"拿来即用"：模板是完整可安装的 skill/插件/hook 包，不是代码片段（`07-plugins/pr-review/` 等可证）；
2. 工程化程度远超同类教程：4 个 CI workflow、5 个内容校验器、题库式 quiz、严格 CHANGELOG（`scripts/`、`.github/workflows/`）；
3. 可视化架构合理：Mermaid 单一图源多端渲染，自托管无 CDN（`scripts/build_website.py`）；
4. 诚实纠错记录：v2.0.0 修正清单公开列出早期虚构功能并逐一对照官方文档纠正（`CHANGELOG.md`）；
5. 五语翻译树完整，覆盖东亚主流开发者市场。

**风险/局限**：
1. **内容时效风险是结构性**：整个仓库的价值绑定在 Claude Code 上游行为上，上游一次大改版（hook 事件、MCP 规范）就会让大量内容过期，维护成本终身存在——CHANGELOG 里多次"版本声明互相矛盾"的自我修正即是征兆；
2. **版本号混乱**：README 徽章 v2.1.310 与 CHANGELOG 顶部 v2.1.245 不一致，zh 翻译徽章停在 v2.1.112，翻译滞后于英文主版本（`README.md`、`CHANGELOG.md`、`zh/README.md` 可证）；
3. 单文件巨型教程（1,688 行的 hooks README）阅读负担重，网页端有侧边栏但 GitHub 上滚动体验一般；
4. `coverage.xml` 误提交进仓库、`docs/` 内部规划文档（含"竞品 moat"表述）公开可见，仓库卫生有瑕疵。

## 总结评价

claude-howto 是 2026 年 Claude Code 教程潮中跑出来的头部社区项目：它把"教人用 AI 编码工具"这件事当软件工程做——模板即装即用、图源单一可校验、变更逐版追踪、五个语言树社区共建。12 个月 46.6k 星的速度放在任何品类都属罕见，仓库内容本身的质量密度（580 个 md、351 张图、60+ 可安装模板）与其流行度基本相称；它更像一本"随上游软件周更的开源书"，长期价值取决于作者能否持续跟随 Claude Code 演进。适合三类人：想系统学 Claude Code 的中高级开发者（按 LEARNING-ROADMAP 走）、想找现成 hook/skill/插件模板抄进项目的工程团队（直接搬 05-08 模块）、以及研究"AI 教程仓如何工程化运营"的社区运营者（看 scripts/ 与 CHANGELOG 的做法）。源码层面唯一值得细读的是 `scripts/pele_renderer.py` 的 Mermaid 批量渲染设计与 `build_website.py` 的零 CDN 自托管方案。

---
*分析基于 2026-11-06 下载的 HEAD tarball（默认分支 main，1,029 个条目）与 gh api 元数据；所有路径均相对仓库根目录。*
