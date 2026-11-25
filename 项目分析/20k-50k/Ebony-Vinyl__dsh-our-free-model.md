# Ebony-Vinyl/dsh-our-free-model 项目分析

> 星数 72,898（快照 2026-10-24；2026-11-14 gh api 复测 72,901）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：DeepSeek Harness（dsh）平台的"免费前沿模型"插件——把多条免费/共享上游包装成开箱即用的模型车道，宣称无需登录与 API Key 即可使用 DeepSeek V4.1 Flash、Kimi K3 等模型。

| 项目 | 值 |
| --- | --- |
| 名称 | Ebony-Vinyl/dsh-our-free-model（v2.8.3） |
| 星数 / Fork / Watch | 72,901 / 2,018 / 65（gh api，2026-11-14） |
| 语言 | JavaScript（ESM，零运行时依赖）+ TypeScript（vendored 渠道包）+ Go/WASM 二进制 |
| 许可证 | MIT（LICENSE） |
| 创建 / 最近 push | 2026-09-24 / 2026-11-13（gh api） |
| 贡献者 / Release | 30 人 / 16 个 release，最新 v2.8.3（2026-11-08） |
| 宿主 | dsh = DeepSeek Harness（`package.json` peerDependencies 全为 `@deepseek-ai/*`；`adapter/kernel.js` 回退 UA 为 `deepseek-harness/0.1.7`） |
| 代码规模 | 自有 JS 约 4.3 万行；vendor 渠道包 TS/JS 约 13.6 万行（148 个 TS 源文件） |

## 2. 项目是什么，解决什么问题

dsh（DeepSeek Harness）是 DeepSeek 官方的 AI 编程 harness/agent 平台（web、桌面 Electron/Tauri、TUI 多种形态，插件框架基于 cordis）。它本身只提供框架，模型接入需要用户自行配置 provider 与 API Key。本插件做的事情是：装上即在内置模型选择器中出现一组"免费"模型（README.md 开头；`index.js` 注册 LlmAdapter），用户零配置。

**"免费"的真实构成是五条车道**（README「上游是哪些源」与 `catalog/provenance.json`、`src/` 各车道模块互证）：

1. **免费车道**：OpenCode（opencode.ai）的匿名免费推理额度，凭据是写死的公共令牌 `Authorization: Bearer public`（`src/o2a/headers.js`、`src/upstream.js` 注释）。该车道存在，但上游按客户端指纹设闸，本插件用一整层"伪装成官方 OpenCode CLI"的请求整形来通过（详见第 5 节）。
2. **Kilo 渠道**：Kilo AI 公共网关（`ai-gateway.kilo.ai`）的 `isFree: true` 免费池，真无凭据（`src/kilo.js`），且插件如实声明上游可能记录 prompt。
3. **EAC 渠道（协付，桌面端专属）**：由仓库维护者运营的共享凭据池。凭据以 AES-256-GCM 密文密封在插件内（`src/vault-data.js`/`vault-anchor.js`），请求以 HMAC 签名发往维护者自建的 Cloudflare Worker / 宝塔 Node 网关，真实上游 Key 只存在网关环境变量中（`src/eac.js`、`worker/wrangler.toml`、`worker/README.md`）；访问以"GitHub 登录 + star 本仓库"为闸门（`worker/gateway-node.mjs` 的 star 复查逻辑）。模型白名单：deepseek-v4.1-flash、kimi-k2.6、kimi-k3、gpt-oss-20b、glm-5.3/5.3-flash（`worker/wrangler.toml` MODELS）。
4. **十个账号渠道**：vendored 的第三方插件（来自 gitee.com/iJetLi，revision 记录于 `catalog/provenance.json`），打包为 `vendor/channel-pack/`。用户登录自己在 LobsterAI（有道）、Qoder（阿里）、TRAE（字节）、Cline、Loomy（讯飞）、Raccoon（商汤）、MiniMax、ZCode（智谱）、Gemini 等产品的账号，插件以"模拟官方客户端"的方式调用这些产品的模型额度，并提供账号池轮换、每日积分自动领取、验证码自动化等能力（`vendor/channel-pack/src/account-pool.ts`、`auto-checkin.ts`、`captcha-carrier.ts`）。
5. **聚合渠道**：76 家提供商注册表（`src/providers/registry.json`），用户自带 Key，属常规聚合功能。

