# KeygraphHQ/shannon 项目分析

> 星数 49247（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-04

## 1. 项目速览

Shannon 是 Keygraph 公司开源的**自主 AI 渗透测试 agent**：分析 Web 应用源码找出攻击路径，再用浏览器自动化与命令行工具对运行中的应用执行**真实漏洞利用**，只用带可复现 PoC 的发现出报告——"No exploit, no report"。

| 项目 | 内容 |
| --- | --- |
| 名称 | KeygraphHQ/shannon（npm 包 `@keygraph/shannon`） |
| 星数 | 49247（快照）；gh api 实测 51070（2026-11-04），仍在快速上涨 |
| 语言 | TypeScript（apps 下约 53,041 行 .ts，实测统计） |
| 许可证 | AGPL-3.0（`LICENSE`；`apps/cli/package.json` 标 `AGPL-3.0-only`） |
| 创建时间 | 2025-09-27（gh api `created_at`） |
| 最近 push | 2026-10-27（gh api `pushed_at`），最新 release v3.5.0（2026-10-27） |
| 贡献者 | 7 人（gh api contributors），不接受外部代码贡献（README"Community and Support"节） |
| 定位 | 白盒（源码辅助）AI 渗透测试，面向开发/CI/CD 场景，商业版 Pro/Enterprise 的引擎 |

## 2. 项目是什么，解决什么问题

团队用 Claude Code、Cursor 高频发版，但渗透测试往往一年一次，中间 364 天可能在向生产环境持续引入漏洞（README"Why Shannon Exists"节）。Shannon 把渗透测试左移进 SDLC：按发布节奏对 staging 环境做按需的、利用实证驱动的自动化渗透测试，把专家人力留给需要组织级判断的高风险系统。

工作方式是**白盒双通道**（README"Architecture"节 mermaid 图）：一路对运行中的应用做侦察与漏洞分析，一路用 agentic SAST 分析源码（架构、信任边界、接口、数据流），两路候选经 reconciliation 合并去重后进入利用队列；利用 agent 对运行中的应用实施真实攻击，证明不了利用的假设直接丢弃。只有被活体验证的漏洞才计入报告与 CI/CD 严重度门禁。

覆盖面刻意收窄（`docs/coverage-roadmap.md`、`COVERAGE.md`）：Broken Authentication/Authorization、SQL/命令注入、XSS、SSRF 共 5 大类加一个 miscellaneous 兜底通道，对应 OWASP WSTG checklist 逐项标注。依赖漏洞、弱加密、不安全配置等纯静态发现明确不在开源版范围内——这是商业版 Pro/Enterprise（SCA、secrets、fix PR、Jira）的边界（README"Editions"表）。

命名取自信息论之父 Claude Shannon：渗透测试本质是信息问题，每次探测都在降低系统状态的不确定性（README"Why Shannon?"节）。仓库致谢 Pi（`@earendil-works/pi-*` 依赖）、Playwright CLI 与 Google Mantis（README"Acknowledgements"，`THIRD_PARTY_NOTICES.md`）。

注意它不是被动扫描器：利用 agent 会建用户、提交表单、变更应用状态（README"Safety, Scope, and Limitations"节），只能对自有或书面授权的环境运行，单次全量扫描约 1-1.5 小时并产生 LLM API 费用。

## 3. 技术栈与依赖

