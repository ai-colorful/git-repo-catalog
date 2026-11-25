# realworld-apps/realworld 项目分析

> 星数 84318（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：RealWorld 是"所有 Demo 应用之母"（the mother of all demo apps）——用同一个 Medium.com 克隆应用（Conduit）的统一 API 规范 + 共享测试套件 + 文档中心，把上百种前端/后端框架的"真实级"示例应用统一起来。

| 事实 | 值 | 出处 |
| --- | --- | --- |
| 仓库 | realworld-apps/realworld | GitHub |
| 创建时间 | 2016-02-26 | gh api repos/realworld-apps/realworld |
| 星数 | 84318（快照 2026-10-24）；forks 7700，watchers 1498 | 任务参数 / gh api |
| 主语言 | TypeScript（本仓库主体是规范与测试文件，24 个 .ts 为 Playwright E2E 套件） | tarball 统计 |
| 许可证 | MIT（2021 Thinkster + 2026 c4ffein），框架 Logo 除外，故 GitHub 标 NOASSERTION | `LICENSE` |
| 最近活跃 | 最后推送 2026-10-30，最近提交 2026-10-29 | gh api pushed_at / commits |
| 本仓库性质 | 规范与文档中心（spec & docs hub），不含任何可运行应用 | `CLAUDE.md` |
| 主页 | https://docs.realworld.show | gh api homepage |

## 2. 项目是什么，解决什么问题

RealWorld 要解决的是框架示例"todo 应用太浅"的经典问题：todo demo 只能展示一个框架的表面能力，传达不了用该框架构建**真实应用**所需的知识与工程约束（认证、CRUD、分页、关注、点赞等）（`README.md`、`docs/src/content/docs/introduction.mdx`）。

它的做法是定义一个"甜点位"的示例应用——Conduit（Medium.com 克隆），功能覆盖 JWT 认证、用户注册与设置、文章 CRUD、评论（可删不可改）、文章分页列表、收藏、关注用户（`docs/src/content/docs/implementation-creation/features.md`）。所有实现都遵守同一份 API 规范，因此**任意前端可搭配任意后端**，这构成了"mix and match"的核心卖点（`README.md`）。

需要特别澄清的是：这个 84k 星仓库本身**不是**任何实现，而是整个生态的"合同文本 + 验收标准"所在地。`CLAUDE.md` 开头即写明："This is the RealWorld spec & docs hub — not an implementation. ... Nothing here is a runnable app."。上百个真正的实现（React/Angular/Node/Django 等 150+ 个）各自住在独立仓库，通过 CodebaseShow（codebase.show）索引展示（`README.md`、`docs/src/content/docs/introduction.mdx`）。

仓库同时提供三类"公共基础设施"：一是 API 契约与后端验收测试（`specs/api/`），二是前端共享 E2E 验收测试与选择器契约（`specs/e2e/`），三是所有前端实现必须采用的统一 UI 主题（`assets/theme/styles.css`）。另有官方托管服务：api.realworld.show（免 key 的公共后端）与 demo.realworld.show（在线演示）（`README.md`）。

生态位上，RealWorld 已成为各框架社区事实上的"标准练手项目"：框架作者用它证明自家框架能做真实应用，学习者用它横向对比同一应用在不同技术栈下的写法。这也是它八年多持续涨星到 84k 的根本原因——它不是某一个框架的文档，而是所有框架共享的参照系。

## 3. 技术栈与依赖

本仓库没有应用代码，技术栈围绕"规范编写 + 测试套件 + 文档站"：

- **API 契约**：OpenAPI 3.1.0（`specs/api/openapi.yml`，版本号 2.0.0）
- **API 测试**：Hurl（事实源，`specs/api/hurl/*.hurl`）+ Bruno（由 Hurl 生成，`specs/api/bruno/`），转换器为无依赖的 Bun 脚本（`specs/api/hurl-to-bruno.js`）
- **E2E 测试**：Playwright（`specs/e2e/playwright.base.ts` 引 `@playwright/test`），纯 TypeScript
- **文档站**：Astro 5.16 + Starlight 0.37 + React 19 + Tailwind CSS 4 + TypeScript 5.9（`docs/package.json`），构建用 Astro Check 做类型检查
- **运行时/包管理**：仓库约定一律使用 Bun（`CLAUDE.md`："Use bun, not npm/node"）
- **任务入口**：Makefile（`Makefile`）
- **CI**：GitHub Actions（`/.github/workflows/`）

