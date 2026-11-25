# laravel/laravel 项目分析

> 星数 85252（快照 2026-10-24）｜主语言 Blade｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：**Laravel 官方应用骨架（starter template）**——用户执行 `laravel new` 或 `composer create-project laravel/laravel` 时实际拉取的模板仓库，是几乎所有新 Laravel 应用的起点；框架本体不在这里，而在 laravel/framework 仓库。

| 项目 | 内容 |
|---|---|
| 名称 | laravel/laravel（composer.json 中 `"description": "The skeleton application for the Laravel framework."`，见 `composer.json`） |
| 星数 | 85252（快照 2026-10-24） |
| 语言 | Blade 模板 + PHP 配置（全部 58 个文件为骨架配置/模板，无业务逻辑） |
| 许可证 | MIT（README.md「License」章节声明；仓库根目录无 LICENSE 文件，GitHub API license 字段为 null） |
| 默认分支 | 13.x（随 Laravel 13 主版本命名，gh api 确认） |
| 最近活跃 | pushed_at 2026-10-30（gh api，即分析前一天仍在提交） |
| 创建时间 | 2011-06-08（gh api） |
| 贡献者 | 397 人（gh api contributors 分页推算） |
| 最新 release | v13.11.0，2026-10-06（gh api） |

## 2. 项目是什么，解决什么问题

这个仓库回答的问题是："一个新 Laravel 项目应该长什么样"。它是骨架（skeleton）而非库：不含可复用代码，而是提供一套**目录约定、启动流程、默认配置与最小示例**，由 `create-project` 流程复制并按项目名改写后成为用户的应用起点（`composer.json` 的 `post-create-project-cmd` 脚本负责生成 APP_KEY、创建 SQLite 数据库并跑首次迁移）。

需要明确仓库边界：Laravel 框架的全部实现——路由、容器、Eloquent ORM、队列等（README.md「About Laravel」所列能力）——都在 `laravel/framework` 仓库。本仓库只定义"应用层"：`app/`（模型、控制器、服务提供者的空壳）、`bootstrap/app.php`（应用装配入口）、`config/`（10 个配置文件）、`routes/`、`database/`（迁移与工厂示例）、`tests/`、`resources/`。正因为它是"每个 Laravel 开发者都会 clone 一次"的仓库，其 85k 星更多衡量的是 Laravel 生态的体量与关注度，而非本仓库自身代码量（全仓仅 58 个文件）。

它同时承担三重角色：一是新项目模板；二是**默认配置的最佳实践宣言**——默认数据库从 MySQL 改为 SQLite（`config/database.php` 第 20 行 `'default' => env('DB_CONNECTION', 'sqlite')`），缓存/队列/会话默认走 database 驱动（`.env.example`），"开箱即零依赖"；三是**脚手架演进的风向标**，其每次结构变更都会被整个生态跟进。

近年最显著的演进是 2024 年（Laravel 11 时代）的"大瘦身"：删除了 HTTP Kernel、Console Kernel 与中间件目录，改为 `bootstrap/app.php` 单文件流式配置；本快照（13.x 分支）延续了该风格并进一步现代化——详见第 5 章。

## 3. 技术栈与依赖

依赖极薄，全部价值由 `laravel/framework` 承担（出处：`composer.json`）。

运行时依赖（require）：
- `php: ^8.3`
- `laravel/framework: ^13.17`（框架本体，即 Laravel 13）
- `laravel/tinker: ^3.0`（交互式 REPL）

开发依赖（require-dev）：
- `phpunit/phpunit: ^12.5.12` + `mockery/mockery: ^1.6`（测试）
- `fakerphp/faker: ^1.23`（测试数据填充，`database/factories/UserFactory.php` 使用）
- `laravel/pint: ^1.27`（代码风格修复）
- `laravel/pail: ^1.2.5`（日志实时 tail）
- `laravel/pao: ^1.0.6`（laravel 官方新 dev 包，骨架内无任何代码引用，具体作用未确认）
- `nunomaduro/collision: ^8.6`（命令行错误美化）

