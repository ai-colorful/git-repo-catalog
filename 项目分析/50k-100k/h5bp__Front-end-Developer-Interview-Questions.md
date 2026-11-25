# h5bp/Front-end-Developer-Interview-Questions 项目分析

> 星数 60861（快照 2026-10-24）｜主语言 Nunjucks｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：HTML5 Boilerplate（h5bp）组织旗下的一套开源前端面试题库——给面试官挑人、给求职者自测、也可以当作前端知识自检清单。

| 项目 | 内容 |
| --- | --- |
| 名称 | h5bp/Front-end-Developer-Interview-Questions |
| 官网 | https://h5bp.org/Front-end-Developer-Interview-Questions/（GitHub API homepage 字段） |
| 星数 / Fork | 60861（快照）/ 10960（GitHub API） |
| 主语言 | Nunjucks（网站模板）；实质主体是 Markdown/纯文本内容 |
| 许可证 | MIT（`LICENSE.md`，版权 2012-2023 Contributors） |
| 创建时间 | 2012-02-09（GitHub API `created_at`）；项目始于 2009 年，2012 年公开（`src/about.njk`） |
| 最近活跃 | 仓库 push 时间 2026-06-16（GitHub API）；main 分支最后一次人工内容合并 2024-07，2026-03 仅有 dependabot CI 升级（commits API） |
| 默认分支 | main（GitHub API） |

## 2. 项目是什么，解决什么问题

这是一个几乎不需要介绍的"元老级"内容型仓库：一份按主题分类的前端面试问题清单，覆盖通用素养、HTML、CSS、JavaScript、测试、性能、网络、编码实战和趣味题九大板块（`src/questions/` 下 9 个 markdown 文件 + `src/_data/questions.json` 中的分类索引）。README 开宗明义：不要把所有题都问同一位候选人，从中挑选几道即可（`README.md`）。题目刻意写成开放式——不设标准答案，贡献指南甚至明文禁止在 issue/PR 中提交答案，鼓励面试时通过讨论观察候选人的思考过程（`.github/CONTRIBUTING.md` 第 21 行、`src/about.njk`）。

项目历史本身就是一个开源社区故事：2009 年由 Darcy Clarke 发起，最初的题目来自 Paul Irish 起头的一个共享 etherpad 文档，2012 年正式公开并进入 h5bp 组织（`src/about.njk` 的 Overview 一节）。`src/_data/og.json` 记录了 14 位"原始贡献者"，Paul Irish、Ben Alman、Rebecca Murphey、Alex Sexton 等早期 jQuery/前端社区核心人物都在列。项目曾被 The Changelog 播客第 143 期专题介绍（`src/about.njk` 内嵌播放器）。

除了英文主库，仓库最大的特色是 34 种语言的社区翻译（`src/translations/` 下 34 个目录，`src/_data/translations.json` 收录索引），从中文简繁体到阿拉伯语、缅甸语、希伯来语，包括 RTL 支持（`src/src/index.njk` 和布局中的 `rtl` 变量）。这让同一份内容覆盖了远超英语社区的受众，也是星数能累积到 6 万的重要原因之一。

2020 年前后项目从纯 README 仓库升级为一个 Eleventy 静态网站（`config/eleventy.config.js`、`package.json`），英文题库拆分为独立分类页，翻译统一挂到站点 `/translations/<lang>/` 路径下，同时保留 GitHub 上直接阅读的体验。

## 3. 技术栈与依赖

仓库的"代码"只是内容展示层，非常轻量（`package.json`）：

- **静态站点生成**：@11ty/eleventy 2.x + Nunjucks 模板引擎（`config/eleventy.config.js` 中配置 `templateFormats: ['njk', 'md', 'html']`），这也是 GitHub 语言统计显示 Nunjucks 的原因。
- **Markdown 渲染**：markdown-it 14 + markdown-it-anchor（为二级标题生成锚点，`config/eleventy.config.js` 第 7-17 行），EleventyI18nPlugin 处理多语言路由（默认语言 en）。
- **构建期压缩**：clean-css（CSS 内联压缩 filter `cssmin`）、uglify-es（JS 压缩 filter `jsmin`）、html-minifier（HTML 输出 transform `htmlmin`）——三者均在 `config/eleventy.config.js` 注册。
- **代码高亮**：@11ty/eleventy-plugin-syntaxhighlight（编码题里的 JS 代码块）。
- **辅助**：luxon（日期格式化 filter `html_date_string`）、all-contributors-cli（生成贡献者表格并复制到 `src/_data/contributors.json`，`package.json` 的 `contributors:generate` 脚本）。
- **前端运行时**：仅一个约 30 行的原生 JS 文件（`src/_includes/assets/js/app.js`，移动端导航开合 + 一句 console 彩蛋），无任何前端框架。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `src/questions/*.md` | 英文题库本体，9 个分类各一个 markdown 文件 |
| `src/translations/<lang>/README.md` | 34 种语言翻译，每个语言一个自包含的 README（旧版锚点格式），另有 `_template/` 供新翻译起步 |
| `src/_data/*.json` | Eleventy 数据层：分类索引（questions.json）、翻译索引（translations.json）、维护者（maintainers.json）、原始贡献者（og.json）、all-contributors 生成的贡献者表（contributors.json）、站点元信息（site.json） |
| `src/_includes/layouts/` | default.njk（SEO/OG/Twitter meta + 内联 CSS）、page.njk（内容页 + "Edit this page" 链接） |
| `src/_includes/components/` | navigation.njk（由 questions.json 数据驱动的导航）、footer.njk |
| `src/_includes/assets/` | 少量 CSS（按 header/content/footer/navigation 拆分，构建时内联）与 app.js |
| `config/eleventy.config.js` | Eleventy 唯一配置文件（filters/transforms/目录约定） |
| `.github/` | CONTRIBUTING、issue/PR 模板、CodeQL、gh-pages 部署工作流、dependabot、stale bot 配置 |