此外插件还附带大量增值面：OpenAI/Anthropic 兼容本地网关与局域网中继（`src/forward.js`）、降级链路由（`src/combos.js`）、省 Token 工具输出压缩（`src/token-saver/`）、用量看板、公告 feed + Ed25519 签名自升级 + 热重载（`src/updater.js`/`reload.js`）、插件中心市场（指向姊妹仓库 ofm-extensions）、独立本地服务（`packages/standalone`）与桌面壳（`packages/desktop`，Tauri）。README 也明确"不限量"的限定语：上游按 session/IP 计速率，短时打满即 429（README「已知边界」）。

README 声明与代码实现的匹配度总体较高：每条出网目标在 README「上游是哪些源」逐条列出，与代码核对一致；免责声明（`src/disclaimer.js`）明确定位为"个人学习与技术研究项目"。但 README 营销口径（"完全免费、不限量、无配置"）省略了两件代码里可见的事实：免费车道依赖对上游客户端指纹门的伪装与配额记账规避；EAC 渠道凭据来源不公开且专门做了抗分析混淆。

## 3. 技术栈与依赖

- **运行时**：Node ^22.19.0 / >=24，插件本体零 npm 依赖、无构建步骤（`package.json`；README 徽章）
- **宿主接缝**：peerDependencies 为 `@deepseek-ai/cordis`（插件框架）、`dsh-llm`、`dsh-credentials`、`dsh-commands`、`schemastery`，仅 `adapter/kernel.js` 允许 import（隔离由 `scripts/catalog-test.mjs` 强制）
- **vendored 组件**：channel-pack（TypeScript + esbuild 打包为 2.8MB `pack.js` + Qoder 认证 WASM）；magpie 引擎（yetone/magpie 的 Go 预编译二进制，`vendor/magpie/engine.json`）；dsh-std 插件清单 schema
- **基础设施**：Cloudflare Worker（EAC 网关，`worker/`）；Vite（standalone 管理界面）；Tauri（桌面壳）；GitHub Actions CI（5 个 workflow，动作全部 pin SHA）
- **自研设施**：Ed25519 签名清单自升级、SSE 公告推送、本地 JSON 存储（0600 权限）

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `index.js`（2,757 行） | Host 半身：适配器注册、探测循环、设置/用量存储、webServer 路由、转发端口、公告/升级/热重载接线 |
| `client.js` | 浏览器半身：手写 ModuleLoader bundle，无构建步骤 |
| `adapter/` | 内核接缝（唯一允许 import `@deepseek-ai/*` 的位置，51 行） |
| `src/`（60+ 模块） | 各车道出网线（upstream/kilo/eac/aggregate）、三种线协议编解码（stream/messages）、指纹整形（o2a/）、转发网关、token-saver、升级器等 |
| `vendor/channel-pack/` | 第三方账号渠道包（148 个 TS 源文件 + 2.8MB bundle + Qoder WASM） |
| `vendor/magpie/` | Go 预编译引擎（Claude 订阅车道） |
| `worker/` | EAC Cloudflare Worker 网关 + 自建 Node 网关 + 部署文档 |
| `packages/standalone`、`packages/desktop` | 不依赖 dsh 的独立服务 / Tauri 桌面壳 |
| `scripts/`（168 个文件） | 离线测试套件、探针取证脚本、构建与发布工具 |
| `docs/`（194 个文件） | 车道线缆逆向记录、发布说明、验收与验证记录、设计文档 |
| `feed/`、`catalog/`、`locale/` | 公告/升级清单、插件清单与出处记录、多语言文案 |

