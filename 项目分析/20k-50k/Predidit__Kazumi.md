# Predidit/Kazumi 项目分析

> 星数 42419（快照 2026-10-24）｜主语言 Dart｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：基于自定义 XPath/JSON 规则采集第三方番剧站点的 Flutter 跨平台播放器，集成了 mpv 播放内核、弹幕、Anime4K 实时超分辨率与完整的番剧追踪功能。

| 项 | 值 |
|---|---|
| 名称 | Predidit/Kazumi |
| 星数 | 42419（gh api 核实，2026-11-14） |
| 语言 | Dart（Flutter，gh api 核实） |
| 许可证 | GPL-3.0（`LICENSE`） |
| 版本 | 2.4.4+20404（`pubspec.yaml`），release 2.4.4 发布于 2026-11-12 |
| 最近活跃 | 默认分支 main，最近 push 2026-11-12（gh api 核实） |
| 规模 | lib/ 下 405 个 Dart 文件、约 7.48 万行；test/ 24 个文件约 5666 行 |
| 贡献者 | 30 人（gh api contributors） |

## 2. 项目是什么，解决什么问题

Kazumi 解决的核心问题是：不同番剧资源站的目录、搜索、播放页结构各不相同，观众要在各站点间手动跳转。它把"从某个站点找到视频"这件事抽象成一条**自定义规则**——README 称"最多五行 XPath 选择器"即可描述一个站（`README.md`），应用据此完成搜索、章节（番剧线路/分集）解析与直链提取，再交给内置播放器统一播放（`lib/plugins/plugins.dart`、`lib/services/plugin/rule_engine.dart`）。

规则本身是 JSON 对象，字段包括 `searchURL/searchList/searchName/searchResult/chapterRoads/chapterResult` 及 `searchMode/chapterMode`（xpath 或 api 两种模式）（`lib/plugins/plugins.dart`）。规则可导入、导出、分享，官方维护配套规则仓库 Predidit/KazumiRules，应用内置其 raw 地址作默认"规则商店"，并支持用户自填任意 HTTP 规则仓库（`lib/request/config/api_endpoints.dart` 的 `pluginShop`、`lib/request/config/rules_repository_config.dart` 的 `catalogUri/ruleUri` 约定 `index.json` + `<规则名>.json`）。README 的"贡献"一节明确引导社区向规则仓库提交规则（`README.md`）。

在采集之上，它补齐了完整的观看体验闭环：对接 Bangumi API 的番剧目录/时间表/收藏（`lib/modules/bangumi/`、`lib/request/clients/bangumi_client.dart`）、弹幕（弹弹play API，`lib/request/clients/danmaku_client.dart`）、追番/历史记录及 WebDAV 跨设备同步（`lib/services/sync/webdav.dart`）、番剧下载、DLNA 投屏、外部播放器调用、Syncplay 协议"一起看"（`lib/services/player/syncplay_client.dart`）。

工程上值得一提的还有网络层：针对部分 API 域名的访问受限问题，集成了 `ech_http`（基于 BoringSSL/curl 的 ECH 传输，见 `licenses/ech_http/`）与阿里 DoH 解析（`lib/request/core/ech_transport.dart`），对 Bangumi API 等域名走 ECH 加密连接；播放端则内置 m3u8 广告分段过滤（`lib/utils/m3u8_ad_filter.dart`，仿 FFmpeg hls_ad_filter 的 discontinuity 分组逻辑）。

## 3. 技术栈与依赖

