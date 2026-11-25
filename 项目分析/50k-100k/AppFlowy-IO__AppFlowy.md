# AppFlowy-IO/AppFlowy 项目分析

> 星数 77672（快照 2026-10-24，API 实测 78005）｜主语言 Dart｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：开源的 Notion 替代品——用 Flutter + Rust 构建的 AI 知识与项目工作区，覆盖文档、看板/表格/日历数据库、Wiki 与 AI 助手，数据可完全自托管。

| 项目 | 内容 |
|---|---|
| 名称 | AppFlowy-IO/AppFlowy |
| 星数 / Fork | 78005 / 6132（gh api 2026-10-31） |
| 主语言 | Dart（UI）+ Rust（核心业务） |
| 许可证 | AGPL-3.0（LICENSE 文件） |
| 版本 | 主分支 pubspec 0.11.4（frontend/appflowy_flutter/pubspec.yaml）；GitHub 最新 release 0.14.11（2026-10-26，target main） |
| 最近活跃 | 最近 push 2026-10-28（gh api），月均 2-5 个 release |
| 贡献者 | 362 人（gh api contributors 分页求和） |
| 平台 | Windows / macOS / Linux / iOS / Android（frontend/appflowy_flutter 下五套平台目录） |

## 2. 项目是什么，解决什么问题

AppFlowy 自我定位是 "The Open Source Alternative To Notion"（README.md 开头），是一个集文档编辑、结构化数据库（Grid/Board/Calendar）、团队 Wiki 与 AI 能力于一体的工作区应用。它解决的核心问题是：Notion 类协作工具的数据主权——README 强调 "achieve more without losing control of your data"，支持 AppFlowy Cloud 与完全自托管两种模式。

架构上最显著的特点是"胖客户端"设计：所有业务逻辑（文档 CRDT、数据库引擎、文件夹树、用户认证、AI 调用）都在 Rust 侧实现，Flutter 只负责 UI 渲染与交互。Rust 编译为 C-ABI 动态库（`libdart_ffi.so/.dylib`、`dart_ffi.dll`），通过 FFI 嵌入 Flutter 进程（见 frontend/appflowy_flutter/packages/appflowy_backend/lib/ffi.dart）。这让同一套核心逻辑可以同时支撑桌面与移动端，也为未来 Web/Tauri 前端复用核心留了口子（frontend/Makefile.toml 中已扩展 web.toml、tauri.toml 任务）。

文档采用块（block）树模型而非自由文本：一篇文档是 `page_id + blocks 映射 + meta(children_map, text_map)`（frontend/rust-lib/flowy-document/src/document_data.rs），底层由 AppFlowy 自研的 collab-rs（基于 yrs，即 Yjs CRDT 的 Rust 移植）驱动，天然支持实时协作、离线编辑与 undo/redo。

数据库视图（表格、看板、日历）是同一份数据的不同呈现，在 Flutter 侧以不同"插件"实现（lib/plugins/database/ 下 grid、board、calendar 三个子插件共用 database_document 数据）。

## 3. 技术栈与依赖

- **UI 层**：Flutter/Dart，状态管理用 bloc（pubspec.yaml: `bloc: ^9.0.0`），自研 UI 组件包 `appflowy_ui`、`appflowy_popover`、编辑器用 `appflowy_editor`（均为 AppFlowy-IO 自家包，pubspec.yaml dependencies）
- **核心层**：Rust workspace 共 31 个 crate（frontend/rust-lib/Cargo.toml members），关键依赖：
  - `collab`/`collab-document`/`collab-database`/`collab-folder` 0.2（crates.io，AppFlowy-IO 自研，yrs CRDT 内核，Cargo.toml 90-97 行）
  - `diesel 2.1` + `r2d2`（SQLite ORM 与连接池，本地持久化，Cargo.toml 74-81 行）
  - `tokio` + `allo_isolate`（异步运行时与 Dart isolate 通信，dart-ffi/src/lib.rs）
  - `protobuf 2.28`（Dart↔Rust 消息序列化）
