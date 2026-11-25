# TeamNewPipe/NewPipe 项目分析

> 星数 37500（快照 2026-10-24）｜主语言 Java｜档位 20k-50k｜分析日期 2026-11-14
> 分析基于 dev 分支 HEAD tarball（2026-11-13 快照）；gh api 实测当前 40659 星、GPL-3.0、默认分支 dev。

## 1. 项目速览

一句话定位：安卓端开源流媒体前端，不登录任何账号即可观看/收听 YouTube、PeerTube、SoundCloud、Bandcamp、media.ccc.de 等多个平台的内容，站点解析核心在独立仓 NewPipeExtractor 中。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | TeamNewPipe/NewPipe | gh api |
| 星数 | 37500（快照）；2026-11-14 实测 40659 | 任务参数 / gh api |
| 语言 | Java（存量）+ Kotlin（新代码为主，共 349 个 .kt / 293 个 .java） | 源码统计 |
| 许可证 | GPL-3.0-or-later | `LICENSE`、SPDX 文件头 |
| 最新版本 | v0.29.3（2026-11-10），versionCode 1017 | gh api releases、`buildSrc/src/main/kotlin/ProjectConfig.kt` |
| 最近活跃 | 最近 push 2026-11-13（前一天） | gh api |
| 贡献者 | 约 1056 人 | gh api contributors 分页 |
| 法律主体 | NewPipe e.V.（德国注册非营利协会） | 文件版权头、`doc/README.md` |

## 2. 项目是什么，解决什么问题

NewPipe 是"轻量流媒体前端"：它不内嵌 WebView，也不用平台官方 SDK，而是直接解析目标网站的网页或内部接口拿取视频、评论、频道数据，播放用本地播放器内核完成。这样做换来了三个能力：无需账号即可订阅频道（本地订阅存数据库）、不装 Google Play Services 也能运行（可刷进无 GMS 的自定义 ROM）、后台播放与悬浮窗播放成为一等公民（`doc/README.md` Description 节）。

核心设计是"前端与抽取器分离"：站点解析逻辑全部在独立仓库 TeamNewPipe/NewPipeExtractor 中，本仓通过 JitPack 依赖（`com.github.TeamNewPipe:NewPipeExtractor:v0.27.1`，见 `gradle/libs.versions.toml` 第 78、160 行；本地联调时可在 `settings.gradle.kts` 中打开 `includeBuild("../NewPipeExtractor")` 复合构建）。服务注册表 `ServiceList` 也定义在 Extractor 侧，主仓 11 处直接 import 使用（`app/src/main/java/.../fragments/list/kiosk/KioskFragment.java` 等）。这一解耦让"适配新站点"与"改进 App"两条线可以独立发版。

由于不调用平台官方 API（YouTube 的官方 API 对此用途受限），项目长期处于与上游平台变化的对抗性维护状态：官方接口不可用就解析网页（`doc/README.md`："parses the website or uses an internal API instead"）；平台升级校验后就需要同步更新请求头、Cookie 与流 URL 获取逻辑。仓库内的对应实现包括 YouTube 受限模式 Cookie 与 reCAPTCHA Cookie 管理（`app/src/main/java/org/schabi/newpipe/DownloaderImpl.java`）、专门的 `YoutubeHttpDataSource`（`app/src/main/java/org/schabi/newpipe/player/datasource/`）以及 YouTube DASH 直播清单解析器（`player/helper/YoutubeDashLiveManifestParser.java`）。近两次发版间隔仅 3 天（v0.29.2 于 2026-11-07、v0.29.3 于 2026-11-10，gh api releases），体现了这类快速响应节奏。

分发上官方渠道是 F-Droid 与 GitHub Releases，刻意不上架 Google Play——README 明确警告"将 NewPipe 或其分支放入 Google Play 违反其条款"（`doc/README.md`），同时 NewPipe 是注册商标、有商标政策（`doc/README.md`）。目前项目正处在一次大规模重写的中期：从单模块 Java 安卓应用迁往 Kotlin Multiplatform（见第 5 章），贡献指南已声明"暂不接受新功能 PR，仅收 bugfix"（`.github/CONTRIBUTING.md` 首行）。

