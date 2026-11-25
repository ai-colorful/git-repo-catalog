# pcottle/learnGitBranching 项目分析

> 星数 32000（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：一个纯前端的交互式 Git 分支可视化教学应用——用户在浏览器里敲 git 命令，旁边的 commit 树实时动画更新，配合 36 个闯关关卡和 git/Hg 双模式模拟器，是 Git 可视化教学领域的事实标杆。

| 项目 | 内容 |
|---|---|
| 名称 | pcottle/learnGitBranching（LGB） |
| 星数 | 32000（快照）/ 34233（2026-11-21 gh api 实测） |
| 语言 | JavaScript（含少量 JSX，CommonJS 风格） |
| 许可证 | MIT（LICENSE.md） |
| 创建时间 | 2012-08-13（gh api `created_at`） |
| 最近推送 | 2026-11-20（gh api `pushed_at`，即分析前一天） |
| 官网 | https://learngitbranching.js.org（CNAME，见 `CNAME` 文件） |
| 线上形态 | 100% 客户端应用，无后端、无 AJAX（README "Other Technical Details" 一节明确声明） |

## 2. 项目是什么，解决什么问题

Git 的分支、rebase、远程同步等概念在命令行里是"看不见摸不着"的，初学者难以建立心智模型。LearnGitBranching 的解法是把 Git 仓库做成一个内存中的 commit 图模拟器：用户在左侧输入 `git commit`、`git rebase -i`、`git push` 等命令，右侧的 SVG 树立刻以动画形式呈现每个命令对图结构的影响（README 开篇即陈述这一动机，配演示 GIF）。

应用有三种使用形态：
1. **沙盒模式**：默认启动，随敲随玩，支持 `undo`、`reset`、`git fakeTeamwork`（模拟队友推远程）、`git fakeCreateRemote`（模拟建远程）等教学专用假命令（README "Sandbox Mode"）；
2. **关卡模式**：输入 `levels` 进入，共 36 个关卡分 7 个世界（intro/rampup/rebase/remote/mixed/advanced/workingDir，目录 `src/levels/`），每关给出目标树结构，引擎用 `treeCompare` 判定过关，还记录命令数支持 "git golf" 最少命令挑战（README "Levels"）；
3. **Level Builder**：`build level` 命令引导式造关卡，导出 JSON 可通过 gist 分享或提 PR 收录（README "Level Builder"，实现见 `src/js/level/builder.js`）。

命令体系完全模拟真实 Git 语义：分支、相对引用（`HEAD~2`、`main^`）、detached HEAD、交互式 rebase（弹出编辑对话框）、cherry-pick、revert、远程跟踪分支、`push`/`pull`/`fetch`/`clone` 全部在 `src/js/git/index.js` 的 GitEngine 中实现。用户可通过 URL 参数分享可复现场景（`?command=...&NODEMO`），并支持 `debug_copyTree()` 导出树状态用于报 bug（README "Reporting Bugs"）。

由于零后端、零网络请求，整个应用本质上是一个"单页静态站点 + 大型状态机"，这让它能在 GitHub Pages 上免费稳定运行十余年。

## 3. 技术栈与依赖

运行时依赖（`package.json`）：

| 依赖 | 用途 |
|---|---|
| raphael ^2.1 | SVG 渲染引擎，commit 树可视化全部由它画（`src/js/visuals/`） |
| react ^17 + react-dom ^17 | 对话框与部分工具栏组件（`src/js/react_views/*.jsx`、`src/js/dialogs/`） |
| flux ^4 | Flux 架构骨架：Dispatcher/Store/Action（`src/js/dispatcher/`、`src/js/stores/`、`src/js/actions/`） |
| jquery ^3.4 + jquery-ui 1.13 | 命令行输入框、DOM 操作、拖拽等老 UI 层（`src/js/views/`） |
| underscore ^1.13 | 模板插值（intl 用 `{branchName}` 占位符，`src/js/intl/index.js` 的 `templateSettings`）与工具函数 |
| marked ^18 | 关卡开场对话框渲染 Markdown（`src/js/views/` 中 ModalAlert 使用） |
| prop-types | React 组件 props 校验 |

