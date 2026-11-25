# storytold/artcraft 项目分析

> 星数 41798（快照 2026-10-24）｜主语言 Rust｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：ArtCraft 是"面向艺术家的 IDE"——一个 Rust 编写的桌面应用，把 AI 图像/视频/音频/3D 生成从"提示词"升级为可精确操控的"创作工艺"（crafting）：2D 合成、3D 场景排布、角色摆姿、多模型聚合生成。

| 项目 | 内容 |
|------|------|
| 名称 | storytold/artcraft（ArtCraft，getartcraft.com） |
| 星数 / Fork / Open Issues | 41798 / 6464 / 770（gh api，2026-11-21） |
| 主语言 | Rust（约 27.3 万行）+ TypeScript（约 20.7 万行） |
| 许可证 | MIT OR Apache-2.0 双许可（`LICENSE-MIT`、`LICENSE-APACHE`、`NOTICE`） |
| 创建 / 最近推送 | 2022-03-31 / 2026-11-20（昨天仍在推送） |
| 当前形态 | Tauri 2 桌面应用（默认产物）+ 纯 Rust egui 原生应用（`artcraft-native`） |
| 最新 release | v0.41.1（2026-11-07），共 29 个可见 release |
| 默认分支 | main；2026-09-11 曾"fresh start"重置历史 |

## 2. 项目是什么，解决什么问题

ArtCraft 把 AI 生成类应用常见的"输入提示词→碰运气"流程，替换成影视创作里的"预演（blocking）"流程：先在画布/3D 场景里把构图、机位、角色姿态、图层深度排好，再调用生成模型产出最终画面或视频（`README.md`："Turn prompting into *crafting* with visual tools for precise, repeatable results"）。

具体能力按 README 与源码可归纳为三块：

1. **场景构建工具**：2D 图像合成（图层、抠背景、画笔，`frontend/libs/components/pagedraw`，约 8.3k 行）；3D 场景合成（Three.js + ammo.js 物理，图转 3D 网格、kitbashing 场景排布、角色摆姿、摄像机控制，`frontend/libs/components/pagescene`，约 22.4k 行）；以及一个规模最大的视频编辑器（`frontend/libs/components/video-editor`，约 71k 行 TS/TSX）。
2. **多模型聚合生成**：内置 62 个模型目录（README 折叠表；图像 16 / 视频 25 / 音乐音效 5 / 3D 网格 11 / 世界与高斯泼溅 5），聚合 Nano Banana、GPT Image、FLUX、Seedream、Kling、Veo、Sora、Seedance、Suno、Hunyuan 3D、Meshy、Rodin、Marble（World Labs）等家族，并直连用户自己的 Midjourney / Grok / fal / World Labs 账号（`crates/api_clients/artcraft/artcraft_router/src/api/router_provider.rs` 定义 7 个 provider 路由）。
3. **任务与产出管理**：生成任务持久化在嵌入式 SQLite（`crates/schema/database/sqlite_tasks`，sqlx 编译期校验），产出进入 Library/Feed（缩略图缓存、批次、批量下载）。

解决的核心痛点：AI 创作中"构图、角色一致性、机位不可控"。ArtCraft 用预排场景 + 可重复参数 + 参考图/关键帧槽位（`artcraft-native/README.md` 的 Prompt box parity 表）来换取确定性。

商业模式：桌面客户端开源免费，生成走 ArtCraft 托管服务按 credit 计费（桌面 crate 内含 `storyteller_get_credits_command.rs`、`storyteller_get_subscription_command.rs`、`stripe_checkout/`、`stripe_customer_portal/`，位于 `crates/desktop/artcraft/src/services/storyteller/commands/`）；同时支持自带 API key（fal）或自有账号（Midjourney）。`ROADMAP.md` 明确 2026 架构目标包含"Remove dependence on ArtCraft hosted services"。

组织背景：storytold 组织（"Open Source Tools and Models for Makers, Builders, and Creatives"，2021 年创建，83 个公开仓库）是 AI 语音社区著名项目 FakeYou 的团队——`_docs/artcraft_omni_api.md` 写明生产 API 为 `api.storyteller.ai`、CDN 为 `cdn-2.fakeyou.com`。仓库 2022-03-31 创建，2026-09-11 由核心开发者 Brandon Thomas 执行"fresh start"：把数百 GB 历史的旧 monorepo（Web API 网关、任务服务、SRE 工具）迁往 `storytold/artcraft-services`，保留本仓库位置与已有 star（该提交信息可在 commits API 第 1 页最早一条看到）。

