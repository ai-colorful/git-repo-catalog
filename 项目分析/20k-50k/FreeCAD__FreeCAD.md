# FreeCAD/FreeCAD 项目分析

> 星数 22500（快照 2026-10-24；分析日 2026-11-21 经 gh api 核实已达 35,654）｜主语言 C++｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：开源参数化 3D CAD 建模器——机械、建筑（BIM）、钣金、FEM、CAM 等全功能覆盖的自由软件替代品，几何内核基于 OpenCASCADE。

| 项目 | 值 | 出处 |
| --- | --- | --- |
| 名称 | FreeCAD/FreeCAD | gh api repos/FreeCAD/FreeCAD |
| 星数 | 35,654（2026-11-21 核实；快照 22,500） | gh api |
| Forks / Open Issues | 6,434 / 4,084 | gh api |
| 主语言 | C++（2526 .cpp + 1971 .h），Python 1938 个 .py | tarball 统计 |
| 许可证 | LGPL-2.1-or-later | LICENSE、CONTRIBUTING.md 第 43 行 |
| 默认分支 | main（GitHub 仓库创建于 2012-09-09） | gh api |
| 最近 push | 2026-11-20（分析前一天，高度活跃） | gh api pushed_at |
| 当前版本 | HEAD 为 27.1.0.dev；最新稳定版 1.1.4（2026-09-28）；1.0.0 发布于 2024-11-19 | version.json、gh api releases |
| 贡献者 | 约 1112 人（anon=true 分页计数） | gh api contributors Link 头 |

## 2. 项目是什么，解决什么问题

FreeCAD 是一个开源的参数化三维 CAD 建模器（README.md "Your own 3D Parametric Modeler"），面向现实物体的设计——从小型零件到建筑整体。其核心卖点是参数化建模：通过模型历史中的参数修改设计，用户改变草图尺寸或特征参数后系统自动重算下游几何（src/App/Document.h 中的 recompute 信号体系即为此服务）。

与商业 CAD（SolidWorks、Onshape）相比，FreeCAD 解决的问题域包括：机械零件设计与装配、2D 草图→3D 特征的正反向转换（README.md "Create 3D from 2D and back"）、生产级工程图（TechDraw 模块）、建筑 BIM/IFC（BIM 模块 + ifcopenshell）、有限元仿真（Fem 模块）与 CAM 刀具路径。它按 LGPL 发布，可免费商用，且以 Python API 与宏录制提供完整自动化能力。

工程形态上，FreeCAD 是一个"平台 + 工作台（Workbench）"体系：核心提供文档/对象/属性/重算框架，33+ 个 Mod 模块各自以工作台形式提供领域功能（见第 4 章统计），用户还可通过 Python（FeaturePython 机制，src/App/FeaturePython.h）编写自定义对象，无需写 C++。

从版权头可追溯到 2001 年（src/Mod/Part/Init.py "Copyright (c) 2001, 2002 Juergen Riegel"），项目历史悠久；版权头中大量 `users.sourceforge.net` 邮箱（如 src/Base/BindingManager.cpp:4）是 SourceForge SVN 时代的第一手证据。1.0.0 于 2024-11-19 发布（gh api releases/tags/1.0.0），标志项目进入成熟期。

## 3. 技术栈与依赖

