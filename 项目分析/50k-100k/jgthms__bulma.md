# jgthms/bulma 项目分析

> 星数 50035（快照 2026-10-24）｜主语言 CSS（Sass）｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

Bulma 是一个**纯 CSS（无 JavaScript）的现代 UI 框架**，基于 Flexbox 与 CSS Grid 构建，以 Sass 作为开发语言，最终产物只有一个 CSS 文件。

| 事实 | 值 | 出处 |
| --- | --- | --- |
| 名称/版本 | jgthms/bulma，v1.0.4（CHANGELOG 中 1.1.0 开发中） | `package.json`、`CHANGELOG.md` |
| 星数 | 50035（快照 2026-10-24） | 任务清单 |
| 主语言 | SCSS（73 个 .scss 文件，约 11250 行）+ 少量 JS（仅测试/文档工具） | `sass/` 统计 |
| 许可证 | MIT（Copyright (c) 2023 Jeremy Thomas） | `LICENSE` |
| 创建时间 | 2016-01-23 | GitHub API `repos/jgthms/bulma` |
| 最近 push | 2026-09-21 | GitHub API（tarball 内文件 mtime 同为 2026-09-21，互相印证） |
| 最新 release | 1.0.4（2025-04-19）；此前 1.0.0~1.0.3 集中于 2024-03~2024-12 | GitHub API releases |
| 社区规模 | forks 3860，watch 569，open issues 532，贡献者 200+（contributors API 前两页均满 100） | GitHub API |

## 2. 项目是什么，解决什么问题

Bulma 解决的问题是：开发者需要一个开箱即用、风格现代的界面样式层，但不想被某个 JS 框架（React/Vue/Angular）或某个 JS 组件库的运行时绑定。README 明确写道 "There is **no** JavaScript included"，并把 Bulma 定位为 "environment agnostic"——只是叠加在业务逻辑之上的样式层（`README.md`）。整个仓库除 `test/` 与 `docs/` 的工程脚本外没有任何 `.js` 文件，这一设计承诺在源码层面得到实证。

它的核心价值有三层。第一是**组件体系完整**：`sass/elements/`（button、box、tag、table 等 14 类原子元素）、`sass/components/`（navbar、card、modal、tabs 等 11 类复合组件）、`sass/form/`（6 类表单控件）、`sass/layout/`（hero、section、level、media 等 5 类页面布局），覆盖了传统 CSS 框架的全部常用面。第二是**可定制性**：所有设计变量（色板、字号、间距、圆角）都以 `!default` 声明在 `sass/utilities/initial-variables.scss` 中，用户可用 Sass `@use ... with` 覆盖；v1.0 起更进一步，每个变量同时以 CSS 自定义属性（`--bulma-*`）暴露到运行时，不写 Sass 也能改主题。第三是**无侵入的类名体系**：`is-*`/`has-*` 修饰符风格，语义清晰且可通过构建变体加前缀避免冲突。

Bulma 诞生于 2016 年 Bootstrap 4 尚未稳定、Flexbox 刚普及的窗口期，凭借"纯 CSS + Flexbox + 好看"的差异化定位迅速成为 Bootstrap 之外最流行的选择之一。1.0.0（2024-03）是里程碑版本：引入 CSS 变量体系、明暗双主题、新的 Smart Grid（CSS Grid + 容器查询），摆脱了早期版本对 CSS 变量支持不足的历史包袱（`CHANGELOG.md`）。

## 3. 技术栈与依赖

- **开发语言**：SCSS（Dart Sass 语法，`@use`/`@forward` 模块系统，无 `@import` 遗留）。
- **构建工具链**（`package.json` devDependencies，全部为构建期工具，运行时零依赖）：
  - `sass ^1.97.3`：Dart Sass 编译器，`npm run build-bulma` 一条命令产出 `css/bulma.css`；
  - `postcss-cli ^11` + `cssnano ^7`：压缩产出 `css/bulma.min.css`；
  - `prettier ^3.8.1`：代码格式化（含 `.scss` 格式规范）；`rimraf`：清理。
