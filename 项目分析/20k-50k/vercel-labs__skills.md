# vercel-labs/skills 项目分析

> 星数 39,677（快照 2026-11-17，gh api 核实）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-17

## 1. 项目速览

一句话定位：开放 agent skills 生态的官方 CLI——`npx skills add <owner/repo>` 一条命令把可复用的 agent 技能（SKILL.md 指令包）安装进 79 种 AI 编码 agent 的对应目录。

| 事实 | 值 | 出处 |
|------|------|------|
| 名称 | vercel-labs/skills（npm 包名 `skills`，v1.9.0） | `package.json` |
| 星数 / fork | 39,677 / 3,493（2026-11-17 gh api） | gh api repos/vercel-labs/skills |
| 创建时间 | 2026-01-14，10 个月即 39.7k 星（超火箭增速） | gh api |
| 最近 push | 2026-11-16（v1.9.0 同日发布） | gh api |
| 语言 / 许可 | TypeScript / MIT（Copyright (c) 2026 Vercel, Inc.） | `LICENSE`、gh api |
| Node 要求 | >= 22.20（启用 `module.enableCompileCache`） | `package.json`、`bin/cli.mjs` |
| 发现站点 | skills.sh（Vercel 运营的 skills 目录/索引） | `README.md` |

## 2. 项目是什么，解决什么问题

Agent skills 是目前逐步收敛出事实标准的"给 AI 编码 agent 装说明书"的机制：一个技能是一个含 `SKILL.md`（frontmatter 声明 name/description，正文是操作指令）的文件夹，agent 在运行时按需加载。问题在于各家 agent 的技能目录互不相同——Claude Code 读 `.claude/skills/`，Codex/Cursor/Cline/Zed 等读 `.agents/skills/`，另有数十家各有各的路径（`docs/agents.md`）——技能作者与用户都被平台碎片化困扰。

skills CLI 解决三件事：其一，**统一入口**，`npx skills add vercel-labs/agent-skills` 交互式选择要装的技能与目标 agent，一次写进多个平台的目录（含项目级与全局级两种作用域，`-g` 切换）；其二，**统一分发**，技能本体仍留在各自 Git 仓库中（去中心化），skills.sh 只做索引与快照缓存，CLI 支持 `find/list/use/check/update/remove/init` 全生命周期（`README.md`、`docs/commands.md`）；其三，**标准化推动**，该项目与 agentskills.io 规范、`/.well-known/agent-skills/index.json` 发现协议（RFC 8615 风格）绑定，把 skills 从 Claude Code 专属特性推向跨 agent 的通用包格式（`src/providers/wellknown.ts`）。

值得注意的定位差异：与 anthropics/claude-plugins-official（Anthropic 官方对 Claude Code 插件的中心化策展市场）不同，本项目**不托管技能内容、不做审核门禁**，只做安装器 + 索引，任何 Git 仓库、任何符合 SKILL.md 约定的站点都能成为技能源。Vercel Labs 品牌与 79 平台适配使其事实上成为 skills 生态的"npm"层。

## 3. 技术栈与依赖

- 语言/运行时：TypeScript（ESM，`"type": "module"`），Node >= 22.20，pnpm 12（`package.json`）
- 运行时依赖极简，仅 3 个：`nostics`（结构化诊断错误体系）、`tar`（压缩包流式处理）、`yaml`（解析 SKILL.md frontmatter）（`package.json` dependencies）
- 关键 devDependencies：`@clack/prompts`（交互式终端 UI）、`@vercel/detect-agent`（检测当前运行于哪个 agent 内）、`obuild`（打包 dist）、`tsnapi`（公开 API 快照对比，防 breaking change）、`vitest`、`xdg-basedir`、`husky`+`lint-staged`+`prettier`、changesets（版本发布）（`package.json`）
- 构建：`pnpm build` = `node scripts/generate-licenses.ts && obuild`，产物 `dist/index.mjs` + `dist/cli.mjs`（`package.json` scripts、`build.config.mjs`）

