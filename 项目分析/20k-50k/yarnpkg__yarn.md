# yarnpkg/yarn 项目分析

> 星数 41800（快照 2026-10-24）｜主语言 JavaScript（Flow 注解，非 TypeScript）｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

**一句话定位：Yarn 经典版（1.x）的源码仓库——一个已被冻结维护的历史版本，主线开发已整体迁移至 yarnpkg/berry（2.x/3.x/4.x）。**

| 项目 | 事实 | 出处 |
| --- | --- | --- |
| 名称 | yarnpkg/yarn（默认分支 master） | gh api repos/yarnpkg/yarn |
| 星数 | 41456~41800（快照期内） | gh api |
| 主语言 | JavaScript + Flow 类型注解（`/* @flow */` 遍布 src/） | `src/cli/index.js` 等 |
| 许可证 | BSD-2-Clause | `LICENSE`、`package.json` |
| 当前版本 | 1.23.0-0（master 上的开发预发布号）；最新正式 release v1.22.22（2024-03-09） | `package.json`、gh api releases |
| 最近提交 | 2026-02-12（仅删除 CircleCI 夜间 E2E 测试）；上一批为 2024-05 签名工作流 | gh api commits |
| 维护状态 | **冻结**：README 顶部明确声明 1.x 仅保留历史与偶发热修复 | `README.md` 第 1 段 |

## 2. 项目是什么，解决什么问题

Yarn 1.x 是 2016 年 Facebook/Google 等推出的 JavaScript 包管理器，针对当年 npm 的三大痛点：安装慢（无缓存、串行请求）、不可靠（无锁文件、依赖图不确定）、不安全（无完整性校验）。README 用三个关键词概括：**Fast**（全局缓存 + 并发请求）、**Reliable**（`yarn.lock` 锁文件 + 确定性安装算法）、**Secure**（安装前校验包的 checksum）（`README.md`）。

它的历史贡献非常大：`yarn.lock` 锁文件格式、workspaces monorepo 协议、离线镜像、`--frozen-lockfile`（CI 确定性安装）等机制均由 1.x 首创或普及，后来 npm 的 package-lock.json、workspaces 等都吸收了这些设计。

**但必须如实指出：本仓库是冻结的历史版本。** `README.md` 顶部 Important note 原文声明：本仓库仅承载 Yarn 1.x（最新 1.22），新版本（2.x/3.x/4.x）在 yarnpkg/berry 开发，这里"mostly kept for historical purposes and the occasional hotfix"，并强烈建议用户按官方迁移指南迁到新版本。`CHANGELOG.md` 1.22.11 条目也写明"Yarn 1.x won't receive further functional improvements"。

1.x 与 Berry 的分仓演进关系在代码里有实际痕迹：1.x 内置了两座"迁移桥"——`yarn init -2` 会安装 berry 到新项目（`src/cli/commands/init.js:32`，`flags['2'] ? 'berry' : flags.install`），以及 `src/util/generate-pnp-map-api.tpl.js` 会为项目生成 PnP 映射文件，方便向 Berry 的 Plug'n'Play 体系靠拢。

生态位对比：npm 随 Node 内置、生态默认；yarn 1.x 开创了锁文件与确定性安装但已冻结，存量大（大量老项目仍用 `yarn` 命令）；pnpm 走内容寻址 + symlink 路线，主打磁盘节省；Berry（yarn 2+）走 Plug'n'Play 路线，彻底改造 node_modules。Berry 的 README 同样提供 `nodeLinker` 设置让用户自选 node_modules/pnpm/PnP 三种安装模式（`README.md` 迁移段落）。

## 3. 技术栈与依赖

