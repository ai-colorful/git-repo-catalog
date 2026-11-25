# makeplane/plane 项目分析

> 星数 61335（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：开源项目管理工具（Jira/Linear 替代品），提供工作项、周期（Cycles）、模块、视图与实时协作文档。

| 事实 | 内容 |
| --- | --- |
| 名称 | makeplane/plane（公司主体 Plane Software Inc.） |
| 星数 | 61335（另有 forks 6279、open issues 1091，`gh api repos/makeplane/plane`） |
| 主语言 | TypeScript（前端）+ Python（Django 后端），仓库根 `package.json` 标注 AGPL-3.0 |
| 版本 | 仓库版本 1.5.0（根 `package.json`），最新 release v1.4.3（2026-10-28） |
| 默认分支 | preview；创建于 2022-11-19，最近 push 2026-11-02（极活跃） |
| 规模 | 全仓 TS/TSX/JS + Python 合计约 41 万行（本次统计） |
| 部署形态 | Docker Compose / Kubernetes 自托管 + Plane Cloud |

## 2. 项目是什么，解决什么问题

Plane 是一个面向团队的开源项目管理系统，对标 Jira 与 Linear：以"工作项（Work Items，即 Issue）"为核心单元，向上组织出项目（Project）与工作区（Workspace）两级结构，并提供周期（Cycles，类似 Sprint 的燃尽图迭代）、模块（Module，按功能切分）、视图（View，自定义过滤器）、页面（Pages，实时协作文档）与收件箱（Intake）等概念。README（`README.md`）将其概括为"track issues, run cycles, and manage product roadmaps without the chaos of managing the tool itself"。

与 Jira 的差异点在于：Plane 是 AGPL-3.0 开源可自托管的，官方提供 Docker Compose 与 Kubernetes 两条部署路径（`docker-compose.yml`、`deployments/`），实例管理员可通过 "God mode"（`apps/admin`）配置实例设置。商业模式的云端版（Plane Cloud）与自托管版共用同一代码库。

架构上它是一个典型的多端产品：一个 Django/DRF 的 REST API 后端（`apps/api`），多个前端应用——主工作台 `apps/web`、实例管理 `apps/admin`、对外发布的"公开站点" `apps/space`（发布页面/文档，见 `plane/db/models/deploy_board.py` 的发布看板模型），以及一个独立的实时协作服务 `apps/live`（基于 Yjs/Hocuspocus，为富文本编辑器与页面提供多人协同）。

值得注意的一个事实核对：社区常把 Plane 前端记为 Next.js，但当前 HEAD 快照中 `apps/web` 已是 **React Router 8 + Vite** 的 SPA（`apps/web/react-router.config.ts`、`apps/web/vite.config.ts`、依赖 `react-router` catalog 版本对应 8.3.0），并非 Next.js；monorepo 根用 pnpm workspace + Turborepo 编排（`pnpm-workspace.yaml`、`turbo.json`）。

## 3. 技术栈与依赖

**后端（apps/api，Python 3.12）**（`apps/api/pyproject.toml`）：
- Django 5.2.17 + djangorestframework 3.18.3，数据库 PostgreSQL（psycopg 3，生产用 psycopg-c 编译版），缓存/锁 Redis（django-redis）
- Celery 5.5.3 + django_celery_beat/results（后台任务与定时任务），消息队列 RabbitMQ（docker-compose 中 `plane-mq`）
- 依赖管理用 uv（`apps/api/uv.lock`，uv 版本锁定 required-version），Python 侧 lint 用 ruff（`apps/api/ruff.toml`）

**前端（TS，Node >=22.22，pnpm 11）**（根 `package.json`、`pnpm-workspace.yaml` catalog）：
- React 19 + React Router 8 + Vite；状态管理 **MobX**（`apps/web/store/` 下 30+ 个 store 文件）
- 富文本编辑器自研并独立成包 `packages/editor`（约 2.16 万行），协作底座 Yjs/Hocuspocus（`@hocuspocus/server/provider/extension-*` 3.4.4）
- 组件基元来自官方发布的 npm 包 `@makeplane/propel`，组合组件在 `packages/blocks`（`AGENTS.md` 有明确约定）
- 工具链：Turborepo、oxlint/oxfmt（Lint/格式化）、husky + lint-staged、react-doctor；live 服务端采用 Effect-TS（`@effect/platform-node`）

