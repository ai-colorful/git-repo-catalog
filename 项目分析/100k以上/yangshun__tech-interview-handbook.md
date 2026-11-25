# yangshun/tech-interview-handbook 项目分析

> 星数 143,123（快照 2026-10-24）｜主语言 TypeScript｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：面向"忙碌工程师"的技术面试免费准备手册——以 Docusaurus 内容站为主体、Next.js 全栈门户为延伸的内容型 + 站点工程混合仓库，作者是 Blind 75 的作者、前 Meta 工程师 Yangshun Tay。

| 项目 | 事实 |
| --- | --- |
| 仓库 | yangshun/tech-interview-handbook |
| 星数 / Fork | 143,123（GitHub API 实测 143,182）/ 16,874（`gh api repos/yangshun/tech-interview-handbook`） |
| 主语言 | TypeScript（内容为 Markdown/MDX） |
| 许可证 | MIT（`LICENSE`，Copyright 2017-Present Yangshun Tay） |
| 创建时间 | 2016-07-05 |
| 最近 push | 2026-08-07（GitHub API），仓库仍在维护 |
| 官网 | https://www.techinterviewhandbook.org |
| 贡献者 | 约 186 人（contributors API 分页 last=187） |

## 2. 项目是什么，解决什么问题

技术面试准备材料浩如烟海，但多数面试资源是"外部链接的堆砌"，或者只覆盖算法题。本项目（`README.md`）的差异化主张是：**直接提供原创、精炼、成体系的策展内容**，覆盖求职全流程——简历、自我介绍、 coding 面试技巧、算法速查表、行为面试、与公司选择、薪资谈判，直到 offer 决策。作者明确表示"不给上百道 LeetCode 的时间，就给最浓缩的必需知识"。

内容侧最知名的两张名片：一是 **Blind 75**（作者 2018 年在 Blind 论坛发布的 75 题精选清单，本仓库 README 自述出处），二是其进化版 **Grind 75**（可按剩余周数和每周时长动态生成刷题计划的工具，官网导航栏直达；工具本体部署在 `grind75.pages.dev`，仓库内通过 Cloudflare Pages Function 反代：`apps/website/functions/grind75/[[catchall]].js` 将 `/grind75/*` 请求改写到该域名）。

算法部分（`apps/website/contents/algorithms/`）按数据结构/题型组织为 21 篇专题速查（array、string、hash-table、linked-list、stack、tree、graph、dynamic-programming、interval、trie 等），每篇统一模板：概览、学习资源、语言库推荐、时间复杂度表、面试注意事项、corner cases、实用技巧与配套练习题（见 `algorithms/study-cheatsheet.md` 的 "Contents of each study guide"）。

非算法部分是本仓库相对同类项目（如各类 awesome 清单）的核心价值：`resume.md`（418 行的 FAANG 简历指南）、`behavioral-interview.md` 系列含评分细则（`behavioral-interview-rubrics.md`）与高级候选人版、`negotiation.md` / `negotiation-rules.md` 谈判守则、`understanding-compensation.md` / `choosing-between-companies.md` 补偿与选 offer、甚至还有面向面试官的 `interviewer-cheatsheet.md`。工程侧则有 `coding-interview-rubrics.md` 这样的评分维度文件。

流量入口为 Docusaurus 网站，互动产品（简历互评、offer 数据分享、题库）在 Next.js 门户应用中，商业化通过联盟课程推荐与站内广告位实现（详见第 5 章）。

## 3. 技术栈与依赖

pnpm monorepo（`pnpm-workspace.yaml`：`apps/*` + `packages/*`），根 `package.json` 指定 Node 25.8.1、pnpm 10.32.1。

