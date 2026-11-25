# tw93/Mole 项目分析

> 星数 72546（快照 2026-10-24）｜主语言 Shell（实际为 Shell+Go 混合）｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：开源免费的 macOS 终端系统维护工具箱——深度清理、应用卸载、磁盘分析、系统优化与健康监控合而为一的 `mo` 命令，对标 CleanMyMac + AppCleaner + DaisyDisk + iStat Menus 的 CLI 集合体。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | Mole | README.md |
| 星数 | 73,383（2026-10-31 实时，任务快照 72,546） | gh api repos/tw93/Mole |
| 主语言 | Shell（~14.1 万行 sh/bash/bats）+ Go（~2.9 万行/74 文件） | 源码统计 |
| 许可证 | GPL-3.0 | LICENSE、gh api |
| 创建时间 | 2025-09-23 | gh api |
| 最近 push | 2026-10-30 | gh api |
| 最新版本 | V1.66.0（2026-10-29 发布，共 30 个 release） | gh api releases |
| 贡献者 | 165 人（作者 Tw93 主导，闭环 AI 辅助开发） | gh api contributors |

## 2. 项目是什么，解决什么问题

Mole 解决 macOS 用户"系统越来越满、垃圾越积越多，但商业清理软件又贵又不透明"的问题。README.md 开篇即点明定位：把 CleanMyMac、AppCleaner、DaisyDisk、iStat Menus 四类工具的工作流合并进一个终端命令。仓库本体是免费开源的 CLI（`mo`），同作者另有付费原生应用 Mole Mac（https://mole.fit）与 Mole for Windows，属于"开源 CLI 获客 + 付费 App 变现"的双产品策略（README.md、llms.txt）。

功能面覆盖七条主线（README.md "Run" 与 "Features in Detail" 章节、bin/ 目录）：

- `mo clean`：深度清理缓存、日志、已卸载应用残留、开发工具链缓存（Xcode DerivedData、Homebrew、Conda、AI CLI 缓存等，见 lib/clean/ 下 10 个模块）；
- `mo uninstall`：连带 LaunchAgents、偏好设置、残留目录整体卸载应用；
- `mo analyze`：交互式 TUI 磁盘占用浏览器，可按目录下钻并直接删除大文件；
- `mo optimize`：刷新 DNS、QuickLook、图标缓存、修复系统数据库等有界维护任务；
- `mo status`：实时系统健康仪表盘（CPU/内存/磁盘 IO/网络/进程/电池/GPU）；
- `mo purge` / `mo installer` / `mo slim`：清理项目构建产物、安装包文件、去除通用二进制中的 Intel 切片；
- `mo touchid` / `mo completion` / `mo update` / `mo remove`：配套周边。

值得注意的是它把"安全"作为第一性设计目标：README.md Safety 章节声明"当 Mole 无法证明某项操作安全时，就跳过或拒绝"；每个有删除能力的子命令都支持 `--dry-run` 预览，卸载/分析/瘦身类删除默认进废纸篓可恢复，清理类永久删除但有白名单保护（`mo clean --whitelist`）。另一条特色是"AI 代理友好"：仓库自带 llms.txt、Claude Code 插件市场清单（.claude-plugin/marketplace.json）与技能包（skills/mole/），可让 Claude Code、Codex、Cursor 等 agent 用 `--dry-run` 先预览、再由人确认的方式代为执行清理——这是它在 AI 工具社区快速传播的差异化卖点。

## 3. 技术栈与依赖

- **Shell（主体）**：Bash（要求兼容 macOS 自带 bash 3.2，见 mole 头部注释与 .claude/rules/shell.md），lib/ 下约 14 万行，配 shellcheck（.shellcheckrc、.golangci.yml 之外还有独立 shell lint，scripts/check.sh）。
- **Go 1.26**（go.mod）：两个独立二进制，均 `//go:build darwin` 限定 macOS：
  - `cmd/analyze`：bubbletea v1.3.10 + lipgloss v1.1.0 构建交互式 TUI（磁盘扫描复用系统 `/usr/bin/du` 与 `mdfind` 聚焦查询，scanner.go）；
  - `cmd/status`：gopsutil/v4 v4.26.9 采集 CPU/内存/磁盘/网络/GPU/电池指标（cmd/status/metrics_*.go）。
  - 其余直接依赖仅 xxhash（快速哈希）与 golang.org/x/sys，依赖面极克制（go.mod）。
