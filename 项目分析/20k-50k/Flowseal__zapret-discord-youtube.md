# Flowseal/zapret-discord-youtube 项目分析

> 星数 36515（快照 2026-10-24）｜主语言 Batchfile｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：bol-van/zapret（winws.exe）的 Windows 预配置发行包——把上游 zapret 的可执行文件与 WinDivert 组件直接打包，再用约 20 个参数预设（"策略"）批处理脚本和一个 1300 行的服务管理器，面向 Discord/YouTube 访问受阻地区（以俄罗斯为主）的用户提供开箱即用的配置选择。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | Flowseal/zapret-discord-youtube | gh api repos/... |
| 星数 / Fork | 36515 / 2758 | gh api（2026-11-21 快照） |
| 主语言 | Batchfile（22 个策略 .bat + service.bat） | tarball 文件清单 |
| 许可证 | MIT 双版权（bol-van 2016-2026 + Flowseal 2024-2026），附 WinDivert LGPLv3/GPLv2 说明；GitHub 识别为 NOASSERTION | LICENSE.txt |
| 创建 / 最近 push | 2024-10-08 / 2026-11-20 | gh api |
| 最新 release | 1.10.4（2026-10-20，zip/rar/tar.gz 三种包 + SHA256） | gh api releases/latest；.service/version.txt |
| 贡献者 | 约 54 人（contributors API 分页推断） | gh api |
| 开放 issue | 560 | gh api |
| 仓库实质规模 | 79 个 tar 条目；脚本约 2900 行（.bat 约 1870 行 + .ps1 1020 行）；二进制约 3.4 MB；数据列表约 60 万行 CIDR/域名 | tar -tvzf 统计 |

## 2. 项目是什么，解决什么问题

仓库本体不是一个新的 DPI 处理引擎，而是一份"发行/配置层"工程。真正的核心工具是 bol-van/zapret 的 `winws.exe`（约 204 KB，Cygwin 环境编译，配套 `cygwin1.dll`）与流量拦截组件 WinDivert（`WinDivert.dll` 47 KB + `WinDivert64.sys` 94 KB）。README 明确声明 `bin/` 内所有可执行与系统文件取自 bol-van/zapret-win-bundle 的 zapret-winws 目录与 bol-van/zapret 的 releases，并提示用户可用哈希/校验和核对（README.md "IMPORTANT" 区块）。也就是说，本仓库把"上游二进制 + 参数预设 + 服务化管理 + 社区支持自动化"组装成了一个对小白可用的整体，README 自我定位为 bol-van/zapret-win-bundle 的替代发行（README.md 顶部链接）。

仓库解决的核心矛盾是：zapret 上游只提供工具与参数文档，普通用户面对几十个 `--dpi-desync-*` 参数无从下手；而目标站点（Discord、YouTube）被阻断的具体手段因 ISP、地区、时间而异，单一参数组合不可能普适。本仓库的答案是**预设矩阵**：22 个 `general*.bat`，每个是一个完整可运行的参数组合，用户逐一尝试、用内置测试器筛选，再把有效的那一个安装为 Windows 服务。README 直接承认"策略会随时间失效"（被识别后失效），并建议用户基于现有参数自改（README.md "Обход не работает / перестал работать" 节）。

星数增长背景（客观陈述）：仓库创建于 2024-10-08（gh api），恰逢俄罗斯对 YouTube 降速、对 Discord 实施阻断的时期；README、issue 模板、自动回复全部使用俄语，issue 模板还请用户填写"运营商与地区"（.github/ISSUE_TEMPLATE/bug_report.yml），域名与 IP 列表也以俄区服务为排除对象（lists/list-exclude.txt 内 yandex/vk/sber/ozon/gosuslugi 等）。13 个月从 0 增长到 36.5k 星，反映的是该时段俄语用户对这类工具的强需求与本仓库"开箱即用"形态的传播效应。项目另有一个姊妹仓库 Flowseal/tg-ws-proxy 处理 Telegram（README.md 顶部 NEW 链接）。

需要说明的边界：本仓库按 README 的声明只分发上游二进制，不改 winws 源码；仓库内没有任何 C/C++ 源码与构建脚本（bin/ 之外无 .c/.h/Makefile），分析与使用边界都由 LICENSE.txt 的三方声明约束（见第 3、9 节）。

