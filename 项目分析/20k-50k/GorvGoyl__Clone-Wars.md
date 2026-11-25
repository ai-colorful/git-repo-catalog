# GorvGoyl/Clone-Wars 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 —｜档位 20k-50k｜分析日期 2026-11-17

## 1. 项目速览

一句话定位：一个纯内容型仓库——用两张结构化 Markdown 大表收录"知名网站/软件的开源克隆与开源替代品"，每条附源码仓库、在线 Demo、技术栈和实时星数徽章，定位是学习与选型索引。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | Clone-Wars（GorvGoyl/Clone-Wars） | gh api repos/GorvGoyl/Clone-Wars |
| 星数 | 39500（快照）；2026-11 实查 37292 | 任务快照 / gh api |
| Fork / Watch | 3370 / 559 | gh api |
| 语言 | 无代码（纯 Markdown 内容仓） | tarball 全量文件 |
| 许可证 | AGPL-3.0 | `src/LICENSE` |
| 创建时间 | 2020-12-02 | gh api |
| 最近 push | 2024-08-06（此后无合并提交，进入停滞） | gh api / 提交记录 |
| 主页 | https://gourav.io/clone-wars（作者自建站） | gh api `homepage` |
| 仓库体积 | 约 190KB 内容（README 138KB）+ 131KB 图片 | `src/README.md`、`src/img/og.png` |

## 2. 项目是什么，解决什么问题

Clone-Wars 解决的问题是"想学某知名产品是怎么做出来的，却不知道有哪些开源实现可看"。作者 Gourav Goyal（个人主页 gourav.io）在 2020 年 12 月创建该仓库，把散落 GitHub 各处的知名站点克隆项目集中成一张可扫读的大表，并给每个条目补上三件学习最需要的信息：能不能在线看效果（Demo 链接）、用什么技术栈写的、项目热度如何（GitHub 星数）。README 开头即点明定位："List contains source code, tutorials, demo links, tech stack, and GitHub stars count. Great for learning purpose!"（`src/README.md` 前 10 行）。

仓库把条目明确分为两类并分置两张表（`src/README.md` 第 17-31 行）："Clones"指 UI 上高度模仿但功能不完整、以练手为目的的项目；"Alternatives"指功能完备、可实际使用的开源替代品（如 Bitwarden 之于 1Password、Baserow 之于 Airtable）。这个区分是本清单的核心编辑观点——读者按目的（学 UI 实现 vs 找可用替代软件）选表即可。

第一张表 "Clones with Tutorials"（第 33-49 行）更进一步：只收"带免费教程"的全栈克隆，每条同时给出 Demo 视频和教程出处（多为 freeCodeCamp 文章与 YouTube 课程），把"克隆项目"延伸到"跟做课程"入口，这是它区别于一般 awesome 清单的最大特色。

项目的传播故事本身也被作者写成博客《My simple GitHub project went viral》（README 第 29 行引用），仓库自 2020 年底走红后成为"克隆学习"类搜索的头部入口，3.9 万星主要由这一波传播与持续被教程/Roadmap 引用累积而来。仓库同时通过 `_config.yml` 用 Jekyll 在 GitHub Pages 上渲染 README，作者另维护 gourav.io/clone-wars 提供更好的表格视图（`src/_config.yml`）。

需要说明的边界：本仓库不含任何被克隆站点的代码或素材，只是一个外链索引；所有商标、版权与合规风险都落在各被收录项目本身（详见第 8 章）。

病毒式传播之外值得注意的是它的"低门槛高密度"：任何人打开 README 第一屏就能看到可直接点开的 demo 链接列表，无需理解项目结构——这种"打开即消费"的体验是它获得大量收藏与 3.3 万 fork 的直接原因。

## 3. 技术栈与依赖

作为内容型仓库，"自身技术栈"极薄（均见 tarball 解包结果）：

- 内容载体：单个 `README.md`（138KB、288 行，但每行表格条目极长），Markdown GFM 表格语法。
- 发布链路：`src/_config.yml` 仅 5 行——Jekyll 主题 `jekyll-theme-leap-day`，启用 `jekyll-relative-links`、`jekyll-github-metadata` 两个插件，即 GitHub Pages 标准渲染，无自定义前端代码。
- 动态星数：条目星数列不用静态数字，而是 shields.io（153 处）与 flat.badgen.net（50 处）徽章 URL，GitHub 渲染时自动显示各被收录仓库的当前星数（README 全文统计）。
- 收录项目的技术栈（作为清单"内容"统计，见第 7 章）：React 系绝对主流，另有 Flutter、Django、Go、PHP/Yii2、Kotlin/Android 等多元分布。

