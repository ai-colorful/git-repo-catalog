# freestylefly/awesome-gpt-image-2 项目分析

> 星数 36017（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：GPT Image 2/2.5 的 "Prompt as Code" 工业级提示词案例库——把散落的社区爆款出图案例反向拆解为结构化、可填空、可被 Agent 复用的提示词资产，并配套一个可直接在线试生成的可视化网站与 Agent Skill。

| 项 | 值 |
| --- | --- |
| 名称 | freestylefly/awesome-gpt-image-2（`package.json` 内站点名 `awesome-gpt-image-2-site`） |
| 星数 / Forks | 36017 / 3411（gh api，2026-11-21 核实） |
| 创建时间 | 2026-04-25（gh api `created_at`） |
| 最近推送 | 2026-11-18（gh api `pushed_at`，分析时 3 天前） |
| 主语言 | JavaScript（TypeScript 仅 `middleware.ts` 一个文件） |
| 许可证 | MIT（`LICENSE`） |
| 实际形态 | 内容库（541 个案例 + 22 套模板）+ React/Vite 可视化网站 + Vercel Serverless API + Claude/Codex Agent Skill 的混合体 |
| 在线站点 | gpt-image2.canghe.ai（README.md 第 39 行、`vercel.json`） |

## 2. 项目是什么，解决什么问题

2026 年 GPT-Image2 发布后，AI 图像生成的问题从"能不能画出来"变成"能不能稳定、可控、可复用地画出来"（README.md §Project Vision）。社区在 X/Twitter 上涌现大量惊艳案例，但原始提示词散落在一条条推文里：无法检索、无法批量复用、无法给自动化流程调用。本项目把这些案例"扒了个底朝天"（`docs/templates.md` 开头自述），逐条收录成"效果图 + 完整提示词 + 来源链接"的三件套，再做二次加工——从案例中提炼出带防坑指南的工业级模板，并进一步把散文式提示词压缩为结构化协议。

项目的核心主张是 Prompt as Code：提示词应当像代码一样有 schema、有变量占位符、有约束条件、可以版本化与程序化组装。体现在三层（`data/style-library.json`、`data/cases.json`）：案例提示词使用 `[产品类型]`、`[FRUIT]` 等可替换占位符，很多采用 `[中文]/[English]` 双语块；模板层每套带 `useWhen/guidance/pitfalls/exampleCases` 字段；部分模板直接给出 JSON 形态的提示词（`docs/templates.md`），"推荐给 Agent 调用"。

三个使用场景在 README §How To Use 中写得很清楚：先看精选案例确定视觉方向 → 在完整画廊里找邻近案例"抄结构再抄风格词" → 回到模板页把业务变量填进通用或 JSON 模板。除了内容本身，仓库还运营着一个落地网站（登录后可用平台积分或自有 API Key 实测生成，README §Website Auth & Generation），以及一个发布到 npm 和 GitHub Packages 的 Agent Skill（`agents/skills/gpt-image-2-style-library/`），让 Claude Code/Codex/Cursor 等 Agent 能直接引用同一份风格库写提示词。

## 3. 技术栈与依赖

基于 `package.json` 与 `api/`、`supabase/` 目录：