- **测试框架**：BATS（bats-core）测 shell、`go test` 测 Go，另有 Python 脚本做仓库自审计（scripts/audit_*.py）。
- **打包分发**：Homebrew formula（release.yml 打包 binaries）、curl 安装脚本 install.sh（70KB，含 checksum 校验）、Nix flake（flake.nix、package.nix、nix.yml CI）。
- **CI**：GitHub Actions 7 个 workflow（test/check/release/codeql/nix/bundle_audit/update-contributors，.github/workflows/）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 | 出处 |
|---|---|---|
| `mo` / `mole` | 入口：`mo` 仅是别名转发；`mole` 是主 bash 调度器，路由子命令与交互菜单、拒绝 root 运行 | mole, mo |
| `bin/` | 11 个子命令脚本（clean/uninstall/analyze/optimize/status/purge/installer/slim/touchid/completion/history） | bin/ |
| `lib/core/` | 核心库：file_ops.sh（删除漏斗 4244 行）、app_protection.sh（系统保护 2787 行）、sudo、timeout、history、ui、log 等 16 个模块 | lib/core/ |
| `lib/clean/` | 清理域模块：caches/dev/app_caches/apps/system/user/brew/hints/project 等 | lib/clean/ |
| `lib/uninstall/` `lib/optimize/` `lib/manage/` `lib/check/` | 卸载、优化、自更新管理、自检库 | lib/ |
| `lib/ui/` | bash 侧交互菜单（分页菜单、应用选择器） | lib/ui/ |
| `cmd/analyze` `cmd/status` | Go TUI 二进制（磁盘分析器、健康仪表盘），34 个 `*_test.go` | cmd/ |
| `tests/` | 74 个 BATS 测试文件，2812 个 @test 用例 | tests/ |
| `scripts/` | 测试编排（test.sh/check.sh）+ 仓库自审计脚本（audit_destructive_sinks.py、audit_bats_assertions.py、e2e_macos.py 等 14 个） | scripts/ |
| `skills/mole/` `plugins/mole/` `.claude/` `.agents/` | 面向 AI agent 的技能/插件分发与开发规则 | 各目录 |
| `AGENTS.md`（34KB）`.claude/rules/`（13 个区域规则） | AI 辅助开发的"宪法"：按文件路径注入的区域规约 | AGENTS.md |
| `SECURITY_AUDIT.md`（43KB）`docs/SECURITY_DESIGN.md` `docs/incidents.md` `docs/weekly-audit.md` `SYNC.md`（38KB） | 安全审计报告、威胁模型、事故记录、每周审计日志、四产品线同步台账 | 根与 docs/ |

组织方式评述：单仓库、双语言、分层清晰——bash 承担全部业务逻辑（领域模块按功能域切分，core 层提供可复用原语），Go 只做 bash 不擅长的两块交互式 TUI；`mo` 命令运行时按需 source 对应 lib 模块。这是一个用工程化纪律武装到牙齿的"超大号 bash 项目"：最大的单文件 file_ops.sh 有 4244 行，AGENTS.md 明确警告"热点文件刻意保持大，不要上来就拆"（AGENTS.md）。

## 5. 核心架构与关键实现

### 5.1 统一删除漏斗 lib/core/file_ops.sh（`mole_delete()`，2396 行起）

所有破坏性操作收敛到一个入口：五种参数（路径、是否需 sudo、期望文件身份、期望 SHA256、期望消失路径）+ 两种模式（`permanent | trash`），先校验再删除，拒绝事件写入取证日志以便审计区分"策略拒绝"与"从未尝试"（docs/SECURITY_DESIGN.md）：

```bash
mole_delete() {
    local path="$1"
    local needs_sudo="${2:-false}"
    local expected_identity="${3:-}"
    local expected_file_sha256="${4:-}"
    local expected_absent_path="${5:-}"
    local mode="${MOLE_DELETE_MODE:-permanent}"
    [[ -z "$path" ]] && return 1
    case "$mode" in
        permanent | trash) ;;
        *) ... return 1 ;;  # 未知模式拒绝并记日志
    esac
```

配套的 `lib/core/app_protection.sh` 维护系统关键组件正则（TCC、loginitems、controlcenter 等）与受保护应用 bundle id 数据表（app_protection_data.sh），并检测 Xcode 构建进程占用以跳过活跃缓存。

