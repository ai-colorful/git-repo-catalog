# aseprite/aseprite 项目分析

> 星数 37500（快照 2026-10-24）｜主语言 C++｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：业界事实标准的开源像素画与动画精灵编辑器（Animated sprite editor & pixel art tool），跨 Windows/macOS/Linux 桌面端。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | aseprite/aseprite | gh api 元数据 |
| 星数 | 37500（快照） | 任务参数 |
| 创建/最近 push | 2014-08-19 / 2026-11-10 | gh api `repos/aseprite/aseprite` |
| 语言 | C++（C++17），辅助 Lua 脚本生态 | `CMakeLists.txt` |
| 许可证 | 自研 EULA（主体）+ 多个子库 MIT；GitHub license 字段为 null | `EULA.txt`、`README.md` License 节 |
| 版本 | main 分支 `1.x-dev`；最新 release v1.3.18.6（2026-09-22） | `src/ver/CMakeLists.txt`、gh api releases |
| 贡献者 | 约 139 人（contributors API 分页计） | gh api contributors |
| 维护方 | Igara Studio S.A.（David Capello 创立） | `README.md` Credits 节 |

## 2. 项目是什么，解决什么问题

Aseprite 是为游戏开发设计的动画精灵（sprite）创作工具：以"图层 × 帧"双维度组织画面（`README.md`），支持 RGBA/索引色（256 色调色板）/灰度三种颜色模式（`src/doc/color_mode.h`、`src/doc/pixel_format.h`），实时动画预览与洋葱皮（onion skinning，`src/render/onionskin_options.h`），并可导出精灵图集（sprite sheet）、GIF、PNG 序列及 FLC/FLI 等复古格式（`src/dio/`、`src/app/doc_exporter.cpp`）。

它的差异化在于"像素级正确"：Pixel Perfect 手绘模式、Shading ink、自定义笔刷、轮廓、Wide Pixels 等专门为像素画设计的工具（`README.md` 特性列表），以及对称、平铺（Tiled mode）、压力感应等绘图辅助（同上）。撤销系统支持所有操作和非线性 undo（README；实现见第 5 节）。

工程层面它同时是一个"编辑器 + 引擎生态节点"：`.aseprite` 文件格式有公开规格说明（`docs/ase-file-specs.md`），官方维护各语言运行库读取该格式；内置 Lua 脚本 API（API_VERSION 41，`src/app/script/api_version.h`）和 CLI 自动化（`tests/cli/` 展示了 `--sheet`、`--list-layers` 等用法），让导出管线可以嵌入游戏工作流。

商业化路径独特：源码在 GitHub 公开，但官方二进制按份收费（Steam/itch.io），Steam 发行版走 Steam 订阅协议；教育机构可申请免费教育许可（`README.md` License 节）。社区可以直接源码自编译免费使用，但 EULA 限制再分发（详见第 5.5 节）。

## 3. 技术栈与依赖

- **语言**：C++17（`CMakeLists.txt` 第 7-9 行），Lua（脚本 API，third_party/lua 子模块）
- **UI/绘图框架**：自研两层——
  - **laf**（"The Lost Application Framework"，独立仓库子模块，MIT）：OS 抽象（窗口/输入/Wacom）、`base`/`gfx`/`os`/`ft`/`text`/`clip` 模块；后端可选 `skia` 或 `none`（Pixman 兜底），Skia 后端时用 Skia 做屏幕绘制与字体渲染（laf README，经 gh api 确认）
  - **src/ui**（MIT）：自研可移植控件库（button/entry/combobox/manager 消息循环等，`src/ui/`），与 laf 的 `os` 层对接
- **Skia**：Google Skia 的 aseprite 维护分支 `aseprite-m124`，提供官方预编译包（`INSTALL.md` Dependencies 节）；应用内 GPU 着色器渲染与文字、取色器等直接使用 Skia（`src/app/render/shader_renderer.cpp`、`src/app/util/render_text.cpp`）
- **IO/编解码**：自研 `src/dio`（.aseprite 格式），子模块 giflib、libpng、zlib、libwebp、curl、freetype2、harfbuzz、libarchive、tinyxml2、json11、qoi、TinyEXIF、pixman、simpleini 等（`.gitmodules` 共 27 项）
- **算法库子模块**：`src/undo`（撤销历史）、`src/observable`（信号槽）、`src/flic`（FLI/FLC）、`src/tga`、`src/psd`（均 aseprite 官方独立仓库，MIT 或随主仓 EULA，见 `src/README.md` Level 0 列表）
- **测试/基准**：Google benchmark（third_party/benchmark）；自研 shell 脚本端到端测试
- **构建**：CMake ≥3.20 + Ninja；MSVC（Windows，明确不支持 MinGW）/Xcode/clang（`INSTALL.md`）

