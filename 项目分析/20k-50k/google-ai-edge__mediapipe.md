# google-ai-edge/mediapipe 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 C++｜档位 20k-50k｜分析日期 2026-11-17

## 1. 项目速览

一句话定位：Google 出品的跨端（Android/iOS/Web/桌面）多模态 ML 管线框架，底层是 Calculator 图流水线引擎，上层封装为开箱即用的人脸/手势/姿态/分割等检测任务 API。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | google-ai-edge/mediapipe（2024 年从 google/mediapipe 迁至 google-ai-edge 组织，GitHub 自动重定向；仓库 created_at 保留 2019-06-13） | gh api repos/google-ai-edge/mediapipe |
| 星数 | 39500（快照）；gh api 当前 37716 | 任务参数 / gh api |
| 语言 | C++ 为主（1281 个 .cc + 869 个 .h，约 44.4 万行），另含 Python/Java/Swift/TS 绑定 | 源码统计 |
| 许可证 | Apache-2.0（LICENSE） | LICENSE |
| 默认分支 | master，最近 push 2026-11-16（当天，极活跃） | gh api |
| 当前版本 | 1.2.4（开发中，`mediapipe/version.bzl`）；v1.0.0 于 2026-07-28 发布 | mediapipe/version.bzl、gh api releases |
| 规模 | 解压后 83MB，4793 个文件，仓库总大小约 600MB | tarball 统计 / gh api |

## 2. 项目是什么，解决什么问题

MediaPipe 要解决的问题是把"实时多模态 ML 推理"从研究原型变成能在手机、浏览器、桌面、边缘设备上高效跑起来的产品能力。它的核心思路来自两层次设计（README.md）：

**底层 MediaPipe Framework**（C++）：一个图流水线引擎。开发者把处理步骤（Calculator）声明成 DAG（`.pbtxt` 图配置或 C++ Builder），框架负责跨线程调度 packet、管理背压、时间戳同步、GPU/CPU 数据搬运。这个模型天然适合"摄像头帧进来 → 检测 → 跟踪 → 渲染"这类实时流媒体管线，也是它区别于普通推理库（如直接调 TFLite）的关键。

**上层 MediaPipe Solutions / Tasks**：预先搭好的图 + 预训练模型，封装成各平台一致的任务 API。README 列出 audio/decision/genai/retrieval/text/vision 六大类；Python 端 vision 就有 11 个 API（face_detector、face_landmarker、hand_landmarker、pose_landmarker、holistic_landmarker、gesture_recognizer、image_classifier、image_segmenter、interactive_segmenter、image_embedder、object_detector，见 `mediapipe/tasks/python/vision/`）。隐私声明（README.md）强调全部推理在设备端完成，数据不出设备。

代码中保留了两代 API 的完整痕迹：旧 "Solutions"（`mediapipe/python/solutions/` 的 face_mesh/hands/holistic/pose 等、`mediapipe/graphs/` 与 `mediapipe/modules/` 的旧图、`mediapipe/examples/android/solutions/`）已被新 Tasks API（`mediapipe/tasks/`）逐步取代，但旧版仍在维护，兼容存量用户。

## 3. 技术栈与依赖

