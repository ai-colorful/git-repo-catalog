# SillyTavern/SillyTavern 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：面向"高级玩家"的本地自托管 LLM 角色扮演前端——把几十种文本/图像/语音生成后端统一到一个可深度定制的聊天界面里。

| 项 | 值 |
|---|---|
| 名称 | SillyTavern/SillyTavern |
| 星数 | 39500（快照）；gh api 实测 35894（2026-11-21，见第 8 章） |
| 语言 | JavaScript（纯 JS，无前端框架） |
| 版本 | 1.19.1（package.json） |
| 许可证 | AGPL-3.0（LICENSE / package.json） |
| Node 要求 | >= 20（package.json engines），ESM（"type": "module"） |
| 创建时间 | 2023-02-09（gh api），TavernAI 1.2.8 分叉 |
| 默认分支 | release（gh api） |
| 最近 push | 2026-11-20（gh api，昨日） |
| 代码量 | 后端 src/ 约 3.36 万行 + 前端 scripts 约 4.0 万行（wc 统计） |

## 2. 项目是什么，解决什么问题

SillyTavern（简称 ST）是一个本地安装的 Node.js Web 应用，让用户通过浏览器与文本生成 LLM、图像生成引擎和 TTS 模型交互。它 2023 年 2 月从 TavernAI 1.2.8（MIT，作者 Humi）分叉而来，三年独立发展后已成为 AI 角色扮演/聊天爱好者社区的事实标准前端之一（`.github/readme.md`「What is SillyTavern?」章节、致谢列表）。

它解决的核心问题是"碎片化"：本地推理栈（KoboldCpp、text-generation-webui、llama.cpp、Ollama）与商业 API（OpenAI、Claude、Gemini 等）各有各的界面和参数体系，ST 把它们收进一个统一界面——26 个 chat-completion 来源 + 15 个 text-completion 后端 + Horde/NovelAI（`public/scripts/openai.js`、`public/scripts/textgen-settings.js` 的枚举），API 密钥体系里登记了 66 个不同服务的凭证槽位（`public/scripts/secrets.js` SECRET_KEYS）。

第二个核心是"角色卡"生态：角色以 PNG 图片内嵌元数据的形式分发，实现 Character Card V2 规范并前瞻性支持 V3（`src/character-card-parser.js`），兼容 .charx 打包格式（`src/charx.js`）与 BYAF 格式（`src/byaf.js`）。用户可从社区导入角色卡、世界书（lorebook）、预设主题，形成完整的"内容包"流转体系。

第三是极致的提示词控制力：世界书扫描引擎（约 6400 行）、作者注释、指令模式、群聊、STscript 脚本语言（带闭包/作用域/断点的解释器，`public/scripts/slash-commands/` 27 个文件）、正则脚本、宏系统等。README 明确其愿景是"给用户尽可能多的提示词控制权，陡峭的学习曲线也是乐趣的一部分"（`.github/readme.md`）。

项目自我定位为"激情驱动的社区项目"，不提供任何在线托管服务、不做用户数据跟踪（`.github/readme.md` Our Vision），所有服务端逻辑都跑在用户自己的机器上。

## 3. 技术栈与依赖

