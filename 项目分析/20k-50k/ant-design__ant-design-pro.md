# ant-design/ant-design-pro 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：Ant Design 官方出品的企业级中后台前端脚手架（boilerplate/template），口号 "Use Ant Design like a Pro"。

| 项目 | 内容 |
| --- | --- |
| 名称 | ant-design/ant-design-pro |
| 星数 | 39500（快照）；gh api 2026-11-14 实测 38,958 stars / 8,245 forks |
| 语言 | TypeScript（README.md、package.json 均无 JS 源码，全部 TS/TSX） |
| 许可证 | MIT（LICENSE） |
| 创建时间 | 2017-08-25（gh api） |
| 当前版本 | v6.0.3（package.json `"version": "6.0.3"`；v6.0.0 于 2026-04-30 发布） |
| 最近活跃 | 最近 push 2026-11-13（gh api）；tarball 内文件时间戳 2026-10-26 |
| 仓库性质 | 应用模板仓库（`"private": true`），非 npm 发布包 |

## 2. 项目是什么，解决什么问题

Ant Design Pro 是蚂蚁 Ant Design 团队维护的**开箱即用的中后台管理系统模板**。企业中后台应用有大量同质化需求：登录鉴权、菜单布局、权限控制、表格增删改查、表单分步提交、国际化等。这个仓库把这些"每个中后台项目都要写一遍"的基础设施做成可直接克隆使用的完整工程，让开发者从模板起步而不是 `create-react-app` 从零搭（README.md "An out-of-box UI solution for enterprise applications as a React boilerplate"）。

它同时是 antd 生态的**官方最佳实践示范**：仓库里每个页面都演示了 ProComponents（ProTable/ProForm/PageContainer）、Umi Max 插件体系、权限方案、i18n 的标准用法，因此它的另一重身份是"可运行的文档"。开发者常以 `src/pages/table-list` 为参考拷贝到自己项目里。

v6（2026-04 发布，release tag v6.0.0）是一次彻底现代化：升级到 React 19 / antd 6 / Umi Max 4，引入 Tailwind CSS v4 + antd-style 主题方案，构建器从 Webpack 迁移到 UmiJS 团队自研的 Rust 打包器 **utoopack**（`config/config.ts` 中 `utoopack` 配置段），并新增了基于 Ant Design X 的 AI 聊天助手页面（`src/pages/chatbot/`）和内置 Claude Code Skills（`CLAUDE.md`、`.claude/skills/`）。

与 ant-design 主仓的关系：同为 `ant-design` GitHub 组织下的姊妹仓库，定位互补——antd 是组件库，Pro 是用组件库搭好的完整应用模板；运行时依赖 `antd ^6.6.7` 与 `@ant-design/pro-components ^3.1.14`（package.json），预览站由同组织的 Cloudflare Worker demo API 支撑。

在中文社区生态位：它是"蚂蚁系中后台全家桶"（Umi + antd + ProComponents + dvajs→model/react-query）的旗舰入口，自 2017 年起长期是国内 React 中后台的事实标准模板之一，中文文档完备（README.zh-CN.md、docs/cheatsheet.zh-CN.md）。

## 3. 技术栈与依赖

全部来自 package.json（v6.0.3）：

