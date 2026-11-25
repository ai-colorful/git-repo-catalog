# google/guava 项目分析

> 星数 51918（快照 2026-10-24）｜主语言 Java｜档位 50k-100k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：Google 出品的 Java 核心工具库，为 JDK 提供集合扩展（Multimap/Multiset）、不可变集合、图库、缓存、并发原语与字符串/IO/哈希等日常工具，是 Java 生态引用最广的"事实标准补充库"之一。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | google/guava（Guava: Google Core Libraries for Java） | `README.md` |
| 星数 | 51918（快照 2026-10-24）；watchers 51917、open issues 766 | 派发参数 / gh api |
| 主语言 | Java（100% 纯 Java，无构建脚本外的其他语言源码） | 源码树 |
| 许可证 | Apache-2.0（`LICENSE`，父 POM 同样声明） | `LICENSE`、`pom.xml` |
| 最近活跃 | 最近 push 2026-11-03，最新 release v33.8.0 发布于 2026-11-02 | gh api |
| 贡献者 | 500+ 人（contributors API 翻页至 501 页） | gh api |
| 构建体系 | Maven 多模块（附 mvnw wrapper），JRE/Android 双 flavor | `pom.xml`、`android/pom.xml` |
| 运行要求 | 源码/目标级别 Java 8（JRE flavor 需 JDK 8+，Android flavor API 24） | `README.md`、`pom.xml` |

## 2. 项目是什么，解决什么问题

Guava 起源于 Google 内部（前身是 google-collections，这一点仍以 Gradle capability `com.google.collections:google-collections` 的形式保留在发布元数据中，见 `guava/module.json`），2010 年开源后成为 Java 世界的"JDK 前传"：在 JDK 覆盖不足的年代，它把 Google 内部生产验证过的工具抽象沉淀成公共库。README 的自我定位是 "a set of core Java libraries from Google"，并明确说明它 "is widely used on most Java projects within Google"（`README.md`）。

它解决的问题可以归纳为四类。第一，**集合能力扩展**：JDK 原生集合缺少 Multimap、Multiset、BiMap、Table、RangeMap 等结构，Guava 的 `collect` 包以 216 个源文件（本仓库最大的包）补齐，同时提供了成体系构建器风格的不可变集合（`ImmutableList/ImmutableSet/ImmutableMap/ImmutableBiMap` 等）。第二，**不可变与防御性编程**：不可变集合是 Guava 的招牌设计，配套 `Preconditions`（`base/Preconditions.java`，1540 行）形成参数校验惯例。第三，**并发原语**：`ListenableFuture` 比 `CompletableFuture` 早诞生数年，`Service` 生命周期框架、`Monitor`、`Striped`、`CycleDetectingLockFactory` 等都是 JDK 没有的能力（`util/concurrent/` 84 个文件）。第四，**本地缓存**：`cache` 包的 `CacheBuilder`/`LocalCache` 在 Caffeine 出现之前是 JVM 内存缓存的事实标准（`cache/LocalCache.java`，4996 行）。

Guava 有一个对兼容性极为认真的版本承诺（`README.md` "IMPORTANT WARNINGS" 节）：非 `@Beta` API 永久保持二进制兼容（最后一次删除非 Beta API 是 21.0 版）；`@Beta` API 随时可能改动或删除，并提供了专门的 guava-beta-checker 工具帮助下游库自查。全仓库当前仅 16 个文件含 `@Beta`（`grep -rl "@Beta"` 统计），说明 Beta 面已大幅收缩。

发布形态上，Guava 分 **JRE flavor** 与 **Android flavor** 两个变体，版本号分别为 `33.8.0-jre` / `33.8.0-android`。两者的唯一运行期依赖是 `com.google.guava:failureaccess:1.0.3`，其余均为编译期注解依赖（error_prone_annotations、jspecify、j2objc-annotations）（`README.md`、`guava/pom.xml`）。

## 3. 技术栈与依赖