## 4. 目录结构与代码组织

| 目录/文件 | 职责 | 出处 |
|-----------|------|------|
| `src/cli.ts` | 命令路由：find/init/add/install/use/remove/sync/list/check/update + 别名 | `src/cli.ts` |
| `src/api/` | 可编程 API 层：installSkills/listSkills/removeSkills/resolveSource/detectAgents，经 `src/index.ts` 再导出——CLI 同时是库 | `src/api/install.ts`、`src/index.ts` |
| `src/agents.ts` | 79 个 agent 适配器注册表（目录约定 + 安装探测） | `src/agents.ts` |
| `src/add.ts`（3713 行） | 核心 add 流程：源解析→发现→筛选→安装→锁文件→遥测 | `src/add.ts` |
| `src/blob.ts` | skills.sh 快照 fast-path 安装（免 clone） | `src/blob.ts` |
| `src/installer.ts` | 安装模式 copy/symlink/link、规范目录与跨平台符号链接 | `src/installer.ts` |
| `src/providers/` | HostProvider 注册表 + WellKnown 发现协议实现 | `src/providers/*.ts` |
| `src/skill-lock.ts` | 本地锁文件 `.skill-lock.json`（v3，含 folder hash） | `src/skill-lock.ts` |
| `src/sync.ts`（1107 行） | 实验性 npm 依赖技能同步（experimental_sync） | `src/sync.ts` |
| `skills/find-skills/` | 仓库自带的"元技能"：教 agent 帮用户找/装技能（自举 dogfood） | `skills/find-skills/SKILL.md` |
| `tests/` + `e2e/` | ~80 个集成/单元测试文件 + 9 个 e2e 规格 | 目录实测 |
| `.automations/`、`.github/workflows/agents.yml` | 用 agent 自动化维护本仓库（看板、PR 打标、triage） | 目录实测 |

组织方式：单包单体（非 monorepo），但通过 `src/api/` 划出清晰的库边界；源码共 219 文件、约 2.6 万行 TS。

## 5. 核心架构与关键实现

### 5.1 多 agent 适配层：79 个平台的目录约定表

`src/agents.ts` 以 `Record<AgentType, AgentConfig>` 声明全部 79 个 agent（`src/types.ts` 的 `AgentType` 联合类型逐个枚举，计数实测 79）。每个条目包含项目级目录、全局级目录、安装探测函数，并处理各家环境变量（`CLAUDE_CONFIG_DIR`、`CODEX_HOME` 等）。 notable 设计是"规范目录 + 重定向"：多数 agent 写入通用 `.agents/skills/`，但全局路径各不相同（如 Cursor 全局 `~/.cursor/skills`，Antigravity 全局 `~/.gemini/config/skills`），配置字段 `globalReadsCanonical`、`showInUniversalPrompt` 精确描述各家的读写行为差异：

```ts
antigravity: {
  name: 'antigravity',
  skillsDir: '.agents/skills',
  // agy, the IDE and Antigravity 2.0 read global skills from ~/.gemini/config/skills,
  // and none of them reads ~/.agents/skills outside a workspace.
  globalSkillsDir: join(home, '.gemini/config/skills'),
  globalReadsCanonical: false,
  ...
}
```

CI 用 `scripts/validate-agents.ts` 校验该表合法性，push 后 `sync-agents.ts` 自动再生成 `docs/agents.md` 的表格（`.github/workflows/agents.yml`）——文档与代码强一致。

### 5.2 分发机制：去中心化源 + skills.sh 快照缓存（blob fast-path）

安装源支持四类：`owner/repo` Git 引用（含 GitLab 子组、SSH/HTTPS URL）、本地路径、直链压缩包/SKILL.md、well-known 端点。没有中心 registry；skills.sh 是索引与缓存层。`src/blob.ts` 的 fast-path 三步走避免整仓 clone：

