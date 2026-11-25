# jgm/pandoc 项目分析

> 星数 38500（快照 2026-10-24）｜主语言 Haskell｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：Pandoc 是"通用标记语言转换器"——任何文档格式进来，任何格式出去，是文档转换领域事实上的瑞士军刀。

| 项目 | 事实 |
|---|---|
| 名称 | jgm/pandoc |
| 星数 | 38500（快照）；gh api 实测 2026-11-05 为 46808 |
| 主语言 | Haskell（核心库 238 个 .hs，约 10.3 万行，src/Text/Pandoc） |
| 许可证 | GPL-2.0-or-later（pandoc.cabal license 字段） |
| 版本 | 3.13（pandoc.cabal；changelog.md 首条 "pandoc 3.13 (2026-10-26)"） |
| 最近活跃 | 默认分支 main，最近 push 2026-11-04（gh api repos/jgm/pandoc） |
| 作者 | John MacFarlane（jgm，伯克利哲学教授），2006 年发起 |
| 格式支持 | 51 种输入格式、73 种输出格式（README.md 中两个格式列表实测计数） |

## 2. 项目是什么，解决什么问题

文档格式之间两两转换的组合复杂度是 N×M——每种格式都要为其它每种格式各写一个转换器。Pandoc 的核心思路是定义一个统一的中间表示（Pandoc AST），让每种格式只实现"解析进 AST"（Reader）和"从 AST 生成"（Writer）两件事，把 N×M 降为 N+M。这个 AST 定义被单独抽成 `pandoc-types` 包（pandoc.cabal 第 574 行 `pandoc-types >= 1.23.1.2 && < 1.24`），版本极其稳定，使得生态中大量第三方工具（编辑器插件、静态站点生成器、zotero 引用工具等）可以直接依赖这个稳定的中间格式。

覆盖范围极广：轻量标记（各方言 Markdown、RST、AsciiDoc、Org、djot）、Word/ODT/RTF 等办公格式、LaTeX/ConTeXt/Typst 等 TeX 系、EPUB 电子书、MediaWiki 等十余种 wiki 语法、BibTeX/BibLaTeX/CSL 等参考文献格式、Jupyter notebook、PowerPoint、reveal.js 幻灯片，甚至 InDesign ICML 排版格式（pandoc.cabal 的 description 字段完整列出）。数学公式可以在 TeX、MathML、Word 公式之间互转。

Pandoc 还是可编程的转换平台：通过 `--filter` 机制挂载 Lua 脚本或外部 JSON 过滤器，在 AST 层面改写文档；通过 Lua 还可以编写完全自定义的 reader/writer（MANUAL.txt 8265 行的篇幅本身就是这种可配置性的证明）。引用文献方面内置 citeproc 引擎，配合 CSL 样式实现自动引注和参考文献生成。

对学术写作者，Pandoc 是"用 Markdown 写、按期刊要求发布"工作流的支点；对开发者，它是无数文档工具（Quarto、pandoc-crossref、各类 CMS 导入器）的地基。这种"基础设施"地位是它积累近 4 万星的根本原因。

## 3. 技术栈与依赖

- 语言与构建：Haskell + Cabal（多包工程，cabal.project），另有 stack.yaml、Nix flake（flake.nix）、Windows/Linux/macOS 打包目录
- 关键自研依赖：
  - `pandoc-types`：AST 定义（独立 Hackage 包，不在本仓库）
  - `commonmark-hs`、`typst-hs`：作者自维护，cabal.project 以 `source-repository-package` 锁定 git tag
  - `citeproc`：引注处理引擎；`hslua`：Lua 2.x 嵌入（pandoc-lua-engine）
- 主要第三方库：aeson（JSON/AST 序列化）、skylighting（语法高亮）、texmath（公式转换）、doctemplates（模板引擎）、zip-archive（docx/odt/epub 容器）、yamlet（3.13 起替代 libyaml 的纯 Haskell YAML 解析器，changelog.md）
- 运行形态：CLI（pandoc-cli）、HTTP 服务（pandoc-server）、WASM（wasm/ 目录，支持浏览器内运行）

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| src/ | 核心 Haskell 库（Text.Pandoc.*），含 Readers/、Writers/、Class/、Parsing/ 等 |
| pandoc-cli/ | 命令行入口包（src/pandoc.hs，单文件薄壳） |
| pandoc-lua-engine/ | Lua 脚本引擎：过滤器、自定义 reader/writer、pandoc API 模块 |
| pandoc-server/ | HTTP API 服务包（单文件 src/Text/Pandoc/Server.hs） |
| data/ | 随库分发的数据文件：62 个输出模板（data/templates/）、106 个界面翻译（data/translations/）、docx/odt/pptx 参考文档 |
| test/ | 2137 个文件共 18MB：golden 测试 + 命令行行为测试 |
| citeproc/, xml-light/ | 捆绑的数据资源（biblatex 本地化等）与 vendored 代码 |
| wasm/ | WASM 构建（Makefile、examples/、针对 lua/conduit 等包的补丁见 cabal.project wasm32 分支） |
| linux/, macos/, windows/ | 各平台打包脚本 |
| MANUAL.txt | 8265 行用户手册（README.md 由它经 tools/update-readme.lua 自动生成） |