### 5.2 Go 磁盘分析器 cmd/analyze/

扫描层（scanner.go）用 worker 池并发调 `/usr/bin/du` 并用 Spotlight（`mdfind -onlyin`）辅助定位，扫描失败只保留首个错误与首个瞬时错误，避免超大目录把超时掩盖掉；删除层（delete.go）实现"废纸篓优先"三级回退：Finder > filesystem API > 独立二进制（`moveToTrash*` 函数族），并内置双重路径保护（`isProtectedAnalyzeDeletePath` / `isCriticalAnalyzeDeletePath`，数百行白名单判断），路径校验有 fuzz 测试压阵（delete_fuzz_test.go）。

### 5.3 实时监控 cmd/status/

按指标族拆文件（metrics_cpu/memory/disk/network/gpu/battery/health/process 等），gopsutil 采集 + `view.go` 渲染 TUI，同时支持 JSON/NDJSON 稳定输出供脚本消费（README.md Status 章节）；`prefs.go` 持久化用户偏好。只读设计，AGENTS.md 明确"不做成 iStat 克隆或告警守护进程"。

### 5.4 AI-Native 工程体系（本仓库最具特色的实现）

AGENTS.md（34KB）+ `.claude/rules/`（13 个按路径 frontmatter 注入的区域规则）+ `.claude/skills/`（bugs/release-flow/release-notes/weekly-audit，含 evals 与事故案例库）构成完整 agent 开发协议；`docs/weekly-audit.md` 记录每周自动审计的提交窗口与遗留队列，`SYNC.md`（38KB）维护 macOS CLI/Windows CLI/Mole Mac/Mole for Windows 四条产品线的同步台账。SECURITY_AUDIT.md 的 V1.66.0（2026-10-30）复审展示了成果：记录 4 个 Medium 级安全缺陷的发现、修复与回归测试证据（如"删除进行中隐藏操作仍可触发""符号链接绕过物理白名单根""Jupyter 活跃运行时被当作可清缓存"）。

## 6. 构建与运行

来自 Makefile 与 README.md，均为项目文档声明的实际命令：

```bash
# 构建（本地架构，产物 bin/analyze-go、bin/status-go）
make build            # = go mod download + go build -ldflags="-s -w"
# 检查与测试
make check            # ./scripts/check.sh --no-format（shellcheck 等静态检查）
make test             # MOLE_TEST_NO_AUTH=1 ./scripts/test.sh（BATS）
make test-go          # go test ./...
make verify           # check + test-go
# 发布（tag 触发，amd64/arm64 矩阵，CGO_ENABLED=0 纯 Go）
make release-amd64 / make release-arm64
# 用户安装与运行
brew install mole     # 或 curl -fsSL https://raw.githubusercontent.com/tw93/mole/main/install.sh | bash
mo clean --dry-run    # 预览模式运行任一清理命令
```

CI 矩阵：test.yml 在 macos-latest 上跑 BATS 全量（bats-core + shellcheck + coreutils + parallel + fd，6 并行），另有 macos-15 / macos-26 兼容性矩阵；release.yml 发布前强制校验"tag 与源码版本一致"且该 commit 上 Check/Validation/CodeQL/Nix 四项 workflow 全绿，否则拒绝出包。

## 7. 测试与工程质量

- **测试规模**：shell 侧 74 个 BATS 文件、2812 个 `@test` 用例（实测 grep 统计），按命令/域精细拆分（clean_core、uninstall_safety、path_validation_fuzz、install_checksum、manage_sudo……）；Go 侧 34 个测试文件，含 fuzz 与 TUI 生命周期测试（cmd/analyze/lifecycle_test.go 等）。
- **测试文化**：SECURITY_AUDIT.md 每条修复都绑定"回归证据"（指出旧代码下哪些测试失败）；scripts/audit_bats_assertions.py 审计测试断言质量，防"假通过"；scripts/audit_destructive_sinks.py 静态扫描破坏性操作的出口是否都过了删除漏斗。
- **CI**：7 个 workflow，含 CodeQL、bundle 漂移审计（bundle_audit.yml）、Nix 构建验证；发布门禁要求多 workflow 全绿（release.yml）。
- **静态质量**：shellcheck 强制 + .golangci.yml；代码中大量防御性注释解释"为什么"（如 mole 中对 bash 3.2 BASH_SOURCE、BSD readlink 无 -f 的兼容处理），可读性远超一般 shell 项目。
- e2e：scripts/e2e_macos.py 提供 macOS 端到端验证。

