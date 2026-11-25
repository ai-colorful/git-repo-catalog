# krahets/hello-algo 项目分析

> 星数 130935（快照 2026-10-24）｜主语言 Java（GitHub linguist 口径）｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：一本开源免费、动画图解、代码可一键运行的数据结构与算法入门书《Hello 算法》，以 mkdocs-material 站点为载体，配套 14 种编程语言的可运行示例。

| 项目 | 内容 |
| --- | --- |
| 名称 | krahets/hello-algo |
| 星数 / Forks | 130935 / 15570（gh api，2026-10-30） |
| 主语言 | Java（GitHub 统计口径；仓库实际为"文档 + 多语言代码"混合体） |
| 许可证 | CC BY-NC-SA 4.0（LICENSE、README.md 结尾） |
| 当前版本 | 1.3.0（mkdocs.yml `version: 1.3.0`；release 1.3.0 发布于 2026-01-01） |
| 最近活跃 | 最近 push 2026-10-29（gh api pushed_at），仓库创建于 2022-11-04 |
| 贡献者 | 约 214 人（contributors API 分页 Link 头 last=214 推算） |

## 2. 项目是什么，解决什么问题

hello-algo 的本质是一本"书"，而不是一个代码库：正文全部以 Markdown 编写（中文主站 docs/ 共 120 个 .md），通过 mkdocs-material 渲染成网站 hello-algo.com，同时提供 PDF/EPUB 下载（README.md 按钮）与纸质书（docs/chapter_paperbook/index.md，顶栏公告也指向纸质书，见 overrides/main.html）。它解决的是"算法入门无从下手"的问题：作者在序言（docs/chapter_hello_algo/index.md）中将盲目刷题比作玩扫雷，本书的目标是先给出一张"知识地图"。

全书共 16 章（mkdocs.yml nav），从复杂度分析、数组链表、栈队列、哈希、树、堆、图，到搜索、排序、分治、回溯、动态规划、贪心，覆盖国内算法面试的完整知识体系；每章配有"小结"与"练习"小节。内容特点是"动画图解"——每个小节配手绘风格插图（如 docs/chapter_heap/heap.assets/min_heap_and_max_heap.png），关键算法还有"可视化运行"链接，指向 pythontutor.com 的单步演示（如 docs/chapter_stack_and_queue/stack.md 内嵌的 URL）。

与一般教程最大的差异是代码工程化：codes/ 目录为 14 种语言（C、C++、C#、Dart、Go、Java、JavaScript、Kotlin、Python、Ruby、Rust、Swift、TypeScript、Zig）提供与章节一一对应的完整可运行示例（如 codes/java/chapter_heap/ 下 heap.java、my_heap.java、top_k.java），并由 14 个 GitHub Actions workflow 保证其始终可编译、可执行。正文中的代码块以 `=== "Python"` 多页签形式内嵌全部语言实现（docs/chapter_heap/heap.md）。

项目同时是一个多语言出版工程：除简体中文主站外，zh-hant、en、ja、ru、ko 五个目录各自维护一套完整镜像（docs + codes + mkdocs.yml + README），近期仍在高强度推进——最近三天提交包括韩语版上线（#2006，2026-10-28）与日语全书重译（#2008，2026-10-29）。

## 3. 技术栈与依赖

