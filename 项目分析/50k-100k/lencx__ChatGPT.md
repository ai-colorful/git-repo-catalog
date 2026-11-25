# lencx/ChatGPT 项目分析

> 星数 54656（快照 2026-10-24）｜主语言 Rust｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：ChatGPT 网页的桌面壳封装（Mac/Windows/Linux），2022 年底 ChatGPT 爆发期最早、最知名的第三方桌面客户端之一；当前默认分支是停更前的 v2 重写原型，v1 正式版在 `release-v1.1.0` 分支。

| 项 | 值 |
| --- | --- |
| 仓库 | lencx/ChatGPT（未 fork） |
| 星数 | 54656（快照）；GitHub API 实测 54669（2026-11-03） |
| 主语言 | Rust（Tauri）+ TypeScript/React |
| 许可证 | AGPL-3.0（`src-tauri/Cargo.toml` 声明；v1 分支有 LICENSE 文件，当前默认分支 v2-dev 无 LICENSE 文件、仓库级 license 为 null） |
| 创建时间 | 2022-12-07（gh api `repos/lencx/ChatGPT`） |
| 最近 release | v1.1.0（2023-08-03，共 34 个 release，首个 v0.1.4 为 2022-12-07） |
| 最近 push | 2024-08-29（gh api `pushed_at`） |
| 默认分支 | v2-dev（本次分析的 tarball 即该分支 HEAD） |
| 维护状态 | 停更（archived 标志为 false，但 2024-08 后无任何提交，957 个 open issue 无人处理） |

## 2. 项目是什么，解决什么问题

ChatGPT 刚上线时（2022 年 12 月）只有网页版，官方桌面应用要到 2024 年 5 月才发布（README 中引用了 OpenAI 的公告链接：`README.md` "OpenAI has now released the macOS version of the application"）。本项目的价值就是把这个网页包成原生桌面应用：独立窗口、系统托盘级的常驻体验、全局快捷键、窗口置顶、明暗主题跟随系统、多平台打包安装包。它赶上并放大了"给ChatGPT 套壳"这个品类，是 2023 年初最出圈的 ChatGPT 桌面客户端（星数主要在 v1 时期积累）。

v1 版本（`release-v1.1.0` 分支）功能相当丰富：自定义 prompt 库、斜杠命令、弹窗搜索、导出聊天记录等。本次分析的默认分支 v2-dev 则是一次"推倒重来"：作者在 README 明确说明 OpenAI 出官方应用后，这个项目失去了差异化空间，因此转向寻找新方向，v2 只保留最小骨架（把网页装进多 webview 窗口 + 底部 Ask 输入条），并把精力投向继任项目 Noi（`README.md`："try the Noi, which is a successor to the ChatGPT desktop application concept"）。v1 的旧版下载入口也只在 README 里以分支链接形式保留。

从代码形态看，v2 HEAD 是一个"概念验证壳"：核心思路从 v1 的"大量注入脚本改造网页"收敛为"Tauri 2 多 webview 拼装"——一个窗口内叠三个 webview（自绘标题栏 / chatgpt.com 主视图 / 底部 Ask 输入条），Rust 侧只做布局、配置持久化和少量命令转发。它更适合作为"Tauri 2 多 webview 桌面壳"的最小参考实现来读，而不是功能完整的 ChatGPT 客户端。

需要如实记录的现状：项目实质已死（最后提交 2024-08-29，issue 积压 957 个），README 用醒目 NOTE 引导用户转向 Noi。它今天的意义主要是历史标本 + Tauri 2 早期采用者的工程样本。

## 3. 技术栈与依赖

Rust 侧（`src-tauri/Cargo.toml`、根 `Cargo.toml`）：
- tauri 2.0.0-beta（开启 `unstable`、`devtools` feature——`unstable` 正是启用多 webview 的开关），tauri-plugin-shell/dialog/os 2.0.0-beta
- serde/serde_json（配置序列化）、once_cell、anyhow、log、regex、semver（脚本版本比对）、tokio（macros）、dark-light（系统明暗检测）
- 工作区单成员 `src-tauri`；release profile：`opt-level="s"`、`lto=true`、`panic="abort"`、`codegen-units=1`（根 `Cargo.toml`）
- rust-version 1.77.1，edition 2021

