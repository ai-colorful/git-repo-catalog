# iptv-org/iptv 项目分析

> 星数 142091（快照 2026-10-24）｜主语言 TypeScript｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

全球公开 IPTV（网络电视）频道播放列表的旗舰收集仓库：把全世界网友提交的公开直播流地址整理成结构化、可分组的 M3U 播放列表，通过 GitHub Pages 每日自动发布，任何播放器（VLC 等）粘贴链接即可观看。

| 项目 | 值 |
| --- | --- |
| 名称 | iptv-org/iptv |
| 星数 / Forks | 142,091 / 8,447（gh api，2026-10-30） |
| 主语言 | TypeScript（工具链）；本体是 M3U 数据 |
| 许可证 | Unlicense（公有领域，`LICENSE`；`package.json` 中脚本部分标 MIT） |
| 创建时间 | 2018-11-14（gh api） |
| 最近活跃 | 2026-10-29（`pushed_at`，机器人每日提交） |
| 数据规模 | `streams/` 下 326 个 .m3u，共 18,401 个频道条目、11,611 个唯一 tvg-id（实测统计） |
| Release | 无传统 release，通过 gh-pages 分支持续部署（`releases/latest` 返回 404） |

## 2. 项目是什么，解决什么问题

互联网上散落着大量电视台主动公开的 HLS/DASH 直播流地址，但它们没有统一的目录：地址分散、时效性强（经常失效）、缺乏频道元数据（台标、节目单、地区归属）。iptv-org/iptv 要解决的就是"把全球公开直播流变成一份持续维护、按国家/语言/分类可检索的标准化播放列表"这一数据工程问题（`README.md`：Collection of publicly available IPTV channels from all over the world）。

用户侧的使用方式极简：把 `https://iptv-org.github.io/iptv/index.m3u` 粘贴进任意支持直播的播放器即可（`README.md` "How to use?"）。除总表外，还提供按分类（`index.category.m3u`）、国家、语言分组的多种播放列表，索引见 `PLAYLISTS.md`。

这个仓库本质上是一个"数据 + 自动化整理工具链"的混合体：`streams/` 目录是内部数据源（按频道所属国家拆分的 326 个 M3U 文件），`scripts/` 是一套约 4,400 行的 TypeScript 管线，负责从 issue 表单收集增删改请求、校验、生成对外发布的多种播放列表，并导出 JSON API。频道元数据（台标、分类、广播区域）不存于本仓库，而是来自姊妹仓库 iptv-org/database，通过 `@iptv-org/sdk` 加载（`scripts/api.ts`、`README.md` "Database"）。

社区协作采用"issue 表单驱动 + 机器人执行"模式：贡献者填写 Add/Edit/Broken 表单，maintainer 打上 `approved` 标签后，每日定时任务自动把变更落到 `streams/`（`CONTRIBUTING.md`、`.github/workflows/update.yml`）。也可以直接发 PR 手动改 M3U 文件，PR 由 CI 按 m3u-linter 规则校验。

法律定位上，仓库明确声明不存储任何视频文件，只收集"据其所知由版权方主动公开"的链接，并提供版权投诉表单与 blocklist 机制（`README.md` "Legal"）。

## 3. 技术栈与依赖

工具链全部为 TypeScript + Node.js 22（CI 中 `setup-node` 指定，`.github/workflows/update.yml`），以 `tsx` 直接运行 TS 脚本、无构建产物（`package.json` scripts）。关键依赖（`package.json`）：

- `@iptv-org/sdk`：iptv-org 生态的统一数据 SDK，提供 Channel/Feed/Guide/BlocklistRecord 等模型与 DataManager（`scripts/api.ts`）
- `@freearhey/core` / `@freearhey/storage-js`：作者（Arhey）自研的集合操作与文件存储抽象，全管线大量使用
- `iptv-playlist-parser` / `hls-parser` / `mpd-parser`：M3U/M3U8/MPD 解析
- `mediainfo.js`：探测流是否真的含视频轨（`scripts/core/streamTester.ts`）
- `@octokit/*`：读写 GitHub issues（机器人工作流）
- `axios` + `socks-proxy-agent`：流可用性测试与代理支持
- `vitest`：测试框架；`eslint` + `m3u-linter`：代码与 M3U 数据双重 lint

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `streams/` | 核心数据：326 个按国家拆分的 M3U（`cn.m3u`、`us.m3u`…，部分带来源后缀如 `at_pluto.m3u`），内部"真相源" |
| `scripts/models/` | 领域模型：`stream.ts`（549 行，最核心）、`playlist.ts`、`issue.ts`、`discussion.ts` |
| `scripts/commands/` | CLI 命令：`playlist/{update,generate,validate,format,test,edit,export}`、`issue/validate`、`discussion/validate`、`readme/update`、`report/create` |
| `scripts/generators/` | 12 个播放列表生成器（categories/countries/languages/regions/subdivisions/cities/sources/index 系列/raw） |
| `scripts/core/` | 基础设施：`playlistParser.ts`、`streamTester.ts`、`logParser.ts`、markdown/html 表格等 |
| `scripts/tables/` | 生成 PLAYLISTS.md 中的统计表格 |
| `tests/` | vitest 测试，13 个 .test.ts + `__data__/` 黄金样本 |
| `docs/` | 面向贡献者的 playlists/workflows 说明文档 |
| `.github/` | 6 个 workflow、issue 模板、讨论模板 |
| `PLAYLISTS.md` | 对外播放列表索引（自动生成，124KB） |
| `m3u-linter.json` | M3U 数据的自定义 lint 规则（require-header/require-link 等 12 条） |

