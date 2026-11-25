# hugohe3/ppt-master 项目分析

> 星数 66605（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

**一句话定位：一套运行在 AI Agent 工具（Claude Code / Codex / Cursor 等）里的 "skill" 工作流包，把任意文档或主题变成原生可编辑的 PowerPoint 文件——交付真正的 DrawingML 形状、母版、数据驱动图表与 OMML 公式，而不是图片式或扁平文本框拼贴。**

| 基本事实 | 值 | 出处 |
|---|---|---|
| 名称 | ppt-master（PPT Master）v6.8.0 | `skills/ppt-master/SKILL.md` metadata |
| 星数 | 66,605（快照 2026-10-24）；gh api 实测 71,339（2026-10-31） | 任务参数 / gh api |
| 语言 | Python（336 个 .py，约 20.7 万行；另有 277 个 .md 约 3.6 万行提示词/文档） | 源码统计 |
| 许可证 | MIT（Copyright (c) 2025-2026 Hugo He） | `LICENSE` |
| 最近活跃 | 最近 push 2026-10-30（当天发布 v6.8.0），创建于 2025-12-10 | gh api |
| 形态 | Agent skill 工作流包（非应用、非库、无服务端），`.claude-plugin/marketplace.json` 支持 Claude Code 插件市场安装 | `.claude-plugin/marketplace.json` |

## 2. 项目是什么，解决什么问题

市面上的 "AI 生成 PPT" 大多交付两条退化路线之一：把整页渲染成图片（完全不可编辑），或用 python-pptx 堆一堆扁平文本框（能编辑但毫无设计深度）。PPT Master 的核心主张是"原生深度"：交付物必须是 PowerPoint 自己的对象模型——带调节手柄的原生形状与连接符、`p:sldMaster`/`p:sldLayout` 继承的真母版、可"编辑数据"的原生 Chart/Table、PowerPoint 2010+ 可编辑的 OMML 公式、原生转场与动画、逐页旁白音频（README_CN.md 产品定位节、docs/zh/powerpoint-svg-mapping.md）。

它的形态在同类项目中非常特殊：**仓库本身既不是应用也不是库，而是一个给 AI Agent 用的 "skill" 包**。用户装好 Python 和任意带 agent 能力的工具后，在聊天框里说"用这份 PDF 做一份 PPT"，agent 按 skill 的路由与门禁流程在本机完成转换→规划→SVG 创作→质量门→导出 `.pptx`（README_CN.md 快速开始；`skills/ppt-master/SKILL.md`）。数据不出本地、不锁定平台与模型、无订阅费——唯一成本是用户自己的模型用量（README_CN.md 三个承诺）。

作者何雨果（Hugo He）是投融资领域从业者（注册会计师、资产评估师），日常工作大量审阅 PPT，痛点明确：AI 生成的幻灯片必须能在 PowerPoint 里继续编辑（README_CN.md 关于作者）。项目还有一个显式的理念输出：仅凭 Python 和 agent 能力能走多远——整个仓库的"程序"部分是一套极克制的确定性 Python 工具链，"智能"部分全部外包给宿主模型，`harness + model = agent` 是 README 中反复强调的边界（README_CN.md "这是一个工具，不是一个许愿池"；docs/technical-design.md 设计哲学节）。

能力范围不止"从零生成"：三条顶层路线覆盖 Generate PPTX（从源材料/主题生成，含 Beautify 1:1 美化、Image to PPTX 图片重建、快速模式）、Create Template（从参考材料提炼可复用的 Brand/Style/Layout/Deck 模板工作区）、Edit Native PPTX（保留原设计的逐字节 round-trip 编辑，未改页面 `rebuilt=0`），还能追加原生转场、逐对象动画、TTS 旁白并合成带字幕的 MP4（`skills/ppt-master/workflows/routing.md` §2；`docs/animations.md`、`docs/audio-narration.md`）。

