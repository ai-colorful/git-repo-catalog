# resume/resume.github.com 项目分析

> 星数 62913（快照 2026-10-24）｜主语言 JavaScript｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：**GitHub Résumé**——输入 GitHub 用户名，基于其公开 API 数据在浏览器端自动生成一页式开发者简历的纯前端 Web 服务（无任何后端）。

| 项目 | 内容 | 出处 |
| --- | --- | --- |
| 名称 | resume/resume.github.com | gh api |
| 简介 | "Resumes generated using the GitHub informations" | gh api description |
| 星数 | 62,913（快照）；实测 62,908（2026-10-31） | gh api |
| 语言 | JavaScript（实际为 100% 前端静态资源） | gh api |
| 许可证 | **无**（`license: null`，仓库内无 LICENSE 文件） | gh api + tarball 文件清单 |
| 默认分支 | master（另有 gh-pages 及 4 个历史 opt-out 分支） | gh api branches |
| 最后实质代码提交 | 2016-08-05（master："Rewrite and cleanup README"） | gh api commits |
| pushed_at | 2023-02-15（无对应任何分支的可见提交，见第 8 章） | gh api |
| 当前服务状态 | **实质停运**（推断依据见第 8 章；未直接访问站点验证） | 本分析 |

## 2. 项目是什么，解决什么问题

这是 GitHub 生态最早的"标志性玩具"之一：招聘者/面试官拿到候选人 GitHub ID 后，访问 `resume.github.io/?用户名` 即可得到一份排版好的单页简历，包含个人活跃度评级、语言分布、最受欢迎仓库、合并 PR 贡献和组织归属（`README.markdown:3-7`、`views/resume.html`）。作者在首页自述动机：作为初创公司老板，"收到附带 GitHub 账号的简历时能直接看到实际产出"，灵感来自 John Resig 的一条推文（`views/index.html:5-13`）。

服务由 David Coallier（@davidcoallier，页脚署名 `views/index.html:55`）创建，简历版式来自 Things That Are Brown 2009 年的免费模板（`css/resume.css:1-10` 页头注释）。

最具特色的是 **opt-in 机制**：为了让简历只对"愿意公开"的用户生成，用户必须 **star 本仓库** 才会开启自己的简历——前端逐页翻查目标用户的 starred 列表，确认含有 `resume/resume.github.com` 才继续渲染（`js/githubresume.js:101-147`，`README.markdown:5`）。这个"star 换简历"的设计既是隐私开关，也是病毒式涨星引擎：62k 星中有相当一部分来自想开启简历的用户，而非源码使用者。更早期（2013 年）的退出方式是维护者手工在 git 分支里记录 opt-out 名单，仓库至今保留 `opt-out`/`opt-outs`/`optouts`/`more-opt-outs` 4 个分支，最后一次提交均在 2013-11-19（gh api branches/commits）。

技术上它没有传统意义上的"项目工程"：没有任何后端代码，唯一的 Ruby 文件 `config.ru` 只是用 Rack::Static 在本地开发时托管静态文件（`config.ru:1-2`、`README.markdown:9-15`）。线上形态是 GitHub Pages（README 给出的地址是 `https://resume.github.io/`，仓库名中的 `.com` 应是 GitHub 当年特殊配置的域名别名，仓库内无 CNAME 文件，此点未确认）。

## 3. 技术栈与依赖

| 依赖 | 版本 | 用途 | 出处 |
| --- | --- | --- | --- |
| jQuery | 2.1.4 | DOM/AJAX 主体；Google CDN 加载，`window.jQuery` 不存在时回退本地副本 | `index.html:14-15`，`js/vendor/jquery-2.1.4.min.js` |
| Mustache.js | 0.3.0 | 客户端模板渲染 | `index.html:16`，`js/githubresume.js:304` |
| YUI | 2.7.0 reset-fonts-grids | 页面栅格/重置 CSS（2009 年代遗产） | `index.html:11` |
| html5shiv | 3.7.0 | IE 下 HTML5 标签兼容 | `index.html:6-8` |
| GitHub REST API v3 | — | 全部数据源，走 JSON-P（`?callback=?`） | `js/githubresume.js:56-99` |
| Google Analytics | UA-21222559-1 | 访问统计（旧 `_gaq` 异步写法） | `js/githubresume.js:3,560-570` |
| Ruby + Rack | — | 仅开发用静态服务器 | `config.ru`，`README.markdown:11-15` |

