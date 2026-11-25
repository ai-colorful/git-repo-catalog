# chatboxai/chatbox 项目分析

> 星数 39000（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：跨平台 AI LLM 桌面客户端（Windows/macOS/Linux/Web/iOS/Android），社区版以 GPLv3 开源，官方付费 Chatbox AI 服务与之共生。

| 项目 | 值 |
|------|----|
| 名称 | chatboxai/chatbox（原 Bin-Huang/chatbox） |
| 星数 | 39,000（快照）；gh api 实测 42,323 |
| 语言 | TypeScript（全栈 TS，React 18 + Electron 35） |
| 许可证 | GPL-3.0（LICENSE） |
| 创建时间 | 2023-03-06（gh api） |
| 最近推送 | 2026-11-12（gh api pushed_at） |
| 最新 Release | v1.24.4，2026-11-12（gh api releases） |
| 贡献者 | 75 人（gh api contributors 分页计数） |
| Forks / Open Issues | 4,332 / 1,369（gh api） |

## 2. 项目是什么，解决什么问题

Chatbox 是一个"开箱即用"的 LLM 桌面客户端：用户不写代码、不搭终端，即可接入 OpenAI、Claude、Gemini、Ollama 本地模型等各家 LLM 进行对话（README.md「Your Ultimate AI Copilot on the Desktop」）。它把 API Key 管理、多模型切换、Markdown/LaTeX/代码高亮渲染、会话历史、提示词库、图片生成等散落的能力收进一个统一 GUI，覆盖桌面三平台 + Web + 移动端（README「Platform Availability」章节，iOS/Android 商店链接可见）。

仓库现为 **Community Edition**：README 明确"This is the repository for the Chatbox Community Edition, open-sourced under the GPLv3 license"，并提到"We regularly sync code from the pro repo to this repo, and vice versa"（README.md 开头，链接 issue #2266「Chatbox is going open-source Again!」）。即商业公司 chatboxai 的 Pro 闭源版与社区版双向同步，社区版承担开源门面与社区协作。

数据隐私是其核心卖点：README 徽章「Local First」+「Local Data Storage: Your data remains on your device」。源码层面可印证——桌面端配置存文件、会话存 IndexedDB，全部本地介质（docs/storage.md），付费云服务仅是可选 Provider 而非强制后端。

商业边界：内置 `Chatbox AI` Provider（`src/shared/providers/definitions/chatboxai.ts`）按 `licenseKey` 走官方代理模型；内置 MCP 服务器（mcp.chatboxai.app 的 fetch/arxiv/context7）需要 license（`packages/chatbox-core/src/mcp/builtin.ts`）；RAG 文档解析中 unstructured API 标注"付费用户可用"（docs/rag.md）。开源核心本身完整可用——用户自带任意 Provider 的 API Key 即可全功能使用。

## 3. 技术栈与依赖

- **桌面框架**：Electron 35 + electron-vite + electron-builder（package.json、electron-builder.yml）；`dev:web` / `build:web` 表明同一渲染层可出 Web 版（vite.config.web.ts）。
- **移动端**：Capacitor 7 全家桶（@capacitor/android、ios、sqlite 等，package.json），`scripts/build-mobile-work-worker.cjs` 打包移动 worker。
- **UI**：React 18 + Mantine 7 + MUI 5 + Tailwind + emotion，TanStack Router/Query，jotai/zustand 状态管理（package.json）。
- **模型接入**：Vercel AI SDK v7（`ai` 7.0.93）+ 官方 Provider 包 @ai-sdk/openai、anthropic、google、azure、deepseek、mistral、perplexity、amazon-bedrock、openai-compatible、@openrouter/ai-sdk-provider（package.json devDependencies，该仓把 SDK 依赖放 devDependencies 由构建统一处理）。
- **MCP**：@modelcontextprotocol/sdk + @ai-sdk/mcp 双客户端（packages/chatbox-core/src/mcp/controller.ts 头部导入）。
- **RAG**：@mastra/core + @mastra/rag + @libsql/client，libsql 作向量库（docs/rag.md：Mastra MDocument loader、vercel ai sdk embedding、libsql vector store）。
- **工具链**：pnpm 10 workspace、Biome（lint/format）、Vitest、husky + lint-staged、Storybook、TypeScript 5.8。
- **供应链安全**：pnpm-workspace.yaml 设 `minimumReleaseAge: 10080`（依赖发布满 7 天才允许安装）、`strictDepBuilds: true`（安装脚本白名单），并给 6 个上游包打了 patch（patches/）。

## 4. 目录结构与代码组织

