# PhilJay/MPAndroidChart 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Kotlin（gh api 2026-11-14 核实：Kotlin 1.48MB vs Java 3KB；任务参数 LANG=Java 为 3.x 历史信息）｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：Android 生态最经典的 View 体系图表库，自 2014 年服役至今，2026 年 9 月由原作者以 Kotlin 全量重写并新增 Jetpack Compose 模块的方式发布 4.0"复活"。

| 项目 | 事实 |
|---|---|
| 名称 | PhilJay/MPAndroidChart |
| 星数 | 39500（快照）；gh api 2026-11-14 读数 38187 |
| 主语言 | Kotlin（4.0 重写后；3.x 时代为 Java，现仅余约 3KB Java） |
| 许可证 | Apache-2.0（`LICENSE`） |
| 默认分支 | master |
| 最近 push | 2026-10-09（gh api） |
| 最新 release | v4.0.1（2026-09-28，gh api） |
| 贡献者 | 81 人（gh api contributors，含匿名） |
| 发布渠道 | JitPack（`jitpack.yml`） |

## 2. 项目是什么，解决什么问题

MPAndroidChart 是 Android 平台的老牌图表库，提供 9 种图表类型（折线、柱状、横向柱状、饼图、散点、蜡烛、气泡、雷达、组合图），支持缩放、拖拽、高亮、标记（marker）、动画与"对每个像素的样式控制"，README 自述"三十万数据点保持流畅"（`README.md`；注意 README 实际写的是 "Thirty thousand points stay smooth"，即 3 万点）。它解决的痛点是：在 Android 原生没有系统级图表组件的年代，给开发者一个零第三方依赖、纯 Canvas 自绘、可深度定制的图表引擎。

项目历史有清晰的两段式：2014 年起作者 Philipp Jahoda 用 Java 构建，2016-2017 密集迭代 3.0.x，2019-03 发布 v3.1.0 后进入近 7 年的维护停滞（gh api releases 时间线可证：v3.1.0 2019-03-20 → 下一个 release 直到 2026-09）。2026 年 9 月，作者一次性发布 v3.2.0（为旧坐标补发）和 v4.0.0（Kotlin 全量重写，`CHANGELOG.md`："A rewrite of the library in Kotlin. Same class, package and interface names, so imports survive"），三天后跟进 v4.0.1 修复批次（关闭 33 个 issue/PR，见 `CHANGELOG.md` 末段链接列表）。

4.0 重写的核心变化（`MIGRATION.md`）：getter/setter 变属性、Entry 携带类型化 payload（`Entry(x, y, data = order)` 是 `Entry<Order>`）、formatter/listener 改为 `fun interface` 支持 lambda、颜色等改为 list、尺寸统一 dp 存储、新增 Jetpack Compose 模块 `MPChartCompose`、新增像素列降采样（`isDecimationEnabled`）与数据变化动画（`animateDataChange`）。包名 `com.github.mikephil.charting` 保持不变（`MPChartLib/build.gradle.kts`），老用户迁移只需按 MIGRATION.md 改调用点。

iOS 侧有对应姊妹项目 danielgindi/Charts（README 首段明确提及），可见该库的设计在移动图表领域影响力深远。

## 3. 技术栈与依赖

- 语言：Kotlin 2.2.20 编译，但 `apiVersion`/`languageVersion` 锁定 2.0，`coreLibrariesVersion = "2.0.21"`，保证 Kotlin 2.0 消费者可用（`MPChartLib/build.gradle.kts`）
- 构建：Gradle 9.7 + AGP 9.4.1（`gradle/libs.versions.toml`：agp 9.4.1，kotlin 2.2.20），Java 17 目标（CI 运行环境 Temurin 21）
- SDK 要求：minSdk 23，compileSdk 35；MPChartCompose 因 Compose 自身要求 compileSdk 37（README Requirements 节）
- 运行时依赖：库本体零第三方依赖——`MPChartLib/build.gradle.kts` 的 dependencies 仅 `testImplementation(libs.junit)`，全部能力基于 Android SDK Canvas/Paint 自绘
- Compose 模块依赖 androidx compose-bom 2026.09.00、activity-compose 等（`gradle/libs.versions.toml`）
- 工具链：Dokka 2.2.0（KDoc → javadoc jar 发布）、kotlin abi-tools（公共 API dump 校验）、JitPack（jdk openjdk21）

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `MPChartLib/` | 核心图表库，155 个 kt 文件、约 2.14 万行（`src/main/kotlin` 统计） |
| `MPChartCompose/` | Jetpack Compose 包装模块：9 个 composable + ChartState |
| `MPChartExample/` | 演示 app，76 个 kt，每种图表/特性一个 Activity（含 Java 示例页） |
| `MPChartDeviceTest/` | 设备端 instrumented 测试，25 个 kt（动画、轴绘制、位图导出等） |
| `MPChartLib/api/` | Kotlin ABI dump，`apiCheck` 任务比对公共 API（根 `build.gradle.kts` 自定义 AbiDump/AbiCheck 任务） |
| `design/`、`screenshots/` | logo 与演示截图素材 |
| `gradle/` | version catalog（libs.versions.toml） |
| `.github/` | build CI、issue 模板、FUNDING.yml |