- **语言/编译**：Java，`source/target 1.8`（`pom.xml` maven-compiler-plugin 配置），但构建环境要求 Java 11+，工具链默认 JDK 27 用于运行 javac 与 Javadoc（`pom.xml` toolchains 插件与注释）。这一点在 `.github/workflows/ci.yml` 的注释中有明确解释。
- **构建**：Maven 多模块 + `mvnw` wrapper（`.mvn/wrapper/maven-wrapper.properties`）；用 maven-bundle-plugin 产出 OSGi bundle，并用 `/module-info.class` + `META-INF/versions/9/` 提供 JPMS module-info（父 POM `maven-jar-plugin` 排除规则注释）。
- **静态检查**：Error Prone 编译器插件以 `Java8ApiChecker:ERROR` 级别内嵌进每次编译（`pom.xml` compilerArgs），配合 animal-sniffer 用 `java18` 签名校验不得越界使用 Java 8 之后的 API（`pom.xml` animal-sniffer-maven-plugin 配置）——这是"库必须运行在 Java 8"的机器强制。
- **空值注解**：迁移至 JSpecify（`org.jspecify`，依赖管理见 `pom.xml`），源码中大量 `@Nullable` 出自 jspecify 命名空间（如 `AbstractFuture.java` 的 import）。
- **测试栈**：JUnit 4.13.2、Google Truth 1.4.5（342 个测试文件引用）、Mockito 4.11、test-parameter-injector 1.22（`pom.xml` properties）。
- **性能测试**：Caliper 1.0-beta-3（`guava-tests/benchmark/`）。
- **平台/转译注解**：GWT（gwt.version 2.13.1，独立模块 guava-gwt）、J2ObjC（j2objc-annotations 3.1）与 Kotlin/J2ktIncompatible 注解——Guava 同一套源码要服务 JVM/JS(GWT)/iOS(J2ObjC) 三个目标（各文件头部 `@GwtCompatible/@GwtIncompatible/@J2ObjCIncompatible` 注解可见，如 `util/concurrent/AbstractFutureState.java`）。
- **运行期依赖总计：1 个**（failureaccess，`README.md` 第 3 条警告）。

## 4. 目录结构与代码组织

顶层目录（`ls` 实测）：

| 目录 | 职责 |
|---|---|
| `guava/` | JRE flavor 主库模块（`src/` 611 个 Java 文件；`module.json` 为 Gradle Module Metadata；`javadoc-link/` 供离线 Javadoc 链接） |
| `android/` | Android flavor 完整平行构建树（自带 pom.xml + guava/guava-bom/guava-testlib/guava-tests 四个子模块，`android/guava/src` 610 个文件） |
| `futures/` | 一次性发布的独立构件：`failureaccess`（仅 2 个类 + module-info）、`listenablefuture1`、`listenablefuture9999`（空 jar 防冲突占位，见 `futures/README.md`："The modules under this directory will be released exactly once each"） |
| `guava-bom/` | Maven BOM 物料清单 |
| `guava-gwt/` | GWT 超源码（super-source）模块 |
| `guava-testlib/` | 独立发布的"集合行为测试框架"（298 个文件，`README.md` 有独立坐标说明） |
| `guava-tests/` | 主库测试（`test/` 609 个文件）+ Caliper 基准（`benchmark/`） |
| `integration-tests/` | Gradle 消费方集成测试（`gradle/`，由 `util/gradle_integration_tests.sh` 驱动） |
| `proguard/` | 各包的 ProGuard keep 规则（base/cache/collect/concurrent/hash/primitives.pro） |
| `util/` | 维护脚本：快照发布、Gradle 集成测试、Surefire 报告打印、快照文档生成 |
| `cycle_suppress_list.txt` | 依赖环检测豁免清单（jdepend 风格 NAMESPACE 条目） |

包组织（`guava/src/com/google/common/`，按文件数排序）：collect 216、util(concurrent) 84、graph 54、base 53、io 38、hash 34、primitives 26、cache 23、reflect 17、math 17、eventbus 12、escape 11、net 10、annotations 6、xml/html 各 3。

组织方式评述：这不是常规的"每模块一个业务域"多模块仓库，而是**同一套 API 的双源码树 + 外围工程件**结构。Android flavor 不走构建期变体（profile/aspect），而是维护一棵逐文件平行的源码树（JRE 607 vs Android 607 个 com/google/common 文件），两边各有一份 `base/Platform.java` 承担平台差异的运行时分派；其余代码保持人工同步（发布元数据层面则通过 `guava/module.json` 的 Gradle variants 暴露 jre/android 两个 `org.gradle.jvm.environment` 属性供 Gradle 消费方自动选择）。主包内部采用"公开 API + 私有实现类拆分"风格：如 `ImmutableSet.java`（944 行，构建器与校验）与 `RegularImmutableSet.java`（106 行，紧凑数组 + 开地址哈希表）、`cache/CacheBuilder.java`（参数面）与 `cache/LocalCache.java`（4996 行实现面）分离。

