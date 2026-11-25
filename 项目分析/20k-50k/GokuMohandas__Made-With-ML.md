# GokuMohandas/Made-With-ML 项目分析

> 星数 49808（快照 2026-10-24）｜主语言 Jupyter Notebook｜档位 20k-50k｜分析日期 2026-11-04

## 1. 项目速览

一句话定位：**Made With ML 是一套面向生产环境的 MLOps 实战课程（madewithml.com），本仓库是其官方代码配套——用一个"ML 项目自动打标签"的端到端案例，完整演示从实验（设计+开发）到生产（部署+迭代）的全流程工程实践。**

| 项目 | 内容 |
| --- | --- |
| 名称 | GokuMohandas/Made-With-ML |
| 星数 | 49808（快照 2026-10-24；gh api 2026-11-04 实测 49919） |
| 主语言 | Jupyter Notebook / Python（25 个 .py、2 个 .ipynb） |
| 许可证 | MIT（LICENSE，"Copyright (c) 2023 Made With ML"） |
| 创建时间 | 2018-11-05（gh api `created_at`） |
| 最近推送 | 2026-03-04（gh api `pushed_at`；同日发布 release v1.1.0） |
| 主页 | https://madewithml.com（README.md；gh api `homepage`） |
| 规模 | 源码 tarball 共 56 个文件；核心包约 1421 行 Python |
| Forks / Watch | 7817 forks、1262 watchers（gh api） |

## 2. 项目是什么，解决什么问题

绝大多数机器学习教程止步于"在 notebook 里把模型跑通"，而真实工业界需要的是可靠的、可迭代的**生产级 ML 系统**。Made With ML 的核心主张（README.md "Overview"）是：从第一性原理出发理解每个 ML 概念，在开发过程中贯穿软件工程最佳实践（测试、文档、日志、版本化），用 Python 可扩展地完成数据、训练、调参、推理四类负载，最后通过 CI/CD 无缝地从开发走向生产。

课程载体是一条完整的业务流水线：对 ML 项目做**多分类打标签**（输入 title + description，输出 computer-vision / natural-language-processing / mlops / other 四类之一，见 `datasets/tags.csv` 与 `tests/data/test_dataset.py` 中的标签集合）。数据集来自真实抓取的 GitHub ML 项目（`datasets/projects.csv`，约 800 行训练数据 + 208 行 holdout）。

整个课程按"workload"组织（README.md 引用的 systems-design 图）：数据摄取 → 划分 → 预处理 → 训练（分布式）→ 调参（超参搜索）→ 评估（粗/细粒度、切片、行为测试）→ 推理（批量/在线）→ 服务化 → CI/CD 持续训练与部署。学习路径分两步走：先在 `notebooks/madewithml.ipynb`（243 个 cell）里交互式理解，再把同样的逻辑重构为 `madewithml/` 包下的规范脚本（README.md "Scripts" 一节明确说明了这个 notebook → 脚本的映射关系）。

值得一提的教学设计：课程以 **Ray 生态为骨架**（Ray Data / Train / Tune / Serve），并默认提供 **Anyscale** 云集群路径（README.md "Cluster" 部分按 Local / Anyscale / 其他云三套下拉说明折叠给出），让同一份代码可以在笔记本单机或 GPU 集群上无差别运行——这正是"不换语言/不换基础设施就能 scale"这一卖点的落点。

## 3. 技术栈与依赖

来自 `requirements.txt`（全部锁定版本，Python 3.10 推荐，见 README.md）：

| 类别 | 库（版本） | 用途 |
| --- | --- | --- |
| 分布式框架 | ray[air]==2.7.0 | Ray Data（数据管道）、Ray Train（分布式训练）、Ray Tune（调参）、Ray Serve（在线服务） |
| 深度学习 | torch==2.0.0、transformers==4.28.1 | 模型实现；`allenai/scibert_scivocab_uncased` 预训练底座（`madewithml/models.py`） |
| 实验跟踪 | mlflow==2.3.1 | 实验/模型注册表，作为 Ray Train 回调接入（`madewithml/train.py`） |
| 数据科学 | pandas==2.0.1、scikit-learn==1.2.2、numpy==1.24.3、nltk | 数据处理与指标计算 |
| 数据质量 | great-expectations==0.16.5 | 数据集完整性校验（`tests/data/test_dataset.py`） |
| 弱监督/切片 | snorkel==0.9.9 | 切片函数（slice metrics）评估（`madewithml/evaluate.py`） |
| 超参搜索 | hyperopt==0.2.7 | Ray Tune 的搜索后端（`madewithml/tune.py`） |
| 服务/CLI | fastapi==0.95.2、typer==0.9.0 | Ray Serve 的 FastAPI ingress；各 workload 的命令行入口 |
| 部署 | anyscale==0.5.131 | Anyscale Job/Service 提交（`deploy/`） |
| 文档 | mkdocs==1.4.2、mkdocstrings==0.21.2 | API 文档自动生成，部署到 madewithml.com |
| 质量 | black / flake8 / isort / pyupgrade / pre-commit / pytest / pytest-cov | 风格与测试（`requirements.txt`、`pyproject.toml`、`Makefile`） |

