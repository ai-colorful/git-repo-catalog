# openclaw/openclaw 项目分析

> 星数 392,842（快照 2026-10-24）｜主语言 TypeScript｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：**OpenClaw 是一个自托管的个人/团队 AI 助手网关——在你的设备上运行一个常驻 Gateway，通过插件接入 WhatsApp、Telegram、Discord、Slack、iMessage 等 20+ 消息渠道，把任意模型（Claude/Codex/GPT/本地模型）变成"真的能动手做事"的助手。**

| 项目 | 值 |
| --- | --- |
| 名称 | openclaw/openclaw（前身 Clawdbot → Moltbot，见 `VISION.md`） |
| 星数 | 393,357（`gh api` 2026-10-30 实测；任务快照 392,842） |
| 语言 | TypeScript（含 Swift/Kotlin 原生 App、Rust crates、SolidJS UI） |
| 许可证 | MIT（`LICENSE`，Copyright 2026 OpenClaw Foundation） |
| 版本 | 2026.10.8（CalVer，`package.json`；最新 release v2026.10.9 发布于 2026-10-29） |
| 最近活跃 | 2026-10-29（`gh api pushed_at`），创建于 2025-11-24 |
| 贡献者 | 384 人（`gh api /contributors` 分页计数） |
| 维护方 | OpenClaw Foundation（独立 501(c)(3)），作者 Peter Steinberger（`README.md`） |

## 2. 项目是什么，解决什么问题

OpenClaw 要解决的问题是：现有 AI 助手要么是托管服务（数据不在自己手里），要么只能聊天不能执行。它把"助手的大脑"放在你自己的机器上——README 明确承诺 *"State, memory, and credentials live on your hardware"*，默认只做每日版本检查、功能统计需主动开启（`README.md`）。

形态上，它是一个**常驻 Gateway 守护进程 + 多端接入**的架构：Gateway 是本机控制平面，持有会话、工具、事件和渠道连接；CLI、Control UI（Web）、TUI、macOS/iOS/Android 原生 App 都作为 WebSocket 客户端接入；消息渠道插件把 WhatsApp/Telegram/Discord 等对话引到同一个助手（`README.md` "How it fits together" 一节、`docs/concepts/architecture.md`）。同一个 Gateway 既可做个人助手，也可做团队共享部署，"配置是唯一区别"。

模型与 agent 框架全部**插件化**：文档把 Provider（认证与模型发现）、Model、Agent Runtime（驱动一轮推理的循环）、Channel（消息出入口）分为四层，内置 `openclaw` 嵌入式 harness，另支持 `codex`、`copilot`、`muse` 等 harness 与 `claude-cli` 这类 CLI 后端（`docs/concepts/agent-runtimes.md`）。这意味着你可以换底层 agent 框架而不动会话与记忆。

安全模型是其宣传核心："trusted gateway, untrusted execution"——入站消息默认视为不可信，DM 渠道默认配对（pairing）审批，工具默认跑在宿主机上、可配置沙箱（`README.md` Security 节、`docs/gateway/security/`）。项目由基金会而非公司运营，"no paid tier, hosted service, or token"。

增长背景值得一提：仓库 2025-11-24 创建，约 11 个月冲到 39 万星（`gh api created_at`），其病毒式出圈（龙虾吉祥物 Molty、steipete 个人影响力）与产品本身的完成度共同推动了星数。

## 3. 技术栈与依赖

- **核心语言**：TypeScript（Node 24.22.1+/26 推荐，`README.md`）；76 个直接依赖（`package.json`）。
- **构建**：pnpm workspace（`pnpm-workspace.yaml`）+ tsdown（rolldown 系）；大量自研构建/检查脚本（`scripts/`）。
- **关键依赖**：
  - 模型 SDK：`@anthropic-ai/sdk`、`openai`、`@google/genai`、`@modelcontextprotocol/sdk`（MCP 客户端）；
  - 消息渠道：`baileys`（WhatsApp）、`grammy`（Telegram）、Discord/Slack/Signal 等各自插件内实现（`extensions/`）；
  - 服务与协议：`ws`、`express` 5、`typebox`/`zod`（schema 校验）、`kysely` + `pg`（数据库）、`undici`；
  - 浏览器/系统操作：`playwright-core`、`chrome-devtools-mcp`、`@lydell/node-pty`、`@trycua/cua-driver`（computer use）；
  - UI：SolidJS（Control UI，`ui/`；pnpm-workspace 中对 solid-js rc 的豁免注释可证）、`@earendil-works/pi-tui`（TUI，致谢 Mario Zechner 的 pi 项目）。
