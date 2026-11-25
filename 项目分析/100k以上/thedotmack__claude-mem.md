# thedotmack/claude-mem 项目分析

> 星数 104780（快照 2026-10-24，检索时 2026-10-30）｜主语言 TypeScript｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：为 Claude Code（及十余种 AI 编码 agent 宿主）提供跨会话持久记忆的插件——自动捕获每次工具调用，用 LLM 压缩成结构化"观察"，在新会话开始时按需回灌上下文。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | thedotmack/claude-mem | package.json |
| 版本 | 15.7.0 | package.json |
| 星数 / fork | 104,780 / 9,241（gh api，2026-10-30） | GitHub API |
| 许可证 | Apache-2.0 | LICENSE, package.json |
| 创建时间 | 2025-08-31 | GitHub API |
| 最近 push | 2026-10-29（每天有提交） | GitHub API |
| 代码规模 | src/ 639 个 TS 文件约 15.6 万行；tests/ 1130 个文件约 21.3 万行 | find 统计 |
| 贡献者 | 约 314 人（contributors 分页推算） | GitHub API |

## 2. 项目是什么，解决什么问题

LLM 编码 agent 的会话是"金鱼记忆"：每个新会话从零开始，上一个会话修过什么 bug、做过什么架构决策、踩过什么坑全部丢失。claude-mem 通过 Claude Code 的生命周期 hook 体系解决这个问题（README.md "How It Works"；plugin/hooks/hooks.json）：

1. **捕获**：`PostToolUse` hook 对每一次工具调用触发 `claude-mem hook claude-code observation`，把工具输入/输出打包进本地 spool 文件（`src/cli/handlers/observation.ts`）。
2. **压缩**：后台 worker 服务消费 spool，用 LLM（默认走用户自己的 Anthropic 计划，可选 claude-mem 云端/ OpenRouter / Gemini 密钥）把若干次工具调用压缩成带类型标签的"观察"（bugfix/feature/decision/discovery/security_alert 等，`plugin/modes/code.json` 定义 37 种模式配置）。
3. **存储**：观察写入本地 SQLite（FTS5 全文索引）+ 向量库，支持内容哈希去重（`src/services/sqlite/SessionStore.ts`）。
4. **回灌**：`SessionStart` hook 注入上一会话摘要与近期观察；会话中 Claude 通过 MCP 工具按"search → timeline → get_observations"三层渐进式检索（README "MCP Search Tools"，号称比直接取全文省 ~10x token；plugin/skills/mem-search/SKILL.md）。

项目名虽叫 claude-mem，实际已扩展为多宿主记忆层：Claude Code、Codex、OpenCode、Cursor、Windsurf、Grok Bot、Antigravity、OpenClaw 网关、Kimi、OMP/Pi/DeepSeek Harness 等均有集成入口（README Quick Start；src/integrations/）。它同时是一个开源插件和一个商业托管服务（cmem.ai 的 "CMEM Pro"）的开源核心，README 明确二者边界（docs/ip-boundary.md 被引用）。

## 3. 技术栈与依赖

- **语言/运行时**：TypeScript（ESM），Node ≥20.19、Bun ≥1.1.31（package.json engines）。worker 以 Bun 运行，hook 命令以 Node 起动。
- **运行依赖极简**：仅 better-auth（云端鉴权）与 sharp（媒体解码），其余全部 esbuild 内联进 bundle——express、bullmq、ioredis、zod、MCP SDK、@anthropic-ai/claude-agent-sdk、React 19 等都是 build-time devDependency（package.json `//dependencies-note` 有详细说明）。
- **LLM 接入**：Claude Agent SDK（观察生成）、OpenAI 兼容/OpenRouter/Gemini 多 provider（src/services/worker/ClaudeProvider.ts、OpenRouterProvider.ts、GeminiProvider.ts）。
- **本地嵌入**：@huggingface/transformers + onnxruntime-node，默认 all-MiniLM-L6-v2（与 Chroma 默认嵌入函数同维度，便于迁移），并预留多语言模型表（src/services/vector/LocalEmbedder.ts、models.ts）。
- **存储**：better-sqlite3（FTS5 全文检索 + sync outbox 增量表）、Chroma（向量，用户环境可退化）、云端运行时用 Postgres + pgvector + BullMQ/Redis（src/storage/postgres/、src/server/generation/ProviderObservationGenerator.ts）。
- **代码解析**：约 20 种 tree-sitter 语法包，用于智能文件读取/上下文裁剪（devDependencies）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| src/cli/ | hook 命令入口与 14 个事件 handler（observation/summarize/context/session-init 等） |
| src/npx-cli/ | `npx claude-mem` 安装器（install/doctor/uninstall/backup 等 11 个命令） |
| src/services/worker/ | 本地 worker：多 provider 观察生成、搜索、会话管理（ClaudeProvider.ts 1441 行等） |
| src/services/sqlite/ | 存储层核心，SessionStore.ts 4979 行 + SessionSearch.ts（FTS5） |
| src/services/vector/ | 本地嵌入器（transformers.js/onnx） |
| src/server/ | 自托管 server 运行时：BullMQ 生成管线、Postgres 仓储、MCP HTTP |
| src/servers/ | MCP server（mcp-server.ts）与宿主适配 |
| src/supervisor/ | worker 进程注册、健康检查、孤儿清理 |
| src/ui/viewer/ | React 19 web 查看器（/viewer 观察浏览、My Work 工作台） |
| plugin/ | 发布给 Claude Code 插件市场的产物：hooks/hooks.json、modes/、skills/（27 个技能）、sqlite/ 预编译产物 |
| docs/ | 文档站源码 + 27 种语言 README 翻译（docs/i18n/） |
| tests/ | 1130 个测试文件，按域分目录 |
| openclaw/ omp/ pi/ dsh/ hermes/ cursor-hooks/ 等 | 各宿主集成 |

