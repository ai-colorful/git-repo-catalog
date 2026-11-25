# Z4nzu/hackingtool 项目分析

> 星数 81332（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：面向授权安全测试的一体化工具集合与启动器——把 219 个第三方安全工具组织进统一目录，叠加 AI 引导层（自然语言 → 工具与命令）、三种终端前端（经典菜单 / 行内 REPL / Textual TUI）和一条无头作战流水线。

| 事实 | 值 | 出处 |
|---|---|---|
| 名称 | hackingtool | pyproject.toml |
| 星数 / Fork | 81332（快照）/ 9179 | gh api 元数据 |
| 主语言 | Python（3.10+） | pyproject.toml |
| 许可证 | MIT | gh api、LICENSE |
| 当前版本 | v3.1.0 | src/hackingtool/constants.py |
| 最近推送 | 2026-10-15 | gh api repos/Z4nzu/hackingtool |
| 平台 | Linux / macOS（Kali、Parrot、Debian/Ubuntu、Arch）；Windows 不支持 | README.md |
| 贡献者 | 30 人 | gh api contributors |

## 2. 项目是什么，解决什么问题

hackingtool 创建于 2020-04（gh api created_at），最初是一个 `python3 hackingtool.py` 平面菜单脚本（现 Dockerfile 注释自述："not the old `python3 hackingtool.py` flat script"），2026 年经历了彻底重写：v3.0.0（2026-07-26，提交记录 `chore(release): 3.0.0`）把仓库改造成标准 src 布局的 Python 包，v3.1.0（2026-10-15，提交 `feat: interactive TUI, self-update & native settings`）加入 Textual 全屏 TUI 与自更新。它解决的问题是渗透测试/OSINT 学习者面对上千个散落 GitHub 的安全工具时的"发现、安装、记忆命令"成本。

产品形态分三层。第一层是**策展目录**：219 个活跃工具、21 个分类、44 个归档条目（docs/TOOLS.md），每个工具带描述、安装方式、运行命令、用法速查（usage cheatsheet）和"实验室安全提示"（lab_safe_notes）。第二层是**三个终端前端**：默认 Textual TUI（浏览/搜索/运行）、`hackingtool --classic` 行内 REPL（`/` 命令、`@` 工具名、裸文本三种语法，src/hackingtool/repl.py）、非交互终端回退的经典编号菜单（cli.py）。第三层是 **AI 引导层**（全部 opt-in、自带密钥）：`/ai` 把自然语言意图映射到工具、`/goal` 一次建模生成逐步计划并逐步确认执行、`c` 键为单个工具生成命令、无头模式对真实发现做摘要与报告草稿。

项目的差异化卖点在"防幻觉与防滥用设计"：AI 模型只能从 63 个标签的封闭分类（tags.py `TAXONOMY`）里选词，目录负责"标签 → 工具"的解析，因此模型永远无法捏造一个不存在的工具；模型输出与工具原始输出都被当作不可信数据（`<scan_data>` 包裹 + 字符消毒，ai_summary.py、skill/OPERATOR.md 章程）。范围外请求（DDoS、隐匿、大规模目标）在任何网络调用之前就被拒绝（discover.py 的 refusal filter）。

需要明确：hackingtool 本身**不是**攻击工具，它是一个启动器与引导层——真正的攻击能力来自它安装的第三方工具（nmap、subfinder、nuclei、SET、Evilginx 等），项目把自己定位为"仅用于你拥有或获授权测试的系统"（README、CONTRIBUTING 地面规则、SECURITY.md 威胁模型）。

## 3. 技术栈与依赖

- 语言与运行时：纯 Python，`requires-python >= 3.10`（pyproject.toml），src 布局 + setuptools 构建，版本号单源化自 `hackingtool.constants.VERSION`（pyproject.toml `[tool.setuptools.dynamic]`）。
- 运行依赖（pyproject.toml）：`rich>=13`（终端 UI/表格/面板）、`textual>=8.2`（全屏 TUI）、`prompt_toolkit>=3.0`（REPL 行编辑/补全）、`pyyaml>=6`（目录与流水线）、`platformdirs>=4`（跨平台路径）、`python-dotenv>=1.0`（`.env` 读 API key）。
- 外部工具依赖（README "Optional runtimes"）：Go 1.21+（nuclei/ffuf/amass 等安装）、Ruby（haiti、evil-winrm）、tmux（后台面板）、Docker（Mythic、MobSF 可选）。核心应用本身不需要这些。
- AI 传输：无 SDK——标准库 `urllib` 直连 OpenAI 兼容端点或本地 Ollama（ai_recommend.py `_byo_key`/`_ollama`），API key 只存 `~/.hackingtool/.env`（模式 600）。
- 开发工具链：uv（依赖组 dev：pytest>=8、ruff>=0.6）、fpm（打包 .deb）、CycloneDX SBOM（release.yml）。

