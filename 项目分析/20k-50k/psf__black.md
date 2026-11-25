# psf/black 项目分析

> 星数 41700（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：Python 生态事实标准的"毫不妥协"代码格式化工具——用极少的配置项换取完全确定性的格式输出，类比 Go 的 gofmt / Rust 的 rustfmt。

| 事实 | 值 | 出处 |
|---|---|---|
| 名称/组织 | psf/black（Python Software Foundation 旗下） | `pyproject.toml`、gh api |
| 星数 | 41700（快照），forks 2989，open issues 255 | gh api repos/psf/black |
| 语言/许可证 | Python（含 fork 自 CPython 的 blib2to3 解析器）；MIT | `pyproject.toml`、`LICENSE` |
| 运行要求 | Python >= 3.11，自身输出支持到 3.15 语法 | `pyproject.toml`、`src/black/mode.py` |
| 最新版本 | 26.10.1（2026-10-10 发布），CalVer 年.月 节奏 | `CHANGES.md`、gh api releases |
| 最近活跃 | main 分支最近 push 2026-11-13（分析前一天） | gh api pushed_at |
| 运行时安全网 | 格式化后 AST 等价校验 + 二次格式化幂等校验 | `src/black/__init__.py`（assert_equivalent/assert_stable） |

## 2. 项目是什么，解决什么问题

Black 解决的是 Python 团队协作中"代码风格争论"问题。README 的核心承诺是：使用 Black 即代表放弃对手工格式细节的控制权，换回速度、确定性和免受 pycodestyle 纠缠（`README.md:15-20`）。它刻意把可配置项压到最少（行宽、字符串引号规范化、目标 Python 版本等寥寥几个），使任何项目经 Black 处理后的代码外观一致，从而让代码评审聚焦内容而非格式，并产生最小 diff（`README.md:21-24`）。

技术上，Black 是一个"解析-重渲染"式格式化器，而非基于正则/逐行修补的工具：先把源码用 fork 自 CPython lib2to3 的 blib2to3 解析为具体语法树（CST），再按统一规则重建全部空白与换行，"几乎不参考既有格式"（`docs/the_black_code_style/current_style.md:6-8`）。这保证了幂等性与可预测性。

它还是风格演进的治理样板：稳定风格逐年推进，实验特性放在 `--preview`，已知有 bug 的特性降级到 `--unstable`（见第 5 节）。2025-2026 年间项目完成了一次显著的底层现代化：tokenizer 从自带的 pgen2 实现迁移到新库 pytokens（`src/blib2to3/pgen2/tokenize.py:60-61`），并默认通过 mypyc 编译发布二进制 wheel 提升性能（`pyproject.toml` 中 hatch-mypyc 配置）。

生态位上，Black 是 `black` 这个"风格名"的定义者——isort 内置 `profile = "black"`、pre-commit hook、GitHub Action、Vim 插件、blackd HTTP 服务、Jupyter 集成一应俱全（`pyproject.toml`、`plugin/`、`action/`、`src/blackd/`）。Rust 实现的 ruff format 以"兼容 Black 风格"为卖点构成性能竞争，但本仓库文档未直接回应（仓库内对 ruff 的唯一痕迹是默认排除目录列表含 `.ruff_cache`，`docs/usage_and_configuration/the_basics.md:327`）。

## 3. 技术栈与依赖

