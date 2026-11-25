# FreeCodeCampChina/freecodecamp.cn 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 JavaScript（GitHub 语言统计以 CSS 居首）｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

freeCodeCamp 的中文社区本地化部署版：把 freeCodeCamp 官方开源学习平台（前后端 + 课程库）整体 fork 后完成中文翻译，并叠加中国社区运营功能（微信群/QQ 群报名、城市学习小组），曾支撑 freeCodeCamp.cn 独立运营，是 2016-2018 年中国编程教育开源社区热潮的代表性产物；现已停更，被官方中文版取代。

| 项 | 值 | 出处 |
|---|---|---|
| 仓库 | FreeCodeCampChina/freecodecamp.cn | gh api |
| 创建时间 | 2016-09-30 | gh api `created_at` |
| 默认分支 | dev | gh api `default_branch` |
| 最后实质代码提交 | 2018-05-29（PR #512 Safari 兼容） | gh api commits |
| 最后 push | 2023-07-16（仅 dependabot 分支活动） | gh api `pushed_at` |
| 许可证 | BSD-3-Clause（软件）+ CC-BY-SA-4.0（课程）+ CC-BY-NC-4.0（翻译，禁商用） | `LICENSE`、`LICENSE-freeCodeCamp-Translation.md` |
| 贡献者 | 约 361 人 | gh api contributors 分页 |
| Release | 0 个（未发版） | gh api releases |
| 官网 | https://freecodecamp.cn（已失效，主页字段现为 https://fcc.asia/） | gh api `homepage` |

## 2. 项目是什么，解决什么问题

2016 年前后，freeCodeCamp 在全球爆红，但官方只有英文站，中国学习者面临语言门槛和境内访问不稳定两大障碍。FreeCodeCampChina 社区（由一批中国志愿者组织，Issue 模板和 README 中可见其以微信群/QQ 群/Gitter 中文室组织线下活动）于是把官方平台的完整代码库 fork 到本仓库，翻译课程内容后以 freeCodeCamp.cn 域名独立部署运营（README.md 明言 "This code is running live at freeCodeCamp.cn"）。

它解决三件事：其一，课程中文化——把官方四大认证（前端/数据可视化/后端/全栈）的全部挑战题描述翻译成中文；其二，本地运营——在官方代码之上加入报名表单、微信/微博导流等中国式社区运营触点；其三，离线可用性——提供一套可在境内自行部署的完整学习平台源码。

代码本体是 freeCodeCamp 官方仓库 2017 年前后的快照（`package.json` 中 React 0.14、webpack 1、gulp 3、LoopBack 2 的组合是官方当时的真实技术栈），本仓库在此基础上做中文翻译与中国化改造。四个认证路径共 411 个挑战（统计自 `seed/challenges/**/*.json`），描述文本已中文化（题目标题保留英文，如 basic-javascript.json 首题 title 为 "Comment your JavaScript Code" 而描述为中文）。

值得注意的是本仓库并非 GitHub 意义上的 fork（gh api 返回 `fork: false`，推测为脱离 fork 关系的独立导入），这使其积累了独立的 star 数和 issue 生态。随着 freeCodeCamp 官方把中文（chinese / chinese-traditional）正式纳入主仓的 i18n 体系（已核实官方仓 `client/i18n/locales/` 下存在 chinese、chinese-traditional 目录），本仓库的历史使命完结，2018 年中即实质停更。

## 3. 技术栈与依赖

技术栈冻结在 2017 年初官方 freeCodeCamp 的形态（出处 `package.json`、`bower.json`）：

- **服务端**：Node.js（Travis 配置锁 4.2.1，见 `.travis.yml`）+ LoopBack 2（StrongLoop 框架，`server/server.js` 以 `loopback()` 创建 app、`loopback-boot` 启动）+ MongoDB（`loopback-connector-mongodb` 1.13）
- **认证**：loopback-component-passport + passport-{github,facebook,google-oauth2,linkedin-oauth2,twitter,local}（`package.json`、`server/passport-providers.js`）
- **前端**：React 0.14 + Redux 3 + react-router 2 + redux-form/react-router-redux/reselect，另有 thundercats 3（官方当年自研的 Rx 风格同构框架）共存；客户端补库走 Bower：D3 3.5、jQuery 2.1、Bootstrap 3.3、CodeMirror 5.8、cal-heatmap 等（`bower.json`）
- **构建**：gulp 3 全家桶（less/uglify/rev/webpack-stream），webpack 1 打包，jade 模板渲染，Less 样式 98 个文件（`client/less/`）
- **生产部署**：pm2 + pmx（Keymetrics 监控），入口 `pm2Start.js`
- **测试**：tape + tap-spec + sinon，挑战校验走 `seed/test-challenges.js`

## 4. 目录结构与代码组织

