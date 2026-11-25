# LizardByte/Sunshine 项目分析

> 星数 32500（快照 2026-10-24）｜主语言 C++｜档位 20k-50k｜分析日期 2026-11-13

## 1. 项目速览

一句话定位：开源的 NVIDIA GameStream 协议服务端实现——在自家 PC 上运行 Sunshine，即可用任意 Moonlight 客户端（手机、电视盒子、Mac、浏览器等）低延迟串流玩本机游戏。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | LizardByte/Sunshine（已迁移至 SunshineStream/Sunshine，旧地址重定向） | GitHub API `full_name: SunshineStream/Sunshine`、README.md 徽章 |
| 星数 | 快照 32500（2026-10-24）；API 实测 43343（2026-11-13） | GitHub API |
| 主语言 | C++23（CUDA 文件 C++17） | `CMakeLists.txt` 第 25-30 行 |
| 许可证 | GPL-3.0（`PROJECT_LICENSE "GPL-3.0-only"`） | `CMakeLists.txt` 第 32 行、LICENSE |
| 默认分支 | master | GitHub API |
| 最近活跃 | 最后 push 2026-11-13（与本文分析同日）；最新 release v2026.1113.11031 发布于 2026-11-13 | GitHub API |
| 贡献者 | 约 227 人（contributors 分页计数） | GitHub API |
| 规模 | `src/` 根模块约 3.3 万行，平台后端另有约 3.3 万行（Linux 18k / Windows 10.5k / macOS 4k） | `wc -l` 统计 |

## 2. 项目是什么，解决什么问题

NVIDIA 曾通过 GeForce Experience 的 GameStream 功能让用户把 PC 游戏串流到 SHIELD 设备，Moonlight 社区则逆向出了该私有协议做出开源客户端。2023 年 NVIDIA 官方宣布移除 GameStream 后，服务端出现真空——Sunshine 正是这一真空的开源填补者：它实现了 GameStream 的服务端协议（HTTPS 控制端点 + RTSP 会话协商 + ENet 可靠控制流 + UDP 音视频广播），使 Moonlight 这一久经考验的客户端生态无需任何改动即可继续工作（`src/nvhttp.cpp`、`src/rtsp.cpp`、`src/stream.cpp`；README "Self-hosted game stream host for Moonlight"）。

与被替代的官方方案相比，Sunshine 的差异化在于：不锁定 NVIDIA——支持 AMD（AMF/VAAPI）、Intel（QuickSync/VAAPI）、NVIDIA（NVENC）以及各平台原生 API（macOS VideoToolbox、Windows Media Foundation/Qualcomm）和纯软件编码（`README.md` 编码 API 兼容表）；不锁定 Windows——Windows/Linux/FreeBSD/macOS 四平台覆盖，Linux 下提供 KMS/DRM、Wayland(wlroots)、X11、XDG Portal、KWin Screencast 等多种捕获后端；支持 HDR、YUV 4:4:4、AV1 等现代编码特性（`src/video.cpp` 编码器表）。

它同时是一个完整的"云游戏主机端"产品：内置 Web UI（Vue 3 SPA）完成配置、应用添加、配对管理；捕获并虚拟化手柄输入（Xbox/DualShock/DualSense/Switch Pro 多 Profile，`README.md` 手柄兼容表、`src/input.cpp`）；自动进行显示器模式切换（串流开始切分辨率/帧率、结束恢复，`src/display_device.cpp`）；mDNS 广播方便客户端发现主机（`main.cpp` 第 481 行 `platf::publish::start()`）。

商业模式上，项目属于 LizardByte 社区，但手柄虚拟化已转为"Virtual HID Broker"独立授权组件（`CMakeLists.txt` 第 13 行 `LIBVIRTUALHID_MINIMUM_VERSION`、main.cpp 中 `lvh::get_license_status()`），核心串流功能保持 GPL 开源。

## 3. 技术栈与依赖

