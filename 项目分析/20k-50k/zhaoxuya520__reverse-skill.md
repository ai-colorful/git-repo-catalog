# zhaoxuya520/reverse-skill 项目分析

> 星数 44159（快照 2026-10-24）｜主语言 PowerShell｜档位 20k-50k｜分析日期 2026-11-06

## 1. 项目速览

一句话定位：面向 AI 编码客户端（Claude Code / Codex / Cursor / Cline / Kiro 等）的网络安全技能路由包——用结构化路由配置把逆向/渗透/CTF 任务分流到对应的"技能模块"，再按需自举工具链，并把每次实战经验沉淀为可持续进化的经验库。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | zhaoxuya520/reverse-skill | gh api |
| 星数 / Fork | 44159 / 6559（gh api，2026-11-06 核实） | gh api |
| 主语言 | PowerShell（GitHub linguist；实际为 Markdown 为主 + 40 个 .ps1 脚本） | tarball 文件统计 |
| 许可证 | MIT | LICENSE、gh api |
| 版本 | 1.0.1（VERSION 文件；CHANGELOG 记录 1.0.1 于 2026-08-08 发布） | VERSION、CHANGELOG.md |
| 创建时间 | 2026-05-13 | gh api |
| 最后推送 | 2026-09-22 | gh api |
| 仓库定位 | 自声明"逆向/授权渗透测试/安全研究"用途，强调授权场景门控 | README.md、RULES.md |

## 2. 项目是什么，解决什么问题

这本质上是一个"给 AI Agent 用的安全作业方法论包"，由三类内容组成：**路由层**（把任务文本映射到正确的技能模块）、**技能层**（约 90 个 SKILL.md，每个描述一类场景的工具链与工作流）、**运维契约层**（scope 授权门、证据链、角色分工、时间线）。README.md 的 About 节给出的动机很直接：AI agent 拿到一个 APK/二进制/前端加密/CTF 题目时，不知道该用 jadx、apktool、Frida、IDA 还是 BurpSuite；工具、MCP 服务器、脚本散落在各台机器上；同样的错误反复出现因为经验没有复用。

它解决的四个痛点（README.md "Why this exists"）逐一对应四层机制：不知道用什么工具 → 路由矩阵 + tool-index；不同任务需要不同 playbook → 45 个领域 skill 模块；工具分散 → bootstrap 清单自举；经验不复用 → field-journal 经验库。README 中还给出了核心执行链：`User task → RULES.md → MASTER-ROUTING / master-route.ps1 → case-init/scope.md → Scenario skill → timeline + Evidence→Finding→Path → report + field-journal`。

与常见的"提示词集合"仓库不同，它把路由做成了**可回归测试的软件工程**：路由规则集中在 `skills/config/routing.json` 作为单一事实源，配 178 条中英双语基准用例（`skills/tests/routing-benchmark.json`），CI 在 Windows + Ubuntu 双平台验证 PowerShell 与 Bash 两个路由实现的行为一致（`.github/workflows/ci.yml`）。这与"纯 Markdown 提示词包"拉开了工程化差距。

值得强调的是仓库对"授权"的机制化处理：RULES.md 声明"读取仓库文件不等于授权执行"，README_AI.md 要求 AI 在首次副作用前展示完整命令计划并获得用户批准；case-init 生成的 scope.md 中 `auth.status=granted` 是对目标 ACT 的硬门，`-Force`/`--force` 参数被明确设计为**不能**绕过该门（RULES.md、skills/MASTER-ROUTING.md）。仓库自我定位为"轻量路由包"而非平台（skills/ops/IDENTITY.md）。

## 3. 技术栈与依赖

