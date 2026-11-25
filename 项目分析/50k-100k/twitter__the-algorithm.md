# twitter/the-algorithm 项目分析

> 星数 73979（快照 2026-10-24）｜主语言 Scala｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

X（原 Twitter）推荐算法的官方开源仓库：驱动 For You 时间线、推荐通知等产品表面的推荐系统服务与离线作业集合，2023 年 3 月由 Twitter 开源，是业界公开的最大规模工业级推荐系统源码。

| 项目 | 内容 |
|---|---|
| 名称 | twitter/the-algorithm |
| 星数 | 73,979（快照 2026-10-24） |
| 主语言 | Scala（约 44 万行），辅以 Java / Python / Rust |
| 许可证 | AGPL-3.0（`COPYING`） |
| 创建时间 | 2023-03-27 |
| 最近活跃 | 2025-09（gh api `pushed_at=2025-09-08`，tarball 内文件 mtime 均为 2025-09-04），此后维护停滞 |
| 构建系统 | Bazel（仅 BUILD 文件，无顶层 WORKSPACE） |

## 2. 项目是什么，解决什么问题

这是 X 内部推荐 monorepo 对外开源的子集，覆盖"从约 10 亿帖中筛出几百条时间线"的完整链路：候选召回 → 特征注入 → 打分排序 → 过滤混排 → 服务输出（`README.md`）。它不是单一算法，而是一组相互独立的 Finagle/Thrift 微服务加一批离线计算作业，通过 RPC 与 Kafka 流水线拼装成完整系统。

仓库定位偏"透明化展示"而非"可复现运行"：README 明确说明仅包含 Bazel BUILD 文件而不含顶层 BUILD/WORKSPACE，无法开箱构建（`README.md`）。Heavy Ranker 神经网络模型本体放在姊妹仓库 `twitter/the-algorithm-ml`，本仓库只含调用侧代码；部分组件（如 visibilitylib）README 自述"正在审查重建、部分代码已移除、注释已脱敏"（`visibilitylib/README.md`）。因此社区主要将其作为工业推荐架构的参考读物，而非可直接部署的系统。

推荐信号体系有专门文档 `RETREIVAL_SIGNALS.md`，枚举了 18 类用户行为信号（关注、点赞、回复、视频观看、举报等）在 USS、SimClusters、TwHIN、UTEG、FRS、Light Ranking 各组件中作为特征/标签的使用矩阵，是理解全仓库数据流的一把钥匙。

除 For You 时间线外，仓库还包含第二个产品表面"推荐通知"：pushservice 以 light ranker（从海量候选中预筛高相关推文）与 heavy ranker（多任务模型预测打开/互动概率）两段式模型驱动推送决策（`README.md` "Recommended Notifications" 节、`pushservice/src/main/python/models/`）。这意味着同一个仓库内可以对照两套共享底层信号但管线独立的推荐产品实现。

## 3. 技术栈与依赖

