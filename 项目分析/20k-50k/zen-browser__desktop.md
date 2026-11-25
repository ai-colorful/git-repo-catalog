# zen-browser/desktop 项目分析

> 星数 39000（快照 2026-10-24，gh api 2026-11-05 实测 45.6k）｜主语言 JavaScript (CSS/Shell)｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：Zen Browser 是一个基于 Firefox 的"无偏见浏览器"，主打生产力布局（工作区 Spaces、分屏、垂直标签、Compact Mode）与开箱即用的隐私（锁定遥测），仓库本体是 Firefox 之上的补丁/品牌/构建层。

| 事实 | 值 | 出处 |
|---|---|---|
| 名称 | Zen Browser (`zen-desktop`) | `package.json`、`surfer.json` |
| 星数 | 45,568（2026-11-05 gh api；任务快照 39k） | gh api repos/zen-browser/desktop |
| 语言 | JavaScript/MJS + CSS，少量 C++/Rust/Python/Shell | `src/zen/**` 统计 |
| 许可证 | MPL-2.0 | `LICENSE`、`package.json` |
| 基于 | Firefox 159.0.1（Release 通道显示版本 1.24.1b） | `surfer.json` |
| 默认分支 | `dev`（另有 `stable`、`l10n_dev`、`perf/*`） | gh api branches、`docs/contribute.md` |
| 最近推送 | 2026-11-04（活跃） | gh api pushed_at |
| 贡献者 | 282 人 | gh api contributors 分页头 |
| 创建时间 | 2024-03-28 | gh api created_at |

## 2. 项目是什么，解决什么问题

Firefox 长期以横排标签、单窗口单任务流为主，重度多标签用户长期依赖第三方扩展拼凑"工作区 + 垂直标签"体验；Arc 证明了"浏览器即工作台"的形态有价值，但闭源且已停止迭代。Zen 选择一条务实路线：不重写内核，而是把 Firefox 完整拉下来、打上自己的 UI/行为补丁、换掉品牌与默认隐私策略，产出可直接安装的 Zen 浏览器（README.md；`surfer.json`）。

仓库并不是 Firefox 的 fork 镜像，而是一个**补丁层仓库**：Firefox 源码（仓库内称 `engine/`）由自研构建工具 surfer 按版本号（`surfer.json` 中 `product: firefox, version: 159.0.1`）下载，再由 `surfer import` 把本仓库 `src/` 下的改动套用到引擎上（`package.json` scripts）。因此 `src/` 目录结构刻意模仿 Firefox 源码树（`browser/`、`toolkit/`、`dom/`、`layout/`、`widget/`、`gfx/` 等），其中以 `.patch` 文件（全仓 273 个）修改 Firefox 原生代码，而 Zen 自己新增的功能代码集中放在 `src/zen/` 一个目录里，通过 Firefox 的 moz.build 构建系统接入（`src/zen/moz.build`、各子目录 `moz.build`）。

产品层面 Zen 提供两大渠道：`release`（稳定版 1.24.1b）与 `twilight`（近似 nightly，显示版本 1.25t，每 2 天定时构建，`.github/workflows/twilight-release-schedule.yml` cron `0 22 */2 * *`）。更新走自建服务器 `updates.zen-browser.app`，以 MAR 差分包形式分发（`surfer.json` archives 字段：windows/macos-aarch64/linux/windows-arm64/linux-aarch64/macos-x64 六平台 MAR；`scripts/mar_sign.sh`、`mar_verify.sh` 负责签名校验）。

隐私是第二大卖点：`prefs/privatefox/privacy.yaml` 将 `datareporting.*`、`toolkit.telemetry.*`、`toolkit.coverage.*` 等十余项遥测偏好置 false 且 `locked: true`（遥测服务器直接指到 `"data:,"`），`prefs/privatefox/disablemozilla.yaml` 关闭 Mozilla 推广/发现页/VPN 导流等；`prefs/fastfox/`（滚动性能）与部分偏好提取自 Betterfox 项目（`prefs/README.md` 自述）。

## 3. 技术栈与依赖

