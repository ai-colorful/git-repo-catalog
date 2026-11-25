# alchaincyf/nuwa-skill 项目分析

> 星数 38706（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：女娲.skill 是一个 Claude Code / Agent Skills 格式的"造 skill 工具"——输入一个人名，引导 LLM 走完调研→提炼→成形→验证的六步流水线，把公众人物的思维方式蒸馏成一个可运行的人物 skill（心智模型、决策启发式、表达 DNA）。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | alchaincyf/nuwa-skill（女娲.skill，huashu-nuwa） | package.json、SKILL.md |
| 星数 / Fork | 38,706 / 4,937 | gh api 2026-11-21 |
| 语言 | Python（GitHub 统计；实际主体是 Markdown 方法论文档，Python 仅为 4 个辅助脚本约 800 行） | 文件清单核实 |
| 许可证 | MIT，Copyright (c) 2026 Huashu（花叔） | LICENSE |
| 创建 | 2026-04-05，7.5 个月涨到 38.7k 星 | gh api |
| 最近活跃 | 2026-11-17 推送 v2.0.2（release 与最后 commit 同日），持续高频迭代 | gh api commits/releases |
| 版本 | v1.0.0 → v2.0.2（5 个 tag），2026-11-10 一天内连发 v2.0.0/v2.0.1 | gh api tags |
| 贡献者 | 默认分支 10 人，花叔本人 72 次提交居首；社区贡献以 PR 并入（#78、#103 等） | gh api contributors |
| 社区生态 | COMMUNITY.md 收录 11 个第三方项目；官方另有 13 个独立人物 skill 仓库 | COMMUNITY.md、README |
| 规模 | 263 个文件，42MB（大量图片/promo 素材）；Markdown 代码约 7500 行，Python 约 800 行 | tarball 统计 |
| 开放 issue | 仅 1 个（issue 处理及时或讨论走 issue 即关闭） | gh api |

## 2. 项目是什么，解决什么问题

这是 2026 年 "AI Agent Skills" 生态热潮中的代表性项目。Anthropic 的 skill 规范（SKILL.md + YAML frontmatter + references/ 按需加载）普及后，出现了一批"用 skill 造 skill"的元工具。女娲的切入点是"人物思维蒸馏"：它自己不是某个人物的角色扮演 skill，而是生产这类 skill 的工厂。README（src/README.md）明确其出发点是借势 titanwings/colleague-skill（"同事.skill"）——既然能把离职同事蒸馏成 AI 分身，不如蒸馏留下大量公开材料的乔布斯、芒格、费曼。

技术本质上，这是一套**纯提示词工程的流水线**：没有任何传统意义的软件。SKILL.md（235 行）是运行时加载的编排指令，把 LLM 定位为"传记作者+认知研究者+编辑+质检员"四位一体，再按阶段指引去读 references/ 下的分阶段手册（采集 research-playbook、提炼 extraction-framework、成形 skill-template、验证 validation-playbook）。三个 Python 脚本是流水线里的机检工具，不是功能本体。

核心设计思想是"增量导向"（src/SKILL.md「女娲造的是什么」一节）：先让模型盲写先验快照（00-prior.md），之后调研只找三类东西——推翻先验的、比先验深的、先验没有的；"模型本来就答得出的内容，不占 Skill 的篇幅"。这条判据贯穿采集（每路底稿末尾必须列"和先验不一样的地方"，合计≥8 条才算过关）、提炼（"不看这个 Skill 的模型会不会也这么说？会，就删"）和验证（带 skill 与裸答两臂对照）。

质量与诚实机制是本项目最下功夫的部分：封存件（holdout.md）——提炼前挑 3-5 件一手材料封存、内容不进 skill，留作验证时的独立考题，防止"评分依据就是 skill 自己"的自证循环；双/三臂盲测（带 skill / 裸答 / 旧版，评分代理不知情）；保真度评分卡 5 维度 100 分（references/fidelity-scorecard.md，开篇引用 SkillLens 论文"LLM 自评准确率仅 46.4%"作为双 agent 独立评审的依据）；以及"窄桥"十二条硬规则（不编造引语、知乎/公众号/百度百科列入来源黑名单、负面评价必须有专门材料等）。

