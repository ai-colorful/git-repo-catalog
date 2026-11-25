# gorhill/uBlock 项目分析

> 星数 68775（快照 2026-10-24）｜主语言 JavaScript｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：uBlock Origin（uBO）是一个以 CPU/内存效率著称的广谱浏览器内容拦截扩展，通过自研的静态/动态过滤引擎在浏览器内拦截广告、追踪器、挖矿脚本与反拦截脚本。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | uBlock Origin (uBO) | `README.md` |
| 星数 | 68775（快照）/ 68946（2026-10-31 gh api 实测） | gh api repos/gorhill/uBlock |
| 主语言 | JavaScript（纯 ES Modules，无框架、无打包器） | `src/js/`、`package.json` |
| 许可证 | GPL-3.0 | `LICENSE.txt` |
| 默认分支 | master，最近 push 2026-10-30 | gh api |
| 最新 release | 1.75.0（2026-09-16），开发版号 1.75.1.104 | gh api、`dist/version` |
| 贡献者 | 约 110 人（gh api contributors 分页统计） | gh api |
| 本地化 | 71 种语言（`src/_locales/`，Crowdin 同步） | `src/_locales/`、`tools/import-crowdin.sh` |

## 2. 项目是什么，解决什么问题

uBlock Origin 是 Raymond Hill 自 2014 年起维护的浏览器扩展（`src/js/static-net-filtering.js` 版权头"Copyright (C) 2014-present Raymond Hill"），目标是高效拦截广告、追踪器、矿机脚本、弹窗与"反拦截"脚本。默认启用 EasyList、EasyPrivacy、Peter Lowe's、Online Malicious URL Blocklist 及 uBO 自有过滤清单，同时支持 hosts 文件格式（`README.md`）。它使用 EasyList 语法并对其做了大量扩展（自定义规则、脚本注入、过程式 cosmetic 过滤等，`README.md` Extended Syntax 一节）。

与 Adblock Plus 类产品的核心差异在理念与技术两层面：理念上 README 明确反对"可接受广告"模式，定位为"中和隐私侵犯手段"的工具；技术上 uBO 不满足于拦截网络请求，而是构建了四层过滤体系——静态网络过滤、动态网络过滤（按站点开关与 URL 规则）、cosmetic/脚本注入过滤、HTML/响应头过滤（分别对应 `src/js/static-net-filtering.js`、`src/js/dynamic-net-filtering.js`、`src/js/cosmetic-filtering.js`、`src/js/html-filtering.js`、`src/js/httpheader-filtering.js`）。

关于 Manifest V3 迁移的现状（截至 2026-10）：主仓库仍是 MV2 扩展。README 表格明确记录了平台时间线——Chrome Web Store 已于 2026-08-31 下架全部 MV2 扩展，Edge 从 2026-08 起启动消费者端 MV2 退役；因此 uBO 在 Chromium 系浏览器上的主渠道已收缩为 GitHub Releases 手动安装，官方推荐 Firefox（"uBO works best on Firefox"）。MV3 形态的产品是姊妹项目 uBlock Origin Lite（uBlockOrigin/uBOL-home），本仓库承担其构建链路：`platform/mv3/` 下的 `make-rulesets.js` 把过滤清单预编译为声明式 DNR 规则集，`Makefile` 提供 `mv3-chromium/mv3-firefox/mv3-edge/mv3-safari` 目标（`platform/mv3/README.md`、`Makefile`）。

## 3. 技术栈与依赖

