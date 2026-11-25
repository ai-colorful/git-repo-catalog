# Unitech/pm2 项目分析

> 星数 42600（快照 2026-10-24）｜主语言 JavaScript (TypeScript)｜档位 20k-50k｜分析日期 2026-11-06

## 1. 项目速览

一句话定位：Node.js 生态事实标准的生产级进程管理器——把应用守护化、崩溃自动拉起、多核 Cluster 负载均衡与零停机热重载打包进一个 CLI。

| 项目 | 内容 |
|---|---|
| 名称 | Unitech/pm2（npm 包 `pm2`） |
| 版本 | 7.1.2（package.json，CHANGELOG 最新条目一致） |
| 星数 | 42600（快照），gh api 实测 43325 |
| 主语言 | JavaScript（CommonJS），代码库本身无 TypeScript 重构，见第 5 节 |
| 许可证 | AGPL-3.0（LICENSE 指向 GNU-AGPL-3.0.txt） |
| 运行要求 | Node.js >= 18，同时支持 Bun 1.1.25+（package.json engines、binaries/CLI.js 版本检查） |
| 最近活跃 | gh api pushed_at 2026-11-03（3 天前），维护极其活跃 |
| 规模 | lib/ 约 2.24 万行 JS，modules/（内嵌依赖）约 3.8 万行，测试约 1.85 万行 |

## 2. 项目是什么，解决什么问题

PM2（Process Manager 2）解决的是 Node.js 应用"上了生产怎么办"的完整问题链。Node 进程默认单线程、异常即退出，直接 `node app.js` 无法满足生产要求。PM2 用一条命令把应用变成守护进程：`pm2 start app.js` 之后由常驻守护进程持续监控，进程退出或超内存就自动重启（README "Managing Applications"、lib/God.js `handleExit`）。

第二层能力是多核利用与负载均衡。`pm2 start api.js -i max` 以 cluster 模式按 CPU 数 fork 多个 worker，HTTP/TCP 连接由 Node cluster 内建的 round-robin 调度分发（lib/God/ClusterMode.js 注释 "It will wrap the code and enable load-balancing mode"；多实例循环见 lib/God.js `God.prepare` 的 `timesLimit`）。配合 `pm2 reload all` 实现零停机热重载：新 worker 先启动并监听成功后，老 worker 才收到 `shutdown` 消息优雅退出（lib/God/Reload.js）。

第三层是运维工具面：日志聚合与轮转（`pm2 logs`、`pm2 flush`、logrotate 模板）、开机自启脚本生成（`pm2 startup` 自动识别 systemd/upstart/openrc/launchd，lib/API/Startup.js）、cron 定时重启（lib/Worker.js 用 croner 注册）、文件变更自动重启（lib/Watcher.js + 内嵌 chokidar）、`pm2 monit` 终端监控面板（lib/API/Monit.js + 内嵌 blessed）。

第四层是商业监控闭环（Keymetrics / PM2 Plus）：`pm2 link` 后由独立 Interactor 守护进程通过 WebSocket 把指标推到 pm2.io，并接收反向指令（modules/pm2-io-agent/src/InteractorDaemon.js，Push/Reverse 双向通道）。应用进程内部还会注入 pmx 探针（lib/ProcessUtils.js `injectModules` → modules/pm2-io-bpm），支持自定义指标与触发式函数。此外 pm2-runtime 作为 `node` 的 drop-in 替换专为容器设计（lib/binaries/Runtime4Docker.js，前台运行、PM2_DISCRETE_MODE）。

它也兼容多语言与多种应用形态：interpreter 机制可跑 Python/Ruby/二进制（test/e2e/cli/python-support.sh 有覆盖），ESM（.mjs 与 package.json type:module 检测，lib/ProcessUtils.js `isESModule`）、.ts 应用、Bun 均支持。

## 3. 技术栈与依赖

