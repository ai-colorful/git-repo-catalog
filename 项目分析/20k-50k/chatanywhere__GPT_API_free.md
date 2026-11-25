# chatanywhere/GPT_API_free 项目分析

> 星数 45463（快照 2026-10-24）｜主语言 —（gh api 显示 language=null，无主语言）｜档位 20k-50k｜分析日期 2026-11-06

## 1. 项目速览

一句话定位：ChatAnywhere 团队运营的大模型 API 转发/转售服务的 GitHub 推广仓库——本体是一份高密度服务说明文档（README，442 行）加 8 个 OpenAI SDK 调用示例脚本，仓库内不含任何服务端实现代码。

| 项 | 值 | 出处 |
| --- | --- | --- |
| 名称 | chatanywhere/GPT_API_free | gh api |
| 星数 | 45,463（快照 2026-10-24） | 任务参数 / gh api |
| 主语言 | 无（language=null） | gh api |
| 许可证 | MIT，版权归"牛爷爷信息技术（南京）有限公司" | `LICENSE` |
| 创建时间 | 2023-04-23 | gh api |
| 最近推送 | 2026-10-30（Update README.md，价格表维护） | gh api / commits |
| Forks / Open Issues | 2,766 / 11 | gh api |
| 服务入口 | https://api.chatanywhere.tech（国内）/ https://api.chatanywhere.org（国外） | `README.md` |
| 文档站 / 状态页 | docs.chatanywhere.tech / status.chatanywhere.tech | `README.md` |
| Topics | api, chatgpt, claude, deepseek, gemini, gpt, grok, llm-agents | gh api |

## 2. 项目是什么，解决什么问题

这不是一个可运行的软件项目，而是一个商业服务的"门面仓库"：ChatAnywhere 将自家的 API 转发服务文档放在 GitHub 上，用免费额度吸引用户领取 Key、引导 Star，并将付费转售作为持续经营手段（`README.md` 自述"纯公益提供免费Key显然不是能持久运营下去的方案，所以我们引入付费API Key维持项目的日常开销"）。

其转发服务的技术形态在 README 中描述为：提供与 OpenAI 官方完全一致的接口标准，用户只需把 `base_url` 从 `api.openai.com` 换成 `api.chatanywhere.tech`（国内中转）或 `api.chatanywhere.org`（国外），即可用既有 OpenAI SDK / 插件直接调用。协议层面是"all-in-one"：各家厂商模型统一走 OpenAI 标准协议，仅更换 model 名称；claude 系列另支持 Anthropic 官方协议（`README.md` 特点一节）。

解决的问题从用户侧看有三点：一是国内网络环境下无需代理直连调用主流大模型（README 强调"国内线路使用动态加速"）；二是免费试用——注册并绑定 GitHub 账号后领取免费 Key，配额为每周 50000 点、每天 100 次请求，仅限个人非商业用途、教育与非营利性科研（`README.md` "免费使用"与"注意事项"节）；三是模型聚合——一个 Key/一个接口同时覆盖 gpt、deepseek、claude、gemini、qwen、kimi、glm、minimax 等系列。

免费与付费的差异：免费 Key 对思考模型的思考强度设限、有速率与配额限制；付费 Key 按 CA 币充值计费，宣称价格为官方 1-2 折，余额永久有效，按官方返回的 tokens 用量计费（`README.md` "付费版API"节）。

供给侧信息 README 有明确披露：多数第三方厂商模型"由第三方供应商提供，有小概率可能会出现响应速度较慢或报错"；`-ca` 后缀系列（gpt-*-ca、gpt-image-2-ca 等）标注为"第三方优质提供商提供的服务，优点价格便宜，但是稳定性没有非-ca的好"；其中 claude-sonnet-4-6 / claude-sonnet-4-5 明确写着"现在使用的为Claude code的逆向或官方逆向的渠道"（`README.md` 计价表"特点"列）。这些是仓库自己对服务构成的客观描述。

