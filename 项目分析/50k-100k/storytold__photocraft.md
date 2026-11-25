# storytold/photocraft 项目分析

> 星数 71747（快照 2026-10-24）｜主语言 Rust｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：**PhotoCraft 是一个用纯 Rust 从零净室（clean-room）重写的开源 Adobe Photoshop**——图层、蒙版、调整图层、图层样式、文字、矢量、笔刷引擎与真实 PSD 文件，原生跨平台（macOS/Windows/Linux/FreeBSD/Web）。（出处 `README.md`）

| 事实 | 值 | 出处 |
|---|---|---|
| 名称 | PhotoCraft（ArtCraft "Crafting Apps" 家族之一） | `README.md` |
| 星数 | 71,747（快照）；gh api 实测 75,220（2026-10-31） | 任务清单 + gh api |
| 主语言 | Rust（100%，workspace `unsafe_code = "forbid"`） | `Cargo.toml` |
| 许可证 | MIT OR Apache-2.0 双许可（GitHub 标注 Apache-2.0） | `Cargo.toml`、`LICENSE-MIT`/`LICENSE-APACHE` |
| 版本 | 0.17.0，edition 2024，rust-version 1.95 | `Cargo.toml` |
| 状态 | early alpha（README 明确"尚非日常专业工作的 Photoshop 替代品"） | `README.md` |
| 创建/最近推送 | 2026-09-30 / 2026-10-30（仓库仅一个月大） | gh api |
| 主页 | https://getartcraft.com/apps/photocraft | gh api |

## 2. 项目是什么，解决什么问题

PhotoCraft 要解决的问题非常直接：Photoshop 是图像编辑的事实标准，但闭源、订阅制、仅限特定平台；而开源侧的 GIMP 因历史包袱（8-bit 硬编码、GEGL 迁移十年、CMYK 缺失、PSD 导入有损）长期无法承接专业工作流。PhotoCraft 的答案是"从第一天就把 Photoshop 的语义当作数据建模"：位深（8/16/32f）、色彩模型（RGB/Gray/CMYK/Lab）、ICC 色彩管理、非破坏式图层（调整/填充/文字/形状/智能对象）全部是运行时数据而非核心假设（`docs/architecture.md` §1.1 "Avoiding GIMP's hole" 一节逐条对照 GIMP 的教训并立规）。

项目自称 clean-room：只研究 Photoshop 的行为与观感，实现全部来自公开规范（Adobe PSD 规范、ICC、ISO 32000 混合模式）与观察，绝不复制专有代码/着色器/资产（`docs/architecture.md` §1 原则 8、`AGENTS.md` §3）。PSD 支持是独立 crate，从 Adobe 公开规范手写：打开再保存后，psd-tools 测试集 309 个文件中 307 个渲染一致；未建模的原始块会被原样保留而非丢弃（`README.md` "PSD without compromise"、`crates/io/tests/it/corpus.rs`）。

最有个性的设计是"为 Agent 而生"：每个菜单项/工具/对话框都是命令注册表（500+ 条）里的一条命令，UI、CLI、JSON 控制通道、MCP 服务器四条前端走同一个 `Session::execute` 分发，"你能点到的，脚本和 AI agent 都能做到"（`README.md` "Built for agents"、`crates/engine/src/lib.rs`）。README 里所有截图都是通过 headless 控制通道离屏渲染的真实画面。

定位与诚实度值得一提：README 与 `ROADMAP.md` 公开承认处于 alpha——菜单接线率 628/628，但"接线率不等于行为"；特性广度约 76%、"可真实工作"约 45%，写出的 PSD 尚不能被 Photoshop 稳定接受（#1281、#2469），22/25 项性能预算未达标。这种量化自评在同类项目中罕见。

## 3. 技术栈与依赖

