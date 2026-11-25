# 1c7/chinese-independent-developer 项目分析

> 星数 61905（快照 2026-10-24）｜主语言 —｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：**中国独立开发者项目清单**——聚合所有中国独立开发者自建网站/App 的长青目录，靠"开发者自助提交 + AI 自动化收录流水线"持续生长。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | 1c7/chinese-independent-developer | gh api |
| 星数 | 61905（快照）；分析日 gh api 实测 61,992 | gh api |
| 主语言 | 无代码主体（Python 仅用于自动化脚本） | 仓库结构 |
| 许可证 | 无 LICENSE（gh api license=null，tarball 内无 LICENSE 文件） | gh api / tarball |
| 创建时间 | 2017-09-21 | gh api |
| 最近活跃 | 2026-11-02（分析前一天仍在收录，pushed_at 2026-11-02T15:25Z） | gh api |
| Forks / Watch | 5,576 / 1,348 | gh api |
| 条目规模 | 四份清单合计约 3,450 个产品条目、1.16 万行 | 本地统计（见第 7 章） |

## 2. 项目是什么，解决什么问题

这是 GitHub 上最知名的中国独立开发者作品目录。作者 1c7 在 README 开头交代动机：作为开发者好奇其他人在做什么业余项目（盈利/玩票/试试看），于是建了这个库，欢迎开发者通过 Pull Request 或 Issue 把自己的项目加进来（`README.md` 第 11-14 行）。

它解决的其实是两个问题。对**浏览者**：在一个地方集中发现中国独立开发者做了什么、哪些还活着——每条带一句话介绍和三态标记（:clock8: 开发中 / :white_check_mark: 已上线 / :x: 已关闭，`README.md` 第 19-23 行）。对**提交者**：这是一个流量入口和归属地，作者只需要按模板在 Issue #160 下评论或开 Issue，就能进入主列表。

收录标准很克制：必须是网站或 App，不能是开发者工具或论坛型网站（`README.md` 第 14 行）；SKILL.md 把拒绝类别进一步收窄为垃圾广告、非中国开发者、大厂产品、返佣链接、刷量平台（`.claude/skills/chinese-indie-dev/SKILL.md` 硬红线第 5 条）。这种"宁窄勿滥"的边界让列表始终是"打开即用的真产品"合集，而不是大杂烩。

仓库本身几乎没有传统意义的"代码"——它的主体是 README.md（476KB）与三份分册，但它围绕这份清单建了一套完整的工程化设施：提交格式模板（`.github/CONTRIBUTING.md`）、Issue 模板（`.github/issue_template.md`）、Python 自动化脚本、GitHub Actions 工作流、Claude Code 运维 skill 和 unittest 测试。这是它区别于普通 awesome 清单的最大特征：**把人工维护目录这件事做成了可持续的自动化流水线**。

主 README 末尾还维护着"基于本列表数据源的产品"生态区（`README.md` 第 5209-5216 行），收录了 4 个第三方开发者用列表数据做的可视化/搜索产品，形成"数据源 → 衍生品 → 反哺曝光"的生态位。

## 3. 技术栈与依赖

清单本体是纯 Markdown，无运行时依赖。自动化设施的技术栈（均为仓库内实证）：

