# jqlang/jq 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 C｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：jq 是命令行 JSON 处理器的事实标准——用一门自带 DSL（jq 语言）对 JSON 做切片、过滤、映射与变换，地位相当于 sed/awk/grep 之于文本（README.md）。

| 项目 | 值 |
| --- | --- |
| 名称 | jqlang/jq（原 stedolan/jq，2012-07 创建，gh api） |
| 星数 | 39500（任务快照）；gh api 今日返回 36196，两口径并存以任务快照为准 |
| 主语言 | C（可移植 C，零运行时依赖，README.md） |
| 许可证 | MIT（COPYING）；decNumber 部分为 ICU License（README.md、COPYING） |
| 默认分支 | master（gh api） |
| 最近活跃 | pushed_at 2026-11-02（gh api）；最新 release jq-1.8.2（2026-06-20，gh api） |
| 贡献者 | 约 272 人（gh api contributors 分页推算） |
| Fork 数 / open issues | 2343 / 455（gh api） |

## 2. 项目是什么，解决什么问题

jq 解决的核心问题：在 shell 管道里对结构化 JSON 数据做查询与变换，就像 sed/awk 处理文本一样顺手。它不是"用正则抠 JSON"的临时方案，而是带了一门完整语言：管道 `|`、过滤器、路径表达式、模式绑定 `as $x`、`reduce/foreach` 折叠、`try/catch`、`label/break`、自定义函数 `def`、模块 `import/include`（`src/parser.y` 的 token 表完整列出这些语法设施）。

典型用法是 `jq '.foo[] | select(.bar > 1)'` 这类一行式；复杂时也可以写成多行程序甚至 shebang 脚本（NEWS.md 1.8.0 记录了对 Tcl 风格多行注释的支持，明确提到 shebang 场景）。CLI 面覆盖 -n/-R/-s/-c/-r/-j/--stream/--seq/--arg/--slurpfile 等（`src/main.c:49` usage()），`--stream` 甚至提供了流式解析路径（`src/main.c` usage 文本）。

参数注入同样是一等公民：`--arg name value` 传入字符串、`--argjson` 传 JSON、`--slurpfile/--rawfile` 读文件，程序内通过 `$name` 与 `$ARGS.named[]/$ARGS.positional[]` 访问（src/main.c usage() 末段）——这让 jq 可以无引号转义地组装脚本，是它能在 CI 和运维脚本里大规模铺开的原因之一。

工程上 jq 的特点是"小而硬核"：整个解释器（词法→语法→字节码编译→VM 执行）不到 1 万行手写 C，外加一套引用计数的 JSON 值模型 `jv`（`src/jv.h`、`src/jv.c`）。它把自己编译成 libjq 供其他程序嵌入（`libjq.pc.in` 提供 pkg-config），这直接催生了大量下游生态。

生态位：jq 语言已是各语言重实现的对标对象——Go 的 gojq、Rust 的 jaq、yq 的 jq 语法模式、交互式的 jid/jaq 等，都是围绕 jq 语法兼容性转（此为业界通识，本次未逐一核实源码）。jq 官网 jqlang.org 与在线 Playground play.jqlang.org（README.md）进一步巩固了它作为"参考实现"的地位。

治理史：jq 1.6（2018-11，gh api release）之后原维护者 stedolan 停更近 5 年，社区接手将仓库迁至 jqlang 组织，2023-09 发布 1.7 恢复活跃（gh api：1.7 2023-09-06、1.7.1 2023-12-13），此后进入稳定发版节奏：1.8.0（2025-06）、1.8.1（2025-07）、1.8.2（2026-06，含 16 个 CVE 修复，NEWS.md）。组织迁移的具体决议过程本次未在仓库内找到文档，标注"未完全核实"，但发版断档-恢复的时间线有 gh api 数据支撑。

## 3. 技术栈与依赖

