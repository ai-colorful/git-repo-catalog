# koalaman/shellcheck 项目分析

> 星数 41500（快照 2026-10-24）｜主语言 Haskell｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

ShellCheck 是 Shell 脚本静态分析工具的事实标准：给出脚本中引号缺失、bashism、变量陷阱等问题的 SC 编号警告与修复建议。单维护者（Vidar Holen）主导、Haskell 编写，被几乎所有编辑器 lint 插件与 CI 平台集成。

| 项 | 值 |
|---|---|
| 名称 | koalaman/shellcheck |
| 星数 | 41500（快照）；gh api 实测 40335（2026-11-14） |
| 语言 | Haskell（src 下约 2.1 万行 .hs） |
| 许可证 | GPL-3.0（ShellCheck.cabal / LICENSE） |
| 版本 | 0.11.0（2025-08-03 发布，ShellCheck.cabal `Version: 0.11.0`） |
| 最近活跃 | 最近 push 2026-11-09（gh api），CHANGELOG.md 顶部"Git"段有未发布变更 |
| 贡献者 | 约 179 人（gh api contributors 分页推算） |

## 2. 项目是什么，解决什么问题

ShellCheck 解析 shell 脚本并在语法/语义层面给出诊断。README.md 明确了三层目标：解释初学者常见的语法错误（shell 报错信息晦涩）；指出中级水平的语义问题（如变量未加引号导致的分词/glob）；提示高级用户脚本在极端场景下才暴露的陷阱。每条诊断有唯一 SC 编号（如 SC2086），编号即文档——用户在 wiki 按编号查详情，编辑器按编号做忽略。

与大多数 linter 不同，ShellCheck 是先完整解析再分析的：`Parser.hs`（3701 行）用 parsec 手写了 sh/bash 语法的解析器（而非正则匹配），产出 `AST.hs` 定义的抽象语法树，再由分析器在树上做模式匹配与数据流推断。这使得它对 here-doc、嵌套引号、子 shell 等难题的处理远比基于正则的工具可靠。

使用形态覆盖全生态：终端 `shellcheck script.sh`（shellcheck.hs CLI）；网页版 shellcheck.net（README 称"始终同步最新 git commit"）；编辑器插件（Vim ALE/Syntastic、Emacs Flycheck、VSCode vscode-shellcheck 等，README 列表）；CI/平台预装（Travis、Codacy、Code Climate、CircleCI Orb、GitHub Actions Linux 镜像等，README）。`--format` 提供 tty/gcc/checkstyle/json/json1/diff/quiet 七种输出（shellcheck.hs 第 170-176 行），其中 gcc 格式兼容错误定位协议，是它被第三方工具广泛消费的关键。

忽略机制完备：行内注释 `# shellcheck disable=SC2086`、文件级注解、`.shellcheckrc` 配置文件、以及新近加入的 EditorConfig `shellcheck.*` 键（CHANGELOG "Git" 段"Support EditorConfig shellcheck.* keys"；`EditorConfig.hs` 468 行完整实现解析）。可执行文件自动推导 shell 类型：`.ksh/.bash/.bats/.dash/.envrc` 后缀映射到对应方言，`.sh` 回落 Bash 并触发 SC2148 提醒缺 shebang（Checker.hs `shellFromFilename`）。

## 3. 技术栈与依赖

- 语言/编译器：Haskell (Haskell2010)，兼容 GHC 8.0 至 9.14（cabal `tested-with`，CI matrix 实测 8.0/8.10/9.0.1/9.4.7/9.6.6/latest）。
- 构建体系：Cabal 为主（ShellCheck.cabal），附带 stack.yaml（resolver lts-22.6）。
- 关键依赖（ShellCheck.cabal build-depends）：
  - `parsec` — 手写 shell 解析器的基础；
  - `aeson` — JSON 输出格式器；
  - `fgl` — 函数式图库，支撑 CFG 控制流图（CFG.hs）；
  - `regex-tdfa` — 文本正则（Regex.hs 封装）；
  - `QuickCheck` — 属性测试（发布前由 `striptests` 剥离）；
  - `Diff` — diff 输出格式器；
  - `mtl`/`transformers` — Writer monad 收集告警（AnalyzerLib.hs）；
  - `containers`/`array`/`bytestring`/`directory` 等基础库。
