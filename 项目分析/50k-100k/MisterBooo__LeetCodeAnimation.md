# MisterBooo/LeetCodeAnimation 项目分析

> 星数 76715（快照 2026-10-24）｜主语言 Java（见第 3 章口径说明）｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：**用动画（GIF/视频）逐题图解 LeetCode 算法题解思路的中文内容仓库**，2026 年已转型为 algomooc.com（"吴师兄学算法"）的公开索引与历史素材库。

| 项目 | 值 | 出处 |
| :-- | :-- | :-- |
| 名称 | MisterBooo/LeetCodeAnimation | GitHub 元数据 |
| 星数 | 76,715（fork 13,856 / watch 2,493） | 任务快照 + gh api |
| 创建时间 | 2018-12-06 | gh api `created_at` |
| 默认分支 | master，约 252 个 commit | gh api commits Link 头（page=253, per_page=1） |
| 最近推送 | 2026-06-12T13:49Z（本地解包文件 mtime 均为 2026-06-12，一致） | gh api `pushed_at` |
| 许可证 | **无**（`license: null`，仓库内亦无 LICENSE 文件） | gh api + 顶层目录清单 |
| 主语言 | GitHub 显示 Java（实为 `.gitattributes` 强制指定，见第 3 章） | `src/.gitattributes` |
| 主页 | https://www.algomooc.com | gh api `homepage` |
| 仓库体积 | 约 498MB（其中动画媒体约 750MB 解压前 515MB tarball） | gh api `size` + tarball 实测 |

## 2. 项目是什么，解决什么问题

LeetCodeAnimation 是中文算法学习圈最知名的"动画图解题解"项目之一：每道题配一篇中文题解文章（题目描述 → 题目解析 → 动画描述 → 多语言代码实现），核心卖点是**把指针移动、状态转移、递归展开这类不可见的过程画成动画**，让初学者"先看懂过程，再回到代码"。早期文章首发于公众号「图解面试算法」，同步博客 algomooc.com（`problems/0001-Two-Sum/Article/0001-Two-Sum.md` 头部声明）。

仓库本体分两层：
- **历史素材层**：`problems/` 下 110 个题目目录（109 个含 `Article/`、89 个含 `Animation/` 动画素材），加上 `docs/notes/` 的 69 篇早期题解文章，是 2018-2022 年间的存量内容（`src/problems/`、`src/docs/notes/` 实数）。
- **索引与同步层**：2026 年 6 月重构后，仓库定位改为"网站是源头，GitHub 是公开索引和门面"（`docs/HOW-TO-INTERACT.md` 原话）。`docs/data/manifest.json` 记录 256 道 LeetCode 动画题（71 简单/160 中等/25 困难，`docs/data/stats.json`），由脚本从网站侧 `study_index.js` 同步生成；README 中的题数数字也由脚本注入，禁止手改（`README.md` 的 `<!-- LCA-AUTOGEN:STATS -->` 标记区）。

README「2026 大版本进行中」区披露了网站侧的升级：题解配 Python/C++/Java 三语言参考实现（网站版）、AI 私教"小欧"、AI 合成的语音讲解，以及全站 299 题的质量清算（`README.md` 近况区；README 只声明网站侧能力，GitHub 端素材并未随之全量更新）。README 同时挂出三个带 UTM 追踪参数的转化按钮（在线交互版/按专题刷/AI 私教），全部指向 algomooc.com（`docs/readme-revamp/utm-links.md` 有完整 UTM 清单）。

能涨到 7.6 万星的逻辑：2018 年底入场的时机好（LeetCode 中文刷题潮起点）、"动画讲算法"这一形式在中文社区当时近乎空白、README 直接铺 GIF 预览传播性极强、再叠加公众号生态的稳定导流。星数的主要增长期在 2019-2021 年（从 Star History 徽章与 fork 量推断，未逐段核实）。

## 3. 技术栈与依赖

