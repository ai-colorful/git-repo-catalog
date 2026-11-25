# keycloak/keycloak 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Java｜档位 20k-50k｜分析日期 2026-11-16

## 1. 项目速览

一句话定位：开源身份认证与访问管理（IAM）平台，为现代应用提供统一的登录、SSO、身份联邦与细粒度授权，是 IAM 领域的事实标准（Red Hat 主导，CNCF 孵化项目）。

| 项目 | 内容 |
|---|---|
| 名称 | keycloak/keycloak |
| 星数 | 39500（快照），gh api 实测 37892（2026-11-16） |
| 语言 | Java（服务端）+ TypeScript（前端控制台） |
| 许可证 | Apache-2.0（gh api 核实） |
| 分支/最近活跃 | main，最近 push 2026-11-16（当日仍活跃） |
| Forks / Open Issues | 9198 / 3579（gh api） |
| 贡献者 | 约 320 人（contributors 分页末页=320，per_page=1 推算） |
| 最新版本 | 26.8.2（2026-11-05 发布） |

## 2. 项目是什么，解决什么问题

应用开发中最繁琐的部分从来不是业务逻辑，而是"用户是谁、怎么登录、权限如何管"。Keycloak 把这些完全收编为平台能力：应用只需对接标准协议（OIDC/OAuth2/SAML），由 Keycloak 托管登录页、注册、密码找回、双因素认证、社交账号绑定等全部身份流程。README（`README.md`）概括为 user federation、strong authentication、user management、fine-grained authorization 四大能力。

它解决的核心痛点有三层。第一是**协议标准化**：内置 OIDC、OAuth2、SAML 完整协议栈（`services/src/main/java/org/keycloak/protocol/`），还有 PAR、RAR、设备授权、Client Initiated Backchannel Login 等扩展规范，另有 OID4VC（可验证凭证）这类前沿标准的实现。第二是**身份联邦**：企业存量用户在 LDAP/AD/Kerberos 里，外部用户在社交平台，Keycloak 通过 `federation/`（ldap、kerberos、sssd、ipatuura）与 `services/src/main/java/org/keycloak/social/`、`broker/`（身份代理）把异构身份源统一映射为本地会话。第三是**多租户治理**：以 Realm 为隔离单位，每个 Realm 独立的用户池、客户端、认证流、主题与密钥，一台实例可同时服务几十个业务域。

作为服务端，它以 Quarkus 扩展形式打包运行（`quarkus/README.md`："Keycloak on Quarkus is basically a Quarkus Extension"），即 `bin/kc.sh start-dev` 即起；也能作为依赖嵌入任意 Quarkus 应用。管理面由 React 编写的 Admin Console 与面向最终用户的 Account Console 构成（`js/apps/`）。

生态位上，Keycloak 与商业 SaaS 的 Auth0/Okta 同题对解：功能矩阵最接近 Okta，但可自托管、零许可费；与自建方案（Spring Authorization Server、Ory Hydra/Kratos）相比，它是"完整产品"而非"组件库"——开箱即含认证 UI、用户管理后台与运维（Operator/K8s 支持），代价是体量大、模型约束强。CNCF 孵化身份、月度级发版、活跃量在全球开源 IAM 中居首。

## 3. 技术栈与依赖

| 层 | 技术 | 出处 |
|---|---|---|
| 语言/运行时 | Java 17 编译目标，支持 JDK 21/25 运行 | `pom.xml`、`docs/building.md` |
| 服务端框架 | Quarkus 3.40.1.2（自研 Quarkus Extension） | `pom.xml`、`quarkus/` |
| 会话缓存/集群 | Infinispan 16.0.15（分布式 session、离线 token 撤销） | `pom.xml`、`model/infinispan/` |
| 持久化 | JPA/Hibernate（主存储），UserStorageProvider SPI 允许外挂自定义存储 | `model/jpa/`、`model/storage/` |
| 协议/安全 | 自研 saml-core、JOSE（jose/）、BouncyCastle 系 crypto 模块 | `saml-core/`、`crypto/` |
| 认证 UI | FreeMarker 模板 98 个 .ftl（登录页可换主题 base/keycloak/keycloak.v3） | `themes/` |
| 前端 | React 18.3 + Vite 7 + react-router-dom 7 + PatternFly，pnpm workspace | `js/apps/admin-ui/package.json` |
| 构建/交付 | Maven（36 个顶层 module）+ mvnw + Galleon feature pack 打包 + K8s Operator | `pom.xml`、`distribution/`、`operator/` |

