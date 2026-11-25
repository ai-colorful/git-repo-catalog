# bannedbook/fanqiang 项目分析

> 星数 55771（快照 2026-10-24）｜主语言 Kotlin｜档位 50k-100k｜分析日期 2026-11-24

## 1. 项目速览

`bannedbook/fanqiang` 是一个"翻墙 / 科学上网"主题的**内容型合集仓库**：主体是按平台分类的中文翻墙教程 Markdown，附带一个自研的安卓 RSS 阅读器 **FQNews**（`fqnews2/`，Kotlin + Go，内嵌 sing-box 代理内核）和一组自建服务器脚本。GitHub 标注的 Kotlin 主语言完全来自 `fqnews2/` 与旧版 `fqnews/` 两个安卓工程。

| 项目 | 事实 | 出处 |
|---|---|---|
| 名称 | bannedbook/fanqiang（README 标题"翻墙-科学上网、翻墙工具、翻墙教程项目库"） | `README.md` |
| 星数 | 55771（快照 2026-10-24） | 任务参数 |
| 主语言 | Kotlin（约 49k 行 `.kt` + 10k 行 `.java`，全部在 `fqnews2/app`） | `fqnews2/app/src/main` |
| 许可证 | 仓库根目录**无** LICENSE；仅 `fqnews2/LICENSE`、`fqnews/LICENSE` 为 GPLv3；README 版权声明为"请随意分发，勿做商业使用" | `fqnews2/LICENSE`、`README.md` |
| 规模 | 1887 个文件，约 98MB；92 个 Markdown；309+63 张 PNG | 解压后 `find` 统计 |
| 最近活跃 | tarball 所有文件 mtime 统一为打包时间，无法据此推断；`ChromeGo/README.md` 记录"2026年6月30日更新公告"，`gitupdate.bat` 打 tag `FQNews-v1.3.8` 与 `app/build.gradle.kts` 的 `versionName = "1.3.8"` 一致 | `ChromeGo/README.md`、`gitupdate.bat` |
| CI | `.github/` 下只有两个 issue 模板，**没有** workflows | `.github/ISSUE_TEMPLATE/` |

## 2. 项目是什么，解决什么问题

仓库面向中国大陆网民，目标是"降低翻墙门槛"：用户按自己使用的平台（Windows / macOS / Linux / Android / iOS / 路由器 / 游戏机）进入对应目录，照着图文教程一步步完成客户端安装、节点导入或自建服务器。README 首页就是一个总索引，列出 17 个入口链接，其中大部分指向 GitHub Wiki 而非仓库内文件（`README.md` 第 3-19 行）。

内容分三层：

1. **平台客户端教程**：`windows/`（V2rayN、Clash、SSTap、SSR、TorBrowser）、`macos/`（ClashX、V2rayU、V2rayX、Surge）、`android/`（V2RayNG、Clash、BifrostV、Surfboard 等）、`ios/`（Shadowrocket、Quantumult X、Surge、Potatso）、`router/`（梅林、OpenWRT）、`linux/`（mihomo 命令行安装）、`game/`（PS4/PS5、Switch、Apple TV 通过旁路由翻墙）。
2. **自建服务器教程与脚本**：`v2ss/` 下 20 篇 Markdown 讲 V2Ray / Shadowsocks / Brook 的 TLS+WebSocket+Nginx+CDN 配置、VPS 购买、BBR 加速；`v2ss/server-cfg/` 存放可直接在 VPS 上执行的 `ss-init.sh`、`v2ray-init.sh`、nginx/systemd/sysctl 配置。
3. **一键翻墙包与 APP**：`ChromeGo/`、`EdgeGo/`、`ChromeGoMac/`、`FirefoxFqLinux/`、`tor-browser-portable/` 各只含一个 README 或启动脚本，实际二进制放在 Releases/Wiki；`fqnews2/` 是完整源码的安卓 APP，`fqnews/` 是已标注"暂时失效"的旧版。

