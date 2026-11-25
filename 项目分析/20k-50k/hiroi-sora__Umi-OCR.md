# hiroi-sora/Umi-OCR 项目分析

> 星数 35300（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：Windows/Linux 桌面端开源离线 OCR 工具箱——截图识别、批量图片识别、PDF 文档识别、二维码扫/生成，全部本地运行、无需联网。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | Umi-OCR | `README.md` |
| 星数 | 48428（gh api 实测 2026-11-05，快照为 35300） | gh api repos/hiroi-sora/Umi-OCR |
| 主语言 | Python（+QML） | gh api |
| 许可证 | MIT | `LICENSE` |
| 最新版本 | v2.1.5（2025-03-25 发布） | gh api releases、`CHANGE_LOG.md` |
| 最近提交 | 2025-11-20（Weblate 翻译自动更新） | gh api commits |
| 贡献者 | 14 人（核心作者 hiroi-sora 单人主导） | gh api contributors |

## 2. 项目是什么，解决什么问题

Umi-OCR 是一款"解压即用、离线运行"的桌面 OCR 软件（`README.md`：免费、开源、可批量的离线OCR软件，方便：解压即用，离线运行，无需网络）。它把 PaddleOCR 系的识别能力包装成普通用户可用的 GUI 工具，主打四类场景（`UmiOCR-data/py_src/tag_pages/`）：截图 OCR（ScreenshotOCR）、批量图片识别（BatchOCR）、批量文档/PDF 识别（BatchDOC，支持提取 PDF 自带文本层、生成双层可搜索 PDF）、二维码识别与生成（QRCode）。

软件对隐私敏感用户（无法上云的文档、内网环境）友好：OCR 引擎、语言模型库全部内置在发行包中，识别过程不经任何网络请求。同时它内置多国语言识别库与 14+ 种界面语言翻译（`UmiOCR-data/i18n/`、`dev-tools/i18n/release/*.ts`），中文用户体验尤其完善。

除了 GUI，它还提供三种自动化接口：本地 HTTP API（`/api/ocr`、`/api/doc/*`、`/api/qrcode`，见 `UmiOCR-data/py_src/server/`）、命令行调用（`docs/README_CLI.md`）以及全局热键/贴图等桌面增强功能。HTTP 服务只监听 127.0.0.1（`py_src/server/web_server.py` 中 `Host = "127.0.0.1"`），命令行本质上是"另一个进程通过本地环回 HTTP 与常驻主进程通信"的单实例设计（`docs/README_CLI.md` 明确说明依赖 HTTP 接口进行跨进程通信）。

在 OCR 质量层面，软件不满足于引擎原始输出：识别结果先经置信度聚合（`mission_ocr.py` 计算平均置信度），再由 TBPU 依用户选择的排版方案重组文本；PDF 场景还能复用文档自带文本层、剔除水印/页眉页脚（忽略区域）并产出可搜索的双层 PDF（`py_src/ocr/output/output_pdf_layered.py`）。这些"识别之外"的工程化处理是它在同类工具中建立口碑的关键。

项目采用"主仓库 + 卫星仓库"的拆分（`README.md` "关于项目结构"）：主仓库只含应用源码；OCR 引擎插件在 Umi-OCR_plugins 仓库；Windows/Linux 运行时（Python 解释器 + 依赖库）在 Umi-OCR_runtime_windows / _linux 仓库；底层引擎是作者自研的 PaddleOCR-json / RapidOCR-json 子进程引擎。因此本仓库内看不到模型文件与第三方依赖清单，发行包在发布时才组装。

## 3. 技术栈与依赖

