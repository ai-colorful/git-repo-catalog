# acmesh-official/acme.sh 项目分析

> 星数 45500（快照 2026-10-24）｜主语言 Shell｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：**一个纯 Unix Shell（POSIX 兼容）实现的 ACME 客户端**，用于向 Let's Encrypt、ZeroSSL 等 CA 自动申请、续期、部署免费 TLS 证书，是同类工具中部署量最大的 shell 脚本之一。

| 项目 | 内容 |
|---|---|
| 名称 | acme.sh（acmesh-official/acme.sh） |
| 星数 | 45500（快照）；gh api 实测 47924 |
| 语言 | Shell（单主脚本 + 插件目录），版本 VER=3.1.7（`acme.sh:3`） |
| 许可证 | GPL-3.0（`LICENSE.md`，gh api 确认） |
| 创建时间 | 2015-12-26 |
| 最近 push | 2026-10-23（gh api，持续活跃） |
| 默认分支 | master（gh api） |
| 代码规模 | 主脚本 9931 行；全仓 Shell 约 67,500 行 |

## 2. 项目是什么，解决什么问题

HTTPS 证书需要每 60-90 天续期，手动管理极易中断服务。acme.sh 实现了完整的 ACME 协议（RFC8555），让用户一条命令完成"验证域名所有权 → 签发证书 → 安装到服务 → 注册 cron 自动续期"的闭环。README（`README.md` §Features）概括其卖点：纯 Shell、兼容 bash/dash/sh、不依赖 Python、不需要 root、支持 ECDSA/SAN/泛域名证书、IPv6 就绪、Docker 就绪。

与 certbot（Python）最大的差异化是零依赖：`_precheck()`（`acme.sh:8140`）显示硬性依赖只有两项——`curl` 或 `wget`（网络），`openssl`（密钥/签名）；`socat` 或 `python` 仅 standalone 模式需要。这使它能在最小化系统、老旧发行版、OpenWRT、各类 NAS 和 BSD 上直接运行，这也是它被 Proxmox、pfSense、OPNsense、FreeBSD.org 等基础设施项目集成的核心原因（`README.md` §Who Uses）。

项目同时是一个多 CA 客户端：`acme.sh:15-45` 内置 ZeroSSL（默认 CA）、Let's Encrypt（含 staging）、SSL.com、Google Trust Services、Actalis 的 ACME directory URL，并支持任意 RFC8555 兼容 CA 与 Pebble 测试模式。验证方式覆盖 http-01、dns-01、tls-alpn-01，甚至包含尚未定稿的 dns-persist-01 草案（`acme.sh:43-46`）。

## 3. 技术栈与依赖

- **语言**：POSIX Shell。主脚本 shebang `#!/usr/bin/env sh`（`acme.sh:1`），刻意避开 bash 专有语法，兼容 Bash/dash/sh（README）。
- **外部命令依赖**（`_precheck`，`acme.sh:8140-8178`）：curl/wget 二选一；openssl（必需）；socat/python（仅 standalone 模式，软性建议）；crontab/fcrontab（自动续期，Windows 下退化为 schtasks，见 `installcronjob` 中的 cygwin 分支 `acme.sh:7611-7621`）。
- **无第三方库**：无任何包管理器清单文件（无 package.json/pyproject 等），DNS API 交互全部用 curl/wget 手写 HTTP 调用完成。
- **Docker**：基于 alpine:3.23，附加 openssl、curl、socat、bind-tools、libidn、jq、supercronic 等运行时（`Dockerfile:1-14`），用 supercronic 在容器内跑 cron。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `acme.sh` | 唯一核心，9931 行、241 个顶层函数的单文件实现 |
| `dnsapi/` | 202 个 DNS 服务商验证插件（dns_cf.sh、dns_ali.sh…），共 43,345 行，另有 README.md |
| `deploy/` | 63 个证书部署 hook（nginx、apache、haproxy、k8s、CDN 等），共 11,353 行 |
| `notify/` | 29 个通知渠道插件（telegram、feishu、dingtalk、slack、smtp…），共 2,554 行 |
| `Dockerfile` | Alpine 镜像构建，含 cron 守护模式 |
| `.github/workflows/` | 34 个 CI workflow：24 个平台矩阵 + shellcheck + PebbleStrict E2E 等 |
| `acme.sh.completion` | Shell 补全脚本 |

