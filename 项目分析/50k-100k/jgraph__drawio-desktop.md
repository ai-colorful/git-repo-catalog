# jgraph/drawio-desktop 项目分析

> 星数 63625（快照 2026-10-24）｜主语言 JavaScript｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：draw.io 桌面版——一个把 Web 端 draw.io 图表编辑器（jgraph/drawio）包装成本地离线应用的 Electron 壳，主打"安全隔离"与跨平台分发。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | jgraph/drawio-desktop（包名 `draw.io`） | `package.json` |
| 星数 | 63625（快照） | 任务参数 |
| 语言 | JavaScript（ESM，无 TypeScript） | `package.json`、`src/main/*.js` |
| 许可证 | GPL-3.0-only（v31.5.2 起由 Apache 2.0 改为 GPL v3；核心 drawio 子模块仍 Apache 2.0） | `package.json`、`CLAUDE.md` |
| 最近活跃 | 仓库最近 push 2026-10-26；最新 release v32.6.1（2026-10-23，29 个资产） | gh api 元数据 |
| 版本 | 32.6.1（版本号与 drawio/VERSION 同步） | `package.json`、`sync.cjs` |
| 创建时间 | 2017-05-25，forks 5803，watchers 563 | gh api 元数据 |

## 2. 项目是什么，解决什么问题

drawio-desktop 是 draw.io 图表编辑器的桌面容器。真正的编辑器核心（渲染、图形、菜单等全部 Web 逻辑）在姊妹仓库 jgraph/drawio 中，以 git 子模块形式挂在 `drawio/` 目录（`.gitmodules`，指向 dev 分支）；本仓库只负责 Electron 壳：窗口管理、文件系统 IPC、命令行导出、自动更新、打包签名与多渠道分发（`README.md`、`CLAUDE.md`）。

它解决的第一个问题是**离线与数据隔离**：README "Security" 一节明确设计目标是"与应用外完全隔离，除更新检查外不联网"，CSP 禁止远程加载脚本、限制 connect-src 只能连自身，图数据不外发、无使用统计（`README.md`、`src/main/electron.js` 中 `onHeadersReceived` 设置的 CSP 头）。对内网/涉密环境这是刚需，也是它相对在线版的核心卖点。

第二个问题是**本地文件集成**：原生打开/保存对话框、文件关联启动、外部文件变更监听（`file-watcher.js` 按窗口注册 fs.watchFile）、窗口位置恢复（`window-bounds.js` 对多显示器校验，#2282）、macOS Quick Look 预览 `.drawio` 文件（`build/notarize.mjs` 组装 .appex）、以及 `draw.io --export` 命令行批量导出 PDF/PNG/SVG 等（`src/main/args.js`，格式支持见 `validFormatRegExp`）。

第三个问题是**发布工程**：一次打标签产出 Windows x64/ARM64（NSIS/MSI/zip/AppX）、macOS（签名+公证）、Linux（AppImage/deb/rpm/Snap/Flatpak）全套安装包，并接 GitHub Releases 自动更新与 Microsoft Store 提交（`.github/workflows/electron-builder*.yml`、`electron-builder-*.json`）。

值得注意的是维护姿态：README "Not open-contribution" 一节声明项目**不接受外部 PR**（改动牵一发动全身，社区以 issue 反馈参与），但 GPL v3 允许 fork 自建，专门提供了 `doc/BUILDING_FOR_PERSONAL_USE.md` 和无秘密的 `personal-build.yml` 工作流。

## 3. 技术栈与依赖

- 运行时：Node.js >= 22.12（`package.json` engines），Electron ^44.9.3，纯 JavaScript ES6 模块（`"type": "module"`），无 TypeScript、无前端框架——壳层体量刻意极小。
- 运行时依赖（`package.json`）：`electron-updater`（GitHub provider 自动更新）、`electron-store`（设置持久化）、`electron-log`（日志）、`electron-context-menu`、`@cantoo/pdf-lib`（PDF 导出，只取 ESM 构建）、`compression`/`crc`/`buffer`/`tslib`。
- 构建依赖：`electron-builder` ^26.15.7（五份平台配置）、`@electron/fuses`（安全 fuse，`build/fuses.mjs`）、`@electron/notarize`（macOS 公证）、`quicklookjs`（Quick Look 扩展）、`sumchecker`、`dotenv`。
- CLI 参数解析为手写实现（`src/main/args.js`，`CLAUDE.md` 明确"no commander"）。
- 测试：Node 内置 `node --test`，无第三方测试框架。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 | 出处 |
|---|---|---|
| `src/main/electron.js` | 主进程全部逻辑（4573 行）：窗口、IPC、菜单、更新、CSP、路径授权 | `src/main/` |
| `src/main/electron-preload.js` | contextBridge IPC 桥（130 行）+ about: 导航拦截 | 同上 |
| `src/main/args.js` 等 13 个小模块 | CLI 参数/导出/备份/路径授权/文件监听/更新模式与渠道/打印/URL 参数等单一职责模块 | 同上 |
| `src/test/*.test.js` | 12 个单元测试文件（约 2800 行），Node 内置 runner | `src/test/` |
| `drawio/` | 子模块：Web 编辑器核心（tarball 中为空目录，本仓库不含其代码） | `.gitmodules` |
| `build/` | 打包资源：图标、fuses.mjs、notarize.mjs、sign-trusted.mjs、msi-project-created.mjs、Quick Look 模板与 entitlements | `build/` |
| `doc/` | RELEASE_PROCESS.md（429 行）、BUILDING_FOR_PERSONAL_USE.md（254 行） | `doc/` |
| `electron-builder-*.json`（5 份） | win / win-arm64 / linux-mac / snap / appx 各自的打包配置 | 仓库根 |
| `.github/workflows/`（8 份） | 发布、签名、商店提交、校验和、fork 构建、stale 等 CI | `.github/` |

