# SheetJS/sheetjs 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-17

> **重要说明（本仓特殊状态）**：GitHub 上 `SheetJS/sheetjs` 的默认分支为 `github`，该分支是 2024 年"撤码"后的纯指引桩（tarball 仅 12KB，无源码）。本文档的源码分析全部基于官方保留的 `master` 分支完整源码快照（33MB tarball，冻结于 2022-10-04 提交 `515d1c6f`，即撤码前最后一个提交），分支事实经 `gh api` 与 README 双重核实。这一特殊状态本身正是本项目最值得记录的历史事件，详见第 8 章。

## 1. 项目速览

一句话定位：SheetJS 社区版（SheetJS CE）是 JavaScript 生态覆盖面最广的电子表格读写工具包——用一个零依赖的单文件库在浏览器/Node/Deno 中解析和生成 XLSX/XLSB/XLS/ODS/CSV 等二十余种电子表格格式。

| 事实 | 值 | 出处 |
|---|---|---|
| 名称 | SheetJS/sheetjs（npm 包名 `xlsx`） | `package.json` |
| 星数 | 39500（清单快照 2026-10-24）；API 实测 36341（2026-11-17） | gh api |
| 主语言 | JavaScript（含 Flow 类型注释；GitHub language 字段现为 null，因默认分支已无代码） | `bits/*.js` |
| 许可证 | Apache-2.0 | `LICENSE`（master 分支与默认分支均含） |
| 源码主战场 | https://git.sheetjs.com/sheetjs/sheetjs（自建 Gitea） | 默认分支 `README.md` |
| master 冻结时间 | 2022-10-04（提交 `515d1c6f` "mini refresh"，对应 v0.18.12） | gh api commits |
| 仓库最后 push | 2024-04-18 | gh api pushed_at |
| 版本发布渠道 | cdn.sheetjs.com（GitHub tags 止于 v0.18.5） | gh api tags、`README.md` |

## 2. 项目是什么，解决什么问题

电子表格数据交换是 Web 应用最常见的"脏活"之一：后端报表要导出 Excel，前台要粘贴/上传 Excel，数据分析要把 JSON 变成表格文件。SheetJS CE 把这件事抽象成一个统一的数据模型（CSF，Common Spreadsheet Format）和一对入口函数 `XLSX.read()` / `XLSX.write()`，让调用方完全不感知底层格式差异——无论输入是 XLSX、Excel 2003 XML、Lotus 1-2-3 还是纯 CSV，`read()` 都会吐出同一个 `Workbook` 对象（`README.md`、`bits/87_read.js`）。

它的核心价值在"格式覆盖广度"：`bits/88_write.js` 中的 bookType 分发表列出了 25 种可写出的 bookType，从现代的 xlsx/xlsm/xlsb/ods/numbers，到考古级的 biff2/3/4/5/8（xls）、wk1/wk3（Lotus 1-2-3）、dbf、dif、sylk、prn、eth、xlml（Excel 2003 XML）等。其中 Apple Numbers 的 IWA 格式（protobuf 压缩流）读写在开源 JS 生态几乎独一无二（`bits/83_numbers.js`，1910 行）。

第二个价值是"运行环境广度"：单文件 `xlsx.js`（26,447 行，911KB）零运行时依赖，`package.json` 中 `"dependencies": {}`，CFB/ZIP/码表/SSF 格式化引擎全部内联在 `bits/` 里（依赖包 cfb、codepage、fflate 只是构建期 devDependency）。同一份构建产物可在浏览器、Node、Deno、Bun、ExtendScript（Photoshop 脚本）甚至老 IE 上运行（`dist/xlsx.extendscript.js`、`demos/` 下 29 个框架集成示例）。

SheetJS 以 CE（社区版）+ Pro（商业版）双轨运营：数据解析/生成免费，样式编辑、图片/图表/PivotTable、公式求值移植等能力属 Pro（`README.md` 商业版段落）。这种"开源引擎 + 商业增值"模式支撑了一个约 200+ 贡献者规模的项目长期专职维护。

## 3. 技术栈与依赖

