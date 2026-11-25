# HKUDS/CLI-Anything 项目分析

> 星数 56455（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-11-03
>
> 注：分析时经 `gh api repos/HKUDS/CLI-Anything` 复核，当前星数已达 58120，最近 push 2026-09-22。

## 1. 项目速览

一句话定位：**把任意专业软件变成 AI Agent 可直接驱动的 CLI 工具**——一个"生成器插件 + 社区 CLI 仓库 + 包管理器"三位一体的 monorepo，口号是 "Making ALL Software Agent-Native"。

| 项目 | 事实 |
|------|------|
| 名称 | HKUDS/CLI-Anything（HKU 数据智能实验室出品） |
| 星数 | 56455（快照）/ 58120（分析时） |
| 语言 | Python（要求 ≥3.10，Click ≥8.0，见 `README.md` 徽章区） |
| 许可证 | Apache-2.0（`LICENSE`） |
| 创建时间 | 2026-03-08（gh api） |
| 最近活跃 | 2026-09-22（gh api pushed_at） |
| 规模 | 仓库约 42MB，tarball 2583 个条目，83 个顶层目录 |
| 贡献者 | GitHub 贡献者约 121 人（gh api 分页推算）；registry.json 内署名 60 人 |
| 技术报告 | arXiv:2606.03854《CLI-Anything: Towards Agent-Native Computer Use》（`CITATION.cff`） |

## 2. 项目是什么，解决什么问题

AI Agent 擅长推理但不擅长操作为人类设计的专业软件：截图+点击式的 GUI 自动化脆弱、像素级 RPA 极易失效，而 API 覆盖面又往往不足（README.md "The Agent-Software Gap" 一节）。CLI-Anything 的解法是：CLI 是人类与 LLM 的"通用接口"——结构化、可组合、`--help` 自描述、JSON 输出可直接被 Agent 消费（README.md "Why CLI?" 一节）。项目因此把专业软件的完整能力封装成有状态的 CLI harness，让 Claude Code、Cursor、Pi、OpenClaw 等编码 Agent 通过纯命令行驱动 Blender 渲染 3D、FreeCAD 建模、LibreOffice 转 PDF、Shotcut 剪视频等真实软件后端（README.md "Authentic Software Integration"）。

仓库本身由三部分组成，构成一个完整闭环（`README.md` "Project Structure"、`registry.json`）：

1. **生成器插件**（`cli-anything-plugin/`）：一个 Claude Code 插件市场包（另有 Cursor、Pi、Codex、OpenCode 等十余平台的分发形式），核心是 `HARNESS.md` 定义的 7 阶段 SOP，指导 Agent 自动分析任意 GUI 软件源码并产出生产级 CLI。
2. **社区 harness 仓库**：约 80 个按软件划分的顶层目录（blender、gimp、freecad、obs-studio、musescore、kdenlive、QGIS、godot、iterm2、slay_the_spire_ii 等），每个都是可 `pip install` 的独立包，统一挂在 `cli_anything.*` 命名空间下。
3. **CLI-Hub 生态**：PyPI 包 `cli-anything-hub`（`cli-hub/`，v0.4.1）作为包管理器，配合根目录 `registry.json`（79 个 CLI）与 `public_registry.json`（24 个第三方公共 CLI），`pip install cli-anything-hub` 后即可 `cli-hub install gimp` 一键安装；GitHub Pages 站点（`docs/hub/`）直接读 main 分支的 registry，PR 合入即刻生效（`CONTRIBUTING.md`）。

项目的增长逻辑非常清晰：以 issue 模板招募贡献者（`.github/ISSUE_TEMPLATE/contributor-signup.yml`、`cli-wishlist.yml`），每个新 CLI 的合入都同时扩充注册表、SKILL.md 技能库和 README 新闻墙，形成"生态飞轮"。README 的 News 区从 2026-03 到 2026-05 几乎每日都有新 harness 合入记录，星数在半年内冲到 56k+。

## 3. 技术栈与依赖

