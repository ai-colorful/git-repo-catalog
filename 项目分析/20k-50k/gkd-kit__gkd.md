# gkd-kit/gkd 项目分析

> 星数 33400（快照 2026-10-24）｜主语言 Kotlin（GitHub API 口径；任务参数标 TypeScript，实际 TS 仅占工具链，见 §3）｜档位 20k-50k｜分析日期 2026-11-06

## 1. 项目速览

一句话定位：基于 Android 无障碍服务 + CSS 风格高级选择器 + 订阅规则协议的开源屏幕自动化应用，典型用途是在指定界面出现指定节点（如广告弹窗）时自动点击跳过。

| 项 | 值 |
| --- | --- |
| 名称/全名 | gkd / gkd-kit/gkd |
| 星数 | 33400（快照） |
| 主语言 | Kotlin（Kotlin Multiplatform），TypeScript 为开发工具链 |
| 许可证 | GPL-3.0-only（`LICENSE`、`README.md`） |
| 创建时间 | 2023-07-04（GitHub API） |
| 最近推送 | 2026-11-05（GitHub API，分析前一天） |
| 最新版本 | v1.13.0-beta-7（2026-11-03，GitHub Releases；源码内版本号一致，`build.gradle.kts:30`） |
| 应用包名 | li.songe.gkd（`gkd-android/build.gradle.kts:25`） |
| Fork/贡献者 | 2103 forks / 11 contributors（GitHub API） |
| 官网 | https://gkd.li |

## 2. 项目是什么，解决什么问题

GKD（gkd.li）是一个 Android 端自定义屏幕点击应用。工作方式：无障碍服务持续接收窗口事件，当"当前界面满足规则条件（如存在特定文字/控件）"时，对匹配到的节点或坐标执行点击、长按、滑动、返回等动作（`README.md`、`docs/architecture.md`）。官方给出的两类场景是"快捷操作"（简化重复流程）与"跳过流程"（跳过应用启动时的烦人流程，如开屏广告的跳过按钮）。

设计上最关键的一条：**应用本体不内置任何规则**。"GKD 默认不提供规则，需自行添加本地规则，或者通过订阅链接的方式获取远程规则"（`README.md`）。规则以版本化的 JSON5 订阅文件形式分发，任何人可以用官方模板仓库 subscription-template 搭建自己的订阅；第三方订阅通过 GitHub Topics `gkd-subscription` 聚合（`README.md`）。这种"引擎与规则彻底分离"的形态，把规则维护的压力和责任都转移到了社区生态，引擎本身保持中立工具属性。

配套工程生态完整：网页端快照审查工具（gkd-kit/inspect，分享链接形如 `i.gkd.li/i/...`）、`@gkd-kit/selector` npm 包（选择器引擎编译到 JS 供网页/Node 使用）、gkd.li 文档站（`README.md` 衍生项目清单、`gkd-selector/README.md`）。App 内还提供 ExposeService 供第三方应用主动触发快照采集（`gkd-android/src/main/AndroidManifest.xml`）。分发渠道客观陈述：并非仅靠 GitHub——README 安装区并列 gkd.li、Google Play（li.songe.gkd）与 GitHub Releases 三个入口，应用内另有自更新检测；多渠道由 Gradle channel flavor 区分，AGENTS.md 规定日常验证"默认仅 gkd 渠道，未经明确要求禁止 play"。

工程上它同时是一个罕见的 Kotlin Multiplatform 实践样本：同一套 commonMain 业务/UI/ViewModel 同时驱动 Android 真机与 JVM Desktop 开发宿主，Desktop 端用模拟输入替代真实自动化，"不支持的具体动作返回 PlatformResult.Unsupported"（`docs/architecture.md`）。

## 3. 技术栈与依赖

