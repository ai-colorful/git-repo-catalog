# AykutSarac/jsoncrack.com 项目分析

> 星数 39200（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-06

## 1. 项目速览

一句话定位：JSON Crack 是一个开源的 JSON 编辑器与交互式图谱可视化工具，把 JSON/YAML/XML/CSV 等数据瞬间渲染成可折叠、可缩放的节点图，号称"数据格式的可视化水晶球"。

| 事实 | 值 |
| --- | --- |
| 名称 | JSON Crack（AykutSarac/jsoncrack.com） |
| 星数 | 39200（快照）；gh api 实测 44588（2026-11-06） |
| 主语言 | TypeScript（React） |
| 许可证 | Apache-2.0（VS Code 扩展子包为 MIT，见 `apps/vscode/package.json`） |
| 创建时间 | 2022-01-30（gh api） |
| 最近 push | 2026-09-14（gh api，距分析日约 2 个月） |
| 最新 release | v5.0.0（2026-03-11，gh api） |
| 仓库形态 | pnpm monorepo（Turborepo 编排） |

## 2. 项目是什么，解决什么问题

开发者在排查 API 响应、配置文件、日志结构时，面对深层嵌套的 JSON 往往需要在脑中还原树形结构。JSON Crack 把这件事变成图形操作：左侧是 Monaco 代码编辑器，右侧实时渲染出对象/数组/标量节点的图谱，支持点击折叠、缩放平移、暗色主题（README "About the Project"）。

功能面覆盖一个完整的数据工具链（README + 源码核对）：多格式可视化（JSON/YAML/XML/CSV，`apps/www/src/enums/file.enum.ts`）、格式互转（`apps/www/src/lib/utils/jsonAdapter.ts`）、格式化与校验、代码生成（TypeScript/Go/Kotlin/Rust/JSON Schema，`apps/www/src/features/modals/TypeModal/index.tsx`）、jq 与 JSONPath 查询（`apps/www/src/hooks/useJsonQuery.ts`）、导出图片（html-to-image）。README 强调隐私：所有解析在浏览器本地完成，数据不上传服务器。

工程上它有三层产品形态（`apps/` 与 `packages/` 划分）：jsoncrack.com 网站（Next.js 静态站）、VS Code 扩展（Marketplace 上架，v5.1.0）、Chrome 扩展，以及抽出复用的 npm 组件 `jsoncrack-react`。商业化走"开源核心 + 姊妹商业产品"路线：站点内嵌 "Open in ToDiagram" 按钮，通过 postMessage 把文档交给作者的商业产品 todiagram.com（`apps/www/src/lib/utils/todiagramHandoff.ts`），README 赞助区块也指向 ToDiagram。

仓库本身没有后端代码：`apps/www/next.config.js` 配置 `output: "export"`，整站静态导出后由 GitHub Actions 部署到 GitHub Pages（`.github/workflows/deploy.yml`），这与"数据不出本地"的产品承诺一致。

## 3. 技术栈与依赖

- 语言/运行时：TypeScript 5.9，Node >= 24，pnpm >= 10（根 `package.json` engines）
- 框架：Next.js 16.3.5（Pages Router，`next dev --webpack`）+ React 19.3（pnpm catalog 统一版本，`pnpm-workspace.yaml`）
- 图谱渲染：`reaflow` 5.4.1（内部封装 ELK.js 分层布局）+ `react-zoomable-ui`（平移/缩放视口），`packages/jsoncrack-react/package.json`
- 解析/转换：`jsonc-parser`（JSON/JSONC 解析与语法树）、`js-yaml`、`fast-xml-parser`、`json-2-csv`、`jsonpath-plus`、`jq-web`（jq 的 WASM 实现）、`json_typegen_wasm`（类型代码生成）——均见 `apps/www/package.json`
- UI：Mantine 8 组件库、styled-components、@monaco-editor/react、react-hot-toast、shiki
- 导出：html-to-image（PNG/JPEG/SVG 截图）
- 状态管理：zustand 5（含 persist 中间件）
- 工程：Turborepo 2.10 + pnpm workspace；Vite 8 构建 `jsoncrack-react` 与 Chrome 扩展；VS Code 扩展用 esbuild（host）+ Vite（webview），见 `apps/vscode/README.md`

## 4. 目录结构与代码组织

