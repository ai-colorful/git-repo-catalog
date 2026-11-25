# FuelLabs/fuels-ts 项目分析

> 星数 43000（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-07

## 1. 项目速览

一句话定位：Fuel 区块链的官方 TypeScript SDK——用一套 monorepo 打包了从 GraphQL 节点访问、ABI 编解码、类型生成到钱包/合约调用的全栈开发能力，是 fuels-rs 的姊妹 TS 实现。

| 项目 | 内容 |
| --- | --- |
| 名称 | FuelLabs/fuels-ts |
| 星数 | 42,911（gh api 实测 2026-11-07） |
| 主语言 | TypeScript |
| 许可证 | Apache-2.0（`LICENSE`、根 `package.json`） |
| 创建时间 | 2021-09-03（gh api） |
| 最近活跃 | 最近 push 2026-10-30，默认分支 master（gh api） |
| 最新版本 | 0.104.0（`packages/fuels/package.json`、`packages/fuels/CHANGELOG.md`；GitHub Release 标签停留在 v0.103.0，npm 走 changesets 持续发版） |
| 维护方 | Fuel Labs <contact@fuel.sh>（`package.json` author 字段） |
| 贡献者 | 约 62 人（contributors API 分页推断，含匿名） |

## 2. 项目是什么，解决什么问题

Fuel 是一个基于 UTXO 模型的模块化区块链执行层（Fuel VM / fuel-core 节点 / Sway 智能合约语言 / Forc 工具链）。fuels-ts 是官方为这一整套技术栈提供的 TypeScript SDK（README 自述 "Typescript SDK for Fuel"，`README.md`），让前端与 Node.js 应用可以在不依赖 Rust 的情况下完成与 Fuel 网络的全部交互。

它解决的核心问题链：Sway 合约编译产物是 ABI JSON + 字节码，TS 应用需要一个类型安全层把"调用合约函数"翻译成 Fuel VM 的交易编码与 GraphQL 请求。仓库通过 `abi-typegen` 从 ABI JSON 生成 TS 定义（`packages/abi-typegen/package.json` 描述 "Generates Typescript definitions from Sway ABI Json files"），`abi-coder` 负责 ABI 编解码，`contract`/`program`/`script` 提供合约部署与调用抽象，`account` 提供钱包、签名、谓词与 Provider，最终由伞形包 `fuels` 统一出口并附带 CLI（`packages/fuels/src/index.ts` 逐个 re-export 全部子包）。

与姊妹仓 fuels-rs 是同一套 SDK 思想在两种语言中的实现：同样的 Provider-over-GraphQL 分层、同样的钱包/签名体系、同样的 typegen 产物形态，甚至命令行心智模型也一致（README 的官方文档区把 "Rust SDK — fuels-rs" 与本 SDK 并列，`README.md`）。命名与组织上明显借鉴 ethers.js v6：`ContractFactory`、`Interface`、`FunctionFragment`、`JsonAbi` 等概念一一对应（`packages/contract/src/contract-factory.ts`、`packages/abi-coder/src/Interface.ts`）。

仓库还内置了完整的"开发到上线"工具链：`fuels` CLI 提供 init/build/deploy/dev/node/typegen/versions 七个命令（`README.md`），其中 `dev` 可带热重载启动本地 Fuel 节点；`create-fuels` 是 `npm create fuels` 的脚手架，配合 `templates/nextjs`、`templates/vite` 两套模板生成全栈 dApp（`templates/`、`packages/create-fuels/src/`）。

由于燃料费估算、UTXO 资源挑选、谓词估算等都是链上概念，SDK 在 `Provider` 里封装了 `assembleTx`、dryRun、依赖预估算等高阶逻辑（`packages/account/src/providers/provider.ts`），让开发者基本可以只面向 TS 类型编程。

## 3. 技术栈与依赖

