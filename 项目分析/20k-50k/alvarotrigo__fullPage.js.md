# alvarotrigo/fullPage.js 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：无框架依赖的全屏滚动网页插件，把整页 HTML 变成"一屏一节"的整页滑动演示，是这一品类的开创者与事实标准。

| 项目 | 内容 |
|---|---|
| 名称 | alvarotrigo/fullPage.js |
| 星数 | 35354（gh api 实时值 2026-11-21，任务快照 39500） |
| 语言 | JavaScript（gh api 核实） |
| 许可证 | GPL-3.0（开源版）+ 商业许可证双轨（LICENSE 文件 + README 第 70-75 行） |
| 版本 | 4.0.41（package.json，发布于 2026-03-04，gh api releases） |
| 默认分支 | master（gh api） |
| 最近活跃 | 最近 push 2026-09-20（gh api）；最近提交 2026-03-04 发布 4.0.41（gh api commits） |
| 创建时间 | 2013-09-20（gh api），13 年长期项目 |

## 2. 项目是什么，解决什么问题

fullPage.js 解决的问题非常具体：设计师与营销页开发者想要"一屏一幕、滚动即翻页"的全屏体验——滚轮一格翻一节、触摸滑动翻页、键盘方向键导航、侧边圆点导航，而浏览器原生滚动无法提供这种"吸附式"的交互。fullPage.js 把这套体验封装成一个零依赖的 JS 函数调用：`new fullpage('#container', options)`，配合约定的 HTML 结构（`section` + 内嵌 `slide`），即可获得完整的整页滚动系统（README "Required HTML structure" 一节，README.md 第 116 行起）。

它的核心价值在于把大量难啃的细节全部内置：滚轮/触摸板惯性加速度判定（避免触控板一次滑动连翻数节）、横纵向手势区分、节内超长内容的内部滚动（scrollOverflow）、锚点与 URL 同步、懒加载图片与媒体、响应式降级（窄屏时退回普通滚动）、以及节/页级生命周期回调（beforeLeave/onLeave/afterLoad 等）。这些正是"看起来简单、实现起来全是坑"的部分，也是它能在 13 年里保持该品类头部地位的原因（src/js/mouse/wheel.js、src/js/touch.js、src/js/scrolloverflow.js）。

项目自 v3 起完成了一次彻底重写：从单文件巨型脚本变为 ES 模块化代码库（src/js/ 下约 100 个文件，每个文件单一职责），并以事件总线贯穿各子系统。v4 延续该架构并持续加入新扩展（如 4.0.41 新增 cinematic 扩展选项，gh api commits："new option cinematic to use the Cinematic extension"）。

需要特别指出的是它的商业模式：开源版为 GPL-3.0，要求在非开源项目中购买商业许可证（licenseKey 选项为"compulsory"，README 第 395 行），且开源版会在页面右下角注入 "Made with fullPage.js" 水印（src/js/mixed/waterMark.js）。进阶功能（横向连续滚动、拖拽翻页、视差、卡片 3D、淡入淡出切换等）以"付费扩展"形式提供，需使用 dist/fullpage.extensions.min.js 构建并配合激活密钥（README "Use extensions" 一节，第 375-387 行）。GPL 主仓库本身是完整可用的，扩展只是增强。

## 3. 技术栈与依赖