## 4. 目录结构与代码组织

| 路径 | 职责 |
|---|---|
| `src/hackingtool/core.py` | HackingTool/HackingToolsCollection 基类、菜单循环、命令执行与审计日志 |
| `src/hackingtool/catalog/*.yaml` | 数据驱动工具目录（22 个文件，含 legacy_overlays.yaml 归档覆盖） |
| `src/hackingtool/registry.py` | YAML → CatalogTool 加载器；overlay 增强机制；用户目录 inert 加载 |
| `src/hackingtool/tags.py` | 63 个标签的封闭分类法 |
| `src/hackingtool/tools/*.py` | 21 个 Python 定义的分类集合（旧层，392-900 行/文件） |
| `src/hackingtool/cli.py`（865 行） | 经典菜单入口 + 分类装配 + 无头模式 CLI 参数 |
| `src/hackingtool/repl.py` / `prompt.py` | 行内 REPL（prompt_toolkit）与统一输入面 |
| `src/hackingtool/tui/` | Textual TUI：screens（main/category/tool/config）、widgets、app.tcss |
| `src/hackingtool/ai_*.py` | AI 层五模块：recommend/command/goal/summary/report |
| `src/hackingtool/skill/` | OPERATOR.md 操作员章程 + METHODOLOGY.md，AI 提示词的单源 |
| `src/hackingtool/engagement.py`、`orchestrator.py`、`findings.py`、`report.py`、`pipelines/recon.yaml` | 无头作战：工作区、流水线执行、发现解析、确定性报告 |
| `src/hackingtool/discover.py`（791 行） | `/find`：目录优先 + GitHub Search API，仅建议不执行 |
| `src/hackingtool/session.py` | tmux 后台面板封装（全部 list-form subprocess） |
| `scripts/` | check.sh 门禁、audit_tools.py 工具维护状态审计、discover_tools.py、gen_catalog_owners.py、apply_archive.py |
| `tests/` | 34 个测试文件，约 3972 行，341 个测试函数 |
| `.github/workflows/` | checks.yml（CI 门禁）、release.yml（PyPI/GHCR/.deb + SBOM 签名） |

组织方式评述：单包 src 布局，采用"Python 类定义层（tools/*.py，存量）+ YAML 目录层（catalog/*.yaml，增量）"双轨制——新工具只需一条 YAML 条目（CONTRIBUTING.md "catalog-first"），YAML 通过 `merge_into` 并入既有 Python 分类、通过 `overlay` 按标题增强既有工具（registry.py）。规模：src 约 10088 行 Python + 3196 行目录 YAML + 3972 行测试（tarball 流式统计）。

## 5. 核心架构与关键实现

### 5.1 数据驱动目录与 inert 加载（registry.py）

每个 YAML 文件声明 category 头 + tools 列表 + overlay；`CatalogTool` 把安装规格翻译成具体命令（`install: {go: ...}` → `go install -v ...`），`url` 安装被刻意排除在外（"no blind curl | bash"，等待 SHA-256 验证的 safe-fetch 路径）。最关键的安全设计是用户目录（`/find` 写入的 found.yaml）按 **inert 模式**加载——在读取端剥离一切可执行键，保证"GitHub 搜索结果永远不能变成可运行命令"：

```python
# src/hackingtool/registry.py
_EXECUTABLE_KEYS = ("install", "run", "system_pkgs", "kind", "url")

def _load_file(path: Path, inert: bool = False):
    data = yaml.safe_load(path.read_text(encoding="utf-8")) or {}
    entries = [e for e in (data.get("tools") or []) if isinstance(e, dict)]
    if inert:
        entries = [{k: v for k, v in e.items() if k not in _EXECUTABLE_KEYS}
                   for e in entries]
```

注释明确说明理由：不信任磁盘上的文件仍是 `/find` 写的那份（防篡改），在读取端强制不变量。

### 5.2 工具基类与执行面（core.py，727 行）

`HackingTool` 基类用类属性声明 TITLE/INSTALL_COMMANDS/RUN_COMMANDS/TAGS/USAGE/ARCHIVED 等，`__init__` 按能力组装 OPTIONS；`is_installed` 走"reuse-first"启发（system_pkgs.which 命中 PATH 即已装）。命令执行分两条路：目录/工具声明的 shell 一行（`_run_shell`，`shell=True` 但注明是用户自选命令，逐条写 `commands.log` 审计）；编排器与 `/goal` 走严格 list-form `subprocess`。URL 安装是"验证后执行"：

