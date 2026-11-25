# Dokploy/dokploy 项目分析

> 星数 38490（gh api 核实于 2026-11-14）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：开源的自托管 PaaS（Vercel / Netlify / Heroku 的替代品），通过 Docker + Traefik 让任何 VPS 变成应用与数据库的部署平台。

| 项目 | 事实 |
| --- | --- |
| 名称 | Dokploy/dokploy |
| 星数 | 38490（gh api 实测，2026-11-14） |
| 主语言 | TypeScript（另有少量 Go：监控 agent `apps/monitoring`） |
| 许可证 | Apache-2.0 为主 + 专有条款的 open-core 双许可（`LICENSE.MD` + `LICENSE_PROPRIETARY.MD`，GitHub 识别为 NOASSERTION） |
| 版本 | v0.31.2（`apps/dokploy/package.json`，release 于 2026-11-13） |
| 最近活跃 | 默认分支 canary 最近推送 2026-11-13（gh api 实测），活跃度极高 |
| 创建时间 | 2024-04-19（gh api），约 30 个月达到 38.5k 星 |
| 贡献者 | 约 390 人（gh api contributors 分页 last=390） |
| 规模 | monorepo 内 TS/TSX 源文件约 1143 个，UI 组件 417 个（本地统计） |

## 2. 项目是什么，解决什么问题

Dokploy 是一个部署在自己 VPS 上的 Web PaaS：装一条命令（`curl -sSL https://dokploy.com/install.sh | bash`，README），即可获得一个图形化控制台，用于部署应用（任意语言）、数据库（MySQL/PostgreSQL/MongoDB/MariaDB/Redis/libsql）、以及 Docker Compose 项目（README「Features」与 `apps/dokploy/server/api/routers/` 中 postgres.ts、mongo.ts、compose.ts 等 50+ 路由对应）。

它解决的核心痛点是：Heroku/Render 等托管平台价格高、数据在他方；而纯手工管 VPS 又缺 UI、缺构建流水线、缺域名/证书/备份等配套。Dokploy 把这些封装为「面板 + 底层 Docker/Traefik 编排」，用户保有数据与服务器的完全控制权。面向的典型场景是独立开发者与小团队的 VPS 自托管（README 自述 "Self-Hosted: Self-host Dokploy on your VPS"）。

功能面覆盖完整部署生命周期：Git 源码拉取（GitHub/GitLab/Gitea/Bitbucket/自定义 Git，`packages/server/src/services/git-provider.ts` 等）、8 种构建策略、容器编排、Traefik 域名与证书、数据库备份到外部存储（`services/backup.ts`）、部署通知（Slack/Discord/Telegram/邮件，`services/notification.ts`）、多服务器远程部署（`services/server.ts` + SSH）、Docker Swarm 多节点（README「Multi Node」）、实时监控与日志（Go 采集 agent + WebSocket 日志/终端流）。

商业模式为 open-core：Apache-2.0 覆盖主体代码，`packages/server/src/services/proprietary/`（audit-log、forward-auth、license-key、sso、whitelabeling）与部分云功能走专有许可（`LICENSE.MD` 头部明确按目录划分）；同时提供托管版 Dokploy Cloud（README「Try the Dokploy Cloud」，代码中 `IS_CLOUD` 开关出现在 106 个文件、Stripe 计费路由 `apps/dokploy/pages/api/stripe/`）。

增长背景：仓库 2024-04 创建，恰逢 Coolify 等带动的一波「自托管 PaaS」热潮；Dokploy 凭借更现代的 UI（shadcn/ui + tRPC）与快速迭代（近 3 天连发 v0.31.2/v0.31.1，gh api releases）在竞争中达到 38.5k 星。此为基于时间数据的客观陈述。

## 3. 技术栈与依赖

- 前端/全栈框架：Next.js（Pages Router，`apps/dokploy/pages/`；构建命令 `next build --webpack`，`apps/dokploy/package.json`），React 18 + TanStack Query/Table/Virtual
- API 层：tRPC 11 + `@trpc/server`，并用 `@dokploy/trpc-openapi` 从 tRPC 过程生成 OpenAPI（`openapi.json`、`scripts/generate-openapi.ts`、Swagger 页 `pages/swagger.tsx`）
- 鉴权：better-auth 1.6.23（含 passkey / SSO / SCIM / API-key 插件，`apps/dokploy/package.json` dependencies）
- ORM/数据库：Drizzle ORM + PostgreSQL（`packages/server/src/db/schema/`，30+ 张表 schema；迁移 SQL 在 `apps/dokploy/drizzle/`）
- 容器与远程执行：dockerode（本地 + SSH 远程 Docker API）、ssh2（远程命令）、node-pty（交互终端）、docker CLI（shell 调用）
- 反向代理：Traefik（动态配置文件由代码生成，`packages/server/src/utils/traefik/`）
- 任务队列：自研内存队列（`apps/dokploy/server/queues/in-memory-queue.ts`）+ BullMQ/ioredis（云版定时任务 `apps/schedules`）+ Inngest（`apps/api`）
- 监控：Go（gofiber）编写的指标采集 agent（`apps/monitoring/main.go`），日志/指标外发走 Vector（`packages/server/src/services/logs-and-metrics/vector-agent.ts`）
- AI 能力：Vercel AI SDK，接 OpenAI/Anthropic/Azure/Cohere/Mistral/Ollama 等多家（`@ai-sdk/*` 依赖 + `packages/server/src/utils/ai/`）
- 工程工具：pnpm 10 monorepo、Biome 2.5（lint+format，`biome.json`）、esbuild（server 打包）、Vitest（测试）、Node 24（`package.json` engines）