- **Scala**（4880 个文件，约 440k 行）：主体服务语言，基于 Twitter 自研技术栈——Finagle RPC、Thrift（114 个 .thrift IDL）、Guice 依赖注入、Stitch 并行请求编排、Util/Futures。
- **Java**（1043 个文件）：主要是 Earlybird 搜索索引（基于 Apache Lucene 的实时检索）与 ANN 近邻检索（FAISS JNI 绑定，`ann/src/main/java/com/twitter/ann/faiss/`）。
- **Python**（180 个文件）：Light Ranker 训练管线（DeepBird，`src/python/twitter/deepbird/projects/timelines/scripts/models/earlybird/`）与 legacy ML 框架 twml（基于 TensorFlow v1，`twml/README.md` 自述"已停止开发，仅为训练 light ranker 保留"）。
- **Rust**（30 个文件）：navi 高性能模型推理服务，含 ONNX 与 TF 模型加载（`navi/navi/src/onnx_model.rs`、`tf_model.rs`）。
- **Bazel**：699 个 BUILD 文件描述构建目标，但无顶层 WORKSPACE，仅作内部构建映射参考。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| `home-mixer/` | For You 时间线主服务（排序/过滤/混排编排，基于 product-mixer） |
| `product-mixer/` | 内容流构建框架（component-library + core 管道引擎） |
| `tweet-mixer/` | 站外推文候选获取协调层，22 个候选源 |
| `cr-mixer/` | 候选生成服务（相似度引擎、SimClusters/TwHIN ANN 检索） |
| `follow-recommendations-service/` | 关注推荐（WTF 模块）与 FutureGraph 推文推荐 |
| `src/java/com/twitter/search/` | Earlybird 推文搜索索引（站内候选主源） |
| `src/scala/com/twitter/recos/` | GraphJet 系内存图服务（UTEG 等 5 个图服务） |
| `tweetypie/` | 推文数据读写核心服务 |
| `unified-user-actions/`、`user-signal-service/` | 实时用户行为流、显式/隐式信号平台 |
| `simclusters-ann/`、`representation-manager/`、`representation-scorer/` | 社区 embedding 检索与向量打分服务 |
| `trust_and_safety_models/` | NSFW/滥用内容检测模型 |
| `visibilitylib/` | 内容可见性过滤规则引擎（部分代码移除） |
| `timelineranker/` | 遗留排序服务（Earlybird+UTEG 结果合并） |
| `pushservice/` | 推荐通知服务（含 push 专用 light/heavy ranker） |
| `navi/`、`twml/`、`ann/` | Rust 推理框架、TF v1 训练框架、FAISS ANN |
| `src/python/twitter/deepbird/` | Light Ranker 训练代码 |

组织方式是典型的"内部 monorepo 抽取子集"：每个服务自成一棵 Maven 风格目录树（`server/src/main/scala/...`），服务间只以 thrift IDL 契约耦合；各服务结构高度同构（candidate_pipeline / functional_component / model / param / module），框架复用度很高。另有 `science/`（实验分析 notebooks）、`timelines/`（聚合特征框架等遗留代码）、`src/java/com/twitter/search/`（Java 大模块）等跨层目录。整体是"多服务松耦合 + 服务内强框架约束"的混合体：读单个服务要顺着管道抽象走，读全局要靠 thrift 契约和 README 的架构图（`docs/system-diagram.png`）。

## 5. 核心架构与关键实现

### 5.1 候选召回链路（Candidate Sources）

For You 的推文候选由四条并行管道汇成（`home-mixer/README.md`）：

1. **站内（In-Network，约占 50%）**：Earlybird 搜索索引。基于 Lucene 的分布式倒排索引，分 realtime（近 7 天）、protected、archive 三集群多分区，以 userId 查询粉丝关注对象的近期推文（`src/java/com/twitter/search/README.md`）。home-mixer 侧入口为 `ScoredTweetsInNetworkCandidatePipelineConfig.scala`，经 TimelineRanker 的 Recycled 源取回轻排后的站内候选。
2. **UTEG 社交图召回**：基于 GraphJet 内存图引擎（Vldb'16 论文）的协同过滤——以用户的加权关注图为输入做图遍历聚合，输出"XXX Liked"类站外推文；有状态服务，Kafka 流摄入，内存只保留 24-48 小时互动（`src/scala/com/twitter/recos/user_tweet_entity_graph/README.md`）。
3. **tweet-mixer 站外协调层**：聚合 22 个底层候选源，包括 SimClusters ANN（社区向量近邻）、TwHIN ANN（知识图谱向量）、内容/文本 embedding ANN、热门话题推文、USS 信号服务等（`tweet-mixer/server/src/main/scala/com/twitter/tweet_mixer/candidate_source/`）。
4. **FRS 关注推荐**：账号级召回，其产出的作者推文以"FutureGraph"形式进入 For You（`follow-recommendations-service/README.md`）。

各管道用 Gate（前置开关，如 `MinCachedTweetsGate`）、Filter、Feature Hydrator 三类功能组件串联，配置集中在 `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/candidate_pipeline/`。

### 5.2 排序链路（Light Ranker 与 Heavy Ranker）