- 前端：React 19 + Vite 7（`vite.config.js`、`src/main.jsx`），lucide-react 图标库；单页应用，画廊主逻辑集中在 `src/main.jsx`（4315 行）
- 后端：Vercel Serverless Functions（`api/` 下约 57 个 JS 文件），Node 原生 `node --test` 跑单元测试
- 数据库/认证：Supabase（`@supabase/supabase-js`），`supabase/migrations/` 13 个 SQL 迁移（用户积分、会员计费、收藏、OAuth 账号、APIMart 生成任务等，2026-05-09 至 2026-08-28）
- 支付：Stripe + 支付宝（`alipay-sdk`，`api/billing/`、`api/community/alipay/`）
- 分析：Google Analytics Data API（`@google-analytics/data`，`api/_lib/ga4.js`）
- 生成上游：APIMart API 中转（`shared/apimart.js`、`api/_lib/apimart.js`）
- 内容管线：纯 Node 脚本 `scripts/generate-site-data.mjs` 把 markdown 画廊解析成 `data/cases.json`；`scripts/generate-style-skill.mjs` 把 `data/style-library.json` 渲染成 Skill 的 reference 文档

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `docs/` | 内容主战场：`gallery-part-1.md`（例 1-165，6776 行）、`gallery-part-2.md`（例 166-544，12527 行）、`gallery.md`（838 行总览）、`templates.md`（1080 行模板与防坑指南）、免责声明与付费社区文档 |
| `data/` | `cases.json`（541 案例的结构化数据）、`style-library.json`（22 套模板 + 分类/风格/场景标签体系）、`images/`（550 个文件、约 160MB 案例效果图） |
| `src/` | React 前端：`main.jsx` 画廊主应用、`community.jsx` 付费社区页、`image25/` GPT-Image-2.5 专题页（对比滑块演示）、`apimartClient.js` 浏览器端直连生成 |
| `api/` | Serverless API：认证、积分、Stripe/支付宝账单、生成任务、收藏、管理后台、付费社区（`api/_lib/` 公共库带单元测试） |
| `agents/skills/gpt-image-2-style-library/` | Agent Skill：`SKILL.md` + `references/style-library.md`（659 行，由 JSON 自动生成）+ npm 安装器 `bin/install.mjs` |
| `scripts/` | 6 个 .mjs 脚本（836 行）：站点数据生成、Skill 生成/安装、支付宝沙箱、GA4 OAuth、本地 API 代理 |
| `supabase/` | 13 个 SQL 迁移（积分扣减 RPC `reserve_generation_usage`、会员计划、支付宝、付费社区等） |
| `.claude-plugin/` + `.github/workflows/` | Claude Code 插件市场配置；`publish-style-skill.yml` 按 tag 发布 npm 包 |
| `middleware.ts` + `vercel.json` | Vercel Edge 中间件做图片/JSON 防盗链；缓存策略与 SPA rewrite |

组织评述：这是"内容仓库长出了产品"的典型形态——内容（docs + data）与站点（src + api + supabase）共享同一份数据源：markdown 画廊是人工维护的源头，脚本生成 `cases.json` 供网站和 Skill 消费（`scripts/generate-site-data.mjs` 的解析逻辑反向依赖 markdown 格式）。单向流水线设计规避了双份数据漂移。

## 5. 核心架构与关键实现

**（1）Prompt as Code 数据模型**（`data/style-library.json`，22 套模板）。每套模板不是一段文字而是一个双语 schema，例如 `ui-screenshot-system`：

```json
{
  "id": "ui-screenshot-system", "anchor": "tpl-ui",
  "title": {"en": "UI Screenshot System", "zh": "UI 截图系统"},
  "category": "UI & Interfaces", "tags": ["UI", "Dashboard", "Screenshot"],
  "useWhen": {"zh": "用于 App 截图、仪表盘、社媒截图和直播界面。"},
  "guidance": {"zh": ["锁定平台、比例、层级和画面文字。",
    "明确状态栏、Tab、操作区、评论层等 UI 元素。"]},
  "pitfalls": {"zh": ["避免平台描述过泛。", "约束文字可读性和平台特征。"]},
  "exampleCases": [17, 2, 4]
}
```

`guidance/pitfalls` 把逆向工程的经验固化为"防坑指南"，`exampleCases` 反向链接到具体案例形成"模板↔案例"双向索引。标签体系有 13 个分类、19 种风格、10 种场景（`data/cases.json` 顶部字段）。

**（2）markdown→结构化数据的解析管线**（`scripts/generate-site-data.mjs`）。从 `docs/gallery.md` 的 `<a name="cat-...">` 锚点切分出每个分类的案例号映射，再用正则从 `**提示词：**\`\`\`text ... \`\`\`` 块提取完整提示词、从 `**来源：**` 行提取作者与推文链接，产出 `data/cases.json`。网站构建（`prebuild`）和 Skill 生成都在这条管线上。

**（3）积分制生成链路**（`api/generate-image.js`）。登录用户发起生成前先通过 Supabase RPC `reserve_generation_usage` 预扣积分，失败（`CREDITS_REQUIRED`）直接拒绝；调用 APIMart 生成后按结果释放或回滚预留（`releaseReservation`），错误码统一收敛为面向公众的 `UPSTREAM_BUSY/SERVER_NOT_CONFIGURED` 等，不泄漏上游细节。同一套模式延伸到 Stripe webhook（`api/billing/webhook.js`）与支付宝回调（`api/billing/alipay/notify.js`）。

