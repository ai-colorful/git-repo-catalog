# django/django 项目分析

> 星数 91570（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：Django 是 Python 生态最老牌的"全家桶"式高级 Web 框架，以"快速开发 + 务实设计"为核心哲学，内置 ORM、模板引擎、Admin 后台、认证、表单等完整设施。

| 项目 | 内容 |
|---|---|
| 名称 | django/django |
| 星数 | 91,570（快照 2026-10-24） |
| 语言 | Python（要求 >= 3.12，支持 3.12–3.15 含 free-threading，见 `pyproject.toml`） |
| 许可证 | BSD-3-Clause（`LICENSE`、`pyproject.toml`） |
| 当前开发版本 | 6.2.0 alpha（`django/__init__.py` 中 `VERSION = VersionTuple(6, 2, 0, "alpha", 0)`） |
| 最近活跃 | GitHub API 显示 pushed_at 2026-10-30，每日仍在合码 |
| 运行时依赖 | 仅 3 个：asgiref、sqlparse、tzdata（仅 Windows），见 `pyproject.toml` |

## 2. 项目是什么，解决什么问题

Django 诞生于 2005 年（Lawrence Journal-World 报社），要解决的问题是：让开发者不必反复造轮子就能构建数据驱动型网站。它把 Web 开发中几乎所有公共部分——数据库访问、URL 路由、请求/响应处理、表单校验、模板渲染、用户认证、会话、后台管理——全部收进一个框架，并强约束地组织起来（`README.rst` 自述"high-level Python web framework that encourages rapid development and clean, pragmatic design"）。

与 Flask/FastAPI 这类微框架相对，Django 走的是"batteries included"路线：`django/contrib/` 下的 15 个可选应用（`django/contrib/` 目录）提供 admin、auth、sessions、contenttypes、gis、staticfiles、sitemaps 等，装上即用。其中自动生成的 Admin 后台是它最著名的差异化能力——为任意模型自动生成增删改查界面。

Django 的第二个显著特征是"渐进式异步"。从 3.0 引入 ASGI 支持，到如今整个请求管线是 sync/async 双轨：中间件链按 `sync_capable`/`async_capable` 标志自动适配模式，WSGI 与 ASGI 共享同一套 handler 逻辑（`django/core/handlers/base.py` 的 `load_middleware` 与 `adapt_method_mode`）。

第三个特征是极其严谨的工程纪律：文档以 Sphinx 源码形式随仓库维护（`docs/` 约 680 个 .txt/.rst、17 万行），测试代码量约为框架本身的两倍多，每次提交都要过测试、文档、迁移一致性等多重 CI 检查（`.github/workflows/`）。

## 3. 技术栈与依赖

- 语言：纯 Python，当前主线要求 Python >= 3.12，分类器标注支持到 3.15，且加入 `Python :: Free Threading :: 2 - Beta`（无 GIL 自由线程为 Beta 支持，`pyproject.toml`、`tox.ini` 中有 `py315t` 环境）。
- 运行时依赖极简（`pyproject.toml` [project] dependencies）：
  - `asgiref>=3.13`：sync/async 适配层（`async_to_sync`/`sync_to_async`），handler 中直接使用（`django/core/handlers/base.py`）；
  - `sqlparse>=0.5.0`：SQL 解析/格式化，用于迁移与 SQL 日志；
  - `tzdata`：仅 Windows 需要的时区数据。
- 可选依赖：`argon2-cffi`、`bcrypt`（密码哈希算法，`pyproject.toml` optional-dependencies）。
- 测试期依赖较宽：Playwright（screenshot/浏览器测试）、redis、pymemcache/pylibmc（缓存后端）、jinja2（模板后端）、Pillow、numpy 等（`tests/requirements/py3.txt`）。
- 构建：setuptools>=83（`pyproject.toml` [build-system]），入口脚本 `django-admin`。
- 前端/工具链：`Gruntfile.js` + `package.json` 管理 admin 静态资源，`biome.json` 做 JS lint。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `django/` | 框架本体，907 个 .py、约 16.6 万行（本次实测统计） |
| `django/core/` | 请求/响应管线（handlers）、缓存、管理命令（management）、checks 框架、序列化、mail、ASGI/WSGI 服务 |
| `django/db/` | ORM：models（QuerySet/Model/fields）、sql（查询编译）、migrations、4 个后端 backends/{sqlite3,postgresql,mysql,oracle} + dummy |
| `django/template/` | 模板引擎：Engine/Template/上下文/过滤器默认库/smartif 表达式解析器/loader |
| `django/contrib/` | 15 个可插拔应用：admin、auth、contenttypes、sessions、gis、staticfiles 等 |
| `django/forms/`、`django/http/`、`django/urls/`、`django/views/`、`django/middleware/`、`django/apps/`、`django/conf/`、`django/utils/` | 表单、HTTP 对象、路由、通用视图、中间件、应用注册、设置系统、工具集 |
| `tests/` | 221 个顶层目录、2011 个 .py、约 36 万行测试（实测），自带 `runtests.py` 入口 |
| `docs/` | Sphinx 文档源码，约 680 个文件、17.4 万行（实测） |
| `scripts/` | 发布与维护脚本：`do_django_release.py`、`verify_release.sh`、`backport.sh`、`manage_translations.py` 等 |
| `.github/workflows/` | 20 余个 CI workflow（tests、docs、linters、playwright、screenshots、benchmark、check-migrations 等） |