- **核心语言**：C++23（`CMakeLists.txt`），少量 CUDA `.cu`（`src/platform/linux/cuda.cu`）与 Objective-C/++（macOS `*.mm`，如 `src/platform/macos/av_video.m`）。
- **构建**：CMake ≥3.24 + CPM 依赖管理（`cmake/cpm/CPM.cmake`、`package-lock.cmake` 锁版本）；另有 `package.json`/`vite.config.js` 构建 Web UI，`pyproject.toml`/`uv.lock` 支撑文档与打包脚本。
- **关键第三方库**（`.gitmodules` 列出 16 个子模块，另经 CPM 引入）：
  - FFmpeg（libavcodec 硬件编码入口，VAAPI/Vulkan/Media Foundation 走 avcodec 抽象，见 `src/video.cpp` 的 `encoder_platform_formats_avcodec`）；
  - NVENC SDK（`src/nvenc/` 自研封装，含 D3D11、CUDA interop、动态加载工厂）；
  - libopus（多声道音频编码，`src/audio.cpp`）；
  - boost（asio/log/process）与 Simple-Web-Server（LizardByte fork，HTTPS 服务，`main.cpp` 中注明"待更新或替换"）;
  - moonlight-common-c（协议公共结构）、libdisplaydevice（显示器操作）、nvapi、glad、wayland/wlr/plasma 协议、TPCircularBuffer、tray（托盘）。
- **前端**：Vue 3.5 + Vite 8 + Bootstrap 5.3 + vue-i18n + vitest（`package.json`）。
- **测试**：GoogleTest（经 lizardbyte-common 子模块引入，`tests/CMakeLists.txt`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `src/`（根级 ~30 个模块） | 协议与编排核心：`nvhttp`(GameStream HTTPS)、`rtsp`(会话协商)、`stream`(音视频会话)、`video`/`audio`(编码管线)、`input`(输入注入)、`confighttp`(Web UI API)、`config`、`crypto`、`display_device`、`upnp`、`system_tray` 等 |
| `src/nvenc/` | 自包含的 NVENC 封装层（D3D11 / D3D11-on-CUDA / 动态 DLL 工厂），约 30 个文件 |
| `src/platform/linux/` | 捕获后端：`kmsgrab`、`wlgrab`、`x11grab`、`portalgrab`、`kwingrab`；`vaapi`/`vulkan_encode`/`cuda` 编码；`pipewire` 音频 |
| `src/platform/windows/` | `display_base`/`display_ram`/`display_vram`/`display_wgc`（DXGI 复制与 Windows.Graphics.Capture）、WASAPI 音频、`nvprefs/`（NVIDIA 控制面板设置修改与崩溃恢复） |
| `src/platform/macos/` | ScreenCaptureKit 捕获（`sc_capture`）、VideoToolbox 编码（`vt_encoder`）、CoreAudio |
| `src/platform/common.h` | 平台抽象接口（`platf::display_t::capture()` 纯虚函数等） |
| `src_assets/common/assets/web/` | Vue 3 管理界面源码（37 个 .vue 组件） |
| `tools/` | 辅助二进制：`sunshinesvc.cpp`(Windows 服务)、`dxgi.cpp`、`audio.cpp` |
| `cmake/`、`packaging/`、`docker/` | CMake 模块（prep/targets/packaging）、各发行版打包（deb/rpm/Flatpak/AppImage/Homebrew/FreeBSD port）、容器构建 |
| `tests/` | gtest 单元测试 20+ 文件 + 集成测试 5 文件 + Web 测试 |
| `docs/` | 用户文档（`configuration.md`、`building.md`、`api.md`、`changelog.md`、`awesome_sunshine.md`） |
| `.github/workflows/` | 27 个 CI workflow（详见第 7 章） |

