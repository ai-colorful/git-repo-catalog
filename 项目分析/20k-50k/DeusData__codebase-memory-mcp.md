# DeusData/codebase-memory-mcp 项目分析

> 星数 48106（快照 2026-10-24）｜主语言 C｜档位 20k-50k｜分析日期 2026-11-05

## 1. 项目速览

一句话定位：用纯 C 实现的本地代码智能 MCP 服务器——把代码、文档与基础设施索引进一个持久的本地知识图谱，供开发者与 AI agent 通过 17 个 MCP 工具、Cypher 子集查询和 CBM Lens 可视化界面使用。

| 项 | 值 |
|---|---|
| 名称 | DeusData/codebase-memory-mcp |
| 星数 | 48,106（gh api，2026-11-05） |
| Fork | 3,964 |
| 主语言 | C（GitHub language 统计；实际为 C 后端 + TS/React 前端 + Go/npm/pypi 分发包装层） |
| 许可证 | MIT（LICENSE） |
| 版本 | v0.12.1（server.json，2026-10-16 发布） |
| 仓库创建 | 2026-02-24（gh api created_at） |
| 最近推送 | 2026-11-04（gh api pushed_at，极活跃） |
| 贡献者 | 98 人（gh api contributors 分页计数） |
| 官网 | https://deusdata.github.io/codebase-memory-mcp/ |

## 2. 项目是什么，解决什么问题

AI 编码 agent 靠"反复读文件 + grep"理解代码库，上下文消耗大、结构性信息（谁调用谁、哪个部署引用哪个镜像）基本丢失。codebase-memory-mcp 的思路是：把整个仓库（含文档、Docker/K8s/Terraform/CI 等基础设施）一次性索引为持久知识图谱，之后 agent 通过图查询获得精确的结构性上下文（README.md "Why codebase-memory-mcp"）。

核心能力分四块：

1. **代码图谱**：基于 162 个 vendored tree-sitter 语法（internal/cbm/vendored/grammars/，实测 162 个语法目录 + MANIFEST.md）提取符号与 20 余种边类型——`CALLS`、`CALL_REFERENCE`、`USAGE`、`IMPORTS`、`SPAWNS`、`INFRA_MAPS`、`CONNECTS_TO`、`DEPLOYS`、`CONFIGURES`、`SIMILAR_TO`（MinHash+LSH 近克隆）、`SEMANTICALLY_RELATED`（≥0.80 分）等（README "Edge types (selected)"）。
2. **MCP 服务层**：17 个工具（index_repository、search_graph、query_graph、trace_path、get_code_snippet、get_file_outline、get_graph_schema、compare_graphs、get_architecture、search_code、list_projects、delete_project、index_status、check_index_coverage、detect_changes、manage_adr、ingest_traces；src/mcp/mcp.c TOOLS 表），JSON-RPC 2.0 over stdio。
3. **CBM Lens（open beta v0.0.1）**：TS/React 前端（graph-ui/），提供源码阅读器、架构图、3D Galaxy 图谱浏览与本地代码问答，通过 daemon 的 HTTP 端口 9749 访问（README "CBM Lens Beta"、graph-ui/package.json）。
4. **变更感知测试选择（Smart CI，实验性）**：结合静态可达性 + 每测试覆盖率，产出 `test-impact select` 选择与理由；证据不足时回退 run_all（README "Smart CI and test impact"、src/mcp/test_impact* 约 40 个文件）。

作者自述架构是"结构分析后端"：后端不需要生成式模型，智能层由 MCP 客户端承担；索引与查询完全本地，无需 API key（README "How It Works"）。性能章节给出 Apple M3 Pro 基准：Linux 内核 28M LOC 全量索引 3 分钟得 4.81M 节点/7.72M 边，Django 全量约 6 秒，Cypher 遍历 <1ms（README "Performance"，为作者自测数据）。配套 arXiv 预印本（2603.27277，2026-03）声称 31 仓库实验中 token 用量降低 10 倍、工具调用减少 2.1 倍（README "Performance" 引述，未独立验证）。

## 3. 技术栈与依赖

