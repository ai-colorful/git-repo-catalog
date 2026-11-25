# expo/expo 项目分析

> 星数 53033（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：Expo 是构建通用原生 App（Android / iOS / Web）的开源平台工具链，本仓库包含 Expo SDK、Modules API、CLI、expo-router、Expo Go 客户端与官方文档的全部源码。

| 事实 | 值 | 出处 |
| --- | --- | --- |
| 名称 | expo/expo（npm 包 `expo`，SDK 58，v58.0.19） | `packages/expo/package.json` |
| 星数 | 53033（快照）/ 53316（gh api 实时） | 任务参数 + gh api |
| 主语言 | TypeScript（辅以 Kotlin 2485 文件、Swift 2095 文件、C++/ObjC 384 文件） | 源码统计 |
| 许可证 | MIT（Copyright 650 Industries, Inc. aka Expo） | `LICENSE` |
| 创建/最近活跃 | 2016-08-15 创建，2026-11-03 仍在 push | gh api repos/expo/expo |
| 工程形态 | pnpm monorepo + turbo，约 155 个包 | `pnpm-workspace.yaml`、`packages/` |

## 2. 项目是什么，解决什么问题

React Native 让开发者用 React 写原生 App，但"裸 RN"需要自行搭建原生工程、管理 Gradle/Xcode 依赖、配置构建签名与原生权限，门槛高且琐碎。Expo（`README.md`："an open-source platform for making universal native apps that run on Android, iOS, and the web"）把这条链路产品化：一个统一的运行时 + 一套带原生实现的模块库 + 一组工具命令，让开发者基本只写 TypeScript/React 就能产出三大平台的应用。

具体由四层构成（目录职责见 `README.md` 的 Project Layout 一节）：

1. **统一运行时与模块系统**：`packages/expo` 是面向用户的 SDK 门面，真正能力来自 `expo-modules-core` 定义的原生模块协议（Kotlin/Swift/C++ 三端实现），所有功能模块（camera、sqlite、updates 等百余个）按同一协议编写，可被自动链接（autolinking）进任意 RN 工程。
2. **工程配置层**：`@expo/config` 读取 `app.json` / `app.config.js` 声明式描述 App；`@expo/config-plugins` 允许以可组合的插件修改原生工程，`prebuild` 命令据此生成 android/ios 目录——这是"受管工作流"与"裸工作流"（bare workflow）的连接点。
3. **工具链**：`@expo/cli`（约 742 个 TS 文件）提供 `start/run/prebuild/export/install` 等命令；`apps/expo-go` 是可从商店安装的通用宿主 App，开发期扫码即可运行项目而无需本地编译原生代码。
4. **配套生态**：`expo-router`（902 个源文件）提供文件系统路由；`react-native-lab/` 是为 Expo Go 定制的 RN fork；EAS Build/Submit/Update 等云服务（`expo.dev/eas`）与开源工具深度集成但服务端闭源（`README.md`）。

与纯云平台不同，Expo 自 SDK 46 起全面转向"prebuild 生成原生工程 + 开发者可随时 eject 到裸工作流"的路线，既保留零配置体验，又不锁定用户——这是它能覆盖从入门到大型团队的关键设计。

## 3. 技术栈与依赖

