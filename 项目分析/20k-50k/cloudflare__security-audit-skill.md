# cloudflare/security-audit-skill 项目分析

> 星数 71725（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k（清单快照档位，实际星数已超 50k 线）｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：Cloudflare 出品的一个"编码代理 skill"（agent skill），把通用编码代理变成一名流程化、可验证的安全审计员——多阶段审计编排 + 机器可读 findings + 独立复核机制。

| 项 | 值 |
|---|---|
| 名称 | cloudflare/security-audit-skill |
| 星数 / Fork / Open Issues | 71725 / 4777 / 126（gh api，2026-11-14） |
| 主语言 | JavaScript（GitHub 判定；内容主体为 Markdown 指令文档） |
| 许可证 | MIT（`LICENSE`，Copyright 2025-2026 Cloudflare, Inc.） |
| 创建 / 最后推送 | 2026-06-18 / 2026-09-14（commit "Clarify guidance and full audit modes"，gh api） |
| 仓库体量 | 21 个文件，约 5276 行（tarball 实测），无 release/tag，3 名贡献者 |
| 关联背景 | README 自述为 Cloudflare 漏洞发现 harness 的单仓库起点，对应博客《Build your own vulnerability harness》 |

## 2. 项目是什么，解决什么问题

这是一个发布到 Claude Code / skills 生态的"技能包"（skill）：核心不是一个可执行程序，而是一套供 LLM 编码代理阅读执行的自然语言协议，外加两个零依赖的 Node.js 校验脚本。安装后，用户对代理说"security audit this codebase"即可触发完整的六阶段审计工作流（`README.md`、`skills/security-audit/SKILL.md`）。

它要解决的问题直指 AI 代码审计的通病（`SKILL.md` 反模式清单）：

- **幻觉漏洞**：代理常把"偏离清单项 / 缺少纵深防御"当成漏洞。该 skill 要求每条 finding 必须指名低信任主体、被跨越的信任边界、受影响资源与实际观察到的结果，否则不得确认。
- **自审不独立**：发现漏洞的代理自己验证自己。该 skill 规定"验证者永远不是发现者"——每个候选都交给一个全新的子代理去证伪（Phase 3），最终记录再由另一批全新代理逐条复核（Phase 5）。
- **输出不可机读**：prose 报告无法去重、无法程序化消费。该 skill 强制所有结论写入 `findings.json` 并通过 JSON Schema + 校验器验证。
- **覆盖不可度量**："我看过了"不构成覆盖证据。该 skill 用确定性的 `coverage-ledger.json`（入口面 × 信任边界 × 子系统 × 攻击类四维单元）追踪每个单元的状态。

工作流分六阶段（`SKILL.md` Full audit workflow、`RECONNAISSANCE.md`、`HUNTING.md`、`VALIDATION-AND-REPORTING.md`）：侦察建账本 → 按账本派遣隔离 hunter 波次 + 覆盖批评家 → 独立证伪候选 → 结构化输出过 schema → 全新代理逐条复核 → 生成目标中立报告（`REPORT.md` / `FINDINGS-DETAIL.md` / `NEEDS-VALIDATION.md`）。

Skill 有两种运行模式（`SKILL.md` Operating modes）：默认 guidance mode（只回答安全问题、不写文件），仅当用户明确要求审计/渗透测试时进入 full audit mode。结论分三种互斥判定：`confirmed`（完整源码链路 + 有界本地实证）、`needs_validation`（有精确的未决事实、不给严重级别）、`rejected`（被证伪的候选，保留以防重复）。README 明确这是防御性、源码优先的工作流，禁止探测线上端点、使用真实凭据或扩大影响。

## 3. 技术栈与依赖

- **Markdown 指令文档（约 4100 行）**：`skills/security-audit/` 下 15 个 `.md` 文件，是 skill 的真正"代码"——供宿主 LLM 代理执行的提示词协议。仅依赖宿主代理具备工具调用与并行子代理能力（`SKILL.md` Requirements）。
- **JavaScript（4 个 `.cjs`，约 3037 行）**：两个零依赖校验器 + 两份测试，只用 Node.js 内置模块 `node:fs`/`node:path`/`node:util`/`node:test`（`validate-coverage-ledger.cjs` 头部）。角色是"协议的机器执法者"：把 SKILL 文档里的人读规则变成可执行的 JSON 校验。
- **report-schema.json（461 行）**：JSON Schema，含自定义关键字 `visibleContent`（要求字段有可见正文，防纯空白/控制字符充数）。
- **无构建系统**：没有 `package.json`、`npm` 依赖、CI 配置或 `.github/` 目录（tarball 文件清单确认）；发行即 git 仓库本身。
- **安装通道**：`npx skills add https://github.com/cloudflare/security-audit-skill --skill security-audit`（`README.md`，经 skills.sh CLI，可 `--global` 装到用户级）。

