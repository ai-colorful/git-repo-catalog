# goabstract/Awesome-Design-Tools 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：由设计协作平台公司 Abstract（原 Flawless App 团队）出品的设计工具精选清单，覆盖从线框图、原型、动效到无障碍、3D 建模的 37 个方向，是设计领域最知名的 awesome 系列清单之一。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | goabstract/Awesome-Design-Tools | gh api repos/goabstract/Awesome-Design-Tools |
| 星数 | 39500（快照）；2026-11 实查 41877 | 任务参数 / gh api |
| Fork | 2420；open issues 250；贡献者约 204 | gh api |
| 语言 | JavaScript（清单本体为 Markdown） | 任务参数 / package.json |
| 许可证 | MIT（Copyright 2019 Lisa Dziuba） | src/LICENSE；package.json 内标 ISC，以 LICENSE 为准 |
| 创建 | 2019-02-12 | gh api |
| 默认分支 | master，内容最后提交 2020-09-28 | gh api（pushed_at 2024-07-28 仅为分支/管理动作，master 内容未变） |
| 仓库状态 | 内容实质冻结（2020-09 后无内容更新） | gh api commits、分支列表、tarball 文件 mtime 均为 2020-09-29 |

## 2. 项目是什么，解决什么问题

设计师日常面对的一个真实痛点是：设计工具生态极度碎片化——原型、动效、配色、图标、交付、用户研究各有一堆竞品，且散落在 Product Hunt、Twitter、各类博客里，没有统一的索引。Awesome Design Tools 用一份人工精选、社区共修的 Markdown 清单来解决这个问题：每个工具一行，格式统一为「[名称](链接) — 一句话英文简介 + 属性徽章」（src/Contribution_Guidelines.md L22-34），按设计任务类型分成 37 个类别（src/README.md L51-86）。

它不是一个孤立文件，而是一个"清单家族"：主文件 README.md 收录工具，Awesome-Design-Plugins.md 收录 Sketch/Adobe XD/Figma 插件，Awesome-Design-UI-Kits.md 收录 UI Kit，Awesome-Design-Conferences.md 收录设计大会（四个文件互相链接导流，src/README.md L5-7）。四份清单合计约 1987 个条目（实测统计见第 7 章）。

项目有较强的"公司出品"背景：它由乌克兰创业团队 Flawless App（Lisa Dziuba、Valia Havruliyk，5 人团队）于 2019 年创建并维护，2020 年初 Flawless App 被 Abstract 收购，README 顶部保留了完整的收购公告"Awesome Design Tools as a part of Flawless App family joins Abstract!"（src/README.md L19-34），此后仓库迁移到 goabstract 组织下。清单同时也是 Flawless/Abstract 的内容营销资产：配套网站托管在 flawlessapp.io/designtools（src/docs/modules/config/tools.js L35）。

值得注意的双重形态：仓库既是给人看的 GitHub Markdown，也是一台静态站点的源码——根目录 index.js 会把这四份 Markdown 转译成带搜索、导航、赞助商 banner 的完整网页（详见第 5 章），转译产物已提交在 docs/ 目录（GitHub Pages 站点）。

## 3. 技术栈与依赖

代码部分极薄（约 30 个 JS 文件），本质是"Markdown 进、HTML 出"的构建脚本（src/package.json、src/index.js）：

- **markdown-it ^8.4.2**（commonmark 模式）：把 Markdown 清单渲染为 HTML —— src/index.js L2、L43-50
- **jsdom ^14.1.0**：在 Node 里对渲染后的 DOM 做 15 步改写（加导航、加汉堡菜单、注入 banner 等）—— src/index.js L52-92
- **html-minifier ^3.5.21**：生产模式压缩输出 —— src/index.js L5、L28-41
- **express ^4.17.1**：仅用于 docs/server.js 的本地预览服务器（127.0.0.1:8001）—— src/docs/server.js
- **tinycolor2 ^1.4.1**：给 logo 加背景色（addBackgroundColorToLogo 模块）
- **showdown ^1.9.1**：声明在 dependencies 中但 index.js 实际只用 markdown-it，属冗余依赖（src/package.json L33-34 vs src/index.js L2）
- 无前端框架、无打包器：客户端就是原生 JS（src/docs/js/script.js，305 行）+ CSS（src/docs/css/design-tools-style.css，1.1MB）

