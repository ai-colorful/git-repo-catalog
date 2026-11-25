# modelcontextprotocol/servers 项目分析

> 星数 91516（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：MCP（Model Context Protocol）官方参考服务器集合——用官方 SDK 演示协议全部特性的"教科书式"示范仓库，由 MCP 指导组/Anthropic 维护。

| 项目 | 内容 |
| --- | --- |
| 名称 | modelcontextprotocol/servers |
| 星数 | 91516（快照），fork 11891，watchers 630（gh api，2026-10-31） |
| 主语言 | TypeScript（4 个 TS 包）+ Python（3 个 Py 包） |
| 许可证 | 许可过渡期：新代码 Apache-2.0，历史代码 MIT（`LICENSE` 首段声明，GitHub 标注 NOASSERTION） |
| 创建时间 | 2024-11-19（gh api） |
| 最近活跃 | 2026-10-29 仍有 push；最新 Release v2.2.0（2026-10-28） |
| 包结构 | npm workspaces monorepo，7 个独立发布的包（`package.json` 的 `workspaces: ["src/*"]`） |
| 贡献者 | 427 人（gh api contributors），但实际提交集中于少数维护者（CHANGELOG 证据） |

## 2. 项目是什么，解决什么问题

MCP 是 Anthropic 于 2024 年 11 月开源的"LLM 与外部工具/数据源之间的标准连接协议"，本仓库是协议官方的参考实现集合。`README.md` 开宗明义：仓库只收录"MCP 指导组维护的少量参考服务器"，其定位是**教育性示范**——演示 MCP 特性如何用官方 SDK 实现，"不是生产就绪方案"（`README.md` 第 8-9 行的 WARNING 框）。想找全量服务器列表，README 引导用户去 MCP Registry（registry.modelcontextprotocol.io）。

仓库历史上有 13 个参考服务器（GitHub、GitLab、Slack、PostgreSQL、Puppeteer、Brave Search、Google Drive 等），现已全部移入 `servers-archived` 仓库归档（`README.md` "Archived" 一节，第 37-53 行）。现在的策略是"少而精"：只保留 7 个最能教学价值的包——4 个 TypeScript（everything、filesystem、memory、sequentialthinking）+ 3 个 Python（fetch、git、time），每个独立发版到 npm/PyPI（`AGENTS.md` "Project structure" 节有完整清单）。

7 个包各有分工：`everything` 是协议全特性展示与客户端测试床（prompts/tools/resources/sampling/elicitation/tasks 全部覆盖）；`filesystem` 演示带安全边界的文件操作；`memory` 演示知识图谱持久化；`sequentialthinking` 演示逐步推理工具；`fetch`/`git`/`time` 则是 Python SDK 侧的对应示范。它们共同回答一个问题："一个规范的 MCP 服务器应该长什么样"。

值得一提的是仓库当前正处于 "V2 重写" 浪潮中：Release 历史显示 2026-10-11 发 v1.0.0、10-19 发 v2.0.0、10-25 发 v2.1.0、10-28 发 v2.2.0（gh api releases），changesets 的 baseBranch 为 `v2/main`（`.changeset/config.json`），代码注释中大量 `#4852`、`#4854` 等 issue 编号指向重构工作。星数能涨到 9 万+，很大程度是因为它是 MCP 生态的"官方起点"——几乎每个 MCP 服务器开发者的第一站。

## 3. 技术栈与依赖

**TypeScript 侧**（根 `package.json` + 各包 `package.json`）：
- Node >= 24（`src/filesystem/package.json` engines），TypeScript ^5.8，ESM（`"type": "module"`）
- MCP SDK v2：`@modelcontextprotocol/server` ^2.3.1（核心）、`@modelcontextprotocol/server/stdio`（serveStdio）、`@modelcontextprotocol/client` ^2.3.1（测试用客户端）
- zod ^4.2 做 schema 定义（`.meta()` 标注 JSON Schema 2020-12 方言，见 `src/filesystem/server.ts` 第 23-33 行）
- 辅助库：glob、minimatch、diff（filesystem）；chalk（sequentialthinking）
- 测试：vitest ^4.1.12 + @vitest/coverage-v8

