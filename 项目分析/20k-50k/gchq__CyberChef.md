# gchq/CyberChef 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-17

## 1. 项目速览

一句话定位：GCHQ（英国政府通信总部）出品的"网络瑞士军刀"——一个纯浏览器端运行的编码/加密/压缩/数据分析工具，用"操作拖拽成流水线（Recipe）"的方式替代手写脚本。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | gchq/CyberChef | GitHub |
| 星数 | 39500（快照）；gh api 2026-11-17 实测 36,426 | 任务参数 + gh api |
| 语言 | JavaScript（纯 JS，无 TypeScript） | gh api / package.json |
| 许可证 | Apache-2.0（Crown Copyright） | LICENSE、package.json |
| 版本 | v11.5.0（2026-09-18 发布） | package.json、CHANGELOG.md、gh api releases |
| 最近活跃 | 最近 push 2026-11-16（分析前一天） | gh api pushed_at |
| 创建时间 | 2016-11-28 | gh api created_at |
| 贡献者 | 264 人 | gh api contributors 分页统计 |

## 2. 项目是什么，解决什么问题

安全分析人员在日常工作中经常需要把 Base64 解码再解压再解 AES、解析 hexdump、拆 IPv6 地址这类"一次性小流水线"。传统做法是每次现写 Python 脚本，而 CyberChef 把这类工作变成图形化操作：左侧是按 17 个分类组织的几百个操作（Operation），中间把操作拖成 Recipe 链，右上输入、右下输出，改任何一步都会自动重新"烘焙"（Auto Bake）。README 明确说明该工具最初由一位分析师在 GCHQ 的 10% 创新时间内设计并逐年迭代而成（README.md "How it works" 前）。

它最关键的设计决策是**纯前端**：所有处理都在浏览器内完成，输入与配置永远不会发往服务器（README.md Features 节 "CyberChef is entirely client-side"）。这带来三个直接后果：其一，构建产物可以整体下载为一个单文件 HTML，放进内网隔离环境或虚拟机离线使用，这对情报/安全机构是刚需；其二，官方托管站 gchq.github.io/CyberChef 本身只是个静态页，零后端成本；其三，数据不出本机，降低了涉密数据外流的合规风险。

操作覆盖面非常宽：Base64/XOR 等简单编码、AES/DES/Blowfish/Ascon/Argon2 等对称与非对称加密及密钥派生、gzip/zlib/bzip2/LZMA/LZ4 等压缩、MD5/SHA 系列与各类校验和哈希、X.509/PEM/JWT/Protobuf/Avro/AMF 等格式解析、IP/GeoIP/UUID 分析、文件类型识别（含 libyara-wasm YARA 规则、tesseract.js OCR、jimp 图像处理）乃至 jq/JSONata/SQL 格式化等数据处理（依赖清单见 package.json dependencies）。"Magic" 操作还能对输入做推测性执行，自动猜测嵌套编码层数并给出一键解码建议（src/core/operations/Magic.mjs，1122 行）。

在生态位上，它几乎垄断了"浏览器端万能解码/解密瑞士军刀"这一品类：CTF 比赛、恶意样本分析、渗透测试前期数据处理的事实标准工具。同类工具要么能力窄（如 Burp Suite 的 Hackvertor 插件只做编码转换）、要么形态不同（Python 脚本/jupyter 需要环境），社区里 cyberchef-server（aerieq）等项目则是把 CyberChef 核心封装成后端 API 的衍生形态——本质上都以 CyberChef 的操作库为基准。官方还发布 npm 包 `cyberchef` 并提供 Node.js API（src/node/），让同一套操作库可以在无浏览器环境下脚本化复用。

## 3. 技术栈与依赖

- **语言/运行时**：纯 JavaScript（ESM `.mjs` 为主），要求 Node.js v24（`engines: ">=24 <27"`，package.json），浏览器基线 Chrome 50+/Firefox 38+（README Browser support、browserslist）。
- **UI**：无框架、无 React/Vue——原生 JS 类 + jQuery 3.7 + Bootstrap 4.6 + CodeMirror 6（编辑器），分裂面板用 split.js（package.json、src/web/App.mjs）。
- **构建链**：Grunt 1.6 任务编排 + Webpack 5 打包 + Babel 转译 + PostCSS；产物做 gzip/brotli 压缩（Gruntfile.js、webpack.config.js）。
- **密码学/编码库**（均为前端可用实现）：crypto-js、node-forge、openpgp、jsrsasign、bcryptjs、scryptsy、argon2-browser、otpauth、kbpgp、@noble/hashes、js-sha3 等；国密/俄标支持靠 @wavesenterprise/crypto-gost-js（package.json dependencies）。
- **重型 WASM/移植库**：libyara-wasm（YARA 规则）、libbzip2-wasm、tesseract.js（OCR）、jq-web（jq 语言）、@alexaltea/capstone-js（反汇编）、jimp（图像）。
- **工具库**：d3（可视化）、moment-timezone（时间）、jsonata/jsonpath-plus/jq-web（查询）、esprima/escodegen（JS 处理）、protobufjs、avsc、bson、cbor2（序列化）。
- 运行时依赖约 100 项，devDependencies 约 50 项（package.json 实数：dependencies 96、devDependencies 52）。

