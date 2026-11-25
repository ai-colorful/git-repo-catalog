# Yuan1z0825/nature-skills 项目分析

> 星数 52032（快照 2026-10-24，gh api 实测 52032）｜主语言 Python｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：面向科研全流程的 AI Agent 技能（Skills）集合——23 个围绕 `SKILL.md` 组织的可安装技能包，覆盖科研绘图、论文润色/写作、审稿模拟、文献检索/精读、论文转 PPT/Word/专利等场景，供 Claude Code、Codex、Chatbox 等各类编程智能体加载使用。

| 项目 | 事实 |
|---|---|
| 名称 | nature-skills（Nature Skills） |
| 星数 | 52,032（gh api，2026-11-04 实测）；fork 2,712 |
| 语言 | Python（标注语言；主体实为 Markdown 技能规则 + Python 配套工具，Python 约 4.9 万行） |
| 许可证 | Apache-2.0（`LICENSE`、README 徽章） |
| 创建时间 | 2026-04-24（gh api） |
| 最近推送 | 2026-11-02（gh api，分析时 2 天前） |
| 贡献者 | 39 人（gh api contributors） |
| 规模 | 884 个文件，其中 440 个 Markdown、221 个 Python；`skills/` 占 40MB |

## 2. 项目是什么，解决什么问题

nature-skills 是一组"可复用科研技能包"，每个技能是一个自包含目录，核心是 `SKILL.md`：由 frontmatter（`name`、`description`）声明触发条件，正文写给 AI Agent 阅读的工作流与规则。用户把论文、段落、审稿意见或任务描述交给装有该技能的智能体，即获得可用的科研产物——投稿级图件、润色文本、审稿报告、中英文对照 Markdown reader、文献汇报 PPT 等（`README.md` 第 4、6 节）。

技能覆盖科研工作闭环的多个环节（`README.md` 技能索引表）：写作侧有 `nature-polishing`（润色）、`nature-writing`（起草）、`nature-reviewer`（模拟三份互盲审稿报告）、`nature-response`（返修信与逐点回复）；阅读侧有 `nature-reader`、`nature-paper-xray`（精读复现作者推理）、`nature-paper-card`、`nature-paper-trans`（整本 PDF 逐页生图翻译）；产出侧有 `nature-figure`（Python/R 投稿级绘图）、`nature-paper2ppt`、`nature-paper2word`、`nature-image2ppt`（截图重建可编辑 PPT）、`nature-paper-to-patent`；文献侧有 `nature-academic-search`（多源检索含自建 MCP server）、`nature-ref-verifier`（参考文献逐字段交叉验证）、`nature-downloader`、`nature-literature-pipeline`（定时文献推送管线）。状态标签分 Draft/Beta/Stable 三档，当前 Stable 仅 4 个（figure、polishing、ref-verifier、literature-pipeline）。

项目明确强调"规则来自一手来源"：写作/图件规则基于 Nature 系期刊官方指南与已发表内容，而非泛泛审美偏好；每条规则说明理由；输出优先，强调返回 `.svg`、`.pptx`、`.docx` 等可直接使用产物（`README.md` 7.1 节共享设计原则）。

分发与运营是它区别于普通 prompt 库的关键：提供 `npx skills add` 按需安装、Claude Code subagent/slash wrapper 与 `~/.claude/skills` 自动同步脚本、Codex 同步脚本与 Codex 插件清单（`.codex-plugin/plugin.json`）、`.agents/plugins/marketplace.json` 市场清单、Chatbox 图形化安装，以及 OpenClaw/OpenCode/Hermes 接入文档（`docs/open-source-agent-frameworks.md`）。同时附带运营性质的网页版（`index.html` 单页站点，标题"Nature Skills — 面向全球学者的 AI 科研技能库"，在线版 natureskills.cn）。

README 中有大量创始人袁一哲的运营内容：GPT/Claude 代充网站、知识星球、抖音视频教程与微信社群（`README.md` 1.2-1.4 节）。3.1 节声称谷歌 DeepMind 参考其引用体系与技能设计哲学推出了 Science Skills（此为作者自述，未独立核实）。

