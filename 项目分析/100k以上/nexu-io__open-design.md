# nexu-io/open-design 项目分析

> 星数 103533（快照 2026-10-24）｜主语言 TypeScript｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

**OpenDesign 是一个"协作式 AI 设计 Agent 工作台"（The 1st Collaborative AI Design Agent Workspace）：本地优先的桌面/Web 应用，把用户已安装的编码 Agent CLI（Claude Code、Codex、Cursor 等 26 种）当作"设计引擎"，按品牌设计系统（DESIGN.md）生成网页原型、幻灯片、图片、视频动效等交付物，并支持团队协作与 Agent 二次迭代。**

| 事实 | 值 | 出处 |
|---|---|---|
| 名称 | OpenDesign（nexu-io/open-design） | README.md |
| 星数 | 103,533（快照 2026-10-24）；2026-10-30 gh api 实测 104,872 | gh api repos/nexu-io/open-design |
| 主语言 | TypeScript（monorepo，pnpm workspace） | package.json |
| 许可证 | Apache-2.0（捆绑的第三方 skill 保留各自 MIT 等许可） | package.json、README.md 末尾 |
| 当前版本 | 0.25.1（tarball HEAD），最新 release 0.26.1 "Intelligent Router"（2026-10-22） | package.json；gh api releases/latest |
| 创建时间 | 2026-04-28（约 6 个月冲到 10 万星） | gh api |
| 最近 push | 2026-10-29（前一天仍在提交，极度活跃） | gh api |
| 贡献者 | 约 449 人（contributors 分页 last=450） | gh api contributors |
| Fork / Open Issues | 11,967 / 1,308 | gh api |

## 2. 项目是什么，解决什么问题

OpenDesign 解决的核心问题是：**让设计产出"品牌级"且"Agent 原生"**。通用 AI 生成页面的痛点是缺乏品牌约束、产物是一次性代码片段、无法在团队内迭代。OpenDesign 的答案是四层可组合资产：**插件（可运行工作流）+ 功能 skill（Agent 行为）+ 设计模板（渲染蓝图）+ 设计系统（品牌契约 DESIGN.md）**，全部是可移植、可版本化的目录（README.md "Why OpenDesign"）。

它刻意**不自带 Agent**：daemon 扫描用户 PATH 上的编码 Agent CLI，把选中的 skill/模板与当前项目的 DESIGN.md 组装成提示词，spawn 真实 CLI 进程在托管项目工作目录里执行，产物是真实 HTML/CSS 文件，在沙箱 iframe 中预览（docs/architecture.md §4）。因此"换模型/换 Agent"是一键操作，模型无关性写进了架构本质。

工作流被概括为 `brief → plugin → direction → design system → artifact → handoff → memory`（README.md）：PM 提需求 → 选场景插件 → 锁定品牌方向 → Agent 产出交付物 → 交给 Cursor/Claude Code 继续工程化或导出 PPTX/PDF/MP4 → 截图、字体、调色板沉淀为下次会话的默认记忆。

除原型外，产品覆盖四类产出面：原型/仪表盘（单页 HTML 工件）、幻灯片（15 套 deck 模板 × 36 主题，导出 HTML/PDF/PPTX/ZIP/Markdown）、图片（93 个 prompt 模板）、视频动效（集成了 HeyGen 开源的 HyperFrames HTML→MP4 框架，README.md "Product tour"）。这也是它能在设计师、运营、市场等非纯工程人群中迅速传播的原因。

商业模式为开源核心 + 云服务：0.9.0 内置 OpenDesign Cloud/AMR 官方模型服务（零配置一键登录），README 顶部有 "Celebrating 100K GitHub stars with OpenDesign Go" 的付费额度促销，本地 BYOK 与云端并存（CHANGELOG.md 0.9.0、README.md）。

## 3. 技术栈与依赖

