# adam-p/markdown-here 项目分析

> 星数 60252（快照 2026-10-24）｜主语言 JavaScript｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：Markdown Here 是一个浏览器/邮件客户端扩展，让你在 Gmail、Outlook、Yahoo 等富文本编辑器里用 Markdown 写邮件，点击"Markdown Toggle"一键渲染成漂亮 HTML 再发送，且可随时还原。

| 项目 | 值 | 出处 |
|---|---|---|
| 名称 | adam-p/markdown-here | GitHub API |
| 星数 | 60,262（分析时点 API 值） | GitHub API |
| 主语言 | JavaScript（纯 JS，无框架、无打包工具链） | `src/common/` |
| 许可证 | MIT（代码）；Logo/图片另有 CC BY 3.0 约定 | `LICENSE`、`README.md` Credits 节 |
| 当前版本 | v2.16.0（2025-07-10） | `src/common/CHANGES.md`、GitHub Releases |
| 最近活跃 | 最近代码提交 2025-07-10，仓库 push 至 2025-08-22 | GitHub API commits/pushed_at |
| 平台 | Chrome / Firefox / Opera（WebExtension MV3）+ Thunderbird | `src/manifest.json`、`README.md` |
| 贡献者 | 约 14 人（含匿名，GitHub API contributors） | GitHub API |

## 2. 项目是什么，解决什么问题

写带代码、表格、公式的邮件一直很痛苦：富文本编辑器排版能力弱，纯 Markdown 又不能直接发。作者在 README 中自述动机：自己曾在 GitHub 网页编辑器里写 Markdown、预览后再复制粘贴进邮件，觉得这个流程太荒谬，于是做了这个"在邮件编辑框里直接写 Markdown 并就地渲染"的工具（`README.md` 第 12 段）。

使用方式是典型的"开关式"交互：在富文本编辑框（contenteditable）里写 Markdown，然后右键菜单、工具栏按钮或快捷键（默认 Shift+Alt+M）触发"Markdown Toggle"，编辑框内容被替换为渲染后的 HTML；再触发一次则从渲染态还原为原始 Markdown（`README.md` Usage 节，`src/common/markdown-here.js` 的 render/unrender 逻辑）。支持选中局部转换、回复时自动跳过引用块（blockquote）、自动排除 `-- ` 签名分隔线之后的内容。

能力上支持 GitHub Flavored Markdown、fenced code block 指定语言后的语法高亮（highlight.js）、可选的 TeX 数学公式（默认关闭，经 CodeCogs 图片服务渲染，`src/common/options-store.js` DEFAULTS 与 `src/common/CHANGES.md` v2.15.0）。除网页邮箱外，兼容 Google Groups、Blogger、Evernote、WordPress 等一切有富文本编辑器的站点（`README.md` 兼容性脚注）。

项目 2012 年创建，是"在邮箱里写 Markdown"这一品类的开创者，60k+ 星主要来自其十年间极高的实用口碑——它解决的问题极普遍，且零依赖、装完即用。

## 3. 技术栈与依赖

- **语言**：纯 JavaScript（ES5 风格为主 + 少量 ES6+，如 async/await），无 TypeScript，无前端框架（`src/common/*.js`）。
- **扩展标准**：WebExtension Manifest V3（`src/manifest.json` `"manifest_version": 3`），同时以 background page + service worker 双形态兼容 Firefox/Chrome 差异。
- **Vendored 第三方库**（直接打包进仓库，无 npm 运行时依赖）：
  - `src/common/marked.js` — Markdown→HTML 解析器（2011-2013 年的 chjj/marked 老版本 fork，约 1286 行）；
  - `src/common/highlightjs/highlight.js` — highlight.js v11.11.1 单文件打包（61,722 行）+ 79 个主题 CSS；
  - `src/common/vendor/dompurify.min.js` — HTML 消毒，渲染结果插入 DOM 前必经（`src/common/utils.js` `safelyParseHTML`）；
  - `src/common/jsHtmlToText.js` — HTML→纯文本（用于还原/检测 Markdown）。
- **构建依赖**（仅开发用，`utils/package.json`）：`archiver`（打 zip/xpi）、`file`、`glob`，共 3 个包——运行时依赖数为 0。
- **测试栈**：浏览器内运行的 Mocha + Chai（vendored 于 `src/common/test/`）。
- **代码规范**：根目录 `.eslintrc.json`（较大规则集），无 CI 强制执行。
- **样式与国际化**：用户可编辑的默认渲染样式在 `src/common/default.css`（202 行，刻意规避 `-moz-`/`-webkit-` 与 `a:hover` 等邮件环境无效写法，见 README Styling 节）；13 语言包由 manifest `default_locale: "en"` 驱动。