- **基座**：Firefox 159.0.1 源码（C++/JS/Python 混合，moz.build 构建体系），由 `@zen-browser/surfer ^1.14.9` 下载并管理（`package.json` devDependencies、`surfer.json`）。
- **Zen 功能代码**：浏览器前端 JS（ES Module，`src/zen/**` 约 77,400 行 .js/.mjs）+ CSS（约 13,800 行），需要碰引擎层时写 C++（约 3,400 行 .cpp/.h，如 `src/zen/mods/nsZenModsBackend.cpp`）与 XPIDL 接口（`src/zen/mods/nsIZenModsBackend.idl`）。
- **Rust 工具**：`tools/ffprefs`（Cargo 项目，`cargo run --bin ffprefs`）把 `prefs/*.yaml` 编译进浏览器默认偏好；`rust-toolchain.toml` 固定工具链。
- **Python 脚本**：`scripts/update_ff.py`（同步 Firefox 版本/RC 跟踪）、`scripts/update_external_patches.py`（从 Mozilla Phabricator 下载补丁）、`scripts/run_tests.py`（测试入口）等。
- **前端工程化**：TypeScript 5.9、Babel、ESLint（`eslint.config.mjs` 近 15KB，含 spidermonkey-js 自研插件 `tools/eslint-plugin-spidermonkey-js/`）、husky + lint-staged；`src/zen/@types/`（2.5MB）是由 `scripts/update_ts_types.py` 生成的 Gecko 平台 TS 类型定义。
- **本地化**：Crowdin 托管，`crowdin.yml` + `locales/` 45 个语言目录。
- **CI**：GitHub Actions，自托管 blacksmith runner，支持 sccache 与 PGO（`.github/workflows/*-release-build.yml`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `src/zen/` | Zen 自研功能核心：spaces、split-view、glance、boosts、mods、folders、live-folders、tabs、compact-mode、welcome、migration、sessionstore、kbs、urlbar、sync 等约 30 个功能模块 + tests |
| `src/browser/` `src/toolkit/` 等 | 对 Firefox 对应源码目录的就地修改，以 `.patch` 文件形式存放（如 `src/browser/base/content/` 19 个补丁） |
| `src/external-patches/` | 外来补丁：manifest.json 声明从 Mozilla Phabricator（D299584 等）与 LibreWolf（codeberg）拉取的临时补丁 |
| `prefs/` | YAML 偏好定义：`zen/`（27 个功能偏好文件）、`firefox/`、`privatefox/`、`fastfox/` |
| `configs/` | `common/mozconfig`、`branding/release|twilight`（图标/安装器资源）、各 OS 构建配置 |
| `scripts/` | 版本同步、补丁更新、MAR 签名/校验、Flatpak 发布、语言包、测试入口 |
| `tools/` | `ffprefs`（Rust）、`eslint-plugin-spidermonkey-js`、`virustotal-checker` |
| `build/` | ts 构建产物、`flatpak/`、`windows/` 安装器、`signing/`、`firefox-cache/` |
| `locales/` | 45 语言翻译 |
| `.github/workflows/` | 三平台 release 构建、twilight 定时构建、上游同步、PR 测试 |

组织方式评述：这是典型的"上游补丁 + 自有目录"双轨组织——能用纯前端（chrome:// 页面、JS 模块、CSS）实现的都收进 `src/zen/` 自成体系；必须改 Firefox 内核行为的则以 `.patch` 散落在镜像目录中，由 surfer 在 import 时套用。模块间通过 `moz.build` + `jar.inc.mn`（chrome 资源注册）接入 Firefox 构建系统，本质上是寄宿在 Firefox 单体构建内的"子单体"。

## 5. 核心架构与关键实现

### 5.1 Spaces（工作区）——`src/zen/spaces/`
核心类 `nsZenWorkspaces`（`src/zen/spaces/ZenSpaceManager.mjs`，约 330KB 目录）管理空间的生命周期、当前空间、空间间切换动画（`ZenSpacesSwipe.mjs`，触控板滑动手势 + `nsZenSwipeOverrides.cpp` 引擎侧覆盖）、空间级标签隔离与书签存储（`ZenSpaceBookmarksStorage.js`）、空间路由（`src/zen/space-routing/` 可让指定域名自动落入某空间）。空间图标渐变背景由 `ZenGradientGenerator.mjs` 生成。关键代码风格是 Firefox 惯用的 lazy getter 注入：

