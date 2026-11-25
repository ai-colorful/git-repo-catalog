# qishibo/AnotherRedisDesktopManager 项目分析

> 星数 31000（快照 2026-10-24，gh api 2026-11-21 实测 34,858）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：**Another Redis Desktop Manager（ARDM）是一个基于 Electron + Vue 的跨平台 Redis 图形客户端，以"更快、更好、更稳、海量 key 加载不崩溃"为核心卖点，由中国开发者 qishibo 单人主导维护**。

| 项目 | 内容 |
| --- | --- |
| 名称 | Another Redis Desktop Manager（ARDM） |
| 星数 | 34,858（gh api 实测，2026-11-21） |
| 语言 | JavaScript（Vue 2 单文件组件 58 个 + JS 39 个，不含锁文件） |
| 许可证 | MIT（`LICENSE`） |
| 仓库创建 | 2019-01-08（gh api `created_at`） |
| 最近 push | 2026-11-17（gh api `pushed_at`，非常活跃） |
| 最新版本 | v1.7.4（2026-08-18，gh api releases） |
| 版本号基准 | `pack/electron/package.json` version 1.7.4（发布版本），外层 `package.json` version 1.1.1 为前端包版本，两者不同步 |

## 2. 项目是什么，解决什么问题

ARDM 解决的是 Redis 桌面管理 GUI 的需求：连接管理、key 浏览/增删改、多数据类型查看编辑、CLI 控制台、服务器状态监控、慢日志、内存分析等。README.md 开篇即宣称 "A faster, better and more stable redis desktop manager... it won't crash when loading massive keys"，明确把「大数据量场景下的稳定性」作为与同类工具的差异点（`README.md`）。

作者认为当时的主流 Redis GUI（README 未点名，社区语境下即 RDM 商业版与若干 Electron 客户端）要么收费、要么在 key 数量大时卡死。ARDM 的技术对策是全链路采用 SCAN 流式分页 + 虚拟滚动渲染，保证百万级 key 下 UI 不冻结（详见第 5 章，`src/components/KeyList.vue`、`src/components/KeyListVirtualTree.vue`）。

功能覆盖面相当完整：支持 standalone/Cluster/Sentinel 三种拓扑、SSH 隧道（含 SSH+Cluster 组合）、SSL/TLS、Redis 6 ACL、只读模式；数据类型除 5 种基础类型外还支持 Stream（含 consumer group）、RedisJSON、TimeSeries、Vector 等 Redis 模块类型（`src/components/contents/` 下 10 个组件）；值格式解析覆盖 Gzip/Brotli/Deflate/Msgpack/Protobuf/PHP Serialize/Java Serialize/Pickle/Binary/Hex 等 15 种（`src/components/viewers/`）。

分发模式是「GitHub 免费下载 + 商店付费赞助」：GitHub/Gitee release、chocolatey、winget、snap、brew 渠道全部免费，Microsoft Store 与 Mac App Store 版本则收费作为赞助渠道（`README.md` Sponsor 章节）。作者另维护国内可达的 Gitee 镜像发布渠道，这与其面向中国用户群体的定位一致。

## 3. 技术栈与依赖

来自根 `package.json` 与 `pack/electron/package.json`：

| 层 | 技术 | 版本 | 用途 |
| --- | --- | --- | --- |
| 桌面壳 | Electron | 12.2.3 | 主进程/窗口/IPC（`devDependencies.electron`） |
| 桌面壳 | electron-builder / electron-updater | 23.x / 4.6.5 | 多平台打包与自动更新（`pack/electron/package.json`） |
| 前端框架 | Vue 2 | ^2.6.11 | 渲染进程 UI，Options API |
| UI 组件库 | Element UI | ^2.4.11 | 主体控件，配 `element-variables.scss` 主题定制与 dark/light 双主题 |
| 表格 | vxe-table | ^3.9.0-rc.23 | 大数据表格（Hash/List/Zset 内容） |
| Redis 客户端 | ioredis | ^5.3.2 | 渲染进程内直连 Redis，唯一 Redis 协议依赖 |
| SSH | tunnel-ssh | ^5.1.2 | SSH 隧道建立 |
| 编辑器 | monaco-editor | ^0.30.1 | CLI/JSON/自定义格式编辑器 |
| i18n | vue-i18n | ^8.7.0 | 多语言（Vue 2 配套） |
| 作者自有 fork | @qii404/json-bigint、@qii404/redis-splitargs、@qii404/vue-easy-tree | - | 大数 JSON、命令行拆参、虚拟滚动树 |
| 格式解析 | protobufjs、php-serialize、pickleparser、java-object-serialization、algo-msgpack-with-bigint、rawproto | - | 各类序列化格式的识别与解析 |

