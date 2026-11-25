# chatwoot/chatwoot 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Ruby｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：开源客服/客户互动平台，Intercom、Zendesk 的开源自托管替代品，把网站聊天、邮件、社交 IM 等十余渠道会话统一到一个坐席工作台。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | chatwoot/chatwoot | GitHub |
| 星数 | 39500（快照 2026-10-24）；gh api 实测 39827（2026-11-14） | gh api |
| 主语言 | Ruby（Rails 8.0.5.2）+ Vue 3 前端 | Gemfile、package.json |
| 版本 | v4.19.1（package.json 与最新 release 一致） | package.json |
| 许可证 | 双许可：根目录 MIT + enterprise/ 目录商业许可（Enterprise License） | LICENSE、enterprise/LICENSE |
| 最近推送 | 2026-11-13（分析前一日） | gh api |
| 默认分支 | develop（git-flow，master 为稳定版） | README.md「Branching model」 |
| 创建时间 | 2019-08-14 | gh api |

## 2. 项目是什么，解决什么问题

Chatwoot 是一个多租户（按 Account 隔离）的客服工单/会话平台：企业把网站实时聊天、Email、WhatsApp、Facebook、Instagram、Telegram、Line、SMS/Twilio、Twitter/X、TikTok、通用 API 等接入为不同的 inbox（收件箱），所有渠道消息汇聚成统一的 Conversation，由坐席在一个仪表盘内分配、回复、协作与统计（README.md「Omnichannel Support Desk」；渠道模型见 `app/models/channel/`，共 12 个）。

核心领域模型是 Account → Inbox（多态指向 12 种 Channel）→ Contact → ContactInbox（联系人与渠道会话入口的映射）→ Conversation → Message（`app/models/conversation.rb`、`app/models/inbox.rb`、`app/models/message.rb`）。Conversation 以 `(account_id, display_id)` 唯一编号，用 jsonb 存 custom_attributes/additional_attributes，索引按 `(account_id, inbox_id, status, assignee_id)` 组织，说明查询以坐席视图为中心。

产品能力对齐 Intercom：坐席协作（私信备注、@提及、团队、自动化规则、宏、快捷回复）、外呼运营（Campaigns、CSAT）、自助帮助中心 Portal（文章/分类，`app/models/portal.rb`、`app/models/article.rb`）、报表、以及 AI 坐席 Captain（README「Captain – AI Agent for Support」；OSS 侧放分类器等基础服务 `app/services/captain/`，完整的助手/Copilot/工具调用在 `enterprise/lib/captain/`）。集成生态含 Slack、Dialogflow、Shopify、Linear、Google 翻译等（README「Integrations」）。

商业化采用 GitLab 式 open-core：`enterprise/` 目录（约 200+ 文件）包含 SLA、审计日志、坐席容量策略、SAML、语音通话、Captain 高级能力等付费功能，通过 license 校验（`enterprise/lib/enterprise/chatwoot_hub.rb` 连接 Chatwoot 官方订阅服务 hub.2.chatwoot.com）与 24 个 `premium: true` 特性开关（`config/features.yml`）控制，核心平台仍为 MIT 可自托管。

## 3. 技术栈与依赖

后端（Gemfile）：
- Ruby 3.4.4 + Rails 8.0.5.2，Puma 7.2，PostgreSQL（pg），Redis（redis-namespace）
- Sidekiq 7.3 + sidekiq-cron（后台任务与定时任务，`config/schedule.yml`）
- wisper 2.0（进程内 pub/sub 事件总线，事件驱动的骨架）
- searchkick + opensearch-ruby（可选 OpenSearch 全文检索）；pg_search（文章搜索）、pgvector + neighbor（AI 向量检索）
- 认证授权：devise_token_auth + devise-two-factor + Pundit + JWT；OmniAuth（Google/SAML）
- 渠道 SDK：facebook-messenger、koala（FB）、line-bot-api、twilio-ruby、twitty（Twitter）、gmail_xoauth 等
- AI：ruby_llm、ruby-openai、ai-agents、cld3（语言识别）、firecrawl-sdk；模型路由走 `config/llm.yml`（AGENTS.md「AI Models」）
- 其余：Stripe（云订阅计费）、ActionMailbox（邮件入站）、hairtrigger（DB 触发器）、audited、rack-attack（限流）、datadog/sentry/NewRelic 等 APM（按环境变量加载）

