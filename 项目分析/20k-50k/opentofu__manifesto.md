# opentofu/manifesto 项目分析

> 星数 35557（快照 2026-10-24）｜主语言 HTML｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：The OpenTF Manifesto——2023 年 HashiCorp 将 Terraform 从 MPL 转为 BSL 许可后，社区发起的集体抗议宣言网站（opentf.org），以"要求回退许可，否则 fork 进基金会"的最后通牒式两步计划，最终催生了 OpenTofu，是开源历史上罕见的"抗议成功"完整档案。

| 事实项 | 内容 | 出处 |
|---|---|---|
| 名称 | opentofu/manifesto（原 opentffoundation/manifesto，gh api 确认旧名 301 重定向） | gh api 元数据 |
| 星数 | 35557，fork 1017 | gh api 元数据 |
| 语言 | HTML（单页静态站，无构建框架） | `src/index.html` |
| 许可证 | Apache-2.0 | `src/LICENSE` |
| 创建时间 | 2023-08-13（BSL 宣布后第 3 天） | gh api 元数据 |
| 最后提交 | 2025-03-18（更新 Gruntwork 承诺信息） | gh api commits |
| 状态 | **已归档（archived=true）**——使命完成后功成身退 | gh api 元数据 |
| 官网 | https://opentf.org（`src/CNAME`） | `src/CNAME` |
| 成果承接 | opentofu/opentofu（MPL-2.0，30942 星，2026-11 仍在活跃推送） | gh api 元数据 |

## 2. 项目是什么，解决什么问题

2023 年 8 月 10 日，HashiCorp 宣布将 Terraform 及其全部核心产品从 Mozilla Public License 2.0 改为 Business Source License（BUSL 1.1）——一种不被 OSI 承认为开源的源码可用许可。宣言正文开篇即陈述这一事实与后果：约 9 年 MPL 历史培育出的数万用户、贡献者、厂商与模块/插件生态，一夜之间面临"是否有资格继续使用"的不确定性（`src/index.html` 52-63 行）。

宣言的核心判断以加粗形式写出："Our concern: the BUSL license is a poison pill for Terraform"（`src/index.html` 65-67 行）。它论证的担忧点包括：BUSL 及附加使用授权中"竞争"一词措辞模糊，任何企业的使用合法性实际取决于 HashiCorp 的单方面解释；FAQ 中专门回答了"普通用户为什么要关心"——你如何确定自己不与 HashiCorp 竞争？答案是要去问 HashiCorp，而这种许可状态"不可能作为任何东西的立足点"（`src/index.html` 298-336 行）。

针对"HashiCorp 是为了反制白嫖厂商"的官方理由，FAQ 用生态贡献数据反驳：Terraform core 有 1700+ 贡献者、AWS provider 2800+、Azure provider 1300+，Registry 中有 14K+ 模块与数百个第三方工具（Terragrunt、tfsec、tflint 等），绝大多数非 HashiCorp 员工所写；并举出经典反例——社区提交的状态加密 issue/PR（hashicorp/terraform #516、#1421、#9556）多年被搁置，因为加密状态是 TFC/TFE 付费产品的卖点（`src/index.html` 380-465 行）。

宣言还引用 HashiCorp 已删除的 CLA 页面的 Internet Archive 快照，证明 HashiCorp 曾书面承诺"保持标准 FOSS 许可、允许完全 fork"，据此指控 BSL 变更打破了"隐式契约"、构成 bait-and switch（`src/index.html` 349-378 行）。这是整份宣言中最具杀伤力的证据链设计。

宣言提出的行动是两步走：第一步请求 HashiCorp 把 Terraform 切回开源许可（MPL 或 Apache 2.0）并永久承诺；第二步若被拒绝，则 fork 旧的 MPL 版 Terraform 交由中立基金会维护，并给出基金会五原则：truly open source、community-driven、impartial、layered and modular、backwards-compatible（`src/index.html` 109-170 行）。

## 3. 技术栈与依赖

这是一个刻意保持极简的内容型静态站，技术选型以"任何人都能用 PR 签名"为准绳：

