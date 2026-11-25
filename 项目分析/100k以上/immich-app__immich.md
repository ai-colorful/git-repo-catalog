# immich-app/immich 项目分析

> 星数 117002（快照 2026-10-24）｜主语言 TypeScript｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：高性能自托管照片/视频管理方案（Google Photos 的开源自托管替代品），覆盖移动端自动备份、Web 相册、人脸/CLIP 智能搜索的完整单体仓库。

| 事实 | 值 | 出处 |
| --- | --- | --- |
| 名称 | immich-app/immich | GitHub |
| 星数 | 117002（快照）；`gh api` 实测 117620（2026-10-30） | GitHub API |
| 主语言 | TypeScript（后端+Web）+ Dart（移动端）+ Python（ML）+ Rust（移动原生） | 仓库结构 |
| 许可证 | AGPL-3.0（`LICENSE`） | LICENSE |
| 版本 | v3.4.0-rc.0（`package.json`） | package.json |
| 最近推送 | 2026-10-29（分析前一天） | GitHub API |
| 创建时间 | 2022-02-03；forks 7405；贡献者约 463 人 | GitHub API |

## 2. 项目是什么，解决什么问题

Immich 解决的核心问题是"个人照片/视频数据的自主可控"：把散落在手机、云服务商那里的媒体资产集中备份到自己控制的服务器上，同时提供不逊于商业云相册的浏览与检索体验。README 自述为 "High performance self-hosted photo and video management solution"（`README.md`），并在开篇显著位置提醒用户对珍贵数据遵循 3-2-1 备份策略，态度务实。

功能上（`README.md` Features 表 + 源码验证）：手机端打开即自动备份、选择性相册备份、上传去重（checksum 拦截，`server/src/services/asset-media.service.ts`）、LivePhoto/MotionPhoto 完整保留（动态照片视频提取见 `server/src/services/metadata.service.ts`）、多用户与伙伴共享、相册/共享链接、全局地图、回忆（x 年前的今天）、回收站与归档收藏。检索能力是亮点：按元数据（EXIF）、按人脸、按 CLIP 语义、按图片内文字（OCR）搜索，分别对应 `server/src/services/search.service.ts`、`person.service.ts`、`smart-info.service.ts`、`ocr.service.ts`。

从部署形态看，官方 `docker/docker-compose.yml` 只需四个容器即可跑通：`immich_server`（API+任务处理）、`immich_machine_learning`（推理）、Valkey（队列/缓存）、内嵌 VectorChord 的 PostgreSQL（向量检索）。Web 前端由 server 直接托管，无需独立容器，这是 v3 相对早期架构的明显简化。

工程定位上是"真正的全平台产品"：NestJS 后端 + SvelteKit Web + Flutter 移动端 + Rust 移动原生加速 + Python ML 服务，全部在一个 pnpm monorepo 内迭代，由 `mise.toml` 统一任务入口（见 `AGENTS.md`）。

## 3. 技术栈与依赖

| 层 | 技术 | 证据 |
| --- | --- | --- |
| 后端 | NestJS 12 + Express 5 + Socket.IO，BullMQ 任务队列，Kysely + PostgreSQL | `server/package.json`、`server/src/repositories/job.repository.ts` |
| 媒体处理 | exiftool-vendored（元数据）、sharp（图像解码/缩放）、ffmpeg/ffprobe（转码、HLS） | `server/package.json`、`server/src/services/transcoding.service.ts` |
| 数据库扩展 | VectorChord（向量 HNSW）+ pgvectors，自建 postgres 镜像 | `docker/docker-compose.yml`（`ghcr.io/immich-app/postgres:14-vectorchord0.4.3-...`） |
| 队列/缓存 | Valkey 9（Redis 协议兼容） | `docker/docker-compose.yml` |
| Web | SvelteKit + Vite + Tailwind CSS 4，vitest | `web/package.json`、`pnpm-workspace.yaml` |
| 移动端 | Flutter/Dart（drift 本地库、pigeon 平台通道） | `mobile/pubspec.yaml`、`mobile/drift_schemas/`、`mobile/pigeon/` |
| 移动原生 | Rust workspace：codecs（JPEG XL 解码）、moxcms（色彩管理）、pic-scale（缩放），带 ARM NEON 特性，经 FFI/JNI 桥接 | `native/Cargo.toml`、`native/crates/` |
| 机器学习 | FastAPI + Gunicorn + ONNX Runtime，OpenCLIP/M-CLIP、人脸检测/识别、RapidOCR | `machine-learning/pyproject.toml`、`machine-learning/immich_ml/` |
| 可观测 | OpenTelemetry SDK + Prometheus exporter | `server/package.json` |
| 工程化 | pnpm workspace（12.15.0）、mise 任务运行器、renovate 自动升级 | `pnpm-workspace.yaml`、`mise.toml`、`renovate.json` |