组织方式评述：**"单体内核 + 三类文件式插件"**。主脚本不 import 任何模块，所有协议逻辑、配置管理、HTTP 客户端、加密原语都在一个文件里按功能分区排布（工具函数 → HTTP → JWS/加密 → 配置存储 → 验证服务器 → 签发流程 → 安装/升级 → 命令行分发 `_process()`）。而把变化频繁的部分（DNS 服务商、部署目标、通知渠道）外置为独立脚本，通过运行时 `source` + 约定函数名动态挂载。这是插件体系在无模块系统语言里的经典实现。

## 5. 核心架构与关键实现

### 5.1 ACME 协议核心：JWS 签名请求

`_send_signed_request()`（`acme.sh:2689-2853`）是全部 ACME API 交互的咽喉。它先 `_calcjwk()`（`acme.sh:1819`）从账户密钥导出 JWK，base64url 编码 payload，向 ACME 服务器取 Replay-Nonce（先 HEAD 后 GET 双重回退），再用 openssl 完成签名（`_sign()`，`acme.sh:1115`），最后带最多 20 次指数退避重试发送：

```sh
MAX_REQUEST_RETRY_TIMES=20
_sleep_retry_sec=1
_request_retry_times=0
while [ "${_request_retry_times}" -lt "$MAX_REQUEST_RETRY_TIMES" ]; do
  _request_retry_times=$(_math "$_request_retry_times" + 1)
  if [ -z "$_CACHED_NONCE" ]; then
    if _post "" "$nonceurl" "" "HEAD" "$__request_conent_type" >/dev/null; then
      _headers="$(cat "$HTTP_HEADER")"
      _CACHED_NONCE="$(echo "$_headers" | grep -i "Replay-Nonce:" ...)"
```

签名、CSR 生成、证书解析全部靠 openssl 单命令 + 文本处理拼装，没有任何 helper 二进制——这是"纯 Shell 实现加密协议"的关键技巧。

### 5.2 签发主流程 issue() 与双验证路径

`issue()`（`acme.sh:5408-6650`，约 1200 行）是核心状态机：解析域名/SAN → `_initpath()` 建立每域 conf 与目录 → 支持 Le_Vlist 断点续签（order 恢复，见 `acme.sh:5460-5475` 注释引用 issue#3635）→ 按 `--webroot/--standalone/--apache/--nginx/--dns` 分派验证。

**HTTP-01**：`_startserver()`（`acme.sh:3147`）优先 socat、退回 nc/python 起 80 端口临时的 challenge 响应；apache/nginx 模式则临时改写服务器配置签完即还原（`_setApache`/`_restoreApache`，`acme.sh:3835/3810`）。

**DNS-01（泛域名唯一途径）**：在 `issue()` 内通过 `_findHook()`（`acme.sh:4983-5007`）在安装目录/工作目录按 `dns_<provider>` 名字定位插件文件，然后 `source` 进来并调用约定函数：

```sh
d_api="$(_findHook "$_dns_root_d" $_SUB_FOLDER_DNSAPI "$_currentRoot")"
...
if ! . "$d_api"; then
  _err "Error loading file $d_api. ..."
...
addcommand="${_currentRoot}_add"
$addcommand "$txtdomain" "$txt"
```

插件只需实现 `dns_<x>_add()` / `dns_<x>_rm()` 两个函数（见 `dnsapi/dns_cf.sh` 的 Cloudflare 实现，支持 CF_Token 新版 API 与旧版 CF_Key/CF_Email）。TXT 记录生效检测 `_check_dns_entries()`（`acme.sh:5223`）不用本地 dig，而是通过 DoH（Cloudflare/Google/Ali/DNSPod 四家，`_ns_lookup*`，`acme.sh:5063-5196`）查询，最长轮询 20 分钟，避免本机 DNS 缓存与公网传播不一致的坑。

