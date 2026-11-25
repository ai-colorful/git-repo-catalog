# carbon-language/carbon-lang 项目分析

> 星数 32500（快照 2026-10-24）｜主语言 C++｜档位 20k-50k｜分析日期 2026-11-21
> gh api 复核（2026-11-21）：实际 33968 星、1626 forks、默认分支 trunk、最近 push 2026-11-21，Google 主导、活跃。

## 1. 项目速览

Carbon 是 Google 主导的"C++ 继任者"实验语言（successor language，类比 JavaScript→TypeScript、Java→Kotlin），由 Chandler Carruth 在 2022 年 CppNorth 大会公开宣布，本仓库是其唯一主仓库：语言设计文档、提案流程、工具链实现与标准库全部在此。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | carbon-language/carbon-lang | gh api |
| 星数 | 33968（任务快照 32500 @2026-10-24） | gh api |
| 主语言 | C++（750 个 .cpp/.h/.cppm，约 20.2 万行） | 源码统计 |
| 许可证 | Apache-2.0 WITH LLVM-exception（`LICENSE`，GitHub 标记 NOASSERTION） | LICENSE |
| 创建 | 2020-04-27（设计工作早于 2022 公开宣布） | gh api |
| 默认分支 | trunk（非 main/master） | gh api |
| 最近活跃 | push 2026-11-21（当日），最新提交 2026-11-20 by Chandler Carruth（PR #8162+） | gh api / commits API |
| 版本状态 | 0.0.0（`version_base.bzl`："尚未达到 0.1 里程碑"），仅每日 nightly 预发布 | version_base.bzl、releases API |

## 2. 项目是什么，解决什么问题

C++ 统治性能敏感软件领域，但几十年技术债使渐进演化极其困难——README 用 `docs/project/difficulties_improving_cpp.md` 专门论证"为什么不继续改进 C++"。Carbon 的判断是：能用 Go/Rust/Swift 的开发者应该直接用它们；剩下被锁死在 C++ 生态上的大规模代码库和开发者，需要一个能与 C++ 双向无缝互操作、迁移成本可控的继任语言。这是"继任语言"路线而非"C++ 演化"路线。

语言目标（README + `docs/project/goals.md`）：性能对齐 C++（基于 LLVM）、与 C++ 双向互操作（含继承与模板）、对 C++ 开发者平缓的学习曲线、可规模化的迁移（源到源翻译工具）。明确的非目标：全语言稳定 ABI、完美前后向兼容——这是与 ISO C++ 标准化最根本的路线分歧。

内存安全是当前叙事的重心（2025 路线图两项核心目标之一）。README 给出两步迁移：先高度自动化迁到"为 C++ 互操作设计的 Carbon 方言"，再增量重构到内存安全子集；并在迁移中立即获得初始化跟踪、动态边界检查、以 erroneous behavior 替代 UB、`unsafe` 显式标注等收益。

项目状态非常克制：README 明确写着 "not ready for use"。当前精力全部投向工具链，直到能支撑 Carbon↔C++ 互操作、并达到 0.1 语言（`docs/project/milestones.md`：0.1 MVP → 0.2 功能完整 → 1.0 脱离实验）。早期原型解释器 explorer（旧设计版本）已停止开发并归档移出主仓库，移至独立的 carbon-language/explorer 仓库（README "Project status" 一节）——这是设计大转向的客观记录，旧实现与新 toolchain 不再并存。

## 3. 技术栈与依赖