```js
ChromeUtils.defineESModuleGetters(lazy, {
  ZenKeyboardShortcuts: "resource:///modules/zen/kbs/ZenKeyboardShortcuts.sys.mjs",
  ZenSessionStore: "resource:///modules/zen/ZenSessionManager.sys.mjs",
});
```

预加载脚本 `src/zen/common/ZenPreloadedScripts.js` 统一注册约 20 个 Zen 管理器（ZenSpaceManager、ZenViewSplitter、ZenMods、ZenGlanceManager、ZenPinnedTabManager 等），是整个前端层的组装点。

### 5.2 Boosts 与 Mods——用户样式/脚本注入体系
`src/zen/boosts/`（编辑器 `ZenBoostsEditor.mjs` 1,813 行、管理器 `ZenBoostsManager.sys.mjs` 957 行、Zap 即时元素隐藏 `ZenZapDissolve.sys.mjs`）允许用户对任意网站注入 CSS/JS，背后有 C++ 后端 `nsZenBoostsBackend.cpp`；`src/zen/mods/` 是主题 Mods 系统，含 `nsZenModsBackend.cpp`、`nsIZenModsBackend.idl`、`ZenStyleSheetCache.cpp`（C++ 层缓存用户样式表）。两者共用"前端编辑器 + XPIDL 桥 + 引擎后端"的三层结构。

### 5.3 Split View（分屏）——`src/zen/split-view/ZenViewSplitter.mjs`
自实现二叉分割树：`nsSplitLeafNode`/`nsSplitNode` 记录 `sizeInParent`、`direction: row|column`、`positionToRoot`，以百分比描述每个叶节点（标签页）在父节点中的尺寸，拖拽分隔条即调整兄弟叶子的 `sizeInParent`。纯前端实现，不依赖 Firefox 原生分屏。

### 5.4 偏好管道（Rust ffprefs + YAML）
`prefs/**/*.yaml` → `tools/ffprefs/src/main.rs` 解析 → 编译为 Firefox 默认偏好。文件格式为条目列表：

```yaml
- name: toolkit.telemetry.unified
  value: false
  locked: true
```

这使品牌级偏好（遥测关闭、Zen 功能开关）可审查、可 diff，构建期而非运行期生效（`prefs/README.md`、`tools/ffprefs/`）。

### 5.5 外部补丁管理
`src/external-patches/manifest.json` 声明补丁来源（Phabricator ID / 任意 URL / 本地路径）及 `replaces` 文本替换规则，`scripts/update_external_patches.py` 负责下载；README 明确"这些是向前移植的 Firefox 未来版本补丁，上游发布后即删除"。Zen 还直接复用 LibreWolf 的 `firefox-in-ua.patch`，体现 fork 间的补丁互助。

## 6. 构建与运行

来自 `package.json` scripts 与 `docs/contribute.md`（详细指南在 docs.zen-browser.app/guides/building）：

```bash
npm run download   # surfer 下载 Firefox 源码到 engine/
npm run import     # ffprefs 编译偏好 + 更新 service dumps + surfer import 套用补丁
npm run bootstrap  # surfer bootstrap
npm run build      # surfer build（--ui 仅前端增量）
npm start          # cd engine && python3 ./mach run --noprofile
npm run package    # surfer package 出安装包
npm run sync       # scripts/update_ff.py 跟随新 Firefox / --rc 跟踪 RC
npm test           # python3 scripts/run_tests.py
npm run lint       # cd engine && ./mach lint zen
```

分支流程（`docs/contribute.md`）：`dev`（默认/开发）→ `stable`（发布，可直挂热修）；twilight 为每 2 天定时构建的预览渠道。

## 7. 测试与工程质量