- **语言**：纯 Rust（edition 2024，MSRV 1.95），workspace 25 个库 crate + 4 个 app + xtask + 5 个 fuzz crate；实测 2,076 个 `.rs` 文件、约 76.9 万行（2026-10-31 HEAD，`find | wc -l`）。
- **UI**：egui/eframe 0.37 跑在 wgpu 上，无 webview 无 JavaScript；`Cargo.toml` 中 `[patch.crates-io]` 将整个 egui 家族与 winit 钉到 storytold 自己的 fork commit。
- **GPU**：wgpu（Metal/Vulkan/DX12/WebGPU），WGSL 着色器经 naga 翻译；浏览器版即 WebGPU/WebGL2（`crates/gpu`）。
- **并行**：rayon 按瓦片并行滤波；大半径模糊用 running-sum box 多核实现（`README.md`）。
- **序列化/CLI**：serde/serde_json（命令参数全是 JSON）、clap 4（CLI + man page 补全）。
- **自研关键件**：ICC 色彩管理引擎（`crates/cms`，纯 Rust，含四渲染意图/黑点补偿/软打样）、PSD/PSB 解析器（`crates/psd`）、相机 raw（`crates/raw`）、Affinity 文档只读解析（`crates/affinity`）。
- **第三方编解码**：image、zune-jpeg、flate2(zlib-rs)、OpenEXR、QOI 等；HEIC 经可选 `heif` feature（官方构建内置）。
- **测试**：proptest、criterion（bench）、egui_kittest（UI 快照/kit 测试）、cargo-nextest（`.config/nextest.toml`）。
- **浏览器**：trunk + wasm-bindgen，wasm 体积门禁 24 MiB（`Cargo.toml` `[profile.wasm-release]` 详细注释了逐 crate 的 opt-level 取舍）。

## 4. 目录结构与代码组织

Cargo workspace 单体仓库，按依赖分层（层号见 `AGENTS.md` §2，由 `cargo xtask layers` 在 CI 强制执行，规则源 `xtask/src/layers.rs`）：

| 目录 | 层 | 职责（实测行数） |
|---|---|---|
| `crates/tablet` | L0 | 数位笔输入（macOS AppKit/X11/Win32），全仓库唯一允许 unsafe 的 crate |
| `crates/doc` | L1 | 文档模型：图层树/蒙版/调整/样式/智能对象，纯数据+serde（5,416 行） |
| `crates/ops` `paint` `algo` `text` `vector` | L2 | 历史/撤销、笔刷引擎、成像算法（68,123 行）、文字排版、路径形状 |
| `crates/compose` `gpu` `format` | L3 | CPU 参考合成器（22,402 行，"oracle"）、wgpu 合成器、`.pcraft` 原生格式（原子保存） |
| `crates/io` `plugins` | L4 | 文档↔PSD/平铺格式映射（53,512 行）、沙箱 WASM 插件 |
| `crates/engine` | L5 | Session + 命令注册表（161,029 行，442 文件） |
| `crates/ui-egui` `automation` | L6 | egui 壳（277,577 行，最大 crate）+ MCP 服务器 |
| `crates/psd` `codecs` `cms` `raw` `heif` `affinity` 等 | 基础件 | 不依赖 workspace 的独立格式/色彩 crate |
| `apps/photocraft` | — | 桌面 app（eframe/wgpu）+ TCP 控制服务器 |
| `apps/photocraft-cli` | — | headless CLI（convert/info/run/batch/commands/mcp） |
| `apps/photocraft-web` `-web-boot` | — | 浏览器版（同一引擎编译到 wasm）与启动器 |
| `crates/*/fuzz` ×5 | — | psd/codecs/format/raw/affinity 的 cargo-fuzz 目标 |
| `xtask/` | — | `cargo xtask layers\|wasm\|ci\|stats\|corpus\|test-corpus\|parity\|perf\|scorecard` |
| `docs/`（80+ 文档）、`book/` | — | 架构、性能、格式 parity、本地化、安全文档与 mdBook 用户手册 |