无任何测试依赖、无 CI 配置（仓库无 .github/workflows，tarball 中未见）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 | 出处 |
|---|---|---|
| README.md | 主清单：637 个工具、37 类、1236 行 | src/README.md |
| Awesome-Design-Plugins.md | 插件清单：1112 条、46 类 | src/Awesome-Design-Plugins.md |
| Awesome-Design-UI-Kits.md | UI Kit 清单：224 条、33 类 | src/Awesome-Design-UI-Kits.md |
| Awesome-Design-Conferences.md | 设计大会清单：14 条、按月份 12 类 | src/Awesome-Design-Conferences.md |
| Contribution_Guidelines.md | 收录规则与 PR 规范 | src/Contribution_Guidelines.md |
| index.js | 构建入口：4 份 MD → 3 份 HTML | src/index.js |
| package.json / package-lock.json | Node 工程清单 | src/package.json |
| docs/ | GitHub Pages 产物 + 构建模块源码 | src/docs/ |
| docs/modules/ | 15 个 jsdom DOM 改写模块（加头、加菜单、加 banner、加 ID 等） | src/docs/modules/ |
| docs/modules/config/ | 三份清单各自的站点配置（标题/主题色/导航/favicon） | src/docs/modules/config/tools.js |
| docs/index*.html | 已提交的转译产物（1.4–1.9MB 单文件） | src/docs/index.html 等 |
| Media/ | 徽章 SVG（free/open-source/mac 等）与推广图 | src/Media/ |

组织方式评述：这是典型的"内容型 monolith"——内容（Markdown）与展示层（docs/ 构建器）共仓。巧妙之处在于 Markdown 里直接内嵌 HTML 片段（`<article id>`、`<div class="nav">`、`<div class="hidden-in-page">`，src/README.md L46-92），GitHub 渲染时这些 class 无效果但结构成立，自建站构建器则拿它们当 DOM 钩子做增强，一份内容服务两个渠道。package.json 的 repository 字段还指向旧地址 berezovskycom/Awesome-Design-Tools，是组织迁移的残留（src/package.json L15）。

## 5. 核心架构与关键实现

**（1）MD→HTML 转译流水线（src/index.js）**。整个构建就是一个 104 行的 Promise 链：读取 MD → markdown-it 渲染 → jsdom 加载 → 依次执行 15 个改写模块 → 写入 docs/index*.html。三份清单（Tools/Plugins/Kits）各走一遍，Tools 走生产压缩、另两份不压缩（L94-104）。页面类型靠 H1 文案判断（`isTool/isPlugin/isKit`，L66-68），略脆弱但够用。

**（2）DOM 改写模块化（src/docs/modules/）**。每个增强点独立成模块：addHeader（站点头）、addHamburgerMenu（移动端菜单）、addIDsForHeadings（锚点）、addWelcomeArticle（首屏）、addAllPromoBanners（赞助商 banner）等。配置集中在 docs/modules/config/tools.js：主题色 #0054d7、SEO meta、favicon 全套、导航三项（Tools/Plugins/UI Kits 互相导流，L53-74）。

**（3）赞助商 banner 注入（src/docs/modules/addAllPromoBanners.js）**。这是理解项目商业模式的钥匙：banner 以代码形式配置（标题、文案、链接、目标分类锚点数组），注入到网页端对应分类下，且带完整 UTM 追踪（`utm_medium=Partner-Advertising&utm_campaign=CY19-PA-FlawlessApp-1018`，L30）。当前只启用了 Abstract 的 banner，Supernova、ProtoPie、Maze、Porkbun、Adobe XD 五个赞助位全部被注释掉（L4-117）——能看出收购前这是一处活跃的广告位，收购后只保留自家 Abstract。

**（4）一个真实的构建 bug（src/index.js L28-41）**：

```js
const writeHtml = (html, fileToWrite, isProduction = true) => {
    const minified = minify(html, {...});
    const chooseVersion = isProduction ? minified : html;
    fs.writeFile(fileToWrite, html, function(err, data) { ...
```

压缩结果存进 `chooseVersion` 后没有被使用，`fs.writeFile` 写入的始终是未压缩的 `html`。这就是 docs/index.html 高达 1.8MB、而"生产模式"形同虚设的原因——侧面反映这个仓库的重心从来不在代码质量上。

