# VoltAgent/awesome-openclaw-skills 项目分析

> 星数 53284（快照 2026-10-24）｜主语言 —（纯 Markdown 内容仓）｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：**OpenClaw（本地运行的 AI 助手）社区 skills 的官方级精选清单**——从 ClawHub 公共注册表数千个技能中过滤、分类，做成一张可检索的"技能货架"。

| 项目 | 内容 |
|------|------|
| 名称 | VoltAgent/awesome-openclaw-skills |
| 星数 | 53,284（快照）；50,72 forks（`gh api` 2026-11-04） |
| 语言 | 无代码，纯 Markdown 清单（README 137KB + categories/ 856KB） |
| 许可证 | MIT（LICENSE，Copyright (c) 2026 VoltAgent） |
| 创建时间 | 2026-01-25（`gh api repos/VoltAgent/awesome-openclaw-skills`） |
| 最近推送 | 2026-10-30（同上），持续高频更新中 |
| 主页 | https://clawskills.sh/（团队自建配套站点） |
| 规模 | 30 个分类、5,213 条收录条目（`categories/*.md` 统计） |

## 2. 项目是什么，解决什么问题

OpenClaw 是一个在本机运行的 AI 助手（前身名为 Clawdbot，README 中仍残留 32 处旧名，issue 链接仍指向 `VoltAgent/awesome-clawdbot-skills`，可见仓库经历过改名——`README.md`、`CONTRIBUTING.md`）。它的能力靠"技能"（skills，即 SKILL.md 指令包）扩展，社区把技能发布到公共注册表 ClawHub。问题在于：注册表短期内涌入了上万个技能，质量参差、良莠难辨，普通用户没法从噪音里找到可用之物。

本仓库的角色是 **ClawHub 之上的一层人工+半自动过滤层**。`README.md` 的"Why This List Exists"一节给出了明确的筛选账目：从官方注册表剔除了 7,215 个条目——疑似垃圾/机器人账号 4,065、重名/相似名 1,040、低质或非英文描述 851、加密货币/金融/交易类 886、被研究者安全审计标记为恶意的 373。最终收录 5,200+ 条，按 30 个主题分类呈现。

因此它的价值不是"最全"，而是"过滤后的可得性"：一页之内按类浏览、一键拿到安装命令、并附安全风险提示（见第 7 章）。对一个爆发期生态来说，这类"入口型"仓库天然承载了巨大的检索需求——这也是它 9 个月冲到 5 万+ 星的直接原因（创建于 2026-01-25，恰逢 Clawdbot 改名 OpenClaw 生态爆发窗口）。

## 3. 技术栈与依赖

无传统技术栈，工程化程度极低但流程闭环完整：

- **内容**：纯 Markdown（`README.md` 主列表 + `categories/*.md` 分明细页），无生成脚本入库；
- **CI**：1 个 GitHub Actions 工作流 `.github/workflows/pr-check.yml`，用 `actions/github-script@v7` 正则校验 PR 描述里必须含 `https://clawhub.ai/<author>/<skill>` 链接，不合规自动评论并 fail；
- **辅助工具痕迹**：`.claude/settings.local.json` 中保留了维护者的本地权限配置（`Read(//Users/necatiozmen/...)`、`Bash(python3 /tmp/categorize.py)`），说明分类工作借助 Claude + 临时 Python 脚本半自动完成，但脚本未入库；
- **生态链接**：安装走 OpenClaw CLI（`openclaw skills install <slug>`）或 `npx clawhub install <skill-slug>`（`README.md` Installation 节）。

## 4. 目录结构与代码组织

