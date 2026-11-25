# dylanaraps/pure-bash-bible 项目分析

> 星数 38500（快照 2026-10-24）｜主语言 Shell｜档位 20k-50k｜分析日期 2026-11-14
> 注：gh api 实时查询（2026-11-14）星数已达 41,710，仓库状态为 archived（已归档）。

## 1. 项目速览

一句话定位：一本"只 Bash 不外部命令"的技巧手册——用纯 bash 内建功能替代 sed/awk/cut/head/tail/date 等外部进程的函数与片段合集，以单文件 README 为书、以测试驱动内容质量。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | dylanaraps/pure-bash-bible | gh api |
| 星数 | 38,500（快照）/ 41,710（2026-11-14 实时） | 任务参数 / gh api |
| 语言 | Shell（内容为 bash 代码块） | gh api / README.md |
| 许可证 | MIT（Copyright (c) 2018 Dylan Araps） | src/LICENSE.md |
| 创建时间 | 2018-06-13 | gh api |
| 最后 push | 2023-11-28，随后仓库归档 | gh api |
| 贡献者 | 20 人；0 个 release；62 个 open issues | gh api |
| 体量 | README.md 约 2200 行 / 49KB，全书 103 条技巧 | src/README.md 统计 |

## 2. 项目是什么，解决什么问题

bash 调用外部进程（sed、awk、cut、head、wc、date……）开销很大，fork/exec 的成本在循环中会被放大。作者在开发 neofetch、pxltrm 等项目的过程中积累了大量"用 bash 内建完成同一件事"的写法，遂整理成书（src/README.md 前言段）。书的核心主张写在 FOREWORD 章（README.md L174-178）：bash 比人们以为的强大得多，多数任务无需外部程序即可完成，用内建实现的脚本更快、依赖更少，还能加深对语言本身的理解。

内容形态是"函数 + 用法示例 + 注意事项"：每条技巧给出一个可直接粘贴的 `Example Function`（带 `# Usage:` 注释），随后用 `$` 提示符展示输入输出，需要提醒时用 `**CAVEAT**` 标注 bash 版本要求（如 bash 4+）或平台差异（如 regex 依赖系统正则引擎）。除函数类技巧外，还有参考表式的章节（PARAMETER EXPANSION、CONDITIONAL EXPRESSIONS、ARITHMETIC OPERATORS 等速查表）。

这本书同时也是一本 Leanpub 电子书（https://leanpub.com/bash，README.md L30），仓库里的 `manuscript/` 目录就是 Leanpub 出版稿件，由 `build.sh` 从 README 自动切分生成（src/build.sh L9-14）。质量保障方式在同类内容仓库中相当少见：README 里标记为 ` ```sh ` 的代码块会被 test.sh 提取出来跑 shellcheck 并逐条单元测试（src/test.sh L224-233）。

姊妹篇 pure-sh-bible（POSIX sh 版本）在 README 顶部置顶链接（README.md L1），两者构成"纯 shell 技巧"系列。

## 3. 技术栈与依赖

- 语言：内容全部是 bash（4+ 为主，部分技巧标注 4.2+/4.4+，个别回退兼容 bash 3），仓库自身脚本也是 bash（src/build.sh、src/test.sh）。
- 运行依赖：零。这是项目的卖点——所有技巧只用 bash 内建（参数展开、`read`、`mapfile`、`printf`、`declare -A/-n`、`trap` 等），CONTRIBUTING.md 明文规定"代码必须只用 bash 内建，仅允许 `${HOSTNAME:-$(hostname)}` 式的外部回退"（src/CONTRIBUTING.md L14-17）。
- 开发期工具链：shellcheck（test.sh L232 对 README 代码块 lint）、Travis CI（.travis.yml：Linux 跑 `./test.sh`）。书籍发布依赖 Leanpub（manuscript/Book.txt 是其章节清单格式）。
- 无任何 package.json/go.mod 类工程清单——整个仓库没有第三方依赖。

