# affaan-m/ECC 项目分析

> 星数 287044（快照 2026-10-24）｜主语言 JavaScript｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：ECC 是一个"agent harness 性能优化操作系统"——把 71 个子代理、311 个技能、97 个命令、hook 强制门、跨 harness 记忆库和 Rust 控制平面打包成一套可安装进 Claude Code / Codex / Cursor 等 10+ 编码代理的内容与运行时体系。

| 事实 | 值 | 出处 |
| --- | --- | --- |
| 名称 | ECC（npm 包 `ecc-universal`） | `package.json`、`VERSION` |
| 版本 | 2.2.3（2026-10-01 发布） | `VERSION`、CHANGELOG.md |
| 星数 | 287044（快照）；gh api 实测 292476 | 任务参数、gh api repos/affaan-m/ECC |
| Forks | ~43286；watchers 1426；open issues 260 | gh api |
| 主语言 | JavaScript（GitHub 标记）；实际为 JS+Rust+Python 混合 | `package.json`、`ecc2/Cargo.toml`、`pyproject.toml` |
| 许可证 | MIT（Copyright 2026 Affaan Mustafa） | `LICENSE` |
| 创建时间 | 2026-01-18，最近 push 2026-10-26 | gh api |
| 贡献者 | 约 457 人（含匿名，gh api 分页计数）；README 自述"单一维护者"驱动 | gh api、README.md |
| 社区 | Discord、ecc.tools 官网、GitHub App（ecc-tools） | README.md |

## 2. 项目是什么，解决什么问题

ECC 的核心论点写在 README：你的 agent 会写代码，但没有"协调一致的工程系统"——它缺规划、缺验证、缺自审、缺记忆。ECC 把 `plan -> test -> implement -> review -> verify -> remember -> improve` 这条流水线一次性安装进 agent harness（README.md），口号是 "Optimize the context window. Persist everything else."。

它分三层交付：

1. **内容层**：311 个技能（`skills/`，SKILL.md 形式）、71 个子代理定义（`agents/*.md`）、97 个命令入口（`commands/`）、23 个规则包（`rules/`），覆盖 TDD、安全审查、前端/后端模式、运维、科研等场景。README 明确说正"走向 skills-first 表面"，命令多为 legacy shim。
2. **运行时层**：Node 编写的 hook 与 CLI。GateGuard 是 PreToolUse 强制门——第一次 Edit/Write/Bash 直接 DENY，要求 agent 先给出具体事实（谁 import 了这个文件、数据 schema 是什么、用户原话指令），调查本身制造上下文（`scripts/hooks/gateguard-fact-force.js` 头注释、`skills/gateguard/SKILL.md`）。记忆库（`scripts/lib/memory-vault.js`）把可移植的 `ecc.memory.v1` Markdown 文档存到本地，供 Claude/Codex/Hermes 等跨 harness 共享，带路径安全、密钥检测、符号链接防护。
3. **控制平面层**：`ecc2/` 是 Rust 写的 ECC 2.0 alpha——ratatui TUI 仪表盘 + SQLite 会话存储 + 后台 daemon + worktree 感知的多会话管理（`ecc2/README.md`、`ecc2/Cargo.toml`）。

安装形态是关键卖点：`npx ecc-universal setup` 引导式安装，或 Claude Code 原生 `/plugin marketplace add https://github.com/affaan-m/ECC` + `/plugin install ecc@ecc`（README.md）。另有 ECC Pro 托管 GitHub App（私有仓库 $19/座/月）作为商业化层（README.md、SPONSORS.md），CodeRabbit、Greptile、Moonshot(Kimi)、SerpApi 等为赞助方。

## 3. 技术栈与依赖

- **Node.js >= 18**：主体。运行时依赖极克制——`ajv`（schema 校验）、`zod`、`js-yaml`、`@iarna/toml`、`sql.js`（SQLite in JS）共 5 个（`package.json` dependencies）。
- **Rust**：`ecc2/` 用 ratatui + crossterm（TUI）、tokio（异步）、rusqlite（状态库）、git2（git 集成）、clap（CLI），release profile 开 LTO（`ecc2/Cargo.toml`），Rust 代码约 5.46 万行。
- **Python >= 3.11**：`src/llm` 是 provider 无关的 LLM 抽象层（providers：claude/openai/ollama/openai_compat/astraflow/atlas，`src/llm/providers/`），依赖 anthropic、openai SDK（`pyproject.toml`）；另有根目录 `ecc_dashboard.py`（约 4.1 万字节的终端仪表盘）。
- **工具链**：Yarn 4.9.2、ESLint 10 + markdownlint、c8 覆盖率、pytest/ruff/mypy（Python 侧）（`package.json`、`pyproject.toml`）。

