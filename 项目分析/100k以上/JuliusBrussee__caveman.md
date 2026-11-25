# JuliusBrussee/caveman 项目分析

> 星数 114558（快照 2026-10-24）｜主语言 Go｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：**面向 AI 编码 agent 的"省 token"工具集**——用一个"穴居人"口吻技能让模型回答更短（Say less），再用本地可恢复压缩引擎把日志/JSON/表格/工具清单等喂给模型前先瘦身（Read less），外加模型路由、可复用脚本块、网页压缩等六件套，全部本地运行、Apache-2.0 开源。

| 事实 | 值 | 出处 |
|---|---|---|
| 名称 | caveman（Caveman Engine / @caveman-ai/cli） | `README.md`、`packages/cli/package.json` |
| 星数 | 114,558（快照）；分析时 gh api 显示 116,090 | gh api repos/JuliusBrussee/caveman |
| 主语言 | Go（运行时）+ TypeScript（CLI/安装器）+ Python（SDK/评测） | `go.mod`、`packages/cli/package.json` |
| 许可证 | Apache-2.0（3.0.0 起全仓库统一；云服务闭源） | `LICENSE`、`LICENSING.md` |
| 最新 release | v4.0.1（2026-10-13） | gh api releases/latest |
| 最近活跃 | push 于 2026-10-29（分析前一天），创建于 2026-04-04 | gh api |
| 贡献者 | 约 104 人（contributors 分页 last=104）；文档自述"一人免费维护" | gh api；`README.md` Privacy 节 |

## 2. 项目是什么，解决什么问题

AI 编码 agent（Claude Code、Codex、Gemini CLI 等）每个会话都在为两类东西付钱：模型输出的冗长措辞，以及反复回读的日志、JSON、CSV、测试输出等工具结果。caveman 从两端同时下手（`README.md`）：

- **Say less（口吻技能）**：`skills/caveman/SKILL.md` 是一份约 1,160 token 的行为规范，要求"答案先行、一句一个事实、代码/命令/路径逐字保留、绝不丢 not/never/no/only"。它自称"a voice, not broken grammar"——基于航空维修手册的受控英语 ASD-STE100，而不是真的破坏语法。README 实测同一修复建议从 63 token 压到 20。
- **Read less（压缩引擎）**：`engine/`（Go）在字节到达模型前检测内容类型（日志/JSON/表格/代码清单/diff/终端输出等），路由到匹配压缩器，把数万 token 的噪声压成几百；原文先落盘（CCR 存储），模型或 agent 随时可用 `caveman_retrieve` 拿回原文（`engine/README.md`、`engine/engine.go`）。
- **Auto（模型路由）**：按每次提问的难度自动选模型和推理档位，宣称比"Opus 5.5 全程 xhigh"省 54% 且分数持平（`README.md`、`docs/AUTO-BENCHMARK.md`）。
- **Blocks（代码模式脚本块）**：agent 反复手写同一段脚本（README 引用 Purdue 论文：占任务成本最高 9.57%），Blocks 把这些脚本固化为仓库内可复用、PR 可校验的块（`README.md`；主仓库在 caveman-ai/blocks）。

项目的独特之处在于**诚实性工程**：每个数字都标注 `inferred`（本地 o200k 估算）或 `verified`；压缩失败、压不更小、无恢复存储时一律 fail-closed 透传原始字节，"claim nothing"（`engine/engine.go` 注释、`proxy/README.md`）。`docs/HONEST-NUMBERS.md` 专门列出"caveman 何时会输"，包括 HTML 无压缩器导致会话 token 反升 9.9% 的负面数据。外部背书：Adobe Research CAVEWOMAN 论文（arXiv 2606.24083）引用、JetBrains 86 任务 A/B（质量无显著损失 p=0.82）、Elasticsearch Labs 复刻（响应 token 省 63.6%）。

该项目 2026 年 4 月以玩笑起步（README："Started as a joke in April 2026"），7 月登顶 Hacker News（904 分）与 GitHub Trending #1，半年破 10 万星。

## 3. 技术栈与依赖