## 3. 技术栈与依赖

语言与框架（出处：`gradle/libs.versions.toml`、各模块 `build.gradle.kts`）：

- Kotlin 2.4.21 + AGP 9.5.1 + JDK 21 toolchain（`app/build.gradle.kts` jvmToolchain(21)）
- Android：minSdk 23 / targetSdk 35 / compileSdk 37（`buildSrc/src/main/kotlin/ProjectConfig.kt`）
- 播放内核：ExoPlayer 2.19.1（旧包 `com.google.android.exoplayer2`，非 Media3；`libs.versions.toml` 第 27 行及注释"media 1.7.1 # Keep at 1.7.1 for compatibility with exoplayer"）
- 新 UI：Compose Multiplatform 1.12.1（shared 模块，`shared/build.gradle.kts` 声明 android/iosArm64/iosSimulatorArm64/jvm 四目标）
- DI：Koin 4.2.2（含 koin-annotations 与 KSP 编译器插件，仅 shared 新代码使用）
- 数据库：Room 3.0.3（`androidx.room3:room3-*`，KMP 版 Room + bundled sqlite）
- 旧代码异步栈：RxJava 3.1.12 / RxAndroid / RxBinding，配合 kotlinx-coroutines-rx3 桥接
- 网络：OkHttp 5.5.1、Jsoup 1.24.1（HTML 解析）、Coil 3.6.3（图片）
- 后台任务：WorkManager 2.11.2；序列化：kotlinx-serialization 1.12.0
- 站点抽取：NewPipeExtractor v0.27.1（独立仓，JitPack 引入）

## 4. 目录结构与代码组织

顶层目录（HEAD 快照）：

| 目录 | 职责 |
|---|---|
| `app/` | Android 主应用（约 11.6MB 源码资源），含播放器、Fragment UI、数据库、下载 |
| `shared/` | KMP 共享模块（约 10.8k 行 Kotlin），Compose Multiplatform 新 UI + 平台抽象 |
| `desktopApp/` | Compose Desktop 入口（仅 `net/newpipe/app/Main.kt` 一个文件） |
| `iosApp/` | Xcode 工程壳（iOS 目标在 shared 中声明，UI 尚未迁移） |
| `buildSrc/` | `ProjectConfig.kt` 集中管理 SDK 版本/版本号 |
| `fastlane/` | F-Droid 元数据与应用截图（13.8MB，含各版本 changelog 文本） |
| `doc/` | 多语言 README（20 种语言）与 gradle 说明 |
| `checkstyle/`、`config/` | 静态检查配置 |
| `.github/` | CI 工作流、贡献/issue/PR 模板、FUNDING |

组织方式：单仓库三 Gradle 模块（`:app`、`:shared`、`:desktopApp`），Extractor 以外部依赖形态存在而非子模块。`app/src/main/java` 内是经典安卓分层：`org/schabi/newpipe/{player,fragments,local,database,settings,download,streams,util,views,info_list,error,ktx}` + 自带的下载管理器包 `us/shandian/giga/`；shared 内是新架构 `net/newpipe/app/{screen,composable,viewmodel,di,navigation,platform,preferences,theme,model}`。新旧两套 UI 并存，是典型"绞杀者模式"渐进迁移形态。

## 5. 核心架构与关键实现

### 5.1 播放器体系：Player + 可插拔 PlayerUi

核心是一个 2492 行的 `Player.java`（`app/src/main/java/org/schabi/newpipe/player/Player.java`），承载 ExoPlayer 实例、播放状态机与全部播放逻辑。UI 则被拆成可插拔组件，官方注释直接给出了设计意图（`player/ui/PlayerUi.java`）：

```java
/**
 * A player UI is a component that can seamlessly connect and disconnect from the {@link Player}
 * and provide a user interface of some sort. Try to extend this class instead of adding
 * more code to {@link Player}!
 */
public abstract class PlayerUi {
```