## 4. 目录结构与代码组织

| 目录 | 职责 | 出处 |
| --- | --- | --- |
| `skills/` | 311 个技能目录（6.1MB），每个含 SKILL.md | `ls skills \| wc -l` |
| `agents/` | 71 个子代理 Markdown 定义（architect、code-reviewer、build-error-resolver 等） | `ls agents` |
| `commands/` | 97 个命令入口（legacy shim） | `ls commands \| wc -l` |
| `rules/` | 23 个按语言/项目选装的规则包 | `ls rules \| wc -l` |
| `scripts/` | 356 个 JS 文件约 10 万行：`ecc.js` CLI 分发（setup/install/plan/catalog/consult/doctor/repair/memory/status 等 20+ 子命令）、`scripts/hooks/`（40+ hook）、`scripts/lib/`（112 个模块） | `find scripts` 统计、`scripts/ecc.js` |
| `hooks/` | hooks.json + codex/qoder/hermes 各 harness hook 配置 + memory-persistence | `hooks/` |
| `ecc2/` | Rust 控制平面 alpha（54638 行 .rs） | `ecc2/src/` |
| `src/llm/` | Python LLM 抽象层 | `pyproject.toml` |
| `pi/core/` | 为 Pi 打包器生成的精简画像（123 技能+24 命令，无网络依赖） | CHANGELOG 2.2.2、`pi/core/` |
| `docs/` | 1527 个 Markdown，含 16 种语言 README 翻译 | `find docs -name '*.md' \| wc -l` |
| `tests/` | 488 个测试文件（443 个 JS `.test.js` + 26 个 pytest） | `find tests` 统计 |
| `.github/` | 16 个 workflow + 问题/PR 审查策略 | `.github/workflows/` |
| 其余 | `manifests/`、`schemas/`、`mcp-configs/`、`docker/`、`examples/`、`integrations/`、`.claude-plugin/`（插件清单） | 顶层 `ls` |

组织方式评述：单仓库、内容与运行时同体。内容（skills/agents/commands）是"资产"，scripts 是"引擎"，两者通过 manifest/profile/catalog 系统松耦合——`package.json` 的 `files` 白名单直接把全部技能目录打进 npm 包。这是一个刻意为多 harness 分发设计的 monorepo，而非传统应用代码库。

## 5. 核心架构与关键实现

### 5.1 GateGuard 事实强制门（scripts/hooks/gateguard-fact-force.js）

PreToolUse hook，设计动机直白（文件头注释）：问 LLM"你确定吗？"永远得到"是"，改为要求具体事实。三段式 `DENY -> FORCE -> ALLOW`，技能文档称 A/B 测试 +2.25 质量分（`skills/gateguard/SKILL.md`）。实现上体现明显的性能自觉：模块懒加载、已检查文件键缓存、只解析命名目标的搜索（CHANGELOG Unreleased 段）、只读 shell 命令不计入门额（`scripts/hooks/gateguard-fact-force.js`）：

```js
function lazily(load) {
  let loaded = null;
  return () => loaded || (loaded = load());
}
const targetClass = lazily(() => require('../lib/gateguard-target-class'));
const turnScan = lazily(() => require('../lib/gateguard-turn-scan'));
const changeProfile = lazily(() => require('../lib/gateguard-change-profile'));
```

敏感目标（.env、密钥、CI workflow 等）不享受任何豁免，且在 subagent 中同样拦截（CHANGELOG Unreleased）。`hooks/hooks.json` 中每个 hook 都内嵌一段压缩的引导 one-liner，负责解析 `CLAUDE_PLUGIN_ROOT`（含 WSL 路径翻译与插件缓存目录探测）——这是为了适配 Claude 插件的各种安装位置。

