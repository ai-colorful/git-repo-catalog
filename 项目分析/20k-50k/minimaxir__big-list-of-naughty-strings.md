# minimaxir/big-list-of-naughty-strings 项目分析

> 星数 47300（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：一份经过社区筛选、按引发故障的原理分类的"恶意/捣乱字符串"测试数据集，是软件 QA 与模糊测试领域的事实标准输入语料（俗称 BLNS）。

| 项目 | 事实 |
| ---- | ---- |
| 名称 | minimaxir/big-list-of-naughty-strings |
| 作者 | Max Woolf（@minimaxir，亦是 gpt-2-simple / textgenrnn 作者） |
| 创建时间 | 2015-08-08（`gh api repos/...`） |
| 星数 | 47,300（快照）；2026-11-05 实时 API 为 47,718 |
| 主语言 | Python（元数据标注）；实际主体是纯文本数据 `blns.txt` + 少量 Python/Go 读取代码 |
| 许可证 | MIT（`LICENSE`，Copyright 2015-2020 Max Woolf） |
| 最近活跃 | master 最后一次提交 2021-04-17（PR #226，`gh api commits`）；repo pushed_at 2024-04-18 |
| 仓库体积 | 约 330KB，默认分支 master，未归档（`gh api repos/...`） |

## 2. 项目是什么，解决什么问题

即使是拥有海量自动化测试的大公司，也拦不住"奇怪的用户输入"。README 开篇给出的例子：在 Twitter 上发布一个零宽空格（U+200B）会直接触发 internal server error（`README.md` "Why Test Naughty Strings?" 一节，附 GIF）。这类问题不构成安全攻击——普通用户也不太会输入诡异 Unicode——但"意外输入导致 500"永远是糟糕的体验，且往往是更深层的字符串校验缺陷的症状。

BLNS 的解决方式极其朴素：把已知的、高概率引发故障的字符串集中成一份清单 `blns.txt`，每行一条，用 `#` 注释行划分小节，供两类用法：（1）人工 QA 直接打开文件，复制粘贴到输入框里做探索性测试；（2）程序读取（配套 `blns.json` 及各语言封装），作为自动化测试/模糊测试的固定输入语料。README 明确说明它只用于测试自己拥有的软件，且不能替代正规的安全渗透测试（`README.md` Disclaimer）。

清单的条目来源有明确谱系：XSS 部分来自经典 fuzz 语料（条目中的 `fuzzelement1` 系列与 html5sec 变体），Shellshock、Juniper 后门签名、iOS 文字炸弹等直接对应真实 CVE 与历史事故，Zalgo 文本、倒序 Unicode 等来自网络亚文化工具站（各分类注释中标注了来源链接）。这种"每一条都有出处"的性质是它比随手编的乱码列表有价值得多的原因。

项目于 2015 年 8 月以 Show HN 发布即走红（README 收录 7 次 HN/Reddit 热帖，2015-2018 三轮传播），此后成为 QA 圈的标准引用物：至今仍有大量测试框架教程、模糊测试工具把"拿 blns.txt 喂一遍输入框"作为常规步骤。

## 3. 技术栈与依赖

- **数据主体**：纯文本 `blns.txt`（UTF-8，`#` 行为分类注释），无任何运行时依赖。
- **派生格式**：`blns.json`（数组）、`blns.base64.txt` / `blns.base64.json`（整行 base64 编码版，规避查看器/传输层对控制字符的处理），由仓库自带脚本生成。
- **Python**：`naughtystrings/__init__.py`，仅用标准库 `os`，读取上级目录 `blns.txt`，无 PyPI 打包配置。
- **Go**：`naughtystrings/` 包（go.mod 无第三方依赖），用 go-bindata 把两个 JSON 内嵌进 `internal/resource.go`（生成文件，1400+ 行），实现编译期零外部文件。
- **Node**：`package.json` 仅作元数据描述，`main` 指向 `blns.json`，version 1.0.0，无 npm 构建链。
- **构建工具**：`naughtystrings/Makefile` 调用 go-bindata 重新生成 resource.go；`scripts/texttobase64.sh` 用 bash + `base64` 命令逐行转码。

没有 CI、没有测试框架（Go 有一个最小测试）、没有 lint 配置——这是一个刻意保持"数据即仓库"的极简工程。

## 4. 目录结构与代码组织

