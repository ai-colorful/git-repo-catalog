# type-challenges/type-challenges 项目分析

> 星数 47500（快照 2026-10-24）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：TypeScript 类型体操题库 + 借助 TypeScript 编译器本身的"类型级在线判题"系统，由 Anthony Fu（antfu）于 2020 年 7 月创建。

| 项目 | 值 |
| --- | --- |
| 名称 | type-challenges/type-challenges（package.json 中 repository 指向 antfu/type-challenges，已迁移为组织仓库） |
| 星数 | 47500（快照）；查询时 48557（gh api，2026-11-04） |
| 主语言 | TypeScript（题目与工具链全部为 TS） |
| 许可证 | MIT（`LICENSE`、`package.json` 均声明） |
| 创建时间 | 2020-07-23（gh api `repos/type-challenges/type-challenges`） |
| 最近 push | 2026-10-26（gh api），持续活跃维护 |
| 贡献者 | 约 240 人（gh api contributors 分页推算） |
| 官网 | https://tsch.js.org （Netlify 跳转站，`scripts/build.ts`、`scripts/toUrl.ts`） |
| Forks | 5253；open issues 33337（含全部"答案 issue"，见第 5 章） |

## 2. 项目是什么，解决什么问题

type-challenges 是一套系统化的 TypeScript 类型编程练习题。作者在 `README.md`（Intro 段）中的定位是：帮助开发者理解 TypeScript 类型系统的运作方式、学会编写自己的类型工具（utility types），并形成一个讨论真实世界类型问题的社区。README 开头那行被删除线的"by the power of TypeScript's well-known Turing Completed type system"（`README.md` 第 25 行）点明了项目的趣味核心——TS 类型系统图灵完备，可以把类型当作一门编程语言来解题。

它解决的问题是：日常 TS 使用者大多停留在标注类型层面，对泛型、条件类型、映射类型、模板字面量类型、`infer` 等高级特性缺乏练习场景。该项目把这些特性拆解成从 warm-up 到 extreme 五个难度的 190 道题（统计见第 4 章），例如用类型实现 `Pick`/`Omit`/`Awaited`，进阶到类型级斐波那契（`questions/04182-medium-fibonacci-sequence/`）、类型级大数乘法（`questions/00517-extreme-multiply/`）、类型级数独（`questions/31797-hard-sudoku/`）。

本仓库介于"内容型"与"工具型"之间：主体资产是 `questions/` 下 190 个题目的题目描述、模板与测试用例，但它同时自带一套完整的判题与运营工具链（`scripts/`、`utils/`、`.github/workflows/`）：

- 判题不依赖任何外部 OJ 服务，而是把"测试用例"写成类型级断言，由 `tsc` 在 strict 模式下直接报错判分（机制详见第 5 章）；
- 题目的提交、翻译、答案分享全部走 GitHub Issue/PR 流程，由 GitHub Actions 机器人自动解析 issue 并生成 PR（`scripts/actions/issue-pr.ts`）；
- `tsch.js.org` 只是一个 Netlify 短链跳转层（`scripts/build.ts` 生成 `_redirects`），把 `/4/play` 跳到携带压缩代码的 TypeScript Playground，把 `/4/answer` 跳到预填模板的新建 issue 页。

生态位上，它是"类型体操"这个概念的事实起源与最大社区：README 的 Recommended Readings 段收录了官方文档、社区文章、演讲、相关项目与书籍；周边还有 TypeScript Playground 插件（`@type-challenges/playground-plugin`，README 徽章链接）、VS Code 扩展（YRM.type-challenges，README 第 106 行链接）、多语言镜像学习资料，以及把本仓库判题工具发布为 npm 包的 `@type-challenges/utils`（`utils/package.json`）。

## 3. 技术栈与依赖