- **语言**：TypeScript 为主（packages/ 下 5442 个 .ts/.tsx），原生层 Kotlin（Android）、Swift（iOS，全面支持 Swift Package Manager：`packages/expo/Package.swift`、`packages/expo-modules-core/spm.config.json`）、C++ 共享核心（`packages/expo-modules-core/common/cpp/`）。
- **框架**：React 19.3 + React Native 0.88.1（`pnpm-workspace.yaml` overrides 强制统一版本）；web 端走 react-native-web。
- **Monorepo 工程**：pnpm 12 workspace（`pnpm-workspace.yaml`）+ turborepo 编排（`package.json` scripts 均为 `./bin/expo-turbo ...`，`turbo.json`）；changesets 管理约 150 个包的版本与发布（`.changeset/config.json`）。
- **代码质量工具**：oxlint + oxfmt（`packages/expo/oxlint.config.mjs`、根 `package.json` lint-staged），TypeScript 6。
- **测试**：Jest 30（根 `package.json` devDependencies），`jest-expo` 自研 preset。
- **值得注意的供应链防护**：`pnpm-workspace.yaml` 设置 `minimumReleaseAge: 1440`（依赖发布满 24 小时才允许安装）、`strictDepBuilds: true`、`blockExoticSubdeps: true`；对 netinfo/reanimated 等以 `patchedDependencies` 维护补丁。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `packages/` | 全部开源包：125 个顶层条目 + `@expo/` scoped 30 个 + `@unimodules/`，约 155 个包（源码统计） |
| `packages/expo` | SDK 门面包，v58.0.19；bin 下 4 个可执行入口（expo/autolinking/fingerprint 等） |
| `packages/expo-modules-core` | 原生模块系统核心：JSI 桥接协议、NativeModule 基类、SharedObject/SharedRef 生命周期 |
| `packages/@expo/cli` | 现 CLI（19.6k 行 TS），start/run/prebuild/export 等全部命令 |
| `packages/@expo/config` | app.json/app.config.js 解析与合并（3.9k 行 TS） |
| `packages/@expo/config-plugins` | 原生工程修改插件系统（依赖注入到 expo 门面包，`packages/expo/package.json`） |
| `packages/expo-router` | 文件路由（902 个 src 文件，源码统计） |
| `apps/` | 17 个内部 App 项目：expo-go（商店客户端）、bare-expo（开发用宿主）、test-suite、native-component-list、router-e2e 等，共 1294 个 TS 文件 |
| `docs/` | docs.expo.dev 源码：1810 个 mdx/md + 439 个 TS |
| `react-native-lab/` | Expo Go 使用的 RN fork |
| `templates/` | `create-expo-app` 的项目模板 |
| `tools/`、`scripts/`、`.github/workflows/` | expotools 构建工具、CI 脚本、55 个 workflow |

组织方式评述：典型的大型 pnpm monorepo——每个包自带 `package.json`/`CHANGELOG.md`/`turbo.json`/双端原生目录（`android/`、`ios/`），通过 `workspace:*` 依赖互链，changesets 独立发版。内部由 apps（bare-expo 等）作为"活体测试场"驱动包开发，docs 与 packages 同仓保证文档与代码同步。

## 5. 核心架构与关键实现

### 5.1 原生模块桥接：JSI 优先的三级回退

`packages/expo-modules-core/src/requireNativeModule.ts` 是全 SDK 的取模块入口，逻辑极清晰地展示了 Expo 的桥接演进（旧 bridge → TurboModule → JSI host 对象）：

```ts
return (
  globalThis.expo?.modules?.[moduleName] ??      // ① JSI host object（同步、高性能）
  NativeModulesProxy[moduleName] ??              // ② legacy bridge 代理
  createTurboModuleToExpoProxy(                  // ③ RN TurboModule 适配
    TurboModuleRegistry.get(moduleName) ?? null, moduleName
  ) ?? null
);
```

JSI 路径下连 `NativeModule` 类本身都由原生注入：`src/NativeModule.ts` 直接取 `globalThis.expo.NativeModule`。配套的 `src/SharedObject.ts` / `src/SharedRef.ts` 实现原生对象在 JS 侧的确定性释放（`useReleasingSharedObject` hook 导出于 `packages/expo/src/Expo.ts`）。C++ 共享核心在 `packages/expo-modules-core/common/cpp/`，Kotlin 与 Swift 端各自有 `expo-module-gradle-plugin/`、`ExpoModulesCore.podspec` 等构建接线。

### 5.2 配置系统：声明式 config + 可组合 plugins

