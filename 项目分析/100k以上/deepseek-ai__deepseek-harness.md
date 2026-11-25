# deepseek-ai/deepseek-harness 项目分析

> 星数 259199（快照 2026-10-24）｜主语言 TypeScript｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：DeepSeek 官方开源的 agent harness（智能体运行框架）`dsh`，以"一切皆插件"（Everything is a Plugin）为核心理念，全部能力——模型适配、工具注册、会话日志乃至 agent 主循环本身——都是可替换的 Cordis 插件。

| 项目 | 事实 |
|---|---|
| 名称 | deepseek-ai/deepseek-harness（CLI 名 `dsh`，npm 包 `@deepseek-ai/dsh`） |
| 星数 | 259199（快照 2026-10-24）；`gh api` 实测 265676（2026-10-30） |
| 语言 | TypeScript（Node >= 24.9）+ Python SDK + 少量 native addon |
| 许可证 | MIT（`LICENSE`，Copyright (c) 2026 DeepSeek） |
| 版本 | 0.2.2-alpha.2（`package.json` / 最新 release `dsh-v0.2.2-alpha.2`），官方声明为 developer preview |
| 最近活跃 | 仓库创建于 2026-08-13，最后 push 2026-10-28（`gh api repos/deepseek-ai/deepseek-harness`），极为活跃 |
| 代码规模 | packages/apps/native 下约 115 万行 TypeScript、5444 个 .ts 文件；packages/ 下 378 个 package.json |

## 2. 项目是什么，解决什么问题

DeepSeek Harness 是 DeepSeek AI 开发的开源 agent harness（`README.md`），定位类似 Claude Code / OpenCode 这类"驱动 LLM agent 干活"的运行时，但架构取向非常鲜明：**没有特权内核，一切皆插件**。它构建在 Cordis 插件框架之上（`docs/architecture.md`、`docs/cordis-primer.md`），设计思想有对应论文《A Programming Paradigm for Spatiotemporal Composability》（README 引用 arXiv 2608.25512）。

它解决的核心问题是"agent 框架的可替换性"：在常见 harness 中，模型接入、工具执行、上下文管理等往往是硬编码内核，扩展只能靠有限的 hook；而在 dsh 中，模型适配器、工具注册表、会话日志、agent 循环都是挂载在同一上下文里的插件，"扩展 dsh 的方式就是在其他插件旁边再挂一个插件"（`docs/architecture.md` §Cordis），且注册都是可逆 effect，插件卸载时自动回退。

产品形态覆盖四种运行档（profile）：`web`（本地 Web UI，`npx @deepseek-ai/dsh web`，默认 127.0.0.1:3080，`README.md`）、`headless`（一次性跑任务）、`sdk` / `sdk-minimal`（JSON-RPC SDK，供 Python/TS SDK 驱动）、`acp`（自动化协议），另有 Electron 桌面应用（`apps/desktop`、`docs/architecture.md` §Desktop application）。

一个显著的工程特征是"事件溯源"：模型看到的上下文必须可以从会话日志重建——"Model-visible means logged"（`docs/architecture.md` §Session log）。每条 `assistant/message` 内嵌产生它的完整压缩流，fork、resume、转写、遥测、持久化都从持久事件派生。这使会话格式有 v0→v4 的逐版本迁移链（`packages/session/session-format-v0-to-v1/` 至 `v3-to-v4/`）。

该项目也是一个高度"agent 原生"的仓库：根目录 `AGENTS.md`（19KB 开发规范）、`.agents/notes/`（带日期的架构决策笔记）、`.agents/skills/`（预推送检查等 agent 技能），顶层 `CLAUDE.md` 等是指向 `AGENTS.md` 的符号链接。

## 3. 技术栈与依赖