- **站点生成**：mkdocs-material 9.5.5 + mkdocs-glightbox（Dockerfile 固定版本 `pip install mkdocs-material==9.5.5 mkdocs-glightbox`），构建于 python:3.10.0-alpine 镜像。
- **Markdown 扩展**：pymdownx 全家桶——tabbed（多语言代码页签）、superfences、arithmatex（数学公式）、snippets、tasklist 等（mkdocs.yml 116-150 行）；MathJax 3.2.2 CDN 渲染公式（mkdocs.yml extra_javascript）。
- **主题定制**：overrides/ 下 main.html、content.html、comments.html、actions.html、i18n.html、404.html、starfield.js（首页星空背景）等 mkdocs-material 模板覆载。
- **评论系统**：giscus（基于 GitHub Discussions），配置在 overrides/partials/comments.html 与 giscus.json。
- **代码示例**：14 种语言的标准库/官方工具链，无第三方依赖；Python 示例仅用 black 做格式化（.github/workflows/python.yml）。
- **构建/部署**：Docker + docker-compose 一键构建全站；CI 为 GitHub Actions（.github/workflows/ 下 14 个语言 workflow）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| docs/ | 简体中文主站正文：16 章 + 附录 + 参考文献 + 纸质书页，含全部插图资产（docs/assets 约 8.6MB） |
| codes/ | 14 种语言的示例代码 + pythontutor/（可视化演示源码），每语言按 chapter_* 与正文对应 |
| en/ ja/ ko/ ru/ zh-hant/ | 五个语言版本完整镜像：各含 README、docs/、codes/、mkdocs.yml（en 另有 CONTRIBUTING.md） |
| overrides/ | mkdocs-material 主题覆载（页脚评论注入、i18n 文案、公告栏、404） |
| mkdocs.yml | 根站点配置（320 行）：主题、插件、扩展、全站 nav |
| Dockerfile / docker-compose.yml | 六语言站点的一次性构建与本地预览 |
| giscus.json | 评论组件的域名白名单与排序配置 |
| .github/workflows/ | 14 个按语言划分的 CI workflow（c/cpp/csharp/dart/dotnet/go/java/javascript/kotlin/python/ruby/rust/swift/typescript/zig） |

