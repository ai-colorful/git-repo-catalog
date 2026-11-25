# tldr-pages/tldr 项目分析

> 星数 64022（快照 2026-10-24）｜主语言 Markdown｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：社区共同维护的"简化版 man 手册"——为命令行工具提供以实用例子为主的帮助页，是传统 man page 的平易近人的补充（`README.md`）。

| 项目 | 事实 |
| --- | --- |
| 名称 | tldr-pages/tldr |
| 星数 | 64022（快照 2026-10-24） |
| 主语言 | Markdown（内容）；工具链为 Bash/Python/JavaScript |
| 许可证 | 内容 CC-BY 4.0；`scripts/` 目录 MIT（`LICENSE.md`） |
| 创建时间 | 2013-12-08（gh api repos/tldr-pages/tldr） |
| 默认分支 | main；最近 push 2026-10-31（分析当日，极活跃） |
| 最新 release | v2.3（2025-03-07，81 个资产：多语言 zip + PDF + 校验和） |
| 规模 | 英文页 7604，40 个翻译目录，全部 .md 共 40188 个（本仓库快照统计） |
| 贡献者 | ≥434（gh api contributors 分页 Link 头 last=434） |
| forks / open issues | 5609 / 293（gh api，2026-10-31） |

## 2. 项目是什么，解决什么问题

传统 man page 面向"完整参考"，动辄数千词，常见场景是查一个命令的常用用法。README 用 `man tar` 的真实输出举例：开篇讲的是"512 字节磁带 I/O 块大小"这类几乎没人再需要的内容。tldr-pages 的回答是：每个命令一页，只给 5~8 个最常用例子的 Markdown 帮助页，用 `tldr tar` 代替 `man tar`（`README.md`、`CONTRIBUTING.md`）。

页面格式高度统一：`# 命令名` 标题、`>` 描述与官网链接、若干 `- 例子说明:` 加反引号命令行，命令参数用 `{{占位符}}` 标注（样例 `pages/common/tar.md`）。这个格式由 `contributing-guides/style-guide.md` 严格约束（最多 8 个例子、优先长选项、逐步引入复杂度），并由专用 linter 强制执行。

覆盖范围极广：英文页按平台分 11 个目录——common（4743 页）、linux（2071）、osx（375）、windows（305）、dos、android、cisco-ios、freebsd、netbsd、openbsd、sunos（`pages/` 目录统计）。跨平台命令放 `common`，平台特有差异在对应平台目录放定制副本（`CLIENT-SPECIFICATION.md` "Platform" 一节）。

多语言是项目一大支柱：仓库里有 40 个翻译目录（`pages.zh/`、`pages.ko/`、`pages.es/` 等），中文已译 1553 页，韩语高达 6959 页、几乎与英文全集持平（各 `pages.*` 目录统计）。翻译进度有专门的可视化站点（README 链接 tldri18n）。

本仓库只承载"内容 + 内容工具链"。客户端是独立项目：官方 Python 客户端（tldr-python-client，PyPI `tldr`）、Rust 客户端 tlrc、Node.js 客户端（README 自述"fallen behind in updates"），另有免安装的 PWA 网页客户端 tldr.inbrowser.app；客户端必须实现的接口由仓库内的 `CLIENT-SPECIFICATION.md` 规范（详见第 5 章）。

## 3. 技术栈与依赖

- 内容：纯 Markdown，强制 LF（`.gitattributes`：`* text=auto eol=lf`），并显式设置 `linguist-detectable=true` 让 GitHub 语言统计把页面算作内容而非文档（`.gitattributes`）。
- Node.js 工具链（`package.json`）：`glob`（索引构建）、`markdownlint-cli`（Markdown 语法检查）、`tldr-lint ^0.0.23`（tldr 页面专用 linter，检查页面结构/占位符规则）、`husky`（pre-commit 钩子直接跑 `npm test`，`.husky/pre-commit`）。
- Python 工具链（`requirements.txt`、`scripts/test-requirements.txt`、`scripts/pdf/requirements.txt`）：`black`、`flake8`（脚本风格）、`pytest`（脚本单元测试）、`requests`（与 GitHub API 交互）；PDF 生成用 `markdown` + `weasyprint`（无 LaTeX 依赖，`scripts/pdf/render.py` 模块注释）。
- Bash 脚本：测试、构建、部署、PR 检查（`scripts/*.sh`），用 shellcheck 约束自身质量（`scripts/test.sh`）。
- CI/发布：GitHub Actions（7 个 workflow，见第 7、8 章），发布资产带 build provenance attestation（`.github/workflows/ci.yml`）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `pages/` | 英文页，按 11 个平台分子目录（common 4743 / linux 2071 / osx 375 / windows 305 等） |
| `pages.<lang>/` ×40 | 各语言翻译页，目录结构与英文页镜像（如 `pages.zh/common/tar.md`）；git 中另有 `pages.en -> pages` 符号链接作客户端别名 |
| `scripts/` | 内容工具链：测试、构建、索引、部署、PR 检查、翻译同步、PDF 生成（MIT 许可） |
| `scripts/pdf/` | `render.py`（Markdown→HTML→WeasyPrint PDF）+ 配套 CSS/字体 |
| `contributing-guides/` | 风格指南、维护者指南、AI 政策、git 入门、翻译模板、多语种风格指南 |
| `.github/` | 7 个 workflow、4 类 issue 模板（新增页/改页/翻译/lets-document）、CODEOWNERS |
| 顶层文档 | `README.md`、`CONTRIBUTING.md`、`CLIENT-SPECIFICATION.md`、`GOVERNANCE.md`、`COMMUNITY-ROLES.md`、`MAINTAINERS.md`、`LICENSE.md` |

