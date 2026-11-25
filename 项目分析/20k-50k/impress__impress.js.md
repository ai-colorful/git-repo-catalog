# impress/impress.js 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

**一句话定位**：基于 CSS3 3D transforms/transitions 的浏览器演示框架——把每张"幻灯片"当作三维空间中自由摆放的物体，通过移动"摄像机"完成翻页，是最早的 Prezi 开源替代品。

| 事实 | 值 | 出处 |
|---|---|---|
| 名称 | impress/impress.js（master 分支） | gh api |
| 星数 | 39500（快照）；gh api 2026-11-14 实测 38153 | 任务参数 / gh api |
| 主语言 | JavaScript（纯原生 JS，零运行时依赖） | package.json |
| 许可证 | MIT（Copyright (c) 2011-2016 Bartek Szopka） | LICENSE |
| 创建时间 | 2011-12-28 | gh api created_at |
| 最近 push | 2026-07-23（docs(toolbar) 提交） | gh api / tarball HEAD mtime |
| 最新 release | v2.0.0（2022-07-22） | gh api releases |
| 贡献者 | 93 人（contributors API 分页统计） | gh api |
| 主页 | http://impress.js.org（demo：impress.github.io/impress.js） | gh api homepage |

## 2. 项目是什么，解决什么问题

impress.js 由 Bartek Szopka（@bartaz）于 2011 年底创作，README 开篇即说明"based on the power of CSS3 transforms and transitions in modern browsers and inspired by the idea behind prezi.com"（README.md）。它解决的问题是：在浏览器里做出 Prezi 那种"整体缩放、旋转、飞入飞出"的空间感演示，而不依赖 Flash 或任何桌面软件。

与 PowerPoint/Reveal.js 的"第 1 页、第 2 页"线性隐喻不同，impress.js 的核心抽象是**三维画布**：作者用 HTML `data-*` 属性给每个 `.step` 元素声明它在 3D 空间中的坐标（`data-x/y/z`）、旋转（`data-rotate-x/y/z`）、缩放（`data-scale`），核心引擎再把整个"摄像机"反向变换过去（src/impress.js goto() 中的 target 计算）。翻页就是一次 CSS transition 驱动的空间飞行，全程 GPU 合成，无需逐帧 JS 动画。

项目的另一条生命线是其作者哲学：README 的 BROWSER SUPPORT 一节明确"specifically do not use jQuery or any other JavaScript libraries"——不兼容旧浏览器、不引入任何运行时依赖，用它来展示现代浏览器 CSS3 能力的上限。这一点使它成为 2012 年前后 CSS 3D 变换的"教科书级示范项目"，源码自带大量教学注释（结尾的"THAT'S ALL FOLKS!"即出自 src/impress.js:920-926）。

仓库同时是完整的作品生态：index.html 是被官方当作教程使用的、带逐行注释的 demo；examples/ 下有 6 个示例演示（2D-navigation、3D-positions、3D-rotations、classic-slides、cube、markdown，见 examples/ 目录）；配套 DOCUMENTATION.md（Reference API）与 GettingStarted.md。姊妹项目 Impressionist（henrikingo/impressionist）提供 3D 可视化编辑器（README.md 提及）。

## 3. 技术栈与依赖