组织方式评述：这是教科书级的"分层强制 + 引擎优先"workspace——UI 不可越层依赖引擎之下的任何渲染件，`psd`/`codecs`/`cms` 零内部依赖可独立发布。代价是 `ui-egui`（27.8 万行）与 `engine`（16.1 万行）两个巨型 crate 内聚了大量 `*_cmds.rs` 模块（`crates/engine/src/cmd.rs` 声明了 60+ 命令子模块），属于刻意的"按菜单域拆文件、按层拆 crate"折中。

## 5. 核心架构与关键实现

### 5.1 一切皆命令的引擎（`crates/engine/src/cmd.rs`）

命令注册表是整个项目的枢纽。每条命令是一个静态 `CommandSpec`，UI/CLI/控制通道/MCP 四端共用：

```rust
pub struct CommandSpec {
    pub id: &'static str,          // 如 filter.blur.gaussianBlur
    pub label: &'static str,
    pub menu: &'static [&'static str],  // ["Layer", "New"]，空=不在菜单
    pub shortcut: Option<&'static str>,
    pub params: &'static str,      // 人类/agent 可读的 JSON 参数说明
    pub enabled: Enabled,          // fn(&Session) -> Result<(), String>
    pub run: Run,                  // fn(&mut Session, &Value) -> Result<Value>
    pub journal: bool,             // 是否记入历史（查询类命令为 false）
}
```

`Registry` 建立 id→index 哈希索引；`list_entry()` 为每条命令输出含 `enabled`/`disabledReason`/`needsDocument` 的 JSON 行，专为 agent 判断"先开文档还是条件不满足"设计（#3664）。命令 id 直接沿用 `crates/ui-egui/src/menu_catalog.rs` 里 Photoshop 菜单的 id，628 个菜单项自动接线。

### 5.2 文档模型与 COW 瓦片（`crates/doc/src/lib.rs`、`crates/raster/src/lib.rs`）

文档模型是纯数据、无渲染、无 UI，`#![forbid(unsafe_code)]` 且 deny 全套 panic 类 clippy lint；`LayerContent` 枚举从 v1 起就包含 Adjustment/Fill/Text/Shape/Smart，非破坏式编辑是"构造即如此"。像素存储是稀疏 COW 瓦片：

```rust
/// A `Surface` is an infinite plane of 256×256 tiles. Missing tiles read as the
/// surface's default pixel ... Tiles are `Arc`-shared, so cloning a surface is
/// O(tiles) and mutation copies only the touched tiles. That makes undo
/// snapshots, background jobs and autosave cheap.
```

（`crates/raster/src/lib.rs` 模块注释）快照取档 O(图层数)，另有 `TileInterner` 对均匀色瓦片做整仓去重；文档内一切大负载（XMP、EXIF、原始 PSD 块）都 `Arc` 共享，快照永不深拷贝（`docs/architecture.md` §1 原则 5、#4535）。

### 5.3 双合成器互验（`crates/compose/src/lib.rs`、`crates/gpu`）

CPU 合成器是参考实现（"the oracle"），把 Photoshop 图层语义（混合模式×不透明度×填充不透明度、带密度的蒙版、剪贴组、pass-through 组、调整图层、Dissolve）编码在一处；GPU 合成器必须与其逐像素一致到 1/255，两者互为测试 oracle（`crates/compose/src/lib.rs` 模块注释、`README.md` "Under the hood"）。色彩经自研 ICC 引擎（`crates/cms`），GPU 端显示变换走矩阵/shaper 阶梯或 3D LUT 兜底（`docs/architecture.md` Current state）。

### 5.4 净室 PSD 往返（`crates/psd`、`crates/io/tests/it/corpus.rs`）