## 3. 技术栈与依赖

| 层 | 技术 | 出处 |
|----|------|------|
| 桌面壳 | Tauri 2.11（`unstable`、`devtools`、`protocol-asset`） | `Cargo.toml` L111 |
| 备用 UI | egui（纯 Rust 原生应用，无 webview，仅 Midjourney 登录窗口例外） | `artcraft-native/README.md`、`Cargo.toml` L6 |
| Rust HTTP | reqwest（正式 API）+ wreq（TLS/JA3 浏览器指纹模拟，应对 Cloudflare 保护的网页端 API） | `Cargo.toml` L104/L130、`crates/lib/browser_emulation/src/lib.rs` |
| 前端 | Nx monorepo + React + Vite + Zustand + Three.js + Tailwind | `AGENTS.md` L67、`frontend/nx.json` |
| 3D 物理 | ammo.js（Bullet 物理 wasm 版） | `NOTICE`、`pagescene/src/lib/engine/ammoLoader.ts` |
| 数据库 | SQLite + SQLx（编译期校验查询，`.sqlx/` 离线元数据） | `AGENTS.md` L61 |
| 测试 | cargo test（5314 个 `#[test]` + 752 个 `#[tokio::test]`）、egui_kittest 无头 UI 测试、Vitest + Jest | 全仓 grep、`frontend/vitest.workspace.ts` |
| 质量设施 | SonarCloud、CodeRabbit PR 审查、step-security 加固 runner | `README.md` 徽章、`.github/workflows/*.yml` |

## 4. 目录结构与代码组织

| 目录 | 职责 | 规模（Rust LOC） |
|------|------|------------------|
| `crates/desktop/artcraft` | Tauri 桌面应用：core（api_adapters/commands/events/lifecycle/providers/state/threads/windows）+ services（storyteller/grok/midjourney/sora/worldlabs） | 30.5k |
| `crates/api_clients` | 各生成服务商客户端（本仓库最大 Rust 区块） | 195.3k（artcraft 91.8k、fal 54.3k、kinovi 18.4k、grok/sora/worldlabs/midjourney/gmicloud 其余） |
| `crates/lib` | 桌面支撑库：browser_emulation、cookie_store、jwt_light、cloudflare_mitigation、files/images 等 15 个小 crate | 5.7k |
| `crates/schema` | `sqlite_tasks`（任务持久化）+ `public/enums`、`public/tokens`（线格式枚举） | 26.0k |
| `artcraft-native` | 纯 Rust egui 桌面应用（backend/models/prompt_box/pages/feed/overlays/shell/ui） | 14.2k |
| `frontend/` | Nx TS 工作区：`apps/artcraft`（Tauri 前端）+ 15 个 `libs/`（components 下约 47 个组件库，含 pagescene/pagedraw/video-editor/vfx/viewer-3d/promptbox/moodboard） | TS 206.6k |
| `_database/sql/artcraft_migrations` | 嵌入式 SQLite 迁移（现仅 `20250708000000_create_tasks_table.sql`） | — |
| `.github/workflows` | 三平台 publish + freebsd + sonarcloud + house-rules + packaging-lint | — |
| `_docs/`、`docs/`、`script/`、`packaging/` | 开发文档、性能文档、dev 脚本、Linux 打包（AppImage/DEB/RPM/Arch/Flatpak） | — |

组织方式评述：Cargo workspace 单体（46 个成员 crate，`default-members = ["crates/desktop/artcraft"]`）+ Nx 前端工作区，前后端同仓。API 客户端层刻意做厚（195k 行），使 Tauri 应用与 egui 应用共享同一套客户端（`artcraft-native/README.md` 架构表）。后端服务（计费、队列、网站）在姊妹仓库 `artcraft-services`，本仓库只含桌面端。

## 5. 核心架构与关键实现

**1) 多 Provider 生成路由器**（`crates/api_clients/artcraft/artcraft_router/`）
`RouterProvider` 枚举定义 7 条通道；`client/multi_router_client_builder.rs` 组装各 provider 客户端，`generation_mode_mismatch_strategy.rs` 处理请求参数与模型能力不匹配时的降级策略。模型/参数全部是独立枚举文件（`router_image_model.rs`、`router_video_model.rs` 等 16 个 `api/` 模块），模型能力由服务端下发，新增模型无需发版（`artcraft-native/README.md`："Model capabilities come from the server, so new models and options appear without an app update"）。

