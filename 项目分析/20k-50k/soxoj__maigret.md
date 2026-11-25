# soxoj/maigret 项目分析

> 星数 35500（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：仅凭一个用户名，在数千个公开站点上异步并发探测账号存在性，并从命中的个人主页聚合可机读的公开身份信息（"档案"），生成多种格式报告的 OSINT 用户名调查工具。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | maigret（名字取自小说侦探 Maigret） | docs/source/philosophy.rst |
| 星数 | 35,500（本清单快照）；fork 3,274 | 任务参数 / gh api repos/soxoj/maigret |
| 语言 | Python，支持 3.10–3.14 | pyproject.toml classifiers |
| 许可证 | MIT | pyproject.toml、LICENSE |
| 版本 | 0.6.6（PyPI Production/Stable） | pyproject.toml |
| 创建时间 | 2020-06-27 | gh api 元数据 |
| 最近 push | 2026-11-13（分析前一天，极活跃） | gh api 元数据 |
| 最新 release | v0.6.6（2026-09-18） | gh api releases/latest |
| 贡献者 | 105 人 | gh api contributors 分页统计 |

## 2. 项目是什么，解决什么问题

Maigret 解决的是"账号枚举只是起点"的问题。它输入一个用户名（或 12 种支持标识符之一，如 vk_id、steam_id、qq_id、bilibili_id、orcid 等，见 maigret/checking.py 的 `SUPPORTED_IDS` 常量），并发地向数据库中数千个站点发起探测请求，判断"这个用户名在这些站点上是否已注册"，然后在命中（CLAIMED）的页面上继续用配套库 socid-extractor 抽取结构化信息：其他用户名、全名、关联账号、头像、生日、地区、性别等，并支持用发现的新 ID/用户名做递归搜索，最终汇总成一份"档案"（README.md "Main features"、docs/source/philosophy.rst "TL;DR: Username => Dossier"）。

与同类工具（如 Sherlock 只做存在性判断）的差异在于三点：一是"找到不等于结束"——内置信息抽取与递归扩展，把单点命中变成关系网络；二是工程化对抗——针对站点封禁做了 TLS 指纹模拟、Cloudflare 绕过、代理/Tor/I2P 路由、失败重试等一整套机制；三是数据规模——站点数据库不只收录社交网络，还通过"引擎模板"批量覆盖了海量论坛/CMS 社区（Discourse、XenForo、phpBB、MediaWiki、Gitea 等），这是其他用户名检查工具没有覆盖的长尾（docs/source/philosophy.rst "Origins" 节，作者自述研究过"existing namecheckers 不真正提供调查价值"）。

项目明确自述出身：docs/source/philosophy.rst 写明 Maigret "started as a fork of Sherlock but has long since outgrown the"。README 的 "Used by" 节列出 Social Links、UserSearch 等商业 OSINT 产品基于它构建；作者另提供每日更新的私有站点库与商业 API（README "Commercial Use"）。

定位是合法合规的公开信息聚合：README Disclaimer 写明 "For educational and lawful purposes only"，要求使用者自行遵守 GDPR/CCPA 等法律；docs 中有专门的 privacy.rst 与 philosophy.rst 说明边界。它只访问公开可匿名访问的页面，不尝试登录、不绕过认证、不做邮箱探测（docs/source/faq.rst 明确拒绝邮箱类查询并推荐了专门的邮箱工具）。

## 3. 技术栈与依赖

