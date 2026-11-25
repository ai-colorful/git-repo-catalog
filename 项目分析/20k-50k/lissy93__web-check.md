# lissy93/web-check 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 JavaScript（TypeScript 为主）｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：对任意网站做"一站式被动 OSINT 体检"的开源工具——输入 URL，并行执行 SSL 证书、DNS、HTTP 安全头、开放端口、技术栈、WHOIS、威胁情报等数十项检查，汇总成一份可视化报告。

| 事实 | 值 | 出处 |
| --- | --- | --- |
| 名称 | Web-Check（v2.4.2） | `package.json` |
| 星数 | 39500（快照）；gh api 实测 35555（2026-11-21） | 任务参数 / gh api |
| 主语言 | TypeScript + JavaScript（Astro/React/Express） | `package.json`、`src/` |
| 许可证 | MIT © Alicia Sykes 2023-2026 | `LICENSE`、`.github/README.md#L261` |
| 最近活跃 | push 2026-11-13；最新 release v2.4.0（2026-11-05）；tag 已到 2.4.2 | gh api |
| 在线实例 | web-check.as93.net（README 内），镜像 Codeberg | `.github/README.md#L87-91` |
| 贡献者 | 约 40 人（gh api contributors 分页推算），单核心维护者 lissy93 | gh api |
| 代码规模 | ~20.5k 行（api/ + src/，JS/TS/TSX/Astro/Svelte/SCSS 合计） | wc -l 统计 |

## 2. 项目是什么，解决什么问题

Web-Check 的目标是让任何人无需登录、无需安装，就能看到"一个网站从外面看是什么样"：证书链与签发者、DNS 记录、响应安全头、Cookie 标志、开放端口、所用技术栈、邮件配置（SPF/DKIM/DMARC）、页面碳排量等。README 明确其定位为"uncover potential attack vectors, analyse server architecture, view security configurations"（`.github/README.md#L70-72`）——即站在外部观察者视角做防御性体检，属于被动/准被动 OSINT，不主动入侵目标。

站点按 7 个分类组织检查项：security、seo、server、domain、email、performance、privacy（`src/data/categories.ts`）。每个检查项都有结构化元数据（标题、摘要、描述、用途说明、延伸阅读资源），如 `src/data/checks/ssl.ts` 为 SSL 检查写明了"证书是公开的，天然是一份记录，能顺藤摸瓜找出未公开的子域名"这类解读思路——项目把"教你怎么读结果"当作一等公民。

工程上它是一个"一次编写、多平台运行"的 serverless 应用：同一份 `api/*.js` 端点代码通过适配层同时跑在 Vercel（Serverless Functions）、Netlify（Lambda）和自托管 Node/Express（`server.js`）上（`src/api/_common/middleware.js`）。公开实例承担大量流量，作者在 README 提到 lambda 开销约 $25/月，靠 GitHub Sponsors 维持（`.github/README.md#L238-241`）。

需要说明：当前 master（v2.4.x）是一次大重构后的 Astro 版，目录为 `src/`（Astro 页面 + React 客户端 SPA）；旧版"frontend/ React + server/ Express"结构已不存在，仅保留 `server.js` 作为自托管入口。

## 3. 技术栈与依赖