## 4. 目录结构与代码组织

| 路径 | 职责 |
| --- | --- |
| README.md | 全书主体，约 2200 行：目录 + 22 个一级章节（FOREWORD 至 AFTERWORD） |
| test.sh | 质量核心：提取 README 中 ```sh 代码块 → shellcheck → source → 运行 30 个 test_* 断言 |
| build.sh | 以 `<!-- CHAPTER START/END -->` 注释为界把 README 切分成 manuscript 章节 |
| manuscript/ | Leanpub 稿件：chapter0.txt～chapter19.txt 共 20 章 + Book.txt 章节清单（2020-10 生成） |
| CONTRIBUTING.md | 收录规则：只 bash 内建、函数化包装、写示例、测试命名 `test_func_name` |
| LICENSE.md | MIT |
| .travis.yml | Travis CI 配置，仅一条 script: `./test.sh` |

组织方式评述：这是典型的"单 README 即书"内容仓，README 是唯一事实源（single source of truth），其余一切（书籍稿件、测试输入）都由它派生——build.sh 读 README 切章节，test.sh 读 README 抽代码块。CHAPter 标记注释、` ```sh `/` ```shell ` 两种围栏语义（前者受测、后者仅展示，CONTRIBUTING.md L26-38）构成了极轻量的"元数据嵌入"方案。manuscript 是一次性构建产物（早于当前 README），并非活跃同步。

## 5. 核心架构与关键实现

**机制一：README 即代码库，test.sh 即测试运行器（src/test.sh L224-248）**

test.sh 先用 while+正则把 README 中所有 ```sh 围栏内容抽到临时文件 `readme_code`，交给 shellcheck 校验，失败即整体退出；通过后 `. readme_code` 把这些函数 source 进当前 shell，再遍历 `declare -F` 自动发现并执行所有 `test_*` 函数，最后按 pass/fail 计数决定退出码：

```bash
while IFS=$'\n' read -r line; do
    [[ "$code" && "$line" != \`\`\` ]] && printf '%s\n' "$line"
    [[ "$line" =~ ^\`\`\`sh$ ]] && code=1
    [[ "$line" =~ ^\`\`\`$ ]]   && code=
done < README.md > readme_code
shellcheck -s bash readme_code test.sh build.sh || exit 1
. readme_code
```

这意味着文档中每个函数都是被真实执行过断言的，示例代码与测试永不脱节——文档即测试夹具。

**机制二：README → 书籍稿件（src/build.sh L10-21）**

build.sh 逐行扫描 README，用 `<!-- CHAPTER START/END -->` 把内容装进 `chapter[]` 数组，再用三重参数展开技巧 `: "${chapter[$i]/$'\n'*}"; : "${_/\# }"; : "${_,,}"` 把首行标题转成小写文件名，输出 manuscript/chapterN.txt 与 Book.txt。当前 manuscript 里 chapter18 实为 CONVERSION+CODE GOLF 两章合并（README 中这两章之间缺 CHAPTER END 标记），AFTERWORD 未入稿件——可反推 manuscript 是旧版构建产物。

**机制三：代表性技巧风格（src/README.md）**

- `trim_string`（L194-199）：利用 `: "${1#...}"` 不落临时变量、`$_` 承接上次命令末参数的惯用法，两行完成去首尾空白；
- `cycle`/`toggle`（L750-768）：`${arr[${i:=0}]}` 用 `:=` 默认值初始化游标，`((i=i>=${#arr[@]}-1?0:++i))` 算术三元回卷，实现环形遍历；
- `dirname` 纯参数展开实现（L1072-1092）：14 行处理 `//`、`something/`、`""` 等边界（test.sh L114-156 为其写了 14 个断言，是全书测试最密的一条）；
- 速查表章（PARAMETER EXPANSION L1244-1311、CONDITIONAL EXPRESSIONS L1362-1415）：以 Markdown 表格罗列 `${VAR%%}`、`-nt`、`-v` 等全部展开/测试操作符，充当离线参考卡。

