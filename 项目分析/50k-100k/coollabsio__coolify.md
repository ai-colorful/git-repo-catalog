# coollabsio/coolify 项目分析

> 星数 63540（快照 2026-10-24）｜主语言 PHP｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：开源自托管的 Heroku/Netlify/Vercel 替代品——一条 SSH 连接 + Docker，就能在自己的服务器上部署应用、数据库与 360+ 一键服务。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | coollabsio/coolify | README.md |
| 星数 | 63540（任务快照） | 任务参数 |
| 主语言 | PHP 8.4（Laravel 12），前端 Livewire/Alpine/Tailwind | composer.json、TECH_STACK.md |
| 许可证 | Apache-2.0 | LICENSE |
| 默认分支 | main | gh api |
| 最近推送 | 2026-10-31（分析当天） | gh api repos/coollabsio/coolify |
| 最新 release | v4.5.3（2026-10-28 发布）；versions.json 标注 4.5.4、nightly 4.6-rc.1 | gh api、versions.json |
| 贡献者 | 约 412 人 | gh api contributors |

## 2. 项目是什么，解决什么问题

Coolify 解决的是"部署不想到厂商那里租房"的问题：它把 PaaS 的核心体验（推代码即部署、自动 HTTPS、数据库托管、预览环境、日志与终端）搬到任意一台能 SSH 上去的服务器上——README.md 明确写"VPS、裸金属、树莓派，任何接受 SSH 连接的机器都行"。与一般 DevOps 工具的关键差异是它强调"配置留在你自己的服务器上"，停用 Coolify 后已部署的资源仍然独立运行。

能力面覆盖完整 PaaS 链路（README.md）：从 GitHub/GitLab/Bitbucket/Gitea 拉代码，用 Nixpacks、Railpack、Dockerfile、Docker Compose 或现成镜像构建；托管 PostgreSQL/Redis/MySQL/MongoDB 等数据库（`app/Models/Standalone*.php` 共 9 种）；一键部署 388 个 compose 服务模板（`templates/` 下 388 个 yaml + service-templates.json）；自动域名/HTTPS/反向代理、健康检查；定时任务、数据库与卷备份、通知（邮件/Discord/Slack/Telegram/Pushover，见 `app/Models/*NotificationSettings.php`）。

商业形态是双轨的：自托管版免费开源，Coolify Cloud 是托管收费版——同一套代码里包含 Stripe 计费（`app/Actions/Stripe/`、`app/Models/Subscription.php`）、Horizon 多队列云化配置（`config/horizon.php` 中专门的 cloud 队列池）。

开发者入口齐备：325 条 REST API 路由声明（`routes/api.php`，1.2MB OpenAPI 规格 `openapi.json`）、CLI、以及基于 laravel/mcp 的 AI 代理接口（`app/Mcp/Tools/` 提供 Deploy、CreateApplication、CancelDeployment 等工具）——这是 2026 年很典型的"AI 原生"演进。

## 3. 技术栈与依赖

后端（composer.json，权威；注意 TECH_STACK.md 写 Laravel 11 已过时）：

- PHP 8.4 + Laravel 12（`laravel/framework ^12.70.1`）
- Livewire 3.8（全站 UI，272 个组件）+ Alpine.js/Tailwind（TECH_STACK.md）
- 队列：laravel/horizon（Redis 驱动）；实时推送：laravel/reverb（WebSocket）
- 认证：fortify、sanctum、socialite + 7 个第三方 IdP 插件（Authentik、Clerk、Zitadel、Azure、Google、Discord、Infomaniak）
- SSH：phpseclib/phpseclib（`app/Helpers/SshMultiplexingHelper.php` 做连接复用）
- 数据：doctrine/dbal、spatie/laravel-activitylog（部署日志落库）、spatie/laravel-data、schemaless-attributes
- API：zircote/swagger-php 生成 OpenAPI；laravel/mcp 提供 MCP Server
- 计费：stripe/stripe-php；对象存储：flysystem-aws-s3；cron 解析：poliander/cron

前端（package.json）：Vite + Svelte（局部组件）+ Tailwind 4 时代插件、`@xterm/xterm`（Web 终端，`docker/coolify-terminal/` + `app/Livewire/Terminal/`）、playwright（浏览器测试）。

运行时镜像：`docker/production/Dockerfile` 基于 serversideup/php 8.4-fpm-nginx-alpine，内嵌 Nginx、postgres 客户端、Node 24、cloudflared；TECH_STACK.md 注明用 s6-overlay 做进程监督。数据库 PostgreSQL 15、Redis 7。

## 4. 目录结构与代码组织

