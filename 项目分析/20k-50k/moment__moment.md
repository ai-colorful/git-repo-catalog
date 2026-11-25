# moment/moment 项目分析

> 星数 48200（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：JavaScript 生态最经典的日期处理库（解析/校验/操纵/格式化"四合一"），曾长期是事实标准，现已官方宣布进入维护模式（legacy project）。

| 项目 | 内容 |
|---|---|
| 名称 | moment/moment（npm 包 `moment`） |
| 星数 | 48200（GitHub API 实测 47,878，2026-11-05） |
| 语言 | JavaScript（ES Modules 源码 + UMD 产物），附 TypeScript 类型定义 |
| 许可证 | MIT（`LICENSE`） |
| 版本 | 2.31.0（`package.json` / `CHANGELOG.md`，2026-09-14 发布） |
| 最近活跃 | 最近 push 2026-10-08（GitHub API）；最新 release 2.31.0（2026-09-15） |
| 创建时间 | 2011-03-01（GitHub API） |
| 归属 | OpenJS Foundation（Linux Foundation 旗下），`README.md` |
| 默认分支 | develop（GitHub API） |
| 贡献者 | 约 390+（contributors API 分页推断，未精确统计） |

## 2. 项目是什么，解决什么问题

Moment.js 诞生于 2011 年，解决的是当年 JavaScript 原生 `Date` 对象几乎不可用的痛点：解析非标准字符串、格式化输出、跨时区换算、月份加减的边界行为（如 1 月 31 日加一个月）、相对时间（"3 天前"）与国际化文案。它把这一切收敛成一个链式 API：`moment().add(7, 'days').format('YYYY-MM-DD')`（`README.md`，`src/lib/moment/add-subtract.js`）。

库的能力面覆盖四块（`src/lib/` 目录划分即对应）：解析（`lib/create/`、`lib/parse/`，支持 ISO 8601、RFC 2822、自定义格式串）、取值/设值（`lib/units/`，年月日时分秒周季度纪元全维度）、格式化与相对时间（`lib/format/`、`lib/duration/humanize.js`）、国际化（`locale/` 下 139 个 locale 文件）。时区支持则由姊妹项目 Moment Timezone 提供（本仓库 2.31.0 还专门为其加了内部钩子 `date-default`，见 `CHANGELOG.md` 2.31.0 "New features"）。

必须强调项目现状：`README.md` 开头明确写着 "⚠️ Moment.js is a legacy project, now in **maintenance mode**. New features are not being accepted."。这是 2020 年官方状态公告（momentjs.com/news）落地到仓库的长期声明——不再接受新功能，仅做 bug 修复、安全修复与 locale 维护。2026 年的活跃记录（2.30.1、2.31.0 两个 release）证实它没有死，只是"冻结式活着"。

维护模式并未让它边缘化：`CHANGELOG.md` 2.31.0（2026-09）仍包含一次安全修复（CVE-2026-17495 / GHSA-4p3w-j4w9-5jqw）、一批原型链加固（如修复 `locale('__proto__')` 污染全局 locale 的 #6442、阻止对象原型属性被当作 format token 的 #6376）和新语言（Pashto、Amharic-Ethiopia）。作为 OpenJS Foundation 项目，它有正式的 `SECURITY.md`（走 OpenJS CNA 漏洞上报流程）与 `THREAT_MODEL.md`，这种"养老项目"级别的治理完整度在同类仓库中少见。

在生态位上，Moment 已把"更现代的替代品"写进了自己官网的推荐名单（dayjs、Luxon、date-fns、js-Joda，见 momentjs.com 官网公告；本批前面 dayjs 分析中亦确认其 API 兼容 Moment 的定位）。今天的 Moment 更像"存量基础设施"：npm 周下载量依然巨大（README 徽章项），大量老项目依赖，但新项目首选已是替代品。

## 3. 技术栈与依赖