- 语言与运行时：全 TypeScript；工具链用 esno（`esbuild` 驱动的 ts 执行器）直接运行脚本，无编译产物（`package.json` scripts：`esno ./scripts/build.ts` 等）。
- 包管理：pnpm 8.12.1 + workspace（`packageManager` 字段与 `pnpm-workspace.yaml`，workspace 成员为 `utils`、`scripts`）。
- 核心依赖（根 `package.json` / `scripts/package.json`）：
  - `fast-glob`：扫描 `questions/` 目录（`scripts/loader.ts` 的 `loadQuizzes`）；
  - `js-yaml`：解析题目元数据 `info.yml`；
  - `lz-string`：把题目代码压缩进 TypeScript Playground URL（`scripts/toUrl.ts` 的 `toPlaygroundUrl`）；
  - `fs-extra`、`prompts`、`ansis`：文件操作与 CLI 交互（本地 playground 生成器）；
  - `@actions/core`/`@actions/github` + `@type-challenges/octokit-create-pull-request`：GitHub Actions 内直接调 GitHub API 自动建 PR；
  - `@vitalets/google-translate-api`、`limax`：题目 README 的机器翻译与 slug 生成（`scripts/translate.ts`、issue→PR 中的目录命名）；
  - `@antfu/eslint-config` + `eslint`：唯一的"质量门"（`eslint.config.js`，Markdown lint 已关闭）。
- 类型检查工具就是 TypeScript 本体（^5.3.3），同时也是"判题机"。题目要求在 strict 模式下作答（`README.md` Challenges 段注明；`tsconfig.base.json` 全局 `"strict": true`）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `questions/` | 核心内容：190 个题目目录，命名 `<编号>-<难度>-<slug>`（如 `00004-easy-pick`） |
| `questions/<题>/template.ts` | 作答起点模板，答案占位为 `any` |
| `questions/<题>/test-cases.ts` | 类型级测试用例（判题依据，见第 5 章） |
| `questions/<题>/README.md` + 4 个语言变体 | 题目描述；`info-header/footer` HTML 注释块由脚本生成 |
| `questions/<题>/info.yml`（+ zh-CN 变体） | 元数据：title/author/tags/related/difficulty |
| `scripts/` | 工具链 workspace 包 `@type-challenges/scripts`：加载器、构建器、README 索引生成、翻译 CLI、Actions 处理器 |
| `scripts/actions/` | GitHub Actions 入口：issue-pr / labeling / toggle-pr-with-issue 三个处理器（`scripts/actions/loader.ts` 分发） |
| `utils/` | 独立发布包 `@type-challenges/utils`（仅 `index.d.ts`），判题断言工具 |
| `site/` | tsch.js.org 落地静态页（`index.html` + 样式，Netlify 托管，`scripts/netlify.toml`） |
| `guides/` | 三篇专题攻略：`infer.md`、`key-in.md`、`recursive.md`（各带 zh-CN 版） |
| `.github/` | 4 个 workflow + 11 个 issue 模板（新题/答题，覆盖 en/zh-CN/ja/ko） |
| `README.*.md` ×5 | 项目主页索引，`<!--challenges-start/end-->` 区块由 `pnpm readme` 每日自动重建 |

组织方式：典型的"内容即数据"单仓（monorepo 含两个 workspace 包）。题目是第一公民，目录名本身就是结构化数据（编号/难度/slug 全部可从目录名解析，`scripts/loader.ts` 的 `loadQuiz` 用正则 `^(\d+)-(.+?)-.*$` 提取）。题目规模统计（`ls questions/`，2026-10-26 快照）：**共 190 题：warm 1 / easy 13 / medium 104 / hard 55 / extreme 17**。翻译覆盖：README 中文版 78 题、日文 65、韩文 48、葡语 9（`find questions -name "README.<locale>.md"` 统计），未覆盖语言回退英文（`loadLocaleVariations` 逻辑，`scripts/loader.ts`）。全仓 1088 个文件，其中 `questions/` 含 380 个 `.ts`（190 模板 + 190 测试）。

