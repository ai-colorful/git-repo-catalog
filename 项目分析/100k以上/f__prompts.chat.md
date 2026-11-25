# f/prompts.chat 项目分析

> 星数 174175（快照 2026-10-24）｜主语言 HTML｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：全球最大的开源 AI 提示词库（f.k.a. Awesome ChatGPT Prompts），已从单一 README 清单演化为一整套"提示词社交平台 + 数据集 + CLI/MCP/插件生态"。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | f/prompts.chat（prompts.chat-v2） | `package.json` name 字段 |
| 星数 | 174,175（快照）；查询时 175,072，fork 22,340 | gh api repos/f/prompts.chat |
| 创建时间 | 2022-12-05 | gh api |
| 最近 push | 2026-10-29（分析前一天，极度活跃） | gh api |
| 默认分支 | main | gh api |
| 主语言 | GitHub 标记 HTML；实际代码为 TypeScript（Next.js 16 + React 19） | `package.json`、`next.config.ts` |
| 许可证 | 双许可：代码 MIT（`LICENSE-MIT`），提示词数据 CC0 公有领域（`LICENSE-CC0`），`LICENSE` 为说明文件 |
| 贡献者 | GitHub API 显示约 1,475 名（contributor 列表 last page=1475）；`prompts.csv` 内 distinct contributor 1,166 |
| 提示词规模 | `prompts.csv` 2,744 条（RFC4180 解析统计），文件 174,736 行 / 8.9MB；`PROMPTS.md` 208,467 行 / 9.2MB |

## 2. 项目是什么，解决什么问题

prompts.chat 起源于 2022 年 12 月 ChatGPT 爆发之初，最早形态是单文件 README 精选清单"Awesome ChatGPT Prompts"（`README.md` 自述 "First prompt library (Dec 2022)"）。它解决的问题是：普通用户不知道如何向 AI 提问，项目提供经过社区验证、开箱即用的角色扮演式提示词（如 "act as a Linux terminal"），复制即用。

三年间项目完成了三次形态升级，这一点从仓库结构可直接看出：

1. **静态清单 → 数据集**：提示词数据固化为机器可读的 `prompts.csv`（字段 act/prompt/for_devs/type/contributor）与 `PROMPTS.md`，并同步到 Hugging Face 成为"most liked dataset"（`README.md` 徽章、`.github/workflows/update-contributors.yml` 中的 Sync to Hugging Face 步骤）。
2. **数据集 → 平台**：仓库现在是一个完整的 Next.js 16 全栈社交平台源码（`CLAUDE.md` 自述 "social platform for AI prompts"），支持提示词的提交、投票、评论、分类、收藏、版本化（ChangeRequest）与 AI 语义搜索，线上 prompts.chat 是数据的源头。
3. **平台 → 生态**：通过 `npx prompts.chat` CLI（`packages/prompts.chat/`）、MCP Server（`src/pages/api/mcp.ts`）、Claude Code 插件市场（`.claude-plugin/marketplace.json`）、Raycast 扩展（`packages/raycast-extension/`）和自托管方案（`SELF-HOSTING.md`、`compose.yml`）把提示词库嵌入各种 AI 工作流。

对自托管用户，项目强调"私有提示词库"：`scripts/setup.js`（596 行安装向导）配置品牌、主题、认证（GitHub/Google/Azure AD/credentials），`prompts.config.ts` 提供 white-label 克隆开关（`useCloneBranding`）与 PCHAT_* 运行时环境变量（`compose.yml` 注释）。

## 3. 技术栈与依赖

依据 `package.json`、`AGENTS.md`、`prisma/schema.prisma`：