- **语言**：C99 级可移植 C，构建系统为 autotools（configure.ac、Makefile.am）。
- **编译器工具链**：flex（`src/lexer.l`）+ bison/yacc（`src/parser.y`）；发布 tarball 内已提交生成好的 `src/lexer.c`、`src/parser.c`，因此从 release 构建不需要 flex/bison（README.md、configure.ac `AC_PROG_YACC`）。
- **Oniguruma**：正则引擎，以 git submodule 形式引入（`.gitmodules` 指向 kkos/oniguruma；tarball 中 vendor/oniguruma 为空目录，子模块内容不入包），支持 `--with-oniguruma=builtin` 内置编译或系统库（README.md、configure.ac）。`test/match/capture` 系列函数依赖它（`src/builtin.c` `_match_impl`）。
- **decNumber**：IBM 十进制浮点库，直接进树（vendor/decNumber/），用于保留数字字面量精度——jv 的 NUMBER 内部按十进制存储（`src/jv.c:198` include decNumber.h，`src/jv.c:533` jv 结构内嵌 decNumber 字段），这是 1.7+ "不丢大整数精度"的基础。
- **jv_dtoa.c**（4276 行）：内置修改版 David Gay dtoa，负责浮点↔字符串转换（`src/jv_dtoa.c`）。
- **零运行时依赖**：静态链接可产出单文件可执行程序（README.md `make LDFLAGS=-all-static`）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| src/ | 全部核心：词法、语法、编译器、VM、jv 值模型、内建函数、CLI（约 2.87 万行，含生成文件） |
| tests/ | .test 格式测试集、模块导入测试（tests/modules/）、6 个 fuzz 目标（tests/jq_fuzz_*.c/cpp） |
| docs/ | Pipfile + build_manpage.py/build_website.py/build_mantests.py：手册与官网、从手册生成测试 |
| vendor/ | decNumber（进树）+ oniguruma（submodule） |
| config/ m4/ build/ | autotools 支撑文件 |
| scripts/ | version（git describe 出版本号）、gen_utf8_tables.py、crosscompile |
| .github/workflows/ | ci.yml（多平台交叉编译矩阵）、bsd.yml、valgrind.yml、scanbuild.yml、oniguruma.yml、decnum.yml、manpage.yml、website.yml |
| jq.1.prebuilt | 预构建 man 手册（128KB） |

组织方式是典型的"单包 C 工程"：一个 src/ 装下整台解释器，模块边界靠头文件（jq.h 对外 API、jv.h 值模型、compile.h/bytecode.h 中间层）。没有 CMake/多包花活，靠 autotools 一个 Makefile.am（230 行）管到底。

## 5. 核心架构与关键实现

本次精读了三个子系统：语法前端（parser.y）、执行引擎（execute.c）、编译中间层（compile.c），并统计了内建函数库。

### 5.1 编译管线：lexer → bison 语法 → block IR → 字节码

jq 语言实现是教科书式的解释器分层：

1. **词法**：`src/lexer.l`（192 行手写规则），用 flex start-condition 状态机处理字符串插值（IN_QQSTRING/IN_QQINTERP）、注释、括号配对（IN_PAREN/IN_BRACKET/IN_BRACE，`src/lexer.l:82-88` enter/try_exit）；变量 token 规则在 `src/lexer.l:130`（支持 `a::b` 命名空间形式）。
2. **语法**：`src/parser.y`（971 行）用 bison，`%expect 0`（`src/parser.y:97`）——语法冲突数被硬性压到零，靠一长串优先级声明（`src/parser.y:99-113`）消解，这是对 jq 运算符语义（管道右结合、逗号左结合、`//` 定义或、赋值系非结合）的精确编码。语法动作直接产出 **block**（指令双向链表），即 IR：`src/compile.c:13-16` 注释明确 "The intermediate representation for jq filters is as a sequence of struct inst, which form a doubly-linked list"。
3. **绑定解析**：IR 里的名字引用靠 `inst.bound_by` 三态（未绑定/绑定者/已绑定）+ `block_bind()` 在编译期解析词法作用域（`src/compile.c:38-44` 注释），函数、参数、变量统一为 closure 模型。
4. **字节码**：最终编译为 42 个 opcode 的栈式字节码（`src/opcode_list.h`，grep 计数 42 个 OP 宏），含 `CALL_BUILTIN`（调 C 函数）、`CALL_JQ`/`TAIL_CALL_JQ`（调 jq 函数，支持尾调用）、`CLOSURE_CREATE`、`PATH_BEGIN/PATH_END`（为 `|=` 更新赋值维护路径）、`FORK`/`BACKTRACK`（生成器分支）。

