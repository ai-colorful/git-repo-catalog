# NaiboWang/EasySpider 项目分析

> 星数 42500（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-06
> 注：分析当日 GitHub API 实测星数为 44,766，fork 5,395，最近 push 2026-11-05。

## 1. 项目速览

一句话定位：一个"在网页上点选内容即可生成爬虫规则"的可视化无代码采集/浏览器自动化软件，Electron 设计端 + 浏览器扩展采集端 + Python 执行引擎三件套，全平台免费开源。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | EasySpider（易采集） | Readme.md |
| 星数 | 42,500（任务快照）/ 44,766（API 实测 2026-11-06） | gh api repos/NaiboWang/EasySpider |
| 语言 | JavaScript（Electron 主程序）+ Python（执行引擎）+ TypeScript/Vue（扩展） | 各子目录源码 |
| 许可证 | AGPL-3.0 | LICENSE、Readme.md |
| 当前版本 | 0.6.5 | ElectronJS/package.json |
| 最近发布 | v0.6.5（2026-08-19），上一版 v0.6.3（2025-01-01） | GitHub Releases API |
| 最近提交 | 2026-11-05（PR #1010 合并） | GitHub Commits API |
| 贡献者 | 13 人（作者主导） | GitHub Contributors API |
| 学术背景 | WWW 2023（CCF-A）论文 + 国家发明专利 + 浙大硕士论文 | Readme.md 出版物章节 |

## 2. 项目是什么，解决什么问题

EasySpider 解决的核心问题是"写爬虫的门槛"：传统采集需要定位元素、写 XPath、处理翻页循环、等待与解析，而 EasySpider 把这一整套流程压缩为"右键点选目标元素 → 软件自动匹配同类元素 → 选择采集/点击/循环等选项"，任务被表示为一张可视流程图并保存为 JSON，之后既可以回放执行，也可以通过命令行或 HTTP 接口嵌入其他系统（Readme.md 开头描述；ElectronJS/server.js 提供任务存储接口）。

产品形态是桌面软件而非库或 SaaS：下载即用的安装包内置了专属的 Chrome 浏览器副本和对应版本 chromedriver，用户不需要自备浏览器环境（ElectronJS/README.md 环境构建章节）。设计阶段由浏览器扩展在目标网页内注入点选工具条，Electron 主程序通过 Selenium 驱动一个"设计浏览器"实时回放用户操作；执行阶段则由 PyInstaller 打包的 Python 程序独立运行任务，脱离设计界面。

任务模型上，一个任务就是一个 JSON 文档（`tasks/<id>.json`），其中 `graph` 字段是扁平的节点列表，节点之间用 `sequence`/`parentId` 连成流程树，支持循环嵌套、条件分支、自定义 JS/Python 代码、打开网页、点击、输入、提取数据等节点类型（Examples/最简单的翻页采集列表.json 实测；ExecuteStage/constants.py 的 GraphOption 枚举）。提取到的数据支持 CSV/TXT/JSON/XLSX/MySQL 五种落盘方式（ExecuteStage/easyspider_executestage.py 中 outputFormat 分支）。

该项目同时有较强的学术血统：作者将其作为浙江大学硕士论文课题（Docs/ 目录收录论文 PDF），系统被 WWW 2023 接收为 demo paper，并有"一种自定义提取流程的服务封装系统"发明专利（Readme.md 出版物章节）。这也解释了它"可视化规则生成 + 流程图解释执行"的整体设计取向。

README 中有明确的免责与合规声明（禁止违法爬取、后果自负），并详细解释了 AGPL-3.0 的传染性与 SaaS 开源义务，明确"个人和企业可免费使用、免费二次开发、无需商业授权"（Readme.md 声明章节）。仓库另提供中文样例任务集（Examples/：翻页列表、循环进详情页、JS 判断条件、含 Python 自定义代码的任务等），供用户导入 tasks 目录后直接体会流程模型，这相当于用可执行样例替代了大部分使用文档。

## 3. 技术栈与依赖

