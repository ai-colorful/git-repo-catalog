# puppeteer/puppeteer 项目分析

> 星数 95742（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：Chrome/Firefox 浏览器自动化的事实标准库，通过 DevTools Protocol（CDP）与 WebDriver BiDi 双协议提供高层控制 API。

| 项 | 值 |
| --- | --- |
| 名称 | puppeteer/puppeteer |
| 星数 | 95742（快照），分析时 gh api 实测 95771 |
| 主语言 | TypeScript（puppeteer-core/src 共 217 个 .ts 文件） |
| 许可证 | Apache-2.0（LICENSE，"Copyright 2017-2026 Google Inc."） |
| 创建时间 | 2017-05-09（gh api） |
| 最近 push | 2026-10-30（gh api，即分析前一天） |
| 当前版本 | 25.16.0（packages/puppeteer-core/package.json，2026-10-22 发布） |
| 浏览器钉版 | Chrome 156.0.8078.80 / Firefox stable_159.0（packages/puppeteer-core/src/revisions.ts） |
| 贡献者 | 445 人（gh api contributors 分页累计） |

## 2. 项目是什么，解决什么问题

Puppeteer 是 Google Chrome DevTools 团队维护的浏览器自动化库（README.md 自述"a JavaScript library which provides a high-level API to control Chrome or Firefox over the DevTools Protocol or WebDriver BiDi"）。它把底层协议消息封装成面向页面对象的 API：`launch()` 启动浏览器、`page.goto()` 导航、`page.click()`/`keyboard` 输入、截图/PDF 生成、请求拦截、Cookie 管理等，默认 headless 运行（README.md）。

它解决的核心问题是：让开发者不必手写 WebSocket 上的 JSON 协议报文、不必自己管理 target/session 生命周期，就能可靠地驱动浏览器。典型场景包括端到端测试、网页抓取、PDF/截图生成、性能录制（node/ScreenRecorder.ts）等。

项目从纯 CDP 驱动 Chrome 起家，近年最大的架构动作是引入 WebDriver BiDi 作为第二协议：Firefox 从某版本起已完全走 BiDi（node/BrowserLauncher.ts 第 156-167 行注释 "Default to 'webDriverBiDi' for Firefox"，且 Firefox 下强制选 cdp 会抛错），Chrome 也可通过 `protocol: 'webDriverBiDi'` 选项切换，同一套 Page/Browser API 屏蔽协议差异（test/TestSuites.json 中 firefox 套件参数均为 webDriverBiDi）。

`puppeteer` 与 `puppeteer-core` 双包设计：core 不附带浏览器下载逻辑入口，仅提供协议实现；`puppeteer` 包在其上增加"安装时下载 Chrome"与浏览器缓存管理（packages/puppeteer/src/node/PuppeteerNode.ts、@puppeteer/browsers 依赖）。README 明确给出 `npm i puppeteer`（带浏览器）与 `npm i puppeteer-core`（纯库）两种用法，并针对 npm/pnpm/Yarn/Bun 新版默认禁用 install script 的趋势提供了 `npx puppeteer browsers install` 手动兜底。

生态位上，Puppeteer 是 Chrome 自动化的"官方参考客户端"：devtools-protocol 类型直接生成自 CDP 规范（依赖 devtools-protocol 0.0.1704951），README 还推荐了基于它的 MCP 服务器 chrome-devtools-mcp，并实验性支持 WebMCP（api/Page.ts 引入 cdp/WebMCP.ts）。

## 3. 技术栈与依赖

- 语言与运行时：TypeScript 6.0（根 devDependencies），ESM（"type": "module"），Node >= 22.12.0（packages/puppeteer-core/package.json engines）；另产出浏览器端 es5/iife 包（puppeteer-core-browser，rollup + babel 构建）。
- 运行时依赖（packages/puppeteer-core/package.json）：
  - `devtools-protocol` 0.0.1704951：CDP 类型与协议映射（protocol-mapping.d.ts）
  - `chromium-bidi` 157.0.8090-0：Chromium 的 BiDi mapper 实现，BidiOverCdp.ts 直接调用其 BidiMapper.BidiServer
  - `webdriver-bidi-protocol` 1.0.1：BiDi 规范类型
  - `@puppeteer/browsers` 3.2.6：浏览器下载/安装/启动/缓存（packages/browsers 子包，含 CLI）
  - `ws` ^8.22.0：WebSocket 传输；`typed-query-selector`：querySelector 类型收窄
