# Leonxlnx/taste-skill 项目分析

> 星数 101463（快照 2026-10-24）｜主语言 JavaScript｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

**一句话定位**：一个面向编码 Agent（Claude Code、Codex、Cursor 等）的"设计品味"技能包——用实测数据+可执行脚本，逼着 AI 前端脱离"AI 味"模板页，产出有艺术指导、真实图片、真浏览器自检的网站与仪表盘。

| 事实 | 值 | 出处 |
| --- | --- | --- |
| 名称/作者 | Leonxlnx/taste-skill（tasteskill v2） | gh api 元数据 |
| 星数 | 105,535（2026-10-30 实测，快照期 101,463），7,127 forks | gh api |
| 语言 | JavaScript（标注），实际为"Markdown 知识库 + 零依赖 Node 脚本 + JSON 数据" | `skills/taste-skill/` |
| 许可证 | MIT（Copyright (c) 2026 Leonxlnx） | `LICENSE` |
| 创建时间 | 2026-02-19；最近 push 2026-10-29（活跃） | gh api |
| 最新版本 | v2.1.5（2026-10-24） | `CHANGELOG.md`、gh api releases |
| 依赖 | 零 npm 依赖；运行需 Node 22+ 与已安装的 Chrome/Edge/Chromium | `skills/taste-skill/SKILL.md`、`scripts/lib/cdp.mjs` |
| 主页 | https://tasteskill.dev | gh api |
| 安装 | `npx skills add https://github.com/Leonxlnx/taste-skill --skill "design-taste-frontend"` | `README.md` |

## 2. 项目是什么，解决什么问题

这不是一个可 import 的前端库，而是当前流行的 **Agent Skill** 形态：一个按约定组织的文件夹（`SKILL.md` 指令 + 参考文档 + 脚本 + 数据库），由 agent 平台在构建前端页面时加载执行。它解决的问题是 README 与 SKILL.md 开篇直接点破的：模型生成的页面是"模型见过的所有页面的平均值"——居中大标题、小 eyebrow 标签、三栏图标卡片、柔和渐变、细分隔线、等宽字体小字（`skills/taste-skill/SKILL.md` §开篇）。"没有一个决定是错的，也没有一个决定是它自己做的。"

项目的方法论建立在一份实证研究上（`skills/taste-skill/references/research.md`）：抓取 247 个策展站点（156 个 One Page Love + 92 个 Awwwards Sites of the Day），用 21 个模型评审按 Awwwards 权重（设计 40%/可用性 30%/创意 20%/内容 10%）打分，再从中蒸馏出"高分站做什么、低分站做什么"的量化结论。例如最强站的最大字号中位数 8.4vw 对弱站 5.0vw；图标卡片在高分站出现率 1%、弱站 28%（`SKILL.md` §House rules 末尾）。

一个 build 跑十条流水线（`SKILL.md` §The run）：读 brief 并盘点"只有这个客户有的东西"→ 写论证（每节一条 claim）→ 脚本发牌三种艺术方向、agent 择一 → 从盘点中提炼"只有本主体能成立的想法" → 全文文案 deck → 静态构建（真实图片、滚动场景、无动效）→ lint + 真浏览器检查门禁 → 按评审 rubric 看截图改三轮 → 补动效层 → 汇报。官方称一次构建约 30–45 分钟、几十万 token（`README.md` §How v2 works）。

v2.1（2026-10-18）补上了第二形态：**应用界面**（dashboard、admin、settings），基于对 23 个真实产品界面（17 浅色/6 深色）的像素级测量另立一套规则，`draw.mjs --app` 发牌 7 种应用风格、5 种外壳、25 种屏型（`library/app.json`、`CHANGELOG.md` 2.1.0 节）。

值得注意的是工程纪律的来源：发牌脚本存在的理由是"让脚本选，而不是让模型选——被要求自选方向时，模型永远返回品类默认值"（`scripts/draw.mjs` 头注释）。整个仓库本质上是**把设计决策从模型权重里搬到外部数据与工具里**的一次系统尝试。

