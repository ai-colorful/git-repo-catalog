# CorentinTh/it-tools 项目分析

> 星数 41500（快照 2026-10-24）｜主语言 Vue｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：一个完全在浏览器本地运行、无任何后端的开发者在线工具箱，86 个小工具覆盖加密、转换、网络、文本等高频场景，以"极简自托管 + 好用 UX"著称。

| 项 | 值 |
| --- | --- |
| 名称 | CorentinTh/it-tools |
| 星数 | 41500（快照） |
| 语言 | Vue 3 + TypeScript |
| 许可证 | GPL-3.0（`LICENSE`、`package.json`） |
| 创建时间 | 2020-04-05（gh api） |
| 最近 push | 2026-11-12（gh api）；main 最近提交 2026-02-12 |
| 最近 release | v2024.10.22（2024-10-22，gh api） |
| 工具总数 | 86 个工具、10 个分类（`src/tools/index.ts`） |
| 贡献者 | 44 人，单维护者主导（gh api contributors） |
| 线上站点 | https://it-tools.tech（`README.md`） |

## 2. 项目是什么，解决什么问题

开发者日常会遇到大量"一次性小需求"：解码一个 JWT、算一个 IPv4 子网、生成一个强随机 token、把 docker run 命令转成 compose、比对两段文本差异等。为此装 CLI 或写脚本太重，用各种广告缠身的在线工具站又不放心（敏感数据贴进第三方网页有隐私风险）。it-tools 把这类需求收敛成一个纯前端单页应用：所有计算都在浏览器本地完成，没有账号、没有上传、没有服务端，敏感数据不出本机（`Dockerfile` 生产镜像仅为 nginx 静态托管，`src/config.ts` 中唯一的"联网"能力是可选关闭的 Plausible 匿名统计）。

项目由法国开发者 Corentin Thomasset（CorentinTh）个人发起并长期维护（`package.json` author、`README.md` Credits）。2020 年创建后以"每合并一个 PR 就多一个小工具"的方式有机生长，当前收录 86 个工具，按 Crypto、Converter、Web、Images and videos、Development、Network、Math、Measurement、Text、Data 十个分类组织（`src/tools/index.ts` 中 `toolsByCategory` 数组）。

架构上它是一个"工具注册表"型应用：每个工具是 `src/tools/` 下的一个自包含目录，通过 `defineTool()` 声明元信息（路径、关键词、图标、懒加载组件），在 `src/tools/index.ts` 中归入分类；路由、首页卡片、Ctrl+K 命令面板、i18n 标题全部从这张注册表自动派生，新增一个工具不需要碰任何框架代码（`src/router.ts`、`src/tools/tools.store.ts`）。

部署形态主打自托管 homelab：一条 `docker run -d -p 8080:80 corentinth/it-tools` 即可跑起来（`README.md` Self host），镜像同时发布到 Docker Hub 与 GHCR，并另有每日 nightly 镜像（`.github/workflows/docker-nightly-release.yml`）。此外官方站由 Vercel 持续部署（`README.md`、`vercel.json`），也提供 Netlify 配置（`netlify.toml`）。

## 3. 技术栈与依赖

| 层 | 选型 | 出处 |
| --- | --- | --- |
| UI 框架 | Vue 3.3（`<script setup>` 组合式 API） | `package.json` |
| UI 组件库 | Naive UI 2.35（`naive.plugin.ts` 全量注册） | `package.json`、`src/plugins/naive.plugin.ts` |
| 原子 CSS | UnoCSS 0.65 | `unocss.config.ts`、`package.json` |
| 路由 | vue-router 4（history 模式，base 可配） | `src/router.ts` |
| 状态 | Pinia 2 + @vueuse/core `useStorage` 持久化 | `src/tools/tools.store.ts` |
| i18n | vue-i18n 9 + @intlify/unplugin-vue-i18n，YAML 语言包 | `locales/*.yml`（9 语言） |
| 构建 | Vite 4 + vue-tsc 类型检查 + PWA（vite-plugin-pwa） | `package.json` scripts、`vite.config.ts` |
| 测试 | Vitest（jsdom，单元）+ Playwright（e2e） | `package.json`、`playwright.config.ts` |
| Lint | ESLint + @antfu/eslint-config | `package.json` devDependencies |
| 配置 | figue（声明式 env 配置校验） | `src/config.ts` |