组织评述：单体插件 + vendor 子包 + 独立产品双形态。自有代码模块边界清晰（车道/编解码/网关/升级各一模块），文档密度罕见地高——docs/ 占近四分之一文件数，且 README 以逐 issue 验收表方式记录每个修复。

## 5. 核心架构与关键实现

### 5.1 免费车道：客户端指纹伪装与配额记账规避（`src/upstream.js` + `src/o2a/`）

免费车道直连 `https://opencode.ai/inference/…`，凭据为公共令牌 `Bearer public`。上游对匿名免费层设有多重客户端指纹闸（会话头、会话 id 形状、官方工具定义逐字比对、UA）。插件的做法（`docs/free-lane-wire.md` 完整记录了 MITM 抓官方客户端 + 逐字节单变量消融的逆向过程）：

- `src/o2a/` 是第三方逆向网关项目 **opencode2api v1.4.6**（jasonxu114514/opencode2api）的 JavaScript 移植，`src/o2a/SYNC.md` 给出逐文件对照表并随上游版本跟进；
- 每个请求把调用方全部工具改名为 `clint_<sha256[:12]>`，注入从官方 CLI 抓取的 13 工具逐字定义与官方系统提示（`src/o2a/official-tools.js`、`official-prompt.js`）；
- 会话 id 按"慢钟"铸造：时间戳回拨最多 30 天以落进上游接受窗口（`src/o2a/identity.js`，README 注明伪造时间戳原因是不合理时间会 429）；
- **配额记账规避**：上游按 project id × 出口 IP 记账，插件默认每请求铸造一枚全新 40-hex `x-opencode-project`（`src/upstream.js` `gatewayHeaders` 注释："a shared id is one shared bucket that arrives pre-burned"）；
- UA 中含 `deepseek-harness` 会被上游 403，插件把自家归属 UA 改写为 `dsh` 规避（`docs/free-lane-wire.md`、`src/http.js` `userAgentWith`）。

```js
// src/upstream.js（节选注释，L40-44）
// - **Per-request 40-hex `x-opencode-project`** replaces the shared `global`
//   bucket: free-tier quota is ledged per project id × egress, so a shared id
//   is one shared bucket that arrives pre-burned.
// - Session minting follows the reference's slow clock ... so a session's
//   date never leaves the window the upstream accepts
```

### 5.2 EAC 协付渠道：密封凭据 + 签名网关 + star 闸门（`src/eac.js`、`src/vault*.js`、`worker/`）

这是唯一涉及"共享凭据"的车道。凭据不落盘、不出进程：AES-256-GCM 密文分片存于 `src/vault-data.js` 与 `vault-anchor.js`（注释自称"对搜索引擎与字符串扫描而言仅为噪声"——刻意的抗静态分析设计），解锁按请求即时发生。请求发往维护者网关，携带 HMAC-SHA256 签名（`signSealedRequest`，`src/eac.js` L252-259）与 GitHub 授权令牌 `x-ofm-user`；网关（Cloudflare Worker 或宝塔自建）验签、验 star、限速后注入真实上游 Key 转发给"中继"（worker/README 显示中继为 New API 面板形态）。star 闸门每 12 小时复查，取消 star 即吊销令牌。

客观定性：该渠道在技术上就是一个**由仓库维护者运营的共享号池/共享凭据分发系统**，上游凭据的真实来源（自购 API、还是第三方产品的账号池）在代码与文档中均未披露，无法核实。插件侧文档明确提醒"请勿尝试绕过授权闸门"（README 免责声明节）。

### 5.3 账号渠道：第三方产品官方客户端模拟（`vendor/channel-pack/src/`）

vendored 包以用户自己的账号登录九类产品，但通信形态是模拟官方客户端而非走开放 API，证据充分：

