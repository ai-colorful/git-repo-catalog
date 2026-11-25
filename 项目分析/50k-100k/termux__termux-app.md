# termux/termux-app 项目分析

> 星数 63258（快照 2026-10-24）｜主语言 Java｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：Termux 是 Android 上的终端模拟器与 Linux 环境 App，把 Android 设备变成一台可安装 GNU/Linux 工具链的"口袋工作站"。

| 项目 | 内容 |
|---|---|
| 名称 | termux/termux-app |
| 星数 | 63258（快照 2026-10-24） |
| 主语言 | Java（纯 Java，无 Kotlin；201 个 .java 文件，主代码约 44.4k 行） |
| 许可证 | GPLv3 only（`LICENSE.md`；terminal-emulator/terminal-view 内含 Apache 2.0 的 Android Terminal Emulator 例外） |
| 最近活跃 | 最近 push 2026-09-26（gh api repos/termux/termux-app）；最新 release v0.118.3（2025-05-22） |
| 其他 | 8102 forks、649 open issues、1684 watchers、 contributors 首页 30 人（gh api 快照） |

## 2. 项目是什么，解决什么问题

Android 是基于 Linux 内核的操作系统，但官方并不提供 shell、包管理器或任何类 Unix 用户态。Termux 解决的正是这个问题：它在普通（无需 root）的 Android App 沙箱里搭起一个终端环境——App 自带一个嵌入 APK 内的 bootstrap 最小包集合（来自姊妹仓库 termux-packages），首次启动时解压到 `$PREFIX`（`app/src/main/java/com/termux/app/TermuxInstaller.java`），之后用户就可以在 `$PREFIX` 下使用 apt/pkg 安装数千个编译好的软件包（python、nodejs、openssh、vim 等）。本仓库只负责 App 本体（终端模拟 + 会话 UI + 插件 API），包的构建与托管在 termux/termux-packages。

技术上它做了三件难事。其一是完整的 xterm/VT100 终端模拟器（`terminal-emulator` 模块），包括转义序列状态机、滚动回看缓冲、宽字符/东亚宽度处理、Sixel 图形输出等；其二是通过 NDK/JNI fork+execvp 在 pty 上启动真实 shell 子进程（`terminal-emulator/src/main/jni/termux.c`）；其三是 foreground Service 常驻的会话管理，使终端会话和后台任务在 Activity 被系统回收后依然存活（`app/src/main/java/com/termux/app/TermuxService.java`）。

Termux 同时是一个生态枢纽：App 通过 `sharedUserId com.termux` 与 6 个插件 App（API、Boot、Float、Styling、Tasker、Widget）共享 uid（README "Termux App and Plugins" 一节），并通过导出的 `RunCommandService` 向第三方 App/Tasker 开放"执行命令"的 Intent API。这些插件各有独立仓库，共享代码则抽取为 `termux-shared` 库（README "For Maintainers and Contributors"）。

分发上，Termux 处于一个特殊处境：Play 商店政策不支持"下载可执行文件并运行"的完整功能，因此正式版走 F-Droid 与 GitHub Releases，Play 商店版是由独立仓库 termux-play-store 维护的实验分支（README "Google Play Store (Experimental branch)"）。不同来源 APK 签名不同且因 sharedUserId 不可混装，README 用大量篇幅解释这一点。

## 3. 技术栈与依赖

- 语言与运行时：纯 Java（`find src -name "*.kt"` 为 0），无第三方 UI 框架，终端 View 为自绘 `android.view.View`。
- 构建：Gradle + Android Gradle Plugin 8.13.2（`build.gradle`），NDK 29.0.14206865（`gradle.properties`），minSdk 21 / targetSdk 28 / compileSdk 36。targetSdk 刻意停在 28，规避 Scoped Storage 等对终端类 App 的限制（`gradle.properties` 与 manifest 中 `MANAGE_EXTERNAL_STORAGE` 可佐证）。
- CI 固定 Java 17（`.github/workflows/run_tests.yml`）。
- 依赖极少（`app/build.gradle`）：AndroidX annotation/core/drawerlayout/preference/viewpager、Material 1.12.0、Guava 24.1-jre、Markwon 4.6.2（HelpActivity 渲染 Markdown 文档用）。
- 原生层：约 220 行 C（`app/src/main/cpp/termux-bootstrap.c`、`terminal-emulator/src/main/jni/termux.c`），分别用于导出内置 bootstrap zip 字节流和创建 pty 子进程。

## 4. 目录结构与代码组织

