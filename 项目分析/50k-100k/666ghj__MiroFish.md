# 666ghj/MiroFish 项目分析

> 星数 81442（快照 2026-10-24）｜主语言 Python｜档位 50k-100k｜分析日期 2026-10-31

## 1. 项目速览

一句话定位：基于多智能体社交媒体模拟的"预测引擎"——把种子材料（新闻、政策、小说等）灌入 Zep 知识图谱，生成数千个有人设的 Agent，在模拟的 Twitter/Reddit 双平台上互动演化，最终由 ReportAgent 产出预测报告，并支持与模拟世界中的任意 Agent 对话。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | MiroFish（"简洁通用的群体智能引擎，预测万物"） | README.md |
| 星数 | 82,865（分析时 gh api 实测；快照 81,442） | gh api repos/666ghj/MiroFish |
| 主语言 | Python（后端约 2.4 万行）+ Vue 3 前端 | backend/pyproject.toml、frontend/package.json |
| 许可证 | AGPL-3.0 | LICENSE、pyproject.toml |
| 创建 / 最近 push | 2025-11-26 / 2026-10-16 | gh api |
| 贡献者 / Release | 16 人 / 3 个 release | gh api |
| 背景 | 盛大集团（Shanda）战略支持与孵化，团队招聘邮箱 mirofish@shanda.com | README.md |
| 核心依赖 | camel-ai OASIS（模拟引擎）+ Zep Cloud（记忆图谱） | README.md 致谢、pyproject.toml |

## 2. 项目是什么，解决什么问题

MiroFish 要解决的是"如何低成本预演一个未来事件的社会演化过程"。传统预测依赖数据模型或专家直觉，MiroFish 的思路是：把事件的种子信息变成一个可运行的微观社会——用 LLM 从种子文本中抽取实体与关系、构建知识图谱，再把图谱中的实体转写成有独立人格、记忆和行为逻辑的 Agent，放到模拟的社交媒体平台上互动。用户只需上传材料并用自然语言描述预测需求，其余流程全自动（README.md Overview、Workflow 节）。

整个流水线分五步（README.md Workflow，与 frontend/src/components/Step1~5 组件一一对应）：图谱构建（种子抽取 + GraphRAG）→ 环境搭建（实体关系抽取 + Persona 生成 + Agent 配置注入）→ 模拟（双平台并行、需求自动解析、时序记忆更新）→ 报告生成（ReportAgent 携带工具集与模拟后环境深度交互）→ 深度交互（与任意 Agent 对话）。

定位上有"宏观"与"微观"两副面孔：宏观是决策者的预演实验室（舆情、政策、公关推演），微观是个人的创作沙盒（README 的演示就是用《红楼梦》前八十回推演佚失结局，见 README Demo Videos 节）。项目从"武大舆情模拟"到"红楼梦结局推演"的案例说明它刻意覆盖严肃预测与娱乐推演两端。

模拟引擎没有重造轮子：直接构建在 CAMEL-AI 团队的 OASIS（Open Agent Social Interaction Simulations）之上（README.md Acknowledgments），MiroFish 自身的代码价值在"编排层"——种子→本体→图谱→人设→配置→模拟→报告的全链路自动化。

官方提供在线演示环境（mirofish-demo，README.md Live Demo 节）和 B 站完整演示视频（武大舆情模拟、红楼梦结局推演），营销与案例材料齐备，"产品先行"的姿态明显强于纯框架型开源项目。典型使用路径是：用户上传数据分析报告或小说文本（支持 PDF，PyMuPDF 解析，backend/app/utils/file_parser.py），用自然语言写明预测需求，然后等待系统自动跑完全部五步。

## 3. 技术栈与依赖

后端（backend/pyproject.toml，Python >=3.11,<3.13，uv 管理依赖）：
- Flask + flask-cors：REST API 服务，默认端口 5001（backend/run.py）
- openai SDK：统一 LLM 接入，支持任意 OpenAI 兼容 API，推荐阿里百炼 qwen-plus（.env.example）
- zep-cloud==3.25.0（精确锁版）：Zep Cloud 知识图谱/记忆服务，是图谱构建与检索的核心
- camel-oasis==0.2.5 + camel-ai==0.2.78：OASIS 社交媒体模拟框架（Twitter/Reddit 平台模拟）
- PyMuPDF：PDF 等种子文件解析（backend/app/utils/file_parser.py）
- pydantic、python-dotenv、httpx 等工具库

