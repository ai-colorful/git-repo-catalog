# shadowsocks/shadowsocks-android 项目分析

> 星数 36814（快照 2026-10-24）｜主语言 Kotlin｜档位 20k-50k｜分析日期 2026-11-24

## 1. 项目速览

Shadowsocks for Android 是 shadowsocks 协议的官方 Android 客户端，通过 Android `VpnService` 把整机流量导入本地 SOCKS5 代理，由 Rust 实现的 `ss-local` 转发到远端服务器。项目 2013 年底由 Max Lv 创建，现由 Mygod Studio 长期维护，同时发布手机版与 Android TV 版两个应用（`README.md`、`AUTHORS`）。

| 项目 | 值 | 出处 |
|---|---|---|
| 名称 | shadowsocks-android（应用包名 `com.github.shadowsocks` / `com.github.shadowsocks.tv`） | `mobile/build.gradle.kts`、`tv/build.gradle.kts` |
| 星数 | 36814（快照 2026-10-24） | 任务参数 |
| 主语言 | Kotlin（104 个 .kt 文件，约 11.2k 行；另有 NDK 构建的 C 依赖与 Rust 子模块） | 仓库文件统计 |
| 许可证 | GPL-3.0-or-later | `LICENSE`、`README.md` |
| 当前版本 | 5.3.5-nightly（versionCode 5030550） | `buildSrc/src/main/kotlin/Helpers.kt` |
| SDK 范围 | minSdk 23 / targetSdk 36 / compileSdk 36，JDK 11 字节码 | `buildSrc/src/main/kotlin/Helpers.kt` |
| 最近活跃 | 源码快照文件 mtime 为 2026-02-10；最近提交日期未确认（无 git 历史） | tarball |
| CI | CircleCI（构建 + 单元测试）、GitHub Actions（模拟器 E2E） | `.circleci/config.yml`、`.github/workflows/e2e-test.yml` |

## 2. 项目是什么，解决什么问题

Shadowsocks 是一种基于 SOCKS5 的加密代理协议：客户端在本机监听 SOCKS5 端口，把流量加密后转发给远端 shadowsocks 服务端。本项目是其在 Android 上的参考实现，目标是以普通应用权限（无需 root）为整台设备提供透明代理。

核心难点在于 Android 普通应用无法修改系统路由表，唯一合法的全局接管手段是 `android.net.VpnService`：系统建立一个 TUN 虚拟网卡，应用拿到其文件描述符后自行处理里面的 IP 包。本项目的做法是用 C 写的 tun2socks 把 TUN 包转成 SOCKS5 流量，交给 `ss-local` 加密发出（`core/src/main/java/com/github/shadowsocks/bg/VpnService.kt`）。

除了 VPN 模式，项目还提供两种仅代理、不接管全局的模式：`ProxyService` 只启动 `ss-local` 提供 SOCKS5 端口；`TransproxyService` 额外启动 redsocks，供 root 用户配合 iptables 做透明代理（`core/src/main/java/com/github/shadowsocks/bg/ProxyService.kt`、`TransproxyService.kt`）。

功能层面，项目支持多配置文件管理、`ss://` URL 与 SIP002 JSON 导入、订阅自动更新、按应用分流（per-app proxy/bypass）、ACL 分流规则（绕过局域网 / 绕过中国大陆 / GFWList / 自定义规则）、UDP fallback 配置、SIP003 原生插件（v2ray-plugin、kcptun、simple-obfs 等）、Direct Boot、Tasker 集成与快捷开关 Tile（`core/`、`mobile/` 各源码文件）。

## 3. 技术栈与依赖

**语言与运行时**
- Kotlin 2.2.0 + 协程 1.10.2：核心服务逻辑全部基于 `suspend` 函数与 `CoroutineScope`（`gradle/libs.versions.toml`）。
- C（ndk-build）：tun2socks（badvpn）、redsocks、libevent、libancillary 四个子模块（`.gitmodules`、`core/src/main/jni/Android.mk`）。
- Rust：`shadowsocks-rust` 子模块编译为 `libsslocal.so`，即实际的协议实现（`core/build.gradle.kts` 的 `cargo {}` 块）。

**关键第三方库**（`gradle/libs.versions.toml`）
- AndroidX Room 2.8.4 + KSP：配置文件与键值对持久化，带 schema 导出（`core/schemas/`）。
- AndroidX WorkManager 2.11.0（含 multi-process）：ACL 与订阅后台同步。
- dnsjava 3.6.4：解析/构造 DNS 报文，实现本地 DNS 中继。
- Firebase Analytics + Crashlytics：启动/停止事件与崩溃上报（`Core.kt`）。
- Timber：日志树，同时把日志转发至 Crashlytics。
- ML Kit barcode-scanning + ZXing：扫码导入配置（`mobile/build.gradle.kts`）。
- AndroidX Leanback：TV 版界面（`tv/build.gradle.kts`）。
- Mozilla `rust-android-gradle` 0.9.6：把 cargo 构建接入 Gradle。

