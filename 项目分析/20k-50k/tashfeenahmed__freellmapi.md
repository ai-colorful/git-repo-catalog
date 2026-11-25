# tashfeenahmed/freellmapi 项目分析

> 星数 51736（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-14
>
> gh api 实测（2026-11-14）：51,737 stars，7,211 forks，最近 push 2026-11-13（昨日），最新 release v0.16.28（2026-11-13 发布）。实测已超 50k 档线，按清单快照归入 20k-50k 档。

## 1. 项目速览

一句话定位：**本地优先的 LLM 网关**——把几十家供应商的免费额度聚合到一个 OpenAI 兼容的 `/v1` 端点后面，自带智能路由、自动故障转移、逐 key 配额跟踪与加密存储。

| 项目 | 值 | 出处 |
|---|---|---|
| 名称 | tashfeenahmed/freellmapi | gh api |
| 星数 | 51,736（快照）/ 51,737（实测 2026-11-14） | gh api |
| 语言 | TypeScript（851 个 .ts 文件） | 源码统计 |
| 许可证 | MIT（`LICENSE`，Copyright (c) 2026 Tashfeen Ahmed） | LICENSE |
| 创建时间 | 2026-04-21（仅约 7 个月） | gh api |
| 最近活跃 | 2026-11-13（push + release v0.16.28 同日） | gh api |
| 贡献者 | 127 人（contributors 分页推算） | gh api Link header |

## 2. 项目是什么，解决什么问题

每个大模型厂商都提供免费额度（每月几百万 token、每天几千请求），但单独看每个额度都很小，而且"堆叠"它们的手工成本极高：`README.md` 称这是 87 套 SDK、87 套限流规则、87 个可能失败的点。FreeLLMAPI 把这些问题收敛为一个自托管服务：你在它的管理面板里录入各家供应商的 API key，之后任何 OpenAI 兼容客户端只需指向 `http://localhost:3001/v1` 并使用一个统一的 `freellmapi-…` bearer token（`README.md`）。

请求到达后由路由器决策（`server/src/app.ts` 挂载，`server/src/services/router.ts` 实现）：挑选优先级最高、key 健康、且未触及任何限流窗口的模型，内存中解密 key 后调用上游；遇到 429/5xx 则冷却该 key 并沿 fallback 链重试下一个模型。每个响应带 `X-Routed-Via` 头标明实际服务的供应商（`README.md`）。

免费额度格局每周都在变——厂商上新模型、调低配额、改线协议。项目采用"签名模型目录"机制应对：路由器每天两次从 freellmapi.co 拉取 Ed25519 签名的目录并应用（`server/src/services/catalog-sync.ts`）；免费安装走月度快照（新模型延迟 30 天），付费订阅（$19/年）走实时目录。README 称当前目录跟踪 87 个供应商、736 个模型族、1,313 个免费端点、"约 10.1B token/月"的免费额度——**这些数字来自其托管目录的口径，是 README 声明，仓库内无法独立复核**（目录本体不在仓库中，通过签名 feed 下发）。

兼容面刻意做宽：OpenAI 全套表面（chat/completions、responses、completions、images、videos、audio、embeddings、models）+ Anthropic Messages API（供 Claude Code）+ Gemini `/v1beta`（供 Gemini CLI）+ 可选 Ollama 模拟 + `/mcp` MCP 服务器（`server/src/app.ts`）。桌面端（Electron 菜单栏应用）、CLI（`npx freellmapi`，自动配置各类 coding agent）、Docker 镜像全覆盖。

除基本的"选一个模型转发"外，还有两个建在路由器之上的增值特性：一是 **Fusion（多模型合成）**——请求虚拟 `fusion` 模型时路由器把同一提示并行扇出到一组异构免费模型，再由一个 judge 模型综合草稿成答案（`server/src/services/fusion.ts`）；二是**提示压缩（opt-in）**——请求进入路由前可先过一条 fail-open 的共享管道：去重提示、过滤工具输出、压缩重复 JSON、裁剪过期上下文（`server/src/services/compression/`，含 engines/fidelity-gate/pipeline 等模块）。另有粘性会话（同一会话 30 分钟锚定一个模型）与上下文交接（模型切换时附带压缩交接摘要）保证对话连贯（`server/src/services/context-handoff.ts`）。

