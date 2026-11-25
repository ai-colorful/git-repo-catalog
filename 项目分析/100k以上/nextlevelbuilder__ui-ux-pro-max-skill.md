# nextlevelbuilder/ui-ux-pro-max-skill 项目分析

> 星数 139292（快照 2026-10-24，gh api 实测 142505）｜主语言 Python｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：为 AI 编程助手（Claude Code、Cursor、Codex 等 20+ 平台）提供"设计智能"的 Agent Skill——内置可本地检索的 UI 风格/配色/字体/UX 规则知识库与一个纯标准库实现的 BM25 搜索引擎，让 AI 在写界面前先查一份专业的设计系统方案。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | nextlevelbuilder/ui-ux-pro-max-skill（npm 包名 ui-ux-pro-max-cli） | skill.json、cli/package.json |
| 星数 | 142,505（gh api，2026-10-30 实测；快照 139,292） | gh api repos/... |
| 主语言 | Python（搜索引擎）+ TypeScript（CLI 安装器） | 目录结构 |
| 许可证 | MIT | LICENSE |
| 版本 | v2.15.0（skill.json 与 GitHub Release 一致，发布于 2026-08-13） | skill.json、gh api releases/latest |
| 最近活跃 | push 2026-10-26（分析前 4 天） | gh api |
| 创建时间 | 2025-11-30（约 11 个月从 0 涨到 14 万星） | gh api |
| Forks / Contributors | 15,080 / ≥100（contributors 首页已满 100） | gh api |

## 2. 项目是什么，解决什么问题

AI 生成的界面普遍存在"一眼假"问题：配色随意、字体搭配不当、违反可访问性规则。本项目把这些设计经验**做成结构化数据 + 检索工具**，打包成各 AI 助手都能加载的 Skill：当用户说"给我的 SaaS 做个落地页"时，AI 先调用本项目的 `search.py` 查询知识库，拿到一份完整设计系统（版式模式、UI 风格、色板、字体组合、动效、反模式清单、交付前检查表），再据此写代码（README.md "How It Works" 一节；`.claude/skills/ui-ux-pro-max/SKILL.md`）。

核心资产是人工策划的 CSV 知识库（`src/ui-ux-pro-max/data/`）：79 个可检索 UI 风格（50 个活跃，含 Glassmorphism、Bento Grid 等，styles.csv 89 行）、192 个产品类型与一一对应的配色方案（products.csv / colors.csv 各 193 行）、74 组字体搭配（typography.csv 75 行）、119 条 UX 准则（ux-guidelines.csv 120 行）、25 种图表选型（charts.csv）、34 种落地页结构（landing.csv）、105 条图标指引（icons.csv）、17 个 GSAP 动效预设（motion.csv）、1934 个 Google Fonts 全量目录（google-fonts.csv 1935 行）以及 22 个技术栈专属准则 CSV（data/stacks/）。README 宣称的所有数字都能在对应文件行数上对上，数据与宣传一致。

v2.0 的旗舰功能是"设计系统生成器"（`scripts/design_system.py`）：先匹配产品类目，再套用 192 条行业推理规则（ui-reasoning.csv，每条含推荐版式、风格优先级、色彩基调、反模式和 JSON 格式的条件决策规则），在 5 个领域并行检索后合成一份 ASCII/Markdown 格式的完整设计方案。它还支持三个 1-10 的"设计旋钮"（variance 大胆程度、motion 动效强度、density 视觉密度）以及 `--persist` 把方案落盘为 MASTER.md + 页面级覆盖文件的跨会话检索结构（search.py 模块 docstring；design_system.py:474-629）。