## 4. 目录结构与代码组织

pnpm workspace monorepo（`pnpm-workspace.yaml`：apps/api、apps/dokploy、apps/schedules、packages/server 四个包）。

| 目录 | 职责 |
| --- | --- |
| `apps/dokploy/` | 主应用：Next.js Pages Router 前端 + tRPC API + 自定义 HTTP 服务器 + WebSocket 服务 + 部署队列（`server/server.ts`、`server/api/`、`server/wss/`、`server/queues/`） |
| `packages/server/` | 核心业务包 `@dokploy/server`：60 个服务模块（services/）、构建/数据库/Docker/Traefik 等工具集（utils/）、DB schema（db/schema/）、企业版专有逻辑（services/proprietary/） |
| `apps/monitoring/` | Go 语言监控 agent：采集主机/容器指标并回调 Dokploy（`main.go`、README） |
| `apps/schedules/` | 云版定时任务服务：BullMQ + Hono（`package.json`） |
| `apps/api/` | 云版辅助 API：Hono + Inngest 工作流（`package.json`） |
| `scripts/` | 开发辅助脚本（worktree 并行开发等） |
| `Dockerfile*` | 5 个 Dockerfile：主程序、server、cloud、monitoring、schedule 变体（根目录） |

组织方式评述：业务逻辑几乎全部下沉到 `packages/server` 供各 app 共享，`apps/dokploy` 同时承担 UI、tRPC、WebSocket 与队列 worker（自托管单容器形态）；云版通过 `IS_CLOUD` 环境变量在同一代码库中分叉行为（106 处引用，如 `packages/server/src/constants/index.ts:5`），并拆出 api/schedules 独立进程。这是「一套代码、自托管/云双形态」的典型 open-core 结构。UI 侧 417 个组件集中在 `apps/dokploy/components/`，页面按 dashboard 路由组织（`pages/dashboard/`）。

## 5. 核心架构与关键实现

### 5.1 服务器入口：Next.js + 6 个 WebSocket 服务

自托管形态下 Dokploy 不是纯 Next 应用，而是自定义 HTTP server 同时挂载 Next 请求处理与 6 个 WebSocket 服务（`apps/dokploy/server/server.ts`）：部署日志流（listen-deployment）、容器日志（docker-container-logs）、容器终端（docker-container-terminal，node-pty）、宿主终端（terminal）、抽屉日志（drawer-logs）、容器统计（docker-stats）。启动时还初始化 Traefik 默认配置、目录结构、Cron 任务，并恢复被中断的 Traefik 配置（`restoreInterruptedTraefik`）。容器日志按 URL 参数接收 containerId/tail/search/since，并先经 `canAccessDockerOverWss` 授权与 `containerBelongsToService` 归属校验（`apps/dokploy/server/wss/docker-container-logs.ts`）。

### 5.2 Docker 集成：dockerode + shell 双轨，远程经 SSH

Docker 操作采用「API + CLI」混合：本地与远程容器 API 走 dockerode，远程时按 serverId 查出 SSH 密钥并构造 `protocol: "ssh"` 的 dockerode 客户端（`packages/server/src/utils/servers/remote-docker.ts`）：

```ts
export const getRemoteDocker = async (serverId?: string | null) => {
  if (!serverId) return docker;
  const server = await findServerById(serverId);
  if (!server.sshKeyId) throw new Error("No SSH key available for this server");
  const dockerode = new Dockerode({
    host: server.ipAddress, port: server.port, username: server.username,
    protocol: "ssh",
    sshOptions: { privateKey: server.sshKey?.privateKey, ...sshKeepalive },
  });
  return dockerode;
};
```