- 构建/工程链：wireit 增量任务（根 package.json）、hereby 任务器、api-extractor 生成单文件类型 lib/types.d.ts、rollup 打 es5-iife、esbuild 编译注入脚本、prettier + eslint 9 + tsdoc lint。
- 第三方小库 vendor 进仓库（puppeteer-core/third_party/）：rxjs、mitt、parsel-js（P 选择器解析）、urlpattern-polyfill，均带上游 LICENSE（third_party/README.md）。
- 测试栈：自研 mocha-runner（tools/mocha-runner）跑集成测试，node:test 跑包内单元测试，tsd 做类型测试，expect/sinon 做断言与打桩。

## 4. 目录结构与代码组织

npm workspaces 单仓多包（根 package.json "workspaces"）：

| 目录 | 职责 |
| --- | --- |
| packages/puppeteer-core | 协议核心库，双协议实现，约 217 个 ts 文件 |
| packages/puppeteer | 发布主包，叠加浏览器下载与 Node 启动配置（getConfiguration.ts） |
| packages/browsers | @puppeteer/browsers：浏览器下载/安装/启动/卸载，自带 CLI（src/CLI.ts、install.ts、launch.ts） |
| packages/ng-schematics | Angular CLI 集成脚手架 |
| packages/testserver | 测试用 HTTP 服务器 |
| test/ | 集成测试（mocha，91 个测试文件），TestSuites.json 定义 8 个套件，golden-chrome/golden-firefox 截图基线 |
| test/installation | 安装流程专项测试（npm run test-install） |
| tools/ | eslint 插件、docgen、mocha-runner、license 校验等内部工具 |
| website/ | Docusaurus 官网（pptr.dev），docs/ 为文档源 |
| examples/ | 官方示例：pdf、proxy、cross-browser、webdriver-bidi.mjs、puppeteer-in-browser/extension 等 |
| docker/ | headless shell 官方 Dockerfile |
| .github/workflows | 14 个工作流（见第 7/8 章） |

组织方式评述：核心库内部分层清晰——`api/`（与协议无关的抽象基类：Page 3622 行、Browser 948 行、ElementHandle、Realm、Locator 等 26 个文件）、`cdp/` 与 `bidi/`（两套平行的协议实现，各自有 Browser/BrowserContext/Page/Frame/ElementHandle/Connection），`common/`（共享基础设施：EventEmitter、BrowserConnector、CallbackRegistry、各 QueryHandler、Configuration）、`injected/`（注入页面的查询/轮询脚本，esbuild 编译进 src/generated，见 injected/README.md）、`node/`（启动器与 Node 传输层）。API 继承体系：api 层抽象类 + cdp/bidi 层具体实现（如 api/Page.ts:827 `export abstract class Page` 由 cdp/CdpPage、bidi/Page 实现，bidi/ 目录文件清单可证）。

## 5. 核心架构与关键实现

### 5.1 CDP 连接层（packages/puppeteer-core/src/cdp/Connection.ts）

`Connection extends EventEmitter` 管理一条浏览器级 WebSocket：内部 `#sessions = new Map<string, CdpCDPSession>()` 按 sessionId 维护多路复用的子会话，`#callbacks: CallbackRegistry` 将自增 id 与 pending Promise 配对，默认单命令超时 180_000ms（第 69 行）。cdp/CdpSession.ts 的 `CdpCDPSession.send()` 组装 `{sessionId, method, params}` 报文，会话关闭时报 "Protocol error (...): Session closed. Most likely the ... has been closed."（第 102 行）。协议类型直接来自 devtools-protocol 包的 `Protocol`/`ProtocolMapping`，保证与 CDP 规范零漂移（cdp/Connection.ts 第 7-8 行）。

### 5.2 双协议接入与选择（common/BrowserConnector.ts + node/BrowserLauncher.ts）

`puppeteer.connect()` 统一入口 `_connectToBrowser()`：先校验 browserWSEndpoint/browserURL/transport/channel 四选一，再按 `options.protocol === 'webDriverBiDi'` 分发到 `_connectToBiDiBrowser` 或 `_connectToCdpBrowser`（common/BrowserConnector.ts 第 39-55 行）。`channel` 路径会读取用户数据目录下的 DevToolsActivePort 文件推回 ws 端点（第 109-173 行）。启动路径同理：Firefox 缺省切到 BiDi，Chrome 保持 CDP（node/BrowserLauncher.ts 第 156-167 行）。