| 层 | 选择 | 证据 |
|---|---|---|
| 语言 | ES5 级 JavaScript + Flow 类型注释（`/*:: ... */`） | `bits/*.js`、`.flowconfig`（已在撤码时删除） |
| 运行时依赖 | 无（`"dependencies": {}`，全部自包含） | `package.json` |
| 构建期依赖 | cfb ~1.2.2、codepage ~1.15.0、fflate ^0.7.1、crc-32、adler-32、@sheetjs/uglify-js 等 | `package.json` devDependencies |
| 类型定义 | 手写 `types/index.d.ts`（自带 Workbook/CellObject/写入选项全量类型），dtslint 校验 | `types/index.d.ts`、`package.json` scripts |
| 双模块发布 | CJS `xlsx.js` + ESM `xlsx.mjs`，package.json `exports` 全子路径映射 | `package.json` |
| 构建/测试 | Makefile（无打包器，靠 `cat` 按编号拼接）+ eslint + closure compiler | `Makefile` |
| CI | GitHub Actions：node ≥4 版本矩阵、bun、deno、node 0.x、iojs 共 8 个 workflow | `.github/workflows/` |

## 4. 目录结构与代码组织

| 目录 | 职责 | 规模证据 |
|---|---|---|
| `bits/` | 库源码本体：84 个按 00-99 编号的模块，构建时按序 `cat` 拼接 | 29,482 行合计 |
| `dist/` | 预编译产物：core/full/mini 三档 min.js + sourcemap + cpexcel 码表 + zahl 本地化 | 16 个文件，约 5.6MB |
| `modules/` | 较新模块的 TypeScript 源（83_numbers 等），esbuild 编译后回填 bits | 7 组 .js/.ts 对 |
| `packages/` | 附属子包：ssf（ECMA-376 格式码引擎）、xlsx-cli、ssf-cli、s、otorp | `packages/*/package.json` |
| `demos/` | 29 个框架/环境集成示例（react/vue/angular/electron/deno/headless/oldie 等） | `demos/` |
| `tests/` | 浏览器端 mocha 页面 + fixture 清单（core.js/fixtures.js 为生成物） | `tests/fixtures.lst` |
| `test_files/` | 测试语料占位目录——**主仓内为空**，真实语料在独立仓 SheetJS/test_files | git tree API：0 个条目 |
| `types/` | TypeScript 类型定义与 dtslint 测试 | `types/index.d.ts` |
| `misc/` | 构建清单（mini.lst/mjs.lst）、Flow 头尾、deno 配置 | `misc/` |
| `bin/` | CLI 入口 `xlsx.njs` | `package.json` bin 字段 |

组织方式评述：这是典型的"**编号拼接单体**"架构——不用 webpack/rollup，`Makefile` 用 `cat bits/*.js | tr -d '\15\32'` 把 84 个 IIFE 化的模块拼成一个大文件再剥 Flow 注释（`Makefile` 第 34-40 行）。模块编号即依赖顺序（00 头 → 10 格式化 → 18 容器 → 20-60 解析层 → 70-80 读写层 → 87/88 入口 → 95 API → 99 尾）。这种反现代的极简构建换来的是对任意环境的零摩擦移植，是项目"全端可跑"卖点的技术根基。

## 5. 核心架构与关键实现

### 5.1 统一读入口：魔数嗅探（bits/87_read.js）

`readSync` 不看扩展名，只看前 8 字节魔数决定分发路径，任何格式最终都归一到 `Workbook`：

```javascript
// bits/87_read.js
case 0xD0: if(n[1] === 0xCF && n[2] === 0x11 && n[3] === 0xE0 /* OLE2 CFB */)
    return read_cfb(CFB.read(d, o), o); break;      // XLS/XLSB/加密包
case 0x09: if(n[1] <= 0x08) return parse_xlscfb(d, o); break;  // 裸 BIFF
case 0x50: return (n[1] === 0x4B && n[2] < 0x09 && n[3] < 0x09)
    ? read_zip(d, o) : read_prn(data, d, o, str);  // PK ZIP → xlsx/xlsb/ods/numbers
case 0x03: case 0x83: case 0x8B: case 0x8C: return DBF.to_workbook(d, o);
case 0x7B: if(n[1] === 0x5C && n[2] === 0x72 && n[3] === 0x74) return rtf_to_workbook(d, o); break;
case 0x0A: case 0x0D: case 0x20: return read_plaintext_raw(d, o); // CSV/PRN/SYLK 兜底
```

