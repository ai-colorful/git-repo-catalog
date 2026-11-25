# geekan/HowToLiveLonger 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 —（内容型仓库；工程部分为 Python + JavaScript）｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：《程序员延寿指南》——一份以全因死亡率（ACM）为主线、汇总饮食/运动/睡眠/药物等干预研究的文献综述型清单，2026-10 已从单一 README 重构为八语言"证据图谱"静态站点。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | geekan/HowToLiveLonger | gh api repos/geekan/HowToLiveLonger |
| 星数 | 快照 39,500（2026-10-24）；gh api 实测 36,170（2026-11-21） | 任务参数；gh api |
| 主语言 | —（GitHub 未标注；工具链为 Python 3.10+ 标准库 + 原生 JS） | 任务参数；scripts/build.py |
| 许可证 | Unlicense（公有领域） | LICENSE |
| 创建时间 | 2022-04-16 | gh api |
| 最近 push | 2026-10-22（与 docs/review-2026-10-22.json 吻合） | gh api；docs/review-2026-10-22.json |
| 在线站点 | https://geekan.github.io/HowToLiveLonger/（GitHub Pages，`.nojekyll`） | README.md；CONTRIBUTING.md |
| 作者 | geekan（Alexander Wu，MetaGPT 作者，2012 年注册账号） | gh api users/geekan；docs/original-readme.zh.md 徽章 |

> 声明：本仓库内容为健康/医学文献汇总与核验台账，信息仅供研究参考，不构成医疗建议；本文档仅做仓库结构与内容组织的客观记录，不呈现、不复述任何个体健康建议。

## 2. 项目是什么，解决什么问题

仓库起源于一份纯 README 内容仓：作者以 OKR（目标/关键结果/行动/证据）的工程师视角，把饮食、运动、睡眠、药物等日常因素与"全因死亡率 ACM"的研究结果整理成带数字的清单，原始版本冻结于 `docs/original-readme.zh.md`（306 行，字节级保留，其 SHA-256 由 `scripts/validate.py` 与 `data/evidence.json` 的 `metadata.sourceReadmeSHA256` 字段比对校验）。原版以"挥拍运动 -47% ACM、晒太阳 -40% ACM"等量化条目在 2023 年 7 月前后（提交历史显示该时段密集更新 README）被广泛传播，是仓库星数的主要来源。

2026-10-22 的一批提交（"Make the eight-language longevity website the repository homepage"等，见 gh api commits）将仓库整体重构：原 README 的每条说法被拆解为结构化研究记录（`data/evidence.json`），逐条标注结论类型、效应量、置信区间、研究人群与出处，并生成一个自包含的八语言静态站点 `index.html`（2.4MB，数据内嵌、可离线打开）。重构同时保留了原始中文/英文 README 作为不可改写的基线档案（`docs/original-readme.*.md`），原话（originalClaim）与核验后的当前结论（finding）分离存档。

它解决的问题可以概括为：网络流行的健康说法往往只有断言没有出处，本仓库把"说法—研究—证据强度—适用人群"四层信息绑定在一起，并以机器可读的 JSON 发布；同时通过 `skills/howtolivelonger/SKILL.md` 提供可安装的 AI 智能体技能（`npx skills add geekan/HowToLiveLonger --skill howtolivelonger`），让智能体按同一数据源回答并附论文链接（README.md、SKILL.md）。

需要注意仓库的元数据自我定位（`data/evidence.json` metadata.batchNotes）：项目的 A/B/C/D/U 分级"不是正式 GRADE"；观察性研究的关联不能当作个体因果效应；各因素 HR 不能相乘或相加。README 与站点 UI 中的措辞是"日常参考"（`locales/interface.json` zh.ui.action）而非医疗指导。

## 3. 技术栈与依赖

