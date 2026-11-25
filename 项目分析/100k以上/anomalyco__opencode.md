# anomalyco/opencode 项目分析

> 星数 220484（快照 2026-10-24）｜主语言 TypeScript｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：开源 AI 编码 agent（"The open source AI coding agent"），以 CLI/TUI 为主入口，提供会话式智能体来读写代码、执行命令并完成开发任务。

| 项目 | 内容 |
| --- | --- |
| 仓库 | anomalyco/opencode（默认分支 `dev`；VS Code 插件清单仍指向 `sst/opencode`，证实项目由 SST 团队起家后迁移至 anomalyco 组织，见 `sdks/vscode/package.json`） |
| 星数 | 220484（快照）；查询时点 gh api 实测 222992，forks 约 29894 |
| 语言 | TypeScript（Bun 运行时），少量 Nix/Shell |
| 许可证 | MIT（`LICENSE`、`package.json`） |
| 版本 | v2.2.1（2026-10-29 发布，`CHANGELOG.md`；v1 旧线仍维护，最新 v1.19.5） |
| 最近活跃 | 2026-10-29 仍有 push（gh api `pushed_at`），发布节奏为数天一版 |

## 2. 项目是什么，解决什么问题

OpenCode 是一个终端优先的 AI 编码 agent：用户在终端（TUI）、桌面端（Electron）或 Web 界面里与 agent 对话，agent 通过一组内置工具（读写文件、执行 shell、检索、子 agent 等）自主完成多步编码任务。README.md 给出的两个内置 agent 体现了产品哲学：`build`（默认，全权限开发）与 `plan`（只读分析，改文件与执行命令都要先请求许可），另有 `general` 子 agent 处理复杂搜索与多步任务（`README.md` Agents 节）。

它解决的核心问题是"把 LLM 变成可安全操控本地代码库的工程助手"。围绕这一点，项目构建了完整的技术闭环：模型接入层抽象了几十家 provider（`packages/ai/src/providers/` 含 alibaba、amazon-bedrock、google-vertex 等）；工具系统带权限审批与插件扩展；会话层负责提示编排、上下文压缩、断点恢复与重试（`packages/core/src/session/`）；持久化用 SQLite（drizzle-orm）；MCP 客户端接入外部工具生态（`packages/core/src/mcp/`）。

架构上是典型的"本地服务端 + 多客户端"：核心能力集中在 `packages/core`，由 `packages/server` 暴露 HTTP API（接口定义在 `packages/protocol`），TUI、Desktop、Web App、VS Code 插件都是该服务的客户端。这使同一个 agent 内核可以同时服务终端用户和 GUI 用户，也是其 v2 重构后最显著的架构决策。`specs/v2/README.md` 用"权威归属表"明确了 HTTP/传输归 Protocol、公开领域模型归 Schema、运行时行为归 Core 的分层契约。

项目还带着明显的"公司化开源"特征：仓库内同时容纳了云端控制台（`packages/console`）、营销官网（`packages/web`）、统计服务（`packages/stats`）与 SST 基础设施配置（`sst.config.ts`、`services/`），商业化与开源内核同仓演进。

## 3. 技术栈与依赖