## 3. 技术栈与依赖

- **语言**：JavaScript（ESM, `.mjs`/`.js`），Node 22+（依赖内置 `WebSocket`；`lint.mjs` 降级到 Node 18+）。
- **零 npm 依赖**：整个仓库无 `package.json`，CDP 客户端、页面测量、图片搜索全部用 `node:` 内置模块手写（`scripts/lib/cdp.mjs` 头注释："Zero-dependency Chrome DevTools Protocol client"）。
- **外部浏览器**：驱动已安装的 Chrome/Edge/Brave/Chromium（`findChrome()` 覆盖 Windows/macOS/Linux 常见路径，`scripts/lib/cdp.mjs`）。
- **外部 API（仅图片检索，无密钥）**：Openverse、Wikimedia Commons、Art Institute of Chicago、Met、Internet Archive 等（`scripts/assets.mjs` §Image sources、`SKILL.md` 步骤 6）。
- **生成页可选运行时**：`assets/scenes/stage.js` 使用 GSAP ScrollTrigger + 可选 Lenis（CDN 固定版本），由被生成的页面引用，不是本仓库依赖（`CHANGELOG.md` 2.1.0 节）。
- **分发形态**：vercel-labs `npx skills` CLI、Claude Code 插件市场（`.claude-plugin/marketplace.json`）、Codex 插件（`.codex-plugin/plugin.json`）、`.agents/plugins/marketplace.json`、`install.sh` 符号链接安装——同一份 skill，四种分发渠道。
- **CI**：GitHub Actions `check-skills.yml`，跑 4 个自研测试 + 全部脚本 `node --check`。

## 4. 目录结构与代码组织

仓库 72 个文件（tarball 解压统计），顶层扁平、单 skill 聚焦：

| 目录/文件 | 职责 |
| --- | --- |
| `skills/taste-skill/SKILL.md` | 核心：542 行 agent 指令（frontmatter 触发条件 + 10 步 run + 18 条房规 + 工艺法则 + 特例路由） |
| `skills/taste-skill/references/` | 11 篇知识文档（约 4,300 行）：type/colour/layout/imagery/motion/scenes/copy/critique/tells/app-screens/research |
| `skills/taste-skill/scripts/` | 5 个 CLI 脚本 + `lib/` 5 个模块，全部零依赖（约 3,600 行） |
| `skills/taste-skill/library/` | 研究蒸馏出的 JSON 数据库（约 2.6 万行，见 §5） |
| `skills/taste-skill/assets/` | 生成页的中性底座：`base.css`/`base.js`、应用外壳 `app.css`、SVG 图表 `charts.js`、滚动场景引擎 `scenes/stage.js` |
| `skills/taste-skill/evals/evals.json` | skill 形式的评测定义 |
| `.claude-plugin/` `.codex-plugin/` `.agents/` | 三套 agent 平台的插件清单（版本 2.1.5） |
| `.github/workflows/check-skills.yml` | CI：frontmatter 严格 YAML 解析、UTF-8 编码、脚本语法、单测 |
| `tests/` | 4 个测试文件（约 260 行） |
| `install.sh` | symlink 安装器（bash，无其他依赖） |
| `examples/v2/` | 展示页截图 + `CREDITS.md`（示例页所用开源图片的完整署名） |
| `CHANGELOG.md` | 253 行，SemVer 式版本史，逐条注明来源 issue/PR 与贡献者 |

组织评述：**"prompt 即代码"的单包仓库**。没有源码构建、没有 monorepo；数据（library）、知识（references）、工具（scripts）、行为规范（SKILL.md）四层分离，脚本只负责"发牌"与"验收"，中间的创作全部交给 agent 按文档执行。v2.1.3 起 repo 刻意收敛为单 skill（ classics 移至姊妹仓 `Leonxlnx/taste-skill-classics`、图像生成类移至 `Leonxlnx/imagegen-skills`，`README.md` §Moved skills），避免多 skill 安装时的触发混淆。

## 5. 核心架构与关键实现

