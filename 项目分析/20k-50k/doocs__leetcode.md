# doocs/leetcode 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-17

> 注：本仓为内容型仓库（题解集合），按内容型仓库特例模板分析。gh api 2026-11-17 实测：36,770 stars / 9,481 forks / 0 open issues，最近 push 2026-11-16，许可证 CC-BY-SA-4.0，默认分支 main，官网 https://leetcode.doocs.org。

## 1. 项目速览

一句话定位：中文社区规模最大的多语言 LeetCode 题解库之一，覆盖 LeetCode 全题库、《剑指 Offer》两版、《程序员面试金典》及力扣竞赛题，每题提供 Python/Java/C++/Go/TypeScript/Rust 等十余种语言的实现与结构化题解文档，并有完整自动化部署的双语阅读站点。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | doocs/leetcode（Doocs 开源社区，yanglbme 出品） | gh api repos/doocs/leetcode |
| 星数 | 39,500（快照）；36,770（2026-11-17 gh api 实测） | 任务参数 / gh api |
| 主语言 | 内容为 17 种编程语言题解；工具链为 Python + Node | src/ 目录统计 |
| 许可证 | CC-BY-SA-4.0（内容署名-相同方式共享） | src/LICENSE |
| 最近活跃 | push 2026-11-16（分析前一日），日更节奏 | gh api、src/solution/4100-4199/ |
| 阅读站点 | leetcode.doocs.org（MkDocs 双语站 + Cloudflare Workers） | src/.github/workflows/deploy.yml、src/wrangler.jsonc |

## 2. 项目是什么，解决什么问题

刷算法题的人面临三个常见痛点：官方题解语言单一、中文优质题解分散且质量参差、书面题解缺乏统一的难度/标签/思路结构。doocs/leetcode 用"一题一目录、一目录多语言、一文档双结构（描述+解法）"的固定格式来系统化解决这些问题（见 `src/solution/template.md` 的 problem_readme_template：题目描述、按 `### 方法一/方法二` 分节、多语言代码以 `<!-- tabs:start -->` 标签页组织）。

收录范围非常完整，六条内容线全部基于实测目录：

- LeetCode 主题库：按题号分 42 个区段（`src/solution/0000-0099/` 到 `src/solution/4100-4199/`），实测 4,114 道题，每题一个目录，含 README.md + README_EN.md + 各语言 Solution.*，另有竞赛合集页 `src/solution/CONTEST_README.md`；
- 《剑指 Offer（第 2 版）》：75 题（`src/lcof/`，目录用中文题名，附带 lcof.json 元数据）；
- 《剑指 Offer（专项突击版）》：119 题（`src/lcof2/`）；
- 《程序员面试金典（第 6 版）》：109 题（`src/lcci/`，`01.01.Is Unique` 式编号目录）；
- 力扣竞赛题：LCP 82 题 + LCS 3 题（`src/lcp/`、`src/lcs/`）；
- 基础算法参考实现（`src/basic/`，99 个文件）：`basic/sorting/BubbleSort/`、`basic/searching/BinarySearch/` 等，每种算法同样多语言实现（BubbleSort.cpp/.cs/.go/.java/.js/.py/.rs）。

此外还有两条专项索引线：JavaScript 专项练习（`src/solution/JAVASCRIPT_README.md`）与数据库专项练习（`src/solution/DATABASE_README.md`，对应 378 个 .sql 文件），分别面向前端与数据岗位。仓库总计约 3.7 万个文件。

题解质量有明确工程约束：所有语言必须实现同一算法逻辑、代码必须通过对应语言的格式化工具（`src/AGENTS.md` "All language solutions must implement the same algorithm logic"）。从 2025 年起题解还引入了可选的"思考/Thinking"段（`src/scripts/check_thinking.py`、`src/AGENTS.md` Thinking section 一章）：在给出最终算法之前，用 3-6 句讲清"朴素想法为何不满足约束 → 瓶颈在哪 → 关键观察 → 为什么选择该数据结构"，把"怎么想到的"教给读者，这是它与多数题解仓在内容深度上的差异化点。

中文与英文双文档（每题 README.md / README_EN.md）加上 CN/EN 双站构建（`src/.github/workflows/deploy.yml` 的 zh/en matrix），让仓库同时服务国内面试人群与海外用户；README 中 "algos: Python | Java | C++ | Go | TypeScript | Rust | ..." 徽章与全部目录均为双语成对出现（`src/README.md`、`src/README_EN.md`）。

## 3. 技术栈与依赖

内容侧（题解本体）：

