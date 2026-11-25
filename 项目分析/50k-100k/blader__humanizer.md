# blader/humanizer 项目分析

> 星数 59889（快照 2026-10-24）｜主语言 Python（GitHub 标注，实为"提示词产品"仓库）｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：一个跨 agent 通用的 "Agent Skill"，用一份人工精修的提示词（SKILL.md）把 AI 腔文本改写成自然的人写风格，核心方法论来自维基百科的《Signs of AI writing》。

| 项目 | 值 | 出处 |
|---|---|---|
| 名称 | blader/humanizer | gh api repos/blader/humanizer |
| 星数 | 59889（快照）；分析时 gh api 实测 63929，9 个月涨到此量级 | 同上 |
| 创建时间 | 2026-01-18 | gh api |
| 最近 push | 2026-10-28（合并 PR #307 修复 plugin manifest schema） | gh api commits |
| 语言 | GitHub 标注 Python，但全仓库唯一的 Python 是 146 行校验脚本；产品本体是英文 Markdown | src/scripts/validate-package.py |
| 许可证 | MIT（Copyright (c) 2025 Siqi Chen） | src/LICENSE |
| 当前版本 | 3.1.0（2026-09-28 发布） | src/CHANGELOG.md、gh api releases |
| 仓库体积 | 约 214KB，16 个文件，无构建步骤 | tarball 实测；src/AGENTS.md |

## 2. 项目是什么，解决什么问题

Humanizer 解决的是一个随着 LLM 普及而爆发的写作问题：机器生成的文本有一种可辨识的"平均味"。README 开篇给出典型对照——AI 版公告充满 "I'm thrilled to announce 🚀"、"It's not just a feature; it's a whole new way to collaborate"、"Let that sink in" 这类句式，而改写后只剩事实本身（src/README.md:10-14）。

它的方法论根基是维基百科编辑社群用于甄别 AI 生成内容的指南《Wikipedia:Signs of AI writing》（由 WikiProject AI Cleanup 维护）。项目把该指南工程化为一套可执行的提示词：将 26 种"AI 痕迹"按强度与出现频率编号分组（src/SKILL.md），前 5 种（Not X but Y、单句收尾、伪深刻格言、铺垫式开场、与假想敌辩论）单独出现一次即足以触发改写，弱模式（如破折号、卷曲引号）需同段落多证并存才行动。这种"证据强度分级"是它区别于普通提示词的核心设计。

理论基础写在 SKILL.md 开头：语言模型默认选择对最广读者、最广主题都"安全"的表达，而人类作者为具体的读者和主题做具体选择，因此每一条 AI 痕迹本质上都是这种"默认选择"的某种形态（src/SKILL.md:19-28）。整个技能不靠词表替换，而是靠这个统一因果模型驱动结构性改写。

README 单独列出五个最强 tell 作为门面（src/README.md:20-30）：1) Not X but Y 对比句（"It's not just a feature, it's a shift"）；2) 单句收尾（"Let that sink in."）；3) 伪深刻格言（"At its core, what really matters is..."）；4) 铺垫式开场（"Here's the thing."）；5) 与假想敌辩论（"I'm not saying X, but..."）。这五条对不熟悉该话题的读者构成最直观的"第一分钟价值"。

值得强调的边界：README 明确声明 "Getting past AI detectors is not a goal"——它的目标是让人类读者读得舒服，不是规避检测器，且检测器仍会标记其大部分输出（src/README.md:18，issue 模板 config.yml 也专门设置了引导链接）。项目还有一条硬性防幻觉规则：改写不得编造任何姓名、数字、日期、引用，缺细节时必须询问而非补造（src/SKILL.md:37）。

从形态上说，这不是传统软件，而是"提示词即产品"（prompt-as-product）：SKILL.md 就是全部产品逻辑，5217 词（受校验器 5500 词上限约束，src/scripts/validate-package.py:142-144），配合 Claude/Cursor/插件清单和一致性校验脚本构成完整发行包。它抓住了 2026 年 Claude Code skills 生态爆发的窗口期。

## 3. 技术栈与依赖

- 产品语言：无代码运行时。核心是 Markdown + YAML frontmatter（src/SKILL.md:1-11）。
- Python 3.12：仅用于 scripts/validate-package.py 的包一致性校验，零第三方依赖（脚本 docstring 自述 "without external dependencies"，仅用标准库 json/re/pathlib）。
- Node 22：仅 CI 中用于 `npx skills` 技能发现校验与 `claude plugin validate`（src/.github/workflows/validate.yml）。
- 分发适配层：.claude-plugin/plugin.json + marketplace.json（Claude Code 插件/市场）、.cursor-plugin/plugin.json（Cursor）、agents/openai.yaml（OpenAI 兼容 agent 的显示名与默认 prompt）。
- 无 pyproject/package.json/lockfile——仓库刻意保持"无构建步骤"（src/AGENTS.md:7）。

## 4. 目录结构与代码组织

