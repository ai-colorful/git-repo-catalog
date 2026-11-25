# zenorocha/clipboard.js 项目分析

> 星数 33500（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：零依赖、gzip 后约 3KB 的浏览器端"复制/剪切到剪贴板"库，用声明式 HTML data 属性 + 极简 JS API 取代了 Flash 时代的 ZeroClipboard。

| 项目 | 内容 |
| --- | --- |
| 名称 | zenorocha/clipboard.js（npm 包名 `clipboard`） |
| 星数 | 34077（gh api 实测 2026-11-21，快照 33500） |
| 语言 | JavaScript（ESM 源码 + 手写 TypeScript 声明） |
| 许可证 | MIT（LICENSE，© Zeno Rocha） |
| 最近活跃 | master 最后人工提交 2022-05-04（v2.0.11 发版）；仓库最近 push 2026-02-12 实为 dependabot PR 分支（详见第 8 章） |
| 官网 | https://clipboardjs.com |
| 版本 | v2.0.11（2022-05-04，npm/CHANGELOG 均以 release 为准） |

## 2. 项目是什么，解决什么问题

在 2015 年之前，网页端"一键复制"几乎只有两条路：依赖 Flash 的 ZeroClipboard，或者手写一段涉及 `document.execCommand('copy')`、离屏 textarea、Selection API 的繁琐样板代码。clipboard.js 的出发点（readme.md "Why" 一节）是：复制文本不该需要几十步配置、几百 KB 体积，更不该依赖 Flash 或任何大框架。它把这些浏览器怪癖封装成约 3KB（gzip 后）的零依赖库，作者 Zeno Rocha 是巴西籍知名开源开发者。

API 设计走"声明式优先"路线（readme.md "Usage"）：按钮上放 `data-clipboard-target="#foo"`（复制某元素内容）、`data-clipboard-action="cut"`（剪切，仅限 input/textarea）、或 `data-clipboard-text="..."`（复制字面量），JS 侧只需 `new ClipboardJS('.btn')` 一行。结果通过自定义 `success`/`error` 事件回调（readme.md "Events"），`ClipboardJS.isSupported()` 用于能力检测降级（readme.md "Browser Support"）。

技术选型上它刻意保守：整个库建立在 Selection API 与 `document.execCommand('copy'/'cut')` 之上（readme.md "Browser Support" 明确列出两个依赖的 API），经实测源码全库（src/ 与 dist/）均未使用 `navigator.clipboard` 异步 API。这让 IE 9+ / 老浏览器也能工作（同一节兼容表：Chrome 42+、Edge 12+、Firefox 41+、IE 9+、Opera 29+、Safari 10+），代价是沿用了一条 MDN 已标记废弃的同步 API 路径。

流行度方面：34k+ 星、3892 forks（gh api 2026-11-21），README 末尾还衍生了一个给 GitHub/MDN/StackOverflow 等代码块加复制按钮的浏览器扩展 codecopy（readme.md "Bonus"）。它是"小而美"单功能库的代表——解决的问题极窄，但足够普遍，以至于长期被大量站点与文档站内嵌使用。

## 3. 技术栈与依赖

- 语言：ES2020 风格 ESM JavaScript（`import/export`，src/clipboard.js），Babel 转译为 ES5 产物（webpack.config.js `target: ['web','es5']` + .babelrc.json `@babel/preset-env`）
- 运行时依赖仅 3 个，且都是作者自己的微库（package.json dependencies）：
  - `tiny-emitter` ^2.0.0 —— 为 Clipboard 类提供 `on/emit` 事件能力（src/clipboard.js 第 1 行 import）
  - `good-listener` ^1.2.2 —— 事件绑定/委托封装（同文件第 2 行，`listen(trigger,'click',cb)`，委托结果经 `e.delegateTarget` 取回，src/clipboard.js onClick）
  - `select` ^1.1.2 —— 跨元素类型（input/textarea/contentEditable）选中文本（src/actions/copy.js、src/actions/cut.js）