```
Flow:
  1. GitHub Trees API → discover SKILL.md locations
  2. raw.githubusercontent.com → fetch frontmatter to get skill names
  3. skills.sh/api/download → fetch full file contents from cached blob
```

端点可经 `SKILLS_API_URL`/`SKILLS_DOWNLOAD_URL` 环境变量覆盖，且 `BLOB_ALLOWED_REPOS` 允许仓库自托管快照（如 `zapier/connectors`），失败回退 Git clone（`src/blob.ts`、`src/find.ts`）。WellKnownProvider 实现 `/.well-known/agent-skills/index.json`（v0.2.0 schema，挂 agentskills.io）及 legacy `/.well-known/skills/index.json`，支持 zip 归档条目并有解包上限（50MB/1000 文件）与摘要校验（`src/providers/wellknown.ts`、`src/download-source.ts`）。

### 5.3 安装器：copy/symlink/link 三模式 + 锁文件完整性

`src/installer.ts` 定义三种安装模式：`copy` 直拷、`symlink`（规范目录放本体、各 agent 目录放符号链接，Windows 失败自动回退 copy）、`link`（链接到技能源目录，随源更新）。锁文件有两层：项目级 `.skill-lock.json` v3 记录 source/commit/GitHub tree SHA（`skillFolderHash`，任一文件变更即变，驱动 `check`/`update`）；README 详细规定了 `skills-lock.json` v2 的 `computedHash` 算法——按 UTF-8 路径字节序排序，path+NUL+长度+NUL+内容 单遍 SHA-256，文本文件 CRLF 归一为 LF，并给出测试向量（`README.md` Lock File 章节、`src/skill-hash.ts`）。第三方工具可复现同一哈希，这是把"技能完整性校验"标准化的关键一步。

### 5.4 安全与卫生设计

SKILL.md 本质是被 agent 执行的指令，属供应链攻击面。代码有多层防御：元数据清洗与终端转义剥离（`src/sanitize.ts`）、子路径穿越测试（`tests/subpath-traversal.test.ts`）、私有仓库安全策略（`tests/private-repo-add-security.test.ts`）、下载/解包字节与文件数上限（`src/download-source.ts`）、well-known 完整性测试（`tests/wellknown-integrity.test.ts`）。遥测上报 install/remove/update 事件（来源、技能名、目标 agent）至 skills.sh，可通过 `DISABLE_TELEMETRY` 或 `DO_NOT_TRACK` 关闭且 CI 环境自动静默（`src/telemetry.ts:76`）。

## 6. 构建与运行

```bash
pnpm install --frozen-lockfile   # 安装依赖（Node >= 22.20）
pnpm build                       # generate-licenses.ts + obuild → dist/
pnpm dev                         # node src/cli.ts 本地跑 CLI
pnpm test                        # vitest 单元/集成
pnpm test:e2e                    # vitest run --config e2e/vitest.config.ts
pnpm type-check                  # tsc --noEmit（含 e2e 工程）
pnpm format / format:check       # prettier

# 终端用户用法
npx skills add <owner/repo> [--skill <name>] [-a claude-code] [-g]
npx skills find [query] | list | use <src>@<skill> | check | update | remove | init
```

均出自 `package.json` scripts 与 `README.md`。

## 7. 测试与工程质量

- 测试规模：全仓 103 个 `*.test.ts`（src 内约 40 + `tests/` 约 80，含 fixtures），另有 `e2e/` 9 个端到端规格（api/blob/github/npm/rename/well-known 等，e2e 用真实 sandbox 环境跑构建产物，`e2e/sandbox.ts`）
- CI（`.github/workflows/ci.yml`）：矩阵 ubuntu+windows × Node 22.20/24/26，执行 build+test；单独 typecheck job；prettier 检查
- 专项 CI（`.github/workflows/agents.yml`）：`src/agents.ts` 一有改动即校验 agent 表并自动同步文档
- 工程质量观察：公开 API 用 tsnapi 快照对比防意外 breaking（`api:update`/`TSNAPI_ALLOW_BREAKING` 脚本，`package.json`）；changesets 管版本；husky+lint-staged 提交卫生；安全类测试（穿越、私有仓库、完整性）覆盖到位。整体是高成熟度工程。

