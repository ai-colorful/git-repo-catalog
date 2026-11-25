# bytedance/UI-TARS-desktop 项目分析

> 星数 39599（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：字节跳动开源的多模态 AI Agent 全栈（Multimodal AI Agent Stack），从 GUI Agent 桌面客户端（UI-TARS Desktop）、终端/浏览器通用 Agent（Agent TARS CLI）到自研 Agent 框架（Tarko），把 UI-TARS / Doubao 视觉模型接到真实的电脑与浏览器操作上。

| 项目 | 内容 |
| --- | --- |
| 名称 | bytedance/UI-TARS-desktop |
| 星数 | 39599（forks 4041，open issues 511） |
| 语言 | TypeScript（1452 个 TS/TSX 文件，pnpm monorepo） |
| 许可证 | Apache-2.0（`LICENSE`，gh api license.spdx_id 一致） |
| 创建时间 | 2025-01-19 |
| 最近活跃 | 最后推送 2026-10-09（`chore(all): release 0.3.1`），**仓库已归档（archived=true）** |
| 最新版本 | v0.3.1（2026-10-08，agent-server 安全修复） |

## 2. 项目是什么，解决什么问题

UI-TARS-desktop 是字节 Seed 团队 UI-TARS 视觉语言模型（论文 arXiv:2501.12326，`README.md` Citation 节）的官方落地客户端群。它解决的核心问题是：让多模态大模型"看懂"屏幕截图后，直接以人类的方式（鼠标点击、键盘输入、滚动）操作真实 GUI，而不依赖 DOM/Accessibility API。仓库演化至今实际承载两条产品线（`README.md` 开篇表格）：

- **UI-TARS Desktop**：Electron 桌面应用，用自然语言驱动本地电脑/本地浏览器/远程电脑/远程浏览器四类操作器（`apps/ui-tars/src/main/services/runAgent.ts` 中 `Operator` 枚举四种 case）。
- **Agent TARS**：通用多模态 Agent 栈，CLI + Web UI 形态，内核建立在 MCP 上，混合浏览器控制（GUI/DOM/混合策略），`npx @agent-tars/cli@latest` 即可运行。

仓库已从单一桌面 App 生长为四层栈：桌面应用（`apps/ui-tars`）→ GUI Agent SDK 与操作器（`packages/ui-tars`）→ MCP/浏览器基础设施（`packages/agent-infra`）→ 独立的 Agent 框架与新一代 SDK（`multimodal/` 子 monorepo：tarko 框架、agent-tars、gui-agent、omni-tars）。分析时需注意 `multimodal/` 有自己的 pnpm-lock 与 CHANGELOG，是嵌套的独立子仓。

**归档说明**：gh api 确认 `archived=true`，最后一次推送即 v0.3.1 安全修复发布（2026-10-09）。README 与 issue 检索未发现明确的官方迁移公告/继任仓库指向（未确认）；其生态关联项目 `agent-infra/sandbox`（Agent 一体化沙箱，6.3k 星）仍独立活跃，npm 上的 `@agent-tars/cli`、`@tarko/*` 包随 v0.3.1 发布后冻结。

## 3. 技术栈与依赖