## 4. 目录结构与代码组织

| 目录 | 职责 | 出处 |
|---|---|---|
| `src/app/` | 应用层主体：命令系统、撤销事务、编辑器 UI、脚本引擎、工具（~1093 个文件条目） | `src/app/` |
| `src/doc/` | 文档模型库（MIT）：Sprite/Layer/Cel/Image/Palette/Tileset/Slice/Tag | `src/doc/` |
| `src/render/` | 渲染库（MIT）：合成、混合、抖动、量化、洋葱皮 | `src/render/` |
| `src/ui/` | 可移植控件库（MIT）：widget 基类、消息循环、主题 | `src/ui/` |
| `src/dio/` | 文件读写库（MIT）：.aseprite 编解码、格式探测 | `src/dio/` |
| `src/filters/` | 图像效果（卷积、色彩曲线等） | `src/filters/` |
| `src/view/` | 抽象 timeline/range 视图辅助 | `src/README.md` |
| `src/laf/`（子模块） | OS 抽象层：base/gfx/os/ft/text/clip | laf 仓库 |
| `src/main/` | 入口（main.cpp + win/osx 平台细节） | `src/main/` |
| `src/cfg`,`src/net`,`src/gen`,`src/updater`,`src/steam`,`src/desktop` | ini 配置、HTTP、代码生成、更新检查、Steam 封装、桌面集成 | `src/README.md` |
| `data/` | 运行时资源：gui.xml（菜单/快捷键声明）、widgets/、主题扩展（aseprite-theme）、社区调色板、字体、多语言 strings/ | `data/` |
| `tests/` | 端到端：cli/（bash 用例）、scripts/（67 个 Lua API 测试）、sprites/（格式回归样本） | `tests/README.md` |
| `docs/` | 文件格式规格、编码风格、依赖许可清单 | `docs/` |

组织评述：严格的**分层单体**。`src/README.md` 官方定义 Level 0→5 依赖层级（Level 0 完全独立库 → Level 5 main），每层只能依赖低层；可复用库（doc/render/ui/dio）单独 LICENSE 为 MIT，与主程序 EULA 分离。第三方依赖以子模块 vendored 进 `third_party/`，并保留 `USE_SHARED_*` CMake 选项允许 Linux 发行版换用系统库（`CMakeLists.txt` 67-76 行）。

## 5. 核心架构与关键实现

### 5.1 文档模型（src/doc/）

`Sprite` 是全程序核心结构（`src/doc/sprite.h`："The main structure used in the whole program to handle a sprite"），上限 65535×65535（同文件宏）。层级关系为 Sprite → Layer（含 LayerGroup/LayerImage/LayerTilemap，`src/doc/layer_tilemap.h`）→ Cel → Image；`CelData`（`src/doc/cel_data.h`）用引用计数（`int refs()`）持有 Image 与位置/透明度，多个 Cel 可共享同一 CelData——这正是"链接 cels"功能的实现基础。颜色模式下发到模板化 Image（`src/doc/image_traits.h`），索引色映射由 octree（`src/doc/octree_map.cpp`）与 RGB5A3（`src/doc/rgbmap_rgb5a3.cpp`）两套算法承担。

### 5.2 撤销系统（src/app/cmd/ + src/undo）

每个文档操作是一个 `Cmd` 子类，继承自独立 MIT 库 `undo::UndoCommand`（`src/app/cmd.h`）：

```cpp
class Cmd : public undo::UndoCommand {
public:
  void execute(Context* ctx);
  void undo() override;
  void redo() override;
protected:
  virtual void onExecute();
  virtual void onUndo();
  virtual void onRedo();
};
```

`src/app/cmd/` 下有 194 个文件（add_cel、clear_rect、copy_region、flip_image…），即"一条命令一个类"的命令模式全集。`Transaction`（`src/app/transaction.h`）把多个 Cmd 打包为原子事务，异常时整体回滚，并监听 DocObserver 在事务结束时统一刷新 UI。每次提交通过 `DocUndo`（`src/app/doc_undo.*`）入历史栈，支持非线性 undo（README 特性表；undo 库为 aseprite/undo 独立仓库，`src/README.md`）。

### 5.3 渲染管线（src/render/ + src/app/render/）