## 3. 技术栈与依赖

- **Batchfile**：`service.bat`（1311 行，38,954 B）+ 22 个策略脚本（各 25 行，2.1-4.4 KB）。大量使用 `sc`/`netsh`/`reg`/`tasklist` 等 Windows 原生命令，颜色输出借助内嵌 `powershell -NoProfile -Command "Write-Host ... -ForegroundColor ..."` 单行调用（service.bat 底部 :PrintGreen/:PrintRed/:PrintYellow）。
- **PowerShell 5.x（Windows 自带）**：`utils/test zapret.ps1`（1020 行，41,693 B）为策略测试器；service.bat 中的 JSON 级别解析（列目录、取哈希、查 DoH 注册表）也都以单行 PowerShell 完成。运行测试要求 PowerShell 3.0+（service.bat :run_tests 有版本门槛检查）。
- **上游二进制（vendored）**：`bin/winws.exe`（203,776 B）、`bin/cygwin1.dll`（2,954,293 B）、`bin/WinDivert.dll`（47,616 B）、`bin/WinDivert64.sys`（94,144 B）——winws.exe 依赖 Cygwin 运行时与 WinDivert 驱动，二者随包分发。
- **静态包样本资产**：`bin/*.bin` 共 23 个（100 B - 1,357 B），按文件名分为 `tls_clienthello_*`（10 个）、`quic_initial_*`（9 个）、`stun/stun2`（2 个）、`ACTIVE_DISCORD_UDP/ACTIVE_GAME_UDP`（2 个当前生效副本）。策略脚本与 service.bat 的 "Replace active fakes" 功能按文件名与 SHA256 引用它们（service.bat :replace_active_fakes 用 Get-FileHash 比对定位当前激活文件）。
- **运行时网络依赖（仅本项目自己的数据文件）**：更新检查读 `https://raw.githubusercontent.com/Flowseal/zapret-discord-youtube/main/.service/version.txt`；IPSet 列表与 hosts 更新同样只从本仓库 raw 地址下载（service.bat :service_check_updates / :ipset_update / :hosts_update）。上游 zapret 二进制不按需下载，完全嵌入。
- **无包管理器/无构建系统**：没有 package.json/go.mod 等；发布由 GitHub Actions 直接打包目录（.github/workflows/release.yml）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 | 依据 |
|---|---|---|
| `bin/` | 29 个文件：winws.exe、cygwin1.dll、WinDivert.dll/sys + 23 个静态包样本 .bin | tar 清单与大小表 |
| `lists/` | 策略用域名/IP 清单：list-general.txt（59 行，Discord+Cloudflare+DoH 域名）、list-google.txt（22 行，YouTube/Google）、list-exclude.txt（129 行排除域名）、ipset-exclude.txt（11 行保留网段）、ipset-all.txt（18 B 哨兵占位）| tar 内容 |
| `.service/` | 发布侧数据：ipset-service.txt（577,240 B，33,336 行 CIDR，真实 IP 列表）、hosts（82 行静态解析：GitHub raw + Telegram Web + discord.com）、version.txt（"1.10.4"）| tar 内容 |
| `general*.bat` × 22 | 策略预设（ALT / FAKE TLS AUTO / SIMPLE FAKE / EXP 等族）| tar 清单 |
| `service.bat` | 服务管理器（菜单 12 项）| 文件本体 |
| `utils/` | test zapret.ps1（测试器）、targets.txt（16 个测试端点）、check_updates.enabled（开关标记文件）| 文件本体 |
| `.github/` | workflows：release（手动触发打包发布）、stale（issue 保活/关闭）、comment（新 issue 自动回复俄语排查清单）；github-bot.yml（13 名 trusted_users）；ISSUE_TEMPLATE/bug_report.yml（俄语模板，必填版本）| 文件本体 |
| `LICENSE.txt`、`README.md` | MIT 三方声明；22.9 KB 俄语长文 README（含常见问题/防假冒声明/杀软提示）| 文件本体 |
| `.gitattributes` / `.gitignore` | 仅 EOL 规则（bat/ps1/txt 用 CRLF），无 export-ignore；.gitignore 忽略用户自建清单与测试结果 | 文件本体 |

