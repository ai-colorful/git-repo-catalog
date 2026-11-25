# pi-hole/pi-hole 项目分析

> 星数 61408（快照 2026-10-24）｜主语言 Shell｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：Pi-hole 是部署在自有 Linux 硬件上的全网 DNS 沉洞（DNS sinkhole），在 DNS 层拦截广告与追踪，无需在客户端装任何软件（`README.md`）。

| 项目 | 内容 |
| --- | --- |
| 名称 | pi-hole/pi-hole（Core 仓库） |
| 星数 | 61408（快照值）；GitHub API：fork 3373、watch 710、open issues 41 |
| 主语言 | Shell（Bash，几乎全部为 bash 脚本，见 `AGENTS.md` "almost entirely bash"） |
| 许可证 | EUPL 1.2（`LICENSE`，GitHub API 标记 NOASSERTION 属正常） |
| 默认分支 | master（gh api 确认）；开发分支为 development（`AGENTS.md` PR 规则） |
| 最近活跃 | 最近 push 2026-11-02；最新 release v6.5 发布于 2026-10-17（gh api） |
| 贡献者 | 约 264 人（gh api contributors 分页头推算） |

## 2. 项目是什么，解决什么问题

传统广告拦截靠浏览器插件，覆盖不到手机 App、智能电视、IoT 设备等非浏览器场景。Pi-hole 的思路是把拦截点上移到网络层：让局域网所有设备把 DNS 指向 Pi-hole，对广告/追踪域名的查询直接返回 NXDOMAIN 或 0.0.0.0，从源头"沉洞"（`README.md`，设备无需客户端软件）。

本仓库是 Pi-hole 三大组件中的 Core，职责有三（`AGENTS.md` 明确划分）：
1. **安装器**：`automated install/basic-install.sh`，一条 `curl -sSL https://install.pi-hole.net | bash` 即可完成安装，同一脚本兼做更新与修复；
2. **CLI**：`pihole` 命令，覆盖 allow/deny 列表管理、gravity 更新、调试、状态、密码等全部管理面；
3. **Gravity 子系统**：`gravity.sh` 把用户配置的 blocklist 下载、解析、去重后编译进一个 SQLite 数据库 `gravity.db`，这是实际拦截的数据源。

组件边界（均为独立仓库/产物，本仓库不含其源码）：DNS 解析与守护进程是 `pi-hole/ftl`（C 实现 dnsmasq 分支 + 内嵌 Web 服务器 + REST API），安装时以**预编译二进制**方式从 GitHub release 下载并做 sha1 校验（`basic-install.sh` 的 `FTLinstall()`，行 1991 起）；Web 管理界面是 `pi-hole/web`，安装器以 git clone 方式获取（`basic-install.sh` 行 82 `webInterfaceGitUrl`）。Docker 部署另有 `docker-pi-hole` 仓库（`README.md`）。

相比 v5 时代，v6 架构的关键变化是 lighttpd 被移除，Web 服务内嵌进 FTL，CLI 的 enable/disable、状态查询等也改为直接调 FTL 的 REST API（`pihole` 中 `piholeEnable()` 调 `PostFTLData "dns/blocking"`，安装器 `disableLighttpd()` 及其对话框文案）。

## 3. 技术栈与依赖