- **测试规模**：`src/zen/tests/` 共 797 个文件，主体是 Firefox mochitest/browser-chrome 风格的 `browser_*.js`（spaces、split_view、glance、folders、kbs、migration 等各功能目录均有），manifest 为 `manifest.toml`；另有 gtest（`src/zen/boosts/gtest/`）。命名可见 `browser_issue_10455.js` 这类按 issue 回归的测试，覆盖意识好。
- **测试入口**：`npm test` → `scripts/run_tests.py`（支持 `--jsdebugger --debug-on-failure`），gtest 走 `./mach gtest Zen*`（`package.json`）。
- **CI**：PR 有 `pr-test.yml`；发布构建按平台拆分（`windows/macos/linux-release-build.yml`，macOS 还有 universal 与 profile 构建），支持 sccache 缓存与 PGO（可禁用/重生成 PGO profile，build.yml inputs）；另有 `sync-upstream.yml`（手动触发的 Firefox 版本同步）。
- **类型**：2.5MB 自动生成 Gecko 类型定义（`src/zen/@types/lib.gecko.*.d.ts`），前端代码在 VSCode 里有完整平台 API 补全，工程质量明显高于一般 fork。

## 8. 社区与维护状态

- **活跃度**：gh api 显示最近推送 2026-11-04（前一天），828 open issues，1,960 forks，282 贡献者（含匿名）。
- **发布节奏**：2026 年 10 月连续发布 1.23b（10-02）、1.23.1b（10-06）、1.23.2b（10-09）、1.23.3b（10-16）、1.23.4b（10-20）、1.24b（10-29）——约每月一个 minor、随修随发的点版本；twilight 每 2 天自动构建（`.github/workflows/twilight-release-schedule.yml`）。
- **跟随 Firefox**：`scripts/update_ff.py` 自动跟踪 Firefox RC（`check_rc_response.py`），新 RC 出现即重建 engine；`surfer.json` 显示当前基线 159.0.1，README 口径为 158.0（略滞后于 surfer.json）。
- **本地化**：Crowdin 徽章 + 45 语言目录，有独立 `l10n_dev` 分支。
- **商业化痕迹**：赞助商 Blacksmith（CI runner）出现在 README 与 workflows（`blacksmith-*` runner），更新服务器为自建域名。

## 9. 亮点与风险

**亮点**
1. 补丁层架构清晰：自有功能全部收敛在 `src/zen/`（近 9.5 万行 JS/CSS/C++），对 Firefox 的侵入以 273 个 `.patch` 隔离，跟随上游重基线的成本被 surfer 工具化（`surfer.json`、`package.json`）。
2. 隐私默认可信：遥测/Mozilla 推广在构建期锁定关闭（`prefs/privatefox/*.yaml`），且全部可审查。
3. 功能原创度高：Split View 分割树、Glance 临时预览、Boosts 站点级注入、Spaces 路由等均为自研实现，非扩展拼装。
4. 工程化扎实：Rust 偏好编译器、生成式 TS 类型、mochitest 回归测试、PGO/sccache CI、MAR 签名校验、六平台产物。
5. 社区生态位明确：借 Phabricator 前置补丁与 LibreWolf 补丁互通，形成 Firefox fork 间的协作网络（`src/external-patches/`）。

**风险/局限**
1. 生存线系于 Firefox 上游：每次 Firefox 大版本变更都可能造成补丁失效，`src/external-patches/firefox/README.md` 自己也承认临时补丁"新版本上失败就该删"。
2. 维护强度要求极高：273 个补丁 + 45 语言 + 6 平台构建，核心负担集中在少数维护者（282 贡献者中长尾贡献有限，未确认具体集中度）。
3. 仓库不含 Firefox 源码，源码审计需先 `surfer download` 数 GB engine，本地构建门槛高（需 Rust/Python/各平台工具链）。
4. 自建更新通道 `updates.zen-browser.app` 是供应链单点（MAR 有签名校验缓解，`scripts/mar_verify.sh`）。

## 10. 总结评价

Zen Browser 是目前 Firefox fork 生态中工程质量与产品完成度最高的项目之一：它没有选择"改改默认配置换个皮"，而是自研 surfer 补丁流水线，在 Firefox 之上做出了 Spaces/分屏/Glance/Boosts 等成体系的原创能力，同时用构建期锁定的隐私偏好兑现"bias-free"承诺。四万余星、月度发布、双日 twilight、282 位贡献者，证明了这一重投入模式在社区驱动下可持续。适合想要"开箱即用的 Arc 替代品"的普通用户，适合研究"如何维护一个大型 Firefox fork"的工程师（`src/zen/` 与 `prefs/` 是最佳入口），也适合想给浏览器加原生功能的贡献者——从 `src/zen/tests/` 的 mochitest 学起即可上手。
