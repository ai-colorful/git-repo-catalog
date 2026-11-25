# yuliskov/SmartTube 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Java｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：面向 Android TV / Fire TV 的开源 YouTube 客户端，自带完整的数据获取层（InnerTube API + SABR 流解析），无需 Google 服务即可在大屏设备上浏览与播放 YouTube 内容。

| 项 | 值 |
|---|---|
| 名称 | yuliskov/SmartTube（前身为 SmartYouTubeTV） |
| 星数 | 39500（快照；gh api 2026-11-21 核实 36508） |
| 主语言 | Java（核心数据层 Kotlin/Java 混合） |
| 许可证 | MIT（主仓及 SharedModules、MediaServiceCore 两个子模块均为 MIT，见 `LICENSE`、`.reuse/dep5`） |
| 当前版本 | 32.80（versionCode 2470，`smarttubetv/build.gradle`） |
| 最近 push | 2026-11-19（gh api 核实，活跃） |
| 平台 | Android TV / Fire TV / Chromecast with Google TV，minSdk 17（Android 4.3+），明确不支持手机/平板 |

## 2. 项目是什么，解决什么问题

SmartTube 是一个 TV 形态的 YouTube 第三方客户端：用户在没有 Google Play 服务的电视盒、老旧 Fire TV、国产安卓盒子上也能看 YouTube，并且自带 SponsorBlock 段落跳过、倍速、8K/60fps/HDR、画中画、直播聊天、DeArrow 缩略图替换等功能（功能清单见 `README.md` Features 节）。

与 NewPipe 这类"抽取器 + 手机 UI"的路线不同，SmartTube 是纯 TV 工程：UI 直接基于 Android 官方 Leanback 模板构建（甚至内置了一份魔改版 leanback 库），交互围绕遥控器 D-pad 设计，README 明确声明"不支持触屏、不出手机版"。它不依赖官方 YouTube for TV 应用，也不通过 WebView 内嵌页面，而是自建了一套对 YouTube 内部 InnerTube API 的请求与解析层。

工程上它要解决的核心难题是 YouTube 对第三方客户端层层加码的反制：签名参数（sig/s 参数）、n 参数限流、BotGuard/PoToken 证明、以及最新的 SABR 流式协议。SmartTube 在仓库内维护了 JS 求解器资产、TV 端 PoToken 生成器、自研 SABR 解析模块等对策，这也是它星数持续增长的技术口碑来源。

分发不走应用商店：README 明确警告不要从第三方 APK 站下载，官方渠道只有 GitHub Releases、F-Droid（fdroid flavor）和应用内自动更新。2025-10 后的新款 Fire TV（VegaOS）不支持（README Device support 节）。

## 3. 技术栈与依赖

- 语言与构建：Java 为主（主仓应用层约 860 个 .java / 214 个 .kt，非库目录统计）；Android Gradle Plugin 7.4.2，compileSdk 34 / minSdk 17（`SharedModules/constants.gradle`），CI 使用 OpenJDK 17（`.github/workflows/CI.yml`、README Build 节）
- 网络：Retrofit + okhttp 3.12（保 Android 4 兼容）+ Cronet 105（Chromium 网络栈，`constants.gradle`）；conscrypt 引入用于"ByeByeDPI fix"（`smarttubetv/build.gradle:254`）
- 播放器：ExoPlayer 2.10.6 的 Amazon fork 源码级集成（`exoplayer-amzn-2.10.6/`，1712 个文件），作者在其上新增了 `library/sabr` 模块
- JS 引擎（用于 PoToken/n 参数求解）：j2v8、duktape、quickjs、rhino 多引擎并存（`constants.gradle:115-120`），当前 PoToken 默认走 WebView 方案 `PoTokenWebView4`（`MediaServiceCore/youtubeapi/.../potokennp2/misc/PoTokenSelector.kt`）
- UI：Leanback 1.0.0（本地魔改副本 `leanback-1.0.0/`，构建时 exclude 官方 androidx.leanback，根 `build.gradle:60`）、Glide、chatkit（直播聊天 UI）、RxJava2 + Kotlin 协程
- 服务化接口：`MediaServiceCore/mediaserviceinterfaces` 定义 ServiceManager/ContentService/SignInService 等门面
- 可选遥测：Firebase Crashlytics 仅 stbeta/ststable flavor 启用，stfdroid flavor 不含（`smarttubetv/build.gradle:239-248`）

## 4. 目录结构与代码组织

