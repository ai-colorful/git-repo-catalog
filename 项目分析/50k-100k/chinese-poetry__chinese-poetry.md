# chinese-poetry/chinese-poetry 项目分析

> 星数 53818（快照 2026-10-24）｜主语言 JavaScript｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：**最全的中华古典文集开源 JSON 数据库**——唐诗、宋诗、宋词、元曲、诗经、楚辞、论语、蒙学等 30 余万首/篇古典诗文，以即取即用的 JSON 文件分发，是中文 AI 写诗与诗词类应用的"标配语料库"。

| 项 | 值 |
|---|---|
| 名称 | chinese-poetry/chinese-poetry |
| 星数 / Forks | 53,818 / 10,882（gh api，2026-11-04） |
| 主语言 | JavaScript（按 GitHub 分类；实际以 JSON 数据为主体，脚本为 Python + Node.js） |
| 许可证 | MIT（`LICENSE`，Copyright (c) 2016 JackeyGao） |
| 创建时间 | 2016-09-02（gh api `created_at`） |
| 最近推送 | 2026-06-17（gh api `pushed_at`，合并 PR #417） |
| 默认分支 | master；仓库体积约 222MB（gh api `size`） |
| 主页 | https://awesome-poetry.top/ |

## 2. 项目是什么，解决什么问题

这是 GitHub 上最知名的中文古典文学数据集仓库。README（`README.md`）自称"最全的中华古典文集数据库"，包含 5.5 万首唐诗、26 万首宋诗、2.1 万首宋词及其他古典文集，覆盖唐宋两朝近 1.4 万古诗人与两宋 1.5 千词人。本次对实际文件的全量统计验证了这些数字基本属实（见第 4 章）。

它解决的核心问题是**"结构化、机器可读的古诗文集获取难"**：纸质文集庞大昂贵，网上零散文本格式各异、爬取受限（README 提到"目标网站有限制，采集过程经常中断超过了一个星期"）。作者把采集来的数据统一清洗为 JSON 格式分发，开发者可以直接 `git clone` 后开始构建诗词查询网站、App、小程序或 AI 生成模型。

数据来源于互联网爬取与整理：`宋词/UpdateCi.py` 是针对搜韵网（qsc.zww.cn）的 requests+BeautifulSoup 爬虫；`rank/server.js` 则用 puppeteer 驱动浏览器向百度、必应、360、谷歌四个搜索引擎批量查询"作者+作品名"的结果数，生成诗词知名度排名数据。原始爬取过程未入库，只有最终数据。

项目定位是**数据仓库而非软件项目**：仓库内几乎没有"产品代码"，全部价值在 2000+ 个 JSON 文件上；脚本、测试、CI 都是为保障数据质量服务的（详见第 5、7 章）。它是中文 NLP/AI 圈的著名语料：README 案例区列出的下游项目包括 char-RNN 写诗（pytorch-poetry-gen）、PaddleNLP ERNIE-GEN 诗词生成、离线全唐诗 Android（QuanTangshi）、诗词小程序等十余个。

## 3. 技术栈与依赖

数据仓库，技术栈极薄（出处：`requirements.txt`、`.github/workflows/test.yml`、`.travis.yml`）：

- **数据本体**：UTF-8 JSON 文件（繁体与简体按文集分片共存，见第 4 章）
- **Python**：仅 `pytest==5.3.2` 一个依赖（`requirements.txt`）；`test_poetry.py` 做全库 JSON 校验；`loader/data_loader.py` 提供语料抽取；`宋词/UpdateCi.py` 用 requests/BeautifulSoup（未列入 requirements）
- **Node.js**：`rank/server.js`（约 870 行）基于 puppeteer 的搜索引擎排名爬虫，代码中注释与分支繁多，属一次性运维脚本
- **CI**：GitHub Actions（push/PR 到 master 时跑 pytest）+ 遗留的 Travis CI 配置（Python 3.7，flake8 只查 E9/F63/F7/F82 致命错误后跑 pytest）
- **杂项**：`_config.yml` 仅设 `theme: jekyll-theme-tactile`（GitHub Pages）；`.github/FUNDING.yml` 指向 patreon: jackeygao