组织方式：典型的"CMake 单体 + 平台子目录 + 子模块生态"。跨平台抽象不靠中间件，而是靠 `src/platform/common.h` 定义接口、各平台目录提供实现、`src/video.cpp`/`src/audio.cpp` 在运行时探测选择——这是性能敏感流媒体软件的务实做法。

## 5. 核心架构与关键实现

### 5.1 启动编排与服务线程模型（src/main.cpp）

`main()` 按序完成：配置解析 → 日志 → 显示器状态守护（`display_device::init`，防串流后黑屏）→ 平台初始化 → 输入/编码器探测（`video::probe_encoders`）→ HTTP 栈初始化，随后启动三个常驻服务线程（第 495-497 行）：

```cpp
std::jthread httpThread {nvhttp::start};      // GameStream HTTPS, 默认 47989
std::jthread configThread {confighttp::start}; // Web UI API
std::jthread rtspThread {rtsp_stream::start};  // RTSP 会话协商, 默认 48010
```

Linux 上有精细的特权管理：先在持权时启动 privileged DRM/EGL worker 线程，再逐级 drop capabilities（`main.cpp` 第 208-227 行），兼顾 KMS 捕获权限与最小化攻击面。Windows 上则通过隐藏窗口接收会话注销消息并修改 NVIDIA 控制面板设置（`main.cpp` 第 300-385 行）。

### 5.2 GameStream 配对协议（src/nvhttp.cpp, src/crypto.cpp）

复刻 NVIDIA GameStream 的 PIN 配对握手：客户端在 Web UI 提交 4 位 PIN（`is_valid_pairing_pin` 限定 4 位数字，第 655 行），服务端以 PIN+盐值派生 AES 密钥（`getservercert`，第 752 行起：`crypto::gen_aes_key(salt, pin)`），依次走 `clientchallenge` → `serverchallengeresp` → `clientsecret` 阶段，签名用服务端 X509 证书完成（第 894-896 行拼装 serverchallenge + x509_sign + secret）。配对状态机有超时（`PAIRING_SESSION_TIMEOUT`）与乱序保护（`fail_pair(sess, tree, "Out of order call to ...")`）。`launch/resume/cancel` 端点在此创建 `launch_session_t` 并交给 RTSP 服务。

### 5.3 串流管线：捕获 → 编码 → 分片 + FEC → UDP（src/stream.cpp, src/video.cpp, src/audio.cpp）

- **捕获抽象**：`src/platform/common.h` 第 768 行定义纯虚 `capture(push_captured_image_cb, pull_free_image_cb, cursor)`，各平台 display_t 实现推拉式帧交换；`video.cpp` 用 `PARALLEL_ENCODING` 能力位标记"捕获与编码可并行"的后端（第 405 行）。
- **编码器注册表**：`src/video.cpp` 第 1290 行起以 `encoder_t` 表声明各后端：`vaapi`（av1/hevc/h264_vaapi 三个 codec 条目）、`vulkan`（`SUNSHINE_BUILD_VULKAN` 条件编译）、NVENC、Media Foundation、VideoToolbox、`libx264` 软编；启动时 `probe_encoders()` 逐一试编选出可用者。
- **发送侧**：`videoBroadcastThread`（`stream.cpp` 第 1610 行）从 mail 队列取帧，对 IDR 帧先做参数集替换（SPS/VPSS 重写），打上 `video_short_frame_header_t`，按 `config.packetsize` 分片为 NV_VIDEO_PACKET，再用 Reed-Solomon 编码出 parity 分片（第 933 行 `reed_solomon_new(data_shards, parity_shards)`、第 935 行 `reed_solomon_encode`）；音频同理走 RTP + RS-FEC（第 1935 行 `RTPA_DATA_SHARDS/RTPA_FEC_SHARDS`）。发送线程显式调高优先级 `platf::adjust_thread_priority(high)`。
- **控制流**：ENet 可靠通道承载控制报文，`stream.cpp` 第 381 行的 control_server_t 以 `map(type, callback)` 注册各控制消息处理器；输入报文路由至 `src/input.cpp`（2602 行，含手柄虚拟化、键盘、鼠标、触控笔）。
- **音频**：`src/audio.cpp` 用 opus_multistream 按 2.0/5.1/7.1 配置编码（第 52-98 行 `stream_configs`），固定 CBR（第 132 行 `OPUS_SET_VBR(0)`）以匹配协议预期。

