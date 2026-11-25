# tauri-apps/tauri 项目分析

> 星数 112142（快照 2026-10-24）｜主语言 Rust｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：用 Rust 后端 + 系统 WebView 前端构建体积小、速度快、跨五大平台（Windows/macOS/Linux/iOS/Android）桌面与移动应用的框架——Electron 的"轻量替代者"。

| 项目 | 内容 |
| --- | --- |
| 名称 | tauri-apps/tauri |
| 星数 | 112,306（gh api 实时；快照 112,142） |
| 语言 | Rust 核心 + TypeScript JS API（polyglot monorepo） |
| 许可证 | MIT OR Apache-2.0 双许可（`Cargo.toml`、`LICENSE-APACHE-2.0`/`LICENSE-MIT`；GitHub 元数据标 Apache-2.0） |
| 创建时间 | 2019-07-13（gh api） |
| 最近推送 | 2026-10-29（gh api，分析前一天，极活跃） |
| 最新 release | tauri-cli-v2.12.4（2026-10-21）；核心 crate tauri v2.12.5（`crates/tauri/Cargo.toml`） |
| 贡献者 | 约 617 人（gh api contributors 分页推断，含匿名） |
| 默认分支 | dev |

## 2. 项目是什么，解决什么问题

Tauri 是一个"polyglot 且高度可组合"的应用工具包（`ARCHITECTURE.md`），解决的核心问题是：**如何用 Web 技术写界面、又不为每个应用捆绑一整个 Chromium/Node 运行时**。它的答案是直接复用操作系统自带的 WebView——macOS/iOS 用 WKWebView，Windows 用 WebView2，Linux 用 WebKitGTK，Android 用 System WebView（`README.md`）。因此最终产物是纯 Rust 编译的二进制，体积小（几 MB 量级）、无需分发运行时，逆向成本也更高。

架构上 Tauri 明确自己"不是内核级轻封装"，重量级的窗口与 WebView 系统调用交给组织内维护的两个上游 crate：**TAO**（winit 的 fork，负责窗口/菜单/托盘）和 **WRY**（统一各平台 WebView 的接口）（`ARCHITECTURE.md`）。本仓库的 `tauri` 核心 crate 负责把它们粘合起来：编译期读取 `tauri.conf.json` 生成配置与 ACL，运行时注入 polyfill 脚本、托管命令 API、管理事件循环（`ARCHITECTURE.md`）。

前端可以用任何能产出 HTML/JS/CSS 的框架，前后端通过 IPC 消息传递通信：前端调用 `@tauri-apps/api` 的 `invoke`，后端 Rust 以 `#[tauri::command]` 宏暴露命令（`crates/tauri-macros`、`packages/api/src/core.ts`）。这一"任意前端 + Rust 后端"的组合是它区别于 Electron（捆绑 Chromium+Node）和 Slint/iced（纯 Rust UI）的生态位。

安全是 Tauri 的头号卖点之一：默认拒绝（deny-first）的能力（Capability/ACL）模型、可选择的 Isolation 隔离模式、运行时生成的 IPC invoke key，并且接受过 Radically Open Security 的 v1、v2 两次第三方审计（审计报告 PDF 直接放在仓库 `audits/` 目录，`supply-chain/` 内还有 cargo-vet 供应链审计文件）。

除核心框架外，仓库还自带完整工具链：打包器（.app/.dmg/.deb/.rpm/.AppImage/NSIS/WiX）、自更新器、CLI、WebDriver 驱动（`tauri-driver`）与官方 GitHub Action（`README.md` Features）。

## 3. 技术栈与依赖

- **语言/工具链**：Rust workspace（edition 2024，rust-version 1.90，resolver 3，见 `Cargo.toml`）；TypeScript（`packages/api`，rollup 构建，`packages/api/package.json`）；pnpm monorepo + napi-rs（`packages/cli` 通过 napi-rs 把 Rust CLI 包成各平台 npm 包，`ARCHITECTURE.md`）。
- **关键 Rust 依赖**（`crates/tauri-runtime-wry/Cargo.toml`、`crates/tauri/Cargo.toml`）：
  - `wry 0.57.0`、`tao 0.37.0`——WebView 与窗口抽象（tauri-apps 组织的上游 crate）
  - `tokio`（rt/fs/io-util/sync）——异步运行时
  - `serde`/`serde_json`（raw_value）、`http`、`url`、`thiserror 2`、`getrandom`、`z85`（invoke key 编码）、`raw-window-handle`、`glob`、`mime`、`dirs`