底层 `render::Render`（MIT，`src/render/render.h`）以函数指针 `CompositeImageFunc` 组织合成：按 RenderPlan 生成的 cel 序列（按图层顺序 + zIndex 排序，`src/doc/render_plan.h`）逐层 blend（`src/render/blend_funcs.cpp` 定义新旧两种混合公式），并叠加洋葱皮、背景棋盘格、参考图层。上层 `app::Renderer` 接口（`src/app/render/renderer.h`）有两个实现：`SimpleRenderer`（纯 CPU）与 `ShaderRenderer`（Skia GPU 加速，`src/app/render/shader_renderer.cpp` 中可见 SkImages/SkImageInfo 调用）；开发者模式下 F1 可在 new/old/shader 三个渲染器间切换做对比（`src/README.md` Debugging Tricks）。

### 5.4 UI 框架与 Skia 的关系（src/ui/ + laf）

`src/ui` 是与业务无关的控件库（Widget/Manager 消息循环，`src/ui/manager.cpp` 等），落屏交给 laf 的 `os` 抽象；构建时 `LAF_BACKEND=skia`（指定 `SKIA_DIR` 时自动选择，`CMakeLists.txt` 127-135 行）即由 Skia 承担光栅化，`LAF_BACKEND=none` 退化到 Pixman/Pure CPU 路径。主题系统放在 data/extensions/aseprite-theme，控件树用 XML 声明（`data/gui.xml`、`src/app/widget_loader.cpp` 解析）。

### 5.5 许可模式：EULA + MIT 双轨（客观陈述）

- 主程序（`src/app/`、`src/main/` 等所有文件头）与官方二进制按自研 **EULA** 授权（`EULA.txt`）：权利人为 Igara Studio S.A.；其中 2(g) 条规定"仅可为个人目的编译/修改源码，或为向本项目贡献提案"；2(b) 禁止向第三方再分发副本。
- 核心可复用库显式 **MIT**：laf、clip、undo、observable 及仓内 `src/ui`、`src/doc`、`src/render`、`src/dio`、`src/filters`（各自 `LICENSE.txt`，文件头注释亦注明 "released under the terms of the MIT license"）；`README.md` License 节明确列出这一划分。
- 收费边界：官方编译版（含 Steam/itch.io 发行、自动更新、Steam 集成 `src/steam/`）是付费点；源码本身公开可自编译。GitHub license 检测为 null（gh api `license: null`），因此不被 OSI 生态识别为开源许可——社区常称其为"源码可得（source-available）"模式。
- 付费版还启用 `ENABLE_DRM`/`ENABLE_TRIAL_MODE`/`ENABLE_STEAM` 等编译开关（`CMakeLists.txt` 85-87 行，DRM 逻辑在私有 drm 仓库，构建时若启用则要求本地存在 `drm/CMakeLists.txt`，同文件 137-139 行）。

### 5.6 脚本 API（src/app/script/）

Lua 引擎（third_party/lua）上封装了 60+ 个绑定文件：`sprite_class.cpp`、`image_class.cpp`、`dialog_class.cpp`、`websocket_class.cpp` 等，`engine.cpp` 负责生命周期；API 以 `API_VERSION 41` 做兼容性闸口（`src/app/script/api_version.h`），并有 `security.cpp` 控制脚本权限（如打开文件对话框）。官方 API 兼容性测试直接用 67 个 Lua 脚本驱动真程序（`tests/scripts/`）。

## 6. 构建与运行

官方辅助脚本（`build.sh` / `build.ps1` / `build.cmd`，交互式下载预编译 Skia）：

```bash
git clone --recursive https://github.com/aseprite/aseprite.git
cd aseprite && ./build.sh --auto --norun   # 自动配 Skia + cmake + ninja
# 产物: <build>/bin/aseprite
```

手动 CMake（`INSTALL.md` Manual Building 节）：

```bash
cmake -B build -S . -G Ninja -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DLAF_BACKEND=skia -DSKIA_DIR=/path/to/skia -DSKIA_LIBRARY_DIR=/path/to/skia/out/Release-x64
cmake --build build
```

关键开关（`CMakeLists.txt` 67-98 行）：`ENABLE_SCRIPTING`（默认 on）、`ENABLE_TESTS`、`ENABLE_BENCHMARKS`、`ENABLE_STEAM`、`ENABLE_DRM`、`ENABLE_TRIAL_MODE`、`USE_SHARED_LIBPNG/ZLIB/FREETYPE…`（系统库替换）。要求 CMake 3.20+、Ninja、C++17；Windows 仅支持 MSVC（不支持 MinGW），Skia 需 `aseprite-m124` 分支（官方提供预编译包）。

## 7. 测试与工程质量