## 4. 目录结构与代码组织

顶层（tarball HEAD，master 分支）：

| 目录/文件 | 职责 |
|---|---|
| `src/` | 扩展本体（浏览器"加载已解压扩展"直接指向此目录，`README.md` Manual/Development 节） |
| `src/manifest.json` | MV3 清单，v2.16.0，含 Firefox 专属 `browser_specific_settings` |
| `src/chrome/` | Chrome 侧胶水层：`backgroundscript.js`（后台）、`contentscript.js`（页面注入入口）、升级提示弹窗 |
| `src/common/` | 平台无关核心：渲染管线、toggle 逻辑、选项存储、CSS、图片、vendored 库、测试 |
| `src/_locales/` | 13 种语言包（de/en/es/fr/it/ja/ko/pl/pt_BR/ru/tr/zh_CN/zh_TW） |
| `utils/` | Node 构建脚本 `build.js`（打三个平台的包）+ `Makefile` 入口 |
| `test/` | 顶层测试入口页 `index.html`（contenteditable iframe 环境） |
| `store-assets/` | 应用商店截图/描述/评审备注 |
| `contributors/` | 活跃贡献者小传（markdown 文件） |
| `.github/` | 仅 `FUNDING.yml`（GitHub Sponsors + PayPal），**无 CI/工作流** |

组织方式评述：这是典型的"单包多目标"扩展仓库——一份 `src/common/` 跨平台共享，平台差异被压到极薄的 `chrome/` 层；每个 JS 模块既是浏览器全局脚本又能 `module.exports`（如 `markdown-render.js` 尾部导出判断），从而同一份代码可在 content script、background、Mocha 三种环境运行。无打包器、无转译，vendored 库与手写代码混排，构建只是"按平台过滤文件打 zip"。

## 5. 核心架构与关键实现

### 5.1 双模式后台与按需注入（MV3 适配核心）

`src/manifest.json` 同时声明 `"background": {"page": ..., "service_worker": ...}`，由 `src/chrome/backgroundscript.js` 用 `location.hash` 区分两种加载形态：background page 下依赖脚本已随页面加载；service worker 下用 `importScripts` 现场引入 marked/hljs/DOMPurify 等，并注明 SW"30 秒无活动即销毁，不能依赖全局变量存状态"（文件头注释，第 18-45 行）。

```json
"background": {
  "page": "chrome/background.html#background-page",
  "service_worker": "chrome/backgroundscript.js"
},
"permissions": ["activeTab", "contextMenus", "storage", "scripting"],
"optional_host_permissions": ["http://*/*", "https://*/*"]
```
（出处：`src/manifest.json` 第 15-20 行，`#background-page` hash 即运行时区分两种形态的开关。）

权限模型是 2025 年 v2.16.0 的最大变化（`src/common/CHANGES.md`）：从"安装即拥有所有网站权限"改为 `activeTab + optional_host_permissions` + `chrome.scripting` 按需注入（`src/chrome/backgroundscript.js` 的 `Injector`，第 209-263 行）：点击按钮/菜单时先写入 `window.markdownHereInjected` 标记防重复，再按序注入 8 个脚本；对已授权域名则利用 `tabs.onUpdated` 预注入以支持"忘记渲染"提醒。

### 5.2 渲染请求管线（刻意把重代码留在后台）

content script 只做薄壳：`src/chrome/contentscript.js` 收到 button-click 后校验焦点元素是否 `contenteditable`（`markdownHere.elementCanBeRendered`），然后调用通用入口 `markdownHere()`；真正渲染通过消息把"Markdown-in-HTML 文本"发给后台，由后台用完整 marked+hljs+用户 CSS 渲染后回传（`requestMarkdownConversion`，第 57-68 行）。`src/common/markdown-render.js` 头部注释点明设计意图：把渲染代码与 content script 分离，是为了"最小化每个页面都要加载的代码量"。渲染函数本身纯函数化，自定义 heading/link renderer 实现标题锚点与自动补 `https://`，用 `langPrefix: 'hljs language-'` 把 hljs 类名混入 code 块（第 75-96 行）。

### 5.3 渲染/还原的"包装器"机制（项目最巧妙的持久化技巧）