## 4. 目录结构与数据组织

顶层是"一个文集一个目录"的扁平组织，每个目录自带 README 说明数据格式（出处：解压后 `ls` 顶层 + 各目录 README）：

| 目录 | 内容 | 文件/条目统计（实测） | 编码 |
|---|---|---|---|
| `全唐诗/` | 全唐诗 + 全宋诗，`poet.tang.[0-57000].json`、`poet.song.[0-254000].json`，每文件 1000 条 | 633 个 JSON；唐诗 57,607 首、宋诗 254,248 首（144MB） | 繁体 |
| `御定全唐詩/json/` | 康熙御定全唐诗，按卷分文件 `001.json`-`900.json`（对应 900 卷） | 900 个文件，43,103 条 | 繁体 |
| `宋词/` | 全宋词 `ci.song.[0-21000].json`，另有 SQLite 文件 `ci.db`（6.3MB）与爬虫 `UpdateCi.py` | 23 个数据文件，21,053 首 | 简体 |
| `元曲/` | 单文件 `yuanqu.json` | 11,057 首（含 dynasty 字段） | 简体 |
| `五代诗词/` | `huajianji/`（花间集，11 个分卷文件 498 首）+ `nantang/`（南唐二主词 45 首） | 14 个文件 | 简体 |
| `诗经/` | 单文件 `shijing.json`，按 国风/小雅 等章节 | 305 篇 | 简体 |
| `楚辞/` | 单文件 `chuci.json` | 65 篇 | 简体 |
| `论语/` | 单文件 `lunyu.json`，按篇章 | 20 章 | 简体 |
| `四书五经/` | `daxue.json`、`zhongyong.json`、`mengzi.json` 三个分册 | 3 个文件 | 繁体 |
| `蒙学/` | 三字经、百家姓、千字文、弟子规、古文观止、千家诗、声律启蒙、唐诗三百首、幼学琼林、增广贤文等 13 部 | 13 个文件（嵌套 title/content 结构） | 简体 |
| `幽梦影/`、`曹操诗集/`、`纳兰性德/`、`水墨唐诗/` | 单文件文集 | 219 条 / 26 首 / 258 首 / 176 首 | 简体 |
| `strains/` | 诗词平仄标注，与 `全唐诗/poet.*.json` 文件与数组下标一一对应 | 313 个文件，311,861 条（105MB） | — |
| `rank/` | 诗词知名度（4+1 个搜索引擎结果数），同样与本体一一对应 | 336 个文件（66MB），含生成脚本 `server.js` | — |
| `loader/` | `data_loader.py` + `datas.json` 语料抽取配置 | 2 个文件 | — |
| `images/`、`log.log` | README 高频词分析图；**误提交的终端输出日志（垃圾文件）** | — | — |

组织方式评述：**纯数据 monorepo**，无包管理、无版本号、无 release；大文集（唐诗宋诗宋词）按"每 1000 条一个分片文件"切分以便增量提交与按需加载。README 数字与实测吻合（5.5 万唐诗≈57,607；26 万宋诗≈254,248；2.1 万宋词≈21,053；作者合计 3,675+8,934=12,609≈"近 1.4 万"）。需要注意 schema 并不统一（见第 5 章），且`全唐诗/error/` 下另存 313 个与主目录不同 md5 的"问题文件"副本。

## 5. 数据模式与关键实现

**（1）主体数据模式（JSON schema）**——不同文集字段名不一致，是最主要的使用摩擦：

