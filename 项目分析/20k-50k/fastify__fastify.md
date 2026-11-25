# fastify/fastify 项目分析

> 星数 39500（快照 2026-10-24，gh api 2026-11-17 核实 37428）｜主语言 JavaScript（CommonJS，附完整 TS 类型声明）｜档位 20k-50k｜分析日期 2026-11-17

## 1. 项目速览

一句话定位：Node.js 生态主打"低开销 + 高性能 + 强插件架构"的 Web 框架，以 JSON Schema 为一等公民，自称受 Hapi 与 Express 启发（README.md）。

| 项目 | 值 | 出处 |
| --- | --- | --- |
| 名称 | fastify/fastify | gh api |
| 星数 | 39500（快照）/ 37428（2026-11-17 核实），forks 3103 | gh api |
| 语言 | JavaScript（GitHub 标注），`fastify.d.ts` + `types/` 提供完整 TypeScript 声明 | package.json, fastify.d.ts |
| 许可证 | MIT | package.json |
| 版本 | main 分支为 v6.0.0-alpha.6，稳定线 v5.12.x 双轨并行 | package.json, gh api releases |
| 最近活跃 | 最近 push 2026-11-16（分析前 1 天），极活跃 | gh api |
| 官网 | https://fastify.dev/ | package.json homepage |

## 2. 项目是什么，解决什么问题

Fastify 的出发点写在 README 开头："高效的服务器意味着更低的基础设施成本、负载下更好的响应与更快的用户"。它是针对 Express 的痛点（全局中间件链无封装隔离、校验/序列化无内建方案、日志无内建方案）的体系化回答，核心主张有三条（README.md、docs/Reference/Principles.md）：

1. **性能优先**：所有热路径尽量少做分配、少走抽象。README 提供的 autocannon 合成基准（autocannon -c 100 -d 40 -p 10）中 Fastify 77,193 req/s，高于裸 `http.Server` 的 74,513（即框架开销近乎为零）、Koa 54,272、hapi 42,284、Express 14,200（README.md L198-218）。该表方法学公开、并明确提示"请自行基准测试"，口径诚实，数字可信但属 hello-world 场景。
2. **JSON Schema 一等公民**：路由声明 schema 后，请求校验用 ajv 编译为函数、响应序列化用 fast-json-stringify 按schema 生成专用的 stringify 代码，省掉通用序列化的开销（package.json dependencies、docs/Reference/Validation-and-Serialization.md）。
3. **插件封装架构**：`register` 的每个插件获得一个原型继承自父实例的"封装上下文"，钩子、装饰器、内容类型解析器默认私有，可用 `fastify-plugin` 声明为穿透共享（lib/plugin-override.js、docs/Reference/Encapsulation.md）。这是它与 Express/Koa 最本质的架构差异。

与 Express/Koa/hapi 的架构差异小结：Express 是"全局中间件栈 + 无 schema"；Koa 是"洋葱模型 + 无封装边界"；hapi 有强大配置体系但生态偏封闭；Fastify 则是"显式生命周期钩子 + 层级封装上下文 + schema 编译"，把 hapi 的工程化与 Express 的易用性折中到一起，并靠 avvio 管理异步启动顺序。

## 3. 技术栈与依赖

- 语言与运行时：纯 JavaScript（CommonJS，`"type": "commonjs"`，package.json），Node.js 原生 `http`/`http2`、`diagnostics_channel`（lib/handle-request.js、fastify.js）；TypeScript 通过 `fastify.d.ts`（294 行）与 `types/`（15 个 .d.ts）支持。
- 运行时依赖（package.json dependencies）：
  - `avvio` ^9：异步插件启动/关闭序列管理器，fastify 封装体系的地基
  - `find-my-way` ^9：radix 路由树，支持参数、通配符与约束（版本/host）
  - `fast-json-stringify` ^7 + `@fastify/fast-json-stringify-compiler`：按 JSON Schema 生成序列化函数
  - `@fastify/ajv-compiler` ^4：ajv 校验编译器封装
  - `pino` ^9/^10：默认日志器（peer 型依赖）
  - `light-my-request` ^6：`inject()` 假 HTTP 注入测试
  - `secure-json-parse` ^4：防原型污染的 JSON 解析（默认 __proto__ 投毒保护）
  - `abstract-logging`、`process-warning`、`rfdc`、`semver`、`toad-cache`、`@fastify/proxy-addr`、`@fastify/error`（错误码工厂）
