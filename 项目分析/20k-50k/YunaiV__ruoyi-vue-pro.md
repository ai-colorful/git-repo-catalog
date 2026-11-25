# YunaiV/ruoyi-vue-pro 项目分析

> 星数 38500（快照 2026-10-24）｜主语言 Java｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：若依（RuoYi-Vue）生态中最活跃的深度重构分叉——「芋道源码」，以 Spring Boot 多模块单体实现的权限管理系统底座，向上长出覆盖商城/ERP/CRM/MES/HRM/IoT/AI 等 18 个业务模块的全家桶式快速开发平台。

| 项目 | 内容 |
|---|---|
| 名称 | YunaiV/ruoyi-vue-pro（芋道 ruoyi-vue-pro，包名 `cn.iocoder.yudao`） |
| 星数 | 38500（快照），GitHub forks 8672、watchers(subscribers) 269 |
| 语言 | Java（后端），前端 Vue2/Vue3/uni-app 独立仓库 |
| 许可证 | MIT（`LICENSE`，声明"Copyright (c) 2021 ruoyi-vue-pro"） |
| 创建 / 最近 push | 2021-01-24 / 2026-11-13（分析前一日仍在提交，`gh api repos/YunaiV/ruoyi-vue-pro`） |
| 版本线 | `master`(JDK8+SB2.7)、`master-jdk17`(JDK17/21+SB3.5)、`master-jdk25`(JDK25+SB4.x)（README「版本说明」表） |
| 代码规模 | 7673 个 Java 文件：主代码约 55.4 万行、测试约 16 万行（本地统计） |
| 贡献者 | 284 人（GitHub contributors 分页统计） |

## 2. 项目是什么，解决什么问题

ruoyi-vue-pro 是在原版若依 RuoYi-Vue 基础上的"重构优化"版：README 明确写道"本项目基于 RuoYi-Vue 修改，重构优化后端的代码，美化前端的界面"，并用 🚀（新增）/⭐️（重新实现）两类标记区分差异（`src/README.md` 133 行附近）。最直观的工程差异是把原版相对平面的工程结构改为严格的 Maven 多模块 + "api/service 分离"的模块化分层，包名换成 `cn.iocoder.yudao`，并以 starter 形式把横切能力（多租户、操作日志、数据权限、安全、MyBatis 封装等）沉淀进 `yudao-framework`（见 `pom.xml` 模块列表与 `yudao-framework/` 目录）。

它解决的核心问题是"企业后台系统的第一公里"：RBAC 动态权限、部门数据权限、SaaS 多租户、Flowable 工作流、支付、短信/邮件/站内信、代码生成、监控告警这些几乎所有企业项目都需要的能力被开箱即用地提供（README「平台简介」）。在此之上按需启用业务模块——完整版包含 system/infra/bpm/pay/report/mp/member/mall/erp/wms/crm/mes/hrm/fms/pms/oa/ai/iot/im 共 20 个模块，另有只含 system+infra 的精简版独立仓库 yudao-boot-mini，官方宣称 5-10 分钟即可把完整版模块按需迁移到精简版（README「版本说明」）。

版本策略上采用"三条 JDK 分支并行维护"：`master` 为 JDK 8 + Spring Boot 2.7.18（根 `pom.xml` `java.version=1.8`、`spring.boot.version=2.7.18`），`master-jdk17` 为 JDK 17/21 + Spring Boot 3.5，`master-jdk25` 为 JDK 25 + Spring Boot 4.x。本次分析的 HEAD 即 `master` 分支，版本号 `2026.10-jdk8-SNAPSHOT`（`pom.xml` revision）。

商业化边界方面，README 开篇"严肃声明：现在、未来都不会有商业版本，所有代码全部开源！"，并以 MIT 协议、对比"其他项目只开源部分代码"作为推荐卖点；变现路径见 README「项目外包」一节（微信 Aix9975，承接商城/SCRM/OA/ERP 等外包）与 doc.iocoder.cn 的文档/视频教程。经核对源码树，业务模块（如 mall、erp、mes）代码均完整入库，未发现"源码收费墙"式文件缺失；SQL 目录（`sql/`，19MB）也提供了 10 种数据库的初始化脚本。结论：代码层全开源属实，商业化以服务/教程为主（基于 README 与源码树核对）。

