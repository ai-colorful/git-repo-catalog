# Crosstalk-Solutions/project-nomad 项目分析

> 星数 43351（快照 2026-10-24，2026-11-14 gh api 复核 43352）｜主语言 TypeScript｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：Project NOMAD 是一个离线优先的自托管知识与教育服务器——用一个 Web 控制台（"Command Center"）编排一组容器化工具，把维基百科镜像、医疗参考、电子书、Khan Academy 课程、离线地图和本地 AI 问答装进用户自己的硬件，全程无需互联网。

| 项目 | 内容 |
|------|------|
| 名称 | Crosstalk-Solutions/project-nomad |
| 星数 | 43,351（2026-11-14 实测 43,352，forks 4,463） |
| 主语言 | TypeScript（后端 AdonisJS + 前端 React） |
| 许可证 | Apache-2.0（`LICENSE`；Creator Pack 内容另立许可，见第 9 节） |
| 创建时间 | 2025-06-24（gh api） |
| 最近推送 | 2026-11-12（gh api），最新 release v1.35.1（2026-10-05） |
| 代码规模 | admin/app + admin/inertia 约 25,000 行 TS/TSX，664 个文件（tarball 统计） |
| 官网 | https://www.projectnomad.us |

## 2. 项目是什么，解决什么问题

NOMAD 解决的核心问题是"断网环境下的完整知识基础设施"。灾备准备、偏远地区教育、出海航行、审查规避等场景需要的不只是离线文件，而是一套可管理、可持续更新的服务栈：内容从哪下载、放在哪、如何检索、如何在低算力机器上用 AI 问答。NOMAD 的答案是一个 Docker 编排控制台：用户在浏览器里点选要装的"资源"（Wikipedia 快照、医学库、课程包、地图区域），控制台负责拉镜像、建容器、挂载数据目录、广播安装进度（README "How It Works"，`install/management_compose.yaml`）。

它本身不带内容。所有离线内容都来自明确的第三方源：Wikipedia/医学参考通过 Kiwix 的 ZIM 格式（`collections/wikipedia.json` 直接指向 `download.kiwix.org` 的多个体积档位），课程通过 Kolibri，地图是 ProtoMaps 的 pmtiles 切片（托管在姊妹仓库 project-nomad-maps）。控制台只做目录、下载队列和容器管理（`collections/kiwix-categories.json`、`collections/maps.json`）。

AI 是差异化卖点：内置 Ollama（NVIDIA CUDA / AMD ROCm 双镜像分支）+ Qdrant 向量库，文档上传、OCR、向量化、检索全部本地完成；更有特色的是可以把已装的 ZIM 文件（如整本 Wikipedia）直接抽取进知识库做 RAG（`admin/app/services/zim_extraction_service.ts`）。RAG 全链路有配套的离线评测框架，用 golden set 回归验证提示词改动（`admin/tests/eval/`）。

工程风格明显区别于同类"求生 NAS"项目：README 明确反对裸机最低配路线，推荐 GPU 机型（最优 32GB RAM + RTX 3060）；更新走 opt-in 的自动更新 sidecar；有社区硬件跑分排行榜（`admin/app/services/benchmark_service.ts` + benchmark.projectnomad.us）。作者是 Crosstalk Solutions（UniFi/家庭实验室领域的 YouTube 频道），2025-06 开源，约 17 个月涨到 43k 星，社区以 Discord 为主阵地（README 徽章、FAQ）。

## 3. 技术栈与依赖

后端（`admin/package.json`）：
- **AdonisJS 6** 全栈框架（`@adonisjs/core` 6.19、Lucid ORM、Edge 模板、`@vinejs/vine` 校验、`@adonisjs/transmit` SSE 广播）
- **MySQL 8**（mysql2）持久化服务清单/聊天记录/跑分结果；**Redis 7 + BullMQ + ioredis** 做下载/AI 模型下载/跑分三类后台队列（`admin/package.json` scripts 中的 `work:downloads` 等）
- **dockerode 5** 驱动整套服务编排；**@openzim/libzim** 原生绑定读取 ZIM 归档；**@qdrant/js-client-rest** + **ollama** + **openai** 客户端做本地/远程推理
- 内容解析：**cheerio**（ZIM HTML 清洗）、**pdf-parse/pdf2pic/mammoth/tesseract.js/sharp**（PDF→图片→OCR 的知识库摄取链）