## 3. 技术栈与依赖

仓库本体（内容型，无工程构建体系）：

- 内容：Markdown（README.md 43KB / 442 行）、SVG logo、GitHub issue 模板。
- 示例脚本：Python，依赖 `openai`（官方 Python SDK）与 `requests`（仅 `demo/openai_chat_completion_audio_demo.py` 用于下载音频），见 `demo/*.py` 各文件 import 区。
- 服务端：转发 API 的真实实现（网关、计费、账号系统）不在本仓库中，技术栈未公开——"未确认"。

## 4. 目录结构与代码组织

| 路径 | 职责 |
| --- | --- |
| `README.md` | 唯一核心内容：服务介绍、隐私声明、免费/付费说明、约 100 行的付费模型计价表（分 4 张表）与阶梯计价注释、使用入口 |
| `demo/` | 8 个 Python 示例脚本，演示 chat.completions / responses / 生图 / TTS / 语音转写 / 音频输入，明细见下 |

`demo/` 明细（行数为实际统计）：

| 文件 | 行数 | 演示内容 |
| --- | --- | --- |
| `openai_response_demo.py` | 150 | responses API 五合一：对话/图像/PDF/联网搜索/函数调用 |
| `openai_chat_completion_picture_demo.py` | 101 | gpt-4o 视觉：URL 图片与本地 Base64 两种识别方式 |
| `openai_chat_completion_audio_demo.py` | 36 | 音频输入多模态对话（gemini-2.5-pro + wav 输入） |
| `audio_speech_demo.py` | 32 | TTS 语音合成 |
| `transcriptions_demo.py` | 28 | Whisper 语音转写 |
| `openai_chat_completion_demo.py` | 22 | 最小 chat.completions 示例（含流式） |
| `gpt_image_1_edit_demo.py` | 27 | gpt-image-1 图片编辑 |
| `gpt_image_1_gen_demo.py` | 17 | gpt-image-1 文生图 |
| `images/` | `logo-lockup.svg` 品牌图；含一个 macOS 遗留的 `.DS_Store`（6KB） |
| `.github/ISSUE_TEMPLATE/` | 中英双语 bug_report.md（27 行）与 feature_request.md（20 行） |
| `LICENSE` | MIT 文本 |
| `.gitattributes` / `.gitignore` | 各 1 行：LF 规范化；忽略 `.DS_Store`（仓库里却仍提交了一份） |

组织方式评述：无构建系统、无工程清单（没有 package.json / requirements.txt / CI 配置）、无测试目录。整个仓库就是"文档 + 示例"的扁平结构，维护方式是官方账号直接改 README 的价格表。此外，2026-07-27 的提交曾"Remove demo files for Java, Go, Node.js, Python, and add new audio and image processing demos"——多语言示例被裁撤，改为以 responses API 与音频/图像为重点的 Python 示例（gh api commits）。

## 5. 核心架构与关键实现

仓库内可分析的"核心内容"是两块：README 的计价信息体系，和 demo 脚本演示的接入面。

**（1）计价表（`README.md` 87-270 行）**。付费模型分四张表罗列：deepseek/qwen/kimi/glm/minimax 系列、gpt 系列与 -ca 系列、claude 与 gemini 系列、图像/语音/embedding 类模型（gpt-image-2、tts-1、whisper、text-embedding-3 等）。每行给出输入/输出单价（CA 币/1K tokens）与来源备注；对长上下文模型附"阶梯计价"子表（如 gpt-6.1-terra 以 272K 输入为界分两档，qwen3.5-plus 分三档）。价格列中出现大量 2025-2026 年份的模型名（gpt-5.4、gemini-3.6-flash、kimi-k3 等），README 同时声明"列表可能未及时更新，请以官方信息为准"。这张表是仓库保持高频提交的原因——上游价格变动就改表。

