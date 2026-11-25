# iamkun/dayjs 项目分析

> 星数 47800（快照 2026-10-24，gh api 实测 48654 @ 2026-11-04）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：一个仅 2kB（gzip）的不可变日期时间库，以与 Moment.js 几乎一致的 API 提供解析、校验、操作与格式化能力，是 Moment.js 停维后的主流轻量替代品。

| 项目 | 事实 |
| --- | --- |
| 名称 | iamkun/dayjs（npm 包 `dayjs`） |
| 星数 | 47800（快照）；gh api 实测 48654（2026-11-04） |
| 主语言 | JavaScript（ES2015+，无 TypeScript 源码但自带完整 .d.ts） |
| 许可证 | MIT（`LICENSE`、`package.json`） |
| 创建时间 | 2018-04-10（gh api `created_at`） |
| 最近活跃 | 最近 push 2026-09-15（gh api `pushed_at`）；最新版 1.11.22 发布于 2026-08-16（`CHANGELOG.md`） |
| 默认分支 | dev（gh api；master 用于发布） |
| 代码规模 | 核心 3 文件共 552 行；37 个插件、143 个语言包；测试 93 个文件约 9400 行 |

## 2. 项目是什么，解决什么问题

Day.js 解决的是 Moment.js 的核心痛点——体积与不可变性。Moment.js 全量引入约 70kB+ 且对象可变（mutable）易引发副作用 bug；Day.js 压缩后约 2kB，所有操作返回新实例（README 明确列出 "Immutable"、"2kb mini library"、"Familiar Moment.js API & patterns" 三大卖点，见 `README.md`）。API 刻意与 Moment 对齐：`dayjs().startOf('month').add(1, 'day').set('year', 2018).format('YYYY-MM-DD HH:mm:ss')`（`README.md` 示例），官方口号是"会用 Moment 就会用 Day.js"，目标是让存量 Moment 项目近乎零成本迁移。

设计上 Day.js 把"核心最小化、功能插件化"做到极致：核心只有解析/取值/设值/加减/格式化/对比等 Moment 最高频的能力（`src/index.js`），时区、Duration、自定义解析、相对时间等全部拆成按需引入的插件（`src/plugin/`），国际化语言包也全部按文件拆分（`src/locale/`，143 个）。这与 Moment 的"全家桶"形成直接对照——moment-timezone 之类的能力在 Day.js 里是 `import timezonePlugin from 'dayjs/plugin/timezone'`。

一个重要的工程事实是：Day.js 的正确性锚定在 Moment.js 上。仓库直接把 `moment`（2.29.2）和 `moment-timezone` 作为 devDependencies（`package.json`），测试目录中的 `test/comparison.test.js` 是从 Moment 测试套件移植的对照用例，保证输出与 Moment 逐位一致；`test/plugin/timezone.test.js` 也以 moment-timezone 为参照。这种"以竞品为黄金标准"的测试策略是其敢宣称 API 兼容的底气。

解析性能与跨浏览器一致性也有取舍：对形如 `YYYY-MM-DD HH:mm:ss` 的字符串，Day.js 不走 `new Date(string)`（各引擎行为不一），而是用自写正则 `REGEX_PARSE` 拆出年月日后构造 `Date`（`src/constant.js`、`src/index.js` 的 `parseDate()`），代价是不支持 Moment 那样丰富的自然语言解析（需另挂 customParseFormat 插件）。

生态位上，Day.js 是 Vue/element-plus、Ant Design Vue 等大量 UI 库的默认日期依赖，也是"Moment 停维公告"后社区迁移清单的首推项之一。它不追求 calendar 库的完整功能，而是死守"解析-计算-展示"这一条最常用链路。

## 3. 技术栈与依赖