前端（package.json，pnpm + Vite）：
- Vue 3（Composition API + `<script setup>`，见 AGENTS.md 前端规范）、Tailwind、Pinia/Vuex、vue-i18n（Crowdin 翻译，30+ locale）
- 构建用 vite_rails（widget/lib 单独 vite.config.ts、vite.lib.config.ts），vitest 单测，Histoire 组件沙盒
- size-limit 约束 widget 产物 ≤300KB、sdk.js ≤40KB（package.json `size-limit` 段）

规模：Ruby 代码（app/lib/enterprise/db/spec）约 28 万行、3131 个 .rb 文件；前端（app/javascript）约 32.6 万行、1266 个 .vue；214 个 DB 迁移（db/migrate）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| app/models | 116 个 ActiveRecord 模型，含 12 个 Channel:: 子类与 integrations/、shopify/ 子目录 |
| app/controllers | 212 个控制器：api/v1（坐席/管理员 API）、api/v1/widget（访客端 API）、platform/public/enterprise API、super_admin |
| app/services | 310 个服务对象，按领域分包（conversations/、whatsapp/、captain/、imap/ 等），业务逻辑主体 |
| app/jobs | 114 个 Sidekiq 任务（含 ActionCableBroadcastJob、WebhookJob、渠道任务） |
| app/listeners、app/dispatchers | 事件总线两侧：Dispatcher 发布 wisper 事件，12 个 Listener 消费（ActionCable、Webhook、通知、自动化规则等） |
| app/mailboxes | ActionMailbox 邮件入站解析（reply_mailbox、imap/） |
| app/policies、app/builders、app/finders | Pundit 授权、对象构建、查询封装 |
| app/javascript | 多前端单体：dashboard（坐席台）、widget（站点聊天挂件）、portal（帮助中心）、superadmin、survey、sdk、shared、design-system |
| enterprise/ | 企业版覆盖层：EE 模型/控制器/服务 + Captain 提示词与 Copilot 引擎（enterprise/lib/captain/） |
| spec/ | 1085 个 RSpec 文件 + spec/enterprise 397 个 EE 规格镜像 OSS 目录 |
| config/ | features.yml（232 个特性开关，24 个 premium）、llm.yml、schedule.yml、routes.rb |
| docker/、deployment/、swagger/ | Docker 与安装脚本、K8s/Helm、OpenAPI 文档（swagger/paths） |

组织方式：单仓单体 Rails + 同仓多前端（Vite 多 entrypoint：`app/javascript/entrypoints/` 下 dashboard.js/widget.js/portal.js/sdk.js 等）。EE 不分叉：EE 代码集中在 `enterprise/`，通过 Ruby prepend 注入 OSS 类（见第 5 节），OSS/EE 规格也镜像分置。

## 5. 核心架构与关键实现

### 5.1 事件驱动的消息流转（wisper + Listener）

模型回调触发领域事件，Dispatcher 用 wisper 广播，各 Listener 解耦消费（实时推送、webhook、通知、自动化规则、报表）。Message 在 `after_create_commit` 中 dispatch `MESSAGE_CREATED`/`FIRST_REPLY_CREATED`（`app/models/message.rb:402-407`）；`lib/events/types.rb` 集中定义 `conversation.created`、`message.created` 等字符串事件常量。这种"模型回调→事件→多消费者"是全项目的主干模式，AGENTS.md 明确要求优先用既有 job/concurrency 模式而非 Thread.new。

```ruby
# app/dispatchers/base_dispatcher.rb
class BaseDispatcher
  include Wisper::Publisher
  def listeners; []; end
  def load_listeners
    listeners.each { |listener| subscribe(listener) }
  end
end
```

### 5.2 实时层：ActionCable 双通道推送

`app/listeners/action_cable_listener.rb` 把 20+ 种领域事件转成 ActionCable 广播：坐席按 `user.pubsub_token` 定向，访客 widget 会话走 `widget_push_event_data` 精简载荷，同一事件在一次 broadcast 里同时发给坐席 token 列表与 widget token 列表（`broadcast(..., widget: { tokens:, data: })`）。实际落盘推送异步化为 `::ActionCableBroadcastJob.perform_later`（该文件 228 行），避免在事件路径上阻塞。连接鉴权在 `app/channels/application_cable/connection.rb`，widget 访客与坐席共用 `RoomChannel`。