## 5. 核心架构与关键实现

### 5.1 判题机制：让 tsc 当 OJ（`utils/index.d.ts` + `test-cases.ts`）

判题的核心是三十行的类型工具库 `utils/index.d.ts`（发布为 `@type-challenges/utils`）。两块基石：

```ts
export type Expect<T extends true> = T
export type Equal<X, Y> =
  (<T>() => T extends X ? 1 : 2) extends
  (<T>() => T extends Y ? 1 : 2) ? true : false
export type IsAny<T> = 0 extends (1 & T) ? true : false
```

- `Equal<X, Y>` 用"泛型函数类型的双重条件类型比较"实现严格类型相等（连 `any`/`unknown`、可选性、修饰符差异都能区分），注释注明出处是 StackOverflow 的经典回答（`utils/index.d.ts` 第 12 行）；
- `Expect<T extends true>` 则把"断言为真"编码成类型约束：期望值不是 `true` 时约束失败，tsc 直接报错。

以 `questions/00004-easy-pick/test-cases.ts` 为例：

```ts
type cases = [
  Expect<Equal<Expected1, MyPick<Todo, 'title'>>>,
  // @ts-expect-error
  MyPick<Todo, 'title' | 'completed' | 'invalid'>,
]
```

判题流程因此是：用户在 `template.ts` 实现 `MyPick` → `pnpm generate` 把模板与测试合成一个文件放进 `playground/`（`scripts/generate-play.ts` 的 `formatToCode` 合并输出）→ IDE/tsc 的红线就是"未通过"，无红线即"通过"。负面用例用 `@ts-expect-error` 断言"必须报错"。这套机制没有任何自研服务，完全复用 TypeScript 编译器做判题引擎，是该仓库最巧妙的设计。`utils/index.d.test.ts` 用同样的机制对断言工具自身做了自测。

### 5.2 题目数据模型与多语言加载（`scripts/loader.ts`）

`loadQuiz(dir)` 把目录名解析为 `{ no, difficulty, path }`，再用 `loadLocaleVariations` 按 `supportedLocales`（`scripts/locales.ts`）加载 `info.yml`、`README.md` 的各语言变体，缺失语言回退默认英文；`readmeCleanUp` 会剥掉 README 中脚本生成的 header/footer 注释块，只留人工撰写的题面。整条工具链（构建索引、生成 playground、翻译、Actions）都建立在这一个加载器之上——题目数据只有一份事实源。

### 5.3 Issue → PR 全自动出题流水线（`scripts/actions/issue-pr.ts` + `.github/workflows/issue-pr.yaml`）

新题贡献不需要会 git：贡献者用 issue 模板 `.github/ISSUE_TEMPLATE/0-new.md` 提交（body 内含 yaml info / ts template / ts tests / question 四段代码块），带 `new-challenge` 标签后触发 Actions：

1. 机器人按 issue 标签判定语言（ja/zh-CN/en），用 `YAML.load` 与代码块解析出题目四件套，解析失败则自动回评"请依照模板修正"（`issue_invalid_reply` 多语言文案）；
2. 通过 `@type-challenges/octokit-create-pull-request` 的 `PushCommit` 向 `pulls/<issue号>` 分支提交 `questions/<目录>/` 四个文件，commit author 归属真实贡献者；
3. 自动 `pulls.create` 建 PR（打 `auto-generated` 标签）并在原 issue 回复 PR 链接与 Playground 预览徽章；issue 编辑后重跑会幂等更新同一 PR（`existing_pull` 查找逻辑）。