- 语言：ES 模块化 JavaScript（ES2020 语法 + 自带 polyfills，src/js/polyfills/ 含 array.from、object.assign、string.padStart 等）。
- 运行时依赖：零。全部 devDependencies，无任何 dependencies（package.json），运行时不依赖 jQuery；jQuery 支持通过适配器兼容（src/js/jquery-adaptor.js，注册 `$.fn.fullpage`）。
- 构建链：Rollup（@rollup/plugin-babel、@rollup/plugin-commonjs、@rollup/plugin-node-resolve、rollup-plugin-terser）+ Gulp 4（clean-css、rename、replace、sourcemaps、terser、uglify）+ Babel preset-env（package.json devDependencies；rollup.config.js）。
- 测试：QUnit（浏览器端运行，tests/index.html 引入 qunit-2.17.1）+ jQuery 辅助断言。
- 类型：手写 TypeScript 声明 types/index.d.ts（251 行），package.json `types` 字段指向它。
- 唯一 vendored 代码：vendors/easings.js（缓动函数，供 JS 动画模式使用）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| src/js/ | 全部核心源码，约 100 个 ES 模块，按领域分子目录 |
| src/js/scroll/ | 滚动引擎：scrollPage、moveTo、scrollHandler、silentMove 等 12 个模块 |
| src/js/mouse/、src/js/touch.js、src/js/keyboard/ | 输入层：滚轮、鼠标、触摸、键盘事件捕获与手势判定 |
| src/js/common/ | 基础设施：eventEmitter、state、options、cache、selectors、utils、transformContainer |
| src/js/slides/ | 横向 slide 系统：landscapeScroll、moveSlide、scrollSlider |
| src/js/scrolloverflow.js | 节内容超长时的内部滚动处理 |
| src/js/lazyLoad/、src/js/media.js | 图片懒加载与音视频自动播放/暂停 |
| src/js/menu/、src/js/nav/ | 导航菜单高亮与圆点导航生成 |
| src/js/anchors/ | 锚点与 URL 状态同步 |
| src/js/responsive.js、src/js/fitToSection.js、src/js/resize.js | 响应式降级与自适应 |
| src/js/beyondFullpage/、src/js/infiniteScroll/ | 整页之后的自然滚动、continuousVertical 无限循环 |
| src/js/mixed/ | 水印 waterMark.js、许可证警告 warnings.js（商业双轨相关） |
| src/css/、dist/ | 源 CSS 与构建产物（fullpage.js 204KB / fullpage.min.js 59KB / fullpage.extensions.min.js 69KB） |
| tests/unit/ | 36 个 QUnit 测试文件（浏览器运行） |
| examples/ | 60 个示例 HTML（从 angular 集成到各类背景/回调用法） |
| types/ | index.d.ts 类型定义 |
| gulp/、rollup.config.js | 构建脚本 |

组织方式评述：典型的"单包 + 领域模块化"结构，不是 monorepo。模块间不直接互调，而是通过全局单例 `FP`（common/constants.js 中定义，最终挂到 `win.fullpage_api`）与 EventEmitter 事件总线（bindEvents、onLeave、afterSectionLoads、onDestroy 等生命周期事件）解耦，形成了插件式的初始化流程：`fullpage.js` 入口只做 15 次 `import './xxx/index.js'` 让各子系统自注册副作用（src/js/fullpage.js 第 1-15 行）。这一设计让开源/商业构建可以用同一套源码按需打包。

## 5. 核心架构与关键实现

### 5.1 入口与生命周期：副作用导入 + 事件总线

入口 `fullpage(containerSelector, options)`（src/js/fullpage.js）只负责设缓存、合并默认选项、发 `onInitialise` 事件，然后 `init()`（src/js/instance.js）依次执行 prepareDom → setAllowScrolling → setMouseHijack → setAutoScrolling → responsive → scrollToAnchor。所有子系统（wheel、touch、keyboard、menu、nav、scrollOverflow、waterMark……）都在模块顶层 `EventEmitter.on(events.bindEvents, init)` 订阅生命周期，销毁时统一响应 `onDestroy`。这是一套精巧的手工 IoC：没有框架，靠 import 副作用完成依赖装配。EventEmitter 本身仅 40 行 on/once/emit/removeListener（src/js/common/eventEmitter.js）。

### 5.2 滚轮引擎：加速度判定防误触

滚轮劫持是 fullPage 的灵魂。src/js/mouse/wheel.js 的 `MouseWheelHandler` 做了几层关键处理：

1. 跨浏览器事件探测（wheel / mousewheel / DOMMouseScroll / MozMousePixelScroll，第 55-58 行）；
2. 纵横手势区分：deltaX/deltaY 大小比较 + Shift 键检测，横向增量映射为 slide 翻页（第 158-171 行）；
3. 加速度判定防止触控板惯性连翻——维护最近 150 条滚动值队列，仅当"末段均值 ≥ 中段均值"（滚动力度在增强，说明是一次主动的新滚动）才触发翻节（第 109-113 行 `isAccelerating`）：

