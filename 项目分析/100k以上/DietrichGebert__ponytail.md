# DietrichGebert/ponytail 项目分析

> 星数 174500（快照 2026-10-24）｜主语言 JavaScript｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：把"房间里最懒的资深开发"装进 AI 编程 agent——一个强制 agent 写"最小完整改动"（YAGNI）的提示词/技能包，配 30+ 个 agent 的分发适配层。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | DietrichGebert/ponytail（npm: `@dietrichgebert/ponytail`） | `package.json` |
| 星数 / fork | 174.5k（快照）；gh api 实时 180,025 / 9,870 fork | gh api |
| 主语言 | JavaScript（另有 TS/TSX、Python，产品本体是 Markdown） | gh api |
| 许可证 | MIT | `LICENSE` |
| 创建 / 最近 push | 2026-06-12 / 2026-10-29 | gh api |
| 最新 release | v5.3.3（2026-10-29） | gh api releases/latest |
| 贡献者 | 约 132 人 | gh api contributors 分页 |
| 主页 | https://ponytail.dev | gh api homepage |

## 2. 项目是什么，解决什么问题

Ponytail 解决的是 AI 编程 agent 最常见的毛病：过度工程。你只要一个日期选择器，agent 会装一个第三方库或手搓 335 行日历（README "Before / after" 节给出的真实对照，10 行方案见 `examples/infinite-scroll.md` 等）。Ponytail 把这个行为纠正压缩成一段人格化提示词："你是懒惰的资深开发，最好的代码是你没写出来的代码"（`skills/ponytail/SKILL.md:15`）。

核心方法论是"梯子"（`skills/ponytail/SKILL.md` "The smallest complete change" 节）：理解任务后，按 6 级顺序取第一个成立的选项——①真的需要存在吗（YAGNI）②代码库里已有吗 ③标准库/平台原生 ④已安装的依赖 ⑤能一行读懂吗 ⑥都不行才写最小实现。同时明确"懒于方案、绝不懒于阅读"（先读改动触及的调用方、测试、配置），且**永不砍**信任边界校验、防数据丢失的错误处理、安全、无障碍（SKILL.md 第 44 行）。

仓库实际上分两层。**行为层**：一个 53 行的核心 prompt（`skills/ponytail/SKILL.md`）+ 精简版 `AGENTS.md`，外加 5 个扩展技能——`/ponytail-review`（按"正确、安全、承压、有测试、快、精简"顺序审查 diff，读上下文而非只看 diff）、`/ponytail-audit`（全仓审计并排序）、`/ponytail-debt`（收割 `shortcut:` 注释为债务台账）、`/ponytail-gain`（成效记分板）、`/ponytail-help`。**分发层**：把这些内容装进 30+ 个 agent——Claude Code/Codex/Copilot 走插件市场，Cursor 走 hooks，Windsurf/Cline/Kiro 等走规则文件（完整矩阵见 `docs/agent-portability.md`，逐行标注每个 host 的安装方式、能力层级和验证版本）。

v5 系列还加入了真实代码形态的功能"the band"：Claude Code 提示框上方的状态条（成本、token、agent 时间、复用率、测试状态），以及本地 board 页面（`hooks/band/board.ts`），数据默认只留在 `~/.config/ponytail/band/`，可选 webhook 推送（`.claude-plugin/plugin.json` userConfig）。

项目对效果可验证性异常执着：`benchmarks/` 内置 promptfoo 单发基准和 agentic 基准（无头 Claude Code 会话 + 种子仓库 + 对抗输入安全层），并在 `benchmarks/README.md` 公开承认早期单发数字夸大（issue #126），改用"同一 agent 带/不带技能"的诚实基线。

## 3. 技术栈与依赖