需要客观指出：README 中的对话示例（Naval/马斯克/乔布斯答问）是作者自制的宣传样例，无法从仓库内独立验证其真实性；"15 个官方 Skill 全员 A 级（89-97 分）"的评分也由作者自己按自家评分卡跑出（examples/*/FIDELITY.md 有完整答卷与评审留痕，芒格一例用了三臂盲评且明确记录了新版 Skill 的失分点，过程透明度较高，但答题/评分/出题环节仍在同一作者控制下）。

## 3. 技术栈与依赖

- **"语言"**：GitHub 标注 Python，实为 Markdown（约 7400 行方法论与产物文档）+ Python 3 标准库脚本。quality_check.py 与 merge_research.py 均 import re/sys/pathlib，零第三方依赖（src/scripts/quality_check.py:13-15、src/scripts/merge_research.py）
- **运行宿主**：任何兼容 Agent Skills 协议的 runtime（Claude Code、Codex、Cursor、OpenClaw、DeepSeek Harness 等，README 称 50+）；跨 runtime 安装走 vercel-labs/skills 的 `npx skills add`
- **DeepSeek Harness 插件**：package.json 含 `dsh.bundle` 字段，cordis.patch.yml 复用 DSH 的文件系统 skill 提供方把 node_modules 目录注册为 skill 根
- **辅助工具链**：download_subtitles.sh 依赖外部 yt-dlp（优先人工字幕，语言优先级中>英>其他）；srt_to_transcript.py 清洗 SRT/VTT 为纯文本
- **CI**：GitHub Actions 两个 workflow——community-pr-check.yml（PR 只许改 COMMUNITY.md，跑 .github/scripts/community_check.py 用 GitHub API 核验收录仓库的 FIDELITY.md 真实性）与 retire-mcstomach.yml（定时摘除 README 里的到期推广横幅，侧面印证其运营节奏）
- 依赖声明为零：无 requirements.txt/pyproject.toml，脚本仅用标准库

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| SKILL.md | 女娲本体：编排六步蒸馏流程、入口分流、窄桥规则、成本分档（快速/标准/深度） |
| references/ | 8 份手册共约 976 行：采集/提炼/验证/诊断 playbook、skill 模板、保真度评分卡、特殊场景、作者自介 |
| scripts/ | quality_check.py（415 行机检）、merge_research.py（208 行调研汇总）、srt_to_transcript.py、download_subtitles.sh |
| examples/ | 15 个已蒸馏产物（13 人物 perspective + 孙宇晨 + X 导师主题），每个含 SKILL.md、FIDELITY.md、references/research/ 全套调研底稿 |
| .github/ | 社区收录 PR 机器人 + 到期横幅摘除 workflow |
| promo/、assets/ | 推广物料：deck PDF、landing 长图、人物卡片、hero 动画（约 30MB，占仓库体积大头） |
| COMMUNITY.md / CONTRIBUTING.md | 社区生态索引与贡献规则 |
| package.json / cordis.patch.yml | npm 包元数据与 DeepSeek Harness 插件入口 |

行数分布（tarball 统计）：SKILL.md 235 行、references/ 共 976 行、scripts/ 共 802 行、examples/ 15 个 SKILL.md 合计 6439 行（最短芒格 237 行、最长孙宇晨 541 行），全部产物自带数量不等的 research/ 底稿。

组织方式评述：这是典型的"单 skill 仓库 + 内容生态"结构，而非软件工程的单体/monorepo。产物自包含是被写死的纪律（窄桥第 5 条：底稿必须落盘在产物目录内），每个 example 都是一份可独立审计的"蒸馏档案"——调研怎么做的、哪些是推翻先验的发现、盲测答卷与评审原文全部留档（如 examples/munger-perspective/references/research/validation-20261109/ 下 11 个文件）。examples 里芒格/乔布斯已升级到 v2 骨架（SKILL.md+casebook+voice），其余 13 个仍是 v1 格式，README 明确标注了这一点。

## 5. 核心架构与关键实现

**（1）六步蒸馏管线（src/SKILL.md「蒸馏流程」）**：定题（报成本分档，README 承认顶级模型单次蒸馏花费可达数十美元）→ 先验快照（盲写 00-prior.md）→ 采集封存（六路分工：著作/对话/表达/他者/决策/时间线，对应 research/01-06.md；有子代理并行、没有则串行降级，见 research-playbook.md 第 8 节）→ 提炼（招式 5-8 个、心智模型 3-7 个过"跨域复现/有生成力/有排他性"三重验证、案例卡 6-20 张、常见误读≥2 条）→ 成形（SKILL.md≤350 行 + casebook.md + voice.md 三层拆分以省运行时上下文）→ 验证（三项测试写 FIDELITY.md）。

**（2）机检器 quality_check.py（415 行，全仓库唯一的"硬"工程）**：对产物做结构与证据合规检查，分 v2/v1 两种骨架标准。检查项包括：description 长度（软 300/硬 1024 字，对应宿主限制）、SKILL.md 行数、绝对路径泄露、招式是否带出处、心智模型是否含"证据"与"失效条件"、案例索引与 casebook.md 编号一致性、一手来源占比是否过半、封存件 URL 是否泄漏进案例库（防考题污染）。输出逐项 ✓/!/✗ 加整改建议，有 ✗ 时退出码 1。实现为纯正则+分段解析（无 AST/LLM），如下示意（src/scripts/quality_check.py:254-266）：

```python
nolimit = [s for s, b in subs if not re.search(r"不灵|失效|局限|不适用|limit", b, re.IGNORECASE)]
noevid  = [s for s, b in subs if not re.search(r"证据|evidence", b, re.IGNORECASE)]
if not 3 <= n <= 7:
    rep.add(BAD, "心智模型", f"{n} 个，应为 3–7 个", ...)
elif nolimit or noevid:
    rep.add(BAD, "心智模型", f"{n} 个；" + "；".join(lack), ...)
```

**（3）双盲验证协议（references/validation-playbook.md）**：三臂对照设计——带 skill 臂、裸答臂、（升级时）旧版臂，各自在隔离目录作答且不许读 research/；评分代理只拿答卷与 holdout.md 对照打分（0-2 分/题），且答卷在送评前要清掉身份声明、案例编号、材料自述三类"暴露身份"的字样。芒格示例的 FIDELITY.md 记录了裸答混入贝佐斯"遗憾最小化"这类失败模式，以及新版 skill 的三条失分点，协议确实被执行过。

**（4）社区收录机器人（.github/scripts/community_check.py）**：半自动审核——PR 只许改 COMMUNITY.md；对收录仓库用 GitHub API 核验 SKILL.md 存在、"诚实边界"章节、FIDELITY.md 总分≥70 且关键字段行（测试日期/总分/各维评分）非占位（专门设计了否定语境免疫与"只扫字段行不扫全文"的对抗规则，注释引用了 PR #70 与 2026-07-26 对抗审查 P1-22/P1-23 的先例）。这是把质量控制扩展到生态的尝试，罕见地把"防刷分"写成了代码。

**（5）自适应执行与生命周期（src/SKILL.md）**：SKILL.md 为不同能力的模型设计了"两种走法"——强模型按各步过关条件放手做，弱模型/受限环境照手册逐条执行；窄桥第 11、12 条明确要求无子代理环境直接串行、200k 窗口按三段分 会话续跑（底稿即断点）。人物 skill 有完整的更新/升级语义：「更新」只补三路新材料做增量修改；「升级」从旧底稿重新提炼并拆出 casebook/voice，旧版另存 SKILL-v1.md 备查再走验证。仓库自身也内置版本检查：每次使用读 .last-update-check，满 30 天用 `git ls-remote --tags` 比对 metadata.version，仅提示不自动更新（SKILL.md「版本检查」一节）。

## 6. 构建与运行

无构建、无测试套件、无可执行安装物。实际用法（README「安装」一节）：

```bash
npx skills add alchaincyf/nuwa-skill          # 跨 runtime 一行安装
# 或手动：git clone 到 ~/.claude/skills/nuwa-skill/ 等对应目录
dsh plugin --profile web add github:alchaincyf/nuwa-skill   # DeepSeek Harness
```

装好后对 agent 说「蒸馏一个芒格」即触发。仓库内两个脚本可直接跑（作者面向"照着走"路径提供的机检）：

```bash
python3 scripts/merge_research.py <人物skill目录>   # 调研摘要表
python3 scripts/quality_check.py <人物skill目录>    # 产物合规机检，✗ 退出码 1
```

## 7. 测试与工程质量

- **无单元测试/CI 测试**：对脚本本身没有任何 pytest/测试目录；quality_check.py 是被"当测试用"的工具，检查对象是 LLM 产物而非代码
- **"测试"即验证协议**：examples/*/FIDELITY.md 是测试报告，芒格一例附完整三臂答卷与评审原文（examples/munger-perspective/references/research/validation-20261109/），乔布斯一例有五份答卷（裸答/v1 两轮/v2 边界）
- **代码质量**：Python 脚本规范——UTF-8 重配置兼容 Windows GBK 控制台、函数职责清晰、退出码语义明确；正则遍布（如来源统计靠"一手/二手"标记词计数，merge_research.py:43-44），对非规范产物会误判，但作者在输出里给了人工核对的出路
- **文档质量极高**：SKILL.md 与 8 份手册是精心迭代过的作品——有"做坏了长什么样"的反模式小节（extraction-framework.md 各节末尾）、实测翻车案例反哺的规则（skill-template.md 注明七条运行纪律中第 2、5 条来自一次真实翻车）、明确的成本红线（"有用户一觉醒来花了 40 美元"）
- **示例内容有数据驱动延伸**：MrBeast 示例自带 4 个分析脚本（examples/mrbeast-perspective/scripts/：analyze_titles.py 标题模式统计、thumbnail_audit.py 封面审计、retention_curve_checker.py、fetch_youtube_subtitles.sh），说明主题/内容类人物的蒸馏被设计成"方法论 + 可复跑的分析工具"，超出纯提示词形态
- **多语言营销**：README 四语版本（EN/JA/KO/ES）+ star-history 徽章 + promo/ 目录，运营投入与工程投入几乎等量