无 package.json、无构建系统、无打包工具、无 lint 配置——整个仓库就是可直接部署的静态文件（tarball 全部 28 个条目均为源文件，无产物目录）。

## 4. 目录结构与代码组织

| 路径 | 职责 |
| --- | --- |
| `index.html` | 单页壳：引 CDN 依赖，body 只有一个 `#resume` 容器，其余全由 JS 注入 |
| `js/githubresume.js` | **全部业务逻辑**（572 行单文件）：路由、API 调用、评分、聚合、渲染调度 |
| `js/vendor/jquery-2.1.4.min.js` | jQuery 本地兜底副本 |
| `views/resume.html` / `resumeOrgs.html` | 个人/组织两种简历主模板（Mustache） |
| `views/job.html`、`contrib.html`、`org.html` | 仓库条目、贡献条目、组织条目子模板 |
| `views/index.html` | 首页（含输入用户名表单及示例链接） |
| `views/api_limit.html`、`not_found.html`、`error.html`、`opt_out.html` | 四种降级/错误视图 |
| `css/resume.css`、`css/print.css` | 屏幕样式与专门的打印样式 |
| `images/` | favicon、loading gif、背景纹理等 |
| `config.ru` | 本地开发静态服务器（2 行） |

组织方式评述：典型的"一页应用即一个 JS 文件 + 一把 HTML 模板"的 2010 年代初形态。没有模块化（全局函数 + jQuery 回调），没有状态管理，数据获取与渲染交织在同一个文件里。以今天标准看几乎不可维护，但在"零依赖部署、复制即用"的语境下是当时的主流做法。

## 5. 核心架构与关键实现

### 5.1 极简路由与数据获取层

入口逻辑只有 30 行：解析 `?username` 查询参数决定进入 `run()` 还是 `home()`（`js/githubresume.js:1-32`）。所有 GitHub 请求走 JSON-P：

```js
var github_user_repos = function(username, callback, page_number, prev_data) {
    var page = (page_number ? page_number : 1),
        url = 'https://api.github.com/users/' + username + '/repos?per_page=100&callback=?',
        data = (prev_data ? prev_data : []);
    if (page_number > 1) { url += '&page=' + page_number; }
    $.getJSON(url, function(repos) {
        data = data.concat(repos.data);
        if (repos.data.length == 100) {
            github_user_repos(username, callback, page + 1, data);
        } else { callback(data); }
    });
}
```
（`js/githubresume.js:60-76`）——手写递归翻页累积全量仓库，同类实现还有 merged PR 搜索（`:78-95`）。注意 `:12-14` 处 URL 参数解析有缺陷：正则循环里始终写 `urlParams[0]`，多个查询参数会互相覆盖。

### 5.2 opt-in 检查：star 列表全量翻页（服务的命门）

```js
var github_user_starred_resume = function(username, page) {
    var star  = false;  var repos = [];
    var url   = 'https://api.github.com/users/' + username + '/starred?per_page=100&page=' + page;
    $.ajax({
        url: url, async: false, dataType: 'json',
        success: function(data) { repos = data; },
        error: function(e) {
            if (e.status == 403) { errorMsg = 'api_limit' }
            else if (e.status == 404) { errorMsg = 'not_found' }
        }
    });
    $.each(repos, function(i, repo) {
        if (repo.full_name == "resume/resume.github.com") { star = true; return false; }
    });
    if (star) { return star; }
    if (repos.length == 100) { star = github_user_starred_resume(username, page + 1); }
    return star;
}
```
（`js/githubresume.js:104-147`）——用**同步 AJAX**（`async: false`，现代 jQuery 已废弃）匿名翻页遍历目标用户全部 starred 仓库，找到本项目即视为 opt-in；403 映射到 api_limit 视图、404 映射到 not_found（`:149-185`）。匿名 API 限额 60 次/小时，意味着给一个 star 数超过几百的用户生成简历几乎必然触发限额——这是该服务后期可用性崩塌的结构性原因。

### 5.3 活跃度评级与简历聚合