业务依赖约 70 个，全部是浏览器端库，对应各工具的领域能力：`crypto-js`/`node-forge`（加密、RSA）、`jwt-decode`、`bcryptjs`、`libphonenumber-js`（电话解析）、`mathjs`（数学求值）、`monaco-editor`（代码编辑器）、`sql-formatter`、`xml-js`/`xml-formatter`、`yaml`、`cronstrue`、`figlet`（ASCII 艺术）、`qrcode`、`pdf-signature-reader`、`composerize-ts`（docker run → compose）等（`package.json` dependencies）。作者自己的两个包 `@it-tools/bip39`、`@it-tools/oggen` 也在依赖中。无任何服务端运行时依赖——整个应用是一个纯静态 SPA。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `src/tools/` | 核心：86 个工具目录 + 注册表基础设施（`index.ts`、`tool.ts`、`tools.types.ts`、`tools.store.ts`） |
| `src/ui/` | 自研基础组件库（c-input-text 等，附 `demo.routes` 仅 dev 环境挂载） |
| `src/modules/` | 应用级模块：command-palette（Ctrl+K 搜索）、i18n、tracker（Plausible）、shared |
| `src/plugins/` | naive、i18n、plausible 等 Vue 插件 |
| `src/composable/`、`src/utils/` | 通用组合式函数与工具函数（copy、fuzzySearch、validation、random、base64 等） |
| `src/layouts/` | base / tool 两种布局 |
| `src/pages/` | 仅 3 个页面：Home、About、404（工具页由注册表动态生成路由） |
| `locales/` | 9 语言全局 YAML 包（de/en/es/fr/no/pt/uk/vi/zh） |
| `scripts/` | create-tool 脚手架、build-locales-files、release、getLatestChangelog |
| `.github/` | 4 个 workflow + issue/PR 模板 + stale 机器人配置 |
| `_templates/` | hygen 模板（UI 组件生成） |

组织方式评述：单包 SPA，不是 monorepo；代码组织围绕"工具 = 目录"的约定展开。每个工具目录内部遵循固定四件套：`index.ts`（defineTool 注册）、`*.tool.vue`（界面）、`*.service.ts`（纯逻辑，可单测）、`*.test.ts`/`*.e2e.spec.ts`（测试）。UI 与逻辑分离是全仓库的一致约定（例：`src/tools/token-generator/` 下 5 个文件职责清晰）。

## 5. 核心架构与关键实现

### 5.1 工具注册表与 defineTool（最核心的设计）

`src/tools/tool.ts` 全文仅 13 行：`defineTool()` 接收工具元数据，自动根据 `createdAt` 计算"两周内新建"的 `isNew` 标记，首页据此展示"最新工具"角标：

```ts
export function defineTool(tool: WithOptional<Tool, 'isNew'>) {
  const isNew = tool.createdAt ? isAfter(tool.createdAt, subWeeks(new Date(), 2)) : false;
  return { isNew, ...tool };
}
```

`Tool` 接口（`src/tools/tools.types.ts`）只有 8 个字段：name、path、description、keywords、component（`() => Promise<Component>` 懒加载）、icon、redirectFrom（旧路径重定向）、createdAt。所有工具以 `component: () => import('./xxx.vue')` 声明，天然按工具代码分割，首页只加载注册表元数据。

### 5.2 注册表驱动的路由与首页

`src/router.ts` 直接从注册表派生全部路由：`tools.map()` 生成工具页（挂在 toolLayout 上）、`redirectFrom` 展开成重定向、仅 dev 环境追加 UI 组件 demo 路由。这意味着新增工具只写一个目录 + 在 `index.ts` 里 import 一行（脚手架 `scripts/create-tool.mjs` 连这两步都替你做了），路由、菜单、搜索、i18n key 全部自动生效。