## 3. 技术栈与依赖

后端核心（版本来自根 `pom.xml`、`yudao-dependencies/pom.xml` 与 README 技术栈表）：

| 组件 | 版本 | 用途 |
|---|---|---|
| Spring Boot | 2.7.18 | 应用框架（master 分支） |
| MyBatis Plus | 3.5.17 | ORM 增强，配套 mybatis-plus-join（MPJ）连表 |
| Druid | 1.2.28 | 连接池 + 监控；dynamic-datasource 多数据源 |
| Redisson | 4.8.0 | Redis 客户端/分布式对象 |
| Spring Security | 5.8.x | 认证授权，Token + Redis 会话 |
| Flowable | 6.8.0 | BPM 工作流（pom 注释明确"不要升级 6.8.1：退回有 bug"，见 `yudao-dependencies/pom.xml`） |
| Quartz | 2.3.2 | 定时任务 |
| springdoc + knife4j | 1.8.0 | API 文档（`yudao-server/src/main/resources/application.yaml`） |
| MapStruct / Lombok | 1.7.0.Beta2 / 1.18.48 | Bean 转换 / 代码简化（根 `pom.xml`） |
| mzt-biz-log | - | Service 层操作日志注解（`com.mzt.logapi.starter.annotation.LogRecord`，见 `yudao-module-system/.../service/permission/RoleServiceImpl.java` import） |

AI 模块技术栈（`yudao-module-ai/pom.xml`）：基于 Spring AI BOM，引入 `spring-ai-starter-model-openai/azure-openai/anthropic/deepseek/ollama/stability-ai`、阿里 DashScope（`spring-ai-alibaba-starter-dashscope`）、向量库 qdrant/redis/milvus、tika 文档读取，以及 `spring-ai-starter-mcp-server-webmvc`（MCP 服务端）。AI 业务能力含 chat/image/knowledge/mindmap/model/music/workflow/write 九组 service（`yudao-module-ai/.../service/` 目录）。

前端三端不在本仓库：`yudao-ui/` 下五个目录均为 README 指引 stub，指向独立仓库 yudao-ui-admin-vue3（element-plus）、yudao-ui-admin-vben（ant-design-vue）、yudao-ui-admin-vue2（element-ui）、yudao-ui-admin-uniapp（管理后台移动端）、yudao-ui-mall-uniapp（商城小程序）；本仓库内 vue3 目录仅含 MES 相关的 4 个 ts 文件与 1 个 vue 文件（`yudao-ui/yudao-ui-admin-vue3/src/`）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `pom.xml` | 根聚合 POM，业务模块靠注释/反注释按需启用，默认仅 system+infra |
| `yudao-dependencies/` | BOM 依赖版本管理（内含 Flowable 6.8.0 等版本锁定注释） |
| `yudao-framework/` | 框架层：`yudao-common` + 14 个 `yudao-spring-boot-starter-*`（见第 5 章） |
| `yudao-server/` | 唯一启动入口（YudaoServerApplication），聚合各模块 |
| `yudao-module-system/` | 系统功能（469 个 Java 文件）：用户/角色/菜单/租户/OAuth2/短信/邮件/站内信 |
| `yudao-module-infra/` | 基础设施（232 个）：代码生成/文件/定时任务/API 日志/监控 |
| `yudao-module-bpm/` 等 18 个 | 业务模块，Java 文件数：mes 1153、mall 856、hrm 629、iot 564、oa 477、pms 312、crm 288、fms 284、bpm 270、ai 255、pay 250、erp 216、member 194、ai1-springai 170、wms 160、im 301、mp 125、report 53 |
| `yudao-ui/` | 五个前端仓库的 README 指引 stub + 少量 MES vue3 文件 |
| `sql/` | 19MB，mysql/oracle/postgresql/sqlserver/db2/dm(达梦)/kingbase/opengauss/highgo/oceanbase 十种数据库脚本 |
| `script/` | docker-compose/jenkins/sharding/sentinel/seata/shell 等部署配套 |