| 层 | 选型 | 说明 |
| --- | --- | --- |
| UI | antd ^6.6.7、@ant-design/pro-components ^3.1.14 | 组件库 + 高阶业务组件（ProTable/ProForm/Layout） |
| AI | @ant-design/x ^2.9.0（+x-markdown、x-sdk） | 聊天助手页面的 Bubble/Sender/Think 组件 |
| 框架 | @umijs/max ^4.7.27（Umi Max 4） | 路由、构建、插件运行时（initialState/access/request/reactQuery/locale/layout 等） |
| 视图 | react / react-dom ^19.3.1 | React 19 |
| 数据 | @tanstack/react-query ^5.104.5 | 服务端状态管理（v6 取代传统 dva 的主数据层） |
| 样式 | tailwindcss ^4.3.3 + antd-style ^4.1.0 | 原子类布局 + antd theme token 的 CSS-in-JS |
| 图表/地图 | @ant-design/plots ^2.6.8、d3 ^7.9.0、topojson-client、china-division | Dashboard 图表与中国行政区数据（src/utils/chinaDivision.ts） |
| 构建 | utoopack（@utoo/pack，Rust 打包器）、Node ≥ 22 | `max build` 底层走 utoo；CI 用 `ut build`（.github/workflows/ci.yml） |
| 语言/质量 | typescript ^7.0.2、@biomejs/biome ^2.6.3 | Biome 全面替代 ESLint+Prettier（biome.json） |
| 测试 | vitest 5.0.5（browser 模式）+ Playwright | 单测跑在真实浏览器里（vitest.config.ts `browser.enabled: true`） |

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `config/` | Umi Max 构建期配置：config.ts（主配置）、routes.ts（声明式路由）、proxy.ts（代理）、defaultSettings.ts（ProLayout 默认外观）、oneapi.json（OpenAPI schema） |
| `src/app.tsx` | Umi 约定的运行时入口：getInitialState + layout + request 配置 |
| `src/access.ts` | 权限定义（基于 initialState 的 access 插件约定） |
| `src/pages/` | 页面，按模板域分组：dashboard/form/list/profile/result/exception/account/user/chatbot/table-list |
| `src/services/ant-design-pro/` | 由 OpenAPI 生成的请求层（`max openapi` 生成，禁止手改） |
| `src/locales/` | 8 种语言的翻译文件 |
| `src/components/` | 模板支撑组件（Footer/RightContent/ErrorBoundary/OfflineBanner 等） |
| `mock/` + `src/pages/**/_mock.ts` | 全局 mock + 页面级共置 mock（13 个页面 _mock.ts） |
| `cloudflare-worker/` | 独立的 Hono 应用，为预览站提供后端 API（wrangler.toml name=pro-api） |
| `e2e/`、`tests/` | Playwright e2e 与 Vitest 设置 |
| `scripts/` | i18n-remove.js（剥离国际化）、simple.js（精简版模板）两个一次性转换脚本 |
| `docs/` | cheatsheet（中英各 624 行）+ roadmap（v6.1.0 发布计划） |
| `.claude/skills/`、`CLAUDE.md` | 内置 AI 编码助手技能与项目说明 |

组织方式：单体应用仓库（非 monorepo），但内嵌一个独立 `cloudflare-worker/` 子项目（CLAUDE.md 明确说明它不是 npm workspace）。核心组织原则有两条：一是 **Umi 约定文件**（app.tsx/access.ts/global.tsx/loading.tsx 由框架自动装配），二是**页面共置**——每个页面目录包含 `index.tsx + service.ts + _mock.ts + data.d.ts + style.ts`（CLAUDE.md "Page Co-location" 一节），v6 相比 v5 把数据与 mock 下沉到了页面内部。

## 5. 核心架构与关键实现

### 5.1 Umi Max 插件式骨架（config/config.ts）

整个应用的骨架是一份 240 行的 defineConfig，**声明即架构**：路由来自 `routes`，运行时插件 `model/initialState/access/request/reactQuery` 全开，i18n 默认 zh-CN 且跟随浏览器（`baseNavigator: true`），antd 统一 filled 变体与 AlibabaSans 字体，`openAPI` 插件读 `config/oneapi.json` 生成 services，`mock.include: ['src/pages/**/_mock.ts']` 定义页面级 mock 约定，`exportStatic` 开启静态导出，`define` 注入 `__APP_VERSION__/__UMI_VERSION__/__UTOO_VERSION__`（config/config.ts L36-239）。utoopack 段配置了 md 原始加载器与懒编译（L217-230）。

### 5.2 登录态与权限：initialState → access → 路由守卫

三段式权限链（这是 Pro 最被模仿的模式）：

1. `src/app.tsx` 的 `getInitialState()` 在应用启动时请求 `/api/currentUser`，失败则 `history.replace('/user/login?redirect=...')`（L32-73）；
2. `src/access.ts` 把用户映射为权限键，全仓只有一条规则：
   ```ts
   export default function access(initialState) {
     const { currentUser } = initialState ?? {};
     return { canAdmin: currentUser && currentUser.access === 'admin' };
   }
   ```
3. `config/routes.ts` 在路由上声明 `access: 'canAdmin'`（L52-68 的 /admin 分支），布局菜单自动隐藏无权页面；`layout.onPageChange` 再兜底重定向未登录访问（src/app.tsx L113-121）。mock 凭据为 `admin/ant.design`、`user/ant.design`（CLAUDE.md）。