- **单元测试**：仓内 23 个 `*_tests.cpp`（doc/image_tests、doc/sprite_tests、doc/render_plan_tests、render/ordered_dither_tests、app/app_tests 等，`src/**/*_tests.cpp`），由 `ENABLE_TESTS` 开关接入 CTest（`CMakeLists.txt` 61 行 `enable_testing()`）。
- **基准**：blend_benchmark、primitives_benchmark（Google benchmark，`src/doc/blend_benchmark.cpp` 等）。
- **端到端**：`tests/run-tests.sh` 对编译产物跑 CLI 用例（`tests/cli/version.sh`、`sheet.sh`、`file-formats.sh`…），依赖 `tests/sprites/` 固定样本做格式回归（`tests/README.md`）；脚本 API 兼容测试 67 个 Lua 文件（`tests/scripts/`）。
- **CI**：`.github/workflows/build.yml` 三平台矩阵（windows/macos/ubuntu × RelWithDebInfo/Debug × gui/cli × lua/noscripts），自动下载预编译 Skia + ccache；另有 clang_tidy、precommit、lint_commit 工作流（`.github/workflows/`）。开发规范见 `docs/CODING_STYLE.md`，提交走 lint_commit 校验。

## 8. 社区与维护状态

- **活跃度**：最近 push 2026-11-10（分析日 4 天前），gh api 确认常年高频提交；CI 徽章 + WebEstate 翻译平台（hosted weblate 徽章，`README.md`）。
- **Release 节奏**：密集补丁式——v1.3.18.2→.6 间隔约 2-6 周（gh api releases，2026-08 至 2026-09 连发 5 个）。
- **团队结构**：Igara Studio 全职团队 + 社区 PR；`AUTHORS.md` 列出核心开发名单；CODEOWNERS、PR/issue 模板齐全（`.github/`）。
- **社区渠道**：Discourse 论坛、Discord、每年付费用户支撑商业模式（`README.md` Support 节）。issue 积压约 2066 个（gh api），有专门 `close_spam.yml` 反垃圾工作流。

## 9. 亮点与风险

**亮点**

1. 分层纪律严明的架构：官方 Level 0-5 模块分层（`src/README.md`），核心库与主程序解耦甚至拆成独立 MIT 仓库（undo/observable/laf/clip），工程可测试性和复用性极佳。
2. 命令模式做撤销的教科书实现：194 个 Cmd 类 + 原子 Transaction + 非线性历史（`src/app/cmd/`、`src/app/transaction.h`），每个操作可精确逆操作。
3. 依赖治理成熟：27 个子模块全部 vendored 且提供 `USE_SHARED_*` 逃生门，Skia 官方 fork 带预编译产物，跨平台构建体验在 C++ 桌面项目中属第一梯队（`INSTALL.md`、`build.sh` 自动化）。
4. 测试金字塔覆盖格式回归与脚本 API 兼容（`tests/` 三目录），多维度 CI 矩阵保证 2×2×3 配置组合不烂尾。
5. 自研 UI/laf + Skia 的组合在性能与体积间取得平衡，且 laf/ui 已反哺为独立可复用库。

**风险/局限**

1. 主程序 EULA 不是 OSI 开源许可（GitHub license=null），不能自由再分发二进制、不可用于发行版打包聚合，引用/集成前必须读 `EULA.txt`（尤其 2(b)/2(g)）。
2. 构建重：需要特定 Skia 分支（aseprite-m124）、CMake+Ninja、以及版本敏感的工具链（Win 仅 MSVC），离线自编译门槛高。
3. 文档与代码同步依赖仓内 README：架构文档少而精但更新不保证，深层机制（如新旧 blend 公式差异）需读代码与 git 历史确认。
4. tarball 自编译产物不可再分发 + 自动更新/DRM 在私有仓库，二次开发者无法获得完整付费版功能链路。

## 10. 总结评价

Aseprite 是"商业级 C++ 桌面应用如何做开源"的标杆样本：源码可得但通过自研 EULA 保留商业化（付费二进制 + Steam 发行），同时把真正可复用的核心（文档模型、渲染、UI、undo）以 MIT 独立成库回馈生态。技术上它展示了分层单体架构、命令模式撤销系统、双渲染器管线（CPU/Skia GPU）与自研 UI 框架的完整实践，测试与 CI 纪律严明，20 余年持续演进（2001 年版权头至今）。适合：游戏开发者把它当生产力工具；C++ 工程师把它当桌面应用架构教科书（重点读 `src/README.md`、`src/doc/`、`src/app/cmd/`、`src/render/`）；想学习"source-available 商业模式"的维护者参考其许可划分。不适合：需要 OSI 开源许可再分发的场景。
