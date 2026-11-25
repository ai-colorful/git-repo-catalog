# LC044/WeChatMsg 项目分析

> 星数 37500（快照 2026-10-24，分析日 gh api 实测 42.3k）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-14

**分析口径说明（重要）**：原仓库 master 已于 2025-04 被作者重置清空（gh api 实测当前 HEAD 树仅 `readme.md` + `doc/` 4 张图片，历史可见提交最早为 2025-04-12 的 "init"，无 tag、无 release、contributors 接口仅剩 1 人）。本分析基于 fork 网络中保存的最终代码快照 **tqjason/WeChatMsg**（其 master 同步于 2025-04-10，即原仓库重置前两天，`wxManager/__init__.py` 标注 `__version__ = '3.0.0'`），快照含 83 个 `.py` 文件、约 1.66 万行 Python。下文所有源码路径出处均指该快照；原仓库现状均以 gh api 实测为准。

## 1. 项目速览

一句话定位：微信 PC 端聊天记录的本地解密、导出与分析工具，品牌名"留痕 / MemoTrace"，口号"我的数据我做主"。

| 项 | 值 |
|---|---|
| 名称 | LC044/WeChatMsg（留痕 MemoTrace） |
| 星数 | 37500（快照 2026-10-24）；分析日 42357 |
| 主语言 | Python（快照实测 83 个 .py、约 16.6k 行） |
| 许可证 | MIT（fork 快照 `LICENSE`，Copyright 2024 SiYuan；原仓库现树已无 LICENSE 文件） |
| 最近活跃 | 原仓库最后 push 2025-12-30（仅更新 readme 告别文案）；代码实质冻结于 2025-04 |
| 当前状态 | 已停更清空；5425 个 fork 存续代码，官网 memotrace.cn 提供 exe |
| 平台 | 仅 Windows（依赖 winreg/pymem/ctypes Windows API） |

## 2. 项目是什么，解决什么问题

微信官方不提供 PC 端聊天记录的批量导出，本地数据又以加密 SQLite 存储，一旦设备损坏或账号受限，多年聊天记录便难以留存。WeChatMsg 针对这一痛点，把"从已登录微信客户端进程提取数据库密钥 → 解密本地数据库 → 还原会话 → 导出为通用文档格式"整条链路工程化：支持解密 Windows 本地微信数据库（微信 3.x 与 4.0），按联系人还原聊天界面，批量导出 HTML / Word / CSV / TXT / Markdown / JSON / XLSX，并基于聊天统计生成可视化年度报告（readme"功能"清单，fork 快照 `readme.md`）。

项目的定位声明非常明确（fork 快照 `readme.md` 末尾 IMPORTANT 块）：有且只有一个目的"留痕"，前提是"我的数据"，禁止用于非法用途；软件不能找回已删除的聊天记录，"任何企图篡改微信聊天数据的想法都是无稽之谈"；若侵权请联系删除。即：它只处理用户本人设备上已存在的数据，是备份/导出工具，而非入侵或恢复工具。

发行方式是理解该仓库形态的关键：普通用户从官网 memotrace.cn 下载打包好的 exe（GUI 程序），GitHub 仓库承载的是核心库（`wxManager` 数据层 + `exporter` 导出层）、三个 CLI 示例脚本（`example/`）与 AI 相关实验（`MemoAI/`）。快照中不含任何 GUI 代码；而 2023 年的早期 fork（如 qianyu223/WeChatMsg）仍保留 `app/` 目录的 PyQt 单体 GUI，可见 3.0 版本把 v1.x 的 PyQt 单体重构为 headless 核心库，GUI 走"前后端彻底分离"并以闭源 exe 分发（fork 快照 `readme.md`"3.0 全面来袭/全新的Ui"）。

2025-04 起，作者将仓库历史重置、代码清空，未在 readme 中说明原因；2025-12-29/30 的最后更新补上告别文案："此项目很久没有（也不会）更新了"，精力转向 AI 相册项目 TrailSnap，年度报告部分拆分至独立仓库 LC044/AnnualReport（当前原仓库 `readme.md`）。星数却仍在增长（分析日 42.3k），说明其作为"微信数据导出"标志性项目的地位与 fork 存续价值。

## 3. 技术栈与依赖

