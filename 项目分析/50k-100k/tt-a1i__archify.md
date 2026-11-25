# tt-a1i/archify 项目分析

> 星数 88177（快照 2026-10-24）｜主语言 JavaScript｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

**一句话定位：Archify 是一个面向 AI Agent（Claude Code、Codex、Cursor、OpenCode 等）的"技能包（Skill）"，让 Agent 把一段描述、一个计划或一个真实代码库变成可交互、经过校验、单文件分发的 HTML 可视化图（架构图/流程图/时序图等 10 种）+ 一页式说明页（explainer）。**

| 事实 | 内容 | 出处 |
|---|---|---|
| 名称 | tt-a1i/archify | gh api repos/tt-a1i/archify |
| 星数 | 91,785（分析日实时值；快照 88,177） | gh api，2026-10-30 |
| Fork / Issue | 6,250 fork / 268 open issues | gh api |
| 主语言 | JavaScript（Node.js >= 18，零运行时依赖） | `package.json`、`archify/package.json` |
| 许可证 | Apache-2.0（历史 MIT 授权保留，见 `NOTICE`、`LICENSE-MIT`） | `LICENSE`、README License 节 |
| 当前版本 | v3.2.0（2026-10-18 发布） | `archify/package.json`、`CHANGELOG.md` |
| 创建时间 | 2026-04-15 | gh api created_at |
| 最近 push | 2026-10-30（分析前一天，极活跃） | gh api pushed_at |
| 官网 | https://archify.si | gh api homepage |
| 安装方式 | `npx skills add tt-a1i/archify -g`（skills.sh 显示 130K+ 安装） | `README.md` |

## 2. 项目是什么，解决什么问题

Archify 诞生的场景是：AI 编码 Agent 无处不在，但 Agent 生成的图（典型如 Mermaid）往往不可校验、不可复现、布局混乱。Archify 的答案不是"又一个画图库"，而是把**图的生成、校验、交付做成一条有合同（contract）的流水线**，交给 Agent 执行（`README.md` "Why Archify"、"How it works" 节）。

工作方式：用户在 Claude Code / Codex / Cursor 等 Agent 里说一句"用 Archify 画：浏览器 -> API -> Redis 缓存 -> PostgreSQL"。Agent 读取技能包中的 `archify/SKILL.md`（137 行的强约束工作流），按 10 种图类型路由（architecture/workflow/sequence/dataflow/lifecycle/erd/class/tree/timeline/waterfall，另有 explainer 一页文档），写出一份**类型化 JSON 中间表示（IR）**，然后调用仓库自带的零依赖 CLI `node bin/archify.mjs finalize <type> <candidate.json> <output.html>` 交付（`archify/SKILL.md`）。

`finalize` 是整个项目的核心卖点：它串起 schema 校验 → 布局规则检查（路由质量、标签避让、交叉检测）→ HTML/SVG 渲染 → 原子交付 → 真实浏览器回归检查 → 溯源校验六道门。任何一道失败都以机器可读 JSON 返回**稳定规则码 + 精确主体 + 修复建议（supportedFixes）**，Agent 据此在两轮内修复重跑，而不是拿到一段 Node 堆栈乱猜（`README.md` "The engineering behind the experience"，`archify/renderers/shared/diagnostics.mjs`）。

对真实代码库，Archify 支持"证据图"：Agent 先读仓库，架构节点标注 `SRC n`，链接到**固定 commit 的 Git 验证文件与行区间**；`repository-evidence.mjs` 用 `git --no-replace-objects` 批量预取 blob、逐行核对引用，上限 16MB/文件，引用不实则交付失败（`archify/renderers/shared/repository-evidence.mjs`）。

产出物是**单文件自包含 HTML**：离线可打开、可发给任何人，内置节点聚焦、上下游 reach、路径探测（R 键）、角色对比（LENS）、概览雷达（MAP）、演示模式等查看器能力，可导出 PNG/WebP/JPG/WebM 与 1200×630 路线分享卡（`README.md` "Explore and share the output"、`viewer/` 目录）。

## 3. 技术栈与依赖