### 5.3 页面标准范式：ProTable + react-query（src/pages/table-list/index.tsx）

`table-list` 是全仓库的"金标准"页面（348 行），v6 数据流的关键变化是写入走 TanStack Query：查询仍用 ProTable 的声明式 `request={rule}`，删除用 `useMutation` + `actionRef.current?.reloadAndRest()` + `queryClient.invalidateQueries({ queryKey: ['rule'] })`（L36-48）；列定义用 `valueEnum`/`valueType` 描述渲染，详情复用同一 columns 传入 `ProDescriptions`（L330-340）；文案全部 `FormattedMessage`/`useIntl`。dashboard/analysis 页同样以 `useQuery` 取数（src/pages/dashboard/analysis/index.tsx）。

### 5.4 后端与 mock 双轨制

演示数据有两条真实链路：本地开发走 `mock/`（5 个文件，含 fakeList.ts 的 mockjs 随机数据）与 13 个页面级 `_mock.ts`；而线上预览站（preview.pro.ant.design）由 `cloudflare-worker/` 提供——一个 Hono 应用挂载 user/dashboard/table/monitor 等路由，CORS 限定 preview 域名（cloudflare-worker/src/index.ts），生产 baseURL 在 src/app.tsx L193 硬编码为 `https://pro-api.ant-design-demo.workers.dev`。另有 `@umijs/request-record` 支持 `npm run record` 录制真实请求回放（package.json scripts）。

### 5.5 AI 原生工程化（CLAUDE.md + .claude/skills/）

v6 最 distinctive 的一点：仓库自带两个 Claude Code Skills——`/antd`（封装 `@ant-design/cli` 离线元数据，强制"写 antd 代码前先 `npx antd info <Component>`"，并提供废弃用法 lint 与跨版本迁移）和 `/pro-upgrade`（diff 最新模板与本项目，合并框架改动保留业务代码，支持 v5→v6 跨版本升级，CLAUDE.md L45-58）。配套 `scripts/simple.js`（不可逆删除 10 类演示页面产出精简版）与 `scripts/i18n-remove.js`（把 intl 调用替换为中文硬编码），均为面向 AI/人工二次开发的模板裁剪工具。

## 6. 构建与运行

来自 package.json scripts 与 CLAUDE.md：

```bash
npm start          # UMI_ENV=dev max dev（带 mock）
npm run dev        # MOCK=none max dev（无 mock）
npm run build      # max build（utoopack）
npm run lint       # biome lint + tsc --noEmit
npx antd lint ./src   # antd 专属检查（提交前必须通过）
npm test           # vitest run（浏览器模式，需 Playwright chromium）
npm run e2e        # playwright test
npm run openapi    # 重新生成 src/services/ant-design-pro/
npm run simple     # 不可逆：裁剪为精简模板
npm run i18n-remove# 不可逆：剥离 i18n
```

环境要求 Node ≥ 22（engines），包管理器锁定 npm（package-lock.json，CLAUDE.md 明确禁 yarn/pnpm）。

## 7. 测试与工程质量

- **单测**：9 个测试文件（src/access.test.ts、app.test.tsx、requestErrorConfig.test.ts、table-list/account-settings/dashboard/basic-form/chinaDivision 等），特点是 vitest **browser mode** 跑在 Playwright 驱动的真实 chromium 里（vitest.config.ts L18-26），coverage 用 v8 provider 并排除生成代码。
- **e2e**：仅 `e2e/login.spec.ts` 一个登录冒烟用例 + playwright.config.ts；另有 `npm run record` 请求录制场景（package.json）。
- **CI**（.github/workflows/，10 个 workflow）：ci.yml 四道关（biome lint → tsc → 三系统 build 矩阵 ubuntu/windows/macOS 用 `ut build` → antd-cli.yml 的 antd lint）；e2e.yml、coverage.yml（codecov，codecov.yml）、preview-build/deploy.yml（PR 预览）、react-doctor.yml（React 反模式检查）、issue-labeled/emoji-helper/dependabot.yml。Actions 均按 commit SHA 锁版本。
- **代码质量观察**：Biome 2.6 单工具链、TS strict、commitlint+husky+lint-staged（.commitlintrc.js、.lintstagedrc）；测试数量对模板仓库定位而言够用但偏薄（e2e 仅登录一条）。小瑕疵：CLAUDE.md 仍写 `npm run test (Jest)`，实际已是 Vitest（CLAUDE.md L9 vs package.json L27），文档略有滞后。