构建链是经典的 Vue CLI 2 时代 webpack 4 手工配置（`build/webpack.base.conf.js`、`build/webpack.dev.conf.js`、`build/webpack.prod.conf.js`），未迁移到 Vite。整个工程刻意保持零 TypeScript、零状态管理库（Vuex 未引入），组件间通信靠自建事件总线 `src/bus.js`。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `pack/electron/` | Electron 主进程：electron-main.js（窗口/IPC）、update.js（自动更新）、win-state.js（窗口状态持久化）、font-manager.js、electron-builder 配置 |
| `pack/scripts/` | 打包辅助（mac 公证 notarize.js） |
| `src/src/`（仓库根 `src/`） | 渲染进程全部代码：入口 main.js、App.vue、redisClient.js（连接层）、storage.js（localStorage 封装）、commands.js（命令表/CLI 提示）、util.js（工具）、addon.js（启动钩子） |
| `src/src/components/` | 32 个业务组件：连接管理、key 列表、key 详情、CLI、状态、慢日志、内存分析、设置等 |
| `src/src/components/contents/` | 10 种数据类型的展示/编辑组件（String/List/Hash/Set/Zset/Stream/ReJson/TimeSeries/Vector/Array） |
| `src/src/components/viewers/` | 15 种值格式查看器（Text/Json/Binary/Hex/Gzip/Brotli/Deflate/DeflateRaw/Msgpack/Protobuf/PHPSerialize/JavaSerialize/Pickle/Custom/OverSize） |
| `src/src/i18n/` | vue-i18n 配置与 13 个语言文件 |
| `build/` + `config/` | webpack 4 构建脚本与环境配置 |
| `.github/workflows/` | 6 个 CI 工作流（各平台构建发布、winget、sponsors 生成） |
| `static/` | dark/light 两套预编译 Element 主题 CSS 与图标字体 |

组织方式评述：单体仓库、双 package.json 布局——根 package.json 负责前端构建，`pack/electron/package.json` 独立声明主进程依赖与 electron-builder 配置（打包时把构建产物 dist 与 pack/electron 合并，见 `package.json` scripts `pack:prepare`）。主进程与渲染进程代码物理分离，界限清晰；渲染进程内部未再分层（无 store/router 深度使用，`src/router/index.js` 仅注册空路由），是典型的「中小型桌面应用直接平铺组件」结构。

## 5. 核心架构与关键实现

### 5.1 薄主进程：Redis 连接直接放在渲染进程

与多数 Electron 应用「重逻辑进主进程」不同，ARDM 的主进程极薄（`pack/electron/electron-main.js`，250 行），只负责：单实例锁、窗口创建与状态恢复（`win-state.js`）、主题切换 IPC、命令行参数透传（`getMainArgs`）、macOS 菜单和自动更新（`update.js`，手动确认下载模式 `autoDownload = false`）。

BrowserWindow 以 `nodeIntegration: true, contextIsolation: false, enableRemoteModule: true` 启动（electron-main.js:66-71），因此 ioredis、tunnel-ssh、fs 等全部在渲染进程直接运行，无需主进程 IPC 转发命令。这是代码量小、响应直接的原因，代价是放弃了 contextIsolation 的安全隔离（本应用不加载远程内容，风险可控，但 Electron 12 已属旧版本）。