| 路径 | 职责 |
| ---- | ---- |
| `blns.txt` | 唯一的人工编辑源文件：516 条数据行 + 150 行分类注释，742 物理行 |
| `blns.json` | 程序读取用 JSON 数组（515 条，含特意插入的首位空字符串） |
| `blns.base64.txt` / `blns.base64.json` | base64 编码版（json 676 条），绕开编辑器/协议对特殊字符的改写 |
| `naughtystrings/__init__.py` | Python 读取函数 `naughty_strings()`，解析 txt 并过滤注释 |
| `naughtystrings/*.go`, `internal/resource.go` | Go 封装：`Unencoded()` / `Base64Encoded()` 两个 API，数据由 go-bindata 内嵌 |
| `naughtystrings/go.mod`, `Makefile`, `naughtystrings_test.go` | Go 模块定义、go-bindata 再生成规则、最小测试 |
| `scripts/txt_to_json.py` | `blns.txt` → `blns.json` 转换脚本（作者 Max Woolf） |
| `scripts/texttobase64.sh` | `blns.txt` → `blns.base64.txt` 转换脚本（保留注释行原样） |
| `package.json` | npm 元数据，`main: blns.json` |
| `README.md`, `LICENSE`, `.gitattributes` | 文档、MIT 许可、git 文本属性（仅 `*.gitattributes text`、`.txt diff`，无 export-ignore） |

组织方式评述：单文件数据源 + 派生制品 + 每语言一个薄封装的"星型"结构。`blns.txt` 是唯一事实源，其余全部应由脚本再生成；README 的贡献规则（"adding or removing a string please update all files"，`README.md` Contributions）就是为维持这个不变量而设。

## 5. 核心架构与关键实现

### 5.1 数据管线：txt 是源，其余是制品

生成链路（`scripts/txt_to_json.py`）：读入全部行 → 去掉行尾换行 → 过滤空行与 `#` 注释 → 在列表头部插入一个空字符串 `content.insert(0, "")`（`scripts/txt_to_json.py` 第 19 行，理由见注释 "insert empty string since all are being removed"）→ `json.dump(..., ensure_ascii=False)` 输出。这个"首位空字符串"是个精心细节：空输入本身就是最常见的捣乱输入，但它在 txt 里无法表达（空行会被过滤）。

base64 版本由 `scripts/texttobase64.sh` 生成：非注释行整行 `base64` 编码，注释行原样保留，因此 `blns.base64.txt` 仍可按小节阅读。

### 5.2 分类体系与条目规模

`blns.txt` 共 **516 条数据行**（`grep -cv "^#\|^$"`），被 **35 个命名分类**的注释段划分（统计自 `blns.txt` 全部 `#` 标题行）。规模分布极不均匀：

- **最大段：Script Injection（行 341-581，236 条，占全清单近半）**——XSS fuzz 语料，覆盖事件处理器、`javascript:` 伪协议、控制字节分隔的标签变体、mXSS、`<plaintext>`、IE `expression()` 等。
- **Numeric Strings 71 条**（行 25-100）：`08`/`09`（八进制误解析）、`-2147483648/-1`（整数溢出）、`2.2250738585072011e-308`（浮点解析 DoS）、三种本地化千分位写法 `1,000.00` / `1.000,00` / `1'000.00`。
- 其余段多为 3-20 条的精准小段，如 Special Unicode Characters Union 只有 1 条 VMware 全球化团队推荐的"超级字符串" `表ポあA鷗ŒéＢ逍Üßªąñ丂㐀𠀀`（注释逐字符列出码位，行 202-222）。

35 个分类可分为三大族：**Unicode/编码族**（Reserved、Numeric、Special Characters 及其 C0/C1 控制字符/空白/Cf 格式字符/BOM 子段、Unicode Symbols、上下标、引号、双字节、Emoji、Regional Indicator、RTL、Ogham、Trick Unicode、Zalgo、倒排、Unicode font、波斯文字符）；**注入攻击族**（Script/SQL/Server Code/Command(Ruby)/XXE/Unwanted Interpolation/File Inclusion/jinja2、Known CVEs）；**环境特定与人为因素族**（MSDOS 保留名、IRC、Scunthorpe Problem、Human injection、Terminal escape codes、iOS Vulnerabilities）。

### 5.3 每类字符串为什么能引发 bug（技术原理）