PSD crate 从 Adobe 公开规范独立实现，不依赖文档模型即可解析/写出；`io` 层负责映射到文档模型。质量护栏是三层：psd-tools 语料 307/309 渲染一致、混合语料 169/170、composite oracle 直接对比 Photoshop 自身合成结果（渐变插值/图层效果/形状描边/剪贴/填充不透明度）。语料通过 `cargo xtask corpus --all` 拉取并 sha256 校验（`README.md`、`docs/development.md`）。

## 6. 构建与运行

（均出自 `README.md` "Get started"、`AGENTS.md`、`docs/development.md`）

```sh
cargo run --release -p photocraft -- image.psd   # 桌面 app
cargo test --workspace                           # 全量测试
cargo run --release -p photocraft-cli -- run wave.psd \
  --cmd filter.sharpen.smartSharpen --params '{"amount":80}' --out out.png  # headless
photocraft-cli mcp                               # MCP 服务器（stdio）
cargo xtask corpus --all && cargo xtask test-corpus         # 语料测试
cargo xtask layers && cargo xtask wasm           # 分层检查 / wasm 门禁
cargo run -p photocraft-ui-egui --example snapshot          # 离屏 UI 快照
# 可选：日文字体 craft-fonts 作为构建输入
CRAFT_FONTS_DIR="$PWD/../craft-fonts" cargo run --release -p photocraft
```

发布产物覆盖 macOS universal DMG（签名+公证）、Windows x64/arm64/x86 MSI（代码签名）、Linux AppImage/Flatpak/deb/rpm/tarball（含 riscv64）、FreeBSD tarball 与静态 Web zip（`README.md` Downloads 表）。

## 7. 测试与工程质量

- **规模**：实测 HEAD 上 12,581 个 `#[test]`、分布于 1,240 个文件（ui-egui 与 engine 是大头）；`docs/architecture.md` 记录 2026-10-13 时约 8,040 个，两周内增长过半，与代码增速一致。
- **风格**：单元+集成+语料（corpus）+合成生成器+oracle（CPU/GPU 互验与 Photoshop 合成对比）+模糊测试（5 个 fuzz crate）+proptest+criterion 基准；UI 用 egui_kittest 离屏快照（`AGENTS.md` §3.6）。
- **CI**（`.github/workflows/ci.yml`）：平台矩阵（含 windows-arm64、freebsd、nix 专用 workflow）、clippy、**依赖分层检查**、测试/Doctest、无 heif 变体、语料测试、Web 构建+wasm 体积门禁、模糊测试矩阵（psd parse/descriptor/image_source_data/grd）、perf-nightly、sonarcloud；另有 CodeRabbit PR 评审（README 徽章、`.coderabbit.yaml`）。
- **代码质量纪律**（`AGENTS.md` §3，执行力见于各 crate 根的 lint 声明）：非测试代码禁 panic（deny unwrap/expect/panic/unreachable）、禁 unsafe（唯一豁免 `crates/tablet` 且每块带 SAFETY 注释）、输入派生数字按敌意处理（checked 算术、分配上限、递归深度限）、wasm 目标永不破坏（L0-L6 全部 `cargo check --target wasm32`）。
- 诚实的短板记录：`ROADMAP.md` 明确 22/25 性能预算未达标、953 个 open issue（2026-10-13；gh api 现值 3,477，含大量 agent 报告的细粒度问题）。

## 8. 社区与维护状态

- **活跃度**：仓库 2026-09-30 创建，一个月内 75k+ 星；最新提交 2026-10-30（PR #9893，说明 PR 编号已近万）；release 节奏近乎每日——v0.13.0（10-22）→ v0.16.0（10-25）→ v0.17.0（10-28）（gh api releases）。
- **贡献者**：API 首页返回满 100 人（截断，实际≥100）；`ROADMAP.md` 记录已合并 1,004 个 PR（2026-10-17），并以"Opus 5.5 agent 工时"为单位做剩余工作量校准——这是一个以 AI agent 为主力开发模式、同时用 AGENTS.md/文档/门禁把 agent 当一等公民管理的项目。
- **生态**：姊妹项目 VectorCraft/FilmCraft/LightCraft/PdfCraft/EffectCraft/DesignCraft（`README.md` "The Crafting Apps"）；社区 Gentoo overlay 包（`README.md` Community packages）；语料仓 photocraft-corpus、字体仓 craft-fonts、公共库 craft-libs 均为同 org 独立仓库。
- **治理**：双许可 + NOTICE + ATTRIBUTION.md 逐资产列许可；品牌 logo 商标受限（fork 须移除，`docs/brand/LICENSE-brand.txt`）；SECURITY.md + book/src/security 安全文档与 fuzz 策略。

