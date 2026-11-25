# PDFMathTranslate/PDFMathTranslate 项目分析

> 星数 39500（快照 2026-10-24；gh api 实测 38409）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：开源科学 PDF 论文翻译工具（PyPI 包名 `pdf2zh`），在保留公式、图表、目录与排版的前提下整篇翻译，输出 mono（纯译文）与 dual（双语对照）两份 PDF。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | PDFMathTranslate（pdf2zh） | `pyproject.toml` |
| 星数 | 38409（2026-11-14 gh api） | gh api repos/PDFMathTranslate/PDFMathTranslate |
| 语言 | Python（>=3.11,<3.13） | `pyproject.toml` |
| 许可证 | AGPL-3.0 | `LICENSE`（661 行完整 AGPL 文本）、`pyproject.toml` |
| 版本 | 1.9.12 | `pdf2zh/__init__.py`、`pyproject.toml` |
| 最近 push | 2026-11-14（当日仍活跃） | gh api pushed_at |
| 创建时间 | 2024-09-06 | gh api created_at |
| 学术背书 | EMNLP 2025 System Demonstrations 论文 | `README.md` 5.1 引用条目 |
| 主页 | https://pdf2zh.com（官方在线服务） | gh api homepage |

## 2. 项目是什么，解决什么问题

学术论文 PDF 的翻译长期被两类方案瓜分：一类是纯文本提取（丢公式、丢排版），一类是浏览器级翻译（无法处理 PDF）。PDFMathTranslate 的切入点是"保留排版的整篇翻译"：解析 PDF 内容流，把文字段落抽出来交给翻译引擎，公式、图表、表格等非文字元素原样保留，再把译文按原位置重新排版写回 PDF（`README.md` 第 1 节）。它是自称"世界第一个开源保留排版科学文档翻译软件"的工具，已被 EMNLP 2025 Demo track 接收（`README.md` 5.1）。

核心洞察是：PDF 里公式从来不是图片，而是一串用数学字体（Computer Modern 等）绘制的字符。因此只要识别"哪些字符属于公式"，就可以在指令流层面把它们摘出来不动，只翻译自然语言段落，翻译后再把公式按原字形、原偏移"插回"译文（`pdf2zh/converter.py` receive_layout 的 A/B/C 三段逻辑）。这使它不依赖公式 OCR 或 LaTeX 还原，速度快且保真度高——公式仍然是可以选中、可搜索的矢量字符。

输出为双文件：`-mono.pdf`（纯译文）与 `-dual.pdf`（左右/原文-译文对照），由 PyMuPDF 合并生成（`pdf2zh/high_level.py`）。默认中英翻译，支持数十种目标语言（为 CJK/阿/俄/泰等按语言配了默认行距表与 Go Noto Universal 备用字体，`pdf2zh/converter.py` LANG_LINEHEIGHT_MAP、`pdf2zh/high_level.py` noto_list）。

工程背景是中国开发者社区项目（作者 Byaidu，大量中文注释与中文 issue 模板 `.github/ISSUE_TEMPLATE/问题反馈_cn.yaml`），与"沉浸式翻译"生态深度联动：后者为贡献者提供 Pro 会员奖励，并基于同团队后续的 BabelDOC 提供在线服务（`README.md` 5.2 Acknowledgement）。对比定位上，沉浸式翻译是浏览器插件形态、面向网页与文档；pdf2zh 是本地 CLI/GUI 工具、面向论文 PDF；`pdf2zh.com` 与 HF/ModelScope Demo 提供免安装在线版（`README.md` 3.1）。

## 3. 技术栈与依赖