| 目录 | 职责 | 规模证据 |
| --- | --- | --- |
| app/ | Laravel 分层主体（1063 个 PHP 文件，11MB） | find 统计 |
| app/Livewire | 全部管理界面（272 组件：Project/Server/Database/Team/Settings…） | 同上 |
| app/Jobs | 异步任务（70 个）：部署、备份、清理、服务器巡检 | 同上 |
| app/Actions | 命令式业务动作（100 个，按 Application/Database/Server/Proxy/Service/Shared 等 15 个子域） | 同上 |
| app/Models | 73 个 Eloquent 模型（Application、Service、Standalone* 数据库族、Server、Team…） | 同上 |
| app/Mcp | MCP Server 的 89 个文件（Tools/Prompts/Resources） | 同上 |
| app/Http | 控制器与 API（94 个；API 控制器按资源族拆分） | 同上 |
| bootstrap/helpers | 全局函数库：shared.php 6399 行、parsers.php 3677 行，合计约 1.9 万行 | wc -l |
| database/ | 487 个迁移，最近一条 2026-10-29 | ls database/migrations |
| routes/ | web 298 条、api 325 条声明，另有 webhooks/console/ai/channels | grep -c |
| templates/ | 388 个一键服务 compose 模板 + JSON 目录 | find 统计 |
| docker/ | production/helper/terminal/development/testing-host 镜像构建 | ls docker |
| tests/ | 1377 个测试文件（Pest） | find 统计 |
| resources/views | 513 个 Blade 视图 | find 统计 |

组织方式评述：单体 Laravel 应用（非 monorepo），但"周边"多仓库协作——helper 镜像、sentinel 监控代理、traefik 版本都由 versions.json 统一钉版本（versions.json 中 sentinel 1.1.3、helper 1.0.19、traefik 9 个版本线），说明 Coolify 是"主控面板 + 每台目标服务器上的轻量 agent（helper/sentinel）"的分布式结构。

## 5. 核心架构与关键实现

### 5.1 部署流水线：ApplicationDeploymentJob（核心中的核心）

`app/Jobs/ApplicationDeploymentJob.php` 单文件 6565 行，是全仓库最大的类，负责一次应用部署的完整生命周期。`handle()` 依次做：取消状态复查 → 环境变量键名校验/注册表镜像校验/多服务器警告 → 标记 IN_PROGRESS 并记录所在 Horizon worker → 写私钥到文件系统 → `docker network inspect` 建立 host↔IP 映射（`addHostFlags()`）→ `selectBuildServer()`（多服务器时挑选构建机）→ `detectBuildKitCapabilities()` → 进入 `decide_what_to_do()` 分发（第 594 行）：

```php
private function decide_what_to_do()
{
    if ($this->restart_only) {
        $this->just_restart(); return;
    } elseif ($this->application->build_pack === 'dockerimage') {
        $this->deploy_dockerimage_buildpack();
    } elseif ($this->pull_request_id !== 0) {
        $this->deploy_pull_request();
    } elseif ($this->application->dockerfile) {
        $this->deploy_simple_dockerfile();
    } elseif ($this->application->build_pack === 'dockercompose') {
        $this->deploy_docker_compose_buildpack();
    } elseif ...
```

共 9 条路径：重启复用、现成镜像、PR 预览、纯 Dockerfile、docker compose、构建型 Dockerfile、静态站点、Nixpacks、Railpack。各路径把构建脚本写到远端 `/artifacts/`（BUILD_SCRIPT_PATH、thegameplan.json 等常量，第 55-71 行），支持 BuildKit 缓存、构建指纹复用（`docker_compose_build_fingerprint()`，跳过无变化镜像的重建）、推/拉 Docker registry（多服务器部署经由 registry 中转，`pull_image_for_additional_server()`）。任务 `tries=1`、`timeout=3600`（第 95-99 行），PR 状态通过 `ApplicationPullRequestUpdateJob` 回写。

### 5.2 远程执行抽象：同步与排队两条通道

所有对目标服务器的操作走 SSH，代码里是两层（`bootstrap/helpers/remoteProcess.php`）：

- 同步通道 `instant_remote_process()`（第 219 行）：即时执行短命令并返回输出，内建 SshMultiplexingHelper 连接复用（`app/Helpers/SshMultiplexingHelper.php`，501 行，ControlMaster 复用 TCP 连接）与 SshRetryHandler 重试；非 root 服务器自动用 `parseCommandsByLineForSudo()` 注入 sudo（`bootstrap/helpers/sudo.php`）。
- 排队通道 `remote_process()` → `App\Jobs\CoolifyTask` → `app/Actions/CoolifyTask/RunRemoteProcess.php`：长任务（部署/备份）入队，逐行读远端进程输出写入 spatie activitylog，实时经 Reverb 广播给前端日志面板；`SensitiveDataRedactor` 在落库前脱敏私钥等敏感串（类头 `UNFINISHED_PRIVATE_KEY_PATTERN`）。

