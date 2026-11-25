# koala73/worldmonitor 项目分析

> 星数 89313（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：**World Monitor 是一个实时全球情报仪表盘（OSINT situational awareness），将新闻、地缘政治、军事航班、金融市场、网络威胁、气候灾害等数十类数据流聚合到一张可交互地图 + 面板网格中，并叠加浏览器端 AI 分析**。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | worldmonitor（npm 包 `world-monitor` v2.11.6） | `package.json` |
| 作者 | Elie Habib（koala73），Copyright 2024-2026 | `README.md`、`LICENSE` |
| 仓库创建 | 2026-01-08，最近 push 2026-10-30 | `gh api repos/koala73/worldmonitor` |
| 语言 | TypeScript（Vanilla TS，无 React/Vue），桌面壳 Rust（Tauri） | `package.json`、`src-tauri/` |
| 许可证 | AGPL-3.0-only（另有商业授权通道） | `LICENSE`、`README.md` |
| 官网 | worldmonitor.app（另有 tech./finance./commodity./happy./energy. 五个变体站点） | `README.md` |
| 规模 | 全仓 9002 文件；`src/` 951 个 TS 文件约 30 万行；贡献者 168 人 | 源码统计、gh api |
| 最新发布 | v2.11.6（2026-10-25），一周内连发 6 个版本 | gh api releases |

## 2. 项目是什么，解决什么问题

开源情报（OSINT）数据分散在几十个上游：GDELT/新闻 RSS、ACLED 冲突数据、OpenSky 军事航班、Finnhub/Yahoo 行情、FIRMS 火情、GPS 干扰图等。普通用户不可能同时盯住这些源。World Monitor 的答案是把这些流统一拉取、缓存、AI 摘要后渲染在一个"作战室"式界面里：左侧双引擎地图（deck.gl 平面图 + globe.gl 三维球），右侧是 100 多个专题面板（行情、冲突、网络、气候、卫星、预测市场、直播电视等）。`ARCHITECTURE.md` 自述其聚合了 **805 个上游源主机**（`ARCHITECTURE.md` §1）。

项目有几个明显的产品层设计：其一是 **变体系统**——同一份代码库通过 `VITE_VARIANT` 编译出 world/tech/finance/commodity/happy/energy 六个站点，各变体默认面板、图层、主题不同（`ARCHITECTURE.md` §3、`package.json` 的 `dev:tech` 等脚本）。其二是 **AI 能力本地优先**——浏览器内通过 Transformers.js + ONNX 跑 MiniLM 嵌入、情感分析、摘要、NER（`src/workers/ml.worker.ts`），也可接 Ollama/OpenRouter，零 API key 即可运行（`README.md` "What It Does"）。其三是 **CII 国家不稳定指数**——对 31 个 Tier-1 国家实时打分，权重表显式硬编码每个国家的基线风险与事件乘数（`shared/cii-weights.ts`）。

它同时是一个"agent 时代"的产品：内置 MCP 服务器（`worldmonitor.app/mcp`）、REST API（`api.worldmonitor.app` + OpenAPI 规范）、CLI（`npx worldmonitor` 终端仪表盘）、Python/Ruby/Go/Rust 四语言 SDK（`sdk/`），README 的 badge 区一半在讲程序化接入（`README.md` "Programmatic Access"）。这解释了它在 2026 年一年内冲到近 9 万星的原因之一：它把自己做成了 AI agent 可直接调用的全球情报工具，并明确以 "Palantir 开源平替" 类话题传播（repo topics 含 `palantir`、`osint`、`agent`，gh api）。

## 3. 技术栈与依赖