**（2）接入面（`demo/`）**。所有脚本共享同一个模式：用 openai SDK 把 `base_url` 指到转发地址。最小示例 `demo/openai_chat_completion_demo.py` 仅 22 行：

```python
client = OpenAI(
    # 输入转发API Key
    api_key="sk-xxxxxxx",
    base_url="https://api.chatanywhere.tech/v1"
)
completion = client.chat.completions.create(
    model="gpt-3.5-turbo-1106",
    messages=[...],
    stream=True  # 是否开启流式输出
)
```

覆盖面最全的是 `demo/openai_response_demo.py`（150 行）：依次演示 responses API 的基础对话、图像识别（input_image）、PDF 文件分析（input_file）、联网搜索（web_search_preview 工具）与 function calling 五类用法，每个用例独立成函数并带异常兜底打印。函数调用部分直接使用 OpenAI 官方工具声明格式（`demo/openai_response_demo.py`）：

```python
tools = [
    {
        "type": "function",
        "name": "get_current_weather",
        "description": "获取指定地点的当前天气",
        "parameters": {
            "type": "object",
            "properties": {
                "location": {
                    "type": "string",
                    "description": "城市和州，例如 San Francisco, CA",
                },
                "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]},
            },
            "required": ["location", "unit"],
        }
    }
]
```其余脚本分别演示 gpt-image-1 的生成与编辑（`gpt_image_1_gen_demo.py` / `gpt_image_1_edit_demo.py`）、TTS 语音合成（`audio_speech_demo.py`）、whisper 转写（`transcriptions_demo.py`）、音频输入的多模态对话（`openai_chat_completion_audio_demo.py`，用 gemini-2.5-pro 模型名+OpenAI 协议调音频）。

**（3）服务端架构（不在仓库内，未确认）**。转发网关、计费系统、Key 管理与 GitHub 绑定发放逻辑均为闭源服务，仓库仅暴露其对外契约（两个 Host + OpenAI 兼容协议）。

## 6. 构建与运行

仓库无可构建产物；可用操作只有运行示例脚本（来自 `demo/*.py` 文件头与内容，README 未给出安装命令）：

```bash
pip install openai requests          # 依赖（各 demo import 所示）
# 将 demo/openai_chat_completion_demo.py 中 api_key 换成自己领取的转发 Key 后：
python demo/openai_chat_completion_demo.py
```

服务本身不需要部署——注册绑定 GitHub 领取 Key（README 指向 chatanywhere.tech），把 SDK 的 base_url 改为 `https://api.chatanywhere.tech/v1` 即可调用；可用性通过 status.chatanywhere.tech 状态页查询。

## 7. 内容与示例工程质量

- 无测试、无 CI、无 lint 配置：全仓库无 .github/workflows，示例脚本即全部"代码"。
- 示例脚本质量：结构清晰、中文注释完整、每个用例带 try/except 与分隔打印（`demo/openai_response_demo.py`）；但配置方式为直接改源码常量，无环境变量/参数化。
- 一处明显瑕疵：`demo/openai_chat_completion_audio_demo.py` 第 7 行硬编码了一个疑似真实格式的 API Key（`sk-dohF...eXaH7P`），其余脚本均为 `sk-xxxxxx` 占位符。该 Key 是否有效未确认（不验证），但属于不应提交的敏感信息样式。
- 文档质量：README 信息密度高且更新勤，但结构冗余——同一批模型在多张表中重复出现（deepseek 系列在表一、表二重复整段列出），价格表与文末阶梯计价注释需要交叉对照。
- 仓库卫生：`.gitignore` 声明忽略 `.DS_Store` 却仍提交了一份在 `images/` 中。

## 8. 社区与维护状态

- 维护者结构：提交历史中可见的作者几乎全部是官方账号"ChatAnywhere"单方面更新（gh api commits 前 8 条均如此）；contributors 分页推断约 2 人（per_page=1 时 last page=2）。外部主要参与方式是 Issues 反馈（当前 open 11 个）而非代码贡献。
- 更新节奏：2026 年内可见 7 月（重构 demo、更新 logo）、9 月、10 月多次提交，最近一次为 2026-10-30 的 README 价格表更新——维护是"运营型"的，围绕价格与服务说明，而非功能开发。近期提交记录（gh api commits）：

