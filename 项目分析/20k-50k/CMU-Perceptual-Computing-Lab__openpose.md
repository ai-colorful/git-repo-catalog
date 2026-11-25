# CMU-Perceptual-Computing-Lab/openpose 项目分析

> 星数 34,606（gh api 2026-11-21 核实；派发快照 26,000）｜主语言 C++｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：CMU 感知计算实验室出品的**首个实时多人全身 2D 人体关键点检测库**（body/foot/hand/face 共 135 个关键点），人体姿态估计领域 CVPR 2017 Part Affinity Fields 论文的官方工程实现，也是 2017-2020 年间该领域事实上的基准工具。

| 项目 | 值 |
|---|---|
| 名称 | CMU-Perceptual-Computing-Lab/openpose |
| 星数 | 34,606（gh api 2026-11-21 核实） |
| Fork | 8,038（gh api 核实） |
| 语言 | C++（含 CUDA / OpenCL 代码） |
| 许可证 | 自定义"非商业研究许可"（GitHub 标记 NOASSERTION，见 `LICENSE`） |
| 创建时间 | 2017-04-24 |
| 最后 push | 2024-08-03（最后 commit "Models working again"，5c5d9652） |
| 最后 release | v1.7.0（2020-11-17） |
| 维护状态 | **未 archived，但实质停更**：2020 年后无 release，2023-2024 每年仅个位数 commit，最后一次提交仅为修复模型下载 URL |

## 2. 项目是什么，解决什么问题

OpenPose 解决的核心问题是**多人场景下的实时全身姿态估计**。传统 top-down 方法先检测人再逐个估计关键点，耗时随人数线性增长；OpenPose 采用 bottom-up 方案（CVPR 2017 论文，README "Related Work"/Citation 节，arXiv 1611.08050 与 TPAMI 2019 扩展版 arXiv 1812.08008）：网络一次性输出所有人体部位的热图与部位间亲和场（Part Affinity Fields, PAF），再通过二分图匹配组装成多人骨架，使**推理耗时与画面中人数无关**（README "Runtime Analysis" 节及对比图）。

功能范围（README "Features" 节）：
- 2D 多人关键点：15/18/25 点 body+foot（BODY_25 含 6 个脚部关键点）、2x21 点 hand、70 点 face；
- 3D 模块：多视角单点三角化重建 + Flir/Point Grey 相机同步（`doc/advanced/3d_reconstruction_module.md`）；
- 标定工具箱：相机畸变/内外参估计（`doc/advanced/calibration_module.md`）；
- 单人跟踪：加速与视觉平滑（`src/openpose/tracking/`）。

输入输出均做了插件化：输入支持图像目录/视频/摄像头/IP 相机/FLIR 并允许自定义，输出支持 PNG/JPG/AVI 渲染图、JSON/XML/YML 关键点及数组类（README "Main Functionality"/"Input"/"Output" 节）。

使用形态有三种：命令行 demo（`examples/openpose/openpose.bin`）、C++ API、Python API（`doc/01_demo.md`、`doc/03_python_api.md`、`doc/04_cpp_api.md`）。另有 Windows 免编译便携版与独立仓库的 Unity 插件（README "Installation"/"Unity Plugin" 节）。

**学术脉络**：代码对应四篇论文——CVPR 2016 Convolutional Pose Machines（骨架基础）、CVPR 2017 PAF（主算法）、CVPR 2017 Multiview Bootstrapping（hand/face 训练方法）、TPAMI 2019 扩展版（README Citation 节完整引用）。注意**训练代码不在本仓库**，在独立的 `CMU-Perceptual-Computing-Lab/openpose_train` 仓库（README "Related Work" 节），本仓库是纯推理/部署工程。后续社区生态以 fork 与衍生实现（如 TensorFlow/PyTorch 移植）为主，主仓库自 2024-08 后无活动，属停更项目。

## 3. 技术栈与依赖