- **测试工具**：Node 脚本直接调用 `sass`/`node-sass` 做"编译矩阵"测试（见第 7 章）；文档站用 Cypress 做 e2e。
- **文档站**：Jekyll（Ruby，见 `docs/_config.yml`），Cypress 端到端测试。
- **兼容性遗留**：`bower.json` 仍在维护，保留 Bower 安装通道；`test/` 下同时保留 dart-sass / node-sass / ruby-sass 三套编译验证目录。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `bulma.scss` | 唯一入口，仅 3 行：charset 注释 + `@use "sass"`（实际加载 `sass/_index.scss`） |
| `sass/` | 全部框架源码，73 个 SCSS 文件，按职责分 10 个子层 |
| `sass/utilities/` | 地基层：变量、函数、mixin、CSS 变量桥、继承占位符（约 1600 行） |
| `sass/themes/` | v1.0 明暗主题系统（light/dark/setup） |
| `sass/base/` | minireset 重置、generic 全局默认、animations、skeleton 骨架屏 |
| `sass/elements/` | 14 类原子元素（button、box、tag、notification、title 等） |
| `sass/form/` | 6 类表单控件（input、select、checkbox-radio、file 等） |
| `sass/components/` | 11 类复合组件（navbar 799 行、card、modal、tabs、pagination 等） |
| `sass/grid/` | 两代栅格：`columns.scss`（961 行，Flexbox 经典）+ `grid.scss`（209 行，v1.0 CSS Grid） |
| `sass/layout/` | 5 类页面结构（container、hero、section、level、media） |
| `sass/helpers/` | 13 类工具类（spacing、typography、flexbox、color、visibility 等） |
| `css/` | 构建产物：bulma.css（764KB）、bulma.min.css（678KB）及 4 个变体 |
| `versions/` | 4 个变体入口：prefixed / no-dark-mode / no-helpers / no-helpers-prefixed |
| `test/` | 三套 Sass 实现的编译矩阵测试（dart-sass/node-sass/ruby-sass）+ utils.js |
| `docs/` | Jekyll 文档站（356 个 HTML 页面）+ Cypress e2e 测试 |
| `.github/` | CI workflow、issue/PR 模板、stale bot、FUNDING |

组织方式评述：**单包单体**结构，但内部分层极其清晰——`sass/_index.scss` 用 10 行 `@forward` 固定了层序：utilities → themes → base → elements → form → components → grid → layout → skeleton → helpers（`sass/_index.scss`）。依赖方向严格单向：下层不知晓上层；每个子目录有自己的 `_index.scss`，既支持整包引入，也支持按模块单文件引入（`test/utils.js` 的 SOURCES 列表逐个单独编译验证了这一点）。

## 5. 核心架构与关键实现

### 5.1 变量体系：Sass 变量 → CSS 自定义属性的双层设计

v1.0 最核心的抽象是 `sass/utilities/css-variables.scss`（502 行）：它提供 `buildVarName`（生成 `--bulma-*` 名）、`getVar`（在 Sass 里引用 CSS 变量）、`register-vars`（把一组 Sass 变量批量写成 CSS 自定义属性）等工具。结果是每个组件呈现统一模式，以 `sass/components/card.scss` 为例：

```scss
$card-radius: 0.75rem !default;                      // 1) 局部 Sass 变量，可被 with 覆盖
$card-color: cv.getVar("text") !default;             //    默认值引用全局 CSS 变量

.#{iv.$class-prefix}card {
  @include cv.register-vars((                        // 2) 全部以 --bulma-card-* 落到 CSS
    "card-color": #{$card-color},
    "card-radius": #{$card-radius},
    ...
  ));
  // 3) 规则体内用 var() 消费，运行时可覆盖
}
```

这使组件既能编译期定制（Sass）也能运行时定制（CSS 变量），且 `$class-prefix` 变量贯穿所有类名，支撑 prefixed 构建变体。

### 5.2 颜色系统与自动派生

`sass/utilities/functions.scss`（322 行）实现 `mergeColorMaps`：用户只需提供一个基础色，框架自动用 `bulmaFindColorInvert` / `bulmaFindLightColor` / `bulmaFindDarkColor` 派生出反色、浅色、深色四个分量并合并进色板。配合 `sass/utilities/initial-variables.scss` 里基于 HSL 的色板声明（`$scheme-h: 221 !default; $scheme-s: 14% !default`），整个色彩体系可以只改 2 个变量就整体换色调。

