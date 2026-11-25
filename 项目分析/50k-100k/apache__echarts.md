# apache/echarts 项目分析

> 星数 67575（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：Apache ECharts 是 Apache 软件基金会顶级项目，一款声明式配置、支持 Canvas/SVG 双渲染、覆盖 25 种图表类型的开源数据可视化图表库。

| 项 | 值 |
|---|---|
| 名称 | apache/echarts |
| 版本（HEAD） | 6.1.0（`package.json`、`src/core/echarts.ts` 中 `export const version = '6.1.0'`） |
| 星数 | 67575（快照）；forks 19928（gh api 2026-10-31） |
| 语言 | TypeScript（编译产物含 JS/ESM/类型声明） |
| 许可证 | Apache-2.0（LICENSE、NOTICE，符合 ASF 规范） |
| 最近活跃 | 最近 push 2026-10-13；最新 release 6.1.0 发布于 2026-05-19（gh api） |
| 运行时依赖 | 仅 2 个：`tslib`、`zrender@6.1.0`（package.json） |

## 2. 项目是什么，解决什么问题

ECharts 解决的是"在浏览器里用声明式配置快速得到可交互、可定制、大数据量下仍流畅的可视化图表"这一问题。用户只需 `echarts.init(dom)` + `setOption({...})` 一份 JSON 配置，库内部负责数据建模、坐标系布局、视觉映射、动画与交互事件（README.md；`src/core/echarts.ts` 的 `setOption`）。

它走"配置驱动"路线而非组件拼装路线：option 中的 `series`/`xAxis`/`legend`/`tooltip` 等字段被解析为对应的 Model 实例树（`src/model/Global.ts`），再由 Scheduler 管线驱动处理与渲染。这使非前端专业人员也能产出专业图表，是其在中文生态与企业报表场景渗透率极高的核心原因。

ECharts 起源于百度（前身 Based on Canvas 的 echarts 1.x），2018 年进入 Apache 孵化、2021 年毕业为 ASF 顶级项目，是目前 Apache 基金会下 star 数最多的可视化项目。渲染层完全建立在自家 `zrender`（轻量 2D 绘制库）之上，图表库本体不含任何第三方可视化依赖（package.json dependencies）。

v6 主干（本快照）相对 5.x 的演进可见：新增 `matrix` 坐标系（`src/coord/matrix/`）、`thumbnail` 组件（`src/component/thumbnail/`）、`AxisBreak`/`ScatterJitter` 等 feature（`src/echarts.all.ts`）、独立的 SSR 服务端渲染子包（`ssr/client/`，配合 `build:ssr` 脚本）。

## 3. 技术栈与依赖

