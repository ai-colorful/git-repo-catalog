# pbakaus/impeccable 项目分析

> 星数 85307（快照 2026-10-24）｜主语言 Rust｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：给 AI 编码 agent 用的"设计词汇表"——24 个斜杠命令引导 agent 做出有品味的前端设计，配一个 63 条确定性规则的"AI slop 检测器"（纯 Rust 引擎，无需 LLM/API key），在 agent 编辑、CI、浏览器三个场景拦截千篇一律的 AI 生成 UI。

| 项目 | 内容 |
|---|---|
| 名称 | pbakaus/impeccable（官网 impeccable.style） |
| 星数 | 85307 |
| 语言 | Rust（引擎）+ Markdown（skill 剧本）+ JS/TS（构建与测试） |
| 许可证 | Apache-2.0（LICENSE） |
| 创建时间 | 2025-11-16（gh api） |
| 最近推送 | 2026-10-30（gh api，分析前一天） |
| 最新 release | cli-v4.8.0，2026-10-26（gh api） |
| 作者 | Paul Bakaus（jQuery UI 作者、前 Google DevTools/AMP 成员，README 署名 + 个人主页链接） |
| Fork / Issue | 5703 forks / 53 open issues（gh api） |

## 2. 项目是什么，解决什么问题

问题出发点写在 README 的 "Why" 一节：让 agent 生成页面时，无论什么模型、什么项目，都会反复收敛到同一套"AI 默认审美"——Inter 字体、紫蓝渐变、卡片套卡片、灰字压彩底、每个标题上方一个圆角方块图标（README.md L25-29）。这些不是 bug，而是模型在无设计约束时的统计均值，作者称之为 "AI slop"。

Impeccable 的解法分三层：

1. **引导层（skill 剧本）**：一个 Markdown 形式的 Agent Skill（`skill/SKILL.src.md`），安装后提供 `/impeccable <命令>` 词汇——`bolder`（放大保守设计）、`quieter`（收敛过度设计）、`critique`（按启发式打分的 UX 评审）、`audit`（可访问性/性能/响应式技术审查）、`polish`（发布前最后一遍打磨）等 24 个命令。每个命令对应一份参考剧本（`skill/reference/*.md`，共 40 个文件）。核心思想是"给想要的东西起名字"：说 `/impeccable bolder the pricing cards` 比说 "make it pop" 可执行得多。
2. **检测层（确定性引擎）**：63 条反模式规则（`crates/foundation/src/registry.rs` 的 `ANTIPATTERNS` 注册表，README 写 62 条，注册表现为 63 条）覆盖 side-tab 描边、渐变文字、cream 配色、嵌套卡片、AI 配色盘等"审美 tell"，以及紧凑内边距、跳级标题等质量问题。全部为静态/确定性检查，不调用 LLM。可通过 CLI（`npx impeccable detect <目录或URL>`）、CI（`--github` 输出 GitHub Actions 注解）、浏览器扩展三种方式运行（README.md L72-90）。
3. **迭代层（live mode）**：`/impeccable live` 起一个本地服务器并注入 overlay，用户在浏览器里选中元素、切换变体、确认后回写源码，支持 Vite/Next/SvelteKit/Astro/Nuxt/纯 HTML（README.md L66-70；引擎实现见 `crates/live/`）。

配套上下文机制：`init` 命令询问产品定位并写入 `PRODUCT.md`，`document` 命令从现有代码提炼视觉系统写入 `DESIGN.md`（仓库根目录就有这两个文件的自用样例：`PRODUCT.md`、`DESIGN.md`），后续命令都会读取，保证改动不跑偏。

