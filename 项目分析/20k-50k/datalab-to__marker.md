# datalab-to/marker 项目分析

> 星数 38500（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：基于"文本层 + 轻量 ONNX + VLM"混合管线的文档转 Markdown 高精度转换工具，datalab 公司（Vik Paruchuri）出品，是 Surya OCR 的上层应用。

| 项 | 值 |
|---|---|
| 名称 | datalab-to/marker（PyPI 包名 `marker-pdf`，版本 2.2.0，见 `pyproject.toml`） |
| 星数 | 快照 38500（gh api 实测 2026-11-14 为 40902，仍在快速增长） |
| 语言 | Python 3.10+（130 个 py 文件，marker 包约 17.8k 行） |
| 许可证 | 代码 Apache-2.0（`LICENSE`）；模型权重为修改版 AI Pubs Open RAIL-M（`MODEL_LICENSE`） |
| 默认分支 | master（gh api 确认） |
| 最近活跃 | 最后 push 2026-11-13；最新 release v2.2.0 发布于 2026-11-12（分析时点前 2 天） |
| 主要作者 | VikParuchuri（794 commits），iammosespaulr（264）、tarun-menta（187） |
| Forks / Open issues | 2955 / 82（gh api） |

## 2. 项目是什么，解决什么问题

Marker 解决"PDF（及图片、DOCX、PPTX、XLSX、HTML、EPUB）到 Markdown/JSON/HTML/Chunks 的高保真转换"问题：保留标题层级、表格结构、LaTeX 公式、代码块、图片与阅读顺序，同时剥离页眉页脚等噪声（`README.md`、`marker/converters/pdf.py`）。它是 RAG 数据 prepared pipeline 的上游刚需工具，与 MinerU、docling、PaddleOCR PP-Structure、各类 OCR VLM 直接竞争。

v2.x 架构的核心思路是**混合分层路由**而非"全页大模型"：数字 PDF 先走内嵌文本层（pdftext），干净的拉丁文页面完全不调 GPU；版面与阅读顺序用约 18MB 的 Indra ONNX 模型在进程内完成；只有公式、复杂表格、CJK/非拉丁文、扫描页才路由给 Surya 2.5 VLM（0.69B，经本地 vLLM/llama.cpp 推理服务）。README（`README.md` #How it works）明确阐述该策略："It only calls the VLM where necessary"。

四种模式（`README.md` Usage、`marker/converters/pdf.py: mode` 注解）：
- `no_ocr`：纯文本层 + Indra ONNX 版面，零 OCR 调用（57 pg/s，适合批量数字 PDF）
- `fast`：CPU/MPS 默认，进程内 Indra ONNX 版面 + 行级 OCR（23.6 pg/s）
- `balanced`：GPU 默认，智能路由——干净页走 CPU，复杂页走 Surya VLM（16.9 pg/s）
- `vllm`：每页全量 Surya VLM，精度最高（85.9 OmniParseBench，4.5 pg/s）

另支持 `--use_llm` 把 Gemini/Claude/OpenAI 等 LLM 接入管线，用于跨页表格合并、行内数学、表单取值、图片描述等（`marker/processors/llm/` 下 10 个处理器、`marker/services/` 下 7 个服务适配器）。

关于任务提示中的 Texify：当前代码库（v2.2.0）中 grep 不到任何 texify 引用（`marker/`、`pyproject.toml`、`README.md` 均无），公式识别已改由 Indra ONNX 数学权重（`marker/ocr/indra.py` 中 MATH_FILES）与 Surya VLM 承担，`EquationProcessor` 只做 pdftext 页面上公式块的 OCR 编排与后处理（`marker/processors/equation.py`）。Texify 已退出当前架构（早期版本使用情况未在本次源码中确认）。

## 3. 技术栈与依赖

（出处：`pyproject.toml`）

