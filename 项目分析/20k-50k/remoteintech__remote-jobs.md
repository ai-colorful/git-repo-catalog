# remoteintech/remote-jobs 项目分析

> 星数 34500（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：互联网上最早、规模最大的"远程友好科技公司"社区名录之一，2015 年以巨型 README 起家，现已进化为 Eleventy 驱动的完整网站 remoteintech.company，收录 886 家公司的结构化档案。

| 项目 | 值 | 出处 |
|---|---|---|
| 名称 | remoteintech/remote-jobs（Remote In Tech） | `README.md`、`package.json` |
| 星数 | 34500（快照）；gh api 实测 41,150（2026-11-14），两周内 +6.6k | gh api repos/remoteintech/remote-jobs |
| 创建时间 | 2015-10-13，最近 push 2026-10-30 | gh api |
| 主语言 | JavaScript（Markdown 内容占绝对多数） | 任务参数 + `package.json` |
| 许可证 | ISC（含 MIT/SIL 组件声明） | `LICENSE` |
| 版本 | 4.16.0，changelog 页约定"新增公司升 minor、修错升 patch" | `package.json`、`src/pages/changelog.njk` |
| 技术栈 | Eleventy v3 + Nunjucks + WebC + Tailwind 3 + Pagefind，Netlify 部署 | `package.json`、`CLAUDE.md` |
| 内容规模 | 886 个公司档案（`src/companies/*.md`），约 2200 条外链 URL | 目录统计、`scripts/README.md` |
| 贡献者 | 100+（API 每页 100 已满） | gh api contributors |

## 2. 项目是什么，解决什么问题

找工作时最大的信息不对称是"哪些公司真的允许远程办公"——招聘网站上 job board 遍地都是，但"这家公司是否真心远程、远程到什么程度、招哪些地区"分散在各家 careers 页里。Remote In Tech 用一份人工核验的公司名录回答这个问题：每家公司一个 Markdown 档案，写清公司简介、远程政策、团队规模、地区、技术栈和申请方式（`CONTRIBUTING.md` 定义收录标准）。

项目由 Doug Aitken 于 2015 年 10 月创建（`package.json` author 字段，gh api created_at），是最早一批系统性整理远程工作公司的仓库。前十年它以"一个巨型 README"的形态存在；2026 年 1 月完成大改版（博客文章 `src/blog/2026-01-13-the-big-redesign.md`），迁移为 Eleventy v3 静态站，公司数据从 README 搬进 `src/companies/` 下的独立档案文件，README 现在只剩 29 行指向网站（`README.md`）。

与普通 job board 的本质区别在收录标准而非数量：要求提交者是公司员工或可验证信息来源、公司直接雇佣（排除训练营/猎头/外包平台）、提供真正的远程受薪岗位（排除纯佣金销售）、且属于科技行业（`CONTRIBUTING.md` "What Belongs in the Directory"）。不再达标的公司"直接删除档案而不是改写"（同文件），以此保持名录可信度。

仓库里还保留 16 篇博客（`src/blog/`），2017-2018 年是 WordPress 时代的运营记录，2026 年改版后又恢复更新；其中《Anatomy of a Traffic Spike》（`src/blog/2026-05-07-anatomy-of-a-traffic-spike.md`）分析了 AI 数据集爬虫批量 clone 该仓库的现象（488 clones/140 cloners），侧面印证其作为"训练语料"的地位。

## 3. 技术栈与依赖

内容型仓库，"代码"是为内容服务的站点工程（`CLAUDE.md` 自述基于 Eleventy Excellent 起手架，作者 Lene Saile）：

- **静态站点生成**：Eleventy v3.1.6（`package.json`），Nunjucks 模板 + WebC 组件（`@11ty/eleventy-plugin-webc`）+ eleventy-img 图片自动转 webp/jpeg。
- **样式**：Tailwind 3.4 仅作 design token 处理器，整体采用 CUBE CSS 方法论，设计 token 在 `src/_data/designTokens/`（`CLAUDE.md` Coding Conventions）。
- **搜索**：Pagefind——构建后对 `dist` 生成静态搜索索引，导航下拉即时搜索 + `/search/` 全量页（`CLAUDE.md` Search 节）。
- **Markdown 管线**：markdown-it 全家桶（anchor/attrs/footnote/emoji/link-attributes 等，`package.json` devDependencies）。
- **运行时要求**：Node.js >=22（`package.json` engines、`.nvmrc`=22）。
- **部署/分析**：Netlify + Fathom 隐私分析（仅生产构建加载），旧 URL 重定向由 `src/common/_redirects.njk` 自动生成。