分发方式是该项目的一大特色：仓库根目录同时提交了 20 多个 agent 工具的安装产物（`.claude/`、`.cursor/`、`.codex/`、`.agents/`、`.gemini/`、`.devin/`、`.github/` 等，见 README.md L128-158 的映射表），由 `scripts/build.js` 从 `skill/` 源文件同步生成；运行时载体是 npm 包 `impeccable`（`package.json`），通过 optionalDependencies 按平台分发预编译的 Rust 引擎二进制（`@impeccable/cli-<os>-<arch>`），skill 内置 launcher 首次运行时也可下载到 `~/.impeccable/bin/`（README.md L102、docs/ENGINE.md L20）。

项目血缘：README 结尾注明它脱胎于 Anthropic 官方的 frontend-design skill（anthropics/skills 仓库），从"一个提示词文件"扩展成了完整引擎。

## 3. 技术栈与依赖

- **引擎（核心）**：Rust 2021 edition，Cargo workspace，版本 0.1.18（`Cargo.toml`）。关键依赖极少：`serde`/`serde_json`、`thiserror`、`regex`、`once_cell`（`Cargo.toml` L27-30）；本地 HTTP 用自写的 `impeccable_common::http`（每连接一线程的 `TcpListener` 服务器），docs/ENGINE.md L70-77 解释了弃用 tiny_http 的原因（0.12 连接池漏答浏览器连接突发）。
- **浏览器交互**：Chrome/Edge 经 CDP（Chrome DevTools Protocol）驱动（`crates/browser/src/`），无 selenium 等重型依赖。
- **wasm**：`crates/wasm` 通过 wasm-bindgen 把 core 规则导出给浏览器内页 bundle 和 Chrome 扩展的 offscreen core（docs/ENGINE.md L41-43）。
- **Node/Bun（构建与分发）**：npm 包壳 `impeccable`（bin 指向 `cli/bin/cli.js`），Bun 做 skill 构建（`scripts/build.js`），发布平台包与引擎校验（`scripts/fetch-engine.mjs`、`engine-checksums.txt`）。
- **测试侧 devDependencies**：Playwright、Puppeteer、`@anthropic-ai/claude-agent-sdk`、`@ai-sdk/openai`/`google`/`anthropic`、Svelte 5、Zod（`package.json` L93-106）——说明测试不仅跑代码，还用真实 LLM 驱动 skill 行为测试。
- **运行时要求**：Node 仅 npx 安装需要；引擎是自包含静态二进制，离线可用（docs/ENGINE.md L12-13、README.md L102）。

## 4. 目录结构与代码组织

顶层目录表（职责出处：docs/ENGINE.md 的 Layout 表 + 实际清点）：

| 目录 | 职责 |
|---|---|
| `crates/cli` | `impeccable` 二进制入口：动词路由、退出码（`crates/cli/src/main.rs`） |
| `crates/common` | IO 抽象、路径/进程助手、自写 HTTP/1.1 服务器核心 |
| `crates/foundation` | 规则注册表、findings、颜色、JS 语义助手、Dom trait、内联 ignore |
| `crates/core` | 规则逻辑本体：所有 `check_*`/`scan_*` 与启发式（`checks/rules.rs` 4003 行） |
| `crates/html` | 静态 HTML 引擎：解析器、CSS 级联（内嵌 csstree AST）、静态 DOM、规则适配 |
| `crates/browser` | URL 引擎：Chrome 发现、CDP、页面快照、视觉检查 |
| `crates/detect` | `impeccable detect`：文件遍历、配置、ignore、输出、GitHub 注解、正则匹配器 |
| `crates/live` | live mode：本地服务器、变体回写、手动编辑、Svelte/Vue 适配（最大 crate） |
| `crates/hook` | 设计 hook（agent 编辑 UI 文件后自动跑检测） |
| `crates/context` | 上下文/doctor/陈旧检测/命令 pin |
| `crates/skills` | install/update/check/link（原 npm CLI 动词的 Rust 化） |
| `crates/comp` + `comp-verbs` | comp 保真：光栅、PNG、字体度量与构建期 comp-diff |
| `crates/wasm` / `crates/bundle` / `crates/xtask` | wasm 导出、页内 bundle 与打包、`cargo xtask bundle` |
| `skill/` | skill 源：`SKILL.src.md` + `reference/` 40 份剧本 + `scripts/`（launcher 与 13862 行的 live-browser.js 注入前端） |
| `.claude/` `.cursor/` `.agents/` 等 20 余目录 | 构建产物按工具分发，根目录即"应用商店"（`.agent/skills/impeccable/SKILL.md` 等） |
| `browser-bundle/` | 页内 bundle 的 JS 源 |
| `extension/` | Chrome 扩展（manifest、devtools、offscreen） |
| `vscode/`、`cursor-plugin/`、`plugin/` | VS Code 扩展 / Cursor 插件 / Grok 插件 |
| `tests/` | JS 侧测试（oracle、live-e2e、skill-behavior 等 60+ 文件） |
| `demos/`、`docs/` | 演示页面；22 份工程文档（ENGINE.md、CLI-CONTRACT.md、DEVELOP.md 等） |

