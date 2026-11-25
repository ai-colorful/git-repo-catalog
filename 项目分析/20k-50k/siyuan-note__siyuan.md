# siyuan-note/siyuan 项目分析

> 星数 37000（快照 2026-10-24）｜主语言 TypeScript (Go)｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：思源笔记是一款本地优先的个人知识管理系统，以"块"为一等公民，支持细粒度块级引用/双链与 Markdown 所见即所得编辑。

| 项 | 值 |
|---|---|
| 名称 | siyuan-note/siyuan |
| 星数 | 37000（快照 2026-10-24；分析当日 gh api 实测 47082，增长迅速） |
| 主语言 | TypeScript（GitHub 统计）+ Go（kernel 后端） |
| 许可证 | AGPL-3.0（LICENSE 文件；README 徽章一致） |
| 当前版本 | v3.8.9（app/package.json `version: 3.8.9`） |
| 最近活跃 | master 最近推送 2026-11-04（gh api pushed_at），最新 release v3.8.10-alpha.5（2026-11-03） |
| 创建时间 | 2020-08-30（gh api created_at） |
| 维护方 | 云南链滴科技（Yunnan Liandi Technology Co., Ltd.，app/package.json author），核心维护者 88250（Liang Ding）与 Vanessa（Liyuan Li） |

## 2. 项目是什么，解决什么问题

思源笔记解决"块级颗粒度的知识管理"问题。传统 Markdown 笔记以文件为单位组织，思源把每个段落、列表项、表格行等拆成带全局唯一 ID 的块（block），块与块之间可建立引用（block ref），反向自动生成反链（backlink），并支持查询嵌入（query embed），从而实现细粒度的双链知识网络（README.md "Features" 与 docs/SY-FORMAT.md）。

它坚持"本地优先"：所有笔记以 `.sy` JSON 文件形式保存在用户本地工作区（workspace）目录，不强制依赖云端；同时提供官方云同步（会员功能）与第三方对象存储/S3/WebDAV 同步通道。数据格式为 Lute 引擎 AST 的 JSON 序列化，官方专门维护了格式规范文档 docs/SY-FORMAT.md（含 Spec 版本 2-6 的兼容规则），这是其"数据可迁移、可被第三方读写"承诺的直接证据。

编辑器层面，思源自研了一套所见即所得方案：前端 protyle 编辑器直接渲染块 DOM，所有编辑操作以"事务（transaction）"形式通过 HTTP API 提交到 Go kernel，由 kernel 重写 `.sy` 文件并同步重建两套索引（块树索引 blocktree.db 与全文检索索引 siyuan.db，见 docs/SY-FORMAT.md 第 0.5 节），保证"所见即所得"与"底层 Markdown/JSON 真值"的一致性。

工程形态为双端架构：Go kernel 是能力中枢（HTTP API、存储、索引、同步、AI、CLI、MCP），Electron 壳负责桌面打包，浏览器/移动端直接连 kernel 的 6806 端口。kernel 本身被编译为各平台二进制（含 Docker 服务端部署），Android/iOS/HarmonyOS 为独立仓库复用同一 kernel（README.md "Architecture and Ecosystem" 表格）。

商业化上，仓库明确"Most features are free, even for commercial use"，付费会员 mainly 涉及官方云同步/云存储、AI 配额等（README.md FAQ "Do I need to pay for it?"）；开源核心与订阅功能的代码边界在源码中可清晰定位（见第 5 节）。

## 3. 技术栈与依赖

后端（kernel/，Go 1.26，kernel/go.mod）：
- Web 框架：gin v1.12（含 gin-contrib/gzip、sessions、sse），WebSocket 用 gorilla/websocket 与 lxzan/gws
- 存储：mattn/go-sqlite3（CGO，SQLite 全文检索 FTS5）、ristretto（内存缓存）
- Markdown/AST：88250/lute（自研编辑器引擎，独立仓库）
- 同步体系：siyuan-note/dejavu（数据仓库快照同步）、siyuan-note/encryption（加密笔记本）、siyuan-note/filelock、siyuan-note/logging 等自家组件库（go.mod 第 77-84 行）
- AI/Agent：modelcontextprotocol/go-sdk（MCP 服务端）、dop251/goja_nodejs（JS 运行时）、OpenAI/Anthropic 客户端自研封装（kernel/agent/）
- 内容解析：docconv、pdfium（klippa-app）、goquery、bluemonday 等，共 228 个模块（go.mod require 计数）