| 层 | 技术 | 出处 |
| --- | --- | --- |
| 语言 | Kotlin 2.4.21（KMP：Android + JVM + JS） | `gradle/libs.versions.toml`、`gkd-selector/build.gradle.kts` |
| UI | Jetpack Compose Multiplatform 1.12.2（Material3）、Navigation3 | `gradle/libs.versions.toml`；无 React Native，任务描述中 LANG=TypeScript 与实际不符 |
| 构建 | AGP 9.5.0、Gradle KTS、buildSrc 自定义插件、版本目录 | `settings.gradle.kts`、`buildSrc/` |
| 持久化 | Room 3.0.3（room3 包）、DataStore 式设置仓库 | `gradle/libs.versions.toml`、`gkd-db/` |
| 网络 | Ktor 3.6.1（client-okhttp 拉订阅 + server-CIO 本地调试 HTTP） | `gradle/libs.versions.toml` |
| 序列化 | kotlinx-serialization + 自研 kotlin-json5（订阅 JSON5） | `libs.versions.toml`（json5 = li.songe:json5） |
| 特权能力 | Shizuku 13.1.5、LSPosed HiddenApiBypass 6.2、priv-kit、自研 remap 隐藏 API 重映射 | `libs.versions.toml`、`gkd-hidden-api/` |
| 表达式 | li.songe:expression 0.1.2（坐标表达式沙箱）、exp4j 仅测试对照 | `libs.versions.toml`、`docs/rule-expressions.md` |
| JS 工具链 | TypeScript 7、pnpm workspace、Node --test；脚本约 5.8k 行 | `package.json`、`scripts/desktop/` |

规模：全仓库约 3.8 万行 Kotlin（1527 个 tarball 条目），其中 gkd-app 867 个 .kt 文件为主体；gkd-selector 56 个文件；测试文件 131 个。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `gkd-android/` | 纯打包壳：Manifest、原生 res、渠道 flavor、签名、输出命名（无 Kotlin 源码） |
| `gkd-app/` | 应用主体，KMP 三源码集：commonMain（业务/UI/VM/导航）、androidMain（Service/权限/无障碍/特权）、jvmMain（Desktop 宿主） |
| `gkd-db/` | Room 实体、DAO、分页与事务 |
| `gkd-selector/` | 选择器引擎（纯 KMP 库，jvm+js 双目标，发布 npm） |
| `gkd-aidl/` | Binder 协议 AIDL（特权 UserService、Shizuku 启动、表面监听等 4 个接口） |
| `gkd-hidden-api/` | Android 隐藏 API 编译期 stub（IAccessibilityServiceClient、IUiAutomationConnection 等），经 remap 插件重映射字节码 |
| `gkd-a11y-fixture/` | 无障碍测试夹具应用 |
| `scripts/desktop/` | TypeScript 开发工具集（adb 安装、模拟控制、HTTP 调试客户端、表达式语料采集等，含 node --test 测试） |
| `docs/` | architecture / local-development / matching-engine-debugging / matching-regression / rule-expressions 五篇工程文档 |
| `.github/` | build / release / publish-selector 三个 workflow，FUNDING 与 issue 模板 |
| `.agents/skills/` | 面向 AI 协作开发的 android-api-diff、android-hidden-api-r8 两个 skill（SKILL.md + 参考） |

组织方式评述：单仓库多 Gradle 模块 + 单个 pnpm 子包（`pnpm-workspace.yaml` 仅含 gkd-selector），依赖方向严格单向 `gkd-android → gkd-app → gkd-db / gkd-selector`（`AGENTS.md` 明文禁止反向依赖）。gkd-app 内部再按职责分包（rule / matching{.input,.query,.action,.diagnostics} / subscription / a11y / priv / ui 等 20+ 包，见 `gkd-app/src/commonMain/kotlin/li/gkd/app/`），且 `AGENTS.md`（约 90 行、非常细的开发约束）与 `docs/architecture.md` 构成一套罕见的"文档即治理"体系——包名、路由序列化名、AIDL、数据库结构均被定义为不可破坏的兼容契约。

## 5. 核心架构与关键实现

### 5.1 无障碍服务层：三入口 + 身份伪装