## 3. 技术栈与依赖

- **主体：Markdown + YAML**。技能规则、期刊格式语料、审稿/写作纪律均以 Markdown 组织；`manifest.yaml` 声明路由轴与片段映射。主语言标注 Python 实际来自配套工具链。
- **Python 工具链（221 个文件，约 49,472 行）**：
  - 绘图审计：PyMuPDF 做 PDF 几何分析（文字/图形碰撞、多面板对齐、PDF 文字审计），见 `skills/nature-figure/scripts/audit_figure_collisions.py`；
  - 文档处理：python-docx、pdfplumber、pypdf、Pillow、lxml（`paper2word`、`image2ppt`、`paper-trans` 等）；
  - 检索：自建 MCP server（`skills/nature-academic-search/mcp-server/academic_search_server.py`），依赖 mcp、httpx、pybliometrics（Scopus）等；
  - 图像生成：OpenRouter GPT Image 2 API 封装（`skills/nature-figure/scripts/generate_openrouter_schematic.py`）。
- **R**：`nature-figure` 含 `scripts/panel_alignment.R`，支持 R 出图工作流。
- **CI 依赖锁定**：`.github/requirements-ci.txt` 提供 Python 3.11 全量固定版本的复现测试环境（PyMuPDF 1.28.2、mcp 1.29.0、pytest 9.1.1 等 48 项）。
- **运行时外部条件**：MCP 检索需配置 `PUBMED_EMAIL`；Scopus/ScienceDirect 需本机凭据；专利检索可选 Playwright + Chromium（`README.md` 5.3 节）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `skills/nature-*/` | 23 个技能包 + `nature-shared` 共享包，每个含 SKILL.md、双语 README、manifest.yaml、references/、scripts/ 等 |
| `skills/nature-shared/` | 跨技能共享语料：`core/*.md`（摘要、引言、结果讨论、术语台账、一致性清扫等纪律文件）、`journal-formats/`（nature、Nat Commun、NMI 格式）、`scripts/check_consistency.py` |
| `skills/nature-figure/` | 最大技能（34MB，其中 `assets/figures4papers` 28MB 第三方参考图，附 `THIRD_PARTY_NOTICES.md`）；含 scripts/、tests/、evals/ |
| `scripts/` | 仓库工程化脚本：9 个 validate-* 校验器、2 个安装同步 shell、星标历史生成、26 个测试文件 |
| `.github/workflows/` | 9 个 CI 工作流（内容契约测试、仓库元数据校验、README 镜像校验等） |
| `docs/` | paper-card 教程、开源 agent 框架接入、技能范围冒烟评估报告（中英双语） |
| `index.html` | 105KB 单页官网（含 i18n） |
| `assets/` | README 横幅、星标历史 SVG 等静态资源 |

组织方式：非代码库也非传统 monorepo，而是"技能包集合 + 工程化外壳"。每个技能自包含（新增技能不改动既有技能），共享内容下沉到 `nature-shared`；仓库级用一组自研校验器把 README 索引、技能元数据、双语镜像当作契约来维护。同时打包为 Codex 插件与 agent 市场清单（`.codex-plugin/plugin.json`、`.agents/plugins/marketplace.json`）。

## 5. 核心架构与关键实现

**1) Router 式技能架构（最核心设计）**。每个 `SKILL.md` 不是规则堆砌，而是路由器：`manifest.yaml` 声明分类轴（`paper_type`/`section`/`language`/`journal`）、取值及各值映射的规则片段文件；Agent 先加载 `always_load` 核心层，再按检测到的轴值按需加载对应片段，明确禁止一次读完 `static/` 全部内容（`skills/nature-polishing/SKILL.md` Routing protocol 一节）。这是典型的上下文经济学设计：单技能规则语料可达数百 KB，路由机制让 Agent 每次只吃进与当前论文类型、章节、目标期刊相关的切片，且要求把检测到的轴值先报告给用户以便低成本纠偏。

