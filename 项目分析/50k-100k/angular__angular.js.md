# angular/angular.js 项目分析

> 星数 58475（快照 2026-10-24）｜主语言 JavaScript｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：Google 出品的 AngularJS 1.x 前端框架本体——第一个把"双双向数据绑定 + 依赖注入 + 指令扩展 HTML"做成完整体系的主流 MV* 框架，是现代前端工程化的奠基者之一；已于 2022-01-06 正式 EOL，仓库 2024 年起归档只读。

| 事实 | 值 |
|---|---|
| 名称 | angular/angular.js |
| 星数 | 58,475（快照 2026-10-24） |
| 主语言 | JavaScript（ES5，全量源码约 15.3 万行，见 `src/`） |
| 许可证 | MIT（`LICENSE`，Copyright 2010-2020 Google LLC） |
| 创建时间 | 2010-01-06（GitHub API `created_at`） |
| 最终版本 | 1.8.3 "ultimate-farewell"（2022-04-07，见 `CHANGELOG.md`） |
| 最近 push | 2024-04-12（GitHub API `pushed_at`），仓库 `archived: true` |
| 默认分支 | master |

## 2. 项目是什么，解决什么问题

AngularJS 的自我定位是"让浏览器变聪明"：把普通 HTML 当模板语言，通过指令（directive）扩展 HTML 语法，用双向数据绑定自动同步视图与模型，用依赖注入（DI）实现控制反转，从而让客户端应用结构化、可测试（`README.md` 开篇即如此陈述）。在 2010 年前后的 jQuery 时代，这是范式级的转变——开发者从"手动操作 DOM"转向"声明式描述 UI 与状态的关系"。

它解决的四个核心问题：其一，数据绑定——模型变了视图自动更新、反之亦然，靠 scope 上的脏检查循环实现；其二，HTML 扩展——`ng-repeat`、`ng-model`、自定义元素/属性等指令让 UI 结构声明式可读；其三，可测试性——内建 DI 容器让服务可以注入 mock，`ngMock` 模块直接为单测而生；其四，应用骨架——路由（ngRoute）、表单校验、AJAX 封装（$http）、i18n、动画、触摸等全家桶模块开箱即用（模块清单见 `src/AngularPublic.js` 与 `src/` 下各 `ng*` 目录）。

README 中"MV\* done the right way"、"less than 10 lines of code"等表述（`README.md` 53-109 行）反映了它当年的推广叙事。今天读这个仓库的价值在于：它是"框架设计如何影响一代开发者"的活化石——脏检查的取舍、DI 的实现方式、指令系统的 scope/编译模型，全都成了后续框架（包括 Angular 2+、Vue 的响应式与模板指令）反复对照与批判的参照系。

生命周期：2010-01-06 创建仓库，2012 年发布 1.0，2014 年宣布 Angular 2 走完全不同的路线，1.x 按承诺维护到 2021-12-31 LTS 结束，2022-01 官宣 EOL，最终以 1.8.3 "ultimate-farewell" 收官（`CHANGELOG.md`），2024-04 最后一次 push 后归档只读。

## 3. 技术栈与依赖

- **运行时零依赖**：`package.json` 中 `"dependencies": {}`——框架本体是纯 ES5 JavaScript，不依赖任何第三方库。
- **jqLite**：自带一个精简版 jQuery（`src/jqLite.js`，约 1176 行），有 jQuery 时可切换为其封装；测试矩阵覆盖 jqlite / jQuery 2.1 / 2.2 / 3.5.1（多个 `karma-jquery*.conf.js`）。
- **构建工具链（devDependencies，`package.json`）**：Grunt 为主构建系统（`Gruntfile.js`，约 500+ 行），Gulp 辅助 minify/sourcemap，Yarn 锁定依赖（`yarn.lock`，engines 要求 node>=12.14.1）。
- **测试链**：Karma + Jasmine 2.x（单元）、Protractor 7 + Selenium（e2e）、jasmine-node（i18n 测试，`package.json` scripts）。
- **文档系统**：dgeni + dgeni-packages 生成的文档站点（`docs/` 目录，含自己的 gulpfile 与 protractor 配置）。
- **其他**：promises-aplus-tests 用于 $q 的规范符合性测试；closure-compiler 在 `vendor/` 下用于产出压缩版（另有 `README.closure.md` 说明）。

## 4. 目录结构与代码组织

顶层目录（单仓单包 monolith，无子包拆分）：

