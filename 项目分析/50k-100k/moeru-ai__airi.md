# moeru-ai/airi 项目分析

> 星数 50625（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：开源版 Neuro-sama——基于 LLM 的虚拟角色 AI 伴侣（cyber waifu / AI VTuber），可在浏览器、桌面、手机上拥有一个能聊天、玩游戏、看屏幕的"数字生命"。

| 事实 | 值 | 出处 |
| --- | --- | --- |
| 名称 | moeru-ai/airi（Project AIRI / アイリ） | README.md |
| 星数 | 50625（元数据实查 51064，2026-11-04） | gh api |
| 主语言 | TypeScript（Vue 3 生态） | package.json / pnpm-workspace.yaml |
| 许可证 | MIT（Copyright 2024-PRESENT Neko Ayaka） | LICENSE |
| 创建时间 | 2024-12-01，约两年达成 50k 星 | gh api created_at |
| 最近推送 | 2026-11-03（分析前一天，极活跃） | gh api pushed_at |
| 最新 release | v0.14.1（2026-10-30），HEAD 版本号 0.15.0 | gh api / package.json |
| 贡献者 | 约 215 人（contributors 分页统计） | gh api |
| 规模 | 6317 个文件（不含 .git），pnpm monorepo | 源码统计 |

## 2. 项目是什么，解决什么问题

AIRI 的目标是复刻闭源的 AI 主播 Neuro-sama：一个能打游戏、聊天、看直播互动的虚拟存在（README.md "Re-creating Neuro-sama, a soul container of AI waifu"）。作者的观点是：Character.ai、SillyTavern 等方案只解决了"聊天"，而 Neuro-sama 展示了"边玩游戏边互动"的可能性但不开源，AIRI 要让每个人都能拥有自己的数字生命。

产品形态上是"模型无关 + 端形态多样"的伴侣容器：模型侧通过自研的 xsai SDK 接入 30+ 家 LLM 提供商（OpenAI/Claude/Gemini/DeepSeek/Qwen/Ollama 等，README 列表）；形象侧同时支持 VRM 与 Live2D 两种主流皮套格式（README Body 能力清单）。能力侧不止于聊天：能玩 Minecraft、Factorio（README Brain 清单），能听（浏览器/Discord 音频输入、客户端语音识别与说话检测）、能说（ElevenLabs/Azure/OpenAI TTS/本地 Kokoro）、能看（桌面端有 computer-use 与屏幕采集服务）。

技术口号是"从第一天起就构建在 Web 技术之上"（README）：WebGPU、WebAudio、Web Workers、WebAssembly、WebSocket。浏览器版可以直接访问（airi.moeru.ai，PWA），桌面版则可用 NVIDIA CUDA / Apple Metal 原生加速（借助 HuggingFace candle，README 说明）。项目还在早期快速演进，README 自述"仍处早期阶段"，另有官方组织 @proj-airi 承载衍生子项目（RAG、记忆系统、内嵌数据库、Live2D 工具等）。

## 3. 技术栈与依赖

- 语言/框架：TypeScript 6 + Vue 3.5 + Pinia 4 + Vue Router 5 + vue-i18n + UnoCSS（pnpm-workspace.yaml catalog）
- 构建：Vite 8、electron-vite、Turborepo、tsdown、pnpm workspace（约 90 个子包）；pnpm-workspace.yaml 以 catalog 统一版本
- 桌面：Electron 43 + electron-builder（apps/stage-tamagotchi）
- 移动：Capacitor（apps/stage-pocket，iOS + Android）
- 3D/2D 渲染：three ^0.185 + three-stdlib（VRM）、pixi/Live2D 组合（packages/stage-ui-live2d）；另有 MMD、Spine、立绘（tachie）渲染包
- 浏览器端推理：onnxruntime-web ^1.27（VAD、说话检测等，packages/stage-ui/src/stores/ai/models/vad.ts）
- 数据：localforage + unstorage(IndexedDB)（本地记忆/消息）、drizzle-orm、DuckDB WASM/pglite（README 路线图）、pgvector（packages/memory-pgvector）
- LLM SDK：xsai（自研，README 称"比 Vercel AI SDK 小得多"）
- 后端：server/ 下 apps/api + apps/auth（Better Auth + OIDC），PostgreSQL + Redis + Caddy，docker-compose 本地一键起，Railway 部署（server/README.md）
- 测试：Vitest（含浏览器模式 + Playwright）、vitest-plugin-fakemic（假麦克风）、vishot 场景截图
- 工程化：@antfu/eslint-config + moeru-lint + oxlint、knip、simple-git-hooks、Nix flake（flake.nix，含 NixOS FHS shell）

