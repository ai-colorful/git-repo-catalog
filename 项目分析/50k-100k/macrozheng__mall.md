# macrozheng/mall 项目分析

> 星数 84950（快照 2026-10-24）｜主语言 Java｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：一套基于 SpringBoot + MyBatis 的完整电商系统（前台商城 + 后台管理），中文社区知名度最高的 Java 电商实战项目。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | macrozheng/mall | gh api |
| 星数 | 84950（快照 2026-10-24，gh api 实测 84990） | 任务参数 / gh api |
| 语言 | Java（SpringBoot 3.5.14 + JDK 17） | `pom.xml` |
| 许可证 | Apache-2.0 | `LICENSE` / gh api |
| 创建时间 | 2018-04-04 | gh api |
| 最近 push | 2026-09-15（master 分支） | gh api |
| 默认分支 | master（SpringBoot 3.5 + JDK17）；dev-v2 分支为 SpringBoot 2.7 + JDK8 | `README.md` |
| 规模 | 约 524 个 Java 文件，76 张数据库表 | 源码统计 |

## 2. 项目是什么，解决什么问题

mall 项目致力于打造一个完整的电商系统（`README.md`「前言」）。它覆盖电商业务的双端：前台商城系统包含首页门户、商品推荐、商品搜索、商品展示、购物车、订单流程、会员中心等模块；后台管理系统包含商品、订单、会员、促销、运营、内容、统计、权限管理等模块（`README.md`「项目介绍」）。

与"演示级 CRUD 项目"不同，mall 的核心价值在于把电商领域的关键工程问题做成了可学习的范例：库存锁定与超时取消（`mall-portal/.../OmsPortalOrderServiceImpl.java`）、促销规则计算（满减/阶梯/会员价，见 `pms_product_ladder`、`pms_product_full_reduction` 等表）、动态权限（RBAC + 接口级动态授权，`mall-security`）、以及搜索/缓存/消息/对象存储等中间件的真实集成。

数据库层按业务域划分清晰，`document/sql/mall.sql`（3257 行、76 张表）以表前缀划分五大域：pms（商品）、oms（订单）、sms（营销）、ums（用户）、cms（内容），是理解整个系统业务模型的最好入口。

项目定位是"电商实战 + 学习教程"的配套代码：README 大量链接到作者站点 macrozheng.com 的图文/视频教程，配套生态包括前端项目 mall-admin-web、mall-app-web、微服务版 mall-swarm 和教程仓库 mall-learning（`README.md` 徽章区）。这解释了其星数的主要来源——它是中国 Java 学习者群体中事实上的"SpringBoot 电商标准练手项目"。

master 分支已完成向 SpringBoot 3.5 + JDK 17 的升级（根 `pom.xml` parent 3.5.14），旧技术栈维护在 dev-v2 分支（`README.md`「分支说明」）。

## 3. 技术栈与依赖

后端（出处：根 `pom.xml`、`README.md` 技术选型表）：

- 框架：SpringBoot 3.5.14（parent）、Spring Security + JWT（jjwt）、MyBatis 3.5.19 + PageHelper 6.1.1（物理分页）、SpringDoc OpenAPI 2.8.17（接口文档）
- 数据层：MySQL（connector 9.3.0）、Druid 1.2.24 连接池、Redis 7（自封装 RedisService）、MongoDB 5.0（会员浏览/收藏历史）
- 中间件：RabbitMQ 3.10（延迟队列取消订单）、Elasticsearch 7.17.3（商品搜索，Spring Data Elasticsearch）、Logstash + Kibana（ELK 日志收集）
- 存储/支付：MinIO 8.6 + 阿里云 OSS（文件上传）、支付宝 SDK 4.40（支付，`mall-portal/.../AlipayServiceImpl.java`）
- 工具：Hutool 5.8.40、Lombok、Bouncy Castle
- 构建部署：Maven 多模块 + fabric8 docker-maven-plugin 0.45.1（打包即构建镜像，`pom.xml` build 段）

前端（独立仓库 mall-admin-web / mall-app-web，`README.md`）：Vue2 + Element-UI（管理端）、uni-app（移动端商城）。本仓库不含前端代码。

## 4. 目录结构与代码组织

Maven 多模块单体（7 个 module，出处：根 `pom.xml`）：

| 目录 | 职责 | 规模（Java 文件数） |
| --- | --- | --- |
| `mall-common/` | 通用工具：CommonResult/CommonPage 统一响应、全局异常、WebLog 切面、RedisService 抽象 | 14 |
| `mall-mbg/` | MyBatisGenerator 生成的实体/Mapper（对应 76 张表）+ `Generator.java` 代码生成入口 | 230 |
| `mall-security/` | SpringSecurity 通用封装：JWT 过滤器、动态权限管理器，可被各应用复用 | 14 |
| `mall-admin/` | 后台管理系统接口（8080 端口），31 个 Controller 按域划分 | 154 |
| `mall-search/` | 商品搜索系统（8081），MySQL→ES 同步与搜索 API | 12 |
| `mall-portal/` | 前台商城接口（8085），购物车/订单/会员/支付 | 86 |
| `mall-demo/` | 框架搭建时的测试演示代码 | 14 |
| `document/` | SQL 脚本、docker-compose、ELK 配置、Postman 集合、部署文档、架构图 | — |

