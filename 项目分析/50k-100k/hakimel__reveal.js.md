# hakimel/reveal.js 项目分析

> 星数 72,497（gh api 实测快照 2026-10-31）｜主语言 JavaScript｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：最老牌、使用最广泛的 HTML 演示框架——用浏览器和 `<section>` 标签写幻灯片，替代 PowerPoint 的开源方案。

| 项目 | 值 |
|---|---|
| 仓库 | hakimel/reveal.js |
| 星数 | 72,497（fork 17,034，open issues 931） |
| 主语言 | JavaScript（核心为带 JSDoc 的 ES 模块 + TypeScript 类型） |
| 版本 | 6.0.2（package.json） |
| 许可证 | MIT（LICENSE，版权 2011-2026 Hakim El Hattab） |
| 默认分支 | master |
| 最近活跃 | 最近 push 2026-09-30，最近 release 6.0.2（2026-09-10） |

## 2. 项目是什么，解决什么问题

reveal.js 是一个纯浏览器端的演示文稿框架：作者在 HTML 里写 `<section>` 作为幻灯片，框架负责水平/垂直嵌套导航、键盘/触摸/URL 定位、缩放适配、过渡动画、分片逐步展示、演讲者备注、Markdown 渲染、PDF 导出等（README.md 功能列表，js/controllers/ 下 20 余个控制器逐一对应这些能力）。它从 2011 年发展至今（js/reveal.js 版权头），是"HTML 做 slides"这一品类的开创者和事实标准。

解决的痛点很明确：让任何人"打开浏览器就能做和放演示"，无需安装桌面软件、无需构建工具（引入 dist 后一个 script 标签即可），同时保留 Web 的全部表达力——视频、iframe、交互代码、CSS 主题。核心特色功能包括 Auto-Animate（相邻两页自动补间动画，js/controllers/autoanimate.js）、滚动阅读视图（scroll view，js/controllers/scrollview.js）、多实例嵌入（同页面跑多套演示）、以及 React 组件封装（react/ 目录）。

生态位上，它与作者的商业产品 Slides.com 同源共生（README.md 明确推荐），revealjs.com 官网 + 付费视频课程构成其商业模式；5.x 起单页可多实例、6.x 引入滚动视图与官方 React 包，仍在活跃演进。

## 3. 技术栈与依赖

- **运行时零依赖**：核心 js/ 目录不依赖任何第三方运行时库，纯 DOM API 实现（package.json 只有 devDependencies），这是它能以单文件分发的前提。
- **构建链**：Vite 8 库模式（vite.config.ts，ESM + UMD 双格式，target es2015）+ `vite-plugin-dts` 生成类型；TypeScript 6 负责类型检查与 .d.ts（package.json build 脚本先 `tsc` 再 vite build）。
- **样式**：SCSS（sass），css/reveal.scss 为主样式，css/theme/ 下 14 个主题 SCSS 经独立配置 vite.config.styles.ts 逐个编译。
- **内置插件依赖（构建时打进插件产物）**：highlight.js 11（语法高亮，plugin/highlight/）、marked 17（Markdown，plugin/markdown/）、KaTeX/MathJax 按需 CDN 加载（plugin/math/）、fitty 2.4（按字号自适应，js/controllers/slidecontent.js 第 244 行）。
- **测试链**：QUnit 2 + Puppeteer 25（node-qunit-puppeteer），React 子包用 Vitest 4 + @testing-library/react。
- **React 子包**：`@revealjs/react` 0.2.2（react/package.json），peer 依赖 react >=18、reveal.js >=5。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `js/reveal.js` | 核心入口，单实例工厂函数（2,988 行），组装全部控制器与公开 API |
| `js/controllers/` | 23 个职责单一的控制器：导航、fragment、动画、背景、键盘、触摸、滚动视图等 |
| `js/config.ts` | 全部配置项的 TS 类型定义与 defaultConfig（973 行） |
| `js/utils/` | util/dom 工具、设备探测、脚本加载器 |
| `plugin/` | 6 个官方插件：highlight、markdown、math（4 种引擎）、notes、search、zoom，各带独立 vite.config |
| `react/` | 独立 npm 包 @revealjs/react：Deck/Slide/Stack/Fragment/Markdown/Code 组件 + Vitest 测试 |
| `css/` | reveal.scss 主样式 + theme/ 主题体系（模板 mixins/settings + 14 主题） |
| `dist/` | 预构建产物（提交入库，main 指向 dist/reveal.js） |
| `test/` | 21 个 QUnit 测试 HTML 夹具 + 资源 |
| `examples/` | 13 个功能示例页（auto-animate、scroll、multiple-presentations 等） |
| `scripts/` | 构建/打包/测试辅助：test.js、build-es5.js、add-banner.js、zip.js |
| `build/` | dts-paths.ts（.d.ts 产物路径兼容处理） |