- **多语言组件**：`apps/macos`、`apps/ios`（Swift）、`apps/android`（Kotlin，含 wear）、`apps/linux`；`crates/` 两个 Rust crate（openclaw-gateway-client、openclaw-node-host，`crates/Cargo.toml`）。
- **存储**：SQLite（默认，含自研 `node-sqlite.mjs` 加载器）或 PostgreSQL；`package.json` 声明 `databaseEngines: ["sqlite","postgres"]`，state/agent 两大库族 schema 版本 36/38。

## 4. 目录结构与代码组织

pnpm workspace 单仓 monorepo，根包即主程序 `openclaw`（bin: `openclaw.mjs`）。

| 目录 | 职责 |
| --- | --- |
| `src/` | 主程序全部核心源码（296MB，约 626 万行 TS，其中 328 万行为测试） |
| `src/gateway/` | Gateway 守护进程（2,798 个文件）：WS 服务、方法分发、认证、调度 |
| `src/agents/` | agent 运行时：嵌入式 agent loop、harness 选择、压缩、MCP 绑定 |
| `src/agents/tools/` | 内置工具（computer-use、cron、gateway、ask-user 等） |
| `src/plugins/` | 插件系统：激活计划、bundle 能力、API facade |
| `src/channels/` | 渠道抽象与已加载插件注册表 |
| `src/memory/` + `extensions/memory-lancedb` 等 | 记忆系统（文件+SQLite 为主，LanceDB 向量可选） |
| `extensions/` | 189 个插件包：178 份 `openclaw.plugin.json`，其中 30 个渠道插件，其余为 ~100 个模型 Provider、诊断、浏览器等 |
| `packages/` | 24 个内部分享包（llm-core、agent-core、plugin-sdk、gateway-protocol 等） |
| `ui/` | Control UI（SolidJS + Vite，172MB） |
| `apps/` | macOS/iOS/Android/Linux 原生 App |
| `skills/` | 51 个捆绑技能（SKILL.md 形态：github、obsidian、apple-notes 等） |
| `docs/` | 完整文档站源文件（36MB，含 `docs/concepts/` 概念文档） |
| `test/` | 跨模块 e2e/集成测试与 vitest 项目矩阵配置 |
| `security/opengrep/` | 自带静态安全规则；`.github/workflows/` 40+ 条 CI 流水线 |

组织方式评述：这是典型的"**核心单体 + 插件星系**"结构——核心（gateway/agents/plugins）体量极大且内部按"一个职责一个文件"高度切分（如 `src/agents/compact.*.ts` 系列十几份文件），可独立复用的协议/算法下沉到 `packages/*`，而增长最快的渠道与 Provider 全部隔离在 `extensions/*`，由插件清单与 SDK 约束边界。

## 5. 核心架构与关键实现

### 5.1 Gateway：WebSocket 控制平面

单一长驻进程持有所有消息面；控制面客户端与设备节点（`role: node`）连同一个 WS 端口（默认 `127.0.0.1:18789`）。协议为 `{type:"req",id,method,params}` → `{type:"res"}` + 服务端推送 `{type:"event"}`，首帧必须是 `connect`，副作用方法（`send`/`agent`）强制幂等键（`docs/concepts/architecture.md`）。方法发现由核心方法表与渠道插件描述符合并而成（`src/gateway/server-methods-list.ts`）：

```ts
export function listGatewayMethods(): string[] {
  const coreMethods = listCoreAdvertisedGatewayMethodNames();
  const methods: string[] = [];
  for (const plugin of listLoadedChannelPlugins()) {
    methods.push(...(plugin.gatewayMethods ?? []));
    for (const descriptor of plugin.gatewayMethodDescriptors ?? []) {
      methods.push(descriptor.name);
    }
  }
  return Array.from(new Set([...coreMethods, ...methods]));
}
```

信任建立相当严谨：设备身份 + 配对审批 + 设备 token + `connect.challenge` nonce 签名（v3 绑定平台/设备族），环回连接才可自动批准（`docs/concepts/architecture.md` "Pairing and local trust"）。

### 5.2 Agent Loop：嵌入式运行时与模型故障转移

一轮对话的流水线为：`agent` RPC 校验并立即返回 `{runId}` → `runEmbeddedAgent` 预留会话控制器 turn、全局容量准入、解析模型与 auth profile → 订阅运行时事件流式输出（tool/assistant/lifecycle 三类 stream）→ `agent.wait` 等终态（`docs/concepts/agent-loop.md`）。入口实现 `src/agents/embedded-agent-runner/run-entry.ts`：

