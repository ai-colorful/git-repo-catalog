# opendatalab/MinerU 项目分析

> 星数 82366（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：MinerU 是 OpenDataLab 出品的高质量文档解析工具，把 PDF、Office、EPUB、OFD、HTML、图片等格式转换成 Markdown/JSON，专为 LLM 语料准备与 Agent 文档阅读设计。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | MinerU（PyPI 包名 `mineru`） | `pyproject.toml` |
| 当前版本 | 4.1.3 | `mineru/version.py` |
| 语言/Python 要求 | Python 3.11–3.14 | `pyproject.toml` |
| 许可证 | Apache 2.0 + 附加条款（超大规型商用需商业授权） | `LICENSE.md` |
| 最近活跃 | 最近 push 2026-10-30；近三天连发 4.1.0–4.1.3 | GitHub API、releases |
| 仓库规模 | `mineru/` 包约 7.7 万行 Python、623 个 .py 文件 | 源码统计 |
| Stars/Forks | 82366 / 6849，约 104 位贡献者 | 快照、GitHub API |

## 2. 项目是什么，解决什么问题

MinerU 解决的核心问题是"把复杂版式文档变成机器可读的结构化内容"。学术论文里的多栏排版、教科书中的公式、财报中的嵌套表格、扫描件的印章与手写体，直接用 PDF 文本层抽取都会碎成无法使用的文本流。MinerU 用"版面分析小模型 + 专用识别模型 + 可选 VLM"的混合管线，将文档重建为带层级、公式 LaTeX、表格 HTML 的 Markdown 与 JSON（`README.md`、`mineru/backend/analysis/pdf/`）。

4.x 版本（当前 4.1）是一次彻底的产品化重构，从单纯的"PDF 转 Markdown 工具"演进为"文档解析 + 本地文档库 + 服务工具"一体化工作流（`README.md` Overview 节）。它提供四个解析档位：Flash（快速预览/索引）、Basic（OCR 与模型解析）、Standard/Advanced（高要求版面与质量），档位决定调用哪些模型（`mineru/parser/tier.py`、`mineru/integrations/docvortex.py` 中 effort→tier 映射）。

输入格式覆盖面很宽：PDF、图片、DOC/DOCX、PPT/PPTX、XLS/XLSX、RTF、ODT/ODS/ODP、EPUB、OFD、HTML/MHTML、CSV/TSV（`README.md`、`mineru/backend/analyze.py` 按后缀路由到各格式分析器）。输出统一为内部 MiddleJson/ModelJson 文档协议，再由渲染层产出 9 种目标：Markdown、HTML、LaTeX、DOCX、EPUB、PDF、Structured Content、Content List V1/V2（`mineru/render/__init__.py`）。

4.x 的差异化方向是"面向 Agent 的文档阅读"：本地文档库支持文件发现、结果缓存、全文搜索、按页/块定位符续读、稳定引用定位；`README.md` 的 YAML frontmatter 本身就是一段给编码 Agent 的 skill 描述，仓库还内置 `skills/mineru/` 供 `npx skills add` 安装。原生 Office 等格式解析由同团队的 DocVortex 库承担（`mineru/integrations/docvortex.py`）。

项目有正式技术报告：MinerU（arXiv:2409.18839）、MinerU2.5（arXiv:2509.22186）、MinerU2.5 Pro（arXiv:2604.04771），模型权重发布在 HuggingFace/ModelScope（`mineru/model/registry.py`）。

## 3. 技术栈与依赖