- 工具链：eslint 9 + neostandard（代码风格）、borp（node:test 运行器）、c8（覆盖率，CI 强制 100% 行覆盖）、tstyche（类型测试）、markdownlint-cli2（package.json scripts）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `fastify.js`（1043 行） | 唯一入口：`fastify()` 实例工厂，组装公共 API、安装 avvio |
| `lib/`（30 个文件，与入口合计 9459 行） | 核心实现：路由、钩子、请求/回复、封装、校验、日志、错误 |
| `types/`（15 个 .d.ts） | 按 API 面拆分的类型声明 |
| `test/`（196 个 .test.js，约 46000 行） | 单元/集成测试，node:test 编写 |
| `docs/Guides/`, `docs/Reference/`（21 篇） | 用户文档源码（发布到 fastify.dev） |
| `examples/` | 最小示例（simple、hooks、http2、benchmark 等 14 项） |
| `build/` | 生成校验器/错误序列化代码、版本同步的构建脚本 |
| `integration/` | 与打包器/运行时的集成测试 |
| `.github/workflows/`（20 个） | CI、覆盖、lint、生态链接校验等 |

组织方式评述：单包单体仓库（非 monorepo），核心库刻意保持极小（约 9.5k 行），把认证、静态文件、Swagger 等拆到 `@fastify/*` 组织下各自动仓库——fastify 主仓只留内核与最通用的能力。这是"内核小 + 生态 monorepo 化"的分布式架构。

## 5. 核心架构与关键实现

### 5.1 实例工厂与 Avvio 启动序列（fastify.js）

`fastify()` 不是类，而是工厂：创建 find-my-way 路由器（`buildRouting`，lib/route.js L77 `FindMyWay(options)`）、404 路由器、HTTP 服务器，然后组装一个纯对象字面量形式的公共 API（fastify.js L137-306），最后安装 avvio 并接管生命周期（fastify.js L384-397）：

```js
const avvio = Avvio(fastify, {
  autostart: false,
  timeout: ...,
  expose: { use: 'register' }
})
// Override to allow the plugin encapsulation
avvio.override = override
avvio.on('start', () => (fastify[kState].started = true))
fastify[kAvvioBoot] = fastify.ready
fastify.ready = ready // overwrite the avvio ready function
```

即 `register()` 实为 avvio 的 `use`，`ready()` 触发异步启动树（依赖顺序、可 await）；`preReady` 事件后一次性编译路由树（fastify.js L400-406），此后增删路由惰性重编译。

### 5.2 插件封装的"魔法"（lib/plugin-override.js）

封装的核心只有约 90 行：注册插件时用原型链生成子实例，而不是拷贝整个对象（lib/plugin-override.js L38-56）：

```js
const instance = Object.create(old)          // 原型继承父实例
instance[kContentTypeParser] = ContentTypeParser.helpers.buildContentTypeParser(instance[kContentTypeParser])
instance[kHooks] = buildHooks(instance[kHooks])   // 钩子数组浅拷贝隔离
instance[kRoutePrefix] = buildRoutePrefix(instance[kRoutePrefix], opts.prefix)
instance[kSchemaController] = SchemaController.buildSchemaController(old[kSchemaController])
instance[pluginUtils.kRegisteredPlugins] = Object.create(instance[pluginUtils.kRegisteredPlugins])
```