## 4. 目录结构与代码组织

顶层目录（均为 pnpm workspace 成员，package.json workspaces）：

| 目录 | 职责 |
| --- | --- |
| apps/ | 终端产品：stage-web（浏览器版）、stage-tamagotchi（Electron 桌面版）、stage-pocket（手机版）、stage-tamagotchi-kirie（变体）、ui-server-auth（登录 UI）、component-calling |
| packages/ | 约 90 个共享包：core-agent、core-character、stage-ui（共享 UI+全部 Pinia stores）、stage-ui-three/live2d/mmd/spine/tachie（渲染器）、provider-inference、server-runtime/sdk、memory-pgvector、drizzle-duckdb-wasm 等 |
| server/ | 托管后端：apps/api（资源 API）、apps/auth（Better Auth/OIDC）、docker-compose（server/README.md） |
| integrations/ | discord-bot、minecraft、telegram-bot、satori-bot、twitter-services、vscode |
| plugins/ | airi-plugin-bilibili-laplace、claude-code、game-chess、web-extension |
| engines/ | stage-tamagotchi-godot（Godot 引擎形态实验） |
| services/ | computer-use-mcp（MCP 服务） |
| deploy/、docs/、scripts/、nix/ | 部署、文档资源、脚本、Nix |

组织方式评述：重度模块化的 pnpm monorepo + Turborepo，按"终端 app（apps）— 共享产品包（packages）— 集成（integrations/plugins）— 后端（server）"分层。README 的 mermaid 图明确了依赖方向：三个 Stage 前端共享 stage-ui 与渲染器包，桌面版额外内嵌 server-runtime 作为"channel"供 Pocket 无线连接，集成机器人通过 server-sdk 对接。同名产品逻辑（聊天、记忆、渲染）只在包里写一份、多端复用，是本仓最核心的组织原则。

## 5. 核心架构与关键实现

### 5.1 多端形态：一份逻辑，四种壳

- 浏览器（apps/stage-web/src：pages/、components/、workers/，PWA）
- 桌面 Electron（apps/stage-tamagotchi/src：main/ + preload/ + renderer/ 三进程结构）。main 进程服务相当丰富（apps/stage-tamagotchi/src/main/services/airi/）：channel-server（供 Pocket 的安全 WebSocket）、computer-use（让角色操作电脑）、apple-vision / apple-speech-transcription（macOS 原生能力）、home-assistant、godot-stage、http-server 等
- 移动（apps/stage-pocket：capacitor.config.ts + android/ + ios/）
- Godot 引擎（engines/stage-tamagotchi-godot，实验性）

桌面与手机之间的联动方式：tamagotchi 内置 channel-server，pocket 通过（可加密的）WebSocket 连接，README 明确写了"无线模式需 sudo pnpm dev:tamagotchi 以启用 secure websocket"。

### 5.2 Agent 状态与记忆层（本次分析重点）

分层清晰，接口先行：

- **端口契约**（packages/core-agent/src/contracts/）：AgentContextPort（ingest/snapshot/reset）、AgentLLMPort（stream：按 provider 投影上下文并接管请求生命周期）、SessionPort、StreamPort。core-agent/src/agents/ 下有 spark-command（指挥型）与 spark-notify（事件驱动，含 observer/reaction 插件）两类 agent 实现。
- **上下文组装与压缩**（packages/core-agent/src/messages/compaction.ts）：对话历史按"turn"切分投影，超过 recentTurnLimit 时把旧轮次折叠成 summary 段，支持自定义 summarizeCompactedHistory 回调：

```ts
return {
  type: 'history-block',
  compacted: true,
  items: [{
    type: 'summary',
    text: summarizeCompactedHistory?.({...})
      ?? `Compacted ${removedTurnCount} older turns with paired reactions.`,
    fromTurnIndex: ...,
    toTurnIndex: ...,
  }],
```