| 类别 | 内容 |
|---|---|
| 语言 | TypeScript 4.4（devDependencies），编译目标 ES5+（tsconfig.json） |
| 渲染 | `zrender@6.1.0`（自研 2D 库，提供 Canvas/SVG 双 painter、事件、动画、dirty 矩形管理） |
| 运行时依赖 | `tslib@2.3.0`（仅此两项，package.json） |
| 构建 | rollup 2 + @rollup/plugin-terser + esbuild（build/ 目录自有脚本 build.js/build-lib.js/build-i18n.js） |
| 测试 | jest 26 + ts-jest（单测）；自研 visual 测试服务器（test/runTest/server.js） |
| 代码质量 | eslint 7 + @echarts-x/eslint-plugin-ec（自有规则插件）、husky pre-commit、checkHeader.js 校验 ASF license header |
| 模块形态 | `"type": "module"` + 完整 `exports` map + `sideEffects` 白名单（package.json，支撑 tree-shaking） |

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `src/chart/` | 25 种图表的 Model+View 实现，171 个 ts 文件（bar/line/pie/scatter/radar/map/tree/treemap/graph/sankey/sunburst/custom 等） |
| `src/component/` | 约 30 类 UI/功能组件，187 个 ts 文件（grid/legend/tooltip/dataZoom/visualMap/timeline/markLine/dataset/matrix 等） |
| `src/coord/` | 8 套坐标系：cartesian(grid)/polar/radar/geo/calendar/parallel/single/matrix，71 个 ts 文件 |
| `src/core/` | 框架心脏：`echarts.ts`(3434 行 ECharts 类)、`Scheduler.ts`(681 行)、`task.ts`(478 行)、`lifecycle.ts`、`ExtensionAPI.ts` |
| `src/model/` | option→Model：`Global.ts`(1100 行 GlobalModel)、OptionManager、Component/Series 基类 |
| `src/data/` | 数据层 SeriesData/DataStore 等内部数据结构（19 个文件） |
| `src/processor/`、`src/visual/`、`src/layout/`、`src/scale/` | 管线四类阶段任务：数据处理、视觉编码、布局、刻度计算 |
| `src/echarts.all.ts / .common.ts / .simple.ts / .blank.ts / .ts` | 5 个打包入口（全量/常用/精简/空壳/核心默认） |
| `src/export/` | 对外导出整理层（core/charts/components/features/renderers） |
| `ssr/client/` | v6 新增服务端渲染入口 |
| `extension-src/` | 官方扩展（百度地图 bmap、dataTool） |
| `i18n/`、`theme/` | 语言包与主题 |
| `test/` | 612 个可视化用例 html + `test/ut/`（26 个 jest spec）+ 用例生成器 mktest |
| `build/` | rollup 打包脚本（自有，非依赖包） |

组织方式评述：单包 monorepo 倾向——本体一个 npm 包，但通过 `exports` map 暴露 `echarts/core`、`echarts/charts`、`echarts/components`、`echarts/renderers` 等子路径（package.json），配合 5 个入口实现按需引入；SSR 作为独立子包随主包分发。图表/组件均遵循统一的 Model(配置) + View(渲染) 双类结构，靠 install() 注册进全局，扩展点一致性非常好。

## 5. 核心架构与关键实现

### 5.1 插件式注册与按需打包（src/extension.ts）

一切图表/组件/坐标系/渲染器都是 "extension"，通过 `use()` 安装。install 函数拿到 `extensionRegisters`，内含 `registerChartView/registerSeriesModel/registerCoordinateSystem/registerVisual/registerLayout/registerProcessor/registerAction/registerPainter` 等约 18 个注册器：

```ts
export function use(ext: EChartsExtensionInstaller | EChartsExtension | ...) {
    if (isArray(ext)) { each(ext, (singleExt) => use(singleExt)); return; }
    if (indexOf(extensions, ext) >= 0) { return; }   // 幂等
    extensions.push(ext);
    if (isFunction(ext)) { ext = { install: ext }; }
    ext.install(extensionRegisters);
}
```

`src/echarts.all.ts` 末尾对全量 25 图表 + 30 组件逐个 `use()`；`src/echarts.ts`（默认入口）只默认安装 CanvasRenderer + DatasetComponent——这就是 tree-shaking 的关键：用户 `import * as echarts from 'echarts/core'` 后按需 `use(LineChart)`，未引用的模块因 `sideEffects` 白名单被 bundler 裁掉（src/extension.ts、src/echarts.ts、package.json）。

### 5.2 option 解析：OptionManager → GlobalModel（src/model/Global.ts）

`setOption` 生成 `GlobalModel`，其 `_mergeOption`（312 行起）用拓扑排序遍历组件主类型，把用户 option 映射/合并到既有 Model 实例：

```ts
const mergeMode =
      !oldCmptList ? 'replaceAll'
    : (replaceMergeMainTypeMap && replaceMergeMainTypeMap.get(mainType)) ? 'replaceMerge'
    : 'normalMerge';
const mappingResult = modelUtil.mappingToExists(oldCmptList, newCmptOptionList, mergeMode);
```