```ts
/** Runs one logical turn across model candidates and advances only the accepted winner. */
export async function runEmbeddedAgentEntry<T extends EmbeddedAgentRunResult>(
  params: EmbeddedAgentRunEntryParams<T>,
): Promise<EmbeddedAgentRunEntryResult<T>> {
  const admission = params.preparedRunAdmission;
  const requester = { ...params.identity, preparedRunAdmission: admission, ...
```

围绕它有三层值得注意的设计：

1. **并发纪律**：session controller 先按物理会话串行化，再进全局容量；渠道消息另有 steer/followup/collect/interrupt 四种排队模式（`docs/concepts/agent-loop.md`、`docs/concepts/queue.md`）。
2. **写入门禁（writer fencing）**：准入后记录持久化 `activeWriterRunId`，每次 transcript 追加都带 `expectedWriterRunId`，SQLite 同步提交事务校验匹配，被取代的 run 无法写入陈旧数据；同目录另有状态目录锁防双 Gateway（同上文档）。
3. **模型故障转移**：`runWithModelFallback`（`src/agents/model-fallback-runner.ts`）在多个模型候选间推进，只有被接受的 winner 才推进 turn 状态；配合 `failover-error.ts`、`fallback-skip-cache.ts` 等。

### 5.3 Harness 选择与工作区注入

`src/agents/harness/selection.ts` 在每轮运行时选择 agent harness（内置 `openclaw` 或插件 harness），注入执行环境策略（工具允许表、operator 权限、秘钥哨兵解包等）。助手人格与记忆来自**工作区引导文件**：`AGENTS.md`/`SOUL.md`/`IDENTITY.md`/`USER.md`/`BOOTSTRAP.md`/`MEMORY.md` 首轮注入 system prompt，空白文件跳过、超长截断；`BOOTSTRAP.md` 的首次仪式有 SQLite attestation 防止被清空后静默重播（`docs/concepts/agent.md`）。

### 5.4 记忆系统

Memory Core 遵循"无隐藏状态"原则：模型只记得写进工作区文件的内容，全部记忆可用文本编辑器查看；写入与检索分离——检索用文件+SQLite 索引，策展挪到后台 pass（"dreaming"），并有分级信任与投毒防御（`docs/concepts/memory-architecture.md`）。向量检索为可选插件（`pnpm-workspace.yaml` 中 LanceDB 依赖豁免注释、`package.json` 的 `!dist/extensions/memory-lancedb/**`）。

### 5.5 插件体系

每个插件以 `openclaw.plugin.json` 声明 id、类别（channels 等）、激活时机、契约（如 Discord 声明 `transcriptSourceProviders`、doctor 修复契约，`extensions/discord/openclaw.plugin.json`）。`src/plugins/activation-planner.ts` 决定按需激活，`packages/plugin-sdk` 提供公开 API，能力面（skills、MCP bundle、LSP、诊断）通过 bundle 机制挂接（`src/plugins/bundle-*.ts`）。

## 6. 构建与运行

来自 `README.md` 与 `package.json` scripts（未实际运行）：

```bash
# 安装（脚本安装器，自动准备 Node）
curl -fsSL https://openclaw.ai/install.sh | bash        # macOS/Linux/WSL2
npm install -g openclaw@latest --allow-scripts=openclaw  # 或 npm 包（npm 11.16+/12）
openclaw onboard --install-daemon                        # 引导式初始化
openclaw gateway status && openclaw dashboard            # 查看状态/打开控制台

# 源码开发（必须 pnpm，npm install 不支持）
pnpm install && pnpm build && pnpm ui:build
```

常用 scripts：`pnpm check`（聚合自检）、`pnpm check:architecture`（import 环、DB 方言 ratchet 等）、`pnpm test`（vitest 矩阵，`vitest.config.ts` → `test/vitest/vitest.config.ts`）、Android 侧 `pnpm android:*`（Gradle/fastlane 封装）。容器部署有 `Dockerfile`、`docker-compose.yml`、`fly.toml`、`render.yaml`。

## 7. 测试与工程质量