多模块 Gradle 单体 + 两个 git 子模块（`settings.gradle`、`.gitmodules`）：

| 目录 | 职责 |
|---|---|
| `smarttubetv/` | 主应用模块（applicationId app.smarttube），TV UI 层：browse/playback/search/signin 等 Leanback 界面 |
| `common/` | 与 UI 解耦的应用层：MVP Presenter、播放控制器、偏好管理、ExoPlayer 封装 |
| `MediaServiceCore/`（子模块） | `youtubeapi`（InnerTube/Google API 数据层）+ `mediaserviceinterfaces`（服务接口） |
| `SharedModules/`（子模块） | `sharedutils`（日志/助手）、`appupdatechecker2`（应用内更新）、j2v8、commons-io 定制版 |
| `exoplayer-amzn-2.10.6/` | Amazon ExoPlayer fork 源码，含作者新增的 `library/sabr` SABR 流模块 |
| `leanback-1.0.0/`、`fragment-1.1.0/` | 魔改的 androidx leanback / fragment 本地库 |
| `chatkit/`、`doubletapplayerview/`、`slidableactivity/`、`filepicker-lib/`、`leanbackassistant/` | 内嵌的第三方/自研支撑库 |
| `.github/workflows/` | CI 构建 + VirusTotal 扫描 + stale/cleanup |
| `fastlane/` | F-Droid 元数据 |

组织评述：典型的"主仓 + 私有公共库子模块"结构，youtubeapi 数据层完全下沉到子模块复用；应用层用 MVP（presenter）+ 播放控制器组合（controller）分层，TV UI 与业务逻辑分离（common 不依赖 TV UI）。

## 5. 核心架构与关键实现

### 5.1 InnerTube 数据层：多客户端矩阵 + URL 解密流水线

`MediaServiceCore/youtubeapi/src/main/java/com/liskovsoft/youtubeapi/common/helpers/AppClient.kt` 定义了完整的 InnerTube 客户端矩阵（TV、TV_EMBED、TV_SIMPLY、TV_SAMSUNG、WEB、WEB_EMBED、ANDROID、ANDROID_VR、IOS、VISIONOS 等 20 余种），每个客户端带 clientName/clientVersion/userAgent/postData，注释中直接引用 yt-dlp 的实现作参考。请求体由 `common/helpers/QueryBuilder.kt` 组装。

拿到 player 响应后的 URL 解密流水线在 `videoinfo/VideoInfoServiceBase.java`：批量提取各格式的 s 参数与 n 参数 → `AppService.bulkSigExtract` 求解（JS 求解器资产在 `youtubeapi/src/main/assets/nsigsolver/`，含 meriyah/astring 与 `yt.solver.core.js`）→ 回填 URL → 再叠加 PoToken：

```java
Pair<List<String>, List<String>> result =
    mAppService.bulkSigExtract(videoInfo.getClient(), extractNParams(urlHolders), extractSParams(urlHolders));
...
String poToken = PoTokenGate.getPoToken(videoInfo.getClient(), videoInfo.getVideoDetails().getVideoId());
videoInfo.setPoToken(poToken);
applySessionPoToken(urlHolders, poToken);
```

### 5.2 PoToken/BotGuard：Web 与 TV 双轨生成

`youtubeapi/app/PoTokenGate.kt` 区分两类 PoToken：Web 内容 PoToken（按 videoId 生成、3 分钟缓存）与 TV"living room" PoToken（`TvPoTokenProvider`，使用 TV 客户端自己的 BotGuard 配置）。实现上有多代方案：`potokennp2/generators/` 下 PoTokenV8（j2v8 运行 `assets/potokennp2/v8/` 的 bootstrap 脚本）、PoTokenWebView 1-4（Android WebView 执行 BotGuard 挑战），当前默认 `PoTokenWebView4`（`PoTokenSelector.kt` 仅一行：`return PoTokenWebView4`）。visitorData 由 `potokennp2/visitor/` 获取。

### 5.3 播放集成：魔改 ExoPlayer + 自研 SABR 源

