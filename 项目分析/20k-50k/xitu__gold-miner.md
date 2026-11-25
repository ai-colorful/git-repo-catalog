# xitu/gold-miner 项目分析

> 星数 34376（快照 2026-10-24）｜主语言 无（GitHub languages API 返回空对象，纯 Markdown 内容仓库）｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

**掘金翻译计划**：中文互联网规模最大的 IT 技术文章英译中社区之一，以 GitHub 为协作平台、掘金（juejin.cn）为发布渠道，曾产出 4000 余篇译文与 13 个官方文档中文版。

| 项目 | 值 | 出处 |
|------|------|------|
| 名称 | xitu/gold-miner | gh api repos/xitu/gold-miner |
| 星数 / Fork | 34376 / 4969 | 同上（2026-11-21 查询） |
| 创建时间 | 2015-11-04 | 同上 |
| 语言 | 无代码，纯 Markdown（languages={}） | gh api repos/xitu/gold-miner/languages |
| 许可证 | 无 LICENSE 文件（license: null），README 末尾有版权声明 | `README.md`、gh api |
| 默认分支 | master（唯一分支），最后提交 2024-01-21 | gh api branches/commits |
| 最近推送 | 2024-04-17（pushed_at），实质已停更 | gh api |
| archived | false（未归档，但也未再维护） | gh api |
| 仓库体积 | 约 80MB，2580 个 .md 文件 | gh api size=80499；tarball 内 find 统计 |

## 2. 项目是什么，解决什么问题

gold-miner 是掘金社区的翻译基础设施：把海外优质技术文章（来源为掘金平台收录的英文分享）由社区译者翻译成中文，覆盖区块链、AI、Android、iOS、前端、后端、设计、产品、算法等领域，并延伸出 TensorFlow 中文文档、JavaScript Tutorial 中文版、系统设计入门教程等 13 个整册翻译子项目（`README.md` "官方文档及手册" 一节）。它解决的问题是 2015-2022 年间中文技术社区优质一手内容匮乏：由平台组织选题，用积分激励把散落的个人译者组织成"翻译—校对—审核—发布"的流水线。

其运作模式是"GitHub 工作台 + 掘金发布渠道"的组合：译文在仓库内以 Markdown 协作完成，定稿后发布到掘金平台，各分类索引文件（如 `front-end.md`）中的条目大多直接链接 juejin.im / juejin.cn 的文章页。README 自述"翻译完成 4000 余篇文章、官方文档及手册 13 个、1500 余名译者"（`README.md` 第 7 行附近）。仓库内实际保存的译文正文约 2560 篇（见第 7 章统计），其余早期译文以掘金站内链接形式收录在索引中。

掘金翻译计划是"平台官方组织 + 社区众包"翻译模式的代表样本：xitu 组织即掘金社区的 GitHub 组织（README 自述文章来源为掘金、homepage 指向 juejin.cn 标签页、README 合作伙伴为图灵教育），译者通过微信群维系、通过积分兑换礼品（`integrals.md` 大量"减去积分"的奖品兑换记录）。

## 3. 技术栈与依赖

- **无工程代码**：languages API 返回 `{}`，仓库全部内容为 Markdown 与少量 CI 配置（tarball 顶层清单核实）。
- **GitHub Actions（唯一的"技术栈"）**：3 个 workflow（`.github/workflows/`）——`generate-catalog.yml` 用 Python（lxml/requests/markdown）配合姊妹仓库 `xitu/juejin-integral-database` 的脚本自动生成分类目录；`stale.yml` 自动关闭 7 天无活动的"文章推荐"issue；`translator-application.yaml` 自动回复译者申请。
- **无 LICENSE 文件**：版权声明写在 README 末尾（"译文仅用于学习、研究和交流……转载前请联系译者或管理员获取授权"）。

## 4. 目录结构与代码组织

| 路径 | 职责 |
|------|------|
| `README.md` | 项目门面：各板块近期译文精选 + 指向 10 个分类索引 |
| `front-end.md` `backend.md` `android.md` `ios.md` `design.md` `AI.md` `others.md` `product.md` `blockchain.md` `algorithm.md` | 10 个分类目录（自动生成的译文索引） |
| `integrals.md` | 译者积分表，936KB / 12103 行，全仓最大文件 |
| `article/` | 2020 年起按年份归档的译文（2020/2021/2022/2023/ECMA-TC39） |
| `TODO/`、`TODO1/` | 2016-2019 年代早期译文的平铺存放目录（各约 1000 篇；文件均为完成稿，并非"待办"） |
| `glossary/` | 术语表（仅 blockchain.md，指向 wiki） |
| `a-new-post-template.md` | 译文头部元数据模板（原文地址/作者/译者/校对者/永久链接） |
| `CODE_OF_CONDUCT.md` | 行为准则 |
| `.github/` | CONTRIBUTING、2 个 issue 模板、PR 模板、3 个 Actions workflow |