- 构建链：webpack 5 + babel-loader + UglifyJSPlugin（package.json devDependencies、webpack.config.js）
- 测试链：karma 6 + mocha + chai + sinon + ChromeHeadless（karma.conf.js）；类型测试用 tsd（package.json `test:types`）
- 类型：手写声明文件 src/clipboard.d.ts（package.json `"types": "src/clipboard.d.ts"`）
- 代码质量工具：eslint（airbnb-base 配置）+ prettier + husky + lint-staged（package.json、.eslintrc.json、.husky/pre-commit）
- Node 版本锁定 14（.nvmrc）

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| src/clipboard.js | 主类 Clipboard（对外名 ClipboardJS），事件委托 + 动态回调 + 静态 API |
| src/actions/ | 三个动作工厂函数：default.js（校验与分派）、copy.js、cut.js |
| src/common/ | command.js（execCommand 封装）、create-fake-element.js（离屏 textarea 构造） |
| src/clipboard.d.ts | 手写 TypeScript 类型声明（ClipboardJS 类、Options、Event） |
| src/clipboard.test-d.ts | tsd 类型测试 |
| demo/ | 13 个自包含 HTML 示例（constructor-selector/node/nodelist、function-target/text、target-div/input/input-number/textarea、programmatic-copy/cut、text-programmatic-copy） |
| test/ | karma+mocha 真浏览器测试，目录结构镜像 src/（test/clipboard.js、test/actions/、test/common/） |
| dist/ | 构建产物 clipboard.js（26.8KB）与 clipboard.min.js（9.2KB），提交进仓库，mtime 2022-12 |
| .github/ | CI（workflows/test.js.yml、publish.yml）、issue/PR 模板、stale bot 配置（stale.yml） |
| bower.json / composer.json / package.js | 老渠道分发：Bower、PHP Composer（zenorocha/clipboardjs）、Meteor |

组织方式评述：单体单包仓库，无 monorepo/子包。源码本身极小（src/ 下 6 个 JS 文件，总计不到 300 行），靠 webpack 打成 UMD 单文件。分发面铺得很广是历史产物——npm 之外连 Bower/Composer/Meteor 清单都保留在仓库根部，体现了 2015 年代前端生态的分发习惯。官网 clipboardjs.com（gh-pages 分支维护，最后提交 2021-01-19，gh api 分支记录）与仓库内 demo/ 是两套演示。

## 5. 核心架构与关键实现

### 5.1 Clipboard 类：委托监听 + 动态回调（src/clipboard.js）

Clipboard 继承 tiny-emitter 的 Emitter（第 26 行 `class Clipboard extends Emitter`），构造函数只做两件事：解析 options、挂载点击监听（第 31-36 行）。监听通过 good-listener 完成，README 明确说明采用事件委托以避免对几百个匹配元素逐个 addEventListener 造成的内存开销（readme.md "Setup" 末段）；点击回调里用 `e.delegateTarget || e.currentTarget` 兼容委托与直绑两种事件（onClick，第 70-72 行）。

action/target/text 三个配置项遵循同一模式：传函数则用自定义回调，否则回退到读 `data-clipboard-action/target/text` 属性的 defaultAction/defaultTarget/defaultText（resolveOptions 第 43-56 行，getAttributeValue 第 12-20 行）。这就是 README "Advanced Options" 中"不写 HTML 也能动态决定复制内容"的机制，例如 `text: (trigger) => trigger.getAttribute('aria-label')`。

```js
// src/clipboard.js onClick（第 70-92 行，节选）
onClick(e) {
  const trigger = e.delegateTarget || e.currentTarget;
  const action = this.action(trigger) || 'copy';
  const text = ClipboardActionDefault({
    action,
    container: this.container,
    target: this.target(trigger),
    text: this.text(trigger),
  });
  // Fires an event based on the copy operation result.
  this.emit(text ? 'success' : 'error', {
    action, text, trigger,
    clearSelection() { /* focus + removeAllRanges */ },
  });
}
```

值得注意：动作执行被设计成同步函数调用——`ClipboardActionDefault` 直接返回复制到的文本（成功）或 undefined/抛错（失败），事件由 Clipboard 实例自身 emit，事件对象携带 `action/text/trigger/clearSelection()`（src/clipboard.d.ts Event 接口与之一致）。另有三个静态方法：`Clipboard.copy(target, options)` 与 `Clipboard.cut(target)` 提供无按钮的编程式复制（第 120-131 行），`Clipboard.isSupported()` 基于 `document.queryCommandSupported`（第 138-147 行）。`destroy()` 仅销毁监听器（第 160-162 行）。

### 5.2 动作层：校验与分派（src/actions/default.js）

ClipboardActionDefault 是纯函数入口，负责三件事：把 action 限定为 copy/cut（否则抛错，第 14-16 行）；对 target 元素做合法性与业务规则校验——copy 不允许 `disabled` 元素（提示改用 readonly）、cut 不允许 `readonly`/`disabled` 元素（第 20-38 行）；然后按"text 优先于 target"分派（第 41-50 行）：给了 `text` 就直接复制字符串，否则按 action 调 ClipboardActionCopy 或 ClipboardActionCut。

