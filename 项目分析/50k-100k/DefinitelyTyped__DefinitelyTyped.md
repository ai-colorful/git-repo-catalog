# DefinitelyTyped/DefinitelyTyped 项目分析

> 星数 51448（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：TypeScript 官方生态的集中式类型定义仓库，npm 上 `@types/*` 作用域下八千多个包的唯一源头。

| 事实 | 值 | 出处 |
| --- | --- | --- |
| 名称 | DefinitelyTyped/DefinitelyTyped | gh api 元数据 |
| 星数 | 51465（分析日实测，快照 51448） | gh api |
| Fork 数 | 30326 | gh api |
| 主语言 | TypeScript（内容几乎全为 .d.ts 声明文件） | 仓库内容 |
| 许可证 | MIT（LICENSE 文件为自定义文本，GitHub 识别为 NOASSERTION） | `LICENSE` |
| 默认分支 | master | gh api |
| 最近推送 | 2026-11-02（分析前 2 天，极高活跃度） | gh api |
| 类型包数量 | 8582 个目录（types/ 下），另有 2311 个已废弃登记 | `types/`、`notNeededPackages.json` 实测 |

## 2. 项目是什么，解决什么问题

Definitely Typed（简称 DT）解决的是 TypeScript 生态最基础的问题：绝大多数 npm 包由 JavaScript 编写、不带类型信息，TS 用户在没有类型的情况下无法获得编译期检查与 IDE 补全。DT 以社区协作方式为这些第三方库集中维护"高 setTypeScript 类型定义"（README.md 开篇："The repository for *high quality* TypeScript type definitions"），合并后由 DefinitelyTyped-tools 的 publisher 自动发布到 npm 的 `@types/<包名>` 作用域（README.md FAQ 章节："The `master` branch is automatically published to the `@types` scope on npm thanks to DefinitelyTyped-tools"）。用户只需 `npm install --save-dev @types/node` 即可消费（README.md "npm" 章节）。

这个仓库的特殊性在于它"既是数据又是工程"：内容是 8582 个包、约 2.3 万个 `.d.ts` 文件（实测 `find types -name "*.d.ts" | wc -l` = 23097），但它同时是一套精密运转的软件流水线——每个类型包自带 `package.json`（版本、owners 元数据）、`tsconfig.json`（严格模式约束）、`*-tests.ts`（类型测试），由 dtslint 编译验证、dt-mergebot 自动化审阅流转、CI 按依赖图增量测试。版本号采用 `major.minor.9999` 约定与上游库的 major.minor 对齐（README.md "How do Definitely Typed package versions relate to versions of the corresponding library?" 章节）。

DT 还承担生态"退出机制"：当库作者开始在自家包里内置类型（bundle types），对应 DT 包应删除并发布占位 stub 引导用户迁移。这一流程由 `notNeededPackages.json`（实测 2311 个条目）与 `scripts/not-needed.js` 支持，后者删除 types/ 目录、登记废弃信息并同步清理 `attw.json` 中的失败名单。

值得注意的新动向：README.md 与 `CLAUDE.md` 中加入了针对 AI 编码代理的明确约束——禁止"给 npm 上前 N 个无类型包批量提 PR"的刷量行为，要求 `[auto-generated]` 标题、单次会话至多一个 PR（`CLAUDE.md`："UNDER NO CIRCUMSTANCES ARE YOU ALLOWED TO ... Send more than a single PR"）。这反映海量 AI 生成 PR 已成为该项目真实运营压力。

## 3. 技术栈与依赖