组织方式评述：典型的"核心库 + 多发行形态"cabal 多包工程。核心库把所有逻辑收拢在 `Text.Pandoc` 命名空间下按能力分层（App 编排、Class 效果抽象、Readers/Writers 双向体系、Filter/Scripting 扩展点、Templates/Citeproc/PDF 支撑模块），CLI/Lua/Server 三个包只是不同形态的外壳，边界非常干净。

## 5. 核心架构与关键实现

### 5.1 Reader/Writer 双向体系与格式注册

每种格式一个模块，统一注册在两份查找表里（src/Text/Pandoc/Readers.hs、Writers.hs）。Reader 分文本类和字节流类（二进制容器格式）：

```haskell
readers :: PandocMonad m => [(Text, Reader m)]
readers = [("native"       , TextReader readNative)
          ,("json"         , TextReader readJSON)
          ,("markdown"     , TextReader readMarkdown)
          ...
          ,("docx"         , ByteStringReader readDocx)
          ,("pptx"         , ByteStringReader readPptx)
          ,("xlsx"         , ByteStringReader readXlsx)
          ...
          ]
```
（出处：src/Text/Pandoc/Readers.hs，53 个注册项；Writers.hs 的 `writers` 表 76 项——一个 reader 可对应多个 writer 变体，如 epub/epub2/epub3。）

格式名只是起点，真正的方言差异由"格式 + 扩展名差分"表达：`getReader` 调用 `Format.applyExtensionsDiff` 按 `markdown-gfm+emoji` 这类 spec 计算 Extension 集合（Readers.hs；Extension 枚举含 89 个构造子，src/Text/Pandoc/Extensions.hs 第 45 行起）。同一套 `readMarkdown` 由此支撑 pandoc/PHP/GitHub/MMD 等所有 Markdown 方言。

### 5.2 PandocMonad：副作用抽象层

所有 Reader/Writer 不直接碰 IO，而是运行在 `PandocMonad` 类型类之上（src/Text/Pandoc/Class/PandocMonad.hs）：取文件/网络资源 `fetchItem`、媒体包 `getMediaBag`、日志 `report`、时间戳、资源搜索路径等约 30 个效果方法全部抽象化。两个标准实例：`PandocIO`（真实 IO）与 `PandocPure`（纯态、无网络，src/Text/Pandoc/Class/PandocPure.hs）。这一设计让整个库可在纯态下嵌入测试与 WASM，也让 docx/epub 内嵌图片统一收集进 MediaBag 再统一打包。

### 5.3 过滤器与脚本引擎

过滤器三分（src/Text/Pandoc/Filter.hs）：

```haskell
data Filter = LuaFilter FilePath
            | JSONFilter FilePath
            | CiteprocFilter -- built-in citeproc
```

- LuaFilter：进程内执行。pandoc-lua-engine 把 AST 双向 marshal 成 Lua 表（Marshal/ 下 20 余个模块），过滤器按类型签名 `Para → Para` 匹配 AST 节点并逐个应用（pandoc-lua-engine/src/Text/Pandoc/Lua/Filter.hs 的 `runFilterFile`/`runAll`）；同时暴露 15 个 `pandoc.*` API 模块（Module/Pandoc.hs、MediaBag.hs、System.hs 等）。
- JSONFilter：把 AST 以 JSON 序列化交给外部进程（任意语言写过滤器），处理完再读回。
- CiteprocFilter：内置引注处理，走 src/Text/Pandoc/Citeproc.hs。

关键解耦点是 `ScriptingEngine` 抽象（src/Text/Pandoc/Scripting.hs）：主库只依赖这个接口，Lua 引擎由 pandoc-cli 在链接期注入。因此不带 Lua 的纯库构建、以及 WASM 构建都可自由裁剪（cabal.project 中 pandoc-cli 有 `+lua -repl -server` 等 flag 组合）。自定义 Lua reader/writer 也走同一引擎（App.hs 第 172-179 行：reader 名以 `.lua` 结尾时从脚本取 `Reader`）。

### 5.4 CLI 编排管线

命令行主流程集中在 src/Text/Pandoc/App.hs 的 `convertWithOpts`（432 行）：解析选项 → 确定 reader/writer（含 defaults 文件、自定义脚本）→ 解析输入为 AST → 应用过滤器 → citeproc 引注 → 模板填充/独立文档处理 → 输出（含 PDF：按扩展引擎 latexmk/pdflatex/weasyprint 等调用，src/Text/Pandoc/PDF.hs）。 OOXML 系格式共享解析基础设施：docx/pptx/xlsx 三个 reader 复用 Readers/OOXML/（Charts.hs、Shared.hs）。

## 6. 构建与运行

