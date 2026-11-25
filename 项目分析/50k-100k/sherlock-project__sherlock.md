# sherlock-project/sherlock 项目分析

> 星数 94775（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：Sherlock 是一款命令行 OSINT 工具，输入用户名即可在 400+（当前清单 481 个）社交网络与网站上并行探测该用户名是否已被注册，用于跨平台账号足迹搜寻。

| 项 | 值 | 出处 |
| - | - | - |
| 名称 | sherlock-project/sherlock | pyproject.toml |
| 星数 | 94775（watchers 95246，forks 11331） | gh api 快照 2026-10-31 |
| 语言 | Python（^3.9，CI 矩阵 3.10–3.13） | pyproject.toml / tox.ini |
| 许可证 | MIT © 2019 Sherlock Project | LICENSE |
| 版本 | v0.16.2（2026-09-08 发布） | gh api releases/latest |
| 最近活跃 | 最近 push 2026-10-30（分析前一天），持续维护 | gh api repos |

## 2. 项目是什么，解决什么问题

在互联网上，同一个人往往在多个平台使用相同或相似的用户名。Sherlock 解决的问题是：给定一个用户名，快速回答"这个用户名在哪些网站上存在"。它由 Siddharth Dushantha 于 2018 年 12 月创建（gh api `created_at: 2018-12-24`），现由 Paul Pfeister、Matheus Felipe、Sondre Karlsen Dyrnes 三位 maintainer 维护（pyproject.toml），是 OSINT（开源情报侦察）领域最知名的入门级工具之一，关键词标注为 `osint / reconnaissance / information gathering`（pyproject.toml）。

工作方式非常直接：每个目标站点在清单里维护一条"用户主页 URL 模板"（如 `https://github.com/{}`）和一条判定规则。程序把用户名填入模板并发请求，按响应状态码、错误消息文本或重定向行为判定该用户名是 CLAIMED（已占用）、AVAILABLE（可用）、UNKNOWN（出错）、ILLEGAL（不符合站点用户名规则）还是 WAF（被防护拦截）（sherlock_project/result.py）。所有 481 个站点并发查询，几秒到几十秒内汇总结果。

默认情况下程序不使用打包内的清单，而是从 `https://data.sherlockproject.xyz` 拉取最新站点清单，并叠加一份"误报排除列表"（`EXCLUSIONS_URL`，指向 GitHub exclusions 分支的 false_positive_exclusions.txt），以减少陈旧数据导致的假阳性（sherlock_project/sites.py 的 MANIFEST_URL/EXCLUSIONS_URL 与构造函数逻辑）。用户也可用 `--local` 强制本地清单，或 `--json` 指定自定义清单、甚至直接传上游 PR 编号来测试未合并的站点条目（sherlock.py `main()` 中对 `args.json_file.isnumeric()` 的处理）。

该工具定位为正当用途的侦察辅助（渗透测试前期信息收集、自名号足迹自查），README 未内嵌许可证之外的合规声明；但其能力天然具有双刃剑属性，风险详见第 9 章。

## 3. 技术栈与依赖

- 语言：纯 Python 3.9+，无 C 扩展自编译；类型标注完整（含 py.typed 标记，sherlock_project/py.typed）。
- 构建：Poetry（`poetry-core` 后端），入口 `sherlock = sherlock_project.sherlock:main`（pyproject.toml `[tool.poetry.scripts]`）。
- 运行时依赖（pyproject.toml）：
  - `requests` + `requests-futures`：HTTP 请求与基于线程池的并发 Future 封装——这是整个探测引擎的底座；
  - `PySocks` / `stem`：SOCKS 代理支持与 Tor 控制（stem 为 Tor 控制库，实际代码中未见直接引用，属历史遗留依赖，未确认使用点）；
  - `pandas` + `openpyxl`：仅为 `--xlsx` 导出 Excel 报表服务（sherlock.py 尾部 `pd.DataFrame(...).to_excel`）；
  - `colorama`：跨平台终端着色；`certifi`：CA 证书；`tomli`：未安装包时从 pyproject.toml 读版本号（`__init__.py get_version()`）。
- 开发依赖：`pytest` + `pytest-xdist`、`jsonschema`（校验 data.json）、`rstr`（按正则反向生成随机用户名，用于误报测试）。
- 分发：PyPI 包名 `sherlock-project`、Docker 镜像 `sherlock/sherlock`、dnf/Homebrew/Kali/BlackArch 等社区包（README 安装表）；另含 Apify Actor 封装（.actor/actor.json）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| - | - |
| sherlock_project/ | 全部主代码，单包扁平结构（6 个 py 文件共约 1633 行） |
| sherlock_project/resources/data.json | 站点清单：481 个站点的探测定义（约 102KB） |
| sherlock_project/resources/data.schema.json | 清单 JSON Schema（149 行） |
| tests/ | 8 个测试文件共约 446 行 |
| .github/workflows/ | 4 个 CI 工作流（回归、目标校验、站点列表同步、排除列表） |
| .actor/ | Apify 平台 Actor 封装（Dockerfile、input/dataset schema） |
| devel/ | 维护者脚本：site-list.py（生成官网站点列表页）、summarize_site_validation.py |
| docs/ | 行为准则、被移除站点说明（removed-sites.md）、镜像资源 |
| Dockerfile / pytest.ini / tox.ini / pyproject.toml | 容器化与工程配置 |