- **后端：纯 C**（README 徽章 "Pure C: no language runtime"；自述 pure C rewrite，Makefile.cbm 头注释）。非 vendored C/H 约 52.7 万行、634 个文件（实测统计，排除 vendored）。
- **第三方库全部 vendored**（vendored/）：SQLite3（图存储）、mimalloc（分配器）、yyjson（JSON）、xxhash（哈希）、tre（正则）、nomic（嵌入向量）。tree-sitter 运行时与 162 个语法在 internal/cbm/vendored/grammars/。tarball 解压后 1.4GB，绝大部分是语法生成的 parser.c（lean 单文件 292 万行）。
- **前端 CBM Lens：TypeScript + React + Vite**（graph-ui/package.json，"cbm-lens" v0.0.1），约 2 万行 TS/TSX（实测 graph-ui/src 统计）。
- **分发包装层**：pkg/ 下有 go、npm、pypi、homebrew、winget、scoop、chocolatey、aur、glama 共 9 种打包目录；server.json 声明 npm/pypi 双注册表入口（runtimeHint npx/uvx）。
- **构建工具链**：自研 Makefile.cbm（90KB，90710 字节）+ scripts/build.sh；CI 用 cppcheck/clang-format/clang-tidy/CodeQL（.github/workflows/）。
- 协议：MCP（Model Context Protocol）、JSON-RPC 2.0 over stdio + daemon HTTP（main.c 头注释）。

## 4. 目录结构与代码组织

| 目录 | 职责 |
|---|---|
| src/ | "新地基"层 C 代码：mcp/（MCP 服务器与 test-impact 引擎）、store/（SQLite 图存储）、pipeline/（索引管线与 40+ pass）、foundation/（arena/内存/线程/跨平台兼容）、daemon/（会话守护进程）、cypher/（Cypher 子集引擎）、cli/、discover/、git/、graph_buffer/、semantic/、simhash/、watcher/、traces/ |
| internal/cbm/ | 旧核心"提取层"（cbm.c、arena.c、ac.c、doclink_* 等 293 个文件）+ vendored tree-sitter 运行时与 162 语法 |
| graph-ui/ | CBM Lens 前端（React/Vite），含 llm/（本地模型）、eval/、大量冒烟测试 |
| tests/ | 183 个 test_*.c + 65 个 shell 合同测试 + fixtures/fuzz/repro，共 278 个条目 |
| pkg/ | 9 种包管理器分发包装（含 Go 包装） |
| tools/ | 自有/分叉的 tree-sitter 语法（arkts、chialisp、form、magma、pascal 等） |
| scripts/ | 129 个构建/CI/基准/审计脚本（audit-grammar-security.sh、check-lsp-originality.sh 等） |
| docs/ | BENCHMARK、SMART_CI、INDEX_RESOURCE_LIMITS、cbmignore 等技术文档 |
| .github/ | 24 个 workflow（build/lint/test/smoke/soak/security/memwaste/codeql/scorecard/release） |
| plugin/、.claude-plugin/ | Claude Code 插件市场清单 |

组织方式评述：单体 C 仓库 + 内嵌 TS 前端 + 多语言分发包装的混合形态。src/ 与 internal/cbm/ 是两代代码共存（Makefile.cbm 注释称 internal/cbm 为 "existing extraction headers"，src/ 为 "new foundation"），Go 风格的 internal//pkg/ 目录命名但内容全是 C。测试、脚本、CI 配置规模异常庞大，工程质量导向明显。

## 5. 核心架构与关键实现

### 5.1 SQLite 图存储（src/store/store.c，13210 行）

不透明句柄 `cbm_store_t` 封装全部 SQLite 细节，带预编译语句缓存；节点/边模式：

```c
// src/store/store.h
typedef struct {
    int64_t id;
    const char *project;
    const char *label;          /* Function, Class, Method, Module, File, ... */
    const char *name;           /* short name */
    const char *qualified_name; /* full dotted path */
    const char *file_path;
    int start_line, end_line;
    const char *properties_json;
} cbm_node_t;
```

Schema 含 nodes/edges/projects/file_hashes/project_summaries/lsp_surface/index_coverage/observed_calls 等表 + `nodes_fts` FTS5 虚拟表（store.c:365-634），节点按 project+label/name/file_path 建索引。查询优化痕迹密集：regex 搜索先做 SQL LIKE 预过滤 + 锚定名前缀范围扫描（最多 2^3 个大小写变体 UNION ALL 臂，store.c 枚举 ST_NAME_RANGE_*），BFS 用有界 CTE（ST_BFS_MAX_CTE_ROWS=4096）。索引格式版本号 CBM_INDEX_FORMAT_VERSION=4，升级触发全量重建（store.h）。

### 5.2 索引管线（src/pipeline/pipeline.c，5531 行 + 40 个 pass 文件）

七阶段编排（pipeline.c 头注释）：

```
1. Discover files → 2. Build structure → 3. Bulk load (LZ4 HC 压缩)
→ 4. Extract definitions → 5. Resolve imports/calls/usages/semantic edges
→ 6. Post-passes (tests/communities/HTTP/git history) → 7. Dump graph buffer to SQLite
```