增长侧写：仓库 2025-12-10 创建，约 10 个月冲到 66k+ 星（gh api 显示当前 71k），配了 AtomGit 镜像、示例仓库 hugohe3/ppt-master-examples 在线预览、Kimi/PackyCode 等 API 中转商赞助位（README_CN.md 页首），是典型的"爆款个人项目 + 商业化赞助"运营模式。

## 3. 技术栈与依赖

- **主语言**：纯 Python（3.10–3.14），面向用户无任何前端构建链；`index.html` 仅是仓库占位页。
- **确定性工具链（核心依赖，`skills/ppt-master/requirements.txt`）**：
  - `python-pptx` + `XlsxWriter`：SVG→PPTX 打包与图表工作簿；
  - `skia-pathops`（布尔运算物化）+ `uharfbuzz`（OpenType 排版与字形轮廓）：把 SVG 路径/文字 operand 编译为 DrawingML freeform；
  - `PyMuPDF`（PDF→MD，注意 AGPL-3.0 许可证已在注释中显式声明）、`mammoth`/`markdownify`/`ebooklib`/`nbconvert`（DOCX/HTML/EPUB/ipynb 原生转换）、`openpyxl`（Excel），pandoc 仅作小众格式兜底；
  - `requests` + `beautifulsoup4` + 可选 `curl_cffi`（TLS 指纹伪装抓微信公众号等）；
  - `edge-tts`（旁白音频）、`google-genai`/OpenAI 兼容 API（AI 生图，`.env.example` 还列出 qwen/zhipu/volcengine 后端）、`flask`（本地确认 UI）。
- **"提示词即代码"层**：约 3.6 万行 Markdown 构成工作流权威文件、角色规范（strategist/executor/checker 系列 `references/executor-*.md`）、187 种 Office 预设形状词汇表（`references/preset-shape-vocabulary.md`）、共享视觉标准等，与 Python 脚本以 `${SKILL_DIR}` 路径锚点协作。
- **宿主**：Claude Code（作者推荐、测试最充分）、Codex、Cursor、Gemini CLI 等；`npx skills add hugohe3/ppt-master` 或 Claude Code plugin marketplace 一行安装（`.claude-plugin/marketplace.json`）。

## 4. 目录结构与代码组织

| 目录 | 职责 | 出处 |
|---|---|---|
| `skills/ppt-master/` | skill 本体（唯一功能主体）：SKILL.md 入口 + workflows + references + scripts + templates | 顶层 `ls` |
| `skills/ppt-master/workflows/` | 路由与运行时权威：routing.md 三路线判定、generate-pptx.md 七步主管线、profiles/（quick-generate、beautify、image-to-pptx）、stages/（topic-research、verify-charts、customize-animations、generate-audio 等 10 个） | `workflows/` |
| `skills/ppt-master/scripts/` | 102 个顶层 Python 脚本（约 5.9 万行）+ 4 个子包（svg_to_pptx、pptx_to_svg、pptx_ooxml、pptx_shapes）+ tests/ + 文档化 docs/ | `scripts/` |
| `skills/ppt-master/scripts/svg_to_pptx/` | SVG→DrawingML 编译器：drawingml/（转换器、路径、文本度量）、native_objects/（图表 XML/chartex、公式 AST→OMML 编译器、表格、工作簿）、pptx_package/（打包、母版结构、旁白、备注），52 个文件 | `scripts/svg_to_pptx/` |
| `skills/ppt-master/scripts/pptx_to_svg/` | 反向 round-trip：ooxml_loader、chart/custgeom/fill/ln/effect→SVG 等 24 个模块，支撑 Edit Native PPTX | `scripts/pptx_to_svg/` |
| `skills/ppt-master/scripts/svg_quality/` + `svg_quality_checker.py` | SVG 创作契约检查器（svg_contracts.py 1131 行），错误阻断导出 | `scripts/svg_quality/` |
| `skills/ppt-master/scripts/confirm_ui/` | Flask 本地确认界面（server.py 约 2986 行），实现 Stage-1/Stage-2 阻塞门禁 | `scripts/confirm_ui/server.py` |
| `skills/ppt-master/scripts/source_to_md/` | PDF/DOCX/HTML/EPUB/ipynb/XLSX/网页→Markdown 转换后端 | `scripts/source_to_md/` |
| `skills/ppt-master/templates/` | 内置模板库：brands/（麦肯锡、BCG、AWS、中国电信、小米等 20+）、styles/（13 种：mbb-consulting、investor-pitch、incident-postmortem…）、layouts/（6 种画布版式）、charts/（36 种图表 SVG）、icons/（phosphor、simple-icons）、decks/、scaffolds/（design_spec 与 spec_lock 骨架） | `templates/` 各 index.json |
| `projects/` | 用户工作区：每个项目含 sources/、analysis/、images/、svg_output/、exports/、validation/workflow.log | `projects/README.md` |
| `docs/`、`docs/rules/` | 面向人的文档（技术设计、路线图、FAQ、Windows 安装）+ 面向 agent 的仓库级风格规则（code-style、prompt-layers、ownership、rule-owners） | `docs/` |
| `.github/` | 仅 FUNDING、issue/PR 模板、维护者手册；**无 CI workflows**（有意为之） | `.github/`、AGENTS.md |