- 语言与运行时：TypeScript 5.7.3（根 devDependencies），Node ^20.18 || ^22 || ^24（根 `package.json` engines）。
- 构建体系：pnpm 9.4 + Turborepo（`turbo.json`），tsup 打包（`internal/tsup`），graphql-codegen 生成 GraphQL SDK（`packages/account/package.json` 的 `build:operations`）。
- 关键第三方库：
  - `graphql-request` + `graphql-tag`：Provider 的 GraphQL 传输层（`packages/account/src/providers/provider.ts` 头部 import）。
  - `@noble/curves` secp256k1：签名核心（`packages/account/src/signer/signer.ts`）。
  - `handlebars`（经 hbs 模板渲染）：typegen 代码模板（`packages/abi-typegen/src/templates/renderHbsTemplate.ts`）。
  - `ramda`（clone 等）、`@fuel-ts/math`（自研 u64/BN）、`@fuels/vm-asm`（Fuel VM 汇编，用于合约调用脚本，`packages/fuels/src/index.ts` 导出）。
- 测试：vitest 3（node/browser 双 project）、@vitest/coverage-istanbul、Playwright（浏览器 demo E2E，`playwright.config.ts`）。
- 质量工具：eslint 8 + airbnb 规则 + tsdoc 插件、prettier、knip（死代码检测）、syncpack、turbo 缓存（根 `package.json` scripts）。

## 4. 目录结构与代码组织

顶层（tarball 解压根）：

| 目录 | 职责 |
| --- | --- |
| `packages/` | 21 个发布包/内部包（见下） |
| `apps/` | docs（VitePress 文档站，`apps/docs/package.json`）、docs-api（typedoc API 文档）、6 个 demo 工程（demo-fuels、demo-nextjs、demo-bun-fuels、demo-wallet-sdk-react 等） |
| `internal/` | forc 与 fuel-core 二进制封装（工具链下载/构建）、tsup 共享配置、check-imports/check-tests 校验脚本 |
| `templates/` | nextjs、vite 两套 create-fuels 模板（含 sway-programs/） |
| `scripts/` | CI 辅助：tests-ci.sh、tests-find.sh（按 tag 分流 node/browser/e2e/integration/network）、changeset 发布脚本、lint-package-jsons 等 |
| `playwright-tests/` | 浏览器端 E2E |
| `.fuel-core/` | 本地节点快照：chainConfig.json、faucets.json、state_transition_bytecode.wasm |

`packages/` 主要成员（`packages/<name>/package.json`）：

| 包 | 职责 |
| --- | --- |
| `fuels` | 伞形包 + CLI（bin: fuels），聚合导出全部子包 |
| `abi-coder` | ABI 编解码：Interface/FunctionFragment + 17 个类型 coder |
| `abi-typegen` | Sway ABI JSON → TS 定义（Handlebars 模板） |
| `account` | Account/Wallet/Predicate/Signer/Connectors/HDWallet/Mnemonic + Provider（GraphQL） |
| `contract` | ContractFactory、blob 分块部署（loader/） |
| `program` | 合约调用抽象：contract-call-script、instruction-set（VM 指令集拼接） |
| `script` / `predicate` 位置 | script 包独立；Predicate 在 account 包内（`packages/account/src/predicate/predicate.ts`） |
| `transactions` | 交易编码/回执解码（TransactionCoder、receipt.ts） |
| `address` `crypto` `math` `merkle` `hasher` `utils` `errors` `versions` | 基础库：Bech32 地址、加密、bignum、默克尔证明、sha256、工具、错误码、工具链版本校验 |
| `recipes` | Sway 程序配方（contracts/） |
| `create-fuels` | npm create fuels 脚手架 |
| `fuel-gauge` | 集成/e2e 测试仓（非发布）：84 个 Forc.toml Sway 测试工程 + 53 个测试文件 |

组织方式评述：典型的强内聚 pnpm monorepo——每个基础能力独立成包、workspace:* 相互依赖、`fuels` 一个包统一出口；构建用 turbo 编排，测试按 project（node/browser）与 tag（e2e/integration/network）分流。这是"一 SDK 多包"的标准形态，比单包分发更利于按需引用，但也带来 21 份 package.json 的版本同步成本（用 changesets + syncpack 缓解）。

