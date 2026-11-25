# android/architecture-samples 项目分析

> 星数 45,600（快照 2026-10-24）｜主语言 Kotlin｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

Android 官方架构示范仓库：用同一个 TODO 小应用演示不同年代、不同技术栈的 Android 应用架构，main 分支为 Jetpack Compose + ViewModel + Kotlin Flow 的 UDF（单向数据流）实现，历史架构（View 体系、LiveData、Dagger 等）保存在仓库分支中。

| 事实 | 值 | 出处 |
| --- | --- | --- |
| 仓库 | android/architecture-samples | gh api |
| 星数 | 45,889（实时 API，快照 45,600） | gh api repos/android/architecture-samples |
| Fork | 11,852｜Open issues 228 | 同上 |
| 主语言 | Kotlin（Kotlin 2.1.10，`gradle/libs.versions.toml`） | 源码 |
| 许可证 | Apache-2.0（`LICENSE`） | 源码 |
| 创建时间 | 2016-02-05（原 googlesamples/android-architecture，todo-mvp 系列祖先） | gh api |
| 默认分支 | main；master 为 main 的自动镜像副本（`.github/workflows/copy-branch.yml`） | 源码 |
| 最近活跃 | main 最新提交 2025-07-17（Add CODEOWNERS）；Renovate 机器人分支 renovate/all 更新至 2026-11-04 | gh api |
| Release | 无任何 release（gh api releases 为空） | gh api |
| 贡献者 | 60 人（gh api contributors per_page=100） | gh api |

## 2. 项目是什么，解决什么问题

这是 Google Android 团队维护的教学型仓库，回答一个持续多年的问题："Android 应用应该怎么组织架构"。它刻意不用大型业务应用，而是用一个足够简单、又能承载真实设计决策的 TODO 应用（任务列表、筛选、增改、详情、统计五类界面），把每个时代的"推荐架构"做成一个独立分支，供开发者对比学习（`README.md` "Why a to-do app?" 一节）。

main 分支展示的是当前官方推荐：Jetpack Compose 界面 + 单 Activity + Navigation Compose、每屏一个 ViewModel、Kotlin Flow/协程驱动的响应式 UI、Repository + 双数据源（Room 本地为唯一真源 + 模拟远端）、Hilt 依赖注入（`README.md` 开头列表）。这套结构就是 Android 官方架构指南（UDF 分层）的可运行最小样本。

仓库同时是 Android 架构演进史的时间线：`views` 分支是 View 体系 + ViewModel + LiveData + Data Binding 的"Blueprints v2"版本（其 README 仍引用旧 googlesamples wiki）；`livedata` 分支是 Compose + LiveData 变体；另有 `hilt`、`dev-dagger`、`service-locator`、`multimodule`、`coroutine-experiments` 等约 31 个分支（gh api branches 列表）。历史上著名的 todo-mvp、todo-mvvm-live 系列分支已被重命名/精简，早期 MVP/MVVM 对比需查 git 历史或旧 fork，tarball 快照中不可见（未确认细节）。

README 明确了边界：它不是项目模板（模板是 android/architecture-templates）、不是 UI 示范（看 compose-samples）、不是生产级应用（看 nowinandroid）。它只为"如何分层、如何可测试"提供参考（`README.md` "What is it not?"）。

## 3. 技术栈与依赖

| 类别 | 内容 | 出处 |
| --- | --- | --- |
| 语言/平台 | Kotlin 2.1.10，Android SDK（compileSdk/targetSdk 35，minSdk 21） | `gradle/libs.versions.toml` |
| UI | Jetpack Compose（BOM 2024.12.01，Material 3）+ Navigation Compose + Accompanist（appcompat 主题、swiperefresh） | `gradle/libs.versions.toml`、`app/build.gradle.kts` |
| 架构组件 | Lifecycle ViewModel Compose、Room 2.6.1（KSP 编译）、SavedStateHandle | 同上 |
| 异步 | Kotlin 协程 + Flow（kotlinx-coroutines 1.10.1） | 同上 |
| 依赖注入 | Hilt 2.53.1（+ androidx.hilt navigation compose） | 同上 |
| 日志 | Timber | `app/build.gradle.kts` dependencies |
| 测试 | JUnit4、Truth、Turbine（Flow 测试）、Mockito、Espresso、androidx test、compose ui test、Jacoco 0.8.7 | `gradle/libs.versions.toml`、`app/src/test/resources/mockito-extensions` |
| 构建 | Gradle 8.x + AGP 8.7.3，KSP，version catalog 统一版本，Spotless（版权头模板在 `spotless/`），Renovate 每日依赖自动化（`renovate.json`） | 源码 |