- **产品本体**：Markdown 提示词（`skills/*/SKILL.md`、`AGENTS.md`、`commands/*.toml`），零运行时依赖（`package.json` 无 dependencies）。
- **Node.js（CommonJS hooks + ESM 插件）**：`hooks/*.js` 生命周期钩子、`scripts/` 维护脚本、`.opencode/plugins/ponytail.mjs`（OpenCode 服务器插件，双 API：V2 `setup()` + V1 `server()`）。仅用 Node 内置模块（fs/path/os/crypto）。CJS→ESM 用 `createRequire` 桥接（`.opencode/plugins/ponytail.mjs:24`）。
- **TypeScript/TSX**：`hooks/band/`（register.tsx 495 行、board.ts 560 行、metrics.ts 311 行），依赖宿主注入的 `claude-code` 模块（`hooks/band/register.tsx:1`），测试用 `claude-code/testing`（`tests/band.test.tsx:1`）。
- **Python**：`__init__.py`（Hermes Agent 插件，254 行）+ `benchmarks/agentic/`（run.py/judge.py/compare.py 等，含 pandas 依赖）。
- **工具链**：`npm test` = Node 22 原生 `node --test`（`package.json:38`）；CI 用 GitHub Actions；npm 发布走 OIDC trusted publishing（`.github/workflows/publish.yml`，无需 token，自动附加 provenance）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `skills/` | 6 个技能的本体（ponytail + review/audit/debt/gain/help），唯一事实源 |
| `AGENTS.md` | 精简规则集，instruction-tier host 的通用入口 |
| `commands/*.toml` | 6 个 slash 命令定义（Gemini/Qwen 等自动发现） |
| `hooks/` | Claude/Codex/Copilot/Qoder/Cursor 钩子（模式激活、追踪、注入、band） |
| `hooks/band/` | v5.3 band UI：TSX 渲染 + 指标计算 + 本地 board 页 |
| `.opencode/` | OpenCode/Kilo 服务器插件 `.mjs` |
| `pi-extension/` | pi agent 的包扩展（`package.json` 的 `pi` 字段接线） |
| `__init__.py`、`plugin.yaml` | Hermes Agent 原生插件（Python） |
| `.claude-plugin/`、`.codex-plugin/`、`.github/plugin/` 等 | 各插件市场的清单（marketplace/plugin.json） |
| `.cursor/`、`.windsurf/`、`.clinerules/`、`.kiro/` 等 | instruction-tier host 的规则文件副本 |
| `tests/` | 22 个测试文件（Node test runner + claude-code/testing） |
| `benchmarks/` | promptfoo 单发基准 + agentic 基准 + 10 份结果报告 |
| `examples/` | 12 个"无技能 vs 有技能"真实模型输出对照 |
| `scripts/` | 版本/规则一致性检查、Cursor hooks 安装器、OpenClaw 技能构建 |
| `i18n/` | README 四语翻译（西/韩/中/日） |
| `docs/agent-portability.md` | 30+ host 的适配矩阵（约 35 行表格，每行注明验证版本） |

组织方式评述：典型的"单一事实源 + 生成/校验式多目标分发"。所有规则文件副本由 CI 强制与 `AGENTS.md` 逐字一致（`scripts/check-rule-copies.js`），所有清单版本由 `scripts/check-versions.js` 校验为 5.3.3——这解决了"适配 30 个 agent 格式"必然带来的漂移问题，是这个仓库区别于普通 prompt 集合的关键。

## 5. 核心架构与关键实现

### 5.1 共享指令构建器：一份 SKILL.md 按级别过滤

所有 host 共用 `hooks/ponytail-instructions.js`：剥掉 frontmatter 后，按当前级别（lite/full/ultra）过滤掉表格里其他级别的行和 `- lite: "..."` 形式的示例。过滤规则刻意保守——只匹配 mode 名做标签的行，避免误删以 "Full: " 开头的普通规则（代码注释明确引用了修复 issue #571）：

```js
// hooks/ponytail-instructions.js:28-36
// Require a quoted value: every worked example is `- lite: "..."`. Without
// this, an ordinary rule bullet that happens to start with a mode word
// (e.g. "- Full: ...") is silently dropped in every other mode
const exampleLabel = line.match(/^-\s*([^:]+):\s*"/);
if (exampleLabel) {
  const labelMode = normalizeMode(exampleLabel[1].trim());
  if (labelMode) return labelMode === effectiveMode;
}
```