- 语言与运行时：纯 Python，`requires-python = ">=3.11"`；发布时用 mypyc 将核心模块编译为 C 扩展（blackd、`__main__.py`、debug/files/report 等除外，`pyproject.toml` [tool.hatch.build.targets.wheel.hooks.mypyc]）。
- 核心运行时依赖（`pyproject.toml` dependencies）：click（CLI）、pathspec（gitignore 风格文件匹配）、platformdirs（缓存目录）、packaging、mypy-extensions、pytokens~=0.4.0（新 tokenizer 内核）。
- 可选 extras（`pyproject.toml` [project.optional-dependencies]）：`jupyter`（ipython + tokenize-rt，用于 notebook 格式化）、`d`（aiohttp，blackd 服务）、`uvloop`/`colorama`。
- 构建：hatchling + hatch-vcs（版本号来自 git tag）+ hatch-fancy-pypi-readme（README+CHANGES 拼接）；cibuildwheel 构建 Linux/Win/macOS 多平台 wheel。
- 质量工具链自举：mypy strict、isort（profile=black）、pytest + pytest-xdist、coverage（分支覆盖）、hypothesmith 模糊测试（`pyproject.toml` dependency-groups）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 | 证据 |
|---|---|---|
| `src/black/` | 格式化引擎主体（约 16.7k 行 Python，27 个模块） | `wc -l src/black/*.py` |
| `src/blib2to3/` | fork 自 CPython lib2to3 的 pgen2 解析器（Grammar.txt、pytree、pgen2/tokenize），tokenizer 已委托 pytokens | `src/blib2to3/pgen2/tokenize.py` |
| `src/blackd/` | 基于 aiohttp 的 HTTP 格式化服务（`black[d]`） | `src/blackd/__init__.py` |
| `tests/` | pytest 测试 + `tests/data/` 格式化语料库（401 个文件，1.4MB） | `tests/test_format.py` |
| `docs/` | MkDocs/Sphinx 文档源（black.readthedocs.io），含代码风格指南 | `docs/index.md` |
| `action/` + `action.yml` | 官方 GitHub Action 入口 | `action/main.py` |
| `plugin/`、`autoload/` | Vim 插件（black.vim） | 目录内容 |
| `scripts/` | release.py、fuzz.py、generate_schema.py、make_width_table.py 等维护脚本 | `scripts/` |
| `profiling/` | 性能剖析样例 | 目录存在 |
| `.github/workflows/` | 13 个工作流：test/lint/fuzz/diff_shades/prepare_release/pypi_upload/changelog/zizmor 等 | 工作流文件 |

组织方式评述：单包单体（src 布局），解析器（blib2to3）与格式化引擎（black）物理隔离但同仓发布；无 monorepo/多包复杂度。`src/black` 内部按职责切分清晰：CLI/编排（`__init__.py`）、解析（`parsing.py`）、行生成（`linegen.py`）、空白策略（`lines.py`）、括号与拆分优先级（`brackets.py`）、字符串变换（`trans.py`，2849 行最大模块）、缓存（`cache.py`）、配置（`mode.py`）。

## 5. 核心架构与关键实现

### 5.1 格式化主流水线（`src/black/__init__.py:1485-1547` `_format_str_once`）

一次格式化的路径：`decode_bytes`（编码/换行规范化）→ `lib2to3_parse`（blib2to3 解析为 CST）→ `detect_target_versions`/`get_future_imports`（从源码反推目标版本与语言特性）→ `normalize_fmt_off`（处理 `# fmt: off/on` 区域，`src/black/comments.py:28-30`，同时兼容 `# yapf: disable`）→ `LineGenerator.visit` 逐节点生成 `Line` → `EmptyLineTracker.maybe_empty_lines` 决定垂直空白 → `transform_line` 做超长行拆分 → 输出。运行时双重安全网：`assert_equivalent`（格式化前后 AST 等价，`__init__.py:1898`）与 `assert_stable`（二次格式化必须逐字节一致，`__init__.py:1931-1958`），出错时报"INTERNAL ERROR"引导用户提 bug——这是 Black 敢于"毫不妥协"的底气。

```python
# src/black/__init__.py:1529-1535（核心循环，有删减）
for current_line in line_generator.visit(src_node):
    block = elt.maybe_empty_lines(current_line)
    dst_blocks.append(block)
    for line in transform_line(
        current_line, mode=mode, features=split_line_features
    ):
        block.content_lines.append(str(line))
```

### 5.2 超长行拆分：括号优先级体系（`src/black/brackets.py` + `src/black/linegen.py`）

