# jeecgboot/JeecgBoot 项目分析

> 星数 44600（快照 2026-10-24）｜主语言 Java｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：企业级 AI 低代码开发平台，前后端分离（Spring Boot + Vue3），内置权限、工作流、报表大屏、AI 应用/知识库与代码生成器，是 Java 生态最流行的开源业务系统基座之一。

| 项目 | 值 |
|---|---|
| 名称 | jeecgboot/JeecgBoot（当前版本 3.9.6，见 `README.md`） |
| 星数 | 44600（快照）/ 48373（分析当日 gh api 实测） |
| 语言 | Java（后端 995 个 .java 文件）+ TypeScript/Vue（前端 918 个 .vue、667 个 .ts，实测统计） |
| 许可证 | Apache-2.0（`LICENSE`，且 `jeecg-boot/LICENSE`、`jeecgboot-vue3/LICENSE` 各有一份） |
| 默认分支 | main；最近 push 2026-11-04（gh api，分析前一天，极活跃） |
| 维护方 | 北京国炬软件（jeecg.com，`package.json` author 字段） |

## 2. 项目是什么，解决什么问题

JeecgBoot 定位是"AI 时代构建复杂业务系统的开源基座"（`README.md`）。它要解决的核心问题是：企业业务系统（OA/ERP/CRM/MIS 类）80% 的工作是重复的——用户、角色、组织、菜单、数据字典、按钮/数据/字段权限、审批流、报表导入导出、多租户隔离，这些每个项目都要重做一遍。平台把这些通用能力内置成开箱即用的模块，开发者只需生成或手写"业务差异部分"。

平台的低代码能力分三层（`README.md`）：一是 Online 表单零代码模式，基于元数据驱动直接配置出可运行页面（实现以闭源构件 `org.jeecgframework.boot3:jeecg-online:3.9.6-beta` 依赖引入，见 `jeecg-module-system/jeecg-system-biz/pom.xml`）；二是代码生成器，针对单表/树表/一对一/一对多数据模型一键生成前后端代码与建表 SQL（生成器引擎为外部构件 `org.jeecgframework.boot:codegenerate:1.6.0`，见 `jeecg-boot-base-core/pom.xml`，仓库内保留其启动入口 `jeecg-module-system/jeecg-system-start/src/main/java/org/jeecg/codegenerate/JeecgOneGUI.java`）；三是 3.9.x 新增的 AI Skills 路线，配合 Claude Code 用自然语言生成建表 SQL、前后端代码、菜单权限（`README-AI.md`、`jeecg-boot/CLAUDE.md`）。

3.9.x 版本最大的增量是 AI 能力栈：后端新增了完整的 `jeecg-boot-module-airag` 模块（AI 应用 + RAG 知识库），提供 AI 应用管理、大模型管理、流程编排、OCR、语音、视频、知识库文档解析等子包（`jeecg-boot/jeecg-boot-module/jeecg-boot-module-airag/src/main/java/org/jeecg/modules/airag/`，含 `app/llm/ocr/voice/video/prompts/wordtpl` 子目录），README-AI.md 明确对标 Dify。部署编排中甚至包含 pgvector 容器作为向量库（`docker-compose.yml`）。

适用场景与边界：README 明确列出适合表多、流程多、权限复杂的企业系统与信创国产化（达梦/人大金仓/东方通，`README.md` 信创兼容说明）；不适合纯 C 端产品和非 Java 技术栈团队。

## 3. 技术栈与依赖

后端（`jeecg-boot/pom.xml`，关键版本均为实测）：
- Spring Boot 4.1.0（spring-boot-starter-parent 4.1.0）、Java 17（CLAUDE.md 注明同时支持 21/24），全面使用 `jakarta` 命名空间（`jeecg-boot/CLAUDE.md`）
- Spring Cloud 2025.1.2 + Spring Cloud Alibaba（微服务模式，Nacos 注册/配置中心）
- MyBatis-Plus（含租户插件 TenantLineInnerInterceptor）、Druid、Redis
- Shiro 3.0.0 + shiro-redis 4.1.0 + java-jwt 4.5.0（认证体系）
- Flowable（BPM 工作流，README 说明）、xxl-job 3.4.2（定时任务）、knife4j 4.5.0（API 文档）
- 积木报表 jimureport-spring-boot4-starter 2.5.6.1、积木大屏 jimubi 2.5.6.1（同为 jeecgframework 系闭源构件）
- JustAuth（第三方登录）、MinIO 9.0.3（对象存储）