### 5.3 明暗主题：亮度刻度驱动

`sass/themes/dark.scss` 不是手写一套暗色值，而是把所有语义变量抽象为 **HSL 亮度（lightness）刻度**：暗色主题仅重新注册亮度参数（`"scheme-main-l": 9%`、`"text-l": 71%`、hover/active 的亮度增量等），再通过 `generate-on-scheme-colors` 对每个色板色重算（`sass/themes/dark.scss`）。`themes/setup.scss` 配合 `prefers-color-scheme` 与 `[data-theme]` 选择器落盘。这使暗色主题与亮色主题共享同一套逻辑，维护成本极低。

### 5.4 Smart Grid：CSS Grid + 容器查询

`sass/grid/grid.scss`（209 行）是 v1.0 新栅格：`.fixed-grid` 声明 `container-type: inline-size`，列数变量 `--bulma-grid-column-count` 随**容器宽度**（而非视口）通过 `container-until/container-from` mixin 在 tablet/desktop/widescreen/fullhd 各断点切换，支持 `has-*-cols` 固定列数与 `has-auto-count` 自适应列数。对比旧 `columns.scss`（961 行 Flexbox 实现），新栅格代码量仅 1/4。这是仓库里最能体现"跟随 CSS 平台演进"意图的模块。

### 5.5 构建变体

`versions/` 提供 4 个入口组合出不同发行形态：`bulma-prefixed.scss`（类名加前缀）、`bulma-no-dark-mode.scss`、`bulma-no-helpers.scss` 及两者组合（`versions/*.scss`、`package.json` scripts）。同一份源码按需裁剪/改名，不靠运行时开关。

## 6. 构建与运行

来自 `package.json`（实测可用，未运行）：

```sh
npm install
npm run build-bulma     # dart-sass 编译 bulma.scss → css/bulma.css（含 source map）
npm run minify-bulma    # postcss + cssnano → css/bulma.min.css
npm run build-versions  # 编译 4 个变体
npm run deploy          # clean + build-all + minify-all，发布前全流程
npm run start           # build-bulma --watch 开发模式
npm test                # 仅 sass --watch test.scss（仓库级 test 脚本较形式化）
```

使用方式（`README.md`）：`npm install bulma` 后 `@import 'bulma/css/bulma.css'`，或 CDN（jsdelivr），或引入 `sass/` 源码自行定制编译。文档站本地运行：在 `docs/` 下 Jekyll serve（CI 配置可见 `gem install jekyll bundler` + `jekyll serve`，`.github/workflows/main.yml`）。

## 7. 测试与工程质量

- **编译矩阵测试**（本仓库最主要的自动化验证）：`test/utils.js` 定义全部模块清单，`test/dart-sass/dart-sass.js` 与 `test/node-sass/node-sass.js` 分别做三类编译——整包编译、自定义变量编译（`@use "./sass/components/navbar.sass" with ($scheme-main: red)`）、逐模块单文件编译；`test/ruby-sass/` 保留第三套实现验证。这保证了"任一子模块可独立引入"的承诺。
- **e2e 测试**：`docs/cypress/` 有 39 个测试文件，按 base/components/elements/form/grid/layout 分目录覆盖各组件的渲染行为；但 `.github/workflows/main.yml` 中 Cypress 步骤整体被注释，当前 CI 仅安装 Jekyll，**e2e 实际未在 CI 中执行**（标注：静态观察结论）。
- **CI 现状**：仓库仅 1 条 workflow（main.yml），且核心断言步骤被注释——CI 强度与项目体量明显不匹配，质量主要靠作者手工与社区 issue 反馈。
- **代码质量观察**：Sass 代码风格高度统一（prettier 管辖），命名规范严格（`is-*`/`has-*`/组件前缀），注释充分；版本发布走 CHANGELOG 手工维护，条目关联 issue 编号，可追溯性好（`CHANGELOG.md`）。

## 8. 社区与维护状态

