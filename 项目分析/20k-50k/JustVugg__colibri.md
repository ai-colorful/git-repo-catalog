# JustVugg/colibri 项目分析

> 星数 48156（快照 2026-10-24）｜主语言 C｜档位 20k-50k｜分析日期 2026-11-06

## 1. 项目速览

一句话定位：纯 C 编写的分层（tiered）MoE 推理引擎——热专家放 GPU（CUDA/Vulkan/Metal），温专家放 RAM，冷专家从 NVMe 流式按需读取，让 744B~2.8T 的前沿 MoE 模型在普通桌面硬件上"能跑"。

| 项目 | 值 |
|---|---|
| 名称 | JustVugg/colibri（colibrì，蜂鸟） |
| 星数 / fork | 48156 / 5600（gh api，2026-11-06 核实） |
| 语言 | C（引擎）+ Python（安装器/网关/UI） |
| 许可证 | Apache-2.0（LICENSE，Copyright 2026 Vincenzo Fornaro） |
| 创建 / 最近推送 | 2026-07-01 / 2026-11-05（gh api） |
| 最新版本 | v2.1.0（2026-10-31，CHANGELOG.md） |
| 规模 | c/ 下 225 个文件、约 28.5 万行 C/CUDA 代码；全仓 1697 个文件 |
| 主页 | justvugg.github.io/colibri，社区 Discord |

## 2. 项目是什么，解决什么问题

核心观察（README.md "What colibri is, and why"）：MoE 模型"磁盘上巨大、每 token 很小"。GLM-5.2 共 744B 参数，每 token 只激活约 40B，其中逐 token 变化的部分（路由专家）约 11 GB。因此模型不需要"装进内存"，而需要"放置"：稠密部分（注意力、共享专家、嵌入）常驻 RAM，路由专家留在磁盘上由 router 按需读取。作者给出的最低起点是一台 25 GB RAM 的 12 核笔记本。

设计第一原则（README.md "How it works"）：**placement only ever decides speed**——专家住在 VRAM、RAM 还是磁盘，只改变速度，不改变路由决策和权重数值。每个 token 每层走固定的五步：route → union → place → overlap → learn。

工程形态（README.md "Repo layout"）：一个模型家族一个 `.c` 文件（如 `c/colibri.c` 对应 GLM-5.2/5.3 与 Mistral-Large-4，1.58 万行；`c/deepseek_v4.c` 2.27 万行为最大引擎），共享逻辑以单头文件形式被各引擎 include（`expert_ffn.h` 内核、`expert_store.h` 缓存契约、`route_trace.h` 路由遥测等）。运行时无 BLAS、无 Python；Python 只承担安装器、启动器、格式转换与 API 网关。

支持的模型面（README.md + c/ 各引擎文件）：13 个引擎覆盖 GLM-5.2/5.3、GLM-5.3-Flash、Inkling（975B）、Kimi K3（2.8T）、DeepSeek V4/V4.1 Flash、MiMo-V2.6、Qwen3.6/Qwen3.8-Flash-Next、OLMoE、Qwen-Image-2.1（文生图），以及决策模型 Laya、GLiNER2.5-Decide、Clef。v2.1.0 起所有引擎原生直读 GGUF（`c/gguf.h`），可直接消费 unsloth 等社区发布的现成 GGUF 文件，无需转换。

## 3. 技术栈与依赖