技术栈整体冻结在 2023 年中期的版本组合（ray 2.7.0、torch 2.0.0、mlflow 2.3.1），与课程 cohort 时代一致（`requirements.txt`）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `madewithml/` | 生产级 Python 包（10 个文件，约 1421 行）：config/data/models/train/tune/evaluate/predict/serve/utils |
| `notebooks/` | 课程主 notebook `madewithml.ipynb`（243 cells）+ `benchmarks.ipynb`（54 cells）+ 清理脚本 |
| `tests/` | 三层测试：`code/`（单元测试）、`data/`（数据质量）、`model/`（行为测试） |
| `datasets/` | 4 个 CSV：dataset.csv（813 行）、holdout.csv（208 行）、projects.csv、tags.csv |
| `deploy/` | Anyscale 部署配置：`jobs/`（workloads.yaml + workloads.sh）、`services/`（serve_model.yaml/py） |
| `.github/workflows/` | 3 条 CI/CD 流水线 + `json_to_md.py` 结果转 Markdown 工具 |
| `docs/madewithml/` | 8 个 mkdocstrings 桩文件（如 `::: madewithml.data`），自动生成 API 文档 |
| `mkdocs.yml` | 文档站点配置，nav 覆盖全部 8 个模块 |
| `pyproject.toml` / `Makefile` | black/isort/flake8/pytest/coverage 配置；style/clean 任务 |
| `README.md` | 课程入口、环境搭建、全部 workload 命令、生产部署与 CI/CD 说明（约 540 行） |

组织方式评述：这是典型的**"课程站点 + 单包应用"双仓体**。教学叙事放在 madewithml.com 与 notebook 里，仓库本身保持极小（56 个文件）；`madewithml/` 是一个单层平铺包，按 workload 垂直切分模块而非按技术分层，配合 Typer 让每个模块既是可 import 的库又是 CLI 入口（`madewithml/train.py` 末尾 `app()`）。没有 submodule、没有 monorepo 工具链，复杂度被刻意压到教学可承载的最低限度。

## 5. 核心架构与关键实现

### 5.1 模型层：极简微调头（`madewithml/models.py`）

模型就是 SciBERT + dropout + 一个线性层，刻意保持简单以突出工程而非建模：

```python
class FinetunedLLM(nn.Module):
    def __init__(self, llm, dropout_p, embedding_dim, num_classes):
        super(FinetunedLLM, self).__init__()
        self.llm = llm
        self.dropout = torch.nn.Dropout(dropout_p)
        self.fc1 = torch.nn.Linear(embedding_dim, num_classes)

    def forward(self, batch):
        ids, masks = batch["ids"], batch["masks"]
        seq, pool = self.llm(input_ids=ids, attention_mask=masks)
        z = self.dropout(pool)
        z = self.fc1(z)
        return z
```

`save/load` 把超参存进 `args.json`、权重存 `model.pt`（`madewithml/models.py`），保证 checkpoint 自描述、可从 MLflow 任意 run 还原。

### 5.2 训练层：Ray TorchTrainer 分布式训练（`madewithml/train.py`）

`train_loop_per_worker` 是每个 worker 执行的循环：数据 shard → `train.torch.prepare_model`（自动 DDP）→ BCEWithLogitsLoss + Adam + ReduceLROnPlateau → 每 epoch 按 val_loss 保留最优 checkpoint 并 `train.report`：

```python
for epoch in range(num_epochs):
    train_loss = train_step(train_ds, batch_size_per_worker, model, num_classes, loss_fn, optimizer)
    val_loss, _, _ = eval_step(val_ds, batch_size_per_worker, model, num_classes, loss_fn, optimizer)
    scheduler.step(val_loss)
    with tempfile.TemporaryDirectory() as dp:
        ...
        checkpoint = Checkpoint.from_directory(dp)
        train.report(metrics, checkpoint=checkpoint)
```

