# faif/python-patterns 项目分析

> 星数 42200（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-06

## 1. 项目速览

一句话定位：GitHub 上最经典的 Python 设计模式参考实现集——每个模式一个独立、可运行、带 doctest 自测的单文件示例。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | faif/python-patterns | gh api repos/faif/python-patterns |
| 星数 | 42200（快照），分析时 API 显示 43085 | 任务参数 / gh api |
| 语言 | Python（纯 Python，零运行时依赖） | pyproject.toml |
| 许可证 | **无 LICENSE 文件**（API license 字段为 null，源码包内亦无） | gh api git/trees + 本地解包 |
| 创建时间 | 2012-06-06 | gh api |
| 最近提交 | 2026-11-01（push 2026-11-05），持续活跃 | gh api commits/HEAD |
| 默认分支 | master，无 release（不打版本号，"最新即文档"） | gh api |
| 贡献者 | 122 人 | gh api contributors |

## 2. 项目是什么，解决什么问题

GoF 的 23 个设计模式以 Java/C++ 为语境写成，直接搬到 Python 常显得笨重：Python 有模块级单例、鸭子类型、一等函数、元类、描述符等语言特性，不少"模式"要么有更简洁的表达，要么根本不需要。python-patterns 要解决的就是"**这些模式在 Python 里到底该怎么写、哪些值得写、哪些不该写**"这个教学缺口。

组织形式极度克制：`patterns/` 下每个模式就是一个自包含的 `.py` 文件，文件头是统一的模块 docstring（固定小节：What is this pattern about? / What does this example do? / Where is the pattern used practically? / References / TL;DR），正文是带类型注解的最小示例，结尾用 `main()` 内嵌 doctest 做可运行自测（见 `patterns/creational/borg.py`）。没有框架、没有抽象基类层、没有安装步骤——clone 下来任何一个文件都能直接 `python xxx.py` 跑通并看到输出断言。

README 在模式表之外还维护了一个"🚫 Anti-Patterns"章节，明确警告单例（Python 模块本身就是单例）、上帝对象、滥用继承在 Python 中不推荐（README.md 第 130-151 行）。这种"教模式也教反模式"的立场是该仓库区别于普通模式清单的关键。

项目定位是"参考书"而非"库"：pyproject 里 `dependencies = []`，不发布 PyPI release（releases=0），代码即文档，22k+ fork 证明大量用户是直接抄文件进自己项目。

## 3. 技术栈与依赖

- **语言**：纯 Python，`requires-python = ">=3.10"`，classifiers 标注支持 3.10–3.14（pyproject.toml）。代码使用 3.10+ 语法：`str | None` 联合类型、`dict[str, str]` 内建泛型、`TypeAlias`（`patterns/behavioral/strategy.py`）。
- **运行时依赖：零**（pyproject.toml `[project] dependencies= []`）。
- **开发工具链**（Pipfile / Pipfile.lock，pipenv 管理）：black>=25.1、isort、flake8>=7.1、mypy、pytest、pytest-cov、pytest-randomly、tox、codespell、pyupgrade——全部为质量工具，无业务依赖。
- **Python 2 兼容**：旧版实现移到 `legacy` 标签（README.md Contributing 节）。

## 4. 目录结构与代码组织

顶层（tarball HEAD，共 144 个文件条目）：

| 目录/文件 | 职责 |
|---|---|
| `patterns/` | 全部模式实现，按 GoF 三大类 + fundamental + other 分目录 |
| `patterns/behavioral/` | 行为型 17 个文件（含 `iterator_alt.py` 备选实现）+ `viz/` 类图 |
| `patterns/creational/` | 创建型 7 个文件 + `viz/` |
| `patterns/structural/` | 结构型 11 个文件（含 `flyweight_with_metaclass.py` 变体）+ `viz/` |
| `patterns/fundamental/` | 基础型 1 个（delegation_pattern.py） |
| `patterns/other/` | 非 GoF：blackboard.py、graph_search.py、`hsm/hsm.py`（层次状态机） |
| `patterns/dependency_injection.py` | 可测试性模式（依赖注入 3 变体），置于包顶层 |
| `tests/` | 27 个 pytest 测试文件，938 行，目录结构与 patterns 镜像 |
| `patterns/*/viz/` | 33 张 PNG 类图（如 `patterns/behavioral/viz/observer.py.png`） |
| `pyproject.toml` | 包元数据 + pytest/coverage/mypy/flake8/black/tox 全套配置 |
| `Pipfile` / `Pipfile.lock` | pipenv 开发依赖锁定 |
| `lint.sh` / `Makefile` / `.travis.yml` | 一键质量检查 / 辅助 make 目标 / 旧 Travis CI |
| `config_backup/` | 旧 setup.cfg、tox.ini 的备份归档 |
| `.github/workflows/lint_python.yml` | GitHub Actions CI |

