# docker/awesome-compose 项目分析

> 星数 40500（快照 2026-10-24）｜主语言 —｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：Docker 官方出品的 Docker Compose 示例集——39 个可直接 `docker compose up -d` 跑起来的小型应用工程，覆盖从前端框架到反向代理到数据库的常见组合。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | docker/awesome-compose | README.md |
| 星数 | 40500（快照） | 任务参数 |
| 语言 | 无单一主语言（JS/Python/Go/Java/Rust/C#/PHP 混合样例） | 文件统计 |
| 许可证 | CC0-1.0 | LICENSE |
| 创建时间 | 2020-02-13 | gh api repos/docker/awesome-compose |
| 最近推送 | 2026-10-26（快照后 2 天仍在更新） | gh api |
| Forks / 贡献者 | 8347 forks / 61 contributors | gh api |
| 维护者 | 2 位 Docker 员工（aiordache、glours） | MAINTAINERS |

## 2. 项目是什么，解决什么问题

Docker Compose 是定义和运行多容器应用的事实标准，但初学者常卡在"这些服务怎么连起来"：网络怎么划分、依赖顺序怎么声明、环境变量和密码怎么传、反向代理怎么指向后端。awesome-compose 用一组最小可运行的完整工程回答这些问题——每个示例目录就是一个独立项目，进入目录执行 `docker compose up -d` 即可复现（README.md "Running a sample" 一节）。

它与 awesome 清单类仓库不同：不是外部链接的列表，而是全部样例代码自包含在仓库内（README.md 开头 "A curated list of Docker Compose samples"，样例目录即仓库目录）。README 明确声明这些样例面向本地开发/体验环境，**不得用于生产**（README.md Note 段）。

仓库还承担 Docker 官方文档的配套角色：`official-documentation-samples/` 目录收录 Django、Rails、WordPress 三个 quickstart，与 docs.docker.com 的 Compose 快速上手教程一一对应（official-documentation-samples/README.md）。

## 3. 技术栈与依赖

样例矩阵（基于各目录源码统计，代码行合计约 1.9 万行）：

- **前端**：React（react-express-mysql 等 6 个示例，`frontend/package.json` 中 react ^17）、Angular（angular/）、VueJS（vuejs/），另配 Nginx 做静态托管（react-nginx/）
- **后端**：Node.js/Express（express ^4.17）、Python/Flask 与 FastAPI 与 Django、Go（nginx-golang*/ 等 4 个）、Java Spring 与 Spark Java、Rust（react-rust-postgres/、含 WasmEdge 的 2 个）、C#/ASP.NET（aspnet-mssql/）、PHP（apache-php/）
- **数据库/中间件**：MySQL/MariaDB、PostgreSQL、MongoDB、Redis、MS SQL、Kafka(Redpanda)
- **基础设施/平台软件**：Nginx、Traefik 反向代理；ELK（elasticsearch-logstash-kibana/）、Prometheus/Grafana 监控栈；Nextcloud、Gitea、WordPress、Pi-hole、Plex、Portainer、Minecraft、Wireguard 等自托管应用
- **前沿方向**：2 个 WasmEdge 示例标记兼容 Docker+Wasm（README.md 中 icon_wasm.svg 图标说明）

依赖特点：各示例自带 package.json / go.mod / pom.xml / Cargo.toml，镜像层用官方基础镜像（golang:1.18-alpine、mariadb:10.6.4-focal 等，见各 Dockerfile 与 compose.yaml）；根目录 `.npmrc` 设 `ignore-scripts=true`、`.yarnrc.yml` 设 `enableScripts: false`，防止安装依赖时执行任意脚本。

## 4. 目录结构与代码组织

顶层共 40 个目录：39 个示例 + 1 个官方文档样例集（tarball 实测，共 691 个文件）。按 README 的官方分类：

| 分类 | 数量 | 代表目录 |
|---|---|---|
| 多服务集成应用 | 21 | react-express-mysql、nginx-golang-postgres、aspnet-mssql、elasticsearch-logstash-kibana |
| 单服务示例 | 12 | angular、vuejs、flask、fastapi、django、portainer |
| 平台基础配置（个人自用） | 6 | wordpress-mysql、nextcloud-redis-mariadb、gitea-postgres、prometheus-grafana |
| 官方文档 quickstart | 3（子目录） | official-documentation-samples/{django,rails,wordpress} |

每个示例遵循高度一致的标准结构（以 nginx-golang/ 为例，实测 7 个文件）：

```
nginx-golang/
├── compose.yaml          # 服务编排定义（核心）
├── README.md             # 结构图 + compose 内联 + 部署步骤 + 预期输出
├── backend/
│   ├── Dockerfile        # 多阶段构建
│   └── main.go           # 应用代码（刻意保持最小）
└── proxy/
    └── nginx.conf        # 反向代理配置
```

统计：39 个示例全部有顶层 compose 文件（37 个 `compose.yaml` + 2 个 wasmedge 用 `compose.yml`），35 个 Dockerfile，55 个 README.md（含子目录），23 张 png 截图（output.png/capture.png 展示预期结果）。组织方式是"扁平目录 + 约定优于配置"——无 monorepo 工具、无构建系统，目录名即示例名，命名用连字符表达服务组合（如 nginx-flask-mongo）。

## 5. 核心架构与关键实现

作为示例集，"核心实现"即 compose 编写模式的最佳实践，以 3 个代表性模式说明：