- **内容主体**：Markdown（tarball 内 447 个 .md），仓库本质是"文档即程序"——SKILL.md 既是 AI 的提示词也是行为规范。
- **PowerShell（40 个 .ps1）**：Windows 侧核心脚本——master-route.ps1（路由器）、case-init.ps1 / case-guard.ps1（案件初始化与门禁）、bootstrap-reverse.ps1（工具自举）、refresh-tool-index.ps1（工具索引刷新）、lib/ 下五个模块（ToolDiscovery、BootstrapSupplyChain、RouteScope、WorkRoot、HostRuntime），均要求 PS 5.1+ 且非 ASCII 脚本带 UTF-8 BOM（CHANGELOG.md Fixed 节）。
- **Bash（20 个 .sh）**：Linux/macOS/Kali 侧与 PS 平行的同契约实现（master-route.sh、case-init.sh、bootstrap-reverse.sh 等），CHANGELOG 明确要求"两平台路由契约相同，平台只改变执行入口"。
- **Python（8 个 .py）**：证据链审查器 review_case.py（--verify-hashes --strict）、文档链接校验 verify-doc-links.py 等。
- **Java（2 个 .java）**：burp-mcp-full/ 下的 Burp Suite 扩展（BurpMcpExtension.java、McpHttpServer.java），Gradle 构建，把 Burp 通过 MCP 协议暴露给 AI 客户端；另有 mcp-bridge.js 桥接层。
- **自举依赖清单**（skills/scripts/bootstrap-manifest.json）：对每个能力声明 bootstrapKind（github-release-zip / pip-package / winget-package / manual）、锁定版本与 assetSha256，如 jadx v1.5.6、apktool v3.0.2、frida-tools 14.10.4、pwntools 4.15.0；商业工具（JEB Pro、IDA、Binary Ninja）标 canAutoInstall=false。
- **CI**：GitHub Actions，Windows + Ubuntu 矩阵，actions checkout 固定到不可变 commit（ci.yml）。

## 4. 目录结构与代码组织