- **语言/运行时**：纯 JavaScript（ESM），Node >= 18，**运行时零第三方依赖**——渲染、校验、CLI 全部自实现（`archify/package.json` 无 dependencies，仅 devDependencies）。
- **校验**：devDependencies 用 `ajv ^8` 做 JSON Schema 校验，但校验器代码是**生成产物**（`archify/renderers/shared/generated-validators.mjs` 由 `scripts/generate-validators.mjs` 生成，CLI 内置运行时不依赖 ajv）。
- **工程辅助**（仅开发期）：`parse5`/`saxes`（HTML/SVG 解析检查）、`yaml`、`simple-icons`（品牌图标生成）、`skills` CLI。
- **网站**：`website/` 为 Astro 站点（archify.si，`website/astro.config.mjs`）。
- **测试**：Node 内置 `node:test` + 自研浏览器测试管道（真实 Chrome）。
- **产物形态**：查看器 16 个 JS 模块（约 10,146 行，`viewer/*.js`）经 `scripts/generate-viewer.mjs` 内联进 167KB 的 `viewer/template.source.html`，与渲染出的 SVG 合成最终单文件 HTML。

## 4. 目录结构与代码组织

| 目录 | 职责 | 出处 |
|---|---|---|
| `archify/` | **核心技能包**（npm 私有包 v3.2.0）：SKILL.md、bin/ CLI、renderers/、schemas/、examples/、locales/、references/、recipes/ | `archify/` |
| `viewer/` | 查看（交互）运行时源码：相机、导出、焦点、路由探测、语义雷达等 16 模块 | `viewer/` |
| `scripts/` | 工程脚本：测试运行器、查看器/画廊/官网构建、发布身份检查、Windows 路径测试 | `scripts/` |
| `test/` | 235 个单测文件 + 41 个浏览器测试文件 + golden 基准 | `test/` |
| `docs/` | 官网静态产物、gallery、验收证据、cookbook（39MB，含大量截图与案例 HTML） | `docs/` |
| `examples/` | 19MB 渲染好的示例图 HTML | `examples/` |
| `website/` | Astro 官网源码 | `website/` |
| `benchmarks/` | 三个基准：ordinary-model-floor（普通模型一次可用率）、repair-rounds（修复轮次）、real-repo-generation（真实仓库生成） | `benchmarks/` |
| `integrations/` | hermes-agent、deepseek-harness 两个社区集成适配器 | `integrations/` |
| `.agents/skills/` | 仓库自用的开发工作流 Skill（archify-review、archify-tuning、simplify-codebase、test-value） | `.agents/skills/` |
| 根目录文档 | README×4 语言、CHANGELOG(120KB)、CONTRIBUTING(33KB)、ROADMAP(48KB)、DESIGN.md 设计系统、REVIEWING、SECURITY | 根 |

组织方式评述：**monorepo 单包核心**形态。可分发的只有 `archify/`（安装时复制该目录为 Skill），其余全是开发与证据基建；website、integrations、benchmarks 各自独立小包。仓库把"验收证据"（acceptance 文档、画廊、案例）当作一等公民放进 git，这与"可校验交付"的产品哲学一致，但也让 tarball 膨胀到 143MB（gh api size）。

## 5. 核心架构与关键实现

### 5.1 CLI 与 finalize 门禁流水线（`archify/bin/archify.mjs`，7,289 行）

CLI 提供 render/validate/deliver/finalize/compare/patch/preview/check/browser-check/visual-check/migrate/inspect/guide/doctor/demo 等 17 个子命令（`archify/bin/archify.mjs` 7216-7276 行的命令分发）。`finalize` 一次调用串起全部门禁，输出"回执（receipt）"：validate → deliver（原子替换）→ 严格溯源 check → browser-check。边车文件（`.delivery.json`、`.delivery-lock.json` 等）用专门的 sidecar-path 运行时管理 Windows 大小写/路径归一问题（bin/archify.mjs 头部 DELIVERY_SIDECAR_SUFFIXES 与 `renderers/shared/sidecar-path.mjs`）。

### 5.2 原子交付（`archify/renderers/shared/atomic-output.mjs`，1,982 行）

交付不是"写文件"而是可验证的事务：候选文件先写到同目录临时名，校验通过后经 rename 原子替换；模块对 Windows 做了极其防御的处理——用句柄 fstat（bigint）作权威身份、`O_NOFOLLOW`、打开前后 lstat 快照对账、失败一律 fail-closed 返回结构化 reason code（文件头注释："Windows lstat can fall back to directory enumeration ... Treat handle-backed fstat as authoritative and fail closed"）。这保证了"失败时保留上一个已验证的好文件"（last-good live preview）的承诺。所有状态判定都以统一的 relation 结构返回，绝不静默降级（`atomic-output.mjs` 9-25 行）：