- 语言/运行时：TypeScript + Bun 1.4.4（`package.json` `packageManager`），无 Node 专属绑定处通过 import map 兼容 Node/workerd（`packages/core/package.json` 的 `imports` 里 `#sqlite`、`#pty` 分 bun/node/workerd 三实现）。
- 函数式内核：Effect 4.0.6 全仓铺开（catalog 依赖），服务用 `Context.Service` + `Layer` 组织，错误用 tagged enum，见 `packages/core/src/session/runner/index.ts`。
- 数据：drizzle-orm + SQLite（`packages/core/src/database/`）。
- 服务端：Hono 4.x + 自研协议层 `@opencode/protocol`（`packages/server/src/api.ts`）。
- 模型接入：自研 `@opencode/ai`（`packages/ai/`），底层复用/改造 Vercel AI SDK 系列包，仓库维护了 18 个 `patchedDependencies`（`package.json`）。
- 终端 UI：SolidJS 系 `@opentui/solid` 渲染 TUI（`packages/tui/src/index.tsx`）；桌面端 Electron 44（`packages/desktop/package.json`）；官网/文档 Astro + Starlight（`packages/web`）。
- 解析：web-tree-sitter + tree-sitter-bash/powershell 做 shell 命令解析（`packages/core/package.json`，服务于命令权限分析）。
- 工程：turbo 任务编排、oxlint + ast-grep 自定义 lint 规则、husky、Playwright e2e、fast-check 属性测试。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `packages/core` | agent 内核 `@opencode/core`：session、tool、agent、permission、mcp、plugin、database、provider、skills、worktree 等 |
| `packages/ai` | 模型接入层 `@opencode/ai`：LLM 客户端、provider 适配、事件流 |
| `packages/schema` | 公开领域模型（session/tool/agent 等 zod/effect schema） |
| `packages/protocol` | HTTP API 契约与事件协议 |
| `packages/server` | 本地 HTTP 服务（handlers、event-feed、middleware） |
| `packages/cli` / `packages/tui` | 命令行入口与终端 UI（含 ACP 客户端协议支持 `packages/cli/src/acp/`） |
| `packages/app` / `packages/desktop` / `packages/web` / `packages/console` | Web App、Electron 桌面端、官网文档、云控制台 |
| `packages/plugin` / `packages/gui-plugins` / `packages/plugins` | 插件运行时与内置插件 |
| `packages/codemode` + `core/src/codemode` | "CodeMode"：把工具暴露为代码 API 供模型调用 |
| `packages/client` / `packages/sdk` | 生成的类型化客户端与 SDK 组合层 |
| `sdks/vscode` | VS Code 扩展 |
| `specs/v2` | 架构规格说明（session、tools、事件流、provider 策略） |
| `services/` `infra/` `salesforce/` | 云端配套服务与基础设施 |

组织方式为 Bun workspaces + turbo 的超大 monorepo（`packages/` 下约 37 个包）。依赖方向有明文约束（根 `AGENTS.md`）：Schema ← Core/Protocol ← Server，Client 只可依赖 Schema/Protocol，`sdk` 负责组合，违反方向的代码会被评审拒绝。这是该仓库最值得学习的工程治理设计之一。

## 5. 核心架构与关键实现

### 5.1 会话执行循环（durable session runner）

核心在 `packages/core/src/session/runner/`。`SessionRunner` 的接口只暴露一个 `drain`：从已持久化的会话历史中"排空"可执行的工作，返回 `Complete / Moved / Reloaded` 三种续跑结果（`session/runner/index.ts`）。即 agent 循环不是内存中的 while 循环，而是围绕持久化消息流的可恢复状态机——进程崩溃后凭 `continuation` 续跑。

单步执行在 `session/runner/step.ts`：调 `LLMClient`（来自 `@opencode/ai`）发起一次 LLM 请求，处理工具调用，用 tagged enum `Outcome` 归纳六种走向——`Completed / Retry / Continue / RecoverFull / Compacted / OutputLimit`。其中 `RecoverFull` 与 `Compacted` 对应上下文溢出的自动恢复：运行器拥有压缩策略（`session/compaction.ts`），在持久化输出前触发上下文压缩后重试。用户消息通过 `SessionInbox` 排队，支持 `steer`（插入当前意图）与排队两种投递语义（`session/prompt.ts` 的 `delivery` 字段）。

系统提示词是静态模板 + 按工具集注入的引导段：`session/system-prompt.ts` 读取 `session/runner/prompt/system.txt`，把 `${OPENCODE_TOOL_GUIDANCE}` 替换为按已注册工具（shell/write/edit）生成的使用守则。

### 5.2 工具系统：可重放的注册表

工具注册表在 `packages/core/src/tool.ts`（366 行），19 个内置工具实现于 `core/src/tool/plugin/`（shell、edit、write、read、grep、glob、patch、todo、skill、subagent、webfetch、websearch、question、schedule、image-generate、mcp-resource 等，合计约 3700 行）。设计要点（依据 `core/src/tool/AGENTS.md`）：