## 4. 目录结构与代码组织

| 目录 | 职责 | 出处 |
| --- | --- | --- |
| src/core/ | 核心引擎与操作库（与 UI 完全解耦，浏览器/Node/Worker 三端共用） | src/core/ |
| src/core/operations/ | 506 个操作实现，一个操作一个 `.mjs` 文件 | 目录统计 |
| src/core/lib/ | 74 个操作支撑库（Base64、Magic、FileType 等算法实现） | 目录统计 |
| src/core/dishTypes/ | 8 种数据类型适配器（String/byteArray/BigNumber/JSON/File/ListFile/Number/HTML） | src/core/dishTypes/ |
| src/core/config/ | Categories.json（17 分类）+ config/scripts/ 构建期代码生成器 | src/core/config/ |
| src/web/ | 浏览器 UI：App.mjs 主控 + Manager + 14 个 waiter + html/stylesheets | src/web/ |
| src/web/workers/ | 4 个辅助 Web Worker（Input/Loader/Zip/Dish） | src/web/workers/ |
| src/node/ | Node API：CJS wrapper.js、ESM index.mjs、NodeDish/NodeRecipe、repl | src/node/ |
| tests/ | operations（203 个声明式测试文件）/ node（自研 runner）/ browser（Nightwatch e2e） | tests/ |
| .github/workflows/ | master.yml 构建测试部署、releases.yml、CLA 系列机器人 | .github/workflows/ |

组织方式评述：单仓单体（非 monorepo），但内部有清晰的三层划分——core 引擎层、operations 操作层（数据驱动的插件集合）、web/node 两个宿主。值得注意的是**大量代码是构建期生成的**：`generateOpsIndex.mjs` 扫描 operations 目录生成 import 索引，`generateConfig.mjs` 实例化每个操作生成 OperationConfig.json 与 OpModules 模块映射（src/core/config/scripts/generateConfig.mjs 头注释），因此 tarball 中看不到 OperationConfig.json——这是设计而非缺失。

## 5. 核心架构与关键实现

### 5.1 Operation 抽象：一切皆操作

所有 506 个操作继承同一个基类 `Operation`（src/core/Operation.mjs，355 行）。基类定义了四类信息：类型契约（inputType/outputType/presentType）、参数清单（args，声明式 UI 描述）、行为（run/highlight/highlightReverse/present 四个可覆盖方法）、执行语义开关（flowControl/manualBake/breakpoint/disabled）。以 FromBase64 为例（src/core/operations/FromBase64.mjs）：

```js
class FromBase64 extends Operation {
    constructor() {
        super();
        this.name = "From Base64";
        this.module = "Default";
        this.inputType = "string";
        this.outputType = "byteArray";
        this.args = [
            { name: "Alphabet", type: "editableOption", value: ALPHABET_OPTIONS },
            { name: "Remove non-alphabet chars", type: "boolean", value: true },
        ];
    }
}
```

每个操作文件还可携带 `checks` 正则模式数组（FromBase64.mjs 内含 8 条），供 Magic 操作在推测执行时反向匹配"这串数据像 Base64"并给出参数建议——操作自描述是其插件生态低门槛的关键：README 明确说"会写基本 JS 就能写操作"，并配有 `npm run newop` 脚手架（package.json scripts、src/core/config/scripts/newOperation.mjs）。

### 5.2 Recipe 执行引擎：带流程控制的线性流水线

Recipe 不是 DAG，而是**可被流程控制操作改写游标的线性链**。`Recipe.execute()`（src/core/Recipe.mjs L183-268）按序执行：每个操作先 `dish.get(op.inputType)` 做类型转换，再 `op.run(input, args)`，结果 `dish.set(output, op.outputType)` 写回；支持 breakpoint 断点暂停与 disabled 跳过。关键在 flowControl 分支——Fork/Jump/ConditionalJump/Register/Label/Return/Merge/Subsection 等操作（src/core/operations/ 下含 flowControl 标记）会把整个执行状态打包传入 `op.run(state)` 并接管游标 `i = state.progress`，由此实现分支跳转、数据分叉（Fork 把 dish 切成多份各自走完剩余流水线）与寄存器捕获。OperationError 被捕获后直接 `dish.set(err.message)` 作为输出而非中断（L240-249），这符合分析工具"部分失败也要看到中间结果"的使用习惯。操作实例通过构建期生成的 OpModules 映射动态 hydrate（Recipe.mjs L61-84，webpackMode "eager"）。