- **Python 3.13**：脚本语言（`.github/workflows/process_list.yml`）。
- **PyGithub**：读写 Issue 评论、创建分支与 PR（`process_item.py` 第 5 行 `from github import Github`）。
- **OpenAI SDK + DeepSeek**：条目文案格式化，`BASE_URL` 默认 `https://api.deepseek.com`，模型 `deepseek-v4-pro`，temperature 0.3（`process_item.py` 第 49-85 行；`process_list.yml` 注入 `LLM_API_KEY`/`LLM_BASE_URL`）。
- **Claude Code Routines**：`trigger_claude_routine.yml` 每 6 小时 POST Anthropic 的 routine fire 接口，驱动 `.claude/skills/chinese-indie-dev` 的 agent 流程（这是当前主力，`process_list.yml` 已降级为手动备用）。
- **Python 标准库**：流量采集 `collect_traffic.py` 明确"only the Python standard library"，无第三方依赖（脚本 docstring；`TRAFFIC.md`）。
- **测试**：标准库 `unittest` + mock（`.github/tests/`）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `README.md` | 主版面清单本体，476KB / 5250 行，按"日期 → 作者 → 产品行"三级组织 |
| `.github/pages/README-Earlier-Projects.md` | 2018~2025 年历史归档分册，302KB / 4175 行 |
| `.github/pages/README-Programmer-Edition.md` | 程序员版面（需要命令行/写代码的产品），145KB / 405 条 |
| `.github/pages/README-Game.md` | 游戏版面，45KB / 155 条 |
| `.github/CONTRIBUTING.md` | 条目格式模板 + "如何写好一句话介绍"指南 |
| `.github/MAINTAINER.md` | 收录流水线说明书：🚀 表情 → 脚本 → PR → merge |
| `.github/TRAFFIC.md` | 访问统计采集方案文档 |
| `.github/scripts/process_item.py` | 核心：扫描被 🚀 标记的提交，LLM 格式化，建 PR |
| `.github/scripts/pending_submissions.py` | 按域名+产品名两个维度查重，输出 新/同名/无链接 三类标签 |
| `.github/scripts/migrate_archive.py` | 容量管理：把最旧日期区块搬进历史分册（预演/`--write` 双模式） |
| `.github/scripts/collect_traffic.py` | 每周归档仓库访问数据到 `traffic-data` 分支 |
| `.github/tests/` | unittest 测试：`test_process_item.py`、`test_collect_traffic.py` |
| `.github/workflows/` | 3 个工作流：process_list / trigger_claude_routine / collect_traffic |
| `.claude/skills/chinese-indie-dev/` | AI 运维手册：`SKILL.md`（433 行流程+硬红线）、`references/precedents.md`（193 行判例库） |
| `.claude/settings.json`、`.gitignore`、`issue_template.md` | 常规配置 |

组织方式评述：这是典型的"内容仓库 + `.github` 运维设施"结构。清单按容量拆成 1 主册 + 3 分册（主册只留 2026 年后"打开即用"类，程序员/游戏单独成册，历史归档另册），内容与工具彻底分离；没有构建系统、没有包管理文件，Python 脚本靠 CI 内 `pip install PyGithub openai` 直接运行（`process_list.yml` 第 19 行）。

## 5. 核心架构与关键实现

**（1）收录流水线：表情符号驱动 + LLM 格式化**（`.github/scripts/process_item.py`）

完整链路见 `.github/MAINTAINER.md`：用户在 Issue #160 评论提交 → 维护者给评论点 🚀 → 脚本扫描"近 3 天带 🚀 且无管理员 🎉 的评论"（第 98-126 行 `collect_pending_items`）→ 清洗引用块、修复富文本双重嵌套链接（第 32-45 行）→ LLM 去营销词、规范为 `* :white_check_mark: [名](网址)：用途` 格式（第 47-86 行）→ 插入主 README 当日日期节 → 建分支开批量 PR → 回评论"感谢提交，已添加！"并点 🎉。触发判定只认**管理员本人**的表情，防止伪造（第 88-96 行）。

```python
def check_reactions(item):
    reactions = item.get_reactions()
    has_trigger = any(r.content == TRIGGER_EMOJI and r.user.login == ADMIN_HANDLE for r in reactions)
    has_success = any(
        r.content == SUCCESS_EMOJI and r.user.login == ADMIN_HANDLE
        for r in reactions
    )
    return has_trigger, has_success
```

**（2）AI Agent 运维手册：SKILL.md + 判例库**（`.claude/skills/chinese-indie-dev/`）

当前主力机制是每 6 小时由 Claude Routine 触发 agent 跑一遍收录流程。`SKILL.md`（433 行）定义了 11 条硬红线（不得删改他人条目、合并前必须逐行 `gh pr diff` 审查、格式问题不能作为拒绝理由、状态图标以提交者为准等）、72 小时窗口扫描（#160 评论 + open Issue + open PR）、只按"域名+产品名"查重（明确禁止按提交者去重——同一作者跨月反复投稿是常态）、以及 490,000 字节的容量熔断线。`references/precedents.md` 沉淀了 A~L 共 12 组历史判例（身份模糊、返佣边界、误判纠正、PR 合并事故、致谢串号等，时间戳到 2026-10-22），形成"事故 → 判例 → 下不为例"的运维知识库。