**2) 图件几何审计流水线**。`nature-figure` 不只给绘图模板，还内置交付前自动审计：`scripts/audit_figure_collisions.py` 用 PyMuPDF 从最终 PDF 几何层面检测文字-文字、文字-线条碰撞与文字出血，可靠碰撞阻断交付、部分重叠仅报告复核（该文件模块 docstring）；`audit_panel_alignment.py` 做渲染时子图对齐检查，`audit_svg_schematic.py` 审计矢量示意图。Python 与 R 统一以最终 PDF 为审计对象，与绘图后端解耦。

**3) 规则即契约的内容测试**。仓库对 Markdown 规则本身做单元测试：`scripts/tests/test_main_text_discipline.py` 断言 `skills/nature-shared/core/main-text-discipline.md` 中包含规定的结果分类（core_discovery、necessary_support、qualification 等七类）及放置纪律；类似的内容契约测试覆盖摘要、引言、结果讨论指南（`test_nature_abstract_guidance.py` 等）。代码测试则覆盖碰撞审计、面板对齐、OpenRouter 错误处理等（`scripts/tests/test_nature_figure_*.py`，26 个测试文件）。

**4) 元数据一致性校验**。`scripts/validate-repository.py` 校验 README 技能徽章数字与 `skills/` 实际目录数一致、索引链接可达、JSON/TOML 合法、每个技能的 SKILL.md/双语 README 齐全；`validate-readme-mirror.py` 强制中英 README 章节数量与顺序镜像；CI 中 `update-reference-tocs.py` 重新生成目录后 `git diff --exit-code` 确保未漂移（`.github/workflows/validate-repository.yml`）。

## 6. 构建与运行

无传统"构建"。安装与使用（均出自 `README.md` 第 5 节）：

```bash
# 方式一：npx skills（需 Node 18+），先列出可安装技能
npx skills add Yuan1z0825/nature-skills --list
# 按需安装（共享依赖需一并选择）
npx skills add Yuan1z0825/nature-skills --global --agent codex \
  --skill nature-reader --skill nature-shared --yes --copy

# 方式二：Codex 全量同步脚本
git clone https://github.com/Yuan1z0825/nature-skills.git && cd nature-skills
scripts/update-codex-skills.sh --pull        # --check 验证 / --prune 清理

# 方式三：Claude Code 自动同步（SessionStart 钩子）
scripts/autoupdate-skills.sh --force         # 配合 ~/.claude/settings.json 钩子

# Python 运行依赖按需安装
python -m pip install -r skills/nature-paper-to-patent/requirements.txt
python -m pip install -r skills/nature-academic-search/mcp-server/requirements.txt
```

本地校验与测试：

```bash
python scripts/validate-repository.py
python scripts/validate-readmes.py
python -m unittest discover -s scripts/tests      # 内容契约与工具回归
```

使用即对话触发，如"把这篇 PDF 做成图文对应的中英文对照 Markdown reader"；技能名可显式写入提示词（`README.md` 第 4 节）。

## 7. 测试与工程质量

- **测试规模**：仓库级 26 个测试文件（`scripts/tests/`）+ 根级 `tests/test_generate_star_history.py` + 技能内测试（`skills/nature-figure/tests/`、`skills/nature-shared/tests/test_check_consistency.py`、`nature-academic-search/mcp-server/tests/`）。含 unittest 与 pytest 两种风格。
- **测试特色**：罕见地对 Markdown 规则内容做契约测试（规则文本改动导致缺关键词即 CI 失败），把"技能质量"部分工程化；另有 `docs/skill-scope-smoke-evaluation.md` 记录 2026-09-06 六项任务的隔离 Agent 冒烟评估，声明仅为定性观察、非自动基准。
- **CI**：9 个工作流按路径触发——仓库元数据校验、技能索引/元数据校验、中英 README 镜像校验、工作流自身校验、内容契约测试、paper-trans 测试、skill 工具测试、星标历史自动更新（`.github/workflows/`）；actions 均按 commit SHA 锁版本。
- **质量观察**：Python 代码普遍类型注解齐全、模块 docstring 说明设计动机；CI 依赖全量 pin。README 与文档双语镜像由工具强制，文档纪律在同类项目中少见。短板是部分技能自评仍处 Draft/Beta，真正效果依赖宿主 Agent 遵循指令的能力。

