# msitarzewski/agency-agents 项目分析

> 星数 163440（快照 2026-10-24，查询时 165129）｜主语言 Shell｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：一个"AI 代理人才库"——收录 283 个精心撰写人格设定（persona）的 AI 代理 Markdown 文件，按 18 个业务部门（division）组织，可一键安装到 Claude Code、Cursor、Codex、Gemini CLI 等 16 种主流 AI 编码工具。

| 项目 | 值 |
|---|---|
| 名称 | msitarzewski/agency-agents（The Agency） |
| 星数 | 163,440（快照）/ 165,129（2026-10-30 查询，gh api） |
| Fork | 26,672（gh api） |
| 主语言 | Shell（GitHub 统计；实际主体是 Markdown 内容 + Bash 工具链） |
| 许可证 | MIT（LICENSE） |
| 创建时间 | 2025-10-13（gh api） |
| 最近 push | 2026-10-25（gh api，分析日前几天仍活跃） |
| 内容规模 | 283 个代理文件、18 个部门（自脚本统计） |
| 贡献者 | 130+（contributors API） |

## 2. 项目是什么，解决什么问题

The Agency 起源于一个 Reddit 帖子（README.md 第 21 行自述"Born from a Reddit thread"），本质是一个**大型 AI 代理提示词（系统提示/persona）精选库**。它解决的问题是：通用 AI 编码助手（Claude Code 等）拿到的是泛化人格，缺少针对具体岗位（前端、数据库、SRE、UI 设计、SEO、财务分析……）的领域知识结构与工作流程。项目为每个"岗位"手写一份代理定义，包含身份与记忆设定、核心任务、必须遵守的规则、可交付物模板、成功指标与沟通风格，让用户"按需雇佣"一个虚拟专家团队。

每个代理文件是一个带 YAML frontmatter 的 Markdown 文件。以 `engineering/engineering-frontend-developer.md`（230 行）为例：frontmatter 定义 `name/description/color/emoji/vibe`（可选 `services` 声明外部依赖），正文分为 "Identity & Memory"、"Core Mission"、"Critical Rules"、"Technical Deliverables"、"Success Metrics"、"Communication Style" 等固定段落。这是内容工程（prompt engineering）而非可执行代码——代理的"能力"完全由提示词文本表达。

使用方式有四种（README.md Quick Start）：(1) 安装官方桌面 App（独立仓库 agency-agents-app，支持 macOS/Linux/Windows，brew cask 安装）；(2) `./scripts/install.sh --tool claude-code` 把代理复制到 `~/.claude/agents/`，会话中说"activate Frontend Developer mode"即可切换；(3) 直接阅读复制改写；(4) `./scripts/convert.sh` + `install.sh` 转换并安装到其他 15 种工具。

部门分布（按各目录 .md 文件统计）：engineering 66、specialized 59、marketing 37、gis 13、security 12、design 10、sales 9、testing 9、paid-media 7、project-management 7、academic 6、product 6、spatial-computing 6、support 6、finance 5、game-development 21（含引擎子目录）、healthcare 3、research 1。工程与营销类内容最厚，新兴领域（research、healthcare）尚单薄。

项目还提供 `strategy/` 目录下的 "NEXUS" 编排层：7 个阶段的 playbooks（phase-0-discovery 到 phase-6-operate）、4 套 runbooks（startup-mvp 18 个代理、enterprise-feature 22 个等，`strategy/runbooks.json`）、代理交接与激活话术模板（`strategy/coordination/`），把单代理扩展成"多代理团队协作"的预设剧本。`examples/` 提供 4 个端到端工作流示例（如 workflow-startup-mvp.md）。

## 3. 技术栈与依赖

