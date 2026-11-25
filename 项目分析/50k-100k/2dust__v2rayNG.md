# 2dust/v2rayNG 项目分析

> 星数 64535（快照 2026-10-24）｜主语言 Kotlin｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：Android 平台上最主流的 V2Ray/Xray 代理客户端，通过 Android VpnService 将系统流量接入内嵌的 Xray 内核，支持多种协议与订阅管理。

| 项目 | 事实 |
|---|---|
| 名称 | 2dust/v2rayNG |
| 星数 | 64535（快照 2026-10-24；gh api 显示 65022 watchers，2026-10-28） |
| 主语言 | Kotlin（约 214 个 .kt 文件、3.16 万行，`V2rayNG/` 目录统计） |
| 许可证 | GPL-3.0（`LICENSE`） |
| 创建时间 | 2019-05-15（gh api `repos/2dust/v2rayNG`） |
| 最近活跃 | 最近 push 2026-10-28；最新 release 2.3.14 发布于 2026-10-28（gh api） |
| 当前版本 | versionName 2.3.14 / versionCode 754（`V2rayNG/app/build.gradle.kts`） |
| 平台要求 | minSdk 24（Android 7.0+），target/compileSdk 37 |
| Forks | 8465（gh api） |

## 2. 项目是什么，解决什么问题

v2rayNG 是桌面客户端 v2rayN（同作者）的手机版（`README.md` 明确说明"v2rayNG 是手机版，电脑版请访问 v2rayN"）。它解决的核心问题是：把 Xray-core / v2fly-core 的代理能力装进 Android 系统级 VPN 通道，让任意应用的网络流量无需 root 即可按路由规则分流（国内直连、代理出境、广告屏蔽等）。

作为"壳 + 内核"架构的客户端，v2rayNG 自身不实现任何代理协议。真正做协议落地的是以 gomobile AAR 形式打包的 libv2ray（来自姊妹仓库 AndroidLibXrayLite，见 `.gitmodules` 与 CI 中 `Download libv2ray` 步骤），v2rayNG 负责三件事：其一，管理 Android 生命周期（VpnService 前台服务、开机自启、快捷磁贴、Always-on VPN）；其二，把用户导入的节点/订阅/路由规则翻译成 Xray 的 JSON 配置（`core/CoreConfigManager.kt`，约 1300 行）；其三，把 VPN TUN 设备的 fd 交给内核或 tun2socks 组件，完成流量从系统到内核的搬运。

节点配置入口非常丰富：vmess://、vless://、trojan://、ss://、hysteria2://（含 hy2://）、socks://、wireguard 等分享链接解析（`fmt/` 包 10 个解析器，`handler/AngConfigManager.kt` 的 `configFmtParsers` 映射表），扫码导入（CameraX + ZXing）、剪贴板导入、订阅 URL 批量更新（`service/SubscriptionUpdateService.kt`），还支持 `v2rayng://install-config` 深链（`AndroidManifest.xml`）。

多维度能力包括：分应用代理（允许/绕过名单，`CoreVpnService.configurePerAppProxy`）、按路由规则集分流（assets 内置黑白名单/全球/伊朗/俄罗斯等预设 `custom_routing_*`）、本地 DNS/FakeDNS 配置、流量统计与测速（`CoreServiceManager.queryAllOutboundTrafficStats`）、WebDAV 备份（`handler/WebDavManager.kt`）、Root 模式 iptables 全局接管（`root/` 包）。

## 3. 技术栈与依赖

- **语言/运行时**：Kotlin 2.4.21，JVM target 17，开启 coreLibraryDesugaring（`V2rayNG/gradle/libs.versions.toml`、`app/build.gradle.kts`）
- **UI**：Jetpack Compose（BOM 2026.09.00）+ Material3 逐步替换传统 View；Coil、reorderable（列表拖拽）
- **存储**：腾讯 MMKV（`mmkv-static`，`handler/MmkvManager.kt`，66 个函数的统一存储门面）+ Gson
- **网络**：OkHttp 5（订阅/更新下载）
- **并发**：kotlinx-coroutines；WorkManager（含 `work-multiprocess`，配合多进程架构）
- **扫码**：CameraX 1.6 + ZXing core 3.5
- **内核（外部依赖，不在此仓库）**：libv2ray.aar（AndroidLibXrayLite，gomobile 绑定 Xray-core）；hev-socks5-tunnel（NDK 源码编译，`compile-hevtun.sh`）
- **测试**：JUnit Jupiter（JUnit 6 BOM）+ Mockito/Mockito-Kotlin；androidTest 用 Espresso
- **构建**：AGP 9.4.1，Gradle 版本目录（libs.versions.toml），flavor 分 fdroid/playstore 双发行渠道，ABI 拆分 arm64-v8a/armeabi-v7a/x86_64/x86 + universal