- **核心**：C++17 + Abseil（absl::Status/Packet/Mutex 贯穿全部头文件，如 `mediapipe/framework/packet.h`、`calculator_graph.h`）+ Protobuf（333 个 .proto，图配置与选项全部 proto 化）。
- **推理引擎**：与 TensorFlow Lite（现更名 LiteRT）深度耦合——`mediapipe/calculators/tflite/`、`mediapipe/calculators/tensorflow/`、`mediapipe/tasks/cc/core/mediapipe_builtin_op_resolver.cc`（自定义 op resolver）；`WORKSPACE` 精确 pin org_tensorflow commit `a481b102...`，并注释说明与 `@litert`（LiteRT v2.3.0）的 XNNPACK 版本对齐。
- **构建**：Bazel 7.7.0（`.bazelversion`），MODULE.bazel（bzlmod）+ WORKSPACE 双轨；rules_apple/rules_swift/rules_kotlin 支持 iOS/Android 目标。
- **Python 包**：`setup.py`（Bazel 编译静态 OpenCV 打进 wheel）、`requirements.txt`：numpy、opencv-contrib-python、absl-py、flatbuffers、sounddevice。
- **Web**：TypeScript + WASM，rollup 打包（根 `package.json`、`tsconfig.json`、pnpm）；WebGPU 支持见 `mediapipe/web/graph_runner/graph_runner_webgpu.ts`。
- **GPU**：`mediapipe/gpu/` 同时提供 OpenGL ES/EGL（Android/桌面）与 Metal（`MPPMetalHelper.h`，iOS/macOS）。
- **iOS**：`mediapipe/objc/`、`mediapipe/tasks/ios/`；根 `Package.swift` + `Sources/MediaPipeTasksCommon.swift` 仅是 SwiftPM 占位壳（1 个源文件）。

## 4. 目录结构与代码组织

顶层为"单一巨型仓库 + 多语言绑定"的单体结构，一切经 Bazel 统一构建：

| 目录 | 职责 | 大小 |
|---|---|---|
| `mediapipe/framework/` | 图引擎核心：Calculator/Packet/Scheduler/Executor，158 个文件 | 7MB |
| `mediapipe/calculators/` | 402 个 REGISTER_CALCULATOR 的实现（core/image/video/audio/tensor/tflite/util） | 22MB |
| `mediapipe/tasks/` | 新一代任务层，按 c/cc/ios/java/python/web 六端 × vision/text/audio/decision/retrieval/genai 组织 | 17MB |
| `mediapipe/examples/` | android/ios/desktop/coral 示例应用 | 20MB |
| `mediapipe/modules/` | 可复用子图（palm_detection、face_landmark、pose_landmark 等 11 个） | 3MB |
| `mediapipe/graphs/` | 旧版完整方案图（face_mesh、hand_tracking、holistic_tracking 等 17 个） | 3MB |
| `mediapipe/python/` | pybind11 绑定（framework_bindings.cc）+ 旧 Solutions API | 1MB |
| `mediapipe/web/` | Web 图运行时（TS/WASM/WebGPU） | 1MB |
| `mediapipe/gpu/`、`java/`、`objc/`、`model_maker/`、`models/` | GPU 基础设施、Android/iOS 原生层、迁移训练工具 | 各 1-3MB |

组织评述：所有平台绑定共用同一套 C++ 核心（tasks 下 c/cc/ios/java/python/web 六个镜像目录），是典型的"核心引擎 + 每平台薄壳"架构；旧/新两代 API 并存使仓库体量偏大，目录语义需要时间理解。

## 5. 核心架构与关键实现

### 5.1 Calculator 图引擎（mediapipe/framework/）

`CalculatorGraph`（`calculator_graph.h`，791 行头文件 + 1542 行实现）是框架主 API：从 `CalculatorGraphConfig` proto 初始化 DAG，`Run()/StartRun()` 时用 side packets 注入一次性参数，`ObserveOutputStream` 订阅输出。图内数据流由三个抽象支撑：

- **Packet**（`packet.h`）：引用计数的类型擦除容器 + `Timestamp`，浅拷贝零成本，跨节点传递数据与时间戳。
- **CalculatorNode / CalculatorBase**（`calculator_node.h`、`calculator_base.h`）：节点生命周期 Open/Process/Close，`GetContract` 声明端口类型。
- **Scheduler**（`scheduler.h`、`scheduler_queue.cc`）：多队列 + Executor 线程池并行调度就绪节点，WaitUntilDone/WaitUntilIdle 管理运行终点。

值得注意的工程细节：背压由调用方选择——`calculator_graph.h` 中定义 `GraphInputStreamAddMode::{WAIT_TILL_NOT_FULL, ADD_IF_NOT_FULL}`，前者控制内存（队列满则阻塞），后者用于实时场景丢帧。子图通过 `subgraph.h` 按类型递归展开（`CalculatorGraph::Initialize` 接受多配置 + 模板参数）。