### 5.3 BiDi over CDP 桥（bidi/BidiOverCdp.ts）

Chrome 上的 BiDi 支持不依赖浏览器原生 BiDi 端点，而是把 chromium-bidi 的 BidiMapper 进程内嵌：`connectBidiOverCdp(cdp)` 用一个 NoOpTransport 把 Puppeteer 发出的 BiDi 命令喂给 `BidiMapper.BidiServer.createAndStart(transportBiDi, cdpConnectionAdapter, ...)`，BidiServer 再通过 CdpConnectionAdapter 把 BiDi 翻译成 CDP 命令；响应沿原路回传（第 21-68 行）。另在 bidi/Connection.ts 定义扩展命令 `goog:cdp.sendCommand`/`getSession`/`resolveRealm`（第 2-20 行），并在 emit 前递归剥离 `goog:` 前缀字段保持事件符合 W3C 面（第 91-103 行附近）。

### 5.4 注入脚本与选择器引擎（src/injected/）

页面内查询逻辑以纯 JS 写在 injected/ 下：CSSSelector、TextQuerySelector、XPathQuerySelector、ARIAQuerySelector、PierceQuerySelector、以及基于 parsel-js 的 PQuerySelector（Puppeteer 自有 `p-` 选择器：`text=`、`xpath=`、`aria=`、`>>` 组合等，解析器在 common/PSelectorParser.ts）。构建时用 esbuild 把每个脚本转译进 src/generated 再由服务端代码 import（injected/README.md、tools/generate_sources.ts），这样注入内容不依赖网络传输且可内联 ScriptInjector（common/ScriptInjector.ts）。等待/轮询由 injected/Poller.ts 提供多种策略。

### 5.5 Page 抽象与响应式状态（api/Page.ts）

抽象 Page 类（827 行声明起，共 3622 行）内置 rxjs：`#inflight$ = new ReplaySubject<number>(1)`（第 848 行）以响应式流跟踪进行中请求数，用于导航等待网络空闲等语义；超时统一交给 common/TimeoutSettings.ts。这解释了为何 rxjs 被 vendor：页面状态订阅是框架级依赖而非工具点缀。

## 6. 构建与运行

来自根 package.json scripts 与 wireit 定义（均未实际运行，仅静态确认）：

```bash
npm i                 # postinstall 逐 workspace 执行；puppeteer 包会下载钉版浏览器
npm run build         # wireit 编排：browsers/ng-schematics/puppeteer-core/puppeteer/testserver/test 全量构建
npm run unit          # 各 workspace 包内 node:test 单测（lib/**/*.test.js）
npm test              # 自研 moocha-runner 跑集成测试，套件由 test/TestSuites.json 定义
npm run test:chrome   # = chrome-bidi + headful + headless + headless-shell + pipe 五套件
npm run test:firefox  # firefox-headless/headful（webDriverBiDi）
npm run test-install  # 安装流程专项
npm run test-types    # tsd -t packages/puppeteer 类型测试
npm run docs          # api-extractor + docgen 生成 pptr.dev 文档
node packages/browsers/dist/cjs/main-cli.js   # @puppeteer/browsers CLI（源码 src/CLI.ts）
```

运行示例：`npm i puppeteer && node`，`const b = await puppeteer.launch(); const p = await b.newPage(); await p.goto(url)`（README.md Example 段）。

## 7. 测试与工程质量

- 集成测试规模：test/src 下 91 个 .ts 测试文件，覆盖 accessibility、cookies、drag-and-drop、oopif（OOPIF）、proxy、requestinterception、screenshot 等；按浏览器+协议+模式组合成 8 个套件（test/TestSuites.json）：chrome 四种形态走 CDP，firefox 两种形态与 chrome-bidi 走 BiDi——同一测试集在双协议上跑，是 API 一致性的直接证据。
- 期望管理：test/TestExpectations.json 与 CanaryTestExpectations.json 显式记录已知失败，`npm run lint:expectations` 强制排序（根 wireit），测试方差治理机制化。
- 单元测试与源码同目录（*.test.ts，如 cdp/Browser.test.ts、bidi/HTTPHTTPRequest.test.ts、common/EventEmitter.test.ts），用 node:test 运行器。
- 截图回归：golden-chrome/、golden-firefox/ 双基线目录。
- CI：.github/workflows 共 14 个——ci.yml（构建+lint+分套件测试）、daily.yml（每日跑）、deflake.yml（去噪重跑）、bisect.yml（提交二分）、devtools.yml（与 DevTools 前端联动）、update-browser-pins.yml（浏览器钉版自动滚动）、release-please.yml + publish.yml + pre-release.yml（发布链）、conventional-commit.yml（提交规范检查）、scorecards-analysis.yml（供应链安全评分）。工程质量在同类项目里属第一梯队。