- **语言/运行时**：TypeScript 全栈，Node >= 20（根 `package.json` engines），pnpm 9.10 + Turborepo（`turbo.json`）。
- **桌面端**：Electron + electron-vite（开发）+ electron-forge（打包：maker-squirrel/dmg/zip/pkg，`apps/ui-tars/package.json` scripts 与 `forge.config.ts`）。
- **渲染层**：React + zustand 5 + Tailwind CSS 4（`apps/ui-tars/package.json` dependencies），页面位于 `apps/ui-tars/src/renderer/src/pages/`（home/local/remote/settings/widget）。
- **GUI 自动化**：`@computer-use/nut-js`（字节 fork 的鼠标键盘模拟库）、Electron `desktopCapturer` 截屏、`jimp`/`sharp` 图像处理（`apps/ui-tars/src/main/agent/operator.ts`、`packages/ui-tars/operators/nut-js/src/index.ts`）。
- **模型接口**：OpenAI SDK 兼容协议接 UI-TARS / Doubao 系列模型（`packages/ui-tars/sdk/src/Model.ts` 直接 import `openai`），支持自部署 vLLM（`docs/quick-start.md` 引导 HuggingFace Space 部署取 Base URL/API Key）。
- **浏览器控制**：`@agent-infra/browser`（"A tiny Browser Control library"，`packages/agent-infra/browser/package.json`）、CDP（远程屏传用 `Page.startScreencast`，`apps/ui-tars/src/renderer/src/pages/remote/cast/canvas.tsx:259`）。
- **MCP**：`@agent-infra/mcp-client`、`mcp-http-server`、`mcp-servers`（browser/filesystem/search/commands 四个内置服务，`packages/agent-infra/mcp-servers/`）。
- **测试/质量**：vitest + @playwright/test（e2e）、codecov 组件化覆盖（`codecov.yml`）、secretlint/secret-scan、commitlint + husky。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `apps/ui-tars/` | Electron 桌面应用（main/preload/renderer 三进程结构，v0.2.4） |
| `packages/ui-tars/` | 桌面端核心库：sdk（GUIAgent 循环+模型）、action-parser（模型输出解析）、operators（nut-js/browser-operator/browserbase/adb）、electron-ipc（类型安全 IPC）、shared |
| `packages/agent-infra/` | Agent 基础设施：browser、browser-use、mcp-client、mcp-http-server、mcp-servers、search（bing/duckduckgo/browser-search） |
| `packages/common/` | 共享 configs 与 electron-build 打包配置 |
| `multimodal/` | 嵌套子 monorepo（独立 lock/CHANGELOG）：`agent-tars`（cli/core/interface）、`tarko`（约 20 个包的 Agent 框架）、`gui-agent`（新一代 SDK + operator-nutjs/adb/browser/aio）、`omni-tars`（可组合 Agent）、benchmark、websites |
| `infra/` | pdk 发布工具链 |
| `docs/` | quick-start/deployment/preset/setting + `archive-1.0/`（v1.0 时期文档归档） |
| `examples/` | gui-agent-2.0 示例、operator-browserbase 示例 |
| `rfcs/`、`scripts/`、`patches/` | RFC、发布脚本、依赖补丁 |

组织方式评述：三层嵌套 monorepo（根 → multimodal 子仓 → 各包），靠 workspace 协议互链。`packages/ui-tars/*` 已发 npm（各包带 CHANGELOG，changesets 管版本），是"应用与可复用包同仓"的典型形态；`multimodal/` 独立成嵌套仓则是后期 Agent TARS 快速扩张的产物，结构上略复杂但边界清晰。

## 5. 核心架构与关键实现

### 5.1 GUI Agent 主循环（截图 → 模型 → 动作执行）

`packages/ui-tars/sdk/src/GUIAgent.ts`（约 600 行）是全仓库最核心的类：`run(instruction)` 进入 `while(true)` 循环，每轮：`operator.screenshot()`（asyncRetry 重试）→ Jimp 校验图像有效性（无效则计入 snapshotErrCnt）→ `toVlmModelFormat` 组装多模态消息（滑动图片窗口）→ `model.invoke` → `actionParser` 解析出 `parsedPredictions` → 逐个 `operator.execute()` 执行动作；`finished()`/`call_user()`/达到 `maxLoopCount` 时终止。支持 `pause/resume/stop`（Promise 门闩实现，L495-513）与 `onData` 流式回调。模型调用失败按 30 秒间隔重试、动作执行失败重试 1 次（重试策略由调用方注入，见 `apps/ui-tars/src/main/services/runAgent.ts` L217-227）。

```ts
// packages/ui-tars/sdk/src/GUIAgent.ts L185-189, L352-353
const snapshot = await asyncRetry(() => operator.screenshot(), {...});
// ...
for (const parsedPrediction of parsedPredictions) {
  const executeOutput = await asyncRetry(() =>
    operator.execute({ prediction, parsedPrediction, screenWidth: width, ... }), {...});
```

### 5.2 VLM 接口层（UITarsModel）

`packages/ui-tars/sdk/src/Model.ts`：`UITarsModel extends Model`，走 OpenAI 兼容协议；支持 Chat Completions 与 Response API 双通道（`useResponsesApi`，用 `previousResponseId`/`headImageContext` 复用上文，L58-64），按模型版本区分生成上限与像素约束（`MAX_PIXELS_V1_0/V1_5/DOUBAO`，V1_5 的 max_tokens 提到 65535，L110）。坐标因子 `factors: [number, number]`（L67-69）负责把模型输出的相对坐标换算为屏幕物理坐标。模型端点完全由配置决定——既可指向 Volcengine Ark、也可指向自部署 vLLM/HuggingFace Space（`docs/quick-start.md` L63-73）。

### 5.3 操作器层：鼠标键盘模拟与四种 Operator

统一抽象 `Operator`（`packages/ui-tars/sdk/src/types.ts`）要求实现 `screenshot()` 与 `execute({action_type, action_inputs})`；每个 Operator 用 `static MANUAL.ACTION_SPACES` 声明动作空间（click/drag/hotkey/type/scroll/wait/finished/call_user），SDK 用它渲染 system prompt（`GUIAgent.ts` L515-525）。四个实现：