前端（app/，TypeScript，app/package.json）：
- Electron 44.10.0 + Node 24 + pnpm 12.6
- 构建：webpack 多配置（webpack.config.js / desktop / mobile / export 四套入口）
- 图标/图形库、dayjs 等常规依赖；无重型前端框架，UI 以原生 DOM + 自研 protyle 为主
- 打包：electron-builder 各平台 yml（含 arm64 变体）、NSIS（Windows）、appx（微软商店）

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| kernel/ | Go 后端核心，约 52 万行、2377 个 .go 文件（实测统计） |
| kernel/api/ | HTTP API 层，337 文件 6.6 万行；router.go 约 712 条路由注册 |
| kernel/model/ | 业务逻辑层，822 文件 22.3 万行（backlink/sync/export/repository 等） |
| kernel/sql/ | SQLite 索引层，102 文件 2.1 万行（blocks FTS5、refs、av 等） |
| kernel/treenode/ | AST 树与块结构工具（.sy 序列化、块树、标题层级） |
| kernel/av/ | 数据库/属性视图（Attribute View），93 文件 2.5 万行 |
| kernel/bazaar/ | 集市（插件/主题/图标/模板安装与元数据） |
| kernel/mcp/、kernel/cli/、kernel/agent/ | MCP 服务端、内置 CLI、AI Agent 运行时 |
| app/src/ | TypeScript 前端，2053 文件约 45 万行；protyle 编辑器占 925 文件 |
| app/electron/ | Electron 主进程（连接管理、窗口、崩溃恢复等，含测试） |
| docs/ | 32 份英文/中文工程规范文档（SY-FORMAT、WORKSPACE、API、PLUGIN-PUBLISH 等） |
| scripts/ | 构建发布、OCR 打包、翻译校验等脚本（含大量脚本自身的测试） |
| .github/workflows/ | cd.yml（发布）、desktop-build.yml、dockerimage.yml 等 7 个工作流 |

组织方式评述：这是"前后端同仓但分层严格"的单体仓库——kernel 与 app 互不 import，仅以 HTTP/WebSocket 契约通信；契约甚至有独立校验（app/package.json 的 `api:generate`/`api:check` 脚本与 docs/API-CONTRACTS.md）。生态拆分为约 10 个姊妹仓库（lute、dejavu、petal、bazaar、移动端等），主仓聚焦 UI + kernel。

## 5. 核心架构与关键实现

### 5.1 HTTP API 与路由（kernel/api/router.go）

kernel/main.go 仅作 CLI 入口转发到 kernel/cli/cmd；服务端由 kernel/server/serve.go 启动 gin 引擎（默认端口 6806），api/router.go 注册约 712 条 POST 路由，按域分组（block/search/filetree/format/transaction 等），并大量采用"契约路由"包装：

```go
ginServer.Handle("POST", "/api/lute/spinBlockDOM",
    contractRouteHandlers(apicontract.SpinBlockDOM, spinBlockDOM)...)
```

`contractRouteHandlers` 将 handler 与 apicontract 声明绑定（kernel/apicontract/ 包），配合 docs/API-CONTRACTS.md 与前端 `api:check` 校验，保证 700+ 接口的请求/响应结构不被意外破坏——这是本仓库质量工程的一个特色。

### 5.2 .sy 存储格式与块引擎（kernel/treenode/、kernel/filesys/tree.go）

`.sy` 文件 = Lute AST 的 JSON 序列化，根节点为 `NodeDocument`，子节点递归 `Children`（docs/SY-FORMAT.md）。每个节点携带全局唯一 ID（时间戳-随机段）与 IAL 属性（id/title/updated/custom-*），NodeHash 用 SHA-256 前 7 位做内容指纹（kernel/treenode/tree.go `NodeHashWithHeadingParent`）。落盘由 kernel/filesys/tree.go:555 `WriteTreeWithPreflight` 统一处理：文档级锁 → 序列化 → 加密租约（`acquireCryptoLease`，加密笔记本按 notebook DEK 加密为密文 encData）→ 写入。SQLite 索引双库并存：blocktree.db（块 ID→文件路径，供引用与面包屑）与 siyuan.db（blocks FTS5 全文检索，kernel/sql/block.go 定义 `Block` 结构含 ID/ParentID/RootID/Box/Content/Markdown/IAL 等字段）。

### 5.3 块级引用与反链（kernel/model/backlink.go、kernel/sql/block_ref.go）

引用解析发生在 kernel 侧：文本中的 `((blockID "content"))` 语法经 lute 解析后写入 refs 表，反链查询基于 `sql.QueryRefsByDefID`。kernel/model/backlink.go 的刷新链路体现了异步化设计：

```go
func refreshRefsByDefID(defID string) {
    refs := sql.QueryRefsByDefID(defID, true)   // 全局查 + 加密笔记本 fallback
    ...
    for _, ref := range refs {
        task.AppendAsyncTaskWithDelay(task.SetDefRefCount,
            util.SQLFlushInterval, refreshRefCount, ref.DefBlockID)
    }
    trees := filesys.LoadTrees(rootIDs)
    for _, tree := range trees {
        sql.UpdateRefsTreeQueue(tree)           // 引用树重建走队列
        ...
    }
}
```