镜像拉取、构建等则直接拼 shell 命令经 `spawnAsync("docker", ["pull", ...])` 或 `execAsync` 执行（`packages/server/src/utils/docker/utils.ts` 的 `pullImage`）。所有服务函数都带可选 `serverId` 参数，内部按 `execAsyncRemote`（ssh2 Client，`packages/server/src/utils/process/execAsync.ts`）或本地 `exec` 分叉——这就是「多服务器远程管理」的底层实现：Dokploy 只在控制面持有 SSH 私钥，数据面动作推送到目标服务器执行。容器编排侧基于 Docker Swarm：自托管启动时初始化 swarm（`packages/server/src/setup/server-setup.ts`），服务以 swarm service 方式部署（`packages/server/src/utils/builders/index.ts` 引用 `CreateServiceOptions`）。

### 5.3 部署流水线与 8 种构建策略

应用部署主流程 `deployApplication`（`packages/server/src/services/application.ts:178`）：创建 deployment 记录 → 按来源类型拼装「clone 仓库 + 应用补丁 + 构建命令」的 shell 脚本 → 整体重定向到部署日志文件 `(command) >> logPath 2>&1` 在本地或远程执行 → `mechanizeDockerContainer` 创建/更新容器 → 状态与通知更新，失败时写错误日志并发通知。部署请求先进内存队列排队（`apps/dokploy/server/queues/deployments-queue.ts` + `in-memory-queue.ts`），保证并发可控。

构建策略在 `packages/server/src/utils/builders/` 下有 8 种：nixpacks、railpack、heroku(buildpacks)、paketo、docker-file、static、drop（直接上传 zip）、compose；入口 `getBuildCommand`（`utils/builders/index.ts`）按应用配置选择并生成命令，同时做路径/参数注入防护（`assertSafeBuildFields` 校验 `isSafeBuildPath`/`BUILD_TOKEN_REGEX`）。Compose 部署则直接生成 `docker compose -p <app> -f <path> up -d` 命令并用 `env -i PATH="$PATH"` 清洗环境后执行（`packages/server/src/services/compose.ts:490`）。

### 5.4 Traefik 动态路由与域名管理

每个应用的域名被生成为一个 Traefik 动态配置 YAML 写入文件系统：`createTraefikConfig` 构造 router/service/middleware 结构后 `writeFileSync` 到 `DYNAMIC_TRAEFIK_PATH`，远程服务器则经 `writeFileRemote` 下发（`packages/server/src/utils/traefik/application.ts`）。同目录还覆盖 domain/middleware/redirect/security/forward-auth/web-server 配置生成，证书申请走 Traefik ACME 或自有 certificate 服务（`apps/dokploy/server/api/routers/certificate.ts`）。启动时 `restoreInterruptedTraefik` 修复因重启导致的配置丢失（`apps/dokploy/server/server.ts`）。

### 5.5 备份、监控与企业版门控

数据库备份支持 mysqldump/pg_dump 等并推送到 S3 类目的地（`packages/server/src/services/backup.ts`、`utils/backups/`）；监控由 Go agent 采集、经 WebSocket 推给面板（`apps/monitoring/main.go`），日志/指标可外发 Grafana Loki、Datadog 等 10 种后端（`packages/server/src/services/logs-and-metrics/`，vector-agent 下发）。企业能力（SSO/SCIM/审计日志/白标/forward-auth）集中在 `packages/server/src/services/proprietary/`，由 `hasValidLicense` 查询用户表的 `enableEnterpriseFeatures`/`isValidEnterpriseLicense` 门控（`services/proprietary/license-key.ts`）——专有代码本身也在本仓库可见，但许可受 `LICENSE_PROPRIETARY.MD` 约束。

## 6. 构建与运行

来自根 `package.json`、`apps/dokploy/package.json` 与 README：

```bash
# 安装（目标 VPS 上的自托管形态，README）
curl -sSL https://dokploy.com/install.sh | bash

# 本地开发（monorepo，需 PostgreSQL；pnpm 10 + Node 24）
pnpm install
pnpm dokploy:setup        # 初始化 env + 数据库迁移（apps/dokploy setup.ts）
pnpm dokploy:dev          # tsx 直接跑 server/server.ts（Next dev 模式）

# 构建与启动
pnpm build                # 递归构建所有包
pnpm dokploy:build        # build-server（esbuild）+ build-next
pnpm dokploy:start        # 跑迁移后启动 dist/server.mjs

# 测试 / 质量
pnpm test                 # vitest（apps/dokploy/__test__/vitest.config.ts）
pnpm typecheck            # 各包 tsc --noEmit
pnpm format-and-lint      # biome check .

# 数据库
pnpm --filter=dokploy migration:generate / migration:run / studio   # drizzle-kit

# OpenAPI 与 Docker 镜像
pnpm generate:openapi     # 生成 openapi.json（tRPC → OpenAPI）
./apps/dokploy/docker/build.sh [canary]
```

## 7. 测试与工程质量