- **语言**：C++ 为主（`src/openpose` + `include/openpose` 约 4.9 万行），另有 CUDA（`.cu`）与 OpenCL（`*CL.cpp`）双 GPU 路径、pybind11 Python 绑定。
- **深度学习推理**：Caffe 1.0.0（版本锁定见 `3rdparty/Versions.txt`，Ubuntu 用 BVLC master 分支特定 SHA `8645207`，Windows 用 caffe/windows 分支）。
- **图像与 GUI**：OpenCV（Windows 预编译 4.5.0，`3rdparty/Versions.txt`；Ubuntu 兼容 2.X/3.X，`doc/installation/0_index.md` Dependencies 节）。
- **其他第三方**（`3rdparty/Versions.txt`）：Eigen 3.3.8、Asio 1.12.1、FreeGLUT 3.0（3D 渲染）、Ceres（可选高级 3D）、Spinnaker SDK（FLIR 相机）、gflags/glog（随 Caffe 引入）。
- **绑定**：pybind11（git submodule `3rdparty/pybind11/`，`CMakeLists.txt:1000-1003`）；Unity 通过 `src/openpose/unity/unityBinding.cpp` 配合独立插件仓库。
- **构建**：CMake（最低 3.0，`CMakeLists.txt:34`）。
- 硬件路径三选一：CUDA（NVIDIA，建议 cuDNN）、OpenCL（AMD Vega）、纯 CPU（约 8GB 内存）（`doc/installation/0_index.md` Requirements 节）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `include/openpose/` | 17 个模块的公开头文件（API 层），193 个文件 |
| `src/openpose/` | 各模块实现（实现层），145 个文件，与 include 一一对应 |
| `include/openpose/{core,net,pose,hand,face,...}/` | 模块划分见下表 |
| `python/openpose/` | pybind11 绑定（`openpose_python.cpp`、`__init__.py`） |
| `examples/` | demo 入口（`openpose/`）+ 18 个 C++ 教程（`tutorial_api_cpp/`）+ 9 个 Python 教程 + 手动测试（`tests/`） |
| `models/` | 仅模型结构 prototxt 与下载脚本（`getModels.sh`/`getModels.bat`），**权重不入仓** |
| `3rdparty/` | Caffe/pybind11 submodule、Windows 预编译库、版本清单 `Versions.txt` |
| `cmake/` | CMake 辅助模块 |
| `scripts/` | Ubuntu 安装脚本、CI 脚本、精度基准脚本（`scripts/tests/`） |
| `doc/` | doxygen 生成的文档源（demo/output/python api/cpp api/faq/release notes 等 10 个章节） |
| `.github/` | GitHub Actions 工作流、issue 模板、stale bot 配置、`CI_deprecated/`（旧 Travis/AppVeyor 遗留） |

模块划分（`include/openpose/` 与 `src/openpose/` 对称）：`core`（Datum/Array/缩放等基础设施）、`net`（NMS/PAF 连接等网络算子）、`pose`（body 主算法）、`hand`/`face`、`producer`（输入源）、`filestream`（JSON/XML/图片读写）、`gui`、`gpu`、`thread`（多线程队列框架）、`wrapper`（一体化 API）、`3d`、`calibration`、`tracking`、`unity`、`utilities`、`filestream`。

组织方式评述：单体仓库、单库多模块。每个模块在 `src/openpose/<module>/` 有独立 CMakeLists 聚合源文件，最终由 `src/openpose/CMakeLists.txt` 统一构建成单一 `openpose` 动态库（CUDA 模式走 `cuda_add_library`，否则 `add_library`）；头文件与源文件按模块镜像分层，公开 API（hpp）与私有实现（cpp）边界清晰，模板类（如 WrapperT、Array）实现内联在头文件中。

## 5. 核心架构与关键实现

### 5.1 WrapperT：一体化多线程管线（`include/openpose/wrapper/wrapper.hpp`）

716 行的 header-only 模板类，是官方推荐的用户入口。它把 producer→预处理→推理→后处理→渲染→输出 串成一条由 `ThreadManager` 驱动的 worker 流水线，支持四种线程拓扑（`include/openpose/thread/enumClasses.hpp`）：`Asynchronous`（输入输出队列都交给用户，原型模式）、`AsynchronousIn/Out`（半托管）、`Synchronous`（全托管，生产模式）。

