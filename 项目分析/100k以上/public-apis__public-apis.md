# public-apis/public-apis 项目分析

> 星数 491589（快照 2026-10-24）｜主语言 Python｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：全球最著名的免费公共 API 目录，GitHub 历史上星数最高的内容型仓库之一——本体是一份 328KB、两千余条目的社区人工策展清单，配一个小型 Python 校验工具链保证清单质量。

| 项目 | 内容 |
| --- | --- |
| 名称 | public-apis/public-apis |
| 星数 | 491,589（快照 2026-10-24；查询日 API 已达 493k+） |
| 主语言 | Python（仅指脚本工具；仓库主体是 Markdown 清单） |
| 许可证 | MIT（Copyright (c) 2022 public-apis，见 `LICENSE`） |
| 默认分支 | master，最近 push 2026-10-29（gh api 元数据） |
| 最近活跃 | 极活跃：2026-10-29 单日合并 10 个 PR（gh api commits） |
| 贡献者 | 约 2138 人（gh api contributors 分页头） |
| 历史作者 | davemachado（README"API for this project"链接指向 `davemachado/public-api`） |

## 2. 项目是什么，解决什么问题

开发者做 side project 或原型时，最常见的需求之一是"找一个能免费调用的数据 API"：天气、汇率、政府开放数据、游戏行情……这些 API 散落在各家官网，质量参差、认证方式各异，逐个试错成本很高。public-apis 把这些 API 收拢为一张按领域分类的大表，每个条目统一标注五个字段：API 名称（链接到文档）、一句描述、认证方式（No/apiKey/OAuth 等）、是否支持 HTTPS、是否支持 CORS——后两者直接决定了"能不能在浏览器前端里直接用"，这是同类清单里少见的实用标注。条目格式定义见 `CONTRIBUTING.md` 的 Formatting 一节。

它解决的核心问题是"发现与初筛"：不替你调用 API，而是告诉你有哪些选择、要不要注册、能不能跨域调用。描述限制在 100 字符以内（`CONTRIBUTING.md` PR 规则），因此整份清单保持了"索引"而非"教程"的密度，适合 Ctrl+F 式查阅。

收录范围的取舍写在 `CONTRIBUTING.md` 开篇：必须是"完全免费或至少含免费层"的 API，且不依赖购买某个硬件/服务才能使用（文中举的反例是"API 免费但需先买智能插座"）；明确声明"本清单不是营销工具"，纯商业推广类 PR 一律拒绝。MCP 条目则采用独立格式（Name/Description/Auth/Transport/Install 五列），因为 MCP server 是"装进客户端"而非 HTTP 调用，HTTPS/CORS 字段不适用（`CONTRIBUTING.md` MCP Servers 节）。

仓库现状（以 2026-10 下载的 master 快照为准）与早期版本已有明显演化：顶部新增了赞助方 APILayer 的商业 API 专区（11 条，`README.md` "APILayer APIs" 节），新增了面向 AI Agent 时代的 "MCP Servers" 专区（26 条，列出 transport 与 Cursor/Anthropic/Glama 等 marketplace 安装入口），并给出官方配套网站 publicapis.lol（搜索、过滤、存活状态监测）。历史上的机器可读索引 `APIs.json` 在当前 master 文件树中已不存在（经 gh api 全量文件树核实），README 是唯一的数据源。

需要说明的边界：这是内容型仓库（清单 + 少量工具脚本），不是可运行的软件。README 占仓库全部文本量的 90% 以上（328KB / 全仓库约 355KB），Python 代码只服务于"保证清单格式正确、链接存活"这一件事。

## 3. 技术栈与依赖

