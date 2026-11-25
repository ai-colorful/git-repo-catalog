# mingrammer/diagrams 项目分析

> 星数 41500（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-13

## 1. 项目速览

一句话定位：用 Python 代码声明式绘制云系统架构图的库（Diagram as Code），依托 Graphviz 渲染，内置 AWS/Azure/GCP/K8s 等 17 个提供商、约 2500+ 个官方风格图标节点。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | diagrams（PyPI 包名 `diagrams`） | `pyproject.toml` |
| 星数 | 41500（快照）；gh api 实测 42809（2026-11-13） | 任务参数 / gh api |
| 主语言 | Python（>=3.9），附 TypeScript playground | `pyproject.toml`、`playground/package.json` |
| 许可证 | MIT | `LICENSE`、gh api |
| 当前版本 | 0.25.1（2025-11-22 发布） | `pyproject.toml`、gh api releases |
| 最近活跃 | 默认分支 master 最近 push 2026-11-02 | gh api |
| 代码规模 | 核心包 257 个 .py / 约 1.4 万行（多数为生成代码），2540 个 PNG 图标 | 源码统计 |

## 2. 项目是什么，解决什么问题

画架构图通常依赖 Visio/draw.io 这类手工拖拽工具：改一版要重新对齐所有图标，且无法用版本管理追踪演进。diagrams 的答案是"Diagram as Code"——在 Python 里用 `EC2 >> RDS` 这样的表达式声明节点和流向，`with Diagram(...)` 退出时自动调 Graphviz 出图（`README.md`、`diagrams/__init__.py`）。因为图就是代码，架构变更可以进 Git review，这正是它对工程师群体的核心吸引力。

图标资源是它的第二重价值：仓库内置 2540 个 PNG 图标，覆盖 AWS(536)、Azure(879)、GCP(124)、IBM(181)、Alibaba Cloud(95)、OCI(141)、K8s(46)、On-Prem(173)、SaaS/编程语言/GIS/C4 等 17 个提供商目录（`resources/` 统计）。用户写 `from diagrams.aws.compute import EC2` 即可得到官方风格的节点，不用自己找图标抠图。

它明确不做 IaC：README 注明"不控制真实云资源、不生成 CloudFormation/Terraform 代码"，定位是原型设计与文档可视化。Apache Airflow 官方文档用它生成架构图（README "Who uses it" 节）。生态上有 Go 移植 go-diagrams、KubeDiagrams、airflow-diagrams 等下游项目。

最新的亮点是在线 Playground（diagrams.mingrammer.com/playground）：通过 Pyodide 在浏览器里运行真实 diagrams 包 + wasm 版 Graphviz，免安装即可写代码出图，支持补全/搜索/导出（`README.md`、`playground/`）。学习曲线因此进一步降低。

## 3. 技术栈与依赖

| 组件 | 用途 | 出处 |
| --- | --- | --- |
| Python ~=3.9 | 运行时（CI 矩阵测到 3.13） | `pyproject.toml`、`.github/workflows/test.yml` |
| graphviz 0.13-0.21 | 唯一重量级依赖，负责 dot 生成与调用本机 Graphviz 渲染 | `pyproject.toml`、`diagrams/__init__.py` |
| jinja2 | 图标→节点类代码生成的模板引擎 | `pyproject.toml`、`scripts/generate.py` |
| hatchling | 构建后端；wheel 只含 `diagrams` + `resources` 两个顶层目录 | `pyproject.toml` |
| Go 工具 `round`、inkscape、ImageMagick、black | 仅图标预处理流水线需要（贡献者侧） | `autogen.sh`、`DEVELOPMENT.md` |
| React 18 + CodeMirror 6 + Pyodide + @hpcc-js/wasm-graphviz + Vite + Playwright | 在线 playground（独立子项目） | `playground/package.json` |
| Docusaurus v1 | 官网文档站 | `website/siteConfig.js` |

