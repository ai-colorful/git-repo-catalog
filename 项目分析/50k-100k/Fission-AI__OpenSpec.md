# Fission-AI/OpenSpec 项目分析

> 星数 72840（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：面向 AI 编码助手的"规范驱动开发"（Spec-Driven Development）CLI——在写代码之前，先让人和 AI 在一份纯 Markdown 规范上达成一致。

| 事实 | 值 | 出处 |
|------|-----|------|
| 名称 | Fission-AI/OpenSpec（npm 包 `@fission-ai/openspec`） | `package.json` |
| 简介 | "Spec-driven development (SDD) for AI coding assistants." | gh api repos/Fission-AI/OpenSpec |
| 星数 | 快照 72,840；gh api 实时 73,468，fork 5,029，open issue 188 | gh api |
| 语言 | TypeScript（ESM，Node >= 22.12.0） | `package.json` |
| 许可证 | MIT | `LICENSE`、`package.json` |
| 版本 | v1.15.0（2026-10-24 发布） | `package.json`、gh api releases |
| 创建时间 | 2025-08-05 | gh api |
| 最近 push | 2026-10-30（分析前一日，极活跃） | gh api |
| 贡献者 | 150 人（gh api contributors 全量分页） | gh api |
| 维护者 | 核心维护者 2 人（@TabishB 领队、@clay-good）+ 1 个自动化账号 + 1 名顾问 | `MAINTAINERS.md` |

## 2. 项目是什么，解决什么问题

OpenSpec 解决的核心问题是：AI 编码助手能力很强，但当需求只存在于聊天记录里时，产出不可预测、不可复审、也无法沉淀。它在仓库根部引入一个轻量的 `openspec/` 目录层，把"要做什么"变成结构化的 Markdown 规范（含需求与场景），人和 AI 在任何代码被写出来之前先对齐（`README.md` "Why OpenSpec?" 一节）。

工作流由斜杠命令驱动：`/opsx:explore`（无风险的方案探讨）→ `/opsx:propose`（生成 proposal.md、specs/ 增量、design.md、tasks.md 四件套）→ `/opsx:apply`（按任务清单实现）→ `/opsx:archive`（归档并把增量合并回基线规范）。每个 change 一个文件夹，产物随时可改，没有刚性的阶段门（`README.md` "See it in action" 演示）。

规范就是纯 Markdown，没有专门 DSL：需求用 "The app SHALL ..." 句式，场景用 WHEN/THEN 结构（`README.md` 内嵌示例）。CLI 的 `validate` 命令会对这些文件做结构校验（需求编号、场景完整性、任务复选框等，见 `src/core/validation/validator.ts` 与 `src/core/validation/` 下各检查器），保证 AI 产出的规范不跑形。

它明确主打"棕地友好"（brownfield）：`openspec init` 会扫描项目里已存在的 AI 工具配置目录，为选中的工具生成对应的斜杠命令/技能文件；`openspec update` 再把指令刷到最新版（`src/core/available-tools.ts`、`src/core/update.ts`）。README 强调与 Spec Kit（GitHub 官方、流程较重）和 Kiro（AWS、锁定 IDE/模型）的差异定位：更轻、不锁定工具。

新特性 **Stores**（beta）把规划挪到独立仓库：同一个 `openspec/` 形状，用 git push 共享，让跨仓库功能、平台团队持有规范、先规划后写码成为可能（`README.md` "Why teams adopt OpenSpec"、实现见 `src/core/store/`）。

值得注意：仓库本身就用 OpenSpec 管理——`openspec/specs/` 下有约 20 个能力规范目录、`openspec/changes/` 有在途变更，README 明说 "OpenSpec is built with OpenSpec"（`openspec/` 目录，dogfooding）。

## 3. 技术栈与依赖