- 清单本体：纯 Markdown（GitHub Flavored 表格），无任何静态站点生成器或数据文件依赖，GitHub 仓库页面本身就是交付界面。
- 校验脚本：Python 3（CI 用 3.8，`scripts/README.md`），依赖极简——`scripts/requirements.txt` 仅 5 行，核心是 `requests>=2.32.4`，其余为 certifi 2021.10.8、charset-normalizer 2.0.10、idna 3.3、urllib3 1.26.8 的历史版本 pin（明显多年未升级）。
- 依赖管理观察：无 lockfile、无虚拟环境/打包配置，核心依赖 requests 用 `>=` 下限而传递依赖用旧版 `==` 精确 pin，两种策略混用且整体陈旧，属于"能跑就不再动"的运维风格。
- 测试：标准库 `unittest`，无 pytest 等第三方框架。
- CI：GitHub Actions，3 个 workflow（见第 8 章），action 版本偏旧（`actions/checkout@v2`、`actions/setup-python@v2`）。
- 无 package.json/pyproject 等工程清单，无构建产物；`.gitignore` 是一份通用 Python 模板（`.gitignore`）。

## 4. 目录结构与代码组织

| 路径 | 职责 |
| --- | --- |
| `README.md` | 产品本体：赞助区 + MCP 专区 + 导览 + Index + 52 个分类表（328KB） |
| `CONTRIBUTING.md` | 条目格式规范（API 表 5 列、MCP 表 5 列）与 PR 规则（一 PR 一条、按字母序、squash 提交等） |
| `scripts/validate/format.py` | README 格式校验器（8.5KB） |
| `scripts/validate/links.py` | 链接存活/重复校验器（8.1KB） |
| `scripts/tests/` | 上述两个模块的单元测试（639 行） |
| `scripts/github_pull_request.sh` | 只校验 PR diff 新增行的链接检查脚本 |
| `scripts/requirements.txt`、`scripts/README.md` | 依赖与脚本使用说明 |
| `.github/workflows/` | 3 个 CI workflow（格式校验/链接日检/单元测试） |
| `.github/ISSUE_TEMPLATE.md` 等 | 引导新增条目走 PR 的模板 |

组织方式评述：单文件单目录的极简结构——一个 README 承载全部内容，一个 `scripts/` 包承载全部质量保障逻辑，无构建、无发布、无 release（gh api 显示 releases 为 0）。一个有趣的细节：`.gitattributes` 声明了 `README.md export-ignore` 与 `/.github export-ignore`（且模式无前导斜杠，连 `scripts/README.md` 一并被排除），导致 GitHub 的 codeload tarball 里没有 README——下载该仓库源码包时会"看不见"主体内容，需单独获取。

## 5. 核心架构与关键实现

本章按内容型仓库视角，分析"清单如何组织"与"质量如何保障"两件事。

### 5.1 清单组织：Index + 分类表的两级结构

`README.md` 从第 177 行起是 52 个 `###` 分类节（Animals 到 Weather），每节一张五列表；第 122 行的 `## Index` 是锚点目录。`scripts/validate/format.py` 的 `check_file_format()`（第 192-251 行）强制两者一致：每个 `###` 标题必须在 Index 中登记，反之索引标题也要有对应分类节；每个分类至少 3 个条目（`min_entries_per_category = 3`）。条目在分类内必须按字母序排列——`check_alphabetical_order()` 把每个分类的标题列表与排序结果比对，不一致即报错并给出行号（`(L{line:03d})` 格式）。这套约束让 2252 个条目在纯人工维护下保持结构不塌。

### 5.2 format.py：把贡献规范编译成校验器

`CONTRIBUTING.md` 的每条格式规则在 `scripts/validate/format.py` 里都有对应检查函数：`check_title()` 要求标题为 `[TITLE](LINK)` 语法且不以 "... API" 结尾；`check_description()` 要求首字母大写、不以标点结尾、不超过 100 字符（第 107-126 行）；`check_auth()` 要求认证值必须是 `apiKey/OAuth/X-Mashape-Key/User-Agent/No` 之一且用反引号包裹；`check_https()/check_cors()` 是枚举校验。甚至表格空格风格（"每段前后各一个空格"）也被逐行检查。所有错误一次性收集输出并 `sys.exit(1)`，作为 CI 门禁。其中最有意思的是"标题不得以 API 结尾"这条——它把英文社区最常见的命名冗余直接禁掉了：

```python
# scripts/validate/format.py check_title()（第 97-102 行）
title = title_match.group(1)
if title.upper().endswith(' API'):
    err_msg = error_message(
        line_num,
        'Title should not end with "... API". Every entry is an API here!'
    )
    err_msgs.append(err_msg)
```