兜底策略层层递进：OLE2 容器（`bits/18_cfb.js` 1871 行自研 CFB 解析）、ZIP 容器（`bits/85_parsezip.js`）、XML/SST、二进制 BIFF 记录表（`bits/77_parsetab.js` 1334 行生成码表）、最后是纯文本猜测（PRN/CSV，`bits/40_harb.js` 同时承担 Lotus/Quattro 等 legacy 解析）。开头遇到 PNG/TIFF 等图片魔数还会给出专门的错误提示（87_read.js）。

### 5.2 数据模型：CSF（Common Spreadsheet Format）

CSF 是全库的"枢纽语言"：`Workbook = { SheetNames: [], Sheets: {}, Props }`，每个 Sheet 是以 A1 风格地址为键的稀疏字典，单元格对象含 `v`(原始值)/`w`(格式化文本)/`t`(类型 b/n/e/s/d/z)/`f`(公式) 等字段（`types/index.d.ts` 第 633-655 行 CellObject 定义）。地址编解码、列字母/行号互转、Range 解析全部集中在 `bits/27_csfutils.js`（`decode_cell`/`encode_cell`/`decode_range` 等纯函数）。每个格式解析器的职责就是"格式 → CSF"，写出器则是"CSF → 格式"，两侧互不依赖——这是能用一套代码覆盖 25 种格式的架构原因。

### 5.3 写出引擎：bookType 分发（bits/88_write.js）

写侧按 bookType 一把分发，容器层只分两类（ZIP 或 CFB），其余为字符串生成器：

```javascript
// bits/88_write.js 第 13-17 行
switch(opts.bookType) {
    case "ods": return write_ods(wb, opts);
    case "numbers": return write_numbers_iwa(wb, opts);
    case "xlsb": return write_zip_xlsb(wb, opts);
    default: return write_zip_xlsx(wb, opts);
}
```

第 133-163 行的第二级 switch 覆盖全部 bookType：xlsx/xlsm/xlam/xlsb/numbers/ods 走 ZIP（`bits/86_writezip.js`），xls/biff2-8 走 CFB（`bits/78_writebiff.js`），csv/txt/html/sylk/dif/prn/rtf/eth/dbf/wk1/wk3/fods/xlml 都是单表字符串/字节生成器。写出 CSV 时自动补 BOM（`\ufeff`），`o.password` 时切换到加密 ZIP 输出（88_write.js denouement 段）。

### 5.4 Apple NUMBERS IWA 读写（bits/83_numbers.js）

1910 行，是 bits 中最大的手写模块。Numbers 文件是 ZIP 包裹的 IWA（iWork Archive，Snappy 压缩的 protobuf 流），该模块实现了 IWA 反解、TSP Reference 对象图解析与回写（含 `.numbers`/`.key`/`.pages` 的探测，提交历史中"parse icloud.com numbers exports"等）。开源生态中能读写 Numbers 的 JS 库仅此一家，说明其对"任意复杂电子表格"定位的认真程度。

### 5.5 工具层与 API 面（bits/90_utils.js、bits/95_api.js）

`XLSX.utils` 导出 30+ 个工具：`aoa_to_sheet`/`json_to_sheet`/`table_to_sheet`（DOM 表格抓取）及其逆向 `sheet_to_json`/`sheet_to_csv`/`sheet_to_html`/`sheet_to_formulae`，外加 `book_append_sheet`/`sheet_set_sheet_visibility` 等簿操作。`sheet_to_json`（90_utils.js 第 46 行）支持 header/raw/defval/range/blankrows 等约 15 个选项，是使用频率最高的 API。整库对外 API 面收敛在 `bits/95_api.js` 的 `utils` 表和 `bits/98_exports.js`（20 行）里，边界清晰。

## 6. 构建与运行

来自 `package.json` scripts 与 `Makefile`（均在 master 快照中验证）：

```bash
npm install                 # 安装构建期依赖（零运行时依赖）
git submodule init && git submodule update   # pretest-only 会拉子模块
npm run build               # make —— 拼接 bits/ 生成 xlsx.js / xlsx.mjs / xlsx.mini.js
npm test                    # pretest(lint) + tests-only(make travis)
npm run lint                # make fullint = eslint + mdlint
npm run dtslint             # types/index.d.ts 类型回归
make lint                   # 仅 eslint（含 HTML/JSON 扩展检查，见 Makefile 第 205 行）
node bin/xlsx.njs            # CLI 直接读取/转换表格文件
```