| 目录/文件 | 职责 |
|---|---|
| `src/` | 框架全部源码（约 15.3 万行，含 13 个可选模块） |
| `src/ng/` | 核心模块：scope、compile、http、q、parse、location、30 个内置指令等（70 个文件，约 3.1 万行） |
| `src/auto/` | 自动注入器（injector.js，约 975 行） |
| `src/ngAnimate/` `src/ngAria/` `src/ngCookies/` `src/ngMessages/` `src/ngResource/` `src/ngRoute/` `src/ngSanitize/` `src/ngTouch/` `src/ngLocale/` `src/ngMessageFormat/` `src/ngParseExt/` | 可选功能模块，按 `angular.module('ngX').requires` 挂载 |
| `src/ngMock/` | 单测 mock 模块（$httpBackend、$timeout 假实现等） |
| `src/ngComponentRouter/` | 1.6 时代实验性组件路由，从未正式发布（仅 `Router.js` 等少量文件） |
| `test/` | 与 src 镜像的单元测试（约 10 万行）+ `test/e2e/` 端到端测试 |
| `i18n/` | 8 个地区化文件的生成与 spec |
| `docs/` | dgeni 文档站生成配置与内容 |
| `benchmarks/` | benchpress 基准测试（largetable、ng-class、orderby 等场景） |
| `lib/` `scripts/` `vendor/` | 构建辅助脚本、closure 编译器等 |
| `.circleci/` | CircleCI 2.1 流水线配置 |

组织方式评述：典型的"核心 + 插件式模块"单仓结构。`src/loader.js`（约 413 行）实现模块注册器（`angular.module(name, requires, configFn)`），`src/AngularPublic.js` 汇出公开 API，`src/module.prefix/suffix` 拼装 IIFE——构建产物按模块切片，用户按需引入。代码风格为带大段 API 文档注释的 ES5（文档注释本身就是 dgeni 的文档源），每个核心文件自带 minErr 错误工厂（`src/minErr.js`）生成带链接的错误码。

## 5. 核心架构与关键实现

### 5.1 scope 脏检查（`src/ng/rootScope.js`，约 1500 行）

AngularJS 的心脏：`$digest` 遍历 scope 树上所有 watcher，反复求值直到稳定或超过 TTL（默认 10，`rootScope.js` 第 73 行）。关键优化有三处：

- **`lastDirtyWatch` 短路**：上一轮最后变脏的 watcher 如果本轮已干净，说明其后的 watcher 均已测过，直接退出整轮循环（`rootScope.js` 841-846 行）；
- **子树剪枝**：scope 维护 `$$watchersCount`，无 watcher 的子树整棵跳过（859 行的遍历条件）；
- **TTL 熔断**：10 次迭代仍不稳定则抛 `infdig` 错误并列出最后 5 次迭代的脏 watcher（869-875 行）。

```js
// src/ng/rootScope.js $digest 内核（节选，821-831 行）
get = watch.get;
if ((value = get(current)) !== (last = watch.last) &&
    !(watch.eq
        ? equals(value, last)
        : (isNumberNaN(value) && isNumberNaN(last)))) {
  dirty = true;
  lastDirtyWatch = watch;
  watch.last = watch.eq ? copy(value, null) : value;
  fn = watch.fn;              // listener
  fn(value, ((last === initWatchVal) ? value : last), current);
```

对比基于值的 `===` 优先、失败才走深度 `equals`，是典型的"原始值快路径"设计。这套"变更不可知、每次全量轮询"的模型正是后来响应式框架（observable/signal 式细粒度推送）所要取代的对象。

### 5.2 依赖注入（`src/auto/injector.js`，约 975 行）

DI 通过函数签名反射实现：`annotate` 用 `fn.toString()` 配合 `FN_ARGS`/`ARROW_ARG` 正则从参数表抠出服务名（66-93 行），支持内联数组注解 `['$http', fn]` 与 `$inject` 属性两种防混淆写法；`ngAnnotate` 构建期可自动补写。服务解析在 `createInternalInjector` 的 `getService` 中完成，用 `INSTANTIATING` 哨兵值检出循环依赖：

```js
// src/auto/injector.js getService（876-897 行，节选）
if (cache.hasOwnProperty(serviceName)) {
  if (cache[serviceName] === INSTANTIATING) {
    throw $injectorMinErr('cdep', 'Circular dependency found: {0}',
              serviceName + ' <- ' + path.join(' <- '));
  }
  return cache[serviceName];
}
```

