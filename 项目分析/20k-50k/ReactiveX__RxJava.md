# ReactiveX/RxJava 项目分析

> 星数 48200（快照 2026-10-24）｜主语言 Java｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：JVM 上 Reactive Extensions 的参考实现——用可观察序列组合异步与基于事件的程序的 Java 库，Reactive Streams 规范的事实标准之一。

| 项目 | 内容 |
|---|---|
| 名称 | ReactiveX/RxJava |
| 星数 | ~48.2k（gh api 实测 48155，2026-11-05） |
| 语言 | Java（纯 Java，零第三方运行时依赖） |
| 许可证 | Apache-2.0（`LICENSE`） |
| 默认分支 | `4.x`，版本 4.0.0-SNAPSHOT（`gradle.properties`），计划 2026-11-30 正式发布 4.0.0（`README.md` 第 3 行） |
| 最近活跃 | 最近 push 2026-11-03（gh api），最新 release v4.0.0-alpha-21（2026-08-05） |
| 规模 | 主代码 945 个 Java 文件 / 测试 1079 个 / JMH 基准 64 个 |
| 维护者 | David Karnok（akarnokd，POM 开发者字段）等，贡献者 100+ |

重要说明：本分析基于默认分支 `4.x` 的 HEAD tarball。该分支是一次面向 Java 26 与虚拟线程的大版本重写（4.0.0，计划 2026-11-30 发布）；生产中广泛使用的 3.x（`io.reactivex.rxjava3`）进入维护模式（`README.md:31`："RxJava 3.x support will be toned down... offered for +1 year after 4.x official release"）。3.x 的架构设计与 4.x 同源（`DESIGN.md` 即 v3 设计文档），核心结论对 3.x 大体适用。

## 2. 项目是什么，解决什么问题

RxJava 是 Observer 模式的扩展：把数据/事件序列化，用声明式操作符（map/filter/flatMap/...）组合，并把底层线程、同步、线程安全与并发数据结构等细节抽象掉（`README.md`）。它是 Reactive Streams 规范在 JVM 上的最早、最广泛的实现之一，是 Android 与 Java 后端响应式编程生态的奠基项目。

它解决的核心问题是**异步程序的组合性**。回调嵌套（callback hell）、错误传播断裂、资源取消困难、多线程数据竞争，这些问题在传统异步代码中各自为战；RxJava 用统一的订阅契约 `onSubscribe onNext* (onError|onComplete)?`（`DESIGN.md` "Type Signature" 部分）把它们归约为一个可组合的流模型，错误沿流传播、取消沿流回传、线程切换由调度器声明。

第二个核心问题是**流控（背压）**。当生产者快于消费者时，无界缓冲会撑爆内存。RxJava 的答案是双类型体系：`Observable` 面向无需流控的场景（短序列、GUI 事件），`Flowable` 实现带 `request(n)` 信用机制的 Reactive Streams `Publisher`；4.0 又新增第三种答案 `Streamable`——基于虚拟线程的"生产者-消费者互相等待"的自然背压拉模型（`README.md:109-111`，`core/Streamable.java`）。这套设计取舍在 `DESIGN.md` 中有系统论述（Interactive vs Reactive、Hot vs Cold、三种流控模式）。

4.0 版本是维护者 akarnokd 重掌主仓后的一次大重写：原生 Java 26、直接实现 `java.util.concurrent.Flow.Publisher`（不再依赖 org.reactivestreams 运行时）、虚拟线程一等公民（`Schedulers.virtual()`、`virtualCreate()`）、新增 `Streamable`/`Streamer` 类型（自比 C# 的 `IAsyncEnumerable`，`core/Streamer.java` javadoc）、用 Java Cleaner API 做资源泄漏检测（`README.md` 特性列表）。

## 3. 技术栈与依赖

