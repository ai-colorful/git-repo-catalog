# ocrmypdf/OCRmyPDF 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：给扫描版 PDF 无损叠加可搜索 OCR 文字层的命令行工具——栅格化预处理、调用 Tesseract 识别、把定位文字层"嫁接"回原 PDF，并输出带校验的 PDF/A。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | ocrmypdf/OCRmyPDF | gh api（组织 ocrmypdf，原 fritz-hh） |
| 星数 | 快照 39500（2026-11-21 gh api 实测 35,365） | gh api |
| 语言 | Python（>=3.11），纯 Python 无扩展模块 | pyproject.toml |
| 许可证 | MPL-2.0（文档/测试 CC-BY-SA-4.0，REUSE/SPDX 管理） | LICENSE、REUSE.toml |
| 当前版本 | 17.14.2（2026-11-16 发布） | pyproject.toml、gh api releases |
| 最近活跃 | 最近 push 2026-11-16，月度级发版 | gh api pushed_at |
| 官网 | ocrmypdf.readthedocs.io | README.md |

## 2. 项目是什么，解决什么问题

OCRmyPDF 解决的是"扫描件不可搜索"这一痛点：扫描得到的 PDF 本质是一张张图片，没有文字层，无法搜索和复制。作者 Julien Pfefferkorn（fritz-hh）在 README「Motivation」中列出了当年现有工具的缺陷——文字层错位导致复制乱码、丢多语言重音字符、改变嵌入图像分辨率、输出体积爆炸、产出非法 PDF、不输出 PDF/A（README.md），因此自研了这个工具。

它的工作方式是一条多步骤页级流水线：先把 PDF 页栅格化为高分辨率图像（可顺便纠偏 deskew、去背景、clean 除斑、旋转矫正），交给 OCR 引擎（默认 Tesseract，支持 100+ 语言包）识别，再把带精确坐标的不可见文字层"嫁接"到原 PDF 页面下方，最后做 PDF/A 转换、图像压缩优化和线性化。整个过程对原始图像"无损"——保留原分辨率，文字层只叠加不替换（README.md、src/ocrmypdf/_pipeline.py）。

工程上它有三大立身之本：一是正确性（输入输出双重校验、PDF/A 验证，built on pikepdf/QPDF）；二是并行（页级 worker 池，默认吃满全部 CPU 核，`--jobs` 控制）；三是可扩展（pluggy 插件体系允许整体替换 OCR 引擎、光栅化器、PDF/A 后端）。paperless-ngx 等文档管理系统把它作为 OCR 后端集成（README.md Plugins 节）。

项目已有 12 年历史：2013-12 由 fritz-hh 创建（gh api 验证 fritz-hh/OCRmyPDF 仍在、261 星、2016-01 停更），2015 年起 James R. Barlow 全面接手并迁入 ocrmypdf 组织，此后十余年由其主导维护至今。

## 3. 技术栈与依赖

- 语言与构建：Python >=3.11，hatchling 构建，uv 管理依赖（uv.lock 867KB），ruff + mypy + prek 做质量门禁（pyproject.toml）。
- 核心 Python 依赖（pyproject.toml `dependencies`）：
  - pikepdf（QPDF 绑定）——PDF 读写、嫁接、PDF/A 校验的基石；pdfminer.six——解析已有文本层（skip/redo 模式判断）；pypdfium2（PDFium）——备选光栅化器
  - Pillow + img2pdf——图像处理与图像转 PDF；fpdf2 + uharfbuzz——v17 新文字层渲染器（fpdf2 绘制 + HarfBuzz 字体整形，含 BiDi 双向文字）
  - pluggy——插件系统；pydantic v2——`OcrOptions` 配置模型；rich——终端交互