模块加载顺序（config 块先于 run 块）、provider/service/factory/value/constant 五种服务配方都在 `createInjector`（706 行起）中归一为 `provider` 注册。这套"字符串 token + 单例缓存"的 DI 在当年几乎独一份，深刻影响了后来的 NestJS/Angular 2 的 DI 设计。

### 5.3 指令编译系统（`src/ng/compile.js`，约 4354 行，全仓最大单文件）

指令系统分两阶段：**compile 阶段**做纯 DOM 变换——`collectDirectives`（2694 行）按属性/元素/类名收集指令并按 priority 排序、处理 terminal 与 replace，`applyDirectivesToNode`（2926 行）为节点生成 `nodeLinkFn`，期间完成 transclusion 槽（`$$slots`，2673 行）与 isolate scope 绑定解析（`parseIsolateBindings`，含 `@`/`<`/`&`/`=` 四种绑定符）；**link 阶段**由 `publicLinkFn`（2428 行）/`compositeLinkFn`（2573 行）对克隆节点执行 scope 接线，pre-link 先父后子、post-link 先子后父。`$compile` 是整个框架中最复杂的设计，isolate scope + bindToController 也是 1.x 后期向组件化（`component()` API）演进的桥梁。

### 5.4 表达式解析（`src/ng/parse.js`，约 2017 行）

模板里的 `{{expr}}` 与指令表达式由手写的四层解析器处理：`Lexer`（63 行）词法分析 → `AST`（272 行）语法树 → `ASTCompiler`（797 行，编译为 JS 函数）或 `ASTInterpreter`（1263 行，CSP 模式降级解释执行）→ `Parser`（1637 行）门面，结果按表达式字符串缓存。解析器内建 `ensureSafeMemberName/ensureSafeFunction` 等"防窗户打脸"检查，禁止表达式访问 `window`、`Function`、构造器等危险出口——这是把安全边界做进表达式引擎的早期范例。

### 5.5 其他值得一提的机制

- **ngModel 双向绑定管线**（`src/ng/directive/ngModel.js`）：DOM 输入 → `$parsers` → `$validators`/`$asyncValidators` → model；model → `$formatters` → 视图，外加 `$render`/`$setViewValue` 生命周期——表单校验体系的原型。
- **$http 拦截器**（`src/ng/http.js`，376-463 行）：`interceptorFactories` 注册 request/response 错误四元组，promise 链式组合。
- **$q**（`src/ng/q.js`）：参考 Q 实现的精简 promise（~693 行），2.x 分支曾实验异步 digest 调度，1.8 稳定版仍为同步。
- **jQLite**（`src/jqLite.js`）：仅实现链式操作/事件/on 等必要子集，保证框架零依赖可跑。

## 6. 构建与运行

以下命令来自 `DEVELOPERS.md` 与 `Gruntfile.js`（本分析未实际运行构建）：

```bash
yarn install                 # 安装依赖（DEVELOPERS.md 68 行）
yarn grunt package           # 构建产物到 build/（DEVELOPERS.md 71 行）
yarn grunt webserver         # 开发服务器（DEVELOPERS.md 97 行）
yarn grunt test:unit         # Karma 单元测试（119 行），--browsers 可指定
yarn grunt test:e2e          # Protractor 端到端（176 行）
yarn grunt docs              # 构建 dgeni 文档站（322 行）
yarn grunt minify            # closure 压缩（Gruntfile.js 497 行）
yarn run commit              # commitizen 规范提交（235 行）
```

注意：`yarn grunt package` 在 Windows 上需要管理员权限（创建符号链接，`DEVELOPERS.md` 75 行）；构建需 node>=12.14.1（`package.json` engines）。

## 7. 测试与工程质量

- **规模**：`test/` 下 147 个 JS 文件、约 9.97 万行，其中 107 个 `*Spec.js`——测试代码行数与框架源码约 15.3 万行接近 1:1，覆盖密度极高。测试目录与 `src/` 严格镜像（`test/ng/`、`test/ngRoute/`……）。
- **风格**：Jasmine 2.x 单元测试为主，大量使用 `$provide`/`ngMock` 注入 mock 服务；e2e 用 Protractor（`test/e2e/`，含 fixtures 与独立工具）；`$q` 另跑 promises-aplus 规范测试（`Gruntfile.js` 493 行）；i18n 有独立的 jasmine-node 套件。
- **浏览器矩阵**：jqlite/jQuery 2.1/2.2/3.x 四套配置并行（多个 `karma-*.conf.js`），CI 支持本地 Chrome/Firefox + BrowserStack（BS_*）+ SauceLabs（SL_*）远程矩阵（`DEVELOPERS.md` 119-139 行），兼容目标包含 IE9-11——这解释了源码里大量的 `// Support: IE 9-11 only` 注释。
- **CI**：`.circleci/config.yml`（CircleCI 2.1），docker 镜像按 SHA 精确 pin（node:14.16.1），另有 `ci-checks`、`prepareDeploy` 等 Grunt 任务。
- **代码质量观察**：ES5 + 严格内联 API 文档（文档即源码的一部分）、eslint 多套配置（`.eslintrc-base/browser/node/todo.json`）、minErr 错误码体系、commitplease/cz-conventional-changelog 强约束提交格式。工程质量在同期项目中属顶配。