## 5. 核心架构与关键实现

### 5.1 Provider：GraphQL 访问层（packages/account/src/providers/provider.ts，2748 行）

Provider 是 SDK 的单例入口，封装 fuel-core 的 GraphQL API。schema 与操作定义随仓库维护（`packages/account/src/providers/fuel-core-schema.graphql` 1725 行、`operations.graphql` 1159 行），构建时用 graphql-codegen 生成 `__generated__/operations` SDK（`packages/account/package.json` 的 prebuild → `build:operations`；tarball 中该目录为构建产物故不存在）。Provider 职责包括：链/节点信息与 gas 配置读取、UTXO 资源挑选与缓存（`resource-cache.ts`）、dryRun 与交易依赖预估算、`assembleTx` 统一收费与资金组装（`assemble-tx-helpers.ts`）、自动重试 fetch（`utils/auto-retry-fetch.ts`）与订阅（`fuel-graphql-subscriber.ts`）。典型 API（`README.md` 示例）：

```ts
const provider = new Provider('https://mainnet.fuel.network/v1/graphql');
const chainId = await provider.getChainId();
const gasConfig = await provider.getGasConfig();
const baseAssetId = await provider.getBaseAssetId();
```

### 5.2 abi-coder：ABI 编解码（packages/abi-coder）

`Interface` 接收 JsonAbi，按 `encodingVersion` 选择 v0/v1 编码策略，旧 ABI 经 `transpileAbi` 归一化（`packages/abi-coder/src/Interface.ts` 构造函数：`this.jsonAbiOld = transpileAbi(jsonAbi)`）。17 个类型 coder 一一对应 Sway 类型系统：Struct/Tuple/Enum/Option/Vec/String/StdString/StrSlice/RawSlice/Byte/RawPtr 系列（`packages/abi-coder/src/encoding/coders/`），v1 策略在 `encoding/strategies/getCoderV1.ts`。函数寻址支持三种方式：函数名、签名 `transfer(address,uint256)`、4 字节选择器（`getFunction`，Interface.ts）。这与 fuels-rs 的 abi 编解码层同构，编码版本随 ABI 1.1/1.2 演进（`packages/fuels/CHANGELOG.md` 0.104.0 条目）。

### 5.3 ContractFactory 与 blob 分块部署（packages/contract/src/contract-factory.ts，508 行）

`ContractFactory` 从 ABI + 字节码构造合约，`deploy` 支持 salt/storageSlots/stateRoot/configurableConstants 覆盖，并实现了大合约的分块部署：字节码切片后以 blob 交易上链、再由 loader 字节码按需加载（`deployContractOptions.chunkSizeMultiplier`、`const CHUNK_SIZE_MULTIPLIER = 0.95`、`loader/getContractChunks.ts`）。返回的 `DeployContractResult` 提供 `waitForTransactionId()` / `waitForResult()` / `waitForPreConfirmation()` 三级确认 API（contract-factory.ts 第 43-56 行），对应链上交易从提交到预确认的异步窗口。

### 5.4 钱包与账户体系（packages/account/src）

- `signer/signer.ts`：基于 @noble/curves 的 secp256k1，Fuel 地址即未压缩公钥（去掉 1 字节编码前缀，`this.address = new Address(this.publicKey)`），签名为 64 字节 r||s 并把 recovery 位塞进 s 首字节——Fuel 规范的自有格式，与以太坊不同。
- `wallet/`、`mnemonic/`、`hdwallet/`：锁定/解锁钱包、BIP39 助记词、HD 派生；`keystore-wallet.ts` 提供加密 keystore。
- `predicate/predicate.ts`（381 行）：Predicate 账户抽象——把验证脚本作为"花费条件"上链，调用时附带谓词数据，是 Fuel 版账户抽象的核心封装。
- `connectors/fuel-connector.ts`：钱包连接器抽象基类，定义 `hasWallet`/`connect`/`signMessage`/`sendTransaction` 等事件与接口，供浏览器钱包插件对接（fuel-connectors 生态的标准接口在本包定义）。

