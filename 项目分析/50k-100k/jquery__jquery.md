# jquery/jquery 项目分析

> 星数 59908（快照 2026-10-24）｜主语言 JavaScript｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：改变过整个 Web 的 JavaScript DOM 操作库——链式 API + 跨浏览器兼容层的开创者，2026 年初刚发布 4.0 大版本，仍是最广泛部署的 JS 库。

| 项目 | 值 |
| --- | --- |
| 仓库 | jquery/jquery |
| 星数 / Forks | 59908 / 约 20676（gh api 快照 2026-11-02） |
| 主语言 | JavaScript（纯 JS，源码 138 个文件、约 10547 行，`src/`） |
| 许可证 | MIT（`LICENSE.txt`） |
| 当前版本 | 4.0.0（2026-01-18 发布，`package.json` version 字段与 changelog.md） |
| 默认分支 | main |
| 最近 push | 2026-11-02（gh api），持续活跃 |
| 归属 | OpenJS Foundation，作者署名 "OpenJS Foundation and other contributors"（`package.json`） |

## 2. 项目是什么，解决什么问题

jQuery 是一个客户端 JavaScript 库，核心价值是三件事：统一的 DOM 查询与操作 API、跨浏览器兼容层、链式调用编程模型。它诞生于 2006 年（`AUTHORS.txt` 第一位作者即创始人 John Resig），在原生 API 碎片化的年代通过 `$()` 选择器、`.on()` 事件委托、`$.ajax()` 等接口大幅降低了前端开发门槛，长期占据"全球部署量最大的 JS 库"位置。

4.0 版本（`changelog.md`、README 版本支持表）标志着项目进入维护型现代化阶段：放弃 IE11 以下、iOS <11、Firefox <65、Edge Legacy 等旧浏览器（changelog Core 节 "Drop support for IE <11, iOS <11, Firefox <65..."），删除大量 IE workaround 与废弃 API（`src/deprecated/` 仅保留过渡期方法），内部从 AMD 全面迁移到 ES Modules（changelog "Migrate from AMD to ES modules 🎉 (d0ce00cd)"），并新增 TrustedHTML 基础支持、`$.parseHTML` 改用 `DOMParser` 等安全相关改进。

README 的版本支持表明确了分支策略：4.x（main）全量支持，3.x 仅关键修复，1.x/2.x 停止支持，旧版本商业支持外包给 HeroDevs/TuxCare。这说明 jQuery 的定位已从"前沿工具"转为"存量基础设施的可靠维护方"——它仍然被海量未重写的存量网站依赖，项目的首要职责是稳定与安全。

尽管现代框架（React/Vue 等）接管了新项目，jQuery 依然解决真实问题：无需构建链的渐进增强页面、WordPress 等 CMS 生态、以及任何想用 30KB 内（slim 构建）脚本搞定 DOM 的场景。它也是"API 设计如何影响一代开发者"的活教材。

## 3. 技术栈与依赖

- 语言：JavaScript（ES2020+ 语法，`package.json` 声明 `"type": "module"`，源码用原生 ESM import 组织）
- 运行时：浏览器为主，兼支持 Node（含 `dist-module/wrappers/jquery.node-module-wrapper.js` 的 Node 适配）与浏览器扩展等非浏览器环境（README "Environments in which to use jQuery"）
- 构建：Rollup 打包（`build/tasks/build.js`），@swc/core 压缩（`build/tasks/minify.js`），自研构建命令 `build/command.js` 支持模块裁剪、`--esm`/`--factory`/`--slim`/`--amd` 等参数
- 运行时依赖：**零**——`package.json` 无 dependencies 字段，jQuery 完全自包含
- 开发依赖（`package.json` devDependencies）：eslint 9 + eslint-config-jquery（代码风格）、qunit 2.26（测试框架）、jquery-test-runner（jtr，自研浏览器测试运行器）、jsdom（无头测试）、sinon（mock）、release-it（发版）、commitplease + husky（提交规范）、webpack（bundler 冒烟测试）、promises-aplus-tests（Deferred 合规测试）
- CI 平台：GitHub Actions（Node 测试矩阵、浏览器矩阵、BrowserStack 云真机、CodeQL 安全扫描）

## 4. 目录结构与代码组织

顶层目录（tarball 快照，HEAD = main）：