pnpm workspace 单仓多包：主应用 + `packages/*` 共享包（pnpm-workspace.yaml）。

| 目录 | 职责（出处） |
|------|-------------|
| `src/main/` | Electron 主进程：MCP IPC 传输、skills 执行器、sandbox 管理器、知识库/RAG、oauth、deeplinks、自动更新、ripgrep 搜索等 |
| `src/preload/` | Electron preload 桥接 |
| `src/renderer/` | React 渲染层：routes/（TanStack Router 页面）、stores/（jotai/zustand）、storage/（会话行存储适配）、platform/（三平台适配） |
| `src/shared/` | 主/渲染共享：providers 注册表、model-registry、tools、skills、sandbox、token-estimation |
| `packages/chatbox-core/` | 框架无关领域层：application/domain/generation/persistence/session/mcp/ports（package.json 导出声明"React Native-safe"） |
| `packages/chatbox-react/` | 共享 React 组件库 |
| `packages/chatbox-i18n/` | 15 语言文案包（src/languages.ts） |
| `docs/`、`doc/` | 工程文档（storage.md、rag.md、technical/、adding-new-provider.md）与用户文档（FAQ、中文 README） |
| `test/` | 集成测试：model-provider（真实 API 冒烟）、mcp、file-conversation、context-management |
| `features/` | Gherkin BDD 用例（chat-streaming.feature 等 5 个） |
| `team-sharing/` | 团队共享 API 资源的说明文档（team-sharing/README.md） |
| `.erb/` | electron-react-boilerplate 衍生的构建脚本 |

组织方式评述：典型的"三进程 + 领域包"分层。亮点是把大部分业务逻辑抽进 `packages/chatbox-core`（含 ports/ 依赖倒置、persistence/ 纯 SQL 模块），使同一核心可跑在 Electron / Capacitor / Web 上；`docs/` 内含治理性文档（session-module-split-plan.md、dependency-reorg.md）显示有意识的模块边界治理，且有配套守卫脚本（`check:shared-boundaries`，package.json scripts）。

## 5. 核心架构与关键实现

### 5.1 多模型 Provider 注册表（约 30+ 内置 Provider）

`src/shared/providers/registry.ts` 用 `Map` + `defineProvider()` 实现注册表模式，`src/shared/providers/definitions/` 下每个厂商一个定义文件：claude.ts、openai.ts、openai-responses.ts、gemini.ts、azure.ts、bedrock.ts、ollama.ts、lmstudio.ts、openrouter.ts、github-copilot.ts、deepseek.ts、qwen.ts、moonshot.ts、zhipu-glm-coding-plan.ts、minimax.ts、volcengine.ts、siliconflow.ts、tencent-hunyuan.ts、xiaomi-mimo.ts、xai.ts、grok 系、perplexity.ts、mistral-ai.ts、chatboxai.ts 等（definitions/ 目录实测）。每个定义携带 `createModel` 工厂、平台可用性、默认设置；`src/shared/providers/api-style.ts` 把 Provider 类型映射到 4 种 API 风格（anthropic/google/openai/openai-responses），供推理控制与请求路由共享"单一事实源"。官方接入文档见 docs/adding-new-provider.md。

```ts
// src/shared/providers/registry.ts（节选）
export function defineProvider(definition: ProviderDefinitionInput): ProviderDefinition {
  if (providerRegistry.has(definition.id)) {
    console.warn(`Provider "${definition.id}" is already registered. Overwriting.`)
  }
  providerRegistry.set(definition.id, definition)
  return definition
}
export function isProviderAvailableOnPlatform(id: string, platformType: PlatformType): boolean {
  const platforms = providerRegistry.get(id)?.platforms
  return !platforms || platforms.includes(platformType)
}
```

本地模型走 Ollama/LM Studio 两个定义（definitions/ollama.ts、lmstudio.ts），对应用户隐私主张——可完全离线使用。

### 5.2 会话存储：KV 三介质 + configVersion 迁移机

docs/storage.md 系统记录了存储策略：桌面端 Settings/Configs 存文件、Sessions 存 IndexedDB（localforage）；移动端统一 SQLite（Capacitor）；Web 端 IndexedDB。版本历史表从 v1.9.8 演进到 v1.19.0/v1.21.0，每次变更伴随 `configVersion` 递增迁移函数链。会话本身在 `packages/chatbox-core/src/persistence/` 已从"单一列表"演进为**行式 SQL 存储**（session-row-sql.ts、session-graph.ts、session-branch-*.ts 等 40+ 文件），支持会话树/分支/线程链接（thread-links）与 token 缓存（session-token-cache-sql.ts），并有 `check:shared-boundaries` 脚本守住"行存储读取边界"（package.json scripts）。