`src/common/markdown-here.js`（自称"the heart of Markdown Here"）解决了"如何把原始 Markdown 存回网页 DOM 以便还原"的难题：很多站点会剥属性，实测发现 `div` 的 `title` 属性能被 Gmail 等保留（文件头注释，第 12-35 行）。渲染时把原始 HTML base64 后藏进一个零尺寸 div 的 `title="MDH:..."`，外面套 `markdown-here-wrapper`（`renderMarkdown`，约第 468-500 行）；还原时反查该 div、base64 解码、替换回去（`unrenderMarkdown`）。核心代码：

```js
rawHolder = '<div ' +
            'title="' + WRAPPER_TITLE_PREFIX + Utils.utf8StringToBase64(originalHtml) + '" ' +
            'style="height:0;width:0;max-height:0;max-width:0;overflow:hidden;..."' +
            '>&#8203;</div>';
mdHtml = '<div class="markdown-here-wrapper" data-md-url="...">' +
         mdHtml + rawHolder + '</div>';
```
（出处：`src/common/markdown-here.js` `renderMarkdown`，节选自约第 478-489 行，`&#8203;` 零宽空格是防 Thunderbird 丢弃空 div 的补丁。）

同时用 `MutationObserver` 监测渲染后内容是否被用户手改（标记 `markdown-here-wrapper-content-modified`），防止还原时静默丢失改动。因网页邮箱会剥离 `<style>` 块，`makeStylesExplicit`（第 264-319 行）把 CSS 规则逐条解析后以内联 style 显式写到每个匹配元素上——这是"发出去对方看到的样式一致"的关键。还原入口 `unrenderMarkdown` 还专门处理 Thunderbird 把长 `title` 折行导致的 base64 破损（剥除全部空白后再解码）。

### 5.4 反向转换与"忘记渲染"检查

发送前需要把 HTML 转回纯文本交给 marked 判断/渲染，`src/common/mdh-html-to-text.js`（381 行）+ `src/common/jsHtmlToText.js`（270 行）承担 HTML→text，含"仅检测是否含 Markdown"的保守模式。"忘记渲染"检查（`src/common/common-logic.js` 第 33-130 行起）按站点定制发送按钮选择器（Gmail：`[role="button"][tabindex="1"][aria-label][data-tooltip]`；Fastmail：`[class~="s-send"]`），拦截点击/空格/Ctrl+Enter 四种发送路径，检测未渲染 Markdown 时弹确认框；content script 以 2 秒间隔轮询挂钩（`src/chrome/contentscript.js` 第 113 行）。

## 6. 构建与运行

- **开发运行（免构建）**：浏览器开发者模式直接"加载已解压的扩展"指向仓库 `src/` 目录（Chrome/Opera）；Firefox 按 MDN WebExtension 临时加载流程（`README.md` Installation Instructions）。
- **构建发布包**：
  ```bash
  cd utils && npm install
  node build.js        # 或仓库根目录 make build
  ```
  产出 `dist/chrome.zip`、`dist/firefox.zip`、`dist/thunderbird.xpi`（`utils/build.js` 第 18-21 行）；Chrome 包会正则剔除 manifest 中 Firefox 专属字段（`addBuildFile`，第 61-71 行）。
- **包内清单**：Chrome/Firefox 包只含 `manifest.json`、`common/`、`chrome/`、`_locales/` 四项，`common/test/` 被构建脚本排除（`utils/build.js` skipFileRegexes，第 31-33 行；`README.md` Building 节）。
- **注意事项**：Makefile 的 `build` 目标会先 `clean`（删除 `dist/`）再构建；仓库无 lint/test 的 npm script——`utils/package.json` 甚至没有 scripts 字段，工程化程度有意保持最低。
- **运行测试**：项目为浏览器内测试——加载扩展后在 `src/common/test/index.html` 打开 Mocha 页面运行（`src/common/test/test-run.js`）；无 headless/命令行测试命令（未确认有替代入口）。

## 7. 测试与工程质量

- **测试规模**：`src/common/test/` 下 6 个测试文件（common-logic、markdown-here、markdown-render、mdh-html-to-text、options-store、utils-test，合计约 1600 行），框架 Mocha+Chai 连同 underscore 全部 vendored；另有顶层 `test/index.html` 提供 contenteditable iframe 手动测试环境。
- **CI**：无。`.github/` 下仅有 FUNDING.yml，无 workflows——测试靠本地手动跑，发布前靠 AMO 人工审核兜底。
- **代码质量观察**：根目录有大而全的 `.eslintrc.json`；注释密度高、几乎每个模块头都有设计动机说明（如 wrapper 持久化的实验记录、Firefox spacebar 键的怪癖），并大量链接对应 issue 编号（#85、#297、#427 等），可追溯性极好。短板：无 CI/无类型检查；vendored 的 marked 是 2013 年老版本（`sanitize: false` 传入，安全完全依赖 DOMPurify 后置消毒，`src/common/markdown-render.js` 第 79 行 + `src/common/utils.js` `safelyParseHTML`）。