### 5.3 多渠道 Inbox 抽象

12 个 `Channel::` 模型（api/email/facebook_page/instagram/line/sms/telegram/tiktok/twilio_sms/twitter_profile/web_widget/whatsapp，`app/models/channel/`）挂到 Inbox 的多态 `channel` 上；`app/models/inbox.rb` 用 `channel_type == 'Channel::Xxx'` 提供谓词。每个渠道有一组服务对象承接入站/出站：如 WhatsApp 有 30+ 文件（incoming_message_service、embedded_signup_service、media_upload_service、message_dedup_lock 等，`app/services/whatsapp/`），邮件走 ActionMailbox（`app/mailboxes/`）。新增渠道 = 新 Channel 模型 + inbox 谓词 + services 包 + webhook 路由（config/routes.rb 中 whatsapp/instagram/tiktok 等各有命名空间）。

### 5.4 企业版注入机制（GitLab 式 EE overlay）

`config/initializers/01_inject_enterprise_edition_module.rb`（注明源自 GitLab）给所有 Module prepend 了 `prepend_mod_with/include_mod_with`：OSS 类文件末尾写 `ActionCableListener.prepend_mod_with('ActionCableListener')`，EE 侧存在 `Enterprise::ActionCableListener` 时自动 prepend，使 EE 方法能 `super` 回 OSS 实现（实例：`enterprise/app/models/enterprise/conversation.rb` 在 `can_reply?`/`allowed_keys?` 中 super 后扩展通话与 SLA 逻辑）。232 个特性开关（config/features.yml，bigint 位图存储）+ `ChatwootApp.extensions` 共同决定功能可见性，保证 EE 缺席时纯 OSS 可独立运行。

### 5.5 双前端

坐席台（dashboard）与站点挂件（widget）是两套独立 Vue 3 应用，共享 design-system/shared 代码；widget 从 `widget.js` entrypoint 单独构建并由 size-limit 卡 300KB 体积，实时同样连 ActionCable（`app/javascript/widget/helpers/actionCable`），访客端 REST 走 `api/v1/widget` 命名空间（config/routes.rb:568）。帮助中心 portal、内部 superadmin（Administrate）、survey 各有独立 entrypoint。

## 6. 构建与运行

来自 Makefile、AGENTS.md、package.json 与 docker-compose.yaml：

```bash
# 本地开发（Ruby 版本见 .ruby-version = 3.4.4）
make setup                    # bundle install + pnpm install
make db                       # db:chatwoot_prepare（建库/迁移/种子）
pnpm dev                      # 或 overmind start -f Procfile.dev（rails + sidekiq + vite）
# docker-compose 本地编排含 rails/sidekiq/vite/postgres/redis/mailhog

# 测试
bundle exec rspec spec/path/to/file_spec.rb   # 后端 RSpec
pnpm test / pnpm test:watch                   # 前端 vitest（TZ=UTC）

# Lint
bundle exec rubocop <files>   # 150 字符行宽
pnpm exec eslint <files>

# 生产
docker-compose -f docker-compose.production.yaml up   # 或官方 Docker 镜像 chatwoot/chatwoot
```

部署面：Heroku 一键、DigitalOcean 1-Click K8s、Helm chart（artifacthub 徽章）、`nightly_installer.yml` 每夜构建安装器；docker-compose.production.yaml 含 rails/sidekiq/postgres/redis，OpenSearch 为可选组件（Gemfile 注释与 searchkick 按环境变量启用）。

## 7. 测试与工程质量

- 后端：RSpec 1085 个 OSS spec 文件 + 397 个 EE spec（`spec/`、`spec/enterprise/`），覆盖 models/controllers/requests/services/jobs/listeners 全层，含 factories、integration、swagger 契约测试目录；CI 覆盖率开启（`.circleci/config.yml` `COVERAGE: true`）。
- 前端：551 个 .spec.js/ts（vitest），另有 Histoire 组件沙盒（histoire.config.ts）。
- CI/CD：CircleCI（lint + rspec 并行机）为主；GitHub Actions 18 条工作流——run_foss_spec（纯 OSS 规格单独跑，防止误依赖 EE）、run_mfa_spec、migration_version_check、size-limit、publish_foss_docker/publish_ee_docker 双镜像发布、nightly_installer（`.github/workflows/`）。
- 工程规范沉淀极重：AGENTS.md 14KB 的仓库级工程守则（查询要 tenant-scoped、禁 raw SQL 进生产路径、EE 兼容检查清单、依赖健康审计等），并配套 docs/production-reliability.md。质量流程显著高于同类开源 Rails 项目平均水位。