Gradle settings 中声明 4 个模块（`settings.gradle`），代码行数分布（主代码）如下：

| 目录 | 主代码行数 | 职责 |
|---|---|---|
| `terminal-emulator/` | 7801 | 独立 Java 库：VT/xterm 终端模拟核心 + pty JNI，不依赖 Android UI |
| `terminal-view/` | 2880 | 终端 View 控件：自绘渲染、手势缩放、文本选择光标 |
| `termux-shared/` | 26748 | App 与插件共享库：常量、shell 执行框架、偏好/属性、通知、本地 socket、崩溃上报 |
| `app/` | 6953 | 应用本体：Activity/Service、bootstrap 安装、插件入口、设置界面 |
| `.github/workflows/` | - | 6 个 workflow：Build、Unit tests、Jitpack 触发、依赖提交等 |
| `art/`、`fastlane/`、`site/`、`docs/` | - | 图标、F-Droid 元数据、官网资产、文档页 |

组织方式评述：标准 Android 多模块单体。分层清晰——`terminal-emulator` 与 `terminal-view` 是可复用的纯终端库（其他项目可直接引入，也发布到 Jitpack，`jitpack.yml`），`termux-shared` 承担"App + 6 个插件"之间的代码共享（v0.109 引入，README 有贡献规范：禁止硬编码路径，必须走 `TermuxConstants`）。`termux-shared/src/main/java/com/termux/shared/termux/TermuxConstants.java` 是 1338 行的常量总纲，集中定义包名、路径、Intent action 等，并内嵌 fork 指南。

## 5. 核心架构与关键实现

### 5.1 terminal-emulator：转义序列状态机 + 屏幕缓冲

`TerminalEmulator.java`（3569 行）是整个仓库的核心。它用一个显式状态机解析 VT100/xterm 字节流，状态常量覆盖 ESC、CSI、OSC、DCS、APC 等（如 `ESC_CSI`、`ESC_OSC`、`ESC_DCS`、`ESC_APC`，见文件 56-173 行），DECSET/DECRST 私有模式用位标志管理（application cursor keys、reverse video、autowrap、mouse tracking、SGR mouse、bracketed paste、left-right margin mode 等，179-212 行）。屏幕内容存于 `TerminalBuffer.java`（769 行）的行数组，回滚行数可配置 100-50000（默认 2000，`TerminalEmulator.java` 228-230 行）；字符宽度用 `WcWidth.java`（移植自 POSIX wcwidth 表）处理东亚宽字符；`TerminalSixel.java` + `ITermImage`/`TerminalBitmap` 实现了 Sixel 位图图形协议——这在移动端终端模拟器中相当少见。

### 5.2 TerminalSession：pty 子进程与三线程 I/O 模型

`TerminalSession.java`（373 行）把"进程 + 仿真器 + I/O"绑成一个会话。构造时延迟到首次 `updateSize()` 才通过 JNI `createSubprocess` fork 出 shell（`initializeEmulator()`，123-174 行），随后启动三个线程：读线程把 pty 输出写入 64KB `ByteQueue` 并通知主线程、写线程把用户输入从 4KB 队列写回 pty、waiter 线程阻塞在 `JNI.waitFor` 等子进程退出。pty master fd 是 int，通过反射塞进 `java.io.FileDescriptor` 包装（`wrapFileDescriptor()`，317-334 行）。主线程 Handler 把字节喂给 `mEmulator.append()`，进程退出时往屏幕追加 `[Process completed (signal 9) - press Enter]`（349-368 行）——正是 README 中 Android 12 phantom process 问题的可见现象。

```java
// terminal-emulator/src/main/java/com/termux/terminal/TerminalSession.java (127-133)
int[] processId = new int[1];
mTerminalFileDescriptor = JNI.createSubprocess(mShellPath, mCwd, mArgs, mEnv,
    processId, rows, columns, cellWidthPixels, cellHeightPixels);
mShellPid = processId[0];
```

JNI 侧（`terminal-emulator/src/main/jni/termux.c`）用经典做法：`open("/dev/ptmx")`、`grantpt/unlockpt/ptsname_r`、设置 `IUTF8` 并关掉 `IXON/IXOFF`（防 Ctrl+S 锁屏）、`TIOCSWINSZ` 设窗口尺寸，子进程 `setsid()` 后把 0/1/2 全部 dup 到 pts、`clearenv()` 后灌入 Termux 环境、`execvp`。38-115 行约 80 行 C 完成了整个 Unix 终端栈的搭建，非常干净。

### 5.3 TermuxService：会话生命周期与插件执行入口

