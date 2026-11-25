# drawdb-io/drawdb 项目分析

> 星数 36500（快照 2026-10-24）｜主语言 JavaScript｜档位 20k-50k｜分析日期 2026-11-14
>
> 注：抓取当日 GitHub API 实测星数已达 40,323（`gh api repos/drawdb-io/drawdb`，2026-11-14），增长仍在持续。

## 1. 项目速览

一句话定位：纯浏览器端的数据库 ER 图（实体关系图）编辑器与 SQL 生成器——拖拽建表、连线建关系，反向导入 SQL/DBML 生成图，正向导出多方言 DDL，无需注册账号。

| 项目 | 内容 |
|---|---|
| 名称 | drawdb-io/drawdb |
| 星数 | 36,500（快照）→ 40,323（2026-11-14 实测） |
| 主语言 | JavaScript（React 18，无 TypeScript） |
| 许可证 | AGPL-3.0（`LICENSE`） |
| 创建时间 | 2023-07-16（GitHub API） |
| 最近 push | 2026-11-12（GitHub API，两天前，高度活跃） |
| Forks / Open issues | 3,365 / 250（GitHub API） |
| 官网 | https://drawdb.app（`package.json` 无 homepage 字段，README 徽章指向） |

## 2. 项目是什么，解决什么问题

数据库schema 设计长期缺少一个"轻量、免费、即开即用"的可视化工具：传统桌面工具（MySQL Workbench、Navicat、dbdiagram.io 付费版）要么重、要么收费、要么需要账号。drawDB 的答案是一个**纯前端 SPA**：打开网页即用，所有数据默认存浏览器 IndexedDB（`src/data/db.js`，Dexie 封装，schema 已迭代到 version 67），不建账号、不上传数据。

核心工作流围绕"图 ↔ SQL"双向转换：
- **正向**：在画布上拖拽创建表/字段/索引/关系（`src/components/EditorCanvas/`），一键导出为 MySQL/PostgreSQL/SQLite/MariaDB/MSSQL/Oracle 等方言的 DDL（`src/utils/exportSQL/`）；
- **反向**：粘贴 SQL 脚本，经 `node-sql-parser`（Oracle 用专用 `oracle-sql-parser`）解析为 AST，再逐方言把 AST 反解为图数据（`src/utils/importSQL/`），自动布局成 ER 图；
- **DBML 支持**：基于 `@dbml/core` 实现 DBML 导入/导出/差异对比（`src/utils/dbml/`），兼容 dbdiagram.io 生态。

项目还有两条"进阶线"：一是 `src/utils/migrations/diffToSQL.js` 支持对两个 diagram 版本做 diff 生成迁移 SQL（`Migration.jsx` 侧栏入口）；二是 `src/utils/importFrom/schemaQuery/` 内置了按方言对 `information_schema` 的查询语句，用于从**已连接的真实数据库**反向抽取 schema（依赖外部后端，见第 5 章）。开源核心与商业托管（drawdb.app 的 Pro 档）通过扩展槽位解耦（`src/context/ExtensionsContext.jsx` 的 `Slot` 机制、`src/components/Workspace.jsx` 中 `cloudSave` 与 `/checkout?tier=solo_pro` 分支），开源版无后端也能完整运行。

README 明确：如需"分享"功能才需要部署独立的 `drawdb-io/drawdb-server`（gist 式分享链接，`src/api/gists.js`），其余功能均为纯前端。

## 3. 技术栈与依赖

来自 `package.json`（`"type": "module"`，纯 JS + JSX，无 TS）：

| 类别 | 依赖 | 用途 |
|---|---|---|
| 框架 | react 18 + react-dom | UI，函数组件 + Hooks |
| 构建 | vite 8 + @vitejs/plugin-react | 开发/构建 |
| 样式 | tailwindcss 4 + @douyinfe/semi-ui 2 | Tailwind 原子类 + 字节跳动 Semi Design 组件库（Toast/Modal/侧栏等） |
| 状态 | React Context（自研 15 个 Context）+ hooks | 无 Redux/Zustand，见第 5 章 |
| 持久化 | dexie 3 + dexie-react-hooks | IndexedDB ORM，存 diagram/模板 |
| SQL 解析 | node-sql-parser 5、oracle-sql-parser | SQL → AST（导入） |
| DBML | @dbml/core 3 | DBML 解析/序列化 |
| 图布局 | @dagrejs/dagre、elkjs | 自动排列表（`src/utils/autoArrange.js`） |
| 画布编辑 | 自研 SVG 渲染（无 react-flow/konva） | 见第 5 章 |
| 编辑器 | @monaco-editor/react、lexical 0.12 | SQL 视图代码编辑、富文本笔记 |
| 拖拽列表 | @dnd-kit/core / sortable | 字段/索引排序 |
| 导出 | jspdf、html-to-image、jszip、file-saver | PDF/图片/JSON 导出 |
| 动画 | framer-motion | 界面动效 |
| i18n | i18next + react-i18next | 51 个语言包（`src/i18n/locales/`，从 ar 到 zh-tw） |
| 路由 | react-router-dom 7 | SPA 路由 |