## 4. 目录结构与代码组织

顶层（源码根）：

| 目录/文件 | 职责 |
|---|---|
| `V2rayNG/` | Android 应用工程（全部业务代码所在，约 3.16 万行 Kotlin） |
| `AndroidLibXrayLite/` | git submodule（tarball 中为空目录），生成 libv2ray.aar 的 Go 项目 |
| `hev-socks5-tunnel/` | git submodule（tarball 中为空目录），TUN→SOCKS5 转发的 C 库 |
| `compile-hevtun.sh` | 用 NDK 把 hev-socks5-tunnel 编成 `libhev-socks5-tunnel.so` 的脚本 |
| `.github/workflows/build.yml` | 唯一 CI：编译、签名、GPG 签名发布 |
| `fastlane/` | F-Droid 商店元数据 |
| `AGENTS.md`、`CR.md`、`GEMINI.md`、`CLAUDE.md` | AI 辅助开发规范（含隐私政策 CR.md） |

`V2rayNG/app/src/main/java/com/v2ray/ang/` 内按职责分层（包结构即架构）：

| 包 | 职责 |
|---|---|
| `core/` | 内核生命周期与配置生成：CoreServiceManager（总控）、CoreConfigManager（Xray JSON 组装）、CoreOutboundBuilder（各协议出站）、CoreNativeManager（libv2ray JNI 门面）、NetworkMonitor |
| `service/` | Android 服务层：三种运行模式服务 + 测速/订阅服务 + 快捷磁贴 |
| `handler/` | 业务处理器：MmkvManager（存储）、AngConfigManager（导入导出）、SettingsManager、SubscriptionUpdater、WebDavManager 等 |
| `fmt/` | 分享链接解析/生成，每协议一个文件 |
| `dto/`、`enums/`、`contracts/` | 数据模型、枚举、服务间接口（ServiceControl/Tun2SocksControl/IDialerService） |
| `ui/` | Compose 化界面：main/server/settings/subscription/routing/perappproxy/backup 等 15+ 子包，每协议一个编辑 Activity |
| `root/` | Root 模式：RootShell、RootProxyManager（iptables）、RootLanSharing |
| `receiver/` | 开机自启、桌面小部件、Tasker 联动 |

组织方式评述：单模块单包应用（app 一个 Gradle module），但包内分层清晰、依赖方向单向（ui → handler → core），`contracts/` 接口把服务层与内核管理层解耦。多进程职责切分在 Manifest 中完成（见第 5 节）。

## 5. 核心架构与关键实现

### 5.1 三种运行模式 + 双进程隔离（service/ + AndroidManifest.xml）

Manifest 把组件切到三个进程：主进程（UI）、`:daemon`（三种代理模式服务 + 小部件 + 快捷开关，`AndroidManifest.xml` L169-L258）、`:tasks`（测速 CoreTestService 与订阅更新 SubscriptionUpdateService）。内核崩溃只损失 `:daemon` 进程，UI 进程存活可感知并重启（`service/AGENTS.md` 的生命周期不变量："`:RunSoLibV2RayDaemon` 守护进程是内核是否运行的唯一权威"）。

三种模式共用同一个 `CoreServiceManager` 总控，以 `ServiceControl` 接口（`contracts/ServiceControl.kt`）注入：

- **VPN 模式** `service/CoreVpnService.kt`：标准 VpnService。`configureVpnService()` 用 Builder 配置 MTU、IPv4/IPv6 地址与路由（bypass LAN 时只加 `ROUTED_IP_LIST` 而非 0.0.0.0/0）、DNS、分应用 allow/disallow、Android 10+ 的 setMetered(false) 与可选 HTTP 代理（L199-L254）。`onStartCommand` 用 AtomicBoolean 防并发启动，并显式兼容系统 Always-on VPN 重启（intent 为 null 或 SERVICE_INTERFACE action 时先解锁，L77-L98）。
- **仅代理模式** `service/CoreProxyOnlyService.kt`：不建 VPN，只在本机开 SOCKS/HTTP 入站（`startCoreLoop(null)`，tunFd 为 0）。
- **Root 模式** `service/CoreRootService.kt`：注释明确"不用 VpnService，流量由 iptables 接管（见 RootProxyManager）"，先起内核再在子线程装 iptables 规则，拆除时先删规则再停内核。

### 5.2 内核进程管理：in-process 内核 + 双 tun2socks 通道（core/）

与很多"拉起独立 core 二进制进程"的客户端不同，v2rayNG 把 Xray 内核以 gomobile AAR 形式内嵌在 `:daemon` 进程内运行。`core/CoreNativeManager.kt` 是 JNI 门面：`Libv2ray.initCoreEnv(assetPath, deviceId)` 初始化、`Libv2ray.newCoreController(handler)` 拿到 `CoreController`，随后 `coreController.startLoop(configJson, tunFd)` 在 Go 运行时里启动内核（`core/CoreServiceManager.kt` L156）；停止走 `stopLoop()`，且在 IO 协程异步执行（L200-L208）。