- **运行时**：纯原生 JavaScript（ES5 风格 IIFE + "use strict"），仅依赖浏览器 DOM API：dataset、classList、CSS3 3D transforms/transitions、CustomEvent（README.md BROWSER SUPPORT 一节；src/impress.js:143-157 的支持性检测）。
- **运行时依赖：0 个**。package.json 的 dependencies 字段不存在，全部条目在 devDependencies。
- **构建**：`build.js`（Node）把 src/impress.js + 3 个 lib + 22 个插件按固定顺序**字符串拼接**为 js/impress.js，再用 terser 压缩出 js/impress.min.js + sourcemap，并自动生成 examples/index.html 目录页（build.js）。
- **测试**：karma + QUnit + syn（合成键盘/鼠标事件）+ puppeteer 驱动的 Chrome/Firefox（package.json、karma.conf.js）。
- **Lint**：jshint + jscs（`npm run lint`），另提供 `npm run new-lint`（eslint，.eslintrc.js 9KB 规则）（package.json）。
- **CI**：CircleCI（.circleci/config.yml，node current-browsers：build/lint/test）+ GitHub Actions（.github/workflows/pipeline.yaml，node 22，verification→testing→assembly 三段流水线，PR/push 到 master 触发）。
- **要求**：node >= 7.6（README.md Requirements）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| index.html | 官方 demo，注释极密，兼作官方教程（README.md 明言） |
| src/impress.js | 核心（约 918 行）：支持检测、步骤模型、goto 状态机、插件注册口 |
| src/lib/ | 3 个按 addLibraryFactory 注册的内建库：util.js（DOM/数值工具）、gc.js（可逆生命周期/垃圾回收）、rotation.js（四元数旋转数学，Tommy Tam 2021） |
| src/plugins/ | 22 个默认插件（autoplay/blackout/bookmark/extras/form/fullscreen/goto/help/impressConsole/media/mobile/mouse-timeout/navigation/navigation-ui/progress/rel/resize/skip/stop/substep/toolbar/touch），每插件一目录、一 JS + README + 可选测试 |
| extras/ | git submodule → impress/impress-extras：第三方附加插件集合（highlight.js/markdown/MathJax/Mermaid，gh api 显示仅 19 星、最后 push 2017-11） |
| js/impress.js | build.js 拼接产物（186KB），用户实际 `<script>` 引用的文件 |
| test/ | QUnit 测试（core_tests.js 266 行、non_default.js 144 行、helpers.js、test/plugins/rel/ 下 4 个 rel 插件测试） |
| examples/ | 6 个示例演示目录 + 构建生成的索引页 |
| css/ | impress-common.css（插件公共样式）、impress-demo.css（仅 demo 用，README 强调非必需） |
| DOCUMENTATION.md / GettingStarted.md / npm-readme.md | Reference API / 入门指南 / npm 展示页 |
| build.js / karma.conf.js / qunit_test_runner.html | 构建、karma 配置、浏览器直跑测试入口 |

组织方式评述：**单包单库 + "拼接式"插件分发**。没有模块系统，插件按目录约定存放，build.js 把核心与插件 cat 成一个 IIFE 链；这是一种刻意反模块化的极简设计（README："impress.js itself doesn't depend on Node or any NPM modules"）。插件纪律严格：默认插件不允许携带 css/html/图片文件、不允许加全局变量（src/plugins/README.md "Structure, naming and policy"）。

## 5. 核心架构与关键实现

### 5.1 双元素"摄像机"模型与 transform 取反（src/impress.js）

impress.js 最关键的设计决策：`init()` 把 root 的所有子节点包进一个新建的 `canvas` div（src/impress.js:359-363），**root 只负责 scale + perspective，canvas 只负责 translate + rotate**。goto() 计算目标状态时对步骤变换做全量取反（src/impress.js:494-507）：

```js
// Compute target state of the canvas based on given step
var target = {
    rotate: {
        x: -step.rotate.x, y: -step.rotate.y, z: -step.rotate.z,
        order: step.rotate.order },
    translate: {
        x: -step.translate.x, y: -step.translate.y, z: -step.translate.z },
    scale: 1 / step.scale
};
```

"拉近"（zoomin）时 root 的缩放过渡延迟 duration/2、canvas 的位移立即开始；"拉远"时反过来（src/impress.js:509-557 的 zoomin/delay 逻辑与两个 css() 调用）——两段 CSS transition 错峰执行，产生先旋转位移后缩放的"自然"观感。perspective 除以 targetScale（src/impress.js:547），保证不同缩放级别下透视强度视觉一致。