- **本地电脑**：`NutJSElectronOperator`（`apps/ui-tars/src/main/agent/operator.ts`）继承 `@ui-tars/operator-nut-js`，重写截屏用 Electron `desktopCapturer` 抓主屏（JPEG 75 + scaleFactor，L52-83）；Windows 下 `type` 动作改为剪贴板 Ctrl+V 注入以兼容中文输入（L89-100）。底层动作用 `@computer-use/nut-js` 的 `mouse/keyboard/straightTo`（`packages/ui-tars/operators/nut-js/src/index.ts`）。
- **本地浏览器**：`DefaultBrowserOperator`（`@ui-tars/operator-browser`，CDP 连本机 Chrome）。
- **远程电脑/浏览器**：`RemoteComputerOperator`（`apps/ui-tars/src/main/remote/operators.ts`）经 `ProxyClient`（`apps/ui-tars/src/main/remote/proxyClient.ts`，849 行：register/proxy/browsers/time-balance 沙箱 API）操作云端沙箱，分 free（免费试用 30 分钟，L25）与 subscription 两种模式；远程浏览器屏传走 CDP `Page.startScreencast` 渲染到 canvas（`pages/remote/cast/canvas.tsx`）。**注意**：任务简报中提到的"火山引擎 RTC 连接层"在 HEAD 源码中无证据（全库 grep 无 RTC 依赖），现行远程链路是 HTTP 沙箱代理 + CDP screencast；与火山引擎的关联仅体现为 FaaS 模板引流（`renderer/src/const/index.ts` 中 console.volcengine.com/vefaas 链接）与 `docs/quick-start.md` L55 的"远程服务 2025-08-20 停止"公告。早期版本是否用过 RTC 未确认。

### 5.4 Electron 集成与 Agent TARS 框架

- **类型安全 IPC**：`packages/ui-tars/electron-ipc` 提供 `initIpc.create()` + `t.router/t.procedure`（tRPC 风格）；`apps/ui-tars/src/main/ipcRoutes/agent.ts` 暴露 `runAgent/pauseRun/resumeRun/stopRun/setInstructions` 等路由，`GUIAgentManager` 单例持有当前 GUIAgent 实例供 stop/pause 控制。
- **runAgent 编排**：`apps/ui-tars/src/main/services/runAgent.ts` 按 settings 选择操作器、拼装模型配置（本地 vlmBaseUrl/vlmApiKey 或远程 FREE_MODEL_BASE_URL）、注入重试策略，并在 `onData` 中做 SoM 标注（`markClickPosition` + ScreenMarker 置顶窗口把模型点击位置画在屏幕上，L60-111）。
- **Agent TARS 内核**：`multimodal/agent-tars/core/src/agent-tars.ts` 中 `AgentTARS extends MCPAgent`（来自 `@tarko/mcp-agent`），聚合 browser/filesystem/search 三类本地环境（`multimodal/agent-tars/core/src/environments/local/`），浏览器控制模式经 `browser-control-validator` 校验；事件流（Event Stream）驱动 Web UI 与上下文工程。框架本体在 `multimodal/tarko/agent/src/agent/`（base-agent/agent-runner/event-stream/tool-manager/tool-call-engine，约 2400 行），`multimodal/omni-tars/` 进一步提供 `AgentComposer`/`ComposableAgent` 插件化组合。`multimodal/gui-agent/agent-sdk`（npm 名 `@multimodal/seed-gui-agent`）是 GUI Agent 的 2.0 版 SDK，把动作执行改造为 ToolCallEngine 工具调用形态。

## 6. 构建与运行

```bash
# 安装（Node >= 20，pnpm 9.10）
pnpm install

# 开发运行桌面应用（turbo -> electron-vite dev）
pnpm run dev:ui-tars        # CONTRIBUTING.md L110

# 构建与打包（electron-forge）
pnpm run build              # typecheck + electron-vite build + forge make（apps/ui-tars/package.json）

# 单元测试 / 覆盖率 / 基准（vitest）
pnpm test && pnpm run coverage && pnpm run test:bench
# e2e（Playwright）
cd apps/ui-tars && pnpm run build:e2e && pnpm run test:e2e

# Agent TARS CLI（免安装体验）
npx @agent-tars/cli@latest
agent-tars --provider volcengine --model doubao-1-5-thinking-vision-pro-250428 --apiKey your-key
```

## 7. 测试与工程质量