- **引擎（C11 + 可选扩展）**：OpenMP 线程池；AVX2/FMA、AVX-VNNI、AVX512-VNNI 手写内核，带定义数值语义的标量回退（`c/expert_ffn.h` 头注释）。无 BLAS。
- **GPU 后端**：Vulkan 1.2（`backend_vulkan.c` 7887 行 + `vk_chain.c` + `vk_tier.c`，GLSL shader 经 Makefile 生成 SPV）；CUDA（`backend_cuda.cu`、`cuda_chain.cu`、`cu_tier.h`，Windows 下为独立 DLL）；Metal（`backend_metal.mm`）；另有 `backend_xdna.c`（AMD XDNA NPU 实验）。`backend_gpu_compat.h` 把 `nvcuda::wmma` 映射到 rocWMMA，使一份 .cu 可编译给 NVIDIA 与 AMD HIP（README "Acknowledgements"）。
- **格式**：safetensors 读取（`c/st.h`，1562 行，含 fp8/I64 dtype）；GGUF 原位 pread 直读（`c/gguf.h`，头部作为不可信输入做全量校验），支持 IQ1_S~Q8_0、F16/BF16、MXFP4 块格式（`c/ggml_quants.h`）。
- **Python 工具链**（pyproject.toml，`colibri-engine`，Python ≥3.10）：一步安装（`c/setup_*.py`）、OpenAI/Anthropic 兼容网关（`c/openai_server.py`，10449 行）、资源规划（`c/resource_plan.py`）、MCP 服务器（`c/mcp_server.py`）；可选 extras：convert（numpy/huggingface_hub）、oracle（torch/transformers，仅测试对拍）、bench、vision。
- **外围**：web/ 为 TypeScript 仪表盘（package.json），desktop/ 为 Tauri 壳，ui/ 为无构建单页 + Python stdlib server（ui/README.md），docker/ 提供 slim 镜像，flake.nix 支持 Nix。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `c/` | 全部引擎与共享头文件、Makefile（3466 行）、CLI `coli`、Python 安装器/网关、`tests/`（553 个 test 文件） |
| `c/colibri.c` 等 13 个引擎 .c | 每模型家族一个引擎：架构定义 + 采样 + 该家族特有优化 |
| `c/expert_ffn.h` / `expert_store.h` / `route_trace.h` | 共享 MoE 内核 / 专家缓存 lease 契约 / 路由遥测与 `.coli_usage` 热度历史 |
| `c/backend_vulkan*` `vk_tier.c` `vk_chain.c` | Vulkan 设备层、专家 tier、稠密链 |
| `c/backend_cuda*` `cu_tier.*` `cuda_chain.cu` `cu_dense.h` | CUDA 对应物 |
| `c/st.h` `gguf.h` `tok.h` `grammar.h` | 容器读取、tiktoken 兼容分词、GBNF 子集语法约束 |
| `c/setup_*.py` `openai_server.py` `resource_plan.py` | 安装流水线、HTTP 网关、RAM/VRAM 规划 |
| `docs/` | 54 个参考文档 + experiments/ 实验日志 + baselines/ |
| `ui/` `web/` `desktop/` | 聊天 UI（单页无构建）、TS 仪表盘、Tauri 桌面壳 |
| `start-here.sh` / `START-HERE.bat` | 一步安装入口（探测硬件→推荐模型→构建/下载→启动） |

组织评述：**"单文件引擎 + 共享单头文件"** 是刻意的架构选择（README "Repo layout" 末段）：引擎只拥有自己的架构，两个引擎都需要的逻辑放进双方 include 的头文件，一处修复全体生效。与 llama.cpp 的统一图执行器（ggml 全图调度）不同，colibri 是每家族手写前向循环的"反框架"路线，代价是 deepseek_v4.c 等单文件超过 2 万行。

## 5. 核心架构与关键实现

### 5.1 三层专家放置（VRAM / RAM / NVMe）

**RAM 层**（`c/colibri.c`，GLM-5.2 引擎）：常驻稠密权重 + 专家缓存 = pinned 热集（`PIN`/`PIN_GB`，启动时从 `.coli_usage` 历史 AUTOPIN）+ 每层 LRU。`c/route_trace.h` 定义了跨引擎的 `.coli_usage` 持久化格式（文本三元组 `<layer> <expert> <count>`，负数字段做头部、engine_id 取 32 位哈希，注释详细论证了向后兼容旧 reader 的字段布局约束），并保留 inkling 的 IKU1 二进制格式兼容。`colibri.c:10862-10871` 还实现了回合间 re-pin：session 局部热层自适应，而持久历史不动。