TUN 流量有两条通道（`CoreVpnService.runTun2socks` L305-L318）：

1. 默认：把 `mInterface.fd` 直接传给内核，由 Xray 内部的 TUN 入站处理；
2. 可选 hev 模式：`service/TProxyService.kt` 加载 `libhev-socks5-tunnel.so`（`System.loadLibrary`，L38-L40），生成 YAML 配置写到 filesDir 后以 JNI `TProxyStartService(configPath, fd)` 在进程内启动转发，TUN→本地 SOCKS5（L46-L61），崩溃可经 `restartCallback` 自愈。

网络切换健壮性做得很细：`NetworkMonitor`（API 28+）监听上游网络变化，发生 handover 时触发 `reloadCore()`——复用既有 VPN fd 但**重新生成配置**，因为"出站服务器域名在旧网络上解析的地址在新网络上可能不可用"（`CoreServiceManager.kt` L255-L280 注释）。分应用归属查询则通过 `XrayProcessFinder` 调 `ConnectivityManager.getConnectionOwnerUid` 供内核做进程级路由（L416-L447）。

### 5.3 配置导入与生成管线（fmt/ → handler/AngConfigManager → core/CoreConfigManager）

导入侧：`AngConfigManager` 维护 scheme→parser 映射表，把 10 种链接格式统一解析为 `ProfileItem`；订阅走 OkHttp 拉取 + 按 `SubscriptionInfoParser` 解析流量信息，后台由 `SubscriptionUpdateService` 在 `:tasks` 进程执行。

生成侧是全项目最重的文件 `core/CoreConfigManager.kt`（1317 行）：`getV2rayConfig()` 把 ProfileItem 经 `CoreOutboundBuilder`（708 行，逐协议构造出站）转成 outbound，再按运行模式装配 inbounds（TUN/SOCKS/HTTP）、DNS（含按路由规则反推 DNS hosts 与国内外分流 DNS，L635-L1075）、FakeDNS、路由规则集，最终输出完整 Xray JSON。策略组/代理链（observatory、fallback tag 解析，L368-L455）也在此处理。

UI 组织：主界面已 Compose 化（`ui/main/MainScreen.kt` + `MainViewModel` + `MainRepository`），服务器编辑保留每协议一个 Activity（`ui/server/ServerVmessActivity.kt` 等 13 个），共享 `BaseServerActivity`；公共 Compose 组件集中在 `ui/compose/`。

## 6. 构建与运行

以下均来自仓库文件，未经实际执行验证：

```bash
# 前置：NDK 30（CI 安装 ndk;30.0.16248370，.github/workflows/build.yml）
# 1) 编译 hev-socks5-tunnel（需要子模块 + $NDK_HOME）
bash compile-hevtun.sh            # 产物拷到 V2rayNG/app/libs
# 2) 获取内核 AAR：从 AndroidLibXrayLite release 下载 libv2ray.aar 放入 V2rayNG/app/libs/
#    （CI 用 robinraju/release-downloader 按子模块 tag 下载）
# 3) 构建 APK
cd V2rayNG
./gradlew assembleRelease -Pandroid.injected.signing.store.file=... \
    -Pandroid.injected.signing.store.password=... -Pandroid.injected.signing.key.alias=... \
    -Pandroid.injected.signing.key.password=...
./gradlew licenseFdroidReleaseReport   # 生成开源许可报告（CI 中使用）
# 单元测试（JUnit Jupiter，app/build.gradle.kts testOptions）
./gradlew test
```

ABI 可用 `-PABI_FILTERS=arm64-v8a;armeabi-v7a` 裁剪（`app/build.gradle.kts` L19-L36）。fdroid flavor 产物名带 `-fdroid` 且 versionCode 另算偏移（L82-L124）。README 提示可直接在 Android Studio 打开 `V2rayNG/` 编译，但 aar 内置内核可能过时。

## 7. 测试与工程质量