- **语言/运行时**：TypeScript 5.9 + Node.js 18+（`package.json`、`apps/cli/package.json` engines），ESM（`"type": "module"`）
- **Monorepo**：pnpm 10.33 workspace（`pnpm-workspace.yaml`，仅 `apps/*`）+ Turborepo 2.5（`turbo.json`，build/check/clean 三任务）
- **持久化编排**：Temporalio SDK 1.15.0（workflow/activity/client，`apps/worker/package.json`）+ Temporal Server 1.7.0 容器（`docker-compose.yml`，start-dev 模式 SQLite 落盘）
- **AI agent 框架**：Pi 全家桶 `@earendil-works/pi-agent-core / pi-ai / pi-coding-agent` 0.84.4 + `@gotgenes/pi-permission-system`（`apps/worker/src/ai/pi/pi-executor.ts`），agent 内置 read/bash/edit/write/grep/find/ls 工具，浏览器类 agent 走 Playwright（`utils/browser-agents.ts`）
- **CLI**：tsdown 打包、@clack/prompts 交互式向导、chokidar 监听、smol-toml 读 `~/.shannon/config.toml`（`apps/cli/package.json`）
- **校验/模板**：ajv + ajv-formats、typebox（schema）、handlebars（`apps/worker/package.json`）；报告渲染含 Typst（Dockerfile 安装 xz 用于解压 Typst tarball）
- **lint**：Biome 2.x（根 `biome.json`、`biome` script）
- **容器**：Chainguard Wolfi 基底两阶段 Dockerfile（`Dockerfile`，最小攻击面 + 供应链考量，含 `shannon.worker-protocol` label 防容器信任伪造）

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `apps/cli/` | npm 发布的 `@keygraph/shannon` CLI：setup/start/status/logs/stop/scans/reset/build 命令（`src/commands/`）、Docker 编排（`docker.ts`）、Temporal 客户端、扫描进度渲染（`scan/`）、实验性 php2shell 探针（`exp-php2shell/`） |
| `apps/worker/` | 渗透引擎本体：Temporal 工作流（`src/temporal/`）、agent 执行与渲染服务（`src/services/`）、Pi 执行层（`src/ai/pi/`）、reconciliation（`src/ai/reconciliation/`）、agentic SAST"Capella"（`src/ai/sast/capella/`）、发现收集器（`src/collectors/`）、审计（`src/audit/`）、prompt 模板（`prompts/`） |
| `apps/worker/prompts/` | 34 个 agent prompt 模板（`vuln-*`/`exploit-*`/`recon`/`report-executive` 等，17-27KB/个）+ `shared/`、`sast/` partials |
| `docs/` | 11 篇运维文档：development/configuration/ai-providers/platforms/workspaces/safety/coverage-roadmap 等 |
| `benchmark/` | Photoview 2.4.0 实测报告（PDF + SARIF，DeepSeek/Grok/Claude 三模型） |
| `sample-reports/` | CRAPI、Juice Shop、Capital 三份示例报告 |
| `llms.txt` / `llms-full.txt` | 面向 AI agent 的仓库入口（README TIP 节） |
| `docker-compose.yml` / `Dockerfile` / `entrypoint.sh` | Temporal server + worker 容器编排 |
| `.github/workflows/` | release.yml / release-beta.yml / rollback.yml / rollback-beta.yml（全部 workflow_dispatch 手动触发） |
| `workspaces/`、`repos/` | 运行时生成（tarball 中为空目录） |

组织方式：**紧凑双包 monorepo**。CLI 是薄壳（48 个文件，负责交互与容器生命周期），worker 是重引擎（约 170+ 文件）。两者通过 Temporal（gRPC 7233）解耦，CLI 拉起/监控容器内 worker，引擎代码不进 npm 包（CLI `files` 仅 dist + infra）。

## 5. 核心架构与关键实现

### 5.1 Temporal 持久化渗透流水线

核心是 `apps/worker/src/temporal/workflows.ts`（65KB，约 1500 行）的 `pentestPipeline`：全流水线跑在 Temporal workflow VM 里，天然获得可恢复（resume 从 workspace 跳过已完成 agent，`shouldSkip`）、部分失败降级（`addPartialReason` 非致命失败有界记录，`MAX_NON_FATAL_FAILURES` 封顶防 workflow 历史膨胀）、进度查询（`setHandler(getProgress)`）等能力。Capella agentic SAST 作为子工作流（`executeChild`，`CAPELLA_CHILD_WORKFLOW_OPTIONS`）通过 signal 上报阶段进度，且跨边界 payload 全部重新校验（`recordCapellaStage`）。持久化状态与投影分离的注释写得非常认真：

```ts
// The durable degradation record. Codes plus bounded context are the identity; the
// projection into state carries derived safe messages for every consumer surface.
let partialReasons: readonly PartialReason[] = [];
```

入口分两个：`pentestPipeline`（测试模式）与 `pentestPipelineWorkflow`（OSS 正式入口），workflow 负责重启决策而 agent 进程不持有（文件头注释，`workflows.ts:12-14`）。

### 5.2 Agent 注册表与执行生命周期