## 4. 目录结构与代码组织

顶层是按"SPI 能力域"切分的多模块 Maven 单体：

| 目录 | 职责 | 规模 |
|---|---|---|
| `services/` | 服务端业务全集：认证流、协议端点、身份代理、授权服务、事件、导出导入 | 1634 个 Java 文件 |
| `server-spi/` `server-spi-private/` | Provider/ProviderFactory/Spi 等扩展契约（公共+私有两层） | — |
| `model/` | 存储层：jpa（主持久化）、infinispan（会话缓存）、storage（用户存储 SPI） | 924 个 Java 文件 |
| `quarkus/` | Quarkus 扩展与发行包：deployment/runtime/dist/config-api/tests | — |
| `js/` | pnpm monorepo：admin-ui（634 个 ts/tsx）、account-ui、keycloak-admin-client | — |
| `federation/` | LDAP、Kerberos、SSSD、iPATUURA 联邦实现 | 144 个 Java 文件 |
| `protocol/` 相关 | oidc/saml/oauth2/oid4vc/docker 五类协议实现（位于 services 内） | — |
| `themes/` | 登录/邮件 FreeMarker 主题 | 98 个 .ftl |
| `tests/` + `testsuite/` | 新一代测试框架（1082 文件）+ 遗留 Arquillian 套件（1068 文件，DEPRECATED.md 标记） | — |
| `authz/` `authzen/` `ssf/` `scim/` | 细粒度授权、AuthZEN 标准、共享信号框架（SSF）、SCIM | — |
| `adapters/` `operator/` `distribution/` | 应用适配器（旧）、K8s Operator、发行打包 | — |

组织方式评述：典型的"核心引擎 + 全 SPI 插件化"架构，`server-spi` 定义契约、`services` 提供默认实现、`quarkus` 负责打包成可启动的发行版；前端 `js/` 是独立 pnpm workspace，与 Java 通过 Maven 模块桥接（`js/pom.xml`）。模块间依赖单向向下，业务能力几乎全部可被自定义 provider 替换。

## 5. 核心架构与关键实现

### 5.1 SPI 扩展体系（一切皆 Provider）

Keycloak 的可扩展性建立在双层契约上：`Spi` 描述一种能力（名字、Provider 类、Factory 类），`ProviderFactory` 管理生命周期（`server-spi/src/main/java/org/keycloak/provider/ProviderFactory.java`——boot 时发现所有 factory，逐个 `init()`、全部就绪后 `postInit()`、关机 `close()`）。装配过程在 `services/src/main/java/org/keycloak/services/DefaultKeycloakSessionFactory.java`（`loadFactories()`，第 287 行起）：遍历每个 Spi，按配置 `Config.getProvider(spi.getName())` 选定实现并初始化。

发现机制由 `services/src/main/java/org/keycloak/provider/ProviderManager.java` 完成，核心是 Java ServiceLoader：

```java
for (ProviderLoaderFactory f : ServiceLoader.load(ProviderLoaderFactory.class,
        getClass().getClassLoader())) {
    ...
}
```

配合 `META-INF/services` 描述符（如 `authzen/services/src/main/resources/META-INF/`），内置 provider 与用户部署的第三方 JAR（drop-in 部署）走同一套加载路径。`Config.getProvider` 还允许按 Realm 维度覆写默认实现——这就是"自定义认证器、自定义身份代理、自定义用户存储"都只需一个 Factory 类的原因。SDK 层面另提供 `ProviderConfigurationBuilder`（`server-spi/`）简化配置声明。

