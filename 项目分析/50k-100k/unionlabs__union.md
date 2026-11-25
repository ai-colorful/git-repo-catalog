# unionlabs/union 项目分析

> 星数 73734（快照 2026-10-24）｜主语言 Rust｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：Union 是一个以零知识证明为核心信任机制的通用跨链协议（ZK IBC bridging），让 Cosmos、Ethereum 及其 L2、Sui、Starknet 等异构链之间做无需多签/预言机/MPC 信任方的通用消息传递与资产转移。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 仓库 | unionlabs/union | gh api 元数据 |
| 星数 | 73,728（API 实时） / 73,734（清单快照） | gh api 2026-10-31 |
| 主语言 | Rust（含 Go/Solidity/TypeScript/Cairo/Move 多语言单体仓） | gh api + 顶层目录 |
| 许可证 | Apache-2.0（根目录另有 LICENSE-MIT） | gh api；LICENSE-APACHE、LICENSE-MIT |
| 默认分支 | main，最后推送 2026-07-25 | gh api 元数据 |
| Release 数 | 30 | gh api /releases |
| 贡献者 | 37（per_page=100 含匿名） | gh api /contributors |
| 规模 | Cargo 工作区 254 个成员 crate；Rust 文件约 1324 个（不含 site/docs） | Cargo.toml；find 统计 |

## 2. 项目是什么，解决什么问题

传统跨链桥依赖多签、MPC 委员会或预言机网络来"证明另一条链上发生了什么"，这些前提本身就是被攻击与作恶的中心点。Union 的做法是把跨链证明问题转化为密码学问题：用零知识证明（Gnark/Groth16，BN254 曲线，利用 EVM 预编译）压缩并验证对手链的共识，使得验证成本与验证者数量解耦。README.md 明确表述："based on Consensus Verification and has no dependencies on trusted third parties, oracles, multi-signatures or MPC"（README.md）。

协议核心由三部分组成（README.md Components 表、ARCHITECTURE.md）：
1. **Union 链本体**（`uniond/`）：基于 Cosmos SDK + CometBLS 的 PoS 链。CometBLS 是 CometBFT 的改造版，把验证者聚合签名换成 BLS12-377 聚合签名，使整个链的共识状态可以被 ZK 电路高效证明（`11-cometbls/` 为其 IBC 侧 Go 模块，含 `zk_verifier.go`）。
2. **ZK 证明服务**（`galoisd/`）：Go + Gnark 实现的 gRPC 服务，为 CometBLS 区块头生成/验证 Groth16 证明，电路对 2^n 个验证者数量泛化（galoisd/README.md）。
3. **Relayer**（`voyager/`）：Rust 实现的模块化跨链中继器，只做数据搬运，不做信任假设。

在链上，两端各自部署轻客户端体系：EVM 侧是 Solidity 实现的 IBC（`evm/contracts/core/` 按 ICS-02/03/04/24/25 分层，`clients/CometblsClient.sol` 验证 CometBLS ZK 证明），Cosmos 侧是 CosmWasm 轻客户端族（`cosmwasm/lightclient/`，覆盖 ethereum/cometbls/arbitrum/base/berachain/bob/parlia/gno/sui/starknet 等 19 个目录）。应用层则是 UCS03-ZKGM 通用消息协议（`cosmwasm/app/ucs03-zkgm/`），支持代币转移、任意合约调用、指令批量与转发。

支持的链（README.md Supported Chains 表）：Ethereum、Arbitrum、Base、Berachain、BSC、Bob、Corn、Babylon、Osmosis、Sei、Sui、Xion 及 Union 自身，主网+测试网共 20+ 条。

## 3. 技术栈与依赖