## 4. 目录结构与代码组织

顶层目录（tarball HEAD 实测）：

| 目录 | 职责 |
| --- | --- |
| `server/` | NestJS 后端：REST API + 后台任务 worker，572 个 TS 文件约 9.7 万行 |
| `web/` | SvelteKit 前端（用户端 + 管理端），约 738 个 ts/svelte 文件 |
| `mobile/` | Flutter 移动端（Android/iOS），`lib/` 下 616 个 Dart 文件 |
| `native/` | Rust 移动原生核心（图像编解码/色彩/缩放） |
| `machine-learning/` | Python ML 推理服务（FastAPI + ONNX） |
| `e2e/` | API e2e + Playwright UI 测试（53 个 spec） |
| `packages/` | `@immich/sdk`（TS SDK）、`@immich/cli`、plugin-sdk/plugin-core、发布脚本 |
| `open-api/` | 生成的 OpenAPI spec 与客户端补丁（禁止手改，见 `AGENTS.md`） |
| `docs/` | Docusaurus 官方文档站 |
| `i18n/` | Weblate 驱动的翻译，仅 `en.json` 手工维护 |
| `docker/`、`deployment/` | docker-compose 与 Helm 等部署物 |
| `readme_i18n/`、`fastlane/`、`design/`、`misc/` | 多语言 README、应用商店元数据、设计资产、杂项 |

组织方式评述：典型多语言 monorepo，但边界切得干净——server/web 由同一团队维护（CODEOWNERS 均为 @danieldietzler），ML 单独由 @mertalev 负责，移动端由三人小组负责（`CODEOWNERS`）。前后端契约通过 OpenAPI 单一事实源约束：改 controller/DTO 后必须跑 `mise //:open-api` 重新生成 TS SDK 与 Dart 客户端，CI 强制校验（`AGENTS.md`）。server 内部是 NestJS 经典三层：controllers（~30 个）→ services（105 个文件）→ repositories（~40 个），SQL 全部抽到 `server/src/queries/*.sql` 由 Kysely 加载，schema 定义在 `server/src/schema/tables/`（40+ 表）。

## 5. 核心架构与关键实现

### 5.1 双 worker 进程模型与 19 条任务队列

server 以两种 worker 角色运行：`api`（HTTP）与 `microservices`（纯任务处理），入口分别在 `server/src/workers/api.ts` 与 `server/src/workers/microservices.ts`，由 `server/src/main.ts` 的 `Workers` 类统一 fork/守护，并支持维护模式 worker（`workers/maintenance.ts`）。任务基于 BullMQ，队列枚举共 19 条（`server/src/enum.ts` 的 `QueueName`）：缩略图、元数据提取、视频转码、人脸检测/识别、智能搜索、去重检测、存储模板迁移、侧车文件、备份、OCR、工作流等。业务代码用装饰器声明消费：

```ts
@OnJob({ name: JobName.AssetGenerateThumbnailsQueueAll, queue: QueueName.ThumbnailGeneration })
async handleQueueGenerateThumbnails({ force }: JobOf<JobName.AssetGenerateThumbnailsQueueAll>) {
  for await (const assets of batched(this.assetJobRepository.streamForThumbnailJob(...))) {
    // ...逐资产入队 AssetGenerateThumbnails
  }
}
```

（`server/src/services/media.service.ts`；`job.repository.ts` 为每条队列建 Worker，默认 concurrency 1，防止大文件任务打满 IO。）

### 5.2 媒体处理管线

上传链路：`asset-media.service.ts` 的 `uploadAsset` 落盘到上传目录 → checksum 去重 → 入队 `AssetExtractMetadata`。元提取（`metadata.service.ts`）用 exiftool-vendored 读 EXIF/GPS/时区，并特别处理三星/苹果动态照片：从原图二进制中抠出 `MotionPhotoVideo`/`EmbeddedVideoFile` 段生成隐藏视频资产（该文件约 700-815 行），时区处理还注释了 exiftool 对 UTC+0 的已知缺陷（约 1023 行）。