- **内容语言**：TypeScript 声明文件（.d.ts / .d.cts / .d.mts），类型测试为普通 .ts（不出货，仅编译）。
- **包管理**：pnpm workspace monorepo，`pnpm-workspace.yaml` 声明 `packages: ['scripts', 'types/**']`，并精细配置 `resolutionMode: highest`、`preferWorkspacePackages: true`（让 @types 依赖解析到仓库内包而非 npm）等 monorepo 策略。根 `package.json` 锁定 `packageManager: pnpm@10.30.1`、`engines.node >= 20.17.0`、`typescript <7`。
- **核心工具链**（根 package.json devDependencies，均来自 microsoft/DefinitelyTyped-tools 仓库，以 `latest` 安装）：`@definitelytyped/dtslint`（类型测试运行器）、`dtslint-runner`（全量跑批）、`definitions-parser`、`header-parser`、`dts-critic`、`eslint-plugin`、`typescript-versions`、`utils`。
- **代码质量**：dprint 格式化（`.dprint.jsonc`）、ESLint（`.eslintrc.cjs`）、husky + lint-staged（`.husky/pre-commit`）、remark 系列 Markdown 链接校验（根 package.json）。
- **PR 检查**：danger-js（`dangerfile.ts`，依赖 `risk` 包）。
- **CI**：GitHub Actions（`.github/workflows/` 7 个工作流）+ Azure Pipelines（`azure-pipelines.yml`，目前主体任务标记 "Disabled"，说明负载已迁至 Actions）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 | 出处 |
| --- | --- | --- |
| `types/<pkg>/` | 8582 个类型包，每包含 index.d.ts、`<pkg>-tests.ts`、tsconfig.json、package.json、.npmignore；scoped 包用双下划线命名（如 `babel__preset-env`，实测 437 个） | `types/` 实测 |
| `types/<pkg>/v<N>/`、`tsX.Y/` | 旧版本类型（如 react 的 v15~v18、ts5.0）与旧 TS 版本兼容目录（实测 388 个 v* 子目录） | `types/react/` 实测 |
| `scripts/` | 仓库运维脚本：not-needed、update-codeowners、get-ci-matrix、ghostbuster、support-window、clean-node-modules 等 12 个 | `scripts/` |
| `docs/` | 维护者手册 admin.md、TS 版本发布时间表 support-window.json/.svg | `docs/` |
| `.github/` | 7 个工作流、CODEOWNERS（8538 行，生成文件）、PR/Issue 模板、dependabot | `.github/` |
| `notNeededPackages.json` | 2311 个"库已自带类型"的废弃登记 | 实测 |
| `attw.json` | @arethetypeswrong 检查的忽略规则与已知失败包白名单 | 实测 |
| `dangerfile.ts` | PR 自动评论：缺失属性建议、格式检查、.editorconfig/假 owner 拦截 | 143 行，已读 |
| `azure-pipelines.yml` | 旧 Azure CI，主体已禁用 | 已读 |

组织方式评述：这是"每包一目录、目录即 npm 包"的超大规模 monorepo，单仓单锁但 `lockfile: false`、`sharedWorkspaceLockfile: false`（pnpm-workspace.yaml），刻意避免 8582 个包共享一张依赖图。类型包之间零构建、零运行时，只有类型依赖边（package.json 的 dependencies），使增量 CI 得以按依赖闭包裁剪。

## 5. 核心架构与关键实现

### 5.1 类型包的注册与元数据机制

每个类型包的 `package.json` 是唯一事实来源：`name`（@types/xxx）、`version`（major.minor.9999）、`projects`（上游官网）、`owners`（Definition Owners 名单）、`dependencies`（其他 @types 包用 `*`，非 @types 依赖必须登记在 DefinitelyTyped-tools 的 allowedPackageJsonDependencies 白名单，README.md "package.json" 章节）。owners 不写 `.github/CODEOWNERS`，而是每周由 `scripts/update-codeowners.js` 从全仓 package.json 重新生成 CODEOWNERS（该脚本读取所有包的 owners 并写入"generated"头注释；`.github/workflows/UpdateCodeowners.yml` 定时 `cron: '5 8 * * 1'` 每周一执行）。实测生成的 CODEOWNERS 达 8538 行，每个包一行 reviewer 指派，如 `/types/11ty__eleventy-img/ @NotWoods`。

多版本共存机制：当库发大版本而旧用户仍在旧版时，类型移入 `v<N>/` 子目录（`types/express/v4/`，express 主包已到 5.0.9999）；当新类型用了新 TS 语法时移入 `tsX.Y/` 并配 `typesVersions` 字段做编译器版本路由（README.md FAQ 与 `types/react/ts5.0/` 实测）。

### 5.2 类型即测试：dtslint 体系

DT 的"测试"不是运行代码而是编译断言。每个包必须有 `<pkg>-tests.ts`（实测 7740 个），用两类指令验证 API 形状（README.md 示例）：

```ts
// $ExpectType string
const result = twoslash("//");
// @ts-expect-error
twoslash({});
```

