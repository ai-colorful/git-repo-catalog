# virgiliojr94/book-to-skill 项目分析

> 星数 38590（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：把任何技术书籍 PDF（以及 EPUB/DOCX/HTML/Markdown/RTF/MOBI）转换成一个结构化的 AI agent skill，让 Claude Code、Copilot CLI、Amp、Codex 等 agent 在工作时按需查阅书本内容，号称回答一个问题的 token 消耗比把书直接塞进上下文少 24–51 倍。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | book-to-skill（v2.0.1） | pyproject.toml |
| 星数 / fork | 38590 / 4108 | gh api 2026-11-21 |
| 主语言 | Python（>=3.9，零网络请求核心包） | pyproject.toml |
| 许可证 | MIT | LICENSE.md |
| 创建时间 | 2026-05-01 | gh api（6.5 个月到 38.6k 星） |
| 最近活跃 | 2026-11-19 push，同日发布 v2.0.0/v2.0.1 | gh api releases |
| 贡献者 | 60 人 | gh api contributors |
| 维护者 | Virgilio Junior（virgiliojr94），巴西支付公司 Stone/Pagar.me 的 SRE | gh api users/virgiliojr94 |

## 2. 项目是什么，解决什么问题

痛点是"买过的好书三个月后想不起来第 7 章讲了什么"（README.md "Why"节）。现有替代方案都低效：直接搜 PDF 只得到页码列表；让 agent 回答书的内容要么幻觉要么没上下文；自己记笔记则写完就再也不看。book-to-skill 的解法是**把书变成结构化 skill**：转换一次，之后在 agent 里输入 `/your-book-slug replication` 之类的话题提问，agent 就会读取对应章节文件、基于真实内容回答。

核心架构是"两半"（README.md "How it works"、docs/architecture.md）：
1. **确定性 Python 提取器**（`book_to_skill/` 包）：文档 → 干净文本 + 元数据。这一半完全不调 LLM、零网络请求，只负责解析格式、清洗排版噪音、检测章节结构。
2. **规范驱动的生成器**：仓库根目录的 `SKILL.md`（895 行）本身就是被安装的 skill 定义，宿主 agent（Claude Code 等）加载它后，按其中 Step 0–11 的规范**自己充当 LLM**去完成蒸馏——抽取框架、决策规则、反模式，逐章生成摘要文件。项目 Python 代码里没有任何 LLM API 调用，这是它与常见"PDF→摘要管线"工具的本质区别。

生成的 skill 落在跨 agent 技能根目录 `~/.agents/skills/<slug>/`，一套文件同时被多个宿主发现（SKILL.md 头部注释列出了各家的 skill root 兼容表）。产物结构是渐进式披露（progressive disclosure）设计：`SKILL.md` 核心（约 4000 token，心智模型 + 章节索引）常驻上下文，`chapters/ch01-*.md`（每章约 1000 token）、`glossary.md`、`patterns.md`、`cheatsheet.md` 按需加载（README.md "What it generates"表）。

除了书，README "Beyond books" 节强调输入可以是任何结构化散文：内部文档/runbook、品牌手册、论文集、RFC 规格。v2.0 又加了 Search & Read（MCP 模式）：不用本地文件，通过 `libgen-mcp` 等书源 MCP provider 按标题/ISBN/DOI 搜索并逐章增量合成，不在磁盘留全本（README.md "Search & Read (MCP)"）。

## 3. 技术栈与依赖

- **语言/工程**：纯 Python（`pyproject.toml`，hatchling 构建，requires-python >=3.9），console script 入口 `book-to-skill = "book_to_skill.cli:main"`。
- **依赖策略是"全可选 + 优雅降级"**：核心包只用标准库，按格式声明 extras（pyproject.toml `[project.optional-dependencies]`）：
  - PDF：`pdftotext`（poppler，外部二进制）→ `pypdf` → `pdfminer.six` 三级回退；技术类书用 `docling`（版面感知，约 1.5s/页，保留表格/代码块）；`pdf-inspector>=1.15,<2` 作为可选加速/信任层。
  - EPUB：`ebooklib` + `beautifulsoup4`，回退 stdlib `zipfile`。
  - DOCX：`python-docx`，回退 stdlib ZIP/XML；HTML：`trafilatura`/`bs4`，回退 stdlib `html.parser`；RTF：`striprtf`，回退正则；MOBI/AZW：Calibre `ebook-convert` 外部程序。