- 可选 extras：heic（pillow-heif）、watcher（watchfiles 目录监控自动 OCR）、webservice（streamlit Web 界面）、rapidocr（onnxruntime 的 PP-OCR 模型，内置替代 OCR 引擎）（pyproject.toml `[project.optional-dependencies]`）
- 外部二进制依赖（强约束）：Tesseract OCR 4.1.1+ 与 Ghostscript 必装；可选 unpaper、pngquant、jbig2enc、verapdf（src/ocrmypdf/_exec/ 目录、README.md Requirements 节）

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| src/ocrmypdf/ | 单一主包（src layout），197 个 py 文件约 5.7 万行 |
| ├ _pipelines/ | 三条顶层管线：ocr.py（标准 OCR）、hocr_to_ocr_pdf.py、pdf_to_hocr.py |
| ├ _pipeline.py | 页级处理步骤库（栅格化/预处理/OCR 调度/PDF/A/优化，1765 行） |
| ├ _graft.py | 文字层嫁接器 OcrGrafter（1072 行） |
| ├ fpdf_renderer/ + font/ | fpdf2 文字层渲染器与字体管理（MultiFontManager、系统字体探测） |
| ├ hocrtransform/ + models/ | hOCR 解析为引擎无关的 OcrElement 树 |
| ├ builtin_plugins/ | 内置插件：tesseract_ocr、rapidocr_ocr、ghostscript、pypdfium、optimize、concurrency、null_ocr |
| ├ _exec/ | 外部二进制封装：tesseract、ghostscript、unpaper、pngquant、jbig2enc、verapdf |
| ├ pluginspec.py | pluggy hookspec 全集 + OcrEngine 抽象基类（684 行） |
| ├ cli.py / api.py / _options.py | argparse CLI（52 选项）、公开 Python API（1118 行）、pydantic OcrOptions |
| ├ pdfinfo/ | 页信息与已有文本 layout 分析（pdfminer） |
| └ optimize.py、pdfa.py | 输出图像压缩优化、PDF/A 转换 |
| tests/ | 85 个测试文件、1380 个测试、2.8 万行 + 测试插件 tests/plugins/ |
| docs/ | Sphinx 站点（含 design_notes.md、releasenotes/version02..17.md 十七个大版本纪事） |
| .docker/、snapcraft.yaml、scripts/ | Docker（含 alpine/rapidocr 变体）、Snap 打包、发版辅助脚本 |

组织方式评述：单体单包、插件化内核。核心只做调度与 PDF 装配，所有"能力"（OCR 引擎、光栅化器、PDF/A 后端、优化器）都注册为 pluggy 插件，内置功能与第三方扩展同一条接口——这是它十余年保持可演进的关键。

## 5. 核心架构与关键实现

### 5.1 页级 OCR 流水线（src/ocrmypdf/_pipeline.py、_pipelines/ocr.py）

`_pipelines/ocr.py::_run_pipeline` 串起总流程：work folder → `triage()`（`_pipeline.py:273`，图像输入自动包成 PDF）→ `do_get_pdfinfo` → `validate_pdfinfo_options` → `exec_concurrent`（每页一个 worker）→ postprocess。单页流水线 `_exec_page_sync` 依次为 `is_ocr_required`（用 pdfminer 检查是否已有文本层，决定按 default/force/skip/redo/strip 六种 ProcessingMode 处理，`_options.py:49`）→ `process_page`（rasterize → deskew/clean/remove_background → create_ocr_image）→ `_image_to_ocr_text` → 结果交给 OcrGrafter。并发由 `Executor` 统一封装线程/进程两种模型（`_concurrent.py`，docs/design_notes.md 解释其同接口设计）。

文字层生成有三种路径（`_pipelines/ocr.py:58`）：
- `ocr_engine_direct`：引擎实现新 `generate_ocr()` API，直接返回 OcrElement 树（fpdf2 默认渲染器）；
- `ocr_engine_hocr`：老路径，引擎产出 hOCR 文件，`hocrtransform/hocr_parser.py` 解析为 OcrElement 树；
- `ocr_engine_textonly_pdf`：Tesseract `textonly_pdf` 模式直出文字 PDF，用于 sandwich 渲染。

### 5.2 文字层嫁接与 fpdf2 渲染器（src/ocrmypdf/_graft.py、fpdf_renderer/renderer.py）