- **语言/运行时**：TypeScript（devDep `typescript ^6.0.3`），ESM（`"type": "module"`），要求 Node >= 22.12.0（`package.json` `engines`；`.changeset/drop-node-20.md` 显示刚放弃 Node 20）。
- **CLI 框架**：`commander ^14`；交互式问答用 `@inquirer/prompts`；输出美化 `chalk`、`ora`（`--json` 模式刻意不加载 ora 以提速，CHANGELOG 1.15.0）。
- **校验与解析**：`zod ^4`（schema 校验，含 artifact graph 的环检测）、`yaml ^2.8`；Markdown 结构解析为自研 parser（`src/core/parsers/`）。
- **文件/ glob**：`tinyglobby`、`picomatch`、`brace-expansion`（含自护防御：过深嵌套会被拒绝，CHANGELOG 1.14.3）。
- **构建**：无打包器，`build.js` 直接调 `tsc` 输出 `dist/`；另经 `scripts/binary/` 与 `.github/workflows/binaries.yml` 产出 6 平台（macOS/Linux glibc+musl/Windows × x64/arm64）单文件二进制，无需 Node 即可安装（CHANGELOG 1.15.0）。
- **测试**：`vitest ^4` + `@vitest/ui`；lint 用 `eslint ^10` + `typescript-eslint`。
- **发布**：pnpm 10.34.5 workspace + Changesets（`@changesets/cli`），release 走 `changeset publish`（`package.json` scripts）。
- **文档站**：`website/` 为独立 pnpm workspace 的 Next.js 静态导出站（openspec.dev），预览用零依赖自写服务器 `scripts/preview.mjs`，部署到 Cloudflare（`website/cloudflare/`、CHANGELOG 1.14.2）。
- **Nix 打包**：`flake.nix` + CI 里的 nix 校验 job（`.github/workflows/ci.yml`、`flake.lock`）。

依赖面刻意收得很窄：运行时依赖仅 11 个（`package.json` dependencies），全部是小型成熟库。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|------|------|
| `src/cli/` | commander 入口（`index.ts` 908 行）与薄命令层（`commands/`） |
| `src/commands/` | 各命令实现：change、spec、validate、show、status、config、doctor、store、workset、completion、schema、context、feedback 等 |
| `src/core/` | 业务核心，约 46 个模块：artifact-graph、validation、parsers、command-generation、store、templates、profiles、completions、telemetry 等 |
| `skills/` | 12 个面向 AI 助手的 SKILL.md（propose、apply、archive、explore、verify 等），是 agent 侧工作流的真正逻辑载体 |
| `schemas/spec-driven/` | 默认 artifact schema（`schema.yaml`）+ proposal/spec/design/tasks 四个模板 |
| `test/` | 241 个 TS 文件，234 个 `*.test.ts`，含 `cli-e2e/` 与 docs-claims 类测试 |
| `openspec/` | 仓库自身 dogfooding 的规范与在途变更 |
| `website/` | Next.js 文档站（openspec.dev），独立 workspace |
| `docs/`、`docs-lab/` | 文档内容与实验性文档 |
| `scripts/` | 构建/发布辅助：binary 安装脚本、generate-skillssh.mjs、parity 哈希再生成等 |
| `.github/workflows/` | ci.yml、binaries.yml、release-prepare.yml、security.yml |

组织方式：单包（pnpm workspace 内主包 + website 子包），src 内按 "cli 薄层 / commands 中层 / core 厚核" 分层，`src/cli/index.ts` 只注册命令，实现集中在 `src/commands/` 与 `src/core/`。无 monorepo 多包复杂度，但用 `pnpm-workspace.yaml` 隔离 website。

## 5. 核心架构与关键实现

### 5.1 工具适配层：一份注册表适配 51 个 AI 工具

`src/core/config.ts` 中的 `AI_TOOLS` 数组是全项目最关键的数据结构：51 个条目（含 "Other / Universal" 共享 `.agents` 的兜底项），每个工具声明 `skillsDir`、可选 `detectionPaths`、`globalSkillsDir`、`legacySkillsDirs` 等。`src/core/available-tools.ts` 据此扫描项目目录判断哪些工具在场：

```ts
// src/core/available-tools.ts（节选）
const available = AI_TOOLS.filter((tool) => {
  if (!toolSupportsSkills(tool)) return false;
  if (tool.globalSkillsDir) { ... }
  if (!tool.skillsDir) return false;
  if (tool.detectionPaths && tool.detectionPaths.length > 0) {
    return tool.detectionPaths.some((p) => view.exists(path.join(projectPath, p)));
  }
  return view.isDirectory(path.join(projectPath, tool.skillsDir));
});
```

`src/core/command-generation/` 再按工具家族生成不同格式（YAML、TOML、Markdown frontmatter，`adapters/`、`yaml.ts`、`toml.ts`），Cursor 的 `/opsx-propose`、Codex 的 `$openspec-propose` 等命令拼写差异都在这层消化（`README.md` 说明 + `src/core/command-generation/`）。

### 5.2 Artifact Graph：schema 即数据的工作流引擎