### 5.3 i18n 与收藏的键值约定

`src/tools/tools.store.ts` 用工具 path 反推 i18n key（`tool.path.replace(/\//g,'')` → `tools.uuid-generator.title`），语言包缺失时回退到工具源码里的英文默认值；收藏夹用 @vueuse 的 `useStorage('favoriteToolsName')` 持久化到 localStorage，支持拖拽排序（`updateFavoriteTools`，CHANGELOG 中"favorites: store favorites regardless of languages"正是修这个键的稳定性）。全局 `locales/` 共 9 个 YAML（en/no/de 各约 390-456 行、zh 390 行、vi 383 行，es/pt/uk 尚不完整仅 72 行），覆盖首页、分类名与各工具标题/描述。

### 5.4 本地优先的运行时配置与搜索

`src/config.ts` 用 figue 把应用版本、baseUrl、Plausible 统计开关等收敛为"声明式配置 + env 注入"，且 `isTrackerEnabled` 默认 false——自托管时默认零外呼，这是"数据不出浏览器"承诺的关键落点。全局搜索走 Ctrl+K 命令面板（`src/modules/command-palette/command-palette.vue` 用 `useMagicKeys` 监听，fuse.js 模糊匹配 `src/composable/fuzzySearch.ts`）。

## 6. 构建与运行

来自 `README.md` 与 `package.json` scripts：

```sh
pnpm install            # 安装（packageManager: pnpm@9.11.0）
pnpm dev                # 开发服务器
pnpm build              # vue-tsc --noEmit + vite build（需 NODE_OPTIONS=--max_old_space_size=4096）
pnpm test               # Vitest 单元测试（jsdom）
pnpm test:e2e           # Playwright e2e（playwright.config.ts）
pnpm lint               # ESLint
pnpm typecheck          # vue-tsc -p tsconfig.vitest.json
pnpm run script:create:tool my-tool-name   # 生成新工具脚手架
pnpm release            # scripts/release.mjs（standard-version 式发版）
```

自托管（`README.md`、`Dockerfile`、`nginx.conf`）：

```sh
docker run -d --name it-tools --restart unless-stopped -p 8080:80 corentinth/it-tools:latest
# 或 ghcr.io/corentinth/it-tools:latest
```

镜像为两阶段构建：`node:lts-alpine` 执行 pnpm build → 产物拷入 `nginx:stable-alpine`，仅暴露 80 端口，`try_files $uri /index.html` 支持 SPA 路由，无任何动态服务。

## 7. 测试与工程质量

- 单元测试 33 个文件（Vitest，jsdom）：集中在各工具的 `*.service.test.ts` 纯逻辑层与 `src/utils/`、`src/composable/`（如 `token-generator.service.test.ts` 98 行、`ipv4-range-expander.service.test.ts`），以及 2 个 UI 组件测试（`src/ui/c-input-text/c-input-text.test.ts`）。
- e2e 测试 26 个 `*.e2e.spec.ts`（Playwright）：覆盖 26 个工具的关键交互（进入页面断言标题、点击按钮断言输出变化，见 `src/tools/token-generator/token-generator.e2e.spec.ts`），CI 中按 3 个 shard 并行（`.github/workflows/e2e-tests.yml`）。
- CI 双流水线：`ci.yml`（PR/push main → lint + 单测 + typecheck + build）、`e2e-tests.yml`（build 后跑 Playwright，缓存浏览器）。
- 发布流水线：`releases.yml`（tag 触发，构建 linux/amd64+arm64 双架构镜像推 Docker Hub + GHCR 并发 GH release）；`docker-nightly-release.yml` 每日检查 24h 内是否有新提交，有则构建 `nightly` 标签镜像（先跑 lint+test+build）。
- 质量观察：ESLint 用 @antfu/eslint-config 全量严格风格；renovate.json 自动化依赖升级（贡献榜第二名是 renovate[bot]，55 次贡献）；代码分层约定（vue 薄、service 厚）执行得相当一致，service 均为无副作用纯函数，方便测试与复用。

## 8. 社区与维护状态