前端工具链（`package.json`，仅 devDependencies，无运行时 JS 依赖）：
- `vite: ^8.0.0` + `laravel-vite-plugin: ^3.1`（Vite 集成，Laravel 9.19 起替代 webpack mix 的产物，此处已是 Vite 8）
- `tailwindcss: ^4.0.0` + `@tailwindcss/vite`（Tailwind v4，经 Vite 插件接入而非 PostCSS）
- `concurrently: ^10.0.3`（多进程并行）
- `@laravel/multiplex: ^0.4.1`（optionalDependencies，官方新包，骨架内无引用，作用未确认）

AI 协作配套：`AGENTS.md` 与 `CLAUDE.md`（两者 diff 确认内容完全相同），内容为 laravel-boost-guidelines，指导 AI 编码 agent 安装 `laravel/boost` 以获得 15+ 工具（README.md「Agentic Development」章节同步宣传）。

## 4. 目录结构与代码组织

全仓 58 个文件（`find . -type f | wc -l`），顶层结构：

| 目录/文件 | 职责 |
|---|---|
| `app/` | 应用层空壳：`Http/Controllers/Controller.php`（抽象基类）、`Models/User.php`（唯一示例模型）、`Providers/AppServiceProvider.php`（空的服务提供者） |
| `bootstrap/` | `app.php`（应用装配入口）、`providers.php`（服务提供者清单）、`cache/`（框架缓存） |
| `config/` | 10 个配置文件：app/auth/cache/database/filesystems/logging/mail/queue/services/session，共约 1300 行，全部注释驱动、env 驱动 |
| `database/` | `migrations/`（3 个合并迁移）、`factories/UserFactory.php`、`seeders/DatabaseSeeder.php` |
| `routes/` | `web.php`（1 条路由）、`console.php`（1 条 inspire 命令） |
| `resources/` | `css/app.css`（9 行，Tailwind v4 @import + @theme）、`js/app.js`（1 行占位）、`views/welcome.blade.php`（223 行欢迎页） |
| `tests/` | `TestCase.php` + Unit/Feature 各 1 个 ExampleTest |
| `storage/` | 10 个带 .gitignore 的运行时目录（缓存/会话/日志/视图编译产物） |
| `public/` | `index.php`（前端控制器）、`.htaccess`、`favicon.ico`、`robots.txt` |
| 根文件 | `artisan`（CLI 入口）、`composer.json`、`package.json`、`vite.config.js`、`phpunit.xml`、`.env.example`、`.editorconfig`、`.npmrc`、`.gitattributes`、`AGENTS.md`/`CLAUDE.md`、`README.md` |

组织方式评述：这是"约定优于配置"的教科书式单体骨架——应用代码（app/）、框架装配（bootstrap/）、静态约定（config/）、资源（resources/）分层清晰，且每个目录都被框架按固定路径约定直接读取，不需要任何注册动作。`.gitattributes` 含 export-ignore（`.github`、`CHANGELOG.md` 不进发行包，这解释了 tarball 中 .github 缺失，经 gh api 补查）。

## 5. 核心架构与关键实现

### 5.1 单文件流式启动链（本仓库最核心的两段代码）

请求入口 `public/index.php`：检查维护模式 → 加载 Composer 自动加载 → 装配应用 → 处理请求，全程 20 行：

```php
// public/index.php（节选）
define('LARAVEL_START', microtime(true));
if (file_exists($maintenance = __DIR__.'/../storage/framework/maintenance.php')) {
    require $maintenance;
}
require __DIR__.'/../vendor/autoload.php';
$app = require_once __DIR__.'/../bootstrap/app.php';
$app->handleRequest(Request::capture());
```

装配本身在 `bootstrap/app.php`，这是 Laravel 11+ 对旧版 HTTP Kernel + Console Kernel + Middleware 三处分散配置的替代——路由、中间件、异常全部以闭包流式声明，连健康检查端点都是一行配置：

```php
// bootstrap/app.php（节选）
return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware): void { //
    })
    ->withExceptions(function (Exceptions $exceptions): void {
        $exceptions->shouldRenderJsonWhen(
            fn (Request $request) => $request->is('api/*') || $request->expectsJson(),
        );
    })->create();
```

`artisan`（CLI 入口）复用同一份 `bootstrap/app.php`，仅以 `handleCommand(new ArgvInput)` 收尾。骨架自带的唯一定制是异常层：`api/*` 路径与期望 JSON 的请求渲染为 JSON 错误（`bootstrap/app.php` 第 17-20 行）。