- **内容本体**：Markdown（184 个 `.md`）+ GIF/MP4/M4V 动画（97 gif、33 m4v、22 mp4，来自 tarball 全列表统计），媒体占仓库体积 98% 以上，最大单个 GIF 38MB（`problems/0084-Largest-Rectangle-In-Histogram/Animation/84.gif`，tar 清单实测）。
- **题解代码**：文章内嵌 C++/Python/Java 代码块；仓库文件形式存在的代码极少——仅 11 个 `.java`（`problems/0004…/Code/1.java` 等）。Java 11 个文件撑不起"主语言"。
- **Java 显示口径**：`.gitattributes` 写了 `*.js linguist-language=java`、`*.md linguist-language=java` 等四行（`src/.gitattributes`），把仓库几乎所有文本都标成 Java，GitHub 语言条因此显示 Java。这是历史遗留的语言统计"美化"，与真实技术栈不符。
- **工具链（Node.js，无第三方依赖）**：`tools/scripts/` 下 4 个零依赖 Node 脚本共 671 行——`sync-algomooc-index.js`（278 行，解析网站侧 `window.WS_PROBLEMS` 生成 manifest 与索引）、`validate-manifest.js`（74 行，schema/唯一性/URL 校验）、`build-readme.js`（169 行，`--write` 注入 README 数字段并产出 `stats.json`）、`review-site-changes.js`（150 行）。运行仅依赖 Node 20（CI 配置指定，`.github/workflows/manifest.yml`）。
- **脚手架（Python）**：`tools/anima.py` + `tools/anima/`（fire CLI，102 行），用于按模板生成新题解目录骨架（`tools/anima/model.py` 定义 `ProblemInfo`/`Solution`，含 Animation/Article/Code 三目录）。
- **无包管理锁文件**：`package.json` 只有 5 个 npm scripts，无 dependencies（`src/package.json`）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 | 出处 |
| :-- | :-- | :-- |
| `problems/` | 110 个题解目录，每目录 `Article/`（题解 md）、`Animation/`（GIF/MP4）、部分有 `Code/` | 实测统计 |
| `docs/notes/` | 69 篇早期中文题解文章（2018-2022 存量） | `src/docs/notes/` |
| `docs/data/` | `manifest.json`（256 条索引，单一数据源）、`stats.json`（脚本实算计数） | `src/docs/data/` |
| `docs/leetcode-animation-index.md` | 按题号索引（266 行，脚本生成） | `src/docs/` |
| `docs/index-by-topic.md` | 按专题索引（975 行、123 个专题，脚本生成） | `src/docs/` |
| `docs/sync-log.md` / `sync-workflow.md` | 同步留痕 / 网站与 GitHub 联动规则 | `src/docs/` |
| `docs/HOW-TO-INTERACT.md` | 维护者备忘（含发布红线） | `src/docs/` |
| `docs/readme-revamp/` | README 二期改造草稿、diff 说明、UTM 链接清单 | `src/docs/readme-revamp/` |
| `docs/assets/previews/` | README 用的 4 张网站版动画预览 GIF | `src/docs/assets/` |
| `tools/scripts/` | 4 个 Node 同步/校验/注入脚本 | `src/tools/scripts/` |
| `tools/anima/` | Python 题解目录脚手架 | `src/tools/anima/` |
| `.github/workflows/manifest.yml` | 唯一 CI workflow | 实测 |

组织方式评述：典型的"内容仓 + 轻工具链"结构，不是软件工程意义上的 monorepo。2026 年 6 月的重构（sync-log 首条记录为 2026-06-10）把生成物（索引 md、stats.json）与源数据（manifest.json）分离，并用 README 标记区把"数字段（脚本管）"与"转化层文案（人手管）"解耦，设计上明显是为人 + AI 会话协作维护准备的（`docs/HOW-TO-INTERACT.md` 写明本地工作路径 `/Users/wuzhibo/Desktop/Claude/LeetCodeAnimation-github` 与"发布需原话确认"的人审闸门）。