组织方式评述：典型的"共享库 + 多应用"结构——mall-common/mall-mbg/mall-security 是被依赖的库模块，mall-admin/mall-search/mall-portal 是三个可独立打包部署的 SpringBoot 应用（`document/docker/docker-compose-app.yml` 正是按这三个应用编排）。业务分层沿用 Controller → Service → DAO，DAO 层分 MBG 生成的 Mapper 与手写 Dao（复杂联表查询，如 `mall-admin/src/main/resources/dao/OmsOrderDao.xml`）。代码整体风格统一、中文注释详尽，明显为教学可读性优化。

## 5. 核心架构与关键实现

### 5.1 订单流程：库存锁定 + 优惠券/积分分摊（mall-portal）

下单核心逻辑在 `mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java`（775 行）的 `generateOrder`：校验收货地址 → 取购物车并计算促销（`cartItemService.listPromotion`）→ 校验库存 → 优惠券金额按商品分摊（`handleCouponAmount`）→ 积分抵扣按金额比例分摊（`productPrice.divide(totalAmount, 3, RoundingMode.HALF_EVEN).multiply(integrationAmount)`）→ `handleRealAmount` 计算实付 → `lockStock` 锁定 SKU 库存 → 生成订单与订单项。金额全程使用 BigDecimal 并显式指定舍入模式，是电商金额处理的规范写法。

### 5.2 RabbitMQ TTL + 死信队列实现订单超时自动取消（mall-portal）

不引入延迟插件，用 RabbitMQ 原生 TTL + 死信交换机实现延迟消息。`mall-portal/src/main/java/com/macro/mall/portal/domain/QueueEnum.java` 定义两条队列（mall.order.cancel 与 mall.order.cancel.ttl）；`mall-portal/.../config/RabbitMqConfig.java` 中 TTL 队列声明了死信转发参数：

```java
return QueueBuilder
        .durable(QueueEnum.QUEUE_TTL_ORDER_CANCEL.getName())
        .withArgument("x-dead-letter-exchange", QueueEnum.QUEUE_ORDER_CANCEL.getExchange())
        .withArgument("x-dead-letter-routing-key", QueueEnum.QUEUE_ORDER_CANCEL.getRouteKey())
        .build();
```

下单成功后 `CancelOrderSender.sendMessage(orderId, delayTimes)` 发送带 expiration 的消息到 TTL 队列，到期后死信转发到 `mall.order.cancel`，由 `CancelOrderReceiver`（`@RabbitListener(queues = "mall.order.cancel")`）消费并执行 `cancelOrder` 释放库存。另有定时任务 `OrderTimeOutCancelTask` 兜底扫描超时未支付订单。

### 5.3 可复用的动态权限安全模块（mall-security）

`mall-security` 把认证与授权抽象成可插拔组件：`JwtAuthenticationTokenFilter` 解析 token 注入认证上下文；`DynamicAuthorizationManager`（实现 Spring Security 6 的 `AuthorizationManager<RequestAuthorizationContext>`）配合 `DynamicSecurityMetadataSource` 按 URL 动态取所需权限，权限数据由使用方实现 `DynamicSecurityService` 提供（mall-admin 用 ums_resource/ums_menu 表，`mall-admin` 模块实现）。应用只需在 `application.yml` 配 `secure.ignored.urls` 白名单和 `jwt.*` 参数（`mall-portal/src/main/resources/application.yml`）即可接入。

### 5.4 商品搜索与数据同步（mall-search）

`mall-search/src/main/java/com/macro/mall/search/` 仅 12 个类：`EsProduct`（@Document 映射）、`EsProductRepository`（Spring Data Elasticsearch，含按名称/副标题/品牌/类目检索的方法命名查询）、`EsProductServiceImpl` 负责从 MySQL 批量导入商品到 ES 及简单/复合搜索。设计与主应用解耦，独立成 8081 服务。

## 6. 构建与运行

构建（标准 Maven，根 `pom.xml`，默认 `skipTests=true`）：

```bash
mvn package                     # 打包全部模块
mvn package -pl mall-admin -am  # 只构建 mall-admin 及依赖
```

运行：三个应用均可直接 `java -jar` 启动（依赖 MySQL/Redis，`README.md`「搭建步骤」指出只启动 mall-admin 仅需 MySQL + Redis）；配置按 dev/prod profile 区分（`application-dev.yml`/`application-prod.yml`）。

