# GitSquared/edex-ui 项目分析

> 星数 41500（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：一款全屏、跨平台的"科幻电影风"终端模拟器 + 系统监视器，把 TRON Legacy 式的界面做成了真正可用的日常工具。

| 项 | 值 |
| --- | --- |
| 名称 | eDEX-UI（GitSquared/edex-ui） |
| 星数 | 41500（快照 2026-10-24；gh api 实时值 45047） |
| 语言 | JavaScript（Electron 桌面应用，无框架，原生 DOM） |
| 许可证 | GPL-3.0（LICENSE） |
| 版本 | v2.2.8（最终版，package.json version 2.2.8） |
| 归档状态 | **已归档（EOL）**：README 徽章与 v2.2.8 release 均明示 "Project archived oct. 18th 2021"；gh api `archived: true`，`pushed_at: 2021-10-19T22:38:47Z` |
| 贡献者 | 约 63 人（contributors API 含匿名，per_page=1 分页 last=64）；v2.2.8 release 说明称三年间 46 位贡献者参与 |
| 分支 | master（默认分支，无后续提交） |

注意：任务备注中提到 "EOL 2022"，经 gh api 核实，实际归档时间为 2021 年 10 月（README 第 17 行、v2.2.8 release published_at 2021-10-18、pushed_at 2021-10-19 三处证据一致）。

## 2. 项目是什么，解决什么问题

eDEX-UI 是一个"看起来像好莱坞电影里黑客界面"的桌面应用：全屏运行，中间是一个功能完整的终端（基于 xterm.js），左右两列是实时系统与网络监控面板（CPU 曲线、内存点阵、进程 Top 列表、网络状态、GeoIP 全球连线地球仪），底部是屏幕虚拟键盘。它起源于 r/unixporn 上的 dex-ui 艺术项目，作者的定位是"少一点艺术、多一点可分发软件"（README），即把科幻 UX 从概念图变成日常可用的程序。

它解决的问题并非"更好的终端"（这方面它远不如专业终端），而是"有真实功能的科幻界面"：终端是真的 shell（通过 node-pty 启动用户配置的 bash/powershell），监控数据是真的（systeminformation 采集），地球仪上的连线是真的 GeoIP 定位（maxmind + geolite2）。README 自嘲"这可能是个开过头了的玩笑"（It might or might not be a joke taken too seriously），这种"功能性艺术项目"定位在 v2.2.8 归档公告中被作者再次确认。