| 目录/文件 | 职责 |
| --- | --- |
| `src/` | 全部库源码，138 个 ES 模块文件（约 1.05 万行），按功能域分目录 |
| `build/` | 构建系统：`command.js` 入口、`tasks/`（build/dist/minify/npmcopy）、`release/`（发版自动化脚本） |
| `test/` | 测试：`unit/`（24 个 QUnit 文件约 3 万行）、`data/`（83 个 fixture/mock 文件）、`bundler_smoke_tests/`、`node_smoke_tests/`、`middleware-mockserver.cjs` |
| `dist/`、`dist-module/` | 构建产物目录，仓库只跟踪其中 `package.json` 与 `wrappers/`（产出的 JS 被忽略，发布时生成） |
| `.github/workflows/` | 9 个 CI workflow（node.js、browser-tests、browserstack、codeql、verify-release、lock-threads 等） |
| `changelog.md` | 4.0.0 完整变更记录（按模块分节，逐条附 commit 链接） |
| `AUTHORS.txt` | 373 位贡献者，按首次贡献排序 |
| `jtr-isolate.yml` | 测试模块隔离配置（每个 unit 模块独立 run，防止互相污染） |
| `eslint.config.js` | ESLint 9 flat config |

组织方式评述：单包单体库，但内部模块化程度极高。`src/` 采用"每方法一文件"的细粒度拆分（如 `src/core/ready.js`、`src/event/trigger.js`、`src/ajax/jsonp.js`），顶层一个入口文件（`src/jquery.js`）按序 import 全部模块。这个结构直接服务于构建系统的按模块裁剪能力：`build/tasks/build.js` 中的 `removeWith` 依赖图声明了"排除某模块时连带排除哪些下游模块"（如排除 `css` 连带排除 `effects/dimensions/offset`），用户可据此生成只含所需功能的定制构建。4.0 将 v3 时代的 AMD 异步模块定义切换为标准 ESM（`src/jquery.js` 全部为 `import` 语句，尾部 `export { jQuery, jQuery as $ }`），是架构上最大的一次进化。

## 5. 核心架构与关键实现

### 5.1 jQuery 对象与链式调用的核心机制（src/core.js + src/core/init.js）

整个库的基石是一个只有两行的工厂函数 + 原型共享技巧（`src/core.js:19-23`）：

```js
// Define a local copy of jQuery
jQuery = function( selector, context ) {

    // The jQuery object is actually just the init constructor 'enhanced'
    // Need init if jQuery is called (just allow error to be thrown if not included)
    return new jQuery.fn.init( selector, context );
};
```

`jQuery.fn.init`（`src/core/init.js`）是真正的构造器，按入参类型分流：空值返回空集合、DOM 节点直接包装、函数转为 ready 回调、HTML 字符串经 `isObviousHtml` 快速路径 + `rhtmlOrId` 正则区分 `$(html)` 与 `$("#id")`（含 trac-9521 注释的 XSS 防护：优先按 `#id` 处理）、选择器字符串转交 `find()`。关键点在 `init.prototype = jQuery.fn`（init.js 尾部）：所有实例共享同一原型，而 `pushStack()`（core.js）通过 `ret.prevObject = this` 把每次筛选的新集合与旧集合串成链，这就是 `.end()` 回退与无限链式调用的实现基础。v4 还删掉了原型上的 push/sort/splice 私有副本，改用数组方法借用（changelog Core 节）。

### 5.2 选择器引擎：Sizzle 遗产的自托管形态（src/selector.js + src/selector/）

v4 的选择器引擎是历史上独立仓库 Sizzle 的直接继承者，3.x 起内联进主仓库。`src/selector.js`（约 1400 行）实现了完整的 tokenize → compile → select 管线：`tokenize.js` 把选择器串解析为 token 序列，`compile()`（selector.js:1267）将其编译为匹配函数并缓存于 `compilerCache`，`select()`（selector.js:1308）负责执行；`classCache`/`nonnativeSelectorCache` 三个 `createCache()` 缓存热点。引擎在支持 QSA 的浏览器上优先走 `querySelectorAll` 快速路径（selector.js 中多处 QSA 分支与 `rbuggyQSA.js` 的浏览器缺陷检测），只对 jQuery 扩展伪类（`:even`/`:odd` 等）与缺陷选择器回退自研实现——v4 甚至把 `:even`/`:odd` 从选择器层移除、改为 `.even()`/`.odd()` 方法（changelog）。尾部保留了 Sizzle 时代的兼容出口（selector.js:1388-1395）：

