# Dogfalo/materialize 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 SCSS｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：Materialize 是 2014 年诞生的经典 Material Design CSS 框架（SCSS + vanilla JS 交互组件），曾是与 Bootstrap 并列的前端 UI 框架选择，v1.0.0（2018-09）后项目实质冻结，成为"化石级"但星数长青的老牌仓库。

| 项目 | 内容 |
|---|---|
| 名称 | Dogfalo/materialize（npm 包名 `materialize-css`，package.json） |
| 星数 | 39500（快照）；gh api 实查 38768（2026-11-14） |
| 语言 | SCSS（sass/ 约 6300 行）+ JavaScript（js/ 约 11200 行） |
| 许可证 | MIT（LICENSE，版权声明 "2014-2019 Materialize"） |
| 默认分支 | v1-dev（gh api 核实；master 为 v0.100.2 遗留线） |
| 最近代码提交 | 2020-06-01（v1-dev 最后一次提交，内容仅为 Patreon 更新） |
| 最后发版 | 1.0.0，2018-09-09（gh api releases 核实） |
| 贡献者 | 239 人（gh api contributors 分页核实） |
| 维护状态 | 代码冻结约 6 年；gh-pages 分支仍每月更新 Patreon 赞助横幅（2026-08-20 "patreon update august 2026"）；后继组织 materializecss/materialize 接棒发布 v2.x（2026-09 已发 v2.4.0） |

## 2. 项目是什么，解决什么问题

Materialize 是一个基于 Google Material Design 规范的 CSS 框架，诞生于 2014 年 9 月（仓库创建 2014-09-12，恰在 Google 2014 I/O 发布 Material Design 规范三个月后，gh api 核实）。它提供两大产出物：一套完整的 SCSS 样式系统（编译为 materialize.css，涵盖配色、栅格、排版、阴影、组件外观）和一组配套的 JS 交互组件（编译为 materialize.js，提供下拉、模态框、侧边导航等行为），让开发者无需理解 Material Design 细节即可快速搭出符合该设计语言的页面（README.md：http://materializecss.com/）。

与 Bootstrap 的"工具箱式"中立风格不同，Materialize 从变量到组件都强绑定 Material Design 1.0 视觉语言：Material 配色板（sass/components/_color-variables.scss）、涟漪点击效果 waves（sass/components/_waves.scss、js/waves.js）、悬浮按钮 FAB、卡片阴影、Roboto 字体式排版、侧边抽屉 sidenav 等，都是规范的时代印记。这意味着开箱即得"谷歌味"界面，但也意味着视觉上难以跳出 Material 范式。

JS 侧的定位是"CSS 组件的行为层"：每个组件遵循 HTML 类名约定 + JS 初始化的模式（如 `.modal`、`.dropdown-trigger`、`.sidenav`），v1.0.0 后通过 `M.AutoInit()`（js/global.js:106）按类名扫描自动实例化。这一"类名即契约"的模式贯穿全部 20 余个交互组件。

该项目的衰落路径同样有代表性：v1.0.0 之后核心维护者（Alvin Wang、Alan Chang，package.json author）投入锐减，2019 年起仅有零星 Patreon 相关提交，2018-09 后再无任何 release；而被冻结的 Material Design 1.0 视觉与 jQuery 时代架构也随前端技术栈迁移（组件化框架、CSS-in-JS、Material Design 2/3）逐渐退出主流。

## 3. 技术栈与依赖

**运行时（npm `dependencies: {}` 为空，package.json）：**
- SCSS：sass/ 目录约 30 个组件样式 + forms/ 8 个表单样式，经 node-sass（grunt-sass）编译
- JS：ES6 类组件，经 Babel（babel-preset-es2015，Gruntfile.js）转译输出 dist/js/materialize.js
- 内嵌第三方库（vendor 进 js/，非 npm 依赖）：
  - `js/cash.js`：cash-dom 1.3.5（文件头注释），960 行的 jQuery 兼容精简 DOM 库——v1.0.0 移除 jQuery 硬依赖后改用它
  - `js/anime.min.js`：anime.js v2.2.0（文件头注释），JS 动画引擎，供组件 open/close 动画调用