"RAM-first"：LZ4 压缩常驻内存的 SQLite，结束时一次性落盘（README "Indexing pipeline"）。pass 覆盖面极宽：pass_docker/k8s/terraform/ansible/ci/helm_template/laravel_routing/cross_repo/lsp_cross/semantic_edges/similarity（MinHash 近克隆）/importance 等（src/pipeline/ 目录）。增量索引走 pipeline_delta.c/pipeline_incremental.c，并行用 worker_pool.c。

### 5.3 MCP 服务层（src/mcp/mcp.c，22646 行）

手写 JSON-RPC 2.0 解析与 17 个工具的静态注册表（mcp.c:536 `TOOLS[]`），JSON 处理用 yyjson。值得注意的工程细节：支持字符串型 request id（issue #253）、长索引转 daemon 异步 + 轮询（CBM_MCP_INDEX_ASYNC_HINT，issue #2144）、工具结果附加 notice 的重写机制（cbm_mcp_tool_result_add_notice）。test_impact_* 子模块约 40 个文件构成独立的测试影响引擎（可达性 + 覆盖图 + 变更分类 + 产出回执），含 Windows/POSIX ACL 处理。

### 5.4 进程模型与查询引擎

入口 main.c 定义三种模式：默认 stdio MCP 服务器、`cli <tool> <json>` 单次调用、`--ui` HTTP 界面（端口 9749）。长会话由"每账号强制 daemon"承载，MCP 与 hook 前端是 daemon 的瘦客户端（main.c 头注释、src/daemon/ 含 ipc/host/service/version_cohort/project_lock）。查询侧自实现 openCypher 读子集（src/cypher/cypher.h）：支持 `MATCH (n)-[:TYPE*1..3]->(m)` 变长路径、WHERE 正则、聚合、ORDER BY/LIMIT、UNION/UNWIND，词法-解析-计划-执行四段式。

### 5.5 内存管理（src/foundation/）

自建 arena + 分层内存设施：mem_core.c 把 mimalloc 绑定为全局分配器（main.c 注释"bind 3rd-party allocators to mimalloc before any sqlite/git init"），mem_override_*.c 按平台覆盖 malloc，vmem.c 管理保留内存，mem_events.c 记录内存事件——与其自研 "memwaste" CI 工作流（.github/workflows/_memwaste.yml）呼应。

## 6. 构建与运行

来自 README "Build from Source" 与 scripts/：

```bash
# 安装（macOS/Linux）
curl -fsSL https://raw.githubusercontent.com/DeusData/codebase-memory-mcp/main/install.sh | bash
codebase-memory-mcp daemon start --open --port=9749   # 启动并打开 CBM Lens

# 源码构建（需 C/C++ 编译器、zlib；--with-ui 需 Node.js 22+）
scripts/build.sh --with-ui     # 产出 build/c/codebase-memory-mcp，CBM Lens 内嵌
scripts/build.sh               # 纯后端（开发用）

# 测试
scripts/test.sh                    # 全量：sanitize 构建 + 全部套件
scripts/test.sh --suites <name>    # 单套件增量
make -f Makefile.cbm test          # ASan+UBSan 构建 + 全部测试（Makefile.cbm 头注释）

# CBM Lens 前端
cd graph-ui && npm ci && npm run test:unit && npm run build
```

支持 9 种分发渠道（pkg/：npm/pypi/homebrew/winget/scoop/chocolatey/aur/go/glama），Nix flake（flake.nix）可选不带 UI 的构建。

## 7. 测试与工程质量

- **规模**：README 自述 main 分支 9,388 个原生 C 测试/189 套件 + 3,970 个前端 Vitest 测试 + Go/npm/pypi 包装层 97 个测试（README "Tests" 表，注明核查于 commit 4d9592b5）。本地实测 tests/ 有 183 个 test_*.c、65 个 shell 合同测试，另含 fuzz/ 与 repro/ 目录。
- **框架**：自研极简 C 测试框架 tests/test_framework.h（TEST/SUITE 宏、CBM_TEST_ONLY 选择、每测试覆盖率 profile）。
- **CI**：24 个 workflow（.github/workflows/）——分片 build/lint/test/smoke/soak/security 外加 codeql、OpenSSF scorecard、nightly-soak、memwaste（内存浪费检测）、test-impact-artifact；Smart CI 在 Linux x86_64 测试腿上以 TEST_IMPACT_MODE=gate 接线，用自身仓库做狗粮（README "Smart CI"、.codebase-memory.json 内置 test_impact 规则）。
- **供应链安全**：SLSA 3 徽章、VirusTotal 每次发布扫描、OpenSSF Scorecard、scripts/audit-grammar-security.sh 与 audit-license-provenance.py（README 徽章、scripts/ 目录）。
- **代码质量观察**：clang-format/clang-tidy/cppcheck 三件套；注释密度高且引用 issue 号（如 #1083 WAL 饥饿告警、#1913 Go 方法 QN 修正）；README 坦承 1 个失败测试。单一 Makefile 90KB、mcp.c 单文件 2.26 万行，模块内聚但文件粒度偏大。