## 4. 目录结构与代码组织

pnpm workspace 声明 `apps/*` 与 `packages/*`（排除 `apps/api`、`apps/proxy`，后者非 node 包）：

| 目录 | 职责 | 规模（代码行） |
| --- | --- | --- |
| `apps/api` | Django/DRF 后端 + Celery 任务 + 全部 REST API | 735 文件 / 12.1 万行（Python） |
| `apps/web` | 主前端工作台（React Router 8 + Vite + MobX） | 1850 文件 / 17.5 万行 |
| `apps/live` | 实时协作服务器（Hocuspocus/Yjs） | 56 文件 / 6.3 千行 |
| `apps/admin` | 实例管理员 UI（God mode） | 97 文件 / 7.4 千行 |
| `apps/space` | 对外公开发布站点（发布页面/站点） | 167 文件 / 1.17 万行 |
| `apps/proxy` | Caddy 反向代理配置 | 配置文件 |
| `packages/editor` | 自研富文本编辑器（含实时协作） | 226 文件 / 2.16 万行 |
| `packages/blocks` | 跨应用共享的组合 UI 组件 | 278 文件 / 3.0 万行 |
| `packages/services` | 前端 API 客户端服务层 | 59 文件 / 3.8 千行 |
| `packages/types` / `utils` / `shared-state` / `constants` | 共享类型、工具、MobX 状态基座、常量 | 合计约 2.3 万行 |
| `packages/i18n` | 国际化文案（locales，含符号链接目录） | 16 文件 |
| `packages/codemods` | 大版本迁移 codemod | 5 文件 |
| `deployments/` | Helm/K8s 部署清单 | — |

组织方式评述：标准的"多应用 + 共享包" monorepo，前后端各自独立成 app；前端按"editor/blocks（组件）→ services（API 客户端）→ store（MobX）→ app（页面）"分层，与后端 REST 资源一一对应（如 `apps/web/services/cycle.service.ts` 对应 `plane/app/urls/cycle.py`）。依赖版本统一走 pnpm catalog，避免多包漂移。

## 5. 核心架构与关键实现

### 5.1 数据模型：工作区 → 项目 → 工作项的状态机（apps/api/plane/db/models/）

共 **102 个 Django 模型类**（`grep "^class" plane/db/models/*.py` 统计），核心链路为 `Workspace → Project → Issue`，外围模型高度拆分：仅 `issue.py` 就有 19 个类（Issue、IssueAssignee、IssueLabel、IssueLink、IssueAttachment、IssueActivity、IssueComment、IssueSequence、IssueSubscriber、IssueReaction、IssueVote、IssueVersion 等）。

两个有代表性的设计：

1) **项目内序号与快照版本**。每个 Issue 在项目内有递增 `sequence_id`（人类可读的编号），通过 `IssueSequence` 表记录历史：

```python
# apps/api/plane/db/models/issue.py (Issue.save)
last_sequence = IssueSequence.objects.filter(project=self.project).aggregate(
    largest=models.Max("sequence")
)
self.sequence_id = last_sequence + 1 if last_sequence else 1
...
IssueSequence.objects.create(issue=self, sequence=self.sequence_id, project=self.project)
```

同时 `IssueVersion`（issue.py）用 `ArrayField(models.UUIDField())` 冗余存储 state/assignees/labels 等 UUID 快照，配合 `issue_version_sync.py`、`issue_description_version_task.py`（`plane/bgtasks/`）实现编辑历史回溯。

2) **状态机与"triage"隔离**。`state.py` 定义五+一状态组 `StateGroup`（backlog/unstarted/started/completed/cancelled + triage），内置默认状态序列号以 10000 间隔排布（15000/25000/...），为用户在任意两态之间插入新状态留白。默认 Manager 直接排除 triage 组，只有专门的 `TriageStateManager` 能看到——即"待分拣"队列对普通查询透明。

```python
# apps/api/plane/db/models/state.py
class StateManager(SoftDeletionManager):
    """Default manager - excludes triage states"""
    def get_queryset(self):
        return super().get_queryset().exclude(group=StateGroup.TRIAGE.value)
```

模型层还统一接入 `SoftDeletionManager` 软删除与 `ChangeTrackerMixin` 变更追踪（`plane/db/mixins.py`；issue.py 的 `Issue._sync_completed_at` 在状态切到 completed 组时回填完成时间）。