组织方式为单仓多 Gradle 模块（`settings.gradle.kts` include 四模块），库/Compose/示例/设备测试职责分明，非 monorepo。

`MPChartLib` 内包结构（`src/main/kotlin/com/github/mikephil/charting/`）：

| 包 | 内容 |
|---|---|
| `charts/` | 13 个文件：Chart 基类、BarLineChartBase、PieRadarChartBase + 9 种具体图表 |
| `data/` | 30 个文件：Entry/DataSet/ChartData 三层 + `filter/Approximator`（Douglas-Peucker 抽稀） |
| `renderer/` | 24 个文件：数据渲染器 + 轴渲染器 + `scatter/` 8 个形状渲染器 + PointReducer |
| `components/` | 轴、图例、LimitLine、Marker 等周边组件 |
| `animation/` | ChartAnimator、Easing、ValueTransition |
| `highlight/` | 高亮定位策略（ChartHighlighter 及 6 个变体） |
| `listener/` | 触摸手势（BarLineChartTouchListener 等） |
| `jobs/` | 视口动画 Job（缩放/平移） |
| `interfaces/` | dataprovider 与 datasets 两组接口（11 个 IDataSet 系列） |
| `buffer/`、`utils/` | 绘制缓冲、Transformer 坐标变换、ViewPortHandler、对象池 |

## 5. 核心架构与关键实现

### 5.1 图表类型体系（`MPChartLib/.../charts/`）

顶层抽象 `Chart<T : ChartData<out IDataSet<out Entry<*>>>>`（`charts/Chart.kt`，1078 行，继承 ViewGroup）持有所有图表共享的部分：数据、坐标轴组件、图例、paint、高亮、marker、动画器与视口 job。其下分两个家族：

- `BarLineChartBase`（1033 行）：加入 x/y 轴、拖拽缩放与视口管理 → 派生 LineChart、BarChart、HorizontalBarChart、BubbleChart、CandleStickChart、CombinedChart（6 种）
- `PieRadarChartBase`：加入旋转手势 → 派生 PieChart、RadarChart（2 种）

加上 CombinedChart 共 9 种图表类型（`MPChartLib/Module.md` 明确列全）。具体图表类薄到极致——`charts/LineChart.kt` 仅 54 行，职责只是绑定数据类型、渲染器与高亮器：

```kotlin
// charts/LineChart.kt —— 具体图表只做"装配"
init {
    initRenderer()
    initMarker()
    this.highlighter = ChartHighlighter(this)
    axisLeft.mLabelCount = 6
}
```

### 5.2 渲染器体系（`MPChartLib/.../renderer/`，24 个文件）

绘制与数据彻底分离。`Renderer` → `DataRenderer`（140 行）定义四阶段绘制模板：每次数据变化调 `initBuffers`，每帧依次 `drawData` / `drawHighlighted` / `drawExtras` / `drawValues`（`renderer/DataRenderer.kt` 类注释）。各图表渲染器一一对应：LineChartRenderer（879 行，含 linear 与 cubic/horizontal bezier 路径）、BarChartRenderer、PieChartRenderer、CandleStickChartRenderer、BubbleChartRenderer、RadarChartRenderer、ScatterChartRenderer、CombinedChartRenderer；轴侧为 AxisRenderer → XAxisRenderer / YAxisRenderer 及 HorizontalBarChart、RadarChart 变体（共 23+ 文件）。散点形状独立成 `renderer/scatter/` 的 `IShapeRenderer` 策略接口 + 圆/方/三角/十字/X/双箭头 8 个实现。

动画与渲染器的耦合点是 `ChartAnimator` 的 phaseX/phaseY（0..1），渲染器据此只画前 phaseX 比例的条目、把 y 值乘以 phaseY（`renderer/LineChartRenderer.kt:197` `cur.y * cur.y * phaseY` 处为 `cubicPath.moveTo(cur.x, cur.y * phaseY)`）。性能关键是 `renderer/PointReducer.kt`：按像素列聚合条目、每列只保留最大者（气泡按 size 判断），配合 `BarLineChartBase.isDecimationEnabled` 实现 3 万点流畅（README + CHANGELOG 4.0.0 Added 节）。