组织方式评述：典型的"薄壳 + 子模块"单包结构。壳侧代码刻意拆得很碎——`src/main/` 下除 electron.js 外每个模块都在 500 行以内且一一对应同名测试文件（`package.json` 的 test 脚本逐个点名测试文件，新增测试必须登记）。两仓库通过 `drawio/VERSION` + `sync.cjs` 打通版本号。

## 5. 核心架构与关键实现

### 5.1 主进程与安全 IPC（src/main/electron.js, electron-preload.js）

渲染进程完全沙盒化：`contextIsolation: true, nodeIntegration: false, webviewTag: false, webSecurity: true`（`createWindow` 选项，electron.js 541-546 行附近）。preload 用 contextBridge 只暴露一个 `electron.request({action, data}, callback, error)` 请求/响应原语和只读 `process.versions`（`electron-preload.js` 87-123 行）。

每个 IPC 入口先用 `validateSender(frame)` 校验发送方 frame 的 URL 以 `codeUrl` 开头（即只接受本地 app 代码发起的请求；electron.js 511-517 行）。主进程在 `onHeadersReceived` 上强制 CSP：`default-src 'self'; script-src 'self' 'wasm-unsafe-eval'; connect-src 'self'...`（electron.js 829-838 行附近）。

但作者明确不满足于 validateSender——CLAUDE.md 写道"a renderer XSS passes it"，因此文件类 IPC 还有一层**路径授权**：

```js
// src/main/electron.js（239-248 行附近）
// Paths the user has authorised through trusted UI (file picker, file
// association, command line). The renderer is not allowed to write to
// anything else, even via IPC handlers that pass validateSender.
const BLESSED_PATHS_KEY = 'blessedPaths';
const BLESSED_PATHS_MAX = 500;
const blessedPaths = new Set();
```

`blessPath` 同时写入 `path.resolve` 结果和 `fs.realpathSync.native` 结果（两者必须一致地用 native realpath，否则映射盘/ subst 盘文件会被全部拒绝，#2559）；写入走 `assertWritablePath`，读取/stat/监听走 `assertReadablePath`（额外放行配置声明的本地库/模板路径），都先规范化再比较，符号链接无法逃逸。配置路径授权只在窗口首次 `app-load-finished` 时读一次 localStorage 配置，且必须在主进程对话框中用户确认（`src/main/config-paths.js`），因为配置本身是渲染进程可写的状态。

历史上三次真实漏洞驱动了这些防御（出处 `CLAUDE.md`，对应 `src/main/export-files.js`、`src/main/backup-file.js`、`src/main/system-path.js`）：CLI 导出不跟随非用户输入的符号链接（GHSA-2w35-fgjm-2vvh）；GUI 备份改为独占临时文件 + rename，防止授权后植入的链接重定向写目标（GHSA-36x5-vw5q-29rv）；Windows 子进程一律用 System32 绝对路径且不经过 shell，防止"从资源管理器打开图时工作目录被劫持"（GHSA-qg46-52fx-h7p8）。`url-params.js` 同理：urlParams.json 只在安装目录/userData 读取，cwd 仅限 dev 模式，且不许覆盖镜像主进程设置的参数。

### 5.2 壳与 Web 核心的边界（sync.cjs + drawio/ 子模块 + 打包 files 裁剪）

壳通过 URL 参数把能力注入 Web 编辑器（`queryObj` 含 `appLang`、`autoUpdate`、`dev` 等，createWindow 内 `url.format` 加载 `codeDir/index.html`）；编辑器侧对应 `drawio/src/main/webapp/js/diagramly/ElectronApp.js` 与 `DesktopLibrary.js`（`CLAUDE.md`、`electron-builder-win.json` files 数组反向保留这两个文件，因为 `bootstrap.js` 直接加载它们）。

