# mrdoob/three.js 项目分析

> 星数 116822（快照 2026-10-24）｜主语言 JavaScript｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：Web 端事实标准的 3D 渲染库——用纯 JavaScript（零运行时依赖）封装 WebGL/WebGPU，让浏览器里的三维图形开发像搭积木一样简单。

| 项目 | 内容 |
|---|---|
| 名称 | mrdoob/three.js（npm 包名 `three`） |
| 星数 | 116,822（快照）；gh api 实测 117,186（2026-10-29） |
| 语言 | JavaScript（ESM，纯 JS 无 TypeScript） |
| 许可证 | MIT（LICENSE，版权 2010-2026 three.js authors） |
| 版本 | package.json `0.186.0`；dev 分支 `REVISION = '187dev'`（src/constants.js） |
| 默认分支 | dev（gh api） |
| 最近 push | 2026-10-29（gh api，几乎每天有提交） |
| 创建时间 | 2010-03-23（gh api） |
| 贡献者 | 约 2,600+（gh api contributors 分页末页 2610） |

## 2. 项目是什么，解决什么问题

three.js 由 Ricardo Cabello（mrdoob）于 2010 年创建，目标是"易用、轻量、跨浏览器的通用 3D 库"（README.md）。它把浏览器底层图形 API（WebGL/WebGPU）复杂的着色器、矩阵数学、资源加载等细节封装成场景图（Scene Graph）+ 渲染器的经典范式：开发者只需组织 `Scene / Camera / Mesh(geometry, material)` 对象树，交给 `renderer.render(scene, camera)` 即可出图（README.md 提供的 30 行示例即为完整可运行代码）。

WebGL 编程要求开发者手写 GLSL、管理缓冲区和状态机，门槛高且重复劳动多；three.js 用一套统一的抽象层解决了这个问题：几何体（BufferGeometry）、材质（Material 体系）、光照（各类 Light）、相机、加载器（Loader）各司其职，且全部对象基于 `Object3D` 场景图组合（src/core/Object3D.js，1719 行）。它同时内建了色彩管理（ColorManagement）、PBR 材质模型（MeshStandardMaterial/MeshPhysicalMaterial）、阴影、后处理、WebXR 等引擎级特性。

当前版本正处于从 WebGL 向 WebGPU 的架构迁移期：新版 `WebGPURenderer` 采用"统一 Renderer 基类 + 可插拔 Backend"设计（src/renderers/common/Renderer.js 的 `Renderer` 类 + src/renderers/webgpu/WebGPUBackend.js 与 src/renderers/webgl-fallback/WebGLBackend.js），并引入节点化着色系统 TSL（Three Shading Language，src/nodes/），一套节点图可同时编译为 WGSL 和 GLSL。传统 `WebGLRenderer` 仍在维护（src/renderers/WebGLRenderer.js），两者并行提供（package.json 的 `exports` 同时暴露 `.` 与 `./webgpu`、`./tsl` 入口）。

按 README 的说法，当前构建包含 WebGL 与 WebGPU 渲染器，SVG 与 CSS3D 渲染器以 addon 形式提供。生态位上它是 Web 3D 的"默认答案"：npm 每周下载量以百万计（README 徽章）、教程/书籍/求职技能栈均以其为准，GitHub 上数量庞大的下游项目（可视化、数字孪生、Web 游戏、产品展示）直接或间接依赖它。

## 3. 技术栈与依赖

- 语言：纯 JavaScript（ESM，`"type": "module"`，package.json），无 TypeScript，靠 JSDoc 注释生成 API 文档（`build-docs` 脚本用 jsdoc）。
- **运行时依赖为零**：package.json 的 `dependencies` 字段不存在，所有 `devDependencies`（eslint、rollup、puppeteer、jsdoc 等 17 项）均为工程链工具。第三方功能（DRACO 解码、物理引擎 ammo/rapier 等）以 addon 内嵌库或示例外部脚本形式提供（examples/jsm/libs/）。
- 构建：Rollup（`build`: `rollup -c utils/build/rollup.config.js`），产出多入口包（build/ 目录）：
  - `three.core.js`（约 1.44MB，核心共享模块）、`three.module.js`（约 0.64MB，WebGL 版，从 three.core 再导出）、`three.webgpu.js`（约 1.29MB）、`three.tsl.js`、`three.cjs`。
