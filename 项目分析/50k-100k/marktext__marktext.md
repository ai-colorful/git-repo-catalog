# marktext/marktext 项目分析

> 星数 63033（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：MarkText 是一款开源（MIT）的所见即所得（WYSIWYG）Markdown 桌面编辑器，主打"实时渲染、无干扰写作"，基于 Electron + Vue 3 + 自研 Muya 编辑内核，覆盖 Linux/macOS/Windows。

| 事实 | 内容 | 出处 |
| --- | --- | --- |
| 名称/版本 | marktext，`0.24.0-dev`（monorepo 根与 desktop 包一致） | `package.json`、`packages/desktop/package.json` |
| 星数 | 63033（快照 2026-10-24） | 任务参数 |
| 语言 | TypeScript（strict），渲染层含 Vue SFC | `packages/desktop/package.json`、AGENTS.md |
| 许可证 | MIT | `LICENSE`、GitHub API |
| 默认分支 | `develop`（贡献须提交到 develop） | GitHub API、AGENTS.md |
| 最近活跃 | 最近 push 2026-10-30；最新 release v0.23.0（2026-10-21） | GitHub API |
| 平台 | Linux x64/arm64、macOS 13+ x64/arm64、Windows 10/11 x64/arm64 | `README.md`、`packages/desktop/package.json` |

## 2. 项目是什么，解决什么问题

MarkText 解决的核心痛点是：主流 Markdown 编辑器要么是"纯文本 + 侧边预览"（Typora 之前的双栏模式），要么是商业闭源（Typora 收费）。MarkText 直接在编辑视图中实时渲染 Markdown 元素（标题、加粗、公式、表格所见即所得），让作者在不离开纯文本语义的情况下获得接近成品的视觉反馈（`README.md` Features 节）。

功能层面（`README.md` + `packages/website/content/docs/CHANGELOG.md`）：支持 CommonMark、GFM 及选择性支持 Pandoc Markdown；扩展包括 KaTeX 数学公式、Mermaid/flowchart.js/PlantUML/Vega 图表、front matter、emoji；支持 Source Code、Typewriter、Focus 三种编辑模式；可导出 HTML 与 PDF；支持剪贴板贴图、多主题（Cadmium Light、Material Dark 等）；v0.22/0.23 新增 GitHub Alerts（NOTE/TIP 等）、可折叠标题、独立文档大纲、per-tab 编辑/只读模式与 `--read-only` CLI 选项。

架构层面（AGENTS.md "Three-Process Electron Model"）：这是一个严格分层的三进程 Electron 应用——主进程管 IO/窗口/文件监听/自动更新，渲染进程跑 Vue 3 UI 与两个编辑器实例（Muya 所见即所得 + CodeMirror 源码模式），两者之间通过类型化 IPC 契约通信；编辑内核 Muya 独立成无 Electron 依赖的 npm 包 `@muyajs/core`，可脱离桌面端单独使用（`packages/muya/package.json`）。

值得注意的是该项目在 2025-2026 年经历了大规模复兴：从旧的单包 `src/main|renderer|muya` 结构迁移到 pnpm monorepo，Muya 用 TypeScript 完全重写（v2，`packages/muya/AGENTS.md`），依赖升级到 Electron 45 / Vue 3.5 / Node ≥22.12（`package.json`、`pnpm-workspace.yaml`），并在 2026 年 10 月两周内连发 v0.21.0→v0.23.0 三个 release（GitHub API）。

## 3. 技术栈与依赖

