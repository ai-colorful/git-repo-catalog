# Genymobile/scrcpy 项目分析

> 星数 152949（快照 2026-10-24）｜主语言 C｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

scrcpy（读作 "screen copy"）是一个在电脑上通过 USB 或 TCP/IP 镜像并控制 Android 设备的命令行工具：显示视频与音频、用键鼠操控设备，无需 root、无需在设备上安装任何 App。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | scrcpy（v5.0.1） | `README.md`、`meson.build` |
| 星数 | 152,949（fork 14,088） | 任务快照 + gh api |
| 主语言 | C（客户端）+ Java（设备端服务） | `app/src`、`server/src` |
| 许可证 | Apache-2.0 | `LICENSE` |
| 最近活跃 | 最近 push 2026-10-26；最新 release v5.0.1（2026-10-08） | gh api `repos/Genymobile/scrcpy` |
| 平台 | Linux / Windows / macOS | `README.md` |

## 2. 项目是什么，解决什么问题

scrcpy 解决的核心问题是"把 Android 设备的屏幕和声音实时搬到电脑上，并用电脑键鼠操控设备"。它由 Genymobile（Android 模拟器厂商）开发，最初为内部调试服务，后发展为通用投控工具。`README.md` 明确其设计目标：轻量（原生应用、只显示设备画面）、高性能（30~120fps）、低延迟（35~70ms）、约 1 秒启动、非侵入（设备上不留任何东西）、无账号无广告无联网。

实现上的关键洞察是利用 Android `adb shell` 自带的 `shell` 用户权限：屏幕采集所需的特权正是 `shell` 用户拥有的特权（`doc/develop.md` "Privileges" 一节），因此既不需要 root 也不需要装 App。客户端把一个约几十 KB 的 Java 服务端 jar 推到 `/data/local/tmp`，用 `app_process` 以 `shell` 身份启动，采集/编码/注入全部在设备端完成，电脑端只负责解码显示与输入捕获；进程结束即一切消失，天然满足"非侵入"。

功能覆盖远超普通投屏：音频转发（Android 11+）、录像、虚拟显示、息屏镜像、双向剪贴板、摄像头镜像（Android 12+）、Linux V4L2 虚拟摄像头、HID 物理键盘/鼠标/手柄、OTG 模式（无需 adb 也能当键鼠用）（`README.md` 功能列表与 `doc/` 目录各专题文档）。

项目量级：客户端 C 代码约 18,900 行（`app/src` 184 个 .c/.h），服务端 Java 约 12,000 行（98 个 .java，`wc -l` 统计）。15 万星使其成为 GitHub 上最受欢迎的投控方案，也是"设备端无需安装"这一技术路线的代表作品。

## 3. 技术栈与依赖

**客户端（C11，meson/ninja 构建）**
- SDL3（>= 3.2.0）：窗口、渲染、输入事件（`app/meson.build:150`）
- FFmpeg（libavformat >= 60.3 / libavcodec >= 60.3 / libavutil >= 58.2 / libswresample）：解封装、解码、重采样（`app/meson.build:146-149`）
- libusb-1.0：USB 直连 HID（AOA）与 OTG 模式（`app/meson.build:174`）
- libdrm / libva（可选，Linux）：V4L2/VA-API 相关（`app/meson.build:158-165`）
- OpenGL/EGL：硬件解码帧直接上屏（`app/src/egl.c`、`app/src/opengl.c`）

**服务端（Java，Gradle 构建）**
- 编译目标为未签名 APK 改名 `scrcpy-server.jar`，内含 classes.dex（`doc/develop.md`）
- 使用 Android 隐藏 API：MediaCodec（硬编码）、SurfaceControl（虚拟显示）、AudioPlaybackCapture（内录），全部经反射封装（`server/src/main/java/com/genymobile/scrcpy/wrappers/`）
- AIDL 接口：`server/src/main/aidl/android/`（剪贴板监听、显示窗口监听）

