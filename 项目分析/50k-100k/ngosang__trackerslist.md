# ngosang/trackerslist 项目分析

> 星数 55481（快照 2026-10-24）｜主语言 —（纯数据仓库，无编程语言）｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：BitTorrent 生态中最权威的公共 tracker 服务器列表集合，由机器人每天自动测活、排序、去重并提交更新。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | ngosang/trackerslist | gh api repos/ngosang/trackerslist |
| 星数 | 55481（快照 2026-10-24） | 任务参数 |
| 主语言 | 无（12 个明文 .txt 列表） | tarball 文件清单 |
| 许可证 | GPL-2.0 | `LICENSE`（正文为 GPLv2 全文）、gh api license 字段 |
| 创建时间 | 2016-04-24 | gh api created_at |
| 最近提交 | 2026-11-01T23:09Z（"Update 2026/11/02"） | gh api commits |
| 默认分支 | master | gh api default_branch |
| Forks / Watchers / Open issues | 6576 / 1286 / 6 | gh api |
| 主页 | https://ngosang.github.io/trackerslist/ | gh api homepage、`_config.yml` |

## 2. 项目是什么，解决什么问题

BT 下载的速度高度依赖客户端能连上多少个 tracker——它们负责在 swarm 中交换 peer 列表。但公共 tracker 服务器生死无常：每天有新的上线，也有大量失效、转私有、携带假种子甚至被安全厂商标记的。手工维护一份"当前可用且优质"的 tracker 清单几乎不可能。

本仓库就是这个问题的工业化答案：一个仓库存放多份按协议（udp/http/https/websocket/i2p/yggdrasil）和按质量（best vs all）划分的 tracker 列表，README 声称"A bot automatically checks the trackers and updates the lists"（`README.md` Notes 节），即机器人每天探测所有候选 tracker 的可用性、延迟与流行度，把结果排序后提交进来。用户只需把 raw 链接粘进 qBittorrent/Transmission/aria2 等客户端的 tracker 设置即可。

仓库有几个体现工程严谨性的设计（均出自 `README.md` Notes 节并在数据文件中得到验证）：

- **排序有依据**：列表"按流行度和延迟从优到劣排序"，`trackers_best.txt` 头部清一色是 `udp://tracker.opentrackr.org:1337/announce`、`udp://open.stealth.si:80/announce` 这类久经考验的大容量 tracker。
- **去重有黑名单**：域名不同但解析到同一 IP、或同一 tracker 的多种写法会被合并，被剔除的条目连同理由记入 `blacklist.txt`。
- **DNS 受限场景兜底**：提供 `*_ip.txt` 版本，把域名替换为 IP 地址，供 DNS 污染环境使用；Cloudflare 背后的 tracker 因 IP 共享无意义而被剔除，所以 IP 版列表更短（README 明确说明）。
- **协议全覆盖**：从传统 udp/http 到 WebTorrent 的 ws/wss、I2P、Yggdrasil 等暗网/overlay 网络的 tracker 各设独立列表。

2016 年创建至今近十年，它已成为 BT 教程和客户端文档里事实上的标准引用源，README 还收录了十多个围绕它运转的第三方工具，形成了一个小生态（详见第 8 节）。

## 3. 技术栈与依赖

这是一个"数据即代码"的仓库，几乎没有传统意义的技术栈：

- **数据格式**：明文文本，每行一个 announce URL，条目间以空行分隔（`trackers_all.txt`、`trackers_best_ip.txt` 等，实际查看验证）。无 JSON/YAML/CSV 等结构化格式——最大化兼容一切客户端和脚本。
- **注释语法**：`blacklist.txt` 用 ` # 原因` 行内注释标记封禁理由（`blacklist.txt` 实文）。
- **页面**：`_config.yml` 仅两行有效配置，用 Jekyll remote theme（pages-themes/modernist）渲染 GitHub Pages 镜像站，这是仓库唯一的"框架"依赖。
- **自动化**：`.github/` 下只有 `FUNDING.yml`，**没有任何 workflows**（tarball 解压与 gh api repos/ngosang/trackerslist/contents/.github 双重确认）。探测与更新机器人完全运行在本仓库之外，源码未公开（未确认其实现语言与托管位置）。
- **无包管理器、无构建、无测试**：不存在 package.json/Makefile 等任何工程清单。

## 4. 目录结构与代码组织

顶层共 18 个条目，全部平铺在根目录，无子目录（tarball 文件清单）：