- **语言**：JavaScript（ES2015+ 模块语法）+ Flow 类型注解（`flow-bin ^0.66`，devDependencies），构建时经 Babel 6 转译（`gulpfile.js`、`babel-preset-env`）。注意任务卡标注"TypeScript"，实际源码为 Flow，无 .ts 文件（`src/` 全目录核实）。
- **运行时**：Node.js >= 4.0.0（`package.json` engines，是典型的 2016-2018 兼容区间）。
- **关键运行时依赖**（`package.json`）：
  - `request ^2.87`：HTTP 请求（已废弃的老库，冻结原因之一）
  - `commander ^2`：CLI 参数解析
  - `semver ^5`、`js-yaml ^3`（lockfile 解析）、`ssri ^5`（子资源完整性）、`micromatch`（glob 过滤）
  - `tar-fs`/`tar-stream`/`gunzip-maybe`：tarball 拉取解压
  - `proper-lockfile`：全局缓存文件锁；`inquirer`：交互提示
- **构建工具链**：gulp（增量编译）、webpack（`scripts/build-webpack.js` 打单文件 dist）、jest ^22（测试）、eslint + prettier（lint）、flow（类型检查）。

## 4. 目录结构与代码组织

| 目录 | 职责 | 出处 |
| --- | --- | --- |
| `src/cli/` | CLI 入口与 44 个子命令实现（add/install/publish/workspaces/audit 等） | `src/cli/index.js`、`src/cli/commands/` |
| `src/config.js` | 全局/项目配置、registry、缓存路径（1001 行） | `src/config.js` |
| `src/package-resolver.js` / `src/package-request.js` | 依赖解析管线：请求队列 → 解析器分发 → 锁文件优先 | `src/` |
| `src/resolvers/` | 解析器实现：registries（npm/yarn）+ exotics（git/github/bitbucket/gitlab/gist/file/link/tarball） | `src/resolvers/` |
| `src/fetchers/` | 包获取：tarball/git/copy/workspace 四种 fetcher | `src/fetchers/` |
| `src/lockfile/` | yarn.lock 解析与序列化（`@yarnpkg/lockfile` 的前身） | `src/lockfile/` |
| `src/package-hoister.js` | node_modules 扁平化提升算法（1024 行，核心） | `src/package-hoister.js` |
| `src/package-linker.js` / `integrity-checker.js` / `package-install-scripts.js` | 落盘链接、`.yarn-integrity` 校验、生命周期脚本执行 | `src/` |
| `src/reporters/` | 输出层：console/JSON/buffer 三种 reporter + 多语言文案（`reporters/lang/`） | `src/reporters/` |
| `src/registries/` | npm/yarn registry 适配 | `src/registries/` |
| `packages/` | 两个子包：`@yarnpkg/lockfile` 独立发布、`pkg-tests` 规格测试框架 | `packages/` |
| `__tests__/` | jest 单元/集成测试（95 个测试文件 + 5089 个 fixture） | `__tests__/` |
| `end_to_end_tests/` | Ubuntu 端到端 shell 测试脚本（夜间 E2E 已于 2026-02 移除） | `end_to_end_tests/` |
| `bin/`、`scripts/` | 启动器（yarn.js/cmd/ps1）与发布脚本（deb/Chocolatey/Windows 安装器/webpack bundle） | `bin/`、`scripts/` |

组织方式评述：**单体单包 + 2 个支持子包**，非 monorepo。核心架构是清晰的管道分层：CLI 命令 → Config → Resolver（解析出依赖图）→ Fetcher（下载）→ Hoister/Linker（扁平落盘）→ InstallScripts（生命周期钩子），各环节都有独立模块文件，边界用 Flow 类型约束。工程化程度对 2017 年的 JS 项目而言相当高。

## 5. 核心架构与关键实现

### 5.1 扁平化提升算法（package-hoister.js，1024 行）

Yarn 1.x 最有研究价值的模块。目标：把嵌套依赖图"提升"（hoist）到顶层 node_modules，减少重复、兼容 Node 的逐级查找算法。`seed()` 用层级队列（levelQueue）逐层播种，显式排序保证确定性——先按 key 字母序、再让无 peer 依赖的包先提升（`src/package-hoister.js:137-175`）：

