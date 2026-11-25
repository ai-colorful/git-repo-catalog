# spacedriveapp/spacedrive 项目分析

> 星数 39507（快照 2026-10-24）｜主语言 TypeScript（任务档案）／Rust（GitHub 实际归类）｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：开源个人数据基础设施——用 Rust 守护进程把散落在本地磁盘、NAS、移动设备、私有服务器与可选云存储上的文件，统一成一台"虚拟分布式文件系统（VDFS）"，本地优先、无托管账号、无遥测。

| 项目 | 事实 |
|---|---|
| 名称 | spacedriveapp/spacedrive |
| 星数 | 39,507（gh api，2026-11-14 实测） |
| 语言 | Rust 主体（约 28.8 万行）+ TypeScript 前端（约 4.9 万行）；GitHub linguist 归类为 Rust |
| 许可证 | Apache-2.0（2026-03-24 之前的 release 仍为 AGPL-3.0，见 `README.md` License 节） |
| 版本 | 2.0.0-beta.1（`core/Cargo.toml`、`apps/tauri/package.json`） |
| 创建时间 | 2021-09-27（gh api） |
| 最近 push | 2026-11-13（gh api，解压文件 mtime 同日） |
| 贡献者 | 118 人（contributors API 分页合计） |
| Release | 共 29 个 tag release（gh api） |

## 2. 项目是什么，解决什么问题

Spacedrive 的出发点写在 `README.md` 首行："你的数据不必住在别人的电脑上才有用"。云服务让"文件放在别人服务器上"成为默认，个人文件与行为数据堆积在你无法控制安全策略的基础设施上；而自己持有磁盘只是第一步——你还需要搜索、跨设备访问、备份状态感知、历史恢复、按自己条件分享。Spacedrive 要把这些云盘的便利性搬回自有存储（`README.md` "Own where your data lives" 节）。

技术上它是一个 VDFS（Virtual Distributed File System）：所有数据来源（文件夹、整盘、NAS 数据集、另一台 Spacedrive 节点、云桶、外部服务）抽象为"source"，用同一种记录形态与同一套操作。每个 source 携带一个可移植的存储（store），由"可重建的 generation"（索引、记录、FTS5 搜索）与"不可重建的 durable assertions"（人工标注、tag 断言）两半组成——拔掉硬盘后目录仍可搜索，换机器后元数据随盘走（`crates/store/src/lib.rs` 开头注释、`README.md` "One filesystem, every source" 节）。

它同时把"保护"内建为文件系统职责：每次改动文件的操作先经过 validation（权限/容量/安全规则）与 preview（基于索引投影出执行后的目录树，可逐级预览结果），守护进程在分发时二次校验；删除类操作只在全字节比对确认后才视文件为重复，采样哈希只用于快速筛选候选（`README.md` "See every change before it happens" 节，对应 `core/src/ops/files/` 下的 strategy 实现）。

节点之间用 Iroh + QUIC 端到端加密直连（必要时回落中继），配对节点可互相浏览授权的 source、按字节范围流式读取；还支持把任意 source/subtree 挂载进 macOS/Windows/Linux（`core/src/service/mounts/`、`core/src/service/network/protocol/byterange.rs`）。桌面、Web、移动端、CLI、生成的 SDK、skill 与 MCP 服务器共用同一套编译期注册的 typed 操作面，由 Specta 从 Rust 类型生成 TypeScript 与 Swift 客户端，保证"人点击"与"软件调用"走同一契约（`README.md` Architecture 节）。

项目历史值得注意：2021 年立项的 v1（0.2~0.4.x 系列发布）验证了需求，也暴露了"两套文件模型、脆弱传输、默认构建塞太多产品"的代价，于是团队推倒重来。V2 围绕 sources、可移植 store、持久 job、单一 typed 操作面重建（`README.md` Project history 节、`docs/overview/history.mdx`）。当前仓库即 V2 代码基线，`main` 分支于 2026-10-05 由 `sources` 分支快进而来（`PROJECT_STATUS.md`）。

## 3. 技术栈与依赖