- **语言/运行时**：TypeScript 6.0（`tsconfig.base.json` 55KB 严格配置，project references 分 host/client 两面），要求 Node >= 24.9（`package.json` engines）；Python SDK（`python/sdk`，经 stdio JSON-RPC 驱动打包的 dsh 运行时）；native addon `@deepseek-ai/node-addon-system`（`native/`）。
- **插件框架**：Cordis 4.0.0-rc.7 **源码 vendor 进仓库**（`vendor/cordis/`、`vendor/README.md` 清单），连同 cosmokit、schemastery（配置 schema）、loader、hmr、timer、group、include、logger-console，全部改用 `@deepseek-ai/*` scope 并记录上游 commit 与本地改动日志。
- **构建链**：pnpm 11 workspaces（`pnpm-workspace.yaml`）、tsc 项目引用 + tsdown 打包（`tsdown.config.ts`）、Vite 8 + React 18 做 Web 前端（`apps/web/package.json`）、Electron 桌面（`apps/desktop`）。
- **质量工具**：vitest 4 测试 + v8 覆盖率、oxlint（`​.oxlintrc.json` 12KB，非 ESLint）、jscpd 克隆检测、lefthook git hooks（`lefthook.yml`）、约 80 个 `verify-*`/`gen-*` 门禁与目录生成脚本（`package.json` scripts）。
- **模型接入**：`packages/llm/` 下有 llm 核心词汇、llm-deepseek（含 account/api-key 两种鉴权）、llm-pi-ai（多 provider 数据）、llm-retry、token-meter 等。

## 4. 目录结构与代码组织

| 目录 | 职责（依据 `AGENTS.md` §Repository layout 与实际清单） |
|---|---|
| `vendor/` | 源码 vendor 的 Cordis 框架及基础库（含升级/改动日志） |
| `packages/core/` | agent/session API：agent、agent-loop、agent-default-model、scope、session、system-prompt、tools、tool-executor-native/ptc |
| `packages/llm/` | 模型消息/流词汇与 provider 适配（deepseek、pi-ai、retry、token-meter） |
| `packages/session/` | 持久会话：format v0-v4 与逐版迁移、JSONL 持久化、projection、telemetry(otel)、title 等 |
| `packages/shell|subprocess|ssh|terminal|sandbox|fs|lsp/` | 执行面 capability（命令执行、进程、SSH、持久终端、沙箱、文件系统、语言服务器） |
| `packages/web|computer-use|browser-use/` | 搜索抓取、计算机/浏览器操作等模型能力 |
| `packages/compaction|context|spill|attachment/` | 上下文压缩、请求上下文、输出溢出、二进制附件 |
| `packages/subagent|workflow|jobs|schedule|goal|plan|todo/` | 委派子代理、工作流、后台任务、定时、目标、计划、todo 工具 |
| `packages/bundle/` | profile 打包层：base、web-app、headless、sdk-app、sdk-minimal、acp-app |
| `packages/boot|preset|guard|extensions|mcp|acp/` | 启动组装、agent preset 组合、守卫、运行时自扩展、MCP 外部工具、ACP |
| `packages/client|host|sdk|api/` | GUI 客户端/宿主、JSON-RPC SDK、远程 BFF |
| `packages/experimental/` | 无产品支持承诺的试验能力（agent-team、forge、worktree 等） |
| `apps/` | cli（`dsh` 入口）、web（React 前端）、desktop + desktop-host（Electron）、plugin-market-generator |
| `python/` | Python SDK（sdk、sdk-runtime 捆绑运行时 wheel） |
| `docs/` | 极完备文档：architecture、cordis-primer/tutorial、subsystems/、cookbook/、upgrade-guide/、postmortem/、中英双语 |
| `scripts/` | 约 80 个 gen-*/verify-* 门禁与发布脚本 |
| `snapshots/`、`benchmarks/`、`native/`、`website/`、`.agents/` | 快照测试夹具、性能门禁、原生 addon、文档站、agent 工作流 |

组织方式评述：超大规模 **pnpm monorepo**，`packages/<域>/<包>/` 两级分组（378 个包），每个包自带 README（中英）、tests、`cordis.patch.yml` 或 `dsh.bundle` 声明。核心与能力层、应用层、打包层界限清晰；依赖全部 workspace 内解析，第三方仅经 vendor 或少量运行时依赖。这是一份"官方全托管"式代码组织，罕见地连框架层也源码内联（`vendor/README.md` 明言为了可审计、可补丁、可钉死）。

## 5. 核心架构与关键实现

### 5.1 Cordis 插件模型与 profile/bundle 组装