- 全唐诗/宋诗（`全唐诗/poet.tang.0.json`）：`{author, paragraphs[], title, note[], id(UUID)}`，四字段数组式分行；
- 宋词（`宋词/ci.song.0.json`）：`{author, paragraphs[], rhythmic(词牌名)}`；
- 诗经/楚辞（`诗经/shijing.json`、`楚辞/chuci.json`）：`{title, chapter, section, author?, content[]}`，用 `content` 而非 `paragraphs`；
- 元曲（`元曲/yuanqu.json`）：`{author, dynasty:"yuan", paragraphs[], title}`；
- 纳兰性德（`纳兰性德/纳兰性德诗集.json`）：字段名为 `para`；幽梦影为 `{content, comment[]}`；水墨唐诗含 `prologue` 赏析字段。

**（2）"本体 + 附加数据分离"设计**（出处：`strains/README.md`、`rank/README.md`）：平仄标注与知名度排名不混入诗词本体，而是通过"文件一一对应 + 数组下标一一对应"的方式旁挂。例如 `strains/json/poet.song.rang.8000.json` 第 n 条对应 `全唐诗/poet.song.8000.json` 第 n 条，格式 `{strains:["○仄仄，仄仄仄仄。"...], id}`；rank 数据为 `{author, title/rhythmic, baidu, so360, google, bing, bing_en}`。README 明确解释动机："保持诗词数据的纯洁……需要时把附加数据挂上即可"。这是全仓库最有设计感的决策。

**（3）语料抽取器**（出处：`loader/data_loader.py`，74 行）：`PlainDataLoader` 读 `loader/datas.json` 中 12+ 个数据集的注册配置（路径、tag 字段名、excludes 排除表），把任意文集抽取为纯文本段落数组，支持单文件与目录两种形态——典型用途是给 char-RNN/大模型拼接训练语料。

**（4）知名度排名爬虫**（出处：`rank/server.js`）：puppeteer 开 5 个浏览器页并行查询百度/360/谷歌/必应（中英版），正则抽取"约 N 个结果"，带随机延迟、缓存续跑（`s.*.cache`）与零结果修复重查（`runLessNumberTask`），最后 `mergeCi/mergePoet` 四源合并。属一次性重运维脚本，硬编码绕反爬逻辑，随时可能因搜索引擎改版失效。

## 6. 数据获取与使用

无可构建产物，使用方式即"clone 后读 JSON"（出处：`README.md` 及各文集 README）：

```bash
git clone https://github.com/chinese-poetry/chinese-poetry.git
# 或只取数据子目录的浅克隆；Python 中 json.load 即用
python -c "import json; print(json.load(open('全唐诗/poet.tang.0.json', encoding='utf-8'))[0])"
# 语料抽取（训练用）
python loader/data_loader.py
# 数据完整性校验
pip install pytest==5.3.2 && pytest
```

各文集 README 给出权威字段说明（如 `全唐诗/README.md` 的 poet.tang/poet.song 样例、`四书五经/README.md` 声明"繁体中文分发"）。使用注意：全唐诗/全宋诗/四书五经为**繁体**，宋词/元曲/蒙学等为**简体**，需自行转换（README 提醒"转换后的字不符合上下文"）。

## 7. 数据质量与工程保障（测试与工程质量）

- **测试即数据校验**（出处：`test_poetry.py`）：扫描根目录下所有含中文名的文集目录，用 `functools.partial` 动态生成 `test_<目录名>` pytest 用例，对每个 JSON 文件做 `json.loads` 解析断言——保证 PR 改坏任何一个数据文件都会被 CI 拦截。简单粗暴但对纯数据仓库非常合适。
- **CI**（出处：`.github/workflows/test.yml`、`.travis.yml`）：Actions 在 push/PR 时跑 pytest；Travis 时代配置还留有 flake8 致命错误检查，双 CI 并存属于历史包袱。
- **质量机制**：贡献规范在 wiki（README 链接），要求纠错 PR 标明出处；争议数据需社区投票；`全唐诗/error/` 保存 313 个待修正的旧版文件。
- **工程质量观察**：无打包/无版本发布，数据只能整仓获取；schema 不统一（paragraphs/content/para 三种正文字段名并存）；`log.log` 误提交、`宋词/ci.db`（6.3MB SQLite）入 git；`rank/server.js` 中大量注释掉的分支。作为"代码仓库"质量很低，但作为"数据仓库"校验闭环是完整的。