| 层 | 技术 | 出处 |
| --- | --- | --- |
| 构建工具 | Python 3.10+，仅标准库（json/re/hashlib/base64/html.parser/subprocess 等），零第三方依赖 | scripts/build.py、scripts/validate.py |
| 前端 | 单文件原生 JavaScript（无框架、无构建链），数据以 `<script type="application/json">` 内嵌 | src/index.template.html |
| 数据 | JSON：evidence.json（221KB）、glossary.json（50KB）、8 语言 locale 包（各 117-158KB） | data/、locales/ |
| 发布 | GitHub Pages 服务 main 分支根目录，`.nojekyll` 跳过 Jekyll；index.html 标记 `linguist-generated` | CONTRIBUTING.md；.gitattributes |
| 许可 | Unlicense（公有领域） | LICENSE |

仓库刻意保持"无 CI、无 workflow 文件"（CONTRIBUTING.md 明文要求 "Keep workflow files out of this repository"），构建与校验全部在本地完成，产物 index.html 与源码一起提交。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `data/evidence.json` | 核心数据：metadata + 64 条研究记录 + 74 条引用台账 |
| `data/glossary.json` | HR/RR/OR/CI/ACM 等 19 个术语 × 8 语言释义 |
| `src/index.template.html` | 75KB 页面模板：表格/树/台账三视图、搜索筛选、CSV/JSON 导出、暗色主题、RTL 阿语适配 |
| `locales/` | interface.json（131 个 UI 键 × 8 语言）+ 7 个翻译文件（每文件 64 条记录 × 11 字段）+ readers/（8 语言 × 64 条读者版摘要/行动建议文案） |
| `docs/original-readme.*.md` | 冻结的原始中/英 README 基线（306/312 行，字节不变） |
| `docs/review-*.json` | 两批文献复核台账（2026-10-21/22），含 HTTP 状态与实际阅读范围 |
| `scripts/build.py`、`scripts/validate.py` | 构建与契约校验（可复现构建的字节级比对） |
| `skills/howtolivelonger/SKILL.md` | 可安装智能体技能，指向 raw evidence.json |
| `assets/` | 封面与 logo（5.5MB，封面以 base64 内嵌进 index.html） |
| `index.html` | 生成产物（2.4MB），在线阅读与离线下载的唯一入口 |

组织方式是"数据 + 生成产物"的单仓模式：JSON 是唯一事实源，index.html 是可复现的派生物，翻译文件与主数据按记录 ID 严格对齐（build.py 中 `set(translated) != {r['id'] for r in records}` 即报错）。

## 5. 核心架构与关键实现

**（1）数据模型（`data/evidence.json`）**。64 条记录共享 19 个字段：id、category（输入/输出/上下文）、subcategory、factor、origin（原库 47 条/新增 17 条）、originalClaim（原 README 原话存档）、verdict（支持但需限定 34/需纠正 15/部分支持 10/无直接 ACM 证据 5）、finding（当前核验结论）、effect/ci/comparison/population/design、grade（A 6/B 32/C 17/D 9）、gradeReason、limitations、action、sources（85 条文献引用，81 条带 DOI）、originalClaimRaw（含原 Markdown 上下文）。sources 条目还带 readLevel（实际读到什么程度）与 status：

```json
{
  "id": "poultry-fish", "category": "输入", "grade": "B",
  "verdict": "部分支持",
  "finding": "原段混合两篇论文：…白肉ACM无显著关联不等于证明无风险…",
  "effect": "加工肉HR 1.03；红肉HR 1.03；家禽HR 0.99…",
  "sources": [{ "doi": "10.1001/jamainternmed.2019.6969",
                "readLevel": "摘要", "status": "已核验" }]
}
```

**（2）构建器（`scripts/build.py`）**。把 evidence.json、8 语言 locale、readers、glossary、冻结 README、base64 封面全部注入模板占位符，输出自包含 index.html；任何语言缺记录/缺字段/源引用数不齐都会直接抛错（`load_locales` 的严格契约检查），并打印产物 SHA-256。

**（3）校验器（`scripts/validate.py`）**。最有特色的部分：跨语言数值一致性校验——用 `numeric_tokens` 把中/英/日/阿等 8 语的 effect/ci 字段归一化（"1万"→10000、月份名→数字等）后比对 Counter，防止翻译时丢改数字；`HR|RR|OR|STR` 效应量类型也必须逐语言一致；读者版文案不得引入数据集中不存在的数字（`mentioned <= available`）；最后 `--check-build` 在临时目录重建并与仓库内 index.html 做字节比对。