- 语言与构建：Python 3.11–3.14，setuptools 构建，官方推荐 uv 安装（`pyproject.toml`、`README.md`）。
- CLI/服务框架：typer + click + rich（CLI）、fastapi + uvicorn + pydantic v2（HTTP 服务）、gradio（WebUI）（`pyproject.toml` 依赖表、`mineru/cli/main.py`）。
- PDF 底层：pypdfium2 负责页面渲染、页范围重写（`mineru/parser/mineru_parser.py`、`docvortex` 的 pdf 模块）。
- 小模型推理：ONNX Runtime 全平台自动加速——Windows 走 onnxruntime-directml，Linux x86_64 另有 WebGPU EP，macOS 依赖 Torch/MPS（`pyproject.toml` 平台条件依赖、`mineru/model/runtime/` 下 onnx/webgpu/vulkan/intel_gpu 等模块）。
- 专用小模型：PP-DocLayoutV2（版面）、PP-FormulaNet_plus-M（公式）、PP-OCRv6 det/rec + 印章检测（OCR）、SLANet-plus/UNet/PP-LCNet（表格）（`mineru/model/registry.py`）。
- VLM：MinerU2.5-Pro-2605-1.2B，四种引擎 llama.cpp（GGUF，默认）、vLLM、LMDeploy、MLX 按平台自动选择（`mineru/model/vlm/selector.py`）。
- 文档库：SQLite（aiosqlite）存储 + jieba 中文分词全文检索（`mineru/doclib/core/db.py`）。
- 模型分发：huggingface-hub 与 modelscope 双源下载（`mineru/model/download.py`）。
- 可选重型依赖：torch/transformers（`mineru[torch]`）、vLLM（Linux）、LMDeploy（Windows）（`pyproject.toml` extras）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `mineru/parser/` | 解析门面：MinerUParser、档位（tier）、页范围、API client/server |
| `mineru/backend/analyze.py` | 统一分析入口 `doc_analyze`/`aio_doc_analyze`，按格式路由 |
| `mineru/backend/analysis/` | 各格式分析器：pdf/、csv、epub、html、mhtml、ofd、office |
| `mineru/backend/analysis/pdf/` | PDF 管线核心：pipeline、window、layout、ocr、formulas、tables、text/ |
| `mineru/backend/postprocess/` | ModelJson → MiddleJson 的后处理与 LLM 辅助标题分级 |
| `mineru/model/` | 模型层：layout/ mfr/ ocr/ table/ vlm/ + registry（模型仓库）+ download + runtime/ |
| `mineru/model/runtime/` | 硬件探测、设备选择、ONNX EP 管理、显存/内存治理（27 个模块） |
| `mineru/render/` | 9 种输出格式的渲染器 |
| `mineru/doclib/` | 本地文档库：server、client SDK、SQLite、FTS、解析队列、扫描、搜索服务 |
| `mineru/cli/` | typer CLI，16 个顶层命令（parse/read/scan/watch/search/server 等） |
| `mineru/kit/` | 服务工具：api_server、file_parse_server、router、vlm_server、webui、hardware |
| `mineru/tasks/` | 统一异步 Tasks API（/v1/tasks）的契约与执行 |
| `mineru/integrations/` | DocVortex 文档协议集成 |
| `tests/` | 222 个 .py，unittest/ 下 199 个测试文件，另有 benchmarks/browser/ci/fixtures |
| `docs/` `docker/` `skills/` `demo/` | mkdocs 文档站、Docker（china/global 双源）、Agent skill、示例 |

组织方式评述：单体单包（single PyPI package `mineru`），内部按"门面层（parser/cli/kit）→ 编排层（backend）→ 模型层（model）→ 数据层（doclib）→ 渲染层（render）"清晰分层；重型可选依赖通过 extras 和平台条件依赖隔离，服务端进程与客户端 SDK 分离。原 `projects/` 子项目目录已归档，生态项目迁移到 awesome-mineru（`projects/README.md`）。

## 5. 核心架构与关键实现

### 5.1 统一解析门面与格式路由

入口链是 `MinerUParser.parse()` → `doc_analyze()`。后者按 file_suffix 路由：PDF 走完整模型管线，csv/epub/html/mhtml/ofd/office 各有轻量分析器，最后统一构造 ModelJson 并由后处理转 MiddleJson（`mineru/backend/analyze.py`）。PDF 输入在此层完成图片转 PDF、页范围重写（pdfium 重写字节流）与坏页占位补齐（`mineru/parser/mineru_parser.py` 的 `_maybe_adjust_pdf_bytes`、`_insert_broken_pages`）。

### 5.2 窗口化 PDF 混合管线（核心）

长文档按 64 页窗口（环境变量 `MINERU_PROCESSING_WINDOW_SIZE` 可调）分段处理，每窗口经历：渲染页图 → PP-DocLayoutV2 批量版面 → 表格方向检测 → 文本模式原生表格优先合并 →（按档位分流）→ OCR/公式/文本回填 → 印章 OCR → 视觉裁图挂载（`mineru/backend/analysis/pdf/window.py` 的 `_prepare_pdf_window`/`_finish_pdf_window`）。

四个档位对应内部 effort（flash/medium/high/xhigh，映射见 `mineru/integrations/docvortex.py`）：