- 单维护者模式（gh api contributors 核实）：CorentinTh 476 次提交，第二名人类贡献者 sharevb 仅 11 次，44 位贡献者总量。README 署名"Coded with love by Corentin Thomasset"。这是一个典型的"个人作品爆红"项目。
- 发版节奏：CalVer 命名（`2024.10.22-7ca5933`，日期 + commit SHA），CHANGELOG 14 个版本（`CHANGELOG.md`），gh api 显示 2023 年密集发版（6 个）、2024 年 2 个，最后 release 停在 2024-10-22。
- 活跃度拐点明显：main 分支最近提交为 2026-02-12（chore(readme)），2025 年仅 4 次提交且多为赞助横幅与小修（gh api commits）。gh api 显示 repo pushed_at 2026-11-12（可能来自其他分支/元数据更新）。865 个 open issue 长期未消化（README 明确把 issues 当 roadmap 用）。
- 功能/新工具主要通过社区 PR + issue feature request 流程；`.github/stale.yml` 配置了 stale 机器人清理 issue；FUNDING.yml 与赞助横幅是作者的持续收入方式。

## 9. 亮点与风险

**亮点**

1. 注册表 + 脚手架的"低摩擦插件架构"：新增工具成本极低（一个目录 + 一行 import），路由/首页/搜索/i18n 全自动派生（`src/tools/index.ts`、`src/router.ts`、`scripts/create-tool.mjs`），这是它能靠社区 PR 滚到 86 个工具的关键。
2. 本地优先隐私模型：纯静态 SPA、无后端、统计默认关闭（`src/config.ts`），天然适合放内网处理敏感数据。
3. 极简自托管：单 nginx-alpine 镜像、双架构、Docker Hub + GHCR 双发布 + 每日 nightly（`Dockerfile`、`.github/workflows/`），homelab 人群极易 adoption。
4. 一致的分层与双层测试约定：`.tool.vue` 薄 UI + `.service.ts` 纯逻辑 + Vitest/Playwright，33 个单测 + 26 个 e2e，工程规范在小工具仓库里属于上乘。
5. UX 打磨：Ctrl+K 命令面板、可拖拽收藏夹、PWA、暗色主题、9 语言 i18n、"两周内新工具"角标（`src/modules/command-palette/`、`src/tools/tools.store.ts`、`src/tools/tool.ts`）。

**风险/局限**

1. 单维护者 + 明显停滞：最后 release 已停在 2024-10，2025 年以来提交量骤减，865 个 open issue 积压，长期存在"弃坑"风险（gh api 数据）。
2. 构建资源偏重：全部工具打进一个 bundle，引入 monaco-editor、mathjs 等重库，build 需手动调大到 4GB heap（`package.json` build 脚本），首屏体积对低端设备不友好。
3. 工具深度有限：多数工具是单文件薄封装（如 `token-generator.service.ts` 仅 26 行），复杂场景（如真正的安全审计）不能替代专业工具。
4. i18n 完整度不均：9 语言中 es/pt/uk 仅覆盖约 72/394 个 key（`locales/*.yml` 行数对比），非英文用户部分界面仍会回退英文。

## 10. 总结评价

it-tools 是"个人开发者 + 纯前端 + 注册表架构"三者相乘的典范项目：它解决的需求真实高频，隐私与自托管卖点踩中了 homelab/安全敏感人群，而极低的新工具贡献成本让它像滚雪球一样长出 86 个工具并冲到 4 万星；代价则是所有重量都压在一个人肩上，2024 年末之后项目实质进入低维护状态。代码本身是学习 Vue 3 中大型 SPA 组织方式（动态路由、模块化 i18n、纯逻辑分层、双层测试）的极佳教材。适合：想要本地/内网开发者工具箱的人直接 Docker 部署使用；想抄"工具集合型产品"架构的 Vue 开发者精读 `src/tools/` 注册表三件套与 `scripts/create-tool.mjs`；贡献者按脚手架加一个新工具是最友好的入门 PR。使用前需接受其维护放缓的现状，新功能预期应以自建 fork 为准。