## 4. 目录结构与代码组织

| 目录 | 职责 | 出处 |
| --- | --- | --- |
| `specs/api/` | 后端 API 契约：openapi.yml + Hurl/Bruno 测试套件 + 转换与运行脚本 | `CLAUDE.md` |
| `specs/api/hurl/` | 13 个 Hurl 测试文件，后端验收的事实源 | 目录统计 |
| `specs/api/bruno/` | 155 个 .bru 文件，从 Hurl 生成的 Bruno 集合，禁止手改 | `CLAUDE.md` |
| `specs/e2e/` | 共享 Playwright E2E 套件（12 个 spec）+ SELECTORS.md 选择器契约 + 基础配置 | `CLAUDE.md` |
| `docs/` | Astro/Starlight 文档站，发布于 docs.realworld.show | `CLAUDE.md` |
| `assets/theme/` | Conduit Minimal CSS v4 共享主题（1526 行），所有前端实现统一 UI | `assets/theme/styles.css` |
| `assets/media/` | 框架 Logo 图与移动端图标资源 | 目录 |
| `.github/` | 4 个 CI workflow + 2 个 issue 模板 + CODEOWNERS | 目录 |

组织方式评述：这是一个典型的"规范仓库（spec repo）"而非 monorepo——自己不实现任何东西，却被各实现仓库以 submodule/ vendored 方式嵌入（E2E 套件按约定被实现方放在 `./e2e` 消费，见 `CLAUDE.md`）。三块内容（API 契约、E2E 套件、文档）物理分目录、逻辑上靠文档站 URL 串成一体，边界清晰。334 个文件中 156 个是 .bru、62 个是 png（多为图标资源），真正的"源码"只有转换器、E2E 套件与文档站配置，体量小而密度高。

## 5. 核心架构与关键实现

### 5.1 API 契约三件套：OpenAPI + Hurl（事实源）+ Bruno（生成物）

`specs/api/openapi.yml` 定义了 Conduit 的 12 个端点：`/users/login`、`/users`、`/user`、`/profiles/{username}`、`/profiles/{username}/follow`、`/articles/feed`、`/articles`、`/articles/{slug}`、`/articles/{slug}/comments`、`/articles/{slug}/comments/{id}`、`/articles/{slug}/favorite`、`/tags`（grep 自 openapi.yml）。

关键设计是**单一事实源**：Hurl 文件是手写维护的真相，Bruno 集合完全由 `specs/api/hurl-to-bruno.js`（约 17.5KB 的自研 Hurl 解析器/转换器，支持 `--check` 模式）生成，`CLAUDE.md` 明令 "Don't hand-edit specs/api/bruno/"，且 CI 用 `make bruno-check` 校验两者同步（`.github/workflows/bruno-check.yml`）。这消除了双测试套件常见的漂移问题。

Hurl 测试同时承担断言职责，例如 `specs/api/hurl/auth.hurl` 对注册响应逐字段断言（bio/image 必须为 null、token 必须为非空字符串），并通过 `[Captures]` 在请求间传递认证 token——一份文件既是文档又是可执行验收标准。

### 5.2 前端 E2E 套件：三模式 + 选择器契约

`specs/e2e/` 是 128 个 test 用例、3291 行 TypeScript 的 Playwright 套件，覆盖 auth、articles、comments、social、settings、navigation、url-navigation、error-handling、null-fields、user-fetch-errors、health、xss-security 共 12 个维度（`wc -l specs/e2e/*.spec.ts`）。

最有意思的设计有二。其一是**三运行模式**（`specs/e2e/helpers/config.ts`）：同一套测试以 `TEST_MODE` 适配 spa（浏览器直连 REST、JWT 存客户端）、ssr（服务端渲染 + httpOnly cookie）、fullstack（全栈自有，不假设外部 API）三类实现，通过 `BROWSER_API`/`EXTERNAL_API` 两个能力开关让用例自行跳过不适用的假设——这保证套件能同时验收 SvelteKit/Next 类 SSR 应用和纯 SPA。其二是**选择器契约**（`specs/e2e/SELECTORS.md`）：逐条列出实现方必须提供的 CSS 类、name 属性、路由与文本标签，使共享测试与任意实现解耦。