| 层 | 技术 | 出处 |
|---|---|---|
| 核心引擎/代理/记忆 | Go 1.26，647 个 .go 文件（非测试约 9.37 万行） | `go.mod`；全仓统计 |
| JSON 处理 | go-json-experiment/json（实验性 encoding/json/v2） | `go.mod` |
| 内容检测 | smacker/go-tree-sitter（代码压缩，cgo 构建时）；纯 Go go/ast 为无 cgo 回退 | `go.mod`、`engine/compressors/compressor.go` |
| token 计数 | tiktoken-go/tokenizer（离线 o200k_base 估算） | `go.mod`、`engine/README.md` |
| 浏览器抓取 | chromedp/cdproto（CDP 协议驱动浏览器） | `go.mod`、`browse/cdp.go` |
| 本地存储 | modernc.org/sqlite（无 CGO 的 SQLite）、pgx/v5（云侧 Postgres） | `go.mod` |
| CLI/安装器 | TypeScript + Node（CLI 要求 ≥22.13，安装器 ≥18），esbuild 打包，node:test 测试 | `packages/cli/package.json` |
| SDK/中间件 | TypeScript + Python 双实现（Vercel AI SDK、LangChain、OpenAI、Anthropic 适配） | `packages/middleware/` |
| 云后端 | Supabase（functions + migrations） | `supabase/` |
| 部署 | Docker、docker-compose、Kubernetes（含 HA）、AWS ECS | `Dockerfile`、`deploy/` |
| 压缩 | klauspost/compress（zstd 请求体解码重编码） | `go.mod`、`proxy/README.md` |

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `engine/` | 压缩核心：detect→路由→压缩→CCR 恢复，16 个压缩器 + S0-S4 安全分级 + WASM 构建（`cmd/caveman-wasm`） |
| `proxy/` | `caveman-proxy` 本地反向代理（base-URL 换入即用）：gateway、多 provider 适配（anthropic/openai/bedrock/gemini）、缓存保温、SQLite 账本 |
| `mem/` | cavemem：跨会话持久记忆，SQLite + BM25 召回，召回内容再过引擎压缩 |
| `mcp/` | MCP 服务器，向 agent 暴露 `caveman_retrieve`/`caveman_stats` 等工具 |
| `browse/` | 基于 CDP 的网页抓取与压缩（"129.8× smaller page"） |
| `shrink/` | MCP/OpenAI 工具清单压缩器（预编译二进制 + 签名校验 launcher） |
| `packages/` | TS/Py 发布包：`cli`（`caveman` 命令）、`middleware`、`sdk`（含 parity 测试）、`device-auth` |
| `skills/` | 20+ agent 技能：caveman/ultracave/megacave 口吻、commit/review/compress/stats、工作模式（lean-build、surgical-patch 等）、cavecrew 子代理 |
| `src/` | 安装侧钩子：会话启动 hook、状态栏脚本（bash/PowerShell）、模式跟踪 |
| `agents/` | agent 适配注册表（agents.json、profiles） |
| `installer/` | `npx @caveman-ai/cli` 一键安装器，探测 30+ 种 agent |
| `extension/` | Firefox 浏览器扩展 |
| `evals/`、`benchmarks/`、`docs/` | 评测 harness、基准脚本、HONEST-NUMBERS 等技术文档 |
| `supabase/`、`deploy/` | 云端函数/迁移；Compose/K8s/ECS 部署清单 |

组织方式评述：**polyglot monorepo**——Go 部分按子模块各自带 `go.mod` 外还共享根 `go.mod`（单一 module `github.com/JuliusBrussee/caveman`），Node 侧用 pnpm workspace（`pnpm-workspace.yaml`）。每个主要目录自带 `README.md`/`CLAUDE.md`/`AGENTS.md`/`LICENSE`，是典型的"AI agent 可导航"仓库风格。README 声称 4.0 起为"six rocks, one install"（六个功能件一次装齐）。

## 5. 核心架构与关键实现

### 5.1 引擎的 fail-closed 压缩管线（engine/engine.go）

四调用面 `Compress/Retrieve/Detect/Stats` 是代理、SDK、CLI、MCP、WASM 共享的稳定契约。压缩前先做五重防线：无匹配压缩器、解析失败、结果不更小、未知安全等级、有损但无恢复存储——任何一条不满足都原样透传并"claim nothing"：

```go
func (e *Engine) Compress(input []byte, opts Options) (Result, error) {
    // ...detect...
    // A lossy (S4) compressor may only run if we can store the original for
    // recovery. Without a store, fail closed to pass-through.
    info, known := safety.Lookup(comp.SafetyClass())
    if !known { return res, nil }              // unknown safety class → fail closed
    if info.RequiresCCR && e.store == nil && !opts.ExternalRecovery { return res, nil }
    out, meta, ok := compressWith(comp, body, opts.Query)
    if !ok || out == nil { return res, nil }   // parse problem → pass-through
    after := e.counter.Count(out)
    if after >= before || bytes.Equal(out, input) { return res, nil } // not smaller → claim nothing
```

