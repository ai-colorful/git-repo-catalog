# mastodon/mastodon 项目分析

> 星数 50436（快照 2026-10-24）｜主语言 Ruby｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：Mastodon 是基于 ActivityPub 的去中心化（联邦式）社交网络服务端，是 Fediverse 生态的旗舰实现——任何人都可以自建实例，并通过开放协议与其他实例乃至不同软件实现互操作。

| 项 | 值 |
| --- | --- |
| 名称 | mastodon/mastodon（服务端 + Web 前端单仓） |
| 星数 | 50,436（快照） |
| 主语言 | Ruby（Rails 8.1）+ TypeScript/React 19 前端 + Node.js 流媒体服务 |
| 许可证 | AGPL-3.0（`LICENSE`，gh api 确认） |
| 最近活跃 | push 2026-11-03；最新版 v4.7.5 发布于 2026-11-03（CHANGELOG.md / gh api releases） |
| 维护版本 | 同日发布 v4.7.5 / v4.6.11 / v4.5.21 / v4.4.28，并行维护 4 条发布线 |
| 贡献者 | ~404 人（gh api contributors 分页推断），fork 7537，open issues 4626 |
| 核心作者 | Eugen Rochko（Gargron），2016 年发起（`AUTHORS.md`、`LICENSE`） |

## 2. 项目是什么，解决什么问题

中心化社交平台存在单一公司控制数据、规则与用户命运的问题。Mastodon 的答案是"自托管 + 联邦"：每个组织/社区运行自己的服务器（实例），用户账号形如 `username@domain`（README.md "Part of the Fediverse" 段），实例之间通过 W3C ActivityPub S2S 协议互通，关注、转发、回复可以无缝跨越实例边界，甚至与非 Mastodon 的 ActivityPub 软件互通（README.md 第 21 行）。

功能层面，它是一个完整的微博客产品：实时时间线、媒体附件（图/视频/GIF）、投票、引用（FEP-044f 合意式引用，`FEDERATION.md` 第 16 行）、话题标签、列表、隐私分级（public/unlisted/private/limited/direct）、屏蔽/静音/过滤词、举报与治理后台（README.md Features 段 + `app/services/fan_out_on_write_service.rb` 的可见性分支）。安全与治理是其强调点：服务端内置管理员界面、域名封锁、趋势审核队列（`app/workers/admin/`、`config/sidekiq.yml` 中的 trends 相关调度）。

对第三方开放是另一支柱：Mastodon 自身充当 OAuth2 Provider（`Gemfile` 中的 `doorkeeper`），对外提供 REST API（`config/routes/api.rb` 的 v1/v2/v1_alpha 命名空间）与 Streaming WebSocket API（`streaming/index.js`），由此形成了独立客户端生态（README.md "OAuth2 and a straightforward REST API" 段）。

商业模式上由非营利实体 Mastodon gGmbH 维护，翻译走 Crowdin（`crowdin.yml`），是"数字公共产品"认证项目（README.md 徽章）。作为 Fediverse 事实标准客户端，Mastodon 的实现细节（如 HTTP 签名兼容策略、大小限制表 `FEDERATION.md`）深刻影响整个生态的互操作实践。

## 3. 技术栈与依赖

后端（`Gemfile`）：
- Ruby 4.0（`.ruby-version`）+ Rails ~8.1；Web 服务器 puma，资源管线 propshaft，模板 haml-rails
- PostgreSQL（`pg`，含 pghero 监控）；Redis（`redis` 5 + hiredis）做缓存、锁、feed 与 Sidekiq 队列
- 异步体系：sidekiq < 9 + sidekiq-bulk + sidekiq-scheduler + sidekiq-unique-jobs
- 认证/授权：devise + devise-two-factor + webauthn、doorkeeper（OAuth2）、pundit（策略对象，`app/policies/` 45 个）、omniauth 全家桶（LDAP/CAS/OIDC/SAML）
- ActivityPub 协议栈：json-ld、json-ld-preloaded、rdf-normalize、json-canonicalization、linzer（HTTP Message Signatures）；图片处理 ruby-vips + kt-paperclip，对象存储 aws-sdk-s3 / fog-openstack
- 搜索：chewy（Elasticsearch 封装，索引定义在 `app/chewy/`）；熔断 stoplight；限流 rack-attack；迁移保护 strong_migrations；可观测性 prometheus_exporter + opentelemetry 全家桶

前端（`package.json`）：
- React 19.3.1 + Redux Toolkit 2.13 + react-redux + Immutable.js（历史包袱与新版共存）、react-intl（i18n）、react-router 5
- 构建 vite 8.3.5（已从 webpack 迁出）、测试 vitest、Storybook + Chromatic 视觉回归、tesseract.js（图片 OCR）
- 共 83 个运行时依赖、58 个开发依赖