- **核心（Rust，edition 2021，工具链锁 1.97.1，见 `rust-toolchain.toml`）**
  - `tokio` 1.42 异步运行时；`sea-orm` 1.1 + `sqlx` 0.8（SQLite）为库级数据库；`notify` 6.1 文件监听（`crates/fs-watcher/`）
  - `blake3` 1.5 内容寻址（文件内容身份）、`uhlc` 混合逻辑时钟（同步排序）
  - `iroh` 0.95 + QUIC 做 P2P 传输（`Cargo.toml` workspace deps、`core/src/service/network/`）
  - `opendal` 0.54 作为云存储后端抽象；`axum` 提供服务端 HTTP
  - 向量/AI：`lancedb` 0.15、`fastembed` 4、`ort`（ONNX Runtime）；语音转文字走 `whisper-rs`（feature `ai`，`core/Cargo.toml`）
  - 媒体：`sd-ffmpeg` 链接宿主 FFmpeg 出缩略图/thumbstrip，`sd-images` 支持 libheif（`crates/ffmpeg/`、`crates/images/`）
  - UI：GPUI 来自 spacedriveapp/zed fork（锁 rev），用于 `apps/native` 原生子应用（`Cargo.toml` L52-55 注释）
  - WASM 扩展运行时 `wasmer` 6（feature `wasm`，`core/Cargo.toml`）
- **前端（TypeScript，Bun 1.3.4 monorepo，`package.json`）**
  - React 19 + react-router 6 + Tailwind CSS 4 + Vite 5；桌面壳为 Tauri 2（`apps/tauri/package.json`：`@tauri-apps/api` ^2.1.1）
  - `packages/interface` 为跨端共享 React 界面层，`packages/ts-client` 为生成的客户端，`packages/swift-client` 为 iOS 侧
  - Turborepo 管理 JS 构建管线（`package.json` devDeps `turbo`）
- **代码生成**：`specta`（Jamie Pine fork）+ `specta-typescript`/`specta-swift` 从 Rust ops 类型生成前端契约（`core/Cargo.toml`；`.gitattributes` 标记 `/apps/desktop/src/commands.ts`、`/packages/client/src/core.ts` 为生成物）
- **构建编排**：`just`（`justfile`）+ `xtask`（Rust 写的任务编排，`xtask/`）

## 4. 目录结构与代码组织

Rust workspace + Bun workspace 叠加的 monorepo（`Cargo.toml`、`package.json`）：

| 目录 | 职责 |
|---|---|
| `core/` | `sd-core`：VDFS 主体。`domain/`（Entry/Library/Device 模型）、`ops/`（CQRS 操作注册表）、`infra/`（数据库实体、事件总线、wire 协议、job）、`service/`（network/sync/thumbs/mounts/watcher/file_sharing）、`location/`、`library/`、`device/`、`volume/`、`crypto/`（`core/README.md`） |
| `crates/` | 25 个支撑 crate：`store`（每源 SQLite 便携存储）、`fs-watcher`、`task-system`（持久任务）、`sync` 相关由 core/service/sync 承担、`ffmpeg`、`images`、`media-metadata`、`photos`、`crypto`、`sdk`/`sdk-macros`、`sd-client`、`actors`、`supervisor`、`archive`、`job-derive`、`fda`、`rc4-shim`（为 iroh 兼容打的补丁，`Cargo.toml` L193-194） |
| `apps/` | `tauri`（桌面 React+Tauri 2）、`web`（Vite Web 端）、`server`（无头服务 + Web UI）、`cli`（54 个 rs 文件的 `sd` 命令行）、`native`（GPUI 原生应用，默认不参与构建）、`mobile`（React Native，含 `sd-mobile-core`） |
| `packages/` | `interface`（共享 React UI 路由：explorer/sources/tag/trash/redundancy/analyzer/settings 等）、`ts-client`、`swift-client`、`assets`、`config` |
| `adapters/` | 11 个数据源适配器：gmail、slack、github、chrome 书签/历史、safari 历史、obsidian、apple-notes、macos 日历/通讯录、opencode |
| `extensions/` | 扩展示例（photos、test-extension）+ README |
| `xtask/` | Rust 编写的构建/安装编排（`cargo run -p xtask -- setup`） |
| `docs/` | Fumadocs 文档站（独立 bun 安装）；`.github/` 含 CI、nightly、release 工作流与自定义 action |

组织方式评述：典型的"Rust 核心 + 多端薄客户端"分层。核心按 DDD 风格分层（domain/ops/infra/service），操作层统一 CQRS；JS 侧只保留壳与共享 UI，业务逻辑全部下沉 Rust，前端契约靠代码生成而非手写，避免多端漂移。默认 `default-members` 排除 Tauri/native/mobile（需前端先构建），日常校验聚焦 daemon+cli+crates（`Cargo.toml` L25-32 注释）。

## 5. 核心架构与关键实现

### 5.1 VDFS 核心：Core 上下文与 CQRS 操作面

`core/src/lib.rs` 中 `Core` struct 聚合 DeviceManager、LibraryManager、VolumeManager、EventBus、LogBus、Services 与 ApiDispatcher，是守护进程的唯一入口。操作层为 CQRS：`core/src/ops/` 下按资源域（files/indexing/search/tags/devices/network/mounts/redundancy...）划分 action（写）与 query（读），经 `inventory` crate 编译期注册（`core/README.md` Operations Layer 节），同一注册表同时服务 daemon RPC、iOS FFI 与 WASM 扩展——这是"一套 API 服务所有客户端"承诺的落地位置。