前端（`jeecgboot-vue3/package.json`）：Vue 3.5.39、Ant Design Vue 4.2.6、TypeScript 5.9.3、Vite 8.1.4、Pinia 3.0.4、pnpm workspace，附带 qiankun 微前端目录（`src/qiankun/`）与 Electron 打包支持（`electron/`、`electron-builder.yaml`）。

## 4. 目录结构与代码组织

Monorepo 双端结构：一个仓库装下后端 Maven 多模块工程与前端独立工程。

| 目录 | 职责 |
|---|---|
| `jeecg-boot/` | 后端 Maven 多模块工程（单体默认，微服务按 profile 开启） |
| `jeecg-boot/jeecg-boot-base-core/` | 技术底座：Shiro/JWT 配置、通用切面（日志/字典/数据权限）、MyBatis-Plus 配置、OSS/签名/防火墙配置 |
| `jeecg-boot/jeecg-module-system/` | 系统核心业务：`jeecg-system-api`（跨模块 API 接口）、`jeecg-system-biz`（用户/角色/菜单/部门/租户等实现）、`jeecg-system-start`（单体启动器） |
| `jeecg-boot/jeecg-boot-module/` | 扩展业务模块：`jeecg-boot-module-airag`（AI+RAG）、`jeecg-module-demo`（示例） |
| `jeecg-boot/jeecg-server-cloud/` | 微服务模块：网关、Nacos、system-cloud-start、demo-cloud-start、jeecg-visual（monitor/sentinel/xxljob/seata 测试） |
| `jeecg-boot/db/` | 数据库脚本：`jeecgboot-mysql-5.7.sql`（8.2MB，143 张表）+ nacos/xxl-job 建表脚本 + 国产库转换说明 |
| `jeecgboot-vue3/` | 前端 Vue3 工程：src/views 分 dashboard/monitor/openapi/report/super/sys/system 等 8 大区 |
| `docker-compose.yml` / `docker-compose-cloud.yml` | 单体编排（mysql/redis/system/vue/pgvector）与云编排（另含 nacos/xxljob/gateway） |
| `check_jeecgenv.py`、`start-docker-compose.*` | 环境自检脚本与一键启动脚本 |

组织方式评述：后端按"底座 → 系统 → 扩展 → 部署形态"四层划分，模块边界清晰；业务扩展点通过 `jeecg-system-api` 接口解耦（如 ShiroRealm 只依赖 CommonAPI，不依赖实现）。值得注意：Online 表单、代码生成器、积木报表三个招牌能力都以外部构件（部分闭源）形式集成，开源仓库内只有调用与集成代码。

## 5. 核心架构与关键实现

### 5.1 认证体系：Shiro + JWT（无状态、单体/微服务复用）

鉴权链路为：请求携带 `X-Access-Token` 头 → JwtFilter 提取并包装成 JwtToken → ShiroRealm 校验。`JwtFilter` 继承 `BasicHttpAuthenticationFilter`，在 `executeLogin` 中从自定义头（或 URL 参数）取 token，并在 `preHandle` 中处理跨域与多租户头透传（`jeecg-boot-base-core/src/main/java/org/jeecg/config/shiro/filters/JwtFilter.java`）：

```java
JwtToken jwtToken = new JwtToken(token);
getSubject(request, response).login(jwtToken);
// 多租户用到
String tenantId = httpServletRequest.getHeader(CommonConstant.TENANT_ID);
TenantContext.setTenant(tenantId);
```

`ShiroRealm` 只依赖 `CommonAPI` 接口查询角色与权限集合，不直接依赖数据库实现（`jeecg-boot-base-core/src/main/java/org/jeecg/config/shiro/ShiroRealm.java` 第 66-88 行），因此单体与微服务共用同一套 Realm，云模式下由网关或 Feign 提供 CommonAPI 实现。免登路径通过 `@IgnoreAuth` 注解 + `InMemoryIgnoreAuth` 内存注册表实现（`jeecg-boot-base-core/src/main/java/org/jeecg/config/shiro/IgnoreAuth.java`），ShiroConfig 中再对 `/sys/login`、验证码等路径逐条放行（`ShiroConfig.java` 第 99-114 行）。JWT 工具类为 `org/jeecg/common/system/util/JwtUtil.java`（393 行），配合 shiro-redis 实现同一账号多地登录控制等会话策略。

