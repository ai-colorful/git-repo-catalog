# anuraghazra/github-readme-stats 项目分析

> 星数 79842（快照 2026-10-24）｜主语言 JavaScript｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：在 GitHub README 里嵌入"动态生成的 SVG 统计卡片"（用户统计、语言榜、仓库钉选、Gist 钉选、WakaTime），以 Vercel Serverless 函数按需渲染，是 GitHub Profile 美化生态的开创者与事实标准。

| 项目 | 内容 |
|---|---|
| 名称 | anuraghazra/github-readme-stats |
| 星数 | 79842（快照 2026-10-24）；forks 38191（gh api 元数据） |
| 主语言 | JavaScript（纯 ESM，`"type": "module"`，见 `package.json`） |
| 许可证 | MIT（`package.json` `license` 字段；附 `LICENSE` 文件） |
| 默认分支 | master（gh api） |
| 最近 push | 2026-10-01（gh api；README 已挂"停止维护"横幅） |
| 运行形态 | Vercel Serverless Functions（`vercel.json`），Node >= 22 |

## 2. 项目是什么，解决什么问题

GitHub 的个人 Profile 页只是一个 README 仓库，默认没有"数据看板"。本项目把这个空白填上：用户在 README 里写一行 `![](https://github-readme-stats.vercel.app/api?username=xxx)` 形式的图片链接，Vercel 上的 serverless 函数就实时调用 GitHub API，取回该用户的提交、PR、Issue、Star、关注者等数据，在服务端渲染成一张 SVG 卡片返回。因为 Markdown 图片里的 URL 会被 GitHub 的 Camo 代理抓取，SVG 卡片可以像图片一样嵌入任何 README，且每次缓存过期后自动刷新（`readme.md` 用法章节）。

项目提供五类卡片：GitHub Stats Card（用户总统计 + 等级评级）、Top Languages Card（语言占比，支持条形/紧凑/环形/竖环形/饼图五种布局）、Repo Pin Card（模拟 GitHub 仓库钉选，可"额外钉选"超过 6 个仓库）、Gist Pin Card、WakaTime Card（编程时长）。参数全部通过 URL query 传递，如 `hide`、`theme`、`locale`、`bg_color`（支持多色渐变）、`cache_seconds` 等（`readme.md`，`api/index.js` 的 query 解构）。

架构上的核心约束是"必须跑在免费 serverless 里、必须扛住全网 README 的图片请求量"。为此代码只依赖 5 个运行时库（axios、dotenv、emoji-name-map、github-username-regex、word-wrap，见 `package.json` `dependencies`），SVG 完全用模板字符串手工拼装，不引入任何渲染框架；同时用 HTTP 缓存头把压力转嫁给 Vercel Edge/CDN（`src/common/cache.js`），用多 PAT 轮换对抗 GitHub API 限流（`src/common/retryer.js`）。

仓库当前状态特殊：README 顶部有 CAUTION 横幅，声明"本仓库不再维护，请使用后继项目 GitHub Stats Extended（stats-organization/github-stats-extended）"，旧文档整体折叠进 "outdated documentation" 折叠块保留（`readme.md` 第 28-31 行）。源码功能完整，但官方已停止演进。

## 3. 技术栈与依赖