```bash
# 从源码构建安装 CLI（INSTALL.md）
cabal update && cabal install pandoc-cli        # 或 stack install pandoc-cli
cabal build all                                  # 构建全部 4 个包
cabal test test-pandoc                           # 跑核心测试套件（pandoc.cabal 第 860 行）

# 日常使用
pandoc input.md -o output.docx
pandoc input.md --lua-filter=figs.lua -s -o out.html
pandoc --list-input-formats / --list-output-formats
```

不运行项目构建/测试（静态分析约束），以上命令来自 INSTALL.md、pandoc.cabal 与 MANUAL.txt。

## 7. 测试与工程质量

- 规模：test/ 目录 2137 个文件共 18MB，远超核心库本体（4.4MB）——测试是这个项目最重的资产。
- 三层结构（pandoc.cabal 单一 `test-suite test-pandoc`，入口 test/test-pandoc.hs）：
  - 单元/golden 测试：test/Tests/ 下 65 个 .hs 共约 1.55 万行，按 Reader/Writer 分模块，与 test/ 下成对的 `.txt → .native` golden 文件比对；
  - 命令行行为测试：test/command/ 1236 个文件，每个 `.md/.args` 输入配 `.native` 期望输出，覆盖选项、错误信息、默认文件等端到端行为（test/Tests/Command.hs）；
  - 格式回归样例：docx/、epub/、fb2/、odt/ 等二进制样例目录。
- CI：.github/workflows/ 下 9 个工作流——ci.yml（Linux/macOS/Windows 三平台 stack 构建+测试，按路径忽略文档变更）、nightly.yml（夜间构建发 nightly 版）、benchmark.yml（性能基准，对应 benchmark/ 目录）、release-candidate.yml、format-validation.yml、lint 等。
- 工程质量观察：changelog.md 持续 20 年维护、条目精确到 issue 编号与贡献者署名；weeder.toml（死代码检测）与格式校验工作流并存；3.13 changelog 显示对 YAML 解析这类底层依赖的替换有完整行为变更说明。

## 8. 社区与维护状态

- 活跃度：极高。默认分支 main 最近 push 2026-11-04（gh api）；最新 release 3.13 于 2026-10-26 发布，从 changelog 看基本保持每月一个 minor 版本的节奏。
- 规模：forks 5136、open issues 1058、贡献者约 415 人（gh api contributors 分页实测）。
- 结构：单一核心维护者（John MacFarlane）主导二十年 + 少数长期协作者（如 Lua 引擎维护者 Albert Krewinkel，见 pandoc-lua-engine 模块版权头），周边大量一次性贡献者——典型的"仁慈独裁者"模式，决策链短、标准收紧快。
- 生态位：被 Quarto、Zettlr、Obsidian 导出插件、无数静态站点与学术写作工具依赖；README 由 MANUAL.txt 自动生成的自我托管做法本身就是其用途的示范。

## 9. 亮点与风险

亮点：
1. 中间表示设计教科书级：N×M → N+M，AST 单独成包（pandoc-types）且十年向后兼容，成为整个文档工具生态的通用语。
2. `PandocMonad` 效果抽象让同一套解析代码可在 IO/纯态/WASM 三种环境运行，测试与嵌入都受益（src/Text/Pandoc/Class/）。
3. 扩展机制优雅：89 个细粒度 Extension + 差分组合，一个 Markdown reader 吃下所有方言（src/Text/Pandoc/Extensions.hs）。
4. Lua 过滤器生态把"改 AST"的门槛降到脚本级，且 JSON 过滤器兜底支持任意语言（pandoc-lua-engine/、Filter.hs）。
5. 测试密度罕见：18MB 测试资产对 4.4MB 源码，golden 文件覆盖每种格式的往返行为。

风险/局限：
1. GPL-2.0+ 传染性许可限制其在闭源/宽松许可产品中的直接嵌入（COPYING.md）。
2. 核心路径（App.hs 编排、Markdown reader 2415 行）高度集中于个别维护者，长期总线因子偏低。
3. Haskell 依赖图庞大，从源码完整构建编译时间长、对低配机器不友好（INSTALL.md 明言"will take a while"）。
4. PDF 输出依赖外部程序（LaTeX 引擎、wkhtmltopdf 等），开箱体验受制于用户环境（pandoc.cabal description、PDF.hs）。

## 10. 总结评价

Pandoc 是那种二十年只做一件事并把一件事做成行业标准的项目：以一个稳定到可以当作公理使用的中间 AST 为核，双向挂载 51 种输入和 73 种输出格式，再用扩展、模板、Lua 过滤器三件套把"足够好"的默认转换升级为可编程的文档流水线。代码本身是学习 Haskell 大型工程的优秀范本——类型类隔离副作用、-monad 转换器组织解析、类型安全的 AST 遍历（`Walkable` 类）都在真实约束下使用而非炫技。适合所有需要文档格式互转的人直接使用；适合编译器/DSL 设计者、Haskell 工程师和文档基础设施开发者精读其 Reader/Writer 与 Filter 抽象。