| 文件/目录 | 职责 |
|---|---|
| SKILL.md | 产品本体：技能提示词（26 模式 + 工作流），"source of truth"（src/AGENTS.md:13） |
| README.md | 面向用户：安装、用法、26 模式速查表、完整示例 |
| AGENTS.md | 面向贡献者/agent 的修改规范：版本同步、模式编号规则、5500 词预算、文风要求 |
| CHANGELOG.md | 22 个版本的发布记录，1.0.0 → 3.1.0 |
| LICENSE | MIT |
| .claude-plugin/ | Claude Code 插件与 marketplace 清单 |
| .cursor-plugin/ | Cursor 插件清单（3.1.0 新增，#278） |
| agents/openai.yaml | OpenAI 兼容 agent 的接口元数据 |
| scripts/validate-package.py | 包一致性校验器（CI 与发布前手工运行） |
| .github/workflows/validate.yml | CI：三项检查 |
| .github/ISSUE_TEMPLATE/ | 两类结构化 issue：新 AI 痕迹上报 / 改写缺陷报告 |

组织方式评述：极简单包结构，根目录即发行包。所有清单都指向根 SKILL.md（Claude 插件 `"skills": ["./"]`，Cursor 刻意省略 skills 字段），保证"一个 prompt、多个分发面"，并有一条校验规则禁止出现第二个 SKILL.md 或符号链接（src/scripts/validate-package.py:72-80）——这是 2.10.x/2.11.x 系列被符号链接问题反复踩坑后的制度性修复（src/CHANGELOG.md:24-36）。

## 5. 核心架构与关键实现

### 5.1 提示词的状态机：四步工作流（src/SKILL.md:36-39）

SKILL.md 把改写定义为一个带自检的流水线，而非一次性重写：

1. **标记痕迹**——通读全文按强度标注模式，并扩展到段落级形态（跨句对比、三连排比、节节相同的收尾都是同一 tell 的放大版）；
2. **起草**——可改结构但必须保留全部有依据的论断，禁止编造事实（虚构创作除外）；
3. **检查**——朗读草稿，把"改写后仍最常幸存的 tell"（§1 对比句、§2 收尾、§6 三连、§8 破折号、§19 加粗）再查一遍，把丢信息和加私货都定义为 error；
4. **定稿**——围绕主旨自然重述而非逐句打补丁，长短句交错。

这种"生成-批评-再生成"的自我审查结构是提示词工程里的成熟模式，这里被明确写成可审计的步骤。

### 5.2 模式分级体系（src/SKILL.md:55-381）

26 个模式按六大类组织，编号即优先级：A 表演代替陈述（#1-5）、B 规则化节奏（#6-11）、C 夸大与借势权威（#12-18）、D 规则化排版（#19-21）、E 聊天残留（#22-25）、F 对错误读者写作（#26）。每条模式统一为 Watch for / Problem / Before / After 四段式，附正反例。3.0.0 版做了一次大规模重构：把 35 个模式合并为 25 个，给出完整的旧→新编号映射表（src/CHANGELOG.md:14-16），3.1.0 又新增 #26（回复场景中向知情者复述背景）。词表随模型代际更新（"Word habits change with every model release"），但结构性习惯被排在最前——这是有意的降耦设计。

### 5.3 一致性校验器（src/scripts/validate-package.py，146 行）

这是仓库唯一的"真代码"，却浓缩了完整的发布工程：

- 版本四方同步：SKILL.md metadata.version、CHANGELOG 首个标题、plugin.json、cursor-plugin/plugin.json 必须一致（L52-70）；
- 模式编号连续性：从 SKILL.md 的 `### N. Name` 标题推导数量，要求从 1 连续无断档（L102-109）；
- README 与 SKILL.md 的模式名逐一比对（L111-127），标题必须是 "The 26 patterns"；
- SKILL.md 内所有 `§N` 引用必须落在合法模式区间（L131-139）；
- 字数预算：`len(SKILL.split()) > 5500` 即失败（L142-144）——因为"SKILL.md 的每个词在每次使用时都会被读进上下文"，直接对 token 成本和指令遵循率负责。

### 5.4 多 agent 分发适配（src/.claude-plugin/、src/.cursor-plugin/、src/agents/openai.yaml）

同一产品的三个发行面，共享同一句 description（取 SKILL.md 描述首句，由校验器强制）。README 的安装路径覆盖 Claude Code（`/plugin marketplace add blader/humanizer`）、Codex 与任意 Skills CLI 支持的 agent（`npx skills add blader/humanizer --global --agent '*'`）、Claude.ai/Desktop（ZIP 上传），兜底方案是直接把 SKILL.md 拷进技能目录（src/README.md:36-61）。AGENTS.md 明确要求"不要写把技能限定在一两个 agent 工具上的指令"。

## 6. 构建与运行

无构建。实际可用命令（均出自 README/AGENTS.md/CI）：