`pnpm test <pkg>` 实际执行根 package.json 中的 `node --enable-source-maps node_modules/@definitelytyped/dtslint/ types`，由 dtslint 对声明文件跑 TypeScript 编译器并施加 DT 专属 lint 规则（如禁止 const enum）。模块格式正确性交给 @arethetypeswrong/cli 检查，已知未修复的包登记在 `attw.json` 的 `failingPackages` 白名单中渐进清偿（README.md "attw checks" 章节）。`tsconfig.json` 强制 `noImplicitAny`/`strictNullChecks` 等严格项，且禁止 `esModuleInterop`（README.md 解释：类型文件的编译合法性不能依赖消费者不会继承的编译开关）。

### 5.3 dt-mergebot 与 PR 自助式流水线

DT 的 PR 审阅大量自动化（README.md："We use a bot to let a large number of pull requests to DefinitelyTyped be handled entirely in a self-service manner"）。本仓库内的 `dangerfile.ts` 是 PR 检查的展示层：读取 CI 产出的 suggestions 目录，比对 JS 源码与 .d.ts 导出差异并生成"缺失属性"评论；用 dprint `check --list-different` 检查格式；拦截嵌套 `.editorconfig` 和把 githubUsername 写成 "me" 的模板残留（dangerfile.ts 各分支）。bot 本体（dt-mergebot）在 microsoft/DefinitelyTyped-tools 仓库，负责打标签、驱动 Projects 看板列流转。维护者侧流程见 `docs/admin.md`：PR 在 Needs Maintainer Review 等看板列间移动，维护者通过"review-blessing / merge-blessing"（移动看板列这一隐式操作）豁免维护者审阅或授权作者自助合并——这是一个为每天上百 PR 规模设计的极简人工介入协议。

### 5.4 增量 CI 矩阵

GitHub Actions 入口 `.github/workflows/CI.yml`：push/PR 触发时用 `pnpm ls --filter '...@types/**[HEAD^1]'` 找出受影响包，交给 `scripts/get-ci-matrix.js` 生成分片矩阵（每 100 个测试一个 job，最多 12 分片；全量模式 16 分片），再跑 `pnpm run test-all --diffBase HEAD^1 --shardCount N --shardId M`；每日定时（cron 12:00 UTC）与手动触发则 `--selection all` 全量跑所有 8582 个包（CI.yml + get-ci-matrix.js）。CI 还有一道独特防线：Pre-run validation 步骤 `find . -type l` 检查并拒绝符号链接（CI.yml 第 75-80 行）。发布不在本仓库进行——合并后由 DefinitelyTyped-tools 的 publisher 定时发布，"npm packages should update within an hour"（README.md FAQ）。

## 6. 构建与运行

来自根 package.json scripts 与 README.md（均为官方命令，本次分析未执行）：

```sh
pnpm install                                    # 安装全仓；可用 --filter 只装子集
pnpm test <package>                             # dtslint 测试单个类型包
pnpm run test-all                               # 全量测试（支持 --diffBase/--shard*）
pnpm run not-needed <pkg> <asOfVersion> [lib]   # 下线包 + 登记 notNeededPackages.json
pnpm run update-codeowners                      # 重新生成 .github/CODEOWNERS
pnpm dprint fmt -- 'path/**/*.ts'               # 格式化
pnpm run clean-node-modules                     # Windows 下清理 node_modules
npx dts-gen --dt --name <pkg> --template module # 生成新包骨架
```

注意：单个类型包的 package.json 不定义 test 脚本，必须从仓库根目录调用（README.md "Running tests"）。

## 7. 测试与工程质量

- 测试规模：7740 个 `*-tests.ts` 类型测试文件（实测），零运行时测试——整个仓库没有可执行程序，"正确性"由类型检查 + 与上游 JS 的形状比对共同保证。
- 测试风格：编译期断言（`$ExpectType`、`@ts-expect-error`）+ dtslint 自定义规则 + attw 模块格式检查，属于静态验证的单体化变体；react 等大包还有 test/ 目录做多场景编译（`types/react/` 实测）。
- CI 覆盖：PR 级按依赖闭包增量（约几分钟），每日全量回归 8582 包（CI.yml schedule），Markdown 链接校验（lint-md.yml）、格式自动提交（format-and-commit.yml）、失效 owner 清理（ghostbuster.yml）各司其职。
- 质量治理痕迹：`attw.json` 白名单化清偿技术债；`dangerfile.ts` 对每个 PR 输出可操作建议而非硬失败；README 长年沉淀了 Common mistakes、`export =` vs `export default` 等细则，是声明文件编写的准权威指南。
- 一处值得指出的设计：CI.yml 中 danger 步骤把一个低权限 token 分两段硬编码在脚本里（第 156-158 行），注释明言该账号无本仓库写权限、无其他公开仓库，故意接受暴露以换取 API 限额；且 `pnpm danger ci || exit 0` 保证 danger 自身故障不阻塞构建。