### 5.2 新版 Node/Builder API（mediapipe/framework/api2/）

`api2/README.md` 明确说明这是 Calculator API 的演进：端口从字符串 tag 变为带类型的常量声明（`static constexpr Input<int>::Optional kSelect{"SELECT"}`），契约由 `MEDIAPIPE_NODE_CONTRACT` 声明式给出；Builder API 用 C++ 直接拼图。与旧 API 完全互操作，tasks 层新图已全部迁移到这套写法（见下）。

### 5.3 Tasks 层解剖：以 HandLandmarker 为例（mediapipe/tasks/cc/vision/hand_landmarker/）

`hand_landmarker_graph.cc`（397 行）展示了任务图的典型模式：类继承 `core::ModelTaskGraph`，用 api2 Builder 组装，注册为 `"mediapipe.tasks.vision.hand_landmarker.HandLandmarkerGraph"`。文件头注释描述了检测-跟踪混合策略：HandDetectorGraph 全帧检测一次，之后逐帧跟踪 landmarks；跟踪丢失或手数不足 `num_hands` 时才重新触发检测器——这是实时性能的关键设计。模型以 .task 包（ModelAssetBundleResources，`tasks/cc/core/model_asset_bundle_resources.cc`）加载，经 `mediapipe_builtin_op_resolver.cc` 进 TFLite 推理。

### 5.4 多语言绑定

- **Python**：旧框架 API 走 pybind11（`mediapipe/python/framework_bindings.cc`、`solution_base.py`）；新 Tasks API 自 2025 年起改为 ctypes 直调 C 共享库——`mediapipe/tasks/python/core/mediapipe_c_bindings.py` 中 `load_raw_library` 加载 `mediapipe/tasks/c/` 产出的 `libmediapipe`，一套 C ABI（75 个头文件，`mediapipe/tasks/c/`）同时服务 Python/iOS/Android 绑定，显著降低各端重复封装成本。
- **Web**：`mediapipe/web/graph_runner/graph_runner.ts` + WASM 运行时，`graph_runner_webgpu.ts` 提供 WebGPU 后端；`mediapipe/tasks/web/` 提供 22 个 TS 任务 API（vision 11、text 5、audio 3、retrieval 2、genai 1、decision 1，按目录统计）。

## 6. 构建与运行

来自根目录文档与脚本（均未实际运行）：

```bash
# Python 包（PyPI 直接安装预编译 wheel；源码构建需 Bazel 7.7.0）
pip install mediapipe                    # README 推荐路径
python setup.py gen_proto && python -m pip install -e .   # setup.py 流程

# 桌面示例（C++）
./setup_opencv.sh                        # OpenCV for desktop examples
bazel build -c opt --define MEDIAPIPE_DISABLE_GPU=1 \
  mediapipe/examples/desktop/hello_world
./build_desktop_examples.sh -d out_dir   # 批量构建桌面示例（脚本头注释）

# Android
./setup_android_sdk_and_ndk.sh           # 配置 SDK/NDK
bazel build -c opt --config=android_arm64 mediapipe/examples/android/src:...  # Android 示例
# iOS: Xcode 打开 mediapipe/MediaPipe.tulsiproj 或 bazel + rules_apple

# Docker（官方提供构建环境）
docker build -t mediapipe .              # Dockerfile（ubuntu:22.04）
```

Web 端用 pnpm + rollup（根 package.json），任务 API 通过 npm `@mediapipe/tasks-vision` 等包分发（分发方式见官方文档，仓库内未含发布脚本细节——未确认）。

## 7. 测试与工程质量