- **语言**：Bash 脚本占绝对主体（约 9400 行核心脚本，`wc -l` 统计）；man page 为 roff（`manpages/pihole.8`）。
- **运行时依赖**：curl、git、SQLite3——值得注意的是 gravity 不直接调系统 `sqlite3`，而是统一走 `pihole-FTL sqlite3 -ni ...`，即用 FTL 内嵌的 sqlite3 接口（`gravity.sh` 行 90、105 等），保证 FTL 与 CLI 操作同一个数据库实现；jq 用于 API 响应解析（`pihole` 行 253）。
- **数据层**：单个 SQLite 库 `gravity.db`，模板见 `advanced/Templates/gravity.db.sql`，核心表：`gravity`（黑名单域）、`antigravity`（白名单域）、`domainlist`（allow/deny/regex 统一表）、`adlist`（blocklist 源）、`group`/`client_by_group`/`adlist_by_group`（按客户端分组的精细策略）、`info`（元数据）。schema 版本演进靠 `advanced/Scripts/database_migration/gravity/` 下 1→21 的 SQL 迁移链（21 个迁移文件）。
- **测试**：BATS + Docker（`test/` 目录），CI 覆盖 17 个发行版。
- **代码质量工具**：ShellCheck（`.shellcheckrc`，CI differential-shellcheck）、codespell、editorconfig-checker、TruffleHog 密钥扫描（`.github/workflows/test.yml`）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `automated install/basic-install.sh` | 安装/更新/修复三合一安装器，2679 行，全仓最大文件 |
| `automated install/uninstall.sh` | 卸载器（240 行） |
| `pihole` | CLI 主入口/路由器，596 行 |
| `gravity.sh` | gravity 数据库构建流水线，1292 行 |
| `advanced/Scripts/` | CLI 子命令实现：`list.sh`（域名列表）、`piholeDebug.sh`（1328 行调试器）、`update.sh`、`updatecheck.sh`、`query.sh`、`api.sh`（FTL REST API 客户端）、`version.sh`、`piholeCheckout.sh`（分支切换）、日志/网络 flush |
| `advanced/Scripts/database_migration/` | gravity.db schema 1→21 的迁移脚本与 SQL |
| `advanced/Templates/` | 落盘资产：`gravity.db.sql` schema、`gravity_copy.sql`、systemd/openrc/cron/logrotate 配置、FTL prestart/poststop 钩子 |
| `advanced/bash-completion/` | pihole 与 pihole-FTL 的补全 |
| `manpages/pihole.8` | 手册页 |
| `test/` | BATS 测试套件 + 17 个发行版 Dockerfile + `run.sh` 运行器 |

组织方式评述：单仓库、扁平脚本式组织，没有构建系统——"构建"就是把脚本安装到 `/opt/pihole`、`/usr/local/bin`（`basic-install.sh` 的 `installScripts()`）。三个产品组件（Core/Web/FTL）严格分仓，Core 通过 git clone（Web）+ 二进制下载（FTL）把它们在装机时粘合。仓库自带 `AGENTS.md` 供 AI/新贡献者速览，明确"stability comes before features"的维护哲学。

## 5. 核心架构与关键实现

### 5.1 安装管线（basic-install.sh）

安装器同时服务三种入口：curl 管道（自动重新下载并以 sudo 重跑自己，`main()` 开头）、本地 clone、Docker。流程：root 检查 → SELinux 强制模式检查（`checkSelinux`，直接中止）→ 包管理器探测（apt/dnf/yum/pacman/apk，`package_manager_detect`，行 283 起）→ 依赖安装 → FTL 二进制按架构检测下载（`get_binary_name` 支持 x86_64/aarch64/ARMv6/v7/riscv64 等，`test_installer_ftl.bats` 有逐架构用例）→ 克隆 Web 仓库 → 安装脚本与模板 → 创建 `pihole` 专用低权限用户 → 交互式选网卡/上游 DNS/隐私等级 → 跑一次 gravity。pihole-meta 包机制的引入让 apt 缓存只在 meta 版本变化时刷新，明显是为树莓派 SD 卡这类 I/O 受限设备做的优化（`main()` 内注释，行 2443 附近）。

### 5.2 Gravity 列表生成流水线（gravity.sh）

`pihole -g` 的主流程（`gravity.sh` 行 1223-1290 的顺序调用）非常清晰地体现"先备库、再下载、最后原子换库"的设计：

1. **不以 root 运行**：入口即把 `/etc/pihole` 交给 pihole 用户并 `exec sudo -u pihole` 降权重跑（行 18-21），因为 gravity 会按 FTL 配置里的路径写文件，路径来自可被 Web 端修改的配置，属于明确的权限边界设计；
2. **迁移**：`migrate_to_database()` 把 v4 时代的 whitelist.txt/blacklist.txt/adlists.list 散文件导入库（对 v5+ 安装为空操作）；
3. **下载**：`gravity_DownloadBlocklists()` 从 `vw_adlist` 视图取所有启用的 adlist URL，逐个调 `gravity_DownloadBlocklistFromUrl()`；下载用 curl 的 **ETag（`--etag-save`/`--etag-compare`）+ If-Modified-Since 双缓存机制**跳过未变更的列表（行 645-682），并有"目标域名本身被 Pi-hole 拦截"的自环检测——此时改用配置的第一个上游 DNS 解析（行 700-730）；
4. **建树与优化**：为 `gravity`/`antigravity` 表建非 UNIQUE 索引（刻意不唯一，容忍低质量 adlist 的重复域名，行 104 注释），`gravity_optimize()` 压缩库；
5. **原子替换**：`gravity_swap_databases()` 先按磁盘剩余空间决定是否保留旧库（剩余块数 ≥ 2 倍旧库才留，行 139-145），旧库滚进 `gravity_backups/gravity.db.1..10` 十代备份轮转，再 mv 换上新库——下载和构建全程发生在 `*_temp` 文件上，任何失败走 `gravity_Cleanup` 且线上库不受影响。