发行形态有三层：① 源数据与引擎在 `src/ui-ux-pro-max/`（唯一事实源）；② `cli/` 是 TypeScript 写的安装器（npm `ui-ux-pro-max-cli`，命令 `uipro`），按 24 份平台模板（templates/platforms/*.json，claude/cursor/zcode/codex 等）动态生成各助手需要的目录结构与 SKILL 文件；③ `stack/` 是一个"即克隆即用"的 Claude Code 项目模板，把本 skill 与 Anthropic 官方 frontend-design 插件、shadcn MCP、Playwright MCP 组合成"知识+品味+视觉反馈"三层设计工作流（stack/README.md）。

变现模式方面，README 明确区分开源版与 Premium 版（品牌设计、Logo、企业级 Design Token 等），配套 uupm.cc 官网与 PayPal 捐赠入口，并交叉推广姊妹项目 3dviz-pro-max（README.md "Basic vs. Premium" 一节）。

## 3. 技术栈与依赖

| 层 | 技术 | 依赖策略 | 出处 |
|---|---|---|---|
| 搜索引擎 | Python 3，纯标准库（csv/re/math/collections） | **零依赖、零网络**，靠 CSV 内置数据 | src/ui-ux-pro-max/scripts/core.py |
| CLI 安装器 | TypeScript + Node 20，commander/chalk/ora/prompts，Bun 构建（tsc 兜底） | 4 个运行时依赖 | cli/package.json |
| 风格画廊 | Next.js 14 + React 18 + Tailwind，node --test 单测 | 独立子应用 | gallery/package.json |
| 测试 | Python unittest/pytest 165 个测试函数 + Playwright e2e 32 个 | cli/package.json scripts、scripts/tests/ |  |
| 发布 | semantic-release + Conventional Commits，自动同步版本号到 5 个文件 | .releaserc.json、README "Automated Releases" |  |

设计上刻意把"运行时"压到最低：skill 部署后只需要一个 Python 3，不装包、不发请求（README "Prerequisites"；SKILL.md "Requires Python 3.x, no external dependencies"）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `src/ui-ux-pro-max/` | 唯一事实源：data/（13 个领域 CSV + 2 个大型 JSON 上游清单）、scripts/（搜索引擎）、templates/（24 份平台模板） |
| `cli/` | npm 安装器源码（src/commands/init.ts 等）、assets（同步后的 skill 副本）、e2e 测试 |
| `.claude/`、`.claude-plugin/` | Claude Code 本地开发用 skill 副本与 marketplace/plugin 清单 |
| `stack/` | 即克隆即用的 Claude Code 网站设计工作台（组合第三方 skill 与 MCP） |
| `gallery/` | Next.js 风格画廊（浏览 79 种风格，data/styles.csv 经 symlink 共享源数据） |
| `projects/` | 3 个示例产物（saas-landing、healthcare-dashboard、portfolio-dark 各一个 index.html） |
| `scripts/` | 仓库级工具：目录刷新、目录摘要生成、相关性评估、CSV 校验、smoke 测试 |
| `docs/journals/` | 开发日志 |

组织方式是"单一事实源 + 生成式分发"：所有平台文件由 CLI 从 `src/ui-ux-pro-max/templates/` 生成，`npm run sync:assets` 把 skill 同步进 cli/assets，`check:assets` 保证两侧一致（README "Architecture & Contributing"；cli/package.json）。仓库同时内嵌了 4 个相对独立的子应用（cli/gallery/stack/主 skill），属于多包单仓，边界靠同步脚本维持。

## 5. 核心架构与关键实现

### 5.1 手写 BM25 搜索引擎（src/ui-ux-pro-max/scripts/core.py）

不引任何第三方库，从零实现 BM25（k1=1.5, b=0.75），带同义词归一化（最长优先、token 边界替换）、停用词过滤、按文件签名（mtime+size）缓存 CSV 行与倒排索引，避免一次 `--design-system` 里重复建索引（core.py:282-384）：

```python
def fit(self, documents):
    self.corpus = [self.tokenize(doc) for doc in documents]
    self.N = len(self.corpus)
    ...
    for word, freq in self.doc_freqs.items():
        self.idf[word] = log((self.N - freq + 0.5) / (freq + 0.5) + 1)
```

领域路由 `detect_domain()`（core.py:628）在没有显式 `--domain` 时按产品关键词自动选域；无结果时给"最近似词"建议并要求 AI 明说"库中无匹配、以下为兜底通用建议"，这是对 LLM 幻觉的针对性防御（search.py:74-82）。

### 5.2 设计系统生成器（src/ui-ux-pro-max/scripts/design_system.py，1650 行）

`DesignSystemGenerator.generate()`（design_system.py:474）的流水线：产品检索定类目 → 应用推理规则（含 JSON 条件决策规则与 1-10 三旋钮）→ 5 域并行检索 → 按风格优先级选最优 → 色彩模式判定。一个值得注意的工程细节：先解析风格与查询的明暗意图，再据此选色板，代码注释明确记录了修过的 bug——"独立对色彩排序曾导致暗色风格配出亮色背景"（design_system.py:514-519）。持久化支持 MASTER.md 全局真源 + pages/*.md 页面覆盖的分层检索模式（design_system.py:1020）。

### 5.3 版本感知的栈准则（core.py:123-148）

每个技术栈声明"当前主版本"（react 19、nextjs 16、svelte 5 等），查询含旧版关键词（如 "Next.js 15"）时只返回标注了 `Status`/`Applies To` 的遗留行；没有策展的旧版指引就直接返回空，避免把两代框架的实践混在一起（`_stack_query_requests_legacy`、`_stack_row_filter`，core.py:848-950）。22 个栈 CSV 各自带 `Verified At` 列做新鲜度标记。

### 5.4 模板化多平台分发（src/ui-ux-pro-max/templates/platforms/*.json + cli/src/commands/init.ts）

24 份平台 JSON 声明各自安装目录（.claude/skills、.cursor/skills、.zcode/skills、.agents/skills 等）、SKILL.md frontmatter 与脚本引用路径，CLI 的 `initCommand` 支持 GitHub 下载安装与模板安装两条路径（init.ts:50,114,132）。一次维护、25+ 平台生效，是这个项目传播速度的关键杠杆。

## 6. 构建与运行

来自 README 与 cli/package.json 的实际命令：

```bash
# 终端用户：装 CLI 后在项目根初始化（支持 claude/cursor/zcode/codex...25 个值）
npm install -g ui-ux-pro-max-cli
uipro init --ai claude          # 或 --ai zcode / --ai all / --global

# 直接用 skill（无需 CLI，装好后）：
python3 .claude/skills/ui-ux-pro-max/scripts/search.py "fintech banking" --design-system -f markdown
python3 .../search.py "glassmorphism" --domain style
python3 .../search.py "form validation" --stack react
python3 .../search.py "SaaS dashboard" --design-system --persist -p "MyApp"   # 落盘 MASTER.md

# 贡献者：全量离线数据门禁（9 步串行）
npm --prefix cli run verify:data   # = validate:csv + validate:semantic + agent-guide
                                   #  + catalog-summary + unittest(165) + relevance 评估
                                   #  + smoke:domains + smoke:stacks + check:assets
npm --prefix cli run typecheck && npm --prefix cli run build
cd gallery && npm run dev          # 风格画廊
```

## 7. 测试与工程质量

- **测试规模**：Python 侧 13 个测试文件、165 个测试函数（`src/ui-ux-pro-max/scripts/tests/`），覆盖搜索引擎、数据契约、风格分类法、文本韧性、设计系统模式与栈新鲜度；CLI 侧 9 个 Playwright e2e 规范、32 个用例（`cli/tests/e2e/`），覆盖 dry-run、平台探测、各平台 skills 布局契约。
- **相关性回归**：自建评估 harness（scripts/evaluate-relevance.py + relevance_metrics.py），用 fixtures/relevance-cases.json 按校准/留出两个 split 量化搜索质量，阈值入库（fixtures/relevance-thresholds.json）——对"检索质量"本身做了工程化度量，这在同类 skill 项目里少见。
- **数据质量门禁**：`validate_data.py`（52KB）做语义校验；catalog-summary.json 记录数据文件 sha256 并在 CI 校验，.gitattributes 强制相关文件 LF 以保证哈希跨平台可复现（.gitattributes 1-16 行注释，引用 issue #462/#478）。
- **CI**：6 个 workflow（tests.yml 按路径触发并跑 `verify:data`；check-asset-sync.yml 防源/资产漂移；refresh-catalogs.yml 每周一 03:17 UTC 拉 Google Fonts API 与 Phosphor 上游生成候选 diff，只产出 artifact、不自动合入，需人工晋升——README "Catalog provenance and refresh"；release.yml 语义化发布并同步 5 处版本号；bump-versions.yml；smoke-stacks.yml）。
- **观察**：工程纪律明显高于典型"提示词仓库"：数据有溯源（data-provenance.json 36KB）、有哈希门禁、有相关性评估，README 对示例 demo 也标注"社区贡献、非官方基准"。运行时零依赖的设计让安全审计面极小。

## 8. 社区与维护状态

- **活跃度**：gh api 显示 push 于 2026-10-26；最新 Release v2.15.0（2026-08-13），版本号已迭代到 2.15，说明发布节奏稳定（dev 分支出 beta、main 出正式版的语义化发布，README "Automated Releases"）。
- **规模**：142.5k 星、15.1k fork、≥100 贡献者（首页满 100）、91 个 open issue（gh api）。创建于 2025-11-30，约 11 个月达到 14 万星，是典型的"AI 助手生态红利"曲线；README 有 6 个语言版本（az/id/ko/vi/zh/en）并针对各国社区做了本地化。
- **贡献结构**：README 明确"永远不要直接 push 到 main，走 PR"（README 贡献节）；CLAUDE.md（118 行）给贡献者和 AI 协作者同一套开发规范。更细的贡献者分布"未确认"（tarball 无 git 历史）。
- **商业化**：开源版 + Premium 版（uupm.cc）+ PayPal 捐赠 + 多个姊妹项目导流，README 头部约三分之一版面用于推广与交叉引流。

## 9. 亮点与风险

**亮点**
1. 把"设计经验"数据化、可检索、可测试：所有宣称数字（79 风格/192 配色/74 字体/119 UX）都能在 CSV 行数上验证，配 sha256 门禁与相关性评估，数据工程严谨（src/ui-ux-pro-max/data/、scripts/evaluate-relevance.py）。
2. 运行时极致轻量：纯 Python 标准库、零网络、零依赖，任何能跑 Python 3 的 AI 助手都能用（scripts/core.py）。
3. 一次维护、25+ 平台分发的模板架构，是星数增长的核心杠杆（templates/platforms/ + cli/src/commands/init.ts）。
4. 对 LLM 使用场景的细节防御到位：无结果时的反幻觉话术、旧版本准则隔离、MASTER.md + 页面覆盖的跨会话记忆结构（search.py:74-82、core.py:848-950、design_system.py:1020）。
5. CI/发布自动化完整，6 个 workflow 分工清晰，上游目录刷新走"只出 diff 人工晋升"的安全流程（.github/workflows/）。

**风险/局限**
1. 推荐质量上限取决于人工策展的 CSV；BM25 关键词匹配对含糊查询可能错配，输出只是"建议"，最终效果仍由 LLM 实现水平决定。
2. 仓库定位混杂：skill 源、CLI、画廊、stack 模板、姊妹项目推广同仓，README 营销/捐赠/Premium 内容占比高，开发者需要自行分辨信噪比。
3. 打包细节有历史包袱：内部 symlink 导致部分工具报错（README Troubleshooting 记载 v2.5.1 前 marketplace 安装因 symlink 失败；本次 tarball 解压在 Windows 上同样因 gallery/data/styles.csv symlink 失败，需手工复制修复）。
4. 上游依赖型数据（Google Fonts 1934 条、Phosphor 1512 图标）需要持续的刷新机制维持新鲜度，依赖每周 CI 与人工审查的执行力。

## 10. 总结评价

这是"AI 助手技能生态"里的头部基础设施项目：用一套人工策展、可测试、带溯源的设计知识库，加上零依赖的本地 BM25 检索和一个覆盖 25+ AI 编程工具的模板化分发体系，精准切中了"AI 写界面不好看"这个高频痛点，因而在一年内冲到 14 万星。它的工程质量（数据门禁、相关性评估、语义化发布、CI 纪律）远超一般 prompt 仓库，但其本质仍是"建议生成器"——不执行、不渲染，效果取决于下游 LLM。适合两类人：给 AI 助手装上就能立刻受益的前端/全栈开发者（目标用户），以及想学习"如何把领域知识工程化为 AI skill"的技能开发者（源码读者，重点读 scripts/core.py 的 BM25 与 design_system.py 的推理合成、templates/ 的分发设计）。