| 日期 | 作者 | 内容 |
| --- | --- | --- |
| 2026-10-30 | ChatAnywhere | Update README.md（价格表） |
| 2026-09-07 | ChatAnywhere | Update README.md |
| 2026-07-28 | ChatAnywhere | update |
| 2026-07-27 | ChatAnywhere | 删除 Java/Go/Node.js/Python 多语言 demo，新增音频与图像 demo；新建 .gitignore |
| 2026-07-26 | ChatAnywhere | update logo |
- Release：0 个（gh api releases）。
- 活跃度判断：服务在持续运营（状态页徽章、价格表随上游模型代际滚动更新），仓库星数 45.4k / fork 2.8k，主要来自 README 的免费额度引导（README 明文"如果该项目对你有帮助，还请为我们点一个Star"）。

## 9. 亮点与风险

亮点：

1. 文档即产品：计价表全量公开（含阶梯计价明细），接入说明一步到位，信息密度远高于同类转售服务仓库（`README.md`）。
2. 接入成本极低：完整兼容 OpenAI 协议，存量代码/插件改一行 base_url 即可切换，demo 覆盖对话/视觉/文件/搜索/函数调用/生图/TTS/转写全模态（`demo/`）。
3. 免费额度真实可领：绑定 GitHub 即得每周 5 万点 + 每天 100 次，为个人学习/原型验证提供了零成本入口（`README.md`）。
4. 运维透明度尚可：提供 status 状态页与各模型 uptime 徽章、中英双语 issue 模板、价格免责声明（`README.md` 头部徽章、`.github/ISSUE_TEMPLATE/`）。

风险/局限（基于仓库自述的客观陈述，不做合规性评判）：

1. 供给链依赖第三方：多数模型"由第三方供应商提供"、-ca 系列"稳定性没有非-ca的好"、部分 claude 模型自述使用"逆向"渠道——README 反复出现"有小概率响应速度较慢或报错"的提示，稳定性不受本仓库控制（`README.md` 计价表）。
2. 转售模式的上游政策风险：转售 API 与"逆向渠道"均处在上游厂商服务条款的灰色地带，README 自行声明"商用或面向大众使用请自行承担风险"，并要求用户在"合法合规场景下使用"（`README.md` 注意事项与温馨提示）。
3. 隐私与数据路径：所有请求内容经转发服务器中转；README 隐私声明仅约束服务方自身不记录，同时明示上游 OpenAI 按其政策保留数据 30 天——使用者在协议之外多了一个中间方（`README.md` 隐私声明）。
4. 工程卫生问题：demo 中硬编码疑似真实 Key（`demo/openai_chat_completion_audio_demo.py:7`）、无 CI/测试、免费 Key 限额且严禁商用、滥用封禁依赖 QQ 群人工申诉。

## 10. 总结评价

GPT_API_free 是"星数极高、代码极少"的典型：45k+ 星对应的仓库实体只是一份精心维护的服务说明书加几百行接入示例，星数本质上是 ChatAnywhere 转发服务的用户量与免费额度吸引力的度量，而非开源软件社区的度量。作为技术产出，它的价值在于一份公开、细到阶梯计价的模型价格表和一套覆盖全模态的 OpenAI 兼容接入示例；作为服务，它是一个依赖第三方供应商与逆向渠道的商业转售门面，稳定性、隐私路径与上游政策风险均由使用侧承担（README 亦如此自述）。适合想以最低成本在国内环境试用各家大模型 API 的个人学习者快速上手，或想参考"如何写一份高转化 API 服务 README"的从业者阅读；不适合作为学习工程架构的源码材料——仓库里没有架构可读，服务端实现未开源。
