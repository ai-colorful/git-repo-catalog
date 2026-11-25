# alex/what-happens-when 项目分析

> 星数 41500（快照 2026-10-24）｜主语言 —｜档位 20k-50k｜分析日期 2026-11-06

## 1. 项目速览

一句话定位：把"在浏览器地址栏输入 google.com 并回车后发生了什么"这道经典面试题，写成一篇从键盘物理信号到浏览器渲染的全链路深度长文，并以 wiki 方式由社区持续修正。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | alex/what-happens-when | GitHub 元数据 |
| 星数 | 41500（快照）；2026-11-06 实测 43436 | GitHub 元数据 |
| 主语言 | 无代码（内容型仓库） | 根目录文件清单 |
| 许可证 | CC0 1.0（README 正文声明，仓库无 LICENSE 文件，GitHub license 字段显示 null） | README.rst L15、L692 |
| 创建时间 | 2013-11-07 | GitHub 元数据 |
| 最近内容提交 | 2022-02-08（master 分支，新增西语翻译 PR #343） | commits API |
| 最近 push | 2024-08-19（元数据；master 内容提交停在 2022-02） | GitHub 元数据 |
| 仓库体积 | 260 KB；根目录仅 2 个文件 | GitHub 元数据 + tarball |
| Fork | 5730 | GitHub 元数据 |
| Issue/PR | open issue 247 + open PR 1809 = 2058（累计合并 PR 101 个） | GitHub search API |
| 贡献者 | 81 人（含匿名贡献者） | contributors API |
| 翻译 | 简中/日/韩/西 4 个独立译文仓库 | README.rst L17-19、L706-711 |

## 2. 项目是什么，解决什么问题

这是一个"单文件内容仓库"：全部内容就是根目录的 `README.rst`（711 行、约 5158 词、33KB），没有任何代码。它回答的是一个每个程序员都被问过的面试题——"输入 google.com 回车之后发生了什么"。通常的回答只到"浏览器发了个 HTTP 请求、DNS 解析、服务器返回 HTML、浏览器渲染"为止，而这个项目的野心是把中间被跳过的每一层都展开：键盘按键的电路闭合、USB 轮询与差分信号、操作系统中断与 IDT、三大 OS 各自的按键事件分发路径、URL 解析与 HSTS 预热列表、DNS 与 ARP 的逐字段报文、TCP 三次握手与拥塞控制、TLS 握手、HTTP 报文、服务器端 daemon 的请求处理，最后落到浏览器内部（HTML 解析器、渲染树、图层合成、GPU 渲染）和渲染后的 JS 执行（README.rst L1-19、L28-690）。

项目开篇即声明"这是个协作过程，来发 PR 补齐缺失的细节吧"（README.rst L11-13），定位是一篇由社区维基式打磨的长文，而非一人之作。作者 alex（Alex Gaynor）只做最原始的骨架与整合，正文的许多章节来自合并进来的社区 PR——例如 TCP 拥塞控制一节就是 2021 年由社区成员以 PR #308 加入（commits API）。

它解决的实际痛点是：这道题看似常识，但能讲全"从电流到像素"的人极少；本文把散落在教科书、Wikipedia、内核文档里的知识串成一条单一叙事线，成为中文技术圈（尤其面试准备场景）被引用最多的"标准长答案"。它也顺带成为一篇"如何诚实地承认未知"的示范——文中留有空章节和大量 `未确认` 式的留白（如 Window Server 一节为空标题，README.rst L678-680）。

在同类内容中的位置：网络上流传的同题答案大多是"DNS→TCP→HTTP→渲染"四段式提纲，或指向 Tali Garsiel《How Browsers Work》这类只覆盖浏览器内部的专文；本文的独特性在于把"浏览器之前"的三层（硬件电气层、OS 内核层、网络栈逐字段层）和"浏览器之内"的渲染管线接成同一条线，且每一步都落到具名的机制（`KBDHID.sys`、`IDT`、`chrome://net-internals/#dns`、`AF_INET/AF_INET6` + `SOCK_STREAM`，README.rst L104-107、L213、L305-306）。这种"每章都给你一个可以再去深挖的具体锚点"的写法，是它区别于普通科普文的核心竞争力。

## 3. 技术栈与依赖

没有运行时依赖——它是纯文档仓库。工程层面的全部"技术栈"：

| 层 | 内容 | 出处 |
|---|---|---|
| 内容格式 | reStructuredText（.rst，非 Markdown），用 `::` 代码块与 `.. _` 链接定义写全文 | README.rst 全文 |
| CI | Travis CI（`.travis.yml`，已停摆于 Travis 免费时代） | .travis.yml |
| 文档 lint | `doc8 README.rst` —— reST 语法检查 | .travis.yml L10 |
| 英文写作 lint | `write-good README.rg`（--so --thereIs --cliches），检查被动语态/口语化/陈词滥调 | .travis.yml L11 |