- HTML/CSS/原生 JS 三件套：无任何前端框架、无构建工具、无包管理器（`src/` 顶层仅有 index.html、fork.html、announcement.html、style.css、count.js）
- 字体：Google Fonts 托管的 Montserrat（`src/index.html` 15 行），仅此外部依赖
- GitHub Pages + Jekyll 部署：`.github/workflows/jekyll-gh-pages.yml`，push main 即自动构建部署（Jekyll 仅作壳，页面全是原生 HTML）
- count.js：24 行原生 JS，页面加载时遍历签名表 DOM，按 Type 列（Company/Project/Foundation/Individual）动态计数并渲染（`src/count.js`）
- GitHub buttons.js：Star 按钮（`src/index.html` 38 行）

统计意义上的"主语言 HTML"即指 index.html 本体——5407 行、251KB，占仓库文本内容约 85%。

## 4. 目录结构与代码组织

| 文件/目录 | 职责 |
|---|---|
| `index.html` | 宣言正文 + FAQ + 支持者签名表（529-5394 行），即 opentf.org 首页 |
| `announcement.html` | 2023-08-25 公告："已创建私有 fork"，附 roadmap 与 FAQ |
| `fork.html` | 2023-09-05 公告："OpenTF 仓库公开"，附 open roadmap 与 FAQ |
| `count.js` | 客户端签名计数脚本 |
| `style.css` | 手写响应式样式（451 行，含 640/768px 断点） |
| `CNAME` | 自定义域名 opentf.org |
| `images/` | logo、og 分享图、star-history 增长对比图（star-history-202395.png） |
| `favicons/` | 站点图标 6 件套 |
| `.github/workflows/jekyll-gh-pages.yml` | GitHub Pages 自动部署 |
| `README.md` / `LICENSE` / `.gitignore` | 简介（13 行）、Apache-2.0、Jekyll 忽略项 |

组织方式评述：单页 + 两个公告子页的扁平结构，没有内容与展示分离——宣言文字、签名数据、页面标记混在同一 HTML 中。这个"缺点"恰是它的设计意图：签名者只需 fork 后在表格末尾加一个 `<tr>`，零门槛，保证 PR 冲突最小化（签名按时间顺序追加，`src/index.html` 503-514 行的签名指引）。全仓库 22 个文件，无 .gitattributes，无符号链接，tarball 完整（已解压核对）。

## 5. 核心架构与关键实现

本仓库无代码架构，其"核心实现"是三段内容：宣言文本、签名机制、公告页时间线。

**宣言正文（index.html 52-170 行）**采用提案文档的标准结构：Concern（担忧）→ Goal（目标）→ Request（请求）→ Fallback plan（备用方案），每段以加粗一句话点题，再展开论证。Fallback 部分的基金会五原则清单后来被原封不动复制进两份 fork 公告（`src/announcement.html` 62-73 行、`src/fork.html`），作为对外承诺的一致性锚点。

**签名机制（index.html 529-5394 行 + count.js）**：签名表三列——Name / Type / How you'd like to help。Type 为 Company/Project/Foundation/Individual 四类；第三列让签名者声明贡献意向，个人多写"Development; open-source community efforts"，企业则写出具体资源承诺，例如首批四家：

```html
<tr><td><a href="https://www.harness.io/">Harness</a></td>
    <td>Company</td>
    <td>Cover the cost of 5 FTEs for at least 5 years</td></tr>
<tr><td><a href="https://gruntwork.io">Gruntwork</a></td>
    <td>Company</td>
    <td>Cover the cost of 1 FTE for at least 5 years</td></tr>
```

（`src/index.html` 537-546 行，Spacelift、env0 同为 5 FTEs/5 年）

签名流程完全走 GitHub 标准协作：fork → 表格末尾加行 → 提 PR（`src/index.html` 507-514 行）。count.js 在浏览器端实时统计四类数量显示在 Co-signed 区域，无需后端。这一机制使签名既是公开承诺、又是可审计的 git 历史，还顺带贡献了约 736 个 contributors（gh api，含匿名）。

**公告页叙事（announcement.html / fork.html）**：两页公告按"承诺兑现"的节奏发布，各自带独立 FAQ。announcement.html 首次公开"私有 fork 已存在"并解释为何提前开工（"若 HashiCorp 回心转意，最坏损失一周工作量"）；fork.html 则在 fork 公开当天给出已完成/进行中/计划中的三级 roadmap，并附 star-history 对比图——OpenTofu 数周达 33k 星 vs Terraform 用近 10 年达 38k 星（`src/fork.html` 106-113 行）。

## 6. 构建与运行

无构建系统，运行方式即静态站托管：

