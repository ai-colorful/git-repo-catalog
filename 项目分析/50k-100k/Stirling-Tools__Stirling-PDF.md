# Stirling-Tools/Stirling-PDF 项目分析

> 星数 94801（快照 2026-10-24）｜主语言 TypeScript｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：开源自托管 PDF 工具平台——浏览器 UI、桌面客户端（Tauri）与私有 REST API 三种形态交付 50+ PDF 操作工具，数据不出本机。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | Stirling PDF（Stirling-Tools/Stirling-PDF） | README.md |
| 星数 | 94,801（快照）；gh api 实时 95,413 | gh api repos/Stirling-Tools/Stirling-PDF |
| 主语言 | GitHub 标 TypeScript；实际为 Java(后端) + TS(前端) 双栈 | 同上 + 文件统计 |
| 许可证 | 开核（open-core）：主体 MIT，proprietary/saas/engine/desktop 等目录各有专有许可 | LICENSE、app/proprietary/LICENSE |
| 版本 | 3.2.2 | build.gradle（`version = '3.2.2'`） |
| 最近 push | 2026-10-30（分析前一天） | gh api `pushed_at` |
| forks / open issues | 10,620 / 499 | gh api |

主语言核实（清单标 TypeScript）：GitHub 官方 language=TypeScript，按文件数前端 4,152 个 .ts/.tsx 对 Java 主代码 1,504 个，前端目录字节数（90MB）也超过后端（66MB）；但产品核心（PDF 处理管线、API、安全、pipeline）全在 Java 侧。准确表述是"双栈项目：TS 为 GitHub 统计主语言，Java 为功能主体"；历史上该项目主体确实是 Java（早期为单一 Spring Boot 应用，近年才重写前端为 React SPA，docs/developer/DeveloperGuide.md 有分层说明）。

## 2. 项目是什么，解决什么问题

Stirling PDF 的起点是"自己搭一个不外发文档的 PDF 工具站"：合并、拆分、旋转、压缩、OCR、签名、加水印、改权限、格式转换等日常 PDF 操作全部在本机或私有服务器完成（README.md「Key Capabilities」：50+ 工具、SSO/审计/私有 API、40+ 语言 UI）。安装入口是一行 `docker run -p 8080:8080 docker.stirlingpdf.com/stirlingtools/stirling-pdf`（README.md Quick Start）。

项目已从单一 Web 应用演化为"一份代码、三种交付形态"：浏览器 SPA（自托管服务器）、桌面客户端（Tauri 2，内嵌同一 Spring Boot 后端）、以及 SaaS 版（独立计费/账号层）。这由 Gradle 的 `STIRLING_FLAVOR`（core/proprietary/saas）与前端 `@editor/*` 分层别名机制支撑（settings.gradle、AGENTS.md「Frontend cloud/ Layer」）。

后端是无状态 Java 服务：文件只在内存/临时目录流转，处理完即清理（AGENTS.md「File Persistence」）。所有能力同时暴露为 REST API（`/api/v1/...`，Swagger 自动文档），使它可以直接当作"PDF 微服务"嵌入别人的系统，这也是企业采用的主要动机之一。

API 面的规模可从控制器划分看出：`app/core/.../controller/api/` 下按领域分 6 组（converters、filters、form、misc、pipeline、security），另有 merge/split/rotate/crop/split-by-*(按大小/章节/分节) 等 18 个页面操作控制器平铺在同级——仅 misc 一组就有 27 个控制器（压缩、OCR、拆分扫描件、提取图片、盖章、页码等）。

AI 是最新增长点：`engine/` 目录是独立的 Python FastAPI 推理服务（pydantic-ai，可接 Anthropic/OpenAI 或自托管模型），承担 PDF 问答、审阅、评论、文档分类、矛盾检测等推理型任务，由 Java 端代理转发（AGENTS.md「Python Development (AI Engine)」、engine/src/stirling/agents/）。

商业化路线清晰：MIT 部分覆盖个人/自托管全部日常工具，付费能力（SSO、审计、集群、计费、存储加密）隔离在 proprietary/saas 模块并按目录切分许可证（LICENSE 前 25 行明确列出各目录授权归属）。

## 3. 技术栈与依赖

后端（Java，1,504 个主代码 .java 文件）：
- Spring Boot 4.1.2 + Jackson 3（`tools.jackson` 新命名空间）+ JDK 25 工具链，Web 容器用 Jetty（显式排除 Tomcat）——build.gradle plugins 段、`allprojects` 配置、app/core/build.gradle
- Apache PDFBox 3.0.8（含 xmpbox/preflight/jbig2-imageio）、BouncyCastle 1.85（签名/加密）、OpenSAML 5.2.1 + Spring Security SAML（SSO）——build.gradle `ext` 块
- Twelvemonkeys imageio 全家桶（BMP/TIFF/WebP/PSD 等图像格式）、telegrambots 6.9.7（Telegram 机器人通知）——app/core/build.gradle