### 5.2 连接层：四种拓扑 + sendCommand 补丁（src/redisClient.js）

`createConnection` 按配置分流为 standalone / Sentinel（ioredis sentinel 参数）/ Cluster（ioredis Cluster + natMap）/ SSH 隧道四种模式（redisClient.js:52-88）。两个关键设计：

其一，monkey-patch `Redis.prototype.sendCommand`（redisClient.js:12-38）实现两个横切能力：只读模式下查 `writeCMD` 命令表（`src/commands.js`）直接 reject 写命令；同时记录每条命令耗时并通过事件总线广播给 CLI 界面的执行日志面板。

```js
Redis.prototype.sendCommand = function (...options) {
  const command = options[0];
  if (this.options.connectionReadOnly && writeCMD[command.name.toUpperCase()]) {
    command.reject(new Error('You are in readonly mode! ...'));
    return command.promise;
  }
  const start = performance.now();
  const response = sendCommand.apply(this, options);
  ...
  vue.$bus.$emit('commandLog', record);
```

其二，SSH+Cluster 组合（redisClient.js:126-147、277-323）：先通过隧道连任一节点执行 `CLUSTER NODES` 解析出全部 master，再为每个节点各建一条 SSH 隧道，将「内网地址 → 本地端口」写入 ioredis 的 `natMap`，让 Cluster 客户端重定向时也走隧道。SSH+Sentinel 则先查 `SENTINEL GET-MASTER-ADDR-BY-NAME` 再对 master 建隧道（redisClient.js:101-123）。

其他细节：`stringNumbers: true` 防止大整数精度丢失（redisClient.js:221）；TLS 允许自签名证书（`rejectUnauthorized: false` 并跳过主机名校验，redisClient.js:337-341）；`family: 0` 支持 IPv4/IPv6 双栈；重试 3 次失败后主动断连并提示（redisClient.js:344-355）。协议层面使用 ioredis 5 默认的 RESP2，源码中无 RESP3/protocol 配置项（未确认支持 RESP3）。

### 5.3 大 key 量浏览器：SCAN 流 + 虚拟树（卖点实现处）

key 列表不使用 KEYS，而是 ioredis 的 `scanBufferStream`（SCAN 命令的流式封装 + Buffer 返回，避免编码损失）：`src/components/KeyList.vue:147-162` 对每个 master 节点建一条 scan 流，默认每页 500 条（cluster 模式按 master 数均分，KeyList.vue:62-78），搜索模式加大到 10000，"Load All" 模式 50000，通过 stream.pause/resume 实现「滚动到底再加载」与一次全量两种形态。Hash/Set/Zset 的成员浏览同样用 hscanBufferStream/sscanBufferStream/zscanBufferStream 流式分页（`src/components/contents/KeyContentHash.vue:196` 等）。

渲染端 `src/components/KeyListVirtualTree.vue` 使用作者 fork 的 `@qii404/vue-easy-tree` 做虚拟滚动树，key 按 separator（默认 `:`）在 `src/util.js` 的 `keysToTree`/`formatTreeData`（util.js:240-300）中构建成文件夹树并统计 keyCount；节点超过 `treeNodesOverflow: 20e4`（20 万，KeyListVirtualTree.vue:89）时切换渲染策略。树展开时异步排序（nodeExpand → sortByTreeNodes）避免一次性排序大数组阻塞。这套组合是 README 中「海量 key 不崩溃」卖点的实际出处。

### 5.4 值查看器矩阵与 localStorage 持久化

内容展示按「数据类型组件（contents/10 个）× 格式查看器（viewers/15 个）」矩阵组合：`src/util.js` 提供 isJson/isPHPSerialize/isMsgpack/isProtobuf/isJavaSerialize/isPickle/isGzip 等嗅探函数（util.js:81-183），自动选择查看器；另有 ViewerCustom 支持用户注册外部脚本（python/node/shell）解码值，通过 `{KEY}/{VALUE}/{HEX}/{HEX_FILE}` 占位符传参（README Custom Viewer 章节）。Protobuf 查看器还支持上传 .proto profile 并与 redis key 绑定（`src/storage.js:108-168` 的 protobufStore）。