构建/开发链（`package.json` devDependencies）：
- **Vite ^8.3.8**：2012 年项目最初用 browserify + babelify 打包，近期已迁移到 Vite。`vite.config.mjs` 里有大量针对"CommonJS 老代码 + 循环 require"的兼容层：`@originjs/vite-plugin-commonjs`、`vite-plugin-node-polyfills`，以及自定义插件 `cjsLazyInterop` 处理 browserify 部分导出语义下的 TDZ 问题——这是老项目渐进迁移的好样本。
- **jasmine ^4.6 + nyc**：测试与覆盖率。
- **eslint ^10**：代码检查（`eslint.config.js`）。

工程约定：Node >= 20.19（`engines`），强制 yarn 1（`preinstall` 跑 `scripts/check-package-manager.js`，`packageManager: yarn@1.22.22`）。

## 4. 目录结构与代码组织

顶层（tarball HEAD 快照）：

| 目录/文件 | 职责 |
|---|---|
| `src/js/` | 全部应用代码，119 个 JS 文件约 9.4 万行 |
| `src/levels/` | 关卡定义（7 个子目录 + `index.js` + `sequenceInfo.js`） |
| `src/style/` | 手写 CSS（`main.css`、`font-awesome.css`、`rainbows.css` 等） |
| `__tests__/` | Jasmine 测试，30 个 spec 文件约 5.4 千行 |
| `scripts/` | 代码生成与工具（`generate.js`、`generateIntlStrings.js`、`generateLevelManifest.js`、`generateLevelDocs.js`、`translate.js`、`validate-locale.js` 等） |
| `index.html` | Vite 入口，内含视图模板与大量 OG meta |
| `vite.config.mjs` | 构建配置 + 自定义 CJS 兼容插件 |
| `.github/workflows/` | `pages.yml`（Pages 部署）、`build-docker.yml`（GHCR 镜像） |
| `CLAUDE.md` | 给 Claude Code 的架构导读（约 2025-2026 年加入），README/构建流程描述与之一致 |

`src/js/` 模块划分（按目录）：

| 目录 | 职责 |
|---|---|
| `git/` | Git 模拟器引擎（`index.js` 3768 行 + `commands.js` 命令注册 1185 行 + `headless.js` 无头模式 + `gitShim.js`） |
| `graph/` | 树比较器（过关判定，`treeCompare.js` 484 行）与图结构（`index.js`） |
| `visuals/` | Raphael 可视化层（visNode/visBranch/visTag/visEdge/visStagingArea + `animation/` 动画队列，共约 4.4 千行） |
| `levels/` + `level/` | 关卡数据（数据在 `src/levels/`，解析/过关/进度逻辑在 `src/js/level/`，含 builder、undoBookkeeping、disabledMap） |
| `intl/` | 国际化（`strings.js` 4474 行 213 个键 × 约 30 种语言；构建时按语言拆分生成 `generated/`） |
| `stores/`、`actions/`、`dispatcher/` | Flux 状态层（4 store / 4 action / 1 dispatcher） |
| `commands/` | 命令解析框架与 optionParser（命令行 token 解析） |
| `views/`、`react_views/`、`dialogs/` | UI 层（jQuery 老视图 + React 视图 + 对话框） |
| `models/`、`util/` | Command/Branch/Commit 模型、EventBaton、errors、mock 等（`util/` 22 个文件） |
| `mercurial/` | Mercurial 命令适配（260 行，引擎层 `mode` 切换 git/hg） |
| `sandbox/`、`log/`、`app/` | 沙盒专用命令、终端日志、应用入口 |

组织方式评述：单体单包应用，无 monorepo。数据（levels）与逻辑（js）分离；视图层是"jQuery + React + Raphael"三代 UI 技术共存的活化石；核心引擎不依赖 DOM（`headless.js` 可脱离可视化运行），这使得测试和过关判定可以直接跑纯逻辑。

## 5. 核心架构与关键实现

### 5.1 GitEngine：内存中的 Git 模拟器（`src/js/git/index.js`）

