# filebrowser/filebrowser 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Go｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：**自托管的单二进制 Web 文件浏览器**——把服务器上一个目录变成带上传/下载/预览/编辑/分享功能的"私有云盘"，Go 后端 + Vue 3 前端全部嵌入一个可执行文件。

| 项目 | 事实 |
|---|---|
| 名称 | filebrowser/filebrowser（module `github.com/filebrowser/filebrowser/v2`） |
| 星数 | 39500（快照）；gh api 当前 35841，forks 4202 |
| 语言 | Go 1.25 后端（约 1.1 万行）+ TypeScript/Vue 3 前端（约 1.4 万行） |
| 许可证 | Apache License 2.0（`LICENSE`） |
| 存储依赖 | 嵌入式 bbolt（经 asdine/storm），无外部数据库 |
| 默认分支 | master（gh api 核实） |
| 生命周期 | **2026-09-01 正式归档，停止维护**（README 顶部声明 + gh api `archived: true`） |
| 最后版本 | v2.63.23（2026-07-27，`CHANGELOG.md`） |
| 最后 push | 2026-07-31（gh api） |
| 贡献者 | 261 人（gh api contributors 分页统计） |

## 2. 项目是什么，解决什么问题

File Browser 解决的问题非常朴素：服务器上有一堆文件，需要一个不依赖 FTP/SSH 客户端、开箱即用的 Web 界面来管理它们。README（`README.md`）将其定位为 "create-your-own-cloud" 软件——安装到服务器、指向一个路径，就能通过浏览器完成文件的上传、删除、预览、编辑与分享。

它的核心价值主张是**部署成本极低**：整个应用（含前端资产）编译为一个二进制文件（`frontend/assets.go` 通过 `go:embed dist/*` 嵌入前端构建产物），数据存储用嵌入式 bbolt 而非 MySQL/PostgreSQL（`storage/bolt/bolt.go`），因此典型部署只需要一个 Docker 容器加一个挂载卷（`Dockerfile`、`compose.yaml`）。这与 Nextcloud 这类需要完整 LAMP 栈、数据库与应用生态的"全家桶"网盘形成鲜明对比；相比 filestash 面向 S3/FTP 等多后端的云控制台，filebrowser 始终聚焦"本地文件系统 + 多用户目录隔离"这一件事（此段为生态位评述，非源码结论）。

多用户与权限体系是它的另一根支柱：每个用户有独立的 `Scope`（目录范围）、8 项细粒度权限（admin/execute/create/rename/modify/delete/share/download）、路径规则（allow/deny）与个人偏好（`users/users.go`、`users/permissions.go`）。管理员可以建一个只读某子目录、隐藏 dotfile、禁止下载的受限账户。

需要特别说明的是项目现状：README 顶部用醒目的 WARNING 声明 File Browser 已于 2026-09-01 归档，"不会再有发布、修复或安全修复"。归档前最后一个月（2026-07）项目密集发布了 v2.63.18 至 v2.63.23 六个版本（`CHANGELOG.md`），集中修复了一批安全漏洞（路径规范化、TUS 上传越权、规则绕过等），随后宣布终结。README 同时点明两类未修复的结构性问题：命令执行功能（runner/hooks）漏洞频发（#5199），以及自包含 JWT 会话无法撤销（#5216）。

## 3. 技术栈与依赖

后端（`go.mod`，Go 1.25.0）：

| 依赖 | 用途 |
|---|---|
| `gorilla/mux` | HTTP 路由（`http/http.go`） |
| `gorilla/websocket` | /api/command 的交互式 shell（`http/commands.go`） |
| `golang-jwt/jwt/v5` | 会话令牌（HS256，`http/auth.go`） |
| `asdine/storm/v3` + `go.etcd.io/bbolt` | 嵌入式 KV 存储，映射 users/share/settings/auth 四类对象（`storage/bolt/bolt.go`） |
| `spf13/cobra` / `viper` / `pflag` / `afero` | CLI 框架、配置、虚拟文件系统抽象（`cmd/`、`files/scoped.go`） |
| `mholt/archives` | 多格式压缩/解压与流式打包下载（`http/raw.go`） |
| `disintegration/imaging` + `go-exif` | 缩略图与 EXIF/图片分辨率（`http/preview.go`、`files/file.go`） |
| `redis/go-redis/v9` + `jellydator/ttlcache` | 上传缓存可插拔后端（`http/upload_cache_redis.go`、`upload_cache_memory.go`） |
| `shirou/gopsutil/v4` | 磁盘用量（`http/data.go` 相关的 /api/usage） |
| `maruel/natural` | 文件名自然排序（`files/sorting.go`） |
| `gopkg.in/natefinch/lumberjack.v2` | 日志轮转 |