`packages/@expo/config/src/Config.ts`（3.9k 行包的核心）将 `app.json`（静态）与 `app.config.js`（动态，可读环境变量）统一 reduce 为 `ExpoConfig`，并对"根级 expo 对象"给出迁移告警（`reduceExpoObject`，Config.ts 前 60 行）。插件链由 `src/plugins/withConfigPlugins.ts` 按 `sdkVersion` 从 `@expo/config-plugins` 逐个套用，最终在 `expo prebuild` 时决定生成的 android/ios 原生工程内容（CLI 的 prebuild 命令见 `packages/@expo/cli/src/prebuild/`）。

### 5.3 CLI：懒加载命令 + 面向 agent 的会话协议

`packages/@expo/cli/src/index.ts` 以一张命令表做按需 `import()`（19 个子命令：run:ios/run:android/start/prebuild/config/export/export:embed/serve/compile/install/customize/lint/login 等），默认命令为 `start`，并兼容旧 `EXPO_DEBUG` 环境变量。值得注意的是 2026 年新增的 `command` / `command:ps` / `command:events` 三个命令，注释标注 "Recorded command sessions (primarily for agents)"（`src/index.ts` 第 45-49 行），把 CLI 执行过程事件流化供编码代理消费；根目录 `AGENTS.md` 亦是给 AI 贡献者的说明（`CONTRIBUTING.md` 明确要求 coding agents 先读它）。

### 5.4 EAS 生态边界

开源仓库与 EAS 云服务的边界在本仓库内有明确刻痕：CLI 不含 `eas build/submit` 命令（命令表无 eas 项），EAS 相关能力以库形态存在——`packages/expo-eas-client`（EAS 元数据客户端）、`packages/expo-updates`（OTA 更新，含代码签名证书测试夹具 `expo-updates/android/src/.../certificates`）、`packages/@expo/fingerprint`（项目指纹，用于 EAS Build 缓存判断）。eas-cli 本体在独立仓库 expo/eas-cli（未确认，本仓库内无源码）。这种"开源 SDK/CLI + 闭源云服务"的组合是商业模式核心（`README.md`）。

## 6. 构建与运行

来自根 `package.json` 与 `CONTRIBUTING.md` 的实际命令（均未实际运行，静态摘录）：

```bash
# 初始化（pnpm 12 + Node 22+）
pnpm install                 # preprepare/prepare 钩子会跑 turbo build
pnpm install:react-native-lab  # RN fork 的 node_modules

# 开发 SDK 包（推荐在 apps/bare-expo 中联调，见 CONTRIBUTING.md）
pnpm build        # = ./bin/expo-turbo build（turbo 编排全包）
pnpm typecheck    # = ./bin/expo-turbo typecheck
pnpm lint         # oxlint/oxfmt
pnpm test         # = ./bin/expo-turbo test（各包 jest）

# 单包示例（packages/expo/package.json）
pnpm --filter expo build && pnpm --filter expo test   # jest + spm 插件测试 + 模板校验

# 面向用户的脚手架
npx create-expo-app my-app    # 使用 templates/ 下模板（README.md）
```

## 7. 测试与工程质量

- **单元测试**：packages/ 内 1029 个 *.test.ts(x) 文件（源码统计），Jest 30 + 自研 `jest-expo` preset；expo-modules-core 还有专门的 `__rsc_tests__/`（React Server Components 场景）。
- **E2E/仪器测试**：`apps/test-suite`（SDK 行为断言）、`apps/router-e2e`（router 的 Maestro/Detox 类测试）、`apps/native-tests`；Android 有独立 workflow（`.github/workflows/android-instrumentation-tests.yml`、`android-unit-tests.yml`）。
- **CI**：55 个 workflow，覆盖按包触发（cli.yml、fingerprint.yml、create-expo-app.yml 等）、文档部署（docs.yml、docs-pr.yml）、 nightly issue 检查（check-issues-nightly.yml）、甚至 AI 代码评审（expo-code-review.yml、code-review.yml）（`.github/workflows/` 目录清单）。
- **质量观察**：类型全覆盖（TS 6 + 每包独立 tsconfig）、oxlint 静态检查、lint-staged 提交卡点；`pnpm-workspace.yaml` 的 minimumReleaseAge/strictDepBuilds 显示对供应链攻击有主动防御。包级 CHANGELOG 粒度到 patch 版本（`packages/expo/CHANGELOG.md` 58.0.19 条目引用 PR #52096），工程纪律极强。

