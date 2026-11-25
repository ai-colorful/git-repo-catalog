# qarmin/czkawka 项目分析

> 星数 23000（快照 2026-10-24）｜主语言 Rust｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：用 Rust 写的全能磁盘清理器——重复文件、相似图片/视频、空目录、坏文件等 14 类"垃圾"一网打尽，核心扫描引擎 + GTK4/Slint 双 GUI + CLI 三形态。

| 项目 | 值 |
|---|---|
| 名称 | qarmin/czkawka（波兰语"打嗝"，v12.0 后主推姊妹前端 Krokiet"炸肉饼"） |
| 星数 | 23000（快照）；gh api 实测 35375（2026-11-21） |
| 语言 | Rust（GitHub 语言统计显示 Fluent，因 .ftl 翻译文件占多，实际代码为 Rust） |
| 许可证 | 多重许可：core/cli/GTK GUI 为 MIT；Krokiet/Cedinia 因 Slint 许可要求为 GPL-3.0-only；图标/测试素材 CC BY 4.0（见 `czkawka_core/Cargo.toml`、`krokiet/Cargo.toml`、`README.md` License 节） |
| 版本 | 12.0.2（2026-09-09 发布），edition 2024，最低 Rust 1.94.1（`czkawka_core/Cargo.toml`） |
| 最近活跃 | push 2026-11-16（gh api），持续活跃 |
| 代码规模 | 5 个主 crate 共 298 个 .rs 文件、约 6.88 万行（统计自 czkawka_core/cli/gui/krokiet/cedinia） |

## 2. 项目是什么，解决什么问题

Czkawka 是一个"文件清理全家桶"：作者 Rafał Mikrut（qarmin）受 FSlint 启发（README 致谢节明确说明），用 Rust 重写并大幅扩展了磁盘垃圾查找的品类。`czkawka_core/src/lib.rs` 中 `TOOLS_NUMBER = 14`，`czkawka_core/src/tools/` 下有 15 个工具模块（disk_usage 另算）：重复文件（按名/大小/哈希）、空文件夹、大文件、空文件、临时文件、相似图片、相似视频、同音乐（按标签或音频指纹）、失效符号链接、坏文件、坏扩展名、Exif 清除、视频优化器、坏文件名、磁盘占用分析。

它的差异化在于"同一引擎、多端形态"：`czkawka_core` 是无 UI 依赖的库，四个前端共用——GTK4 版 `czkawka_gui`、Slint 版桌面 GUI `krokiet`、Android 版 `cedinia`、命令行 `czkawka_cli`（README"Projects using Czkawka"节 + workspace 成员表 `Cargo.toml`）。核心库还被外部项目复用（czkawka-tauri、Python 绑定、page-dewarp 等，README 列举）。

工程上它强调三点（`AGENTS.md`"Project Goals"节）：一是性能优先，rayon 并行、blake3 哈希、感知哈希等高效算法，避免热路径多余分配；二是尽量少用非 Rust 原生依赖以便交叉编译，原生库（如 ffmpeg）按 feature 门控；三是缓存支持，第二次扫描远快于第一次（`common/cache.rs`）。

值得注意的版本转折：`Changelog.md` 12.0.0 节（2026-06-28）正式宣布 GTK4 版 Czkawka 停止发布二进制（12.0 为最后版本），用户转向 Slint 版 Krokiet；GTK 源码保留且保持可编译。即"GTK4 迁移已完成其历史使命、正被 Slint 替代"——以源码为准的现状是：`czkawka_gui` 仍在仓库内依赖 gtk4 0.11.5（`czkawka_gui/Cargo.toml`），但不再发版。

## 3. 技术栈与依赖