数据库初始化：导入 `document/sql/mall.sql`（含表结构 + 演示数据）。

Docker Compose 部署（出处：`document/docker/`）：

```bash
docker-compose -f docker-compose-env.yml up -d   # mysql/redis/nginx/rabbitmq/es/logstash/kibana/mongo/minio 共 9 个中间件
docker-compose -f docker-compose-app.yml up -d   # mall-admin(8080)/mall-search(8081)/mall-portal(8085)
```

镜像构建：fabric8 docker-maven-plugin 在 package 阶段构建 `mall/<module>:1.0-SNAPSHOT` 镜像（docker.host 指向远程 Docker），另有手写 `document/sh/Dockerfile`（openjdk:17 基础镜像）。Jenkins 自动化部署流程见 `README.md` 部署章节链接。

## 7. 测试与工程质量

- 测试极少：全仓库仅 5 个测试文件（`mall-admin/src/test/.../PmsDaoTests.java`、`mall-portal/src/test/.../PortalProductDaoTests.java` 等），且根 `pom.xml` 显式 `<skipTests>true</skipTests>`。这是教学项目的典型取舍——测试让位于可读性。
- 无 CI：仓库不含 `.github/` 目录，无 GitHub Actions / issue 模板（源码目录确认）。
- 质量观察：依赖版本新且维护到位（SpringBoot 3.5.14、MySQL connector 9.3.0、Druid 1.2.24 均为近期版本）；统一响应体/全局异常/日志切面（`mall-common`）使接口风格一致；MBG 生成代码与手写代码严格分离在 mall-mbg 与各应用的 dao 目录。弱点：单体 Service 类偏大（订单实现 775 行）、配置中明文密钥（JWT secret、Druid/MQ 密码，教学项目常见）。

## 8. 社区与维护状态

- 活跃度：gh api 显示最近 push 为 2026-09-15，项目持续维护 8 年以上（2018-04 创建）；本 tarball 快照无 git 历史，具体提交节奏"未确认"。
- 版本：未发现 CHANGELOG；技术演进从 README 可见一斑——master 已升级 SpringBoot 3.5 + JDK17，SpringBoot 2.7 + JDK8 移至 dev-v2 分支（`README.md`「分支说明」）。
- 社区结构：17 个 open issues（gh api，作者倾向用公众号/微信群收集反馈，`README.md`「公众号」）；fork 29839，配套仓库 5 个（mall-swarm、mall-admin-web、mall-app-web、mall-learning、mall-learning-tech 教程体系），是典型的"个人作者 + 教学生态"项目。贡献者列表"未确认"（tarball 无历史）。

## 9. 亮点与风险

亮点：

1. 业务完整度高：76 张表覆盖商品/订单/营销/用户/内容五大域（`document/sql/mall.sql`），是少见的前后台闭环电商实现。
2. 关键工程问题均有可读的实现范本：RabbitMQ 死信延迟队列取消订单（`mall-portal/.../RabbitMqConfig.java`）、促销金额分摊（`OmsPortalOrderServiceImpl.java`）、URL 级动态权限（`mall-security`）。
3. 模块边界清晰：common/mbg/security 三个库模块 + 三个可独立部署的应用，微服务化路径（mall-swarm）已铺好。
4. 部署链路完整：docker-compose 一键拉起 9 个中间件 + 3 个应用，镜像由 Maven 插件在打包时构建。
5. 文档/教程生态强大：架构图、功能结构图（`document/resource/`、`document/mind/`）、Postman 集合（`document/postman/`）齐全，学习曲线友好。

风险/局限：

1. 几乎无测试与 CI，重构与升级缺乏回归保障，直接用于生产需自建测试体系。
2. 教学导向的配置简化（明文密码、默认密钥、docker 2375 端口暴露在 `pom.xml` docker.host），生产使用必须整改安全配置。
3. 部分技术选型偏旧（MySQL 5.7、mongo 4、Vue2 前端在独立仓库），新项目需自行评估替代。
4. 单应用 Service 层偏重（如订单实现 775 行），复杂促销逻辑集中，二次开发需注意拆分。

## 10. 总结评价

mall 是中文 Java 社区最具影响力的电商实战项目：它以完整闭环的业务模型、清晰的多模块组织和覆盖缓存/消息/搜索/安全的真实中间件集成，把"电商系统怎么写"拆成了可逐模块学习的课程，长期维护与 SpringBoot 3.5 的及时升级体现了作者持续的投入。它不是开箱即用的生产系统（无测试、无 CI、配置简化），而是教科书级的参考实现。适合 Java 后端学习者用它理解电商业务全貌与中间件落地方式，适合开发者做二次开发的业务底板参考，也适合想研究"单体如何演进到微服务"的读者对照 mall-swarm 阅读。
