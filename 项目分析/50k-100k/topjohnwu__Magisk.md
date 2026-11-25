# topjohnwu/Magisk 项目分析

> 星数 63439（快照 2026-10-24）｜主语言 Kotlin｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：Android 平台最主流的开源 systemless root 方案，通过修补 boot 镜像在系统分区不做任何修改的前提下实现 root、模块化系统修改与进程注入。

| 项目 | 内容 |
| --- | --- |
| 名称 | topjohnwu/Magisk（"The Magic Mask for Android"，README.MD） |
| 星数 | 63439（任务快照）；gh api 实测 63,636，仍在增长 |
| 主语言 | Kotlin（App 端）+ Rust/C++（核心 native 端） |
| 许可证 | GPL-3.0（LICENSE，README 声明含全部子模块） |
| 默认分支 | master（gh api） |
| 最近活跃 | push 2026-10-27，最新 release v31.1（2026-10-25），极活跃 |
| 代码规模 | 约 623 个文件：Kotlin/Java ~2.2 万行，Rust ~1.96 万行，C/C++ ~9.8 千行，Shell ~2.9 千行（实测统计） |

## 2. 项目是什么，解决什么问题

Android 的系统分区（/system 等）是只读且受 dm-verity/AVB 校验保护的，传统刷机修改系统需要解锁后直接改分区，既破坏完整性校验也不便回滚。Magisk 的解法是把所有改动"搬出"分区：在 boot 镜像的 ramdisk 中植入自己的 init 与 daemon，启动后通过 bind mount 在运行时叠加修改，从而做到 systemless——不写入任何系统分区、可随时完整卸载（README.MD、docs/details.md）。

它由四个核心能力组成（README.MD 明确列出）：
- **MagiskSU**：为应用提供 root 授权，带 SQLite 策略库、多用户模式与超时缓存；
- **Magisk Modules**：通过 "Magic Mount" 让模块修改只读分区，模块可含脚本、SELinux 规则与 system 属性；
- **MagiskBoot**：自称最完整的 Android boot 镜像解包/重打包工具，覆盖各家厂商私有头部与 AVB；
- **Zygisk**：在 Zygote（所有应用进程的孵化器）中注入代码的能力，向第三方模块提供稳定 API。

从安全视角看，它是 root 权限的分发器：Magisk app（管理端）与 root daemon 通过 Unix socket 通信，每次授权请求记录在案并可在 UI 中撤回；DenyList（v24 起取代 MagiskHide）可为指定应用提供隔离挂载命名空间。仓库 README 强调官方下载渠道只有 GitHub Releases，且"仅接受 Debug 构建的 bug 报告"。值得注意的是 README 首行即声明"This is not an officially supported Google product"（README.MD）。

## 3. 技术栈与依赖

**App 端（Kotlin）**（依据 app/gradle/libs.versions.toml、app/settings.gradle.kts）：
- Kotlin 2.4.20，Jetpack Compose（compose-ui 1.12、material3 1.4、navigation3 1.1.7），Room 2.8.5（su 日志/策略持久化）
- 网络：OkHttp 5.5（含 DoH）、Retrofit 3.0、Moshi（moshix KSP）
- 协议：Square Wire 7.0.3（protobuf）；root shell 交互使用作者自己的 libsu 6.0.0（jitpack）
- Gradle 多模块：`:apk`（UI）、`:core`（业务与 IPC）、`:test`，自定义构建逻辑在 app/build-logic

**Native 端（Rust + C++）**（依据 native/src/Cargo.toml、.gitmodules）：
- Rust workspace（edition 2024）：base / boot / core / init / sepolicy 六个 crate；`unwrap_used = "deny"` 的 clippy lint；release profile `opt-level="z"` + fat LTO + immediate-abort panic（极致体积优化）
- C++ 侧保留少量胶水代码（zygisk hook、deny、su 客户端等），通过 cxx（自维护 fork）与 Rust 互操作
- 关键第三方：lsplt（LSPosed 的 PLT hook 库）、nix、quick-protobuf、flate2/bzip2/lz4/lzma-rust2（压缩全家族）、RustCrypto 系（sha1/sha2/p256/p384/p521/rsa/x509-cert，用于 AVB 签名）
- 子模块全部为作者 fork 或定制：selinux、libcxx、cxx-rs、system_properties、crt0、xz-embedded（.gitmodules）