流媒体服务（`streaming/`，独立 Node.js 服务）：原生 http/ws 实现的 WebSocket 推送服务（`streaming/index.js` 1478 行），直接用 pg 查询校验 OAuth token（`streaming/index.js:383`），经 Redis pub/sub 接收 Rails 侧事件后推给客户端。

部署面：Dockerfile（多阶段，13.7KB）、docker-compose.yml、Helm chart（`chart/`，另见 mastodon/chart 仓库）、Heroku/Scalingo 配置（Procfile、scalingo.json）、.devcontainer、Vagrantfile。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `app/models` | 149 个领域模型；核心 `account.rb`(615 行)、`user.rb`(591)、`status.rb`(502)，concerns 细分到 `app/models/concerns/`（含 account/user/status 子目录） |
| `app/services` | 82 个服务对象（业务用例层）：发帖、关注、时间线、域封锁、搜索解析等；`app/services/activitypub/` 21 个专管联邦收发 |
| `app/lib` | 领域库：`feed_manager.rb`（feed 算法）、`activitypub/`（活动基类/签名/序列化）、`fasp/`（辅助服务协议）、`annual_report/` 等 |
| `app/workers` | 66 个顶层 + `activitypub/` 24 个 + `admin/`、`scheduler/` 等子目录的 Sidekiq 任务 |
| `app/controllers` | 面板/Web 控制器 + `api/`（REST）、`activitypub/`（收件箱/出件箱等 S2S 端点）、`admin/`、`well_known/`（WebFinger/nodeinfo） |
| `app/serializers` `app/presenters` `app/policies` | active_model_serializers 序列化、视图呈现、pundit 授权三件套 |
| `app/chewy` | Elasticsearch 索引：accounts/statuses/public_statuses/tags/instances |
| `app/javascript` | React 前端 922 个 JS/TS 文件共约 10.9 万行，features/ 按页面组织 |
| `streaming/` | 独立 Node.js WebSocket 服务 |
| `spec/` | RSpec 1121 个 spec 文件（models/services/workers/controllers/requests/system 等 20+ 子目录） |
| `db/migrate` | 548 个迁移；`db/schema.rb` 1768 行 |
| `config/routes` | 路由拆分为 api/admin/settings/fasp/web_app 五块（`config/routes.rb:182-240`） |
| `lib/mastodon` | 框架扩展与 CLI（`lib/mastodon/cli/`） |
| `chart/` `dist/` `public/` | Helm chart、nginx 配置等部署产物、静态资源 |

组织方式评述：典型的"胖 Rails 单体 + 前端子应用 + 独立 Node 服务"三进程架构（web / streaming / sidekiq，见 `Procfile`）。业务逻辑严格收敛在 Service 对象与 Worker 中，控制器薄；联邦协议代码按 lib（协议原语）/ services（编排）/ workers（投递）/ controllers（端点）四层分置，边界清晰。前端虽已引入 Redux Toolkit，但大量存量代码仍基于 Immutable.js + 旧版 redux 模式，处于渐进迁移状态。

## 5. 核心架构与关键实现

### 5.1 联邦收件链路：Inbox → ProcessActivityService → Activity 工厂

S2S 活动入口是 `app/controllers/activitypub/inboxes_controller.rb`：强制 `require_actor_signature!`、超过 `MAX_JSON_SIZE`（1MB，`app/lib/activitypub/activity.rb:8`）直接 413，随后 `head 202` 把处理交给后台——HTTP 层只做验证与快速应答。

入站编排在 `app/services/activitypub/process_activity_service.rb`，两处设计值得注意：
1. **按需 JSON-LD 压缩**：只有 payload 带 `signature` 时才做昂贵的 JSON-LD compaction（源码注释解释：LD 签名验证本身需要 JSON-LD，且提前压缩可防"外部 context 文档替换攻击"），这在其 4.7.5 安全公告（JSON-LD compaction DoS）前后仍是核心防御逻辑。
2. **双重签名验证**：转发活动（actor ≠ 提交者）先验 Linked-Data Signature（`app/lib/activitypub/linked_data_signature.rb`），失败再尝试 FEP-8b32 Object Integrity Proof（`app/lib/activitypub/object_integrity_proof.rb`）。

验证通过后进入工厂分发（`app/lib/activitypub/activity.rb:24-65`），17 种活动类型各对应一个类（`app/lib/activitypub/activity/` 下 create/announce/follow/undo/quote_request 等 16 个文件），如 `SUPPORTED_TYPES = %w(Note Question)`、`CONVERTED_TYPES = %w(Image Audio Video Article Page Event)`（后者会被转换为 Note 处理）。