- 语言/编译：Rust（edition 2024，resolver 3，`Cargo.toml`），最低 rustc 1.94.1 跟随 Debian Sid（`AGENTS.md` Workspace Structure 节）。
- 哈希：blake3 / crc32fast / xxhash-rust(xxh3)（`czkawka_core/Cargo.toml` "Hashes for duplicate files" 节）。
- 图像：image 0.25（15 种格式 feature）、image_hasher 3.1（感知哈希）、fast_image_resize、hamming-bitwise-fast（汉明距离）；纯 Rust 扩展格式解码 rawer（RAW）、jxl-oxide（JPEG-XL）、imgalifier-heif/avif（`czkawka_core/Cargo.toml`）。
- 音视频：rusty-chromaprint（音频指纹）、symphonia（解码）、similario_core（视频视觉签名对比）；ffmpeg/ffprobe 以外部命令行方式调用（`czkawka_core/src/common/ffmpeg_utils.rs`、`src/helpers/ffprobe.rs`）。
- 坏文件检测：zip、lopdf（PDF）、sevenz-rust2、tar、bzip2-rs、lzma-rs、ttf-parser、quick-xml、toml、yaml-rust2、usvg 等（`czkawka_core/Cargo.toml` "Needed by broken files" 节）。
- 并发/异步：rayon（并行哈希）、crossbeam-channel（进度上报）、AtomicBool 停止标志；trash 5.3（回收站删除，Linux 可选 xdg-portal feature）。
- GUI：gtk4/gdk4 0.11.5（v4_6，`czkawka_gui/Cargo.toml`）；slint 1.17（Krokiet/Cedinia，`krokiet/Cargo.toml`）；rfd 文件对话框、arboard 剪贴板。
- CLI：clap 4.6 derive + clap_complete、indicatif 进度条、ctrlc（`czkawka_cli/Cargo.toml`）。
- i18n：i18n-embed + Fluent（.ftl），27 种语言（`czkawka_gui/i18n/` 目录计数），翻译经 Crowdin 分发 + AI 批量翻译（`misc/ai_translate/translate.py`，AGENTS.md i18n 节）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `czkawka_core/` | 共享扫描引擎库：`common/`（遍历、缓存、进度、公共 trait）、`tools/`（15 个工具模块）、`helpers/`（ffprobe、音频校验等） |
| `czkawka_cli/` | clap 驱动的命令行前端，纯薄封装（commands.rs/main.rs/parsers.rs/progress.rs 四个源文件） |
| `czkawka_gui/` | GTK4 前端（已停发版，代码保留）：`connect_things/`（信号连接）、`gui_structs/`、`helpers/` |
| `krokiet/` | 主力桌面 GUI（Slint）：`ui/*.slint` 声明 UI，`src/connect_*.rs` 回调接线，`shared_models.rs` 状态 |
| `cedinia/` | Android 触屏 GUI（Slint，cdylib），桌面版为副产物 |
| `data/` | desktop/metainfo 文件与图标（Flatpak 用） |
| `e2e/`、`ci_tester/`、`misc/` | 端到端测试 crate、CI 辅助 crate、脚本（翻译/死代码检测/打包/docker/nix） |

工具模块清单（`czkawka_core/src/tools/` 各子目录）：duplicate（重复文件）、empty_folder（空目录）、empty_files（空文件）、big_file（大文件）、disk_usage（磁盘占用）、temporary（临时文件）、similar_images（相似图片）、similar_videos（相似视频）、same_music（同音乐）、invalid_symlinks（失效链接）、broken_files（坏文件）、bad_extensions（坏扩展名）、bad_names（坏文件名）、exif_remover（Exif 清除）、video_optimizer（视频优化）——每个目录均为 core.rs/mod.rs/traits.rs/tests.rs 四件套。

CLI 子命令与工具一一对应（`czkawka_cli/src/commands.rs` Commands 枚举）：dup、empty-folders、big、disk-usage、empty-files、temporary、similar-images、similar-videos、same-music、invalid-symlinks、broken-files、bad-extensions、exif-remover、video-optimize、bad-names，另附 completions 子命令（clap_complete）。

组织方式评述：典型 cargo workspace 多包结构（`Cargo.toml` members 五 crate，exclude 测试辅助 crate）。核心组织原则是"库厚前端薄"——所有扫描/删除/缓存逻辑在 `czkawka_core`，前端只做参数收集与结果呈现；每个工具在 core 内是独立子模块（`core.rs` 实现 + `mod.rs` 类型/参数 + `traits.rs` + `tests.rs`），结构高度一致。代码风格治理极强：workspace 统一 lints（`Cargo.toml [workspace.lints]` 约 100 条 clippy 规则）、`AGENTS.md` 明确单文件 ≤500 行、`unwrap()` 仅限测试、`just fix` 作为强制质量门。