配套还有 `labeling.yaml`（自动打标签）、`toggle-pr-with-issue.yaml`（issue 关闭/重开联动 PR）、`index-update.yaml`（每日 cron 跑 `pnpm readme` + `lint --fix` 自动重建 README 索引并提交，"chore: update index"）。答题侧则完全"issue 即题解"：`scripts/toUrl.ts` 的 `toShareAnswerFull` 生成预填模板的答题 issue 链接，`toSolutionsFull` 生成的题解列表实际是 `issues?q=label:<no>+label:answer+sort:reactions-+1-desc`——用 GitHub issue + reactions 充当题解区与点赞排序。33337 个 open issue 主体就是这 190 题的社区题解库。

### 5.4 tsch.js.org 短链层与本地练习场（`scripts/build.ts`、`generate-play.ts`）

`pnpm build` 遍历全部题目 × 5 语言，为每题生成 `/编号`（题面）、`/编号/play`（Playground）、`/编号/answer`（答题 issue）、`/编号/solutions`（题解列表）等短链规则，写入 Netlify `_redirects`；Playground URL 用 lz-string 压缩完整代码（题目描述 + 模板 + 测试 + 依赖 utils）放进 `#code/` 片段（`scripts/toUrl.ts`）。本地练习则由 `pnpm generate` 交互式选语言后，把每题合成单文件写入 `playground/<难度>/<编号-题名>.ts`，并用 SHA-1 快照缓存（`.playgroundcache`）实现 `--keep-changes`——更新题目时不覆盖学员已写的答案。

## 6. 构建与运行

来自根 `package.json` scripts 与 `README.md` "Play Locally" 段：

```bash
pnpm install          # 安装依赖（pnpm 8，workspace）
pnpm generate [语言]  # 生成本地 playground/ 练习目录；--keep-changes / -K 保留已作答文件
pnpm readme           # 重新生成各语言 README 的题目索引（CI 每日自动跑）
pnpm build            # 生成 Netlify _redirects 短链表（scripts/dist/）
pnpm generate-play    # 同 generate（generate-play.ts）
pnpm lint             # eslint .
pnpm translate <编号> # 机器翻译题目 README（scripts/translate-cli.ts）
pnpm utils:release    # 发布 @type-challenges/utils（bumpp + publish）
```

无传统意义的"构建/测试命令"：题目判题 = 在 IDE/tsc 中打开 playground 文件看红线；判题不跑 CI（仓库没有对题目答案跑 tsc 的 workflow，通过性由答题者自验 + 社区 review 把关，`.github/workflows/` 4 个 workflow 均为运营自动化，未确认存在题目级 CI 校验）。

## 7. 测试与工程质量

- 测试形态特殊：全仓 380 个 `.ts` 中一半是 `test-cases.ts`（190 个），本质是 190 套类型级测试集；测试框架为零——`tsc` 即测试运行器，`Equal/Expect/@ts-expect-error` 即断言库。断言工具自身有自测（`utils/index.d.test.ts`，覆盖 Expect/Equal/IsAny/Alike 等行为）。
- 负面测试规范：题目普遍用 `// @ts-expect-error` 断言非法输入必须报错（如 `00004-easy-pick` 第 6-7 行），能防"绕过约束的 any 实现"；`IsAny`/`Alike`（`utils/index.d.ts`）专门用于识破 `any` 与修饰符差异。
- 工具链质量：统一走 `@antfu/eslint-config`（`eslint.config.js`），CI 每日 `lint --fix` 后自动提交；测试覆盖仅限类型层，脚本部分（loader/build/actions）无单元测试（未发现 `scripts/**/*.test.ts`）。
- 工程瑕疵：`tsconfig.base.json` target es2017 但 lib 为 ESNext；`generate-play.ts` 用同步 `fs` 为主的混合风格；翻译依赖非官方 Google Translate API 包，稳定性存疑。总体属于"内容为主、工具够用"的务实工程水准。

## 8. 社区与维护状态