## 4. 目录结构与代码组织

| 路径（skills/security-audit/ 下） | 职责 |
|---|---|
| `SKILL.md`（192 行） | 入口：frontmatter 触发描述、双模式、代理中立术语、执行安全、写隔离与工件提升、profile/预算、核心原则、六阶段总纲、反模式 |
| `RECONNAISSANCE.md`（156 行） | Phase 1：4 个并行侦察代理的提示词模板、prior-run 接入规则、架构摘要（≤1000 词）、覆盖账本规格 |
| `HUNTING.md`（251 行） | Phase 2：hunter 指派与排序、必配 hunter 提示词（9 部分）、核心狩猎方法、候选门槛、结构化返回契约、覆盖批评家波次 |
| `VALIDATION-AND-REPORTING.md`（186 行） | Phase 3-6：独立证伪、findings.json 写入、全新记录复核、三份报告产出规则 |
| `ATTACK-CLASSES.md`（130 行） | 常规攻击类（注入、访问控制、加密、业务逻辑、链式漏洞、Wildcard、显性问题等）+ 10 个领域路由指针 |
| 10 个领域伴生文件（73-106 行/个） | AI/LLM（含 MCP）、客户端/浏览器、云与部署、数据隔离与生命周期、桌面/移动/IPC、内存安全/二进制、协议/RPC/消息、资源耗尽、供应链/发布、Web 协议/认证；各含 Core discipline、攻击类小节、Universal moves、Validation rules |
| `report-schema.json` | findings.json 三种判定分支的 Schema（`oneOf`，全部 `additionalProperties: false`） |
| `validate-findings.cjs`（773 行） | findings.json 校验器：Schema 子集解释器 + 语义检查 |
| `validate-coverage-ledger.cjs`（872 行） | 覆盖账本校验器：单元状态机、canonical coverage_id、agent ID、证据归属 |
| `validate-*.test.cjs`（652 + 740 行） | 两份校验器的 node:test 测试，含对抗性输入用例 |

组织方式评述：单一 skill 包、文件平铺、无构建无依赖。文档与校验器严格成对——每个流程文档里定义的机器约束（状态表、ID 规则、路径规则）都能在对应 `.cjs` 中找到实现；hunter/verifier 提示词要求整段复制 SKILL.md 中的"工件提升程序"块（`HUNTING.md`/`VALIDATION-AND-REPORTING.md` 均携带同一 11 步 fenced block），保证跨文件指令逐字一致。

## 5. 核心架构与关键实现

### 5.1 六阶段多代理编排（`SKILL.md` + 三份流程文档）

父代理（Parent）独占所有共享运行文件（`run-metadata.json`、`architecture.md`、`coverage-ledger.json`、`findings.json`、三份报告）；每个 hunter/verifier 只能写自己唯一的 `scratch/`，保留产物 `artifacts/` 只允许受信父侧代码提升写入（Write isolation，`SKILL.md`）。术语层做了代理中立抽象：Parent / Task tool / `research` 与 `general` 代理对应 Claude Code 概念，但明确"用等价平台能力保持角色、写隔离与独立性边界"。三条硬性门禁贯穿全程：

- **预算门**：以"代理调用次数"为计数单位，wave 前必须预留下一轮批评家与验证器储备，预算不够时宁可把单元标记 `deferred` 并写明原因，也不静默缩减证据（`SKILL.md` Cost budget、`HUNTING.md`）。
- **执行门**：目标代码只能在 OS 强制沙箱内运行（无外网、空环境白名单、只读目标、scratch-only 写入、显式资源限额），任一控制缺失就降级为 `needs_validation`（`SKILL.md` Universal execution safety）。
- **证据门**：`confirmed` 必须有 entrypoint→propagation→sink 的有序 trace、有界本地观察结果；severity 不得超过已证实的 impact（`SKILL.md` 核心原则，校验器强制执行，见 5.3）。

### 5.2 确定性覆盖账本与 canonical coverage_id（`RECONNAISSANCE.md` + `validate-coverage-ledger.cjs`）

账本把覆盖声明变成数据：每个单元由 `surface × boundary × subsystem × attack_class (+lifecycle)` 四维 canonical_refs 唯一标识，ID 用 RFC 3986 大写百分号编码后以 `::` 连接（`RECONNAISSANCE.md`，实现在 `validate-coverage-ledger.cjs` 的 `encodeCanonicalRef`/`canonicalCoverageId`，354-376 行）。单元状态机为 `planned → in_progress → covered|candidate|blocked`（外加 `deferred`/`out_of_scope`），被批评家重开的单元按 wave 归档进 `attempts`，历史证据与新尝试严格隔离。校验器同时强制：agent ID 必须小写且不得是 Windows 保留设备名（`con`、`com1-9` 等，84 行）；本地证据必须位于属主代理的 `agents/<id>/artifacts/` 前缀下（`isOwnedArtifactPath`，394 行）；文件 5 MiB / 单元 1 万 / 错误输出 100 条等资源上限。