| 层 | 技术 | 出处 |
| --- | --- | --- |
| 语言 | TypeScript strict（desktop）；muya 独立 antfu ESLint 配置 | `packages/desktop/package.json`、`packages/muya/eslint.config.mjs` |
| 桌面壳 | Electron ~45，electron-vite 6 构建，electron-builder 26 打包 | `package.json`、`packages/desktop/package.json` |
| 前端 | Vue 3.5 + Pinia 4 + Vue Router + Element Plus + vue-i18n 11 | `packages/desktop/package.json` |
| 编辑内核 | `@muyajs/core` 0.2.0：ot-json1 + ot-text-unicode（OT 操作）、snabbdom（虚拟 DOM）、marked 16（解析）、rxjs（事件流合并） | `packages/muya/package.json`、`packages/muya/AGENTS.md` |
| 源码模式 | CodeMirror 5.65 | `packages/desktop/package.json` |
| 文件监听 | chokidar 5 | `packages/desktop/src/main/filesystem/watcher.ts` |
| 其他关键库 | katex、mermaid 12、prismjs、flowchart.js、snapsvg-cjs（图表渲染）；@vscode/ripgrep（全文搜索）；electron-store/updater/log；keytar（凭据）；ced（编码检测）；dompurify、turndown | `packages/desktop/package.json` |
| 工程化 | pnpm ≥10 workspace（lockfile 2 万行）、ESLint 10 flat config、Prettier、husky + lint-staged、Vitest 4、Playwright | `package.json`、`pnpm-lock.yaml`、`eslint.config.js` |
| 运行时 | Node ≥22.12，pnpm 10.33.4 | `package.json` |

## 4. 目录结构与代码组织

顶层为 pnpm monorepo，根目录只保留共享工具链（`AGENTS.md` "Directory Structure"）：

| 目录 | 职责 |
| --- | --- |
| `packages/desktop/` | Electron 应用本体（包名 `marktext`），含全部主/渲染/preload 源码与测试 |
| `packages/muya/` | 编辑器内核 `@muyajs/core`（TypeScript 重写版），自含 ESLint/stylelint/madge/vitest 工具链 |
| `packages/website/` | 官网与文档站（Next.js 15.5 + React 19，`packages/website/package.json`） |
| `scripts/` | workspace 级脚本：postinstall、minify-locales、第三方许可证生成/校验 |
| `docs/` | 仅 assets 与 i18n 翻译 README |
| `.github/workflows/` | 14 条 CI 工作流（desktop 与 muya 各自成套） |

desktop 包内部（`packages/desktop/src/`，616 个 TS/Vue 文件、约 3.2 万行 TS）：

- `main/`（98 个 TS 文件）：`app/`（App 类、WindowManager、accessor）、`filesystem/`（含 watcher.ts）、`windows/`（editor/setting/projectRoots）、`menu/`、`commands/`、`ipc/`、`preferences/`、`spellchecker/`、`dataCenter/`、`keyboard/`、`cli/`、`applicationState/`
- `preload/index.ts`：唯一桥接文件（302 行），类型化 contextBridge
- `renderer/src/`：Vue 应用，61 个 `.vue` 组件、13 个 Pinia store（editor/preferences/layout/project/treeCtrl/commandCenter/keybindings 等）
- `shared/types/`：跨进程类型 + IPC 契约 `ipc.ts`（298 行）

组织方式评述：典型"双单体拆分"——desktop 与 muya 是强边界（muya 不 import Electron，root ESLint 显式 ignore muya 目录，muya 自带全套工具链），website 再独立。这种"内核可独立发布 + 壳应用"的布局在 Electron 项目中少见，是本次重构最大的结构改进。

## 5. 核心架构与关键实现

### 5.1 Muya 编辑器内核：block 树 + OT 状态（`packages/muya/src/`）

Muya v2 是整个项目的灵魂。文档模型是自研 block 树：所有块继承 `TreeNode → Parent → (Content | Format)`（`src/block/base/`），具体块类型分布在 `src/block/{commonMark,gfm,extra,content}` 并必须在 `src/block/index.ts::registerBlocks()` 注册，根节点 `ScrollPage`（`src/block/scrollPage/`）通过 `ScrollPage.loadBlock(name).create(...)` 工厂创建块实例（`packages/muya/AGENTS.md` Architecture 节）。

状态层以 `JSONState`（`src/state/index.ts`）为唯一事实源，直接暴露 `ot-json1` 的 `invert/compose/transform` 静态方法——架构已为 OT 协同编辑预留，只是尚未接入传输层。Markdown 双向转换由 `state/markdownToState.ts`（marked 解析，506 行）与 `state/stateToMarkdown.ts`（719 行）完成，HTML 桥接用 turndown（`state/htmlToMarkdown.ts`）。