- 无运行时外部依赖：发布产物为单文件静态二进制（builders/ 下各平台 Docker 构建静态链接）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `src/ShellCheck/Interface.hs` | 对外数据契约：CheckSpec/ParseResult/Severity/Fix 等 + SystemInterface 抽象（343 行） |
| `src/ShellCheck/AST.hs` / `ASTLib.hs` | 语法树类型与树操作工具 |
| `src/ShellCheck/Parser.hs` | parsec 手写解析器，产出 AST 与 1xxx 解析错误（3701 行） |
| `src/ShellCheck/Checker.hs` | 总管线 checkScript + 内嵌属性测试（611 行） |
| `src/ShellCheck/Analyzer.hs` / `AnalyzerLib.hs` | 分析入口与共享基础设施（参数、告警原语 warn/err/info/style） |
| `src/ShellCheck/Analytics.hs` | 最大规则库：111 个节点检查 + 15 个树检查（5500 行） |
| `src/ShellCheck/Checks/` | Commands.hs（命令级检查）、ShellSupport.hs（bashism/方言检查）、ControlFlow.hs、Custom.hs（站点定制钩子） |
| `src/ShellCheck/CFG.hs` / `CFGAnalysis.hs` | 控制流图构建与数据流分析（"extended-analysis"） |
| `src/ShellCheck/Formatter/` | 7 种输出格式器 |
| `src/ShellCheck/Fixer.hs` | 依据 Fix/Replacement 应用自动修复 |
| `src/ShellCheck/EditorConfig.hs` | .editorconfig 中 shellcheck.* 键的解析（新特性） |
| `shellcheck.hs` | CLI 入口（780 行）：flag 解析、格式器调度、多文件处理 |
| `builders/` | 7 个平台（linux x86_64/aarch64/armv6hf/riscv64、darwin x86_64/aarch64、windows x86_64）的 Docker 静态构建脚本 |
| `test/` | 测试入口 shellcheck.hs（43 行）与发布校验脚本（buildtest/check_release/distrotest） |
| `striptests`、`quickrun`、`quicktest`、`setgitversion`、`manpage` | 发布辅助脚本：剥离测试、快速运行、版本注入、man 页生成 |

组织方式：单 cabal 包单体仓库，模块按"解析 → 分析 → 格式化"分层清晰；测试不是独立测试目录而是内嵌在各源文件中（QuickCheck `prop_*` 属性），这是 Haskell 社区的经典实践。

## 5. 核心架构与关键实现

### 5.1 检查管线（src/ShellCheck/Checker.hs）

`checkScript :: Monad m => SystemInterface m -> CheckSpec -> m CheckResult` 是唯一入口：先 `parseScript`（Parser.hs）得 AST、token 位置表与解析期告警；再对解析成功的根节点跑 `analyzeScript`（Analyzer.hs）；最后用 `tokenToPosition` 把分析期 Token 告警翻译回行列号，与解析告警合并去重、按位置排序，并按 severity/include/exclude 过滤（`shouldInclude`）。核心流程如下：

```haskell
checkScript contents = do
    result <- parseScript sys newParseSpec { psScript = contents, ... }
    let analysisMessages =
            maybe []
                (arComments . analyzeScript . analysisSpec)
                    $ prRoot result
    let translator = tokenToPosition tokenPositions
    return . nub . sortMessages . filter shouldInclude $
        (parseMessages ++ map translator analysisMessages)
```

### 5.2 规则体系与 SC 编号（src/ShellCheck/Analytics.hs 等）

规则按编号分区：1xxx 为解析期错误（Parser.hs 经 `parseProblemAt` 直接给出，如 SC1017 字面回车、SC1071 不支持的 shell）；2xxx 为分析期警告，跨 Analytics.hs、Checks/Commands.hs、Checks/ShellSupport.hs、Checks/ControlFlow.hs、CFGAnalysis.hs 等模块。用 perl 扫描全部 `warn/err/info/style` 调用统计，源码实际发出 **309 个唯一 2xxx 码**（含可选项），与官方 wiki 规则总量同级。

Analyzer.hs 组合 5 个 checker：

```haskell
checkers spec params = mconcat $ map ($ params) [
    ShellCheck.Analytics.checker spec,
    ShellCheck.Checks.Commands.checker spec,
    ShellCheck.Checks.ControlFlow.checker spec,
    ShellCheck.Checks.Custom.checker,
    ShellCheck.Checks.ShellSupport.checker
    ]
```

Analytics.hs 内部把 111 个逐节点检查（`nodeChecks` 列表）经 `nodeChecksToTreeCheck` 单遍遍历打包成树检查，与 15 个独立树检查（`treeChecks`：未使用变量、未传参函数、cd/pushd 未检查返回值等）共同构成默认规则集。另有 12 组可选规则（`optionalTreeChecks`，如 `avoid-nullary-conditions`→SC2243/2244、`require-variable-braces`、`useless-use-of-cat`），经 `--enable=name` 或 rc 文件激活。

Haskell 规则写法是"AST 模式匹配 + 告警原语"，一条规则即一个函数，旁边紧跟正反例属性测试。以 SC2066（for 循环双引号只跑一次）为例（Analytics.hs）：