两段式排序：Light Ranker 是轻量模型，内嵌在 Earlybird 内做初排（训练代码 `src/python/twitter/deepbird/projects/timelines/scripts/models/earlybird/train.py`，依赖已停维护的 twml/TF v1 框架）；Heavy Ranker 是多任务神经网络主排模型，**模型本体在姊妹仓库 the-algorithm-ml，本仓库只含服务调用侧**。调用实现见 `home-mixer/.../scorer/NaviModelScorer.scala`：

```scala
case class NaviModelScorer @Inject() (
  predictionGRPCService: PredictionGRPCService, ...) extends Scorer[...] {
  override val identifier: ScorerIdentifier = ScorerIdentifier("NaviModel")
  private val RequestBatchSize = 64
  private val ModelId = "Home"
  // candidates/query 转 DataRecord 后经 gRPC 批量预测，取回 PredictedScoreFeatures
```

排序前需特征注入：约 6000 个特征、109 个 Feature Hydrator（`home-mixer/.../functional_component/feature_hydrator/`），覆盖 SimClusters/TwHIN/RealGraph 向量、实时聚合互动数、大 embedding 等（`ScoredTweetsModelScoringPipelineConfig.scala` 构造函数一口气注入 30+ hydrator）。打分之后还有一批 listwise 重排器做人群控制：`ContentExplorationListwiseRescoringProvider` 将探索类内容比例下调（系数 0.0001）、`GrokSlopScoreRescorer` 压制 AI 生成内容、`DiversityDiscountProvider` 做多样性折扣（同目录）。

### 5.3 服务编排（home-mixer / product-mixer 框架）

product-mixer 是核心抽象：管道分 Candidate / Mixer / Recommendation / Scoring / Product 五级，统一由 Gate → 候选获取 → 特征注入 → Scorer → Filter → Selector → Decorator → SideEffect 的 step 序列驱动（`product-mixer/core/src/main/scala/com/twitter/product_mixer/core/pipeline/`）。For You 的嵌套结构为：

```
ForYouProductPipelineConfig                      (产品路由)
 └─ ForYouScoredTweetsMixerPipelineConfig        (混排推文/广告/用户)
     └─ ForYouScoredTweetsCandidatePipelineConfig
         └─ ScoredTweetsRecommendationPipelineConfig   (主推荐层)
             ├─ InNetwork / TweetMixer / Uteg / FRS 四条候选管道
             └─ ScoredTweetsModelScoringPipelineConfig (特征注入+heavy ranker)
```

（结构出处：`home-mixer/README.md` "Pipeline Structure" 节与 `product/for_you/` 下同名 Scala 文件。）

打分完成后进入过滤与混排：35 个过滤器做安全与体验控制（`functional_component/filter/`：GrokNsfw/Gore/Spam/Violent 四类 Grok 判定、FeedbackFatigue、PreviouslySeenTweets、AuthorDedup、SlopFilter 等），再经 Selector 组装结果。以探索内容限额为例，`ContentExplorationListwiseRescoringProvider.scala` 按 `ServedTypeFeature` 分组并按序位折扣：

```scala
override def candidateRescoringFactor(query, candidate, index): Double = {
  if (query.params(EnableContentExplorationCandidateMaxCountParam)) {
    val servedType = candidate.features.get(ServedTypeFeature)
    if (servedType == hmt.ServedType.ForYouContentExploration ||
        servedType == hmt.ServedType.ForYouContentExplorationTier2) 0.0001
    else 1.0
  } else 1.0
}
```

最终经 Domain/Transport Marshaller 生成 URT（Unified Render Timeline）结构返回客户端，并通过 Kafka SideEffect 落服务日志（`ForYouScoredTweetsMixerPipelineConfig.scala` 中注入的 `ServedCandidateFeatureKeysKafkaSideEffectBuilder` 等）。

## 6. 构建与运行