特色场景包括：触屏设备支持（完整的屏幕键盘 + 触摸滚动终端，README Features；tarball 内 19 种键盘布局 JSON）、CSS/主题深度定制（21 套内置主题，主题可注入自定义 CSS）、以及作者请音效设计师专门制作的 13 个操作音效（src/assets/audio/*.wav）营造"黑客氛围"。

由于已归档，它的现实意义更多是：一个完整展示"Electron + node-pty + xterm.js 终端三件套"如何组装的参考实现，以及一个 UI/动效工程化的教科书案例（启动引导动画、面板渐入、故障艺术标题等全部用原生 DOM + CSS 实现）。

## 3. 技术栈与依赖

- 运行时：Electron ^12.1.0（2021 年版本），Node 14（CI 指定，.github/workflows/build-binaries.yaml）
- 终端：`xterm` 4.14.1 + 官方插件 attach/fit/ligatures/webgl（src/src/package.json）
- PTY：`node-pty` 0.10.1（原生模块，需 electron-rebuild 重编译，根 package.json install-* 脚本）
- 系统信息：`systeminformation` 5.9.7（CPU/RAM/网络/进程/电池等统一采集）
- 数据桥：`ws` 7.5.5（主进程内起本地 WebSocket 服务器转发 PTY 字节流）
- 曲线图：`smoothie` 1.35.0（实时流式折线图）；地球仪：`encom-globe`（已 vendor 进 src/src/assets/vendor/）
- GeoIP：`geolite2-redist` + `maxmind`（src/src/classes/netstat.class.js）
- 音效：`howler` 2.2.3；UI 立体边框：`augmented-ui`；温度（macOS）：`osx-temperature-sensor`（可选依赖）
- 其他：`pdfjs-dist`（文档阅读器模块）、`shell-env`（启动时获取登录 shell 的干净环境变量）、`signale`（日志）、`nanoid`（IPC 请求 ID）
- 构建链：`electron-builder`（AppImage/DMG/NSIS）+ `electron-rebuild`（原生模块）；发布前用 `terser`/`clean-css`/`node-json-minify` 做源码压缩（prebuild-minify.js）
- 双 package.json 结构：根目录管理构建工具链，src/src/ 管理运行时依赖（Electron 传统做法）

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| src/_boot.js | Electron 主进程入口：单实例锁、默认配置写入、资产镜像、窗口创建、TTY 管理（375 行） |
| src/_renderer.js | 渲染进程引导：安全助手、主题加载、系统信息 IPC 代理、启动动画、initUI 装配全部模块（1152 行） |
| src/_multithread.js | systeminformation 调用的 cluster 多线程分发器（89 行） |
| src/ui.html | 单页 UI 骨架 + CSP 策略 |
| src/classes/*.class.js | 19 个 UI 模块类（terminal、filesystem、keyboard、cpuinfo、ramwatcher、netstat、locationGlobe、toplist、sysinfo、clock、conninfo、fuzzyFinder、mediaPlayer、docReader、modal、audiofx、hardwareInspector、updateChecker） |
| src/assets/themes/ | 21 套主题 JSON（tron/blade/nord/matrix/cyborg 等，含 -notype/-disrupted 变体） |
| src/assets/kb_layouts/ | 19 种屏幕键盘布局 JSON（QWERTY/DVORAK/COLEMAK/WORKMAN/BEPO 等） |
| src/assets/audio/ | 13 个音效 wav（按键、stdout、报错、授权通过等） |
| src/assets/css/ | 每个模块一个 CSS 文件的拆分式样式 |
| file-icons/ | 6 个图标字体 git submodule（tarball 不含），由 file-icons-generator.js 生成 src/assets/icons/file-icons.json |
| media/ | 截图、logo、应用图标 |
| .github/workflows/ | build-binaries.yaml（五平台打包）+ codeql-analysis.yml |

组织方式评述：单包单体应用，但内部划分非常清晰——"一个类一个面板模块"，模块通过 `window.mods.*` 注册、类文件与 CSS 一一对应（src/ui.html、src/assets/css/mod_*.css）。没有打包器（无 webpack/babel），直接以 CommonJS require 在浏览器环境运行，靠发布前 minify 脚本压缩。核心 JS 约 6200 行（src/src/classes + 3 个 _*.js，wc -l 统计），规模克制。

## 5. 核心架构与关键实现

### 5.1 终端：主进程 PTY 与渲染进程 xterm 的 WebSocket 桥（src/src/classes/terminal.class.js）

Terminal 是一个双角色类（489 行），同一份代码按 `opts.role` 走两条完全不同的分支：

- **server 角色**（主进程）：node-pty 启动 shell（Windows 默认 powershell.exe，其他 bash --login，src/src/_boot.js 默认配置），并在 127.0.0.1 上起一个 ws WebSocket 服务器（端口 3000，verifyClient 限制单客户端）。PTY 数据 → `ws.send(data)` 直推渲染进程；渲染进程发来的字节 → `tty.write(msg)`。
- **client 角色**（渲染进程）：xterm.js 实例加载 AttachAddon 绑定该 WebSocket、WebglAddon 渲染、LigaturesAddon 连字；主题未显式指定 16 色时用 color.js 以 `grayscale().mix(主题色, 0.3)` 自动派生一套和谐配色。

数据面走 WebSocket（低延迟字节流），控制面走 Electron IPC（`terminal_channel-<port>` 通道传 resize、cwd 变更、进程名）。CWD 追踪是亮点：每秒 tick，Linux 读 `/proc/<pid>/cwd` 符号链接，macOS 用 `lsof -a -d cwd -p <pid>`，Windows 明确不支持而回退 "Fallback cwd" 分离模式——这解释了 README 中"Windows 文件浏览器显示 Tracking Failed"的说明。

```js
// src/src/classes/terminal.class.js:318-332（节选）
this._getTtyCWD = tty => {
    return new Promise((resolve, reject) => {
        let pid = tty._pid;
        switch(require("os").type()) {
            case "Linux":
                require("fs").readlink(`/proc/${pid}/cwd`, (e, cwd) => { /* ... */ });
                break;
            case "Darwin":
                require("child_process").exec(`lsof -a -d cwd -p ${pid} | tail -1 | awk ...`);
                break;
            default:
                reject("Unsupported OS");
        }
    });
};
```

标签页支持：主进程预分配 4 个额外 TTY 端口（_boot.js:276-283），每开一个标签 spawn 一个新 Terminal server，上限 5 个标签。

### 5.2 系统监控的多线程化：cluster 分发器（src/src/_multithread.js）

systeminformation 的部分调用（如 processes、网络探测）开销大，会阻塞主进程。方案：主进程用 Node `cluster` fork 至多 7 个 worker（CPU 核数-1，上限 7，见 "See #904" 注释），渲染进程通过 `window.si` Proxy（_renderer.js:178-201）发起调用——Proxy 拦截属性访问生成函数，每次调用生成 nanoid 作为请求 ID，经 IPC 转发到 master，master 轮询分发给 worker，结果按 ID 回送：

```js
// src/src/_renderer.js:181-197（节选）
window.si = new Proxy({}, {
    get: (target, prop, receiver) => {
        return function(...args) {
            return new Promise((resolve, reject) => {
                let id = nanoid();
                ipc.once("systeminformation-reply-"+id, (e, res) => { resolve(res); });
                ipc.send("systeminformation-call", prop, id, ...args);
            });
        };
    }
});
```

这个 Proxy 让渲染层代码可以像直接调 si.mem() 一样写，多线程细节完全透明。

### 5.3 监控可视化：点阵内存图与实时曲线（src/src/classes/ramwatcher.class.js、cpuinfo.class.js）

RAMwatcher 是最具辨识度的模块：构造时生成 440 个 `<div>` 点并随机洗牌（让"占用"看起来像噪点扩散而非整齐填充），每次更新（1.5s 间隔）按 active/available/free 比例给前三段点设置 class，由 CSS 过渡呈现动画；swap 用 `<progress>` 条显示。CPU 模块用 smoothie 库画两条实时流式曲线（各半数核心），并展示温度/频率/任务数。这种"纯 DOM + CSS class 切换"的可视化不依赖任何图表库（曲线除外），是低依赖高表现力的典型做法。

### 5.4 主题系统与触屏键盘（src/assets/themes/*.json、src/classes/keyboard.class.js）

主题 = 一个 JSON（src/src/assets/themes/tron.json 为范本：colors/cssvars/terminal/globe 四段）。启动时主进程把内置主题镜像到 userData/themes（_boot.js:128-145），用户可直接编辑目录里的 JSON 换肤。加载时（_renderer.js:89-140）将主题值写入 CSS 自定义属性生成 `<style class="theming">`，字体用 FontFace API 从 userData/fonts 动态注册，`theme.injectCSS` 经 `_purifyCSS`（剥除 `<`）净化后注入；渲染进程收到主题后还会驱动终端 16 色/colorFilter 派生（terminal.class.js:33-96）。主题/键盘均支持运行时热切换（_boot.js:332-346 的 setThemeOverride IPC + 页面重载）。

屏幕键盘（keyboard.class.js，1291 行，全项目最大单文件）从布局 JSON 构建 DOM，用 `dataset.isShiftOn` 等状态属性驱动 CSS 修饰键高亮；与 shortcuts.json 定义的快捷键系统集成（Ctrl+Shift+P 密码输入模式等）。触屏适配散落各处：终端容器监听 touchstart/touchmove 实现手势滚动、xterm 隐藏 textarea 设 readonly 防止弹出系统软键盘（terminal.class.js:148-149，"See #733"）。

## 6. 构建与运行

来自根 package.json scripts 与 README "Useful commands for the nerds"：

```bash
# 从源码运行（Linux/macOS 需 Xcode CLT；Windows 需管理员 CMD）
npm run install-linux    # 或 install-windows：装依赖 + electron-rebuild 重编 node-pty
npm run start            # electron src --nointro

