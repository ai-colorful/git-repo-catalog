# NanmiCoder/MediaCrawler 项目分析

> 星数 67771（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：基于 Playwright 浏览器自动化登录态的多平台自媒体数据采集工具，覆盖小红书、抖音、快手、B站、微博、贴吧、知乎七个中文主流平台的内容、评论与创作者信息抓取（README.md「项目简介」）。

| 事实 | 值 | 出处 |
| --- | --- | --- |
| 名称 | MediaCrawler（mediacrawler） | pyproject.toml |
| 星数 / Fork | 67,771 / 13,125 | 任务参数 / gh api |
| 主语言 | Python（186 个 .py 文件，约 3.25 万行） | 源码统计 |
| 前端 | TypeScript + React 18（webui/） | webui/package.json |
| 许可证 | NON-COMMERCIAL LEARNING LICENSE 1.1（自定义，禁商用，GitHub 识别为 NOASSERTION） | LICENSE |
| 创建时间 | 2023-06-09 | gh api repos/NanmiCoder/MediaCrawler |
| 最近 push | 2026-10-27（持续活跃） | gh api |
| 默认分支 | main（tarball 无 tag，release 数 0，未确认其发版习惯） | gh api |

## 2. 项目是什么，解决什么问题

MediaCrawler 解决的核心问题是：中文自媒体平台反爬强度高、签名算法复杂，普通开发者很难直接调用其 Web API。项目的技术路线是**绕开 JS 逆向，用 Playwright 驱动真实浏览器完成扫码/手机号/Cookie 登录并保存登录态，之后在带登录态的浏览器上下文环境里获取签名参数，再用 httpx 异步客户端批量请求平台接口**（README.md「技术原理」；media_platform/xhs/client.py）。这把"逆向加密算法"的高门槛降成了"配置 + 扫码"的低门槛，是它在中文社区快速积累六万多星的根本原因。

功能上支持三种爬取模式（main.py、config/base_config.py）：关键词搜索（`--type search`）、指定帖子/视频 ID 抓详情与二级评论（`--type detail`）、指定创作者主页（`--type creator`）。七平台的功能矩阵（搜索/详情/二级评论/创作者主页/登录态缓存/IP代理/词云）在 README 功能表中全部标 ✅。

工程形态上它已从"单脚本爬虫"演进为"带 Web 控制台的爬虫框架"：新增 api/（FastAPI 后端，含 crawler/data/websocket 三个路由）与 webui/（React + Vite + Tailwind 控制台，构建产物输出到 api/webui/ 由后端静态托管，api/main.py），可通过 `uv run uvicorn api.main:app` 启动图形界面操作爬虫（README.md 运行章节）。

需要特别说明的是项目的合规姿态：README 顶部与结尾有大段免责声明，LICENSE 为自定义"非商业学习许可证"，main.py 文件头写明五条使用原则（遵守平台条款与 robots.txt、不得大规模爬取、控制请求频率等）；教学版还内置了隐私脱敏层（见第 5 节）。同时 README 与 docs/（mediacrawlerpro订阅.md、知识付费介绍.md）明显导流到收费的 MediaCrawlerPro（断点续爬、多账号、去 Playwright），开源版承担"引流 + 教学"双重角色。

## 3. 技术栈与依赖

- 语言与运行时：Python >= 3.11（pyproject.toml），包管理主推 uv（README 安装章节，uv.lock 随仓分发）。
- 浏览器自动化：playwright >= 1.61（登录、签名上下文、CDP 模式）；自带 libs/stealth.min.js 反检测脚本。
- HTTP 客户端：httpx 0.28.1（异步 API 请求，tools/httpx_util.py 统一构建 client，支持配置关闭 SSL 校验以配合抓包调试）。
- 签名执行：pyexecjs（执行 libs/douyin.js、libs/zhihu.js 生成 a_bogus 等参数，media_platform/douyin/help.py）；小红书另引入纯算法签名库 xhshow >= 0.2.0（media_platform/xhs/playwright_sign.py 注释）。
- 存储：SQLAlchemy 2.x 异步 + alembic（迁移）+ aiomysql/asyncmy（MySQL）、aiosqlite（SQLite）、asyncpg（PostgreSQL）、motor（MongoDB）；pandas + openpyxl（Excel 导出）。
- Web 控制台：FastAPI 0.110 + uvicorn（api/main.py）；前端 React 18 + TypeScript + Vite + Radix UI + Tailwind + zustand + TanStack Query + i18next（webui/package.json）。
- 辅助：tenacity（重试）、redis 4.6（可选分布式缓存）、jieba + wordcloud + matplotlib（评论词云）、opencv-python + pillow（滑块验证码处理，tools/slider_util.py）、typer（CLI）、pydantic v2（模型校验）。

