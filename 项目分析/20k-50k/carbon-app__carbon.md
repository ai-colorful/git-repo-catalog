# carbon-app/carbon 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-17
> gh api 复核（2026-11-17）：stars 36145、forks 1973、MIT、默认分支 main、最近 push 2026-11-05

## 1. 项目速览

一句话定位：Carbon 是"代码截图美化工具"的开山之作——粘贴代码，调节主题、背景、字体、窗口控件，一键导出可发布到 Twitter/博客的精美代码图片（carbon.now.sh）。

| 项目 | 事实 |
| --- | --- |
| 名称 | carbon-app/carbon |
| 主页 | https://carbon.now.sh（Vercel 托管，`vercel.json` alias） |
| 星数 | 36145（2026-11-17 gh api 实测；任务快照 39500） |
| 语言 | JavaScript（React 17 + Next.js 12，无 TypeScript） |
| 许可证 | MIT（`LICENSE`，Copyright (c) 2022 Carbon） |
| 版本 | 4.9.10（`package.json`；最后一次 release 也是 2022-06-13 的 4.9.10） |
| 形态 | 纯 Web 应用 + PWA（`next-pwa`）；**HEAD 已无 Electron 桌面目录**，README 改为引导 Chrome "Install Carbon..." PWA 离线安装（`README.md` "Installing Carbon for Desktop (Offline)" 一节） |
| 贡献者 | gh api contributors 分页拉满 100+（README all-contributors 徽章标注 160） |
| 最近活跃 | 2026-10 月有 CI 修复与 dependabot 批量升级，2026-11-05 仍有 push |

## 2. 项目是什么，解决什么问题

Twitter/技术博客上大量流传的"带 macOS 窗口、语法高亮、渐变背景"的代码截图，多数就是 Carbon 生成的。项目 2017 年由 Mike Fix（@mfix22）、Brian Dennis、Jake Dexheimer 三人创建（`README.md` Authors 节），动机很直白：代码本身精彩，但截图太丑，于是把"排版 + 高亮 + 窗口装饰 + 背景美学"打包成一个网页工具（`README.md` Introduction 节）。

用户在左侧编辑器粘贴或拖入代码文件，右侧即时渲染成一张"假窗口"图：可调语法主题、字体、窗口主题（mac 三色圆点/直角/none）、内边距、阴影、背景（纯色/渐变/Unsplash 随机图，`components/RandomImage.js`、`components/BackgroundSelect.js`），还支持对选中片段单独加粗、倾斜、下划线、变色（`components/Carbon.js` 的 `onSelectionChange` + `components/SelectionEditor.js`）。导出支持 PNG/SVG/JPEG、1x/2x/4x 缩放、方形图，可复制到剪贴板或直接上传 Imgur（`components/ExportMenu.js`、`components/CopyMenu.js`、`lib/api.js`）。

除下载图片外，Carbon 的另一条腿是"分享链接与嵌入"：整份编辑器状态（代码 + 全部配置）被编码进 URL query（`lib/routing.js` 的 `getRouteState`），登录用户还可存为云端 snippet（Firebase 认证 + `/api/snippets` 后端），分享链接在 Twitter/Slack 中会自动 unfurl 成图片（`pages/api/image/[id].js` 专门为 Twitterbot/Slackbot 服务）；`carbon.now.sh/embed` 提供 iframe 嵌入（`pages/embed/`、`components/CopyMenu.js` 生成 iframe/Medium 编码 URL）。

生态位：它定义了这类工具的交互范式（窗口控件、主题面板、双栏预览），催生了大量编辑器插件生态——VS Code/IntelliJ/Vim/Emacs/Sublime 的 carbon-now-sh 插件、`carbon-now-cli` 等（`README.md` Community 节列出 15+ 社区工具）。同类产品 ray.so（Raycast 出品）、polacode、snipaste 等都是这个赛道的后来者或并行者。

## 3. 技术栈与依赖

