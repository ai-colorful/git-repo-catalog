# xkcoding/spring-boot-demo 项目分析

> 星数 25000（快照 2026-10-24；gh api 2026-11-21 核实实际为 34035）｜主语言 Java｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：国内最知名的 Spring Boot 教学型 demo 集合仓之一，以 Maven 多模块形式按功能逐个演示 Spring Boot 集成主流组件（ORM/缓存/MQ/安全/定时任务/搜索等），作者沈扬凯（Yangkai.Shen，JustAuth 作者）。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | xkcoding/spring-boot-demo | GitHub |
| 星数 | 34035（2026-11-21 gh api；任务快照 25000） | gh api repos/xkcoding/spring-boot-demo |
| Fork | 10811 | 同上 |
| 语言 | Java（484 个 .java 文件，共 890 文件） | 源码统计 |
| 许可证 | MIT（LICENSE，Copyright (c) 2018 Yangkai.Shen） | src/LICENSE |
| Spring Boot 基线 | 2.1.0.RELEASE，Java 8 | src/pom.xml |
| 最近真实代码提交 | master 最后 commit 2022-09-02（"更新 README 感谢"） | gh api commits?per_page=5 |
| repo pushed_at | 2024-07-24（非 master 内容更新，见第 8 章） | gh api |
| Release/Tag | 无任何 release 与 tag | gh api releases/tags 均为空 |

## 2. 项目是什么，解决什么问题

spring-boot-demo 是一个"深入学习并实战 Spring Boot"的示例工程（src/README.md《项目简介》）。它把 Spring Boot 生态里开发者最常遇到的集成场景拆成 60 余个互相独立的 Maven 模块，每个模块只演示一个主题：从 helloworld、properties 这类入门项，到 mybatis-plus、Redis 缓存、RabbitMQ/Kafka、Spring Security RBAC 动态权限、xxl-job 分布式调度、sharding-jdbc 分库分表、Elasticsearch 7.x RestHighLevelClient 等企业级场景（src/pom.xml 第 10-73 行 modules 列表，src/README.md 集成清单）。

作者自述规划 66 个 demo、已完成 55 个（src/README.md 第 23 行、src/TODO.md "模块计划(已完成：55 / 66)"）。实际 master 快照有 62 个顶层 demo 目录；其中部分是骨架或半成品（demo-tio、demo-mq-rocketmq、demo-pay 只有 Application 启动类 + 空/极简 README；demo-rbac-shiro、demo-oauth、demo-urule、demo-uflo、demo-activiti 标注"待完成"）。完成度数字与目录现状之间有出入，属教学仓常见的"规划先行"状态。

项目的差异化在于"每模块配套详细 README + SQL 初始化脚本 + 可运行代码"的三件套组织：以 demo-rbac-security 为例，模块 README 给出功能清单（RBAC 模型、JWT + Redis 会话控制、在线人数统计、踢人）、环境要求、建库建表步骤、默认账号和请求示例，并引用模块内 security.sql（src/demo-rbac-security/README.md、src/demo-rbac-security/sql/security.sql）。这使它不是"代码堆"而是可对照操作的教材。

历史上提供两个版本分支：master 基于 Spring Boot 2.1.0.RELEASE（每模块 parent 指向根 pom），v-1.5.x 基于 1.5.8.RELEASE，后者已声明停止维护（src/README.md《分支介绍》）。2022 年作者开过 3.x 分支尝试升级 Spring Boot 3，最后一条 commit 为 2022-11-25"升级进行中"后停滞（gh api commits?sha=3.x）。

它涨星的时代背景：2018-2020 年 Spring Boot 2.x 在国内普及期，缺乏中文、结构化、可运行的"全家桶式"示例库，本项目恰好填补了这一空档；作者个人影响力（JustAuth 作者、博客 xkcoding.com）与 README 中的开源互推（JustAuth/Mica/SpringBlade/Pig，src/README.md 开源推荐）也放大了曝光。

## 3. 技术栈与依赖