- **前端**：Vanilla TypeScript + Vite；地图双引擎 deck.gl 9 + MapLibre GL / globe.gl + Three.js；PMTiles 自托管底图、supercluster 聚合、h3-js 六边形格（`package.json` dependencies）
- **AI/ML**：`@xenova/transformers` + `onnxruntime-web`（浏览器端推理）、`@anthropic-ai/sdk` / OpenRouter（服务端摘要）、Ollama 本地推理（`README.md` Tech Stack 表）
- **后端**：Vercel Edge Functions（`api/` 290 文件）、Convex Cloud（计费/用户/向量记忆，`convex/` 200 文件）、Railway 中继与种子任务（`scripts/ais-relay.cjs` 等）、Upstash Redis 三层缓存、Cloudflare Worker 做 CORS 预检（`workers/api-cors-preflight/`）
- **契约**：Protocol Buffers + sebuf HTTP 注解生成客户端/服务端/OpenAPI（`proto/`、`ARCHITECTURE.md` §5）
- **桌面**：Tauri 2（Rust）+ Node.js sidecar，密钥存系统 keyring（`src-tauri/`、`ARCHITECTURE.md` §7）
- **发行面**：npm CLI、PyPI/RubyGems/Go/Rust SDK（`sdk/`）、GHCR 容器镜像（`docker/`）、PWA
- 依赖策略：运行时依赖 51 个，少量依赖以 `file:vendor/` 内置（braces、vercel-og、stream-json），规避供应链/老版本 npm 解析问题（`package.json`）

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `src/` | SPA 主体：`components/`（218 文件，109 个 Panel 子类）、`services/`（数百个领域服务）、`workers/`（ML/分析 Web Worker）、`config/`、`locales/` |
| `api/` | Vercel Edge Functions 入口：域网关（proto 生成）+ 手写运营端点（`api/api-route-exceptions.json` 管控例外） |
| `server/` | Edge Function 复用层：`gateway.ts`（网关工厂）、`router.ts`、`worldmonitor/<domain>/v1/handler.ts`（各域 RPC 处理器） |
| `convex/` | Convex 后端：计费、API key、用户偏好、邮件、情报历史向量检索（`convex/schema.ts`） |
| `scripts/` | 858 个文件：`seed-*.mjs` 数据种子、几十个自研 lint/enforce 治理脚本、 Railway 定时任务 |
| `shared/` | 前后端共享纯逻辑（CII 权重、国家头条匹配、威胁分类器），明确"两处副本即缺陷"（`ARCHITECTURE.md` §4） |
| `cli/` | npm 包 `worldmonitor` 的 CLI + 终端仪表盘（`cli/` 独立 package.json） |
| `sdk/` | go/python/ruby/rust 四语言零依赖 SDK |
| `src-tauri/` | Tauri 2 桌面壳（Rust + Node sidecar） |
| `proto/` | protobuf 服务定义，`make generate` 产出 `src/generated/` 与 OpenAPI |
| `tests/` | 2343 个 node:test 风格 `.mjs/.mts` 数据/契约测试 |
| `e2e/` | 97 个 Playwright 规格文件（含视觉金标、a11y、性能预算） |
| `docs/` | 288 个 mdx（Mintlify 文档站源），另有 `ARCHITECTURE.md`(53KB)、`CONCEPTS.md`(189KB)、`CHANGELOG.md`(75KB) |

组织方式评述：这是**单体多表面仓库**——一个 SPA 代码库同时产出六个网站变体、桌面端、CLI、SDK 和 MCP 服务；边界不靠包管理器 monorepo（没有 workspaces），而靠大量自研 lint 脚本（`lint:boundaries`、`lint:api-contract`、`enforce-panel-content-writes` 等，见 `package.json` scripts）在 CI 里强制。工程治理密度在开源项目里罕见。

## 5. 核心架构与关键实现

### 5.1 面板框架与客户端状态（`src/components/Panel.ts`、`src/App.ts`）

所有面板继承 `Panel` 基类：`setContent(html)` 以 150ms 防抖渲染，事件委托挂在稳定的 `this.content` 上，行列跨度（col-span/row-span）可拖拽并持久化到 localStorage（`src/components/Panel.ts` 中 `persistPanelColSpan`/`deltaToColSpan`）。全应用**无外部状态库**，用一个中央可变对象 `AppContext` 持有地图引用、面板实例、全部缓存数据，URL 状态双向同步（`ARCHITECTURE.md` §3）。`App.init()` 分 8 个阶段：存储/i18n → ML Worker 预热 → sidecar 就绪 → `/api/bootstrap` 双层并发水合 → 布局 → UI → 数据 → 智能轮询（`ARCHITECTURE.md` §3）。

### 5.2 重计算 Web Worker：新闻聚类与跨流关联（`src/workers/analysis.worker.ts`、`src/services/analysis-core.ts`）