**（5）客户端交互（src/docs/js/script.js）**。305 行原生 JS 实现三件事：站内搜索（`.js-search-input` 实时过滤列表项）、按平台过滤的模态框（Sketch/XD/Figma）、滚动 spy 高亮侧边导航。对 600+ 条目的单页来说够用。

## 6. 构建与运行

来自 src/package.json L9-12，均为构建脚本，无测试命令：

```bash
npm install            # 安装 markdown-it/jsdom/html-minifier 等
npm run dev            # PRODUCTION=false node index.js —— 开发模式转译
npm run production     # PRODUCTION=true  node index.js —— 生产模式转译（实际压缩无效，见第 5 章）
node docs/server.js    # 本地 127.0.0.1:8001 预览（读当前目录 index.html）
```

产物直接写进 docs/（index.html / index-plugins.html / index-kits.html），由 GitHub Pages 托管。按任务规范本次未实际运行构建，以上为静态阅读结论。

## 7. 清单组织、收录标准与规模统计

**规模统计（基于 tarball 实测，`grep -cE '^[-*] \['` 计条目、`grep -c '^### '` 计分类）**：

| 文件 | 行数 | 条目数 | 分类数 | 备注 |
|---|---|---|---|---|
| README.md | 1236 | 637 | 37 | 徽章：free 615 / open-source 144 / mac 124 |
| Awesome-Design-Plugins.md | 1556 | 1112 | 46 | 徽章：Sketch 1399 / Figma 470 / Adobe XD 290（多平台重复计） |
| Awesome-Design-UI-Kits.md | 644 | 224 | 33 | 按应用场景分类（Blog/Dashboard/Crypto…） |
| Awesome-Design-Conferences.md | 226 | 14 | 12 | 按月份组织 |
| **合计** | 3700 | **约 1987** | — | — |

37 个分类覆盖设计工作全流程：Accessibility → Animation → AR → Collaboration → Color Picker → Design Feedback/Handoff/Inspiration/System/Version Control → Design to Code → Font/Gradient/Icons/Illustrations → Mockup → No Code → Prototyping → Stock Photos/Videos → User Research → Wireframing → 3D Modeling（src/README.md L51-86）。

**收录标准（src/Contribution_Guidelines.md）**：规则简单且可执行——先查重（L5）；链接格式统一 `[名称](链接) — 描述`（L22）；工具按字母排序、"所有工具都很棒所以一视同仁"（L10）；一个工具只归一个分类，跨场景用途用交叉引用句带过（L15-18）；新分类需至少 2 个条目才开（L12）。标签定义相当克制：free 徽章明确要求"完全免费，freemium 或带免费试用的不算"（L25）。最后一句坦承人工筛选的主观性："贡献可能因未通过我们的 curation 被拒，这很主观"（L38）。

**收录逻辑评述**：与大多数 awesome 清单"来者不拒挂链接"不同，这份清单有三点做得更认真：一是每条有实质性的英文一句话评测（说明工具干什么、谁做的），不是裸链接；二是徽章体系（免费/开源/Mac only；插件清单用 Sketch/XD/Figma 平台徽章）让读者能按约束快速过滤；三是 PR 模板把格式要求做成 checklist（src/docs/pull_request_template.md），用流程保证格式一致性。代价是维护完全依赖创始团队的人工 curation，这在团队被收购后成了致命弱点（见第 8 章）。

## 8. 社区、维护状态与赛道竞争

**活跃度（gh api 实查）**：内容提交止于 2020-09-28——最后一个实质 commit 是合并 PR #349 和清理失效链接（"Removing microcopyinspirations.com - website down"），合并人是 Tom Moor（Abstract 联合创始人），说明收购后 Abstract 团队短暂接管过维护。此后 master 冻结超过 6 年：2021-2022 年 dependabot 提的一批安全升级分支（express 4.17.3、jsdom 16.5.0、lodash、qs、ws 等）全部未合并，仍挂在分支列表里；250 个 issue 处于 open 状态；约 204 名贡献者中绝大多数是 2020 年前的 PR 提交者。pushed_at 显示 2024-07-28，但 master 上查不到该时点之后的内容提交，判断为分支/管理操作（未确认具体动作）。