`app/src/main/java/com/termux/app/TermuxService.java`（959 行）是 foreground Service，持有 `TermuxShellManager` 中的三份状态：`mTermuxSessions`（前台终端会话）、`mTermuxTasks`（后台 AppShell 任务）、`mPendingPluginExecutionCommands`（尚未执行的插件命令）。Activity 与 Service 解耦是设计要点：Service 通常比 Activity 长寿，因此 `TerminalSessionClient` 有两套实现——Activity 绑定时换成功能齐全的 `TermuxTerminalSessionActivityClient`，解绑后回退到不持有 Activity 引用的 `TermuxTerminalSessionServiceClient`（`getTermuxTerminalSessionClient()`，745-750 行），避免内存泄漏与悬空引用。

对外的插件 API 走 `ACTION_SERVICE_EXECUTE` Intent（359-425 行 `actionServiceExecute`）：解析 executable URI、参数、stdin、工作目录、runner（TERMINAL_SESSION 前台会话 / APP_SHELL 后台任务）、sessionAction（是否切换会话/拉起 Activity）、结果回传方式（PendingIntent 或结果文件目录），然后交给 `createTermuxSession`/`createTermuxTask`。Service 销毁时 `killAllTermuxExecutionCommands()`（263-299 行）精细区分"用户主动退出"与"系统回收"，对期待回传结果的插件命令发送取消通知，防止 Tasker/RUN_COMMAND 调用方永久挂起——注释里对此有完整推演。

### 5.4 插件安全模型：RunCommandService 与 allow-external-apps

`app/src/main/java/com/termux/app/RunCommandService.java`（300 行）是导出的第三方入口（manifest 中 `android:exported="true"` 且要求自定义权限 `com.termux.permission.RUN_COMMAND`，`app/src/main/AndroidManifest.xml` 172-177 行）。它校验可执行文件存在与权限、处理 `am` 命令逗号转义等边角问题后，组装成内部 Intent 转发给 TermuxService 自己即退出。安全闸门是 `allow-external-apps` 属性：未在 termux.properties 中显式开启时拒绝执行，并强制弹通知告知"有 App 试图在 Termux 上下文执行命令"（132-137 行）。133-150 行的长注释还披露了一个真实漏洞的防御：文件型结果回传的 format 字符串可被恶意客户端用来覆写 termux.properties，因此结果目录回传被推迟到策略校验之后。

### 5.5 bootstrap 安装

`TermuxInstaller.java`（386 行）：bootstrap zip 以字节常量形式编进共享库（`app/src/main/cpp/termux-bootstrap-zip.S` 汇编内嵌 blob，`termux-bootstrap.c` 的 `getZip()` JNI 返回），安装时解压到 `$STAGING_PREFIX`、按 `SYMLINKS.txt` 重建符号链接、再落位 `$PREFIX`。把 bootstrap 藏进 .so 是为了让 APK 保持单一实体（也顺带绕开资源大小限制），注释 42-58 行完整描述了流程。

## 6. 构建与运行

```bash
# 单元测试（CI 同款，.github/workflows/run_tests.yml）
./gradlew test

# 构建 APK（两个 bootstrap 变体矩阵，.github/workflows/debug_build.yml）
export TERMUX_PACKAGE_VARIANT=apt-android-7   # 或 apt-android-5
export TERMUX_APP_VERSION_NAME=0.118.0        # 可选，须符合 semver 2.0.0
./gradlew assembleDebug

# 变体默认 apt-android-7，代码路径见 app/build.gradle ext.packageVariant
```

注意：仓库自带 `testkey_untrusted.jks` 公共测试签名（README 安全警告），GitHub 构建互相兼容但不能与 F-Droid 版混装。版本名校验由 `validateVersionName` 在构建期强制执行（`app/build.gradle`）。

## 7. 测试与工程质量

