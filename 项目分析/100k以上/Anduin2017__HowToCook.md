# Anduin2017/HowToCook 项目分析

> 星数 102999（快照 2026-10-24）｜主语言 —（Markdown 内容仓库，工具链为 Node.js）｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：**程序员在家做饭指南**——用形式化、可量化的"程序员友好"写法重写中文家常菜谱的开源内容仓库。

| 项目 | 事实 |
| --- | --- |
| 名称 | Anduin2017/HowToCook（程序员做饭指南） |
| 星数 | 102999（快照）；GitHub API 实测 103363（2026-10-30） |
| 语言 | 无代码主体；Markdown 菜谱 + Node.js 质检脚本 |
| 许可证 | The Unlicense（公共领域，`LICENSE`、`package.json` 均确认） |
| 创建时间 | 2020-02-29（gh api） |
| 最近活跃 | 最近 push 2026-10-26（gh api，距分析日 4 天），持续活跃近 7 年 |
| Forks / 贡献者 | 11174 forks；约 488 名贡献者（contributors API 分页推算） |
| 版本 | tag 1.3.0 → 1.6.0 共 5 个 release；`package.json` version 1.6.0 |
| 官网 | https://howtocook.aiursoft.com/（README.md） |

## 2. 项目是什么，解决什么问题

项目起源于作者的真实痛点（`README.md` 开篇）：网上的菜谱"写法千奇百怪，经常中间莫名出来一些材料"，对习惯了形式语言的程序员极其不友好。于是作者用**清晰、精准、可计算**的描述重新整理常见菜的做法，并明确希望它"由社区驱动和维护"。

它的核心方法论是把菜谱当成"可执行的规格说明"来写（`dishes/template/示例菜/示例菜.md`）：

- **固定四段结构**：必备原料和工具 → 计算 → 操作 → 附加内容；
- **所有用量以克/毫升计**，杜绝"适量""少许""一勺"这类模糊单位；
- **"计算"章节给出公式**（如"鸡蛋 = 1.5 个 * 份数，向上取整"），支持按人数缩放；
- **每个等待步骤必须给出结束判断标准或时间公式**，不许出现"你/我"等人称代词。

这种风格贯穿整个仓库，甚至延伸到生活决策本身：`tips/如何选择现在吃什么.md` 用不等式组 `a + b = N + 1; a ≤ b ≤ a + 1` 推导"几个人该做几个荤菜几个素菜"，并给出 JavaScript 代码实现；`tips/厨房准备.md` 建议追求仪式感的程序员采购"电子秤、游标卡尺、量筒、停表、烧杯、测温枪、移液器"——程序员的严谨与幽默是整个仓库的标志性气质。

除了菜谱正文，仓库还提供一条完整的学习路径（`README.md`）：从"厨房准备""如何洗碗"等入门须知，到 `tips/learn/` 下的"学习用刀""学习焯水""学习炒与煎"等单项技能，再到 `tips/advanced/` 的"油温判断技巧""糖色的炒制"等进阶内容，构成从零基础到进阶的阶梯。

浏览方式有两种：直接读 GitHub 上的 Markdown，或访问官方可视化网站 howtocook.aiursoft.com / 部署 Docker 镜像 `aiursoft/howtocookviewer`（`README.md` 提供命令，默认账号 admin，启动后 30 分钟内自动索引）。

## 3. 技术栈与依赖

内容本体是纯 Markdown + 图片，没有运行时代码；工程化工具链全部服务于**内容质量管控**（`package.json`）：

| 依赖 | 用途 |
| --- | --- |
| markdownlint-cli ^0.48.0 | Markdown 语法规范检查（`.markdownlint.json` 关闭了 MD013 行长等 7 条规则） |
| textlint ^15.7.1 | 自然语言 lint 框架 |
| textlint-rule-ja-space-between-half-and-full-width | 强制中英文/数字之间留一个空格 |
| textlint-rule-zh-half-and-full-width-bracket | 中文半角/全角括号规范 |
| glob ^7.2.0（devDependencies） | 供自研校验脚本遍历文件 |

自研 Node.js 脚本两个（`.github/`）：`readme-generate.js` 自动生成目录索引，`manual_lint.js` 执行菜谱格式校验（详见第 5 章）。CI 运行环境为 Node 22（`.github/workflows/ci.yml`）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `dishes/` | 菜谱主体，按 10 个分类建子目录，每道菜一个同名文件夹（md + 成品图） |
| `dishes/template/示例菜/` | 菜谱模板，新菜谱的唯一起点（README 贡献指南明确要求复制它） |
| `tips/` | 厨房通用知识：厨房准备、洗碗、决策吃什么 |
| `tips/learn/` | 12 篇基础技能教程（用刀、焯水、蒸、煮、空气炸锅等） |
| `tips/advanced/` | 4 篇进阶技巧（油温、糖色、辅料、专业术语） |
| `.github/workflows/ci.yml` | PR 触发：npm run lint |
| `.github/workflows/build.yml` | push 到 master 触发：重建 README + lint + 机器人自动提交 |
| `.github/manual_lint.js` | 约 300 行的菜谱格式校验脚本（仓库真正的"核心代码"） |
| `.github/readme-generate.js` | README 目录自动生成器 |
| `.github/templates/readme_template.md` | README 模板（占位符 before/main/after） |
| `README.md` | 由脚本自动生成的总索引（勿手改） |