**（4）发布链**。无 CI；CONTRIBUTING.md 规定"构建+校验→同时提交源与生成物→Pages 直接服务 main"。两份 review JSON（docs/review-2026-10-21.json 25KB、review-2026-10-22.json 2.5KB）记录每条文献复核的请求 URL、HTTP 状态、阅读定位与变更，是人工/AI 核验流程的审计痕迹。

## 6. 构建与运行

来自 CONTRIBUTING.md 的实际命令（Python 3.10+，无需安装任何依赖）：

```sh
git clone https://github.com/geekan/HowToLiveLonger.git
cd HowToLiveLonger
python3 scripts/build.py                      # 生成 index.html
python3 scripts/validate.py --check-build     # 契约校验 + 可复现构建比对
python3 -m http.server 8000                   # 本地预览 http://localhost:8000
```

离线使用：直接下载仓库并双击 `index.html`（数据已内嵌，无需服务器）。

## 7. 内容组织与规模统计（内容型特例）

**收录范围与分类体系**：延续原 README 的"输入/输出/上下文"三分法，下分 13 个子类，共 64 条记录（统计自 data/evidence.json）：

| 类别 | 子类（条数） |
| --- | --- |
| 输入 45 | 固体 19（白肉/辣椒/坚果/钠/碳水/槟榔/热量限制/超加工食品/全谷物/膳食纤维/植物蛋白等）、药物 15（二甲双胍/NMN/复合维生素/葡萄糖胺/他汀/维生素D/司美格鲁肽/达格列净/阿司匹林等）、液体 8（咖啡/茶/牛奶/含糖饮料/果汁/酒精）、气体 2（吸烟、PM2.5）、光照 1 |
| 输出 12 | 运动 6（挥拍/剧烈/步行×2/抗阻/心肺适能）、日常 3（刷牙/泡澡/家务）、睡眠 2（时长/就寝时间）、久坐 1 |
| 上下文 7 | 情绪 2（悲观乐观、社会孤独）、体重 2（体重变化、腰围）、贫富 1、新冠 1、睡眠呼吸暂停 1 |

**全部 64 条记录明细**（factor 字段，按 category/subcategory 归组，出处 data/evidence.json）：
- 输入·固体（19）：白肉红肉与加工肉、新鲜水果（糖尿病人群）、含黄酮食物、红辣椒（美国）、辣椒（意大利）、鸡蛋、核桃、坚果频率、钠摄入跨国生态关联、低钠富钾盐替代、碳水能量占比、槟榔、热量限制（非人灵长类）、综合营养与宏量营养配比、超加工食品与 ACM、超加工食品（CVD 结局边界）、全谷物、膳食纤维、植物蛋白来源
- 输入·药物（15）：膳食亚精胺、NMN、二甲双胍（超重新诊断 2 型糖尿病）、二甲双胍（健康人及安全性断言）、复合维生素、葡萄糖胺/软骨素、综合抗衰药候选、酒精/饮酒、他汀（按适应证）、维生素 D（一般老年成人）、符合适应证的强化降压、达格列净（特定慢性肾病）、司美格鲁肽（SELECT 适应人群）、阿司匹林（老年人初级预防）、二甲双胍不良反应口径核对
- 输入·液体（8）：乳制品/牛奶、饮茶（China-PAR）、饮茶（CKB）、人工甜味软饮料、含糖饮料、100% 果汁、咖啡（其余 1 条归入药物类酒精）；输入·气体（2）：吸烟与戒烟、长期环境 PM2.5；输入·光照（1）：晒太阳/紫外线
- 输出·运动（6）：挥拍运动、剧烈运动、走路/每日步数、抗阻/肌肉力量训练、心肺适能（CRF）、每日步数（2025 综述更新）；输出·日常（3）：刷牙、泡澡、重型家务（老年男性）；输出·睡眠（2）：睡眠时长、就寝时间；输出·久坐（1）：久坐及看电视
- 上下文（7）：悲观/乐观情绪、社会隔离与孤独、净资产/经济资源、体重变化、腰围与腹型肥胖、新冠疫情与超额死亡、阻塞性睡眠呼吸暂停

