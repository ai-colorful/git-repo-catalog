# linshenkx/prompt-optimizer 项目分析

> 星数 39500（快照 2026-10-24，gh api 复核 2026-11-14 实测 41447）｜主语言 Vue（GitHub 归类 TypeScript）｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：一款"用 AI 优化提示词"的纯前端 AI 工具——把粗糙的手写提示词交给大模型按模板重写成结构化提示词，支持 17+ 模型接入与 Web/Chrome 插件/桌面/MCP 四种使用形态。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | prompt-optimizer（提示词优化器） | README.zh-CN.md |
| 星数 | 41,447（2026-11-14 gh api；快照 39,500） | gh api repos/linshenkx/prompt-optimizer |
| 主语言 | TypeScript + Vue 3（UI 层 Vue，GitHub 语言统计 TS） | packages/*/package.json、gh api |
| 许可证 | AGPL-3.0-only | LICENSE（GitHub 标 NOASSERTION，因许可证文件为自定义全文） |
| 版本 | v2.15.2（2026-11-09 发布） | package.json、CHANGELOG.md |
| 最近活跃 | 最近 push 2026-11-13（前一天），数天一个 release | gh api pushed_at |
| 创建时间 | 2025-02-12（约 21 个月涨到 41k 星） | gh api created_at |
| 默认分支 | develop | gh api |

## 2. 项目是什么，解决什么问题

写好提示词是所有 LLM 使用者的共同痛点：普通用户写的提示词往往目标模糊、缺少角色设定与输出约束，导致模型回答"助手腔"严重。prompt-optimizer 的解法是"以提示词优化提示词"——用户输入原始提示词，选择一个已配置的模型，系统按内置优化模板（Mustache 渲染）生成一条"优化指令"发给该模型，让模型把原始提示词重写成带 Role/Profile/Skills/Rules/Workflows 结构的规范提示词（`packages/core/src/services/template/default-templates/optimize/general-optimize.ts`）。优化结果可继续迭代改进、可做分析、单结果评估与多结果对比评估（`packages/core/src/services/evaluation/`），并有 Arena 工作区把同一输入并排喂给多个文本/图像模型横向比较（CHANGELOG 2.15.0）。

功能面已从文本扩展到全链路提示词资产：系统/用户提示词双模式；文生图 T2I、图生图 I2I、多图生图（Gemini/Seedream/Grok 等图像模型，`packages/core/src/services/image/`、`docs/image-mode.md`）；上下文变量管理、多轮会话测试、Function Calling 工具调用；提示词收藏（版本历史、来源绑定）与 Prompt Garden 提示词库联动（README.zh-CN.md 核心特性、`packages/core/src/services/favorite/`）。

架构上强调"纯前端本地化"：所有数据（模型配置、API Key、历史、收藏）只存浏览器本地（IndexedDB，见 `packages/core/src/services/storage/dexieStorageProvider.ts`），请求由浏览器直连各 AI 服务商，不经中间服务器（README.zh-CN.md"安全架构"）。同时提供可选的自部署形态：Vercel/Cloudflare/Docker 部署时可加访问密码保护（`api/auth.js` Vercel serverless 校验 ACCESS_PASSWORD、`middleware.js`、`docker/`）。

值得一提的差异化能力是 MCP 集成：`packages/mcp-server` 把优化能力封装成 Model Context Protocol 服务器，暴露 optimize-user-prompt、optimize-system-prompt、iterate-prompt 三个工具，可直接挂进 Claude Desktop 等 MCP 宿主，让 AI 在对话里调用提示词优化器（`packages/mcp-server/src/index.ts`）。

## 3. 技术栈与依赖

| 层 | 技术 | 出处 |
|---|---|---|
| UI 框架 | Vue 3 + TypeScript + Pinia + Naive UI + TailwindCSS + CodeMirror 6 编辑器 | packages/ui/package.json |
| 构建 | Vite + tsup（core 双格式构建）+ pnpm 10.6.1 workspace | packages/core/package.json、pnpm-workspace.yaml |
| LLM SDK | openai、@anthropic-ai/sdk、@google/genai 官方 SDK，其余厂商走兼容层 | packages/core/package.json |
| 模板/校验 | mustache（优化模板渲染）、zod（参数校验）、jsonrepair（容错解析 LLM JSON 输出）、diff | packages/core/package.json |
| 本地存储 | dexie（IndexedDB 封装） | packages/core/package.json |
| 桌面端 | Electron + electron-builder（win/mac/linux 三平台）+ electron-updater 自动更新 | packages/desktop/package.json、main.js |
| 浏览器插件 | Chrome Manifest V3（minimum_chrome_version 88） | packages/extension/public/manifest.json |
| Web 增强 | vite-plugin-pwa + workbox-precaching（可安装 PWA） | packages/web/package.json |
| MCP/服务端 | @modelcontextprotocol/sdk + Express；Vercel serverless（api/auth.js）；Cloudflare Workers（wrangler.jsonc，纯静态资源发布） | packages/mcp-server、wrangler.jsonc |
| 测试 | Vitest（单元）+ Playwright（E2E，含 VCR 录制回放）+ node:test（工程脚本自检） | 各包 package.json、tests/e2e |

## 4. 目录结构与代码组织

pnpm monorepo，六个子包 + 一批工程支撑目录（顶层 `ls`、pnpm-workspace.yaml）：

| 目录 | 职责 |
|---|---|
| packages/core | 核心引擎：LLM 适配器、提示词优化服务、模板、存储、收藏、评估等，零 UI 依赖 |
| packages/ui | 共享 UI 层：全部界面组件、Pinia store、i18n（en-US/zh-CN/zh-TW）、路由 |
| packages/web | Web 端薄壳（复用 ui），产物可发 PWA / Cloudflare / Docker |
| packages/extension | Chrome 插件端薄壳（MV3） |
| packages/desktop | Electron 壳：main.js 加载 web 构建产物 + electron-updater |
| packages/mcp-server | MCP 协议服务器（3 个优化工具） |
| api/ + middleware.js | Vercel serverless 访问密码校验 |
| scripts/ | 20+ 自研工程脚本：分组合跑、e2e 智能选择、release 渠道管理、locale parity 检查等 |
| tests/e2e | Playwright E2E（含 e2e-vcr-guide.md 的录制回放体系） |
| docs/ + mkdocs/ | MkDocs 文档站源（部署/开发/用户指南），docs/developer/development.md 为开发主文档 |
| releases/ | 每版本双语（en/zh-CN）发布说明，CHANGELOG.md 作索引 |
| docker/ | docker-compose 与 nginx/supervisord 配置 |

组织方式评述：典型的"核心包 + 共享 UI 包 + 多端薄壳"分层，复用度高——三端共享同一套 ui 包界面（`packages/web/package.json` 仅依赖 @prompt-optimizer/ui + vue），core 保持纯逻辑可被 mcp-server 也能复用。工程化程度在同类开源工具里偏重：版本同步脚本（scripts/sync-versions.js）、双语 release notes 门禁（release.yml validate-release-notes）、运行时禁中文检查（scripts/check-no-chinese-runtime.mjs，保证硬编码文案不漏进运行时）。

## 5. 核心架构与关键实现

### 5.1 LLM Provider 适配器层（模板方法 + 注册表）

17+ 家模型厂商的接入统一抽象为适配器。基类 `AbstractTextProviderAdapter` 用模板方法模式固化公共校验，子类只实现 `doSendMessage` / `doSendMessageStream` 等抽象方法（`packages/core/src/services/llm/adapters/abstract-adapter.ts`）：

```ts
export abstract class AbstractTextProviderAdapter implements ITextProviderAdapter {
  public abstract getProvider(): TextProvider
  public abstract getModels(): TextModel[]
  protected abstract doSendMessage(messages: Message[], config: TextModelConfig): Promise<LLMResponse>
  protected abstract doSendMessageStream(messages: Message[], config: TextModelConfig,
    callbacks: StreamHandlers): Promise<void>
  protected abstract getParameterDefinitions(modelId: string): readonly ParameterDefinition[]
}
```

注册表 `TextAdapterRegistry` 在初始化时装载全部适配器（`registry.ts`）：OpenAI、OpenAI-Compatible（通用兼容端点）、Anthropic、Gemini、DeepSeek、SiliconFlow、智谱、DashScope、OpenRouter、ModelScope、Ollama（本地模型）、Magpie、MiniMax、Cloudflare、Grok、Chrome 内置模型（`chrome-built-in-adapter.ts`，Chrome 浏览器自带的 window.ai）、小米 MiMo 等。本地模型支持除 Ollama 外还有本机服务自动发现（`services/llm/local-discovery.ts`、CHANGELOG 2.14.0"本机服务发现与模型连接引导"）。新增厂商 = 新增一个适配器文件 + 注册一行，扩展点清晰。

### 5.2 提示词优化引擎：模板体系 + PromptService

优化本体是"模板驱动的元提示词"。内置模板按策略分类存放（`packages/core/src/services/template/default-templates/`）：optimize 下有 general-optimize（上述结构化 Role/Skills/Rules/Workflows 范式）、analytical-optimize（分析式）、output-format-optimize（输出格式约束）、soul-hermes/soul-openclaw-compose（两种"灵魂"组合式），且每条模板都有中英双语版本；另有 iterate（迭代改写）、evaluation / evaluation-rewrite / evaluation-structured-compare（评估与评估后重写）、variable-extraction / variable-value-generation（变量提取与取值）、image-optimize 等图像模板。`TemplateProcessor`（`services/template/processor.ts`）负责 Mustache 渲染与上下文注入。

`PromptService`（`services/prompt/service.ts`）编排整条链路：构造时注入 IModelManager / ILLMService / ITemplateManager / IHistoryManager 四个依赖并逐一检查；优化时按 DEFAULT_TEMPLATES（OPTIMIZE/ITERATE/TEST）选模板 → 渲染 → 经 ILLMService 流式调用 → 结果落历史。服务间全部依赖接口而非实现（types.ts 定义契约），是较干净的六边形架构实践。

### 5.3 存储抽象与多端一致性

存储层用 Provider 模式抹平差异（`packages/core/src/services/storage/`）：factory.ts 按运行环境选择 dexieStorageProvider（浏览器 IndexedDB）、localStorageProvider、fileStorageProvider（桌面端文件）或 memoryStorageProvider；另有 startup-safety-check.ts 做启动安全检查、preference/ 服务做全局设置的 CAS（compare-and-set，见 core 测试 test:gate 里的 `tests/unit/preference/compare-and-set.test.ts`）。桌面端与浏览器端通过 electron-proxy.ts 系列文件桥接 Electron IPC（如 `services/llm/electron-proxy.ts`、`services/data/electron-config.ts`），保证 core 同一份代码在三种宿主行为一致。

### 5.4 多端复用与 MCP

桌面壳极薄：`packages/desktop/main.js` 开发态 loadURL localhost:18181、生产态 loadFile webDistPath（直接加载 web 包构建产物），再加 electron-updater 自动更新。MCP 服务器（`packages/mcp-server/src/index.ts`）注册 optimize-user-prompt、optimize-system-prompt、iterate-prompt 三个工具，把 core 的优化链路暴露为任何 MCP 宿主可调用的能力——这是该项目区别于普通网页工具的生态位亮点。

## 6. 构建与运行

来自根 package.json 与 docs/developer/development.md，均未实际运行（规范要求静态阅读）：

```bash
pnpm install
pnpm dev              # Web 开发：构建 core/ui 并启动 web
pnpm dev:desktop      # 桌面开发：同时跑 web + electron
pnpm build            # build:core -> build:ui -> 并行 build:web/build:ext
pnpm build:desktop    # 完整桌面打包（electron-builder）
pnpm test             # 全 workspace Vitest 单测
pnpm test:e2e         # Playwright E2E
pnpm test:gate        # 合入门禁：repo 自检 + core/ui/web 分包门禁
pnpm lint             # 多包 lint + typecheck 串联
pnpm mcp:build        # 构建 MCP server
```

部署形态：静态产物（packages/web/dist）可直接发 Cloudflare Workers（wrangler.jsonc，SPA fallback）或 Vercel；`docker/docker-compose.yml` + Dockerfile 提供带访问密码的容器部署；桌面端走 tag 触发的 GitHub release（.github/workflows/release.yml）。

## 7. 测试与工程质量

- 单测规模：388 个测试文件（core 154、ui 225、web 5、mcp-server 4），Vitest 框架，core 侧覆盖 VCR mock、偏好 CAS、IndexedDB（fake-indexeddb）、参数兼容性等专项（packages/core/package.json test:gate 清单）。
- E2E：Playwright，tests/e2e 含 analysis/favorite-management/import-export 等场景，且实现了 E2E VCR 录制/回放模式（E2E_VCR_MODE=replay，tests/e2e/e2e-vcr-guide.md），避免真实调模型；另有 scripts/e2e-groups.js 分组与 smart-e2e 智能选择性执行。
- 工程自检脚本用 node:test 直测 scripts/ 本身（package-scripts.test.mjs、release-channel.test.mjs 等 9 个），并强制 check:locale（多语言键位对齐）与 check:no-chinese-runtime（运行时禁止硬编码中文）。
- CI：test.yml 覆盖 push/PR（develop/main/master）；release.yml 由 v*.*.* tag 触发，先 validate-release-notes 再构建桌面三平台并联动 docker.yml 发布镜像（.github/workflows/）。
- 观感：质量门禁体系（test:gate/test:gate:full）明显高于同类个人项目，代价是根 package.json 脚本数量庞大（60+ 条），认知成本偏高。

## 8. 社区与维护状态

- 活跃度：极活跃。gh api 显示最近 push 2026-11-13；CHANGELOG.md 显示 2026-10-25 至 11-09 半个月内发布 2.12.0/2.13.0/2.14.0/2.15.0/2.15.1/2.15.2 六个版本（另含 beta），发布节奏约每周 1-2 版，每版都有中英双语说明。
- 贡献者结构：33 名贡献者（gh api contributors），但高度集中——linshenkx 941 次贡献居绝对主导，第二名 hexart 66 次，其余多为个位数；open issues 仅 4 个（issue 处理极勤或以讨论/discussion 分流）。属"单主创 + 少量外围贡献"模式。
- 增长：2025-02 创建，2026-11 达 41.4k stars（约 21 个月），README 挂 Trendshift 趋势榜徽章与 Chrome 商店用户数徽章（README.zh-CN.md）；forks 4,654。star 时间线接口对本仓库不可用（gh api star+json 分页 404），精确增长曲线未确认，但从 Trendshift 徽章与半年 6 版的节奏推断有持续的社区曝光输入。
- 生态面：官网/在线优化器/Prompt Garden 提示词库/MkDocs 文档站/DeepWiki/ZRead 一应俱全（README.zh-CN.md），Docker 镜像 linshen/prompt-optimizer 已发布。

## 9. 亮点与风险

亮点：
1. 架构分层教科书级：core 纯逻辑 + ui 共享组件 + 三端薄壳，一份引擎服务 Web/插件/桌面/MCP 四种形态（packages/ 结构）。
2. Provider 抽象完整且克制：模板方法基类 + 注册表，17+ 厂商（含 Chrome 内置模型、Ollama 本地模型）统一接入，新增厂商成本极低（packages/core/src/services/llm/adapters/）。
3. 纯前端隐私设计到位：数据全本地 IndexedDB、直连服务商，可选部署形态再叠加密码保护，兼顾"开箱即用"与"自托管"（services/storage/、api/auth.js）。
4. 工程质量超出体量预期：388 个单测文件、VCR 式 E2E、双语 release 门禁、禁中文硬编码检查（tests/、scripts/、.github/workflows/）。
5. MCP 集成前瞻：把优化能力做成 MCP 工具接入 Claude Desktop，工具型产品生态卡位早（packages/mcp-server）。

风险/局限：
1. 优化质量强依赖被调用的模型本身与内置模板范式，模板同质化（都是结构化重写）可能让不同策略的收益差异有限（default-templates/optimize/）。
2. AGPL-3.0 对商用集成不友好，企业内部二次分发需谨慎评估（LICENSE）。
3. 贡献者高度单点：941/1019 的贡献占比意味着维护可持续性系于主创一人（gh api contributors）。
4. 发布节奏快、脚本面大（根 package.json 60+ 脚本、半月 6 版），普通贡献者跟进成本高；浏览器端部分功能受 MV3 CSP 限制需分流到 Web 版（CHANGELOG 2.13.0-beta.1 自述）。

## 10. 总结评价

这是中文开发者社区近两年最有代表性的 AI 前端工具之一：以"提示词优化提示词"这一简单可感知的核心价值切入，靠 monorepo 多端复用把一个网页工具铺成 Web/Chrome 插件/桌面/MCP 全形态产品，再以远超同类个人项目的工程门禁维持每周双版的迭代速度，21 个月冲到 41k 星属于产品与工程双轮驱动的自然结果。适合普通 AI 用户当日常提示词工作台（在线版零配置即用），适合想学"多端 monorepo + Provider 抽象 + 纯前端本地化"架构的开发者精读 packages/core 与 packages/ui，也适合作为 AGPL 语境下自托管提示词管理方案的候选。