组织评述：典型的"小型核心 + 数据驱动"单体。代码量极小（核心引擎 sherlock.py 934 行），项目复杂度几乎全部沉淀在 data.json 数据文件里——新站点贡献通常只需改 JSON 而非代码，这使非程序员也能参与，是它能积累近千贡献者与 481 个站点的关键结构原因。

## 5. 核心架构与关键实现

### 5.1 站点清单：data.json 与其 Schema

每个站点条目必填 `url`（用户主页模板，`{}` 为占位符）、`urlMain`、`errorType`、`username_claimed`（用于校验站点可用的已知存在用户名），可选 `errorMsg`/`errorCode`、`urlProbe`（独立探测地址）、`regexCheck`（用户名合法性正则）、`headers`、`request_method`/`request_payload`、`isNSFW`、`tags`（data.schema.json 的 required/properties）。实际统计（Python 解析 data.json）：

- 判定方式分布：`status_code` 327 个、`message` 127 个、`response_url` 27 个——多数站点靠 HTTP 状态码即可判定，无需下载整页；
- 52 个站点定义了独立 `urlProbe`，7 个需要自定义 headers，3 个用 POST、24 个显式 GET，19 个标记 isNSFW。

条目示例（data.json `1337x`）：`errorType: "message"`，`errorMsg` 列出两种 404 页面特征，命中即 AVAILABLE。

### 5.2 请求引擎：两阶段 Future 流水线（sherlock.py）

核心函数 `sherlock()` 把流程拆成两轮循环：第一轮为所有站点创建异步 Future（线程池上限 20 workers），请求先全部发出；第二轮再逐个 `future.result()` 收割结果。这样总耗时近似等于最慢站点而非所有站点之和：

```python
# sherlock.py — 摘要
if len(site_data) >= 20:
    max_workers = 20
session = SherlockFuturesSession(max_workers=max_workers, session=underlying_session)
for social_network, net_info in site_data.items():
    ...
    future = request(url=url_probe, headers=headers,
                     allow_redirects=allow_redirects, timeout=timeout, ...)
    net_info["request_future"] = future   # 第一轮：全部发射
for social_network, net_info in site_data.items():
    r, error_text, exception_text = get_response(net_info["request_future"], ...)  # 第二轮：收割
```

自定义的 `SherlockFuturesSession` 通过 response hook 记录每个请求的耗时（sherlock.py 顶部），最终写入结果供 CSV/XLSX 输出。

### 5.3 匹配规则与反误报设计（sherlock.py 判定段）

三种判定方式按清单声明组合评估：`message` 在响应文本中查找 errorMsg（找到=AVAILABLE）；`status_code` 将响应码与 errorCode 对照（默认 2xx 为 CLAIMED，>=300 或 <200 为 AVAILABLE）；`response_url` 禁用重定向后按状态码判定。判定前有两道闸门：

1. `regexCheck`：用户名不符合站点正则则直接标记 ILLEGAL，连请求都不发（sherlock.py `re.search(regex_check, username) is None` 分支）；
2. WAF 指纹过滤：内置一组 Cloudflare/Cloudfront(AWS)/PerimeterX 挑战页特征串，命中则返回 WAF 状态而非误判为 AVAILABLE（sherlock.py `WAFHitMsgs` 列表，注释要求每条注明目标与日期）。

判定结果还依赖 `username_unclaimed` 的设计：sites.py 用 `secrets.token_urlsafe(32)` 生成随机不可猜测用户名作为"必然不存在"的对照样本，避免硬编码对照值被污染。

### 5.4 清单加载与排除机制（sites.py）

`SitesInformation` 支持三种数据源（URL / 绝对或相对路径），默认拉取线上清单；随后请求排除列表并 `pop` 掉已知误报站点（可用 `--ignore-exclusions` 关闭、`--site` 指定站点时豁免排除）。NSFW 过滤独立成 `remove_nsfw_sites()`，由 `--nsfw` 开关控制（sherlock.py main 中默认调用）。

## 6. 构建与运行

```bash
# 安装（README 安装表）
pipx install sherlock-project        # 或 pip/uv
docker run -it --rm sherlock/sherlock dretch   # Docker（镜像 sherlock/sherlock）

# 运行（README 用法）
sherlock user123                     # 单用户名
sherlock user1 user2                 # 多用户名
sherlock dretch --csv --timeout 30   # 常用选项：CSV 导出、超时
sherlock dretch --site GitHub --site Docker Hub   # 限定站点

# 测试（tox.ini / pytest.ini）
tox -e lint                          # ruff 检查
tox -e py312                         # 构建+coverage+pytest
pytest -v -m "not online"            # 离线子集
```

注意：不运行 `python3 sherlock/sherlock.py`，需按模块方式执行——`__init__.py` 用 `import_error_test_var` 哨兵变量检测旧式脚本运行并提示正确安装方式（sherlock_project/__init__.py、`__main__.py` 校验 Python >= 3.9）。

