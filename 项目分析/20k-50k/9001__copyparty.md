# 9001/copyparty 项目分析

> 星数 28000（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：一个几乎只依赖 Python 标准库的便携文件服务器，浏览器即客户端，主打断点续传上传、内容寻址去重、多协议（HTTP/WebDAV/FTP/SFTP/TFTP/SMB）与媒体索引浏览。

| 事实 | 值 | 出处 |
|---|---|---|
| 名称/作者 | copyparty，作者 ed（9001，挪威） | `pyproject.toml`、README 头部 |
| 星数 | 28000（快照）；watchers 47253、fork 1920、open issues 271 | gh api repos/9001/copyparty |
| 语言 | Python（后端 ~58k 行）+ 原生 JS 前端（~36k 行） | `copyparty/`、`copyparty/web/` 统计 |
| 许可证 | MIT | `LICENSE` |
| 版本 | v1.20.26（build 2026-10-14） | `copyparty/__version__.py`、latest release |
| 最近活跃 | 仓库最后 push 2026-10-21；changelog 已记录 349 个版本 | gh api、`docs/changelog.md` |
| 创建时间 | 2019-05-26（v0.2.3 "hello world" 2020-01） | gh api、`docs/changelog.md` |

## 2. 项目是什么，解决什么问题

copyparty 要解决的问题是：**把任意一台设备（NAS、树莓派、旧手机、Windows 台式机）变成一个功能完整的文件服务器，且客户端只需要一个浏览器**。README 开篇即点题 "turn almost any device into a file server with resumable uploads/downloads using any web browser"（`README.md`）。它的差异化在于三点：

1. **极低的部署成本**。运行时唯一硬依赖是 Jinja2（`pyproject.toml` 的 `dependencies = ["Jinja2"]`），所有增强功能（缩略图、SFTP、FTP、音频标签等）都是可选 extras。官方甚至提供单文件 `copyparty-sfx.py` 自解压发行版和 `.pyz` zipapp，让"没有管理员权限、装不了东西"的机器也能跑（`README.md` quickstart 一节、`scripts/make-sfx.py`）。
2. **大文件可靠上传**。自研 up2k 上传协议：客户端分块计算 SHA-512，服务端按块落盘稀疏文件并逐块确认，断网、刷新页面甚至换浏览器后都能续传，且"无文件大小上限、可穿过 Cloudflare"（`README.md` features/upload 节，实现见 `copyparty/up2k.py`）。
3. **内容寻址去重与索引**。上传时若服务器已存在同内容文件，默认直接建符号链接/硬链接/reflink，不占第二份空间；配合 sqlite 按卷索引（文件名、路径、大小、音视频元数据），提供类 Everything 的文件搜索和音乐库浏览（`copyparty/up2k.py`、`copyparty/u2idx.py`）。

在它之上还叠了完整的媒体层：图片/视频/RAW 缩略图、音频谱图、浏览器内音乐播放器（服务端按需转码 opus/mp3）、漫画 cbz 阅读器、Markdown 查看器/编辑器、RSS/OPDS 订阅源（`README.md` features 节、`copyparty/th_srv.py`、`copyparty/mtag.py`）。前端是无框架的原生 JS 单页应用，支持 20 余种语言（`copyparty/web/browser.js`、`copyparty/web/tl/`）。

项目自带一份非常认真的同类软件对比文档 `docs/versus.md`（对比 nextcloud、seafile、hfs3、dufs、filebrowser 等），其生态位是"个人/家庭/小团队的瑞士军刀式文件服务"，刻意不追求企业级多用户协作。

## 3. 技术栈与依赖

- **语言与运行时**：Python，`requires-python >= 3.3`，classifiers 声称支持 CPython/PyPy/Jython 甚至 Python 2 兼容路径（`pyproject.toml`）。为兼容老解释器大量使用了 `bos/` 抽象层把 `os.*` 调用统一封装（`copyparty/bos/bos.py`）。
- **硬依赖（仅 1 个）**：Jinja2，用于服务端渲染 HTML 模板；缺失时启动即退出并给出友好安装提示（`copyparty/httpsrv.py` 开头）。
- **可选 extras**（`pyproject.toml` `[project.optional-dependencies]`）：
  - `Pillow` / `pyvips`：图片缩略图；`ffmpeg`（外部程序）：视频缩略图、谱图、音频转码
  - `mutagen` 或 FFprobe：音频元数据索引（`copyparty/mtag.py`）
  - `paramiko`：SFTP 服务端（`copyparty/sftpd.py`）
  - `pyftpdlib` + `pyopenssl`：FTP/FTPS（`copyparty/ftpd.py`）
  - `partftpy`：TFTP；`argon2-cffi`：密码哈希；`pyzmq`：多机消息（`copyparty/broker_mpw.py` 相关）
  - 外部程序 ffmpeg 为"推荐依赖"，Docker 镜像内预装（`README.md` quickstart、`scripts/docker/`）
