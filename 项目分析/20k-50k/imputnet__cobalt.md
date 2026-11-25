# imputnet/cobalt 项目分析

> 星数 36600（快照 2026-10-24）｜主语言 TypeScript (Svelte)｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：cobalt 是一个"贴链接、拿文件"的社交媒体媒体下载工具，从 21 个平台提取视频/音频直链，服务端不做缓存、当"高级代理"流式转发，官方主站无广告、无追踪、无付费墙。

| 项目 | 事实 |
| :--- | :--- |
| 名称 | imputnet/cobalt（原 imput/cobalt） |
| 主页 | https://cobalt.tools |
| 星数 | 36,600（快照 2026-10-24）；分析日 gh api 实测 45,798 stars / 4,046 forks / 286 open issues |
| 语言 | TypeScript + JavaScript（前端 Svelte/SvelteKit，后端纯 Node.js ESM） |
| 许可证 | 主体与 api 为 AGPL-3.0（`LICENSE`）；web 前端为 CC-BY-NC-SA-4.0（`web/package.json` license 字段） |
| 创建时间 | 2022-07-08（gh api `created_at`） |
| 最近提交 | 2026-04-06（gh api `commits/HEAD`；tarball 内文件 mtime 均为该日） |
| 组织方式 | pnpm monorepo：`api/`（后端）+ `web/`（前端）+ `packages/*`（`pnpm-workspace.yaml`） |

## 2. 项目是什么，解决什么问题

各社交平台的视频/音频没有原生下载入口， cobalt 把"从平台拿到可保存的媒体文件"这件事标准化：用户提交一条内容链接，后端按平台解析出媒体直链（必要时借助 ffmpeg 转封装/转码），再通过带时效的 tunnel 接口把字节流交给用户（`README.md`："paste the link, get the file, move on"）。

它与同类工具（yt-dlp 等命令行全家桶）的差异化在于产品形态与立场：前端是一个极简的静态网页（SvelteKit，`web/README.md`），后端是一个干净的 JSON API（`docs/api.md` 只暴露 `POST /`、`POST /session`、`GET /`、`GET /tunnel` 四个端点），并把"无广告、无追踪、无付费墙、不缓存内容"写进 README 首段与 ethics 小节（`README.md` "ethics"：不缓存任何内容、"works like a fancy proxy"，见 `api/src/stream/`）。

官方明确伦理边界：cobalt 只处理"免费且公开可访问"的内容，不破解 DRM，不缓存，零责任声明，"不是盗版工具"（`README.md` ethics 段）。这一立场同时决定了技术形态——内容始终由用户端直取，服务器只是中转管道。

仓库本身是一个 monorepo，包含 api 后端、web 前端、共享包与四份文档（`docs/run-an-instance.md`、`docs/protect-an-instance.md`、`docs/api-env-variables.md`、`docs/api.md`），定位上鼓励用户自建实例而非依赖官方托管实例（`api/README.md`："there is currently no publicly available pre-hosted api"）。

## 3. 技术栈与依赖

**后端（`api/package.json`，`@imput/cobalt-api` v11.7.1，Node >= 18.17，ESM）**
- Web 框架：Express 4 + cors + express-rate-limit（`api/src/core/api.js`）
- HTTP 客户端：undici（支持 `HTTP_PROXY`/`FREEBIND_CIDR` 多出口 IP）
- 平台对接：`youtubei.js`（YouTube InnerTube API，`api/README.md` 明确致谢）、`hls-parser`（HLS 播放列表解析）
- 媒体处理：`ffmpeg-static`（服务端 remux/转码/提取音频）
- 校验与工具：zod（请求 schema 锁定）、url-pattern（链接模式匹配）、nanoid（tunnel 标识）、`@imput/psl`（自维护的 psl fork）
- 可选依赖：redis / rate-limit-redis（分布式限流与会话存储）、freebind（随机出口 IP）
- 沙箱：`isolated-vm`，仅在 `api/src/processing/services/youtube.js` 中用于把 YouTube 下发的 JS 挑战代码放进独立 V8 isolate 执行（5 秒超时，注释指向 LuanRT/YouTube.js#1052）