- **后端**：Node.js >= 20（ESM），Express 4（`package.json`）。安全中间件：helmet、csrf-sync、rate-limiter-flexible、cookie-session；上传 multer；压缩 compression。HTTP 客户端 node-fetch + proxy-agent（代理支持）。
- **前端**：无框架的 jQuery 老栈（devDeps 中 @types/jquery、@types/jquery-ui、select2、toastr），配 morphdom（DOM diff）、localforage（IndexedDB 存储）、handlebars 模板（`public/scripts/templates/`）、showdown（Markdown）、DOMPurify（XSS 清洗，`package.json`）。
- **Tokenizer**：tiktoken（OpenAI 系）+ @agnai/sentencepiece-js / web-tokenizers（本地加载 llama/mistral/gemma/jamba/yi/nerdstash 等 .model 文件，`src/tokenizers/`）+ sillytavern-transformers（fork 的 transformers.js，用于本地向量化）。
- **图像处理**：Jimp v1 全家桶（头像缩略图、PNG 元数据）；PNG tEXt 块读写用 png-chunks-extract / png-chunk-text（`src/character-card-parser.js`）。
- **存储**：文件系统为主（node-persist、write-file-atomic），向量化检索用 vectra（本地向量库），无任何数据库依赖。
- **工具链**：webpack（打包前端 lib 与扩展服务）、eslint 8 + jsdoc 插件、TypeScript 仅用于类型检查（jsconfig/index.d.ts）；测试 jest + playwright。
- **分发**：Docker（node:lts-alpine3.23，`Dockerfile`）、npm bin 全局安装（`src/server-global.js`）、甚至支持 deno/bun 启动（package.json scripts `start:deno`/`start:bun`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `server.js`（17 行） | 唯一入口：解析 CLI → 设 DATA_ROOT → 动态 import server-main |
| `src/` | Node.js 后端：33,642 行 JS |
| `src/endpoints/` | 45 个 API 路由文件（characters/worldinfo/openai/anthropic/google/stable-diffusion/…）+ `backends/`（chat-completions 2995 行、text-completions、kobold、google-models） |
| `src/middleware/` | 10 个中间件（whitelist、basicAuth、corsProxy、hostWhitelist、userCss 等） |
| `src/tokenizers/` | sentencepiece 模型文件 + 分词服务 |
| `src/vectors/` | 8 个向量检索后端适配（openai/cohere/google/llamacpp/nomicai/ollama/vllm/extras） |
| `src/users.js`（1219 行） | 多用户存储、迁移、登录（每用户独立 data 目录） |
| `src/character-card-parser.js` / `charx.js` / `byaf.js` | 角色卡三种格式的解析 |
| `public/` | 前端：`script.js`（12,599 行主文件）+ `scripts/`（82 个模块，40,097 行） |
| `public/scripts/extensions/` | 16 个内置扩展目录 + `third-party/`（用户安装） |
| `public/scripts/slash-commands/` | STscript 脚本语言的完整解释器（27 个文件） |
| `public/locales/` | 18 个 UI 语言包 |
| `tests/` | jest 单测（约 20 个 *.test.js）+ playwright e2e（16 个用例） |
| `default/` | 出厂配置与内容脚手架（config.yaml、默认角色 Seraphina、预设、主题） |
| `data/` | 运行时用户数据根（按用户名分目录，克隆时为空） |
| `docker/` | Dockerfile 附属：entrypoint、compose、build-lib |

组织方式评述：典型的"单体前端 + 单体后端"双大文件结构——后端靠 endpoints/ 目录按资源拆分尚属清晰，前端则以 `script.js`（1.26 万行）+ `scripts/*.js` 的扁平模块划分，模块间大量直接 import 而非事件解耦；ESM + JSDoc 类型注解（`index.d.ts`、各文件 @param）是全项目的类型化策略。

## 5. 核心架构与关键实现

### 5.1 角色卡系统（PNG 内嵌元数据，Character Card V2/V3）

`src/character-card-parser.js`（98 行）直接操作 PNG 二进制块，把角色 JSON 以 base64 写入 `tEXt` 块；读卡时 ccv3（V3 规范）优先于 chara（V2）：

```js
export const write = (image, data) => {
    const chunks = extract(new Uint8Array(image));
    // ...移除旧 chara/ccv3 块...
    const base64EncodedData = Buffer.from(data, 'utf8').toString('base64');
    chunks.splice(-1, 0, PNGtext.encode('chara', base64EncodedData));
    try {  // 同时尝试写入 v3 块
        const v3Data = JSON.parse(data);
        v3Data.spec = 'chara_card_v3';
        v3Data.spec_version = '3.0';
        chunks.splice(-1, 0, PNGtext.encode('ccv3', Buffer.from(JSON.stringify(v3Data), 'utf8').toString('base64')));
    } catch (error) { /* 忽略非 JSON 数据 */ }
    return Buffer.from(encode(chunks));
};
```

（write/read 函数，`src/character-card-parser.js`。）双规范兼容让 ST 能"写 V2 读 V3"，社区角色卡迁移成本极低。`.charx`（`src/charx.js`，405 行）是新增的 zip 容器格式，可携带资产附件；BYAF（`src/byaf.js`，449 行）是第三种互导格式。角色 CRUD 在 `src/endpoints/characters.js`（1688 行，13 个路由：create/rename/edit/edit-attribute/delete/all/get/chats/import/duplicate/export/merge-attributes）。

### 5.2 世界书 / Lorebook 引擎