项目自我定位非常明确（`README.md` Disclaimer 原文）：**"This project is for personal experimentation and learning, not production."** 本地优先、单用户设计，不支持多租户认证。

## 3. 技术栈与依赖

- **后端**（`server/package.json`）：Express 5 + TypeScript 5.8（ESM），better-sqlite3（optionalDependency，SQLite 存储），undici（上游 HTTP），zod + ajv（校验），sharp（图片处理），helmet/cors/compression/multer（Web 中间件），socks-proxy-agent（代理支持）。运行时要求 Node >=20.18 <25。
- **前端**（`client/package.json`）：React 19 + Vite 8 + TypeScript ~6.0，60 种语言的 i18n 字典（`client/src/i18n/locales/`，60 个 JSON 文件实测）。
- **桌面**（`desktop/package.json`，v0.16.28）：Electron + electron-builder（mac arm64/x64、win、linux 四个发布目标），esbuild 打包。
- **CLI**（`cli/package.json`）：npm 包名 `freellmapi`，bin 形式分发，负责给 Claude Code/Codex 等写配置。
- **测试**：vitest（server/cli/desktop 三处），Node 原生 test runner（脚本钩子）。
- **组织**：npm workspaces 单仓（`package.json`：shared/server/client/cli 四个 workspace），desktop 独立管理（有自己的 package-lock）。

依赖面非常克制：server 直接依赖仅 11 个，核心网络/存储/加密全部走成熟基础库，没有重型框架。

## 4. 目录结构与代码组织

| 目录 | 职责 | 规模（文件数） |
|---|---|---|
| `server/` | 核心：Express API、路由器、87+ 适配器、SQLite | 722 文件，约 42k 行（不含测试，wc 实测） |
| `client/` | React 管理面板（Keys/Models/Playground/Analytics 等页面） | 299 文件 |
| `cli/` | `npx freellmapi`：agent 配置器 + key 管理 | 22 文件 |
| `desktop/` | Electron 菜单栏应用（内嵌 router+dashboard） | 59 文件 |
| `shared/` | 跨端类型定义（`types.ts` 单文件） | 2 文件 |
| `docs/` | 16 个主题分区、174 个文件（含 CHANGELOG） | 174 文件 |
| `scripts/`、`docker/`、`examples/` | 引导脚本、Dockerfile、示例 | 少量 |

组织评述：清晰的 workspace 单体仓，server 内部按 `providers/`（适配器层）、`services/`（业务服务 50+ 个）、`lib/`（纯工具 60+ 个）、`routes/`（31 个路由模块）、`db/`（迁移体系）分层。`db/migrations/` 有独立迁移 runner 及往返测试。代码组织密度高，注释普遍引用 issue 编号解释"为什么这么写"，属于少见的重视长期维护性的风格（如 `server/src/lib/fallback-loop.ts` 开头整段解释三份重复代码如何合并）。

## 5. 核心架构与关键实现

### 5.1 供应商适配器统一抽象

`server/src/providers/base.ts`（566 行）定义 `BaseProvider` 基类：统一的 chat/embeddings/媒体接口 + `ProviderHttpError`（携带 status、retryAfterMs、错误归类）。一个亮点是对"退避提示"的解析深度——不只读 `Retry-After` 头，还解析 Gemini 的 protobuf `google.rpc.RetryInfo`（"17s"）和各家散文式提示（"try again in 1m26s"），并钳制在 24h 上限防恶意值（`server/src/providers/base.ts`）：

```ts
const MAX_RETRY_AFTER_MS = 24 * 60 * 60 * 1000;
export function parseRetryAfterMs(value) { /* delta-seconds 或 HTTP-date，钳制 */ }
export function parseStatedRetryMs(body) { /* 错误体深度搜索 retryDelay 字段 */ }
```