- **语言与框架**：Python ≥3.10；CLI 层统一使用 Click（`blender/agent-harness/cli_anything/blender/blender_cli.py` 等）；REPL 基于 prompt_toolkit（`cli-anything-plugin/repl_skin.py` 567 行，统一横幅/表格/进度条/彩色消息）。
- **后端集成**：`subprocess` + `shutil.which()` 调用真实软件（如 `blender --background --python <script>`，`blender/agent-harness/cli_anything/blender/utils/blender_backend.py`）；有 MCP server 但无 CLI 的软件走 MCP 后端模式（`cli-anything-plugin/guides/mcp-backend.md`）。
- **数据与安全**：会话状态为 JSON 文件，用 fcntl 独占锁防并发写坏（`blender/agent-harness/cli_anything/blender/core/session.py` 的 `_locked_save_json`）；不可信 XML/SVG/ODF 输入统一走 `defusedxml`（README News 2026-05-19）。
- **打包分发**：每 harness 一个 `setup.py`，支持 `pip install git+...#subdirectory=<name>/agent-harness`（registry.json install_cmd）；CLI-Hub 的 wheel 构建时把 `cli-hub-matrix/` 内容 vendor 进包（`cli-hub/setup.py`）；PyPI 自动发布用 trusted publishing（`.github/workflows/publish-cli-hub.yml`）。
- **Agent 技能分发**：SKILL.md 规范（YAML frontmatter + 命令组 + 示例），通过 `npx skills add HKUDS/CLI-Anything --skill <name>` 安装；`cli-anything-plugin/skill_generator.py`（603 行）从 Click 装饰器/setup.py/README 自动提取元数据生成 SKILL.md。

## 4. 目录结构与代码组织

| 顶层目录 | 职责 |
|----------|------|
| `cli-anything-plugin/` | 生成器插件本体：HARNESS.md（7 阶段 SOP）、commands/（build/refine/test/validate）、guides/（9 篇方法论指南）、repl_skin.py、skill_generator.py、preview_bundle.py |
| `blender/`、`gimp/`、`freecad/` 等 ~80 个目录 | 每个 `<software>/agent-harness/` 是一个独立 pip 包（`cli_anything/<software>/`） |
| `cli-hub/` | PyPI 包管理器 `cli-anything-hub`：cli.py（1048 行）、installer.py（604 行）、registry/analytics/matrix 等 |
| `cli-hub-matrix/` | 5 个任务域"能力矩阵"内容（3d-cad、video-creation、game-development、image-design、knowledge-research），构建时 vendor 进 cli-hub |
| `skills/` | 69 个 `cli-anything-<software>/SKILL.md`，全仓库技能的唯一权威来源 |
| `docs/hub/` | GitHub Pages 前端 + `llms.txt`/`llms-full.txt`（面向 Agent 的文档）+ `.well-known/` agent 清单 |
| `macrocli/` | 混合式 harness：backends 含 gui_agent/gui_macro/native_api/semantic_ui 等，CLI 优先、GUI 兜底 |
| `unimol_tools/`、`3MF/`、`cloudanalyzer/` 等 | 科学计算/领域类 harness |
| 根目录 JSON | `registry.json`（79 CLI，31 类别）、`public_registry.json`（24 个第三方）、`matrix_registry.json`（5 个矩阵） |
| `.github/` | 7 个 workflow、PR/issue 模板、自动化脚本（sync/validate_root_skills.py、generate_meta_skill.py、update_registry_dates.py） |

组织方式评述：这是典型的"**每软件一子目录的多包 monorepo**"，但比常规 monorepo 更进一步——每个子目录是完整独立发行单元（含 setup.py、tests、SKILL.md、SOP 文档），根目录只放共享规范（HARNESS.md、repl_skin.py）与索引（registry、skills/）。代码强制模板化：所有 harness 必须内嵌未经修改的 `repl_skin.py` 副本并由 CI 校验（`.github/workflows/check-root-skills.yml` + `.github/scripts/validate_root_skills.py`），以此在数百贡献者的松散协作下保证体验一致性。

## 5. 核心架构与关键实现

### 5.1 七阶段生成管线（`cli-anything-plugin/HARNESS.md`，747 行）

这是整个项目的"宪法"：Phase 1 代码库分析（找后端引擎、映射 GUI 动作到 API、盘点撤销/命令模式）→ Phase 2 架构设计（REPL/子命令双模、命令组、`--json` 双输出）→ Phase 3 实现（数据层→probe→mutation→backend→渲染→会话→REPL）→ Phase 4/5/6 测试计划/实现/文档 → Phase 6.5 SKILL.md 生成 → Phase 7 PyPI 发布。关键设计原则是"用真实软件，不重新实现"：

```python
# HARNESS.md "Use the Real Software" 一节的模式示例
# 1. Build the document as a valid ODF file (our XML builder)
# 2. Convert via the REAL LibreOffice (not a reimplementation)
lo = find_libreoffice()  # raises RuntimeError with install instructions
subprocess.run([lo, "--headless", "--convert-to", output_format, ...])
# Result: a real PDF rendered by LibreOffice's full engine
```