### 5.2 每源便携存储（sd-store）

`crates/store/src/lib.rs` 用一段高密度注释定义了数据模型：一个 source 一个 SQLite 文件，`record`（每索引对象一行，(type, external_id) 唯一）、`facet_<model>`（各类型自有列）、`content`（字节身份）、`edge`（源内关系）、`search_index`（FTS5）构成可重建的 generation；`record_overlay` 与 tag 表存"没有 ingest 能再造出"的人工断言，永不重建。文档明确"任何代码路径不得假设 store 可重建"——拔盘、token 失效、账号关闭都是常态。跨源查询不跨库 join，而是逐 store 查询后合并。设计依据在 `docs/core/design/source-durability.md`（该文件在仓库 docs/ 中）。

### 5.3 Leaderless P2P 同步

`core/src/service/sync/mod.rs` 头注释言明架构："Real-time library synchronization (Leaderless)"，混合模型：设备自有数据走状态同步，共享资源走基于 HLC（`uhlc`）的日志同步。实现包含 `peer.rs`（PeerSync）、`protocol_handler.rs`、`retry_queue.rs`、`backfill.rs`（BackfillCheckpoint 断点续传）、`state.rs`（DeviceSyncState/BufferQueue）与 `metrics/`，挂接 `service/network/protocol/` 的 SyncProtocolHandler。传输层由 Iroh/QUIC 承担（`core/src/service/network/transports/`），配对走 `protocol/pairing`，文件传输与字节范围读取各有独立协议处理器（`byterange.rs`、`file_transfer.rs`）。结论：P2P 同步不是纸面设计，是已成规模的实现（sync 服务约十余个模块），但当前处于 2.0 beta 前夜，协议稳定性以 nightly 口径对待。

### 5.4 缩略图与媒体管线

缩略图服务在 `core/src/service/thumbs/`，生成链路依赖 `crates/ffmpeg`（链接宿主 FFmpeg 出视频缩略图与 thumbstrip）、`crates/images`（含 libheif feature）、`crates/media-metadata`、`crates/sidecar-path`（sidecar 存放路径）。重度编解码按设计留在默认二进制之外，由扩展按需引入（`README.md` "Extend the filesystem" 节）。桌面端默认构建需要宿主安装 FFmpeg 开发库，否则 `ffmpeg-sys-next` 报 `libavutil.pc not found`（`README.md` Build from source 节）。

## 6. 构建与运行

来自 `README.md` Build from source 与 `justfile`：

```bash
./scripts/setup.sh          # Windows: .\scripts\setup.ps1，装系统依赖
bun install
cargo run -p xtask -- setup # 生成 cargo 配置；--native-deps 装 FFmpeg/libheif/Pdfium 预编译包

just dev-daemon             # 跑 Rust 守护进程（cargo run --bin sd-daemon）
just dev-desktop            # 桌面客户端（Tauri，先 build-apps 编译 sd-native）
just dev-server             # 无头服务 + Web 界面（sd-server）
just dev-mobile             # 移动端

just test                   # 全 workspace 测试
just check                  # 提交前检查
```

依赖：Rust ≥1.95（锁 1.97.1）、Bun 1.3.4（CI 与 `package.json` packageManager 双重锁定）、Node 20、`just`、宿主 FFmpeg 开发库（`libav*-dev`+`libclang-dev`，Debian/Ubuntu）。包管理强制 Bun（`package.json` preinstall `npx only-allow bun`）。

## 7. 测试与工程质量

- 测试规模：Rust 侧 1327 个 `#[test]`/`#[tokio::test]` 注解（grep 全 workspace 实测），另有 `core/tests/`、各 crate `tests/` 目录与 `core/benchmarks/`；TS 侧 `.test.ts(x)`/`.spec.ts(x)` 约 12 个文件——测试重心明显在 Rust 核心。Windows 相关验收套件（文件操作、source/drive）尚未在 Windows 上运行（`README.md` nightly 节明言）。
- CI（`.github/workflows/`）：`ci.yml`（push/PR/merge_group 触发，rustfmt、clippy、三平台构建，Windows runner 把 target 目录符号链接到 C 盘省空间）、`core_tests.yml`、`server.yml`、`mobile.yml`、`nightly.yml`+`nightly-smoke.yml`（跟随 main 的每 push 构建与冒烟）、`release.yml`、`cache-factory.yaml`（自建缓存体系）。
- 质量工具链：clippy 配置（`.clippy.toml`）、cspell 三份词表（`.cspell/`）、prettier + 排序导入插件、taplo（TOML 格式化）、eslint。发布 profile 开 LTO、单 codegen-unit、按体积优化（`Cargo.toml` `[profile.release]`）。
- 特色工程实践：仓库根有 `AGENTS.md`、`CLAUDE.md` 与 `.agents/skills/spacedrive-cli/`，配套 `PROJECT_STATUS.md`（2026-11-05 更新）做跨会话上下文交接——开发流程本身围绕 AI 编码代理组织，属于少见的"agent-first 仓库治理"样本。