`OcrGrafter.graft_page`（`_graft.py:661`）是核心："emplacement update"用同一 objgen 原位替换页面内容（保留交叉引用），再把文字层以 `RenderMode.UNDERNEATH` 置于图像之下（保证复制粘贴体验）。v17 的主力渲染器 `fpdf_renderer/renderer.py`（1186 行）直接从 OcrElement 树用 fpdf2 绘制不可见文字：字体子模块（font/multi_font_manager.py 578 行、system_font_provider.py 579 行）负责按语种挑字体并用 uharfbuzz 整形，连阿拉伯语/希伯来语的 BiDi 视觉序重排都做了（renderer.py `_visual_order`，处理 issue #1781）。配套 `_graft.py` 还实现 strip 模式所需的"剥离既有不可见文字"（`strip_invisible_text`）。

### 5.3 Tesseract 集成（src/ocrmypdf/_exec/tesseract.py、builtin_plugins/tesseract_ocr.py）

与 PyTesseract 式 ABI/绑定不同，OCRmyPDF 始终以外部进程 CLI 方式调用 Tesseract（docs/design_notes.md 明确解释了这一取舍：CLI 稳定不脆弱）。`_exec/tesseract.py` 是参数矩阵的全部：`tess_base_args` 拼 `-l` 语言链与 `--oem`，可加 `--psm` 页分割模式、`thresholding_method` 配置项、`user-words/user-patterns` 用户词典；识别时一次调用同时要 `hocr txt` 两种输出（tesseract.py:393）。健壮性细节：识别超时或"Empty page"时生成空 hOCR 兜底继续跑（`_generate_null_hocr`）；`OMP_THREAD_LIMIT` 压制 Tesseract 内部多线程以避免与页级并行打架；用 `--psm 0` 做方向探测、另有 deskew 角度探测。选项层（tesseract_ocr.py `TesseractOptions` + `add_options`）把 `--tesseract-*` 系列参数注册进 CLI。

### 5.4 插件体系（src/ocrmypdf/pluginspec.py、_plugin_manager.py）

pluggy（pytest 同款机制）驱动，hookspec 全集在 pluginspec.py：生命周期（initialize/add_options/check_options）、基础设施（get_executor/get_progressbar_class/get_logging_console）、页处理可替换点（rasterize_pdf_page、filter_ocr_image/filter_page_image/filter_pdf_page）、OCR 引擎（get_ocr_engine）、输出（generate_pdfa、optimize_pdf）。OCR 引擎抽象 `OcrEngine` ABC（pluginspec.py:413）定义 version/languages/get_orientation/generate_hocr/generate_pdf/（可选）generate_ocr 契约。自带 rapidocr_ocr.py（2161 行）即以纯插件实现了 PP-OCR/onnxruntime 引擎，证明接口完备；社区还有 AppleVision、EasyOCR、PaddleOCR 引擎插件（README.md）。插件可经 `--plugin` 传入或在 entry-points 注册，pydantic `register_options` 机制让插件自持配置命名空间（`_options.py`）。

## 6. 构建与运行

```bash
# 安装（需系统先装 tesseract-ocr 与 ghostscript）
pip install ocrmypdf          # 或 apt/brew/snap 安装发行版（README.md）
uv sync --all-extras          # 开发环境（pyproject dependency-groups）

# 典型用法（README.md）
ocrmypdf --deskew --rotate-pages --jobs 4 --output-type pdfa -l eng+fra in.pdf out.pdf
ocrmypdf --mode redo in.pdf out.pdf      # 重做已有文字层
ocrmypdf --mode strip in.pdf out.pdf     # 仅剥离旧 OCR 层

# Python API（api.py::ocr）
import ocrmypdf; ocrmypdf.ocr('in.pdf', 'out.pdf', deskew=True)

# 测试（pyproject [tool.pytest.ini_options]，默认 -n auto 并行）
pytest                        # 1380 个测试，需 PATH 中有 tesseract/ghostscript
pytest -m 'not slow'
```

## 7. 测试与工程质量

- 规模：tests/ 下 85 个 py 文件、1380 个 `def test_`、28,099 行，另有 tests/plugins/ 自定义测试插件与 tests/resources/ 大量样张。覆盖面极广：从 hOCR 解析、graft、PDF/A 验证、rotion 到 watcher 安全、rapidocr 引擎、decompression bomb 等（tests/ 目录）。
- 风格：pytest + pytest-xdist（`-n auto`）+ hypothesis 属性测试 + coverage（branch、multiprocessing、subprocess patch，pyproject.toml `[tool.coverage.run]`）。
- CI（.github/workflows/build.yml「Test and deploy」）：prek lint → Linux 多 Python 版矩阵（真实安装 Tesseract PPA/Ghostscript）→ macOS（Homebrew 依赖）→ Windows，Codecov 上传；另有 release.yml、triage.yml、dependabot。
- 质量观察：全库 SPDX/REUSE 版权标注（REUSE.toml）、mypy check_untyped_defs、ruff 十余条规则族、frozen pydantic 配置模型、中文 README（README_ZH.md）与 17 个大版本的 release notes（docs/releasenotes/）。文档有专门的 design_notes.md、errors.md 等深度材料。