## 4. 目录结构与代码组织

Gradle 多模块工程，`settings.gradle.kts` 声明 `:core`、`:plugin`、`:mobile`、`:tv` 四个模块，另有 `buildSrc` 承载共享构建逻辑。

| 目录 | 职责 |
|---|---|
| `core/` | 核心库（Android library）：后台服务、VPN 实现、进程守护、数据库、ACL、DNS、插件加载；含 `src/main/jni/`（NDK 构建）与 `src/main/rust/`（Rust 子模块入口） |
| `plugin/` | 独立发布的插件开发 SDK（`com.github.shadowsocks:plugin:2.0.1`），定义插件与宿主间的契约 |
| `mobile/` | 手机/平板应用：Fragment 为主的传统 View UI、设置页、扫码、Tasker 集成 |
| `tv/` | Android TV 应用，基于 Leanback，复用 `:core` |
| `buildSrc/` | `setupCommon()` / `setupCore()` / `setupApp()` 三个共享配置函数 |
| `core/gfwlist/` | 生成内置 GFWList ACL 资源的 Perl/Python 脚本 |
| `.github/`、`.circleci/`、`.travis.yml` | Issue 模板、FAQ、E2E workflow、CircleCI 配置、遗留 Travis 配置 |

依赖方向：`mobile`/`tv` → `core` → `plugin`（`core/build.gradle.kts` 使用 `api(project(":plugin"))`）。UI 层不直接接触任何原生进程，所有与服务相关的交互都经 `core` 暴露的 AIDL 接口完成。

## 5. 核心架构与关键实现

### 5.1 服务骨架：BaseService.Interface 模拟多继承

三个服务（`VpnService` / `ProxyService` / `TransproxyService`）都实现 `BaseService.Interface`，共享状态机、通知、进程池与启动流程，仅覆写差异部分。作者在注释中说明这是"用接口默认方法模拟多继承"（`core/src/main/java/com/github/shadowsocks/bg/BaseService.kt`）。

启动流程集中在 `Interface.onStartCommand`：读取当前 Profile → 构造 `ProxyInstance` → 注册关闭/重载广播 → 切换到 `Connecting` → 协程中依次执行 `Executable.killAll()`、`preInit()`、`startProcesses()`、`scheduleUpdate()`，成功后进入 `Connected`；任何异常都被包装成 `ExpectedException` 并通过 `stopRunner` 以通知形式提示用户。状态枚举为 `Idle / Connecting / Connected / Stopping / Stopped`。

服务运行在独立进程 `:bg` 中，并声明 `directBootAware="true"`，使得开机未解锁时也能自动启动（`core/src/main/AndroidManifest.xml`）。UI 进程通过 `ShadowsocksConnection` 绑定服务，并用 `IShadowsocksService` AIDL 接口获取状态、注册回调、订阅流量统计（`core/src/main/aidl/.../IShadowsocksService.aidl`、`core/src/main/java/com/github/shadowsocks/aidl/ShadowsocksConnection.kt`）。

### 5.2 VPN 服务：TUN 建立与 fd 传递

`VpnService.startVpn()`（`core/src/main/java/com/github/shadowsocks/bg/VpnService.kt`）是全局接管的核心：

```kotlin
val builder = Builder()
        .setMtu(VPN_MTU)
        .addAddress(PRIVATE_VLAN4_CLIENT, 30)      // 172.19.0.1
        .addDnsServer(PRIVATE_VLAN4_ROUTER)        // 172.19.0.2
if (profile.proxyApps) { /* addAllowed/DisallowedApplication 逐包 */ }
else builder.addDisallowedApplication(me)         // 默认排除自身，防止回环
when (profile.route) {
    Acl.ALL, Acl.BYPASS_CHN, Acl.CUSTOM_RULES -> builder.addRoute("0.0.0.0", 0)
    else -> resources.getStringArray(R.array.bypass_private_route).forEach { ... }
}
val conn = builder.establish() ?: throw NullConnectionException()
```