| 层 | 技术 | 出处 |
| --- | --- | --- |
| 前端框架 | Astro 7（营销页/文档页 SSG）+ React 19（扫描结果 SPA）+ Svelte 5（少量组件） | `package.json`、`astro.config.mjs`、`src/client/` |
| 样式/交互 | Emotion、framer-motion、recharts、react-simple-maps、react-toastify | `package.json#L27-58` |
| 后端 | Express 5（自托管入口）、express-rate-limit、cors、dotenv | `package.json`、`server.js` |
| 浏览器自动化 | Puppeteer 24 + @sparticuz/chromium（serverless 免下载 Chromium） | `api/screenshot.js`、`api/_common/browser.js` |
| 关键领域库 | wappalyzer（技术栈识别）、whoiser（WHOIS）、cheerio（HTML 解析）、psl（公共后缀）、xml2js、@ghostery/adblocker + @ghostery/trackerdb（追踪器识别） | `package.json`、`api/tech-stack.js`、`api/trackers.js` |
| 工具链 | yarn 1、ESLint 10、Prettier、astro check（TS 类型检查）、sass | `package.json#L72-87` |
| Node 要求 | ^22.23.4 ‖ ^24.16.0 ‖ ≥26.3.0 | `package.json#L6-8` |

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `api/` | 40 个检查端点，每文件一个 serverless function（如 `ssl.js`、`dns.js`、`ports.js`） |
| `api/_common/` | 跨平台 middleware、SSRF 检查跳过逻辑、HTTP 工具、日志、浏览器启动封装 |
| `src/pages/` | Astro 页面：首页营销页、`/check`（扫描界面）、`/checks`（检查项目录）、`/glossary`、API 文档页等 16 个 |
| `src/client/` | React SPA：`App.tsx`（react-router 路由）、`views/`、`components/`、`jobs/`（任务编排）、`analysis/`（结果分析规则引擎）、`hooks/`、`utils/` |
| `src/data/checks/` | 44 个检查项的静态元数据定义（TS），供文档页与前端使用 |
| `src/data/categories.ts` | 7 大分类的文案与 SEO 信息 |
| `src/components/` | Astro 组件（homepage、check、form、scafold 等） |
| `server.js` | 自托管 Express 入口：动态注册 api/ 为路由、聚合端点、限流、健康检查 |
| `public/resources/openapi-spec.yml` | 3633 行 OpenAPI 3.0 规范，描述全部 API 端点 |
| `Dockerfile`/`vercel.json`/`netlify.toml`/`render.yaml`/`fly.toml` | 各平台部署配置 |
| `.github/workflows/` | 6 个 workflow：ci、docker、release、tag、mirror（Codeberg 镜像）、label-pr |

组织方式评述：单一仓库、按"检查项"垂直切分——每个检查 = 1 个 API 文件 + 1 份数据定义 + 1 条前端分析规则，三方对齐良好（如 `ssl` 对应 `api/ssl.js`、`src/data/checks/ssl.ts`、`src/client/analysis/rules/ssl.ts`）。前端混用三个框架略显混杂（Astro 壳 + React SPA + 1 个 Svelte 组件），是重构过渡期的痕迹。

## 5. 核心架构与关键实现

### 5.1 一套 handler，三种运行时（api/_common/middleware.js）

所有检查端点写成纯函数 `(url) => data`，由 `commonMiddleware` 包装成 Vercel 的 `(req,res)` 或 Netlify 的 `(event,context)` 签名（`src/api/_common/middleware.js#L40-109`，按环境变量 `PLATFORM`/`VERCEL`/`WC_SERVER` 判定）。每个端点统一获得：URL 规范化、超时（`PUBLIC_API_TIMEOUT_LIMIT`，默认 40s）、CORS、跳过检查判定。这是"零成本多平台部署"的关键设计。

### 5.2 自托管聚合服务（server.js）

`server.js` 启动时动态 `import` `api/` 下全部 40 个文件注册为 Express 路由，并额外提供 `/api` 聚合端点：并行执行所有检查，每个检查带独立超时（`Promise.race`），单点失败不影响整体（`server.js#L138-191`）。限流三级（100 次/10 分钟、250 次/小时、500 次/12 小时，`API_ENABLE_RATE_LIMIT` 开启，`server.js#L70-98`）。GUI 未编译时返回占位页，编译后静态托管 `dist/client` 并接 Astro SSR（`server.js#L202-235`）。另有 `/healthz` 健康端点。

### 5.3 SSRF/滥用防护（api/_common/check-skipper.js）

作为"输入任意 URL 去探测"的工具，防内网探测是核心安全问题。`shouldSkip()` 在每个请求前判断（`api/_common/check-skipper.js#L145-160`）：总开关 `VITE_DISABLE_EVERYTHING`（实例滥用保护）；管理员黑名单 `API_BLOCKED_HOSTS`（支持域名后缀与 CIDR 段，`isBlockedTarget`）；白/黑名单启用检查项；私网 IP 识别——覆盖 localhost、`.local`、RFC1918、回环、链路本地 169.254、CGNAT 100.64/10、IPv6 回环/ULA（`isPrivateTarget`，L123-142），私网目标只允许本地类检查（get-ip、ssl、dns、ports 等 13 个依赖外部 API 的检查会被跳过）。

### 5.4 检查实现的两条路径：本地探测 vs 外部情报源