Jaccard 相似度 O(n²) 新闻聚类和跨域关联检测（军事/经济/灾害/升级信号汇聚）全部放进 Worker，避免阻塞主线程。核心逻辑抽取到 `analysis-core.ts` 保证 Worker 与主线程单一实现来源；Worker 内保留 `previousSnapshot` 与 30 分钟 `recentSignalKeys` 去重集，防止关联信号重复触发（`src/workers/analysis.worker.ts` 头部与 `markSignalSeen`）。ML 侧 `ml.worker.ts` 在 Worker 内跑 ONNX（MiniLM-L6 嵌入 + 情感 + 摘要 + NER），并维护 IndexedDB 向量库 `vector-db.ts` 做头条语义记忆。

### 5.3 Edge 网关工厂与缓存契约（`server/gateway.ts`）

每个业务域独立打成单个 Edge Function（`api/<domain>/v<N>/[rpc].ts`），冷启动开销降低约 20 倍（`server/gateway.ts` 头部注释）。`createDomainGateway(routes)` 固定执行管线：Origin 校验 → CORS → OPTIONS → API key → 端点级+全局兜底限流 → 路由匹配 → POST-to-GET 兼容（旧客户端 JSON 标量体转查询参数）→ 错误边界 → FNV-1a ETag/304 → 缓存头。缓存分六档契约（fast 300s / medium 600s / slow 1800s / static 7200s / daily 86400s / no-store），由测试 `tests/edge-functions.test.mjs` 强制每函数不得引入无关模块（`ARCHITECTURE.md` §4）。

### 5.4 数据管道：种子任务与原子发布（`scripts/_seed-utils.mjs`、`scripts/ais-relay.cjs`）

上游数据由两类进程写入 Redis：Railway 上常驻的 AIS 中继跑市场/航空/GPSJAM/CII/UCDP 等 seed loop；`scripts/seed-*.mjs` 在 Railway cron 定时跑。发布走 `atomicPublish()`：Redis SET NX 抢锁 → 校验 → 写缓存 key → 写 `seed-meta:<key>`（fetchedAt/recordCount）→ 释放锁；`api/health.js` 逐 key 对比 maxStaleMin 输出 OK/STALE/WARN/EMPTY，健康度直接暴露给前端面板（`ARCHITECTURE.md` §6）。读模型类 RPC（如五因子评分卡、供应链脆弱性）采用"双槽写入 + 指针原子切换"避免半成品数据可见（`ARCHITECTURE.md` §4 scorecard/supply-chain 段）。

### 5.5 CII 国家不稳定指数（`shared/cii-weights.ts`、`shared/cii-band.js`）

指数 = 国家基线风险 × 事件乘数叠加上升实时事件流，权重表显式可审计：例如 `RU: { baselineRisk: 35, eventMultiplier: 2.0 }`、`KP: { baselineRisk: 45, eventMultiplier: 3.0 }`（`shared/cii-weights.ts`）。前端 `src/components/CIIPanel.ts` 渲染 v14 分数、分带与 24 小时漂移（`README.md`）。

## 6. 构建与运行

```bash
# 开发（需 Node 24，见 .nvmrc；无任何环境变量即可启动）
npm install && npm run dev            # localhost:3000
npm run dev:finance                   # 其他变体同理 dev:tech/dev:happy/...

# 校验与构建
npm run typecheck:all                 # tsc --noEmit 主项目 + api
npm run lint                          # biome + safe-html 等
npm run build:full                    # 生产构建（含 OpenAPI/agent-skills/blog/pro 子构建）

# 测试
npm run test:data                     # node --test 跑 tests/*.test.mjs|mts
npm run test:convex && npm run test:dom   # vitest
npm run test:e2e                      # Playwright（按变体拆分）
```

出处：`README.md` Quick Start、`package.json` scripts、`.nvmrc`（Node 24）。README 特别警告 Debian 12 自带 Node 18 会因 `file:vendor/braces` 依赖而 `npm install` 失败。Docker 部署见 `docker-compose.yml`、`Dockerfile`（nginx 托管 SPA 并反代 API）、`SELF_HOSTING.md`。

## 7. 测试与工程质量