## 6. 构建与运行

本项目无构建产物，"运行"即验证与出书：

```bash
# 运行全部测试（同时 shellcheck lint，需安装 shellcheck）
./test.sh                      # 见 src/test.sh、src/.travis.yml、CONTRIBUTING.md

# 重新生成 Leanpub 稿件（会先 rm -rf manuscript 再重建）
./build.sh                     # 见 src/build.sh

# 阅读本书：直接读 README.md，或访问 https://leanpub.com/bash
```

CI：Travis CI 在 Linux 环境执行 `./test.sh`（src/.travis.yml）。注意 Travis 已于 2021 年终止 OSS 免费服务，README 头部的 travis-ci.com 徽章现已无实际意义，归档后 CI 实际处于停摆状态。

## 7. 收录组织与技巧条目统计

全书按主题分 20 个内容章（FOREWORD/AFTERWORD 除外），共 103 条技巧条目（README 中 103 个 `## ` 标题，与目录条目数一致）。各章条目分布（据 README.md 目录统计）：

| 章 | 条目 | 章 | 条目 |
| --- | --- | --- | --- |
| STRINGS | 17 | TRAPS | 5 |
| ARRAYS | 5 | PERFORMANCE | 1 |
| LOOPS | 6 | OBSOLETE SYNTAX | 3 |
| FILE HANDLING | 8 | INTERNAL VARIABLES | 10 |
| FILE PATHS | 2 | TERMINAL | 3 |
| VARIABLES | 2 | CONVERSION | 2 |
| ESCAPE SEQUENCES | 4 | CODE GOLF | 5 |
| PARAMETER EXPANSION | 6 | OTHER | 11 |
| BRACE EXPANSION | 2 | ARITHMETIC OPERATORS | 5 |
| CONDITIONAL EXPRESSIONS | 4 | ARITHMETIC | 2 |

形态上分两类：函数型技巧（可测试，共 39 个 ```sh 代码块）与片段/速查型（仅展示，83 个 ```shell 代码块），合计 122 个代码块。STRINGS 是绝对重心（17 条，占 1/6），其余章更像参考手册。条目粒度极小（一条一个明确操作），配合目录锚点使其成为"Ctrl+F 优先"的查询型文档。

## 8. 示例设计与命令准确性抽查

**参数化示例设计**：每条函数型技巧遵循统一三段式——`Example Function`（含 `# Usage:` 行）→ `Example Usage`（`$` 提示符交互样例）→（必要时）`Example Usage in script`（脚本化用法）。作者惯用"自定义变量展示法"：先设 `name="   John Black  "` 再把变量传入函数（如 README L208-210、L670-672），让读者看清输入形态；版本要求一律用 `**CAVEAT:** Requires bash 4+` 前置标注（如 split、lower、read_sleep）。这种统一模板让 103 条风格零漂移，也是它适合当速查表的原因。

**准确性抽查（本机 bash 5.2.37，Git Bash/Windows 实测）**：随机抽取 4 条，全部通过——

- `trim_string "    Hello,    World    "` → `Hello,    World`（与 README L206 示例一致）✓
- `hex_to_rgb "#FFFFFF"` → `255 255 255`（README L1819）✓；`rgb_to_hex 0 0 0` → `#000000` ✓
- `printf "%(%Y-%m-%d)T" -1` → `2026-11-14`（strftime 技巧，README L2035）✓
- `cycle` 三连调用 → `a b c `（README L750-753，与 test.sh test_cycle 断言一致）✓