### 5.5 fuels CLI（packages/fuels/src/cli/）

伞形包同时是 CLI（bin: fuels → fuels.js，`packages/fuels/package.json`）。`fuels build` 调 Forc 编译 Sway 工作区并跑 typegen 生成 TS；`fuels dev` 启动本地 fuel-core 节点 + 热重载（`setup-launch-node-server.ts`、`packages/account/src/test-utils/launchNode.ts` 负责拉起 fuel-core 二进制并等待 GraphQL health 查询成功，见 0.104.0 changelog）。工具链版本由 `packages/versions` 校验（`checkFuelCoreVersionCompatibility`，provider.ts import）。

## 6. 构建与运行

来自根 `package.json` scripts 与 `CONTRIBUTING.md`：

```bash
pnpm install          # Node ^20.18 || ^22 || ^24, pnpm 9.4
pnpm build            # turbo run build（各包 tsup；account 包 prebuild 先跑 graphql-codegen）
pnpm dev              # nodemon watch 模式构建
pnpm test             # vitest node project + coverage（scripts/tests-find.sh --node 选文件）
pnpm test:browser     # vitest browser project
pnpm test:e2e         # e2e 标签测试
pnpm test:integration # 集成测试
pnpm lint             # type:check-tests + eslint + prettier + type:check
```

Sway 测试工程需要 Forc；`internal/forc`、`internal/fuel-core` 会在安装/构建流程中获取对应版本二进制（`CONTRIBUTING.md` 提到从 VERSION 文件指定分支构建 forc/fuel-core）。本地节点数据在 `.fuel-core/`（`pnpm node:run` / `node:clean`）。

## 7. 测试与工程质量