## 8. 社区与维护状态

- 活跃度极高：gh api 显示 pushed_at 2026-10-30（分析前一天）；最新 release 25.16.0 发布于 2026-10-22（packages/puppeteer-core/CHANGELOG.md），该版本含 20+ 条 fix，且持续滚动 Chrome 156 / Firefox 158/159 支持。
- 发布节奏：release-please 按 conventional commits 自动出 PR 与 per-package CHANGELOG（release-please-config.json、.github/workflows/release-please.yml）；历史合并 CHANGELOG 有 306 个版本条目（根 CHANGELOG.md），平均每月 2-3 个 minor/patch。
- 贡献者 445 人（gh api 分页累计），由 Google Chrome DevTools 团队主导（LICENSE 版权行与 chromium-bidi 同源可佐证），社区 PR 通过 conventional-commit + stale 机器人管理。open_issues 214（含 PR，gh api）。
- 文档：pptr.dev 由 website/ Docusaurus 源码与 tools/docgen 从 TSdoc 注释生成，API 文档与代码同步发布。

## 9. 亮点与风险

亮点：
1. 双协议架构清晰：api 抽象层 + cdp/bidi 平行实现，同一套集成测试在 CDP 与 BiDi 上跑，Firefox 支持是协议层成果而非补丁（test/TestSuites.json、node/BrowserLauncher.ts）。
2. BiDi over CDP 内嵌 mapper 的桥接设计（bidi/BidiOverCdp.ts），让 Chrome 用户无需浏览器原生 BiDi 端点即可前向迁移。
3. 协议类型零手工维护：devtools-protocol 生成类型直接驱动 cdp/ 层，并有 tools/ensure-correct-devtools-protocol-package.mts 校验一致性（puppeteer-core wireit check）。
4. 测试工程标杆：8 套件矩阵 + expectations 文件 + deflake/bisect 专职工作流 + 双浏览器截图基线。
5. 工具链产品化：@puppeteer/browsers 独立成包并提供 CLI，浏览器版本钉版/滚动自动化（update-browser-pins.yml）。

风险/局限：
1. 强绑定 Chromium 节奏：Chrome 钉版随版本滚动，安装脚本被新型包管理器默认拦截是现实摩擦点（README 已专门写 note）。
2. Firefox 能力面仍以 BiDi 支持度为界，部分 CDP 独有功能（如部分 DevTools 特有域）在 Firefox 上不可用；具体差异面未逐一验证（未确认）。
3. 核心抽象 Page 单文件 3622 行、Browser 948 行，api/cdp/bidi 三处同名类并存，新人读源码需先建立"抽象类→实现类"映射才能跟上。
4. vendor 依赖（third_party/rxjs 等）需随上游手动同步，存在滞后风险（third_party/README.md）。

## 10. 总结评价

Puppeteer 是浏览器自动化领域的奠基者之一，如今在保持"Chrome 官方推荐客户端"地位的同时完成了向 WebDriver BiDi 的战略转身：api 抽象层把协议差异压在 cdp/ 与 bidi/ 两个平行实现里，测试矩阵确保两条路径行为一致，这种"协议可替换"的架构是它区别于 Playwright 等竞品的最大技术看点。项目由 Google 团队全职维护，release 节奏、测试基建、供应链安全（scorecards、license 校验）都达到工具库的工业级水准。适合做端到端测试、爬虫、页面渲染服务（PDF/截图）的工程师直接使用；想深入浏览器协议或学习大型 TS 库分层设计（抽象 API / 多协议实现 / 注入脚本 / wireit monorepo）的开发者，packages/puppeteer-core 的 api→cdp/bidi→common 三层源码非常值得通读，建议从 common/BrowserConnector.ts（协议分叉点）与 cdp/Connection.ts（会话多路复用）入手。
