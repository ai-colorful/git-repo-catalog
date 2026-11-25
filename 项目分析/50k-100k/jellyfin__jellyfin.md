# jellyfin/jellyfin 项目分析

> 星数 58444（快照 2026-10-24）｜主语言 C#｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：**Jellyfin 是自由软件（FOSS）媒体系统，即"自己搭建的 Plex/Emby"——一个把本地影视、音乐、图书等媒体库组织好并流式推送到全平台客户端的开源后端服务器**。本仓库为其服务端主仓，前端在 jellyfin-web 等姊妹仓库。

| 事实项 | 值 |
|---|---|
| 名称/仓库 | jellyfin/jellyfin（默认分支 master） |
| 星数 | 58,444（快照），实时 API 查询 58,940；forks 5,757 |
| 主语言 | C#（.NET 10，`global.json` 指定 sdk 10.0.0） |
| 许可证 | GPL-2.0（`LICENSE`） |
| 当前主版本 | 13.0.0（`SharedVersion.cs`，master 开发版）；最新 release v12.4（2026-10-26） |
| 最近 push | 2026-11-01（GitHub API） |
| 规模 | 2,251 个 .cs 文件；60 个 API 控制器；315 个测试文件 |
| 历史 | 由 Emby 3.5.2 fork 而来并完全去私有化（`README.md` 自述） |

## 2. 项目是什么，解决什么问题

Jellyfin 解决的核心问题是：**把散落在磁盘上的个人媒体（电影、剧集、音乐、有声书、照片、Live TV）变成一个有元数据、有海报、可多端播放的"私人 Netflix"**，且完全由用户自托管、无订阅、无付费墙、不上传任何数据到云端（`README.md`："The Free Software Media System ... puts you in control of managing and streaming your media"）。

从代码看，服务端承担四类职责：

1. **媒体库管理**：扫描目录、按文件命名规则识别影视条目、抓取在线元数据与海报、维护统一的条目树（items 树），入口是 `Emby.Server.Implementations/Library/LibraryManager.cs`（4,234 行）与 `MediaBrowser.Controller/Library/ILibraryManager.cs`。
2. **元数据供给**：内置 TheMovieDb、MusicBrainz 等远程 Provider 与 NFO 本地解析（`MediaBrowser.Providers/` 160 文件、`MediaBrowser.XbmcMetadata/`、`MediaBrowser.LocalMetadata/`）。
3. **流媒体分发**：直接播放（direct play）不够时调用 ffmpeg 实时转码，输出 HLS/TS/MP4，核心在 `MediaBrowser.MediaEncoding/` 与 `Jellyfin.Api/Controllers/DynamicHlsController.cs`。
4. **多客户端 API**：60 个 ASP.NET Core 控制器（`Jellyfin.Api/Controllers/`）同时服务 Web、移动端、电视端、第三方客户端，OpenAPI 文档由 Swashbuckle 在运行时生成（`Jellyfin.Server/Extensions/ApiServiceCollectionExtensions.cs:197`）。

项目定位是 Emby/Plex 的完整替代品，因此代码里保留了 Emby 命名空间的"考古层"（`Emby.Server.Implementations/`、`Emby.Naming/`），这是 fork 历史留下的直接证据。

## 3. 技术栈与依赖

（依赖版本见中央依赖管理文件 `Directory.Packages.props`）