**Python 侧**（`src/{fetch,git,time}/pyproject.toml`）：
- mcp >= 2.3,<3（Python SDK 的高级 `MCPServer` API）、pydantic >= 2、hatchling 构建、uv 管理（uv.lock 在库）
- fetch：httpx[socks]、beautifulsoup4、markdownify、readabilipy、protego（robots.txt 解析）
- git：GitPython（含 gitdb）；time：tzdata、tzlocal
- 测试：pytest + pytest-asyncio + pytest-cov，类型检查 pyright，lint ruff

**工程工具链**（根 `package.json` devDependencies + `scripts/`）：changesets（版本管理）、eslint ^10 + typescript-eslint、prettier，以及 30 余个自研守卫/发布脚本（`scripts/*.mjs`，每个都配有 `.test.mjs`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `src/everything/` | TS，`@modelcontextprotocol/server-everything`，协议全特性展示服务器，25 个工具文件 + 4 种传输实现 + 自己的 docs/ |
| `src/filesystem/` | TS，`server-filesystem`，受限目录内文件操作；`server.ts`(934 行) + `lib.ts`(982 行) |
| `src/memory/` | TS，`server-memory`，知识图谱（实体/关系/观察）JSONL 持久化，1280 行单文件 |
| `src/sequentialthinking/` | TS，`server-sequential-thinking`，最小实现（229 行），动态思考序列工具 |
| `src/fetch/`、`src/git/`、`src/time/` | Python 包，`mcp-server-fetch/git/time`，标准 src 布局（`src/<pkg>/src/mcp_server_*/server.py`） |
| `scripts/` | 质量门与发布流水线脚本（gate-lease、smoke-servers、interface-diff、pack-and-verify、release-manifest 等，均带测试） |
| `docs/` | 贡献模型、质量门、skill 编写指南等元文档（`docs/quality-gate.md`、`docs/contribution-model.md`） |
| `AGENTS.md` | 50KB 的"AI 会话规则书"：仓库约定、技能索引、开发流程（面向 AI 辅助开发） |
| `.claude/skills/` | 11 个按需加载的操作技能（board-ops、pr-flow、issue-triage、pre-push-gate、release、testing 等） |
| `.github/workflows/` | 12 个 CI 工作流共 2397 行（typescript/python/release/everything-mcp-diff/sdk-watch/dco 等） |
| `.changeset/` | TS 包的 changesets 版本管理 |

组织方式评述：标准 npm workspaces monorepo，每个服务器是一个自治单元（自己的 package.json/pyproject.toml、自己的 tsconfig、自己的测试与 CHANGELOG），根目录只放共享工具链。TS 与 Python 两种技术栈在一库共存，靠统一的 `scripts/` 质量门（`validate:py`、`coverage:py`）拉齐工程标准。每个包结构高度一致（入口 `index.ts`/`__main__.py` + 工厂函数 + `__tests__`），可读性极强。

## 5. 核心架构与关键实现

### 5.1 服务器工厂模式 + 双协议时代支持（`src/filesystem/index.ts`、`src/everything/server/index.ts`）

所有 TS 服务器都采用"工厂建服务器、入口只做装配"的结构：`createServer()` 每次调用返回一个全新 `McpServer` 实例，模块导入不产生副作用（`isEntryPoint()` 守卫，`src/filesystem/index.ts` 第 146-165 行），使测试能在进程内用内存传输驱动真实服务器（注释自述对应 issue #4854）。入口通过 SDK v2 的 `serveStdio` 接一个**工厂**而非实例，SDK 按连接的协议时代调用它：

```ts
const handle = serveStdio(
  ({ era }) =>
    createServer(allowedDirectories, {
      era,
      onCannotOperate: () => process.exit(1),
    }),
  { transport: new StdioServerTransport(stdin, stdout), ... },
);
```

`src/everything/server/index.ts` 头部注释详细解释了"era"设计：2026-07-28 修订版客户端用 `server/discover` 无会话连接，能力随每个请求的 `_meta` 携带；2025 时代客户端走 `initialize` 握手 + roots 同步。同一个工厂按 era 注册不同的工具集（`registerModernTools` / `registerLegacyTools`），已废弃特性（logging、resources/subscribe）仅对 2025 客户端注册。这是理解 MCP 协议演进的第一手材料。

### 5.2 安全工程：路径校验与注入防护（`src/filesystem/path-validation.ts`、`src/git/src/mcp_server_git/server.py`）