### 5.3 数据抽象（`MPChartLib/.../data/` + `interfaces/datasets/`）

经典三层：Entry（点）→ DataSet（一个序列及其样式）→ ChartData（全部序列 + 值域缓存）。4.0 引入类型化 payload：

```kotlin
// data/Entry.kt / data/BaseEntry.kt
public abstract class BaseEntry<out D>(
    public var y: Float = 0f,
    public var icon: Drawable? = null,
    public val data: D?          // 类型化业务载荷
)
public open class Entry<out D>(public var x: Float, ...) : BaseEntry<D>(...)
```

条目子类 BarEntry（含 stackValues 堆叠值）、PieEntry、CandleEntry（高低开合）、BubbleEntry（size）、RadarEntry；数据集侧 `BaseDataSet → DataSet → BarLineScatterCandleBubbleDataSet → LineRadarDataSet → LineDataSet` 链式继承，并与 `interfaces/datasets/` 的 IDataSet/ILineDataSet/IBarDataSet 等 11 个接口分离；另含 `data/filter/Approximator`（Douglas-Peucker 抽稀，供超大序列预处理，`Module.md` 注明库内不自动调用）。

### 5.4 动画、触摸交互与高亮

- 动画（`animation/ChartAnimator.kt`，166 行）：ValueAnimator 驱动 phaseX/phaseY，槽位化管理（x/y/spin/每个条目/数据变化各一槽），同一槽新动画取消旧动画且旧动画仍触发 onEnd；`animation/Easing.kt` 提供缓动曲线
- 触摸（`listener/BarLineChartTouchListener.kt`，464 行）：VelocityTracker + 手势状态机（NONE/DRAG/PINCH_ZOOM/X_ZOOM/Y_ZOOM），双指缩放按两指间距方向自动选轴，支持双击缩放、惯性减速滚动与高亮拖拽；饼/雷达图的旋转在 `PieRadarChartTouchListener`
- 高亮（`highlight/ChartHighlighter.kt`）：触摸像素 → 值空间 x（经 `utils/Transformer` 矩阵），在每个开启高亮的 DataSet 找最近条目，按 y 轴偏好取像素距离最近者，受 `maxHighlightDistance` 约束；Bar/HorizontalBar/Pie/Radar/Combined 各有变体
- 视口动画（`jobs/`）：ZoomJob/MoveViewJob 及 Animated 变体，图表未获尺寸时排队至 `onSizeChanged`（`charts/Chart.kt` jobs 注释）

### 5.5 Compose 模块（`MPChartCompose/`）

4.0 补齐了 Compose 适配（任务提示中"Compose 缺位"的旧印象已不成立）：`compose/Charts.kt` 提供 9 个与图表同名的 composable；`compose/ChartState.kt`（`rememberChartState()`，rememberSaveable 支持）暴露 selectedEntry、可见范围、zoom，并提供 highlight/zoomIn/fitScreen/moveViewToX/animateX/animateDataChange 命令；`compose/ChartView.kt` 通过 AndroidView 桥接 View 体系图表，`ComposeMarker.kt` 支持用 composable 写 marker。用法见 README "Quick start with Compose"（`setup` 单次配置、`marker` lambda 化）。

## 6. 构建与运行

来自 CI（`.github/workflows/build.yml`）与 README：

```bash
./gradlew :MPChartLib:test :MPChartLib:lintRelease :MPChartCompose:lintRelease :MPChartExample:assembleDebug
./gradlew dokkaGenerate          # 生成 API 文档（javadoc jar 交 JitPack）
./gradlew apiCheck               # 校验公共 API 与 api/*.api dump 一致（根 build.gradle.kts）
```

消费（README Install 节，JitPack）：

```kotlin
implementation("com.github.PhilJay.MPAndroidChart:MPChartLib:v4.0.1")
implementation("com.github.PhilJay.MPAndroidChart:MPChartCompose:v4.0.1") // 仅 Compose
```

3.x 旧坐标 `com.github.PhilJay:MPAndroidChart` 固定在 v3.x 不受影响（README 明确说明）。

## 7. 测试与工程质量

