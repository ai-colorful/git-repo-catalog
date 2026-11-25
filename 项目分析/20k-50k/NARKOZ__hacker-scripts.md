# NARKOZ/hacker-scripts 项目分析

> 星数 49853（快照 2026-10-24）｜主语言 JavaScript（实际为 17 种语言的多语言合集）｜档位 20k-50k｜分析日期 2026-11-04

## 1. 项目速览

GitHub 上最著名的"幽默型"仓库之一：一整套把程序员日常开挂到极致的搞笑自动化脚本——给老婆发加班短信、自动敷衍 DBA、宿醉请假、远程遥控咖啡机——基于一段在俄语笑话站流传的"真实故事"改编，随后被社区用十几种语言重写。

| 事实 | 值 | 出处 |
|---|---|---|
| 名称 | NARKOZ/hacker-scripts（描述："Based on a true story"） | GitHub API `repos/NARKOZ/hacker-scripts` |
| 星数 | 49853（快照）；分析时 API 实测 49858，6625 forks | 同上 |
| 创建时间 | 2015-11-21 | 同上 |
| 最近提交 | 2022-08-07（merge PR #221）；最近 push 2023-10-23 | GitHub API commits |
| 贡献者 | 47 人 | GitHub API contributors |
| 许可证 | README 声明 WTFPL，但仓库无 LICENSE 文件，GitHub license 字段为 null | `README.md` 末行、GitHub API |
| 规模 | 54 个文件、约 2471 行，根目录 + 15 个语言移植目录 | 本地统计 `src/` |

## 2. 项目是什么，解决什么问题

这不是一个"解决工程问题"的仓库，而是一个**内容档案 + 社区游乐场**。它复刻了一段出自俄语笑话站 bash.im（quote 436725，原链已失效，README 附 Archive.org 备份链接）的段子，后经 Jitbit 博客以英文转述（链接见 `README.md`）：一位离职构建工程师留下四个 cron 脚本，把"任何超过 90 秒的事都自动化"的信条推到荒诞的极致。作者 NARKOZ 于 2015 年 11 月将其实现开源，一夜爆红（`README.md` 开头即引用原故事全文）。

四个脚本各对应一个梗（出处：`README.md` 叙述 + 根目录脚本实现）：

1. **smack-my-bitch-up.sh**——cron 设定工作日 21:20，若服务器上仍有自己用户名的 SSH 会话（`who | grep -wq $USER`），就通过 Twilio API 给老婆发"今晚要加班了"，理由从数组中随机抽取：`'Working hard' / 'Gotta ship this feature' / 'Someone fucked the system again'`（`smack-my-bitch-up.sh:12-20`）。
2. **kumar_asshole.rb**——每 10 分钟扫描 Gmail，若客户 DBA Kumar 发来未读邮件且命中关键词 `sorry|help|wrong`、正文中含 `*_staging` 库名，就用前一天的备份文件回滚 staging 数据库（`gunzip -c … | psql …`），打上 "Database fixes" 标签并回复"I've fixed it, please be careful next time"（`kumar_asshole.rb:25-38`）。
3. **hangover.sh**——工作日 8:45 检查，若服务器上**没有**自己的交互会话（即没来上班），随机借口向老板发短信请假：`'Locked out' / 'Pipes broke' / 'Food poisoning' / 'Not feeling well'`（`hangover.sh:12-21`）。
4. **fucking_coffee.rb**——先精确 `sleep 17` 秒（正是从工位走到咖啡机的时间），再 telnet 到咖啡机 `10.10.42.42`（密码 `1234`）发送 `sys brew`，等 24 秒后 `sys pour`（`fucking_coffee.rb:6-20`）。

仓库的价值不在实用性——脚本里的 IP、邮箱、手机号全是占位符——而在两点：其一，它把"程序员过度自动化"这个梗固化成了可运行、可 fork、可翻译的实体；其二，它成了一个开源现象样本：README 有官方中文翻译（`README.zh-CN.md`），社区自发把四个脚本重写成 15 种语言，PR 本身成为参与仪式。它能涨到近 5 万星，靠的是共鸣与传播，而非功能。

## 3. 技术栈与依赖

- **根目录权威实现**：Bash（`smack-my-bitch-up.sh`、`hangover.sh`，仅依赖 `curl` + Twilio HTTP API）与 Ruby（`fucking_coffee.rb` 依赖 `net/telnet` 标准库；`kumar_asshole.rb` 依赖 `dotenv`、`gmail` 两个 gem，见 `README.md` Usage 一节）。
- **语言移植目录**（15 个）：C#（`CSharp/`）、R（`R/`）、Clojure、CoffeeScript、Go、Groovy、Java、Kotlin、Node.js、Perl、PHP、PowerShell、Python 2、Python 3、Scala。
- 各移植的代表性依赖（均为真实文件）：
  - `php/composer.json`：`bestnetwork/telnet`、`vlucas/phpdotenv`、`twilio/sdk`。
  - `go/fucking-coffee.go:10-12`：`codeskyblue/go-sh`（shell 管道）+ `google/goexpect`（telnet 交互）。
  - `nodejs/kumar_asshole.js:17-26`：`imap` + `mailparser` 收件、`nodemailer` 发件（作者注释承认 IMAP 只能收、nodemailer 只能发，故拼合使用）。
  - `python3/requirements.txt`：`dotenv`、`twilio`、`yagmail`、`git+https://github.com/charlierguo/gmail`。
