# DIYgod/RSSHub 项目分析

> 星数 36500（快照 2026-10-24）｜主语言 JavaScript (TypeScript)｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：全球最大的自建 RSS 生成网络——"万物皆可 RSS"，通过 2000 个站点命名空间、近 7000 个社区路由把任意网站/应用/接口的内容标准化输出为 RSS/Atom/JSON Feed。

| 项 | 值 |
| --- | --- |
| 名称 | DIYgod/RSSHub |
| 星数 | 36500（快照） |
| 语言 | TypeScript（ESM，Node 24/26，`package.json`） |
| 许可证 | AGPL-3.0（`LICENSE`、`package.json`） |
| 创建时间 | 2018-04-02（gh api repos/DIYgod/RSSHub） |
| 最近 push | 2026-11-04（分析前一日，极活跃） |
| 发布节奏 | 每次提交自动发版，如 v2026.11.04-8573.8f4fd43（gh api releases） |
| 贡献者 | 约 1746（gh api contributors 分页推算） |
| 默认分支 | master |

## 2. 项目是什么，解决什么问题

RSSHub 解决的核心问题是：绝大多数网站不再提供 RSS 订阅源。它把"为某个站点写一个抓取器"这件事标准化成路由（route）——每个路由声明 URL 模式、参数、抓取逻辑，把目标站内容转成统一的 `Data` 结构，再由框架序列化为 RSS 2.0/Atom/JSON Feed 输出（README、`lib/types.ts`）。用户只需自建或使用公共实例，即可订阅微博、B 站、GitHub、播客、政务公告等一切内容。

它是典型的"社区路由式"项目：本体代码约 40 万行，其中 `lib/routes/` 占 40.8 万行、6864 个 TS 文件，绝大部分由社区按"一站一目录"贡献（`lib/routes/` 统计）。新路由的门槛被压到极低：写一个导出 `route` 对象的 TS 文件即可，`scripts/generate-route.mjs` 还提供脚手架（`pnpm new:route`）。

RADAR 规则是它的差异化设计：每个路由可声明 `radar: [{ source: ['github.com/:user/:repo/issues'], target: '/issue/:user/:repo' }]`（`lib/routes/github/issue.ts`），构建时 `scripts/workflow/build-routes.ts` 用 tldts 按主域/子域聚合生成规则表，经 `/api/radar/rules/{domain}` 下发（`lib/api/radar/rules/one.ts`）。浏览器扩展 RSSHub Radar（及其移动端衍生 RSSBud/RSSAid）据此在用户浏览当前网页时自动提示"此页可订阅的 RSSHub 路由"，形成"路由库 ↔ 发现端"双向生态（README Related Projects、`lib/types.ts` RadarItem 注释）。

项目同时是一套可运营的基础设施：缓存、限速、反爬、鉴权、热链保护等生产问题都在框架层统一解决，路由作者只需关心解析。官方文档称全网 5000+ 实例（README），是自托管 RSS 生态的事实标准。

## 3. 技术栈与依赖

- 语言/运行时：TypeScript + Node ESM（`"type": "module"`，Node ^24.15 || ^26.13，`package.json`）；同时可编译到 Cloudflare Workers（`wrangler.toml`、`lib/worker.ts`）。
- Web 框架：Hono（`lib/app.ts` import 自 'hono'），Node 侧用 `@hono/node-server`；开放 API 用 `@hono/zod-openapi` + Scalar 文档（`lib/api/index.ts`）。这是从早期 Koa 迁移的结果——缓存键前缀仍保留历史名 `rsshub:koa-redis-cache:`（`lib/middleware/cache.ts:30`）。
- HTTP 抓取：undici/ofetch 为主（`lib/utils/ofetch.ts`、`package.json` 依赖 undici 8.12.3）；`lib/utils/got.ts` 现在是 ofetch 之上的 got 兼容垫片（fakeGot 支持 cookieJar、hooks、buffer responseType），让数千个旧路由无需改动即运行在新栈上。
- 解析：cheerio 1.2、jsdom、html-to-text、rss-parser（带 patch，`patches/rss-parser@3.13.0.patch`）、markdown-it。
- 浏览器自动化：patchright（Playwright 反检测分支，`lib/utils/playwright.ts`），支持远程浏览器 WS/CDP 端点（`config.ts` PLAYWRIGHT_WS_ENDPOINT）；保留 `puppeteer-utils.ts` 兼容旧路由。
- 缓存/限流：ioredis、lru-cache、rate-limiter-flexible（按域名限速，`lib/utils/host-rate-limit.ts`）。
- 可观测性：OpenTelemetry（Prometheus/OTLP exporter）、Sentry、Honeybadger、winston（`package.json`、`lib/middleware/`）。
- 构建：tsdown（多配置对应 Docker/Worker/Vercel/lib 四种产物）、rolldown、tsx 开发直跑；质量工具 oxlint+oxfmt+eslint、vitest。