组织评述：单 skill 仓库、prompt 与工具双层的"AI 原生 monorepo"。Python 侧是严格的分层编译器架构（checker 契约层 → converter 编译层 → package 打包层），Markdown 侧则有 `docs/rules/rule-owners.md` 记录每条跨文件规则的唯一 owner，工程化程度在个人项目中罕见。

## 5. 核心架构与关键实现

### 5.1 SVG 作为项目专有中间语言（设计核心）

docs/technical-design.md 明确定义：`svg_output/` 是页面设计的唯一完整来源，但这是"项目契约约束下的 SVG"——借用 SVG 语法，关闭任意元素/属性/单位，三种输入状态分别处理：

| 输入状态 | 定义 | 处理方式 |
|---|---|---|
| Canonical 创作输入 | 提示词/模板/示例推荐的标准拼写 | 直接编译，无警告 |
| 兼容输入 | 有文档记录的历史拼写，可确定性归一化 | 检查器非阻塞警告，转换器静默归一 |
| 无效输入 | 无映射、歧义、破坏结构契约 | 检查器报错并阻断标准流程 |

方向是"SVG 适应 PPT Master，而不是扩展去兼容整个 SVG 标准"。这把"AI 生成"从自由发挥变成在封闭语法内创作，使后续编译器可以做到确定性映射——例如排版用 SVG px 语义，`font-size="24"` 是 canonical 拼写，其他单位仅在有确定性归一化时被接受为兼容输入（docs/technical-design.md）。

### 5.2 路由式工作流与门禁纪律（agent 编排层）

`SKILL.md` 规定硬性加载顺序（原文要点）：

```text
1. Read this file.（SKILL.md 自身）
2. Run attribution_guard.py —— 任何非零结果立即停止，不检查、不修复、不绕过
3. Read workflows/routing.md（路由权威）
4. Select exactly one top-level route + active profile
5. Read only the resulting runtime authority 及其显式触发的支持文档
```

路线判定是确定性查表而非问用户：Generate PPTX（含 Beautify / Image to PPTX / Quick 三种 profile）、Create Template、Edit Native PPTX 三选一；`routing.md` §3 还规定了 15+ 条细分触发（如"保留措辞与页序的 1:1 美化→Beautify；页序被改→Edit Native；图表存在→导出前跑 verify-charts"），并明确禁止给用户展示"路线选择菜单"。

`generate-pptx.md` 主管线为七步：Source→Project→Template 候选→Stage-1 沟通契约确认（⛔ 阻塞）→Stage-2 方案→Executor 创作+质检→后处理导出，配 `templates/scaffolds/design_spec.md` + `spec_lock.md` 双工件。全局纪律 7 条（串行执行、阻塞即停、失败在"拥有该错误的浅层"修复等），配合 `workflow_log.py` 把每条 Python 命令信封写入 `validation/workflow.log` 作冷审计证据（docs/technical-design.md 职责表）。