- 语言：C++（核心框架与几何相关模块）+ Python（工作台逻辑、扩展、测试）；GUI 层 Qt6（顶层 CMakeLists.txt:91-92 `find_package(Qt6 REQUIRED COMPONENTS Core)`，`FREECAD_QT_MAJOR_VERSION 6`）。
- 几何内核：OpenCASCADE，最低版本 7.6（cMake/FreeCAD_Helpers/SetupOpenCasCade.cmake：`find_package(OCC 7.6 REQUIRED)`）。README.md 称其为 "the most important component of FreeCAD"。
- 3D 显示：Coin3D（Open Inventor 场景图），经 FreeCAD 维护的 coin/pivy fork 子模块接入（.gitmodules；SetupCoinPivy）。
- 网格剖分：Salome SMESH（SetupSalomeSMESH）+ 可选 NETGEN（BUILD_FEM_NETGEN）。
- Python 集成：CPython 嵌入（src/Base/Interpreter.h）+ PyCXX 封装层 + 自研绑定代码生成器（见第 5 章）；PySide/Shiboken 用于 UI 脚本（SetupShibokenAndPyside）；pybind11 亦有 CMake 接入（顶层 CMakeLists.txt:99 SetupPybind11），抽样中发现 CAM 模块 tsp_solver_pybind.cpp 使用。
- 其他关键依赖（顶层 CMakeLists.txt:97-146 + pixi.toml）：Boost、Eigen、XercesC（XML）、zlib、yaml-cpp、clipper2、ICU、PCL、HDF5/VTK（经 SMESH）、Freetype、matplotlib；BIM 需 Python 运行时依赖 ifcopenshell（CMakeLists.txt:136）；Assembly 仿真视频导出用 av/pillow（CMakeLists.txt:139-141）。
- 5 个 git 子模块（.gitmodules）：src/3rdParty/OndselSolver（装配求解器）、src/3rdParty/GSL（微软 Guidelines Support Library）、src/Mod/AddonManager（插件管理器）、coin 与 pivy（FreeCAD 维护分支）。

## 4. 目录结构与代码组织

tarball 共 14,155 个条目（约 425MB 解压），src/ 占 13,389。顶层结构：

| 目录 | 职责 |
| --- | --- |
| src/ | 全部源码（src/CMakeLists.txt 依序组织 Build→3rdParty→Base→App→Main→Mod→Ext→Doc→Gui） |
| src/Base | 工具层：Python 解释器、参数系统、单位、控制台、持久化、Placement/Matrix 等（87 个 .cpp） |
| src/App | 核心对象模型：Document/DocumentObject/Property/表达式引擎（100 个 .cpp） |
| src/Gui | Qt6 界面层：ViewProvider/Workbench/Command/宏录制（377 个 .cpp，最大框架层） |
| src/Main | 入口：MainGui.cpp（GUI）、MainCmd.cpp（无头）、FreeCADGuiPy.cpp（模块嵌入） |
| src/Mod | 33+ 个功能模块（见下） |
| src/3rdParty | 内置第三方与子模块（OndselSolver/GSL/coin/pivy 等，1114 文件） |
| src/Ext | 独立 Python 包 `freecad/`（可脱离 GUI 以 `from freecad import ...` 使用） |
| src/Tools | 构建期工具，含 bindings/generate.py 绑定生成器 |
| tests/ | C++ 单元测试（GTest/CTest）：App/Base/Gui + 13 个 Mod，192 个测试 .cpp；另有 tests/visual 视觉回归 |
| package/ | 发行打包：WindowsInstaller、bundle（macOS）、fedora、ubuntu、libpack-windows |
| cMake/ | 自研 CMake 宏库（FreeCAD_Helpers 39 个 Setup*.cmake） |
| data/ | 示例文件与测试资产 |
| contrib/ | 开发者贡献指南（guidelines/architecture.md 等） |

src/Mod 模块按文件数排序（tarball 统计）：TechDraw 1231、Fem 1208、CAM 1121、Part 836、Sketcher 606、BIM 605、Material 536、Draft 532、PartDesign 491、Robot 314、Mesh 313、Assembly 170、Tux 164、OpenSCAD 156、Spreadsheet 154、Measure 131、Surface 123、Test 114、Inspection 111、Start/Points/MeshPart 各 103、ReverseEngineering/Import 各 102、Help 92、Show/Web/Plot 等。语言上呈清晰分层：Part/Sketcher/PartDesign/TechDraw/CAM/Fem 为 C++ 核心，Draft（246 个 .py、无 .cpp）、BIM（229 个 .py）、OpenSCAD、Spreadsheet 为纯 Python 工作台。

组织方式评述：单体仓库（monorepo），但内部是严格的"核心框架 + 可插拔模块"架构——每个 Mod 内部统一为 `App/`（领域对象）+ `Gui/`（ViewProvider）+ Python 层（Init.py/InitGui.py）三段式（见 src/Mod/Part/）。C++ 文件 2526 + 头文件约 2363（h/hxx/hpp 合计），图标 2311 个 SVG、翻译文件 1232 个 .ts + 369 个 .qm、Qt Designer 表单 495 个 .ui、材料库 218 个 .FCMat，是典型的桌面级大型 GUI 工程。