- **质量工具**：pytest（testpaths=tests）、ruff（只门禁 E9/F，风格不卡）。
- **无任何 LLM SDK / 无网络依赖**：`book_to_skill/` 包零网络请求是项目反复强调并测试的约束（CI 有 dependency-free smoke leg，见 .github/workflows/ci.yml）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 | 出处 |
|---|---|---|
| `SKILL.md` | 被安装的 skill 本体 + Step 0–11 生成器规范（895 行） | SKILL.md |
| `references/search-and-read.md` | 懒加载的 Mode 2 MCP Search & Read 协议 | 顶层结构 |
| `scripts/extract.py` | 薄入口 shim（UTF-8 强制、sys.path 修正、禁止写 __pycache__） | scripts/extract.py |
| `book_to_skill/` | 核心包：cli、config、dependencies（可选依赖探测/--check）、sanitize、utils（2145 行，章节结构检测主逻辑）、pdf_inspector_integration | 包内文件 |
| `book_to_skill/parsers/` | 7 个格式解析器：pdf(345 行)、epub、docx、html、rtf、calibre、text | parsers/ |
| `book_to_skill/remote/` | MCP 模式离线件：discovery(686 行，搜索结果归一化)、reader(465 行，outline/section/cursor 读法)、policy(版权/确认门禁)、checkpoint、coverage、provenance | remote/ |
| `tools/` | discovery_tax.py（token 成本测量）、scan_generated_skill.py（生成物注入扫描）、validate_skill.py（6 宿主视角校验）、evals/ | tools/（部分 export-ignore） |
| `tests/` | 88 个测试文件（pytest），含 mock_mcp 离线 E2E | gh api 文件树 |
| `docs/` | mkdocs 站点：architecture/performance/usage/install/faq/how-it-works | docs/（export-ignore） |

组织方式评述：单包单体，skill 定义（SKILL.md）与辅助 Python 包同仓——这是"Agent Skills 开放标准"仓库的典型形态（仓库本身就是一个可 `npx skills add` 的 skill）。`.gitattributes` 将 tests/docs/CI 等 25 条路径 `export-ignore`，因此 release tarball 只含运行必需文件，开发者仓库才有全量（.gitattributes 已核对，本次分析经 gh api 文件树补齐）。

## 5. 核心架构与关键实现

### 5.1 "Python 提取 + 宿主 LLM 合成"的分工（架构精髓）

`SKILL.md` 规范把整个转换拆成 agent 可执行的流程：Step 0 作用域/重跑防护 → Step 2 运行 extract.py → Step 2.5 预检成本估算 → Step 2.6 大书 REPL 式访问 → Step 3 结构分析 → Step 7 逐章生成 → Step 8 词汇表/模式/速查表 → Step 9 主 SKILL.md → Step 9.5 注入扫描 → Step 10 报告 → Step 11 可选发布。其中最能体现工程意识的是 Step 2.6（SKILL.md:311）：

```text
For books over ~50k tokens, prefer programmatic probes over Read(full_text.txt):
  wc -w "$FULL_TEXT_PATH"
  grep -n -E "^\s*(Chapter|CHAPTER)\s+[0-9]+" "$FULL_TEXT_PATH" | head -40
  sed -n '<start>,<end>p' "$FULL_TEXT_PATH"
```

灵感来自 RLM 范式：一本 200 页的书约 75k token，逐章重读 28 遍要 200 万输入 token，用 grep/sed 只拉相关切片让生成成本正比于输出而非源文本。Step 2.5 则把生成成本预先报给用户（输入 ≈ 提取 token ×1.3，输出 ≈ 章节数 × 每章预算 + 常量），并在 docs/performance.md 给出实测：整书转换约 1 美元（Sonnet 4.5 价）。

### 5.2 PDF 提取管线与排版清洗（book_to_skill/parsers/pdf.py + utils.py）

