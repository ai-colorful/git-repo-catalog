# wg/wrk 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 C｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：wrk 是经典的高性能 HTTP 压测工具，用多线程 + epoll/kqueue 事件驱动在单台多核机器上打出极高压测负载，并可通过内嵌 LuaJIT 脚本定制请求生成、响应处理与结果报告。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | wg/wrk | GitHub |
| 星数 | 39500（快照） | 派发参数 |
| 语言 | C（约 6558 行 src/*.c/h，其中 2408 行为移植的 http_parser） | `src/` 目录 wc -l |
| 许可证 | Modified Apache 2.0（自定义修订版，GitHub 标记 NOASSERTION） | `LICENSE` 首行 |
| 最近代码提交 | 2021-02-07（a211dd5，即 4.2.0 tag） | gh api commits |
| 最新 tag | 4.2.0（无 GitHub Release，仅 tag） | gh api tags/releases |
| 主要平台 | Linux/macOS/BSD/Solaris（无 Windows 原生支持） | `src/config.h` |
| 默认分支 | master（与 next 同指 a211dd5） | gh api branches |

## 2. 项目是什么，解决什么问题

wrk 解决的核心问题是：如何用一台普通机器对 HTTP 服务产生足够大的负载并测出可信的延迟分布。传统工具 ab 是单线程模型，连接数上千后工具自身成为瓶颈；wrk 用"N 线程 × 每线程独立事件循环 × 每连接状态机"的架构，让数千个 keep-alive 连接的开销被 epoll/kqueue 摊薄，README 中的示例输出为单机 12 线程 400 连接、30 秒打出约 74.9 万 QPS、2246 万请求（`README.md` Basic Usage）。

第二个差异化能力是可编程性。wrk 内嵌 LuaJIT（`deps/LuaJIT-2.1.zip`，构建时静态编译链接，`Makefile:75-90`），用户脚本可以定义 `setup/init/delay/request/response/done` 六个钩子，覆盖压测全生命周期：多线程注入参数（setup + thread:set）、动态请求（request）、按毫秒延迟发压（delay）、响应内容校验（response）、自定义报告（done，如 `scripts/report.lua` 输出 CSV 分位数）。`SCRIPTING` 文件完整定义了这份 API。

第三是统计质量。自 4.0.0 起 wrk 对**每个**请求记录延迟（CHANGES: "Record latency for every request instead of random samples"），并引入 coordinated omission 事后修正（CHANGES: "Improve correction of coordinated omission"）——这是 wrk 与 Gil Tene 提出的 Coordinated Omission 问题对话的直接证据，也使其与 wrk2 分支的争论成为压测领域最著名的讨论之一。

适用场景：服务端性能基线测试、容量评估、回归压测。不适用：需要精确恒定速率注入的开环测试（这是 wrk2 的定位）、需要分布式施压（wrk 是单机工具）。

## 3. 技术栈与依赖

| 组件 | 用途 | 出处 |
|---|---|---|
| C99 + pthread | 核心实现，`-std=c99 -Wall -O2 -D_REENTRANT` | `Makefile:1` |
| ae 事件库（自 Redis 移植） | 每线程一个事件循环，4 种后端：epoll/kqueue/evport/select | `src/ae.c`、`src/ae.h` 头注（Salvatore Sanfilippo, 2006-2012）、`src/config.h` |
| http-parser（nginx/joyent/node.js 移植） | 增量解析 HTTP 响应 | `src/http_parser.c/h`，致谢见 `README.md` Acknowledgements |
| LuaJIT 2.1（内嵌 deps，v2.1 branch ec6edc5） | 脚本扩展 | `deps/LuaJIT-2.1.zip`、`Makefile:75`、提交 2221a30 |
| OpenSSL 1.1.1i（内嵌 deps，可用 WITH_OPENSSL 换系统库） | HTTPS 支持 | `deps/openssl-1.1.1i.tar.gz`、提交 a211dd5 |
| zmalloc + atomicvar（自 Redis 移植） | 分配器封装与原子计数 | `src/zmalloc.c`、`src/atomicvar.h` 头注 |

构建产物为单个静态二进制 `wrk`，无运行时依赖（LuaJIT/OpenSSL 均以 `-l` 链接为静态库，`Makefile:37-46`）。wrk.lua 被预编译为字节码后链接进二进制（`Makefile:62-64` 用 `luajit -b` 生成 bytecode.o）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `src/wrk.c` (585 行) | main()、参数解析、多线程编排、连接状态机回调、结果输出 |
| `src/script.c` (576 行) | LuaJIT 桥接：线程级 lua_State、六钩子调用、stats/thread 的 userdata 绑定 |
| `src/ae.c` + ae_epoll/kqueue/evport/select.c (约 1290 行) | 事件循环及四个平台后端 |
| `src/stats.c` (107 行) | 扁平数组精确直方图 + coordinated omission 修正 |
| `src/ssl.c` (76 行) | OpenSSL 封装，TLS SNI |
| `src/net.c` (39 行) | 明文 socket 读写原语 |
| `src/http_parser.c` (2408 行) | 移植的 HTTP 响应解析器 |
| `src/wrk.lua` (74 行) | 运行时 Lua 基础库：wrk 表、wrk.format、默认 request |
| `src/zmalloc.c`、`atomicvar.h`、`units.c`、`aprintf.c` | Redis 移植的分配器/原子变量、单位换算、格式化 |
| `scripts/`（9 个 .lua） | 官方示例：post/pipeline/addr/auth/counter/delay/report/setup/stop |
| `deps/` | LuaJIT-2.1.zip、openssl-1.1.1i.tar.gz 两个内嵌依赖包 |
| `.github/workflows/build.yml` | 唯一 CI：push 时在 ubuntu/macos 上 `make` |

组织评述：极致的单体小工程。全部自有逻辑约 1600 行（wrk.c + script.c + stats.c + ssl.c + net.c + units.c），其余为成熟的移植件（ae/http_parser/zmalloc/LuaJIT）。依赖管理不走子模块而是把依赖源码包直接放进 `deps/`，构建时解包静态编译——牺牲包管理器便利换取"一条 make 命令出二进制"的确定性。没有测试目录、没有 configure，平台差异全部收敛在 Makefile 顶部的条件分支和 `config.h` 的宏选择中。

## 5. 核心架构与关键实现

### 5.1 多线程反应器模型（wrk.c）

主线程只做四件事：解析参数与 TLS 初始化、主 lua_State 中做 DNS 解析（`script_resolve`，wrk.c:96）、按线程数派生 pthread、`sleep(duration)` 后汇总（wrk.c:61-200）。真正施压的是工作线程，每个线程：

```c
// src/wrk.c thread_main (202-234)
t->loop = aeCreateEventLoop(10 + cfg.connections * 3);  // 主线程为每线程建独立事件循环
t->connections = cfg.connections / cfg.threads;
...
aeCreateTimeEvent(loop, RECORD_INTERVAL_MS, record_rate, thread, NULL); // 100ms 定时器
thread->start = time_us();
aeMain(loop);                                            // 阻塞在此直到 aeStop
```

关键点：线程之间零共享锁。每个线程独占一个 aeEventLoop、一组 connection（`src/wrk.h:41-64`）、一个独立 lua_State（wrk.c:109，`script_create` 每线程调用一次）。唯一的跨线程同步是工作线程向主进程的两块统计直方图做无锁原子写入（`stats_record` 用 `__sync_fetch_and_add`/CAS，`src/stats.c:24-29`）。停止机制同样是弱同步：主线程置 `volatile stop=1`，工作线程的 100ms 定时器回调 `record_rate` 轮询到后调 `aeStop`（wrk.c:286-287），优雅停机延迟上限即 100ms。

连接生命周期是一条纯事件驱动状态机：`connect_socket`（非阻塞 connect + TCP_NODELAY，wrk.c:236-264）→ `socket_connected`（注册 AE_READABLE|AE_WRITABLE，wrk.c:362-382）→ `socket_writeable` 写请求（动态模式下每次先调 `script_request` 生成，wrk.c:395-401）→ `socket_readable` 循环读取并喂给 http_parser（wrk.c:426-448）→ `response_complete` 回调：计数、记录延迟、pending 流水线未清零则继续等、否则重新挂 AE_WRITABLE 发下一轮（wrk.c:324-360）。断连/解析错误统一走 `reconnect_socket` 重连并继续，保证压测不断流。

TLS 通过函数指针表实现透明替换：明文默认绑定 `sock_connect/read/write/...`，检测到 https 后整体换成 ssl_* 版本（wrk.c:26-32、81-85），`src/ssl.c` 各函数以 RETRY/ERROR/OK 三态封装 OpenSSL 的 WANT_READ/WANT_WRITE 语义，与事件循环无缝衔接。

### 5.2 LuaJIT 脚本扩展机制（script.c + wrk.lua）

六个钩子的触发链：`setup(thread)` 在主线程对每个线程调用（可 `thread:set/get/stop` 与改 `thread.addr`，`src/script.c:392-437`），值经 `script_copy_value` 限制为 boolean/nil/number/string/纯表跨环境拷贝（script.c:477-505）；`init/delay/request/response` 在工作线程的事件循环内被 C 侧按需调用（script.c:144-183）；`done(summary, latency, requests)` 在结束后于主线程调用，stats 以 userdata + 元表暴露，`latency:percentile(99)` 直接调回 C 的 `stats_percentile`（script.c:351-384）。

两个值得注意的性能设计：

- **静态请求短路**：脚本未定义全局 `request` 函数时 `script_is_static` 为真（script.c:192-194），wrk 在线程启动前生成一次请求字符串，之后每个连接直接复用指针（wrk.c:208-210、218-219），LuaJIT 零参与热路径；README "Benchmarking Tips" 明确说明这种情况无性能损耗。
- **响应解析按需开启**：只有定义了 `response()` 才注册 header_field/header_value/response_body 回调（wrk.c:112-120），否则解析器只触发 `on_message_complete`，头与体完全不缓冲。请求合法性在启动前用 http_parser 预检，顺带得到流水线深度（`script_verify_request` 数 `on_message_complete` 次数，script.c:268-301；`c->pending = cfg.pipeline`，wrk.c:400），`scripts/pipeline.lua` 即靠"一个字符串里拼多个请求"实现 HTTP pipelining。

### 5.3 延迟统计与 coordinated omission 修正（stats.c）

wrk 不用 HDR Histogram，而是一块**以微秒值为下标的扁平 uint64 数组**：`stats_alloc(cfg.timeout * 1000)`（wrk.c:91），默认上限 2,000,000 μs（`SOCKET_TIMEOUT_MS 2000`，wrk.h:22），即约 16MB、1μs 分辨率的精确直方图；超过上限的响应 `stats_record` 返回 0，在 `response_complete` 中被记为 timeout（wrk.c:344-345）。min/max 用 CAS 维护，mean/stdev/percentile 事后单线程扫描数组计算（stats.c:45-88）。这是"内存换精确"的取舍：同一套结构也用于 Req/Sec 统计，上限 `MAX_THREAD_RATE_S = 10000000`（wrk.h:21），靠 calloc 惰性映射避免实际占用满额内存。

Coordinated omission 修正全部在 `stats_correct`（stats.c:33-43），跑在所有线程 join 之后：

```c
// src/stats.c stats_correct
void stats_correct(stats *stats, int64_t expected) {
    for (uint64_t n = expected * 2; n <= stats->max; n++) {
        uint64_t count = stats->data[n];
        int64_t m = (int64_t) n - expected;
        while (count && m > expected) {   // 把间隔过长的样本折算出"漏发"的虚拟样本
            stats->data[m] += count;
            stats->count += count;
            m -= expected;
        }
    }
}
```

其中 `expected = runtime_us / (complete / connections)` 是每连接平均期望请求间隔（wrk.c:168-171）。逻辑：若某次响应耗时 n 超过平均间隔两倍，则按间隔步长向 n-expected、n-2*expected……补记虚拟样本，模拟"请求若按时发出本应在更早时刻收到响应"的采样点。这是**基于平均间隔的事后修正**，只修正长尾、不修正中位数附近的失真，修正粒度受制于期望间隔的估计质量——这正是与 wrk2 分歧的技术根源。

### 5.4 与 ab/hey/vegeta/wrk2 的对比及 wrk2 争议

- **ab**（ApacheBench）：单线程模型，高连接数下工具自身先到瓶颈；无脚本能力。wrk 的多线程反应器正是对它的直接超越。
- **hey**（Go）：单进程、安装简单，适合快速冒烟；无脚本扩展，极限施压能力弱于 wrk。
- **vegeta**（Go）：以"恒定攻击速率"为设计核心，天然规避部分 coordinated omission，输出直方图/JSON，适合作为库集成；但单进程吞吐上限低于 wrk 的 C 实现。
- **wrk2**（ginatron/wrk2，Brent Welch 基于 wrk fork）：核心改动是新增 `-R` 参数以恒定 QPS 调度发送，从机制上消除系统性 coordinated omission，可测出稳定的 p99.99。wrk 侧的回应体现在 4.0.0 的两项变更：移除校准阶段、固定间隔记录速率 + `stats_correct` 事后修正（CHANGES: "Remove calibration phase and record rate at fixed interval"、"Improve correction of coordinated omission"）。
- **争议的客观陈述**：双方分歧在于压测哲学——wrk 一派认为开环饱和施压才能测出服务器真实容量上限，事后修正足以弥补采样偏差；wrk2 一派（受 Gil Tene 的 Coordinated Omission 论述影响）认为饱和施压下"响应变慢 → 下一请求被推迟 → 慢响应被系统性漏采"的失真无法靠事后平均修正挽回，必须恒定速率注入。双方在社区多年争论（相关 issue 讨论串存在，具体编号未能通过 API 核实，标注未确认），wrk 上游始终未采纳恒定速率模式，两个工具因此长期并存：要极限吞吐与脚本灵活性选 wrk，要稳定精确的高分位延迟选 wrk2。

## 6. 构建与运行

构建（`Makefile`、`INSTALL`）：

```bash
make                                  # 默认：解包 deps/ 内的 LuaJIT+OpenSSL，静态编译出 wrk
make WITH_LUAJIT=/usr WITH_OPENSSL=/usr   # 使用系统库（打发行包时推荐）
make clean
```

依赖仅 GNU make + C 编译器（LuaJIT/OpenSSL 自动从 deps/ 解包构建）；macOS/BSD 用 gmake。CI 即 `make`（`.github/workflows/build.yml:17`）。

运行（`README.md`）：

```bash
wrk -t12 -c400 -d30s --latency --timeout 2s http://127.0.0.1:8080/index.html
wrk -t2 -c100 -d10s -s scripts/post.lua http://127.0.0.1:8080/     # POST 示例
wrk -t2 -c100 -d10s -s scripts/addr.lua http://127.0.0.1:8080/     # 多地址
wrk -t4 -c200 -d10s -s scripts/setup.lua http://127.0.0.1:8080/ -- 1 2 3   # 传参给 init()
```

选项：`-t` 线程数（默认 2）、`-c` 连接数（默认 10，必须 ≥ 线程数，wrk.c:535-538）、`-d` 时长（秒级精度）、`-s` 脚本、`-H` 头、`--latency` 打印 50/75/90/99 分位（wrk.c:575-585）、`--timeout` 延迟记录上限。数值支持 SI 单位（1k/1M），时长支持 s/m/h（`src/units.c`）。

## 7. 测试与工程质量

- **无自动化测试**：仓库没有任何测试目录、测试框架或 fuzz 目标（tarball 全量 54 个条目中仅 `.github/workflows/build.yml` 一个 CI 文件）。CI（macos+ubuntu 双平台 `make`）只验证可构建，属"编译即测试"的最低保障。
- **代码质量**：核心自有代码风格紧凑一致（C99、`-Wall` 干净）、模块边界清晰（net/ssl/stats/script/ae 各司其职）；错误路径统一走 reconnect/计数而非退出，长期运行健壮性靠架构而非测试保证。
- **文档质量**：README/SCRIPTING/INSTALL/CHANGES/NOTICE 五份短文档覆盖用法、脚本 API、构建、变更与致谢，精准无冗水分；但无 man page，`--timeout` 与 `SOCKET_TIMEOUT_MS` 命名（值实为 2000 秒语义下再乘 1000 取 μs 上限）等细节只能读源码确认。
- **已知小瑕疵**：连接数按整除分配到线程（`cfg.connections / cfg.threads`，wrk.c:107），如 400 连接 12 线程实际只建 396 个；`done()` 阶段的 `script_summary/script_errors` 通过栈上表格传递（script.c:212-242），行为依赖 Lua 侧调用顺序，属脆弱但未出问题的设计。

## 8. 社区与维护状态

- **单人项目**：contributors API 含匿名仅返回 1 人——wg（Will Glozer），80 次提交；近期 8 条提交作者均为 Will（gh api commits）。PR 已在该仓库禁用（issues API 返回 410 "Pull requests are disabled"），外部贡献只能走 fork 或 issue。
- **活跃度**：最后一次代码提交 2021-02-07（a211dd5，升级 OpenSSL 1.1.1i 并配置 GitHub Actions），master 与 next 分支同指该提交；仓库 pushed_at 显示 2023-12-30 但无对应代码变更（两分支 HEAD 未变，应为仓库级操作）。截至分析日（2026-11-14）已近 5 年无代码更新，203 个 open issue 无人回应。
- **发布节奏**：从不用 GitHub Release，仅打 tag。4.x 共 4 个版本（4.0.0-4.2.0，2019-2021），4.2.0 为最新。
- **社区惯性**：尽管停滞，wrk 仍是压测事实标准之一：3022 fork、727 watch、几乎所有性能工程教程的默认工具；Linux 各发行版、Homebrew 均有打包（打包生态未在仓库内核实，未确认）。社区衍生修复散布在大量 fork 与发行版补丁中。

## 9. 亮点与风险

**亮点**

1. 极致精悍的架构：约 1600 行自有 C 实现"多线程反应器 + 每连接状态机 + 无锁统计"，每线程独立事件循环与 lua_State，跨线程仅靠原子直方图写入，是学习高性能网络编程与 C/Lua 嵌入式设计的教科书级范本（src/wrk.c、src/script.c、src/stats.c）。
2. 1μs 分辨率的精确延迟直方图 + coordinated omission 事后修正，统计方法论在同类工具中领先一个身位（src/stats.c）。
3. 六钩子 LuaJIT 脚本 API 设计干净：setup/init/delay/request/response/done 全周期覆盖，静态请求零损耗短路、响应解析按需开启两处优化体现了压测工具对热路径的苛刻要求（SCRIPTING、src/wrk.c:112-120）。
4. 一条 `make` 出静态单文件二进制，deps 内嵌源码包的构建策略 11 年未失手，跨 Linux/macOS/BSD/Solaris 可移植。
5. 作为压测领域事实标准，文档、示例脚本（scripts/ 9 例）、API 语义高度稳定，生态兼容者众多。

**风险/局限**

1. 维护停滞近 5 年：内嵌 OpenSSL 1.1.1i（早已 EOL，含已知 CVE）、LuaJIT 2.1.0-beta3 滚动快照、actions/checkout@v2 均停在上游旧版，issue 平均无人处理（gh api：最后提交 2021-02-07，203 open issues）。
2. 修正方法论的先天局限：`stats_correct` 基于平均间隔，饱和施压下高分位延迟仍可能失真，与 wrk2 的争论使其不适合作为 p99.99 的权威来源（src/stats.c:33-43）。
3. 无测试、PR 禁用、单人决策：任何修复或安全补丁都只能期待作者本人，社区只能靠 fork 承接。
4. 平台与功能边界：无 Windows 原生支持（ae 依赖 epoll/kqueue/evport/select）；无 HTTP/2、无分布式施压、连接数整除分配有微小损耗（wrk.c:107）、时长秒级精度。

## 10. 总结评价

wrk 是"一个人用最小代码量做成行业事实标准"的典范：ae 事件循环 + http-parser + LuaJIT 三块成熟积木之上，用约 1600 行自有 C 完成了多线程反应器、每连接状态机、精确延迟直方图与 CO 修正，工程判断力处处可见。它已近 5 年未更新，但这反而印证了其"完成度"——压测工具的需求十年未变，wrk 的能力也未被淘汰，只是内嵌依赖老化与安全维护缺位使其在生产环境宜通过发行版打包版或补丁版使用。适合谁用：需要单机极限施压、可脚本化压测场景的服务端工程师；适合谁读源码：想精通 epoll 事件驱动模型、C 与 LuaJIT 嵌入集成、无锁统计结构、以及理解 coordinated omission 争议的技术细节的每一个性能工程师——5000 余行读懂一个时代经典，投入产出比极高。