| 层 | 技术 | 关键依赖 | 出处 |
|---|---|---|---|
| 主程序 | Electron 27 + Node.js | express/cors/multer（本地 Web 服务器）、ws（WebSocket）、selenium-webdriver（设计期浏览器控制）、xlsx、formidable、node-window-manager | ElectronJS/package.json |
| 设计端 UI | 本地 HTML + Vue 3（打包内嵌 vue.global.js）+ Bootstrap | ElectronJS/src/ 静态页面，主窗口加载 `http://localhost:8074/index.html` | ElectronJS/main.js:214 |
| 浏览器扩展 | Manifest V3 + TypeScript + Vue 2 + rollup + jquery | Extension/manifest_v3/package.json、rollup.config.js | Extension/manifest_v3/src/manifest.json |
| 执行引擎 | Python 3.7+，PyInstaller 打包 | selenium==4.47.0、lxml、ddddocr（OCR）、pymysql、xlsxwriter/openpyxl、pandas、undetected-chromedriver（自带 fork）、pynput（键盘暂停）、commandline_config | ExecuteStage/requirements.txt |
| 浏览器 | 便携版 Chrome + 匹配版本 chromedriver（各平台目录随安装包分发） | chrome_win32/win64/linux64/mac64 目录约定 | ElectronJS/README.md、ElectronJS/main.js:144-167 |

值得注意的一个架构事实：主程序的"无代码设计"依赖两套浏览器通道——扩展负责网页内交互与规则采集，Selenium（JS 版）负责设计期真正点击/输入/切标签页；执行期则完全换到 Python 版 Selenium（myChrome.py 自定义了 MyChrome/MyEdge/MyUCChrome 三个驱动类，ExecuteStage/myChrome.py）。

## 4. 目录结构与代码组织

| 目录 | 职责 | 证据 |
|---|---|---|
| ElectronJS/ | Electron 主程序：main.js（进程编排、WebSocket 枢纽）、server.js（本地 HTTP 服务 8074，任务/执行实例 CRUD + /invokeTask API）、src/（设计端页面：taskGrid 流程图、bootstrap 工具页）、打包脚本 package_win64.cmd 等 | ElectronJS/README.md |
| Extension/manifest_v3/ | 浏览器扩展源码（TS+Vue），package.js 用 rollup 打包成 CRX；打包产物以 EasySpider_zh.crx / EasySpider_en.crx 内置于主程序 | Extension/README.md、ElectronJS/package_win64.cmd |
| ExecuteStage/ | Python 执行引擎（easyspider_executestage.py 2702 行为核心），PyInstaller 打包脚本，自带 undetected_chromedriver_ES vendored fork | ExecuteStage/Readme.md |
| Examples/ | 样例任务 JSON（翻页、详情页循环、JS 判断条件等） | Examples/Readme.md |
| scripts/ | 无浏览器依赖的单元/回归测试（unittest） | scripts/test_preprocess.py |
| Docs/ | 作者硕士论文 PDF | 目录清单 |
| media/ | README 截图与赞助商 banner 图片 | 目录清单 |

组织方式评述：三个可独立构建的子项目（Node / 浏览器扩展 / Python）靠"固定端口（8074/8084）+ 固定文件名（EasySpider.crx、easyspider_executestage.exe、tasks/*.json）"约定粘合，不是 monorepo 工具链管理的多包，而是典型的"手工约定式多组件"结构。ElectronJS/README.md 明确给出三者的编译顺序依赖（先扩展、再主程序、再执行器），交叉引用靠文件复制（如执行器 exe 放进 chrome_win64 目录）完成。

## 5. 核心架构与关键实现

### 5.1 三进程协作与"点选即规则"链路

设计期的消息流是理解本项目架构的关键（证据：ElectronJS/main.js:1411-1480、Extension/manifest_v3/src/content-scripts/messageInteraction.js）：