### 5.3 findings 契约与机器执法（`report-schema.json` + `validate-findings.cjs`）

Schema 用 `oneOf` 区分三种判定分支且全部禁用额外字段；校验器在其上追加语义检查（`collectFindingSemanticErrors`，485-583 行）：

```js
if (verdict === "confirmed") {
  for (const forbidden of ["claimed_root_cause", "blockers", "validation_plan", "reason"]) {
    if (hasOwn(finding, forbidden)) errors.push(...);
  }
  ...
  if (SEVERITY_RANK.get(overall) > SEVERITY_RANK.get(impact)) {
    errors.push(`${base}.severity.overall_severity: cannot exceed demonstrated impact ...`);
  }
}
```

其他执法点包括：fingerprint 全局唯一且按字典序排序；trace 必须首 entrypoint、末 sink、中间 propagation，单条 trace 只能是 entrypoint 或 sink；源文件路径必须是安全的仓库相对路径（拒绝绝对路径、`..`、反斜杠、Windows 保留名，`isSafeRelativeSourcePath`，473 行）；`needs_validation` 禁带 severity 且必须有可见的 local/deployment 验证计划。整个校验器自带一个受支持的 JSON Schema 关键字白名单解释器，输入层做 UTF-8 fatal 解码与控制字符防护，超出限额直接拒收。

### 5.4 独立验证与反污染机制（`VALIDATION-AND-REPORTING.md`）

独立性靠"换人 + 换信息"实现：Phase 3 的验证器只拿到候选、相关账本检查与 schema 分支，"绝不能拿到其他验证器的结论"，任务被表述为"试图证伪它"。Phase 5 对每条最终记录再派一个全新代理逐项核对源码路径、条件、严重度与修复方案；凡是会**提升判定等级**或实质改动 root cause/trace/impact 的替换记录，必须再交给一个未参与过任何环节的新验证器复核（材料替换规则），不可为时将争议记录移出 findings 并标记运行不完整。对抗性还体现在测试里：`validate-findings.test.cjs` 含终端控制字符载荷（BEL、NEL、Bidi 覆盖符等）与 `O_NOFOLLOW`/`O_NONBLOCK` 可用性探测（头部 14-35 行），针对"LLM 输出经终端渲染"和符号链接竞争两类攻击面。

## 6. 构建与运行

本仓库无可构建组件，运行时即宿主编码代理。

```bash
# 安装（README.md；skills.sh CLI，自动进入代理的 skill 目录）
npx skills add https://github.com/cloudflare/security-audit-skill --skill security-audit
npx skills add https://github.com/cloudflare/security-audit-skill --skill security-audit --global

# 使用：在目标代码库中启动代理后自然语言触发（README.md）
#   "security audit this codebase" / "find security vulnerabilities in ./src"

# 运行中由代理执行的两个校验命令（SKILL.md / VALIDATION-AND-REPORTING.md）
node <skill-dir>/validate-findings.cjs <output-dir>/findings.json
node <skill-dir>/validate-coverage-ledger.cjs <output-dir>/coverage-ledger.json
```

环境要求（`README.md` / `SKILL.md`）：支持工具调用与并行子代理的模型；Node.js；完整审计模式要求 OS 强制沙箱（无外网、空环境白名单、只读目标、资源限额）——沙箱能力不全时，工作流会把本可确认的发现保持为 `needs_validation` 而非执行目标代码。输出目录默认 `~/security-audit-skill/<repo-name>/run-<N>`。测试命令 README 未提供；测试基于 `node:test`（两个 test 文件头部 require `node:test`），可用 `node --test validate-findings.test.cjs` 方式运行（推断，未在文档中确认）。

## 7. 测试与工程质量

- 测试规模：两份测试共约 1392 行、约 226 处断言（`validate-findings.test.cjs` 652 行 / 99 断言，`validate-coverage-ledger.test.cjs` 740 行 / 127 断言），风格为 `node:test` + `node:assert/strict` 的表驱动用例，另含 spawnSync 驱动的 CLI 级测试（含 5s/15s 超时与"敌意输入"专用超时档）。
- 对抗性测试覆盖：终端控制字符输出消毒、非跟随打开（symlink 竞争）、UTF-8 边界、限额溢出（>5MiB、>64 层嵌套、重复 fingerprint、乱序排序）等（`validate-findings.test.cjs` 头部常量与 `validate-coverage-ledger.test.cjs`）。
- 无 CI 配置：tarball 无 `.github/`、无工作流文件，质量保障完全依赖本地测试与代码审查（未确认 GitHub 侧是否另有分支保护）。
- 代码质量观察：校验器风格紧凑严谨，错误信息可控（诊断字符串 256 字符截断、危险字符转义，`validate-findings.cjs` 54-55 行），资源限额集中定义于 `LIMITS` 常量，两份校验器共享同一套 Unicode/路径安全原语。