- **内容层**：Markdown + YAML frontmatter（代理定义的唯一载体），无任何运行时依赖。
- **工具链层**：Bash（GitHub 语言统计中占主导，故主语言显示为 Shell）——`scripts/install.sh`（约 75KB）、`scripts/convert.sh`（约 30KB）、共享库 `scripts/lib.sh`；少量 Python3 辅助脚本（`scripts/build-hermes-plugin.py`、`scripts/check-hermes-config-rewrite.py`、`scripts/test-hermes-plugin.py`）。
- **目录即目录服务的约定**：根目录 `divisions.json` 是部门清单的 single source of truth（label/Lucide 图标/品牌色），`tools.json` 是 16 种目标工具的安装契约（id、检测目录、dest 模板、render format、installKind: per-agent/roster/plugin），两者都由 CI 脚本与 shell 数组交叉校验（见两个文件的 `_note` 字段）。
- **多工具落地形态各异**（integrations/README.md）：Cursor 用 `.mdc` 规则文件、Codex 用 `.toml`、Aider 用单个 CONVENTIONS.md roster 索引、Windsurf 用单个 `.windsurfrules`、OpenClaw 生成含 SOUL.md 的完整工作区、Hermes 是 lazy-router 插件——同一份内容按各工具宿主约定重新封装。
- **CI**：GitHub Actions，6 个 workflow（`.github/workflows/`：lint-agents.yml、check-divisions.yml、check-tools.yml、check-runbooks.yml、check-hermes-config-rewrite.yml、test-install.yml）。
- 无 package.json/pyproject 等工程清单；依赖仅为 bash、coreutils、可选 python3、jq（未确认，脚本内未强制）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `academic/ design/ engineering/ finance/ game-development/ gis/ healthcare/ marketing/ paid-media/ product/ project-management/ research/ sales/ security/ spatial-computing/ specialized/ support/ testing/` | 18 个代理"部门"，每目录若干 `<division>-<slug>.md` 代理文件；game-development 下还有 unity/、unreal-engine/、godot/、roblox-studio/、blender/ 引擎子目录（共 21 个代理） |
| `integrations/` | `convert.sh` 生成的各工具格式输出（cursor/.mdc、codex/.toml、aider/CONVENTIONS.md 等 17 个子目录），属生成物不手改（divisions.json `_note` 明确排除其为部门） |
| `scripts/` | Bash/Python 工具链：安装、转换、lint、一致性检查、originality 查重、以及约 2284 行的 test-*.sh/.py 自测 |
| `strategy/` | NEXUS 编排：playbooks（7 阶段）、runbooks.json（4 套团队配置）、coordination 话术模板 |
| `examples/` | 4 个工作流示例 + README |
| `.github/` | 6 个 CI workflow、PR 模板、issue 模板、FUNDING.yml |
| 根文件 | README.md（86KB，含全部 283 个代理的索引表）、CONTRIBUTING.md（21KB 规范）、divisions.json、tools.json、LICENSE(MIT)、SECURITY.md |

新增一个代理的标准路径（CONTRIBUTING.md "Create a New Agent" 节）：fork → 选定或提议部门（改 divisions.json 并同步两处 AGENT_DIRS）→ 按模板写 md → 真实场景测试 → PR 走 CI 门禁。中英双语贡献指南（CONTRIBUTING_zh-CN.md）是为中文社区准备的。

组织方式评述：这是"内容 monorepo"形态——内容（代理 md）与内容供应链（转换/安装/lint 脚本、生成物、目录元数据）共处一库。官方 App 放在独立仓库（agency-agents-app），本仓库只通过 `tools.json` 声明契约供其消费，职责切分清晰。

## 5. 核心架构与关键实现

本仓库"代码"部分的价值在于**内容供应链工程**，三个核心模块：

### 5.1 转换器 convert.sh —— 一份内容、16 种格式

每个工具一个 `convert_<tool>()` 函数（`scripts/convert.sh:133` 起），共享 `lib.sh` 的 frontmatter 解析与 slug 工具。`tools.json` 的 `_note` 定义了关键契约："同一 `format` 名保证字节级一致的输出；`installKind` 分 per-agent（每代理一个文件）/ roster（全部合并一个文件）/ plugin（构建产物）"。转换支持 `--parallel --jobs N`，并有 `convert-outputs.sha256`（28KB）清单锁定生成物指纹，CI 中防漂移（提交记录 `chore(eval): regenerate the convert-outputs manifest` 印证该机制在运转）。

