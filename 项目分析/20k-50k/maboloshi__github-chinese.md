# maboloshi/github-chinese 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-17

## 1. 项目速览

一句话定位：把 GitHub 界面全面中文化的油猴（Tampermonkey/Violentmonkey）用户脚本，通过"本地词库 + DOM 遍历替换"让全球最大代码平台的 UI 变成中文。

| 项 | 值 |
|---|---|
| 名称 | maboloshi/github-chinese（GitHub 中文化插件） |
| 星数 | 38679（gh api 实查 2026-11-17；任务快照 39500） |
| 语言 | JavaScript（gh api 确认） |
| 许可证 | GPL-3.0（`LICENSE` 为 GPLv3 全文，`package.json` 与脚本头 `@license GPL-3.0` 一致） |
| 默认分支 | gh-pages（发布分支即默认分支，无 GitHub Release，发布走分支文件 + GreasyFork） |
| 最近活跃 | 最近 push 2026-11-16（gh api），最新词库版本 1.9.4.6-2026-11-15 |
| 规模 | 主脚本 1565 行 + 词库 3.1 万行/2.1MB；实测约 4.9 万条静态词条、4200 余条正则规则 |
| 前身 | 52cik/github-hans（楼教主），maboloshi（沙漠之子）自 2021 年接手重写 |

## 2. 项目是什么，解决什么问题

GitHub 官方本地化长期只覆盖文档，界面始终是英文。本项目以用户脚本形式注入 `github.com`、`gist.github.com`、`education.github.com`、`skills.github.com`、`www.githubstatus.com` 等站点（`main.user.js` 的 `@match` 规则），把菜单、按钮、表单、提示、页面标题等 UI 文案替换为中文，另附仓库描述的人机翻译按钮和相对时间元素中文化。

它脱胎于楼教主 2016 年的 github-hans（`locals.js` 版权头保留 "Copyright (C) 2016-2021 楼教主"），maboloshi 自 2021 年接手后完成了根本性重写：从"全局一把梭的字符串替换"演进为"按页面类型精细化词块 + 双机制翻译引擎"的架构。词库（`locals.js`）与引擎（`main.user.js`）分离——主脚本通过 `@require` 引用 gh-pages 分支上的词库文件，词库可以由 GitHub Actions 每周自动更新发布，而无需动主脚本版本。

这个项目的本质难点不在翻译本身，而在与 GitHub 前端架构的持续追更：GitHub 的页面加载从 jQuery pjax（2022 年前）迁移到 Turbo（2022），又在新版仓库页引入 React 19 SSR 水合（2026），每次迁移都会让旧的动态监测方式失效。仓库内 `docs/github-react.md` 是一份完整的实测报告，记录了 React 水合机制对 DOM 注入脚本的影响与对策——这在国内社区属罕见的深度逆向分析文档。README 更新日志里 v1.9.4.1 → v1.9.4.5 连续五个热修版本全部是适配 GitHub React 改版，直观体现了这种"猫鼠游戏"的密度。

## 3. 技术栈与依赖

- **主语言**：纯 JavaScript（ES2019+），无运行时框架，零第三方运行时依赖。词条引擎是手写 IIFE（`main.user.js`），词库是纯数据对象。
- **用户脚本 API**：`GM_addStyle`、`GM_xmlhttpRequest`（跨域调用讯飞翻译 API）、`GM_getValue/GM_setValue`（功能开关与未命中词条存储）、`GM_registerMenuCommand`（脚本菜单）。
- **构建/开发依赖**：仅 devDependencies——esbuild ^0.28.2 与 typescript ^7.0.2（`package.json`），用于 VS Code 扩展子工程与测试辅助；发布产物 `main.user.js` 直接随仓库提供。
- **测试**：Node.js 原生测试运行器（`node --test test/`，`package.json` scripts.test）。
- **CI/自动化**：GitHub Actions 7 条 workflow（词条更新同步、术语检查、词条去重、AI 代码审查、VS Code 扩展构建、贡献者头像更新），Python/Perl 辅助脚本（`script/ai_review.py`、`script/rd.py`、`script/t2s-convert.pl` 简繁转换）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `main.user.js` | 主脚本引擎（GitHub 源开发版），全部翻译管线逻辑 |
| `locals.js` | zh-CN 词库 + 页面匹配/忽略规则（`I18N` 对象，2.1MB） |
| `locals_zh-TW.js` | 繁体中文词库，由 CI 从简体自动转换（`script/t2s-convert.pl` + `t2s_rules.conf`） |
| `main(greasyfork).user.js` | GreasyFork 稳定版（版本线 1.9.2.4，`@run-at document-end`，词库走 GreasyFork 托管） |
| `main(nju.edu).user.js` | 南京大学镜像源版（国内直连加速） |
| `main_zh-TW.user.js` | 繁体版主脚本 |
| `vscode-extension/` | VS Code 扩展（TypeScript），通过 `enabledApiProposals:["browser"]` 向集成浏览器注入同一套脚本 |
| `test/` | Node 原生测试，20 个 `.cjs` 文件共 2628 行 |
| `tools/perf/` | 性能基准台（真实浏览器 + 登录态，开发用） |
| `script/` | CI 辅助脚本：术语检查（`check-terms.mjs`）、变体分析（`analyze-variants.mjs`）、AI 审查、去重、简繁转换 |
| `docs/` | 术语表 `terminology.md`、React 渲染机制实测 `github-react.md`、AI 审查说明、术语数据 `terms.json`/`terms-baseline.json` |
| `.github/workflows/` | 7 条自动化 workflow（每周词库发布、术语检查、去重等） |