组织方式评述：数据与代码严格分离——数据在 `streams/`，元数据在外部 database 仓库，本仓库代码只负责"流转与发布"。代码侧是单包小型 monorepo 式 CLI 集合，commands → models/generators/core 分层清晰，每个 npm script 对应一个命令入口。

## 5. 核心架构与关键实现

### 5.1 Stream 领域模型（`scripts/models/stream.ts`）

Stream 继承自 `@iptv-org/sdk` 的 Stream 模型并补充播放列表语境的字段（filepath/line/tvgId/labels 等）。从 M3U 条目解析时，用正则从标题中剥离规范化标记——方括号标签（如 `[Not 24/7]`、`[Geo-blocked]`）与画质后缀（`(1080p)`）（`fromPlaylistItem`，约 81-120 行）：

```ts
function parseName(name: string) {
  let title = name
  const [, labels] = title.match(/ \[(.*)\]$/) || [null, '']
  title = title.replace(new RegExp(` \\[${escapeRegExp(labels)}\\]$`), '')
  const [, quality] = title.match(/ \(([0-9]+[pi])\)$/) || [null, '']
  title = title.replace(new RegExp(` \\(${quality}\\)$`), '')
  return { title, labels: labels.split(';'), quality }
}
```

文件的归属路径由频道元数据决定：`updateFilepath()` 取 `channel.country` 生成 `<国家码>.m3u`（约 375 行），这就是 streams/ 按国家组织的实现机制。`isSFW()` 查询频道的 `is_nsfw` 标志，是公共播放列表过滤 NSFW 的开关（约 137 行）。

### 5.2 issue 驱动的机器人更新管线（`scripts/commands/playlist/update.ts`）

每日 CI 调用此命令：加载 GitHub 上带 `approved` 标签的 issue，按标签分流到 remove/edit/add 三个处理器，全部执行完按 filepath 分组回写 `streams/`。add 流程中有两层风控：频道必须存在于 database（`apiData.channelsKeyById`），且命中 blocklist（`dmca`/`nsfw` 理由）直接拒绝（约 215-236 行）。编辑采用事务式快照：改动前 `cacheData()` 克隆全量数据，校验失败则 `resetData()` 回滚并输出 "All changes have been reverted"（约 165-181 行）。若 issue 未填画质，还会用 mediainfo 实测流的分辨率自动补齐（`getStreamInfo`）。

### 5.3 发布列表生成管线（`scripts/commands/playlist/generate.ts`）

这是数据对外的最后一公里：加载 database → 解析 `streams/**/*.m3u` → RawGenerator → 计算频道唯一名 → 排序（频道名 asc、主 feed 优先、地区封锁/非 24/7 降权、按垂直分辨率降序，约 60-71 行）→ 过滤（必须有 channel+feed、按 id 去重）→ 依次跑 12 个生成器，产出 `.gh-pages/` 下的 `index.m3u`、`index.{category,country,language}.m3u` 以及 categories/、countries/、languages/、sources/ 等子目录。`IndexGenerator` 只输出 SFW 流并把频道分类写进 `group-title`（`scripts/generators/indexGenerator.ts` 24-35 行）。每个播放列表还会把关联频道的 EPG 源写进 `x-tvg-url` 头（`scripts/models/playlist.ts` `toString`）。

### 5.4 流可用性测试（`scripts/core/streamTester.ts`）

`playlist:test` 命令用它实测每个 URL：axios 带 UA/Referer 请求，再用 mediainfo.js 分析返回缓冲区里有没有视频轨，区分 `OK / NO_VIDEO / TIMEOUT / HTTP_403_*` 等错误码；测试模式下从固定样本注入结果以保证可复现。支持 HTTP 与 SOCKS 代理，应对地区限制流。

## 6. 构建与运行

来自 `package.json`（无独立构建步骤，tsx 直跑；`postinstall` 自动拉取 database 数据到 `temp/data/`）：

```bash
npm install                # postinstall 会执行 api:load 拉取频道数据库
npm run playlist:update    # 处理 approved issue，回写 streams/（机器人主流程）
npm run playlist:generate  # 生成 .gh-pages/ 全部公共播放列表
npm run playlist:validate  # 校验 M3U 结构与 tvg-id 有效性
npm run playlist:lint      # m3u-linter 格式检查
npm run playlist:test      # 实测流可用性（支持 --proxy、--timeout）
npm run playlist:format    # 规范化 M3U 排序与格式
npm run playlist:export    # 导出 .api/streams.json 供 iptv-org/api
npm test                   # vitest run --no-file-parallelism
npm run lint               # eslint
```