- 语言：纯 JavaScript（CommonJS，无构建步骤，`npm install -g` 直接可用）；types/index.d.ts 提供编程式 API 的 TypeScript 类型定义（package.json `"types": "types/index.d.ts"`）。
- 关键运行时依赖（package.json）：
  - `async`（并发控制：timesLimit/eachLimit/series）、`commander`（CLI 解析）、`ansis`（终端着色）、`cli-tableau`（进程列表表格）
  - `croner`（cron 表达式）、`dayjs`（日志时间戳）、`semver`（版本判断）、`pidusage`（跨平台 CPU/内存采集）
  - `ws`（Interactor WebSocket 传输）、`js-yaml`（YAML 配置解析）、`@pm2/js-api`（PM2 Plus 客户端）、`tx2`（自定义指标协议）、`pm2-deploy`（远程部署）
- 内嵌（vendored）第三方：`modules/` 下有 pm2-axon、pm2-axon-rpc（RPC/事件总线）、chokidar（文件监听）、blessed（TUI）、vizion（git 仓库元信息解析）、pm2-io-agent、pm2-io-bpm（pmx 探针）——均有 VENDORED.md 记录，减少 npm 依赖漂移（CHANGELOG 7.1.0 明确 "@pm2/blessed is now vendored"）。
- 可选依赖：fsevents（macOS 文件监听）。
- devDependencies 极简：mocha、should、express、chokidar。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `bin/` | 4 个入口：pm2、pm2-dev、pm2-docker、pm2-runtime（+ PowerShell 版 pm2.ps1），均只 require lib/binaries/ 下对应 CLI |
| `lib/` | 核心库（约 2.24 万行）：API.js（CLI 命令实现，1980 行）、Client.js（CLI↔守护进程 RPC 客户端）、Daemon.js（守护进程骨架）、God.js + God/（进程管理内核）、ProcessContainer*.js（worker 包裹器）、Worker.js（周期任务）、Watcher.js、API/（子命令模块：Startup/Log/LogManagement/Monit/Dashboard/Modules/Deploy/pm2-plus 等） |
| `lib/binaries/` | CLI.js（全局命令）、DevCLI.js（pm2-dev）、Runtime.js 与 Runtime4Docker.js（容器前台模式） |
| `lib/God/` | ActionMethods（955 行，RPC 动作实现）、Reload（软/硬重载）、ClusterMode/ForkMode（两种执行模式）、Methods（kill/treekill 等基础操作）、StdioLogger（cluster worker 日志接管） |
| `modules/` | 内嵌依赖（axon RPC、chokidar、blessed、vizion、pm2-io-agent、pm2-io-bpm 等） |
| `types/` | 编程式 API 的 .d.ts |
| `test/` | programmatic（mocha 单测）、e2e（bash 脚本）、interface、fixtures、benchmarks、systemd |
| `examples/` | 25+ 个可运行示例（cluster-http、esm、docker-pm2、ts 等） |
| `packager/` | deb/rpm/alpine 打包脚本 |
| `lib/templates/` | ecosystem 配置模板、init 启动脚本模板、logrotate 模板 |

组织方式评述：单体仓库 + 轻量 vendoring。没有 monorepo 工具链，但把核心依赖 vendor 进 modules/ 自持；lib/God 采用"命名空间混入"模式（`require('./God/xxx')(God)` 逐个挂载方法），API 子命令同样以 `CLI.prototype.xxx` 混入（lib/API/LogManagement.js 等），是典型的原型式插件组织，代码零编译直发 npm。

## 5. 核心架构与关键实现

### 5.1 双进程架构：CLI 客户端 + God 守护进程

PM2 是经典 C/S：CLI 进程（lib/binaries/CLI.js）通过 lib/Client.js 连接常驻守护进程（首次执行时自动 fork）。通信走内嵌 axon 的两路 Unix socket（Windows 用命名管道）：`rep` 承载请求-响应 RPC，`pub-emitter` 承载实时事件总线（lib/Daemon.js `innerStart`）。守护进程把约 30 个 God 方法直接暴露为 RPC 端点（lib/Daemon.js `server.expose`：prepare、restartProcessId、reloadProcessId、getMonitorData 等）。CLI 侧所有操作本质都是"连上 RPC 调一个方法"（lib/Client.js `start` → `pingDaemon` → `launchRPC`）。

守护进程自身用 domain 包住全局错误做自愈：捕获致命错误后 detached 重新 spawn `pm2 update` 再退出（lib/Daemon.js `start`），保证守护进程不因单次异常永久挂掉。