Cordis 的五个核心概念（`docs/cordis-primer.md`）：插件是实现 Service 的对象；context 是服务仓库（服务认领 `ctx.<key>` 如 `ctx.tools`/`ctx.llm`）；用 `inject` 声明依赖实现按需激活而非手工排序；事件通信分 emit/waterfall/parallel/serial/bail 五种派发模式；所有注册都是可逆 effect。

产品组装用"profile = 有序 bundle 层叠 + 逐行 patch"表达：bundle 在自己 `package.json` 的 `dsh.bundle` 字段声明 patch 文件（`packages/bundle/base/package.json`），`dsh-base` 作为 web/headless/sdk/acp 四档共享第一层，其 `cordis.patch.yml` 以 97 个 id 行插入核心服务（llm、session、tools、sandbox、credentials 等），上层 bundle 与用户的 `cordis.patch.yml` 按 id 整行替换、"last write winning"（`packages/bundle/base/cordis.patch.yml` 头注释）。例如：

```yaml
- id: plugin-manager
  name: '@deepseek-ai/dsh-plugin-manager'
  disabled: !!js "!ctx.get('profileContext')"
- id: hmr
  name: '@deepseek-ai/dsh-hmr'
  disabled: !!js "!ctx.get('profileContext')"
  config:
    root: []
```

`dsh --profile web --dump-config` 可打印本机实际启动树，任何一行都能被用户 patch 替换（`docs/architecture.md` §Profiles and bundles）。这就是"Everything is a Plugin"的落地方式。

### 5.2 agent-loop：事件驱动的 turn/step 主循环

`packages/core/agent-loop/src/index.ts`（838 行）定义 `export class AgentLoop extends Service implements AgentFactory`（index.ts:291），`agent.ts`（745 行）是默认驱动。turn/step 模型在 `docs/architecture.md` §Turn flow 有完整时序：一次 turn 由零或多个 step 组成，每 step = 一次模型请求 + 其工具调用。扩展点分三类事件域：持久 session 事件（`turn/*`、`tool/*` 等，入日志）、活跃 `agent/*` 事件（观察/拦截在途工作）、能力事件（`fs/*`、`tools/*`）。其中 `agent/pre-step`、`agent/request`、`llm/stream`、`tools/*` 是 waterfall（around 中间件，监听者必须 `next()` 委托），可用于在请求前改写或拒绝输入。

### 5.3 tools：作用域注册表 + 带守卫的执行管线

`packages/core/tools/src/`（约 3900 行）：`contracts.ts` 定义 `ToolDefinition`——schema 之外强制 `output: ToolOutputDefinition`（工具必须声明输出的 JSON Schema 与纯投影 render），`execute(args, exec)` 收到冻结快照参数与取消信号；`pipeline.ts`（771 行）实现 pre-execute 审批策略、waterfall 拦截、取消传播与结果结算；`json-schema.ts`/`schema.ts` 负责校验。工具经 `defineTool` 声明，如 `packages/todo/tool-todo/src/index.ts` 所示：

```ts
export const name = 'tool-todo'
export const inject = ['tools', 'sessionProjections']
export const Config: z<Config> = z.object({
  allowParallelInProgress: z.boolean().required(),
})
```

命名导出 + `inject` 数组就是 loader 的注入元数据；内置模型工具约 30 个（bash、pwsh、fs、str-replace-editor、web、todo、subagent、agent-team、lsp、skill、jobs、workflow、worktree 等，`docs/tool-catalog.md`）。

### 5.4 session：事件溯源日志与版本化持久化

`packages/core/session/src/`（index.ts、types.ts、fork.ts、repair.ts 等）维护只追加的 `SessionEvent` 日志，`deriveMessages()` 从日志投影模型历史（`docs/architecture.md` §Session log）。物理层由 `packages/session/session-persistence-jsonl/` 负责 JSONL+zstd 帧与代际文件（`session.vN.jsonl`，"committed generation paths are never renamed, replaced, or deleted"），`packages/session/session-format-v0-to-v1/`…`v3-to-v4/` 每包恰好承担一版迁移；读取方经 `ctx.sessionProjections` 投影缝（`packages/session/session-projection/`）消费。会话读模型、UI 增量（`agent/assistant-stream`）与持久结算被严格分离。

### 5.5 capability seams：一处换供应商，全局生效