### 5.2 记忆库（scripts/lib/memory-vault.js + scripts/memory.js）

跨 harness 的持久上下文层，存储格式是带 frontmatter 的 `ecc.memory.v1` Markdown 文档（`skills/unified-memory/SKILL.md`），支持 project/team/user 三个 scope、save/search/init 等 CLI 子命令（`scripts/memory.js` usage）。安全设计密集：`assertWithinTrustedRoot` 路径约束、`findPotentialSecrets` 入库前密钥检测、限制 MAX_FILES=5000 / MAX_SCAN_BYTES=16MB、拒绝符号链接（`scripts/lib/memory-vault.js` 头部常量）。配套 `scripts/memory-mcp.mjs` 把同一能力暴露为 MCP server，实现"Claude 存、Codex 取"。

### 5.3 ECC 2.0 控制平面（ecc2/，Rust）

定位是"单个 harness 安装之上的那一层"：多会话统一管理、风险评分、worktree 编排（`ecc2/README.md`）。架构要点：会话 runner 是独立进程，**SQLite 数据库是跨进程唯一事实源**，仪表盘启动时从库水合，稳态刷新只读单调游标之后新增的行（`ecc2/README.md`）。自评 alpha 质量，`cd ecc2 && cargo run -- dashboard` 可直接跑。

### 5.4 安装与治理系统（scripts/ci/ + manifests/ + .claude-plugin/）

ECC 的另一核心是"内容供应链"：`manifests/` 定义 lean/full 画像与 per-harness 清单，`scripts/ci/validate-skills.js` 等 14 个校验器在 `npm test` 中依次执行（frontmatter、宿主兼容、Unicode 安全、无个人路径、catalog 一致性……，`package.json` test 脚本）；`scripts/build-pi-core.js` 确定性生成 pi/core 画像，构建对不安全内容（非白名单 URL、pipe-to-shell、机密、绝对家目录路径、软链）硬失败（CHANGELOG 2.2.2）。

## 6. 构建与运行

来自 README 与 `package.json`：

```bash
# 安装（用户侧，二选一，不要叠加）
npx ecc-universal@2.2.3 setup                      # 引导式 Claude 插件安装
# 或在 Claude Code 内：/plugin marketplace add https://github.com/affaan-m/ECC 然后 /plugin install ecc@ecc
npx ecc-universal@2.2.3 install --guided --harness claude --harness codex   # 多 harness 向导

# 开发者侧（仓库内）
npm install
npm test                 # 14 个校验器 + catalog 检查 + node tests/run-all.js
npm run test:python      # pytest tests/ -q
npm run coverage         # c8，行/函数/分支/语句 80% 硬门
npm run lint             # eslint . && markdownlint
node scripts/ecc.js doctor    # 诊断漂移；repair / auto-update / memory / consult 同理
cd ecc2 && cargo run -- dashboard   # Rust 控制平面
python3 ./ecc_dashboard.py          # Python 仪表盘
```

## 7. 测试与工程质量

- **规模**：488 个测试文件，其中 443 个 JS `.test.js`（自带 runner `tests/run-all.js`，支持 worker 池 `ECC_TEST_JOBS`、SKIP/ONLY/SERIAL 环境变量，串行队列保护共享 checkout）+ 26 个 pytest 文件（`tests/`、`pyproject.toml`）。
- **风格**：以"清单校验 + 单元/集成"为主——大量测试针对 hook 行为、配置合并、插件清单（如 `tests/opencode-config.test.js`、`tests/plugin-manifest.test.js`、`tests/codex-native-hooks.test.js`），Python 侧覆盖 `src/llm` 各 provider。
- **CI**（`.github/workflows/ci.yml`）：test、pack-installer、packed-install-lifecycle、validate、pi-core（离线加载验证）、python-tests、security、coverage、lint 共 10 个 job；另有 release.yml（SLSA3 generic generator）、supply-chain-watch.yml、pr-burst-guard.yml、issue-submission-guard.yml 等治理流水线。
- **观察**：覆盖率 80% 四维硬门（`package.json` coverage 脚本）、CHANGELOG 极其细致（每条附 issue 号与致谢）、markdownlint 全仓执行——工程纪律显著高于同类项目；代价是 `npm test` 链条非常长。