filesystem 服务器把安全当头等事：`path-validation.ts` 的 `isPathWithinAllowedDirectories` 拒绝 null 字节、强制规范化后再比对允许目录；启动时用 `realpathWithRetry` 解析符号链接并同时保留原始/解析两种拼法以处理 macOS `/tmp -> /private/tmp`（`src/filesystem/index.ts` 第 33-61 行）；2026.10 月的 CHANGELOG 还显示它们为 iCloud Drive 等慢文件系统加了 15 秒 realpath 超时与 10 万条目遍历上限（`src/filesystem/CHANGELOG.md` 2.1.0 节）。Python 的 git 服务器则有专门的 `FlagInjectionError`，拒绝以 `-` 开头的参数防止 git 标志注入（`src/git/src/mcp_server_git/server.py` 第 40-46 行），注释明确"每个工具调用必须先过仓库限定与注入守卫"。

### 5.3 memory 服务器的跨进程文件锁与 JSONL 持久化（`src/memory/index.ts`）

memory 用 1280 行演示了一个完整的"小型持久化系统"：实体/关系/观察三元组的 JSONL 追加写，加上自研的跨进程锁——锁文件带 JSON 内容记录持有者进程，`isLockOwnerDead()` 检查进程是否仍存活、过期锁自动破坏，Windows 上 EPERM 也视为"被占用"（`src/memory/index.ts` 第 170-180 行附近的 `isLockBusy`/`isLockOwnerDead`）。还有从旧 `memory.json` 到 `memory.jsonl` 的迁移逻辑，且坚持"迁移失败绝不静默用空库启动"（`ensureMemoryFilePath`，注释引用 #5203）。

### 5.4 everything：协议特性全景 + 传输层（`src/everything/`）

everything 有 25 个工具模块（`src/everything/tools/`：sampling、elicitation、roots、结构化内容、注解、长任务、多轮 elicitation 等），4 个传输实现（`transports/stdio.ts`、`sse.ts`、`streamableHttp.ts` 404 行——含 in-memory 事件存储、`listen.ts`）。连示范用的工具都带安全设计：`get-env` 默认对 KEY/TOKEN/SECRET 等命名的环境变量打码（`src/everything/README.md` 与 CHANGELOG 2.1.0），`gzip-file-as-resource` 拒绝回环/私网/云元数据地址并在每个重定向跳重新校验（SSRF 防护）。

## 6. 构建与运行

来自根 `package.json` scripts 与各 README，均可用：

```sh
# 构建/测试/覆盖率（全部 workspace）
npm run build && npm test && npm run coverage

# 质量门（本地完整门 = CI 同款检查序列，docs/quality-gate.md）
npm run local:gate

# Python 侧
npm run validate:py -- <server>; npm run coverage:py -- <server>

# 直接运行服务器（README "Getting Started"）
npx -y @modelcontextprotocol/server-memory        # TS 服务器
uvx mcp-server-git                                # Python 服务器

# 无客户端试用：MCP Inspector
npx @modelcontextprotocol/inspector uvx mcp-server-git
npx @modelcontextprotocol/server-everything streamableHttp   # 唯一支持 HTTP 的服务器
```

注意：所有服务器默认 stdio 传输——启动后"看起来挂起"是正常的，它在等客户端从 stdin 发消息（`README.md` "Transports" 一节）。

## 7. 测试与工程质量

- 测试规模：TS 侧 57 个 `*.test.ts`、约 21700 行测试代码（`__tests__/` 统计），生产代码约 10750 行 TS——测试比超 2:1；Python 侧各包均有 tests/ 目录。filesystem 的测试覆盖 win32 行为、符号链接、CRLF 边界等平台细节（`src/filesystem/__tests__/win32.test.ts` 等 14 个测试文件）。
- 风格：**进程内协议级测试**——测试用 `@modelcontextprotocol/client` 对工厂构建的服务器发起真实 MCP 调用，而非 mock（`docs/testing` 技能、`src/everything/__tests__/harness.ts`）；测试几乎不 retry（有 `verify:no-test-retries` 守卫强制）。
- CI：12 个工作流（`.github/workflows/`），typescript.yml 在 Node 24/26 双版本矩阵 + Windows 专用腿；`everything-mcp-diff.yml`（340 行）在 SDK 协议变化时做接口差异检测（配合 `mcp-server-diff` 工具）；release.yml（478 行）将 build 与 publish 拆为两个 job，publish job 独占 OIDC token，且所有 action 钉死到 commit SHA（`verify:action-pins` 强制）。
- 发布工程：TS 包走 changesets（v2/main 分支），Python 包走 CalVer（版本号即日期，如 `2026.10.28`，`src/time/pyproject.toml`）；npm/PyPI 均 OIDC 可信发布、无长期 token（`.github/workflows/release.yml` 头注 + `RELEASING.md`）；发布幂等——已在 registry 的版本跳过而非失败。
- 代码质量观察：注释密度与质量罕见地高，几乎所有非常规代码都带"为什么"注释并引用 issue 号；每个 `scripts/*.mjs` 守卫都有自己的 `.test.mjs`；per-file 覆盖率门槛（`docs/quality-gate.md`）。