- 单元测试：`MPChartLib/src/test/kotlin` 26 个文件（JUnit 4），覆盖数据类（BarDataTest/StackedBarTest/TypedEntryTest）、渲染器（BarRendererTest/AxisRendererTest/RendererEdgeCaseTest）、高亮（HighlighterTest）、格式化与回归（RegressionTest、HostileDataTest、EntryIndexLookupTest）
- 设备测试：`MPChartDeviceTest` 25 个 instrumented 测试（AnimationTest、BackgroundThreadTest、BitmapExportTest、BarBoundsTest 等，测真实绘制路径）
- Compose 测试：MPChartCompose 1 个单元测试（状态保存）+ 4 个 androidTest（含 LazyList 与图表重建场景）
- CI：push/PR 触发，Temurin 21，跑单测 + 两模块 lintRelease + 示例 assembleDebug + dokkaGenerate（`.github/workflows/build.yml`）
- 质量观察：库启用 `explicitApi()` 与 `allWarningsAsErrors`（`MPChartLib/build.gradle.kts`）；公共 API 有 ABI dump 快照校验（根 `build.gradle.kts` 自定义任务 + `MPChartLib/api/`）；KDoc 覆盖全部公共成员并随发布输出 javadoc（README Documentation 节）

## 8. 社区与维护状态

- 活跃度：当前活跃。最近提交 2026-10-09（gh api commits，作者 Philipp Jahoda）；2026-09-19 beta01 → 2026-09-26 v3.2.0 与 v4.0.0 → 2026-09-28 v4.0.1 连续三发（gh api releases）
- 历史停滞属实：v3.1.0（2019-03-20）到 v4.0.0-beta01（2026-09-19）之间无任何 release，"近年几近停更"的旧印象对 2019-2026 阶段成立；4.0 重写后状态彻底扭转
- 贡献者结构：81 名贡献者（gh api），核心贡献高度集中于作者本人（近期提交全部为 Philipp Jahoda，单人主导）
- Issue 处理：v4.0.1 一版关闭 33 个 issue/PR（`CHANGELOG.md` 末段），当前 open issues 仅 1（gh api）——作者对 4.0.0 发布做了大规模清库
- 发布机制：JitPack 构建（`jitpack.yml` 指定 JDK 21），无 Maven Central；通过 GitHub Sponsors 与 PayPal 接受赞助（`.github/FUNDING.yml`）

## 9. 亮点与风险

亮点：

1. 三层解耦的教科书架构：图表（charts）/数据（data）/渲染（renderer）分离，新增图表类型只需装配 chart + renderer + highlighter（`MPChartLib/Module.md`、`charts/LineChart.kt` 仅 54 行可证）
2. 性能设计到位：像素列降采样 PointReducer + 可开关 decimation，ObjectPool 复用触点对象（`renderer/PointReducer.kt`、`utils/ObjectPool.kt`），大序列另有 Douglas-Peucker 抽稀工具
3. 重写不换血：4.0 保留包名/类名/接口名，MIGRATION.md 把破坏点列成清单，旧项目迁移成本可控（`MIGRATION.md`）
4. Compose 一等模块：不是文档级兼容说明，而是独立 Gradle 模块、9 个 composable、可保存的 ChartState 与 ComposeMarker（`MPChartCompose/`）
5. 工程纪律罕见地好：explicitApi + ABI dump 校验 + 设备测试模块 + 全量 KDoc + Dokka 发布文档

风险/局限：

1. 单人维护，Bus factor = 1，且有过 7 年停摆前科；4.0 的长期迭代可持续性未知
2. 绘制内核仍是 View 体系命令式 Canvas/Paint，Compose 集成靠 AndroidView 桥接（`compose/ChartView.kt`），在纯 Compose 应用中状态同步与组合销毁语义弱于原生 Compose 实现
3. 生态分叉：4.0 换了发布坐标（`com.github.PhilJay.MPAndroidChart:MPChartLib`），海量存量项目仍锁定 3.x 旧坐标，社区代码/教程长期停留在 Java 3.x
4. 相比 Vico、Compose Charts 等 Compose 原生图表库，MPAndroidChart 的 Compose 层是包装而非原生绘制（此条为分析推断，README 未做对比）；3.x 停摆期间流失的用户能否回流待观察

## 10. 总结评价

MPAndroidChart 是 Android 图表领域事实上的教科书：十年 Java 时代的积累让它成为星数最高的 Android 图表库，而 2026 年的 4.0 Kotlin 重写——保持 API 骨架、补齐 Compose 模块、加入 ABI 校验与设备级测试——显示它不是靠惯性维持的遗产项目，而是被作者认真复活。它的架构（Chart/Renderer/DataSet/Highlighter 四层）至今值得任何做自绘控件的 Android 开发者精读。适合需要在 View 体系里快速获得 9 种可深度定制图表的业务项目，也适合想学习 Android Canvas 自绘、手势交互与性能优化（降采样、对象池）的人通读源码；若团队已全面转向 Compose 原生开发且无历史包袱，可同时评估 Vico 等原生方案。
