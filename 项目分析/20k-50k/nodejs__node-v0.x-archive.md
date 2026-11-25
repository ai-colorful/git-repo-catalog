# nodejs/node-v0.x-archive 项目分析

> 星数 12,000（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：Node.js v0.x 时代（2009-2015，joyent/node 主线）的官方只读历史归档仓——现代 Node.js 的"出生证明"，开发已整体迁移至 `nodejs/node`。

| 项目 | 内容 |
|---|---|
| 名称 | nodejs/node-v0.x-archive |
| 星数 | 队列快照 12,000（2026-10-24）；gh api 复核（2026-11-21）实际为 34,258，归档仓持续吸星，以复核值为准 |
| 主语言 | JavaScript（C++ 为运行时核心，gh api language 字段为 null，因默认分支是占位分支） |
| 许可证 | MIT 风格（Joyent, Inc. and other Node contributors，见 `LICENSE` 首行） |
| archived | **true**（gh api 已核实，2026-11-21） |
| 默认分支 | `moved`（占位分支，description 为 "Moved to https://github.com/nodejs/node"） |
| 仓库创建时间 | 2009-05-27（即原 joyent/node 仓库本体，随仓库转移保留创建时间，gh api 核实） |
| 最后实质提交 | master 分支 2015-08-31；v0.12 维护分支 2015-09-30（gh api 核实） |
| 最后 push | 2024-08-15（元数据层面，gh api 核实） |

分析说明：默认分支 `moved` 仅含 4 个文件的跳转说明（README + `.github/` 两个模板），真实 v0.x 代码全部在历史分支中。本文以 `master` 分支 tarball（v0.13.0 开发态，冻结于 2015-08-31）为代码快照做分析，分支列表用 gh api 核实。

## 2. 项目是什么，解决什么问题

这个仓库解决的不是技术问题，而是**历史保全问题**：它是 Node.js 在 2015 年 8 月"迁移日"之前的完整源码快照。`README.md`（moved 分支）写明："This repository is an archive of Node.js before the move to nodejs/node"，并明确"New issues and pull requests opened here will automatically be rejected"（新 issue/PR 会被自动拒绝）——它是设计上的只读墓碑，不是活跃项目。

它的存在背景是 Node.js 历史上最著名的一次分裂与和解：2014 年底社区因 Joyent 治理不满分叉出 io.js，2015 年初 Node.js Foundation 成立，同年各方达成"收敛协议"（Convergence），io.js 与 joyent/node 的开发合流到新建的 `nodejs/node` 仓库。合流后，joyent/node 的主线开发停摆：master 分支最后一笔提交是 2015-08-31 的 "Update README to reflect move to nodejs/node"（引用 joyent/node#25876，PR nodejs/node-v0.x-archive#25897），v0.12 维护分支苟延至 2015-09-30 最后一笔 "domains: port caeb677 from v0.10 to v0.12"，随后整个仓库转入 nodejs 组织、更名为 node-v0.x-archive，默认分支换成只写跳转说明的 `moved` 分支。这些时间点均经 gh api 核实。

仓库本体保存了 Node.js 从 2009 年 Ry Dahl 的原型到 v0.12/v0.13 时代的全部演进轨迹：分支列表（共 143 个，gh api 分页核实）里有 `v0.2`、`v0.4`、`v0.6`、`v0.8` 主线分支和从 `v0.8.7-release` 到 `v0.12.x-release` 的逐版本发布分支，还有 `npm-v1.4.x`、`libuv-upgrade-latest-1x`、`orangemocha-*` 等功能分支——一个早期开源项目工作流的完整化石层。Issues/PR 记录也被完整冻结保留：closed issue 6,368 个、closed PR 3,700 个（Search API 核实），是研究早期 Node.js 决策过程（如 v0.10 → v0.12 的 streams2 改造）的一手档案。

对今天的读者，它的价值是三点：第一，作为 `nodejs/node` 现代代码的对照组——现代 Node 的 `lib/` 依然是这个目录结构的直系后裔；第二，理解 Node.js 设计哲学（事件循环 + 非阻塞 IO + CommonJS 模块）在最朴素形态下的实现；第三，gh api 元数据里 `created_at: 2009-05-27` 这个字段本身就是 Node.js 项目年龄的原始凭证。

## 3. 技术栈与依赖

master 分支（v0.13.0-pre，`src/node_version.h`）快照下的技术栈：