值得注意的是 CI 的选择：一个只有 711 行文本的仓库专门配了 `language: python` + npm 的双工具链，把"文档当代码"的理念做到了极致——语法错误和糟糕的英文表达都会让 CI 挂掉。这是它区别于普通博文的核心工程姿态。

格式选择的细节也值得注意：正文用 reST 而非 Markdown，并使用 reST 原生的 `.. contents::` 指令自动生成目录（README.rst L24-26），目录与章节标题永远同步、无需手维护；同时 reST 的链接定义集中在文末（L692-711），正文保持短链接形态，长文可读性优于 Markdown 的内联链接写法。2013 年项目创建时 GitHub 对 .rst 的渲染支持已成熟，这属于当时技术作者的正常选择，沿用至今未迁移。

## 4. 目录结构与代码组织

| 文件 | 职责 |
|---|---|
| `README.rst` | 全部正文（711 行），GitHub 仓库首页即全文 |
| `.travis.yml` | Travis CI 配置（doc8 + write-good 两道 lint） |

组织方式评述：极端的单文件形态（根目录仅 2 个文件，tarball 与 `gh api contents` 核对一致，无 export-ignore 缺漏）。没有 LICENSE 文件、没有 CONTRIBUTING、没有 issue 模板、没有 CHANGELOG——所有治理都压缩在 README 开头 19 行内（协作邀请 + CC0 声明 + 翻译链接）。对内容型仓库来说这是合理的极简：唯一需要管理的工件就是那篇长文本身，章节即模块，TOC 即导航。

## 5. 核心架构与关键实现

内容仓库的"架构"即章节体系。全文 25 个二级章节构成一条严格顺序的全链路（标题由 reST 下划线逐行提取自 README.rst）：

| 阶段 | 章节（行号） | 覆盖内容 |
|---|---|---|
| 键盘物理层 | The "g" key is pressed（L29）、The "enter" key bottoms out（L43）、Interrupt fires（L100） | 按键电路、去抖、keycode 13、USB/蓝牙/PS2 历史 |
| 传输细节 | USB 分支（L55-77）：endpoint 寄存器、主机每 ~10ms 轮询、SIE 打包、D+/D- 差分信号 1.5Mb/s 低速设备；虚拟键盘分支（L79-96）：电容触摸屏电压降、软中断 | 硬件细节罕见地写到寄存器级 |
| OS 事件分发 | (On Windows) WM_KEYDOWN（L110-138）、(On OS X) KeyDown NSEvent（L141-151）、(On GNU/Linux) Xorg/evdev（L154-163） | Windows 三级驱动链 KBDHID.sys→KBDCLASS.sys→Win32K.sys→SendMessage/WindowProc；OS X I/O Kit kext→WindowServer→Mach port；Linux evdev→window manager |
| URL 处理 | Parse URL（L166）、Is it a URL or a search term?（L179）、Punycode（L187）、Check HSTS list（L196） | 非法字符 Punycode 转码、HSTS 预载防降级攻击 |
| 网络栈 | DNS lookup（L210）、ARP process（L229-298）、Opening of a socket（L301-368）、If a packet is dropped（L401-416） | DNS 缓存链（chrome://net-internals → hosts → 递归）、ARP 报文逐字段 + hub/switch/CAM 表三分支、TCP 分段/IP 包/帧封装、ISN+SYN/ACK/FIN 全流程、慢启动与 AIMD（cubic/New Reno） |
| 应用层 | TLS handshake（L371-398）、HTTP protocol（L418-481）、HTTP Server Request Handle（L484-511） | ClientHello/ServerHello/证书校验/对称密钥/Finished；GET 报文格式、304/ETag 缓存协商；Apache/nginx/IIS、虚拟主机、rewrite、PHP handler |
| 浏览器内部 | Behind the scenes of the Browser（L514）、Browser 高层结构（L524-572）、HTML parsing（L575-621）、CSS interpretation（L623-631）、Page Rendering（L633-665）、GPU Rendering（L667-676）、Window Server（L678，**空章节**）、Post-rendering（L682-690） | 七组件结构、HTML 为何不能用常规解析器（容错性/`document.write` 改输入）、tokenization+树构建、Frame/Render Tree、图层+纹理+合成+D2D/SkiaGL、JS 触发的二次渲染 |

三大 OS 的按键分发路径采用平行章节写法（Windows/OS X/Linux 各一节，L110-163），每个分支都写到具名的系统组件：Windows 到驱动文件名与消息循环（`KBDHID.sys`、`Win32K.sys`、`GetForegroundWindow()`、`WindowProc`），OS X 到内核扩展与 IPC（I/O Kit kext、`WindowServer`、Mach port、`NSEventType KeyDown`），Linux 到 X 协议栈（`evdev`、keymap 重映射、window manager 转发）。三节篇幅相近、粒度对齐，读者可按自己的平台跳读——这是长文处理多分支事实流的干净做法。