## 5. 核心架构与关键实现

### 5.1 重复文件查找：三级漏斗 + 多线程哈希（`czkawka_core/src/tools/duplicate/`）

流程为 Name → Size/SizeName → Hash 分级缩小候选（`core.rs` 中 `check_files_name`/`check_files_size`/`check_files_hash`）。哈希阶段两个关键设计：

- 预哈希只读文件头尾各 4KB（`PREHASHING_BUFFER_SIZE = 4 * 1024`），头尾重叠时整体单遍读完；全量哈希仅在预哈希撞车后进行（`mod.rs` `hash_calculation_limit`，225 行处有 `const_assert!` 保证缓冲区尺寸关系）。
- 每线程复用 2MB 缓冲区（`THREAD_BUFFER_SIZE = 2MB`，thread_local `RefCell<Vec<u8>>`），哈希算法经 `MyHasher` trait 统一适配 blake3/crc32fast（`mod.rs` 329-339 行）。

```rust
// czkawka_core/src/tools/duplicate/mod.rs
pub const PREHASHING_BUFFER_SIZE: u64 = 4 * 1024;
pub const THREAD_BUFFER_SIZE: usize = 2 * 1024 * 1024;
thread_local! {
    static THREAD_BUFFER: RefCell<Vec<u8>> = RefCell::new(vec![0u8; THREAD_BUFFER_SIZE]);
}
```

并行用 rayon 并显式调 `with_max_len` 控制任务粒度（`core.rs` 230 行 `.into_par_iter().with_max_len(rayon_max_len)`），进度字节量用 `Arc<AtomicU64>` 累加，取消靠轮询 stop flag（`hash_calculation` 循环内 `check_if_stop_received`，mod.rs 322 行）。此外硬链接检测（`filter_hard_links`）可在哈希前直接排除同一 inode 的"假重复"，Windows 上借助 file-id crate（`czkawka_core/Cargo.toml` cfg(windows) 依赖）。

### 5.2 相似图片：感知哈希 + 自研 BK 树（`czkawka_core/src/tools/similar_images/`、`common/bk_tree/`）

图片先经 `image_hasher` 计算感知哈希（支持多算法 `HashAlg`，通过 `re_exported` 暴露给前端，`lib.rs`），再在自研 BK 树里做汉明距离近邻搜索：距离函数用 `hamming_bitwise_fast`，并给 `Metric` trait 加了 `threshold_distance` 方法——超过阈值直接返回 None，让 `BKTree::find` 剪掉整棵子树（`similar_images/mod.rs` 101-112 行注释）。BK 树基于 bk-tree 0.5.0（MIT）改造为单 arena 存储（`common/bk_tree/mod.rs` 首行注释 + `struct BKNode { children: Vec<(u32,u32)> }`）。12.0 新增几何不变性（GeometricInvariance）与旋转/翻转容忍（`czkawka_cli/src/commands.rs` 引用该枚举）。相似视频侧同理：缩略图网格签名走 `similario_core`（tolerance 0..=20 线性映射到其 0.0..=0.5，`similar_videos/core.rs` 55 行注释），音频指纹走 `rusty-chromaprint` + `symphonia` 解码，ffmpeg 仅用于抽帧与转码（`common/ffmpeg_utils.rs`）。

### 5.3 缓存体系：版本化 bincode 持久化 + 自动清理（`czkawka_core/src/common/cache.rs`）