- **实现语言**：C++（工具链 611 个文件 + `common/` 105 个通用支撑库文件）。
- **后端/互操作底座**：LLVM/Clang/LLD，通过 Bazel `git_override` 跟踪上游 top-of-tree（MODULE.bazel 注释：HEAD as of 2026-11-17，commit `c03c4571b1b0…`），不锁定 release 版本。
- **构建**：Bazel（bazelisk，`scripts/run_bazelisk.py` 包装）；`MODULE.bazel` 声明 abseil-cpp、re2、googletest、google_benchmark、tcmalloc、boost.unordered、protobuf、zlib-ng/zstd/xz（供 LLVM）等。
- **辅助脚本**：Python 3.12（`pyproject.toml`，ruff + ty 静态检查），59 个 .py 约 1.1 万行，覆盖构建包装、CI 校验、GitHub 工具（`github_tools/`）。
- **标准库**：用 Carbon 自己写——`core/` 下 32 个 .carbon 文件约 3700 行（prelude 含 int/float/string/optional/array/slice 等类型，`core/prelude/types/` 内还有 `cpp/` 子目录服务互操作）。
- **文档**：356 个 Markdown（设计 58 篇、项目管理 31 篇、提案 193 篇、spec 9 篇）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `toolchain/` | 编译器工具链主体（611 个 C++ 文件，当前实现重心） |
| `common/` | 跨工具链的 C++ 通用库（命令行解析、错误处理、enum 基类等） |
| `core/` | Carbon 标准库 prelude（Carbon 语言编写，含少量 C++ 互操作部分） |
| `docs/` | 设计文档（design/）、项目治理（project/）、指南（guides/）、WIP 语言规范（spec/） |
| `proposals/` | 已接受提案存档（193 份，`p######-slug.md` 以 PR 号编号） |
| `testing/` | 测试框架（file_test 黄金测试框架、fuzzing 支撑） |
| `examples/` | Carbon 示例（hello_world.carbon、interop/、advent2024/、re2_playground/） |
| `scripts/`、`github_tools/` | Bazel 包装、仓库校验脚本、PR/CI 辅助工具 |
| `bazel/` | LLVM 等外部依赖的 Bazel 补丁 |
| `website/` | GitHub Pages 站点（Jekyll） |
| `.github/` | 12 个 workflow（tests、nightly_release、benchmarks、gh_pages、clangd_tidy 等） |
| `utils/vscode/` | VS Code 语法高亮插件（TextMate grammar） |

组织方式为单仓多模块 monorepo：设计文档与实现同仓、以 Bazel BUILD 文件细粒度管理依赖、每个编译步骤一个独立子目录（输入输出结构明确划分）。

## 5. 核心架构与关键实现

### 5.1 六步编译流水线（toolchain/docs/README.md "High-level architecture"）

1. Source → `SourceBuffer`（`toolchain/source/source_buffer.h`）
2. Lex → `Lex::TokenizedBuffer`（`toolchain/lex/tokenized_buffer.h`）
3. Parse → `Parse::Tree`（`toolchain/parse/tree.h`）
4. Check → `SemIR::File`（语义分析与类型检查，产出语义 IR）
5. Lower → LLVM Module（`toolchain/lower/`）
6. CodeGen → 目标文件

各步骤产出独立数据结构、不做回调穿透；数据以向量 + flyweight 索引存储（如 `llvm::SmallVector<Parse::Tree::NodeImpl>`，索引包装 int32_t）；解析用状态栈迭代而非递归。这三条设计模式在 `toolchain/docs/README.md` 明文记录，目标是缓存局部性与可并行。

### 5.2 各阶段代码落点（按文件数）

- **check/（197 个文件，最大模块）**：入口 `check.cpp`（620 行）；按语言特性拆分实现文件（class.cpp、call.cpp、deduce.cpp、convert.cpp、eval.cpp、facet_type.cpp 等）+ `check/cpp/` 子目录 30 个文件专做 C++ 互操作——`generate_ast.cpp` 调 Clang 建 AST、`overload_resolution.cpp` 复用 C++ 重载决议、`impl_lookup.cpp`/`type_mapping.cpp`/`thunk.cpp` 处理跨语言类型映射与 thunk 生成，配套设计文档 `toolchain/docs/check/cpp/thunks.md`。
- **sem_ir/（114 个文件）**：自研语义 IR，`inst.h`（726 行）定义指令模型，`file.h`（533 行）为编译单元容器；SemIR 是 Check 与 Lower 之间的稳定中间层，也是诊断与工具共享的语义真相源。
- **parse/（71 个文件）**：手写状态机解析器，`handle_*.cpp` 每个语法构造一个处理函数（handle_class.cpp、handle_brace_expr.cpp 等），`coverage_test.cpp` 强制覆盖统计。
- **lex/（36 个文件）**：数值/字符串字面量独立处理（numeric_literal.cpp、string_literal.cpp），每个核心文件配对 _test/_fuzzer/_benchmark 三件套。
- **lower/（23 个文件）**：SemIR→LLVM IR，`file_context.cpp`、`clang_global_decl.cpp` 直接引用 clang/ 头，把导入的 C++ 声明降入同一 LLVM 模块——互操作在 IR 层汇合的落地证据。
- **driver/（58 个文件）**：`carbon` 命令行入口，含 `clang_runner.cpp`、`clang_subcommand.cpp`、`lld_subcommand.cpp`——单个可执行文件内嵌 Clang 与 LLD，这是"混合工具链"策略的实现基础。
- 其他：`language_server/`（LSP 支持）、`format/`（格式化器）、`diagnostics/`（诊断产生 + 终端渲染）、`install/`（打包）。