Python 侧 `__init__.py:61-86` 用 `re.split(r"\r?\n", ...)` 逐行镜像了同一逻辑，注释甚至解释了为何不用 `str.splitlines()`（会丢结尾换行、与 JS 行为不一致）——跨语言行为一致性被当成了显式工程目标。

### 5.2 运行时状态机：环境变量指纹识别宿主

`hooks/ponytail-runtime.js` 开头是一段宿主识别链：按 `COPILOT_PLUGIN_DATA`、`PLUGIN_DATA`（Codex）、`QODER_SESSION_ID`、`CODEBUDDY_PLUGIN_ROOT`、`CURSOR_VERSION` 依次判别 Copilot/Codex/Qoder/CodeBuddy/Cursor，各自落到不同的状态目录。模式标志写入 `.ponytail-active`，且对 Claude Code 按 `sha256(projectDir)` 分项目存（`ponytail-runtime.js:47-50`），避免不同仓库的并发会话互相覆盖（#662、#809）。注释里连 VS Code Copilot 不设 `COPILOT_PLUGIN_DATA` 只注入 `CLAUDE_PLUGIN_ROOT` 的怪癖（#528）和 Cursor 只在 hook 进程环境设置 `CURSOR_VERSION` 的细节都记录在案。

### 5.3 注入路径：三种 host 三条管线

- **Claude Code**：`hooks/claude-codex-hooks.json` 把 `node -e "require(...)"`（路径反斜杠内联替换，规避 Windows 引号问题）挂到 SessionStart/SubagentStart/UserPromptSubmit 三个事件。
- **OpenCode**：`.opencode/plugins/ponytail.mjs` 每回合通过 `context` 与 `compaction` 两个 hook 把规则集 push 进 system prompt（`ponytail.mjs:157-163`，注释说明 compaction 也注入是为了保住 provider prompt cache 命中）。
- **Hermes**：`__init__.py` 的 `pre_llm_call` 注入前，先倒序扫描历史 `api_content` 里已持久化的模式标记，模式没变就不重注入（`__init__.py:114-135`），节省上下文。

### 5.4 band：纯函数拆分保证可测

`hooks/band/metrics.ts` 头部注释写明"Pure helpers for the band: no `$`, so the tests call them directly"。复用率检测用每语言正则提取顶层定义（py/go/rs/rb/ts/js），配显式的 `shortcut:` 注释自我标记精度上限："regex, not a parser; nested and dynamic definitions are missed. Use tree-sitter if it matters."（`metrics.ts:14`）——项目自己的债务约定被自己的代码遵守。

## 6. 构建与运行

安装（来自 README/INSTALL.md）：

```bash
# Claude Code（两条分开的提示词）
/plugin marketplace add DietrichGebert/ponytail
/plugin install ponytail@ponytail
# Codex
codex plugin marketplace add DietrichGebert/ponytail
codex plugin add ponytail@ponytail
# 任意其他 agent：把 AGENTS.md 拷进项目，或安装 skills/ponytail/SKILL.md
```

测试与校验（`package.json`、`.github/workflows/test.yml`）：

```bash
npm test              # node --test tests/*.test.js && npm test --prefix pi-extension
node scripts/check-rule-copies.js   # 规则副本与 AGENTS.md 一致性
node scripts/check-versions.js      # 各清单版本一致性
python3 benchmarks/agentic/run.py --selftest
```

复现基准（`benchmarks/README.md`）：

```bash
npx promptfoo@latest eval -c benchmarks/promptfooconfig.yaml --repeat 10
```

## 7. 测试与工程质量

- **规模**：22 个测试文件、约 5,000 行测试代码；最大的是 `tests/hooks.test.js`（1005 行）与 `tests/band.test.tsx`（520 行）。覆盖钩子行为、各插件清单、模式过滤、命令解析、复用统计，甚至专门的 `hooks-windows.test.js`（Windows 路径/引号）。
- **框架**：Node 原生 `node --test`（零测试依赖）+ 宿主提供的 `claude-code/testing`。
- **CI**（`.github/workflows/test.yml`）：规则副本一致性 → 版本一致性 → npm test → 基准 selftest，四步全绿才算过；action 按 commit SHA 固定。
- **工程质量观察**：代码注释密度高且全部指向真实 issue 编号（#126、#528、#571、#662、#736、#809、#854），几乎是"注释即变更日志"；`shortcut:` 债务注释约定在自家代码里践行；发布流水线用 OIDC 免 token。缺点是大量依赖宿主未文档化的行为（环境变量、hook 输出格式），兼容矩阵靠人工逐版本验证（`docs/agent-portability.md` 每行标注"Verified with X 1.x.y"）。