- 语言：纯 JavaScript（ES Module 源码，Babel 转译；`babel.config.js`），类型由手写 `types/index.d.ts`（429 行）+ 各插件/语言包声明提供（`types/` 目录）。
- 运行时依赖：**零依赖**（`package.json` 无 `dependencies` 字段），核心只用原生 `Date` 与 `Intl.DateTimeFormat`（timezone 插件，`src/plugin/timezone/index.js`）。
- 构建链：Rollup 2 + rollup-plugin-babel + rollup-plugin-terser（`build/rollup.config.js`），输出 UMD；ESM 版本由 Babel 直接转译生成（`build/esm.js`、`package.json` 的 `babel` 脚本）。
- 测试链：Jest 22 + babel-jest + mockdate（`test/comparison.test.js` 头部使用）；跨浏览器用 Karma + karma-sauce-launcher 跑 SauceLabs（`karma.sauce.conf.js`）。
- 质量工具：ESLint（airbnb-base 规则，`package.json` devDependencies）、Prettier（仅 docs）、size-limit（体积门禁）、semantic-release + @semantic-release/changelog（`.releaserc`）。
- 特殊 dev 依赖：`moment` 2.29.2 与 `moment-timezone` 0.5.31——仅用于对照测试，不进入产物（`package.json`）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `src/index.js` | 核心：`dayjs` 工厂函数 + `Dayjs` 类（468 行，全部核心能力） |
| `src/constant.js` | 常量：时间换算、单位名、解析/格式化正则（`REGEX_PARSE`、`REGEX_FORMAT`） |
| `src/utils.js` | 内部工具：补零、时区偏移串、月份差、单位别名归一化（单字母缩写映射） |
| `src/plugin/` | 37 个插件，每个独立目录 + `index.js`，可独立引入 |
| `src/locale/` | 143 个语言包（如 `en.js`、`zh-cn.js`），纯数据对象 |
| `build/` | Rollup 构建脚本：核心/插件/语言包批量产出 UMD + 生成 `locale.json` 索引 |
| `types/` | 手写 TypeScript 声明（`index.d.ts`、`plugin/`、`locale/`） |
| `test/` | Jest 测试：核心域拆分（parse/get-set/manipulate/display/query 等）+ `plugin/` + `locale/` + `issues/` 回归用例 |
| `.github/workflows/` | CI：`check.yml`/`lint-test.yml`（lint+测试）、`release.yml`（发布） |
| `patches/` | 依赖补丁目录（`package.json` 未见 patch-package 集成，具体用途未确认） |
| `docs/` | 多语言文档站与 README 译文（zh-cn/ja/ko/ru/tr 等 10 种） |

组织方式评述：单包（单 npm 包）+ 严格"核心/插件/语言包"三分。构建时核心打成一个 `dayjs.min.js`，37 个插件与 143 个语言包各产出独立 UMD 文件（`build/index.js` 逐个顺序构建以省内存），用户按需 `<script>` 或 `import`。这是典型的"微内核 + 官方插件注册表"模式，插件与语言包是事实上的多包发布形态，但代码仍在一个仓库内统一演进。

## 5. 核心架构与关键实现

### 5.1 不可变核心：Dayjs 类与克隆式写操作（`src/index.js`）

实例内只持有一个原生 Date（`this.$d`）加一组预解析字段（`$y/$M/$D/$W/$H/$m/$s/$ms`，`init()` 一次性解出，避免每次取值重复调 getter，`src/index.js` 的 `init()`）。不可变性靠"先克隆再改"实现：

```js
// src/index.js
$set(units, int) { // private set
  ...
  if (unit === C.M || unit === C.Y) {
    // clone is for badMutable plugin
    const date = this.clone().set(C.DATE, 1)
    date.$d[name](arg)
    date.init()
    this.$d = date.set(C.DATE, Math.min(this.$D, date.daysInMonth())).$d
  } else if (name) this.$d[name](arg)
  this.init()
  return this
}
set(string, int) {
  return this.clone().$set(string, int)   // 对外入口永远返回新实例
}
```

所有公开操作（`add/subtract/set/locale`）都先 `clone()`（经 `Utils.w` 包装携带原实例的 locale/utc 上下文），`add()` 内部也统一走 `Utils.w(nextTimeStamp, this)` 生成新实例（`src/index.js`）。月份加减先钳到 1 号再回设 `Math.min($D, daysInMonth())`，避免"1月31日加一个月变3月3日"的经典 bug。有趣的反转设计：`badMutable` 插件（`src/plugin/badMutable/index.js`）反向覆写 `set/startOf/add` 为原地修改，专门服务想把 Moment 代码直接换成 dayjs 且依赖可变语义的存量项目——不可变是默认，可变反而是插件。

### 5.2 插件机制：一次安装、原型增强、责任链式包装（`src/index.js` + `src/plugin/*`)

插件就是一个函数 `(option, DayjsClass, dayjsFactory)`，`dayjs.extend()` 用 `$i` 标记保证只安装一次：

```js
// src/index.js
dayjs.extend = (plugin, option) => {
  if (!plugin.$i) { // install plugin only once
    plugin(option, Dayjs, dayjs)
    plugin.$i = true
  }
  return dayjs
}
```