外层 `train_model` CLI 组装 ScalingConfig（worker 数/CPU/GPU 全部参数化，同一命令在笔记本和集群上通用）、CheckpointConfig（按 val_loss 保留 1 个）、MLflowLoggerCallback 与 TorchTrainer（`madewithml/train.py`）。

### 5.3 数据层：分布式分层划分（`madewithml/data.py`）

`stratify_split` 展示了 Ray Data 的 map_groups 用法——按标签分组后在每组的 pandas DataFrame 上调用 sklearn 的 `train_test_split`，实现分布式环境下的分层抽样，保证各 split 类别比例一致（`madewithml/data.py`）：

```python
grouped = ds.groupby(stratify).map_groups(_add_split, batch_format="pandas")
train_ds = grouped.map_batches(_filter_split, fn_kwargs={"split": "train"}, batch_format="pandas")
test_ds = grouped.map_batches(_filter_split, fn_kwargs={"split": "test"}, batch_format="pandas")
```

`CustomPreprocessor` 实现 fit/transform 协议（`fit` 学习 class→index 映射，`transform` 做 title+description 拼接、正则清洗、BertTokenizer 编码），预处理器随 checkpoint 一起序列化，推理时从 checkpoint 还原（`madewithml/predict.py` 的 `TorchPredictor.from_checkpoint`）。

### 5.4 服务层：Ray Serve + FastAPI（`madewithml/serve.py`）

在线服务用 `@serve.deployment` + `@serve.ingress(app)` 把 FastAPI 应用挂到 Ray actor 上，提供 `/`、`/run_id/`、`/evaluate/`、`/predict/` 四个端点，并演示了"预测后处理"自定义逻辑——置信度低于 0.9 归入 other 类（`madewithml/serve.py`）：

```python
@serve.deployment(num_replicas="1", ray_actor_options={"num_cpus": 8, "num_gpus": 0})
@serve.ingress(app)
class ModelDeployment:
    def __init__(self, run_id: str, threshold: int = 0.9):
        mlflow.set_tracking_uri(MLFLOW_TRACKING_URI)
        best_checkpoint = predict.get_best_checkpoint(run_id=run_id)
        self.predictor = predict.TorchPredictor.from_checkpoint(best_checkpoint)
```

生产入口 `deploy/services/serve_model.py` 在启动时从 S3 拉取 mlflow 注册表与结果文件、读取 `run_id.txt` 后 `ModelDeployment.bind(...)`，实现"训练产物 → 服务"的解耦衔接。

### 5.5 CI/CD：持续训练与持续部署（`.github/workflows/`）

这是课程最有辨识度的部分。`workloads.yaml`（PR → main 触发）：GitHub Actions 用 OIDC 角色拿 AWS 凭证 → `anyscale jobs submit deploy/jobs/workloads.yaml`（其 entrypoint `deploy/jobs/workloads.sh` 依次跑数据测试 → 代码测试 → 训练 → 评估）→ 从 S3 拉回结果 → `json_to_md.py` 转 Markdown → 用 `actions-comment-pull-request` 把训练/评估结果**直接评论在 PR 上**。`serve.yaml`（push → main 触发）则执行 `anyscale service rollout deploy/services/serve_model.yaml` 上线新模型——"merge 即上线"的完整闭环（`.github/workflows/workloads.yaml`、`serve.yaml`、`deploy/jobs/workloads.sh`）。

## 6. 构建与运行

以下命令均来自 README.md 与 Makefile，为课程实际使用的工作流（本分析仅静态阅读，未执行）：

```bash
# 环境搭建（README.md "Set up"，推荐 Python 3.10）
export PYTHONPATH=$PYTHONPATH:$PWD
python3 -m venv venv && source venv/bin/activate
python3 -m pip install -r requirements.txt
pre-commit install && pre-commit autoupdate

# 交互式课程 notebook
jupyter lab notebooks/madewithml.ipynb

# 分布式训练（num-workers/cpu/gpu 按机器调整）
python madewithml/train.py --experiment-name llm --dataset-loc $DATASET_LOC \
    --train-loop-config '{"dropout_p": 0.5, "lr": 1e-4}' \
    --num-workers 1 --cpu-per-worker 3 --gpu-per-worker 1 --num-epochs 10 --batch-size 256

# 超参搜索 / 评估 / 批量推理
python madewithml/tune.py --experiment-name llm --initial-params "$INITIAL_PARAMS" --num-runs 2 ...
python madewithml/evaluate.py --run-id $RUN_ID --dataset-loc $HOLDOUT_LOC ...
python madewithml/predict.py predict --run-id $RUN_ID --title ... --description ...

# 在线服务（Ray Serve）
ray start --head
python madewithml/serve.py --run_id $RUN_ID   # POST http://127.0.0.1:8000/predict

# 测试（三层，README.md "Testing"）
python3 -m pytest tests/code --verbose --disable-warnings
pytest --dataset-loc=$DATASET_LOC tests/data
pytest --run-id=$RUN_ID tests/model
python3 -m pytest tests/code --cov madewithml --cov-report term --disable-warnings

# 代码风格与清理（Makefile）
make style   # black + flake8 + isort + pyupgrade
make clean
```

