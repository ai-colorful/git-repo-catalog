# trpc/trpc 项目分析

> 星数 41500（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

tRPC 是 TypeScript 全栈生态中最主流的**端到端类型安全 RPC 框架**：不需要 schema、不需要代码生成，客户端直接从服务端 router 的 TypeScript 类型推导出输入/输出/错误类型，"Move fast and break nothing"。

| 项目 | 事实 |
|---|---|
| 名称 | trpc/trpc（npm 包 `@trpc/server`、`@trpc/client`、`@trpc/react-query`、`@trpc/next` 等） |
| 星数 | 41500（快照）；gh api 实测 40805（2026-11-13） |
| 语言 | TypeScript（100%，严格 TS 5.7+ 类型体操） |
| 许可证 | MIT（`LICENSE`，Copyright 2023 Alex Johansson） |
| 创建 / 最近推送 | 2020-07-18 / 2026-11-13（gh api，极度活跃） |
| 默认分支 | `main` = v11（包版本 11.28.0，`packages/server/package.json`）；v12 为 `v12` 分支上的重写（`AGENTS.md` 明确说明） |
| 架构 | pnpm + Turborepo monorepo，8 个子包 + 29 个示例 + 文档站 |
| 贡献者 | 462 人（gh api contributors） |

## 2. 项目是什么，解决什么问题

全栈 TypeScript 应用中，前后端之间的 API 边界传统上靠三种方式衔接：手写 REST 接口 + 前端凭记忆拼 URL、GraphQL（需要 SDL schema 和 codegen）、OpenAPI（先写 spec 再生成）。三者都要维护一份"接口的描述"，描述与实现会漂移，类型安全靠生成器事后补上。

tRPC 的核心洞察是：既然前后端都是 TypeScript，**接口描述可以直接就是服务端代码本身的类型**。服务端用链式 builder 定义 procedure（带 zod 等校验器的输入、resolver 的返回值），客户端只导入 `type { AppRouter }` 这一个纯类型（README 强调 "The client above is **not** importing any code from the server, only its type declarations"，`README.md`），即可获得完整的输入/输出/错误自动补全与编译期校验。没有运行时开销，没有构建管线，客户端零依赖（`packages/client/package.json` 无 `dependencies` 字段，仅 peerDeps `@trpc/server` + `typescript >=5.7.2`）。

它不是"又一个 RPC 库"，而是围绕 React/Next.js 全栈工作流深度集成：`@trpc/react-query` 把 procedure 直接变成 TanStack Query hooks（`useQuery`/`useMutation`/无限查询/预取/SWR 式缓存全部类型安全），`@trpc/next` 提供 App Router 的 server-side helper 与缓存标签集成。传输层支持 HTTP 批量请求、SSE 流式、WebSocket，订阅（subscription）是原生能力（`packages/server/src/unstable-core-do-not-import/stream/`）。

定位上与 GraphQL 的关键差异：GraphQL 面向"多客户端、按需裁剪字段、语言无关"，tRPC 面向"同一个 TS monorepo 内的前后端"，以放弃跨语言换来零 schema、零 codegen 的开发体验。v11 新增的 `@trpc/openapi` 包（从 router 生成 OpenAPI 3.1 spec，`packages/openapi/README.md`）补上了对 Postman、多语言客户端和 AI agent 集成的兼容出口，弱化了这一短板。

## 3. 技术栈与依赖