语言为 Python（`requirements.txt`，fork 快照），关键依赖按用途分组：

- **进程内存取证**：`pymem`（内存读取/特征扫描）、`psutil`（进程枚举）、`pywin32`（winreg/COM）、`yara-python`（内存特征规则匹配，v4 密钥搜索核心）
- **密码学**：`pycryptodome`/`cryptography`（AES-256-CBC 页解密、PBKDF2-SHA1/SHA512 密钥派生）
- **数据解析**：标准库 sqlite3、`zstandard`/`lz4`（CompressContent 解压）、`protobuf`（BytesExtra/packed_info_data 反序列化）、`xmltodict`/`beautifulsoup4`/`lxml`（消息 XML 解析）
- **媒体处理**：`pillow`（图片）、`pysilk-mod`（微信 silk 语音转码）、仓库内直接捆绑 `exporter/ffmpeg.exe`（音频转换）
- **导出**：`python-docx`（Word）、`openpyxl`（Excel）
- **其他**：`aiofiles`（图片异步批量解码）、`dateparser`

值得注意：`requirements.txt` 与实际 import 不一致——`exporter/exporter_docx.py` 顶部 `import docx` 但清单未列 python-docx；`Crypto~=1.4.1` 是 PyPI 上与 pycryptodome 无关的同名包，属可疑写法。快照中无 GUI 依赖（无 PyQt/PySide）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `wxManager/` | 核心数据层：`decrypt/`（密钥提取+数据库/图片解密）、`db_v3/` 与 `db_v4/`（两代微信数据库封装）、`manager_v3.py`/`manager_v4.py`（接口实现）、`parser/`（消息解析工厂+protobuf）、`model/`（消息/联系人数据模型）、`log/` |
| `exporter/` | 导出层：8 种格式导出器（`exporter_html/docx/csv/xlsx/json/markdown/txt/ai_txt.py`）+ 基类 `exporter.py` + `resources/template.html`（419KB 单文件 HTML 模板）与表情包资源、ffmpeg.exe |
| `example/` | 三步 CLI 示例：1-decrypt.py（解密）、2-contact.py（联系人）、3-exporter.py（导出） |
| `MemoAI/` | 聊天记录→LLM 训练数据与微调实验（ChatGLM3-6B 微调指南 `MemoAI/readme.md`、qwen2-0.5b 训练 notebook、`api_server.py`） |
| `doc/` | `数据库介绍.md`（微信库表结构逆向长文，转载自 PyWxDump）、AI 训练文档、截图 |
| `.github/` | 仅 issue/PR 模板（bug/feature/question 三个 yml），无 CI workflow |

组织方式评述：干净的"两层库 + 薄示例"结构——`wxManager` 对下负责解密与数据建模，`exporter` 只面向 `DataBaseInterface` 抽象消费数据；`__init__.py` 用 `DatabaseConnection(db_dir, db_version)` 工厂屏蔽版本差异。导出器与（仓库外的）GUI 都面向同一接口，是 3.0"前后端分离"重构的直接体现。单体倾向存在于层内：`manager_v3.py`（698 行）、`parser/wechat_v4.py`（1015 行）均为大文件，消息类型靠巨型常量值区分。

## 5. 核心架构与关键实现

### 5.1 解密层：密钥获取与 SQLCipher 式页解密（`wxManager/decrypt/`）

微信 PC 端数据库使用 SQLCipher 风格加密（算法注释见 `decrypt_v3.py` 头部）：每 4096 字节一页独立 AES-256-CBC 加密，页尾存 IV 与 HMAC，密钥由主密钥+盐经 PBKDF2 派生。两条版本路线差异很大：