### 5.2 标准 harness 形态（以 `blender/agent-harness/` 为例）

入口 `blender_cli.py`（1157 行）是 Click 组：无子命令时自动进 REPL（`invoke_without_command=True`），`--json` 全局开关由 `output()` 统一分流人类可读/机器可读输出。`core/` 按领域拆分 scene/objects/materials/modifiers/lighting/animation/render/preview/session 九个模块；`utils/blender_backend.py` 负责定位可执行文件并以 `blender --background --python` 跑 bpy 脚本完成真实渲染。核心思路是 **JSON 场景描述 + bpy 脚本代码生成**：Agent 操作内存中的场景模型，导出时生成 Python 脚本交给 Blender 执行。

### 5.3 有状态会话与 undo/redo（`core/session.py`）

所有 harness 共享同一会话模式：JSON 项目状态 + 50 级 undo/redo 栈 + fcntl 文件锁原子落盘，避免并发写坏会话文件（`blender/.../core/session.py`，`MAX_UNDO = 50`；锁实现源自 `cli-anything-plugin/guides/session-locking.md` 的 `_locked_save_json` 模式——先 `open("r+")`，加锁，锁内 truncate 再写）。

### 5.4 CLI-Hub 包管理器（`cli-hub/cli_hub/`）

`cli.py`（1048 行）实现 list/search/info/install/update/uninstall/launch 七个动词；`installer.py` 按 registry 的 install_cmd 分发 pip/npm/brew 三类来源，安装状态记录在 `~/.cli-hub/installed.json`（`installer.py` 顶部常量）。`matrix.py` + `matrix_skill.py` 支持按"能力矩阵"安装整套任务工具链——例如 video-creation 矩阵把 harness CLI、公共 CLI、Python 库、本地二进制、Agent 技能与云 API 组合成端到端流水线（`matrix_registry.json` 第 1 条）。

### 5.5 面向 Agent 的分发面

除 SKILL.md 外，`docs/hub/llms.txt`、`llms-full.txt` 与 `.well-known/agent.json`、`agent-skills/index.json` 让 Agent 可自主发现目录；`cli-hub-meta-skill/` 是装进 Agent 的"元技能"，提示词只需 "Find appropriate CLI software in CLI-Hub and complete the task: ..."（README.md "Empower Your Agents with CLI-Hub"）。

## 6. 构建与运行

来自 README 与各工程文件的实际命令：

```bash
# 用现有生态（README.md Quick Start Phase 1）
pip install cli-anything-hub
cli-hub list && cli-hub search image
cli-hub install gimp && cli-hub launch gimp

# 给 Agent 装元技能（Phase 1.5）
npx skills add HKUDS/CLI-Anything --skill cli-hub-meta-skill -g -y

# 生成新 CLI（Claude Code 插件，Phase 3）
/plugin marketplace add HKUDS/CLI-Anything && /plugin install cli-anything
/cli-anything /path/to/gimp            # 七阶段全流程
/cli-anything:refine /path/to/gimp "batch filters"   # 定向扩充

# 单个 harness 的开发安装（各 <software>/agent-harness/setup.py）
pip install -e ./blender/agent-harness
pip install git+https://github.com/HKUDS/CLI-Anything.git#subdirectory=blender/agent-harness

# 单 harness 测试（约定：tests/test_core.py 不需后端，test_full_e2e.py 需真实软件）
pytest blender/agent-harness/cli_anything/blender/tests/
```

## 7. 测试与工程质量