### 5.1 发牌器 draw.mjs——用随机性对抗模型默认值

`scripts/draw.mjs`（584 行）从 library 中"发牌"一套完整艺术方向。关键设计：

- **可种子化的随机**（`scripts/draw.mjs:101-108`）：mulberry32 风格 PRNG，`--seed` 可复现，`--options 3` 发三张让 agent 择一，选定后 `--direction <id>` 出完整方向表。
- **历史避重**（`scripts/draw.mjs:110-113`）：最近 24 次发牌记录写入 `~/.taste-skill/history.json`，`fresh()` 优先发未用过的项，"连续项目不至于撞脸"。

```js
// scripts/draw.mjs:102-108
let seed = opt('seed') !== undefined && opt('seed') !== true ? Number(opt('seed')) : randomInt(1, 2 ** 31 - 1);
const rnd = () => { seed |= 0; seed = (seed + 0x6d2b79f5) | 0; let t = Math.imul(seed ^ (seed >>> 15), 1 | seed); t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t; return ((t ^ (t >>> 14)) >>> 0) / 4294967296; };
const pick = (arr) => arr[Math.floor(rnd() * arr.length)];
// prefer items not used recently, without ever emptying the pool
const fresh = (pool, recent) => { const f = pool.filter((x) => !recent.includes(x.id)); return f.length ? f : pool; };
```

- **数据规模**（`library/`）：20 个研究方向、185 个分节蓝图（`sections.json`，含桌面/手机 blueprint 与出处站点）、44 套字体系统、56 套调色板、58 个 hero 蓝图、202 个实测参考站点（带面板分，如 `{"name":"PP Neue Montreal","score":8.6}`）、19 组 420 张备选照片、18 个签名动效；`app.json` 另含 7 风格/5 外壳/25 屏型/13 产品类别。`scenes.json` 尚未随库发布（2.2 规划，`draw.mjs:78` 用 `existsSync` 优雅降级——未确认发布时间）。

### 5.2 零依赖 CDP 客户端与真浏览器检查器 inspect.mjs

`scripts/lib/cdp.mjs`（248 行）手写了 Chrome DevTools Protocol 客户端：spawn headless Chrome → stderr 抓 `DevTools listening on ws://` → 内置 WebSocket 通信，封装 `Page.goto/eval/screenshot/wheel`（`wheel()` 用 `Input.dispatchMouseEvent` 派发真实滚轮输入，让平滑滚动库与滚动触发的 reveal 像访客一样触发，`cdp.mjs:230-241`）。一次性浏览器 profile 带 owner 文件（主机名+双 PID），只有确认同机且两个进程都死了才回收，避免并行运行互删（`cdp.mjs:40-62`，2.1.4 修复的竞态，有回归测试 `tests/profile-cleanup.mjs`）。

`scripts/inspect.mjs`（308 行）在 1440/768/390 三宽度各跑一轮：注入 MOTION_RECORDER 统计真实动效元素数（CSS 动画/过渡/WAAPI/脚本驱动 style 变更四类归因，`inspect.mjs:78-88`）；逐屏滚动截图并测量；加载后 1.2s 检查标题可读性；额外跑一遍 `prefers-reduced-motion` 页面，抓"永久隐形"的文本（`inspect.mjs:244-259`）。发现项合并去重后写 `report.json`，有 FAIL 即退出码 1。

### 5.3 规则引擎 rules.mjs——90+ 条可解释的 FAIL/WARN

`scripts/lib/rules.mjs`（243 行）把 `probe.mjs`（929 行页面内测量函数）的指标翻译成发现项，`RULE_IDS` 导出 92 个规则 ID。三条最有代表性的：