## 5. 核心架构与关键实现

### 5.1 App 核心对象模型：DocumentObject + Property 体系

一切领域对象继承自 `App::DocumentObject`（src/App/DocumentObject.h:138，继承 TransactionalObject 以获得 undo/redo 事务能力）。每个对象由属性（Property）构成状态，属性变更触发信号与依赖图重算：

```cpp
class AppExport DocumentObject: public App::TransactionalObject
{
    PROPERTY_HEADER_WITH_OVERRIDE(App::DocumentObject);
public:
    PropertyString Label;
    PropertyString Label2;
    PropertyExpressionEngine ExpressionEngine;  // 表达式引擎属性
    PropertyBool Visibility;
    fastsignals::signal<void(const App::DocumentObject&, const App::Property&)> signalBeforeChange;
    fastsignals::signal<void(const App::DocumentObject&, const App::Property&)> signalChanged;
```

（src/App/DocumentObject.h:138-189）属性容器基类 `App::PropertyContainer : public Base::Persistence`（src/App/PropertyContainer.h:296），App 层有 22 个 Property*.h 文件（PropertyStandard.h 单文件 1373 行），覆盖字符串/数值/链接/几何等类型。文档类 `App::Document : public PropertyContainer`（src/App/Document.h:100）持有对象图并定义重算事务信号（signalBeforeRecompute/signalRecomputed，Document.h:256-262）。几何对象由 `GeoFeature : public DocumentObject`（src/App/GeoFeature.h:49）承载 Placement。

### 5.2 App/Gui 解耦：ViewProvider 体系

App 层不依赖 Qt；显示由 Gui 层的 `Gui::ViewProvider : public App::TransactionalObject`（src/Gui/ViewProvider.h:212）对应，`ViewProviderDocumentObject`（src/Gui/ViewProviderDocumentObject.h:49）将其挂接到文档对象。绑定关系通过 App 侧的"边界方法"声明：

```cpp
/**
 * @brief Get the name of the type of the ViewProvider.
 * By overriding this function, document objects can indicate what the
 * type of the belonging view provider is.  Note that this function is
 * essentially the boundary between the App and Gui namespaces.
 */
virtual const char* getViewProviderName() const { return ""; }
```

（src/App/DocumentObject.h:215-226）这使得同一核心可无头运行（src/Main/MainCmd.cpp）或带 GUI 运行（src/Main/MainGui.cpp），也让 Python 工作台可用 `getViewProviderNameOverride`（DocumentObject.h:232-241）动态替换视图类。界面按 Workbench 组织（src/Gui/Workbench.h:53），启动时由 src/Gui/FreeCADGuiInit.py 注册各模块工作台。

### 5.3 C++/Python 双层结构：自研绑定生成器 + 宏录制

Python 绑定不走 SWIG，而是自研两件套：构建期代码生成 + PyCXX 运行时封装。生成器 src/Tools/bindings/generate.py 读取 `.pyi` 模型文件（历史为 XML，`generate_from_xml` 宏注释 "To be removed once all instances are migrated"，src/cMake/FreeCadMacros.cmake:123；现行 `generate_from_py` 宏在 FreeCadMacros.cmake:153），产出 `<Class>Py.h/.cpp`：

```cpp
// inclusion of the generated files (generated out of DocumentObjectPy.xml)
#include <App/DocumentObjectPy.h>
#include <App/DocumentObjectPy.cpp>
```

（src/App/DocumentObjectPyImp.cpp:30-31）源码树中 194 个 .pyi（src/App/Document.pyi 等）既是绑定模型又是 IDE 类型存根，一份文件两用。Python 工作台对象则经 `FeaturePythonImp`（src/App/FeaturePython.h:35）桥接：C++ 核心回调 Python 实现的 execute/onChanged 等。启动期模块装载本身也写在 Python 里：src/App/FreeCADInit.py（约 1500+ 行）在构建期以 `generate_embed_from_py(FreeCADInit InitScript.h)`（src/App/CMakeLists.txt:125）嵌入二进制，运行时扫描标准/用户/附加 Mod 目录（FreeCADInit.py:1486-1496）、设置 `App.__ModDirs__` 并逐个加载 Init.py。宏录制由 `Gui::MacroManager : public Base::Observer<const char*>`（src/Gui/Macro.h:106）实现，命令执行时按 App/Gui/Cmt 三类行写入宏文件。