- **无法开箱运行**：README 明确"包含多数组件的 Bazel BUILD 文件，但不含顶层 BUILD 或 WORKSPACE 文件"（`README.md`）；开源的是内部 monorepo 子集，缺少内部依赖与基础设施。
- 可行的"构建"只有参考性阅读：各服务 BUILD.bazel 列出目标与依赖，`.thrift` 文件描述服务契约。
- 无公开 CI：`ci/ci.sh` 内容仅 `exit 0`，`.github/` 无 workflows。
- 本地训练参考：Light Ranker 训练脚本 `src/python/twitter/deepbird/projects/timelines/scripts/models/earlybird/train.py`（需 twml 环境，实际难以脱离内部集群运行）。

## 7. 测试与工程质量

- 测试规模极小：全仓 `src/test` 下仅 32 个 Scala 测试文件（对照 4880 个主源码文件），绝大多数内部单测未随开源发布——工程质量只能推断"内部有测试、开源部分不带"。
- 代码质量观感好：服务结构高度一致（框架强制），类注释、参数说明较完整；但 visibilitylib README 自述部分代码被移除、注释脱敏，说明存在审查裁剪。
- 大量 Decider/Param 开关（如 `EnableInNetworkParam`、`EnableContentExplorationCandidateMaxCountParam`）体现生产级渐进发布风格，社区无法使用这些开关对应的内部配置系统。

## 8. 社区与维护状态

- **活跃度**：创建于 2023-03（gh api），最后一次推送 2025-09-08，tarball 内所有文件 mtime 为 2025-09-04——自 2025 年 9 月起无任何更新，维护实质停滞（与裁撤开源团队的大背景一致，属推断）。
- **贡献者结构**：代码由 X 内部工程师以内部 monorepo 同步方式写入，issue 524 个基本无官方处理（gh api `open_issues_count=524`，未确认逐条状态）。PR 流程 README 自述"正在建设工具同步内部仓库"，社区贡献未形成闭环。
- **Release 节奏**：无 git tag/release 发布机制，只有 main 分支持续快照（未确认存在 GitHub Releases）。
- 姊妹仓库 `twitter/the-algorithm-ml`（Heavy Ranker 等 ML 项目）与 `twitter/GraphJet`（图引擎）构成生态。

## 9. 亮点与风险

**亮点**
1. 公开世界级规模推荐系统全链路：从亿级候选到最终时间线的每一步都有真实源码与 README，含信号使用矩阵（`RETREIVAL_SIGNALS.md`）与系统架构图（`docs/system-diagram.png`）。
2. product-mixer 框架的管道抽象设计优秀：Gate/Filter/Scorer/Selector 组件化 + FailOpenPolicy + 分级管道，是推荐服务工程化的范本（`product-mixer/core/`）。
3. 候选召回技术谱系完整：Lucene 系实时索引（Earlybird）、内存图遍历协同过滤（GraphJet/UTEG）、双塔向量 ANN（SimClusters/TwHIN），可对照学习。
4. 罕见的 Rust ML 推理层（navi）与 Scala 服务栈混部的工业实践样本。

**风险/局限**
1. 不可构建、不可运行：无 WORKSPACE、缺内部依赖、Heavy Ranker 模型在另一个仓库，只能当"架构参考"。
2. 维护停滞：2025-09 后无提交，issue 无人响应；twml/TF v1、timelineranker 等遗留组件自述过时。
3. 开源子集不完整：visibilitylib 部分代码移除、注释脱敏，测试几乎全部缺失，阅读时无法验证运行时行为。
4. AGPL-3.0 许可证对二次分发与商用集成有较强传染性约束。

## 10. 总结评价

the-algorithm 的价值不在"能用"，而在"能读"：它是目前公开可得的、规模最大的完整工业推荐系统源码，把候选召回、两段式排序、6000 特征注入、多级过滤混排的生产实现摊开在读者面前。代码自 2025 年 9 月起停止更新，应视作一份冻结的架构快照而非活跃项目。适合推荐系统工程师对照研究工程架构（product-mixer 管道设计与候选源组织尤其值得精读），适合研究者理解工业界如何组合协同过滤、社区向量与多任务排序；不适合期望直接部署、复现或继续开发的人。