| 路径 | 职责 |
|------|------|
| `README.md` | 主清单（1278 行）：品牌横幅、安装方式、筛选统计、赞助位、30 类的"预览列表"（每类展示约 20 条 + "View all N skills →" 链到明细页）、贡献指引 |
| `categories/` | 30 个分类明细文件，共 5,213 条目（合计 856KB），每文件头部含条目数与返回主列表链接 |
| `.github/workflows/pr-check.yml` | PR 描述 ClawHub 链接强制校验 |
| `CONTRIBUTING.md` | 收录规则：格式、门槛、PR 标题规范、安全政策 |
| `LICENSE` | MIT |
| `.claude/settings.local.json` | 维护者本地 AI 工具权限配置（未清理的 incidental 文件） |

组织方式评述：**"README 预览 + 分类明细页"的双层结构**是 awesome 清单在条目破千后的典型演进——主 README 保持可浏览性，大文件外置避免渲染卡顿。所有条目统一格式 `- [skill-name](https://clawskills.sh/skills/author-skill) - 一句话描述`（`CONTRIBUTING.md` 规定描述 ≤10 词），机器可解析、人类可扫读。分类文件条目数与 README 目录标注略有漂移（如 Coding Agents & IDEs 目录标 1184，明细文件自称 1202 skills、实数 1162 条，`categories/coding-agents-and-ides.md`），属手工/半自动更新的正常滞后。

## 5. 核心架构与关键实现

内容仓无代码架构，"核心实现"即其**筛选与分类体系**：

1. **准入过滤（负向清单）**：五类硬剔除标准——垃圾账号、重名、低质描述、crypto/金融、恶意技能（数字见 `README.md` 筛选表）。其中"恶意——依据研究者公开的安全审计"一条是明确的供应链安全立场。
2. **准入门槛（正向标准）**（`CONTRIBUTING.md` Requirements 节）：必须在 ClawHub 发布、其测试通过且安全状态非 suspicious、有 SKILL.md 文档、描述 ≤10 词、**必须有真实社区使用量**（明确拒绝"刚写完 3 小时的新技能"，README 结尾 Note 重申），同领域多技能只收代表作。
3. **分类体系**：30 类覆盖开发（Coding Agents & IDEs 1162 条、Web & Frontend 905、DevOps & Cloud 376）、个人效率（Productivity & Tasks 210、Notes & PKM 70）、生活（Health & Fitness 87、Transportation 109、Gaming 26）乃至生态自指类（Clawdbot Tools 37、Moltbook 35）。开发类占绝对大头，反映用户画像是开发者。
4. **质量护栏自动化**：`.github/workflows/pr-check.yml` 用一行正则 `/https:\/\/clawhub\.ai\/[\w-]+\/[\w-]+/` 把"只收 ClawHub 已发布技能"的规则变成机器强制，未附链接的 PR 会被机器人自动评论打回——规则-工具-文档三者一致，是清单型仓库里少见的严谨。

## 6. 构建与运行

无构建。使用方式（`README.md` Installation 节）：

```bash
# OpenClaw CLI 安装
openclaw skills install <skill-slug>
# 或 ClawHub CLI
npx clawhub install <skill-skill>
```

手动安装则把技能目录放入 `~/.openclaw/skills/`（全局）或 `<project>/skills/`（工作区，优先级最高）；也可以直接把技能的 GitHub 链接粘给助手让它自行安装。仓库本身如需贡献：编辑对应 `categories/*.md` 与 `README.md` 同步条目，PR 描述附 ClawHub 链接，标题格式 `Add skill: author/skill-name`（`CONTRIBUTING.md`）。

## 7. 收录流程与质量控制（内容仓特例：对应"测试与工程质量"）

- **人工审核 + 机器前置校验**：PR 必须过 `pr-check.yml` 的链接校验，之后人工复核（"We review all submissions and may decline skills"）；
- **溯源强约束**：只收 ClawHub 发布物，拒绝个人仓库/gist——好处是每个条目都有注册表背书与安全扫描可查（ClawHub 有 VirusTotal 合作扫描）；
- **免责声明完整**：`README.md` Security Notice 明示"curated, not audited"（筛选≠审计），技能可能在收录后被上游修改，并推荐 Snyk agent-scan、Agent Trust Hub 等第三方扫描工具；License 节再次声明不审计、不背书。安全话语体系在同类清单里是最完备的；
- **质量观察**：条目描述本身质量参差（不少条目描述是被截断的半句话，如 `categories/coding-agents-and-ides.md` 中 "3d-cog - Other tools need perfect images."），反映出收录规模远超人工打磨能力；README 中 clawhub.ai 直链仅 47 条而 clawskills.sh 链接 794 条，条目实际指向自建站点而非注册表原页，与 CONTRIBUTING 里"不要提交 clawskills.sh 链接"的规定形成微妙张力。

