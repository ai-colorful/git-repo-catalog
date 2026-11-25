# lerna/lerna 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 JavaScript（现仓库主体已为 TypeScript，gh api 语言标记 TypeScript）｜档位 20k-50k｜分析日期 2026-11-17

## 1. 项目速览

一句话定位：JavaScript/TypeScript monorepo 工具鼻祖，专注多包工作区的**版本管理与 npm 发布**编排，v6+ 起将任务执行委托给 Nx 构建系统。

| 事实 | 值 | 出处 |
|---|---|---|
| 名称 | lerna/lerna | gh api |
| 星数 | 39500（快照 2026-10-24）；gh api 实时 36030 | 任务参数 / gh api |
| 语言 | TypeScript（ESM，`"type": "module"`） | `packages/lerna/package.json` |
| 许可证 | MIT | `LICENSE`、gh api |
| 当前版本 | 10.1.2（2026-11-06 发布） | `packages/lerna/package.json`、CHANGELOG |
| 最近 push | 2026-11-13 | gh api |
| 创建时间 | 2015-12-04，作者 Sebastian McKenzie（Babel 作者） | gh api、`packages/lerna/package.json` author 字段 |
| 维护方 | Nx（Nrwl）团队自 2022 年接管 | `README.md`、`website/docs/lerna-and-nx.md` |

## 2. 项目是什么，解决什么问题

Lerna 是 JavaScript 生态最早的 monorepo 管理工具（2015 年由 Sebastian McKenzie 创建，曾服务于 Babel、React、Jest 等多包仓库，见 `website/docs/introduction.md`："Lerna is the original monorepo tool for JavaScript/TypeScript... used by tens of thousands of projects, including React and Jest"）。它解决两个核心问题：一是跨多包高效执行命令（按依赖关系排序、并行、可分发到多机）；二是管理发包流程——从版本号变更到 npm 发布的全链路（同上 `introduction.md`）。

2022 年 Lerna 原维护者淡出、项目濒临失修，Nx 公司 Nrwl 接管 stewardship（`README.md` 顶部声明及 issue #3121 链接，`website/docs/lerna-and-nx.md`）。接管后演进路径清晰：v5 清理更新依赖，v6 起把任务调度委托给 Nx 的任务运行器（缓存、并行、分布式执行直接复用 Nx 能力）；v7 起默认移除 `bootstrap/add/link` 包管理命令（依赖安装交还给 npm/yarn/pnpm/bun 原生 workspaces），v9 彻底删除这些命令只留提示壳（`packages/lerna/src/index.ts` 中 `explicitlyHandleLegacyPackageManagementCommands`，注释明言"removed by default in v7... removed in v9"）。

当前定位（HEAD 快照）：Lerna = "包发现 + 依赖图 + 版本/发布编排"层，运行在某个 workspace 包管理器（npm/yarn/pnpm/bun，见 `website/docs/getting-started.md` 前置要求）之上，任务执行层复用 Nx（缓存、DTE、项目图）。与 pnpm workspace 互补（lerna 不管安装），与 Turborepo 相比官网文档未直接对比，但功能重叠在任务编排层、Lerna 独有的 value 在 version/publish。

## 3. 技术栈与依赖

- 语言/运行时：TypeScript 全量编译为 ESM；Node `^22.22.2 || ^24.15.0 || ^26.0.0`（`packages/lerna/package.json` engines）
- 构建体系：Nx 23.2.1 自举（根 `package.json`："lerna dog-fooding lerna"），esbuild 打包，oxlint/oxfmt，vitest 5（根 `package.json` devDependencies）
- CLI 框架：yargs 18（`lernaCLI()` 见 `libs/core/src/lib/cli.ts`）
- npm 交互栈（`packages/lerna/package.json` dependencies）：`@npmcli/arborist`（依赖树）、`libnpmpublish`/`libnpmaccess`/`npm-registry-fetch`/`make-fetch-happen`/`pacote`/`npm-packlist`（打包与发布）、`ssri`
- 版本/变更日志：`conventional-changelog` 套件（conventional-commits-parser、conventional-recommended-bump、conventional-changelog-angular）、`semver`
- 配置：`cosmiconfig`（lerna.json 发现）；GitHub 集成：`@octokit/rest`
- 并发/进程：`p-queue`、`p-map`、`execa`；交互：`inquirer`
- 对 Nx 的运行时依赖：`nx`、`@nx/devkit` peer 范围 `>=23.1.0 < 24.0.0`（`packages/lerna/package.json`）

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `packages/lerna/` | 唯一发布包：CLI 入口 `src/cli.ts`、命令注册 `src/index.ts`、`schemas/lerna-schema.json`、`migrations.json` |
| `libs/core/` | 核心库 `@lerna/core`：Command 基类、Project、项目图、拓扑排序、变更检测、conventional-commits、npm-conf、cycles |
| `libs/commands/` | 12 个命令源码：changed/clean/create/diff/exec/import/info/init/list/publish/run/version |
| `libs/child-process/`、`libs/test-helpers/`、`libs/e2e-utils/` | 基础设施 |
| `libs/nx-plugin/` | Nx 生成器（e2e-project） |
| `e2e/` | 端到端测试（按命令分目录，基于本地 Verdaccio registry） |
| `integration/` | 集成测试 |
| `website/` | Docusaurus 文档站（lerna.js.org） |
| `tools/` | 构建/发布脚本（`build.sh`、`lerna-release.mjs`） |

