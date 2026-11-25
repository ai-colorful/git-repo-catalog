# fastapi/full-stack-fastapi-template 项目分析

> 星数 35900（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：FastAPI 官方出品的全栈 Web 应用模板——一个带认证、数据库、邮件、前端面板和部署方案、开箱即用的"生产级起点"，也是 FastAPI 作者演示"正确组织一个 FastAPI 项目"的参考实现。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | fastapi/full-stack-fastapi-template（原 tiangolo/full-stack-fastapi-postgresql） | `gh api repos/tiangolo/full-stack-fastapi-postgresql` 重定向验证 |
| 星数 | 35900（快照）；分析时 GitHub API 显示 46,166 | gh api |
| 主语言 | TypeScript（前端体量 9.5k 行 > 后端 Python 2.7k 行） | 本地 wc 统计 |
| 许可证 | MIT | `LICENSE` |
| 默认分支 | master | gh api |
| 创建时间 | 2019-02-23 | gh api |
| 最近推送 | 2026-11-03（分析前 2 天，高度活跃） | gh api |
| 贡献者 | 约 83 人（contributors 分页推算） | gh api |

## 2. 项目是什么，解决什么问题

这是 FastAPI 官方组织下的"_reference stack_（参考技术栈）"。它在 README.md 中自述为 "Full stack, modern web application template"，覆盖了一个真实 Web 产品需要的全部基础设施：JWT 认证 + 用户系统、PostgreSQL + 迁移、邮件（密码找回）、React 管理面板、反向代理与 HTTPS、CI/CD（`README.md`）。

它解决的核心痛点是"FastAPI 会了，但项目不知道怎么组织"。模板用一套小而完整的业务（用户 + 条目 Items）演示了官方推荐的工程分层：路由-依赖注入-模型-CRUD 的职责划分（`backend/app/`），前端则演示了如何从 OpenAPI 规范自动生成类型安全的 API 客户端（`scripts/generate-client.sh`）。

使用方式是 GitHub 的 "Use this template" 按钮：用户基于它建仓后直接在真实业务上增删改，而不是当库安装（`README.md`）。历史上有过两代生成方案：早期用 Cookiecutter，后来明确放弃（"Refactor project generation, discard cookiecutter"，`release-notes.md` #553），短暂引入 Copier（#612），最终定型为"直接 fork/clone + Use this template"。`hooks/post_gen_project.py` 只剩一个 CRLF→LF 转换的旧 hook 残留。

项目由 tiangolo 个人仓（full-stack-fastapi-postgresql / full-stack-fastapi-template）转入 fastapi 官方组织，2024-08 发布 0.7.0（Playwright E2E、分页），2026-08 发布 0.12.0 接入 FastAPI Cloud 一键部署（`release-notes.md`，部署文档 `deployment.md`）。它是事实上的"FastAPI 官方推荐项目结构"，很多教程和脚手架都以它为蓝本。

## 3. 技术栈与依赖

后端（`backend/pyproject.toml`，Python `>=3.14`，uv 管理依赖 `uv.lock`）：

| 依赖 | 用途 |
|---|---|
| fastapi[standard] >=0.141.1 | Web 框架（含 `fastapi run/dev` CLI） |
| sqlmodel | ORM：Pydantic + SQLAlchemy 融合模型 |
| pydantic / pydantic-settings | 校验与配置（`backend/app/core/config.py`） |
| psycopg[binary] 3.x | PostgreSQL 驱动 |
| alembic | 数据库迁移（`backend/app/alembic/`） |
| pyjwt | JWT 签发/校验（HS256） |
| pwdlib[argon2,bcrypt] | 密码哈希（Argon2 主 + Bcrypt 兼容验证） |
| emails + jinja2 | SMTP 发信 |
| sentry-sdk[fastapi] | 生产监控（`backend/app/main.py`） |
| pytest / mypy(strict) / ruff / ty / coverage | 开发工具链（ty 为 Astral 新型检查器） |

前端（`frontend/package.json`，Bun 管理，根 `package.json` 定义 workspaces）：

| 依赖 | 用途 |
|---|---|
| react 19.3 + typescript 6.0 + vite 8.3 | 前端基座 |
| @tanstack/react-query v5 | 服务端状态/数据请求（`frontend/src/hooks/useAuth.ts`） |
| @tanstack/react-router v1 + react-table v9 | 路由（文件式 `frontend/src/routes/`）与表格 |
| radix-ui 系列 + class-variance-authority + tailwind-merge | shadcn/ui 组件体系（`frontend/src/components/ui/`） |
| tailwindcss v4 | 样式 |
| react-hook-form + zod v4 | 表单与校验 |
| @hey-api/openapi-ts | 从 openapi.json 生成 TS 客户端（`frontend/openapi-ts.config.ts`） |
| axios | HTTP 客户端（生成的 client 底层） |
| @biomejs/biome | Lint/格式化（替代 ESLint+Prettier） |
| @playwright/test | E2E 测试 |