## 8. 社区与维护状态

- **活跃度**：gh api 显示 pushed_at 2026-11-04（分析前一天），555 个开放 issue，30 个 release、56 个 tag；最近 release v0.12.1（2026-10-16），此前 v0.12.0（2026-10-15）、v0.11.0（2026-09-15）、v0.10.8（2026-08-19），月度级发布节奏，近期明显加速。
- **增长事实（客观陈述）**：仓库创建于 2026-02-24（gh api created_at），至 2026-11-05 达 48,106 星、3,964 fork，即约 8 个多月内完成千星到近 5 万星的跃升；同期配套发布了 arXiv 预印本（2603.27277）与 Discord 社区（README 徽章）。该增速在 GitHub 历史上属于极快档；仓库无星标历史数据可供进一步归因，本报告不作推测。
- **贡献者结构**：gh api contributors 显示 DeusData 3,445 次贡献居首（约占前 10 名总量 87%），第二位 BernhardJackiewicz 210 次，其余均低于 80 次；总计 98 名贡献者 + dependabot。典型单主体主导、社区开始参与的早期形态。MAINTAINERS.md（9KB）存在正式维护者文档。

## 9. 亮点与风险

**亮点**

1. 纯 C 无运行时依赖的本地代码智能后端：162 个语法内嵌、单二进制分发 9 渠道，冷启动与隐私友好（vendored/、pkg/、server.json）。
2. 图谱范围独到：不止代码符号，还把 Docker/K8s/Helm/Terraform/Ansible/CI 脚本纳入同一张图并互连（SPAWNS/DEPLOYS/CONFIGURES 等边类型，README "Edge types"），"Service → workload → Image → Dockerfile → main" 的跨层追踪是同类 MCP 工具少见的。
3. 工程纪律罕见地重：9,388 个 C 测试、sanitize/soak/memwaste/codeql/SLSA3 全套、自身 Smart CI 狗粮化（.github/workflows/、.codebase-memory.json）。
4. Agent 集成深度第一：45 个客户端表面的安装器 + Scout/Verify/Auditor 三层 agent 契约 + hook 注入（README "Multi-Agent Support"），显著降低接入摩擦。
5. 自建 Cypher 子集查询引擎（src/cypher/）让图查询表达能力超出固定工具集。

**风险/局限**

1. 单点依赖：核心代码 87% 出自单一账号，bus factor 低；555 个开放 issue 的消化能力取决于该主体。
2. 二进制体积与索引成本：1.4GB 源码树、162 语法全编译进二进制，"fast/moderate/full" 模式分层只是部分缓解；自测数据（3 分钟索引 Linux 内核）未独立复核。
3. Smart CI 测试收窄仅支持 C/C++ 系测试约定，跨语言场景退化为 run_all（README "Smart CI"）；CBM Lens 仍处 v0.0.1 beta。
4. 文档指标多为作者自报（token 节省 10 倍、性能表），第三方验证有限；Defender 误报问题需要用户自行甄别（README 安装段自述）。

## 10. 总结评价

codebase-memory-mcp 是 2026 年 MCP/agent 工具潮中工程质量最重的项目之一：以 52 万行纯 C 实现了"解析-图谱-存储-查询-可视化-分发"全链路，把代码智能从"多读几个文件"推进到"跨代码与基础设施的结构查询"，并用 9,388 个测试与 SLSA3/Scorecard 级 CI 支撑发行。它极度匹配当前"给 agent 结构性上下文"的核心痛点，agent 集成矩阵（45 客户端）本身就是获客机器；8 个月 48k 星的增长与该痛点强度、发布节奏和可安装性一致，但贡献高度集中于单一作者，长期演进与独立验证仍需观察。适合给 Claude Code/Codex 等编码 agent 配结构上下文的开发者、以及想研究"C 语言工程化 + 图索引 + MCP 协议实现"的工程师读源码；src/store/store.c、src/pipeline/ 与 src/mcp/mcp.c 是最佳切入点。