```python
# src/hackingtool/core.py  _url_install()
if not self.INSTALL_SHA256:
    console.print(f"[error]✘ Refusing url install for {self.TITLE}: no sha256 pinned. ...")
    return
with urllib.request.urlopen(self.INSTALL_URL) as resp:
    data = resp.read()
digest = hashlib.sha256(data).hexdigest()
if digest.lower() != self.INSTALL_SHA256.strip().lower():
    console.print("[error]✘ sha256 MISMATCH — refusing to run.[/error]\n" ...)
```

### 5.3 AI 层：封闭词表防幻觉（ai_recommend.py / ai_goal.py / skill/OPERATOR.md)

AI 只做"意图 → 标签"：提示词强制模型从 ALLOWED 标签列表回答 JSON 数组，解析端二次过滤（非法标签丢弃，ai_recommend.py `_parse_tags`）；无模型时退化为 stdlib 关键词匹配。`/goal` 把模型输出校验成 `Step(argv: list[str])`（ai_goal.py `_validate`，非法 argv 直接丢弃），执行时逐条 `[y]/[s]/[e]/[q]` 确认并要求确认授权；模型只调用一次做规划，工具输出不回喂。OPERATOR.md 章程以 `charter()` 注入所有 AI 提示（persona + 禁捏造 + `<scan_data>` 是数据不是指令）。

### 5.4 无头流水线（orchestrator.py + pipelines/recon.yaml)

`hackingtool --engagement acme --targets example.com --pipeline recon` 驱动确定性流水线，工具间用 stdin 链接（subfinder → httpx → nuclei），每步 list-form subprocess、30 分钟超时且超时后抢救已产出的记录，scope 外域名在转发前被剔除（engagement.scope_out），结果统一进 findings.json，报告由 report.py 确定性生成：

```yaml
# src/hackingtool/pipelines/recon.yaml
steps:
  - tool: subfinder
    args: ["-silent"]
    input: targets
    output: raw/subfinder.txt
    parser: subfinder
  - tool: httpx
    args: ["-silent", "-json"]
    input: previous
```

## 6. 构建与运行

来自 README.md 与 pyproject.toml（未实际运行，仅静态核对）：

```bash
# 推荐：pipx 从源码安装（需 Python 3.10+，Linux/macOS）
git clone https://github.com/Z4nzu/hackingtool.git && cd hackingtool
pipx install .            # 或 uv tool install . / venv+pip install .
hackingtool               # 默认 TUI；hackingtool --classic 进 REPL

# 开发
uv sync && uv run hackingtool
make setup                # 启用 pre-push 钩子
make check                # ruff（E9,F63,F7,F82,PLE,YTT）+ pytest（scripts/check.sh）
make test                 # uv run --group dev pytest -q

# Docker
docker run -it --rm hardikzinzu/hackingtool:latest
# 本地构建：docker build -t hackingtool . && docker run -it --rm hackingtool

# 无头模式
hackingtool --engagement acme --targets example.com --pipeline recon
hackingtool --engagement acme --report
```

## 7. 测试与工程质量

- 测试规模：34 个 `tests/test_*.py` 文件、约 3972 行、341 个测试函数（tarball 流式统计），纯 pytest，无 e2e；覆盖 AI 五模块、REPL、TUI 各屏幕/组件、目录一致性、会话、报告等。
- 目录即契约：test_registry.py 把 YAML 目录当作被测对象——所有标签必须在 TAXONOMY 内、resource 条目不得带 install/run、install 条目必须带 usage 速查，坏的 YAML 条目在 CI 变红而不是上线。
- 门禁单一源：scripts/check.sh 同时是本地 pre-push 钩子（Makefile `make setup` → .githooks/）与 CI 步骤（checks.yml），"green push = green CI"；ruff 只查必须修的类（E9/F63/F7/F82/PLE/YTT）。
- 供应链工程化程度在同类项目中罕见：release.yml 实现第三方 Action 全部 pin 到 commit SHA、SLSA 构建来源证明 + CycloneDX SBOM 签名（`gh attestation verify`）、PyPI Trusted Publishing（PEP 740）、GHCR 镜像、fpm 打包 .deb；dependabot 自动升级 pin。
- 代码质量观察：模块普遍带高密度设计注释（决策原因、安全不变量），核心循环用迭代而非递归（core.py show_options "no recursion, no stack growth"），错误路径（无 tmux、无模型、坏用户 YAML）都有降级分支；conftest.py 为空文件（0 字节）。uv.lock 在 README 提及但未包含在快照中（未确认原因）。

## 8. 社区与维护状态

