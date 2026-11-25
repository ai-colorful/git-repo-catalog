# psf/requests 项目分析

> 星数 54626（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：Python 生态最流行的"人类友好"HTTP 客户端库，以极简 API 封装 urllib3，是事实上的 Python HTTP 标准库。

| 项目 | 内容 |
| --- | --- |
| 名称 | psf/requests |
| 星数 | 54626（快照）；fork 11510；watch 54596 |
| 语言 | Python（纯 Python，约 6400 行核心代码） |
| 许可证 | Apache-2.0（`pyproject.toml`、`LICENSE`） |
| 当前版本 | 2.34.2（2026-05-14 发布，`HISTORY.md`、`src/requests/__version__.py`） |
| 最近 push | 2026-11-02（GitHub API，分析前一天仍活跃） |
| 贡献者 | 约 795 人（GitHub contributors 分页推算，`anon=true` 含匿名） |
| 依赖规模 | 官方口径：周下载约 3 亿次，被 400 万+ 仓库依赖（`README.md`） |

## 2. 项目是什么，解决什么问题

Requests 解决的是 Python 标准库 `urllib` 难用的问题：发起一个带认证、查询串、JSON 体、TLS 校验的 HTTP 请求，标准库需要十几行样板代码，而 Requests 只需 `requests.get(url, auth=..., json=...)`。它把 URL 编码、表单/文件上传、cookie 管理、内容自动解压与编码探测、代理与重定向全部自动化（`README.md` "Supported Features" 一节）。

设计哲学是"为人类设计的 API"：顶层 `requests.get/post/...` 是无状态便捷入口（`src/requests/api.py`，每次调用临时创建并关闭一个 `Session`，见 `api.py` 第 70-71 行的 `with sessions.Session() as session`）；需要跨请求保持 cookie、连接池、配置时用 `requests.Session()`。接口文档以 docstring 形式内嵌于源码，同时支撑 `--doctest-modules` 的文档测试（`pyproject.toml` pytest 配置）。

它刻意不做底层传输：所有 socket、连接池、TLS 细节委托给 urllib3（`urllib3>=1.26,<3`，`pyproject.toml` 第 18-23 行），Requests 自身只负责"请求的构造、设置的合并、响应的建模、重定向与认证的策略"。这种薄封装策略是它长期稳定的关键——API 十几年几乎不变。

项目由 Kenneth Reitz 于 2011 年创建，现归 Python 软件基金会（PSF）所有，README 页脚同时署名作者与 PSF；日常维护由 Ian Stapleton Cordasco 与 Nate Prewitt 两人负责（`pyproject.toml` maintainers 字段）。

## 3. 技术栈与依赖

- 语言：纯 Python，声明支持 CPython 3.10–3.15 与 PyPy、自由线程 3.14t（`pyproject.toml` classifiers）。
- 运行时依赖仅 4 个（`pyproject.toml`）：
  - `urllib3`（1.26–2.x）：实际 HTTP 传输、连接池、重试；
  - `certifi`：CA 根证书包；
  - `charset_normalizer`：响应体字符编码探测（可用 extras `use_chardet_on_py3` 换回 chardet）；
  - `idna`：国际化域名编码。