组织方式：扁平单目录发行包结构（发布 zip 即仓库内容去掉 .service/.github）。仓库内部呈现清晰的三层：**预设层**（general*.bat，纯数据化参数）— **管理层**（service.bat + test zapret.ps1）— **资产层**（bin 二进制与 lists 数据）。没有子模块、没有多包；策略与管理的耦合通过 `call service.bat <子命令>` 单向完成。

## 5. 核心架构与关键实现

### 5.1 策略预设层：一个骨架，22 份参数变体

每个 `general*.bat` 都是同一个 25 行骨架：先 `call service.bat` 的四个子命令（`status_zapret` 检查服务冲突、`check_updates` 静默更新检查、`load_game_filter` 注入端口范围变量、`load_user_lists` 生成用户清单文件），再以 `start /min winws.exe ...` 启动核心。`general.bat` 的启动参数由 `--new` 分隔成 9 个串行段（profile），按目标分流——这就是 Discord/YouTube 分离配置策略的实际形态：

```bat
--filter-udp=443  --hostlist=...list-general.txt ... --dpi-desync=fake ... --new
--filter-udp=19294-19344,50000-50100 --filter-l7=discord,stun ... --new
--filter-tcp=2053,...  --hostlist-domains=discord.media ... --new
--filter-tcp=443  --hostlist=...list-google.txt --ip-id=zero ... --new
--filter-tcp=80,443 --hostlist=...list-general.txt ... --new
（后接 ipset 段 ×2 与 Game Filter 段 ×2）
```

（出处：zapret-discord-youtube-HEAD/general.bat；下同，仅节选）

各段分别面向：QUIC/UDP-443 通用列表、Discord 语音 UDP（l7 协议识别 + 专用端口段）、discord.media 专用 TCP 端口、YouTube/Google 域名（list-google.txt）、通用 TCP 80/443、ipset IP 列表、游戏端口范围。列表项 user 后缀文件由 :load_user_lists 首次生成占位（service.bat），用户追加内容不被 git 覆盖（.gitignore）。

22 个变体之间只改各段的 desync 机制参数（如 multisplit、fake,fakedsplit、fake,multidisorder 三族，配合 repeats 6-12、cutoff n2-n4、fooling=ts/badseq 的取值差异），对比 general.bat 与 general (ALT).bat、general (FAKE TLS AUTO).bat 可见除参数外逐行相同。设计上等价于"一份配置模板 × 一张参数组合表"，用文件命名（ALT/FAKE/SIMPLE/EXP 后缀）代替了命令行 flag，使 Install Service 可以直接按文件名安装任意预设。

### 5.2 service.bat：纯批处理实现的服务管理器（1311 行）

职责覆盖 12 项菜单功能（service.bat :menu）。最有工程含量的三处：

**① 把策略 bat 解析成服务启动参数。** Install Service 读取用户选中的 general*.bat，逐行截取 `winws.exe` 之后的参数，处理续行符 `^`、引号路径、以及需要值的参数列表（`set "args_with_value=sni host altorder"`），用状态机变量 `mergeargs`（0-3）区分"新参数/值参数/合并值"，最终 `sc create zapret binPath= "\"...winws.exe\" <args>" start= auto` 注册服务，并把所选策略名写入注册表值 `HKLM\...\Services\zapret\zapret-discord-youtube` 以便菜单回显（service.bat :service_install、:get_strategy_name）。这是在没有真正 shell 解析器的 cmd 环境里手写的参数解析器。

**② IPSet 三态切换。** 用哨兵值实现三态：`lists/ipset-all.txt` 含哨兵网段 `203.0.113.113/32` = none（仅占位，仓库初始态即 18 B 哨兵文件），文件为空 = any（任意 IP 均入过滤），其余 = loaded（真实 CIDR 列表生效）。none↔loaded 靠与 `ipset-all.txt.backup`（577 KB）互改名实现；真实列表放在 `.service/ipset-service.txt`，"Update IPSet List" 用 curl（按 libcurl 版本区分 --ssl-no-revoke 参数）或 PowerShell Invoke-WebRequest 下载覆盖（service.bat :ipset_switch_status/:ipset_switch/:ipset_update）。测试器也复用同一哨兵逻辑（test zapret.ps1 :Get-IpsetStatus）。

