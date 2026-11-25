# FlowiseAI/Flowise 项目分析

> 星数 55521（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：Flowise 是一个拖拽式的 LLM 应用可视化编排平台——用 React Flow 画布连接 LangChain 节点，即可搭建聊天机器人、RAG 问答、Agent 工作流，并以 API/嵌入组件形式对外服务。

| 事实 | 内容 |
| --- | --- |
| 名称 / 版本 | FlowiseAI/Flowise，v3.1.4（`package.json`） |
| 星数 / Fork | 55521 stars / 25090 forks（gh api 快照） |
| 主语言 | TypeScript（后端+组件），前端为 286 个 .jsx 文件（`packages/ui/src`） |
| 许可证 | Apache 2.0 + 商业许可例外（`packages/server/src/enterprise` 目录专有，`LICENSE.md`） |
| 创建时间 | 2023-03-31（gh api） |
| 最近推送 | 2026-08-13 —— **同日官方宣布停止运营并归档仓库**（`gh api` 返回 `archived: true`；README 顶部横幅指向 discussion #6727 "The Future of Flowise"） |
| 贡献者 | 约 316 人（contributors API 分页推断） |

## 2. 项目是什么，解决什么问题

Flowise 解决的核心问题是：LangChain 等底层 LLM 库的编程门槛高、接线繁琐。它把"选模型 → 配提示词 → 接向量库 → 加记忆/工具"这些步骤抽象成画布上的可视化节点，开发者拖拽连线即可组成一条完整链路（Chatflow）或带分支、循环、人工介入的 Agent 工作流（Agentflow），保存后由后端把画布 JSON 编译成可执行的调用图，通过 `/api/v1/prediction/{id}` 直接对外提供 API 和流式聊天界面。README 的定位语是 "Build AI Agents, Visually"（`README.md`）。

仓库是典型的 pnpm monorepo，README "Developers" 一节自述包含三个模块：`server`（Node 后端）、`ui`（React 前端）、`components`（第三方节点集成），外加自动生成的 swagger 文档包（`README.md`、`pnpm-workspace.yaml`）。`packages/components` 按 25 个类别收录了约 278 个节点集成：chatmodels 29 个（OpenAI/Anthropic/Ollama/阿里通义等）、tools 40、documentloaders 39、vectorstores 24、embeddings 16 等（`packages/components/nodes/` 目录实测统计），基本等于一份"LangChain 生态集成百科"。

产品形态覆盖三层：个人开发者可 `npm install -g flowise` 一行启动（server 包是 oclif CLI，`packages/server/package.json`）；团队可用 Docker/queue 模式部署（`docker/docker-compose-queue-*.yml`，BullMQ + Redis worker）；商业版在 `packages/server/src/enterprise` 提供工作区、RBAC、SSO 等企业能力（该目录受 `proprietary-path-guard.yml` CI 保护，不适用 Apache 条款）。

另一个值得注意的演进：仓库后期新增 `packages/agentflow`——一个可独立发布的嵌入式 React 画布组件 `@flowiseai/agentflow`（0.0.0-dev.14），采用 Domain-Driven 分层（atoms/features/core/infrastructure），配套 `ARCHITECTURE.md` 与大量组件测试，代表项目向"可嵌入画布 SDK"方向的最后一次探索。

需要如实记录的是：**该项目已于 2026-08-13 归档停运**。官方在 discussion #6727 中给出的理由是：随着模型推理能力变强，开发者越来越多地用编码智能体直接写复杂任务，"刚性低代码工作流在复杂度面前很快触顶"，因此决定 wind down Flowise 的运营（gh api discussions/6727，2026-08-13）。星数虽然仍在 55k 高位，但这是一个已 EOL 的活跃快照。

## 3. 技术栈与依赖