依赖特点：面向网络/图片的库（okhttp、retrofit、coil）虽在 version catalog 中声明版本，但 app 实际未使用——远端数据源是纯内存 fake（见第 5 节）。

## 4. 目录结构与代码组织

顶层：

| 目录/文件 | 职责 |
| --- | --- |
| `app/` | 唯一的 Android 应用模块（全部业务代码 + 测试） |
| `shared-test/` | 独立 Gradle 模块：单元测试与仪器化测试共享的 fake 与规则 |
| `gradle/` | version catalog（`libs.versions.toml`）、wrapper、`init.gradle.kts` |
| `spotless/` | Spotless 版权头模板（kt/kts/xml） |
| `.github/workflows/` | CI：`build_test.yaml`（构建+测试）、`copy-branch.yml`（main→master 镜像） |
| `screenshots/`、`README.md`、`LICENSE`、`CONTRIBUTING.md`、`CODEOWNERS`、`renovate.json` | 文档与配置 |

`app/src/main` 包结构（按 feature 分包，共 33 个 Kotlin 文件、约 3200 行）：

| 包 | 内容 |
| --- | --- |
| 根包 | `TodoActivity`（单 Activity）、`TodoApplication`、`TodoNavGraph`、`TodoNavigation`、`TodoTheme` |
| `tasks/`、`taskdetail/`、`addedittask/`、`statistics/` | 四个功能屏，每屏一个 `XxxScreen.kt`（Compose UI）+ `XxxViewModel.kt` |
| `data/` | `Task` 领域模型、`TaskRepository` 接口、`DefaultTaskRepository` 实现、模型映射 `ModelMappingExt` |
| `data/source/local/` | Room：`ToDoDatabase`、`TaskDao`、`LocalTask` 实体 |
| `data/source/network/` | `NetworkDataSource` 接口 + `TaskNetworkDataSource` 内存 fake |
| `di/` | Hilt 模块：`DataModules.kt`、`CoroutinesModule.kt` |
| `util/` | `Async`（Loading/Error/Success 封装）、`WhileUiSubscribed` 等协程/Compose 工具、IdlingResource |

组织方式评述：典型单模块 + feature 分包教学结构。刻意不分多模块（多模块形态被做成独立的 `multimodule` 分支），以保证初学者可一次读懂；`shared-test` 是唯一拆出的模块，用于演示"一套 fake 同时服务 JVM 测试与仪器化测试"的工程实践（`settings.gradle.kts`、`shared-test/src/main/java/.../FakeTaskRepository.kt`）。

## 5. 核心架构与关键实现

### 5.1 数据层：Room 为真源 + 模拟远端的一次性同步

`data/DefaultTaskRepository.kt` 是唯一数据入口。所有写操作先落 Room，再异步推送到"网络"；读操作走 Flow 响应式流：

```kotlin
override fun getTasksStream(): Flow<List<Task>> {
    return localDataSource.observeAll().map { tasks ->
        withContext(dispatcher) { tasks.toExternal() }
    }
}
```

同步策略是刻意的简化：`refresh()` 直接"删光本地、全量灌入远端"（`DefaultTaskRepository.kt` refresh/saveTasksToNetwork），注释明说真实应用应参考官方 offline-first 文档做增量同步；`saveTasksToNetwork` 在 `@ApplicationScope` 作用域里 fire-and-forget，异常只留注释不处理。这是教学取舍的典型示范——每个简化点都有注释指出生产环境的正确做法。