- flash：跳过 VLM，layout 块直接由本地 OCR 填充正文与表格（`_process_flash_ocr`）。
- medium：小模型全流程（layout + OCR + 公式识别）。
- high：Hybrid——layout 结果拆分为"原生表格接受区"与"VLM 区"，VLM 只解析复杂块，原生表格结果最后回插（`_split_native_high_table_blocks`、`_restore_native_high_table_blocks`）。
- xhigh：VLM 主导 + 图片分析，layout 仅用于标题切分与块补充。

窗口编排关键代码（`mineru/backend/analysis/pdf/window.py`）：

```python
def _inference_options(state, effort, parse_mode, image_analysis):
    options = {"images": state.images_pil_list,
               "image_analysis": image_analysis if effort == "xhigh" else False}
    if effort == "high":
        options["blocks_list"] = state.high_vlm_blocks
    if parse_mode == "txt":
        options["not_extract_list"] = NOT_EXTRACT_TYPES
    return options
```

### 5.3 模型层与硬件自适应运行时

`HybridLocalModelContextSingleton` 以单例持有全部小模型（layout/MFR/OCR/表格三件套/方向分类），通过 `acquire_document`/`release_document` 引用计数管理设备缓存，最后一个文档退出才回收显存（`mineru/model/runtime/hybrid.py`、`pipeline.py` 的 `_close_analysis`）。`mineru/model/runtime/` 是独立亮点：hardware_probe/onnx_calibration/tier_recommendation 等 27 个模块实现跨 Intel/AMD/NVIDIA/Apple Silicon 的设备探测、ONNX Execution Provider 自动选择与性能校准，支撑 README 宣称的"核心推理约 2–3 倍 CPU 速度"自动加速。

VLM 引擎选择逻辑（`mineru/model/vlm/selector.py`）：macOS/CPU 固定 llama.cpp；Linux+GPU 且安装了 vLLM 用 vLLM；否则尝试 LMDeploy（排除 Intel XPU）；兜底 llama.cpp。

### 5.4 本地文档库与 Agent 阅读

`mineru/doclib/` 实现一个常驻本地服务（`mineru server start`）：ingest/scan/parse_queue/parse_execution/search 等服务模块围绕 SQLite 组织，解析结果带页/块定位符（`doc:{short_id}/tier:standard/page:1`，见 `README.md` SDK 示例），支持续读、失效与遗忘（CLI 的 invalidate/forget/cleanup 命令，`mineru/cli/commands/`）。Python SDK `DoclibClient` 是稳定接口面（`mineru/doclib/__init__.py`）。

## 6. 构建与运行

来自 `README.md` 与 `pyproject.toml` 的实际命令：

```bash
# 安装（推荐 uv，虚拟环境）
uv pip install -U "mineru>=4.0,<5"        # NVIDIA 高吞吐部署用 mineru[full]

# 无状态解析（全页）
mineru-kit parse document.pdf -o document.md --tier standard

# 文档库模式（默认前 10 页，定位符续读）
mineru parse document.pdf --json
mineru server start                        # 常驻服务 + DoclibClient SDK

# 其他入口（pyproject [project.scripts]）
mineru-api / mineru-webui / mineru-router / mineru-openai-server / mineru-models-download

# 测试（pytest，pyproject 定义 testpaths=tests）
pytest -m "not slow and not accelerator"   # 按 marker 分层：torch/e2e/smoke/remote/full_stack 等
```

## 7. 测试与工程质量

- 测试规模：`tests/unittest/` 199 个测试文件（另含 conftest 与 15 个测试工具模块），覆盖 API 契约、CLI 契约、i18n、后端架构约束、各格式解析；另有 `tests/benchmarks/`、`tests/browser/`、`tests/ci/`（`tests/` 目录统计）。
- pytest marker 体系精细：torch/accelerator/e2e/smoke/slow/remote/full_stack/isolated_home/lifecycle 九类，区分 CPU CI 回归与需要真实设备的用例（`pyproject.toml` `[tool.pytest.ini_options]`）。
- CI：`python-package.yml` 构建 wheel + PyPI 发布 + 供应链 provenance，3.11–3.14 四版本矩阵验证安装；`cli.yml` 多 OS 测试矩阵 + 独立 Torch CPU 回归任务 + ruff/packaging 静态检查（`.github/workflows/`）。
- 代码质量：ruff line-length 128 并启用 ANN（类型注解强制）；覆盖率配置排除纯入口文件（`pyproject.toml` `[tool.coverage]`）。源码统一中文 docstring，loguru 结构化日志 + `stage_timer` 分阶段计时贯穿管线（如 `pipeline.py` 的 `pdf.prepare/pdf.layout/pdf.cleanup`）。
- 观察到的工程习惯：异常路径的资源清理写得非常防御（`window.py` `_prepare_pdf_window` 的 try/except/finally 三层关闭图片与快照缓存）。