**构建编排**：Python 的 build.py 统一管理 ONDK（作者打包的 NDK+Rust 工具链）、Gradle 与打包产物。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| app/ | Android 应用：`apk/` Compose UI（安装/模块/超级用户/DenyList/日志/终端），`core/` 与 daemon 的 IPC、下载、刷入任务，`test/` 设备端 instrumentation |
| native/src/ | Rust workspace：`boot/`(magiskboot)、`init/`(magiskinit)、`core/`(magiskd+su+zygisk+module+deny+resetprop)、`sepolicy/`(magiskpolicy)、`base/`(公共库)；C++ 胶水与 `external/` 子模块 |
| scripts/ | 安装期 shell：boot_patch.sh、util_functions.sh、module_installer.sh、update_binary.sh、uninstaller.sh、AVD/cuttlefish 支持等 16 个脚本 |
| tools/ | 辅助工具（如 elf-cleaner，剥离二进制符号） |
| docs/ | 官方文档站源码（topjohnwu.github.io/Magisk）：build.md、install.md、details.md、boot.md 等 |
| build.py + config.prop.sample | 构建编排器与签名/版本配置样例 |
| .agents/ + AGENTS.md | 面向 AI 编码代理的仓库指南与三个子项目 skill（较新的实践） |

组织方式评述：这是"双技术栈单仓"结构——Gradle 多模块 App 与 Cargo workspace native 各自成体系，由 Python build.py 顶层编排。native 侧采用"Rust 为主、C++ 为辅"的渐进式重写架构（Rust 1.96 万行已明显超过 C++ 9.8 千行），公共代码沉淀在 base crate，各二进制 crate 之间通过 cxx 生成的 FFI 桥接（native/src/base/cxx_extern.rs、native/src/core/include/core.hpp）。

## 5. 核心架构与关键实现

### 5.1 MagiskBoot：boot 镜像解包与修补（native/src/boot/）

兼容性是它的核心价值：bootimg.hpp 除标准 boot 头外还处理 MTK（mtk_hdr）、HTC/高通 DHTB、MSM blob 等厂商私有头部以及 AVB footer/vbmeta 结构；zimage.rs 处理压缩内核，dtb.rs 处理设备树，payload.rs 支持 OTA payload 镜像，sign.rs 用 RustCrypto 实现签名。修补策略集中在 patch.rs——从 ramdisk 的 fstab 中**按字节流删除**校验/加密相关属性，使verity与强制加密失效：

```rust
// native/src/boot/patch.rs
pub fn patch_verity(buf: &mut [u8]) -> usize {
    unsafe fn match_verity_pattern(buf: &[u8]) -> Option<usize> {
        unsafe {
            match_patterns!(buf, b"verifyatboot", b"verify", b"avb_keys",
                b"avb", b"support_scfs", b"fsverity")
        }
    }
    remove_pattern(buf, match_verity_pattern)
}
```

另有 hexpatch（对镜像做十六进制模式替换）与 cpio.rs（853 行，ramdisk cpio 的增删改/备份/恢复）。安装脚本通过退出码区分"原厂镜像(0)/已修补(1)/被第三方工具修补过(2)"（scripts/boot_patch.sh）。

### 5.2 magiskinit 与 magiskd 守护进程（native/src/init/、native/src/core/daemon.rs）

安装时把 `/init` 替换为 magiskinit（scripts/boot_patch.sh 中 `add 0750 init magiskinit`），并注入 overlay.d 目录、压缩后的 magisk 二进制与 init-ld（LD_PRELOAD 库）。preload.c 以 `__attribute__((constructor))` 拦截 `security_load_policy`，先让 magiskd 修改完 SELinux 策略再放行真实加载（native/src/init/preload.c）——SELinux 策略注入由此完成。magiskd 常驻后监听主 socket，按 RequestCode 分发同步/异步请求（daemon.rs 的 handle_request_sync/async），状态存 SQLite（db.rs + core/include/sqlite.hpp，绑定系统 sqlite3）。

