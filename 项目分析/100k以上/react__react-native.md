# react/react-native 项目分析

> 星数 126821（快照 2026-10-24）｜主语言 C++｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：Meta 出品的跨平台原生应用框架——用 React 写 UI，由 C++ 核心引擎驱动，最终渲染为 iOS/Android 的真正原生视图。

| 项目 | 事实 |
| --- | --- |
| 名称 | react/react-native（npm 包 `react-native`） |
| 星数 | 126,821（gh api，2026-10-30 实测；任务快照 126,769） |
| 主语言 | C++（GitHub 统计），另有大量 JavaScript(Flow)、Kotlin、Objective-C++、Swift |
| 许可证 | MIT（`LICENSE`） |
| 创建时间 | 2015-01-09（gh api） |
| 最近推送 | 2026-10-29（gh api，几乎每日活跃） |
| 当前版本 | v0.88.1（`CHANGELOG.md` 顶部，发布于 2026-10-27） |
| 仓库形态 | Yarn workspaces monorepo（`package.json` 的 `workspaces` 字段） |

## 2. 项目是什么，解决什么问题

React Native 解决的问题是：移动开发者长期面临"一套代码跨平台"与"原生体验"之间的两难——纯 Web 方案（WebView/H5）交互与性能达不到原生水准，而双端各写一套原生代码成本翻倍。React Native 的答案是 `README.md` 里的口号 **"Learn once, write anywhere"**：业务代码用 React（JSX/Hooks/Suspense）声明式地描述 UI，框架在原生侧用各平台的真实 View 渲染，手势、无障碍、字体缩放等行为与原生应用一致（`README.md`）。

与 2015 年初版相比，当前架构（所谓"新架构"，New Architecture，现已默认）有三块基石：**JSI**（JavaScript Interface，C++ 抽象层，让 JS 引擎与原生代码直接持有彼此引用、同步互调，取代旧的 JSON 序列化 Bridge）、**Fabric**（C++ 实现的渲染器，把 React 组件树 diff 成"挂载指令"应用到原生视图）、**TurboModules**（按需加载、类型安全的原生模块系统）。核心代码几乎全部下沉到跨平台 C++（`ReactCommon/`），iOS/Android 只保留薄薄的平台层，这使 Windows/macOS 等 out-of-tree 平台也能复用同一核心。

开发体验方面，Fast Refresh 秒级热更新（改动 JS 无需重编译原生工程，`README.md`），配合 Metro 打包器与 `dev-middleware`（`packages/dev-middleware/`）形成完整的调试链路；新一代调试器 React Native DevTools 基于 Chrome DevTools Protocol，其代理实现就在仓库内（`packages/react-native/ReactCommon/jsinspector-modern/`）。

React Native 已是事实上的跨端基础设施：由 React Foundation 背书（`README.md`），Microsoft（Windows/macOS）、Shopify、Exp（Expo）等公司深度参与核心贡献，Expo 生态直接构建其上。

## 3. 技术栈与依赖

- **C++17（核心层）**：`ReactCommon/` 全部跨平台逻辑，含自研布局引擎 **Yoga**（Flexbox 实现，`ReactCommon/yoga/`）；构建用 CMake（各子目录 `CMakeLists.txt`）+ CocoaPods podspec + SwiftPM 三轨并行。
- **JavaScript（Flow 类型）**：组件与 API 层（`Libraries/`、`src/private/`），静态类型用 Flow 而非 TS，另发布官方 TS 类型 `ReactNativeApi.d.ts`（`packages/react-native/`）。
- **Android 平台层**：Kotlin/Java + JNI（`ReactAndroid/src/main/java/`、`ReactAndroid/src/main/jni/`），Gradle 构建（Kotlin DSL `build.gradle.kts`）。
- **Apple 平台层**：Objective-C++ 为主（`React/`），新引入 Swift/SwiftPM 支持（`ReactApple/`、`Package.swift`）。
- **工具链**：Yarn 1.22.22（`packageManager` 钉死）、Jest、ESLint、Prettier、Metro；格式化覆盖 6 种语言（`package.json` 的 `format-*` 脚本）；C++ API 快照校验用 Python（`cxx-api-build` 脚本）。
- **JS 引擎**：Hermes（Meta 自研，经 `ReactCommon/hermes/` 适配）与 JSC（`ReactCommon/jsc/`）双支持。