Manifest 注册三个无障碍服务，类名全部落在 `com.google.android.accessibility.selecttospeak.*` 命名空间下（standard / privileged / companion 三个变体，配置分别为 `a11y_info_standard|privileged|companion.xml`）。AGENTS.md 明文要求"已注册组件（含 SelectToSpeakService）保留原身份"、包名整理不得改变组件身份。客观陈述该事实：服务对外呈现与 Google 系统应用"随选朗读"（Select to Speak）同名的类名与 label，与部分应用针对自动化工具包名/服务名的检测形成对抗；`a11y_info_standard.xml` 中 `isAccessibilityTool` 指向按渠道切换的 bool 资源。标准/特权实例的选择逻辑见 `A11yService.kt:287-294`——持有 Shizuku 特权上下文且具备输入注入权限时优先用 privileged 变体。Manifest 中另有整套前台服务与快捷磁贴：StatusService/DebugService（局域网 HTTP 调试）/ButtonService/ActivityService/EventService/TrackService/ExposeService/MatchDiagnosticService 八个 specialUse 或 mediaProjection 前台服务，以及开关、快照、匹配诊断等 7 个 QuickSettings 磁贴服务；`gkd://` deep link（OpenSchemeActivity）承担订阅导入与页面跳转入口（`gkd-android/src/main/AndroidManifest.xml`）。服务本体是抽象类 `li.gkd.app.service.A11yService`：事件回调直通 `matchHost.onA11yEvent`，API R+ 走 `AccessibilityService.takeScreenshot` 原生截图（`A11yService.kt:62-93`），并用 `TYPE_ACCESSIBILITY_OVERLAY` 挂保活覆盖层（`A11yService.kt:197-230`）。

### 5.2 匹配运行时：协调器 + 调度状态机 + 共享规则遍历

匹配链路按职责分包（`docs/architecture.md` 的包职责表）：`MatchingRuntime.coordinator` 作为应用生命周期单例，统一自动规则、业务 HTTP 与单次诊断三类输入通道；`MatchCoordinator`（`gkd-app/src/commonMain/kotlin/li/gkd/app/matching/MatchCoordinator.kt`）是调度状态机 `MatchScheduleMachine` 的协程宿主，以 WeakHashMap 身份表管理任务/规则/快照描述符，服务交接时取消旧上下文。规则遍历的核心是平台无关的 `queryRules` 泛型函数（`matching/query/RuleQuery.kt:26-142`）：对前台快照的 `priorityRules` 逐条检查规则状态（Ready/MatchDelay…）→ 强制窗口有效期 → 前台应用一致性（foreground_conflict 即中止）→ 选择器匹配 → 动作延迟 → 执行动作，全程通过 `work.trace(MatchStage.*)` 记录诊断轨迹，N 为原始节点类型（Android 的 AccessibilityNodeInfo 或桌面模拟节点），同一套遍历同时服务于真机与模拟。

### 5.3 选择器引擎：CSS 风格语法的完整编译管线

`gkd-selector`（发布为 npm 包 `@gkd-kit/selector` 0.6.1）内部按运行时职责分四个子包（`gkd-selector/README.md` 的 mermaid 图）：`syntax`（Tokenizer/Parser/位置记录/打印）、`property`（属性表达式、内建成员、正则、类型检查）、`relation`（树关系算子 `<` `>` `-`）、`engine`（编译与匹配执行）。公开 API 三入口 `Selector.compile / parse / tokenize` 分别面向纯匹配、带源码位置的诊断/高亮、纯词法着色；匹配支持 `matchWithTrace` 返回成功路径。两个值得注意的工程设计：一是**快照契约**——每次匹配观测一棵稳定节点树，宿主实现 `NodeAdapter.getNodeKey` 提供稳定等价键，引擎据此在回溯时记忆化失败的搜索状态（`gkd-selector/README.md` "Query snapshot contract"）；二是**快速查询钩子**——`FastQuery`（`gkd-selector/src/commonMain/kotlin/li/gkd/selector/FastQuery.kt`）从选择器中提取 id/vid/text 三类候选过滤器，宿主可用 `getFastQueryDescendants` 跳过全树遍历，允许假阳性但不允许漏报。JS 目标编译到 ES2015 ESM 并生成 TypeScript 定义，依赖宿主支持 WebAssembly GC（正则经 regex-wasm 实现 JVM/JS 行为一致，`gkd-selector/README.md`）。

### 5.4 订阅协议与规则数据模型