## 4. 目录结构与代码组织

顶层为单包仓库（pnpm-workspace.yaml 只有依赖策略无 packages 字段，非 monorepo）：

| 目录 | 职责 |
| --- | --- |
| `lib/` | 全部核心代码：框架 + 2000 个路由目录 |
| `lib/routes/` | 路由集合：1998 个站点目录、6864 个路由 TS 文件、1986 个 namespace.ts（约 40.8 万行） |
| `lib/middleware/` | cache/parameter/access-control/anti-hotlink/header/trace 等 Hono 中间件 |
| `lib/utils/` | got 垫片、ofetch、cache 后端、playwright、parse-date、host-rate-limit、request-rewriter 等 |
| `lib/views/` | 输出模板：rss.tsx/atom.tsx/json.ts/rss3.ts（hono/jsx 组件） |
| `lib/api/` | OpenAPI 子应用：namespace/category/radar rules/route status/follow config |
| `lib/magpie/` | 作为 npm 包/magpie 插件嵌入运行的支持代码 |
| `scripts/workflow/` | build-routes（生成路由产物与 radar 规则）、build-docs、孤儿文件检查等 CI 脚本 |
| `eslint-plugins/` | 自定义 lint 插件：anti-slop、nsfw-flag（约束路由 PR 质量） |
| `patches/` | pnpm patch（rss-parser） |
| 根配置 | wrangler*.toml / vercel.json / fly.toml / railway.json / Dockerfile / docker-compose.yml（多平台部署） |

组织方式评述：单体仓库 + "声明式微模块"。每个路由目录含 `namespace.ts`（站点元数据）+ 若干路由文件，模块通过 `lib/registry-dev.ts` 的目录扫描与 `lib/registry-helpers.ts` 的 `applyModulesToNamespaces` 动态聚合，生产环境则由 `build:routes` 预编译成单一 `assets/build/routes.js` 产物（`lib/app.ts` 按 NODE_ENV 分流加载）。

## 5. 核心架构与关键实现

### 5.1 路由注册体系（`lib/registry-helpers.ts`）

`collectNamespaceRoots` 把含 `namespace.ts` 的目录识别为命名空间根；`resolveModuleNamespace` 让深层模块归属最长匹配根；`applyModulesToNamespaces` 将 route/apiRoute 归并进 `namespaces` 表。注册顺序有讲究：命名空间按深度降序注册（防止父级参数路由遮蔽子命名空间），同命名空间内按"字面段 > 正则参数 > 普通参数"的段秩排序，确保 `/issue/:user/:repo` 不被 `/:link{.+}` 之类通配路由抢先（`sortRoutes` 及其注释）。

```ts
// lib/registry-helpers.ts:172-204（节选）
for (const namespace of sortNamespacesByDepth(namespaces)) {
    const subApp = app.basePath(`/${namespace}`);
    const sortedRoutes = sortRoutes(namespaceData.routes);
    for (const [path, routeData] of sortedRoutes) {
        const wrappedHandler: Handler = async (ctx) => {
            if (!routeData.handler) {   // 首次命中才动态 import 路由模块（懒加载）
                const { route } = await routeData.module();
                routeData.handler = route.handler;
            }
            const response = await routeData.handler(ctx);
            ...
            ctx.set('data', response);
        };
        subApp.get(path, wrappedHandler);
    }
}
```

### 5.2 数据流：请求 → 抓取 → 解析 → 输出