前端（`package.json`，pnpm）：
- React 18.3 + Vite 5 + TypeScript 5.4 + Tailwind 3.4（`tailwind.config.js`、`postcss.config.js`）
- `@tauri-apps/api` 2.0.0-beta.15（多 webview API）、plugin-os/shell
- react-hotkeys-hook（Ask 输入条快捷键）、lodash（debounce）、clsx、react-router-dom（声明了依赖，但 `src/App.tsx` 实际按 webview label 切换视图，未见使用 router——以代码为准）

语言占比与规模：Rust 9 个文件约 837 行（`src-tauri/src/`），TS/TSX 约 550 行（`src/`），全仓 74 个文件（含图标与锁文件）。是个小仓。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `src-tauri/src/main.rs` | Tauri Builder 入口：注册 3 个插件 + 11 个命令 |
| `src-tauri/src/core/setup.rs` | 启动核心逻辑：建窗口、拼装 3 个 webview、布局与下载处理（258 行，最大模块） |
| `src-tauri/src/core/cmd.rs` | 全部 Tauri command（刷新/前进后退/置顶/Ask 同步发送/主题/读取配置） |
| `src-tauri/src/core/conf.rs` | AppConf 结构与 config.json 读写、合并、主题解析 |
| `src-tauri/src/core/template.rs` | 内置脚本 ask.js 的落盘与 semver 版本化更新 |
| `src-tauri/src/core/constant.rs` | 常量：标题栏 28px、Ask 条 120px、INIT_SCRIPT 注入脚本 |
| `src-tauri/src/core/window.rs` | 打开设置窗口的命令（17 行） |
| `src-tauri/scripts/ask.js` | 注入 chatgpt.com 的桥接脚本（ChatAsk 类） |
| `src-tauri/capabilities/desktop.json` | Tauri 2 capability：对 `https://chatgpt.com/*` 远程页授权 |
| `src/` | 前端（React）：App.tsx 按 webview label 分发到 Titlebar/Ask/Settings 三个视图；icons/ 为内联 SVG 组件 |
| `docs/static/` | 演示 GIF；`public/` 应用图标 |

组织方式评述：单体应用、双层结构（Rust 壳 + React 前端），前端同一个 `index.html` 被三个不同 label 的 webview 复用，由 `src/App.tsx` 的 `viewMap` 按 label 渲染不同组件——这是 Tauri 2 多 webview 模式下一个很紧凑的组织法。无测试目录、无 CI 配置（v2-dev 分支无 `.github/`，已用 gh api 404 核实）。

## 5. 核心架构与关键实现

### 5.1 三 webview 拼装窗口（`src-tauri/src/core/setup.rs`）

应用只有一个 "core" 窗口，内部手工叠放三个 webview：`main`（加载 `https://chatgpt.com`）、`titlebar`（本地 index.html，自绘标题栏）、`ask`（本地 index.html，底部输入条）。macOS 走 `TitleBarStyle::Overlay` 保留红绿灯，Windows/Linux 则完全自绘：

```rust
let main_view =
    WebviewBuilder::new("main", WebviewUrl::App("https://chatgpt.com".into()))
        .auto_resize()
        .on_download({ /* 下载事件：改写到系统下载目录，完成后 shell 打开 */ })
        .initialization_script(&AppConf::load_script(&handle, "ask.js"))
        .initialization_script(INIT_SCRIPT);
```

窗口 `Resized` 事件里对三个视图逐一 `set_position/set_size` 重新布局（macOS 与非 macOS 两套坐标分支），Ask 条显隐通过高度 0/120px（`constant.rs` 的 `ASK_HEIGHT`）切换。这个手工布局逻辑在 `setup.rs` 与 `cmd.rs::set_view_ask` 中重复出现两份，是代码中明显的重复。

