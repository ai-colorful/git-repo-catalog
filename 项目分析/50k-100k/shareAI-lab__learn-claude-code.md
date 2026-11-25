# shareAI-lab/learn-claude-code 项目分析

> 星数 79092（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：一门"从 0 到 1 造一个 Claude Code 式 agent harness"的开源课程，17 个递进式 Python 章节，每章只教一个 harness 机制。

| 事实 | 值 | 出处 |
|---|---|---|
| 名称 | shareAI-lab/learn-claude-code | gh api |
| 简介 | "Bash is all you need - A nano claude code-like agent harness, built from 0 to 1" | gh api |
| 星数 / fork | 79092 / 约 12.7k（gh api 实测 79582，2026-10-27） | gh api |
| 主语言 | Python（教学代码）；配套 web 平台为 TypeScript/Next.js | gh api、web/package.json |
| 许可证 | MIT（Copyright (c) 2024 shareAI Lab） | LICENSE |
| 创建 / 最近推送 | 2025-06-29 / 2026-10-27 | gh api |
| 主页 | https://learn.shareai.run | gh api、README-zh.md |
| 章节规模 | 新版 17 章（s01-s17）+ 旧版 12 章过渡（docs/、agents/） | README-zh.md、顶层目录 |
| 代码量 | 除 web 外 Python 共约 25154 行，其中 17 章 code.py 合计约 12990 行 | wc 统计 |

## 2. 项目是什么，解决什么问题

这不是一个可部署的 agent 产品，而是一个**教学型源码仓库**：把 Claude Code 的架构拆成 17 个可独立运行的 Python 文件，逐章重建 agent harness 的每个机制。核心理念写在 README-zh.md 开篇——"Agency 来自模型，Agent 产品 = 模型 + Harness"：智能是模型训练出来的，代码只负责造"载具"（工具、知识、上下文管理、权限边界），本仓库教的就是造载具。

课程主线按能力递进组织（README-zh.md"学习路径"一节及目录名）：

- **s01-s04 基础**：agent loop（一个 `while True` + bash 工具）、工具注册表、权限审批管线、hooks 扩展点；
- **s05-s09 复杂任务与记忆**：TodoWrite 计划、子 agent 上下文隔离、按需 skill 加载、四步上下文压缩、跨会话记忆；
- **s10-s13 长期运行与协作**：磁盘持久化任务图、后台线程任务、cron 定时调度、多 agent 团队（原子认领 + git worktree 隔离）；
- **s14-s17 扩展与收口**：MCP 外部工具接入、s15 把所有机制集成为完整 harness（3330 行）、s16 workflow 编排与 journal 续跑、s17 目标闭环（独立评估器决定何时停止）。

每章一个文件夹，含英文/中文/日文三语 README、独立可运行的 `code.py` 和 SVG 图示（全仓 153 张 SVG）。仓库同时保留旧版 12 章体系（`docs/`、`agents/`）供老读者过渡，README-zh.md 明确给出新旧章节对应表并提醒不要混用章节号。

教学之外它也是 shareAI-lab 的生态入口：README 结尾导出姊妹产品 Kode Agent CLI（`npm i -g @shareai-lab/kode`）、Kode Agent SDK，以及姊妹教程仓库 claw0（常驻型 agent harness）。

## 3. 技术栈与依赖

- **Python 3.11+**（CI 指定版本，.github/workflows/test.yml）：唯一的运行时主线，刻意只依赖三个包（requirements.txt）：
  - `anthropic>=0.25.0` — 调用 Claude / Anthropic 兼容 API（消息、tool_use 块）；
  - `python-dotenv>=1.0.0` — 加载 `.env` 中的 API key 与 MODEL_ID；
  - `pyyaml>=6.0` — skill/配置 frontmatter 解析（s07、s15）。
- **零框架教学代码**：不用 LangChain/编排库，全部用标准库（subprocess、threading、asyncio、dataclasses、pathlib 等）实现机制——这本身就是仓库的教学主张（README-zh.md"Agent 不是什么"一节）。
- **多供应商兼容**：.env.example 内置 MiniMax、GLM（智谱）、Kimi、DeepSeek 的 Anthropic 兼容端点与 MODEL_ID 对照表，可通过 `ANTHROPIC_BASE_URL` 切换。
- **Web 平台**（web/package.json）：Next.js 16.1.6 + React 19.2 + Tailwind CSS 4 + TypeScript 5，依赖 unified/remark/rehype 生态做 Markdown 渲染，`npm run extract` 从根目录课程文件提取内容生成站点。

## 4. 目录结构与代码组织