组织方式评述：**"一模式一文件"的扁平单体**。没有内部包依赖（`patterns/__init__.py` 为空文件），文件之间互不引用，目录结构即分类体系。这是内容型教学仓库最优的组织——按目录找模式，按文件名 grep，按 docstring 读懂。

## 5. 核心架构与关键实现

所谓"核心"就是代表性模式文件本身。三个能体现"Python 化改编"深度的样本：

**1）Borg/Monostate：用共享 `__dict__` 替代单例**（`patterns/creational/borg.py`）
核心只有两行——把所有实例的 `__dict__` 指向同一个类级字典，实现"多实例共享状态"而非"单实例"：

```python
class Borg:
    _shared_state: dict[str, str] = {}
    def __init__(self) -> None:
        self.__dict__ = self._shared_state
```

docstring 完整解释了 Python 实例属性存储机制（`__dict__`），这是 GoF Singleton 的地道 Python 替代——文件里 `rm1 is rm2` 为 `False` 但状态同步的 doctest 是自证。

**2）Strategy：函数即策略 + 描述符校验**（`patterns/behavioral/strategy.py`）
不用策略接口类，直接 `DiscountStrategy: TypeAlias = Callable[["Order"], float]`，一等函数做策略；再用描述符协议（`__set_name__`/`__set__`/`__get__`）实现 `Order.discount_strategy = DiscountStrategyValidator()` 的赋值时合法性校验。GoF 中一个接口 + N 个实现类，这里压缩成一个类型别名 + N 个函数，是典型的"Python 特有惯用法改编"。

**3）Flyweight：元类劫持 `__call__` 做对象池**（`patterns/structural/flyweight_with_metaclass.py`）
`FlyweightMeta(type)` 在 `__new__` 里给类挂 `weakref.WeakValueDictionary()` 池，`__call__` 里按 `repr((cls.__name__, args, sorted(kwargs.items())))` 序列化参数为键、命中即复用。与同目录常规版 `flyweight.py` 并存，展示"同一模式两种实现深度"。该文件是全仓库少数无 doctest 的（另有 `patterns/other/hsm/hsm.py`），由 `tests/structural/test_flyweight_with_metaclass.py` 覆盖。

非 GoF 部分亦有分量：`patterns/other/hsm/hsm.py` 是 C++ 层次状态机示例的 Python 移植（约含 4 个状态类的转译），配 `tests/test_hsm.py` 与两张 PNG 类图。

## 6. 构建与运行

零依赖、无构建，运行任一模式：

```bash
python patterns/creational/borg.py        # 直接跑，docstring doctest 即自测
python -m doctest patterns/behavioral/strategy.py -v   # 手动跑 doctest
```

开发环境与全套检查（README.md Contributing 节 + lint.sh）：

```bash
python3.10 -m venv venv && source venv/bin/activate   # 要求 3.10+
make lock                # pipenv lock --dev && pipenv sync --dev
./lint.sh                # codespell + flake8 + isort --check + black --check + mypy + pytest
```

测试命令（CI 中实际执行，`.github/workflows/lint_python.yml`）：

```bash
python -m pytest tests patterns   # doctest-modules 模式，模式文件本身即测试目标
```

## 7. 收录范围、组织与模式分类

- **模式文件总数：40 个**（`find patterns -name "*.py" ! -name "__init__.py"`），分类：创建型 7、结构型 11（含 1 个元类变体）、行为型 17（含 1 个备选实现 `iterator_alt.py`）、基础型 1、可测试性（依赖注入）1、非 GoF 其他 3（blackboard/graph_search/hsm）。
- **README 对照表**（README.md）：按 Creational/Structural/Behavioral/Testability/Fundamental/Others 六张 Markdown 表组织，每行"模式名 → 源文件链接 → 一句话描述"，并新增 mermaid 关系图（README 第 11-87 行）。表是导航主入口。
- **README 与实际的两处不同步**（真实核对结论）：
  1. README 行为型表列有 `patterns/behavioral/interpreter.py`，但该文件在 HEAD 文件树中**不存在**（gh api git/trees 144 条目中无 interpreter，本地解包亦无）——README 超前于实现或文件被删未更新；
  2. `patterns/structural/flyweight_with_metaclass.py` 在磁盘上存在但 README 结构型表中未收录。
- **收录口径**：GoF 23 个模式基本全覆盖（interpreter 缺文件是唯一例外），另扩展 blackboard、graph_search、HSM、依赖注入等非 GoF 条目；开放 issue 中还有 Null Object/Singleton、Pipeline、CoR fallback 等扩充提案（gh api issues #475/#461/#473）。