## 8. 社区与维护状态

- **维护极活跃**：创建于 2026-04-05，最后推送 2026-11-17（分析日 4 天前）；11 月连发 4 个版本（v1.1.0→v2.0.2），v2.0.2 是社区 PR 修复（quality_check 对英文 skill 的适配，commit 2026-11-16/17，PR #103 来自 Kavender）
- **核心强收敛**：CONTRIBUTING.md 明文规定 SKILL.md 核心资产不接受外部 PR（issue 讨论后由维护者实现并致谢），贡献者结构是"单人核心 + 少量 PR 贡献者"（默认分支 10 人，花叔 72 commits）
- **生态外溢**：COMMUNITY.md 已收录 11 个社区项目（135+ 人物合集 awesome-nuwa、诸子辩论 skill、地缘政治 skill 等），收录由机器人预审+人工终审；另有 13 个官方人物 skill 拆成独立仓库（steve-jobs-skill 等，README 表格），形成 repo 矩阵互链
- **增长背景**（客观陈述）：2026 年 Agent Skills 协议生态爆发 + 同事.skill 热点的直接借势 + 作者自有渠道流量（B 站 14 万粉级别未确认、YouTube、公众号"花叔"、小红书，README「关于作者」）+ promo 物料投放（含 README 内限时合作横幅 mcstomach，由 retire workflow 定时摘除）。7.5 个月 38.7k 星的增速属于典型的"热点+流量+多 runtime 叙事"组合拳，与其内容质量和生态位共同作用，无法归因于单一因素