## 4. 目录结构与代码组织

单体单包 Vite 应用，无 monorepo、无 submodule。顶层即前端工程（`index.html` + `src/`），配 Docker/Vercel 部署文件。

| 目录 | 职责 |
|---|---|
| `src/context/` | 15 个 React Context：Diagram/Tables/Relationships/Areas/Notes/Types/Enums/Views/Transform/Select/UndoRedo/SaveState/Settings/Layout/Canvas/Collab/Extensions |
| `src/hooks/` | 与 Context 一一对应的 19 个 hook（`useDiagram`、`useTransform` 等） |
| `src/components/EditorCanvas/` | 画布：Canvas.jsx（1137 行）、Table.jsx（679 行）、Relationship.jsx、Area.jsx、Note.jsx、View.jsx、ResizeHandles.jsx |
| `src/components/EditorHeader/` | 顶栏：ControlPanel.jsx（导出/导入/撤销等 1200+ 行）、Modal/（导入 SQL/DBML/JSON、连接数据库）、SideSheet/（迁移、元数据） |
| `src/components/EditorSidePanel/` | 侧栏各 Tab：Tables/Relationships/Types/Enums/Views/Areas/Notes/Issues/DBML 编辑器 |
| `src/components/Workspace.jsx` | 编辑器工作区壳层，负责 IndexedDB 加载与自动保存（674 行） |
| `src/utils/exportSQL/` | 6 方言 DDL 生成器（mysql/postgres/sqlite/mariadb/mssql/oraclesql + shared/generic） |
| `src/utils/importSQL/` | 6 方言 AST → 图反解器 |
| `src/utils/importFrom/schemaQuery/` | 连接真实库时按方言的 information_schema 查询 |
| `src/utils/dbml/` | DBML parse/diff/applyPlan/reconcile |
| `src/utils/exportAs/` | 导出为 DBML/Mermaid/Markdown 文档 |
| `src/utils/migrations/` | diffToSQL：两版差异 → 迁移 SQL |
| `src/data/` | constants、databases（方言能力开关）、datatypes（2254 行类型映射）、db.js（Dexie）、6 个模板 seed |
| `src/i18n/` | i18next 配置 + 51 语言包 |
| `src/templates/` | 6 个内置示例图（template1-6） |
| `src/api/` | gists.js（分享）、email.js（反馈）——仅有的对外 HTTP 调用，走可选后端 |

代码规模：`src/` 下 JS/JSX 共 280 个文件、约 5.5 万行；其中 51 个语言包占大头，核心代码约 3 万行。

## 5. 核心架构与关键实现

### 5.1 画布渲染：SVG 自绘引擎（非 Canvas2D、非第三方图库）

`src/components/EditorCanvas/Canvas.jsx`（1137 行）是核心：整张图画在一个 `<svg>` 里，通过 `viewBox` 实现无限画布的平移/缩放（约 944-1061 行），表格/注释/区域/关系线均为 SVG 分组，随 viewBox 缩放。这带来 DOM 级事件、CSS 样式和无障碍能力，且省去了自绘位图的 DPR 适配。

- **变换模型**：`TransformContext` 维护 `{pan, zoom}`；指针坐标由 `pointer.spaces.screen` 经 zoom 换算到图坐标系（Canvas.jsx 约 431-440 行的 pan 计算：`pan = panStart + (cursorStart - screen) / zoom`）。
- **拖拽**：表/注释/区域的 `onPointerDown` 记录 `grabOffset`（`notDragging` 哨兵，Canvas.jsx 49-57 行），移动时按图坐标回写位置；右侧缩放手柄由 `ResizeHandles.jsx` 实现。
- **连线几何**：`src/utils/calcPath.js`（245 行）纯几何计算——根据两表相对位置（左/右/重叠）与字段 Y 偏移，生成带圆角半径的折线 `d` 属性（`M/L/A` 路径，radius=10*zoom），字段位置由 `getFieldOffsetY` 按表头高度+注释高度逐行累加得出。这是"连线跟随字段"的关键。
- **自动布局**：`src/utils/autoArrange.js`（208 行）调用 dagre/elkjs 计算无重叠布局。