### 5.3 复制路径：真元素选中 vs 离屏假元素（src/actions/copy.js、src/common/）

复制分两条路径（src/actions/copy.js ClipboardActionCopy，第 27-45 行）：

1. 目标是 input/textarea 等可选元素（input.type 属于 text/search/url/tel/password，支持 setSelectionRange）→ `select(target)` 后直接 `command('copy')`；
2. 目标是纯字符串，或是不支持 setSelectionRange 的 input（如 type=number，第 34-39 行，对应 2.0.11 修的 issue #800）→ 构造离屏假 textarea：`createFakeElement`（src/common/create-fake-element.js）做了大量浏览器怪癖处理——fontSize 12pt 防iOS 聚焦缩放、`-9999px` 水平移出屏幕（RTL 时反向）、top 跟随页面滚动位置防跳动、设 readonly 防软键盘弹出（第 6-26 行）。

`command('copy')` 是对 `document.execCommand` 的 try/catch 封装（src/common/command.js 第 6-12 行），失败返回 false 而非抛异常；剪切路径同样只有三行——select 后 command('cut')（src/actions/cut.js 第 9-13 行）。整条链没有 `navigator.clipboard.writeText`，即官方 README 兼容策略：依赖 execCommand + Selection，靠 success/error 事件让调用方自行降级提示（readme.md "Browser Support"）。

### 5.4 类型与对外契约（src/clipboard.d.ts）

手写声明文件定义 `ClipboardJS` 类、Options（action/target/text 三个 `(elem)=>T` 回调签名、container）、Event（action/text/trigger/clearSelection），并用 `export = ClipboardJS` + `export as namespace ClipboardJS` 同时兼容模块导入与全局 script 标签用法（src/clipboard.d.ts 第 13-91 行）。tsd 类型测试仅一行构造校验（src/clipboard.test-d.ts）。

## 6. 构建与运行

来自 package.json scripts 与 CI 实际命令：

```bash
npm install          # 安装依赖
npm run build        # 产物：webpack 先后产出 dist/clipboard.js 与 dist/clipboard.min.js
npm run lint         # eslint --ext .js src/
npm test             # karma start --single-run（ChromeHeadless 真浏览器跑 mocha）
npm run test:types   # tsd 类型测试
npm run build-watch  # webpack --watch 开发模式
```

发布流程：release 创建时 GitHub Actions 自动跑测试后向 npm 与 GitHub Packages 双发（.github/workflows/publish.yml）。仓库内 dist/ 为提交产物（webpack.config.js 生成 UMD：`library: 'ClipboardJS'`、`libraryTarget: 'umd'`、ES5 target，文件头 BannerPlugin 注入版本与版权注释）。

## 7. 测试与工程质量

- 测试形态：真浏览器集成测试而非纯单元测试——karma 起 ChromeHeadless，测试用例真实创建 DOM（button/input/textarea）并真实执行 execCommand（karma.conf.js、test/clipboard.js before 钩子）。例如 test/clipboard.js 的 `#onClick` 用例会实际触发复制并断言 success 事件对象的属性（第 76-115 行），test/actions/copy.js 覆盖 input/元素/字符串/input number 四种复制路径（第 10-67 行）。
- 规模：测试文件 6 个，与 src/ 一一镜像（test/clipboard.js 192 行、test/actions/copy.js 69 行、test/common/command.js 49 行等），类型测试 1 个。
- CI：push/PR 到 master 触发 build+lint+test，Node 10/12/14/15 矩阵（.github/workflows/test.js.yml，矩阵版本今天看已相当陈旧）。
- 质量观察：eslint airbnb-base + prettier 强制风格（.eslintrc.json、husky pre-commit + lint-staged）；注释以 JSDoc 为主，src/ 六个文件结构清晰、函数式动作层无状态，可读性高。一处与实现脱节的残留：test/clipboard.js `#destroy` 用例仍断言 `clipboard.clipboardAction`（第 144-153 行），该属性在当前 src/clipboard.js 中已不存在（现版动作不再实例化到 this 上），断言依赖 chai 宽松相等（null == undefined）通过，属旧实现时代的测试残留。

## 8. 社区与维护状态

维护节奏的核实结论（gh api 2026-11-21）：