组织方式：典型 Nx monorepo（`nx.json` + `libs/` 分层 + `packages/` 发布物）。根 `package.json` workspaces 仅 3 项（packages/lerna、website、libs/e2e-utils），其余 libs 通过 Nx/TS path 解析。命令实现收敛在 libs/commands（纯逻辑）+ packages/lerna/src/commands（CLI 包装，含 3 个 Nx 系命令 add-caching/repair/watch），共注册 15 个命令（`packages/lerna/src/index.ts`）。

## 5. 核心架构与关键实现

### 5.1 命令体系与 Command 基类

15 个命令统一经 yargs 注册（`packages/lerna/src/index.ts` main()）。所有命令继承 `Command` 基类（`libs/core/src/lib/command/index.ts`）：构造时 pause 日志、解析 Project/项目图（`detect-projects.ts`），生命周期为 initialize() → execute()，默认并发度为 CPU 核数（`const DEFAULT_CONCURRENCY = os.cpus().length`）。命令分为三类：只读查询（list/changed/info/diff）、工作区操作（init/create/clean/import/exec/run）、版本发布（version/publish，二者可组合：publish 内部可先跑 version）。旧三件套 bootstrap/add/link 在 v9 后是纯提示壳（同文件 `explicitlyHandleLegacyPackageManagementCommands`）。

### 5.2 包发现与依赖图：从自研 PackageGraph 到 Nx 项目图

历史版本的 `PackageGraph` 已不存在（全库 `find *package-graph*` 无结果）；现实现是 `ProjectGraphWithPackages`（`libs/core/src/lib/project-graph-with-packages.ts`），由 `create-project-graph-with-packages.ts`（`libs/core/src/lib/command/`）把 Nx 的项目图（含 `daemonClient` 守护进程，见 `command/index.ts` import）与各包 package.json 合成为带 package 节点的图，暴露 `localPackageDependencies`。执行排序在 `toposort-projects.ts`：Kahn 分批算法 + 环处理——先 `getCycles` 找环、`mergeOverlappingCycles` 合并重叠环，无外部依赖的环作为一批同时执行，`rejectCycles` 可强制失败；`run-projects-topologically.ts` 为异步版，用 PQueue 按拓扑序最大化并发并动态解锁后续节点。关键片段（`toposort-projects.ts`）：

```ts
let batch = Object.keys(dependenciesBySource)
  .filter((p) => dependenciesBySource[p].size === 0)
  .filter((p) => !seen.has(p));
if (batch.length === 0) {
  const cycle = Array.from(cycles.values()).find((cycle) => {
    // only process the cycle if it has NO nodes with dependencies outside this same cycle
    return !cycle.some((project) => {
      const projectDeps = dependenciesBySource[project];
      return projectDeps && Array.from(projectDeps).filter((d) => cycle.indexOf(d) === -1).length > 0;
    });
  });
  if (cycle) { cycles.delete(cycle); batch = cycle.filter((p) => projectsMap.has(p)); }
}
```

### 5.3 变更检测：git diff 基线

`collect-updates/make-diff-predicate.ts`（`libs/core/src/lib/collect-updates/`）用 `git diff --name-only <committish> -- <包目录相对路径>` 判断包是否变更，再对结果文件列表应用 `ignorePatterns`（lerna.json `ignoreChanges`，minimatch 过滤）；`collect-project-updates.ts` 汇总后还会沿依赖图传播变更（下游包连带更新）。自举配置示例（根 `lerna.json`）：`"ignoreChanges": ["**/__fixtures__/**", "**/__tests__/**", "**/*.md"]`。