`apps/worker/src/session-manager.ts` 用冻结常量 `AGENTS` 声明全部 agent（name → promptTemplate + deliverableFilename）：pre-recon-code → recon → vuln-{injection,xss,auth,ssrf,authz} 5 个分析 agent → exploit-{injection,xss,auth,ssrf,authz,miscellaneous} 6 个利用 agent → report-executive。`apps/worker/src/services/agent-execution.ts` 实现完整生命周期：加载配置 → 渲染 prompt → 建 git checkpoint → 开审计日志 → `runPiPrompt`（Pi harness）→ 按校验器验证 deliverable → 成功 commit / 失败回滚（`git-manager.ts` 的 `rollbackGitWorkspace`）。失败处理区分可重试与不可重试错误码（`types/errors.ts` 的 `ErrorCode`）。

### 5.3 Prompt 资产与利用哲学

prompt 不是散落在代码里的字符串，而是 `apps/worker/prompts/*.txt` 模板资产（本地模式可 live-edit，`CLAUDE.md` 双 CLI 表），支持 `@include(shared/...)` partials 与 `{{WEB_URL}}` 插值。以 `exploit-injection.txt` 为例：角色设定为"Injection Exploitation Specialist"，强制"proof by construction"（SQLi 必须给出实际提取的数据、Command Injection 必须证明命令执行）、默认怀疑（"Assume NOT vulnerable until proven otherwise"）、以及不可协商的法律/范围声明（仅限受控测试环境）。利用 agent 的产出经 `collectors/`（exploit/finding/recon/vuln/pre-recon 五类 collector + ajv schema）结构化回收，reconciliation 层（`src/ai/reconciliation/`，27 文件）负责合并、去重、队列校验（`services/queue-validation.ts`）与物化发布。

### 5.4 Capella：agentic SAST 子系统

`apps/worker/src/ai/sast/capella/`（30 文件）是源码分析引擎，按 architecture → research → verdicts → export 四阶段推进（`stages/` 目录），产出带 schema 的 SAST 候选（`schemas.ts`、`finding-types.ts`）并导出 SARIF（`sarif-exporter.ts`、`ai/sast/sarif-profile.ts`），enrichment prompts（`prompts/sast-enrichment-*.txt`）把候选按漏洞类别富化后交给 reconciliation。

## 6. 构建与运行

来自 README、`CLAUDE.md` 与 `docs/development.md`：

```bash
# npx 模式（零安装，拉 Docker Hub keygraph/shannon:latest 镜像）
npx @keygraph/shannon@latest setup          # 交互式配置凭据（写 ~/.shannon/config.toml）
npx @keygraph/shannon@latest start -u https://your-app.com -r /path/to/repo

# 源码构建模式（SHANNON_LOCAL=1，本地构建 shannon-worker 镜像）
git clone https://github.com/KeygraphHQ/shannon.git && cd shannon
cp .env.example .env                        # 至少一个 provider key，如 ANTHROPIC_API_KEY
pnpm install && pnpm build                  # turbo run build（cli: tsdown，worker: tsc）
./shannon start -u https://your-app.com -r /path/to/repo

# 工程命令
pnpm check    # turbo run check（各包 tsc --noEmit）
pnpm biome    # biome check .
```

前置依赖：Docker + Docker Compose v2、Node 18+、AI provider 凭据（BYOK，Anthropic/OpenAI/xAI/AWS Bedrock 或任意 Pi 目录 provider/本地 Ollama/vLLM/LM Studio，`docs/ai-providers.md`）。首次扫描前需完成 provider 的网络安全保障验证（README"Prerequisites"）。

## 7. 测试与工程质量

- **单测极少**：全仓库仅 4 个 `node:test` 风格测试文件，全部位于 `apps/cli/src/exp-php2shell/`（classify/multipart/probe/transport，实测 `find -name "*.test.ts"`）；无 vitest/jest 配置，package.json 无 test script。引擎核心（temporal/ai/services）无公开单测，推测依赖内部 `pipelineTestingMode`（`workflows.ts` 输入项、`prompts/pipeline-testing/`）与真实靶场验证（benchmark/ Photoview 报告）
- **类型与 lint 严格**：两包均有 `tsc --noEmit` check，根级 Biome；worker 大量使用 Result 类型（`types/result.ts`）、错误码枚举与"安全字段"投影（`audit/safe-fields.ts`，防止敏感信息泄入日志/UI）
- **CI 只管发布**：`.github/workflows/` 四个 workflow 全是手动触发的 release/beta/rollback（SHA-pinned actions、`--frozen-lockfile`、`.releaserc.json` 语义化版本）；没有 PR CI——与"不接受外部贡献"一致
- **供应链意识强**：Wolfi 最小基底、CI actions 按 commit SHA 固定（`release.yml`）、`pnpm.onlyBuiltDependencies` 白名单只允许 @swc/core 与 protobufjs 跑构建脚本（根 `package.json`）
- 代码注释质量高：大量解释"为什么"的持久化边界、状态所有权注释（如 `workflows.ts` 对 ApplicationFailure 终态语义的警告）