- 遗留兼容：`M.initializeJqueryWrapper`（js/global.js:65）在页面已有 jQuery 时把组件注册为 jQuery 插件，22 个组件文件调用该函数（grep 统计），保留 jQuery 生态兼容

**构建/测试工具链（devDependencies，package.json）：**
- Grunt 1.x 全家桶：grunt-sass、grunt-contrib-concat、grunt-babel、grunt-contrib-uglify、grunt-contrib-jade（文档站点）、grunt-contrib-compress（release zip）、grunt-browser-sync（本地开发）
- 测试：Jasmine 2.6 + jasmine-jquery + PhantomJS 2.1（phantomjs-prebuilt）——均为已停更的旧工具
- 质量工具：ESLint（.eslintrc）、Prettier（.prettierrc、lint-staged + husky 0.14 precommit）

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| sass/ | 样式源码：materialize.scss 为唯一入口，@import 引入 components/ 下 29 个局部样式 + forms/ 8 个表单样式；_style.scss（1345 行）为官方文档站样式，ghpages-materialize.scss 为文档站编译入口 |
| js/ | JS 组件源码：27 个组件文件 + component.js 基类（44 行）+ global.js（M 命名空间/AutoInit/jQuery 包装，444 行）+ 内嵌的 cash.js 与 anime.min.js |
| dist/ | 构建产物：materialize.css/.min.css、materialize.js/.min.js（提交入库，banner 显示 v1.0.0） |
| jade/ | 文档站源码：58 个 .jade 页面（buttons.jade、modal.jade 等，每组件一页）+ 布局片段（_head.jade、_navbar.jade） |
| docs/ | 文档站 JS 资源 |
| templates/ | 示例模板：starter-template、parallax-template、masonry-template（含 zip 打包） |
| extras/ | 外部依赖库：noUiSlider（range 滑块组件的底层） |
| tests/spec/ | Jasmine 测试：14 个组件的 *Spec.js |
| test/html/ | 测试用 HTML fixture |
| .github/ | 仅 ISSUE_TEMPLATE.md 与 PULL_REQUEST_TEMPLATE.md（无 CI workflow 文件） |
| .travis.yml | Travis CI 配置（node 6.10，grunt travis） |

组织方式评述：典型的单包单体仓库，一个入口文件串联全部样式、一个 concat+babel 管线串联全部 JS，无模块打包器（无 webpack/rollup 发布产物——分支列表中有一个未完成的 rollup 分支，gh api 核实，可佐证现代化尝试被放弃）。文档站点用已淘汰的 Jade/Pug 模板引擎构建，与源码同仓。

## 5. 核心架构与关键实现

### 5.1 SCSS 组件组织与变量体系

`sass/materialize.scss`（41 行）是唯一入口，按"配色 → 变量 → normalize → 组件"顺序 @import 29 个局部样式；`sass/components/_variables.scss`（349 行）集中管理全部可定制变量，带 24 节目录注释。主配色定义为 Material Design 风格的色板函数调用：

```scss
// sass/components/_variables.scss
$primary-color: color("materialize-red", "lighten-2") !default;
$primary-color-light: lighten($primary-color, 15%) !default;
$secondary-color: color("teal", "lighten-1") !default;
$success-color: color("green", "base") !default;
$error-color: color("red", "base") !default;
```

`_color-variables.scss` 内置完整 Material 色板（red/teal/blue-grey 等数十种 base + lighten/darken 阶），`_color-classes.scss` 据此生成 `.red.lighten-2` 类的笛卡尔积。栅格为 12 列体系（`$num-cols: 12`，_variables.scss:236）。全部变量带 `!default`，支持用户 Sass 覆盖定制——这是 CSS 框架时代的标准定制方案（对比现代 CSS 自定义属性）。

### 5.2 组件基类与初始化架构

JS 组件统一继承 `Component` 基类（js/component.js，44 行），核心设计：构造时若元素上已有实例则先 destroy 再重建（幂等初始化），静态 `init` 方法统一接受 Element/NodeList/jQuery/cash 集合：

```js
// js/component.js
static init(classDef, els, options) {
  let instances = null;
  if (els instanceof Element) {
    instances = new classDef(els, options);
  } else if (!!els && (els.jquery || els.cash || els instanceof NodeList)) {
    let instancesArr = [];
    for (let i = 0; i < els.length; i++) {
      instancesArr.push(new classDef(els[i], options));
    }
    instances = instancesArr;
  }
  return instances;
}
```