## 8. 社区与维护状态

- **活跃度**：创建于 2026-06-12，4 个半月冲到 17 万+ 星（README 展示 Trendshift 日/周/月三枚徽章）；最近 push 2026-10-29（当天两次提交，最新一条是 #1313 文档更新）；最新 release v5.3.3 发布于 2026-10-29——按版本号 5.3.3 和 `benchmarks/results/` 下 2026-06 至 2026-10 的 10 份报告推断，迭代节奏约为每周小版本。
- **贡献者**：约 132 人（gh api contributors 分页推算）。从代码注释风格看，核心逻辑（hooks、指令构建器、基准）高度集中于作者本人，社区贡献多为适配与文档。
- **生态**：npm 包 `@dietrichgebert/ponytail`；README 有赞助商（GreenPT）与"用 Ponytail 做出的产品"展示位；issue 数 135 open，基准方法论直接回应社区批评（#126）说明 issue 驱动开发明显。非官方 fork 等更深社区结构：未确认。

## 9. 亮点与风险

**亮点**

1. 单一事实源 + CI 强制一致的适配层：`scripts/check-rule-copies.js` 让 30+ 个格式各异的 agent 规则文件永不漂移，是"prompt 即代码"工程化的范本（`docs/agent-portability.md`）。
2. 证据文化：`benchmarks/` 不只报好数字，还保留被打脸的旧方法并公开修正（`benchmarks/README.md` "#126" 段），agentic 基准用对抗输入安全层验证"代码少但没更不安全"（30/30 通过）。
3. 跨语言行为一致性：Python 与 JS 两份模式过滤逻辑逐行对齐，连 `splitlines` 的换行差异都有注释（`__init__.py:66-68`）。
4. 自我豁免式幽默与纪律并存：`AGENTS.md:32` "(Yes, this file also applies to agents working on the ponytail repo itself. Especially to them.)"，且代码里确实遵守 `shortcut:` 约定。
5. v5.3 band 是真功能而非噱头：本地优先（数据不出机器）、webhook 可选、token 进系统钥匙串（`.claude-plugin/plugin.json`）。

**风险/局限**

1. 本质是行为调优提示词：instruction-tier host（Cursor 规则、Windsurf、Cline 等）无生命周期钩子，级别不持久、子 agent 不注入，体验在 host 之间差异显著（`docs/agent-portability.md` 能力分层）。
2. 效果数字全部来自自建基准（方法透明但自己执棒），第三方基准仅"链接供参考"（`benchmarks/README.md` "Independent benchmarks" 节）。
3. 适配层强依赖宿主未文档化的行为（`CURSOR_VERSION` 只在 hook 进程设置、CodeBuddy 丢弃 SubagentStart 输出等），宿主一升级就可能碎，长期维护成本高。
4. band 仅支持 Claude Code ≥ 2.1.290，且 VS Code 聊天面板、云端会话、`claude -p` 不显示（README "New in 5.3" 节）。

## 10. 总结评价

Ponytail 是 2026 年"agent 行为工程"赛道最成功的单品：把一个 53 行的反过度工程提示词，用插件市场适配、CI 一致性校验、跨宿主状态机、可复现基准和本地仪表盘包装成了基础设施级的分发工程。星数里无疑有梗文化的病毒成分（"马尾辫 lazy senior dev"人设），但代码质量和自我批判的诚实度撑得起热度。适合三类人使用/阅读：被 AI 生成代码膨胀困扰的开发者（直接装）；想做自己的 agent 技能包/插件市场的作者（读 `docs/agent-portability.md` + `scripts/check-rule-copies.js` 的分发设计）；研究 LLM 行为可控性的人（读 `skills/ponytail/SKILL.md` 的规则措辞与 `benchmarks/agentic/` 的评测方法）。
