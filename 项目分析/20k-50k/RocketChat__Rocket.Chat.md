# RocketChat/Rocket.Chat 项目分析

> 星数 43500（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：开源团队沟通平台（Slack 的开源替代），集频道聊天、音视频会议、Omnichannel 客服、联邦互通与应用市场于一体的自托管消息系统。

| 项 | 值 |
|---|---|
| 仓库 | RocketChat/Rocket.Chat（默认分支 develop） |
| 星数 | 43500（快照）；gh api 实测 46371（2026-11-05） |
| 语言 | TypeScript（全仓 10912 个 .ts/.tsx，`find apps packages ee` 统计） |
| 许可证 | MIT 主体 + `ee/`、`apps/meteor/ee/` 目录企业版专有许可（根 `LICENSE`、`apps/meteor/ee/LICENSE`） |
| 版本 | develop 为 9.1.0-develop（`package.json`）；最新 release 8.9.0（2026-10-05，gh api） |
| 活跃度 | 最近 push 2026-11-04（gh api），日级活跃 |
| 贡献者 | 头号贡献者 4380 次提交（gh api）；总贡献者数未确认（官方宣传 400+ 核心贡献者） |

## 2. 项目是什么，解决什么问题

Rocket.Chat 自我定位是 "The ultimate secure open-source solution for team communications"（`README.md`），面向对数据主权有高要求的组织（README 举例如 Deutsche Bahn、美国海军），提供可与 Slack/MS Teams 对标的能力集：

- **团队协作**：频道/私聊/线程/讨论/端到端加密消息，实时 WebSocket 推送；
- **Omnichannel 全渠道客服**：网页 Livechat（`packages/livechat` 嵌入式组件）、邮件、社交渠道统一进队列分配坐席（`apps/meteor/server/services/omnichannel/queue.ts`）;
- **音视频**：LiveKit 会议集成（`apps/meteor/package.json` 依赖 `@livekit/components-react`，`server/services/video-conference`）与 SIP 媒体呼叫（`server/services/media-call`）;
- **联邦互通**：基于 Matrix 协议的跨服务器互通（`ee/packages/federation-matrix`，企业版模块）;
- **可扩展**：Apps-Engine 沙箱应用框架 + Marketplace 应用市场（`README.md` "Apps-Engine" 章节、`packages/apps-engine`）。

商业模式为"开放核心"：MIT 许可的核心功能完整可自部署，企业级能力（LDAP/SAML 企业集成、可扩展性、ABAC、审计、FIPS 等）放在 `ee/` 目录以专有许可发布，需订阅授权（`apps/meteor/ee/LICENSE` 明确"仅在生产环境拥有有效 EE 订阅时可用"）。仓库内置 `scripts/fossify.ts`，可一键删除 `ee/` 与 `apps/meteor/ee/` 生成纯 FOSS 发行版，这是其开源/商业化边界的工程化体现。

## 3. 技术栈与依赖