子实例默认可读父实例的一切（继承），但写钩子/装饰器/解析器都落在自己的拷贝上（私有）；`fastify-plugin` 包裹的插件会命中 `shouldSkipOverride`（L29-36）直接返回父实例，实现跨封装共享。Request/Reply 两个类也在每层封装中重建（`Reply.buildReply(instance[kReply])`），保证 `decorateRequest/Reply` 不污染兄弟上下文。这条原型链 + 按写时拷贝的设计，是 Fastify 内存开销低的直接原因之一。

### 5.3 请求生命周期管线（lib/handle-request.js + lib/hooks.js）

生命周期钩子分两组（lib/hooks.js L3-23）：6 个应用级（onRoute/onRegister/onReady/onListen/preClose/onClose）+ 10 个请求级（onTimeout、onRequest、preParsing、preValidation、preSerialization、preHandler、onSend、onResponse、onError、onRequestAbort），共 16 个。每条请求的执行序在 lib/handle-request.js 中呈显式状态机：路由命中拿到每路由 `Context`（lib/context.js，持有编译好的 validator/serializer 与解析器缓存）→ body 需要时走 `contentTypeParser.run` → `preValidationHookRunner` → `validateSchema`（lib/validation.js）→ `preHandlerHookRunner` → `context.handler(request, reply)` → 返回 Promise 则 `wrapThenable` 接到 reply（L204-210）。错误统一进 onError/`reply.send(err)` 管线，并接 `diagnostics_channel` tracingChannel（L22、L154-168）供 APM 无侵入订阅。

### 5.4 校验与序列化的 schema 编译（lib/route.js + lib/validation.js + lib/schemas.js）

路由注册时若带 schema，`lib/route.js` 调 `compileSchemasForValidation`（ajv）与 `compileSchemasForSerialization`（fast-json-stringify），编译产物缓存在该路由的 `Context` 上（context.js L59-60 `kRequestCacheValidateFns`/`kReplyCacheSerializeFns`），请求期零反射、直接调用生成的函数。序列化器还能按状态码区分（lib/error-serializer.js）。这一"提前编译、热路径直调"的思路贯穿全库。

### 5.5 日志：pino 深度集成（lib/logger-pino.js、lib/logger-factory.js、lib/log-controller.js）

`createPinoLogger`（logger-pino.js L17 起，代码注明改自 pino-http）为每个实例创建 pino logger，并为每条请求用 `childLoggerFactory` 生成带 reqId 的子 logger；`kGenReqId`/`reqIdGenFactory.js` 负责请求 ID 生成。可通过 `logger: false` 或传任意 logger 实例替换，`abstract-logging` 保证关闭时零开销空实现（fastify.js 依赖表）。

## 6. 构建与运行

均出自 package.json scripts（未运行，仅静态核对）：

```bash
npm install
npm test            # lint:eslint + unit(borp) + test:types(tstyche)
npm run unit        # borp 运行 node:test 测试
npm run unit:report # c8 覆盖率 + 行报告
npm run coverage:ci-check-coverage  # 强制 --lines 100（100% 行覆盖）
npm run lint:markdown
npm run benchmark   # examples/benchmark/simple.js + autocannon -c 100 -d 30
```

用户侧使用：`npm i fastify`，`fastify({logger:true}).get(...)`，`fastify.listen({port})`；无类型/浏览器端另有 `light-my-request` 提供 `inject()` 伪请求（fastify.js 的 inject 方法，L293）。

## 7. 测试与工程质量

- 规模：`test/` 196 个 .test.js、约 46000 行测试代码（实测 `cat test/*.test.js | wc -l`），测试数约为核心代码（9459 行）的 4.8 倍；另有 `integration/` 打包器集成与 `test/build/` 构建产物测试。
- 风格：node:test 原生 `test` + 断言（test/hooks.test.js L4），borp 为运行器；覆盖模式为"每个 lib 模块 + 每类行为一个专项测试文件"（如 body-limit、case-insensitive、client-timeout、versioned-routes、constrained-routes 等，test/ 目录实测）。
- 质量红线：CI 强制 100% 行覆盖（`--check-coverage --lines 100`，package.json `coverage:ci-check-coverage`）；校验器与错误序列化器是 `build/` 生成的，专门有 `test:validator:integrity` 校验生成物与提交一致（package.json L25）。
- CI：20 个 workflow（.github/workflows/），包括主 ci、coverage-nix/win、package-manager-ci（npm/pnpm/yarn/bun 生态验证）、ci-alternative-runtime 与 integration-alternative-runtimes（非 Node 运行时兼容）、citgm（Node.js 基金会模块健康度）、links-check、labeler、lock-threads，工程化程度非常高。