**构建脚本**：`meson.build`（客户端）、`build.gradle` + `server/build.gradle`（服务端）、`server/build_without_gradle.sh`（无 Gradle 兜底）、`cross_win32/64.txt`（MinGW 交叉编译 Windows 包）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `app/src/` | C 客户端全部源码（约 18900 行） |
| `app/src/adb/` | adb 进程封装、设备列表解析、隧道建立（forward/reverse） |
| `app/src/usb/` | USB AOA HID（`aoa_hid.c`）、OTG 模式（`scrcpy_otg.c`）、手柄/键鼠 AOA |
| `app/src/uhid/` | UHID 虚拟输入设备（需 Android 11+） |
| `app/src/util/` | 通用库：线程池、net_intr、vecdeque、audiobuf、环形缓冲等 |
| `app/src/sys/`、`app/src/interop/` | 平台层（win/unix 进程、串口互操作） |
| `app/tests/` | 10 个纯逻辑 C 测试（见第 7 章） |
| `server/src/main/java/com/genymobile/scrcpy/` | 设备端服务（约 12000 行），按 audio/control/device/display/video/wrappers/util 分包 |
| `server/src/test/` | 服务端 JUnit 单测 |
| `doc/` | 22 篇用户/开发者文档（connection、control、video、audio、tunnels、develop 等） |
| `release/` | 各平台打包脚本（`build_linux.sh`、`build_windows.sh` 等） |
| `.github/workflows/release.yml` | CI/发布流水线 |

组织方式评述：典型的"客户端-设备端双进程"单体仓库，两半各自独立构建（meson + gradle），靠自定义二进制协议通信；C 侧以 `sc_xxx` 前缀结构体 + 回调做手工面向对象（trait 模式，见 `app/src/trait/`），模块边界清晰，几乎每个 `.c` 对应一个 `.h`，风格高度一致（Genymobile 代码规范要求下划线小写命名、4 空格缩进）。

## 5. 核心架构与关键实现

### 5.1 启动与连接：推送服务端 + 网络角色反转

客户端启动时把 jar 推到 `/data/local/tmp/scrcpy-server.jar`（`app/src/server.c:21,67`），再拼出 adb 命令执行（`app/src/server.c:218-228`）：

```c
cmd[count++] = "shell";
cmd[count++] = "CLASSPATH=" SC_DEVICE_SERVER_PATH;
cmd[count++] = "app_process";
cmd[count++] = "/"; // unused
cmd[count++] = "com.genymobile.scrcpy.Server";
cmd[count++] = SCRCPY_VERSION;
```

连接设计很讲究（`doc/develop.md`）：默认做网络角色反转——客户端先开本地 socket 监听，再让设备端 `app_process` 主动连回来，避免"先启动谁"的竞态，无需轮询；`--force-adb-forward` 才退回 adb forward 模式。视频、音频、控制使用**独立 socket**（1~3 个，可单独关闭），每个 socket 在客户端与服务端各有专用线程。`scid` 参数把多个 socket 归属到同一次会话。

### 5.2 服务端采集编码管道（Java）

`server/src/main/java/com/genymobile/scrcpy/Server.java` 中，控制、音频、视频各自封装为 `AsyncProcessor` 并行运行，主线程 `Looper.loop()` 等待完成。视频链路：`ScreenCapture`（或 `CameraCapture`/`NewDisplayCapture`）→ `SurfaceEncoder`。屏幕镜像用 `SurfaceControl.createDisplay("scrcpy", secure)` 建虚拟显示并绑定 Surface（`ScreenCapture.java:210-223`），编码用设备硬编码器 MediaCodec（`SurfaceEncoder.java:75-133`），默认 H.264（可 H.265/AV1/VP8/VP9）。音频链路：`AudioPlaybackCapture`（系统内录，Android 11+）或 `AudioDirectCapture`（麦克风等直采），经 `AudioEncoder`（Opus/AAC/FLAC）或 `AudioRawRecorder`（PCM 直传）输出（`Server.java:103-127`）。