**（4）Agent Skill 分发**（`agents/skills/gpt-image-2-style-library/SKILL.md`）。SKILL.md 定义了工作流：识别语言 → 匹配"模板分类→风格标签→场景标签→示例案例" → 按六个块组装提示词（主体/构图布局/风格材质/文字标签/比例输出/约束负面词）。reference 由 `npm run generate:style-skill` 从 JSON 再生成，保证"网站、Skill、仓库三者同一份风格库"。分发渠道齐全：`npx skills add`、Claude Code 插件市场（`.claude-plugin/marketplace.json`）、npm CLI（已发布 `gpt-image-2-style-library` v1.0.4）、GitHub Packages（CI 按 tag 发布）。

## 6. 构建与运行

来自 `package.json` scripts 与 README（未实际运行，静态核对）：

```bash
npm install
npm run dev            # predev 先生成 cases.json 与 style-skill，再起 Vite
npm run build          # prebuild 同上 + vite build（Vercel 同款）
npm run test           # node --test api/_lib/*.test.js
npm run test:apimart   # APIMart 相关 4 个测试文件
npm run generate:site-data    # docs/*.md → data/cases.json
npm run generate:style-skill  # data/style-library.json → Skill reference
npm run install:skill         # 装入 ~/.codex/skills、~/.claude/skills 等
# Skill 一键安装（面向终端用户，README §Agent Skill）
npx skills add freestylefly/awesome-gpt-image-2 --global --all --copy
```

网站自部署需要配置 Supabase/Stripe/支付宝/GA4/APIMart 等约 12 个环境变量并按顺序执行 13 个 SQL 迁移（README §Website Auth & Generation 有完整 checklist）。

## 7. 内容组织与规模统计（内容核验）

**案例规模实测：真有 530+。** 逐条统计 `data/cases.json`：实际 541 条，ID 覆盖 1-544（缺 12、169、170 三个编号，系删除后未重排）；`docs/gallery-part-1.md` 164 个 `### 例` 标题 + `docs/gallery-part-2.md` 377 个 = 541，两侧完全一致。README 徽章与 `docs/gallery.md` 声称"544"略有夸大（按编号上限计），实际 541。541 条全部有对应图片文件（`data/images/` 550 个文件零缺失）与完整提示词，平均提示词长度约 1256 字符；516 条（95%）带来源链接，161 条为 `[中文]/[English]` 双语提示词。

**模板数量：20+ 属实。** `data/style-library.json` 实测 22 套模板（`docs/templates.md` 文本自称 21 套，写作时为 393 案例版本，后续补录未同步文案），覆盖：UI 界面、信息图/图表、海报排版、营销海报、字体海报、产品电商、品牌识别包、建筑空间、写实摄影、插画艺术、角色设计表、3D 玩具、场景叙事、历史古典长卷、文档出版、概念产品拆解等 16 个领域。

**案例分类覆盖面**（`data/cases.json` 实测分布）：海报与排版 90、摄影写实 78、UI 界面 73、插画艺术 59、图表信息图 53、产品电商 42、人物 31、品牌 Logo 27、场景叙事 21、历史古典 16、建筑空间 12、文档出版 11、其他 28——海报/图标/信息图/漫画叙事等热门方向均有专门分类，选题明显追随社区热点（十二黄金圣斗士卡牌、赤壁怀古长卷、水果拆解学习卡等，见 `docs/gallery.md` 推荐入口）。

**来源与更新机制**：内容主要是从 X/Twitter 公开案例逆向整理（README §Disclaimer），声明参考了 YouMind 与 OpenNana 的公开提示词库并逐条保留原作者与原帖链接，提供侵权删除通道；贡献者仅 4 人（见第 8 章），更新以维护者单人持续精选补录为主，无 PR 自动化流水线。

## 8. 社区与维护状态、商业化运营

- 维护强度：gh api `commits` 显示最近推送 2026-11-18（refresh APIXO sponsor 资产）、11-10、11-05 均为赞助商相关 docs 提交；创建 6.5 个月 36k 星，属超高速增长
- 贡献者结构：gh api `contributors` 仅 4 人——freestylefly 168 commits（约 98%）、ImgBotApp/sscodeai/LengShangbai 各 1，实质单人项目；open issues 33，无 release（Skill 用 tag 触发 npm 发布代替 release）
- 商业化已相当成熟（`README.md` §Sponsors + `api/` + `docs/paid-community.md`）：8 个 API 中转商赞助位、积分商城（$5/300 credits）、Stripe + 支付宝双通道、付费交流社区（二维码/退款/对账一整套管理后台 `api/admin/`）、GA4 运营分析
- 生态动态：README §GPT Image 2.5 Spotlight 追踪了 OpenAI 2026-09 推出的 GPT-Image-2.5 Sunburst/Flare 两个模型（developers.openai.com 官方文档已核实存在），用 4 个真实复刻案例做对比演示（`src/image25/` 专题应用）