### 5.3 MagiskSU 授权链路（native/src/core/su/）

su 客户端（su.cpp 482 行）连接 daemon，daemon.rs 的 `su_daemon_handler` 读取 SuRequest，查询 SQLite 中的策略（Allow/Deny/Restrict），未授权时 `notify_app` 唤醒 Magisk app 弹窗询问；授权后 fork 出 root 子进程并 `exec_root_shell`，父进程 waitpid 回传退出码。策略结果有 3 秒内存缓存（`is_fresh`），支持 Restrict 模式（drop_cap 降权）与 owner-only/多用户模式。

### 5.4 Magic Mount 模块系统（native/src/core/module.rs，936 行）

核心是一个 FsNode BTreeMap 树：先扫描全部模块目录（处理 remove/disable/update/update.bin 标记），合并每个模块声称要替换的 /system 等分区路径，然后自底向上在 worker tmpfs 中创建白化节点并 bind mount 回真实路径（bind_mount 强制 MS_RDONLY）。系统目录被注入 magisk 二进制与 zygisk 桩（inject_magisk_bins/inject_zygisk_bins），同时兼容 riru 排除逻辑。模块还可携带 `sepolicy.rule` 与 `system.prop`（module.rs 调用 load_prop_file）。这是"不写分区却改系统"的关键实现。

### 5.5 Zygisk 进程注入（native/src/core/zygisk/）

hook.cpp 开头用一张 ASCII 图完整描述了引导流程：Zygisk 借助 Android"Native Bridge"机制被 runtime 加载，再通过 LSPlt 对 libandroid_runtime/libart/libnative_bridge 做 PLT hook，在 Zygote 特化（fork 应用进程）的各时机切入。模块 .so 以 fd 形式从 magiskd 传给 zygiskd64/32（entry.cpp），用 `android_dlopen_ext(ANDROID_DLEXT_USE_LIBRARY_FD)` 加载并调用其 `zygisk_companion_entry`；对应用侧通过 jni_hooks.hpp 挂钩 JNI 提供 Zygisk API（api.hpp 定义模块回调与 companion 进程通信）。

### 5.6 sepolicy 引擎与 DenyList（native/src/sepolicy/、native/src/core/deny/）

sepolicy 用一个自定义规则 DSL（statement.rs 解析，`rules!` 宏展开）生成 magisk 域：创建无约束的 proc/file/log 类型、`deny(all, ["kernel"], ["security"], ["load_policy"])` 阻止他人改策略、让 magisk 域 permissive（rules.rs magisk_rules()）。DenyList（deny/utils.cpp、cli.cpp）为列出的应用在隔离挂载命名空间中执行卸载类 mount 操作，`magisk --denylist` 提供 CLI。

## 6. 构建与运行

来自 docs/build.md 与 build.py（未实际执行，静态阅读）：

```bash
./build.py ndk      # 下载安装 ONDK（含 Rust 工具链）
./build.py all      # 构建全部并产出 Magisk APK
./build.py native   # 仅 native；./build.py gen 生成 FFI/protobuf 绑定
./build.py clippy   # Rust lint
# 单独调用 gradle/cargo 需加环境前缀：
scripts/env.py ./gradlew :apk:assembleDebug
```

- 配置与签名：复制 config.prop.sample 为 config.prop，填 keystore 四项（build.py、config.prop.sample）
- CI：.github/workflows/build.yml 在 macos-26 上以 `submodules: recursive` 检出，`./build.py -vr all` 出 release、再出 debug，上传产物与符号表；另有 test-build 矩阵验证多平台可构建性
- 环境注意：AGENTS.md 要求独立工具必须经 `scripts/env.py` 前缀注入环境变量

## 7. 测试与工程质量