| 类别 | 依赖 | 用途 |
|---|---|---|
| 运行时 | Python >=3.10，torch>=2.7.0 | 小型本地模型设备选择（cuda/mps/cpu，`marker/settings.py`） |
| 同门模型 | `surya-ocr>=0.23,<0.24` | Surya 2.5 VLM OCR + 推理管理器（spawn vllm/llama.cpp 服务，`marker/models.py`） |
| 同门模型 | `indra-ocr>=1.1.2,<2` | 进程内 ONNX 版面/阅读顺序/行级 OCR（约 18MB 权重） |
| PDF 解析 | `pdftext>=0.7,<0.8` | 内嵌文本层抽取（同为 datalab 出品） |
| 后处理 | `ftfy`、`rapidfuzz`、`regex`、`markdownify`、`markdown2` | 文本修复、模糊匹配、HTML→Markdown 渲染 |
| 结构 | `pydantic v2` + `pydantic-settings` | 全库配置/块模型均用 Annotated pydantic 字段 |
| LLM 接入 | `google-genai`、`anthropic`、`openai` | 三类 SDK 覆盖 Gemini/Claude/OpenAI 兼容端点 |
| 输入格式扩展 | `[full]` extra：mammoth(docx)、openpyxl、python-pptx、ebooklib、weasyprint | 非 PDF 格式支持 |
| CLI/服务 | `click`（CLI）；FastAPI/uvicorn 在 dev 组（`marker_server` 运行需另装） | 命令行与本地 API 服务 |
| 构建 | hatchling + uv.lock | 打包与锁定 |

## 4. 目录结构与代码组织

单包单体仓库，`marker/` 为唯一主包，按管线的六类角色分包（`README.md` #Internals 明确给出该分类）：

| 目录 | 职责 |
|---|---|
| `marker/converters/` | 三个端到端管线：`pdf.py`（主）、`table.py`（仅表格）、`ocr.py`（仅 OCR） |
| `marker/providers/` | 输入适配：pdf/image/document(docx)/spreadsheet(xlsx)/powerpoint/epub/html，`registry.py` 用 filetype 魔数识别选 provider |
| `marker/builders/` | 构建初始文档块：document（编排）、layout（版面）、line、ocr（OCR 路由）、structure（组块） |
| `marker/processors/` | 28 个默认后处理器 + `llm/` 子目录 10 个 LLM 处理器 |
| `marker/renderers/` | 输出渲染：markdown（基于 markdownify）、html、json、chunk（RAG 扁平化）、ocr_json |
| `marker/schema/` | 约 28 种块类型（`schema/blocks/`、`schema/groups/`、`schema/text/`），Document→Page→Block 树 + polygon |
| `marker/services/` | 7 个 LLM 服务适配器（gemini 默认、vertex、claude、openai、azure_openai、ollama、openrouter） |
| `marker/ocr/indra.py` | Indra ONNX 封装：权重下载、标签映射（Indra→Surya canonical）、版面检测 |
| `marker/scripts/` | CLI 入口（convert/convert_single/server/viewer/streamlit_app） |
| `marker/skills/SKILL.md` | 自带 AI coding agent skill 文档（`marker_single --install_skill` 安装） |
| `benchmarks/` | 可复现基准 harness：inference.py、summarize.py、plot_benchmarks.py |
| `tests/` | pytest 测试（43 文件） |
| `examples/` | Modal 云端部署示例 |
| `data/` | 示例文档与 README 配图 |

组织方式评述：依赖注入风格统一——`BaseConverter.resolve_dependencies` 按类型签名自动装配 builder/processor/renderer/service（`marker/converters/pdf.py`），processors 列表可被 `--processors` 完全替换，扩展点清晰（自定义 provider/renderer/processor 各有明确插槽）。

## 5. 核心架构与关键实现

### 5.1 主管线 PdfConverter（`marker/converters/pdf.py`）

`build_document` 串起全流程：provider 打开文件 → DocumentBuilder（协调 layout/line/ocr 四个 builder）→ StructureBuilder 组块 → 28 个 processor 依序处理 → renderer 输出。模式解析在构造器中完成：

```python
if not config.get("mode"):
    if config.get("disable_ocr"):
        config["mode"] = "no_ocr"
    else:
        config["mode"] = (
            "balanced" if settings.TORCH_DEVICE_MODEL == "cuda" else "fast"
        )
```

`vllm/surya/accurate` 模式会被翻译为 `force_ocr=True`，即"全页 VLM"是 balanced 语义的极限情形，而非独立管线。

### 5.2 OCR 路由（`marker/builders/ocr.py`，750 行，管线中最重的决策点）

OcrBuilder 按模式选择识别模型：`fast`/`indra` 模式下若满足条件（PDF、非强制 OCR、无 layout 覆盖）直接走 `convert_pdf_with_indra` 全页 Indra OCR，整页一次调用并在 balanced 模式下对非拉丁文结果回退 `text_extraction_method="surya"`（交 VLM 重做）。全页 VLM 路径每页单次调用（"tight block crops lose context"，类 docstring），失败页自动降级为逐块 OCR 重试。数字化表格优先用 CPU 启发式 + OrderNet table heads 从文本层重建，低置信度才回落 VLM（`README.md` #Table Conversion）。