| 目录 | 职责 | 出处 |
|---|---|---|
| `skills/` | 核心：约 45 个领域技能模块 + 路由 + 脚本 + 契约 | tarball 结构 |
| `skills/config/routing.json` | 路由规则单一事实源（R0–R45，must/mustAll/exclude 正则语义） | config/routing.json meta.description |
| `skills/scripts/` | 平台原生脚本：路由/初始化/门禁/自举/测试（PS + sh 双实现） | 目录清单 |
| `skills/field-journal/` | 自进化经验库：21 条真实条目 + 17 条种子案例 + 索引/模板/脱敏规范 | field-journal/_index.md 统计节 |
| `skills/ops/` | 作战契约：IDENTITY、scope-contract、evidence-finding-path、role-map、skill-supply-chain 等 8 份 | 目录清单 |
| `skills/tests/routing-benchmark.json` | 178 条中英双语路由回归基准 | 文件内容统计 |
| `CTF-Sandbox-Orchestrator/` | 42 个 CTF 竞赛子技能，每个含 SKILL.md + agents/openai.yaml + references/*.md | README.md、tarball 结构 |
| `kali/` | Kali Linux 专属入口：README-kali、RULES-kali、独立 bootstrap/refresh 脚本 | kali/ 目录 |
| `burp-mcp-full/` | Java 版 Burp MCP 扩展 + Node 桥（Gradle 工程） | 目录结构 |
| `plugins/reverse-skill/` | 可选 Codex 适配器插件（.codex-plugin/plugin.json），不改变客户端中立核心 | CHANGELOG.md、plugins/ |
| `docs/` | 架构、平台指南、安全审计（PACKAGE-SECURITY-AUDIT、SECURITY-REVIEW-2026-09-03、UV-AND-DOWNLOAD-SECURITY） | docs/ 清单 |
| `examples/ctf-demo/` | 完整示例案件：scope → evidence → timeline → report 全流程 | examples/ 目录 |
| 顶层文件 | README（英/中/AI 版）、RULES（英/中）、AGENTS.md、CLAUDE.md、SECURITY.md、CHANGELOG、VERSION | tarball 结构 |

组织方式评述：单仓库单体结构，但内部职责边界清晰——"核心（路由+契约+脚本）客户端中立，客户端适配可选且外置"是被反复声明的架构原则（RULES.md "Client integration boundary"、CHANGELOG "Client-neutral integration contract"）。AGENTS.md / CLAUDE.md 这类指针文件只是引导 AI 先读 README_AI.md / RULES.md。全仓库 89 个 SKILL.md（tarball 统计），其中 skills/ 下 45 个左右，CTF 侧 42 个，插件侧 1 个。

## 5. 核心架构与关键实现

### 5.1 路由器：配置驱动 + 计分选择（skills/scripts/master-route.ps1）

路由不硬编码在脚本里，而是读取 `skills/config/routing.json`：每条规则含 `must`（正则命中）、`mustAll`（须同时命中多个子正则）、`exclude`（命中则否决，用于消歧，如"越狱"在 LLM 语境归 R14、iOS 语境归 R2）。脚本核心逻辑约 40 行：遍历规则计分 → 按 priority 数组顺序取最高分为主路由 → 并列时靠前者胜出 → 无命中回退 fallbackId（R0）。关键片段（master-route.ps1）：

```powershell
foreach ($p in $priority) {
    if ($scores.Contains($p)) {
        $score = $scores[$p]
        if ($score -gt $maxScore) { $maxScore = $score; $primary = $p }
    }
}
```

工程化细节到位：priority 与 routes 不一致时告警（防新增路由漏加优先级）；PRIMARY 技能文件缺失时 exit 2 而非崩溃；输出 `work/master-route-<ts>/route-scope.md` 供下游解析。`verify-routing-coherence.ps1` 会解析 MASTER-ROUTING.md 的优先级表并与 JSON 校验一致（routing.json meta.maintainers 声明）。

### 5.2 授权门与案件工作流（skills/scripts/case-init.ps1 / case-guard.ps1 + ops/scope-contract.md）

执行契约（skills/MASTER-ROUTING.md）强制顺序：先路由后动手 → case-init 落地 `work/<case>/scope.md`（auth + network_profile）→ auth 未 granted 禁止对目标 ACT → 打开 PRIMARY SKILL.md → 工具只认 tool-index.md。case-guard 在 ACT 前做轻量门禁（未就绪 exit 2），`-Force` 是兼容参数但不能绕过硬门。离线样本走 `offline-sample` preset（需显式提供样本路径），避免为纯本地分析伪造授权字段。这套"授权是数据而非口头承诺"的设计是仓库最鲜明的安全工程特征。

### 5.3 按需自举（skills/scripts/bootstrap-reverse.ps1 + bootstrap-manifest.json）

workflow 发现缺工具时不报错，而是调用平台原生 bootstrap；能力清单全部来自 manifest，每项含锁定版本、下载源、SHA256（github-release 类）或 GitHub API digest 校验，"校验失败即删除文件并失败"（CHANGELOG Security 节）。脚本侧通过 `lib/ToolDiscovery.ps1` 探测本机工具路径，`refresh-tool-index.ps1/sh` 生成 gitignore 的 `tool-index.md` 作为"本机工具唯一事实源"（RULES.md Important 节）。供应链门闩另见 ops/skill-supply-chain.md——外部 skill/MCP 安装需过审。

### 5.4 经验库自进化（skills/field-journal/ + .github/workflows/auto-merge-journal.yml）

机制分三层：**模板化沉淀**（_template.md 规定场景分类/脱敏 scope 摘要/角色/执行链路/Evidence 链摘要/Finding，字段与 review_case.py 契约对齐）；**脱敏规范**（anonymization.md + CONTRIBUTE-BACK.md 的二次检查清单：真实域名/IP/Token/PII 必须替换）；**社区回流自动化**（auto-merge-journal.yml 对 field-journal 路径的 PR 做 9 类机器检查——文件白名单、≤5 文件、prompt injection 正则、HTML/JS 注入、可执行代码伪装、base64 绕过、真实 API key 模式、非私有网段 IP、50KB 上限与 Markdown 结构——全部通过后自动合并）。当前库含 21 条真实条目 + 17 条种子案例，索引 _index.md 按场景分类，去重只需读索引（约 200-500 token，CONTRIBUTE-BACK.md）。

### 5.5 CTF 编排器（CTF-Sandbox-Orchestrator/）

42 个竞赛子技能（README.md 声明；tarball 中 42 个 competition-* 目录），每个统一三件套：SKILL.md（入口）、agents/openai.yaml（代理配置）、references/*.md（领域参考），主题覆盖 AD 证书滥用、Android hook、固件布局、取证时间线等竞赛方向，与主包 skills/ctf-sandbox 单入口路由衔接（skills/SKILL.md "CTF 入口"行）。

## 6. 构建与运行

无编译产物，"安装"即克隆 + 刷新工具索引（README.md Getting Started）：

```bash
git clone https://github.com/zhaoxuya520/reverse-skill.git
# Windows
powershell -File skills/scripts/refresh-tool-index.ps1
# Linux / macOS
bash skills/scripts/refresh-tool-index.sh
# Kali
bash kali/scripts/refresh-tool-index.sh
```

核心命令（skills/MASTER-ROUTING.md，未实际执行，仅静态确认）：

```powershell
powershell -File skills/scripts/master-route.ps1 -Hint "<用户任务>"          # 路由
powershell -File skills/scripts/case-init.ps1 -CaseName "my-case" -Preset offline-sample -Sample ".\app.apk"
powershell -File skills/scripts/case-guard.ps1 -CaseRoot work\my-case        # ACT 前门禁
powershell -File skills/scripts/smoke.ps1                                    # 冒烟：verify+解析+路由矩阵
python3 skills/case-review/scripts/review_case.py work/<case> --verify-hashes --strict
```

burp-mcp-full 侧：`gradlew` 构建（build.gradle）、`build.bat/build.sh`。注意 tool-index.md 在新 clone 中不存在（gitignore），首次需在用户批准后生成（README_AI.md 首次安装节）。

## 7. 测试与工程质量

- **路由回归基准**：`skills/tests/routing-benchmark.json` 178 条中英双语用例（文件统计），test-routing.ps1/sh 双平台跑；CHANGELOG 显示"加路由必须加用例"已成为 CI 强制门（verify-routing-coherence.ps1 反向检查无用例路由即 fail）。
- **脚本级测试**：skills/scripts/ 下十余个 test-*.ps1/sh（bootstrap-manifest、consolidate-evidence、p0-friction、parse-contracts、workflow-title-safety 等），CHANGELOG Unreleased 节专门记录了"把仓库里存在但从未被 CI 执行的测试接进 CI"的补课。
- **CI**：ci.yml Windows + Ubuntu 矩阵（routing-tests / case-contract job），macos-bash-compat.yml 专测 BSD sed 等 macOS 兼容性；INDEX.md 由 extract-summaries.ps1 从 SKILL.md frontmatter 生成并有 CI drift 检测；`bash -n` 语法检查。
- **安全工程**：docs/PACKAGE-SECURITY-AUDIT.md（包内可执行文件静态审计）、docs/SECURITY-REVIEW-2026-09-03.md、verify-repository-security.py、scan-leaks.ps1；PS 5.1 中文编码（UTF-8 BOM）被作为 CI 门（CHANGELOG）。
- 质量观察：文档与脚本的一致性有机器校验（doc-links、doc-facts、routing-coherence），这对一个"以 Markdown 为主体"的仓库尤其重要；CHANGELOG 条目粒度细到具体 winget id 不存在这类问题，维护严谨度高。

## 8. 社区与维护状态

- **增长曲线**：2026-05-13 创建，6 个月到 44.2k 星（gh api），README 挂 trendshift 徽章（README.md），属于超高速增长的安全类内容仓库；star-history 由 update-star-history.ps1 定时刷新（skills/scripts/）。
- **贡献者**：15 人（gh api contributors），主体明显是单人维护 + 社区 field-journal PR 回流（auto-merge 工作流的存在即是为此设计）。
- **发布节奏**：仅 1 个 release/tag（v1.0.1，2026-08-08，gh api），日常迭代走 main + CHANGELOG Unreleased 节；最后推送 2026-09-22，距分析日约 6 周无新提交。
- **社区生态**：README_AI.md 链接 linux.do 社区；赞助商 3 家（UCloud AstraFlow、Atlas Cloud、Kite AI，README.md Sponsors 节）；docs/reviews/ 与 docs/plans/ 留有 PR 整合评审与设计文档（如 2026-09-21 pwntools-kali-discovery 设计稿）。
- **issue**：41 个 open（gh api），docs/COMMUNITY-ISSUE-TRIAGE.md 有分诊文档。

## 9. 亮点与风险

**亮点**

1. 路由即软件工程：单一事实源 JSON + 178 条双语回归基准 + 双平台 parity 测试 + 一致性校验脚本，把"提示词包"做成了可回归验证的系统（skills/config/routing.json、skills/tests/routing-benchmark.json、ci.yml）。
2. 授权机制化：scope.md 数据化授权门 + case-guard 硬门 + `-Force` 不能绕过的显式设计，把合规要求内嵌进执行链而非停留在免责声明（RULES.md、skills/MASTER-ROUTING.md）。
3. 供应链自控：自举清单全量锁版本 + SHA256/digest 校验 + 商业工具禁止自动安装 + skill-supply-chain 外部技能门闩（skills/scripts/bootstrap-manifest.json、skills/ops/skill-supply-chain.md）。
4. 经验库闭环：模板 → 脱敏 → 自动审核合并的社区回流管道，且用 GitHub Actions 实现 9 类机器检查，低 token 去重设计务实（field-journal/CONTRIBUTE-BACK.md、auto-merge-journal.yml）。
5. 客户端中立架构：核心不写任何客户端全局配置，适配器外置可选，避免绑定单一 AI 客户端（RULES.md Client integration boundary）。

**风险 / 局限**

1. **滥用与合规风险**：仓库编排的能力面覆盖渗透测试全链（含 EDR 对抗、AD 攻击路径、LLM 越狱测试等模块，见 skills/SKILL.md 模块表）。仓库自声明面向授权测试并通过 scope/auth 门控约束，但其约束依赖 AI 客户端遵循文本契约，没有技术手段能阻止用户在未授权目标上套用方法论——使用者的法律合规责任无法被仓库卸载。
2. **提示词注入面**：整个体系依赖 AI 忠实执行 RULES.md/SKILL.md 中的指令；这类"仓库文本指挥 Agent"的形态天然是注入攻击面（仓库自身在 auto-merge 工作流里防注入，也说明作者意识到该威胁，但无法根除）。第三方 fork/镜像可能植入恶意指令，下载应认准官方源。
3. **单人维护 + 低频发布**：15 个贡献者中实际维护高度集中于作者，近 6 周无推送，41 个 open issue 未处理；技能内容的时效性（工具版本、MCP 生态）依赖持续维护。
4. **Windows 重度假设**：虽有三平台脚本，但工具自举、路径探测、编码处理大量围绕 Windows/PowerShell 5.1 设计，Linux/macOS 体验依赖 parity 测试覆盖程度（master-route.ps1 头部注释、CHANGELOG 编码修复记录）。

## 10. 总结评价

reverse-skill 是"AI Agent + 安全作业"这一新兴形态中工程化程度最高的开源实现之一：它没有发明任何新工具，而是把散落的方法论、工具链、授权契约和实战经验组织成一个可路由、可测试、可自举、可进化的系统，且在授权门控与供应链安全上表现出超出同类仓库的自觉。6 个月 44k 星的增长反映了 AI 辅助安全研究需求的真实爆发。它适合两类人：一是用 Claude Code/Codex/Cursor 等客户端做授权安全测试或 CTF 的从业者，可直接安装使用；二是研究"提示词工程如何软件工程化"的开发者，其路由 SSoT、回归基准、经验库自动回流管道都是值得读源码借鉴的设计。反过来说，指望它是"开箱即用的攻击武器"或"一劳永逸的自动化渗透平台"都不符合其自我定位——skills/ops/IDENTITY.md 说得很清楚：它是一个路由包，不是平台。