```js
jQuery.find = find;

// These have always been private, but they used to be documented as part of
// Sizzle so let's maintain them for now for backwards compatibility purposes.
find.compile = compile;
find.select = select;
find.setDocument = setDocument;
find.tokenize = tokenize;
```

此外 `src/selector-native.js` 提供一个纯 `querySelectorAll` 的轻量替身，构建时 `--exclude=selector` 即可换装（README "Building a Custom jQuery"），使最小构建体积骤减。

### 5.3 事件系统：data 私有存储 + 全局委托（src/event.js）

`src/event.js`（884 行，v4 最大的单模块）不用元素属性存事件，而是通过 `dataPriv`（`src/data/var/dataPriv.js`，基于 `src/data.js` 的安全数据存储）把处理程序挂在元素与 `jQuery.expando` 键上，避免直接扩充 DOM 节点造成循环引用泄漏。入口 `on()`（event.js:27 起）支持 selector 参数——这是 jQuery 事件委托的核心 API：事件绑定在容器上，`dispatch` 时沿 DOM 树向上匹配 delegate 目标。`src/event/trigger.js` 另实现了程序化触发（含模拟冒泡与 triggerHandler）。这份设计是"为什么 jQuery.data 是事件系统前置依赖"的最好注解，也是理解jQuery 内存管理模型（`empty()`/`remove()` 时清理 data 与事件）的入口。

### 5.4 Deferred/Promise 与模块裁剪依赖图（src/deferred.js + build/tasks/build.js）

`src/deferred.js`（392 行）是 jQuery.Promises/A+ 合规实现，配套 `build/tasks/promises_aplus_tests.js` 用 promises-aplus-tests 套件强制合规（devDependencies 可证）。它向下支撑 ajax、effects、queue、core/ready 等模块，因此构建系统专门处理了这条依赖链：`build/tasks/build.js` 的 `removeWith` 图（见第 4 节）在排除 `deferred` 时移除 ajax/effects/queue 并用 `core/ready-no-deferred.js`（`src/core/ready-no-deferred.js`）顶替 ready 实现——slim 构建正是利用该机制剔除 callbacks/deferred（changelog "Exclude callbacks & deferred modules in the slim build as well"）。

## 6. 构建与运行

来自 README 与 `package.json` scripts 的实际命令：

```bash
npm install          # 安装开发依赖
npm run build        # 构建标准 jquery.js 到 dist/（含 min + sourcemap）
npm run build:all    # 构建全部发布变体：jquery.js / jquery.slim.js /
                     #   jquery.module.js / jquery.slim.module.js + 压缩与 map
npm start            # watch 模式，改源码自动重建

# 定制构建（排除模块生成精简版）
npm run build -- --exclude=ajax,css,deprecated
npm run build -- --slim --esm --factory --filename=jquery.factory.slim.module.js

# 测试（全套）
npm test             # build:all + lint + browserless + browser + esm + slim
                     #   + no-deprecated + selector-native 七连测
npm run test:browserless   # jsdom + Node 冒烟 + promises-aplus（无需真浏览器）
npm run test:browser       # chrome/firefox headless 真机测试
npm run test:jsdom        # jsdom 单独跑 basic 模块

# lint
npm run lint         # eslint --cache + package.json jsonlint 并发执行
```

测试需要本地服务器：v4 用自研 `jquery-test-runner (jtr)` 配 `test/middleware-mockserver.cjs`（Node 中间件 mock 服务器，替代旧 PHP 方案），README 中旧的 PHP/MAMP 指引尚未清理，属文档滞后。

## 7. 测试与工程质量

- 规模：`test/unit/` 24 个 QUnit 测试文件共约 30039 行，与 `src/` 约 1:2.9 的测试代码比；`test/data/` 83 个文件提供 HTML fixture 与 mock.php 后端桩（`test/` 目录实测）。
- 分层：单元测试（QUnit，按 src 模块一一对应：core/selector/event/ajax/deferred…）+ 变体测试（test:slim / test:no-deprecated / test:selector-native / test:esm，保证每种官方构建变体都过全量测试）+ bundler 冒烟测试（`test/bundler_smoke_tests/`，用 webpack/jsdom 验证 CJS/ESM 互操作，对应 package.json exports 修复 #5416）+ Node 冒烟测试（`build/tasks/node_smoke_tests.js`）+ Promise 合规测试。
- 浏览器矩阵：`.github/workflows/browser-tests.yml` + `browserstack.yml` 走 BrowserStack 云真机，覆盖 Chrome/Firefox/Edge/Safari（package.json 中逐浏览器 test 脚本佐证）；Node 矩阵为 20.x/22.x/24.x（`.github/workflows/node.js.yml`）。
- 工程质量：eslint 9 flat config + eslint-config-jquery 统一风格；husky + commitplease 强制提交信息格式（package.json 中 commitplease.components 列出 23 个模块名，即提交前缀规范，与源码模块一一对应）；CodeQL 安全扫描（`.github/workflows/codeql-analysis.yml`）；`verify-release.yml` 发版后校验。测试隔离通过 `jtr-isolate.yml` 按模块拆 run。