- 本地预览：任意静态服务器指向仓库根目录（如 `python -m http.server`），入口 `index.html`
- 官方部署：push 到 main 自动触发 `.github/workflows/jekyll-gh-pages.yml`，经 actions/jekyll-build-pages 构建后发布到 GitHub Pages，绑定域名 opentf.org（`src/CNAME`）
- 无测试、无 lint、无依赖安装步骤；`.gitignore` 仅忽略 Jekyll 产物（Gemfile、_site 等，`src/.gitignore`）

## 7. 宣言内容分析：核心诉求与签名规模

**核心诉求条目统计**。宣言担忧点可归纳为五类（依据 `src/index.html` 正文与 FAQ 的段落主旨）：① 许可措辞模糊导致全员法律不确定性（"竞争"由 HashiCorp 单方解释，65-79、298-336 行）；② 生态萎缩预期——开发者与公司将转向真开源替代品（83-89 行）；③ 波及效应——动摇行业对所有开源项目可持续性的信心（91-97 行）；④ 隐式契约被打破——CLA 承诺与 9 年 MPL 事实构成 bait-and-switch（349-378 行）；⑤ 反白嫖理由不成立——生态贡献数据 + 状态加密功能被刻意拒绝的案例（380-465 行）。诉求方案两条：请求回退许可（109-127 行）与 fork 进基金会（129-170 行），后者含五原则。

**签名规模与结构（gh api 核实 + 表格逐行统计）**。签名表实际收录 **966 个签名**：企业 163、项目 12、基金会 1、个人 790（`src/index.html` 529-5394 行逐行 grep 统计）。对比公告页快照可还原增长曲线：2023-08-25 为 100+ 公司/10 项目/400+ 个人（`src/announcement.html` 52 行），2023-09-05 为 130+ 公司/680+ 个人（`src/fork.html` 108-110 行）——即约 6 周内签名基本收敛，最终定格于 966。企业签名质量突出：首批 5 家（Harness、Gruntwork、Spacelift、env0、Scalr）互为 Terraform 平台竞品却联合承诺 FTE 资源，书面 FTE 承诺合计 15 个工程师年（5×3 + 3×1 + 1×1 + 1×2，表格逐条统计）；Cloudflare 等大厂亦在名单中（`src/index.html` Companies 区块）。12 个项目签名均为 Terraform 生态工具（terraform-docs、Terrakube、Terraspace、tenv 等，1319-1384 行区域）。个人签名含知名人物如 Jim Jagielski（Apache 软件基金会联合创始人，条目链接至其 Wikipedia 页，`src/index.html` 约 1333 行）。

**签名可撤销与可修订**也是机制的一部分：git 历史显示 2024-11-01 有 PR #1165 "withdraw-the-signature"（eliastor 撤回签名）、2025-03-11 PR #1169 "Update Gruntwork's OpenTofu commitment!"（gh api commits 核实）——签名是活的承诺清单而非一次性签名墙。

## 8. 时间线与归档定性

完整时间线（全部经源文件或 gh api 核实）：

- **2023-08-10**：HashiCorp 宣布 Terraform 转 BSL（`src/index.html` 58-60 行）
- **2023-08-13**：manifesto 仓库创建（gh api created_at），三天内星数破 4k（`src/announcement.html`）
- **2023-08-25**：给 HashiCorp 的公开回复期限届满、无回应（`src/fork.html` 88 行）；同日发布 announcement.html，宣布私有 fork 已建立、已递交加入 Linux Foundation 的全部文件、目标进入 CNCF，4 家公司承诺 14 FTE（`src/announcement.html` 74-100、139 行）
- **2023-09-05**：OpenTF 仓库公开（github.com/opentffoundation/opentf，页面顶部横幅 `src/index.html` 42 行），发布 fork.html；首个版本定为 1.6.0-alpha，从最后一个 MPL 许可的提交 fork（`src/fork.html` 177 行）
- **2023 下半年**：项目经 Linux Foundation 托管并更名 OpenTofu，仓库由 opentffoundation/manifesto 迁移为 opentofu/manifesto（gh api 301 重定向核实）；fork 后续演化为今天的 opentofu/opentofu（MPL-2.0，30942 星，2026-11-21 仍有推送，gh api）
- **2024-11 ～ 2025-03**：仓库进入象征性维护——仅剩签名修订类 PR（撤回、更新承诺）
- **归档**：archived=true（gh api 核实），README 仍指向 opentf.org 与 fork 页（`src/README.md`）