- **运行时/框架**：.NET 10 + ASP.NET Core（MVC 控制器式 API，非 Minimal API 为主体）
- **数据访问**：Microsoft.EntityFrameworkCore + `Jellyfin.Database.Providers.Sqlite`（本仓快照只内置 SQLite Provider，但 `src/Jellyfin.Database/` 抽象出了 Provider 体系，为多数据库预留）
- **日志/可观测**：Serilog（控制台/文件/Graylog sink）、prometheus-net（`Directory.Packages.props`）
- **媒体处理**：外部进程 jellyfin-ffmpeg（自维护 fork，`README.md` 安装前置），仓内 `MediaBrowser.MediaEncoding/Encoder/MediaEncoder.cs` 负责参数拼装与校验（`EncoderValidator.cs` 探测编码器可用性）
- **图像**：SkiaSharp + Svg.Skia + BlurHashSharp（缩略图/占位图）
- **API 文档**：Swashbuckle.AspNetCore（OpenAPI 生成，`.github/workflows/openapi-generate.yml` 自动产出规范）
- **韧性/工具**：Polly、AsyncKeyedLock、BitFaster.Caching、SharpCompress、PlaylistsNET、MetaBrainz.MusicBrainz
- **质量工具**：StyleCop.Analyzers、BannedApiAnalyzers（`BannedSymbols.txt`）、MultithreadingAnalyzer、renovate 自动依赖升级（`.github/renovate.json`）
- **测试**：xunit v3 + Moq + AutoFixture + FsCheck（属性测试）+ SharpFuzz（模糊测试）+ Microsoft.AspNetCore.Mvc.Testing（集成测试）

## 4. 目录结构与代码组织

单仓多项目（一个 `Jellyfin.slnx` 解决方案，约 25 个项目），按"Model → Controller（抽象层）→ Implementations（实现层）"三层分层，命名上保留 Emby 遗产与 Jellyfin 新层并存：

| 目录 | 职责（均以仓内实际内容为准） |
|---|---|
| `MediaBrowser.Model/`（307 cs） | 跨进程 DTO 与枚举——API 契约层，不含业务逻辑 |
| `MediaBrowser.Common/`（41 cs） | 基础抽象：插件接口（`Plugins/IPlugin.cs`、`BasePlugin.cs`、`LocalPlugin.cs`、`PluginManifest.cs`）、配置、事件 |
| `MediaBrowser.Controller/`（387 cs） | 服务端核心抽象层：`Entities/BaseItem.cs`、`Library/ILibraryManager.cs`、Provider/Persistence/Streaming 等全部接口 |
| `Emby.Server.Implementations/`（156 cs） | 核心实现：`Library/LibraryManager.cs`（4,234 行）、`Plugins/PluginManager.cs`+`PluginLoadContext.cs`、Session、安全 |
| `Jellyfin.Api/`（170 cs） | HTTP 层：60 个控制器 + 自定义鉴权策略（`Auth/`）+ 流媒体 Helper（`Helpers/`） |
| `Jellyfin.Server/`（89 cs） | 宿主启动项目：`Program.cs`（Serilog + DI 组装）、`CoreAppHost.cs`、`Migrations/` |
| `Jellyfin.Server.Implementations/`（67 cs） | EF Core 数据层之上的新实现（对旧 Emby 实现的渐进替换） |
| `Jellyfin.Data/`（26 cs） | 数据库实体与枚举 |
| `src/Jellyfin.Database/` | EF Core Provider 抽象 + SQLite Provider + Migrations |
| `MediaBrowser.MediaEncoding/`（27 cs） | 转码引擎：`Encoder/`、`Transcoding/TranscodeManager.cs`、`Probing/`、`Subtitles/`、`Attachments/` |
| `src/Jellyfin.MediaEncoding.Hls/`、`Keyframes/` | HLS 播放列表生成、关键帧解析 |
| `MediaBrowser.Providers/`（160 cs） | 远程元数据 Provider（TMDB/MusicBrainz 等）与刷新调度 |
| `MediaBrowser.XbmcMetadata/`、`MediaBrowser.LocalMetadata/` | NFO/Kodi 风格本地元数据读写 |
| `Emby.Naming/`（48 cs） | 文件名/路径识别规则（从文件名推断剧名、季、集、分辨率） |
| `src/Jellyfin.LiveTv/`、`Jellyfin.Networking/`、`Jellyfin.Drawing.Skia/`、`Jellyfin.Extensions/` | 直播电视、网络配置、图像绘制、通用工具 |
| `tests/`（17 个测试项目，315 cs）、`fuzz/`（2 个 fuzz 项目） | 测试与模糊测试 |
| `deployment/`（unraid 模板）、`.devcontainer/` | 部署模板与云开发环境 |