- 语言：Java，基线 **JDK 26**（`build.gradle` toolchain，可用 `JDK_EXPERIMENTAL` 环境变量切到 27 做预览构建）。
- 运行时依赖：**零**（`build.gradle` dependencies 全部为 testImplementation；`README.md` 特性宣称 "No 3rd party library required at runtime"）。
- 测试依赖：JUnit Jupiter 6.1.3、TestNG 7.12.0（跑 Reactive Streams TCK）、reactive-streams-tck(-flow) 1.0.4、Mockito 5.x、Guava（仅测试，`build.gradle` ext 块）。
- 构建体系：Gradle 单模块工程（`settings.gradle` 仅 `rootProject.name='rxjava'`），插件：java-library、checkstyle、JMH（me.champeau.jmh 0.7.4）、license、bnd.builder（OSGi）、vanniktech.maven.publish、beryx.jar（JPMS 打包）。
- 规范实现：`java.util.concurrent.Flow.Publisher`（`core/Flowable.java:18` `import java.util.concurrent.Flow.*`）；Reactive Streams org.reactivestreams 仅作为 TCK 测试依赖存在。
- 模块化：JPMS module `io.reactivex.rxjava4`（`src/main/module/module-info.java`）+ OSGi Bundle（`build.gradle` bnd 任务）双轨。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `src/main/java/io/reactivex/rxjava4/core/` | 五大基类 Observable/Flowable/Single/Maybe/Completable + 新增 Streamable，以及 Scheduler、Emitter、BackpressureStrategy 等门面 API |
| `src/main/java/io/reactivex/rxjava4/internal/operators/` | 全部操作符实现，589 个类，按目标类型分子包：flowable 165 / observable 149 / single 58 / maybe 76 / completable 46 / mixed 26 / parallel 17 / streamable 52 |
| `internal/schedulers/` | 调度器实现 23 个类（Computation/NewThread/Single/Executor/Cached/Parallel 等） |
| `internal/fuseable/`、`internal/queue/`、`internal/subscribers|observers/` | 操作符融合协议、内部队列（仅 MpscLinkedQueue 1 个，其余用 j.u.c）、订阅者骨架 |
| `internal/virtual/` | 4.x 新增：虚拟线程 create/transform 执行器（3 个类） |
| `subjects/`、`processors/` | 热source（Publish/Behavior/Replay/Unicast 等 11 个）与 Reactive Streams Processor（10 个） |
| `disposables/`、`exceptions/`、`functions/`、`plugins/`、`parallel/`、`annotations/` | 取消机制、异常类型、函数接口、RxJavaPlugins 全局钩子、ParallelFlowable、注解 |
| `src/main/module/` | JPMS module-info.java |
| `src/test/java/.../tck/` | 128 个 Reactive Streams TCK 测试类（TestNG） |
| `src/test/java/.../validators/` | 自研源码扫描"元测试"（检查 license 头、throwIfFatal、javadoc 规范等） |
| `src/jmh/` | 64 个 JMH 微基准 |
| `docs/` | 41 篇 wiki 镜像文档（操作符索引、背压、贡献指南等） |
| `.github/workflows/` | 12 个 CI 工作流（见第 7 章） |

组织方式评述：经典**单模块大单体**，但内部划分极其严格——公共 API 只在 `core/` 等顶层包，所有操作符实现藏进 `internal/`（javadoc 与 `validators/CheckInternalWrongNamingTest` 强制内部类命名规范）。设计上"基类抽象 + 每操作符一个 final 类 + 内部 Subscriber 实现"（见第 5 章），数量庞大的操作符文件按目标类型分包，Observable 与 Flowable 两套近似镜像的实现并存（CONTRIBUTING.md 明示改一个类型常需同步改另两个）。

## 5. 核心架构与关键实现

### 5.1 基类抽象与订阅协议（core/）

五大基类均为抽象类，持有操作符方法；订阅入口 `subscribe()` 为 final，做 null 检查 + `RxJavaPlugins.onSubscribe` 钩子 + 异常兜底（不能违反 RS 协议抛任意异常，故包成 NPE 重抛），再转发给子类实现的 `subscribeActual()`（`core/Observable.java:11027-11059`）：

```java
public final void subscribe(@NonNull Observer<? super T> observer) {
    Objects.requireNonNull(observer, "observer is null");
    try {
        observer = RxJavaPlugins.onSubscribe(this, observer);
        ...
        subscribeActual(observer);
    } catch (Throwable e) {
        Exceptions.throwIfFatal(e);
        // can't call onError because no way to know if a Disposable has been set or not
        RxJavaPlugins.onError(e);
        NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
        npe.initCause(e);
        throw npe;
    }
}
protected abstract void subscribeActual(@NonNull Observer<? super T> observer);
```

每个具体操作符就是一个继承基类的 final 类（组装期只多分配一个操作符节点），典型如 map（`internal/operators/flowable/FlowableMap.java`）：`FlowableMap extends AbstractFlowableWithUpstream`，内部 `MapSubscriber extends BasicFuseableSubscriber` 实现 `onNext`/`requestFusion`/`poll`。`Flowable` 直接实现 `java.util.concurrent.Flow.Publisher`（`core/Flowable.java:157`），对裸 `Flow.Subscriber` 自动包一层 `StrictSubscriber` 强制协议合规（`core/Flowable.java:13307-13315`）。

### 5.2 背压体系（Flowable 与请求策略）

背压三支柱：