- 可选依赖：`PySocks`（SOCKS 代理，extras `socks`）。
- 开发工具链：pytest（含 pytest-httpbin / trustme / xdist / cov）、ruff（lint+format，`pyproject.toml` [tool.ruff]）、pyright strict（`[tool.pyright]` typeCheckingMode="strict"）、tox 多环境、GitHub Actions（test/lint/typecheck/codeql/zizmor/publish）。
- 打包：setuptools（src 布局，版本号从 `requests.__version__.__version__` 动态读取，`pyproject.toml` 第 80-84 行）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `src/requests/` | 全部核心代码，19 个模块约 6400 行，单包无子包 |
| `src/requests/api.py` | 顶层 `get/post/...` 便捷函数，委托给一次性 Session |
| `src/requests/sessions.py` | `Session`、设置合并、重定向策略（920 行） |
| `src/requests/adapters.py` | `BaseAdapter`/`HTTPAdapter`，urllib3 集成边界（748 行） |
| `src/requests/models.py` | `Request`/`PreparedRequest`/`Response` 数据建模（1184 行） |
| `src/requests/utils.py` | URL/proxy/编码等杂项工具（1155 行） |
| `src/requests/cookies.py` | `RequestsCookieJar` 及 cookie 提取合并（625 行） |
| `src/requests/auth.py` | Basic/Digest 及自定义认证 `AuthBase` |
| `src/requests/hooks.py` | hooks 事件分发（48 行） |
| `src/requests/exceptions.py` | 全部异常类型层次 |
| `src/requests/compat.py` | Python 版本差异抹平层 |
| `tests/` | pytest 测试约 5000 行 + 本地 `testserver/` + 证书目录 |
| `docs/` | Sphinx 文档源（requests.readthedocs.io） |
| `ext/` | README 徽章图片等静态资源 |
| `HISTORY.md` | 完整 release 历史（文本型 changelog） |

组织方式评述：典型的"单包极简"结构——所有公共 API 平铺在一个包内、无内部子包，`__init__.py` 直接 re-export 公共符号（`src/requests/__init__.py` 第 150 行以后的 `from .api import ...` 区段）。没有任何生成代码或 vendored 目录（历史上 vendor 过 urllib3/chardet，2019 年起全部移除，`packages.py` 现仅 23 行兼容 shim）。

## 5. 核心架构与关键实现

### 5.1 三层调用链：api → Session → Adapter → urllib3

调用链是严格单向的：`requests.get()` 创建临时 `Session` → `Session.request()` 把参数装进 `Request` 并 `prepare_request()` 生成 `PreparedRequest`（`src/requests/sessions.py` 第 511-555 行）→ `Session.send()` 按 URL 前缀选 adapter → `HTTPAdapter.send()` 调 `conn.urlopen()`。每层职责清晰：PreparedRequest 是"冻结的待发请求"，Session 负责策略，Adapter 只管传输。

```python
# src/requests/sessions.py L776-L791（Session.send 核心段）
adapter = self.get_adapter(url=request.url)
start = preferred_clock()
r = adapter.send(request, **kwargs)
elapsed = preferred_clock() - start
r.elapsed = timedelta(seconds=elapsed)
r = dispatch_hook("response", hooks, r, **kwargs)
extract_cookies_to_jar(self.cookies, request, r.raw)
```

### 5.2 设置合并模型：请求级覆盖会话级，None 表示删除

Session 最核心的设计是 `merge_setting`（`sessions.py` 第 76-105 行）：请求级设置覆盖会话级，字典类设置（headers/params）做浅合并，且**值为 None 的键会被删除**——这是用户取消 Session 默认 header 的官方机制。`merge_hooks` 单独处理 hooks 的空列表边界（第 108-124 行）。环境信任由 `merge_environment_settings` 完成：`trust_env=True` 时读取 `HTTP_PROXY/HTTPS_PROXY/NO_PROXY`，verify 优先级为 `REQUESTS_CA_BUNDLE` > `CURL_CA_BUNDLE` > 会话值（第 831-868 行）。

### 5.3 重定向策略引擎（SessionRedirectMixin）

重定向不是在传输层做（urlopen 传 `redirect=False`，`adapters.py` 第 701 行），而是由 Session 层的 `resolve_redirects` 生成器逐跳处理（`sessions.py` 第 186-307 行）：每跳释放上一响应的 socket、处理相对 Location 与 RFC 3986 重编码、跨请求搬运 cookie、重建代理与认证。安全策略体现在两处：

- `rebuild_auth`：主机名变化即剥离 `Authorization` 头，http→https 同主机例外（`sessions.py` 第 309-332 行 + `should_strip_auth`）；
- `rebuild_method`：模仿浏览器——303 一律转 GET，302 转 GET，301 且原方法为 POST 也转 GET（第 370-392 行），而非严格遵循 RFC。