**（3）容量管理：自动分册搬迁**（`.github/scripts/migrate_archive.py`）

GitHub 渲染大文件有实际极限，脚本设定主 README 超过 490,000 字节就把最旧的日期区块剪到 `.github/pages/README-Earlier-Projects.md`（docstring：一次剪到默认 target 487,500；兼容"日/号"两种历史日期写法）。当前主 README 实测 486,476 字节，最近一次提交就是"第 23 次存档搬迁：2025-06-06、2025-06-10 区块移入历史分册"（gh api commits，2026-11-02）。

**（4）自助流量统计**（`.github/scripts/collect_traffic.py` + `.github/workflows/collect_traffic.yml`）

纯标准库实现，每周一 09:23（北京）把 views/clones 日数据合并进独立 `traffic-data` 分支的 `traffic.json`，窗口重叠做覆盖更新、不重复累计，快照保存 14 天滚动来源与热门页（`.github/TRAFFIC.md`）。用独立分支避免统计提交污染 master 历史。

## 6. 构建与运行

无构建、无包管理文件，均为脚本级运行（来自仓库文档）：

```bash
# 收录脚本本地调试（.github/MAINTAINER.md）
cp .env.example .env        # 配置 PAT_TOKEN / LLM_API_KEY
uv sync
uv run .github/scripts/process_item.py

# 查重脚本（SKILL.md 0.3 节）
gh api "repos/1c7/chinese-independent-developer/issues/160/comments?since=$SINCE&per_page=100" > /tmp/c160.json
python3 .github/scripts/pending_submissions.py /tmp/c160.json

# 容量搬迁（先预演再写入，migrate_archive.py docstring）
python3 .github/scripts/migrate_archive.py
python3 .github/scripts/migrate_archive.py --write

# 测试（.github/TRAFFIC.md；同目录另有 test_process_item.py）
python3 -m unittest discover -s .github/tests -p 'test_collect_traffic.py' -v
```

## 7. 收录范围、分类体系与条目规模（本地实测）

- **主版面** `README.md`：5250 行 / 486KB；399 个"YYYY 年 M 月 D 号添加"日期节（2026 年以来），1397 个作者条目（`####`），1690 个产品行；状态分布：✅ 1656、🕗 6、❌ 45（`grep` 统计）。
- **历史分册** `.github/pages/README-Earlier-Projects.md`：4175 行，1199 个产品行，最早日期节"2018年3月17号添加"（grep）。
- **程序员版面** `README-Programmer-Edition.md`：405 个产品行；**游戏版面** `README-Game.md`：155 个产品行。
- **合计约 3,450 条**，时间跨度 2018-03 至 2026-11，近九年持续追加。
- **分类体系**：不按品类分，而按"版面（主/程序员/游戏/历史）→ 添加日期 → 作者"三级组织；产品行内不设标签，靠一句话介绍自述品类。收录范围限定"网站或 App"，开发者工具、论坛、博客、企业官网明确不收（`README.md` 第 14 行、SKILL.md 判例 B5/B6）。
- **条目格式**严格统一（`.github/CONTRIBUTING.md` 模板）：`#### 名字(城市) - [Github](url)` + `* :状态: [产品名](网址)：一句话介绍 - [更多介绍]()`。CONTRIBUTING 还专设"先定性，再加分"的介绍语写法教程，并给了 10 条本列表真实范例。

## 8. 更新机制与社区生态