- **框架**：Flutter 3.50.0 / Dart SDK >=3.10.0（`pubspec.yaml` environment）
- **架构/状态**：flutter_modular 7（模块化路由+依赖注入，`lib/app_module.dart`）、MobX 2（控制器 `with Store` + `.g.dart` 代码生成，39 个文件引用 mobx）
- **存储**：hive_ce（本地盒子，`lib/main.dart` 初始化）、webdav_client（云同步）
- **网络**：dio 5 + cookie_jar、ech_http（ECH 传输）、http；xpath_selector + xpath_selector_html_parser、json_path（规则解析双引擎）
- **播放**：media_kit / media_kit_video（libmpv 内核，全平台统一，`lib/pages/player/controller/player_playback_controller.dart`）、audio_service 及作者自维护的 mpris/win 扩展（git 依赖 `Predidit/audio-service-mpris`、`Predidit/audio_service_win`）
- **弹幕**：canvas_danmaku 0.3（自绘弹幕组件）
- **WebView**：flutter_inappwebview（iOS/macOS/Android）+ webview_windows（作者 fork）+ Linux 自实现，按平台策略分发（`lib/webview/captcha/impl/`、`lib/webview/video/impl/`）
- **其他**：antlr4（4.13）、dlna_dart（投屏）、flutter_foreground_task（后台下载）、window_manager/tray_manager（桌面端）、flutter_displaymode（高刷）

## 4. 目录结构与代码组织

| 目录 | 职责 | 行数 |
|---|---|---|
| `lib/pages/` | 按功能划分的 UI 层：popular/timeline/search/info/video/player/history/collect/settings/plugin_editor/onboarding 等，player 子目录有 20+ 文件 | ~41.5k |
| `lib/services/` | 领域服务层：plugin/（规则引擎）、player/（音频/截图/同步播放/下载策略）、sync/（WebDAV/收藏/历史/弹幕屏蔽同步）、network/、shaders/、download/、storage/、logging/、update/、deep_links/ | ~11.6k |
| `lib/modules/` | 数据模型（bangumi/danmaku/comments/download/history/roads 等域对象） | ~4.5k |
| `lib/request/` | HTTP 层：apis/（bangumi/danmaku/规则目录 API）、clients/、config/（端点与规则仓库配置）、core/（dio 工厂、ECH 传输、拦截器、错误映射） | ~2.2k |
| `lib/plugins/` | Plugin 规则模型、API 规则配置、反爬配置、规则控制器 | ~1.2k |
| `lib/webview/` | 验证码 WebView 与视频源 WebView，均按平台 impl 分发 | ~3.3k |
| `lib/bbcode/` | Bangumi BBCode 富文本渲染 | ~1.2k |
| `lib/utils/` | m3u8 解析/广告过滤、异步原语（rate limiter/single-flight/serial queue）、编码、番剧季度等 | ~1.8k |
| `android/ ios/ windows/ macos/ linux/ web/` | 六端 Flutter 平台壳（web 为占位，实际 Web 版走分支仓库） | - |
| `.github/workflows/` | pr.yaml、release.yaml | - |
| `test/` | 24 个 Dart 单测 | ~5.7k |

组织方式评述：典型 Flutter 分层架构——UI(pages) → 状态(MobX controller) → 服务(services) → 请求(request) → 模型(modules)，用 flutter_modular 做模块注册（每页一个 `*_module.dart`）。无 monorepo、无 code-gen 路由；代码生成仅用于 MobX（`.g.dart` 23 处）与 hive 适配器。分层清晰、命名一致，但 pages 目录体量偏大，播放器相关代码散在 pages/player 与 services/player 两处。

## 5. 核心架构与关键实现

### 5.1 规则引擎（策略模式双解析器）

`lib/services/plugin/rule_engine.dart` 是采集核心：`RuleEngine` 持有两个策略实现，按规则的 `searchMode/chapterMode` 在 `ApiRuleStrategy`（JSON API 规则，546 行，`api_rule_strategy.dart`）与 `XPathRuleStrategy`（HTML XPath 规则，333 行，`xpath_rule_strategy.dart`）间分发。请求执行抽象为 `RuleRequestExecutor` 接口（可注入替换，便于测试）；解析结果统一封装为带诊断信息的 `RuleSearchTrace`；对验证码页面抛 `CaptchaRequiredException` 交由上层 WebView 处理：

```dart
final parsed = config.searchMode == RuleMode.api
    ? _apiStrategy.parseSearch(raw, config.searchApiConfig)
    : _xpathStrategy.parseSearch(raw, config);
if (parsed.items.isEmpty) {
  throw NoResultException(config.pluginName);
}
```

（`lib/services/plugin/rule_engine.dart`）