依赖面控制得很好：运行依赖只有 7 个 Eleventy 官方包 + Tailwind，其余全在 devDependencies。`package.json` 里还通过 `overrides` 钉住 nanoid/liquidjs/ws/sharp 的最低版本，是针对传递依赖安全公告的主动升级（`package.json` overrides 字段）；依赖更新交给 `.github/dependabot.yml` 自动维护，另有 CodeQL 静态扫描（`.github/workflows/codeql-analysis.yml`）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `src/companies/` | 核心内容：886 个公司档案，Markdown + YAML frontmatter，另含 `companies.11tydata.js`/`companies.json` 目录级数据 |
| `src/blog/` | 16 篇博客文章 |
| `src/pages/` | 首页、公司列表、标签页、搜索、about/changelog 等 11 个页面模板 |
| `src/_config/` | Eleventy 配置拆分：collections/filters/events/plugins/shortcodes/utils |
| `src/_data/` | 全局数据：`labels.js`（枚举真源）、`companyHelpers.js`（精选公司）、`changelog.json`、designTokens |
| `src/_layouts/`、`src/_includes/` | Nunjucks 布局（base/company/post/tags）与 WebC 组件 |
| `src/assets/` | CUBE CSS 分层样式、字体、图片、前端脚本 |
| `.github/scripts/` | `validate-companies.js` 校验机器人脚本 |
| `.github/workflows/` | CI、公司档案校验、CodeQL、dependabot |
| `scripts/` | 维护者用的链接巡检管道（`check-links.sh` + `fix-links.mjs`），不参与构建 |
| `tests/` | `collections.test.js` 集合逻辑集成测试 |
| `.github/` | `CONTRIBUTING.md`/`CODE_OF_CONDUCT.md`/`SECURITY.md`、issue 模板（bug + new_company）、PR 模板、FUNDING、dependabot 与 CodeQL 配置 |

顶层另有 `.nvmrc`（Node 22）、`.nojekyll`（Pages 兼容残留）、`.env-sample` 与 `.prettierrc`/`.prettierignore` 等常规工程卫生文件（目录列表）。值得注意的一点：仓库的"README 主体"时代已经结束——`README.md` 仅 29 行，真正的清单主体是 `src/companies/` 下 886 个档案文件，这是它与典型 awesome 清单仓库的最大结构差异。

组织方式评述：典型的"内容 + 薄工程"结构——内容按一公司一文件的扁平清单组织（886 个文件全部平铺在一个目录，靠 slug 全局唯一）；工程代码按 Eleventy 社区惯例拆成 config/data/layouts/includes 四层，`CLAUDE.md` 是面向 AI 代理与人的工程导览，`CONTRIBUTING.md` 是面向投稿人的内容规范，两者分工明确。

## 5. 核心架构与关键实现

**（1）公司档案的数据模型与枚举真源。** 每个档案的 frontmatter 含 `title/slug/website/careers_url/region/remote_policy/company_size/technologies/addedAt/updatedAt` 十个字段。所有合法枚举值集中定义在 `src/_data/labels.js`（region 6 值、remote_policy 4 值、company_size 5 值、technologies 25 值），注释明确写着"single source of truth——校验脚本和站点渲染从同一处取值，改一处两端同步生效"（`src/_data/labels.js` 头部注释、`.github/scripts/validate-companies.js` 第 18-24 行 import）。这是整个数据质量体系的锚点。

**（2）零依赖校验机器人。** `.github/scripts/validate-companies.js`（253 行）不引任何 YAML 库，手写了一个约 40 行的 frontmatter 解析器处理键值对/数组/引号剥离，然后校验：必填字段、枚举合法性（从 `labels.js` 动态取）、slug 必须等于文件名、URL 格式、三个必需章节（`## Company blurb`、`## Remote status`、`## How to apply`）、空 blurb 告警，并识别 `company-profiles/` 等旧格式路径引导投稿者迁移（`.github/scripts/validate-companies.js` REQUIRED_FIELDS/REQUIRED_SECTIONS 与 oldFormatFiles 逻辑）。

**（3）PR 校验工作流的安全设计。** `.github/workflows/validate-companies.yml` 用 `pull_request_target`（才能给新人 PR 发评论），但只 sparse-checkout PR 里的公司数据文件、`persist-credentials: false`、注明"fork 内容永不执行"，另用 API 而非 diff 列变更文件以正确处理"删除档案"的 PR；机器人评论带 `<!-- validate-companies -->` 标记实现幂等更新；仓库主和 Bot 免校验。这是 fork PR 自动化里相当规范的安全姿势。

**（4）集合与缓存的性能意识。** `src/_config/collections.js` 用 WeakMap 按 build 记忆化 glob 结果，注释直言"避免把约 850 个条目过滤 6 次"；集合分 alphabetical 全量、精选轮换（12 选 8 随机）、按 addedAt 取最近 12 家、按 region/technology 分组等 9 个（`src/_config/collections.js`、`eleventy.config.js`）。