- **语言/运行时**：TypeScript（`typescript >=5.7.2` peerDep），Node ^24（根 `package.json` engines）
- **构建**：Turbo（任务编排）+ pnpm workspace + tsdown（打包，各包 `tsdown.config.ts`）
- **测试**：Vitest + @vitest/coverage-istanbul（根 `vitest.config.ts`、`codecov.yml`），Testing Library（React）
- **lint/格式**：oxlint + 自研 `tooling/oxlint-plugin`，oxfmt（根 `package.json` scripts）
- **核心运行时依赖**：刻意做到 `@trpc/server` 零第三方依赖；校验器（zod 等）走 **Standard Schema V1** 接口（`packages/server/src/vendor/standard-schema-v1/spec.ts`），transformer、cookie 等极少量第三方库以 vendored 方式内嵌（`packages/server/src/vendor/`）
- **peer 生态**：@tanstack/react-query（react-query/tanstack-react-query 包）、next、express、fastify、ws（适配器）

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `packages/server` | 核心：`initTRPC`、procedure/router/middleware、HTTP/RPC 编解码、全部服务端适配器（86 个源码文件） |
| `packages/client` | 客户端：类型化 Proxy、links 体系（http/batch/SSE/ws/local）、TRPCClientError（40 文件） |
| `packages/react-query` | `@trpc/react-query`：TanStack Query hooks 集成（36 文件，54 个测试文件） |
| `packages/tanstack-react-query` | 面向 TanStack Query 官方包（非 React 专用封装）的新一代集成（20 文件） |
| `packages/next` | `@trpc/next`：pages router 集成 + App Router 的 server helpers/缓存标签（13 文件） |
| `packages/openapi` | v11 新增：从 router 类型提取 schema 生成 OpenAPI 3.1 + heyapi 客户端代码生成（源码仅 6 个文件约 2.8k 行） |
| `packages/upgrade` | `@trpc/upgrade` codemod CLI：基于 AST 的升级迁移工具（hooks→options 等 transforms） |
| `packages/tests` | 集成测试主战场：102 个测试文件覆盖 adapters/links/subscriptions/batching 等端到端场景 |
| `examples/` | 29 个可运行示例（next-prisma-starter、express/fastify/cloudflare-workers/lambda/deno/nuxt 等） |
| `www/` | 文档站（vitepress 类），docs 按 v9/v10/v11 分版本目录 |
| `tooling/` | 内部工程设施：oxlint-plugin、release CLI、subtree、ui |
| `AGENTS.md` | AI 代理协作规范（测试/类型检查命令、回归测试政策、v11/v12 分支规则） |

组织方式评述：典型的多包 monorepo，但依赖方向严格单向——`server` 为零依赖底座，`client`/`react-query`/`next` 逐层向上 peer 依赖 `@trpc/server`；`@trpc/server/unstable-core-do-not-import` 子路径导出让 client 直接复用 server 的类型工具与 `createRecursiveProxy`（`packages/client/src/createTRPCClient.ts`），这是 client 与 server 类型系统咬合的关键设计。

## 5. 核心架构与关键实现

### 5.1 initTRPC：两段式 builder 装配类型上下文

`packages/server/src/unstable-core-do-not-import/initTRPC.ts` 用一个 `TRPCBuilder` 类实现 `.context<T>()`/`.meta<T>()` 链——每次调用返回携带新类型参数的**新实例**，`.create()` 时把 ctx/meta/errorFormatter/transformer 固化进 `RootConfig`，产出 `procedure`（9 泛型参数的初始 builder）、`middleware`、`router`、`createCallerFactory` 四件套（initTRPC.ts:135-251）。`$types: null as any` 的注释直白说明：这些是纯类型槽位，运行时不存在（initTRPC.ts:185）。

### 5.2 ProcedureBuilder：链式 procedure 与中间件即一切

`procedureBuilder.ts` 是框架心脏。`ProcedureBuilder` 携带 9 个类型参数（`TContext, TMeta, TContextOverrides, TInputIn, TInputOut, TOutputIn, TOutputOut, TCaller, TErrorShape`），每个方法返回新 builder 并**推进类型状态**：`.input()` 用 `IntersectIfDefined<TInputIn, inferParser<$Parser>['in']>` 累积输入类型并对"链式校验器必须都是对象"给出 `TypeError<'...'>` 编译期报错（procedureBuilder.ts:232-254）；`.use()` 把中间件的 ctx 覆盖量折叠进 `TContextOverrides`（:317-342）；`.query()`/`.mutation()`/`.subscription()` 终结为携带 input/output 类型的 Procedure 类型（:429-489）。`UnsetMarker` 哨兵类型区分"未设置"与 `undefined`。

运行时实现异常简洁：每个 builder 内部只是累积一个 `middlewares: AnyMiddlewareFunction[]`，`.input()`/`.output()`/`.errors()` 本身也被实现为内置中间件（`createInputMiddleware`/`createOutputMiddleware`/`createErrorFormatterMiddleware`，procedureBuilder.ts:584-612），调用时 `callRecursive` 递归执行中间件数组、resolver 是最后一个中间件（procedureBuilder.ts:655-746）。中间件必须返回带 `middlewareMarker` 品牌标记的 `next()` 结果，防止忘记透传（middleware.ts:16-49）。