- **语言**：纯 JavaScript（浏览器端 ES Modules + 后台页 service/page context），构建脚本为 Bash + Python（生成 manifest，`tools/make-chromium-meta.py` 等），构建要求 Node >= 22（`package.json` engines）。
- **运行时零第三方依赖**：扩展本体不引 npm 依赖，`package.json` 的 devDependencies 仅 ESLint 9 全家桶用于 lint（`package.json`）。
- **vendored 第三方库**（`src/lib/`）：CodeMirror（规则编辑器）、css-tree（cosmetic 选择器解析）、lz4（缓存压缩）、publicsuffixlist、punycode、js-beautify、regexanalyzer（正则 RE2 兼容性判定）、diff。
- **WASM**：自研 trie 的 WASM 版本 `src/js/wasm/hntrie.wasm`、`biditrie.wasm`，附 WAT 源码（`src/js/wasm/*.wat`）；加载受 CSP `'wasm-unsafe-eval'` 门控（`platform/common/vapi-background.js`）。
- **工程设施**：Makefile + shell 脚本构建；ESLint 9（含 JSON 插件）做静态检查；GitHub Actions 仅做 tag 触发的发布构建（`.github/workflows/main.yml`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `src/` | 扩展本体：manifest 模板、各 UI 页面（dashboard/popup/logger 等 html）、`js/`（92 个核心 JS 文件）、`lib/`、`_locales/`（71 语言）、`web_accessible_resources/`（52 个重定向资源） |
| `platform/` | 平台适配层：chromium/firefox/opera/thunderbird/safari/mv3/nodejs/npm/dig，`common/` 放跨平台 vAPI 抽象 |
| `tools/` | 构建脚本：各平台打包（make-chromium.sh 等）、manifest 生成（*meta.py）、清单拉取（pull-assets.sh）、Crowdin 导入 |
| `assets/` | 清单源注册表 `assets.json`（默认启用列表与更新地址） |
| `dist/` | 构建产物目录（`version` 文件当前为 1.75.1.104） |
| `docs/` | 官网页面 + 手工测试/基准页（`docs/tests/`） |
| `.github/` | 发布 workflow 与 issue 模板配置 |

组织方式评述：单仓单包（非 monorepo），以"一份共享核心 `src/` + 多平台薄适配 `platform/`"的方式支撑六个发布目标；核心代码按引擎职责一文件一模块，`src/js/` 内部用相对路径 ES Module 互相引用，构建时不打包、原样拷贝（`tools/make-chromium.sh` 复制共享文件，平台差异由 `platform/<p>/manifest.json` 与 `vapi-*` 覆盖）。uBO Lite 的 MV3 构建链与主扩展同仓但产物独立（`dist/build/uBOLite.*`）。

## 5. 核心架构与关键实现

### 5.1 过滤规则解析器：整数 AST（`src/js/static-filtering-parser.js`，4475 行）

uBO 没有使用对象树表示过滤规则，而是把每行规则解析为"整数-only 数组中的单向 AST"，节点即数字序列，整块数组在解析下一行时复用，从设计上消灭了逐行解析的 GC 压力（文件头注释原文：*"The parser creates a simple unidirectional AST from a raw line of text. Each node in the AST is a sequence of numbers, so as to avoid the need to make frequent memory allocation to represent the AST."*）。AST 类型覆盖网络规则（AST_TYPE_NETWORK）、扩展规则（cosmetic/scriptlet/html/responseheader）与预解析注释指令；同一 AST 还被复用于规则编辑器的语法着色。

### 5.2 静态网络过滤引擎（`src/js/static-net-filtering.js`，6124 行）

编译产物是大量小过滤单元类（`FilterPatternAny`、`FilterImportant`、`FilterAnchorHnLeft`、`FilterDomainHitAny` 等，每个类带静态 `compile()`），单元之间组合成过滤链；匹配语义用位域"realm"划分（`static-net-filtering.js:62-73`）：

```js
const BLOCK_REALM          = 0b0000_0000_0000_0000_0000;
const ALLOW_REALM          = 0b0000_0000_0000_0000_0001;
const IMPORTANT_REALM      = 0b0000_0000_0000_0000_0010;
const TYPE_REALM           = 0b0000_0000_0011_1110_0000;
const HEADERS_REALM        = 0b0000_0000_0100_0000_0000;
const REDIRECT_REALM       = 0b0000_0000_1000_0000_0000;
```