**（5）链接腐烂巡检管道（人工触发的"自动化维护"）。** `scripts/check-links.sh`（239 行）对全部约 2200 条 URL 做 20 并发 curl 巡检输出 CSV；`scripts/fix-links.mjs`（650 行）把结果分为 11 类——HTTPS 升级、同域名重定向、死掉的 careers 页自动安全修复，PARKED（停放页）/DEAD_DNS（域名失效）直接删除公司档案，REBRAND（换域名，疑似被收购）等留人工审（`scripts/README.md` 分类表）。这不是 CI 定时任务，而是维护者策展时手动跑的批处理，但配合 `updatedAt` 日期字段构成了名录保鲜机制。

## 6. 构建与运行

来自 `README.md`、`CLAUDE.md` 与 `package.json` scripts（本次未实际运行，静态核对）：

```bash
npm install          # 安装依赖（Node 22+）
npm run start        # 开发服务器，热重载（eleventy --serve）
npm run build        # 生产构建：clean + eleventy + pagefind 索引
npm run build:11ty   # 仅 Eleventy 构建（调试更快）
npm run clean        # 清理 dist 与生成物

node .github/scripts/validate-companies.js src/companies/xxx.md   # 校验单/多档案
./scripts/check-links.sh [--quick|--refresh]                       # 链接巡检（约 8 分钟）
node scripts/fix-links.mjs [--reverify|--apply]                    # 链接修复
```

CI（`.github/workflows/ci.yml`）在 push/PR 时执行 `npm ci && npm run build`，需 `NODE_OPTIONS=--max-old-space-size=8192`，并统计生成的 HTML 页数验证产物。本次分析按规范约束未实际运行构建（只做静态阅读），但 CI 的存在意味着每个 PR 至少通过"能否完整构建出全站"这一最低门槛；`npm run clean` 会同时清掉 `src/_includes/css|scripts` 这两个由构建事件（`src/_config/events/build-css.js`/`build-js.js`，esbuild）生成、再注入模板的目录。

## 7. 收录规模与分类体系

**规模统计**（基于 `src/companies/` 886 个 `.md` 文件的 frontmatter 全量统计）：

- **远程政策**：fully-remote 404（45.6%）、remote-friendly 253、hybrid 137、remote-first 92。即约一半公司"无办公室、全远程"，四档分类把"semi to fully remote"的原生描述量化成了可筛选的维度。
- **地区**：worldwide 405、americas 184、other 147、europe 84、americas-europe 42、asia-pacific 24——worldwide（全球可雇）是最大类别，亚太覆盖明显薄弱。
- **规模**：small(11-50) 276、medium(51-200) 214、tiny(1-10) 147、large(201-1000) 126、enterprise(1000+) 89——中小公司占七成，符合"远程友好早期采用者多为科技公司"的画像。
- **技术标签**：共 3192 个标签实例，javascript 636、cloud 266、devops 252、sql/python 各 247、go 204、ruby 180、java 172 居前；同时存在 linux(38)/infrastructure(38)/graphql(32)/api(31) 等**枚举外遗留标签**，说明存量档案早于当前校验规则（对比 `src/_data/labels.js` 枚举）。
- ** careers_url 覆盖**：616/886（70%）有独立招聘页，其余回落到官网。

**组织方式**：单层扁平列表 + frontmatter 结构化字段 + 网站端按 region/tech/size 三轴动态分组聚合（`src/_config/collections.js` 的 companiesByRegion/companiesByTech），首页展示精选轮换与最近收录，辅以 Pagefind 全文搜索。

**首页两个策展位**（`src/_config/collections.js`、`src/_data/companyHelpers.js`）：精选区从维护者手工圈定的 `featuredCompanySlugs` 名单（12 家）中每次构建随机轮换展示 8 家；"最近收录"按 frontmatter 的 `addedAt` 取最新 12 家。此外 `src/_data/companyTags.js` 预生成带计数的标签页数据，用于浏览页的导航统计。

**博客与运营内容**：`src/blog/` 的 16 篇文章一半是 2017-2018 年 WordPress 时代的运营日志（上线公告、流量分析、Hacktoberfest 动员），一半是 2026 年改版后的工程与运营复盘（大改版、SEO、数据清理、爬虫流量侦探记），构成项目的"编年史"证据链。

## 8. 自动化维护与社区贡献