## 9. 亮点与风险

**亮点**
1. **架构纪律罕见的严格**：构建期强制依赖分层（`cargo xtask layers`）、全仓 forbid unsafe、非测试代码零 panic、CPU/GPU 双合成器互验到 1/255——把"永不崩溃、永不断色"写成了可 CI 执行的规则。
2. **"一切皆命令"的引擎优先设计**：同一命令表喂饱 UI/CLI/控制通道/MCP 四端，天然成为 AI agent 可编程的图像引擎，README 所有截图即由该通道离屏产出，是这类设计少有的自证。
3. **净室 PSD 往返达到可用级**：307/309 语料渲染一致、未建模块原样保留，附带 composite oracle 与 sha256 校验的语料体系（`crates/psd`、`crates/io/tests/it/corpus.rs`）。
4. **量化自我评估文化**：`ROADMAP.md`/`docs/target-app-parity.md` 用实测数字（菜单 628/628、工具 56/68、面板 30/35）标定与 Photoshop 2026 的差距，不吹"替代品"。
5. **极致的工程元数据**：80+ 篇 docs、mdBook、逐性能专题文档（oil-paint-performance.md 等）、wasm 逐 crate opt-level 注释到 KB 级（`Cargo.toml`）。

**风险/局限**
1. **早期 alpha**：官方明示不能承担日常专业工作；写出的 PSD 尚不能被 Photoshop 稳定接受（#1281、#2469），22/25 性能预算未达标（`ROADMAP.md`）。
2. **AI 高速生成的可持续性存疑**：一个月 76.9 万行、日更 release、PR 编号近万，代码评审与长期维护压力巨大；`ui-egui` 单 crate 27.8 万行已有巨石化倾向。
3. **单一组织驱动**：核心引擎、公共库、字体、语料、egui/winit fork 全部押在 storytold 一个 org 上（`Cargo.toml` `[patch.crates-io]`），上游漂移风险集中。
4. **Photoshop 是移动靶**：生成式 AI 功能、插件生态两大缺口被官方列为需"owner 决策"的最大差距（`ROADMAP.md`），追赶成本只会更高。

## 10. 总结评价

PhotoCraft 是 2026 年"AI 辅助开发能造出什么"最极端的公开样本之一：一个团队用一个月时间、以近乎日更的节奏，写出 76.9 万行分层严整、测试过万、文档密度极高的纯 Rust Photoshop 复刻，并且没有因为速度而放弃净室原则与工程纪律——分层检查、零 panic、双 oracle、语料门禁都是真实跑在 CI 里的。它现在还不是谁的日常工具（alpha、45% 可用度），但其"引擎优先 + 一切皆命令 + 量化 parity"的路线图执行方式，已经比多数十余年的开源图像编辑器更接近 Photoshop 的骨架。适合三类人：想找可脚本化/agent 驱动图像引擎的开发者（CLI/MCP 即插即用）；想学习大型 Rust workspace 分层与测试体系建设的工程师（`AGENTS.md` + `docs/architecture.md` 是绝佳教材）；以及愿意陪伴 alpha、按 `docs/gaps.md` 挑活贡献的贡献者。想把它当 Photoshop 替代品的普通用户，建议等 beta（官方估算约 1,100–1,800 agent 工时之后）。