- 语言：Python 3.11–3.12，构建用 hatchling，包管理推荐 uv（`pyproject.toml`）。
- PDF 解析：pdfminer.six（内容流解释与版面元素），在其上做了深度子类化（`pdf2zh/pdfinterp.py`、`pdf2zh/converter.py`）。
- PDF 写入/渲染/合并：PyMuPDF（<1.25.3，页面对象 patch、pixmap 渲染、OCR 引擎）、pikepdf（字体子集化相关）。
- 版面检测：ONNX Runtime + DocLayout-YOLO（`wybxc/DocLayout-YOLO-DocStructBench-onnx`，经 babeldoc 的 assets 机制下载），OpenCV 做预处理（`pdf2zh/doclayout.py`）。
- 翻译服务 SDK：googletrans、deepl、openai、azure-ai-translation-text、tencentcloud-sdk-python-tmt、ollama、xinference-client 等（`pyproject.toml` dependencies）。
- GUI：Gradio 6（>=6.16.0，注明修复 CVE-2026-28414）+ gradio_pdf（`pdf2zh/gui.py`）。
- 缓存：peewee + SQLite（`pdf2zh/cache.py`）；重试 tenacity；输出美化 rich。
- 可选 extras：`ultrafast`（pdf-inspector）、`ocr`（pooch 下载 tessdata）、`backend`（Flask+Celery+Redis）、`mcp`（mcp>=1.6.0）、`cuda`/`dml`（GPU 推理）、`argostranslate`（`pyproject.toml` [project.optional-dependencies]）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `pdf2zh/` | 全部核心代码的单包 Python 工程（约 6300 行） |
| `pdf2zh/pdfinterp.py` | 内容流解释器子类，拦截 PDF 操作符、剥离原文字指令、产出补丁指令流 |
| `pdf2zh/converter.py` | 核心翻译转换器：段落/公式切分、翻译调度、译文重排版 |
| `pdf2zh/high_level.py` | 高层编排（translate/translate_stream）：版面检测→逐页 patch→输出 mono/dual |
| `pdf2zh/translator.py` | 24 个翻译引擎 Provider 的统一抽象（1189 行） |
| `pdf2zh/doclayout.py` | DocLayout-YOLO ONNX 推理封装（CPU/CUDA/DML 后端选择） |
| `pdf2zh/kernel/` | 翻译内核注册表 + v2（next）桥接：protocol/registry/legacy/precise/v2_bridge/v2_worker |
| `pdf2zh/gui.py` | Gradio WebUI（932 行） |
| `pdf2zh/pdf2zh.py` | CLI 入口（argparse，`pdf2zh` 命令） |
| `pdf2zh/backend.py` | Flask + Celery 异步任务后端（HTTP API） |
| `pdf2zh/mcp_server.py` | MCP 服务（STDIO/SSE），可接入 LLM 客户端 |
| `pdf2zh/ultrafast.py`、`ocr.py`、`text_spacing.py` | 极速模式版面、扫描页 OCR 段落重组、CJK-拉丁间距 |
| `pdf2zh/cache.py`、`config.py` | SQLite 翻译缓存、JSON 配置管理 |
| `pdf2zh/converter_docx.py` | Word(doc/docx) 经 LibreOffice 转 PDF 的预处理 |
| `test/` | pytest 测试（10 个文件、约 2064 行）+ 3 个测试用 PDF |
| `docs/` | ADVANCED/APIS/GUI 文档与 5 语种 README（en/zh-CN/zh-TW/ja/ko） |
| `.github/workflows/` | 测试、发布、Windows exe 构建、star history 等 7 个工作流 |
| `Dockerfile`、`docker-compose.yml`、`app.json` | Docker/Heroku/Zeabur 等部署 |

组织方式评述：单包单体（src 平铺在 `pdf2zh/`），无内部子包分层，但模块职责清晰、文件粒度合理（最大 translator.py 1189 行）。2.0 内核以 git submodule（`.gitmodules` 指向 PDFMathTranslate-next）+ 隔离 venv 的方式挂在 `pdf2zh/kernel/` 下，主仓保持 1.x 稳定发布线。

## 5. 核心架构与关键实现

### 5.1 指令流级"补丁"翻译（PDF 解析与版面重建核心）

`pdf2zh/pdfinterp.py` 子类化 pdfminer 的 `PDFPageInterpreter`，逐条执行 PDF 内容流操作符时做两件事：一是 `execute()` 过滤掉所有 T 系列文字指令（Tj/TJ/Tf/Tm…），保留图形/图像指令作为"底图"；二是把译文排版指令追加为新的页面内容流，通过 `obj_patch` 记录每个页面对象（含 Form XObject）的替换指令，最后用 `q … Q cm` 把原页面坐标系还原：