前端（`admin/inertia/`）：
- **Inertia.js + React 19 + Tailwind CSS 4 + Vite 6**，TanStack Query/Virtual，Uppy 文件上传，CodeMirror 6，**maplibre-gl + react-map-gl + pmtiles** 离线地图渲染（`admin/package.json`）

部署与外围：
- Docker 多阶段构建（`Dockerfile`，node:22-slim），安装入口 `install/install_nomad.sh`（686 行 bash），自更新/磁盘采集两个 sidecar 镜像（`install/sidecar-updater/`、`install/sidecar-disk-collector/`），离线翻译代理 nomad-translate（Python，`install/nomad-translate/`）

## 4. 目录结构与代码组织

| 目录 | 职责 |
|------|------|
| `admin/` | 主应用 "Command Center"：AdonisJS 后端 + Inertia 前端 + 测试 + 文档 |
| `admin/app/services/` | 37 个服务模块，编排核心（docker、rag、ollama、zim、benchmark、各类 auto_update） |
| `admin/app/controllers/` | 19 个控制器，约 178 条路由（`admin/start/routes.ts`，910 行） |
| `admin/app/models/` | 14 个 Lucid 模型（Service、ChatSession、BenchmarkResult、WikipediaSelection 等） |
| `admin/inertia/pages/` | 前端页面：chat、maps、supply-depot、easy-setup、drug-reference、settings 等（130 个 tsx） |
| `admin/database/seeders/service_seeder.ts` | 全部子应用的镜像清单（Kiwix、Qdrant、Ollama、Kolibri、CyberChef、Vaultwarden、Jellyfin 等 17 项） |
| `admin/tests/` | Japa + node:test：unit / functional / eval 三层，67 个测试文件 |
| `collections/` | 内容目录清单（JSON）：wikipedia、kiwix-categories、maps、conditions、remedies、creator-packs |
| `install/` | 安装脚本、compose 模板、sidecar 源码、两个引导用 mini ZIM（各约 4.4MB） |
| `.github/workflows/` | 7 条 CI：主镜像/三个 sidecar/翻译代理的构建、PR 构建、release（semantic-release）、内容 URL 有效性校验 |

组织方式：单仓多镜像的 monorepo（admin 主镜像 + 3 个 sidecar/代理镜像 + 嵌入式资产），无独立 npm workspace，admin 是唯一 Node 包。服务清单写死在 seeder 而非配置文件，新增"Supply Depot"应用 = 改 `service_seeder.ts` + `docker_service.ts` 加 install 分支。

## 5. 核心架构与关键实现

### 5.1 Docker-outside-of-Docker 编排（`admin/app/services/docker_service.ts`，2443 行）

NOMAD 容器本身不跑 Docker daemon，而是挂载宿主机 `/var/run/docker.sock`（DooD 模式），通过 dockerode 创建/升级/监视兄弟容器（`install/management_compose.yaml` 注释、`docker_service.ts` 构造函数按平台选 Unix socket 或 Windows named pipe）。关键设计：

- **存储路径单源**：admin 启动时解析自身 `/app/storage` 挂载对应的宿主机真实路径，所有子容器 bind mount 都改写到该根下，用户挪存储只改 compose 一处（`docker_service.ts` 中 `_hostStorageRoot`，对应 issue #938 注释）
- **安装进度实时广播**：每个 install 分支通过 `@adonisjs/transmit` 向 UI 广播 preinstall/installing 事件（`docker_service.ts` `_broadcast`）
- **GPU 感知的 Ollama 安装**：拉镜像前先探测 GPU，AMD 卡自动换 `ollama/ollama:rocm`，并按 DB 记录的版本 tag 升级（`docker_service.ts:775`、`:1765` 起）
- 安装子服务前有 `_runPreinstallActions__*` 钩子：Kiwix 先从磁盘重建 library XML、Calibre-Web 注入空书库（`admin/install/calibre-empty-library/`）