TUN 建立后，通过 Unix domain socket 把 TUN fd 发送给 tun2socks 进程（`sendFd()` 使用 `LocalSocket.setFileDescriptorsForSend` + 辅助数据传递），并带指数退避重试。tun2socks 以命令行参数启动，把 TUN 流量转发到 `ss-local` 的 SOCKS5 端口，DNS 则指向本地 DNS 中继端口：

```kotlin
val cmd = arrayListOf(File(applicationInfo.nativeLibraryDir, Executable.TUN2SOCKS).absolutePath,
        "--netif-ipaddr", PRIVATE_VLAN4_ROUTER,
        "--socks-server-addr", "${DataStore.listenAddress}:${DataStore.portProxy}",
        "--tunmtu", VPN_MTU.toString(), "--sock-path", "sock_path",
        "--dnsgw", "127.0.0.1:${DataStore.portLocalDns}", "--loglevel", "warning")
```

另一个关键设计是 `ProtectWorker`：原生进程自己发出的 socket 若不"protect"就会被 TUN 再次捕获形成回环。`ProtectWorker` 在 `protect_path` 上监听，接收原生进程传来的 fd，优先用 `underlyingNetwork.bindSocket(fd)` 绑定到真实物理网络，失败则退回 `VpnService.protect()`。底层网络由 `DefaultNetworkListener` 跟踪，并根据 Android 版本选用 `registerBestMatchingNetworkCallback` / `requestNetwork` / `registerDefaultNetworkCallback`（`core/src/main/java/com/github/shadowsocks/net/DefaultNetworkListener.kt`）。

### 5.3 与原生层的交互方式

项目**不使用 JNI**，Kotlin 与原生代码的边界全部是"进程 + 文件 + Unix socket"：

| 原生组件 | 产物 | 启动方式 | 通信方式 |
|---|---|---|---|
| shadowsocks-rust（ss-local） | `libsslocal.so` | `ProcessBuilder`，传 `-c config.json` | JSON 配置文件；`stat_main` socket 回传 16 字节 tx/rx；`local_dns_path` socket 接收 DNS 查询 |
| tun2socks（badvpn + lwIP） | `libtun2socks.so` | `ProcessBuilder` | `sock_path` socket 接收 TUN fd；`protect_path` socket 请求 protect |
| redsocks | `libredsocks.so` | `ProcessBuilder`，传 `-c redsocks.conf` | 配置文件 |

把可执行文件命名为 `lib*.so` 是为了利用 Android 的 `nativeLibraryDir` 让系统赋予其可执行权限（`core/src/main/jni/build-shared-executable.mk` 基于 NDK 的 `build-executable.mk` 改造，产物为 `EXECUTABLE` 类别但以 `.so` 命名；`buildSrc` 中 `useLegacyPackaging = true` 保证不被压缩）。

`ProxyInstance.start()` 负责生成 ss-local 的 JSON 配置并启动：校验加密方法（拒绝已弃用的 `chacha20`/`salsa20`/`aes-192-gcm`，校验 2022 系列 BLAKE3 密钥长度），注入插件路径与选项，声明 `locals` 数组（SOCKS5 入口 + DNS 中继入口），`dns` 指向 `unix://local_dns_path`（`core/src/main/java/com/github/shadowsocks/bg/ProxyInstance.kt`）。

`GuardedProcessPool` 为每个原生进程提供守护：stdout/stderr 各一个线程转发到 Timber，等待退出码；若进程存活不足 1 秒即退出视为致命错误，否则自动重启并回调（tun2socks 重启后需重新 `sendFd`）；停止时先 SIGTERM/`destroy()`，1 秒后 `destroyForcibly()`（`core/src/main/java/com/github/shadowsocks/bg/GuardedProcessPool.kt`）。

`LocalDnsWorker` 在 `local_dns_path` 上接收长度前缀的 DNS 原始报文，交给 `DnsResolverCompat` 在底层网络上解析（API 29+ 用 `DnsResolver.rawQuery`，API 23-28 用 `Network.getAllByName` 重新组包），失败时返回 SERVFAIL（`core/src/main/java/com/github/shadowsocks/bg/LocalDnsWorker.kt`、`net/DnsResolverCompat.kt`）。`TrafficMonitor` 则在 `stat_main` socket 上读取 ss-local 推送的 16 字节小端 tx/rx 总量，计算速率并通过 AIDL 回调给 UI。

### 5.4 插件体系：ContentProvider 作为二进制分发通道

SIP003 插件（obfs、v2ray、kcptun 等）作为独立 APK 安装。宿主通过 `PluginContract` 定义的 `plugin://com.github.shadowsocks/<id>` URI 与 `ACTION_NATIVE_PLUGIN` 查询 ContentProvider，再按三级策略获取可执行文件（`core/src/main/java/com/github/shadowsocks/plugin/PluginManager.kt`）：