仓库还承担**分发与变现**功能：几乎所有教程页顶部都嵌入了同一张"V2free 机场"推荐横幅和带 `code=cd79` 的注册引荐链接（`V2ray机场.md`、`免费ss账号.md`、`安卓翻墙软件.md`），并在 `fqnews2/app/build.gradle.kts` 中硬编码了 AdMob 广告位 ID。

## 3. 技术栈与依赖

仓库本身没有统一工程清单；这里只列 `fqnews2` 这个唯一可构建的工程。

| 层 | 技术 | 版本 | 出处 |
|---|---|---|---|
| 语言 | Kotlin / Java / Go | Kotlin 1.9.23；Go 1.20 | `fqnews2/settings.gradle.kts`、`fqnews2/libcore/go.mod` |
| 构建 | Gradle (KTS) + AGP + KSP + ktlint-gradle | Gradle 8.4、AGP 8.3.0 | `fqnews2/gradle/wrapper/gradle-wrapper.properties`、`build.gradle.kts` |
| UI | Jetpack Compose (BOM 2024.04.00)、Material3、Navigation Compose、Paging 3、Accompanist | — | `settings.gradle.kts` libs 版本目录 |
| 架构 | Room 2.5.2、WorkManager 2.9.1、Lifecycle/ViewModel、Kodein DI 7.5.0、kotlinx-coroutines 1.7.3 | — | 同上 |
| 网络/解析 | OkHttp 4.10、Okio 3.2、Conscrypt、Retrofit 2.9 + Moshi、jsoup 1.7.3、TagSoup、readability4j、Coil 2.4 | — | 同上 |
| 代理内核 | **libcore**：基于 `sagernet/sing-box` + `matsuridayo/libneko` + `sing-box-extra`，用 `gomobile bind` 编译为 `libcore.aar`，编译 tags 含 `with_quic,with_wireguard,with_utls,with_clash_api,with_ech` | Go 1.20 | `fqnews2/libcore/build.sh`、`go.mod` |
| 代理管理层 | 从 SagerNet/NekoBoxForAndroid 移植的 Kotlin 代码（`io.nekohasekai.sagernet.*`，72 个文件） | — | `fqnews2/app/src/main/java/io/nekohasekai/` |
| 发布 | fastlane（Google Play 上传）、Roomigrant、kryo 5.2.1、Gson、SnakeYAML、ini4j | — | `fqnews2/fastlane/Fastfile`、`app/build.gradle.kts` 第 269-282 行 |
| 测试 | JUnit 4、MockK、Mockito、MockWebServer、Espresso、Compose UI Test、coroutines-test | — | `settings.gradle.kts` test bundle |

注意 `libcore/go.mod` 第 96-125 行用 `replace` 把 `libneko`、`sing-box-extra`、`sing-box`、`sing-quic` 指向 `../../xxx` 的**同级本地目录**，这些目录不在本仓库中，因此 libcore 无法从本仓库单独构建；`app/build.gradle.kts` 第 269 行以 `files("../libcore/libcore.aar")` 方式依赖预编译产物，而该 `.aar` 被 `libcore/.gitignore`（`*.[a|j]ar`）排除，仓库中不存在。

## 4. 目录结构与代码组织

| 目录 | 文件数 / 体积 | 职责 |
|---|---|---|
| `README.md` + 12 个顶层 `.md` | 约 12 篇 | 总索引、免费账号页、ChatGPT 注册、链式代理等专题 |
| `fqnews2/` | 990 文件 / 48MB | FQNews 新版安卓工程（Kotlin+Compose+Go） |
| `fqnews/` | 378 文件 / 23MB | FQNews 旧版工程（已失效，含 `v2ray-lib-FQNews`、`plugin/`、`tools/`） |
| `v2ss/` | 329 文件 / 20MB | 自建 V2Ray/SS/Brook 服务器教程 + `server-cfg/` 脚本与配置 |
| `deprecated/` | 86 文件 / 4MB | GoAgent/GAE 时代遗留（`go.py` 为 base64 混淆内容）、Resilio Sync、旧 PAC |
| `game/` | 31 文件 | 游戏机/Apple TV/Oculus 翻墙教程 |
| `android/` `ios/` `macos/` `windows/` `router/` `linux/` | 1-10 篇/目录 | 各平台客户端教程 |
| `ChromeGo/` `EdgeGo/` `ChromeGoMac/` `FirefoxFqLinux/` `tor-browser-portable/` | 各 1-2 文件 | 一键翻墙包说明页与启动脚本 |
| `docs/` | 11 文件 | 内容为 base64 混淆的 `.py` 脚本与 `index.html`，用途未确认 |
| `apk/` | 2 张图 | 二维码图片 |
| `gitupdate.sh` / `gitupdate.bat` | — | 维护者用脚本：`git add -A && commit -m "update" && tag && push` |