- **前端**：无框架原生 JS + CSS，`copyparty/web/` 下手工维护（browser.js 268KB），仅引入少量 vendored 库（prism、marked、easymde 等，`copyparty/web/deps/`）。
- **vendored 第三方**：`copyparty/stolen/` 内置 dnslib、ifaddr、qrcodegen 等小型库副本，避免 pip 依赖。
- **并发模型**：纯标准库 threading + multiprocessing（`BrokerThr`/`BrokerMp`，`copyparty/broker_mp.py`），不依赖 asyncio；sqlite3 为索引存储。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `copyparty/` | 核心后端包，全部服务端逻辑（58k 行 Python 的主体） |
| `copyparty/web/` | 内嵌前端 SPA（HTML/JS/CSS/字体/翻译），打包为 gzip 随 pip 分发 |
| `copyparty/bos/` | 跨平台文件系统调用抽象（py2/py3、Windows 差异抹平） |
| `copyparty/stolen/` | vendored 的第三方小库（dnslib、ifaddr、qrcodegen） |
| `bin/` | 独立客户端工具：`u2c.py`（CLI 上传器）、`partyfuse.py`（FUSE 挂载）、事件钩子示例 |
| `docs/` | changelog、versus 对比、示例配置、devnotes、翻译/美化指南等 59 个文件 |
| `tests/` | unittest 测试套件（18 个文件，~3.4k 行） |
| `scripts/` | 发布脚本（sfx/pyz/rpm/pypi/docker）、压测与冒烟测试 |
| `contrib/` | systemd/openrc/rc 单元、nginx 反代配置、Windows WebDAV 配置等部署件 |
| `srv/` | 示例服务配置（`srv/` 下 6 个文件） |

组织方式评述：这是典型的**"单包巨型模块"结构**——没有内部子包分层，核心逻辑集中在少数几个大文件里：`httpcli.py`（287KB，HTTP 协议与所有请求处理）、`up2k.py`（203KB，上传引擎+索引）、`__main__.py`（184KB，695 个 CLI 参数定义）、`authsrv.py`（156KB，账户/卷/权限）。文件间依赖清晰但粒度极粗，作者是唯一长期贡献者（30 名贡献者中绝大多数为一次性提交），这种"单人 monolith"反而维持了极高的一致性。

## 5. 核心架构与关键实现

### 5.1 服务骨架：TcpSrv → SvcHub → HttpSrv 线程池

监听器把连接交给 SvcHub（托管日志、封禁、信号等无法并行化的单例资源，`copyparty/svchub.py:148`），再分发给 HttpSrv 的动态线程池（客户端多于线程时自动扩容，`copyparty/httpsrv.py:269-474`）。重活（上传落盘、索引、移动）通过 Broker 派发：默认 BrokerThr 线程版，`-j2` 以上启用多进程 BrokerMp（`copyparty/broker_mp.py`，用 pyzmq/mp.Process 队列，见 `copyparty/svchub.py` 类 docstring）。同一进程内还托管 FTP/SFTP/TFTP/SMB 等协议服务器，共享 AuthSrv 与 Up2k 实例。

### 5.2 up2k 断点续传上传协议

三步握手（`copyparty/up2k.py`、`copyparty/httpcli.py:3363 handle_post_binary`）：