编辑操作通过 `Editor.updateContents(operation, selection, source)` 把 `ot-json1` 操作应用到活跃 block 树（`src/editor/index.ts:542`），内部手写 pick/drop 遍历以同步操作 block 树与 JSON 状态；行内文本编辑编码为嵌套在 json1 op 里的 `ot-text-unicode` op：

```ts
// packages/muya/src/editor/index.ts:542
updateContents(operation: JSONOp, selection: Nullable<TSelectionState>, source: ChangeSource) {
    if (!this._muya.canMutate)
        return;
    return this.transactions.apply(operation, { selection, source });
}
```

渲染层：`src/inlineRenderer/` 用自研 lexer/rules 管线做行内 token 化，经 snabbdom 输出虚拟 DOM；UI 层约 20 个浮动工具（格式工具栏、图片工具、表格拖拽、emoji 选择器等）继承 `baseFloat/baseScrollFloat`，用 `@floating-ui/dom` 定位，由消费方通过静态 `Muya.use(Plugin, options)` 注册（`src/muya.ts:143`）。核心文件 `src/muya.ts` 达 1650 行。

### 5.2 文件监听器：自愈式 chokidar 封装（`packages/desktop/src/main/filesystem/watcher.ts`，539 行）

Watcher 分两类：`file`（单文件内容热重载）与 `dir`（侧边栏目录树），用 chokidar v5 实现，处理了大量真实世界边角问题：

- **写稳定延迟差异化**：仅 file 型 watcher 开 `awaitWriteFinish`（stabilityThreshold=1000ms, poll=150ms），避免读到半截文件（GH#1043）；dir 型不延迟，否则侧边栏新文件出现会慢约 1 秒（GH#3955）。
- **乱序读保护**：按 `(windowId, type, pathname)` 维护读序列号（`readSequences` WeakMap），异步 `loadMarkdownFile` 返回后用 `isLatestRead` 丢弃过期结果。
- **自我修改回环抑制**：编辑器自己保存文件时调用 `ignoreChangedEvent` 在时间窗内忽略事件，并对云盘延迟同步场景补一次 `stat` mtime 复核（GH#3044，`_shouldIgnoreEvent`）。
- **移动/重命名屏障**：`withIgnoredFileDeletions` 在移动操作期间屏蔽 rename 产生的 unlink，失败时复查磁盘并上报缺失文件。
- **平台兼容**：macOS 与 UNC 路径强制 polling；Linux 上通过 raw rename 事件 + 150ms 防抖重挂 watch 修复原子重命名（chokidar#591）；ENOSPC（inotify 上限）降级为用户通知。

```ts
// packages/desktop/src/main/filesystem/watcher.ts:318
...(type === 'file'
  ? {
    awaitWriteFinish: {
      stabilityThreshold: WATCHER_STABILITY_THRESHOLD,
      pollInterval: WATCHER_STABILITY_POLL_INTERVAL
    }
  }
  : {}),
```

事件统一经 `emit` 发往渲染进程 IPC 通道 `mt::update-file` / `mt::update-object-tree`，发送前检查 `win.isDestroyed()`；文件头保留 `TODO(refactor): GH#1035`，作者自己标注了该模块待重构。

### 5.3 安全模型与 IPC 契约（`packages/desktop/src/main/config.ts` + `src/preload/index.ts` + `src/shared/types/ipc.ts`）

两个窗口均配置 `contextIsolation: true`、`sandbox: true`、`nodeIntegration: false`（`src/main/config.ts:15-22,39-44`）；`webSecurity` 仅开发态关闭（Vite HTTP origin 加载本地 `file://` 图片的需要，`config.ts:9`）。注意 `AGENTS.md` 架构节里"editor/preferences 窗口用 contextIsolation: false"的说法已过时，以代码与 `packages/website/content/docs/dev/ARCHITECTURE.md` 为准。

