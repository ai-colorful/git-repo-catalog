# golang-standards/project-layout 项目分析

> 星数 56707（快照 2026-10-24）｜主语言 Makefile（实际内容 100% Markdown）｜档位 50k-100k｜分析日期 2026-11-03

## 1. 项目速览

一句话定位：Go 社区流传最广的项目目录布局"参考标准"——一个纯文档型仓库，用一组带说明和真实案例链接的空目录，定义了 `/cmd`、`/internal`、`/pkg` 等约定。

| 项目 | 事实 |
| --- | --- |
| 名称 | golang-standards/project-layout（Standard Go Project Layout） |
| 星数 | 56707（快照），gh api 实测 56731（2026-11-03） |
| Fork | 5420；Watch 591；Open Issues 93（gh api，2026-11-03） |
| 主语言 | GitHub 标记 Makefile，实为 26 个根级文件 + 19 个目录 README，约 4560 行 Markdown，零 Go 代码 |
| 许可证 | `LICENSE.md` 全文仅一行 "Hacker license!"，GitHub 识别为 NOASSERTION（非标准许可） |
| 创建时间 | 2017-09-09（gh api） |
| 最近推送 | 2026-04-28（gh api；最后一次提交为合并 README 更新 #193） |
| Release | 无任何 release（gh api 返回 404） |
| 默认分支 | master |
| 内容规模 | 根级 20 份 README（1 份英文主文档 + 19 份译本）+ 19 份目录 README，共 39 份 Markdown 约 4560 行；另含 go.mod/Makefile/.editorconfig 模板与 9 个 .keep 占位文件 |

注：星数、fork、贡献者等元数据均于 2026-11-03 通过 `gh api repos/golang-standards/project-layout` 复核；源码分析基于 HEAD tarball（master 分支，85 个归档条目、54 个文件全量核验）。

## 2. 项目是什么，解决什么问题

Go 上手极简：一个 `main.go` 加 `go.mod` 就能跑。但当项目变大、多人协作、或要开源给别人 import 时，"代码放哪"就成了每个 Go 团队都要回答一次的问题。Go 官方长期只给出零散建议（`internal` 的编译器语义、`cmd` 惯例等），没有一份完整的目录组织参考。本仓库填补的正是这个空白：它把 Go 生态里历史形成、逐渐趋同的目录模式汇总成一页清单，并给出可直接克隆的空骨架（README.md，"Overview" 一节）。

它同时回答三个层次的问题：目录叫什么名字（命名约定）、什么代码放哪里（归属规则，如"main 保持极薄、可复用代码去 /pkg、不想被人复用的去 /internal"，`cmd/README.md`）、为什么这么放（每条约定附理由、争议与真实案例）。

仓库本体就是一个"活的模板"：19 个顶层目录，每个目录里只有一个 README.md 说明该放什么，并链接若干真实大厂项目的对应目录作为佐证（如 `cmd/README.md` 列出 kubernetes、moby、prometheus、go-ethereum 的 `/cmd`）。另有 9 个 `.keep` 占位文件标出示例位置（`cmd/_your_app_/`、`internal/app/_your_app_/`、`pkg/_your_public_lib_/`、`web/{app,static,template}`、`build/{ci,package}`），以及一个填着 `YOUR-USER-OR-ORG-NAME` 占位符的 `go.mod` 模板（`go.mod`）。

它定义的核心约定按用途分组（README.md 第 59-189 行）：Go 代码四件套 `/cmd`（main 入口，只做组装）、`/internal`（编译器强制不可被外部 import 的私有代码，Go 1.4 起生效）、`/pkg`（可供外部 import 的库代码，附带"三思而后放"的警告）、`/vendor`（依赖，Go 1.14 + Modules 时代已多数不需要）；服务与 Web 类的 `/api`（OpenAPI/Swagger/协议定义）、`/web`；通用应用的 `/configs`、`/init`（systemd/supervisord 配置）、`/scripts`、`/build`（打包与 CI）、`/deployments`（k8s/helm/terraform 模板）、`/test`（外部测试与数据）；以及 `/docs`、`/tools`、`/examples`、`/third_party`、`/githooks`、`/assets`、`/website`。还有一节反模式：别在 Go 项目里建 `/src`（Java 习惯，会与 GOPATH 工作区混淆）（README.md，"Directories You Shouldn't Have"）。