## 8. 社区与维护状态

以下均来自 gh api 实测（2026-11-14）与仓库文件证据：

- **活跃度**：最近 push 2026-11-13；过去一年 52 周中 27 周有提交（stats/commit_activity）；近期提交主题为 vitest browser mode 迁移（#11950）、utoopack 懒编译（#11974）、依赖升级（dependabot 常态化），处于持续现代化节奏。
- **Release 节奏**：2026 年 4 个 release——v6.0.0（04-30）、v6.0.1（05-06）、v6.0.2（05-28）、v6.0.3（08-17）；仓库内 docs/roadmap/v6.1.0-release-plan.md（草案，2026-06-01）列了带 issue 编号的分层计划（清理 legacy 类名、修 6 个 bug、依赖对齐），说明下一版本有明确路线。
- **贡献者**：约 371 人（contributors 分页 last=372，gh api）；提交者含团队核心成员（Yiheng、zoomdong 等）与 dependabot。
- **维护判断**：模板仓库天然改动频率低于库类项目，但证据显示并非停滞——v6 大版本、Rust 打包器迁移、AI Skills 体系都是 2026 年的大动作；UmiJS 生态整体热度不及 2018-2020 高峰（星标增速放缓、社区讨论向 Next.js 等迁移）属于公开事实，但该仓库本身维护健康，未观察到弃维迹象。
- **Issue 管理**：open_issues 64，有 issue-labeled、emoji-helper 等自动化机器人（.github/workflows/）。

## 9. 亮点与风险

**亮点**

1. 中后台最佳实践的"活文档"：ProTable/ProForm/权限/i18n 每个模式都有可直接抄的真实代码（src/pages/table-list、src/app.tsx）。
2. 声明式架构密度高：config/config.ts 一个文件即完整骨架，OpenAPI→services 生成、页面级 mock 共置、8 语言 i18n 全部约定化。
3. AI 原生工程化先行者：内置 Claude Code Skills（antd 元数据查询 + 模板自动升级）在同类模板中属首创（CLAUDE.md、.claude/skills/）。
4. 工具链激进现代化：React 19 + antd 6 + TS 7 + Rust 打包器 utoopack + Biome + 浏览器模式测试，CI 覆盖三大操作系统。
5. 诚实的模板裁剪出口：`simple`/`i18n-remove` 两个不可逆脚本明确标注风险（scripts/*.js 注释），降低二次开发成本。

**风险/局限**

1. 强绑定 Umi 生态：Umi Max、max CLI、约定文件深耦合，一旦选定迁移到 Vite/Next.js 成本高；且 utoopack 属 UmiJS 团队新生的 Rust 工具链，成熟度风险需要使用者自担。
2. 它是模板不是框架：无 npm 发布、无版本兼容承诺，业务项目通常靠"克隆+改造"继承，升级依赖 `/pro-upgrade` 的 diff-merge，机制尚新。
3. 测试深度有限：e2e 仅登录一条（e2e/login.spec.ts），核心页面回归主要靠 lint+tsc+build 矩阵兜底。
4. CLAUDE.md 与实际存在轻微漂移（如 test 命令仍写 Jest），提示文档需人工同步。

## 10. 总结评价

ant-design-pro 是"蚂蚁系中后台全家桶"的官方门面：把 antd 6 + Umi Max 4 + ProComponents 3 + react-query 组合成一个权限、布局、i18n、mock、CI 全部就绪的企业应用起点，v6 又用 Rust 打包器、浏览器模式测试和内置 AI Skills 完成了一次对 2026 年工程标准的对齐。它维护活跃（v6.0.3，2026-11 仍有提交）、约 371 位贡献者、发布与路线图规范，短板在于深度绑定 Umi/utoopack 生态以及作为模板仓库固有的低测试深度。适合：需要快速启动 React 中后台项目、且技术选型愿意押注 antd+Umi 生态的团队；想学习 ProTable/ProForm/权限/国际化标准写法的 antd 用户——即便不整体采用，`src/pages/table-list` 与 `src/app.tsx` 也值得通读。对追求 Vite/Next.js 生态或需要长期框架演进保证的团队，应把它当作参考模板而非工程基座。