`server/src/providers/index.ts`（710 行）以 `Map<Platform, BaseProvider>` 注册 60 个适配器文件（`providers/` 目录实测），其中 `openai-compat.ts` 作为通用模板可按不同 baseUrl 重复注册；`shared/types.ts` 的 `Platform` 联合类型含 129 个平台 ID（含 search-only/decision-only/`custom`）。README 声称的"87 个供应商"与本地适配器数不完全对应——部分平台只存在于托管目录（签名 feed 下发模型行，仓库内无种子数据），这类差额是口径问题而非矛盾。

适配器按能力分三档：全功能聊天适配器（`google.ts`、`groq.ts` 等约 30 个具体实现）、专用适配器（search-only.ts、decision-only.ts、媒体类 media-* 服务）与"仅目录"平台——即 `shared/types.ts` 中注释标注 "signed catalog only" 的几十个 ID，它们没有本地模型种子，模型行完全靠托管目录下发，每个平台的免费额度形态（月度 credit、日配额、共享池）都逐条注释在 Platform 类型定义里，是了解各家免费策略的一手资料。

### 5.2 路由与打分

`server/src/services/scoring.ts`（541 行）实现基于实时统计的复合打分：reliability/speed/intelligence 三个维度按策略加权（`BANDIT_PRESETS`：balanced 0.5/0.25/0.25、smartest 0.35/0.1/0.55、fastest、reliable），并叠加高峰时段权重调整与任务类型偏置。可靠性用 Beta 后验（`reliabilityPosterior`/`sampleBeta`），即失败率高的模型自动沉底。`server/src/services/router.ts`（2,596 行）负责链式择路：逐模型检查 key 健康、冷却状态、RPM/RPD/TPM/TPD 窗口、上下文窗口下限、vision/tool 能力标记，全部淘汰后抛出带逐模型诊断的 `RouteError`，并汇总为人读得懂的耗尽摘要（`summarizeExhaustion`，如 "All models exhausted: 12 routes checked (5 rate-limited, 3 no usable key…)"）。

### 5.3 自动故障转移

`server/src/lib/fallback-loop.ts`（1,728 行）是全部三类聊天表面（OpenAI `/v1/chat/completions`、`/v1/responses`、Anthropic `/v1/messages`）**共享的唯一重试循环**——文件头注释明确记录了它的由来：原本三个表面各有一份约 150 行的副本且行为漂移（同一个 403 有的表面冷却一天、有的只 90 秒），后合并为单一事实源，各表面只留薄薄的 dispatch 适配器。关键参数：`FALLBACK_MAX_RETRIES = 20`；错误分类器（`lib/error-classify.ts`）区分可重试/限流/认证/日配额/流截断等十余类信号；除逐 key 冷却外还有**模型级 benching**——15 分钟滑动窗口内跨 key 累计 3 次失败即把整个模型按 10 分钟下架（`fallback-loop.ts` 内 `MODEL_FAILURE_*` 常量），恢复由 `cooldown-probe.ts` 后台探测自动完成。

```ts
// fallback-loop.ts：模型级 benching 参数
export const MODEL_FAILURE_WINDOW_MS = 15 * 60 * 1000;   // 滑动窗口 15 分钟
export const MODEL_FAILURE_THRESHOLD = 3;                // 窗口内失败次数
export const MODEL_FAILURE_COOLDOWN_MS = 10 * 60 * 1000; // 下架时长 10 分钟
```

冷却设计上还有一个值得注意的细节：永不缩短既有的冷却——若上游声明的重置时间（`Retry-After`）比启发式梯子更长，则以声明为准，防止启发式覆盖权威信号（`fallback-loop.ts` noteModelFailure 内注释）。

### 5.4 密钥加密与配额管理