设计上刻意保持运行时零冷门依赖：读 PNG 宽高判断竖版图标补边距时直接解析 PNG IHDR 字节头而不引入 Pillow（`diagrams/__init__.py` 的 `_portrait_icon_padding`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `diagrams/` | 包本体：`__init__.py` 是全部核心抽象（约 675 行）；`cli.py` 提供 `diagrams` 命令；17 个提供商子包 + `base/`/`custom/`/`c4/` |
| `resources/` | 2540 个 PNG 图标，按 `resources/<provider>/<type>/` 分层，wheel 随包分发 |
| `scripts/` | `generate.py`（Jinja2 生成节点模块与 API 文档）、`resource.py`（图标清洗/转格式/圆角）、`check_release_version.py` |
| `templates/` | `module.tmpl`、`apidoc*.tmpl` 代码生成模板 |
| `config.py` | 提供商清单、类名大小写修正表、别名表、外部命令参数 |
| `autogen.sh` | 图标预处理 + 代码生成总入口脚本 |
| `docs/` | 文档源：`guides/`（Diagram/Cluster/Node/Edge 四篇）+ `nodes/`（自动生成的 API 文档，如 azure.md 2736 行） |
| `website/` | Docusaurus v1 官网 |
| `playground/` | 浏览器端 playground（React + Pyodide worker + wasm graphviz，含 Playwright e2e） |
| `tests/` | unittest 测试（5 个文件、823 行） |
| `.github/workflows/` | 6 个 workflow：test、package、pre-commit、codeql、deploy-site、playground |
| `docker/`、`.devcontainer/` | 开发容器（内置 inkscape/imagemagick/round 等图标工具链） |

组织方式评述：单包单体结构，但"手写核心 + 生成模块"双层分明——`diagrams/__init__.py` 与 `c4/`、`custom/` 是手写，其余 257 个文件中的绝大部分由 `autogen.sh` 从 `resources/` 目录树自动生成（每个生成文件头注明 "DO NOT EDIT"，`diagrams/aws/compute.py`）。图标即数据源，代码即产物，这种"资产驱动代码生成"的组织是本项目最鲜明的工程特征。

## 5. 核心架构与关键实现

### 5.1 Diagram/Cluster 上下文栈：contextvars + with 协议

不靠传参、不靠线程局部分页变量，`diagrams/__init__.py` 用 `contextvars.ContextVar` 保存当前 Diagram 与 Cluster 上下文（`__diagram`/`__cluster`，第 67-90 行）。`with Diagram(...)` 进入时 `setdiagram(self)`，退出时渲染并清空；`Cluster` 在 `__enter__`/`__exit__` 间把自身压入上下文，退出时将自己内部的 Digraph 作为 subgraph 挂回父级（Cluster 或顶层 Diagram），实现任意嵌套：

```python
# diagrams/__init__.py Cluster.__exit__
def __exit__(self, exc_type, exc_value, traceback):
    if self._parent:
        self._parent.subgraph(self.dot)
    else:
        self._diagram.subgraph(self.dot)
    setcluster(self._parent)
```

节点实例化时只需 `getdiagram()`/`getcluster()` 探测当前上下文（`Node.__init__` 第 377-417 行），无上下文则抛 `EnvironmentError("Global diagrams context not set up")`。Cluster 按嵌套深度 `depth % len(bgcolors)` 轮换主题背景色，实现分层底色。用 ContextVar 而非全局变量保证了多线程各自成图的隔离。

### 5.2 Node 运算符重载：`>>` / `-` / `<<` 流向 DSL

流向语法完全由 `Node` 的六个反射运算符实现（`diagrams/__init__.py` 第 423-487 行）：`__sub__`（无向）、`__rshift__`（正向 `>>`）、`__lshift__`（反向 `<<`），以及列表场景的 `__rsub__`/`__rrshift__`/`__rlshift__`。支持 Node-Node、Node-[Nodes]、Node-Edge 三类操作数；Edge 对象先与左节点绑定，再链到右节点，属性沿途合并：

```python
# diagrams/__init__.py Node.__rshift__
def __rshift__(self, other):
    if isinstance(other, list):
        for node in other:
            self.connect(node, Edge(self, forward=True))
        return other
    elif isinstance(other, Node):
        return self.connect(other, Edge(self, forward=True))
    else:
        other.forward = True
        other.node = self
        return other
```

`Edge.attrs` 属性最终把 forward/reverse 组合翻译成 Graphviz 的 `dir=forward/back/both/none`（第 661-671 行）。`connect()` 把边写到顶层 Diagram 而非 Cluster（边属于全图，第 494-507 行注释）。设计上 `-`、`>>`、`<<` 与 `Edge(color=..., label=..., style=...)` 组合即可表达全部连线语义，这是该库 API 广受好评的部分。

