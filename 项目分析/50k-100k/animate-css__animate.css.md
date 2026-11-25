# animate-css/animate.css 项目分析

> 星数 82910（快照 2026-10-24）｜主语言 CSS｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：Animate.css 是"开箱即用"的纯 CSS 动画库——引入一个文件、给元素加一个 class（如 `animate__bounce`）就能播放预置动画，是 Web 前端历史上最经典的动画工具库之一。

| 项目 | 内容 |
| --- | --- |
| 名称 | animate-css/animate.css |
| 星数 | 82910（快照 2026-10-24，gh api 实测 82940） |
| 主语言 | CSS（源码为原生 CSS + PostCSS 工具链，非 Sass） |
| 许可证 | Hippocratic License 2.1（伦理许可，GitHub 标记 NOASSERTION） |
| 官网 | https://animate.style/（docs/ 目录即该站点源码） |
| 当前版本 | 4.1.1（2020-09-07 发布，package.json 同步为 4.1.1） |
| 最近推送 | 2024-07-29（gh api pushed_at），此后无提交，维护已停滞 |
| 创建时间 | 2011-10-12，至今约 15 年 |

## 2. 项目是什么，解决什么问题

Animate.css 解决的问题非常朴素：让开发者不做任何动画设计就能给页面元素加上专业水准的入场/退场/强调动画。传统做法需要手写 `@keyframes`、调 `cubic-bezier` 缓动曲线、处理 `transform-origin` 等细节，Animate.css 把这些沉淀为 97 个调好的动画，用户只需三步（README.md 与 docs/index.html 均有说明）：引入样式文件、给元素加 `animated` 基类、加具体动画类如 `animate__bounce`。

它有几条清晰的设计决策（出处见 `source/_base.css`、`source/_vars.css`）：

1. **类名前缀防冲突**：v4 起所有动画类加 `animate__` 前缀（package.json 的 `animateConfig.prefix` 配置），避免与业务样式冲突；同时提供无前缀的 `animate.compat.css` 兼容 v3 老用户。
2. **CSS 自定义属性做全局调速**：`:root` 暴露 `--animate-duration`、`--animate-delay`、`--animate-repeat` 三个变量，配合 `--animate-delay`/`--animate-repeat` 的 `calc()` 派生，实现不改源码即可整体调节动画节奏。
3. **工具类分层**：`.animated` 基类统一 `animation-fill-mode: both` 与时长；`.infinite`、`.repeat-1/2/3`、`.delay-1s~5s`、`.fast/faster/slow/slower` 提供常用的速度与延迟组合。
4. **无障碍内建**：`_base.css` 末尾通过 `@media print, (prefers-reduced-motion: reduce)` 将动画压缩到 1ms 并把所有 Out 类动画直接置 `opacity: 0`，尊重系统"减少动态效果"设置。

为什么能涨到 8 万+ 星：它诞生于 2011 年 jQuery 时代（作者 Daniel Eden），恰逢 CSS3 动画普及期，是"零 JS、一个 class 搞定动画"这一品类的事实开创者与标准答案；用法门槛几乎为零，被无数教程、模板、低代码平台默认集成，形成极强的网络效应。v3→v4 的前缀化重构与文档站（animate.style，带实时预览与复制按钮）进一步巩固了口碑。

## 3. 技术栈与依赖

项目本体是纯 CSS，工程化链路全部是构建期工具（无任何运行时依赖，`package.json` 的 `dependencies` 字段不存在，`devDependencies` 全部为构建/文档工具）：

- **PostCSS 8 生态**：`postcss-cli` + `postcss-import`（把 `@import` 内联成单文件）、`postcss-prefixer`（批量加 `animate__` 前缀）、`postcss-preset-env`（含 autoprefixer 与 custom-properties 降级）、`cssnano`（压缩）、`postcss-header`（注入版本/许可证头）。
- **文档站生成**：Node 脚本 `docsSource/index.js` + `compileMD.js` + `compileAnimationList.js`，用 `markdown-it` 把 Markdown 编译进 `template.html`，并解析入口文件的 `@import` 行自动生成动画分组列表（含复制按钮）。
- **工程辅助**：`npm-run-all`（脚本串联）、`husky` + `lint-staged` + `prettier`（提交时格式化）、`eslint`。
- **注意**：任务清单标注"Sass 源"与实际不符——v3 时代曾用 Sass，但 v4 源码（`source/` 目录）是原生 CSS + PostCSS，全部 101 个文件均为 `.css`（find 统计），无任何 `.scss`。