1. **faster**：读取 provider 的 `METADATA_KEY_EXECUTABLE_PATH`，直接指向插件 APK 的 `nativeLibraryDir`，无需启动插件进程；
2. **fast**：调用 `ContentResolver.call(METHOD_GET_EXECUTABLE)` 获取绝对路径；
3. **slow**：`query` 列出文件清单，逐个 `openInputStream` 拷贝到宿主私有目录并 `chmod`。

宿主只信任与自身签名相同或内置于 `trustedSignatures` 的两枚证书（@Mygod、@madeye）。`plugin/` 模块是对外发布的 SDK，提供 `NativePluginProvider`、`ConfigurationActivity`、`HelpActivity` 等基类，`CHANGES.md` 显示当前 2.0.1，并引入 `__android_vpn` 选项替代旧的 `-V` 参数。

### 5.5 数据与规则

- Room 数据库 `PrivateDatabase`（version 29）含 `Profile` 与 `KeyValuePair` 两张表，手写 Migration 26-29，schema JSON 导出到 `core/schemas/` 供迁移测试；`enableMultiInstanceInvalidation()` 配合 `:bg` 进程的 `MultiInstanceInvalidationService` 实现跨进程同步（`core/src/main/java/com/github/shadowsocks/database/PrivateDatabase.kt`）。
- `Profile` 同时是 Room 实体、Parcelable，并实现了 SIP002 JSON 与 `ss://` URI 的双向解析（`database/Profile.kt`）。
- `Acl` 类解析自己的 ACL 文本格式（`[bypass_all]` / `[proxy_list]` / `[bypass_list]` / `#IMPORT_URL <...>`），`flatten()` 递归拉取远程规则，由 `AclSyncer` 用 WorkManager 定期刷新（`acl/Acl.kt`）。内置规则位于 `core/src/main/assets/acl/`。

## 6. 构建与运行

前置条件（`README.md`）：JDK 11+、Android SDK 与 NDK、Rust 工具链及四个 Android target（`rustup target add armv7-linux-androideabi aarch64-linux-android i686-linux-android x86_64-linux-android`）、Python 3（rust-android-gradle 的链接包装脚本需要）。

```bash
git clone --recurse-submodules https://github.com/shadowsocks/shadowsocks-android.git
./gradlew assembleDebug                                   # 构建 mobile + tv 调试包
./gradlew assembleDebug -PCARGO_PROFILE=debug             # 以 debug profile 编译 Rust（CI 用法）
./gradlew assembleDebug -PTARGET_ABI=x86_64               # 仅编译单一 ABI
./gradlew assembleDebug check -PCARGO_PROFILE=debug       # CircleCI 完整任务：构建 + lint + 测试
bash test-e2e.sh                                          # 模拟器端到端测试（需 ssserver 与 AVD）
./release.sh <version>                                    # 重命名 release APK 便于发布
```

模块级开关：`-PTARGET_ABI`（arm / arm64 / x86 / x86_64）与 `-PCARGO_PROFILE`（`core/build.gradle.kts`）。TV 版另有 `freedom` / `google` 两个 product flavor（`tv/build.gradle.kts`）。

## 7. 测试与工程质量

- **单元测试**：`core/src/test`（`KeyValuePairTest`）与 `plugin/src/test`（`PluginOptionsTest`），共 2 个文件约 100 行，覆盖极薄。
- **Instrumented 测试**：`core/src/androidTest` 含 `AclTest`、`MigrationTest`（基于导出的 schema 做迁移验证）、`ProfileTest`、`SubnetTest`，共 4 个文件约 190 行。
- **E2E 测试**：`test-e2e.sh`（390 行）在 GitHub Actions 的 x86_64 模拟器（API 34）上启动 `ssserver`，安装 APK，用 `run-as + sqlite3` 改写默认配置，点击 FAB 连接 VPN 并验证连通性，截图作为 artifact（`.github/workflows/e2e-test.yml`）。
- **CI**：CircleCI 使用 `circleci/android@2025.04.1-ndk` 镜像执行 `assembleDebug check`，Rust 1.86.0（`.circleci/config.yml`）；`.travis.yml` 仍在仓库中但 README 提到的 Travis 状态已过时。
- **静态检查**：`detekt.yml` 关闭了 comments、MagicNumber 等规则；Android Lint 在 `buildSrc` 中把翻译缺失降级为 informational；Room schema 导出启用了增量。
- **代码质量观察**：Kotlin 代码风格紧凑，大量使用 Kotlin 扩展与默认参数，关键路径有详尽的平台 bug 注释（如 issuetracker 链接）；但也使用了 `GlobalScope`、已弃用的 `actor`、反射读取 `pid` 等手段，可读性与可移植性有一定代价。