关键路径示意（calcPath.js，节选）：

```js
if (y1 <= y2) {
  if (x1 + startWidth <= x2) {
    return `M ${x1 + startWidth} ${y1} L ${midX - radius} ${y1}
      A ${radius} ${radius} 0 0 1 ${midX} ${y1 + radius}
      L ${midX} ${y2 - radius}
      A ${radius} ${radius} 0 0 0 ${midX + radius} ${y2} L ${endX} ${y2}`;
```

### 5.2 状态管理：15 个细粒度 Context + 动作式 Undo/Redo

不用 Redux/Zustand，而是按实体拆分 Context（`src/context/`），每个 Context 配 `useXxx` hook，且普遍拆分"数据 Context"与"Actions Context"双 Provider（如 `DiagramContext` / `DiagramActionsContext`，`src/context/DiagramContext.jsx`）以减少订阅面。`pages/Editor.jsx` 里可见 14 层嵌套 Provider 的组装顺序。

撤销/重做是**命令栈**而非快照：每次 mutation 向 `UndoRedoContext` 的 `undoStack` push `{action: ADD/DELETE/UPDATE, element: ObjectType, data, message}` 并清空 redoStack（DiagramContext.jsx `addTable` 实现）；undo 时按 action 类型逆操作。

### 5.3 多方言 SQL 生成与反解：对称的逐方言模块

- **导出**：`utils/exportSQL/index.js` 按 `diagram.database` 分发到 `toMySQL/toPostgres/toSqlite/toMariaDB/toMSSQL/toOracleSQL`；每个方言文件独立处理引号、类型映射（`data/datatypes.js` 的 `dbToTypes` 2254 行，定义每方言可用类型及其 checkDefault/increment 等能力）、枚举/数组/物化视图等特性开关（`data/databases.js` 中 `hasTypes/hasEnums/hasArrays/hasMaterializedViews/beta` 标记，Oracle 标记 `beta: true`）。共 6 方言 + 1 个 Generic（方言无关模式）。
- **导入**：`EditorHeader/Modal/Modal.jsx` 用 `node-sql-parser`（Oracle 用 `oracle-sql-parser`）把 SQL 文本解析为 AST，再交给 `utils/importSQL/index.js` 分发到 `fromMySQL/fromPostgres/...`，从 AST 反解出 tables/relationships，最后 `arrangeTables` 自动布局。"SQL 反向建图"能力是真实存在的。
- **DBML**：`utils/dbml/parse.js` 用 `@dbml/core` 的 Parser 解析 DBML；`reconcile.js`/`applyPlan.js` 支持把 DBML 编辑以增量计划方式合并回图。
- **迁移 SQL**：`utils/migrations/diffToSQL.js` 对比两版 diagram 生成 ALTER 语句（引用 `dbToTypes` 与各方言引号规则）。

### 5.4 持久化与开源/商业边界

`src/data/db.js` 用 Dexie 建两个对象表（diagrams/templates），schema 已升到 version 67 并带 uuid 迁移钩子；`Workspace.jsx` 实现打开/自动保存（保存 `tables/references/notes/areas/views/sqlHeader/pan/zoom` 全量）。6 个内置模板由 `data/seeds.js` 在 `db.on("populate")` 时写入。

开源版保留了两处"托管版插槽"：
- `context/CollabContext.jsx` 的 `emitDelta/emitAwareness` 是 no-op 默认实现，但 DiagramContext 等各处已布好 `isApplyingRemoteRef` 远程回放抑制逻辑——实时协作的接线留在托管版（开源版未含实现，基于源码可确认是占位）。
- `context/ExtensionsContext.jsx` 提供 `Slot name="..."` 渲染槽 + `Workspace.jsx` 的 `cloudSave`（402 状态跳 `/checkout?tier=solo_pro`）——云同步/付费档走托管版。
- `api/gists.js` 指向可选的 drawdb-server（gist 式分享），Docker 里通过运行时注入的 `window.__DRAWDB_CONFIG__` 配置后端地址（`Dockerfile` 生成 `/config.js`，`src/config/index.js` 读取）。

## 6. 构建与运行

来自 `package.json` scripts 与 README：

```bash
npm install
npm run dev        # vite 开发服务器（compose.yml 用 node:20-alpine 挂载跑 5173）
npm run build      # vite build 产出 dist
npm run lint       # eslint --max-warnings 0 + prettier（husky + lint-staged 预提交钩子）
npm run preview    # 本地预览构建产物

# Docker（多阶段：node:20-alpine 构建 → nginx:stable-alpine 托管静态文件，EXPOSE 80）
docker build -t drawdb .
docker run -p 3000:80 -e VITE_BACKEND_URL=https://your-server ghcr.io/drawdb-io/drawdb:latest
```