- **运行时/框架**：Node 22.13+/24.15+（`package.json` engines）、Meteor 全栈框架（`apps/meteor/package.json` 的 `meteor.mainModule` 指定 `server/main.ts` 与 `client/meteorShell.ts`，modern 构建）、React ~19.2.8；
- **语言与构建**：TypeScript ~7.0.2、Turborepo 2.x 编排（`turbo.json`）、Yarn 4 workspaces（`apps/* packages/* ee/apps/* ee/packages/*`，根 `package.json`）、客户端构建已迁移 Vite（`apps/meteor/vite/`、`vite/buildForServer.mjs`）；
- **数据层**：MongoDB（driver 6.16.0，根 resolutions 强制钉版）、自研 `packages/mongo-adapter`、`packages/models` 81 个模型；
- **实时通信**：自研 DDP 栈拆为 `packages/ddp-server`（服务端）、`packages/ddp-client`（客户端 SDK）、`packages/streamer`（事件广播）；
- **UI 体系**：自研 Fuselage 设计系统（`packages/fuselage` React 组件库 + `fuselage-tokens/hooks/toastbar/forms`）、`packages/ui-kit`（消息 Block Kit 渲染引擎）、约 30 个 `ui-*` 业务组件包；
- **其他关键依赖**：`@hono/node-server`（HTTP）、Playwright（E2E）、Jest + node:test（单测）、Storybook、OpenTelemetry 全链路追踪（`packages/tracing`）、Bugsnag 错误上报；
- 主应用 dependencies 达 272 个（`apps/meteor/package.json`），根 `resolutions` 有 200+ 条依赖钉版，可见对供应链安全的重度管控。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `apps/meteor/` | 主应用：Meteor 全栈单体，`server/`（启动/API/服务）、`client/`（React 视图）、`ee/`（主应用内企业代码） |
| `apps/uikit-playground/` | UI Kit 交互调试沙盒 |
| `packages/` | 83 个开源工作区包：数据（models/core-typings/rest-typings）、服务（core-services/streamer/ddp-*）、UI（fuselage/ui-kit/ui-*）、应用框架（apps-engine） |
| `ee/apps/` | 企业版微服务可执行体：ddp-streamer、presence-service、queue-worker、account-service、authorization-service、omnichannel-transcript、xmpp-server-service |
| `ee/packages/` | 企业版库：license（授权引擎）、federation-matrix、presence、abac、omnichannel-services、pdf-worker 等 |
| `scripts/` | fossify、发布辅助脚本 |
| `development/`、`docker-compose-*.yml` | 本地开发与 CI 容器编排 |
| `.changeset/` | Changesets 版本发布流程（大量 pending 变更文件，活跃开发证据） |
| `.github/workflows/` | 17 个 CI workflow |

组织方式评述：典型的 **大 monorepo + 分层包体系**。主应用是 Meteor 单体，但业务逻辑已抽为 28 个"服务"（`server/services/`），通过 `packages/core-services` 的 Broker 抽象既可在单体内本地调用（`LocalBroker.ts`），也可在 `TRANSPORTER=nats/TCP` 时拆为独立微服务（`apps/meteor/server/lib/isRunningMs.ts`）。开源/付费代码以目录硬隔离，MIT 部分不 import ee 目录即可构建 FOSS 版。

## 5. 核心架构与关键实现

### 5.1 服务端启动与微服务化 Broker

入口 `apps/meteor/server/main.ts`（52 行）用顶层 await 串起启动链：`configureServer → registerServices（28 个服务）→ startup → startRocketChat（license→联邦）→ startRestAPI → runStartupTasks`，任何失败即 `process.exit(1)`，避免僵尸进程。

服务注册于 `apps/meteor/server/services/startup.ts`：Message、Room、User、Realtime、Omnichannel、Push、VideoConf、AISearch 等 28 个服务统一 `api.registerService()`。Broker 抽象在 `packages/core-services/src/LocalBroker.ts`：单体模式下用 Node EventEmitter 本地路由，微服务模式下由 `ee/packages/network-broker` 经 NATS/TCP 代理到远端节点。判断逻辑仅一行：

```ts
// apps/meteor/server/lib/isRunningMs.ts
export function isRunningMs(): boolean {
	return !!process.env.TRANSPORTER?.match(/^(?:nats|TCP)/);
}
```

同一套代码支持"单容器全功能"与"按服务横向扩容"两种部署形态，是这套架构最大的设计价值。

### 5.2 MongoDB 数据层

`packages/models/src/models/` 共 81 个模型（Rooms、Messages、Users、Livechat*、FederationKeys、OAuth* 等），全部继承 `BaseRaw`（`packages/models/src/models/BaseRaw.ts`）的仓储模式，配合 `proxify.ts` 做模型方法代理与 `updater.ts` 做文档更新封装。类型定义独立在 `packages/core-typings`，REST 契约在 `packages/rest-typings`，实现"模型-类型-接口"三包对齐。

### 5.3 Realtime：自研 DDP + Streamer

Rocket.Chat 没有停留在 Meteor 默认 DDP，而是自研了可独立于 Meteor 运行的实时栈：