**归档性质：这是"任务完成"式成功归档，不是弃坑。**判据有三：其一，宣言的两步计划至少 fallback 路径 100% 兑现——HashiCorp 未回退许可，但 fork 如期公开、进入基金会（Linux Foundation）、保持 MPL-2.0、承诺 backwards-compatible，五原则逐条落地（对照 `src/index.html` 145-170 行与 opentofu/opentofu 现状）；其二，承接主体 opentofu/opentofu 至今活跃（gh api pushed_at 2026-11-21），宣言没有被废弃而是被"超越"；其三，归档前最后的内容变更仍是签名修订而非烂尾改动。归档是历史档案化的仪式：宣言、966 个签名与两份公告原地保存，供后世查阅这场行业事件。

**开源许可争议史坐标**：这是 2023 年开源行业标志性事件——HashiCorp 同期将旗下全部核心产品 BSL 化，本仓库正是社区反弹的集中爆发点。与本项目此前分析过的仓库对照：hashicorp/vault 是同一批 BSL 变更的直接受害者（其分析文档记录了许可争议背景），hashicorp/terraform 系则是争议主角本体；而 opentofu/manifesto 是争议的另一面——社区端的应对范本。三方档案合起来构成 2023 年"BSL 之冬"的完整证据链。

## 9. 亮点与局限

**亮点**

1. 历史上罕见的抗议成功案例：从发布到 fork 公开仅 23 天（2023-08-13 → 2023-09-05），每一步承诺都有公告页与 git 记录背书，执行纪律极强（`src/announcement.html`、`src/fork.html`）
2. 证据链设计专业：用 Internet Archive 快照固定 HashiCorp 已删除的 CLA 承诺，用具体 issue 编号（#516/#1421/#9556）支撑"压制社区贡献"的指控，使宣言在争议中站得住脚（`src/index.html` 349-465 行）
3. 零门槛签名机制：fork + 加一行表格 + PR，配合 count.js 实时计数，把 966 个签名变成可审计的 git 历史（`src/index.html` 503-5394 行、`src/count.js`）
4. 企业资源承诺落到实处：书面 FTE 数字（首批即 15 个工程师年）而非口头声援，为 fork 的可持续性提供了直接人力证明（`src/index.html` Companies 区块）
5. 极简工程选型：无框架无构建，任何人可审、可改、可部署，与"社区驱动"的宣言气质完全一致（全仓库 22 文件）

**局限**

1. 签名总量 966 相对 35557 星是极小比例（约 2.7%）——绝大多数支持者以 Star 代替签名，签名表在 2023-10 后增长趋于停滞（表格逐行统计 + 公告页快照对比）
2. 页面内容与展示耦合在单个 251KB HTML 中，签名追加剧增时合并冲突频发（这从 2116 个提交中大量签名 PR 可推断，gh api commits 核实）
3. 信息更新停滞：README 与页面仍称 fork 位于 opentffoundation/opentf，未回填 opentofu 组织更名后的最终地址（`src/README.md`、`src/index.html` 188 行）
4. 单点叙事风险：宣言为单方面立场文件，HashiCorp 观点仅在转述中出场；不过其对 CLA 快照与 issue 编号的引用为读者提供了自行核查的入口

## 10. 总结评价

opentofu/manifesto 是开源史上少有的"抗议即产品"案例：一份写得极其克制的 HTML 宣言、一张 966 行的签名表、两页踩着承诺日期发布的公告，在 23 天内把 HashiCorp BSL 变更引发的社区愤怒转化为一个进入 Linux Foundation、至今活跃的 MPL-2.0 替代品 OpenTofu。它的星数（35.5k）衡量的是行业情绪的烈度，它的归档则是使命达成的句号——今日 Terraform 生态的 split（Terraform/BSL vs OpenTofu/MPL）正是这份宣言的直接产物。适合三类读者：研究开源治理与许可政治的从业者（全文 + 签名表是第一手史料）、正在起草社区联合声明的组织者（两步计划 + FTE 承诺 + 公告节奏是可直接套用的操作手册）、以及想理解 2023 年"BSL 之冬"全貌的工程师（建议与 hashicorp/vault、hashicorp/terraform 的既有分析对照阅读）。作为代码仓库它没有阅读价值，但作为社区行动的完整档案，GitHub 上几乎没有第二个。