依赖面相当宽（pyproject 列 30+ 项），属于"教学项目全家桶"风格，几乎每个功能点都有对应依赖。

## 4. 目录结构与代码组织

| 目录 | 职责 | 出处 |
| --- | --- | --- |
| main.py | 入口：CrawlerFactory 按平台实例化爬虫，生命周期与清理 | main.py |
| base/ | 四个抽象基类：AbstractCrawler / AbstractLogin / AbstractStore / AbstractApiClient | base/base_crawler.py |
| media_platform/ | 7 个平台子包（xhs/dy/ks/bili/wb/tieba/zhihu），每包统一为 core/client/login/help/field/media 等文件 | media_platform/ |
| store/ | 每平台一个存储包 + 工厂，实现 csv/json/jsonl/db/excel 多落盘方式 | store/xhs/__init__.py |
| database/ | SQLAlchemy 异步模型（15 张表：7 平台 × 内容+评论）、会话管理、MongoDB 基类 | database/models.py |
| proxy/ | IP 代理池：providers（快代理/豌豆HTTP/极致HTTP/静态）、proxy_ip_pool、proxy_mixin | proxy/ |
| config/ | base_config + 每平台配置（纯 Python 常量文件，非 .env） | config/base_config.py |
| tools/ | CDP 浏览器管理、浏览器启动、滑块验证码、异步文件写入、用户脱敏、词库等横向工具 | tools/ |
| cache/ | 本地/Redis 双实现缓存抽象（登录态去重等） | cache/cache_factory.py |
| media_downloader/ | 封面/视频/图片下载与 ffmpeg 处理 | media_downloader/ |
| api/ + webui/ | FastAPI 后端 + React 前端控制台 | api/main.py |
| test/ + tests/ | 共 34 个测试文件（单元 + 集成 + 媒体下载 e2e） | tests/ |
| docs/ | 约 20 篇中文文档：架构（含 mermaid 图）、CDP 指南、代理、存储、FAQ、Pro 订阅 | docs/ |
| libs/ | douyin.js、zhihu.js 签名脚本与 stealth.min.js | libs/ |

组织方式评述：单仓单体（非 monorepo），但采用"抽象基类 + 每平台平行子包"的插件式结构，新增平台即新增一个 media_platform/<p>/ 与 store/<p>/；webui 虽用 npm 管理但与 Python 同仓。整体是"框架壳 + 平台实现"的经典分层，docs/项目架构文档.md 用 mermaid 给出了同构的分层图（入口层/核心爬虫层/API客户端层/存储层/基础设施层）。

## 5. 核心架构与关键实现

### 5.1 爬虫主流程与 CDP 反检测模式（media_platform/xhs/core.py）

每个平台爬虫实现 AbstractCrawler，start() 的流程统一为：可选建 IP 代理池 → 启动浏览器（标准 Playwright 模式或 CDP 模式）→ 打开首页 → 创建带签名的 API 客户端 → pong() 探活失败则走登录 → 开始搜索/详情/创作者爬取。CDP 模式是本项目最有特色的反检测设计：`tools/cdp_browser.py` 的 CDPBrowserManager 自动探测本机 Chrome/Edge 安装路径，以 `--remote-debugging-port` 启动用户真实浏览器并经 CDP 接管（config/base_config.py：ENABLE_CDP_MODE 默认 True、CDP_CONNECT_EXISTING 支持直连用户已开的浏览器），借用真实浏览器的 Cookie、扩展与指纹"大幅降低被风控检测的风险"（config 注释原文）。标准模式则回退为注入 libs/stealth.min.js（core.py）。

### 5.2 登录与签名体系（base/base_crawler.py、media_platform/xhs/login.py、playwright_sign.py）