| 目录 | 职责 |
| --- | --- |
| `apps/www/` | jsoncrack.com 主站：Next.js 静态导出，编辑器 UI、store、格式转换 |
| `apps/www/src/store/` | zustand 全局状态（useJson/useFile/useConfig/useModal） |
| `apps/www/src/features/` | 编辑器主体（editor/）、弹窗集合（modals/：Download、Import、JQ、JPath、Schema、Type 等） |
| `apps/www/src/features/editor/views/` | GraphView（图谱视图）与 TreeView（树视图）双视图 |
| `apps/www/src/lib/utils/` | jsonAdapter（格式互转）、json2go、todiagramHandoff、search 等 |
| `packages/jsoncrack-react/` | 核心图谱引擎，可独立发布的 React 组件（npm 包） |
| `apps/vscode/` | VS Code 扩展（ext-src 宿主 + src React webview） |
| `apps/chrome-extension/` | Chrome 扩展（Vite + jsoncrack-react） |
| `.github/workflows/` | deploy.yml（Pages 部署）、pull-request.yml（lint+build） |

组织方式评述：标准 Turborepo 单仓多包。2025-2026 年的重构方向清晰——把原本内嵌在 www 里的画布引擎抽成 `packages/jsoncrack-react`，使 www、Chrome 扩展、npm 包三方复用同一渲染核心（`apps/www/src/features/editor/views/GraphView/index.tsx` 直接 `import { JSONCrack } from "jsoncrack-react"`）。引擎包仅约 1900 行 TS（wc -l 统计），职责单一，是这个仓库真正的技术资产。

## 5. 核心架构与关键实现

### 5.1 图谱解析器：JSON 文本 → 节点/边（packages/jsoncrack-react/src/parser.ts）

不经过中间 AST，直接遍历 `jsonc-parser` 的 `parseTree` 语法树，把"对象=一个多行节点、数组元素=独立节点"转成扁平的 nodes/edges。关键设计是数组不去重复合结构——数组每个元素独立建节点并通过 `parentKey` 标注归属：

```ts
if (isArray && isRootArray) {
  nodes.push({ id, text: [{ key: null,
    value: `[${node.children?.length ?? 0} items]`, ... }] });
  node.children?.forEach(child => traverse(child, id));
}
```

节点文本行（`NodeRow`）携带 `key/value/type/childrenCount/to`，对象与子对象的引用以 `edges` 表达（`{from, to, text: key}`），同时用 `getNodePath` 记录每个节点的 JSONPath——这是折叠功能和节点弹窗的基础。节点尺寸由 `utils/calculateNodeSize.ts` 按等宽字体 8px/字符估算，带 120 秒 TTL 的全局缓存。

### 5.2 渲染管线与折叠（packages/jsoncrack-react/src/JSONCrackComponent.tsx）

渲染栈为 reaflow `Canvas`（ELK 分层布局，`layoutOptions` 配置了 NETWORK_SIMPLEX 节点安置与 EDGE_LENGTH 后压缩）嵌在 `react-zoomable-ui` 的 `Space` 里实现平移缩放。解析入口做了三态分发（`canvasHelpers.ts` 的 `parseJsonGraph`）：正常 / 超出 `maxRenderableNodes`（默认 1500，可用 `NEXT_PUBLIC_NODE_LIMIT` 调整，`apps/www/src/constants/graph.ts`）/ 解析异常，超限时渲染降级 UI 而非卡死页面。

折叠实现相当讲究：折叠状态以"序列化 JSONPath 字符串集合"表达，支持受控/非受控两种模式；编辑器内容变化后用 `prunePaths` 清掉已失效的折叠路径；可见性过滤用预解析的路径前缀比对，避免每节点重复 parse；点击折叠按钮时记录屏幕坐标，ELK 重排完成后把该按钮"钉"回光标下，过渡期淡出画布避免闪烁（JSONCrackComponent.tsx 的 `pendingRecenterRef` / `isRelayouting` 段落）。

### 5.3 格式转换层（apps/www/src/lib/utils/jsonAdapter.ts）

`contentToJson` / `jsonToContent` 两个函数构成唯一的格式边界：JSONC/JSON 用 jsonc-parser（容错解析，失败再抛 `JSON.parse`）、YAML 用 js-yaml、XML 用 fast-xml-parser（属性以 `$` 前缀映射）、CSV 用 json-2-csv。全部动态 `import()`，保证这些解析器不进首屏 bundle。格式切换即"旧格式→JSON→新格式"的两跳转换（`store/useFile.ts` 的 `setFormat`），失败则清空并提示。

### 5.4 状态管理与编辑联动（apps/www/src/store/）

四个 zustand store 切分清晰：`useJson`（图谱消费的 JSON 字符串）、`useFile`（编辑器文本 contents/格式/错误/会话恢复，<80KB 内容存 sessionStorage）、`useConfig`（persist 持久化的暗色/实时转换/手势/网格偏好）、`useModal`。编辑→图的链路：Monaco 输入 → `setContents` → `contentToJson` → 400ms debounce 写入 `useJson` → GraphView 重解析渲染（`store/useFile.ts`）。

## 6. 构建与运行

来自根 `package.json` scripts 与 README：

