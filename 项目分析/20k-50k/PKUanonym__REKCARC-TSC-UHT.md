# PKUanonym/REKCARC-TSC-UHT 项目分析

> 星数 37950（快照 2026-10-24）｜主语言 HTML｜档位 20k-50k｜分析日期 2026-11-24

> 说明：本仓库为内容型资料集（无实际软件产品）。tarball 体积 4.85 GB，按规范采用"全量文件清单 + 抽取全部非二进制文件（7,743 个）"的降采样方式分析，PDF/图片/音视频未逐个阅读。以下所有结论均注明出处文件路径（相对仓库根目录）。

## 1. 项目速览

一句话定位：清华大学计算机系（DCST）学生自发维护的课程攻略资料库，按"学期 → 课程 → readme/hw/exam"三级目录汇总选课经验、历年试题、作业与复习资料，并通过 Sphinx 自动生成可浏览的文档站。

| 项目 | 事实 | 出处 |
| --- | --- | --- |
| 名称 | 清华大学计算机系课程攻略（仓库名 REKCARC-TSC-UHT 为 "THU-CST-CRACKER" 倒写） | `README.md`、`source/conf.py`（project = 'THU-CST-Cracker'） |
| 星数 | 37,950（快照 2026-10-24） | 任务参数 |
| 主语言 | HTML（GitHub 自动识别；实为 603 个 Django 模板文件所致，见第 4 章） | 文件清单统计 |
| 许可证 | 贡献者编写部分为 CC BY-SA 4.0；其余版权归各自作者 | `LICENSE`、`README.md`"许可"节 |
| 默认分支 | master（`source/gen.py` 中 PREFIX 硬编码 `/blob/master/`） | `source/gen.py:9` |
| HEAD 快照时间 | 文件 mtime 统一为 2026-11-16（tarball 保留提交时间） | 解压后文件 mtime |
| 体积 | 压缩 4.85 GB，解压约 6.06 GB；14,634 个文件 / 2,047 个目录 | tarball 清单 `listing.txt` |
| CI / 测试 / Release | 均无：仓库不含 `.github/`、无测试目录、无 CHANGELOG | 文件清单 |

## 2. 项目是什么，解决什么问题

这是一个面向清华计算机系本科（兼顾少量研究生课）的"课程经验与资料档案"。每门课程一个目录，目录内的 `readme.md` 是学长留下的课程评价、备考建议和外链，`hw/` 放往年作业与答案，`exam/` 放历年试卷和复习资料，`a4paper/` 放开卷考试用 A4 纸。这一约定在 `收录内容.md` 中被明确定义："课程攻略：readme.md / 平时作业答案与大作业资料：hw/[学生年级_学生姓名]/ / 历年试卷和复习资料：exam/ / 开卷考试 A4 纸：a4paper/"。

项目有明确的传承关系。`README.md`"前言"写明"继承自 Trinkle23897 的项目（thu-cst-cracker），删去了其中侵权内容"；首段又写"受浙江大学相关项目（QSCTech/zju-icicles）启发，创立了本项目"。因此它是 zju-icicles 模式在清华的落地版，并以匿名账号 PKUanonym 作为组织主体。

内容风格高度口语化、实战导向。例如 `大三上/编译原理/readme.md` 开篇即"作业就写PA，书面作业抄往年答案之后自己脑补脑补也会做了"，随后记录了一段考前 24 小时的真实复习流水；`大三下/readme.md` 以"【操作系统】努力学操统……考前一定要复习piazza（血的教训）——by赖涵光学长"的格式逐课点评。这类"过来人经验"是 README 中反复强调的核心价值："您的作品一旦被合并至该仓库中，有可能会被无数学弟仰慕"（`收录内容.md` 末段）。

项目对版权与学术纪律有清醒认识：`收录内容.md` 注明电子版课件"由于可能存在的版权问题，暂未加入"；`.gitignore` 直接忽略 `**/ebook` 与 `**/bbs`；`README.md` 附录全文转载了《清华大学学生纪律处分管理规定实施细则》第六章，提醒使用者作业抄袭、篡改实验数据会受处分。

## 3. 技术栈与依赖

仓库本身几乎不含"业务代码"，唯一的工程部分是一个 ~180 行的 Python 文档站生成器。