```ts
// docs/storage.md（迁移核心逻辑节选）
const [oldConfigVersion, oldStorage] = await findNewestStorage(getOldVersionStorages())
if (oldStorage && oldStorage.getStorageType() !== storage.getStorageType()) {
  await doMigrateStorage(oldStorage)   // 跨介质迁移
}
for (; configVersion < CurrentVersion; configVersion++) {
  await migrateFunctions[configVersion]?.(dataStore)  // 增量 schema 升级
}
```

### 5.3 MCP 双协议客户端 + 付费内置服务

`packages/chatbox-core/src/mcp/controller.ts` 同时接入官方 MCP SDK（@modelcontextprotocol/sdk 的 SSE/StreamableHTTP legacy 传输）与 @ai-sdk/mcp 协商式客户端，统一转成 AI SDK 的 `ToolSet`（`dynamicTool`），含超时/中止/错误格式化。`src/main/mcp/ipc-stdio-transport.ts` 把 stdio 类型 MCP 服务器的进程宿主放进主进程（规避渲染进程权限）。`builtin.ts` 定义 3 个官方托管 MCP 服务器（fetch/arxiv/context7，域名 mcp.chatboxai.app），配置函数签名 `getBuiltinServerConfig(id, licenseKey)` 直接体现 license 门槛。

### 5.4 本地 Agent 执行栈：Skills + Sandbox

`src/main/skills/` 实现 Agent Skills 体系：builtin 目录 + 用户技能安装（installer.ts）、gray-matter 解析 SKILL.md 前置元数据（parser.ts）、GitHub 技能抓取（github-fetcher.ts）、用户命令执行器 user-exec-runner.ts 配合 `src/main/command-execution-policy.ts` 白名单策略（且有 windows 专项测试 user-exec-runner.windows.test.ts）。`src/main/sandbox/` 提供 exec-script、file-read、path-safety、preview-server、persist-artifact 等沙箱原语，manager.ts 统一管理。这构成桌面端本地 agent（跑脚本/改文件/起预览）的安全层，`src/shared/task-sandbox.ts` 与渲染层联动。知识库 RAG 见 `src/main/knowledge-base/`（parsers 分免费 officeparser 与付费 unstructured 两路，knowledge-base/parsers/）。

### 5.5 生成管线

`packages/chatbox-core/src/generation/`（20+ 文件）是会话生成中枢：generation-flow.ts 编排请求、stream-chunk-processor.ts 处理流式分片、MidRunCompactionController.ts 做运行中上下文压缩（mid-run-compaction.ts）、legacy-tool-fallback.ts 兜底不支持工具调用的模型（对应 features/legacy-tool-fallback.feature BDD 用例）、prompt-cache-policy.ts 针对各家 prompt cache 优化（test/integration/model-provider/claude-prompt-cache-live.test.ts 有真机验证）。

## 6. 构建与运行

来自 package.json scripts（Node >=22.13、pnpm >=10.17）：

```bash
pnpm install            # postinstall 跑 .erb/scripts/postinstall.cjs
pnpm dev                # electron-vite dev 开发模式
pnpm dev:web            # 仅 Web 渲染层开发
pnpm build              # build:main + build:preload + build:renderer
pnpm package            # electron-builder 打包（UPDATE_CHANNEL=alpha）
pnpm package:all        # --win --mac --linux 三平台
pnpm test               # vitest run
pnpm test:unit          # 排除集成测试
pnpm test:integration   # test/integration，300s 超时
pnpm test:model-provider # RUN_MODEL_PROVIDER_TESTS=1 真 API 冒烟
pnpm lint               # biome lint .
pnpm check              # 路由生成 + tsc --noEmit 全量类型检查
```

打包产物（electron-builder.yml）：mac（dmg）、win（nsis，`custom_win_sign.js` 签名）、linux（AppImage），productName=Chatbox，appId=xyz.chatboxapp.app。

## 7. 测试与工程质量