```bash
pnpm install
pnpm dev:www        # 开发 http://localhost:3000
pnpm build:www      # 构建（静态导出到 apps/www/out）
pnpm dev:vscode     # VS Code 扩展 watch
pnpm dev:chrome     # Chrome 扩展 watch
pnpm build          # turbo 全量构建
pnpm lint           # tsc + eslint + prettier
pnpm test           # turbo run test（实际只有 jsoncrack-react 有 vitest）
```

要求 Node >= 24、pnpm >= 10（`packageManager: pnpm@10.20.0`）。网站是纯静态产物，可托管任意静态服务。

## 7. 测试与工程质量

- 测试规模：仅 `packages/jsoncrack-react/src/__tests__/` 下 3 个 vitest 文件（parser、canvasHelpers、CollapseContext），覆盖引擎包的核心纯函数；www 应用层无测试（find 全仓 *.test.* 证实）。
- CI：`pull-request.yml` 对 PR 执行 turbo lint + build，未运行 test；`deploy.yml` 在 push main 时构建并部署 GitHub Pages。即 CI 不含测试门禁，测试靠本地 `pnpm test`。
- 代码质量观察：引擎包代码注释密度高且注释解释"为什么"（如 handoff 不加 noopener 的原因），大量 useMemo/Callback 与 ref-mirror 模式控制重渲染，ResizeObserver 处理容器尺寸同步，工程成熟度明显高于同体量项目。lint 链条完整（tsc --noEmit + eslint + prettier --check 三合一，`apps/www/package.json` lint script）。

## 8. 社区与维护状态

- 活跃度：gh api 显示最近 push 2026-09-14，距分析日约 2 个月；46 个开放 issue；fork 3585。tarball 无历史，contributor 总数按 API 分页末页推算约 37 人（contributors?per_page=1 末页 page=38），核心主导者单一（作者 Aykut Saraç，root package.json author）。
- Release 节奏：大版本约一年一个——v3.2.0（2023-12）、v4.0.0（2025-04）、v5.0.0（2026-03），v5 与"引擎抽包 jsoncrack-react"重构对应，节奏规律但偏慢。
- 社区运营：README 挂 Discord、HN/Product Hunt 徽章；CONTRIBUTING.md 要求 issue-first 工作流（先开 issue 经作者批准再写 PR），显示维护者对贡献方向有较强控制。
- 未确认：近两月低提交频率是否代表维护放缓（可能因商业产品 ToDiagram 分流），需跟踪后续观察。

## 9. 亮点与风险

亮点：
1. 引擎抽包设计干净：`jsoncrack-react` 约 1900 行自成一体（解析、布局、折叠、缩放、主题），Web/VS Code/Chrome/npm 四端复用，是可以单独借鉴的 React 图谱组件范本。
2. 折叠交互细节打磨到位：路径失效自动修剪、重排后视口锚定、受控/非受控双模式，体现了对图编辑器这种"异步布局 + 用户意图"冲突的深入理解。
3. 隐私架构自洽：纯静态站 + 全本地解析（WASM jq、WASM 代码生成），无后端依赖，自托管零成本，企业内网可用。
4. 功能闭环完整：可视化、互转、校验、代码生成、查询、导图，一个工具覆盖 JSON 工作流大部分场景。

风险/局限：
1. 渲染上限 1500 节点（默认），大文件场景直接进入 NotSupported 降级（`GraphView/index.tsx` + `constants/graph.ts`），ELK 全量重排在超大文档下也无增量布局。
2. 测试覆盖薄弱且 CI 无测试门禁，www 应用层（store/转换逻辑）零测试，回归主要靠人工。
3. 商业化与开源边界依赖作者自觉：核心资产在姊妹闭源产品 ToDiagram 中延续，开源版更新节奏受商业优先级影响（近两月 push 停滞，未确认趋势）。
4. Pages Router + `--webpack` 在 Next 16 时代属保守选择，styled-components 与 Mantine 双 UI 体系并存有一定技术债。

## 10. 总结评价

JSON Crack 是"单点体验做到极致"的典型：把 JSON 可视化这一个动作打磨成星数近 4 万的现象级工具，再用 VS Code/Chrome 扩展和 npm 包把体验铺满开发者动线。其开源核心完整、可自托管、隐私友好，架构上最值得读的是 `packages/jsoncrack-react`——小而美的 ELK 图谱引擎，适合前端工程师学习"数据结构到交互图形"的完整管线，以及需要在自己的产品中嵌入 JSON 可视化能力的团队直接引用。把它当作日常 JSON 排查工具或自托管内网工具都非常合适；但若期待大规模数据渲染、团队协作或持久化，则需注意其静态站定位与 1500 节点上限，作者显然把这部分能力留在了商业产品侧。