### 5.2 网页桥接：注入脚本与命令转发（`src-tauri/scripts/ask.js`、`cmd.rs`）

Ask 条把用户输入"打"进 chatgpt.com 的表单：前端 `src/view/Ask.tsx` 以 300ms debounce 调 `ask_sync`，Rust 直接对 main webview `eval("ChatAsk.sync(...)")`；发送则 `eval` 调 `ChatAsk.submit()` 并把焦点还回 ask webview（`cmd.rs::ask_sync/ask_send`）。注入侧用 React 受控组件的经典绕法——原生 setter + 派发 InputEvent：

```js
const nativeTextareaSetter = Object.getOwnPropertyDescriptor(
  window.HTMLTextAreaElement.prototype, 'value').set;
nativeTextareaSetter.call(inputElement, message);
inputElement.dispatchEvent(new InputEvent('input', { bubbles: true }));
// submit(): 点击 main form 最后一个 button
```

ask.js 由 `template.rs` 用 `include_bytes!` 编译进二进制，启动时写入用户配置目录，并解析脚本头部 `@version` 与现存文件做 semver 比较、仅在新版本时覆盖——自带一套极简的"用户脚本更新机制"。

### 5.3 导航同步注入脚本（`src-tauri/src/core/constant.rs`）

`INIT_SCRIPT` 钩住 `pushState/replaceState/popstate`（含对 history API 的猴子补丁），把 URL 变化以 `navigation:change` 事件发给 titlebar webview，自绘标题栏据此显示当前域名、提供前进/后退/刷新按钮；同时强制所有链接 `target=_blank` 防止在壳内跳走。这是"壳"与"被包装网页"之间唯一的常规通信通道。

### 5.4 配置持久化（`src-tauri/src/core/conf.rs`）

`AppConf { theme, stay_on_top, ask_mode, mac_titlebar_hidden }` 持久化在 `config_dir/com.nofwl.chatgpt/config.json`。`amend()` 用 BTreeMap 把旧配置与新增 JSON 合并后再反序列化，容忍字段漂移；解析失败时回退默认值并回写。主题支持 system/dark/light，system 用 dark-light crate 探测（`conf.rs::get_theme`）。切换主题通过 `cmd.rs::set_theme` 保存后 `app.restart()` 整个重启生效——简单粗暴但诚实。

## 6. 构建与运行

来自 `package.json` scripts 与 `src-tauri/tauri.conf.json`（未实际运行，按工程文件记录）：

```bash
pnpm install          # 前端依赖
pnpm tauri dev        # 开发（beforeDevCommand: pnpm dev，Vite 固定 1420 端口）
pnpm tauri build      # 出三平台安装包（bundle.targets: "all"）
pnpm build            # 仅前端：tsc && vite build
```

注意：代码锁定 Tauri 2.0.0-beta 系列依赖（Cargo.toml 与 package.json 均为 beta 版），以两年后的今天复现构建大概率需要处理 beta API 变动。v1 用户的实际安装走 GitHub Releases（v1.1.0，2023-08-03）。

## 7. 测试与工程质量

- 测试：零。全仓无任何 `#[test]`、无测试目录、无 e2e（`find` 全仓无 test/spec 文件），v1 分支历史 CI 仅一个 `release.yml`（gh api 核实），v2-dev 无 `.github/`。
- 代码质量观察：Rust 侧风格统一（`rustfmt.toml` 配置），但命令层大量 `unwrap()`（`cmd.rs` 几乎每行 unwrap），webview 不存在时直接 panic；错误处理混用 anyhow/log 与 `eprintln!`（`setup.rs`、`cmd.rs`）。引入了 `log` crate 但未初始化任何 logger，日志实际无处可去。
- 工程化亮点：`capabilities/desktop.json` 对远程域 `https://chatgpt.com/*` 精细授权（window/webview/event/shell/os 权限白名单），符合 Tauri 2 的安全模型；`withGlobalTauri: true` 配合注入脚本使用 `__TAURI__` 全局对象。
- 死代码痕迹：`react-router-dom` 在依赖中但未使用；`Settings.tsx` 只有一行占位文本（4 行），v2 明显未完成。