### 5.3 配置存储：无数据库的键值文件

所有持久化状态（账户 conf、每域名 conf、CA conf）都是 `KEY=value` 文本文件，由 `_setopt()/_save_conf()/_read_conf()`（`acme.sh:2888-3010`）以原子方式改写。插件凭证统一走 `_saveaccountconf_mutable()` 存档，`dns_cf.sh:20-22` 展示了标准插件取值模式：环境变量优先、回落读取 conf。这套机制让 202 个插件共享统一的凭证持久化语义。

### 5.4 安装、cron 续期与自升级

`install()`（`acme.sh:8270`）把脚本复制到 `~/.acme.sh/`、装三类插件子目录、注册 shell alias，并调 `installcronjob()`（`acme.sh:7590`）写 crontab：随机分钟 + 随机小时（防 CA 端洪峰），每 6 小时执行一次 `acme.sh --cron`（`acme.sh:7641`）；无 crontab 的 Windows/cygwin 环境自动改用 schtasks（`_install_win_taskscheduler`，`acme.sh:7513`）。

`cron()`（`acme.sh:8473`）每次先检查 `AUTO_UPGRADE=1` 则自升级（`upgrade()` → `installOnline()`，`acme.sh:8888/8827`，直接 curl GitHub 拉新版覆盖安装），再跑 `renewAll()`。续期时机由 `_calc_next_renew_time()` 与 ARI（ACME Renewal Information，`_getARICertID`/`_calc_ari_renew_time`，`acme.sh:8025/2101`）决定，默认剩余 30 天时续期（`DEFAULT_RENEW=30`，`acme.sh:67`）。若系统包管理器接管（`ACME_PACKAGED`），自升级被禁用（`upgrade():8890`）。

## 6. 构建与运行

本项目无构建步骤。来自 README 与源码的实际命令：

```sh
# 安装（在线脚本）
curl https://get.acme.sh | sh -s email=my@example.com
# 签发：webroot 模式
acme.sh --issue -d example.com -w /var/www/html
# 签发：DNS 模式（Cloudflare 插件）
export CF_Token="xxx"; acme.sh --issue --dns dns_cf -d example.com
# 安装证书到 nginx 并 reload
acme.sh --install-cert -d example.com --key-file ... --reloadcmd "nginx -s reload"
# 手动续期 / 强制重签
acme.sh --renew -d example.com --force
# 切换 CA
acme.sh --set-default-ca --server letsencrypt
```

Docker：`Dockerfile` 构建后支持 `daemon` 模式（supercronic 跑 crontab），并为每个子命令生成 `/usr/local/bin/--issue` 式包装命令（`Dockerfile:44-56`）。CI 测试命令见 `.github/workflows/PebbleStrict.yml`：对本地 Pebble 服务器全流程 E2E。

## 7. 测试与工程质量

- **E2E 测试**：`PebbleStrict.yml` 用 Let's Encrypt 官方 Pebble 测试 CA 走完整签发链路（含 DNS manual 模式，且显式设 `PEBBLE_AUTHZREUSE: "0"` 保证 TXT 真被验证——该文件含详细注释）；`DNS.yml`/`pr_dns.yml` 对 DNS 插件做按需触发测试；`pr_notify.yml` 同理。
- **平台矩阵**：24 个平台 workflow（Linux/MacOS/Windows/FreeBSD/OpenBSD/NetBSD/Solaris/Haiku/Hurd/openEuler 等），README §Tested OS 列出 31 个平台，配套独立测试仓 acmesh-official/acmetest。
- **静态检查**：`shellcheck.yml` 对全部 `**.sh` 做 ShellCheck；源码中大量 `# shellcheck disable=SC2034` 标注（如 `dns_cf.sh:2`）说明检查在持续执行。
- **质量观察**：主脚本函数粒度小、注释到位（引用具体 issue 编号）；调试日志分三级（_debug/_debug2/_debug3，`acme.sh:348-428`）并有 `_secure_debug` 避免打印敏感值。`.github/copilot-instructions.md`（8.4KB）专门为 AI 辅助编码定义了 Shell 编码规范，属较新实践。