## 8. 社区与维护状态

- 活跃度：最近 push 2026-11-16（分析前 1 天）；open issues 196（gh api）。
- 发版节奏：双轨制——v5.12.x 稳定线补丁（9-11 月发布 5.12.4~5.12.8）与 v6.0.0-alpha 预告线（alpha.4 9/17、alpha.5 10/16、alpha.6 11/2），月均 2-4 个 release（gh api releases）。main 即 v6，README 明确"main 分支对应 Fastify v6，v5 用 5.x 分支"（README.md）。
- 贡献者：contributors 分页 per_page=1 显示 433 页，即约 433 位贡献者（gh api）；package.json 维护 24 位核心 contributors 名单，作者 Matteo Collina（Node.js TSC 成员）。
- 治理：有正式的 GOVERNANCE.md、PROJECT_CHARTER.md、SECURITY.md、CII Best Practices 徽章（仓库顶层文件、README）。
- 组织背景：Fastify 是独立开源组织（Open Collective + GitHub Sponsors 资助，package.json funding），生态用 `@fastify/*` 前缀多仓扩展。

## 9. 亮点与风险

亮点：
1. 封装式插件架构是同类框架中最完备的：Object.create 原型继承 + 写时拷贝（lib/plugin-override.js），作用域规则明确、内存友好。
2. schema 编译路线（ajv + fast-json-stringify）把校验/序列化成本前移到启动期，热路径直调生成代码；README 基准显示开销接近裸 http.Server（README.md L205-213）。
3. 工程质量极高：100% 行覆盖红线、20 个 CI workflow、生成物一致性校验、类型测试（tstyche）、多运行时验证。
4. 启动序列用 avvio 显式化，异步插件依赖可 await、可打印（`printPlugins`），解决了 Express 中间件加载时序黑洞。
5. 文档即源码：docs/ 下 21 篇 Reference 随仓库维护，版本分支（5.x/main）与文档同步。

风险/局限：
1. 核心版本处于 v6 alpha 期（package.json 6.0.0-alpha.6），生产用户需选 5.x 线，存在双版本迁移成本。
2. schema 一等公民意味着"不用 schema 的 Fastify"损失了它大部分性能优势；schema 编译也让启动时间随路由数增长（router.compileRoutes 在 preReady，fastify.js L400-406）。
3. README 基准为 hello-world 合成场景，多中间件/复杂业务下与 Koa/hapi 的差距会显著缩小（README 自己也如此声明）。
4. 生态分散在 `@fastify/*` 多仓，主仓不含开箱即用的认证/静态资源等能力，需自行组装（docs/Reference/Index.md 生态列表）。

## 10. 总结评价

Fastify 是"把工程纪律做成框架特性"的典范：约 9.5k 行的内核（fastify.js + lib/）用原型链封装、schema 编译与 16 个钩子的显式生命周期，把 Node.js HTTP 框架的运行时开销压到接近裸服务器，同时用 avvio、pino、ajv/fast-json-stringify 补齐了 Express 时代缺的启动时序、日志与校验三块短板；433 位贡献者、近 4.6 万行测试、100% 覆盖红线与月度级发版表明这是一个治理成熟、长期活跃的一线基础设施项目。适合追求性能与可维护性的 Node.js 后端团队作为默认框架选用；想学"如何在 JS 里做分层作用域与零抽象成本设计"的人，直接读 lib/plugin-override.js、lib/handle-request.js、lib/route.js 这三个文件收益最大。