异步模式下用户通过 `emplace()`/`waitAndEmplace()`/`waitAndPop()`/`emplaceAndPop()` 手动喂帧取帧（`wrapper.hpp:130-227`）；`tryEmplace` 在队列满或已停止时返回 false 而不阻塞（`wrapper.hpp:150-152`）。注：任务提示中的 `KeepOpen` 在当前快照全仓 grep 无匹配，对应机制实际为 `isRunning()` + `tryEmplace` 组合（如实记录，未确认旧版本存在该字段）。

### 5.2 线程队列框架（`include/openpose/thread/queueBase.hpp`）

`QueueBase` 是所有队列的基类：mutex + condition_variable 实现阻塞队列，维护 pusher/popper 计数与 `stop()`/`stopPusher()` 语义，保证下游线程能感知上游关闭（`queueBase.hpp:12-88`）。其上有 `Queue`（std::queue 包装）与 `PriorityQueue`（按 Datum 的 id/subId 排序，多线程乱序完成后重排帧，`priorityQueue.hpp:11`、`include/openpose/core/datum.hpp` 中 `id/subId/subIdMax` 字段）。Worker 抽象（`worker.hpp`）+ `Thread`/`ThreadManager`（`threadManager.hpp:76-77`，线程与队列均为容器托管）构成可自由拼装的 DAG 式流水线；`wQueueAssembler.hpp`、`wFpsMax.hpp`、`wIdGenerator.hpp` 等是内置 worker 组件。

### 5.3 多后端推理与渲染：CPU/CUDA/OpenCL 三路径

`src/openpose/net/` 是理解"一份算法、三份实现"的关键，每个核心算子都有三个文件：
- NMS：`nmsBase.cpp`（CPU）/ `nmsBase.cu`（CUDA）/ `nmsBaseCL.cpp`（OpenCL）；
- PAF 部件连接：`bodyPartConnectorBase.cpp/.cu/bodyPartConnectorBaseCL.cpp`；
- 热图上采样合并：`resizeAndMergeBase.cpp/.cu/BaseCL.cpp`；极值搜索：`maximumBase.cpp/.cu`。

推理引擎抽象为 `Net` 接口，主实现是 `netCaffe.cpp`（Caffe 前向）；`netOpenCv.cpp` 是 OpenCV DNN 备选实现，但文件头注释 `// TODO: After completely adding the OpenCV DNN module, add this flag to CMake as alternative to USE_CAFFE` 与 `#define USE_OPEN_CV_DNN` 处于注释状态（`netOpenCv.cpp:1-24`），属未启用的实验路径，`poseExtractorCaffe.cpp:88` 处的调用同样被注释。

渲染同样双路径：`poseGpuRenderer.cpp`（CUDA kernel `renderPose.cu`，需手动 `cudaFree` 管理 GPU 缓冲）与 `poseCpuRenderer.cpp`（调 `renderPoseKeypointsCpu` 用 OpenCV 画骨架），选择由 CMake 的 `GPU_MODE` 决定。OpenCL 后端由 `src/openpose/gpu/opencl.cpp` 提供；CMake 中 `USE_CUDNN`、`GPU_MODE ∈ {CUDA, OPENCL, CPU_ONLY}`（`CMakeLists.txt:197-275`）。

主算法装配在 `poseExtractorCaffe.cpp`：Caffe 前向输出 → `resizeAndMergeCaffe` → `nmsCaffe` → `bodyPartConnectorCaffe` → `maximumCaffe`（hand/face 用）四级 blob 流水，全部以 `std::shared_ptr<ArrayCpuGpu<float>>` 在 GPU 间传递（`poseExtractorCaffe.cpp:22-50`）。

### 5.4 Datum：贯穿全管线的数据单元（`include/openpose/core/datum.hpp`）

`Datum` 结构携带 id/subId（多线程排序）、cvInputData/cvOutputData（输入输出图像）、inputNetData/outputData（网络输入与热图）、poseKeypoints/faceKeypoints/handKeypoints 等字段，是 wrapper 队列中流动的统一载体；Python 绑定通过 `PYBIND11_MAKE_OPAQUE(std::vector<std::shared_ptr<op::Datum>>)` 直接暴露该类型（`python/openpose/openpose_python.cpp:14`）。