## 8. 社区与维护状态

- **极度活跃**：gh api 显示最近 push 为 2026-10-29（分析前一天），2026 年 10 月连续发 4 个 release（v1.0.0 → v2.2.0，8 天内跨大版本）。
- **维护者结构**：CHANGELOG 中几乎全部条目署名 @cliffhall（MCP 团队），427 名 contributors 主要是历史贡献。这不是普通开源协作仓库——`CONTRIBUTING.md` 明确宣布 **"We accept issues, not pull requests"**：外部贡献者只提 issue（若已本地实现，分享 prompt 而非代码），PR 仅限维护者本人，外部 PR 会被关闭。这是"AI 辅助、prompt 驱动开发"流程的组成部分（该文档"Why this policy exists"一节）。
- **AI 原生工作流**：50KB 的 `AGENTS.md` 定义 AI 会话规则，`.claude/` 下 11 个技能覆盖 issue 分诊、PR 流程、发布、安全通告全流程，甚至有 `session-instructions-check` 脚本校验会话指令是否过期。
- 仓库由 Anthropic 管理、LF Projects 托管（`package.json` author 字段、README 页脚）。
- 归档策略清晰：13 个旧服务器移至 `servers-archived` 独立仓库（`README.md` 第 37-53 行）。

## 9. 亮点与风险

**亮点**
1. MCP 生态的权威范本：想学 MCP SDK 用法（工具注册、zod schema、双协议时代、elicitation/tasks 扩展），这里是第一手且随协议同步更新的资料（`src/everything/docs/features.md` 有全特性清单）。
2. 工程纪律天花板：本地质量门与 CI 检查完全对齐（`npm run local:gate`）、发布 OIDC 化且幂等自愈、守卫脚本自己也有测试（`scripts/`、`docs/quality-gate.md`）。
3. 参考级安全实践：SSRF 防护、符号链接解析、flag 注入防护、秘密打码——示范代码本身就是安全教程（第 5.2/5.4 节所引路径）。
4. AI 辅助开发的活标本：AGENTS.md + skills + "只收 issue 不收 PR"构成一套完整可复制的 AI 原生维护模式。
5. 注释文化：关键设计决策全部就地注释并关联 issue 号，源码即设计文档。

**风险/局限**
1. 定位是教学参考而非产品：README 明确警告不可直接生产使用，需自行评估威胁模型。
2. 覆盖面窄：仅 7 个服务器，历史上的 GitHub/Slack/Postgres 等热门实现已归档他处，本仓库不再是"服务器大全"。
3. 传输限制：除 everything 外全部只有 stdio 传输，无 Streamable HTTP 部署范本（`README.md` transports 表格）。
4. V2 激进重构期：v1.0.0 与 v2.2.0 间隔仅 17 天，API 与约定仍在快速变动，学习材料需注意版本时效。

## 10. 总结评价

这是"官方参考实现"四个字的标杆形态：代码量不大（TS 生产代码约 1 万行 + Python 约 2000 行），但测试比 2:1、守卫层层设防、注释详尽到解释每个设计决策的来龙去脉，2026 年 10 月的 V2 重写更让它成为观察"MCP 协议 2026-07-28 修订版如何落地"的唯一窗口。它 9 万+ 星的来源并非代码本身的功能价值，而是整个 MCP 生态的入口地位——多数开发者认识 MCP 的第一站就是这里。适合三类人读源码：要写自己 MCP 服务器的开发者（照 filesystem/everything 学 SDK v2 用法与安全边界）、关心 AI 原生工程流程的维护者（研究 AGENTS.md + skills + issue-only 贡献模型）、以及做协议实现/一致性测试的工程师（everything 的双时代工厂与 interface-diff 工作流）。把它当教材用，别当组件库用。