- 单元测试 25 个文件（`V2rayNG/app/src/test/`），覆盖链接解析（fmt）、订阅信息解析、配置清理（OrphanProfileCleaner）、Root 模块、Compose ViewModel（SettingsViewModelTest 等）与工具类；JUnit Jupiter + Mockito，无 UI 自动化用例，androidTest 仅模板级（androidx.junit/espresso 依赖，未见自定义用例文件——基于 tarball 文件树，未确认）。
- 测试配置有细节：`app/build.gradle.kts` L153-L162 把 JUnit 发现告警升级为失败，并留注释解释原因。
- 无专门 test CI workflow；`.github/workflows/build.yml` 只构建发布，另有 `fastlane.yml`（部署商店元数据）。测试靠本地/PR 时执行（未确认 CI 是否跑 test task）。
- 质量基建：CodeFactor 徽章（README）、gradle-license-plugin 生成许可报告、proguard-rules.pro 存在但 release `isMinifyEnabled = false`（`app/build.gradle.kts` L41-L49，即发布包未开混淆/收缩——包体换稳定性/可诊断性的取舍）。
- 文档化程度高且少见地好：根 `AGENTS.md`（12KB）+ `service/AGENTS.md`、`ui/AGENTS.md` 分域规范，定义了必须遵守的生命周期不变量（幂等启动、前台先行、拆解顺序、多进程状态权威等），`copilot-instructions.md` 也在库中。

## 8. 社区与维护状态

- 活跃度极高：gh api 最近 5 条 commit 全部在 2026-10-25 至 10-28 之间，涵盖功能修复、依赖升级（OkHttp 5.5.1）、翻译更新（阿塞拜疆语）与子模块升级（hev-socks5-tunnel 2.19.3）。
- Release 节奏稳定：最新 2.3.14 于 2026-10-28 发布，含 21 个资产（多 ABI APK + GPG 签名 `.sig` + 公钥，CI sign-and-release job）；README 提供指纹供用户校验，防镜像/CDN 劫持。
- 贡献者 100+（gh api contributors per_page=100 anon=true 返回满 100 条）；issues 仅 53 个 open（维护积极）；提交信息规范（conventional commits 风格 + PR 编号），翻译走社区 PR。
- 渠道：Telegram 群/频道（README）、F-Droid 与 Play 双渠道 flavor、GPG 签名发布，生态配套成熟（v2rayN 桌面版、AndroidLibXrayLite）。

## 9. 亮点与风险

**亮点**

1. 多进程 + in-process 内核的架构组合：内核以 AAR 嵌入 `:daemon` 进程，避免独立进程 IPC 复杂度，同时与 UI 进程隔离，测速/订阅再拆到 `:tasks` 进程（`AndroidManifest.xml`）。
2. 网络健壮性细节扎实：Always-on VPN 重启路径显式兼容（`CoreVpnService.onStartCommand` L80-L84）、网络 handover 时保留 fd 但重建配置的 `reloadCore()`、启动锁防并发（AtomicBoolean + 双路径解锁）。
3. 协议生态覆盖全：10 种分享链接解析器 + v2rayN 批量格式 + Wireguard conf 文件导入（`fmt/`），策略组、代理链、分应用、FakeDNS、IPv6、Root iptables 模式一应俱全。
4. AI 协作规范体系罕见地完整：分层 AGENTS.md 把"守护进程唯一权威""幂等启动""前台先行"等硬约束写成机器可执行规则，等于一份高质量架构文档。
5. 发布可信链完整：GPG 签名 + 公钥指纹公示 + license 报告，对防劫持敏感的用户场景针对性很强。

**风险/局限**

1. 发布包未启用 R8（`isMinifyEnabled = false`），APK 体积与静态可审计性之间选择了后者，也说明依赖 tree-shaking 的优化缺失。
2. 核心能力强依赖两个仓库外构件（libv2ray.aar 二进制 release + hev 子模块），版本漂移与供应链信任依赖作者自律；tarball 拿不到子模块，源码可复现构建需额外步骤。
3. 单 Gradle 模块单包内 3.16 万行，`CoreConfigManager` 1317 行、`DialerNativeService` 775 行等巨型 object 持续膨胀，核心逻辑无接口抽象（object 单例 + SoftReference 回调），测试只能覆盖纯函数部分。
4. 隐私/合规属性决定其不可在部分应用商店分发（Play flavor 的存在说明曾受审核约束），用户资产（节点凭据）以明文存于 MMKV（`MmkvManager.kt` 未见加密逻辑——基于抽查，未确认无其他保护层）。

## 10. 总结评价

v2rayNG 是"Android 代理客户端"这一品类事实上的参照实现：它把一个 Go 写的网络内核干净地嵌入 Android 生命周期（多进程隔离 + 接口解耦 + 健壮的重启/换网自愈），同时维持了极高的迭代速度与发布质量（GPG 签名、双渠道、多 ABI）。代码不算"教科书式"——巨型单例、单模块、无 R8——但分层方向（ui→handler→core）和多进程职责划分非常清楚，且用 AGENTS.md 体系把架构不变量固化了下来。适合三类人：想研究 Android VpnService/多进程/NDK 混合工程实践的开发者（读 `service/` + `core/` 收益最大）；想给 Xray 生态做客户端或移植壳的作者（`fmt/` + `AngConfigManager` 的导入管线可直接参考）；以及需要可自校验安装包的重度用户。不提供使用与部署指导，请参考项目官方 wiki。