```python
# pdf2zh/pdfinterp.py:274-276
self.obj_patch[page.page_xref] = (
    f"q {ops_base}Q 1 0 0 1 {x0} {y0} cm {ops_new}"
    # ops_base 里可能有图，需要让 ops_new 里的文字覆盖在上面，使用 q/Q 重置位置矩阵
)
```

`do_S()` 只保留"独立水平黑色直线"（数学分式线），其余填/描边路径全部丢弃（`pdf2zh/pdfinterp.py:112-150`）。新内容流直接写入新建 xref 并 `set_contents` 替换原页（`pdf2zh/high_level.py:227-230`）。这是该项目区别于"重建 PDF"类方案的关键：不重排整个文档，只做指令流减法+加法，公式字形、图片、注释全部原封不动。

### 5.2 公式/段落分离（启发式 + 神经版面双通道）

`pdf2zh/converter.py` 的 `receive_layout` 分 A（解析）、B（翻译）、C（重排版）三阶段。公式判定 `vflag()` 用字体名正则（LaTeX 数学字体家族）+ Unicode 类别（数学符号、修饰符、希腊字母区 U+0370-03FF）双规则，可被 `--vfont/--vchar` 覆盖：

```python
# pdf2zh/converter.py:211-215
if re.match(
    r"(CM[^R]|MS.M|XY|MT|BL|RM|EU|LA|RS|LINE|LCIRCLE|TeX-|rsfs|txsy|wasy|stmary|.*Mono|.*Code|.*Ital|.*Sym|.*Math)",
    font,
):
    return True
```

版面通道则用 DocLayout-YOLO 对整页渲染图推理出 10 类区域，非 abandon/figure/table/公式类区域标号 2..n、保留区标 0，形成与像素对齐的逐字符类别掩码 `layout[pageid]`（`pdf2zh/high_level.py:196-225`）。字符级判定综合"掩码类别 + 角标字号比例 0.79 + 公式字体 + 垂直字体 + 括号组"五种条件（`pdf2zh/converter.py:248-262`）。公式组以 `{vN}` 占位符内嵌进段落文本，翻译 prompt 明确要求"Keep the formula notation {v*} unchanged"（`pdf2zh/translator.py:139-152`），译文排版时再按记录的宽度 `vlen` 与纵向偏移 `varf` 原字形重绘公式字符与分式线（`pdf2zh/converter.py:478-506`）。行数超框时按目标语言行距自适应压缩（zh 1.4 → 逐次 -0.05，`pdf2zh/converter.py:397-401,536-537`）。

### 5.3 翻译引擎抽象与缓存

`pdf2zh/translator.py` 以 `BaseTranslator`（translate 模板方法 + do_translate 钩子 + SQLite 缓存 + envs 三级优先：显式 > 环境变量 > 配置文件）为基类，派生 24 个 Provider：Google、Bing、DeepL、DeepLX、Ollama、Xinference、OpenAI、AzureOpenAI、Azure、腾讯、Zhipu/智谱、ModelScope、SiliconFlow、302.AI、Gemini、AnythingLLM、Dify、Argos（本地离线）、Grok、Groq、DeepSeek、MiniMax、OpenAIliked（任意 OpenAI 兼容端点）、QwenMT；其中 9 个直接继承 OpenAITranslator 复用其实现（类声明见 `pdf2zh/translator.py:169-1116`，import 清单见 `pdf2zh/converter.py:20-46`）。缓存按（引擎、参数、原文）唯一键落 SQLite（`pdf2zh/cache.py`），支持 `--ignore-cache` 与按参数区分缓存命中。

### 5.4 内核注册表：v1/v2 双内核与降级路径

`pdf2zh/kernel/` 实现了可热切换的翻译内核层：`protocol.py` 定义 `TranslateRequest/TranslateResult` 统一数据类；`registry.py` 提供线程安全注册/切换（默认 `fast`=v1）；`precise.py` 把 v2 内核（PDFMathTranslate-next 子模块）装进独立 venv 并以子进程运行，stderr 逐行 JSON 回传进度、stdout 返回结果：