订阅是版本化 JSON5 文档，数据模型集中在 `subscription/RawSubscription.kt`：顶层 `id/name/version/author/updateUrl/checkUpdateUrl + categories + globalGroups + apps[]`，规则组内每条规则携带 `matches / anyMatches / excludeMatches / excludeAllMatches` 四组选择器、`action`（点击/长按/返回等）、`actionCd / actionDelay / matchDelay / matchTime / actionMaximum / resetMatch / priorityTime / forcedTime` 等节流与限频属性，以及 `snapshotUrls`（关联 i.gkd.li 快照示例，供审查）。点击/滑动坐标支持表达式（`Position`，`RawSubscription.kt:183-206`）：由 `li.songe:expression` 编译，变量白名单仅 left/top/right/bottom/width/height/screenWidth/screenHeight/random，**所有函数调用被拒绝**，静态校验在动作注入前完成（`docs/rule-expressions.md`）。订阅的拉取、解析、原子写文件与数据库补偿由 `SubscriptionRepository` 协调，规则快照安装经 `RuleReconciliation` 保留未受影响的启用状态（`docs/architecture.md` 数据读写表）。特权链路独立成套：`gkd-aidl` 的 `IUserService`/`IPrivilegeShizukuStartService` 等 4 个 Binder 接口编译进 `gkd-app/androidMain`，`priv/` 包内 PrivilegeContext、ProxyUiAutomationConnection、HiddenApiCast 与 Manifest 里的 ShizukuProvider 组合成"免 root 提权"路径，特权进程的 Binder 方法要求最外层捕获 LinkageError 防止崩溃逃逸（`AGENTS.md` 依赖与 Android 专项章节）。

## 6. 构建与运行

来自工程文件与 AGENTS.md/README 的实际命令：

```bash
# 桌面端（开发宿主，JVM）
./gradlew.bat :gkd-app:run

# Android 构建（渠道 flavor：默认 gkd 渠道；CI 用 GKD_RENAME_PACKAGE_FLAG 改包名）
./gradlew :gkd-android:assembleGkdRelease

# 选择器 npm 包：本地构建 / 拉取已发布 dist
pnpm --dir gkd-selector build
pnpm fetch-selector-dist

# TS 开发工具与测试
pnpm app-tools
pnpm type-check
node --test ./scripts/desktop/lib/*.test.ts
```

npm 发布走 GitHub Trusted Publishing（`.github/workflows/publish-selector.yml`），tag 必须等于 `name@version`，prepack 钩子强制 JVM/JS 测试、构建与类型检查全绿。Android CI（`.github/workflows/build.yml`）用 base64 密钥注入签名。Desktop 宿主自带模拟器与调试 HTTP（绑定 0.0.0.0 供局域网访问，页面展示真实 IPv4；诊断 HTTP 用短期凭据，Release 构建不装配诊断路由），本地数据固定 `.local/desktop/`，测试会话隔离在 `.local/tests/desktop/<会话ID>/`（`AGENTS.md`、`docs/local-development.md`）。

## 7. 测试与工程质量

测试文件 131 个，覆盖四个源码集：`gkd-app/src/commonTest`、`jvmTest`、`androidHostTest`、`gkd-selector/src/commonTest`，加上 `scripts/desktop` 的 Node 测试（android-apk、device-install、desktop-client 等约 15 个 .test.ts）。风格上偏状态机/协议级回归：`docs/matching-regression.md` 定义了匹配引擎的 JVM 场景回归边界；`docs/rule-expressions.md` 提到 `RuleExpressionCompatibilityTest` 用从真实订阅脱敏出的表达式夹具 + 固定随机种子算式做兼容性对照并报告耗时/分配量；Robolectric 覆盖 API 26/33 双版本。CI 仅 build.yml（构建产物）与 publish-selector.yml（发布门禁），未见独立测试 workflow（未确认是否合并在 prepack 链路内执行）。代码质量观察：AGENTS.md 对防御式编程有明确反约束（"不为假设中的异常新增兜底"、捕获必须有实际处理边界），架构文档与实现高度同步，`explicitApi()` 用于 selector 模块——整体工程纪律在同类个人项目中属于第一梯队。

## 8. 社区与维护状态

- 活跃度：创建于 2023-07，分析时最近推送为前一天（2026-11-05，GitHub API）；近三周连发 v1.13.0-beta-2 至 beta-7 五个版本，处于活跃开发期（GitHub Releases）。
- 贡献者：11 人（GitHub API contributors），主体为单人主导（作者 lisonge，赞助入口 `lisonge/sponsor`），fork 2103、open issues 仅 23——issue 大量依赖模板（`.github/ISSUE_TEMPLATE/bug_report.yml` 等）并随发布快速关闭（数字为快照，趋势未确认）。
- 发布节奏：应用本身走 GitHub Releases + Google Play + gkd.li 三渠道（`README.md` 安装区），应用内"设置-关于-检测更新"自更新（`CHANGELOG.md`）；选择器引擎独立走 npm 版本线（当前 0.6.1）。
- 社区生态的承接方式：规则生态不进本仓库，而以 GitHub Topics `gkd-subscription` 与 subscription-template 模板组织；快照审查依赖姊妹仓库 gkd-kit/inspect。这种组织让主仓库贡献者少但生态参与面大。
- 仓库内还维护了面向 AI 协作开发的 skills（`.agents/skills/android-api-diff/`、`.agents/skills/android-hidden-api-r8/`，含 agents/openai.yaml 清单与失败案例参考），并与根 `AGENTS.md` 的开发约束互相引用——把隐藏 API 定位、R8 验证等高危操作固化成标准流程。