## 8. 社区与维护状态

- **活跃度极高**：创建于 2026-04-24，约 6 个月达 52k 星（平均月增约 8.6k）；最近推送 2026-11-02（gh api，分析前 2 天），最新提交为 academic-search/citation 文档更新（作者署名 Juampi）。
- **贡献者**：39 人（gh api contributors）。README 列出核心三人：袁一哲（创始人/维护者）、马昕瑞（核心开发）、胡彬（Agent 方向）（`README.md` 第 2 节）；实操上大量来自社区 PR（根目录 `pr-body-*.md` 为 PR 模板残件）。
- **贡献流程**：`README.md` 第 7 节定义完整流程——技能目录、双语镜像 README、SKILL.md frontmatter、索引更新、Draft/Beta/Stable 状态标签，且要求提交时附使用教程视频。
- **增长渠道**：抖音视频教程、微信社群、知识星球、网页版与代充业务互相导流（`README.md` 第 1、3 节）；中文科研社区"用 Agent 做科研"的传播是星数爆发的主要推手（作者自述，见 3.1 节）。
- **release 节奏**：无 GitHub Release/tags 机制（以 main 滚动更新 + `npx skills update`/同步脚本为更新通道）。

## 9. 亮点与风险

**亮点**

1. Router+manifest 技能架构：按论文类型/章节/期刊轴按需加载规则片段，直击长规则语料的上下文成本问题（`skills/*/SKILL.md`、`manifest.yaml`）。
2. 把规则当契约测：26 个测试文件中相当一部分直接断言 Markdown 指南的关键内容，技能质量进入 CI 闭环（`scripts/tests/test_main_text_discipline.py` 等）。
3. 交付物可验证：图件走 PDF 几何碰撞/对齐自动审计，参考文献逐字段多源交叉验证，而非只给"建议"（`skills/nature-figure/scripts/audit_figure_collisions.py`、`skills/nature-ref-verifier/`）。
4. 分发工程完整：npx skills、Claude Code、Codex、Chatbox、OpenClaw 等多通道安装脚本与插件清单，含带节流与失败保护的自动更新（`scripts/autoupdate-skills.sh`、`README.md` 第 5 节）。
5. 双语文档与元数据一致性全部工具化强制，第三方图库附 `THIRD_PARTY_NOTICES.md`，工程严谨度远超一般 prompt 合集。

**风险/局限**

1. README 混入大量商业导流（代充网站、知识星球、微信客服），开源仓库同时是营销漏斗，选型时需剥离（`README.md` 1.2-1.4 节）。
2. 23 个技能中仅 4 个 Stable，多数 Beta/Draft；规则以 Nature 系期刊为中心，其他学科/期刊覆盖有限（技能索引状态列、`skills/nature-shared/journal-formats/`）。
3. 技能本质是指令文本，实际产出质量强依赖宿主模型能力与对 `SKILL.md` 的遵循度，冒烟评估也承认"非自动基准"（`docs/skill-scope-smoke-evaluation.md`）。
4. 深度绑定 Claude Code/Codex 的工作流细节（hook、目录约定），迁移到其他 Agent 需自行适配；`nature-figure` 捆绑 28MB 第三方参考图使仓库体积偏大（`skills/nature-figure/assets/figures4papers/`）。

## 10. 总结评价

nature-skills 是 2026 年"AI for Research"浪潮中增长最快的中文开源项目之一：它把科研写作、绘图、审稿、文献四个环节沉淀为一套可被任意编程智能体加载的技能包，并用接近软件工程的标准管理这些"提示词资产"——路由式上下文加载、内容契约测试、双语镜像校验、交付物几何审计，这在同类 skills 仓库中属于最认真的一档。半年 52k 星既有产品力因素，也明显受益于抖音/微信社群运营与商业导流的放大。适合科研人员把它当作 Claude Code/Codex 的即装即用科研插件，也适合构建自有技能库的开发者研读其 router 架构与内容工程实践；使用者应意识到多数技能仍处 Beta/Draft、规则以 Nature 系为锚点，且需自行配置 Python/R、MCP 与 API 凭据等运行时依赖。