辅助配置：`_config.yml` 仅一行 `theme: jekyll-theme-cayman`，说明仓库启用了 GitHub Pages 渲染 README；`.bookignore` 排除了 `tor-browser-portable/*`、`deprecated/*`、`.github/*`，是 GitBook 导出用的忽略表；`.gitattributes` 声明 `* text=auto` 且 `*.* binary`（即所有带扩展名的文件都不做换行转换）。

组织方式评述：这是一个**单仓库多产物**的松散合集，而非 monorepo。三个相互独立的部分（文档、安卓工程、服务器脚本）只是放在同一仓库下，没有统一的构建入口或 CI。`fqnews2` 内部则是标准的单模块 Android 工程（`rootProject.name = "feeder"`，`include(":app")`），包结构为 `com.nononsenseapps.feeder.*`（上游 Feeder 的 RSS 业务）+ `io.nekohasekai.sagernet.*`（代理管理）+ `app.*`（flavor 专属的 `NodeImporter`、`Constants`）。产品 flavor `fqnews` 的 sourceSet 根目录被改为 `app/src/fqnews`（`app/build.gradle.kts` 第 47-49 行）。

## 5. 核心架构与关键实现

### 5.1 FQNews 的"免 VPN"代理架构

FQNews README 声称"无需 VPN，只需导入一个翻墙节点"（`fqnews2/README.md`）。实现上，它**没有**注册 `VpnService`，`AndroidManifest.xml` 只声明了 `io.nekohasekai.sagernet.bg.ProxyService`（运行在 `:bg` 进程，`foregroundServiceType="dataSync"`）。也就是说 sing-box 以**本地 HTTP/SOCKS 混合端口**方式工作，不接管系统流量，只有 APP 自身的 OkHttp 请求走代理：

```kotlin
// fqnews2/app/src/main/java/com/nononsenseapps/jsonfeed/JsonFeedParser.kt 第 25-28 行
// 配置代理信息
val proxy = Proxy(Proxy.Type.HTTP, InetSocketAddress("127.0.0.1", DataStore.mixedPort))
val builder: OkHttpClient.Builder = OkHttpClient.Builder().proxy(proxy)
```

`FeederApplication.onCreate` 中调用 `Libcore.initCore(...)` 初始化 Go 内核（`jww/app/FeederApplication.kt` 第 239-248 行），`DefaultNetworkListener` 监听网络变化以更新 `underlyingNetwork`。`MainActivity.startPortMonitor()` 起一个死循环线程，每 3 秒 `Socket("127.0.0.1", DataStore.mixedPort)` 探测端口，不通就 `startService()` 拉起代理（`ui/MainActivity.kt` 第 144-174 行）。

### 5.2 节点导入与批量测速

`app/src/fqnews/java/app/NodeImporter.kt` 实现"剪贴板导入 → 批量 URL Test"：读取剪贴板文本，交给 SagerNet 移植来的 `RawUpdater.parseRaw()` 解析 `ss:// vmess:// vless:// trojan:// hysteria` 等 URI（`io/nekohasekai/sagernet/group/RawUpdater.kt` 第 210 行起），逐条 `ProfileManager.createProfile()` 入库，然后用 `newFixedThreadPoolContext(DataStore.connectionTestConcurrent)` 开协程池并发 `UrlTest().doTest(profile)`，测通的写回 `ping`，失败的直接 `deleteProxy()`。这是对 SagerNet 原生订阅流程的简化：没有分组/订阅概念，一切以剪贴板为入口。