- **Qoder 机器身份**：`qoder-machine.ts` 读取本机 Qoder IDE 的 `machine_token.json` 并回放 `Cosy-MachineToken/MachineType` 头族；文件头注释完整记录了通过抓包（pcapng + SSLKEYLOGFILE 解密）+ 消融实验逆向该协议的过程；`qoder-auth-wasm.wasm` 是从 Qoder IDE 中提取的认证 WASM（`scripts/extract-qoder-wasm.mjs`）。
- **验证码自动化**：`captcha-carrier.ts` 为 ZCode 渠道在 webview guest 或外挂 chromium 中铸造验证码 param，被上游 3007 拒绝时"当次回退"重产重发。
- **账号池与轮换**：`account-pool.ts` 支持同一 provider 多账号、按模型限流状态选号；`auto-checkin.ts` 自动领取每日积分；`queue-rotation.ts` 队列轮换。

### 5.4 适配器与本地网关

`src/adapter.js` 以鸭子类型实现结构性 LlmAdapter（不 import 内核，兼容 0.1.5/0.1.7 两版）；`src/stream.js` 把 chat/responses/messages 三种线协议归一为 harness StreamChunk；`src/forward.js` 提供本地 OpenAI 兼容监听（仅绑回环、crypto 生成 Key、timingSafeEqual 比对、0600 存储）与 Anthropic Messages 端点；`src/token-saver/` 在唯一收口 `payloadFor()` 上做幂等压缩且"失败开放"。

## 6. 构建与运行

```bash
npm test                  # 全部离线套件 + 清单一致性检查（不出网）
npm run typecheck         # tsc --noEmit
npm run start:standalone  # 独立本地服务（端口 18900，数据 ~/.our-free-model）
npm run manifest          # 重新生成发布清单（需 Ed25519 私钥，未签名清单被拒绝）
dsh plugin --profile web add /绝对路径/dsh-our-free-model   # dsh 插件安装
```

（来源：`package.json` scripts、README「开发」节。）无安装步骤、无运行时依赖；发布走 `scripts/build-manifest.mjs` + SHA-256 清单 + Ed25519 签名 + CI（`.github/workflows/prepare-release.yml`、`release-sign.yml`）。

## 7. 测试与工程质量

- `scripts/` 下 168 个脚本，其中数十个离线测试套件（sniff/projection/fingerprint/truncation/retry-safety/updater/feed/trust/sanitize 等），全部不出网、不消耗免费额度；`scripts/test-all.mjs` 统一调度，含 contributor/release 两级模式（`package.json`）。
- README 以逐 issue 验收表记录每个修复的验证方式，并明述做过**变异测试**（逐条还原旧行为验证断言会变红）；CI（`.github/workflows/test.yml`）在 push/PR 上强制离线套件，动作全部 pin commit SHA。
- 完整性链工程化程度高：清单 LF 归一化哈希、Ed25519 验签、staging 回读、防路径逃逸（`src/updater.js`）；`catalog/` 提供 manifest 0.15 + integrity + provenance 三件套。
- 观察到的短板：EAC 凭据密封、2.8MB vendored pack.js、预编译 Go 二进制与 WASM 均无法从源码直接审计；部分上游事实（指纹门行为）依赖持续逆向跟进，上游一旦变更即全线失效（`/zen/v1`→`/inference/v1` 与 v1→v2 指纹两次迁移已发生，`docs/free-lane-wire.md`）。

## 8. 社区与维护状态

- 极高活跃：创建于 2026-09-24，至 2026-11-13 每日仍有 push；16 个 release，v2.8.x 保持周更节奏（`docs/releases/` 逐版说明）。
- 174 个 open issue、30 名贡献者（gh api）；README 验收记录显示核心贡献高度集中于维护者本人。
- 增长背景（客观陈述）：72.9k 星产生于约 7 周（≈1,000+ 星/天），曾登 Trendshift 日榜第 1 与 JavaScript 周榜第 1（README 徽章，含 trendshift 图标）；README/docs 内嵌 17+ 个 QQ 交流群二维码（`docs/images/qq-group-*.png`），表明增长主要来自中文社区社群推广。值得注意的机制性事实：**EAC 渠道以"star 本仓库"为解锁闸门且每 12 小时复查**（`worker/README.md`），即访问最高价值车道与 star 行为直接挂钩，构成持续拉动星数的内生激励；星/叉比约 36:1，明显高于常见热门仓库。