前端（`frontend/package.json`，Vue 3.5 + TypeScript + Vite，pnpm ≥10、Node ≥24）：

- 状态：Pinia（`frontend/src/stores/`：auth/file/upload/clipboard/layout）
- 路由：vue-router；i18n：vue-i18n，**34 种语言**（`frontend/src/i18n/` 34 个 json）
- 大文件上传：`tus-js-client`（对接后端 TUS 协议）；编辑器：ace-builds；视频：video.js；电子书：epubjs/vue-reader；Markdown：marked + KaTeX；净化：dompurify
- 测试：vitest（`frontend/package.json` scripts.test）

构建编排：[Taskfile](https://taskfile.dev)（`Taskfile.yml`），发版用 commit-and-tag-version。

## 4. 目录结构与代码组织

| 目录 | 职责（出处：仓库顶层 `ls`） |
|---|---|
| `main.go` | 13 行入口，转发到 `cmd.Execute()`（cobra） |
| `cmd/` | CLI 子命令：users/config/rules 的增删改查导入导出、`docs` 命令生成 CLI 文档 |
| `http/` | 全部 HTTP handler（约 25 个 .go 文件）+ JWT 中间件 + 静态资源服务 |
| `files/` | 文件/目录抽象：FileInfo、列目录、排序、mime、`ScopedFs` 作用域文件系统 |
| `users/` | 用户模型、8 项权限、per-user 规则与 scope、用户存储 |
| `auth/` | Auther 接口与四种实现：json（密码+可选 reCAPTCHA）、proxy（反代头注入+自动建户）、hook（外部命令认证）、none |
| `settings/` | 全局 Settings（品牌/认证方式/规则/TUS/密码长度/文件权限位）与 Server 配置 |
| `share/` | 分享链接模型（hash/path/userID/过期/密码哈希/下载 token） |
| `storage/` | 存储门面 + bolt 实现；`bolt/` 把 users、share、settings、auth 各 backend 装配起来 |
| `rules/` | allow/deny 路径规则（字面路径或正则），路径匹配支持大小写折叠 |
| `runner/` | before/after 事件命令钩子（受 EnableExec 总开关约束） |
| `search/` | 目录递归搜索（conditions + search） |
| `diskcache/` | 缩略图缓存（文件实现 + noop 实现） |
| `frontend/` | Vue 3 + Vite 工程；`assets.go` 提供嵌入版（`//go:embed dist/*`，`!dev` 构建标签） |
| `docs/` | 安装/认证/部署/自定义/命令执行文档 + 自动生成的 CLI 文档 |
| `docker/`、`Dockerfile`、`compose.yaml` | 多阶段构建（alpine 拉依赖 → busybox-musl 运行，非 root + tini），compose 附带 redis 上传缓存示例 |

组织方式评述：经典 Go 单体 + 单前端子目录，不是 monorepo。后端按"领域包"划分（files/users/auth/settings/share/storage/rules/runner），http 层只做协议适配，数据落盘集中在 storage 层，层次清晰、依赖方向单一。前后端通过 `//go:embed` + 自定义模板定界符（`http/static.go`）缝合为单二进制。

## 5. 核心架构与关键实现

### 5.1 路由与请求管线（http/http.go、http/data.go）

`NewHandler` 用 gorilla/mux 注册全局路由：`/health`、`/static`、SPA fallback（NotFoundHandler=index）以及 `/api` 子路由（login/signup/renew、users CRUD、resources、tus、shares、settings、raw、preview、command、search、subtitle、public 分享下载）。每个 handler 都是 `handleFunc(w, r, d *data) (int, error)` 签名，`handle()` 包装器统一注入全局安全头、逐请求加载 settings、错误码转 `http.Error`（`http/data.go:87`）。

中间件栈 `withUser`/`withAdmin`（`http/auth.go:116`）强制 JWT：只接受 HS256、要求携带 exp，5 秒钟窗容忍过期后的 renew。每个请求的授权判断由 `data.Check()` 实现（`http/data.go:37`）：先按 HideDotfiles 拦截隐藏文件，再级联匹配全局规则与用户规则——后匹配者覆盖前者（last-match-wins），路径先经 `rulePath` 统一分隔符并对公共分享场景重挂 `checkerPrefix`，防止规则绕过。

### 5.2 ScopedFs：作用域文件系统与符号链接逃逸防护（files/scoped.go）

这是本项目最有代表性的安全设计。每个用户的文件系统由 `users.User.Clean()` 构建：`u.Fs = files.NewFs(afero.NewOsFs(), scope, followExternalSymlinks)`（`users/users.go:91`）。默认返回 `ScopedFs`——在 `afero.BasePathFs` 词法限定之上，对**每个会解引用符号链接的操作**（Open/Create/Mkdir/Rename/Remove/Chmod…）执行 `guard()`：用 `filepath.EvalSymlinks` 解析真实落点，再校验是否仍落在 scope 根之内（`files/scoped.go:96` 的 `within()`）。细节处理相当严谨：

- 不存在的新路径按最近存在祖先校验；悬空符号链接则跟读一层（上限 255 跳），按**真实父目录**解析相对目标，防止写入越界；
- 前缀比较强制带路径分隔符，避免 `/srvother` 被误判为 `/srv` 的子路径；
- opt-in 的 `FollowExternalSymlinks` 关掉该防护（`settings.Server` 字段，`settings/settings.go:73`）。

### 5.3 上传与下载：TUS 断点续传 + 多格式流式打包

上传走 TUS 协议（`http/tus_handlers.go`）：POST/HEAD/PATCH/DELETE 完整实现，上传会话登记在可插拔缓存（内存 ttlcache 或 Redis，`http/upload_cache_memory.go`、`upload_cache_redis.go`），传输期间由 `keepUploadActive` 每 2 秒 Touch 防逐出。被拒绝的 PATCH 会主动排干最多 32MB 请求体以保住连接复用，避免客户端把 409 当网络故障无限重试（`tus_handlers.go:26` 注释）。

下载由 `rawHandler`（`http/raw.go`）承担：单文件直出；目录/多选文件用 `mholt/archives` 现场流式打包，支持 zip/tar/targz/tarbz2/tarxz/tarlz4/tarsz/tarbr/tarzst 九种格式（`parseQueryAlgorithm`，`raw.go:31`）；附 md5/sha1/sha256/sha512 校验和分支；`Content-Disposition` 按 RFC6266 区分 inline/attachment 并做 `filename*=utf-8''` 转义。

### 5.4 单二进制嵌入与运行时配置注入

`frontend/assets.go` 用 `//go:embed dist/*` 把整个前端构建产物编进二进制（`!dev` 标签下）。`http/static.go` 的 `getStaticHandlers` 里，index.html 不是原样返回，而是用自定义定界符 `[{[ ]}]` 的 Go template 执行，把品牌名、主题色、认证方式、版本号、TUS 配置、EnableExec 等运行时配置注入 SPA；JS 资产读取预压缩的 `.gz` 版本按 Accept-Encoding 直出或现场解压（`static.go:142`）。品牌自定义支持外部 `custom.css` 与图片覆盖（`static.go:120`）。

### 5.5 存储与分享

`storage.Storage` 是四个子存储的聚合门面（`storage/storage.go:14`），bolt 实现用 storm 的 struct tag 映射：User 有 `storm:"id,increment"`、username unique，Share.Link 有 hash 主键 + path 索引（`storage/bolt/bolt.go`、`share/share.go`）。分享链接支持过期时间与密码（bcrypt 哈希），带密码的分享还会生成 URL-Safe 的下载 token 供 query 参数直链下载（`share/share.go:12`）。

## 6. 构建与运行

构建（`Taskfile.yml`、`CONTRIBUTING.md`）：

```bash
# 前端（frontend/ 内）
pnpm install --frozen-lockfile && pnpm run build   # 含 vue-tsc 类型检查
# 后端（注入版本 ldflags）
go build -ldflags='-s -w -X ".../version.Version=..." -X ".../version.CommitSHA=..."' -o filebrowser .
# 一键
task build          # 需要 go-task
task docs:cli:generate   # 重新生成 docs/cli
```

开发模式：后端 `go run . --address 127.0.0.1 --port 8080 -r /path/to/files`；前端 `pnpm run dev`（Vite 代理，需走前端 dev server 访问）。

测试：后端 `go test ./...`（根目录 testify 在依赖中）；前端 `pnpm test`（vitest）。

容器运行（`Dockerfile`、`compose.yaml`）：

```bash
docker run -v /path/on/host:/srv -v filebrowser.db:/database \
  -p 8080:80 filebrowser/filebrowser
```

官方镜像基于 busybox-musl，非 root 用户（UID/GID 1000）+ tini 运行；compose 示例附带 Redis 作上传缓存。

## 7. 测试与工程质量

- 后端测试：33 个 `_test.go`，覆盖高度对准安全面——`scoped` 相关、规则路径（`http/rules_path_test.go`、`rules_recursive_test.go`）、TUS 越权（`tus_symlink_test.go`、`tus_upload_length_test.go`、`tus_multichunk_test.go`）、公共分享符号链接（`public_symlink_test.go`）、大小写折叠冲突（`files/case_test.go`）、递归操作（`resource_recursive_test.go`）。风格是标准库 + testify 的表驱动单测，无端到端框架。
- 前端测试：5 个 spec 文件（vitest），覆盖远低于后端；`pnpm run build` 强制先过 `vue-tsc` 类型检查作为质量闸门（`frontend/package.json`）。
- CI（`.github/workflows/ci.yaml`）：push master/tag/PR 触发，前端 lint + test + build、后端 build + test、Docker 构建；另有 `lint-pr.yaml` 强制语义化 PR 标题（配合 commit-and-tag-version 自动发版）。
- 代码质量观察：注释密度高且多为"为什么"型注释（如 `tus_handlers.go` 对 drain 字节数的推理、`scoped.go` 对悬空符号链接的逐段论证），核心安全路径有对应测试；CHANGELOG 由工具生成、条目规整。整体工程成熟度高。

## 8. 社区与维护状态

- **已归档**：gh api `archived: true`；README 声明 2026-09-01 归档、最后计划版本已发布。作者 hacdias 发布了告别文《Goodbye File Browser, for Real This Time》（2026-07-28，README 引用）。
- 发布节奏：生命末期极密——v2.63.18（07-04）→ v2.63.23（07-27）六个版本，多数为安全修复（`CHANGELOG.md`）。长期采用 commit-and-tag-version 自动版本管理。
- 贡献者结构：261 名贡献者（gh api 分页统计），历史上由 hacdias 与社区共同维护；归档后官方转向建议 fork 接力（README："CONTRIBUTING.md … remains useful to anyone forking it"）。社区还存在独立延续版（生态情况未在源码中确认）。
- 量化活跃度：最后 push 2026-07-31；open issues 归零（归档动作）。

## 9. 亮点与风险

亮点：

1. **单二进制即产品**：前端 embed + bbolt 嵌入式存储，部署面积极小，是 Go 自托管工具的教科书式形态（`frontend/assets.go`、`storage/bolt/bolt.go`）。
2. **ScopedFs 逐操作越界防护**：符号链接逃逸的边界条件（悬空链接、相对链接、大小写折叠、255 跳上限）处理完整且有专项测试（`files/scoped.go`、`http/tus_symlink_test.go`）。
3. **规则引擎分层**：全局规则 + 用户规则级联、last-match-wins 语义简单可预测，且规则匹配考虑了大小写不敏感文件系统的绕过面（`rules/rules.go:32`、`http/data.go:48`）。
4. **现代上传/下载管线**：TUS 断点续传 + 可插拔缓存后端，九种压缩格式的现场流式打包下载（`http/tus_handlers.go`、`http/raw.go`）。
5. **运维细节成熟**：非 root 容器、tini、日志轮转、预压缩静态资产、CSP 等安全头默认开启。

风险/局限：

1. **项目已死**：无补丁、无安全修复，README 明确要求"当作无人维护的软件对待"——生产使用必须前置反代认证并容器化降权（`README.md` Security 一节）。
2. **命令执行功能结构性不安全**：runner/hooks 多年漏洞缠身，需整体重写才可能安全；默认关闭，重开等于把主机 shell 交给用户（#5199）。
3. **JWT 无法撤销**：自包含令牌意味着登出/改密不失效，泄漏即有效到期（#5216）；令牌过期时间默认可配置但泄漏窗口始终存在。
4. **权限模型只有全局 8 项布尔位**：无目录级 ACL、无角色组，复杂场景（部门隔离、细粒度授权）需靠 per-user scope + 规则拼凑（`users/permissions.go`）。

## 10. 总结评价

File Browser 是"小而美"自托管工具的典范：用最小的依赖面（一个二进制 + 一个挂载卷）交付了完整的多用户文件管理体验，其 ScopedFs、规则引擎与 TUS 上传管线即使在归档后仍值得 Go 后端开发者精读。但它也以最彻底的方式演示了一个长期维护的安全敏感产品的终点——作者在归档前一个月集中还清了能还的安全债，并诚实地把还不清的两笔（命令执行、可撤销会话）写进了 README。它适合个人/NAS/内网场景继续使用（务必加反代认证、关 exec），适合作为学习"Go 单二进制 Web 产品如何组织分层、嵌入前端与防护文件系统"的源码教材；不适合需要持续安全支持的生产多租户部署——那应该去看它的社区 fork 或转向 Nextcloud/filestash 类仍在活跃维护的方案。