每个组件的文件结构高度一致（以 js/dropdown.js 为例）：IIFE `(function($, anim){...})(cash, M.anime)` 闭包内定义 defaults、ES6 class、实例挂在 `el.M_Dropdown`、静态注册表 `Dropdown._dropdowns`、最后 `M.Dropdown = Dropdown` 并在 jQuery 存在时注册 jQuery 插件。生命周期回调采用四段式命名（onOpenStart/onOpenEnd/onCloseStart/onCloseEnd，js/dropdown.js:9-20 defaults）。这套模式与 jQuery Widget Factory 一脉相承，是 v1.0.0 的主要现代化改造（v1-upgrade-guide.md：移除自动初始化、`data-activates` 改 `data-target`、`material_chip` 改名 `chips` 等）。

### 5.3 M 命名空间与 AutoInit

`js/global.js`（444 行）承担全局基建：AMD/CommonJS/Meteor 三环境导出 `M` 对象；键盘状态跟踪（M.tabPressed、body.keyboard-focused 类，js/global.js:30-50）；`M.AutoInit(context)` 通过"类名选择器 → 组件类"注册表批量初始化 18 个组件（js/global.js:106-140），支持 `.no-autoinit` 排除类。该方法的存在是因为 v1 移除了 0.x 时代 document 加载时的全自动初始化（v1-upgrade-guide.md "Auto Init" 节），改为显式调用，兼顾了 SPA 场景的可控性。

### 5.4 jQuery 遗留依赖的处理

v1.0.0 是项目对 jQuery 时代的告别尝试：`js/cash.js`（cash-dom 1.3.5，960 行）作为轻量 jQuery 替身内嵌，所有组件用 `$`（实为 cash）操作 DOM；同时 `M.initializeJqueryWrapper`（js/global.js:65 起）保留对真 jQuery 的兼容——若页面加载了 jQuery（`M.jQueryLoaded`，global.js:8），22 个组件自动注册为 `$(el).modal()` 式插件。测试环境则直接喂入真实 jQuery（Gruntfile.js jasmine.vendor 配置 node_modules/jquery），双轨验证兼容性。

## 6. 构建与运行

来自 package.json scripts 与 Gruntfile.js：

```bash
npm install          # node >= 6（engine 约定，package.json）
npm run dev          # grunt monitor：browserSync + sass/babel/jade watch
npm test             # grunt travis = js_compile + sass_compile + jasmine
# 构建细节（Gruntfile.js registerTask，行 635-671）：
grunt release        # 完整发布流程：编译 + uglify + 压缩 zip
grunt js_compile     # concat → babel（configureBabel）→ clean
grunt sass_compile   # node-sass 输出 dist/css/materialize.css/.min.css
grunt jade_compile   # jade/ 文档站编译
```

文档站本地运行：README.md 提供安装说明（npm i + grunt 命令，依赖 node-sass/phantomjs 等停更依赖，在当前 Node 版本上大概率构建失败——未验证，仅静态推断）。使用方式：直接引 dist/ 产物或 CDN，或 npm install materialize-css（README.md Quickstart）。

## 7. 测试与工程质量

- 测试框架：Jasmine 2.6 + jasmine-jquery，PhantomJS 2.1 无头浏览器（Gruntfile.js、package.json devDependencies）
- 覆盖规模：tests/spec/ 下 14 个组件各一个 *Spec.js（autocomplete、cards、carousel、chips、collapsible、dropdown、fab、materialbox、modal、select、sidenav、tabs、toast、tooltip），27 个 JS 组件中约半数有测试；datepicker/timepicker/slider/waves/scrollspy 等无 spec（目录清单核实）
- CI：.travis.yml 单节点 6.10 环境跑 grunt travis；.github/ 下无 Actions workflow
- 质量设施：ESLint + Prettier + lint-staged/husky precommit（package.json）；代码风格统一（全库 ES6 类 + JSDoc 注释，组件文件内 options 文档化程度很高，如 js/dropdown.js:32-47 逐项 @prop 注释）
- 评价：以 2018 年标准是规范的前端框架工程；以当下标准，PhantomJS/node-sass/node6 工具链已完全不可运行，测试对现代浏览器的参考价值有限