数据结构层面依赖两个自研容器：HNTrieContainer（主机名 trie，字符串内部反转存储以便后缀匹配子域名，全树以类型化数组整数索引实现，`src/js/hntrie.js` 头注释）与 BidiTrieContainer（双向 trie，`src/js/biditrie.js`），两者均有 WASM 实现（`src/js/wasm/`）。正则规则先经 `regex-analyzer.js` 的 `isRE2()` 判定是否可在匹配时安全使用。清单加载与编译持久化由 `src/js/storage.js` 的 `loadFilterLists()/applyCompiledFilters()` 编排（storage.js:711/645）。

### 5.3 Cosmetic 与脚本注入过滤（`src/js/cosmetic-filtering.js`、`scriptlet-filtering.js` 等）

cosmetic 引擎把选择器按主机名存入 `StaticExtFilteringHostnameDB`（`src/js/static-ext-filtering-db.js`），运行时配合每标签页的 `SelectorCacheEntry`（MRU 缓存，`src/js/mrucache.js`）避免重复查询；支持"低通用选择器"（纯 id/class）与过程式过滤器的分级处理（cosmetic-filtering.js:214 附近的分类注释）。脚本注入走 `scriptlet-filtering.js` + `src/js/resources/`（32 个注入资源文件，如 `json-prune.js`、`abort-current-script.js`），模板化打包为 `scriptlet.template.js`/`scriptlet-begin.template.js`。重定向资源为 52 个 `src/web_accessible_resources/` 文件，由 `redirect-engine.js` 统一调度。

### 5.4 动态过滤与权限模型

动态规则三层实现：按主机名的开关矩阵 `hnswitches.js`（位偏移定义 no-popups、no-cosmetic-filtering、no-scripting 等开关，hnswitches.js:26-34）、按类型的动态规则 `dynamic-net-filtering.js`（3p/inline-script/1p-script/3p-frame 等类型位）、以及临时 URL 规则 `url-net-filtering.js`（block/allow/noop）。信任体系由 `trusted-sites.js`、`trusted-tokens.js` 支撑（CHANGELOG 中大量 `trusted-*` scriptlet 依赖它）。

权限模型（`platform/chromium/manifest.json`）：MV2 manifest 请求 `webRequest` + `webRequestBlocking` + `<all_urls>` + `privacy` + `unlimitedStorage` 等十项权限——这正是 uBO 能做同步式请求改写与 HTML 过滤的根基，也是它在 MV3 下能力受限的直接原因；Firefox 版另含 `dns`、`menus`（`platform/firefox/manifest.json`），gecko id 为 `uBlock0@raymondhill.net`，最低 Firefox 115。MV3 版改为构建期把清单编译成 DNR 规则集（`platform/mv3/make-rulesets.js`、`src/js/static-dnr-filtering.js`）。

## 6. 构建与运行

来自 `Makefile` 与 `package.json` 的实际命令（未运行验证，仅静态确认）：

```bash
npm install            # 仅装 ESLint 等开发依赖
make chromium          # 构建 Chromium MV2 包 → dist/build/uBlock0.chromium
make firefox           # 构建 Firefox 包 → dist/build/uBlock0.firefox
make opera / make npm  # 其他目标
make dig && make dig-snfe   # 构建并运行 SNFE 基准 CLI（@gorhill/ubo-dig，platform/dig/）
make mv3-chromium      # 构建 uBO Lite（需 submodule 与在线拉取清单，platform/mv3/README.md）
npm run lint           # ESLint 检查 src/js 与 platform
```

## 7. 测试与工程质量