### 5.2 反反爬体系（规则驱动 + 平台 WebView）

`lib/plugins/anti_crawler_config.dart` 定义了规则可声明的三种验证方式：图片验证码（WebView 抓图引导手输）、自动点击验证按钮、执行规则自带 JS；检测方式支持 XPath/text/regex。命中验证码后由 `captcha_verification_service.dart` 拉起对应平台 WebView（inappwebview / windows / linux 三套 impl，`lib/webview/captcha/impl/`），完成验证保存 Cookie 后重试搜索（`lib/services/plugin/plugin_cookie_manager.dart`）。视频源解析也有对应的 WebView 路径（`lib/services/video_source/webview_video_source_service.dart`、`lib/webview/video/impl/`）。

### 5.3 播放器内核与 Anime4K 超分辨率

播放内核为 media_kit（libmpv 封装）。实时超分即 mpv 的 GLSL shader 链：内置 9 个 Anime4K GLSL 文件（`assets/shaders/`，含 Upscale_CNN_x2 VL/M/S、Restore_CNN VL/M/S、AutoDownscalePre、Clamp_Highlights），启动时由 `ShaderAssetService` 拷贝到应用目录（`lib/services/shaders/shader_asset_service.dart`），播放时按"效率档/质量档"切换预设组合，直接以 mpv 原生命令挂载：

```dart
await pp.command([
  'change-list', 'glsl-shaders', 'set',
  buildShadersAbsolutePath(
    shaderAssetService.shadersDirectory.path, mpvAnime4KShaders),
]);
// off: await pp.command(['change-list', 'glsl-shaders', 'clr', '']);
```

（`lib/pages/player/controller/player_playback_controller.dart` 的 `setShader`）

播放器控制器按职责拆分为 playback/danmaku/panel/seek/screenshot/syncplay/debug 等多个 MobX store（`lib/pages/player/controller/`），交互层含手势、键盘快捷键、进度条、截图工作流等独立文件。

### 5.4 弹幕与一起看

弹幕走弹弹play API（`lib/request/config/api_endpoints.dart` 的 `dandanAPIDomain`），`DanmakuEntry` 解析 dandanplay 的 `m/p` 行格式（时间/类型/颜色/来源），含简繁转换 `chConvert` 与按来源（BiliBili/Gamer）标注（`lib/modules/danmaku/danmaku_module.dart`），渲染交给 canvas_danmaku 自绘组件；另有弹幕屏蔽规则及其 WebDAV 同步（`lib/modules/danmaku/danmaku_shield_rule.dart`、`lib/services/sync/danmaku_shield_sync_service.dart`）。"一起看"是 Syncplay 协议的纯 Dart 客户端实现（消息模型、ping 均值延迟、TLS 超时处理，`lib/services/player/syncplay_client.dart`）。

## 6. 构建与运行

来自 CI 与工程文件（`.github/workflows/release.yaml`、`pubspec.yaml`），未实际运行验证：

```bash
flutter pub get
flutter test                      # 24 个单测文件
flutter build apk --split-per-abi          # Android
flutter build windows                      # Windows（另有 msix 打包配置，SignPath 签名）
flutter build linux                        # Linux x64/arm64（依赖 libgtk-3/libayatana-appindicator/webkit2gtk 等）
flutter build ios --release --no-codesign  # iOS（侧载分发）
```

发行渠道：GitHub Releases、F-Droid、Flathub、AUR（`README.md`）。macOS 10.15+ 与 HarmonyOS/Web 分别由分支仓库 ErBWs/Kazumi、0Chencc/KazumiWeb 承接（`README.md`）。

## 7. 测试与工程质量