## 8. 社区与维护状态

- **活跃度**：gh api 显示最近 push 时间 2026-11-03（分析前一天），open issues 约 1025，PR 编号已达 52k+（`packages/expo/CHANGELOG.md` 引用 #52096），约 2286 名贡献者（gh api contributors 分页计数，含匿名）。
- **发布节奏**：changesets 驱动的日级 npm 发布——最新 GitHub release 为 2026-11-03 的 `expo-template-tabs@58.0.32`（gh api releases/latest，多数包发布不建 GitHub release 而是直发 npm）；大版本按 SDK 年度推进：SDK 57 于 2026-07-08 发布（根 `CHANGELOG.md`），SDK 58 当前为 58.0.19。
- **贡献者结构**：核心由 Expo 公司（650 Industries）全职团队维护（`packages/expo` 等包 changelog 中作者高度集中于内部成员），外部 PR 可经由 `CONTRIBUTING.md` 与 `AGENTS.md` 规范进入；公司有专职文档、CI、AI 评审岗位痕迹（workflow 清单为证）。

## 9. 亮点与风险

**亮点**
1. 模块系统设计先进：JSI 同步桥 + SharedObject/SharedRef 显式生命周期 + SwiftPM/Gradle 插件化构建接线，是 RN 生态里原生层工程化的标杆实现（`packages/expo-modules-core/src/`）。
2. 不锁定的受管工作流：prebuild + config-plugins 让 `app.json` 与原生工程可互相转换，用户随时可控原生代码（`packages/@expo/cli/src/prebuild/`、`packages/@expo/config/`）。
3. 超大 monorepo 的工程范本：155 包独立发版、turbo 缓存编排、changesets 自动化、55 条 CI 流水线、1029 个单测文件。
4. 供应链安全意识：依赖 24 小时冷却期、禁止 exotic 子依赖、关键 RN 依赖统一 override + 补丁（`pnpm-workspace.yaml`）。
5. 面向 AI 代理的前瞻布局：AGENTS.md、`expo command` 会话事件流命令（`packages/@expo/cli/src/index.ts`）。

**风险/局限**
1. 仓库体量巨大（源码 362MB 压缩包 172MB），外部贡献者定位改动点与本地全量构建成本高（CONTRIBUTING 自己也建议用 bare-expo 而非 Expo Go 联调）。
2. SDK 年度大版本升级（57→58）伴随 breaking changes 面较宽（根 `CHANGELOG.md` 57.0.0 一节），生态内第三方库需持续跟进。
3. 商业边界依赖闭源：EAS 服务端不可自托管，深度使用（Build/Update 托管）即绑定 Expo 云（`README.md`）。
4. RN fork（react-native-lab）与上游 RN 的同步存在长期维护成本（目录存在，`pnpm-workspace.yaml` 中 react-native 被 override 到 0.88.1）。

## 10. 总结评价

expo/expo 是 React Native 生态事实上的"平台层"：它把移动开发中最难工程化的原生配置、构建、模块桥接三件事做成了可声明、可组合、可回退的开源工具链，并以此支撑起 EAS 云服务商业闭环。源码呈现的是教科书级 monorepo 治理——数百包独立演进却保持 SDK 整体一致性。适合三类人：移动开发者（把它当生产力工具）、想学习跨端原生桥接/构建系统设计的工程师（读 `expo-modules-core` 与 `@expo/config-plugins`）、以及研究大型 monorepo 运营（CI/发布/文档一体化）的工程效能团队。唯一要有心理准备的是它的体量——降采样阅读是正确姿势。