- 语言：Python（后端约 12.6k 行，`UmiOCR-data/py_src/`）+ QML（前端 84 个文件约 13.1k 行，`UmiOCR-data/qt_res/qml/`），合计约 22k 行（不含 vendored bottle）。
- GUI 框架：PySide2（Qt for Python），QML 声明式界面；OpenGL 渲染器初始化与兼容性检查在 `py_src/utils/app_opengl.py`。
- HTTP 服务：内置 fork 的 bottle 4.2k 行（`py_src/server/bottle.py`）+ 标准库 wsgiref，仅本地回环。
- 第三方库（发行包内置，仓库无 requirements.txt，版本见 `CHANGE_LOG.md` v2.1.5 段）：PyMuPDF（PDF 解析/双层 PDF 写入）、Pillow、psutil、pynput（全局热键）、zxing-cpp（二维码）。
- OCR 引擎：PaddleOCR-json / RapidOCR-json（作者自己的项目），以独立可执行文件+stdin/stdout JSON 管道方式供插件调用；Linux 版另有 tar.xz 发行包（gh api releases 资产列表）。
- 打包：PyStand 定制版启动器（`UmiOCR-data/main.py` 注释明确说明负责 Windows + PyStand 运行环境初始化）。
- 国际化：界面翻译经 Hosted Weblate 众包，源码内维护 14+ 语言 .ts 源文件（`dev-tools/i18n/release/`）与 lupdate/lrelease 工具链脚本，构建产物为 `UmiOCR-data/i18n/` 下 6 种编译后 .qm；QML 侧由 `py_src/utils/i18n_configs.py` 的 I18n 模块接入 Qt 翻译体系（`py_src/run.py` 步骤 5）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `UmiOCR-data/main.py` | Windows 入口：PyStand 环境初始化（stdout 重定向、site-packages 路径、Qt 插件路径），失败时 ctypes 弹窗兜底 |
| `UmiOCR-data/py_src/run.py` | 正式入口：注册 10 个 QML 连接器类、启动翻译与 QML 引擎、加载 `qt_res/qml/Main.qml` |
| `py_src/tag_pages/` | 四大功能页（BatchOCR/BatchDOC/QRCode/ScreenshotOCR）+ Page 基类，页面只做编排，任务交给 mission |
| `py_src/mission/` | 异步任务系统：通用队列 Mission + OCR/DOC/QRCode 三个单例子类 |
| `py_src/ocr/` | OCR 插件接口（api）、文块后处理 TBPU（tbpu/）、8 种结果输出格式（output/） |
| `py_src/server/` | 本地 HTTP API 与命令行协议（bottle + web_server + ocr/doc/qrcode_server + cmd_client/cmd_server） |
| `py_src/plugins_controller/` | 动态插件加载器：扫描 `plugins/` 目录，校验 PluginInfo 契约 |
| `py_src/event_bus/` | 发布/订阅事件总线 + 全局键盘/鼠标监听（pynput） |
| `py_src/image_controller/` | 截图控制器、Qt 图片提供器（贴图/预览） |
| `py_src/platform/` | win32（ctypes API、按键翻译）/ linux 平台适配层 |
| `py_src/utils/` | 配置持久化、i18n、主题、线程池、OpenGL 等横切工具 |
| `py_src/imports/` | 加入搜索路径的独立模块：umi_about（版本/作者信息）、umi_log（日志）、plugin_i18n |
| `qt_res/qml/` | QML 界面源码（Main.qml、TabPages/、Widgets/、Themes/、Configs/、ApiManager/ 等 10 个子模块） |
| `dev-tools/i18n/` | 翻译工具链（lupdate/lrelease 批处理脚本、多语言 .ts） |
| `docs/` | 命令行手册、HTTP API 文档（api_doc.md 21.8KB）与示例 |
| 根目录 | README×3 语言、CHANGE_LOG.md、`Umi-OCR_Rapid_v2.1.5.7z`（99MB 发行包随仓库托管） |

组织方式评述：单体应用仓库，按"GUI 连接器 / 业务编排 / 任务系统 / 引擎插件 / 平台适配"分层，职责边界清晰；Python 侧几乎每个模块都有对应的 `*_connector.py` 作为 QML 桥（`py_src/run.py` 集中注册）。引擎、运行时、模型全部外置到卫星仓库，主仓库保持轻量——代价是本仓库无法独立构建完整可运行产品。

## 5. 核心架构与关键实现

### 5.1 QML-Python 连接器架构（入口链）

启动链为 `UmiOCR-data/main.py`（PyStand 初始化）→ `py_src/run.py:runQml()`。Python 类通过 `qmlRegisterType` 暴露给 QML，QML 持有连接器对象、Python 通过回调把结果推回界面（`py_src/run.py`）：

```python
qmlRegisterType(TagPageConnector, "TagPageConnector", 1, 0, "TagPageConnector")
qmlRegisterType(MissionConnector, "MissionConnector", 1, 0, "MissionConnector")
...
engine.load("qt_res/qml/Main.qml")  # 通过本地文件启动
```

这是典型的"QML 视图 + Python 连接器"混合架构：界面全在 QML，业务全在 Python，连接器是唯一交界。

### 5.2 OCR 引擎插件系统与外置引擎调用

引擎不在仓库内。`py_src/plugins_controller/plugins_controller.py` 在启动时扫描 `plugins/` 目录，`importlib.import_module` 动态加载每个包，校验其 `__init__.py` 必须定义 `PluginInfo`（含 `group`、`api_class`、可选 `global_options/local_options`），注册进 `ApiDict`。OCR 任务侧（`py_src/mission/mission_ocr.py`）用统一鸭子接口驱动引擎：

```python
if "path" in msn:
    res = self._api.runPath(msn["path"])
elif "bytes" in msn:
    res = self._api.runBytes(msn["bytes"])
elif "base64" in msn:
    res = self._api.runBase64(msn["base64"])
```