## 9. 亮点与风险

**亮点**

1. 工程质量显著高于同类"白嫖网关"项目：零依赖纯 JS、离线测试体系、签名自升级、逐 issue 验收与变异测试文化（`scripts/`、README 验收表）。
2. 文档与事实披露意愿强：出网目标逐条列出（README「上游是哪些源」）、Kilo 记录 prompt 的风险如实转述、逆向过程完整留档（`docs/free-lane-wire.md`），与代码可互证。
3. 安全边界设计认真：转发仅绑回环、Key 0600 + timingSafeEqual、公告 HTML 白名单渲染（XSS 语料测试）、升级信任根 pin 公钥、信任围栏 fail-closed（`src/trust.js`、README「安全与隐私」）。
4. 多车道抽象统一：五种来源汇入同一适配器与网关口面，失败分类/降级链/恢复机制在车道间复用（`src/route/classify.js`、`src/combos.js`、`src/recovery.js`）。

**风险与局限（含服务条款与法律风险，客观陈述）**

1. **对上游服务条款的系统性规避**：免费车道以伪造官方客户端指纹（工具改名 + 官方提示注入 + 会话铸造）、每请求新铸 project id 规避按项目记账、改写 UA 隐藏 harness 身份等方式通过上游门禁；上游明确限定"free tier can only be used from within OpenCode"（`docs/free-lane-wire.md` 引述的 403 文案）。此类行为普遍违反上游服务条款，随时可能被封禁（历史上已两次被迫迁移）。
2. **EAC 共享凭据池来源不透明**：凭据密封且刻意抗分析，真实上游凭据来源未披露；网关以 star 换访问权，实质是凭据分发体系。若上游凭据非自购或涉及第三方账号资源，存在滥用第三方资源与凭据来源合法性的双重风险；星闸门亦构成对仓库星数的人为激励。
3. **第三方产品客户端模拟的合规风险**：账号渠道模拟 Qoder/TRAE/Loomy/ZCode 等官方客户端（含提取 IDE 内 WASM、回放机器指纹、自动化验证码与签到），违反对应产品条款的概率高，用户账号面临封禁；上游条款由用户自担（`src/disclaimer.js` 明确将责任推给使用者）。
4. **隐私与供应链**：prompt 经免费车道/EAC 发往第三方且 EAC 流量必经维护者网关；Kilo 免费池声明记录 prompt；插件含密封密文、2.8MB vendored bundle、预编译二进制与 WASM 等不可从源码审计的成分。README 另警告 npm 上的裸名 `dsh-our-free-model` 为第三方重打包（非本仓库发布）。

## 10. 总结评价

这是一个技术完成度极高、文档纪律罕见的"免费模型聚合网关"插件：它把 OpenCode 匿名免费层、Kilo 公共免费池、维护者自营的 EAC 共享号池与九类第三方产品的账号额度，统一包装进 DeepSeek Harness 的模型选择器，并在工程侧（测试、升级完整性、安全围栏）达到了许多正经基础设施项目都没有的水平。它的 7 周 72.9k 星由"零配置白嫖前沿模型"的需求、Trendshift 曝光与 QQ 社群推广共同驱动，而 star 解锁 EAC 的设计又内生地巩固了这一增长。客观地说：README 的"免费"声明与代码事实基本对得上，但"免费"的真实成本被转移到对上游服务条款的系统性规避、不透明的共享凭据池与对第三方产品的客户端模拟上——工程越精致，这些越是被稳定地规模化。适合谁：想研究 provider 插件架构、线协议适配、签名升级体系与离线测试文化的开发者，其源码是一份高质量的教科书；对想直接使用其"免费车道"的用户而言，需要清醒认识第 9 节所列的条款、隐私与封禁风险——本项目文档自身也反复传达了同一结论。