### 5.2 API 分层（apps/api/plane/app/）

标准 DRF 分层但按资源域切分得非常细：`views/` 下 60 个文件按 domain 分目录（issue/ 下 13 个模块：activity、archive、attachment、comment、label、link、reaction、relation、sub_issue、subscriber、version 等），`urls/` 按域拆分路由文件（issue.py 注册 40 条 path，为全仓最多），`serializers/` 与 `permissions/`（workspace/project/page 四级权限，`base.py` 中按 `ROLE.ADMIN/GUEST` 等角色判定）。视图统一继承自 `BaseAPIView`/`BaseViewSet`（`app/views/issue/base.py:81,209`），鉴权、工作区定位、分页等横切逻辑集中在这两个基类。另有面向第三方的 `app/views/external/` 与 API Key 鉴权（`plane/api/`、`db/models/api.py`）。

重活全部甩给 Celery：`plane/bgtasks/` 有 **32 个任务模块**（webhook 分发、邮件通知、issue 活动记录、版本同步、分析导出、S3 对象复制、工作区级联删除等），API 请求只写库 + 投递消息，保证读路径轻。

### 5.3 实时协作服务（apps/live）

独立 Node 服务，基于 Hocuspocus（Yjs 协作服务框架）封装为单例（`apps/live/src/hocuspocus.ts` 的 `HocusPocusServerManager`），`debounce: 10000` 控制文档持久化频率；通过扩展机制挂载 database（文档落 PG）、redis（多实例间广播）、logger、title-sync（防抖同步文档标题，`src/extensions/title-update/`）与 force-close-handler；鉴权走 `src/lib/auth.ts` 自实现的 `onAuthenticate`（校验 API 会话 cookie/token）。它同时承担页面 PDF 导出（`src/controllers/pdf-export.controller.ts`，用 `@react-pdf/renderer`）。部署上依赖 Redis 与 Postgres（`docker-compose.yml` 中 `live` 服务与 `plane-redis`/`plane-db` 同网）。

### 5.4 前端状态组织（apps/web）

MobX 面向领域建模：`apps/web/store/` 下按 cycle/module/project/issue/inbox/pages 等建 store，`root.store.ts` 聚合；`apps/web/services/` 提供 30+ 个对应后端资源的 service 类，形成 `组件 → store → service → REST` 的单向链路。共享基座在 `packages/shared-state`（AGENTS.md 明确"State Management: MobX stores in packages/shared-state"）。

## 6. 构建与运行

来自根 `package.json`、`AGENTS.md` 与 `docker-compose.yml`：

```bash
# 前端 monorepo（需要 Node >=22.22 + pnpm 11）
pnpm install
pnpm dev            # turbo run dev --concurrency=18（web:3000, admin:3001）
pnpm build          # turbo run build
pnpm check          # 全量检查（format/lint/types）
pnpm check:lint     # oxlint

# 后端（apps/api，Python 3.12 + uv）
cd apps/api && uv sync
python manage.py migrate && python manage.py runserver   # 常规 Django 流程

# 一键自托管（含 web/admin/space/api/worker/beat-worker/migrator/live/
#   plane-db(Postgres)/plane-redis/plane-mq(RabbitMQ)/plane-minio/proxy 共 13 个服务）
./setup.sh          # 或 docker compose up -d（docker-compose.yml）
```

API 测试：`apps/api/pytest.ini`，`DJANGO_SETTINGS_MODULE=plane.settings.test`，运行 `run_tests.sh`（pytest，标记 unit/contract/smoke/slow，`--reuse-db --nomigrations`）。

## 7. 测试与工程质量