# 构建（只能构建宿主 OS 的目标；产物在 dist/）
npm install              # 注意不是 install-linux
npm run build-linux      # -l --x64 --ia32 --arm64 --armv7l，AppImage
npm run build-windows    # NSIS 安装器
npm run build-darwin     # DMG

# 测试（实际是 snyk 安全扫描，无单元测试）
npm run test             # rsync 源码 → prebuild-minify → npm install → snyk test
```

构建流程细节：prebuild-* 先把 src/ 复制到 prebuild-src（排除 node_modules）再跑 prebuild-minify.js（terser 压 JS、clean-css 压 CSS、node-json-minify 压主题 JSON），electron-builder 以 prebuild-src 为 app 目录打包（根 package.json build.directories）。

## 7. 测试与工程质量

- **测试规模：零单元/集成测试**。仓库无 test 目录；package.json 的 test 脚本是 snyk 依赖漏洞扫描（根 package.json:29）；另有 CodeQL 静态分析 workflow（.github/workflows/codeql-analysis.yml）。对 2021 年的"功能型艺术项目"来说这是有意识的取舍，但也意味着原生模块升级全靠手工验证。
- **CI**：build-binaries.yaml 相当完善——Linux x64/ia32、armv7l/arm64（QEMU docker 容器内构建）、Windows NSIS、macOS DMG 五路产物，配 npm/Electron 双缓存；每次 push/PR 都出 nightly 制品（README "Getting the bleeding edge"）。
- **安全意识**（src/ui.html、src/_renderer.js、src/_boot.js）：CSP 头（default-src file: + connect-src ws:）、禁用 eval、全局 HTML 转义助手 `_escapeHtml`、主题 CSS 注入前 `_purifyCSS` 净化、禁止新窗口与页面跳转（_boot.js:349-360）、单实例锁。但 Electron 12 + `nodeIntegration: true` + `contextIsolation: false`（_boot.js:193-203）是如今视角下的遗留风险配置，渲染进程被攻破即等于拿到 Node 权限——缓解手段仅靠输入面净化。
- 代码风格统一（单类单文件、注释引用 issue 编号如 "See #733/#904/#302"），LGTM 徽章（README）说明曾接静态扫描。无 linter 配置文件（未确认）。

## 8. 社区与维护状态

- **已归档，确认无争议**：gh api `archived: true`；最后 push 2021-10-19；v2.2.8 release（2021-10-18）标题即 "eDEX-UI v2.2.8, and archiving the project"，正文 "So Long, And Thanks For All The Fish" 说明归档原因：作者从学生转为全职工程师、无暇维护，且认为代码库急需重构而项目定位是"功能性艺术作品"（limited usefulness），选择体面退场。三年 16 个版本、约 40 万次下载、46 位贡献者。
- **归档前活跃度**：稳定的一年数更节奏（v2.2.x 系列），PR 贡献者大量出现在 release notes（键盘布局、主题多为社区贡献：nl-BE #1118、sv-SE #1172、cyborg 主题 #1098 等）。
- **归档后状态**：仓库仍可 star/fork（gh api 显示 45047 星、3227 fork，快照后仍自然增长），开放 issue 仅 6 个（已冻结不接受新反馈的典型状态）；无官方接续项目（未确认任何 fork 成为事实继任者）；README 引用的 Homebrew/AUR 等第三方发行渠道在归档后是否仍可用未确认。
- 传播度证据（README "Featured in"）：Hacker News、BoingBoing、O'Reilly、LinuxLinks、GitHub Blog Release Radar 2018-12、BestOfJS Rising Stars 2020、JSNation OS Awards 2021 提名（Fun Side Project of the Year）。

## 9. 亮点与风险

**亮点**
1. 教科书级的 Electron 终端架构：PTY（主进程）→ 本地 WebSocket → xterm.js（渲染进程）的控制面/数据面分离设计清晰且完整可抄（src/src/classes/terminal.class.js）。
2. 用 Node cluster + IPC Proxy 把阻塞型系统信息采集多线程化，且对上层完全透明（src/src/_multithread.js、_renderer.js 的 window.si Proxy）。
3. 低依赖高表现力的可视化：440 点阵内存图、纯 DOM 模块、每模块独立 CSS，加 vendored encom-globe 地球仪，视觉冲击力与代码量成反比。
4. 完整的触屏方案：19 种布局 JSON + 状态机键盘 + 触摸滚动 + 软键盘抑制，是 Electron 触屏应用少有的完整参考。
5. 主题/键盘布局全部数据化（JSON），支持用户目录热替换与运行时热切换，扩展成本极低。

**风险/局限**
1. **项目已死**：Electron 12（2021）、xterm 4.x、systeminformation 5.9.7 均已严重过期，存在已知 CVE 的依赖不会再更新；不应再用于生产环境。
2. nodeIntegration:true + contextIsolation:false 的旧式安全模型与现代 Electron 最佳实践相悖，二进制也未签名（README 说明）。
3. 无任何自动化测试，原生模块（node-pty）在新平台/新 Node 上的兼容性无保障，fork 者接手成本高。
4. Windows 功能残缺（CWD 追踪不可用、无温度显示），部分体验依赖 Linux/macOS。

## 10. 总结评价

eDEX-UI 是"把玩笑开成正经软件"的典范：它以 41.5k 星证明了极致视觉完成度本身就是一种生产力，又用归档时的坦诚（作者公开承认维护不下去、定位是艺术项目）示范了开源项目如何体面退场。代码本身虽已停止演进，但其架构选型——双 package.json、PTY/WebSocket/xterm 三层解耦、cluster 多线程化系统采集、JSON 数据化主题——至今仍是 Electron 终端类应用的可抄模板。它适合三类人：想学 Electron 终端模拟器实现的开发者（读 terminal.class.js 与 _multithread.js 即可值回票价）、想给触屏设备做酷炫 dashboard 的极客（直接用最后的 v2.2.8，注意安全风险）、以及研究"如何运营一个视觉驱动型开源项目"的社区观察者。不适合把它的二进制当作日常安全终端使用。