### 5.3 SVG→PPTX 编译器（工具链核心）

`scripts/svg_to_pptx/`（52 文件）是主线编译器：`drawingml/converter.py` 分发 SVG 元素到 DrawingML，`paths.py` 借 skia-pathops 做 PowerPoint 风格布尔合并、uharfbuzz 提取字形轮廓生成 freeform；`native_objects/` 把标记为"原生对象"的 SVG 编译为带数据源的 chart XML/chartex、OMML 公式（formula_parser → formula_ast → formula_compiler → formula_omml 的完整小型编译器）、可编辑表格与 Excel 工作簿；`pptx_package/` 负责 OPC 打包、`template_structure.py` 产出真母版/版式、`narration.py` 注入逐页音频。反向的 `pptx_to_svg/` 实现 `--roundtrip`：未编辑页面逐字节保留（Edit Native PPTX 的导出契约 `rebuilt=0`）。

### 5.4 归因守卫与完整性门（治理设计）

`scripts/attribution_guard.py`：SKILL.md 的第 2 步是硬门禁，校验 LICENSE 哈希（`_LICENSE_DIGEST`）、SPONSORS 文件、9 个关键 gate 文件的完整性，任何缺失或篡改即 fail-closed 拒绝运行。这是对"白嫖 skill 但删掉赞助/署名文件"的分发行为的程序性反制，也解释了 SPONSORS.md 被列为必需文件的商业动机。

## 6. 构建与运行

无构建步骤（纯脚本包），实际运行命令（均出自 README/AGENTS.md/工程文件）：

```bash
# 安装（方式 A：clone；方式 B：ZIP；方式 C：skill marketplace）
git clone https://github.com/hugohe3/ppt-master.git && cd ppt-master
pip install -r requirements.txt          # 根目录，递归指向 skills/ppt-master/requirements.txt

# 使用：在 agent 聊天框内自然语言驱动；确定性命令由 agent 按工作流调用：
python3 skills/ppt-master/scripts/source_to_md.py <file_or_URL_or_dir>   # 源材料转换
python3 skills/ppt-master/scripts/project_manager.py init <name>         # 项目初始化
python3 skills/ppt-master/scripts/svg_quality_checker.py <project>       # 质量门
python3 skills/ppt-master/scripts/svg_to_pptx.py <project>               # 导出 PPTX
python3 skills/ppt-master/scripts/svg_to_pptx.py <project> --roundtrip   # Edit Native 工作区
python3 skills/ppt-master/scripts/update_repo.py                         # 拉更新并同步依赖

# 测试（唯一允许位置，标准库 unittest，逐模块运行，禁 pytest/discover）
cd skills/ppt-master/scripts && python3 -m unittest tests.test_<topic>
```

## 7. 测试与工程质量

- **测试规模**：89 个 `test_*.py` 全部位于 `skills/ppt-master/scripts/tests/`，标准库 `unittest`，通过公开入口（CLI `main(argv)`、converter、checker）在临时目录最小夹具上断言，网络调用一律 mock（`docs/rules/code-style.md` §11；`tests/test_apply_template.py` 为典型样例）。命名含大量 `test_dogfood_*_fixes.py`，说明"真实生成事故→回归测试"的工作流。
- **CI**：刻意没有——code-style.md §11 明文禁止 CI/pytest/coverage 工具/合并门禁，理由是"本仓库是工作流/skill 包，测试是回归网而非发布闸门"（`.github/` 无 workflows 目录佐证）。
- **质量观察**：契约检查器（`svg_quality/svg_contracts.py`）与导出器互相独立、导出器不信任检查器（"defensively validate"），这种双保险设计少见；转换器内嵌 `font_metrics.json`/`font_advances.json` 做真实字宽测量，文本溢出控制是确定性的。风险面：prompt 层与代码层约 24 万行的耦合一致性只靠 `docs/rules/` 的 owner 制度人工维护；PyMuPDF 是 AGPL 依赖（源码注释已显式提示再分发风险）。