- **规模**：全仓 168 个测试文件，grep 统计约 5460 个 Python 测试函数；57 份 `TEST.md` 测试计划（每 harness 强制先写计划再写代码，HARNESS.md Phase 4）。README 徽章自述 2,461 项测试通过（口径可能只统计部分 harness，与全仓 grep 数字差异较大，未确认精确口径）。
- **分层**：单元测试（合成数据、无需装目标软件）+ E2E（真实软件出真实产物）+ CLI 子进程级验证（HARNESS.md "Testing Strategy"；blender 有独立的 `test_bpy_gen.py`/`test_core.py`/`test_full_e2e.py` 三层）。
- **CI**（`.github/workflows/`，7 条）：root-skills 镜像校验、Codex/Cursor 插件一致性检查、PR 自动打标（含打标器的独立测试 `pr-labeler.test.js`）、registry 日期自动更新、`publish-cli-hub.yml` 在 cli-hub/** 变更时查 PyPI 版本号并 trusted-publish。
- **质量观察**：模板化程度极高，安全响应迅速——Path traversal/symlink 逃逸加固（News #304）、defusedxml 全面上料（#296）、registry 不可变性修复（#281）均有记录；但质量依赖贡献者自查，各 harness 深度参差（如 freecad_cli.py 达 4911 行，而部分小 harness 仅数百行）。

## 8. 社区与维护状态

- **活跃度**：2026-03-08 建仓，半年 56k+ 星（Trendshift #22991、Star History Trending 每日徽章，README 头部）；最近 push 2026-09-22（gh api），tarball 内所有文件 mtime 均为该日，此后约 6 周未再提交，237 个 open issue（gh api）。
- **发布节奏**：PyPI 侧 cli-anything-hub 已迭代到 v0.4.1（`cli-hub/setup.py`），harness 各自独立版本（registry.json 中多为 0.1.0-0.4.x）；GitHub Releases 仅 3 个（gh api），主要发行渠道是 PyPI + npm skills + 插件市场，而非 GitHub Release。
- **贡献者结构**：GitHub 贡献者约 121 人；registry.json 署名 60 人，条目级 contributors 字段把每个 CLI 归属到具体社区作者；HKUDS 组织成员（如 chaohuang-ai）负责合并与 README 维护（gh api 最近 5 条 commit 均为组织方合并/改 README）。
- **招募机制**：contributor-signup / cli-wishlist 两个专用 issue 模板构成"需求池→认领→合入→署名"流水线，是星数与 harness 数量双增长的核心引擎。

## 9. 亮点与风险

**亮点**

1. **方法论即产品**：HARNESS.md 把"如何为任意软件写 Agent CLI"沉淀为可复制的 7 阶段 SOP + 9 篇专题指南，生成器本身是"给 Agent 看的文档工程"，这一资产比任何单个 harness 都有价值（`cli-anything-plugin/HARNESS.md`）。
2. **真实软件集成而非玩具实现**：渲染/转换一律调用真实后端（Blender/LibreOffice/sox/FFmpeg），配合 preview/live preview 验证产物（`HARNESS.md` "Use the Real Software"、`cli-anything-plugin/preview_bundle.py`）。
3. **完整的 Agent 分发面**：PyPI 包管理器 + npx skills + 多平台插件 + llms.txt + 元技能，Agent 从发现、安装到使用全程无需人工介入。
4. **强模板 + CI 守护的一致性**：repl_skin 统一交互、SKILL.md 双份同步校验、注册表即索引，79 个异构 harness 保持了统一的使用范式。
5. **生态飞轮设计**：wishlist 模板、registry 即时生效、贡献者署名，把社区增长机制内建进了仓库结构。

**风险/局限**

1. **维护压力随规模陡增**：约 80 个 harness × 各自的目标软件版本，issue 已积压 237 个；主仓自 2026-09-22 后约 6 周无新提交（gh api），后续节奏未确认。
2. **测试数字口径不一**：README 徽章"2,461 passing"与全仓 grep 的 5460 个测试函数差距大，各 harness 覆盖深浅不齐，多数 E2E 依赖本机装有真实软件。
3. **安全面宽**：harness 大量 subprocess 调外部软件并处理不可信文件，历史上已有多轮加固（#304/#296/#281），但长尾社区贡献的审计难度随数量增长。
4. **单一组织主导**：模板、registry、发布渠道均由 HKUDS 掌控，standalone 仓库模式虽有（registry-only PR），但元数据与分发入口仍集中在主仓。

## 10. 总结评价

CLI-Anything 是 2026 年"Agent-Native 软件"赛道最具代表性的工程化项目：它没有停留在"CLI 工具合集"层面，而是同时交付了**生成方法论（HARNESS.md）、生成工具（插件）、分发网络（CLI-Hub/skills/注册表）和社区工厂（模板+CI+招募流水线）**四层资产，用半年时间把 80 个专业软件接入 Agent 世界，星数从 0 冲到 58k+。其代码本身不复杂（大量模板化的 Click + subprocess），真正的护城河是规范、流程与生态位。适合三类人：想让 Agent 操作 Blender/GIMP/FreeCAD 等真实软件的 AI 重度用户（直接 `pip install cli-anything-hub`）；研究"如何为 LLM 设计工具接口"的工程师（精读 `cli-anything-plugin/HARNESS.md` 与任一 harness 的 tests/SKILL.md）；想给自己的软件做 Agent 接入的厂商（参照 standalone 仓库的 registry-only 模式）。读源码的性价比最高路径：HARNESS.md → repl_skin.py + session.py → blender 或 freecad 的 cli 与 backend → cli-hub/installer.py。
