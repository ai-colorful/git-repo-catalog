# LadybirdBrowser/ladybird 项目分析

> 星数 66633（快照 2026-10-24）｜主语言 C++｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：Ladybird 是一款从零开始、不基于 Chromium/WebKit/Gecko 任何现有引擎代码的"真独立"跨平台网页浏览器，自带完整的渲染引擎（LibWeb）与 JavaScript 引擎（LibJS）。

| 项目 | 内容 |
|---|---|
| 名称 | LadybirdBrowser/ladybird（由 SerenityOS 项目孵化的独立组织） |
| 星数 | 66,633（快照），分析当日 API 显示 66,768 |
| 主语言 | C++（C++23）+ Rust（约各占一半，Rust 迁移进行中） |
| 许可证 | BSD-2-Clause（`LICENSE`） |
| 默认分支 | master（`gh api repos/LadybirdBrowser/ladybird`） |
| 最近活跃 | push 时间 2026-10-31（分析当天仍在提交） |
| 状态 | pre-alpha，仅面向开发者（`README.md` 明示） |
| 发布 | GitHub 上无 tag / 无 release，滚动开发 |

## 2. 项目是什么，解决什么问题

浏览器市场长期被三大引擎（Blink、WebKit、Gecko）垄断，且"新浏览器"几乎都是 Chromium 换壳。Ladybird 要解决的问题是引擎层面的垄断与同质化：按照 Web 标准（WHATWG HTML/IDL/Fetch、CSS WG、ECMAScript 规范）从零实现一个完整引擎，让现代网页在非三大引擎的内核上也能正确渲染（`README.md`："a truly independent web browser, using a novel engine based on web standards"）。

项目前身是 SerenityOS 的浏览器移植版，2024 年独立为 Ladybird Browser Initiative 后完全切断了 SerenityOS 依赖，只保留从其继承的基础库与代码风格（`README.md`："many core library support components are inherited from SerenityOS"）。当前定位是 pre-alpha，作者明确表示不适合普通用户使用。

工程规模非常可观：C/C++/头文件 4,673 个约 83 万行，Rust 源码 1,476 个 `.rs` 文件约 91 万行（`find` 统计，含生成代码）。最大单体是 LibWeb（2,326 文件 / 41.3 万行），几乎按规范领域一对一组织了 90 个子目录：CSS、DOM、HTML、Layout、Painting、Fetch、WebGL、WebAudio、ServiceWorker、WebXR、IndexedDB、ViewTransition 等（`Libraries/LibWeb/` 目录列表）。

最值得关注的方向是进行中的 Rust 迁移：Cargo workspace 包含 26 个成员 crate（根 `Cargo.toml`），LibJS 的词法/语法分析、字节码、解释器、GC 与优化 JIT 已全部是 Rust 实现，HTML tokenizer、正则引擎、Unicode、URL、CSS/布局/绘制的关键路径也在逐步迁移（`Libraries/LibJS/Rust/`、`Libraries/LibWeb/Rust/`、`Libraries/LibWeb/HTML/Parser/Rust/Cargo.toml`）。

## 3. 技术栈与依赖

- **C++23 + Rust（1.98.0，`rust-toolchain.toml`）**：C++ 为主壳与尚未迁移的模块，Rust 承载 JS 引擎核心与渲染新架构；`Cargo.lock` 共 188 个 crate。
- **CMake（≥3.30）+ vcpkg 依赖管理**（`CMakePresets.json`、`vcpkg.json`），关键第三方库及用途：
  - `skia`（v148）：2D/GPU 光栅化后端，合成器进程的 DisplayList 播放器用它绘制（`Services/Compositor/DisplayListPlayerSkia.*`）
  - `angle`（chromium_7258 版）：WebGL → 原生 GPU API 转译（`Services/Compositor/HostWebGLContext.*`）
  - `ffmpeg` + `dav1d`：音视频解码（`vcpkg.json`、`Libraries/LibMedia`）
  - `qtbase` 6.11.2：桌面 UI 前端（`UI/Qt/`）
  - `icu`、`harfbuzz`、`freetype`：Unicode/排版/字体；`openssl`+`curl`：TLS/网络栈支撑
  - `mimalloc`、`simdjson`、`wuffs`、`libxml2`、`sqlite3`、`pdfjs`（PDF 查看器内嵌方案）