## 5. 核心架构与关键实现

### 5.1 不可变集合（collect）

`ImmutableList`/`ImmutableSet`/`ImmutableMap` 的共同设计是：公开类只做参数校验、构建器和序列化（用静态代理类防反序列化伪造，`ImmutableList.java` 中 `readObject` 抛 `InvalidObjectException`），真正的存储是包私有的 `Regular*` 类——`RegularImmutableList` 直接持有数组；`RegularImmutableSet` 则用"元素数组 + 开地址哈希表 + smear 哈希"三件套（`RegularImmutableSet.java`：`smearedHash`、`table`、`mask` 字段），把查找成本和内存占用同时压到最低。构建期对哈希碰撞做 `HashCodeSmearing`（基于 `Hashing.smearedHash`）而非 JDK `Object.hashCode` 直用，缓解用户类 hashCode 质量差导致的退化。

```java
// guava/src/com/google/common/collect/RegularImmutableSet.java
final class RegularImmutableSet<E> extends ImmutableSet.CachingAsList<E> {
  static final RegularImmutableSet<Object> EMPTY =
      new RegularImmutableSet<>(EMPTY_ARRAY, 0, EMPTY_ARRAY, 0);
  private final transient int hashCode;
  @VisibleForTesting final transient @Nullable Object[] table;
  // 'and' with an int to get a valid table index.
```

不可变集合体系还通过 `ImmutableCollection` 抽象 + `ArrayBasedBuilder` 共享容量管理；`ImmutableMap`（1344 行）内部用 `ImmutableMapEntry` 闭链哈希表（`collect/ImmutableMapEntry.java`）。

### 5.2 缓存：LocalCache 的分段设计（cache）

`LocalCache.java` 是全仓库最大的类（4996 行），是照着老版 `ConcurrentHashMap` 的分段思路为"带失效语义的缓存"重写的实现：段数取不小于 concurrencyLevel 的最小 2 的幂，读路径无锁，写/失效路径按段加 `ReentrantLock`。它同时支持 size 上限、weight 上限（配合 `Weigher`）、expireAfterWrite/Access、refreshAfterWrite 与弱键/弱值/软值（`CacheBuilder` 参数面）。

```java
// guava/src/com/google/common/cache/LocalCache.java
final Segment<K, V> segmentFor(int hash) {
  return segments[(hash >>> segmentShift) & segmentMask];
}
...
/**
 * Segments are specialized versions of hash tables. This subclass inherits from ReentrantLock
 * opportunistically, just to simplify some locking and avoid separate construction.
 */
static final class Segment<K, V> extends ReentrantLock {
```

有意思的工程细节：`LocalCache` 为了能在 Java 8 上同时兼容 `java.util.concurrent.atomic.LongAdder` 与老 JDK，抽象出 `LongAddable`（`cache/LongAddables.java`），运行时探测后选择实现——这是 Guava "在最低语言级别上服务最长生命周期" 的典型做法。

### 5.3 并发：AbstractFuture 的无锁状态机（util/concurrent）

`ListenableFuture` 的基类 `AbstractFuture.java`（1058 行）已把全部可变状态抽到 `AbstractFutureState.java`（786 行）：`valueField`、`waitersField`（等待线程组成的无锁 Treiber 栈）、`listenersField`（监听器栈）三个字段用单一 CAS 协议驱动完成/传播。底层原子操作按运行环境动态选择：检测到 `java.lang.invoke.VarHandle`（Java 9+）就用 VarHandle 实现，否则回退；J2ObjC 目标走单独分支（`AbstractFutureState.java` 的 `VarHandleAtomicHelperMaker` 与 `AtomicHelper` 抽象）。

```java
// guava/src/com/google/common/util/concurrent/AbstractFutureState.java
private abstract static class AtomicHelper {
  abstract void putThread(Waiter waiter, Thread newValue);
  abstract void putNext(Waiter waiter, @Nullable Waiter newValue);
  abstract boolean casWaiters(AbstractFutureState<?> future,
      @Nullable Waiter expect, @Nullable Waiter update);
  abstract boolean casListeners(AbstractFutureState<?> future,
      @Nullable Listener expect, Listener update);
  abstract boolean casValue(AbstractFutureState<?> future,
      @Nullable Object expect, Object update);
}
```