- **语言**：纯 JavaScript（ES2020+ 模块源码，无 TypeScript 源码）。`src/lib/**/*.js` 共 109 个文件、约 6,585 行。
- **运行时零依赖**：`package.json` 中没有任何 `dependencies`，全部能力基于原生 `Date` 与正则。`engines.node: "*"`（任何 Node 版本可用），另设 `devEngines` 约束开发环境 Node 22/24/26+。
- **构建**：pnpm（`packageManager: pnpm@11.22.0`）+ Node 脚本（`scripts/build.js`）+ rollup（打包 ESM 产物 `dist/moment.js`）+ uglify-js（压缩 `min/`）。**注意：历史上的 Grunt 构建链已被整体移除**——仓库内不存在 Gruntfile（`ls Gruntfile*` 无结果），构建已迁到自研 node 脚本，这是对"Moment 用 Grunt"旧印象的重要修正。
- **测试**：QUnit（`qunit ^2.26.0`），自定义 runner `scripts/test.js`；覆盖率用 nyc + Coveralls。
- **代码质量**：ESLint 10、Prettier、markdownlint-cli2。
- **类型**：手写 `moment.d.ts`（798 行）+ `ts3.1-typings/`（965 行，针对 TS >= 3.1 的 `typesVersions` 分发）；类型测试横跨 TypeScript 1.8 → 7 共 9 个大版本（`package.json` 中 typescript1~typescript7 多版本 devDependencies + `typing-tests/legacy|modern` 两套工程）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `src/lib/` | 核心源码，ES Modules，按域分子目录（109 文件 / 6585 行） |
| `src/locale/` | 139 个 locale 的 ESM 源码 |
| `locale/` | 同样 139 个 locale 的 UMD 打包产物（发布形态） |
| `moment.js` | 仓库根的 UMD 单文件 bundle（5917 行），npm `main` 入口 |
| `dist/`、`min/` | ESM 产物与压缩产物（moment.min.js、moment-with-locales.js 等 8 个文件） |
| `moment.d.ts`、`ts3.1-typings/` | TypeScript 类型定义 |
| `typing-tests/` | 9 个 TS 版本的类型测试工程（legacy/modern 两套） |
| `scripts/` | 15 个 Node 构建/测试/发布脚本（build、test、minify、benchmark、release-* 等） |
| `benchmarks/` | 16 个基准测试（parse、add、format、clone 等热点路径） |
| `.github/workflows/` | 7 个 CI workflow（ci、coverage、fossa、pull-request、release、runtime-compatibility、timezones） |
| `meteor/`、`bower.json`、`component.json`、`composer.json`、`ender.js`、`dojo.profile.js` | 遗留多平台分发适配（Meteor 打包内含指向根 `moment.js` 的符号链接） |
| `SECURITY.md`、`THREAT_MODEL.md`、`RELEASING.md`、`AGENTS.md` | 治理文档（含 AI 编码助手政策链接） |

组织方式评述：典型的"单包多形态"仓库——一套 ESM 小模块源码，经自研脚本拼接/rollup 出 UMD、ESM、min、with-locales 四类产物，并维护 bower/component/composer/spm/meteor 等历史包管理器的适配元数据（全部集中在 `package.json` 末尾）。源码风格是"一文件一职责"的函数式模块：`src/lib/utils/` 下 30 个文件多数只有 3-20 行（如 `abs-floor.js` 8 行、`is-string.js` 3 行），粒度极细但总量可控。

## 5. 核心架构与关键实现

### 5.1 可变对象核心：Moment 构造器与原型

Moment 实例本质是一个包着原生 `Date` 的对象，内部字段全部以下划线前缀存储（`_d` 时间值、`_i` 输入、`_f` 格式、`_l` locale、`_pf` 解析标志、`_isUTC` 等），见 `src/lib/moment/constructor.js`：

```js
export function Moment(config) {
    copyConfig(this, config);
    this._d = new Date(config._d != null ? config._d.getTime() : NaN);
    if (!this.isValid()) {
        this._d = new Date(NaN);
    }
    if (updateInProgress === false) {
        updateInProgress = true;
        hooks.updateOffset(this);
        updateInProgress = false;
    }
}
```

可变（mutable）设计是最重要的架构事实，也是它被 dayjs 等不可变库取代的根因之一。证据遍布核心路径：`src/lib/moment/add-subtract.js` 的 `addSubtract` 直接 `mom._d.setTime(mom._d.valueOf() + milliseconds * isAdding)` 原地改写后 `return this`（第 53、60-61 行）；`src/lib/moment/get-set.js` 的 `makeGetSet` 同样 `set(this, unit, value); hooks.updateOffset(this); return this;`。链式调用串的是同一个对象，`m.add(1,'day')` 会改变 `m` 本身——老用户踩过的经典坑。克隆需显式 `.clone()`（`copyConfig` + 插件字段拷贝，`constructor.js` 中 `momentProperties` 机制专门为插件扩展克隆语义）。

### 5.2 格式化引擎：token 注册表 + 编译缓存

`src/lib/format/format.js`（107 行）是 Moment API 表面积的集中体现。每个格式 token（YYYY、MM、Do……）通过 `addFormatToken` 注册成一个原型方法；`makeFormatFunction` 把格式串解析成"函数/字面量混合数组"，即把格式串**编译成可重复执行的闭包**，再按 `'$' + format` 缓存在 `formatFunctions` 对象里——同一个格式串第二次使用零解析成本（第 81-86 行）。`expandFormat` 负责把 locale 的 `LT/LL/LLL` 等本地化长格式展开成基础 token（最多递归 5 层，第 89-107 行）。解析侧对称：`src/lib/parse/token.js` 注册解析正则、`src/lib/parse/regex.js` 生成匹配器，`src/lib/create/from-string-and-format.js`（135 行）负责严格/宽松模式与年月日回填。