配置键 `ocr.win32_PaddleOCR-json.path`（`mission_ocr.py:144` 附近的短键还原逻辑）证实引擎形态：发行包内的插件包把 PaddleOCR-json/RapidOCR-json 可执行文件封装成 `api_class`，以子进程 JSON 管道交互；切换引擎即切换插件（README：可额外导入插件，随时切换不同OCR引擎）。这套设计让"应用"与"重达数百 MB 的引擎/模型"解耦，用户可按需下载 Paddle 或 Rapid（轻量 ONNX）发行包。

### 5.3 通用异步任务队列（Mission）

`py_src/mission/mission.py` 是所有批量操作的地基：每条队列携带 `onStart/onReady/onGet/onEnd` 四个回调，加入互斥锁保护的字典后由工作线程串行消费；`state` 支持 waiting/running/stop 与暂停恢复（`_msnPausedDict`）。`mission_ocr.py` 在队列前插两步：实例化 TBPU 后处理器（忽略区域 + 排版解析器），并在每队列开始时 `self._api.start(startInfo)` 热更新引擎参数；任务成功后统一计算平均置信度并逐个过 TBPU（见 5.4）。BatchDOC 页（`py_src/tag_pages/BatchDOC.py`）按 `ocr./doc./tbpu.` 前缀拆参数提交文档队列，体现同一任务系统对图片与 PDF 两类负载的复用。

### 5.4 TBPU 文块后处理（最具原创性的算法层）

OCR 引擎返回的是零散文块（box+score+text），`py_src/ocr/tbpu/` 负责把它们还原成可读文本：`tbpu.py` 定义基类（输入文块列表、输出加 `end` 分隔符），`parser_multi_*/parser_single_*.py` 提供 8 种排版方案（多栏/单栏 × 自然段/换行/无换行/保留缩进，选项清单见 `py_src/server/ocr_server.py` 的 `tbpu.parser` 配置），`parser_tools/gap_tree.py` 做基于间隙的分栏判定，`ignore_area.py` 实现水印/页眉页脚的忽略区域。这是 Umi-OCR 区别于"裸调 PaddleOCR 脚本"的核心价值之一。

### 5.5 本地 HTTP 服务与单实例命令行

`py_src/server/web_server.py` 用 vendored bottle + wsgiref 监听 127.0.0.1，按模块挂载路由：`/api/ocr`（GET options + POST 识别）、`/api/doc/upload|result|download|clear`、`/api/qrcode`（各 server 文件中的 `@UmiWeb.route`）。命令行指令（`umi-ocr --show/--quit/--slide` 等）由 `cmd_server.py` 定义、`cmd_client.py` 发送到常驻进程——既是自动化 API 也是防多开机制（`run.py`：`initCmd()` 初始化命令行，如果已有 Umi-OCR 在运行则结束运行）。

### 5.6 结果输出与文档识别

`py_src/ocr/output/` 将 OCR 结果落为 8 种格式：txt（含逐行/纯文本变体）、csv、jsonl、md、以及两类 PDF（单层纯文本层、双层可搜索层），由 `output/__init__.py` 的 Output 工厂按配置分发。BatchDOC 页进一步支持：PDF 自带文本层直接提取（不必重跑 OCR）、按页范围识别、带密码文档（`py_src/tag_pages/BatchDOC.py` 的 docs 参数含 `range_start/range_end/password`）。事件与热键层 `py_src/event_bus/` 提供 pubsub 总线与 pynput 全局键鼠监听，支撑"截图后自动贴图"等桌面工作流。

## 6. 构建与运行

- 终端用户：GitHub Releases 下载 `Umi-OCR_Paddle_v2.1.5.7z.exe`（自解压）或 Linux tar.xz，解压即用（gh api releases；`Umi-OCR_Rapid_v2.1.5.7z` 也直接托管在仓库根目录）。
- 开发者：需先按卫星仓库部署运行环境——Windows 见 Umi-OCR_runtime_windows、Linux 见 Umi-OCR_runtime_linux（`README.md` "构建项目"节）；运行时把 `UmiOCR-data/main.py` 作为入口（依赖 PyStand 提供的 python.exe + site-packages）。
- 源码直接调试（Linux/开发机）：进入 `UmiOCR-data/` 后 `python3 main.py`（需自备 PySide2/PyMuPDF/Pillow/psutil/pynput/zxing-cpp 等依赖与引擎插件包；仓库内无 requirements.txt，属"未确认"的可复现性缺口）。
- 翻译流程：`dev-tools/i18n/lupdate_all.py` → weblate 平台协作 → `lrelease_all.py` 生成 `.qm`（`dev-tools/i18n/README.md`、`UmiOCR-data/i18n/`）。

## 7. 测试与工程质量