一个很有意思的私货：`src/parser.y:687-701` 有个 `'$' '$' '$' BINDING` 规则，注释原话 "This `$$$$varname` hack is strictly private to jq builtins. DO NOT USE!!... I will break your jq code if you do use this outside src/builtin.jq"——它编译为 `LOADVN`（加载变量并"偷走"引用而非拷贝），仅用于 `src/builtin.jq:22-31` 的 `_modify` 实现里避免持有 `.` 的引用。

### 5.2 执行引擎：基于回溯栈的生成器 VM（execute.c）

jq 每个过滤器都是"多值生成器"（一个输入产生 0..n 个输出），VM 为此设计了一套非递归的、可暂停/恢复的执行栈：

- 状态机 `struct jq_state`（`src/execute.c:19-51`）持有执行栈 `stk`、当前帧 `curr_frame`、fork 栈 `fork_top`。
- 函数调用帧 `struct frame`（`src/execute.c:65-71`）+ closure（字节码指针 + 栈地址），词法作用域用 `frame_get_level()` 沿 env 链上溯整数层（`src/execute.c:91-98`）——闭包捕获的是栈位置而非堆环境，配合 `src/exec_stack.h` 用"栈内索引 stack_ptr"而非裸指针实现，栈重分配时不需要修指针。
- 对外 API 是 `jv jq_next(jq_state*)`（`src/execute.c:340`）：每次调用推进到下一个输出并**完整保存现场**，返回时从 `stack_restore(jq)` 恢复——这就是 jq 顶层能流式吐出多个结果、且 `--seq`/`--stream` 能边解析边执行的原因。
- 回溯机制的实现很巧：`#define ON_BACKTRACK(op) ((op)+NUM_OPCODES)`（`src/execute.c:338`），每个 opcode 天然有一个"回溯版"——失败/继续产出时把当前 opcode 加上 NUM_OPCODES 变成另一半分发分支（`src/execute.c:406-408`），`EACH`、`RANGE`、`TRY_BEGIN`、`FORK`、`DESTRUCTURE_ALT` 等都有配对的 ON_BACKTRACK 处理（grep 可见 30 余处 goto do_backtrack）。这把"迭代器状态"完全编码进字节码双入口，VM 主循环保持单一 while(1)。

```c
// src/execute.c —— 回溯即"第二个 opcode 入口"
#define ON_BACKTRACK(op) ((op)+NUM_OPCODES)

jv jq_next(jq_state *jq) {
  uint16_t* pc = stack_restore(jq);          // 恢复上次暂停现场
  ...
    if (backtracking) {
      opcode = ON_BACKTRACK(opcode);         // 切到该指令的回溯分支
      backtracking = 0;
      raising = !jv_is_valid(jq->error);
    }
```

这个设计解释了 jq 语言最反直觉的地方：`select` 返回 `empty`、`foreach` 逐个产出、`try/catch` 恢复执行，全都是同一套 FORK/BACKTRACK 字节码在 VM 栈上的表现，语言层无需任何运行时线程。

### 5.3 值模型与内建库：jv + 双层 builtin