| 层 | 技术 | 出处 |
|---|---|---|
| 前端 | Next.js 16 App Router + React 18 + TypeScript | docs/architecture.md §3.1、README.md 架构表 |
| 守护进程 | Node ~24 + Express + SSE 流式 + better-sqlite3 | apps/daemon/package.json、README.md |
| 桌面 | Electron 壳 + 沙箱 renderer + sidecar IPC（STATUS/EVAL/SCREENSHOT/CONSOLE/CLICK/SHUTDOWN） | README.md 架构表、apps/desktop/package.json |
| CLI | `od`（apps/daemon/bin/od.mjs），与 Web UI 调用同一套 daemon HTTP API | package.json bin、docs/architecture.md §2 |
| MCP | @modelcontextprotocol/sdk 1.29.0，stdio MCP server 暴露项目/文件/插件/skill | apps/daemon/package.json dependencies |
| 包管理 | pnpm 10.33 monorepo（packages/*、apps/*、tools/*、shells/*、e2e） | pnpm-workspace.yaml |
| 测试 | vitest（26 个 workspace 包配置了 vitest）+ Playwright（e2e） | e2e/playwright.config.ts、各包 package.json |
| 打包/分发 | esbuild、electron-winstaller、@ffmpeg-installer/ffmpeg；shells/terminal 提供 POSIX sh + PowerShell 载体 | package.json onlyBuiltDependencies、shells/terminal/package.json |

值得注意的工程选择：依赖了 `@typescript/native`（npm 别名指向 typescript@7.0.2，package.json devDependencies）；pnpm overrides 锁了十几个传递依赖版本（package.json）。

## 4. 目录结构与代码组织

顶层（tarball HEAD 实测）：

| 目录 | 职责 |
|---|---|
| `apps/web/` | Next.js 前端：项目、聊天、文件工作区、预览、设置、市场（773 个 ts/tsx 文件） |
| `apps/daemon/` | Express 守护进程：API 权威、持久化、Agent 检测与运行（782 个 ts 文件，src 下 171 个模块/子目录） |
| `apps/desktop/` | Electron 主进程与 IPC |
| `apps/packaged/` | 打包版 daemon+web sidecar 启动（含 headless 入口） |
| `apps/closure/` | 打包闭包相关 |
| `packages/` | 15 个共享包：contracts（web/daemon 共享 DTO，166 个 ts 文件）、plugin-runtime、sidecar/sidecar-proto、dsh-runtime（DeepSeek Harness）、host、platform、standalone、release、diagnostics、download、agui-adapter、launcher-proto、metatool、registry-protocol |
| `tools/` | dev / pack / release / serve 四个生命周期工具；`pnpm tools-dev` 是唯一开发入口 |
| `shells/terminal/` | Standalone 精确分发的原生 sh/PowerShell 载体 |
| `skills/` | 171 个功能 skill 目录（169 个 SKILL.md），遵循 Claude Code Agent Skills 规范 |
| `design-templates/` | 116 个渲染模板（prototype/deck/image/video/utility 模式） |
| `design-systems/` | 154 个品牌设计系统包（以 DESIGN.md 为核心） |
| `plugins/` | `_official/` 277 官方插件 + 183 examples + community/ + registry/ + spec/SPEC.md |
| `e2e/` | Playwright 用户级验证（tests/ 50 个目录 + specs/ 8 个） |
| `deploy/` | Docker Compose、Sealos、AWS/Azure 部署 |
| `docs/` | 50+ 篇：architecture、spec、skills-protocol、agent-adapters、adr/、rfc-drafts/ 等 |
| `prompt-templates/` `craft/` `mocks/` `charts/` `figma-plugin/` | 93 个图片 prompt、通用 craft 规则、契约 mock、图表、Figma 插件 |

组织方式评述：典型 pnpm monorepo，"应用层（apps）+ 共享包（packages）+ 内容资产（skills/design-*/plugins 三大目录）+ 工具链（tools）"四象限清晰。内容资产本身就是产品主体（README 明确"drop a folder in, the picker finds it"），代码与内容同等是一等公民。分层纪律靠根/层两级 `AGENTS.md`（共 4 个：根、apps、packages、tools）约束，另有 496 行的根 AGENTS.md 规定 daemon 数据目录契约、双轨对齐规则（改能力须同步 endpoint/contract/web/CLI 四处）。

## 5. 核心架构与关键实现

### 5.1 运行时注册表：声明式适配 26 种 Agent CLI

每个 Agent 一个纯声明式 `RuntimeAgentDef`，注册表只是数组拼接（`apps/daemon/src/runtimes/registry.ts`，91 行，含 27 个 shipped def —— 26 个真实 CLI + byok-opencode 复用 OpenCode 可执行文件）。以 Claude 为例（`apps/daemon/src/runtimes/defs/claude.ts`）：

```ts
export const claudeAgentDef = {
    id: 'claude',
    bin: 'claude',
    fallbackBins: ['openclaude'],   // argv 兼容的分叉自动兜底
    versionArgs: ['--version'],
    authProbe: { args: ['auth', 'status'], timeoutMs: 5000 },
    capabilityFlags: {   // 探测 --help 后按子串置位能力表
      '--include-partial-messages': 'partialMessages',
      '--add-dir': 'addDir',
    },
    fetchModels: async (_resolvedBin, env) => loadMmdRouteModels(env, CLAUDE_FALLBACK_MODELS),
```

