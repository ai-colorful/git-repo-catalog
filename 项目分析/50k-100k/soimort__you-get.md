# soimort/you-get 项目分析

> 星数 56858（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：`you-get` 是一个"笨但好用"的命令行网页抓取式媒体下载器（"Dumb downloader that scrapes the web"，出处 `you-get.json`），用纯 Python 标准库从近百个视频/音频/图片站点解析出真实媒体地址并下载。

| 项目 | 事实 |
| --- | --- |
| 名称 | you-get（PyPI 包名 `you-get`） |
| 星数 | 56858（快照）/ 56861（GitHub API 实测 2026-11-03） |
| 语言 | Python 3（仅标准库 + 单一运行时依赖 dukpy） |
| 许可证 | MIT（`LICENSE.txt`，版权行 2012-2024 Mort Yao 与贡献者；GitHub 侧显示 NOASSERTION，因版权声明为自定义格式） |
| 版本 | 0.4.1743（`src/you_get/version.py`），最新 release v0.4.1743 于 2025-01-04 发布 |
| 默认分支 | develop（`gh api repos/soimort/you-get` 实测） |
| 代码规模 | 137 个 .py 文件，约 14700 行；其中站点提取器 99 个文件约 10200 行 |
| 最近活跃 | develop 分支 HEAD 提交 2025-04-27（commit 049548f3，"README.md: add --force-reinstall to pip..."）；仓库 pushed_at 元数据为 2026-10-06，但四个分支（develop/master/gh-pages/youtube-fix）最新提交均不晚于 2025-04-27，2026-10 的推送内容未在分支提交中可见（未确认，可能为非分支 ref 变更） |

## 2. 项目是什么，解决什么问题

you-get 解决的问题由 README 开篇定义得非常直白："download media contents (videos, audios, images) from the Web, in case there is no other handy way to do it"（`README.md`）。作者动机一栏甚至写明了 hacker 文化立场：当网页禁止你保存视频、当页面依赖闭源 JavaScript/Flash 时，你要有一个能拿回数据控制权的开源工具。

用法即它的全部卖点：`you-get <URL>` 即可完成"识别站点 → 解析出全部清晰度流 → 展示（site/title/itag/size）→ 下载 → 自动合并分片"的全流程（示例输出见 `README.md` Getting Started）。除直接下载外还支持三种旁路模式：`-i/--info` 只解析不下载、`--json` 输出机器可读结果（`src/you_get/json_output.py`）、`-p` 直接把流管道进 mplayer/vlc 等本地播放器（`src/you_get/cli_wrapper/player/` 下有 vlc.py、mplayer.py、wmp.py 等包装）。

站点覆盖是它的核心竞争力：`src/you_get/extractors/` 下有 99 个提取器模块，SITES 注册表（`src/you_get/common.py`）映射约 100 个域名关键词，横跨 YouTube、X(Twitter)、Instagram、TikTok、Vimeo、SoundCloud 等国际站点和 Bilibili、优酷、爱奇艺、抖音、快手、喜马拉雅等中国站点（README Supported Sites 表约 79 行站点条目）。仓库与 yt-dlp 属同赛道，但 you-get 更轻（零重量级依赖）且对中国站点的历史覆盖更深。

按 URL 自动分发的机制让"新增一个站点支持"就是"新增一个 extractor 文件 + 在 SITES 表加一行"，这种低门槛的扩展模型是它在 2012-2016 年间靠社区 PR 快速堆出近百个站点支持的制度基础。未匹配到任何注册域名时还有两级兜底：跟随重定向后重试，最终落到 `universal` 通用提取器或用 `google_search()` 把非 URL 文本当搜索词转成视频链接（`src/you_get/common.py: url_to_module`、`google_search`）。

## 3. 技术栈与依赖