## 6. 构建与运行

来自 `doc/installation/0_index.md` 与 CI 配置的实际流程（未实际运行验证，按文档与 CI 脚本记录）：

```bash
# 1. 下载模型权重（不入仓，外链 wget）
cd models && ./getModels.sh          # Windows 用 getModels.bat（models/getModels.sh）

# 2. CMake 配置（Ubuntu, CUDA）
cmake -DBUILD_PYTHON=ON ..           # 顶层 CMakeLists.txt:316 BUILD_PYTHON 默认 OFF
# GPU_MODE 可选 CUDA / OPENCL / CPU_ONLY（CMakeLists.txt:197,209）
# 可选：WITH_3D_RENDERER / WITH_CERES / WITH_FLIR_CAMERA（CMakeLists.txt:278-282）

# 3. 编译
make -j`nproc`

# 4. 运行 demo（webcam 实时显示 body 关键点）
./build/examples/openpose/openpose.bin
./build/examples/openpose/openpose.bin --image_dir examples/media/ --write_json output/
```

平台差异（`doc/installation/0_index.md` + `appveyor.yml`）：
- **官方支持**：Windows 10、Ubuntu 20、macOS Mavericks+、Jetson TX1/TX2；Ubuntu 14/16/18 与 Win 7/8"不再官方维护但应仍可用"。
- **Windows**：提供免编译便携版；源码编译依赖 `3rdparty/windows/` 下的预编译 OpenCV 4.5.0/caffe3rdparty/FreeGLUT；AppVeyor 仅做 CPU_ONLY 构建 + `OpenPoseDemo.exe --image_dir ... --net_resolution -1x32` 冒烟测试（`appveyor.yml`）。
- **Ubuntu**：GitHub Actions 45 个矩阵 job（U18/U20 × CUDA/CPU/OpenCL/debug × python），含文档生成 job（`.github/workflows/main.yml`）。
- Python 绑定用法见 `examples/tutorial_api_python/01_body_from_image.py`：Windows 手动补 PATH 后 `import pyopenpose`，Ubuntu 从安装目录导入；参数以 dict 透传给 gflags（`python/openpose/openpose_python.cpp:20-36` 的 `parse_gflags`）。
- 仓内无 Java 绑定（全仓无 Java 相关 CMake 项；Unity 场景走独立插件仓库，未确认官方 Java SDK 存在）。

## 7. 测试与工程质量

- **无单元测试框架**：没有 gtest 等测试依赖。`examples/tests/` 仅 3 个手动测试程序（`clTest.cpp`、`handFromJsonTest.cpp`、`resizeTest.cpp`、`wrapperHandFromJsonTest.hpp`）；`scripts/tests/` 是精度与速度基准 shell 脚本（COCO val/test-dev 评测、逐层耗时分析，`scripts/tests/pose_accuracy_coco_val.sh` 等）。
- **CI 即测试**：质量保障主要靠 GitHub Actions 45 个构建矩阵 job（编译通过 + demo 运行）与 AppVeyor Windows 冒烟测试（`appveyor.yml` after_build 段）。旧 Travis 配置残留在 `.github/CI_deprecated/.travis.yml`。
- **代码质量观察**：全仓统一的 `try/catch + error()` 异常处理风格（几乎所有实现文件可见）；配置校验集中在 `src/openpose/wrapper/wrapperAuxiliary.cpp`（对无效输出组合、CPU 模式限制、OpenCL 静态分辨率等给出明确中文程度不一的错误提示，见该文件 70-200 行）；长函数较多（如 `wrapper.hpp` 内联实现、`CMakeLists.txt` 达 1069 行）；头文件注释非常详尽，是 API 文档的主要载体（配合 doxygen，`.doc_autogeneration.doxygen`）。
- **issue 治理**：`.github/stale.yml` 配置 60 天未活动自动标记 stale/old、7 天后关闭（bug/typo 与 enhancement 豁免）。

## 8. 社区与维护状态

全部经 gh api 于 2026-11-21 核实：