- **jv 值模型**：8 种 kind（INVALID/NULL/FALSE/TRUE/NUMBER/STRING/ARRAY/OBJECT，`src/jv.h:20-27`），引用计数 + 不可变字符串，对象/数组用开放定址与 mmap 式结构（`src/jv.c` 2190 行）；JSON 解析独立成 `src/jv_parse.c`（899 行），支持 `--stream` 增量解析。数字一律走 decNumber 保留字面量精度（`src/jv.c:580` `decNumberFromString`）。
- **双层内建**：C 层 `src/builtin.c:2039-2111` 的 `function_list[]` 显式注册约 70 个原语（setpath/getpath/delpaths、sort/_sort_by_impl、_match_impl、strftime 系、tojson/fromjson 等），另加 BINOPS 宏批量注册二元运算和 libm.h 的数学函数；jq 层 `src/builtin.jq`（235 行）用 jq 语言本身定义了 103 个 `def`（grep 统计）——`map`、`select`、`sort_by`、`group_by`、`recurse`、`to_entries/from_entries`、`_modify` 等全是"用 jq 写 jq 标准库"，C 只提供够用的原语。这种"C 原语 + jq 语法糖"分层让标准库既快又可读，也是理解 jq 语义的最佳教材。
- **模块系统**：`src/linker.c`（484 行）实现 import/include 的路径解析与模块元数据（`modulemeta`），tests/modules/ 下有 shadowing、循环导入等专测（cycle_a.jq、cycle_b.jq）。
- **CLI**：`src/main.c`（738 行）约 30 个选项（usage() 全文列出：-n/-R/-s/-c/-r/--raw-output0/-j/-a/-S/-C/-M/--tab/--indent/--stream/--seq/-f/-L/--arg/--argjson/--slurpfile/--rawfile/--args/--jsonargs/-e 等），named/positional 参数通过 `$ARGS` 暴露给程序。

## 6. 构建与运行

来自 README.md 与 Makefile.am 的实际命令：

```bash
# 从 git 构建（需 libtool/automake/autoconf/flex/bison）
git submodule update --init
autoreconf -i
./configure --with-oniguruma=builtin
make -j8
make check          # 跑全部 .test 测试集
sudo make install

# 静态单文件构建
make clean && make LDFLAGS=-all-static

# 从 release tarball 构建可跳过 autoreconf（生成文件已提交），无需 flex/bison
```

运行：`jq [options] <filter> [file...]`，Docker 镜像 `ghcr.io/jqlang/jq`（README.md 示例 `docker run --rm -i ghcr.io/jqlang/jq:latest < package.json '.version'`）。

## 7. 测试与工程质量

- **测试格式自成一派**：`tests/jq.test`（2896 行）是"程序/输入/期望输出"三行一组的数据驱动测试（文件头注释），由 `src/jq_test.c`（763 行）编译进 `make check` 执行；统计约 679 个正向用例 + 19 个 `%%FAIL` 语法错误用例。另有 `tests/man.test`（1031 行）由 `docs/build_mantests.py` 从手册 manual.yml 的示例自动生成——**文档即测试**，手册示例与实现永不漂移，这是 jq 工程质量最值得抄的设计（docs/build_mantests.py、tests/man.test）。
- **测试面**：onig.test（224 行，正则）、base64.test、uri.test、optional.test；tests/modules/ 覆盖模块导入/遮蔽/循环依赖。合计 .test 文件约 4400 行（wc 统计）。
- **make check 目标**：`TESTS = tests/mantest tests/jqtest tests/shtest tests/utf8test tests/base64test tests/uritest`（Makefile.am:144），开启 oniguruma 时追加 onigtest/manonigtest（Makefile.am:219）。
- **Fuzz**：tests/ 下 6 个 libFuzzer 目标（jq_fuzz_parse.c、jq_fuzz_parse_extended.c、jq_fuzz_parse_stream.c、jq_fuzz_load_file.c、jq_fuzz_compile.c、jq_fuzz_execute.cpp），覆盖解析、编译、执行全链路。
- **CI 矩阵**（.github/workflows/）：ci.yml 交叉编译 Linux 多架构（含 mips）+ macOS + Windows arm64；bsd.yml 跑 FreeBSD；valgrind.yml 内存检查（tests/local.supp 抑制文件）；scanbuild.yml 静态分析；decnum.yml 验证无 decNumber 构建路径；oniguruma.yml 对比内置/系统正则库。1.8.2 的 NEWS.md 一次性修复 16 个 CVE（含哈希种子随机化防碰撞 DoS、递归深度限制防栈溢出），说明安全审计与 fuzz 已成为常规流程。

