# reactive-resume/reactive-resume 项目分析

> 星数 32500（快照 2026-10-24；GitHub API 实测 45382，2026-11-05）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-06

## 1. 项目速览

免费开源的简历构建器：可视化编辑简历与求职信、实时 PDF 预览、ATS 检查、求职申请跟踪，支持完全自托管。**注意：当前 HEAD（v6.1.0）是彻底重构后的新架构，派发提示中的 NestJS + Angular + Nx + Puppeteer 属于旧 v4 架构，已被完全替换。**

| 项目 | 值 | 出处 |
|------|------|------|
| 名称 | reactive-resume/reactive-resume | package.json |
| 版本 | 6.1.0 | package.json |
| 星数 | 45382（2026-11-05 API 实测） | gh api |
| 语言 | TypeScript（约 1959 个 TS/TSX 文件，约 33.4 万行） | find 统计 |
| 许可证 | MIT（Copyright (c) 2026 Amruth Pillai） | LICENSE |
| 创建时间 | 2020-03-25 | gh api |
| 最近 push | 2026-11-05（分析前一天） | gh api |
| 贡献者 | 约 239 人 | gh api contributors 分页 |
| 最新 release | v6.1.0（2026-11-05）；v6.0.9/6.0.8 于 2026-10-29 | gh api releases |

## 2. 项目是什么，解决什么问题

Reactive Resume 由 Amruth Pillai 于 2020 年创建（gh api created_at），目标是让用户在不被收费、不被追踪的前提下写出专业的简历。用户注册后可从零开始或导入既有简历（支持 JSON Resume、LinkedIn 数据导出、PDF、Word 文件），所见即所得地编辑，最终导出 PDF、DOCX、Markdown、JSON 或生成带密码保护的公开分享链接（README.md "Features" 一节）。

与"模板填空"类工具不同，v6 的编辑器强调页面即画布：Write/Design/Check 三种模式、页面内点击选中与行内编辑、实时 PDF 预览、200 步撤销与自动保存（README.md）。Check 模式内置 ATS（求职系统）检查：可读性规则、职位描述关键词匹配、"招聘软件读到的纯文本"视图，甚至对导出后的 PDF 做二次检查（apps/web/src/features/resume/editor/check、packages/resume/src/ats、packages/resume/src/ats-pdf）。

它还把"简历"延展为一个求职工作台：求职申请跟踪（applications/lanes 看板）、求职信（cover letters，v6 起独立存储）、AI 助手（用户自带 API Key 的多供应商接入）、REST API 与 MCP 服务器供自动化/AI 客户端操作简历（apps/web/src/features/applications、packages/mcp、README.md）。文档站明确 v6 是"full redesign"（docs/self-hosting/upgrading-to-v6.mdx），从 v4/v5 迁移有专门指南与启动时自动迁移。

商业模式极为克制：README 明言"MIT 许可、无追踪、无广告、无付费档（no paid tier），可选 AI 功能使用你自己连接的供应商"，收入来自 GitHub Sponsors / OpenCollective 与 Vultr、Hostinger、.cv 域名、Vercel OSS Program 等赞助（README.md）。代码库中不存在付费墙或功能门禁逻辑。

## 3. 技术栈与依赖

**运行形态**（均为真实文件核实）：