基础设施（`compose.yml`）：PostgreSQL 18、Traefik v3.7（反向代理/自动路由）、Adminer（DB 管理 UI）、Mailpit（本地邮件捕获，`compose.override.yml`）。邮件模板用 React Email（`packages/react-email/`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `backend/` | FastAPI 应用：`app/` 为源码，含 pyproject.toml、Dockerfile、alembic.ini |
| `backend/app/api/routes/` | 按资源拆分的路由：items / login / users / utils / private |
| `backend/app/core/` | config.py（设置）、db.py（engine/会话）、security.py（JWT+哈希） |
| `backend/app/alembic/versions/` | 5 个迁移脚本（建表、UUID 化、created_at、级联删除等） |
| `backend/tests/` | pytest 测试：api/routes/ + crud/ + scripts/ + utils/ |
| `frontend/src/routes/` | TanStack Router 文件式路由（login/signup/recover/reset + _layout/ 内 admin/items/settings） |
| `frontend/src/client/` | 自动生成的 API 客户端（types.gen.ts / sdk.gen.ts，勿手改） |
| `frontend/src/components/` | 按页面分组的组件 + ui/（shadcn）+ hooks/ |
| `frontend/tests/` | Playwright E2E（login/items/admin/sign-up/user-settings/reset-password 6 个 spec） |
| `packages/react-email/` | React Email 邮件模板包（workspace 成员） |
| `scripts/` | generate-client.sh、test.sh、prepare_release.py 等流水线脚本 |
| `hooks/` | 旧 cookiecutter post-gen hook 残留（仅 CRLF 转换） |
| `.github/workflows/` | 15 个 CI/CD workflow |

组织方式评述：这是一个刻意保持"扁平简单"的单仓双应用结构——后端 `app/` 下没有层层嵌套的包，`models.py` 和 `crud.py` 都是单文件（约 127/69 行），路由按资源一个文件；前端同样按 routes/components/hooks 惯例平铺。模板的立场很明确：起步不引入架构税，业务长大再自己拆。根 `package.json` 用 bun workspaces 把 frontend 和 packages/react-email 统一管理，但 Python 侧独立用 uv，两套包管理并存是唯一稍显"双轨"的地方。

## 5. 核心架构与关键实现

### 5.1 后端分层：路由 → 依赖注入 → 模型 → CRUD

所有路由通过 `backend/app/api/main.py` 汇聚为 `api_router`，其中 `private.py`（免认证的建用户调试端点）只在 `FASTAPI_ENV == "development"` 时挂载——模板对"演示便利"与"生产安全"的边界处理得很干净：

```python
api_router = APIRouter()
api_router.include_router(login.router)
api_router.include_router(users.router)
api_router.include_router(utils.router)
api_router.include_router(items.router)

if settings.FASTAPI_ENV == "development":
    api_router.include_router(private.router)
```

依赖注入是全模板的中枢（`backend/app/api/deps.py`）：`get_db` 用生成器管理 SQLModel Session 生命周期，打包成 `SessionDep`；`get_current_user` 解 JWT → 查用户 → 查活跃状态，打包成 `CurrentUser`；再加一层 `get_current_active_superuser` 做管理员鉴权。路由函数只需声明参数即可获得会话或当前用户。

### 5.2 JWT 认证链与密码安全

登录链路（`backend/app/api/routes/login.py` → `crud.py` → `core/security.py`）：`POST /login/access-token` 接收 OAuth2PasswordRequestForm → `crud.authenticate` 查邮箱 → `security.create_access_token`（pyjwt HS256，`sub` 存用户 UUID，默认 8 天过期，`core/config.py` 的 `ACCESS_TOKEN_EXPIRE_MINUTES`）→ 后续请求由 `deps.get_current_user` 解码验证。

两个值得注意的安全细节。其一是防时序攻击（`backend/app/crud.py`）：用户不存在时仍对一个预置 Argon2 哈希跑一遍验证，使"邮箱不存在"和"密码错误"响应耗时一致：

```python
DUMMY_HASH = "$argon2id$v=19$m=65536,t=3,p=4$MjQyZWE1MzBjYjJlZTI0Yw$..."

def authenticate(*, session, email, password):
    db_user = get_user_by_email(session=session, email=email)
    if not db_user:
        verify_password(password, DUMMY_HASH)  # 常数时间占位
        return None
    verified, updated_hash = verify_password(password, db_user.hashed_password)
    if updated_hash:   # pwdlib 自动重哈希（如算法参数升级）
        db_user.hashed_password = updated_hash
        ...
```