### 5.2 模型定义转向 PHP 属性注解（本快照最值得注意的演进）

`app/Models/User.php` 不再使用传统的 `$fillable`/`$hidden` 类属性，而是用 Eloquent 的 PHP 8 属性（Attribute）注解声明批量赋值白名单与序列化隐藏字段：

```php
// app/Models/User.php（节选）
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Attributes\Hidden;

#[Fillable(['name', 'email', 'password'])]
#[Hidden(['password', 'remember_token'])]
class User extends Authenticatable
{
    use HasFactory, Notifiable;

    protected function casts(): array
    {
        return ['email_verified_at' => 'datetime', 'password' => 'hashed'];
    }
}
```

`casts()` 方法返回强类型数组（而非旧版 `$casts` 属性）、`'password' => 'hashed'` 自动哈希，是官方骨架近几版的既有演进；`#[Fillable]`/`#[Hidden]` 注解则是本快照（13.x）刚落地的新写法——官方模板以此向全生态推广新 API。

### 5.3 数据库默认值策略：SQLite 与"三合一"迁移

`database/migrations/` 仅 3 个文件，将传统十几个迁移合并为按功能域划分的单元：`0001_01_01_000000_create_users_table.php` 一次建 users、password_reset_tokens、sessions 三张表（第 14-37 行），另两个分别建 cache 与 jobs 表——缓存、队列、会话默认全部落库。配合 `config/database.php` 默认 SQLite、`.env.example` 的 `DB_CONNECTION=sqlite`、`composer.json` 中 `post-create-project-cmd` 自动 touch SQLite 文件，新项目无需任何数据库服务即可跑通。`config/session.php`/`config/cache.php`/`config/queue.php` 的默认驱动均为 env 中的 `database`。

### 5.4 前端集成：Vite 8 + Tailwind 4 与"零构建兜底"

`vite.config.js` 展示了当前官方推荐的前端栈：laravel-vite-plugin 托管两个入口（css/js）并开启 Blade 模板热刷新，字体经 `bunny('Instrument Sans', ...)` 走 Bunny Fonts（GDPR 友好的 Google Fonts 替代），Tailwind 4 以 Vite 插件形式接入。`resources/views/welcome.blade.php` 则有一个务实的兜底设计（第 11-14 行）：仅当 `public/build/manifest.json` 或 `public/hot` 存在时才输出 `@vite` 标签，否则内联一份预编译的 Tailwind CSS——即用户没跑 `npm install && npm run build`，欢迎页也能完整渲染。

### 5.5 composer 脚本即工作流

`composer.json` 的 scripts 段把项目生命周期编码为可执行命令：`composer setup`（install + 复制 .env + key:generate + migrate + npm install/build）一键初始化；`composer dev` 调 `php artisan dev`（并行启动开发服务，package.json 中的 concurrently 配合多进程开发）；`composer test` 先清配置缓存再 `artisan test`。`post-autoload-dump` 的 `package:discover` 则是框架自动发现包的标准钩子。

## 6. 构建与运行

以下命令均出自 `composer.json` scripts 与 `package.json`（静态提取，未实际运行）：

```bash
composer create-project laravel/laravel myapp   # 官方推荐创建方式（README 未提，create-project 是 composer 通用机制；laravel new 为安装器等价物）
cd myapp

composer setup        # composer install → 复制 .env → key:generate → migrate → npm install → npm run build
composer dev          # php artisan dev：并行启动 HTTP 服务、队列、日志、Vite
composer test         # config:clear + php artisan test

npm run dev           # vite 开发服务器（HMR）
npm run build         # vite 生产构建
php artisan serve     # 备选：单进程开发服务器（框架内置命令，非本仓库脚本）
```

测试配置见 `phpunit.xml`：测试环境强制 `DB_CONNECTION=sqlite` + `:memory:`、缓存/会话/队列用 array 驱动、关闭 PULSE/TELESCOPE/NIGHTWATCH 观测工具。

## 7. 测试与工程质量