要点：没有注册 Model 类的字段（如 color/backgroundColor）直接 merge 到全局；有类的（series/grid/legend…）按 id/index 做 `normalMerge`（增量合并）、`replaceMerge`（`setOption(opt, {replaceMerge:['series']})` 时替换删除）或 `replaceAll` 三态合并，这是增量更新语义的实现基础（src/model/Global.ts、src/core/echarts.ts setOption 中对 replaceMerge 的透传）。

### 5.3 更新管线与调度（src/core/echarts.ts + src/core/Scheduler.ts + task.ts）

一次 `setOption` 触发 EC_FULL_UPDATE_CYCLE，`updateMethods.update`（约 1900 行）写明了固定阶段顺序：

```ts
scheduler.restoreData(ecModel, payload);
scheduler.performSeriesTasks(ecModel);
coordSysMgr.create(ecModel, api);          // 坐标系实例只在这一步重建
lifecycle.trigger('coordsys:aftercreate', ecModel, api);
scheduler.performDataProcessorTasks(ecModel, payload);
coordSysMgr.update(ecModel, api);
scheduler.performVisualTasks(ecModel, payload);
render(this, ecModel, api, payload, updateParams);
lifecycle.trigger('afterupdate', ecModel, api);
```

文件头部大注释块（226-300 行）系统描述了六种周期：FULL/PARTIAL（dispatchAction 短路优化）/PROGRESSIVE（分帧）/APPEND_DATA/SERIES_SPECIFIC，并规定"Model 只能在周期开头被修改"。`Scheduler` 把每类阶段封装为 Task/Pipeline，`Pipeline.threshold` 达到阈值且数据量够大时切换 progressive 渲染（每帧只画一小段），配合 large 模式应对大数据集（src/core/Scheduler.ts 的 Pipeline/PipelineContext 类型定义）。

### 5.4 zrender 渲染集成（src/view/Chart.ts + src/core/echarts.ts）

每个 ChartView 持有一个 zrender `Group`（`this.group = new Group()`，src/view/Chart.ts:139），渲染时整组挂到 zrender 实例（`zr.add(view.group)`，src/core/echarts.ts:1739），脏检查、事件命中、动画全部交给 zrender。渲染器本身也走注册制：Canvas/SVG painter 从 `zrender/src/zrender` 的 `registerPainter` 挂进来，`renderer/installCanvasRenderer.ts` 打包成 `CanvasRenderer` install 函数（src/extension.ts:60、src/echarts.all.ts 的 renderers use）。坐标系抽象见 `src/coord/CoordinateSystem.ts`（`create`/`update`/`getAxis` 等接口），系列通过 option 里 `coordinateSystem: 'cartesian2d' | 'geo' | ...` 与坐标系解耦绑定。

## 6. 构建与运行

来自 README.md 与 package.json scripts：

```bash
npm install
npm run dev        # dev:fast 构建 + http-server 打开 test/ 可视化用例页（watch 模式）
npm run checktype  # tsc --noEmit（含 extension-src）
npm run release    # build:lib + build:i18n + build(all,common,simple --min) + esm + extension + ssr
npm test           # jest --config test/ut/jest.config.cjs
npm run test:visual        # node test/runTest/server.js（可视化回归，浏览器逐 case 比对）
npm run test:dts           # dtslint 校验对外类型声明
npm run mktest             # 生成可视化测试用例骨架
```

产物在 `dist/`（tarball 内已带一份预构建：echarts.js/common/simple/esm 及 min 版，dist/）。

## 7. 测试与工程质量