transform 字符串由 98-122 行的 translate/rotate/scale 三个纯函数拼出：`translate3d(xpx,ypx,zpx)`、按 `data-rotate-order`（默认 xyz，可用 validateOrder 校验，src/impress.js:59-79）排列的 `rotateX/rotateY/rotateZ(deg)`、`scale(s)`。CSS 属性全部经 pfx()（Modernizr 思路的前缀探测，32-57 行）注入，兼容老 Webkit/Firefox。

### 5.2 goto 状态机与 stepenter 时序（src/impress.js:429-605）

goto() 每次调用都重新执行 `initAllSteps()`（440 行）重扫 DOM——步骤的 data-* 属性可以**动态修改甚至增删步骤**后立即生效。pre-stepleave 插件在此阶段同步运行，可改写 `event.detail.next/transitionDuration` 或返回 false 中止过渡（462-483 行；stop、substep、goto 三个插件依赖此钩子）。

一个值得记录的工程权衡：`impress:stepenter` 不用 `transitionend`，而用 `setTimeout(duration + delay)` 触发（599-602 行）。作者在注释中详细解释了放弃 transitionend 的原因——过渡分布在两个元素上、且 transitionend 只在值真正变化时触发，边界情况太多；并指路 0.5.2 版本查看旧实现。这是"简单性优先于新 API"的典型样本。此外还有窗口尺寸自适应 `computeWindowScale`（126-140 行，min/maxScale 夹逼）、`#/step-id` hash 路由（783-806 行，加 `/` 是为阻止浏览器原生锚点滚动）、future/present/past 三态 class（744-773 行）、触摸拖拽中途态插值 `swipe(pct)`（656-731 行，quaternion 不连续性在注释中被明确承认）。

### 5.3 插件体系：三种生命周期 + lib 工厂（src/plugins/README.md、src/impress.js:833-916）

插件 API 是本项目长期演进的骨架，src/plugins/README.md 给出四类插件契约：

1. **init 插件**：监听 `impress:init` 事件后自启（navigation、autoplay 等）。
2. **pre-init 插件**：`impress.addPreInitPlugin(fn, weight)` 注册，按 weight 升序在 init() 开头同步执行，是"过滤器"——典型如 rel 插件（src/plugins/rel/rel.js:284），把 `data-rel-x/y/z`（支持 `1.5w`/`1.5h` 屏宽屏高单位，util.js:89-102 toNumberAdvanced）换算成绝对 data-x/y/z 后再交给核心解析，还通过 lib.rotation.translateRelative 实现带旋转继承的相对定位。
3. **pre-stepleave 插件**：`impress.addPreStepLeavePlugin` 注册，在 goto() 开头运行，可改写目的地或中止过渡——goto 插件借此实现键盘映射的非线性跳转（`data-goto-key-list`/`data-goto-next-list`，src/plugins/goto/goto.js 头注），substep 插件借此实现同屏子步骤（返回 false 吃掉一次 next 并广播 `impress:substep:enter`，src/plugins/substep/substep.js:25-60）。
4. **GUI 插件**：仅当用户放置 `<div id="impress-插件名">` 占位符时才注入可见 UI（toolbar、progress）。

命名空间纪律：root 内用 `data-pluginname-*`，root 外用 `impress-pluginname`，事件用 `impress:pluginname:*`。库级扩展走 `impress.addLibraryFactory`（src/impress.js:834-854）：每个工厂按 rootId 实例化并缓存，重复注册同名库直接 throw——gc/util/rotation 三库即用此机制挂到 API 的 `lib` 命名空间。

### 5.4 gc.js：可完整还原的生命周期（src/lib/gc.js）

Henrik Ingo 2016 年加入的垃圾回收库是插件纪律的物质保障：插件不得直接 addEventListener/appendChild，必须走 `lib.gc.addEventListener/appendChild/pushCallback` 包装，全部入栈；`teardown()` 以 LIFO 执行回调、移除元素与监听器，并把 body class、meta viewport、自动生成的 step id、canvas 包裹层全部还原到 init 前状态（recordStartingState/resetStartingState，gc.js:114-236）。演示框架提供完整反初始化 API 在同类库中相当罕见。