登录抽象为 qrcode/phone/cookie 三式（AbstractLogin），小红书实现的 check_login_state 用 tenacity 每秒轮询、最多 600 次，双重判定：侧栏"我"按钮的 XPath 可见性 + web_session Cookie 是否变化（media_platform/xhs/login.py）。签名方面小红书走双轨：优先用纯算法库 xhshow 在本地生成 x-s/x-t/x-s-common/x-b3-traceid（media_platform/xhs/playwright_sign.py），本地算法（xhs_sign.py，自定义 Base64 表 + CRC32）作补充；抖音则用 execjs 执行 libs/douyin.js 计算 a_bogus（media_platform/douyin/help.py，注明 copy 自 ShilongLee/Crawler）。客户端签名头注入的核心代码：

```python
signs = sign_with_xhshow(
    uri, data, cookie_str=self.headers.get("Cookie", ""), method=method)
self.headers.update({
    "X-S": signs["x-s"],
    "X-T": signs["x-t"],
    "x-S-Common": signs["x-s-common"],
    "X-B3-Traceid": signs["x-b3-traceid"],
})
```
（media_platform/xhs/client.py L111-122）

### 5.3 存储管道与隐私脱敏（store/xhs/__init__.py、store/xhs/_store_impl.py、tools/user_hash.py）

存储层用工厂模式按 SAVE_DATA_OPTION（csv/json/jsonl/sqlite/db/postgres/excel/mongodb）选择实现，每平台包的 __init__.py 暴露 update_xhs_note/update_xhs_note_comment 等函数供爬虫调用，内部经 XhsStoreFactory 分发到具体 Implement 类（store/xhs/__init__.py）。DB 实现基于 SQLAlchemy 异步会话 + 15 张去重表（database/models.py）；Excel 有统一基类并在 main 结束时 flush（store/excel_store_base.py、main.py _flush_excel_if_needed）。值得注意的是教学版的主动数据最小化：tools/user_hash.py 头部声明"不保留任何可定位到真人的个人信息"，user_id 一律 sha256 截断 16 位匿名化、昵称中间打星，IP 归属地/头像/主页链接等直接不采集——在爬虫项目里属于少见的合规内建设计。

### 5.4 横向基础设施（proxy/、cache/、var.py、tools/）

代理子系统由 ProxyIpPool + 三家商业代理 Provider（kuaidaili/wandouhttp/jishu）+ 静态代理组成，支持 IP 有效性校验与自动刷新（proxy/proxy_ip_pool.py、proxy/providers/）。缓存抽象 abs_cache.py 有 local 与 redis 两个实现，经 cache_factory 按配置切换（cache/）。并发上下文用 contextvars 传递关键词与爬虫类型（var.py）。tools/crawler_util.py、tools/slider_util.py（OpenCV 缺口识别）等提供跨平台公用逻辑。

## 6. 构建与运行

来自 README.md 与 pyproject.toml 的实际命令：

```bash
uv sync                                   # 安装依赖（或 pip install -r requirements.txt）
uv run playwright install                 # 安装浏览器内核
uv run main.py --platform xhs --lt qrcode --type search   # 关键词搜索爬取
uv run main.py --platform xhs --lt qrcode --type detail   # 指定帖子详情
uv run main.py --platform xhs --type detail --specified_id <URL> --get_media true
uv run main.py --init_db sqlite           # 初始化数据库
uv run main.py --help                     # 全部参数
# WebUI
cd webui && npm install && npm run build  # 前端产物输出到 api/webui/
uv run uvicorn api.main:app --port 8080 --reload
```

关键配置在 config/base_config.py 直接改常量（PLATFORM/KEYWORDS/LOGIN_TYPE/SAVE_DATA_OPTION 等）；首次运行需扫码登录，登录态按平台缓存到 USER_DATA_DIR（config SAVE_LOGIN_STATE）。

## 7. 测试与工程质量

- 测试规模：test/ 6 个文件（db 同步、本地缓存过期、MongoDB 集成、代理池、Redis 缓存、utils）+ tests/ 28 个文件（各平台 client 分页/登录态/参数构造、CDP 浏览器、命令行参数、Excel 存储、存储工厂、媒体下载含 e2e），共 34 个测试文件；框架 pytest + pytest-asyncio（requirements.txt），tests/media_server.py 提供本地 mock 服务器支撑媒体下载端到端测试。
- CI：仅 .github/workflows/deploy.yml（VitePress 文档部署到 GitHub Pages），未配置测试/静态检查 CI——32k 行规模下这是明显短板（以 .github/workflows/ 目录内容为准）。
- 质量工具：.pre-commit-config.yaml 含自研 file_header_manager 强制版权头检查 + trailing-whitespace/end-of-file-fixer 等通用钩子；mypy.ini 存在但 mypy 未进依赖清单（未确认是否实际启用）。
- 观感：每个文件头部统一版权声明、关键模块中文注释与 docs/ 架构文档齐全，工程规范意识高于一般个人爬虫项目；但配置用 Python 常量文件而非 .env（仅 .env.example 放少量密钥类配置），环境隔离较弱。