### 5.2 ZIM → 知识库的摄取链（`admin/app/services/zim_extraction_service.ts` + `rag_service.ts`）

用原生 `@openzim/libzim` 打开 ZIM 归档，读元数据（Title/Creator/Language 等），再以 cheerio 清洗 HTML（去导航/脚注等，`admin/constants/zim_extraction.ts`），按块切分为 chunk 写入 Qdrant——即"把整本离线 Wikipedia 变成本地 AI 的语料"。`rag_service.ts`（2776 行）封装 Qdrant 客户端惰性初始化、集合管理与嵌入调用（`OllamaService.embed`，`admin/app/services/ollama_service.ts:1042`）。

### 5.3 RAG 提示管线与评测驱动（`admin/app/services/rag_pipeline_service.ts`，363 行）

`buildPrompt` 是"用户发消息到 payload 进 Ollama"的唯一实现：注入默认系统提示与用户 NOMAD.md、按历史改写查询（query rewrite）、检索、按模型上下文窗口裁剪、决定 num_ctx。文件头注释明言设计动机——chat 端点和 eval harness 都调用同一个 `buildPrompt`，"测 harness 就是测生产，而非测一个会漂移的副本"。配套 `RelevanceJudgeService`、`TokenCalibrationService`、`context_budget.ts`，评测产物（baselines/goldens/reports）直接提交在 `admin/tests/eval/`。

### 5.4 内容目录与更新体系（`collections/` + `admin/app/services/collection_*_service.ts`）