"seam"= 服务定义 + 服务提供者 + 消费者三角色（`docs/architecture.md` §Capability seams）。文件系统与子进程提供者共享同一执行世界，把它们指向远程沙箱即整体迁移 Bash、PTY、LSP，无需分叉 provider；子代理提供者从"全新子 agent"到"委派给另一产品的一个 turn"都在同一接口后变化。`docs/architecture.md` 末尾的 18 行"新行为去哪"表（加 provider→注册 `ctx.llm`；加工具→`ctx.tools`；拦截 turn→`agent/turn-stopping` 等）是整个扩展面的索引。

## 6. 构建与运行

来自根 `package.json` scripts 与 `AGENTS.md` §Commands：

```sh
npx @deepseek-ai/dsh web            # npm 直接运行 Web UI（README.md）
git clone … && pnpm install         # Node >= 24.9, pnpm 11
pnpm run build                      # build:artifacts + build:check（tsc + tsdown + Web）
pnpm dsh web                        # 用构建产物启动（等价 make web）
pnpm dsh --profile headless "task"  # 无头跑一个任务（需 DEEPSEEK_API_KEY）
pnpm run dev:web                    # 构建+启动+前端热重建（scripts/dev-web.ts）
pnpm run test                       # vitest 单测（先 build:native-system）
pnpm run test:coverage              # CI 覆盖率门禁：packages/*/*/src 逐文件 100%
pnpm run test:e2e                   # 真实 API 测试（无 key 自动跳过）
pnpm run test:snapshot              # 录制会话回放的快照测试（无 key）
pnpm run typecheck / lint / duplication / hygiene / doc-sync
pnpm run package:desktop:mac:arm64  # Electron 桌面打包（win/x64、unsigned 等变体齐全）
```

Python 侧：`python/sdk`（`deepseek-harness-sdk`，高层 turns API + JSON-RPC 客户端）与 `python/sdk-runtime`（捆绑 dsh 可执行与原生 sidecar 的 wheel，`python/README.md`）；发布走 GitLab CI（`.gitlab-ci.yml`，按 `python-v*` tag 构建 wheel）。

## 7. 测试与工程质量

- **规模**：2137 个 `*.spec.ts`、360 个包级 `tests/` 目录（仓库实测统计）；vitest 4 分 8 个专用配置（主/e2e/expected/snapshot/web/web-stress/web-perf/bench，见根目录 `vitest.*.config.ts`）。
- **分层策略**（`docs/testing.md`，写得非常认真）：单测 → 覆盖率门禁（**逐文件 100%**，"未覆盖行常是被门禁揪出的死代码"）→ 真实 API e2e（"We are DeepSeek — do not ration real-API tests"，冒烟测试启动真实 profile 发一条 prompt 验证世界状态）→ owner 本地期望输出 → 性能基准门禁（`benchmarks/`，限时间/堆/扩展性预算）→ 快照回放（无 key 录制回放，进程/Web 双通道）→ 浏览器快照（Chromium/WebKit）。
- **质量哲学**：优先真实实现而非 mock（只 mock LLM/网络/时钟边界）；"验证世界而非自我汇报"（e2e 断言重跑命令或外部重读文件）；"测试真实入口路径"（对构建后的 `lib/bin.js` 跑裸 node 冒烟）；每个注册表都有 HMR 安全测试（dispose fiber 断言清理）。还有 `postmortem/` 事后分析与 `snapshot` 契约回归测试。
- **静态门禁**：约 80 个 `verify-*` 脚本检查生成目录、文档链接/术语/翻译配对、依赖闭包、包元数据等（`package.json`），配 `gen-*` 生成器与 `--check` 模式；jscpd 查重复代码。
- **CI**：GitHub Actions 30+ 工作流（`.github/workflows/`：ci、ci-windows、e2e、desktop/macOS/windows 打包、release、docs-pages、weighted-approval 等）+ GitLab CI（Python wheel）。有多平台矩阵与自托管 runner 说明（`docs/testing.md`）。

## 8. 社区与维护状态