| 文件 | 职责 |
|---|---|
| `trackers_best.txt` | 20 个最优 tracker（流行度+延迟排序） |
| `trackers_all.txt` | 72 个全部可用 tracker 的总表 |
| `trackers_all_udp.txt` / `_http` / `_https` / `_ws` / `_i2p` / `_yggdrasil` | 按协议拆分的子集（46/16/10/4/16/1 条） |
| `trackers_all_ip.txt` / `trackers_best_ip.txt` / `trackers_all_yggdrasil_ip.txt` | 域名替换为 IP 的 DNS 免污染版（51/20/4 条） |
| `blacklist.txt` | 391 条被剔除 tracker 及封禁理由 |
| `README.md` | 列表索引（含条数徽章与三路镜像链接）、使用说明、生态工具收录 |
| `LICENSE` | GPL-2.0 全文 |
| `_config.yml` | GitHub Pages Jekyll 主题配置 |
| `.github/FUNDING.yml` | 赞助渠道（ko-fi 徽章 + PayPal custom 字段） |

组织方式评述：极端扁平的单层数据布局。分类不是靠目录而是编码进文件名（`trackers_all_<协议>`、`_ip` 后缀），这使每个列表都有稳定的 raw URL 可直接引用——对一个"被 URL 消费"的仓库来说比目录结构更合理。所有条目数与 README 徽章及各列表链接注释完全一致（逐一 grep -c 核对通过）。

## 5. 核心架构与关键实现

本仓库的核心"架构"是三条流水线：**数据组织、质量流水线、分发通道**。

### 5.1 列表体系与数据组织

主表 `trackers_all.txt`（72 条）是全集，`trackers_best.txt` 是其头部 20 条的质量子集，协议子列表是按 scheme 的正交切分（46+16+10+4+16+1 = 93 > 72，因为一个 tracker 可以同时以 udp 和 http 两种 scheme提供服务，如 `tracker.dler.org` 同时出现在 `trackers_all_udp.txt` 与 `trackers_all_http.txt`）。`trackers_all_ip.txt`（51 条）与 all 表的差值即是 Cloudflare 代理等无法用单一 IP 表达的条目（README Notes 节）。

### 5.2 黑名单：带审计痕迹的质量流水线

`blacklist.txt` 是仓库最有分析价值的文件——391 条封禁记录每条都带机器人生成的理由注释，按原因分类统计（对文件 `sed 's/.*# //' | sort | uniq -c` 结果）：

- 178 条 `registered torrents`（tracker 报告注册制种子，不适用于公共下载）
- 100+ 条 `duplicate of <URL>`（域名或解析 IP 与在册 tracker 重复，被指向规范条目，如 41 条重复指向 `udp://9.rarbg.com:2810/announce`、27 条指向 `http://tracker.bittor.pw:1337/announce`）
- 11 条 `malfunction`、7 条 `deprecated by owner`、5 条 `detected by antivirus software`、3 条 `fake seeds`、2 条 `requested by sysadmin`、1 条 `blocked by IDNA ban` 等

这等于一份公开的否决日志：任何被剔除的 tracker 都能查到为什么，且部分理由关联 issue 编号（如 `malfunction issue #374`），质量决策可追溯。

### 5.3 自动更新机制：仓库外的机器人 + 定时提交

关键结论（多重证据）：**更新机器人不在本仓库内**。`.github/` 无 workflows；gh api 最近 10 次提交全部是名为 "Update <日期>" 的机器人式提交，时间戳高度规律（绝大多数在 23:09:2x UTC，2026-10-24 至 2026-11-01 连续每日一次，仅 10-27 出现两次，其中一次在 05:52）。README:29 仅称"A bot automatically checks the trackers"。可见机器人以 GitHub App/独立账号身份在仓库外定时运行，探测→排序→写文件→以日期为 message 提交。机器人实现与探测脚本均未公开（未确认）。

### 5.4 分发通道

每个列表提供三路镜像（README 列表节）：raw.githubusercontent.com 直链、GitHub Pages 镜像（`ngosang.github.io/trackerslist/`，由 `_config.yml` 驱动）、jsDelivr CDN 镜像（`cdn.jsdelivr.net/gh/ngosang/trackerslist@master/...`）。对中国大陆等 GitHub raw 不稳的地区，jsDelivr 与 Pages 镜像显著降低使用门槛——这也是它在中文 BT 社区流传极广的原因之一（列表中 `dler.org`、`acgnx.se`、`t.acg.rip` 等中文系 tracker 也印证了受众构成）。

## 6. 构建与运行

无构建、无测试、无运行时。使用方式（全部来自 README）：