- 注册走 `Tool.Service.transform`，注册表只保存"变换序列"，`reload()` 时按注册顺序对新 editor 重放，因此插件/MCP/内置工具的覆盖优先级天然确定且可重建；每个注册返回幂等 `dispose`。
- 工具可声明 namespace，模型侧名称展平为 `<namespace>_<tool>`；默认进入 CodeMode（工具以代码 API 形式暴露给模型），`codemode: false` 才走 provider 原生 tool 列表。
- 边界校验集中在 `tool/runtime.ts`：执行前解码输入、把插件抛出的任意错误归一为 `Tool.Error`，权限拒绝以 defect 形式穿透到 `SessionModelRequest.executeTool`，避免中间层误吞中断信号。
- 内置工具均接入 `Permission.Service`，权限来源固定为 `{type:"tool", messageID, id}`（`tool/plugin/shell.ts` 的执行路径）。

```ts
// packages/core/src/tool/runtime.ts（节选）
const result = yield* tool.execute(decoded, context).pipe(
  Effect.mapError((error: unknown) =>
    error instanceof Tool.Error ? error : new Tool.Error({...}),
  ),
)
```

### 5.3 模型接入与多客户端协议

`packages/ai`（`@opencode/ai`）统一封装 LLM 客户端、provider 配置、事件流与重试（`src/llm.ts`、`src/provider-error.ts`），provider 目录按厂商+协议细分（如 `providers/alibaba/chat`、`/responses`），并对接 models.dev 的模型元数据（`core/src/models-dev/`）。服务端把会话事件编码为单一事件流（决策记录 `specs/v2/event-stream-architecture.md`），客户端通过 `@opencode/protocol` 定义的 API + `packages/client` 生成代码消费，这解释了一个仓库能同时高质量支撑 TUI/Desktop/Web 三端的原因。

### 5.4 子 agent 与扩展

子 agent 通过 `tool/plugin/subagent.ts` 派生独立会话任务（`session/subagent-job.ts`、`subagent-completion.ts` 负责回执）；agent 本身是可配置实体（`core/src/agent.ts`，默认 `build`），权限、模式（subagent/hidden）都在 Agent.Info 上声明。插件系统（`packages/plugin`）支持 bun/node/包三种来源加载，hook 覆盖 session 提示、shell 创建、权限等环节（`plugin/src/effect/`），GUI 插件单独成包。

## 6. 构建与运行

```bash
# 安装与开发（根 CONTRIBUTING.md / package.json）
bun install
bun dev [directory]          # 本地跑 CLI
bun run dev:live             # TUI 连接已安装的后台服务联调（AGENTS.md）
bun run dev:desktop          # 桌面端（packages/desktop）
bun run check                # lint(oxlint + ast-grep) + typecheck(turbo)
bun turbo typecheck          # 类型检查（并发 3）

# 测试：根 package.json 明确禁止从根目录跑测试（"do not run tests from root"）
cd packages/core && bun run test      # 各包自带 test 脚本（bun test）

# 最终用户安装（README.md）
curl -fsSL https://opencode.ai/v2/install | bash
npm i -g @opencode/cli@latest
brew install anomalyco/tap/opencode-v2
```

## 7. 测试与工程质量

- 规模：全仓 1256 个 `*.test.ts`/`*.spec.ts`（find 统计，不含 node_modules）；仅 `packages/core/test/` 就有 183 个测试文件，覆盖 bus、database 迁移、文件访问、权限、codemode、instructions 等子域，另在 `packages/app/e2e/` 有 Playwright 回归与性能基准（如 session-timeline-benchmark）。
- 风格：bun test 为主，fast-check 做属性测试（catalog 依赖），AI 层有基于录制的 provider 回放测试（`packages/ai/test/provider/*.test.ts` 配合 `@opencode/http-recorder`）。
- CI：21 个 workflow（`.github/workflows/`），含 `check.yml`、`check-test.yml`、`release.yml`、`nightly-beta.yml`，以及成套的机器人治理（`triage-close-issues.yml`、`triage-pr-standards.yml`、`triage-contributor.yml`）——对 issue/PR 做自动分类与关闭。
- 质量观察：除常规 lint 外还维护 ast-grep 自定义规则集（`lint:effect-patterns`、`test:lint-rules`），把 Effect 使用规范固化为机器检查；`AGENTS.md` 用一整节规定风格（禁 `any`/star import、避免 try/catch 等），规格说明 `specs/v2/` 记录跨模块契约与决策记录，工程纪律在同类开源项目中属第一梯队。