## 8. 社区与维护状态

- 维护强度极高：GitHub API 显示最近 push 为 2026-10-30；releases 显示 4.1.0（10-29）、4.1.1（10-29）、4.1.2（10-30）、4.1.3（10-30）三天四发，之前 4.0.x 系列自 10 月上旬持续迭代。
- 社区规模：约 104 位贡献者（GitHub contributors API 分页推断）、6849 forks、watchers 279；open issues 仅 11 个，配合 issue 关闭徽章说明维护方响应极快。
- 背书与生态：OpenDataLab（上海AI实验室）官方项目，提供 mineru.net 云服务、HuggingFace/ModelScope 在线 Demo、Discord/微信社区；4.1 引入带 break change 的统一 Tasks API 并提供迁移指南（`README.md`）。
- 未确认：贡献者中核心成员占比、CI 平均时长（tarball 无 git 历史，未验证）。

## 9. 亮点与风险

**亮点**

1. 小模型 + VLM 的四档混合管线设计务实：flash 档纯 ONNX CPU 可跑，xhigh 才动用 VLM 与图片分析，成本与质量按需伸缩（`mineru/backend/analysis/pdf/window.py`）。
2. 全平台硬件自适应推理层罕见地完整：设备探测、EP 自动选择、性能校准、档位推荐独立成 27 个模块，覆盖 Intel/AMD/NVIDIA/Apple 集显独显（`mineru/model/runtime/`）。
3. Agent-first 产品化超前：README frontmatter 即 skill 声明、内置可安装 skill、定位符续读与稳定引用、结构化输出契约，围绕"让编码 Agent 读文档"全链路设计（`README.md`、`skills/mineru/`、`mineru/doclib/`）。
4. 质量工程扎实：199 个测试文件、九类 pytest marker 分层、多 OS/多版本 CI 矩阵、wheel 供应链 provenance。
5. 文档协议与渲染解耦：统一 MiddleJson/ModelJson 契约（托管在 DocVortex）+ 9 种渲染目标，输出扩展成本低。

**风险/局限**

1. 许可证为 Apache 2.0 加附加条款：MAU 超 1 亿或月收入超 2000 万美元需商业授权，对外提供在线服务须显著署名 MinerU（`LICENSE.md`），大厂采用前需法务评估。
2. 4.x 是破坏性大重构（HTTP API 全换、Python 版本要求升至 3.11+），旧教程与 3.x 生态（含 magic-pdf 时代资料）大量失效，网上经验迁移成本高（`README.md` Upgrade note）。
3. 依赖面大且迭代极快：gradio/fastapi/vLLM 等重型依赖、三天四个 release，下游 pin 版本与文档同步压力大（`pyproject.toml`、releases）。
4. 默认安装不含 Torch/vLLM，Windows 上 `mineru[full]` 还需另装 GPU 版 torch，NVIDIA 高性能路径的安装体验仍有摩擦（`README.md` TIP）。

## 10. 总结评价

MinerU 是目前开源文档解析领域工程化程度最高的项目之一：8 万+ star 背后不是单一模型，而是"专用小模型管线 + 可插拔 VLM + 硬件自适应运行时 + 本地文档库 + 多格式渲染"的完整产品栈，且在 2026 年保持三天四发的迭代强度。它已经从"PDF 转 Markdown 工具"转向"Agent 的文档基础设施"，这个定位在同类项目（marker、docling、pdf-extract 等）中独树一帜。适合需要批量高质量文档转 LLM 语料的数据团队、想让编码 Agent 直接读论文/财报/手册的开发者，以及研究多模态文档解析管线工程实现的工程师——后者尤其值得精读 `mineru/backend/analysis/pdf/window.py` 与 `mineru/model/runtime/` 的资源治理设计。需要注意其自定义许可证条款与 4.x 快速演进带来的版本跟随成本。