SmartTube 不满足于 DASH/HLS：作者在 Amazon ExoPlayer fork 内新增了 `exoplayer-amzn-2.10.6/library/sabr/`（约 50+ 类），实现 YouTube 新一代 SABR 流协议的客户端解析：`SabrManifestParser`、`DefaultSabrChunkSource`、`parser/adapter/` 下把 SABR 分段适配为 fMP4/Matroska，`parser/frames/` 下有 AVC/VP9/Opus/AAC/Vorbis 帧抽取器，并定义了 PoToken 失效、退避重载等专用异常（`SabrPoTokenError`、`SabrBackoffError` 等）。应用侧 `common/exoplayer/ExoMediaSourceFactory.java` 据格式信息选择构建 `SabrMediaSource`（含 SABR_READ_TIMEOUT_MS=4000 与自定义 `SabrDefaultLoadErrorHandlingPolicy`）。

播放器抽象接口 `common/app/models/playback/manager/PlayerEngine.java` 暴露 openSabr/openDash/openHlsUrl/openMerged 等多入口，使 DASH、SABR、HLS、URL 列表可按可用性切换。DASH 场景下由 `youtubeapi/formatbuilders/mpdbuilder/YouTubeMPDBuilder.java` 用自适应格式自行拼装 MPD 清单交给 ExoPlayer。

### 5.4 TV UI 与播放控制器组合

`common/app/presenters/` 是 MVP 核心（Browse/Playback/Search/Channel/SignIn 等 13+ presenter）；播放页把功能拆成 11 个可组合控制器（`common/app/models/playback/controllers/`）：SponsorBlockController（每秒轮询位置、按类别跳段并在进度条渲染段块）、AutoFrameRateController、ChatController、CommentsController、HQDialogController 等，全部挂在 PlaybackPresenter 下按事件驱动。TV 端 `smarttubetv/.../tv/ui/` 只承担 Leanback 视图。

### 5.5 广告规避的工程实现（客观描述）

应用本身不含任何广告展示代码（README FAQ "Can I whitelist ads" 明确回答没有广告代码）。对 YouTube 服务端广告的规避是请求层完成的：`QueryBuilder.kt:290` 在 playbackContext 中固定写入 `"isInlinePlaybackNoAd": true`（代码注释引用社区研究 https://iter.ca/post/yt-adblock/ 与 yt-dlp 的 use_ad_playback_context 提交），使 TV 客户端返回的播放清单不含广告段。内容内赞助段落则通过 SponsorBlock 公益 API 跳过（`youtubeapi/block/SponsorBlockService.java`，App 层 `SponsorBlockController.java`），并支持 DeArrow 缩略图/标题替换（`youtubeapi/dearrow/DeArrowService.kt`）。

### 5.6 登录、投放与自更新

- 登录：YouTube OAuth2 设备码流程（`youtubeapi/auth/V2/AuthApi.java`，POST `youtube.com/o/oauth2/device/code`），另有 Google Drive 备份（`googleapi/drive3`）
- 手机投放/遥控：实现 DIAL 协议 SSDP 发现（`common/misc/DialServer.java`，注释明确"Minimal DIAL server… show up in the cast dialog"）+ YouTube Lounge API 配对（`youtubeapi/lounge/`）
- 自更新：`SharedModules/appupdatechecker2/core/AppVersionChecker.java` 对比 versionCode、拉取 changelog 与 APK 下载地址，配合 `AppDownloader` 完成应用内更新

## 6. 构建与运行

来自 `README.md` Build 节与 CI 配置：

```bash
git clone https://github.com/yuliskov/SmartTube.git
cd SmartTube
git submodule update --init        # SharedModules + MediaServiceCore
# OpenJDK 17
gradlew clean installStstableDebug # 或 assembleStbetaRelease / assembleStfdroidRelease
adb connect <device_ip>            # 安装到电视
```

CI（`.github/workflows/CI.yml`）：master push 触发 → `./gradlew lintStbetaRelease` → `assembleStbetaRelease` → 按ABI上传 arm64/armv7/x86/universal APK → 全部 APK 提交 VirusTotal 扫描并在 Step Summary 输出徽章（对应 README 顶部关于构建环境安全事件的整改声明）。flavor：stbeta（org.smarttube.beta）/ ststable（org.smarttube.stable）/ stfdroid（app.smarttube.fdroid，无 Firebase）。

## 7. 测试与工程质量