1. 主程序启动本地 HTTP 服务（端口 8074，server.js）和 WebSocket 服务（端口 8084，写死，main.js:115 注释"扩展里面写死了"；绑定 127.0.0.1，注释引用 issue #986 修复局域网窃取 Cookie 风险）。
2. 扩展 content script 注入所有 frame（manifest.json `all_frames: true`），监听 mousemove 并用 `document.elementFromPoint` 抓取当前元素，叠加高亮框（content-scripts/main.js 顶部）。
3. 用户点选后，扩展通过 `global.js` 维护 nodeList/outputParameters 状态机，生成多条候选 XPath（`getElementXPaths`：绝对 XPath、文本 contains、id() 等多条备选，global.js:49），工具条（toolkit.vue，934 行 Vue 组件）提供"选中全部/选中子元素/采集数据/循环点击"等选项。
4. 选择结果以 `{type:3, from:0, message:{pipe: JSON}}` 经 WebSocket 发给主程序，payload 携带 `xpath`、`allXPaths`、`useLoop`、`loopType` 等语义字段（messageInteraction.js sendSingleClick 等）。
5. 主程序用 Selenium 在设计浏览器中定位并回放该元素操作，同时流程图窗口（src/taskGrid/FlowChart_CN.html）实时加节点；保存时经 server.js 的 /manageTask 写入 `tasks/<id>.json`（server.js:436 附近）。

WebSocket 连接按 `msg.message.id` 注册角色：0=主窗口、1=启动页、2=流程图、3=popup、4=后端，其余 id 视为浏览器标签页并触发 Selenium 切换 window handle（main.js:1424-1480），这是一套自定义的轻量多端会话协议。

### 5.2 同类元素自动匹配算法

自动匹配是产品核心卖点，实现意外地朴素有效（Extension/manifest_v3/src/content-scripts/global.js:1033 `findRelated`）：把选中元素的 XPath 拆成段数组（如 `["html","body","div[1]"]`），从最深层开始逐层去掉索引谓词生成候选 XPath，用 `document.evaluate` 枚举，若能枚举到第二个元素即认为找到"同类块"，把该弱化 XPath 交给用户确认。配合 `combineXpath`/`relatedTest`（global.js:1125、1142）完成批量选中。没有用 DOM 树相似度或机器学习，纯 XPath 结构泛化，代价是对结构高度规则的页面有效、对异构列表需要手动补选（工具条提供多模式兜底，Readme.md 截图"四种选择方式"）。

### 5.3 执行引擎：流程图 JSON 解释器

执行核心是 `BrowserThread` 类（ExecuteStage/easyspider_executestage.py:109），一个任务一个线程，可多任务并行（easyspider_executestage_single.py 文件头注明旧版单任务脚本已废弃）。`executeNode`（:965）按 option 分发节点：0/10=根/容器递归 sequence，1=openPage，2=click，3=extract，4=input，5=customOperation，6=switchSelect，7=moveToElement，8=loopExecute，9=judgeExecute。每节点执行后按 `wait/waitType` 做固定或随机等待（`random.uniform(wait*0.5, wait*1.5)`，:1046 附近），异常捕获后仅记日志继续执行，保证长任务健壮性。

循环类型覆盖 8 种（loopExecute :1200）：0 单元素、1 不固定元素列表、2 固定元素列表、3 文本列表、4 网址列表、5 JS 命令、6 系统命令、7 Python 命令返回值。条件分支 `judgeExecute`（:1058）支持页面包含文本/元素、循环项包含文本/元素、JS/System 命令返回值等 9 类条件，逐条短路匹配。

一个重要的性能设计是"快速提取"：`getData`（:2064）对每个提取参数带 `optimizable` 标记，条件满足时直接对 `page_source` 用 lxml 的 XPath 一次性求值，绕开逐元素 Selenium 调用；循环层也有 `quickExtractable` 分支（:1230 附近）把整页当 HTML 树批量抽取行与字段，并对链接/图片做 urljoin 补全。`detect_optimizable`（ExecuteStage/utils.py:184）在设计期预判参数是否可走快车道（含 waitElement、JS 前后处理等则不可）。

参数化与数据流：`replace_field_values`（utils.py:394）把模板中的 `Field["参数名"]`、`eval(...)`、`JS(...)` 展开——Python 侧用 repr 字面量、JS 侧用 json.dumps 转义（utils.py:363/378 两个安全替换函数），把前序提取值注入后续节点的 XPath/输入值/代码中，实现"提取值作为变量输入"的链式数据流。