`failureaccess` 构件里只有 `InternalFutureFailureAccess` 与 `InternalFutures` 两个类（`futures/failureaccess/src/.../internal/`），其存在意义是让 `AbstractFuture` 实现 `InternalFutureFailureAccess` 接口、允许下游在不依赖 Guava 全量 jar 的情况下复用 tryFastPath 逻辑——一个极小化依赖面的发布工程决策。

### 5.4 图库（graph）

`graph/` 包（54 个文件）按"元素类型 × 视角"组织：`ValueGraph`（节点+边值）、`Network`（区分节点与边对象、支持平行边）、`Graph`（纯拓扑）三套 API，底下共享 `AbstractBaseGraph` 与 `DirectedGraphConnections`/`AbstractDirectedNetworkConnections` 等连接表实现（目录列表见 4 节输出）。该包自 20.0 起脱离 @Beta（未在本仓库内验证版本节点，标注为"未确认"），是 Guava 少数"新造领域模型"而非包装 JDK 的部分。

## 6. 构建与运行

来自 `README.md` 与 CI 脚本（`.github/workflows/ci.yml`）的实测命令：

```bash
# 构建全量（跳过测试；构建需 JDK 11+，工具链会自动下载 JDK 27）
./mvnw -B -Dtoolchain.skip install -DskipTests=true

# 跑测试（可指定被测 JDK：8/11/17/25；默认两棵树：根 pom = JRE，android/pom.xml = Android）
./mvnw -B -Dtoolchain.skip verify -Dsurefire.toolchain.version=25
./mvnw -B -Dtoolchain.skip verify -f android/pom.xml

# Gradle 消费方集成测试
util/gradle_integration_tests.sh
```

消费方使用（`README.md`）：

```gradle
dependencies {
  api("com.google.guava:guava:33.8.0-jre")      // 或 33.8.0-android
}
```

注意事项（`pom.xml`/CI 注释）：本地构建者可用 `-Dsurefire.toolchain.version=21` 切换测试 JDK；快照版本为 `999.0.0-HEAD-jre-SNAPSHOT`，由 CI 在 master push 后自动发布（`util/deploy_snapshot.sh`）。

## 7. 测试与工程质量

- **规模**：主库测试 609 个文件（`guava-tests/test/`，与主包一一镜像，collect 包测试就有 236 个）；testlib 另有 298 个文件；Android 树自带独立测试模块。
- **风格**：三层体系。(1) 普通单元测试（JUnit4 + Truth，342 个文件引用 Truth）。(2) **guava-testlib 行为契约测试**：`AbstractCollectionTestSuiteBuilder` 等生成器为每种集合特性自动组合出 tester 套件，同一套契约同时约束 JDK 集合适配器与 Guava 自有实现——这是仓库最独特的质量资产。(3) 覆盖式集成：Gradle 消费方集成测试 + GWT/J2ObjC 编译目标。
- **CI**：`.github/workflows/ci.yml` —— JDK 8/11/17/25 × 两棵根 pom 的矩阵，外加 Windows(latest JDK)、Gradle 集成测试、master push 后的快照发布与快照文档生成（含 JDiff API diff）。另有 scorecard.yml（OpenSSF Scorecard）与 dependabot.yml。
- **质量细节**：测试 JVM 故意设 `-Duser.language=hi -Duser.country=IN` 以捕获对默认 locale 的隐式依赖（`pom.xml` argLine 注释 "Catch dependencies on the default locale by setting it to hi-IN"）；surefire 显式排除 `*PackageSanityTest`、`*Tester` 等辅助类；`cycle_suppress_list.txt` 维护包依赖环豁免。Error Prone `Java8ApiChecker:ERROR` + animal-sniffer java18 双保险防 API 越界（`pom.xml`）。
- **观察**：611 个主源文件对 609 个测试文件，测试/源码比接近 1:1；单文件巨类（LocalCache 4996 行、Maps 4500 行）与超长 Javadoc 并存，属于"文档即 API 契约"的教科书式写法，但新贡献者面对单文件时上下文负担明显。

## 8. 社区与维护状态