- 语言：Python ≥3.10（pyproject.toml）。
- 异步网络栈：aiohttp（并发请求）、aiohttp-socks（SOCKS/HTTP 代理连接器）、aiodns（异步 DNS 解析）、curl-cffi（浏览器 TLS 指纹模拟，绕 WAF）（pyproject.toml dependencies）。
- 信息抽取：socid-extractor（作者本人的结构化身份抽取库，>=0.1.1,<0.2.0）。
- 报告与可视化：Jinja2（HTML 模板）、xhtml2pdf + reportlab（PDF，可选 extras `maigret[pdf]`，因 pycairo 在 Linux/macOS 无 wheel 而默认不装）、XMind、networkx + pyvis（交互关系图）、Neo4j Cypher 导出（自实现，report.py）。
- Web UI：flask（async extras）（maigret/web/app.py）。
- 其他：alive_progress（进度条）、colorama、pycountry（国家标签）、html5lib/lxml/soupsieve（页面解析）、PySocks、certifi、platformdirs。
- 工程化：Poetry 构建（poetry-core backend）、pytest 全家桶、black/flake8/mypy、coverage --fail-under=60、Poetry.lock 锁定版本。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 | 出处 |
|---|---|---|
| maigret/ | 核心包（全部逻辑约 9,000 行 Python） | tarball 文件树 |
| maigret/checking.py | 并发探测引擎与结果判定（1,976 行，最大模块） | 行数统计 |
| maigret/maigret.py | CLI 入口（run）、参数解析（argparse）、搜索编排 | maigret/maigret.py |
| maigret/sites.py | MaigretSite/MaigretEngine/MaigretDatabase 数据模型与加载 | maigret/sites.py |
| maigret/report.py | 9 种报告格式生成（1,085 行） | maigret/report.py |
| maigret/executors.py | 异步队列并发执行器（112 行） | maigret/executors.py |
| maigret/db_updater.py | 站点库自动更新（24h 检查 meta → 拉新库缓存 ~/.maigret/） | maigret/db_updater.py |
| maigret/error_detection.py / errors.py | WAF/封禁页检测、错误分类 | — |
| maigret/activation.py | Twitter guest token、Vimeo 等站点"激活器" | maigret/activation.py |
| maigret/ai.py | AI 分析模块（OpenAI 兼容 API 流式总结） | maigret/ai.py |
| maigret/web/ | Flask Web 界面（app.py 714 行 + Jinja2 模板） | maigret/web/app.py |
| maigret/resources/ | data.json（站点库，3.69MB）、报告模板、AI prompt | tarball 文件树 |
| utils/ | 维护工具：update_site_data.py、selfcheck_triage.py、site_check.py 等 | — |
| tests/ | 31 个 test_*.py | tarball 文件树 |
| .github/workflows/ | 6 个工作流：测试、CodeQL、Docker 构建、PyInstaller、PyPI 发布、站点数据更新 | tarball 文件树 |
| docs/ | Sphinx 文档（readthedocs），含英/法/中三语 README 与 docs i18n | docs/ |
| pyinstaller/ | Windows standalone exe 打包 | — |

组织方式评述：单包单体应用（非 monorepo），核心逻辑集中在一个 `maigret/` 包内按职责分文件；数据（站点库）与代码分离为 JSON；外围有独立维护工具目录 utils/ 与打包目录 pyinstaller/。模块边界清晰：数据模型（sites.py）→ 执行引擎（executors/checking）→ 结果（result.py）→ 输出（report.py/notify.py），是教科书式的管道分层。

## 5. 核心架构与关键实现

### 5.1 站点数据库：引擎模板机制（maigret/resources/data.json + maigret/sites.py）

data.json（3,693,773 字节）顶层三键：`sites`（9,535 条）、`engines`（28 种）、`tags`（75 种）。实测统计：3,192 条有显式 `url`；其余约 6,300 条仅声明 `engine` 字段，由引擎模板在加载时批量展开成完整探测配置。引擎本质是社区软件的"指纹+探测模板"：

- DiscourseJson 1,232 条、Discourse 912 条、XenForo 794 条、uCoz 709 条、phpBB/Search 611 条、Gitea 422 条、MediaWikiJson 391 条、GitLab 287 条、Flarum 207 条、Lemmy 143 条、vBulletin 141 条、Mastodon 77 条、MediaWiki 94 条等（data.json 实测计数）。也就是说数据库的主力是论坛/自建社区的长尾站点，靠 28 个引擎复用一套探测逻辑。
- 探测方式分类（实测计数 + checking.py `process_site_result`）：`checkType` 分 message（presenseStrs/absenceStrs 字符串匹配）、status_code（2XX 判定）、response_url（禁重定向+状态码+presence 标志）三类；168 条带 `regexCheck`（URL 用户名合法性预校验）。
- 每条站点带 `usernameClaimed` / `usernameUnclaimed` 校准对，供维护者 `--self-check` 对线上站点做防误报审计（README "Useful CLI flags"、checking.py `site_self_check`）。
- tags 含类别（forum 5,180、discussion 4,200、tech 1,331、coding 1,256、wiki 1,108、social 866、gaming 733…）与国家码（ru 1,558、de 462、fr 244、cn 等），支撑 `--tags` 过滤与"默认只查流量前 500、`-a` 查全部"的策略（README "Main features"）。

sites.py 中 `MaigretSite.update_from_engine()` 把引擎字段合并进站点条目（站点可逐字段覆盖引擎默认值），`strip_engine_data()` 保存时剥离冗余，保持 JSON 精简。

### 5.2 并发探测引擎（maigret/checking.py + maigret/executors.py）

主入口 `maigret()` 协程把全部站点任务投给 `AsyncioQueueGeneratorExecutor`：N 个 worker 协程（默认 `max_connections=100`）+ asyncio.Queue 消费，worker 内用 `asyncio.wait({task}, timeout)` 而非 wait_for，避免被反爬站点"挂住连接不完成"拖死整个扫描；超时任务后台取消并打日志（executors.py 注释，issue 跟踪经验）。失败站点在重试循环中自动重跑（`get_failed_sites`），Ctrl+C 时通过 `output_container` 原地落盘部分结果。