## 8. 模式文件代码风格与 GoF 的 Python 化改编

- **可运行自测为主，教学片段为辅——实际全部是可运行**：40 个模式文件中 38 个内嵌 `doctest.testmod()`（含 `optionflags=doctest.ELLIPSIS` 变体），`main()` 里的 doctest 既是文档示例又是断言；仅 `hsm.py`（有独立 pytest）和 `flyweight_with_metaclass.py` 例外。配合 pyproject 的 `addopts = "--doctest-modules ..."`（pyproject.toml 第 61 行），模式文件本身进入 CI 测试面，"文档即测试"。
- **统一 docstring 五段式**（What is this pattern about? / What does this example do? / Where is the pattern used practically? / References / TL;DR），且"Examples in Python ecosystem"小节鼓励链接真实开源项目用法——贡献指南明文要求（README.md Contributing 节），这是内容一致性的制度保障。
- **Python 化改编的惯用法清单**（逐文件核对所得）：共享 `__dict__` 的 Borg 替代 Singleton（borg.py）；`TypeAlias + Callable` 函数策略替代策略接口（strategy.py）；元类 `__call__` 池化替代 Flyweight 工厂（flyweight_with_metaclass.py）；描述符做属性校验（strategy.py）；`weakref.WeakValueDictionary` 管理生命周期；`from __future__ import annotations` 延迟求值注解。
- **代码规模克制**：patterns 全部 4042 行 / 47 个 py 文件（含 7 个空 `__init__.py`），平均每模式约 100 行，无一处为了"完整性"而堆砌企业级样板。

## 9. 维护状态、亮点与风险

**维护状态（活跃，且有明显的近期现代化投入）**
- 最新提交 2026-11-01，push 2026-11-05（gh api）；issue 面干净（仅 15 个 open，多为 dependabot 依赖升级）。
- CI 现代且严格：`.github/workflows/lint_python.yml` 双 job——quality（lint.sh 五件套：codespell/flake8/isort/black/mypy/pytest）+ tests（Python 3.10–3.14 五版本矩阵）；旧 `.travis.yml` 仍在但配置已同步到 3.10-3.14。
- 近期工程化痕迹：pyproject 从 config_backup 中迁出（`config_backup/setup.cfg`、`tox.ini` 是归档旧配置），black>=25.1、checkout@v7 均为新版。
- 122 位贡献者、CodeTriage 挂接（README 徽章），PR 流程有明文贡献规范（docstring 规范、更新 README、跑 lint.sh）。

**亮点**
1. 每个文件可独立运行、自断言（doctest），教学正确性由 CI 强制——内容型仓库罕见的质量下限保障。
2. "教模式也教反模式"（README Anti-Patterns 节），立场务实。
3. Python 化改编到位：不是 GoF 的机械翻译，而是按语言特性重构（函数策略、Borg、元类池化）。
4. 零依赖、一模式一文件，复制粘贴即用，门槛极低。
5. 13 年持续维护 + 五版本 Python 矩阵测试，时效性好。

**风险/局限**
1. **无 LICENSE 文件**（API license=null，源码树中亦无），严格法律意义上不可复制使用——对"抄代码进项目"的主要使用方式是实质风险。
2. README 与实现轻微失同步（interpreter.py 幽灵条目、flyweight_with_metaclass.py 缺表项）。
3. 模式偏"教学最小实现"，无生产级考虑（线程安全、并发池等），读者需自行辨析适用边界——仓库自己也这么说（README 开头"each pattern has its own trade-offs"）。
4. 无 release/版本号，引用时只能 pin commit。

## 10. 总结评价

python-patterns 是"内容即代码"教学仓库的标杆样本：40 个模式文件、4042 行代码、零依赖，却拥有五版本 Python 矩阵 CI、mypy/black/flake8 全套质量门禁和 122 位贡献者的 13 年持续投入——它证明了内容型仓库同样可以用工程手段保证教学正确性。它能积累 42k+ 星的原因清晰：2012 年起步占据"Python 设计模式"搜索心智、每个示例可一键运行且自带断言、README 六表导航即查即用、以及"反模式"章节传达的务实立场。适合两类人：Python 初中级开发者把它当模式入门的对照手册（先读 docstring 五段式再跑 doctest）；工程规范实践者把它当"如何用 CI 管理教学内容仓库"的参考实现。读源码的首选路径是 borg.py → strategy.py → flyweight_with_metaclass.py，三个文件正好演示从语言特性替代 GoF、到一等函数、再到元类的 Python 化梯度。唯一必须提醒的是其无许可证状态，商用复制前需联系作者确认。