- **统一工具链 vite-plus（voidzero）**：根脚本全部经 `vp` CLI（`package.json`：`build: vp run --cache -r build`、`ci: vp check && vp test && vp run --cache -r build`）；`pnpm-workspace.yaml` 的 catalog 把 `vite` 映射到 `@voidzero-dev/vite-plus-core`。`AGENTS.md` 是专门写给 AI 编码代理的 Vite+ 使用规范（禁止直接用 pnpm、必须从 `vite-plus` 导入 defineConfig 等），属近期（2026-08）工具链迁移的直接证据。
- **apps/website — Docusaurus 2.4.3**（`apps/website/package.json`）：`@docusaurus/core` + preset-classic + plugin-client-redirects + theme-common，React 18；自swizzle 主题组件（`src/theme/DocItem/`、`src/theme/DocSidebar/`、`src/theme/TOC/`）。
- **apps/portal — T3 全栈**（`apps/portal/package.json`）：Next.js 14.2.4、tRPC 9、Prisma 4、next-auth 4、Supabase JS、react-query 3、Tailwind（共享配置 `packages/tailwind-config`）、zod；文件上传用 formidable + react-pdf，Excel 种子数据用 read-excel-file/xlsx。`ct3aMetadata.initVersion: 5.13.1` 表明由 Create T3 App 脚手架初始化（`apps/portal/README.md`）。
- **共享包**：`packages/tsconfig`（base/nextjs/react-library 三套 tsconfig）、`packages/tailwind-config`。
- 质量工具集中在根 `vite.config.ts`：Oxlint 规则集（camelcase、eqeqeq、prefer-const、react/jsx 系列、typescript/consistent-type-imports 等）+ Oxfmt 格式化 + staged 钩子（`'*': 'vp check --fix'`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `apps/website/` | Docusaurus 内容站（主站 techinterviewhandbook.org） |
| `apps/website/contents/` | 全部正文：28 篇顶层指南 + `algorithms/` 21 篇专题 + `_components/`（MDX 组件：InDocAd、QuestionList）+ `_courses/`（课程推荐页） |
| `apps/website/src/` | 定制主题（DocItem/DocSidebar/TOC）、首页 `src/pages/index.js`、`src/data/successStories.js` 成功案例、SidebarAd 广告组件 |
| `apps/website/blog/` | 13 篇博客（2019-2022，Meta 工作经历、前端面试观察等） |
| `apps/website/functions/grind75/` | Cloudflare Pages Function，把 `/grind75/*` 反代到 grind75.pages.dev |
| `apps/website/experimental/` | 实验性内容（front-end 迁出后的残留、topics、utilities） |
| `apps/portal/` | Next.js 14 全栈门户（约 3.3 万行 TS/TSX，`find apps/portal/src -name '*.ts*' \| xargs wc -l`）：简历互评、offer 薪资数据、题库、待办 |
| `apps/portal/prisma/` | `schema.prisma`（30+ model）+ 迁移 + 种子脚本（seed-salaries/seed-questions/seed-companies 等）+ companies.csv、salaries.xlsx 原始数据 |
| `packages/tsconfig`、`packages/tailwind-config` | 共享工程配置包 |
| `algorithms/`（仓库根） | 仅剩一个跳转 README（`algorithms/README.md`：内容已移至 website） |
| `assets/`、`.github/workflows/` | Logo 与社交图；CI 工作流 |

组织方式评述：标准的 pnpm workspace 双应用结构——"内容站 + 用户产品门户"分离。内容（Markdown）与站点代码（React/主题）同仓但目录隔离，MDX 组件让正文可内嵌广告与交互组件。根目录 `algorithms/` 的存根 README 说明内容经历过大迁移（旧扁平目录 → apps/website/contents），历史包袱清理得比较干净。

## 5. 核心架构与关键实现

**（1）内容引擎：Docusaurus + MDX 组件化**（`apps/website/contents/`）
正文共 50 个 md/mdx 文件、5,831 行、594 个外链（统计自 `find apps/website/contents -name '*.md*'`）。组织上 sidebars.js 手工维护章节树（Introduction → Getting an interview → Coding interview preparation → …），每篇 frontmatter 自带 SEO 元数据，并内嵌 `<head>` OG 图。正文可插入 React 组件，例如 `algorithms/study-cheatsheet.md` 顶部 `import InDocAd from '../\_components/InDocAd'`——这是内容与商业化在文件级别耦合的直观体现。