```js
function isAccelerating(scrollings) {
    var averageEnd = utils.getAverage(scrollings, 10);
    var averageMiddle = utils.getAverage(scrollings, 70);
    return averageEnd >= averageMiddle;
}
```

4. 两次滚动间隔 >200ms 即清空队列（第 219-222 行），视为新手势；
5. `state.canScroll` 单飞锁：动画期间禁止再次触发（scrollPage.js 第 129 行）。

触摸端同理：src/js/touch.js 记录 touchstart/touchmove 位移，与 touchSensitivity 阈值比较决定翻页方向，并对 body 绑定 `{passive:false}` 的 preventBouncing 阻止浏览器橡皮筋（touch.js 第 46-57 行）。

### 5.3 滚动执行引擎：CSS3 transform 与 JS 动画双通道

src/js/scroll/scrollPage.js 是翻节核心。`scrollPage()` 先构造上下文对象 `v`（目标位置、方向、离开节索引、activeSlide 等），触发 `beforeLeave`/`onLeave` 回调（回调返回 false 可取消滚动，第 97-106 行），然后 `performMovement()` 按配置走两条通道：默认 `css3:true` 时对容器施加 `translate3d(0, -dtop, 0)` 硬件加速位移（第 203-223 行），否则用 JS scrollTo 动画驱动真实 scrollTop。`afterSectionLoads` 统一收尾：修复 continuousVertical 的节顺序、触发 afterLoad、播放媒体、懒加载、解锁 canScroll（第 266-293 行）。对超出视口的"大节"，`getDestinationPosition` 结合 `bigSectionsDestination`（top/bottom）决定对齐边（第 154-186 行）。

一个有趣的工程细节：快速滚动（scrollingSpeed<700ms）时不等动画结束就提前恢复 canScroll，避免快速翻页时出现"吞输入"的顿挫感（performMovement 第 199-200、255-260 行）。

### 5.4 商业双轨机制：水印、许可证警告与扩展构建

开源/商业的分界在代码里有明确落点：

- **水印**：初始化时若 `state.isValid` 为 false 且未关闭 credits，就向最后一节注入 "Made with fullPage.js" 链接（src/js/mixed/waterMark.js，第 22 行 `shouldUseWaterMark = !state.isValid || getOptions().credits.enabled`）。
- **许可证警告**：licenseKey 无效时向 body 注入 "The license key for fullPage.js is missing or is not valid" 警告条（src/js/mixed/warnings.js）。
- **扩展门控**：默认选项表（src/js/optionsDefault.js）中 `continuousHorizontal`、`scrollHorizontally`、`interlockedSlides`、`dragAndMove`、`offsetSections`、`resetSliders`、`fadingEffect`、`responsiveSlides`、`parallax`、`cards`、`effects`、`cinematic` 等选项开源版均置 false 且不生效，仅注释标注为 "Extension of fullpage.js"（README 第 524-652 行逐一说明）。
- **私有构建配置**：package.json 的 `extensions`/`limited` 脚本引用 `rollup.ext.config.js`、`rollup.limited.config.js`，但这两个文件不在仓库中（仅 rollup.config.js 存在）——付费扩展的打包在私有流程完成，仓库里预置了产物 dist/fullpage.extensions.min.js（69KB）供购买者使用。混淆配置中保留了全部扩展选项名（rollup.config.js terser reserved 列表），暗示扩展代码以同名选项接入。

客观地说，这是一种成熟的 open-core 模式：GPL 主库功能完整、无功能阉割的"时间炸弹"，商业价值集中在增值扩展与商用授权，同时以水印/警告维持转化压力。

## 6. 构建与运行

来自 package.json scripts 与 README：

```bash
npm install
npm run dev        # rollup --watch 开发模式
npm run build      # gulp getDistFiles/update-version/css + rollup -c
npm test           # jest（注意：devDependencies 未包含 jest，见第 7 章观察）
```