| 组件 | 版本 | 出处 |
|---|---|---|
| JavaScript 引擎 | V8 3.28.71.19 | `deps/v8/src/version.cc` L35-38 |
| 事件循环/IO | libuv 1.6.1 | `deps/uv/include/uv-version.h` |
| TLS | OpenSSL 1.0.1p（fips 变体一并捆绑） | `deps/openssl/openssl/include/openssl/opensslv.h` |
| HTTP 解析 | http_parser 2.3.0 | `deps/http_parser/http_parser.h` L28-30 |
| 包管理器 | npm 2.11.3（随 Node 捆绑分发） | `deps/npm/package.json` |
| DNS 解析 | c-ares | `deps/cares/` |
| 压缩 | zlib | `deps/zlib/` |
| 构建系统 | GYP（`tools/gyp/`）+ Python | `configure`、`node.gyp` |
| 测试运行器 | 自研 Python 工具（源自 V8 test runner） | `tools/test.py`（文件头标注 "Copyright 2008 the V8 project authors"） |

值得注意的历史断代特征：libuv 已是 1.x（0.12 时代刚完成 0.x → 1.x 升级），而 V8 还是 3.28 系列（现代 Node 早已用 7.x/8.x+）；构建时代特征鲜明——Unix 用 `./configure && make`，Windows 用 `vcbuild.bat`（`README-pre-convergence.md`，要求 GCC 4.2+/Python 2.6-2.7/VS2010-2012），没有 CMake、没有 npm-based 构建链。`node.gyp` 开头的开关列表（`v8_use_snapshot`、`node_use_dtrace`、`node_use_etw` 等）记录了当时对快照、各平台追踪器的编译期配置方式。

## 4. 目录结构与代码组织

顶层目录（master 分支快照）：

| 目录/文件 | 职责 |
|---|---|
| `lib/` | JavaScript 标准库（50 个 .js，共 25,308 行）——用户 `require()` 到的一切 |
| `src/` | C++ 运行时核心（约 25,063 行 .cc/.h）——事件循环接入、句柄包装、V8 绑定 |
| `deps/` | 捆绑的第三方库：v8、uv、openssl、http_parser、cares、zlib、npm、mdb_v8、debugger-agent |
| `doc/` | API 文档（`doc/api/*.markdown` 旧式 markdown + `doc/node.1` man 页） |
| `test/` | 测试套件（1,000 个 .js，按 simple/pummel/internet/gc/addons 等分类） |
| `benchmark/` | 性能基准（http/fs/crypto/buffers 等） |
| `tools/` | 构建/发布工具链（gyp、js2c.py、test.py、update-authors.js、changelog 脚本、msvs/pkgsrc 打包） |
| `Makefile` / `BSDmakefile` / `vcbuild.bat` | Unix / BSD / Windows 三套构建入口 |
| `configure` / `node.gyp` / `common.gypi` | GYP 配置生成链 |
| `AUTHORS` / `ChangeLog` / `LICENSE` / `CONTRIBUTING.md` | 社区档案 |

组织方式是典型的"**单仓单体 + 预捆绑依赖**"：所有第三方库以源码形式 vendored 进 `deps/`，构建期 `js2c.py`（`tools/js2c.py`）把 `lib/*.js` 编译进二进制。这正是现代 Node 仓库结构的雏形——`nodejs/node` 至今沿用 lib/src/deps/test/tools 的五段式布局，本仓可视为该布局的"出生版本"。

JS 侧按"内部前缀 + 公开面"组织：`_http_agent.js`、`_http_client.js`、`_stream_readable.js` 等下划线前缀模块是内部实现，`http.js`、`stream.js` 只是薄薄一层 re-export（如 `lib/http.js`）。这种"公开壳 + 私有实现"的拆分是 v0.11 时代 streams2/HTTP 重构的产物，与现代 Node 完全一致。

## 5. 核心架构与关键实现

### 5.1 CommonJS 模块系统（lib/module.js，543 行）

Node 的灵魂部件之一，当时已是今天的样子：模块缓存、路径解析、扩展名分发都在这里。构造函数极简（`lib/module.js` L40-53）：

```js
function Module(id, parent) {
  this.parent = parent;
  if (parent && parent.children) {
    parent.children.push(this);
  }
  this.filename = null;
  this.loaded = false;
  this.children = [];
}
Module._cache = {};
Module._pathCache = {};
Module._extensions = {};
```

`Module._cache` / `Module._pathCache` / `Module._extensions` 三张表就是现代 `require` 缓存与解析机制的原型，`Module._contextLoad`（L55，`NODE_MODULE_CONTEXTS` 环境变量）记录了当时对 vm 上下文隔离加载的实验。文件顶部还留着 joyent/node#1707 的防御性注释——早期 issue 编号直接嵌在生产代码里，是这个仓的年代质感。