| 组件 | 内容 | 出处 |
| --- | --- | --- |
| Python 3 + setuptools | `setup.py` 定义包名 `thu-cst-cracker`，安装时先调用 `source/gen.py --root . --output source` 生成 Markdown 目录树 | `setup.py` |
| Sphinx | 文档构建器，`master_doc = 'index'`，`language = 'zh'` | `source/conf.py` |
| recommonmark | 让 Sphinx 渲染仓库内的 Markdown（已被 Sphinx 官方弃用，仍在此使用） | `source/conf.py` extensions |
| sphinx_markdown_tables | 渲染 `收录内容.md` 等处的 Markdown 表格 | `source/conf.py`、`setup.py` install_requires |
| sphinx_rtd_theme | Read the Docs 主题 | `source/conf.py` |
| Makefile / make.bat | 标准 Sphinx 快速构建脚本（`make html`） | `Makefile`、`make.bat` |
| 托管 | Read the Docs（README 徽章与 `rekcarc-tsc-uht.readthedocs.io` 链接） | `README.md` |

内容文件本身涉及的"语言"则非常杂：Java（689，多为编译原理 Decaf 框架与 OJ 作业）、C（525）、C++（413）、Verilog（346，数字逻辑/计算机组成原理）、Python（242）、Django HTML 模板（603）、以及大量 PDF/图片/Word/PPT（见第 4 章）。这些是被收录的学生作业，而非项目代码。

## 4. 目录结构与内容组织

顶层结构极简：4 个说明文件 + 3 个构建文件 + `source/` + 11 个按学期命名的中文目录。

| 目录 / 文件 | 职责 | 文件数 |
| --- | --- | --- |
| `README.md` | 项目介绍、致谢名单（35 位具名贡献者）、许可、校规附录 | 1 |
| `收录内容.md` | 收录范围说明 + 按学期的课程清单表格 | 1 |
| `贡献方法.md` | Web 端上传 + PR 的贡献流程 | 1 |
| `参考书目.md` | 252 行，按学期/课程列出参考教材 | 1 |
| `source/` | `gen.py`（目录树生成器）、`conf.py`（Sphinx 配置） | 2 |
| `大一上` ~ `大四上`、各"小学期" | 按学期分桶的课程资料 | 见下表 |
| `研究生` | 研究生课程资料 | 80 |

各学期规模（来自 tarball 清单统计）：

| 学期 | 课程目录数 | 文件数 | 说明 |
| --- | --- | --- | --- |
| 大一上 | 9 | 646 | 微积分、线代、离散、程序设计基础等 |
| 大一下 | 8 | 895 | 面向对象按两位老师分目录 |
| 大一小学期 | 5 | 3,406 | 其中 `Week_3-Python` 独占 3,000+，为 Django 作业工程 |
| 大二上 | 9 | 1,122 | 数据结构、计算机系统概论、形式语言等 |
| 大二下 | 12 | 1,063 | 人工智能导论、数字逻辑、随机数学等 |
| 大二小学期 | 0 | 1 | 仅一个 `量子计算研讨.md` |
| 大三上 | 11 | 3,613 | 编译原理、计组、网络、软工、人机交互等 |
| 大三下 | 18 | 3,164 | 操作系统、体系结构、机器学习等，课程最多 |
| 大三小学期 | 2 | 580 | 保研考试、并行计算 |
| 大四上 | 5 | 53 | 嵌入式、网络安全工程等 |
| 研究生 | 5 | 80 | 最优化方法、应用随机过程等 |

合计 83 个课程级目录。课程目录内部的子目录高度模式化：`hw` 出现 43 次、`exam` 34 次，其余为少量自命名目录（`notes`、`docs`、`a4paper`、"2022-李鹏辉-exam"等），说明约定虽无强制校验但被绝大多数贡献者遵守。

文件类型分布（按数量）：pdf 1,884（2.2 GB，占体积最大头）、jpg 1,556、png 1,502、txt 1,076、java 689、html 622、c 525、h 521、cpp 413、docx 343、md 285、pptx 71、mp4 7。**"主语言 HTML" 完全是误判**：622 个 html 中 603 个位于 `大一小学期/Week_3-Python/hw/*/` 下的 Django 项目 `templates/` 目录（如 `.../douban/search/templates/search/index.html`），同样 750 个 txt 也来自该目录（爬虫数据集）。