CI 仅有一条 CodeQL 安全扫描工作流（`.github/workflows/codeql-analysis.yml`），另有 dependabot 配置（`.github/dependabot.yml`）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `source/` | 动画唯一真源：`animate.css` 入口 + 19 个分类子目录，共 101 个 CSS 文件 |
| `source/animate.css` | 入口文件，99 条 `@import` 按分类注释排列，决定打包内容与顺序 |
| `source/_vars.css` | 三个全局自定义属性定义 |
| `source/_base.css` | `.animated` 基类族 + prefers-reduced-motion 无障碍规则 |
| `source/<分类>/<动画名>.css` | 一个文件一个动画：`@keyframes` + 绑定类，如 `attention_seekers/bounce.css` |
| `animate.css` / `animate.min.css` | 构建产物：开发版（95KB）/ 生产压缩版（72KB） |
| `animate.compat.css` | 无前缀兼容版（71KB），供 v3 老用户平滑迁移 |
| `docs/` | animate.style 官网静态站点（GitHub Pages，含 CNAME），`animate.min.css` 同时被用作站点自身动画 |
| `docsSource/` | 文档站生成脚本：解析 `@import` 生成动画清单、编译 MD、填充模板 |
| `postcss.config.js` | 构建中枢，按 `--env`（development/production/compat）切换插件行为 |
| `package.json` | 版本、prefix 配置、构建脚本、files 白名单（`style` 字段指向包入口） |
| `.github/` | CodeQL 工作流、issue 模板、dependabot |

组织方式评述：单仓库单包、扁平清晰。"一个动画一个文件 + 入口集中 `@import`"的物理拆分让 PR 互不冲突、按需取用容易（用户可只拷贝需要的文件），文档清单直接从入口文件正则解析生成（`docsSource/compileAnimationList.js`），保证文档与源码不会失联——这是小工具库工程化做得相当到位的范例。

## 5. 核心架构与关键实现

### 5.1 动画文件的原子化模式（source/attention_seekers/bounce.css）

每个动画都是"keyframes + 类绑定"的最小单元，动画名即类名即文件名：

```css
@keyframes bounce {
  from, 20%, 53%, to {
    animation-timing-function: cubic-bezier(0.215, 0.61, 0.355, 1);
    transform: translate3d(0, 0, 0);
  }
  40%, 43% {
    animation-timing-function: cubic-bezier(0.755, 0.05, 0.855, 0.06);
    transform: translate3d(0, -30px, 0) scaleY(1.1);
  }
  /* 70% / 80% / 90% 略 */
}
.bounce {
  animation-name: bounce;
  transform-origin: center bottom;
}
```

值得注意的实现细节：大量使用 `translate3d`/`scale3d` 强制 GPU 合成层避免卡顿；关键帧只动 `transform` 与 `opacity` 这两个可合成属性，是性能友好的写法。个别动画（如 `tada`，source/attention_seekers/tada.css）甚至不需要 `transform-origin`，类绑定只剩一行 `animation-name`。

### 5.2 可调速度系统（source/_base.css + source/_vars.css）

三个 CSS 变量 + `calc()` 派生出整套调速工具类，运行时改一个变量即可全局变速：

```css
:root {
  --animate-duration: 1s;
  --animate-delay: 1s;
  --animate-repeat: 1;
}
.animated.faster { animation-duration: calc(var(--animate-duration) / 2); }
.animated.slower { animation-duration: calc(var(--animate-duration) * 3); }
.animated.repeat-2 { animation-iteration-count: calc(var(--animate-repeat) * 2); }
```

这是 v4 相对 v3（Sass 变量编译期定制）最大的体验升级：从"改源码重编译"变成"运行时覆盖 CSS 变量"。

### 5.3 构建中枢 postcss.config.js

一个文件按 `--env` 驱动四种产物（development/production/compat/docs），核心逻辑：

```js
module.exports = (ctx) => {
  const prefix = ctx.env === 'compat' ? '' : animateConfig.prefix;
  return {
    plugins: {
      'postcss-import': {root: ctx.file.dirname},
      'postcss-prefixer': {prefix, ignore: [/\[class\*=.*\]/]},
      'postcss-preset-env': {features: {'custom-properties': true}},
      cssnano: ctx.env === 'production' || ctx.env === 'compat' ? {} : false,
      'postcss-header': {header},
    },
  };
};
```

三个巧思：prefixer 对 `[class*=...]` 选择器豁免，避免 `.animated[class*='Out']` 这类无障碍选择器被错误加前缀；compat 环境不加前缀且额外开启 custom-properties 降级（实测 `animate.compat.css` 同时内联了字面值与 `var()` 两份声明）；头部注释从 package.json 动态读版本号，产物自描述。

### 5.4 文档与源码同源生成（docsSource/compileAnimationList.js）

官网的动画清单不是手写的，而是用正则解析入口文件的 `@import` 与分类注释自动生成分组 HTML 列表，配合 `docs/main.mjs` 实现点击演示、复制类名、暗色模式。文档清单与实际打包内容天然一致，杜绝"文档里有但库里没有"的漂移。

## 6. 构建与运行

来自 package.json scripts（未实际运行，仅静态核对）：

```bash
npm install            # 安装构建依赖（无运行时依赖）
npm start              # raw + prod + compat，一次性产出三种 CSS
npm run dev            # 监听模式构建 animate.css
npm run docs           # docs:library + docs:pages，重建 docs/ 站点
npm run format         # prettier 全量格式化
```