前端（frontend/package.json）：Vue 3.5 + Vite 7 + vue-router + vue-i18n（中英双语，locales/zh.json、en.json）+ axios；d3 用于知识图谱可视化（frontend/src/components/GraphPanel.vue）。

工程组织：根 package.json 用 concurrently 同时拉起前后端（npm run dev），Docker 镜像 python:3.11 + Node 18 合一（Dockerfile）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| backend/app/api/ | Flask 蓝图：graph.py（图谱构建）、simulation.py（模拟全生命周期，约 2900 行 30+ 端点）、report.py（报告） |
| backend/app/services/ | 核心业务：graph_builder、ontology_generator、oasis_profile_generator、simulation_config_generator、simulation_runner/manager/ipc、report_agent、zep_tools 等 12 个服务 |
| backend/app/utils/ | llm_client、openai_chat_compat、retry、file_parser、zep 客户端封装（zep/zep_paging/zep_lifecycle）等 |
| backend/scripts/ | 独立模拟脚本：run_parallel_simulation.py（双平台主循环）、run_twitter/reddit_simulation.py、action_logger.py、validate_zep_cloud_integration.py |
| backend/tests/ | 19 个 pytest 测试文件 |
| frontend/src/ | views + components（Step1~5 五步向导）+ api 封装 + i18n + d3 图谱面板 |
| locales/ | 前后端共用文案（zh/en） |
| static/、scripts/ | README 图片与 star 统计脚本 |
| .github/workflows/ | docker-image.yml（镜像发布）、update-star-history.yml（star 曲线） |

组织方式评述：单体全栈仓库（前后端同仓），后端按"api 层 / services 层 / utils 层"标准分层，模拟执行体放在 scripts/ 下以子进程方式被 SimulationRunner 拉起（backend/app/services/simulation_runner.py 的 SCRIPTS_DIR 指向 backend/scripts），进程边界清晰。services 层文件普遍偏大（report_agent.py 2619 行、simulation_runner.py 2033 行），有上帝类倾向。

## 5. 核心架构与关键实现

### 5.1 图谱构建：种子文本 → Zep 知识图谱（backend/app/services/graph_builder.py，879 行）

文本按 chunk_size=500/overlap=50 切块、以 batch_size=350 分批写入 Zep Cloud，由后台线程执行并跟踪任务进度（build_graph_async / _build_graph_worker）。本体（实体/关系类型）先由 ontology_generator.py 用 LLM 生成——提示词明确要求"实体必须是现实中真实存在的、可以在社媒上发声和互动的主体"（ontology_generator.py ONTOLOGY_SYSTEM_PROMPT），说明本体设计是为社媒模拟定制的，不是通用知识图谱。

### 5.2 人设与配置生成：图谱 → Agent 军团（oasis_profile_generator.py、simulation_config_generator.py）

oasis_profile_generator.py（1248 行）把 Zep 图谱实体转成 OASIS Agent Profile：先用 Zep 检索二次丰富节点信息，区分"个人实体"与"抽象群体实体"（文件头注释），LLM 生成详细人设，落成 twitter_profiles.csv / reddit_profiles.csv。simulation_config_generator.py（993 行）采用"分步生成"策略（时间配置 → 事件配置 → 分批 Agent 配置 → 平台配置）避免单次生成过长失败；其中 CHINA_TIMEZONE_CONFIG 按"深夜无人/晚间高峰"给 Agent 排作息（simulation_config_generator.py 第 23 行起），是模拟真实感的细节设计。

### 5.3 双平台并行模拟：OASIS 主循环（backend/scripts/run_parallel_simulation.py，1699 行）

Twitter 与 Reddit 两个模拟用 asyncio.gather 并行跑（main() 第 1585 行）；每个平台用 `oasis.make(agent_graph, platform, database_path, semaphore=30)` 建环境，sqlite 落地动作数据，semaphore 限并发防 LLM API 过载（run_twitter_simulation.py 第 1155-1160 行）。初始事件以 ManualAction(CREATE_POST) 注入，之后每轮 `env.step()` 推进，动作实时写 actions.jsonl。模拟结束后进程不退出，进入"等待命令模式"，通过 IPC 处理 interview / batch_interview / close_env 命令（main() 第 1596-1611 行）——这是"模拟后仍可采访任意 Agent"深度交互功能的基础。