```js
function relation(status, code, details = {}) {
  return { status, reason: { code, ...details } };
}

// Windows lstat can fall back to directory enumeration when a file handle
// cannot be opened, and that fallback reports a synthetic link count of one.
// Treat handle-backed fstat as authoritative and fail closed when it cannot be
// reconciled with the no-follow path snapshots on either side of the open.
function captureRegularFileHandle(filePath, initial, subject) {
  ...
  if (initial.ino === 0n) {
    return relation('unknown', `${subject}-identity-unavailable`, { [pathKey]: filePath });
  }
```

### 5.3 布局引擎与几何质量（`archify/renderers/shared/geometry.mjs` 2,073 行 + `renderers/architecture/routing.mjs` 1,469 行）

布局判断优先于通用自动布局是 README 明示的设计立场。`render-architecture.mjs`（1,204 行）导入十余个清理函数：`cleanCrossingProblems`、`cleanLabelRouteClearanceProblems`、`cleanAmbiguousCorridorProblems`、`cleanRouteRhythmProblems`、`collectArrowheadCollisions` 等——每类布局缺陷都有专属检测与修复建议；`grid.mjs` 做 ERD 域分组的矩形网格排布（对应 SKILL.md 中"tagged table 必须落格"的约定），`shared/automatic-labels.mjs` 让共享端点的连线在多个确定点位分散而非堆在同一个中点（`README.md` "Layout judgment over generic auto-layout"）。

### 5.4 Skill 合同（`archify/SKILL.md`，137 行）

这是 Agent 的"操作系统"：类型路由表给出 10 种图各自的 schema 与示例路径；规定 Agent 直接写完整候选（不规划坐标）、`meta.quality_profile` 默认 showcase、ERD 表格优先布局、失败读 diagnostics 只做两轮修复、每轮 browser-check 后必须换 `--out-dir review-<n>` 避免证据被覆盖；还定义了 Mermaid 输入的语义转换（读拓扑不抄样式）和 explainer 的触发词（含中文"做成一页/用页面讲"）。Agent 侧的标准交付入口只有一条命令（`archify/SKILL.md`）：

```bash
node bin/archify.mjs finalize <type> <candidate.json> <output.html> \
  --quality showcase --json
# 仓库证据图追加 --repo-root <repo-root>
```

它本质上是把工程约束写成 prompt 合同——该项目最独特的资产。

## 6. 构建与运行

来自 `README.md`、根 `package.json` 与 `archify/package.json`（按规范未实际运行）：

```bash
# 安装为 Agent 技能（用户侧）
npx skills add tt-a1i/archify -g

# 仓库内自检 / 演示 / 场景路由建议（零依赖，Node>=18）
cd archify && node bin/archify.mjs doctor
node bin/archify.mjs demo /tmp/archify-demo
node bin/archify.mjs guide "Show CI/CD checks, approval, deploy, and rollback"

# 校验/预览/交付一张图
node bin/archify.mjs validate workflow examples/agent-tool-call.workflow.json --quality showcase --json
node bin/archify.mjs finalize workflow examples/agent-tool-call.workflow.json out.html --quality showcase --json

# 测试（根 package.json）
npm run test:core      # 冒烟
npm run test:full      # 生成物一致性 + 全量
npm run test:browser   # 真实浏览器测试
npm run test:affected  # 按改动范围选测
```

## 7. 测试与工程质量

- **规模**：`test/` 下 235 个 `*.test.mjs` 单测 + 41 个浏览器测试 + golden 基准（`test/golden.mjs`）；覆盖面按渲染器与机制命名（route-quality、label-clearance、atomic-write、repository-evidence、i18n、workflow-migration 等），粒度极细（`test/` 目录清单）。
- **CI**：`.github/workflows/ci.yml` 先跑 scope 分类 job（输出 website/generated/browser/windows/package 等 11 个开关），按改动范围裁剪执行；所有 action 按 commit SHA 固定版本；另有 release、contributors、star-history、dsh 四条 workflow（`.github/workflows/`）。
- **质量观察**：生成物一致性纳入测试（`test:generated` 用 `--check` 校验 viewer/brand-marks/validators 由源码可再生）；基准套件自带公平性协议文档（同 prompt、同 commit、冻结候选、外部复核，`benchmarks/ordinary-model-floor/README.md`）；还有专门的 Windows 路径测试与修复脚本（`scripts/run-windows-path-tests.mjs`、`test/checkout-line-endings.test.mjs`）。整体工程纪律在同类个人项目中罕见。