- 语言：纯 JavaScript（ESM），未用 TypeScript，但全部源码文件头部带 `// @ts-check` 且以 JSDoc 标注类型，借 VS Code/ESLint 获得准静态检查（如 `src/fetchers/stats.js`、`src/common/Card.js`）。
- 运行时依赖（`package.json`）：axios（GitHub REST/GraphQL 请求）、dotenv、emoji-name-map（语言卡片 emoji）、github-username-regex（入参校验）、word-wrap（文本折行）。依赖面极小，是能塞进 128MB serverless 函数并保持冷启动快的关键。
- 部署：Vercel Functions，`vercel.json` 限定 `api/*.js` 内存 128MB、maxDuration 10 秒，根路径 302 到 GitHub 仓库。
- 本地开发：`express.js` 用 Express 5 把 5 个 serverless handler 挂到 `/api/*` 路由上（devDependencies 中的 express 仅为此服务）。
- 工程链：Jest 30（含 e2e 与 benchmark 配置）、ESLint 9 flat config（`eslint.config.mjs`，含 eslint-plugin-jsdoc）、Prettier、husky + lint-staged、Codecov。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `api/` | Vercel serverless 入口：`index.js`（统计卡）、`pin.js`、`top-langs.js`、`gist.js`、`wakatime.js`；`status/up.js` 与 `status/pat-info.js` 是 PAT 健康检查端点 |
| `src/cards/` | 各卡片的 SVG 渲染器：`stats.js`（602 行）、`top-languages.js`（965 行）、`wakatime.js`、`repo.js`、`gist.js` |
| `src/fetchers/` | 数据层：每个卡片对应一个 fetcher，封装 GraphQL/REST 查询与重试 |
| `src/common/` | 横向公共件：`Card.js`（SVG 卡片基类）、`render.js`（布局/测宽/进度条）、`cache.js`、`retryer.js`、`color.js`、`error.js`、`access.js`、`http.js`、`ops.js`、`fmt.js` 等 |
| `src/calculateRank.js` | 等级评级算法（S/A+/A/…/C） |
| `src/translations.js` | i18n：4 组文案字典，约 41 个语言（该文件 1105 行） |
| `themes/` | `index.js` 内置 75 套主题（色板常量） |
| `tests/` | 单测 + `__snapshots__` 快照 + `e2e/`（真实调用线上 API 的冒烟）+ `bench/`（Jest 基准） |
| `scripts/` | 运维脚本：主题文档生成、语言 JSON 生成、过期主题 PR 关闭等 |
| `.github/workflows/` | CI/自动化：test、e2e、CodeQL、OSSF Scorecard、theme PR 管理等 16 个 workflow |
| `express.js` / `vercel.json` | 本地 Express 开发服务器 / Vercel 部署配置 |

组织方式评述：单体仓库、单包、零框架的小型代码库（全部源码约 6900 行 JS，`wc -l` 统计）。分层清晰：`api/*` 只做参数解析与错误兜底，`fetchers/*` 只做取数，`cards/*` 只做渲染，公共逻辑沉到 `common/`。每个卡片都是"一个 fetcher + 一个 renderer + 一个 api 入口"的三件套，扩展新卡片即是复制这条垂直切片。

## 5. 核心架构与关键实现

### 5.1 SVG 卡片渲染器（src/common/Card.js + src/common/render.js）

`Card` 是所有卡片的基类：负责外框、标题、渐变背景、动画 CSS 与无障碍标签，子类填充 body。渐变背景支持 `bg_color` 传数组时生成 `linearGradient`，并用 CSS `@supports(-moz-appearance: auto)` 单独修正 Firefox 的字体渲染（`src/common/Card.js` `render()`）。所有用户可控文本先经 `encodeHTML` 转义再插入模板，防止 SVG 注入（`src/common/html.js`、`Card.js` 构造函数）。

服务端没有浏览器，无法用 canvas 量文本宽度，`measureText` 用一张手工 ASCII 字符宽度表 + 非 ASCII 平均宽度估算任意字符串的像素宽，供自动布局使用（`src/common/render.js` 197 行起）；`flexLayout` 则以 `translate` 变换实现横向/纵向 flex 排版（`render.js` 第 18 行起）。这两件事是"零依赖手写 SVG 排版"的根基。

### 5.2 GraphQL 数据层与 PAT 轮换重试（src/fetchers/stats.js + src/common/retryer.js + src/common/http.js）

统计卡的数据来自一条 GraphQL 查询，用 `@include(if: $includeMergedPullRequests)` 等指令按 query 参数动态增删字段，避免多次往返（`src/fetchers/stats.js` 41-79 行）。Star 总数需要对用户仓库分页累加，`statsFetcher` 用 while 循环翻页；但在公共实例上因限流默认只取第一页，仅自部署设置 `FETCH_MULTI_PAGE_STARS=true` 才全量翻页（`stats.js` 148-155 行）。`include_all_commits` 时因 GraphQL 无法取全量提交，改走 REST `search/commits`（`stats.js` 170-180 行）。