- **语言/运行时**：TypeScript 5.4（后端、组件、agentflow 包）+ JavaScript/JSX（旧版 UI），Node ^24、pnpm ^10.26（根 `package.json` engines）。
- **后端**：Express 4 + TypeORM（实体见 `packages/server/src/database/entities/`，23 个实体；迁移分 sqlite/mysql/postgres/mariadb 四套，`packages/server/src/database/migrations/`）；任务队列 BullMQ（`packages/server/src/queue/QueueManager.ts`，queue 模式可选）；SSE 流式输出（`packages/server/src/utils/SSEStreamer.ts`）；oclif 打包成全局 CLI。
- **编排/执行**：核心 LLM 能力来自 LangChain 全家桶（`@langchain/core` 1.1.20 锁定、`@langchain/anthropic`、openai 6.19 等，根 `package.json` resolutions），组件包统一封装为 INode 类。
- **前端**：React + Vite + Material UI(emotion) + reactflow 画布 + Redux（`packages/ui/src/store/reducer.jsx`）；新 agentflow 包为 TS 组件 + 自建 atoms 体系 + jest。
- **工程化**：Turborepo 编排构建（`turbo.json`）、Changesets 发版、husky/lint-staged、Jest（150 个 `*.test.ts`，实测）+ Cypress e2e（CI 中跑真实 server，`.github/workflows/main.yml`）。
- **安全加固痕迹**：根 `package.json` 的 pnpm overrides 强制锁定了 axios、multer、protobufjs、ws、tar-fs 等一长串版本——明显是历次供应链漏洞通告后的补丁；`packages/server/src/utils/XSS.ts` 提供 sanitize 中间件。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `packages/server/` | Express 后端：60 余个路由目录（`src/routes/`）、50 余个 controller、TypeORM 持久层、queue/schedule、enterprise 专有代码；604 个 ts 文件 |
| `packages/ui/` | 旧版主前端（React+jsx，286 个 jsx 文件）：画布编辑器、聊天窗口、文档管理后台 |
| `packages/components/` | 节点库（556 个 ts 文件）：25 类约 278 个节点 + credentials + evaluation 工具 |
| `packages/agentflow/` | 新一代可嵌入画布组件 `@flowiseai/agentflow`（231 个 ts 文件，DDD 分层 + 自带 ARCHITECTURE.md） |
| `packages/observe/` | 可观测性组件包（79 个 ts 文件，publish-observe.yml 单独发版） |
| `packages/api-documentation/` | 由 express 路由自动生成的 swagger-ui 文档 |
| `docker/` | Dockerfile 与标准/queue 两种 compose 拓扑（含 worker 镜像） |
| `i18n/` | README 的多语言译本（繁中/简中/日/韩） |
| `.github/workflows/` | 8 条工作流：Node CI、DockerHub/ECR 发布、三个子包独立发版、专有目录守卫 |

组织方式评述：标准的 pnpm monorepo + Turborepo，包间依赖单向（server → components → langchain；ui 经 API 调 server），`turbo.json` 只定义 build/test/dev 三条流水线，职责边界清晰。server 内部又是"controller/service/utils + 实体"的分层单体，enterprise 子目录用许可证隔离商业代码，是开源核心+商业专有的常见切分。

## 5. 核心架构与关键实现

### 5.1 节点池与节点协议（`packages/server/src/NodesPool.ts`、`packages/components/src/Interface.ts`）

服务启动时 `NodesPool.initialize()` 扫描 `flowise-components/dist/nodes` 目录，`require()` 每个编译产物，凡导出 `nodeClass` 的都实例化并登记进 `componentNodes` 映射；支持 `DISABLED_NODES` 环境变量和社区节点开关（`NodesPool.ts` 的 `loadNodesFromDir`）。节点协议是一致的 INode 结构，以 ChatAnthropic 为例（`packages/components/nodes/chatmodels/ChatAnthropic/ChatAnthropic.ts`）：

```ts
class ChatAnthropic_ChatModels implements INode {
    constructor() {
        this.label = 'Anthropic Claude'
        this.name = 'chatAnthropic'
        this.version = 8.0
        this.category = 'Chat Models'
        this.credential = { name: 'credential', type: 'credential',
            credentialNames: ['anthropicApi'] }
        this.inputs = [{ label: 'Model Name', name: 'modelName',
            type: 'asyncOptions', loadMethod: 'listModels',
            default: 'claude-haiku-4-5' }, ...]
    }
}
```

`inputs` 声明输入桩的类型与可选性，`type: 'asyncOptions' + loadMethod` 让 UI 能动态拉取模型列表——这是画布与节点间统一的"自描述"契约，新集成只需写一个类文件即可被 UI、API、执行引擎三方识别。