- 17 种语言：Python(4,699 文件)、Java(4,595)、C++(4,534)、Go(4,499)、TypeScript(3,724)、Rust(1,592)、JavaScript(829)、C#(592)、SQL(378)、Swift(355)、PHP(142)、Kotlin(21)、Ruby(14)、Scala(5)、Shell(4)、Nim(4)、Dart(2)（全仓 `find ... -name "*.ext"` 统计）
- 题解均为 LeetCode 平台标准签名（`class Solution` 风格），如 `src/solution/0000-0099/0034.../Solution.py`

工具链侧（仓库自带脚本，均为真实文件）：

- Python 3.13：`src/solution/util.py`（773 行，题解 README 生成引擎）、`src/solution/rating.py`（512 行，抓取力扣竞赛 Guardian/Knight 段位分数线）、`src/run_format.py`（clang-format/gofmt 封装）、`src/requirements.txt`（black 26.10.1、clang-format 23.1.5、PyYAML、requests）
- Node/pnpm：`src/package.json`（prettier、husky、lint-staged、commitlint；lint-staged 按语言路由到 prettier/black/clang-format/gofmt/rustfmt）
- 站点：MkDocs 双语构建（构建引擎放在 `docs` 分支，部署时 overlay 到 main，见 `src/.github/workflows/deploy.yml` 的 prepare job）
- 边缘：Cloudflare Workers 静态站托管 + `src/worker.js`（31 行，流式解压超过 25MiB 限制的搜索索引）
- CI：14 个 GitHub Actions workflow（`src/.github/workflows/`）

## 4. 目录结构与代码组织

| 目录 | 职责 | 规模（实测） |
| --- | --- | --- |
| `solution/` | LeetCode 主题库题解，按题号分 42 个区段目录 | 4,114 道题 / 33,785 个文件 |
| `lcof/` | 剑指 Offer（第 2 版），目录用中文题名 | 75 题 / 859 文件 |
| `lcof2/` | 剑指 Offer（专项突击版） | 119 题 / 1,042 文件 |
| `lcci/` | 程序员面试金典（第 6 版），`01.01.Is Unique` 命名 | 109 题 / 1,015 文件 |
| `lcp/` | 力扣竞赛题（LCP 系列） | 82 题 / 433 文件 |
| `lcs/` | 力扣竞赛题（LCS 系列） | 3 题 / 23 文件 |
| `basic/` | 基础算法参考实现（排序/查找） | 99 文件 |
| `solution/template.md` | 全部题解/索引页的 Markdown 模板（HTML 注释标记切分） | 约 470 行 |
| `solution/util.py`、`rating.py`、`rating.json`、`result.json` | 题解生成引擎、竞赛段位爬虫、评分数据（20,076 行）、全量题目元数据索引 | — |
| `scripts/` | `check_thinking.py`（思考段校验）、`preview.py`（本地 MkDocs 选题预览）、`run-py.js` | — |
| `.github/workflows/` | 14 个 workflow：部署、镜像、格式化、校验 | — |
| `worker.js`、`wrangler.jsonc` | Cloudflare Workers 部署与搜索索引特殊处理 | 31 行 / 15 行 |

组织方式评述：这是一个"数据驱动的内容 monorepo"。内容按题号平铺而非按算法分类（分类通过 README 的标签列与 `src/README.md` 的"算法提升专题"人工索引实现）；工程文件极少而规则极密——真正的"架构"不在代码里，而在模板标记、元数据索引和 CI 流程中。`docs` 分支单独存放站点引擎，main 只存内容与 lint 工具，部署时两分支叠加（`src/AGENTS.md` Branch model 一章），内容与站点外观解耦。

## 5. 核心架构与关键实现

（1）模板与标记系统——内容一致性的根基。`src/solution/template.md` 用 `<!-- 这里是xxx模板开始/结束位置 -->` 定义 8 套模板（中/英 × 通用/SQL/Shell/TS 竞赛等），题解页内用 `<!-- problem:start -->`、`<!-- description:start -->`、`<!-- solution:start -->`、`<!-- thinking:start -->` 等标记划段。机器可据此精确替换描述、表格、难度标签，人只写算法与代码，机器保证结构不漂移。

（2）题解生成引擎。`src/solution/util.py`：启动时 `load_ratings()` 从 `solution/rating.json`（缓存）+ zerotrac.github.io 外部数据源拉取每题竞赛评分，渲染进索引表；`refresh(result)`（util.py 第 720 行）遍历 `result.json` 里的全量题目元数据（question_id、paid_only、CN/EN url、标题 slug），对每题重渲染中英 README 并只在内容变化时写盘（`write_text` 带 Windows 文件锁重试，util.py 第 33 行）；`generate_category_readme` 重建 42 个区段页与 JAVASCRIPT/DATABASE 专项索引页的 Markdown 表格。贡献者本地跑 `pnpm run setup:python` 安装该引擎（`src/package.json`），维护者用它做新题批量初始化与标题/标签同步。