## 4. 目录结构与代码组织

顶层是标准 monorepo（官方说明见 `AGENTS.md` 的"Repo structure"表）：

| 目录 | 职责 |
| --- | --- |
| `packages/react-native/` | 主包：C++ 核心 + 双平台实现 + JS 组件库 |
| ├ `ReactCommon/` | 跨平台 C++：JSI、Fabric 渲染器、Yoga、TurboModules、调试器代理（11MB，1287 个 .cpp/.h） |
| ├ `ReactAndroid/` | Android 运行时（Kotlin/Java/JNI，8.8MB） |
| ├ `React/` | Apple 平台层 Objective-C++（2.8MB）：CoreModules、Fabric 桥接、视图 |
| ├ `ReactApple/` | 新 SwiftPM 化的 Apple 层（起步阶段） |
| ├ `Libraries/` | JS 组件与 API（Flow，47 个子目录：Text、Lists、Animated、TurboModule…） |
| ├ `src/private/` | 迁移中的新 JS 源码区（AGENTS.md 注明"代码逐渐从 Libraries 迁入"） |
| `packages/rn-tester/` | RNTester：官方组件演示/测试 App |
| `packages/react-native-codegen/` | 从 JS spec 生成原生胶水代码的 Codegen |
| `packages/dev-middleware/`、`debugger-frontend/` | 调试服务器中间件与 DevTools 前端 |
| `private/react-native-fantom/` | Fantom：原生集成测试运行器 |
| `scripts/`、`.github/workflows/` | 构建/发布脚本与 30+ 个 CI 工作流 |

组织方式评述：典型的"核心下沉 + 平台薄壳"单仓多包结构。C++ 核心按子系统分目录（`renderer/`、`runtime/`、`jsi/`、`bridging/`），每个子系统自带 podspec + CMakeLists 双构建配置；JS 侧新旧并存（Libraries 旧区 → src/private 新区），迁移痕迹明显（还存在 `types_DEPRECATED/`、`specs_DEPRECATED/`）。架构文档就近放在 `__docs__/` 目录（`__docs__/README.md` 索引）。

## 5. 核心架构与关键实现

### 5.1 JSI：JS 与 C++ 的互操作抽象（`ReactCommon/jsi/jsi/jsi.h`，2266 行）

JSI 定义了与具体引擎无关的 `jsi::Runtime` 抽象基类，Hermes/JSC 各自实现。宿主对象 `HostObject` 是关键设计——C++ 对象以属性访问代理的形式暴露给 JS，属性读写直接回调 C++，无需经过消息队列：

```cpp
class JSI_EXPORT HostObject {
 public:
  virtual ~HostObject();
  // When JS wants a property with a given name from the HostObject,
  // it will call this method. ...
  virtual Value get(Runtime&, const PropNameID& name);
  virtual void set(Runtime&, const PropNameID& name, const Value& value);
  virtual std::vector<PropNameID> getPropertyNames(Runtime& rt);
};
```

`Runtime` 提供 `evaluateJavaScript` / `prepareJavaScript`（jsi.h L357/L372，后者允许预编译字节码，配合 Hermes 启动提速）。JSI 是 TurboModule、Fabric 事件绑定、调试器共同的底层。

### 5.2 Bridgeless 运行时（`ReactCommon/react/runtime/ReactInstance.h`）

旧 Bridge（`ReactCommon/cxxreact/`，含 `NativeToJsBridge`、`ModuleRegistry`、`JSExecutor`，仍在仓库中）已被 `ReactInstance` 取代——它是新架构的 JS 引擎宿主，聚合了缓冲执行器、调用器、定时器与调试器目标：