## 8. 社区与维护状态

- **活跃度**：极高。gh api：最近 push 2026-10-30（即分析前一天），open issues 仅 1（issue 处理极快或倾向关闭），forks 5,565。
- **release 节奏**：密集小版本——v6.7.0（10-08）、v6.7.1（10-16）、v6.7.2（10-17）、v6.7.3（10-22）、v6.8.0（10-30），约每周一版，README 徽章与 AtomGit 双平台分发（gh api releases）。
- **贡献者**：28 名（含匿名，gh api contributors），主体是单人维护；`.github/MAINTAINER_PLAYBOOK.md` 与 CONTRIBUTING.md 提供社区协作规范，`docs/zh/roadmap.md` 自述"个人维护的开源项目，按优先级而非时间表推进"。
- **商业化**：Kimi、PackyCode、APIKEY.FAN、RunAPI 赞助位 + SPONSORING.md 招商 + 推广位引流作者个人站点，star 增长与该运营强相关。

## 9. 亮点与风险

**亮点**
1. 架构命题成立且执行彻底："约束中间语言（项目 SVG）+ 确定性编译器 + 模型只做创作决策"，把 LLM 输出的不确定性压缩进可校验的封闭语法（docs/technical-design.md）。
2. 交付真原生 PPTX 的深度在同领域领先：187 预设形状、真母版继承、原生 Chart/Table/OMML 公式、逐字节 round-trip 编辑，均有对应源码实现而非宣传词。
3. 提示词工程被工程化管理：rule-owners 单一负责制、prompt 分层规则、vocabulary 统一术语表，是可学习的 agent 项目治理范本。
4. 商业闭环与开源平衡成熟：attribution guard 程序化保护赞助位，同时工具本体完全免费、数据本地化。
5. 文档密度极高且中英双语，FAQ/Windows 安装/路线图覆盖真实用户路径。

**风险/局限**
1. 单人维护 + 无 CI：24 万行规模下回归依赖 89 个 unittest 与 dogfood 习惯，总线因子为 1。
2. 效果天花板由模型决定（官方自述 `harness + model = agent`），低端模型下体验显著降级；Image to PPTX 路线仅支持 Codex 宿主。
3. attribution_guard fail-closed 对再分发/魔改不友好，二次开发者必须保留赞助文件，开源洁癖者可能有顾虑。
4. PyMuPDF（AGPL-3.0）使整体再分发需单独评估许可证条款。
5. 学习曲线真实存在：Windows PATH/执行策略、agent 工具安装、依赖一次性装齐，官方专门写了 windows-installation.md 与 FAQ 承接（docs/zh/）。

## 10. 总结评价

ppt-master 是 2025–2026 年 AI agent 生态里最成功的"skill 形态"项目之一：它没有做应用，而是把一整套 PPT 生产知识封装成"路由 + 门禁 + 确定性 Python 工具链"，让任何 agent 宿主变成 PPT 专家，并在"生成物必须原生可编辑"这一条窄路上做到了同类最深的 PowerPoint 对象模型覆盖。10 个月 66k+ 星源于精准痛点（AI PPT 不可编辑）、可立刻验证的示例仓库、双语极致文档与赞助位运营的组合拳。适合三类人：想把文档变成真 PPT 的重度用户（配 Claude/Kimi 级模型）、研究 agent 工作流与提示词工程治理的工程师（rule-owners、路由门禁、SVG 中间语言三套设计都值得精读）、以及想学习个人开源项目商业化运营的开发者。源码阅读入口建议按 SKILL.md → workflows/routing.md → docs/technical-design.md → scripts/svg_to_pptx/drawingml/converter.py 的顺序。