**③ 冲突诊断清单。** Run Diagnostics 依次检查：Base Filtering Engine 服务、系统代理、TCP timestamps（未启用则 netsh 自动打开）、已知冲突软件（Adguard、Killer 网卡服务、Intel Connectivity、Check Point、SmartByte、VPN"Happ"）、安装路径含西里尔字母、OneDrive 目录、WinDivert 孤儿服务（winws 未运行但 WinDivert 在跑时自动 stop/delete，并顺带处理 GoodbyeDPI 等同类工具服务）、hosts 文件残留 YouTube 条目，最后可选清理四个版本的 Discord 缓存目录（service.bat :service_diagnostics/:clear_discord_cache）。

其余：Game Filter 把 TCP/UDP 端口范围存入 `utils/game_filter.enabled`（mode/tcp/udp 三键），带正则与 65535 上限校验（:gf_validate_item）；更新检查对比本地 `LOCAL_VERSION=1.10.4` 与 raw 上的 version.txt；hosts 更新只做"首行/末行是否已存在"的比对，需要改动时打开记事本让用户手动合并（不自动写系统 hosts）。

### 5.3 utils/test zapret.ps1：策略测试器（1020 行）

对每个策略配置：备份 ipset 状态 → 以所测参数启动 winws → `Wait-WinwsReady`（5 秒轮询进程出现）→ 按 `utils/targets.txt` 的 16 个端点做 HTTP/TLS + ping 探测 → 结束后恢复原状态；顶部注册 `trap` 保证中断时还原 ipset 备份与标记文件（test zapret.ps1 开头）。两种测试类型：`Standard tests (HTTP/ping)` 与 `DPI checkers`（targets.txt 里 Cloudflare/Amazon 等），配置可选 All configs 跑全部 bat。结果写入 `utils/test results`（.gitignore 忽略）。脚本对非交互宿主做了兼容（Wait-AnyKey 在 stdin 重定向下退化为 Read-Host）。

### 5.4 发布与支持自动化（.github/）