| 目录 | 职责 | 出处 |
|---|---|---|
| `server/` | LoopBack 服务端：boot 启动脚本、中间件、模型配置、jade 视图、passport 认证 | `server/server.js`、`server/boot/` |
| `common/` | 同构 React 应用（app/components/routes/redux/sagas）+ LoopBack 模型定义（models/*.json） | `common/app/`、`common/models/` |
| `seed/` | 课程库核心：challenges/ 全部挑战 JSON + 加载/校验脚本 | `seed/challenges/`、`seed/getChallenges.js` |
| `client/` | 浏览器端入口脚本、Less 样式、iframe 沙箱脚本（挑战运行环境） | `client/` |
| `public/` | 静态资源：字体、图片、独立落地页（含中国化的 join-group 报名页） | `public/js/join-group.js` |
| `config/` | 密钥配置占位（`config/secrets.js`） | — |
| `test/` | 服务端单元测试（仅 2 个文件） | `test/server/` |

组织方式为官方 monorepo 式单仓：前后端 + 课程内容同仓，课程（数据：JSON）与平台（代码：JS）分离是 freeCodeCamp 的经典设计——课程文件纯 JSON 声明式描述挑战（题目、种子代码、测试、提示），平台引擎负责渲染与判题，这使翻译工作可以只动 JSON 不碰代码。

## 5. 核心架构与关键实现

### 5.1 课程库与翻译体系（本项目最有价值的部分）

`seed/challenges/` 按 4 个认证板块 + 视频挑战组织为 5 个子目录、49 个 JSON 文件、17,233 行，共 411 个挑战。前端认证板块最完整：basic-javascript.json 108 题、html5-and-css.json 68 题、bootstrap.json 31 题、bonfires（算法题）合计 48 题；后端与数据可视化认证仅骨架（d3.json/react.json 各 1 题，多为占位），说明翻译完成度随上游课程推进而递减（上游 2017 年初这两个认证也才刚成形）。中文改造不止于翻译：`seed/challenges/00-getting-started/getting-started.json` 里插入了"欢迎关注微信公众号：FreeCodeCamp"和微博图片导流内容，属社区自加。挑战加载器 `seed/getChallenges.js` 按目录序号（00-04）解析 superBlock 顺序并递归收集 JSON。

### 5.2 服务端启动与认证流水线

`server/server.js`（全文约 150 行）是经典的 LoopBack 装配：`loopback()` 实例 → express-state 注入 → PassportConfigurator 初始化 → `loopback-boot` 按 `server/boot/` 目录加载路由。第三方登录的 profile 归一化在 `passportOptions.profileToUser` 中完成：无邮箱时生成 `fcc+uuid` 临时用户名（对应官方 GitHub 登录为主的使用习惯），十八个 boot 文件（`server/boot/`，如 `challenge.js`、`certificate.js`、`user.js`）按功能拆分路由。

### 5.3 同构 React 前端

`common/app/` 采用 Redux + saga 的同构架构：`create-app.jsx` 负责服务端/客户端双端挂载，`redux/` 持 actions/reducer/types，`sagas.js` 处理副作用，`routes/` 下按 Map（课程地图）、Bonfires（算法）、Hikes（视频）、Jobs 四大页面组织。同时保留了官方当年的实验框架 thundercats 依赖，两套 React 数据流方案并存是 2016-2017 官方仓的原始状态。

### 5.4 中国化社区运营功能（本 fork 独有）

`public/js/join-group.js` 是独立的城市学习群报名页：Angular 1 写的表单（手机号正则、城市、背景收集），提交到 `/update-profile` 后提示"我们稍后会把你拉到对应城市的学习群"，并引导添加微信号 fcczhongguo；对应地 `common/models/user.json` 第 112 行给 User 模型新增了 `wechat` 字段——这是官方代码中不存在的私有扩展。`.github/ISSUE_TEMPLATE.md` 则留下了社区治理痕迹："这里只处理 FreeCodeCamp.cn 的相关问题，不处理 w3cschool 的任何问题"，QQ 群 526289580 验证答案为 freeCodeCamp。

## 6. 构建与运行

来自 `CONTRIBUTING.md` 与 `package.json` scripts（Node 4 时代命令，今需旧环境）：

```bash
git clone --depth=1 https://github.com/freecodecampchina/freecodecamp.cn.git
npm install && npm install -g gulp bower && bower install
cp sample.env .env        # 关键项：MONGOHQ_URL、SESSION_SECRET，邮件登录需 MANDRILL_*
mongod                    # 本地 MongoDB
npm run only-once         # 建库 seed
gulp                      # 开发模式，http://localhost:3001
```

其他命令：`npm run build`（生产构建 gulp build -p）、`npm start`（babel-node 起 server）、`npm run start-production`（pm2）、`npm test`（先 lint 全部 js/json 再跑挑战校验 `npm run test-challenges`，即 tape 校验全部课程 JSON 结构）、`npm run lint`（eslint 2 + jsonlint）。

## 7. 测试与工程质量

- 测试核心是课程数据校验而非业务逻辑：`seed/test-challenges.js` 用 tape + tap-spec 对全部挑战 JSON 做结构断言（id 格式、必填字段、去重等），这是上游"课程即数据"质量观的表现；`npm test` 前置运行 eslint 与 jsonlint（`package.json` scripts、`gulpfile.js` 中 lint 任务）。
- 单元测试极薄：`test/server/` 仅 date-utils-test.js 与 user-stats-test.js 两个文件。
- CI：`.travis.yml` 仅声明 Node 4.2.1 跑默认 test，配置简单（fork 自上游同款）。
- 工程质量整体是官方仓 2017 年水准的忠实快照：eslint 规则沿用官方（80 列限制、react 插件），缺陷在于依赖全面过期（React 0.14、webpack 1、LoopBack 2 均已 EOL），且本仓库 fork 后未跟进上游重构（官方 2018 年后迁移至 Gatsby/Next 新架构），代码已无升级路径。

## 8. 社区与维护状态

- **活跃期 2016-09 至 2018-05**：gh api 提交历史显示，实质开发止于 2018-05-29（Xing Liu 合并 PR #512 "fix/safari-compatibility-description"），此前 2018-05-24 还在合并社区翻译/修复 PR（#486 等）。分支列表现存 dev（默认）、master、staging、feature/cn-site 与 3 个 2022 年 dependabot 自动分支。
- **尾声 2019-2020**：仅剩 JackJin 两次 README 更新（2019-07-18、2020-01-22），无代码变更。
- **2023-07-16 的最后 push** 来自 dependabot 分支活动，主干 dev 自 2020 年后无实质变化。
- 0 个 release、约 361 名贡献者（含匿名统计）、159 个遗留 open issue（gh api），issue 区已是无人值守状态。
- 衰落原因可从仓库内部证据推知：其一，官方 freeCodeCamp 主仓已原生支持中文（官方仓 `client/i18n/locales/chinese` 存在，已核实），freecodecamp.org 中文版上线后社区版失去存在意义；其二，`.github/ISSUE_TEMPLATE.md` 中"不处理 w3cschool 的任何问题"的声明侧面反映 freeCodeCamp.cn 域名运营方在 2018 年前后与 w3cschool（编程狮）的商业化纠葛，社区版与官方关系逐渐疏远。组织背景细节（运营主体变更的确切时间线）为公开报道层面的信息，本仓库内无更多一手证据，细节未确认。

## 9. 亮点与风险

**亮点**

1. 中文开源学习平台的开创性样本：完整翻译 411 个挑战（17,233 行课程 JSON），在官方无中文版的年代服务了大量中国初学者，39.5k star 是社区需求的直接投票。
2. "课程即数据"架构的教科书：课程内容纯 JSON 声明、与平台代码解耦（`seed/challenges/`），翻译者无需懂代码即可贡献，本仓库正是这一架构的成功用例。
3. 本地化改造有真实增量：用户模型微信字段、城市学习群报名流程（`common/models/user.json`、`public/js/join-group.js`）展示了开源平台社区化运营的可行路径。
4. 保留了一份 2017 年 freeCodeCamp 原始架构快照（LoopBack 2 + thundercats + gulp/webpack 1），对研究官方平台演进史有档案价值。

**风险/局限**

1. 已死项目：三年无代码提交，依赖全部 EOL（含多个已知漏洞版本），严禁在生产部署。
2. 翻译完成度参差：后端/数据可视化认证仅占位挑战，与上游现版课程差距巨大，内容已过时（jQuery/Bootstrap 时代）。
3. fork 断代：脱离上游后无升级路径，官方中文版（freecodecamp.org/chinese）才是现役方案。
4. 翻译采用 CC-BY-NC-4.0（禁商用，`LICENSE-freeCodeCamp-Translation.md`），与课程本身的 CC-BY-SA-4.0 叠加后使用边界复杂，商用需谨慎。

## 10. 总结评价

这是一个"使命已完成"的仓库：它是中国开发者社区在官方缺席的年代自建中文编程学习平台的集体努力，411 个中文化挑战、361 名贡献者和近 4 万 star 记录了 2016-2018 年中国开源社区的热情峰值；随着 freeCodeCamp 官方收编中文本地化，它于 2018 年年中自然谢幕，如今的价值在于历史档案——想研究 freeCodeCamp 早期架构（LoopBack + React 同构 + 课程数据化）或中文开源社区运营史的人值得翻阅，想学编程或部署学习平台的人则应直接使用官方 freeCodeCamp/freeCodeCamp 主仓及其在线中文版。