**NVMe 层**（`c/colibri.c` 磁盘流式路径）：一个专家的 gate/up/down 三矩阵合并为一次 `pread`；`DIRECT=1` 走 O_DIRECT 绕过 page cache（`colibri.c:1535`、`3643`，注释明确"默认 OFF，取决于主机"）；读完用 `posix_fadvise(POSIX_FADV_DONTNEED)` 主动归还 page cache（`colibri.c:3696-3697`）；loader 线程池在常驻专家计算期间并行补读缺失专家；router-lookahead 线程预取下一层（README：GLM-5.2 路由一层前瞻可预测性 71.6%，`colibri.c:7320` 有"边算边 WILLNEED 读下一块"的异步 I/O 注释）。多盘镜像 `COLI_MODEL_MIRROR`（`colibri.c:3220-3247`）把读请求按相对带宽分摊到多块盘的副本——两块独立控制器 NVMe 实测 +37.5% decode。

**VRAM 层**（`c/vk_tier.c`，3360 行）：设备上的路由专家缓存。预算取自 `VK_EXT_memory_budget` 减去预留与稠密主干；启动时按 `.coli_usage` 热度填充，聊天中持续适应——CPU 计算过的专家成为晋升候选，热度超过最冷驻留者 LFRU 边际（LRU+频率，25%+4 次滞回，`vk_tier.c` 顶部 `VKT_UP_MARGIN`）才换入，避免抖动；上传走独立线程不阻塞层计算；`COLI_VK_TIER_EXCLUSIVE` 保证 RAM 与 VRAM 不重复持有同一专家（RAM 缓存先让出 device 已持有的槽位 `vkt_ram_first`）。还支持第二块设备（`COLI_VK_DEV2`）与 host pool（RAM 池中的专家供设备经 PCIe 批量读取）。

```c
/* c/vk_tier.c — 晋升策略的滞回参数（节选注释） */
#define VKT_UP_ROWS 8      /* promote 只接受 ≤8 行的步（decode/verify） */
#define VKT_UP_MARGIN 2u   /* 新人必须比最冷驻留者热 2 次路由才换入 */
/* Qwen3.8 IQ2_XS @ RTX PRO 4000：无边距时写历史的 prompt 因无效换页损失 4% */
```

**CUDA 层**（`c/cu_tier.h`）：关键设计是 `cu_tier.c` 把 `vk_tier.c` **用 CUDA 后端再编译一遍**（经 `cu_tier_ops.h`/`cu_xb.cu` 抽象），同一份 tier 逻辑服务两个 GPU 后端，日志标签区分 "[CUDA] tier"。

### 5.2 共享 MoE 内核与"数值忠实"

`c/expert_ffn.h`（925 行）是所有引擎共用的路由专家 FFN 内核：planar unsigned int4 gs=64 布局，块内低/高半字节天然对应元素 0-31/32-63，免 unpack 指令，直接喂 `vpdpbusd`（dot(u,x) − 8·sum(x) 补偿）；激活默认 f32（与 f32 参考只差累加顺序），i8 量化激活为 opt-in；整层一次 `xf_moe_run()`（两个 OpenMP region 取代 3×K 个），prompt 多行共享一次权重读取。头注释用实测数据解释了它存在的理由：全驻留下 MoE 层 34 ms/token，而 int4 磁盘字节只有 9 ms 的 DRAM 底线——装载时把 int4 解包成 int8 会同时翻倍字节量与常驻集。

### 5.3 GPU 混合执行与稠密链

Vulkan/CUDA 各有两条路径（README "GPUs"、`c/vk_chain.c`、`c/cuda_chain.cu`）：专家 tier（设备算它持有的专家，CPU 同时算其余，同一帧内合流）与 dense chain（整层记录为一次 GPU submission，模型状态跨层留在设备上）。CPU/GPU 同帧重叠是"overlap"步骤的核心：`colibri.c:7369` 明确目标 `t_emm = max(cpu, gpu)` 而非相加。对无 Resizable BAR 的老卡，Vulkan 路径自动改用 staging buffer 拷入权重（README "GPUs"，docs/vulkan.md）。

