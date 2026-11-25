# sqlmapproject/sqlmap 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：sqlmap 是开源的 SQL 注入自动化检测与数据库接管工具，面向**授权渗透测试**场景，是该领域事实上的行业标准（定位见 `README.md`）。

| 项目 | 事实 |
|------|------|
| 名称 | sqlmap（sqlmapproject/sqlmap） |
| 星数 | 39,500（快照）；gh api 2026-11-14 实测 38,950 |
| 主语言 | Python（纯 Python，2.7 与 3.x 双兼容） |
| 许可证 | GPL-2.0（附加澄清与例外条款，`LICENSE`；GitHub 侧显示 NOASSERTION） |
| 创建时间 | 2012-06-26（gh api） |
| 最近 push | 2026-11-12（gh api，分析前一天仍有提交） |
| 当前版本 | 主干 VERSION "1.10.11.88"（`lib/core/settings.py`）；最新正式版 1.10（2026-01-01） |
| 仓库规模 | 1,053 个文件（tarball 统计），tarball 约 8.4MB |

## 2. 项目是什么，解决什么问题

sqlmap 自动化完成 SQL 注入漏洞的检测与利用全流程：从识别注入点、判断后端数据库类型，到枚举库表数据、访问文件系统、乃至取得操作系统级控制，全部由引擎驱动（`README.md`）。它把原本高度依赖专家经验的手工注入过程，工程化为"参数化、可复现、可中断续测"的扫描流水线，是授权渗透测试与防御方验证防护有效性的基准工具。

其核心竞争力在于**数据驱动的检测引擎**：注入测试的定义不写死在代码里，而是声明在 `data/xml/` 下的 XML 中（边界 `boundaries.xml` + 六类技术各自的 `payloads/*.xml`），由引擎按 level/risk/子句上下文动态调度（`doc/ARCHITECTURE.md` 第 5 节）。这让检测知识的更新与代码解耦，社区贡献 payload 与贡献代码走同一条 PR 通道。

项目边界也在扩展：2.0 开发版在 SQL 之外新增了 NoSQL、XPath、LDAP、SSTI、GraphQL、HQL、SPARQL、OData、XSLT、XXE、JWT 等一批自包含扫描器，统一挂接在 `lib/techniques/<family>/inject.py` 并以 `NONSQL_TECHNIQUES` 注册（`doc/CHANGELOG.md`、`lib/core/settings.py`）。sqlmap 已从"SQL 注入工具"演进为"注入类漏洞通用检测框架"。

工程上它坚持**零运行时第三方依赖**：bottle、chardet、beautifulsoup、six 等全部 vendor 进 `thirdparty/`，克隆即可在 Python 2.7/3.x 下直接运行，无需 pip 安装（`doc/ARCHITECTURE.md` 第 3 节、仓库内无 requirements/pyproject 文件）。这是它能成为各种环境（离线内网、老系统）里随身工具的重要原因。

## 3. 技术栈与依赖

- **语言**：Python 2.7 与 3.x 双兼容（`README.md` 徽章、CI 矩阵含 pypy-2.7）；兼容层在 `lib/core/compat.py` 与 `thirdparty/six`。
- **运行时依赖**：无 pip 依赖，14 个第三方库全部 vendored 于 `thirdparty/`（bottle=REST API 服务器、chardet=编码探测、beautifulsoup=HTML 解析、colorama/termcolor=终端着色、identywaf=WAF 指纹识别、socks=代理、pydes/fcrypt=加密辅助等，目录清单核实）。
- **可选增强**：lxml/jinja2/h2/cryptography 仅测试用（CI `tests.yml` 中标注"optional test deps"）。
- **打包**：无 setup.py/pyproject/requirements——以"克隆即用"的脚本形态分发（tarball 全量清单核实）。
- **会话存储**：SQLite（`lib/utils/hashdb.py`，每目标一个 session.sqlite）。

## 4. 目录结构与代码组织