任务中提到的"Instant scaler"：HEAD 中没有名为 scaler 的组件（全库 grep "scale" 仅命中常量与 timescale 字样，未确认存在独立伸缩器）。与之对应的能力由三处实现：`just_restart()` 免构建重启；compose 部署的镜像指纹复用跳过重建；以及 API 对单个 compose 服务执行 `docker compose up --no-deps` 的即时单服务操作（`app/Http/Controllers/Api/ServiceApplicationsController.php:555` 的 action_start 描述，同文件对 Swarm 明确返回 501 "Swarm not supported"）。

### 5.3 队列体系：Horizon 多池隔离

`config/horizon.php` 自注释了设计意图："Coolify Cloud: one fixed-size pool per queue, so a busy queue cannot starve"——每个队列独立固定 worker 池，另有 `maintenance` 池专门跑慢速远程 Docker 操作、`webhooks` 池处理 GitHub PR 状态回写；自托管模式合并为单一 supervisor（第 269-278 行）。定时侧（`app/Console/Kernel.php`）：ServerManagerJob 每分钟巡检服务器（onOneServer 防重）、备份缺失检查、每日清理 unreachable servers 与孤儿 PR 容器——这是典型的"控制面自身也要被守护"的自愈设计。

### 5.4 领域模型与服务层

`app/Models/` 73 个模型清晰刻画了资源树：Team→Project→Environment→（Application | Service | Standalone*Database | 其他），Server 承载 Destination（StandaloneDocker/SwarmDocker），PrivateKey 关联服务器；周边横切面有 SslCertificate、ScheduledTask/ScheduledDatabaseBackup、GithubApp（源码托管集成）、CloudProviderToken（Hetzner/DigitalOcean/Hostinger，`app/Services/` 对应厂商 service）。`app/Services/` 60 个类里还能看到 Secret Manager 集成（DopplerService、InfisicalService）、DNS 管理（app/Services/Dns/、ManagedDnsRecord 模型）与 Sentinel 指标客户端。

## 6. 构建与运行

- 一键安装自托管：`curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash`（README.md，脚本本体在 scripts/install.sh 可先审阅）
- 本地开发（DEVELOPMENT.md）：装 Docker（WSL2/Orbstack 均有步骤）→ fork 仓库 → 配 .env → `docker compose -f docker-compose.yml -f docker-compose.dev.yml up` 启动依赖与面板；`docker-compose.dev.yml` 等多个 dev 变体（含 multi/side-by-side）在根目录
- 前端构建：`npm run dev`（vite --host）、`npm run build`（package.json scripts）
- 测试：`vendor/bin/pest`（phpunit.xml / tests/Pest.php），浏览器测试走 phpunit.dusk.xml + playwright/pest-plugin-browser；dev 依赖 serversideup/spin 提供容器化测试环境（composer.json require-dev）
- 生产镜像：`docker/production/Dockerfile`（多阶段：composer install → nginx 换源安装 → serversideup/php 基底）；helper 镜像 `docker/coolify-helper/Dockerfile`
- API 规格：`openapi.yaml`/`openapi.json` 由 swagger-php 注解生成（控制器内 `#[OA\Post]` 属性可直接证实）

## 7. 测试与工程质量

- 规模：1377 个测试 PHP 文件，其中 Feature 961、Unit 356（find 统计），另有 tests/v4/ 为新一代测试（Browser/ + Feature/，含 AuthorizationTest、BackupSortTest 等）与 tests/Scripts/ 基础设施。以 Pest 框架组织。
- 风格：以 Feature 级回归为主，大量安全向命名（ApiTokenExpirationTest、ActivityMonitorCrossTeamTest、AdvisorySecurityRegressionTest、AllResourceRestartLimitsTest），说明团队把越权与令牌生命周期当一等公民；单元/集成比例不高。
- CI（.github 被 tarball export-ignore，以下来自 gh api contents/.github/workflows/）：19 个工作流，含 coolify-main-build.yml（主镜像构建）、coolify-release.yml 与 coolify-rc-release.yml（正式/RC 双轨，配套 coolify-next-build.yml + sync-main-to-next.yml 的 main→next 同步）、coolify-helper.yml、generate-changelog.yml（git-cliff，根目录 cliff.toml 印证）。
- 有个少见的质量闸门：pr-quality.yaml 引入 peakoss/anti-slop@v0，强制 conventional PR 标题、模板章节、限制 emoji 与 AI 署名（blocked-terms 直接列了 "Generated with Claude Code"），并做账号年龄/叉数反垃圾。
- 代码质量观察：Pint（pint.json）、PHPStan、Rector（rector.php、rector-laravel）齐备；但存在明显的"巨型文件"风格——ApplicationDeploymentJob 6565 行、bootstrap/helpers/shared.php 6399 行全局函数。