对抗限流的重试器是本项目的招牌设计：启动时数出环境变量里 `PAT_1`、`PAT_2`… 的个数作为重试上限，请求失败（`RATE_LIMITED` 类型或消息正则匹配 rate limit、Bad credentials、账号被封）就换下一把 PAT 递归重试（`src/common/retryer.js` 9-12、49-89 行）。公共实例的额度 = 所有贡献者捐出的 PAT 总和；`api/status/pat-info.js` 还提供 PAT 存活巡检端点（限 5 分钟一次）。

### 5.3 缓存策略（src/common/cache.js）

没有服务端存储，缓存完全靠 HTTP 头驱动 CDN：每类卡片一张 TTL 表（统计卡默认 1 天、可调 12h-2d；语言卡默认 6 天、上限 10 天；钉选卡默认 10 天；错误响应仅缓存 10 分钟防止错误卡在 CDN 上），输出 `Cache-Control: max-age, s-maxage, stale-while-revalidate=1天`（`cache.js` 34-61、107-145 行）。用户可用 `cache_seconds` 参数夹取（clamp）到该卡片的 min/max 区间，`CACHE_SECONDS` 环境变量可整体覆盖——这是自部署用户绕开公共实例缓存生效慢的标准手段。

### 5.4 等级评级算法（src/calculateRank.js）

把提交、PR、Issue、Review、Star、关注者六项指标各自除以预设中位数，用指数 CDF（star/关注者用对数正态近似 CDF）压缩到 0-1，加权平均后换算成百分位与 S/A+/A/A-/B+/B/B-/C+/C 九档（`calculateRank.js` 36-84 行，star 权重最高为 4）。纯函数、无副作用，也是社区吐槽"评等级"时引用最多的文件。

## 6. 构建与运行

来自 `package.json` `scripts`（本项目无编译步骤，运行即源码）：

```bash
npm install                 # 安装依赖（Node >= 22）
npm run test                # Jest 单测 + 覆盖率（需 --experimental-vm-modules）
npm run test:e2e            # e2e（真实调用 GitHub API）
npm run bench               # Jest 基准测试
npm run lint                # ESLint --max-warnings 0
npm run format:check        # Prettier 检查
npm run theme-readme-gen    # 生成 themes/README.md 主题文档
node express.js             # 本地起 Express 服务（默认 :9000，/api 路由）
```

线上使用：README 里嵌入 `https://github-readme-stats.vercel.app/api?username=<user>` 等图片 URL；自部署则 fork 后一键部署到 Vercel 并配置 `PAT_1`（可选 `PAT_2…`、`WHITELIST`、`EXCLUDE_REPO`、`CACHE_SECONDS`，见 `src/common/envs.js`、`readme.md` "Deploy on your own" 章节）。

## 7. 测试与工程质量

- 测试规模：`tests/` 下 29 个测试文件，覆盖全部 5 个 fetcher、5 个渲染器、api 层、公共工具（retryer、cache、i18n、rank 等），另有 1 个快照文件与 `tests/e2e/e2e.test.js` 线上冒烟、`tests/bench/` 4 个基准文件（目录清单实测）。
- 风格：以 axios-mock-adapter 模拟 GitHub API 的单元测试为主，SVG 断言大量使用 `data-testid`（渲染器在 SVG 节点上预埋 `data-testid="card-bg"`、`data-testid="icon"` 等，`src/common/Card.js`、`src/cards/stats.js`），配合 @testing-library 风格查询；快照测试锁定渲染输出。
- CI（`.github/workflows/test.yml`）：单 Node 22 矩阵，跑 test+coverage（上传 Codecov）、lint（`--max-warnings 0`）、bench、prettier check；所有 action 以 commit SHA 固定版本，权限收敛为 `read-all`。
- 其他工程信号：CodeQL 分析、OSSF Scorecard、dependabot（`.github/workflows/`、`codecov.yml`、`SECURITY.md`）；代码注释质量高，JSDoc 完整，错误类型体系（`CustomError` 分类 GRAPHQL_ERROR/USER_NOT_FOUND/MAX_RETRY 等，`src/common/error.js`）明确。