`release.yml` 手动触发，先校验输入 tag 与 `.service/version.txt` 一致，再拷贝 bin/lists/utils/*.bat 组包，产出 zip/rar/tar.gz 并打印 SHA256，用 softprops/action-gh-release 发布——发布物与仓库内容的对应关系完全机械可验证。`comment.yml` 对每个新 issue 自动贴一段长俄语排查清单（README/诊断/更新 hosts/换策略/游戏问题转 Discussions），是 560 个开放 issue 高流量下的分流机制。`stale.yml` 60 天置旧、7 天后自动关闭，豁免 pinned/games 标签。`github-bot.yml` 列出 13 个可信用户（Flowseal、ControlRe、ekungurov 等），配合 issues 支持。

## 6. 构建与运行

- **无构建环节**。上游 zapret/WinDivert 二进制以成品形态提交在 bin/，仓库不做任何编译；release 产物由 CI 从仓库目录直接打包（.github/workflows/release.yml 的 cp/zip/rar/tar 步骤）。
- 脚本层的"解释环境"就是 Windows 自带组件：cmd 解释 .bat、Windows PowerShell 解释 .ps1（要求 3.0+，service.bat :run_tests 有检查）、curl.exe 或 Invoke-WebRequest 做下载。测试器结果目录与用户清单文件按需创建（.gitignore 与 :load_user_lists）。
- 版本一致性由三处共同锁定：`.service/version.txt` = `service.bat` 首行 `LOCAL_VERSION` = 最新 release tag `1.10.4`（三者均已核对一致）。
- （遵照任务边界，此处不展开任何运行与使用指导；本节仅描述工程构成。）

## 7. 测试与工程质量

- **测试设施是产品的核心卖点而非附属品**：1020 行测试器 + targets.txt 端点表 + ipset 状态备份/还原 + trap 兜底，构成一个可复现的策略筛选流程；测试结果文件化（utils/test results），issue 自动回复要求用户附上测试结果与诊断截图（comment.yml）。
- **CI 只有运维类 workflow，没有代码测试**：release/stale/comment 三条流水线均不含 lint 或脚本语法检查；批处理参数解析器（:service_install）逻辑复杂、依赖转义规则，属于全仓库最脆弱的一段，靠 :check_extracted/:check_command 的前置检查（bin 目录存在、chcp/find/findstr/netsh/powershell 在 PATH）降低失败面。
- 工程卫生良好：EOL 统一 CRLF（.gitattributes）、颜色输出统一封装、错误信息带 issue 编号引用、PowerShell 调用全部带 `-NoProfile`；哨兵值（203.0.113.113/32，TEST-NET-3 文档地址段）复用在批处理与 PowerShell 两侧，保持一致。
- 文档密度高：22.9 KB README 覆盖故障树；俄语注释与界面降低目标用户门槛。

## 8. 社区与维护状态

- 活跃度：2026-11-20 仍有 push（README 更新 #19047），2026-11-04 更新 ipset 列表（#18572），2026-10-20 集中合入列表更新、诊断修复（happ 检测 #17740）与 1.10.4 发版（gh api commits）。列表类文件（ipset/hosts/exclude）的高频更新是该仓库的实际维护主体。
- 发布节奏：最新 1.10.4（2026-10-20）提供 zip/rar/tar.gz 三格式；version.txt 驱动的更新检查构成发行通道。
- 贡献者结构：约 54 名贡献者（contributors API 分页推断）；github-bot.yml 的 trusted_users 名单显示一个 13 人的核心支持团队；560 开放 issue + 新 issue 自动分流到 Discussions 的机制，说明社区支持负载高且已流程化。README 顶部有"防假冒"声明（作者不运营其他 TG/YouTube 频道），从侧面反映项目影响力带来的仿冒风险。
- 涨星轨迹（客观）：2024-10-08 创建，约 13 个月 36515 星、2758 fork；期间 issue 编号已超过 19000（PR 引用 #19047），社区体量大。

## 9. 亮点与风险

**亮点**
1. 发行层设计成熟：上游二进制零改动嵌入 + 预设矩阵 + 一键安装为系统服务，把"参数调优"问题转化为"文件选择"问题，显著降低使用门槛。
2. 一份 25 行模板派生 22 个预设，参数差异集中于 desync 机制与重复次数，策略演进（失效即替换）成本极低；列表与策略解耦（lists/ 独立更新）。
3. 内建可复现的测试器与诊断器（1020 行 PS1 + 诊断清单），把"为什么在我机器上不行"变成可结构化采集的信息，反哺 issue 分流。
4. 支持自动化完整：新 issue 自动回复、stale 管理、可信用户名单、模板强制填写版本，配套俄语文档贴合用户群。
5. 许可证处理规范：LICENSE.txt 明确三段（MIT 双版权 + bol-van 二进制来源声明 + WinDivert LGPLv3/GPLv2 合规说明），并提供哈希校验提示。

**风险 / 局限**
1. 反病毒拦截是日常问题：README 用整节解释 WinDivert 会被判为 `Not-a-virus:RiskTool`（README "АНТИВИРУСЫ"），诊断器也处理冲突软件；使用者需自行权衡。
2. 仓库不含 winws 源码，修复上游缺陷只能等 bol-van/zapret 发版后换 bin；策略参数与上游参数文档强耦合，上游接口变动需要全量重写预设。
3. 策略时效性脆弱：README 明示预设会因目标侧调整而失效，维护依赖持续的列表/参数更新（这正是仓库 11 月仍在 push ipset 的原因）。
4. GitHub 许可证识别为 NOASSERTION（LICENSE.txt 为 MIT 附加三方声明文本），自动工具链无法直接判定；批处理解析器为全仓最脆点，异常参数组合可能安装失败。

## 10. 总结评价

这是一个"工具发行包"而非"工具本体"的典型样本：上游 bol-van/zapret 提供引擎与参数体系，本仓库用 22 份预设批处理、一个 1311 行的纯批处理服务管理器、一个 1020 行的 PowerShell 测试器和一套俄语支持自动化，把引擎封装成了面向俄语大众用户的发行版。13 个月 36.5k 星说明封装层与社区支持本身就能构成一个高星项目。代码上最有阅读价值的部分是 service.bat 的批处理参数解析器与 ipset 三态哨兵设计，以及 test zapret.ps1 的"启动-探测-还原"测试骨架；对研究发行包形态、Windows 服务封装、高流量开源项目支持自动化的人，这个仓库是很好的实例。适合关注开源分发工程与社区运营模式的读者；其功能域的合法性与可用性因司法辖区而异，使用决策与当地法规、目标平台条款相关，本文不做判断。