- **Rust**（主体，约 11.7 万行）：Cargo workspace 254 成员（Cargo.toml）。关键依赖：`tokio 1.45`（异步运行时）、`alloy 1.0 / alloy-primitives 1.1`（EVM 类型与 ABI）、`cosmwasm-std 2.2`（CosmWasm 合约）、自研 `ssz/ssz-derive`（ Beacon SSZ 序列化，lib/ssz/）、自研 `unionlabs`/`unionlabs-primitives`（全仓统一类型层，lib/unionlabs/）、`pg-queue`（Postgres 任务队列，lib/pg-queue/）。
- **Go**（约 73 文件 1.66 万行）：`uniond`（Cosmos SDK 节点，uniond/go.mod）、`galoisd`（Gnark ZK 证明服务，galoisd/go.mod）、`11-cometbls`（CometBLS IBC 模块）。
- **Solidity**（83 个 .sol，evm/）：IBC core（02-client/03-connection/04-channel/24-host/25-handler）、CometblsClient 等 6 类轻客户端、UCS 应用（evm/contracts/）。
- **TypeScript**：pnpm workspace 11 个包（pnpm-workspace.yaml）——app2（app.union.build，Svelte）、site（union.build，Astro）、ts-sdk/ts-sdk-cosmos/ts-sdk-evm/ts-sdk-sui（多端 SDK）、docs（docs.union.build）、effect-svelte、sentinel2、ceremony、zkgm-dev。
- **Cairo**（28 个 .cairo，cairo/）：Starknet 侧 CometBLS 轻客户端与 zkgm 合约。
- **Move**：`aptos/`、`sui/` 提供 Aptos/Sui 侧 IBC 模块；`lib/move-bindgen-derive` 做类型绑定生成。
- **Nix**：整个仓库用 flake.nix 做可复现构建与 dev shell，CI 用 garnix.io（flake.nix、garnix.yaml、README.md Quickstart）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `uniond/` | Union 链节点（Go/Cosmos SDK），app/、x/staking/、proto/ |
| `unionvisor/` | Rust 节点监督器，管理 uniond 升级生命周期（unionvisor/README.md） |
| `11-cometbls/` | CometBLS 共识的 IBC Go 模块与 ZK 验证器 |
| `galoisd/` | ZK 共识证明服务（Go+Gnark，gRPC） |
| `voyager/` | Rust 跨链中继器；modules/（client/finality/proof/state 四类模块）+ plugins/（14 个插件：event-source、packet-batch、client-update、transaction 等，voyager/modules、voyager/plugins） |
| `lib/` | 约 90 个 Rust 基础 crate：unionlabs 类型库、各链 light-client-types、verifier、client（beacon-api、cometbft-rpc、gno-rpc 等）、voyager-vm、pg-queue、ics23、ssz |
| `cosmwasm/` | CosmWasm 合约栈：app/ucs03-zkgm（核心消息协议）、lightclient/（19 个轻客户端）、lst（流动性质押）、multicall、upgradable/pausable/gatekeeper 等治理组件 |
| `evm/` | Solidity IBC 实现与 UCS 应用，含 ARCHITECTURE.md、27 个 Foundry 测试文件（evm/tests） |
| `cairo/`、`aptos/`、`sui/` | Starknet/Aptos/Sui 侧合约 |
| `ts-sdk*`、`typescript-sdk/` | 面向开发者的多端 TypeScript SDK |
| `app2/`、`site/`、`docs/` | 前端应用、官网、文档站（Svelte/Astro，pnpm 包） |
| `mpc/`、`ceremony/` | Groth16 SRS 可信设置 MPC 仪式工具（mpc/README.md） |
| `e2e/`、`networks/`、`devnet-compose/` | Nix 化端到端测试与本地多链 devnet 编排 |
| `generated/`、`deployments/`、`versions/` | protobuf 生成物、链 ID/合约部署登记、版本登记 |

组织方式评述：典型的超大多语言 monorepo。Rust 侧用 workspace 成员显式枚举（注释说明因 Nix 无法读取 `cargo metadata`，Cargo.toml）；TS 侧用 pnpm workspace；构建与 CI 全部收敛到 Nix flake + garnix。一个显著特征是"每种链 × 每层职责"都拆成独立小 crate（light-client-types / verifier / client 三件套），边界清晰、利于增量编译与按需复用。

## 5. 核心架构与关键实现

### 5.1 UCS03-ZKGM：通用跨链消息协议（CosmWasm 合约）

`cosmwasm/app/ucs03-zkgm/`（contract.rs 3556 行 + tests.rs 4137 行）。数据包是指令树，四种操作码（src/com.rs）：

```rust
pub const OP_FORWARD: u8 = 0x00;
pub const OP_CALL: u8 = 0x01;
pub const OP_BATCH: u8 = 0x02;
pub const OP_TOKEN_ORDER: u8 = 0x03;
```

指令带版本号（INSTR_VERSION_0/1/2），TokenOrderV2 用 `kind`（INITIALIZE/ESCROW/UNESCROW/SOLVE）区分代币转移语义（src/com.rs 的 alloy_sol_types::sol! 定义）。`execute()` 入口强制仅 IBC host 可调用，OnRecvPacket/OnTimeoutPacket/OnAcknowledgementPacket 分发（contract.rs:125）。安全设计上可见三处细节：`packet execution reentrancy not allowed`（重入锁，lib.rs:52）、`token_bucket.rs` 实现通道级限速（rate limit，msg.rs RestrictedExecuteMsg::SetBucketConfig）、salt 含 FORWARD_SALT_MAGIC 用于转发包去重（com.rs:26-29）。路径（path）用 U256 位编码 channel 树，支持 forward 多跳路由（contract.rs:3355-3430 的 path/salt 推导函数族）。