引用变更不直接同步刷新，而是进 task 队列按 SQLFlushInterval 延迟合并执行（kernel/task/），这是思源能支撑"百万字大文档编辑"的关键取舍之一（README.md 功能列表称支持百万字文档）。

### 5.4 同步体系与加密（kernel/model/sync*.go、siyuan-note/dejavu）

同步不自行实现传输层，而是把快照/增量/冲突处理委托给自家 dejavu 库（go.mod 引入，kernel/model/sync.go 第 25-26 行导入 dejavu 与 dejavu/cloud）。kernel/model/sync.go 负责编排：请求去重锁（lockSyncRequest）、云锁错误映射（ErrLockCloudFailed/ErrCloudLocked/ErrRepoFatal）、WebSocket 长连接实时推送（`GetCloudWebSocketServer() + "/apis/siyuan/dejavu/ws"`，sync.go:1309）。加密笔记本在存储层钩子实现：kernel/filesys/crypto_hook.go 校验 `.sy` 后缀并按笔记本派生 DEK 加密落盘（docs/ENCRYPTED-NOTEBOOK.md 亦有规范文档）。

### 5.5 插件与集市机制（kernel/bazaar/、app/src/plugin/）

集市分两层：kernel/bazaar/ 负责包管理（pkgType 分 plugins/themes/icons/templates/widgets；`GetBazaarPackages` 从官方 stage 索引 + bazaar 统计 + 评分聚合出包列表，bazaar/bazaar.go 第 21-38 行；install.go 处理下载、暂存、重命名与版本回退，Windows 特化 rename 逻辑单独成文件）；前端 app/src/plugin/ 是插件运行时（即 petal API 的实现处）：EventBus、命令面板适配（commandAdapter.ts）、Protyle 方法注入（ProtyleMethod.ts）、生命周期与卸载（lifecycle.ts/uninstall.ts）等 40+ 模块，配套 40+ 个测试文件。插件本质是前端 JS 包 + kernel API 调用，发布流程见 docs/PLUGIN-PUBLISH.md。

### 5.6 开源核心 vs 订阅功能边界

付费门禁集中在 kernel 侧一个判定函数（kernel/model/conf.go:1197）：

```go
func IsSubscriber() bool {
    u := Conf.GetUser()
    return nil != u && (-1 == u.UserSiYuanProExpireTime || 0 < u.UserSiYuanProExpireTime)
        && 0 == u.UserSiYuanSubscriptionStatus
}
```

源码内 `IsSubscriber()`/`IsPaidUser()` 调用点即为订阅边界，实测包括：第三方云同步（S3/WebDAV provider，kernel/model/repository.go:1600 起 4 处）、资源文件云端下载（model/asset_download.go:107）、AI 相关（model/blockial.go:29 等）、云服务配额（model/cloud_service.go:289）。门禁逻辑在客户端实现（信任云账户校验），本地核心功能（编辑/反链/本地索引/CLI/MCP）完全不受影响——与 README "Most features are free" 的声明一致。

### 5.7 新一代能力：CLI、MCP、AI Agent

README 展示了内置 CLI（kernel/cli/cmd/ 下 notebook/block/search/export 等 20+ 命令组，复用 kernel model 层）；kernel/mcp/server.go 基于 modelcontextprotocol/go-sdk 暴露 MCP 工具供 LLM 调用；kernel/agent/ 是完整 Agent 运行时（capability/compaction/decision/anthropic 客户端），对应 package.json slogan "From thought to insight, with agents"。三者与 HTTP API 并列为官方认可的四种数据访问路径（docs/SY-FORMAT.md 第 0.5 节表格）。

## 6. 构建与运行

来自 .github/CONTRIBUTING.md 与 app/package.json（未实际运行，仅静态摘录）：

```bash
# 依赖：Go 1.26（CGO_ENABLED=1）、Node 24、pnpm 12.x、Electron 44
# 前端
cd app
pnpm install electron@44.10.0 -D
pnpm run install:electron      # 中国大陆设置 ELECTRON_MIRROR
pnpm run dev                   # webpack 开发构建
pnpm run start                 # 启动 Electron（需先手动启动 kernel）
pnpm run dist                  # electron-builder 打包

# kernel（Go）
CGO_ENABLED=1 go build -tags fts5 -o SiYuan-Kernel  # Windows 下另有 scripts/win-build.bat

# 测试
pnpm run test                  # node --test + tsx，src/tests/electron/scripts 四处 *.test
pnpm run lint                  # tsc 类型检查 + eslint + css/menu 契约检查
go test ./...                  # kernel Go 测试（scripts/test-kernel.mjs 编排）

# Docker 服务端
docker run b3log/siyuan serve --workspace=... --accessAuthCode=...（-p 6806）
```