生产路径：`anyscale cluster-env build deploy/cluster_env.yaml` → `anyscale job submit deploy/jobs/workloads.yaml` → `anyscale service rollout -f deploy/services/serve_model.yaml`（README.md "Production"）。注意 `deploy/cluster_env.yaml`、`deploy/cluster_compute.yaml` 在 README 中被引用但 tarball 内不存在，可能经 `export-ignore` 或后续提交移除（未确认，不影响课程主线，README 提供了自建指引）。

## 7. 课程内容组织与工程质量

**收录范围与组织方式**：课程正文在 madewithml.com（按 systems-design / MLOps 课程页组织，README 与 notebook 内多处链接到具体章节如 `courses/mlops/data-engineering/`、`courses/mlops/monitoring/`）。仓库内的教学产物为两本 notebook：`notebooks/madewithml.ipynb`（243 个 cell：80 个 markdown + 163 个 code，标题从 Data ingestion 一路推进到 Online inference 与 Custom logic）与 `notebooks/benchmarks.ipynb`（54 个 cell，性能基准）。notebook → 脚本的对应关系在 README.md "Scripts" 中以目录树形式明确标注，`Makefile` 的 `clean` 目标还包含 `python notebooks/clear_cell_nums.py` 用于清理 cell 编号后提交。

**条目规模**：核心生产代码约 1421 行（`madewithml/*.py`，wc -l 统计）；数据集约 800 个训练样本、208 个 holdout 样本、4 个标签类别（`datasets/*.csv` 行数统计）；文档 API 覆盖全部 8 个模块（`mkdocs.yml` nav）。

**工程质量**：对一个教学仓库而言工程化程度异常高——
- 三层测试体系：`tests/code/`（6 个文件的单元测试，覆盖 data/train/tune/predict/utils）、`tests/data/test_dataset.py`（Great Expectations 校验 schema、标签集合、title+description 唯一性防泄漏、空值、id 唯一性）、`tests/model/test_behavioral.py`（行为测试三件套：INVariance 动词替换不变性、DIRectional 方向期望、MFT 最小功能测试）；
- `# pragma: no cover` 被有意识地用于标注"经由 workload 端到端测试覆盖"的代码路径（`madewithml/train.py`），coverage 配置排除 evaluate/serve（`pyproject.toml` `[tool.coverage.run]`）；
- 全量 pre-commit + black(150 列) / flake8 / isort / pyupgrade（`pyproject.toml`、`requirements.txt`）。

**更新机制**：课程内容更新走 madewithml.com；仓库代码变更经 PR → Anyscale 真实训练+评估 → PR 评论结果 → merge → 自动上线 的自动化流程（`.github/workflows/workloads.yaml`、`serve.yaml`），文档由 `documentation.yaml` 在 push main 后 `mkdocs gh-deploy` 自动发布。

## 8. 更新机制与维护状态

- **最近推送**：2026-03-04（gh api `pushed_at`），同日发布唯一 release v1.1.0（gh api releases），与本任务清单"最后推送 2026-03"一致；仓库 `updated_at` 2026-11-03 说明仍有持续外部互动（star/issue）。
- **贡献者结构**：GitHub contributors API 仅返回 1 人（GokuMohandas，19 次贡献），主分支历史曾因课程大规模重构被整理过（据 contributors 数据推断，未确认细节）；实际社区贡献更多以 issue/PR 讨论形式存在（open issues 25 个）。
- **版本节奏**：依赖版本锁定在 2023 年中期组合（`requirements.txt`），2026 年的推送主要是课程/元数据层面的维护（如 release v1.1.0）。代码快照稳定、更新频率低，但**并未死掉**——课程站 madewithml.com 与 cohort 报名入口仍在 README 中活跃维护。
- **风险点**：本任务"清单未填简介"，其官方定位以 README 与站点为准：MLOps 课程，40K+ 订阅者（README.md 徽章 "Subscribe-40K"）。