各环节深度点评（基于通读全文）：

- **写得最深的环节**：键盘/USB 一段（写到端点寄存器、轮询周期、差分信号速率，L55-77）；Windows 按键事件（具名到具体 .sys 驱动与 API，L110-138）；ARP（含完整请求/应答报文示例和 hub/switch 两种拓扑分支，L250-288）；TCP 握手+拥塞控制（L349-416，2021 年社区 PR 补强）；TLS 握手逐消息（L372-398）。这几节是全文的"技术信用锚点"。
- **写得略的环节**：DNS 一节仅 ~18 行（L210-227），未涉及 EDNS/缓存 TTL/DoH；CSS interpretation 仅 8 行（L623-631）；GPU Rendering 10 行（L667-676）；Window Server 是空标题（L678-680）——协作模式未完成态的活证据。
- **明确过时的内容**：SPDY 协议升级尝试（L418-423，SPDY 已被 HTTP/2 取代）；`gethostbyname` 库函数（L214，现代浏览器均为内置异步 resolver）；Flash/Java 插件（L687）；OS X 一节停留在 WindowServer/mach_ipc_dispatch 时代。文章最后实质更新是 2022 年初，此后技术演进（QUIC/HTTP3、HTTP/2 普及、DNS over HTTPS）完全没有反映。
- **正文中的报文/协议示例**：ARP 请求与应答的逐字段报文（L250-288）、TCP 三次握手与 FIN 关闭的状态机（L349-368）、HTTP GET 请求原文 `GET / HTTP/1.1` + `Host` + `Connection: close`（L425-430）以及 `304 Not Modified`/ETag 缓存协商（L461-470）——这些具体到字节的示例让文章兼具可读性与可操作性，是面试时可以现场白板的素材。

## 6. 构建与运行

无可构建、无可运行的软件。实际可用的操作（均来自项目文件）：

```bash
# 文档语法校验（.travis.yml L10-11 原文命令）
pip install doc8 && npm install -g write-good
doc8 README.rst
write-good README.rst --so --thereIs --cliches

# 阅读：直接渲染 GitHub 仓库首页，或本地
python -m docutils README.rst > README.html
```

提交贡献的唯一路径是标准 GitHub PR 流程（README.rst L11-13），无专门的 CONTRIBUTING 文档。

## 7. 内容结构与编辑质量

- **规模**：单文件 711 行 / 5158 词 / 33KB，25 个章节一条叙事线走完，无分叉、无 FAQ、无外部嵌套文档（README.rst）。
- **组织方式**：严格时序叙事——从按键触点闭合的物理事件起笔，到像素上屏与后续 JS 执行收尾，每个章节都是前一章的自然结果。这种"时间线即目录"的结构是它被反复引用的根本原因：读者可以在任意深度截断（面试背 5 章提纲，或读完 25 章全链路）。
- **编辑质量机制**：doc8 保证 reST 语法，write-good 保证英文文风（.travis.yml）——在 2014 年前后这是相当超前的"文档 CI"实践；Travis 停服后这两道门实际失效（未确认是否有替代 CI）。
- **引用风格**：正文几乎每个硬核声明都挂 Wikipedia 或 W3C 规范链接（L692-711 共 20 个链接定义），可核查性强；但引用多为泛百科级，未引用内核源码或 RFC 编号。
- **翻译生态**：README 头部即列出四个官方认可的译文仓库（L17-19、L706-711）：简中（skyline75489/what-happens-when-zh_CN）、韩语（SantonyChoi/what-happens-when-KR）、日语（tettttsuo/what-happens-when-JA）、西语（gonzaleztroyano/what-happens-when-ES）。四个译本全部是独立仓库而非本仓库的翻译分支，由译者各自维护、各自过时，主仓库对译文质量零背书——这是典型的"CC0 + 去中心化翻译"模式，传播面最大化，一致性归零。
- **中文译本地位**：简中版是此仓库生态中最有名的衍生品，其译文在中文面试圈流传度甚至超过原文；README 明确声明"译文未经维护者 review"（L18-19）。
- **分层阅读建议**（据全文结构归纳）：第一层（10 分钟）——只读 25 个章节标题，即可背出全链路提纲应对面试；第二层（40 分钟）——精读每节正文但跳过报文细节，重点接住五大锚点章节（USB/中断/ARP/TCP/TLS）；第三层（数小时）——沿 20 个链接定义（L692-711）跳出文章，进入 Wikipedia 的拥塞控制、W3C CSS grammar 等二级资料；第四层（工程向）——用 `chrome://net-internals/#dns`（L213）等文中给出的自查入口在本机实际观察每一步。这种"同一篇文本支持四种深度"的分层能力，在长文中属于稀有品。