## 8. 社区与维护状态

- **活跃度**：2012-05 创建，2018-09（v2.13.4）到 2024-10（v2.14.0）间发布断档约六年；2024-10 起明显复苏，连续发布 v2.14.0（MV3 支持）→ v2.15.0/2.15.1（2025-06，hljs 11、LaTeX 默认关、权限收紧铺垫）→ v2.16.0（2025-07-10，最小权限模型），最后 push 2025-08-22（`src/common/CHANGES.md`、GitHub API）。
- **维护结构**：近乎单人维护（Adam Pritchard 主导，最近 3 个提交均为本人；contributor 名单约 14 人含匿名），`contributors/` 目录为 5 位活跃贡献者立传。issue 积压 319 个 open（GitHub API）。
- **发布节奏**：Chrome 商店即时更新；Firefox/Thunderbird 走 AMO 人工审核，README 明言"最长可能滞后一个月"。
- **资金**：GitHub Sponsors + PayPal（`.github/FUNDING.yml`），CHANGES 提到 12 年约 75 笔捐赠。
- **贡献制度**：仓库带 `CONTRIBUTING.md`（3809 字节）与个人/实体两份 CLA（`CLA-individual.md`、`CLA-entity.md`）——对这种规模的 OSS 项目较少见，作者借此保持对发行版权的控制；另有 `PRIVACY.md` 说明 TeX 公式经第三方图片服务的隐私影响（与 v2.15.0 默认关闭 LaTeX 的决定呼应）。

## 9. 亮点与风险

**亮点**
1. MV3 迁移做得完整且克制：background page/service worker 双形态 + `activeTab`/`optional_host_permissions` 按需注入，把"读取所有网站数据"的安装警告彻底消灭（`src/manifest.json`、`src/common/CHANGES.md` v2.16.0）。
2. `title` 属性藏 base64 原文的渲染/还原机制，是对"不可控第三方 DOM"环境的优雅持久化方案，注释里保留了完整的实验与失败案例记录（`src/common/markdown-here.js` 头部）。
3. 架构分层清晰：通用核心（common）与平台胶水（chrome/）严格分离，同一模块可跑在 content script/background/Mocha 三种环境。
4. 内联化 CSS（`makeStylesExplicit`）解决了邮件客户端剥样式表的死结，保证收件人看到的效果一致。
5. 13 语言本地化 + highlight.js 79 主题全量打包，开箱体验完整。

**风险/局限**
1. 维护高度单点：近六年断档后又转为一年一次发布，issue 积压 319；若作者再停更，MV3 后续 API 变更（如 Chrome 对 background page 支持的进一步收紧）将无人跟进。
2. vendored marked 停在 2013 年版本，`sanitize: false`，安全链完全押在 DOMPurify 上；上游 marked 十余年的修复与改进均未受益。
3. Thunderbird 构建已过期：`utils/build.js` 第 13 行 `TODO: Update Thunderbird build`，其打包清单仍指向已不存在的 `chrome.manifest`/`install.rdf`/`firefox/` 目录，实际 Thunderbird 分发依赖 AMO 上的旧渠道。
4. 按需注入模型带来体验退化：工具栏按钮常亮、热键改由浏览器管理且默认重置为 Shift+Alt+M、"忘记渲染"需重新授权（CHANGES v2.16.0 自述），部分老用户迁移成本高。

## 10. 总结评价

Markdown Here 是一个"问题小而普遍、方案稳而巧妙"的教科书级浏览器扩展：零构建工具链、零运行时 npm 依赖，用约 3000 行手写 JS 加三个 vendored 库，在"你无法控制的网页 DOM"这个最恶劣的工程环境里，做出了渲染/还原、样式内联、最小权限注入、防误发检查这一整套可靠机制。2024-2025 年的 MV3 与权限模型改造说明老项目仍能跟上平台演进。它适合所有重度邮件写作者安装使用，更适合作为 WebExtension 开发的必读源码——尤其是如何在 MV3 按需注入、跨浏览器后台差异、富文本 DOM 持久化三个经典难题上做出工程取舍；想给它贡献代码的人则要先接受"无 CI、单人决策、浏览器手动测试"的老派协作方式。