### 5.2 权限体系：RBAC + 三层粒度

菜单/按钮权限走经典 RBAC：`jeecg-module-system/jeecg-system-biz/.../controller/` 下有 SysUserController、SysRoleController、SysPermissionController、SysDepartController、SysTenantController 等 35 个控制器，覆盖用户/角色/菜单/部门/租户/数据字典/数据源/白名单等（实测目录列表）。数据权限通过 AOP 实现：`PermissionDataAspect` 拦截 `@PermissionData` 注解，往当前 request 写入数据权限规则（`jeecg-boot-base-core/src/main/java/org/jeecg/common/aspect/PermissionDataAspect.java`），配合前端系统视图中的角色授权页面（`jeecgboot-vue3/src/views/system/role/` 等）形成按钮、数据、字段三层颗粒度。多租户则是 MyBatis-Plus 行级隔离：`MybatisPlusSaasConfig` 注册 `TenantLineInnerInterceptor`，注释明确"先 add TenantLineInnerInterceptor 再 add PaginationInnerInterceptor"（`jeecg-boot-base-core/src/main/java/org/jeecg/config/mybatis/MybatisPlusSaasConfig.java` 第 91-131 行）。

### 5.3 微服务形态

`jeecg-server-cloud` 模块仅在 Maven profile 下参与构建（根 `pom.xml` 的 `<profile>` 中 `<module>jeecg-server-cloud</module>`），包含：`jeecg-cloud-gateway`（Spring Cloud Gateway，含 `GlobalAccessTokenFilter` 全局令牌过滤、fallback、动态路由）、`jeecg-cloud-nacos`、`jeecg-system-cloud-start`/`jeecg-demo-cloud-start`（两个服务的启动器）、`jeecg-visual`（Spring Boot Admin 监控、Sentinel 流控、xxl-job 调度、Seata 分布式事务测试模块）。docker-compose-cloud.yml 一键拉起 mysql/redis/nacos/xxljob/gateway/system/demo/vue/pgvector 十个容器，是理解部署拓扑的最好入口。

### 5.4 AI/RAG 模块

`jeecg-boot-module-airag` 是 3.9.x 的战略增量，子包划分显示其能力矩阵：`app`（AI 应用）、`llm`（模型接入，README 称兼容 ChatGPT/DeepSeek/Ollama）、`ocr/voice/video`（多模态）、`prompts`（提示词）、`wordtpl`（文档模板）、`api`（对外接口）。该模块自带独立测试目录（`src/test/java/org/jeecg/modules/airag/test`）。docker-compose 中 pgvector 服务即为知识库向量存储而设（`docker-compose.yml`）。

### 5.5 前端组织

前端 `jeecgboot-vue3/src` 按经典分层组织：`api/`（接口层）、`components/`（通用组件）、`hooks/`、`router/`（含 guard/ 路由守卫与 menus/ 动态菜单）、`store/`（Pinia）、`layouts/`、`locales/`（国际化）。视图区按业务域划分 8 大目录（实测 `src/views/`）：`system/` 是最大的一块，含 user/role/menu/depart/tenant/dict/checkRule/fillRule/onlineuser/tableWhiteList 等约 24 个子模块，与后端 SysXxxController 一一对应；`monitor/` 对应系统监控，`report/` 对应积木报表/大屏入口，`super/` 为租户超级管理员视图，`sys/` 为登录与个人设置。另有 `qiankun/` 目录支持微前端接入、`electron/` 支持桌面端打包，以及 `mock/` 与 `jest.config.mjs` 构成本地开发链路。

## 6. 构建与运行

来自 `jeecg-boot/CLAUDE.md` 与 `README.md`（未实际运行验证）：

```bash
# 后端全量构建（surefire 默认跳过测试）
mvn clean package
# 启动单体（端口 8080，context-path /jeecg-boot）
cd jeecg-module-system/jeecg-system-start && mvn spring-boot:run
# 带微服务模块构建
mvn clean package -Pcloud   # CLAUDE.md "Build with microservices modules included"

# 前端（jeecgboot-vue3/package.json scripts）
pnpm install && pnpm dev && pnpm build

# 容器化一键启动
docker-compose up -d              # 单体
./start-docker-compose-cloud.sh   # 微服务全家桶
```

数据库初始化：导入 `jeecg-boot/db/jeecgboot-mysql-5.7.sql`（143 张表）及 nacos/xxl-job 脚本；Oracle/PG/达梦等参考 `db/其他数据库脚本/其他数据库转换方法.md`。