GitEngine 是全项目的心脏（3768 行）。它维护 `refs`、`HEAD`、`rootCommit`、`origin`（一个嵌套的 GitEngine 实例充当远程仓库）以及 commit/branch/tag 三个集合。所有 git 命令最终落到 `GitEngine.prototype` 上的 130+ 个方法：`commit`、`push`、`pull`（含 rebase/merge 两种 finish 路径）、`fetch`、`cloneFromOrigin`、`rebase`、`rebaseInteractive`、`merge`、`cherrypick`、`revert`、`reset`、`checkout`、`tag`、`describe`、`status`、`log` 等（`GitEngine.prototype` 方法清单见该文件 65-3261 行）。

特色设计：
- **EventBaton 命令分发**：构造器里 `this.eventBaton.stealBaton('processGitCommand', this.dispatch, this)`（`src/js/git/index.js:55`），命令以"接力棒"方式路由，hg 模式下换 Mercurial 引擎接棒；
- **模拟工作区模型**：`workingChanges = { path: 'modified' | 'staged' }`（`src/js/git/index.js:46-47`），默认不启用（`changesModelEngaged`），只有涉及 `git add`/`git status` 的 workingDir 关卡才激活，保证老的纯图关卡行为不变；
- **clone 语义重构**：`clonePending` 字段实现"真 Git 方向"的 origin→local 克隆（`src/js/git/index.js:33-38` 注释，对应测试 `__tests__/cloneSemantics.spec.js`），取代早期 local→origin 的 `makeOrigin()` 逆向模拟；
- **无头运行**：`src/js/git/headless.js` 用 mock 动画工厂实例化 GitEngine，供测试与树比较直接调用。

### 5.2 命令注册与解析（`src/js/git/commands.js` + `src/js/commands/`）

每个命令是一个声明式对象：缩写正则、全名正则、国际化描述键、选项表和 execute 函数。以 commit 为例（`src/js/git/commands.js:116-175`）：

```js
commit: {
  sc: /^(gc|git ci)($|\s)/,
  regex: /^git +commit($|\s)/,
  descriptionKey: 'command-description-commit',
  options: [
    '--amend',
    { names: ['-a', '--all'] },
    { names: ['-m', '--message'], args: 1 }
  ],
  execute: function(engine, command) { ... }
}
```

`commandConfig` 里注册了 26 个命令（commit/cherrypick/gc/pull/fakeTeamwork/clone/fakeCreateRemote/remote/fetch/branch/add/restore/reset/revert/merge/revlist/log/show/rebase/status/checkout/push/describe/tag/switch 等，`src/js/git/commands.js:115-1136`）。`src/js/commands/optionParser.js` 负责把 `git <cmd>` 后的 token 解析成 `generalArgs` + options map（支持 `--opt=value`、`-am` 聚合、`--` 分隔，见 `CLAUDE.md` 架构节）。

### 5.3 过关判定：treeCompare（`src/js/graph/treeCompare.js`）

关卡的目标树和用户树都以序列化 JSON 字符串表示（branches/commits/HEAD，必要时嵌套 originTree）。`TreeCompare.dispatch`（`src/js/graph/treeCompare.js:31`）先做浅比较，再递归比较 origin 子树，支持 `compareOnlyMain`、`compareOnlyMainHashAgnostic`（忽略 rebase 产生的 `'` 后缀 ID 差异）等宽容模式（`TreeCompare.onlyMainCompared`，15-27 行），并比较 workingChanges（staging 关卡的暂存区状态）。ID 归一化逻辑与 GitEngine 的 `rebaseAltID`/`getBumpedID`（`src/js/git/index.js:2177-2216`）配套，解决 rebase 后 commit 改名导致的假阳性。

### 5.4 可视化层：Raphael SVG + 动画队列（`src/js/visuals/`）

`GitVisuals`（`src/js/visuals/index.js`，1059 行）持有 Raphael `paper`（41 行），监听引擎事件后为每个 commit/branch/tag/edge 维护可视化对象（visNode 580 行、visBranch 679 行、visTag 442 行、visEdge 222 行、visStagingArea 484 行）。所有变更走 `AnimationQueue` 的 promise 链（`src/js/visuals/animation/`），由 `animationFactory` 把引擎操作翻译成动画序列——这也是测试里 mock 掉的部分。窗口尺寸自适应通过 `shrink/asymShrink` 挤压坐标实现（`src/js/visuals/index.js:182-197`）。