默认工作流 "spec-driven" 完全由 YAML 声明（`schemas/spec-driven/schema.yaml`）：每个 artifact 有 `id`、`generates`（产出文件 glob）、`template`、`instruction`（喂给 AI 的完整指令文本）、`requires`（依赖）。proposal → specs → design → tasks 构成 DAG。加载端用 Zod 校验并做三重检查——重复 ID、悬空引用、环检测（`src/core/artifact-graph/schema.ts` 的 `parseSchema`，调用 `validateNoDuplicateIds`/`validateRequiresReferences`/`validateApplyReferences`/`validateNoCycles`）。社区可通过独立仓库发布自定义 schema 包（`README.md` "Community schemas"，对标 spec-kit 的 extensions 目录）。

### 5.3 Validator：对 AI 产出的规范做结构把关

`src/core/validation/validator.ts` 是最大的核心文件之一，依赖 `src/core/parsers/` 下自研 Markdown 解析器族（`requirement-blocks.ts`、`requirement-text.ts`、`spec-structure.ts`、`change-parser.ts`），检查内容包括：SHALL/MUST 句式、场景 WHEN/THEN 覆盖、任务编号与复选框连续性、delta 规范的 ADDED/MODIFIED/REMOVED 语义、路径包含性（`assertDiscoveredSpecFile` 防符号链接逃逸，CHANGELOG 1.14.3 相关修复可佐证其严格程度）。支持 strict 模式。

### 5.4 Store：跨仓库规范共享（beta）

`src/core/store/` 实现独立于项目的规范仓库：`foundation.ts` 定义存储布局常量（`.openspec-store/store.yaml` 元数据、全局数据目录下的 `stores/` 与 `registry.yaml` 注册表）、原子写入与文件锁（复用 `src/core/file-state.ts`）；`git.ts` 处理 git 后端，`operations.ts`/`registry.ts` 提供注册与读写。所有读取规范/变更的命令都接受 `--store <id>` 标志（`skills/openspec-propose/SKILL.md` 中对粘性行为有详细约定）。

### 5.5 Skills 生成与一致性

`skills/` 下 12 个 SKILL.md 是 agent 侧的真实工作流（含 "planning boundary" 等防越权指令），`scripts/generate-skillssh.mjs` 生成面向各工具的技能文件，`scripts/regen-parity-hashes.mjs` 维护生成物与源的一致性哈希——保证 CLI 更新后各工具目录里的指令不漂移（`package.json` scripts + `src/core/shared-skill-target.ts` 的 reconcile 逻辑）。

## 6. 构建与运行

```bash
# 开发（来自 package.json scripts）
pnpm install
pnpm dev          # tsc --watch
pnpm build        # node build.js（直接 tsc，产物 dist/）
pnpm dev:cli      # pnpm build && node bin/openspec.js

# 测试与质量
pnpm test         # vitest run
pnpm test:watch / pnpm test:ui / pnpm test:coverage
pnpm lint         # eslint src/

# 发布
pnpm changeset && pnpm release   # check:pack-version + changeset publish
pnpm generate:skills             # 重新生成各工具技能文件

# 使用（README Quick Start）
npm install -g @fission-ai/openspec@latest   # 或 curl 安装脚本 / brew install openspec
openspec init      # 在目标项目内初始化 openspec/ 并生成 AI 工具指令
openspec update    # 刷新 agent 指令；openspec config profile 选择工作流配置
```

## 7. 测试与工程质量

- **规模**：`test/` 下 241 个 TS 文件、234 个 `*.test.ts`，而 `src/` 仅 216 个 TS 文件——测试文件数超过源文件数（1.08:1），在 CLI 项目里罕见地充分。
- **层次**：单元测试（`test/core/`、`test/commands/` 等）、CLI e2e（`test/cli-e2e/`，含 `capstone-journeys.test.ts` 全流程旅程、`store-lifecycle.test.ts`）、以及特色鲜明的 **docs-claims 测试**（`test/apply-docs-claims.test.ts`、`test/setup-docs-claims.test.ts`、`test/skills-docs-claims.test.ts` 等）——把文档里的命令与断言逐条对回实际行为，防止文档漂移。另有 `test/vocabulary-sweep.test.ts` 做术语一致性扫描。
- **CI**（`.github/workflows/ci.yml`）：路径过滤（dorny/paths-filter）+ 5 矩阵（linux/macOS/windows @ Node 22.12.0，linux 额外 @ Node 22/24 LTS）、merge group、concurrency 取消、所有 action 按 SHA 固定、`persist-credentials: false`、15 分钟超时。Nix 变更触发单独的 nix 打包校验 job。
- **其他**：binaries.yml 出 6 平台二进制并附 SHA256SUMS；security.yml、CODEOWNERS、三种 issue 模板、SECURITY.md、devcontainer、.coderabbit.yaml——工程化配置齐全。
- 代码质量观察：运行时依赖仅 11 个且持续做依赖瘦身（CHANGELOG 记录移除 `serve`、拒绝未修补 `braces`）；对 TTY 边界（零宽度终端死循环）、glob 爆栈等真实 bug 有细致的回归修复记录。