**品牌关系与引流**：这个仓库的星数有一半可以算作 Flawless App→Abstract 内容营销的成果。证据链在仓库内是完整的：README 顶部的收购公告与 FAQ 链接（src/README.md L19-34）；配套网站托管在 flawlessapp.io 域名并带 SEO meta（src/docs/modules/config/tools.js L30-35）；网页端的赞助商广告位系统带自家 UTM 参数（src/docs/modules/addAllPromoBanners.js L30）；清单收尾的 addendum 署名两位创始人（src/README.md 尾部 article#addendum）。即"免费清单引流 → 自家工具/平台变现"的经典开源营销闭环，清单本身就是产品。

**赛道竞争（背景知识，仓库内无直接证据）**：设计资源清单赛道在 2019-2020 年竞争激烈：uxtools.co（工具对比评测）、Prototypr.io Toolbox（本项目明确承认受其启发，见 README addendum）、designer.news 社区、以及 Figma Community 上线后的官方插件/模板市场。这份清单的差异化是"awesome 徽章 + 公司级人工筛选 + 双形态（GitHub/网站）"，在 2019-2020 年确实是最全的一份；但内容冻结后，Figma 一家独大、AI 设计工具（生成式 UI、AI 配图）浪潮完全缺席，其实用价值逐年衰减，星数的持续增长更多来自历史声望的惯性。

## 9. 亮点与风险

**亮点**：

1. 规模与覆盖面：约 1987 个条目、四份子清单、128 个分类（37+46+33+12），2019-2020 年间是设计工具索引的事实标准之一（实测统计见第 7 章）
2. 条目质量高：每条带一句话实质介绍 + 属性徽章，收录规则明确（字母序、单一分类、free 定义严格），可读性和可信度明显高于普通 awesome 清单（src/Contribution_Guidelines.md）
3. 双形态设计巧妙：一份内嵌 HTML 钩子的 Markdown 同时服务 GitHub 阅读与自建网站，构建器仅 104 行（src/index.js、src/README.md L46-92）
4. 商业闭环完整：清单 → 网站 → 赞助商 banner（带 UTM）→ 自家产品，是"内容营销型开源"的教科书案例（src/docs/modules/addAllPromoBanners.js）
5. 社区流量证明：4.2 万星、2.4 千 fork、204 贡献者，Awesome 官方收录（README.md L16 徽章）

**风险/局限**：

1. 内容冻结 6 年+：最后实质提交 2020-09-28（gh api），设计工具市场剧变（Figma 统治、AI 工具兴起）后清单大量失效——2020 年时项目自己就在清理死链，冻结后腐化只会加速
2. 链接腐化不可逆：依赖人工 curation 的清单一旦无人维护，既不删死链也不收新工具，issue 积压 250 个无人处理
3. 代码细节粗糙：writeHtml 压缩结果未写入（src/index.js L36-37）、showdown 冗余依赖、package.json repository 指向旧组织、docs/index.html 等构建产物 1.4-1.9MB 提交入库
4. 依赖带已知漏洞且补丁未合并：dependabot 的 express/jsdom/lodash/qs/ws 安全升级分支全部滞留（gh api branches），构建器已不宜复用

## 10. 总结评价

总评：这是"内容型开源仓库"的标杆样本——用一个极简的 Markdown + 104 行构建器，把设计师的工具选择问题做成了 4 万星的内容资产，并且完整跑通了"清单引流 → 公司产品变现 → 被收购"的商业路径。它的价值不在于代码（代码薄且有 bug），而在于内容组织方法（分类体系、徽章体系、收录规则）和内容运营方法（双形态发布、赞助位、社区 PR 流程）。但它也是"人工 curation 模式脆弱性"的最佳反面教材：创始团队被收购后维护断档，2020 年 9 月起内容冻结，6 年间设计行业从 Sketch/XD 争霸走向 Figma 一统再到 AI 工具爆发，这份清单已从"权威索引"退化为"2020 年设计工具生态的化石标本"。

适合谁用：想快速了解 2019-2020 年设计工具全景与各分类头部产品的设计新人/独立开发者（注意核对链接与产品现状）；想研究"awesome 清单如何做内容营销与商业化"的运营者；想搭建同类技术清单（代码型）的人可参考其分类与徽章体系。不适合谁：想找 2024 年后设计工具（尤其 AI 设计工具）现状的人；想直接复用其构建器代码的工程团队。源码层面值得读的只有 src/index.js 和 src/docs/modules/addAllPromoBanners.js 两个文件，合计不到 250 行。