- 测试规模：全仓 129 个 `*.test.ts/spec.ts` 文件（find 统计），SDK/操作器/IPC 路由/渲染逻辑均有单测（如 `apps/ui-tars/src/main/ipcRoutes/screen.test.ts`、`packages/ui-tars/sdk/src/utils.test.ts`）；e2e 用 Playwright 驱动 Electron（`apps/ui-tars/e2e/app.test.ts` + `playwright.config.ts`）。
- CI 体系完整：`test.yml`（macOS，turbo 缓存，paths-ignore multimodal）、`agent_tars_test.yml`、`e2e-ui-tars.yml`、`benchmark.yml`（vitest bench）、`scorecard.yml`（OpenSSF）、`secret-scan.yml`/`secretlint.yml`；codecov 按组件上报（`codecov.yml`，action-parser 等有独立徽章）。
- 质量观察：类型安全 IPC、统一的错误分类体系（`GUIAgentError` + `ErrorStatusEnum` 枚举，`GUIAgent.ts` L527-600）、密钥防泄漏扫描进 CI、v0.3.1 专门做了服务端安全加固（仅监听 127.0.0.1、非回环 host 强制 token，见 release 说明）——工程化水准明显高于一般开源 Electron 项目。

## 8. 社区与维护状态

- **活跃度**：2025-01 创建，2025 年高频迭代（v0.1.0 2025-04、v0.2.0 2025-06、Agent TARS Beta 2025-06、CLI v0.3.0 2025-11），2026 年放缓至安全修复（最近 8 条提交均为 fix(security)/docs/release，gh api commits），2026-10-09 发布 v0.3.1 后**归档**。
- **贡献者**：49 人（gh api contributors 分页 Link header 推算），以字节 Seed/Team 团队为主力。
- **发布节奏**：changesets + 自研 pdk 发布工具链（`infra/pdk/`），npm 多包同发（`scripts/release-pkgs.sh`）。
- **issue 状态**：511 个 open issue 随归档冻结；issue 模板区分 agent_tars / ui_tars_desktop 两产品线（`.github/ISSUE_TEMPLATE/`）。
- **归档去向**：README/置顶检索未见明确迁移公告（未确认）；相关生态 `agent-infra/sandbox` 独立活跃，tarko/agent-tars 包冻结在 npm。

## 9. 亮点与风险

**亮点**
1. "模型-客户端协同"的标杆样本：UI-TARS 论文官方客户端，动作空间（ACTION_SPACES）直接内嵌在 Operator 类里并动态注入 system prompt，模型能力与客户端约束强耦合、易于对齐（`packages/ui-tars/sdk/src/GUIAgent.ts`）。
2. Operator 可插拔抽象干净：本地电脑/本地浏览器/远程电脑/远程浏览器/Android(adb)/云沙箱(aio) 共享同一 `Operator` 接口，SDK 可脱离 Electron 复用（`packages/ui-tars/sdk` + `docs/sdk.md`）。
3. 工程化顶尖：tRPC 风格类型安全 IPC、SoM 点击可视化、事件流架构、OpenSSF scorecard + 密钥扫描 + 组件级 codecov。
4. 一步一台阶的栈式演进路径清晰可读：GUIAgent 单类循环 → tarko 事件流框架 → omni-tars 可组合 Agent，是研究 Agent 框架演化的好素材。

**风险/局限**
1. **已归档，项目死亡终结**：最后版本仅是安全修复，无官方继任指向（未确认），生产选型需谨慎；511 个未处理 issue。
2. 免费远程操作器服务已宣布停止（`docs/quick-start.md` L55），远程能力转向火山引擎商业 FaaS 模板，开源侧长期依赖自建 vLLM。
3. 嵌套双 monorepo（根 + `multimodal/` 独立 lock）+ 大量内部 fork 依赖（`@computer-use/*`）抬高了理解与维护成本。
4. GUI Agent 固有局限：纯截图闭环对分辨率/DPI/多屏（仅截主屏，`operator.ts` L52-70）敏感，坐标换算依赖 scaleFactor 因子，脆弱点集中在截屏与坐标层。

## 10. 总结评价

UI-TARS-desktop 是 2025 年 GUI Agent 浪潮的标志性开源项目：它把字节自研 UI-TARS 视觉模型的"截图-推理-动作"能力封装成了从 Electron 桌面应用到 CLI 再到 Agent 框架的完整栈，代码组织（Operator 抽象、事件流、类型安全 IPC）和工程化（安全扫描、组件化覆盖、e2e）均属开源 Agent 项目第一梯队，39.6k 星实至名归。但它已在 2026-10 归档，作为产品不宜再引入生产；作为**源码教材**价值反而更高——想理解"多模态模型如何驱动真实 GUI"的完整工程闭环（截屏缩放、动作解析、坐标换算、重试与人工接管），或想借鉴字节如何把一个模型 Demo 演进成 monorepo 级 Agent Stack 的过程，这个仓库仍然是最好的范本之一。