### 5.2 Voyager：以 Postgres 为唯一状态的"中继虚拟机"

`voyager/` + `lib/voyager-vm/` + `lib/pg-queue/`。设计文档（voyager/README.md）明确三大目标：速度、数据完整性、秒级重启。做法是把中继的一切——链查询、交易提交、证明生成——都建模为有限状态机的消息，全部落 Postgres：

```rust
pub trait Queue<T: QueueMessage>: Debug + Clone + Send + Sync + Sized + 'static {
    fn enqueue<'a, Filter>(
        &'a self,
        item: Op<T>,
        filter: &'a Filter,
    ) -> impl Future<Output = Result<EnqueueResult, Self::Error>> + Send + 'a
    where ...
```

（lib/voyager-vm/src/lib.rs）消息入队前先跑纯函数优化 pass（`pass.rs`）， Voyager 本体无内存状态，崩溃后原地重启；每条消息自包含全部上下文，可安全并行。链适配层是 modules（client/finality/proof/state，按链提供四种能力，voyager/modules/）+ plugins（事件抓取、打包、超时、客户端更新等 14 个，voyager/plugins/），共 60 个 Cargo.toml 成员，扩展点清晰。

### 5.3 轻客户端三件套与类型驱动架构

对每条接入链，仓库给出三层 crate：`*-light-client-types`（该链状态/共识类型，含 SSZ/MPT 编码）、`*-verifier`（纯函数状态验证：base-verifier、arbitrum-verifier、cometbls-groth16-verifier、ethereum-sync-protocol 等，lib/）、`cosmwasm/lightclient/<chain>`（链上合约实例，19 个目录）。L2 客户端是"条件客户端"：先在 L1 验证 rollup 合约的 account root，再验证 L2 状态、最后比对 IBC account root（voyager/README.md Light Clients 节）。类型层由 `lib/unionlabs`（420 行 lib.rs + 按链子模块）与 `subset-of/subset-of-derive` 等宏 crate 支撑，实现 Rust↔Solidity↔Cairo↔Move 类型同源。

### 5.4 CometBLS + galoisd 的 ZK 共识证明链路

uniond 使用的 CometBLS 把 CometBFT 的 ed25519 多签改为 BLS12-377 聚合签名（README.md `[cometbls]` 链接、11-cometbls/zk_verifier.go）。galoisd 用 Gnark 电路对该聚合签名做 Groth16 证明，电路"generic over 2^n validators，只证 (非)相邻区块转移"（galoisd/README.md Circuit 节），EVM 侧靠 BN254 预编译在 `evm/contracts/clients/CometblsClient.sol` 内廉价验证。SRS 参数通过 `mpc/` 的多方仪式生成（Supabase 状态机 + 协调器 + 客户端，mpc/README.md）。

## 6. 构建与运行

构建体系完全 Nix 化（README.md、ARCHITECTURE.md）：

```sh
# 可复现构建任意组件
nix build .#uniond -L
nix build .#voyager -L
nix build .#app -L
nix flake show          # 查看全部包
nix develop             # 含 cargo/rustc/node/go 的 dev shell
nix run .#pre-commit -L # 全仓格式化 + 拼写检查
```

其他命令（来自各组件文档与工程文件）：
- galoisd：`nix run .#galoisd -- --help`，生产用 `docker pull ghcr.io/unionlabs/galoisd:<VERSION>`（galoisd/README.md）
- EVM 测试：Foundry（foundry.toml，evm/tests/ 下 27 个 .sol 测试）
- e2e：`e2e/all-tests.nix`、`e2e/union-test.nix` 等 Nix 检查项（含 devnet-eth-runs、sepolia-runs、upgrade-from-genesis，见 garnix.yaml）
- TS：pnpm workspace（pnpm-workspace.yaml），Vitest（vitest.workspace.ts）

注意：多数组件仅 Linux 可构建，macOS 用户被建议跑 NixOS VM（README.md）。

## 7. 测试与工程质量