注意：`make travis` 实际执行依赖 `test_files/` 语料，主仓内为空，需配合独立仓 `SheetJS/test_files`（`README.md` Related Projects 段）——在本仓快照中无法直接跑通完整测试，属于已知限制。

## 7. 测试与工程质量

- **测试规模**：主套件 `test.js` 2,913 行（Node 端）/ `test.mjs` 2,901 行（ESM 端），Makefile 由其生成浏览器端 mocha 套件 `tests/core.js` 与 `tests/fixtures.js`（`.gitattributes` 将两者标记为 generated）。fixture 清单 `tests/fixtures.lst` 229 个语料文件，覆盖 1904 纪元、biff5、ods、slk、xls、xlsb、xlsx、xml 的成对回环测试样例（同一工作簿 7 种格式并存，见 fixtures.lst 首段）。
- **CI 矩阵（撤码前）**：8 个 workflow（`.github/workflows/`）——`node-4+.yml` 用 ljharb 的动态矩阵测 node 主版本 ≥4，且**每个版本配不同时区**（node14=New_York、node13=London 等，用于时区回归）；`node-zero.yml` 测 node 0.x；`bun.yml`/`deno.yml` 测新引擎；`require-allow-edits.yml`/`rebase.yml` 是 ljharb 社区治理风格的标准件。README 另有 SauceLabs 浏览器矩阵徽章。
- **质量观察**：全量 eslint + markdown lint + dtslint 三层静态检查；`bits/` 内代码风格高度统一（var 声明、IIFE、手工内联依赖）；`modules/` 下的 TS 迁移（esbuild 0.14.14）显示官方在向类型化源码演进但未完成。整体是"测试密度远高于代码风格现代性"的老派高可靠工程。

## 8. 社区与维护状态（含 2023 撤码风波考证）

### 8.1 API 可验证的客观时间线

以下每一行均来自 `gh api repos/SheetJS/sheetjs` 系列调用或下载的分支文件，不含推测：

| 时间 | 事件 | 证据 |
|---|---|---|
| 2012-12-03 | 建仓 | gh api created_at |
| 2013-2022 | GitHub 活跃开发，标签 v0.13.5 → v0.18.5 | gh api tags 列表 |
| 2022-10-04 | master 最后一个提交 `515d1c6f` "mini refresh"（package.json 版本 0.18.12）；此后 GitHub 上再无代码提交 | gh api commits?sha=master |
| 2023-01-02 | 默认分支路径上提交 `859befbc` "🏠 New Home"——官方首次以提交形式宣告项目迁往自建 Gitea | gh api commits?sha=github |
| 2023-01-11 | `5b4806bf` "💼 We're Hiring!" | 同上 |
| 2023 年春（精确日期**未确认**） | 公开撤码风波：官方宣布离开 GitHub，源码一度从公开渠道移除、社区分发只保留编译产物（tarball/脚本），引发大规模社区反弹 | 此阶段的中间状态因历史被恢复而无法从 API 直接复现，日期与操作细节标注为未确认；间接佐证见 8.2 |
| 风波后（日期未确认） | 恢复：master 分支被完整恢复至 `515d1c6f`；桩 README 明文承认该事实 | 默认分支 `README.md`："The `master` branch … includes all commits through `515d1c6f`" |
| 2024-02-01 | 提交 `3f44ddd9` "exit stage left"：一次性删除 300 个文件、2,615,835 行，默认分支从此只剩 README/LICENSE/package.json/demos 存根 | gh api commit stats（additions 23 / deletions 2,615,835） |
| 2024-04-18 | 仓库最后一次 push，此后除 star 外无任何仓库活动 | gh api pushed_at |
| 2026-11（分析时） | issues 仍开放（133 个 open）但官方指引到 Gitea 提交；PR 已名存实亡（历史 PR 约 300+ 条全部来自撤码前）；events 流全部为用户 WatchEvent，每日仍有新 star | gh api issues/pulls/events |

### 8.2 对风波的客观定性