**（2）广告/变现组件**（`apps/website/src/components/SidebarAd/index.js`）
侧边栏与文内广告位组件，每 20 秒轮换（`const AD_REFRESH_RATE = 20 * 1000`），带 gtag 埋点（`window.gtag('event', 'faangtechleads...')`），投放 FAANG Tech Leads 简历模板、GreatFrontEnd（`docusaurus.config.js` announcementBar 亦为 GreatFrontEnd 联盟横幅）等自家/联盟产品。README 中 AlgoMonster、Design Gurus 的 shareasale/designgurus 联盟链接同样成体系。这是一套内容免费 + 流量变现的完整闭环。

**（3）门户后端：tRPC 路由矩阵 + Prisma 领域模型**（`apps/portal/src/server/`）
三大业务域各自由约 30 个 tRPC 路由文件组成（`server/router/offers/`、`resumes/`、`questions/`，以及 companies/locations 公共路由），每个域拆分主路由、user 路由（本人数据）、投票路由等。对应 `apps/portal/prisma/schema.prisma` 中 30+ model：`OffersProfile/OffersOffer/OffersAnalysis`（offer 提交与自动分析）、`ResumesResume/ResumesComment/ResumesStar`（匿名简历分享与点赞评论）、`QuestionsQuestion/QuestionsQuestionEncounter/QuestionsAnswer/QuestionsList`（题库、被问记录、答案与清单）、外加 `Company/Country/State/City` 地点字典与 next-auth 的 `User/Account/Session`。种子脚本直接吃仓库内的 `companies.csv` 和 `salaries.xlsx`（`prisma/seed-salaries.ts`）。

**（4）Grind 75 边车**（`apps/website/functions/grind75/[[catchall]].js`）
整个文件仅十余行：Cloudflare Pages Function 把 `/grind75/*` 的请求原样转发到 `https://grind75.pages.dev`。刷题计划工具与主仓解耦部署、用反代保持域名统一，是"站点工程极简主义"的典型做法。

## 6. 构建与运行

来自根 `package.json` 与两个 app 的 `package.json`（工具链为 vite-plus 的 `vp`，要求 Node 25.8.1 + pnpm 10.32.1）：

```bash
vp install                        # 安装依赖（vite-plus 包装 pnpm）
vp dev                            # 启动 website 开发服务器（docusaurus start）
vp run --filter @tih/portal... dev  # 启动 portal（next dev）
vp run --cache -r build           # 构建全部工作区
vp check                          # 格式 + lint + 类型检查
vp test                           # Vitest（当前无测试，passWithNoTests）
```

portal 专用：`vp run --filter @tih/portal... tsc`（类型检查）、`ts-node prisma/seed-salaries.ts` 等四个种子脚本；运行门户需要 Postgres（DATABASE_URL）、Supabase、NextAuth/GitHub OAuth 等环境变量（CI 里用假值占位，见 `.github/workflows/tsc.yml` 的 env 块）。website 部署走 Docusaurus deploy / Cloudflare Pages（functions 目录），portal 部署面向 Vercel（`apps/portal/README.md`）。

## 7. 测试与工程质量

- **测试**：全仓搜索不到任何 `*.test.*` / `*.spec.*` 文件；根 `vite.config.ts` 配置 `test: { passWithNoTests: true }`，CI 也只跑 `vp test` 形式化过场。内容型项目的典型取舍——质量靠 lint + 类型 + 人工评审保证。
- **CI**（`.github/workflows/`）：两个 PR 工作流——`lint.yml`（名为 Check：`vp check` + `vp test`，用 `voidzero-dev/setup-vp@v1`，带 concurrency 取消与缓存）、`tsc.yml`（名为 Build：分别构建 website 与 portal，占位环境变量）。均为 2026-08 的 Vite+ 迁移产物。
- **代码质量观察**：根 `vite.config.ts` 内联了相当严格的 Oxlint 规则集（eqeqeq、prefer-const、consistent-type-imports、react/button-has-type 等）与 staged 提交钩子（`vp check --fix`）；portal 是 T3 脚手架生成的规范分层（router/db/context 分离）。不足：website 仍是 Docusaurus 2.4.3（3.x 已发布多年，未确认是否有升级计划）；portal 依赖偏旧（tRPC v9、Prisma 4、next-auth 4）。