### 5.3 渲染链路：Node → Digraph → Graphviz

节点实例化即入图：`Node.__init__` 通过 `_load_icon()` 解析图标绝对路径（相对于 diagrams 包父目录的 `resources/`，这正是 wheel 把 `resources` 放顶层的原因，见 `pyproject.toml` 注释），组装 `shape=none + image=<icon>` 属性，然后注册到当前 Cluster 或 Diagram 的 Digraph。竖版 PNG 会按 IHDR 头计算额外 padding，避免图标压住标签。`Diagram.__exit__` 触发 `self.dot.render(format=..., view=show)`，由 graphviz 库调用系统 Graphviz 二进制出图，并删除中间 dot 文件；`outformat` 支持列表一次出多格式（第 205-213、260-265 行）。`_repr_png_` 让 Jupyter 内联预览。

### 5.4 图标资产 → 代码的自动生成流水线

贡献者只需往 `resources/<provider>/<type>/` 放 PNG，`autogen.sh` 完成其余：① `scripts/resource.py` 按提供商清洗文件名（如 AWS 去掉 `Amazon-`/`@4x` 前后缀）、onprem/azure 用 inkscape、oci/ibm 用 ImageMagick 做 SVG→PNG、AWS 图标用 Go 工具 `round` 加圆角；② `scripts/generate.py` 遍历资源目录，用 Jinja2 渲染 `templates/module.tmpl` 生成 `diagrams/<pvd>/<typ>.py`（类名经 `config.py` 的 UPPER_WORDS/TITLE_WORDS/ALIASES 修正）与 `docs/nodes/<pvd>.md` API 文档；③ 图标同步到网站静态目录，最后 black 统一格式。生成模板极简：

```jinja2
# templates/module.tmpl
class _{{ pvd|up_or_title(typ) }}(_{{ pvd|up_or_title(pvd) }}):
    _type = "{{ typ }}"
    _icon_dir = "resources/{{ pvd }}/{{ typ }}"
{% for meta in metas %}
class {{ meta["name"] }}(_{{ pvd|up_or_title(typ) }}):
    _icon = "{{ meta["icon"] }}"{% endfor %}
```

三层继承（Node → 提供商基类 `_AWS` → 类型基类 `_Compute` → 图标类 `EC2`）让样式按提供商/类型两级收敛。`c4/` 是例外：不依赖图标，直接用 Graphviz HTML label 渲染 C4 模型的结构化卡片（`diagrams/c4/__init__.py` 的 `_format_node_label`）。

## 6. 构建与运行

```shell
# 用户侧：需先安装系统 Graphviz（brew install graphviz）
pip install diagrams

# 运行：图在 with 块退出时自动渲染
# diagrams 命令行可直接执行图脚本（console script -> diagrams.cli:main）
diagrams examples/cat_service.py

# 测试（CI 同款，Python 3.9-3.13 矩阵）
python -m pip install .
python -m unittest -v tests/*.py

# 贡献者：生成图标模块（需 inkscape/imagemagick/round/black）
poetry install && ./autogen.sh
# 或用官方容器：docker build -f docker/dev/Dockerfile . 后 docker exec 运行

# 网站/Playground
cd website && yarn install && yarn start      # Docusaurus
cd playground && npm install && npm run dev   # Vite；npm run e2e 跑 Playwright
```

出处：`README.md`、`.github/workflows/test.yml`、`pyproject.toml [project.scripts]`、`DEVELOPMENT.md`、`playground/package.json`。

## 7. 测试与工程质量

- 测试：`tests/` 5 个文件共 823 行，标准库 unittest。`test_diagram.py`（489 行）覆盖方向/曲线/主题/格式校验、全局上下文进出、Cluster 嵌套、Edge 方向组合与运算符；`test_packaging.py`（117 行）安装 wheel 后加载全部入口并逐一核对图标落盘——这是针对 0.25.0 空资源 wheel（#1099）和 CLI 未随包发布（#1149）两次事故补的回归防线（`.github/workflows/package.yml` 注释明确记载了这两次带病发布）。playground 另有 vitest 单测 + Playwright e2e（`playground/package.json`）。
- CI：6 个 workflow（`.github/workflows/`）。`test.yml` 五版本矩阵 + Graphviz 安装；`package.yml` 构建 sdist/wheel 并验证内容物（含防 EPIPE 误报的细节注释）；`codeql-analysis.yml` 安全扫描；`deploy-site.yml` 将文档站与 playground 一次性原子发布到 gh-pages。`pre-commit.yml` + `.pre-commit-config.yaml`（isort/autopep8）。
- 代码质量观察：核心手写代码注释密度高且解释"为什么"（如 Edge label 用 xlabel 导致错位的 issue 链接、strict 模式、PNG 头解析逐行说明）；生成代码由 black 统一。不足：CHANGELOG.md 自 0.17.0 后未再维护（仅 Unreleased 补了一条 gis 弃用说明，`CHANGELOG.md`）；主测试仍用 unittest 而 dev 依赖里装了 pytest，风格略分裂。