Checker 按"防护等级"三级派发（checking.py `make_site_result`，约 978-1006 行）：

1. 站点 `protection` 匹配 Cloudflare 触发条件且用户开启绕过 → `CloudflareWebgateChecker`（可配 FlareSolverr json_api 或 CloudflareBypassForScraping url_rewrite 后端，checking.py `build_cloudflare_bypass_config`）；
2. 需要 `tls_fingerprint` → `CurlCffiChecker`：curl_cffi `impersonate='chrome'` 模拟真实浏览器 TLS/JA3 指纹，并刻意剥掉自定义 User-Agent 以免"UA 与 TLS 指纹组合"触发组合评分（checking.py 460-490 行注释）；
3. 默认 `SimpleAiohttpChecker`：aiohttp + `ProxyConnector`（aiohttp-socks），SSL 上下文禁验证但保留真实握手以规避指纹封锁（checking.py `check()`）。

协议路由按站点协议分流四种 checker 工厂：clearweb（直连/代理）、tor（仅 .onion）、i2p（仅 .i2p）、dns（aiodns 域名探测）。细节打磨多：三种 transport 对 socks5/socks5h 语义不同，`normalize_proxy_scheme` 统一改写为代理侧 DNS（issue #2955）；旋转代理中途换出口导致的瞬态错误只重试一次且不重试凭据错误；"302 自重定向 + Set-Cookie"识别为会话握手而非用户不存在（joyreactor.cc 案例，checking.py 注释）；Twitter/Vimeo 类站点先用 `ParsingActivator` 领 guest token 再重查（activation.py）。

判定核心 `process_site_result`（755-925 行）：先经 `detect_error_page` 识别 WAF/封禁页降级为 UNKNOWN，再按 checkType 判 CLAIMED/AVAILABLE，命中后若开启 `--parse`/enrich 则调 socid_extractor 抽取 ID、并触发 URL 变体递归（`run_url_mutations`）。对非 ASCII 用户名（中文等）有专门防误报：返回页不含用户名本身即判不命中（#2633）。

### 5.3 报告与输出（maigret/report.py，1,085 行）

9 种输出格式：HTML（Jinja2 模板 resources/simple_report.tpl）、PDF（xhtml2pdf + reportlab，模板 simple_report_pdf.tpl + CSS，RTL 文本有 arabic-reshaper/python-bidi 处理）、CSV、TXT、JSON（simple/ndjson 两型）、Markdown、XMind（含 zip 归一化与清单校验 `_normalize_xmind_archive`）、交互关系图（networkx 建图 → pyvis D3 HTML `save_graph_report`）、Neo4j Cypher 脚本（`_graph_to_cypher`，可幂等重导入）。报告按"数据点"数量排序（`sort_report_by_data_points`），即信息量越大的命中越靠前——这是与"只列站点名"工具的观感差异。

### 5.4 站点库分发与自动更新（maigret/db_updater.py）

每次运行检查 GitHub raw 上的轻量 meta 文件（`db_meta.json`，含版本/哈希），距上次检查超 24h 且版本兼容才下载新库缓存到 `~/.maigret/data.json`，离线则回退内置 data.json（db_updater.py 头注释与常量）。CI 端 `update-site-data.yml` 在每次 push 后自动重算 sites.md 与 db_meta.json——数据更新是全自动闭环，站点维护者的 PR 合入后即到达全部用户。

## 6. 构建与运行

来自 pyproject.toml（`[tool.poetry.scripts]`：`maigret = "maigret.maigret:run"`）与 README：

```bash
pip install maigret                 # 或 poetry install --with dev
maigret username                    # 默认查流量前 500 站点
maigret username -a                 # 全库扫描
maigret username --html --pdf --xmind   # 报告输出
maigret username --tags photo,dating     # 按类别/国家过滤
maigret username --proxy socks5://127.0.0.1:1080   # 代理/Tor/I2P
maigret username --ai               # AI 总结（需 OPENAI_API_KEY）
maigret --web 5000                  # Flask Web UI
maigret --self-check                # 维护者校准站点库

# 测试（CI 实际命令）
poetry run coverage run --source=./maigret -m pytest --reruns 3 --reruns-delay 5 tests
poetry run coverage report --fail-under=60

# Docker
docker pull soxoj/maigret           # CLI 版
docker run -p 5000:5000 soxoj/maigret:web   # Web UI 版
```

## 7. 测试与工程质量