**前端（`web/package.json`，`@imput/cobalt-web` v11.7）**
- SvelteKit 2 + Svelte 5 + Vite 7 + `@sveltejs/adapter-static`（纯静态 SPA）
- 浏览器端媒体处理：`@imput/libav.js-remux-cli` / `encode-cli`（ffmpeg 的 WASM 移植），支撑"本地处理"模式（`web/src/lib/libav.ts`）
- i18n：sveltekit-i18n，内置 en、ru 两种语言（`web/i18n/`）
- 部署：`web/wrangler.jsonc`（Cloudflare Pages 静态部署）

**共享包（`packages/`）**：`api-client`（TypeScript API 客户端，v0.0.1）、`version-info`（workspace 内从 git 推导版本号的工具）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| :--- | :--- |
| `api/src/cobalt.js` | 后端入口：加载 dotenv、按 `API_INSTANCE_COUNT` 可选启用 cluster，委托 `core/api.js` |
| `api/src/core/` | Express 路由装配（`api.js`）、环境变量热重载（`env.js`）、`itunnel.js` |
| `api/src/processing/` | 核心：请求 schema（`schema.js`）、链接匹配分发（`match.js`）、21 个平台解析器（`services/*.js`）、文件名生成（`create-filename.js`）、cookie 凭据（`cookie/`） |
| `api/src/stream/` | tunnel 流管理：`stream.js` 按 proxy/internal/merge/remux/mute/audio/gif 分派，`ffmpeg.js` 服务端转封装，`manage.js` 创建/回收 tunnel |
| `api/src/security/` | `turnstile.js`（Cloudflare 人机验证）、`jwt.js`（短期 Bearer 会话）、`api-keys.js`（Api-Key 鉴权） |
| `api/src/store/` | 限流/会话存储抽象：内存实现或 Redis 实现 |
| `api/src/util/` | 测试运行器 `test.js` 与 `tests/*.json` 用例集 |
| `web/` | SvelteKit 前端：`src/lib/api/`、`src/lib/task-manager/`（浏览器端处理队列）、`src/lib/settings/`、`changelogs/`（版本说明）、i18n |
| `packages/` | `api-client`、`version-info` 两个 workspace 包 |
| `docs/` | 四份运营文档：跑实例、护实例、环境变量表、API 规范 |
| `.github/workflows/` | 7 个 workflow：codeql、docker 三套（yml/develop/staging）、test.yml、test-services.yml、fast-forward.yml |

组织评述：标准 pnpm workspace monorepo，前后端彻底解耦——前端是纯静态产物（不跑后端代码），后端只做 JSON API + 字节流 tunnel，两者仅靠 `WEB_DEFAULT_API` 构建期变量（`web/README.md`）与 `packages/api-client` 关联。目录命名全小写、代码风格紧凑，注释口语化（如 `processing/service-config.js:45-50` 解释 instagram share 链接的 pattern 冲突）。

## 5. 核心架构与关键实现

### 5.1 平台解析器体系：声明式 URL 路由 + 统一函数接口

`api/src/processing/service-config.js` 是唯一的路由表：为 21 个服务声明 `url-pattern` 模板（如 youtube 的 `watch?v=:id`、`embed/:id`，共覆盖主站/短链/子域名/替代域名），加载时统一编译成正则模式对象；同文件的 `audioIgnore`、`hlsExceptions` 集合声明各服务的能力差异。

每个解析器是 `api/src/processing/services/<name>.js` 中一个 `export default async function(obj)`，入参为匹配出的 ID + 用户参数（画质/编码/语言等），出参为统一形状 `{ urls, filename, fileMetadata }` 或 `{ error }`。最简单的 `streamable.js` 全文仅 22 行：调平台 API、挑最高码率、返回直链——新平台接入成本极低。规模分布：21 个文件共 3603 行，最大的三个是 `youtube.js`（642 行）、`instagram.js`（541 行）、`twitter.js`（370 行）。

`api/src/processing/match.js` 是总分发器：先用 `service-patterns.js` 的 testers 验证链接可处理，再 switch 到对应解析器，最后把结果交给 `match-action.js` 决定响应动作（`tunnel` 服务端转发 / `redirect` 直链重定向 / 客户端本地处理），错误统一翻译成 `error.api.*` 错误码。

```js
// api/src/processing/match.js:35-46 —— 出口 IP 随机化（freebind）
export default async function({ host, patternMatch, params, authType }) {
    const { url } = params;
    let dispatcher, requestIP;
    if (env.freebindCIDR) {
        if (!freebind) freebind = await import('freebind');
        requestIP = freebind.ip.random(env.freebindCIDR);
        dispatcher = freebind.dispatcherFromIP(requestIP, { strict: false });
    }
```