- **服务端**：Hono 4 + @hono/node-server（apps/server/package.json），单容器同时服务 API 与 Web 静态资源（apps/server/src/http/app.ts 中 serveWebDistStatic/handleWebApp）
- **API 层**：oRPC（@orpc/server/client/zod/tanstack-query）类型安全 RPC + 自动 OpenAPI 文档（apps/server/src/openapi/generate-spec.ts）
- **数据库**：PostgreSQL + Drizzle ORM/drizzle-zod，简历数据以 JSONB 存储（packages/db/src/schema/resume.ts 第 115 行 `data: pg.jsonb("data").notNull().$type<ResumeData>()`）；Redis 用于限流与 agent 流式续传（ioredis、resumable-stream）
- **认证**：better-auth，插件包括 username、twoFactor、passkey、genericOAuth、admin、jwt、apiKey、oauth-provider（packages/auth/src/config.ts 第 26-29 行）
- **前端**：React 19 + TanStack Router/Query/Form + Tailwind CSS 4 + tiptap 富文本 + dnd-kit 拖拽 + lingui 国际化（apps/web/package.json）；构建用 vite-plus（vp dev/build/test）
- **移动端**：Expo SDK 58 / React Native 0.88（apps/mobile/package.json）
- **PDF 引擎**：Forme——Rust 实现的 PDF 引擎（外部 crate `forme-pdf = 0.30.0`，packages/forme-native/rust/Cargo.toml），编译为 WASM 在浏览器 Worker 与 Node 中运行；**全程无 Puppeteer/Chromium**（全仓库 grep 无 puppeteer 依赖）
- **AI**：Vercel AI SDK（`ai` + 17 个 @ai-sdk/* 供应商适配），BYOK 模式，用户/管理员自配模型与 Key（packages/ai/src/defaults.ts、apps/server/package.json）
- **导出**：DOCX 用 `docx` 库、ODT 自研包、Markdown/JSON 自研（packages/docx、packages/odt、packages/resume/src/markdown.ts）
- **存储**：S3 兼容（@aws-sdk/client-s3）、@vercel/blob，自托管默认配 SeaweedFS（compose.yml）

**Monorepo 工具**：pnpm workspace + Turborepo（turbo.json 定义了按 runtime:server/browser/native 与 role:domain/infra 的依赖边界规则），Lefthook 钩子 + commitlint，knip 查死代码，vite-plus lint --deny-warnings（根 package.json、turbo.json）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|------|------|
| apps/server | Hono 后端：http/（路由）、rpc/、mcp/、openapi/、cloudflare/、startup/（启动检查与调度器） |
| apps/web | React 前端：features/（resume、applications、ai、ats-checker、career 等约 17 个功能域）、routes/、dialogs/ |
| apps/mobile | Expo/React Native 客户端，含独立 locales |
| packages/schema | Zod 数据模型全集：resume、cover-letter、applications、career 等，含 JSON Schema 导出 |
| packages/api | oRPC 路由与 DTO（features/、dto/），前后端共享 |
| packages/auth | better-auth 配置、OAuth、限流、备份码 |
| packages/db | Drizzle schema（resume、auth、applications、career、ai-usage 等约 14 张表）与迁移 |
| packages/pdf | Forme PDF 渲染：18 个模板 React 组件、语义样式引擎、字体子集化、浏览器/服务端双入口 |
| packages/forme-native | Rust + Swift/Kotlin 桥，让 RN 直接渲染 PDF（无 WASM/WebView） |
| packages/resume | 简历领域逻辑：ATS 规则、patch（fast-json-patch）、段落、分享、导出分区 |
| packages/ai | AI 任务模型、提示词、供应商默认值 |
| packages/ui、utils、env、fonts、email、docx、odt、import、mcp、config | 组件库、工具、环境变量校验、字体、邮件模板、文档转换、导入、MCP 服务器 |
| tooling/、plugins/、migrations/、tests/、docs/ | 开发工具脚本、浏览器扩展插件、DB 迁移、Playwright e2e、MDX 文档站 |

组织方式评述：典型 Turborepo，领域包（schema/api/resume/db）与应用包严格分层，turbo.json 用 `boundaries` 强制"服务端代码不得依赖浏览器运行时"等规则——这是该仓库工程质量上最鲜明的设计（turbo.json 第 13-60 行）。

## 5. 核心架构与关键实现

### 5.1 简历数据模型（Zod 单一事实源）

`packages/schema/src/resume/data.ts`（约 800+ 行）定义 `resumeDataSchema = z.looseObject({...})`：姓名/联系方式等基础信息、summary、experience、education、projects、skills、languages、interests、awards、certifications、publications、volunteer、references、profiles 共 14 类内置 section + 任意 customSections，layout 里用 section ID 数组声明主栏/侧栏内容与顺序（第 404-521 行）。每个字段带 `.describe()` 供 AI 结构化输出与 JSON Schema 导出（packages/schema/src/resume/json-schema.ts）。模板元数据 `templateLayouts` 声明每个模板的栏数、侧栏位置、页眉位置与是否 ATS-safe（packages/schema/src/templates.ts）——同一份数据同时驱动画廊筛选、ATS 检查与 DOCX 导出。

### 5.2 Forme 渲染链路（取代 Puppeteer 的关键设计）

PDF 渲染不是浏览器打印截图，而是自研 PDF 引擎三端复用（packages/pdf/src/）：

- **模板即 React 组件**：18 个模板目录各含 `{Name}Page.tsx`（packages/pdf/src/templates/index.ts 将 template 名映射到组件，defaultTemplatePage = AzurillPage）
- **浏览器端**：`browser.tsx` 加载 `@formepdf/core/pkg-web/forme_bg.wasm` 并经 `@formepdf/core/worker` 在后台 Worker 中渲染——实时预览与下载都不经服务器（README "rendered in a background worker"）
- **服务端**：`server.tsx` 的 `createResumePdfFile` 直接在 Node 中调 forme WASM `renderResume`，带字体镜像与上传图片读取，`assertPdfFonts` 保证缺字形降级为 "?" 而非失败
- **移动端**：`packages/forme-native` 用 Rust 静态库 + Swift/Kotlin 桥原生渲染，README 明言"No WASM or WebView is used by the renderer"

`packages/pdf/src/forme/` 下是 React reconciler（reconciler.ts）、字体子集化（font-subset.ts）、分页（page-map.ts、list-pagination 集成测试）等底层实现。这条链路使"一份简历、三端像素级一致"且无需 Chromium 容器，是 v6 相对 v4 最大的架构跃迁。

### 5.3 服务端：单 Hono 应用 + 领域化 RPC

`apps/server/src/http/app.ts` 注册了极广的路由面：`/api/auth/*`（better-auth）、`/api/rpc/*`（oRPC，160MB 上限支持大文件导入）、公开简历 PDF/DOCX 下载、上传、`.well-known/` 下 MCP server card、agent-skills、OAuth authorization server 元数据、llms.txt、robots/sitemap（第 86-207 行）——可见其面向"人 + 自动化 + AI 客户端"三种访问者设计。启动流程（apps/server/src/index.ts）先跑 startup checks 与迁移，再懒加载 auth 与调度器，SIGTERM 时优雅排空 HTTP 与 career/storage-cleanup 两个后台调度器。

### 5.4 ATS 检查引擎

`packages/resume/src/ats/rules.ts` + `walk.ts` 对简历数据做静态规则分析；`ats-pdf/`（extract.ts、harvest.ts、score.ts、verdict.ts）则用 pdfjs-dist 解析**导出后的 PDF**，检验真实排版产物中的文本抽取、分节定位与评分（packages/pdf/src/ats-extraction.integration.test.tsx 验证该闭环）。"检查你真正发出去的文件"是同类产品中少见的思路。

## 6. 构建与运行

来自根 package.json 的 scripts（均已核实）：

```bash
pnpm install                  # pnpm@12.16.1, Node 24.x
pnpm dev                      # turbo dev（需 .env.local，dotenvx 加载）
pnpm build                    # pdf:translations + turbo run build
pnpm start                    # node apps/server/dist/index.mjs
pnpm typecheck / pnpm test    # turbo typecheck / vitest（--concurrency=1）
pnpm test:e2e                 # playwright test
pnpm db:generate / db:migrate # Drizzle 迁移
pnpm lint / pnpm format       # vp lint --deny-warnings / vp fmt
```

自托管（compose.yml）：`docker compose up -d` 一键拉起 postgres + redis + seaweedfs（S3 兼容存储）+ reactive_resume（端口 3000，健康检查 `/api/health`）；也支持 Vercel、Cloudflare Workers（wrangler.jsonc + tooling/cloudflare）、Hostinger 1-click、Kubernetes 与 Cloudron（docs/self-hosting/ 下各有专页）。

## 7. 测试与工程质量

- **单元/集成测试**：515 个 *.test.ts(x) 文件（find 统计），vitest（经 vite-plus `vp test`），覆盖 v8；领域包内测试密度高（如 packages/auth/src 几乎每文件配对测试，schema/applications/ 下 20+ 个测试文件）
- **E2E**：Playwright（playwright.config.ts），tests/e2e/ 有 auth、db、ai-stub、extension 等完整 fixture 体系
- **CI**（.github/workflows/ci.yml）：test、check-mobile-bundle、check-cloudflare、check-vercel、smoke-test 多矩阵，可用 Blacksmith 32vcpu 自托管 runner；另有 docker-publish、mobile-release/ota、autofix 等 6 个 workflow
- **质量工具**：turborepo 依赖边界强制、knip 死代码检查、lefthook + commitlint、vp lint --deny-warnings（警告即失败）、turbo --affected 增量检查（package.json check:affected）
- **国际化**：55 个 locale 的 .po 文件（apps/web/locales），Crowdin 托管，含 AI 辅助翻译 skill（.agents/skills/fill-translations）
- 观察：迁移用启动时自动执行并有集成测试（packages/db/src/migrations.integration.test.ts）；仓库带 AGENTS.md、CODING_STANDARDS.md、.agents/（ADR 与 runbooks），显示高度 AI 协作化的工程流程。

## 8. 社区与维护状态

- **极活跃**：最近 push 2026-11-05（分析前一天），v6.1.0 release 2026-11-05，10 月内连发 v6.0.8→v6.0.9→v6.1.0（gh api）
- **贡献者约 239 人**（gh api contributors 分页计数）；单人主导（Amruth Pillai）+ 大量社区贡献
- **生态**：Discord 社区、Crowdin 翻译、GitHub Sponsors/OpenCollective、Vultr/Hostinger/Vercel 赞助（README.md）
- **多平台发布**：GHCR 镜像 ghcr.io/reactive-resume/reactive-resume，Docker Hub 沿用 amruthpillai/reactive-resume（README 顶部声明，仓库 2025 年前后从 amruthpillai/ 迁移至组织 reactive-resume/）

## 9. 亮点与风险

**亮点**

1. Forme 自研 PDF 引擎三端复用（浏览器 Worker / Node WASM / RN 原生 Rust），彻底移除 Puppeteer+Chromium 依赖，PDF 导出快且部署轻（packages/pdf、packages/forme-native）
2. Zod schema 单一事实源 + JSONB 存储 + fast-json-patch 增量更新，数据模型同时服务编辑器、AI 结构化输出、JSON Schema 导出与 oRPC/OpenAPI/MCP 三个 API 面
3. "Check" 模式对导出 PDF 本身做 ATS 验证（packages/resume/src/ats-pdf），闭环检查真实产物
4. turbo.json 机器可读的依赖边界 + 515 个测试文件 + --deny-warnings lint，工程纪律在开源应用中属顶尖水平
5. 真正的零付费档、BYOK AI、多形态部署（Docker/Vercel/Cloudflare/Hostinger/K8s/Cloudron），自托管体验完整

**风险/局限**

1. v6 是"full redesign"：v4/v5 数据库不兼容、旧样式需手动脚本转换、无混合版本滚动升级（docs/self-hosting/upgrading-to-v6.mdx），旧生态插件与文章资料大量失效
2. 技术栈极新（Node 24、React 19、Tailwind 4、vite-plus、oRPC、Expo 58、forme-pdf 锁定 0.30.0），部分依赖小众或 beta（drizzle-zod beta 版），升级与社区问答成本高
3. 公开分析基于 tarball 快照，无 git 历史佐证 commit 节奏；v6.1.0 于 2026-11-05 发布，可能仍处快速迭代期，自托管者需频繁跟版
4. 单维护者主导，bus factor 偏低（contributors 239 人但 release/架构决策集中于作者；此判断基于 README 署名与 release 记录，未做 commit 归属统计）

## 10. 总结评价

Reactive Resume 是开源简历工具的标杆项目：v6 用自研 PDF 引擎替代浏览器打印、用类型安全的三面 API（oRPC/OpenAPI/MCP）把简历变成可编程资产，同时坚持 MIT、零付费、BYOK AI 的彻底开源姿态。它适合（1）想完全掌控简历数据、自托管求职工具箱的个人用户；（2）研究"React 组件 → 自研排版引擎 → PDF"这条渲染链路的工程师（packages/pdf 是绝佳教材）；（3）需要简历/ATS/申请跟踪 SaaS 化改造的团队作为基座。读源码的推荐路径：packages/schema/src/resume/data.ts（数据模型）→ packages/pdf/src/templates/（模板）→ packages/pdf/src/forme/render.ts（渲染核心）→ apps/server/src/http/app.ts（服务面）。