## 8. 社区与维护状态

- 活跃度：gh api 显示最近 push 2026-11-02（分析日前 11 天），open issues 约 400，fork 2730，项目创建于 2020-02-02。近一年的提交集中在工程质量修复（打包、portrait 图标 padding、主题系统等，可从 `pyproject.toml` 与 workflow 注释交叉印证），未见官方路线图文档（未确认）。
- 发布节奏：最新 release v0.25.1 于 2025-11-22 发布（gh api）；`scripts/check_release_version.py` + `tests/test_check_release_version.py` 校验发版一致性。
- 贡献者：81 人（gh api contributors 分页统计），单人项目（mingrammer）主导，图标类 PR 为主要外部贡献形态。
- 文档与社区资产：官网 diagrams.mingrammer.com（Docusaurus），四篇核心指南（`docs/guides/`）+ 自动生成的 19 个节点 API 页；Playground 2024-2025 年新增，README 已用作第一入口。

## 9. 亮点与风险

亮点：
1. 极小内核承载完整 DSL：`diagrams/__init__.py` 一个文件（约 675 行）实现 Diagram/Cluster/Node/Edge 全部抽象，contextvars 上下文 + 运算符重载的组合简洁且无魔法依赖。
2. 资产驱动的代码生成体系：2540 图标 → 清洗/转换 → Jinja2 生成 257 个模块与 API 文档，加新图标零手写代码，且文档与代码永远同步。
3. 图标库即护城河：17 个提供商、按 AWS 官方服务类目组织（27 个分类文件，`diagrams/aws/`），竞品难以短期复制这份持续维护的资产。
4. 事故驱动的工程加固：针对两次 PyPI 带病发布补了 `test_packaging.py` 与 `package.yml` 验证链，workflow 注释完整记录了事故根因，工程质量意识强。
5. Playground 体验创新：Pyodide + wasm Graphviz 让"pip install + 装 Graphviz"两大门槛在浏览器内消失。

风险/局限：
1. 硬依赖系统 Graphviz 二进制，不装无法出图，容器/Serverless 场景偏麻烦；渲染布局完全交给 dot，复杂大图的自动布局可控性有限。
2. `diagrams/cli.py` 直接 `exec()` 用户脚本属预期设计，但也意味着任何 `diagrams some.py` 都在执行任意代码。
3. 图标与云厂商产品目录存在天然滞后，需要贡献者人工跟进（400 个 open issue 中大量为图标请求，未逐条确认）；`autogen.sh` 依赖 inkscape/ImageMagick/Go 工具链，贡献图标的环境成本偏高。
4. 版本维护松散：CHANGELOG 停更于 0.17.0，0.18-0.25 只能靠 GitHub releases 追溯；wheel 因携带全部图标体积较大（仓库 tarball 约 72MB，主要是 `resources/`）。

## 10. 总结评价

diagrams 是"用正确的抽象打中小众刚需"的典范：把架构图从拖拽画布搬进 Python 代码，换来版本管理与代码评审能力；再用一座持续更新的图标库和一条资产生成流水线把贡献成本压到"放一张 PNG"。核心代码小而精，工程化（打包回归、CI、原子发布、浏览器内运行时）近年明显加强，47k+ star 与 Airflow 等采用者证明了其生态位。适合 Infrastructure/DevOps 工程师做架构原型与文档、技术写作者配图；想学习"极小 DSL 内核 + 资产驱动代码生成 + with 上下文管理器"设计的 Python 开发者，`diagrams/__init__.py` 与 `autogen.sh`/`scripts/generate.py` 这两条主线非常值得精读。