## 8. 社区与维护状态

- **活跃度**：gh api 显示最近 push 2026-10-30；2026-09-30 至 2026-10-24 连发 v1.14.0/v1.14.1/v1.14.2/v1.14.3/v1.15.0 五个 release，约每周一版（gh api releases）。
- **贡献者结构**：150 名贡献者（gh api 全量分页）；但 `MAINTAINERS.md` 显示核心维护者仅 2 人 + 1 个自动化账号（@alfred-openspec）+ 1 名技术方向顾问。CHANGELOG 中 PR 作者以 @clay-good、@surlymochan、@TabishB 为主，少量社区 PR（如 @CRYPTONIKAV）。
- **流程**：`CONTRIBUTING.md` 规定"一切始于 issue"——先讨论后实现；release 由 Changesets 驱动，配套维护者自己的 release skill（`.agents/skills/release-openspec/`）。
- **项目年龄**：2025-08-05 创建，约 15 个月从 0 到 73k 星（gh api），增长极快；社区渠道为 Discord（README 徽章）。README 内嵌的 star/down/contributor 徽章及 "The most loved spec framework" 折叠块显示团队重视社区运营。
- 未确认：具体的月 npm 下载量（徽章存在但静态阅读无法取值）。

## 9. 亮点与风险

**亮点**

1. 工具中立架构：51 个 AI 工具收敛为一份声明式注册表 + 生成器适配层（`src/core/config.ts`、`src/core/command-generation/`），新工具接入成本极低，这是它能吃下"AI 助手碎片化"红利的核心设计。
2. 工作流即数据：默认流程由 `schemas/spec-driven/schema.yaml` 声明（含 Zod 校验与环检测），社区可发布自定义 schema，扩展不需要改代码。
3. 测试纪律极强：234 个测试文件超过源文件数，docs-claims 测试把文档当可执行契约，五矩阵 CI + SHA 固定 action，工程化水准高于同体量多数项目。
4. Dogfooding 闭环：仓库自身的 `openspec/` 就是产品最佳实践样本（约 20 个能力规范 + 在途变更），README 直指用户去看"活的例子"。
5. 交付形态完整：npm + Homebrew + Nix + 免 Node 单文件二进制（6 平台、SHA256SUMS），安装路径覆盖极广。

**风险/局限**

1. 总线因子：2 名核心维护者扛着 73k 星、周更节奏与 188 个 open issue（gh api），长期可持续性存疑。
2. 51 个工具集成的维护面很宽：各 AI 工具的技能/命令规范仍在快速演化，`legacySkillsDirs`、detection 兼容逻辑会持续累积复杂度。
3. Node >= 22.12.0 的硬门槛（`package.json`）+ 刚放弃 Node 20，对老环境不够友好（二进制安装是缓解）。
4. 流程开销：对小型变更，proposal/specs/design/tasks 四件套偏重；规范与代码的一致性依赖纪律与 `/opsx:verify`，Store 等新特性仍在 beta。

## 10. 总结评价

OpenSpec 是"规范驱动开发"赛道目前工具中立性最好、工程化最扎实的一个实现：用一份 YAML 注册表和生成器层消化了 51 个 AI 编码助手的接入差异，用纯 Markdown + 结构校验器让规范既轻又能被机器把关，再用 docs-claims 测试和 dogfooding 保证自己说的就是自己做的。15 个月冲到 73k 星虽有品类红利（AI 辅助开发的流程焦虑）加持，但代码质量、测试纪律与发布节奏完全撑得起热度。适合：团队级 AI 辅助开发流程标准化（尤其多仓库/平台团队规范共享，Stores 方向）、想要轻量替代 Spec Kit/Kiro 的开发者；也极适合研究"如何为异构 AI agent 生态做适配层"的人读源码——`src/core/config.ts` + `src/core/command-generation/` + `schemas/spec-driven/schema.yaml` 三处连读即可理解其全部设计精髓。