三个实现对应三种形态（`player/ui/`）：`MainPlayerUi`（1002 行，页面内嵌播放）、`PopupPlayerUi`（602 行，悬浮窗/画中画）、`BackgroundPlayerUi`（后台音频，29 行——只补通知差异，逻辑基本复用）。同一 Player 实例在三种 UI 间热切换，避免重建播放内核。

宿主是 `PlayerService`（`player/PlayerService.java`，extends `MediaBrowserServiceCompat`），前台服务承载后台播放，同时通过 MediaBrowser 协议向 Android Auto / Leanback（TV）暴露媒体库；Manifest 中 `MainActivity` 声明了 `LEANBACK_LAUNCHER`，并注册 `MediaButtonReceiver` 处理耳机/蓝牙媒体键（`app/src/main/AndroidManifest.xml`）。配套子系统还包括：`playqueue/` 播放队列抽象（7 个队列实现：单曲/频道/播放列表等）、`mediasource/ManagedMediaSourcePlaylist` 流媒体源管理、`resolver/` 音视频流选择（按清晰度/格式解析），以及手势层 `gesture/`（双击快进、捏合调 popup 大小等）。

### 5.2 单 Activity + Fragment 的 UI 组织

整个 App 由一个 `MainActivity`（`app/src/main/java/org/schabi/newpipe/MainActivity.java`，1057 行，`launchMode="singleTask"`）承载，页面切换全部走 `FragmentManager` 返回栈，封装在 `util/NavigationHelper` 中。Fragment 体系分层清晰（`fragments/`）：`MainFragment`（首页 tab 容器）、`detail/VideoDetailFragment`（2527 行，视频详情+播放承载页）、`list/BaseListInfoFragment` 之下挂 `channel/comments/kiosk/playlist/search/videos` 六类列表页——每种页面类型对应 Extractor 的一种 Info 结果。外部分享链接统一进入 `RouterActivity`（1078 行，`AndroidManifest.xml` 中注册多 intent-filter），解析 URL 归属哪个服务后路由到对应 Fragment 或直接进播放。

本地功能全部落库：`database/AppDatabase.kt`（Room 3）下有 `feed/history/playlist/stream/subscription` 五个域的 DAO 与实体（`app/src/main/java/org/schabi/newpipe/database/`），订阅不依赖任何平台账号，feed 刷新走独立后台服务 `local/feed/service/FeedLoadService`。下载由内置的 `us/shandian/giga/` 下载管理器（服务 `DownloadManagerService` + 后处理管道 `postprocessing/`）承接。

### 5.3 KMP 重写：shared 模块与绞杀式迁移

本次快照最重要的结构变化：`shared/` 是一个 Kotlin Multiplatform 模块，用 Compose Multiplatform 写新 UI、Koin 做 DI、Room 3 存数据，目标平台 android/ios/jvm 四个（`shared/build.gradle.kts` 第 81-90 行）。目前迁移完成的是"关于"页与设置页的一批界面（`shared/src/commonMain/kotlin/net/newpipe/app/screen/about/`、`screen/settings/` 共 20 余个 Screen/Composable 文件）；平台差异通过接口抽象注入，如 `platform/BackupRestoreActions.kt`（common）+ `androidMain/.../AndroidBackupRestoreActions.kt`（Android 实现）。

Android 侧的接入点是设置页里的 Compose 宿主：`app/src/main/java/org/schabi/newpipe/settings/ComposeBackupRestoreHost.kt`、`ComposeContentHost.kt` 等——旧设置页逐块替换为 shared 中的 Composable。桌面端 `desktopApp/src/main/kotlin/net/newpipe/app/Main.kt` 仅 12 行有效代码，用 `Window { App(...) }` 复用 shared 的根组件 `App.kt`（含 Koin 初始化与 NavDisplay 导航）。注意 shared 目前 0 处引用 Extractor（`grep` 统计），说明核心业务流的迁移尚未开始，先把外围设置/关于页搬过去。贡献指南"仅收 bugfix"的禁令正对应这一重写计划（`.github/CONTRIBUTING.md`）。