用户状态是一个手工调参的加权评分（`js/githubresume.js:247-292`）：

```js
var statusScore = data.public_repos * COEF_REPOS      // 2
                + data.public_gists * COEF_GISTS      // 0.25
                + data.followers * COEF_FOLLOWERS     // 0.5
                + data.following * COEF_FOLLOWING;    // 0.25
// +1：2008 年 6 月前注册的"早期用户"（:241-244）
// +1：blog、email、location 三项资料齐全（:270-272）
// 按 0/5/20/50/150 分档 → Inactive / Newbie / Regular / Advanced / Enthusiastic / Passionate
```

其余聚合逻辑：受欢迎仓库 = 非 fork 仓库按 `watchers + forks` 排序取 Top5（`:320-341`，`maxItems=5`）；语言分布 = 非 fork 仓库主语言计数取 Top9 并算百分比（`:343-398`）；贡献 = 合并 PR 按 `repository_url` 分组计数（`:458-509`）；组织 = orgs 列表 Top5（`:511-556`）。四组数据并行请求，各自 AJAX 回调里再拉取对应 Mustache 模板渲染塞进页面（`:299-312`）。仓库描述中的 `:emoji:` 用正则替换为 GitHub CDN 图片（`:419-423`）。

### 5.4 依赖 GitHub API 私有细节的脆弱点

组织头像处理直接假设头像 URL 形如 `secure.gravatar.com/avatar/<hash>` 并用正则硬匹配（`js/githubresume.js:216-220`），个人头像则依赖 API 响应中的 `gravatar_id` 字段（`:229`，模板 `views/resume.html:19-21`）。这两处都是对 2011-2016 年 API 行为的硬编码，后文第 8 章说明其现状。

## 6. 构建与运行

- 开发运行：`rackup config.ru`（需 Ruby 与 rack gem；`README.markdown:11-15`、`config.ru`）。
- 等价替代：任何静态服务器指向仓库根目录即可（全是静态文件，`config.ru:1` 仅映射 css/images/js/views 并把 `/` 指到 index.html）。
- 线上部署：GitHub Pages（gh-pages 分支，其内容与 master 同构：gh api `git/trees/fa489d1`）。
- 无构建步骤、无测试命令、无依赖安装——这也是它当年"零运维"的卖点。

## 7. 测试与工程质量

- **无测试**：tarball 全部 28 个条目中无任何测试文件/目录；无 CI 配置（无 `.github/` 目录）、无 lint/formatter 配置。
- 代码风格：572 行单文件、全局函数、jQuery 链式回调，无模块化；`async:false` 同步请求、`document.write` 兜底等均为已废弃实践（`js/githubresume.js:113`、`index.html:15`）。
- 可辨识的实际缺陷（静态阅读结论）：URL 参数解析覆盖 bug（`js/githubresume.js:12-14`）；组织简历在现代头像 URL 下会抛异常（`:218`，见第 8 章）。
- 值得肯定处：错误路径考虑较全，api_limit / not_found / opt_out / error 四种降级视图齐备（`views/`）；专门维护了打印样式 `css/print.css`，把"简历可打印"做到位。

## 8. 社区与维护状态

维护时间线（全部来自 gh api 实测，2026-10-31）：

- 2011 年项目创建（GA 账号与推文引用佐证）；2012-09-04 gh-pages 分支最后一次提交；2013-11-19 四个 opt-out 分支最后一次提交（手工退出名单时代终结）。
- **master 最后实质提交 2016-08-05**（PR #237 README 重写；此前 2016-06 PR #233 favicon 修复）；2017-01-01 之后 master 提交数为 0。
- pushed_at 显示 2023-02-15 有 push，但 master、gh-pages 及全部 opt-out 分支在该日期后均无提交记录——该变更不在可见分支提交里，推测为仓库/Pages/域名层面的配置变更（未确认）；时点与"服务从 github.com 域名下线"的社区说法吻合，但仓库内无法直接证实。
- 贡献者 46 人；无 tag、无 release（gh api contributors/tags）。
- 2025-2026 年的 78 个 open issue 抽样前 10 条**全部为垃圾营销帖**（如 #313、#314、#325，gh api issues），无维护者回应；最近的仓库事件是 2026-10-30 的 ForkEvent（gh api events）。仓库未归档，星数仍在缓慢自然增长。