## 8. 社区与维护状态

- **活跃度**：最近 5 次提交（gh api）为 2026-06-17（合并 PR #417 loader 修复）、2026-05-06 与 2026-04-28（README 更新）、2025-04-02（PR #389）。近年节奏为**每年个位数提交的低频维护**，主体是社区 PR 数据纠错与 README 维护；140 个 open issue（gh api）。
- **贡献者**：gh api contributors 分页推算约 72+ 人；README 有 OpenCollective 贡献者墙。
- **资金与作者**：作者 JackeyGao（高骏岐），接受 Patreon/爱发电赞助（`README.md`、`.github/FUNDING.yml`），当前"赞助者：无"。
- **发布节奏**：无 release/tag；数据更新以 commit 为粒度，无版本化（未确认是否存在分支级数据版本管理）。

## 9. 亮点与风险

**亮点**

1. **规模与覆盖无可替代**：33 万+ 条诗文实测入库，唐诗宋词元曲经史蒙学一仓打尽，是中文古典 NLP 事实上的标准语料（`全唐诗/`、`宋词/` 等实测统计）。
2. **JSON 即取即用**：MIT 协议 + 纯数据分发，零依赖门槛，clone 即可喂给任何语言/模型（`README.md`）。
3. **附加数据旁挂设计**：平仄（strains/）与知名度（rank/）与本体按下标一一对应而不污染正文，兼顾"纯洁性"与可扩展性（`strains/README.md`、`rank/README.md`）。
4. **CI 数据质量闭环**：动态生成 pytest 用例校验全库 JSON 可解析，PR 改坏数据即 CI 失败（`test_poetry.py`、`.github/workflows/test.yml`）。
5. **下游生态繁荣**：README 收录十余个案例（QuanTangshi 安卓离线全唐诗、PaddleNLP ERNIE-GEN 生成、诗词小程序等），星数 5 年内从 2 万级涨到 5.4 万。

**风险/局限**

1. **数据质量参差**：README 自认"还有大量错误需要更改"；繁简混杂、schema 三种正文字段并存，跨文集消费需写适配层（第 4、5 章实测）。
2. **来源与准确性无底本保障**：数据爬自互联网，未标注版本依据与校勘来源，学术引用需自行核对（`README.md` "古诗采集没有记录过程"）。
3. **维护低频且不可再生**：近年每年几次提交，爬虫脚本（`rank/server.js`、`宋词/UpdateCi.py`）依赖的目标网站/搜索引擎早已可能改版，数据再生产能力存疑。
4. **仓库卫生差**：误提交的 `log.log`、6.3MB 的 `ci.db` 混在 git 里；无版本化发布，增量消费只能靠 git diff。

## 10. 总结评价

chinese-poetry 是典型的"数据即产品"型仓库：代码几乎不存在（校验脚本 58 行、加载器 74 行），但凭借 33 万+ 条结构化诗文、MIT 协议和极低的使用门槛，成为中文诗词类应用与 AI 写诗研究的公共基础设施，53k 星是社区对"数据资产"价值的直接投票。它的工程短板（schema 不统一、繁简混杂、维护低频、无版本发布）对研究型使用者影响有限，对生产型使用者则需要自建清洗层。适合三类人：给 LLM/char-RNN 做古诗生成与微调的算法工程师、做诗词查询产品（网站/小程序/App）的开发者、以及做古典文学数据统计的研究者；想读"源码"的人只会读到爬虫与校验脚本，真正的"源码"是那 2000 多个 JSON 文件本身。