1. **request(n) 信用传播**：Flowable 链上每个 Subscriber 负责向上游转发需求。典型消费侧如 `observeOn` 用 `AtomicLong requested` + `BackpressureHelper.add` 原子记账、预取 prefetch 后队列排水时 `requested.addAndGet(-e)` 扣减再补请求（`internal/operators/flowable/FlowableObserveOn.java:144,290,411`）。
2. **源端策略**：`core/BackpressureStrategy.java` 枚举 MISSING/ERROR/BUFFER/DROP/LATEST，对应 `internal/operators/flowable/FlowableOnBackpressure{Buffer,Drop,Error,Latest,BufferStrategy,Reduce,ReduceWith}.java` 一组策略操作符；溢出统一抛 `exceptions/MissingBackpressureException`。
3. **自然背压**（4.x 新增）：`Streamable` 消费循环调用 `Streamer.next()` 返回 `CompletionStage<Boolean>`，生产者在消费者 await 前无法推进——"producers and consumers have to wait for each other"，天然适配虚拟线程阻塞模型（`core/Streamer.java:31-70`，`README.md:109`）。`next()/current()/finish()` 生命周期 + `NEXT_TRUE/NEXT_FALSE/FINISHED` 同步快路径常量是关键设计（`core/Streamer.java:123-135`）。

### 5.3 操作符融合（macro/micro fusion）

`DESIGN.md` "Fusion" 章节是全库性能设计纲领：宏融合借 `ScalarCallable`/`Callable` 在组装期内联标量源（`just(1).map(...)` 可整体退化为常量）；微融合让相邻操作符共享同一队列，`QueueSubscription.requestFusion(mode)` 以 SYNC/ASYNC/BOUNDARY 位标志在订阅期协商，`ConditionalSubscriber.tryOnNext()` 让 filter 类操作符丢弃元素时不必回补 request。`FlowableMap` 中 `requestFusion => transitiveBoundaryFusion(mode)`、融合模式下 `onNext` 直接 `downstream.onNext(null)` 走队列 `poll()` 路径（`FlowableMap.java:57-100`），是微融合的最小样例。

### 5.4 调度器体系（schedulers/）

抽象基类 `core/Scheduler.java`：核心抽象仅 `createWorker()`（返回 `Worker implements Disposable`），外加 `scheduleDirect/schedulePeriodicallyDirect` 便捷路径。工厂类 `schedulers/Schedulers.java` 提供 computation、io、cached、trampoline、newThread、single、`from(Executor)`（可配 interruptible/fair）与 4.x 新增：

- `virtual()`：延迟初始化的 `DeferredExecutorScheduler(Executors::newVirtualThreadPerTaskExecutor, true, true)`，每任务一个虚拟线程（`schedulers/Schedulers.java:82,96,241`）；
- `createParallel(ParallelSchedulerConfig)`：record 配置的并行调度器（`Schedulers.java:662-676`）。

虚拟线程还深度接入 Flowable 创建 API：`Flowable.virtualCreate(VirtualGenerator)` 默认绑定 `Schedulers.virtual()`，配合 `internal/virtual/FlowableVirtualCreateExecutor.java` 等执行器实现挂起/恢复式生成（`core/Flowable.java:17863-17911`）。

## 6. 构建与运行

来自 `build.gradle`、`.github/workflows/gradle_pr.yml` 与 README（Java 26 环境）：

```bash
# 构建（含检查）
./gradlew build --stacktrace
# 全量测试
./gradlew test
# 只跑源码规范元测试（CI 的第一步）
./gradlew test --tests "io.reactivex.rxjava4.validators.*"
# 微基准（JMH 插件）
./gradlew jmh
# 源码/文档 jar、OSGi bundle、发布由 vanniktech 插件承担
```

依赖坐标：`implementation "io.reactivex.rxjava4:rxjava:4.x.y"`（3.x 用户为 `io.reactivex.rxjava3:rxjava`，`README.md:42,579`）。快照版本 4.0.0-SNAPSHOT 需 sonatype snapshots 仓（`README.md:607`）。未运行构建（按任务规范仅静态阅读）。

## 7. 测试与工程质量