## 8. 社区与维护状态

- **活跃度**：创建于 2026-04-15，半年到 88k+ 星；最近 push 2026-10-30，近 5 日提交密集（gh api commits）。Release 节奏约每 1-2 周一版（v3.0.0 9/28、v3.1.0 10/13、v3.2.0 10/18，gh api releases）。
- **贡献者结构**：30 名贡献者，但 tt-a1i 一人 1,138 次提交，第二名 outlier27-cell 仅 131 次（gh api contributors）——**强单人主导**，社区贡献以案例、文档与社区包为主（`community/README.md` 收录流程）。
- **生态热度**：README 记录 2026-09-01 登上 GitHub Trending 周榜全语言第一；量子位（QbitAI）两次报道；Kimi Work、Supercode、OpenLux、PackyCode 等多个赞助商（`README.md` Sponsors 节）；Discord + 微信 + QQ 三渠道社区，中文 README 完整。
- **商业化姿态**：开源本体免费，官网提供信息与安装指引，明确声明第三方付费服务与项目无关（`README.md` "Official channels and third-party services"）。

## 9. 亮点与风险

**亮点**
1. **可校验交付合同**是真实创新：把"Agent 产图"从碰运气变成带稳定错误码、修复建议与回执的确定性流水线（`bin/archify.mjs`、`SKILL.md`、`diagnostics.mjs`）。
2. **零运行时依赖 + 单文件产物**：Node>=18 即跑，输出离线自包含 HTML，分发与安全审计成本极低（`archify/package.json`）。
3. **工程质量极高**：235+41 个测试、生成物一致性检查、原子交付的 Windows 边界处理、按范围裁剪的 CI（`test/`、`atomic-output.mjs`、`ci.yml`）。
4. **把基准与验收当产品**：三个 benchmark 套件 + 画廊验收证据全部入库，敢用数据说话（`benchmarks/`、`docs/`）。
5. **多语言与社区运营成熟**：4 语言 README、6 种 UI locale（en/zh-CN/zh-TW/es/ko/ar，`archify/locales/`）、三渠道社区。

**风险/局限**
1. **强单人主导（bus factor）**：核心代码几乎全部出自 tt-a1i，1,138/1,559 次提交（gh api contributors），可持续性系于一人。
2. **仓库体量与信噪比**：143MB tarball 中 docs(39MB)+examples(19MB)+`archify.zip`(5.4MB，为 Claude.ai 上传用) 占大头，克隆/审计成本高（gh api size、根目录清单）。
3. **生态绑定风险**：价值建立在 Claude Code/Codex/Cursor 等 Agent Skill 生态之上，技能规范变化可能要求快速跟进；官方也明确不做通用绘图编辑器、不做 Mermaid 主题、不做托管分享，能力边界较窄（`README.md` "Reference and scope"）。
4. **星数与使用可能存在落差**：88k 星对应 skills.sh 130K+ 安装，但 issue 区 268 open issues，单人维护下响应能力可能吃紧（gh api；后续走向未确认）。

## 10. 总结评价

Archify 是"AI Agent 技能包"赛道目前工程完成度最高的代表之一：它没有把宝押在更炫的自动布局上，而是把信任问题——Agent 生成物如何被机器校验、如何带证据交付、失败如何可修复——做成了一条带回执的确定性流水线，并用海量测试、基准与验收证据自我背书。半年 88k 星、Trending 周榜第一与多家赞助印证了切入点（给 Claude Code/Codex 用户画可分享的交互图）踩中了浪潮。适合两类人：想把零散描述或真实代码库变成可信交互图的 Agent 重度用户；以及想研究"如何为 LLM Agent 设计带验证门的工具合同"的工程师——`archify/SKILL.md` + `atomic-output.mjs` + `renderers/shared/geometry.mjs` 这条"合同→事务→几何"的链路非常值得精读。主要保留意见在单点维护与 Agent 生态依赖，长期健康度需要观察 v4.x 之后的迭代与社区贡献占比变化。