## 9. 亮点与风险

**亮点**
1. 方法论文档是真功夫：增量判据、心智模型三重验证、封存件独立考题、三臂盲测——把"评测 AI 产物"这件事设计成了可复现的协议，且 examples 里留了全套执行痕迹（出处：references/ 全部手册 + examples/munger-perspective/）
2. 诚实边界作为一等公民：来源黑名单、负面评价强制项、"交 60 分不交编造的 90 分"（窄桥第 7 条），在人物模拟这个极易滑向编造的赛道上立了明确的反幻觉规范
3. 上下文经济学自觉：350 行主文件 + references 按需加载的三层拆分、description 300/1024 字双阈值，都是针对 skill 运行时真实成本的工程化设计（src/scripts/quality_check.py:26-29）
4. 社区质量管控代码化：收录机器人对 FIDELITY.md 做占位检测与否定语境免疫（.github/scripts/community_check.py:19-45），把"防自评刷分"落成了可执行的对抗规则

**风险/局限**
1. 实效无法独立验证：所有对话示例与 A 级评分都出自作者之手，"蒸馏出的 skill 比裸答好多少"这一核心主张没有第三方复现；评分卡虽好，答题/出题/评审的代理仍由同一作者指挥
2. 主体是提示词而非代码：效果强依赖宿主模型能力与上下文窗口，且蒸馏一次数十美元的成本（SKILL.md 自述）抬高了真实使用门槛；GitHub "Python" 主语言标签与仓库实际形态有偏差
3. 名人肖像与内容风险：蒸馏在世公众人物的表达 DNA 并对外分发（孙宇晨、特朗普等已内置在仓库中），作者用"视角不是本人"的声明与伦理红线自限，但法律与舆论风险并未消除
4. 营销密度高：42MB 仓库里近 3/4 体积是 promo 素材与图片，README 含限时合作横幅与矩阵互链，商业运营属性明显，评估时需与工程内容分开看

## 10. 总结评价

女娲.skill 是 2026 年 AI skills 热潮中把"元技能"（造 skill 的 skill）做成爆款的最典型样本：以一套写得相当扎实的提示词方法论（增量导向调研 + 封存件盲测 + 诚实边界）为核心资产，配三个零依赖 Python 机检脚本和一个社区收录机器人，在 7.5 个月内借同事.skill 热点与作者自有流量冲到 38.7k 星。它的工程含量低于星数给人的直觉（可执行代码约 800 行），但它的"方法论工程"含量高于绝大多数同类项目——反幻觉设计、双盲验证协议和留档习惯在人物模拟赛道里少见。适合三类人：想学习如何给 LLM 设计可验证工作流的人（直接读 references/ 四份手册，这是仓库精华）、想给自己关心的领域搭建 skill 生产线的 prompt 工程师、以及研究 AI 生态传播与社区运营的观察者。把它当"乔布斯真的给你打工"来用，则应对产出的忠实度保持作者自己都承认的怀疑——直觉蒸不出来，快照会过期，公开表达本来就不等于真实想法。