- **UI 状态层**（packages/stage-ui/src/stores/）：Pinia 组织为 ai/chat-llm（llm.ts、streaming-control.ts、tool-resolver.ts、toolset-prompts.ts 等，负责流控与工具调用）、chat/（context-store.ts、session-store.ts、data-store.ts、stream-store.ts、prompt-recipe.ts 等）。其中 stores/chat/context-store.ts 把 core-agent 的 createContextRegistry 包成 Pinia store，按 sourceKey 分桶维护活跃上下文，历史上限 400 条，供 devtools 与 UI 消费。
- **长期记忆（端上）**（packages/stage-ui/src/database/repos/memory.repo.ts）：localforage（IndexedDB）实现，条目用 UUIDv7，带 tombstone 删除标记以便多设备同步；用 Web Locks API 做跨窗口写串行化：

```ts
function write<T>(run: () => Promise<T>) {
  return navigator.locks.request('airi:memory', run)
}
```

  记忆按 persona 归属（角色卡上的事实）或 general（全局事实），"general 记忆不可被单次 run 遗忘"。database/storage.ts 用 unstorage 挂载 local（airi-local）与 outbox（airi-sync-queue）两个 IndexedDB 空间，后者是离线同步队列。
- **记忆（服务端）**：packages/memory-pgvector（drizzle-orm + postgres + pgvector，经 server-sdk 通信），README 路线图中的 Memory Alaya 仍为 WIP。

### 5.3 VRM / Live2D 渲染

- VRM：packages/stage-ui-three/src/components/Model/VRMModel.vue + vrm-instance-cache.ts（实例缓存），src/stores/ 有 camera、model-store、view-control；README 列出自动眨眼、自动视线、待机眼球动画均已支持。
- Live2D：packages/stage-ui-live2d/src/components/scenes/live2d/（Canvas.vue、Model.vue、presence-bubble.vue），composables 拆得很细且带测试：eye-tracking.ts、focus-bias.ts、head-anchor.ts、beat-sync.ts（节拍同步）、fit-model.ts。
- 其他形态：stage-ui-mmd、stage-ui-spine、stage-ui-tachie，以及 model-driver-lipsync / model-driver-mediapipe / motion-driver-magic 等驱动包（packages/ 目录清单）。

## 6. 构建与运行

来自根 package.json scripts 与 README：

```shell
pnpm i                      # 安装（postinstall 会先构建各包）
pnpm dev                    # 浏览器版（stage-web，airi.moeru.ai）
pnpm dev:tamagotchi         # Electron 桌面版
pnpm dev:pocket:ios --target "<模拟器名>"   # 移动版 iOS（另有 android）
pnpm dev:backend            # docker compose 起 API+Auth+PG+Redis+Caddy（server/）
pnpm build                  # turbo run build（packages/apps/server）
pnpm test                   # vitest --coverage
pnpm typecheck              # turbo run typecheck
pnpm lint                   # moeru-lint .
nix run github:moeru-ai/airi   # Nix 直接跑桌面版
```

安装渠道：winget install MoeruAI.AIRI、scoop（自建 bucket/）、brew install --cask airi（README）。版本发布用 bumpp（README Publish 节）。

## 7. 测试与工程质量

- 测试规模：全仓 957 个 *.test.ts（含大量 *.browser.test.ts 浏览器模式测试），覆盖从 core-agent 的 compaction/projection 单测到 stores 的浏览器模式测试（如 stores/chat/session-store.browser.test.ts）与桌面端 smoke（tamagotchi-linux-window-smoke workflow）。
- CI（.github/workflows/，共 35 个工作流）：ci.yml（含 paths-ignore 策略与 required-checks 兼容设计，注释写明了取舍）；release-* 覆盖 Windows/macOS 桌面、Steam、iOS、Android、Docker、VSIX；crowdin-* 三件套管理翻译；deploy-cloudflare-workers* 系列部署预览环境；还有 copilot PR 分诊、nix hash 自动更新、sponsors SVG 生成等自动化。
- 代码质量观察：架构注释密度高（如 compaction.ts 的 JSDoc）、接口契约文件独立成目录、knip 查死代码、eslint suppressions 有专职清理脚本（lint:prune-suppressions）、AGENTS.md/CLAUDE.md 双份 agent 协作规范。