## 9. 亮点与风险

亮点：

1. 引擎/规则/审查工具三件套分离且各自开源，选择器引擎同时编译到 JVM 与 JS，让网页端审查工具与应用共享同一套匹配语义（`gkd-selector/build.gradle.kts`、gkd-kit/inspect）。
2. 匹配链路的可观测性做得罕见地彻底：全链路 trace 分级（Rule/Root/Selector/Action）、诊断会话、节点指标、查询录制，配套 `matching-engine-debugging.md` 文档（`matching/` 包、docs/）。
3. KMP 单源双端：commonMain 同时驱动 Android 与 Desktop 开发宿主，Desktop 用模拟输入 + 模拟系统栏做 UI/规则调试而不碰真实自动化（`docs/architecture.md`、`AGENTS.md` Desktop 运行时章节）。
4. 安全边界有真实设计而非口号：坐标表达式静态白名单校验拒绝一切函数调用、Depth 上限、NaN/Infinity 拒绝（`docs/rule-expressions.md`）；选择器有类型校验模型（`Selector.validateType`）。
5. 工程治理文档化程度极高（AGENTS.md + 5 篇 docs/），兼容契约（路由名/AIDL/DB 结构）显式列出。

风险/局限：

1. 无障碍自动化本质上是与目标应用 UI 的持续博弈：引擎开源但规则由第三方订阅分发，规则质量、时效与订阅源可信度完全取决于用户选择，应用默认不带规则也意味着开箱即用性为零（`README.md` 订阅章节）。
2. 构建链复杂度高：KMP + 隐藏 API stub + 自研 remap 字节码重映射 + 渠道 flavor + R8 专项验证（AGENTS.md 依赖与 Android 专项章节），外部贡献者门槛显著，实际贡献高度集中于作者一人。
3. 服务类名与 Google 系统组件同名、多个 specialUse 前台服务、QUERY_ALL_PACKAGES/WRITE_SECURE_SETTINGS/DUMP 等权限的组合，在应用商店政策与用户审计两方面都属于持续存在的合规敏感点（`AndroidManifest.xml`）；isAccessibilityTool 按渠道取值也说明作者对此有明确意识。
4. Desktop 端仅是开发宿主，不能替代真机验收（`AGENTS.md` 明文要求 Desktop 不做像素验收替代）；对只想"用"的用户，理解订阅/选择器概念的学习曲线偏陡。

## 10. 总结评价

GKD 是"无障碍自动化"这一赛道的工程化巅峰样本：它把民间跳广告工具常见的"坐标盲点 + 内置规则"模式，重构为可编译、可校验、可审查、可分发的选择器引擎与版本化订阅协议，并用一套极其克制的工程纪律（单向模块依赖、文档化契约、最小实现原则）把一个单人项目做出了平台级代码组织的质感。它不适合想要开箱即用的普通用户——规则生态在仓库之外，需要理解选择器语法并自行选择订阅源；但它非常适合三类读者：想研究 Android 无障碍服务高阶玩法（特权进程、隐藏 API、UiAutomation、截图与手势注入）的客户端工程师，想学习 KMP + Compose Multiplatform 大型落地工程实践的团队，以及想借鉴"引擎与规则生态分离"这种社区运营结构的工具类开源作者。与坐标式自动点击工具相比，其节点级匹配更精准也更脆弱；与闭源同类跳广告工具（如李跳跳，此对比基于公开常识、仓库内无对应证据）相比，GKD 的引擎开源、规则与主体分离、默认不内置规则是其最显著的结构性差异，也是它能在 GitHub 而非仅靠侧载渠道积累 3 万+星信任资产的核心原因。