可确证的部分：(1) GitHub 侧代码冻结于 2022-10-04，比公开风波早了约半年，说明"迁出"是渐进决策而非突然事件；(2) 官方恢复后明确保留了完整 git 历史（master 未被改写，`515d1c6f` 有正常父提交链），并在 README 中承诺 CE 永远 Apache-2.0（"SheetJS CE remains _truly open source_ under the Apache 2.0 License"，默认分支 README.md 第 7 行）；(3) 最终态是"GitHub = 指引墓碑 + 冻结快照，git.sheetjs.com = 开发主场，cdn.sheetjs.com = 唯一下发渠道"。
不能确证的部分：2023 年风波的精确日期、撤码-恢复的具体操作序列、社区 fork（如各类 SheetsJS fork）的传播规模——这些需要新闻/HN/网络检索，超出本任务"仅 gh api"的网络边界，标注**未确认**。
版本渠道影响（可确证）：GitHub tags 止于 v0.18.5，而 master 快照版本为 0.18.12——0.18.12 及之后的 0.19/0.20 系列从未走 GitHub tag 发布，官方渠道只有 cdn.sheetjs.com（README "Downloadable Scripts and Modules"）。

### 8.3 维护节奏结论

以 GitHub 为坐标：2022-10 后源码零提交、2024-02 后仓库零实质变更，GitHub 视角是"**已停止维护的只读镜像**"。项目本体是否仍在 git.sheetjs.com 活跃，本任务无法核实（外部站点不在网络白名单），未确认。贡献者结构：contributors 分页尾部为第 201 页（per_page=1），即约 201+ 人，且几乎全部贡献发生在 2022-10 之前（gh api contributors）。

## 9. 亮点与风险

**亮点**
1. 格式覆盖无出其右：25 种 bookType 写出 + 魔数嗅探统一读入口，含 Lotus 1-2-3/Quattro/Multiplan/DBF/DIF/SYLK 等考古格式（`bits/88_write.js`、`bits/87_read.js`）。
2. 零依赖单文件、全端可跑：`"dependencies": {}`（package.json），从老 IE 到 Deno/Bun/ExtendScript 一份产物通吃，29 个官方框架示例（`demos/`）。
3. CSF 数据模型设计干净：格式解析器与工具层通过 A1 键值稀疏表解耦，`sheet_to_json` 一类 API 上手即用（`bits/27_csfutils.js`、`bits/90_utils.js`）。
4. 测试密度极高：2,900 行主套件 × 229 fixture × node 0.x-最新版本矩阵 × 多时区 × SauceLabs 浏览器（第 7 章所列文件）。
5. 开源罕见的 Apple Numbers IWA 读写实现（`bits/83_numbers.js`）。

**风险**
1. GitHub 主仓是墓碑：默认分支无源码、master 冻结于 2022-10、npm `xlsx` 停更（GitHub tag 视角止于 0.18.5）——按 GitHub 习惯性 `npm i xlsx` 引入的用户拿到的代码已落后官方分发渠道多个版本（后一断言基于 cdn.sheetjs.com 存在性，版本差细节未确认）。
2. 供应链与审计面变化：唯一正式渠道是 cdn.sheetjs.com 单点下发，脚本安装方式对 CI 供应链审计不友好；自建 Gitea 的账号体系与 GitHub 打通方式需使用者自行评估（默认分支 README.md）。
3. 社区分裂：GitHub issues 133 个 open 无人响应（官方指引去 Gitea），PR 流程废弃，基于本仓开发的第三方 fork 与官方 CE 的关系需使用者甄别。
4. 社区版功能边界明确：样式、图片/图表/PivotTable、公式求值移植均属 Pro 闭源部分（README.md Pro 段），需求触碰边界时迁移成本高。

## 10. 总结评价

SheetJS/sheetjs 是 JS 电子表格处理事实上的标准库：以反潮流的"编号拼接单体 + 零依赖"架构换来了无可匹敌的格式广度与全环境兼容性，以 200+ 贡献者、九位数级行删除记录和一场罕见的公开撤码风波，完整演绎了一个头部开源项目"开源引擎 + 商业闭环"的生存实验。它至今仍是处理 Excel 数据的第一推荐，但**引用姿势必须改**：不要再依赖 GitHub master 或 npm 镜像的最新版心智，而应视本仓为 2022-10 冻结的历史快照与指引入口，实际使用与 Issue 跟踪去 git.sheetjs.com 与 cdn.sheetjs.com。适合：需要稳定解析/生成表格数据的全栈工程师（直接用其官方渠道产物）；适合读源码的人：想学"如何用一个数据模型吃掉二十多种文件格式"的库作者（`bits/27_csfutils.js` + `bits/87_read.js` + `bits/88_write.js` 三件套是最佳教材），以及想研究开源治理/平台依赖风险的社区研究者（第 8 章时间线即案例样本）。