### 5.3 流协议与客户端解码管道（C）

自定义帧封装极简：每帧 12 字节头 = 8 字节大端 `pts|flags`（bit62=配置帧、bit61=关键帧，低位为 PTS）+ 4 字节大端负载长度（`app/src/demuxer.c:12-14,69-141`）；编码 ID 用 ASCII 字面量（"h264"、"opus" 等，`app/src/demuxer.c:15-25`）。客户端 `sc_demuxer` → `sc_decoder` → `sc_screen`（SDL3 渲染）分线程接力，`sc_recorder` 可并行落盘 mp4/mkv；解码帧"收到即显示"，不缓冲，以此换取最低延迟（`doc/develop.md`）。EGL 路径让硬件解码帧零拷贝上屏（`app/src/egl.c`）。

### 5.4 控制注入：双向控制通道 + 隐藏 API 反射

控制 socket 是唯一双向通道。客户端把 SDL 事件序列化为紧凑二进制消息（23+ 种类型：INJECT_KEYCODE、INJECT_TOUCH_EVENT、SET_CLIPBOARD、UHID_INPUT、START_APP 等，`app/src/control_msg.h:30-52`），例如按键消息仅 14 字节（`app/src/control_msg.c`）。服务端 `Controller` 解析后经 `InputManager` 反射调用隐藏 API 注入：

```java
Method method = getInjectInputEventMethod();
return (boolean) method.invoke(manager, inputEvent, mode);
```

（`server/src/main/java/com/genymobile/scrcpy/wrappers/InputManager.java:41-51`）

对小米等设备的 `INJECT_EVENTS` 权限错误做了专门提示与限流日志（同文件）。HID 模式（AOA over USB / UHID）走另一条路：把键盘鼠标作为真实 USB/HID 设备注册，输入由系统原生处理，兼容依赖输入指纹的 App（`app/src/usb/aoa_hid.c`、`server/.../control/UhidManager.java`）。设备→客户端方向还回传 `DeviceMessage`（剪贴板变化、APK 崩溃信息等，`server/.../control/DeviceMessage.java`）。音频延迟由 `sc_audio_regulator` 用滑动平均维持目标缓冲量，靠微调播放时钟对齐视频（`app/src/audio_regulator.h:17-20`）。

## 6. 构建与运行

客户端（`doc/build.md`，需先装 `gcc git pkg-config meson ninja-build libsdl3-dev libavformat-dev ...`，见 `doc/build.md:56`）：

```bash
meson setup x --buildtype=release --strip -Db_lto=true
ninja -Cx          # DO NOT RUN AS ROOT
sudo ninja -Cx install
```

服务端：`./gradle 与 release/build_server.sh`；CI 同时验证 `server/build_without_gradle.sh`（`.github/workflows/release.yml` job `test-build-scrcpy-server-without-gradle`）。

日常使用（`README.md`）：

```bash
scrcpy                       # 默认投屏
scrcpy --record=file.mp4     # 投屏并录像
scrcpy --no-video --audio-source=mic   # 只转发麦克风
scrcpy --video-codec=h265 --max-size=1280
```

运行要求：设备 Android 5.0+（API 21），音频转发需 Android 11+，摄像头镜像需 Android 12+（`README.md` Prerequisites）；开发版可直接 `./run <args>` 一键构建并运行。

## 7. 测试与工程质量