另有两个修复入口：`pihole -g -r recover/recreate` 分别做损坏库抢救与从迁移备份重建（`repairSelector`，行 1176 起）。

### 5.3 CLI 与 FTL API 边界（pihole + advanced/Scripts/api.sh）

`pihole` 主入口按是否需要 root 把子命令分成两组（行 528-577 的两个 case），免 root 组（version/query/status/enable/disable/api）直接执行，需要权限组先校验 EUID 并提示 sudo。enable/disable 已不再碰配置文件，而是登录 FTL REST API 后 POST `/dns/blocking`（行 244-261），支持 `pihole disable 5m` 这类定时恢复；`api.sh`（376 行）实现了基于 FTL 侧 session 的登录/登出与 GET/POST 封装，是 CLI 与守护进程的通信层。

### 5.4 数据库版本迁移（database_migration）

`gravity-db.sh` + 21 个编号 SQL 组成 schema 1→21 的链式迁移，`gravity.sh -u`（`upgrade_gravityDB`）在安装/更新时按当前 `info` 表版本号逐级执行；这种"内嵌 FTL sqlite3 + 链式 SQL"让跨大版本升级（v4 文件 → v5 库 → v6 精简）对用户透明。

## 6. 构建与运行

无传统构建。实际可用的命令（出自 `README.md` 与 `AGENTS.md`）：

```bash
# 安装（官方一键脚本）
curl -sSL https://install.pi-hole.net | bash

# 审计后安装（官方推荐给安全敏感用户的方式）
git clone --depth 1 https://github.com/pi-hole/pi-hole.git Pi-hole
sudo bash "Pi-hole/automated install/basic-install.sh"

# 更新 / 修复 / 卸载
pihole -up          # 调 update.sh，复用安装器
pihole -r           # 调 basic-install.sh --repair
pihole uninstall

# 重建拦截列表（核心日常操作）
pihole -g           # 等价于 gravity.sh，支持 -f 强制、-t 计时

# 测试（需 Docker + buildx，不在宿主机直接跑）
bash test/run.sh --distro debian_12
# Lint
shellcheck 脚本     # CI 强制（differential-shellcheck）
```

安装后布局：脚本装到 `/opt/pihole`，`pihole` 链接进 `/usr/local/bin`，Core 仓库本体留在 `/etc/.pihole` 供 repair/update 引用（`basic-install.sh` `installScripts()`、`update.sh` 行 16）。

## 7. 测试与工程质量

- **规模与框架**：8 个 `.bats` 文件（BATS），约 1500 行测试代码，运行器 `test/run.sh` 在 distro 专属 Docker 容器内执行；测试夹具含 17 个 `test/_*.Dockerfile`，覆盖 Debian 12/13、Ubuntu 20/22/24/26、CentOS 9/10、Fedora 40-44、Alpine 3.21-3.24。
- **风格**：偏黑盒集成测试——`test_fresh_install.bats` 装完真实检查文件属主与权限；`test_gravity.bats` 逐条验证坏 URL/不可达主机/无权限文件等失败路径（而非只测成功路径），这在 shell 项目里少见；`test_symlink_safety.bats` 专门测 versions 文件路径上的符号链接攻击防护；`test_installer_ftl.bats` 逐架构测 FTL 二进制名选择。
- **CI**：`.github/workflows/test.yml` 两个 job——smoke-tests（可执行位检查、differential-shellcheck、TruffleHog 密钥扫描、codespell、editorconfig-checker）+ 17 发行版全矩阵 BATS；另有 merge-conflict、stale、sync-back-to-dev 三个维护性 workflow。
- **质量观察**：`.shellcheckrc` 收紧检查；`gravity.sh` 对所有 sqlite 调用统一捕获输出与退出码并给出用户可读错误；大量 BusyBox/Alpine 兼容注释（如 `stat` 长选项不可用、GNU mktemp 依赖由安装器补齐）。