`server/src/lib/crypto.ts`：AES-256-GCM，32 字节 hex 密钥；生产环境强制 `ENCRYPTION_KEY` 环境变量，开发环境自动生成密钥文件存放在数据库旁（chmod 0600、临时文件+原子重命名写入，且刻意**不**与密文同库存储——注释解释了旧方案"拷库即拷走钥匙"的缺陷）。提供 `rotate-encryption-key` 脚本（`server/package.json`）。`server/src/services/ratelimit.ts`（1,740 行）维护 `(platform, model, key)` 粒度的 RPM/RPD/TPM/TPD 计数器，并从上游错误**反向学习**各家真实限额（`learnLimitFromError`）与上下文窗口；并发用租约（`acquireLease`）控制。

### 5.5 /v1 兼容层与签名目录

`server/src/app.ts`（403 行）挂载：`/v1`（docs、models、OpenAI proxy、anthropic、responses、moderation）、`/v1beta`（Gemini）、`/mcp`、Ollama 模拟、URL-token 代理（`/v1/t/:token`）； helmet CSP + CORS + 双层限流。`server/src/services/catalog-sync.ts` 每天两次拉取目录，用**钉死的 Ed25519 公钥**对原始字节验签后才应用（`crypto.verify(null, bytes, catalogPublicKey(), sig)`），且不覆盖用户的自定义选择。

## 6. 构建与运行

来自根 `package.json` 与 `README.md`（未实际运行，静态核对）：

```bash
npm install && npm run dev     # server :3001 + client :5173，双 HMR
npm test                       # bootstrap 钩子 + server + cli + client 全套
npm run build                  # server/cli tsc + client vite
npm run lint                   # eslint（server + cli）
npm run test:migrations        # 数据库迁移往返测试
npm run desktop:dist:win       # 桌面端打包（另有 mac:x64/linux）
docker compose up              # 或 curl -fsSL https://freellmapi.co/install.sh | bash
```

CLI 用法（`cli/README.md`）：`npx freellmapi setup-claude --url http://localhost:3001 --api-key <key>`，另有 `keys add|list|remove|test` 管理 provider key。

## 7. 测试与工程质量

- **规模**：server 侧 412 个 `.test.ts` 文件、约 13,246 行测试代码（wc 实测），覆盖 `db/`、`providers/`、`routes/`、`services/`、`lib/`、`integration/` 六大区；vitest `--pool=forks --fileParallelism=false` 串行跑（避免 SQLite 竞争）。
- **特色**：迁移往返测试强制要求（CI 单独一步）；i18n 每语言 key/占位符一致性测试（README 提及）；`.gitattributes` 全仓 LF 归一化的理由写得很具体——`app.ts` 校验 client/index.html 内联脚本的 SHA-256 并写进 CSP，CRLF 检出会直接导致浏览器拦截脚本（`.gitattributes` 注释）。
- **CI**（`.github/workflows/ci.yml`）：Node 20/22/24 三矩阵，action 全部钉 commit SHA，含迁移检查、测试、lint、构建；另有 `docker.yml`、`cli-release.yml`、`desktop-release.yml`。
- **观察**：测试密度约 0.31（13k 测试行 / 42k 源码行），对个人项目而言非常高；client 的 lint 债务被明确记录为待办（CI 注释"client has its own backlog of ESLint violations"）。

## 8. 社区与维护状态

- **活跃度**：2026-04-21 建仓，7 个月冲到 51.7k stars（平均月增约 7k），最近 push 与最新 release（v0.16.28）都在分析前一天——patch 版本号已滚到 0.16.28，发布节奏极快（`gh api releases/latest`）。server 版本号 0.2.1、cli 0.8.2、desktop 0.16.28 各自独立演进（各 workspace package.json）。
- **贡献者**：127 人（gh api 分页推算），README 致谢区列出 130+ 头像，多为一次性小贡献（文案/i18n/小修复的典型分布，"未确认"具体 commit 分布）。CONTRIBUTING.md 明确欢迎 AI 辅助贡献，但要求与普通 PR 同等质量门槛（README Contributing 一节）。
- **issue**：48 个开放（gh api），README/代码注释大量引用 issue 编号（#423、#657、#682、#1516、#1649 等），问题响应痕迹清晰。`.claude/hooks/` 里甚至有检查 CONTRIBUTING 阅读状态的 PR 钩子及其测试。
- **商业模式**：README 明确 Premium（$19/年或 $49 终身）只卖"实时目录 feed"，路由器本体永久 MIT 免费；这解释了高频维护的资金来源，也意味着项目核心数据（模型目录）部分闭源托管在 freellmapi.co。多语言 README（英/土/中/俄/乌兹）与 60 语言 UI 显示出明确的全球增长运营取向。