- 语言：Python 3 only（`you-get` 启动脚本对 Python 2 直接报 fatal，见根目录 `you-get`），无第三方框架，CLI 用标准库 argparse（`src/you_get/common.py: script_main`）。
- 运行时依赖：仅 `dukpy`（`requirements.txt`、`setup.py` 的 install_requires）——一个嵌入 JS 解释器的 Python 绑定，专用于执行 YouTube 页面的签名解密脚本（`src/you_get/extractors/youtube.py` 顶部 import dukpy 并在缺失时提示安装）。
- 可选依赖：`PySocks`（SOCKS 代理，`setup.py` extras_require 'socks'；`common.py: set_socks_proxy`）。
- 外部进程（可选）：ffmpeg/avconv 用于 DASH 音视频合并与转封装（`src/you_get/processor/ffmpeg.py: get_usable_ffmpeg`），rtmpdump 用于 RTMP 流（`src/you_get/processor/rtmpdump.py`）；无 ffmpeg 时退化到纯 Python 的 FLV/MP4 拼接（`processor/join_flv.py`、`join_mp4.py`）。
- HTTP 层完全基于标准库 `urllib.request`（`src/you_get/common.py` 顶部 import），自带 gzip/deflate 解包（`ungzip`/`undeflate`）、重试（`urlopen_with_retry`）、Cookie 文件加载（`load_cookies` 支持 cookies.txt/sqlite）。
- 打包：setuptools + `you-get.json` 元数据驱动（`setup.py` 读入 name/classifiers/console_scripts），另提供 zsh 插件（`you-get.plugin.zsh`）与 bash/zsh/fish 补全（`contrib/completion/`）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `src/you_get/common.py` | 全项目的中枢（1886 行）：HTTP 工具、SITES 注册表、`url_to_module` 分发、`url_save`/`download_urls` 下载引擎、进度条、CLI `script_main` |
| `src/you_get/extractor.py` | `Extractor`/`VideoExtractor` 基类，prepare→extract→download 生命周期与流信息打印 |
| `src/you_get/extractors/` | 99 个站点提取器（bilibili.py 856 行最大，youtube.py 441 行） |
| `src/you_get/processor/` | 下载后处理：ffmpeg 调用、纯 Python 的 flv/mp4/ts 分片拼接、rtmpdump |
| `src/you_get/cli_wrapper/` | 外部工具包装：player/（mplayer、vlc、wmp 等 5 个播放器）、transcoder/ffmpeg、openssl、downloader（aria2 相关壳） |
| `src/you_get/util/` | fs（文件名清理）、git（从 git 目录推导版本）、log（ANSI 彩色输出）、term、strings、os |
| `src/you_get/json_output.py` | `--json` 模式的结构化输出 |
| `tests/` | unittest 冒烟测试（96 行） |
| `.github/workflows/` | 单个 CI workflow（develop 分支的 lint+test） |
| `contrib/completion/` | bash/zsh/fish 命令补全 |
| `setup.py` + `you-get.json` + `Makefile` | PyPI 打包与发布脚本 |

组织方式评述：典型的"单包 + 平铺插件"单体结构。所有复杂度集中在 common.py 一个中枢文件与 extractors 平铺目录，extractor 之间几乎零耦合、互不 import（各自 `from ..common import *`），新增站点即加文件，这种设计牺牲了公共抽象（每个 extractor 重复大量样板）换取了贡献者并发写入的低冲突。`__main__.py` 里还保留着一个已废弃的 `main_dev`（getopt 版 CLI，引用不存在的 gui/console 模块，实际入口走 legacy `main`），是历史演进的化石层。

## 5. 核心架构与关键实现

### 5.1 URL 分发：SITES 注册表 + 域名关键词匹配

`src/you_get/common.py` 顶部是一张静态字典（约 100 项），键为主域名关键词，值为 extractors 包内的模块名：

```python
SITES = {
    '163'              : 'netease',
    'bilibili'         : 'bilibili',
    'douyin'           : 'douyin',
    'twitter'          : 'twitter',
    'x'                : 'twitter',
    'youtube'          : 'youtube',
    ...
}
```

分发入口 `url_to_module(url)`（common.py:1834）的流程：正则抽出 host → 处理 `.com.cn` 类多级后缀 → 取"后三段"作 domain → 查 SITES 命中则 `import_module('you_get.extractors.<name>')` 并调用该模块的 `download()`/`download_playlist()`；未命中则用 HEAD/GET 探测重定向并递归重试；再不行落回 `universal` 提取器。URL 本身不是链接时（用户直接输关键词），先走 `google_search(url)` 搜索第一条结果（common.py:1823），对应 README 宣传的 "Search on Google Videos and download" 功能。

### 5.2 Extractor 体系：约定式生命周期

`src/you_get/extractor.py` 定义两个基类，`VideoExtractor` 的执行协议是三段式：