- **Python 3**：构建入口与 20+ 个代码生成器（`Meta/ladybird.py`、`Meta/Generators/`）。
- **自研基础层**：`AK/`（约 4.3 万行：字符串、智能指针、格式化、容器）、`LibCore`（事件循环）、`LibIPC`（进程间通信，`.ipc` 文本协议文件驱动代码生成）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `AK/` | 基础模板库（字符串/智能指针/容器/格式化），全项目公共底座 |
| `Libraries/` | 40+ 个静态库：LibWeb（引擎）、LibJS（JS 引擎）、LibGfx、LibWasm、LibMedia、LibCrypto/LibTLS、LibHTTP、LibUnicode、LibURL、LibIPC、LibGC、LibCompositing 等 |
| `Services/` | 各独立进程的可执行体：WebContent（渲染器）、RequestServer（网络）、ImageDecoder、Compositor（合成）、MediaServer、VideoDecoderServer、WebWorker、WebDriver |
| `UI/` | 浏览器前端：`UI/Qt/`（桌面）与 `UI/Android/`（Gradle 工程，有 nightly-android CI） |
| `Meta/` | 构建脚本（ladybird.py、CMake/）、代码生成器（Generators/）、WPT 运行器（WPT.sh）、Fuzzers、Linters、性能测量脚本 |
| `Tests/` | 508 个 C++ 测试文件 + 数万网页式测试资产（详见第 7 节） |
| `Documentation/` | 约 40 篇架构/贡献文档（ProcessArchitecture、LibWebFromLoadingToPainting、CodePolicy、Testing 等） |
| `WebCompat/` | 站点兼容性数据（目前仅 cnn.com.json、nytimes.com.json 两个站点修补） |
| `Utilities/` | 独立命令行工具：js（REPL）、test262-runner、dump-html-tree、wasm 等 |

组织方式为"单仓库 + 库/服务/前端分层"的单体工程：库层无进程边界，服务层把库组装成多进程浏览器，UI 层只做壳。LibWeb 内部按 Web 规范领域划分目录，IDL 接口（725 个 `.idl` 文件）与大量 `.ipc` 文件由 `Meta/Generators/` 生成绑定代码，手写代码与生成代码严格分离。

## 5. 核心架构与关键实现

### 5.1 多进程架构与引擎边界

浏览器被拆成多个 OS 进程（`Documentation/ProcessArchitecture.md`、`Services/` 各子目录）：每个标签页一个 WebContent 渲染进程（承载 LibWeb+LibJS），网络全部经 RequestServer 进程代理，图片解码在一次性 ImageDecoder 进程中完成，另有独立 Compositor 合成进程（`Services/Compositor/main.cpp`）。渲染进程带沙箱封装（`Services/RendererSandboxLinux.cpp` / `RendererSandboxMacOS.cpp`），Compositor 进程同样有 Sandbox 文件。进程间用 LibIPC：协议写成 `.ipc` 文本文件（如 `Services/WebContent/WebContentClient.ipc`），由 `Meta/Generators/generate_ipc_definitions.py` 生成两端代码。UI（Qt）与引擎之间只有 IPC 消息，这是崩溃隔离与沙箱化的基础。

### 5.2 LibWeb：从加载到绘制的管线

管线为 资源加载 → HTML 解析建 DOM → CSS 解析建 CSSOM → 样式计算（选择器匹配+级联）→ 布局树 → 显示列表 → 合成上屏（`Documentation/LibWebFromLoadingToPainting.md` 有逐步说明）。几个关键实现点：

- 样式计算做了"按最右复合选择器分桶缓存"的优化，只对可能命中的元素求值选择器（同上文档 Optimization notes 一节）。
- 布局/绘制正在 Rust 化：`Libraries/LibWeb/Rust/src/render_state.rs` 顶部注释说明"每份文档的 RenderState 由 StyleLayout 线程独占拥有"，宿主线程只能通过投递 job 与之交互——用 Rust 的所有权模型把布局树的线程安全写成类型约束；`paint_stage.rs`、`stage_thread.rs`、`layout/`、`css/` 均为对应 Rust 实现，C++ 侧经 `Libraries/LibWeb/Layout/LayoutRustBridge.h`、`Painting/PaintingRustBridge.h` 桥接。
- 绘制产出显示列表（Display List），跨进程交给合成器：`Libraries/LibCompositing/DisplayList/` 定义指令，`Services/Compositor/DisplayListPlayerSkia.cpp` 用 Skia/Vulkan 播放，配合 `VSyncScheduler.cpp`、`SkiaVulkanMemoryAllocator.cpp` 与光栅缓存（`DisplayListRasterCache.cpp`）构成 GPU 渲染管线；WebGL 经 ANGLE 出原生上下文。