- **更新节奏**：分析前一天（2026-11-02）即有 5 次以上 push，含单人收录、批量收录（"收录 2026-11-02：RapMotion、Dayfly Mail…"）与第 23 次存档搬迁（gh api commits）。日期节粒度到"日"，说明几乎是日更。
- **贡献者结构**：贡献者列表首页即满 100 人（gh api contributors?per_page=100 返回 100）。但高度集中：1c7 本人 3040 次贡献，第二名 lazyglow 91 次，第三名 23 次；机器人账号 `claude` 也在贡献者之列（9 次）——所有 merge 实际由 1c7 一人完成，自动化只压缩了他的操作次数（"我大概点击 3 次左右就可以了"，`MAINTAINER.md`）。
- **生态位与衍生品**：主 README"基于本列表数据源的产品"节收录 4 个第三方衍生品（AI 独立制造所/indiemaker.cn、网页版、独立星球、独立观察），README 明确"本列表本身永久以 GitHub 仓库形式存在"（第 5209-5216 行）。
- **周边资源区**：末尾维护英文/中文独立开发社区资源（Indie Hackers、V2EX 分享创造、电鸭远程工作等）与推荐 Twitter 账号（`README.md` 第 5221-5250 行）。
- Release：0 个（gh api）——纯内容仓库，无版本概念。
- 治理细节：issue 仅剩 1 个 open（即 #160 提交专用 Issue）；2026-09 起引入 agent 运维后，判例库记录了多起"误判纠正"（如 B3 书径 EbookNest），说明存在人工复核回路。

## 9. 亮点与风险

**亮点**

1. **把"人工维护目录"工程化成流水线**：表情触发 → LLM 格式化 → 自动 PR → 逐行审查合并，单人靠自动化撑起日更 3400+ 条目的清单（`process_item.py`、`MAINTAINER.md`），这在同类 awesome 清单里极少见。
2. **AI Agent 运维知识库范本**：SKILL.md 的硬红线 + 按需加载判例库（"不要在执行时全量加载"）+ 72h 窗口与三个计数的运行契约，是可直接借鉴的 agent 生产线设计。
3. **尺度纪律**：拒绝类别白名单化（只有 5 类可拒）、状态图标以提交者为准、查重只看域名+产品名——规则具体到可执行，减少主观判断摇摆（SKILL.md 硬红线）。
4. **前瞻性容量设计**：490KB 熔断线 + 自动分册搬迁 + 先预演后写入，提前化解 GitHub 大文件渲染极限（`migrate_archive.py`）。
5. **质量回归测试**：unittest 覆盖了"贡献者的 🎉 不应误判为已完成""PR 不得被当作 Issue 扫描"等真实事故场景（`test_process_item.py`）。

**风险/局限**

1. **单点依赖**：所有合并决策在 1c7 一人；自动化依赖其 PAT_TOKEN、DeepSeek 与 Claude Routine 付费密钥，密钥或 API 变动即停摆（`process_list.yml`、`trigger_claude_routine.yml`）。
2. **无许可证**：内容在法律上不可复用，虽有 4 个衍生品实际存在，但均处于灰色地带（gh api license=null）。
3. **LLM 处理质量风险**：条目文案由 `deepseek-v4-pro` temperature 0.3 生成，仍靠人 merge 前把关；SKILL.md 判例库里的多起事故说明流程踩坑成本真实存在。
4. **条目腐化**：1690 条主版面条目中仅 45 条标记 ❌，实际失效比例大概率更高；目录无死链巡检机制（未确认是否有站外验证，SKILL.md 判例 C 只处理"打不开 ≠ 有问题"）。

## 10. 总结评价

这是中文互联网最大的独立开发者产品目录，也是"内容仓库 + AI 自动化运维"的一个优秀样本：本体只是一份 Markdown 清单，但它用格式模板、LLM 格式化、表情触发、agent 运维手册、判例库和单元测试，把一件通常会烂尾的人工维护工作做成了九年日更、单人可持续的系统。它的星数增长逻辑很清楚——每位提交者都会分享自己的收录链接，列表因此自带自增长飞轮，而"打开即用 + 状态诚实"的收录纪律维持了浏览价值。适合三类人：想找中国独立开发产品/竞品参考的开发者和用户（直接浏览）；想做 awesome 清单的维护者（抄收录流水线和 SKILL.md 的治理设计）；研究"人 + agent 协作维护开源资产"的工程实践者（读 `.claude/skills/chinese-indie-dev/` 全套）。源码意义上没有可读的程序架构，但那套运维设施本身就是它最值得阅读的"源码"。