- **数据序列化**：Rust 端 prost/自定义 derive（flowy-derive），Dart 端生成的 protobuf 类
- **构建编排**：cargo-make（frontend/Makefile.toml），CI 用 GitHub Actions + codemagic.yaml
- **国际化**：project.inlang（inlang 格式），配套 ninja_i18n.yml 工作流

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| frontend/appflowy_flutter/ | Flutter 应用（lib/ 1526 个 dart 文件），含五端平台工程 |
| frontend/rust-lib/ | Rust 核心 workspace（31 crate，全部 Rust 代码所在） |
| frontend/rust-lib/dart-ffi/ | FFI 入口层，唯一暴露给 Dart 的 C ABI |
| frontend/rust-lib/flowy-core/ | 核心组装器，初始化各模块并织入事件分发 |
| frontend/rust-lib/lib-dispatch/ | 自研事件分发运行时（dispatcher/runtime/module） |
| frontend/rust-lib/build-tool/ | flowy-ast/flowy-derive/flowy-codegen，事件与 protobuf 代码生成 |
| frontend/rust-lib/flowy-document/ | 文档模块（block 模型、协作、快照） |
| frontend/rust-lib/flowy-database2/ | 数据库模块（Grid/Board/Calendar 引擎） |
| frontend/rust-lib/flowy-folder/ | 工作区/视图树管理 |
| frontend/rust-lib/flowy-user/ | 认证、用户资料、工作区成员 |
| frontend/rust-lib/flowy-server/ | 服务端抽象：af_cloud（AppFlowy Cloud）与 local_server 双实现 |
| frontend/rust-lib/flowy-ai/ | AI 对接（本地/云 LLM） |
| frontend/rust-lib/flowy-sqlite(-vec)/ | 本地 SQLite 持久化与向量检索 |
| frontend/rust-lib/event-integration-test/ | Rust 侧无头集成测试框架（61 个 rs 文件） |
| frontend/appflowy_flutter/packages/ | 7 个本地 Dart 包：appflowy_backend（FFI 绑定）、appflowy_ui、flowy_infra 等 |
| doc/ | 贡献指南、路线图 |
| .github/workflows/ | 13 个 CI 工作流（flutter/rust/ios/mobile/docker/release/commit_lint 等） |

组织方式是典型的"双语言分层 monorepo"：一个 Flutter 应用 + 一个 Rust workspace，通过生成的绑定桥接；`*-pub` 后缀的 crate（如 flowy-document-pub）是对外实体/类型层，供其他 crate 依赖解耦。

## 5. 核心架构与关键实现

### 5.1 Dart↔Rust FFI 桥（本项目最核心的机制）

Dart 侧 `appflowy_backend` 包（frontend/appflowy_flutter/packages/appflowy_backend/lib/ffi.dart）按平台打开动态库，注册的两个 C 函数是全部请求的入口：

```dart
/// C function `async_event`.
void async_event(
  int port,
  Pointer<Uint8> input,
  int len,
) {
  _invoke_async(port, input, len);
}
final _invoke_async_Dart _invoke_async = _dart_ffi_lib
    .lookupFunction<_invoke_async_C, _invoke_async_Dart>('async_event');
```

Rust 侧（frontend/rust-lib/dart-ffi/src/lib.rs）`DartAppFlowyCore` 持有核心的懒加载句柄，收到请求后经 tokio mpsc 队列传给单线程 LocalSet 运行时执行，完成后通过 `allo_isolate::Isolate` 把 protobuf 编码结果回投 Dart SendPort。响应经 dispatch.dart 解码为带类型的事件 API（`part 'dart_event/flowy-document/dart_event.dart'` 等生成文件）。通知（后端→Dart 推送）走独立的 DartNotificationSender 流（dart-ffi/src/notification.rs，flowy-notification crate）。