CI 中还可用 `npm run act:*` 系列命令通过 [nektos/act](https://github.com/nektos/act) 在本地复现各 GitHub workflow（`package.json`）。

## 7. 测试与工程质量

- 测试框架 vitest，13 个测试文件集中在命令层（update/generate/validate/format/export 等）与工具函数；采用黄金文件风格——`tests/__data__/expected/` 存放期望输出（生成的 M3U、日志、JSON），命令在临时目录跑完后逐字节比对（`tests/__data__/` 约 860KB 样本）。这种"快照式集成测试"很适合以文件变换为主的管线。
- 数据质量三道闸：PR 级 `check.yml` 只对 `streams/` 变更文件跑 lint+validate（`.github/workflows/check.yml`）；定时 `update.yml` 在机器人提交前再全量 lint+validate（update.yml 34-37 行）；`m3u-linter.json` 定义 12 条 M3U 语法规则。
- CI 共 6 个 workflow：update（每日 00:00 UTC 定时）、check（PR）、format、validate_issue / validate_discussion / validate_label（用脚本自动校验 issue 表单内容合法性，保证机器人输入可控）。
- 工程细节：Action 全部按 commit SHA 固定版本；`permissions: contents: read` 最小权限；机器人用 GitHub App token（iptv-bot）而非个人 PAT 提交。

## 8. 社区与维护状态

- 极度活跃且高度自动化：`pushed_at` 2026-10-29，几乎每天有 `[Bot] Update /streams` 提交（`update.yml` 的 cron `0 0 * * *` 每日跑）。
- 贡献者约 423 人（gh api contributors 分页统计，含匿名）， issue 表单 + 审批标签的模式把维护成本降到"只审批"级别；当前 open issues 66。
- 无 GitHub Release：数据型仓库以 gh-pages 持续交付代替版本发布。
- 组织生态完整：iptv-org 组织下有 database（频道元数据真相源）、api（JSON 接口，由本仓库 export 步骤部署）、epg（节目单抓取）、awesome-iptv（资源清单）、iptv-org.github.io（频道检索站）（`README.md`）。
- 运营依赖 OpenCollective 赞助（README Backers 区块）与 GitHub App 秘钥等基础设施，属于"资金 + 机器人"支撑的长期项目。

## 9. 亮点与风险

**亮点**

1. 教科书级的"数据仓库自动化"：issue 表单 → 标签审批 → 每日机器人批量落地 → 生成发布 → 多仓库部署，全程无人值守（`.github/workflows/update.yml` + `scripts/commands/playlist/update.ts`）。
2. 数据模型与生态解耦：频道元数据集中在 database 仓库，本仓库只存"流链接"这一易变事实，通过 `@iptv-org/sdk` 消费，多仓库间一致性强。
3. 内容风控内建：blocklist 硬拦截 DMCA/NSFW 频道、`isSFW` 过滤公共列表、2024-01-30 起停止分发 NSFW 频道（`PLAYLISTS.md` 开头声明，引用 issue #15723）、版权方有专用下架表单与 <1 工作日的响应承诺（`CONTRIBUTING.md`）。
4. 可用性验证扎实：mediainfo 实测视频轨而非仅 ping URL，区分 NO_VIDEO/TIMEOUT 等细粒度错误码，并支持代理测流。
5. 测试策略与工程形态匹配：黄金文件快照测试覆盖每条命令的端到端输出，M3U 还有专属 linter。

**风险 / 局限**

1. 内容合规的天然灰色地带：链接指向的流是否获授权由提交者声称，仓库方无控制力（README Legal 自己承认 "no control over the destination"），链接列表仍面临地区性法律风险。
2. 数据时效性脆弱：公开直播流失效率高，18,401 条中实际可用比例依赖每日 bot 更新与用户举报（Broken 表单），无自动全量巡检——`playlist:test` 存在但不在定时 workflow 中（仅 act 手动触发，`package.json` act 系列），未确认是否定期运行。
3. 工具链强绑定作者个人生态：`@freearhey/core`、`storage-js` 与 `@iptv-org/sdk` 均为同一维护者的小型库，单点维护风险。
4. 排序/去重/唯一名等逻辑散布正则与多字段约定（标题后缀 `[Not 24/7]`、`(1080p)`），对数据格式极其敏感，改动需同步大量黄金样本。

## 10. 总结评价

iptv-org/iptv 是"把众包数据工程做成基础设施"的标杆项目：14 万星背后的价值不在代码量（脚本仅 4,400 行），而在一套可持续运转的数据治理体系——表单化众包、标签审批、机器人执行、黄金文件测试、多仓库生态与合规流程环环相扣。它适合三类人参考：想搭建自动化数据仓库/机器人流水线的维护者（照抄其 CI 与 issue 驱动模式即可受益）、研究 M3U/IPTV 数据格式的开发者（`scripts/models/stream.ts` 与 generators 是最完整的数据字典）、以及想找可用直播频道的普通用户（直接消费 gh-pages 播放列表）。读源码建议从 `update.ts` 和 `generate.ts` 两条管线入手，配合 `docs/` 与 `CONTRIBUTING.md` 理解全貌。