缩略图链路（`media.service.ts` `handleGenerateThumbnails`）：按资产类型分流——视频走 ffmpeg 抽帧，图片走 sharp 解码（`decodeImage` 带 `THUMBNAIL_DECODING_TIER` 分级解码策略，`server/src/constants.ts`），一次生成 thumbnail/preview/thumbhash 三类产物，启用后还有 fullsize，编辑过的资产叠加编辑图层再生成（`generateEditedThumbnails`），最后与库内文件状态对账（`syncFiles`）。

视频转码（`transcoding.service.ts`）：按系统配置策略（`TranscodePolicy`）决定是否转码，支持硬件加速探测（`videoInterfaces` 含 dri/Mali，`media.service.ts` 顶部初始化），在线播放走 HLS 会话——spawn ffmpeg 输出分段并由 `hls.service.ts` 管理会话生命周期。

### 5.3 ML 服务边界

server 与 ML 是两个独立进程，只通过 HTTP + 严格类型化的请求/响应结构通信。server 侧封装在 `server/src/repositories/machine-learning.repository.ts`：任务类型仅三种（`ModelTask`：facial-recognition / clip / ocr），载荷是 `{imagePath}` 或 `{text}` 加模型选项，路由 `/predict`。ML 侧（`machine-learning/immich_ml/`）用 `pipeline.py` 把模型组装成管道（Clip 的 visual/textual、FacialRecognition 的 detection/recognition、Ocr 的 detection/recognition 各为可插拔 Slot），`models/cache.py` 管理模型驻留，`allocator.py` 管理并发内存配额。推理后端是 ONNX Runtime，可选 CPU/CUDA/OpenVINO/ARM NN/RKNN/ROCm 六种执行提供者（`machine-learning/pyproject.toml` optional-dependencies），这是它能在 NAS/低端盒子上跑起来的关键。

向量落库在 server 侧：人脸 512 维 embedding 存 `face_search` 表并建 HNSW 索引（`server/src/schema/tables/face-search.table.ts`）：

```ts
@Index({
  name: 'face_index',
  using: 'hnsw',
  expression: `embedding vector_cosine_ops`,
  with: 'ef_construction = 300, m = 16',
})
```

CLIP 图文向量与 OCR 向量同理（`smart-info.service.ts` 会校验模型维度与库维度一致，并在换模型时重建索引）。

### 5.4 原生加速（移动端）

`native/Cargo.toml` 定义 Rust workspace：`codecs`（jxl JPEG XL 解码）、`moxcms`（ICC 色彩管理）、`pic-scale`（高性能缩放），均启用 `neon` 特性针对手机 ARM 优化，通过 `native_core_ffi`（JNI）暴露给 Flutter。这解决了 Flutter/Dart 在低端机上解码 HEIC/JXL 类格式慢的问题，是同类自托管项目里少见的投入。

## 6. 构建与运行

来自 `server/package.json`、`web/package.json`、`AGENTS.md`、`e2e/` 的实际命令：

```bash
# 官方部署（生产）
docker compose up -d            # docker/docker-compose.yml（server + ml + valkey + postgres）

# 开发（pnpm monorepo + mise 统一入口）
pnpm install
mise //server:start:dev          # NestJS watch 模式（AGENTS.md 规定所有任务走 mise）
mise //web:dev                   # SvelteKit dev server :3000
mise //:open-api                 # 改 DTO/controller 后重生成 OpenAPI + TS/Dart 客户端

# server 内直接命令
pnpm --filter immich run test            # vitest
pnpm --filter immich run migrations:generate
pnpm --filter immich run check:all       # format+lint+tsc+coverage

# ML 服务
uv run uvicorn immich_ml.main:app       # machine-learning/（FastAPI）

# e2e（API + Playwright）
e2e/ 下 playwright.config.ts + docker-compose.dev.yml
```

移动端构建走 `mise //mobile:*`（`build-mobile.yml` CI 可佐证），Android 另有 F-Droid 发布流水线（`.github/workflows/fdroid.yml`）。

## 7. 测试与工程质量