## 8. 社区与维护状态

- **活跃度**：创建于 2026-01-18，9 个月冲到 28.7 万星（快照值）；最近 push 2026-10-26，持续高频（gh api）。
- **发布节奏**：v2.1.0（07-27）→ v2.2.0（08-28）→ v2.2.1（09-08）→ v2.2.2（09-30）→ v2.2.3（10-01），月度级甚至周级发布（gh api releases、CHANGELOG）。
- **贡献者结构**：gh api 贡献者分页约 457 人（含匿名），但 README 自述"单一维护者每周跨 7 个 harness 发布"（OSS 由赞助者与 Pro 订阅供养）；16 种语言翻译文档暗示大量社区翻译贡献。具体 commit 归属分布 tarball 无 git 历史，未确认。
- **治理痕迹**：`.github/ISSUE_MODERATION.md`、`pr-burst-guard.yml`、`issue-submission-policy.json` 的存在表明项目承受大量自动化/批量 PR 压力（间接佐证其热度）；260 个 open issue（gh api）。

## 9. 亮点与风险

**亮点**

1. 目前编码 agent 生态中规模最大、治理最完整的技能/代理内容库（311 技能/71 代理/97 命令），且配有 14 个校验器、catalog 一致性检查和画像系统，把 prompt 资产当软件工程做（`scripts/ci/`、`manifests/`）。
2. 真实运行时工程而非纯 prompt 集合：记忆库有路径/密钥/软链安全设计（`scripts/lib/memory-vault.js`），GateGuard 对 hook 性能做了懒加载与毫秒级预算（CHANGELOG），Rust 控制平面有清晰的跨进程一致性设计（`ecc2/README.md`）。
3. 多 harness 适配是系统性工作：hooks.json/codex-hooks.json/qoder-hooks.json/hermes/ 各自独立配置，适配层合规测试（`scripts/harness-adapter-compliance.js`）。
4. 文档纪律罕见：16 语言 README、CHANGELOG 逐条可溯、安全指南（`the-security-guide.md`）、故障排查指南（`TROUBLESHOOTING.md`）。
5. CI 覆盖安装生命周期（pack-installer、packed-install-lifecycle、pi-core 离线验证），发布链有 SLSA3 证明。

**风险/局限**

1. **单点维护 + 内容可信度**：核心由一人驱动；部分技能明显 niche 且疑似 LLM 批量生成（如 `skills/bengali-nlp/`、`skills/visa-doc-translate/`、`skills/esign-field-placement/`），311 个技能的实际质量参差，广度优先于深度。
2. **侵入性与可审计性**：hooks.json 内嵌压缩混淆的引导 one-liner，普通用户难以审计；GateGuard 会在首次编辑时拦截（CHANGELOG 自测约 +45% 轮次开销），对追求流畅的用户是显著摩擦。
3. **安装路径复杂**：README 用大篇幅警告"不要叠加安装方式"，97 个 legacy command shim、deprecated 技能与多套 harness 适配并存，认知负担重。
4. **商业化纠缠**：npm 包 welcome 文案、`scripts/ito.js` 赞助方 CLI 桥、Pro 升级入口散布在安装与运行路径中（`package.json` welcome 脚本、README 赞助区），对"中立工具"预期是减分项。

## 10. 总结评价

ECC 是 2026 年 agent harness 军备竞赛的标志性项目：它把"给 Claude Code 装一套工程流程"做成了一门有赞助商、有 Pro 订阅、有 16 国语言文档的生意，28.7 万星是这股浪潮的直接刻度。它并非单纯的 prompt 大礼包——记忆库、GateGuard、Rust 控制平面都是认真写的运行时代码，CI 与内容治理体系也确实超出绝大多数同类项目；但其星数更多反映的是赛道热度与病毒式传播（README 顶部即挂"GitHub Trending Repository of the Day"徽章），而非代码深度本身。适合想给 Claude Code/Codex 加装工程化流程的个人与团队直接使用，也适合研究"如何治理大规模 prompt 资产"的工程者读其 CI 校验器与 manifest 体系；想读核心算法的人则应直奔 `scripts/hooks/gateguard-fact-force.js`、`scripts/lib/memory-vault.js` 与 `ecc2/src/session/`。