- 语言/构建：Java 8，Maven 多模块（根 pom packaging=pom，`<modules>` 声明 62 个顶层模块；src/pom.xml）。
- 版本基线：`spring.boot.version=2.1.0.RELEASE`、`mysql.version=8.0.21`、`hutool.version=5.4.5`、`guava.version=29.0-jre`、`user.agent.version=1.20`（src/pom.xml properties 段）。
- 依赖管理方式：根 pom 用 `spring-boot-dependencies` BOM import 统一版本（dependencyManagement，src/pom.xml 第 106-139 行），各子模块只写坐标不写版本。
- 涉及的第三方组件（按模块）：MyBatis/MyBatis-Plus/通用 Mapper/PageHelper/BeetlSQL/JPA、Redis/Ehcache、RabbitMQ/Kafka/RocketMQ、Quartz/xxl-job、Spring Security/Shiro/JustAuth/LDAP、Elasticsearch（含 7.x RestHighLevelClient）、MongoDB/Neo4j、sharding-jdbc、Zookeeper、Dubbo、netty-socketio、ureport2/uflo/urule/activiti、Flyway、GrayLog 等（各模块 pom 与 README）。
- 代码辅助：Lombok（@Slf4j 等大量使用，如 src/demo-cache-redis/.../UserServiceTest.java）。
- 仓库镜像配置：根 pom 仅配阿里云 Maven 仓库（src/pom.xml repositories 段），面向国内网络环境。

## 4. 目录结构与代码组织

顶层为"根 pom + 62 个平铺的 demo-* 模块目录"（tarball 解压后 890 文件/484 java/92 test java/45 sql/58 个模块 README）：

| 目录（按分类） | 职责 | 数量 |
|---|---|---|
| demo-helloworld/properties/actuator/exception-handler/async/war/https/docker | 基础与工程化（启动、配置、监控端点、统一异常、异步、打包、HTTPS、容器化） | 8 |
| demo-admin/logback/log-aop/graylog | 监控与日志（admin 含 admin-client/admin-server 两个子模块） | 4 |
| demo-template-freemarker/thymeleaf/beetl/enjoy | 模板引擎四件套 | 4 |
| demo-orm-jdbctemplate/jpa/mybatis/mybatis-mapper-page/mybatis-plus/beetlsql | ORM/数据访问六种方案横评 | 6 |
| demo-multi-datasource-jpa/-mybatis、demo-dynamic-datasource、demo-sharding-jdbc、demo-flyway | 多数据源/动态数据源/分库分表/迁移 | 5 |
| demo-cache-redis/ehcache | 缓存 | 2 |
| demo-task/-quartz/-xxl-job | 定时任务三层次（原生/单机动态管理/分布式调度） | 3 |
| demo-mq-rabbitmq/-kafka/-rocketmq | 消息队列（rocketmq 仅骨架） | 3 |
| demo-rbac-security/-shiro、demo-session/-oauth/-social/-ldap | 安全/权限/会话/第三方登录（security 为最完整模块，40+ 类） | 6 |
| demo-swagger/-swagger-beauty | API 文档 | 2 |
| demo-websocket/-socketio、demo-tio | WebSocket/网络编程（tio 仅骨架） | 3 |
| demo-elasticsearch/-rest-high-level-client | 搜索（两代 API 对照） | 2 |
| demo-mongodb/neo4j | NoSQL/图数据库 | 2 |
| demo-ratelimit-guava/-redis | 单机/分布式限流（Guava RateLimiter vs Redis+Lua） | 2 |
| demo-upload/email/dubbo/zookeeper/ureport2/urule/uflo/activiti/codegen/pay | 文件上传、邮件、RPC、分布式锁、报表/规则/流程引擎、代码生成、支付（urule/uflo/activiti 标待完成，pay 为骨架） | 10 |

组织方式评述：这是"parent-pom 聚合 + 平铺模块"的典型教学仓结构。根 pom 只做版本管理与插件管理（spring-boot-maven-plugin repackage，src/pom.xml build 段），模块间零依赖、可单独构建；demo-admin、demo-dubbo、demo-oauth 三个目录内部又各含 2-3 个子模块（client/server、common/provider/consumer、authorization-server/resource-server），实际 Maven reactor 约 69 个模块。缺点是 62 个模块必须逐个写进根 pom（第 10-73 行），新增模块要手工登记；优点是结构一目了然，对初学者零门槛。

## 5. 核心架构与关键实现

教学仓无"业务架构"，核心价值在代表性模块的实现质量。抽样 4 个：