### 5.4 Web UI 管理面（src/confighttp.cpp, src_assets/common/assets/web/）

后端为 REST API（`/api/*`，带 CSRF token 机制，`confighttp.cpp` 第 957、1060 行），端点覆盖：应用管理（GET/POST/DELETE `/api/apps`）、配对（`/api/pin`、`/api/clients/unpair-all`）、配置读写（`/api/config`）、日志查看、密码修改、显示器持久化重置（`/api/reset-display-device-persistence`）、重启等。前端为 Vue 3 SPA（`App.vue`/`Pin.vue`/`Apps.vue`/`Troubleshooting.vue` 等 37 个组件，`vite.config.js` 构建，`vitest` 测试），经 CrowdIn 本地化（`crowdin.yml`）。

## 6. 构建与运行

文档化路径（`docs/building.md`、`AGENTS.md`、CI workflow 同构）：

```bash
# Linux（CMake + CPM 自动拉取依赖；Ninja 可选）
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
sudo cmake --install build

# Windows（AGENTS.md 指定 msys2 ucrt64）
C:\msys64\msys2_shell.cmd -defterm -here -no-start -ucrt64 -c \
  "cmake -B cmake-build-debug && cmake --build cmake-build-debug"

# 测试（gtest，可执行文件名固定为 test_sunshine）
cmake --build build --target test_sunshine && ./build/tests/test_sunshine

# Web UI
npm run build        # vite build
npm test             # vitest run --coverage
```

常用 CMake 选项（`cmake/prep/options.cmake`，28 个 option）：`BUILD_DOCS`、`BUILD_TESTS`、`SUNSHINE_ENABLE_TRAY`、`SUNSHINE_CONFIGURE_ONLY`（只生成打包清单）、各打包格式开关（`SUNSHINE_BUILD_FLATPAK/APPIMAGE/HOMEBREW` 等）。另有官方 Docker 镜像（`docker/`、`docker-bake.hcl`）与 Flathub/winget/Homebrew 分发。

## 7. 测试与工程质量

- **测试规模**：单元测试 20+ 文件（`tests/unit/`，覆盖 config/crypto/pairing/nvenc/input/audio/display_device 等核心模块，含平台子目录），集成测试 5 个（`tests/integration/`：外部命令执行、配置一致性、版本一致性校验等），另有 Web 前端 vitest 测试（`vitest.config.js`）。测试框架 GoogleTest（`tests/CMakeLists.txt`）。
- **质量工程化程度高**：AGENTS.md 规定"新增/修改源码必须附回归测试，目标对改动代码 100% 覆盖"；CI 接入 SonarCloud 质量门禁、Codecov 覆盖率、CodeQL（README 徽章与 `.github/workflows/_codeql.yml`）。
- **CI 矩阵罕见地庞大**：27 个 workflow 文件（`.github/workflows/`），按发行版拆分——alpine、archlinux、freebsd、flatpak、homebrew、copr（ Fedora rpm）、cloudsmith（deb/rpm 仓库），加上核心 ci-linux/macos/windows/web、release 流水线与 changelog/docs/flathub/winget 自动更新机器人。开源 C++ 项目里这种"每发行版独立流水线 + 自动打包分发"的完备度属第一梯队。
- **工程纪律**：clang-format 统一风格、Doxygen 注释（核心文件均有 `@file/@brief` 头）、renovate 依赖自动升级、CrowdIn 翻译流水线。

## 8. 社区与维护状态