- **框架**：Next.js 16（App Router，`next.config.ts`）+ React 19.2 + TypeScript 5，Node 24（engines 限定）
- **数据库**：PostgreSQL 17（`compose.yml`）+ Prisma ORM 6.19（`prisma/schema.prisma` 501 行）
- **认证**：NextAuth.js v5 beta，可插拔 providers（`prompts.config.ts` auth.providers：github/google/apple）
- **UI**：Tailwind CSS 4 + Radix UI 全家桶 + shadcn/ui 模式（`components.json`）、lucide-react、Monaco Editor
- **AI 能力**：openai SDK（语义搜索 embeddings、提示词改进、SQL 生成等，`src/lib/ai/` 下以 YAML 形式存放系统提示词如 `improve-prompt.prompt.yml`、`sql-generation.prompt.yml`）
- **MCP**：`@modelcontextprotocol/sdk` 1.24（`src/pages/api/mcp.ts`）
- **可观测**：Sentry（`sentry.server.config.ts`/`sentry.edge.config.ts`）、pino 日志
- **i18n**：next-intl，`messages/` 下 17 个语言 JSON（en/tr/es/zh/ja/ar/pt/fr/it/de/nl/ko/ru/he/el/az/fa）
- **测试**：Vitest 2 + Testing Library + jsdom；**工程化**：ESLint 9、tsup（CLI 打包）、Puppeteer（书籍 PDF 生成脚本 `scripts/generate-book-pdf.ts`）

数据模型方面，`prisma/schema.prisma`（501 行）定义 20 个模型与 11 个枚举：核心是 User/Prompt/PromptVersion/ChangeRequest，社交侧有 PromptVote/Collection/Comment/Notification，平台运营侧有 Category/Tag/WebhookConfig/PromptConnection/BotPairing/PromptReport 等，说明这不是简单 CRUD 而是带审核、版本、通知闭环的社区产品。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `src/app/` | Next.js App Router 页面与 API：prompts/categories/tags/collection/feed/bots/book/kids/admin/settings 等 30+ 路由组；`src/app/api/` 下 21 个 API 域（prompts、search、generate、improve-prompt、leaderboard、cron 等） |
| `src/pages/api/mcp.ts` | MCP Server 端点（1530 行，唯一的 pages 路由，与 App Router 共存） |
| `src/components/` | 219 个 TSX 组件（含 `ui/` shadcn 基件） |
| `src/lib/` | 核心业务库：auth、ai/、plugins/（媒体生成器/组件注册表）、bots/、similarity、trending、rate-limit、skill-files、variable-detection 等 30+ 模块 |
| `src/content/book/` | 交互式提示词工程书的 32 个 MDX 章节（00a-preface ~ 30+） |
| `src/i18n/`、`messages/` | 多语言框架与翻译文件 |
| `prisma/` | schema、migrations、seed（seed 通过 `https://prompts.chat/prompts.json` 拉取线上数据初始化，见 `prisma/seed.ts`） |
| `packages/prompts.chat/` | 发布到 npm 的 CLI/SDK 包（bin: prompts-chat；子模块 variables/similarity/quality/builder/parser，见其 `package.json` exports） |
| `packages/raycast-extension/` | Raycast 桌面快捷检索扩展 |
| `plugins/claude/`、`.claude-plugin/` | Claude Code 插件与插件市场清单（`marketplace.json`） |
| `prompts.csv` / `PROMPTS.md` | 提示词数据的两个权威导出格式（本仓库的"内容本体"） |
| `public/` | 97MB 静态资源（仓库体积大的主因，tarball 87MB） |
| `scripts/` | 安装向导 setup.js、贡献者生成、翻译检查、书籍 PDF 生成等 11 个脚本 |
| `tests/`、`src/__tests__/` | Vitest 测试（61 个测试文件） |

组织方式评述：npm workspace 式 monorepo（根 `package-lock.json` 统一管理），"平台应用为主体 + 两个子包 + 数据导出文件随根目录提交"。对内容仓库而言罕见的重量级工程结构。

## 5. 核心架构与关键实现

**（1）数据回流管道：网站是源头，仓库是镜像。** `CONTRIBUTING.md` 明确"在 prompts.chat 上提交后自动同步到本仓库"；`src/app/prompts.csv/route.ts` 从 Postgres 动态生成 CSV（`revalidate = 3600`），贡献者标识优先 GitHub 用户名、导入账号用 `@unclaimed.prompts.chat` 后缀用户名以防冒名。`prisma/seed.ts` 反向从 `https://prompts.chat/prompts.json?full_content=true` 拉数据做自托管种子——官方站即数据权威源。