## 8. 社区与维护状态

- **活跃度**：gh api pushed_at 2026-11-02，仓库持续有提交；455 个 open issue 处于有人响应的规模。
- **发版节奏**：迁入 jqlang 后明显规律化——1.7（2023-09）→ 1.7.1（2023-12）→ 1.8.0（2025-06）→ 1.8.1（2025-07）→ 1.8.2（2026-06）（gh api releases）。1.8 系列开始版本号固定为三段式 `1.X.Y`（NEWS.md 1.8.0），release 产物带 provenance attestation、GPG 签名（KEYS、sig/），并提供 `gh attestation verify` 验证方式（NEWS.md 1.8.2）。
- **贡献者结构**：约 272 位贡献者（gh api contributors 分页 272 页×1 推算）；NEWS.md 的 CVE/功能修复署名显示 itchyny、emanuele6、wader 等核心活跃维护者主导，另有大量社区署名——典型的"基金会式组织 + 核心干将"治理形态。
- **治理史**：stedolan 个人仓库 → jqlang 社区组织（仓库 created_at 仍为 2012-07-18，gh api，证实是迁移而非重建）；官方 Discord 社区（README.md）。迁移决策过程仓库内无存档文档，未完全核实。

## 9. 亮点与风险

**亮点**

1. 单一可执行文件、零运行时依赖、静态可链接，跨平台分发体验极佳（README.md；Dockerfile）。
2. 解释器架构教科书级：bison 语法零冲突（`src/parser.y:97`）、block IR + 字节码 + 回溯 VM 三层清晰，1 万行内完成一门带生成器语义的语言（src/compile.c、src/execute.c）。
3. 文档即测试：手册示例自动生成 man.test，文档与实现严格同步（docs/build_mantests.py）。
4. 标准库用 jq 语言自身实现（src/builtin.jq 103 个 def），既降 C 代码复杂度又天然暴露语言表达力。
5. 工程护栏完备：6 个 fuzz 目标、valgrind/scanbuild CI、多架构交叉编译矩阵、release 产物 attestation（tests/jq_fuzz_*、.github/workflows/）。

**风险/局限**

1. C + 手写内存管理天然带安全暴露面：1.8.2 一个版本修复 16 个 CVE（NEWS.md），深度嵌套输入导致的栈溢出类问题反复出现，嵌入方需及时跟版。
2. 发版历史上出现过 5 年断档（1.6→1.7），项目命运一度系于个人维护者；jqlang 治理降低了但未消除单点风险。
3. jq 语言本身仍在演进且有破坏性变更（如 1.8.0 修改 as 绑定优先级、限制数组/对象最大 2^29 元素，NEWS.md），脚本跨版本迁移需读变更日志。
4. Oniguruma 为 submodule，从 git 构建需额外步骤，tarball 用户则拿不到其源码树（vendor/oniguruma 为空，`.gitmodules`）。

## 10. 总结评价

jq 是"小工具做成基础设施"的典范：它用一个不到 3 万行的 C 解释器定义了一门 JSON 处理语言，又靠零依赖、单文件、文档即测试的工程实践把自己铺进了几乎每台开发机和无数 CI 脚本里；从 stedolan 到 jqlang 的组织迁移更是一个经典的开源治理续命案例。适合所有需要稳定处理 JSON 的工程师直接使用；适合想学"如何用 C 写一门带生成器和闭包的 DSL 解释器"的人精读 src/parser.y → src/compile.c → src/execute.c 这条主线；对维护自有 DSL 的团队，它的 `$$$$var` 私有语法 hack（parser.y 注释明言仅供 builtin.jq）和 man.test 文档同步机制也是两份难得的实战样本。