### 5.4 几何与求解器：OCC 封装 + 双求解器并存

Part 模块把 OCC 拓扑封装为 `Part::TopoShape : public Data::ComplexGeoData`（src/Mod/Part/App/TopoShape.h:285；基类在 src/App/ComplexGeoData.h——将几何数据序列化/持久化下沉到核心层），`Part::Feature : public App::GeoFeature`（PartFeature.h:67）是所有形状对象的基类。求解器呈"双轨"：Sketcher 使用内置 planegcs（src/Mod/Sketcher/App/planegcs/GCS.h，Sketch.cpp:30 引入，默认 DogLeg 算法 Sketch.cpp:139-141）；Assembly（2024 年新增）使用 OndselSolver 子模块（src/Mod/Assembly/App/AssemblyObject.cpp:50 `#include <OndselSolver/CREATE.h>`，版权头 2023 Ondsel）——这是商业公司 Ondsel 参与后回馈的架构演进点。

## 6. 构建与运行

- 要求 CMake ≥3.22（顶层 CMakeLists.txt:2）、Qt6、Python3、OCC ≥7.6；Windows 提供 LibPack 预编译依赖包（FreeCADLibpackChecks、package/libpack-windows）。
- 官方推荐开发环境用 pixi（conda-forge）：pixi.toml 定义 linux/osx/win 多平台依赖（cmake、eigen、gmsh ≥4.13.1、gtest、hdf5、ifcopenshell、boost 等），`pixi shell` 后标准 CMake 流程。
- 构建命令（顶层 CMakeLists.txt:172-174 自带提示）：

```bash
cmake --preset <preset>          # 见 CMakePresets.json
cmake --build <build-dir>        # 生成 FreeCAD（GUI）与 FreeCADCmd（无头）
```

- 测试（CMakeLists.txt:163-168，需 `ENABLE_DEVELOPER_TESTS`）：启用 CTest + GTest；C++ 单测在 tests/src/（192 个文件，覆盖 App/Base/Gui/13 个 Mod）；Python 侧各 Mod 自带 Test*.py（如 src/Mod/Part/TestPartApp.py）；GUI 测试用 .github/scripts/run_gui_tests.py；视觉回归在 tests/visual/。

## 7. 测试与工程质量

- 测试规模：GTest 单元测试 192 个 .cpp（tests/src），13 个模块有专属 C++ 测试目录（Assembly/CAM/Part/Sketcher/TechDraw/Mesh/PartDesign 等，tests/src/Mod/）；tests/visual 含 42 个视觉比对条目；Fem 模块另带 femexamples 套件。
- CI 体系成熟（.github/workflows/）：CI_primary.yml 对每个 PR 跑 Prepare/Ubuntu 构建 + Lint；macOS/Windows 构建为标签门控（CI_approved.yml，"Label-gated jobs"），用独立并发组避免互相取消；配 10 个 problemMatcher（clang/cpplint/msvc/pylint/black/codespell 等，.github/problemMatcher/）实现 PR 日志内联标注；启用 dependabot 与 pre-commit（.pre-commit-config.yaml）；还有 PR 自动评分脚本 .github/scripts/score_prs.py。
- 质量观察：代码风格统一由 .clang-format/.clang-tidy 约束；SPDX 许可标识几乎遍布每个文件头；依赖策略保守（CONTRIBUTING.md:56 "A PR SHOULD refrain from adding additional dependencies"）；Python API 兼容性被明文要求（CONTRIBUTING.md:60 破坏性变更必须列明受影响插件）。

## 8. 社区与维护状态