preload（`src/preload/index.ts`）只暴露一个 302 行的类型化桥：`send/sendSync/invoke/on` 全部由 `shared/types/ipc.ts` 的 `IpcInvokeChannels/IpcSendChannels/IpcSyncChannels/IpcMainEventChannels` 四个泛型接口约束，通道名（`mt::` 前缀）、参数元组、返回值类型在调用点即被检查；启动时一次 `mt::boot-info` 同步握手传递平台/环境信息。渲染进程通过 `window.electron`、`window.fileUtils` 等窄接口访问能力，无法直接 import Node/Electron。配套有 `test/e2e/xss.spec.ts` 与 `context-isolation.spec.ts` 两条安全回归测试。

### 5.4 主进程窗口与应用编排（`packages/desktop/src/main/`）

`app/index.ts` 的 `App` 类持有 `Accessor`（偏好/环境聚合）与 `WindowManager`，按 CLI 参数与第二实例请求打开 `EditorWindow`/`SettingWindow`（`windows/`），注册拼写检查、键盘、主题、剪贴板图片补全等 IPC 监听；应用状态可序列化以支持会话恢复（`applicationState/`、`@shared/types/applicationState`）。文件读写集中在 `filesystem/`（`markdown.ts` 负责编码检测/行尾规范化/尾随空行策略，`documentWrites.ts` 管写入，`trash.ts` 管安全删除）。

## 6. 构建与运行

命令均来自根 `package.json` 与 AGENTS.md，已实测存在于源码：

```bash
pnpm install            # postinstall：patch-package 补丁、下载 Electron、rebuild 原生模块、压缩 locale
pnpm run dev            # electron-vite dev，渲染进程 HMR；改主进程需重启
pnpm run build:unpack   # 只构建不打包（快速验证编译）
pnpm run build:win|mac|linux            # 自动先 minify-locales + electron-rebuild 再 electron-builder
pnpm run build:win:x64|arm64            # 各平台 x64/arm64 细分脚本齐全
pnpm run test           # Vitest 全部单测
pnpm run test:e2e       # Playwright 端到端
pnpm run lint && pnpm run typecheck    # CI 强制：eslint --cache + vue-tsc --noEmit
pnpm -C packages/muya test:spec         # CommonMark 0.31 + GFM 0.29 一致性套件
```

打包输出统一到仓库根 `dist/`（`packages/desktop/electron-builder.yml` 的 `directories.output: ../../dist`）。

## 7. 测试与工程质量

测试规模在同类 Electron 项目中属第一梯队：

- **desktop 单测**：`test/unit/specs/` 154 个 `.spec.ts`（Vitest 4）。
- **desktop E2E**：`test/e2e/` 约 147 个 Playwright spec，其中 50+ 个以 `issue-NNNN` 命名的回归测试，覆盖 XSS、context-isolation、会话恢复、大文件、CJK 排版等；CI 矩阵横跨 6 个平台（linux/win/macos × x64/arm64，`.github/workflows/build.yml`）。
- **muya 单测**：`src/**/__tests__/` 335 个测试文件；另有 CommonMark 0.31 / GFM 0.29-gfm 一致性套件（`test/spec/`），用 `expected-failures.json` 做"棘轮锁"——已失败样例列表中任何一项转好会报错提醒移除，任何新失败也报错，一致性只能升不能降（`packages/muya/AGENTS.md`）。当前基线 CommonMark 87.7% / GFM 86.3%（`test/spec/conformance.md`）。
- **CI**：14 条工作流，desktop（build/e2e/lint/test/release）与 muya（build/circular/e2e/lint/spec/test）两套独立流水线，另有 validate-licenses、website-deploy、sponsors 同步（`.github/workflows/`）。
- **工程质量观察**：madge 循环依赖检查（CI 强制）、husky + lint-staged、第三方许可证生成与校验脚本、统一 Prettier/ESLint。文档质量突出：根 `AGENTS.md`（16.5KB，面向 AI 与人类开发者）+ `packages/website/content/docs/dev/` 13 篇开发者文档（ARCHITECTURE/IPC/INTERFACE/DEBUGGING/PERFORMANCE/RELEASE 等）。

## 8. 社区与维护状态