组织方式评述：这是典型的"内容为 body、脚本为 limb"的单体仓——主资产是 4 万个按 `pages[.语言]/平台/命令.md` 三层路径组织的 Markdown 文件，路径本身承载元数据（语言、平台、命令名），无需数据库或配置文件。工具链脚本集中在 `scripts/` 一个目录、按单一职责命名（`set-alias-page.py`、`wrong-filename.py` 等），`scripts/README.md` 逐个说明用途和跨平台兼容性。客户端全部外置到独立仓库，本仓库用 CLIENT-SPECIFICATION 解耦内容与展示端，是很干净的架构决策。

## 5. 核心架构与关键实现

### 5.1 页面格式与路径即元数据

页面本质是带约定的 Markdown（`pages/common/tar.md` 实例）：

```markdown
# tar

> Archiving utility.
> More information: <https://www.gnu.org/software/tar/manual/tar.html>.

- [c]reate an archive and write it to a [f]ile:

`tar cf {{path/to/target.tar}} {{path/to/file1 path/to/file2 ...}}`
```

`{{占位符}}` 让客户端可以渲染出可替换的参数样式；描述里的 `[c]reate` 标记短选项字母，配合客户端 `--short-options/--long-options` 过滤（`CLIENT-SPECIFICATION.md`）。目录路径 `pages[.lang]/platform/command.md` 同时决定命令名、平台、语言，`build-index.js` 正是按路径解析（`scripts/build-index.js` 的 `parsePlatform/parsePagename/parseLanguage`）。

### 5.2 客户端规范 CLIENT-SPECIFICATION.md

21KB 的 RFC 2119 风格规范（`CLIENT-SPECIFICATION.md`），定义官方客户端 MUST/SHOULD 实现的 CLI 参数表（`-v/-p/-u/-l/-L/-S/-E`）、页面名归一化（`git checkout` → `git-checkout`、`eyeD3` → `eyed3`）、平台回退与语言协商规则。这使几十个社区客户端行为一致，是内容型项目里少见的"接口文档驱动生态"。

### 5.3 索引构建

`scripts/build-index.js`（86 行）用 glob 扫描 `pages*/**/*.md`，reduce 成 `{name, platform[], language[], targets[{os,language}]}` 的 JSON，供客户端离线检索；`npm run build-index > index.json` 生成后随 release 发布（`package.json`、`scripts/deploy.sh`）。

### 5.4 翻译同步自动化

月度 workflow（`.github/workflows/sync.yml`）批量运行四个 Python 脚本 `-S` 同步模式——`set-alias-page.py`、`set-more-info-link.py`、`set-page-title.py`、`set-see-also.py`——把英文页的别名页/链接/标题/参见段落同步到所有翻译，再用 peter-evans/create-pull-request 自动开"sync PR"。`scripts/check-pr.sh` 则在 PR 阶段检测"翻译页命令数与英文页不一致"等过时情况（脚本头注释列明 5 类检测）。这套机制是 40 个语言目录能长期不烂掉的关键。

### 5.5 构建与发布管线

`scripts/build.sh` 按 git diff 判断哪些 `pages*` 目录变更，只重建 `tldr.zip`、受影响语言的 zip 和 `index.json`；`scripts/deploy.sh` 把资产上传到最新 release（`gh release upload --clobber`）并重建 `tldr.sha256sums`（`scripts/build.sh`、`scripts/deploy.sh`）。即 release 是"滚动更新的资产池"，v2.3 标签下已有 81 个资产（gh api releases/latest）。

## 6. 构建与运行

来自 `package.json` 与 `scripts/README.md` 的实际命令：

```bash
npm ci                      # 安装 markdownlint-cli / tldr-lint / glob
npm test                    # = bash scripts/test.sh：全量 lint + 脚本检查
npm run lint-markdown       # markdownlint pages*/**/*.md
npm run lint-tldr-pages     # tldr-lint ./pages
npm run build-index         # 生成 index.json（stdout）
bash scripts/build.sh       # 构建 tldr.zip 与多语言 zip（需 git/zip）
pip install -r requirements.txt -r scripts/test-requirements.txt
pytest scripts/*.py         # 工具脚本单元测试
bash scripts/pdf/build-pdf.sh  # 生成 PDF（Linux/macOS，Windows 需 WSL）
```