### 5.5 rotation.js：四元数合并旋转（src/lib/rotation.js）

Tommy Tam 2021 年贡献的数学库，是 rel 插件"相对定位 + 旋转继承"的引擎：把多次旋转合成时，先用四元数（makeQuaternion/leftMulQuaternion/rotateByQuaternion）算出最终朝向的坐标系，再对 6 种旋转顺序（xyz/xzy/yxz/yzx/zxy/zyx）逐一反解出欧拉角序列，取与基准角差值最小者（possibleRotations/bestRotation/coordinateToRotation，rotation.js:262-407）。这解决了 CSS rotate 按轴序依赖欧拉角、同一姿态有多组解且过渡路径不同的问题。

## 6. 构建与运行

```bash
# 开发（README.md WANT TO CONTRIBUTE 一节）
npm install
npm run all        # = build + test + lint
npm run build      # node build.js → js/impress.js + impress.min.js(+map) + examples/index.html
npm run lint       # jshint + jscs（另 npm run new-lint 走 eslint）
npm test           # karma start --single-run，puppeteer Chrome + Firefox

# 手工跑测试（README 提示比 karma 更直观）
firefox qunit_test_runner.html

# 使用（GettingStarted.md / DOCUMENTATION.md）
# HTML 中引入构建产物，root + .step 即可，无需 CSS：
#   <div id="impress"><div class="step" data-x="0" data-y="0">...</div>...</div>
#   impress().init();
```

CDN 直链（README.md）：`https://cdn.jsdelivr.net/gh/impress/impress.js@2.0.0/js/impress.js`。

## 7. 测试与工程质量

- **测试形态**：QUnit 在 karma 中运行（Chrome/Firefox 双浏览器，karma.conf.js），测试加载**构建后的 js/impress.js** 到 iframe 中做行为断言——"测试用户实际使用的东西"是明确的工程约定（src/plugins/README.md Testing 一节）。syn 库合成真实键盘/鼠标事件，因此属于浏览器级端到端测试而非纯单元测试。
- **规模**：核心测试 core_tests.js 266 行 + non_default.js 144 行 + navigation 插件测试 + test/plugins/rel/ 4 个文件（含 rotation 数学测试）；相对 22 个插件的总面积覆盖偏薄，README 也自认"more tests are much welcome"（README.md REPOSITORY STRUCTURE）。
- **质量观察**：代码风格高度统一（jscs/jshint 双门禁 + 9KB eslint 配置）；注释密度极高且多为解释"为什么"；存在少量已知重复（substep.js:13 自注 triggerEvent 是从核心复制的，"Good candidate for moving to src/lib/util.js"）。版本号不一致是一处真实瑕疵：package.json 为 1.1.0，而 src/impress.js:16 头注释标 2.0.0；DOCUMENTATION.md 开头"默认 1024x768"的表述也与代码 defaults（1920x1080，src/impress.js:170-179）不一致（其下方的 NOTE 补充了 v1.2.0 改为 HD 的说明）。
- **CI**：GitHub Actions 三段流水线（lint → test → build，node 22）+ CircleCI 双轨（.github/workflows/pipeline.yaml、.circleci/config.yml）。

## 8. 社区与维护状态