行内每个叶子经 `BracketTracker.mark` 记录括号深度并配对开闭括号；深度为 0 的"分隔符"按固定优先级登记（`brackets.py:30-52`）：COMPREHENSION_PRIORITY=20 > COMMA_PRIORITY=18 > TERNARY_PRIORITY=16 > LOGIC_PRIORITY=14 > STRING_PRIORITY=12 > COMPARATOR_PRIORITY=10 > 数学运算符 2-9 > DOT_PRIORITY=1。拆分时 `delimiter_split` 按最高优先级的分隔符切开；若行不含此类分隔符则走 `left_hand_split`/`right_hand_split`（`linegen.py:1086/1152`）在合适括号处断行。`transform_line`（`linegen.py:905-1027`）按条件编排变换器链：

```python
# src/black/linegen.py:962-973（括号内长行的变换器序列）
transformers = [
    string_merge, string_paren_strip, string_split,
    delimiter_split, type_ignore_comment_split,
    standalone_comment_split, string_paren_wrap,
    right_hand_split_with_omits,
]
```

字符串专项变换（合并/拆分/加括号包裹）独立成 2849 行的 `src/black/trans.py`（StringMerger、StringParenStripper、StringSplitter、StringParenWrapper 等 5 个 StringTransformer 子类），是全仓最大也是 bug 率最高的复杂区。行宽计算支持东亚宽字符：`src/black/_width_table.py` 内嵌由 `scripts/make_width_table.py` 从 wcwidth 0.2.14 / Unicode 17.0.0 生成的宽度表。

### 5.3 magic trailing comma 与行宽判定（`src/black/lines.py` + `src/black/mode.py`）

"魔法尾逗号"是 Black 少数尊重既有格式的例外：集合/调用/签名内若已有尾逗号，则强制展开为每元素一行；`Line.append` 在扫描时捕获 `magic_trailing_comma`（`lines.py:98-111`），并特别区分了语法必需的单元组/单元素下标尾逗号（不可删除）。垂直空白由 `EmptyLineTracker`（`lines.py:592`）统一裁定，从而抹平输入中的空行差异。行宽默认 88（`src/black/const.py` DEFAULT_LINE_LENGTH），判定入口是 `is_line_short_enough`（`linegen.py:943`），宽字符经上述宽度表折算。

### 5.4 风格分级治理：stable / preview / unstable（`src/black/mode.py:256-331` + `docs/the_black_code_style/future_style.md:784-804`）

约 30 个实验特性定义为 `Preview` 枚举（`mode.py:256-291`）。治理规则（future_style.md）：preview 中所有特性"预期进入下一年的稳定风格"；发现 bug 的特性降级为 unstable；`UNSTABLE_FEATURES = {string_processing, hug_parens_with_braces_and_square_brackets}`（`mode.py:294-299`）。`Preview.X in Mode` 语法实现分级开关：unstable 模式全开、preview 模式开全部除 unstable、用户可用 `--enable-unstable-feature` 精确点选（`mode.py:319-331`）。Black 自身源码就用 `unstable = true` 格式化（`pyproject.toml` [tool.black] 注释明言"想要免 bug 的格式化请勿开启"）。

### 5.5 解析层：blib2to3 派生解析器与 pytokens 内核（`src/black/parsing.py:44-69`）

`get_grammars` 按目标版本准备三套文法（3.0-3.6 基础文法、3.7-3.9 async 关键字文法、3.10+ 软关键字文法以支持 match/case），`lib2to3_parse` 依次尝试直至成功，最终用 CPython 原生 `ast` 做语义校验。blib2to3 目录结构保留 pgen2 全套（driver/pgen/grammar/parse），但 `tokenize.py` 已改为 `import pytokens` 的适配层——解析器文法仍是 lib2to3 血统，tokenizer 内核已换新，这是近年最大的架构动作，也把"lib2to3 无法解析新语法"的经典局限（如 f-string 嵌套、t-string）逐一化解（token.py 中已新增 FSTRING_*/TSTRING_* token 类型）。