可靠性细节：暂停/恢复用 threading.Event（`self.event.wait()` 贯穿各节点），键盘 `p` 键暂停（pynput，utils.py:118）；`startFromExit` 把已提取步数写入 steps.txt 支持中断续采（:155-173）；输出按 saveThreshold 批量落盘，dataWriteMode 支持追加/覆盖/重命名（:184-190，constants.py DataWriteMode）。

### 5.4 服务化调用与本地 API

任务可直接被外部系统调用：server.js 的 `/invokeTask`（:500）接受任务 id + 参数 JSON，按 inputParameters 的 name 匹配并回写对应节点（option 1 改 links、option 4 改输入值、option 8 改循环文本/次数），然后以新 execution instance 启动执行器；执行器亦支持纯命令行运行（`--ids`、`--headless`、`--user_data`、`--docker_driver` 等，easyspider_executestage.py:2340 附近 Config 默认值），并内置一个随机端口的 IPC HTTP 服务向 Node 父进程回报 `IPC_SERVER_PORT`（:2287-2325），用于终止信号等控制。

## 6. 构建与运行

构建顺序必须为 扩展 → 主程序 → 执行器（ElectronJS/README.md、ExecuteStage/Readme.md）：

```bash
# 1) 扩展：生成 CRX
cd Extension/manifest_v3 && node package.js

# 2) 主程序：装依赖（Windows 需 VS Build Tools 2017 C++ 组件供 node-gyp）
cd ElectronJS && npm install
npm run start_direct          # 开发模式，只能设计任务

# 3) 执行器（需另配 Python 环境）
cd ExecuteStage && pip install -r requirements.txt
# 各平台打包：generateEXE_win64.cmd / generateExecutable_Linux64.sh / generateExecutable_Macos.sh（PyInstaller）

# 发布包（Windows x64）：先备好 chrome_win64+chromedriver+执行器 exe+CRX
cd ElectronJS && package_win64.cmd && clean_and_release_win64.cmd
```

打包由 electron-forge 完成，maker 覆盖 squirrel（Windows 安装包）、zip（macOS）、deb、rpm（package.json `config.forge.makers`）。注意便携 Chrome 与版本匹配的 chromedriver 不随仓库分发，需按 README 约定目录名手工放置（chrome_win64/ 等）。运行期自定义 JS 支持试运行与正式运行两种语义（main.js:1354-1367 对 Field[""]/eval("") 在试运行期给出告警）。

## 7. 测试与工程质量

- 测试规模：scripts/ 下 8 个 Python 测试文件 + 1 个 JS bidi 测试，共约 51 个 test 函数（grep 统计）。无 e2e 爬取真实站点的测试（test_preprocess.py 文档字符串明确"Do not cover browser startup or crawling a live page"）。
- 测试风格亮点：test_preprocess.py 用 AST 从源码中"摘出" preprocess 函数及纯工具函数加载执行，避免导入执行器模块时初始化浏览器/OCR 依赖——说明作者清楚模块级副作用对可测性的伤害，并给出了低成本解法。其余覆盖 JSON 导出、MySQL 配置、图片下载、控制台编码、捆绑 Chrome 探测等边界。
- CI：没有。.github/ 仅有 FUNDING.yml 和 ISSUE_TEMPLATE.md，无 workflow，测试不进流水线，质量依赖作者本地执行。
- 代码质量观察：核心文件偏长（main.js 1878 行、执行器 2702 行），大量中文注释与日志双语音讯；注释中保留较多历史演进痕迹（如 Win7 检测注释块、ffi-napi 注释掉的原型）；近两年开始出现系统性安全加固痕迹（server.js 的 safeJoin/safeHandler 路径穿越防护、main.js WebSocket 绑定回环地址并注明 issue #986、执行器 remote_control_key 防泄漏处理），结合提交信息（fix(executor): default missing MySQL configuration path 等）可见有外部贡献者（SJ-Xia-Q，使用 codex/ 前缀分支）参与规范化改进。

## 8. 社区与维护状态