（3）双分支部署管线（本仓自动化的核心）。`src/.github/workflows/deploy-request.yml`：main 分支上 `solution/**` 等内容目录任何推送，等 90 秒静默期并确认仍是 HEAD 后 dispatch `deploy.yml`（防抖 + 防陈旧触发，deploy-request.yml 第 30-44 行）；`deploy.yml` checkout main + docs 两分支，rsync 叠加站点引擎，`python3 build_site.py` 生成页面树，zh/en 两个矩阵 job 并行 `mkdocs build`，产物同时发布到 GitHub Pages、Cloudflare Workers（`deploy-cloudflare` job，搜索索引 gzip -9 压缩以绕过 25MiB 资产上限），并 dispatch `publish-gitee.yml` 推 Gitee Pages；`sync-gitee.yml` 把 main/docs 分支 force-push 镜像到 gitee.com/Doocs/leetcode。一次内容合并全自动触发三平台发布。

（4）竞赛段位"答题卡"数据。`src/solution/rating.py`（512 行）分别处理 leetcode.cn（国内排名纯 CN）与 leetcode.com（全球榜混有国服账号，需按真实持有勋章者定界）的 Guardian/Knight 分数线抓取逻辑，结果写进 `solution/CONTEST_README.md` 的段位表（含守门员与"差一线"用户，见 `src/solution/template.md` contest_readme_template 与 `src/solution/rating.json` 20,076 行数据）。配合 `src/.github/workflows/update-starcharts.yml`（每 12 小时 cron 更新 star 趋势图）、`compress-images.yml`（每周日 cron 压图），构成仓库的"数据活性"外观。

## 6. 构建与运行

来自 `src/AGENTS.md`、`src/package.json`、`src/scripts/preview.py` 的实际命令：

```bash
pnpm install             # 安装 prettier/husky 等（pnpm-lock.yaml 锁定，pnpm@12.18.0）
pnpm run setup:python    # 安装 black、clang-format、题解引擎依赖（requirements.txt）
pnpm run preview         # 本地 MkDocs 预览（封装 scripts/preview.py）
# preview.py 支持选题：--problem 1,2,42 / --range 1-99 / --changed / --problem lcof/3

# 提交前格式化（husky + lint-staged 自动触发，也可手动）：
node scripts/run-py.js run_format.py                # 全量格式化
node scripts/run-py.js -m black -S <file.py>        # Python
node scripts/run-py.js run_format.py --clang-format <file>  # C/C++/Java
rustfmt --edition 2021 <file.rs>                    # Rust
```

站点生产构建不在本地执行，由 `deploy.yml` 在 Actions 内完成（需要 docs 分支引擎与 MKDOCS_API_KEYS）。无单元测试、无可执行构建——内容仓的"运行"即站点构建与格式化。

## 7. 收录质量与自动化工程

没有传统测试套件，取而代之的是一整套针对"内容一致性"的校验体系（全部为 `src/.github/workflows/` 真实文件）：

- 格式化：black-lint（Python）、clang-format-lint（C/C++/Java，LLVM 23）、gofmt-lint（Go，自动增删 `package main` 头）、rustfmt-lint（Rust）、prettier-check/prettier-write（JS/TS/PHP/SQL/MD；prettier-write 对同仓 PR 自动改写、对 fork PR 跳过并有不检出来源代码的安全注释）
- 内容校验：thinking-check（`scripts/check_thinking.py` 校验已有思考段非空、数量不超过方法标题数，允许缺失）
- 提交规范：commitlint（conventional）+ husky + `.github/pull_request_template.md` 自检清单（双语文档齐全、各语言同一算法、思考段规则）
- 数据维护：dependabot.yml 同时管 main 与 docs 分支的 npm/Actions/pip 依赖

代码质量观察：工具脚本本身写得相当工程化——`util.py` 处理 Windows 文件锁重试、CJK 标签双语切分（`split_topic_tags`）、HTML 列表清理等大量脏活；`worker.js` 用流式 DecompressionStream 解压响应，避免一次性载入大索引。题解代码风格统一、命名规范，README frontmatter（difficulty/tags）机器可读，可直接二次开发（如搜索、刷题卡片类应用）。

## 8. 社区与维护状态