- **活跃度**：创建于 2016-01，至今约 10 年。最近 push 2026-09-21（GitHub API），即分析时点前约 6 周仍有提交，**项目存活但节奏放缓**。
- **Release 节奏**：1.0.0（2024-03）→ 1.0.1（2024-05）→ 1.0.2（2024-07）→ 1.0.3（2024-12）→ 1.0.4（2025-04），此后 18 个月无正式 release，但 `CHANGELOG.md` 顶部已有 1.1.0 的 bug fix 条目（offset 修正），说明 main 分支在向 1.1.0 积累而未发版。
- **维护结构**：典型的**单主导者项目**——作者 Jeremy Thomas（jgthms）是 package.json 唯一 author；`BACKERS.md` 与 `.github/FUNDING.yml` 显示靠赞助维持（Prime backers $30+/月档位）。贡献者 API 前 200 个条目均满页（累计 200+，总数未确认）。
- **社区压力指标**：open issues 532，watch 569——issue 积压量相对 5 万星体量偏大，`.github/stale.yml` 的 stale bot 也在协助分流。
- **文档/生态**：官网 bulma.io（Jekyll 源码随仓库维护），awesome 列表收录，另有 bulma-start 模板与模板集合页（`docs/bulma-start.html`、`docs/bulma-templates.html`）。

## 9. 亮点与风险

**亮点**
1. 纯 CSS、零运行时的彻底性：全仓库（除测试/文档工具）无一行框架 JS，前端栈无关，这是它与 Bootstrap/Tailwind 类方案最鲜明的差异化（`README.md`、全仓库 .js 文件分布）。
2. 双层定制体系设计精良：Sass `!default` 变量 + 全量 CSS 自定义属性暴露，编译期与运行时可定制并存，且组件级变量命名空间（`--bulma-card-*`）干净（`sass/components/card.scss`、`sass/utilities/css-variables.scss`）。
3. 主题系统的亮度刻度抽象：暗色主题与亮色主题共用一套 HSL 逻辑，仅注册亮度差值，实现极省（`sass/themes/dark.scss`）。
4. 技术栈与时俱进：Dart Sass 模块系统、CSS 容器查询、`:root` 变量、prefixed 变体防冲突，1.0 是一次到位的现代化重写（`sass/grid/grid.scss`、`versions/`）。
5. "任一模块可独立引入"有测试兜底：逐文件编译矩阵保证模块化承诺不腐化（`test/utils.js`）。

**风险/局限**
1. 维护节奏显著放缓：最新 release 距今 18 个月，CI 中 e2e 断言被注释，单人项目的总线因子（bus factor）为 1。
2. 无交互组件实现：modal、dropdown、navbar 汉堡菜单等需要用户自配 JS，对新手是额外成本（这是设计取舍而非缺陷，但构成迁移门槛）。
3. 构建产物体积可观：`css/bulma.css` 764KB（min 678KB），大量 CSS 变量注册占了体积，按需子模块引入虽可行但无官方 tree-shaking 方案（`css/` 实测大小）。
4. 版本跨度断层：0.9.x → 1.0.0 是破坏性重写（变量体系全换），存量项目的升级成本高，社区存在版本碎片化风险（`CHANGELOG.md` 中 1.0.0 前后条目对比）。

## 10. 总结评价

Bulma 是 CSS 框架时代一个设计上近乎教科书级的纯样式层实现：以极小的依赖面（运行时为零）换取了完整的组件体系与双重定制能力，1.0 的 CSS 变量/容器查询现代化证明了作者对 CSS 平台演进的敏锐把握。它的星数（5 万+）建立在"比 Bootstrap 轻、比手写 CSS 快"的清晰生态位上。当下它的主要问题不在代码而在维护：单人驱动、发版停滞、CI 弱化，选型时需把"遇到 bug 可能要自己 patch"纳入预期。**适合**：需要无 JS 样式层的项目、想学习 Sass 架构分层与 CSS 变量体系设计的开发者（`sass/utilities/css-variables.scss` 与 `sass/themes/` 值得精读）；**慎选**：需要开箱即用交互组件、强生命周期保障的大型团队项目。