| 目录 | 职责 | 出处 |
|---|---|---|
| `s01_agent_loop/` … `s17_goal_loop/` | 新版 17 章主线，每章 README×3 语言 + code.py + images/ | 顶层 ls、README-zh.md |
| `agents/` | 旧版 12 章可运行副本（s01-s12 单文件 + s_full.py 全量版） | agents/ 目录 |
| `docs/` | 旧版 12 章文档，分 en/zh/ja 三个语言目录 | docs/ 目录 |
| `skills/` | s07 技能加载章的素材：agent-builder、code-review、mcp-builder、pdf 四个 SKILL.md | skills/ 目录 |
| `tests/` | 21 个 pytest 测试文件，约 200 个测试函数 | tests/ 目录、grep 统计 |
| `web/` | 教学网站（Next.js），构建时从根目录章节提取内容 | web/package.json、web/scripts/ |
| `.github/workflows/` | ci.yml（web 构建）、test.yml（pytest + web 构建） | .github/ |

组织方式评述：**按章节横向切分的"每章一世界"结构**，而非常规的分层包结构。每章 `code.py` 完全自包含（自己 import、自己配 client），章节间靠复制演进代码而非 import 复用——这是教学仓库的刻意取舍（CONTRIBUTING.md 明文要求"keep teaching code minimal"）。代价是 s15 集成章膨胀到 3330 行；好处是任意一章都能单独打开读懂、单独运行。

## 5. 核心架构与关键实现

### 5.1 最小 agent loop（s01_agent_loop/code.py，全文 154 行）

整个课程的"细胞"。`agent_loop()` 就是 README 里的核心模式：调模型 → 提取 `tool_use` 块 → 执行 bash → 结果作为 `tool_result` 塞回 messages → 再循环；模型不再调工具即结束。附带三处工程细节：危险命令黑名单（`rm -rf /`、`sudo` 等）、120 秒超时与 5 万字符截断（`run_bash`，s01_agent_loop/code.py:80-92）、Windows 下系统提示改为 cmd.exe 语法（s01_agent_loop/code.py:56-61）。

### 5.2 四步上下文压缩（s08_context_compact/code.py，`ContextCompactor` 类）

长会话腾地方的四道闸门（类方法 s08_context_compact/code.py:254-493）：`tool_result_budget` 给超大的工具结果落盘并留桩；`snip_compact` 把旧的中间历史归档到 `.transcripts/`；`micro_compact` 保存并缩短更旧的工具结果；最后 `summary` 让模型生成对话摘要替换历史。最近一次提交（2026-10-27，#603）还在修"压缩后保留完整活跃请求"的边界问题，说明这块仍在打磨。

### 5.3 集成 harness（s15_integrated_harness/code.py，3330 行，全仓最大单文件）

把前面所有机制接回一个循环的"毕业作品"：磁盘持久化任务系统（`Task` dataclass + `.task_outputs/` 落盘 + 依赖检查 `can_start`/原子认领 `claim_task`，s15:205-435）、任务绑定的 git worktree 隔离（`create_worktree`/`task_worktree_cwd`，s15:438-657）、skill 扫描与按需加载（`scan_skills`/`load_skill`，s15:724-846）、权限 hook、后台线程任务、团队协作（`ConsoleBroker`）、MCP 工具池，外加生产味细节：`FALLBACK_MODEL_ID` 模型回退、连续 529 错误熔断、3 次指数退避重试、atexit 与信号处理清理子进程组（s15:68-101 常量区与 s15:886-930）。

### 5.4 目标闭环（s17_goal_loop/code.py，903 行，终点章）

给会话挂一个 session 级 Stop hook：worker 模型不再调工具想停时，`GoalController` 调独立的 `PromptGoalEvaluator` 读完整对话判断目标是否达成，未达成则 block 并自动续轮；目标不可能、执行失败或超过续跑上限才交还用户（模块 docstring 与 `GoalController`/`PromptGoalEvaluator`，s17:218-541）。配套 `contains_destructive_command` 防护与 usage 统计（`_usage_total`，s17:120）。

## 6. 构建与运行

来自 README-zh.md"快速开始"与 .env.example：

```sh
git clone https://github.com/shareAI-lab/learn-claude-code
cd learn-claude-code
pip install -r requirements.txt
cp .env.example .env        # 填 ANTHROPIC_API_KEY 与 MODEL_ID（默认 claude-sonnet-4-6）

python s01_agent_loop/code.py        # 起点章
python s17_goal_loop/code.py "/goal pytest tests 退出码为 0"   # 终点章可带目标参数

cd web && npm install && npm run dev   # 教学网站 http://localhost:3000
```