### 5.2 Chatflow 执行引擎（`packages/server/src/utils/buildChatflow.ts`，1163 行）

`utilBuildChatflow` 是 `/api/v1/prediction/{id}` 的主入口：读取 chatflow 的画布 JSON（`chatflow.flowData`），从 Start 节点出发解析依赖，按拓扑顺序实例化各节点的 `nodeClass` 并调用其 `init/run`，LangChain 对象在节点间以 `baseClasses` 类型匹配的方式互相注入（如 ChatModel 实例注入 Chain）。全程通过 `SSEStreamer` 把 token 级增量、工具调用、source 文档等事件推给前端，并把会话/消息/执行记录写回 TypeORM 实体（ChatMessage、Execution）。`overrideConfig` 机制允许 API 调用方在运行时覆盖节点参数（`getAPIOverrideConfig`）。

### 5.3 Agentflow 图执行器（`packages/server/src/utils/buildAgentflow.ts`，2471 行）

Agentflow V2 是项目后期重写的核心：`executeAgentFlow` 把画布编译成有向图（`constructGraphs` 生成 `graph`/`reversedGraph`），用显式的 `nodeExecutionQueue` + `waitingNodes` 做事件驱动的图遍历，支持条件分支（Condition 节点多输出组）、循环（Loop/Iteration 节点带 `loopCounts`）、人工输入挂起恢复、webhook 触发、父子执行（`parentExecutionId`，子流程复用）。主循环骨架（`buildAgentflow.ts` 约 2020 行起）：

```ts
while (nodeExecutionQueue.length > 0 && status === 'INPROGRESS') {
    if (iterations++ > maxIterations) throw new Error('Maximum iteration limit reached')
    const currentNode = nodeExecutionQueue.shift()
    const executionResult = await executeNode({ ... })
    if (executionResult.shouldStop) { status = 'STOPPED'; break }
    // processNodeOutputs 依据 graph 把结果扇出到下游，
    // 条件节点只激活命中的分支，未就绪节点进入 waitingNodes
}
```

细节上可见不少生产化打磨：`{{ $webhook.body.* }}` 模板解析时显式拦截 `__proto__/constructor/prototype` 防原型链污染（`resolveWebhookRefs`，文件头注释）；`AbortControllerPool` 支持中止运行中的执行；每次迭代有 `maxIterations` 兜底防死循环。

### 5.4 企业层与多租户（`packages/server/src/enterprise/`）

server 的 index.ts 无条件 import enterprise 模块（Organization/Workspace 实体、passport JWT、`checkPermission` RBAC），说明商业与开源共享同一份启动代码、靠许可证目录隔离（`packages/server/src/index.ts`）。配合 `IdentityManager.ts`、Stripe 计费（`StripeManager.ts`）与用量缓存（`UsageCacheManager.ts`），构成云版的 SaaS 底座；开源单机模式走 `MODE: 'hosted'` 分支。

## 6. 构建与运行

来自 `README.md` 与各 `package.json` scripts（Node >= 20 实测要求 24，构建需 4GB 堆内存）：

```bash
# 开发者全量构建
pnpm install && pnpm build        # turbo 串行各包，大内存需求见 README 提示
pnpm dev                          # 前后端并行热更，访问 http://localhost:8080
pnpm start                        # 产物模式，http://localhost:3000

# 终端用户一行安装
npm install -g flowise && npx flowise start

# 测试
pnpm test                         # 各包 jest
pnpm test:coverage
cd packages/server && pnpm e2e    # cypress: 起 dev server 后跑 e2e

# Docker
cd docker && docker compose up -d # 标准模式；docker-compose-queue-*.yml 为 Redis 队列模式
```

## 7. 测试与工程质量