无 package.json / 任何包管理文件、无 CI 配置（tarball 顶层仅 `.github/FUNDING.yml`、README、LICENSE、`_config.yml`、`img/og.png` 与一个误传文件），构建与测试概念不适用于本仓。

## 4. 目录结构与代码组织

| 路径 | 职责 |
|---|---|
| `README.md`（138KB） | 唯一实质内容：说明 + 两张收录大表 + Contribution Guide |
| `_config.yml` | GitHub Pages（Jekyll leap-day 主题）渲染配置 |
| `LICENSE` | AGPL-3.0 全文（34.5KB） |
| `img/og.png` | 社交分享头图（131KB） |
| `.github/FUNDING.yml` | 赞助配置，仅启用 ko-fi（ko_fi: gorvgoyl） |
| `Atividade_Clone_Wars` | 1 字节杂散文件（仅含换行符，葡语"活动"之意，疑误传） |

组织方式评述：典型的"单文件内容仓"——全部信息浓缩在一个 README 里，靠 GitHub 渲染即可消费，无代码、无多包/monorepo 概念。元数据管理依赖外部：星数走徽章外链，完整表格视图在作者自建站。杂散空文件 `Atividade_Clone_Wars` 混在根目录（tar 解包实测仅 1 字节），侧面反映文件级维护较随意。

## 5. 核心架构与关键实现

内容仓的"核心实现"即 README 表格的设计，值得拆开看三点：

1. **五列结构化条目**（`src/README.md` 第 54 行表头）。第二表每行为：`Clone/Alt of | Demo | Repo | Tech stack | Repo Stars`。克隆对象列宽约 30 字符、Demo 列近 170 字符，一行一个项目；Demo 列允许 `<br/>` 并置多个演示/视频链接（如 Amazon 行，第 62 行附近）。表格数据行 206 条，条目平均单行 600+ 字符，这是 README 以 288 行装下 138KB 的原因。

2. **星数徽章而非静态值**。星数列统一为 `![GitHub Repo stars](https://img.shields.io/github/stars/<owner>/<repo>)` 形式（实测首行 bitwarden/mobile 即如此），好处是打开 README 即见当前热度、无需维护数字；代价是 GitHub 外（纯文本、镜像站）不可读，且 shields.io 服务抖动时表格破损。混用 img.shields.io（153 处）与 flat.badgen.net（50 处）两套徽章，风格不统一。

3. **双表 + 贡献规则的编辑准则**（`src/README.md` 第 268-288 行 Contribution Guide）。收录取舍写得很具体：必须是"对知名软件的克隆/替代"、"至少有最小功能、不收纯 UI 克隆"、"不再收 Trello、2048 克隆，除非技术栈不同"、带教程进第一表否则进第二表、按字母序插入。这使清单在开放 PR 模式下仍保持基本质量线。

## 6. 构建与运行

无构建/测试/运行概念。实际可用命令只有两类：

```bash
# 本地渲染 README（可选，等价于 GitHub Pages 行为）
bundle exec jekyll serve   # 依赖 _config.yml 中 jekyll-theme-leap-day 等（src/_config.yml）

# 使用方式：直接阅读 README，或访问
# https://gourav.io/clone-wars（作者自建表格视图，README 第 9 行入口）
```

贡献流程为"直接编辑 README.md 提 PR"（Contribution Guide，第 273-288 行），无自动化校验（无 CI、无链接检查脚本）。

## 7. 清单组织与收录规模

**标题宣称 vs 实测**：README 标题宣称 "100+ open-source clones"。实测（tar 解包 `src/README.md` 逐行统计）：

- 第一表 "Clones with Tutorials"：11 条数据行（每行一个带教程的克隆，含 Moodle、Todoist、Discord 等）。
- 第二表 "Clones and Alternatives"：206 条数据行。
- 合计 **217 条**，覆盖 **107 个**不同的克隆对象类别（第二表 "Clone/Alt of" 列去重计数；若并入第一表对象约 110+ 个）。

结论：宣称 "100+" 是保守下限，实际条目数约为宣称的 2 倍；但"100+ 开源克隆站"与"217 个项目"是两个口径（后者含同一站点的多个实现）。