### 5.4 推测解码与长上下文

GLM-5.2 的 int8 MTP 头实测 2.2-2.8 tok/forward；Qwen3.8 MTP 默认开（+16-20%）；DeepSeek V4.1 的 DSpark 头按置信度决定验证深度，gate 跳过不合算的草稿（`docs/speculative.md`、CHANGELOG v2.1.0）。README 的研究表格坦承 MTP 在约 85% 专家命中时实测 -32%——推测与分层放置的相互作用被当作开放问题而非卖点。KV 侧：MLA 压缩 KV 每 token 576 floats（对 32768，57 倍压缩）且可跨重启存活；KV 换 f16 使 GLM-5.3 131k 上下文从 23.9 GB 降到 11.9 GB；`COLI_PARK` 让 agent 的短请求（起标题等）"停靠"会话而非驱逐其状态（14,810 token 的回合 86s → 4.9s）。

### 5.5 服务面：API、System One 与从笔记本到机架

- **API 网关**（`c/openai_server.py`，10449 行）：单服务器同时提供 OpenAI 兼容（/v1/chat/completions 等）与 Anthropic 兼容（/v1/messages）接口，编码 CLI/编辑器可直接接入；`--kv-slots N` 最多 16 个隔离 KV 会话并批量解码其下一 token（README "Use it from other apps"、docs/api.md）。
- **System One 决策模式**（README "System One"、`c/decide_serve.h`）：`POST /v1/systemone` 对给定 state 提类型化问题，模型读每个选项的概率而非生成文本，返回概率与置信度——"不生成"保证答案不越出候选集。请求/回复与 TypeSafe 的 Jev API 兼容，官方 SDK 改 base URL 即可切换。
- **横向扩展**（`c/cluster.py`、docs/cluster.md、docs/multidisk.md）：`COLI_MODEL_MIRROR` 多盘按带宽分摊读；`COLI_NUMA=1` 把常驻权重铺到多路主机的内存控制器；本地集群模式把路由专家放到其他机器上执行；大内存主机可 `CUDA_EXPERT_GB=auto PIN_GB=all` 全驻留，让磁盘彻底退出解码路径。

### 5.6 与 llama.cpp 的架构差异

colibri 与 llama.cpp 是两个不同的切面，且两者兼容（colibri 直读 GGUF，GBNF 语法子集与 Metal 常驻技巧均来自 llama.cpp，README "Acknowledgements"）：

| 维度 | llama.cpp | colibri |
|---|---|---|
| 执行模型 | ggml 统一张量图，全图调度 | 每模型家族一个手写前向的 .c 引擎 |
| 放置粒度 | 按张量/按层 offload（--n-gpu-layers） | 按专家放置：路由热度决定 VRAM/RAM/NVMe 三级驻留 |
| NVMe | mmap + 页缓存为主 | O_DIRECT/多盘镜像/预取线程池为第一公民 |
| 缓存学习 | 无路由历史概念 | `.coli_usage` 持久热度 + LFRU 自适应 tier |
| CPU/GPU 协同 | offload 后该层整体在 GPU | 同帧混合：GPU 算持有的专家，CPU 同时算其余 |
| 定位 | 通用 LLM 推理引擎 | 专注超大 MoE 的"放置引擎"（`c/expert_store.h` 的 lease 抽象即为此设计） |

## 6. 构建与运行

来自 README "Install by hand" 与 `c/Makefile`（3466 行，per-engine target）：