```js
seed(patterns) {
  this.prepass(patterns);
  for (const pattern of this.resolver.dedupePatterns(patterns)) {
    this._seed(pattern, {isDirectRequire: true});
  }
  // ...
  queue = queue.sort(([aPattern], [bPattern]) => sortAlpha(aPattern, bPattern));
  // hoist packages without peer dependencies first
  const peerDependencies = Object.keys(pkg.peerDependencies || {});
  const areDependenciesFulfilled =
    peerDependencies.every(peerDependency => availableSet.has(peerDependency));
```

`hoist()` 负责换位：删除旧 key、`getNewParts` 计算新位置，若被上层同名包满足则标记 "Satisfied from above"，否则 `declareRename` → `taintParents` 沿祖先链"污染"名字防止碰撞（`src/package-hoister.js:466-510`）。另有 nohoist（工作区不提升）与 focus（浅安装）扩展。每个 HoistManifest 维护 `history` 数组记录提升轨迹，`yarn why` 的解释能力即来源于此。

### 5.2 解析管线：锁文件优先 + 双层解析器

`package-resolver.js` 持有 `BlockingQueue` 串行化同名请求，`package-request.js` 的 `getLocked()` 总是**先查 yarn.lock**：锁内有 `resolved` URL+hash 就直接复用（含 git 协议探测），实现确定性；锁缺失才走 resolver 全量解析（`src/package-request.js:59-78`）。解析器按目标类型分发：`src/resolvers/registries/npm-resolver.js` 处理 registry 语义版本，`src/resolvers/exotics/` 处理 git/github/file/tarball 等"异形"依赖，每个 exotic 一个文件，职责单一。

### 5.3 网络弹性：并发队列 + 自动重试

`src/util/request-manager.js` 实现签名特性"Network Resilience"：默认并发 8（`NETWORK_CONCURRENCY = 8`，`src/constants.js:37`）、超时 30 秒、最多重试 5 次（`maxRetryAttempts = 5`，`src/util/request-manager.js:93-94`），单请求失败不拖垮整个安装——这正是 README "Reliable" 承诺的实现层。

### 5.4 完整性校验与安装脚本编排

`src/integrity-checker.js`（454 行）读取 `node_modules/.yarn-integrity` 快照对比当前依赖树，决定跳过/重装；`src/fetchers/tarball-fetcher.js` 用 `ssri` 校验包哈希后才落盘，对应 README "Secure" 承诺。`src/cli/commands/install.js`（约 1200 行）把上述模块编排成完整 install 流程，是理解管线的最佳入口。

## 6. 构建与运行

全部来自 `package.json` scripts 与 `.circleci/config.yml`（未运行验证，静态摘录）：

```bash
yarn install --frozen-lockfile   # 装依赖（CI 标准做法）
yarn build                       # gulp build：Babel 编译 src → lib
yarn build-bundle                # webpack 打单文件 dist
yarn lint                        # eslint . && flow check
yarn test-only                   # jest --verbose（__tests__ 全量）
yarn test                        # lint + test-only
yarn test-coverage               # jest --coverage
yarn pkg-tests                   # packages/pkg-tests 规格测试
```

发布产物链：`build-dist`（tar+bundle）→ deb/Chocolatey/Windows 安装器脚本（`scripts/build-deb.sh` 等）；release 签名走 `.github/workflows/signing.yml`（GPG，workflow_dispatch 手动触发，2024-05 最后更新）。

## 7. 测试与工程质量

- **测试规模**：`__tests__/` 非夹具测试文件 95 个，夹具 5089 个（大量最小化 package.json 场景）；另加 `end_to_end_tests/`（Ubuntu shell）与 `packages/pkg-tests/`（独立规格测试框架，用 fixtures 动态生成 registry）。三层测试结构在当年属先进水平（`__tests__/`、`packages/pkg-tests/`）。
- **框架**：jest ^22 + 快照测试（`__tests__/__snapshots__/`）+ jasmine 风格断言；测试覆盖 hoister、resolver、lockfile、reporters、commands 等全部核心模块（`__tests__/` 目录清单）。
- **CI**：三平台矩阵——CircleCI（Linux/macOS，`test_build` + jest）、Azure Pipelines（acceptance tests，`azure-pipelines.yml`）、AppVeyor（Windows，`appveyor.yml`）；2026-02-12 最后一次提交删除了 CircleCI 夜间 E2E（gh api commits），属维护性裁撤而非功能开发。
- **质量观察**：Flow 类型 + 自定义 eslint 规则（`scripts/eslint-rules/`，如禁止非受控 mkdir）+ commitizen 提交规范；代码风格统一。技术债明显：依赖 `request`（废弃库）、Babel 6、jest 22、Node>=4 兼容层，全部停留在 2018-2019 世代——冻结仓库的自然状态。