组织方式评述：单仓库双产物（引擎 + 词库数据）+ 多渠道分发的扁平结构，没有 monorepo 工具链。工程化的重心不在构建，而在"词库数据的自动化运维"——简繁同步、每周定时发版、术语一致性守门全部由 CI 承担，这在用户脚本项目中相当少见。

## 5. 核心架构与关键实现

### 5.1 翻译管线：页面配置 + 双机制查询（`main.user.js`）

主脚本按"页面类型"组织翻译。`detectPageType()`（main.user.js:582）依据 `I18N.conf.rePagePath` 等路由正则（locals.js:48-54）识别当前页面属于哪个词块，`buildPageConfig()`（main.user.js:470-503）只把该页需要的词块、正则、忽略选择器展开成一个 `pageConfig`，同类型页面内跳转（如仓库间翻页）直接复用，不必重新展开词库——这是 v1.9.4.6 针对词库 2.1MB 体积做的"按页面按需构建"优化。

查询入口 `fetchTransResult()`（main.user.js:1146-1193）实现静态词典与正则的双机制：

```js
// 静态词典查找
const staticResult = pageConfig.staticDict[text];
if (typeof staticResult === 'string') {
    result = staticResult;
} else if (State.featureSet.enable_RegExp) {
    // 正则规则查找：先用「必然出现的字面量」跳过不可能命中的规则
    for (let index = 0; index < regexpRules.length; index += 1) {
        const required = regexpPrefilters?.[index];
        if (required && !text.includes(required)) continue;
        const [pattern, replacement] = regexpRules[index];
        const translated = text.replace(pattern, replacement);
        ...
    }
}
```

静态词典精确匹配覆盖绝大多数 UI 文案（实测去重后约 4.9 万条）；正则规则（约 4254 条）处理带变量的动态文案（"X stars today"、"on <date>" 等）。命中的译文写入单页缓存 `transCache`（Map，上限 5000 条，main.user.js:1077），重复文本零开销。

亮点是 `getRequiredLiteral()`（main.user.js:1098）：静态词典未命中时，从每条正则的 pattern 里静态提取"匹配成功则文本中必然出现的 ≥4 字母连续字面量"，运行时用 `text.includes()` 先跳过不可能命中的正则，把几千条正则的尝试成本压到接近 O(1)。带 `i` 标志、分支、断言、可选量词的 pattern 保守放弃预过滤，保证正确性。

DOM 遍历用 TreeWalker（`traverseNode()`，main.user.js:848-892）：元素节点按标签分派处理属性（INPUT/TEXTAREA 的 placeholder/aria-label/value、BUTTON 的 data-confirm 系列、`tooltipped` 类的 aria-label 等，main.user.js:909-966），文本节点走 `transText()`。用户内容（diff 代码、README 正文、面包屑里的账号/仓库名）通过按页面的 `ignoreSelectorPage`/`ignoreMutationSelectorPage` 规则和 `isBreadcrumbIdentifierLink()`（main.user.js:825，issue #824）拒绝翻。

### 5.2 动态页面应对：Turbo + React 水合的三层监听（`main.user.js` + `docs/github-react.md`）

GitHub 已是"SSR HTML + 客户端 React 19"混合渲染 + Turbo 软导航，脚本为此布置了三层感知：