- 自动化测试极少：Rust 侧仅 sepolicy/statement.rs、external/lz4-sys 有 `#[test]`（全仓 grep）；App 侧 app/test 模块是设备端 instrumentation runner（Runners.kt），CI 中未见大规模单测/集成测试（未确认是否有 emulator 测试 job 使用 .github/kvm.sh）
- 质量约束靠编译器与 lint 承担：Rust 全仓 deny `unwrap_used`、`opt-level="z"` + fat LTO、panic-immediate-abort（native/src/Cargo.toml）；release 二进制做符号剥离与 ELF 清理（tools/）
- 代码注释质量高：zygisk/hook.cpp 内嵌完整 ASCII 时序图，bootimg.hpp 标注了字段语义与 AOSP 出处；文档站（docs/）对 root 授予、boot 阶段、SELinux 规则有逐条解释

## 8. 社区与维护状态

- 维护极度活跃：最近 push 2026-10-27；最新 release v31.1 于 2026-10-25 发布（gh api）；快照星数 63,439，实测 63,636， forks 19,111，open issues 仅 40
- 贡献者约 297 人（contributors API 分页 last=298，含匿名未计）；核心开发高度集中于 topjohnwu 本人，README 的 bug 报告条款（只收 Debug 构建）也体现强个人主导
- 正在进行大规模 Rust 化重写：native 端已从纯 C++ 迁移到 Rust 为主（edition 2024），引入 AGENTS.md/.agents/skills 支持AI 辅助开发；Zygisk API 已到 v4（zygisk/api.hpp）
- 官方文档站持续维护，app 变更日志单独成文（docs/app_changes.md）

## 9. 亮点与风险

**亮点**
1. 工程完成度极高：从厂商私有 boot 头、AVB 签名、SELinux 策略库到 PLT hook，覆盖了 Android 启动链路的几乎每个细节，且全部开源（GPL-3.0）
2. Rust 重写带来现代化：类型安全 + `deny unwrap` + 体积极致优化，base crate 抽象干净（Utf8CStr、result、xwrap 等）
3. systemless 设计经久不衰：Magic Mount + ramdisk 注入 + overlay.d 的组合可完整卸载，风险可控
4. Zygisk 的引导设计与文档化程度（hook.cpp 内嵌时序图）在同类项目中罕见，形成了独立模块生态（官方提供 zygisk-module-sample）
5. 自建配套生态：libsu、ONDK、Compress/SELinux 等全部 fork 自持，供应链可控

**风险/局限（客观记录，不含操作指导）**
1. 法律与合规风险：解锁 bootloader 通常丧失厂商保修；root 后设备完整性证明（Play Integrity 等）会失败，部分银行/企业应用拒绝运行；在受管设备上使用可能违反组织安全策略——这些是设备方/平台方的政策决定，与本项目代码本身无关（docs/faq.md 有相关说明）
2. 高风险操作面：修改 boot 镜像与覆盖 /init 的操作一旦与具体设备/固件不匹配可导致无法启动；项目以"仅官方 GitHub Releases + 只接受 Debug 构建的报告"降低误用面
3. 攻击面固有存在：任何持有 root 的应用/模块即可完全控制系统；项目自身通过 APK 签名校验（docs/build.md：release 签名不匹配的 Magisk app 会被 daemon 拒绝并强制卸载）与 DenyList 缓解，但生态第三方模块无审核
4. 测试薄弱：几乎没有自动化回归测试，正确性高度依赖作者手工验证与社区反馈，大型重构（Rust 化）期间的回归风险靠 CI 构建矩阵兜底

## 10. 总结评价

Magisk 是 Android 定制生态的事实标准：它把"root"从一个粗暴的分区改写问题，做成了一整套包含 boot 镜像工程、init 劫持、SELinux 策略编译、bind mount 文件系统与进程注入的系统工程，且正在用 Rust 完成核心的现代重写，代码质量、文档与工程配套（libsu、ONDK、文档站、AI 协作指南）都属顶级水准。它适合三类人：想理解 Android 启动与安全架构的工程师（magiskboot/sepolicy/zygisk 是极好的教材）、需要在受控环境研究 root 检测与对抗的安全研究者，以及构建系统级模块的开发者；普通用户则应清楚 root 的保修、合规与应用兼容代价后再决定是否使用。