- 测试规模：客户端 `app/tests/` 10 个测试文件，覆盖纯逻辑单元（adb 输出解析 `test_adb_parser.c`、CLI 参数 `test_cli.c`、控制消息序列化 `test_control_msg_serialize.c`、设备消息反序列化、audiobuf、字符串等），自写断言宏风格，无重型框架；服务端 JUnit 测试集中在协议与工具类（`ControlMessageReaderTest`、`DeviceMessageWriterTest`、`CommandParserTest` 等，`server/src/test/`）。没有端到端自动化测试（真机交互难以 CI 化，属合理取舍）。
- CI：`.github/workflows/release.yml`（workflow_dispatch 触发）串联：服务端单测 → 服务端构建 → 无 Gradle 构建验证 → 各平台客户端构建打包（`release/test_client.sh`、`build_linux.sh` 等），发布产物附校验和（`release/generate_checksums.sh`），README 提供 release 验证文档（`doc/verify-release.md`）。
- 代码质量观察：C 侧一致性极佳——统一 `sc_` 前缀、`include "xxx.h"` 直排、断言驱动；`meson.build` 开启 `warning_level=2`、`-Wmissing-prototypes`、`b_ndebug=if-release`（`meson.build:1-10`）。服务端用反射封装层隔离全部隐藏 API 变动，跨 Android 版本兼容逻辑集中（`wrappers/` + `Workarounds.java`）。

## 8. 社区与维护状态

- 维护方：Genymobile 官方团队，核心作者 rom1v（Romain Vimont）长期全职维护；贡献者约 187 人（gh api contributors 分页计数，2026-10-30）。
- 活跃度：最近 push 2026-10-26（gh api），当前为 v5.0.1（`meson.build` version 字段一致）；维护节奏以" yearly 大版本 + patch "为主，v5.0.1 发布于 2026-10-08。
- 社区形态：open issues 约 2,943（含大量问题咨询与设备适配报告，gh api）；`README.md` 顶部显著警告"本仓库是唯一官方来源，勿从第三方网站下载"，并提供 `doc/verify-release.md` 的签名校验流程——这是被大量仿冒后的反制措施。
- 商业背景：作者另有付费分支项目（未在本仓库内，未确认），本仓库本体保持完全开源。

## 9. 亮点与风险

**亮点**
1. "零安装投控"技术路线教科书：只用 shell 权限 + `app_process` 实现完整投控，非侵入性与安全性兼得（`doc/develop.md`）。
2. 协议设计克制而精准：12 字节帧头、字节级控制消息、多 socket 分流，延迟压到 35~70ms（`app/src/demuxer.c`、`doc/develop.md`）。
3. C 侧工程化程度罕见地高：trait 回调模式、intr 可中断 IO、meson 严格告警，18k 行代码无框架却井然（`app/src/util/intr.c`、`app/src/trait/`）。
4. 兼容性工程完善：隐藏 API 全部反射封装，Android 5.0~15+、三家桌面系统、Gradle/无 Gradle 双构建通道。
5. 文档质量一流：22 篇专题文档 + `doc/develop.md` 架构总览，几乎每个设计决策都有出处。

**风险/局限**
1. 深度依赖 Android 隐藏 API，厂商定制 ROM 或未来系统版本可能破坏注入/内录（README 中小米权限设置提示即为例证）。
2. 高级输入（HID/OTG）在部分设备上受厂商驱动限制，OTG 模式无视频能力（`doc/otg.md`）。
3. 网络方案默认依赖 adb，仅 TCP/IP 无自建信令，不适合大规模集群场景的远控改造（`doc/connection.md`）。
4. C 客户端手工并发（多线程 + 回调 + 手工生命周期）可读性依赖规范，新贡献者上手门槛偏高。

## 10. 总结评价

scrcpy 是"用最少权限做最难的事"的典范项目：以 adb shell 权限为支点，用一套自研精简协议打通设备端采集编码与主机端解码渲染，实现了无 root、无安装、低延迟的完整 Android 投控，15 万星与其工程质量完全匹配。它适合所有需要在 PC 上操作 Android 设备的开发者、测试与普通用户直接使用；对想学习 Android 反射封装、SurfaceControl/MediaCodec 体系、低延迟音视频管道或 C 语言大型工程组织的开发者，其源码（尤其 `doc/develop.md` 配合 `Server.java` 与 `app/src/scrcpy.c` 对读）是极少数值得精读的开源范本。