pdftotext `-layout` 输出按 `\f` 分页后做三件精细清洗（parsers/pdf.py:38 clean_pdftotext）：统计出现在 >半数页面的首/尾行判定为页眉页脚样板并剥除；剥除孤行页码（含罗马数字，用"形状"正则而非字符枚举，避免误删 MIX/CIVIL 这类真词，注释里写明了这个回归的成因）；连接跨行连字符断词。章节结构检测在 `utils.py` 的 `detect_structure()`（utils.py:1198），支持多语言章节数字（中文数字 `_cn_numeral_to_int`、波斯语序数、罗马数字）与 PDF outline 应用（`apply_pdf_outline`，utils.py:1290）；识别失败时让 agent 退化为结构式标题切分。代码注释普遍引用 issue 号并记录失败案例（如 #113 章首标题恰为页眉被误删的修复），维护痕迹非常认真。

### 5.3 提示注入防护（book_to_skill/sanitize.py）

对提取文本做不可见字符消毒，按攻击形态分组（sanitize.py:6-90）：零宽/不可见间隔符（U+200B/200C/200D/2060/FEFF/软连字符等 12 个）、双向格式控制符——即 Trojan Source 类攻击（CVE-2021-42574，注释明确引用：人工审阅者看到的渲染顺序与模型读到的逻辑顺序不一致）、不可见字母（韩文填充符等可绕过空白归一化）、弃用注释格式控制符、变体选择符；另有 Kangxi 部首折叠等 CJK 归一化（tests/test_cjk_radical_normalization.py 对应）。`remote/` 全部模块（discovery/reader/policy）都过同一 sanitize 出口。生成完成后 Step 9.5 还会用 `tools/scan_generated_skill.py` 反向扫描生成物，形成进站+出站两道闸。

### 5.4 MCP Search & Read 的离线层（book_to_skill/remote/）

v2.0 新增。设计上把"网络"完全隔离在宿主 agent 的 MCP 调用里，Python 包只做确定性后处理：`discovery.py` 归一化各家 provider 返回的搜索/详情载荷（兼容 results/items/hits/books 等包装键、ISBN/DOI/arXiv URL 识别、开放获取来源白名单 arXiv/PubMed/Gutenberg 等）；`reader.py` 支持 outline/section/cursor 三种增量读法；`policy.py` 通过 `BOOK_TO_SKILL_REMOTE_POLICY` 环境变量强制"读前显式确认 + 默认私有发布 + MCP 临时缓存警告"（policy.py DEFAULT_POLICY）。发布环节（SKILL.md Step 11）有著名的"裸词 public"规则：可见性必须由用户的裸词 `public` 回答确认，任何含 "it's public domain" 的句子都不算——版权门禁设计得非常防御性。

## 6. 构建与运行

```bash
# 安装 skill 本体（跨 agent 通用，README "Install"）
npx skills add virgiliojr94/book-to-skill
# 或手动 clone 到 skills 目录
git clone https://github.com/virgiliojr94/book-to-skill.git ~/.claude/skills/book-to-skill

# 使用（agent 内）
/book-to-skill <path|folder|glob> [skill-name]
/book-to-skill search "Pro Git"

# 独立 CLI（pip 安装后）
book-to-skill <paths> --mode technical|text --install-missing ask
python3 scripts/extract.py --check          # 检查各格式提取器就绪度

# 开发
pytest tests/ -q                            # CI 同款（tests/ 仅开发者仓库有）
ruff check --select E9,F book_to_skill/ scripts/ tests/ tools/
```

## 7. 测试与工程质量

- 规模：tests/ 共 88 个文件，README 与 Sponsor 文案宣称 1,100+ 用例（未逐一复核，文件数属实）。覆盖风格偏向真实回归：每种解析器缺陷一个文件（test_rtf_surrogate_pairs、test_epub_utf16、test_letter_spaced_outline_headings、test_pdf_page_number_detection 等），安全类独立成组（test_sanitize_bidi_controls、test_remote/test_security_boundaries），并有 mock MCP 全链路 E2E（tests/remote/test_e2e_mock_mcp.py）。
- CI（.github/workflows/ci.yml，export-ignore 内）：Python 3.9–3.13 矩阵（3.9 钉在 ubuntu-24.04 以防镜像滚动移除该版本）、ruff 门禁、optional-deps leg（专门跑 importorskip 的可选依赖分支）、dependency-free smoke job（守护零依赖承诺）、CodeQL、mkdocs 文档部署、Dependabot；Action 全部 SHA 钉死。
- 代码质量观察：注释密度高且多为"为什么"而非"是什么"；`ponytail:` 标记的已知权衡（如连字符断词的字典级修复留给未来）表明作者对技术债有自觉管理。