其二是密码找回防枚举（`routes/login.py`）：无论邮箱是否存在都返回同一句 "If that email is registered, we sent..."。密码哈希用 pwdlib 的 Argon2+Bcrypt 双哈希器，`verify_and_update` 在验证同时把旧格式哈希透明升级（`core/security.py`）。

### 5.3 SQLModel 多 Schema 继承模式

`backend/app/models.py` 是模板最具教学价值的部分：每个实体拆成 Base/Create/Update/Public + table 模型五个类，Pydantic 校验模型与 SQLAlchemy 表模型同源。`User(UserBase, table=True)` 继承共享字段后只补 `id/hashed_password/items`；表间用 `Relationship(back_populates=...)` + `ondelete="CASCADE"` 级联删除。这种"每用例一个 Schema"避免了"一个 User 模型管所有输入输出"的经典漏洞（如客户端可提交 `is_superuser`）。

### 5.4 OpenAPI 驱动的前端客户端

`scripts/generate-client.sh` 是前后端契约的核心：先让 FastAPI 导出自身 openapi.json（`custom_generate_unique_id` 保证操作 ID 按 tag 前缀稳定，`backend/app/main.py`），再跑 `@hey-api/openapi-ts` 生成 `frontend/src/client/`。前端所有请求都走生成的 `UsersService/LoginService` 等类型安全 SDK，配合 TanStack Query 的 `useQuery/useMutation` 使用（`frontend/src/hooks/useAuth.ts`）。改后端 → 重新生成 → 前端拿到新类型，契约漂移在编译期暴露。

### 5.5 容器编排与"单域名"部署

`backend/Dockerfile` 为两阶段构建：`oven/bun:1` 构建前端产物到 `backend/app/frontend`，再由 `python:3.14` + uv 阶段 COPY 进镜像，最终 `CMD ["fastapi", "run", "--workers", "4"]`——前后端由 FastAPI 同域伺服（`app.main` 中 `app.frontend("/", directory=FRONTEND_DIR)`），彻底规避 CORS。`compose.yml` 用 Traefik 的 Docker provider 读 label 路由（backend 走 `${DOMAIN}`，adminer 走 `adminer.${DOMAIN}`），`compose.deploy.yml` 增加 HTTPS 证书卷，形成自托管部署方案（`deployment-docker-compose.md`）。

## 6. 构建与运行

本地开发（`development.md`）：

```bash
docker compose up -d db mailpit          # 起支撑服务
cd backend && uv sync && uv run bash scripts/prestart.sh   # 迁移+初始数据
uv run fastapi dev                        # 后端 :8000
bun install && bun run dev                # 前端 :5173
```

全栈容器（`development.md` / `scripts/test.sh`）：

```bash
docker compose run --rm backend bash scripts/prestart.sh
docker compose watch                      # 带文件同步热更
```

测试与质量（`backend/pyproject.toml`、`frontend/package.json`、`scripts/`）：

```bash
cd backend && uv run pytest                                    # 后端测试
bash scripts/test.sh                      # Docker 全栈测试（build→prestart→pytest→down -v）
bash scripts/generate-client.sh           # 重新生成前端 API 客户端
cd frontend && bun run build && bunx playwright test           # E2E
```

`scripts/prestart.sh` 内容即两行：`alembic upgrade head` + `python app/initial_data.py`（创建首个超级用户与示例数据，`app/core/db.py` 的 `init_db`）。

## 7. 测试与工程质量

- 后端：15 个测试文件，5 个 `test_*.py` 共约 1032 行（`backend/tests/api/routes/` 覆盖 items/login/users/private，`crud/test_user.py` 测 CRUD），conftest 提供依赖覆盖（覆盖 `get_db`/token 等）。CI 中跑 coverage 并上传 artifact，再由 smokeshow workflow 发布到 GitHub Pages（`.github/workflows/test-backend.yml`、`smokeshow.yml`）。
- 前端：Playwright E2E 7 个文件约 997 行（`frontend/tests/`），带 `auth.setup.ts` 登录前置；专用 `playwright.yml` 和 compose override 里的 playwright 服务支撑容器化 E2E。
- 静态质量：mypy `strict = true`、ruff（含禁 print、bugbear 等规则集）、Astral 新工具 ty（`backend/pyproject.toml`）、前端 Biome；pre-commit workflow + bump-pre-commit-hooks 自动升级钩子。
- CI 共 15 个 workflow：测试 3 条线（backend / docker-compose / playwright）、发布 3 条（prepare-release / create-draft-release / deploy），另有 issue-manager、detect-conflicts、guard-dependencies、zizmor（GH Actions 安全扫描）等自动化（`.github/workflows/`）。
- 供应链治理写在 `CONTRIBUTING.md`：大改动须先开 Discussion；非团队成员的 PR 禁止触碰 `pyproject.toml` / `uv.lock`（防依赖投毒）。