组织方式评述：典型的"根目录索引 + 平铺/按年归档正文"结构。早期（TODO/TODO1）用英文 slug 平铺文件名，2020 年起改按年份归档，2022 年后不再新建结构；分类索引文件由 CI bot 自动生成再提交，人只写译文正文。

## 5. 核心架构与关键实现

内容型仓库，本章改述其"协作流水线"这一核心设计。

### 5.1 三段式协作流程（推荐 → 认领 → 翻译/校对 → 合并发布）

- 选题入口是规范化 issue 模板：`.github/ISSUE_TEMPLATE/recommendation.md`（"推荐优秀英文文章"，要求先搜索确认无现存中文译本、单次不超过 3 篇）；`.github/ISSUE_TEMPLATE/sign_up.md` 是译者申请表（公司/英语水平/翻译经验/方向）。
- `.github/CONTRIBUTING.md` 规定硬约束：**每个 PR 只允许一篇文章**、"译文请严格遵从译文排版规则指北"、推荐分支名 `translation/文件名`、"在原行上进行修改，以保证行与行之间的原始对应关系"（便于校对 diff）。
- PR 模板仅一句话："译文翻译完成，resolve #id"（`.github/PULL_REQUEST_TEMPLATE.md`）——一个 PR 绑定一个认领 issue，形成可追溯的任务闭环。
- 译文头部元数据由 `a-new-post-template.md` 固定：原文地址、原文作者、译者、校对者、本文永久链接，末尾带掘金翻译计划推广块。

### 5.2 积分制激励（`integrals.md`）

积分表按译者分节共 883 人（`grep -c '^## 译者' integrals.md`），全文 12103 行积分流水，其中明确标注"翻译"的 2563 条、"校对"的 4262 条，其余为积分增减/奖品兑换（如"2月份兑换礼品 -15"、"三月翻译计划奖品 Octocat 大猫 -30"）。翻译一篇 5-8 分、校对 1 分，积分可兑换礼品——这是把志愿翻译组织成持续产能的核心机制，也使本文成为社区贡献史的一手档案。

### 5.3 自动化流水线（`.github/workflows/`）

- `generate-catalog.yml`：当 `integrals.md` 变动时触发，checkout 姊妹仓 `xitu/juejin-integral-database`，运行 `script_generate_catalog.py` 按标签生成 9 个分类索引并自动提交（提交者固定为 lsvih），随后 `reindex.py` 重排 README"近期文章列表"。即"积分流水 → 目录"全自动。
- `stale.yml`：每天两次扫描，"文章推荐"标签的 issue 7 天无活动即关闭（防止选题积压）；PR 不受影响。
- `translator-application.yaml`：监听"申请成为译者" issue，自动关闭并回复统一流程——学 wiki 教程、加管理员微信 `chnyifan` 验证、进译者群、**先校对至少 1 篇才能开始翻译**。准入把关由 GitHub issue + 微信群人工完成。

### 5.4 质量控制链

译文需经译者自校 + 至少一名校对者（见 5.3 准入规则与 `TODO/2018-design-trends.md` 等文件头部的"译者/校对者"署名），最终由管理员（长期为 lsvih，见 workflow 提交者配置）审阅合并。排版规范放在 wiki（译文排版规则指北），仓库内 `CONTRIBUTING.md` 只留硬规则。

## 6. 构建与运行

无可构建产物，使用方式即阅读：

```bash
git clone https://github.com/xitu/gold-miner.git   # 或直接在 GitHub 网页浏览
```

唯一"构建"是 CI 的目录再生成（见 5.3），由 `integrals.md` 的 push 触发，人工无需干预；本地不存在可运行的脚本（生成脚本在姊妹仓 `xitu/juejin-integral-database`，不在本仓）。

## 7. 内容规模与分类体系（替代"测试与工程质量"）

**正文规模**（tarball find 统计）：

| 存放位置 | .md 数 | 说明 |
|----------|--------|------|
| `TODO/` | 1003 | 2016-2019 早期译文 |
| `TODO1/` | 1011 | 同上，第二批 |
| `article/2020/` | 201 | 按年归档开始 |
| `article/2021/` | 273 | |
| `article/2022/` | 42 | 骤降 |
| `article/2023/` | 10 | 最后一年，全部为 GPT/LLM/Next.js 主题 |
| `article/ECMA-TC39/` | 21 | TC39 流程/提案译文专题 |
| 全仓合计 | 2580 | 含 13 个根目录索引/说明文件 |