## 8. 社区与维护状态

- 节奏：2026-05-01 建仓，CHANGELOG 显示 v1.1.0（6 月）→ v1.2.0（6 月）→ v1.3.0（7 月）→ v1.4.0（8 月）→ v2.0.0/2.0.1（11-19），约每月一版，v2.0.0 是大版本（MCP Search & Read）。
- 增长：6.5 个月 38,590 星（gh api 核实），期间挂上 Trendshift 双徽章（README.md），提供英/俄/中/日四语 README，配套 use-cases 仓库收集真实案例（如扫描版 PDF 卡壳的案例 #130）。这是 AI skills 热潮中"细分赛道 + 实用刚需 + 可复现的性能数据"组合的典型爆发曲线。
- 贡献者 60 人，open issues 22；PR 有门禁模板（CI 对 PR body 变更重跑）。维护者 Virgilio Junior 为巴西 Stone 支付体系 SRE，个人时间维护并公开寻求赞助（README Sponsors 节、BACKERS.md）。
- 多宿主适配是持续维护重点：SKILL.md 与 tests 里对 Hermes/OpenClaw/OpenCode 各有专门兼容与测试（test_hermes_host_support.py、test_openclaw_host_support.py 等）。

## 9. 亮点与风险

**亮点**
1. 架构分工干净：确定性 Python 管"提取+安全"，宿主 LLM 按 SKILL.md 规范管"合成"——无 API key、零网络、天然多模型多宿主。
2. 安全工程超出同类水准：进站不可见字符/双向控制符消毒 + 出站生成物扫描 + 版权确认门禁 + 默认私有发布，把"书→模型"链路上的提示注入面想得很全。
3. 成本意识贯穿始终：预检成本估算（Step 2.5）、REPL 式大书访问（Step 2.6）、逐章按需加载的渐进披露，token 经济学可复现（tools/discovery_tax.py）。
4. 渐进披露的产物设计（常驻 4K 核心 + 1K/章按需）是 context engineering 的教科书式落地。
5. 跨 7 个 agent 宿主的兼容矩阵 + 88 个测试文件的真实回归式测试。

**风险/局限**
1. 生成质量依赖宿主模型与 SKILL.md 规范的执行一致性——LLM 步骤没有确定性保证，弱模型可能产出不合格 skill（项目用 Step 9.5 扫描缓解而非消除）。
2. 扫描版 PDF 无法直接处理，需先 ocrmypdf（README Requirements 明确）；无文本层的 PDF 会提前报错。
3. 版权灰色地带：工具本身合法（不携带任何书内容），但 MCP Search & Read 依赖 libgen-mcp 这类书源，面向受版权保护书籍的使用处于作者明确声明的"用户自负"区间。
4. 6.5 个月冲到 38.6k 星的热度存疑部分来自 AI skills 热潮红利，热度回落后的长期维护投入待观察（作者已转向赞助模式）。

## 10. 总结评价

book-to-skill 是本轮 "Agent Skills 开放标准 + context engineering" 浪潮中完成度最高的细分工具之一：它没有把"PDF 转 skill"做成又一条 LLM 摘要流水线，而是拆成"零依赖 Python 提取器 + 宿主 agent 执行的 895 行生成规范"，用渐进披露解决"书读不完"的 token 经济学，用进站/出站双向消毒和版权门禁补齐了这条链路上别人普遍忽略的安全面，再用 24–51 倍的实测数据支撑增长叙事。适合两类人：想把私有文档变成 agent 常驻知识库的重度 agent 用户（直接用），以及想学习 skill 规范设计、提示注入防护和 agent 工作流工程化的开发者（读 SKILL.md、sanitize.py、remote/policy.py 的价值高于读解析器）。