- **规模**：全仓 792 个 *.test.ts(x) 文件（find 统计），Vitest 单测 + test/integration 集成测试（mcp、file-conversation、context-management、model-provider 真实 API 冒烟，按环境变量开关），另有 5 个 Gherkin .feature 文件（features/）以 BDD 描述行为。
- **平台专项**：存在 Windows 专属测试文件（manager.windows.test.ts、user-exec-runner.windows.test.ts、workspace-instructions.windows.test.ts，package.json test:windows），PowerShell 兼容单独处理（src/main/windows-powershell.ts）。
- **质量工具链**：Biome 一次性格式化+lint、husky/lint-staged 提交门禁、`check`（tsc --noEmit 8GB 内存）+ `check:packages` 各子包类型检查、自定义守卫脚本 `check:shared-boundaries`、`check:renderer-colors`、`check:error-i18n-keys`（均见 package.json scripts）。
- **供应链**：pnpm-workspace.yaml minimumReleaseAge=7 天 + strictDepBuilds 白名单 + 6 个补丁包，工程化程度显著高于同类社区项目。
- **CI**：仓库内无 .github/workflows 目录（tarball 与 .github/ 实测仅 FUNDING.yml、ISSUE_TEMPLATE、dependabot.yml、stale.yml），推测构建/发布流水线在私有 pro 仓或外部 CI 中——未确认。

## 8. 社区与维护状态

- **极其活跃**：pushed_at 2026-11-12（与分析日仅隔 2 天）；release 节奏密集，2026-11 月内连续发布 v1.24.2/v1.24.3/v1.24.4（gh api releases）。
- **星数仍在快涨**：任务快照 39,000，实测 42,323（gh api），约一个月 +3.3k。
- **贡献者 75 人**（gh api contributors 分页），但 package.json author 为单一作者 bennhuang（tohuangbin@gmail.com），核心开发疑似以公司团队为主，社区 PR 为辅——从"pro 仓双向同步"的机制看，大量实现可能先在私有仓发生，社区仓更像定期镜像 + issue 入口（open issues 1369 未确认是否 backlog 化）。
- 内部协作留痕：.claude/skills/i18n-translate（自带 AI 翻译技能 + glossary.json + 校验脚本）与 .codex/skills 符号链接，表明团队用 AI agent 工具做 i18n 流水线（.claude/ 目录实测）。
- 国际化规模：15 种 UI 语言（packages/chatbox-i18n/src/languages.ts），每语言一个 translation.json（zh-Hans 约 2009 行），由 .claude/skills/i18n-translate 自动化维护。

## 9. 亮点与风险

**亮点**
1. Provider 注册表 + API 风格映射的抽象干净，30+ 厂商/本地模型/自定义 OpenAI 兼容端点统一进 AI SDK 生态，新增 Provider 有文档模板（docs/adding-new-provider.md）。
2. 框架无关的 chatbox-core 领域包（ports/adapters、行式 SQL 会话存储、生成管线），一套核心跑 Electron/Capacitor/Web 三端，存储迁移机（configVersion）跨 13 个版本演进有完整文档。
3. 工程质量突出：792 测试文件、BDD 用例、边界守卫脚本、供应链年龄门禁与依赖补丁策略，在桌面开源客户端中属于第一梯队。
4. 本地优先隐私主张与实现一致：配置文件/IndexedDB/SQLite 全本地，Ollama/LM Studio 支持完全离线。
5. MCP 支持完整（stdio + HTTP/SSE 双栈、官方托管服务），叠加本地 Agent Skills + 命令策略 + 沙箱，已超出"聊天壳"定位。

**风险/局限**
1. GPL-3.0 + "社区版镜像 pro 仓"的混合模式：社区 PR 与私有开发同步存在冲突风险，外部贡献者的代码路径可能被同步覆盖（README 机制描述，影响未确认）。
2. 付费墙渗透较深：内置 MCP 服务器、unstructured 文件解析、官方 Chatbox AI 均需 license，开源版文档未逐一标注哪些 UI 入口免费可用。
3. 依赖面极大（package.json 约 250 个直接依赖、全量 41MB 源码 + 1MB lock 文件），Electron + Capacitor + Storybook + 双 UI 库并存，维护与审计成本高。
4. CI 配置不在仓内，社区 fork 无法复现其完整发布验证（.github/ 无 workflows，未确认）。

## 10. 总结评价

Chatbox 是目前开源桌面 LLM 客户端中商业化与工程化最成熟的项目之一：以"本地优先 + 多模型聚合"的产品主张拿下 40k+ 星，又靠 Chatbox AI 订阅、license 体系和 pro 仓同步维持商业闭环。其源码最有价值的部分是 chatbox-core 的领域分层（Provider 注册表、行式会话存储、生成管线、MCP/沙箱执行栈）以及跨 Electron/Capacitor/Web 的三端复用手法，适合想构建跨端 AI 应用或研究桌面 Agent 安全执行架构的开发者精读；对普通用户则是自带 API Key 即可零部署使用的多模型桌面客户端首选之一。