## 8. 社区与维护状态

- 活跃度极高：最近 push 2026-11-13，近 5 个提交全部来自创始人 Jamie Pine 当日（gh api commits）。
- 发版节奏（gh api releases）：2024 年 0.2.x 密集发布（最快半月一版，0.2.9→0.2.14 跨两个月）→ 0.3.0/0.3.1（2024-05）→ 0.4.0（2024-07）→ 0.4.2（2024-08）→ **0.4.3（2025-03）后陷入长期静默**（V2 重写期）→ v2.0.0-alpha.1（2025-12-26）→ alpha.2（2026-02-07）→ 2026-10 起转 nightly 通道。2.0 Beta 目标日原定 2026-10-01，两次顺延至 2026-11-15，且 README 强调"这是发布目标，不代表门槛已通过"（`PROJECT_STATUS.md` Immediate register 节）。
- 贡献者 118 人，但近提交高度集中于创始人一人，属于典型的创始人主导型项目；65 个 open issue（gh api）。
- 许可证从 AGPL-3.0 改为 Apache-2.0（2026-03-24 生效），配合 2.0 商业化叙事（`README.md` License 节）。
- 生态：SpaceUI 共享组件库独立成仓（`README.md` Contributing 节）；docs 站、网站在相邻仓库开发。

## 9. 亮点与风险

**亮点**
1. 架构叙事与代码高度一致："一套 typed 操作面 + Rust 守护进程"不是 PPT——Specta 代码生成、CQRS 注册表、portable store 的两半模型都能在源码中指认（`crates/store/src/lib.rs`、`core/src/ops/`）。
2. 数据耐久性设计罕见地认真：store 的"generation 可重建 / assertions 永不重建"铁律、全字节验证才判重复、操作前三问（validation/preview/execution），把文件安全做进了文件系统层而非 UI 弹窗。
3. 规模与测试扎实：约 28.8 万行 Rust、1327 个 Rust 测试、覆盖三平台的 CI + nightly 冒烟，工程底座远超同星数均值。
4. 扩展与适配器生态雏形已现：11 个数据源适配器 + WASM 扩展运行时 + 声明式权限。
5. 源码可读性好：crate 级 README 与模块头注释写得像设计文档（如 `sd-store`），几乎每个关键决策都能在注释里找到理由。

**风险/局限**
1. 版本阶段早于 1.0：从未发布过 stable 1.0（发布历史直接 0.4.x → 2.0 alpha/beta，README 承诺在 launch post 里"诚实解释跳过的 stable 1.0"），nightly 明示"可破坏 schema 且无迁移，请自备备份"。
2. 平台不均衡：2.0 的 Windows 支持仅为预览（无桌面应用、无自动启动、验收套件未跑），主力是 macOS/Linux。
3. 单人主导 + 长静默史：2025-03 至 2025-12 的 9 个月空窗说明重写期交付风险，beta 已两次跳票（10-01 → 11-01 → 11-15）。
4. 构建门槛高：Rust 1.97 + Bun 1.3.4 精确锁定 + 宿主 FFmpeg 开发库 + xtask 编排，贡献者上手成本不低；GPUI 依赖 fork zed 的固定 rev，升级是"deliberate ritual"（`Cargo.toml` L52 注释）。

## 10. 总结评价

Spacedrive 是"个人数据主权"赛道里工程完成度最高的开源尝试：它没有停留在"好看的跨设备文件管理器"，而是把内容寻址、可移植索引、leaderless 同步、操作预验证这四件难事真的写了约 29 万行 Rust 出来，并用代码生成让六种客户端共享同一契约。它还没有 1.0——准确说从未有过任何 stable——但 2.0 Beta（计划 2026-11-15）意味着数据模型即将承诺向后兼容。适合关注本地优先软件、P2P 同步与文件系统设计的工程师读源码（`sd-store` 与 `service/sync` 的注释是最佳入口）；适合愿意陪跑 nightly、有备份习惯的多设备重度用户试用；不适合把它当作日常生产主力或期待 Windows 完整体验的人。