关于身份必须澄清：README 开篇即用大写声明 **"`NOT an official standard defined by the core Go dev team`"**，并主动链接 Go 官方的 "Organizing a Go module" 文档（go.dev/doc/modules/layout）——这是对社区长期批评（组织名 "golang-standards" 暗示官方性，Go 团队并不背书）的直接回应。README 同时劝退初学者："如果你在学 Go 或做 PoC，这套布局是 overkill，一个 `main.go` 足够"（README.md 第 31 行）。这种"非官方标准"的争议本身就是它传播力的一部分。

仓库的年龄也写在细节里：`.gitignore` 残留的 `.glide/` 缓存条目（`src/.gitignore`）表明它诞生于 2017 年——Go Modules 出现之前、glide 还是主流依赖工具的时代。九年间内容随生态持续修订：Go Modules 一节明确"除非有特殊理由否则使用 Modules"（README.md 第 33 行），把已废弃的 golint 更新为推荐 staticcheck（README.md 第 39 行），失效链接改用 web.archive.org 快照引用（README.md 第 44 行）——它保持的是"活文档"而非化石。

它也不试图覆盖一切：README 开篇即说明不涉及 Clean Architecture 类更细粒度的包内结构设计（README.md 第 27 行），代码命名与风格问题则交给 gofmt、staticcheck 和 Go 官方 CodeReviewComments 等外部指南（README.md 第 39-44 行）。每个子目录 README 均为自足的固定三段式：一句话职责定义、"放什么/不放什么"的操作指引、案例链接——19 份文档读起来高度一致。

传播策略上也可见刻意经营：README 自链一篇 Medium 深度文章 "Go Project Layout" 作背景阅读（README.md 第 46 行），并罗列 Peter Bourgon、Kat Zien 等 GopherCon 演讲作为权威锚点（README.md 第 49-52 行）——把自身嵌入 Go 社区既有知识网络，而非孤立存在。

## 3. 技术栈与依赖

- 无任何运行时代码、无第三方依赖：仓库不含任何 `.go` 文件、`go.sum`、`package.json` 或 CI 配置（tarball 全量 54 个文件核验）。
- `go.mod`：仅两行的模板文件，`module github.com/YOUR-USER-OR-ORG-NAME/YOUR-REPO-NAME`、`go 1.19`——用途是告诉克隆者"从这里开始"，不是真实工程清单。`go 1.19` 的版本号（2022 年 8 月发布）说明占位 go.mod 自那之后未随版本演进，对模板本身无实质影响，克隆者通常会自行改成最新版本。
- `Makefile`：全文一行注释 `# note: call scripts from /scripts`（`Makefile`），是布局示范的一部分，不是可执行构建系统。
- `.editorconfig`（`src/.editorconfig`）：定义 Go/Makefile 用 tab、Markdown 用 tab 不去尾空格等团队风格约定。
- GitHub 主语言显示 Makefile 属于 linguist 对极小仓库的误判，实际语言构成为 Markdown。

## 4. 目录结构与代码组织

| 目录 | 职责（依据各目录内 README.md） |
| --- | --- |
| `/cmd` | 各主应用入口，目录名=可执行名，main 保持极薄 |
| `/internal` | 私有应用与库代码，编译器强制外部不可 import；可再分 `/internal/app` 与 `/internal/pkg` |
| `/pkg` | 可被外部安全 import 的库代码（社区有争议，README 明示"非普适"） |
| `/vendor` | 依赖目录，Go Modules + proxy 时代多数项目可省 |
| `/api` | OpenAPI/Swagger、JSON schema、协议定义文件 |
| `/web` | Web 应用专属资产：静态文件、服务端模板、SPA |
| `/configs` | 配置文件模板或默认配置（confd/consul-template） |
| `/init` | systemd/upstart/sysv 及 runit/supervisord 配置 |
| `/scripts` | 构建/安装/分析脚本，给根 Makefile 瘦身 |
| `/build` | 打包（/build/package）与 CI（/build/ci）配置 |
| `/deployments` | IaaS/PaaS/容器编排部署模板（docker-compose、k8s/helm、terraform） |
| `/test` | 额外外部测试应用与测试数据 |
| `/docs` | 设计与用户文档（godoc 之外） |
| `/tools` | 项目支撑工具，可 import `/pkg` 与 `/internal` |
| `/examples` | 应用与公共库的示例 |
| `/third_party` | 外部辅助工具、fork 的代码、第三方实用程序 |
| `/githooks`、`/assets`、`/website` | Git 钩子、仓库素材（logo 等）、非 GitHub Pages 的站点数据 |
| `/src` | 唯一被点名的"不该有"的目录（Java 习惯反模式） |