- 规模：packages 下 316 个 `*.test.ts`，非测试 src TS 文件 529 个、约 4.2 万行（tarball 统计，不含 __generated__）。单测与集成代码量接近 1:1，覆盖密度高。
- 分层：单元测试在各包内（如 `packages/abi-coder/src/encoding/coders/*.test.ts` 与 coder 一一对应）；`packages/fuel-gauge` 是专门的集成/e2e 仓，内含 84 个 Forc.toml Sway 测试合约工程（`packages/fuel-gauge/test/fixtures/forc-projects/`），测试主题覆盖 fee、funding、predicate、blob 部署、pre-confirmation、reentrant calls 等（fuel-gauge/src/*.test.ts 文件名可见）。
- 环境：vitest workspace 双 project（node/browser，`vitest.workspace.ts`），浏览器跑法用 memfs/vite-plugin-node-polyfills 打包适配；Playwright 单独跑 demo 应用（`playwright.config.ts`，chromium）。
- CI：`.github/workflows/test.yaml` 矩阵 node 20.19.2/22.16.0/24.1.0 + browser 四环境，另有 integration job（bun + 自建 npm registry npm-packages.fuel.network 安装 pr-xxx 预览版做真实联调）；13 个 workflow 覆盖 lint、codeql、changesets 校验、release、stale 等（`.github/workflows/`）。
- 质量观察：eslint `--max-warnings 0`、tsdoc 文档检查、knip 死代码检测、syncpack 依赖一致性、coverage diff 报告（`test:coverage-diff`）——工程化程度在同类 SDK 中属第一梯队。发布走 changesets 并带 npm provenance（`.github/workflows/release.yaml` `NPM_CONFIG_PROVENANCE: "true"`）。

## 8. 社区与维护状态

- 活跃度：最近 push 2026-10-30（gh api），仓库持续高频更新；CHANGELOG 至 0.104.0，0.x 小版本迭代密集（`packages/fuels/CHANGELOG.md`）。
- 发版节奏：changesets 驱动，master 分支 push 触发 release workflow（`.github/workflows/release.yaml`），npm 为准、GitHub Release 标签相对滞后（最新标签 v0.103.0，2026-01-30，gh api）。
- 贡献者结构：约 62 名贡献者（contributors API），Fuel Labs 团队主导，`.github/CODEOWNERS` 存在。
- 社区设施：CONTRIBUTING.md、SECURITY.md、CODE_OF_CONDUCT.md、issue 模板（1-bug.yaml/2-feature.yaml）、Discord/Forum 链接齐全（`README.md`）。文档站独立于仓库（apps/docs 用 VitePress 构建并部署到 Vercel，release.yaml 有对应 VERCEL 项目 ID）。
- 客观背景：Fuel Labs 是商业公司，本 SDK 与 fuel-core、forc、fuels-rs、wallet 等组成官方工具链矩阵；星数（约 4.3 万）在同类区块链 SDK 中属于头部水平。

## 9. 亮点与风险

亮点：

1. 架构完整度高：从 GraphQL schema 维护、codegen、ABI 编解码、typegen 到 CLI/脚手架全链路自研闭环，开发者从 `npm create fuels` 到主网部署无需离开 SDK 生态（`README.md`、`packages/create-fuels/`）。
2. 工程质量标杆级：双环境 vitest 矩阵 + 84 个真实 Sway 测试工程的集成测试、零警告 eslint、coverage diff、knip、changesets+provenance 发布（第 7 节所列文件）。
3. 类型安全深入：typegen 产物 + v0/v1 双编码版本兼容 + 17 个类型 coder，把 Sway 类型系统完整映射进 TS（`packages/abi-coder/src/encoding/`）。
4. 钱包连接标准化：`FuelConnector` 抽象（`packages/account/src/connectors/fuel-connector.ts`）为浏览器钱包插件生态提供了统一接口，类似以太坊 EIP-1193 的角色。
5. 与 fuels-rs 同构：同一套心智模型在 Rust/TS 两侧复用，团队与文档可迁移性好（README 官方文档区）。

风险/局限：

1. 0.x 阶段：版本号 0.104.0，changelog 中 `chore(deps)!: update Fuel-owned dependencies and toolchains` 等破坏性变更频繁（`packages/fuels/CHANGELOG.md`），升级需跟着 fuel-core/Forc 版本走。
2. 工具链耦合强：SDK 行为绑定特定 fuel-core/Forc 版本（`packages/versions` 校验 + CHANGELOG 中 gas 表 V4/V7 迁移记录），跨版本兼容是长期维护负担。
3. 生态依附性：价值高度绑定 Fuel L2 的采用情况；若生态收缩，SDK 星数与实用性会脱节（客观推断，非代码证据）。
4. monorepo 体量：21 个包 + turbo/codegen/knip 多套工具，普通贡献者的上手成本不低（根 `package.json`、`pnpm-workspace.yaml`）。

## 10. 总结评价

fuels-ts 是一个把"区块链 SDK 该怎么做"回答得非常完整的 TypeScript 工程：GraphQL Provider、ABI 编解码、类型生成、钱包/谓词/连接器、CLI 与脚手架分层清晰、互为依赖、整体由测试与 CI 高强度托底；它与 fuels-rs 构成 Fuel 官方 SDK 的双语言实现，代码风格上又能看出 ethers.js 传统的清晰延续。适合三类人：要在 Fuel 上构建 dApp 的前端/全栈开发者（直接 `npm install fuels`）；想在 TS 里研究链上抽象（账户抽象谓词、UTXO 资源管理、交易生命周期）的工程师；以及想学习"如何工程化地维护一个 20+ 包的 pnpm monorepo"的团队——它的测试组织与发布流水线本身就是很好的范本。读源码建议从 `packages/fuels/src/index.ts` 进入，按 account/providers → abi-coder → program/contract 的顺序自底向上看。