- `packages/ddp-server/src/`：DDP 服务端实现——`Publication.ts`/`MeteorCollection.ts`（发布订阅与集合 diff）、`ConnectionRegistry.ts`（连接管理）、`codec.ts`、`methods/presence.ts`（在线状态）；
- `packages/streamer/src/streamer.module.ts`：`Streamer` 抽象类，基于 `eventemitter3`，提供 `_allowRead/_allowWrite/_allowEmit` 三级规则（默认 `allowRead('none')`+`allowWrite('none')`，安全默认值），管理订阅集合并支持跨实例 retransmit；
- `packages/streamer/src/StreamPresence.ts` + `StatusVisibilityGate.ts`：在线状态广播与隐私可见性门控（隐私开关企业能力相关）;
- `packages/ddp-client/src/DDPSDK.ts`、`ClientStream.ts`：客户端统一 SDK，`legacy/`、`livechat/` 目录服务不同前端载体；
- 企业版 `ee/apps/ddp-streamer` 可把 WebSocket 流量单独拆进程，配合 `apps/meteor/server/services/realtime/service.ts` 通过 `user.forceLogout` 等事件保持多进程会话一致。

### 5.4 Apps-Engine 应用框架

`packages/apps-engine`（MIT，独立 npm 包，v1.68.0）定义应用的公开 API 与类型（`definition/`）；运行时在 `packages/apps`（存储、管理器、事件触发）；主应用通过 `apps/meteor/server/services/apps-engine/service.ts` 桥接，订阅 `presence.status`、`apps.added` 等内部事件转发给应用沙箱，并处理"集群中应用须在足够节点可用"的调度（`AppsEngineNoNodesFoundError`）。这是对标 Slack App 的生态入口。

### 5.5 授权引擎与企业模块门控

`ee/packages/license/src/` 实现授权证书校验（`license.ts`、`applyLicense.ts`、气隙环境限制 `AirGappedRestriction.ts`）。企业功能以"模块"粒度声明在 `packages/core-typings/src/license/LicenseModule.ts` 的 `CoreModules`（28 项：`ldap-enterprise`、`livechat-enterprise`、`voip-enterprise`、`scalability`、`federation`、`abac`、`fips`、`device-management` 等），各功能点用 `hasModule()` 门控（`ee/packages/license/src/modules.ts`）。启动时 `apps/meteor/startRocketChat.ts` 顺序执行 `startLicense() → enforceFipsLicense() → startFederationMatrixService()`。

## 6. 构建与运行

来自根 `package.json` 与 `apps/meteor/package.json`（未实际运行，静态确认命令存在）：

```bash
# 依赖：Node >=22.13 <23 或 >=24.15 <25，Yarn 4.18.1
yarn install
yarn dev          # turbo 并行启动主应用开发模式
yarn build        # turbo run build 全量构建
yarn testunit     # 单元测试（turbo 编排）
yarn lint         # oxlint + stylelint

# 主应用内（apps/meteor/）：
yarn start        # meteor 启动
yarn testapi      # REST API 集成测试（node --test）
yarn test:e2e     # Playwright E2E
yarn typecheck    # meteor lint + tsc --noEmit
yarn ms           # TRANSPORTER=TCP 微服务模式开发

# 生成纯 FOSS 源码树：
yarn fossify      # scripts/fossify.ts，删除 ee/ 与 apps/meteor/ee/
```

部署面向 Docker/Kubernetes/气隙环境（`README.md` "Deploy Rocket.Chat"），MongoDB 为唯一外部硬依赖。

## 7. 测试与工程质量

- **规模**：全仓 885 个 `*.spec.ts`/`*.test.ts`（`find apps packages ee` 统计）；主应用测试分四层——Jest 单测（`.testunit:jest`）、node:test 服务端单测带 nyc 覆盖率（`.testunit:server:cov`）、REST API 集成测试（`tests/end-to-end/api/`，174 个测试文件）、Playwright E2E（`tests/e2e/`）；
- **CI**：`.github/workflows/` 共 17 个 workflow，按类型拆分（ci-test-unit / ci-test-e2e / ci-test-storybook / ci-test-visual-regression / ci-code-check），另有 CodeQL 安全扫描、codecov 覆盖率（根 `codecov.yml`）；
- **质量观察**：oxlint（Rust 实现）替代 ESLint 主力（根 devDependencies）；TypeScript 严格化程度高（大量 `definition/` 类型目录与 workspace:^ 严格版本约束）；UI 有 Storybook + 视觉回归测试；`packages/icons` 视觉回归与别名机制完备。短板是仓库体积巨大（gh 报告 size 2.2GB），构建与测试成本高，深链依赖（Meteor、200+ resolutions 钉版）维护负担明显。