## 8. 社区与维护状态

- **活跃度**：2025-09-27 创建，约一年到 51k 星（gh api），README 挂 Trendshift 徽章；最近 push 与最新 release v3.5.0 均为 2026-10-27，当前高频迭代中
- **贡献者结构**：7 人（gh api contributors，首 100 页全量），README 明示"At this time, Keygraph is not accepting external code contributions"——典型商业公司开源引擎模式，issues/discussions 只收 bug 与功能请求
- **Release 节奏**：语义化发布 + beta 通道 + 回滚流水线（`.github/workflows/release*.yml`、`rollback*.yml`），GitHub Actions/GitLab CI 官方集成独立成仓（`KeygraphHQ/shannon-action`，README 链接）
- **社区**：Discord + 每周社区 office hours（README"Community and Support"），v3.x 从 Claude Agent SDK 迁移到自研 Pi harness（README 订阅提示：1.9.0 是 Claude Agent SDK 最终版，未确认具体迁移日期）

## 9. 亮点与风险

**亮点**

1. **"No exploit, no report"的实证哲学**：只报活体验证的漏洞，从产品逻辑上消解扫描器噪音，CI 门禁只统计 `status: exploited`（README CI/CD 节）
2. **生产级编排工程**：Temporal 持久化让 1-1.5 小时的多 agent 扫描可恢复、可降级、可审计（`workflows.ts`、git checkpoint/回滚 `git-manager.ts`），这在 AI agent 项目里是少见的严肃工程
3. **Prompt 即资产管理**：34 个版本化 prompt 模板 + partials 体系（`apps/worker/prompts/`），本地模式可热编辑调试
4. **隐私与供应链设计**：BYOK 直连 provider、repo 只读挂载、结果全本地（README FAQ），Wolfi 镜像 + SHA-pinned CI + 构建脚本白名单
5. **AI 原生文档**：llms.txt/llms-full.txt 主动服务 LLM 读者，benchmark 与 sample-reports 公开可查证

**风险/局限**

1. **AGPL-3.0 传染性**：商业化集成、托管服务改造受 AGPL 强约束，商用授权需联系 Keygraph（LICENSE 节）
2. **单公司单点**：7 人贡献、不接受外部 PR，项目走向完全由 Keygraph 商业策略决定；开源版功能边界与付费版刻意区隔（SCA/secrets/fix PR 归 Pro，README Editions 表）
3. **覆盖面窄**：5 大类漏洞 + WSTG checklist 大量空白项（`COVERAGE.md`），不能替代全面 SAST/SCA
4. **运行成本与破坏性**：单次 1-1.5 小时 + 自付 LLM 费用；mutative 操作可能污染目标环境（README Safety）；对不遵循工具调用约束的模型效果明显衰减（README FAQ）

## 10. 总结评价

Shannon 是目前 AI 渗透测试方向工程完成度最高的开源项目之一：把"多 agent 自主渗透"从 demo 做成了带持久化编排、git 检查点、结构化审计与 SARIF 输出的可进 CI/CD 的产品，并用"证明不了就丢弃"的利用哲学建立了差异化。它不是传统扫描器的替代，而是把白盒渗透测试产品化的先锋；代价是 AGPL、闭门开发与刻意收窄的开源覆盖面。适合有授权测试需求的开发团队在 staging 环境按发布节奏跑渗透门禁；对想研究多 agent 编排、agent 生命周期管理、prompt 工程体系的人来说，`apps/worker/src/temporal/workflows.ts`、`services/agent-execution.ts` 与 `prompts/` 目录是极具学习价值的一手材料。