## 8. 社区与维护状态

- 活跃度：2026-06-18 创建，最后推送 2026-09-14（gh api），即快照前约 2 个月无新提交；末次提交主题为"Clarify guidance and full audit modes"，属于文档打磨而非功能迭代。
- 工程化程度：3 名贡献者、0 release、0 tag（gh api）——典型的"单一团队持续重写一份协议文档"模式，而非社区协作项目；126 个 open issues 说明用户量与反馈量大，但未观察到公开的 issue 分诊节奏（未确认）。
- 发布通道：仅 git 仓库 + skills.sh 安装，无版本号语义；README 通过 Cloudflare 官方博客《Build your own vulnerability harness》建立出处背书，并给出联系邮箱 security-ai-research@cloudflare.com。
- 增长客观陈述：创建约 5 个月达 71.7k 星（4777 fork）。可核实的相关因素：Cloudflare 官方背书与博客引流、2025 年底以来 agent skills 生态（Claude Code 等）的爆发、"AI 审计 + 可机读 findings"题材契合当前热点。仓库自身贡献者仅 3 人，星数增长与内容更新节奏（5 个月）并不匹配，属外部关注度驱动的爆发型仓库（此为基于元数据的客观描述，具体传播路径未确认）。

## 9. 亮点与风险

**亮点**

1. 协议即代码的工程化程度罕见：规则不只是写给 LLM 看的散文，`validate-coverage-ledger.cjs` / `validate-findings.cjs` 把状态机、ID 算法、路径安全、严重度上限全部变成机器执法，"prose 与 JSON 不得 disagree"是可验证的（`SKILL.md` 反模式 10）。
2. 对抗性独立验证设计完整：发现者≠验证者、候选证伪→记录复核→材料替换再审的三层换人机制，配合"验证器不得看到彼此结论"的信息隔离，直击 LLM 自审偏置（`VALIDATION-AND-REPORTING.md`）。
3. 覆盖可复算：确定性 coverage_id + 状态表 + attempts 归档，让"审完了"成为可审计的声明；多轮运行可加性（指纹接续、变更源重验）设计周到（`RECONNAISSANCE.md`）。
4. 安全自持性好：执行目标代码前的沙箱清单、11 步无符号链接竞争的工件提升程序、对 `needs_validation` 的强制降级纪律，skill 自身未复刻它所审计的漏洞模式。
5. 零依赖纪律：两个校验器只用 Node 内置模块，供应链面积极小。

**风险 / 局限**

1. 维护节奏存疑：最后推送 2026-09-14、无 release/tag/CI，协议文档的演进和 issue 处理是否持续未确认；71.7k 星带来的 issue 流（126 open）与 3 人贡献者的承载能力不对称。
2. 效果强依赖宿主模型与平台能力：并行子代理、沙箱、预算控制都假设宿主平台具备相应原语；README 自述单轮运行只覆盖重复运行总漏洞量的"约一半"，完整 standard/deep profile 的成本对个人用户偏高。
3. 上下文负担：15 份协议文档约 4100 行，hunter 提示词要求整段复制攻击类块与提升程序，对长上下文与指令遵循能力要求高；协议内部约束相互引用复杂，小模型容易丢失关键门禁。
4. 无自动化评测基准：仓库内没有测试目标集或跑分数据（tarball 文件清单确认），"多轮提高覆盖"等声明只有 README 数字，无第三方可复现实验（未确认）。

## 10. 总结评价

这个仓库本质上是一份用 Markdown 写成的"AI 审计操作系统规格书"，加两个把它变成可执行契约的零依赖校验器。它没有传统意义上的软件架构，但其内部设计密度很高：六阶段编排、确定性覆盖账本、三判定 Schema、多层独立复核、沙箱与提升安全，每一环都有文档-代码双重定义。5 个月 71.7k 星反映的是"Cloudflare 官方 + AI 安全审计 + agent skills 生态"三重风口的叠加，而仓库本身更像一份持续打磨的开放方法论，不是可长期依赖的工程制品。它适合三类人：想给自家编码代理装一个严肃审计流程的安全工程师（直接 `npx skills add` 即用）；研究多代理编排与 LLM 输出验证机制的工程者（`SKILL.md` + 两个 `.cjs` 是极佳的协议设计范本）；以及想理解"如何让 LLM 的安全结论可信、可机读、可复算"的 AI 安全从业者。不适合期待一键全自动扫描器的用户——它显式要求沙箱、预算纪律和人的参与。