**分类目录条目数**（`grep -c '^\s*[*-] \['` 各索引）：front-end 939、backend 420、android 299、ios 220、design 167、AI 144、others 143、product 58、blockchain 33、algorithm 23，合计约 2446 条；多数条目指向掘金站内链接（gold.xitu.io/juejin.im/juejin.cn 三个时期域名并存），README 自称全站累计 4000+ 篇。分类体系随热点演进：区块链分舵独立成仓（`xitu/blockchain-miner`，README 头部横幅），2023 年最后批次全部转向 AI/LLM（`article/2023/` 文件名核实）。

**质量机制**：翻译+校对双署名、排版指北、一 PR 一文、管理员终审；无自动化内容质量检测（无 lint/链接检查类 CI）。

## 8. 社区与维护状态

- **贡献者**：contributors API（含匿名）最后一页为 633，即约 633 人直接提交过；积分表登记 883 人；README 自称 1500+ 译者（含区块链分舵等渠道）。三个口径并存，均为真实来源。
- **任务总量**：PR 编号累计至 #9566（约 9.5k 个 issue+PR 混合编号，gh api）。
- **生命周期时间线**（gh api + 目录结构，客观记录）：
  - 2015-11-04 建仓；2016-2021 为产能高峰（TODO/TODO1/article/2020-2021 合计约 2488 篇存量）；
  - 2022 年产出骤降至 42 篇，2023 年仅 10 篇；
  - 最后一次合并 PR 为 #9566（2024-01-21，"为什么命名是编写整洁代码的首要技能"），此后 master 无新提交；
  - pushed_at 2024-04-17 为仓库最后一次任何推送，此后至今（2026-11）无活动。
- **冻结状态**：仍有 1 个未合并 PR（#9563，2023-11-29 创建，状态 open，无人处理）；2024 年后的 issue 全部是无人受理的"申请成为译者"（#9585-#9593，创建于 2024-02 至 2026-06，全部 open）——translator-application bot 也已不再自动关闭申请（推测为仓库 60 天无活动后 GitHub 自动停用 Actions，未在源码中直接证实）。
- **生态联动停摆**：目录自动化的数据源仓 `xitu/juejin-integral-database` 最后推送 2023-02-02；xitu 组织下多数仓库 2022-2024 年后停更（org repos API）。项目未 archived、未发布停更公告（源码内未见），处于"事实死亡"状态。掘金平台自身经历出售与转型（背景推断，非源码证据）。

## 9. 亮点与风险

**亮点**

1. 大规模志愿翻译协作的完整范式样本：issue 认领、一 PR 一文、双署名校对、积分激励、CI 自动目录，全链路文档与模板齐备，可直接被同类社区复用。
2. 约 2560 篇可直接在仓库内阅读的译文存量，前端板块尤其厚（939 条索引）；`article/ECMA-TC39/` 是少见的 TC39 流程中文专题。
3. 13 个整册翻译子项目生态（TensorFlow 文档、javascript-tutorial-zh、system-design-primer 等，`README.md` 官方文档及手册一节）。
4. `integrals.md` 12103 行积分流水是中文技术社区志愿激励机制的罕见一手档案。

**风险/局限**

1. 已实质停更近三年（最后合并 2024-01-21），2024 年后的译者申请全部无人受理，新读者应将其视为静态存档。
2. 大量索引条目指向 gold.xitu.io / juejin.im 旧域名，链接腐烂风险高；早期译文图片多外链 behance/medium 等源站，随时间失效。
3. 无 LICENSE 文件，只有 README 版权声明（非商业转载需授权），二次分发的合规边界模糊。
4. 结构债：TODO/TODO1 两个数千篇文件的平铺目录无年份/分类维度，检索完全依赖根目录索引文件。

## 10. 总结评价

gold-miner 是中文技术内容众包翻译模式巅峰期的代表作：以 GitHub issue/PR 为工单系统、积分表为激励账本、Actions 为目录引擎，曾在六年里组织数百名译者产出数千篇译文并孵化十余个整册文档翻译项目。它如今最准确的身份是"活着的档案馆"——未归档但已冻结，2025 年后仍不断有新人提交译者申请却无人应答，本身就是平台社区退潮的注脚。适合两类人：一是想读 2016-2022 年高质量中文技术译文的读者（直接按根目录索引检索）；二是研究开源社区运营/众包翻译机制的人（`.github/` 模板、workflow 与 `integrals.md` 合起来就是一份完整的运营手册）。想参与翻译的人则不应再选择此项目。