组织方式评述：单体仓库（monorepo 式单包），框架本体 + 测试 + 文档 + 发布脚本同仓。核心 `django/` 内部按"层"分包（core/db/forms/template/http/urls），功能扩展统一进 `contrib/`，边界清晰。

## 5. 核心架构与关键实现

### 5.1 请求/响应管线：sync/async 双轨中间件链

`django/core/handlers/base.py`（375 行）的 `BaseHandler.load_middleware` 在启动时把 `settings.MIDDLEWARE` 逆序组装成洋葱形调用链，关键在于每个中间件按其 `sync_capable`/`async_capable` 声明决定链上节点的模式，必要时用 `asgiref` 的 `sync_to_async`/`async_to_sync` 做模式桥接（`adapt_method_mode`）：

```python
for middleware_path in reversed(settings.MIDDLEWARE):
    middleware = import_string(middleware_path)
    middleware_can_sync = getattr(middleware, "sync_capable", True)
    middleware_can_async = getattr(middleware, "async_capable", False)
    ...
    adapted_handler = self.adapt_method_mode(
        middleware_is_async, handler, handler_is_async,
        debug=settings.DEBUG, name=name,
    )
```

`_get_response`（`django/core/handlers/base.py:176`）负责链内核心动作：`resolve_request` 经 `django/urls/resolvers.py`（843 行，含正则/路径转换器路由编译）解析 URL 回调，依次套 view 中间件、调用视图、再套 template/exception 中间件；WSGI/ASGI 子类（`wsgi.py`、`asgi.py`）只负责环境对象转换，业务逻辑完全共享。

### 5.2 ORM：惰性 QuerySet + 独立于方言的 SQL 编译层

ORM 是 Django 代码量最大的子系统。`django/db/models/query.py`（3143 行）的 `QuerySet` 自述为"lazy database lookup"：构造时只组装 `sql.Query` 对象，迭代才触发 `_fetch_all`/`iterator` 并写入 `_result_cache`；还带 `_deferred_filter`（延迟求值过滤）与 `_cloning_enabled` 等惰性优化字段。

分层设计是重点：`django/db/models/sql/query.py`（2924 行）持有与方言无关的查询结构，`django/db/models/sql/compiler.py` 负责把它编译为 SQL；各数据库差异被收敛到后端包的标准接口——`django/db/backends/base/` 定义 `base.py/client.py/creation.py/features.py/introspection.py/operations.py/schema.py/validation.py` 七件套，sqlite3/postgresql/mysql/oracle 各自实现（`django/db/backends/`），schema 变迁另由 `django/db/migrations/` 承担。

### 5.3 模板引擎：双后端 + 自建表达式解析器

`django/template/engine.py` 的 `Engine`（248 行）与 `django/template/base.py` 的 `Template` 组成核心，`django/template/backends/` 提供 DjangoTemplates 与 Jinja2 两种后端适配（测试依赖中出现 jinja2，`tests/requirements/py3.txt`）。模板语言的条件/循环标签语法由自研的 `django/template/smartif.py`（smart if，小型运算符优先级解析器）支撑，filter/tag 注册机制在 `django/template/library.py`，加载策略（文件系统/app 目录/缓存）在 `django/template/loaders/`。

### 5.4 Admin：声明式后台的集大成者

`django/contrib/admin/` 是 contrib 中最大的应用：`options.py`（2947 行）定义 `BaseModelAdmin`/`ModelAdmin`/`InlineModelAdmin` 等声明式配置类（类级 `Action`、`ShowFacets` 枚举也在其中），配合 `sites.py` 的 AdminSite 注册中心、`views/` 的 changelist/change 视图、自带 `templates/` 与 `static/`，实现对任意注册模型自动生成整套管理界面。它大量反射模型元数据（`django/db/models/options.py`）与表单系统（`django/forms/models.py`，1727 行的 `modelform_factory` 体系）。

## 6. 构建与运行

来自 `docs/internals/contributing/writing-code/unit-tests.txt` 与 `tox.ini` 的实际命令（本次未执行，仅静态摘录）：

```bash
pip install -e .                       # 可编辑安装（tox.ini testenv 用 usedevelop）
python -m django version               # 直接可用：django/__main__.py 提供入口
cd tests && ./runtests.py              # 跑全量测试套件
cd tests && ./runtests.py basic        # 跑单个测试目录
tox -e py312                           # 多版本/多数据库矩阵（py312-postgres 等）
tox -e py315t                          # free-threading 变体（tox.ini）
```

- 测试依赖清单：`tests/requirements/py3.txt`；数据库后端各有 `postgres.txt`/`mysql.txt`/`oracle.txt`。
- 文档构建：Sphinx（`docs/` 下有构建说明 `docs/README`）。
- 用户安装：`pip install Django`，`django-admin startproject`（`pyproject.toml` [project.scripts]）。