`public/scripts/world-info.js`（6,408 行）是 ST 的"记忆注入"核心：条目按关键词触发，支持 4 种组合逻辑（`world_info_logic`：AND_ANY/NOT_ALL/NOT_ANY/AND_ALL）与递归扫描（scan_state：INITIAL/RECURSION/MIN_ACTIVATIONS），即"触发一个条目后其内容再参与下一轮关键词匹配"。入口 `getWorldInfoPrompt(chat, maxContext, isDryRun, globalScanData)`（world-info.js:892）。服务端配套 `src/endpoints/worldinfo.js`（list/get/delete/import/edit 五路由），世界书以 JSON 落盘在用户目录 lorebooks/ 下。

### 5.3 多后端模型接入矩阵

前端枚举 26 个 chat-completion 来源（`public/scripts/openai.js:177-204`）：OPENAI、CLAUDE、OPENROUTER、AI21、MAKERSUITE、VERTEXAI、MISTRALAI、CUSTOM、COHERE、PERPLEXITY、GROQ、ELECTRONHUB、CHUTES、NANOGPT、DEEPSEEK、AIMLAPI、XAI、POLLINATIONS、MOONSHOT、FIREWORKS、COMETAPI、AZURE_OPENAI、ZAI、SILICONFLOW、WORKERS_AI、MINIMAX；text-completion 侧 15 个（`public/scripts/textgen-settings.js:31-47`：ooba/mancer/vllm/aphrodite/tabby/koboldcpp/togetherai/llamacpp/ollama/infermaticai/dreamgen/openrouter/featherless/huggingface/generic）。后端代理适配集中在 `src/endpoints/backends/chat-completions.js`（2995 行）与 `src/endpoints/anthropic.js`、`google.js`、`openai.js`、`openrouter.js` 等；`src/prompt-converters.js` 负责不同供应商间的消息格式互转（如 Claude 的 system 前置、多模态块转换）。66 个 SECRET_KEYS（`public/scripts/secrets.js:19-88`）覆盖文本/图像/TTS/翻译/搜索全部服务商，密钥只存服务端用户目录 secrets.json，前端仅见掩码。

### 5.4 扩展系统与 STscript

扩展走"git 拉取 + manifest 声明"模式：服务端 `/api/extensions/install` 用 git clone 第三方仓库到 `public/scripts/extensions/third-party/`（`src/endpoints/extensions.js` 提供 install/update/branches/switch/move/version/delete/discover 七个路由），前端按 `manifest.json`（display_name/js/css/hooks.activate 等字段，示例 `public/scripts/extensions/memory/manifest.json`）加载。内置 16 个扩展：expressions（表情包）、memory（摘要记忆）、translate、tts、vectors、stable-diffusion、connection-manager、regex、quick-reply 等。STscript（`public/scripts/slash-commands/`）则是项目自研的宏脚本语言，27 个文件实现了词法器、解析器、闭包、作用域、断点控制器——相当于给聊天界面内嵌了一门编程语言，是"Power Users"定位的直接体现。

## 6. 构建与运行

```bash
npm install          # 安装依赖
npm run start        # node server.js（server.js → src/server-main.js）
npm run start:no-csrf # 关闭 CSRF（仅调试）
npm run lint         # eslint src/ public/ 根目录
cd tests && npm install
npm run test:unit    # jest 单测
npm run test:e2e     # playwright e2e（连 http://127.0.0.1:8000）
```

（来源：`package.json` scripts、`tests/package.json`。）配置通过 `default/config.yaml` 首次启动复制到 `data/`：监听地址/端口、白名单模式（默认仅 127.0.0.1/::1）、basicAuth、SSL、CORS（`default/config.yaml`）。Docker：`docker-compose.yml` + 健康检查心跳文件（config.yaml 的 heartbeatInterval，`Dockerfile`）。移动端可经 Termux 安装，也有 Electron 打包（`src/electron/`、package.json `start:electron`）。

## 7. 测试与工程质量

- **单测**：tests/ 下约 20 个 jest 用例，覆盖角色卡格式（byaf-import、charx-assets、tavern-card-validator）、prompt-converters、util、config-init、mock-server 等（`tests/` 目录清单）。
- **e2e**：16 个 playwright 用例（`tests/frontend/`），集中在两个新近重构点——宏引擎（MacroLexer/Parser/Registry/StoryString 6 个用例）与世界书/快速回复交互；视频与截图仅在失败时录制（`tests/playwright.config.js`）。
- **CI**：`.github/workflows/pr-checks.yml` 在 PR 上跑 ESLint + 单测（Node 24）；`docker-publish.yml` 每次 release 构建 ghcr 镜像且每日 00:00 UTC 构建 staging 镜像；另有 issues/pr 自动标签、冲突检查、stale 关闭等 8 个自动化工作流（`/.github/workflows/`）。
- **质量观察**：安全意识强——SSRF 防护（`src/private-request-filter.js`）、CSRF 同步、host 校验、文件名校验中间件、白名单默认开启；前端无框架约束导致 script.js 巨石化（1.26 万行），是最大的技术债；不过近期代码明显在模块化（macros/、slash-commands/、templates/ 等目录化重构）。

