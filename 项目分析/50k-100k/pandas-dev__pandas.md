# pandas-dev/pandas 项目分析

> 星数 50316（快照 2026-10-24）｜主语言 Python（含 Cython）｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：Python 生态事实上的标准数据分析库，提供 DataFrame/Series 两种带标签的数据结构，覆盖清洗、变换、聚合、时间序列与 IO 全流程。

| 项目 | 内容 |
| --- | --- |
| 名称 | pandas-dev/pandas |
| 星数 / Fork | 50316 / 20707（gh api，2026-11-04） |
| 主语言 | Python + Cython（`pyproject.toml` classifiers 声明 Cython） |
| 许可证 | BSD-3-Clause（`LICENSE`） |
| 默认分支 | main，最近 push 2026-11-03（gh api） |
| 最新 release | v3.1.0，2026-10-22（gh api releases/latest） |
| 创建时间 | 2010-08-24（gh api） |
| 贡献者 | 约 4300+（contributors API 分页推算） |

## 2. 项目是什么，解决什么问题

pandas 自述为"快速、灵活、易用的 Python 数据分析工具包"，目标是让 Python 处理"关系型/带标签"数据像处理表格一样自然（`README.md` "What is it?" 一节）。它解决了 NumPy 裸数组难以表达的三个核心问题：列异构（一列字符串一列浮点）、标签对齐（按索引自动对齐运算而非按位置）、缺失数据处理（`NaN`/`NaT`/`NA` 统一语义）——这三点在 `README.md` Main Features 与 `pandas/core/generic.py` 的实现中均有体现。

代码层面，一切围绕两层抽象展开：上层是用户可见的 `Series`（`pandas/core/series.py`，10428 行）与 `DataFrame`（`pandas/core/frame.py`，20501 行），共同继承 `NDFrame`（`pandas/core/generic.py`，13097 行）；下层是 Block 存储模型——DataFrame 的列数据按 dtype 分组存进若干 `Block`，由 `BlockManager` 统一管理行/列索引与块的映射关系（`pandas/core/internals/managers.py`、`pandas/core/internals/blocks.py`）。

在数据结构之外，pandas 实现了完整的分析原语：split-apply-compare 式的 groupby（`pandas/core/groupby/`）、数据库风格 join/merge（`pandas/core/reshape/merge.py`）、透视/长宽转换（`pandas/core/reshape/pivot.py`、`melt.py`）、滚动窗口（`pandas/core/window/`）、重采样（`pandas/core/resample.py`），以及覆盖 CSV/Excel/JSON/Parquet/HDF5/SAS/SQL 等格式的 IO 层（`pandas/io/`，其中 CSV 的 C 解析引擎是 `pandas/_libs/parsers.pyx`）。

时间序列能力是 pandas 的传统强项，由专门的 Cython 模块族 `pandas/_libs/tslibs/`（timestamps、periods、offsets、nattype、np_datetime 等 15+ 个 .pyx/.pyi 文件）承载，比纯 Python 实现快一个量级。

当前 main 分支已进入 pandas 3.x 时代：要求 Python >= 3.11、NumPy >= 2.0.2 硬依赖、meson-python 构建（`pyproject.toml`），且 Copy-on-Write 语义已成为无开关的默认行为——`pandas/core/flags.py` 中 `Flags` 类仅剩 `allows_duplicate_labels` 一个标志，3.0 前的 `mode.copy_on_write` 开关已不存在。

## 3. 技术栈与依赖