```python
def download_by_url(self, url, **kwargs):
    ...
    self.prepare(**kwargs)      # 站点侧：请求页面/API，填 self.title/streams
    ...
    self.streams_sorted = [... for stream_type in self.__class__.stream_types ...]
    self.extract(**kwargs)      # 站点侧：解析真实媒体地址填入 stream['src']
    self.download(**kwargs)     # 基类统一：info_only/json/下载三分支
```

每个站点子类只需声明类属性 `stream_types`（清晰度优先级表，如 youtube.py 的 itag 列表）并实现 `prepare()`/`extract()`；基类按 stream_types 把 `self.streams` 排序，`download()` 里默认选第一优先级，DASH 流则按体积降序选最大（extractor.py:205-211）。基类还统一负责字幕（caption_tracks → srt）、弹幕（danmaku → .cmt.xml）、歌词（lyrics → .lrc）的落盘（extractor.py:244-267）。以 bilibili.py（856 行，全仓最大提取器）为例，它实现了 prepare/prepare_by_cid/extract/download_playlist_by_url 完整覆盖播放列表与多分 P 场景。

### 5.3 下载引擎与断点续传

`download_urls()`（common.py:988）是所有 extractor 共用的落盘引擎：单 URL 直接 `url_save`；多分片（国内站点常见 flv/ts 切片）逐片下载后合并。合并策略三级降级：DASH 音视频 → `processor/ffmpeg.py: ffmpeg_concat_av`；flv/mp4 → ffmpeg 转封装；无 ffmpeg → 纯 Python 的 `join_flv.concat_flv`/`join_mp4.concat_mp4`（common.py:1070-1107）。续传实现在 `url_save()`（common.py:670）：本地已有部分文件时发 `Range: bytes=<received>-` 请求续传（common.py:767、817），配合 `SimpleProgressBar`（文本进度条）与 `PiecesProgressBar`（分片占位条）。

### 5.4 提取器的武器库：common.py 的工具函数与 dukpy

common.py 上半部是给所有 extractor 复用的抓取工具箱：`get_content`/`post_content`（自动 gzip/deflate/编码探测）、`faker` 伪 UA 模式、`url_info`（HEAD 探测 Content-Type/Length）、rc4 解密（`rc4`，供部分国内站点用）。YouTube 提取器则依赖 dukpy 在 Python 内执行站点 JS 完成签名解密（youtube.py 的 `dethrottle`/`n_to_n`），这是全项目唯一需要第三方库的原因。

## 6. 构建与运行

来自 `Makefile`、`setup.py` 与 README：

```bash
# 安装（PyPI 分发，另有 Homebrew/FreeBSD pkg/Flox/Antigen 等 7 种途径，见 README Installation）
pip install you-get
pip install --upgrade --force-reinstall you-get   # README 明确推荐 force-reinstall

# 源码运行（仓库根目录脚本）
./you-get 'https://www.youtube.com/watch?v=jNQXAC9IVRw'

# 测试（Makefile test 目标 = unittest discover）
make test          # 即 (cd src; python -m unittest discover -s ../tests)

# 打包发布
make build         # python -m build
make release       # twine upload --sign dist/you_get-VERSION*
```

无 requirements-dev/lint 配置文件，lint 由 CI 内联安装 flake8 完成。

## 7. 测试与工程质量

- 测试规模：`tests/` 仅 3 个文件 96 行。`tests/test.py` 是对真实站点 URL 的在线冒烟测试（`info_only=True` 只解析不下载），覆盖 imgur/magisto/acfun/tiktok/twitter/weibo 共 6 个活用例；YouTube、Bilibili、SoundCloud 三个用例整段被注释掉（tests/test.py:26-45），侧面反映这些站点解析频繁失效、维护成本高。`test_common.py`/`test_util.py` 各 11/12 行。
- CI：`.github/workflows/python-package.yml`（workflow 名 "develop"），矩阵含 CPython 3.8-3.13 + PyPy 3.8-3.10 共 9 个目标；flake8 用 `--select=E9,F63,F7,F82` 只硬拦语法/未定义名错误，其余 `--exit-zero` 放行；随后跑 `make test`。这种"只拦致命错误"的宽松 lint 策略与全库大量裸 `except:` 的现状一致。
- 代码质量观察：无类型标注、无 formatter 配置；common.py 单文件 1886 行承担了工具库+引擎+CLI 三重职责；保留 Python 2 时代兼容代码（如 `SimpleProgressBar` 处理、`try import DEVNULL`，processor/ffmpeg.py:7-12）。但核心逻辑的函数级抽象（get_content/url_save/download_urls）边界清晰，新 extractor 的边际成本低，这是它作为社区项目能持续十年的真正工程理由。