- **JS 侧**：`@tauri-apps/api` v2.12.4（`packages/api/package.json`），ESM/CJS 双格式发布。
- **构建优化**：workspace release profile 设 `panic="abort"、lto=true、opt-level="s"、strip=true`（`Cargo.toml`），呼应"小体积"定位。

## 4. 目录结构与代码组织

顶层目录（tarball HEAD，dev 分支）：

| 目录 | 职责 |
| --- | --- |
| `crates/tauri` | 核心 crate（约 35k 行，66 个 .rs 文件）：app/window/webview/event/ipc/protocol/scope/menu/tray/state/plugin |
| `crates/tauri-runtime` | 运行时 trait 抽象层（Runtime/RuntimeHandle/WindowDispatch/WebviewDispatch，约 1.3k 行） |
| `crates/tauri-runtime-wry` | 上述 trait 的 WRY 实现（约 5.4k 行，主线程消息循环） |
| `crates/tauri-macros` / `tauri-codegen` / `tauri-build` | `#[tauri::command]` 等宏；编译期解析 tauri.conf.json、嵌入资产、生成 ACL 与 schema |
| `crates/tauri-utils` | 公共工具：配置解析、平台三元组、CSP 注入、ACL schema（`tauri-utils/src/acl/`） |
| `crates/tauri-plugin` | 插件 build 侧接口：校验约定、自动生成 `allow-$command`/`deny-$command` 权限、链接 Android/iOS 工程（`crates/tauri-plugin/src/lib.rs`） |
| `crates/tauri-cli` / `tauri-bundler` / `tauri-macos-sign` / `tauri-driver` | CLI、安装包打包、macOS 签名/公证、WebDriver e2e 驱动 |
| `crates/tauri-schema-generator` / `tauri-schema-worker` | 配置/权限 JSON Schema 生成与在线 schema 服务 |
| `crates/tests/` | 集成测试（acl、restart） |
| `packages/api` / `packages/api-e2e` / `packages/cli` | TS API 包、API 端到端测试、napi-rs CLI 的 npm 外壳 |
| `examples/` | 13 个可运行示例（api、isolation、multiwindow、splashscreen、streaming 等） |
| `bench/`、`audits/`、`supply-chain/` | 基准测试、ROS 审计报告、cargo-vet 供应链审计 |

组织方式评述：典型的 **Rust workspace + pnpm 混合 monorepo**，按"核心 / 抽象层 / 实现层 / 宏与构建 / 工具链 / 测试"严格分层，crate 粒度细且版本独立发布（workspace.dependencies 中各 crate 版本号不同，如 tauri 2.12.5、tauri-build 2.7.4）。规模上 `crates/` 共 293 个 .rs 文件、约 10.6 万行。

## 5. 核心架构与关键实现

### 5.1 运行时分层：trait 抽象 + WRY 实现 + 主线程消息循环

`tauri-runtime` 定义了 `Runtime`、`RuntimeHandle`、`WindowDispatch`、`WebviewDispatch` 四组 trait（`crates/tauri-runtime/src/lib.rs:289-675`），核心 crate 只面向抽象编程；`tauri-runtime-wry` 是唯一官方实现，通过 `Message<T>` 枚举（Application/Window/Webview/EventLoopWindowTarget 四类）+ `send_user_message`/EventLoopProxy 把所有 UI 操作序列化到主线程事件循环（`crates/tauri-runtime-wry/src/lib.rs:1497-1518`），并提供 `run_on_main_thread` 供上层派发闭包。这一设计使 Tauri 理论上可替换 WRY 而无需改动应用代码。

### 5.2 IPC：双通道传输 + 运行时 invoke key

前端→Rust 有两条传输路径，都封装在 `crates/tauri/src/ipc/protocol.rs`：

- **自定义协议（首选，桌面）**：JS 侧 `fetch("ipc://localhost/<cmd>", {method: POST})`，命令名、回调 ID、invoke key 走 HTTP 头（`Tauri-Callback`/`Tauri-Error`/`Tauri-Invoke-Key`），body 走请求体（`crates/tauri/scripts/ipc-protocol.js:24-45`）；Rust 侧 `get()` 注册 URI scheme 处理器，响应通过 `Tauri-Response` 头区分 ok/error（`crates/tauri/src/ipc/protocol.rs:37-63`）。不走 localhost HTTP 服务器。
- **postMessage（兜底，Android）**：Android WebView 读不到请求 body，改用 `window.__TAURI_INTERNALS__.postMessage`（`scripts/ipc.js:147`、`scripts/ipc-protocol.js:33`），由 `message_handler` → `handle_ipc_message` 处理（`crates/tauri/src/ipc/protocol.rs:185-260`）。