- 测试规模可观：456 个 `*_test.cc`、87 个 `*_test.py`、35 个 `.test.ts`；框架核心几乎每个文件都有对应测试（`calculator_graph_*_test.cc` 系列 10+ 个，覆盖 side packet、stopping、event loop、并行执行等专项）。
- 测试风格为 Bazel 单元测试 + 图级集成测试（calculator_runner_test.cc、tasks 端到端 testdata），Web 端 jasmine（package.json devDependencies）。
- CI 观察：`.github/workflows/` 仅有 `stale.yaml`（关闭陈旧 issue），无公开构建/测试流水线——主 CI 在 Google 内部（推断，README/CONTRIBUTING 未说明）。
- 代码质量观察：注释密度高、文件头含完整用法示例（如 `hand_landmarker_graph.cc` 的输入输出契约注释）；错误处理统一用 absl::Status/StatusOr；`api2` README 明示命名空间未定、可能有破坏性变更（框架演进中的自留余地）。

## 8. 社区与维护状态

- **活跃度**：gh api 显示最近 push 2026-11-16（分析当天）；open issues 250。
- **发版节奏**：2026 年进入 1.x 时代后明显提速——v1.0.0（2026-07-28）→ v1.1.0（10-07）→ v1.1.1（10-14）→ v1.2.0（10-28）→ v1.2.2（11-03）→ v1.2.3（11-11），约每 1-2 周一个 release（gh api releases）。0.10.x 时代节奏为月度。
- **贡献者**：约 128 位（gh api contributors?anon=true，per_page=1 翻页得 128）；以 Google 员工为主导（提交邮箱/文档署名 mediapipe@google.com，未逐条核实）。
- **版本策略**：`mediapipe/version.bzl` 中 "The patched version code should be bumped after every release" 表明发版流程内部管理。
- **生态位**：组织名 google-ai-edge 还收留 LiteRT 等边缘 AI 项目（WORKSPACE 注释提及 @litert）；姊妹资源 awesome-mediapipe（README 链接）。

## 9. 亮点与风险

**亮点**
1. 图流水线引擎经过大规模产品验证（Google 实时视频产品），背压/时间戳/子图设计在同类框架中独一份（`calculator_graph.h`、`scheduler.h`）。
2. 一套 C ABI + 75 个头文件同时喂饱 Python/iOS/Android/ 四端绑定（`mediapipe/tasks/c/`、`tasks/python/core/mediapipe_c_bindings.py`），跨平台工程化水平高。
3. 检测-跟踪混合调度（`hand_landmarker_graph.cc`）等实用优化直接写在任务图里，是学习实时 CV 管线设计的一手教材。
4. 维护极活跃：当天有 push、1.x 发版周期 1-2 周，0.10→1.x 大版本平滑过渡。

**风险/局限**
1. Bazel 构建链重（MODULE.bazel + WORKSPACE 双轨、pin org_tensorflow 全量），从源码定制构建门槛高，普通用户只能消费 PyPI/npm 预编译包。
2. 与 TFLite/LiteRT 版本强耦合（WORKSPACE 需手动对齐 XNNPACK 编译标志），升级易碎。
3. 新旧两代 API（Solutions vs Tasks、framework vs api2）长期并存，文档与示例易混淆；api2 明示"可能有 breaking changes"。
4. 公开 CI 缺位（仅 stale.yaml），外部贡献者难以验证多平台构建，实质上仍是"Google 内部 trunk + GitHub 镜像"模式。

## 10. 总结评价

MediaPipe 是"设备端 ML 产品化"的标杆工程：把一次性的模型推理重构为可组合、可调度、可背压的流式图管线，再以统一 C ABI 复制到六种语言平台。它既是拿来即用的检测 SDK（face/hand/pose 全家桶），也是少有的、值得精读的实时系统 C++ 源码库。适合三类人：给 App/Web 加人脸/手势/姿态能力的应用开发者（直接用 tasks API）；需要自建实时多媒体推理管线的工程师（精读 `mediapipe/framework/` 的 CalculatorGraph/Scheduler/Packet 三件套）；以及想理解"检测+跟踪"如何落地的 CV 工程师（精读 `mediapipe/tasks/cc/vision/` 任一任务图）。想贡献或深度定制的团队则需接受 Bazel + TFLite 强绑定的构建现实。