- **规模**：`src/` 内测试文件 11,891 个 / 328 万行（占 src 一半以上），另有 `test/` 顶层目录承载 e2e 与跨模块集成（`test/cli-json-stdout.*.e2e.test.ts` 系列、`*.live.test.ts` 真实环境测试）。命名即分层：`*.test.ts` 单元、`*.integration.*`、`*.e2e.*`、`*.live.*`。
- **框架**：Vitest（workspace 项目矩阵，根 `vitest.config.ts` 指向 `test/vitest/vitest.config.ts`），patchedDependencies 中还包含对 vitest 5.0.5 的补丁。
- **CI**：`.github/workflows/` 下 40+ 流水线：主 `ci.yml` 按小时 cron 调度、多 runner 形态（testbox 分箱、Bun/Node 双运行时）；CodeQL（含 Android/macOS 专项）、依赖审计、Android 发布 e2e、Docker 发布链。
- **自研质量门槛**（`package.json` scripts 可见）：import 环检查（自研+madge 双份）、max-lines/line-cap ratchet（文件行数棘轮）、断言安全棘轮、测试超时竞态棘轮、DB 方言/worker 棘轮、配置 schema 生成一致性检查、`security/opengrep/` 自带安全规则。供应链方面 pnpm 配置 7 天依赖冷却（`minimumReleaseAge: 10080`，`pnpm-workspace.yaml`），并对每个豁免条目写明审批人与到期时间。

工程质量总体印象：在"约束棘轮 + 生成物一致性 + 分层测试命名"上做到罕见的极致，明显按大型团队纪律运作。

## 8. 社区与维护状态

- **活跃度**：推送至 2026-10-29（`gh api`）；CHANGELOG 按版本逐日成档（`CHANGELOG.md` 列表从 2026.5.28 到 2026.10.8 密集分布），release 节奏接近每日/隔日 CalVer。
- **贡献者**：384 人（分页计数）；单个补丁版本 2026.10.8 即含 161 个 PR、28 个直接提交、163 位贡献者（`CHANGELOG/2026.10.8.md` "Release scale"）。
- **治理**：OpenClaw Foundation（501(c)(3)）雇佣核心团队并签署 release，README 强调 OpenAI 只是 donor 非所有者。
- **issue 压力**：open issues 7,863、fork 8.27 万（`gh api`），高热度带来的支持负担可见一斑。
- **安全响应**：版本说明含明确的安全修复记录（如 2026.10.8 针对 eMudhra 证书妥协更新出口信任包，`CHANGELOG/2026.10.8.md`），配 `SECURITY.md` 与 `security/` 目录。

## 9. 亮点与风险

**亮点**

1. **架构分层清晰且文档化**：Provider/Model/Runtime/Channel 四层解耦（`docs/concepts/agent-runtimes.md`），换模型、换 agent harness、换渠道互不影响。
2. **并发与一致性设计扎实**：会话串行化 + 全局准入 + transcript writer fencing（`docs/concepts/agent-loop.md`），这类细节决定了"助手同时接多条聊天"时的正确性。
3. **隐私与安全默认值认真**：状态全本地、配对审批、nonce 签名握手、沙箱与 secret egress 代理、自带 opengrep 规则。
4. **工程纪律罕见**：1.2 万测试文件、十余种 ratchet 检查、7 天依赖冷却并逐条写豁免理由。
5. **全平台覆盖真实存在**：Swift/Kotlin 原生 App + Rust gateway client + SolidJS UI + TUI，不只是 Node 单体。

**风险/局限**

1. **体量巨大、上手门槛高**：仅 `src/` 就约 626 万行 TS（含测试），单 `src/gateway` 近 2,800 个文件，二次开发与审计成本高。
2. **迭代极快带来的不稳定**：日更级 release 与频繁改名史（Warelay→Clawdbot→Moltbot→OpenClaw）意味着配置面与 API 持续变动。
3. **安全敏感面天然存在**：工具默认跑在宿主机、直连个人消息账号（如 baileys 非官方 WhatsApp），README 自己警告"入站消息视为不可信、暴露前必读 runbook"；错误配置后果重。
4. **星数含大量话题热度**：11 个月 39 万星远超常规开源曲线，实际部署者/贡献者比例（fork 8.3 万、issue 近 8 千）提示相当部分为关注度而非深度使用。

## 10. 总结评价

OpenClaw 是 2026 年"个人 AI 助理基础设施"方向最具野心的开源实现：它把 Gateway（控制面）、agent runtime（可插拔 harness）、渠道插件、原生多端 App 和一套偏执的工程纪律装进了同一个 monorepo，且文档质量与代码一致性达到商业项目水准。它不适合想读"小而美 agent loop"的人——仓库太大；但它非常适合三类人群：想自托管一个跨消息渠道、能真正操作电脑的助手的进阶用户；需要参考"多渠道高并发 agent 网关"生产级设计（writer fencing、配对信任、故障转移、插件激活）的工程师；以及研究 AI agent 安全模型（不可信入站 + 可信网关 + 沙箱执行）的安全研究者。读源码建议从 `docs/concepts/architecture.md` 与 `docs/concepts/agent-loop.md` 入门，再对照 `src/gateway/server-methods-list.ts` 与 `src/agents/embedded-agent-runner/run-entry.ts` 落到实现。