- **自动化测试**：`package.json` 的 test 脚本是 `"Error: no test specified"`——主仓库没有 CI 化的单元测试。取而代之的是 `docs/tests/` 下的手工基准/验收页（hntrie-benchmark、procedural-cosmetic-filters 等）与 `static-filtering-parser-checklist.txt` 清单，以及 `src/js/benchmarks.js`（15KB 的引擎内基准代码）。CHANGELOG 显示每次发布集中修复 parser/scriptlet 边角问题，回归主要靠清单规模与人工验收。
- **CI**：仅 `.github/workflows/main.yml`，tag 创建时跑发布构建（拉 uAssets → make-chromium/make-firefox → 生成 GitHub Release），无测试门禁。
- **质量观察**：运行时零依赖、全员 ES Module、整数 AST 与类型化数组等"内存友好"手法一以贯之；ESLint 9 覆盖 JS 与 JSON；代码注释密集且解释设计动机（hntrie、urlskip 等文件头都有长注释），可读性高。

## 8. 社区与维护状态

- **活跃度**：master 最近 push 2026-10-30（分析前一天），开发版号 1.75.1.104 领先于 2026-09-16 的 1.75.0 release，发布节奏约每月一个 minor 版本（CHANGELOG.md 从 1.75.0、1.74.0 向上连续排布）。gh api 实测 open_issues 仅 15——issue 实际由 uBlockOrigin/uBlock-issues 仓库承载（README 链接可见）。
- **贡献者结构**：约 110 名贡献者（gh api contributors 分页 Link 头），但架构与核心引擎长期由 Raymond Hill 主导（package.json author、各文件版权头），另有 ryanbr 等清单侧贡献者出现在 CHANGELOG。
- **本地化**：71 语言，经 `tools/import-crowdin.sh` 从 Crowdin 同步（README 顶部徽章）。
- **发布渠道**：GitHub Releases 提供 Firefox/Chromium MV2/Thunderbird 手动安装包；Chrome/Edge 商店渠道已随 MV2 退役政策终止（README 安装表格）。

## 9. 亮点与风险

**亮点**
1. 自研高性能数据结构：typed-array 主机名 trie + 双向 trie + WASM 加速（`src/js/hntrie.js`、`src/js/biditrie.js`、`src/js/wasm/`），在数十万条规则规模下保持低内存低延迟。
2. 整数 AST 解译管线（`src/js/static-filtering-parser.js`）：解析、编译、语法高亮共用一棵零分配 AST，是该领域少见的工程化设计。
3. 四层过滤体系（静态网络/动态规则/cosmetic+scriptlet/HTML+响应头）覆盖 MV2 能力上限，且带 trusted-sites 信任模型。
4. 平台适配层轻而有效：一份核心 + `platform/` 薄壳 + vAPI 抽象（`platform/common/vapi.js` 归一 chrome/browser），支撑六平台与独立 MV3 产物。
5. 运行时零依赖、71 语言、发布全自动化，工程纪律极强。

**风险/局限**
1. Chromium 生态断供：Chrome 已于 2026-08-31 下架 MV2 扩展，Edge 2026 年底前完成退役（README），主扩展在这些平台只能手动加载或转向功能受限的 uBO Lite。
2. 无自动化单元测试与 CI 测试门禁，回归依赖手工基准页与清单实际表现（`package.json`、`docs/tests/`）。
3. 核心架构高度绑定单一维护者，巴士因子偏低（版权头与 author 字段佐证）。
4. vendored 第三方库（csstree、codemirror 等）需手工跟升级，存在滞留风险（`src/lib/`）。

## 10. 总结评价

uBlock Origin 是浏览器内容拦截领域工程质量最高的开源实现：它把"广告拦截"从字符串匹配做成了包含整数 AST 编译器、类型化数组/WASM 索引结构、多级规则语义和信任模型的完整引擎，同时保持了零运行时依赖与极高的内存效率。其代价是绑定 MV2 平台能力——随着 2026 年 Chromium 系完成 MV2 退役，主扩展的适用面收缩到 Firefox 一侧，MV3 化以功能受限的姊妹项目 uBO Lite 承接。适合需要理解过滤器编译/匹配管线、浏览器扩展权限边界或高性能纯 JS 数据结构的开发者精读源码；普通用户在 Firefox 上它仍是首选拦截器。