```cpp
class ReactInstance final : private jsinspector_modern::InstanceTargetDelegate {
 public:
  ReactInstance(std::unique_ptr<JSRuntime> runtime,
      std::shared_ptr<MessageQueueThread> jsMessageQueueThread,
      std::shared_ptr<TimerManager> timerManager,
      JsErrorHandler::OnJsError onJsError, ...);
  RuntimeExecutor getUnbufferedRuntimeExecutor() noexcept;
  RuntimeExecutor getBufferedRuntimeExecutor() noexcept;
  std::shared_ptr<RuntimeScheduler> getRuntimeScheduler() noexcept;
  // The CallInvoker platforms should hand to TurboModules ... async calls
  // are ordered against `callFunctionOnModule` and do not run before the
  // main bundle has finished evaluating.
  std::shared_ptr<CallInvoker> createJSCallInvoker() noexcept;
```

配套的 `BufferedRuntimeExecutor`（缓冲 JS 线程任务直到 bundle 执行完毕）、`RuntimeScheduler`（优先级任务调度，服务并发渲染）同在该目录。

### 5.3 Fabric 渲染器（`ReactCommon/react/renderer/`）

C++ 侧实现了完整的 React 渲染管线：`scheduler/` 协调 surface 生命周期（`Scheduler.h` 自述 "Scheduler coordinates Shadow Tree updates and event flows"），`mounting/ShadowTree.h` 维护不可变 Shadow Tree 的多版本提交（`ShadowTreeCommitStatus: Succeeded/Failed/Cancelled`，支持乐观更新与并发特性），`mounting/Differentiator.cpp` 计算 C++ 树 diff 并生成 `ShadowViewMutation` 挂载指令序列，由平台层（iOS `React/React-RCTFabric.podspec`、Android JNI）应用到真实 View。子目录覆盖 `attributedstring/`（文本度量）、`textlayoutmanager/`、`css/`（新引入的 CSS 值解析）、`viewtransition/` 等。Yoga 布局计算（`ReactCommon/yoga/`）在提交事务内同步执行。

### 5.4 类型化互调层 Bridging 与 Codegen（`ReactCommon/react/bridging/` + `packages/react-native-codegen/`）

`bridging/` 提供按 C++ 类型到 `jsi::Value` 的模板化转换（`Bridging.h`、`Convert.h`、`Promise.h` 等 24 个头文件），TurboModule 借此实现 JS↔C++ 的自动编解码。`react-native-codegen` 再从 JS 侧的 spec 文件生成原生接口胶水，实现"JS 声明、原生实现"的双端类型安全（Codegen 构建产物 `lib/` 不入库，需 `yarn --cwd packages/react-native-codegen build`，见 `AGENTS.md` Gotchas）。

## 6. 构建与运行

来源：根 `package.json` scripts 与 `AGENTS.md`"Common commands"。

```bash
yarn install                                  # 安装依赖（注意 preinstall 会改 yarn.lock，属预期）
yarn test <path>                              # Jest 单测（__tests__ 目录）
yarn fantom <path>                            # Fantom 原生集成测试（*-itest.js）
yarn lint / yarn flow-check / yarn format     # ESLint / Flow / 六语言格式化
yarn start                                    # Metro 起 RNTester（localhost:8081）
yarn android                                  # 编译安装 RNTester 到 Android
./gradlew :packages:react-native:ReactAndroid:build   # 构建 Android 侧（build-android）
./scripts/objc-test.sh test                   # iOS 单测（test-ios）
node ./scripts/build/build.js                 # 构建 npm 发布产物（build）
```

原生构建需要 Xcode + CocoaPods/SwiftPM（macOS）或 Android SDK/NDK + Gradle；纯 JS 开发只需 Node + Yarn（`AGENTS.md` Environment 节）。

## 7. 测试与工程质量

- **规模**：`packages/` 下 108 个 `__tests__` 目录、约 392 个测试文件（`find` 统计，含 Jest 单测与 Fantom 集成测试）。
- **分层**：Jest 单测（JS）→ **Fantom**（自研原生集成测试运行器，`private/react-native-fantom/`，跑真 C++/原生栈但无需模拟器）→ RNTester 手动/E2E 验证 → e2e-android/ios-rntester、maestro-cloud-rntester 等 CI 端到端工作流（`.github/workflows/`）。
- **CI**：30+ 个 GitHub Actions 工作流（`ls .github/workflows/`），覆盖双平台测试、SwiftPM 预编译、npm 发布（`publish-npm.yml`）、草稿 release、格式化报告、PR 自动 rebase、stale 管理等；另有 Expo 公司的 code review 机器人工作流（`expo-code-review.yml`）。
- **质量机制**：六语言统一格式化门禁；C++ API 快照 diff 校验（`js-api-diff`、`api-changes.yml`）；feature flags 体系（`ReactCommon/react/featureflags/`）控制灰度；生成代码禁止手改、CI 校验快照（`AGENTS.md`）。