1. **Turbo 事件**（`setupTurboEvents()`，main.user.js:289-307）：`turbo:before-render` 时在新 body 插入文档前预挂 MutationObserver，让 React 水合"把译文写回英文"的那批变更从第一帧就被接住；`turbo:load` 触发页面配置更新。bfcache 恢复不再全量重翻（缓存快照已是译文），只安排 `requestIdleCallback` 空闲补扫（`scheduleIdleTraversal()`，main.user.js:314）。
2. **MutationObserver 批处理**：突变记录先攒进 `pendingMutations`，一帧一次（requestAnimationFrame）合并处理（`flushMutationBatch()`，main.user.js:678），祖先-后代去重后只遍历顶层节点（`processMutations()`，main.user.js:785）；处理完调 `takeRecords()` 丢弃脚本自身写入引起的回流突变，避免"翻译自己的译文"造成死循环。
3. **React 提交感知**（`watchReactCommits()`，main.user.js:353-366）：新版仓库页由 React 19 水合，在 SSR 树上插"翻译"按钮会触发 hydration mismatch、按钮被丢弃。实测发现 GitHub `react-core` 每次提交后打 `react-root:commit:<app>` performance mark，脚本用 `PerformanceObserver` 监听该 mark，在提交完成后才挂按钮。`docs/github-react.md` 用对照实验表完整记录了这一机制的验证过程（含失效预案：若 GitHub 改掉埋点名，先查 `performance.getEntriesByType('mark')`）。

### 5.3 词库结构：284 个页面词块的惰性定义（`locals.js`）

`I18N.conf`（locals.js:25-346）是规则区：路由正则 `rePagePath`/`rePagePathRepo`/`rePagePathOrg`、启用字符数据监测的 `characterDataPage`（27 页）、忽略突变/遍历元素的 CSS 选择器表（26 页 ignoreSelectorPage）等。`I18N["zh-CN"]` 按 `defineI18nPage(词块名, 工厂函数)` 惰性定义（首次访问才展开，配合按页构建），`defineI18nAlias` 建立路径别名（如 `dashboard`/`feed`/`orgs/dashboard` 共享 `page-dashboard`）。实测（node 加载 locals.js 去重统计）：284 个词块键、页面级静态词条 48337 条 + 公共 1128 条、正则规则 4232 条 + 公共 22 条、标题静态词条 166 条。

### 5.4 词库质量运维闭环（`script/` + `docs/` + CI）

- `MissedTermsManager`（main.user.js:1323-1440）：开发者模式记录所有"词典未命中的文本"，按 pathname 分桶存 GM storage，可导出 JSON、查看统计——用户侧自动沉淀漏翻样本，直接变成词库 PR 的素材。
- `script/check-terms.mjs`：以 `docs/terms.json` 为术语单一数据源校验 zh-CN 词库一致性，`terms-baseline.json` 只拦新增不一致不追溯存量。
- CI（`.github/workflows/`）：每周日自动更新 main.user.js 词库版本号并同步繁体/南大镜像，周四同步 GreasyFork 稳定版；另有词条去重、DeepSeek AI 代码审查、术语检查三条 PR 门禁。

## 6. 构建与运行

无构建步骤，发布产物随仓库提供（`package.json` description 明确说明）。

```bash
# 安装：浏览器装 Tampermonkey 后，从 README 三个安装源任选其一
#   GitHub 源（开发版，每周五词库自动更新）
#   https://github.com/maboloshi/github-chinese/raw/gh-pages/main.user.js
#   GreasyFork 源（稳定版，每周一同步）/ 南大镜像源

# 测试（Node 原生 test runner，无额外依赖）
npm test            # 即 node --test test/

# 本地调试词库：Tampermonkey 开启"允许访问本地文件"，把 @require 改为
# @require file:///D:/github-chinese/locals.js   （README.md "本地调试"节）

# 术语一致性检查
node script/check-terms.mjs [--all | --update-baseline | --dict other.js]
```

VS Code 扩展：`vscode-extension/` 内 `npm run compile`（tsc）、`vsce package` 打包，要求 VS Code ^1.116.0。

## 7. 测试与工程质量