顶层除上述 19 个目录外，根目录仅有主 README 与 19 份译本、LICENSE.md、Makefile、go.mod、.editorconfig、.gitattributes、.gitignore——没有 `.github/` 目录，没有任何代码文件（tarball 文件清单核验）。占位机制方面，9 个 `.keep` 文件标出空目录的示例位置（`cmd/_your_app_/`、`internal/app/_your_app_/`、`internal/pkg/_your_private_lib_/`、`pkg/_your_public_lib_/`、`build/ci/`、`build/package/`、`web/{app,static,template}/`），目录名用下划线包裹，与 Go 工具链忽略 `_` 开头路径的规则一致（该规则在 `test/README.md` 中亦有说明），确保占位目录不会被误当真实包编译。

组织方式评述：这是"文档即仓库"的单体结构——主 README 是唯一权威全文（220 行），19 个子目录 README 各自重复对应章节并附案例链接（如 `pkg/README.md` 长达 112 行、列出约 100 个使用 `/pkg` 的真实项目，从 kubernetes、istio、grafana 到 cilium、kyverno）。主 README 与子 README 内容存在刻意冗余，维护上以主 README 为准。

国际传播靠 19 份根级翻译 README：韩语、简体中文两套（`README_zh.md` 与 `README_zh-CN.md`）、繁体中文、法语、日语、巴西葡语、西班牙语、罗马尼亚语、俄语、土耳其语、意大利语、越南语、乌克兰语、印尼语、印地语、波斯语、白俄罗斯语、孟加拉语（根目录文件清单核验）。翻译由社区各自认领维护，质量与时效参差（详见第 7 章）。README 末尾的徽章一节（README.md 第 200-216 行）同样是给采用者的模板：Go Report Card、GoDoc（已划掉弃用）、Pkg.go.dev、Release 四个徽章都注明"把链接换成你的项目"——连 README 本身也被设计成可复制的样板。

## 5. 核心架构与关键实现

作为无代码仓库，"核心实现"即其内容设计本身，三个关键点：

1. **约定分级与强度标注**。对每个目录明确"约束来源"：`/internal` 是编译器强制的（引用 Go 1.4 release notes，`internal/README.md`），`/pkg` 是纯社会契约——"internal 是更好的保证私有性的方式，因为它由 Go 强制；/pkg 的价值在于显式告诉别人这里可以放心用"（`pkg/README.md`、README.md `/pkg` 节）。对 `vendor` 则给出明确的过时提示：Go 1.13 起默认 module proxy，多数项目不再需要 vendor（README.md 第 99 行）。这种诚实标注让它区别于教条式规范——社区里"每个流行仓库用 /pkg 的同时你能找出十个不用的"，选择权留给项目作者（`pkg/README.md`）。

2. **"克隆后做减法"的使用模型**。README 第 31 行："Clone the repository, keep what you need and delete everything else! Just because it's there it doesn't mean you have to use it all." 配合 `go.mod` 占位模块名与空目录 `.keep` 占位，整个仓库就是一个 fork-and-trim 起步模板，而非需要安装的工具。占位模块名 `YOUR-USER-OR-ORG-NAME/YOUR-REPO-NAME`（`go.mod`）也是这个模型的一部分：克隆者必须改掉它才能编译，这本身就是一次"从此这是你的项目"的提示。

3. **案例驱动的说服结构**。几乎每条约定都附带 3-100 个真实项目链接作证据（`cmd/README.md` 7 个、`internal/README.md` 7 个、`pkg/README.md` 约 100 个），并在主 README 末尾推荐遵循该布局的标杆项目 marmotedu/iam（README.md 第 57 行，即维护者孔令飞本人的项目）。这套"约定+证据+样板"的三件套是它作为文档型仓库的完整"架构"。以 `/cmd` 的原文为例（`cmd/README.md`）：

   > Don't put a lot of code in the application directory. If you think the code can be imported and used in other projects, then it should live in the `/pkg` directory. If the code is not reusable or if you don't want others to reuse it, put that code in the `/internal` directory.

   三句话完成一次归属规则教学——可复用性是唯一的分界线，这正是全文反复强调的判断标准。

4. **配套的工程风格约定**。`.editorconfig` 按文件类型规定缩进（Go/Makefile/go.mod 用 tab，YAML/JSON 用 2 空格，JS/TS/Python 用 4 空格，`src/.editorconfig`），`.gitattributes` 一行 `* -text` 保证跨平台换行一致（`src/.gitattributes`）——克隆者拿到的骨架自带风格基线，不需要再自己配一遍。