- **极度活跃**：最后 push 与最新 release（v2026.1113.11031，日历版本号）均落在分析当日 2026-11-13（GitHub API），维护节奏为"随时合并、按日发版"的 calver 模式。
- **组织迁移进行中**：GitHub API `full_name` 返回 `SunshineStream/Sunshine`（repo id 438803890 不变，旧 URL 重定向），README 徽章已全部指向 SunshineStream/Sunshine，`package.json` repository 字段亦同——项目正从 LizardByte 迁往独立的 SunshineStream 组织（原因未在仓库内文档中确认）。
- **贡献者约 227 人**（contributors API 分页计数），issue 数 167（API `open_issues`）。社区基础设施完善：CrowdIn 多语言、sponsor 页面、`docs/awesome_sunshine.md` 生态清单。
- **生态自持**：LizardByte/SunshineStream 维护着配套子模块链（libdisplaydevice、libvirtualhid、build-deps、tray、lizardbyte-common，见 `.gitmodules`），关键协议组件不依赖单一上游。

## 9. 亮点与风险

**亮点**
1. 协议逆向+工程化双料标杆：完整复刻 GameStream 服务端（HTTP 配对/RTSP/ENet/RS-FEC），让 Moonlight 全平台客户端零成本复用，接管了 NVIDIA 退出后的生态位（`src/nvhttp.cpp`、`src/rtsp.cpp`、`src/stream.cpp`）。
2. 硬件编码覆盖面业内最全：NVENC/AMF/QuickSync/VAAPI/VideoToolbox/Vulkan Video/Media Foundation/软件编码统一进 `encoder_t` 注册表，启动自动探测（`src/video.cpp`）。
3. 平台捕获后端矩阵完整（KMS/DRM、Wayland、X11、Portal、KWin、DXGI、WGC、ScreenCaptureKit），Linux 特权与 capabilities 处理细致（`main.cpp` 第 208-227 行）。
4. 工程质量第一梯队：27 条 CI 流水线覆盖 5+ 发行版打包，SonarCloud+CodeQL+Codecov，测试覆盖要求写入 AGENTS.md。
5. 发版激进（日历版本号按日发版），问题修复落地快。

**风险/局限**
1. 核心依赖链偏"重"：16 个 git 子模块 + CPM 下载，离线/受限网络构建门槛高；Simple-Web-Server 被官方标注"需要更新或替换"（`main.cpp` 第 490 行 FIXME）。
2. 手柄虚拟化依赖单独授权的 Virtual HID Broker（`CMakeLists.txt` 第 13 行、`lvh::get_license_status()`），部分功能（DS4/DS5/Switch Pro 原始 HID 报告）与商业授权绑定，与纯开源预期有落差。
3. 协议本质是逆向实现，GameStream 协议无公开规范，客户端侧演进（Moonlight 变更）或平台 API 变动可能造成隐性破坏。
4. 代码复杂度集中：`video.cpp`(3858 行)、`confighttp.cpp`(2647 行)、`input.cpp`(2602 行) 单文件偏大，平台差异叠加预处理宏，新贡献者上手成本高。

## 10. 总结评价

Sunshine 是"被大厂砍掉的功能由社区完美接盘"的教科书案例：以逆向的 GameStream 协议为底座，把多平台捕获、全厂商硬件编码、输入虚拟化、Web 管理面、五发行版打包分发整合成一个开箱即用的云游戏主机端，并保持着按日发版、CI 覆盖发行版级别的维护强度，星数也在一年内从 32.5k 涨至 43k 以上。适合想在自己 PC 上搭建串流服务（配合 Moonlight 客户端）的终端用户；对想学习低延迟音视频管线（捕获-编码-分片-FEC-传输全链路）、跨平台 CMake 工程组织或协议逆向工程化落地的开发者，其 `src/stream.cpp`、`src/video.cpp` 与 `src/platform/` 是极佳的实战教材。阅读源码前建议先从 `docs/building.md` 与 `src/platform/common.h` 的接口层入手。