## 8. 社区与维护状态

- 贡献者：222+ 人（GitHub contributors API 分页实测），版权行列名作者 Mort Yao（soimort）与 2012 年的共同作者 Boyu Guo（iambus，见 LICENSE.txt）。
- Issue 政策特殊：`CONTRIBUTING.md`（中英双语）明确"不接受一般 Issue，必须以带单元测试的 PR 形式报告问题，不合规格的 PR 可能被直接关闭"——这是为了防 Issue 灌水，但也把用户反馈门槛抬得很高；仓库现存 384 个 open issue（API 实测）。
- Release 节奏：CHANGELOG.rst 记录了 78 个带日期的版本（多为 2012-2015 密集期），但该文件早已停止更新（最后条目 0.3.36，2015-10）；实际版本号靠 `version.py` 逐次递增（0.4.1743），release 靠 master 分支打 tag。最近三个 release：v0.4.1743（2025-01-04）、v0.4.1730（2024-08-19）、v0.4.1718（2024-07-14），即 2024 年约每 1-2 个月一版。
- 活跃度判断：develop 分支最后一次代码提交为 2025-04-27（GitHub commits API 实测），此后至本分析日（2026-11-03）约 18 个月无代码更新；仓库 pushed_at 显示 2026-10-06 有推送记录，但四个分支的最新提交都早于该时间，具体推送内容未确认。综合判断：项目处于事实上的低维护/近停滞状态，站点适配（尤其 YouTube、B站）会随时间持续失效，社区已普遍转向 yt-dlp。

## 9. 亮点与风险

亮点：

1. 极致轻量的依赖策略：14.7k 行代码仅一个运行时依赖（dukpy），HTTP/CLI/进度条全部标准库手写，任意有 Python 3 的机器即可直接运行（`requirements.txt`、`setup.py`）。
2. 清晰的插件化扩展模型：SITES 表 + `Extractor` 约定式基类，新增站点只需一个文件，99 个提取器互不干扰，是"低耦合换可并发贡献"的教科书式设计（`src/you_get/common.py`、`src/you_get/extractor.py`）。
3. 功能完整度高：dry-run 三态（info/url/json）、播放器直连管道、Cookie/代理（HTTP+SOCKS+extractor 专用代理）、断点续传、播放列表、弹幕/字幕/歌词旁路下载，命令行产品完成度远超"玩具"级别（common.py script_main 全部 argparse 选项）。
4. 多级降级策略务实：合并优先 ffmpeg，缺失则退化纯 Python 拼接；URL 失配则重定向追踪 → universal 提取器 → Google 搜索三重兜底。
5. 十年生命周期的社区制度：双语 CONTRIBUTING、强制 PR 带测试、222+ 贡献者，说明该治理模型曾经真实有效。

风险/局限：

1. 维护近停滞：develop 分支自 2025-04-27 无代码提交，最新 release 距今已 21 个月（2025-01-04），站点反爬对抗类项目停更即持续失效。
2. 在线冒烟测试大量失守：tests/test.py 中 YouTube/Bilibili/SoundCloud 用例被整体注释，CI 对真实可用性已无保障。
3. 单文件巨石化：common.py 1886 行集工具库/引擎/CLI 于一体，且大量裸 `except:` 吞错，二次开发与调试成本高。
4. 赛道竞争定位尴尬：yt-dlp 在站点覆盖、更新频率、社区规模上全面领先，you-get 的差异化仅剩"零依赖更轻"与部分中国站点的历史积累。

## 10. 总结评价

you-get 是中文开源世界早期最具代表性的工具类项目之一：它以近乎偏执的零依赖立场和"一个站点一个文件"的扁平插件架构，在 2012-2016 年间用社区众包方式堆出了近百个站点的下载支持，高峰期是无数用户命令行里唯一的海量视频下载方案。它的架构至今仍是"如何设计低门槛社区贡献模型"的好教材——但作为反爬对抗型工具，停更 18 个月意味着实际可用性正在缓慢衰减，其生态位已被 yt-dlp 实质性取代。今天它更适合两类人：想要一个轻量、无广告、不装 Node.js 就能顺手抓个视频/音频的终端用户，以及想学习"插件化爬虫工具如何组织代码与社区"的 Python 学习者；若追求最新站点支持，应直接使用 yt-dlp。