- **控制字符与零宽字符**：C0/C1 控制、U+200B 零宽空格、U+FEFF BOM、U+202E RTL Override 等不可见但真实存在——会被复用为内部分隔符（注释原文：'reused for internal delimiters on the theory that they should never appear in input'，`blns.txt` 行 110-115）、导致"看起来一样实际不同"的匹配失败，或像 Twitter 那样直接 500。
- **长度错觉**：双字节字符、全角、Emoji（UTF-16 代理对）、组合附加符号（Zalgo）、数学字母数字（Unicode font）都会让"字节数≠字符数≠UTF-16 单元数"，触发数据库列截断、naive 大小写转换变长（专门设有 "Changing length when lowercased" 段：`Ⱥ`/`Ⱦ` 小写化后 2 字节变 3 字节）、正则与截断逻辑错乱。
- **字形簇与显示层**：Regional Indicator 组合的国旗（`🇺🇸🇷🇺🇸`）在不同字体/平台会被拆分显示；`﷽`（U+FDFD 单码位连字）历史上多次击穿按字节处理的缓冲；`🏳0🌈️` 是 iOS 曾出现的渲染崩溃组合。
- **注入族**：SQL 段的 `' OR 1=1 -- 1`、`1'; DROP TABLE users-- 1` 检验参数化查询；Command Injection (Ruby) 段用反引号、`Kernel.exec`、`%x()`、`@{[system ...]}` 检验 shell 转义；jinja2 段（行 736-742）是模板注入 SSTI：`{% print 'x' * 64 * 1024**3 %}` 触发 MemoryError、`{{ "".__class__.__mro__[2].__subclasses__()[40]("/etc/passwd").read() }}` 任意文件读。
- **格式串与插值**：Unwanted Interpolation 段的 `%s`、`%n`（格式串漏洞可写内存）、`$HOME`、`{0}` 检验"用户输入被误当模板/格式串求值"的路径。
- **已知 CVE 复刻**：Shellshock 的 `() { 0; }; touch /tmp/blns.shellshock1.fail;`（CVE-2014-6271）、Juniper 后门流量签名 `<<< %s(un='%s') = %u`、`+++ATH0`（拨号猫挂断）、`DCC SEND STARTKEYLOGGER 0 0 0`（曾让 Norton IDS 卡死的 IRC 串）——用"副作用文件是否出现"这种可判定的方式设计 payload（`blns.txt` Known CVEs 段）。
- **环境保留字**：Windows 保留设备名 `CON`、`PRN`、`NUL`、`COM1-4`、`LPT1-3` 会让文件创建莫名失败；Scunthorpe Problem 段收录脏话过滤误伤的真实地名/人名（Scunthorpe General Hospital、Dr. Herman I. Libshitz、Tyson Gay、Super Bowl XXX），提醒开发者误杀的代价。
- **终端与人类**：Terminal escape codes 段的 ANSI 转义串自嘲"punish the fools who use cat/type on this file"（行 716）；Human injection 段收录"会让人类世界观动摇"的字符串（如著名的 coma 消息），是清单里唯一的幽默段。

### 5.4 语言封装实现

Python 端（`naughtystrings/__init__.py`，35 行）：`naughty_strings(filepath=FILEPATH)` 打开仓库根的 `blns.txt`，`readlines` → 去尾换行 → 过滤空行与 `#` 注释 → 首部插入空串。与生成脚本同逻辑，保证 Python 读到的顺序与 txt 一致。

Go 端（`naughtystrings/naughtystrings.go`）：包级 `init()` 通过 go-bindata 生成的 `internal/resource.go` 加载 `../blns.json` 与 `../blns.base64.json`，暴露 `Unencoded()` / `Base64Encoded()`；把数据编进二进制意味着 Go 使用方无需在部署物中携带数据文件。唯一测试 `naughtystrings_test.go` 仅断言两个列表非空。

## 6. 构建与运行

本项目无构建产物，以下均为数据消费/再生成命令（来源：README 与各脚本自身）：

```bash
# Python：直接以仓库为包使用（无 PyPI 包，README 未提供 pip 安装方式）
from naughtystrings import naughty_strings
naughty_strings()          # 返回含首位空串的完整列表

# Go（仓库内子模块，无第三方依赖）
go test ./naughtystrings/...      # 最小测试：断言列表非空

# 再生成派生文件（在 scripts/ 与 naughtystrings/ 目录内执行，路径为相对硬编码）
python scripts/txt_to_json.py     # blns.txt -> blns.json
bash scripts/texttobase64.sh > blns.base64.txt
make -C naughtystrings naughtystrings/internal/resource.go   # 需要 go-bindata

# npm 生态消费：package.json main=blns.json，可直接 require('big-list-of-naughty-strings')
```

注意：`txt_to_json.py` 与 `texttobase64.sh` 用 `../blns.txt` 相对路径，必须在 `scripts/` 目录内运行；Go 包的 go-bindata 路径同样写死 `../`。

## 7. 内容质量与数据一致性

**质量优点**：每个分类注释段不只给名字，还解释"为什么会出问题"并给出处（如 Zalgo 段标注 eeemo.net、lowercase 变长段标注推文链接、Shellshock 段标注 HN 讨论），使清单本身即是一份微型的字符串故障学教程（`blns.txt` 各注释段）。贡献规则同样服务质量：拒绝 255+ 长串保可读性、拒绝 EICAR 测试毒株防杀软误报、拒绝 U+0000 防 GitHub 把文件判成二进制（`README.md` Contributions）。