```bash
# 安装（Claude Code 2.1.142+）
/plugin marketplace add blader/humanizer
/plugin install humanizer@humanizer

# 安装（Codex / 其他 agent）
npx skills add blader/humanizer --global --agent codex
npx skills add blader/humanizer --global --agent '*'

# 使用
/humanizer
[paste your text here]
Humanize the prose in docs/launch-post.md   # 文件模式

# 声音匹配：附 2-3 段自己的写作样本，改写将跟随其节奏、用词与标点习惯（src/README.md:85-99）
# /humanizer + "Here's a sample of my writing for voice matching:" + 样本 + 待改文本

# 发布前校验（AGENTS.md:32 与 CI 同款三项）
python3 scripts/validate-package.py
npx skills add . --list
claude plugin validate .
```

## 7. 测试与工程质量

- 无传统单元测试——产品是提示词，"测试"以三种形式存在：CI 的三项包校验（src/.github/workflows/validate.yml，PR 与 main push 触发，动作全部 pin 到 commit SHA）；结构化 issue 模板把用户报告变成带输入/输出/期望的结构化回归样本（src/.github/ISSUE_TEMPLATE/rewrite-problem.yml 要求填 agent+模型+prompt+输出+应改成的句子）；README 引用的盲测 #229（16 名评审 16:0 偏好改写版）。
- 有效性验证本质上依赖众包：每个 CHANGELOG 条目几乎都挂着修复的 issue 编号（#187、#212、#238、#247、#269、#277、#290、#295……），形成"用户上报→模式修订→版本发布"的闭环。
- 工程质量观察：AGENTS.md 的修改规则（模式编号无缺口、README/SKILL 同步、版本同步、词数预算）把散落的约定全部变成机器可校验项，对一个"纯提示词仓库"而言工程化程度异常高。仓库自身文风也践行了自己的规范（AGENTS.md 要求 Plain Language）。

## 8. 社区与维护状态

- 活跃度：2026-01-18 创建至 10-28 最近 push，持续 9 个月无空窗；最近提交为合并外部贡献者 PR #307（gh api commits）。
- 发布节奏：22 个版本，2.x 期高频小版本（7 月内连发 2.9.1→2.11.1），3.0.0 大重构后放缓为隔月级（gh api releases：v3.1.0 于 2026-09-28）。
- 贡献者：23 人（gh api contributors），核心明显是作者 blader（Siqi Chen，/LICENSE），外部贡献以模式上报与清单修复为主。
- 商业信号：README 顶部有 cfo.ai 赞助标识（src/README.md:3）；skills.sh 徽章显示其安装量统计渠道（src/README.md:5）。
- 社区分流：Discussions 承接提问/反馈，issue 模板强制分类，config.yml 明确把"过不了 AI 检测器"的诉求引导回 README 声明而非 issue（src/.github/ISSUE_TEMPLATE/config.yml）。
- 星数增速：9 个月约 6 万星，属于 2026 年 agent 生态爆发期的头部案例（快照 59889，分析时 63929）。

## 9. 亮点与风险

亮点：

1. 提示词工程化的范本：统一因果模型 + 强度分级 + 四步自检工作流 + 防幻觉硬规则，远超常见"一句话提示词"。
2. 把文档约定变成机器校验：validate-package.py 用 146 行零依赖脚本锁死版本同步、编号连续性、README 一致性与 5500 词预算，是纯提示词仓库罕见 rigor。
3. 跨 agent 可移植设计到位：单 SKILL.md 根加载 + 三套清单适配，兼容性规则写进贡献指南。
4. 方法论有公信力来源：直接对齐维基百科 WikiProject AI Cleanup 的活文档，词表与结构习惯分离以应对模型迭代。
5. 问题定义诚实：明确不做 AI 检测规避、不编造事实、把虚构创作列为豁免，边界清晰。

风险/局限：

1. 效果强依赖所用模型的指令遵循能力，SKILL.md 不会自动迁移到能力较弱的模型；官方未提供跨模型效果数据（盲测 #229 的模型/评审构成未确认）。
2. 26 个模式是英文语料驱动的，对中文等非英文文本的适用性未在仓库内验证（SKILL.md 仅提示 not-X-but-Y 公式跨语言存在）。
3. 词表时效风险：AI 用词习惯随模型版本漂移，模式列表需要持续人工维护，3.x 后若维护放缓会逐渐失效。
4. 品类门槛低、同质化快：skills.sh 分发渠道一旦变化或大模型原生内置同类能力，项目的独立价值会受挤压。

## 10. 总结评价

humanizer 是 2026 年 agent skills 生态中"提示词即产品"的最佳样本之一：它把维基百科的反 AI 腔指南改造成一份 5217 词、可自检、有强度分级和防幻觉约束的技能提示词，再用一个零依赖校验脚本和三套发行清单把它做成跨 agent 的正规软件包，并以 issue 驱动的高频迭代维持模式库与模型代际同步。它没有一行运行时代码，却展示了比多数代码仓库更完整的发布工程与社区反馈闭环。适合三类人：想在写作中去除 AI 痕迹的 agent 用户（直接安装即用）；提示词/agent skill 开发者（SKILL.md 的结构与 AGENTS.md 的治理规则值得整篇精读）；研究 LLM 文风特征的人（26 模式即一份可操作的 AI 文体taxonomy）。想读"源码"的人应直奔 src/SKILL.md 与 src/scripts/validate-package.py。