组织方式：单包多运行时的 monorepo 式大仓——同一套核心同时产出 npm 包、Claude Code 插件、各宿主插件、自托管 server 和云端服务代码，靠 scripts/build-hooks.js 在构建期组装并施加 bundle 体积护栏。代码注释密度极高且大量引用 issue 编号，工程纪律严格（hook IO discipline、bundle-size guardrail）。

## 5. 核心架构与关键实现

### 5.1 Hook 集成层（plugin/hooks/hooks.json + src/cli/）

所有 hook 收敛为一条 shell-free 命令 `claude-mem hook claude-code <event>`，由安装器放到 PATH 的 launcher 转发。hooks.json 注册了 Setup/SessionStart/UserPromptSubmit/PreToolUse/PostToolUse(+Failure)/Stop/SubagentStop/SessionEnd 共 9 类事件，其中写类 hook 均 `async: true` 避免阻塞宿主。hook 进程有严格的"IO 纪律"：handler 必须是纯函数，stderr 全程缓冲防止第三方库输出泄漏进模型上下文，任何异常都以 no-op JSON 应答而非 exit 2（src/cli/hook-command.ts:107-114、src/shared/hook-io.ts）。

观察事件不直接进库，而是写 spool 文件由 worker 拉取，server 运行时则 POST /v1/events，5 秒超时内失败自动降级回本地 spool（src/cli/handlers/observation.ts，`SERVER_OBSERVATION_TIMEOUT_MS`）。还处理了大量真实世界边界：子 agent 观察默认跳过防配额耗尽（#2736）、`<private>` 标签内容剥离、密钥日志脱敏、Kimi/PreCompact 复用差异等。

### 5.2 记忆压缩管线（src/server/generation/ + src/services/worker/）

压缩的核心是 prompt-builder：把会话事件流（工具调用块）隐私剥离、截断、XML 转义后包进指令模板，要求模型返回若干 `<observation>` 块——含 title/facts/narrative/concepts/files_read/files_modified 字段（src/server/generation/providers/shared/prompt-builder.ts:109-216）。自托管 server 端用 BullMQ worker 逐 job 生成，带 600s provider 超时、按错误分类决定重试、paid-send 预算上限（ProviderObservationGenerator.ts:58-110）。

面对超长会话还有巧思：输入按字节预算（默认 600KB）保留"头部+尾部"——开头承载目标、结尾承载结果与未尽事项，丢弃中段（ProviderObservationGenerator.ts:91-113，注释里有生产实测数据：34MB 事件、155 万 token 的会话）。另有会话级 `Stop` hook 摘要、`ClaudeProvider.ts`（1441 行）处理 worker 运行时的 SDK 对话。

### 5.3 存储与检索层（src/services/sqlite/ + src/services/vector/）

SessionStore.ts 近 5000 行，含 50+ 次迁移的演进痕迹：observations/sdk_sessions/user_prompts 等表、FTS5 external-content 索引及同步触发器、token_df + observation_dedup_candidates 实现的内容哈希/近似去重、以及 sync_entity_heads/sync_outbox 支撑云端增量同步（SessionStore.ts:80-864）。检索是混合式：FTS5 关键词 + 本地 MiniLM 向量（EMBED_CHUNK_SIZE=4 防止 onnx 阻塞事件循环，LocalEmbedder.ts），MCP 三步渐进检索按 ID 过滤后再取全文。多语言有明确认知：默认英文模型对 CJK 召回差，models.ts 预置了多语言候选与评测仓引用（#3422/#3982）。

### 5.4 多宿主分发与运行时选择

同一 hook 语义适配十余个宿主（src/cli/adapters/ 归一化各家 stdin JSON 差异），运行时可选本地 worker（默认，Bun 管理的 HTTP 服务，端口 37700+uid%100，src/shared/SettingsDefaultsManager.ts:439）或自托管/云端 server（Postgres 路线），server 不可用自动回退本地（src/services/hooks/runtime-selector.ts）。

## 6. 构建与运行