组织方式评述：典型的"内容优先"单仓结构——内容（markdown）与展示（Eleventy）共置一仓，数据驱动导航与翻译索引，无 monorepo/多包概念。翻译采用"每语言一个自包含 README"的扁平约定，新增翻译只需复制 `_template` 填空，门槛极低；代价是译文与英文主库没有条目级同步机制，内容漂移只能靠人工对齐。

## 5. 核心架构与关键实现

### 5.1 数据驱动的站点生成

所有导航和分类页都由 `src/_data/questions.json` 驱动：数组中每项含 `abbreviation/title/url/external` 四个字段，无障碍（A11y）分类通过 `external: true` 指向外部站点 Scott O'Hara 的 accessibility_interview_questions。导航组件直接遍历该数据（`src/_includes/components/navigation.njk`）：

```njk
{% for entry in questions.all %}
  {% if not entry.external %}
    <li ...><a ... href="{{ entry.url | url }}">{{ entry.abbreviation }}</a></li>
  {% else %}
    <li ... href="{{ entry.url }}" ...>{{ entry.abbreviation }}</a></li>
  {% endif %}
{% endfor %}
```

### 5.2 构建管线（config/eleventy.config.js）

配置文件本身就是全部"架构"：注册语法高亮与 i18n 插件、把 markdown-it 设为 md 渲染库并开启标题锚点；三个 filter/transform 分别完成 CSS 压缩、JS 压缩、HTML 全量压缩。`default.njk` 把全部 7 个 CSS 文件用一个 `{% set css %}` 块内联进 `<head>`（`src/_includes/layouts/default.njk` 第 45-53 行），因此最终站点零额外 CSS 请求——对一个内容页来说是很典型的"性能预算"做法。SEO/OG/Twitter 卡片 meta 全部由 `site.json` + 页面变量生成（`default.njk`）。

### 5.3 题库内容组织

英文题库按分类拆文件，条目用两级无序列表表达"主问题 + 追问"，例如 `src/questions/javascript-questions.md` 第 11-12 行：

```markdown
* What is the difference between a variable that is: `null`, `undefined` or undeclared?
  * How would you go about checking for any of these states?
```

每个文件带 Eleventy frontmatter（title/layout/permalink），经 page.njk 渲染并自动附带指向 `tree/master/src/...md` 的"Edit this page"链接（`page.njk` 用 `github_permalink` filter 把 .html 路径换回 .md）。`coding-questions.md` 是唯一特例：12 道以 "Question:" 开头的代码输出题，用 fenced code block 呈现。

### 5.4 多语言页面

翻译页沿用同一个 page.njk 布局，靠 frontmatter 的 `lang` 字段与 EleventyI18nPlugin 挂到 `/translations/<lang>/`（`src/translations/chinese/README.md` 第 1-5 行可见 `lang : zh`）。RTL 语言通过 body 的 `rtl` class 处理（`default.njk` 第 58 行）。

## 6. 构建与运行

来自 `package.json` scripts 与 `.github/CONTRIBUTING.md`：

```bash
npm install
npm start      # eleventy --serve --port 9090，本地预览 http://localhost:9090
npm run build  # eleventy --config=config/eleventy.config.js --pathprefix='Front-end-Developer-Interview-Questions/'
npm run contributors:generate   # 生成贡献者表格并同步到 src/_data/contributors.json
```

产物输出到 `_site`，由 `.github/workflows/gh-pages-build.yml` 在 push 到 main 时用 TartanLlama/actions-eleventy 构建、peaceiris/actions-gh-pages 发布到 gh-pages 分支（CI 中固定 Node 16.x）。

## 7. 内容收录与分类体系（内容型特例）