### 5.3 模型获取机制（`marker/ocr/indra.py`、`marker/settings.py`、`marker/models.py`）

- Indra ONNX 权重：从 HuggingFace 逐文件下载（`https://huggingface.co/{HF_REPO}/resolve/{HF_REVISION}/{fname}`，indra.py 约 148 行），含 rec/order/orient/script 等 onnx 与 lm 词表 npz，落本地缓存目录，支持 HF_TOKEN。
- Surya VLM：不直接加载 torch 权重，而是由 `SuryaInferenceManager` 首次使用时自动 spawn 推理服务（NVIDIA 用 Docker+vLLM，CPU/Apple Silicon 用 llama.cpp 的 llama-server），可用 `SURYA_INFERENCE_URL` 指向既有服务；多 worker 共享一个服务、父进程按服务容量预算并发（`README.md` Throughput 节）。模型权重下载由 surya 侧管理（本仓库未直接可见，未确认细节）。
- 渲染字体 GoNotoCurrent-Regular.ttf 运行时从 `https://models.datalab.to/artifacts`（settings.ARTIFACT_URL）下载到 XDG 缓存目录，刻意不放进 site-packages。
- `marker/models.py` 的 `_ensure_surya_cache_writable` 对只读 HOME 容器做缓存目录打补丁，可见生产部署打磨。

### 5.4 LLM 增强处理器（`marker/processors/llm/`）

以 `LLMTableProcessor` 为例（llm_table.py）：表格超 60 行自动分批、175 行上限、旋转表格检测、最多 2 轮重写迭代，prompt 要求模型对比图片与 HTML 输出修正稿，仅允许有限的 HTML 标签白名单。9-11 个 LLM 处理器（table/merge/form/complex/image_description/equation/handwriting/mathblock/sectionheader/page_correction）默认全部挂在 `PdfConverter.default_processors` 里，仅在 `use_llm` 且注入了 llm_service 时才实际生效（pdf.py 构造器把 service 放进 artifact_dict）。

### 5.5 渲染层（`marker/renderers/markdown.py`）

HTMLRenderer 先把块树渲染成 HTML，再由继承 markdownify 的转换器转 Markdown；处理连字符跨行（`_HYPHENS_*_RE`）、C0 控制字符清理、`$` 转义、重复空行压缩等细节。输出格式四选一：markdown/json（块树 + content-ref）/html/chunks（RAG 扁平列表，含整块 HTML）。

## 6. 构建与运行

（出处：`pyproject.toml` [project.scripts]、`README.md` Installation/Usage、`pytest.ini`）

```bash
pip install marker-pdf            # 基础安装（torch 需按官方渠道装）
pip install "marker-pdf[full]"    # 加 DOCX/PPTX/XLSX/EPUB 支持

marker_single file.pdf --output_format markdown   # 单文件
marker /path/to/folder --workers 4                # 批量（--page_range/--force_ocr 等）
marker_single file.pdf --use_llm                  # LLM 增强（默认 Gemini，需 GOOGLE_API_KEY）
marker_gui --port 8000                            # 内置交互式 Web viewer（无额外依赖）
marker_server --port 8001                         # FastAPI 服务：POST /marker 与 /marker/upload，/docs 看 OpenAPI

# 开发
uv sync --frozen --group dev --extra full
uv run pytest                     # GPU runner 全量；CPU 子集带 `cpu` marker
```

Python API：`PdfConverter(artifact_dict=create_model_dict())`，`converter(path)` 返回 pydantic rendered 对象，`text_from_rendered` 取正文/图片（`README.md` #Use from python）。balanced/vllm 模式首次运行会自动 spawn Surya 推理服务，需预装 Docker+NVIDIA Container Toolkit 或 llama.cpp。

## 7. 测试与工程质量