1. 客户端按自适应块大小（`up2k_chunksize`，1MB 起步、大文件增大）计算每块 SHA-512，把"文件名+大小+全量块哈希表"POST 给服务端；服务端在 `registry` 中建 job、返回唯一 `wark` 任务号，并预创建 `name.PARTIAL` 稀疏文件（`_new_upload`，up2k.py:5393，含 Windows/Unix 稀疏探测与 volflag 协商）。
2. 客户端逐块 POST 二进制（`x-up2k-wark`/`x-up2k-hash` 头），服务端 `handle_chunks` 校验块哈希是否在 `job["need"]` 中并加锁防并发写同一块（up2k.py:3875），落盘后 `confirm_chunks` 从 need 列表移除（up2k.py:3985）。支持子块级续传（`x-up2k-subc` 偏移头，httpcli.py:3410 附近）。
3. need 清空后 `_finish_upload` 原子改名 `.PARTIAL →` 最终名并写入 sqlite 索引。

断点续传的关键是"服务端掌握 need/hash 台账"：中断后客户端调 `get_unfinished_by_user`（up2k.py:433）取回未完成任务的哈希表，重扫本地文件后只补缺块。服务端还支持无哈希直传模式 `--nw` 与 multipart 兼容上传（IE6 都能用，`handle_plain_upload`，httpcli.py:3916）。

### 5.3 内容寻址去重：wark 与链接

每个文件的"wark"（文件钥匙）由 `gen_filekey` 基于内容哈希+大小+inode+盐生成（`copyparty/util.py`，`_gen_fk` up2k.py:330）。上传完成时若发现同 wark 的已有文件，`_symlink`（up2k.py:3759）按优先级执行：FICLONE reflink → 同盘 hardlink → 相对路径 symlink → 复制兜底，且由卷 flag `dedup/hardlink/reflink` 控制。v1.20.24 新增 `redup`：索引重建时检测已被 dedup 过的硬链接并跳过（`_redup`，up2k.py:2111；changelog v1.20.24/25）。整个去重是无损的：删除任一路径不影响其余路径。

### 5.4 权限模型：VFS 树 + AXS 九元组

`authsrv.py` 用 `VFS`（authsrv.py:406）把若干真实目录挂载成带路径的虚拟文件树，每个节点持有 `AXS`（authsrv.py:124）权限集：`uread/uwrite/umove/udel/uget/upget/uhtml/uadmin/udot` 九类分别授权不同用户组，支持"读/写/移动/删除分离"与 write-only 匿名投递目录。账户校验在 AuthSrv（authsrv.py:1151），支持 argon2 哈希与恒定时间比较（changelog v1.20.25 "constant-time verification of filekeys"）。同一套 VFS/权限同时喂给 HTTP、WebDAV（PROPFIND 等，httpcli.py:1950-2400）、FTP、SFTP 各协议，这是它能用一坨标准库撑起六种协议的关键。

## 6. 构建与运行

```bash
# 安装（PyPI）
python3 -m pip install --user -U copyparty          # README quickstart
# 免安装运行
python copyparty-sfx.py                              # 官方自解压单文件
uv tool run copyparty                                # uv 临时运行

# 最简运行：对当前目录开放匿名读写
python -m copyparty
# 典型用法：索引+音频标签+挂载卷+账户
python -m copyparty -e2dsa -e2ts -v /mnt/music:/music:r:rw,foo -a foo:bar   # README 推荐参数

# 测试（scripts/run-tests.sh）
python -m unittest discover -s tests
python scripts/test/smoketest.py                     # 另有端到端冒烟测试

# 打包发布（维护者用）
scripts/make-sfx.sh / make-pyz.sh / make-pypi-release.sh / make-rpm.sh
scripts/docker/make.sh                               # 10 种 Dockerfile 变体（min/dj/iv/im/ac…）
```

部署形态覆盖极广：Docker/Podman（`scripts/docker/`）、systemd/openrc/FreeBSD rc（`contrib/`）、NixOS module（`flake.nix`）、Arch/Homebrew 包、Windows exe（pyinstaller）、Android Termux、iOS a-Shell（`README.md` quickstart）。

## 7. 测试与工程质量