每个工具的结果按工具+参数+版本号命名落盘，如 `cache_duplicates_{HashType}_{VERSION}.bin`、`cache_similar_videos_{VERSION}__skip_..__dur_..bin`（各工具 `core.rs` 尾部 get_cache_file_name；版本常量 `CACHE_DUPLICATE_VERSION` 等均为 u8，cache.rs 28-34 行）。反序列化用 bincode 并设 8GB 上限（`MEMORY_LIMIT`），可选同步输出 JSON 便于人读。缓存目录经 `directories-next::ProjectDirs::from("pl","Qarmin",..)` 定位（`common/config_cache_path.rs` 124 行）。`cache/cleaning.rs` 做"有效期校验"：文件大小或 mtime 与缓存不符则剔除；`cleaning_timestamps.json` 记录时间戳，默认每 7 天全量清理过期缓存（`CACHE_CLEANING_INTERVAL_SECONDS = 7*24*60*60`）。12.0 因预哈希算法升级，明确声明旧缓存不兼容自动重建（`Changelog.md` 12.0.0 Breaking changes）。

### 5.4 工具统一抽象与取消/进度协议（`czkawka_core/src/common/`）

所有工具实现 `CommonData` trait（统一设置存取）与 `PrintResults`（CSV/JSON 导出）；扫描统一走 `DirTraversalBuilder`（`common/dir_traversal.rs`）——builder 模式注入 common_data、group_by 分组函数、checking_method、stop_flag、progress_sender。工具结构体构造→配置→在工作线程调 `find_*()`，进度经 crossbeam channel 发 `ProgressData`，停止标志为 `Arc<AtomicBool>`（AGENTS.md czkawka_core 节，与源码一致）。这套协议让 CLI/GUI 对"扫什么、怎么取消、进度长啥样"完全同构。

## 6. 构建与运行

```bash
# 构建全部前端（workspace）
cargo build --release                 # CLI: czkawka_cli, GUI: krokiet（czkawka_gui 需 GTK4 运行库）
cargo run --bin czkawka_cli -- dup -d /path -s hash -D aeo   # 重复文件示例（czkawka_cli/src/commands.rs after_help）

# Krokiet（Slint 前端，见 krokiet/README.md 与 AGENTS.md justfile 节）
just run krokiet          # debug 运行
just runr krokiet         # fast_release 运行
just fix                  # 强制质量门：格式化 + clippy --fix + Python 死代码检查（AGENTS.md）
just android              # 构建/安装/启动 Android 版 Cedinia

# 测试与基准
cargo test --workspace
cargo bench -p czkawka_core   # criterion：hash_calculation / disk_usage / bk_tree 三组（czkawka_core/benches/）
```

Linux 二进制 CI 用 cargo-zigbuild 面向 glibc 2.17 与 musl 交叉构建 x86_64/arm64，Krokiet 可选 vello/femtovg_wgpu/winit_software 多渲染后端打包（`.github/workflows/linux.yml`）。

## 7. 测试与工程质量

- 测试规模：czkawka_core 内 321 个 `#[test]`（grep 计数），42 个文件含 `#[cfg(test)]`；每个工具模块自带 `tests.rs`（如 duplicate/tests.rs 覆盖预哈希头尾重叠、短读稳定性等边界，mod.rs 399-535 行）；另有 `e2e/` 端到端 crate 与 `ci_tester/` CI 验证 crate（均 exclude 于 workspace）。
- 基准：criterion 三组 benchmark + misc 下独立性能 crate（test_read_perf/test_image_perf/test_compilation_speed_size，AGENTS.md misc 节）。
- CI：5 个 workflow（linux/windows/mac/android/quality，`.github/workflows/`），Linux 矩阵覆盖 gnu/musl × x86_64/arm64，Nightly tag 自动发布；quality.yml 跑 clippy/fmt。
- 代码质量观察：workspace lints 开了约百条 clippy 警告级规则（含 `unwrap_used`、`indexing_slicing`、`undocumented_unsafe_blocks`），release 保留 `panic = "unwind"` 以便捕获第三方解析库 panic 并注明理由、`overflow-checks = true` 全 profile 开启（`Cargo.toml` 30-46 行注释）；`profile.dev.package."*"` opt-level 3 保证 debug 跑图也快。README 的 AI Policy 节要求 AI 生成的提交/PR 标题带 `[AI]` 前缀且作者须能解释代码，作者人工评审每个 PR——配套 `AGENTS.md` 本身就是给 AI 编码代理的完整风格手册。