- **Rust 单元测试**：lib/voyager/cosmwasm/unionvisor 中 `#[test]` 属性约 2683 处（grep 统计）；zkgm 合约 tests.rs 达 4137 行，覆盖各 opcode×版本路径（cosmwasm/app/ucs03-zkgm/src/tests.rs）。
- **SSZ 合规测试生成器**：`lib/ssz/tests-generator`，且 garnix.yaml 有 `ssz-tests-up-to-date` 检查项，保证生成测试不漂移。
- **E2E/集成**：Nix 化整链测试——真起 devnet 跑跨链（e2e/union-test.nix、garnix.yaml 中 devnet-eth-runs/sepolia-runs/epoch-completes/upgrade-from-genesis 等检查名）；nightly.yml、nightly-e2e-lst.yml 做夜间回归。
- **CI**：GitHub Actions 13 个 workflow（check/e2e/nightly/package-release/release-component/deploy-*），PR 级 check 含 LFS 校验与 commitlint（.github/workflows/check.yml）；重型构建由 garnix.io 按 x86_64+aarch64 双架构执行（garnix.yaml）。
- **代码质量**：rustfmt.toml、clippy.toml、dprint.json、biome.json、taplo.toml、typos.toml 全套格式化/lint 配置在根目录；voyager-vm 开启 `clippy::pedantic` 并对非测试代码 `warn(clippy::unwrap_used)`（lib/voyager-vm/src/lib.rs 头部）。CODEOWNERS、VERSIONING.md、commitlint 齐备。
- **文档**：每个重要组件有 README；根 ARCHITECTURE.md、evm/ARCHITECTURE.md、voyager/CONCEPTS.md、voyager/doc/。

## 8. 社区与维护状态

- 最后推送 2026-07-25（gh api），tarball 内文件 mtime 与 .changeset 目录（brave-points-wash.md）表明近期仍在持续发版迭代。
- Release 节奏：30 个 release（gh api /releases），配合 release-component.yml / package-release.yml 做组件级打包；unionvisor + versions/ 目录承担链上升级管理（unionvisor/README.md）。
- 贡献者 37 人（含匿名，per_page=100）；CODEOWNERS 存在但归属结构未逐一核实（未确认）。
- 治理：README 称合约升级、连接与代币配置由去中心化治理控制；仓库内 cosmwasm/gatekeeper、access-manager、upgradable、pausable 等组件与其呼应（README.md、cosmwasm/ 目录）。
- 生态动作可从目录推断：Ceremony（MPC 仪式）、主网 union-1 与测试网 union-testnet-10 在 README 链表内，且 2026 年起扩展 Sui/Starknet/Aptos 客户端（lib/sui-*、cairo/、lib/aptos-types）。

## 9. 亮点与风险

**亮点**
1. 真正"去信任化"的技术路线完整落地：自研共识（CometBLS）→ ZK 电路（galoisd）→ 链上验证（Solidity/CosmWasm 轻客户端）→ 通用消息协议（ZKGM），全链路自洽且全部开源（各组件目录）。
2. Voyager 的 Postgres-FSM 中继设计把"崩溃恢复、事务完整性、并行度"三个 relayer 难题一并解决，工程观点成熟（voyager/README.md、lib/voyager-vm）。
3. 类型驱动的多链适配架构：每链 types/verifier/contract 三件套 + unionlabs 统一类型层，新增链的接入成本被压到模板化（lib/ 与 cosmwasm/lightclient/ 的目录规律）。
4. 工程化标杆级：Nix 全量可复现构建、garnix 双架构 CI、2000+ Rust 测试、SSZ 合规测试自动生成、全套 lint 配置。
5. ZKGM 指令树（FORWARD/BATCH/CALL/TOKEN_ORDER + 版本化）设计表达能力远超普通 token bridge，可组合出多跳路由与意图型跨链订单（cosmwasm/app/ucs03-zkgm/src/com.rs）。

**风险/局限**
1. 复杂度与审计面极大：254 个 Rust crate + 5 种语言 + 自研共识 fork，安全论证依赖审计与形式化程度（未确认其公开审计范围）。
2. 构建门槛高：Nix 强绑定、部分组件仅 Linux，普通贡献者上手成本明显高于常规 Cargo/Node 仓库（README.md Quickstart 自述）。
3. 协议尚年轻：ZKGM 指令已出现 V1→V2 迁移逻辑与 rate-limit 运维开关（msg.rs RestrictedExecuteMsg），表明协议语义与经济模型仍在快速演化中。
4. 仓库体积庞大（tarball 解压约 333MB，site/ 资产 222MB），克隆/构建对硬件要求高。

## 10. 总结评价

Union 是"ZK 跨链"方向少见的全栈自研实现：为了一个可被电路证明的共识，宁可 fork CometBFT；为了验证端到端的轻量，把 Solidity/CosmWasm/Cairo/Move 四套轻客户端都写了一遍；再把中继器做成数据库驱动的状态机。它不是套壳 IBC 的桥，而是一整套重新设计的跨链基础设施。7.3 万星与其工程完成度、技术声誉相符。适合跨链协议研究者、ZK 工程师、Rust/Go 系统开发者精读源码；对只想接入跨链的应用开发者，则应从 ts-sdk 与 docs.union.build 入手，而不是直接啃 monorepo。