```js
// lib/Daemon.js —— RPC 端点直接绑定 God 方法（节选）
server.expose({
  killMe                  : that.close.bind(this),
  profileCPU              : profile.bind(this, 'cpu'),
  prepare                 : God.prepare,
  getMonitorData          : God.getMonitorData,
  restartProcessId        : God.restartProcessId,
  reloadProcessId         : God.reloadProcessId,
  ...
});
```

### 5.2 God 内核：进程注册表与生命周期/重启策略

God（lib/God.js）是守护进程内核：`clusters_db` 是以 pm_id 为键的内存进程注册表，`God.bus`（eventemitter2 通配符总线）广播进程事件。新进程经 `God.prepare`（决定单实例还是按 instances 复制，`-i max`→numCPUs、负数→CPU 数减 X，lib/God.js:134-140）进入 `God.executeApp`：分配 pm_id、生成日志路径、注册 cron 与 watch，再按 exec_mode 分派到 ClusterMode.nodeApp 或 ForkMode。

退出处理 `God.handleExit`（lib/God.js:417）是重启策略核心：
- `stop_exit_codes` 命中、`autorestart:false`、正在 stopping 则不重启；
- "不稳定重启"判定：进程存活时间 < `min_uptime`（默认 1000ms）则 unstable_restarts 计数，达到 `max_restarts`（默认 16）标记 ERRORED 并停止拉起，防止 crash 死循环；
- `restart_delay` 固定延迟重启；`exp_backoff_restart_delay` 指数退避：延迟每次 ×1.5、上限 15 秒（lib/God.js:524-533）。
- 细节防御：陈旧 exit 事件（上一代进程）与当前 occupant 比对后忽略（Windows shutdown_with_message 场景，#6142）。

周期任务由 lib/Worker.js 承担：默认每 30s（WORKER_INTERVAL）轮询 monit 数据，超过 `max_memory_restart` 就触发 reload（`maxMemoryRestart`），cron 重启也在此注册（croner 实现）。

### 5.3 Cluster 模式与零停机 reload

Cluster 模式下守护进程调用 `cluster.setupMaster` 把 worker 入口指向 lib/ProcessContainer.js 并设 `silent: true`（lib/God.js:36-40），worker 的 stdout/stderr 由 lib/God/StdioLogger.js 在守护进程侧统一接管写日志文件（ClusterMode.js 中大段注释解释了 #3675/#4719 等 pino/inherit 场景写串 pm2.log 的历史问题）。负载均衡本身复用 Node cluster 内建 round-robin，PM2 负责实例数量管理与 `pm2 scale`（lib/API.js:1788）。

零停机 reload 在 lib/God/Reload.js 实现，思路是"_old_ 槽位停车"：老 worker 先挪到 `_old_<id>` 键下腾出 id，新 worker 启动；软重载（softReload）等新 worker `listening` 或应用显式 `process.send('ready')`（wait_ready 模式）后，才向老 worker 发 `shutdown` 并在 GRACEFUL_TIMEOUT（默认 8s，constants.js）兜底强删；替换失败还有 `unparkOldWorker` 把老 worker 挂回原 id，避免槽位永久占用。

```js
// lib/God/Reload.js —— softReload：新 worker 就绪后才退役老 worker（节选）
var softCleanDeleteProcess = function () {
  if (started) return false;
  started = true;
  var cleanUp = function () {
    clearTimeout(timer);
    return God.deleteProcessId(t_key, cb);   // t_key = '_old_' + id
  };
  old_worker.once('disconnect', cleanUp);
  old_worker.send && old_worker.send('shutdown');
  ...
};
```

### 5.4 日志链路与配置体系

进程内包裹器 lib/ProcessContainer.js hook `process.stdout/stderr.write`：日志既通过 IPC `process.send({type:'log:out'|'log:err'})` 送进守护进程事件总线（供 `pm2 logs` 实时输出、Dashboard 用），又写本地文件（支持 `--log-date-format` 时间戳前缀与 `--log-type json` 结构化）。轮转有三层：`pm2 flush` 清空、`pm2 reloadLogs` 重开句柄、`pm2 logrotate` 安装系统 logrotate 配置（lib/API/LogManagement.js，模板 lib/templates/logrotate.d/pm2：weekly、rotate 12、compress、copytruncate），更细粒度的容量轮转交给外部模块 `pm2 install pm2-logrotate`（README "Log Management"）。