## 7. 测试与工程质量

- 后端：kernel/ 内 1303 个 `*_test.go`（占全部 Go 文件一半以上），覆盖单元测试与大量集成场景（如 serve_auth_test.go、asset_authorization_test.go 专注鉴权），部分测试依赖 testdata/
- 前端：app/ 下 1002 个 .test.ts/.test.js（src/protyle 与 plugin 模块几乎每个文件配一个测试），electron 主进程 35 个测试；运行器为 node:test + tsx（app/scripts/test.mjs，按 CPU 核数限 4 并行）
- 契约测试：apicontract 契约路由 + api-contracts.mjs 校验 + docs/API-CONTRACTS.md 文档三件套
- CI：.github/workflows/ 下 cd.yml（tag 与 package.json 版本一致性校验、多平台产物）、desktop-build.yml（Windows/macOS/Linux 矩阵含 arm64）、dockerimage.yml、自动 AUR 发布等
- 特色质量脚本：scripts/ 内 check-lang-keys.py（翻译键校验）、check-license-headers.mjs、verify-release.py（发布产物校验），且这些脚本自身带 test_* 测试

## 8. 社区与维护状态

- 活跃度极高：master 最近推送 2026-11-04（分析前一日），2026-10-29 发布 v3.8.9 正式版后连续产出 v3.8.10-alpha.1~5（每日 alpha），release 节奏为"alpha 快速迭代 + 定期 stable"
- 贡献者 134 人（gh api contributors 分页实测），但实际提交高度集中于 88250 与 Vanessa 两位核心维护者（package.json maintainers；contributor 图谱为佐证，未逐 commit 核实）
- 商业实体支撑：云南链滴科技提供订阅收入（pricing 页），这是单人/双人仓库能维持 50 万行代码持续重构的资源基础
- 社区生态：官方论坛 liuyun.io、集市仓库 siyuan-note/bazaar、awesome 清单；Docker 镜像 b3log/siyuan 有 K8s/Unraid/TrueNAS 第三方部署文档沉淀（README）
- 议题管理：open_issues 仅 16（gh api），大量 issue 通过 issue 模板与 lock/unlock 工作流治理

## 9. 亮点与风险

亮点：
1. 数据格式主权意识强：.sy 格式、工作区布局、API 契约均有官方规范文档（docs/SY-FORMAT.md、WORKSPACE.md），并给出四种官方数据访问路径的优先级指引，极少见
2. 架构分层干净：kernel（Go）/app（TS）单向依赖自家组件库（lute/dejavu/filelock 等），契约路由 + 契约文档 + 前端校验构成三层接口防回归
3. 测试密度罕见地高：2300+ 测试文件对 4400+ 源文件，且 CI/脚本/契约均有测试文化
4. 多端复用一套 kernel：桌面/移动/Docker/浏览器全走同一 HTTP API，本地优先与云端部署天然统一
5. 订阅边界透明：IsSubscriber 调用点即完整商业功能清单，核心功能开源可自托管

风险/局限：
1. 单点集中：52 万行 Go + 45 万行 TS 由 2 人主导维护，bus factor 低；AGPL-3.0 也限制企业对云服务衍生场景的采用
2. 信任模型的客户端门禁：IsSubscriber 在客户端判定（conf.go），盗版破解空间与官方云账户接口强耦合；第三方同步通道对会员开放，社区存在免费替代实现的合规争议风险
3. 复杂度外溢：AST 直接序列化为 .sy 且 Spec 版本演进（2-6）+ 兼容读取规则，第三方写入器极易破坏索引（官方文档亦反复警告），数据损坏类 issue 处理成本高
4. 依赖自家生态深：lute/dejavu/riff 均为同作者仓库，社区外部贡献者改核心链路（编辑引擎/同步）门槛高

## 10. 总结评价

思源笔记是国产开源应用中工程质量与产品完成度最高的一档：以 Lute AST + .sy JSON 为真值、以 Go kernel 为唯一能力中枢、以事务化编辑 + 双索引异步重建支撑所见即所得，架构自洽且文档覆盖度罕见（连存储格式字节级规范都维护了中英双语版本）。它证明了"本地优先 + 块级双链 + 开源核心/云服务商业化"路线在 37k+ 星规模上的可行性。适合个人知识管理者直接使用（免费功能即覆盖日常），适合想研究"所见即所得编辑器如何与结构化存储达成一致""本地优先应用如何做同步与加密"的工程师阅读源码；想参与贡献则需接受其高度自研的组件栈与两人主导的协作节奏。