持久化全部采用渲染进程 localStorage（`src/storage.js`）：连接列表、连接分组（2026-07 新功能）、设置、搜索历史、自定义 formatter、protobuf 绑定。无主进程侧配置文件、无 SQLite，导出/导入连接也基于 JSON bundle（storage.js:287-319）。

### 5.5 多语言与 CLI 启动

i18n 覆盖 13 种语言：en、cn、tw、tr、ru、pt、de、fr、ua、it、es、ko、vi（`src/i18n/i18n.js:35-88`，langs/ 目录 13 个文件），每份均与 Element UI 对应 locale 合并，保持第三方控件文案同步翻译。

应用支持命令行直启连接：主进程透传 argv（electron-main.js:166-169），渲染进程 `src/addon.js:44-119` 用 getopts 解析 `--host/--port/--auth/--cluster/--ssh-*/--sentinel-*/--ssl-*` 等参数，默认一次性连接、`--save` 才落盘。

## 6. 构建与运行

来自 `README.md` Dev Build 章节与 `package.json` scripts：

```bash
# 开发（webpack-dev-server 热更新于 localhost:9988）
npm install            # electron 下载失败可用 ELECTRON_MIRROR=https://npmmirror.com/mirrors/electron/
npm start              # 前端 dev server
npm run electron       # 另开终端启动 Electron 壳（ARDM_ENV=development）

# 打包
npm run pack:prepare   # npm run build && cp pack/electron/* dist/
npm run pack:win       # Windows（electron-builder -w）
npm run pack:mac       # macOS
npm run pack:linux     # Linux AppImage
npm run pack:win32     # Windows 32 位

# 代码检查
npm run lint           # eslint --ext js,vue src --fix
```

注意 `npm run electron` 只需启动 Electron（`ARDM_ENV=development electron pack/electron`），开发时 UI 从 9988 端口加载（electron-main.js:88-93）。Linux 编译依赖 libfontconfig1-dev（README 已注明）。

## 7. 测试与工程质量

- **测试：无**。package.json 无 test script，仓库无测试目录、无任何测试框架依赖（全文件树核查）；质量完全依赖手动验证与用户 issue 反馈。
- **CI 只做发布不做校验**：`.github/workflows/` 下 build_windows/build_linux/build_mac/build_snap/build_appx 均由 `release: published` 触发打包上传，PR/push 无 CI 检查；codeql-analysis.yml 已改名 `.bak` 停用。另有 publish_winget.yaml 自动发 winget、gen_sponsors.yaml 定时更新 README 赞助者头像。
- **代码质量观察**：eslint（airbnb-base 配置）+ jshint 双配置存在（`.eslintrc.json`、`.jshintrc`）；代码风格统一、注释适度；架构上模块边界清晰（连接层/存储层/组件层分离），但存在 `$parent.$parent.$parent` 这类深层链式取值（KeyList.vue:85、120）等脆弱写法；前端状态靠全局事件总线而非集中式 store，规模继续增长会难以维护。
- **安全注意点**：渲染进程 `nodeIntegration: true` + `contextIsolation: false`（electron-main.js:66-71）、TLS 默认 `rejectUnauthorized: false`（redisClient.js:341）、只读模式依赖前端命令表拦截（可被非命令表写法绕过）——均为可用性优先的取舍，文档未作安全声明。

## 8. 社区与维护状态