- 测试规模：24 个文件约 5666 行（`test/`），全部为纯 Dart 单元测试（flutter test），覆盖核心逻辑：规则引擎（`rule_engine_test.dart`、`api_rule_engine_test.dart`）、规则导入解析、m3u8 解析与广告过滤、WebDAV/收藏/历史同步、异步原语（rate limiter/single-flight/session）、下载路径迁移等；无 widget/integration 测试。
- CI：`pr.yaml` 对 PR 运行检查并支持手动触发各平台构建与 SignPath 签名；`release.yaml` 矩阵构建 Android/Windows/Linux(x64+arm64)/iOS 并发布 release。
- 质量观察：规则引擎、请求执行器等关键抽象均做了依赖注入便于测试（`rule_engine.dart` 构造参数）；统一日志服务（`lib/services/logging/`）、错误映射层（`lib/request/core/network_error_mapper.dart`、`player_error_mapper.dart`）；`analysis_options.yaml` 开启 lints。整体工程质量在同类 Flutter 应用中偏高。

## 8. 社区与维护状态

- 极活跃：gh api 显示最近 push 2026-11-12；release 节奏密集——2.4.0（10-22）、2.4.1（10-22）、2.4.2（10-23）、2.4.3（11-04）、2.4.4（11-12），约每周一版。
- 贡献者 30 人（gh api contributors），主要维护者 Predidit 同时维护 audio-service-mpris、audio_service_win 等配套 git 依赖与官方规则仓库 KazumiRules（`pubspec.yaml`、`api_endpoints.dart`）。
- open_issues 662（gh api），体量偏大，推测以用户反馈为主（未确认内部构成）。
- 配套生态：文档站 kazumi.app（规则开发教程）、Telegram 群、F-Droid/Flathub 上架，README 带 HelloGitHub/trendshift 徽章。

## 9. 亮点与风险

**亮点**
1. 规则采集体系设计完整：XPath/JSON 双解析策略 + 可声明的反反爬配置 + 平台化 WebView 验证 + 规则仓库分发协议（index.json），把"易变的上游站点"问题产品化了（`lib/services/plugin/`、`lib/plugins/`）。
2. 全平台一套 libmpv 内核，实时超分直接复用 mpv GLSL shader 管线挂载 Anime4K，效率/质量两档可切换，实现成本极低而效果直接（`assets/shaders/`、`setShader`）。
3. 工程化程度高：MobX + flutter_modular 分层清晰、核心逻辑可注入可测试、CI 覆盖五平台构建与签名。
4. 面向受限网络的工程对策：ECH 传输 + DoH、规则仓库多镜像（gh-proxy.org/ghfast.top）、m3u8 广告分段过滤（`ech_transport.dart`、`rules_repository_config.dart`、`m3u8_ad_filter.dart`）。
5. 体验闭环完整：Bangumi 数据、弹弹play 弹幕、WebDAV 同步、Syncplay 一起看、下载/投屏/外部播放器一应俱全。

**风险/局限**
1. 版权合规边界：应用本体不含内容，但规则生态指向的第三方采集站点及其内容授权状态存在争议，分发渠道因此受限（iOS 仅侧载、Web 需独立部署），这是此类工具的固有法律风险，各司法辖区结论不同。
2. 上游脆弱性：规则可用性完全依赖目标站点 DOM/接口稳定性，站点改版即失效，维护压力长期存在于规则生态而非应用本体（项目已内置 validity tracker 缓解，`plugin_validity_tracker.dart`）。
3. 单人核心贡献：30 名贡献者中核心开发高度集中于作者，配套依赖亦为其个人 fork，bus factor 偏低（基于 contributors 与 pubspec git 依赖推断）。
4. issues 积压 662 个，社区支持响应能力可能承压（未确认）。

## 10. 总结评价

Kazumi 是"规则化采集 + 专业播放器"路线在 Flutter 生态里完成度最高的实现之一：它把易变的第三方站点抽象为用户可编写的五行规则，用策略模式的双解析引擎、规则驱动的反反爬配置和官方规则仓库把这套生态工程化，再以 media_kit/libmpv 统一六端播放并借 mpv shader 管线白嫖了 Anime4K 实时超分，辅以 Bangumi/弹幕/WebDAV/Syncplay 构成完整观看闭环；代码分层规范、测试覆盖核心逻辑、CI 五平台发版，维护节奏近乎周更。适合关注 Flutter 桌面/移动应用架构、规则引擎与 WebView 混合方案、mpv GLSL 集成的开发者读源码；对普通用户则需自行理解规则生态与相关法律边界后使用。