## 8. 社区与维护状态

- 活跃度：gh api 显示最近 push 为 2026-10-27（距分析日 4 天），仓库创建于 2023-06-09，open issues 218，watch 260——持续高活跃（gh api repos/NanmiCoder/MediaCrawler）。
- 贡献者：contributors 分页 Link 头显示约 84+ 页（per_page=1），即至少 80 余名贡献者，单人项目已演化为多人维护（gh api contributors）。
- 发布：仓库无 tag、无 GitHub Release（gh api tags/releases 均为空），版本演进靠 main 分支滚动提交，pyproject 版本号长期停留在 0.1.0。
- 商业生态：README 置顶 BrowserAct 广告、Pro 版订阅文档（docs/mediacrawlerpro订阅.md）、捐赠名单（docs/捐赠名单.md）、付费社群与微信交流群文档，作者（程序员阿江-Relakkes）以开源版为获客入口的"开源 + 知识付费"模式运转明显；另同步托管于 AtomGit（README 徽章）。

## 9. 亮点与风险

亮点：
1. 低门槛的多平台覆盖：一套抽象基类 + 七个平台平行实现，签名靠登录态浏览器上下文或现成 JS/算法库，免深度逆向（base/、media_platform/）。
2. CDP 真实浏览器接管是实用且少见的反检测工程化方案，自动化探测浏览器路径、端口与连接复用（tools/cdp_browser.py）。
3. 存储管道完整且可插拔：csv/json/jsonl/sqlite/mysql/postgres/mongodb/excel 八种落盘 + 去重表 + alembic 迁移（store/、database/）。
4. 教学版内置隐私脱敏与合规约束：用户 ID 哈希化、昵称脱敏、不采集定位性字段，文件头与 LICENSE 明确非商业学习用途（tools/user_hash.py、main.py、LICENSE）。
5. 文档密度高：docs/ 下约 20 篇中文指南与 mermaid 架构图，WebUI 控制台降低使用门槛（docs/、api/、webui/）。

风险/局限：
1. 法律与平台条款风险：本项目本质是自动化采集登录后可见的平台数据，与各平台用户协议和反爬条款存在天然冲突；README/docs 已如实列出中国爬虫涉刑案例并限"学习用途"，但工具能力本身不因声明而合规，使用者需自行承担且不应将其用于大规模采集或商业用途（LICENSE、README 免责声明）。
2. 自定义许可证：NON-COMMERCIAL LEARNING LICENSE 1.1 不是 OSI 开源许可证，禁止商用，企业场景引用其代码有法律风险（LICENSE）。
3. 维护脆弱性依赖签名实现：平台签名算法（x-s、a_bogus 等）随平台升级随时失效，社区依赖 xhshow、ShilongLee/Crawler 等第三方签名供给，可用性不受本项目控制（media_platform/xhs/playwright_sign.py、libs/douyin.js）。
4. 无发布与测试 CI：无 tag/release、无测试流水线，升级只能跟随 main 滚动；HEAD 即"稳定版"（gh api、.github/workflows/）。

## 10. 总结评价

MediaCrawler 是中文爬虫开源生态里事实级的参考实现：它把"Playwright 登录态 + 上下文签名"这条务实路线做成了结构清晰的框架（抽象基类、平台子包、可插拔存储、代理池、CDP 反检测、WebUI），并罕见地把脱敏与合规声明内建进代码，六万多星与 80+ 贡献者证明了其社区地位。它的边界同样清楚——自定义非商业许可证、依赖易失效的第三方签名、无测试 CI，以及爬虫工具与生俱来的合规争议，决定了它更适合作为学习与技术研究对象而非生产依赖。适合想理解现代 Web 爬虫架构（异步客户端、签名注入、风控对抗、存储管道）的 Python 开发者精读源码，也适合做小规模个人学习实验；任何真实采集需求都应先评估目标平台条款与当地法律，而非直接套用。