组织方式是典型的**"内容即代码"（docs-as-code）单体仓库**：内容按分类物理分目录、每菜一目录一图；索引不靠人维护而由 CI 从目录结构重新生成（`readme-generate.js` 按 `zh-CN` locale 排序、硬编码 10 个分类的中文标题映射）；质量不靠人工评审兜底而靠机器校验前置。贡献流程（`.github/pull_request_template.md`）明确规定：**PR 提交后 1 分钟内得到自动化检查结果，只有通过自动检查的 PR 才会被人工审核**，且必须勾选"内容可声明进入公共领域"的 Unlicense 签署框。

## 5. 核心架构与关键实现

本仓库的"架构"是**格式规范 + 自动化执法**的组合，核心实现有三处：

**(1) 菜谱格式规范（`dishes/template/示例菜/示例菜.md`）**

模板用 HTML 注释逐段解释写作要求：标题必须是"菜名的做法"、难度 1-5 星的星级定义（1 星约 5 分钟即可完成、5 星需 40 分钟以上且易失误）、"计算"章节的两种场景（食堂大锅菜给公式、固定菜量给单份量）等。结尾强制保留一句话："如果您遵循本指南的制作流程而发现有问题或可以改进的流程，请提出 Issue 或 Pull request。"——把持续改进的入口写进了每一篇内容。

**(2) 格式校验器（`.github/manual_lint.js`，validators 数组共 11 项检查）**

这是仓库质量一致性的关键。摘录最"程序员味道"的一段：

```javascript
['适量', '少许'].forEach(w => {
  if (count(w) > 0) {
    errors.push(`文件 ${filePath} 不符合仓库的规范！${w} 不是一个精准的描述！请给出克 g 或毫升 ml。`);
  }
});
if (count('min') > 0) {
  errors.push(`文件 ${filePath} 不符合仓库的规范！min 这个词汇有多重含义。建议改成中文"分钟"。`);
}
['你', '我'].forEach(pronoun => {
  if (count(pronoun) > 0) {
    errors.push(`文件 ${filePath} 不符合仓库的规范！请不要出现人称代词。`);
  }
});
```

11 项检查覆盖：文件名不含空格、主标题必须为"X的做法"、恰好四个二级标题且顺序固定、难度星级 1-5、卡路里格式、"勺/杯"等不精准单位禁用（含"勺子/炒勺/漏勺"白名单排除）、基于份数而非人数的用量、必需的结尾句、菜谱描述 30-300 字、图片 alt 文本不得是拼音或"成品"这类泛词、**图片引用必须真实存在**、单文件不超 1MB。

**(3) README 自动生成（`.github/readme-generate.js`）**

递归收集所有 `.md`，按路径归入 10 个分类（tips 进 before 区、advanced 进 after 区），套用 `readme_template.md` 重写 README。配合 `build.yml` 中的 `stefanzweifel/git-auto-commit-action`，每次 push 到 master 自动提交"[ci skip] Automatic file changes/fix"——**贡献者永远不需要手写目录**，新增菜谱只需建目录写文件。

## 6. 构建与运行

来自 `package.json` scripts 与 `README.md` 的实际命令：

```bash
# 本地质检（贡献者用）
npm install
npm run lint          # = textlint --fix + markdownlint + manuallint 三连
npm run textlint      # textlint . --fix（自动修复中英文空格等）
npm run markdownlint  # markdownlint ./dishes ./tips
npm run manuallint    # node .github/manual_lint.js（自研 11 项格式校验）
npm run build         # node .github/readme-generate.js 重新生成 README

# 网站浏览（读者用，README.md）
docker pull aiursoft/howtocookviewer
docker run -d -p 5000:5000 aiursoft/howtocookviewer
# 默认用户名密码 admin / Admin@123456!，启动后 30 分钟内自动索引
```

## 7. 内容规模与分类体系

实际统计（2026-10-30 基于 WORK 目录源码快照，`find` 计数）：

| 分类（目录） | 菜谱数 |
| --- | --- |
| meat_dish 荤菜 | 111 |
| vegetable_dish 素菜 | 65 |
| staple 主食 | 59 |
| aquatic 水产 | 28 |
| breakfast 早餐 | 25 |
| soup 汤与粥 | 24 |
| drink 饮料 | 23 |
| dessert 甜品 | 19 |
| semi-finished 半成品加工 | 10 |
| condiment 酱料 | 9 |
| **合计**（不含 template） | **373** |