**实测发现的一致性缺陷（本报告核心观察）**：`blns.json` 与 `blns.base64.json` 是 2019-01-30 的旧快照（go-bindata 内嵌的 `bindataFileInfo modTime = 1548804227`，`naughtystrings/internal/resource.go`），而 master 最后一次提交（2021-04-15 "Index XSS strings"，PR #226）扩充改写了 Script Injection 段后**没有重新生成任何派生文件**。实测证据：`blns.txt` 中的 XSS 变体编号 `alert(118)`、`alert(134)`、`alert(145)` 在 `blns.json` 中均为旧版 `alert(1)`；条目数三方不一致——txt 516 数据行、`blns.json` 515 条、`blns.base64.json` 676 条。即 README 自己定下的 "update all files" 规则在最后一次 PR 中未被执行，且无 CI 兜底。

**固有的表达限制**：txt 格式无法表达 NUL、LF、CR 本身（注释明示，`blns.txt` 行 127-128），多行 payload 也放不进去——这是"手工可读性优先"设计的必然代价。

## 8. 社区生态与维护状态

- **传播史**：README 收录 2015-2018 年间 7 次 HN/Reddit 热帖，其中 2015-06 Show HN（item 10035008）为起点，2017-01 与 2018-11 两度再登 HN 首页——星数的三次跳涨与这三轮传播对应（`README.md` Social Media Discussions）。
- **维护状态**：2015-08 创建，提交节奏为 2015-2018 活跃、2019-2021 渐停；master 最后提交 2021-04-17（PR #226 by caasi），`gh api` 显示 pushed_at 2024-04-18（应为其他分支或元数据操作），repo 未归档、未声明停止维护，但实质处于"完成态停更"。open issues 109 个长期未处理（`gh api repos/...`，2026-11-05）。
- **移植生态**：README Library/Packages 表列出社区维护的第三方封装：npm `blns`、npm `big-list-of-naughty-strings`、.NET `SimonCropp/NaughtyStrings`、PHP `mattsparks/blns-php`、C++ `eliabieri/blnscpp`；仓库自身又带了 Python 与 Go 封装（Go 包为社区贡献）。数据本体被广泛内嵌进各类 fuzz 工具与测试框架 fixture。
- **贡献者结构**：约 2,147 forks（`gh api`）；提交以作者 Max Woolf 为主、大量单提交贡献者通过 PR 进条目（commits API 中可见 doroshenko、tryauuum、caasi 等）。近 5 年基本无新条目进入。

## 9. 亮点与局限

**亮点**

1. 极简而正确的发布形态：一个 UTF-8 文本文件作为唯一事实源，机器可读格式全部可由脚本派生，任何语言 10 行代码即可消费。
2. 分类即教学：35 个分类注释解释故障机理与出处，读 `blns.txt` 注释等于上一堂浓缩的"字符串如何搞挂系统"课。
3. 条目有真实谱系：来自 XSS fuzz 语料、真实 CVE（Shellshock、Juniper 后门）、历史事故（iOS 文字炸弹、Norton 卡死），不是拍脑袋的乱码。
4. 覆盖面兼顾"非恶意"场景：Scunthorpe Problem（脏话过滤误杀）、本地化数字、Windows 保留名等提醒开发者：不是只有攻击者才会输入奇怪的东西。
5. 零依赖、零维护成本的纯数据资产，MIT 许可， embedding 进任何产品无负担。

**局限**

1. 派生文件自 2019 年起与主文件不同步（见第 7 章），信任 `blns.json` 的使用者拿到的是过时语料。
2. 无 CI、无再生成校验，"更新所有文件"靠人肉自觉，已实际失守一次。
3. 文件格式天然不含 NUL/LF/CR 与多行 payload，长串被贡献规则禁止，对某些 fuzz 场景覆盖不足。
4. 2021 年后停更：新 Unicode 版本字符、近年框架的模板注入变体、LLM 时代的提示注入类输入均未收录。

## 10. 总结评价

这是那种"一个文件改写一个行业习惯"的项目：它本身几乎没有代码，却靠一份精心分类、条条有出处的字符串清单，把"怪输入测试"从老 QA 的个人手艺变成了可复制、可引用的公共语料，并在停更多年后仍是测试工程里的默认标配。对 QA/测试工程师，它是拿来即用的输入黑名单；对安全工程师，它是一份轻量级、良性的 fuzz 种子集（且仓库明确声明不能替代渗透测试）；对想理解 Unicode 与注入问题原理的开发者，逐段读 `blns.txt` 的注释是极好的教材；而对想借鉴"数据型开源项目"如何设计事实源与派生管线的人，它同时提供了正例（星型生成结构、首位空串这类细节）和反例（派生文件失同步、无 CI 校验）。