- 活跃度：仓库持续活跃，最近 push 2026-11-05（GitHub API）；但 release 节奏慢，5 年仅 5 个正式版（v0.5.0 2023-07 → v0.6.5 2026-08），用户主要通过 master + Releases 安装包跟进。
- 贡献者结构：13 名贡献者（API 统计），作者 Naibo Wang 主导；2026 年下半年出现两位外部贡献者的 PR 合并（#1009、#1010）。Issue 支持：README 建议 GitHub Issue 为主，辅以 QQ 群（682921940）。
- 收入与推广模式（客观陈述）：README 无任何付费墙或云服务入口；盈利途径为 GitHub Sponsor / 支付宝 / 微信 / PayPal 打赏，以及 Bright Data、CapSolver、Proxy-Seller、Byteful 四家代理/验证码服务商的带返佣码的赞助 banner（Readme.md 赞助者章节）。代码中未发现许可证校验、激活逻辑或对 easyspider.cn 的 API 依赖（对 ExecuteStage/ElectronJS 源码 grep 无结果）；"How to Solve Captcha in EasySpider with CapSolver Integration.md" 是随仓库收录的赞助方软文，执行器代码中无 capsolver 调用。
- 官网 easyspider.cn/.net 仅作为国内下载镜像与文档入口（Readme.md），软件本体完全离线可用。

## 9. 亮点与风险

亮点：

1. "点选即规则"的端到端产品化完整：扩展采集语义 → 流程图编辑 → JSON 持久化 → 独立引擎回放 → 本地 API 调用，链路闭环且全平台打包即用（ElectronJS/main.js、server.js、ExecuteStage/ 全链路可读）。
2. 同类元素匹配用 XPath 索引剥离的轻量算法解决 80% 场景，实现 60 行以内（global.js:1033），工程性价比极高。
3. 执行引擎的"快速提取"双通道设计（lxml 批量 vs Selenium 逐元素）显著降低大规模列表采集的开销，且设计期自动预判 optimizable（easyspider_executestage.py:2064、utils.py:184）。
4. 反爬处理有分层：stealth.min.js 经 CDP 注入 + navigator.webdriver 抹除（:169-181）、vendored undetected-chromedriver fork、随机等待、代理与 headless、ddddocr 本地 OCR（contentType 8），均离线免费；付费打码仅作为第三方生态推荐而非代码内硬绑定。
5. 无浏览器依赖的 AST 式单元测试是个聪明的可测性方案（scripts/test_preprocess.py）。

风险/局限：

1. 三组件靠端口/文件名约定松散粘合，构建链路对环境要求苛刻（VS Build Tools、Chrome/chromedriver 手工配版本、CRX 手动同步），新人从源码构建成本高（ElectronJS/README.md 大篇幅 Troubleshooting 可证）。
2. 无 CI、核心文件巨石化，回归风险集中在 main.js 与执行器单文件；安全问题（如 #986 局域网 WebSocket）历史上靠 issue 驱动而非流程保证。
3. AGPL-3.0 对想闭源二次开发或 SaaS 化的企业是实质限制（Readme.md 明确列出网络服务开源义务），商业集成需谨慎评估。
4. 设计期依赖真实浏览器点选，复杂 iframe/验证码/强反爬站点的成功率天然受限；OCR 仅内置 ddddocr 一档，验证码能力主要外推到付费服务商（代码内无集成，需用户自行写代码调用）。

## 10. 总结评价

EasySpider 是"学术原型长成大众工具"的典型样本：以 XPath 结构泛化 + 流程图解释器两个简洁的核心算法，撑起了完整的产品链路（设计、存储、执行、服务化），并以全平台免安装依赖的打包质量换取了数万非程序员的用户群；它的代码不算精致，但每一处设计都能看到对真实用户痛点的回应（快速提取、断点续采、随机等待、试运行告警）。适合不会写代码但有采集需求的用户直接使用；对工程师而言，它是一份研究"可视化自动化工具如何做产品化"的优秀读本，重点可读 Extension/manifest_v3/src/content-scripts/global.js（规则生成）与 ExecuteStage/easyspider_executestage.py（流程解释器）两个文件。