组织方式评述：这是典型的"单一产品 monorepo"——一个 Cargo workspace（17 crates）+ 一个 skill 源 + 多端分发产物全部同居一仓。特点是**构建产物入库**（`.claude/` 等目录是生成的、由 `sync-generated-output.yml` workflow 保持同步），让不支持 npm 的用户能直接拷贝目录使用。分层清晰：foundation（纯数据/类型）→ core（规则）→ html/browser（载体）→ detect/live/hook（入口动词）→ cli（路由），`crates/core` 再导出 foundation 模块让消费者只认一个 crate 名（docs/ENGINE.md L48-53）。

## 5. 核心架构与关键实现

**1) 规则注册表（`crates/foundation/src/registry.rs`）**。内置规则是 JS 时代 `ANTIPATTERNS` 数组的逐字节移植，静态 `&[Antipattern]` 存储，零运行时开销；第三方 rule pack 通过 `extend()` 追加到独立列表，查查找时先内置后扩展，pack 永远不能遮蔽内置 id（文件头注释 L1-8）。每条规则同时携带 `skill_section`/`skill_guideline` 字段，把"检测命中"直接链接回 skill 剧本中的对应准则——检测与引导共享一套词汇：

```rust
pub static ANTIPATTERNS: &[Antipattern] = &[
    Antipattern {
        id: "side-tab",
        category: "slop",
        name: "Side-tab accent border",
        description: "Thick colored border on one side of a card — the most
            recognizable tell of AI-generated UIs. ...",
        skill_section: Some("Visual Details"),
        skill_guideline: Some("colored accent stripe"),
    },
    // ...共 63 条
```

**2) 对比度与颜色检查（`crates/core/src/checks/rules.rs`，4003 行）**。这是最大的单个规则文件，`check_colors`（L957）、`check_placeholder_colors`（L1590）、`check_hover_contrast`（L1645）等实现了完整的对比度打分；设计上做了大量"抗误报"工作：`check_colors_deduped_claiming`（L1102）实现每页去重与"认领"机制——同一元素只报一次、由特定规则认领，SAFE_TAGS 白名单路径（L1584 注释提到 exxonmobil.com 这类全彩企业站案例）避免对刻意用色误报。注释密度很高，几乎每个函数都有"为什么"级别的文档。

**3) live mode 回写链路（`crates/live/src/`）**。最大的 crate（约 4.5MB 源码）。`live_server.rs`（3098 行）在本地起 HTTP 服务器，注入 `skill/scripts/live-browser.js`（13862 行）到目标页面；用户选中元素后按方向生成变体，accept 时由 `accept_css.rs`/`accept_verify.rs`/`manual_edits/` 负责把改动落回源码并有事务与日志（`journal.rs`、`live_commit_manual_edits.rs`），还带 Svelte 专用的 AST 处理（`svelte_ast.rs`、`svelte_bridge.rs`）。这套"浏览器里选中 → 源码回写"是对话式 agent 很难做到的精确迭代。