组织方式评述：单包为主、内嵌一个 React 子包的"包中包"结构。核心采用控制器模式拆分（每个控制器类接收 Reveal 实例引用），源码是 ES 模块但对外同时提供旧式全局 API；dist 提交入库使不装构建链也能直接用，这是面向普通用户框架的务实选择。

## 5. 核心架构与关键实现

### 5.1 双入口 API 与控制器组装（js/reveal.js、js/index.ts）

`js/reveal.js` 是一个工厂函数，内部维护 config、indexh/indexv、scale 等闭包状态，并在构造时实例化 20 余个控制器（js/reveal.js 约 105-135 行）：

```js
slideContent = new SlideContent( Reveal ),
fragments = new Fragments( Reveal ),
autoAnimate = new AutoAnimate( Reveal ),
scrollView = new ScrollView( Reveal ),
keyboard = new Keyboard( Reveal ),
...
```

每个控制器只拿一个 `Reveal` 引用（可回调主实例状态），实现多实例并行。`js/index.ts` 则维持 pre-4.0 的 `Reveal.initialize()` 单例 API 向后兼容：先收集过早调用的 `on/configure` 等方法入队（enqueuedAPICalls），initialize 时一次性注入新建实例（js/index.ts 中间段）。API 兼容层覆盖了 2011 年以来的老用法，是该库长生命周期的关键设计。

### 5.2 缩放布局引擎（js/reveal.js layout()，约 806 行起）

幻灯片按固定设计尺寸（默认 width:960/height:700，js/config.ts）排版，再整体缩放适配视口：

```js
scale = Math.min( size.presentationWidth / size.width,
                  size.presentationHeight / size.height );
scale = Math.max( scale, config.minScale );
scale = Math.min( scale, config.maxScale );
```

配合 `layoutSlideContents()` 用 fitty 对 `[data-fit]` 元素做字号自适应（js/controllers/slidecontent.js:244），并对移动端 100vh 问题自建 `--vh` 自定义属性（js/reveal.js:815 附近）。滚动视图激活时跳过整体缩放（`scrollView.isActive()` 分支），让页面回到自然文档流。

### 5.3 Fragment 分片机制（js/controllers/fragments.js，374 行）

Fragment 是页内逐步显示的元素，按 `data-fragment-index` 显式排序、未标注者按 DOM 序（fragments.js:109-136 的 sort 逻辑）。导航时 Fragments 向滑动方向逐一加减 `visible`/`current-fragment` 类，全部展示/隐藏完后才把导航权交还给整页切换——这解释了"一次按键先走 fragment、走完翻页"的行为。`groups()` 方法将同 index 的 fragment 归组实现"一次全出"。

### 5.4 Auto-Animate 自动补间（js/controllers/autoanimate.js，626 行）

相邻两页只要都带 `data-auto-animate`（且 id 匹配、未遇 restart 属性）即触发：对两页中匹配的同元素（同 tag+文本或显式 data-id）测量前后位置/尺寸，生成 CSS transition 注入动态样式表，逐元素补间（autoanimate.js `run()`/`autoAnimateElements()`）；不匹配的新元素按 unmatched 延迟入场。对隐藏的 from-slide 先临时显示以便测量（autoanimate.js:55 附近），是典型的"测量-生成样式-播放"三段式。

### 5.5 滚动阅读视图（js/controllers/scrollview.js，951 行）

`view: 'scroll'` 配置（js/config.ts:945）切换为线性滚动阅读：activate() 先保存整个 slides 容器 innerHTML 以便还原（scrollview.js:40 附近），再把每页包进 page 元素、去掉缩放、按视口分页，并渲染页间进度条（progress bar/playhead）。与打印视图（js/controllers/printview.js）共享"DOM 重排-还原"模式，两种视图的激活/停用都保证演示状态可无损恢复。

### 5.6 插件体系（js/controllers/plugins.js + plugin/）

插件即普通对象，经 `Reveal.registerPlugin()` 或 `dependencies` 配置注册进 id:instance 映射（plugins.js `load()`），支持同步/异步脚本与 condition 条件加载；初始化串行执行各插件 `init()` 并等待其 Promise（plugins.js `initPlugins()`），确保插件就绪后框架才派发 ready。官方 6 插件各自带 vite.config.ts 独立打包出 UMD/ESM/类型（package.json build 脚本串联 7 次 vite build），npm exports 按 `./plugin/*` 子路径分发；math 插件内置 KaTeX/MathJax2/3/4 四引擎适配（plugin/math/katex.js、mathjax2.js、mathjax3.js、mathjax4.js）。

## 6. 构建与运行

来自 package.json scripts（均未实际执行，静态确认）：