## 8. 社区与维护状态

- 活跃度：gh api 显示最近 push 2026-11-16（分析日前 5 天），297 个 open issue、1271 forks——单人项目能有此 issue 流量说明用户基数大。
- 贡献者结构（gh api contributors 核实）：总约 95 名贡献者，但 qarmin 一人 623 次提交，第二名仅 13 次——绝对的单人主导维护模式；翻译经 Crowdin 众包（AGENTS.md i18n 节）。
- 发版节奏（gh api releases 核实）：12.0.2（2026-09-09）→ 12.0.1（2026-07-29）→ 12.0.0（2026-06-28）→ 11.0.1（2026-02-21）→ 11.0.0（2026-02-14）→ 10.0.0（2025-08-18）→ 9.0.0（2025-03-16）→ 8.0.0（2024-10-11）；每年 2-3 个大版本 + 补丁版，另有 Nightly 滚动构建（README + linux.yml Release 步骤）。Changelog 由作者手写维护（AGENTS.md 明确禁止 AI 代改）。
- 生态：Flathub 上架（com.github.qarmin.czkawka）、crates.io 发布 czkawka_cli/core、社区衍生 Tauri 前端与 Python 绑定（README"Projects using Czkawka"节）。

## 9. 亮点与风险

**亮点**
1. 引擎/前端彻底分层：`czkawka_core` 无 UI 依赖、14+ 工具统一 trait 协议，四个前端与两个外部项目直接复用，是 Rust 桌面工具里少见的"库优先"设计（`czkawka_core/src/lib.rs`、AGENTS.md）。
2. 性能工程扎实：头尾 4KB 预哈希漏斗、thread_local 2MB 缓冲、rayon with_max_len 粒度控制、BK 树阈值剪枝、hamming 位运算加速，且每个设计点都能在源码注释中找到量化依据。
3. 版本化缓存 + 7 天自动清理 + 大小/mtime 失效校验，"第二次扫描更快"是可验证的机制而非口号（`common/cache.rs`）。
4. 质量门自动化到罕见程度：约百条 clippy 规则、Python 脚本检测 Slint/Fluent 死代码、`just fix` 一键修复，配合详细 AGENTS.md（面向人类与 AI 代理）。
5. 平台覆盖极广：Linux/Windows/macOS/FreeBSD/Android，x86/ARM/RISC-V，glibc 2.17 起步的 zigbuild 交叉构建。

**风险/局限**
1. 单人维护、bus factor 为 1：qarmin 占绝对提交主体，feature 方向完全取决于个人愿景（README"How to help"节直言新功能需先对齐"项目愿景"）。
2. 前端碎片化：GTK4 版停发但代码仍在、主力 Krokiet、Android Cedinia 三线并存，文档与用户心智成本上升；README 明示 CLI 功能强版要去看 fclones/rmlint。
3. 许可证分裂：core/cli MIT、Krokiet/Cedinia 因 Slint 为 GPL-3.0-only，商用集成需区分组件（README License 节）。
4. 视频工具依赖外部 ffmpeg/ffprobe 二进制，Android 端因此砍掉 VideoOptimizer（AGENTS.md cedinia 节）；最低 Rust 1.94.1 追随 Debian Sid，对老环境编译不友好。

## 10. 总结评价

Czkawka 是"一个人把文件清理工具做成全家桶"的样板项目：FSlint 的功能愿景 + Rust 的性能纪律 + 极强的工程自管理（lints、基准、缓存、AI 时代的新协作规范）。它对竞品的差异化不在单点最深（哈希速度拼不过 fclones，界面定制不如 DupeGuru），而在工具广度、多平台多形态和缓存驱动的重复扫描体验，README 对比表对此有客观自陈。适合普通用户当跨平台清理器直接用（选 Krokiet），适合 Rust 学习者精读它的 workspace 分层、rayon 哈希流水线与 BK 树实现，也适合想复用扫描能力的开发者直接依赖 `czkawka_core`。主要不确定性在单点维护的可持续性与三前端并存的收敛路径。