```bash
git clone https://github.com/JustVugg/colibri && cd colibri/c
./setup.sh                    # 检查 gcc/OpenMP、构建、自检（预期 ~30/32 通过）
make qwen36 VK=1              # 单引擎，VK=1 Vulkan / CUDA=1 CUDA / METAL=1 Metal
make colibri && make test-c   # GLM 引擎 + C 测试套件

./coli chat  --model /nvme/qwen36     # 终端聊天
./coli serve --model /nvme/qwen36     # OpenAI/Anthropic 兼容 API + 仪表盘
./coli plan  --model /nvme/qwen36     # 打印放置计划：VRAM/RAM/磁盘
./coli tune  --model /nvme/qwen36     # 实测并保存本机最优安全设置
./coli import x.gguf                  # 直读 GGUF，旁边生成模型目录
```

一步安装：Windows 双击 `START-HERE.bat`，Linux/macOS `./start-here.sh`——探测硬件→按 `c/setup_catalog.py` 推荐→本地构建或下载预编译引擎→断点续传下载模型→启动并开仪表盘；也支持让 AI 编码助手照 `docs/AI_SETUP.md` 执行，或经 `coli mcp`（MCP 服务器）操作。

## 7. 测试与工程质量

- **测试规模**：`c/tests/` 含 553 个 test 文件（test_*.c/py），另有大批 `bench_*.c/cu` 微基准与 fuzz（`fuzz_rans.c`、`fuzz_glm53_worker.c`）。风格以"对拍"为主：CI 用随机初始化小模型与 transformers 参考实现逐 token 对拍（README 称之为 "the oracle"）；Vulkan/CUDA 路径在 CI 中与 CPU 输出对拍（软件驱动/无卡环境，ci.yml 与 docs/gpu-parity.md）。
- **CI**（`.github/workflows/ci.yml`）：token 固定只读；每个引擎在每平台单独构建（注释里写明动机：此前 inkling/kimi_k3 在 msys2/UCRT64 编不过数月无人发现，#720）；构建 docker slim 镜像并实际运行 `--version`；另有 check.yml（lint 类）、release.yml（发预编译包）、site.yml。
- **质量观察**：`c/gguf.h`、`c/st.h` 把外部文件当不可信输入做全量校验（注释明确威胁模型"config.json 来自不可信 mirror"，`colibri.c:2057`）；`route_trace.h` 用 30 行注释论证一个 fscanf 兼容性格式的字段排序约束——工程注释密度与审慎度在 C 项目中属少见的高水准。风险面：单文件引擎超过 2 万行（deepseek_v4.c 22,741 行），认知负担集中在少数文件。

## 8. 社区与维护状态

- **节奏**：2026-07-01 建仓，07-19 v1.0.0，此后 3.5 个月发布 20+ 个 tag（v1.1.0 → v2.1.0，CHANGELOG.md 全记录），约每 1-2 周一版；最后推送 2026-11-05（gh api），昨日仍在更新。
- **贡献者**：v2.1.0 发布说明：387 个 PR，其中 208 个来自 35 位贡献者（CHANGELOG.md）；contributors API 分页到 100 封顶（≥100 人）。README "Supporting" 一节显示已从单人项目过渡到"社区真机数据点"模式。
- **社区设施**：Discord、7 种语言 README、issue 模板含 performance_report.yml（专门收基准数据）、`docs/benchmarking.md` 测量协议 + `tools/datapoint.py` 一键产出标准化数据点、docs/experiments/ 保存带完整设置的实验日志（如 glm52-6x5090-2026-07-12.md）。
- **基准数据的来源与可信度处理**：README/文档中的数字分两类——作者四台参考机（A/B/C/D，README 列明具体配置）上的自测，以及带 issue 编号的社区数据点（如 #200、#273、#1852、#455）。项目对"自述基准"的自我约束较为突出：benchmark protocol 要求记录硬件/commit/命令/缓存状态/expert hit rate 并附原始日志，研究表格公开列出负结果（MTP -32%、集成 GPU 上 OLMoE 反而变慢），并明确标注哪些实验"仍需做"。第三方独立复现的数量仍有限（多数大模型行只有 1-2 个数据点），引用具体数字时应视为该项目生态内的测量而非独立审计。