- 活跃度：gh api 最近 10 次提交全部在 2026-11-13 至 11-16 之间，节奏为"维护者日更新题 + 周赛题解当天收录（weekly contest 528 于比赛次日收录）+ 社区 PR 补单语言"（如 No.3420 由三位不同贡献者分别补 Rust/TypeScript 等实现）。open issues 为 0，处理极快（gh api 实测）。
- 贡献者结构：gh api contributors 分页合计 206 人。典型"强核心 + 长尾众包"，Top 12（gh api 2026-11-17 实测）：

  | 贡献者 | 提交数 | 角色（据提交内容推断） |
  | --- | --- | --- |
  | yanglbme | 8,125 | 核心维护者，新题题解日更主力 |
  | YangFong | 667 | 核心 reviewers/题解批量补充 |
  | zhaocchen | 350 | 多语言题解补充 |
  | klever34 | 328 | 多语言题解补充 |
  | github-actions[bot] | 272 | 自动化机器人（starcharts、prettier 等） |
  | thinkasany | 188 | 题解补充 |
  | maolonglong | 176 | 题解补充 |
  | Jerring | 163 | 题解补充 |
  | rain84 | 153 | 题解补充 |
  | idoocs | 119 | 机器人/组织账号 |
  | bluesword12350 | 110 | 题解补充 |
  | Qiu-IT | 97 | 题解补充 |

  大量贡献者只贡献 1-5 次单语言题解（如 2026-11-13 的 No.3420 由 vremyavnikuda 补 Rust、mehdijony 补 TypeScript、sonuworks 补多语言）。C# 题解曾由 kfstorm 一次性贡献 245 号 PR（`src/README.md` 致谢段）。
- 生态位：doocs 组织（github.com/doocs）旗下还有 advanced-java、jmd、mdn 等系列仓，leetcode 是其旗舰；README 显示有 Open Collective 赞助与商业赞助位（APINEBULA），运营属性明显（微信交流群、赞赏码、版权声明要求商业转载授权）。
- 与其他 LeetCode 题解仓的差异：对比常见单语言题解仓（如只做 Python 的集合），本仓卖点在多语言齐备度 + 双语站点 + 新题收录速度（日更到 4100+ 题号）+ 思考段内容规范；爬虫/引擎脚本（util.py/rating.py）公开可复用也是少见的透明度。注：任务提示中的"GitHub Actions 定期抓取 LeetCode 新题"未在 workflows 中找到对应 cron——新题收录实际由维护者以自带爬虫脚本驱动并日更，定时自动化限于 starcharts/压图/部署链路（据 `src/.github/workflows/` 全量清单，如实修正）。

## 9. 亮点与风险

亮点：

1. 规模与完整度冠绝同类：4,114 道 LeetCode 题 + 386 题系列书题 + 竞赛题，约 3.7 万文件，且仍在日更（`src/solution/4100-4199/` 已收录 4114 题号）。
2. 多语言齐备：核心区段每题 Python/Java/C++/Go/TS 四到六种语言保底，部分题达 11 种（实测 `solution/0000-0099/0034.../` 目录）。
3. 模板化数据管线：HTML 注释标记 + result.json 元数据 + util.py 渲染引擎，让 3.7 万文件的结构在几年间保持完全一致，这是内容仓少见的工程能力。
4. 全自动三端发布：内容合并 → 双语 MkDocs 站 → GitHub Pages + Cloudflare Workers + Gitee，零人工干预（deploy.yml / deploy-request.yml / publish-gitee.yml）。
5. 内容规范演进：Thinking 段把"解题思路的推导过程"纳入可校验规范，提升教学价值而非只堆代码。

风险/局限：

1. 单点依赖强：8,125/约2万+ 提交集中在 yanglbme 一人，新题收录速度完全取决于核心维护者投入。
2. 无算法正确性验证：题解代码不经在线判题回测（仓库无测试设施），个别实现可能随 LeetCode 数据集更新而失效，靠 issue 驱动修复。
3. 版权与商用限制：CC-BY-SA-4.0 + "商业转载需授权"自定义条款并存，二次分发需留意（`src/LICENSE`、`src/README.md` 版权段）。
4. 题解深度不均：Thinking 段为可选项，老题与新题、热门与冷门题的解析详细程度差异明显。

## 10. 总结评价

doocs/leetcode 是"内容型开源仓库的工程化范本"：它用模板标记、元数据索引和 14 条自动化流水线，把一个本质上是 Markdown 集合的项目做成了结构稳定、双语、三端发布的算法知识库，多语言覆盖与收录完整度在中文社区几乎无出其右。它适合三类人：准备面试的刷题者（直接读站点或仓库）；想搭建同类题解/文档仓的维护者（util.py + template.md + deploy.yml 是现成蓝图）；以及想以"补一种语言实现"作为开源入门的低门槛贡献者（PR 模板与格式化钩子保证了贡献质量下限）。读它的源码价值不在算法本身，而在那套让内容规模化不失控的工具链。