```python
# pdf2zh/kernel/precise.py:20-22 与 v2_bridge.py 桥接
_SUBMODULE_DIR = Path(__file__).resolve().parent / "PDFMathTranslate-next.git"
_VENV_DIR = _SUBMODULE_DIR / ".venv"
_WORKER_SCRIPT = Path(__file__).resolve().parent / "v2_worker.py"
```

`v2_bridge.py` 负责 v1 请求 → pdf2zh_next CLI 参数 + `PDF2ZH_*` 环境变量的映射（23 个服务名映射，`pdf2zh/kernel/v2_bridge.py:15-39`）。另有三条轻量路径：`--ultrafast` 用 pdf-inspector 原生文本坐标做启发式版面，完全跳过 ONNX 与 OCR，文档级 16 线程翻译池（`pdf2zh/ultrafast.py`、`high_level.py:103-104`）；实验性 OCR 对纯图扫描页用 PyMuPDF 内置 Tesseract 识别后按版面区域重组段落、中位字号排版（`pdf2zh/ocr.py`）；`--babeldoc` 直接调用 funstory-ai/BabelDOC 后端（`pyproject.toml` 依赖 babeldoc>=0.1.22）。入口侧提供 CLI（`pdf2zh.py`）、Gradio WebUI（`gui.py`，端口 7860）、Flask+Celery 后端（`backend.py`）与 MCP server（`mcp_server.py`）四种形态。

## 6. 构建与运行

```bash
# 安装（README.md 3.2）
pip install uv && uv tool install --python 3.12 pdf2zh   # 或 pip install pdf2zh
pdf2zh document.pdf                 # 默认 Google 翻译，输出 example-mono.pdf / example-dual.pdf
pdf2zh example.pdf -s deepl -lo zh -t 4 -p 1-3          # 服务/目标语言/线程/页码
pdf2zh -i                           # Gradio WebUI，http://localhost:7860
pdf2zh example.pdf --ultrafast      # 原生文本极速模式（需 pip install 'pdf2zh[ultrafast]'）
pdf2zh --mode precise example.pdf   # 实验性 v2 内核（先 pdf2zh-setup-precise 装隔离 venv）
pdf2zh --mcp                        # MCP STDIO 服务
# 开发（pyproject.toml / CI）
uv sync && uv run pytest .          # 单元测试
uv build                            # 构建包
# Docker（README.md 3.2.6）
docker pull byaidu/pdf2zh && docker run -d -p 7860:7860 byaidu/pdf2zh
```

Windows 免安装 exe 由 `script/build-win64.ps1` + `.github/workflows/exe-build.yml` 构建发布。

## 7. 测试与工程质量

- 测试：`test/` 10 个 pytest 文件共约 2064 行，覆盖 cache、converter、doclayout、translator、kernel（703 行，最大）、OCR 布局、text_spacing、ultrafast 与 CLI；另带 3 个真实 PDF 端到端用例（`test/file/`）。风格偏功能级/集成测试，未见覆盖率配置。
- CI：`.github/workflows/python-test.yml` 在 PR 时于 Python 3.11/3.12 × ubuntu x86/ARM 矩阵跑 pytest + 2 个真实 PDF 翻译冒烟测试 + GUI 启动测试（timeout 10s 判活）+ uv build；BabelDOC 资产做了哈希键缓存。`python-publish.yml` 负责 PyPI 发布、GHCR/DockerHub 镜像与 Windows exe；另有 black 格式化工作流与 pre-commit（`.pre-commit-config.yaml`）。
- 质量观察：代码注释密集（大量中文"hack"注释解释 Why，如 `pdf2zh/converter.py:250` 角标比例 0.76/0.799 的取中说明）；对上游库行为有精确的版本锚定（pymupdf<1.25.3、tencentcloud-sdk 钉死 3.1.70、gradio>=6.16.0 修 CVE），说明踩坑经验被固化进了依赖声明。flake8 忽略 E501 等格式类规则，依赖 black 统一风格。

## 8. 社区与维护状态