**贡献模式（PR 制 + 机器人把关）**：投稿者在 `src/companies/{slug}.md` 按 frontmatter 模板（`CONTRIBUTING.md` 给出完整模板与合法值表）写档案 → 本地 `npm run build` 自验 → 开 PR → Validate Company Profiles 工作流自动校验并以评论形式反馈问题，未修复前阻断合并（`CONTRIBUTING.md` Validation 节、`.github/workflows/validate-companies.yml`）。`addedAt/updatedAt` 由维护者管理、投稿者禁止填写。另有 "New Company Request" issue 模板（`.github/ISSUE_TEMPLATE/new_company.md`）承接不想写 Markdown 的用户提供信息。

**收录门槛是内容质量的核心**：员工/可验证来源、直接雇佣、真远程受薪岗、科技行业、careers 页指向公司自身（`CONTRIBUTING.md`）。拒绝项包括有害服务、空内容、重复条目；不再达标的公司删档而非改写。

**维护节奏**：维护者手动运行链接巡检管道处理链接腐烂（`scripts/README.md` 自述 "ad-hoc scripts the maintainer runs when curating"）；版本号与 `src/_data/changelog.json` 驱动的 changelog 页面记录每次增删改（`src/pages/changelog.njk`）。

**增长周期与星数曲线的对应关系**（addedAt 年度分布统计）：2015 年创建当年即 69 家，2018 年达到峰值 169 家（早期热潮）；2020-2022 疫情远程办公高峰期收录 322 家（101/99/122），是第二个高峰；2024-2025 回落至 36/22 家；2026 年 1 月大改版后回升到 45 家（截至 10 月）。收录节奏与远程办公热潮周期高度吻合。星数仍在加速：任务快照 34.5k（2026-10-24），gh api 实测 41.2k（2026-11-14），两周 +6.6k，与 2026 年改版后的 SEO 优化（`src/blog/2026-01-17-seo-improvements.md`）和持续传播相符；仓库 topics 含 hacktoberfest，历史上也借势过 Hacktoberfest（`src/blog/2018-09-25-hacktoberfest-is-back.md`）。

## 9. 亮点与风险

**亮点**：

1. 十年持续运营的内容资产：2015 年至今 886 家公司档案，是远程工作领域最老牌、被引用最多的名录之一，也是 AI 时代被批量抓取的语料源（`src/blog/2026-05-07-anatomy-of-a-traffic-spike.md` 记录了 488 clones/天的爬取事件）。
2. 数据质量体系完整闭环：`labels.js` 单一枚举真源 → 零依赖校验脚本 → 安全规范的 PR 机器人（sparse-checkout + persist-credentials:false）→ 链接腐烂巡检与自动修复 → 删档机制，内容型仓库里少见地把"工程严谨度"用在了"内容严谨度"上。
3. 从 README 到真网站的成功演化：2026 年 1 月 Eleventy 大改版保留了十年积累的 URL（自动重定向），把静态清单升级为可筛选、可搜索、带 changelog 的产品。
4. 前端工程质量高：CUBE CSS + design token、Pagefind 零服务端搜索、WebC 组件、图片自动优化、集合 WeakMap 缓存，依赖树极小（运行依赖 7 个）。

**风险/局限**：

1. 信息时效天然衰减：886 家公司中相当部分 `updatedAt` 停留在多年前（如最早的 10up 停在 2018-04-16，`src/companies/10up.md`），公司远程政策在 2023 年后大规模回撤（RTO），名录可能系统性偏乐观——链接巡检能治 URL 腐烂，治不了政策变化。
2. 数据一致性尚未收口：存在 1 个枚举外 `company_size: startup`（`src/companies/maintainnow.md`）和 100+ 枚举外技术标签（linux/graphql/api 等），说明校验规则对存量档案是"只查新增"。
3. 关键人风险：核心维护者单一（Doug Aitken），机器人脚本、链接修复、精选名单都围绕其个人工作流（`scripts/README.md` 自述 ad-hoc），巴士因子低。
4. 亚太覆盖薄弱（region: asia-pacific 仅 24 家，1/37 于 worldwide），对非欧美求职者价值有限。

## 10. 总结评价

这是"内容型开源仓库如何工程化运营"的教科书样本：以 2015 年一个 README 清单起步，靠先发优势、严格的收录标准和对远程办公浪潮每个周期（2018 热潮、2020 疫情、2026 改版）的持续响应积累了 34k+ 星，再用一套小而精的自动化（枚举真源 + PR 校验机器人 + 链接巡检管道）守住十年内容资产的质量底线。它几乎不含可复用的"业务代码"，但其校验脚本、fork-PR 安全工作流和链接腐烂治理方案对任何维护清单型开源项目的人都值得直接借鉴。适合求职者当远程公司调研起点（注意核对时效）、内容仓库维护者当自动化范本学习；读"源码"的正确姿势是读 `CONTRIBUTING.md`、`CLAUDE.md` 和 `.github/scripts/validate-companies.js` 这三件套。