## 7. 测试与工程质量

- 规模（实测统计）：`tests/` 共 221 个顶层目录、2011 个 .py 文件、约 35.96 万行，约为框架本体（16.6 万行）的 2.2 倍。按主题一目录一测：`tests/basic/`、`tests/aggregation/`、`tests/admin_views/`、`tests/migrations/` 等，每目录常含独立 `models.py` 构造隔离的测试数据模型。
- 风格：标准库 `unittest` + Django 自带 `django.test.TestCase`/`SimpleTestCase`（如 `tests/expressions_case/tests.py` 头部 `import unittest` 与 `from django.test import ...`），无 pytest 依赖；分单元、集成（多数据库真实连接）、浏览器级（Playwright，`.github/workflows/playwright.yml`、`screenshots.yml`）三层。
- CI（`.github/workflows/`，20 余个 workflow）：`tests.yml`（Linux/Windows 矩阵 + Python 3.15）、`postgis.yml`、`check-migrations.yml`（迁移文件一致性）、`linters.yml`、`docs.yml`/`docs_win.yml`（文档链接与构建）、`benchmark.yml`、`coverage_tests.yml`、`check_commit_messages.yml`、`check_pr_limit.yml`，另有 `zizmor.yml` 对 GitHub Actions 本身做安全审计——CI 覆盖面在同类项目中属顶级。
- 代码质量观察：框架本体强制 black/isort/flake8（`pyproject.toml`、`tox.ini` envlist）；文档与代码同仓同 PR 审查；`django/core/checks/` 还内置了对用户项目的静态检查框架。

## 8. 社区与维护状态

- 活跃度：GitHub API pushed_at 2026-10-30，open_issues 538，forks 36,550；仓库处于每日提交状态。
- 维护主体：Django Software Foundation（`pyproject.toml` authors），有专职 fellowship（.release 机制未确认，团队资助信息见 djangoproject.com，本仓库内未确认）。
- 发布节奏：版本号 6.2.0 alpha 表明 6.2 处于开发期；GitHub Releases 未使用（releases API 返回空），产物通过 PyPI + 官网源码包分发，`scripts/do_django_release.py` 负责构建 wheel/sdist、生成 MD5/SHA 校验和文件并做 PGP 签名指引，`verify_release.sh` 校验发布，`backport.sh` 支撑安全补丁回移，`archive_eol_stable_branches.py` 归档 EOL 分支（tags 列表中 stable/5.1.x、5.0.x、4.2.x 等应为 EOL 分支转 tag 所致，推断）。
- 贡献者结构：contributors API 单页即满 100（实际更多，精确总数未确认）；`AUTHORS` 文件维护完整名单。
- 翻译治理：`scripts/manage_translations.py` + 每个模块的 `locale/` 目录，国际化是官方流程一部分。

## 9. 亮点与风险

亮点：
1. 极低的运行时依赖（3 个）与极广的自带能力（ORM/Admin/auth/forms）组合，15 年验证的"全家桶"架构（`pyproject.toml`、`django/contrib/`）。
2. sync/async 双轨管线设计干净：一个 handler 两种执行模式，桥接集中在 `adapt_method_mode`（`django/core/handlers/base.py`），对用户中间件向后兼容。
3. 测试量 2.2 倍于本体、221 个主题化测试目录 + 20 余个 CI workflow，回归防线深厚（`tests/`、`.github/workflows/`）。
4. 紧跟 CPython 前沿：Python 3.15 与 free-threading 已进入 tox 矩阵和 Beta 分类器（`tox.ini`、`pyproject.toml`）。
5. 发布工程成熟：签名、校验和、回移、分支归档全部脚本化（`scripts/`）。

风险/局限：
1. 强约定带来高学习曲线与"框架感"，退出成本高；对小型 API 服务偏重。
2. Admin/ORM 的核心类体量庞大（`contrib/admin/options.py` 2947 行、`db/models/query.py` 3143 行），深度定制需阅读大量隐式约定。
3. 官方仅支持 4 个数据库后端，NoSQL/新存储需依赖第三方（`django/db/backends/`）。
4. 全同步 ORM 在 async 视图中仍需 `sync_to_async` 线程桥接，异步性能上限受制于此（`django/core/handlers/base.py`）。

## 10. 总结评价

Django 是"Web 框架作为完整产品"的代表：以最少的第三方依赖提供从数据库到后台管理的全栈能力，并用教科书级的测试、文档、CI 与发布工程维持 20 年的生命力。当前主线（6.2 dev）仍在积极演进异步能力与 free-threading 支持。适合需要快速交付数据驱动站点/内部系统的团队使用；想学习成熟 Python 工程的人，重点推荐阅读请求管线（`django/core/handlers/`）、ORM 编译层（`django/db/models/sql/`）与 admin 的声明式设计（`django/contrib/admin/options.py`）——它们分别展示了适配器模式、编译分层与元编程三种典范。
