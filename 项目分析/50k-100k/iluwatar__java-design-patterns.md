# iluwatar/java-design-patterns 项目分析

> 星数 94815（快照 2026-10-24）｜主语言 Java｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：全球最知名的 Java 设计模式开源教学库，用独立 Maven 模块逐一实现约 190 个设计模式与架构模式，每个模式配讲解 README、UML 图和单元测试，是 java-design-patterns.com 网站的内容源。

| 项目 | 事实 |
| --- | --- |
| 名称 | iluwatar/java-design-patterns |
| 星数 | 94815（任务快照；gh api watchers 94832，2026-10-31） |
| 主语言 | Java（Java 21，见 `pom.xml`、`AGENTS.md`） |
| 许可证 | MIT（`LICENSE.md`）；例外：`model-view-viewmodel` 模块使用 ZK 框架遵循 LGPL（`lgpl-3.0.txt`、根 `pom.xml` 头部注释），故 GitHub 识别为 NOASSERTION |
| 最近活跃 | 最近 push 2026-10-08（gh api），持续维护中 |
| 仓库创建 | 2014-08-09（gh api），版本 1.26.0-SNAPSHOT（`pom.xml`） |
| 官网 | https://java-design-patterns.com |

## 2. 项目是什么，解决什么问题

设计模式是形式化的最佳实践，能帮助程序员复用被验证过的设计方案、避免重复踩坑（README.md "Introduction"）。本项目把这一知识体系落地为**可运行的 Java 代码教程**：每个模式一个独立目录、独立 Maven 模块，读者可以直接阅读、编译、运行和修改。

覆盖面远超 GoF 23 个经典模式：既有 Creational/Structural/Behavioral 三大类（singleton、decorator、observer 等），也大量收录企业级与分布式场景模式，如 circuit-breaker（`circuit-breaker/`）、saga（`saga/`，同时提供 orchestration 与 choreography 两种实现）、microservices 系列 12 个模块（`microservices-api-gateway/`、`microservices-bulkhead/` 等）、架构模式 clean-architecture、hexagonal-architecture、onion-architecture 等。

每个模式模块遵循统一配方（`AGENTS.md` "Example Design Pattern" 一节明确规定）：模块根放讲解 README（含 YAML front-matter：title/category/tag，发布到官网）、`etc/` 放 UML 类图（.puml 源文件 + 渲染 png）与时序图、`src/main/java/com/iluwatar/<pattern>/` 放实现与 `App.java` 演示入口、`src/test/java/` 放 JUnit 5 测试。抽查确认 193 个模式目录全部配有 README.md。

项目的定位是教学参考而非生产库：强调 KISS/YAGNI（README.md），要求实现保持简单、原子、注释充分（`AGENTS.md` "Patterns" 节）。衍生生态包括官方网站（由姊妹仓库 java-design-patterns-vuepress-web 部署，见 `AGENTS.md`）、付费电子书 "Open Source Java Design Patterns"（README.md "The Book" 节）。

## 3. 技术栈与依赖

- **语言/运行时**：Java 21（`AGENTS.md` Stack 节；CI 用 temurin JDK 21，`.github/workflows/maven-ci.yml`）
- **构建**：Maven 多模块 + `mvnw` 包装器（根 `pom.xml`，191 个 `<module>`）；Spotless 强制 Google Java Format（`AGENTS.md`）
- **测试**：JUnit 5.11.4（含 junit-jupiter-params 参数化测试）、Mockito 5.14.2（`pom.xml` properties）
- **演示用途的第三方库**：Spring Boot 3.4.5（web/actuator/data-jpa，供部分企业模式模块使用）、Guice（dependency-injection）、Gson、MongoDB driver 5.4.0、H2 2.3.232、Lombok（`pom.xml`）
- **日志**：SLF4J 2.0.18 + Logback 1.5.34（`pom.xml`）
- **质量工具链**：JaCoCo 0.8.13 覆盖率、SonarCloud（`sonar.*` 属性在 `pom.xml`）、Codecov（`.github/workflows/maven-ci.yml`）
- **文档发布**：README front-matter 由 vuepress 姊妹仓库渲染成官网页面（`AGENTS.md` Quick Overview）

## 4. 目录结构与代码组织

顶层是"一模式一目录"的扁平结构，全部 193 个模式目录同时是 Maven 模块（`pom.xml` 中 191 个 `<module>`，assets/localization 等非代码目录不计）：