## 8. 社区与维护状态

- **活跃度**：gh api 显示最近 push 2026-10-23（本分析前一天级别），tarball 内所有文件 mtime 2026-10-16，处于高频维护状态。
- **体量**：forks 5,707、open issues 111（gh api）；主仓库由原作者 neilpang 主导，README 提及 ZeroSSL 是赞助方（页首 Logo 及 `_ZERO_EAB_ENDPOINT` 支持，`acme.sh:19`）。
- **插件生态**：dnsapi 202 个、deploy 63 个、notify 29 个插件几乎全部由社区 PR 贡献，`pr_dns.yml`/`pr_notify.yml` workflow 显示新插件有专门 CI 通道；项目把使用文档全部放在 GitHub Wiki（`wiki-guard.yml`/`wiki-monitor.yml` 专门守护 wiki）。
- **release 节奏**：通过 `vtag.yml` 打 tag 发布，当前 VER=3.1.7；历史节奏未确认（tarball 无 git 历史）。

## 9. 亮点与风险

**亮点**
1. 极限零依赖：仅 curl/wget + openssl 就实现了完整 ACME 协议（含 JWS 签名、DoH 查询、ARI 续期），可在任何有 POSIX shell 的机器上运行——这是 45k+ 星的根本原因。
2. 插件设计出色：`dns_X_add/rm` 约定 + 运行时 source，202 个 DNS 服务商、63 个部署目标、29 个通知渠道，生态广度远超同类工具。
3. 工程化程度高：24 平台 CI 矩阵 + Pebble E2E + ShellCheck，纯脚本项目做到了编译型软件的测试水位；大量面向真实故障的防御代码（断点续签、nonce 双重回退、重试退避、Windows schtasks 兜底）。
4. 自动化闭环完整：安装即配 cron（每 6 小时随机分钟），续期、部署 hook、失败通知、自升级全自动。

**风险/局限**
1. 单文件 9931 行、241 函数，全局变量通过 `export` 隐式传状态，改动主脚本认知成本很高，回归风险集中在 `issue()` 一个函数里。
2. 硬编码解析 HTTP 头/JSON（grep+cut 拼接，如 nonce 提取），依赖服务器响应格式稳定，极端字符场景健壮性弱于结构化解析。
3. 默认 CA 是 ZeroSSL 而非 Let's Encrypt（`acme.sh:44`），新用户可能对默认商业 CA 感到意外，需 `--set-default-ca` 显式切换。
4. shell 插件直接 `source` 并执行用户凭证环境下的网络调用，凭证保存在明文 conf 文件中，权限依赖 `chmod 700` 目录保护（`install():8327`）。

## 10. 总结评价

acme.sh 是"用最朴素的工具做最难的事"的典范：一个 9 千行的 POSIX shell 文件撑起了完整的 ACME 客户端协议栈，再用文件式插件体系把 DNS/部署/通知生态扩张到近 300 个适配器，最终成为全球服务器管理员事实上的默认证书工具。它适合任何需要在最小化、无 Python 环境里自动化 TLS 证书的运维场景直接使用；对想学 ACME 协议细节（JWS、challenge、order 状态机）或想研究"无依赖 shell 工程"（原子配置改写、DoH 纯 shell 实现、跨 20+ 平台兼容技巧）的开发者，这个仓库本身就是一份高质量教材。读源码建议从 `_process()` 命令分发入口进入，沿 `issue()` → `_send_signed_request()` 主线走读，插件开发只需对照 `dnsapi/dns_cf.sh` 模板。