**克隆对象分布**（第二表按条目数 Top）：Trello 10、Spotify 9、Netflix 9、Twitter 8、Reddit 7、Bit.ly 5、2048 5、WhatsApp 4、Stack Overflow 4、Slack 4、Instagram 4、Hacker News 4、Google Analytics 4；长尾为大量 1 条目对象（如 Zapier、Vimeo、VK、Telegram、Windows 11 等）。Contribution Guide 中"封杀 Trello/2048 新条目"与 Trello 高居榜首（10 条）相互印证了早期收录泛滥后的人为限流。

**条目技术栈标注模式**：每条第 4 列为人工填写的自由文本（非受控词表），统计第二表 206 条：React 及其组合（React、React+Firebase、React+Redux、MERN、React Native/Expo 等）合计约 45 条居首；Flutter、Django、Go、PHP、Vue 系、Kotlin/Android 各若干。标注粒度不均（有的精确到 `Vue 3, TypeScript, ElementUI, Java, Springboot, MySQL, Redis`，有的只写 `JavaScript`），由贡献者自填、无校验。

**代表性条目样例**（`src/README.md` 第二表实读，展示"替代品"与"学习克隆"混排的典型形态）：

| 克隆对象 | 条目 | 技术栈 | 类型 |
|---|---|---|---|
| 1Password / LastPass | Bitwarden mobile（bitwarden.com） | C#, Xamarin | 可用替代品（已归档） |
| Bit.ly | shlink.io、zws.im、kutt.it 等 4 条 | PHP / Node | 可用替代品 |
| Amazon | amazonna.netlify.app（含 YouTube 演示） | 未标注 | 学习克隆 |
| Airbnb | realbnb.vercel.app 等 3 条 | React | 学习克隆 |
| Algolia | MeiliSearch（meilisearch.com） | Rust（表内未标注语言） | 可用替代品 |

**第一表（带教程克隆）细分**：11 条中 freeCodeCamp 提供 4 篇教程（Instagram、Moodle、Todoist、WhatsApp），YouTube 课程占 6 条，覆盖 Airbnb/Netflix/Discord/YouTube 等对象；技术栈从 Next.js+Sanity SDK、React Native+Firebase+Expo 到 Django REST、Yii2 PHP、Android Kotlin 均有（`src/README.md` 第 33-49 行逐行读取）。

**仓库链接分布**：230 条 github.com 链接 + 1 条 gitlab.com（Baserow），几乎全部为 GitHub 生态。

## 8. 条目构成与链接有效性抽检（客观统计）

**抽检方法**：从 README 第二表各区间抽取 5 个仓库链接（用 gh api 核验仓库存在性/归档状态）与 13 个 Demo 链接（curl HEAD/L 实测 HTTP 状态），并对其中高风险类（herokuapp/firebaseapp 免费托管，2022 年后批量失效）加权抽样。抽样时间 2026-11-17，结果只代表抽样时点。

**仓库链接（5/5 存活）**：AChep/2048、AreebKhan619/2048、GShadowBroker/orkut-clone-server、bitwarden/mobile（已归档）、flutter-devs/flutter_paytm_clone 全部返回有效；但除 bitwarden 外最后 push 均停在 2021-2023 年，收录项目本身老化明显。

**Demo 链接（13 个抽检：7 个 200、1 个 403、5 个失效）**：
- 存活：bitwarden.com、zws.im、flagsmith.com、abod-bnb.web.app、prime-clone-e1de6.firebaseapp.com、rowy.io（均 200）；libretranslate.com 返回 403（反爬拦截，站点应为在线）。
- 失效：2048-three.vercel.app、appo-music.herokuapp.com、driwwwle.herokuapp.com、odinclone.herokuapp.com（均无法连接，heroku 免费层 2022 年底关停的直接后果）、start-a-conversation.firebaseapp.com（404）；tinode.co 实测无法连接（该站实际存在，可能为抽检环境网络问题，按"未确认"计）。
- 粗略失效率：按 13 个计约 38%（5/13），若剔除两个存疑项则为 5/11≈45%；且失效集中在 herokuapp 类免费托管，正常域名类存活率高。

**交叉证据**：仓库 open issues 中即有 2026-09-22 提交的《Fix 40 dead demo links: update 5, remove 24, flag the rest》——社区统计 40 个死链并给出处置建议，但该 issue 直至快照仍无人合并处理，与本仓 2024-08 后停止合并 PR 的状态一致。结论：demo 链接腐烂是本清单当前最主要的实用性损耗，GitHub 源码链接则基本健在。