## 8. 社区与维护状态

- **活跃度**：gh api 显示最近推送 2026-10-29；CHANGELOG 显示 v0.88.0（2026-10-15）→ v0.88.1（2026-10-27）→ rc.6 并行，约每月一个 minor、随时 hotfix 的节奏；v0.88.1 单版本贡献者署名含 Meta 员工（mrousavy、jkadamczyk、huntie、alanjhughes）与外部开发者（chrfalch、intergalacticspacehighway、lazerg 等，均见 `CHANGELOG.md`）。
- **贡献者规模**：GitHub contributors API 单页仅返回 30 条且有多页，精确总数未确认；从 CHANGELOG 署名（Meta、Microsoft/Shopify/Expo 系工程师混编）可确认是多公司协作模式。
- **issue/PR**：open issues 约 1333（gh api），有专门的 `check-for-reproducer.yml`（要求可复现仓库）、`on-issue-labeled.yml` 等分流机器人。
- **治理**：React Foundation 托管（`README.md`），版本随 React 19 系（devDependencies `@types/react ^19.1.0`）。

## 9. 亮点与风险

**亮点**
1. 架构完成度极高的"核心下沉"：渲染、布局、模块系统、调试器全部 C++ 化，新平台（ visionOS、Windows）接入成本被压到平台薄壳层（`ReactCommon/` 目录划分可见一斑）。
2. JSI + HostObject 的零拷贝互操作设计，淘汰了 JSON Bridge，成为后来者（如 Lynx 等）竞相参考的范式（`ReactCommon/jsi/jsi/jsi.h`）。
3. 测试金字塔真实落地：自研 Fantom 让集成测试摆脱模拟器拖累，108 个 `__tests__` + 30 条 CI 流水线的工程纪律业内罕见。
4. 文档与代码同仓：`__docs__/` 就近架构文档、`AGENTS.md` 对 monorepo 结构/坑点（hermes-compiler 占位包、codegen 需先构建）的坦诚说明。
5. 调试体验闭环：CDP 代理（`jsinspector-modern/`）、DevTools 前端（`packages/debugger-frontend/`）、dev-middleware 全部开源在仓。

**风险/局限**
1. 复杂度门槛极高：四种语言、三套构建系统（CMake/CocoaPods/Gradle）并存，Libraries → src/private 的迁移与 `*_DEPRECATED` 目录表明重构仍在半途。
2. monorepo 内含大量仅供 Meta/CI 使用的机制（prebuild、release-testing 脚本），外部贡献者理解成本大。
3. Yarn 1（EOL）仍是 `packageManager`，工具链偏保守。
4. 每次 React 大版本升级（并发特性、Suspense 语义变化）都会在 Fabric 侧产生联动改造成本，升级摩擦长期存在。

## 10. 总结评价

React Native 是移动跨平台领域的奠基者与持续领跑者：十年间从"JS Bridge 消息队列"演进到"JSI + Fabric + TurboModules"的 C++ 新架构，把一个曾经依赖序列化桥的框架改造成了引擎级基础设施，且至今保持每月发版、千余人协作的顶级活跃度。它适合：需要一套代码覆盖双端且在意原生手感的业务团队（建议走 Expo 生态而非裸用）；想学习"JS 引擎宿主、响应式渲染器跨平台实现、大型 monorepo 工程"的工程师——`ReactCommon/jsi/`、`react/renderer/mounting/`、`react/runtime/` 是三份彼此独立又互为表里的绝佳源码教材。对于只想快速上手的初学者，这个 126k 星仓库本身太大，应从 reactnative.dev 文档与 Expo 入口进入。