客观评价：抽查与 30 个内置单测表明内容准确度很高；但部分技巧有未在正文突出强调的代价——`head`/`tail`/`dirname`/`basename`/`date` 等函数与外部命令同名（README L907-939、L1072-1122、L2020-2025），在脚本中定义会遮蔽系统命令；`reverse_array` 依赖 `extdebug`+`BASH_ARGV`，bash 5.0+ 需 `shopt -s compat44`（README L645 已注明）；`lines` 用 mapfile 把整文件读入内存（L961-963）。这些是"内建替代"路线的固有折衷，作者基本都作了标注。

## 9. 社区、姊妹项目与作者生态

- **社区参与**：20 位贡献者、62 个 open issues（gh api）；CONTRIBUTING.md 定义了清晰的收录门槛与测试写作流程，issue/PR 驱动内容增补（README L27 "缺什么就开 issue"）。无 release、无版本号，书以"持续追加"方式演进。
- **姊妹项目**：pure-sh-bible（POSIX sh 版，作者仓库现 7.7k 星，gh api）在 README 顶部置顶；另有 writing-a-tui-in-bash（588 星）构成同一作者的"纯 shell 写作"三部曲。
- **作者生态**：dylanaraps（dylan，5542 followers，bio 自述 "natural farmer, natural winemaker, creator of @kisslinux"）是 neofetch（23.6k 星）、pywal（9k）、fff、pfetch、sowm、pxltrm、kiss Linux 等知名项目的作者（gh api users/dylanaraps 及 repos 列表）。**其 GitHub 全部主要仓库均已 archived**：neofetch 2024-07 归档、pure-sh-bible 2023-12 最后 push、pure-bash-bible 2023-11-28 最后 push 后归档。作者约在 2023-2024 年间逐渐淡出（其自述转向务农/酿酒，社区广为流传其离开 kiss Linux 维护；此处仅陈述仓库归档事实，个人动向细节未确认）。bash 4+/shellcheck 相关内容至今大体仍准确，但 unicode/locale 相关建议与 Travis 徽章已过时。

## 10. 亮点、风险与总结评价

**亮点**

1. 文档即代码、文档即测试：```sh 代码块被 shellcheck lint 并逐条单测（src/test.sh），内容准确度在同类技巧书中罕见；
2. 收录纪律严格：只用内建、函数化、CAVEAT 标注版本（src/CONTRIBUTING.md），103 条风格统一，速查效率极高；
3. 覆盖面完整且深：从参数展开/花括号展开速查表到 `$_` 惯用法、nameref、trap、终端像素尺寸（L1755-1760）等冷门技巧；
4. 一次创作多处分发：README → GitHub 阅读 → Leanpub 电子书（manuscript/ 自动生成）；
5. 填补空白：2018 年发布时"纯 bash 替代外部命令"没有系统性中文/英文参考，切中 shell 脚本性能痛点。

**风险与局限**

1. 已归档停更（2023-11-28 后），62 个 issue 无人处理，bash 新版本特性（如 5.1/5.2 的 `${var@K}` 等）不会再收录；
2. 同名函数遮蔽系统命令（head/tail/date/dirname/basename）的设计对初学者有误用风险；
3. Travis CI 徽章与部分性能建议（LC_ALL=C 禁用 unicode）已过时；regex 技巧依赖平台正则引擎，跨平台行为有差异（README L249-251 已标注）；
4. 单文件 2200 行 README，无版本化/release，长文锚点依赖 GitHub 渲染。

**总结评价**：这是 Shell 类内容仓库的标杆之作——把"写手册"做成"写有测试覆盖的代码"，103 条技巧经得起实测抽查，催生了 pure-sh-bible 等一批仿作，也是 dylanaraps 生态（neofetch/pywal/kiss）中流传最广的知识资产。它适合所有写 bash 脚本的人当案头速查表，适合学习 bash 参数展开与内置惯用法的进阶者精读，也适合想学习"内容型仓库如何用工程手段保证质量"的文档作者借鉴其 test.sh 模式；归档意味着它是"完成态的词典"而非持续演进的项目，放心引用、不必期待更新。