- **规模**：`tests/` 2343 个测试文件（node:test 风格，多为数据管道/契约/不变量测试，命名如 `a11y-issue-4373-invariants.test.mjs`），`e2e/` 97 个 Playwright 规格，Convex 与 DOM 分别有独立 vitest 配置（`vitest.config.mts`、`vitest.dom.config.mts`）
- **CI**：52 个 GitHub workflow（`.github/workflows/`），除常规 lint/test/CodeQL/security-audit 外，还有 `mcp-live-smoke.yml` 每 15 分钟探测线上 MCP/OAuth 端点、`proto-check.yml` 强制生成代码新鲜度、`deploy-gate.yml`、`e2e-visual.yml` 视觉金标等
- **自研治理 lint**：`enforce-safe-html`（XSS 面）、`enforce-rate-limit-policies`、`check-vite-env-secrets`（构建时严格模式防止密钥进包）、`check-unicode-safety`、`bundle-budgets`（包体积预算，分 surface）、`check-inventory-count-contracts`（README 数字与实际一致），全部列在 `package.json` scripts
- **观察**：工程质量是这个项目最突出的部分之一——测试金字塔铺到数据源级，安全有专人披露致谢（`README.md` Security Acknowledgments：IPC 暴露、渲染器-sidecar 信任边界、fetch 补丁凭据注入三项 2026 年披露），文档（`ARCHITECTURE.md` 逐条标注 Source files）达到内部工程手册水准

## 8. 社区与维护状态

- **活跃度**：仓库 2026-01-08 创建，2026-10-30 仍有 push；最近 5 个 release 集中在 2026-10-19 至 10-25（约每周 6 版），v2.11.6 与 `package.json` 版本一致（gh api releases）
- **贡献者**：168 人（gh api contributors 分页尾页号），但 commit 主导结构未确认（tarball 无 git 历史）；`AGENTS.md`、`CONTRIBUTING.md`（37KB）、issue 模板齐全
- **社区规模**：89.8k stars、13.8k forks、356 open issues（gh api），README 提供中日俄三语版本
- **商业化**：Convex 上接 Dodo Payments 订阅计费（`convex/payments/`、`src/services/billing.ts`），Pro 档位 + API key 分层（`api/_api-key.js`、premium 面板门控 `PanelGateReason`），AGPL + 商业授权双轨（`README.md` License 表）

## 9. 亮点与风险

**亮点**
1. 数据聚合广度罕见：805 个源主机、六档缓存契约、原子发布与健康度自报（`ARCHITECTURE.md`），把"仪表盘"做成了可运维的数据产品
2. 全发行面覆盖：Web/桌面(Tauri)/CLI/SDK×4/MCP/REST 一套代码，agent 可编程访问是星数暴涨的明显推手（`README.md`、`sdk/`）
3. 隐私友好的本地 AI：Transformers.js/ONNX + Ollama，无 key 可用（`src/workers/ml.worker.ts`）
4. 工程治理自成体系：52 个 CI workflow、2343 个测试文件、几十个自研 enforce 脚本，把架构约束固化成可执行检查
5. 文档质量极高：`ARCHITECTURE.md`/`CONCEPTS.md` 每个结论标注源码路径，且有"同 PR 必须更新架构文档"的所有权规则

**风险/局限**
1. AGPL-3.0 对商用集成有传染性约束，闭源使用需另谈商业授权（`README.md` License 表）
2. 单作者主导 + 一周 6 版的节奏：变更吞吐极大，下游 fork 跟随成本高；未确认 bus factor
3. 重度依赖第三方上游与多平台（Vercel/Railway/Convex/Upstash/Cloudflare），任一上游限流或改版即可能造成面板退化（`api/health.js` 的 STALE/EMPTY 状态本身说明这是常态风险）
4. 仓库体量巨大（全仓 9002 文件、`.env.example` 289 个变量、`vite.config.ts` 83KB、`vercel.json` 69KB），新人上手与局部改动的心智负担高

## 10. 总结评价

World Monitor 是 2026 年"OSINT + AI agent"浪潮中最成功的开源单品之一：它用约 30 万行 TypeScript 把散落的上游情报源压成一张统一态势图，再把同一个代码库裂变出六个站点、桌面端、CLI、四语言 SDK 和 MCP 服务，让浏览器用户、终端用户和 AI agent 都是它的"用户"。技术上它是 Vanilla TS 单体 + Edge Functions 的教科书式实践，工程上其自研治理脚本与测试密度达到了商业公司水准；代价是复杂度集中、AGPL 约束和对少数维护者的强依赖。适合三类人：想要自托管情报仪表盘的分析师/研究者（`SELF_HOSTING.md`）、想把 OSINT 能力接进 agent 工作流的开发者（`sdk/`、MCP）、以及想学习"单代码库多表面 + 边治理脚本化"大型前端工程的工程师（`ARCHITECTURE.md` + `scripts/` 值得整本读）。