终端用户侧不构建、直接消费：`pipx install tldr`（Python 客户端）、`brew install tlrc` / `winget install tldr-pages.tlrc`（Rust 客户端）、`npm install -g tldr`（Node 客户端）、或免安装网页版 tldr.inbrowser.app（`README.md`）。

## 7. 内容质量保障与测试

`scripts/test.sh` 是多层防线，CI 每次 push/PR 全量执行（`.github/workflows/ci.yml`）：

1. 页面层：`markdownlint`（4 万文件按 2000 个一批、`xargs -P` 按核数并行）+ 每个语言目录跑一遍 `tldr-lint`（专用 linter 查标题/占位符/链接格式）。
2. 脚本层：black、flake8、pytest、shellcheck 四件套检查 `scripts/` 自身。
3. PR 语义层：`scripts/check-pr.sh` 检测复制页、平台目录与 common 重复、翻译页无英文对应、翻译过时（命令数或内容与英文不一致）等异常；结果经 `send-to-bot.py` 由 tldr-bot 机器人直接评论在 PR 上（非阻塞建议）。
4. 钩子层：husky pre-commit 直接执行 `npm test`（`.husky/pre-commit`）。
5. 文案层：codespell workflow 拼写检查、`.markdownlint.json` 统一 Markdown 规则。

内容正确性（命令是否真的能跑）无自动验证，依赖人工审校与同行 PR 评审——这是此类项目的固有边界。

## 8. 社区与维护状态

- 活跃度极高：仓库创建于 2013-12-08，最近 push 在分析当日 2026-10-31；293 个 open issue、5609 forks、≥434 名贡献者（gh api）。
- 治理文档化程度罕见：`GOVERNANCE.md`（扁平化、do-ocracy、沟通全公开）、`COMMUNITY-ROLES.md`（社区角色定义）、`MAINTAINERS.md`（三级维护者：仓库协作者/组织成员/组织所有者，逐人列出任职起止与出处链接）。
- 贡献门槛明确（`CONTRIBUTING.md`）：需签 CLA；新收录命令所属项目须维护满一年（知名项目豁免）；明确反对用生成式 AI 直接产页（`contributing-guides/AI-policy.md`），机翻仅可作参考且须人工校对。
- 自动化维护：月度翻译同步 PR（`.github/workflows/sync.yml`）、月度维护者用户名有效性检查（`monthly-check.yml`）、labeler 与 bot-prevention workflow。
- 发布节奏：release 标签不常打（最新 v2.3，2025-03-07），但资产随每次 main 合并滚动更新（deploy.sh `--clobber`），81 个资产含全部语言 zip 与多语种 PDF。

## 9. 亮点与风险

**亮点**

1. 内容规模与语种覆盖无可匹敌：40188 个页面文件、英文 7604 页、40 种翻译，其中韩语 6959 页接近全集（`pages*/` 目录统计），是中文命令行用户的可靠母语资料源（`pages.zh/` 1553 页）。
2. 用 linter + 路径约定 + 专用规范（tldr-lint、CLIENT-SPECIFICATION.md）把"散文式内容"管理成了有接口契约的工程制品。
3. 翻译保鲜机制完整：月度自动同步脚本 + check-pr 过时检测 + tldr-i18n 进度站，解决了多语言 fork 常见的腐烂问题。
4. 治理与贡献体验成熟：角色文档、issue 模板按意图细分（新增页/改页/翻译/求文档）、机器人即时回评 PR。
5. 分发渠道多元：官方三语言客户端 + 社区客户端 Wiki 清单 + PWA 网页版 + release 资产（zip/PDF/校验和）+ build provenance attestation。

**风险/局限**

1. 翻译长尾严重：多数语言仅数百页（如 `pages.de/` 924 页），与英文 7604 页差距悬殊，小语种实际可用性有限。
2. 官方 Node.js 客户端被 README 自述"fallen behind in updates"，三个官方客户端体验不一致，新人选择成本高。
3. 命令示例的正确性靠人工审校，无法自动验证；`CONTRIBUTING.md` 对 genAI 的限制抬高了大规模补页的产能上限。
4. 仓库体量偏重（4 万文件、解压后 79MB、git 中还有 `pages.en` 符号链接，Windows 下解包需特殊处理），全量 clone 对贡献者是负担。

## 10. 总结评价

tldr-pages 是"用开源社区方式重做一件事并做到品类第一"的范本：它把 man page 的痛点翻译成一个极简的 Markdown 格式约定，再围绕这个约定建起了专用 linter、客户端规范、月度同步机器人、多渠道发布管线和成文治理体系，最终以 4 万页面、40 种语言、434+ 贡献者成为命令行学习事实上的公共基础设施。它 6.4 万星的原因不是技术复杂度，而是每个开发者都是它的目标用户、且每一次 `tldr <命令>` 的正向反馈都在强化生态。适合三类人"读"它：想给命令行工具写文档的人（学 `contributing-guides/style-guide.md`）、想运营大规模多语言内容项目的维护者（学它的同步/检查/治理机制）、想实现客户端的开发者（读 `CLIENT-SPECIFICATION.md`）。普通用户直接装 tlrc 或用网页版即可受益。