| 目录 | 职责（依据 `doc/ARCHITECTURE.md` 第 3 节与 tar 清单） |
|------|------|
| `lib/core/` | 全局模型与基础设施：conf/kb 单例、settings 常量、agent（payload 组装）、option 解析、dump、threads、patch |
| `lib/controller/` | 扫描编排器：`controller.py`(start 生命周期)、`checks.py`(检测)、`action.py`(枚举分发)、`handler.py`(DBMS 处理器选择) |
| `lib/request/` | HTTP 层：`connect.py`(唯一收口)、`comparison.py`(真假 oracle)、`inject.py`(取值分发)、20+ 协议 handler |
| `lib/techniques/` | 利用引擎：`blind/`、`error/`、`union/`、`dns/` + 11 个非 SQL 家族目录 |
| `lib/parse/` | 输入解析：CLI、配置文件、HTTP 请求文件、HTML、sitemap、XML payload 加载器 |
| `lib/utils/` | 特性模块：REST API、会话 hashdb、爬虫、哈希破解、WAF bypass、brute 字典猜解等 34 个文件 |
| `lib/takeover/` | OS 级接管模块：shell 抽象、UDF、xp_cmdshell、web 后门、Metasploit、注册表、icmpsh |
| `plugins/generic/` | DBMS 无关的枚举/指纹/文件系统/接管基类（13 个文件） |
| `plugins/dbms/<dbms>/` | 每个受支持 DBMS 一个目录，共 **31 个**（MySQL/PostgreSQL/Oracle/MSSQL/SQLite/ClickHouse/Snowflake/Spanner/HANA 等） |
| `tamper/` | payload 变换脚本，共 85 个 .py（含 `__init__.py`） |
| `data/xml/` | 数据驱动引擎核心：boundaries.xml、payloads/*.xml（6 类）、queries.xml（31 个 DBMS 查询模板）、errors.xml |
| `data/txt/`、`data/udf/`、`data/procs/`、`data/shell/` | 字典词表与资源文件 |
| `tests/` | 纯 stdlib unittest 离线测试套件（125 个条目） |
| `thirdparty/` | vendored 依赖（14 个库） |
| `extra/` | 辅助工具：vulnserver(自测靶场)、esperanto(引擎自检)、cloak、icmpsh、kerberos 等 12 个子目录 |
| `doc/` | `ARCHITECTURE.md`、`CHANGELOG.md`、26 种语言的 README 翻译 |

组织方式评述：**单仓库单包单体**架构，无子模块无 monorepo；模块划分是经典的"core / 编排 / 传输 / 技术 / 插件 / 数据"分层，`doc/ARCHITECTURE.md`（约 190 行的贡献者地图）明确声明"文档与代码冲突时以代码为准"，文档-代码同步纪律良好。

## 5. 核心架构与关键实现

### 5.1 全局状态与扫描生命周期（controller 数据流）

进程全局只挂两个单例：`conf`（用户配置的解析结果）与 `kb`（运行时知识库：已识别 DBMS、注入点、页面模板、锁），均为 `AttribDict`，定义于 `lib/core/data.py`（`doc/ARCHITECTURE.md` 第 2 节）。每个目标的扫描生命周期在 `lib/controller/controller.py: start()`：目标环境初始化与会话恢复（`lib/core/target.py`）→ 连通性/WAF/页面稳定性检查（`lib/controller/checks.py`）→ 启发式预检 → 逐参数数据驱动检测，确认点写入 `kb.injections` → `lib/controller/handler.py: setHandler()` 完成 DBMS 指纹并挂载 `conf.dbmsHandler` → `lib/controller/action.py: action()` 把 `--dbs/--tables/--dump` 等开关路由到 dbmsHandler 方法，结果交 `lib/core/dump.py` 输出（CSV/HTML/SQLITE/JSONL）。

### 5.2 数据驱动检测引擎（payload 组织）

检测知识全部声明在 XML，由 `lib/parse/payloads.py` 的 `loadBoundaries()/loadPayloads()` 加载（核实自 XML 头部注释与 ARCHITECTURE.md 第 5 节）：

- **`data/xml/boundaries.xml`**：注入边界（前缀/后缀对），每个 boundary 标注适用 level（1-5 档请求量）、clause（WHERE/GROUP BY/ORDER BY/LIMIT 等子句）、where 与参数类型（字符型/数字型）上下文。
- **`data/xml/payloads/*.xml`**：六类技术各一文件，每个 `<test>` 声明 stype（1=布尔盲注/2=报错/3=内联查询/4=堆叠查询/5=时间盲注/6=UNION）、level、risk（数据完整性风险 1-3 档）、clause 与比较/grep 判定逻辑。实测 test 数：time_blind 118、error_based 92、boolean_blind 76、stacked_queries 38、union_query 36、inline_query 8，合计 368 个。
- 调度：`lib/core/common.py: getSortedInjectionTests()` 按已识别 DBMS、`--level`、`--risk` 排序候选；`lib/core/agent.py` 把边界、随机数占位符与 tamper 变换组装成最终请求串。

判定"真/假响应"的 oracle 在 `lib/request/comparison.py`：基于 `kb.pageTemplate` 与 `kb.matchRatio` 的 difflib 相似度，叠加标题/报错/HTTP 码信号（ARCHITECTURE.md 第 5 节）。

### 5.3 取值引擎与 DBMS 抽象

确认可注入后，取值由 `lib/request/inject.py: getValue()` 按技术分发给 `lib/techniques/` 引擎：布尔/时间盲注走 `blind/inference.py` 的 `bisection()`（逐字符二分，多线程经 `lib/core/threads.py` 的 kb.locks 协调），报错走 `error/use.py`，UNION 走 `union/{test,use}.py`（先测列数再取值），DNS 带外走 `dns/`。DBMS 层是**模板方法 + 方言数据**的抽象：`plugins/generic/` 提供 fingerprint/enumeration/databases/users/filesystem/takeover 等基类，`plugins/dbms/<dbms>/` 子类化并给方言；SQL 模板统一放在 `data/xml/queries.xml`（31 个 DBMS 节点），通用代码按名取查询、方言由 XML 提供（ARCHITECTURE.md 第 7 节）。`lib/takeover/` 则按通道划分 OS 接管能力：`xp_cmdshell.py`、`udf.py`、`web.py`（写文件落后门）、`metasploit.py`、`registry.py`、`icmpsh.py`，由 `abstraction.py` 统一封装。

### 5.4 HTTP 收口、tamper 体系与 WAF 对抗的工程维度

所有请求经由 `lib/request/connect.py: getPage()` 单一收口发出，协议能力以 handler 挂接（keepalive/redirect/chunked/range/https/ntlm/kerberos/pki/websocket/http2，见 `lib/request/` 清单）。对抗 WAF/IPS 在工程上分四个正交维度（机制层面枚举，不展开细节）：

1. **识别**：`checks.py: checkWaf()` 结果入 `kb.identifiedWafs`，指纹能力来自 vendored 的 `thirdparty/identywaf`。
2. **tamper 脚本链**：`tamper/` 下约 84 个变换脚本，每个只暴露 `tamper(payload, **kwargs)` 接口与 `__priority__` 优先级常量（如 `tamper/apostrophemask.py`），`--tamper=a,b,c` 按优先级链式组合；脚本是纯字符串变换、零引擎耦合，因此可自由叠加（ARCHITECTURE.md 第 9 节）。
3. **指纹中性化**：`lib/utils/wafbypass.py: neutralizeFingerprint()` 让请求呈现为普通浏览器特征（User-Agent 源自 `data/txt/user-agents.txt`，与 `--random-agent` 同源）。
4. **自动 bypass 试错**：WAF 被识别且常规 payload 被拦截时，`wafbypass.py` 对候选 tamper 排序，`controller.py` 的 `_autoWafBypass()` 逐一试用，以"重跑检测后注入恢复"为采纳标准（`doc/CHANGELOG.md` 2.0 节）。

另：REST API 服务端在 `lib/utils/api.py`（bottle 实现，task/option/scan 路由，契约见 `sqlmapapi.yaml`），CLI 的 `--report-json` 与 API 端点复用同一份结果组装代码以防漂移（ARCHITECTURE.md 第 10 节）。

## 6. 构建与运行

零安装、零构建——克隆即用（`README.md`）：

```bash
git clone --depth 1 https://github.com/sqlmapproject/sqlmap.git
python sqlmap.py -h        # 基础选项；-hh 全量选项
python sqlmapapi.py -s     # REST API 服务端；-c 客户端
```

程序化调用：`import sqlmap` 后用 `lib/utils/library.py` 的 `scan()/scanFromRequest()`（以子进程驱动引擎）。

测试与自检命令（全部来自 `.github/workflows/tests.yml` 实文）：

```bash
python -B -m unittest discover -s tests -p "test_*.py"   # 离线单测（-B 不写 .pyc）
python sqlmap.py --smoke-test                            # 全树 doctest+正则自检
python sqlmap.py --vuln-test                             # 对 extra/vulnserver 端到端扫描
python sqlmap.py --api-test                              # REST 往返测试
python sqlmap.py --payload-lint                          # 对全 DBMS payload 做结构 lint
python sqlmap.py --doc-test                              # 文档示例测试（coverage 步骤内）
```

## 7. 测试与工程质量

- **测试规模与风格**：`tests/` 共 125 个条目，纯 stdlib unittest（明确不用 pytest，保证 py2/py3 都能跑），含 `tests/integration/audit/`（Docker compose 集成审计环境）与属性/模糊测试工具 `tests/_testutils.py`（跨版本一致的 Rng PRNG + for_all 装置）。
- **CI 矩阵**（`.github/workflows/tests.yml`）：ubuntu+pypy-2.7 / macos+3.8 / windows+3.14 三平台；步骤含 pyflakes 全量 lint（仅放行 redefines）、导入自检、单测、esperanto 引擎自检、coverage（`--fail-under=50`，统计范围 lib,plugins,tamper）、smoke/payload-lint/vuln/api 四个自测开关。
- **质量观察**：工程纪律突出——pyflakes 零告警门槛、`PYTHONHASHSEED=0` 防哈希序抖动、payload 有专门 lint（离线模拟全 DBMS 枚举并做结构校验）、`doc/ARCHITECTURE.md` 显式维护"改 X 去 Y 文件"的导览表。代码风格为 camelCase 标识符 + 全局 conf/kb 单例（Python 社区视角偏老派，但一致性极高）。

## 8. 社区与维护状态

- **活跃度（gh api 核实）**：最近 push 2026-11-12，最新三条提交均在那一天；项目创建于 2012-06-26，14 年持续高频提交，是 GitHub 上维护时间最长的安全工具之一。
- **发布节奏**：年度大版本 + 年内高频 patch tag——1.6(2022-01)、1.7(2023-01)、1.8(2024-01)、1.9(2025-01)、1.10(2026-01-01)，patch tag 已到 1.10.11；主干 VERSION "1.10.11.88" 滚动开发，`doc/CHANGELOG.md` 顶部即 2.0 (WIP)。
- **贡献者结构**：约 144 位贡献者（gh api contributors 分页统计）；版权主体为 Bernardo Damele A. Guimaraes 与 Miroslav Stampar（`LICENSE`），核心二人组长期主导 + 外围 PR 模式。
- **社区面**：6,441 fork、1,109 watchers、open issues 仅 14（处理极快）；`doc/translations/` 维护 26 种语言 README；`.github/` 有 CONTRIBUTING/CODE_OF_CONDUCT/FUNDING 与 issue 模板。

## 9. 亮点与风险

**亮点**

1. **检测知识数据化**：368 个测试定义、31 套 DBMS 查询方言、注入边界全部在 XML 声明（`data/xml/`），安全研究与软件工程解耦，这是 sqlmap 能持续沉淀十几年的根本设计。
2. **零依赖双解释器兼容**：14 个库全部 vendor，Python 2.7/3.x 单代码库兼容（`thirdparty/`、`lib/core/compat.py`），任意环境解压即用。
3. **插件化扩展面**：tamper 脚本接口极简（`tamper(payload,**kwargs)` + 优先级）且可组合；31 个 DBMS 插件、11 个非 SQL 注入家族以统一注册点接入（`lib/core/settings.py: NONSQL_TECHNIQUES`）。
4. **工程化自测闭环**：五个内建自测开关（smoke/vuln/api/payload-lint/doc-test）全部进 CI，附带自研靶场 `extra/vulnserver` 与引擎自检 `extra/esperanto`，安全工具少见的可回归性。
5. **断点续测与会话持久化**：每目标 SQLite 会话文件缓存注入点/指纹/取值，重跑自动恢复（`lib/utils/hashdb.py`）。

**风险/局限**

1. **滥用风险**：sqlmap 是双刃工具，可被用于未授权攻击，仅应在获得明确授权的测试中使用——本项目文档与社区均强调此边界（客观陈述，一句带过）。
2. **Python 2.7 兼容包袱**：为支持老解释器，代码无法使用现代 Python 语法与工具链（CI 仍在维护 pypy-2.7 通道）。
3. **无标准化打包**：无 requirements/pyproject/wheel，难以作为库集成进现代 Python 工程，只能整树 vendor 或子进程调用。
4. **发布模式偏旧**：正式 release 年度化，日常修复只进主干；coverage 门槛 50% 对安全工具而言不算高（`.github/workflows/tests.yml`）。

## 10. 总结评价

sqlmap 是"用工程方法沉淀领域知识"的典范：它把 SQL 注入检测这门依赖专家直觉的手艺，变成了 XML 声明 + 调度引擎 + 插件体系的数据驱动流水线，并用 14 年不间断的提交、零依赖双解释器兼容和五个自测开关维持着行业基准工具的地位。2.0 开发线把同一套框架延伸到 NoSQL/SSTI/GraphQL 等注入类漏洞，验证了其架构的泛化能力。适合两类人：**授权渗透测试工程师与安全防御人员**（把它当作检测能力基准与 WAF 防护有效性的标尺），以及**想学习"知识数据化 + 插件化引擎"设计的 Python 工程师**（`doc/ARCHITECTURE.md` + `lib/controller/controller.py` + `data/xml/` 的三层阅读路径是理解全项目最短的入口）。