**（1）secrets 挂载替代明文环境变量**（react-express-mysql/compose.yaml）：数据库密码不写 env 明文，而是通过 compose secrets 从 `db/password.txt` 注入，MySQL 侧用 `MYSQL_ROOT_PASSWORD_FILE=/run/secrets/db-password` 读取，后端用 `DATABASE_PASSWORD=/run/secrets/db-password`；网络显式划分为 `public`/`private` 两层，数据库只在 private 网络，前端与后端在 public。这是全部带数据库示例的统一模式（各目录 db/password.txt）。

**（2）多阶段 + BuildKit 缓存挂载的 Dockerfile**（nginx-golang/backend/Dockerfile）：

```dockerfile
# syntax=docker/dockerfile:1.4
FROM --platform=$BUILDPLATFORM golang:1.18-alpine AS builder
RUN --mount=type=cache,target=/go/pkg/mod/cache \
    go mod download
RUN --mount=type=cache,target=/go-build \
    go build -o bin/backend main.go
```

使用 syntax 指令、跨平台构建（$BUILDPLATFORM）、Go 模块与构建缓存挂载、heredoc RUN，还定义了 `dev-envs` 目标（COPY --from=gloursdocker/docker 内置 Docker CLI 工具供 dev 用）——同一 Dockerfile 用 target 区分开发/生产镜像，react-express-mysql/compose.yaml 中 `target: development` 配合源码 bind mount 实现热重载。

**（3）健康检查脚本**（react-express-mysql/backend/healthcheck.js）：独立 10 行 Node 脚本探测 `/healthz` 返回 200 才退出 0，与 Dockerfile 的 HEALTHCHECK 指令对应；依赖服务的启动顺序用 `depends_on` + `restart: always` 兜底（db 服务）。

## 6. 构建与运行

每个示例统一入口（README.md "Getting started"，以 nginx-golang/ 为例）：

```console
docker compose up -d      # 构建并启动
docker compose ps         # 查看容器与端口映射
curl localhost:80         # 验证输出（预期返回 Hello from Docker!）
docker compose down       # 停止并清理
```

前置要求仅为安装 Docker Desktop（含 Compose v2，README.md Prerequisites）。本仓库自身无构建/打包流程——它就是源码本身；各示例内部的应用级命令（如 `npm run start-watch`）写在各 compose.yaml 的 command 字段里。

## 7. 收录范围与条目规模（分类体系）

收录范围克制而聚焦：只收"用 Compose 编排后能直接跑通"的完整工程，不收教程链接、不收 k8s/Swarm 内容。三条分类线（多服务集成 / 单服务 / 平台自用配置）在 README.md 中以锚点目录维护，命名约定 `服务A-服务B-服务C` 让用户扫目录名即知拓扑。条目规模：39 个示例、691 个文件、约 1.9 万行应用代码、35 个 Dockerfile（tarball 实测）。提交新示例需先开 issue 提案，维护者讨论"哪个组合最有价值"后再交 PR，且必须同时补示例 README 并登记进全局 README（CONTRIBUTING.md "Missing an example?"）。这是典型的"官方精选"路线：增速慢但每条都自包含、可验证，与社区 awesome 清单的"链接堆叠"形成鲜明差异。

## 8. 更新机制与 CI 验证

- **PR 流程**：issue 提案 → 维护者评审 → PR 合并；要求 commit 签名（DCO，CONTRIBUTING.md "Sign your work"）
- **CI**：`.github/` 仅有一个 workflow——CodeQL 静态安全分析（go/typescript/python 三语言矩阵，action 以 commit SHA 固定，.github/workflows/codeql-analysis.yml）。**仓库内没有 `docker compose up` 自动验证流水线**：样例可运行性依赖维护者人工验证与社区 issue 反馈，这是本仓库工程质量上最薄弱的一环（截至本次快照未确认有外部验证 bot）
- **维护节奏**：2026-10-26 仍有推送（gh api），439 个 open issue 说明需求旺盛但消化偏慢
- **治理**：CODEOWNERS 一行为 `* aiordache glours`，所有改动须经这两人；MAINTAINERS 以 TOML 声明核心维护者，两人邮箱均为 @docker.com——确证 Docker 公司官方维护身份

## 9. 生态位与增长动因

生态位上，它是 docker/compose 的"陪跑样例库"与 docs.docker.com 的教程落地页：官方文档 quickstart 引用它，Docker Desktop 的 Wasm 能力也借它的 wasm 图标示例做展示（README.md）。涨星动因可归纳为四点：一是 **Docker 官方组织背书**，搜索权重与信任度天然最高；二是 **痛点刚需**，"我想要 nginx+flask+mongo 的最小可跑配置"是每个容器新手的真实搜索词，目录名即答案；三是 **Fork 工厂效应**，8347 个 fork 中大量被直接改造成自己项目的起点，用完顺手点星；四是 **低门槛高杠杆**，CC0 许可（LICENSE）允许任意复制改造，无任何使用摩擦。40.5k 星对内容仓库而言，本质是"Docker 生态第一入口"的位置价值。

## 10. 总结评价

awesome-compose 是内容型仓库里的精品范式：官方维护、结构强约定、每个条目自包含可验证、安全细节（secrets、ignore-scripts、非生产声明）到位。它的局限也来自定位——样例刻意最小化、部分基础镜像版本偏旧（如 golang:1.18、mariadb:10.6，见各 Dockerfile/compose.yaml）、无自动可运行性 CI、明示不可上生产。适合三类人：容器初学者当"第一份能跑的 compose 作业"；搭建自托管服务的人抄 gitea/nextcloud/pihole 等平台配置；写技术教程的人把它当事实标准引用。读"源码"的正确姿势是横向对比同类示例的 compose.yaml 差异，而非纵向读某个应用。