1. demo-rbac-security（最完整的模块，40+ 类，分层 common/config/controller/exception/model/payload/repository/service/util，src/demo-rbac-security/src/main/java/com/xkcoding/rbac/security/ 下）。设计要点：登录/登出走自定义 AuthController 而非 Security 默认端点以适配前后端分离；动态权限由 RbacAuthorityService 从数据库读取权限-URL 映射；JWT 无法主动过期的问题用 Redis 补齐——JwtUtil 配合 Redis 保存会话实现"单设备登录 + 在线人数统计 + 手动踢人"（模块 README 功能清单；src/.../config/JwtAuthenticationFilter.java、JwtUtil.java、MonitorService.java）。
2. demo-cache-redis：标准三层（config/RedisConfig 定制序列化，service/impl 用 @Cacheable/@CacheEvict 注解缓存，test/UserServiceTest 用两次查询日志对比验证缓存命中，src/demo-cache-redis/src/）。测试注释直接解释验证点（"查看日志，只打印一次日志，证明缓存生效"），教学意图清晰。
3. demo-mq-rabbitmq：按队列模式组织——Direct/Topic/Fanout（分列）/延迟队列各一个 Handler，常量集中 RabbitConsts，消息体统一 MessageStruct（src/demo-mq-rabbitmq/src/main/java/com/xkcoding/mq/rabbitmq/{config,constants,handler,message}/）。
4. demo-elasticsearch-rest-high-level-client：展示 ES 7.x 官方客户端替代 spring-data-es 的写法（src/demo-elasticsearch-rest-high-level-client/，README 对比两代集成方式）。

通用代码风格：所有类带 javadoc 头（`<p>说明</p>` + @author yangkai.shen + @date Created in ...），方法级注释用中文说明参数与用途（src/demo-cache-redis/.../UserService.java 全文即典型样本）；.editorconfig 统一 Java 缩进 4 空格、其余 2 空格（src/.editorconfig）。注释质量整体高于一般 demo 仓，且是中文，贴合目标读者。

## 6. 构建与运行

来自 src/README.md《运行方式》与 CI 配置：

```bash
git clone https://github.com/xkcoding/spring-boot-demo.git
# IDEA 导入根目录 pom.xml（需安装 lombok 插件），找到各模块 *Application 类运行
mvn clean package -DskipTests=true -Dmaven.javadoc.skip=true -B -V   # CI 实际命令（.github/workflows/maven.yml）
```

- 环境要求：JDK 1.8+、Maven 3.5+、MySQL 5.7+（src/README.md 开发环境）。
- 多数模块需先执行模块内 SQL 初始化数据库（全仓 45 个 .sql 文件，如 src/demo-rbac-security/sql/security.sql、src/demo-flyway/src/main/resources/db/migration/V1_0__INIT.sql），Redis/MQ/ES 等需本地服务。
- 单模块构建示例：`mvn package -pl demo-cache-redis -am`（由根 pom 聚合关系推出，项目文档未单独给出，此条为推断）。

## 7. 测试与工程质量

- 测试规模：92 个 test java 文件，几乎每模块一个 SpringBootTest 启动类 + 少量功能测试（src/demo-cache-redis/src/test/.../UserServiceTest.java 等）。风格为 JUnit 4（org.junit.Test）+ 集成式验证：依赖真实 MySQL/Redis 等服务，通过日志输出人工核对，断言少。
- CI 实际是"编译冒烟"：GitHub Actions（src/.github/workflows/maven.yml）push/PR 触发，JDK 1.8，`mvn clean package -DskipTests=true`；另有遗留 .travis.yml 同样 skipTests。测试在 CI 中被跳过，与"集成测试需外部服务"的现实一致；gh api actions/runs 返回空，说明 workflow 实际未产生运行记录（或早已失效）。
- 代码质量工具链：Codacy（src/.codacy.yml 排除 md/sql/test）、EditorConfig、Travis badge（src/README.md 头部）。
- 工程观察：模块模板高度一致（pom + README + Application + config + test），复制粘贴式增量的好处是整齐，代价是每个模块各自维护一套 common/util（如 rbac-security 与 rbac-shiro 各有一套 R/ResultCode），无共享公共模块。

## 8. 社区与维护状态

