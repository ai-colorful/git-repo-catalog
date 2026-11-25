# slatedocs/slate 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-17
>
> 特别说明：本仓库已于 2026 年 2 月被原作者从 GitHub 撤下（详见第 8 章），当前 GitHub 上仅存一个 `migration-notice` 通知分支。本分析基于上游真实源码快照——经旧 fork `amanfoundongithub/slate` 保留的上游最后提交 `97c8dd4c`（2024-02-07，与多个独立 fork 交叉核实）取得 tarball 分析；另经 `gh api` 核实仓库元数据（2026-11-16：archived=true，星数 35989）。

## 1. 项目速览

一句话定位：Slate 是 API 静态文档框架的鼻祖级项目——用纯 Markdown 写文档，构建出"左侧目录 + 中间正文 + 右侧深色代码示例"三栏单页文档站，这一布局自 2013 年起成为 API 文档的事实模板。

| 项目 | 内容 |
|---|---|
| 名称 | slatedocs/slate（原 tripit/slate → lord/slate） |
| 星数 | 快照 39500（2026-10-24）；gh api 实测 35989（2026-11-16，撤仓后明显回落） |
| 主语言 | 任务标注 JavaScript；实际工程以 Ruby（Middleman 构建链）为主，前端为少量原生 JS + vendored 库（`.gitattributes` 中 `source/javascripts/lib/* linguist-vendored` 已排除语言统计） |
| 许可证 | Apache-2.0（`LICENSE`，代码版权头为 Concur Technologies 2008-2013） |
| 最近真实代码活跃 | 2024-02-07（最后上游提交，CI actions 升级）；最后 release v2.13.1（2023-01-31，CHANGELOG.md） |
| 当前状态 | GitHub 仓库已 archived，默认分支替换为 `migration-notice`，原 main 已删除（gh api 核实，2026-11-16） |

## 2. 项目是什么，解决什么问题

写 API 文档时开发者面临两难：要么用 Readme.io 这类商业托管服务（付费、被锁定），要么自己手写 HTML（丑、难维护）。Slate 的答案是：文档就是 Markdown，`bundle exec middleman build` 一下，产出一个纯静态单页文档站——README.md 开篇即写明 "Slate helps you create beautiful, intelligent, responsive API documentation"。

它的核心设计决策有三个。第一，**单页展示全部内容**：所有 endpoint 文档渲染进一个 HTML 页面，配合滚动时自动更新的 URL hash 与左侧目录高亮（`source/javascripts/app/_toc.js`），"链接到某节"的体验不因单页而损失；README 称该设计灵感来自 Stripe 与 PayPal 的 API 文档。第二，**三栏布局**：中间是说明文字，右侧是与说明对齐的深色代码示例区，左侧是可折叠目录——文字与代码逐段左右对照，这是 Slate 传入后世的招牌样式。第三，**多语言代码示例 tab 切换**：同一 endpoint 的 curl/Ruby/Python/JS 示例放右上角 tab 里切换，选择记忆在 localStorage 和 URL query 中（README Features 节 + `source/index.html.md`）。

对文档作者而言，使用方式极简：仓库本身就是文档模板（README 鼓励直接按 "use this template" fork），`source/index.html.md` 一个入口文件加任意数量的 `source/includes/*.md` 片段，frontmatter 里声明语言 tab、搜索开关、meta 标签等即可，零配置获得 100+ 语言的语法高亮（内置 Rouge，README Features 节）。

它服务过的组织阵容是当时 API 经济的缩影：README "Companies Using Slate" 节列出 NASA、Sony、Best Buy、Travis-CI、Greenhouse、WooCommerce、Dwolla、Clearbit、Coinbase、Parrot Drones 等；撤仓通知（migration-notice 分支 readme）补充 "It was once used to host API docs for projects at NASA, Sony, Best Buy, Clearbit, Dwolla, Parrot Drones, WooCommerce, and Axidraw, among many others"。

## 3. 技术栈与依赖