| 目录/文件 | 职责 |
| --- | --- |
| `<pattern>/` | 每个设计模式一个独立 Maven 模块，如 `singleton/`、`saga/`、`circuit-breaker/` |
| `<pattern>/README.md` | 模式讲解文档（intent/explanation/class diagram/applicability），发布至官网 |
| `<pattern>/etc/` | UML 图源文件（.puml）与渲染图（如 `singleton/etc/singleton.urm.puml`） |
| `<pattern>/src/main/java/com/iluwatar/<pattern>/` | 模式实现 + `App.java` 演示入口 |
| `<pattern>/src/test/java/` | JUnit 5 单元测试 |
| `localization/` | README 多语言翻译，23 种语言目录（`localization/` 下 ar/zh/ja/de 等） |
| `assets/` | 赞助商图片等静态资源 |
| `pom.xml` / `mvnw*` | 父 POM（191 模块、统一依赖版本）与 Maven 包装器 |
| `AGENTS.md` | 面向 AI 编程助手的贡献上下文（结构、规范、反模式） |
| `.github/` | CI 工作流、dependabot、FUNDING |

组织方式评述：典型"多模块教学 monorepo"。每个模式自成封闭模块，模块间几乎零依赖（避免读者只关心一个模式时被迫理解其他代码），父 POM 统一管理版本与插件。这种结构对教学极友好，代价是父 POM 维护繁琐（新增模式需手动加 `<module>`，`AGENTS.md` 已将此写入标准流程）。

## 5. 核心架构与关键实现

由于是教学库，无统一运行时架构，"核心"体现在模式的实现质量与代表性。以下抽查 4 个（降采样分析，覆盖经典/现代/分布式三类）：

### 5.1 singleton（经典 GoF，多实现对比）

`singleton/src/main/java/com/iluwatar/singleton/` 一次给出 6 种实现：IvoryTower（饿汉）、EnumIvoryTower（枚举）、BillPughImplementation（静态内部类 holder）、InitializingOnDemandHolderIdiom、ThreadSafeDoubleCheckLocking、ThreadSafeLazyLoadedIvoryTower，便于横向对比线程安全与懒加载取舍。Bill Pugh 实现还加入了反射防护：

```java
// singleton/src/main/java/com/iluwatar/singleton/BillPughImplementation.java
private BillPughImplementation() {
  // to prevent instantiating by Reflection call
  if (InstanceHolder.instance != null) {
    throw new IllegalStateException("Already initialized.");
  }
}
private static class InstanceHolder {
  private static BillPughImplementation instance = new BillPughImplementation();
}
public static BillPughImplementation getInstance() {
  return InstanceHolder.instance;
}
```

### 5.2 observer（经典 + 泛型化重写）

`observer/` 提供 22 个 Java 文件：基础版 Weather/Hobbits/Orcs 场景（`observer/Weather.java`），以及 `observer/generic/` 下类型安全的泛型 `Observable`/`Observer` 实现，展示同一模式的演进写法。测试用 `InMemoryAppender` 工具捕获日志断言行为（`observer/src/test/java/com/iluwatar/observer/utils/InMemoryAppender.java`），并大量使用 JUnit 5 `@ParameterizedTest`（`observer/src/test/java/com/iluwatar/observer/generic/ObserverTest.java`）。

### 5.3 circuit-breaker（现代容错模式）

`circuit-breaker/src/main/java/com/iluwatar/circuitbreaker/` 共 9 个类：接口 `CircuitBreaker` 定义 `recordSuccess/recordFailure/getState/setState/attemptRequest`（`CircuitBreaker.java`），`DefaultCircuitBreaker.java` 实现闭路-开路-半开状态机（`State.java`），配 `MonitoringService` 与两个模拟远端服务（`QuickRemoteService`/`DelayedRemoteService`）演示熔断效果。

### 5.4 saga（分布式事务，双实现对比）

`saga/` 同时实现编排式与协奏式（`saga/src/main/java/com/iluwatar/saga/orchestration/` 含 `SagaOrchestrator.java`；`saga/.../choreography/` 基于 `ServiceDiscoveryService` 的事件驱动），用订票-酒店-扣款的旅行预订场景串联 5 个服务，配 5 个测试类（`saga/src/test/java/`）。这种"一个模式给两种范式实现"的做法是仓库的高价值特色。

模块规模分布：最大为 hexagonal-architecture（42 个 Java 文件）、backends-for-frontends（32）、specification（30）、commander（27）、saga（25）；多数经典模式在 5-15 个文件。统计自各模块 `find -name "*.java"`。

## 6. 构建与运行

```bash
# 全量构建 + 测试（CI 同款命令，见 .github/workflows/maven-ci.yml）
./mvnw clean verify
# 单模式运行（模块内 App 均带 main 方法）
cd singleton && ../../mvnw exec:java -Dexec.mainClass=com.iluwatar.singleton.App
# 代码格式化（Spotless / Google Java Format，AGENTS.md）
./mvnw spotless:apply
```