- 单元测试：server 109 个 `*.spec.ts`（与 105 个 service 文件几乎一一对应，vitest，`server/test/vitest.config.mjs`，另有 medium 级配置）；web 62 个 spec（vitest）。
- e2e：`e2e/src/specs/` 下 53 个 spec，分 server API / web UI / maintenance 三组，API 用真实 docker-compose 环境跑，UI 用 Playwright（`e2e/playwright.config.ts`）。
- ML：`machine-learning/test_main.py`（pytest + httpx），另有 `locustfile.py` 做推理压测。
- CI：27 个 workflow（`.github/workflows/`），覆盖单测（test.yml）、静态分析 + CodeQL（static_analysis.yml、codeql-analysis.yml）、OpenAPI 一致性（check-openapi.yml）、迁移顺序校验（migration-order.yml）、移动构建、draft-release/prepare-release 等。
- 代码质量观察：monorepo 根有严格 lint/format 配置（`--max-warnings 0`）；`AGENTS.md` 明文规定"不加代码注释，靠命名自解释"、"禁止 LLM 生成的 PR"、"generated 文件禁手改"——治理风格鲜明。SQL 集中在 `server/src/queries/*.sql` 便于审查；schema 有 audit 表族（asset-audit 等）支撑回收站/同步。

## 8. 社区与维护状态

- 活跃度极高：`gh api` 显示最近 push 为 2026-10-29；release 节奏极快——2026-10 月内已发布 v3.3.0、v3.3.1、v3.3.3、v3.3.4、v3.3.5 五个版本（GitHub releases API），当前主干为 v3.4.0-rc.0（`package.json`）。
- 贡献者约 463 人（GitHub contributors 分页 last=463），是典型的"核心团队主导 + 大量社区 PR"结构：CODEOWNERS 显示 server/web/e2e 由 @danieldietzler 把关、ML 由 @mertalev、移动端由 @shenlong-tanwen/@santoshakil/@agg23（`CODEOWNERS`）。
- 社区配套完善：24 种语言的 README（`readme_i18n/`）、Weblate 翻译流水线（merge-translations.yml）、Discord 徽章、demo 站（demo.immich.app，`README.md`）、独立文档站与 roadmap。
- 商业支撑：项目由 FUTO 资助（server 依赖含 `@futo-org/backups-orchestrator-api`，`server/package.json`），全职团队模式解释了其罕见的迭代速度。

## 9. 亮点与风险

**亮点**

1. 全栈自研完成度极高：备份、浏览、人脸/CLIP/OCR 检索、共享、回忆、HLS 播放、Rust 原生加速一应俱全，且 web/mobile/server 三端同版本节奏发布。
2. 任务系统设计成熟：19 条队列 + 双 worker 分离 + 流式分批（`batched()`）+ concurrency 1 保守默认，媒体重活不会拖垮 API（`server/src/enum.ts`、`job.repository.ts`）。
3. ML 服务边界干净：HTTP + 强类型 schema、多执行提供者（CUDA/OpenVINO/ARM NN/RKNN/ROCm）、模型可插拔（`machine-learning/immich_ml/pipeline.py`），对自托管硬件谱系的兼容性是同类最佳。
4. 工程治理严格：OpenAPI 单一事实源 + 客户端代码生成 + CI 强制校验，SQL 外置可审查，e2e 三层覆盖，27 条 CI 流水线。
5. 数据安全意识：开篇 3-2-1 备份警告、数据库备份任务/完整性检查服务（`database-backup.service.ts`、`integrity.service.ts`）、回收站与审计表。

**风险/局限**

1. AGPL-3.0 对商业化集成有传染性约束，商业产品不可直接闭源套壳。
2. 系统组件多（Postgres+VectorChord、Valkey、ML、server），资源占用与升级复杂度高于轻量方案（如 PhotoPrism 单容器路线）；`docker/docker-compose.yml` 里向量库为 pinned 镜像。
3. release 节奏极快、主版本更迭（v1→v3）伴随大量 breaking/迁移，长期跟版对自托管用户有维护成本。
4. 仓库体量巨大（server 单包近 10 万行 TS，四语言栈），新贡献者上手门槛高；`AGENTS.md` 还明文冻结了共享/外部库等子域（只收 bugfix），外部贡献空间受限。

## 10. 总结评价

Immich 是"自托管 Google Photos"这个赛道目前的标杆：它不只复刻了备份与相册的基本盘，还在语义搜索、人脸聚类、OCR 检索、动态照片保真、移动端原生性能上做到商业产品水准，同时用异常严格的工程治理（OpenAPI 驱动、双层测试、27 条 CI、明文化协作规范）支撑起 463 位贡献者、月发 5 版的迭代机器。星数冲上 11 万+ 是产品完成度与 FUTO 全职投入共同作用的结果。适合想彻底掌控个人影像数据的自托管用户直接部署，也适合研究"多语言 monorepo 如何组织媒体处理管线、任务队列与 ML 推理边界"的工程师精读——`server/src/services/` + `machine-learning/immich_ml/` 这条链路是全仓库最有学习价值的部分。