## 8. 社区与维护状态

- 活跃度极高：默认分支 `dev`，查询时 `pushed_at` 为分析前一日（2026-10-29，gh api）。
- 发布节奏：v2.2.0（10-27）、v2.2.1（10-29）、v1.19.5（10-28）连续数日发版（gh api releases + `CHANGELOG.md`），v1/v2 双线维护。
- 贡献者：gh api contributors 翻页统计约 300+；开放 issue 约 3862、fork 约 29894（gh api），社区体量大但产品路线由核心团队强控——`CONTRIBUTING.md` 明言"UI 与核心功能需先过设计评审，不符合守则的 PR 会被直接关闭"，并引导贡献者只做 bug 修复、LSP/formatter 增补、性能与文档。
- 文档本地化：22 种语言的 README 翻译（根目录 `README.*.md`），有 bot workflow 维护（`.github/workflows/bot-docs-locale.yml`）。

## 9. 亮点与风险

**亮点**

1. 客户端-服务端分层的 agent 内核：一套 Core 服务 TUI/Desktop/Web/VS Code 四端，事件流与协议有决策记录背书（`specs/v2/event-stream-architecture.md`、`packages/protocol`）。
2. 可恢复的持久化执行循环：`drain`/`continuation`/`Compacted` 设计让长任务能跨进程崩溃续跑，工具输出与上下文压缩策略内置（`session/runner/`）。
3. 工具注册表设计成熟：变换重放、幂等 dispose、命名空间、CodeMode 双形态，MCP 刷新不破坏插件覆盖优先级（`tool.ts` + `tool/AGENTS.md`）。
4. 工程治理示范：依赖方向规则、ast-grep 自定义规则、规格/决策记录目录 `specs/v2/`、1256 个测试文件。
5. 生态位完整：MCP 客户端、插件 hook、skills、worktree、ACP 协议、22 语言文档，基本覆盖编码 agent 的全部扩展面。

**风险/局限**

1. Effect-TS + 自研多层抽象（ai/protocol/schema/core 四层）显著抬高贡献与二次开发门槛，读懂一个工具执行路径要跨 5+ 个包。
2. 迭代极快且公开 API 有破坏性变更：v2.2.1 的 changelog 即列出 `@opencode/ai` 多个 provider 移除与入口改名（`CHANGELOG.md`），依赖其 SDK 的下游需频繁适配。
3. 单仓混合开源内核与商业化组件（console/stats/services），部分能力（如 provider 策略的 Console 管控，`specs/v2/provider-policy.md`）与公司云服务耦合，纯社区部署无法获得完整体验。
4. 依赖维护负担重：686KB 的 `bun.lock`、18 个 patchedDependencies，供应链升级与补丁漂移是长期成本。

## 10. 总结评价

OpenCode 是当前开源编码 agent 中工程完成度最高的项目之一：v2 重构后以"持久化会话内核 + 多客户端协议"为核心，把 agent 循环、工具注册、权限审批、MCP、插件和模型接入做成了分层清晰的完整体系，并用规格文档、自定义 lint 和上千个测试文件维持着极快的发布节奏。它适合三类人：想要一个可日常使用的终端 AI 编码助手的开发者（直接安装即用）；想构建自己的 agent 产品、需要参考工具系统与会话恢复设计的工程师（精读 `packages/core/src/session/runner/` 与 `tool.ts`）；以及研究 Effect 大规模落地范式的 TypeScript 学习者。想 casual 地提 feature PR 的人则要先过设计评审这道门——这是一个由核心团队主导、社区辅助的强治理项目。