测试：`python -m pytest tests -q`（CI 同款命令，.github/workflows/test.yml）。web 内容重建：`npm run extract`（web/package.json scripts）。

## 7. 测试与工程质量

- **测试规模**：21 个测试文件、约 200 个测试函数，pytest 框架（tests/ 目录、grep 统计）。
- **测试风格**：不调真实 API 的 smoke/边界测试。典型手法见 tests/test_s08_context_compact.py:1-30——用 monkeypatch 把 `anthropic`、`dotenv` 模块整体替换为 fake，`runpy.run_path` 加载章节 `code.py` 后对纯逻辑（压缩边界、任务依赖、权限命令词、UTF-8 I/O、worktree 名校验等）断言。覆盖面广且贴合教学代码：test_chapter_readmes.py 甚至校验章节 README 的一致性。
- **CI**：双 workflow（.github/workflows/）。test.yml 跑 Python 3.11 pytest + web build；ci.yml 跑 web 的 `tsc --noEmit` + `next build`。actions 用 commit SHA 锁定版本。
- **质量观察**：代码刻意保持极简（CONTRIBUTING.md 明文禁止给教学代码加生产加固，"看起来像 bug 的简化往往是故意的"）；i18n 纪律严格——任何章节改动需三语 README 同步（CONTRIBUTING.md 第 3 条）。s15 单文件 3330 行是可读性上的主要妥协。

## 8. 社区与维护状态

- **非常活跃**：最近推送 2026-10-27（gh api），近 30 天 15 次提交（gh api commits?since 统计），最近一批提交全部是社区 PR 合入（#584-#603）。
- **贡献者**：30 人（gh api contributors），头部为 Bill-Billion（78 次提交）、CrazyBoyM（35）、Gui-Yue（34），呈现"少数核心维护者 + 持续社区 PR"结构。
- **无 release**：releases 数为 0（gh api），课程内容直接随 main 演进，无版本化发布节奏。
- **issue 治理**：51 个 open issue；CONTRIBUTING.md 设有"不细审直接关闭"清单以控制队列。
- **三语生态**：README 与每章文档均有英/中/日三语版本（顶层 README*.md、各章 README.*.md），是其星数快速增长的重要放大器（未确认具体转化数据）。

## 9. 亮点与风险

**亮点**

1. 教学设计极强：17 章每章只加一个机制、每机制一句格言，代码自包含可独立运行，从 154 行的 loop 平滑长到 3330 行的完整 harness（README-zh.md 章节表、各章 code.py）。
2. 真实工程细节密度高：worktree 隔离、原子任务认领、模型回退、529 熔断、Windows/cmd.exe 兼容、readline UTF-8 修复（#143 注释）等，不是玩具代码。
3. 三语同步文档 + 153 张 SVG 图示 + 配套教学网站（learn.shareai.run），学习体验完整。
4. 依赖面极小（3 个包），且给出 GLM/Kimi/DeepSeek 等 Anthropic 兼容端点对照表，中国用户可低成本跑通。
5. 测试与 CI 齐备，200 个测试函数守护教学代码的边界行为。

**风险/局限**

1. 教学代码与生产代码边界需自觉：CONTRIBUTING.md 明说简化是故意的，直接把 s15 拿去当生产 harness 会有坑（无真正沙箱、权限仅命令词黑名单级别）。
2. 强绑定 Anthropic 消息格式（tool_use/tool_result 块），.env.example 自己也注明"Anthropic-compatible 不保证行为一致"。
3. 章节 code.py 之间是复制演进，上游机制改动需多处同步，维护成本随章节数线性涨（s15 已 3330 行）。
4. 运行需自备 API key，按 token 计费；s08/s15 等复杂章对新手是真实的成本门槛（未确认具体消耗量）。

## 10. 总结评价

这是"agent harness 工程"这个新工种目前最好的开源教材之一：它把 Claude Code 的黑盒拆成 17 个能跑、能读、能改的 Python 文件，用极强的观点输出（agency 来自模型、harness 工程师身份叙事）和完整的三语内容体系撑起了近 8 万星——星数本质是为"课程质量 + 内容传播"付费，而非为软件功能付费。维护方（shareAI-lab）活跃、社区 PR 持续合入、生态（Kode CLI/SDK、claw0）明确，短期不会失修。适合两类人：想真正理解 Claude Code 内部机制、动手造自己 coding agent 的工程师（按序读 s01-s17 并跑通每章）；以及研究 agent 教学如何做内容与社区运营的维护者。不适合想找一个开箱即用 agent 框架的人——那该看它的姊妹产品 Kode-CLI，或 Anthropic 官方 SDK。