### 5.3 links.py：对抗链接腐坏的存活检测

2252 个外部链接的最大敌人是链接腐坏。`scripts/validate/links.py` 的实现有三个值得注意的设计：一是链接提取正则直接内联在 `find_links_in_text()`（第 12-23 行），从 `## Index` 之后开始扫描，跳过赞助区；二是 `has_cloudflare_protection()`（第 96-150 行）专门识别 Cloudflare 403/503 盾页的 18 个特征串（如 "Ray ID:"、"_cf_chl"），避免把"被盾挡了"误报为死链——这是大规模链接检查的真实痛点，注释里还引用了当年讨论此问题的 PR #2409 和 issue #2960；三是 `fake_user_agent()` 轮换四种浏览器 UA 并附带 host 头，绕过基础的反爬拦截。单线程逐个 GET、25 秒超时，全量检查一次约需数小时，因此放在每日定时任务而非每个 PR 里跑。

### 5.4 PR 增量校验

`scripts/github_pull_request.sh` 拉取 `patch-diff.githubusercontent.com` 上的 PR diff，用 `egrep "\+"` 抽出新增行，只对新增内容跑链接校验——让每个 PR 的 CI 在秒级完成，与全量日检形成快慢两层。

## 6. 构建与运行

以下命令均出自 `scripts/README.md`（仓库无构建环节，"运行"即跑校验）：

```bash
# 安装校验脚本依赖
python -m pip install -r scripts/requirements.txt

# README 格式校验（CI 门禁，见 .github/workflows/test_of_push_and_pull.yml）
python scripts/validate/format.py README.md

# 链接全量存活检查（慢，每日 cron 执行）
python scripts/validate/links.py README.md

# 只查重复链接（秒级）
python scripts/validate/links.py README.md -odlc

# 运行校验脚本自身的单元测试
cd scripts && python -m unittest discover tests/ --verbose
```

对普通使用者而言，"运行"这个项目只需打开 README 或其配套站 publicapis.lol 检索。

## 7. 收录范围、分类体系与条目规模

- 总量：52 个分类、2252 个社区条目（`README.md` 中以 `| [` 开头的表格行统计），另有 APILayer 赞助专区 11 条、MCP Servers 专区 26 条；全文件外部 URL 共 2315 个、涉及约 2024 个不同域名（本地脚本统计）。
- 分类体系：扁平单级，无子分类。最大类 Development 223 条、Government 128、Geocoding 113、Games & Comics 112、Cryptocurrency/Finance 各 98；最小类 Patent 仅 4 条、Programming 5、Events 5（分类内逐行统计）。类目粒度不均，但"至少 3 条才能开新类"的规则（format.py）抑制了碎片化。
- 字段画像：条目 Auth 字段分布约为 No 1078、apiKey 998、OAuth 155、X-Mashape-Key 6、User-Agent 3（表格第三列解析，近似值）——近半数条目无需注册即可调用，这是该清单"免费"成色的直接量化。
- 收录门槛：`CONTRIBUTING.md` 开篇明确拒绝纯营销型 PR，要求"完全免费或至少有免费层，且不依赖购买某硬件/服务"；已收录 API 的新版本不许另立条目，旧版条目会被清理。
- 内容质量短板：部分描述过于简略（如 Weather 类目中多条仅写"Weather"，`README.md` 2701 行起），CORS 列大量为 Unknown，条目元数据更新依赖人工。
- README 版面结构（行号供检索）：1-21 赞助横幅；42 "APILayer APIs" 专区；60 "MCP Servers" 专区；97 "Learn more"（官方站与贡献入口）；122 "## Index"；177-2756 为 52 个分类节；2758 "## License"。

## 8. 更新机制与社区维护