### 5.5 关卡系统（`src/levels/` + `src/js/level/`）

共 36 关（按 `solutionCommand` 字段统计）：intro 4 关、rampup 6 关、rebase 2 关、remote 16 关、mixed 5 关、advanced 1 关、workingDir 2 关。每关是一个 `exports.level = {...}` JSON-ish 对象（例：`src/levels/intro/commits.js`）：`name`/`hint`/`startDialog` 是按 locale 键控的多语言字典（首关覆盖 30 种语言），`startTree`/`goalTreeString` 是序列化树 JSON，`solutionCommand` 是标准解，`disabledMap` 可在关卡内禁用命令（如禁 `git revert` 防作弊）。序列元数据（Tab 名、about、排序）在 `src/levels/sequenceInfo.js`。`src/js/level/index.js`（741 行）负责加载、进度存储与过关流程；`parseWaterfall.js` 定义同一序列内的解锁顺序。

## 6. 构建与运行

来自 README 与 `package.json` 的实际命令：

```bash
yarn install            # preinstall 强制 yarn 1
yarn dev                # Vite dev server，http://localhost:5173（predev 先跑 scripts/generate.js）
yarn build              # yarn test && vite build && node scripts/postbuild.js，产物在 ./build
yarn test               # Jasmine 测试套件（pretest 同样先跑代码生成）
yarn test:coverage      # nyc 覆盖率
yarn lint               # ESLint 检查 src __tests__ scripts
yarn lint:strings       # 校验 intl 字符串表（src/js/intl/checkStrings）
yarn docs:levels        # 生成关卡文档到 generatedDocs/

# 生产构建先跑测试（package.json build 脚本），CI 同样以 yarn build 一步完成测试+构建

# Docker（README "Docker" / Makefile）
docker run -p 8080:80 ghcr.io/pcottle/learngitbranching
```

代码生成是强制的前置步骤：`scripts/generate.js` 依次跑 `generateLevelManifest` 与 `generateIntlStrings`，产出 `src/js/intl/generated/<locale>.js` 和动态 import 的 `loaders.js`（`scripts/generateIntlStrings.js` 头部注释明确其动机：strings.js 约 270KB 只需按需加载当前语言）。

## 7. 测试与工程质量

- **测试规模**：30 个 spec 文件、5354 行、488 个 `it()` 用例、103 个 `describe()` 组（`__tests__/` 目录 grep 统计）。覆盖面：git 引擎（`git.spec.js` 643 行）、远程操作（`remote.spec.js` 561 行）、克隆语义（`cloneSemantics.spec.js` 301 行）、命令解析（`commandParsing.spec.js` 304 行）、EventEmitter（513 行）、intl/LocaleStore/LevelStore/GlobalStateStore 等 store、mercurial 模式、动画、关卡校验（`levels.spec.js` 会遍历所有关卡验证 solutionCommand 真能过关）、生成物一致性（`generatedArtifacts.spec.js`）。
- **测试风格**：单元 + 集成混合，引擎通过 `headless.js` + mock 动画工厂脱离 DOM 运行；无浏览器端 e2e（无 Playwright/Puppeteer，未确认）。
- **CI**：`.github/workflows/pages.yml`——push main 触发 `yarn install --frozen-lockfile && yarn build`（含测试）后用官方 Pages Actions 部署；`.github/workflows/build-docker.yml`——构建 GHCR 镜像（PR 只构建不推送，main 推 latest/sha 标签）。另有 dependabot.yml。
- **质量观察**：ESLint 10 已启用；`CLAUDE.md` 为 AI 辅助开发提供完整架构导读（近期提交显示作者开始用 AI 工具维护）；代码生成 + `generatedArtifacts.spec.js` 防 drift；`lint:strings` 保证翻译键不缺漏。老代码风格为 ES5 原型链（`GitEngine.prototype.xxx`），与 2012 年起点一致，整体可读性高、注释密度大。