```bash
# 直接把 raw URL 填入任意 BT 客户端的 tracker 列表设置，例如：
# qBittorrent: 工具->选项->BitTorrent->Automatically add these trackers...
# 或下载文本
curl -O https://raw.githubusercontent.com/ngosang/trackerslist/master/trackers_best.txt
# DNS 受限环境用 IP 版
curl -O https://raw.githubusercontent.com/ngosang/trackerslist/master/trackers_all_ip.txt
```

贡献方式也只有一条（README Contribute 节）：发现新 tracker 或失效条目 → 开 issue（"Open a new issue"），列表文件本身不接受手工 PR，完全由机器人写入。想改 .torrent 文件的用户被指向 `bittorrent-tracker-editor` 等第三方工具。

## 7. 数据质量与工程质量

- **条目规模**：12 个列表合计约 265 个有效条目行（各文件 `grep -c` 实测），另有 391 条黑名单记录。规模不大，但胜在每一行都被每日验证。
- **一致性**：所有列表条数与 README 徽章/链接文字逐一吻合；`trackers_all.txt` 与 `trackers_best.txt` 头部顺序一致，验证"best 是 all 的前缀"设计。
- **格式纪律**：统一 `scheme://host:port/announce` 语法，ws 列表用 `wss://` 且无 announce 路径（`trackers_all_ws.txt`），符合 WebTorrent 规范。
- **无代码质量可言**：没有测试、CI、lint——对纯数据仓库这不算缺陷，但意味着黑名单以外的格式错误无人拦截。
- **提交历史即变更日志**：每日 "Update YYYY/MM/DD" 提交充当了事实上的 CHANGELOG，可按日期 diff 出任一 tracker 的进出记录。

## 8. 社区与维护状态

- **维护极度活跃且完全自动化**：gh api 最近 10 次提交覆盖 2026-10-24 至 11-01，每日一次从未间断；仓库非 archived。
- **贡献者结构**：contributors 共 2 人（gh api /contributors 分页计数）——作者 ngosang 与机器人账号。人类参与被刻意收敛到 issue 通道（当前 open issues 仅 6 个），十年间列表文件的写入者始终是机器人，保证了数据的一致性。
- **社区影响**：6576 个 fork；README 收录 12+ 个第三方工具覆盖 Deluge 插件、Transmission/qBittorrent/aria2 的自动加 tracker 脚本（`README.md` Third-party tools 节），形成"列表为核心、脚本为外围"的生态。
- **变现**：ko-fi 徽章 + PayPal 赞助链接（`README.md`、`.github/FUNDING.yml`），无其他商业化。

## 9. 亮点与风险

**亮点**

1. 极简到极致的数据格式：明文 URL 列表零依赖、零门槛，任何 2005 年之后的 BT 客户端都能直接消费。
2. 公开可审计的质量决策：`blacklist.txt` 每条剔除都带机器可读理由，重复项指向规范条目，这是多数同类清单做不到的。
3. 十年如一日的每日更新：规律到可以拿提交时间戳对表（23:09 UTC），可靠性与信任度由此积累。
4. 三路镜像分发 + IP 免 DNS 版本，对网络受限用户（尤其是中国大陆用户）特别友好，直接解释了 55k 星的地域构成。
5. 生态位卡位精准：不做客户端、不做脚本，只做"唯一可信的列表源"，让所有外围工具围绕自己转。

**风险/局限**

1. 核心资产（探测与排序机器人）闭源且不在仓库内，单点依赖作者个人基础设施，作者停更则整条流水线停摆（无社区接管路径，因为列表不接受手工 PR）。
2. GPL-2.0 对纯数据列表的约束力存疑——URL 清单是否构成"程序作品"在版权上是有争议的。
3. 列表权威性高度信任机器人的判定：`detected by antivirus software` 等标记说明候选池里确实混有恶意 tracker，误判/漏判用户无从复核。
4. 每日全量提交使 git 历史近十年膨胀（gh api size 字段 7401KB，对本仓库的数据体量而言几乎全是历史），浅克隆重、全量克隆浪费。

## 10. 总结评价

ngosang/trackerslist 是"把一件小事做到十年不间断"的典范：它没有任何代码，却用自动化流水线、可审计的黑名单和三路镜像把一个脆弱的数据源做成了 BT 生态的基础设施，55k 星是整个 p2p 社区用脚投票的结果。它不适合读源码（没有源码），适合三类人：一是普通 BT 用户，直接把 `trackers_best.txt` 或 `trackers_all_ip.txt` 填进客户端就能提速；二是做下载工具/NAS 脚本的开发者，把它当作可靠的默认 tracker 数据源集成；三是研究数据型开源项目运营的人——它是"机器人维护 + 人类提 issue + URL 即 API"这种极简开源范式的最佳样本。