- 分支层面：master 最后提交 2022-12-10（为 dependabot 锁文件升级），最后一次人工功能提交是 2022-05-04 v2.0.11 发版（"Support more HTML input types" #808，作者 Beto Muniz/obetomuniz）。此后无版本发布。
- 仓库 pushed_at 显示 2026-02-12，但追踪发现该推送来自 dependabot 自动维护的依赖升级分支（dependabot/npm_and_yarn/multi-d31b51a7f2，bump qs/body-parser；此前 2025-08、2025-11 也各有一次 dependabot 分支推送）。即：快照所言"最近推送 2026-02"是机器人在挂起 PR 分支上的活动，主干实质已四年无版本演化。
- 状态：仓库未归档（`archived: false`），开放 issue 仅 16 个——.github/stale.yml 配置 60 天无活动自动标 stale、7 天后关闭，解释了 issue 积压被压平的现象。
- release 历史：v2.0.6（2020-03）→ v2.0.7/v2.0.8（2021-03）→ v2.0.9（2022-01）→ v2.0.10（2022-02）→ v2.0.11（2022-05），节奏从年均数版降到停滞。
- 贡献者结构：共 30 人，zenorocha 本人 205 次提交居首，vitormalencar 45 次、dependabot 24 次、obetomuniz（Beto Muniz，2.0 系列实际发版人）13 次、r3nanp 12 次、helderberto 9 次（gh api contributors）。核心维护圈小而稳定，社区长期以提 issue/PR 为主。
- 生态位：npm 包 `clipboard` 长期位居最常下载的前端工具库之列，README 指引 CDN 用户见 wiki 的 CDN-Providers 页（具体 CDN 引用量未确认，规范内不发起该类外部请求）。

总体定性：项目处于"功能完备后的冻结态"——未宣布放弃、issue 由机器人维持秩序，但作者不再主动演化；这与任务快照"低频但持续"的预期相比更偏保守：持续的是 dependabot 噪音，人工维护在 2022 年中后基本停止。

## 9. 亮点与风险

亮点：

1. API 设计极简且正交：声明式 data 属性 + 动态函数回调 + 静态编程式 copy/cut 三种入口覆盖了绝大多数场景（src/clipboard.js resolveOptions、静态方法）。
2. 体积与依赖控制教科书级：运行时仅 3 个作者自维护微库，min 后 9.2KB、gzip 约 3KB（package.json description、dist/clipboard.min.js 实测字节）。
3. 浏览器怪癖处理集中且文档化：离屏 textarea 的 RTL/滚动位置/iOS 缩放处理全部集中在 create-fake-element.js 一个文件内（第 6-26 行）。
4. 事件委托防内存膨胀，destroy 提供SPA 生命周期清理（readme.md "Setup"、src/clipboard.js destroy）。
5. 测试真实（ChromeHeadless 里真复制），类型声明手写完备并随包发布（src/clipboard.d.ts）。

风险/局限：

1. 技术路线已被平台取代：整库依赖 `document.execCommand`（MDN 已标废弃），未接入 `navigator.clipboard` 异步 API（全库 grep 确认，src/actions/copy.js、src/common/command.js）；新项目在严格安全策略（如 iframe 权限）下可能遇到 execCommand 失效。
2. 维护冻结：四年无 release（最新 v2.0.11，2022-05-04），devDependencies 里 karma/eslint/Node 版本全面陈旧（CI 矩阵仍为 Node 10-15，.github/workflows/test.js.yml），dependabot PR 长期无人合并。
3. 已知遗留问题：test/clipboard.js 残留旧实现断言（见第 7 章）；README 自述该库在 error 场景依赖调用方自行做"请按 Ctrl+C"降级提示（readme.md "Browser Support"）。
4. 无 CHANGELOG 文件，版本历史需看 GitHub releases（未确认仓库内存在独立 CHANGELOG，readme.md 亦未链接）。

## 10. 总结评价

clipboard.js 是前端历史上"用 3KB 取代一个 Flash 插件"的标志性项目：以极小的代码面积（src/ 六文件、动作层纯函数化）、三依赖、声明式 API 和真浏览器测试，把一件人人要做、人人做不好的小事做成了十年代际默认选项。它的源码至今仍是学习"如何优雅封装浏览器怪癖"（离屏元素、事件委托、能力检测、同步 execCommand 路径）的优秀教材。客观地看，它如今处于维护冻结状态：execCommand 路线被 Async Clipboard API 时代超越，四年未发版，dependabot 的活跃只是噪音。适合谁：维护存量站点、需要 IE 级兼容或零构建场景的开发者仍可直接使用（功能稳定，bug 已收敛）；想读懂小而美库的工程设计、或研究"废弃 API 包装层如何设计降级"的人值得一读源码；而新项目若只需现代浏览器，直接用原生 `navigator.clipboard.writeText` 或其轻封装更合适。