```ts
// procedureBuilder.ts:722-737 —— 中间件递归执行，resolver 即最后一个中间件
async function callRecursive(
  index: number,
  _def: AnyProcedureBuilderDef,
  opts: ProcedureCallOptions<any>,
): Promise<MiddlewareResult<any>> {
  try {
    const middleware = _def.middlewares[index]!;
    const result = await middleware({
      ...opts,
      meta: _def.meta,
      next(_nextOpts?: any) { /* ...支持覆盖 ctx/input */ }
```

### 5.3 类型桥接魔法：纯类型导入 + 递归 Proxy

客户端类型推导是 tRPC 的招牌。类型层面：`TRPCClient<TRouter>` = `DecoratedProcedureRecord<TRouter['_def']['record']>`，用条件类型遍历 router record，对每个 procedure 用 `inferProcedureInput`/`inferTransformedProcedureOutput`/`inferProcedureErrorShape` 还原签名，并按 `_def.type` 装配 `.query()`/`.mutate()`/`.subscribe()` 方法（createTRPCClient.ts:52-152）。运行时层面：`createRecursiveProxy` 拦截任意深度的属性访问链，弹出末段（query/mutate/subscribe）确定调用类型，剩余段 join 成 procedure 路径字符串交给 untyped client 发请求（createTRPCClient.ts:177-194）：

```ts
// packages/client/src/createTRPCClient.ts:180-187
const proxy = createRecursiveProxy<TRPCClient<TRouter>>(({ path, args }) => {
  const pathCopy = [...path];
  const procedureType = clientCallTypeToProcedureType(pathCopy.pop()!);
  const fullPath = pathCopy.join('.');
  return (client[procedureType] as any)(fullPath, ...(args as any));
});
```

因为客户端从不 import 服务端实现，打包产物里 router 代码不会被带进 client bundle——类型信息在编译后彻底消失。

### 5.4 Router、懒加载与订阅流

`router.ts` 的 `createRouterFactory` 把嵌套 record 拍平成 `procedures` 全路径映射并检测重复 key；`_def.lazy` 支持懒加载子 router，`getProcedureAtPath` 在按路径查找时按需 `await lazyRouter.load()`（router.ts:380-466）。订阅体系在 `stream/` 下：`tracked(id, data)` 给事件打品牌化 id，SSE 断线后客户端以 `lastEventId` 续传（`stream/tracked.ts`）；`sse.ts`/`jsonl.ts` 提供两种流编解码，客户端对应 `httpSubscriptionLink`（SSE，client/src/links/httpSubscriptionLink.ts）与 `wsLink`（WebSocket）。

### 5.5 适配器矩阵与 React 集成

服务端适配器在 `packages/server/src/adapters/`：`fetch`（`fetchRequestHandler.ts`，通用 Web 标准，覆盖 Next/Vercel Edge/Cloudflare Workers/Deno）、`express`、`fastify`、`aws-lambda`、`standalone`（自带 HTTP server）、`ws`（WebSocket 订阅）+ `node-http` 公共层。React 侧 `createTRPCReact.tsx` 用同一 proxy 手法把每个 procedure 映射为 `useQuery`/`useInfiniteQuery`（自动从 input 推断 cursor 类型，createTRPCReact.tsx:108-131）/`useMutation`/`useSubscription`，并提供 `createTRPCQueryUtils`、RSC 服务端工具（react-query/src/rsc.tsx）。

## 6. 构建与运行

```bash
# 构建（仅 packages/*，turbo 编排；AGENTS.md 与根 package.json）
pnpm build                        # turbo --filter=./packages/* build（tsdown）
pnpm dev                          # 各包 tsdown --watch

# 测试（vitest workspace；测试直接 import 源码，无需先构建）
pnpm test                         # 全量
pnpm vitest run <file>            # 单文件（AGENTS.md 推荐方式）

# 质量
pnpm typecheck-packages           # tsc --noEmit 并行
pnpm lint-fix                     # oxlint --fix + manypkg fix + oxfmt
pnpm build-www                    # 文档站

# 试用
npx create-next-app --example https://github.com/trpc/trpc --example-path examples/next-prisma-starter
```

发布走自研 release CLI：`pnpm release stable/canary`（根 package.json `release` script → `tooling/release/src/bin/release.ts`），GitHub Actions `release.yml` 带 npm provenance。

## 7. 测试与工程质量