插件拿到 `Dayjs.prototype` 后采用"保存旧方法 → 覆写 → 时机调用旧方法"的装饰器模式扩展生命周期。utc 插件是范本：覆写 `parse`/`init` 设置 `$u` 标志并把 getter 切换成 `getUTC*`（`src/plugin/utc/index.js` 的 `oldParse`/`oldInit`），并注册 `dayjs.utc()`、`proto.utc()`、`proto.local()`。这种实现比继承更省字节，也允许多个插件叠加（timezone 插件就同时依赖 utc 插件的 `$u`/`$offset` 通道，见 `src/index.js` 的 `wrapper` 透传 `$offset`）。副作用是插件间通过原型全局生效、存在隐式耦合，插件覆盖同名方法时安装顺序即行为。

### 5.3 体积控制手段（`package.json` + `src/index.js` + `build/`）

2kB 不是偶然，仓库有系统性手段：① `size-limit` 门禁写死 `"limit": "2.99 KB"` 于 `package.json`，`npm run build` 后立即 `npm run size` 超限即失败；② 单字母内部压缩——utils 导出 `{s: padStart, z: padZoneStr, m: monthDiff, a: absFloor, p: prettyUnit, u: isUndefined}`，常量用单字母（`C.M`/`C.D`），实例字段 `$d/$y/$M`（`src/utils.js`、`src/constant.js`）；③ getter/setter 用数据驱动批量生成，8 对字段一行循环生成 8 个方法，不写 8 遍函数体（`src/index.js` 底部 `forEach((g) => { proto[g[1]] = function (input) { return this.$g(input, g[0], g[1]) } })`）；④ `format()` 用单条正则 `REGEX_FORMAT` 一次 `replace` 完成 token 替换，switch 返回 null 时回退到时区串（`src/index.js`）；⑤ 语言包极简——`en.js` 只用下划线分隔字符串 `split('_')` 生成数组，注释兼任构建期元数据（`build/index.js` 用正则 `// (.*) \[` 从注释提取语言名生成 `locale.json`，`src/locale/en.js`）。

### 5.4 时区与解析的两个代表性实现

`timezone` 插件不自带时区数据库，而是借宿主 `Intl.DateTimeFormat.formatToParts` 计算指定时区偏移，并用 `dtfCache` 缓存格式化器（源码注释：Intl.DateTimeFormat 是"非常慢的方法"，`src/plugin/timezone/index.js`）；相比 moment-timezone 需要打包 tzdata，Day.js 方案零数据、体积恒定，代价是依赖宿主 ICU 数据（老浏览器/精简 ICU 环境表现可能不一致）。字符串解析则用自写正则 `REGEX_PARSE = /^(\d{4})[-/]?(\d{1,2})?...$/`（`src/constant.js`）直接拆数字构造 `Date`，绕开各引擎 `Date.parse` 的行为差异；更复杂的模板解析（`dayjs("2018", "YY")`）交给 `customParseFormat` 插件，其内部用一张 token→正则映射表逐段消费（`src/plugin/customParseFormat/index.js`）。

## 6. 构建与运行

来自 `package.json` scripts（静态阅读，未实际运行）：

```bash
npm install            # 安装依赖（含 moment 等测试用）
npm run lint           # eslint src/* test/* build/*
npm test               # 多时区跑测 + jest --coverage，全局行覆盖 100% 门禁
npm run build          # rollup 产出核心/插件/语言包 UMD + size-limit 体积校验
npm run babel          # Babel 转译 src → esm/ 并生成 esm 入口
npm run test:sauce     # Karma + SauceLabs 跨浏览器测试
```

`npm test` 会依次以 `TZ=Pacific/Auckland`、`Europe/London`、`America/Whitehorse`（本机默认）、`Europe/Paris`、`America/New_York` 等时区各跑一遍时区相关用例，最终再带 `--coverageThreshold="{\"global\":{\"lines\":100}}"` 跑全量——行覆盖率不足 100% 直接失败（`package.json`）。

## 7. 测试与工程质量

- 规模：93 个 `*.test.js` 共约 9385 行；核心域测试按能力拆分（`test/parse.test.js`、`get-set.test.js`、`manipulate.test.js`、`display.test.js`、`query.test.js` 等），每个插件和主流语言包在 `test/plugin/`、`test/locale/` 下有同名用例，`test/issues/` 收集用户报 bug 的回归用例。
- 风格：单元测试为主，大量用例直接从 Moment.js 测试套件移植（`test/comparison.test.js` 与 Moment 逐断言对齐，`mockdate` 固定当前时间保证可重复）；`test/timezone.test.js` 专门跨时区验证。
- 门禁：行覆盖率 100%（`package.json` jest 配置）、size-limit 2.99KB、pre-commit 跑 lint（`package.json` `pre-commit` 字段）。
- CI：PR 到 dev 触发 `check.yml` → 复用 `lint-test.yml`（lint + test + Codecov 上报）；push 到 master 触发 `release.yml`：先跑同一套 lint/test，再 build + semantic-release 自动发版，并 `npm audit signatures` 校验 npm provenance（`.github/workflows/lint-test.yml`、`release.yml`）。
- 观感：对一个 500 行核心的库做到 9400 行测试 + 100% 行覆盖 + 多时区矩阵，测试投入远超代码本身，是其作为基础设施库最突出的工程质量特征。