### 5.4 ReportAgent：ReAct 式报告生成（backend/app/services/report_agent.py，2619 行）

ReportAgent 先 plan_outline 规划提纲，再逐节走 ReAct 循环（_generate_section_react），工具有四个（_define_tools，第 925-960 行）：insight_forge（深挖子问题）、panorama_search（全貌搜索）、quick_search（快速检索）、interview_agents（现场采访最多 10 个模拟 Agent）。前三个建立在 zep_tools.py（1734 行）对 Zep 图谱检索的封装之上，第四个则回调到 5.3 的 IPC 采访通道——报告不是事后读日志，而是"活体取证"。ReportManager 负责报告分节持久化与前端日志流（markdown + agent_log + console_log）。

### 5.5 运行编排（backend/app/services/simulation_runner.py，2033 行）

SimulationRunner 以 subprocess.Popen 拉起 5.3 的脚本，类级字典维护运行状态/进程/队列/监控线程，解析 actions.jsonl 还原每个 Agent 的动作供前端实时展示（SimulationRunState，第 110 行起），支持停止（StopPending 超时协议）与 finally 清理。API 层 simulation.py 提供 30+ 端点覆盖全生命周期，代表性端点：

| 端点 | 作用 |
|---|---|
| POST /api/simulation/create、/prepare | 建模拟、准备环境（生成 profile 与配置） |
| POST /api/simulation/start、/stop | 启动/停止模拟子进程 |
| GET /api/simulation/<id>/run-status/detail | 实时轮次、双平台进度、最近动作流 |
| GET /api/simulation/<id>/posts、/comments | 模拟产出的帖子与评论（读 sqlite） |
| POST /api/simulation/interview（含 batch/all） | 模拟后采访指定/批量/全部 Agent |

前端以五步向导组织（frontend/src/router/index.js + components/Step1GraphBuild.vue ~ Step5Interaction.vue），每步对应一个后端阶段；GraphPanel.vue 用 d3 渲染 Zep 图谱实体关系；vue-i18n 支持中英切换，后端文案亦走 locales（backend/app/utils/locale.py）。

## 6. 构建与运行

来自 README.md Quick Start 与根 package.json scripts：

```bash
cp .env.example .env          # 填 LLM_API_KEY（推荐阿里百炼 qwen-plus）与 ZEP_API_KEY
npm run setup:all             # 根+前端 npm install；后端 uv sync 自动建虚拟环境
npm run dev                   # concurrently 同时启动前后端
# 前端 http://localhost:3000 ；后端 API http://localhost:5001

# 也可分步安装/单独启动（根 package.json scripts）
npm run setup                 # 仅 Node 依赖
npm run setup:backend         # 仅 Python 依赖（backend/uv.lock 锁定）
npm run backend               # 仅启动后端
npm run frontend              # 仅启动前端

# Docker 部署（docker-compose.yml，镜像 ghcr.io/666ghj/mirofish:latest，提供南大镜像备用）
docker compose up -d          # 端口 3000/5001，挂载 ./backend/uploads
```

必填环境变量仅两项（.env.example）：LLM_API_KEY（OpenAI 兼容格式，BASE_URL/MODEL_NAME 可换任意供应商）与 ZEP_API_KEY（Zep Cloud，免费月额度可支撑简单使用）；可选 LLM_BOOST_* 三项用于给模拟主循环配第二个加速模型。

测试命令：后端 `cd backend && uv run pytest`（pytest>=8.0 + pytest-asyncio，pyproject.toml dev 组）。Python 版本要求 >=3.11,<3.13，Node >=18。README 特别提醒 LLM 消耗大，"可先进行小于 40 轮的模拟尝试"（.env.example 注释）。

## 7. 测试与工程质量