- 测试规模：1079 个测试 Java 文件，与主代码 945 个几乎 1:1；`src/test/resources/junit-platform.properties` 默认给每个测试方法 30s 超时（`disabled_on_debug`）。
- 测试分层：JUnit Jupiter 单测（906 个文件）+ Reactive Streams TCK 128 个 TestNG 类（`tck/`，覆盖各操作符与 Processor 的协议合规）+ `internal/` 与按类型分包的功能测试 + `src/jmh/` 64 个性能基准。
- 特色"元测试"：`validators/` 目录用 Java 测试代码扫描库自身源码——检查 license 头（`CheckAndFixLicenseHeadersTest`）、强制 `throwIfFatal`（`CheckCatchThrowIfFatalTest`）、内部类命名（`CheckInternalWrongNamingTest`）、javadoc 代码链接/措辞/注解（CheckJavadoc* 系列十几项）、甚至把匿名类改 lambda 的检查（`CheckAnonClassForLambdaTest`）。把代码规范做成可执行测试，是本仓库工程质量最鲜明的特征。
- CI（`.github/workflows/`，12 个工作流）：PR 工作流固定 JDK 26 zulu、先跑 validators 再 build 并发布测试报告（`gradle_pr.yml`）；另有 branch/release/snapshot/preview-JDK 矩阵、gradle-wrapper 校验、OpenSSF Scorecard（`scorecard.yml`）、自研"entropy-beauty-scan"代码熵扫描（`compute-entropy.py`）、Discord/X 发布通告。codecov.yml 在列。

## 8. 社区与维护状态

- 活跃度：默认分支 4.x 最近 push 2026-11-03（gh api），几乎每日有提交；issue 极少积压（open_issues 12）。4.0.0-alpha 系列约每 1-2 周发一版（v4.0.0-alpha-17 至 21 时间跨度 2026-06-29 至 2026-08-05，gh api releases），正式版 4.0.0 定档 2026-11-30（`README.md:3`）。
- 版本线并行：仓库保留 1.x/2.x/3.x/4.x 四条分支；CONTRIBUTING.md 规定新代码只进 4.x，3.x 仅重大修复，1.x/2.x 封禁。
- 贡献者结构：贡献者 100+（gh api contributors 分页封顶），长期由 David Karnok（akarnokd）一人主导——POM 开发者字段、README 4.x 特性列表第一人称（"now that I have the master key"）均指向此。BDFL 模式明显。
- 治理风格（CONTRIBUTING.md）：明确拒绝低质量/灌水 PR，要求 AI 辅助贡献必须披露，违规者公开封禁；社区治理非常强势，但开源 14 年的成熟项目靠此保持代码纯度（"未确认"项：具体 PR 频次与 Slack/Discord 社区规模未在源码内找到证据）。

## 9. 亮点与风险

亮点：

1. 教科书级的协议工程：RS 契约 + 双级融合（`DESIGN.md`）+ StrictSubscriber 强制合规 + 128 个 TCK 类，正确性有体系化保障。
2. 把工程规范做成可执行测试（`validators/` 元测试），4.x 又引入 Cleaner 泄漏检测与熵扫描 CI，质量防线多层冗余。
3. 背压设计三态完备（Observable 推 / Flowable 信用 / Streamable 虚拟线程拉），同一问题给出三种成本模型，教学与实战价值都高。
4. 零运行时依赖 + JPMS/OSGi 双模块化 + 多版本 JDK 兼容策略，嵌入式/Android/服务端通吃。
5. 虚拟线程时代的自我革新：`Streamable`/`Streamer` + `Schedulers.virtual()` 把十年前的推模型与 JDK 21+ 阻塞式并发缝合（`core/Streamer.java`）。

风险/局限：

1. 4.0 是破坏性重写：包名 `io.reactivex.rxjava4`、基线 Java 26，Android 与存量用户的迁移成本高；README 明示 3.x 一年后停止跟进。
2. Observable/Flowable/Streamable 三套实现并存，操作符约 1.5-2 倍重复劳动，单维护者模式下可持续性承压（CONTRIBUTING.md 自己也承认需 PR 同步多处）。
3. 单人 BDFL + 强硬治理，外部贡献门槛极高，bus factor 集中于 akarnokd。
4. 每操作符一个类的架构带来数千级类数量与较大 jar，冷启动与 R8 裁剪对移动端仍有成本（定性判断，未实测）。

## 10. 总结评价

RxJava 是 JVM 响应式编程的开山与常青树：48k 星的背后是十年间定义了"流 + 操作符 + 背压"这套行业语言，再由 Reactor、Project Loom 生态反复引用与追赶。当前 4.x 分支正处在发布前夜（4.0.0 定档 2026-11-30），它用虚拟线程与 `Streamable` 主动回应了"结构化并发会不会杀死响应式库"的时代之问。适合所有做异步/流式系统的 Java 工程师把它当生产库或参照系；对源码学习者，它可能是 JVM 生态中并发正确性工程的最佳教材——建议从 `DESIGN.md` 入门，再按 `core/Observable.java` 的 `subscribe` 骨架、`FlowableMap` 的融合样板、`FlowableObserveOn` 的请求记账、`core/Streamer.java` 的拉模型四步精读。