内置栏目在 `app/src/fqnews/java/app/Constants.kt` 中硬编码两组数组（简体 27 条、正体 26 条），全部指向 `inoreader.com/stream/user/1005659457/tag/...` 的聚合 RSS；首次启动弹出"简体/正體"选择框切换。

### 5.3 Go 内核封装（libcore）

`fqnews2/libcore/` 是对 sing-box 的薄封装：`box.go` 定义 `BoxInstance`（嵌入 `boxbox.Box`，附带 selector、pauseManager、`v2api`）；`nb4a.go` 的 `InitCore` 设置 `RunMode_NekoBoxForAndroid`、日志写入 `neko.log`、读取 `ca.pem` 更新根证书；`build.sh` 用 `gomobile bind -androidapi 21` 带 tags 编译成 AAR 放到 `app/libs/`。该目录源自 MatsuriDayo/NekoBoxForAndroid 的 libcore，与上游基本一致。

### 5.4 教程内容的组织与"更新机制"

教程没有自动化更新流程。`gitupdate.sh` / `gitupdate.bat` 显示维护方式是手动 `git add -A` 一键提交并打 tag。各教程页的"免费节点"并不存放在仓库里，而是引导用户去 Wiki 页面或第三方 APP（`免费ss账号.md`），因此仓库内实际可用的节点信息为零，所谓"长期更新"依赖 Releases 附件和 Wiki，这两者在本次快照中均不可见。

## 6. 构建与运行

以下命令来自 `fqnews2/` 工程文件，**本次分析未执行**：

```bash
cd fqnews2
# 构建 libcore（需要 ../../libneko ../../sing-box 等同级目录与 Android NDK）
cd libcore && ./build.sh && cd ..
# 构建 APK（debug 使用仓库内 shareddebug.keystore，口令 android）
./gradlew :app:assembleFqnewsDebug
# lint + 单元测试 + 模拟器 instrumentation 测试（release.sh 中的校验命令）
./gradlew check pixel2api30DebugAndroidTest
# 发版（交互式脚本：生成 CHANGELOG、改版本号、打 tag）
./release.sh
# Google Play 上传（需要 SERVICEACCOUNTJSON / KEYSTORE 等环境变量）
./prepare_for_build.sh && fastlane android build_and_deploy
```

`app/build.gradle.kts`：`applicationId = "jww.feed.fqnews"`，`minSdk = 23`，`targetSdk/compileSdk = 34`，`versionCode = 292`，`versionName = "1.3.8"`。教程类内容无需构建；`v2ss/server-cfg/ss-init.sh` 等脚本是面向 Debian/Ubuntu VPS 的一键安装脚本，以 root 直接 `wget | sh` 方式运行。

## 7. 测试与工程质量

- **测试规模**：`app/src/test` 18 个 Kotlin 文件（FeedStore、Repository、HtmlUtils、OPML 解析等上游 Feeder 继承的单元测试），`app/src/androidTest` 66 个文件（Room DAO、导航、设置页、Compose UI 测试），使用 MockK/MockWebServer/Espresso。代理相关模块（`io.nekohasekai.*`、`app.NodeImporter`）**无任何测试**。
- **静态检查**：`ktlint-gradle 12.1.1` + `ktlint-compose` 插件已配置；`build.gradle.kts` 将 `allWarningsAsErrors = false`。
- **CI**：仓库内无 GitHub Actions；`ci/before`、`ci/run-if-release` 等脚本是为外部 CI（从 `prepare_for_build.sh` 的 `SERVICEACCOUNTJSON` 变量看应为 GitLab CI 或自建 runner）准备的，但对应配置文件不在仓库中，未确认。
- **代码质量观察**：`NodeImporter.kt` 保留了大段注释掉的旧实现 `urlTestOld()` 和 `Log.e("urlTest", "1"..."7")` 调试日志；`MainActivity.startPortMonitor()` 用 `while(true) + Thread.sleep(3000)` 轮询；`JsonFeedParser.kt` 第 20 行 `trustAllCerts: Boolean = true` 默认开启，`OkHttpBuilderExtensions.kt` 的实现是空 `checkServerTrusted` + 全放行 `HostnameVerifier`，即 APP 对所有 feed 源**关闭 TLS 证书校验**。
- **多语言**：`fastlane/metadata/android/` 有 35 个语言的商店文案；`CHANGELOG.md` 的 209 个版本条目（最新 2.6.30）全部是上游 Feeder 的提交记录，而非 FQNews 自己的变更日志。