构建链（`Gemfile`，Ruby >= 2.6）：

| 依赖 | 版本约束 | 用途 |
|---|---|---|
| middleman | ~> 4.4 | Ruby 静态站生成器（Slate 的骨架），自带 sprockets 资产管线、布局/模板系统 |
| middleman-syntax + rouge | 3.2 / ~3.21 | 代码块服务端语法高亮（Rouge 支持 100+ 语言） |
| redcarpet | ~> 3.6.0 | Markdown 渲染引擎（Slate 深度定制其 renderer，见第 5 章） |
| nokogiri | ~> 1.13.3 | 解析渲染后的 HTML 以抽取标题树（`lib/toc_data.rb`） |
| middleman-autoprefixer / sass / webrick | - | CSS 前缀补全、样式预处理、Ruby 3 自带 webrick 兼容 |

前端（`source/javascripts/`）：jQuery（vendored）、lunr.js（纯前端全文检索，3475 行）、energize.js（触屏点击模拟）、jquery.highlight；样式为 SCSS（`source/stylesheets/`，screen/print 两套）。另有自定义图标字体（`source/fonts/` + `font-selection.json`）。

运行/分发：Dockerfile（`ruby:2.6-slim` 基础镜像，发布到 Docker Hub slatedocs/slate，支持 amd64/arm64/ppc64le，`.github/workflows/publish.yml`）、Vagrantfile、slate.sh 统一 CLI 脚本。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `config.rb` | Middleman 全局配置：markdown 引擎参数、资产管线、构建优化、helper 注入 |
| `lib/` | 4 个自定义 Ruby 扩展（约 120 行）：unique_head/nesting_unique_head（标题锚点）、multilang（代码块语言 tab 标记）、toc_data（目录树抽取）、monokai_sublime_slate（配色主题） |
| `source/` | 文档源与前端资产：`index.html.md`（入口示例文档）、`includes/_errors.md`（include 片段示例）、`layouts/layout.erb`（全站唯一 HTML 骨架）、`javascripts/`（app 逻辑 4 个 + vendored 库）、`stylesheets/`（5 个 SCSS）、`images/`、`fonts/` |
| `.github/workflows/` | 4 个工作流：build（CI 矩阵）、deploy（main → gh-pages）、dev_deploy、publish（Docker Hub） |
| `slate.sh` / `deploy.sh` | serve/build/deploy 统一入口；`Dockerfile`、`Vagrantfile` 为容器/虚拟机运行支持 |

组织方式评述：极小单体（全仓 45 个文件，除 vendored 库外自研代码约 3000 行）。没有"框架库 + 脚手架"的分层——仓库即模板，用户 fork 后直接改 `source/` 里的 Markdown；这种"以仓库为模板分发"的模式早于 GitHub Template Repository 功能，也是它当年快速传播的原因之一。

## 5. 核心架构与关键实现

### 5.1 渲染管线：三个各不到 35 行的 Ruby 扩展撑起全部特色

`config.rb` 将 redcarpet 的 renderer 替换为自定义的 `UniqueHeadCounter`（`lib/unique_head.rb`）：为每个标题生成唯一锚点 id，重复标题自动加 `-2`、`-3` 后缀；若标题是中文/俄文等被 parameterize 清空的文本，则退化为 SHA1 前 10 位。这是单页文档"每节都可链接"的基础。

```ruby
# lib/unique_head.rb（节选）
def header(text, header_level)
  friendly_text = text.gsub(/<[^>]*>/,"").parameterize
  if friendly_text.strip.length == 0
    friendly_text = Digest::SHA1.hexdigest(text)[0,10]
  end
  @head_count[friendly_text] ||= 0
  @head_count[friendly_text] += 1
  if @head_count[friendly_text] > 1
    friendly_text += "-#{@head_count[friendly_text]}"
  end
  return "<h#{header_level} id='#{friendly_text}'>#{text}</h#{header_level}>"
end
```