## 8. 社区与维护状态

- **极度活跃**：创建于 2025-09-23，约 13 个月达到 7.3 万星；最近 push 2026-10-30（昨日）；V1.65.0 到 V1.66.0 仅隔 2 天，30 个 release 说明高频迭代节奏（gh api）。SECURITY_AUDIT.md 与 weekly-audit.md 的时间戳（2026-10-26/10-30）与源码提交哈希相互印证，维护日志密度罕见地高。
- **贡献者结构**：165 名贡献者（gh api）；CONTRIBUTORS.svg（2.1MB）自动生成的贡献者视觉墙，update-contributors.yml workflow 自动维护。但代码主线（AGENTS.md、审计日志的提交哈希署名模式）表明核心决策高度集中于作者 Tw93 本人，AI agent 承担了大量实现与审计工作。
- **问题治理**：open issues 仅 1（gh api），配套 .github/ISSUE_TEMPLATE（含 mac_app 专用模板）与 CODEOWNERS；README 有 Telegram 社群与 Twitter 入口。
- **商业化**：CLI 开源引流，Mole Mac / Mole for Windows 付费 App（mole.fit），README 与 llms.txt 均有明确导流。

## 9. 亮点与风险

**亮点**

1. 安全工程教科书级：统一删除漏斗 + 应用/路径保护数据表 + 白名单兜底（读不到白名单即中止）+ 默认进废纸篓 + --dry-run 全覆盖 + fuzz 测试 + 公开安全审计报告与威胁模型（docs/SECURITY_DESIGN.md、SECURITY_AUDIT.md）。
2. AI-Native 开发范式范本：34KB AGENTS.md、路径注入式区域规则、技能/插件市场分发、每周自动审计与四产品同步台账，展示了"人主导 + agent 大规模执行"的可审计协作模式（AGENTS.md、docs/weekly-audit.md、SYNC.md）。
3. 测试密度极高：2812 个 shell 用例 + 34 个 Go 测试文件，覆盖安装校验、sudo、超时、路径 fuzz 等边缘面。
4. 产品与传播嗅觉敏锐：8 语言 README、llms.txt 让 agent 自助安装、Claude Code/Codex 双插件市场、开源 CLI + 付费 App 的清晰商业模式。
5. 工程克制：Go 侧直接依赖仅 4 个，复用系统 du/mdfind 而非自研扫描。

**风险/局限**

1. 平台绑定强：核心 bash 库大量 BSD/macOS 专有假设（`/usr/bin/du -I`、mdfind、pbcopy 等，scanner.go 注释自述），Windows CLI 在另一分支（windows），Linux 不可用。
2. 复杂度沉淀在巨型文件：file_ops.sh 4244 行、app_protection.sh 2787 行，bash 3.2 兼容约束进一步抬高改动门槛，外部贡献者难以介入核心层（AGENTS.md 也明确不建议拆分）。
3. 删除类工具固有风险：审计报告自认底层仍是基于路径的操作，无法对抗"并发敌意替换"（TOCTOU 残留），安全声明依赖测试与流程而非原子性保证（SECURITY_AUDIT.md）。
4. AI 流程文档体量已超过部分业务代码，规则、审计、同步台账的维护本身是不小的持续成本；GPL-3.0 对想借用代码的商业工具不友好。

## 10. 总结评价

Mole 是 2025-2026 年"AI 辅助工程"浪潮中最有代表性的个人作品之一：Tw93 以 bash+Go 的朴素技术栈，配合同事级规模的 AI 工程纪律（规则库、审计链、同步台账、2800+ 测试），在 13 个月内把一个 macOS 清理 CLI 做成 7 万星、带付费 App 的完整产品。它的源码价值在两端——想学"如何用 shell 写出可靠系统工具"的人可以研究它的删除漏斗与保护层设计；想实践"如何管理 AI 大规模参与的开发"的人，AGENTS.md + .claude/rules/ + weekly-audit.md 这套协议本身就是最好的开源教材。适合 macOS 重度用户直接安装使用（记得先 --dry-run），适合工具开发者与 AI 工程实践者阅读源码。