- 无构建系统、无 lockfile、无框架；`python3/hackerutils.py` 是唯一跨脚本共享的工具模块（dotenv 读取、子进程封装、日志路径，共 24 行）。

## 4. 目录结构与代码组织

| 目录 | 职责 | 规模（文件数/行数） |
|---|---|---|
| `/`（根） | 权威实现：4 个 .sh + 4 个 .rb、README 中英双语、`.env.example` | 12 文件 |
| `nodejs/` | Node 移植，含变体 `fucking_coffee_yo_server.js`（用外部 "yo server" 服务喊人来取咖啡） | 5 文件 / 350 行 |
| `perl/` | Perl 移植 | 4 文件 / 282 行 |
| `kotlin/`、`java/` | JVM 系移植（各覆盖 4 个脚本） | 4 文件 / ~203 行 |
| `python3/`、`python/` | Python 3 版含共享模块 `hackerutils.py`；Python 2 版为历史遗留 | 6 / 4 文件 |
| `powershell/` | PowerShell 移植（`.psm1` 模块 + `.ps1`） | 3 文件 / 163 行 |
| `clojure/`、`go/` | Lisp 系与 Go 系移植 | 4 文件 / 3 文件 |
| `CSharp/`、`R/`、`php/` | 各 2-4 文件的小型移植 | 2-4 文件 |
| `coffee/`、`groovy/`、`scala/` | 单文件移植（仅覆盖 coffee 一个脚本） | 1 文件 |

组织方式是扁平的"按语言分目录"：根目录是故事原型（Ruby/Bash），每个语言一个平级目录，目录内文件名对齐脚本名。没有 monorepo 工具、没有共享 CI，每个移植自包含，读哪个目录都独立成立。这种结构本身就是社区 PR 文化的产物——谁的 PR 合入，就多一个目录。

## 5. 核心架构与关键实现

四个脚本本身极短，值得记录的是三处实现细节：

**(1) 触发条件的镜像设计**（出处：`smack-my-bitch-up.sh:4` 与 `hangover.sh:4`）。两个脚本用同一个 `who | grep -wq $USER` 判断"我是否在服务器上"，只是取反方向：一个"在才发"（加班通知），一个"不在才发"（请假借口）。这个对称性正是段子的笑点骨架：

```bash
# smack-my-bitch-up.sh:4
if ! who | grep -wq $USER; then
  exit
fi
```

**(2) kumar_asshole.rb 的"票据式"DBA 应付流水线**（出处：`kumar_asshole.rb:25-38`）。用正则做最小语义判断——关键词 `/sorry|help|wrong/i` 确认是"他又搞砸了"，库名 `/\S+_staging/` 从邮件正文里抽取要回滚的库——然后拼命令行 `gunzip -c #{backup_file} | psql #{db_name}` 执行还原。备份文件名以昨日日期硬编码校验，找不到直接 `abort`。整条链路是"读邮件 → 正则分类 → 命令行运维 → 回邮件"的闭环，浓缩了当年 DBA 工单文化的讽刺。

**(3) fucking_coffee.rb 的"人体工学时序"**（出处：`fucking_coffee.rb:11-20`）。`delay_before_brew = 17` 与 `delay = 24` 两个魔法数字即故事里"从工位走到咖啡机的步行时间"：先睡 17 秒再启动 telnet 会话（人走向机器），`sys brew` 后再睡 24 秒发送 `sys pour`（机器中杯半因拿铁的出品时间），人到达时咖啡刚好倒好。用 `Net::Telnet` 的 `Match => /Password: /` 做朴素_expect_ 交互。Go 移植（`go/fucking-coffee.go:35-49`）同样保留了这两个数字并用 goexpect 复刻了 telnet 握手序列。

值得注意的不一致：回信文案在各实现中不统一——Ruby 版是 `"No problem. I've fixed it. … Please be careful next time."`（`kumar_asshole.rb:21`），Node 版是 `"No worries mate, be careful next time"`（`nodejs/kumar_asshole.js:15`），Kumar 的邮箱占位符也各不相同（`kumar.a@example.com` vs `kumar.asshole@example.com`）。移植均为社区 PR，无统一规范约束，属于预期内的漂移。

## 6. 构建与运行

无构建、无编译。运行方式来自 `README.md` Usage/Cron 两节：

```sh
# 环境变量（或复制 .env.example 为 .env，配合 dotenv）
TWILIO_ACCOUNT_SID=ACxxxx   TWILIO_AUTH_TOKEN=yyyy   # 发短信用
GMAIL_USERNAME=...          GMAIL_PASSWORD=...        # kumar 脚本用

# Ruby 版依赖
gem install dotenv twilio-ruby gmail

# README 中的 cron 示例（节选）
20 21 * * 1-5  /path/to/smack-my-bitch-up.sh
45 8  * * 1-5  /path/to/hangover.sh
*/10 * * * *   /path/to/kumar-asshole.sh
0 9-18 * * 1-5 /path/to/fucking-coffee.sh
```