## 8. 社区与维护状态

- 活跃度极高：仓库最后推送 2026-10-31（分析当天），最新迁移 2026-10-29，release v4.5.3 于 2026-10-28 发布（gh api）。按版本号与 versions.json 双轨（stable 4.5.4 / nightly 4.6-rc.1）推断处于高频迭代期。
- 贡献者约 412 人（gh api contributors?per_page=1 统计分页），open issues 324。商业模式（Cloud 订阅 + 赞助商，README 列出 SerpAPI 等）支撑全职团队，issue/PR 管理高度自动化（stale 管理、PR 分支管理、changelog 生成等 chore 工作流）。
- 文档体系完整：DEVELOPMENT.md（分平台的环境搭建）、RELEASE.md、SECURITY.md 与 SECURITY_ADVISORY.md（后者说明有安全公告披露实践）、DESIGN.md、AGENTS.md（AI 协作指引，配套 .claude/.agents skills 目录）。CHANGELOG 内容与 .github 一样被 export-ignore，历史节奏未确认。

## 9. 亮点与风险

亮点：

1. 单仓库交付完整 PaaS：Web UI + REST API + CLI + MCP 四套入口共用一个 Laravel 核心，且通过 helper/sentinel agent + versions.json 钉版实现"面板轻、目标机更轻"的分布式控制面。
2. 部署引擎成熟：9 条 buildpack 路径、BuildKit 能力探测、构建指纹跳过重建、registry 中转多服务器部署、PR 级预览环境与镜像回滚（GetRollbackImages.php），细节覆盖度接近商业产品。
3. 运维自愈面广：Horizon 按队列隔离 worker 池、每分钟 ServerManagerJob 巡检、每日清理任务、备份缺失补偿检查（Console/Kernel.php），是难得把"控制面自身可靠性"当主题的开源项目。
4. 安全工程认真：29 个 Policy、审计事件模型、敏感数据脱敏（RunRemoteProcess/SensitiveDataRedactor）、面向越权与令牌的回归测试群、SECURITY_ADVISORY 披露机制。
5. 测试规模可观（1377 个文件）且配有 anti-slop PR 闸门，对 60k+ 星项目的社区治理是务实选择。

风险/局限：

1. 巨类与全局函数：6565 行的 ApplicationDeploymentJob 和 1.9 万行 bootstrap/helpers 全局函数是事实上的"上帝模块"，改动与测试成本高（出处见第 4/5 节）。
2. 文档漂移：TECH_STACK.md 仍写 Laravel 11，实际 composer.json 已是 12；changelogs/ 内容不随 tarball 发布。
3. 编排边界：不支持 Swarm（API 返回 501），Kubernetes 不在路线内；伸缩主要靠 compose 单服务操作与多服务器复制，缺乏原生自动伸缩器（"instant scaler"组件在 HEAD 未发现，未确认）。
4. 部署面暴露面大：以 root SSH + 远端 docker 为前提，拥有托管几百台服务器的能力，误配置的影响半径大，依赖其脱敏/审计/Policy 层的正确性。

## 10. 总结评价

Coolify 是"开源 PaaS"赛道目前完成度最高的作品之一：用一个单体 Laravel 应用吃下了面板、API、队列、SSH 编排和计费，再用轻量 helper/sentinel agent 补齐目标机侧能力，其 63540 星来自"一条命令把自己变成 Heroku"的强叙事和持续数年的高密度迭代。读源码建议直奔三条线：`app/Jobs/ApplicationDeploymentJob.php`（部署编排全景）、`bootstrap/helpers/remoteProcess.php` + `app/Helpers/SshMultiplexingHelper.php`（远程执行抽象）、`config/horizon.php` + `app/Console/Kernel.php`（控制面自愈）；想借鉴工程治理的可以看它的 OpenAPI/MCP 双 API 体系与 anti-slop PR 门禁。适合自托管重度用户、Laravel 中大型项目架构参考者，以及研究"PaaS 控制面如何用最小技术栈做全"的人；不适合需要 K8s 原生编排或强多租户隔离的团队。