远端数据源 `data/source/network/TaskNetworkDataSource.kt` 是硬编码两条任务（比萨塔、塔科马桥）的内存 fake，用 Mutex 保证线程安全并 `delay(2000)` 模拟网络延迟，让 UI 层的加载态、IdlingResource 测试等待都有真实场景可演。

### 5.2 表现层：单 StateFlow 的 UDF ViewModel

每个 ViewModel 对外只暴露一个 `StateFlow<XxxUiState>`，由多个上游 Flow `combine` 后 `stateIn`。以 `tasks/TasksViewModel.kt` 为例：

```kotlin
val uiState: StateFlow<TasksUiState> = combine(
    _filterUiInfo, _isLoading, _userMessage, _filteredTasksAsync
) { filterUiInfo, isLoading, userMessage, tasksAsync -> ... }
    .stateIn(
        scope = viewModelScope,
        started = WhileUiSubscribed,
        initialValue = TasksUiState(isLoading = true)
    )
```

关键设计（均在源码可核）：筛选类型存入 `SavedStateHandle` 持久化并在进程重建后恢复（`TasksViewModel.kt` `TASKS_FILTER_SAVED_STATE_KEY`）；加载失败通过 `util/Async.kt` 的 `Loading/Error/Success` sealed class 包裹后并入 UI 状态，错误以 snackbar 资源 id 呈现、消费后置空（`snackbarMessageShown()`）——即完整的事件-状态分离 UDF 模式。`statistics/StatisticsViewModel.kt` 展示更精简的版本：一条 `getTasksStream().map{...}.stateIn` 直接算出活跃/完成百分比。

### 5.3 依赖注入与协程约定

`di/DataModules.kt` 用 Hilt `@Binds` 把 `DefaultTaskRepository→TaskRepository`、`TaskNetworkDataSource→NetworkDataSource`、Room 数据库/DAO 装配进 SingletonComponent；`di/CoroutinesModule.kt` 提供 `@DefaultDispatcher`、`@ApplicationScope` 等 qualifier，让 Repository 的调度器与全局作用域可注入、可在测试中替换——这是该样本反复强调的"可测试性优先"设计。

### 5.4 导航与单 Activity

`TodoActivity` + `TodoNavGraph`/`TodoNavigation` 用 Navigation Compose 定义 tasks/taskdetail/addedit/statistics 四个目的地，增改屏通过保存态返回结果（`ADD_EDIT_RESULT_OK` 等常量在 `TasksViewModel.kt` 中被消费为 snackbar 消息），演示 Compose 时代的导航与屏幕间通信范式。

## 6. 构建与运行

来自 README 与工程文件（本分析未执行构建）：

```bash
git clone https://github.com/android/architecture-samples.git
# Android Studio 打开根目录，或命令行：
./gradlew build                     # 构建
./gradlew testDebugUnitTest         # JVM 单元测试（app/build.gradle.kts testOptions）
./gradlew connectedAndroidTest      # 仪器化测试（需模拟器/设备，CustomTestRunner）
```

- 要求 JDK 17（CI 用 zulu 17：`.github/workflows/build_test.yaml`）、Android SDK 35。
- CI 实际命令（build_test.yaml）：`./gradlew build` 后跑 `./gradlew :app:connectedDebugAndroidTest`，模拟器 API level 29 + KVM 加速，30 分钟超时。

## 7. 测试与工程质量