### 5.4 HTTPAdapter：urllib3 集成的唯一边界

`HTTPAdapter` 内部持有 `urllib3.PoolManager`（`init_poolmanager`，`adapters.py` 第 239-267 行，默认 pool_connections=maxsize=10），代理按 proxy URL 缓存 `ProxyManager`/`SOCKSProxyManager`（`proxy_manager_for`，第 269-305 行）。2.32.2 起子类扩展点从 `get_connection` 迁移到 `get_connection_with_tls_context`（旧方法已标 DeprecationWarning，第 512-553 行），把 verify/cert 编译进 urllib3 的 PoolKey（`build_connection_pool_key_attributes`，第 403-453 行）。它的另一半职责是**异常翻译**：把 urllib3 异常体系映射为 requests 异常体系。

```python
# src/requests/adapters.py L713-L729（节选：urllib3 异常 → requests 异常）
except MaxRetryError as e:
    if isinstance(e.reason, ConnectTimeoutError):
        if not isinstance(e.reason, NewConnectionError):
            raise ConnectTimeout(e, request=request)
    if isinstance(e.reason, ResponseError):
        raise RetryError(e, request=request)
    if isinstance(e.reason, _ProxyError):
        raise ProxyError(e, request=request)
    if isinstance(e.reason, _SSLError):
        raise SSLError(e, request=request)
    raise ConnectionError(e, request=request)
```

默认 `max_retries=Retry(0, read=False)` 即不重试（`adapters.py` 第 208-209 行）；超时的 `(connect, read)` 元组映射为 `urllib3.util.Timeout`（第 681-693 行）。

### 5.5 Response 建模与 hooks 体系

`Response` 包装 urllib3 response：`raw` 保留原始对象，`content` 属性惰性读取并解码，`iter_content` 流式迭代，`json()` 基于 charset_normalizer 探测的编码反序列化（`src/requests/models.py` 第 732-1184 行，`def content` 在第 1035 行、`json` 在第 1091 行）。hooks 体系极简（整个 `hooks.py` 仅 48 行）：目前只有 `"response"` 一个事件（`HOOKS = ["response"]`），`dispatch_hook` 依次执行回调，回调返回值非 None 时可整体替换 Response 对象——这是公开的响应拦截点，也是 requests-mock 等库的扩展基础。另一个隐式扩展点是 `Session.mount()`：adapter 按前缀长度降序匹配（`sessions.py` 第 888-897 行），用户可挂自定义协议 adapter（如 furl、bolt）。

## 6. 构建与运行

来自 `Makefile` 与 `pyproject.toml` 的实际命令：

```bash
# 环境初始化
python -m pip install -r requirements-dev.txt     # make init
# 运行测试
python -m pytest tests                             # make test
python -m pytest tests --junitxml=report.xml       # make ci
# 覆盖率
python -m pytest --cov=src/requests tests          # make coverage
# 多环境测试（py310-py314 × default/use_chardet_on_py3）
tox
# 类型检查
pyright                                            # 严格模式，见 pyproject.toml [tool.pyright]
# 发布
python -m build && python -m twine upload dist/*   # make publish
# 文档
cd docs && make html                               # make docs
```

无任何编译产物，`pip install requests` 即装即用；版本号由 setuptools 从 `src/requests/__version__.py` 动态读取。

## 7. 测试与工程质量