**规模统计**：
- 原始基线 README：中文 306 行、英文 312 行（docs/original-readme.*.md），原文含 77 处文本 HTTP 引用，合并为 74 条独立引用台账（metadata.coverageNote）。
- 结构化记录 64 条（原库拆分 47 + 新增 17），记录内文献引用 85 条；术语表 19 词 × 8 语言。
- 翻译规模：7 个 locale 文件各含 64 条记录 × 11 个字段 + 逐条 source evidence；readers/ 8 文件 × 64 条 summary+takeaway（实测各语言记录 ID 与主数据完全一致）。
- 生成站点 index.html 2.4MB，单一文件内嵌全部数据/译文/封面。

**为什么能涨到这么多星**：2023 年 7 月 README 密集更新期恰逢其传播高峰（提交历史佐证），"用工程 OKR 框架给延寿课题做文献综述"的形式在中文社区稀缺且易转播；2026 年的重构进一步把网红清单升级为可审计的证据库，并借 AI 智能体技能（SKILL.md）获得新的分发入口。

**姊妹项目与生态位**：同一作者的 MetaGPT（geekan/MetaGPT，多智能体框架）是本仓库的"孪生叙事"——原 README 开宗明义"目标：稳健地活得更久；花更少时间工作：见 MetaGPT"（docs/original-readme.zh.md 第 44 行），"延寿指南（省下的时间）+ 效率工具（省时间）"构成作者的个人品牌组合。在本仓库所属的"健康/循证清单"内容仓生态中，其特色是量化 ACM 指标 + 工程化核验流程，区别于常见的故事型科普或纯链接聚合类 awesome 清单。

## 8. 文献引用质量与核验机制（内容型特例）

**引用标注水平**：85 条记录级文献引用中 81 条带 DOI，全部带 URL、年份、标题、readLevel、status 与摘录证据（evidence 字段抄录摘要 Results 关键数字）。链接多指向 PMC/PubMed/Europe PMC 的论文入口（如 PMC7042891 对应 JAMA Internal Medicine 2020，doi:10.1001/jamainternmed.2019.6969），原出版商 403 时改用同篇 PubMed/PMC 镜像并保留 originalUrl（metadata.batchNotes）。

**证据分级的自我定义**（metadata.batchNotes 原文口径）：A/B/C/D/U 是"本项目按 ACM 适用性的实用分级，非正式 GRADE"；A 可表示高质量随机阴性证据，不表示可推荐延寿；D 可表示随机替代终点或间接暴露证据；"无直接 ACM 证据不同于已证明无效"。每条记录还带 gradeReason 与 limitations 字段逐条说明降级理由，verdict 与 grade 相互独立（实测 grade A 的 6 条记录 verdict 均含限定语，无一为无条件支持）。

**来源层级客观评价**：
- 强项：几乎全部引用为同行期刊论文（JAMA 子刊、BMJ、Science 等）或权威机构页面（中国 CDC、NHLBI、CDC），且诚实区分"实际读到的材料"——readLevel 分布为摘要 53、全文（关键结果段）25、全文 2、元数据 2、其余 4 条（实测计数）；"全文"定义被明确限定为"读取与结论相关的原文内容，非逐字审查全部补充材料"。
- 弱项：引用台账（citations）74 条中"二手报道"占 46 条（多为知乎等原 README 时代的传播链接，已标记"不用于医学定量结论"）、"已核验"仅 20 条、"无法访问"6 条、"错链"1 条——即历史引用有相当比例未升级为一次文献直链；两份 review JSON 也自述"并非穷尽式系统综述"（review-2026-10-21.json method 字段），复核批次仅实际审读 14 条记录、24 篇摘要、4 篇全文关键段。
- 核验机制本身（ID 稳定、originalClaim 不可改写、finding 才能修正、阅读范围留痕）在同类内容仓中罕见地严格，CONTRIBUTING.md 甚至要求"不能通过改写原话消除错误"。