- **v3（微信 3.x，`wx_info_v3.py`）**：依赖预置的偏移表 `version_list.json`（实测收录 **159 个** v3 版本号），按 `WeChatWin.dll` 基址+偏移读昵称/账号/手机号；密钥靠特征扫描——在 dll 内存中定位 `"iphone\0"/"android\0"/"ipad\0"` 字符串，回溯指针候选，逐个用 HMAC 校验候选 32 字节密钥是否匹配 `MicroMsg.db`（`get_key()`，第 161-216 行）。此方案源自 xaoyaoo/PyWxDump（文件头作者注记与 readme 致谢）。
- **v4（微信 4.0，`wx_info_v4.py`，511 行）**：4.0 不再有稳定偏移，改用 **YARA 规则**扫描已提交私有内存区域，特征如 `.{6}\x00{2}\x00{8}\x20\x00{7}\x2f\x00{7}` 圈出密钥桩地址、`xwechat_files\...\db_storage\` 正则定位数据目录；候选密钥经多进程池按 PBKDF2-HMAC-SHA512（**256000 轮**）派生后与 `favorite_fts.db`/`head_image.db` 第一页 HMAC-SHA512 比对（`is_ok()`，第 221-249 行）。文件头注明思路参考 0xlane/wechat-dump-rs。

拿到密钥后的解密是纯密码学操作，v3 与 v4 各一文件（`decrypt_v3.py`/`decrypt_v4.py`），核心都是逐页解密并回写 SQLite 文件头，v4 用 SHA512+256000 轮、64 字节 HMAC，v3 用 SHA1+64000 轮、20 字节 HMAC，均以 `ProcessPoolExecutor(max_workers=16)` 并行处理目录下全部 `.db`。

图片也有独立加密层（`decrypt_dat.py`）：v3 的 `.dat` 是单字节 XOR（按 jpg/png/gif 头自动推码）；4.0 换成 AES，密钥硬编码在 `AES_KEY_MAP` 中，按文件头魔数区分两代（`b'\x07\x08V1\x08\x07' → cfcd208495d565ef`，`V2 → 43e7d25eb1b9bb64`，第 29-32 行）。

```python
# wxManager/decrypt/wx_info_v4.py:221-232（候选密钥验证核心）
def is_ok(passphrase, buf):
    salt = buf[:SALT_SIZE]
    mac_salt = bytes(x ^ 0x3a for x in salt)
    new_key = PBKDF2(passphrase, salt, dkLen=KEY_SIZE, count=ROUND_COUNT, hmac_hash_module=SHA512)
    mac_key = PBKDF2(new_key, mac_salt, dkLen=KEY_SIZE, count=2, hmac_hash_module=SHA512)
    reserve = IV_SIZE + HMAC_SHA512_SIZE
    reserve = ((reserve + AES_BLOCK_SIZE - 1) // AES_BLOCK_SIZE) * AES_BLOCK_SIZE
    mac = hmac.new(mac_key, buf[SALT_SIZE:PAGE_SIZE - reserve + IV_SIZE], SHA512)
    mac.update(struct.pack('<I', 1))  # page number as 1
```

### 5.2 数据层：双版本适配的抽象与消息解析（`wxManager/`）

版本适配的骨架是"抽象接口 + 双实现 + 工厂"：`db_main.py` 定义 ABC `DataBaseInterface`（约 60 个方法：消息分页/按日/按小时统计、联系人、表情包 URL、语音转文字、收藏夹、库合并等），`manager_v3.py` 与 `manager_v4.py` 分别针对 3.x 的 `Msg/` 目录（MSG.db、MicroMsg.db、MediaMSG.db…）与 4.0 的 `db_storage/`（message/contact/session/head_image/media/emotion/biz_message…分库）实现，`wxManager/__init__.py` 的 `DatabaseConnection(db_dir, db_version)` 按版本选择实现类。

4.0 的存储细节值得注意：消息按会话分表 `Msg_<md5(username)>`，与 `Name2Id` 表 join 反查发送者，按 `sort_seq` 排序，多库并行查询用 `ThreadPoolExecutor`（`db_v4/message.py` `MessageDB._get_messages_by_username`，第 85-110 行）。`CompressContent` 走 zstd/lz4 解压，二进制附加数据走 protobuf（`parser/util/protocbuf/` 内含 `msg.proto`、`contact.proto` 等成对的 .proto 与生成代码）。

消息解析采用**工厂注册表模式**（`parser/wechat_v4.py`）：`FACTORY_REGISTRY[消息类型值] → MessageFactory`，为 Text/Image/Audio/Video/Emoji/Link/BusinessCard/Voip/Merged（合并转发，支持递归解析 `parser_merged`）/WeChatVideo/Position/Quote/System 等 15+ 类型各建工厂（第 207-787 行），工厂继承 `Singleton` 共享联系人缓存；消息类型常量直接沿用微信 MSG.Type 的原始数值（`model/message.py` 第 18-46 行，如 `File = 25769803825`），体现了对微信私有协议数值的忠实映射。

### 5.3 导出管线：统一基类 + 8 种格式（`exporter/`）

`exporter/exporter.py` 的 `ExporterBase` 承载公共逻辑：按消息类型/时间范围/群成员三重筛选（`is_selected()`）、进度与完成回调、相邻消息 5 分钟分组辅助（`is_5_min()`）、输出目录约定 `输出目录/聊天记录/<备注>(<wxid>)/`（第 82-180 行）。8 个格式子类各自实现 `export()`：

- **HTML（`exporter_html.py`）是旗舰格式**：读取 419KB 的单文件模板 `resources/template.html`（`/*注意看这是分割线*/` 前后拼头尾），以占位符注入头像 JSON 与会话元数据，消息以 JSON 分页（100 条/页）内嵌，建立时间轴/图片/文件/链接等索引；媒体文件批量异步解码拷贝，表情包资源随包分发——最终产物是一个离线可用、还原聊天界面的自包含 HTML。
- **Word（`exporter_docx.py`）**：python-docx 实现，每 500 条消息拆一个文档，写入前过滤不可打印控制字符（第 40-52 行）。
- **CSV（`exporter_csv.py`）**：`utf-8-sig` 编码（保 Excel 兼容），固定 8 列（消息ID/类型/发送人/时间/内容/备注/昵称/更多信息）。
- 其余 TXT/Markdown/JSON/XLSX 及面向 LLM 微调的 AiTxt。

### 5.4 数据分析与年报、AI 扩展

`db_main.py` 提供的统计接口（`get_messages_by_days/hour/month`、`get_messages_calendar`、`get_chatted_top_contacts` 等）是年度报告的数据源；年报本体（前端可视化）在独立仓库 LC044/AnnualReport，在线预览挂在 memotrace.cn（fork 快照 `readme.md`"2024年度报告"节）。`MemoAI/` 则是"聊天记录训练个人 AI"理念的落地：导出 JSON 按 train/dev 切分，微调 ChatGLM3-6B（`MemoAI/readme.md`），附带 qwen2-0.5b 训练 notebook 与 `api_server.py`。

## 6. 构建与运行

源码运行（来自 `example/README.md` 与 `doc/开发者手册.md`）：

```bash
pip install -r requirements.txt
python example/1-decrypt.py    # 提取密钥并解密数据库（v3 走 version_list.json，v4 走内存扫描）
python example/2-contact.py    # 遍历联系人/群成员（需先填 db_dir、db_version=3或4）
# 3-exporter.py：DatabaseConnection(db_dir, db_version) → get_contact_by_username(wxid)
#              → HtmlExporter(...).start() 即可导出任意格式
```

普通用户路径：从 memotrace.cn 下载打包 exe 双击运行（官方 readme 特别提示闪退需管理员权限运行、杀软报毒需放行——本质是内存取证类操作必然触发的侧效应）。无测试命令；`doc/开发者手册.md` 实际仅是一行链接指向博客设计文档（https://blog.lc044.love/post/13，未验证可达性）。

## 7. 测试与工程质量

- **测试**：快照内无 tests 目录、无 pytest/unittest 配置，测试为零。对一个深度逆向类项目，靠的是 issue 驱动 + QQ 群人工支持（readme"温馨提示/加群方式"）。
- **CI**：原仓库无 workflow（快照 `.github/` 仅 3 个 issue 模板与 PR 模板）；快照仓库里的 `sync.yml` 是 fork 主自己加的上游同步任务，非项目 CI。
- **依赖工程**：`requirements.txt` 与实际 import 不一致（python-docx 缺失、`Crypto~=1.4.1` 包名可疑、`google==3.0.0` 为 protobuf 命名空间垫包），照装难以一次跑通。
- **代码质量观察**：中文注释充分、关键算法注释可读性好；`manager_v3.py`/`wechat_v4.py` 存在 700-1000 行大文件与大量魔数（消息类型值、YARA 字节模式、`AES_KEY_MAP`）；并发使用得当（解密多进程、查询多线程、图片异步 IO）；有统一日志模块（`wxManager/log/logger.py`）。整体是"研究型工程"风格：核心洞察价值高，软件工程规范偏弱。

## 8. 社区与维护状态

- **维护状态：已死亡（作者主动终止）**。时间线（gh api 实测）：2025-04-10 最后一次同步前的 master 仍是完整代码（tqjason 快照）；2025-04-12/26 仓库历史被 "init" 重置、代码删除；2025-12-29/30 最后 push，readme 写明"此项目很久没有（也不会）更新了（未来也许会考虑截图+OCR实现，也许一年或许十年）"，并导流 TrailSnap。未公布原因。
- **Release/tag**：无（接口返回空）。历史上 exe 经 memotrace.cn 分发，GitHub Releases 应已随重置移除（未确认历史版本号）。
- **贡献者结构**：contributors API 仅 1 人——历史重置抹掉了原 2022-2024 的贡献记录（早期 fork 中可见多人协作痕迹，未逐个确认）；readme 有赞助者名单与 QQ 群/公众号运营模式。
- **社区存续**：5425 个 fork 是代码的实际载体，star 数清空代码后仍从 37.5k（2026-10 快照）涨到 42.3k（分析日），典型"仓库虽死，生态仍在"。
- **姊妹项目**：LC044/AnnualReport（年报）、LC044/TrailSnap（AI 相册，作者现主力）、LC044/EasyBox；生态位相关：xaoyaoo/PyWxDump（密钥提取方案来源）、0xlane/wechat-dump-rs（v4 思路来源）。

## 9. 亮点与风险

**亮点**
1. 把"进程内存取证 + SQLCipher 式页解密"的完整链路做成了普通开发者可读的 Python 库，`doc/数据库介绍.md` 对微信库表的逆向整理极具教育价值。
3. v3→v4 适配策略演进教科书级：从"159 条版本偏移表+硬编码特征"（逐版本穷举）转向 YARA 内存特征+候选密钥密码学验证（免逐版本维护），直接决定了项目对微信 4.0 大改版的存活能力。
3. 抽象干净的双版本数据层（ABC 接口 + 工厂 + 消息解析工厂注册表），使导出器与 GUI 均与微信版本解耦；8 种导出格式覆盖全面，HTML 单文件离线还原聊天 UI（分页/时间轴/引用定位）是同类工具中最完整的。
4. 定位克制且有伦理自觉：明示仅处理本人设备数据、不找回已删记录、禁止非法用途，MIT 开源核心库。
5. 个人数据主权理念的完整叙事（导出→年报→个人 AI 微调），超出单纯工具范畴。

**风险/局限**
1. 逆向微信客户端属灰色地带：随微信版本升级随时失效，且可能面临平台方压力——项目最终无解释清空停更即是现实注脚；合规责任完全由使用者承担（readme 免责声明）。
2. 技术路径注定脆弱：需管理员权限读写微信进程内存，杀软误报是常态（readme 自证），仅支持 Windows，微信 4.0.3 之后若再改加密（如 `AES_KEY_MAP` 出现第三代密钥）需社区 fork 跟进。
3. 工程短板：零测试、无 CI、依赖清单残缺（python-docx 缺失、Crypto 包名错误），从源码复现成本高。
4. 仓库已清空：一切引用原仓库路径的教程/链接失效，fork 快照的完整性与可信度需使用者自行核对（本分析的快照经顶层树比对与原仓库重置前状态一致，未逐字节验证）。

## 10. 总结评价

WeChatMsg（留痕）是国内个人数据导出类工具的标志性项目：它以 37.5k+ 星证明了"我的数据我做主"的需求体量，也以 2025 年的无声清空展示了这类依赖逆向微信客户端的项目共同的宿命。其代码价值集中在三处——`wxManager/decrypt/` 的内存取证与页解密实现、v3/v4 双版本适配的抽象设计、以及 exporter 的多格式导出管线；工程规范（测试/CI/依赖管理）虽弱，但核心洞察与可读性使其仍具很高的学习价值。适合三类人：想理解 Windows 进程取证与 SQLCipher 解密原理的安全研究者，想做个人数据备份/数据主权产品的开发者（重点关注其接口抽象与导出管线设计），以及研究"逆向型开源项目如何面对平台压力与版本演进"的开源生态观察者。对想要可用工具的普通用户，应清醒认识到项目已停更、代码散存于 fork，且此类工具的使用边界由用户自行把握。