### 5.2 事件系统与代码生成

每个业务 crate 用 `AFPlugin` 声明自己的一组事件，如文档模块（frontend/rust-lib/flowy-document/src/event_map.rs）注册了 CreateDocument/OpenDocument/ApplyAction/Undo/Redo/SetAwarenessState 等 21 个事件，输入输出全部是 `*PB` protobuf 结构体：

```rust
pub fn init(document_manager: Weak<DocumentManager>) -> AFPlugin {
  AFPlugin::new()
    .name(env!("CARGO_PKG_NAME"))
    .state(document_manager)
    .event(DocumentEvent::CreateDocument, create_document_handler)
    .event(DocumentEvent::OpenDocument, open_document_handler)
    .event(DocumentEvent::ApplyAction, apply_action_handler)
    .event(DocumentEvent::Redo, redo_handler)
    .event(DocumentEvent::SetAwarenessState, set_awareness_local_state_handler)
}
```

事件枚举标注 `#[derive(ProtoBuf_Enum, Flowy_Event)]`，由 build-tool/flowy-codegen 在构建时反向生成 Dart 端 `dart_event/` 类型化调用与 protobuf 类（tarball 中 packages/appflowy_backend/lib 下不存在 dart_event/、protobuf/ 目录，证明这些是构建产物而非提交物，frontend/scripts/code_generation/ 有生成脚本）。这套 derive+codegen 让"加一个事件"变成在 Rust 写一个 handler，Dart 侧自动获得强类型 API。

### 5.3 文档数据模型与 CRDT 协作

文档本体是外部 collab_document crate 的 `Block` 树（frontend/rust-lib/flowy-document/src/document_data.rs 中 `From<DocumentData> for DocumentDataPB` 做 Rust→Dart 结构转换）：blocks 是 `HashMap<String, Block>`，每个 block 有 parent/children（经 meta.children_map）与富文本数据（meta.text_map）。所有编辑以 collab CRDT 事务提交（collab-integrate crate 包装 collab-plugins 做持久化与快照），事件层提供 ApplyAction、Undo/Redo、Awareness（协作光标在线状态）等接口——这是多人实时编辑与离线一致性的基础。

### 5.4 Flutter 插件体系与双服务端

Flutter 侧按"视图类型即插件"组织（frontend/appflowy_flutter/lib/startup/plugin/plugin.dart）：`PluginType` 枚举定义 document/blank/trash/grid/board/calendar/databaseDocument/chat 八类，每个插件实现 `Plugin`（widgetBuilder + notifier 生命周期）+ `PluginBuilder`，各实现在 lib/plugins/ 下（document.dart、grid.dart、board.dart 等）。服务端抽象在 flowy-server/src/：`af_cloud/`（AppFlowy Cloud 客户端）与 `local_server/`（纯本地模式）实现同一接口，AuthenticatorType 切换，这使"本地优先、可选上云"成为架构内建能力而非补丁。

## 6. 构建与运行

- 全量任务编排：`cargo make`（frontend/Makefile.toml，扩展 desktop/mobile/protobuf/tests 等 10 个子 makefile）
- Rust FFI 库：`cargo make --profile production-desktop`（Makefile.toml 中 CRATE_TYPE/cdylib 配置）
- Flutter 运行：`flutter run`（frontend/appflowy_flutter/，需先完成 codegen：frontend/scripts/code_generation/generate.sh）
- Rust 集成测试：`cargo test`（frontend/rust-lib/event-integration-test 无头驱动事件系统）；Flutter 测试：`flutter test`；集成测试在 integration_test/（规范要求静态阅读，未实际运行，以上命令来自 Makefile.toml 与 README 指向的官方文档）

## 7. 测试与工程质量