内容清单是带 spec_version 的 JSON：`wikipedia.json` 提供 Quick Reference(331MB)/Popular Articles 等多档位直链 Kiwix 下载；`home_remedies.json`/`natural_remedies.json` 明确标注来源为 CDC/NIH/NCCIH/FDA 且为美国公有领域；`maps.json` 指向姊妹仓库的 pmtiles。CI 有专门的 `validate-collection-urls.yml` 持续校验这些 URL 不失效；Dockerfile 构建时执行 `gen:curated-data` 从 collections/*.json 重新生成 `admin/app/data/*.ts`，防止两处数据漂移（`Dockerfile` build stage 注释）。

## 6. 构建与运行

安装（README，Debian 系 x86-64）：

```bash
curl -fsSL https://raw.githubusercontent.com/Crosstalk-Solutions/project-nomad/refs/heads/main/install/install_nomad.sh -o install_nomad.sh
sudo bash install_nomad.sh   # 装 Docker → 下载 compose → 注入 APP_KEY/密码 → docker compose up
```

进阶：手动复制 `install/management_compose.yaml` 为 compose.yml 后 `docker compose up -d`；或从源码构建镜像 `docker build -t project-nomad .`（多阶段，含 go-pmtiles 固定版本与 SHA256 校验）。

admin 开发（`admin/package.json`）：

```bash
npm ci && npm run dev            # node ace serve --hmr
npm run build && npm start       # node ace build / node bin/server.js
npm run typecheck && npm run lint
npm test                         # node ace test（Japa）
npm run test:eval                # eval/rag 回归测试
npm run work:all                 # 启动三类 BullMQ worker
npm run gen:curated-data         # collections JSON → app/data TS
```

运行后访问 `http://localhost:8080`；API 健康检查 `GET /api/health`（compose healthcheck）。

## 7. 测试与工程质量

- 测试三层（`admin/tests/`）：unit 65 个文件（node test runner 风格 spec.ts）、functional（openapi 契约测试）、eval 目录存 RAG 评测语料/golden 集/基线/报告。覆盖面偏行为回归：聊天流式与中断（chat_stream_abort）、自动更新退避（content_auto_update_backoff）、KB 摄取决策、GPU 型号识别等，测试名与 issue 号对应（如 #881）
- CI（`.github/workflows/`）：每个镜像独立构建流水线（build-primary-image / sidecar-updater / disk-collector / translate-proxy / admin-on-pr），release.yml 用 semantic-release 自动发版；validate-collection-urls.yml 校验内容清单 URL 可用性
- 工程质量观察：代码注释密度高且记录决策动因（docker_service pullImage 修复 dockerode 假成功的注释、rag_pipeline 的"verbatim port"声明）；es lint + prettier + tsc --noEmit 齐备；安全敏感处有自查（compose 中显式关闭 Dozzle 的 shell/actions，Dockerfile 注释承认 entitlement key 会留在公开镜像层）

## 8. 社区与维护状态

- 活跃度：2026-11-12 仍有推送（gh api）；release 节奏密集，v1.34.1→v1.35.1 之间 5 个月内多版本 + rc 预发布（2026-09~10，gh api releases）
- 贡献者结构：前二为 jakeaturner（364 commits）与 chriscrosstalk（273，即 Crosstalk Solutions 创始人），另有 cosmistack-bot 自动化账号与 dependabot；合计高度集中于 2 人 + 机器人（gh api contributors）。规模看是"小核心团队 + 社区 PR"模式
- 社区渠道：Discord（README 徽章）、公开 roadmap（roadmap.projectnomad.us）、社区跑分排行榜、GitHub Discussions；WSL2/macOS 支持由社区指南承接（FAQ.md）
- 多语言：UI 仅英文，nomad-translate（0.1.1）提供 ZIM 内容的离线机翻代理（`install/nomad-translate/README.md`，对应 issue #1272）

## 9. 亮点与风险

亮点：
1. 完整闭环：内容目录 → 下载队列 → 容器编排 → 浏览器访问 → 自动更新，一条命令装机，普通用户零 Docker 知识也能用
2. ZIM→RAG 的取巧设计：不造离线百科阅读器，直接把 Kiwix 生态的成熟内容抽成向量知识库喂给本地 AI，生态位独特
3. 评测驱动 RAG：生产管线与评测共用同一实现，golden set 基线入库，开源项目里少见的 AI 质量工程实践
4. 内容合规意识强：每个清单标注来源与许可（US 公有领域医疗内容、Kiwix 官方 ZIM），CI 持续校验链接有效性
5. 差异化增长策略：创作者视频包（Creator Packs）+ 社区硬件排行榜，借助 Crosstalk Solutions 的 YouTube 受众冷启动（`collections/creator-packs.json`）

风险/局限：
1. 安全模型激进：默认无任何认证（README "About Security" 明示），同时 admin 容器持有 docker.sock（DooD），一旦接入不可信局域网等于交出宿主机控制权；依赖用户自行用防火墙限暴露面
2. 平台边界窄：官方仅支持 Debian 系 x86-64，无 ARM 官方镜像（FAQ 确认），树莓派等设备只能走社区 fork
3. 资源占用与单点：主栈强依赖 MySQL+Redis 双容器，裸"轻量"安装即 5GB 起步；compose 模板要求替换多处 replaceme 密钥，脚本 sed 注入方式较脆弱（`install/install_nomad.sh:449` 起）
4. 内容版权分层复杂：Creator Packs 采用自拟的 `nomad-creator-pack-1.0` 许可且 README/文件头自述"未过律师审查（DRAFT）"，仅授权官方渠道分发、终端用户个人非商业离线查看（`collections/creator-pack-license.md`）；Wikipedia ZIM 为 CC-BY-SA 由 Kiwix 官方分发，合规边界总体清晰但 Creator 部分仍在演变中

## 10. 总结评价

Project NOMAD 是 2025 年以来"自托管 + 离线准备"浪潮里完成度最高的整合者：它不自研内容也不自研推理引擎，而是把 Kiwix、Kolibri、ProtoMaps、Ollama、Qdrant、CyberChef 等成熟组件用一套 TypeScript 控制台黏合成语义连贯的产品，并以评测驱动的 RAG 管线和内容 URL 持续校验体现出超出同类项目的工程纪律。43k 星在 17 个月内达成，与作者频道的受众高度相关，但代码本身的注释质量、回归测试与发布自动化也配得上这个量级。适合灾备/教育/航海等离线场景的用户直接安装使用；对想研究"如何用 Docker 编排一个多服务离线产品""如何给本地 RAG 做评测回归"的开发者，`docker_service.ts`、`rag_pipeline_service.ts` 与 `admin/tests/eval/` 是很好的范本。读源码前需注意它是典型的 AdonisJS 6 单体，前后端通过 Inertia 深度耦合，与常见"REST API + 独立 SPA"结构差异较大。