### 5.3 Locale 体系：全局单例 + 139 语言包

国际化是 Moment 区别于同类库的最大资产。139 个 locale（`src/locale/` 与发布目录 `locale/` 各一份）以统一 schema 描述：月/周名数组、`longDateFormat`（L/LL/LLL 系列本地化格式）、相对时间模板与复数规则、序数词函数、上下午划分（meridiem）。以 `locale/zh-cn.js` 为例，含 `meridiemParse: /凌晨|早上|上午|中午|下午|晚上/` 和按时段换算小时的 `meridiemHour` 函数——这类逻辑远超"翻译字符串"，是真正的本地化知识库。体系实现集中在 `src/lib/locale/`（14 个文件）：`locale.js` 维护**进程级全局 locale 单例**（`getSetGlobalLocale`），`defineLocale/updateLocale` 提供注册与覆写，子 locale 通过继承父级配置实现（`base-config.js` + `set.js`）。全局单例设计在多租户/SSR 场景是著名缺陷（2.31.0 修的 `locale('__proto__')` 污染全局问题正是这一全局状态的副产物，`CHANGELOG.md` #6442）。

### 5.4 units 抽象与 duration

`src/lib/units/` 把 17 种时间单位（年、季度、月、周、ISO 周、星期、纪元、时区偏移……）统一成 get/set/add/diff 的归一化协议：`aliases.js` 做 `date/DAY` → `day` 的别名归一，`constants.js` 集中换算常量（MS_PER_HOUR 等），`priorities.js` 定义对象入参的优先级排序。`src/lib/moment/diff.js`（79 行）实现两个 moment 的差值计算，按月份天数不均等场景做了月份对齐调整。`src/lib/duration/`（15 个文件）实现时长对象与 `humanize`（`humanize.js` 125 行，带可配置的取整与阈值 `relativeTimeRounding/relativeTimeThreshold`）。

## 6. 构建与运行

来自 `package.json` scripts 与 `RELEASING.md` 的实际命令：

```bash
pnpm install              # pnpm@11.22.0，Node 22/24/26+（devEngines）
pnpm build                # node scripts/build.js：从 src 生成 moment.js/dist/min 全套产物
pnpm build:custom         # 按需打包指定 locale 子集
pnpm test                 # node scripts/test.js：QUnit 单测
pnpm test:coverage        # nyc 覆盖率 + lcov 报告（CI 上报 Coveralls）
pnpm test:typescript      # TS 1.8/2/3/3.1/4/5/6/7 九套类型测试并行
pnpm lint                 # eslint + markdownlint + months-parse 校验 + prettier --check
pnpm benchmark            # node scripts/benchmark.js 性能基准
pnpm release              # lint → test → build → 更新索引/component → minify
```

产物形态（npm `files` 字段）：根 `moment.js`（UMD main）、`dist/moment.js`（ESM `jsnext:main`）、`min/` 下含 locales 的合并包与压缩包、`locale/*.js` 逐语言包。

## 7. 测试与工程质量

- **规模**：`src/test/` 共 202 个测试文件。核心单测 `src/test/moment/` 53 个文件约 23,400 行（zones.js 一项 822 行，覆盖 DST、时区切换等边角）；locale 测试 `src/test/locale/` 140 个文件约 67,600 行——**约四分之三的测试代码在验证 139 个语言的正确性**，这是 Moment 最重的质量资产。框架为 QUnit（`src/test/qunit.js`），配自定义 reporter（`scripts/qunit-reporter.js`）。
- **CI**（`.github/workflows/`）：`ci.yml` 在 Node `lts/*`、`lts/-1`、`latest` 三档矩阵跑构建测试；`coverage.yml` 上报 Coveralls；`runtime-compatibility.yml` 做运行时兼容检查；`timezones.yml` 单独校验时区数据；`release.yml` 实现**可复现发布**——构建 `pnpm pack` 后记录 sha256 并二次构建校验 checksum 一致（"Verify reproducible package" 步骤），这种供应链加固在 2.31.0 时代已就位。
- **类型测试的独特设计**：用 9 个并行安装的 TypeScript 版本（1.8.10 到 7.x，`package.json` devDependencies `typescript1`~`typescript7`）分别编译 `typing-tests/legacy|modern`，保证类型定义对十几年跨度内的用户代码都兼容——维护模式的"不破坏存量"承诺落到了工程上。
- **质量观察**：源码风格统一（Prettier 全仓格式化）、deprecation 体系完善（`src/lib/utils/deprecate.js`，警告带 guide 链接与堆栈，见 CHANGELOG #6450）；`benchmarks/` 持续维护热点路径性能。无 TODO 堆积，代码注释克制而有效。