`lib/multilang.rb` 是语言 tab 的源头：重写 `block_code`，把 ` ```ruby ` 这类围栏代码块的语言名（支持 `language--variant` 双段写法）转换为额外 CSS 类 `tab-<lang>` 追加到 highlight 容器上。于是构建后每个代码块同时带着"高亮语言类"和"tab 归属类"，前端只需按类显隐即可完成切换。

```ruby
# lib/multilang.rb
def block_code(code, full_lang_name)
  if full_lang_name
    parts = full_lang_name.split('--')
    rouge_lang_name = (parts) ? parts[0] : ""
    super(code, rouge_lang_name).sub("highlight #{rouge_lang_name}") do |match|
      match + " tab-" + full_lang_name
    end
  else
    super(code, full_lang_name)
  end
end
```

`lib/toc_data.rb` 用 nokogiri 解析渲染完的 HTML，抽出 h1/h2/h3 平铺列表后两级归并：h3 挂到最近 h2 的 children，h2 挂到 h1，返回两级目录树供 layout 服务端直出目录（目录不是 JS 运行时生成，SEO 与首屏都友好）。

### 5.2 三栏布局：纯 CSS float 方案 + 双份语言选择器

页面骨架在 `source/layouts/layout.erb`（全站唯一模板）：左侧 `.toc-wrapper`（logo、语言选择器、搜索框、目录），右侧 `.page-wrapper` 内依次是 `.dark-box` + `.content` + `.dark-box`。三栏的关键实现（`source/stylesheets/screen.css.scss` + `_variables.scss`）：

- 左栏 `position: fixed`，宽 `$nav-width: 230px`；
- 中栏文字元素统一 `margin-right: $examples-width: 50%`（`.content > h1/p/table/...`），留出右半空间；
- 代码块 `pre, blockquote` 则 `float: right; width: 50%`，漂到右半区，背景接上 `.dark-box` 的深色（`$code-bg: #1E2224`）——文字与代码两列逐段交错堆叠，正好形成"左文右码"的阅读节奏；
- `.content` 用 `position: relative; z-index: 30` 压在深色底之上（并有 `-webkit-transform: translateZ(0)` 注释标明是为修 webkit 渲染 bug，issue #538）。

布局里藏着两份语言选择器：`.toc-wrapper` 里那份供平板/手机显示，`.dark-box` 里那份供桌面右上角 tab 显示，CSS 按断点切换显隐。响应式三级降级（`screen.css.scss` RESPONSIVE DESIGN 节）：`$tablet-width: 930px` 以下目录滑出屏幕改由 NAV 按钮唤出；`$phone-width`（700px）以下 `.dark-box` 直接隐藏、代码块取消浮动占满全宽——手机上看就退化为传统单栏文档。另有完整的 RTL 适配文件 `source/stylesheets/_rtl.scss`（screen.css.scss 中注释一行即可切换）。

### 5.3 多语言 tab 与前端联动（`source/javascripts/app/_lang.js`）

frontmatter 的 `language_tabs` 经 layout 写入 `<body data-languages="[...]">`；`all_nosearch.js` 启动时调用 `setupLanguages()`，优先级为：URL `?language=` 参数 > localStorage 记忆 > 列表第一个。点 tab 时 `activateLanguage()` 隐藏所有 `.highlight.tab-<lang>` 与 `.lang-specific.<lang>`，再显示目标语言，同时 `history.pushState` 把语言写进 query string、滚动位置经 `scrollIntoView` 保持——切换语言不丢阅读位置。URL 与 localStorage 双持久化意味着"把某语言文档链接发给同事，对方打开就是你发的语言"。浏览器后退由 `window.onpopstate`（`all_nosearch.js`）接管。

### 5.4 纯前端搜索（`source/javascripts/app/_search.js`）