- **Rust**：专属集成测试 crate event-integration-test（61 个文件）直接在无 UI 环境调用事件分发层，覆盖用户/文档/数据库/文件夹全链路；另有 21 个含 `#[cfg(test)]` 的文件做 crate 内单测（find 统计）
- **Dart**：test/ 94 个文件 + integration_test/ 176 个文件，共 270 个测试文件（find 统计）；appflowy_backend 包自带 example/integration_test 验证 FFI 通路
- **CI**：13 个工作流——flutter_ci.yaml、rust_ci.yaml、ios_ci.yaml、mobile_ci.yml、docker_ci.yml、release.yml、commit_lint.yml（配合 commitlint.config.js）、ninja_i18n.yml 等（.github/workflows/ 目录）
- 工程约束：analysis_options.yaml（Dart lint）、rustfmt.toml、dart_dependency_validator.yaml、covtest.rs（覆盖率入口）

## 8. 社区与维护状态

- 高度活跃：最近 push 2026-10-28（gh api），最新 commit 为 README 图片更新（#9101）；release 节奏快——2026 年 10 月内发布 0.14.6/0.14.7/0.14.8/0.14.10/0.14.11 五个版本（gh api releases）
- 362 位贡献者（gh api 分页求和），open issues 1023，fork 6132
- 版本号不一致现象：主分支 pubspec.yaml 与 CHANGELOG.md 停在 0.11.4，而 GitHub release 已到 0.14.11（target main），推测版本号由 CI 在发布时注入、仓库内 CHANGELOG 未同步，机制未确认
- 商业公司（AppFlowy-IO）主导 + 社区贡献双轨：Discord/Reddit/Forum 全套社区入口（README.md），另有关联仓库 appflowy-editor、appflowy-board、collab 等以独立包发布

## 9. 亮点与风险

**亮点**
1. 架构分层干净：Rust 核心与 Flutter UI 完全解耦，事件+codegen 使跨语言调用类型安全，核心逻辑可复用于未来 Web/Tauri 端（frontend/Makefile.toml 已含 web/tauri 任务）
2. 本地优先 + 可自托管：flowy-server 的 local_server/af_cloud 双实现是架构级设计，AGPL-3.0 保障开源底线
3. 测试体系完整：无头集成测试 crate 直接驱动事件总线，配合 270 个 Dart 测试文件与 13 条 CI 流水线
4. CRDT 内核（yrs/collab-rs）让实时协作与离线编辑成为底层能力，而非应用层补丁
5. 代码生成贯穿始终：事件、protobuf、图标、多语言文件均自动生成（frontend/scripts/code_generation/）

**风险/局限**
1. 仓库体积与复杂度高：双语言 + 31 crate + 7 个本地 Dart 包 + 构建期代码生成，新人上手成本大，构建链长（需 cargo-make + codegen 两步）
2. 大量核心能力在外部依赖（collab-rs、appflowy_editor 等独立仓库），本仓库分析无法完整覆盖协作内核实现
3. 主分支版本号与 release 版本脱节，仓库内 CHANGELOG 落后于实际发布，版本追溯需依赖 GitHub releases
4. AGPL-3.0 对想二次分发/商用集成的团队有传染性约束；AI 能力依赖其云服务（flowy-ai 本地部分能力边界未确认）

## 10. 总结评价

AppFlowy 是"开源重客户端 + 双语言核心"路线中完成度最高的一批项目：Rust 承载全部业务与 CRDT 协作内核，Flutter 只做表现层，中间用 derive 宏 + 代码生成搭起类型安全的 FFI 事件总线，这套架构本身就是学习跨语言桌面应用工程化的优秀范本。它适合三类人：想要数据主权、自托管 Notion 替代品的用户与团队；想研究 Flutter+Rust FFI、事件驱动架构、CRDT 协作落地的工程师（重点读 lib-dispatch、dart-ffi、flowy-document 与 build-tool/flowy-codegen）；以及寻找成熟 AGPL 代码库二次开发的产品团队。阅读源码的推荐路径是先看 dart-ffi/src/lib.rs 理解桥，再挑 flowy-document 看事件与数据模型的全貌。