- 测试规模：tests/ 共 36 个文件，其中 test_*.py 31 个；框架 pytest + pytest-asyncio + pytest-httpserver（本地 mock HTTP 服务器测探测逻辑）+ pytest-rerunfailures（网络抖动重跑）+ pytest-cov（tarball tests/ 与 .github/workflows/python-package.yml）。
- CI：python-package.yml 在 Python 3.10–3.14 五版本矩阵跑 lint+coverage（阈值 60%），另有 `minimal-install` job 专门验证无 [pdf] extra 时核心不被可选依赖污染；codeql-analysis.yml 安全扫描；pyinstaller.yml 与 build-docker-image.yml 负责发布产物；update-site-data.yml 数据自动化。CI 体系完整且明显经过长期打磨。
- 代码质量观察：注释密度高且大多是"为什么"而非"是什么"——记录了 issue 号（#2633、#2688、#2955）、具体站点案例（joyreactor.cc 自重定向）、传输层语义差异等实战经验；mypy/black/flake8 全配。单个模块偏大（checking.py 近 2,000 行），但内部结构（Checker 类层次 + 纯函数判定）仍清晰。

## 8. 社区与维护状态

- 活跃度极高：最近 push 2026-11-13（分析前一天）；50 个 open issue；3,274 fork；105 名贡献者（gh api 元数据与 contributors 分页）。
- Release 节奏：PyPI 版本 0.6.6（2026-09-18 发布），CITATION.cff 规范引用，Patreon 赞助入口（pyproject.urls）。
- 生态位：README "Used by" 列出 Social Links（商业 Crimewall）、UserSearch 等基于它构建的商业产品；提供 Telegram bot、Cloud Shell/Colab/Binder/Replit 一键运行、snap 包、PyInstaller exe、Docker 双镜像——分发渠道覆盖了几乎全部使用场景。
- 维护自动化程度高：站点数据更新、db meta 生成、changelog 生成（utils/generate_changelog.py）均由 CI 承担；另有 selfcheck_triage.py 帮助维护者分诊 self-check 失败。
- README 有法语/中文翻译，docs 有 i18n po 文件（docs/source/locale/），社区国际化投入明显。

## 9. 亮点与风险

亮点：

1. 引擎模板数据架构：28 个引擎把约 6,300 条论坛/社区长尾站点压缩进一个 3.7MB JSON，单 PR 即可批量新增站点，数据密度与可维护性远超逐站手写的同类库（data.json + sites.py `update_from_engine`）。
2. 对抗能力工程化：TLS 指纹模拟、Cloudflare 绕过后端可插拔、Tor/I2P/代理协议路由、瞬态错误重试、非 ASCII 用户名防误报——每条机制都有 issue 编号与真实案例背书（checking.py）。
3. "找到之后"的纵深：socid-extractor 抽取 + ID 变体递归 + 9 种报告（含图数据库导入），把用户名检查升级为可入库的调查管线。
4. 全自动数据闭环：站点库 24h 自动更新 + CI 自动重算 meta，站点失效的衰减速度被系统性压制（db_updater.py、update-site-data.yml）。
5. 分发与可用性：PyPI、Docker、snap、standalone exe、Web UI、云 Notebook 全覆盖，且 Web UI 可一键部署到 Render 免费档（README）。

风险/局限（客观技术视角）：

1. 滥用边界：工具聚合的是公开信息，但其"一人一档"的输出形态天然适合骚扰、人肉搜索、追踪等用途；项目以 MIT 无限制开源（README "Commercial Use"），仅靠免责声明约束用途（README Disclaimer），无技术性防滥用措施；`maigret --web` 部署到 Render 时无登录（README 自述），公网可达实例可能被第三方匿名使用。
2. 站点库对抗性衰减：5,935+ 站点的存在性判断依赖各站点响应模式，站点改版/WAF 升级即产生误报，`--self-check` 机制只能覆盖维护者主动审计的部分；README 直言"site checks break over time and need active maintenance"。
3. 法律合规随辖区而变：GDPR/CCPA 对个人信息聚合的适用性由使用者判断，企业内使用此类工具前需合规评估（README Disclaimer 未提供具体指引，docs/privacy.rst 偏隐私哲学说明）。
4. 工程局限：checking.py 单文件近 2,000 行、核心依赖 socid-extractor 版本锁定 <0.2.0（上游 API 变动敏感）、PDF extra 依赖系统级 cairo 导致部分平台安装摩擦（pyproject 注释）。

## 10. 总结评价

Maigret 是用户名 OSINT 领域事实上的标杆项目：从 Sherlock 分叉起步，靠"引擎模板化站点库 + 异步并发引擎 + 信息抽取纵深 + 全自动数据维护"四层设计把一个查重名工具做成了完整的身份调查管线，且以 105 位贡献者、次日仍在 push 的活跃度持续进化。代码库是学习"大规模异步站点探测如何对抗真实世界退化"的优质范本——checking.py 里每条注释都是一节免费的实战课。适合 OSINT 研究者与安全从业者做授权范围内的账号关联调查，适合 Python 工程师学习 aiohttp 大规模并发与数据驱动架构；同时应清醒认识到其输出天然可被滥用于骚扰与人肉，任何使用都应以合法授权为前提。