- 贡献者结构：11 名贡献者，xkcoding 本人 717 次提交，其余 10 人合计约 16 次（fxbin 4、lizhongyue248 3、chenqi146 2、其余 1 次；gh api contributors）。典型的"作者主导 + 偶发 PR"模式，ldap 模块即社区贡献（README 致谢 @fxbin，src/README.md 第 138 行）。
- 贡献流程：有 PR 模板（要求先开 issue、PR 提到 dev 分支、模仿既有编码风格，src/.github/PULL_REQUEST_TEMPLATE.md）与 issue 模板（bug_report/feature_request，src/.github/ISSUE_TEMPLATE/），但 dev 分支最后提交停留在 2021-04-10，流程形同虚设。
- 活跃度时间线（gh api 核实）：master 最后 commit 2022-09-02；3.x 分支（Spring Boot 3 升级）2022-11-25 后停滞；6 个 dependabot 依赖升级分支最后活动 2023-06-14（Bump guava 32.0.0-jre），均未合并；仓库 pushed_at 2024-07-24 来自非 master 的分支推送/元数据变更，master 内容自 2022-09 起冻结。未确认项：2024-07 的具体推送分支无法从公开 API 定位。
- 无 release、无 tag、无 CHANGELOG；README 保留"团队纳新"招聘广告（src/jd.md，涂鸦智能 Tuya 数据平台岗位，2021 年语境）。
- 生态位置（gh api 元数据，2026-11-21）：同类中文 Spring Boot 示例仓 ityouknow/spring-boot-examples 30422 星（最后 push 2023-11）、dyc87112/SpringBoot-Learning 15679 星（2023-12）；本项目 34035 星居首，但三者共同特征是"内容在 Spring Boot 2.x 时代成型后陆续停更"，属于存档型的教学资源而非活跃工程。

## 9. 亮点与风险

亮点：
1. 模块矩阵完整度在同类中最高：62 个主题横跨 ORM/缓存/MQ/安全/调度/搜索/限流/报表，且同一主题给多方案对照（4 种模板引擎、6 种 ORM、两代 ES API、单机 vs 分布式限流），适合横向选型学习（src/pom.xml modules + 各模块 README）。
2. "README + SQL + 可运行代码"三件套质量高：模块 README 有功能清单、运行步骤、默认账号、关键代码索引（src/demo-rbac-security/README.md），45 个 SQL 脚本覆盖建表与初始化（源码统计）。
3. 中文注释与 javadoc 规范统一，代码可直接当风格范本（src/demo-cache-redis/.../UserService.java、.editorconfig）。
4. 版本管理干净：根 pom BOM 统一版本，子模块免写版本号（src/pom.xml dependencyManagement）。

风险/局限：
1. 技术栈严重过时：Spring Boot 2.1.0（2018 年版）+ Java 8，与当前 3.x/Java 17+ 生态脱节；3.x 升级分支已停滞 3 年半（gh api commits?sha=3.x）。
2. 项目实质停更：master 自 2022-09 冻结，issue 积压 144 个，无 release/tag； dependabot 安全升级（guava、commons-text、mysql-connector、shiro）长期未合并，照抄依赖存在已知 CVE 风险（dependabot 分支列表）。
3. 完成度虚标：README 称 55/66 完成，但 demo-pay、demo-tio、demo-mq-rocketmq 等仅是骨架，rbac-shiro/oauth/urule/uflo/activiti 未完成（目录内容核实）。
4. 测试为弱验证：92 个测试文件基本靠日志人工核对，CI 又 skipTests，回归保障为零（.github/workflows/maven.yml）。

## 10. 总结评价

spring-boot-demo 是 2018-2022 年间中文 Spring Boot 学习资源的标杆：以 62 个平铺 Maven 模块 + 每模块三件套（详细 README、SQL 脚本、带规范中文注释的可运行代码）把 Spring Boot 2.x 时代的主流集成场景做成了系统性教材，34000+ 星的体量与"作者主导（单人 98% 提交）"的组织方式并存；2022 年后作者停止维护，Spring Boot 3 升级半途而废，如今它更像一份高质量的历史存档而非可以照抄进新工程的依赖源。适合两类人：一是仍在维护 Spring Boot 2.x 老系统的开发者按模块查"怎么接 Redis/MQ/权限"，二是初学者用来理解"一个 Spring 模块的最小完整形态"（配置类 + service + 测试 + 文档该怎么组织）；学习时必须自行将 API 迁移到 Spring Boot 3.x，并注意仓库内依赖版本已不安全，不应原样复制 pom 依赖。