定义声明启动方式、提示词投递（stdin 以规避 Linux `spawn E2BIG`）、模型发现、auth 探测与流格式；共享引擎（`runtimes/runs.ts` 2675 行）负责生命周期。检测时用 `resolveAgentLaunch()` 保证"探测到的可执行文件 = 运行时将 spawn 的文件"，并发探测（docs/architecture.md §3.3）。新增一个 Agent CLI 只需一个 def 文件 + 注册表一行，这是 27 个适配器仍能保持低熵的原因。

### 5.2 两种执行画像：filesystem vs text_artifact

流格式决定交付契约（`packages/contracts/src/execution-profile.ts`，全文 7 行）：

```ts
export type ExecutionProfile = 'filesystem' | 'text_artifact';
export function executionProfileFromStreamFormat(
  streamFormat: string | null | undefined,
): ExecutionProfile {
  return streamFormat === 'plain' ? 'text_artifact' : 'filesystem';
}
```

结构化/带工具的运行时在项目工作目录写规范文件，文件事件驱动文件工作区与预览；plain 流运行时（BYOK/API 直连）无文件工具，交付"一个完整 `<artifact>` 源码块"，由宿主解析落盘进同一套预览面（docs/architecture.md §4）。这让"真 CLI 生成"和"纯 API 生成"收敛到同一预览/导出管线，是整个产品的关键抽象。

### 5.3 内容注册表与提示词组装

四张内容面（skills / design-templates / design-systems / plugins）各有 API（`/api/skills` 等），每次列表请求先扫用户可写根、再扫捆绑根，用户条目可按 id 遮蔽内置条目（docs/architecture.md §3.4）。聊天运行时把"活动设计系统 + 主 skill/模板 + craft 规则 + 项目元数据 + 本轮附加项"组装成最终提示词（docs/architecture.md §3.2；实现散布于 `apps/daemon/src/runtimes/chat-prompt-inputs.ts` 1009 行等）。

### 5.4 守护进程单体与安全边界

`apps/daemon/src/server.ts` 实测 18,961 行，是应用的组装权威（数据根 `OD_DATA_DIR` 一次性解析、静态生产服务、回环/安全策略），路由细节正逐步下沉到 `src/routes/` 的 38 个模块（docs/architecture.md §3.2、§8）。安全面（docs/architecture.md §7）：daemon 默认绑 127.0.0.1；BYOK 代理边缘做 SSRF 防护（RFC1918/链路本地/CGNAT/云元数据 IP 拦截，`OD_ALLOWED_INTERNAL_HOSTS` 严格 opt-in 白名单，README.md）；工件/插件预览跑在无宿主同源权限的沙箱 iframe；目录导入用桌面主进程铸造的短时单次 HMAC 令牌把门（`apps/daemon/src/desktop-auth.ts`，docs/architecture.md §6）。

## 6. 构建与运行

来自 README.md / QUICKSTART.md / package.json 实测命令：

```bash
# 源码运行（Node ~24 + pnpm 10.33，corepack enable）
git clone https://github.com/nexu-io/open-design.git && cd open-design
corepack enable && pnpm install
pnpm tools-dev run web          # daemon+web 前台；端口自动分配

# 质量检查（agent 贡献路径，README "Contributing as an agent"）
pnpm guard && pnpm typecheck    # 全仓 guard + 分包 typecheck
pnpm --filter @open-design/<package> test   # 单包 vitest

# Docker 自托管
cd deploy && cp .env.example .env
echo "OD_API_TOKEN=$(openssl rand -hex 32)" >> .env
docker compose up -d            # http://127.0.0.1:7456

# 插件脚手架
od plugin scaffold --id my-plugin && od plugin validate ./my-plugin
```

生命周期唯一入口是 `pnpm tools-dev`（start/stop/run/status/logs/inspect/check，docs/architecture.md §1）；桌面正式版走 `tools-pack`/`tools-release`。

## 7. 测试与工程质量

- **测试规模大**：全仓实测 3,012 个 `*.test.ts(x)` 文件（daemon 1,051、web 1,413；`find` 统计），26 个 workspace 包各有 vitest 配置；e2e 用 Playwright（`e2e/playwright.config.ts` + `playwright.visual.config.ts` 视觉基线，tests/ 50 个目录、specs/ 8 个）。风格以贴近实现的单元/集成测试为主 + 用户级 e2e 兜底。
- **CI 极重**：`.github/workflows/` 实测 55 个工作流，除 ci.yml（自建 Nexu ARC runner 舰队 + blacksmith/hosted 回退、ci_mode full/hot 两档）外，还有 20+ 个发布链工作流（cut-release、release-gate、release-rollback、release-withdraw、prerelease smoke 等）、插件预览烘焙（bake-plugin-previews 系列）、视觉基线、agent PR 探索沙箱、i18n/目录校验等。
- **代码质量观察**：正面——分层文档密度罕见（spec.md、architecture.md、skills-protocol.md、agent-adapters.md、adr/、rfc-drafts/），AGENTS.md 双轨对齐规则明确，安全设计（SSRF、HMAC 导入门、沙箱 iframe）有专门章节。负面——`apps/daemon/src/server.ts` 单文件 18,961 行、`runtimes/runs.ts` 2,675 行，巨型文件与"路由模块化"并行存在，新贡献者定位成本高（docs/architecture.md §3.2 也承认路由边界仍在下沉中）。