一条请求经过中间件链（`lib/app.ts` 注册路由后由 bootstrap 挂载 `lib/app-bootstrap.tsx`）：access-control（ACCESS_KEY 鉴权）→ cache（见 5.3）→ parameter（处理 limit/filter/filter_title/hotlink/format 等用户参数，`lib/middleware/parameter.ts`）→ 路由 handler。路由内用 `got`/`ofetch` 抓取（自动带 `header-generator` 生成的水合 UA、代理、按域限速），cheerio 解析后返回 `Data { title, link, item: [...] }`，再由 `lib/views/rss.tsx`（hono/jsx 模板，自动按需挂 itunes/media/dc 命名空间）或 atom/json/rss3 视图序列化。

### 5.3 缓存与并发去重（`lib/middleware/cache.ts`）

缓存键为 `rsshub:koa-redis-cache:` + XXH64(路径+format+limit+sorted+filtered)。亮点是"原子认领"防惊群：仅 Redis 等支持原子操作的后端开启，用一段 Lua 脚本 `SET key 1 EX` 抢占，并发请求要么等待轮询缓存要么抛 RequestInProgressError，绝不重复抓源（`lib/utils/cache/index.ts` 的 claim 实现）。后端可插拔：memory/redis/http/KV，Worker 环境退化为 no-op（`lib/utils/cache/index.ts:44-47`）。路由内还有 `cache.tryGet()` 细粒度缓存供循环抓详情使用（AGENTS.md 第 17 条强制要求）。

### 5.4 反爬与稳定性设施

- 浏览器渲染：patchright（Playwright 反检测分支）+ 远程浏览器池端点配置，`lib/utils/playwright.ts` 内置代理协议适配（HTTP/SOCKS/PAC）；Worker 环境走 `@cloudflare/playwright`（`lib/utils/playwright-remote.worker.ts`、`browser-session.worker.ts`）。
- 按域名限速：`REQUEST_RATE_LIMITS` JSON 配置驱动，`lib/utils/host-rate-limit.ts` 实现每 host 排队节流。
- 全局抓取可观测：`lib/utils/request-rewriter/index.ts` monkey-patch 全局 fetch 与 http/https，统一接入追踪。
- 配置体系：`lib/config.ts` 约 90 个环境变量，覆盖网络/代理/缓存/访问控制/日志/路由开关（如 DISABLE_NSFW 过滤 nsfw 路由，`lib/app.ts` safeNamespaces）。

### 5.5 RADAR 与生态联动

构建期 `scripts/workflow/build-routes.ts` 遍历所有路由的 `radar` 字段，按 tldts 解析的 domain/subdomain 聚合成规则树并写入构建产物；运行期 `lib/api/radar/rules/{all,one}.ts` 提供全量与按域查询两个 OpenAPI 端点。RSSHub Radar 扩展、RSSBud（iOS）、RSSAid（Android）加载该规则表，在用户浏览页面时反向匹配出可订阅路由（`lib/types.ts` RadarItem 文档注释明确引用 RSSHub-Radar 仓库的兼容性 commit）。

## 6. 构建与运行

来自 `package.json` scripts：

```bash
pnpm i                     # pnpm 12，Node ^24.15 || ^26.13
pnpm dev                   # 开发模式 tsx watch lib/index.ts
pnpm build                 # build:routes（生成 assets/build/routes.js）+ tsdown 产物
pnpm start                 # NODE_ENV=production node dist/index.mjs
pnpm test                  # format:check + vitest --coverage
pnpm vitest:fullroutes     # 全路由回归测试（lib/app.test.ts，FULL_ROUTES_TEST）
pnpm worker-build / worker-deploy   # Cloudflare Workers 构建/部署
pnpm build:lib             # npm 包构建（发布为 rsshub，入口 dist-lib）
```

Docker：`docker-compose.yml` + `Dockerfile`；也支持 Vercel（`vercel.json`、tsdown-vercel 配置）、Fly.io（`fly.toml`）、Railway（`railway.json`）一键部署。

## 7. 测试与工程质量