- 本地实现（约 24 个端点）：`api/dns.js` 用 Node `dns/promises` 并行解析 9 种记录（A/AAAA/MX/TXT/NS/CNAME/SOA/SRV/PTR）；`api/ssl.js` 用原生 `tls.connect`（`rejectUnauthorized: false`）拉取完整证书链并判有效性；`api/ports.js` 用裸 `net.Socket` 扫 33 个常见端口（每口 1.5s 超时，可经 `PORTS_TO_CHECK` 自定义）；`api/screenshot.js` 优先 execFile 调系统 Chromium，失败回落 Puppeteer；`api/carbon.js` 本地实现 Sustainable Web Design v4 碳排模型（常量与 websitecarbon.com 对齐）；`api/tech-stack.js` 调 Wappalyzer 引擎（serverless 上用 @sparticuz/chromium 兜底二进制路径）。
- 外部公共 API（约 16 个端点）：crt.sh（子域名）、SSL Labs（`api/tls-labs.js`）、Shodan（`api/shodan.js`，需 key）、Google PageSpeed/Safe Browsing（`api/quality.js`，需 key）、Have I Been Pwned（`api/breaches.js`）、urlhaus/PhishTank/Cloudmersive（`api/threats.js`）、Tranco 排名、CertSpotter、RDAP、ipwho.is/geojs/reallyfreegeoip（IP 地理）、archive.org（快照）、GitHub/BlueSky/X（社交存在，`api/social-presence.js`，344 行）等。部分需配 API key 才生效（README 配置表，`.github/README.md#L173-182`）。
- 比例上：**本地实现占多数（~60%），外部依赖集中在威胁情报、排名、地理与社交类检查**。追踪器检查（`api/trackers.js`）用 Ghostery trackerdb 本地引擎，不依赖外部 API。

### 5.5 客户端分析规则引擎（src/client/analysis/）

前端拿到原始检查结果后，由 27 个纯函数 analyzer（`src/client/analysis/rules/`）在浏览器端解读为"发现（finding）"，例如 http-security 规则逐条评 HSTS/CSP/XFO 头是否到位。API 只负责取数，评判逻辑放在客户端，便于即时调整措辞而不动后端。任务编排与状态机在 `src/client/jobs/registry.ts`。

## 6. 构建与运行

```bash
# 源码运行（.github/README.md#L155-161, L206-214）
git clone https://github.com/Lissy93/web-check.git && cd web-check
yarn install
yarn build        # astro check && astro build
yarn start        # node server.js（API + GUI）

# 开发（前后端分离热重载，package.json#L14-16）
yarn dev          # concurrently: nodemon server (3001) + astro dev

# Docker（README Option #5）
docker run -p 3000:3000 lissy93/web-check
# 或自建：docker build -t web-check .

# 一键部署：Netlify / Vercel / Hostinger / Render（README 各有按钮）
# 质量闸门（package.json scripts）
yarn lint         # ESLint（.config/eslint.config.js）
yarn typecheck    # astro check
yarn format:check # Prettier
```

可选环境变量：`SHODAN_API_KEY`、`GOOGLE_CLOUD_API_KEY`、`API_DISABLED_CHECKS`、`API_BLOCKED_HOSTS`、`API_ENABLE_RATE_LIMIT` 等，全部可选（`.github/README.md#L167-201`）。

## 7. 测试与工程质量

- **测试：仓库内无任何单元/集成/e2e 测试文件**（全仓搜索 `*test*`/`*spec*` 仅命中 OpenAPI spec 与文档页），`package.json` 亦无 test script。质量保障完全依赖静态检查：ESLint、Prettier、`astro check`（TS 类型检查，源码大量使用 TS/`satisfies`）。
- CI（`.github/workflows/ci.yml`）：PR 触发 lint、typecheck、format 三道检查，带变更路径检测（lockfile/workflow 变更单独识别）与并发去重；`docker.yml` 构建发布镜像、`release.yml`/`tag.yml` 处理发版、`mirror.yml` 同步 Codeberg、`label-pr.yml` 自动打标。
- 工程质量观察：代码风格统一（Prettier 100 列、单引号、尾逗号），注释密度高且多为"为什么"而非"是什么"；Dockerfile 为三阶段构建并在构建期验证 SSR 入口可加载（`Dockerfile#L49`）；依赖版本新（Astro 7、React 19、Express 5），`resolutions` 显式钉住若干有漏洞通告的传递依赖（`package.json#L65-71`）。主要短板是无自动化测试，重构（v2 大版本）回归只能靠类型检查与人工。

## 8. 社区与维护状态