## 7. 测试与工程质量

- 后端测试薄弱：全仓 `src/test` 下仅 40 个 Java 测试文件（实测），demo 模块内多为功能演示性质测试；构建默认跳过测试（surefire 配置，`CLAUDE.md` 注明）。前端有 `tests/` 目录与 jest.config.mjs，规模有限。
- 无 CI：`.github/` 下只有 ISSUE_TEMPLATE 与 SECURITY.md，没有 workflows 目录，GitHub Actions 流水线缺失（实测），质量保障依赖人工与商业版流程。
- 工程质量观察：模块划分规范、注释完整（大量中文注释含 issue 编号溯源，如 JwtFilter 中 "JT-355"）；对安全有专门投入（`config/firewall/`、接口签名 `config/sign/`、表白名单 SysTableWhiteListController、脱敏包 desensitization）。根目录 CLAUDE.md/AGENTS.md 是为 AI 编程工具准备的工程说明，本身也是质量信号。

## 8. 社区与维护状态

- 活跃度极高：gh api 显示最近 push 为 2026-11-04（分析前一天），当前星数 48373、fork 16221、watch 840；README 中版本 3.9.6 标注发布日期 2026-11-05，与快照窗口吻合，发版密度高（"未确认"具体年发版次数，tarball 无 CHANGELOG 文件）。
- 贡献者结构：主体为北京国炬软件团队（README 徽章、package.json author），tarball 无法统计 contributor 名单（未确认个人贡献者占比）；文档生态（help.jeecg.com）、商业版与闭源构件构成商业闭环。
- 治理细节：提供安全披露渠道（`.github/SECURITY.md`）与双语/三语 README（en-US、ja-JP）。

## 9. 亮点与风险

亮点：
1. 企业级能力完整度罕见：权限（按钮/数据/字段三层）、多租户（TenantLineInnerInterceptor 行级隔离）、工作流、报表大屏、定时任务、多数据库/信创适配一次到位（第 3、5 节所引路径）。
2. 单体/微服务同构切换设计好：ShiroRealm/CommonAPI 等接口化隔离让同一套业务代码跑两种形态，cloud 模块按 profile 隔离不污染单体构建（`pom.xml`、`ShiroRealm.java`）。
3. AI 战略落地快且深：airag 模块 + pgvector 编排 + CLAUDE.md/AGENTS.md AI 工程说明，从"低代码"转向"AI 低代码基座"路线清晰（`README-AI.md`）。
4. 工程文档一流：CLAUDE.md 里构建/测试/模块说明精确到命令与端口，可直接当 onboarding 文档用。

风险/局限：
1. 测试与 CI 缺位：40 个测试文件 + 无 GitHub Actions，核心链路回归靠人工，二次开发需自行补测试基建。
2. 关键能力闭源依赖：Online 表单、代码生成器、积木报表均为 jeecgframework 外部构件（`jeecg-boot-base-core/pom.xml`、`jeecg-system-biz/pom.xml`），深度定制受制于上游发布节奏，且需从阿里云私仓/官方源拉取（pom 注释"如下载失败，请参考此文档"）。
3. 技术栈激进：Spring Boot 4.1.0 + Java 17 + Vite 8 均为新版本，存量生态兼容性问题需自行评估；db 脚本仍以 mysql-5.7 命名，老库迁移注意。
4. 学习曲线：143 张表、35 个系统控制器、多套权限/租户上下文（TenantContext + Shiro + JWT），新人上手成本不低。

## 10. 总结评价

JeecgBoot 是国内 Java 低代码赛道事实上的标杆项目：以完整的企业级基座能力（权限/租户/流程/报表）换取"业务系统从脚手架开始"的定位，再用代码生成器与 AI Skills 把 80% 重复工作自动化，并在 3.9.x 快速叠加了 Dify 式 AI 应用平台。代码组织规范、单体与微服务双形态切换设计成熟，但核心低代码引擎部分依赖闭源构件、测试与 CI 薄弱。适合需要快速交付复杂企业系统的 Java 团队直接使用，也适合学习"权限/租户/网关"工程实践的架构师精读 `jeecg-boot-base-core` 与 `jeecg-server-cloud` 两个模块；想研究低代码引擎本身（元数据驱动、代码生成）的同学则需意识到这部分源码并不在本仓库内。