**4 个月 48k 星的增长背景**（客观陈述，不臆断）：仓库内部可见的吻合因素——（1）恰逢开源超大 MoE 密集发布期（GLM-5.2 744B、Kimi K3 2.8T、DeepSeek V4、Inkling 975B），" weights 公开了但没人跑得动"成为普遍痛点，colibri 的答案是唯一把 NVMe 流式做到一等公民的纯 CPU 可用方案；（2）极低采用门槛：一键安装、预编译 release、直读社区已发布的 GGUF（搭 unsloth 等量化生态的便车）；（3）开放基准文化持续制造可引用的数字与讨论（issue 即数据点）；（4）多语言 README 与 MCP/AI 助手设置贴合 2026 年的使用习惯。外部传播路径（HN/Reddit/新闻等）未在仓库内可见，未确认。

## 9. 亮点与风险

**亮点**
1. 把"分层存储"做成可证明不改变数值的正式契约（route/union/place/overlap/learn 五步 + expert_ffn.h 的标量回退定义精确数值语义），在同赛道项目（kTransformers/HybriMoE 等，README Acknowledgements 自引）中工程完整度突出。
2. VRAM tier 一份代码双后端（`vk_tier.c` 经 `cu_tier_ops.h` 再编译为 `cu_tier.c`），加上 rocWMMA 兼容层让 .cu 跨 NVIDIA/AMD，复用设计聪明。
3. 持久化路由热度（`.coli_usage`）+ LFRU 滞回 + 回合间 re-pin，"越用越快"有机制支撑且有防过拟合设计（新人边际 2 次路由）。
4. 直读 GGUF（含 IQ1_S~Q8_0/MXFP4 全谱块格式）使其立即兼容整个 llama.cpp 量化生态，转换脚本只作可选路径。
5. 文档与实验文化罕见地好：54 篇 docs、负结果公开、基准协议与 datapoint 工具、CI 全引擎对拍 transformers。

**风险/局限**
1. 大模型冷读速度受物理限制：GLM-5.2 冷态 0.05-0.1 tok/s（25 GB 笔记本）、Kimi K3 约 9.4 s/token——可用但"能跑"与"好用"之间差距巨大，强依赖 NVMe 带宽与专家命中率。
2. 自研 int4 容器（gs64）需从 FP8 release 转换或下载 429 GB 镜像，且旧 per-row int4 容器实测质量差约 9 分并引发循环输出（#455）——格式演进有真实踩坑史。
3. 4 个月膨胀到 28.5 万行 C，单文件超 2 万行，快速迭代的维护性风险与审计成本高；核心仍系作者主导（Apache-2.0，个人版权行）。
4. 自述基准占主导，独立第三方复现有限；部分数字（如 6×RTX 5090 全驻留 9.0-9.2 tok/s）需要接近顶配硬件，普通用户难以直接对照。

## 10. 总结评价

colibri 是近一年里"本地推理"赛道最有辨识度的新工程：它不与 llama.cpp/vLLM 比 engine 通用性，而是把 MoE 的稀疏性转化为存储分层问题，用"路由热度学习的 VRAM/RAM/NVMe 三级专家放置 + CPU/GPU 同帧重叠"让前沿规模模型在消费级硬件上可运行，并以罕见的开放实验文化（协议、负结果、对拍 CI）为自述基准建立了内部可信度。4 个月 48k 星是对"在你已有的硬件上跑最大的模型"这一诉求的直接回应。适合：想在无大显存机器上运行超大 MoE 的用户（一步安装 + GGUF 直读）、研究专家放置/预取/推测解码相互作用的系统研究者（experiments/ 与研究表格是现成起点）、以及想学习高性能 C 工程实践的开发者（单头文件组织与注释质量值得精读）。不追求吞吐服务场景（那是 vLLM 的地盘），也尚不适合作为质量敏感生产链路的唯一依据——先按 docs/benchmarking.md 在自己硬件上测，再下结论。