## 8. 社区与维护状态

- **活跃度**：gh api 显示 pushed_at 2026-11-20（分析前一日仍在提交），open issues 543，forks 6,740。星数快照 39500（2026-10-24），gh api 实测 35894（2026-11-21）——差值原因未确认（可能为统计口径差异），两数均在 20k-50k 档。
- **发版节奏**：92 个稳定 release（1.0.4，2023-03-14 → 1.19.1，2026-11-19），约每 1-3 个月一个大版本（1.14.0 2025-11 → 1.15.0 2025-12 → 1.16.0 2026-02 → 1.17.0 2026-03 → 1.18.0 2026-05 → 1.19.0 2026-09 → 1.19.1 2026-11，gh api releases）。最新版本在分析前两天刚发布。
- **贡献者**：contributors API 翻到第 4 页（约 300-400 人），README 称"300+ 贡献者"（`.github/readme.md`）；首位贡献者 7,189 次提交（gh api contributors），核心维护者为 README 署名的 cohee / RossAscends / Wolfsblvt。
- **治理形态**：无公司背书的社区项目；支持渠道为 Discord（discord.gg/sillytavern）、Reddit r/SillyTavernAI 与文档站 docs.sillytavern.app（`.github/readme.md`）；CONTRIBUTING.md（85 行）要求按模板提交 issue/PR。第三方扩展生态靠 `third-party/` 目录 + git 安装机制自组织，未做中心化审核（未确认是否有扩展清单审核流程）。

## 9. 亮点与风险

**亮点**
1. 模型接入广度业界罕见：26 chat 源 + 15 text 后端 + 66 密钥槽位 + 8 向量后端（枚举见第 5.3 节），"统一界面"定位兑现扎实。
2. 角色卡规范实现深入：V2/V3 双写、charx/BYAF 多格式互导，与社区内容生态（PNG 卡片分享）深度绑定（`src/character-card-parser.js`、`src/charx.js`）。
3. 世界书 + STscript + 宏构成完整的高级提示词工程工具链，权力下放理念贯彻到代码（world-info.js、slash-commands/）。
4. 安全工程在同类"本地跑的 AI 前端"中突出：CSRF/白名单/SSRF 过滤/host 校验/密钥服务端隔离。
5. 维护极其活跃：三年 92 个 release、日级提交、成熟 CI 与 issue 自动化。

**风险/局限**
1. 前端 jQuery 巨石（script.js 12,599 行 + 40k 行模块），无类型系统兜底（仅 JSDoc），新人上手与重构成本高。
2. body-parser 上限 500mb、扩展经 git 任意 URL 安装，公网暴露配置下攻击面大（项目默认白名单已部分缓解，`src/server-main.js`、`src/endpoints/extensions.js`）。
3. AGPL-3.0 对商用二次分发较严格；无在线服务形态，官方明确不提供托管。
4. 版本矩阵管理复杂：26+ 后端 API 各自漂移，历史上有上游接口变更导致功能失效的固有风险（README 未承诺 SLA；此为结构性局限，具体事件未确认）。

## 10. 总结评价

SillyTavern 是"社区激情项目长成基础设施"的典型样本：从 TavernAI 1.2.8 分叉起步，靠三年高强度迭代把角色扮演 LLM 前端做成了功能密度最高、后端覆盖最广的瑞士军刀，星数与 Discord 生态互为放大器。代码层面前端偏老的 jQuery 巨石架构与后端清晰的 Express 分层形成反差，但近期宏引擎/脚本语言的目录化重构显示了现代化的明确方向。适合三类人：想要本地、可定制、不依赖任何托管的 LLM 聊天重度用户；研究提示词工程（世界书、预设、宏、脚本）的实践者，其 world-info.js 与 openai.js 是极好的真实场景教材；以及想学习"如何把 26 种异构 LLM API 适配进统一抽象"的后端工程师。想读源码建议从 `src/server-main.js`（启动装配）→ `src/endpoints/characters.js`（领域核心）→ `public/scripts/world-info.js`（最复杂的单模块）的顺序切入。