- 活跃度：gh api 显示最近 push 为 2026-11-14（分析当日），open issues 170，forks 3442；README"Recent Updates"更新至 2026-10-03（CJK-拉丁自动加间距），即最近数周内仍在持续合入功能（`README.md` 2）。
- 贡献者：contributors API 分页 59 页（per_page=1，含匿名），约 58+ 人；核心贡献者 Byaidu 与 reycn（README 更新日志中多数条目署名 reycn）。
- 发布节奏：GitHub Release 最新 v1.9.11（2025-07-11），此前约每月一版；代码版本已到 1.9.12 但未正式发 Release——1.x 线在 2.0 拆分后转为"稳定维护"节奏。2.0 主线（BabelDOC 内核）拆至组织内姊妹仓 PDFMathTranslate-next 独立开发（`README.md` 4.3 明确"该 fork 仅用于开发、不面向社区贡献"），本仓通过 kernel 桥接试用（未确认 v1.9.12 未发 Release 的具体原因）。
- 生态与背景：中文开发者社区项目（HelloGitHub/GitCode/ModelScope 徽章、中文 issue 模板与 Telegram 群，`README.md` 徽章区）；获沉浸式翻译（Immersive Translate）商业赞助；衍生生态包括 Zotero 插件 zotero-pdf2zh（`README.md` 3.2.5）。

## 9. 亮点与风险

亮点：
1. 指令流级 patch 方案（减文字、保图形、加译文）实现了"公式零损耗"：公式以原字体字形重绘，不经过 OCR/LaTeX 还原，保真与速度兼得（`pdf2zh/pdfinterp.py`、`pdf2zh/converter.py`）。
2. 启发式规则（数学字体正则 + Unicode 类别 + 字号比例）与神经版面（DocLayout-YOLO 逐字符掩码）双通道互验，公式/图表/正文分离鲁棒（`pdf2zh/converter.py:197-262`、`pdf2zh/high_level.py:196-225`）。
3. 24 个翻译引擎统一抽象 + SQLite 结果缓存 + `{v*}` 公式占位符协议，LLM 翻译时防公式被改写（`pdf2zh/translator.py`）。
4. 形态完整：CLI/WebUI/Docker/Windows exe/HTTP 后端/MCP server 六种入口，双输出 mono/dual，覆盖学术全流程。
5. 工程治理成熟：PR 矩阵 CI 带真实 PDF 冒烟与 GUI 判活、版本锚定踩坑经验、EMNLP 2025 论文背书。

风险/局限：
1. AGPL-3.0 具强传染性，商业集成（尤其 SaaS 分发）需谨慎评估合规（`LICENSE`）。
2. v1/v2 双线分裂：最新能力集中在 next 仓与 BabelDOC，1.x 依赖 googletrans 等逆向接口，存在上游接口失效风险（`pyproject.toml` 中 googletrans==4.0.2 钉死即是信号）。
3. 依赖较重：ONNX Runtime、OpenCV、Gradio、babeldoc 资产（首次运行需下载模型与字体），弱网环境安装门槛高（README 专设 HF_ENDPOINT 镜像说明）。
4. 排版为"逐段原位重排"，极端复杂版面（多栏跨页、竖排、密集表格）仍是弱项，官方也推荐复杂文档用 normal 模式而非 ultrafast（`README.md` Ultrafast 一节）。

## 10. 总结评价

PDFMathTranslate 是"保留排版翻译 PDF"这一细分方向的事实标准开源实现：核心创意在于把翻译问题转化为 PDF 指令流的减法与加法，用字体启发式 + 版面检测双通道识别公式并原字形回插，从而绕开公式 OCR 的精度陷阱；外围以 24 引擎抽象、多形态入口与完整 CI/发布体系支撑起近 4 万星与 EMNLP 论文的社区影响力。1.x 线目前处于稳定维护期（v2 能力经 kernel 桥接渐进引入），活跃度依然很高。适合科研人员/学生作为论文阅读工具直接安装使用；适合想研究 PDF 内容流操作、版面分析、翻译流水线设计的开发者精读 `pdf2zh/pdfinterp.py` 与 `pdf2zh/converter.py`——两文件不足千行却完整示范了"最小侵入式 PDF 重写"的工程范式；商用集成需先评估 AGPL 合规与 v2（next）迁移路径。