## 8. 社区与维护状态

- 活跃度：gh api 显示最近 push 2026-11-02（本次分析前一天），open issues 99——低 issue 存量反映项目处于稳定维护态而非开发态。
- 发布节奏：4.0.0-beta（2024-02）→ beta2（2024-07）→ rc1（2025-08）→ rc2/4.0.0（2026-01-18，gh api releases 数据），主线大版本历时约两年打磨；3.x 分支转入 critical-only 维护（README 版本支持表）。
- 贡献者结构：contributors API 分页推算约 348 人；`AUTHORS.txt` 记录 373 位按首次贡献排序的历史作者，头部为 John Resig。项目由 OpenJS Foundation 托管，核心团队通过 matrix.org 周会决策（README 首段与 meetings.jquery.org 链接）。
- 流程：发版全自动化（`build/release/` 含 authors/cdn/changelog/dist/verify 五个脚本与 pre/post-release.sh，配合 release-it）；`.github/dependabot.yml` 自动升级依赖；`lock-threads.yml` 自动锁定旧讨论串；安全问题走 security@jquery.com 私密报告（`SECURITY.md`）。

## 9. 亮点与风险

亮点：

1. 极致的模块化与裁剪能力：src/ 每方法一文件 + 构建期依赖图（`build/tasks/build.js` removeWith），可生成从完整版到仅剩 core 的任意组合，v4 还新增 factory 构建支持无 window 环境。
2. 双格式与互操作成熟：package.json exports 按 node/module/import/default 四条件分发 CJS/ESM 产物，并用真实 webpack/jsdom 冒烟测试护航（`test/bundler_smoke_tests/`），解决生态最头疼的双包问题。
3. 测试体系堪称教科书：3 万行 QUnit 单测 + 六种构建变体全量回归 + 云真机矩阵 + Promise 合规套件 + CodeQL。
4. 零运行时依赖、约千行核心即含完整链式 API，代码注释详尽（含 trac/GitHub issue 编号溯源），是 JS 库设计的活标本。
5. 维护流程高度自动化：authors/changelog/CDN 同步/发布校验全部脚本化（`build/release/`）。

风险/局限：

1. 生态位收缩：新项目已被现代框架与原生 API 取代，jQuery 的主要受众是存量系统，创新几乎停滞（4.0 均为减法与现代化而非新能力）。
2. README 构建文档滞后：测试一节仍要求 PHP 本地服务器（README "Running the Unit Tests"），与 v4 实际的 Node mockserver（`test/middleware-mockserver.cjs`）不符，新人上手易困惑。
3. `dist/` 产物不入库，GitHub 上看到的仓库与 npm 包内容不一致；直接引用 GitHub 路径的旧用法会失效。
4. 3.x/1.x/2.x 海量存量部署构成长期安全维护负担，官方仅 critical-only 支持，旧版用户需商业支持或自行升级。

## 10. 总结评价

jQuery 是 Web 开发史上影响力最大的 JavaScript 库，如今以 4.0 为起点完成了从"开拓者"到"可靠存量基础设施"的转身：源码完成 AMD→ESM 现代化，砍掉 IE 包袱后代码更干净，构建、测试、发布全链路自动化程度极高，零依赖双格式发布依旧专业。它不再是新项目的技术选型，但对三类人仍有价值——维护存量 jQuery 代码的工程师（4.0 升级指南价值）、想学习"小而美 API 设计 + 工程化测试"的库作者（src/core.js 的 init/prototype 技巧、src/event.js 的 data 委托模型、build/tasks/build.js 的裁剪依赖图都值得精读）、以及研究前端演化史的开发者。读它的源码像读一本注释完备的 JavaScript 工程教科书。