## 8. 协作模式与维护状态

这是 GitHub 上"维基式文档仓库"兴衰的教科书样本（数据来自 GitHub API，2026-11-06 实测）：

| 指标 | 值 | 解读 |
|---|---|---|
| 已合并 PR | 累计 101 个 | 12 年只消化了 ~100 个贡献 |
| Open PR | 1809 个 | 绝大多数贡献者提交后无人审、永远不合并 |
| Open issue | 247 个 | 同样处于冰封状态 |
| 贡献者 | 81 人（含匿名） | 名义上多人协作，实际长期单人零维护 |
| master 最后提交 | 2022-02-08（西语翻译 #343） | 此后近 4 年无内容变更 |
| 历次提交节奏 | 2019-08 typo 修复、2020-01 DNS 术语修正 #293、2021-01 拥塞控制 #308、2021-04 语法修正 #312、2022-02 加西语版（commits API） | 末期一年 1-3 个小 PR，典型的"半放弃但不清空"状态 |

协作模式的本质：项目的星标增长靠的是"被收藏/被引用"，而非"被参与"。仓库保留了 wikilike 的门面（开篇邀请 PR、开放 issue），但 2020 年后实质上停止吸收社区贡献——1809 个 open PR 意味着社区仍在源源不断提交（纠正错别字、补充新技术段落），只是维护者不再处理。这也造成一个隐忧：正文里最严重的过时内容（SPDY、Flash、gethostbyname）恰恰都有现成的修正 PR 躺在队列里未被合并。

从时间线还能看出合并门槛的变化：2014-2017 年间是真正的活跃协作期（101 个已合并 PR 的主体）；2019 年后每笔 master 提交都是数月乃至一年一遇的"补丁式合并"（typo 修复、术语纠正、单节新增）；2022 年 2 月最后一笔合并的是翻译链接而非内容修正（commits API 五条记录）。5700+ fork 说明大量用户选择"自己改一份"而不是"等上游合并"，fork 数与合并数 56:1 的比值，是协作渠道失效的量化证据。

## 9. 亮点与风险

**亮点**

1. 全链路覆盖无可替代：从 USB 端点寄存器到 GPU 合成的一条完整叙事，网络上没有第二篇达到同等深度与连贯性的中文可读长文（README.rst L29-690）。
2. "文档当代码"的工程姿态：711 行文本配 doc8 + write-good 双 lint CI（.travis.yml），确立了内容仓库的质量门禁范式。
3. 叙事结构自带分级：25 个章节按时序排列，既可作 5 分钟面试提纲，也可作 1 小时精读材料，引用链（Wikipedia/W3C）留足核查路径（L692-711）。
4. CC0 公共领域许可 + 极简单文件形态，使转载、翻译、课程引用零摩擦，是它能在中日韩技术圈病毒式传播的制度基础（README.rst L15）。
5. 多 OS 并列视角（Windows/OS X/Linux 三条按键事件路径，L110-163）与硬件层留白（虚拟键盘分支 L79-96）体现出的严谨完整性，在同类科普文中罕见。

**风险/局限**

1. 技术内容停更于 2022 年初：SPDY（已被 HTTP/2 取代）、Flash/Java 插件、`gethostbyname`、OS X 老 API 等段落已成时代标本，QUIC/HTTP3、DoH、HTTP/2 完全缺席（README.rst L418-423、L214、L687）。
2. 协作机制名存实亡：1809 个 open PR 无人处理，空章节（Window Server，L678-680）多年无人补齐。
3. Travis CI 已失效，doc8/write-good 质量门不再实际执行（.travis.yml，未确认是否迁移到其他 CI）。
4. 许可证合规小坑：CC0 只写在 README 正文里而无独立 LICENSE 文件，GitHub license 检测为 null，商业转载时需人工核实。

## 10. 总结评价

what-happens-when 是 GitHub 内容型仓库的一个标杆物种：它不解决问题、不提供工具，只回答一个问题，却因为把这个问题的答案做到了"从电流到像素"的独占级完整，而在 12 年里积累了 4 万+ 星。它当前处于"知识长青、维护冻结"状态——正文骨架（硬件→OS→网络栈→渲染）几十年不会失效，因此即使四年未更新、1809 个 PR 压仓，它依然是面试准备和技术传播场景的第一引用源，星数至今仍在缓慢上涨（快照 41500 → 实测 43436）。它适合三类读者：面试前需要一份全链路答案框架的求职者（读章节标题+各节首段即可）、想系统理解"一台机器到另一台机器"完整路径的初中级工程师（精读全文）、以及想学习"如何把一个文档仓库做出工程质量"的技术作者（研究它的 CI 与叙事结构）。作为源码来读没有意义，作为长文来读一次值得，作为文档工程的标本值得再读一遍。