## 9. 亮点与风险

**亮点**

1. 工程深度罕见地扎实：共享故障转移循环、模型级 benching、从错误反向学习限额、退避提示多格式解析，每一处都有 issue 编号和设计理由注释（`server/src/lib/fallback-loop.ts`、`server/src/providers/base.ts`）。
2. 兼容面完整且真实落地：OpenAI/Anthropic/Gemini/Ollama/MCP 五套线协议挂同一个路由器（`server/src/app.ts`），对 coding agent 生态一键配置（`cli/`）。
3. 密钥安全设计认真：AES-256-GCM、密钥与密文分存、原子写入、权限收紧、轮换脚本（`server/src/lib/crypto.ts`）。
4. 文档诚实：`docs/en/architecture/00-high-level-index.md` 专设 Limitations 与逐供应商 ToS 审查表，不回避弱点。
5. 测试与 CI 严格（412 个测试文件、三版本矩阵、迁移往返）。

**风险 / 局限**

1. **ToS 合规风险（核心）**：聚合他人免费额度本质是在灰色地带。项目自己的审查表（`docs/en/architecture/00-high-level-index.md`，2026-05 版）承认：Cohere 条款禁止个人用途（标 ❌ Avoid）、NVIDIA NIM 限"evaluation only"、Google Gemini 新条款范围收窄（⚠️）；README Disclaimer 明确仅供个人实验、禁止生产使用，且"你与各上游供应商的条款关系不因本项目而改变"。审视角度应保持客观：代码实现是中性的，但任何部署者需自行承担条款风险。
2. **核心数据闭源依赖**：模型目录由 freellmapi.co 签名 feed 下发，免费用户落后实时 feed 30 天（README 自称落后约 303 个模型）；上游服务一旦停运或改策，安装实例的目录更新即断供。10.1B tokens/月、1,313 端点等数字均为托管目录口径的 README 声明，仓库内不可复核。
3. **能力天花板明确**：无 SLA、无前沿模型的持续访问、一天内有效智能随头部模型耗尽配额而下滑、UTC 零点重置（`docs/en/architecture/00-high-level-index.md` Limitations 一节原文承认）。
4. **单点维护**：项目仅 7 个月历史、由作者 tashfeenahmed 主导并绑定其商业服务，长期可持续性与中立性未经验证（"未确认"）。

## 10. 总结评价

FreeLLMAPI 是 2026 年最典型的"AI 免费额度聚合器"类项目，但工程实现远超同类：统一适配器抽象 + 统计驱动的多维打分路由 + 单一事实源的故障转移循环 + 真实学习的逐 key 配额账本，加上桌面端/CLI/Docker 全形态分发和 412 个测试文件，把它从"脚本玩具"拉开了一个档次；签名目录 + 免费延迟 30 天的付费模式也给出了罕见清晰的个人开源变现路径。它的价值不在于"免费无限 GPT"（作者自己反复否认这一定位），而在于展示了如何把 87 个不稳定、限额各异的 API 收敛成一个可靠的工程问题来解：错误分类、冷却梯子、模型 benching、目录验签这套组合拳对任何做多上游 LLM 网关（哪怕用付费 API）的开发者都有直接参考价值。适合阅读源码的人群：构建 LLM 网关/路由层的工程师、研究多供应商 failover 模式的人；适合使用的人群：仅限接受其"个人实验"定位、理解 ToS 风险并愿意自行核对各家条款的个人开发者。