**4) 双引擎检测路径（`crates/detect/src/` + `crates/browser/src/`）**。`cli.rs`（1227 行）区分两类目标：源文件（JSX/TSX/HTML/CSS）只跑正则/模式规则（`regex_matchers.rs`、`jsp.rs` 做 JS 解析），URL 目标则启动 CDP 拿真实渲染（对比度、小字等渲染型规则只能在真实页面上跑，README.md L86 明确说明这一取舍）。`github_annotations.rs` 输出 CI 注解，退出码 2=有发现、1=扫不了（README.md L88）。

## 6. 构建与运行

来自 docs/ENGINE.md L57-61 与 package.json：

```bash
# 引擎（纯 Rust，离线可构建）
cargo build --release -p impeccable      # 产物 target/release/impeccable
cargo test --workspace
IMPECCABLE_BIN=target/release/impeccable node tests/oracle/run.mjs   # 行为门禁

# skill 构建（需要 Bun）
bun run build          # scripts/build.js 同步生成各工具目录
bun run test           # node scripts/run-tests.mjs，多套件选择器
bun run test:core / test:oracle / test:detector / test:live

# JS 侧专项测试（package.json scripts）
bun run test:live-e2e / test:skill-behavior / test:skill-activation / test:plugin-e2e

# 终端用户
npx impeccable install  # 检测已装工具并安装 skill + hook
npx impeccable detect src/ | detect https://example.com | detect --list-rules
```

工具链锁定：`rust-toolchain.toml`（stable + wasm32 target）、`.bun-version`、Node >= 18.19（`package.json` engines）。引擎二进制版本由 `ENGINE_VERSION`（0.1.18）与 npm 包版本联动，`scripts/check-engine-release.mjs` 校验一致性。

## 7. 测试与工程质量

- **测试规模可观**：JS 侧 `tests/` 60+ 文件；Rust 侧各 crate 均有 `tests/` 集成测试（detect/core/html 等 10+ 套件）。package.json 列出 15+ 个专项测试套件，含 live-e2e、plugin-e2e、skill-activation/workflow 等端到端类别（`package.json` L57-74）。
- **Oracle 测试是亮点**：`tests/oracle/` 把每个动词的可观察行为"逐字节"钉死，配合 `tests/oracle/vectors/calls/` 的冻结函数级向量回放（docs/ENGINE.md L9-10、L79），保证引擎重构不改变外部行为——对一个会被海量项目依赖的 CLI 来说，这是很成熟的契约测试思路。
- **LLM 行为测试**：devDependencies 里的 claude-agent-sdk 与 ai-sdk 表明 skill 剧本本身也有自动化行为测试（`tests/skill-behavior/`、`skill-workflow/`），即"测试提示词工程"。
- **CI**：`.github/workflows/ci.yml` 为 push/PR 触发，16+ 个 job 覆盖 ubuntu 为主、2 个 windows job（跨平台引擎是卖点，Windows 专门验证）；另有 `release-engine.yml`（引擎发布）、`sync-generated-output.yml`（保持根目录分发产物同步）、`sheriff.yml` 三个辅助 workflow。
- **代码质量观察**：Rust 代码约 25.9 万行/429 文件（`find crates -name '*.rs' | wc -l` + 行数统计），注释密度高、模块边界文档化（每个 crate 头部说明职责）；doc/ 下有 ENGINE/CLI-CONTRACT/PORTING-GUIDE 等内部规范，且有版本化评审记录（如 `docs/REVIEW-4.4.0-READINESS.md`）。工程纪律在独立项目中属于顶级水平。

## 8. 社区与维护状态