- 全仓 `dishes/` + `tips/` 共 **393 个 Markdown 文件、约 23662 行**；配图 355 张；仓库总体积约 111MB（图片占绝对大头）。
- 分类体系固定为 10 类，物理目录与 README 章节一一对应，由 `readme-generate.js` 硬编码维护；另有一条"常识/技能"线（tips 19 篇：厨房准备、洗碗、决策、12 篇 learn 教程、4 篇 advanced 进阶）。
- 收录范围以**中式家常菜为绝对主体**，兼顾少量西餐/日料/印度菜/饮品调酒，以及"微波炉/空气炸锅/电饭煲"等厨电器型的适配菜谱（如 `dishes/vegetable_dish/鸡蛋羹/` 下并存炒锅、微波炉、蒸箱三个版本）。
- 每篇菜谱统一带"预估烹饪难度（1-5 星）+ 预估卡路里"元数据（manual_lint.js 强制），使全库内容可比较、可筛选。

## 8. 更新机制与质量控制

- **更新机制**：纯 PR 驱动。流程为"复制示例菜模板 → 按规范写作 → 提 PR → 1 分钟内 CI（`ci.yml`：Node 22 + npm run lint）出结果 → 通过后才人工评审"。`build.yml` 在合入 master 后自动重建 README 并由 github-actions[bot] 自动提交，形成"人写内容、机器写索引"的分工。
- **质量控制是分层的**：textlint 管"语言表面"（中英文空格、括号全半角），markdownlint 管"语法结构"，manual_lint.js 管"内容语义"（单位精准性、结构完整性、图片引用有效性、文件大小），人工只做机器管不了的"菜谱是否真的能做对"判断。
- **活跃度证据**：最近 push 2026-10-26（gh api）；470 个 open issue 说明提案流量大、评审节奏偏保守；约 488 名贡献者、1.1 万 forks，是典型的"海量小额贡献"型社区——绝大多数贡献者一生只提交一两道菜。
- **版本节奏**：以 tag/release 标记里程碑（1.3.0 → 1.6.0 共 5 个 release），对一个内容仓库而言属于少见的"软件化"管理。
- **内容可信度声明**：PR 模板强制签署 Unlicense 条款，确保每篇菜谱原创且可进入公共领域，规避菜谱转载的版权风险。

## 9. 姊妹项目与生态位

README"衍生作品推荐"一节（`README.md` 末尾）列出了围绕本仓库生长的生态：

- **HowToCook-mcp / howtocook-py-mcp**（TypeScript / Python）：把菜谱库接入 MCP 协议，让 AI 助手直接"掌勺"推荐三餐；
- **whatToEat、是啊吃什么**：基于本仓库数据的"今天吃什么"决策工具；
- **图像化菜谱**（king-jingxiang.github.io/HowToCook）：在线预览与 PDF 导出；
- **厨房计划**（proj.kitchen）：开源中文菜谱 API；
- **Vibe Cook、COOKer-agent、Cannon Tales**：基于本仓库的沉浸式烹饪 App、按食材推荐助手、16 种语言的在线菜谱站。

生态位上，HowToCook 已成为**中文互联网的结构化中餐菜谱数据底座**：因为全库格式统一、单位精准、无版权负担（Unlicense），它天然适合被程序抓取、索引和二次开发——这是普通菜谱网站做不到的。官方可视化站与 Docker 镜像（aiursoft/howtocookviewer，README 徽章显示有 Docker pulls 统计）由作者所在 Aiursoft 团队维护，形成"内容仓库 + 官方 viewer"的双件套。

## 10. 为什么能涨到 10 万星与总结评价

**涨星逻辑**：第一，痛点普适且定位独特——"程序员做饭"是疫情（2020-02 创建）期间的完美时间胶囊，"用写代码的方式写菜谱"天然具备社交传播梗属性；第二，内容真的可用——四段式结构、"计算"公式、禁用模糊单位让新手能照做，而不只是乐一乐；第三，工程化极致——一个菜谱仓库做到了 lint、CI、自动索引、格式模板、贡献签署全链路，贡献门槛低（写一篇 md）但质量下限高（机器兜底），让 488 人协作的内容保持了惊人的一致性；第四，Unlicense 公共领域授权极大降低了二次传播和衍生开发的心理门槛，反过来又助推了星数。

**总评**：HowToCook 是"内容型开源项目工程化"的教科书案例——它证明了一个没有一行运行时代码的仓库，也可以用软件工程方法论（模板、lint、CI、自动化索引、语义化版本）把 373 篇社区菜谱管理得像一个严谨的代码库。它适合三类人：想学做饭的程序员（直接当教材用）、想构建菜谱类应用的开发者（当免费的公共领域数据集用）、以及想研究"docs-as-code 社区治理"的开源运营者（当架构案例读——重点看 `.github/manual_lint.js` 和两个 workflow）。

**风险与局限**：374 篇菜谱相对中餐全谱系仍是小样本，荤菜占 30% 而素菜/汤品偏少；"精确到克"的写法对火候、手感等经验性知识表达有限，5 星菜谱的形式化效果存疑；111MB 的图片体积使 clone/tarball 较重；人工评审速度跟不上提案流量（470 个 open issue），内容增长在放缓。