## 8. 社区与维护状态

- **活跃度**：gh api 实测最近一次 push 为 2026-11-20（分析前一天），近 5 天内有 7 个提交（作者本人 + Tsuruko、b84330808 等外部贡献者 + dependabot）；提交总数约 3599（commits API 末页序号）。
- **维护模式**：典型"单核长跑"项目——作者 Peter Cottle 自 2012 年 8 月建仓持续维护至今（14 年），绝大多数提交来自本人；GitHub Releases API 返回 0 条（该项目不用 Release 发版，部署即发布），版本号停在 package.json 的 0.8.0。
- **贡献者**：contributors API 分页序号约 347（含翻译提交者）；README 展示 30+ 头像并单独致谢翻译团队。翻译贡献占外部贡献相当大比例（30 种语言均为社区翻译，流程见 `TRANSLATING.md` + `scripts/translate.js`/`validate-locale.js`）。
- **issue/PR**：open issues 55（gh api），README 明确 PRs welcome。
- **部署设施**：GitHub Pages（主站 learngitbranching.js.org）+ Netlify 兜底（`netlify.toml`）+ GHCR Docker 镜像三条通道，长期可用性有冗余。

## 9. 亮点与风险

亮点：
1. **教科书级的"模拟器 + 可视化"分层**：GitEngine 纯逻辑可无头运行（`src/js/git/headless.js`），可视化层只消费事件，测试、过关判定、可视化三者解耦干净；
2. **长周期渐进现代化样本**：14 年技术栈演进全程留痕——browserify→Vite 8 迁移的兼容层（`vite.config.mjs`）、jQuery/React/Raphael 三代 UI 共存、近期新增 `CLAUDE.md` 拥抱 AI 辅助维护；
3. **国际化工程化**：213 个 UI 键 × 约 30 种语言全部社区翻译（`src/js/intl/strings.js`，最全的语言为 en_US/bn_BD/fr_FR/pt_PT/tr_TR/zh_CN/zh_TW），构建时按语言拆分动态加载，en_US 同步兜底 + 区域回退（zh_TW→zh_CN、es_MX→es_ES，`src/js/intl/index.js:16-19`），配套翻译校验脚本与测试；
4. **内容与代码同仓共生**：36 个关卡即是数据也是可测试资产（`__tests__/levels.spec.js` 验证每个 solutionCommand），Level Builder 让社区可造关卡；
5. **测试纪律**：488 个用例覆盖引擎、解析、store、生成物，生产构建前置测试。

风险/局限：
1. **巴士因子 ≈ 1**：核心引擎、关卡、发布几乎全部依赖 pcottle 一人，347 名贡献者多为翻译/小修；
2. **老栈技术债**：CommonJS + 循环 require 依赖 Vite 兼容层硬扛（`vite.config.mjs` 自定义 `cjsLazyInterop` 插件），jQuery 时代视图与 React 17 混杂，重构成本高；
3. **无版本化发布**：0 GitHub Releases、版本号长期 0.8.0，依赖方（如想复用引擎）没有稳定 API 语义，也没有 npm 包；
4. **无浏览器端 e2e**：回归主要靠引擎级单测，UI/动画层（Raphael/JQuery 交互）缺自动化保障。

## 10. 总结评价

LearnGitBranching 是"个人作品长期主义"的典范：一个 2012 年的前端玩具，凭借极准的产品定位（可视化学 Git）、克制的依赖、扎实的模拟器实现和持续的内容/语言扩充，在零后端、零融资的条件下服务全球开发者 14 年并涨到 3.4 万星。技术上它不是"现代最佳实践"的模板——ES5 原型链、CommonJS、jQuery 老栈都在——但其模拟器/可视化/关卡三层的清晰边界、可无头测试的引擎设计、以及从 browserify 到 Vite 的渐进迁移记录，对任何想维护长周期前端项目的人都是可读性极高的活教材。适合：学 Git 的初学者和教学者直接使用；想研究"如何用纯前端状态机模拟领域语义"、"老前端项目如何渐进现代化"的开发者读源码。