- **活跃度**：未归档，最近 push 2026-10-30（分析前 1 天），open issues 219，forks 4687（GitHub API）。项目 2017 年创建，2022-2024 曾长期沉寂，2025-2026 明显复兴。
- **Release 节奏**：2026-10-08 至 10-21 两周内连发 v0.21.0、v0.21.1、v0.21.2、v0.22.0、v0.23.0 五个版本；release 工作流含 semver tag 校验（GitHub API、`.github/workflows/release.yml`）。
- **贡献者结构**：Jocs（作者）1701 次、fxha 372 次、Tkaixiang（maintainer 名单，`packages/desktop/package.json`），其后为长尾贡献者；近期大量 issue 编号达 #5000-#5800。资金来源为 GitHub Sponsors + OpenCollective（含 serpapi 特别赞助，`README.md`）。
- **近期开发方向**（`packages/website/content/docs/CHANGELOG.md` 0.22/0.23 节）：per-tab 编辑模式、可折叠标题、GitHub Alerts、Linux arm64 官方包等。

## 9. 亮点与风险

**亮点**

1. 真正的 block 级 WYSIWYG 内核：Muya v2 以 OT（ot-json1/ot-text-unicode）为文档操作原语，协同编辑只需接传输层；内核独立为 `@muyajs/core` npm 包，可脱离 Electron 复用（`packages/muya/src/state/index.ts`、`packages/muya/package.json`）。
2. 测试棘轮机制：CommonMark/GFM 一致性基线 + expected-failures.json 锁定，杜绝一致性回退（`packages/muya/test/spec/`）；655+ spec 文件 + 6 平台 CI 构建矩阵。
3. Electron 安全实践规范：sandbox + contextIsolation 全开、类型化 IPC 契约、XSS/隔离性回归测试（`src/main/config.ts`、`src/shared/types/ipc.ts`、`test/e2e/xss.spec.ts`）。
4. 文件监听边角处理极其扎实：乱序读保护、云盘延迟同步、UNC 路径、inotify 上限、Linux 原子重命名等都有对应修复与 issue 引用（`src/main/filesystem/watcher.ts`）。
5. 面向 AI 时代的工程文档：AGENTS.md + COMMENTING-GUIDELINES + 13 篇 dev docs，规则具体到"条件表达式里不得藏副作用"级别。

**风险/局限**

1. 文档漂移：AGENTS.md 中窗口安全配置（称 contextIsolation: false）与实际代码（全 sandbox）不一致，website 技术栈描述在 AGENTS.md（Vite+React18）与 ARCHITECTURE.md（Next.js）之间矛盾——快速重写期的文档债（对比 `src/main/config.ts`、`packages/website/package.json`）。
2. Markdown 解析一致性仅 ~87%，非完整 CommonMark 实现，边缘语料可能渲染异常（`packages/muya/test/spec/conformance.md`）。
3. 双编辑引擎并存（Muya + CodeMirror 5）：源码模式与 WYSIWYG 模式间状态同步是长期复杂度来源，CodeMirror 5 亦属遗留版本（`packages/desktop/src/renderer/src/codeMirror/`）。
4. 依赖面宽（mermaid/flowchart.js/snapsvg-cjs/prism/katex/ripgrep/keytar 等原生或重型依赖），构建要求 Node ≥22.12 + pnpm ≥10，贡献与自建门槛偏高（`packages/desktop/package.json`、`pnpm-workspace.yaml`）。

## 10. 总结评价

MarkText 是"沉寂多年后彻底重造"的典型案例：以 pnpm monorepo + TypeScript 重写的 Muya v2 内核 + 棘轮式一致性测试 + 沙箱化 IPC 为骨架，把一个 2017 年代的老牌编辑器拉到了 2026 年的工程水准，且仍由原作者 Jocs 主导高强度迭代。它适合作为"Markdown 编辑器用户"的首选开源 WYSIWYG 方案（若能接受解析一致性 ~87% 的边缘 case），更适合作为"Electron 复杂应用工程"的源码范本——三进程分层、类型化 IPC、文件监听容错、OT 就绪的编辑内核，每一块都有可细读的真实代码；想研究编辑器内核本身的人应从 `packages/muya/src/editor/index.ts` 与 `src/state/` 入手，想研究 Electron 工程化的人则从根 `AGENTS.md` 与 `packages/desktop/src/main/` 入手。