- **单元测试**：全仓 150 个 `*.test.ts(x)`，Jest；分布合理——server 的 utils（XSS、apiKey、ipValidation、oauth2Security、callbackDispatcher 等）、components、agentflow 包（atoms 几乎每个组件都有配套测试，`packages/agentflow/src/atoms/*.test.tsx`，并有 `__test_utils__`）。
- **E2E**：Cypress，CI 里真实启动 server 后等待 `http://localhost:3000` 再跑（`.github/workflows/main.yml`）。
- **CI**：main.yml 在 Node 24.15.0 上执行 lint → build → test:coverage → cypress；另有 DockerHub/ECR 镜像发布、`proprietary-path-guard.yml` 防止专有代码泄漏进开源面、agentflow/observe/components 三条独立发版流水线。
- **质量观察**：依赖锁定严格（resolutions/overrides 大量安全版本钉扎）；`agentflow` 包附 `ARCHITECTURE.md` 与 `TESTS.md`，是全仓工程化最规范的子包。旧 UI 以 .jsx 为主、无类型约束，与 TS 后端形成明显质量分层。对 2471 行的 `buildAgentflow.ts` 这类巨型文件，测试保障主要靠 utils 级单测，核心执行循环本身的覆盖程度未确认。

## 8. 社区与维护状态

- **活跃度（如实记录）**：仓库创建于 2023-03-31，2023-2025 年间是 LangChain 生态最热的可视化项目之一（55.5k 星、25k fork、316 贡献者）。**2026-08-13 官方发布 "The Future of Flowise" 公告宣布 wind down，仓库同日归档（archived: true），README 顶部挂出归档横幅**（gh api discussions/6727 + repos API）。
- **发版节奏**：归档前仍保持数月一版——3.1.2（2026-04-14）→ 3.1.3（2026-06-25）→ 3.1.4（2026-07-29，最终版），tag 形如 `flowise@3.1.4`，由 Changesets 管理（gh api releases）。
- **最后推送**：2026-08-13，与归档公告同日，此后无提交。
- **社区渠道**：README 徽章含 Discord 与 Twitter；issue 1041 个保持开放（归档后只读状态，未确认是否逐一关闭）。

## 9. 亮点与风险

**亮点**
1. 节点协议设计优秀：INode 自描述（inputs/credential/baseClasses/loadMethod）让 278 个第三方集成以极低成本接入 UI、API 与执行引擎（`packages/components/src/Interface.ts`）。
2. 执行引擎生产化程度高：Agentflow 图执行器覆盖条件/循环/人工介入/子流程/中止/限流，且带有原型链污染防护等安全细节（`packages/server/src/utils/buildAgentflow.ts`）。
3. 工程完整度好：四数据库迁移、BullMQ 队列模式、SSE 流式、OpenTelemetry/Prometheus 指标、Cypress e2e、专有代码 CI 守卫一应俱全。
4. `@flowiseai/agentflow` 展示了将画布抽为可嵌入 SDK 的清晰 DDD 架构，适合作为同类项目的架构参考（`packages/agentflow/ARCHITECTURE.md`）。

**风险/局限**
1. **项目已归档停运**：无后续安全补丁与发版，生产使用需自行 fork 维护；官方结论本身也宣示了"刚性低代码工作流"路线在编码智能体时代的天花板。
2. 双重许可结构：`packages/server/src/enterprise` 为商业许可，二次分发时须逐目录甄别（`LICENSE.md`）。
3. 前端技术债：主 UI 为无类型 jsx + Redux 旧栈，二次开发体验明显弱于 TS 后端；核心引擎存在 2000+ 行巨型文件，改动风险高。
4. LangChain 深度绑定：resolutions 锁定 `@langchain/core` 1.1.20，上游 API 变动会直接冲击 278 个节点集成。

## 10. 总结评价

Flowise 是"LLM 可视化编排"赛道最成功的开源实现之一：它用一套自描述的节点协议把 LangChain 生态打包成 278 个即插即用积木，用图执行器把画布 JSON 变成支持分支、循环与人机协作的生产级运行时，从 2023 年的 Chatflow 一路演进到嵌入式 Agentflow SDK，55k 星与 25k fork 是其影响力的直接证明。但它最终以官方归档收场——团队在停运公告中给出的判断（推理能力更强的模型让开发者转向编码智能体，刚性低代码触顶）本身就是一个值得整个赛道记取的结论。今天接触这份代码的正确姿势是把它当作**架构教材**：想理解节点型低代码平台如何设计协议、编译画布、执行图、隔离商业代码，`packages/components/src/Interface.ts`、`packages/server/src/utils/buildAgentflow.ts`、`packages/agentflow/ARCHITECTURE.md` 这三个入口的源码质量足以支撑学习；若要在生产中继续使用，则必须接受 EOL 现实并准备自行维护。