```haskell
checkForInQuoted _ (T_ForIn _ f [T_NormalWord _ [word@(T_DoubleQuoted id list)]] _)
    | any willSplit list && not (mayBecomeMultipleArgs word)
            || maybe False wouldHaveBeenGlob (getLiteralString word) =
        err id 2066 "Since you double quoted this, it will not word split, and the loop will only run once."
checkForInQuoted _ (T_ForIn _ _ [single] _)
    | maybe False (',' `elem`) $ getUnquotedLiteral single =
        warn (getId single) 2042 "Use spaces, not commas, to separate loop elements."
```

告警原语统一在 AnalyzerLib.hs（`warn/err/info/style` 对应 Severity 四级），`errWithFix` 等变体可携带自动修复补丁（Fix/Replacement，Interface.hs），供 `--format=diff` 与 Fixer.hs 消费。

### 5.3 多方言支持（Interface.hs + Checks/ShellSupport.hs + Parser.hs）

方言枚举为 `data Shell = Ksh | Sh | Bash | Dash | BusyboxSh`（Interface.hs 第 230 行）。**不支持 zsh**：Parser.hs 对 `#!/bin/zsh` 报 SC1071 "ShellCheck only supports sh/bash/dash/ksh/'busybox sh' scripts"（可注释禁用后按近似方言解析）。方言差异的处理方式是"超集解析 + 支持度告警"：统一解析为 AST，再由 ShellSupport.hs 的 `checkBashisms` 按 shebang 指定方言逐节点检查特性可用性——如 `local` 在 dash 中合法（prop_checkBashisms145b 用 `#!/bin/dash` verifyNot SC3043）但在 POSIX sh 中告警、`[[ ]]`/进程替换/扩展 glob 在 sh 中报 SC3010/3001/3002。busybox sh 支持是 v0.10.0 新增（CHANGELOG）。文件后缀兜底推导在 Checker.hs `shellFromFilename`，含 `.envrc→Bash` 这样的生态细节。

### 5.4 数据流分析（src/ShellCheck/CFG.hs + CFGAnalysis.hs）

v0.9.0 起引入控制流图分析（两文件合计约 2700 行）：CFG.hs 用 fgl 构建每个 token 的效果节点（赋值/读/执行），CFGAnalysis.hs 在图上推断变量取值空间与可达性，支撑 SC2248"变量虽安全仍建议引号"（`checkVerboseSpacefulnessCfg`）、SC2317 不可达命令等需要跨语句上下文的检查。该分析默认开启，可用 `extended-analysis=false` 指令或同名 CLI flag 关闭（AnalyzerLib.hs 第 213 行：指令、rc、flag 三层优先级合并）。

### 5.5 可 mock 的 IO 抽象（Interface.hs）

`SystemInterface m` 是一个 monad 参数化的记录类型（siReadFile/siFindSource/siGetConfig），生产环境传入 IO 单子的实现，测试传入 `mockedSystemInterface [("lib", "bar=1")]`——checker 全管线因此是纯函数式的。所有 source/`.shellcheckrc`/EditorConfig 相关属性测试（Checker.hs 611 行里 86 个 prop_*）都建立在该抽象上，例如 `prop_worksWhenSourcing`。这类"依赖注入以换取可测性"的设计在 Haskell 里以类型类/记录实现得极其轻量，是本项目架构上最值得借鉴的一点。

## 6. 构建与运行

来自 ShellCheck.cabal 与 README"Compiling from source"节：

```bash
# 构建（Cabal）
cabal update && cabal build all
# 运行测试（内嵌 QuickCheck 属性，test-suite test-shellcheck → test/shellcheck.hs）
cabal test all
# 生成发布版源码并剥离测试（去除 QuickCheck/TemplateHaskell 依赖，减小二进制）
./striptests
# 本地快速验证单个脚本
./quickrun myscript.sh
# 也可用 Stack
stack build   # stack.yaml: resolver lts-22.6
```

运行：`shellcheck [OPTIONS...] FILES...`，常用选项 `--shell/--enable/--exclude/--severity/--rcfile/--list-optional/--external-sources/--files-from`（shellcheck.hs flag 定义）。发布二进制由 builders/ 各平台 Docker 静态构建，snap/snapcraft.yaml 提供 snap 包，Dockerfile.multi-arch 提供多架构镜像。

## 7. 测试与工程质量