- **框架**：Next.js 12 + React 17（`package.json`：`next ^12.1.6`、`react ^17.0.2`；`scripts.dev = next`），Pages Router，部署到 Vercel
- **PWA**：`next-pwa ^5.5.2`（`next.config.js` 中 `withOffline`，仅生产构建启用），离线可用呼应 README 的"桌面离线安装"
- **代码编辑器**：CodeMirror 5.65.5 + `react-codemirror2`；自定义 mode 补充 elixir、solidity、graphql、nim、riscv、apache（`lib/custom/modes/`）
- **语言自动检测**：`highlight.js ^10.7.2`（仅注册 javascript，`components/Carbon.js` 顶部 `hljs.registerLanguage`，用 `highlightAuto` 检测）
- **出图（客户端）**：vendor 进仓库的 dom-to-image 2.6.0（`lib/dom-to-image.js`，778 行，源码取自 tsayen/dom-to-image），SVG foreignObject 栅格化方案
- **出图（服务端）**：`puppeteer-core ^9.0.0` + `chrome-aws-lambda ^8.0.2`，无头 Chrome 打开 carbon.now.sh 页面后同样调 dom-to-image（`pages/api/image/[id].js`）
- **后端服务**：Firebase 8（认证 + snippet 存取经 `/api/snippets`），自研 API 客户端 axios 封装（`lib/api.js`、`lib/client.js`）
- **UI 辅助**：`react-color`（颜色选择）、`react-image-crop`（背景图裁剪）、`downshift`（下拉）、`dropperx`（拖放）、`actionsack`（复制/键盘 hooks）、`morphmorph`
- **测试**：Cypress 9 e2e + `@applitools/eyes-cypress` 视觉测试；ESLint 8（eslint-config-next + jsx-a11y 插件）、Prettier、husky + lint-staged

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `components/` | 40 余个 React 组件：编辑器主体（`Editor.js`/`EditorContainer.js`/`Carbon.js`）、配置面板（`Settings.js`/`Presets.js`/`ThemeSelect.js`/`FontSelect.js`/`ColorPicker.js`）、菜单（`ExportMenu`/`CopyMenu`/`ShareMenu`）、上下文（`AuthContext`/`ApiContext`）、`Themes/`（主题创建） |
| `lib/` | 纯逻辑层：`constants.js`（1195 行，主题/字体/语言/默认配置的单一事实来源）、`dom-to-image.js`（vendored 出图库）、`api.js`（后端 API 客户端）、`routing.js`（URL 状态编解码）、`util.js`、`custom/modes/`（CodeMirror 自定义语言） |
| `pages/` | Next.js 页面：`index.js`（主编辑器）、`[id].js`（snippet 页）、`embed/`（iframe 嵌入）、`about.js`、`account.js`、`api/`（image/[id].js 出图、oembed.js） |
| `public/` | `manifest.json`（PWA）、`static/themes/`（主题截图）、`static/presets/`（预设缩略图）、品牌 SVG |
| `cypress/` | 7 个 e2e spec（basic、embed、gist、localStorage、security、background-color、visual-testing） |
| `docs/` | 24 份 README 翻译（含简繁中文、日韩、西语等） |
| `.github/` | `workflows/validate.yml`（CI）、issue/PR 模板、dependabot 配置 |
| `bin/deploy.sh`、`release.js` | 部署/发布辅助脚本 |

组织评述：典型的 Next.js 单体应用，`components/` + `lib/` 两层分离视图与逻辑；`lib/constants.js` 把全部主题（id/name/highlights 十余种 token 色）、字体、语言、`DEFAULT_SETTINGS` 集中声明，是全站配置的单点。无 TypeScript、无状态管理库（直接 setState + Context + localStorage/URL 同步），整体是"2017-2021 年代 React 经典写法"的完好样本。

## 5. 核心架构与关键实现

### 5.1 三层状态与 URL 可分享设计（components/Editor.js、lib/routing.js）

`Editor` 组件是状态枢纽：`state = {...DEFAULT_SETTINGS, ...this.props.snippet}`，挂载时按"localStorage → URL queryState"顺序覆盖（`Editor.js` 的 `componentDidMount`）。任何配置变化经 750ms 防抖 `sync()` 上抛给 `EditorContainer` 持久化。这使每个 Carbon 页面都是自描述 URL——分享链接即完整快照，是"零后端分享"的核心设计。

### 5.2 客户端渲染导出：DOM → SVG → PNG（components/Carbon.js + lib/dom-to-image.js）

编辑器本身就是图：`Carbon.js` 渲染 `#export-container`，内含 `WindowControls`（mac 窗口圆点）、CodeMirror 编辑器、水印 SVG、三层背景（白底/透明棋盘格/用户背景），并用 styled-jsx 内联样式按配置拼装 padding/max-width 等。导出时 `Editor.getCarbonImage()` 对该 DOM 节点调 dom-to-image，按 `exportSize` 缩放、用 `filter` 剔除 `eliminateOnRender` 与光标节点：

```js
// components/Editor.js getCarbonImage()
const config = {
  style: { transform: `scale(${exportSize})`, transformOrigin: 'top left', ... },
  filter: n => {
    if (n.className) {
      const className = String(n.className)
      if (className.includes('eliminateOnRender')) return false
      if (className.includes('CodeMirror-cursors')) return false
    }
    return true
  },
  width, height,
}
```