路径中的年份标记覆盖 2009–2026，峰值在 2015–2017（课程作业目录按年份分组），2023–2026 仍有 146 个路径命中，说明近年持续有新资料进入。

## 5. 核心实现：目录树生成器 `source/gen.py`

这是仓库唯一的"代码"，106 行，职责是把文件系统目录结构转换成带 GitHub 链接的 Markdown 导航页，再交给 Sphinx 渲染。

关键设计点：

1. **递归扫描 + 链接回 GitHub**。`get_course()` 对每个目录递归，生成 `- [名称](url)` 列表，URL 由 `url_filter()` 拼接：目录用 `tree`、文件用 `blob`，并对 `(`、`)`、空格做百分号编码（第 30-32 行）。这意味着文档站本身不托管文件，点击即跳回 GitHub，文档站只是"目录页"。

```python
PREFIX = 'https://github.com/PKUanonym/REKCARC-TSC-UHT/blob/master/'
def url_filter(s, p):
    c = 'tree' if os.path.isdir(s) else 'blob'
    return PREFIX.replace('blob', c) + s.replace(p, ...).replace('(', '%28').replace(')', '%29').replace(' ', '%20')
```

2. **大目录折叠**。第 48-50 行：`if depth >= 2 and d_num > threshold:`（threshold=30），子目录文件数超过 30 时不再展开，只保留一个目录链接。这是针对 `Week_3-Python` 这类数千文件的工程目录的保护，避免生成的 Markdown 过长。

3. **readme 内联**。课程目录下若有 `readme.md`，其正文会被读取并原样嵌入生成的页面（第 58-61 行）；学期根目录的 `.md` 文件（如 `大三下/readme.md`）同样被拼接（`get_semester()` 第 68-70 行）。

4. **一处可疑的字符替换**。第 105 行：`rst = rst.replace('二', '无').replace('三', '二').replace('无', '三')`，对 `index.rst` 的 toctree 条目名做"二↔三"互换，导致首页目录中"大二上"显示为"大三上"（反之亦然）。意图未确认，可能是早期排序 hack 的遗留，也可能是刻意为之。

5. **工程健壮性一般**：`setup.py` 用 `subprocess.check_call` 在安装阶段改写 `source/`；`.gitignore` 同时忽略 `source/_static`、`*.pyc`；`conf.py` 中 `recommonmark` 已不被 Sphinx 官方推荐（`recommonmark` 项目已归档），但仓库未迁移到 `myst-parser`。

## 6. 构建与运行

来自 `Makefile`、`make.bat`、`setup.py`：

```bash
pip install sphinx recommonmark sphinx-markdown-tables sphinx_rtd_theme
python3 ./source/gen.py --root . --output source   # 或 pip install . 触发同一命令
make html                                          # Windows: make.bat html
# 产物在 build/html，线上版本见 README 中的 readthedocs 链接
```

注意 `gen.py` 必须在仓库根目录运行（`--root .`），且依赖 `master` 分支 URL，在 fork 上生成的链接会指回原仓库。

## 7. 更新机制与质量控制

- **无 CI、无测试、无自动校验**：清单中不存在 `.github/`，没有任何 workflow；`gen.py` 仅在本地构建时运行。
- **纯人工 PR 流程**：`贡献方法.md` 给出的是面向非开发者的"GitHub Web 端上传 → Pull Request"指引，甚至教用户"把浏览器地址栏中文件夹名称改为你想要新建的文件夹名称"；也接受直接在 Issue 附文件由维护者代为添加，或发邮件到 `PKUanonymous@qq.com` 匿名贡献。
- **软性约定而非校验**：作业目录建议命名 `[年级]_[姓名]`（示例 `2050_anonym`），但实际目录中混有 `zhaochen20`、`2018_周恩贤`、`2017/2014011330_738787_395404561_lab1/` 等多种风格，说明从未统一。
- **体积治理靠口头约束**：README 强调"未启用 git-lfs，请不要上传单个超过 100M 的文件"；但仓库仍累计 4.85 GB，且混入了明显的构建产物（如 `大三上/人机交互理论与技术/hw/2016/大作业/.../New Unity Project 1/Library/metadata/` 下的 Unity 元数据文件），未见清理机制。
- **版权治理**：`.gitignore` 排除 `**/ebook`、`**/bbs`；README"前言"声明已删除前身仓库的侵权内容并欢迎通过 Issue 指出。