- **单人主导**：gh api contributors 显示 qishibo 835 次提交，第 2 名仅 4 次；README 赞助者含 DragonflyDB 等机构头像，OpenCollective + GitHub Sponsors + 双商店付费构成收入渠道。
- **活跃度**：gh api `pushed_at` 2026-11-17，仓库持续有提交；open issues 165，watchers 263，forks 2,719。
- **发版节奏**：2024 年 4 个版本（v1.6.4→v1.7.1），2025 年发布断档约 10 个月（README Feature Log 2025-10-01 写有 "Long time no see! New features will coming soon!"），2026 年明显复苏：v1.7.2（2026-07-29，连接分组）、v1.7.4（2026-08-18，TimeSeries/Vector/Array 类型 + 测试连接功能），Feature Log 与 release 一一对应（`README.md:76-123`、gh api releases）。
- **分发广度**：每个 release 附 7-17 个安装包资产（gh api `assets` 数量从 7 增至 17，对应新增 arm64 与商店包），覆盖 win x64/arm64、mac x64/arm64（dmg + App Store + 公证 notarize.js）、Linux AppImage x64/arm64 + snap + Appx + winget + choco + brew。
- **issue 语言生态**：作为中文开发者项目，gitee 镜像与微信赞助码（README.md:71-73）显示相当比例中国用户；官方 issue 模板仅一个简单 markdown（`.github/ISSUE_TEMPLATE.md`），无 PR 模板（未确认贡献指引，仓库无 CONTRIBUTING 文件）。

## 9. 亮点与风险

**亮点**

1. 大数据量稳定性设计扎实：SCAN 流式分页 + Buffer 传输 + fork 虚拟滚动树三层配合（KeyList.vue / KeyListVirtualTree.vue / util.js keysToTree），是同类客户端中少见的针对百万 key 场景的工程化处理。
2. 连接拓扑覆盖完整且实现细节考究：SSH×{standalone, Cluster, Sentinel} 组合逐一处理（每节点独立隧道 + natMap），Redis 6 ACL、只读模式、TLS、IPv6、双栈命令行启动均有支持（src/redisClient.js、src/addon.js）。
3. 值格式兼容矩阵宽：15 种查看器 + 自定义外部脚本解码 + protobuf profile 绑定 key（src/components/viewers/、storage.js），覆盖 PHP/Java/Pickle/Protobuf 等后端生态常见序列化。
4. 架构精简：主进程 250 行薄壳、渲染进程直连 Redis、无重框架，代码量小（约 5,000 行量级业务代码）易于阅读和改动。
5. 13 语言国际化、全平台（含 arm64）打包、国内 gitee 镜像，面向全球用户的开箱体验完善。

**风险/局限**

1. 零测试、CI 无校验环节，单人维护是单点风险；2025 年曾断档约 10 个月，可持续性依赖作者个人投入。
2. 技术栈老化：Electron 12（2021 年版本，早已停止安全更新）、Vue 2（已 EOL）、webpack 4、Node 14 CI 镜像，升级成本随时间增大。
3. 安全取舍明显：nodeIntegration + 关闭 contextIsolation、TLS 不校验证书、只读模式仅前端拦截，不适合对安全边界要求高的环境当「事实标准」看待。
4. 连接配置存于 localStorage 且明文（storage.js 直接 JSON.stringify 连接含密码），换机迁移需手动导出，亦无加密保护（未确认是否有系统级加密）。

## 10. 总结评价

ARDM 是「单人把一个 GUI 工具做到全球 3 万+ 星」的典型样本：功能覆盖不输商业产品（四种连接拓扑、10 种数据类型、15 种值格式、13 种语言、六平台打包），又用 SCAN 流式加载 + 虚拟滚动在「海量 key 不卡死」这一具体痛点上建立了口碑，MIT 免费与商店付费赞助并行的模式也跑通了。它没有测试、没有 TypeScript、没有新框架，工程质量依靠清晰的分层与克制的依赖维持——对普通开发者与运维，这是当前免费 Redis GUI 中综合体验最顺手的之一；对想学习 Electron 应用组织的开发者，它薄主进程 + 渲染进程直连的架构、sendCommand 补丁式的横切设计、以及流式大数据渲染的实现，都是值得精读的实战范例；对需要在生产环境严格管控安全的团队，则应评估其 Electron 版本与安全取舍后再引入。