### 5.3 语言样貌（examples/hello_world.carbon）

```carbon
import Core library "io";

fn Run() {
  Core.PrintStr("Hello world!\n");
}
```

### 5.4 C++ 互操作设计（文档）

`docs/design/interoperability/philosophy_and_goals.md` 确立目标：支持 Carbon/C++ 混合工具链、兼容 C++ 内存模型、最小化桥接代码、C++ 类型到 Carbon 的直观映射、允许在 Carbon 文件内写 C++ 桥接代码、Carbon 继承 C++ 类型；非目标包括全对等的单工具链、跨语言元编程、C++ 异常无桥支持。互操作被定位为服务于"迁移"而非通用 FFI，优先级低于性能与演化（引用 `docs/project/goals.md` 的优先级序）。

## 6. 构建与运行

来自 README "Getting started" 与 `docs/project/contribution_tools.md`：

```bash
# 构建依赖（Ubuntu）：clang、libc++-dev、libc++abi-dev、lld
git clone https://github.com/carbon-language/carbon-lang && cd carbon-lang

# 构建 toolchain 并查看帮助
./scripts/run_bazelisk.py run //toolchain -- help

# 试用 nightly（releases 页下载 carbon_toolchain-0.0.0-0.nightly.YYYY.MM.DD.tar.gz）
echo "import Core library \"io\"; fn Run() { Core.Print(42); }" > forty_two.carbon
./carbon_toolchain-*/bin/carbon compile --output=forty_two.o forty_two.carbon
sudo apt install libgcc-11-dev   # 链接仍借用 GCC 系统库
./carbon_toolchain-*/bin/carbon link --output=forty_two forty_two.o
./forty_two
```

浏览器零安装试用：carbon.compiler-explorer.com。README 提醒工具链"仍处早期、很多功能不工作"，0.1 里程碑前不鼓励大量报 bug。

## 7. 测试与工程质量

- **黄金测试**：`toolchain/` 内 1795 个 testdata `.carbon` 用例，经 `testing/file_test/` 框架跑驱动级测试（语言实验仓常见的"源文件内嵌期望输出"风格）。
- **模糊测试**：lex 各核心文件配 _fuzzer（numeric_literal_fuzzer.cpp 等），check 有 `check_fuzzer.cpp`，parse/lex 均有 fuzzer_corpus 目录。
- **基准测试**：`toolchain/benchmarking/`（6 个文件）+ `scripts/bench_runner.py` + 独立 CI workflow `benchmarks.yaml`；最新提交仍在做基准回填（PR #8162，2026-11-20）——编译器性能被当作一等指标。
- **集成测试**：`integration_tests/`（make_unique_test.carbon 等端到端用例）。
- **CI**：`tests.yaml`（持续测试）、`clangd_tidy.yaml`（clang-tidy）、`prek.yaml`（pre-commit 钩子集）、`gh_pages_ci/deploy.yaml`、`nightly_release.yaml`（每日 02:00 UTC 构建 x86_64 Linux tarball 发布，workflow 内 TODO 注明暂只构建单平台）。
- 代码质量观察：协议头/风格统一（.clang-format、.clang-tidy、cpplint 式头检查 `scripts/check_header_guards.py`），Python 侧有 ty 全量类型检查规则。

## 8. 社区与维护状态