- **后端测试最有分量**：`apps/api/plane/tests/` 下 130 个测试文件——契约测试 85 个（contract/api 26 + contract/app 59，覆盖权限 parity、写角色、认证、周期、外部 API guest 可见性等）、单元测试 43 个、smoke 1 个；`factories.py` 用工厂模式造数据，`TESTING_GUIDE.md`/`RUNNING_TESTS.md` 提供指南。
- **前端测试偏薄**：测试文件集中在 `packages/blocks`、`packages/editor`（vitest `__tests__`）、`apps/live`（vitest + coverage 脚本）等包内，`apps/web` 本体没有发现 test 文件（find 统计为 0，未确认是否有 e2e 在别的仓库）。
- **CI**（`.github/workflows/`，10 条流水线）：PR 拆分检查 API 与 web 应用（pull-request-build-lint-api / -web-apps），另有 codeql、migration-check（迁移一致性）、i18n-sync-check（多语言同步）、copyright-check、check-version、react-doctor 等，工程化程度高。
- **代码质量工具链**：oxlint+oxfmt（web 侧）、ruff（Python 侧）、husky+lint-staged 强制提交门禁；133 个迁移文件（`find apps/api -path "*migrations*"`），模型演进痕迹清晰。

## 8. 社区与维护状态

- **极活跃**：GitHub 元数据显示最近 push 2026-11-02（分析前一天），open issues 1091，forks 6279（`gh api repos/makeplane/plane`）。
- **发布节奏**：v1.4.3-rc1（10-06）→ rc2（10-23）→ v1.4.3 正式版（10-28），约每月一个 minor/patch，RC 先行（`gh api repos/makeplane/plane/releases`）；仓库内版本号 1.5.0 领先于最近 release，说明 main(preview) 持续演进。
- **公司主导 + 开源社区**：`CODEOWNERS`、`SECURITY.md`、`CONTRIBUTING.md`、`CODE_OF_CONDUCT.md` 齐备；提交主体为 Plane Software 团队（版权头 "Plane Software, Inc. and contributors" 遍布所有源文件，且 CI 有 copyright-check 工作流强制）。贡献者人数未确认（tarball 无 git 历史，未单独拉取 contributors API）。
- **文档与治理**：仓库自带 `AGENTS.md`（AI agent 开发指南，含命令/风格/依赖约定），`docs/` 目录与 developers.plane.so 站点配套。

## 9. 亮点与风险

**亮点**
1. 领域建模精细：102 个模型类把 issue 相关关切拆到 19 个独立模型（活动、订阅、反应、版本、序号），状态机带 triage 隔离与插入留白设计，审计与回溯能力强（`apps/api/plane/db/models/`）。
2. 实时协作自研闭环：编辑器（packages/editor）+ Yjs/Hocuspocus 服务（apps/live）+ Redis 多实例同步 + PDF 导出，全套自主可控，是同类开源产品中少见的完整栈。
3. 工程化成熟：pnpm catalog 锁版本、Turborepo 编排、oxlint/ruff 双 lint、130 个后端测试文件 + 10 条 CI 流水线（含 migration-check、i18n-sync-check 这类领域专用检查）。
4. 自托管体验完整：13 个服务的 compose 编排、K8s/Helm 清单、God mode 实例管理、外部 API（`app/views/external/`），从单机到集群路径清晰。
5. `AGENTS.md` 等面向 AI 协作的开发指南，对新人/agent 上手友好。

**风险/局限**
1. AGPL-3.0 对商业化集成有传染性约束（根 `package.json` license 字段、`LICENSE.txt`），想二次分发为 SaaS 需谨慎评估。
2. 体量大、服务多：完整自托管要跑 Postgres/Redis/RabbitMQ/MinIO + 6 个应用容器，小团队运维成本不低。
3. 前端测试覆盖不均：apps/web 本体无测试文件，回归主要依赖 lint/类型检查与后端契约测试（本次静态统计，未确认是否有外部 e2e）。
4. tarball 快照中 `packages/i18n/locales` 为指向 `src/locales` 的符号链接，Windows 解压需跳过（本次已核对目标本体存在，无内容损失）——对 Windows 开发者是个小摩擦。

## 10. 总结评价

Plane 是目前开源项目管理赛道里完成度最高的产品之一：以 Django/DRF + React 双栈支撑起 Jira 级别的功能面（工作项、迭代、模块、视图、收件箱、协作文档、外部 API），又在实时协作与编辑器这个最难啃的部分做到了全自研。代码库组织规范、模型设计考究、CI/CD 与发布流程成熟，明显是一支有工程纪律的商业团队在持续投入。适合：想自托管替代 Jira/Linear 的中小团队直接用；想做项目管理类产品的团队读它的数据模型（`apps/api/plane/db/models/`）与协作服务（`apps/live`）——这两块是全仓最有学习价值的部分。注意 AGPL-3.0 许可与多服务部署成本这两个现实约束。