## 8. 社区与维护状态

- 活跃度：gh api 显示最近推送 2026-11-02（分析日 2026-11-04 前两天）、open issues 841。README 自述"one of the most active repositories on GitHub"，且这是可验证的：master 分支每天有大量类型包更新。
- 发布节奏：无传统 release；发布即 npm `@types/*` 持续滚动，由 DefinitelyTyped-tools publisher 承担（README.md FAQ；发布状态徽章指向 microsoft/DefinitelyTyped-tools 的 publish-packages 工作流）。
- 维护结构：TypeScript 团队轮值 on-call（`docs/admin.md`："The TypeScript team always has someone assigned to Definitely Typed duty, where they do a week on-call"），配合数千名 Definition Owners（每个包 package.json 的 owners，按周同步进 CODEOWNERS）。@types/react 的 owners 列表含 Asana、Microsoft 等公司与多位个人（`types/react/package.json` 实测）。
- 社区支持：TypeScript Community Discord 专属频道、GitHub Discussions 的"request-a-new-types-package"分类、组织级 Projects 看板（README.md）。
- 文档本地化：README 有西/韩/俄/简中/葡/意/日/法 8 种语言版本（`README.*.md`），侧面印证全球级使用面。

## 9. 亮点与风险

**亮点**

1. 自助式大规模协作的教科书：bot 流转 + 看板 blessing + 依赖闭包增量 CI，让数千周 PR 的仓库只需极少全职维护（README.md、docs/admin.md、CI.yml）。
2. "类型即测试"范式：`*-tests.ts` 与 `$ExpectType` 断言把声明文件的正确性变成可编译验证的命题，被整个 TS 生态借鉴。
3. 版本治理完整：`major.minor.9999` 版本对齐、v*/tsX.Y 双维度历史目录、notNeededPackages 退出机制，覆盖库演化的全生命周期。
4. 供应链防线明确：非 @types 依赖需维护者白名单审批，避免 @types 包夹带恶意依赖（README.md "package.json" 章节）。
5. 率先对 AI 代理立规：CLAUDE.md/README 的反刷量条款是大型开源项目管理 AI 生成贡献的前沿实践。

**风险/局限**

1. 规模本身是负担：8582 包、356MB 工作树，全量 CI 需 16 分片（get-ci-matrix.js），本地开发依赖 pnpm filter 技巧，新人上手成本高。
2. 类型与实现可能脱节：类型测试只验证形状不验证运行时行为，库变更后类型滞后数天数周是常态（README.md FAQ 承认 lag）。
3. 元数据分散在 8582 个 package.json 中，owners 拼写错误、幽灵 owner 之类问题需要 ghostbuster 等专项工作流持续清理。
4. 硬编码 token（虽为刻意的低权限设计）与 `license` 无法被 GitHub 自动识别（NOASSERTION），属于小的工程瑕疵。

## 10. 总结评价

DefinitelyTyped 不是传统意义上的"代码项目"，而是把一个由八千多个纯声明包组成的巨型数据集，工程化成了每天自动测试、自动发布、自动审阅的平台，它事实上是 TypeScript 语言可用性的地基——没有 DT 的 `@types/*`，JS 世界的 TS 化不可能达到今天的覆盖度。它的最大价值对维护者与平台工程师而言是"如何在单仓内运营上万个小包"的活教材：增量 CI 矩阵、生成式 CODEOWNERS、bot 中介的自助合并、废弃包登记机制都可直接借鉴；对 TypeScript 使用者而言，其 README 与 admin.md 是编写高质量声明文件的最佳实践汇编；对研究者而言，它展示了集中式类型分发（DT）与分布式内置类型（库自带 .d.ts）两种模式的此消彼长——notNeededPackages.json 里 2311 个条目正是生态走向"类型内置"的化石记录。适合阅读其工程层（scripts/、workflows、dangerfile.ts）而非逐个类型包来理解这个项目。