## 8. 社区与维护状态

- **贡献者**：约 1854 人（GitHub contributors 分页 Link 头 last=1855，含匿名贡献者，2026-11 查询）。
- **活跃度**：仓库 `archived: true`，最后 push 2024-04-12（API 元数据）；461 个 open issue 与约 2.7 万 fork 属于 EOL 项目的存量状态，不再处理。
- **Release 节奏（历史）**：1.0（2012）→ 1.2（2013）→ 1.3/1.4（2014，弃用 Promise 推荐库后转向）→ 1.5 组件化（2016）→ 1.6（2017）→ 1.7 最后一个大版本（2018）→ 1.8 LTS（2020）→ 1.8.3 "ultimate-farewell"（2022-04-07，`CHANGELOG.md`）。1.8.x 期间遵循 LTS：只修安全与重大 bug。
- **版本号标识**：`package.json` `branchVersion: ^1.8.0`、`branchPattern: 1.8.*`、`distTag: next`。
- 官方指引：README 与 CHANGELOG 顶部均引导用户迁移到 angular.io 的现代 Angular；API 文档站 docs.angularjs.org 仍在但不再更新。

## 9. 亮点与风险

**亮点**

1. 架构教科书级完整：scope 脏检查、DI、指令编译、表达式解析四大子系统环环相扣，且每处都有海量注释与测试对照（`src/ng/rootScope.js`、`src/auto/injector.js`、`src/ng/compile.js`、`src/ng/parse.js`）。
2. 工程实践领先时代：测试与源码约 1:1 的密度、BrowserStack/SauceLabs 多浏览器矩阵、文档即源码、conventional-changelog 全流程自动化，放在今天仍不过时。
3. 历史影响力巨大：DI、指令、双向绑定、组件化过渡（1.5 component）直接塑造了 Angular/Vue/后端 DI 框架的设计语言；读懂它等于读懂半个前端框架史。
4. 安全意识内建：表达式解析器的 sandbox 检查、$sce 信任资源体系、ngSanitize 模块（`src/ng/parse.js`、`src/ng/sce.js`）。

**风险/局限**

1. 已 EOL 且归档：2022-01 起无任何安全补丁，生产使用等于裸奔（README 明示，GitHub archived）。
2. 脏检查的性能天花板：watcher 数量线性影响每轮 digest 开销，深比较（`watch.eq`）代价更高，是"框架税"的原型案例。
3. 技术栈过时：ES5、IE9 兼容代码、Grunt/Gulp 工具链、 karma 4/jasmine 2 等均已退役，构建与运行需旧版 node/yarn（node>=12.14.1 也已 EOL）。
4. 学习成本陷阱仍在：transclusion 槽、isolate scope 绑定符、digest 时机等概念对新手极易误用（`compile.js` 内部复杂度即为佐证）。

## 10. 总结评价

AngularJS 1.x 是前端工程化时代的开创者：它在 jQuery 之上第一次完整回答了"大型客户端应用该如何组织"这个问题，用双向绑定、依赖注入与指令系统定义了一代人的开发范式，58k+ 的星数与约 1854 位贡献者是那段历史的注脚。如今它已按官方承诺体面谢幕（1.8.3 "ultimate-farewell"，2022-04），仓库归档只读，不再是可用性选项，但作为**源码阅读材料**价值极高——尤其推荐精读 `src/ng/rootScope.js` 的 $digest 短路优化、`src/auto/injector.js` 的循环依赖检测与 `src/ng/compile.js` 的两阶段编译，能同时看懂"这套设计为何伟大"与"它为何被响应式/细粒度更新范式取代"。适合框架作者、历史研究者与需要维护存量 AngularJS 应用的工程师阅读；新项目请直接使用 angular.io 的现代 Angular。