- 测试规模：仅 2 个冒烟测试——`tests/Feature/ExampleTest.php`（断言首页返回 200）与 `tests/Unit/ExampleTest.php`（断言 true===true）。骨架的职责是提供测试**基建**（TestCase 基类、phpunit.xml、工厂示例），而非覆盖代码，这与其"模板"定位一致。
- CI（`.github/workflows/tests.yml`，经 gh api 获取）：PHP 8.3/8.4/8.5 三版本矩阵跑 `php artisan test`，push 到 master/*.x 与所有 PR 触发，另有每日定时 cron；全部 GitHub Actions 按 commit SHA 固定版本。
- 其他工作流（gh api 列举）：`dependabot-auto-merge`、`issues`、`pull-requests`、`update-changelog`；`dependabot.yml` 对 GitHub Actions 依赖做每周更新并带 5 天冷却期。
- 供应链细节：`.npmrc` 设 `ignore-scripts=true`（npm 安装不执行第三方生命周期脚本）与 `audit=true`，官方骨架亲自示范 npm 供应链加固。
- 代码风格：`laravel/pint` 在 dev 依赖中，但 composer scripts 未绑定 lint 命令；无 StyleCI（`.gitattributes` 中 `.styleci.yml` export-ignore 为历史残留）。

## 8. 社区与维护状态

- 活跃度极高：pushed_at 2026-10-30（分析前一天），tarball 内文件 mtime 均为 2026-10-30，与 API 一致；gh api 显示 open issues 仅 31（模板仓库 PR 多为依赖小改，问题大多直接修掉）。
- 版本节奏：跟随 Laravel 主版本，release 以 `v13.x.y` 标记骨架自身变更（最新 v13.11.0，2026-10-06，gh api）；默认分支 `13.x`，CI 同时守护 `master` 与 `*.x` 维护分支。
- 贡献者约 397 人（gh api contributors 分页推算，precision 为页数统计）；作为模板仓库，其实质维护者是 Laravel 核心团队，大量改动来自框架新特性发布时的同步更新。
- 历史悠久：created_at 2011-06-08，是 GitHub 上最老牌的 PHP 仓库之一，85k 星里包含大量"用 Laravel 即 star"的生态性关注。

## 9. 亮点与风险

亮点：
1. **约定即文档**：58 个文件就是一份可执行的"标准 Laravel 应用长什么样"规范，`bootstrap/app.php` 30 行讲清全部装配（第 5.1 节）。
2. **零依赖启动体验**：SQLite + database 驱动的缓存/队列/会话 + welcome 页内联 CSS 兜底，新项目不装任何外部服务即可运行（`config/database.php`、`resources/views/welcome.blade.php`）。
3. **第一时间落地新特性**：PHP 8 属性注解模型（`app/Models/User.php`）、Tailwind 4 Vite 插件、Vite 8，官方骨架始终是最新 API 的示范间。
4. **AI 时代的配套意识**：AGENTS.md/CLAUDE.md + Laravel Boost，官方模板已为 AI 编码 agent 提供一等公民指引（`AGENTS.md`）。
5. **工程细节扎实**：CI 三版本矩阵 + SHA 锁定 + Dependabot 冷却期、`.npmrc` ignore-scripts（`.github/workflows/`、`.npmrc`）。

风险/局限：
1. **仓库边界易误解**： star 数反映生态而非代码量；想读框架源码应去 laravel/framework，本仓库无可复用逻辑。
2. **追新即破坏面**：默认值激进（Laravel 13、PHP 8.3+、Vite 8、Tailwind 4、SQLite 默认），从旧版项目升级时大量默认变更需人工对齐，历史版本间骨架差异极大。
3. **示例过于单薄**：2 个冒烟测试、1 个空控制器，认证、授权、API 等实际起步需求依赖独立 starter kit 或文档，骨架本身不提供。
4. **新依赖信息不明**：`laravel/pao`、`@laravel/multiplex` 在骨架内无任何使用痕迹与说明（本次分析无法确认其用途）。

## 10. 总结评价

laravel/laravel 不是"代码库"而是"约定库"——它以 58 个文件定义了十年间 PHP 最主流 Web 框架的标准项目形态，并始终作为框架新特性（流式 bootstrap、属性注解模型、Vite/Tailwind 现代前端链、AI agent 指引）的第一落地点。它适合两类读者：要创建 Laravel 项目的人（直接 create-project 使用即可）和想研究"脚手架设计"的人（对比历史 tag 可清晰看到 Laravel 十余年的理念演进，从 Kernel 分散配置到单文件装配、从 MySQL 默认到 SQLite 默认）。想理解框架本身的实现，请转向 laravel/framework 仓库。