- 语言：Python（约 1440 个 .py 文件）+ Cython（41 个 .pyx 文件，源码统计）
- 构建系统：meson-python（`pyproject.toml` build-system：`meson-python>=0.19.0,<1`、`meson>=1.2.3,<2`、`Cython>3.1.0,<4`），版本号由 versioneer 生成（`generate_version.py`）；C++ 侧通过 meson subprojects 引入 fast_float 与 xsimd（`subprojects/xsimd.wrap`、`subprojects/packagefiles/`）
- 硬依赖（`pyproject.toml` [project].dependencies）：`numpy>=2.0.2`（Python 3.14+ 要求 >=2.3.3）、`python-dateutil>=2.9.0`、Windows/Pyodide 下 `tzdata`
- 可选加速（[project.optional-dependencies]）：`pyarrow>=16.0.0`（Arrow 字符串/ArrowDtype 后端与 Parquet/Feather）、`bottleneck`/`numba`/`numexpr`（performance 组）、`scipy`/`xarray`（computation 组）
- IO 可选依赖按格式分组：excel（openpyxl/xlsxwriter/python-calamine 等）、hdf5（tables）、sql（SQLAlchemy 2 + 各驱动）、html（lxml/bs4）、iceberg（pyiceberg）等，均见 `pyproject.toml`
- 测试：pytest>=8.3.4 + pytest-xdist（`pyproject.toml` test extra）

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `pandas/core/` | 核心实现：frame/series/generic（NDFrame）、internals（BlockManager）、groupby、reshape、indexes、arrays、dtypes、ops、window、strings 等 |
| `pandas/_libs/` | Cython 扩展：algos、hashtable、groupby、join、index、interval、internals、parsers、tslibs/*、ujson、io.sas 等 |
| `pandas/io/` | 各格式读写器：parsers（CSV）、excel、json、parquet、orc、feather、sas、spss、stata、sql、pytables（HDF5）、iceberg |
| `pandas/tseries/` | 时间序列公开层：freqs、holiday、offsets 公共 API |
| `pandas/arrays/` | `pandas.arrays` 命名空间暴露的扩展数组 |
| `pandas/tests/` | 测试套件（1144 个 .py 文件，源码统计） |
| `pandas/_config/`、`pandas/errors/`、`pandas/util/` | 全局配置（options）、异常体系、内部工具 |
| `pandas/plotting/` | 绘图 API（matplotlib 后端经 entry-points 注册，见 `pyproject.toml`） |
| `LICENSES/`、`subprojects/`、`scripts/` | 第三方许可清单、meson 子项目（fast_float/xsimd）、生成脚本 |
| `pyproject.toml`、`meson.build`、`meson.options` | 工程/构建清单 |

注：codeload tarball 按 `.gitattributes` 的 export-ignore 规则省略了 `doc/`、`.github/`、`ci/`、`asv_bench/`、`web/`，本文相关结论（CI、文档）来自 gh api 元数据。

组织方式评述：单包（single-package）单体仓库，但内部分层非常清晰——Python 层（core/io）与 Cython 层（_libs）严格分离，`pandas/_libs/*.pyi` 为每个 .pyx 提供 stub，类型检查全覆盖（`pandas/_typing.py`、大量 `from __future__ import annotations`）。这是典型的"Python 编排 + C 级热点下沉"双轨结构。

## 5. 核心架构与关键实现

### 5.1 BlockManager：DataFrame 的存储引擎

DataFrame 不存"列的字典"，而是把同 dtype 的列打包成 2D 块。`BlockManager`（`pandas/core/internals/managers.py`）继承自 Cython 版基类：

```python
class BlockManager(libinternals.BlockManager, BaseBlockManager):
    """
    BaseBlockManager that holds 2D blocks.
    """
    ndim = 2
```

（`pandas/core/internals/managers.py`；Cython 侧为 `pandas/_libs/internals.pyx`，提供 `BlockPlacement`、`BlockValuesRefs` 与热路径。）

- 每个块 = 一个 2D ndarray 或 ExtensionArray + `mgr_locs`（该块覆盖的列位置，`Block.mgr_locs`，`pandas/core/internals/blocks.py:229`）
- 增删列/再分类操作以"块列表变换"实现；新插入列会打碎块布局，`consolidate()`（managers.py:781）负责把同 dtype 块合并，`is_consolidated`（managers.py:2005-2016）缓存判断结果
- 行切片按块逐个 take（`Block.take_nd`，blocks.py:1090），最终由 Cython `internals.pyx` 拼接

`Block` 基类（blocks.py:146）定义了所有逐块运算的协议：`apply`/`reduce`（blocks.py:350/361）、`setitem`/`putmask`/`where`（blocks.py:1180/1256/1323）、`fillna`/`interpolate`/`shift`/`quantile`（blocks.py:1433 之后），dtype 升格由 `coerce_to_target_dtype`（blocks.py:438）处理。这套设计让"一列一 dtype"的语义跑在"多列同 dtype 连续内存"的性能上。

### 5.2 groupby 引擎：拆分-应用-合并的编译内核

groupby 的执行分三层：

1. API 层 `DataFrameGroupBy/SeriesGroupBy`（`pandas/core/groupby/generic.py`，4351 行）
2. 编排层 `BaseGrouper`/`BinGrouper` + `DataSplitter`（`pandas/core/groupby/ops.py:635/1285/1451`）：把数据按分组码切分为 bin，并决定每个聚合走哪条后端
3. 内核层 `WrappedCythonOp`（ops.py:127）直接调度 `pandas/_libs/groupby.pyx` 中的特化内核

Cython 内核对每个聚合逐 dtype 特化（`group_sum`/`group_var`/`group_mean`/`group_rank`/`group_cummin`/`group_ohlc`/`group_quantile` 等，见 `pandas/_libs/groupby.pyx` 函数列表）。以 `group_sum`（groupby.pyx:721）为例：

```cython
def group_sum(
    sum_t[:, ::1] out,
    int64_t[::1] counts,
    ndarray[sum_t, ndim=2] values,
    const intp_t[::1] labels,
    const uint8_t[:, :] mask,
    uint8_t[:, ::1] result_mask=None,
    Py_ssize_t min_count=0,
    ...
) -> None:
    """
    Only aggregates on axis=0 using Kahan summation
    """
```

即单遍扫描 `labels` 数组、以 Kahan 补偿求和累积，支持 NA mask 与 min_count——浮点分组求和的数值稳定性由此保证。分组码本身来自 `pandas/_libs/hashtable.pyx`（`groupby.pyx` 的 factorize 前置）与 `groupsort_indexer`（`pandas/_libs/algos.pyx:216`）。numba 路线（`pandas/core/groupby/numba_.py`、`pandas/core/_numba/`）为自定义聚合提供 JIT 兜底。

### 5.3 merge 引擎：factorize + 有序 join

`pd.merge` 的主流程在 `pandas/core/reshape/merge.py`：入口 `merge()`（merge.py:155）→ `_MergeOperation`（merge.py:937）→ `get_join_indexers`（merge.py:2062）→ 核心是 `_factorize_keys`（merge.py:2873），它把左右键列编码为整数枚举，并为等值 join 启用哈希路径（docstring 明言 `how` 参数"Used to determine if we can use hash-join"）。排序归并/插值的热点下沉到 `pandas/_libs/join.pyx`：`inner_join`/`left_outer_join`/`full_outer_join`（join.pyx:24/112/211）与 `left/inner/outer_join_indexer`（join.pyx:355/468/570），`merge_asof` 对应 `asof_join_backward/forward/nearest`（join.pyx:725 之后）。`_AsOfMerge`（merge.py:2420）、`merge_ordered`（merge.py:506）、`_CrossMergeOperation`（merge.py:2304）覆盖有序合并与笛卡尔积特例。

### 5.4 Cython 基础设施与字符串/扩展数组

- 热点全家桶：`pandas/_libs/algos.pyx`（2153 行）提供 nancorr、pad/backfill、kth_smallest、groupsort_indexer 等通用数值算法；`hashtable.pyx` + `hashtable_class_helper.pxi.in` 以模板化 .pxi.in 生成逐 C dtype 的哈希表（factorize/unique/value_counts 的地基）
- 时间序列 Cython 族 `pandas/_libs/tslibs/`：`np_datetime.pyx` 与 NumPy datetime64 互操作，`offsets.pyx`/`period.pyx`/`strptime.pyx` 各司其职
- 字符串在 3.x 默认为专用 `str` dtype：`BaseStringArray`（`pandas/core/arrays/string_.py:406`）下有 numpy 存储 `StringArray`（string_.py:618）与 pyarrow 存储 `ArrowStringArray`（`pandas/core/arrays/string_arrow.py:91`），装了 pyarrow 自动优先（string_.py:205 前后逻辑）；另有 `ArrowDtype`（`pandas/core/dtypes/dtypes.py:2217`）与可空掩码 dtype 族 `BaseMaskedDtype`（dtypes.py:1638）

## 6. 构建与运行

来自 `pyproject.toml` / `meson.build`（未实际执行构建，按规范只静态阅读）：

```bash
# 安装构建后端与依赖后从源码构建（构建后端 mesonpy）
pip install meson-python "Cython>3.1" "numpy>=2.0" versioneer meson
pip install --no-build-isolation -e .

# 运行测试（test extra: pytest>=8.3.4, pytest-xdist>=3.6.1）
python -m pytest pandas -x -q
python -m pytest pandas/tests/frame --numprocesses auto   # pytest-xdist 并行
```

- 版本号生成：`generate_version.py`（versioneer 驱动，须与 `meson.build` 同级）
- `.pxi` 模板展开：`generate_pxi.py`
- C++ 依赖 fast_float/xsimd 由 `subprojects/*.wrap` 经 meson 拉取
- 官方 wheel 由 CI 的 Wheel builder 工作流产出（gh api workflows：`wheels.yml`）

## 7. 测试与工程质量

- 规模：`pandas/tests/` 下 1144 个 .py 测试文件（源码统计），覆盖 frame/series/indexes/groupby/reshape/io/tseries/plotting 每个子系统；`pandas/conftest.py` 提供全库共享 fixture（如各种 dtype 组合参数化）
- 风格：pytest 参数化驱动的单元测试为主，大量"同语义在 Series/DataFrame 两个入口各测一遍"的镜像用例；`pandas/_testing.py` 与 `pandas/_testing/` 提供断言工具
- CI（gh api `actions/workflows`）：Unit Tests、Code Checks、Wheel builder、Doc Build and Upload（含 Windows 文档构建）、CodeQL、OpenSSF Scorecard、zizmor 安全分析、Deprecations Bot（弃用 API 追踪）、Link Check、Copilot code review 等 20 条工作流——工程化与供应链安全配置在同类开源项目中属第一梯队
- 质量观察：Python/Cython 边界处全部有 `.pyi` stub（`pandas/_libs/*.pyi`），`pyproject.toml` 固定 Cython/NumPy 上限并注释了具体兼容性问题（如 numpy 2.5.0 段错误被显式排除，GH#66086）

## 8. 社区与维护状态

- 极活跃：最近 push 2026-11-03（gh api，即分析前一天）；v3.1.0 于 2026-10-22 发布，与 v3.0 系列衔接紧凑，release 节奏约半年一个大版本
- 治理：NumFOCUS 赞助项目（README 徽章），The Pandas Development Team 署名（`pyproject.toml` authors）
- 贡献者体量约 4300+（contributors API 分页推算，未逐页核实），开放 issue 约 2000（gh api open_issues_count，含 PR）
- 有专职的 issue/PR 机器人流程（Comment Commands、PR Issue Assignment Gate、unassign-inactive 等，gh api workflows）
- 依赖管理采用 pixi（`update-pixi-lock` 工作流，gh api）

## 9. 亮点与风险

亮点：

1. Block 存储模型把"列式连续内存"与"逐列异构语义"成功统一，`internals/managers.py` + `internals/blocks.py` 是理解 pandas 性能特性的钥匙
2. Python/Cython 分层干净：41 个 .pyx 全部带 stub，热点（take、groupby 内核、join、hashtable、tslibs、CSV 解析）尽数下沉，且逐 dtype 特化
3. `groupby.pyx` 的 Kahan 求和与逐聚合内核体现了对数值正确性的较真（`pandas/_libs/groupby.pyx:721`）
4. 测试规模与 CI/供应链安全体系（CodeQL、Scorecard、zizmor）堪称 Python 生态标杆
5. 3.x 主动做大手术：硬性 NumPy 2.0+、Python 3.11+、CoW 无条件默认、str dtype 默认——用破坏性换长期简化

风险/局限：

1. `frame.py` 20501 行、`generic.py` 13097 行的巨型文件使核心代码可读性与新人上手成本偏高
2. 单体单包结构下，BlockManager 块碎片化/consolidation 的隐性成本仍是用户性能问题的常见来源（设计上依赖 `consolidate()`，managers.py:781）
3. 版本迁移负担重：3.x 对 NumPy/Python/CoW 的硬性要求使旧环境升级需整体回归（`pyproject.toml` requires）
4. pandas 已到 5 万星成熟期，核心 API 增量趋缓，创新前沿（out-of-core、GPU）在生态外围（Dask/Polars 等），源码读者更多是为了兼容与性能调优而非学习新范式

## 10. 总结评价

pandas 是过去十五年 Python 数据分析栈的地基：以 BlockManager 为核心的列块存储、以 Cython 特化内核驱动的 groupby/merge/tslibs、以及覆盖所有常见格式的 IO 层，共同构成一套兼顾表达力与性能的完整实现，工程质量（测试、CI、类型 stub、供应链安全）在开源世界中属于最高水准。适合所有做数据工作的 Python 用户使用；而其源码最适合三类人读——想搞懂 pandas 性能黑盒（为何有些操作慢、CoW 下赋值语义如何变化）的重度用户，需要扩展 dtype/数组后端（ExtensionArray、ArrowDtype）的库作者，以及想研究"Python 编排 + Cython 热点"双轨架构的引擎开发者。