防伪造：每次启动生成随机 128 位 Z85 字符串作为 invoke key（`crates/tauri/src/lib.rs:1281` `generate_invoke_key`），注入 JS 后在每次调用时校验，不匹配直接丢弃并记日志（`crates/tauri/src/webview/mod.rs:2010-2022`）。

Rust→前端异步流式数据由 `Channel` 承担：每个 webview 独立的 `ChannelDataIpcQueue` 暂存待发数据，前端通过 `plugin:__TAURI_CHANNEL__|fetch` 命令拉取；小于 8KB 的 JSON（WebView2 实测 eval 比 fetch 快约 2 倍）直接 eval 注入（`crates/tauri/src/ipc/channel.rs:26-52`）。

### 5.3 权限与安全模型：Capability/ACL，deny-first

能力（Capability）= 窗口/webview 标签集合 + 权限列表，应用在 `capabilities/*.json` 声明，编译期由 `tauri-codegen` 解析并烧入二进制（`crates/tauri-codegen/src/context.rs:23,386-389`）；插件命令的 `allow-$command`/`deny-$command` 权限文件由 `tauri-plugin` 在插件 build.rs 中自动生成并产出 JSON schema（`crates/tauri-plugin/src/lib.rs` 文档注释）。

运行时裁决在 `RuntimeAuthority::resolve_access`（`crates/tauri/src/ipc/authority.rs:462-483`）：

```rust
pub fn resolve_access(
  &self, command: &str, window: &str, webview: &str, origin: &Origin,
) -> Option<Vec<ResolvedCommand>> {
  if self.is_denied(command, origin) {
    None
  } else {
    self.allowed_commands.get(command).and_then(|resolved| { ... })
  }
}
```

要点：**显式 deny 优先于一切 allow**；allow 匹配需同时命中执行上下文（`Origin::Local`/`Origin::Remote{url}`，本地/远程 URL 分治）与 window/webview 标签（webview/mod.rs `on_message` 中按 `is_local_url` 构造 Origin 后裁决，`crates/tauri/src/webview/mod.rs:2044-2056`）。命令还可以携带类型化作用域（`CommandScope<T>`，如 fs 插件的路径白名单，`authority.rs:510-540`），示例见 `examples/api/src-tauri/capabilities/run-app.json`（含 `allow`/`deny` 细粒度条目）。

### 5.4 Isolation 隔离模式与资产协议

可选的 Isolation 模式用一个 `isolation://localhost` 的 iframe 作为可信中间层，IPC 载荷经其加密（运行时生成的 crypto keys）再转发，防止应用侧被注入的第三方脚本篡改/窃听消息（`crates/tauri/src/pattern.rs:16-40`、`scripts/ipc.js:136-204`、`crates/tauri/src/protocol/isolation.rs`）。应用资产则通过 `tauri://localhost` 自定义协议直接服务编译期嵌入、哈希、压缩过的资源（`crates/tauri/src/protocol/tauri.rs:27,160`），避免起本地 HTTP 端口（README 明确将其列为特性）。

## 6. 构建与运行

终端用户/应用开发者（来自 `README.md` 与 `ARCHITECTURE.md`）：

```sh
npm create tauri-app@latest   # 脚手架
pnpm tauri dev                # 开发（热更新）
pnpm tauri build              # 产出到 src-tauri/target/release
```

本仓库开发者（来自根 `package.json` 与 `.github/workflows/test-core.yml`）：

```sh
pnpm install
pnpm build          # 构建 JS 包（api + cli）
pnpm test           # 各包测试（递归）
pnpm test:api-e2e   # @tauri-apps/api 端到端测试
cargo test          # Rust workspace 单元/集成测试（CI 用 toolchain 1.90）
pnpm fmt / pnpm eslint:check / pnpm ts:check
```

## 7. 测试与工程质量