每个业务模块内部遵循统一五层结构：`api`（跨模块 RPC 风格接口）、`controller/admin` + `controller/app`（双端 REST）、`service`、`dal/mysql`（Mapper）+ `dal/dataobject`、`convert`（MapStruct），如 `yudao-module-system/src/main/java/cn/iocoder/yudao/module/system/` 所示。组织方式是"单体多模块"而非微服务：所有模块打进一个 `yudao-server`，模块间只经 `api` 包调用；微服务版是姊妹仓库 yudao-cloud（README「后端项目」表）。

## 5. 核心架构与关键实现

### 5.1 yudao-framework：14 个 starter 的横切能力层

`yudao-framework/` 下除 `yudao-common` 外封装了 14 个 starter：web、security、mybatis、redis、mq、job、excel、test、monitor、protection（幂等/限流/分布式锁）、websocket、biz-tenant（多租户）、biz-data-permission（数据权限）、biz-ip。每个 starter 以 `YudaoXxxAutoConfiguration` 提供自动装配，业务模块按需依赖。这套"框架层与业务层彻底分离"是它区别于原版若依的最大工程特征。

### 5.2 多租户：从 DB 到 MQ 的全链路透明封装

DB 层基于 MyBatis Plus 的 `TenantLineHandler` 做 SQL 级租户隔离（`yudao-framework/yudao-spring-boot-starter-biz-tenant/.../core/db/TenantDatabaseInterceptor.java`）：

```java
public class TenantDatabaseInterceptor implements TenantLineHandler {
    @Override
    public Expression getTenantId() {
        return new LongValue(TenantContextHolder.getRequiredTenantId());
    }
    @Override
    public boolean ignoreTable(String tableName) {
        // 全局忽略（@TenantIgnore 切面）→ 表级忽略缓存
        // computeIgnoreTable: 不在 MP 表信息中→忽略；
        // 继承 TenantBaseDO→强制拦截；标注 @TenantIgnore→忽略
    }
}
```

判定规则是"DO 继承 `TenantBaseDO` 即自动多租户，`@TenantIgnore` 注解豁免"，无需逐表配置。同一 starter 还覆盖了 MQ 四种实现（RocketMQ/Kafka/RabbitMQ/Redis 各自的 message 拦截/initializer）、Web（`TenantContextWebFilter`/`TenantSecurityWebFilter`）、Job（`@TenantJob`）、Redis 缓存（`TenantRedisCacheManager`）与异步任务上下文传递，构成完整的 SaaS 底座（同目录文件列表）。

### 5.3 RBAC 与安全：缓存化的权限判定 + 规则化数据权限

`yudao-module-system/.../service/permission/PermissionServiceImpl.java` 实现用户-角色-菜单三层 RBAC，核心判定带缓存与超管短路：

```java
public boolean hasAnyPermissions(Long userId, String... permissions) {
    List<RoleDO> roles = getEnableUserRoleListByUserIdFromCache(userId);
    if (CollUtil.isEmpty(roles)) return false;
    for (String permission : permissions) {
        if (hasAnyPermission(roles, permission)) return true;
    }
    return roleService.hasAnySuperAdmin(convertSet(roles, RoleDO::getId));
}
```

菜单-角色映射经 `@Cacheable`（Redis）缓存并在变更时 `@CacheEvict`（同文件）。登录认证在 `yudao-spring-boot-starter-security` 的 `TokenAuthenticationFilter` + `YudaoWebSecurityConfigurerAdapter` 完成 Token→LoginUser 解析；数据权限是独立的 `biz-data-permission` starter：`@DataPermission` 注解 AOP（`DataPermissionAnnotationAdvisor`）+ 规则工厂（`DataPermissionRuleFactory`）+ 部门规则（`DeptDataPermissionRule`），在 SQL 改写层注入部门/本人数据范围。操作日志则用 mzt-biz-log 的 `@LogRecord` 注解声明在 Service 方法上（`RoleServiceImpl.java`），starter 内的 `LogRecordServiceImpl` 落库。