## 5. 核心架构与关键实现

内容型仓库，这里分析其**数据管线**而非业务代码。

**（1）manifest.json：单一数据源。** 256 条记录，每条含 `leetcodeId/slug/titleZh/difficulty/topicsZh/status/repoPath/gifPath/siteUrl`（`docs/data/manifest.json`，109KB；62 条有 `gifPath` 指向 `problems/*/Animation/*.gif`）。所有索引与 README 数字都从它派生，是整个仓库的"事实中枢"。

**（2）同步脚本从网站仓库拉数据。** `tools/scripts/sync-algomooc-index.js` 用正则从网站工程文件里抠出数据再渲染：

```js
function readProblems(file) {
  const source = fs.readFileSync(file, "utf8");
  const match = source.match(/window\.WS_PROBLEMS\s*=\s*([\s\S]*?);\s*$/);
  if (!match) throw new Error(`Cannot find window.WS_PROBLEMS in ${file}`);
  return JSON.parse(match[1]);
}
```
（`tools/scripts/sync-algomooc-index.js` 第 48-53 行附近；默认源路径为网站仓库的 `web/public/leetcode-animation/study_index.js`，说明 GitHub 仓库是下游消费方。）

**（3）README 数字段的"脚本接管"设计。** `tools/scripts/build-readme.js --write` 只改写 `<!-- LCA-AUTOGEN:STATS START/END -->` 标记之间的表格并生成 `stats.json`；README 顶部徽章用 shields `dynamic/json` 直接读 `stats.json` 的 `$.count`，README 源码里连数字字符串都不出现（`README.md` 第 91-111 行、`docs/readme-revamp/CHANGES.md` 的边界说明）。配套 `--check` 模式让 CI 能发现数字过期。

**（4）CI 作为一致性闸门。** 唯一 workflow 在每次 push/PR 时跑两步：`node tools/scripts/validate-manifest.js`（校验 slug 格式、难度枚举、siteUrl 域名、leetcodeId/slug 唯一性）和 `sync-algomooc-index.js --check-generated`（核对索引文件与 manifest 是否一致，`tools/scripts/validate-manifest.js`、`.github/workflows/manifest.yml`）。工程质量取向很明确：防手改、防漂移。

## 6. 构建与运行

无传统构建。来自 `package.json` 与 `docs/sync-workflow.md` 的实际命令：

```bash
npm run review:site   # 比较网站侧 study_index.js 与本地 manifest，建议 commit message
npm run sync          # 同步 manifest / 题号索引 / 专题索引 / sync-log
npm run validate      # 校验 manifest（Node 20，零依赖）
node tools/scripts/build-readme.js --write   # 注入 README 数字段 + 生成 stats.json

# 新题解脚手架（Python 3.6+，需 pip install -r tools/requirements.txt）
python tools/anima.py new 1 'Two Sum'
```

读者侧无需构建：直接浏览 README 或 `docs/index-by-topic.md`，点 GIF/链接到 algomooc.com 看交互版。

## 7. 内容质量与更新机制（测试与工程质量的特例观察）

- **内容规模**：256 道在线动画题（本仓库索引口径）+ 110 个历史题解目录 + 69 篇早期文章；专题覆盖 123 个分类标签，二叉树（22 题）、动态规划（21 题）为最大专题（`docs/index-by-topic.md` 头部与分节统计）。
- **文章模板稳定**：题目描述/解析/动画/代码四段式，代码带复杂度注释（如 `problems/0001-Two-Sum/Article/0001-Two-Sum.md`），质量受公众号打磨检验；但 2026 年后文章本体基本停止更新，只做索引同步。
- **无传统测试**：仓库没有代码测试目录，"测试"由两道 CI 校验（manifest schema + 生成物一致性）承担；对内容仓而言这是合理的最小保障。
- **更新的单一通道**：网站侧改内容 → 跑同步脚本 → CI 校验 → commit 规范（`docs: sync lc1234 ...`）在 `docs/sync-workflow.md` 全部写死，并对 README 维护设了"未拿到发布原话不 push"的人审红线（`docs/HOW-TO-INTERACT.md` 第 6 节）。
- **观察**：`docs/HOW-TO-INTERACT.md`、`docs/readme-revamp/CHANGES.md` 等文件行文像 AI 会话交付物（草稿/发布分离、黑话注释），2026 年的这轮 README 改造（转化层 + UTM + 折叠块）大概率是 AI 辅助完成的，工程质量反而优于多数个人内容仓。