## 8. 社区与维护状态

- **活跃度**：gh api 显示最近 push 2026-11-04（前一天），4341 个 open issue，develop 版本号 9.1.0-develop，`.changeset/` 目录有数十个未发布变更文件——日级高频迭代；
- **发布节奏**：最新 release 8.9.0（2026-10-05），按月度节奏发版（`release.yml` workflow 自动化，`packages/release-changelog` 定制 changelog 生成）；
- **贡献结构**：HISTORY.md（6.x 时代）显示贡献分"社区贡献者"与"Core Team"两类；机器人账号（rocketchat-github-ci、kodiak 自动合并配置 `.kodiak.toml`）深度参与；当前贡献者总数未确认（tarball 无 git 历史）；
- **工程文化**：CLAUDE.md（AI 辅助开发指引）、`.claude/skills/`、`.devcontainer/`、Gitpod 配置齐备，对新贡献者友好。

## 9. 亮点与风险

**亮点**

1. **单体/微服务同源架构**：28 个服务经 Broker 抽象一处代码两种部署形态（`packages/core-services` + `ee/apps/*`），成长路径平滑，是同类自托管 IM 中少见的工程深度；
2. **实时栈完全自研可控**：DDP 服务端/客户端/Streamer 三包拆分，脱离 Meteor 黑盒，可单独扩容 WebSocket 网关（`packages/ddp-server`、`ee/apps/ddp-streamer`）；
3. **开源/商业化边界工程化**：`fossify.ts` 脚本 + 目录硬隔离 + 模块化授权引擎，MIT 社区版功能完整可用，边界清晰无"渐进阉割"争议（根 `LICENSE`、`scripts/fossify.ts`）；
4. **应用生态框架成熟**：Apps-Engine 同时是开源 npm 包与市场运行时，事件/持久化/UI Block 全链路（`packages/apps-engine`、`packages/ui-kit`）；
5. **测试与 CI 体系完备**：四层测试 + 视觉回归 + CodeQL，885 个测试文件。

**风险/局限**

1. **复杂度门槛极高**：主应用 272 个依赖、83 个 workspace 包、Meteor+Vite 双构建体系，二次开发与自建成本陡峭；
2. **核心能力向 EE 收拢的趋势**：联邦（Matrix）、presence 无限规模、ABAC、FIPS 等高价值能力均在 `ee/`（`CoreModules` 清单），MIT 版长期竞争力依赖社区意愿；
3. **Meteor 框架路径依赖**：虽已 Vite 化客户端并自研 DDP，但服务端仍深度绑定 Meteor 生态（`meteor build`、Meteor 方法/发布订阅），人才池相对小；
4. **许可历史变动遗留疑虑**：HISTORY.md 停更于 2023 年 6.x 时代，且 `LICENSE` 的 "NOASSERTION"（GitHub 无法识别为纯 MIT）要求使用者在合规审查时自行区分目录许可。

## 10. 总结评价

Rocket.Chat 是开源团队沟通领域工程化程度最高的项目之一：4.6 万星、1.1 万个 TS 文件的 monorepo 里同时装着单体可用性与微服务可扩展性，MIT 核心 + 目录级 EE 边界 + fossify 脚本构成了清晰且可执行的开放核心模式，自研 DDP/Streamer 实时栈与 Apps-Engine 应用框架体现了从"Slack 替代品"到"通信平台底座"的野心。代价是极高的系统复杂度与对 Meteor 生态的路径依赖。适合需要数据主权、多形态部署（气隙/合规场景）的中大型组织选型使用；适合想学习"如何把 Meteor 单体演进为可拆分微服务""如何工程化划分开源/商业边界"的工程师通读 `server/services/` 与 `packages/core-services`。