- 代码规范：eslint + `eslint-config-mdcs`（mrdoob 代码风格），src 与 examples 分开 lint（package.json scripts 的 lint-core/lint-addons）。
- 测试：Puppeteer 驱动的浏览器内单元测试与端到端截图对比测试。
- 环境：仅要求现代浏览器（browserslist `> 1%, not dead, not ie 11`），支持 WebGL2 与 WebGPU。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| src/ | 库本体，约 757 个 JS 文件、约 18.7 万行（实测统计） |
| src/core/ | 场景图基类：Object3D、BufferGeometry、BufferAttribute、Raycaster、RenderTarget 等 |
| src/cameras/ , scenes/ , lights/ | 相机（6 种）、场景与雾、光源 |
| src/materials/ | 19 个经典材质 + materials/nodes/ 下对应的 NodeMaterial 体系 |
| src/geometries/ | 22 种内置几何（Box/Sphere/Extrude/TorusKnot 等） |
| src/renderers/ | WebGLRenderer（旧线）；common/（新线统一渲染器基类，53 个文件）；webgpu/、webgl-fallback/（两个 Backend）；shaders/（GLSL ShaderChunk/ShaderLib）；webxr/ |
| src/nodes/ | TSL 着色语言与节点系统：core/（40 个节点基元）、accessors/、lighting/、math/、parsers/ 等 20 个子目录 |
| src/loaders/ , textures/ , extras/ | 基础加载器（FileLoader/ImageLoader 等）、纹理、PMREMGenerator/曲线等工具 |
| examples/ | 622 个 HTML 示例 + examples/jsm/ 503 个 addon 模块（约 23 万行）+ 437MB 资源文件 |
| editor/ | 官方在线场景编辑器（threejs.org/editor 的源码） |
| manual/ , docs/ , tsl/ | 教程手册、JSDoc 生成的 API 文档、TSL 在线试验场 |
| test/ | unit/（269 个 *.tests.js，浏览器内跑）、e2e/（示例截图对比）、benchmarks/、treeshake/ |
| utils/ | 构建/文档生成本地工具链（rollup 配置、dev server、llms.txt 生成） |
| .github/workflows/ | ci.yml、protected-folders.yml、codeql、report-size 等 6 个工作流 |

组织方式评述：单仓单包（不是 monorepo），但"库 + 海量示例 + 官方编辑器 + 全套文档"同仓共存的巨型单体。addon（examples/jsm）按功能域分目录（57 个 loaders、30 个 postprocessing、controls、exporters 等），通过 `three/addons/*` 的 npm exports 映射对外发布（package.json + examples/jsm/Addons.js 统一 re-export）。约定非常克制：全库无构建期宏、无装饰器，贴近原生 JS。

## 5. 核心架构与关键实现

### 5.1 场景图与渲染主循环（WebGLRenderer）

一切皆 `Object3D`（src/core/Object3D.js）：父子变换矩阵、visible/layers 剔除、事件派发都在此基类。`WebGLRenderer.render()`（src/renderers/WebGLRenderer.js:1641）的经典流程是递归遍历场景树，按对象类型分流：

```javascript
// src/renderers/WebGLRenderer.js:1873
function projectObject( object, camera, groupOrder, sortObjects ) {
    if ( object.visible === false ) return;
    const visible = object.layers.test( camera.layers );
    if ( visible ) {
        if ( object.isGroup ) {
            groupOrder = object.renderOrder;
        } else if ( object.isLOD ) {
            if ( object.autoUpdate === true ) object.update( camera );
        } else if ( object.isLight ) {
            currentRenderState.pushLight( object );
            if ( object.castShadow ) {
                currentRenderState.pushShadow( object );
            }
        } else if ( object.isSprite ) {
        ...
```