## 8. 社区与维护状态

- **活跃度**：最近推送 2026-06-12（gh api），最后一批 commit 集中在 2026-06-10 至 06-12（同步初始化 + README 二期改造上线）；此前更新节奏未逐月核实，但 2018 年创建、仅 252 个 commit，说明是低频、大批量的更新模式。
- **贡献者**：21 人（gh api contributors 分页推算），高度集中于作者 MisterBooo（吴师兄）个人，属"个人品牌型"仓库。
- **社区互动**：23 个 open issue、13.8k fork（多为收藏式 fork）；README 末尾有 Star 引导与 star-history 图，但仓库无 CONTRIBUTING 流程（`docs/contributing.md` 实为 anima 脚手架使用说明），外部贡献基本不构成内容来源。
- **Release**：无 release 机制（内容仓不需要，gh api 未查 release，未确认）。
- **风险信号**：2026-06-12 后无推送（截至分析日近 5 个月），叠加内容重心已完全转向 algomooc.com，GitHub 端更新强度依赖网站侧运营。

## 9. 亮点与风险

**亮点**
1. **形式开创性**：中文社区最早的系统性"动画图解 LeetCode"仓库之一，GIF 直铺 README 的传播设计是它涨到 7.6 万星的核心原因（`README.md` 预览区）。
2. **数据工程超配**：零依赖 Node 脚本 + 单一数据源 manifest + CI 一致性闸门 + README 数字段脚本接管，把内容仓做出了数据管线的严谨度（`tools/scripts/`、`.github/workflows/manifest.yml`）。
3. **同步规则文档化**：什么改动要同步、commit 怎么写、红线是什么，全部落成可执行的文档（`docs/sync-workflow.md`、`docs/HOW-TO-INTERACT.md`），可交接性强。
4. **内容存量厚**：110 题动画素材 + 103 篇题解文章完整留在仓库，离线可用。

**风险/局限**
1. **无许可证**（`license: null`，无 LICENSE 文件）：法律上他人不能复用其文章与素材，对内容型仓库是实打实的减分项。
2. **语言标注失真**：`.gitattributes` 把 `.md/.js/.css/.html` 全部标成 Java，GitHub 语言条与真实技术栈严重不符（`src/.gitattributes`）。
3. **门面重于内容**：README 的转化层（UTM 按钮、AI 私教、语音讲解）宣传的是网站侧能力，GitHub 端素材并未同步升级，点进去可能产生预期落差。
4. **仓库 498MB**：媒体不分级存放，clone/下载成本高，对纯读者不友好。

## 10. 总结评价

LeetCodeAnimation 是"内容创意吃掉工程复杂度"的典型样本：2018 年靠一张张手画动画在中文刷题潮中快速积累起 7.6 万星，2026 年则完成向商业站 algomooc.com 的导流转型，GitHub 端被重新定位为"公开索引 + 历史素材 + 可校验数据"的下游镜像，并配了一套零依赖但相当规范的同步/校验/注入工具链。它的代码价值很低（11 个 Java 文件、671 行脚本），但对做技术内容运营的人，它的 README 转化层设计、数字段脚本接管和同步工作流文档是极好的参考；对刷题者，它仍是免登录即可浏览的高质量中文题解合集——只是要接受"最新最好的体验在站外"这一现实。适合刷题入门者读文章、适合内容创作者抄运营打法；想读"源码"的人会空手而归。