## 8. 社区与维护状态

- **活跃度**：GitHub API 显示最近 push 2026-08-07（本次分析前约 3 个月），当月动作是整仓迁移到 vite-plus 工具链（新增 `AGENTS.md`、改写两个工作流、Node 升到 25.8.1），说明仓库处于"内容稳定、工程工具持续现代化"的状态；open issues 38 个。
- **历史**：2016-07 创建，十年长跑；作者 Yangshun Tay 为 Blind 75 原作者、前 Meta 工程师（README 免责声明自述）。贡献者约 186 人，社区贡献以内容 PR 为主（README 有 OpenCollective backers/sponsors 展示位）。
- **release 节奏**：未发现 release/tag 与 CHANGELOG（站点型项目以持续部署为主，未确认发布方式细节）。
- **生态位**：与作者自己的 Front End Interview Handbook（frontendinterviewhandbook.com，内容已迁出）、Grind 75 工具、GreatFrontEnd 付费平台构成矩阵；README 还互推了 `yangshun/lago`（JS 数据结构库）。

## 9. 亮点与风险

**亮点**
1. 内容策展质量高且全流程覆盖：从简历到谈判的 28 篇指南 + 21 篇算法速查，原创成体系而非链接堆砌（`apps/website/contents/`），是 143k 星的根基。
2. 内容 + 产品 + 变现的完整工程闭环：Docusaurus 站、Next.js 门户（简历互评/薪资数据/题库，30+ Prisma model）、联盟广告组件（`SidebarAd/index.js` 带轮换与埋点）各就各位。
3. 工程工具链激进且统一：2026-08 迁移到 vite-plus 一把梭（`vp` 统一 dev/check/test/build），并配套 `AGENTS.md` 让 AI 代理按规矩干活，在同类内容仓库中罕见。
4. 品牌资产自持：Blind 75 → Grind 75 的品牌演进、独立域名反代（`functions/grind75/[[catchall]].js`），SEO（frontmatter、OG 图、client-redirects 插件）做得细致。
5. CI 虽薄但干净：类型检查 + 严格 lint + staged 钩子 + 分应用构建门禁。

**风险/局限**
1. 零自动化测试（`passWithNoTests: true`），门户 3.3 万行 TS/TSX 全靠类型与 lint 兜底，重构风险随规模累积。
2. 门户新旧技术代差明显：Docusaurus 2.4.3、tRPC v9、Prisma 4、next-auth v4 均落后主流一到两代，升级欠账在增加。
3. 内容更新趋缓：正文最后成批修改集中在早中期，博客停更于 2022（`apps/website/blog/`），部分指南（如 System Design）仍标记"制作中"并导流到付费课程（README）。
4. 商业化密度对开源社区观感的影响：README 与正文内嵌多处理联盟推广链接（shareasale 等），开源纯粹度上见仁见智。

## 10. 总结评价

这是"内容即产品"的教科书式开源项目：用十年把个人面试笔记做成 143k 星的求职第一站，再以极小的工程代价（一个 Docusaurus 站 + 一个 T3 门户 + 一个 Cloudflare 反代）把流量转化为社区产品与可持续收入。它不适合作为代码架构范本来精读——代码以脚手架和配置为主、几乎没有测试——但对"如何运营一个内容型开源项目"极具参考价值：内容组织（统一模板的算法速查、分阶段指南树）、SEO 与分发（官网、社交矩阵、品牌工具引流）、变现闭环（文内广告 + 联盟课程 + 赞助位）、以及近期把工具链现代化并写 `AGENTS.md` 迎接 AI 协作的做法。适合求职者当手册用，适合内容型开源维护者当运营蓝图学，适合工程侧读者只看 `apps/portal` 的 tRPC/Prisma 分域设计与 vite-plus 迁移形态。