### 5.2 无缓存流转发：tunnel 与 ffmpeg 管道

解析结果不落盘：`match-action.js` 调 `stream/manage.js` 用 nanoid 生成临时 tunnel，`api/src/core/api.js` 暴露 `GET /tunnel`，由 `stream/stream.js` 按类型分派——`proxy`（纯转发，`proxy.js`）、`internal`（内部数据流）、`merge`/`remux`/`mute`/`audio`/`gif`（ffmpeg 边拉边转边吐，`ffmpeg.js`）。README 明确"cobalt never caches any content, it works like a fancy proxy"（`README.md` ethics 段），服务端零内容驻留是隐私主张的技术支撑。

```js
// api/src/stream/stream.js:11-24 —— 按流类型分派
switch (streamInfo.type) {
    case "proxy":    return await proxy(streamInfo, res);
    case "internal": return await internalStream(streamInfo.data, res);
    case "merge":
    case "remux":
    case "mute":     return await ffmpeg.remux(streamInfo, res);
    case "audio":    return await ffmpeg.convertAudio(streamInfo, res);
    case "gif":      return await ffmpeg.convertGif(streamInfo, res);
}
```

前端对称地实现了浏览器端处理：`web/src/lib/task-manager/`（queue/scheduler + `runners/fetch.ts`、`runners/ffmpeg.ts`）用 libav.js（WASM ffmpeg）在本地完成 remux/转码，请求参数 `localProcessing: preferred|forced` 可让服务器只回传原始流（`api/src/processing/schema.js`），进一步减少对服务器的信任需求；文件缓存走 OPFS（`web/src/lib/storage/opfs.ts`）。

### 5.3 安全与自托管设计

鉴权是可插拔的两层（`docs/api.md`）：实例可配置 Cloudflare Turnstile——前端过验后 `POST /session` 换短期 JWT（`api/src/core/api.js` 的 `/session` 路由调用 `security/turnstile.js` 向 Cloudflare siteverify 校验，再由 `security/jwt.js` 签发）；也可配置静态 `Api-Key`（`security/api-keys.js`）。限流用 express-rate-limit，可切换 Redis 后端实现多实例共享（`store/redis-ratelimit.js`），所有端点返回标准 `RateLimit-*` 头（`docs/api.md`）。

自托管是官方主推路径：`docs/run-an-instance.md` 给出 docker compose + watchtower 自动更新方案，`docs/protect-an-instance.md` 教配 Turnstile/API-Key 防滥用，`docs/api-env-variables.md` 是完整环境变量表（仅 `API_URL` 必填，其余含 `DURATION_LIMIT`、`FREEBIND_CIDR`、`DISABLED_SERVICES` 等）。`Dockerfile` 为 node:24-alpine 多阶段构建，`pnpm deploy --filter=@imput/cobalt-api` 只部署 api 包，非 root 用户运行，暴露 9000 端口。需要登录态内容的平台用 `cookies.json` 挂载（`docs/run-an-instance.md`）。

## 6. 构建与运行

```bash
# 后端（docs/run-an-instance.md）
cd api && pnpm install
echo "API_URL=http://localhost:9000/" > .env
pnpm start            # 即 node src/cobalt（api/package.json scripts）

# 测试（api/package.json scripts.test）
pnpm test             # node src/util/test，跑全部服务用例
TEST_IGNORE_SERVICES=youtube,twitter pnpm test   # 排除易抖动服务

# 前端（web/README.md）
cd web && pnpm install
WEB_DEFAULT_API=http://localhost:9000/ pnpm run build   # vite build，静态产物
pnpm run check        # svelte-check 类型检查

# Docker（Dockerfile + docs/run-an-instance.md）
docker compose up -d  # 官方镜像 + watchtower 自动更新
```

## 7. 测试与工程质量