**（2）MCP Server（`src/pages/api/mcp.ts`，1530 行）。** 基于 `@modelcontextprotocol/sdk` 的 StreamableHTTP 传输，实现 ListPrompts/GetPrompt 等原语，API Key 认证带 5 分钟内存缓存（`authCache`），并区分通用/工具调用/写入/AI 四级限流（引用 `src/lib/rate-limit.ts` 的 `mcpGeneralLimiter` 等）。这是项目从"网页"走向"AI 基础设施"的关键一跳。

**（3）可配置平台内核。** `prompts.config.ts` 通过 `defineConfig`（实现在 `src/lib/config/`）声明品牌、主题（radius/variant/density/色彩）、认证、i18n、功能开关（aiSearch、mcp、changeRequests、comments 等），配合 `PCHAT_*` 环境变量实现一套代码多种部署形态——这是支撑 self-host 卖点的架构核心。

**（4）多文件 Skill 序列化（`src/lib/skill-files.ts`）。** 用 ASCII 控制字符 `\x1F`/`\x1E` 作为文件分隔符把多文件 skill 存进单个文本字段，注释明确说明该字符不可能出现在正常文本中"making them injection-proof"，是针对提示注入的数据设计细节。

**（5）CLI/SDK 包（`packages/prompts.chat/`）。** tsup 打包、双格式导出，CLI 用 React（ink 风格组件 `src/cli/components/`）实现浏览/运行提示词的终端 UI，并内建 6 个测试文件（builder/parser/quality/similarity/variables/platforms）。

**（6）Bot 集成层（`src/lib/bots/`）。** `schema.ts` 定义 BOT_PLATFORMS 注册表与平台判别函数，数据库侧有 BotPairing 模型与 BotPlatform 枚举（`prisma/schema.prisma`），把提示词库接入第三方聊天机器人平台；`src/app/api/bots/` 提供配对 API。

## 6. 构建与运行

来自 `package.json` scripts 与 `README.md`/`DOCKER.md`：

```bash
# 开发
npm install && npm run setup   # setup 向导（scripts/setup.js）配置品牌/认证
npm run dev                    # localhost:3000
npm run build                  # prisma generate && next build

# 数据库
npm run db:setup               # generate + migrate + seed
npm run db:studio

# 测试与质量
npm test                       # vitest run
npm run lint                   # eslint

# Docker 一键自托管（compose.yml：postgres:17 + ghcr.io/f/prompts.chat:latest）
docker compose up -d           # 端口默认 4444

# CLI / MCP
npx prompts.chat               # 终端浏览提示词
npx prompts.chat new my-library  # 脚手架新自托管实例
```

## 7. 收录范围、组织方式与条目规模

- **收录范围**（`prompts.csv` 实测统计）：共 2,744 条提示词；按 type 分 TEXT 2,367 / STRUCTURED 356 / IMAGE 21；标记 for_devs=TRUE 的 204 条；distinct 贡献者 1,166 人。范围覆盖开发者角色（Ethereum 开发者、Linux 终端等）、写作、教育、图像生成（IMAGE 类含摄影风格提示词）等泛用场景。
- **组织方式**：线上按 Category/Tag 组织（`prisma/schema.prisma` 的 Category/Tag/PromptTag 模型，`src/app/categories/`、`src/app/tags/` 路由），导出层面则是扁平 CSV + 折叠式 Markdown（`PROMPTS.md` 每条一个 `<details>` 块，含标题、贡献者署名与代码块）。CSV 无分类列，分类仅存在于平台数据库——导出格式刻意保持向后兼容（act/prompt 两列是 2022 年的原始格式）。
- **质量机制**：ChangeRequest 版本化（`prompts.config.ts` features.changeRequests、schema 中 ChangeRequestStatus 枚举）、PromptVersion 历史举报（PromptReport/ReportReason 枚举）、AI 质量检查（`src/lib/ai/quality-check.prompt.yml`）。

## 8. 更新机制与自动化