- **AI 品牌色识别**（`rules.mjs:8-15, 90-93`）：cream（奶油底）、terracotta（陶土橙）、ai-purple（紫/靛）三个色相家族用 HSL 区间定义，cream+terracotta 组合直接判 `palette-claude` 失败——"这读起来是助手品牌，不是你的"。
- **画面空洞检测**（`rules.mjs:157-163`）：分节内容挤在一侧、另一侧留白 ≥42% 宽度即 `lopsided` 失败，对应 SKILL.md 的"框架对称、不对称只存在于网格内部"。
- **应用屏独立规则集**（`rules.mjs:201-241`）：`APP_SKIP` 集合豁免 30 余条落地页规则（表格线、小标签、重复卡片在 dashboard 属正常），新增 sidebar 不收窄、数字右对齐、tabular 数字体、图表色相 ≤4、禁止占位数据等应用规则。

### 5.4 图片检索 assets.mjs——"诚实配图"的执行器

`scripts/assets.mjs`（865 行）聚合 Openverse（一个索引覆盖 Flickr/Wikimedia/各博物馆）、Wikimedia Commons API、AIC IIIF、Met、Internet Archive 五个源，仅返回可商用许可（公共领域/CC0/CC BY/CC BY-SA，`assets.mjs:24-57`），`--get` 下载并登记来源与许可，`--credits` 生成页脚署名行。示例页的署名档案见 `examples/v2/CREDITS.md`（逐一列出作者、许可、原始链接）。配套 `lint.mjs` 静态扫描禁用模式与套话，`scenes.mjs` 把每个滚动场景在 0–100% 进度上逐帧拍照验收。

## 6. 构建与运行

无构建步骤（零依赖，无需安装）。来自 `SKILL.md` 与 `README.md` 的实际命令：

```bash
# 安装（任选其一）
npx skills add https://github.com/Leonxlnx/taste-skill --skill "design-taste-frontend"
# Claude Code 内：/plugin marketplace add Leonxlnx/taste-skill && /plugin install taste-skill@taste-skill
./install.sh            # 从 clone symlink 安装（--codex / --all-agents）

# 随后由 agent 在项目根目录执行（<skill> 为 skills/taste-skill）
node <skill>/scripts/draw.mjs --sector saas --brief "..." --sections hero,showcase,pricing,footer --options 3
node <skill>/scripts/assets.mjs "<查询词>"          # 搜开源图片；--get 下载；--credits 署名
node <skill>/scripts/lint.mjs <源文件或目录>        # 静态禁用模式扫描
node <skill>/scripts/inspect.mjs <url|index.html>   # 真浏览器三宽度检查，FAIL→退出码 1
node <skill>/scripts/inspect.mjs <url> --app        # 应用屏（dashboard）规则
node <skill>/scripts/scenes.mjs <url>               # 滚动场景逐帧验收
node <skill>/scripts/draw.mjs --app --kind dashboard --brief "..."  # 应用风格发牌
```

测试：`npm install --no-save --no-package-lock yaml@2.6.1 && node tests/frontmatter.mjs`，以及 `tests/text-encoding.mjs`、`node --test tests/profile-cleanup.mjs tests/inspection-files.mjs`（`.github/workflows/check-skills.yml`）。

## 7. 测试与工程质量

- **测试规模**：4 个测试文件约 260 行（`tests/`），风格是"防线测试"而非覆盖率导向：frontmatter 严格 YAML 解析（2.1.2 曾因 description 里未引号的 `: ` 导致 skill 被 Claude Code 静默丢弃，此后加此测试，`CHANGELOG.md`）、全文本 UTF-8 双重编码检查、浏览器 profile 回收的并发回归、检查器截图文件名。
- **CI**：`check-skills.yml` 四步，push/PR 均跑（`.github/workflows/check-skills.yml`）。
- **质量观察**：工程质量明显高于同类 prompt 仓库。CHANGELOG 逐条链接 issue/PR 并致谢贡献者（2.1.5 一次列 8 条社区修复）；脚本的边界处理细致（并行 profile 回收、JS 对话框自动拒绝、dev-server 检测、`--allow` 白名单豁免机制）；每条规则带 `fix` 建议文本，错误信息可执行。真正的"设计质量"验证走的是 4 轮盲测（8/6/5/3 个 build 与参考截图对比，`CHANGELOG.md` 2.1.0 末段），这是内容形态决定的非常规但合理的评测方式。