没有预建索引文件：页面加载时直接在浏览器里用 lunr.js 遍历 `.content` 的全部 h1/h2 节点建索引（标题权重 boost 10，正文取节内剩余元素文本），词条数超 5000 时自动把输入防抖放宽到 300ms。搜索结果高亮由 jquery.highlight 完成。这带来零后端、零构建配置的搜索，代价是大文档的建索引时间随 DOM 增长——与单页模式的固有取舍一致。`layout.erb` 中 `search: false` 时改引 `all_nosearch.js`，搜索相关 3475 行 lunr 不进包。

### 5.5 includes 机制与文档组织

入口文档 `source/index.html.md` 的 frontmatter 即配置面：`language_tabs`（须为 Rouge 支持的语言名）、`toc_footers`、`search`、`code_clipboard`、`meta`（自定义 meta 标签）。`includes: [errors]` 声明后，layout.erb 会把 `source/includes/_errors.md` 的渲染结果**追加**到正文末尾（layout.erb 第 18-24 行：`page_content += partial("includes/#{include}")`），用于把错误码表等公共内容拆出复用。下划线前缀文件不单独成页，是 Middleman partial 约定。

## 6. 构建与运行

以下均出自项目自带的脚本与 CI（slate.sh、Dockerfile、.github/workflows/）：

```bash
# 原生（需 Ruby >= 2.6、Git）
bundle install
bundle exec middleman serve     # 开发服务器 http://localhost:4567
./slate.sh build                # 产物输出 build/（等价 bundle exec middleman build --clean）
./slate.sh deploy               # 构建并提交 build/ 到 gh-pages 分支

# Docker
docker run --rm -v $(pwd)/build:/srv/slate/build slatedocs/slate build

# CI（.github/workflows/build.yml）：Ruby 2.6/2.7/3.0/3.1/3.2 矩阵，
# 验证内容就是 bundle exec middleman build 是否成功
# 发布：main push → deploy.yml 构建并推 GitHub Pages；release → publish.yml 推 Docker Hub
```

## 7. 测试与工程质量

没有单元测试、没有 e2e：CI 的 job 名虽然叫 `test`（`.github/workflows/build.yml`），实际验证只是"五个 Ruby 版本都能成功构建出站"。对这种以模板分发、代码量极小、变更集中于构建链的项目，构建通过即回归的取舍可以理解，但 2022-2024 年的提交（如 "Handle incorrectly set language parameter (#1756)"）说明纯构建验证确实会漏行为 bug。工程质量的其他侧面：代码版权头与许可证规范（Apache-2.0，全部自研文件带 Concur 头）；注释质量高——如 config.rb 里对 woff/woff2 资产哈希 bug 的 workaround 写明了缘由并链接 issue #1171；`.editorconfig` 统一编辑器风格；vendored 前端库统一下划线前缀并经 `.gitattributes` 标记 `linguist-vendored`。整体是"小而整洁、但无测试网"的典型。

## 8. 社区与维护状态

本项目维护状态是所有 20k-50k 档项目中最特殊的，时间线如下（均有 gh api / 文件证据）：

- **2013-09-13** 仓库创建（gh api created_at）。作者 Robert Lord（GitHub: lord）在 TripIt 开发，最早托管于 tripit/slate——layout.erb 等文件版权头 "Copyright 2008-2013 Concur Technologies"（TripIt 母公司）可佐证出处。
- **社区接棒**：lord 淡出后仓库迁至社区组织 slatedocs（约 2019 年，精确迁移日期未确认），README 记载由 Matthew Peveler（MasterOdin）与 Mike Ralphson（MikeRalphson）维护。CHANGELOG 显示 2.x 时代主要工作是依赖安全升级（nokogiri 地址性漏洞等）与 Ruby 3 适配，发布节奏放缓至一年 1-2 版。
- **v2.13.1**（2023-01-31）是最后一个 release（gh api releases 核实）；上游 main 最后一次真实代码提交停在 **2024-02-07**（#1727 升级 actions，经 fork 历史核实），此后两年无实质活动——Ruby/Middleman 栈明显老化（Dockerfile 仍是 EOL 的 ruby:2.6-slim）。
- **2026-02-01**（gh api pushed_at）：lord 将仓库内容整体替换为 `migration-notice` 单文件分支并 archive。该 readme（本次已全文读取）声明：作为原作者，他因抗议微软与 ICE、IDF 的合同（2026 年初美国抗议潮背景下），将 Slate 从 GitHub 撤下；代码只读镜像保留在其个人网站 code.lord.io/slate，缘由见博客 lord.io/leaving-github。
- **撤仓后果**（gh api 核实）：原 main 分支与全部历史在 GitHub 上不可再取；2026-02 之后 fork 出的仓库也只剩通知分支；星数从队列快照 39500（2026-10-24）回落到 35989（2026-11-16），约 3.5k；archived=true、open_issues 冻结在 50、linguist 主语言字段清空（language=null）。完整源码现存于撤仓前 fork（如 amanfoundongithub/slate 的上游提交 97c8dd4c）与 lord 自托管只读镜像（静态文件浏览器，无 git 克隆接口）。