- **维护节奏**：非常活跃。gh api 显示 push 最近一次 2026-11-13（约一周前）；tag 序列 2.3.11 → 2.4.2 密集发布，最新 release v2.4.0 于 2026-11-05（gh api releases/latest）。项目创建于 2023-06。
- **贡献者结构**：gh api contributors 分页推算约 40 人；README 的贡献者徽章由自动化 workflow（credits.yml）持续插入，但实际核心维护者为单一人 Alicia Sykes（lissy93）——她也是 Dashy（15k+ 星仪表盘项目）的作者，Web-Check 是其个人高产出代表作之一（`.github/README.md` 赞助区与 LICENSE 版权行佐证）。此为"强个人品牌单维护者"模式，与任务档案一致。
- **商业化/资助**：README 顶部有 NinjaPear、Hostinger、Warp 三个赞助商位；`FUNDING.yml` 开启 GitHub Sponsors；公开实例靠赞助覆盖 lambda 费用。
- **风控现状**：`check-skipper.js` 中留有 `VITE_DISABLE_EVERYTHING` 总开关及提示文案（"due to excess abuse and bot traffic"，`api/_common/check-skipper.js#L30-39`），说明公开实例曾因滥用/流量成本临时关停过——大星数免费 OSINT 服务的典型运维压力。
- 星数走势：任务快照 39500（2026-10-24），gh api 实测 35555（2026-11-21），差异较大未确认原因（可能含星标清理或统计口径差），不影响档位归属。

## 9. 亮点与风险

**亮点**
1. 跨运行时 API 架构优雅：一份 `api/*.js` 经 middleware 适配同时服务 Vercel/Netlify/Node/Docker 四种形态，自托管成本极低（`api/_common/middleware.js`、`server.js`）。
2. 检查矩阵完整且文档化到位：44 个检查定义、7 大分类、3633 行 OpenAPI 规范、每项检查带"怎么解读"的教学文案，兼具工具与学习材料价值（`src/data/checks/`、`public/resources/openapi-spec.yml`）。
3. 安全工程自觉性高：私网 IP/CIDR 黑名单/检查项黑白名单/多级限流/实例总开关层层设防，体现作者对"扫描工具自身被滥用"的清醒认知（`api/_common/check-skipper.js`、`server.js`）。
4. 零配置可自托管：无必填环境变量，Docker 镜像内置 Chromium 与 traceroute，`docker run` 即完整功能（`Dockerfile`、README Option #5）。
5. 多数检查本地实现（DNS/TLS/端口/头/Carbon/追踪器等约 6 成），不强绑外部服务，key 缺失时优雅降级为 skipped。

**风险/局限**
1. **无任何自动化测试**：全部质量保障靠 ESLint + 类型检查，对持续大重构的项目是明显短板（第 7 章证据）。
2. 滥用边界（客观陈述，防御性定位明确）：工具可枚举子域名、扫端口、批量截图并串起威胁情报，若被用于对第三方站点的批量侦察会触及目标方接受度与地区法规差异；项目自身已做私网防护、黑名单与限流，公开实例亦有滥用关停机制，但自托管实例的合规责任在使用者。
3. 外部 API 检查可用性受制于人：crt.sh、SSL Labs、HIBP 等公共源的配额与稳定性波动会直接影响约 4 成检查项；Shodan/Google 等需自配 key。
4. 三框架混用（Astro+React+Svelte）与 40 个独立 lambda 的冷启动，使前端栈与部署成本略重；截图/技术栈检查在 serverless 上强依赖 Chromium，资源开销大。

## 10. 总结评价

Web-Check 是"个人维护者做到极致"的典范项目：一个清晰的单一价值主张（把网站的外部可见面一次看全），用一套精巧的跨平台 serverless 架构支撑公开实例与自托管双轨，44 项检查覆盖 DNS/SSL/TLS/HTTP 头/端口/技术栈/邮件配置/威胁情报/碳排等维度，并以教学级文档与 OpenAPI 规范放大其价值。它的星数不来自复杂算法，而来自"开箱即用 + 一键部署 + 结果可读"的产品直觉，这正是 Dashy 作者 Alicia Sykes 的强项。适合安全初学者与站长做自家站点的防御性体检，适合 OSINT 从业者当快速侦察看板，也适合想学"如何把 Node API 写成可移植 serverless 函数"的工程师读源码；但引用其代码时应注意项目尚无自动化测试，且部署公开实例前务必配置私网防护与限流。