**当前可用性判断**（基于代码依赖与 API 实测，未直接访问站点——分析规范限制网络请求）：

1. `gravatar_id` 已废弃：实测 `gh api /users/defunkt` 返回 `"gravatar_id": ""`，导致 `views/resume.html:19` 的 `{{#gravatar_id}}` 段永不渲染，**头像消失**。
2. opt-in 检查在匿名 60 次/小时限额下需翻页全量 starred 列表（`js/githubresume.js:104-147`），对任何有数百以上 star 的活跃用户**几乎必然 403**，页面显示 "API Limit Reached"（`views/api_limit.html`）——即使页面仍在，简历生成对多数用户已不可用。
3. 组织简历：现代头像 URL 为 `avatars.githubusercontent.com`，`js/githubresume.js:218` 的 `secure.gravatar.com` 正则匹配返回 null，后续 `.match(...)[0]` 取下标抛 TypeError。
4. JSON-P `?callback=` 参数实测 2026 年仍可用（探测 `/users/defunkt?callback=foo` 返回 `/**/foo({...})`），此项依赖未失效。

综合：**服务已实质停摆**——八年无人维护 + GitHub API 演进击穿其三处硬编码假设；即便域名仍能打开首页，核心的简历生成流程对绝大多数用户已不可用。

## 9. 亮点与风险

**亮点**

1. 零后端、零运维的纯前端架构：GitHub Pages + 浏览器直连 GitHub API，2011 年即实现"无服务器"服务的极致形态（`config.ru` 仅为本地开发存在）。
2. "star 即 opt-in"机制设计精巧：一处代码（`js/githubresume.js:101-147`）同时解决了隐私授权与病毒式传播，是该项目冲到 62k 星的直接引擎，堪称 star-gating 增长案例的教科书样本。
3. 错误与降级路径完整：限额、用户不存在、未 opt-in、通用错误各有专门视图（`views/api_limit.html` 等 4 个）。
4. 对"简历"场景的细节用心：独立打印样式（`css/print.css`）、Email/Print 快捷操作（`views/resume.html:1-4`）、emoji 描述渲染（`js/githubresume.js:419-423`）。
5. opt-out 分支群完整保存了 2013 年社区治理的原始形态，有生态史价值。

**风险/局限**

1. **无 LICENSE**（gh api `license: null`，仓库无 LICENSE 文件）：法律上默认保留所有权利，他人不能合规复用这份代码。
2. 结构性依赖已失效：同步 AJAX 全量翻页 starred 检查在匿名限额下不可行（见 5.2、8-2），opt-in 机制自身压垮了服务可用性。
3. 硬编码 GitHub API 私有细节（`gravatar_id`、`secure.gravatar.com` URL、`watchers` 字段语义），API 任何演进都会静默破坏渲染（`js/githubresume.js:218,229`）。
4. 遗产技术栈全面停更（jQuery 2.1.4 / YUI 2.7 / Mustache 0.3），且八年无维护、issue 区被垃圾占领，无复活迹象。

## 10. 总结评价

GitHub Résumé 是 GitHub 早期生态的化石级样本：一个 2011 年诞生的纯前端玩具服务，靠"star 仓库才能开启你的简历"这一机制，把隐私开关做成了增长引擎，最终在星数榜上留下了与代码体量（572 行 JS + 9 个模板）完全不成比例的 62k 星。它的架构在当年是优雅的零运维典范，但也正是 opt-in 检查的全量翻页设计，叠加匿名 API 限额与 GitHub API 的字段演进（gravatar_id 废弃、头像 URL 变更），使服务在无人维护的八年里实质失效；2023-02-15 的神秘 push（无对应提交）与 2016 年后归零的提交记录共同指向服务已于 GitHub 侧下线或弃管。它不适合作为现代简历生成器的实现参考，但非常适合三类人：研究 GitHub 生态史与 star 增长机制（star-gating 案例）的人、想看 2010 年代初"Pages + JSON-P + Mustache"纯前端服务完整形态的人、以及做 API 兼容性教训研究的人——这个仓库本身就是"硬编码第三方 API 私有细节会怎样死掉"的最佳实物教材。