## 9. 医学信息风险与免责性质（内容型特例）

基于真实文件的客观记录（不做任何健康建议表述）：
- 仓库自我警示集中在 `data/evidence.json` metadata.batchNotes 与 CONTRIBUTING.md：A/B/C/D/U 是"本项目实用分级，非正式 GRADE"；"观察研究关联不能当作个人因果效应；各因素 HR 不能相乘"；"不要把疾病治疗证据变成健康人建议"；"HR/RR/STR 相对指标不等于绝对风险差或个体寿命增益，也不能直接相加换算活多久"。
- 站点 UI 的 caution 文案（locales/interface.json）是阅读方法指引（"先看研究比较了哪些人，再看效应量和区间"），并未出现 blanket 式"不构成医疗建议"声明；强约束更多写在贡献规范与数据元数据里，普通访客在页面上未必能看到等强度的风险提示——这是值得记录的局限。
- 数据自身的降级记录印证了原清单的不可靠部分：64 条中 verdict 为"需纠正"15 条、"无直接 ACM 证据"5 条，仅 10 条"部分支持"、34 条"支持但需限定"，没有一条被标为无条件"支持"。
- **本文档郑重记录：该仓库全部内容为文献汇总与核验台账，信息仅供研究参考，不构成医疗建议；涉及健康决策应咨询专业医疗机构。**

## 10. 社区与维护状态、总结评价

**维护状态**（gh api 实测 2026-11-21）：创建于 2022-04-16；2023-07 为 README 高产期，2024-01-30 后沉寂，2025-05 出现自动翻译工作流修复提交（#167），2026-10-22 由作者 Alexander Wu 单日完成八语言站点重构并追加 3 条新记录（腰围/睡眠呼吸暂停/丙肝治疗，见 docs/review-2026-10-22.json）。贡献者共 10 人，当前 45 个开放 issue，无 release 与 CI（CONTRIBUTING 明确排除 workflow 文件）。星数快照 39,500（2026-10-24），gh api 实测 36,170（2026-11-21），差值约 3,300，原因未确认（可能为 GitHub 清理非活跃星标）。作者 geekan（Alexander Wu）为 MetaGPT 作者，原 README 徽章即以"减少工作时间：MetaGPT"互链（docs/original-readme.zh.md），引用台账中也把 MetaGPT 仓库列为"非医学来源"（evidence.json citations[0]）——同一作者身份确认。

**亮点**：① "原话存档 + 当前结论"双轨数据模型，修正不抹历史，审计友好；② validate.py 的跨语言数值/效应量一致性校验在内容仓中罕见；③ 零依赖构建 + 单文件离线站点，分发成本极低；④ 八语言完整覆盖（含 RTL 阿语与术语悬浮释义）；⑤ AI 智能体技能与 JSON 数据集契合 2026 年的分发生态。

**风险/局限**：① 健康话题天然高风险，页面级免责声明偏弱，医学结论依赖读者自行阅读 limitations；② 大量历史引用仍是二手链接（台账 46/74），"无法访问"6 条未清理；③ 核验为选择性而非系统性综述，覆盖度有明确边界；④ 单作者驱动，2024-2026 之间有约两年半的低活跃期，持续性依赖个人投入；⑤ 生成式 index.html 使仓库 diff 噪音大（2.4MB 单文件随每次数据变更重写）。

**总结评价**：这是"网红清单仓库自我进化"的一个样板——从 2023 年靠量化断言传播的 README，演进为带 DOI 溯源、证据分级、八语言和机器可读接口的文献核验台账，工程纪律（契约校验、可复现构建、审计留痕）远超同类内容仓；但它的医学内容仍然只能当作"文献检索入口"使用。适合对健康循证方法学、多语言内容工程、AI 技能分发感兴趣的读者研究其数据与流程设计；内容本身仅供研究参考，不构成医疗建议。