- 测试规模：backend/tests/ 19 个文件、110 个测试函数（grep 统计），根 tests/ 另有 2 个文件 28 个函数覆盖 star 统计脚本。后端测试集中在最容易坏的部分：LLM JSON 响应解析（test_openai_chat_compat.py、test_llm_json_responses.py）、本体生成归一化（test_ontology_*.py）、Zep Cloud 契约与分页（test_zep_cloud_contracts.py、test_zep_edge_paging.py、test_zep_graph_lifecycle.py）、报告工具结果消毒（test_report_tool_result_sanitizer.py）、平台 profile 字段归一化（test_platform_profiles.py）、模拟准备失败路径（test_simulation_prepare_failure.py）等，均为带 mock 的单元测试，未见端到端集成测试。另有 backend/scripts/validate_zep_cloud_integration.py（904 行）作为 Zep 集成的手动验证脚本。
- CI：只有 docker-image.yml（打 tag 触发 GHCR 多架构 amd64/arm64 发布）和 star-history 更新两个 workflow（.github/workflows/），**没有跑 pytest 的 CI workflow**——测试写了但不在流水线上强制执行。
- 代码质量观察：分层清晰、中文注释详尽、错误处理与重试封装成体系（utils/retry.py、openai_chat_compat.py 对 LLM JSON 输出做了大量防御性解析）；但 services 层单文件体量失控（最大 2878 行），前端无任何测试。

## 8. 社区与维护状态

- 活跃度：2025-11-26 创建，2026-10-16 仍有 push，约 11 个月从 0 涨到 8.2 万星，上过 Trendshift 趋势榜（README.md 第 5 行徽章）；当前 12,533 fork、206 个 open issue（gh api）。
- 节奏：3 个 release，Docker 镜像按 tag 持续发布（docker-image.yml 支持打 tag 与手动 workflow_dispatch 触发）；无 CHANGELOG 文件，版本节奏细节未确认。
- 生态位：README 致谢 OASIS 与 CAMEL-AI，属于 camel-ai 生态的上游应用层；DeepWiki 徽章（README.md 第 17 行）表明已接入第三方代码解读服务。
- 贡献者：gh api contributors 计 16 人，头部贡献结构未确认；项目背靠盛大集团商业化孵化（README 致谢 + 招聘信息），QQ 群/Discord/X/Instagram 多渠道运营，属于"公司扶持 + 开源社区"混合驱动。

## 9. 亮点与风险

亮点：
1. 全链路自动化的多智能体模拟流水线：种子文本→本体→图谱→人设→配置→模拟→报告一条龙，用户只需自然语言描述需求（frontend/src/components/Step1~5 与 api 层对应实现）。
2. "模拟后世界保持存活"的设计：模拟结束后环境不销毁，通过 IPC 支持 interview/batch_interview（run_parallel_simulation.py 第 1596-1611 行），报告 Agent 能现场采访 Agent，交互性远超同类"跑完出日志"的模拟框架。
3. 善用外部成熟组件而非重造轮子：模拟交给 OASIS、记忆图谱交给 Zep，自身专注编排层，代码边界清楚。
4. 工程防御性强：LLM JSON 解析、Zep 重试/分页/生命周期、报告工具结果消毒都有专门模块与测试（utils/openai_chat_compat.py、utils/retry.py、tests/test_zep_*）。
5. 双语 UI + 中文作息细节（CHINA_TIMEZONE_CONFIG）体现对中文场景的认真适配；Windows 兼容有专门处理（backend/run.py 第 9-16 行 UTF-8 控制台修复）。

风险/局限：
1. 硬依赖 Zep Cloud SaaS（zep-cloud 锁版 3.25.0），无自托管图谱替代路径，服务变更或配额即断粮；LLM + Zep 双重 API 成本高，README 自述"消耗较大"。
2. 无测试 CI：110 个测试函数存在但不进流水线，回归保障靠自觉。
3. 单文件体量过大（api/simulation.py 2878 行、report_agent.py 2619 行），可维护性随功能膨胀会持续恶化。
4. AGPL-3.0 对商用集成限制严格；且"预测"结果的可靠性无法验证——本质是 LLM 驱动的情景推演而非统计预测，需防止过度宣传使用。

## 10. 总结评价

MiroFish 是 2025-2026 年多智能体模拟赛道的明星作品：它没有发明新的 agent 框架，而是把 OASIS（模拟）、Zep（记忆图谱）、OpenAI 兼容 LLM 三块积木拼成了一条"上传材料→自动生成数字社会→推演出报告→还能跟里面的虚拟市民聊天"的完整产品链路，产品化完成度和开箱体验是其八万星的核心原因。代码层面分层规范、防御性编程扎实，但 services 层巨型文件和缺失的测试 CI 显示团队仍处于快速冲刺期。适合舆情/政策研究者做演示级推演、LLM 应用开发者学习"如何编排一个多阶段 agent 流水线"、以及多智能体模拟方向的研究者作为 OASIS 的工程化参考实现；不推荐在无自有 LLM/图谱预算、或需要严格合规许可的商用场景直接采用。