- 活跃度：最近推送 2026-10-15（v3.1.0 发布当天的 TUI/自更新/设置大提交），此前 2026-07-26 发 v3.0.0 大重构、2026-08-23 加 context7.json（AI 文档检索集成）、2026-07 有 ImgBot 与文档维护提交（gh api commits）。v3.0→v3.1 相隔约 3 个月。
- Release 节奏：GitHub Releases 对象为 0（gh api），正式 tag 仅 v1.0/v1.1.0/v3.0.0；release.yml 已就绪（PyPI/GHCR/.deb + SBOM），但 README 中 PyPI/.deb 段落被注释为"等渠道上线后恢复"，故 PyPI 包与 GHCR 镜像是否已实际发布未确认；Docker Hub 的 `hardikzinzu/hackingtool:latest` 在 README 中是当前主推渠道。
- 社区结构：30 名贡献者、9179 fork、133 open issues、1483 watchers（gh api）；有 FUNDING.yml（GitHub Sponsors + BuyMeACoffee）、三类 issue 模板（bug/feature/tool_request）、PR 模板要求 `make check` 绿灯；v3.1.0 提交信息提到 "apply community bug fixes"，说明社区修复被批量吸收。赞助商横幅（WebVetted）出现在 README 顶部。
- 维护信号：scripts/audit_tools.py 会用 GitHub API 审计每个收录工具的维护状态并分类 KEEP/STALE/DEAD/ARCHIVE，44 个条目被归档（docs/TOOLS.md），说明目录在做主动清理而非只进不出。

## 9. 亮点与风险

亮点：
1. 从"脚本合辑"到"工程化产品"的重写教科书：src 布局、单一版本源、声明式目录、CI 门禁与 pre-push 同源、SLSA/SBOM 供应链签名（release.yml）。
2. AI 层的防幻觉设计闭环：封闭标签词表 + 解析端二次过滤 + 章程注入 + 工具输出按不可信数据消毒（ai_recommend.py、ai_summary.py、skill/OPERATOR.md），在"AI + 安全工具"方向上是少见的认真做法。
3. 安全默认值贯穿：SHA-256 验证后才执行的 URL 安装、拒绝 `curl | bash`、用户目录 inert 加载防篡改、命令执行全程审计日志（core.py `_cmd_logger`）。
4. 目录双轨制降低贡献门槛：新工具 = 一条 YAML，且被 conformance 测试强制质量（tests/test_registry.py）。
5. 三前端共享同一目录与语法（TUI/REPL/经典菜单），外加可复现的无头流水线，覆盖演示、日常与集成场景。

风险与局限：
1. 合规边界依赖用户自觉：项目所有"仅限授权目标"约束中，只有 `/find` 的意图过滤、`/goal` 的授权确认和编排器的 scope_out 是程序化强制；大量第三方工具本身（DDoS、钓鱼、RAT 类分类仍在目录中，src/hackingtool/tools/ddos.py、phishing_attack.py）的滥用风险由上游工具承担，hackingtool 的 lab_safe_notes 只是文字提示。用途不当可能违法，各司法辖区对未经授权的渗透测试有刑事边界。
2. 执行面仍有 shell=True 面：`_run_shell`（core.py）以 shell=True 运行目录/工具声明的字符串命令，注释自辩为"用户自选、非攻击者输入"，但若被诱导编辑目录（尤其 user 目录之外的 shipped YAML 被篡改的场景）仍扩大攻击面；旧 Python 类中也残留 `cd X;sudo ./x.sh` 式复合 shell 串（tools/information_gathering.py Dracnmap）。
3. 平台与生态局限：明确不支持 Windows；安装/运行依赖 apt/brew/go 等外部生态，Kali 之外发行版的覆盖程度参差；发布渠道（PyPI/.deb/GHCR）在快照时点尚未确认上线，README 与仓库状态存在少量漂移（uv.lock 缺失、版本计数注释）。
4. AI 质量依赖外部模型：无模型时仅退化为关键词匹配；`/goal` 的计划质量不受本仓库控制，且一次建模不做输出回验之外的语义校验。

## 10. 总结评价

hackingtool 是"GitHub 星数远超其代码体量"的典型样本（81k 星 vs 万余行 Python），但与多数星数虚高的工具合辑不同，它在 2026 年借 v3.0/v3.1 两次重写把社区期望转化成了真实的工程质量：声明式目录、封闭词表 AI、供应链签名发布、目录即测试。它的价值不在工具本身（全是第三方），而在"发现—安装—用法速查—后台运行—报告"这条工作流的一站化，以及把"授权使用"写进产品机制而不只是免责声明的努力。适合安全学习者在 Kali/Parrot 虚拟机里当工具导航与速查手册用，适合工程师把它当作"如何给一个脚本合辑做产品化重写"的参考实现；读源码优先看 registry.py（数据驱动 + inert 加载）、core.py（基类与执行面）、ai_recommend.py 与 skill/OPERATOR.md（AI 护栏）三处。红队/渗透从业者会嫌其编排深度不足，真正的对抗工作仍需专业平台；把它当作入口索引而非作战系统，是准确的定位。