- **规模**：全仓约 6.5 万行源码 + 6.1 万行测试代码（≈1:1）；222 个测试文件分布：packages/tests 102（端到端集成）、react-query 54、server 20、tanstack-react-query 20、client 11、openapi 11、next 4、upgrade 1（实测统计）
- **风格**：以行为级集成测试为主（batching、headers、abortQuery、httpSubscriptionLink.memory/retry 等命名直指场景，`packages/tests/server/`），单元测试内联在各包 `*.test.ts`
- **CI**（`.github/workflows/`）：`main.yml` 在 PR + merge queue + release commit 上跑 build/typecheck/test；另有 CodeQL、semantic-pr（PR 标题规范）、labeler、lock-issues、release（provenance）、subtree 等 workflow；`.github/renovate.json` 管依赖更新
- **质量政策**（`AGENTS.md`）：每个 bug fix 必须附"没有修复就会失败的回归测试"并在 PR 标注 `Regression check:`；v11 不允许行为变更的默认值，新默认进 v12；CodeRabbit 机器人参与每个 push 的评审。工程纪律在同类 OSS 中属于第一梯队。

## 8. 社区与维护状态

- **活跃度**：gh api 显示最近 push 2026-11-13（分析前一天），open issues 113（对 41k 星项目极低）；2026-11-06 至 11-10 五天内连发 v11.25.0→v11.28.0 共 6 个 release，发版节奏极快
- **团队结构**：README 列出 3 位 project leads（Alex/KATT——作者，Julius Marminge，Nick Lucas）+ 1 位 active contributor；462 名贡献者（gh api）
- **双轨开发**：`main` 维护 v11 稳定线，`v12` 分支为重写版（AGENTS.md 明示；client 源码中多处 "will be removed in v12" 弃用注记印证，createTRPCClient.ts:32-42）；CI 同时覆盖 main/next/canary 分支
- **生态**：Discord 社区、GitHub Sponsors（Silver partners 含 CodeRabbit/Greptile/SerpApi，README）、T3 Stack（create-t3-app）把 tRPC 作为默认组件之一带来持续曝光
- **AI 时代工程范式样本**：`.agents/skills/`、`AGENTS.md`、`skills-lock.json`、`packages/*/skills` 打包时从 docs 生成——该仓库本身就是"AI 参与维护"的先行者

## 9. 亮点与风险

**亮点**
1. 类型桥接设计教科书级：纯类型导入 + 递归 Proxy + 条件类型映射，把"端到端类型安全"做到零运行时、零 codegen（createTRPCClient.ts、procedureBuilder.ts）
2. 零依赖内核 + Standard Schema 开放校验器接口，vendor 目录控制依赖面（packages/server/src/vendor/）
3. 测试/源码行数 1:1，回归测试强制政策，merge queue + CodeQL + 机器人评审，工程质量标杆
4. 版本演进有章法：unstable-core-do-not-import 明确内部边界；upgrade codemod 包为 v12 迁移铺路
5. `@trpc/openapi` 补齐与 OpenAPI 世界/AI agent 的互操作出口，同时保住"无 schema"主路线

**风险/局限**
1. 类型体操深（ProcedureBuilder 9 个泛型参数、大量条件类型），TS 版本敏感（peerDep >=5.7.2），大型 router 下编译期类型计算可能拖慢 IDE
2. 强绑定 TypeScript 同构栈：跨语言客户端依赖尚新的 `@trpc/openapi`（OpenAPI 3.1 消费方支持参差）
3. v11/v12 双线并行，`unstable-*` API（unstable_concat、unstable-handles 等）面较大，升级心智成本存在
4. 订阅的 SSE/WS 双通道、transformer（superjson 等）要求 client/server 配置一致，配置漂移是社区常见错误源

## 10. 总结评价

tRPC 用"类型即接口"的极简哲学，把 TS 全栈开发的 API 层摩擦压缩到接近于零，是 2020 年代前端工程最具影响力的原创设计之一；它没有 GraphQL 的通用性野心，却因此在目标场景（TS monorepo 全栈、尤其 React/Next.js）做到了体验与工程质量的双重标杆。仓库本身也值得精读：procedure builder 的类型状态机、middleware 递归执行、纯类型 + Proxy 的类型桥接，是学习 TypeScript 高级类型工程与框架 API 设计的范本。适合全栈 TS 团队直接采用；适合框架作者、类型体操学习者和 AI 协作工程实践研究者精读源码。