## 9. 亮点与风险

**亮点**

1. 内容真实且质量扎实：541 案例、零缺图、95% 带可溯源出处链接，模板带 useWhen/pitfalls 的工程化封装，远超一般 awesome 清单"只堆链接"的密度
2. Prompt as Code 设计成体系：markdown（人读）→ JSON（程序读）→ Skill reference（Agent 读）一份三吃，`exampleCases` 双向索引，`docs/templates.md` 提供 JSON 提示词形态
3. 内容即产品：可视化网站 + Agent Skill + npm 包 + Claude 插件市场四条分发渠道，且全部由同一数据源自动生成（`scripts/generate-style-skill.mjs`）
4. 工程不敷衍：Serverless API 带单元测试（7 个文件 920 行，`node --test`）、13 个 SQL 迁移、防盗链 Edge 中间件、支付回调完整链路
5. 精准踩中浪潮：GPT-Image2 → 2.5 的时效跟进 + 中文社区传播（三语 README），是 6.5 个月 36k 星的直接解释

**风险/局限**

1. 版权灰区是最大隐患：案例图与提示词源自他人推文，靠"来源署名 + CC BY 4.0 引用 + 删除通道"化解（`README.md` §Disclaimer、`docs/disclaimer.md`），商业可用性明确不背书
2. 单点维护：98% 提交来自一人，内容精选依赖作者个人品味与精力；文档数字有漂移（徽章 544 vs 实际 541、模板 21 vs 22）
3. 站点代码与内容耦合在单仓库：`src/main.jsx` 单文件 4315 行，无组件拆分；`generate-site-data.mjs` 用正则解析 markdown，格式变动即断链
4. 链接可验证性受限：516 条来源链接绝大多数指向 x.com，第三方工具无法免登录批量核验推文存活性（本次抽检受环境网络限制，见第 7 章补充核验）

**链接有效率抽检（8 条）**：GitHub 站内 gallery 链接 200 OK；官网 gpt-image2.canghe.ai 200 OK；npm 包 gpt-image-2-style-library 200 OK（v1.0.4）；OpenAI developers 文档（gpt-image-2.5-sunburst）经代理读取确认为真实模型页；trendshift.io 徽章页 403（反爬可达，浏览器正常）；x.com 源链接与 2 个赞助商域名在本分析环境连接被阻断，无法判定 404 与否。抽检未发现 404，GitHub/官网/npm/OpenAI 四类关键链接全部有效。

## 10. 总结评价

总评：这是 2026 年 AI 图像生成热潮中"内容型仓库产品化"的最佳样本之一——它没有停留在 awesome 清单式链接堆砌，而是完成了三级跳：案例逆向整理（541 条全图全提示词、95% 可溯源）→ 结构化协议（22 套带防坑指南的模板 + 13×19×10 标签体系）→ Agent 资产分发（Skill/npm/插件市场），同时用一套完整的积分网站把流量沉淀为可持续的商业闭环。单人 6.5 个月做到 36k 星，靠的是选题踩点（GPT-Image2 发布窗口）、内容密度（真材实料可复制即用）和传播设计（三语 README + Trendshift 徽章 + 公众号）的合力。

与 awesome-gpt 类清单对比：典型 awesome 仓库（如 awesome-gpts 系列）是第三方链接的目录索引，条目轻、验证弱、涨星靠"广"；本仓库条目重（每条约 1.2k 字符提示词 + 配图 + 出处），可离线复现，且"抄结构 → 填模板 → Agent 调用"的递进用法让它对自动化工作流有真实生产力，涨星靠"深 + 快"。两者生态位互补：要浏览选 awesome 目录，要干活用本仓库。

适合谁：做 AI 出图内容的创作者/运营（直接抄案例与模板）；给 Agent/工作流接图像生成能力的开发者（Skill + JSON 模板是现成的提示词中间层）；想研究"内容仓库如何产品化与商业化"的独立开发者（本仓库的 sponsors/积分/付费社区/防盗链是一套可照抄的运营模板）。想读"代码"的人要注意：它的核心资产是 `data/style-library.json` 的 schema 设计与 `scripts/generate-site-data.mjs` 的内容管线，而非 4315 行的画廊前端。