### 5.6 缓存机制（`src/black/cache.py`）

批量格式化跳过未变更文件：缓存文件为 `user_cache_dir("black")/<版本>/cache.<mode哈希>.pickle`（`cache.py:25-47`），按 Mode 全部配置项生成缓存键（`mode.py:333-365`）——不同行宽/preview 开关互不命中。变更检测三级递进：先比 st_mtime/st_size，仅 mtime 变化时才算 SHA-256（`cache.py:103-117`）；写入用临时文件 + `os.replace` 原子替换（`cache.py:134-151`）。

## 6. 构建与运行

```bash
# 安装
pip install black                     # 基础
pip install "black[jupyter]" "black[d]"   # notebook / blackd

# 使用
black path/to/file.py                 # 格式化
black --check --diff src/             # CI 校验模式
black --preview src/                  # 预览风格
black --line-length 79 --target-version py312 .
blackd --port 45484                   # HTTP 服务（需 [d] extra）

# 开发与测试（来自 pyproject.toml / tox.ini）
pip install -e . -c requirements-dev.txt 2>/dev/null || pipx install hatch
hatch env create test && hatch run test:pytest tests/ -x -n auto
tox -e py                             # tox.ini [testenv] commands = pytest
tox -e fuzz                           # hypothesis 模糊测试
tox -e run_self                       # 用开发版 Black 格式化 Black 自身
hatch build                           # 构建（发布 wheel 默认经 mypyc 编译）
```

发布流程高度自动化：`scripts/release.py` + `.github/workflows/prepare_release.yml`、`cut_release.yml`、`pypi_upload.yml`；`changelog.yml` 强制 PR 在 CHANGES.md 留条目。

## 7. 测试与工程质量

- 测试语料库：`tests/data/` 共 401 个文件（1.4MB），其中 `cases/` 下 318 个 `.py` 用例文件；`tests/test_format.py:53-55` 用参数化把每个语料文件变成独立测试（输入/输出块对），另有 `line_ranges_formatted/` 专测 `--line-ranges`。`tests/test_black.py` 4610 行单元测试覆盖 CLI、缓存、文件发现、blackd、notebook 等。
- 测试矩阵：`tests/test_black.py` + `tox.ini` 覆盖 CPython 3.11-3.15 与 pypy3；pytest 标记体系区分 `no_blackd`/`no_jupyter`/`incompatible_with_mypyc`（`pyproject.toml` [tool.pytest.ini_options]），确保有无可选依赖、是否 mypyc 编译两种形态都通过。
- CI（`.github/workflows/`）：test.yml 多 OS × 多版本矩阵 + Coveralls；fuzz.yml 每日用 hypothesmith 对随机 Python 代码做格式化模糊测试（Python 3.11-3.15 矩阵）；diff_shades.yml 在 top PyPI 项目上跑格式化 diff 分析，用于评估风格改动的实际影响面；lint.yml（self-format + flake8 类检查）、zizmor.yml（CI 安全审计）。
- 质量观察：mypy strict 全量类型标注；发布前 release_tests 在编译版上重跑测试套件（cibuildwheel test-command）；工程化成熟度在 Python 生态属第一梯队。

## 8. 社区与维护状态