## 8. 社区与维护状态

活跃度（gh api 核实，2026-11-14）：
- 创建 2014-09-12；星数 38768（快照 39500，两年间微降约 700——星数见顶后缓慢回落）；fork 4575；open issues 790（长期无人处理）
- 代码线分裂：master 停在 v0.100.2 时代（最后代码提交 2019-04-23，Chrome 73 补丁）；v1-dev（默认分支）最后代码提交 2020-06-01；此后 v1-dev 上的"提交"仅为 Patreon 月度赞助更新（2020-02 至 2020-06 共 5 条，作者 Alan Chang）
- 唯一仍在进行的仓库活动是 gh-pages 分支的月度 Patreon 横幅更新，持续至 2026-08-20（"patreon update august 2026"）——即项目冻结 6 年间仍维持着赞助收款页面
- release 历程：2018 年密集发布 1.0.0 的 alpha→beta→rc→正式版 5 个 release 后彻底停止；44 个 tag 终结于 1.0.0/v0.100.2
- 后继生态：新组织 materializecss/materialize（创建 2020-05-02，994 星）接棒开发 Materialize 2.x，2026-09-24 已发布 v2.4.0（gh api releases 核实）——旧仓库本质上是"历史归档 + 跳转入口"，官方文档站与 2.x 演进都由新组织承接
- 贡献者结构：239 位贡献者（gh api 分页核实），v1.0.0 由两位核心维护者（Alvin Wang、Alan Chang，package.json author 字段）主导

## 9. 亮点与风险

**亮点：**
1. 教科书级的 SCSS 组件组织：单入口 + 29 个内聚局部样式 + 集中式带 `!default` 变量表（sass/materialize.scss、sass/components/_variables.scss），定制路径清晰
2. 统一而一致的 JS 组件架构：44 行基类 + 严格模板化的 27 个组件文件，四段式生命周期回调与 destroy/init 幂等初始化，可读性与可仿写性极强
3. 平滑的 jQuery 时代过渡设计：内嵌 cash-dom 替代 jQuery，同时保留 jQuery 插件兼容层（js/cash.js、js/global.js），兼容新旧两种生态
4. 文档即工程：58 个 Jade 页面每组件一页，与源码同仓同步演进（jade/），对组件学习价值高
5. 历史地位确凿：Material Design 1.0 时期最具代表性的开源 CSS 框架之一，239 位贡献者、近 4 万星是那个时代的直接遗产

**风险/局限：**
1. 项目死亡超过 6 年：最后 release 2018-09、最后代码提交 2020-06（gh api 核实），790 个 open issues 无人处理，无任何安全与兼容性修复（如对 2018 年后浏览器 API 变更零响应）
2. 工具链完全过时：node-sass、PhantomJS、Jade、Node 6、Babel es2015 preset 均已停更，本机构建大概率失败（第 6 节静态推断）
3. 视觉冻结在 Material Design 1.0：无 MD2/MD3 支持，与当前 Google 官方设计语言已明显脱节
4. 老仓与新 2.x 的割裂：materializecss/materialize（2.x）与旧 npm 包 materialize-css 名称相近但互不兼容，按旧仓 README 走会停在 1.0.0

## 10. 总结评价

Materialize 是前端框架史上一块保存完好的琥珀：它完整记录了 2014-2018 年间"CSS 框架 + jQuery 式插件"这一主流技术范式的最佳实践——SCSS 组件化组织、类名契约、基类继承、vendored 轻量 DOM 库，代码质量即便放到今天阅读依然清晰。但它同时是"星数长青与实际维护死亡分离"的典型样本：近 4 万星是历史收藏的沉淀，仓库本身自 2018 年发版、2020 年停码后只剩每月一次的 Patreon 横幅更新，其真实使用者早已流向 Bootstrap 5、Tailwind、MUI 或新组织下的 Materialize 2.x。适合两类人：想研究"Material Design 1.0 + SCSS + jQuery 过渡期架构"的框架考古者（读源码价值高），以及维护着旧系统、需要冻结版本 reference 的存量项目开发者；新项目不应选它，应转向其官方后继 materializecss 2.x 或现代替代品。源码仿写建议精读三个文件即可得全貌：js/component.js、js/global.js、sass/components/_variables.scss。