- 测试规模：43 个测试文件约 2702 行，按 builder/processor/provider/renderer/schema/services/security 分目录（`tests/`）；pytest marker 分 `integration`（真实页面端到端质量检查，慢、跑 VLM）与 `cpu`（无 GPU 子集）（`pytest.ini`）。
- CI（`.github/workflows/ci.yml`）：两个 job——自托管 `t4_gpu` runner 跑全量（含 HF_TOKEN、VLLM_DTYPE=float16 兼容 Turing 卡），ubuntu-24.04 CPU runner 跑 `cpu` 子集覆盖无 GPU 路径；另有 `publish.yml`（发版）、`cla.yml`（CLA 机器人）、`scripts.yml`。
- 代码质量观察：全库 pydantic Annotated 配置字段（CLI help 自动生成，`marker/config/crawler.py`）；类型注解完整；server.py 有路径穿越防护（realpath + allowed root 校验）、扩展名白名单、100MB 上传上限，tests/security 有专门测试；uv.lock 锁定依赖。整体工程质量在同类开源文档工具中属上乘。

## 8. 社区与维护状态

- 维护极度活跃：最后 push 2026-11-13（分析前 1 天），v2.2.0 发布于 2026-11-12，2.x 迭代节奏快（gh api）。
- 贡献者结构：VikParuchuri（datalab 创始人）794 commits 主导，前五贡献者均 datalab 相关（iammosespaulr、tarun-menta），外部长尾贡献经 CLA 机制收敛（`.github/workflows/cla.yml`、`CLA.md`）——典型"开源核心 + 公司研发"模式。
- 生态位：datalab-to org 下姊妹项目 surya（VLM OCR）、indra-ocr、pdftext、chandra（托管 API 用的更强模型）、omni_parse_bench（自建基准），marker 是消费端旗舰。
- 社区渠道：Discord（README 徽章）；商业化入口 datalab.to（每月约 5000 免费页）。

## 9. 亮点与风险

**亮点**
1. 混合路由架构是最大差异化：干净数字页零 GPU 成本，VLM 只花在刀刃上，同时给出 CPU-only（fast/no_ocr）可用路径，覆盖"笔记本到多 GPU"全硬件谱系（`README.md` 模式表）。
2. 工程化程度高：类型安全的块 schema、可替换 processor 列表、依赖注入、内置 viewer/API server、自带 agent skill（`marker/skills/SKILL.md`）、Modal 部署示例，可直接嵌生产。
3. 基准可复现：`benchmarks/inference.py`/`summarize.py`/`plot_benchmarks.py` 随仓提供，声明"Everything above is reproducible"。
4. 商业边界清晰：代码 Apache-2.0 无 GPL 污染，模型权重限制单列在 `MODEL_LICENSE`，免费档覆盖初创（<$5M 融资/营收）。

**风险/局限**
1. **旗舰基准 OmniParseBench 为自家出品**（datalab-to/omni_parse_bench）：16,288 条 pass/fail 测试由竞品方设计，且"只输出 Markdown、无块级布局标注的系统在 layout/table 家族直接计 0 分"（README 脚注），规则天然利好 marker 这类带结构标注的管线；表内 Datalab 托管 API（94.1）居首也带明显营销导向。olmOCR-bench（allenai 第三方）数据可作交叉参照，但同为 README 精选呈现。横向对比数据应按"厂商自证"定位采信。
2. 模型链路重、部署门槛升：balanced/vllm 需 spawn 外部推理服务（vLLM 需要 Docker），相对 v1 时代的纯 torch 权重加载复杂度明显上升；模型/字体从 HF 与 models.datalab.to 运行时拉取，离线/内网环境需自行准备。
3. 模型权重许可（修改版 Open RAIL-M）含使用限制与 <$5M 营收/融资免费门槛，超限商用需购买——代码自由不等于模型自由，企业选型需过法务。
4. 复杂嵌套表格与表单仍是已知弱项（README Limitations 节自认），需 `--use_llm` 补救。

## 10. 总结评价

Marker 是"文档智能"赛道工程化完成度最高的开源转换管线之一：v2.x 用 18MB 级 ONNX 版面/OCR 模型 + 0.69B VLM 的混合路由，把"精度-吞吐-硬件成本"三角做到了帕累托前沿（厂商自证，方法学可复现），代码 Apache-2.0、质量上乘、迭代极快。它非常适合需要批量把 PDF/Office/EPUB 变成 LLM 友好 Markdown/Chunks 的 RAG 与数据管线团队，也适合读源码学习"规则 + 小模型 + VLM 分层路由"的实用系统设计；但引用其精度对比时应注意基准为厂商自建，商用重负载或超高精度需求会被自然导流到 datalab 托管 API（Chandra 模型），模型权重的 RAIL-M 限制是成本核算的一部分。