### 5.4 MyBatis Plus 使用模式：BaseMapperX 一统天下

全仓 578 个 Mapper 继承自框架层 `BaseMapperX`，仅 2 个直接继承 MP 原生 `BaseMapper`，手写 XML 只有 27 个（`grep -rl "extends BaseMapperX" | wc -l`；`find -name "*.xml" -path "*mapper*"`）。`BaseMapperX`（`yudao-framework/yudao-spring-boot-starter-mybatis/.../mapper/BaseMapperX.java`）声明为 `interface BaseMapperX<T> extends MPJBaseMapper<T>`——即叠加 mybatis-plus-join 获得类型安全连表，再补充 `selectPage`/`insertBatch`/`selectOne("字段", 值)` 等易用方法；配套 `LambdaQueryWrapperX`（空值自动跳过、模糊/区间快捷方法）让单表 CRUD 几乎零 XML。另有 `YudaoSqlInjector` 注入 `SelectForUpdate`、`EncryptTypeHandler` 等字段加密 TypeHandler（同目录）。

### 5.5 代码生成器：106 个 Velocity 模板 × 7 种前端

`yudao-module-infra/src/main/resources/codegen/` 下 106 个 `.vm` 模板，覆盖 java（controller/vo/do/mapper/service）+ sql + vue/vue3/vue3_admin_uniapp/vue3_vben/vue3_vben5_antd/vue3_vben5_antdv_next/vue3_vben5_ele 七种前端风格（目录列表），实现"读表结构→生成前后端+SQL+单测"的一键 CRUD，是 README 强调的主打效率功能。`sql/mysql/ruoyi-vue-pro.sql` 中的 `yudao_demo01_contact` 等 5 张 demo 表即为代码生成的演示对象。

## 6. 构建与运行

- 初始化数据库：导入 `sql/mysql/ruoyi-vue-pro.sql`（system 32 表 + infra 11 表 + 5 demo 表）与 `sql/mysql/quartz.sql`（11 表）；其它 9 种数据库各有同名脚本（`sql/` 各子目录）
- 按需启用模块：编辑根 `pom.xml`，取消对应 `yudao-module-*` 的注释（AI 模块另需按 `doc.iocoder.cn/ai/build/` 配置，pom 注释注明）
- 本地运行：`yudao-server/src/main/resources/application-local.yaml` 配置 MySQL/Redis 后，运行 `yudao-server` 的 `cn.iocoder.yudao.server.YudaoServerApplication`
- 构建：`mvn clean package -Dmaven.test.skip=true`（CI `mvn -B package --file pom.xml -Dmaven.test.skip=true`，见 `.github/workflows/maven.yml`）
- 测试：`mvn test`；CI 打包时测试被显式跳过
- Docker：`yudao-server/Dockerfile` + `script/docker-compose/`

## 7. 测试与工程质量

测试规模显著高于同类国内开源后台：622 个测试文件、约 16 万行测试代码（本地统计）。框架层 `yudao-spring-boot-starter-test` 提供三类基类——`BaseDbUnitTest`（H2 内存库 + SQL 初始化）、`BaseMockitoUnitTest`（纯 Mock）、`BaseRedisUnitTest`/`BaseDbAndRedisUnitTest`（嵌入式 Redis），配套 `RandomUtils` 随机数据、`AssertUtils` 断言（`.../starter-test/core/ut/`）。模块分布：bpm 46、system 42、mall 37、framework 36、infra 27 个测试文件，主链路模块覆盖率尚可，但相对 55 万行主代码体量，mes/hrm/iot 等新模块测试明显偏薄（各模块 `src/test` 统计）。

CI 仅一条 `.github/workflows/maven.yml`：解析根 pom 的 `java.version` 选择 JDK 后 `mvn -B package -Dmaven.test.skip=true`，作用是"打包校验"而非质量门禁，无 PR 触发（pull_request 被注释）、无 lint/覆盖率。代码风格遵循《阿里巴巴 Java 开发手册》（README 声明），实际源码注释密度高、Service 层方法均有 Javadoc，观感与声明一致。