实现方通过扩展 `specs/e2e/playwright.base.ts` 导出的 baseConfig 接入（覆写 baseURL 与 webServer），接入代码在文件头注释中给出完整示例：

```ts
export default defineConfig({
  ...baseConfig,
  use: { ...baseConfig.use, baseURL: 'http://localhost:3000' },
  webServer: {
    command: 'npm run start',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
    timeout: 120_000,
  },
});
```

基础配置强制 workers:1 串行、失败截图、首次重试开 trace（`specs/e2e/playwright.base.ts`）。

`specs/e2e/xss-security.spec.ts` 值得一提：通过 API 直接注入 XSS payload 绕过前端清洗做冒烟测试，文件头用大段免责声明界定其"非安全审计"边界，工程态度严谨。

### 5.3 共享 UI 主题

`assets/theme/styles.css`（1526 行）是"Conduit Minimal CSS v4"，用 CSS 变量定义品牌色/排版尺度，文件头注释精确列出"只包含本代码库实际用到的类"（Layout/Nav/Buttons/Forms/Cards/Pagination/Tags/Pages 等），替代了早期依赖 Bootstrap 4 模板的做法——所有前端实现据此渲染出像素级一致的 UI，让"对比框架"的实验变量只剩框架本身。

## 6. 构建与运行

仓库自身无可运行应用，构建/验证入口全部在 `Makefile` 与 `CLAUDE.md`：

```bash
# API 套件（对准一个运行中的后端，在 specs/api/ 下）
HOST=http://localhost:3000/api ./run-api-tests-hurl.sh    # 事实源
HOST=http://localhost:3000/api ./run-api-tests-bruno.sh   # 生成镜像

# Bruno 集合维护
make bruno-generate   # 由 hurl/ 重建 bruno/
make bruno-check      # CI 检查：bruno/ 不同步则失败

# 文档站（Bun）
make documentation-setup   # cd docs && bun install
make documentation-dev     # 本地开发服务器
make documentation-build   # 生产构建（astro check && astro build）
```

E2E 侧无本地"运行"概念：实现仓库 extend `playwright.base.ts` 后用自身 Playwright 配置执行（`CLAUDE.md`）。

## 7. 收录规模与内容组织（内容型特例）

- **收录机制**：实现不在本仓库收录，而是"独立仓库 + CodebaseShow 网站索引"；本仓库 README 只直接列出通过全套 API 规范测试的"Spec-compliant backends"：Nitro + Prisma + Zod（TypeScript）与 Django Ninja（Python）两例（`README.md`）——即官方背书以测试通过为准，而非人工 curated 清单。
- **规模统计**（tarball 实测）：OpenAPI 12 端点；Hurl 13 文件约 1709 行、154 个 HTTP 断言；Bruno 155 个 .bru；E2E 12 个 spec、128 用例、3291 行；文档站 23 个 md/mdx 页；主题 CSS 1526 行。
- **文档分类体系**（`docs/src/content/docs/`）：specifications/backend（endpoints、api-response-format、error-handling、cors、hurl、bruno、postman、tests 共 8 页）、specifications/frontend（api、routing、styles、templates、tests 共 5 页）、specifications/mobile-specs、implementation-creation（introduction、features、expectations）、community（authors、resources、special-thanks）。
- **生态外延**：codebase.show（实现索引站）、docs.realworld.show（文档）、api.realworld.show（公共托管后端，免 key）、demo.realworld.show（演示站）、GitHub Discussions 的 "WIP implementations" 分类充当实现认领池（`README.md`）。
- **组织评价**：与典型 awesome 清单"收录他人仓库链接"不同，RealWorld 用"规范 + 验收测试"替代人工审核——一个实现是否算数，取决于能否跑通 `specs/api` 与 `specs/e2e`，收录的公信力由测试而非 curator 背书，这是它在内容型仓库中最独特的机制。