## 8. 社区与维护状态

- **维护者结构**：`AUTHORS` 与各源码头显示 Max Lv（@madeye）为原作者，Mygod Studio 为当前主要维护者；`.github/FUNDING.yml` 仅列出 `Mygod`。贡献者总数未确认（无 gh 元数据）。
- **发布节奏**：`plugin/CHANGES.md` 记录插件 SDK 从 0.0.1 到 2.0.1；主应用无 CHANGELOG，`buildSrc` 中版本为 5.3.5-nightly，README 提供 Google Play 正式版与 beta 渠道。具体发布频率未确认。
- **社区入口**：`CONTRIBUTING.md` 指向 GitHub Issues 与 Discourse 论坛；`.github/ISSUE_TEMPLATE/` 提供 bug / feature / questions 三类模板；翻译通过 POEditor 协作（`README.md`、`translate.py`）。
- **工程活跃度信号**：依赖版本较新（AGP 8.13.2、Kotlin 2.2.0、Room 2.8.4、compileSdk 36），存在 16KB page size 适配（`Application.mk` 的 `APP_SUPPORT_FLEXIBLE_PAGE_SIZES`、`core/build.gradle.kts` 的 `-z,max-page-size=16384`），E2E 流水线近期有维护迹象，表明项目处于持续维护状态。

## 9. 亮点与风险

**亮点**
1. 进程隔离架构清晰：UI 与服务分进程，协议实现完全交给独立 Rust 进程，原生崩溃不会直接拖垮 UI，且便于替换上游 shadowsocks-rust 版本。
2. 不依赖 JNI 的原生互操作方案成熟：fd 传递、Unix socket、WorkManager 等手段组合解决了 TUN 接管、protect 回环、DNS 中继、流量统计等 Android VPN 应用的经典难题。
3. 对 Android 平台碎片化处理细致：`DefaultNetworkListener`、`DnsResolverCompat`、Direct Boot、API 23/28/29/31 分支处理均有多版本适配。
4. 插件体系设计优雅：通过 ContentProvider 分发二进制并支持三级加速路径，兼顾兼容性与启动性能，SDK 独立发布并有变更日志。
5. 具备真实的端到端自动化测试，覆盖"启动模拟器 → 起服务端 → 连接 VPN → 验证连通"的完整链路。

**风险与局限**
1. 单元测试覆盖薄弱，核心服务逻辑（`BaseService`、`GuardedProcessPool`）主要依赖 E2E 与人工验证。
2. 构建门槛高：需要 NDK + Rust + Python + 四个 Rust target，子模块未拉取时无法构建；本次快照中 `core/src/main/jni/*` 与 `core/src/main/rust/shadowsocks-rust` 均为空目录。
3. 文档陈旧：README 仍提及 Travis CI，部分注释链接指向已关闭的 Google issue tracker；无主 CHANGELOG。
4. 代码中存在若干依赖实现细节的手段（反射读取 `pid`、`GlobalScope.actor` 弃用 API），随 Kotlin/AGC 升级存在维护成本。
5. 集成 Firebase Analytics 与 Crashlytics，对注重隐私的用户或 F-Droid 分发而言是额外考量（`Core.kt`）。

## 10. 总结评价

shadowsocks-android 是 Android 平台上历史最久、工程最成熟的 VPN 类客户端之一。它的核心价值在于一套被广泛借鉴的"Kotlin 服务层 + 独立原生代理进程 + tun2socks"架构：通过 Unix socket 传递 fd、protect 回环防护、本地 DNS 中继、守护进程池等机制，在普通应用权限下实现了稳定的全局代理，并把协议实现外包给上游 Rust 库以降低维护负担。同时，其基于 ContentProvider 的 SIP003 插件框架为生态扩展提供了清晰边界。

适合人群：需要理解 Android `VpnService` 真实落地的开发者（TUN 与 fd 传递、per-app 路由、网络切换处理）、要为 Android 设计"宿主 + 原生插件"架构的工程师，以及希望在 shadowsocks-rust 之上做 Android 封装的贡献者。阅读源码建议顺序：`BaseService.kt` → `VpnService.kt` → `ProxyInstance.kt` / `GuardedProcessPool.kt` → `net/` 目录 → `plugin/PluginManager.kt`。需注意其单元测试薄弱、构建链路较重，作为学习材料应聚焦架构而非直接复制实现。