可选后端：部署 drawdb-io/drawdb-server 并设 `VITE_BACKEND_URL`/`VITE_GIST_BACKEND_URL`（`.env.sample`），仅分享/连接数据库功能需要。

## 7. 测试与工程质量

- **无自动化测试**：仓库无任何 test script、无 jest/vitest/cypress 配置，`.github/workflows/build.yml` 的 CI 仅两步——`npm run lint`（eslint 0 警告阈值 + Node 20/22 双版本矩阵）与 `npm run build`。`docker.yml` 在打 tag 时构建并推送 GHCR 镜像。
- 质量闸门依赖 lint + 预提交钩子（husky + lint-staged：JS 走 eslint --fix + prettier，其余文件走 prettier，`package.json` lint-staged 段）。
- 代码风格观察：纯 JS 无类型标注，靠 2254 行 `datatypes.js` 的集中式元数据与 Proxy 兜底（`databases`、`dbToTypes` 均用 `new Proxy` 对未知 key 返回空对象，防御性较强）；单文件偏大（Canvas.jsx 1137 行、Table.jsx 679 行、ControlPanel.jsx 1200+ 行）。

## 8. 社区与维护状态

- **高度活跃**：GitHub API 显示最近 push 2026-11-12（分析前两天）；tarball 文件 mtime 均为 2026-11-12，与 push 时间吻合。
- 版本策略：`package.json` version 恒为 `0.0.0`，发布走 git tag → docker.yml 推镜像（`docker.yml` 触发条件 `on: push: tags`），官网 Vercel 部署（`vercel.json` 全量 rewrite 到 SPA）。
- 国际化社区驱动：51 个语言包由社区 PR 维护（CONTRIBUTING.md 有贡献流程；语言包覆盖广度本身是社区规模证据）。issue 峰值 250 open，Discord 社区（README 徽章）。
- 贡献者结构：未确认（tarball 无 git 历史，未调用 contributors API）。
- 商业模式：托管版 Pro（Workspace.jsx 中 `solo_pro` checkout 分支）支撑可持续发展，开源核心 AGPL-3.0 保证不被闭源套壳。

## 9. 亮点与风险

**亮点**
1. 真正的纯前端零后端架构：IndexedDB 全量持久化 + 可选 gist 分享，隐私友好、部署成本趋近于零（nginx 静态托管即可）。
2. 自研 SVG 渲染引擎 + calcPath 几何连线：不依赖 react-flow/konva 等重库，包体可控且交互细节（字段级连线、圆角路径、缩放联动）完全自主。
3. SQL 双向能力扎实：6 方言导出 + AST 反解导入 + DBML 往返 + 迁移 diff，方言差异收敛在 `datatypes.js`/`databases.js` 的元数据层，新增方言模式清晰。
4. 51 语言国际化与 6 内置模板，上手零门槛，这是星数增长的重要引擎。
5. 开源/商业边界设计干净：Slot 扩展点 + no-op 协作桩，核心功能不依赖托管服务。

**风险/局限**
1. 无任何自动化测试：核心几何/解析逻辑全靠人工回归，重构风险高。
2. 纯 JS 无类型：2500 行的 Context/工具层在贡献者众多时类型安全靠纪律维持。
3. AGPL-3.0 对商业集成有传染性约束（对企业内嵌部署需评估）。
4. 实时协作、连接真实数据库等核心体验留在托管版，开源版对应代码为占位（CollabContext no-op），社区二次开发需自行补齐。

## 10. 总结评价

drawDB 是"纯前端工具型开源项目"的教科书案例：以 IndexedDB 换掉后端、以 SVG 自绘换掉图库依赖、以元数据驱动换掉方言 if-else，用一个约 3 万行的 React 单体做出了商业级的数据库设计体验，也因此两年多从零冲到 4 万星。它最适合作为三类读者的样本：(1) 想要免费 ER 图/SQL 设计工具的工程师与 DBA，直接用官网或 Docker 自托管；(2) 想学习"无限画布 + 命令式 Undo/Redo + AST 双向转换"实现的前端开发者，重点读 `Canvas.jsx`、`calcPath.js`、`DiagramContext.jsx` 与 `utils/exportSQL|importSQL`；(3) 想借鉴开源+托管双轨商业模式的项目作者，其 Slot 扩展点与 AGPL 授权组合值得研究。主要短板是无测试与无类型系统， contributed 代码量大后维护压力将集中在作者身上。