## 8. 社区与维护状态

- 活跃度（gh api，2026-11-21）：最近 push 2026-11-16；open issues 70；forks 2,458；未归档。
- 发版节奏（gh api releases）：v17.14.2（2026-11-16）、v17.14.1（11-09）、v17.14.0（11-08）、v17.13.0（09-28）、v17.12.x（09-16）、v17.11.0（08-28）、v17.10.0（08-05）——近年保持月度级高频发布，release notes 写到变更细节级别（docs/releasenotes/version17.md）。
- 贡献者：123 人（gh api contributors 分页实测）。单人主导特征明显：James R. Barlow 为作者与主要维护者（pyproject authors、release notes 署名）。
- 历史接手：原仓库 fritz-hh/OCRmyPDF（2013-12 创建，2016-01 停更，现存 261 星），2015 年 James R. Barlow 接手后迁至 ocrmypdf 组织独立运作（gh api 两仓库对照验证）。
- 发行渠道广：PyPI、Debian/Fedora 官方源、Homebrew、Snap（snapcraft.yaml）、Docker（.docker/ 三个 Dockerfile）。

## 9. 亮点与风险

亮点：
1. 架构自洽的插件内核——OCR 引擎/光栅化器/PDF/A 后端/优化器全部 hookspec 化，内置 rapidocr 引擎即最佳示范（pluginspec.py、builtin_plugins/rapidocr_ocr.py）。
2. 文字层渲染做到排版级细节：HarfBuzz 整形、多字体回退、BiDi 视觉序、行基线/斜体拉伸处理（fpdf_renderer/renderer.py、font/）。
3. 12 年持续演进的健壮性工程：超时空 hOCR 兜底、解码炸弹防护、stdout 保护、原位 objgen 嫁接保留引用（_exec/tesseract.py、_graft.py、_stream_check.py）。
4. 测试规模与真实环境 CI 在同体量 Python CLI 项目中属顶配（1380 测试 × 3 平台矩阵）。
5. 处理模式体系完整：default/force/skip/redo/strip 六种模式 + sidecar 纯文本 + watch 目录 + streamlit Web UI（_options.py:49、可选 extras）。

风险/局限：
1. 硬外部依赖 Tesseract + Ghostscript：版本敏感（Tesseract 4.1.1+），Windows/裸容器部署门槛高，Ghostscript 的 AGPL 授权关系需注意（docs/ 有 pdfsecurity、docker 专文处理此类问题）。
2. force-ocr 模式整页栅格化，矢量与原生文本会退化为位图（`_options.py` ProcessingMode 文档明示）。
3. 核心维护高度集中于一人，bus factor 偏低（不过 12 年稳定 + 123 贡献者缓解了此风险）。
4. 快照星数（39500）与 gh api 实测（35,365）存在出入，以 GitHub 实时数据为准（未确认差异来源）。

## 10. 总结评价

OCRmyPDF 是"把一件事做到极致"的典范：十二年只做"给扫描件加可搜索文字层"，却在正确性（PDF/A 校验）、性能（页级并行）、可扩展性（pluggy 插件内核）和排版细节（HarfBuzz/BiDi/多字体）上层层加码，最终成为 paperless-ngx 等文档系统的默认 OCR 后端和 Linux 世界的事实标准。它适合三类人使用：需要批量可搜索化的个人与运维（CLI 直接用）、做文档管理/归档系统的开发者（Python API + 插件替换 OCR 引擎）、想学习"外部二进制编排 + 页级并行 + 插件化 Python 架构"的工程师（源码分层清晰、测试完备，_pipeline.py/_graft.py/pluginspec.py 三个文件是最佳入口）。