- **测试规模**：单元测试 6 个文件（覆盖全部 ViewModel + Repository + 工具类，`app/src/test/`）、仪器化测试 7 个文件（4 个 Compose 屏幕 UI 测试、`AppNavigationTest`、Room `TaskDaoTest`，`app/src/androidTest/`）、共享测试设施 7 个文件（`shared-test/` 的 FakeTaskRepository/FakeTaskDao/FakeNetworkDataSource、MainCoroutineRule、CustomTestRunner、测试用 Hilt 模块）。对一个 33 文件的业务代码库，测试文件数（20）超过业务文件数，是该项目最核心的卖点：它演示"每层怎么测"。
- **测试风格**：ViewModel 用 fake Repository + 协程测试规则（不用 mock 领域对象）；Repository/DAO 用仪器化真库测试；屏幕层用 Compose 测试 + IdlingResource 等待（`util/SimpleCountingIdlingResource.kt`）。"shared tests"（同一套测试跑 JVM 与设备）是该仓库首创并被官方文档推广的实践。
- **CI**：两条 workflow——build_test（push/PR to main，单元+仪器化）与 copy-branch（push main 后自动把 main 复制到 master 保留旧链接）（`.github/workflows/`）。
- **代码质量**：全库统一版权头由 Spotless 管理（`spotless/`）；依赖版本全部走 version catalog 且由 Renovate 每日自动升级（`renovate.json`，分支 renovate/all 最新提交 2026-11-04）。代码注释密度高，几乎每个简化处都标注了生产级替代方案。
- **README 漂移**：README 仍声称"Two product flavors, mock and prod"，但 `app/build.gradle.kts` 已无 `productFlavors` 定义（当前远端 fake 在所有构建中直接生效）——文档滞后于代码，未确认 flavors 在哪次重构中移除。

## 8. 社区与维护状态

- **人类维护低频、机器维护高频**：main 最新提交为 2025-07-17（Add CODEOWNERS），2024-12-11 批量升级依赖，2025-04 加 workflow_dispatch；但 Renovate 分支持续有 2026-10/11 的依赖升级提交（gh api commits per branch），pushed_at 2026-11-03 主要由机器人/fork 活动贡献。即代码架构已稳定不再演进，仅依赖保持新鲜。
- **无 release**：教学仓库不发布版本，"版本"即分支/提交（gh api releases 为空）。
- **贡献者 60 人**（gh api contributors，匿名不计），以 Google/Android 团队成员为主（`CODEOWNERS` 存在），外部贡献需签 Google CLA（`CONTRIBUTING.md`）。
- **社区影响**：45.9k 星、11.8k fork；issue 积压 228 个（长期教学仓库常见，未确认处理策略）。官方生态位：与 architecture-templates（模板）、nowinandroid（生产级参考）三件套分工明确（`README.md`）。

## 9. 亮点与风险

**亮点**

1. 官方出品且随时代演进：从 2016 年 MVP 分支到 main 的 Compose UDF，一份代码承载 Android 架构十年演变，分支即教材目录。
2. 测试示范价值极高：单元/仪器化/共享测试三层齐全，测试文件多于业务文件，`shared-test` 模块的 fake 复用是可直接搬走的工程实践。
3. 教学注释质量好：`DefaultTaskRepository.kt` 等文件对每处简化（one-way sync、fire-and-forget 上传）都注明生产环境的正确路径，不会误导初学者。
4. 工程配置现代化：version catalog + KSP + Hilt + Spotless + Renovate + GitHub Actions 模拟器 CI（KVM 加速），配置本身即可当模板抄。
5. 依赖常年保鲜：Renovate 每日调度，2026 年仍在自动升级（renovate/all 分支 2026-11-04 提交）。

**风险/局限**

1. README 与代码有漂移：mock/prod flavor 声明已与 main 的 build 配置不符，照 README 理解会踩空。
2. 无真实网络层：远端是内存 fake，不能当"网络层怎么写"的参考（官方另指 nowinandroid）。
3. 功能刻意单薄：无认证、无真实同步、无多模块，复杂度天花板低，只能当架构入门样本。
4. 维护进入静默期：架构代码一年以上无实质演进，新 API（如 Compose 新范式）不会及时跟进；历史 MVP/MVVM 分支已被清理合并，早期对比材料需另寻旧 fork。

## 10. 总结评价

android/architecture-samples 是 Android 官方架构教学的"活化石 + 现行推荐"二合一仓库：main 分支是一个干净、可测试性拉满的 Compose UDF 最小实现，分支群则是架构演进史的索引。它的价值不在代码量（约 3200 行业务代码）而在"每一步为什么这样做"的注释与三层测试示范。适合 Android 中级开发者初次学习官方推荐分层、以及需要快速对照 UDF/Repository/Hilt 测试写法的资深开发者；想找可直接开工的工程模板或生产级网络/同步实现，应转投官方 architecture-templates 与 nowinandroid。