- 测试规模可观：`apps/dokploy/__test__/` 下 181 个 `*.test.ts` 文件，覆盖 deploy、backups、traefik、wss、queues、permissions、export-import 等约 35 个子域（本地统计），框架为 Vitest。
- CI 共 11 个 workflow（`.github/workflows/`）：pull-request 检查、deploy、monitoring、hotfix-release/cherry-pick、upgrade-integration-test（升级集成测试）、sync-openapi-docs（OpenAPI 文档同步）等，说明发布与回归自动化程度较高。
- 代码质量工具链统一为 Biome（替代 ESLint+Prettier），提交遵循 Conventional Commits（`CONTRIBUTING.md`）；`package.json` 中有大量 pnpm overrides 固定传递依赖版本，并注释了曾启用的 `minimumReleaseAge` 供应链防护（`pnpm-workspace.yaml`），反映对依赖安全的持续关注。
- 观察到的弱点：`next.config.mjs` 设置 `typescript.ignoreBuildErrors: true`，生产构建不拦截类型错误；UI 组件目录（components/）体量大但测试主要覆盖服务层。

## 8. 社区与维护状态

- 活跃度极高：gh api 显示默认分支 canary 最近推送 2026-11-13（分析前一天），open issues 817。
- 发布节奏快：近期 release 为 v0.31.2（2026-11-13）、v0.31.1（2026-11-12）、v0.31.0（2026-11-05），约一周一个 minor 版本（gh api releases）。
- 贡献者结构：contributors 分页尾页为 390，约 390 名贡献者；README 顶部由作者 Siumauricio 维护 GitHub Sponsors，社区渠道为 Discord（README widget）。
- 商业实体已成型：版权主体为 "Dokploy Technology, Inc."（`LICENSE.MD`），配合 Dokploy Cloud 与企业专有模块，属于公司化开源运营。
- 开发流程细节：`scripts/` 提供并行 worktree 开发脚本（spawn-agent-worktree.sh 等），暗示多 agent/多人并行开发模式（用途未在文档中明说，未确认）。

## 9. 亮点与风险

亮点：
1. 架构分层干净：核心逻辑集中在 `packages/server` 纯 TS 包，UI/API/WebSocket/队列薄壳化，云与自托管共用一套业务代码（`IS_CLOUD` 开关）。
2. Docker/SSH 双轨设计务实：dockerode API（含 SSH 远程）+ shell CLI 混用，既能精细控制容器又能复用 docker compose 全部能力（`packages/server/src/utils/servers/remote-docker.ts`、`services/compose.ts`）。
3. 部署生命周期完整：8 种构建策略、Git 多平台、补丁注入、备份、通知、监控、日志外发、AI 助手一应俱全，且多数实现可直接读源码复用。
4. 工程质量高：181 个测试文件、11 条 CI 流水线、Biome 统一风格、依赖供应链防护意识（pnpm overrides）。
5. API 体系三合一：tRPC 内部 API、OpenAPI 导出（swagger.tsx）、CLI/API 外部访问，兼顾面板与自动化。

风险/局限：
1. 安全面广：控制面持有目标服务器 SSH 私钥并通过 SSH 下发 shell 命令，面板被攻破等于所有受管服务器被攻破（架构固有风险，`utils/process/execAsync.ts`）；代码虽有 `assertSafeBuildFields` 等注入防护，但 shell 拼接面仍然很大。
2. Windows/非 Linux 支持缺失：深度依赖 Linux 上的 Docker Swarm、Traefik 文件挂载与本地 exec，只能在 Linux 服务器/容器内运行。
3. `typescript.ignoreBuildErrors: true`（`next.config.mjs`）降低了类型护栏；`apps/dokploy/templates/templates.ts` 在源码包中为空文件，模板内容依赖外部仓库，离线可用性存疑。
4. open-core 边界随版本演进：企业功能（SSO/审计/白标）虽代码可见但许可受专有条款限制，商用集成前需仔细核对 `LICENSE_PROPRIETARY.MD`。

## 10. 总结评价

Dokploy 是目前自托管 PaaS 赛道中工程完成度最高的 TypeScript 实现之一：用 Next.js + tRPC 做控制面，用 Docker（Swarm/dockerode/CLI）+ Traefik + SSH 做数据面，把 Heroku 式体验完整搬到用户自己的 VPS 上，并以 open-core + Cloud 的方式可持续商业化。代码库规模大（1100+ TS 文件）但组织清晰，测试与 CI 覆盖扎实，维护节奏（日级提交、周级发版）在同体量项目中罕见。适合想彻底掌控部署数据的个人开发者与小型团队直接使用；适合研究「PaaS 内部如何工作」（构建流水线、Traefik 动态路由、远程 Docker 管理、部署队列）的开发者精读 `packages/server/src` 与 `apps/dokploy/server`，其分层与双形态（self-host/cloud）代码组织本身就是一个很好的开源商业化范本。