## 8. 社区与维护状态

- 活跃度：gh api 显示最近 push 2026-11-13（分析前一日），open issues 1602，fork 9677。
- 贡献者：约 409 人（gh api contributors 分页统计，含匿名）。公司主体 Chatwoot Inc 主导，核心代码与评审集中在官方团队；README 多处声明 Crowdin 托管 30+ 语言翻译、Discord 社区。
- Release 节奏（gh api releases）：v4.19.1（2026-10-24）、v4.19.0（10-20）、v4.18.0（9-18）、v4.17.0/1（8 月）、v4.16.x（7 月）——约每月一个 minor + 若干 patch，节奏稳定；package.json 版本 4.19.1 与最新 release 同步。
- 云商业化：官方云版 + `enterprise/` 付费功能 + hub.2.chatwoot.com 订阅校验（`enterprise/lib/enterprise/chatwoot_hub.rb`）；Stripe 计费仅服务云订阅（Gemfile「Subscriptions, Billing」）。
- 开发流程：git-flow（develop 为主开发分支）、PR 必须关联 Linear 工单（AGENTS.md「Commits and Pull Requests」）。

## 9. 亮点与风险

亮点：
1. 事件驱动架构干净：模型回调 → wisper 事件 → 12 个 Listener（实时/webhook/通知/自动化/报表）解耦，扩展新能力不侵入主干（`lib/events/types.rb`、`app/listeners/`）。
2. EE 注入机制教科书级：prepend_mod_with 单文件实现（`config/initializers/01_inject_enterprise_edition_module.rb`），OSS/EE 同仓不分叉、纯 OSS 可独立构建发布（run_foss_spec 工作流佐证）。
3. 渠道抽象成熟：12 渠道统一 Inbox/ContactInbox 模型，每个渠道独立服务包，WhatsApp/邮件等复杂协议隔离在 services 内。
4. 前端工程化到位：双前端独立构建、widget 产物 300KB 硬预算、size-limit 入 CI。
5. 测试与规范密度高：1482 个 RSpec 文件 + 551 前端 spec + 详尽仓库级工程守则（AGENTS.md）。

风险/局限：
1. 体量庞大：前后端合计约 60 万行、214 个迁移、232 个特性开关，自托管运维面较宽（Postgres/Redis/Sidekiq/可选 OpenSearch），小团队上手成本高。
2. 双许可认知成本：enterprise/ 商业许可 + 订阅校验，误把 EE 目录当 MIT 使用有合规风险；SLA、审计日志、坐席容量等常用企业能力需付费（config/features.yml premium 段）。
3. 事件依赖模型回调，大扇出路径（消息创建触发的广播/webhook/自动化）在超大会话量下依赖 Sidekiq 队列容量，AGENTS.md 自己也强调"green unit tests do not establish capacity"。
4. Captain 等核心 AI 能力重心在 EE/OSS 混合层，且强依赖外部 LLM 服务与官方 hub，深度自定义为二开成本。

## 10. 总结评价

Chatwoot 是当前开源客服赛道工程完成度最高的项目之一：以 Rails 8 单体承载十二渠道统一收件箱，用 wisper 事件总线 + ActionCable 实现领域事件到实时界面的贯通，用 GitLab 式 prepend 注入把商业化功能干净地悬浮在 MIT 内核之上，再用一套远超平均水准的测试、CI 与工程守则维持约 60 万行代码的演进秩序。它把"Intercom 的功能面"基本补齐（含 AI 坐席 Captain），代价是系统复杂度与运维门槛显著抬高。适合需要数据主权与多渠道整合的中大型客服团队自托管，也适合想学习多租户 Rails 架构、事件驱动设计与 open-core 工程化的开发者精读——建议从 `app/models/conversation.rb`、`app/listeners/action_cable_listener.rb` 与 `config/initializers/01_inject_enterprise_edition_module.rb` 三个文件切入。