## 6. 构建与运行

无可构建、无可运行的内容（全仓库无代码）：

- 使用方式 1（模板）：克隆仓库，保留所需目录并删除其余，改写 `go.mod` 中的模块路径占位符，随后正常 `go build ./...`、`go test ./...` 即可（README.md 第 31、33 行）。
- 使用方式 2（参考）：只读 README.md，按需采纳目录约定，不从本仓库克隆。
- 使用方式 3（引用）：在自己的 README 或培训材料中直接引用该仓库作为布局出处——这是社区最普遍的使用形态，marmotedu/iam 即在主 README 被列为"严格遵循 project-layout 规范"的样板（README.md 第 57 行）。
- 无环境变量、无安装步骤、无可执行脚本：`scripts/` 目录只有一个 README，整个仓库不存在依赖安装或初始化动作（tarball 文件清单核验）。
- `go 1.19` 指令无特殊构建要求，克隆者改掉模块名占位符后即可正常构建；本分析按任务规范未执行任何构建或测试，以上均来自静态阅读（`go.mod`、README.md 第 33 行）。
- `Makefile` 仅为占位注释，无任何目标；本分析过程也未执行任何构建（符合任务规范）。

## 7. 内容质量与工程实践（测试章节特例改写）

- **无测试、无 CI**：仓库没有任何测试对象，也没有 `.github/` 目录或 CI 配置（tarball 文件清单核验，54 个文件中无 `.yml`/`.yaml`）。同样缺失的还有 CONTRIBUTING、CHANGELOG、CODE_OF_CONDUCT——对一个 56k 星的项目，这套协作基建的缺席相当显眼。质量保障完全依赖 PR 评审。
- **协作流程零门槛**：没有贡献指南与 PR/issue 模板意味着任何人可以低 friction 地提翻译或修订，这解释了 77 名贡献者中绝大多数只有个位数提交的"一次性贡献"分布（gh api contributors）。
- 内链健康度：主 README 指向 19 份子目录 README 的内链与 tarball 实际文件一一对应、全部有效（文件清单核验）；外链健康度未测（任务规范限制网络请求），但从维护记录看死链是持续性的修复对象。
- **一致性维护成本可见**：主 README 改动时需同步 19 个子 README 与 19 份翻译，实际已出现漂移——主 README 已将废弃的 golint 更新为推荐 staticcheck（README.md 第 39 行），而 `README_zh.md` 仍写 "请运行 gofmt 和 golint"，且翻译列表里 `README_zh-CN.md` 标着 "???" 表示出处不明（`README_zh.md` 第 8 行）。近期提交多为修死链与翻译润色（gh api 最近 3 条提交：修 rakyll.org 死链 #286、改日语措辞 #275）。
- **死链与旧名是主要维护负担**：`pkg/README.md` 约 100 个案例链接中可见已更名仓库的旧地址仍在（如 heptio/ark，已迁至 vmware-tanzu/velero，而 `cmd/README.md` 里已是 velero 新地址；docker-slim 与 slimtoolkit 两个名字也分属 `examples/README.md` 与 `pkg/README.md`）。
- **文档写作质量高**：对每个约定都给出正反两面（何时该用、何时不该用），主动声明非官方身份并链接官方文档，避免误导；这是它能在争议中站稳的原因。
- **更新机制为轻量协作流**：任何人开 issue 提出新模式或修订（README.md 第 37 行"这是社区的努力……请提一个 issue"），maintainer 评审合并；翻译由各语言社区自发认领。没有 RFC/投票等正式治理流程，轻治理是其能快速汇聚社区共识的原因之一。

## 8. 社区与维护状态