npm 包发布内容经 `files` 白名单收敛为 `animate.css`、`animate.min.css`、`animate.compat.css`、`source/**/*.css`；`version`/`postversion` 脚本把"构建 + 提交产物 + push + push tags"串成发版流水线。消费者侧：CDN 引入 `animate.min.css` 或 npm 安装后直接 import，加 `animate__animated animate__bounce` 两个类即可。

## 7. 测试与工程质量

- **零自动化测试**：package.json 无 `test` 脚本，仓库无任何测试目录/测试框架（全量文件核查）。对一个纯视觉 CSS 库而言属常见取舍——动画效果本质需人眼/浏览器验证，但也意味着回归只能靠构建 + 发布后观察。
- **CI 仅 CodeQL**（`.github/workflows/codeql-analysis.yml`）：只做安全扫描，不跑构建验证。
- **代码质量约束靠格式化与人工规范**：husky + lint-staged 提交时 prettier 格式化；CONTRIBUTING.md/README 明确贡献两条规则——camelCase 命名并归入正确分类、必须附 CodePen 演示。
- 源码本身风格高度一致（每文件一个 keyframes、两空格缩进、单引号），101 个文件的结构几乎完全机械对称，可读性极好。

## 8. 社区与维护状态

- **贡献者规模**：contributors API 分页头显示约 130 位贡献者（per_page=1 共 131 页，含匿名），另 README 列出核心团队 3 人：Daniel Eden（作者）、Elton Mesquita（维护者）、Waren Gonzaga（核心贡献者）。
- **发布节奏**：最近一次 release 为 v4.1.1（2020-09-07），此前 v4.0.0/v4.1.0 集中在 2020 年 5 月。**代码最后一次推送为 2024-07-29（gh api pushed_at）**，即 2020 年后仅零星维护，2024 年 7 月起完全停滞；当前 open issues 89。
- **为何停滞仍被广泛使用**：CSS 动画类名 API 已高度稳定，库的功能面接近"完成态"，无适配新语法/新框架的刚性需求；存量集成（CDN、教程、模板）持续带来曝光。但 issues 无人回应、依赖（postcss 生态）逐年升级无跟进，属于"事实冻结"状态。
- 官网 animate.style 由 `docs/CNAME` 配置，站点内容随仓库提交更新（最后更新 2024-03，docs/ 目录文件 mtime）。

## 9. 亮点与风险

**亮点**

1. 极简 API 设计教科书：基类 + 动画类两个 class 覆盖 97 种动画，学习成本趋近于零（package.json `animateConfig`、docs/index.html）。
2. 运行时可调的变量化调速体系，v4 用 CSS 自定义属性取代 v3 的编译期定制（source/_vars.css、source/_base.css）。
3. 工程化精炼但完整：postcss.config.js 一个文件按环境驱动 4 种产物，prefixer 豁免规则、compat 双值降级等细节考虑周全。
4. 文档清单从源码入口自动生成，文档与实现零漂移（docsSource/compileAnimationList.js）。
5. 无障碍与性能基线内建：prefers-reduced-motion 全量降级、只动 transform/opacity、translate3d 合成层优化（source/_base.css、bounce.css）。

**风险/局限**

1. 维护停滞：最后 release 2020-09、最后 push 2024-07，issue 积压 89 个，未来浏览器兼容问题与新特性无人跟进（gh api 元数据）。
2. 许可证特殊：Hippocratic License 2.1 含"不得用于侵犯人权用途"条款，非 OSI 标准开源许可证，企业合规审查时可能被标注（LICENSE、package.json license 字段，GitHub 标 NOASSERTION）。
3. 全量引入约 97 个动画对多数场景偏重，且不支持 tree-shaking（CSS 按需需用户手动拆文件，source/animate.css 单入口设计）。
4. 小瑕疵佐证维护力下降：source/attention_seekers/shake.css 存在于源码但未被入口 `source/animate.css` @import（入口只引入了 shakeX/shakeY），打包产物 97 个 keyframes 与源码 98 个动画文件相差 1，属于无人修正的漂移。

## 10. 总结评价

Animate.css 是 CSS 动画库这一品类的定义者：15 年历史、8 万+ 星、97 个精心调校的动画和"两个 class 搞定"的 API，使它成为前端工具链里门槛最低、渗透率最高的组件之一。工程上它展示了小而美工具库的标准答案——原子化源文件、单文件构建中枢、文档同源生成、无障碍内建；但 2024 年 7 月后仓库进入事实冻结，v4.1.1 可能就是终版。它依然适合任何需要在页面里快速加动效的开发者直接使用（记得评估 Hippocratic 许可证的合规性）；对想读源码的人来说，它是学习"如何把一个纯 CSS 集合工程化"的最佳样本，重点看 source/animate.css 的组织方式与 postcss.config.js 的环境化构建，半天即可读完全部核心。