- 治理：仓库归属 psf 组织，创建者 Łukasz Langa（`AUTHORS.md:3`），列名维护者 10 人（Carol Willing、Carl Meyer、Jelle Zijlstra、Cooper Lees、Richard Si、Shantanu Jain 等，`AUTHORS.md:5-15`），含多位 CPython 核心开发者——这是 Black 能第一时间跟进新 Python 语法（如 3.14 t-string、3.15 lazy imports）的结构性原因（`src/black/mode.py:59-62` 已有 T_STRINGS、LAZY_IMPORTS 特性位）。
- 活跃度：main 分支最近 push 2026-11-13；open issues 255、forks 2989；贡献者列表（含匿名）≥100（gh api contributors，单页上限 100，实际规模未逐页确认）。
- 发布节奏（gh api releases 实测）：26.3.0（2026-03-06）→ 26.3.1（03-12）→ 26.5.0（05-16）→ 26.5.1（05-18）→ 26.10.0（10-04）→ 26.10.1（10-10），CalVer"年.月.序号"，年均 4-6 个版本，bugfix 版随行。当前 Unreleased 段已在做"drop 3.10 运行时 + 出 3.15 wheel"（`CHANGES.md:8-10`）。
- 基金会背景：PSF 旗下意味着中立治理与长期资金/基础设施保障，未采用 CLA 之外的特别治理文档（ CONTRIBUTING.md 为常规贡献指南；更细的治理机制未确认）。

## 9. 亮点与风险

亮点：

1. 确定性输出的工程保障体系教科书级：CST 重渲染 + 运行时 AST 等价与幂等双重断言（`src/black/__init__.py:1898/1931`），用户永远不会拿到语法损坏的输出。
2. 风格演进的三级治理（stable/preview/unstable + 年度稳定化承诺 + 特性降级机制）在"风格冻结"与"持续改进"间取得平衡，且有 `--enable-unstable-feature` 缓冲降级冲击（`docs/the_black_code_style/future_style.md:784-795`）。
3. 测试语料库 + 每日模糊测试 + diff_shades 全生态影响评估，风格改动的回归风险控制非常扎实（`tests/data/`、`.github/workflows/fuzz.yml`、`diff_shades.yml`）。
4. 持续的底层现代化：tokenizer 迁移 pytokens、mypyc 编译发布、跟进 t-string/lazy imports 等最新语法（`src/blib2to3/pgen2/tokenize.py:60`、`pyproject.toml`、`src/black/mode.py`）。
5. 生态绑定深：isort profile、pre-commit、GitHub Action、Vim 插件、blackd、notebook 支持，形成完整工具链。

风险/局限：

1. 解析器仍是 lib2to3 血统的 CST，新语法支持永远滞后于 CPython（靠三文法轮试 + fork 维护弥补），相对 ruff 的 Rust 实现，纯 Python 引擎在超大仓库上的格式化速度处于劣势（mypyc wheel 是缓解而非根治）。
2. trans.py 等"字符串变换"子系统复杂度高（单文件 2849 行、多个变换器交互），近年 CHANGES.md 的 Unreleased 段大量条目是 `# fmt: skip` 与字符串边界 crash 修复，说明该区域仍属高风险区。
3. 风格年度滚动对多团队大库是迁移成本：每年 1 月稳定风格变化会触发全库重新格式化 diff（black 官方以"一次接受"哲学缓解，但跨版本协作的项目需锁版本）。
4. 竞争格局变化：ruff format 以数量级速度差和"Black 兼容"策略分流新项目，Black 的差异化正从"格式化器本身"转向"风格权威 + 生态兼容基线"（此为行业观察，仓库内无直接论述）。

## 10. 总结评价

psf/black 是"少配置、强约定"设计哲学最成功的实践之一，也是 PSF 旗下基础设施级项目：它用 fork 的 lib2to3 解析器加约 1.7 万行精心分层（解析→行生成→括号/分隔符优先级→字符串变换→空白策略→缓存）的格式化引擎，把 Python 代码格式从"个人偏好"变成了"协议"，并以运行时等价/幂等断言、年度风格治理、每日模糊测试与全生态 diff 影响分析，构建了开源工具中罕见的正确性与演进保障体系。适合所有 Python 团队直接采用；对从事解析器、代码重构工具或"风格治理机制"研究的开发者，`src/black/linegen.py`、`brackets.py`、`trans.py` 与 `docs/the_black_code_style/` 是最佳阅读材料；而它近年的 pytokens 迁移与 mypyc 编译实践，也是观察"老牌纯 Python 工具如何现代化"的鲜活样本。