## 8. 生态位与社区状态

**姊妹项目**（均来自 README 原文，其他同类高校项目未确认）：前身 `Trinkle23897/thu-cst-cracker`；灵感来源 `QSCTech/zju-icicles`；校内镜像 `git.tsinghua.edu.cn/pkuanonym/REKCARC-TSC-UHT`（供清华学生节省流量）；文档站 `rekcarc-tsc-uht.readthedocs.io`。课程 readme 中大量外链指向贡献者个人仓库（如 `Trinkle23897/os2019`、`FztTony/ucore2021`），形成"中心索引 + 分散个人仓库"的生态。

**贡献者结构**：README 具名致谢 35 人；各课程 `hw/` 下共有约 98 个一级子目录（按年份或学生姓名命名）。维护者 PKUanonym 为匿名账号，README 中保留了个人署名（"by 赖涵光学长"、"by 李成江学长"）。

**活跃度**：无 git 历史可查，只能推断——HEAD mtime 2026-11-16；路径中含 2026 年标记的条目 11 个；`大一小学期/2023-python课堂`、`2024级HW` 等目录名表明近年仍在收录。Release 节奏、提交频率、贡献者总数：未确认。

## 9. 为什么能涨到 37,950 星（分析性推断）

基于仓库内容而非外部数据，可归因于四点：

1. **刚性需求 + 稀缺供给**：历年真题、作业参考与"哪位老师给分好"的经验，在校园内部通常靠熟人传递，公开仓库把这种隐性与稀缺信息变成了可检索的公共品（`收录内容.md` 的课程表直接对应培养方案）。
2. **极低的贡献门槛**：Web 上传 + PR，甚至 Issue 附件和邮件代传（`贡献方法.md`），让非开发者也能贡献；成果"被无数学弟仰慕"的激励写进了文案。
3. **清晰的组织与可浏览性**：学期 → 课程的目录命名对目标用户零学习成本；`gen.py` + Read the Docs 让不装 git 的人也能在线翻阅。
4. **正确定位品牌与传承**：承接 zju-icicles 的全国高校"课程攻略"风潮与 thu-cst-cracker 的存量资料，并主动处理版权（删侵权内容、排除 ebook），使项目在敏感地带上能长期存活。

## 10. 亮点、风险与总结

**亮点**

- 内容规模与时间跨度罕见：83 门课、14,634 个文件、资料年份跨 2009–2026，几乎覆盖清华计算机系本科全部核心课程。
- 约定简明且被广泛遵守（`readme.md` / `hw` / `exam` / `a4paper`），目录即可导航。
- 法律与伦理意识到位：CC BY-SA 双层许可、排除电子教材、附校规警示、提供匿名贡献渠道。
- 生成器虽小但务实：自动回链 GitHub、大目录折叠、readme 内联。

**风险 / 局限**

- 仓库体积失控：4.85 GB 的 tarball 让 clone、fork 和备份都很沉重，且已混入 Unity `Library/` 等无关构建产物；README 的 100 MB 口头限制无法阻止总量膨胀。
- 零自动化：无 CI 校验目录规范、文件大小、敏感内容；`gen.py` 的"二↔三"替换等问题无人发现。
- 技术栈陈旧：`recommonmark` 已归档，`setup.py` 在安装期写文件属于反模式。
- 内容适用性随课程改革快速衰减（`编译原理/readme.md` 自述"2019 Update：框架有了比较大的更新，往届资源作用可能比较小"），但仓库没有标注资料过期状态的机制。

**总结**

这是一个"用 Git 做知识归档"而非"做软件"的项目，其价值在内容与社区共识，技术层面只是薄薄一层 Sphinx 封装。适合清华计算机系学生（选课、备考、找参考）、其他高校想复制该模式的人（可参考 `收录内容.md` + `贡献方法.md` + `source/gen.py` 三件套），以及研究开源社区中非代码型协作的研究者。不必期待从中读到优雅代码，但 `readme.md` 里的过来人经验是真正稀缺的东西。