组织方式评述：这是典型的"每语言一镜像"的出版型仓库——en/ja/ko/ru/zh-hant 各自是主站 docs/ 与 codes/ 的完整翻译副本（如 zh-hant/codes/python 有 89 个 .py，与主站 codes/python 完全同规模）；工程清单极简，没有 package.json/pyproject 之类的应用工程文件，构建只依赖 mkdocs 与各语言工具链。docs/ 内部按 chapter_* 划分，与 codes/*/chapter_* 形成严格的一一映射，定位任何一节内容只需两个路径。

## 5. 核心架构与关键实现

### 5.1 章节组织：nav 即目录，文档与代码双向映射

mkdocs.yml（163-320 行）用手工 nav 定义全书结构：序 + 第 0 章前言 + 第 1-15 章 + 附录 + 参考文献 + 纸质书；每章 index.md 作章节封面，小节标注 `*` 表示选学（如 `7.5 AVL 树 *`）。正文与代码靠同名目录约定对应：`docs/chapter_heap/heap.md` 的代码块标题 `title="heap.py"` 对应 `codes/python/chapter_heap/heap.py`，Java 侧则为 `codes/java/chapter_heap/heap.java`（文件头注释带 File/Created Time/Author 三元组，codes/java/chapter_heap/my_heap.java）。

### 5.2 多语言示例同步：全量镜像 + CI 矩阵兜底

每个翻译版目录下的 codes/ 是主站代码的全量副本，而非共享引用。同步质量的保障在 CI：以 java.yml 为例，触发路径同时列出 `codes/java`、`en/codes/java`、`ja/codes/java`、`ko/codes/java` 四套目录，matrix 展开为 2 个 JDK 版本 × 4 个代码目录共 8 个组合，逐套执行 `javac -d <dir>/build <dir>/**/*.java` 全量编译。Python 版更进一步：3 操作系统 × 4 代码目录，先 `black` 再跑冒烟脚本。值得注意的是 zh-hant 与 ru 的代码目录未纳入任何 CI 触发路径（java.yml/python.yml 的 paths 列表均只含 zh/en/ja/ko 四套），其正确性依赖翻译时的人工拷贝。

Python 冒烟测试零依赖且直接了当（codes/python/test_all.py）：glob 所有 `chapter_*/*.py`，逐个 `subprocess.Popen` 运行，统计非零退出码并汇总报错——"示例必须能跑"被固化为可执行的验收标准。

主站 codes/ 各语言规模实测（find 统计文件数）：

| 语言 | c | cpp | csharp | dart | go | java | javascript | kotlin | python | ruby | rust | swift | typescript | zig | pythontutor |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 文件数 | 100 | 106 | 92 | 87 | 140 | 88 | 87 | 87 | 90 | 87 | 91 | 91 | 89 | 59 | 67 |

其中 Java 示例共 87 个 .java、约 6635 行，是全仓库单语言体量最大的实现之一；各语言文件数彼此贴近（87-106），说明多语言同步做到了逐文件对齐，而非按语言选择性覆盖。

### 5.3 mkdocs 站点工程：配置继承 + 六语言一站部署

站点工程有三个关键设计：

1. **构建目录与源目录分离**：根 mkdocs.yml 声明 `docs_dir: build/docs`（第 6 行），各语言 mkdocs.yml 指向 `../build/<lang>/docs`，源码目录保持干净，构建产物统一汇入 site/。
2. **mkdocs-material 配置继承**：子站配置只需一行 `INHERIT: ../mkdocs.yml`（en/mkdocs.yml 第 2 行），再覆写 site_name、docs_dir、主题语言与翻译后的 nav，主题/插件/扩展配置零重复。
3. **Docker 串行组装**：Dockerfile 按 zh → zh-hant → en → ja → ru → ko 顺序 COPY 各语言源码并执行 6 次 `mkdocs build`，最终 site/ 目录内含全部语言版本，由 `python -m http.server 8000` 直接托管（docker-compose.yml 映射 8000 端口）。

主题覆载承担了站点的"产品化"部分：overrides/partials/content.html 在每页正文末尾统一 include 评论块；comments.html 内嵌 giscus 脚本并监听主题切换按钮，通过 `postMessage` 让评论 iframe 跟随深浅色模式（overrides/partials/comments.html）；UI 文案集中在 overrides/partials/i18n.html 的 translations 字典（zh/zh-hant/en/ja/ru/ko 六套），评论提示语、公告栏、giscus 界面语言都由此分发。

### 5.4 示例代码风格：教学优先的最小实现

以 codes/java/chapter_heap/my_heap.java 为例：从列表建堆用自底向上的 `siftDown` 堆化，索引计算 left/right/parent 独立成私有方法并配中文注释；每章一个 Java package（`package chapter_heap;`），跨章节复用的 ListNode/TreeNode/Vertex/PrintUtil 收敛在 codes/java/utils/。所有语言示例保持"文件级自包含 + Driver Code"的风格，保证单文件可运行、可对照阅读。

## 6. 构建与运行

本地一键预览（来自 docker-compose.yml 与 Dockerfile）：

```bash
docker compose up --build
# 浏览器访问 http://localhost:8000，六语言站点同站提供
```

纯 mkdocs 方式（依据 Dockerfile 依赖与根 mkdocs.yml）：

```bash
pip install mkdocs-material==9.5.5 mkdocs-glightbox
cp -r docs build/docs && mkdocs build -f mkdocs.yml   # 或 mkdocs serve 本地预览中文站
```

运行/验证示例代码（依据 CI workflow 与 codes/docker-compose.yml）：

```bash
javac -d codes/java/build codes/java/**/*.java        # Java 示例全量编译（java.yml）
cd codes/python && python test_all.py                 # Python 示例冒烟（python.yml）
cd codes && LANGS="python cpp java csharp" docker compose build   # 多语言练习环境
```

## 7. 测试与工程质量

- **测试形态**：没有传统单元测试框架，采用"示例即测试"——14 个语言 CI 各自全量编译/运行 codes/ 下所有示例；Python 用 test_all.py 冒烟，Java 用 javac 编译验证，其余语言各用自家工具链（dotnet/go test 风格配置等）。
- **CI 规模**：.github/workflows/ 14 个 workflow，全部按 `paths` 过滤只在自己语言的文件变更时触发；Java 覆盖 JDK 11/17 × 4 语言站代码目录，Python 覆盖 ubuntu/macos/windows × 4 目录。
- **代码质量观察**：示例代码风格高度统一（文件头注释、章节 package、utils 复用），Python 强制 black 格式化；.gitattributes 用 linguist 标注把 C/C#/Zig/Rust 等标为 Other、排除 HTML/CSS 统计，使仓库语言画像聚焦在示例代码上。
- **文档工程**：全站 6 语言 × 约 120 页保持结构对齐（en/ja/ko/ru/zh-hant 各 119 个 .md，与主站仅差 1 页），编辑入口 `edit_uri: tree/main/docs` 直达源文件。

## 8. 社区与维护状态

- **活跃度**：极高。最近 push 2026-10-29（gh api），近三日提交为韩语版上线（#2006）与日语全书重译（#2008），翻译扩张仍在进行；open issues 仅 27，消化能力很强。
- **release 节奏**：约一年一个里程碑——1.1.0（2024-04）、1.2.0（2024-12）、1.3.0（2026-01），共 10 个 release；网站 `version: 1.3.0` 与 release 对应。
- **贡献者结构**：约 214 名贡献者（API 分页推算）；README 列出三条参与路径——内容修正（页面底部 giscus 评论区/PR）、代码转译（issue #15 认领语言）、翻译审校。
- **翻译流水线**（en/CONTRIBUTING.md 明文规定四步）：
  1. AI 初译：用大模型完成首译；
  2. 人工润色：译者修正机翻痕迹、保证行文自然；
  3. PR 双审：润色稿经审阅人通过 GitHub PR 复核；
  4. 循环迭代：重复 2-3 步直至定稿。
  这套流程解释了日语"全书重译"这类大动作为何能在数天内完成（#2008）。
- **生态位**：作者 krahets 为力扣"剑指 Offer"题解作者（docs/chapter_hello_algo/index.md 自述），README 获邓俊辉（清华）与李沐（亚马逊）推荐语；已出版纸质书并在顶栏公告引流。
- 评论数据、贡献者排行明细未在仓库内，标为"未确认"。

## 9. 亮点与风险

**亮点**

1. "内容-代码-可视化"三位一体：正文插图 + pythontutor 单步演示 + 14 语言可运行示例，入门体验在同类开源书中几乎没有对手（docs/、codes/、codes/pythontutor/）。
2. 站点工程成熟：`INHERIT` 配置继承 + build/docs 源产物分离 + Docker 六语言串行组装，多语言多站点维护成本被压到配置层面（en/mkdocs.yml、Dockerfile）。
3. 示例代码有 CI 质量兜底：14 个语言 workflow 按 path 精准触发、矩阵覆盖多版本多平台，"书里的代码永远能跑"（.github/workflows/）。
4. 读者互动内建：giscus 评论区随深浅色联动、i18n 文案集中管理，把 GitHub Discussions 变成逐页答疑区（overrides/partials/comments.html、i18n.html）。
5. 增长飞轮完整：名人推荐 + PDF/EPUB 免费 + 纸质书 + 多语言扩张，持续制造传播点（README.md、overrides/main.html 公告）。

**风险/局限**

1. CC BY-NC-SA 4.0 禁止商用，代码不能被商业项目直接引用，也不允许被用于商业分发（LICENSE、README.md License 节）。
2. 教学代码定位：示例追求可读而非性能/健壮性，不能当生产级数据结构库使用（codes/ 各语言示例均为最小实现）。
3. 六份代码全量镜像的同步成本随语言数线性增长，且 zh-hant、ru 的代码目录不在 CI 覆盖内，存在腐化窗口（java.yml/python.yml paths 证据）。
4. 仓库体积近 500MB（gh api size≈498999KB），图片资产与 14 份代码树使 clone/tarball 较重；swift 示例用符号链接表达 *_target 变体，在 Windows 上解压会报错（tar 提取实测）。

## 10. 总结评价

hello-algo 是"用软件工程的方法做开源教育内容"的标杆样本：把一本书拆成文档站、可运行代码、可视化演示、评论社区四条生产线，用 mkdocs-material 主题覆载和配置继承解决多语言站点的工程问题，用 14 条语言 CI 守住"示例永远可运行"的底线，再以翻译流水线和纸质书不断拓宽受众。130k 星并非单点爆款，而是内容质量、工程质量与社区运营三者共振的结果。它适合算法零基础的求职者当第一本教材读，适合想学新语言的人拿对应语言示例当练手材料，也适合研究"开源内容项目如何工程化"的维护者通读其 mkdocs 配置、overrides 模板与 workflow 组织方式。