### 5.2 HTTP 双层实现（lib/_http_client.js 580 行 / lib/_http_server.js 498 行）

`_http_client.js` 的 `ClientRequest(options, cb)`（L44）与 `ClientRequest.prototype.onSocket`（L498）展示了 v0.12 时代 HTTP 客户端的核心契约：请求与 socket 的解绑/重绑（agent 池化复用）。HTTP 栈按协议职责切成六个私有模块（`_http_agent`、`_http_client`、`_http_common`、`_http_incoming`、`_http_outgoing`、`_http_server`），解析交给 C++ 侧 `src/node_http_parser.cc` 包装的 http_parser 2.3.0——"C++ 做字节、JS 做语义"的分层自 v0.x 起就是 Node 的架构基调，至今未变。

### 5.3 C++ 句柄包装层（src/*_wrap.cc）

`src/` 的骨架是"每个 libuv/OS 能力一个 wrap 文件"：`tcp_wrap.cc`、`udp_wrap.cc`、`stream_wrap.cc`、`timer_wrap.cc`、`fs_event_wrap.cc`、`cares_wrap.cc`、`tty_wrap.cc`——把 libuv 句柄包成带回调的 V8 对象。JS 侧 `lib/net.js`（1,482 行）、`lib/fs.js`（1,888 行）再在这些 wrap 上织出公开 API。`src/node.cc` 是主入口，负责启动事件循环并注册内置模块；`src/node_javascript.cc` + `tools/js2c.py` 完成"lib/ 全部 JS 预编译进二进制"的打包。另有两个明显的时代印记模块：`lib/smalloc.js`（已被 Buffer/ArrayBuffer 取代的显式内存工具）与 `lib/freelist.js`、`lib/_linklist.js`（手写的内部数据结构，现代实现已移除）。

## 6. 构建与运行

全部命令来自 `README-pre-convergence.md` 与 `Makefile`（时代产物，今日复现需 2015 年前后的工具链，仅作档案记录）：

```sh
# Unix（需 GCC 4.2+、Python 2.6/2.7、GNU Make 3.81+）
./configure
make
make install
make test            # Makefile L89
make doc             # 生成 API 文档

# Windows（需 Python 2.6/2.7 + VS2010/2012）
vcbuild nosign       # 构建
vcbuild test         # 测试

# Intl 支持（README-pre-convergence.md）
./configure --with-intl=small-icu --download=all
```

`Makefile` 里的目标清单本身就是当时的测试分层档案：`test-http1`（L94）、`test-valgrind`（L97）、`test-gc`（L116）、`test-all`（L121）、`test-ci`（L131）、`test-release`（L134）、`test-pummel`（L146）、`test-internet`（L149）、`test-debugger`（L152）。

## 7. 测试与工程质量

- 规模：`test/` 共 1,000 个 .js 文件，分目录 `simple/`（快速单元级）、`pummel/`（压测类）、`internet/`（需联网）、`message/`、`gc/`、`addons/`（N-API 前身的原生 addon 测试）、`debugger/`、`timers/`、`disabled/`（明确废弃用例）——基于 `tar -tzf` 统计。
- 框架：无现代测试框架，用例即"断言 + 进程退出码"的脚本风格，由 `tools/test.py`（Python，文件头标注源自 V8 项目 2008 年代码）调度。
- CI：master 分支无 `.github/` 目录（仅 moved 占位分支有 issue/PR 模板），当时的 CI 在 joyent 内部 Jenkins 与 Travis，仓库内无配置可考——标注"未确认"。
- 质量观察：v0.12 时代刚完成 streams2 大改造（`_stream_readable.js` 914 行、`_stream_writable.js` 503 行），lib/ 与 src/ 的行数几乎 1:1（各约 2.5 万行），JS 侧注释密度较高且常带 issue 编号溯源，工程纪律靠 `tools/cpplint.py`（Google C++ lint）维持。

## 8. 社区与维护状态

本仓按任务要求重点记录历史定位而非当前维护状态，以下均为核实过的客观事实：