## 8. 社区与维护状态

- **维护状态（核心结论）**：已冻结。证据链：README 顶部声明（`README.md`）、CHANGELOG 1.22.11 声明（`CHANGELOG.md`）、gh api 提交历史（2026-02 仅 CI 清理，2024-05 仅签名工作流，2022-09 为 README/PR 模板）、最新 release v1.22.22 停于 2024-03-09。
- **热修复节奏**：冻结后仍偶发安全/兼容热修复（1.22.19/20/21/22 分布在 2022-2024），如 Corepack 兼容（`CHANGELOG.md`）。
- **社区体量**：贡献者约 438 人（gh api contributors 分页汇总）；open issues 2055（gh api，冻结仓基本不再清零）；赞助商 Doppler/WorkOS 通过 Open Collective 支持的是 berry 主线（`README.md` sponsors 段）。
- **开发重心**：全部在 yarnpkg/berry（2.x 2020 → 3.x 2021 → 4.x 2023），本仓无未确认的持续功能开发。

## 9. 亮点与风险

**亮点**
1. 教科书级的包管理器管线分层：CLI → Config → Resolver → Fetcher → Hoister → Linker → Scripts，每层独立可测（`src/` 整体结构）。
2. `package-hoister.js` 的确定性扁平化算法（显式排序 + taint 机制 + 提升历史追踪），是理解 node_modules 语义与 npm hoisting 差异的绝佳教材。
3. 锁文件优先的解析策略 + `--frozen-lockfile`，开创了 JS 生态的确定性安装范式（`src/package-request.js`）。
4. 工程质量超前：三层测试（单测/规格/E2E）、Flow 类型、三平台 CI、发布签名（`.circleci/config.yml`、`packages/pkg-tests/`、`signing.yml`）。
5. 对生态的持久影响：yarn.lock 格式、workspaces、离线镜像均成为行业标准，`@yarnpkg/lockfile` 至今仍被独立使用（`packages/lockfile/`）。

**风险/局限**
1. **已冻结**：无功能更新、2055 个 open issues 无人清理；安全修复也只是"偶发热修"（`README.md`、gh api）。
2. 依赖栈陈旧：`request` 已废弃、semver 5 / js-yaml 3 / Node>=4 兼容层，继续投入有安全与维护风险（`package.json`）。
3. 与现代工具的兼容靠补丁维持（Corepack、`.yarnrc.yml` 忽略、PnP 模板），语义偏差可能随时间放大（`CHANGELOG.md` 1.22.6-1.22.11）。
4. Flow 已被 TS 生态边缘化，直接复用这套代码的迁移成本高（`src/*.js` 全部 `/* @flow */`）。

## 10. 总结评价

yarnpkg/yarn 是 JS 包管理器历史的里程碑式仓库：它用 yarn.lock、workspaces 和确定性 hoisting 重塑了 npm 生态的工程标准，其架构分层与测试体系至今仍有很高的阅读价值。但今天访问这个仓库应当带着明确的历史语境——这是**冻结的 1.x 存档**，4 万星更多是对其历史贡献的"荣誉分"；主线早已在 yarnpkg/berry 延续，本仓只剩安全热修与 CI 尘埃落定式的清理。适合两类人：一是研究包管理器设计（依赖解析、hoisting、锁文件）的工程师，1.x 代码规模适中（src/ 约 2.6 万行）且管线清晰，比读 berry 的插件体系容易得多；二是仍在维护 1.x 遗留项目、需要理解 `yarn why`/hoist 行为或评估迁移成本的开发者。生产新项目请直接用 berry 或 npm/pnpm。