```rust
// crates/api_clients/artcraft/artcraft_router/src/api/router_provider.rs
/// The provider to route a generation request to.
#[derive(Copy, Clone, Debug, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum RouterProvider {
  Artcraft,
  Midjourney,
  Fal,
  GmiCloud,
  GrokApi,
  KinoviWeb,
  WorldLabs,
}
```

**2) 浏览器指纹模拟层**（`crates/lib/browser_emulation/src/lib.rs`）
为反爬敏感的消费级端点（Midjourney、Grok consumer、World Labs consumer、Kinovi）提供统一浏览器身份：wreq 配置 TLS/JA3 与 HTTP/2 指纹、User-Agent、sec-ch-ua 等一致性头，并支持用从真实浏览器捕获的 `cf_clearance` cookie 匹配对应指纹。配套 `cloudflare_mitigation`、`cookie_store`、`jwt_light` 等小 crate。这是"直连用户自己的账号"这一卖点的基础设施，也是代码量显著膨胀的原因之一。

**3) 3D 场景编辑器 pagescene**（`frontend/libs/components/pagescene/src/lib/`）
约 22.4k 行 TS。`engine/` 内是改造过的 Three.js 管线（`CustomOutlinePass.js`、`SelectionOutlinePass.ts`、`TransformControls.js`、`InfiniteGridHelper.ts`、ammo.js 物理加载、chromakey），`actions/` 以命令模式组织场景操作（`addCharacter.ts`、`cameraView.ts`、`timeline.ts`、`recordOutput.ts`），`signals/` + Zustand 做状态桥接（`EngineStoreBridge.ts`）。角色一致性通过"虚拟演员 + 摆姿 + Identity Transfer"流程实现（README 特性表）。

**4) 双 UI 桌面栈**
Tauri 应用（默认）与 egui 应用共享 `artcraft_router`/`artcraft_client`：egui 端 backend 在 Tokio 上跑全部 API 调用、以 `Event` 回推 UI，`backend::wire` 用宽松解码保证服务端新增枚举值不炸旧客户端（`artcraft-native/README.md` 架构表）。Tauri 侧每个命令一个文件模块（`crates/desktop/artcraft/src/services/storyteller/commands/`，16 个文件 + 专属 AGENTS.md 规范），认证走设备码/密码/浏览器桥（`login_bridge.rs`）。

## 6. 构建与运行

来自 `_docs/dev_setup.md`、`frontend/README.md`、`artcraft-native/README.md`：

```bash
# 前端（frontend/ 目录下）
npm ci && npx nx dev artcraft      # Vite 开发服务器，端口 5173/5193 起
npx nx build artcraft              # 产物 apps/artcraft/dist，由 Tauri 打包

# Tauri 桌面应用（仓库根目录）
./script/artcraft/unix_dev.sh      # Unix 一键前端+Rust 热重载
.\script\artcraft\windows_frontend_dev.ps1 / windows_rust_dev.ps1   # Windows 双终端

# 纯 Rust egui 应用（无 Node 依赖）
cargo run -p artcraft_native --release

# 测试
cargo test -p artcraft_native      # 含 egui_kittest 无头 UI 测试
node --test frontend/scripts/unix-dev.test.mjs   # 启动器自测（真实 Vite 监听 + 假 Cargo 进程树）
```

前置：Rust stable、Node 20+、Tauri CLI 2.x；wreq 需 cmake/perl/nasm（`Cargo.toml` L126-128 注释）。

## 7. 测试与工程质量

- Rust：3264 个 .rs 文件、27.3 万行；1054 个文件含 `#[cfg(test)]`，5314 个 `#[test]` + 752 个 `#[tokio::test]`，另有 egui_kittest 驱动的无头 UI 测试（`artcraft-native/README.md` Tests 节）。
- 前端：26 个 Vitest spec 文件 + `apps/artcraft` 内 Jest 配置；`vitest.workspace.ts` 按 Vite 配置聚合。
- CI（`.github/workflows/`）：Windows/macOS/Linux 三平台 publish（release 分支触发，含 step-security 加固与全部 action 哈希钉死）、freebsd 构建测试、SonarCloud 扫描、house-rules 工作流（从 `storytold/craft-common-actions` 共享复用）、packaging-lint、Dependabot。
- 工程纪律突出：根 `AGENTS.md` 是一整套 AI 辅助开发守则（15 条 house rules：PR 必经 main、修复必须带测试、禁止 agent 署名等），配 `.claude/rules/`、`.coderabbit.yaml`、`sonar-project.properties`、`.config/hakari.toml`（cargo-hakari）。近一个月 42 次提交，open PR 仅 1，PR 队列干净。