说明：CI 中 GUI 相关测试需 xvfb（`maven-ci.yml` 安装 xvfb 步骤），本地 Linux 无头环境同理；`mvnw` 会自动下载 Maven。规范要求本分析不实际执行构建，以上命令来自项目文件而非运行验证。

## 7. 测试与工程质量

- 测试规模：655 个测试类 / 2042 个 Java 文件（约 1:3），单测为主，无独立集成/e2e 层（统计自 `find */src/test -name "*.java"`）
- 风格：JUnit 5 + Mockito + 参数化测试；通用技巧是用 `InMemoryAppender` 等工具类把控制台输出变为可断言对象（`observer/src/test/.../utils/InMemoryAppender.java`）
- 覆盖率与静态分析：JaCoCo + SonarCloud 徽章展示在 README（README.md 头部），Codecov 上传（`maven-ci.yml`）
- CI：`maven-ci.yml`（push master，JDK 21 全量 verify + Sonar）、`maven-pr-builder.yml`（PR 构建门禁）、`stale.yml`（僵尸 issue 治理）
- 亮点：`presubmit.yml` 接入 LLM 代码评审（presubmit/ai-reviewer，模型 gpt-5-nano），配合根目录 `AGENTS.md` 为 AI 贡献者提供结构化上下文——在人海贡献型教学仓库里用 AI 把关代码风格与教学约定，属于少见但合理的现代化运维
- 代码风格强制：Spotless + Google Java Format（`AGENTS.md`），实际抽查的源文件格式高度一致（如 `BillPughImplementation.java` 的 javadoc 密度）

## 8. 社区与维护状态

- 活跃度：创建于 2014-08，最近 push 2026-10-08（gh api），11 年持续活跃；145 个 open issue
- 贡献者：README all-contributors 徽章显示 418 位贡献者（README.md），核心维护者为 Ilkka Seppälä（iluwatar）及 ohbus、npathai 等（README Contributors 表）；27356 forks（gh api）
- 节奏：无固定 release 说明（tarball 无 CHANGELOG），版本号 1.26.0-SNAPSHOT 滚动推进（`pom.xml`）；具体 release 频率未确认
- 生态位：hacktoberfest 参与（topics 含 hacktoberfest，gh api）、SerpApi 赞助（README Supporters）、Gitter 社区（README）、贡献者可免费获电子书（README "The Book"）
- 多语言：23 个语言翻译目录（`localization/`），社区翻译驱动

## 9. 亮点与风险

亮点：
1. 规模与体系化无可匹敌：193 个模式模块、约 13.4 万行 Java（实测统计），从 GoF 经典到微服务/架构模式一站式覆盖
2. 每模式"README + UML + 可运行代码 + 单测"四件套配方统一，自学者体验极佳，也是各语言 pattern 仓库效仿的模板
3. 工程质量超出教学库平均线：JDK 21、JUnit 5 参数化、SonarCloud/Codecov/Spotless 全套门禁、AI 评审工作流（`.github/workflows/presubmit.yml`）
4. `AGENTS.md` 主动为 AI 编码代理提供贡献上下文，规范、反模式、新增模式步骤写得非常清楚
5. 双实现对比教学（saga 编排/协奏、observer 经典/泛型、singleton 六实现），利于理解取舍

风险/局限：
1. 教学代码刻意简化，不能直接作为生产组件使用；部分模式依赖 Spring Boot 等重框架演示，初学者可能被环境问题劝退
2. 191 个 Maven 模块全量构建慢、父 POM 手工维护易漏（`AGENTS.md` 需专门写步骤说明即为佐证）
3. 许可证混合（MIT 主体 + LGPL 的 model-view-viewmodel 模块），商用集成需留意（根 `pom.xml` 头注释、`lgpl-3.0.txt`）
4. 网站文档与仓库 README 双源同步依赖人工/流程，偶有滞后风险（未确认，仅结构推断自 `AGENTS.md`）

## 10. 总结评价

这是设计模式学习领域事实上的标准开源参考：把模式知识从书页搬进可编译、可测试、可 diffs 的代码里，并用统一的四件套模块配方维持了 11 年的一致性。它未必教"最佳实践的生产代码"，但极好地教"模式的形状"。适合 Java 初中工程师按需查阅单个模块精读、面试备战与架构选型前速览；想学习如何组织大型教学型开源仓库、如何用 CI/质量工具链与 AI 辅助治理社区贡献的维护者，也值得把它当范本研究源码。