- **归档状态**：`archived: true`、`disabled: false`（gh api，2026-11-21）。GitHub 归档 = 只读，连作者本人也无法提交。
- **Issue/PR 冻结**：open issue 0、open PR 0；closed issue 6,368、closed PR 3,700（Search API 核实）。`README.md` 声明新 issue/PR 自动拒绝，`.github/ISSUE_TEMPLATE.md` 将求助者指向 nodejs/help 与 nodejs/node。
- **贡献者**：GitHub contributors API 对默认分支 `moved` 只返回 2 人（占位分支无历史）；真实历史贡献者以 `AUTHORS` 文件为准——**681 行，即 681 位署名贡献者**（Joyent 时代官方名册），另可佐证的是 branches 里的逐版本 release 分支均由维护者逐一打点。
- **开发迁移时间线**（客观记录）：
  - 2009-05-27 仓库创建（gh api created_at，即 joyent/node 起点）；
  - 2014 年底 io.js 分叉、2015-02 Node.js Foundation 成立（公认公开历史，仓内无直接文件可证，标注"公开历史"）；
  - 2015-08-31 master 最后提交："Update README to reflect move to nodejs/node"（Ref joyent/node#25876，PR-URL nodejs/node-v0.x-archive#25897，gh api 核实）；
  - 2015-09-30 `v0.12` 分支最后维护提交（gh api 核实）；
  - 此后仓库转入 nodejs 组织改名归档，2024-08-15 最后一次 push（gh api pushed_at，应为分支清理类管理操作，未确认具体内容）。
- **Release 节奏**：`ChangeLog` 顶部记录 2015.07.09 v0.12.7 (Stable)、2015.07.03 v0.12.6、2015.06.22 v0.12.5——月度级稳定版节奏，openssl/npm/libuv 的升级条目占大头，是 v0.12 尾期"仅维护"状态的直接证据。
- **星数悖论**：一个停止开发的归档仓反而从快照 12,000 涨到 34,258（gh api 复核）——"star 数 ≠ 活跃度"的典型样本，涨星来自 Node.js 生态的历史情怀与考古需求。

## 9. 亮点与风险

**亮点**：
1. 完整的官方历史档案：源码、143 个分支、10,000+ 已关闭 issue/PR 全部原样保留（gh api/源码核实），是 Node.js 早期架构演进的唯一官方一手资料。
2. `lib/` 五十模块清单即"Node 设计哲学最小集"：events/net/fs/http/streams/module 全部在 2.5 万行 JS 内完成，适合想读懂最小完整运行时的读者（`lib/` 行数统计）。
3. 架构连续性的活教材：今天的 `nodejs/node` 仍是 lib/src/deps/test/tools 同构布局，对照阅读即可定位任何现代组件的"祖先"（本文目录表）。
4. 依赖治理的时代标本：`deps/` 里 V8 3.28.71.19、libuv 1.6.1、OpenSSL 1.0.1p、npm 2.11.3 的版本组合，精确记录了 2015 年中 Node 生态的依赖地层（各版本文件见第 3 章表）。
5. 分支即日志：`v0.8.7-release` 到 `v0.12.x-release` 的逐版本分支是当时"一版一分支"发布流的完整化石（gh api 分支列表）。

**风险/局限**：
1. 明确不安全：OpenSSL 1.0.1p、V8 3.28 均存在大量已知 CVE 且永不再修（`deps/openssl/openssl/include/openssl/opensslv.h`），任何用途都禁止用于生产。
2. 已无法构建复现：依赖 Python 2.6/2.7、GCC 4.2、VS2010 等已退出当前生态的工具链（`README-pre-convergence.md`），现代机器复刻构建需专门环境。
3. 默认分支是占位分支，GitHub 网页首屏只见跳转说明；分析真实代码必须显式切到 `master` 或 `v0.x` 分支，容易被误判为"空仓"。
4. 文档为旧式 markdown 且指向已失效的 joyent wiki 链接（`README-pre-convergence.md`），与现行 Node 文档体系不兼容。

## 10. 总结评价

nodejs/node-v0.x-archive 是一个"设计如此"的死仓库，但它的死正是它的价值：joyent/node 十年（2009-2015）的全部源码、分支、issue 讨论被原封冻结，成为现代 Node.js 可溯源的起点。对工程研究者，它是理解事件驱动运行时最小实现的经典样本——两个 2.5 万行的 lib/ 与 src/ 加起来就是完整的 Node；对生态考古者，它的 ChangeLog、AUTHORS 名册与逐版本分支是 Node.js Foundation 收敛前最后岁月的一手档案；对普通开发者，它没有使用价值（不可构建、不可部署），但作为 `nodejs/node` 的对照组和 Node 历史的权威出处，值得收藏。star 从 12,000 涨到 34,258 说明：冻结的代码也能持续生长意义。