- 测试：全仓库未发现任何测试文件/测试框架（`find -iname "*test*"` 为空），纯手工测试项目。
- CI：`.github/` 仅有 issue 模板（01_bug.yaml、02_request.yaml），无 GitHub Actions 工作流；发布产物（7z/tar.xz）应为本地打包。issue 模板为 YAML 表单式，要求用户填写版本号与复现步骤，配合"版本号自动生成"机制（`UmiOCR-data/about.json` 中 version.string 为"自动生成"，由运行时仓库构建时注入）。
- 代码质量观察：优点——注释密度高且为双语读者服务，模块划分与命名一致，连接器/业务/任务三层分离彻底，日志系统完善（v2.1.5 引入 loguru 风格 `py_src/imports/umi_log.py`）。缺点——无类型标注约定、无静态检查配置、`__MissionOcrClass` 等单例模式依赖导入时序、部分文件（cmd_server.py 524 行）函数较长。

## 8. 社区与维护状态

- 活跃度：最新提交 2025-11-20（Weblate 翻译自动 PR），2025-05-31 有功能提交（新增click screenshot #892），最近一次正式版 v2.1.5 为 2025-03-25；commit 均带 issue 号（#652、#690、#785、#789 等），issue 驱动开发明显。当前处于"维护模式"（翻译自动化持续、功能迭代放缓），作者正筹备 v2.2 重构（CHANGE_LOG 顶部与 README "开发计划"：重构底层插件机制）。
- 贡献者结构：14 名贡献者，核心代码几乎全部出自 hiroi-sora 一人；翻译通过 Hosted Weblate 众包（最近多次提交均为 "Translations update from Hosted Weblate"）。
- 社区生态：README 提供中/英/日三语说明、赞助渠道，并维护插件库与运行时库等卫星仓库；star 从快照 35300 增长到 48428（gh api，2026-11-05），是中文开源工具类项目的头部。
- Release 节奏：v2.1.2（2024-06）→ v2.1.3（2024-07）→ v2.1.4（2024-08）→ v2.1.5（2025-03），大版本间隔渐长、小版本约季度级；每次发布同时提供 Windows Paddle/Rapid 双引擎包与 Linux 包（gh api releases）。

## 9. 亮点与风险

亮点：
1. 离线 OCR 桌面化的标杆实现：把 PaddleOCR 系引擎做成"解压即用"的完整产品，隐私友好，覆盖截图/批量/PDF/二维码全场景（`README.md`、`py_src/tag_pages/`）。
2. 干净的插件化引擎抽象：`plugins_controller` + `runPath/runBytes/runBase64` 鸭子接口，Paddle/Rapid 引擎可热切换，应用与模型解耦（`py_src/plugins_controller/plugins_controller.py`、`py_src/mission/mission_ocr.py`）。
3. TBPU 文块后处理是稀缺的中文排版还原实现：8 种解析方案 + 间隙树分栏 + 忽略区域（`py_src/ocr/tbpu/`）。
4. 多入口自动化设计统一：HTTP API、CLI、防多开共用 127.0.0.1 本地服务，命令行即 API 客户端（`py_src/server/web_server.py`、`docs/README_CLI.md`）。
5. 工程拆分克制合理：主仓库/插件/运行时/引擎四仓解耦，主仓库 22k 行保持可读（`README.md` 项目结构节）。

风险/局限：
1. 无测试、无 CI：回归完全靠手工，重构（如计划中的插件机制重构 v2.2）风险高（本仓库无 test 文件、`.github/` 无 workflows）。
2. 依赖可复现性差：仓库不含 requirements.txt/锁文件，构建依赖散落在卫星仓库文档中，源码级二次开发门槛高（第 6 节）。
3. 主仓库托管 99MB 发行包 `Umi-OCR_Rapid_v2.1.5.7z`，clone/tarball 体积虚胖（实测 tarball 111MB，其中源码仅约 6MB）。
4. 强绑定 Windows 生态：PyStand、win32_api、ctypes 弹窗等路径 Linux 覆盖较薄（`py_src/platform/` 两平台代码量不对等）。

## 10. 总结评价

Umi-OCR 是"把学术引擎做成民用产品"的典范：单作者以克制的多仓库架构和清晰的 QML-连接器分层，将 PaddleOCR/RapidOCR 封装成人人可用的离线 OCR 工具箱，并额外贡献了 TBPU 中文排版还原、双层 PDF、本地 HTTP API 等高价值组件。它不是依赖管理规范化的现代工程项目（无测试、无 CI、无依赖清单），但作为桌面软件产品其代码组织、注释质量与用户体验打磨在同类开源项目中属第一梯队。适合普通用户直接下载发行版做日常 OCR；适合二次开发者学习 QML+Python 混合桌面架构、插件化引擎集成与本地服务化设计；不适合作为需要持续集成保障的生产级 Python 库来引用。