## 8. 社区与维护状态

- 活跃度极高：GitHub 最近 push 2026-11-13（分析前一日），tarball 内文件 mtime 为 2026-11-06，均为数日内
- 发布节奏：月度发版且三 JDK 分支同步出 tag，如 v2026.10 对应 `jdk8/11`、`jdk17/21`、`jdk25` 三个 release（2026-10-31 发布），v2026.09 同理（GitHub releases API）
- 贡献者 284 人，但提交高度集中于作者本人（艿艿/YunaiV，上海，README 自述"早中晚在 top3 大厂认真搬砖，夜里为开源做贡献"）；forks 8672
- 双平台运营：GitHub（YunaiV/ruoyi-vue-pro）与 Gitee（zhijiantianya/ruoyi-vue-pro）互为镜像，文档站 doc.iocoder.cn、三个在线演示站、视频教程入口齐备（README「新手必读」）
- Issue 面貌：open_issues 仅 5（大量问题被引导至 Gitee/文档/微信群处理，未确认具体机制）
- 生态分支：精简版 yudao-boot-mini、微服务版 yudao-cloud、前端 6 个独立仓库、Spring-Boot-Labs 学习专栏，构成"芋道矩阵"（README「项目关系」）

## 9. 亮点与风险

亮点：
1. 框架层工程化是国内同类标杆：14 个 starter 把多租户、数据权限、操作日志、安全、MyBatis 封装成可单独取用的组件（`yudao-framework/`），多租户做到 DB/MQ/Job/Web/缓存全链路（5.2 节）
2. 业务模块矩阵最广：20 个模块、55 万行 Java，mall(4 子模块)/erp/mes/hrm/iot/ai/im 等一仓俱全，且均按统一 api/controller/service/dal 五层规范实现
3. 三 JDK 分支（8/17-21/25）+ 月度三连发，老项目能停在 JDK8，新项目可直接上 Spring Boot 4
4. 代码生成器 106 模板 × 7 种前端风格，且生成物包含单测模板（`codegen/` 目录）
5. MIT + README 明确承诺永不商业版，代码经核对无功能墙

风险/局限：
1. 前端不在本仓库：`yudao-ui/` 只是 README 指引，真实前端分散于 6 个独立仓库，仅拉取本仓库无法获得完整可运行系统
2. 业务模块建表 SQL 未随模块入库：`sql/mysql/ruoyi-vue-pro.sql` 仅含 system/infra/demo 48 张表，mall/mes 等 18 个业务模块的表结构依赖文档站与迁移文档（README 提供迁移入口），离线部署需额外加工
3. CI 是纯打包校验且跳过测试（`.github/workflows/maven.yml`），55 万行代码的质量主要依赖作者自律，新人 PR 的合入门槛与审查强度未确认
4. 全家桶体量带来认知负担：模块间 `api` 包 + 事件/MQ 联动较多，删模块需按迁移文档逐项处理；部分新模块（mes 1153 文件）演进快、测试薄，稳定性需自行验证

## 10. 总结评价

ruoyi-vue-pro 已经不是"若依的一个分叉"，而是以若依为起点的企业级开发平台事实标准之一：它把国内后台系统 80% 的共性需求（RBAC/多租户/工作流/支付/代码生成/监控）做成带测试的框架层，再以二十个业务模块横向扩张，配合月度发版与三 JDK 分支维护，工程活力在同类中首屈一指；代价是前端分裂到独立仓库、业务 SQL 依赖文档、体量庞大带来的学习与裁剪成本。适合两类人：要快速搭企业后台（尤其需要 SaaS 多租户或信创国产库适配）的团队直接取用；想学习 Spring Boot 工程化封装（starter 设计、MP 多租户拦截、RBAC 缓存化实现、代码生成器）的 Java 开发者精读 `yudao-framework` 与 `yudao-module-system`——这两处的代码密度与注释质量，本身就是一套不错的实战教材。