- 更新流程：完全由社区 PR 驱动，规则严格——一 PR 只加一条、标题格式 `Add X API`、commit 信息要具体（拒绝 "Update Readme.md"）、必须 squash 成单提交、目标分支 master、提交前需搜索既有 issue/PR 避免重复（`CONTRIBUTING.md` PR Guidelines）。PR 模板（`.github/PULL_REQUEST_TEMPLATE.md`）是 9 项自查清单，与 format.py 的检查项一一对应。
- 自动化质量门禁（`.github/workflows/`）：`test_of_push_and_pull.yml` 对每个 PR 跑格式校验 + diff 增量链接检查，对 push 跑重复链接检查；`validate_links.yml` 每日 00:00 UTC 全量检查所有链接存活；`test_of_validate_package.yml` 跑脚本自身单元测试。
- 活跃度（gh api，2026-10-29 查询）：最近 push 同日；最近 10 次提交全部是合并 PR（当日 10 个），合并者账号均为 "commit10"，呈单一维护账号批量合并的模式；累计合并 PR 编号已到 #8239。
- 维护压力：open issues 2471、open PRs 2412，积压规模巨大；releases 为 0（对内容型仓库属正常）。贡献者约 2138 人，是典型的"海量一次性贡献者 + 极少审校者"漏斗结构。
- 商业化现状：APILayer 是仓库维护方/赞助方（README 顶部横幅、专区和全站 utm 追踪参数），配套官网 publicapis.lol 提供搜索、过滤与 API 存活状态页，形成"GitHub 清单 + 商业站点"的双载体。

## 9. 亮点与风险

亮点：

1. 星数 Top 级的"开发者入口"价值：近半条目免认证、五字段标注（Auth/HTTPS/CORS）让初筛成本极低，README 即产品，零依赖零构建。
2. 质量保障工程化程度远超同类 awesome 清单：格式规则全部落为可执行校验（`scripts/validate/format.py`），带 32 个单元测试，PR 门禁 + 每日全量链接检查双层 CI。
3. links.py 对 Cloudflare 盾页的误报治理（`scripts/validate/links.py` 第 96-150 行）是链接检查类工具里少见的实战经验沉淀。
4. 持续进化：2026 年新增 MCP Servers 专区与 AI Agent 视角条目，跟上范式迁移；官方镜像站点承接检索需求。

风险/局限：

1. 维护漏斗堵塞：2400+ open PR、2400+ open issue 依赖单一合并账号消化，单条目审核深度存疑，新条目收录周期不可控。
2. 链接腐坏是持续性消耗：2000+ 外链、描述与 CORS 状态靠人工维护，日检只能事后发现死链，无法自动修复。
3. 商业化与中立性的张力：赞助方专区与全站 utm 参数使"社区清单"带明显导流属性（`README.md` 顶部与 APILayer APIs 节），尽管 CONTRIBUTING 明文拒绝营销 PR。
4. 工具链技术债：Python 3.8、checkout@v2 等旧版本 CI 依赖，requirements.txt 中 certifi/urllib3 等为 2021 年 pin；机器可读的 APIs.json 已移除，程序化消费只剩解析 Markdown 一条路。

## 10. 总结评价

public-apis 是"清单即产品"的极限案例：一份 328KB 的 README 靠严苛的格式规范、双层 CI 和两千余名贡献者的 PR 洪流维持了十年级别的内容质量，星数进入 GitHub 全站历史前列。它证明了内容型仓库的护城河不在代码，而在"收录标准 + 自动化门禁 + 社区惯性"三者构成的质量飞轮；同时它也暴露了此类仓库的宿命——审核人力赶不上贡献洪流，商业化又天然与中立清单存在张力。

它能涨到近 50 万星的逻辑并不神秘：API 是几乎所有开发者的公共刚需，这份清单长期是搜索引擎和"GitHub 上有什么好东西"类内容的首选结果，星标即书签——每个把它当工具书的访客都会顺手 star，累积效应随时间指数放大；而"免费 + 分类全 + 无需登录即可看"的零门槛形态让转发和收藏几乎没有摩擦。

适合谁用：几乎所有开发者都可当 API 发现索引使用（配合 publicapis.lol 的搜索与存活状态页体验更好）。适合谁读"源码"：想研究"如何运营超大规模众包清单"的人，`CONTRIBUTING.md` + `scripts/validate/format.py` + 三份 workflow 组成的最小质量保障体系是最值得精读的部分；对普通工程师而言，其 Python 脚本只是入门级工具代码，专门通读的收益不大。