- 测试以真机/Robolectric 混合的 androidTest 为主：主仓 67 个 androidTest 文件 + 各模块约 300 个测试文件；`youtubeapi/src/androidTest/` 下 `VideoInfoApiSignedTest`、`YouTubeMPDBuilderTest`、`AppServiceTest` 等直接对着真实 YouTube API 验证（属集成测试，依赖网络与时效，不稳定但贴近真实失效场景）
- 单元测试框架：JUnit 4 + Robolectric 4.6.1 + Espresso（`constants.gradle`）；app UI 层未见系统性单测覆盖
- 质量 gate：`lintOptions abortOnError true`（`smarttubetv/build.gradle:151`），CI 跑 lintStbetaRelease；action 全部按 commit SHA 固定版本
- 工程杂讯：大量兼容性 workaround 注释（targetSdk 27 保留"鼠标 D-pad 模拟"等，`constants.gradle:8-22`），版本锁定策略偏保守（okhttp 3.12、kotlin 1.8.10），属于长生命周期老设备兼容项目的典型形态

## 8. 社区与维护状态

- 活跃度：gh api 核实（2026-11-21）stars 36508、open issues 898、最近 push 2026-11-19；近期 release 节奏极快：32.74→32.80 在 2026-11-12 至 11-19 八天内发布 6 个版本（beta + stable 双轨，gh api releases）
- 贡献者结构：contributors API 统计 46 名贡献者、约 1145 次提交计入；README 自述"单人开发、无团队无公司"（Team 节），近期 SABR/PoToken 核心贡献单独致谢 pixkk、Jupiops（README Contributors 节）
- 社区渠道：Telegram（@SmartTubeNewsEN 变更日志频道、@SmartTubeEN 支持群）、Discord、Crowdin 翻译（`crowdin.yml`）、F-Droid 同步发布
- 风险事件：2026 年 README 顶部公告开发环境曾感染恶意软件、签名密钥可能泄露，作者全盘重装并新增 CI VirusTotal 扫描（README Important announcement 节；CI.yml 有对应实现）

## 9. 亮点与风险

**亮点**
1. 数据层工程深度罕见：多 InnerTube 客户端矩阵 + URL 解密 + 双轨 PoToken + SABR 协议解析全部自维护，且以子模块复用（`MediaServiceCore/youtubeapi`、`exoplayer-amzn-2.10.6/library/sabr`）
2. 设备兼容面极宽：minSdk 17，覆盖 Android 4.3 老盒子到 Chromecast/新 Fire TV，配套大量兼容 workaround（`SharedModules/constants.gradle`）
3. 架构分层清晰：数据接口（mediaserviceinterfaces）→ 应用层（common MVP/控制器）→ TV UI（smarttubetv）互不渗透，播放源（SABR/DASH/HLS）可插拔（`PlayerEngine.java`）
4. 分发闭环完整：GitHub Releases + F-Droid + 应用内自更新（versionCode 对比 + changelog），CI 自动构建并 VirusTotal 扫描
5. 合规自觉：REUSE 规范（`.reuse/dep5`）、PRIVACY.md、明确不发布到第三方商店的引导

**风险/局限**
1. 法律与条款风险：依赖 YouTube 私有 InnerTube API、`isInlinePlaybackNoAd` 等参数规避服务端广告，违反 YouTube 服务条款；代码虽为 MIT 许可，使用与再分发的法律风险由使用者自担，且 Google 反制（BotGuard、SABR 变更）可能随时使功能失效（客观陈述，出处 `QueryBuilder.kt:290`、`AppClient.kt`）
2. 猫鼠游戏维护成本高：PoToken 求解器、n 参数算法、SABR 解析都需跟随上游变更高频迭代（近期一天一版的 release 节奏即为佐证），单人主导 + 46 贡献者存在 bus factor
3. 仅 TV 生态：不支持手机/平板/非 Android 电视（README 明确拒绝），2025-10 后 VegaOS Fire TV 亦不支持
4. 侧载渠道仿冒风险：应用不在 Play Store，README 警告第三方 APK 站存在带广告/恶意版本，普通用户获取正版依赖官方下载链接与内置更新器

## 10. 总结评价

SmartTube 是当前开源界对抗 YouTube TV 端封锁最彻底的工程实现：它把 NewPipe 式的"抽取器"思路推进到了 InnerTube 客户端矩阵、TV PoToken 与 SABR 流解析的纵深，同时用标准 Leanback UI 和极低的 minSdk 换来了罕见的设备覆盖面。代码以 Java 为主、分层规整，数据层与 UI 层通过子模块和接口服务隔离，适合研究 YouTube 内部协议（InnerTube/PoToken/SABR）与 Android TV 大屏应用架构的读者精读；对普通用户价值明确，但需自担条款合规与随时失效的风险。它是"单人维护也能达到生产级复杂度上限"的代表性样本。