- **创作者**：Bartek Szopka（@bartaz）2011-12-28 创建（gh api created_at；README 自述 2011-2012 版权）；2016 年起 Henrik Ingo（@henrikingo）接棒维护（src/impress.js:9 版权行"2016-2023 Henrik Ingo"），最新提交（2026-07-23）作者为 Hashim Khan（gh api commits）。
- **活跃度**：最新 release v2.0.0 发布于 2022-07-22，此后近 4 年无版本发布（gh api releases 全表：1.0.0 2018 → 1.1.0 2020 → 2.0.0 2022）；HEAD 最近一次 push 为 2026-07-23（距分析日约 4 个月），且该提交仅为工具栏文档修正——**"维护缓慢"得到 gh api 直接证实**。当前 61 个 open issues，仓库未归档（gh api）。
- **社区**：93 位贡献者（contributors API）；README 列有邮件列表、wiki 示例页与 Packt 出版的第三方图书；姊妹编辑器项目 Impressionist 已随主线一同沉寂（未确认具体状态）。
- **生态**：extras 子模块（impress/impress-extras，19 星）最后 push 停留在 2017-11（gh api）——插件生态的高峰期在 2016-2018 年。

## 9. 亮点与风险

**亮点**
1. **零依赖的极小核心**：核心 918 行 + 3 个 lib + 22 个短插件全部为原生 JS，任何人一个下午可读完全部源码，是学习 CSS 3D transforms、自定义事件与插件设计的优秀教材（src/ 全目录）。
2. **真正的三维自由度**：步骤可在 3D 空间任意平移/旋转/缩放，配合双元素错峰过渡产生 Prezi 式镜头感——与 Reveal.js 的二维横纵切换形成清晰差异（src/impress.js goto 实现 vs Reveal 的 slide 隐喻）。
3. **完整且文档化的插件生命周期**：pre-init / init / pre-stepleave / GUI 四类钩子 + lib 工厂 + 命名空间纪律 + 严格的"默认插件零静态资源"规则（src/plugins/README.md），是小型 JS 插件架构的范本。
4. **可完整卸载**：lib.gc.teardown() 把 DOM 还原到 init 之前，支持同一页面多次 init/tear（src/lib/gc.js）。
5. **注释即文档**：index.html 官方 demo 逐行注释 + DOCUMENTATION.md Reference API + GettingStarted.md，上手成本低。

**风险/局限**
1. **维护近乎停滞**：3 年无 release、extras 生态 2017 年起未更新、构建链陈旧（karma 已停止维护、jscs 项目早已废弃、puppeteer 2.x / eslint 6 均为数年前版本，package.json）。
2. **工程债**：版本号三处不一致（package.json 1.1.0 / 源码头注释 2.0.0 / DOCUMENTATION.md 残留 1024x768 旧默认值）；核心测试 + 插件测试合计不足千行，覆盖相对插件面偏薄。
3. **技术形态老化**：ES5 IIFE、无模块系统、无 TypeScript，二次开发只能跟随拼接式构建；`document.createEvent/initCustomEvent` 属于已废弃 API（src/lib/util.js:107，现代浏览器仍兼容但属弃用路径）。
4. **场景受限**：body overflow hidden + 整页 3D 变换的形态对网页嵌入、可访问性（屏幕阅读器/深链接之外的导航）不友好，README 自己也提示嵌入网页时部分访客可能无法正确观看。

## 10. 总结评价

impress.js 是"一个人 + 一个天才想法 + 克制执行"的典范项目：2011 年底用不到千行代码把 CSS3 3D 变换推向大众，成为 Prezi 的开源镜像和一代前端人的 CSS3 启蒙教材；2016 年 Henrik Ingo 接手后用严谨的插件生命周期把它从个人 demo 养成 22 插件的完整演示平台（v2.0.0，2022）。如今它已进入稳定维护期——三年无 release、生态冻结，但核心代码依然完好可用、教学价值不衰减。它适合三类人：想要"哇"效果且内容自持的演示者（classic-slides 模板可平替 PPT）、想精读 CSS 3D + 四元数 + 插件架构的 JS 学习者、以及需要极简零依赖嵌入方案的开发者。若你需要活跃社区、主题市场或与框架集成，应选 Reveal.js；若你想要的是空间叙事与一段可以逐行读完的优秀源码，impress.js 仍然是唯一答案。