## 8. 社区与维护状态

- 活跃度：gh api 显示最后推送 2026-11-20（昨天）；近 30 天 42 commits；29 个可见 release，2026 年节奏为每月 1-2 个（v0.28.0 2026-04-14 → v0.41.1 2026-11-07，releases API）。
- 贡献者结构：可见历史仅 9 位贡献者（dylanmtaylor 30、echelon 20、FredApps 9、bflatastic 6 等），核心代码由 Brandon Thomas 主导——小团队 + 外围修复模式；同时有 CodeRabbit/SonarCloud 机器人参与质量门禁。
- 社区渠道：Discord（README 徽章）、YouTube @OfficialArtCraftStudios、X、LinkedIn；`ROADMAP.md` 提到要自建内容工作室、办节展、拍 AI 长片。
- 商业/组织：FakeYou 团队（`_docs/artcraft_omni_api.md` 的 CDN 域名佐证）；API key 按账号人工开通（同文件"Contact the Artcraft team to enable it"）。
- 未确认：精确的 star 增长时间线（无 star history 数据源）。可核实的间接证据是 770 个 open issues 与 6464 forks 的巨大用户漏斗，以及仓库虽于 2026-09 重置历史但保留全部 star——说明星标主要积累于此前的 monorepo 时期（2022-03 创建，2025-07 已有桌面任务库迁移，见 `_database/sql/artcraft_migrations/`）。

## 9. 亮点与风险

**亮点**
1. 定位差异化的"AI 创作工艺"：把生成前置到场景预排（2D 合成 + 3D blocking + 摆姿），而非纯提示词——在同类聚合器中少见（README 特性表）。
2. 模型聚合深度真实：62 模型 + 7 provider 通道，含直连自有账号（Midjourney/Grok/fal/World Labs），能力清单服务端下发，客户端免发版扩展（`artcraft_router` + OmniGen 设计）。
3. Rust 工程质量高：27 万行 Rust、6 千+ 测试函数、sqlx 编译期校验、宽松解码 wire 层、cargo-hakari、SonarCloud/CodeRabbit 门禁、哈希钉死的加固 CI。
4. 双 UI 栈押注渐进 Rust 化：egui 原生应用复用全部客户端 crate，与 ROADMAP"More Rust, more speed / Remove dependence on hosted services"一致，为彻底去服务化留了技术路线。
5. AI 辅助开发纪律样本：house rules、分层 AGENTS.md（含子目录级规范）、机器人审查流水线，可作为大型 Rust+TS 仓库的 agent 协作模板。

**风险/局限**
1. 强依赖托管后端：生成、计费、登录都走 `api.storyteller.ai`，开源仓库无法独立运行完整服务（ROADMAP 已列为待解决目标）；API key 开通还受人工审批限制。
2. 仓库刚于 2026-09-11 重置历史：commit 数仅 70，长期可追溯性、bisect 与外部贡献考古都受影响（完整历史在 `artcraft-services`）。
3. 合规遗留物：`NOTICE` 承认打包了任天堂游戏音效、梗图、第三方天空盒、MMD 动作、Mixamo 渲染图等未授权素材（"will be phased out"），分发风险真实存在。
4. 体量与门槛：48 万行双栈代码 + wreq 的原生构建链（cmake/nasm/perl），外部贡献者与自建者成本高；贡献者仅 9 人，bus factor 低。

## 10. 总结评价

ArtCraft 是 FakeYou 团队押注"AI 影视创作 IDE"的开源桌面端：用 2D/3D 场景预排 + 62 模型聚合把 AI 生成做成可控工艺，工程上以一套 unusually 扎实的 Rust 客户端层和 AI 辅助开发纪律支撑，星数与 issue 漏斗说明其在 AI 创作者社区已获得真实采用。适合两类人：想用开源工具做 AI 图像/短片的创作者（下载发行版，自带账号或付费 credits），以及研究 Rust 桌面应用架构、多 provider 路由、浏览器指纹模拟、反爬客户端设计、或 AI-agent 协作工程规范的工程师（读 `artcraft_router`、`browser_emulation`、`artcraft-native/backend` 与各 AGENTS.md）。需要留意它目前不是"可自托管"的项目——核心生成能力仍系于其托管服务。