## 8. 社区与维护状态

- **维护模式**：单维护者（bannedbook）项目。`gitupdate.bat` 的提交信息固定为 `"update"`，贡献者结构无法从快照判断，标注**未确认**。
- **发布节奏**：仓库内可推断的最新时间点是 `ChromeGo/README.md` 的"2026年6月30日更新公告"，FQNews 当前版本 1.3.8（versionCode 292），旧版 `fqnews/update.json` 停留在 version 235。GitHub Releases 页面内容不在快照内，Release 总数与频率**未确认**。
- **用户反馈渠道**：`.github/ISSUE_TEMPLATE/` 两个模板内容相同，仅要求反馈者"明确说明是哪一个软件或服务"；README 给出 Telegram 群 `https://t.me/fqchat` 与邮箱 `freeman105@gmail.com`（注明不解答问题）。
- **姊妹项目**：README 与代码注释中引用了同作者的 `bannedbook/v2ray.vpn`（安卓 VPN）、`bannedbook/shadowsocks_install`、`bannedbook/blockporn`；FQNews 基于开源项目 `spacecowboy/Feeder`（RSS）与 SagerNet/NekoBoxForAndroid（代理）二次开发。

## 9. 亮点与风险

**亮点**

1. **覆盖面极广**：从 Windows/macOS/Linux 桌面到 iOS/Android、路由器固件、游戏主机和 Apple TV，每个平台都给出多种客户端的图文教程，内容型合计约 6000 行 Markdown、438 个去重外部链接。
2. **FQNews 的"代理不接管系统"思路**：对只想读新闻的用户，`ProxyService` + 本地混合端口比全局 VPN 更轻量，不与其他 VPN APP 冲突，也不需要常驻 VPN 权限。
3. **节点批量测速**：剪贴板导入后并发 URL Test 并自动剔除失效节点，降低了对新手的使用门槛。
4. **自建服务器脚本完整**：`v2ss/server-cfg/` 提供 nginx、systemd、sysctl BBR、crontab 等一整套配置，教程与脚本配套。

**风险与局限**

1. **法律与安全属性**：项目核心目的是绕过国家级网络审查，在中国大陆使用具有法律风险；一键包为预编译二进制且建议用户"卸载国产杀毒软件"（`ChromeGo/README.md`），用户无法审计其内容。
2. **TLS 校验被关闭**：FQNews 默认 `trustAllCerts`，所有 RSS 请求可被中间人篡改或窃听。
3. **商业引荐与广告**：教程中大量机场引荐链接（`code=cd79`），APP 内置 AdMob 广告 ID，内容中立性受影响。
4. **不可独立构建**：libcore 依赖四个不在仓库内的同级目录，AAR 被 gitignore；无 CI、代理模块无测试；旧版 `fqnews/` 与 `deprecated/` 大量失效内容未清理。

## 10. 总结评价

`bannedbook/fanqiang` 本质是一个**面向中文用户的翻墙知识库 + 分发入口**，GitHub 语言统计为 Kotlin 只是因为附带了 FQNews 安卓工程。它的高星来自题材本身的搜索需求、十余年的持续积累（`deprecated/gae/` 可追溯到 GoAgent 时代）以及 README 中"点右上角 star 按赞"的直接引导，而非代码质量。FQNews 本身是一个有趣的工程样本：把 Feeder（Compose RSS 阅读器）与 NekoBoxForAndroid 的 SagerNet 层拼在一起，以本地代理端口而非 VPN 的方式让 APP "自带翻墙"，适合研究"Android 应用内嵌 sing-box"或"RSS 阅读器架构"的读者阅读 `fqnews2/app/src/main/java/com/nononsenseapps/` 与 `fqnews2/libcore/`。对一般开发者而言，本仓库的教程内容有时效性（节点、客户端版本随时间失效），代码部分不宜作为生产参考。