## 7. 测试与工程质量

- 测试规模：tests/ 下 8 个文件约 446 行，属于精悍型。分层明确（pytest.ini markers）：`online` 标记需联网的活体目标测试（test_probes.py 用 GitLab/GitHub/Docker Hub 等稳定站点验证三种判定路径的正/反例）；`validate_targets` 标记全清单扫描式校验（test_validate_targets.py 用 rstr 按各站点 regexCheck 反向生成随机用户名，验证"必然不存在"的用户名不会返回 CLAIMED，即误报检测）；另有 manifest schema 校验（test_manifest.py）、Unicode 用户名（test_unicode.py）、UX/版本号等测试。
- CI：regression.yml 以 tox 驱动多 OS 多 Python 版本矩阵 + lint；validate_modified_targets.yml 只对修改到的站点条目做在线校验（控制 CI 成本的聪明做法）；update-site-list.yml 在 data.json 变更后自动重新生成官网站点列表页并推送到 sherlockproject.xyz 仓库（.github/workflows/）。
- 质量观察：类型标注覆盖率很高；`pytest.ini` 默认排除重型 validate_targets；Dockerfile 基于 python:3.12-slim，运行时直接 pip 安装 PyPI 包而非拷贝源码（镜像即发布物，Dockerfile 注释给出发版流程）。不足：xlsx 导出段（sherlock.py `--xlsx` 分支）存在 `response_time_s` 未按站点初始化的逻辑瑕疵（循环前仅 None 判断），pandas 全量引入略重。

## 8. 社区与维护状态

- 活跃度：最近 push 2026-10-30（分析前一天），创建于 2018-12-24，7 年持续维护（gh api）。最新 release v0.16.2 于 2026-09-08 发布，版本节奏慢但稳定（0.16.x 已迭代多个 patch）。
- 贡献者：contributors API 分页页尾为 271（per_page=1 推算约 270+ 人）；README 用 contrib.rocks 展示贡献者墙。项目设有 CODEOWNERS、4 类 issue 模板（含 false-positive / false-negative / site-request 专项模板，.github/ISSUE_TEMPLATE/），说明站点数据维护高度依赖社区 issue 驱动。
- 生态：官网 sherlockproject.xyz 与 data.sherlockproject.xyz 数据端点由仓库内 CI 同步驱动；另有 Apify Actor 商店分发。open issues 355，多为站点失效/新增请求（符合此类项目常态，未逐条确认）。

## 9. 亮点与风险

亮点：
1. 数据驱动架构：代码引擎极小（核心判定逻辑集中在 sherlock.py 一个函数），站点扩展全部收敛到 data.json + JSON Schema，贡献门槛极低，社区可持续扩清单；
2. 两阶段并发 + HEAD 优先：Future 全发射再收割，`status_code` 类目标用 HEAD 省带宽（sherlock.py 请求选择段），481 站点一次查询体验流畅；
3. 反误报体系完整：regexCheck 前置、WAF 指纹过滤、线上排除列表、`username_unclaimed` 随机对照、validate_targets 误报专项测试，形成数据质量闭环；
4. 数据与代码解耦分发：默认拉线上清单，站点修复无需用户升级程序即可生效；
5. 分发面广：PyPI/Docker/dnf/Homebrew/Apify Actor，且 CI 自动同步官网列表页。

风险/局限：
1. 合规与伦理风险（如实记录）：该工具可被用于骚扰、人肉搜索、社工库式画像等场景；它只做"用户名存在性"探测，不绕过认证，但大规模自动化请求可能触碰目标站点的反爬与服务条款，使用者的法律与道德责任自负，项目方无法控制下游滥用；
2. 数据腐化是长期负担：481 个第三方站点随时改版，removed-sites.md 记录了已放弃的站点（如 gpodder.net 全量 500）；清单正确性永远滞后于现实，误报/漏报随时间累积；
3. 技术天花板：纯状态码/文本匹配易被反爬演进击穿（WAF 指纹需要持续人工添加）；线程池上限 20，无异步 IO，超大规模并发场景不是设计目标；stem/PySocks 等依赖与现行为脱节；
4. 功能进入稳态：0.16.x 长期小步维护，引擎层大改（如 asyncio 化）未见规划（未确认，基于 release 节奏与代码现状推断）。

## 10. 总结评价

Sherlock 是"以最小代码撬动最大数据网络"的典范工程：约 1600 行 Python 引擎 + 一份社区共建的 481 站点 JSON 清单 + 完整的误报治理闭环（Schema 校验、排除列表、WAF 指纹、专项测试、专项 CI），把一个看似简单的问题做成了可持续七年的数据基础设施。它适合安全从业者做授权渗透测试前的账号测绘、研究者做平台占用名分析，以及任何想学习"数据驱动架构 + requests-futures 并发 + 社区数据治理"的 Python 开发者读源码——sherlock.py 的 `sherlock()` 函数与 sites.py 的加载层是最佳入口。使用层面务必注意：仅对授权对象与合规场景使用，并理解其结果的概率性质而非绝对真相。