## 8. 社区与维护状态

- 活跃度极高：最近推送 2026-11-03；`release-notes.md` 的 "Latest Changes" 区有大量分析当周前后合并的 PR（#2436 Mailcatcher→Mailpit、#2450/#2451 邮件模板重构等）。open_issues 仅 11（gh api），issue 清理很勤。
- 版本节奏：2019 年密集发 0.1-0.4，2020-04 发布 0.5.0 后停滞近 4 年，2024-03 起（转入官方组织前后）恢复活跃，0.7→0.12 每半年至一年一版，2026-08 一周内连发 0.11.0/0.11.1/0.12.0（`release-notes.md` 版本头）。
- 贡献者结构：核心维护者 tiangolo（亲自做架构级改动，如 #2460、#2438）+ alejsdev（FastAPI 团队）、YuriiMotov、istoutjesdijk 等外部贡献者 + dependabot/pr-submit 机器人承担依赖升级（release-notes 署名统计）。
- 星数变化：任务快照 35900（2026-10-24），分析时 gh api 显示 46,166、fork 9,205——伴随 FastAPI Cloud 发布，增长明显加速（未确认具体归因，供参考）。
- 生态位：FastAPI 官方文档与 fastapi-cloud 均指向它作为标准起点；README 徽章仅两条 CI 状态（无 coveralls/pypi 等，符合"模板仓"定位）。

## 9. 亮点与风险

亮点：

1. **官方背书的"正确答案"**：分层（routes/deps/models/crud）、依赖注入用法、多 Schema 模式都由 FastAPI 作者维护，学到的不是某个人的偏好而是官方立场（`backend/app/`）。
2. **端到端类型安全**：OpenAPI → openapi-ts 生成客户端 → 前端编译期契约检查，`generate_unique_id` 定制保证生成 ID 稳定可复现（`scripts/generate-client.sh`、`backend/app/main.py`）。
3. **安全细节到位**：时序攻击防护、邮箱枚举防护、Argon2+自动重哈希、private 路由仅开发环境挂载、zizmor 扫 CI、锁文件的供应链管控。
4. **部署路径完整**：同一套 compose 从本地开发（override）到生产（deploy + Traefik HTTPS），外加 FastAPI Cloud 一键部署，前后端同域伺服免 CORS。
5. **工程自动化成熟**：15 条 workflow、smokeshow 覆盖率发布、pre-commit 钩子自动升级、release-notes 自动生成。

风险/局限：

1. **技术选型激进**：Python >=3.14、TS 6、Vite 8、React 19 都是最新大版本，老旧环境或某些库的兼容性可能踩坑（`backend/pyproject.toml`、`frontend/package.json`）。
2. **规模天花板低**：单文件 models.py/crud.py 是教学式简化，实体多了需要用户自行重构；无权限体系（仅 is_superuser 布尔位）、无刷新 token（access token 默认 8 天，偏长）。
3. **双包管理双轨**：Bun + uv 并存，团队需同时掌握两套工具链；前端 token 存 localStorage（`useAuth.ts`）虽是常见做法但不如 httpOnly cookie 抗 XSS。
4. **模板≠产品**：Items 业务只是占位示例，邮件发送依赖外部 SMTP 配置，直接上生产仍需用户补齐业务与运维（限流、审计等均无）。

## 10. 总结评价

这是"如何组织一个生产级 FastAPI 项目"的官方标准答案，也是 2019 年至今持续演进近 8 年、随 FastAPI 生态（SQLModel、FastAPI Cloud）同步迭代的活模板。它的价值一半在"拿来即用的起点"（认证/迁移/邮件/部署全配好），另一半在"可通读的教学范本"——3000 行后端代码里浓缩了依赖注入、多 Schema、契约式前后端协作、容器化部署等全套现代 Python Web 工程实践。适合：准备启动新 FastAPI 项目的团队（直接 Use this template）、想学习"官方推荐项目结构"的 FastAPI 初中级开发者（通读 `backend/app/` 与 `generate-client.sh` 链路）、以及需要一套带 E2E 的全栈 CI/CD 样板的技术负责人。不建议当作现成产品或微服务框架使用——它的克制与简单正是设计目标。