## 8. 社区与维护状态

- **活跃度极高**：创建于 2026-02-19，8 个月冲到 10 万+星（gh api）；最近 push 2026-10-29，近两周连发 v2.1.0–v2.1.5 六个版本（gh api releases，2026-10-18 至 10-24），issue 编号已到 #171 且当前仅 7 个 open。
- **贡献者**：21 人（gh api contributors），结构是典型的"单人主导 + 社区报 bug"：CHANGELOG 中功能设计均出自作者，社区贡献以修复与回归为主。
- **生态位**：姊妹仓 `taste-skill-classics`（8 个旧风格 skill）、`imagegen-skills`（4 个图像生成 skill）；衍生桌面应用 TasteCode（tastecode.dev，README 头图）；赞助商含 Vercel OSS Program、Kimi、IMG.LY 等（`README.md`）；官方 FAQ 明确声明"无官方代币"（`README.md` 末尾），侧面印证项目热度已引来仿冒。
- **未见**：公开的路线图文件、人类评审的量化对比（研究明确自述评分是"模型判断，非人类评审"，`references/research.md` §1）。

## 9. 亮点与风险

**亮点**

1. **方法论可证伪**：所有设计禁令都锚定到 247 站研究的量化数据（`references/research.md`），且诚实区分"数据支持"与"作者品味"（如承认 monospace 标签在获奖站其实更常见、但因被生成器模仿而明令封杀，`SKILL.md` §House rules）。
2. **零依赖工程**：手写 CDP 客户端 + 内置 WebSocket，无 npm install、无 API key，把"真浏览器验收"的门槛降到只要装过 Chrome。
3. **发牌器对抗模型均值回归**：种子化随机 + 历史避重 + "agent 择一并写明理由、重发须记录"的纪律，直接对症"模型永远选品类默认"的病根（`scripts/draw.mjs` 头注释）。
4. **诚实配图链路**：开源图库检索→下载→署名→报告，一条龙可审计，示例页 `examples/v2/CREDITS.md` 做了示范；禁止把档案照片冒充客户资产。
5. **README/SKILL 文字质量罕见地高**：规则写成可辩护的设计判断（每条 FAIL 附"怎么改"），而非干瘪的 checklist。

**风险/局限**

1. **成本高**：官方口径一次构建 30–45 分钟、几十万 token（`README.md` §How v2 works），多轮"看截图-改"循环对上下文与费用都不友好。
2. **品味即偏见**：禁令名单（禁渐变、禁细线、禁索引编号、禁衬线强调词等）本质是把一种"反 AI 味"的当代画廊审美制度化，与其他风格偏好冲突时只能靠 `--allow` 逐条豁免。
3. **评审信度**：研究的打分者是 21 个模型而非人类评审，作者自认样本"是时尚也是质量"（`references/research.md` §1 Limits）。
4. **平台耦合**：frontmatter/插件清单依赖 Claude Code、Codex 等平台的 skill 规范，规范变动（如 2.1.2 的 YAML 解析事故）即静默失效。

## 10. 总结评价

taste-skill 是"Agent Skill"浪潮里迄今完成度最高、工程最扎实的代表作之一：它没有停在"写一份好的 prompt"，而是把设计知识做成可测量的数据（247 站研究 → JSON 库）、把品味做成可执行的验收（零依赖 CDP 检查器、92 条规则）、把随机性做成对抗模型惯性的工具（发牌器），并用严格的 CHANGELOG、CI 与盲测流程维护这套行为规范。10 万+星来自它精准踩中"AI 生成页面千篇一律"这一普遍痛点，且开源/MIT、装完即用。适合三类人：用 Claude Code/Codex/Cursor 做前端并苦于"AI 味"的开发者（直接装）；研究 agent 行为工程的人（SKILL.md 是 prompt 工程的范本，值得通读）；想做设计工具链的人（references/research.md 的测量方法与 rules.mjs 的规则设计都可迁移）。不适合把它当成通用 UI 框架——它管"页面长什么样"，不管业务逻辑与可访问性之外的正确性。