## 8. 社区与维护状态

- 活跃度极高：创建于 2026-01-14，10 个月 39.7k 星、3,493 fork；最近 push 2026-11-16 且同日发布 v1.9.0（gh api）
- 发布节奏：30 个 GitHub releases，v1.9.0 为最新；publish.yml 走 npm-publish 环境与 id-token（可信发布）（`.github/workflows/publish.yml`）
- 贡献者：首页 100 个已满页，实际 >100（gh api contributors）
- 议题量：754 open issues——吞吐量大，仓库配有 agent-request/bug/feature 模板与 `.automations/` triage 自动化（`.github/ISSUE_TEMPLATE/`、`gh api`）
- 维护主体：Vercel, Inc.（LICENSE 版权行），Vercel Labs 产品矩阵成员（README 徽章）

## 9. 亮点与风险

**亮点**
1. 79 平台适配层 + `.agents/skills/` 通用约定，把 agent 技能从单平台特性升级为跨平台包生态，碎片化问题的目前最优解（`src/agents.ts`、`docs/agents.md`）
2. 去中心化分发 + skills.sh 快照缓存的双层架构：技能留在 Git 仓库、索引可替换（env 覆盖端点），避免单一中心化门禁（`src/blob.ts`、`src/find.ts`）
3. CLI 即库：`src/api/` 程序化接口经 index.ts 导出，供上游工具集成，API 兼容性有快照 CI 把关（`src/api/`、`package.json`）
4. 完整性标准化：锁文件 hash 算法公开到 README 并附测试向量，第三方可独立校验（`README.md`）
5. 深度 dogfood：自带 find-skills 技能、AGENTS.md、`.automations/` agent 自动化，仓库本身就是生态样本（`skills/find-skills/`、`AGENTS.md`）

**风险/局限**
1. 供应链面：安装的是可执行于 agent 上下文的指令文本，哈希与清洗只保证一致性与部分卫生，防不住 SKILL.md 内容本身的提示注入（机制固有，`src/sanitize.ts` 只是部分缓解）
2. 中心化引力：默认遥测与搜索都指向 skills.sh，虽可 env 覆盖/关闭，生态索引事实上由 Vercel 单点运营（`src/telemetry.ts`、`src/find.ts`）
3. Node >= 22.20 门槛偏高，限制老旧环境使用（`package.json` engines）
4. 754 open issues 高水位，社区支持响应依赖自动化分流，个人 issue 处理时效未确认

## 10. 总结评价

vercel-labs/skills 是 agent skills 生态事实上的包管理器：10 个月 39.7k 星的增速既有 Vercel Labs 品牌与 skills.sh 目录的推动，更源于它精准卡位了"79 个 agent 各有一套技能目录"的碎片化痛点，并以去中心化源 + 可校验锁文件 + 公开规范（agentskills.io）的组合做出中立基础设施的姿态。工程质量在同体量社区项目中属上游：3 个运行时依赖、103 个测试文件、双平台 CI、API 快照防破坏。与 claude-plugins-official 的中心化策展路线形成鲜明对照，两者共同勾勒出 skills 分发的两条路线。适合：想让技能一次触达多家 agent 的技能作者；多 agent 并用的工作流用户；研究 agent 生态标准化的工程团队。源码阅读优先级：`src/agents.ts`（适配层设计）→ `src/blob.ts`（分发架构）→ `src/installer.ts` + `src/skill-lock.ts`（完整性机制）。