- 单测：`test/ut/` 下 26 个 jest spec（jest 26 + ts-jest + jest-canvas-mock），覆盖 core/data/coord 等纯逻辑层（test/ut/spec/）。
- 可视化回归是主要质量手段：`test/` 下 612 个 html 用例（绝大多数图表/组件交互场景），由自研 `test/runTest/server.js` 起服务在真实浏览器中执行比对（package.json test:visual、test/ 目录）。
- CI：`.github/workflows/ci.yml` 双 job（Node 20）——lint + build generated types + checktype；unit test + build release。另有 nightly.yml、nightly-next.yml（每夜构建）、source-release.yml（Apache 源码发布投票流程）、stale.yml（issue 保活），.asf.yaml 配置 ASF 仓库策略。
- 工程约束严格：`build/checkHeader.js` + `.headerignore` 强制所有源文件带 ASF license header；husky pre-commit；自有 eslint 插件 @echarts-x/eslint-plugin-ec；dtslint 保证 4 个版本 TS 的类型兼容（devDependencies @definitelytyped/typescript-versions）。

## 8. 社区与维护状态

- ASF 顶级项目，治理成熟：NOTICE/KEYS/.asf.yaml/dev@echarts.apache.org 邮件列表齐全（仓库根目录、.github/）。
- 活跃度高：最近 push 2026-10-13（gh api），6.1.0 于 2026-05-19 发布；open issues 1490、forks 19928、watch 1953。release 节奏为一年一大版（5.x→6.0→6.1）+ 补丁版。
- CI 徽章显示 GitHub Actions 全绿（README.md）；nightly 工作流存在说明主干每日有自动验证。
- 贡献者结构与 PMC 运作细节本快照未确认（tarball 无 git 历史，未拉取 contributors 列表）。

## 9. 亮点与风险

亮点：
1. 扩展点设计教科书级：Model+View+install() 三件套统一了图表、组件、坐标系、渲染器四类扩展，第三方生态（地图、GL、统计扩展）都吃同一套 API（src/extension.ts、src/export/）。
2. 按需打包做进架构而非构建 trick：核心入口零图表、sideEffects 白名单 + exports 子路径 + 5 档预置入口（blank/simple/common/all），体积可控（package.json、src/echarts*.ts）。
3. 大数据性能方案完整：progressive 分帧渲染 + large 模式 + appendData 增量追加，Scheduler 的 Pipeline 抽象是业界少见的通用化设计（src/core/Scheduler.ts、task.ts）。
4. 更新语义精细：normalMerge/replaceMerge/notMerge 三种合并策略 + universalTransition 跨系列动画（src/model/Global.ts）。
5. 工程质量保障立体：单测 + 612 个可视化用例 + 类型声明专项测试 + license header 检查 + nightly（test/、build/、.github/workflows/）。

风险/局限：
1. 核心文件过大：src/core/echarts.ts 3434 行、Global.ts 1100 行，内部用 `__flagInMainProcess` 等魔法键位（IN_EC_CYCLE_KEY 等）维护隐式状态机，新人读源码成本高（src/core/echarts.ts）。
2. TypeScript 版本偏保守（4.4.3），新语法特性受限；eslint 7、jest 26 也偏旧（devDependencies）。
3. 612 个可视化用例依赖人工维护，HTML 形态难以并行/无头化扩展，回归成本随图表数增长（test/ 目录）。
4. 深度定制仍需理解 option→model→pipeline 的全局心智模型，配置面巨大导致文档与实现间存在灰区（未确认，基于 option 字段量推断）。

## 10. 总结评价

ECharts 是"配置驱动可视化"路线的巅峰之作：以声明式 option 为界面、以 Scheduler 管线为引擎、以 zrender 为画笔，用极少的运行时依赖和极规范的扩展点撑起了 25 种图表与 8 套坐标系，同时把按需加载和大数据渲染这类工程难题做进了架构本身。v6 主干仍在积极演进（SSR、matrix 坐标系等），ASF 治理与测试体系保证了长期可维护性。适合前端/数据团队直接作为生产图表底座；对框架设计者而言，`src/extension.ts` + `src/core/Scheduler.ts` + `src/model/Global.ts` 三个文件是理解"插件注册、任务调度、配置合并"三个经典问题的高质量范本，值得精读。