## 8. 更新机制与维护状态

- **贡献流程**：标准 fork + PR，配套 Angular 风格 commit 规范（type 限 docs/feat/fix，scope 限 specs/project，2025-02 起引入），issue 用两个 YAML 模板（`CONTRIBUTING.md`、`.github/ISSUE_TEMPLATE/`）。
- **自动化**：4 个 workflow——`bruno-check.yml`（每次 push/PR 校验 Bruno 同步）、`deploy-docs.yml`（docs 变更自动发 GitHub Pages）、`codeql.yml`（安全扫描）、`spammy-guardian.yml`（反骚扰），依赖升级靠 dependabot 式 PR（近提交可见 actions/checkout 4→7 等）。
- **活跃度**：gh api 显示 92 位贡献者、open issues 28；最近提交 2026-10-29 密集修复 E2E 断言细节（如 "really delete on the server in the 200-instead-of-204 tests Fixes #1725"），2026-10-30 更新 LICENSE 与主题横幅——规范处于持续微调的活跃维护态（注意：任务清单备注"最后推送 2026-08"已过时，实测为 2026-10-30）。
- **维护者结构**：两名 active maintainer——c4ffein（维护 spec、测试套件与演示站）与 Manuel Vila（CodebaseShow 作者、Layr 框架作者），版权行从 Thinkster（2021）变更为 c4ffein（2026）表明项目已易主接棒（`README.md`、`LICENSE`）。历史上该仓库原属 gothinkster 组织（2016 年创建），现为 realworld-apps 组织（gh api created_at + 组织名）。

## 9. 亮点与风险

**亮点**

1. "规范即测试"的工程范式：Hurl 单一事实源 → Bruno 生成 + CI 强制同步，杜绝双套件漂移（`specs/api/hurl-to-bruno.js`、`.github/workflows/bruno-check.yml`）。
2. 三模式 E2E 设计：spa/ssr/fullstack 一套用例通吃三类架构，能力开关正交拆分，是跨实现验收测试的教科书案例（`specs/e2e/helpers/config.ts`）。
3. 选择器契约（`SELECTORS.md`）把 UI 验收标准写成显式文档，实现了"任意实现 × 共享测试"的解耦。
4. 生态完整闭环：规范 + 测试 + 主题 + 文档站 + 公共托管 API + 演示站 + 实现索引站，八年沉淀形成强大的网络效应与社区惯性。
5. 文档质量高：CLAUDE.md/CONTRIBUTING/文档站分层清楚，甚至为 AI 协作写了专门的语境说明。

**风险/局限**

1. 单点依赖少数维护者：spec 演进与实现认证实质上由 c4ffein 一人主导，接棒历史短（LICENSE 2026 才变更），bus factor 偏低。
2. 实现质量不受本仓库控制：150+ 实现散落在各自仓库，多数长期无人维护，CodebaseShow 索引无法保证时效性；官方认证后端仅 2 个，"通过全套测试"的门槛事实上限制了认证覆盖面。
3. 规范演进与存量实现的张力：E2E 断言细节仍在调整（如 200 vs 204 语义），每次规范微调都可能造成大量下游实现"合规性失效"。
4. 规模中小而星数巨大：84k 星对应的本体只有几百个文件，star 数更多反映生态名望而非本仓库代码体量，评估技术含量时应看向 specs/ 与下游实现。

## 10. 总结评价

RealWorld 是开源世界里罕见的"元项目"：它用一个精心裁剪的 Medium 克隆，把上百个框架的示例应用统一到一份可执行的 API 规范与共享验收测试之下，本体是规范仓库，价值却在整个生态。八年多来它靠"真实感 demo + mix and match + 框架社区自传播"积累了 84k 星，当前仍以每周级别的节奏打磨测试细节。适合三类人：想认真学一门新框架的开发者（按规范实现一遍 Conduit 是最好的练手项目）、框架作者（补齐一个 RealWorld 实现是最有说服力的展示）、以及任何在设计"跨实现验收测试/契约测试体系"的工程负责人——specs/ 目录下的三件套设计（OpenAPI + Hurl 事实源 + 生成式 Bruno + 三模式 Playwright + 选择器契约）本身就是值得通读的范本。