顶层控制器 Chef.bake()（src/core/Chef.mjs L40-89）负责计时、错误包装与 silentBake（后台预热避免浏览器 tab 切换后 JIT/内存回收导致的卡顿，L92-122 注释详述了动机）。

### 5.3 Dish 数据总线：8 种类型间的惰性转换

Dish（src/core/Dish.mjs，567 行）是贯穿流水线的数据容器，内部统一以 ArrayBuffer 持有数据，按需在 8 种类型间 translate（dishTypes/ 下每类型一个适配器，如 DishString、DishBigNumber、DishFile、DishListFile）。操作只声明"我要什么类型"，Dish 负责最小代价转换并尽量复用缓冲——这使 string→byteArray→BigNumber→JSON 的任意操作组合无需用户手工转换，是几百个操作能自由拼接的类型系统基础。

### 5.4 Web UI 与 Worker 并行

UI 无框架，App.mjs（845 行）实例化 Manager（363 行），后者聚合 14 个 waiter（Worker/Window/Controls/Recipe/Operations/Input/Output/Options/Highlighter/Seasonal/Bindings/BackgroundWorker/Tab/Timing，src/web/Manager.mjs L7-20），每个 waiter 负责一块 DOM 的事件绑定——自制的模块化事件总线。真正的执行在 ChefWorker.js（290 行）Web Worker 中，UI 通过 postMessage 的 `bake`/`silentBake`/`getDishAs` 动作通信并实时回传进度（ChefWorker.js 头部消息协议注释）；另有 InputWorker/LoaderWorker/ZipWorker/DishWorker 分别处理大文件载入（README 称支持拖入 2GB 文件）、解压预览等重活，保证主线程不卡。TabWaiter 支持多输入多标签页并行烘焙。

单文件产物由 Grunt 的 `standalone` 任务完成：把构建好的 index.html 原地内联为 `build/prod/CyberChef_v${pkg.version}.html`（Gruntfile.js L333），即 README 所说可下载离线运行的完整副本；CI 顺带产出 75MB+ 的 zip 与 SHA256（Gruntfile.js calcDownloadHash 注释）。Docker 路线为多阶段构建：node:24-alpine 构建 → nginx-unprivileged 托管纯静态文件（Dockerfile）。

## 6. 构建与运行

来自 README.md "Common tasks" 表与 package.json scripts（均为官方原文，未实际运行验证）：

```bash
# Docker 一键运行（官方镜像）
docker run -it -p 8080:8080 ghcr.io/gchq/cyberchef:latest

# 源码开发（Node.js v24 必须，项目提供 .nvmrc）
npm install
npm start        # grunt dev，localhost:8080 热更新
npm run build    # grunt prod，产物 build/prod/（含单文件 HTML）
npm test         # grunt configTests + Node API 测试 + 操作测试
npm run testui   # Nightwatch 浏览器 e2e（需先 prod 构建）
npm run lint     # grunt eslint
npm run newop    # 交互式生成新操作脚手架
```

约束：Node 引擎 `>=24 <27`（package.json engines）；CI 矩阵在 Node 24/26 上跑（.github/workflows/master.yml 注释：24 为官方支持版本，26 提前验证）。

## 7. 测试与工程质量

- **操作测试**：tests/operations/tests/ 下 203 个文件，全部是声明式用例——`TestRegister.addTests([{name, input, expectedOutput, recipeConfig}])`（tests/operations/tests/A1Z26CipherDecode.mjs、AESKeyWrap.mjs 实例，含 RFC 测试向量）。自研 TestRegister runner（tests/lib/TestRegister.mjs），无 jest/mocha 依赖。抽样统计（`"input":` 键计数）约 124+ 用例，实际总数更多（部分文件多向量，未逐文件全量确认）。
- **Node API 测试**：tests/node/index.mjs 汇总 nodeApi/operations/PGP/File/Dish/Utils/Categories 等 13 个套件，用 assertionHandler.mjs 适配 assert（tests/node/index.mjs import 列表）。
- **E2E**：tests/browser/ 下 Nightwatch 套件 5 个文件（00 初始化、01 IO、02 操作、03 recipe 加载、04 拖放），nightwatch.json 配置，依赖 chromedriver 152（package.json devDependencies）。
- **消费兼容**：`npm run testnodeconsumer` 单独验证 CJS/ESM 两种消费方可用（Gruntfile.js testnodeconsumer 任务）。
- **CI**：master.yml 在 push master 时跑 lint→单元测试→prod 构建→sitemap→GitHub Pages 部署，Node 24+26 矩阵，actions 均按 commit SHA 固定版本（.github/workflows/master.yml）；另有 releases.yml、minor_version_update.yml、PR 分类与 CLA 机器人共 9 个 workflow（.github/workflows/ 目录）。
- **质量观察**：全部 107534 行 JS（src/core+web+node 实测 wc -l）有 eslint 10 + eslint-plugin-jsdoc 约束（lint 强制 JSDoc，每个文件头部有作者/版权/许可证三行头）；拼写检查 cspell 集成（npm run lint:grammar）；CHANGELOG 按 semver 严格记录（CHANGELOG.md Versioning 节）。一个诚实的短板标注在 SECURITY.md：密码学操作的正确性/安全性"不提供保证"，仅尽力支持。