## 8. 社区与维护状态

- **活跃度**：创建于 2026-01-25，最近推送 2026-10-30（`gh api`），9 个月持续高频更新；README 徽章动态显示 skills 数（5200）与 last-commit；
- **社区规模**：53.3k stars / 5,072 forks / 308 watchers / 仅 7 个 open issues（`gh api` 2026-11-04）——issue 极少说明维护方消化能力强或讨论迁移到了 Discord（徽章链接 VoltAgent Discord）；
- **维护者**：VoltAgent 团队（TypeScript AI agent 框架公司），`.claude/settings.local.json` 中可见具体维护者（necatiozmen）本机路径；作者署名横幅含 X 账号 @nozmen；
- **商业化**：README 内嵌 Crawlbase、Cloudways（含推广码 VOLTAGENT）、SerpApi 三个赞助位（`README.md` OpenClaw Ecosystem Tools 节），是典型的"流量入口变现"；
- **姊妹项目**：同组织下的 `VoltAgent/awesome-codex-subagents`、`VoltAgent/official-mcp-servers`（README 徽章与引用），构成"agent 生态入口"产品矩阵。

## 9. 亮点与风险

**亮点**
1. 卡位极准：在 OpenClaw 生态爆发初期（2026-01）第一时间做了注册表的过滤层，形成"装技能先来这里查"的默认心智，9 个月 5.3 万星；
2. 筛选可信：给出透明的前期剔除账目（7,215 条，含 373 条经安全审计的恶意技能），并明确拒绝 crypto 类与"婴儿期"技能；
3. 流程工程化：PR 链接机器强制校验 + 明文格式规范 + 双层文档结构，清单规模破 5,000 条仍可维护；
4. 安全话语完备：从准入（ClawHub 安全状态）到免责（curated, not audited）到工具推荐（Snyk/Agent Trust Hub）全链条覆盖。

**风险/局限**
1. 条目指向大量使用自建域名 clawskills.sh 而非 ClawHub 原页，单一方对链接资产的控制度偏高，清单位与注册表存在利益耦合（赞助位+引流）；
2. 收录即安全表象：仓库明确"not audited"，但 5.3 万星的权威光环会让普通用户放松警惕，而上游技能可随时被篡改（供应链风险客观存在）；
3. 数据一致性弱：目录计数、文件内自称计数、徽章计数（5200/5400+）三者互相矛盾，条目描述常有截断，提示半自动流水线缺少收尾校验；
4. 无任何自动化链接有效性检查（无 CI 死链检测），5,213 条外链的腐化风险随时间累积。

## 10. 总结评价

这是"内容型仓库吃生态红利"的教科书样本：技术含量为零，但时机（OpenClaw 更名爆发期）、卡位（注册表与用户之间的过滤层）、执行（透明筛选标准+机器强制贡献门槛）三者叠加，使其成为事实上的 OpenClaw 技能分发入口，9 个月做到 5.3 万星并撑起配套站点 clawskills.sh 与赞助变现。它适合两类人：想装技能的 OpenClaw 用户（当目录用，但装前务必按其 Security Notice 自查上游代码）；以及研究"awesome 清单如何运营与变现"的社区运营者（当方法论案例读，读 `CONTRIBUTING.md` + `pr-check.yml` 的规则设计即可）。对安全敏感的生产环境，不应把"进了这个清单"当作任何意义上的信任凭据。