### 5.4 version 命令：fixed/independent 双模式与发布流水线

`libs/commands/version/src/index.ts`（964 行）。模式由 `this.project.isIndependent()` 决定（`libs/core/src/lib/project/index.ts:201`，读取 lerna.json `version: "independent"`）。`getVersionsForUpdates()` 按 5 种策略选版本：显式 `--bump <version>`（全局版本）、`--bump <semver type>`（fixed 用 lerna.json 的 version 为基、independent 逐包）、`--conventional-commits`（`recommend-version.ts` 按 Angular 规范从提交历史推荐 bump 类型）、交互提示（fixed 只问一次、independent 逐包问），prerelease id 解析为 `--preid || 已有 preid || "alpha"`。fixed 模式维护单一 `globalVersion` 同步所有包；exact/范围依赖同步、跨包依赖版本号回写、lockfile 更新（`update-lockfile-version.ts` 支持 npm/pnpm/yarn，另有 `update-bun-lockfile.ts`）。Git 流水线在 `src/lib/`：`is-anything-committed`（要求先提交）、`is-behind-upstream`（防落后远端）、`git-add`/`git-commit`/`git-tag`/`git-push`、`create-release.ts` 用 @octokit/rest 创建 GitHub Release（`--create-release=github`）。主流程（index.ts L293-321）：collectProjectUpdates → 过滤 private/无版本包 → checkWorkingTree（未提交检查）→ 计算版本 → 确认 → 回写。

### 5.5 publish 命令与 Nx 融合

`libs/commands/publish/src/index.ts`（1264 行）：`pack-directory.ts`（npm-packlist + tar）打 tar 包，`npm-publish.ts` 封装 libnpmpublish；OTP 处理带缓存（`otpCache`，`get-two-factor-auth-required.ts` 预检 registry 是否要求 2FA）、`publishConfig.tag` 作为默认 dist-tag、npm provenance 签名通过拦截 libnpmpublish 内部数据实现（index.ts L967 注释"hackily intercept"）。Nx 融合面：`lerna run` 默认走 `runScriptsUsingNx()`（`libs/commands/run/src/index.ts` L133，`useNx: false` 可退回遗留 runner 但多脚本并发已不支持），`add-caching` 命令交互式写入 nx.json 的 cacheableOperations/targetDefaults（`packages/lerna/src/commands/add-caching/index.ts`），`repair`/`watch` 为 Nx 配套命令；官方说明见 `website/docs/lerna-and-nx.md`（"Lerna uses Nx to detect packages... defers to Nx's powerful task runner... can also be connected to Nx Cloud for free"）。

## 6. 构建与运行

来自根 `package.json` scripts 与 `AGENTS.md`（Node 版本以 `volta.node`=26.12.0 为准）：

```bash
npm ci                                  # 安装依赖
npm run build                           # tools/scripts/build.sh，Nx 打包 + 重链 bin
npm test                                # nx run-many -t test（vitest 单测）
npm run lint                            # nx run-many -t lint（oxlint）
npm run integration                     # 集成测试（maxWorkers=2）
npx nx e2e <project>                    # e2e：本地 Verdaccio registry 发包后跑 CLI（需 pnpm>=8.10.2）
npm run lerna-release                   # 用 lerna 自己的 version/publish 工厂函数发版（tools/scripts/lerna-release.mjs）
```

注意：构建前 `npx lerna` 不可用（bin 指向 `packages/lerna/dist/cli.js`，需先 build，见 `AGENTS.md`）。

## 7. 测试与工程质量

- 规模：单元测试 220 个 `.spec.ts`（约 3.9 万行，占全部 TS 约 45k 行的大头，即源码:测试 ≈ 1:6.5）；e2e 94 个文件（按 12 命令分目录）；integration 28 个文件（实测统计，基于 HEAD 快照）
- 风格：vitest + 大量快照（`__snapshots__/`）；e2e 通过本地 Verdaccio 全真链路（build → publish 到本地 registry → 安装执行）；`libs/e2e-utils` 与 `libs/nx-plugin` 的 e2e-project 生成器支撑
- CI（`.github/workflows/ci.yml`）：`main` + `windows-main`/`windows-agents`（Windows 跨平台覆盖）+ `esm-compat-node-floor`（ESM 兼容性下限检查）；另有 `nx-migrate.yml`、`pkg-pr-new.yml`（PR 即时预览包）、`release.yml`
- 质量观察：TS 严格类型 + ESLint/oxlint 双轨；`AGENTS.md`/`CLAUDE.md` 记录了细致的 AI 协作陷阱（如 Node patch 版本差异导致 snapshot 颜色输出失败、e2e-repair 快照随 Nx migration 漂移），反映 CI 对快照稳定性要求高