- **收录范围**：限定"前端通用面试题"——按 9 个分类统计（`src/questions/`）：JavaScript 49 条主问题、CSS 35、General 29、HTML 13、Network 7、Testing 5、Fun 4、Performance 3、Coding 12 道（`coding-questions.md` 的 "Question:" 计数），合计约 157 道主问题，外加 19 条嵌套追问（css 2 / general 3 / js 8 / network 6）。无障碍单列但外链到 Scott O'Hara 的专门题库（`questions.json` 中 `external: true`），体现"不重复造轮子"的收录观。
- **内容形态**：只有问题没有答案，且明确禁止在 issue/PR 提交答案（`CONTRIBUTING.md`）。题目以开放讨论型为主，Coding 分类则是固定答案的代码输出题。
- **分类体系**：稳定的扁平九分类，十余年基本未变；分类索引在 `questions.json` 中手工维护。
- **条目规模**：英文题库 markdown 合计 332 行（`wc -l src/questions/*.md`）；34 个翻译目录各自一个 200-330 行的自包含 README（如 `src/translations/chinese/README.md` 261 行、russian 329 行）。

## 8. 更新机制与翻译生态（内容型特例）

- **更新机制**：纯 PR 流程，规矩极简且刻意降门槛——"one PR per question"（`CONTRIBUTING.md` 第 6 行）、squash 成单 commit、模板文件照抄填空（`src/translations/_template/README.md`）。没有内容自动化校验（无链接检查、无译文同步 CI），质量靠人工 review。
- **自动化**：维护侧自动化集中在"周边"而非内容——dependabot 每月升级 npm 与 actions 依赖（`.github/dependabot.yml`）、CodeQL 每周扫描（`.github/workflows/codeql-analysis.yml`）、stale bot 30 天标记/37 天关闭 issue（`.github/stale.yml`）。
- **贡献者结构**：GitHub contributors API 分页推算约 227 人；README 的 all-contributors 表格展示约 86 人（`README.md`），翻译（🌍 标签）占大头。维护者三人：Darcy Clarke、Rob Larsen（现主要负责人）、Vasiliy vvscode（`src/_data/maintainers.json`）。
- **生态位**：h5bp（HTML5 Boilerplate）组织旗下项目，与 HTML5 Boilerplate、main 站点互为引流；它是"前端面试题"这一内容品类的开创者之一，后来大量中文/其他语言的面试题仓库都以其为蓝本。姊妹生态包括外链的 accessibility 题库与 The Changelog #143 播客（`about.njk`）。
- **为什么能涨到 6 万星**：2012 年上线即踩中前端行业爆发期+面试资源稀缺的窗口；h5bp 组织光环带来初始曝光；"无答案、开放讨论"的定位让它同时服务面试官与求职者两端；34 种翻译把同一份内容推向全球社区；贡献门槛低（改一行 markdown 即可 PR）持续制造传播事件。星数是十多年复利的产物。

## 9. 亮点与风险

**亮点**

1. 内容定位克制而聪明：只出题不出答案，规避了"答案争议"这一内容型仓库最大的维护负担（`CONTRIBUTING.md`）。
2. 极低贡献门槛的工程化：模板目录 + 数据驱动导航 + all-contributors 自动表格，翻译/加题几乎零学习成本（`src/translations/_template/`、`questions.json`）。
3. 站点工程质量超出预期：全 CSS 内联 + HTML 压缩 + 完整 SEO/OG meta + i18n + RTL + skip-link 无障碍支持（`default.njk`、`page.njk`）。
4. 社区记忆与治理透明：og.json/maintainers.json/about.njk 完整保留项目史与荣誉名单。

**风险/局限**

1. 内容更新实质停滞：main 分支自 2024-07 后仅 2026-03 一次 dependabot 提交（commits API）；JS/CSS 题目停在 2020-2022 时代的知识边界（如 `javascript-questions.md` 无 ES2020+ 新题）。
2. 译文漂移：34 种翻译与英文主库无条目级同步机制，`_template` 仍是旧版锚点格式，与 `src/questions/` 拆分后的新版结构已不一致（对比 `_template/README.md` 与 `src/questions/*.md`）。
3. 构建链老化：CI 固定 Node 16（EOL，`gh-pages-build.yml`），uglify-es 是已停止维护的库（package.json ^3.3.9），`resolutions` 里还钉着一个无依赖引用的 axios 0.18.1（`package.json` 第 50-52 行）。
4. 小瑕疵：`default.njk` 头部残留调试代码 `<script> console.log(page.lang) </script>`，会进生产页面。

## 10. 总结评价

这是内容型开源仓库的教科书样本：一份 2009 年起家、2012 年归入 h5bp 的前端面试题清单，靠克制的内容策略（无答案、开放讨论）、极低的贡献门槛（每题一 PR、翻译填空即成）和 34 种语言翻译，把一份 markdown 累积成了 6 万星的"前端面试事实标准"，如今它更像一个偶有人维护的历史丰碑而非活跃项目——构建链与内容都在老化，但作为面试素材和"如何运营内容型开源项目"的范例依然值得使用与研究。适合三类人：要面试前端候选人的面试官（直接挑题）、准备跳槽的前端工程师（自测清单）、以及想研究社区驱动内容仓库运营模式的开源维护者（读它的 CONTRIBUTING、翻译机制与贡献者体系比读代码更有价值）。