- 测试框架 vitest（根级与 `vitest.workers.config.ts` 双套），`lib/` 下 87 个 `*.test.ts(x)`；测试既有单元（如 `lib/utils/cache/index.test.ts`、`lib/middleware/cache.test.ts`）也有整站级 `lib/app.test.ts` 全路由测试，CI 服务容器内起 Redis（`.github/workflows/test.yml` services）。路由级真实抓取测试由 `test-full-routes.yml` 定期跑。
- CI 共 20 个 workflow：test/lint/format/codeql/semgrep/docker-release/npm-publish/issue-command（评论 `/test route` 触发路由测试）等（`.github/workflows/`）。
- 质量约束面向海量社区 PR：`AGENTS.md` 21 条审查规则（路由命名、radar 格式、必须用 tryGet 缓存、禁止伪造 pubDate 等），并自研 eslint 插件 `eslint-plugins/anti-slop/`、`nsfw-flag.js` 机审；`scripts/workflow/check-orphan-files.ts` 构建期拒绝游离文件。
- 观察：格式化用 oxlint+oxfmt（而非 prettier），TypeScript 7 试运行（devDependencies 里 typescript-7 别名），工具链相当新潮。

## 8. 社区与维护状态

- 极活跃：gh api 显示最近 push 为 2026-11-04（分析前一日），当天已有 3 个自动 release（v2026.11.04-8571/8572/8573，编号即累计发版数）。
- 贡献者约 1746 人（contributors 分页 last=1747），issue/PR 流程高度自动化（dependabot-fork、similar-issues、stale、pr-review 等 workflow）。
- 商业/生态支撑：README 有赞助商位，作者 DIYgod 的姊妹项目 Folo（AI RSS 阅读器）与 RSSHub 互推；`lib/api/follow/config.ts` 表明与 Folo 服务端有配置对接。
- Docker/npm 持续发布（README 徽章），无传统语义化版本，采用日期+序号+短哈希。

## 9. 亮点与风险

亮点：
1. 声明式路由 + 零门槛贡献模型：namespace.ts/route/radar 三件套让 1700+ 贡献者可持续供给内容源，是开源社区自增殖的典范（`lib/routes/`、`lib/registry-helpers.ts`）。
2. 框架层替路由扛住生产难题：缓存惊群防护（Redis 原子认领 Lua）、按域限速、UA 伪装、patchright 反检测、代理 PAC，路由作者完全不感知（`lib/middleware/cache.ts`、`lib/utils/host-rate-limit.ts`、`lib/utils/playwright.ts`）。
3. got→ofetch 无痛迁移：以 90 行兼容垫片保住 6000+ 旧路由在 undici 新栈上运行，工程迁移范本（`lib/utils/got.ts`）。
4. RADAR 双向生态：路由即发现规则，浏览器扩展反向消费，形成数据网络效应（`scripts/workflow/build-routes.ts`、`lib/api/radar/`）。
5. 多运行时一套代码：Node/Docker/Vercel/Cloudflare Workers 四目标产物（tsdown 多配置、`lib/worker.ts`、is-worker 探测）。

风险/局限：
1. 上游站改版即路由批量失效，长期维护成本高，依赖不断有人认领修复（`maintainers` 字段机制部分缓解）。
2. 反爬对抗持续升级（登录墙、风控），部分路由需 cookie/token 配置，公共实例可用性波动。
3. AGPL-3.0 + 项目实质成为聚合抓取基础设施，商用与内容版权层面存在合规争议空间。
4. 单路由文件质量参差（社区产出），需靠 lint 插件与 AI 审查规则兜底，代码一致性无法完全保证。

## 10. 总结评价

RSSHub 是"把 RSS 还给互联网"的十余年长跑项目：以极低贡献门槛的声明式路由体系聚集了全球最大的自建订阅源网络，同时在框架层沉淀出缓存、限速、反爬、多运行时部署等一整套抓取基础设施工程。对想要自建 RSS 服务的人它是直接可用的开箱工具；对做爬虫/内容聚合平台的人，`lib/registry-helpers.ts`、`lib/middleware/cache.ts`、`lib/utils/got.ts` 是三份值得精读的参考实现——前者展示了如何组织上万模块的动态注册与路由消歧，后两者展示了大规模抓取系统的并发去重与依赖迁移技巧。唯一要有的心理预期是：这是一个"活的"代码库，路由库随互联网漂移而持续变动，读它更像是观察一个生态而非阅读一份静态代码。