## 9. 生态位、亮点与风险

**生态位**：Made With ML 长期位居 GitHub ML 主题仓库头部（README.md 自述 "Among the top ML repositories"），处于"ML 教程 → 生产 MLOps"这一最大痛点的交叉口。它是 Ray/Anyscale 生态最知名的**教学样板项目**（topics 含 ray、anyscale 相关关键词，gh api topics；deploy/ 全套 Anyscale 配置），与 full-stack-deep-learning、MLOps-basics 等同类课程相比，它的差异化在于**代码与 CI/CD 是真实可跑的完整闭环**而非幻灯片。

**为什么能涨到 5 万星**（基于仓库事实的推断）：其一，免费、成体系、第一性原理式教学，覆盖从 notebook 到生产的完整路径，填补了大学课程与工业界之间的空白（README.md "Audience" 明确锚定开发者/毕业生/产品经理三类人群）；其二，作者 Goku Mohandas 是 MLOps 社区知名教育者，课程长期迭代并直播授课积累了口碑（40K+ 订阅徽章）；其三，仓库即文档——README 本身就是一份可操作的 MLOps runbook，clone 下来照做即可跑通，这种"可执行课程"的形态极易被传播与收藏。

**亮点**：
1. 完整真实的 CI/CD 闭环：PR 上自动训练、评估并把结果评论回 PR，merge 自动上线（`.github/workflows/workloads.yaml`、`serve.yaml`）——教学与生产实践罕见地无缝统一；
2. 教学分层设计：notebook（直觉）→ 脚本（工程）双轨对照，同一逻辑两种形态（`notebooks/madewithml.ipynb` 与 `madewithml/` 包）；
3. 测试金字塔完整且现代：数据质量（Great Expectations）+ 单元 + 行为测试（invariance/directional/MFT）三层齐备（`tests/`）；
4. 可移植性强：ScalingConfig 参数化 + Ray 抽象，同一命令在笔记本与 GPU 集群通用（`madewithml/train.py`、README "Set up"）；
5. 实验可复现：MLflow 回调 + 自描述 checkpoint（args.json + model.pt）+ 预处理器随模型保存（`madewithml/models.py`、`predict.py`）。

**风险/局限**：
1. 依赖栈冻结于 2023 年中期（ray 2.7.0 / torch 2.0.0 / mlflow 2.3.1，`requirements.txt`），且 ray[air] 的 API（如 `ray.air`、`TorchCheckpoint`）在新版 Ray 中已大幅变动，按当前文档直接装新版依赖大概率跑不通，需严格锁版本；
2. 数据集与任务极小（约 800 样本、4 分类），是为教学牺牲的真实感——不要照搬其规模设定到生产；
3. 重度绑定 Anyscale 商业云与 S3（`deploy/`、`.github/workflows/`），无 Anyscale 账号的读者只能走 README 中较简略的本地/自建集群路径，生产章节体验会打折；
4. 课程正文不在仓库内（在 madewithml.com），纯看仓库只能得到代码骨架；`deploy/cluster_env.yaml` 等个别 README 引用的文件在 tarball 中缺失（未确认是否 export-ignore 所致）。

## 10. 总结评价

总评：Made With ML 是 MLOps 教育领域公认的高标杆项目——它把"如何在真实工程约束下交付可靠 ML 系统"拆解成一条可以亲手跑通的端到端流水线，并用 notebook→脚本→测试→CI/CD 的递进结构让学习者自然完成从数据科学家到 ML 工程师的视角转换。5 万星并非来自炫技建模，而是来自"课程免费且成体系、代码真实且可跑、工程实践现代且完整"三者叠加；仓库代码 2026 年 3 月后趋于稳定，但其代表的方法论（Ray 生态、持续训练、行为测试、PR 评论式实验汇报）至今仍是业界主流范式。

适合谁：想从"会训模型"跨到"会交付模型"的软件工程师与数据科学家（对照课程站逐节复现收益最大）；想为团队搭建 ML 平台化流程、寻找 CI/CD 参考实现的工程负责人；Ray 生态的入门者（这是最好的官方级示例之一）。不适合：期待大规模真实数据集或前沿模型架构的读者；不愿注册 Anyscale 且无法自建 Ray 集群的读者（生产章节体验受限）。读源码的建议顺序：README → `notebooks/madewithml.ipynb` → `madewithml/{data,train,serve}.py` → `.github/workflows/workloads.yaml`，这条线正好复刻课程的"实验到生产"叙事。