- 活跃度：创建于 2017-09-09，至今仍在维护；最后推送 2026-04-28（合并 iam 案例与翻译修复 PR），但提交频率低——这是"内容已稳定"型仓库的典型节奏（gh api commits）。
- 贡献者结构：77 名贡献者（gh api contributors）。头部高度集中：kcq（孔令飞）176 次提交，第二名 arialdomartini 22 次、rostis232 12 次，其余均在个位数——实质上是单一 maintainer 主导、社区提 PR 修错补译的模式。最后一次内容改动即由署名"孔令飞"的提交把 marmotedu/iam 加为主 README 的标杆案例（gh api 提交 #193）；maintainer 用自己的项目为布局背书形成了推荐闭环，但这一利益关联未在 README 中披露（观察，非指控）。
- Issue/PR：open issues 93、open pull requests 22（gh api，2026-11-03），多为模式讨论与翻译协调（数量核验，内容未逐一核验）。
- 增长速率可推算：56.7k 星对应 9 年存续，年均约 6k 星；5420 fork 高于同期文档型仓库的一般水平，印证"克隆当模板"是被广泛实践的使用路径（gh api；增长曲线为推断，非实测时序数据）。
- 维护节奏特征：最近 3 条提交全部集中在 2026-04-28 当天合并（#193 加 iam 案例、#275 日语润色、#286 修死链），呈"攒批定期合"的低频维护形态；翻译社区是第二贡献来源，最近提交中即有译本改进（gh api commits）。
- 星数时序未核验：GitHub API 不提供历史星标曲线，"年均约 6k 星"为总星数除以存续年数的线性估算，实际增长可能前快后慢（未确认）。
- Fork/星比近 10%（5420/56731，gh api），在文档型仓库中属高位，与"克隆当模板"的定位互证；Watch 仅 591，说明多数用户是一次性取用而非持续跟进。
- 发布节奏：无 release、无版本号——文档仓库按主干即最新使用，徽章里的 Release badge 仅是给使用者的模板示例（README.md 第 214-216 行）。

## 9. 亮点与风险

**亮点**

1. 精准踩中 Go 生态最大痛点之一"项目该长什么样"，一页文档 + 可克隆骨架，使用成本趋近于零（README.md）。
2. 每条约定都附带真实大项目案例链接（`pkg/README.md` 约 100 个），用证据而非权威压人。
3. 诚实标注约定强度与自身局限：明说非官方、明说 `/pkg` 有争议、明劝初学者别用全套（README.md 第 29-31 行），反而建立信任。
4. 19 种语言翻译 + 占位 `go.mod`/`Makefile`/`.editorconfig` 齐备，作为起步模板开箱即用（根目录文件清单）。
5. 长期低频但持续维护（2017-2026），近期还在修死链、补案例（gh api 提交记录）。

**风险/局限**

1. **非官方却名含 "standards"**：Go 官方不背书且有自己的布局指南（go.dev/doc/modules/layout），初学者易被组织名误导。README 现行版本的大写免责声明即是对这一争议的回应，但组织与仓库名至今未改——名实张力是它绕不开的原罪。
2. **许可证缺失**：`LICENSE.md` 全文仅一行 "Hacker license!"（玩笑式占位），GitHub 识别为 NOASSERTION——严格法律意义上默认保留所有权利，克隆/复制其 README 文本用于衍生项目存在许可瑕疵；对"克隆整个仓库当模板"这种官方推荐用法，这一缺陷颇有讽刺意味。
3. **内容滞后与冗余漂移**：部分案例链接指向已更名/迁移的仓库（如 heptio/ark 已改 velero，`cmd/README.md` 里两者并存），中文翻译滞后于英文主文档（golint 已废弃仍在 `README_zh.md` 中推荐）。
4. **过度套用的误用风险**：小项目硬套 19 个空目录反而增加噪音，尽管 README 已明确劝退，社区仍常见"为新项目建全套空目录"的反效果实践。

## 10. 总结评价

golang-standards/project-layout 是典型的"一份文档吃下五万星"的案例：它没有一行代码，却成了 Go 生态事实上的目录布局参考。它的价值不在规范性（Go 官方并不背书，`/pkg` 等约定在社区始终有争议），而在降低沟通成本——当团队争论"代码放哪"时，这页文档提供了一个有真实案例背书、且诚实标注了争议与强度的讨论起点。九年间由孔令飞（kcq）一人主导维护、社区修错补译的模式，也让它保持了"约定集合"而非"强制标准"的克制。

适合谁：正在把玩具项目工程化的 Go 新手（读主 README 并按需取用目录约定）、需要统一团队目录约定的 tech lead（克隆后做减法）、想深入研究大厂 Go 项目组织方式的工程师（顺着各目录 README 的案例链接读源码）、以及想研究"文档型仓库如何形成事实标准"的社区观察者。不适合谁同样明确：初学 Go 或写 PoC 的人（作者自己在 README 里说的，先从单个 `main.go` 开始），以及期望从中找到 Go 官方权威规范的人——那份文档在 go.dev/doc/modules/layout。

一句话总评：它未必是你该照抄的标准，但它一定是讨论 Go 项目结构时绕不开的参照系。