### 5.3 LibJS：Rust 化的两层 JS 引擎

JS 引擎核心已迁移到 Rust（`Libraries/LibJS/Rust/src/`：lexer、parser、bytecode/、interpreter/、gc/、jit/），C++ 侧保留 `Runtime/` 中的标准库对象与宿主绑定。执行模型分两层（`Libraries/LibJS/JIT/ARCHITECTURE.md`，该文档与代码同步维护）：

> There is no baseline tier. The interpreter's caches are the baseline: the JIT translates what they saw into speculation, and leaves to the interpreter wherever a speculation fails.

即字节码解释器带内联缓存与反馈，热函数/热循环达到预算后由编译线程快照（Snapshot）→ SSA 图构建+内联 → 图简化 → 寄存器分配 → 代码生成 → 安装执行，支持循环回边 OSR 与生成器/async 恢复；投机失败则回退解释器。GC 为标记-清扫（同文档及 `Libraries/LibGC/`）。正则引擎在 `Libraries/LibRegex/Rust`，并从 V8/WebKit 移植测试（`Meta/import-v8-regexp-tests.py`、`import-webkit-regexp-tests.py`）。

### 5.4 代码生成驱动的规范实现

Web API 绑定不手写：725 个 `.idl` 文件经 `Meta/Generators/generate_libweb_bindings.py` 生成 C++ 绑定与 Rust 桥；CSS 属性枚举/描述符/伪类等同样由 20 余个生成器产出（`Meta/Generators/` 目录）。这使"跟规范对齐"变成了维护 IDL 与生成器的工作，也是 LibWeb 能以有限人力覆盖大量规范领域的原因之一。

## 6. 构建与运行

依赖 vcpkg 与 Rust 工具链，入口是 Meta/ladybird.py（`Documentation/BuildInstructionsLadybird.md`）：

```bash
# Debian/Ubuntu 依赖（文档节选，CMake≥3.30、Qt6、Ninja 必需）
sudo apt install build-essential ccache cmake ninja-build qt6-base-private-dev \
    qt6-positioning-dev qt6-tools-dev-tools qt6-wayland glslang-tools ...

# 配置+构建（自动拉 vcpkg）
Meta/ladybird.py run                      # Release 构建并启动浏览器
Meta/ladybird.py run ladybird --headless  # headless 模式
Meta/ladybird.py test                     # 跑测试
Meta/ladybird.py build js / build ImageDecoder 等  # 单个目标

# Rust 侧
cargo fmt / cargo clippy（CI 强制，根 Cargo.toml 中 clippy::all = "deny"）
```

其他辅助：`Meta/WPT.sh`（克隆并运行 web-platform-tests）、`Utilities/test262-runner.cpp`（跑 ECMAScript 官方测试集）、CMakePresets 提供 Release/Debug/Sanitizer 三种预设。

## 7. 测试与工程质量

- **测试规模（`Tests/` 统计）**：C++ 测试文件 508 个；LibWeb 的网页式测试资产量极大——Text 类 26,044 个、Ref 重绘对比测试 6,015 个、Layout 2,311 个、Crash 回归 746 个、Screenshot 261 个、Accessibility 130 个；LibJS 有 1,712 个 `.js` 测试。
- **规范符合性测试**：WPT（web-platform-tests）经 `Meta/WPT.sh` 集成，test262 有专属 runner 与 CI（`Utilities/test262-runner.cpp`、workflow `libjs-test262.yml`）。
- **CI 共 15 个 workflow**（`.github/workflows/`，经 gh api 确认）：build-and-test、build-release、lint-code、lint-commits、libjs-test262、nightly-android、nightly-lagom（在 Linux 上以单进程 Lagom 模式跑库测试）、web-benchmarks、js-and-wasm-benchmarks、macos-accessibility-tests 等，覆盖三大桌面平台。
- **代码质量机制**：Clang 插件（`Meta/ClangPlugins/`，用于检查 SerenityOS 风格约定）、Fuzzers（`Meta/Fuzzers/`）、Clippy 全量 deny 级 lint（根 `Cargo.toml`）、`Documentation/CodePolicy.md` 约束提交风格。