配置体系：ecosystem.config.js 为默认入口，同时支持 json/json5/yaml（lib/Common.js:267 `parseConfig`，JS 配置经 vm沙箱/require 求值）；全量参数 schema 在 lib/API/schema.json（375 行，带 docDefault/docDescription 供 `pm2 init simple` 生成注释模板），别名归一（fork↔fork_mode、interpreter↔exec_interpreter）。环境切换用 `env_<name>` 差异段（test/e2e/process-file/yaml-configuration.sh 有覆盖）。7.1.1 修复了 shell 环境变量同名覆盖 PM2 配置的问题（lib/God.js `Utility.extendEnv` + lib/tools/pm2-config-keys.js `filterShellEnv`）。

### 5.5 PM2 Plus/Keymetrics 商业闭环与 TS 支持

监控闭环三条链：① `pm2 link` 用 OAuth（macOS/Windows 桌面走 Web 策略、Linux SSH 走 CLI 设备码，lib/API/pm2-plus/PM2IO.js `strategy()`）绑定 pm2.io 账户；② Interactor 守护进程（modules/pm2-io-agent/src/InteractorDaemon.js）经 WebsocketTransport 上行推送、ReverseInteractor 接收下行命令；③ 每个应用进程启动时注入 pmx 探针（lib/ProcessUtils.js `injectModules` → modules/pm2-io-bpm），采集自定义指标/动作（axm_monitor/axm_actions 字段在 lib/God.js executeApp 中初始化）。开源版数据面（getMonitorData、pm2 monit）完整可用，PM2 Plus 只是把同一数据面聚合到云端，不锁本地功能。

TypeScript"重构"实测结论：代码库本体没有任何 TS 迁移（全仓 .ts 文件仅 examples/ 与 types/index.d.ts）。TS 支持是面向用户应用的：Node >= 22.18/23.6 原生类型剥离免配置、22.6+ 自动补 `--experimental-strip-types` 进 execArgv（lib/tools/typestrip.js `supportLevel`，cluster 模式也处理了，lib/God/ClusterMode.js:53-57），旧版本回退到从应用依赖解析 ts-node/register（lib/ProcessUtils.js `enableTypeScript`），.tsx 仍需 ts-node。另有 lib/OtelManager.js 按需安装钉版 OpenTelemetry 包做链路追踪（--deep-monitoring）。

## 6. 构建与运行