遍历结果进入不透明/透明两条渲染队列，按深度排序后逐对象 `renderBufferDirect`（同文件 :1200）。着色器采用"ShaderChunk 拼装"方案：src/renderers/shaders/ 下 109 个 GLSL 片段（ShaderChunk/）由 ShaderLib 按材质特性条件拼接成完整程序，这是 WebGL 线十几年的核心机制。

### 5.2 新一代统一渲染器与双 Backend（WebGPU 线）

src/renderers/common/Renderer.js:129 定义了 4225 行的 `Renderer` 基类，集中了与图形 API 无关的全部逻辑（渲染列表、裁剪上下文、光照、XR、后处理框架、异步管线编译——内部甚至做了主线程让步调度：`_compileYieldInterval = 8` 毫秒预算、最多 16 条并行编译管线，见文件头部）。`WebGPURenderer extends Renderer`（src/renderers/webgpu/WebGPURenderer.js:28）只负责对接 WebGPU API；同构的 `WebGLBackend`（src/renderers/webgl-fallback/WebGLBackend.js，2878 行）让不支持 WebGPU 的浏览器自动降级回 WebGL。

### 5.3 TSL 节点着色系统

新线的着色器不再是 GLSL 字符串，而是节点图：src/nodes/core/ 提供 Node/NodeBuilder/PropertyNode 等 40 个基元，src/materials/nodes/NodeMaterial.js 用 TSL 函数组合出完整的 PBR 管线（其 import 区可见 `output, diffuseColor, ambientOcclusion` 等 PropertyNode 与 `skinning, morphReference` 等 accessors 的拼装）。TSL（src/nodes/tsl/TSLCore.js）提供方法链式 API（`addMethodChaining` 直接改写 Node 原型），用户写的节点流最终由 NodeBuilder（src/nodes/core/NodeBuilder.js，3918 行）编译为 WGSL（WGSLNodeBuilder，2834 行）或 GLSL（GLSLNodeBuilder，1994 行）。这让同一份材质代码同时跑在 WebGPU 与 WebGL2 上，也是 `three.tsl.js` 独立入口的由来。

### 5.4 Addon 生态的组织

examples/jsm 既是 622 个示例的支撑库，也是正式发布的 addon 包：`"exports": { "./addons/*": "./examples/jsm/*" }`（package.json）。57 个格式加载器（GLTF/FBX/USDZ/3DTiles…）、30 个后处理 pass、OrbitControls 等 9 类相机控制、12 种导出器均在此。示例本身就是回归测试载体——e2e 测试逐个渲染示例页面截图比对（test/e2e/）。

## 6. 构建与运行

来自 package.json scripts 与 README：

```bash
npm install                 # 仅 devDependencies，无运行时依赖
npm run dev                 # 启动开发服务器（utils/server.js，端口 8080）
npm run build               # rollup 打包 build/ 多入口
npm run build-docs          # jsdoc 生成 API 文档 + llms.txt
npm run test                # lint + 单元测试 + addons 单元测试
npm run test-unit           # puppeteer headless 跑 test/unit/UnitTests.html
npm run test-e2e            # 逐示例截图对比（配合 xvfb + mesa vulkan 可测 WebGPU）
npm run make-screenshot     # 生成 e2e 基准截图
```

浏览器直接用：`import * as THREE from 'three'`（WebGL 线）或 `import * as THREE from 'three/webgpu'`；addon 经 `import { OrbitControls } from 'three/addons/controls/OrbitControls.js'`。CDN（jsDelivr/unpkg）开箱即用，无需打包器。

## 7. 测试与工程质量