脚本内手机号（`+xxx`）、咖啡机 IP（`10.10.42.42`）、邮箱均为占位/示例值，实际运行需自行替换；`kumar-asshole.sh` 与 `fucking-coffee.sh` 是 `exec` 到对应 Ruby 脚本的薄包装（`kumar-asshole.sh:6`、`fucking-coffee.sh:6`）。

## 7. 测试与工程质量（内容档案型观察）

- **测试**：无任何测试文件与 CI 配置（tarball 全清单中无 `.github/` 目录、无测试框架痕迹）。对这类仓库"测试"不适用，其质量观是"能讲清梗即正确"。
- **实现质量**：以玩具脚本的标准看相当克制——bash 版有 `set -e`、curl 错误回显（`smack-my-bitch-up.sh:27-32`）；Python 3 版抽象出共享工具模块；各移植保留了原版语义（如 17/24 秒时序在 Go/PHP/PowerShell 版中均可对照）。硬编码密码 `1234`、明文 GMAIL 密码环境变量等"坏实践"是段子的组成部分，社区 issue 中曾有人提议清理（如已关闭的 issue "Clean up coffee script and remove passwords"）。
- **文档**：README 即内容本体（故事全文 + 每个 git 链接 + cron 配置），且有维护良好的中文版 `README.zh-CN.md`；许可声明只有末行一句 "Code is released under WTFPL"，没有标准 LICENSE 文件。

## 8. 社区与维护状态

- **生命周期**：2015-11-21 创建后即爆红（当日前后 Hacker News 热帖带动，属公认传播路径，具体榜单位置未确认）；功能层面在 2015-2016 年即"完成"，此后只有零星 PR 合入。
- **提交节奏**：GitHub API commits 显示近 10 年仅约 221 个 PR 编号、47 位贡献者；最近一次实质提交是 2022-08-07 的 README 链接修复（merge PR #221，替换失效的 bash.im 链接为 Archive.org），最近 push 为 2023-10-23。仓库处于**休眠但稳定**状态——内容已定稿，无需维护。
- **Issue 文化**（采样 `issues?state=all`）：issue 几乎是段子接龙现场——"#265 Was hungover, now the office is flooded with coffee"（宿醉者报障咖啡机淹了办公室）、"#257/#256 Banning player …/#256 Admin abuse"（游戏服务器玩家误入发帖）、"#264/#262 Hacker"（纯打卡）。真实工程性 issue/PR 稀少且很快被处理，如已关闭的 "Clean up coffee script and remove passwords"、开放中的 "#263 Add Twilio account SID to hangover.rb"。这是星数与实用价值彻底脱钩的典型样本：fork 6625 次多为围观或练手移植。
- **Release**：无任何 release/tag（GitHub 元数据未提供 release 记录，仓库也无版本需求）。

## 9. 亮点与风险

**亮点**

1. 开源史上最高效的"故事驱动仓库"之一：一段笑话 + 四个短脚本 = 49853 星、6625 fork，单位代码的传播效率极高。
2. 四脚本构成完整自洽的讽刺叙事（加班短信 ↔ 请假借口 ↔ 应付 DBA ↔ 咖啡机时序），README、脚本、cron 配置三者互相印证，档案价值完整。
3. 17 种语言的重写使它成为一套意外的"跨语言代码风格对照教材"——同一个 telnet 握手在 Ruby（`Net::Telnet`）、Go（`goexpect`）、PowerShell（`.psm1`）中的写法差异一目了然。
4. 中英双语 README、Archive.org 原链备份，内容保存意识好。

**风险 / 局限**

1. 无 LICENSE 文件，仅 README 一句 WTFPL，严格意义上复用存在许可瑕疵。
2. 脚本含硬编码密码（`1234`）、明文环境变量凭据、SQL 命令拼接——不可用于生产，也只能当段子看。
3. `python/`（Python 2）目录与各移植依赖（如 `twilio/sdk ^4.6`）早已过时，无法按原样跑通；移植间行为漂移（回信文案、邮箱占位符不一致）。
4. 已休眠：最后实质提交 2022-08，issue 区近于玩梗留言板，不会有功能演进。

## 10. 总结评价

hacker-scripts 是 GitHub 上"星数与工程价值无关"的最极端案例之一：约 2500 行从未真正运行过的占位符脚本，凭一个俄罗斯笑话的英译本和四段能引发全体程序员会心一笑的"过度自动化"幻想，跻身全站前 50 名。它更像一件开源社区的民俗文物——记录了 2015 年那一波"故事型仓库"爆红模式，也用 17 种语言的重写 PR 展示了社区参与如何变成一种仪式。适合两类人：想研究 GitHub 传播现象、meme 仓库生命周期的观察者；以及想快速对比多门脚本语言风格的初学者。想抄进生产环境的人请绕行。