```bash
npm install            # node >= 22.12（devEngines 强制）
npm start              # vite 开发服务器，端口 8000，演示 index.html/demo.html
npm run build          # tsc 类型检查 + vite 核心构建 + 主题样式构建
                       # + 6 个插件各自构建 + add-banner.js 加版权头
npm run build:es5      # 额外产出 es5 兼容包（scripts/build-es5.js）
npm test               # scripts/test.js：起 vite 服务(8009) + Puppeteer 逐个跑 test/*.html
npm run react:build    # react/ 子包构建；react:test 为 vitest
npm run package        # scripts/zip.js 打 zip 发布包
```

使用方（非开发者）只需引入 `dist/reveal.js` + `dist/reveal.css`，或 `import Reveal from 'reveal.js'`（package.json exports 提供双格式与类型入口）。

## 7. 测试与工程质量

- **浏览器级端到端测试为主**：test/ 下 21 个 HTML 夹具共 146 处 `QUnit.test`（grep 统计），覆盖导航、fragment、auto-animate、滚动视图、多实例、iframe、PDF、插件等场景；scripts/test.js 用 Vite 起本地服务，Puppeteer 逐页执行 QUnit 并汇总结果，超时 30 秒/页。
- **CI**：.github/workflows/test.yml 对每次 push 在 Node 22/24 矩阵上跑 `npm run build` + `npm test`；另有 spellcheck.yml 拼写检查。React 子包另有 5 个 Vitest 单测文件（react/src/components/*.test.tsx，合计约 1,500 行，含 deck.test.tsx 496 行）。
- **质量观察**：核心代码带详细注释和版权头；config.ts 用 973 行 TS 类型完整描述全部配置项并发布 .d.ts（vite.config.ts dts 插件）；测试夹具即"最小复现页"的风格使回归定位直观。不足：无覆盖率报告，主库 QUnit 测试偏集成而非单元粒度。

## 8. 社区与维护状态

- **维护活跃**：最近 commit 2026-09-30（修复 syncSlide 背景与 iframe 问题，gh api commits），2026 年内发版 6.0.0（3 月）→6.0.1（4 月）→6.0.2（9 月），5.x→6.x 保持约每年一个大版本的节奏（gh api releases）。
- **贡献结构**：榜首贡献者 2,593 次提交（gh api contributors，即作者 Hakim El Hattab），呈"单人核心 + 社区 PR"结构；README/CONTRIBUTING.md 引导 issue 与 PR，.github/FUNDING.yml 存在，商业转化靠 Slides.com 与付费课程。
- **规模**：17,034 fork、931 open issues（gh api 实测），社区生态成熟；6.x 新增官方 React 包（react/README.md、react/package.json @revealjs/react 0.2.2）显示仍在开拓新集成方向。
- 6.0 的滚动视图、React 包均为 2026 年新增，未确认具体贡献者分工细节。

## 9. 亮点与风险

**亮点**
1. 运行时零依赖 + dist 入库：一个 script 标签即可使用，十多年 API 向后兼容（js/index.ts 兼容层），老演示文稿至今可跑。
2. 控制器化架构：23 个控制器各管一域、只共享 Reveal 引用，单文件可多实例，代码边界清晰（js/controllers/）。
3. Auto-Animate 与滚动视图等产品级差异化功能，实现完整且可配置（autoanimate.js/scrollview.js）。
4. 测试贴近真实浏览器（Puppeteer+QUnit），配 21 个功能夹具页，回归防护扎实。
5. 文档与示例完备：13 个 examples/ 页 + revealjs.com 官网 + TS 全量类型定义。

**风险/局限**
1. 核心仍是闭包+类的命令式 DOM 操作（js/reveal.js 近 3,000 行单文件），无虚拟 DOM/响应式，深度定制成本随状态增多而上升。
2. 单人主导维护，bus factor 偏低；931 个 open issues 表明响应压力不小（未确认处理时长）。
3. 主题/插件产物依赖 Vite 8 与 Node >=22.12 的前沿工具链，贡献者本地环境门槛较高（package.json devEngines）。
4. dist 预构建产物提交入库，仓库体积与 PR diff 噪音增大。

## 10. 总结评价

reveal.js 是"HTML 演示"品类从零到标准的完整样本：运行时零依赖、双 API 兼容层、控制器化架构和浏览器级测试共同支撑了 15 年的持续演进，2026 年的 6.x 仍在滚动视图和官方 React 包上做增量创新。它适合作为直接工具给需要自制演示的技术讲者与文档作者，也适合作为源码范本给研究无框架前端架构（缩放布局、键盘/触摸导航、视图模式切换、插件生命周期）的工程师——后者尤其值得精读 js/reveal.js 与 js/controllers/ 下的分治设计。