### 5.4 与上游平台的技术应对（客观陈述）

针对平台侧的校验机制，仓库内可见三层应对：其一，请求层 Cookie 管理——YouTube 受限模式 Cookie 与 reCAPTCHA Cookie 由 `DownloaderImpl` 按域名注入（`app/src/main/java/org/schabi/newpipe/DownloaderImpl.java` 第 78-98 行）；其二，播放数据源自定义——`YoutubeHttpDataSource` 继承 ExoPlayer `BaseDataSource` 自行控制请求头（同目录 `datasource/`），配合 `helper/PlayerDataSource.java` 组装 DASH/HLS/SS 三种加载器；其三，直播兼容——`helper/YoutubeDashLiveManifestParser.java` 专门处理 YouTube 的直播清单格式。这些属于纯技术适配，抽取器侧的具体解析逻辑在独立仓 NewPipeExtractor 中。

## 6. 构建与运行

```bash
# 构建（官方 CI 同款命令，.github/workflows/ci.yml 第 58 行）
./gradlew assembleContinuous lintContinuous testDebugUnitTest -DskipFormatKtlint

# 单测 / 设备测试 / Sonar（ci.yml 的三个 job）
./gradlew testDebugUnitTest
./gradlew connectedCheck          # 需模拟器，CI 中跑 API 矩阵
./gradlew build sonar

# 本地联调 Extractor：编辑 settings.gradle.kts，取消注释 includeBuild("../NewPipeExtractor")

# 调试构建自动加包名后缀：默认分支 .debug，其他分支 .debug.<分支名>
# （app/build.gradle.kts buildTypes.debug 逻辑）
```

自定义参数（`doc/gradle.md`）：`-DpackageSuffix=` 控制发布包名后缀、`-DversionCodeOverride=`/`-DversionNameSuffix=` 覆盖版本号、`-DskipFormatKtlint` 跳过 ktlint 格式化。debug 构建按 git 分支名自动后缀包名，实现多分支并行安装。

## 7. 测试与工程质量

- 单元测试：`app/src/test` 22 个文件（Java/Kotlin 混合），覆盖播放队列（`PlayQueueTest.java`、`PlayQueueItemTest.java`）、工具类（`util/ListHelperTest.java`、`util/urlfinder/UrlFinderTest.kt`）、订阅导入导出（`settings/ImportAllCombinationsTest.kt`）、数据库播放列表（`database/playlist/PlaylistLocalItemTest.kt`）等纯逻辑单元；框架为 JUnit + 少量 Robolectric 场景。
- 设备测试：`app/src/androidTest` 9 个文件。
- shared 新代码测试密度明显更高：约 20 个 Compose UI 测试（`shared/src/commonMain/kotlin/net/newpipe/app/screen/settings/*SettingsScreenTest.kt` 逐屏覆盖 + `composable/` 组件测试），新写的代码基本做到了屏级测试覆盖。
- CI（`.github/workflows/ci.yml`）：三 job——JVM 构建+lint+单测（含 ktlint 格式检查）、模拟器矩阵 connectedCheck（多 API level）、Sonar 质量扫描；另有 `checkstyle/checkstyle.xml` 配套。发布走 `build-release-apk.yml`，配套 `backport-pr.yml`、`pr-labeler.yml` 等 8 个工作流。
- 代码质量观察：存量巨型类多（Player.java 2492 行、VideoDetailFragment 2527 行、RouterActivity 1078 行、SearchFragment 1190 行），是重写计划的主因；新代码（shared）结构明显更现代（MVVM + Compose + DI）。

## 8. 社区与维护状态