### 5.2 安装器 install.sh —— 交互向导 + 精确选装

`scripts/install.sh:136` 定义 `ALL_TOOLS=(claude-code copilot antigravity gemini-cli opencode openclaw cursor aider windsurf qwen zcode kimi codex osaurus hermes vibe dsh)`（16 种）。支持 `--tool/--division/--agent/--agents-file/--dry-run/--list teams|agents`，可按 runbook slug 导出团队清单批量安装（README.md 第 105-112 行给出 python3 一行提取脚本）。它只写用户配置目录，生成物则由 convert.sh 落在 `integrations/`，两者职责分离（convert.sh 头部注释明确"This script never touches user config dirs"）。

### 5.3 质量门禁：lint + originality + 一致性检查

`scripts/lint-agents.sh` 校验 frontmatter 必填项（name/description/color）与推荐章节（Identity/Core Mission/Critical Rules），颜色名直接从 `convert.sh` 的 `resolve_opencode_color()` 函数体里 awk 抽取而非复制，注释明言"so this can never drift"。`scripts/check-agent-originality.sh` 用"实体中性化的 8 词 shingle 重叠度"识别换皮重复代理（ swapping 国家/平台名无法隐藏抄袭），阈值 `ORIGINALITY_FAIL=40%` 拒绝、`20%` 警告——这是纯 Bash 实现的文本近似查重。`scripts/check-divisions.sh` 强制 divisions.json、两个脚本的 AGENT_DIRS 数组、lint workflow 的 path 过滤器四方一致。`lint-agents.yml` 还体现了少见的自省设计：当 lint 规则本身变更时，触发 `lint-all` 对全部 283 个代理重跑（workflow 内注释解释了颜色检查曾因此漏网的教训）。

### 5.4 内容本体示例

`engineering/engineering-frontend-developer.md` 的规则段（节选）：

```markdown
### Performance-First Development
- Implement Core Web Vitals optimization from the start
- Use modern performance techniques (code splitting, lazy loading, caching)
- Optimize images and assets for web delivery
- Monitor and maintain excellent Lighthouse scores
```

即以"角色 + 任务清单 + 硬性规则 + 交付物模板 + 成功指标"的结构化提示词组织，每个代理 100-300 行不等。

## 6. 构建与运行

无构建；全部命令来自 README.md 与 scripts/：

```bash
# 转换生成所有工具的集成文件（写入 integrations/<tool>/）
./scripts/convert.sh                 # 或 --tool cursor --parallel

# 安装到本机工具配置目录（交互向导）
./scripts/install.sh                 # 自动检测已装工具
./scripts/install.sh --tool claude-code --division engineering,security
./scripts/install.sh --tool cursor --agent frontend-developer,ui-designer
./scripts/install.sh --list teams    # 查看 runbook 团队

# 质量检查（CI 同款）
./scripts/lint-agents.sh
./scripts/check-divisions.sh
./scripts/check-agent-originality.sh

# 测试（shell 自测框架，24 个 test-* 文件）
bash scripts/test-install.sh
bash scripts/test-convert-outputs.sh
```

桌面 App（独立仓库）：`brew install --cask msitarzewski/agency-agents/agency-agents`。

## 7. 测试与工程质量

- 测试规模：`scripts/test-*.sh` + `test-*.py` 共 24 个文件、约 2284 行（wc 统计），全部围绕工具链——install（路径冲突、失败续跑、并行日志、hermes 目标）、convert（frontmatter、围栏模型、slug 冲突、并行失败、sha256 输出比对）、lint（折叠 frontmatter、Qwen 占位符）等边界场景。没有针对"代理提示词效果"的评测（内容本身无法自动评测）。
- CI 覆盖：6 个 workflow 分别守住 lint、部门一致性、工具契约一致性、runbook 合法性、hermes 配置改写、安装冒烟（test-install.yml）。CI 配置内注释密度高、有历史教训复盘，工程质量在同类内容仓库中罕见地高。
- 观察不足：代理内容之间术语/深度不统一属社区共建常态；"Success Metrics" 等段落是不可执行的愿景文本；对 LLM 行为无回归基准。