- **测试规模**：20 个 `.cjs` 测试文件共 2628 行（`test/`），Node 原生测试框架，无浏览器端到端。覆盖面针对性强：页面类型检测、观察器批处理与祖先去重、正则预过滤字面量提取、翻译缓存一致性、词典惰性加载、导航历史（popstate/bfcache）、时间元素格式、面包屑标识符跳过、输入框默认值保护、两个高频词块（two-factor、org-settings）的词典回归等。
- **CI**：7 条 workflow（`.github/workflows/`），除发版自动化外还有"词条去重"（`remove_dup.yml`）、"术语一致性检查"（`terms-check.yml`）、DeepSeek AI 代码审查（`ai-review.yml` + `docs/ai-review.md`）三条 PR 质量门禁。
- **代码质量观察**：主脚本注释密度极高且大多是"为什么"而非"是什么"（如记录 #774 重构中"看起来走不到但不能删"的历史，main.user.js:920-924）；关键函数统一用 `safe()` 错误边界包裹，任何异常不阻断页面；XSS 防护有明确痕迹（翻译结果容器用 `textContent` 赋值，main.user.js:1262-1263，v1.9.4 修复 #692）。`tools/perf/` 提供真实浏览器性能基准台。

## 8. 社区与维护状态

- **活跃度**：极活跃。gh api 实查最近 push 2026-11-16；近 10 条提交全部在 2026-11-14 至 11-16 之间，多为词库增补与 React 适配修复。README 更新日志显示 2026 年 5-11 月密集发布 5 个兼容性热修版本（React GlobalNav 引入的连锁反应）。
- **贡献者**：≥51 人（gh api contributors 首页计数），核心为作者 maboloshi（沙漠之子）+ 协作者 wyc-26、TC999，原作者 52cik 仍在贡献者名单。贡献者墙由 GitHub Action 自动更新（README:421）。
- **发版节奏**：无 GitHub Release（latest release 404）；版本 = 引擎版本 + 词库日期（如 1.9.4.6-2026-11-15），开发版每周五、GreasyFork 稳定版每周一由 CI 自动发布。开发版与稳定版引擎版本暂有分叉（1.9.4.6 vs 1.9.2.4），稳定版词库仍每周同步。
- **渠道**：GreasyFork 安装量徽章（README:15）、HelloGitHub 推荐（README:17）、南大镜像分发。README 特别警告项目从未发布至 GitCode，提示仿冒风险。

## 9. 亮点与风险

**亮点**

1. 工程化程度远超一般用户脚本：词库/引擎分离、按页惰性词块、每周 CI 自动发版、术语数据源守门、未命中词条自动沉淀（MissedTermsManager）形成"发现问题→修复→发布"的完整闭环。
2. 对 GitHub 前端演进的追踪是硬核逆向级：`docs/github-react.md` 用对照实验还原 React 19 水合机制，并以 performance mark 作为注入时机的失效预案，可复用性强。
3. 性能工程细致：正则必现字面量预过滤（`getRequiredLiteral`）、单页翻译缓存、突变一帧合并 + 自写入回流丢弃、祖先去重、bfcache 免重翻、空闲补扫，层层削减 2.1MB 词库和全量 DOM 遍历的开销。
4. 测试贴合真实缺陷：2628 行测试几乎每个文件对应一个真实 issue（#727、#824、#811 等），而非凑覆盖率。
5. 多渠道分发与简繁双语、VS Code 集成浏览器扩展，覆盖国内用户全部使用场景。

**风险/局限**

1. 强耦合 GitHub 非公开前端实现（`react-root:commit:*` 埋点名、`octolytics` meta、Primer CSS 类名模块），GitHub 任一改版即局部失效，维护是永久性负担——维护者停更即项目进入衰退。
2. 词库 2.1MB 通过 `@require` 每次页面加载拉取（有缓存），首次安装与弱网环境启动开销明显；按需构建缓解了运行时成本但下载体积仍在。
3. 正则替换 UI 文案天然有误翻边界：依赖忽略规则保护用户内容，规则外的场景（如 README 正文、代码块）可能出现误伤，历史 issue 已多次出现此类回归。
4. 仓库描述远程翻译依赖讯飞听见非官方 API（`@connect fanyi.iflyrec.com`），接口变更或限流即失效；GreasyFork 稳定版引擎版本落后开发版（1.9.2.4 vs 1.9.4.6），渠道间体验不一致。

## 10. 总结评价

这是中文社区维护最成功的 GitHub 汉化方案，也是把"用户脚本"做出工业级工程质量的罕见样本：4.9 万词条、4200 余条正则规则支撑的翻译覆盖面，配上周自动发版、术语守门、漏翻自动沉淀的运维体系，以及针对 Turbo/React 迁移的深度适配文档。它的星数（3.9 万）反映的是刚需——GitHub 不做界面汉化，中文开发者只能靠社区。适合所有中文 GitHub 重度用户安装使用；对于想研究 SPA 动态内容国际化、MutationObserver/PerformanceObserver 高级用法、或维护同类 DOM 注入型工具的开发者，`main.user.js` 加 `docs/github-react.md` 是一份可直接精读的实战教材。