（出处：`engine/engine.go` 第 72-116 行）

### 5.2 S0-S4 诚实性安全阶梯（engine/safety/safety.go）

每个压缩器声明固有的安全等级，等级决定"是否改变模型可见字节"与"是否必须先存原文"两个契约：

```go
// S0 — byte-safe behavior (metadata, accounting). No model-visible bytes change.
// S1 — provider-native hints (cache, routing). No model-visible bytes change.
// S2 — structural changes that need SDK cooperation.
// S3 — behavioral changes (routing, reasoning); eval-gated in Cloud.
// S4 — lossy structural compression. Alters model-visible bytes, so it is
//      opt-in, must be reversible (CCR), and discloses what it dropped.
```

（出处：`engine/safety/safety.go`；S4 默认 `RequiresCCR: true`，TOON 等无损变换可覆写 `Reversible`）

### 5.3 日志压缩器（engine/compressors/log.go）

默认注册 16 个压缩器（`engine/compressors/compressor.go` 的 `Default()`：JSON、日志、测试报告、代码、diff、搜索结果、文本、HTML、表格、config、工具 schema×2、TOON、可达性树、重复、终端输出）。日志压缩器保留 ERROR/FATAL/栈帧与首尾 2 行，把 INFO 噪声折叠成带结构摘要的标记行，且标记自身保持幂等：

```go
var importantLineRe = regexp.MustCompile(`(?i)(\b(ERROR|FATAL|PANIC|EXCEPTION|
    TRACEBACK|FAIL|WARN|CRIT|ALERT|EMERG)\b|^\s+at\s|...|caused by)`)
// A run of dropped lines is collapsed into this marker, which is itself
// kept on later passes so the compressor stays idempotent.
var logMarkerRe = regexp.MustCompile(`lines elided \(caveman\)`)
```

（出处：`engine/compressors/log.go` 第 12-19、44-47 行）

### 5.4 本地反向代理（proxy/）

`proxy/internal/gateway/`（75 个 Go 文件）实现"换 base URL 即接入"：agent 指向 `127.0.0.1:8787`，流量按 provider 转发（`proxy/providers/`：anthropic/openai/bedrock/gemini），压缩在请求路径上完成，原文经 CCR 落地 SQLite（`proxy/internal/store/`，95 个文件）。特色实现：`cache_warm.go` 系列在你暂停时维持 Anthropic prompt cache 不过期（仅当本地历史显示你会及时回来）；对 Codex 的 zstd 请求体解码-压缩-重编码，失败即转发原始字节并记 `skip_reason`（`proxy/README.md`）。

## 6. 构建与运行

```bash
# 安装（用户入口，README.md）
npx @caveman-ai/cli@latest          # 需要 Node.js 22.13+，Mac/Linux/Windows

# 只装口吻技能
npx skills add JuliusBrussee/caveman -g

# Go 侧构建与测试（proxy/README.md、go.mod）
go build ./proxy/...
ANTHROPIC_API_KEY=… caveman-proxy   # 127.0.0.1:8787
go test ./...                       # 361 个 _test.go

# CLI 构建/测试（packages/cli/package.json）
pnpm install && pnpm --filter @caveman-ai/cli build
node --test tests/*.runtime.mjs

# 常用命令（README.md）
caveman claude    # 以 caveman 启动 claude/codex/gemini/opencode 等
caveman status    # 查看六件套开关
caveman trial -- claude       # A/B 实测一次会话
caveman shrink -- pnpm test   # 压缩命令输出
```

另有 Docker 镜像 `ghcr.io/juliusbrussee/caveman-proxy:bin-v1.1.7`（`proxy/README.md`）与 Compose/K8s/ECS 清单（`deploy/`）。

## 7. 测试与工程质量