- **提示词贡献**：网站提交 → 自动同步进仓库（`CONTRIBUTING.md`），PR 直投仅限代码/文档修复；仓库对广告与跑题 issue 采取零容忍并举报 spam（`CONTRIBUTING.md` Issue Policy 节）。
- **GitHub Actions**（`.github/workflows/`）：`ci.yml`（push/PR → Node 24 + npm ci + lint + vitest）；`update-contributors.yml` 每日 3 点 UTC 跑 `scripts/generate-contributors.sh` 重建贡献者列表后 force-push 并同步 Hugging Face；`docker-publish.yml` 发布 ghcr 镜像；`reset-credits.yml` 定期重置 AI 额度（对应 `src/app/api/cron/reset-credits/`）。
- **维护活跃度**：pushed_at 为分析前一日；治理文档 `GOVERNANCE.md` + `OWNERS.md` 定义 Project Lead（Fatih Kadir Akın，`.claude-plugin/marketplace.json` 中署名）+ maintainers 的 lazy consensus 决策模式。

## 9. 姊妹项目与生态位

- **数据集镜像**：Hugging Face `fka/prompts.chat`，README 称为 most liked dataset（`README.md`）。
- **自有生态闭环**：npm CLI、MCP Server（远程 `prompts.chat/api/mcp` + 本地 npx 两种接入，`README.md` Integrations 节）、Claude Code 插件市场（`/plugin marketplace add f/prompts.chat`）、Raycast 扩展、iOS App 与 Chrome 扩展（`prompts.config.ts` 中 appStoreUrl/chromeExtensionUrl）、儿童提示词学习游戏 Promi（`src/app/kids/`、`src/lib/kids/`）、付费电子书（Gumroad 链接，`README.md`）。
- **生态位**：在 awesome-list 系提示词仓库中它是事实上的"始祖与流量霸主"（awesome 徽章 + Forbes/Harvard 引用，`README.md`）；与后起垂直站相比，它的差异化是"数据公有领域（CC0）+ 全栈可自托管 + AI 工具链原生集成"。商业上靠 Neon/Cognition/Sentry 等赞助位与电子书变现（`prompts.config.ts` homepage.sponsors）。

## 10. 为什么能涨到 17 万星与总结评价

涨星逻辑（均可在仓库内找到证据）：

1. **时间卡位**：2022 年 12 月 ChatGPT 上线同期发布，是第一个提示词库（README "First prompt library"），吃到最大一波搜索与社交流量；
2. **权威背书**：awesome-list 徽章 + Forbes/Harvard/Columbia 引用（`README.md`）带来持续引用与长尾流量；
3. **低门槛参与**：网站提交自动署名同步进仓库（`CONTRIBUTING.md`），把 1,166 名 CSV 贡献者变成星星的传播者；
4. **持续自我更新**：从 README 清单到平台、数据集、MCP/插件，每个 AI 工具形态浪潮都重新进入视野，避免"死清单"式过气。

亮点：

- 数据双许可清晰——代码 MIT、数据 CC0 公有领域（`LICENSE`、`LICENSE-MIT`、`LICENSE-CC0`）；
- 工程质量远超同类清单仓库：61 个测试文件、MCP 四级限流、防注入的 skill 存储设计；
- 生态覆盖 CLI/MCP/插件/自托管全链路，接入形态齐全；
- 治理与贡献机制文档完备（`GOVERNANCE.md`、`OWNERS.md`、`SECURITY.md`）。

风险与局限：

- `public/` 达 97MB，仓库克隆/下载沉重（tarball 87MB 的主因）；
- `prompts.csv`/`PROMPTS.md` 与平台数据库双写，一致性依赖自动同步流程，仓库快照可能滞后；
- NextAuth v5 仍为 beta，Node 24/Next 16 等新版本对自托管者有门槛；
- CSV 存在历史脏数据：早期行 type 列为空、for_devs 列大小写混杂（实测统计所见）。

总评：prompts.chat 是内容型仓库工程化的极致样本——它证明了一份好的开源数据可以长成一个平台、一个数据集和一组 AI 原生工具。适合三类人：想直接抄提示词的普通用户（用 CSV/PROMPTS.md 即可）、想自建团队提示词库的工程团队（self-host 路径成熟）、研究社区数据型产品如何演进的开发者（其回流管道与生态设计值得精读）。