- **活跃度**：创建于 2026-08-13（`gh api`），不到 3 个月 25.9 万星、32205 fork，属于爆发式增长；最后 push 2026-10-28，与 tarball 文件 mtime 一致。
- **release 节奏**：dsh-v0.2.1-rc.1（10-17）→ rc.2（10-17）→ rc.3（10-23）→ rc.4（10-28）→ v0.2.2-alpha.2（10-28），数天一版（`gh api repos/.../releases`）；版本号仍处 0.2.x alpha/rc，README 明示"将有破坏性变更"。
- **贡献者结构**：`CONTRIBUTING.md` 明确"目前不接受外部 PR"，社区参与方式为 Discussions 反馈、编写 `dsh-plugin` 话题的生态插件、写教程；作者自述"very small team"。外部贡献者数量：未确认（tarball 无 git 历史，未做 contributors 统计）。
- **生态意图**：官方仓库自称"一个想法、一个官方样板与灵感来源，而非强制"（CONTRIBUTING.md），插件发现靠 GitHub topic `dsh-plugin`，另有 Discord 与微信群/公众号（`README.zh.md`）。
- **文档与安全**：`SAFETY.md` 安全须知、`docs/upgrade-guide/`、`docs/postmortem/`、`THIRD_PARTY_NOTICES.md`（26KB 第三方许可披露）齐备；中英双语全量维护（大量 `*.i18n.yaml` 配对文件与 `verify-translation-pairing` 门禁）。

## 9. 亮点与风险

**亮点**

1. 架构一致性罕见地彻底：从模型适配到 agent 主循环全是可替换插件，profile/bundle/patch 三层组装 + `--dump-config` 可视化，把"可定制"做成了第一性原理（`docs/architecture.md`、`packages/bundle/base/cordis.patch.yml`）。
2. 事件溯源会话设计："Model-visible means logged"，持久事实与活跃事件严格分离，session 格式带逐版迁移链，fork/resume/审计天然可得（`docs/architecture.md` §Session log、`packages/session/`）。
3. 工程质量天花板级：逐文件 100% 覆盖率门禁、2137 个测试文件、约 80 个 verify 门禁、性能基准纳入 PR 门禁、构建产物冒烟，且 `docs/testing.md` 把方法论讲成了一篇可移植的教材。
4. 框架层完全 vendor（Cordis + 8 个基础库）并记录上游 commit 与逐条本地改动，供应链可审计可控（`vendor/README.md`）。
5. agent 原生仓库范式：AGENTS.md 体系、`.agents/notes` 架构决策、`.agents/skills` 工作流、生成目录 + verify 配对，本身就是"AI 参与大型工程"的示范样本。

**风险/局限**

1. 预览期不稳定：0.2.x alpha，官方明示破坏性变更将持续（README "Developer preview"、`AGENTS.md` "Public APIs are pre-stable"）。
2. 不接受外部 PR，治理高度封闭；星数与 fork 数巨大但贡献面集中，社区只能做生态插件与反馈（`CONTRIBUTING.md`）。
3. 体量与复杂度门槛高：378 个包、115 万行 TS、project references 双面构建、Node 24.9+ 硬要求，fork 或深度二开成本大；仅 pnpm install/build 的磁盘与时间开销即不小（未运行验证，依据工程文件推断）。
4. 生态尚新：`dsh-plugin` 话题与插件市场生成器（`apps/plugin-market-generator`）刚起步，第三方插件丰富度未确认。

## 10. 总结评价

DeepSeek Harness 是 2026 年 agent harness 赛道中架构野心最大的开源实现：它把 Koishi/Cordis 社区验证过的"服务容器 + 可逆 effect + 五类事件派发"插件范式整体搬进 LLM agent 领域，配合事件溯源会话、逐文件 100% 覆盖率门禁和框架层 vendor 化，交付了一份设计自洽、文档完备、工程质量顶格的"官方样板间"。它当前更像一份宣言与参考实现——pre-stable API、封闭治理、两周数版地快速迭代——而不是追求即插即稳的生产工具。适合三类人：想研究"如何组织一个全插件化 agent 系统"的架构师（直接读 `docs/architecture.md` + `docs/cordis-primer.md` + `packages/core/agent-loop`）；想给 DeepSeek 生态写插件/SDK 的开发者；以及想借鉴其测试策略与仓库工程实践（verify 门禁、快照回放、AGENTS.md 体系）的工程团队。想直接当日常编码助手用的用户，建议等 API 稳定后再上车。