## 8. 社区与维护状态

- 时间线（gh api 实测数据）：2022-12-07 建仓并发布 v0.1.4；2022-12 至 2023 年中是黄金期，34 个 release 里最后一个 v1.1.0 于 2023-08-03 发布；之后转向 v2 重写（默认分支 v2-dev），最后 push 2024-08-29；至今无提交。
- 贡献者：29 人（gh api contributors 首页计数），主体是单人项目（作者 lencx，`src-tauri/Cargo.toml` authors 字段），社区贡献以 issue/小修为主。
- issue 状态：957 个 open issue，停更后无人triage；README 直接引导用户转向官方应用或继任项目 Noi。
- 归档状态：仓库未打 archived 标志（gh api `archived: false`），但行为上等同弃维护，"半归档"状态。README 保留了 Buy Me A Coffee、Discord、Twitter、YouTube 等作者引流链接。
- 生态位：v1 是 2023 年初 macOS/Windows/Linux 上事实上的 ChatGPT 第三方客户端标准答案之一，star 在 2023 年初数周内破万（时间点与建仓/早期 release 时间吻合；精确增速曲线未确认）。

## 9. 亮点与风险

亮点：
1. 历史标本价值：ChatGPT 应用生态最早期的标志性壳项目，v1 的功能设计（prompt 库、弹窗搜索）是当年大量仿品的蓝本（README/UPDATE_LOG 与 release 历史可证）。
2. Tauri 2 多 webview 的极简参考实现：单窗口叠 3 个 webview、按 label 复用同一前端入口（`src/App.tsx`），代码量极小（Rust ~840 行）就能看清全部机制。
3. 注入脚本工程化：ask.js 内嵌二进制 + 用户目录落盘 + `@version`/semver 增量更新（`src-tauri/src/core/template.rs`），把"用户脚本分发"做成了机制。
4. 遵循 Tauri 2 安全模型：capability 白名单精确到远程域（`src-tauri/capabilities/desktop.json`）。
5. 作者止损干脆：官方应用出来后立刻停止重复造轮子、公开指路继任项目，README 状态诚实。

风险/局限：
1. 项目已死：核心依赖锁定 Tauri 2.0.0-beta（beta API 后续大量变更），今日无法开箱构建；chatgpt.com 的 DOM 结构一旦变化，ask.js 注入即失效。
2. 稳健性差：命令层满布 `unwrap()`，远端网页加载失败或 webview 缺失即 panic；`csp: null`（`tauri.conf.json`）关闭了内容安全策略。
3. 壳类产品固有脆弱性：功能完全依附于被包装站点，登录/风控/改版都不可控。
4. 许可证文件缺失：默认分支无 LICENSE 文件（仅 Cargo.toml 内声明 AGPL-3.0），二次分发需回查 v1 分支 LICENSE 确认条款。

## 10. 总结评价

lencx/ChatGPT 是 2023 年 ChatGPT 风口最具代表性的"套壳"桌面应用：它赢在时间与完成度，也在 OpenAI 官方客户端发布后迅速失去存在意义，最终以"v1 冻结 + v2 半成品"的状态停在 2024 年 8 月。以今天的默认分支 HEAD 来看，它已经不是一件可用产品，而是两份有价值的遗产：v1 分支的完整功能实现（值得研究"如何优雅地改造一个不受控网页"），与 v2-dev 这份约 1400 行的 Tauri 2 多 webview 最小骨架（值得作为 Tauri 2 桌面壳的入门范本，其 webview 拼装、capability 授权、注入脚本版本化都写得很清楚）。适合 who：想了解 ChatGPT 桌面壳品类起源与兴衰的产品/研究者读 README 与 release 历史；想学 Tauri 2 多 webview 布局的开发者读 `src-tauri/src/core/setup.rs`；不适合把它的代码直接拿来继续开发——正确路径是作者自己指出的继任项目 Noi。