- **测试规模**：`crates/` 内 351 个 `#[test]`/`#[tokio::test]`；核心 crate 自带 `crates/tauri/test/` 与 `tests/` 目录（含 mock runtime：`src/test/`）；场景化集成测试独立成 crate——`crates/tests/acl`（ACL 裁决场景）与 `crates/tests/restart`（`Cargo.toml` workspace members）。JS 侧有 `packages/api-e2e` 桌面+移动双端 e2e（`test-api-e2e.yml`、`test-api-e2e-mobile.yml`）。
- **CI**（`.github/workflows/`，共 25 个 workflow）：`test-core.yml` 在 windows-latest/ubuntu-latest/macos-14 上以固定 toolchain 1.90 跑核心测试；另有 Android 测试、lint（rustfmt/eslint）、udeps、cargo-audit、基准（bench.yml）、license header 与生成文件一致性检查；发布流程用 **covector**（`covector-version-or-publish.yml`）管理多包版本与发布。
- **安全工程**：`supply-chain/audits.toml` 维护 cargo-vet 信任与审计条目；`audits/` 存放 Radically Open Security v1/v2 完整审计报告 PDF；`SECURITY.md` 要求漏洞走私密协调披露，禁止公开提 issue/PR。
- **观察**：依赖 pinned 严格（`Cargo.lock` + renovate + dependabot 双管齐下）、crate 边界清晰、测试偏单元+场景集成而非 UI 自动化（UI 侧交给 WebDriver `tauri-driver` 与外部 e2e 包）。

## 8. 社区与维护状态

- **活跃度**：gh api 显示最近 push 为 2026-10-29（分析前一天）；核心 CHANGELOG（`crates/tauri/CHANGELOG.md`）2.12.x 版本持续滚动，每个版本带 PR 编号与作者署名；最新 CLI release 2026-10-21。维护节奏可判定为高频持续。
- **贡献者**：约 617 名（含匿名，gh api contributors 分页）；CHANGELOG 中可辨识出 core 团队成员（如 amrbashir、lucasfernog 等）与大量外部 PR 并存，属于"核心团队评审 + 社区贡献"的健壮结构。
- **治理**：Tauri Programme within The Commons Conservancy（`README.md` Organization 节），商业赞助方 CrabNebula；文档站点与仓库分离（tauri-docs 仓库）。
- **发布物**：Rust crate（crates.io）与 npm 包（@tauri-apps/api、@tauri-apps/cli）双生态发布，`publish-cli-js.yml`/`publish-cli-rs.yml` 自动化。

## 9. 亮点与风险

**亮点**

1. 小体积哲学贯彻到工程细节：系统 WebView + `opt-level="s"` + `panic="abort"` + 原生协议免本地服务器（`Cargo.toml`、`crates/tauri/src/protocol/tauri.rs`）。
2. 安全模型是同类框架中最完整的：编译期烧入的 deny-first ACL、origin 分治（Local/Remote）、运行时 invoke key、可选 Isolation 隔离层，且有两次第三方审计背书（`crates/tauri/src/ipc/authority.rs`、`audits/`）。
3. 运行时抽象干净（`tauri-runtime` trait 层），WRY/TAO 可替换，也使核心可被 mock 做无头测试（`crates/tauri/src/test/`）。
4. 工具链闭环：脚手架、CLI（napi-rs 多平台 npm 分发）、打包、更新、签名、WebDriver、schema 生成一应俱全。
5. 工程纪律好：25 个 CI workflow、covector 发布、cargo-vet 供应链审计、license header 检查。

**风险/局限**

1. 多 crate + 编译期宏/codegen 的心智负担高，ACL/Capability 概念对新手有学习曲线，排错需要跨 `tauri.conf.json`/`capabilities/`/生成 schema 多处比对。
2. 受系统 WebView 差异制约：Linux 依赖 webkit2gtk 4.1、Android 兜底走 postMessage、部分 API 平台行为不一致需大量条件编译（如 `scripts/ipc-protocol.js:33` 的 Android 特判）。
3. v1→v2 迁移与 2.x 内小版本 API 调整（CHANGELOG 中多处行为修正）对生态插件是持续成本；第三方插件质量参差。
4. GitHub 仓库元数据许可证标 Apache-2.0，与实际的 MIT OR Apache-2.0 双许可不完全一致（小瑕疵，见 `Cargo.toml` license 字段）。

## 10. 总结评价

Tauri 是"用系统 WebView + Rust 重做 Electron"这一命题当前最成熟、工程化程度最高的答案：分层清晰的 workspace、编译期 ACL 与 Isolation 等安全设计达到生产级安全工程水准，配套 CLI/打包/更新/测试工具链完整，且以星数 11 万+、日级活跃、约 617 名贡献者的规模维持着健康的社区治理。适合：想要小体积桌面/移动应用的 Web 团队、需要把 Rust 能力暴露给前端的系统开发者；也非常适合想学习"Rust 大型 monorepo 如何做运行时抽象、进程间/进程内 IPC 安全设计"的工程师——`crates/tauri/src/ipc/`、`crates/tauri/src/ipc/authority.rs` 与 `crates/tauri-runtime-wry/src/lib.rs` 是最值得精读的三个入口。