## 8. 社区与维护状态

- **活跃度**：最近 push 2026-11-02（gh api），距分析日仅一天；open issues 仅 41，说明 issue 消化极快（有 stale bot 维护，`.github/workflows/stale.yml`）。
- **Release 节奏**：最新 v6.5 发布于 2026-10-17（gh api）；v6 大版本重写了架构（内嵌 Web 服务器、去 lighttpd），从仓库内 `migrate_dnsmasq_configs()`、`remove_old_pihole_lighttpd_configs()` 等迁移函数可见升级路径被认真维护。
- **贡献者结构**：约 264 名贡献者（API 分页头推算），`.github/CODEOWNERS` 与 dco.yml 强制 DCO 签署；`AGENTS.md` 的存在表明官方已把 AI 辅助贡献纳入流程。
- **文档与支持**：主要支持渠道是 Discourse 论坛（README），官方文档站 docs.pi-hole.net；未确认项：具体贡献者排行与单人维护风险未逐一核实。

## 9. 亮点与风险

**亮点**
1. **下载-构建-替换全程原子**：gravity 永远写 temp 库再 mv 交换，附十代备份轮转与磁盘空间预判（`gravity.sh` `gravity_swap_databases()`），失败不会弄坏正在服役的拦截库。
2. **权限模型认真**：gravity 主动降权到 pihole 用户运行、CLI 区分 root/非 root 子命令、`test_symlink_safety.bats` 测符号链接攻击——一个以 root 跑安装的 shell 项目对安全边界的处理超出同类水准。
3. **失败路径测试文化**：`test_gravity.bats` 一半用例在测"怎么失败"，17 发行版 CI 矩阵直接对冲了"在一个发行版上测过、在另一个上炸了"这一 shell 项目最大风险（`AGENTS.md` 常见坑第一条）。
4. **组件解耦清晰**：Core（安装/CLI/gravity）/ FTL（解析守护进程）/ Web（前端）三仓分工，Core 通过 sha1 校验的二进制分发 + git clone 粘合，用户可 `pihole checkout ftl development` 单独切换某组件分支（`piholeCheckout.sh`）。
5. **自举式可读性**：README 与 AGENTS.md 明确把安装器与调试器当作 bash 教材推荐阅读，脚本注释密度高。

**风险/局限**
1. **全 bash 代码库的可维护性天花板**：2679 行安装器 + 1328 行调试器，任何改动都要在 17 个发行版×多 init 系统上验证，变更成本随规模线性上涨（`AGENTS.md` 已明言"avoid speculative refactors"）。
2. **curl | bash 安装面**：尽管 README 专门讨论了争议并提供替代方法，一键脚本仍是 root 权限执行远程内容，供应链信任面客观存在。
3. **gravity 强依赖 FTL**：连 sqlite3 都走 `pihole-FTL sqlite3`（`gravity.sh` 行 90），FTL 二进制下载失败即无法安装/修复，Core 侧无本地兜底实现。
4. **拦截原理的固有局限**：纯 DNS 层拦截对 CDN 复用域名、硬编码 IP、DoH 客户端无能为力，这是架构性限制（README 的能力描述未提及绕过场景）。

## 10. 总结评价

Pi-hole Core 是"用 shell 把一件事做到工业级"的代表作：它自己不做 DNS 解析也不渲染界面，而是作为安装器 + CLI + 列表编译流水线，把 FTL 守护进程和 Web 前端两个独立组件粘合成完整产品；其原子换库、降权运行、失败路径测试和 17 发行版 CI 矩阵，展示了防御性 shell 编程的完整范式。星数 61k+ 与日均活跃的维护（最新 release 距今两周）印证其作为自托管广告拦截事实标准的地位。适合自托管用户作为家庭/小办公室全网拦截方案使用，适合想系统学习"生产级 bash 该怎么写"（错误处理、权限边界、跨发行版兼容、集成测试）的开发者精读 `basic-install.sh` 与 `gravity.sh`；若想深入 DNS 拦截引擎本身，应转读 pi-hole/ftl 仓库。