版本同步由 29 行的 `sync.cjs` 完成：读取 `drawio/VERSION`，写入 `package.json` 的 version，并按命令行参数生成 `src/main/disableUpdate.js`（fork 自建版可用 `npm run sync -- disableUpdate` 关掉自动更新，避免被官方版覆盖）。CI 侧有个关键细节：发布工作流会检出**私有的** jgraph/drawio-dev 仓库 release 分支，把其中的 `*.min.js` 和 `VERSION` 覆盖进公开子模块树后再构建（`.github/workflows/electron-builder.yml` "Stage drawio-dev release into public submodule" 步骤）——这样发布可以领先于公开 drawio 标签，而无需改动 sync 脚本。

打包时 files 数组裁掉约 80MB 永不加载的内容（stencils XML、shapes、未压缩源码、*.map 等，`electron-builder-win.json`），五份配置需人工保持同步（`CLAUDE.md`）。

### 5.3 按安装方式分级的自动更新（src/main/update-mode.js, update-channel.js）

更新不是一刀切，而是按"哪种安装方式只能由它自己的安装器原地升级"判定（`update-mode.js` 注释引用 #2142、#2569）：

```js
// src/main/update-mode.js
if (windowsStore || env.SNAP != null || exists('/.flatpak-info')) return 'off';
if (platform === 'darwin') return 'auto';
if (platform === 'win32') {
    if (env.PORTABLE_EXECUTABLE_FILE != null) return 'notify';
    return exists(path.win32.join(path.win32.dirname(execPath),
        'Uninstall ' + productName + '.exe')) ? 'auto' : 'notify';
}
```

即：Store/Snap/Flatpak 由商店负责（off）；NSIS/macOS/AppImage 后台静默下载、退出或"Restart to Update"时安装（auto）；MSI/zip/便携/deb/rpm 只提示新版本并给 release 页链接（notify）——因为用 NSIS 安装包去覆盖这些安装会留下无人管理的残留文件。更新源为 GitHub Releases（`autoUpdater.setFeedURL({provider: 'github', ...})`，electron.js 1925 行附近），启动时与每小时各查一次；Windows ARM64 走独立的 `latest-arm64.yml` 渠道以避免与 x64 的 `latest.yml` 冲突（`update-channel.js`，`npm test` 把两个文件名钉在一起防回归）。

### 5.4 发布工程与签名（build/*.mjs + .github/workflows/）

Windows 用 Azure Trusted Signing（`build/sign-trusted.mjs` 挂在 electron-builder 的 signtoolOptions.sign 钩子，CI 中以 AZURE_TENANT_ID/CLIENT_ID/CLIENT_SECRET 认证，`.github/workflows/electron-builder-win.yml` 75-111 行），且 `win.signExts: [".dll"]` 连 Electron 的 DLL 一起签——未签名 ffmpeg.dll 会触发 Defender 的 ASR 勒索软件规则（#2509）。macOS 流程：afterPack 先打安全 fuse，afterSign 再组装 Quick Look .appex（用沙盒 entitlements 单独签名，Electron helper 本体不能沙盒）、重签外层 .app 并公证（`build/notarize.mjs`、`CLAUDE.md`）。AppX 从不出现在 GitHub Release：`release-appx` 用 `--publish never`，由 CI 的 microsoft-store job 以 msstore CLI 提交到 Partner Center（id 9MVVSZK43QQW）。发布流程文档 `doc/RELEASE_PROCESS.md` 定义了角色、审批、回滚与事故报告，并给出 Critical/Standard/Security 三档手工测试用例——对一个开源壳项目而言流程化程度相当罕见。

## 6. 构建与运行

来自 `README.md`、`CLAUDE.md`、`package.json`：

```bash
git clone --recursive https://github.com/jgraph/drawio-desktop.git  # 必须递归取子模块
npm install
npm run sync            # 用 drawio/VERSION 覆写 package.json 版本（构建前必须）
npm start               # 运行；DRAWIO_ENV=dev 开 DevTools；npm start --enable-logging 调试
npm test                # node --test 运行 src/test/ 下 12 个测试文件
npm run release-win          # Windows x64（NSIS+MSI）
npm run release-win-arm64    # Windows ARM64
npm run release-linux        # Linux（AppImage/deb/rpm/macOS 同配置文件）
npm run release-appx         # Microsoft Store 包（--publish never，由 CI 提交）
npm run release-snap         # Snap
# fork/个人构建：DRAWIO_UNSIGNED=true + electron-builder --publish never
```

不运行项目构建/测试（本分析为静态阅读），命令均取自工程文件与文档原文。

## 7. 测试与工程质量