## 8. 社区与维护状态

- **活跃度**：gh api 实测最近 push 2026-11-16（分析前一天），2016-11-28 创建至今近十年持续维护；CHANGELOG 最新版本 11.5.0 发布于 2026-09-18，单个 minor 版本即收录约 30 条变更（含外部贡献者 ousamabenyounes、sylvesterkaczmarek、lzandman、evan-cohen、Adamkadaban 等署名条目，CHANGELOG.md）。
- **发布节奏**：GitHub releases + npm 包 + ghcr 镜像三渠道同步（README Docker 节、package.json npm 徽章）；minor_version_update.yml 专门自动化 minor 版本流程。
- **贡献者结构**：264 名贡献者（gh api contributors 分页实测）；外部 PR 需签署 GCHQ CLA（README Contributing 节，cla-assistant.io 集成，配套 5 个 CLA 相关 workflow）；核心开发由 GCHQ 内部开发者主导（多个提交署名 @GCHQDeveloper581）。
- **社区渠道**：Gitter 聊天室（README 徽章）、issue 589 open（gh api）。
- **背景**：GCHQ 是英国情报机构，CyberChef 是其最著名的开源项目；贡献 CLA 甚至询问贡献者是否愿意接受 GCHQ 的"致谢或工作机会"（README Contributing 节原文）。
- 星数说明：任务快照 39500（2026-10-24），gh api 于 2026-11-17 返回 36,426，两口径存在差异，以快照为准记录、实测值如实注明（未确认差异原因）。

## 9. 亮点与风险

**亮点**
1. 纯前端零后端 + 单文件 HTML 产物：数据不出本机、可离线/内网部署，是该品类几乎唯一的实现方式（README Features、Gruntfile.js standalone 任务）。
2. 操作库规模与广度惊人：506 个操作、17 大类，从 Base64 到 YARA、OCR、反汇编、国密算法一站覆盖，且每个操作自带 RFC 测试向量级声明式用例（src/core/operations/、tests/operations/tests/）。
3. 低门槛插件生态：一个类 + 声明式 args + `npm run newop` 脚手架 + 构建期自动索引/配置生成，新增操作零接线成本（src/core/config/scripts/generateConfig.mjs）。
4. 架构分层干净：core 与宿主彻底解耦，同一套操作库跑在浏览器 Worker、Node CJS/ESM、REPL 三种环境（src/node/、package.json exports）。
5. 十年持续演进的工程质量：semver + 详尽 CHANGELOG、三端测试、SHA 固定的 CI、双 Node 版本矩阵（.github/workflows/master.yml）。

**风险/局限**
1. 运行时依赖约 100 项且含大量 WASM 包（tesseract.js、libyara-wasm 等），供应链攻击面大；官方在 SECURITY.md 明确"不保证密码学操作的正确性与安全性"，输出不可直接作为安全依据。
2. 纯原生 JS 无框架的 UI 在规模上已达上限：App.mjs 845 行、14 个 waiter 的手工事件管理，可维护性依赖约定而非框架约束（src/web/Manager.mjs）。
3. 大数据场景受限：虽支持拖入 2GB 文件，但浏览器内存与 Worker 单线程烘焙决定了实际可用输入远小于此（README 承认"部分操作在这么大的数据上会跑很久"）。
4. Node v24 硬约束（engines >=24 <27）对旧环境用户较苛刻，且构建期 OOM 需手动调堆（npm run setheapsize，package.json scripts）。

## 10. 总结评价

CyberChef 是"把一类日常工作图形化、并把整工具做成可离线分发的单文件"这一设计哲学的教科书实现：核心引擎（Chef/Recipe/Operation/Dish 四个类共约 1500 行）极简而表达力强，真正的重量在 506 个数据驱动、自描述、自带测试的操作插件上——这套"薄引擎 + 厚操作库 + 构建期代码生成"的组合让它以极低边际成本扩张了十年。作为 GCHQ 以 Crown Copyright/Apache-2.0 开源的项目，它在安全社区的地位已接近基础设施：CTF 选手、恶意软件分析师、渗透测试者是天然用户；想学插件式架构、数据类型总线（Dish）或 Web Worker 并行设计的人，src/core/ 下这 3000 行引擎代码是极好的精读样本。