## 8. 社区与维护状态

- **极度活跃**：push 时间为分析当天（2026-10-31，gh api），最近 5 条提交横跨 LibDNS 测试、LibWeb 焦点/CSS、UI/Qt X11 修复，节奏为每日数十次的主干直接开发。
- **贡献者约 430 人**（contributors 分页 Link 头 last page=430，per_page=1），核心为全职受薪工程师加社区贡献者（组织受 Ladybird Browser Initiative 资助，见官网 ladybird.org；仓库内未含细节，标注为公开常识性背景）。
- **无版本发布**：GitHub 无 tag 无 release，属滚动主干；README 明示 pre-alpha。
- **文档文化突出**：Documentation/ 有 40 篇文档且明确要求与代码同步（如 `Libraries/LibJS/JIT/ARCHITECTURE.md` 自述"文档与代码不一致时二者必有其一有 bug"），另有 GC 堆可视化、性能剖析、Crash 报告等工作流文档。

## 9. 亮点与风险

**亮点**
1. 真正的引擎独立性：四大件（HTML 解析、CSS、JS、渲染合成）全部自研，是浏览器引擎层二十年来最大的从零实现项目之一。
2. 系统级的多进程+沙箱设计成熟：网络、图片解码、合成全部出进程，渲染进程沙箱化（`Services/`、`Documentation/ProcessArchitecture.md`），安全边界清晰。
3. 大胆且成体系的 Rust 迁移：JS 引擎核心、HTML tokenizer、渲染状态机已用 Rust 重写，用所有权模型解决布局树线程安全（`LibWeb/Rust/src/render_state.rs`），且与 C++ 桥接有明确工程规范（`Documentation/WrapperArchitecture.md`）。
4. 测试与 CI 体系完善：WPT + test262 + 数万自建测试 + 15 个 CI workflow + 三平台覆盖，pre-alpha 却有工业级质量基建。
5. 文档与代码同步的工程文化，架构文档（JIT、进程模型、渲染管线）可直接作为源码导读。

**风险**
1. 规范覆盖仍远未完成：90+ 个规范子目录的实现深度参差，README 自认 pre-alpha，现代重度 Web 应用可用性有限（`WebCompat/` 目前只有 2 个站点修补文件，侧面反映兼容问题主要靠"实现规范"而非"打补丁"）。
2. Rust/C++ 双语言迁移期成本：约一半代码在 Rust 且比例持续上升，桥接层（FFI、所有权交接）长期存在，贡献者需同时精通两套生态。
3. 构建门槛高：vcpkg + 188 个 Rust crate + Qt6 + ANGLE/Skia/FFmpeg 全量编译，资源需求与时间成本显著。
4. 无发布渠道、无稳定 API：普通用户与下游集成者当前没有可依赖的版本点。

## 10. 总结评价

Ladybird 是近年最雄心勃勃的开源基础软件项目之一：以浏览器引擎为单位、以 Web 规范为蓝图、以 C++/Rust 双栈和严格的代码生成流程为手段，重建了被三大引擎垄断的整个技术栈。66k+ 星反映的与其说是一件可用的产品，不如说是社区对"浏览器引擎必须独立"这一命题的投票。它当前的真实价值在于架构示范——多进程沙箱模型、显示列表 GPU 合成、带反馈字节码+优化 JIT 的两层 JS 引擎、以及一场教科书级的渐进式 Rust 迁移——都值得引擎/系统开发者精读。适合的人群：想深入理解浏览器内部机制的工程师（从 `Documentation/ProcessArchitecture.md` 与 `LibWebFromLoadingToPainting.md` 入手）、Rust/C++ 混合工程的实践观察者、以及愿意按 `Documentation/GettingStartedContributing.md` 上手贡献的开发者；不适合期待日常可用的终端用户。