组织方式评述：**接口与实现彻底分离**是最大特色——`MediaBrowser.Controller` 定义几百个接口，`Emby.Server.Implementations`/`Jellyfin.Server.Implementations` 提供实现，插件与客户端 SDK 只依赖抽象层。新代码用 `Jellyfin.*` 前缀、旧代码保留 `Emby.*`/`MediaBrowser.*`，形成可辨识的新旧边界。

## 5. 核心架构与关键实现

### 5.1 媒体库模型：一切皆 BaseItem 的条目树

`MediaBrowser.Controller/Entities/BaseItem.cs`（3,124 行）是整个媒体库的根类：电影、剧集、音乐、照片全部继承自它或其子类（同目录下 `Movie.cs`、`Audio/`、`Book.cs`、`Photo.cs`、`Series/` 等），容器型条目（`Folder.cs`、`CollectionFolder.cs`、`AggregateFolder.cs`）构成虚拟文件系统式的层级。条目查询走 `InternalItemsQuery.cs` 翻译成 EF Core 查询，用户维度的播放状态/收藏由 `IUserDataManager` 分离存储。`BaseItem` 聚合了太多横切能力（路径解析、图片、媒体源、权限），是全仓最"上帝对象"的文件，也是理解源码的必读入口。

### 5.2 扫描/识别/元数据管线

入库链路：目录扫描 → `Emby.Naming`（48 cs）按文件名/路径模式识别（剧名、季集、版本、3D 标记等）→ `Emby.Server.Implementations/Library/` 下 `ResolverHelper.cs` + `Resolvers/` 决定条目类型 → `MediaBrowser.Providers/`（160 cs）并发拉取 TMDB 等远程元数据 → `IMetadataSaver`/`IMetadataFileSaver`（`MediaBrowser.Controller/Library/`）写库或写 NFO。`LibraryManager.cs`（4,234 行）是这个管线的编排中心，含 `Validators/`（定期全库校验任务）与变更事件（`ItemChangeEventArgs.cs`）推给客户端实时刷新。

### 5.3 流媒体与转码管线

播放请求先判定 direct play / direct stream / transcode 三态（`Jellyfin.Api/Helpers/MediaInfoHelper.cs` + `MediaBrowser.Model/Dlna/` 设备画像），需要转码时走：

```csharp
// Jellyfin.Api/Controllers/DynamicHlsController.cs
[Route("")]
[Authorize]
public class DynamicHlsController : BaseJellyfinApiController
{
    private const EncoderPreset DefaultVodEncoderPreset = EncoderPreset.veryfast;
    private const EncoderPreset DefaultEventEncoderPreset = EncoderPreset.superfast;
    private const TranscodingJobType TranscodingJobType = ...TranscodingJobType.Hls;
```

`DynamicHlsController` 生成 HLS 主播放列表（`src/Jellyfin.MediaEncoding.Hls/`），按关键帧切分片段（`src/Jellyfin.MediaEncoding.Keyframes/` 直接解析 MKV/MP4 容器关键帧，避免全量 probe）；实际 ffmpeg 进程由 `MediaBrowser.MediaEncoding/Encoder/MediaEncoder.cs` 启动，转码会话生命周期统一由 `Transcoding/TranscodeManager.cs`（实现 `ITranscodeManager`）管理：注册/取消/超时回收，支持多客户端共享同一转码会话。字幕外挂/内嵌烧录在 `MediaBrowser.MediaEncoding/Subtitles/`。

### 5.4 插件系统：AssemblyLoadContext 级隔离

插件接口极简（`MediaBrowser.Common/Plugins/IPlugin.cs` 仅 Name/Id/Version/AssemblyFilePath 等属性），能力扩展靠叠加接口：`IHasPluginConfiguration`（自带配置页）、`BasePluginOfT<TOptions>`。加载侧在 `Emby.Server.Implementations/Plugins/`：

```csharp
// Emby.Server.Implementations/Plugins/PluginLoadContext.cs
public class PluginLoadContext : AssemblyLoadContext
{
    public PluginLoadContext(string path) : base(true)
    {
        _resolver = new AssemblyDependencyResolver(path);
    }
    protected override Assembly? Load(AssemblyName assemblyName) { ... }
}
```