## 8. 社区与维护状态

- 活跃度：最近 push 2026-09-15，最新版 1.11.22 于 2026-08-16 发布（`CHANGELOG.md`），持续维护中但节奏偏慢——2026 年三个版本（1.11.20/21/22，间隔约 2-5 个月）且全部是 bug fix / locale 更新，无新特性。
- 发布机制：semantic-release 全自动（`.releaserc`，仅 master 分支发布，自动生成 CHANGELOG），个人维护者主导（作者 iamkun），贡献者规模约 338 人（gh api contributors 分页推算，per_page=1 共 338 页）。
- 社区负载：open issues 1455（gh api，2026-11-04），对单人维护的项目偏多，issue 积压明显；README 提供 10 种语言译文、官方文档站 day.js.org（`README.md` 顶部链接），并有付费赞助渠道（`.github/FUNDING.yml` 存在）。
- 分支模型：开发在 dev，发布在 master（gh api `default_branch: dev` + `.releaserc` `branches: ["master"]`）。

## 9. 亮点与风险

亮点：
1. 极致的体积工程：零依赖核心 552 行、size-limit 2.99KB 门禁进 CI、单字母内部命名 + 数据驱动方法生成，把"2kB"从口号变成被测试保护的事实（`package.json`、`src/index.js`、`src/utils.js`）。
2. 以竞品为黄金标准的测试策略：把 Moment（及其测试套件）作为 devDependency 逐断言对照（`test/comparison.test.js`），API 兼容性可验证而非口头承诺；辅以 100% 行覆盖与 5+ 时区矩阵。
3. 干净的微内核 + 插件架构：`extend` 一次安装、原型装饰器式增强、utc/timezone 插件间的 `$u/$offset` 通道设计，是学习"如何用最小机制做可扩展库"的优秀范本（`src/index.js`、`src/plugin/utc/index.js`）。
4. Moment 迁移友好：API 同构之外还提供 `isMoment`、`badMutable` 等兼容插件（`src/plugin/badMutable/index.js`），存量 Moment 代码可低成本切换。
5. 发布与国际化基建成熟：semantic-release 自动发版 + npm provenance 校验、143 语言包、10 语种文档（`.github/workflows/release.yml`、`src/locale/`、`docs/`）。

风险/局限：
1. 单人维护 + 1455 个 open issue，问题响应与修复周期长（2026 年仅 3 个纯 bugfix 版本，`CHANGELOG.md`）。
2. 时区能力依赖宿主 Intl/ICU：timezone 插件不内置 tzdata，极端环境（老引擎、精简 ICU）下历史时区规则/DST 准确性不如 moment-timezone（`src/plugin/timezone/index.js` 设计取舍）。
3. 非标准字符串解析覆盖窄：默认仅支持 `REGEX_PARSE` 匹配的数字格式，Moment 式自然语言/多模板解析需额外插件且表达力有限（`src/constant.js`、`src/plugin/customParseFormat/index.js`）。
4. 生态演进压力：Temporal（TC39）标准化落地后将原生覆盖大部分场景，长期看这类库的存在价值会被逐步侵蚀（外部趋势判断，非仓库内证据）。

## 10. 总结评价

Day.js 是"约束驱动设计"的教科书案例：用 552 行核心、零依赖、CI 强制 2.99KB 体积门禁和 100% 行覆盖，复刻了 Moment.js 的 API 表面并修正了其体积与可变性两大顽疾；微内核 + 37 官方插件 + 143 语言包的组织方式让它既轻又能按需长大。它当前处于成熟维护期——不再加特性、只修 bug，单人主导但基建自动化程度高。适合把它当作生产环境日期处理的首选轻量方案（尤其从 Moment 迁移的项目），也极适合想学习"小而美库如何做体积控制、插件机制与兼容性测试"的开发者通读 `src/index.js` 与 `test/comparison.test.js`；若需要重型时区/日历能力，则应评估 luxon、date-fns-tz 或未来的 Temporal。