- 测试形态特殊：`api/src/util/tests/` 下每个服务一份 JSON 用例（共 21 份），每条用例是 `{name, url, params, expected:{code, status}}`，由 `api/src/util/test.js` 驱动**真实调用线上平台**做端到端断言，属于"活体集成测试"而非单元测试。
- 工程上承认第三方抖动：`test.js` 内置 finnicky 服务名单（bilibili、instagram、facebook、youtube、vk、twitter、reddit 等）软失败不阻塞，CI 中以 `::warning` 注解。
- CI（`.github/workflows/`）：`test-services.yml` 把 21 个服务铺成 GitHub Actions matrix 并行跑真实请求；另有 `test.yml`、CodeQL 静态安全扫描、三套 docker 镜像构建（stable/develop/staging）、`fast-forward.yml`。
- 前端质量工具：eslint 9 + typescript-eslint + svelte-check（`web/package.json` devDependencies）；后端为无类型检查的纯 JS，依赖约定与 zod 边界校验保证质量。

## 8. 社区与维护状态

- 体量：gh api 实测 45,798 stars、4,046 forks、286 open issues（2026-11-05）；contributors 至少 44 人（`/contributors` 分页 Link 头 page=45）。
- 节奏：仓库创建于 2022-07-08；版本号已演进到 api v11.7.1 / web v11.7（各自 package.json），web 侧 `changelogs/` 保留 7.x→10.5 的历次更新说明；不使用 GitHub Releases，分发走 docker 镜像 tag。
- 活跃度：默认分支 main 的 HEAD 提交日期为 2026-04-06（gh api `commits/HEAD`，tarball 文件 mtime 一致），即截至分析日约 7 个月无新提交，近期维护状态未确认。
- 贡献结构：核心开发由 imput 团队主导，基础设施由 royalehosting.net 赞助（`README.md` thank you 段）；社区通过 Discord 与 issue/PR 参与（`CONTRIBUTING.md`）。

## 9. 亮点与风险

**亮点**
1. 解析器体系设计干净：`service-config.js` 一张声明式 pattern 表 + 每平台一个统一签名的 async 函数，最小实现 22 行（`services/streamable.js`），可扩展性极强。
2. 隐私架构自洽：服务端零缓存、内容即时流式转发（`stream/`），前端还提供浏览器端 libav WASM 本地处理（`web/src/lib/task-manager/`），把"处理用户数据的服务器组件"压到最小；仅可选接入自托管 plausible 统计。
3. 安全工程完整：zod 全量请求校验、Turnstile 人机验证 + 短期 JWT、Api-Key 双方案、标准 RateLimit 头、用 isolated-vm 沙箱执行不可信的平台 JS（`services/youtube.js:12-21`）。
4. 自托管体验好：文档四件套 + 单容器多阶段 Dockerfile + 环境变量热重载 + 可选 Redis 集群化，"跑自己的 cobalt"是官方推荐用法。
5. 出口 IP 池化（freebind）与代理环境变量，体现对"被平台限流"这一现实问题的工程应对（`processing/match.js`）。

**风险**
1. 平台对抗是持续性成本：解析器依赖各平台私有接口与反爬现状，twitter 在 `api/README.md:59` 自述"may not be 100% reliable due to current management"，21 个服务的可用性随时可能被动失效。
2. 法律与平台条款：抓取公开媒体可能违反各平台服务条款；官方以 ethics 段做"零责任、非盗版工具"的立场声明（`README.md`），使用者与自托管者仍需自行承担合规责任。
3. 许可证组合有约束：api 为 AGPL-3.0（衍生自托管需开源），web 为 CC-BY-NC-SA-4.0（禁止商用），二次开发前需注意。
4. 维护活跃度存疑：main 分支自 2026-04 后截至分析日无新提交，且官方托管实例明确不开放 API 给第三方项目（`docs/api.md`），依赖方需评估项目走向。

## 10. 总结评价

cobalt 是"媒体下载"这个拥挤赛道里工程与产品完成度都突出的作品：后端用一张声明式路由表加 21 个统一接口的解析器覆盖主流平台，以零缓存流转发和无广告无追踪的立场建立了鲜明的产品人格；安全（Turnstile/JWT/Api-Key/沙箱）、运维（Docker/文档/限流）与前端体验（WASM 本地处理、i18n）三面都做得很完整，是学习"如何把一个爬取型工具做成可信服务"的优质范本。适合自托管用户拿来即用、想接入平台解析能力的开发者读 `api/src/processing/` 目录、以及任何在设计"第三方数据中转服务"时寻找隐私架构参考的工程师研读；同时应清醒看待其平台对抗风险与当前提交放缓的状态。