- **极度活跃**：创建于 2025-11-16（gh api），约 11 个月达到 85.3k 星；最近推送 2026-10-30（分析前一天），最新 release cli-v4.8.0 于 2026-10-26。tarball 内 `docs/RELEASE-4.4.0.md` 显示 release 走正式评审流程。
- **维护结构**：贡献者 API 首页返回 30 人（默认分页上限 30，故为"至少 30 人"）；外部贡献要求先开 issue 再动手（README.md L213 + docs/DEVELOP.md），主作者 Paul Bakaus 为核心推动者（个人品牌输出 release notes，README.md L215），属"强个人主导 + 社区外围贡献"结构。
- **生态面**：npm 包 `impeccable` + 5 个平台子包、Chrome Web Store 扩展、VS Code Marketplace 扩展、Claude Code 插件市场、Grok 插件、git submodule 方案（README.md L104-125），发行渠道覆盖极广。
- 风险注记：README "Hooks run outside model approval"（L205）坦承 hook 在 Claude Code 中即使拒绝模型命令也会执行、首次编辑可能触发引擎下载——项目把这点写得很透明，但属于用户需要知晓的行为。

## 9. 亮点与风险

**亮点**

1. **精准踩中时代痛点**："AI 生成 UI 千篇一律"是 2025-2026 年所有前端使用者的直观感受，该项目把模糊抱怨变成可命名、可检测、可执行的词汇表，定位在"设计 lint + 设计教练"的空白生态位（README.md L22-29）。
2. **确定性检测引擎是实打实的 Rust 工程**：63 条规则、自写 HTML 级联引擎与 CDP 驱动、零 LLM 依赖、wasm 复用到浏览器扩展，25.9 万行代码支撑"无 API key 也能跑"的承诺（crates/ 全树、Cargo.toml）。
3. **发行工程极其完善**：20+ agent 工具产物入库、npm 平台二进制、checksum 校验、oracle 行为门禁、Windows CI，把"skill 分发"做成了产品的第一性问题（根目录结构、tests/oracle/、.github/workflows/）。
4. **测试覆盖含 LLM 行为层**：不只测引擎，还用 claude-agent-sdk 测 skill 剧本的实际行为，这在提示词工程领域少见（package.json devDependencies、tests/skill-behavior/）。
5. **文档纪律**：ENGINE.md/CLI-CONTRACT.md/DEVELOP.md 把架构契约写明，README 对隐私（telemetry 内容清单）与安全（hook 权限）主动披露。

**风险/局限**

1. **效果上限依赖 LLM 服从度**：skill 剧本本质是提示词，对不听话的模型没有强制力；检测器只能拦"可枚举的 tell"，设计品质本身不可枚举。
2. **遥测与外部请求**：新设计会从 impeccable.style 目录拉取视觉方向并匿名上报选择（README.md L209），离线/隐私敏感团队需显式关闭；仓库虽开源且引擎离线可构建，但完整体验绑定其官网服务。
3. **仓库形态的双刃剑**：根目录 20+ 个生成产物目录虽是刻意设计，但对读源码者是噪音；README/文档与代码体积比偏高，初次理解成本大。
4. **个人主导的 bus factor**：核心架构与发布节奏高度绑定单一作者，30+ 贡献者中外部贡献占比"未确认"；11 个月冲到 85k 星的高速也可能包含热度回落风险。

## 10. 总结评价

Impeccable 是"AI slop 时代"最具代表性的前端质量基础设施：用一个 Markdown skill 提供设计词汇、用一个严肃的 Rust 引擎提供确定性拦截、用一套近乎偏执的发行工程把三者塞进 20 多个 agent 工具。它证明了对"提示词产品"做传统软件工程（契约测试、oracle 门禁、跨平台二进制、隐私披露）是可行且必要的。适合所有用 AI 编码 agent 写前端、且饱受默认审美折磨的开发者立即试用；适合 Rust 工程师研读 crates/foundation→core→html 的分层与 tests/oracle 的行为钉死手法；对做 agent 生态工具的人，其根目录分发产物 + npm 引擎下载器的双轨发行方案是活的教科书。需要注意它的检测是启发式而非标准（别拿 P0 当 W3C），以及完整体验对 impeccable.style 服务的轻度依赖。