前端（TypeScript，4,152 个 .ts/.tsx 文件）：
- Vite + React 18 + Mantine 8 + TailwindCSS 4；状态管理 React Context（FileContext 为核心）+ TanStack Query/Table/Virtual——frontend/package.json、AGENTS.md
- 客户端 PDF 引擎用 @embedpdf/*（渲染、批注、涂黑、缩略图等 20+ 插件）与 @cantoo/pdf-lib，PDF.js 时代正在被替换——frontend/package.json dependencies
- 桌面壳：Tauri 2（Rust），frontend/src-tauri/

AI 引擎（Python，engine/）：
- FastAPI + pydantic 2 + pydantic-ai-slim[anthropic,openai]，uv 管理依赖——engine/pyproject.toml

外部进程工具链（由 Java `ProcessExecutor` 统一调度）：LibreOffice（nogui，格式转换）、unoserver、tesseract + OCRmyPDF（OCR）、qpdf（源码编译，版本门控）、WeasyPrint（HTML→PDF）、Calibre（电子书转换，带沙箱策略）、ImageMagick、Python3——docker/base/Dockerfile、app/common/src/main/java/stirling/software/common/util/ProcessExecutor.java、app/core/.../config/ExternalAppDepConfig.java

镜像变体（AGENTS.md「Configuration Modes」、docker/embedded/）：
- ultra-lite：只保留 PDFBox 纯 Java 操作，不带外部工具，体积最小
- standard（embedded）：完整功能，按需探测系统工具（ExternalAppDepConfig 对缺失工具按组禁用相关端点）
- fat：预下载全部外部依赖/模型/语言包，面向离线与内网（air-gapped）环境
- 另有 backend/frontend/engine 分离部署镜像与 unoserver 专用镜像（docker/ 目录）

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| app/common/ | 后端共享层（208 文件）：ProcessExecutor、TempFileManager、JobQueue、ToolMetadataService、PdfAnnotationService 等公共服务与通用模型 |
| app/core/ | 主应用（269 文件）：SPDFApplication 入口、全部 `/api/v1` 控制器、pipeline 引擎、配置初始化 |
| app/proprietary/ | 付费特性（816 文件）：SSO(SAML2/OAuth2)、审计、计费、集群(S3/Valkey)、存储加密、workflow 自动化 |
| app/saas/ | SaaS 专属后端（211 文件） |
| frontend/ | React SPA（90MB）：src/editor/{core,proprietary,cloud,saas,desktop} 五层 + src/processor + src-tauri |
| engine/ | Python AI 推理服务：agents（orchestrator、pdf_review、pdf_questions、contradiction 等）+ api + contracts |
| docker/ | 多变体镜像：base（工具链）、embedded（标准/fat/ultra-lite）、backend/frontend/engine 分离镜像、unoserver |
| testing/ | Python behave 集成测试（45 个 .feature）+ Docker 冒烟脚本 |
| docs/ | 开发者文档（ADDING_TOOLS.md、DeveloperGuide.md、DATABASE.md 等） |
| scripts/、devTools/、buildSrc/、gradle/ | 脚本与构建辅助（Taskfile 驱动） |

组织方式评述：这是"多模块单体"而非 monorepo 微服务——一个 Spring Boot 进程按 flavor 装配不同模块（settings.gradle 用 `STIRLING_FLAVOR` 决定 include 哪些子项目）；前端用同名路径 shadow 覆盖（core 定义 stub，desktop/proprietary 同路径真实现，`@editor/*` 按构建 flavor 级联解析）实现零运行时条件判断的多形态编译（AGENTS.md「Component Override Pattern (Stub/Shadow)」）。模型转 SaaS 后代码规模反而向 proprietary 倾斜（816 > core 的 269 文件）。

## 5. 核心架构与关键实现

### 5.1 外部工具调度：ProcessExecutor（app/common/src/main/java/stirling/software/common/util/ProcessExecutor.java）

所有 LibreOffice/qpdf/Python 等外部进程调用收口到一个按 `Processes` 枚举的单例池：每类进程用 `Semaphore` 限并发（如 LibreOffice 会话上限可配），带超时与 liveUpdates；LibreOffice 在 Docker 内以独立用户运行在自研 C 沙箱后（`/usr/local/lib/stirling/lo-sandbox`，源码在 docker/base/lo-sandbox.c），Calibre 有独立沙箱策略类（CalibreSandboxPolicy）。这是"Java 编排 + 原生工具干活"模式的成熟实现，也是该项目能做格式转换的根基。

### 5.2 Pipeline 自动化（app/core/src/main/java/stirling/software/SPDF/controller/api/pipeline/）

PipelineController/PipelineProcessor/PipelineDirectoryProcessor 实现无代码多步工作流：JSON 配置声明操作链，每步通过 `InternalApiClient` 回环调用自身 REST API 复用全部工具端点；`PipelineLimits.requireWithinStepLimit` 强制 maxSteps/maxFilesPerStep 上限，`AutomationRunContext` 把一次 pipeline 的所有子步骤归并到同一 run id 供 SaaS 计费聚合（PipelineProcessor.java runPipelineAgainstFiles 头部注释）。设计上"工具即 API、pipeline 即编排"，避免维护第二套执行路径。

### 5.3 工具端点元数据（app/common/src/main/java/stirling/software/common/service/ToolMetadataService.java）

接口从 OpenAPI 声明推导每个端点的输入输出类型、是否多文件、响应 ZIP 是否需解包（SIMO/MIMO arity 区分"ZIP 是容器"还是"ZIP 就是交付物"）。pipeline 与前端 hook 都依赖它做路由决策，是前后端共享工具语义的枢纽；engine 端的 `tool_models.py` 也由 Java OpenAPI spec 生成（AGENTS.md「Python Commands: engine:tool-models」），保证三端操作清单单一来源。

### 5.4 前端分层覆盖（frontend/src/editor/）

`@editor/*` 别名按构建 flavor 级联：core → proprietary → cloud → saas/desktop。所有平台差异用"同路径 shadow 组件"表达：web 构建拿到返回 null 的 stub，desktop 构建拿到真实现（AGENTS.md 给出 WorkbenchBarFooterExtensions 完整示例）。配合 FileContext 集中管理文件状态（IndexedDB 持久化、PDF.js/Web Worker/blob URL 手动生命周期、宣称支持 100GB+ 大文件不崩溃），以及 52 个工具 hook（frontend/src/editor/core/hooks/tools/，统一走 useToolOperation 编排器：单文件/多文件/自定义三模式）。

### 5.5 AI 引擎边界（engine/src/stirling/agents/）

Python 服务刻意收窄为"推理不做执行"：orchestrator.py 调度 pdf_questions、pdf_review、pdf_comment、document_classifier、contradiction 等 agent；契约（contracts/）与模型（models/tool_models.py）强类型化，AI 输出 schema 设计为"不可能实例化错误"；要求支持任意后端模型包括自托管（AGENTS.md「Python AI Usage」）。

## 6. 构建与运行

统一入口是 Task（Taskfile.yml + .taskfiles/*.yml，AGENTS.md 与 docs/developer/DeveloperGuide.md）：

```bash
docker run -p 8080:8080 docker.stirlingpdf.com/stirlingtools/stirling-pdf  # 快速体验
task install        # 装全部依赖
task dev            # 后端(8080)+前端(5173)；task dev:all 加 AI 引擎(5001)
task build          # 构建所有组件
task test           # backend:test + frontend:test + engine:test
task check          # lint + typecheck + test 全量质量门
task docker:build[:fat|:ultra-lite]   # 三种镜像变体
./test.sh           # Docker 全变体集成测试
```

裸机要求 JDK 25 + Node + Python(uv) + LibreOffice/tesseract/qpdf 等系统工具（AGENTS.md「Important Notes」「Java Version: Requires JDK 25」）。安全特性需 `DOCKER_ENABLE_SECURITY=true`。前端禁止手动跑 build 脚本（CI/CD 负责，frontend/package.json scripts 注释）。配置为 YAML + 环境变量覆盖；安全关闭时 security 相关类直接被排除出编译（条件编译，AGENTS.md）。

## 7. 测试与工程质量

测试金字塔完整（AGENTS.md 自称"No unit tests"已过时，实测远超）：
- 后端单元/集成测试 1,009 个 Java 测试文件（common 173 / core 312 / proprietary 393 / saas 131，`find app/*/src/test -name "*.java"`）
- 前端 vitest 单测 917 个 .test.ts；Playwright e2e 188 个 spec，分 stubbed（mock API，CI 无后端可跑）与 live（真后端）两个 project（frontend/playwright.config.ts）
- Python behave BDD 集成测试 45 个 .feature（testing/cucumber/features/，目录名沿用 cucumber 但框架已是 behave，支持 10 分片并行打真实服务器，testing/cucumber/run-parallel.sh）
- engine pytest 46 个测试文件

CI 是亮点：46 个 GitHub workflow（.github/workflows/），覆盖构建、三变体 Docker 测试、stubbed/live e2e、无障碍（frontend-a11y）、OpenAPI/许可证/生成模型一致性检查、scorecards、nightly、多平台桌面发布（multiOSReleases）、AUR 发布等。代码风格用 Spotless + google-java-format（build.gradle plugins）、oxlint/stylelint/ruff/pyright 各端齐配。AGENTS.md（31KB）本身是高质量工程规范文档：注释四用途论、测试禁固定等待、栈版本"现实核查"（警告 LLM 训练数据停留在 Spring Boot 2/3）。

本地化体量同样可观：frontend/public/locales/ 下 43 种语言的 JSON 翻译包（翻译只维护 en-US，其余由脚本与社区流程同步，AGENTS.md「Translation Rules」）；fat 镜像还预装 OCR 模型与 Tesseract 多语言包（docker/base/Dockerfile、docker/embedded/Dockerfile 预置表单检测 ONNX 模型并做 SHA256 校验）。

## 8. 社区与维护状态

- 极活跃：gh api 显示 pushed_at 2026-10-30（分析前一天），open issues 499，forks 10,620。
- 版本 3.2.2；release 节奏 tarball 无法直接确认（无 git 历史），但 CI 有 nightly 与自动发布链路，README 挂 Discord/OpenSSF Scorecard 徽章（README.md）——活跃度为高置信推断，精确节奏未确认。
- 贡献者结构未确认（需 git 历史）；仓库有 Hacktoberfest topic、auto-labeler、stale-prs 等 12+ 社区自动化 workflow（.github/workflows/），显示成熟的 PR 治理。
- 公司化运作明显：Stirling PDF Inc 版权（LICENSE）、SaaS/企业版/文档站/计费体系齐备，属于"开源核心 + 商业公司"双引擎项目。

## 9. 亮点与风险

亮点：
1. 三形态一码库的架构方案干净：Gradle flavor 装配 + 前端 shadow 覆盖，零运行时条件判断实现 Web/Desktop/SaaS 差异（settings.gradle、AGENTS.md）。
2. 外部工具集成深度罕见：LibreOffice C 级沙箱、Calibre 沙箱策略、qpdf 版本门控降级、会话信号量——自托管工具链的工业级封装（ProcessExecutor.java、docker/base/）。
3. 测试与 CI 成熟度对齐顶级开源项目：1009 后端测试 + 917 前端单测 + 188 e2e + 45 BDD feature + 46 workflow。
4. "工具即 API"哲学贯彻：pipeline、前端 hook、AI engine 三端共享同一 OpenAPI 语义，无平行实现。
5. 工程文档（AGENTS.md、docs/developer/）质量极高，直接可当架构教材。

风险/局限：
1. 开核分层显著增加理解成本：Java 4 模块 + 前端 5 层 flavor，核心功能分散在 core/proprietary，逐文件追溯需先建立 flavor 心智模型。
2. 栈极其激进（Spring Boot 4.1.2、Jackson 3、JDK 25）：项目自己在 AGENTS.md 承认生态与 LLM 训练数据都未跟上，第三方库与贡献者门槛偏高。
3. 重依赖外部进程与系统工具版本，非 Docker 裸机部署体验取决于系统（ExternalAppDepConfig 只能降级禁用部分组）。
4. AI 能力需外部模型 API 或自托管支持结构化输出的模型，纯离线环境（fat 镜像）AI 功能受限（engine/pyproject.toml、AGENTS.md）。

## 10. 总结评价

Stirling-PDF 是自托管 PDF 领域事实上的标杆项目：以 PDFBox + 外部工具链的 Java 后端为底座，用一套精致的 flavor 分层同时支撑 Web、桌面、SaaS 三种交付形态，再叠加 REST API 全覆盖与 pipeline 编排，把"个人工具站"做成了"PDF 平台"。9.5 万星背后是罕见的工程完成度——测试金字塔、46 条 CI 流水线、31KB 的 AI 协作规范文档都达到甚至超过商业团队水准；代价是代码库复杂度陡增、技术栈激进、MIT 核心与商业模块边界需要花时间辨认。适合：想自建 PDF 服务而不愿外发文档的运维/个人（直接 Docker 用）；需要批量处理 API 的后端开发者（读 controller/api/ 即可上手）；研究"开源核心商业化"与"多形态单码库"架构的工程师（重点读 settings.gradle、AGENTS.md、ProcessExecutor、PipelineProcessor 与前端分层设计）。