- **单元测试**：test/unit 下 269 个 `*.tests.js` 文件，运行方式独特——通过 puppeteer 加载 `UnitTests.html`/`UnitTestsAddons.html` 在真实浏览器里跑（package.json `test-unit`），覆盖数学、核心、部分加载器与 exporter；渲染相关逻辑因依赖 GPU 环境无法纯 Node 测试，这是其选择浏览器测试的根本原因。
- **e2e/示例回归**：test/e2e/check-coverage.js 校验"所有示例都已登记测试"，CI 在 ubuntu + mesa vulkan 驱动 + xvfb 下并行 5 个分片跑全部 622 个示例的截图对比（.github/workflows/ci.yml），这意味着每次 PR 都实际上渲染了一遍整个示例库。
- **CI**：ci.yml（lint=eslint src、unit、unit-addons、e2e-cov、e2e 五分片）、codeql 代码扫描、renovate 依赖自动更新、protected-folders.yml 拦截非维护者对 build/、docs/（自动生成目录）的改动。
- 代码质量观察：零依赖、严格 lint（mrdoob 风格，tab 缩进、`const` 纪律）、JSDoc 全覆盖用于生成文档；单文件体量大（Renderer.js 4225 行、NodeBuilder.js 3918 行）是历史演进的结果，但模块边界清晰。

## 8. 社区与维护状态

- 极度活跃：gh api 显示最近 push 2026-10-29，384 个开放 issue，36,808 forks，约 2,600+ 贡献者（分页推断）。创建 16 年仍由 mrdoob 本人主导日常提交。
- Release 节奏：r186（2026-09-24）、r185（2026-07-01）、r184（2026-04-16）、r183（2026-02-20）（gh api releases）——近年约每 2-3 个月一个大版本，每版附带 Migration Guide（README 链接 wiki）。
- 贡献流程：CONTRIBUTING.md 强调 PR 需对应 issue、代码须通过 lint；build/ 与 docs/ 由维护者自动生成，贡献者不得手改（.github/workflows/protected-folders.yml）。社区渠道为 forum.discourse.threejs.org 与 Discord（README）。
- 文档设施罕见地完备：API 文档、manual 教程、622 个可交互示例、面向 LLM 的 docs/llms-full.txt 与根目录 llms.txt（2026 年新增，README/llms.txt 可见）。

## 9. 亮点与风险

亮点：

1. **零运行时依赖的纯 JS 实现**：约 18.7 万行源码自包含数学库到双渲染后端，审计与 tree-shaking 友好（test/treeshake 专门做摇树验证）。
2. **WebGL→WebGPU 平滑迁移架构**：统一 Renderer 基类 + Backend 插件 + TSL 跨 API 着色语言（src/renderers/common/Renderer.js、src/nodes/），是业界少见的双后端过渡范本。
3. **示例即文档、示例即测试**：622 个示例覆盖几乎所有特性，e2e 截图回归把它们变成持续验收网。
4. **16 年持续演化且向后兼容记录良好**：大版本虽频繁但始终提供 Migration Guide，生态信任度极高。

风险/局限：

1. **双渲染线并行的过渡期复杂度**：WebGLRenderer 与 WebGPURenderer 行为差异、TSL 与 GLSL ShaderChunk 两套着色体系并存，学习与迁移成本高（r187dev 源码可见两线并存）。
2. **部分核心文件巨石化**：Renderer.js、NodeBuilder.js、WebGLRenderer.js 均 3000-4000+ 行，新人啃读困难。
3. **版本号长期 0.x**（0.186.0，package.json），API 语义化承诺弱，小版本也可能带破坏性变更。
4. 测试依赖真实浏览器与 GPU 驱动，本地完整跑测试环境搭建门槛高（ci.yml 需要 mesa vulkan + xvfb）。

## 10. 总结评价

three.js 是 Web 3D 领域的"默认基础设施"：它以零依赖、极致克制的纯 JS 实现撑起了浏览器三维渲染的整个生态，又在前端图形 API 换代（WebGL→WebGPU）的关键节点交出了 TSL 节点系统 + 双 Backend 的教科书级迁移方案。它不追引擎大而全（不做物理、不做编辑器级工作流——编辑器也只是示例级产品），专注把"渲染、资源、易用性"做到极致，这正是它能积累 11.6 万星并保持日均提交的根本原因。适合所有需要在浏览器里做 3D 的开发者直接使用；对图形工程师而言，src/renderers/common/Renderer.js、src/nodes/（TSL 实现）与 src/renderers/WebGLRenderer.js 是学习"如何设计可长期演化的渲染器抽象"的一线源码教材。