- 测试规模：`tests/` 共约 5000 行，主文件 `tests/test_requests.py` 含 237 个 test 函数，另有 `test_utils.py` 64 个、`test_lowlevel.py` 13 个（端到端走本地 testserver）、`test_hooks.py`、`test_adapters.py` 等；`pyproject.toml` 开启 `--doctest-modules`，docstring 示例本身也是测试。
- 测试风格：以 pytest-httpbin 在本地起 HTTP 服务模拟真实场景（不依赖外网 httpbin），trustme 动态签发 TLS 证书做真实验证（`pyproject.toml` dependency-groups test），`tests/testserver/` 是自带的可编程测试服务器——单元与集成兼顾，无 mock 泛滥。
- CI 矩阵：Python 3.10/3.11/3.12/3.13/3.14/3.14t/3.15-dev/pypy-3.11 × ubuntu/macOS/windows（`.github/workflows/run-tests.yml` strategy.matrix）。
- 质量门禁：ruff lint+format、pyright strict（`_types.py`、内联类型注解是 2.34.0 新引入并取代 typeshed 存根，`HISTORY.md` 2.34.0 节）、CodeQL 安全扫描、zizmor（Actions 安全审计）、dependabot（`.github/workflows/` 目录）。
- 观察：核心代码有大量历史沉淀的安全注释（如 `Session.verify` 的 MitM 警告、`rebuild_proxies` 避免 TLS 隧道泄露 Proxy-Authorization），工程成熟度高。

## 8. 社区与维护状态

- 活跃度：GitHub API 显示最近 push 为 2026-11-02（分析前一天），tarball 内文件 mtime 为 2026-10-27，持续有合并。
- Release 节奏：`HISTORY.md` 显示 2.34.0（2026-05-11）→ 2.34.1（05-13）→ 2.34.2（05-14），类型系统大版本后快速补丁，最新 release v2.34.2（GitHub API）。近年节奏大致为数月一版（未做全年统计，未确认精确频率）。
- 治理：PSF 旗下项目，两位具名 maintainer（`pyproject.toml`），`.github/CODEOWNERS`、`AI_POLICY.md`（明确 AI 生成贡献政策）、`SECURITY.md` 齐备；issue 管理（close/lock issues 工作流）有自动化。
- 贡献者约 795 人（GitHub API contributors 分页尾页推算），但实际合并权集中在少数 maintainer 手中。

## 9. 亮点与风险

亮点：

1. API 设计教科书级：`get/post/...` 无状态便捷层与 `Session` 有状态层共享同一实现（`api.py` 委托模式），十几年 ABI 稳定，是世界范围内被复用最多的 Python 包之一（周下载 3 亿）。
2. 边界干净：只做策略不做传输，urllib3 的全部复杂性被 `HTTPAdapter` 一个类隔离，异常翻译层（`adapters.py` send 方法）让用户永远只见 requests 异常。
3. 安全策略内建且文档化：跨域重定向剥离凭证、代理认证不进 TLS 隧道、verify=False 的显式 MitM 警告，均写在源码注释里。
4. 测试基建真实：pytest-httpbin + trustme + 自研 testserver，测试走真 HTTP 栈而非 mock。
5. 类型现代化：2.34.0 起内联类型取代 typeshed，pyright strict 通过。

风险/局限：

1. 仅同步 HTTP：无 asyncio 支持（官方推荐 httpx/aiohttp），Python 异步生态已事实上分流。
2. 单 maintainer 风险：日常维护实质依赖 2 人，issue 积压 247 个（GitHub API open_issues）。
3. 行为兼容包袱：如 301 POST→GET 的"浏览器行为"偏离 RFC（`sessions.py` 第 382-391 行注释自认"bizarre"），修正任何长期行为都可能破坏海量下游。
4. 编码探测依赖 charset_normalizer，无 BOM 且无 header 编码时的猜测结果可能与真实编码不符（`utils.py` `get_encodings_from_content` 逻辑边界，未确认具体误判率）。

## 10. 总结评价

Requests 是"薄封装、强策略"设计范式的典范：6400 行核心代码承载了全球 3 亿次周下载，靠的是把复杂度严格下压给 urllib3、把安全与易用策略收敛在 Session 层、并把 API 冻结在最好的版本上。读源码的价值在于三件事——设置合并的覆盖/删除语义、重定向生成器的逐跳状态机、adapter 的异常翻译边界；适合所有 Python 开发者作为日常工具，适合库作者作为"如何设计十年不过时的公共 API"的必读样本。新项目若需异步或 HTTP/2，应考虑 httpx；纯同步场景下 requests 仍是默认正确答案。