- **Go 测试**：361 个 `_test.go` 文件、约 91,240 行测试代码，与约 93,700 行实现接近 1:1（全仓统计）。覆盖单元（每个压缩器都有专测，如 `engine/compressors/log_test.go`、`html_test.go`）、并发（`mem/store_concurrency_test.go`）、集成（`mcp/retrieve_integration_test.go`、`browse/cdp_integration_test.go`）与"发布形态"测试（`engine/release_shape_test.go`）。
- **JS/Python 测试**：136 个测试文件，CLI 用 node:test（`packages/cli/tests/*.runtime.mjs`），Python 侧有压缩安全/并发/保真测试（`tests/test_compress_safety.py`、`tests/test_eval_fidelity.py` 等）。
- **CI**：16 个工作流（`.github/workflows/`）：ci.yml 跑 ubuntu/windows/macos 三平台矩阵 + Go（版本取自 go.mod）+ Node 18/22；engine-ci、middleware 三件套（canary/conformance/python）、agent-conformance、profiles、provider-catalog，另有 CodeQL、OpenSSF Scorecard、supply-chain、release-binaries/packages/smoke、dependabot。
- **质量观察**：代码注释密度高且自述性强（包级 doc 注释解释设计动机）；每个模块带 `CLAUDE.md`/`AGENTS.md` 供 agent 导航；`docs/HONEST-NUMBERS.md` 公开负面结果，工程诚信文化少见地好。

## 8. 社区与维护状态

- **活跃度**：创建于 2026-04-04，最近 push 2026-10-29（分析前一天），持续高频维护（gh api）。
- **release 节奏**：v4.0.1（2026-10-13）为最新 release；4.0 引入"six rocks"架构（README、`packages/cli/package.json` version 4.0.1）。Blocks 为独立仓库 caveman-ai/blocks，pre-release v0.1.0-rc.5。
- **贡献者结构**：约 104 名贡献者（gh api 分页 last=104）；README 自述"核心为一人免费维护"（One person maintains this for free），telemetry 默认开启并明示关闭方式（`caveman telemetry off`）。
- **生态影响**：被 Adobe Research（CAVEWOMAN 论文）引用，JetBrains 与 Elasticsearch Labs 分别 A/B 复刻；HN 904 分 #1、GitHub Trending #1、ThePrimeagen 评测视频（`README.md` "In the wild" 节）。
- 许可史值得一提：3.0.0 前为 MIT + BUSL-1.1 混合许可，2026 年全量转为 Apache-2.0（`LICENSING.md`）。

## 9. 亮点与风险

**亮点**

1. **诚实性作为一等设计**：S0-S4 安全阶梯 + fail-closed 透传 + 全部数字标注 inferred/verified，负面数据公开（`engine/safety/safety.go`、`engine/engine.go`、`docs/HONEST-NUMBERS.md`）。
2. **可恢复的有损压缩**：原文落盘 CCR，模型侧省 token 而 agent 侧零信息丢失，这是与朴素"截断"方案的本质区别（`engine/README.md`）。
3. **零侵入接入**：换 base URL 或一键 npx 安装即可覆盖 30+ 种 agent，无需改 agent 代码（`installer/`、`proxy/README.md`）。
4. **测试与工程化极强**：约 1:1 的 Go 测试行数、三平台 CI、supply-chain 安全工作流。
5. **外部可验证**：Adobe/JetBrains/Elastic 三方独立复测，效果有旁证（README Proof 节）。

**风险/局限**

1. **单点维护者**：核心自述一人免费维护，且 telemetry 默认上报（可关），长期可持续性与隐私敏感用户需注意（`README.md` Privacy 节）。
2. **云服务闭源**：Caveman Cloud（Auto 路由、观测、Autopilot）不开源，Auto 签名登录后会把提问及附带上下文发往云端（`LICENSING.md`、README Privacy 节）。
3. **效果高度依赖工作负载**：HTML 尚无压缩器时 token 反升 9.9%；按请求计费（如 GitHub Copilot）场景下短回答无收益；规则本身约占 1,160 token 输入成本（README、docs/HONEST-NUMBERS.md）。
4. **部分组件未成熟**：Blocks 无沙箱（Python only，pre-release）、Warm cache 等多块"not yet measured"，实测数据滞后于功能宣传（README "Smaller rocks" 表）。

## 10. 总结评价

caveman 是 2026 年"agent 成本优化"赛道最具代表性的项目：把一个病毒式传播的玩笑（"why many token when few do trick"）做成了严肃的、带安全分级和可恢复契约的本地压缩基础设施，Go 引擎 + TS CLI + 双语 SDK 的工程质量达到生产级，测试规模和 CI 完备度在同类里罕见，而 `docs/HONEST-NUMBERS.md` 式的负结果公开是其最独特的工程文化标志。它适合所有重度使用编码 agent、按 token 付费且会话中有大量日志/结构化数据回读的开发者；适合研究"如何设计诚实的数据管线"的工程师精读 `engine/` 与 `engine/safety/`；对纯订阅制用户或隐私极端敏感者，价值打折。10 万星里有相当部分来自梗与热点，但其压缩引擎与代理架构是经得起源码检验的硬货。