无构建步骤（源码即发布物）。来自 package.json scripts 与 test/*.sh：

```bash
npm install pm2 -g          # 安装（preferGlobal）
pm2 start app.js            # 基本使用
pm2 start api.js -i max     # cluster 模式全核
pm2 reload all              # 零停机热重载
pm2 startup / pm2 save      # 开机自启 + 进程列表持久化（resurrect）
pm2 monit / pm2 logs        # 终端监控 / 日志

# 测试（package.json scripts）
npm run test:unit           # bash test/unit.sh：mocha 逐文件跑、失败自动重试一次
npm run test:e2e            # bash test/e2e.sh：60+ bash e2e 场景
npm run test:parallel       # bash test/docker-parallel.sh：Docker 并行矩阵（CI 用）
npm run test:windows        # bash test/windows.sh
mocha 'modules/pm2-io-bpm/test/**/*.spec.js'  # 内嵌模块子测试（test:bpm/test:axon/test:io-agent）
```

## 7. 测试与工程质量

- 测试规模：test/ 下约 204 个 JS 文件、约 1.85 万行（含 bash e2e）。unit.sh 显式列出约 55 个 programmatic 用例（auto_restart、cluster、graceful、signals、exp_backoff、filter_env、dump/resurrect、历史 issue 回归如 issue_6089_prototype_pollution、issue_6075_redos），e2e.sh 约 60 个 bash 场景（CLI、process-file、logs、modules、internals、startup），另有 interface/（事件总线）、systemd/（pm2 update under unit #5000）、benchmarks/。
- 框架与风格：mocha + should（.mocharc.js：bail、timeout 10s、retries 2）；单测以"拉起真守护进程→断言 RPC 行为"的集成式为主，e2e 全走真实 CLI；unit.sh 对不稳定用例有失败重试机制（脚本内置 reset + 二次运行）。
- CI（.github/workflows/node.js.yml）：四个 job——Node 18/20/24 Docker 并行矩阵、Bun 专用 job、Windows（node 18/20/latest）job、systemd 专项 job；另有 docker-images.yml 维护 keymetrics/pm2 镜像。
- 质量观察：CHANGELOG 196 个版本条目且写法极严谨（每条带 issue 号与 breaking change 段）；近期修复涉及大量并发/平台边角（stale exit event、_tree_pids 泄漏、fd 泄漏），注释中频繁引用 issue 编号说明防御性代码成因，是"生产踩坑驱动"的工程文化。ESLint/格式化配置未见（未确认是否依赖编辑器约定）。

## 8. 社区与维护状态

- 活跃度：gh api pushed_at 2026-11-03；CHANGELOG 顶部 7.1.2/7.1.1/7.1.0 均为近期密集修复（monit TUI、filter_env 重做、Windows pidusage 等），主分支持续有提交。
- 版本节奏：CHANGELOG 196 个版本条目，长期稳定发版（7.x 大版本仍在演进，7.1.2 引入 breaking change 也如实标注）。
- 贡献者结构：package.json 列 19 位核心贡献者，maintainer 为作者 Alexandre Strzelewicz（pm2.io 创始人）；仓库 forks 2731、open issues 1084（gh api），issue 堆积较大但 CHANGELOG 显示核心团队持续消化。
- 商业背景：PM2 Plus/Keymetrics 为商业产品，开源进程管理器 + 云监控的常见双轨模式。

## 9. 亮点与风险

亮点：
1. 架构成熟可靠：CLI↔守护进程 RPC + 内存注册表 + 事件总线的三层设计十年验证，守护进程还有 domain 级自愈重启（lib/Daemon.js）。
2. 零停机 reload 实现精细：_old_ 槽位停车、wait_ready 握手、GRACEFUL_TIMEOUT 兜底、失败回挂（lib/God/Reload.js），边界情况覆盖充分。
3. 零构建、强自持：CommonJS 直发 + 关键依赖 vendored（modules/），安装即用，供应链可控。
4. 测试真实且厚：约 1.85 万行集成式测试 + 四平台 CI（Linux/Bun/Windows/systemd），大量历史 issue 回归用例。
5. TS 用户零门槛：Node 原生 type stripping 自动适配，旧 Node 回退 ts-node（lib/tools/typestrip.js、lib/ProcessUtils.js）。

风险/局限：
1. AGPL-3.0 许可证对以库形式集成的商业场景有传染性顾虑（LICENSE/GNU-AGPL-3.0.txt），多数公司只在 CLI 层使用规避。
2. 代码风格偏老：原型混入、var/回调为主、API.js 单文件近 2000 行，新人上手曲线陡。
3. open issues 1084 高位，非核心 issue（平台长尾、生态模块）响应可能滞后。
4. 兼容性包袱重：Windows/Bun/多 Node 版本矩阵导致大量平台特判（ClusterMode.js 中 Bun 500ms 兜底 online 等 workaround），可能引入回归（CHANGELOG 的 breaking change 段已多次出现）。

## 10. 总结评价

PM2 是 Node.js 服务端运维十余年经验的结晶：以一个纯 JS、零构建的单仓库，提供了守护进程、自动重启、cluster 负载均衡、零停机热重载、日志、开机自启、容器模式到云端监控的完整生产闭环，且至今仍在高频迭代（v7.1.2，3 天前有 push）。它不是最现代的代码库（原型混入、巨型文件），但它的边界处理（退出事件竞态、fd 泄漏、平台差异）正是进程管理器最难写对的部分。适合所有需要在裸机/VM/容器里跑 Node 应用的团队作为默认生产工具；对想做进程管理器、守护进程系统或研究 Node cluster 进阶用法的人来说，lib/God.js + lib/God/Reload.js + lib/ProcessContainer.js 是极有价值的参考实现。