## 9. 与 awesome 类清单的差异及生态位（含合规边界）

**与 awesome-* 清单的差异**（对比常规 awesome 系 bullet-list 组织）：
- 结构化表格（五列固定字段）而非 bullets+分类标题，信息密度和可扫读性更高，但牺牲了 Markdown 层面的可维护性（行超长、PR 冲突频发——Contribution Guide 特意提醒"确保没有合并冲突"）。
- 强制 Demo 列：常规 awesome 清单只要求仓库链接，本清单文化上要求"能在线看到效果"，对学习场景更直接。
- 独立的"带教程"表与 freeCodeCamp/YouTube 课程生态挂钩，使其介于"清单"与"课程索引"之间，这是同类仓（如其他 clone 合集）少见的定位。
- 星数用实时徽章而非快照数字，读感与 awesome 清单的静态描述不同。
- 双出口分发：GitHub README 之外还有作者自建站 gourav.io/clone-wars 与 GitHub Pages 渲染（`src/_config.yml`），awesome 清单通常只有单一 README 入口。

**issues 已事实上变成"投稿收件箱"**：40 个 open issues 中近年新增的均为"Add X clone / Fix dead links"类投稿（2026-06 至 2026-09 连续有新投稿，gh api issues 抽样），因无人合并而长期积压——这是判断"星数在涨、内容已死"最直接的证据。

**商标/版权合规边界（客观陈述）**：仓库本身只收录第三方开源项目链接，不含任何克隆站点的源码、UI 截图（img/ 仅有自制头图 og.png）或品牌素材；README 通篇以"clone of X / alternative to X"描述功能对应关系，未使用各品牌 Logo。此类"克隆知名产品 UI/功能"的争议（商标使用、逆向 UI 设计、克隆功能不侵权但克隆品牌资产侵权等）实际落在各被收录项目与其作者身上；索引者风险主要是链接指向项目的合规问题。该清单将"UI 相似的学习用克隆"与"功能完备的开源替代"并列且明示区分，后者多为合规正面的开源项目（Bitwarden、Meilisearch、Baserow、Formbricks 等），在一定程度上稀释了整体争议性。仓库选用 AGPL-3.0 仅约束其自身内容（一张表）。（依据：tarball 全部文件内容 + 抽检 README 条目；未发现作者对收录项目的合规审查机制，未确认。）

## 10. 总结评价与更新停滞状态

**更新停滞状态（证据链）**：gh api 显示最后 push 为 2024-08-06（"Add mastodon clone: echoloop (#255)"，贡献者 zerone0x），此后 26 个月无合并提交；累计 364 次提交、约 104 位贡献者、0 个 release、open issues 持续有 2026 年的新投稿（如 2026-06/07/08/09 月的加条目与修死链 issue）但全部悬置。README 明言"I'm also looking for a maintainer to merge PRs of new clones"（指向 issue #209），作者本人已退出维护并公开寻找接盘者。星数自 2024 年后仍在缓慢自然增长（快照 39500，2026-11 实查 37292，口径可能有出入），但内容已冻结在 2024-08 状态。

**总评**：Clone-Wars 是"结构化表格 + Demo 文化 + 教程联动"三者叠加做出差异化的内容型仓库，3.9 万星主要由 2020 年底的病毒式传播与本清单一度无可替代的实用性堆成；它的五列表格式一度成为同类索引的模板。但其纯手工维护模式（无 CI、无链接检查、行超长易冲突）注定难以长期保鲜——当前 demo 链接约四成失效、PR 库存积压、清单停留在 2024 年的快照。今天使用它的正确姿势是：把第二表当作"开源替代品选型"的历史索引（替代品类存活率高），把第一表当作 freeCodeCamp 系课程入口；而"克隆知名站点学 UI"的场景，建议连同死链校验一起使用，或转向仍活跃的同类清单。适合前端/全栈学习者按图索骥找练手项目；不适合把条目当作仍然在线的实时资源。

**谁该看它**：想批量研究"知名产品有哪些开源实现"的学习者与讲师；做 awesome 类清单想要借鉴表格化组织的维护者。**谁不该**：需要持续更新的生产级选型目录（应直接用各替代品官网与 OpenAlternative 类活跃目录）。