- **规模与活跃**（gh api，2026-11-21）：33968 星、1626 forks、266 open issues、贡献者约 192 人（含匿名）；最近 push 当日，最新三次提交均出自 Chandler Carruth，PR 编号已到 #8162+。
- **发版节奏**：无语义化版本，`version_base.bzl` 固定 0.0.0；releases 页每天一个 nightly（v0.0.0-0.nightly.2026.11.21 等，releases API 证实连续每日发布），nightly_release.yaml 自动化产出，仅 x86_64 Linux tarball。
- **治理**（`docs/project/evolution.md`）：重大变更走提案制——PR 加入 `proposals/` 目录、模板化文档、draft → ready for review → RFC 评论 → leads 决议，193 份已接受提案存档（p000024 → p008110+）。当前三位 Carbon leads：chandlerc（Chandler Carruth，发起人）、KateGregory（Kate Gregory，知名 C++ 布道者）、zygoloid（Richard Smith，Clang 主力实现者）——决策用 blocking consensus（法定人数 2）。另设趣味性 "Painter" 角色专门拍板 bikeshed 类争议。子团队、行为准则团队（`docs/project/teams/conduct_team.md`）、透明度报告（`docs/project/transparency_reports.md`）齐备。
- **对外互动**（README 会议演讲清单 2022-2026）：CppNorth/C++Now/NDC/EuroLLVM/LLVM DevMeeting/RustConf 常态化演讲，2026 年主题集中于内存安全深潜与"graduating from the experiment"；主设计讨论在 Discord。与 ISO C++ 委员会无正式关系（未确认有官方互动记录），路线是绕开标准化流程的社区实验。
- **Google 主导模式的客观痕迹**：CLA 要求（CONTRIBUTING.md）、origin 工作流、核心实现提交高度集中于 chandlerc 等少数人。

## 9. 亮点与风险

**亮点**
1. 团队配置罕见：C++ 生态最有影响力的几位实践者（Chandler Carruth、Richard Smith、Kate Gregory）亲自下场，Clang 实现经验直接内化为工具链设计。
2. 架构先进且自洽：六步解耦流水线、向量化 flyweight 存储、自研 SemIR、单二进制内嵌 Clang/LLD，C++ 互操作在 checker（check/cpp/）与 lower（clang_global_decl.cpp）两层同时落地，是当前同类实验中互操作路径最扎实的实现。
3. 工程化治理典范：提案编号存档、演进流程文档化、goals/roadmap/milestones/版本策略环环相扣，193 份提案构成可追溯的决策日志。
4. 测试与性能文化：1800 个黄金测试、全链路 fuzzer、nightly 基准与回填，编译器以"正确性→代码质量→编译速度→诊断质量"排序的明确目标（toolchain/docs/README.md）。
5. 文档/实现同仓同步：设计文档、内部设计文档（toolchain/docs/design/clang_api.md 等）与代码一起演进。

**风险/局限**
1. 距可用尚远：版本 0.0.0、README 明言 not ready for use，nightly 仅 x86_64 Linux，0.1 里程碑（roadmap 潜在目标 2026）一再依赖后续年份的"potential"表述，时间表存在不确定性。
2. 构建/开发门槛极高：跟踪 LLVM top-of-tree（每日漂移）+ Bazel + Clang/libc++/lld 全套依赖，贡献者需要相当的环境投入。
3. 存活押注在 C++ 社区采纳与 Google 持续投入上：explorer 整体废弃说明实验存在战略级转向先例；若 0.1 评估未形成"临界规模兴趣"，项目可能收缩。
4. star 高但贡献集中：约 192 名贡献者中实际核心实现者高度集中，社区参与以提案评论和 Discord 讨论为主。

## 10. 总结评价

carbon-lang 是语言设计方法论、编译器工程与开源治理三方面都达到教科书画质的实验仓库：它不试图成为下一个 Rust，而是押注"C++ 大规模存量的平滑出路"这一细分命题，并用内嵌 Clang 的混合工具链把互操作从口号做进了源码。对语言设计者，docs/ 与 proposals/ 是理解如何以工程方式设计一门语言的范本；对编译器工程师，toolchain/ 的 SemIR、check/cpp/ 互操作层与向量化解析器值得精读；对普通 C++ 开发者，它目前只适合在 compiler-explorer 上旁观，而非生产采用。星数反映的是社区对"C++ 出路"命题的关注度，项目的真实进度应以其 0.1 里程碑的达成为准。