## 8. 社区与维护状态

- 活跃度极高：gh api 显示最近 push 为 2026-10-25（分析日前 5 天），近期提交如 `feat(engineering): add Electrical Engineer agent`（2026-10-20）、`Merge pull request #1177`（PR 编号已过 1177）。
- 贡献者结构：contributors API 返回 130+ 人（单页已超 100 条），核心维护者为 msitarzewski 本人，另有大量社区 PR（集成/content batch 类 chore 提交如 `integ/content-batch5-6` 显示有组织的批量内容流水线）。
- Release：本仓库无 release（gh api releases 为空）；发版载体是独立仓库 agency-agents-app（README 徽章指向其 releases）。"release 节奏"对本仓库不适用，内容以持续 merge 的滚动方式更新。
- 开放 issue 212 个；README 提及 OpenCode 上游 bug（运行时只注册约 119 个代理，README 第 114 行）并给出规避建议，说明维护者持续跟进各工具生态变化。
- 涨星动因（基于仓库证据推断）：踩中"AI 编码工具子代理（subagent）机制普及"的窗口期——Claude Code 等 2025 年起原生支持自定义 agents 目录，本仓库提供了现成、可浏览、可一键安装的内容供给；Reddit 起源带来初始流量，多工具适配与官方 App 把"看 README 抄提示词"的摩擦降到最低，26672 个 fork 也构成持续曝光。

## 9. 亮点与风险

**亮点**
1. 内容规模与覆盖面断层领先：283 个代理横跨 18 个部门，从软件工程到 GIS、医疗、游戏引擎、付费媒体，README 提供完整索引表，即查即用。
2. 供应链工程认真：一个"提示词合集"却具备 sha256 生成物清单、跨文件一致性 CI、字节级 format 契约（tools.json）、实体中性化查重脚本，把内容仓库做出了软件仓库的工程纪律。
3. 多工具兼容策略清晰：以 per-agent/roster/plugin 三类 installKind + format 契约解耦内容与 16 种工具的格式差异，新工具接入路径在 CONTRIBUTING/工具 `_note` 中写明。
4. 自省式质量文化：lint 规则变更触发全量重扫、颜色映射从源码抽取防漂移、CI 注释记录历史事故，维护心智值得借鉴。
5. 编排层加分：NEXUS playbooks/runbooks 把单代理库升级为按场景预组的"团队"，配合官方 App 降低安装门槛。

**风险/局限**
1. 提示词质量参差且不可度量：代理效果依赖底层模型与用户场景，仓库内无行为评测，"production-ready/battle-tested"（README 语）无法验证。
2. 内容膨胀与重复风险：engineering 已达 66 个代理，originality 检查只能拦文本级重复，拦不住语义重叠。
3. 上游依赖脆弱：16 种工具的代理格式持续变动（如 OpenCode 119 上限、hermes 配置改写检查的存在即是证据），维护负担线性增长。
4. Shell 工具链跨平台体验有限：Windows 原生环境下 bash 脚本路径/权限问题常见（官方答案是用 App）。

## 10. 总结评价

agency-agents 是 2025-2026 年"AI 代理 persona 库"赛道的事实性标杆：以 Reddit 一帖起家，一年内做到 16 万+ 星、2.6 万 fork、283 个代理、16 种工具适配，靠的不仅是内容量，更是把提示词内容当成软件资产管理——目录元数据、转换契约、生成物指纹、查重门禁、全量回归一应俱全。它适合作为 Claude Code/Cursor 等工具用户的即装即用专家库，更适合提示词工程与 AI 工具链开发者研读其供应链设计（`scripts/lint-agents.sh`、`scripts/check-agent-originality.sh`、`tools.json` 契约是精华）；想评估单个代理"好不好用"的读者则应降低预期——这里的"专家"是精心结构化的提示词，不是被验证过的智能体。