### 5.2 出站投递：FanOutOnWriteService + DeliveryWorker

发帖主流程 `app/services/post_status_service.rb`：幂等键保护（`with_idempotency`）、媒体校验、定时/立即分支，随后 `fan_out_on_write_service.rb` 分发——本地收件人（粉丝/列表/提及/会话按可见性分流）写入 Redis feed，公开内容再广播到 hashtag 与公共流（Redis pub/sub 供 streaming 服务消费）。

跨实例投递由 `app/workers/activitypub/delivery_worker.rb` 承担：队列 `push`、`retry: 16, dead: false`，自定义指数退避 + 抖动（`count**4 + 15 + rand(0.5*count**4)`），并叠加 stoplight 熔断（60s 冷却/10 次阈值）与 `DeliveryFailureTracker` 可用性记账，失败实例会进入退避状态避免雪崩式重试。这是典型的"高扇出 + 不可靠网络"容错设计。

### 5.3 Feed 算法与 Sidekiq 体系

`app/lib/feed_manager.rb` 用 Redis 有序集合存 home/list 时间线：`MAX_ITEMS = 800` 截断、`REBLOG_FALLOFF = 80`（近 80 条内已转发过则不再插入重复转发），过滤规则按 home/public 等时间线类型分派（`feed_manager.rb:1-50`）。时间线是"写扩散"模型，配合 `Scheduler::VacuumScheduler` 等清理任务回收不活跃用户 feed。

队列权重在 `config/sidekiq.yml`：`[default,8] [push,6] [ingress,4] [mailers,2] [pull] [scheduler] [fasp]`——联邦收发（push/ingress）优先级高于拉取（pull），`fasp` 是 4.x 新增的 Fediverse 辅助服务（搜索/趋势外置）队列。sidekiq-scheduler 注册了十几个周期任务：趋势刷新、搜索索引（`Scheduler::IndexingScheduler` 每分钟）、数据库 vacuum、关注推荐、用户/IP 清理等，构成完整的自运维体系。

### 5.4 模型与搜索

`app/models/account.rb` 是全库最重的模型（615 行）：本地/远程账号统一存储（`domain` 为空即本地），携带 URI/inbox/public_key 等联邦字段、治理状态（silenced/suspended）与趋势审核字段（`schema` 注释头，annotaterb 维护）。`app/models/status.rb` 502 行，自关联（reblog/reply/quote）+ mentions/media/tags 关联。全文搜索走 Chewy（`app/chewy/statuses_index.rb` 等 5 个索引），索引开关尊重 FEP-5feb 的"搜索索引同意"机制。

## 6. 构建与运行

来自 README.md 与 package.json/Gemfile 的实际命令：

```bash
# 前置依赖：Ruby 4.0、PostgreSQL 14+、Redis 7.0+、Node.js 24+、FFmpeg 5.1+（README "Requirements"）
cp .env.production.sample .env.production   # 生产配置模板
bundle install && yarn install
rails db:setup                               # 建库 + schema + seeds（db/seeds.rb）
foreman start -f Procfile.dev                # 本地起 web/streaming/webpack(vite)（Procfile.dev、.foreman）

# 生产
yarn build:production                        # vite 前端构建（package.json scripts）
RAILS_ENV=production rails db:migrate
node ./streaming/index.js                    # 流媒体服务（package.json start）
sidekiq -C config/sidekiq.yml                # 后台队列
docker compose up                            # 或 docker-compose.yml 一键

# 测试 / 质量门
yarn test                                    # = lint + typecheck + vitest run（package.json）
bundle exec rspec                            # Ruby 测试（.rspec、spec/）
yarn typecheck && yarn lint                  # tsc --noEmit / eslint+stylelint
bundle exec rubocop                          # lint-ruby.yml CI 同款
```

## 7. 测试与工程质量

- **测试规模**：RSpec 1121 个 spec 文件、约 10.1 万行（实测 `find spec -name "*_spec.rb" | wc -l` 及行数统计），覆盖 models/services/workers/controllers/requests/system/policies/serializers/search 等全部层次；前端 vitest（`vitest.config.mts`，legacy-tests 与 storybook 两个 project）+ Storybook/Chromatic 视觉回归 + BrowserStack 真机测试（README 声明）。
- **CI**：`.github/workflows/` 共 22 条流水线——test-ruby、test-js、test-migrations（迁移可重复执行校验）、lint-ruby/js/css/haml、format-check、codeql、bundler-audit、容器镜像构建（含 nightly 与 security 变体）、crowdin 翻译同步、chromatic。门禁相当完备。
- **质量工具链**：rubocop（`.rubocop/` 分层配置）、haml-lint、oxfmt、eslint+stylelint+lint-staged+husky；`strong_migrations` 守护大表迁移；`.simplecov` 覆盖率配置。
- **安全工程**：CHANGELOG 近两版（4.7.5/4.7.4）各带多条 GHSA 安全公告与依赖升级，安全响应节奏密集；配合 codeql、bundler-audit 与自定义 payload 限制（1MB 上限、字段截断表 `FEDERATION.md:80-100`）。