- **活跃度**：最近 push 2026-11-03（分析前一天），最新 release v33.8.0 发布于 2026-11-02（gh api 实测）——一个 16 年老项目仍在高频迭代。
- **发布节奏**：从 v33.8.0（2026-11-02）与 README 中的版本号看，当前处于 33.x 系列；快照由 CI 全自动发布（`ci.yml` publish_snapshot job），正式版通过 central-publishing-maven-plugin 走 Sonatype Central（`pom.xml`）。历史上 Guava 以"数月一版、API diff 全程可查"著称（`util/update_snapshot_docs.sh` 维护快照 Javadoc 与 API diffs，未确认精确历史节奏）。
- **贡献者结构**：contributors 500+（gh api 分页页尾 last=501）；父 POM `<developers>` 只署名 Chris Povirk（cpovirk@google.com），并注释说明 "Sonatype requires that we list someone here. Guava has many contributors"（`pom.xml`）——实际由 Google 内部小组长期主导，外部 PR 走 issue/PR 流程（`.github/pull_request_template.md`、`CONTRIBUTING.md`）。
- **治理**：OpenSSF Best Practices 徽章（`README.md`）、Scorecard 工作流、全 pinned-by-SHA 的 CI action 引用（`ci.yml` 中所有 uses 均带 commit SHA）体现较高的供应链安全水位。
- **issue 面积**：open issues 766（gh api），对一个 5 万星基础设施库属克制水平。

## 9. 亮点与风险

**亮点**

1. **兼容性承诺的工程化落地**：非 Beta API 永久二进制兼容不是口头承诺，而是由 Error Prone Java8ApiChecker（编译期 ERROR 级）、animal-sniffer java18 签名、CI 的 JDK 8 矩阵、`InlineMe`/`DoNotCall` 注解体系共同机器执行（`pom.xml`、`README.md`）。
2. **测试库即产品**：guava-testlib 把"集合应满足什么行为"变成可复用契约，298 个文件的测试生成器框架对整个 JVM 生态的集合正确性都有溢出价值（`guava-testlib/src/com/google/common/collect/testing/`）。
3. **多平台单源码**：同一棵源码服务 JVM/GWT/J2ObjC 三个目标，用 `@GwtIncompatible`（298 个文件）与 `@J2ObjCIncompatible` 等注解精确圈定不可移植面。
4. **发布工程成熟**：JRE/Android 双 flavor + Gradle Module Metadata variants（`guava/module.json`）+ BOM + 极小运行期依赖（failureaccess 1.0.3）+ 快照自动发布与 API diff 文档化。
5. **核心并发/缓存实现的学术级严谨**：`AbstractFutureState` 的 VarHandle 无锁状态机、`LocalCache` 的分段读写设计是学习 JVM 并发编程的高质量范本。

**风险/局限**

1. **双源码树的同步成本**：JRE/Android 两棵 607 文件的平行树靠人工/脚本保持一致，任何修改都要双写（`guava/src` 与 `android/guava/src` 实测计数），长期是维护摩擦点。
2. **与现代 JDK 的重叠**：`CompletableFuture`、`java.util.stream`、`Map.of` 等让 Guava 的部分 API（尤其 Futures 工具与部分 base 工具）在纯 Java 11+ 项目中价值下降；`ListenableFuture` 生态仍深度绑定，但新项目选型需权衡。
3. **单文件巨类**：`LocalCache.java`（4996 行）、`Maps.java`（4500 行）等对源码阅读和增量修改不友好。
4. **序列化形式无承诺**：`README.md` 明确"所有对象的序列化形式都可能变化，不得持久化"，跨版本缓存反序列化等场景有坑。

## 10. 总结评价

Guava 是 Java 生态里少数"既是基础设施又是方法论"的项目：它把 Google 内部十几年对正确性、兼容性与 API 演进的思考固化成了可复制的工程实践——机器强制的兼容性检查、把测试框架当独立产品发布、双 flavor 的发布元数据设计，都远超一般工具库的水准。它如今不再是"新项目必选"，但仍是 Java 开发者的默认词汇表（Multimap、ImmutableList、CacheBuilder 这些概念已经沉淀进生态）。适合所有 Java 工程师作为日常依赖使用；更适合三类人精读源码：想学无锁并发与缓存设计的进阶者（`util/concurrent/AbstractFutureState.java`、`cache/LocalCache.java`）、要构建自己公共库的架构师（看它的注解体系与发布工程）、以及需要高质量集合行为测试思路的测试框架作者（`guava-testlib`）。