- 测试集中在 terminal-emulator 模块：19 个测试类、约 2980 行（`terminal-emulator/src/test/java/com/termux/terminal/`），覆盖 CSI/OSC/DCS/APC 解析（`ControlSequenceIntroducerTest`、`ApcTest`、`OperatingSystemControlTest`）、光标与屏幕操作（`CursorAndScreenTest`、`ScrollRegionTest`、`RectangularAreasTest`）、DECRST 行为（`DecSetTest`）、resize 重排（`ResizeTest`）、历史缓冲（`HistoryTest`）、Unicode 输入与宽度（`UnicodeInputTest`、`WcWidthTest`）等，均为 JUnit 风格的功能级测试，通过 `TerminalTestCase` 直接驱动仿真器字节流——是 Android 开源项目里少见的对终端模拟的系统性测试。
- app 模块仅 2 个测试类（`FileReceiverActivityTest`、`TermuxActivityTest`）；termux-shared 无单元测试目录（`find src/termux-shared/src/test` 为空），其质量主要靠插件 App 侧使用。
- CI：push/PR 触发 Build（apt-android-7/5 矩阵 + semver 校验）与 Unit tests（`./gradlew test`），另有 gradle-wrapper-validation、dependency-submission、release 时自动挂载 debug APK 等辅助 workflow（`.github/workflows/`）。
- 质量观察：提交规范强制 Conventional Commits 并用脚本自动生成 changelog（README "Commit Messages Guidelines"）；日志分级系统化（Logger + Log Levels 一节）；错误处理统一走 `Error`/`Errno` 体系。整体代码注释密度高、防御性强，但部分类较长（TerminalEmulator 3569 行）。

## 8. 社区与维护状态

- 历史与体量：创建于 2015-10，63k+ 星、8.1k forks、长期位居 Android 类开源项目头部（gh api 快照）。
- 活跃度：最近 push 2026-09-26，仓库持续有提交；但 README 顶部写明 "We are looking for Termux Android application maintainers"，App 本体处于寻找维护者的状态（README 第 16 行）。
- Release 节奏：最新正式版 v0.118.3（2025-05-22），0.118.x 系列以稳定性修补为主；F-Droid 版会滞后 GitHub 数天到数周（README "F-Droid" 一节）。
- 资金与组织：获得 GitHub Accelerator（2023 首期）、GitHub Secure Open Source Fund（2025 第二期）、NLnet NGI Mobifree 资助，Cloudflare 赞助包镜像（README "Sponsors and Funders" 及 site/assets/sponsors/）；包托管在 termux-packages 组织。
- 社区渠道：Reddit r/termux、Discord、Gitter/Matrix、独立 wiki（README "Important Links"）。contributors 首页 30 人，具体总数未确认。

## 9. 亮点与风险

亮点：

1. 教科书级的分层：终端模拟（纯 Java 可独立复用）、终端控件、应用层三个 Gradle 模块边界清晰，`termux-emulator`/`terminal-view` 被 Jitpack 分发供第三方复用（`jitpack.yml`）。
2. 测试扎实的终端核心：近 3000 行针对转义序列协议行为的测试，让这个 3500 行的状态机可持续演进。
3. 严谨的进程/生命周期工程：Service 长于 Activity 的双客户端切换、插件命令的取消与结果回传兜底、wake lock 的通知可视化管理，注释记录了每个 Android 生命周期坑。
4. 完整的插件生态设计：RUN_COMMAND Intent API + 自定义权限 + allow-external-apps 开关 + 结果回传协议，把"其他 App 能否安全地指挥终端"这个难题处理得系统化。
5. 极少的外部依赖（AndroidX + Material + Guava + Markwon），原生代码仅 220 行，攻击面与升级负担都很小。

风险/局限：

1. targetSdkVersion 停在 28（`gradle.properties`），依赖 MANAGE_EXTERNAL_STORAGE 等旧权限，Play 商店正式分发受阻，只能依赖 F-Droid/GitHub 侧载，政策风险长期存在。
2. Android 12+ phantom process 限制导致后台 shell 进程被系统杀（README NOTICE 与 issue #2366），App 侧无法根治，需用户手动改系统设置。
3. GitHub 构建使用公开测试签名（`app/testkey_untrusted.jks`），任何人都可伪造"官方" GitHub 版 APK，供应链与社工风险由文档而非技术手段缓解（README 安全警告）。
4. 维护者缺口：App 本体公开招募维护者（README），0.118.x 后近半年无新版本发布（gh api），大特性开发可能放缓。

## 10. 总结评价

Termux 是"在受限平台上重建 Unix 环境"的标杆项目：63k 星背后，是一个把 pty、VT 状态机、Android 生命周期、插件安全模型都处理到工程级严谨度的纯 Java 代码库。它不只是个终端 App，而是整个 Android 上 CLI 生态（termux-packages、六七个插件、无数脚本）的地基。想学习 Android 进程模型、终端模拟协议实现或大型 Java Android 项目分层的开发者，这个仓库的 `terminal-emulator` 模块与 `TermuxService` 是绝佳教材；对普通用户而言，它仍然是把手机变成开发机的第一选择——前提是接受 F-Droid/GitHub 侧载的分发方式与 Android 12+ 的进程限制。