SVG 路径还做了精细后处理：替换 `&nbsp;`、剔除未使用的 `@font-face`（`Editor.js` 135-157 行），控制文件体积。这套"页面即画布 + foreignObject 栅格化"是全项目最核心的技巧——不依赖 canvas 重绘高亮，直接复用浏览器排版结果。

### 5.3 服务端 og:image：为 Twitterbot/Slackbot 生成 unfurl 图（pages/api/image/[id].js）

社交平台爬虫不执行 JS 交互，Carbon 起了专用的 Vercel Serverless 路由：请求校验 User-Agent（放行 Twitterbot/Slackbot/Slack-ImgProxy，或带 origin/authorization），然后用无头 Chrome 打开对应 carbon.now.sh 页面，注入 unpkg 上的 dom-to-image，等待 `.export-container` 可见后转 PNG 直接回图：

```js
// pages/api/image/[id].js
await page.goto(`https://carbon.now.sh/${id ? id : `?${queryString}`}`)
await page.addScriptTag({ url: DOM_TO_IMAGE_URL })
await page.waitForSelector('.export-container', { visible: true, timeout: 9500 })
const dataUrl = await page.evaluate(() => domtoimage.toPng(target, config))
```

`vercel.json` 将该函数 `maxDuration` 设为 10 秒，且用 `chrome.font()` 预载 Noto Color Emoji 保证 emoji 渲染。客户端与客户端共用同一个 dom-to-image 逻辑，是"Puppeteer 只做驱动、渲染逻辑不复制"的范例。

### 5.4 配置体系与富文本选中编辑（lib/constants.js + components/Carbon.js）

42 项"主题"实际由 13 个等宽字体（Anonymous Pro 到 Ubuntu Mono）+ 29 个语法主题（seti 为默认）组成，每个主题声明十余种高亮 token 色（`lib/constants.js` THEMES 数组），经 `toHash` 建索引；81 种语言含 Auto 档（highlight.js 自动检测）。特色交互：用户选中代码片段后弹出 `SelectionEditor`，用 CodeMirror 的 `doc.markText` 给片段打上内联 CSS（加粗/斜体/波浪线/任意色），实现"图内局部强调"：

```js
// components/Carbon.js onSelectionChange()
this.props.editorRef.current.editor.doc.markText(
  this.state.selectionAt.from,
  this.state.selectionAt.to,
  { css }
)
```

### 5.5 账号与 snippet 云存储（components/AuthContext.js + lib/api.js）

Firebase Auth 提供 GitHub OAuth 登录（`lib/client.js` `loginGitHub`），JWT 自动注入 axios 默认头；snippet 的增删改查全部防抖（更新 1s、创建 5s leading），实现"编辑即自动保存"。未登录用户则一切落在 localStorage + URL（`cypress/integration/localStorage.spec.js` 有对应 e2e）。

## 6. 构建与运行

来自 `package.json` scripts 与 `README.md`/`.github/CONTRIBUTING.md`：

```bash
yarn                 # 安装依赖（Node >= 12）
npm run dev          # next 开发服务器（本地 carbon.now.sh 预览）
npm run build        # next build 生产构建
npm run start        # 生产服务器
npm run lint         # next lint
npm run cy:open      # Cypress 交互式 e2e（config 在 cypress/config.json）
npm run test:e2e     # cypress run
npm run deploy       # vercel -S carbon-app 部署
```

注意：`lib/api.js` 与 `lib/client.js` 依赖 `NEXT_PUBLIC_API_URL`、`NEXT_PUBLIC_FIREBASE_*`、`NEXT_PUBLIC_IMGUR_CLIENT_ID` 等环境变量，本地未配置时 Firebase 客户端为 null（`lib/client.js` 按 env 是否存在条件初始化），登录/snippet 功能不可用但编辑与导出不受影响。

## 7. 测试与工程质量

- **e2e 为唯一测试层**：`cypress/integration/` 下 7 个 spec 共 44 个用例（`grep -c "it("` 统计），覆盖基本流程、gist 导入、localStorage 持久化、embed、背景色，无单元测试
- **安全测试罕见地显式**：`security.spec.js` 专门验证 `?bg=` 参数注入 `</style><img onerror=alert>` 不弹窗（XSS 回归）、响应安全头（X-XSS-Protection 等，与 `next.config.js` headers() 对应）、/privacy /terms 路由
- **视觉回归**：devDependencies 含 `@applitools/eyes-cypress`，`visual-testing.spec.js` 存在（CI 中 record key 已注释，属未启用的降级状态）
- **CI**：`.github/workflows/validate.yml`——push 触发，yarn → lint → build → `next start` → wait-on → cypress 全链路
- **代码质量**：ESLint 含 jsx-a11y 无障碍插件（无障碍是团队明确关注点，README 的 Tweet alt text 节也可见）；husky pre-commit 跑 lint-staged；代码风格统一 Prettier
- 观察面：主组件 `Editor.js` 476 行/`Carbon.js` 515 行偏大但职责清晰；vendored 的 dom-to-image 778 行标记 `/* eslint-disable */`，长期不再升级，是该库的上游风险点

## 8. 社区与维护状态

- **活跃度**：gh api 显示最近 push 2026-11-05；2026-10-26/27 有一次密集维护——原作者 Mike Fix 提交 "fix: get CI and Vercel deploys passing again (#1612)"，同期 5+ 条 dependabot 依赖升级（nanoid、browserslist、js-yaml 等）。但此前提交有明显断档（2023-01、2024-04 各一条人类提交），2026-10 的重启更像"复活维护"而非持续开发
- **Release**：最后 release 4.9.10 于 2022-06-13，此后 4 年未发版（package.json 版本号也未动），说明项目按"持续部署到 carbon.now.sh"运营，不依赖 GitHub release
- **贡献者**：contributors API 拉 100 条即满（anon 含在内），README all-contributors 徽章 160 人；提交主体已转向 dependabot + 少量维护者
- **社区生态**：README 记录 8 个编辑器插件 + CLI/Telegram bot/R 包/Slack 命令等社区工具；24 种语言翻译文档（docs/ 目录），OpenCollective 赞助页，Vercel 托管赞助
- **Issue/PR 流程**：`.github/ISSUE_TEMPLATE/` 3 套模板 + PR 模板 + CODE_OF_CONDUCT + FUNDING + dependabot.yml + ranger.yml（Reporanger 机器人），流程完备；open_issues 78（gh api）

## 9. 亮点与风险

**亮点**

1. "页面即画布"的导出架构：CodeMirror 排版结果直接经 dom-to-image 栅格化，客户端零依赖服务即可出 PNG/SVG，服务端 og:image 与客户端共用同一渲染逻辑（`lib/dom-to-image.js` + `pages/api/image/[id].js`）
2. URL 编码全部编辑器状态 + localStorage 兜底 + Firebase snippet 云存档，三层持久化让"分享即快照、链接可再编辑"（`components/Editor.js`、`lib/routing.js`、`lib/api.js`）
3. 配置体系统一收敛在 `lib/constants.js`：29 个语法主题均以结构化 token 色声明，新增主题/语言/字体只改一个文件
4. 工程细节扎实：显式 XSS 回归测试、安全头配置（`next.config.js`）、jsx-a11y、选中片段 markText 富文本、Unsplash 背景图版权署名（`components/PhotoCredit.js`）
5. 生态飞轮：编辑器插件 + CLI + 机器人让 Carbon 从工具变成事实标准，反过来持续供给流量

**风险/局限**

1. 技术栈停更：Next 12/React 17/CodeMirror 5/Firebase 8 均落后当前主版本，vitest/jest 单测缺位，重构成本随时间上升
2. dom-to-image 上游自 2019 年后几乎停更且被 vendor 进仓库；`pages/api/image/[id].js` 还从 unpkg 动态注入该库（第三方 CDN 依赖 + 代码注入面），并有源码中自嘲的 `TODO proper auth`
3. 维护断档风险：2022 后 release 停发、2023-2025 人类提交稀疏，2026-10 起主要靠 dependabot 与修复性提交维持，路线图层面未见新功能动向
4. 无 TypeScript、组件体量偏大（Editor/Carbon 各约 500 行），新人贡献门槛高于现代同类项目

## 10. 总结评价

Carbon 是"用开源把一个美学需求做成类目定义者"的教科书案例：2017 年至今 8 年间，它把代码截图从手工截图升级为一键排版出图，且以 URL 快照 + og:image unfurl + embed iframe 构成了完整的传播闭环。当前代码处于成熟稳定但技术栈老化的状态，维护已从功能开发转入保活模式（CI/依赖修复），社区工具生态仍在承担实际增长。适合三类人：想看"Puppeteer + dom-to-image 出图管线"和"URL 状态同步"经典实现的 Web 工程师；想做一个同类截图/卡片工具的创业者（架构可直接借鉴，竞品 ray.so 在体验上的超越点主要是主题预设与快捷键集成）；以及需要研究 2017-2021 年代 Next.js Pages Router 真实工程形态的学习者。若要在此代码上二次开发，建议优先补齐测试层并考虑把 dom-to-image 换成 html-to-image 类维护中的替代品。