## 8. 社区与维护状态

- 活跃度：gh api 显示最近 push 为 2026-10-01，open issues 297、forks 38191、watch 311；但 README 顶部横幅明确宣布"不再维护"，2026 年的 push 更可能是维护性改动（横幅、脚本）而非功能开发（`readme.md` 第 28-31 行，横幅为源文件内实证；具体 push 内容因 tarball 无 git 历史未确认）。
- 贡献者结构：contributors API 分页末页号为 318（含匿名），即约 300+ 贡献者；大量 PAT 来自社区捐赠（`retryer.js` 的 `PAT_N` 机制即为此设计）。i18n 覆盖 41 个语言（`src/translations.js` 实测），主题 75 套（`themes/index.js` 实测），均为社区众包规模的佐证。
- Release 节奏：仓库无 CHANGELOG，package.json 版本长期停在 1.0.0；部署以 Vercel 滚动 master 分支为主，无显式 release 流程（未确认更细粒度节奏）。
- 生态位：为 GitHub Profile 美化潮流的起点之一，衍生出大量 fork 与后继项目；官方指定后继为 stats-organization/github-stats-extended（README 横幅）。

## 9. 亮点与风险

亮点：

1. 极致克制的依赖面（5 个运行时依赖）+ 纯字符串 SVG 生成，在 128MB/10s 的 serverless 约束下做到亚秒渲染，`package.json`、`vercel.json` 为证。
2. `PAT_N` 轮换重试器（`src/common/retryer.js`）把"公共实例额度不够"这一死结转化为社区众筹 token 的机制设计，是小型基础设施项目的经典解法。
3. 缓存完全靠 `Cache-Control/s-maxage/stale-while-revalidate` 头外包给 CDN，零存储成本扛住千万级 README 图片请求（`src/common/cache.js`）。
4. 无 TypeScript 却通过 `@ts-check` + JSDoc 拿到近似类型安全；测试对渲染输出用 `data-testid` + 快照双保险。
5. 工程自动化完备：i18n 文档生成、主题 PR 自动关闭、CodeQL、OSSF Scorecard 全部上 CI。

风险/局限：

1. 官方已停止维护，README 挂横幅指路后继项目，新 bug（尤其 GitHub API 行为变更类）大概率无人修复。
2. 公共实例依赖社区 PAT，限流导致 Star 只统计第一页（每页 100 仓库）、频繁出现 "Maximum retries exceeded" 错误卡（`src/fetchers/stats.js` 148-155 行、`retryer.js` 32-37 行）。
3. `measureText` 是字符宽度近似值，非 ASCII 文案/长标题的排版可能溢出或错位（`src/common/render.js` 197 行起）。
4. 等级评级权重与中位数为硬编码经验值，rank 结果常被社区质疑统计意义（`src/calculateRank.js` 47-58 行）。

## 10. 总结评价

github-readme-stats 是"用最小依赖在免费 serverless 上做出全球级公共 API"的教科书案例：6900 行 JS、5 个依赖，撑起了近 8 万星和整个 GitHub Profile 美化生态。它解决的问题（README 里嵌动态统计图）至今仍是刚需，其 PAT 众筹、CDN 缓存分层、手写 SVG 排版等设计在同类项目里被反复模仿。尽管官方已宣布停维护并转向后继项目，源码仍是学习 serverless 边缘渲染、限流对抗与零依赖 SVG 生成的优质读物。适合三类人：想美化 Profile 的普通用户（直接用公共实例或后继项目）、想自部署的开发者（fork + Vercel + 自己的 PAT）、以及想研究"小代码大星数"工程范式的源码读者（重点读 `src/common/retryer.js`、`src/common/cache.js`、`src/common/Card.js` 三个文件）。