- 活跃度极高：gh api 显示最近 push 为 2026-10-26（距分析日一周内）；open issue 3.3 万是常态化的题解/新题流水量。
- 贡献者约 240 人（gh api contributors 分页 Last: 241，per_page=1）；题库 190 题中仅少数（如 00004 Pick、00013 Hello World）由核心维护者出题，绝大多数编号即 issue 号（如 35991-medium-myuppercase），印证"社区出题、机器人建档"的运营模型（`scripts/actions/issue-pr.ts` 中 `no = context.issue.number`）。
- 维护结构：作者是 Anthony Fu（antfu，Vite/Vitest 核心），仓库 2020 年由个人仓库迁至 type-challenges 组织；release 节奏上本仓库不发版（private，`package.json` version 0.0.0），只有 `@type-challenges/utils` 单独发 npm（`utils/package.json`，当前 0.1.1）。
- 多语言社区：issue 模板与机器人回复覆盖 en/zh-CN/ja/ko（`scripts/actions/issue-pr.ts` 的 `Messages` 表），README 提供 5 语言；Discord 社区链接在 `README.md` 顶部。
- Star History 与 Contributors 图表见 `README.md` 尾部；具体贡献者排名未确认（tarball 无 git 历史）。

## 9. 亮点与风险

**亮点**

1. "编译器即 OJ"：把判题问题化归为类型检查问题，零服务端基础设施支撑起 4.7 万星规模的可玩性（`utils/index.d.ts` + `test-cases.ts` 模式）。
2. 极低贡献门槛的全自动运营：issue 模板 → 机器人解析 → 自动 PR → 自动翻译 → 每日自动索引，贡献者不需要接触 git（`scripts/actions/issue-pr.ts`、`scripts/translate.ts`、`index-update.yaml`）。
3. 内容体系完整且分层清晰：warm/extreme 五档 190 题，配套 `guides/` 三篇专题攻略与 Recommended Readings 资源导航（`README.md`）。
4. 多端练习入口：Playground 短链（含 `@type-challenges/playground-plugin`）、VS Code 扩展、本地 playground 生成器（带 `--keep-changes` 作答保护）三条路径覆盖不同人群。
5. 题目数据模型设计干净：目录名即元数据、单文件事实源、多语言回退机制，全部工具链复用同一加载器（`scripts/loader.ts`）。

**风险/局限**

1. 判题依赖自验：仓库 CI 不校验题目可解性，题目或测试用例出错（如依赖特定 TS 版本行为）只能靠 issue 反馈；`tsconfig.base.json` 的 TS ^5.3.3 与 Playground 实际版本可能存在行为差异。
2. 运营强耦合 GitHub：题解区（3.3 万 issue）、PR 流程、reactions 排序都寄生在 GitHub 平台上，无法导出为独立产品形态；`site/` 仅是跳转层。
3. 机器翻译质量与覆盖不均：zh-CN 仅 78/190 题、pt-BR 9 题，且 `@vitalets/google-translate-api` 为逆向 API，长期可用性未确认。
4. 学习曲线陡峭无路径依赖：medium 以上占比 62%（141/190），对新手缺渐进引导，需配合 guides 与外部资料。

## 10. 总结评价

type-challenges 用"让 TypeScript 编译器自己当判题机"这一极简架构，把类型体操从少数人的炫技变成有阶梯、有社区、有自动运营的大众练习场——190 道五档难度的题目、约 240 位贡献者、3.3 万条题解 issue 与一周内的最新 push 共同证明了这个模式的飞轮效应。它介于内容库与工具库之间，但代码部分（loader/build/actions 共约千余行）真正值得读的精髓只有两处：`utils/index.d.ts` 的类型断言技巧和 `scripts/actions/issue-pr.ts` 的 issue→PR 自动化。适合三类人：想把 TS 类型系统吃透的工程师（做题）、想出题或练英文技术写作的社区成员（走 issue 模板贡献）、以及想学习"用 GitHub 平台能力搭建零成本 OJ/社区运营系统"的开源运营者（读 scripts/ 与 .github/）。想在工程中复用其判题思路的团队，可直接依赖其发布的 `@type-challenges/utils` 断言包。