### 5.2 认证流引擎（services/authentication）

认证被建模为"可编排的流"：Login 表单、OTP、X509、社交跳转、条件判断都是 Authenticator，串成 flow 后由引擎执行。核心引擎 `services/src/main/java/org/keycloak/authentication/AuthenticationProcessor.java`（1334 行）持有 `AuthenticationSession` 状态机，通过 `authenticationFlow.processFlow()` 推进（第 970、1123 行两处入口分别对应浏览器流与 required action 流），默认执行器是 `DefaultAuthenticationFlow.java`，认证器选择由 `AuthenticationSelectionResolver.java` 完成。浏览器端的验证邮件/重置密码等无会话动作用 action-token 机制实现（`authentication/actiontoken/` 下 resetcred、verifyemail、inviteorg 等独立包）。OIDC 侧的登录入口 `OIDCLoginProtocolService` 把授权请求转交该引擎，token 签发逻辑集中在 `TokenManager.java`（1798 行，含 token 构造、刷新、撤销），token 端点在 `endpoints/TokenEndpoint.java`（334 行，密码/客户端凭证/刷新等 grant 分发）。

### 5.3 运行时演进：WildFly → Quarkus

仓库中 WildFly 已基本无痕迹：服务端运行时代码全部位于 `quarkus/`（deployment 构建期字节码生成 + runtime 运行时 + dist 打包），`quarkus/README.md` 明确其为 Quarkus Extension 并可嵌入应用。`pom.xml` 仅保留 `upstream.wildfly.version`（第 61 行）用于旧版 WildFly/EAP 应用适配器的兼容构建，WildFly 字样只剩历史文档（`docs/tests-oidc-conformance.md` 的旧部署记录）。这是 2022 年 Keycloak 17 起的重大迁移：启动从数十秒降到秒级、内存占用大减，也把 Dev Mode、AOT 配置优化等 Quarkus 生态能力带入。会话与认证态的分布式缓存则由 `model/infinispan/` 提供（`InfinispanAuthenticationSessionProvider`、`ImmutableSession`、`ClientSessionManager` 等），支持跨节点 SSO 与离线 token。

### 5.4 身份联邦与 Realm 隔离

联邦层 `federation/ldap/`（`LDAPStorageProvider.java`、`LDAPIdentityStoreRegistry.java`、只读/只写 delegate 包装类）把 LDAP/AD 用户按需同步进本地模型，写入通过 delegate 控制可回写性；`federation/kerberos/` 提供 SPNEGO 免密登录。外部 IdP 由 `services/.../broker/` 统一处理（oidc、saml、oauth、oid4vp、kubernetes、spiffe、trust 等），社交登录在 `services/social/`。所有数据以 Realm 为硬隔离边界（realm 名贯穿 `model/jpa` 表结构与 admin API 路径），组织维度的多租户新特性在 `services/organization/`。值得注意的是存储层的取舍：旧"Map Storage"实验性路线已从 `model/` 移除（现仅 jpa、infinispan、storage 三线），社区最终回归 JPA 为主、Infinispan 缓存为辅的稳健组合。

## 6. 构建与运行

来自 `docs/building.md`、`docs/tests.md` 与 `README.md`：

```bash
# 构建（JDK 17/21/25）
./mvnw clean install -DskipTests          # 全量构建
./mvnw -pl quarkus/deployment,quarkus/dist -am -DskipTests clean install  # 仅服务端
# 发行包在 quarkus/dist/target

# 运行（下载发行包后）
bin/kc.sh start-dev
docker run quay.io/keycloak/keycloak start-dev

# 测试
./mvnw clean install                      # 全量测试（文档注明可能需数小时）
# 新一代集成测试位于 tests/，遗留 Arquillian 套件见 testsuite/integration-arquillian/HOW-TO-RUN.md
```

## 7. 测试与工程质量