## 8. 社区与维护状态

- **官方定位**：维护模式（maintenance mode），不接受新功能——`README.md` 第 12-14 行的显式警告 + momentjs.com/news 状态公告。
- **实际活跃度**：并未停摆。GitHub API 显示最近 push 2026-10-08；release 节奏 2023-12（2.30.1）→ 2026-07（2.30.1 后时代）→ 2026-09（2.31.0），单次 release 仍整合 20+ PR（`CHANGELOG.md`）。
- **维护重心**：从 2.31.0 变更单可见——安全修复、原型链加固、locale 修正（pt-br/uk/hu/de/sw/uz/pl 等 14 项）、为 Moment Timezone 提供内部钩子。与 2011-2019 年"功能狂奔"期形成鲜明对比。
- **治理**：OpenJS Foundation 正式项目（`README.md`），有 CNA 级安全上报通道（`SECURITY.md`：6 个工作日确认 SLA，超期升级 OpenJS CNA）、威胁模型文档（`THREAT_MODEL.md`，明确"进程级全局状态是设计决定，相关输入路由问题归应用侧"）、`RELEASING.md` 发布手册、行为准则与商标政策链接。
- **贡献者**：约 390+ 位（contributors API 392 页推断），历史核心作者 Tim Wood（creator）、Iskren Chernev（长期维护者，签名在 `moment.js` 头部注释）。

## 9. 亮点与风险

**亮点**

1. **极致的测试与兼容工程**：9 万+ 行测试（其中 locale 测试占 3/4）、TS 1.8→7 全版本类型兼容矩阵、三档 Node LTS CI、可复现构建——维护模式项目的"零回归"承诺做成了工程体系（`src/test/`、`.github/workflows/`）。
2. **全球本地化知识库**：139 个 locale 不只是翻译，含历法、序数、复数、上下午划分等深度规则（`src/locale/`），短时间内无可替代。
3. **清晰解耦的模块化源码**：109 个单职责小文件（`src/lib/`），函数式组织，读源码成本极低，是学习"格式串编译 + token 注册"模式的优秀范本。
4. **零运行时依赖**：纯 JS + 原生 Date 实现，产物多样（UMD/ESM/min/with-locales），部署面极广。
5. **成熟的治理与安全响应**：OpenJS Foundation 治理、CNA 通道、威胁模型、2026 年仍在响应 CVE 与原型链加固。

**风险/局限**

1. **可变 API 设计**：`add/subtract` 等原地修改实例（`src/lib/moment/add-subtract.js`），易引发共享状态 bug，是新库全面转向 immutable 的直接动因。
2. **全局 locale 单例 + 原型扩展**：进程级全局状态（`src/lib/locale/locale.js`）与 `moment.fn` 原型插件机制在 SSR/多实例场景天然不友好，也是近期多个安全修复的根源区（CHANGELOG #6442、#6376）。
3. **体积与性能**：核心 6.6k 行 + with-locales 包体积远大于 dayjs（约 2KB API 兼容层）；token 注册与全局缓存的启动成本在冷启动场景偏高（`benchmarks/` 存在正是为此持续监测）。
4. **维护模式即天花板**：不收新功能、Temporal（JS 标准日期 API）落地后长期生态位将进一步收缩；存量依赖越多，安全维护压力越集中于少数维护者。

## 10. 总结评价

Moment.js 是 JavaScript 日期处理的开山之作与十年事实标准，其价值已从"工具"沉淀为"基础设施 + 教科书"：源码是理解日期库设计（token 编译、单位归一化、locale 继承、可变链式 API）的最佳教材，139 个 locale 是无法快速复制的社区资产，而其维护模式下的工程体系（全版本 TS 兼容测试、可复现发布、CNA 安全通道）则示范了 legacy 项目如何体面地"维持"。作为依赖它仍在服务海量存量项目，作为新选型它已被 dayjs（immutable、API 兼容、2KB）、date-fns（函数式、tree-shaking）、Luxon（官方推荐继任者）与未来的 Temporal 取代。**适合谁用**：需要稳定处理非标准日期格式/重度本地化且不愿重构的存量项目、依赖 Moment Timezone 的时区重度用户。**适合谁读源码**：想造日期库或理解 i18n 实现的工程师——`src/lib/format/format.js` 与 `src/lib/locale/` 这条线尤其值得精读；也适合研究"维护模式如何运作"的开源维护者。