每个插件一个可回收的 `AssemblyLoadContext`，依赖通过 `AssemblyDependencyResolver` 私有解析，实现插件与宿主、插件与插件之间的程序集隔离；`LocalPlugin.cs` 负责从 manifest 解析本地已装插件，`Jellyfin.Api/Controllers/PluginsController.cs` + `PackageController.cs` 提供在线插件目录安装/卸载 API。服务端能力（定时任务、会话、用户管理）都通过 `MediaBrowser.Controller` 的接口暴露给插件注入点。

### 5.5 多客户端 API 与鉴权

60 个控制器按资源域切分（Items/Movies/Shows/Audio/LiveTv/Sessions/SyncPlay/Playstate...），统一继承 `BaseJellyfinApiController`。鉴权是自研组合策略而非简单 Role：`Jellyfin.Api/Auth/` 下有 `CustomAuthenticationHandler.cs`（token/设备识别）、`UserPermissionPolicy/`、`FirstTimeSetupPolicy/`（首装向导期放行）、`AnonymousLanAccessPolicy/`（LAN 匿名访问）、`LocalAccessOrRequiresElevationPolicy/`、`SyncPlayAccessPolicy/`，可精确表达"仅局域网匿名浏览、远程需登录"这类自托管场景的权限语义。

## 6. 构建与运行

来自 `README.md` 与 `global.json`（未实际运行验证）：

```bash
# 前置：.NET 10 SDK + jellyfin-ffmpeg；web 客户端需另行获取（jellyfin-web 仓库或已安装的发行版目录）
git clone https://github.com/jellyfin/jellyfin.git
cd jellyfin
dotnet build                                     # 构建
dotnet run --project Jellyfin.Server --webdir /absolute/path/to/jellyfin-web/dist  # 直接跑
dotnet test Jellyfin.slnx                        # 全量测试（与 CI 一致，见 .github/workflows/ci-tests.yml）
# 构建产物直接运行：Jellyfin.Server/bin/Debug/net10.0/jellyfin(.exe)
```

- 默认端口 8096；Swagger 文档在 `http://localhost:8096/api-docs/swagger/index.html`（README）
- 支持 GitHub Codespaces 开发（`.devcontainer/devcontainer.json` 内含 ffmpeg 安装脚本）
- Docker 镜像由组织侧流程产出，本仓仅有 `deployment/unraid/` 模板与 `.dockerignore`

## 7. 测试与工程质量

- **测试规模**：`tests/` 17 个测试项目、315 个测试文件；大头在 `Jellyfin.Server.Implementations.Tests`（98 文件）、`Jellyfin.Providers.Tests`（34）、`Jellyfin.Server.Integration.Tests`（33，基于 `Microsoft.AspNetCore.Mvc.Testing` 的整服务进程集成测试）、`Jellyfin.Naming.Tests`（28，文件名识别回归）
- **测试风格**：单元为主 + 集成测试为辅；引入 FsCheck 属性测试与 2 个 SharpFuzz 模糊测试项目（`fuzz/Emby.Server.Implementations.Fuzz/`、`fuzz/Jellyfin.Api.Fuzz/`），对输入解析类代码（命名规则、HTTP）做 fuzz，这在 C# 项目中较为少见
- **CI**：`.github/workflows/` 共 15 个 workflow——`ci-tests.yml`（dotnet test）、`ci-format.yml`、`ci-codeql-analysis.yml`（安全扫描）、`ci-compat.yml`、openapi 五件套（规范生成/校验/合并）、`release-bump-version.yaml`、renovate/issue 自动化
- **静态质量**：StyleCop + BannedApiAnalyzers（禁用 API 清单 `BannedSymbols.txt`）+ 并发分析器 + `src/Jellyfin.CodeAnalysis/` 自定义分析器；中央依赖版本（`Directory.Packages.props`）统一升级
- 观察点：`BaseItem.cs`（3,124 行）与 `LibraryManager.cs`（4,234 行）过大，属于历史包袱集中区；新旧实现（Emby.* vs Jellyfin.*）并存带来的双轨维护成本是可预期的债务