- **测试**：`tests/` 18 个文件约 3400 行，unittest 风格，覆盖去重（test_dedup 247 行）、WebDAV（285 行）、VFS 权限（300 行）、共享/shr、mv/cp、idp 认证、路径穿越防护（ptrav.py）等；另有 `scripts/test/smoketest.py` 端到端冒烟与 `scripts/fusefuzz.py` FUSE 模糊测试（`scripts/run-tests.sh`）。
- **CI**：仓库无 GitHub Actions workflow（`.github/` 只有 issue/PR 模板），测试与发布均由作者本地脚本执行——这是单人项目常见取舍，属已确认事实而非推测。
- **静态检查**：pyproject.toml 内配置了 pylint（py3.11）、mypy `strict = true`、ruff、bandit、black/isort，工具链配置齐全且与代码同步维护。
- **代码质量观察**：大文件长函数较多（httpcli.py 单个 `run()` 超过 700 行），但注释密度高、错误路径处理极其细致（稀疏文件探测、文件系统重 labeled 缓存 `fsutil.py`）。安全响应迅速：2026-10-06 的 GHSA-3c88-p299-7hvr（Windows/Mac 大小写不敏感卷 shadowing 绕过）在 changelog 中有完整披露与修复记录（`docs/changelog.md`）。

## 8. 社区与维护状态

- **活跃度**：极高。gh api 显示最后 push 2026-10-21（本分析 2026-11-05 前两周内）；changelog 349 个版本，仅 2026 年就有十余个 release（v1.20.x 系列密集迭代，如 09-19、10-06、10-14）。
- **release 节奏**：平均每月 1-3 个功能版本，安全修复按需即发；版本号命名幽默（v1.20.26 "one 20 26"）。
- **贡献者结构**：30 名贡献者（gh api），但为典型的"独裁者+贡献者"模式——changelog 中外部贡献以 "thx @xxx" 形式致谢，核心代码几乎全部由作者一人编写；issue 处理量大（open 271，作者回复速度在社区有口碑，README testimonials 节）。
- **文档**：README 3479 行 + docs/ 59 个文件 + 完整 `--help`（695 个参数，在线版 copyparty.eu/cli），文档密度罕见地高。

## 9. 亮点与风险

**亮点**

1. **依赖哲学极致**：运行时仅 Jinja2 一个硬依赖，却能提供六协议+媒体库+全文索引，靠的是标准库与可选探测（`pyproject.toml`、各模块头部 try-import）。
2. **up2k 协议工程度高**：块级台账+稀疏文件+子块续传+多线程并发写，实测可在弱网/大文件场景稳定工作（`copyparty/up2k.py`、`copyparty/web/up2k.js`）。
3. **去重默认透明**：内容寻址 wark + reflink/hardlink/symlink 三级降级，跨卷去重且删除安全（`_symlink` up2k.py:3759）。
4. **部署形态全覆盖**：从 IoT 到 NAS 到 Docker 到单文件 sfx，几乎不存在跑不起来的环境（README quickstart、`scripts/`）。
5. **媒体体验完整**：缩略图（含 RAW）、谱图、服务端转码播放、音乐库标签索引，家庭媒体服务器场景开箱即用（`th_srv.py`、`mtag.py`）。

**风险/局限**

1. **代码集中度过高**：五个文件占后端代码约一半，`httpcli.py`/`__main__.py` 均超 18 万字节，外部贡献者改动的门槛客观上很高（目录统计）。
2. **无云端 CI**：测试全靠作者本地脚本（`.github/` 无 workflows），多平台回归依赖作者自测与社区反馈。
3. **SMB 服务端刻意不安全**：为兼容古老设备而实现且默认禁用，需 `--hack-me-bro` 显式开启（changelog v1.20.25；`copyparty/smbd.py`），误用即暴露风险。
4. **单人维护总线**：作者健康状况或精力是这个 28k 星项目的单点风险；changelog 亦由作者手工撰写，无机构化治理（社区结构观察）。

## 10. 总结评价

copyparty 是"标准库 Python 能做到什么程度"的教科书级答案：一个人、一个 Jinja2 依赖、六年 349 个版本，做出了覆盖 HTTP/WebDAV/FTP/SFTP/TFTP/SMB 的文件服务全家桶，其中 up2k 断点续传与内容寻址去重的实现质量足以作为分布式上传协议的教学案例。它适合自托管玩家、家庭 NAS 用户、需要临时大文件收发的工作组；想深入读源码的人建议按 `svchub.py → authsrv.py(VFS/AXS) → up2k.py(_new_upload/_symlink) → httpcli.py` 的顺序进入，配合 `docs/devnotes.md` 与 `docs/versus.md` 食用。它不适合需要企业级多租户、细粒度审计与高可用集群的团队——作者在 versus.md 中对此有清醒的自我定位。