## 8. 社区与维护状态

- **活跃度极高**：gh api 显示 pushed_at 为 2026-11-03（分析前一天）；CHANGELOG.md 显示 4.7.5 于 2026-11-03 发布，4.7.4 于 2026-10-29——发版以天计。
- **多版本并行维护**：同一日发布 v4.4.28 / v4.5.21 / v4.6.11 / v4.7.5 四个版本（gh api releases），v4.4.0 EOL 标注 2026-12-17（`FEDERATION.md:49`），版本支持政策明确。
- **贡献者结构**：约 404 名贡献者（gh api 分页），`AUTHORS.md` 长达 70KB；核心提交者以 ClearlyClaire、Gargron 等全职维护者为主，社区 PR 流入量大（`CONTRIBUTING.md` 指向集中式贡献指南）。翻译由 Crowdin 社区完成（`crowdin.yml`、README 徽章）。
- **治理实体**：Mastodon gGmbH（德国非营利）主导，捐赠/赞助入口在 joinmastodon.org（README.md）。issue 存量 4626（gh api），对 50k 星项目属正常水位。

## 9. 亮点与风险

亮点：
1. **联邦协议工程化典范**：签名验证（draft-cavage → RFC9421 rsa/ed25519/ml-dsa-44 渐进升级）、LD-Signature + FEP-8b32 双通道、JSON-LD 按需压缩防 DoS（`app/services/activitypub/process_activity_service.rb`、`FEDERATION.md:46-74`），兼容性与安全性平衡做得极为细致。
2. **异步投递的容错设计**：16 次指数退避 + 自定义抖动 + stoplight 熔断 + 失败率记账（`app/workers/activitypub/delivery_worker.rb`），在高扇出不可靠网络下依然稳定。
3. **完整的自运维后台**：sidekiq-scheduler 十余个周期任务覆盖趋势、索引、清理、推荐（`config/sidekiq.yml`），中小实例开箱即可长期运行。
4. **测试与 CI 门禁齐全**：双端 20 万行测试代码、22 条 CI 流水线、迁移专项测试，质量基线远超同体量开源项目。
5. **多版本支持纪律**：四条发布线同日发安全补丁，兼顾自托管用户的升级惰性。

风险/局限：
1. **写扩散时间线模型的规模化压力**：每帖写所有粉丝的 Redis zset（`fan_out_on_write_service.rb`），超大型实例（mastodon.social）依赖大量工程优化，自托管中小实例资源规划反而友好，但超大单实例成本高。
2. **历史技术债**：前端 Immutable.js + Redux 旧模式与新 Redux Toolkit 并存（`app/javascript/mastodon/`），react-router 5 已落后两个大版本；kt-paperclip 为社区维护的 legacy 分支（`Gemfile`）。
3. **部署复杂度**：单实例需同时运维 Rails + PostgreSQL + Redis + Elasticsearch（可选）+ Sidekiq + Node streaming 六类进程，`docker-compose.yml` 与 Helm chart 缓解但门槛仍在。
4. **Ruby 单体性能上限**：Web 层依赖 Ruby 运行时，热点路径（时间线渲染、收件箱处理）大量依赖 Redis/DB 优化与缓存旁路，横向扩展需要多台 Sidekiq 分队列。

## 10. 总结评价

Mastodon 是开源社交基础设施里少见的"产品、协议、运维"三者都做到位的项目：它以一个严格的 Rails 单体承载了 ActivityPub 联邦的全部复杂性，用服务对象 + Sidekiq 队列 + Redis feed 的经典组合撑起全球数万个实例，同时以罕见的纪律维护多版本发布与安全响应。代码分层清晰（lib/services/workers/controllers 四层联邦架构尤其值得学习），测试与 CI 是行业标杆水平；代价是体量庞大、前端技术栈新旧并存、部署面较宽。适合三类人：想自建或运维 Fediverse 实例的管理员（用 Dockerfile/chart）、研究分布式社交协议与联邦工程的开发者（读 `app/services/activitypub/` 与 `FEDERATION.md`）、以及想学习大规模 Rails 异步架构的工程师（读 `app/workers/` 与 `config/sidekiq.yml`）。对于想参与 Fediverse 生态或构建兼容客户端的开发者，它既是参考实现也是事实标准。