```bash
# 用户安装（注册插件 hooks + worker 服务）
npx claude-mem install
# 或 Claude Code 内
/plugin marketplace add thedotmack/claude-mem && /plugin install claude-mem

# 开发（package.json scripts）
npm run build          # sync-plugin-manifests + build-hooks(esbuild, 含 bundle 体积护栏) + 生成 lockfile
npm run dev            # build-and-sync + 重启 marketplace worker
npm run typecheck      # 根 + viewer 两套 tsc --noEmit
npm test               # bun scripts/test-lanes.ts fast（单元）
npm run test:integration  # 真实 Chrome / tree-sitter 构建 / e2e 安装
npm run test:all       # 两车道全跑（CI 同款）
npm run worker:start   # bun plugin/scripts/worker-service.cjs start
```

要求 Node ≥20.19、Bun ≥1.1.31；向量检索需 uv/Python（缺失自动安装）（README System Requirements）。

## 7. 测试与工程质量

- 测试规模 1130 个文件约 21.3 万行，超过源码本身；bun test 执行，scripts/test-lanes.ts 把套件切成 fast/integration 两车道并有 `check` 命令证明两车道不相交且全覆盖。
- CI（.github/workflows/ci.yml）：先验证"已提交的预编译插件产物与源码一致"（防陈旧 bundle），再 typecheck → build（bundle 超限直接挂 CI）→ 单测 → Docker 内 pg+redis 完整 e2e；另有 windows.yml、npm-publish.yml、ods-native.yml 等 10 个 workflow。
- 代码质量观察：注释引用 issue 编号的密度罕见（如 #2736/#5079/#4734），关键数值都附实测依据（hook 启动耗时、onnx 批大小阻塞毫秒数、生产会话 token 分布）；自研 lint 约束 hook IO 与 spawn 环境（npm run lint:hook-io / lint:spawn-env）。工程化水平在同类开源项目中属第一梯队。

## 8. 社区与维护状态

- 极高活跃度：仓库 2025-08-31 创建，14 个月到 104k 星；gh api 显示最近 push 2026-10-29；CHANGELOG.md 单文件 53 万行/超 5000 条目，15.5.0 与 15.6.0 同日（2026-10-29）发布，一天内多次发版常见。
- 约 314 位贡献者；issue 编号已到 #5100+（合并节奏极快）。Vercel OSS Program、Greptile、SerpApi 赞助（README 顶部徽章），入选 awesome-claude-code 与 Trendshift。
- 商业实体驱动：cmem.ai 托管记忆（CMEM Pro，30 天试用后回退用户自有 provider）、TeamBrain 团队版；甚至发行了社区代币 CMEM（README 末尾附 BASE 合约地址，作者官方背书）。开源 Apache-2.0 与托管服务的边界在 docs/ip-boundary.md 约定。

## 9. 亮点与风险

**亮点**
1. 工程纪律顶尖：hook IO 纪律、bundle 体积护栏进 CI、预编译产物一致性检查、两车道测试——这些是很多万星仓库都没有的机制。
2. 压缩管线设计务实：字节预算 + 保头保尾截断、错误分类重试、paid-send 预算、多层降级（server→worker spool→no-op），把 LLM 管线当分布式系统做。
3. 渐进式检索省 token：search/timeline/get_observations 三层 MCP 工作流是可复用的上下文工程范式。
4. 宿主覆盖面无出其右：十余种 agent 宿主 + 37 种模式（含多语言）+ 27 种技能。
5. 注释即文档：源码注释详尽解释"为什么"，含大量实测数据。

**风险/局限**
1. 复杂度极高：15.6 万行源码、双运行时（worker/server）、双存储路线（SQLite/Postgres），个人读懂并安全改造的成本大；发版太快，跟进门槛高。
2. 采集面广：hook 捕获所有工具调用的输入输出（含文件内容），虽有 `<private>` 标签与脱敏，敏感仓库使用者需自行评估；遥测用 PostHog（src/services/telemetry/）。
3. 商业引导明显：安装流程默认引导注册 cmem.ai（可用 `--provider` / `CLAUDE_MEM_ONLINE_OPTIN=false` 跳过，README 有说明），试用推广文案在 hook 输出中出现（src/shared/pro-promo.ts）。
4. 质量依赖 observer 模型：压缩质量随所选 provider/模型波动，本地免费路线需消耗用户自己的 Anthropic 计划。

## 10. 总结评价

claude-mem 是"AI agent 持久记忆"赛道的头部开源实现：以极轻的 hook 接入面 + 重型后台管线（LLM 压缩 → SQLite/FTS5 + 向量混合存储 → 渐进式回灌）把跨会话记忆做成了一套可运维的分布式系统，并用罕见的工程纪律（护栏进 CI、注释带实测数据、1100+ 测试文件）支撑 14 个月冲到 10 万星的高速迭代。适合三类人：想给自己的 Claude Code/Codex 等工作流加持久记忆的日常用户（`npx claude-mem install` 即用）；做 agent 记忆/上下文工程的开发者（其 prompt-builder、去重、混合检索、渐进披露设计都值得精读）；以及研究插件生态运营的人（多宿主分发 + 开源核心/托管服务双轨模式是教科书级案例）。读源码建议从 plugin/hooks/hooks.json → src/cli/hook-command.ts → src/cli/handlers/observation.ts → src/server/generation/providers/shared/prompt-builder.ts → src/services/sqlite/SessionStore.ts 这条主线切入。