- 活跃度：最近 push 2026-11-20（分析前一天）；每周自动发布 weekly 构建（gh api releases：weekly-2026.11.18、weekly-2026.11.11、weekly-2026.10.28……每周三准时发布），发版机器化程度高。
- Release 节奏：稳定版 1.1.0（2026-03-25）→ 1.1.1 → 1.1.2 → 1.1.3 → 1.1.4（2026-09-28）；2026-10 起切换年份版本号（26.3rc1/rc2），HEAD version.json 已是 27.1.0.dev——版本策略从语义式转向年份式，同时保持周构建。
- 贡献者结构：约 1112 名贡献者（anon 含未登录署名者），治理走 C3C 契约式流程（CONTRIBUTING.md:6 "inspired by the Collective Code Construction Contract"），明确无强制版权转让（CONTRIBUTING.md:46）。
- 生态位：FreeCAD 项目协会（FPA）提供开发者资助（README.md:91-93），基础设施获 DigitalOcean 与 KiCad Services Corp 支持（README.md:114-118）；AddonManager 以子模块方式内置（.gitmodules），插件生态经 GitHub/官方仓库分发。
- 项目 2001 年启动（源码版权头）、SourceForge SVN 时代可由版权头 sourceforge 邮箱佐证；GitHub 仓库创建于 2012-09-09（gh api created_at）；SVN→GitHub 迁移的确切年月未在源码内找到，标注为公开历史（未确认）。

## 9. 亮点与风险

亮点：
1. App/Gui/Base 三层彻底解耦：核心可无头运行，App 层零 Qt 依赖，边界由 `getViewProviderName()` 一类方法显式声明（src/App/DocumentObject.h:215），32 年演进仍保持清晰分层。
2. 绑定工程自洽：自研 generate.py 把 .pyi 同时用作绑定模型与类型存根（src/Tools/bindings/generate.py、194 个 .pyi），正从 XML 迁移，比 SWIG 更贴合自身对象系统。
3. 模块化程度极高：33+ Mod 全部遵循 App/Gui/Python 三段式约定，Python 工作台（Draft/BIM）与 C++ 工作台（Part/TechDraw）共存于同一框架，第三方扩展还有 FeaturePython 通道。
4. 工程基础设施一流：每周自动发版、标签门控 CI、10 类 problemMatcher、pre-commit/clang-tidy/dependabot 齐备，AI 时代治理也先人一步（AI_POLICY.md 强制披露、AGENTS.md 给 AI 代理的导航表）。
5. 几何内核依赖清晰可控：OCC ≥7.6 单点声明（SetupOpenCasCade.cmake），Coin3D/pivy 用自维护 fork 锁定分支（.gitmodules）。

风险/局限：
1. 依赖面广且重：OCC/SMESH/Netgen/Qt6/Coin3D/PySide/Boost/Eigen/PCL/ICU 等 20+ 外部依赖（顶层 CMakeLists.txt:97-146），从源码全量构建门槛高，Linux 发行版打包负担大。
2. 绑定机制为自研 DSL（.pyi 模型 + 模板生成），外部贡献者学习成本高于主流方案（pybind11 仅局部使用，如 CAM/tsp_solver_pybind.cpp），且历史 XML 路径仍在清理中（FreeCadMacros.cmake:123）。
3. 双求解器并存（Sketcher 内置 planegcs vs Assembly 的 OndselSolver 子模块）是过渡态架构，长期维护两条求解代码线。
4. 体量与复杂度：Core 三层抽样约 42 万行 C++（App+Base+Gui 的 .cpp/.h），TechDraw/Fem/CAM 各千余文件，代码审查与回归测试压力大（open issues 4084，gh api）。

## 10. 总结评价

FreeCAD 是"社区驱动的开源 CAD 平台"这个赛道上几乎唯一的成体系实现：32 年历史、1112 名贡献者、每周发版、33+ 模块，把参数化内核、几何内核封装、GUI 框架、脚本生态和现代工程基础设施整合进一个 LGPL 单体仓库。它的架构价值在于教科书级的 App/Gui 分离与可插拔工作台约定，工程价值在于把一个庞大的 C++/Python 混合桌面项目治理得像现代 Web 项目一样有纪律。适合机械/建筑工程师作为免费设计工具，适合 C++ 与桌面 GUI 开发者研读大型 Qt 工程的分层与模块化实践，也适合研究 Python 绑定生成、依赖治理与开源社区治理（C3C 流程 + AI 政策）的工程团队借鉴。