## 9. 亮点与风险

亮点：

1. **三栏 API 文档布局的开创者**：2013 年定义的"目录/正文/深色代码区"样式成为整个行业模板，Stripe 文档风潮的直接推手，其视觉语言至今在各类 API 文档中可辨。
2. **输入端极简**：整站只需 Markdown + 少量 frontmatter，"文档即代码 + PR 协作"的理念（README："Let your users update your documentation for you"）领先于时代。
3. **零后端架构**：单页静态产物、URL hash 深链、lunr 前端搜索、语言选择双持久化，部署 anywhere（GitHub Pages 一键）。
4. **实现精悍**：全部特色由 3 个不足 35 行的 Ruby 扩展（unique_head/multilang/toc_data）+ 1300 行自研 JS/SCSS 实现，是"用最小代码做完整产品"的范本，适合精读。
5. 无测试框架的负担、无框架版本焦虑，fork 后即可长期自持——大量组织因此把它当模板用了很多年。

风险/局限：

1. **已从 GitHub 撤下且 archived**：无法 fork、提 issue 或贡献；官方入口只剩 lord 个人站点的只读镜像；新用户没有 "use this template" 路径，选型即死路。
2. **技术栈老化**：Middleman 4 + Ruby 2.6 Docker 基础镜像（已 EOL）+ rouge 3.x，最后 release 2023-01、最后代码提交 2024-02，安全维护实质停止。
3. **单页模式天花板**：几百个 endpoint 后 DOM 与 lunr 建索引性能承压（项目自己都在 _search.js 里做了 5000 词条降速补救），且无多版本、多页、i18n、组件化等现代文档平台能力。
4. 前端依赖 jQuery/老 lunr/sprockets 资产链，二次定制的学习成本不低（Middleman/Sprockets 文档生态本身也过时）。

## 10. 总结评价

Slate 是 2013 年一代 API 文档工具的巅峰与活化石：它发明了沿用至今的三栏单页文档范式，用极小的代码量和"纯 Markdown 输入"定义了此后十年 API 文档的默认形态（Readme.io、Docusaurus 的 API reference 插件、Stoplight 等的样式基因都能追到它）；与今天的主流替代品相比，Docusaurus 是版本化多页 React 站、Readme.io/Mintlify 是商业化 SaaS 平台，Slate 则坚守"一个 Markdown 单页、零配置自托管"的极简哲学。它的星数滑坡与撤仓（2026 年初 lord 因政治立场把项目带离 GitHub）为"单点依赖的个人开源项目"写下了最新注脚：39k 星不能保证项目存活，只保证它的代码会在无数 fork 里继续存在。适合谁：想自托管一个不依赖任何现代前端框架的轻量 API 文档、且能接受冻结技术栈的老项目团队；想理解三栏文档布局与服务端 Markdown 管线如何拼装的开发者（源码极小、注释优秀，一天可读完）。对新项目而言，应视 Slate 为历史坐标与设计参考，而非今天的选型答案——现实可用版本只能从撤仓前 fork 中获取，且无上游修复。