- 发版节奏：2026-04 至 2026-11 间 8 个 release（v0.28.5 → v0.29.3），约月度一次；v0.29.2→v0.29.3 仅隔 3 天，属快速修复（gh api releases）。
- 活跃度：最近 push 2026-11-13（Gradle 9.9.1 升级）；最近提交内容为"add backup settings screen and test"（2026-11-12），KMP 迁移持续推进中（gh api commits）。
- 贡献者约 1056 人（含匿名），开源重度协作项目；翻译走 Weblate，README 有 20 种语言（`doc/` 目录）。
- 治理：已注册德国非营利协会 NewPipe e.V.（文件版权头 "SPDX-FileCopyrightText: 2025/2026 NewPipe e.V."），Liberapay 接受捐赠（`doc/README.md`）；贡献指南包含明确的 AI 使用政策（禁止 AI 生成新特性/模板类贡献，允许用于 bug 定位与文档），以及"重写期不接受新功能 PR"的声明（`.github/CONTRIBUTING.md`）。
- 分发：官方 F-Droid 自有仓库（推荐）、GitHub Releases（附签名指纹校验）、F-Droid 官方源（签名不同、更新最慢，README 明示三种通道差异）；明确不上架 Google Play（`doc/README.md` 警告块）。另有 NewPipe-nightly 仓库发每日构建（README 徽章）。

## 9. 亮点与风险

亮点：

1. 抽取器与前端彻底解耦：站点解析独立成 NewPipeExtractor 仓，主仓以 JitPack 版本号消费（`gradle/libs.versions.toml`），新增站点不动主仓架构；主仓 134 个文件引用 `org.schabi.newpipe.extractor` 但全部面向统一接口编程。
2. 播放器 PlayerUi 可插拔设计：一个 Player 实例 + 三种 UI 形态（内嵌/悬浮/后台）热切换（`player/ui/PlayerUi.java`），后台播放/画中画不需重建内核，注释明文约束"别往 Player 里继续堆代码"。
3. 大迁移路径清晰：KMP + Compose Multiplatform + Koin + Room 3 的目标栈已定型，绞杀式迁移从设置页落地，平台差异走接口抽象（`shared/src/.../platform/`），iOS/桌面壳就位，且新代码测试覆盖先行。
4. 无账号化产品设计彻底：本地订阅、本地播放列表、watch history 全部落本地 Room 库（`database/`），隐私默认不收集（`doc/README.md` Privacy 节）。
5. 工程治理成熟：CI 三层（JVM/模拟器矩阵/Sonar）、签名指纹公开校验、20 语言翻译、明确的 AI 政策与商标政策。

风险/局限：

1. 对抗性维护是长期税：上游平台反爬与校验升级会随时打断可用性，历史上多次出现"核心功能整体失效-紧急发版"周期（如本次 v0.29.2→v0.29.3 三天连发）；该风险本质无法消除，只能靠响应速度缓解。
2. 存量代码债重：核心路径集中在数千行级巨类上（`Player.java`、`VideoDetailFragment.java`），旧栈（RxJava、kapt、ExoPlayer 2.19.1——Google 已停止维护的旧包）与新栈并存，双栈期维护成本高。
3. 重写期冻结新功能：CONTRIBUTING 明示仅收 bugfix，新特性诉求在迁移完成前将积压；KMP 迁移工程量大，完成时点未确认。
4. 渠道局限：不上 Google Play 意味着获客依赖 F-Droid/GitHub 侧载；F-Droid 官方源更新滞后且签名独立（README 详述）。

## 10. 总结评价

NewPipe 是"无账号、无 GMS、多站点"安卓流媒体前端的事实标准实现：它的架构答案是前端与抽取器分离（NewPipeExtractor 独立仓 + 接口化消费）、播放内核复用（Player + 可插拔 PlayerUi 覆盖内嵌/悬浮/后台三形态）、单 Activity + Fragment 的经典 UI 组织，而它当下最有观察价值的动作是正用 Kotlin Multiplatform + Compose 把这十年的 Java 安卓工程绞杀式重写为跨平台形态。对想研究安卓播放器后台/画中画实现、站点适配器如何抽象、或大型安卓项目如何渐进迁移 KMP 的人，这个仓库都是少见的真实样本；对普通用户，它是 F-Droid 上最主流的开源视频客户端之一。