## 8. 社区与维护状态

- **活跃度**：极活跃。最近 push 2026-11-01（GitHub API）；近 5 个 release 为 v12.0（2026-09-08）→ v12.1（09-15）→ v12.2（10-05）→ v12.3（10-23）→ v12.4（10-26），保持月内多发的节奏
- **贡献者**：CONTRIBUTORS.md 列出约 315 名；分页 API 拉满 300 条仍有余量，规模 300+
- **治理**：Jellyfin GitHub 组织下多仓库分工（web 客户端、android 客户端、ffmpeg fork、文档、jellyfin-ux），主仓 `.github/CODEOWNERS` + issue 模板 + stale bot + 项目自动化 workflow 表明流程化程度高；翻译走 Weblate（README 徽章）、功能投票走 Fider（features.jellyfin.org）
- **生态位**：README 明确自述为 Emby/Plex 替代品；无商业授权版本、无 premium 功能分级的定位与 Plex 形成差异

## 9. 亮点与风险

**亮点**

1. **自托管媒体服务器的标杆实现**：媒体识别、元数据、转码、多端 API、直播电视、SyncPlay 全功能闭环，且 GPL-2.0 完全免费无功能阉割（`README.md`、控制器清单）
2. **接口/实现分层 + 插件 ALC 隔离**：`MediaBrowser.Controller` 数百接口定义稳定扩展面，`PluginLoadContext`（`Emby.Server.Implementations/Plugins/PluginLoadContext.cs`）用 AssemblyLoadContext 做插件级程序集隔离，工程上很现代
3. **转码管线精细**：关键帧直读容器（`src/Jellyfin.MediaEncoding.Keyframes/`）、VOD/直播两档编码预设、转码会话统一管理（`TranscodeManager.cs`），对 ffmpeg 的封装有深度
4. **测试文化好**：315 个测试文件 + 集成测试 + 属性测试 + fuzz，CI 15 个 workflow 含 CodeQL 与 OpenAPI 契约校验
5. **社区基础设施成熟**：月级 release 节奏、300+ 贡献者、Weblate/Fider/renovate 全套自动化

**风险/局限**

1. **历史包袱重**：Emby fork 遗留命名与新 Jellyfin 层并存；`BaseItem.cs` 3,124 行、`LibraryManager.cs` 4,234 行的巨型类抬高理解与改动成本
2. **数据库支持单一**：仓内仅内置 SQLite Provider（`src/Jellyfin.Database/Jellyfin.Database.Providers.Sqlite/`），超大库（数十万条目）的并发写性能天花板受 SQLite 限制（旧版曾支持的其他 Provider 已移出/待回归，未在本快照确认）
3. **强依赖外部 ffmpeg**：转码正确性依赖 jellyfin-ffmpeg 版本（`EncoderValidator.cs` 做能力探测），异构硬件加速路径（NVENC/QSV/VAAPI）排障门槛高
4. **master 版本号（13.0.0）与 release 标签（v12.x）存在代际差**，跟 master 开发的用户需自行承担不稳定性

## 10. 总结评价

Jellyfin 是"自托管流媒体"赛道星数最高、功能最全的 FOSS 实现：以 .NET 10 + EF Core + ASP.NET Core 为骨，靠接口分层与插件隔离维持了 2,200+ 文件规模下仍可扩展的架构，转码与元数据两条核心管线都有扎实的工程细节（关键帧切分、编码器探测、ALC 隔离、fuzz 测试）。主要代价是 Emby 遗产带来的巨型类与双轨代码。适合的读者/使用者：想自建私人媒体库的普通用户（直接用发行版）；想研究流媒体服务端如何组织条目模型、HLS 转码会话管理、插件沙箱的工程师（本仓是中文互联网上少见的完整范本）；想写插件或第三方客户端的开发者（`MediaBrowser.Controller` + OpenAPI 即完整契约）。不建议直接改 master 大文件练手，从 `Jellyfin.*` 新层与 tests 目录入手更平滑。