## 8. 社区与维护状态

- **活跃度**：gh api pushed_at 2026-10-29（分析前一天）；仓库 2026-04-28 创建，6 个月破 10 万星，README 顶部已挂 100K 星庆祝活动。
- **发布节奏**：CHANGELOG.md 显示 0.9.0（2026-05-29）一周内合并 310 个 PR、88 位贡献者；最新 release 0.26.1 于 2026-10-22 发布，版本号半年内从 0.9 迭代到 0.26，节奏为周更级别。
- **贡献者结构**：约 449 人（gh api contributors 分页），README 列出 3 位日审维护者（@Nagendhra-web、@Sid-Qin、@YOMXXX）+ MAINTAINERS.md 定义晋升规则与"OpenDesign Fellow"计划（$1,000/MR 赞助、免费 LLM 额度）。CHANGELOG 条目大量按贡献者致谢，社区贡献占比高。
- **多语言社区**：13 个语言版 README、QUICKSTART/CONTRIBUTING 多语翻译（docs/i18n/），插件 manifest 内置 18 语种 i18n（plugins/_official/scenarios/od-default/open-design.json）。
- **商业化**：0.9.0 起内置 OpenDesign Cloud/AMR 官方模型服务，README 有 Go 套餐促销与合作模型列表；属"开源核心 + 托管服务"模式。未确认的信息：核心团队雇佣规模、营收数据。

## 9. 亮点与风险

**亮点**

1. **"不造 Agent，而编排 Agent"的定位聪明**：把 26 种已有 CLI 声明式接入（apps/daemon/src/runtimes/defs/），模型无关 + 零自建推理成本，用户迁移路径为零摩擦。
2. **内容资产即产品**：171 skills + 116 模板 + 154 设计系统 + 460 插件全部是可移植目录，社区可"drop a folder"参与，天然形成 UGC 飞轮（README 各目录计数实测）。
3. **filesystem/text_artifact 双执行画像**（packages/contracts/src/execution-profile.ts）用 7 行代码统一了两类运行时的交付管线，抽象性价比极高。
4. **AI 原生的开源治理**：AGENTS.md 分层契约、"Contributing as an agent" 章节、 Fellow 计划、agent-friendly CI（gh issue develop 流程写进 README），明显为"Agent 贡献者"时代设计。
5. **工程质量与文档密度顶尖**：3,000+ 测试文件、55 个 CI 工作流、50+ 篇 docs、18 语种 i18n。

**风险/局限**

1. **daemon 单体膨胀**：server.ts 近 1.9 万行，模块边界仍在迁移中，长期演化有熵增风险（apps/daemon/src/server.ts 实测）。
2. **强依赖上游 CLI 的非契约行为**：适配器靠探测 `--help` 子串、隐藏 flag 值拒绝等方式推断能力（defs/claude.ts 注释自证），上游 CLI 升级随时可能破坏 26 个适配器中的任何一个。
3. **Linux 桌面缺位**：官方无预构建 Linux 产物，仅源码/Docker 路径（README.md 下载节、roadmap #4368），对开源桌面用户是明显缺口。
4. **商业模式张力**：Cloud/AMR 服务与本地 BYOK 并存，赞助商推广、遥测（consent-gated）等使"本地优先"承诺需要持续验证（README.md）。

## 10. 总结评价

OpenDesign 是 2026 年"Agent 编排 + 垂直工作台"浪潮中最成功的产品级样本之一：它不训练模型、不自建 Agent，而是把自己做成所有编码 Agent 的"设计交付层"，用 154 个品牌设计系统约束输出质量，用四层可组合资产目录把社区变成供给方——这正是它 6 个月冲上 10 万星并保持日更的底层逻辑。代码库规模与规范程度都远超同期项目，适合三类人：想把本地 AI 工作台做得"品牌级"的设计师/产品团队直接使用；做 Agent 编排、适配器层、SSE 流式工作台的工程师研读 `runtimes/` 与 `docs/architecture.md`；想给主流 Agent CLI 生态做内容（skill/模板/设计系统）的创作者从 `plugins/spec/SPEC.md` 入手。注意其发布节奏极快、daemon 单体仍在重构，跟进源码宜以 release tag 为锚点。