运行示例：直接浏览器打开 examples/*.html（60 个场景示例）；生产使用 dist/fullpage.min.js + fullpage.min.css，HTML 按约定结构组织 `#fullpage > .section(.slide)` 后调用 `new fullpage('#fullpage', {...})`（README "Usage"/"Initialization" 一节，第 82-165 行）。

## 7. 测试与工程质量

- 测试规模：tests/unit/ 下 36 个 QUnit 文件，覆盖 moveTo、lazyload、responsive、callbacks、destroy、锚点锁定等 API 行为，属"真实 DOM + 浏览器断言"风格（tests/unit/moveTo.js：`QUnit.test(...)` + transform 断言），由 tests/index.html 在浏览器中运行。
- 工程瑕疵：package.json 的 `"test": "jest"` 与实际 QUnit 体系不符，devDependencies 中也没有 jest——CI 或本地直接 `npm test` 并不可用（"未确认"作者是否在别处跑 jest）。此外 .github/ 仅有 issue/PR 模板，仓库内无 GitHub Actions 工作流，测试疑似不在公开 CI 中强制执行（.github/ 目录列举核实）。
- 代码质量观察：源码风格统一（JSDoc、领域目录、单一职责文件），对 IE 系与老 Firefox 仍保留兼容分支；TS 类型为手写声明文件而非 TS 源码；60 个 examples 兼作文档。整体是"老派但扎实"的前端插件工程。

## 8. 社区与维护状态

- 贡献者：contributors API 返回 100+（per_page=100 截断），但 gh api 最近 3 次提交作者均为 Alvaro Trigo（López），近期开发实质为作者单人维护，符合任务提示与社区认知（gh api commits 2026-03-04、2026-01-22）。
- Release 节奏：4.0.x 持续滚动发版——2025-03 至 2026-03 间发布 4.0.34 → 4.0.41 共 8 个 tag，最近一次 2026-03-04（gh api releases）；master 最近 push 2026-09-20，说明发版后仍有持续提交。
- 生态位：官方提供付费扩展站（alvarotrigo.com/fullPage/extensions/）、Wiki、React/Vue/Angular 包装（README 提及 angular-fullpage 示例，examples/angular-fullpage.html）。
- 社区压力：open issues 137（gh api），对一个 35k 星、单人维护的项目属可控水平。

## 9. 亮点与风险

亮点：

1. 零运行时依赖、双通道动画（CSS3 transform / JS scrollTop）、跨浏览器手势处理打磨 13 年，是该品类交互细节最全的实现（src/js/mouse/wheel.js、scrollPage.js）。
2. v3+ 的副作用导入 + EventEmitter 架构让无框架代码保持了高度的模块化与可裁剪性（src/js/fullpage.js、common/eventEmitter.js）。
3. 文档极其完整：62KB README 覆盖全部 70+ 选项、HTML 约定、状态类与回调，另有 60 个可运行示例（README.md、examples/）。
4. open-core 双轨设计清晰：GPL 主库完整可用，付费扩展与许可证机制代码路径分离明确（mixed/waterMark.js、warnings.js、dist/fullpage.extensions.min.js）。

风险/局限：

1. **GPL-3.0 商用合规**：闭源项目必须购买商业许可并配置 licenseKey，否则带水印与警告条；对不知情团队是合规陷阱（README 第 70-75、395 行）。
2. 强 DOM 劫持式交互的固有争议：原生滚动被接管，可访问性与 SEO 需要额外处理；键盘滚动、锚点行为与用户预期可能有偏差（keyboard/index.js、anchors/）。
3. `npm test` 脚本失效（jest 未声明），公开仓库无 CI 工作流，回归质量依赖作者手动验证（package.json、.github/）。
4. 扩展生态锁定：进阶功能需购买 extensions.min.js 与激活密钥，社区无法补位（README "Use extensions"）。

## 10. 总结评价

fullPage.js 是"一个作者用 13 年把一个交互细节做到极致"的典范项目：零依赖、模块化重写后的代码质量与 60 个示例、62KB 文档共同支撑起 35k+ 星与整页滚动品类的事实标准地位；其 GPL + 付费扩展的 open-core 双轨是独立开源商业化的经典样本。适合营销页/作品集/演示站开发者直接采用（注意商用授权），也适合前端工程师阅读其滚轮加速度判定、手势区分与事件总线架构——但对需要自由滚动、强 SEO 长文场景，以及希望绕过测试脚本去跑 CI 的团队，需要另做评估。