## 8. 社区与维护状态

- 活跃度：created 2024-12-01，pushed 2026-11-03（分析前一天），51k 星 / 5.1k forks / 265 open issues（gh api）。约两年从零到 50k+，README 展示 Trendshift"周榜第一"徽章，增长动能强。
- Release 节奏：v0.14.1 发布于 2026-10-30，HEAD 已进 0.15.0；DevLog 博客（README 列出至 2026.03）显示持续的月度级进展披露。
- 贡献者结构：约 215 名贡献者（gh api 分页统计）；有 OpenCollective/Patreon/Ko-fi 赞助（README Supporters），并有 302.AI、AIHubMix 等商业赞助标记（README）。
- 国际化：Crowdin 平台 + packages/i18n，README 有中/日/俄/越/法/韩六语言版本。
- 社区渠道：Discord、X、Telegram、微信群、QQ 群（README 徽章区）。README 专门警告"官方从未发行任何加密货币/代币"，侧面说明项目影响力大到出现仿冒。

## 9. 亮点与风险

亮点：

1. 多端形态完整且同构：浏览器 PWA、Electron、Capacitor 移动端、甚至 Godot 实验，一套共享包支撑四类端（apps/、packages/、engines/）。
2. Agent 状态与记忆设计有真实工程深度：core-agent 端口契约 + 上下文压缩（compaction.ts）+ Web Locks 串行化的跨窗口记忆 + tombstone 同步 + outbox 离线队列（packages/core-agent/、packages/stage-ui/src/database/）。
3. 桌面端"本体感"服务密度高：computer-use、屏幕采集、macOS 原生视觉/语音、Home Assistant、供手机连接的 channel-server（apps/stage-tamagotchi/src/main/services/airi/）。
4. 工程化水平在同类开源 AI VTuber 项目中明显领先：957 个测试文件、35 个 CI 工作流、多平台自动发版、Nix 支持、knip/oxlint 等现代工具链。
5. 供应商中立：自研 xsai + provider-inference 包接入 30+ 模型服务商，用户不被锁定（README、packages/provider-inference/src/providers/cloud/）。

风险/局限：

1. 迭代太快导致面铺得很开：Godot、Spine、MMD、Steam 版、多游戏集成并行推进，README 自述早期阶段，路线图多项（Memory Alaya、浏览器 WebGPU 推理）仍 WIP，稳定性和文档可能滞后。
2. 重度依赖 pnpm workspace 内部包（约 90 个），外部贡献者与二次开发者理解成本高；部分包为 private 不发布。
3. 记忆系统端上以 IndexedDB/localforage 为主，跨设备同步机制（tombstone + outbox）尚在演进，长期记忆的检索质量（向量召回仅在 memory-pgvector/服务端）未完全打通（基于 memory.repo.ts 与 README 路线图推断）。
4. 项目热度和仿冒风险并存：README 已辟谣虚假代币，用户需通过官方渠道（winget/brew/GitHub Release）安装。

## 10. 总结评价

AIRI 是目前开源 AI VTuber / AI 伴侣赛道工程化程度最高、端形态最全的项目之一：它不满足于"能聊天"，而是把 LLM agent、语音链路（VAD/STT/TTS）、VRM/Live2D 渲染、游戏操控、桌面感知与多端同步当成一个完整系统来做，并用约 90 个 workspace 包和近千个测试文件维持这个复杂度的可演进性。星数两年破 50k，靠的是 Neuro-sama 这一精准赛道的空白（闭源）加上持续高产的 DevLog 与社区运营。适合三类人：想要可自托管的 AI 伴侣终端用户（直接装桌面/浏览器版）、做 AI VTuber 或具身 agent 的研究者与工程师（重点读 core-agent、stage-ui stores 与 memory 层）、以及想学习大型 Vue/Electron monorepo 工程实践的前端开发者。想快速理解其架构，建议按 core-agent/src/contracts → packages/stage-ui/src/stores/chat + database/repos/memory.repo.ts → packages/stage-ui-three 与 stage-ui-live2d 的顺序阅读。