- 测试规模：全仓共约 **2435 个 QuickCheck 属性**（`prop_*` 计数：Analytics 1273、Parser 327、Commands 388、ShellSupport 197、Checker 86、EditorConfig 60、ASTLib 61、Fixer 21 等）。每条规则的正反例即属性测试（`verify`/`verifyNot` 风格），且有元测试 `prop_verifyOptionalExamples` 校验每条可选规则的 positive/negative 示例确实触发/不触发规则（Analytics.hs）。
- 测试风格：属性测试 + "expect 具体码列表"断言（如 `check "echo $1" == [2086]`），近似回归测试但由同一框架承载；source 追踪、rc 文件、EditorConfig、DFA 开关均有专门属性覆盖。
- CI（.github/workflows/build.yml）：先 `cabal sdist` 打包源码 artifact，再在 GHC 8.0/8.10/9.0.1/9.4.7/9.6.6/latest 六版本矩阵上测试（fail-fast: false 保证各版本状态独立可见）；distrotest.yaml 对发布产物做发行版级验证（test/distrotest）。
- 工程质量观察：代码分层极清晰（Interface 只放数据、Parser 只管语法、Analytics 只放规则）；`Checks/Custom.hs` 是文档化的站点定制空钩子（"for ease of patching in site specific checks"）；一处值得注意的细节——Checker.hs 头部有 `import Debug.Trace -- DO NOT SUBMIT`，master 快照上仍存在（未确认是否开发者有意保留）。

## 8. 社区与维护状态

- 活跃度：最近 push 2026-11-09（gh api），CHANGELOG.md"Git"段列出未发布变更（SC2337/SC2338 新规则、`--files-from`、EditorConfig 支持），说明代码持续演进。
- Release 节奏：**近年明显放缓，约 1 至 1.5 年一个 minor 版本**——v0.7.1 2020-04、v0.7.2 2021-04、v0.8.0 2021-11、v0.9.0 2022-12、v0.10.0 2024-03、v0.11.0 2025-08（CHANGELOG.md 版本表 + gh api releases 的 published_at 一致）。早期 2014 年曾一年发 6 个版本。这意味着用户报告的修复要等很久才进入 stable 二进制；缓解手段是 shellcheck.net（同步 master）与自行编译。
- 贡献者结构：约 179 名贡献者（gh api），但项目由 Vidar Holen（vidar@vidarholen.net，cabal Maintainer）个人主导十余年，规则文案、架构决策高度集中于一人。1077 个 open issues（gh api）。
- 发行渠道：官方提供 7 平台预编译二进制（builders/），另有 snap 包、Docker 镜像；README 记载大量平台预装（Travis/Codacy/CircleCI/GitHub Actions 等）。

## 9. 亮点与风险

**亮点**
1. 事实标准的生态位：编辑器/CI/lint 框架（ALE、Flycheck、vscode-shellcheck、Trunk、CodeRabbit 等）普遍以 shellcheck 二进制为后端，SC 编号成为社区通用语言（README 集成清单）。
2. "每条规则内嵌正反例属性测试"的工程范式：约 2400 个 QuickCheck 属性与规则同文件共演化，加 `striptests` 发布期剥离，兼得质量与零测试依赖的小体积二进制（Analytics.hs、striptests）。
3. 纯函数式可 mock 架构：`SystemInterface m` 让 source 追踪、rc/EditorConfig、DFA 全部可无副作用测试（Interface.hs、Checker.hs）。
4. 真·解析器而非正则堆砌：parsec 手写 shell 语法 + AST + CFG 数据流，对病态脚本的理解深度远超同类工具（Parser.hs、CFGAnalysis.hs）。
5. 规则文案本身就是教程（"Double quote to prevent globbing and word splitting."），wiki 按码建页，学习成本极低。

**风险/局限**
1. 发版缓慢：stable 版本间隔 13-17 个月，下游发行版打包的常常是落后一年的工具（第 8 节证据）。
2. 不支持 zsh：仅 sh/bash/dash/ksh/busybox sh，zsh 用户被直接拒绝并建议用其他工具（Parser.hs SC1071）。
3. 单维护者风险：架构与规则质量依赖 Vidar Holen 一人，Haskell 门槛进一步限制了外部贡献深度（cabal Maintainer、贡献者分布）。
4. 无 LSP/内置编辑器协议：消费全靠外部插件包装 CLI 输出，`--format=json` 与 checkstyle 是事实接口（未确认有官方 LSP 计划）。

## 10. 总结评价

ShellCheck 是"小工具做成基础设施"的典范：一个单维护者的 Haskell 项目，靠真解析器、数据流分析和每条规则都带正反例的测试纪律，赢得了 Shell 脚本检查的事实标准地位，星数与它的技术纯度高度相称。适合所有写 shell 脚本的工程师把它接进 CI（一行 `shellcheck scripts/*.sh` 即可）；对研究者，它演示了静态分析工具的教科书式分层（Interface/Parser/Analytics/Formatter）与"属性测试即文档"的工程文化；对想在发布二进制里内嵌 linter 能力的项目，它的 `striptests` + 静态链接 + 7 平台 builders 流水线（builders/）也是现成的参考实现。读源码建议从 Interface.hs（数据契约）→ Checker.hs（管线）→ Analytics.hs（挑几条规则）入手，最后进 Parser.hs 与 CFGAnalysis.hs 两座大山。