- 测试规模：新框架 `tests/` 1082 个 Java 文件（base、clustering、conformance、webauthn、custom-providers 等），遗留 `testsuite/` 1068 个文件且根目录 `DEPRECATED.md` 明确标记弃用，正在向 `test-framework/` 的新框架迁移。
- 协议一致性：`docs/tests-oidc-conformance.md` 记录 OIDC 官方 conformance 套件接入；CI 有独立的 `conformance.yml` 工作流。
- CI：`.github/workflows/` 共 17 个流水线，含 ci.yml（Java 全矩阵）、js-ci.yml（前端）、operator-ci.yml、codeql-analysis.yml（代码扫描）、quarkus-next.yml（提前适配 Quarkus 新版）、version-compatibility-matrix.yml（版本兼容矩阵）、scan-dependencies.yml（依赖扫描），安全工程化程度高。
- 质量观察：SPI 契约层接口注释完备；核心类如 TokenManager 长达 1798 行，复杂度集中在协议细节；仓库配有 SECURITY-INSIGHTS.yml、OpenSSF Best Practices/Scorecard 徽章（`README.md`）。

## 8. 社区与维护状态

- 活跃度：gh api 显示最近 push 为 2026-11-16（分析当日），持续高频率提交。
- 发版节奏（`RELEASES.md` 自述 + gh api 核实）：SemVer，minor 约 4 次/年（26.7.0 于 2026-07-09、26.8.0 于 2026-10-01），major 每 2-3 年；patch 按需发布（26.8.1 10-13、26.8.2 11-05，约每 2-4 周一个安全/缺陷补丁），另提供 nightly 构建。
- 治理：Red Hat 主导开发，CNCF 孵化项目（README 徽章与 CLOMonitor 链接），社区会议、Weblate 翻译（translation-notify.yml）齐全；贡献者约 320 人（分页推算），Open Issues 约 3579。

## 9. 亮点与风险

亮点：
1. 插件化彻底——SPI 覆盖认证、联邦、协议、存储、事件等几乎所有能力，drop-in JAR 即可扩展（`server-spi/` + `ProviderManager.java`）。
2. 协议矩阵业界最全——OIDC/OAuth2/SAML/PAR/RAR/设备授权/OID4VC/AuthZEN/SSF，标准跟进速度快（`services/protocol/`、`authzen/`、`ssf/`）。
3. Quarkus 化后启动快、资源占用低，且可作为嵌入式扩展（`quarkus/README.md`）。
4. 前端控制台现代化——React+PatternFly+Vite，管理体验与商业产品无差距（`js/apps/`）。
5. 工程质量高——17 条 CI 流水线、双代测试体系、代码与依赖扫描、OpenSSF 合规。

风险/局限：
1. 复杂度高——Realm/客户端/流映射模型学习曲线陡峭，核心类（TokenManager 1798 行）改动门槛高。
2. 兼容性承诺有裂缝——`RELEASES.md` 自述"重要 bug 与安全修复可能破坏向后兼容"，升级需看迁移文档。
3. 旧适配器（`adapters/`）与遗留测试套件（`testsuite/`）处于维护/弃用状态，老方案用户需迁移。
4. Map Storage 路线已回退（`model/` 仅剩 jpa/infinispan/storage），曾押注该方向的扩展生态需要重建。

## 10. 总结评价

Keycloak 是"把身份做成基础设施"的教科书实现：以 SPI 为骨架、协议栈为血肉、Quarkus 为躯壳，用十年时间沉淀成全球最主流的开源 IAM。它既是可以直接上线的生产级产品（K8s Operator、HA 集群、月度安全补丁），也是学习认证协议与插件化架构的优质源码库。适合需要自托管 IAM 的平台团队、从 Auth0/Okta 迁出降本的组织；想精读源码的工程师，建议从 `server-spi` 契约层入手，再到 `services/authentication/AuthenticationProcessor.java` 认证流引擎与 `services/provider/ProviderManager.java` 加载机制，即可触达其设计精髓。