- **未 archived、未 disabled**，但最后 push 为 **2024-08-03**（commit "Models working again"，修复模型托管 URL），距今已两年余无任何提交——**实质停更项目**。
- **发版节奏**：v1.5.1（2019-09）→ v1.6.0（2020-04）→ v1.7.0（2020-11-17，最后一个 release）。此后仅零星 commit：2023-12、2024-01、2024-03、2024-04 各 1 个，2024-08 最后 1 个（gh api commits 列表）。
- **贡献者**：101 人（非匿名 contributors 分页计数）；核心维护者为 Ginés Hidalgo 与 Yaadhav Raaj（README 作者/维护者声明）。贡献高度集中于创始团队。
- **issue 积压**：360 个 open issue；stale bot 只能自动归档无维护者跟进的问题，无法形成闭环。
- **生态**：官方姊妹仓库 `openpose_train`（训练）、`openpose_unity_plugin`（Unity）；模型权重托管在 SNU VCL 镜像（`models/getModels.sh` 中 `http://vcl.snu.ac.kr/OpenPose/models/`）——最后一次提交正是在修复这条链接，侧面反映外部托管的不稳定。
- README 顶部 Linux/macOS 徽章指向已停摆的 GitHub Actions 状态，Windows 指向 AppVeyor；`doc/installation/0_index.md` 中"欢迎移植到其他框架并给我们发邮件"的表述也停留在项目活跃期的口吻。

## 9. 亮点与风险

**亮点**
1. 工程化的多线程流水线框架（`include/openpose/thread/` + `wrapper.hpp`）：队列/优先队列/Worker 抽象 + 四种托管模式，2017 年即达到生产级并发设计，其 Datum 管线结构至今仍是许多姿态估计工程（含后续 MediaPipe 类工具链）的参考模板。
2. 一份算法三份后端（CUDA/OpenCL/CPU）+ 双渲染路径，且依赖声明集中可查（`3rdparty/Versions.txt` 锁 Caffe SHA、OpenCV 版本），跨平台移植成本被明确控制。
3. API 分层干净：include 公开头文件 / src 实现镜像对应，`examples/tutorial_api_cpp/` 18 个由浅入深的教程覆盖同步/异步/多 GPU/自定义前后处理全场景。
4. 模型权重与代码分离（`models/getModels.sh` 外链下载），仓库本体仅 51MB tarball，prototxt 结构仍在仓内可审查。
5. 学术-工程对应完整：CVPR 2017 / TPAMI 2019 论文 ↔ 仓库实现 ↔ 独立训练仓库，链路可追溯（README Citation 节）。

**风险/局限**
1. **停更**：最后 commit 2024-08-03，最后 release 2020-11；依赖 Caffe 1.0.0 与旧版 CUDA 工具链，在新版 Ubuntu/CUDA/PyTorch 主流环境下编译摩擦大，且不会再有修复。
2. **许可限制**：非商业研究许可（`LICENSE`，CMU 自定义条款），衍生代码所有权归 CMU、禁商用，GitHub 标记 NOASSERTION，商业落地需另行授权。
3. **模型外链单点**：权重托管在 SNU 服务器（`models/getModels.sh`），历史上多次失效（最后一次 commit 即为此修复），无官方镜像清单或校验和。
4. 无单元测试、issue 积压 360 个无维护跟进；Python API 文档滞后于代码（`python/openpose/` 仅 3 个文件，错误提示依赖 try/except）。

## 10. 总结评价

OpenPose 是人体姿态估计从学术走向工程基础设施的标志性项目：bottom-up PAF 算法 + 与人数无关的实时推理 + 完整的 body/hand/face/foot 全身覆盖，使它在 2017-2020 年成为该领域默认基准；其多线程 worker 管线、三后端算子组织和教程体系即使在今天读来仍具教学价值。作为**停更项目**，它的价值已从"可用工具"转为"可读源码的参考实现"：新项目应选择 PyTorch 生态的后续实现（社区 fork 或 MediaPipe 等），需要复现论文基线或研究经典工程化组织方式时，本仓库依然值得精读。适合 CV 研究者（复现/对比基线）、C++ 多媒体管线工程师（学习队列式流水线与跨平台 GPU 抽象）阅读源码；不适合期望开箱即用、长期维护保证或商业授权的生产团队。