## 8. 社区与维护状态

- 活跃度：最近 push 2026-11-13；open issues 302、forks 2266（gh api）
- Release 节奏（gh api releases）：v10.1.2（2026-11-06）、v10.1.1（2026-10-12）、v10.1.0（2026-10-09）、v10.0.1（2026-08-19）、v10.0.0（2026-07-29）、v9.0.0（2025-09-23）——大版本约一年一跳，小版本月级发布，节奏健康；CHANGELOG 显示大量 renovate 自动化依赖升级 PR
- 贡献者：contributors 分页到 344 页（per_page=1，即约 344 名贡献者）；核心团队为 Nx 四人（vsavkin/JamesHenry/bcabanes/juristr，`README.md` 表格），历史贡献者含 Babel 系（sebmck）、evocateur（Daniel Stockman，原长期维护者）、henryzhu（Babel）等
- 商业支持：Nx Cloud（DTE/远程缓存增值服务，`website/docs/lerna-and-nx.md`）

## 9. 亮点与风险

**亮点**
1. 版本/发布编排仍是 JS 生态最完整的实现：fixed/independent 双模式、conventional-commits 自动 bump、GitHub Release 自动化、OTP/provenance/dist-tag 细节俱全（`libs/commands/version`、`libs/commands/publish`）。
2. Nx 融合是干净的分层：包发现/依赖图复用 Nx 项目图（含 daemon），任务执行复用 Nx runner，获得缓存与分布式执行而无需用户迁到完整 Nx 工作区（`libs/core/src/lib/command/create-project-graph-with-packages.ts`、`website/docs/lerna-and-nx.md`）。
3. 极高测试投入与跨平台 CI（1:6.5 的源测比、Windows 专用 job、ESM 兼容下限检查、Verdaccio 全真 e2e）。
4. 彻底的自举（dog-fooding）：用自己的命令工厂函数写发版脚本（`tools/scripts/lerna-release.mjs` import `lerna/commands/publish`），自举 lerna.json 就是文档级示例（根 `lerna.json`）。
5. 迁移善后处理规范：被删命令留清晰提示壳而非静默失败（`packages/lerna/src/index.ts`），并有 `migrations.json` + `nx-migrations` 机制（`packages/lerna/package.json`）。

**风险/局限**
1. 与原生 Nx 功能重叠明显：任务缓存/DTE/项目图/repair/watch/add-caching 全部来自 Nx，重度用户可能直接采用 Nx，Lerna 的长期差异点收窄为 version/publish 编排（对照 `website/docs/introduction.md` 与 `lerna-and-nx.md`）。
2. 依赖升级激进：npm 栈（libnpmpublish/arborist/npm-registry-fetch）、inquirer、execa 等全部紧跟主版本，CHANGELOG 中大量 breaking 依赖升级修复，插件/二次开发生态兼容成本高。
3. 对 Nx 版本窗口的强约束（`nx >=23.1.0 <24.0.0`）造成升级耦合；`AGENTS.md` 自述 e2e-repair 快照随 Nx migration 发布而漂移，是结构性维护负担。
4. 历史包袱仍在：v7/v9 连续删命令、大版本跳跃（v8→v9→v10 仅一年），老项目升级需仔细核对 obsolete options（`website/docs/lerna6-obsolete-options.md`、`legacy-package-management.md`）。

## 10. 总结评价

Lerna 是 JS monorepo 运动的开创者，在 Nx 团队接管后完成了一次成功的"瘦身重生"：放弃包管理与任务执行的底层重造，退守为 workspace 包管理器（npm/pnpm/yarn/bun）与 Nx 构建系统之间的**版本管理与发布编排层**——这正是它至今不可替代的部分。测试与工程化水准在同类 CLI 中属第一梯队。适合的场景是：需要在一个仓库里管理多个要发 npm 的包、希望 fixed/independent 版本策略和 conventional-changelog 开箱即用、同时想免费获得 Nx 缓存与 CI 分发能力的团队；不适合纯内部应用 monorepo（无发包需求，用 Nx 或 Turborepo 更直接）。想读源码的人推荐按 `packages/lerna/src/index.ts` → `libs/core/src/lib/command/` → `libs/commands/version` → `libs/commands/publish` 的顺序阅读，这套"命令基类 + 项目图 + 拓扑执行 + 变更检测"的分层是理解所有 JS monorepo 工具的最佳教材。