- 单元测试：12 个文件约 2800 行（`src/test/`，`wc -l` 统计），全部用 Node 内置 `node --test`，覆盖点与安全修复一一对应：CLI 参数解析（831 行，最大）、备份符号链接回归（467 行）、路径授权（351 行）、导出符号链接（291 行）、文件监听、窗口边界、打印纸张、urlParams、更新渠道/模式、MSI 快捷方式钩子、System32 路径。`package.json` 的 test 脚本逐文件点名，防止新测试漏登记。
- 更大部头靠**手工测试**：`doc/RELEASE_PROCESS.md` 第 6 节按 Critical/Standard/Security 三档列出发前必测用例；无 e2e 自动化（`CLAUDE.md`："Everything else is manual testing"）。
- 无 ESLint/Prettier，代码风格靠人工维持（Allman 大括号、Tab 缩进，`CLAUDE.md` Code Style 节）。
- CI 质量观察：8 个工作流分工明确，含每日 cron 补传校验和（`hash-gen.yml`——GitHub 自 2024 年 12 月起丢 release 事件，注释里写明）、只读的商店凭据检查（`store-credentials-check.yml`）。整体印象：壳很小，但安全细节与发布纪律是重度打磨的。

## 8. 社区与维护状态

- 维护方：JGraph/draw.io 官方团队，明确不接受外部 PR（`README.md` "Not open-contribution"）；社区参与形式为 issue 与 feature request（`.github/` 有对应模板、`stale.yml` 自动标记陈旧 issue）。
- 活跃度：默认分支 dev，最近 push 2026-10-26（分析日 5 天前）；最新 release v32.6.1 发布于 2026-10-23，含 29 个安装包资产（gh api）。以每版三段版本号、约每月多次的节奏发版（版本已到 32.x）。
- 贡献者结构：无官方 PR 流，代码基本由内部维护者产出；未做贡献者统计（未确认）。2026 年的活跃工程决策包括：07/2026 移除第三方插件机制、Electron 44 后砍掉 32 位 Windows 与 macOS 12 支持（`CLAUDE.md`）。
- 商业背景：JGraph 以 draw.io for Confluence/Jira 为商业产品，桌面版免费但无契约化支持（`README.md` Support 节）。

## 9. 亮点与风险

亮点：
1. 安全设计是教科书级的纵深：validateSender 之上再加路径授权（blessed paths + native realpath）、配置路径一次性读取、三份 GHSA 驱动的符号链接/子进程加固，且每个修复都有对应回归测试（`CLAUDE.md`、`src/main/`、`src/test/`）。
2. 更新模式按安装方式精确分级（auto/notify/off），把"谁能升级谁"这个 Electron 应用常见乱源讲清楚并写成可测代码（`src/main/update-mode.js`）。
3. 发布工程成熟：五平台打包、Azure Trusted Signing + macOS 公证、MS Store 自动提交、私有 dev 仓库领先发布的干净机制（CI 覆盖子模块内容）、连"GitHub 丢 release 事件"都有 cron 兜底。
4. `CLAUDE.md` + `doc/RELEASE_PROCESS.md` 构成的文档密度极高，几乎每条设计决策都带 issue 号，源码可读性和可审计性在同类 Electron 项目中罕见。
5. 对 fork 用户友好：无秘密的 personal-build 工作流、签名豁免开关、禁更新开关一应俱全。

风险/局限：
1. 闭门开发：不接受 PR，外部只能提 issue，社区修复速度受制于官方带宽；fork 长期跟进上游需自行合并（`README.md`）。
2. GPL v3 + Apache 核心的混合许可在 v31.5.2 变更过，商用再分发需重新评估（`package.json`、`CLAUDE.md`）。
3. 五份 electron-builder 配置靠人工保持同步（`CLAUDE.md` 自己标注"Keep the five configs in sync"），存在漂移风险。
4. 除单元测试外全靠手工测试清单，UI/集成层回归依赖发布纪律而非自动化（`doc/RELEASE_PROCESS.md`）。

## 10. 总结评价

drawio-desktop 是"壳虽小、功夫深"的典范：4600 行主进程加 13 个单一职责模块，把 Electron 应用最容易做糙的四件事——渲染进程隔离、文件访问授权、按安装方式分级的更新、跨平台签名发布——都做到了工业强度，且用极密的设计文档和回归测试把每个安全决策钉死。星数 63k 与其说是代码量的胜利，不如说是 draw.io 本体生态（离线、安全、免费）加上多年发布可靠性的复利。适合三类人：想把 Web 编辑器/IDE 包装成安全桌面应用并学习其安全模型的 Electron 工程师（第 5.1/5.3 节代码最值得精读）；需要内网离线画图的终端用户（直接用 release 安装包）；以及研究开源项目发布工程与闭门维护模式的人。想读 Web 编辑器内核的人则应转向 jgraph/drawio 本仓库，这里的壳代码不包含绘图逻辑。
