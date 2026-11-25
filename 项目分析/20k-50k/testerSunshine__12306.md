# testerSunshine/12306 项目分析

> 星数 33000（快照 2026-10-24；2026-11-21 gh api 实测 33,997）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-21

## 1. 项目速览

一句话定位：面向 12306 购票站点的 Python 自动化查询/下单脚本，国内火车票抢票类工具中星数最高的项目之一，现已归档停更。

| 项 | 值 |
|---|---|
| 名称 | testerSunshine/12306 |
| 星数 / Forks | 33,997 / 9,557（gh api 2026-11-21） |
| 语言 | Python（Python3.6+，2019.08.31 起放弃 2.7，见 Update.md） |
| 许可证 | MIT（LICENSE，Copyright (c) 2017） |
| 创建时间 | 2017-05-17（gh api） |
| 归档状态 | **archived=true**（gh api 2026-11-21 核实） |
| 仓库级最后推送 | 2023-04-02（gh api pushed_at），默认分支 master 最后提交 2021-01-04「站点信息更新」，快照内文件 mtime 均为 2021-01-04 |
| 提交总数 | 约 435（commits 分页 Link 头 page=435，per_page=1） |
| Release | 仅 1 个：tag 1.0（2019-12-06） |

## 2. 项目是什么，解决什么问题

本项目是一个针对 12306 官方购票网站的自动化购票脚本：将人工"查余票→选车次→填乘车人→提交订单→排队等待→付款"的流程全部脚本化，循环执行余票查询并在出现符合配置条件的车票时自动走完下单链路（README.md「已有功能」清单：自动打码、自动登录、准点预售和捡漏、智能候补、邮件通知、Server 酱通知）。

从源码看，它的工程实质是一套围绕 12306 HTTP 接口的**接口编排引擎**：`config/urlConf.py` 集中登记了 46 个接口端点定义（每个端点含 url/Referer/Host 三要素），`inter/` 目录的 22 个模块与 12306 下单流水线的每一步一一对应（查询、校验用户、提交订单、排队确认、等待出票等），由 `init/select_ticket_info.py` 的主循环按状态机方式串联调度。

项目的核心难点不在功能本身而在**对抗站点环境变化**：Update.md 显示 2018-2019 年间 12306 每次改版/风控调整都会导致登录或下单失效，作者需紧急修复接口（如「2019.04.01 增加登录接口，解决无法03.31登录」「2019.04.23 修复登录302问题」）。这一"改版→失效→抢修"循环在 2019 年 12 月作者停止实质性跟进后终结，master 分支仅剩 2021-01-04 一次车站数据更新，随后进入实质停更直至归档。

README 声明「本软件只供学习交流使用，勿作为商业用途」。作为灰区工具，本文仅从软件架构角度做客观技术分析，不涉及使用指导。

## 3. 技术栈与依赖

依赖清单来自 `requirements.txt`（Python 3.6-3.7.4，README 版本声明）：

| 类别 | 依赖 | 用途 |
|---|---|---|
| HTTP | requests 2.18.4、fake-useragent 0.1.11 | 接口请求与 UA 伪装；项目自带轻量封装 `myUrllib/httpUtils.py`（HTTPClient），未用 requests.Session 的上层抽象 |
| 验证码识别（本地） | tensorflow 1.14、keras 2.2.4、scikit-learn、opencv-python、Pillow、numpy/scipy | 根目录两个 Keras h5 模型文件（12306.image.model.h5 约 60MB、model.v2.0.h5 约 0.7MB，tarball 实测占全部 61.9MB 体积的 98%）；模型来源为 zhaipro/easy12306（README 注明） |
| 云打码 | 无额外依赖 | 通过 HTTP 调用自托管打码服务（docker-compose.yml 中 `yinaoxiong/12306_code_server` 容器） |
| 辅助 | ntplib 0.3.3（`config/AutoSynchroTime.py` 时间同步）、wrapcache 1.0.8（内存缓存，主循环用于 user_info）、beautifulsoup4、selenium 3.11.0（历史遗留，主流程未见引用） | |

工程化：Dockerfile（遗留的 python:2.7.15 版）+ Dockerfile37 + docker-compose.yml 双服务编排（ticket 主服务 + captcha 打码服务），`.github/ISSUE_TEMPLATE/` 含 bug_report/feature_request 模板。

## 4. 目录结构与代码组织

顶层为扁平单包结构，按"职责层"划分目录（README「目录对应说明」与 tarball 实测一致）：

| 目录/文件 | 职责 | 规模 |
|---|---|---|
| `run.py` | 唯一入口，argparse 三个子命令：r（主流程）/c（CDN 过滤）/t（通知通道自测） | 30 行 |
| `init/` | 主运行目录：`select_ticket_info.py`（252 行，主循环调度）+ `login.py`（135 行，GoLogin 类） | 3 项 |
| `inter/` | 12306 接口编排层，22 个模块与下单流水线各步骤一一对应 | 23 项 |
| `config/` | 配置与公共设施：urlConf 端点注册表、通知通道、时间同步、日志、枚举消息、设备指纹参数 | 10 项 |
| `agency/` | CDN/代理池：`cdn_utils.py`（101 行）、`agency_tools.py`（107 行）、proxy_list 占位文件 | 3 项 |
| `verify/` | 验证码识别层：本地模型加载、图像预处理、训练辅助 | 4 项 |
| `myUrllib/` | 自研 HTTP 客户端（HTTPClient：cookie/headers/CDN 管理） | 3 项 |
| `myException/` | 7 个业务异常类型（乘车人、密码、余额、配置、无票等） | 7 项 |
| `UnitTest/` | `TestAll.py` 单测试文件 | 2 项 |
| `TickerConfig.py` | 用户配置文件（157 行，py 文件即配置） | — |
| `cdn_list` / `filter_cdn_list` | CDN 候选 IP 列表 1829 条 / 过滤后 559 条（行数实测） | — |
| `station_name.txt` | 车站电报码映射数据（约 105KB） | — |
| 根目录 | 两个 .h5 模型、LICENSE、README、Update.md、Docker 三件套 | — |

组织方式评述：典型的"接口适配层"单体脚本项目，无包管理工程（无 setup.py/pyproject），依赖根目录相对导入。代码总量小（Python 源码约 30 个文件、最大单文件 252 行），体积大头是模型二进制。命名有历史痕迹：配置文件叫 `TickerConfig.py`（非 Ticket），接口类 `ChechFace`（Check 拼写错误）。

## 5. 核心架构与关键实现

### 5.1 主循环调度（init/select_ticket_info.py）

`run.py` 的 r 子命令进入 `select().main()`，启动序列为：初始化 CDN 代理池 → 设备指纹参数请求（`config/getCookie.py` getDrvicesID）→ 登录（`init/login.py` GoLogin）→ 后台线程校验用户（`inter/CheckUser.py`）→ 拉取乘车人（`inter/GetPassengerDTOs.py`）→ 进入 `while 1` 查询主循环。

主循环每轮：构造 `inter/Query.py` 的 query 对象发送余票查询，有票则按配置分流——正常下单走 `inter/AutoSubmitOrderRequest.py`（快速下单）或 `inter/SubmitOrderRequest.py`（普通下单），候补订单走 `inter/ChechFace.py` 人证校验链路；无票则随机休眠后进入下一轮。支持两种模式（TickerConfig.ORDER_MODEL）：预售模式下在配置的整点前自旋等待（误差注释约 0.004s），捡漏模式按随机间隔轮询。异常体系靠 `myException/` 的 7 类异常终止循环。

```python
# init/select_ticket_info.py main() 片段
if secretStr:  # 正常下单
    if TickerConfig.ORDER_TYPE == 1:  # 快速下单
        a = autoSubmitOrderRequest(selectObj=self, ...)
        a.sendAutoSubmitOrderRequest()
    elif TickerConfig.ORDER_TYPE == 2:  # 普通下单
        sor = submitOrderRequest(self, secretStr, ...)
        sor.sendSubmitOrderRequest()
elif secretList:  # 候补订单
    c = chechFace(self, secretList, train_no)
    c.sendChechFace()
```

### 5.2 接口编排层（inter/，22 模块）

`inter/` 以"一个模块封装一个端点步骤"的方式组织，命名即流程：登录（LoginAysnSuggest/LoginConf/GetPassCodeNewOrderAndLogin/CheckRandCodeAnsyn/GetRandCode）、下单（AutoSubmitOrderRequest/SubmitOrderRequest/CheckOrderInfo/GetRepeatSubmitToken）、排队确认（ConfirmSingleForQueue/ConfirmSingleForQueueAsys/GetQueueCount/GetQueueCountAsync）、出票等待（QueryOrderWaitTime/queryMyOrderNoComplete）、候补（ChechFace/PassengerInitApi/ConfirmHB/GetSuccessRate）。每个模块均为"构造参数对象 + sendXxx() 发送"的统一形态，端点定义集中在 `config/urlConf.py` 的 46 个端点字典中。这种设计使失效时的抢修点收敛到 urlConf + 对应 inter 模块两处。

### 5.3 CDN/代理池层（agency/ + 根目录数据文件）

- `agency/cdn_utils.py`：CDNProxy 继承 threading.Thread 并发测速（run()），`filterCdn()` 为 run.py c 子命令入口，`open_cdn_file()` 加载列表，`sortCdn()` 排序；产出 `filter_cdn_list`（快照中 559 条，源列表 `cdn_list` 1829 条 IP）。
- `agency/agency_tools.py`：proxy 类提供代理 IP 的获取/过滤（get_proxy/filter_proxy/get_filter_proxy），代理 IP 由用户填入 `agency/proxy_list`（快照中为空文件），配合 `TickerConfig.py` 的 IS_PROXY 开关与 `UnitTest/TestAll.py` 的 testProxy 用例。
- 消费侧：`myUrllib/httpUtils.py` 的 HTTPClient 在构造时接收 cdnList，cdn 为 property，主循环随机选取前 5 条之一（`init/select_ticket_info.py` `self.httpClint.cdn = self.cdn_list[random.randint(0, 4)]`）。

机制定位：CDN 轮询仅用于查询侧的连接优化（Update.md 2018.1.29 引入），与代理池是两套独立机制。

### 5.4 验证码处理层（verify/，存在性描述）

项目存在完整的验证码处理模块分层：`verify/localVerifyCode.py`（115 行，Verify 类，职责边界为加载文本与图像两个本地 Keras 模型并给出识别结果）、`verify/pretreatment.py`（102 行，图像预处理/切分）、`verify/mlearn_for_image.py`（94 行，VGG16 迁移学习训练辅助）；模型文件置于根目录（README 注明来源 zhaipro/easy12306）。另有一条云打码通道：`TickerConfig.py` 中 AUTO_CODE_TYPE 支持本地/云两种模式，云模式指向 docker-compose.yml 编排的 `yinaoxiong/12306_code_server` 自托管服务；`inter/GetRandCode.py`、`inter/CheckRandCodeAnsyn.py` 负责验证码的获取与校验回传。登录侧的设备指纹参数构造集中在 `config/getCookie.py`（195 行，request_device_id/request_alg_id 等函数）。本文按约定不展开各识别/参数构造算法的实现细节。

## 6. 构建与运行

来自 README.md 与 Dockerfile 的客观记录（仅陈述，非使用建议）：

- 常规方式：`pip3 install -r requirements.txt`；`run.py` 三子命令 r/c/t（c 为 CDN 过滤前置步骤，t 为通知通道自测）。
- Docker 方式：`docker-compose.yml` 定义 ticket（build 自 Dockerfile37，镜像 ticket:v1.2.004）与 captcha（yinaoxiong/12306_code_server:amd64）双服务，`docker-compose up --build -d` 启动。
- 测试：`python -m unittest UnitTest/TestAll.py`（仅一个测试类，见第 7 节）。
- 注：Dockerfile 遗留 python:2.7.15 基础镜像且 pip 安装了 pyspider（与本项目无关的疑似复制粘贴残留），实际编排使用 Dockerfile37。

## 7. 测试与工程质量

- 测试规模：全项目仅 `UnitTest/TestAll.py` 一个测试文件（约 120 行），unittest 框架，7 个用例：testProxy、testEmail、testServerChan、testUserAgent、testVerfyImage（本地打码）、testRemoteVerfy（云打码）、testCdn。全部为依赖外部服务/真实凭据的联调式冒烟测试，无单元级隔离测试，无 CI 配置（`.github/` 仅含 ISSUE_TEMPLATE，无 workflows）。
- 代码质量观察：单文件行数克制（最大 252 行），异常类型化（7 个专用异常类）；但存在语法遗留问题——`init/select_ticket_info.py` 中 `if TickerConfig.ORDER_MODEL is 1` 使用 `is` 比较整数（Python3 语法警告），类名拼写错误（ChechFace），配置文件名拼写不一致（TickerConfig）。中英文混杂的面向过程风格，注释密度较高（多为中文说明）。
- 无 lint/format 工具配置，无代码覆盖率设施。

## 8. 社区与维护状态

- 活跃度时间线（Update.md + gh api commits 双源印证）：
  - 2017-05：仓库创建，Update.md 首条 2017.5.13；
  - 2018-01~02：高频抢修期（月内十余条更新：自动配置、邮箱通知、CDN 轮询、多账号等）；
  - 2019-01~09：预售踩点、候补订单、放弃 py2.7、yaml 配置迁改为 py 配置；
  - 2019-12：Docker 镜像体积优化（2.2GB→760MB，commit 2019-12-26）、唯一 Release 1.0（2019-12-06）、最后一次功能合并（2019-12-27，PR #633）；
  - 2021-01-04：master 最后提交「站点信息更新」（作者 gzldc），此后默认分支无代码演进；
  - 2022-11-21：dependabot/pip/tensorflow-2.9.3 分支最后提交（依赖升级 PR 分支，未合入 master）；
  - 2023-04-02：仓库级最后推送（gh api pushed_at；该推送未在默认分支产生新提交，具体推送对象未确认）；
  - 2026-11-21：gh api 核实 archived=true。
- Issue/PR 规模：issue 总数 689（其中 open 278，多为停更后"失效无法使用"类反馈），PR 总数 145（open 55，多数为停更后的社区 PR 堆积未合并）。
- 贡献者结构：具名贡献者 21 人（contributors API per_page=100 返回 21），含匿名贡献者约 31（anon=true 分页 Link 头 page=32）；主体代码由作者 testerSunshine（gzldc）完成，2019-12 前的合并记录显示 Renzhi 等少量协作者参与 Docker/代码规范化贡献。
- Release 节奏：仅 1 个 Release（1.0，2019-12-06），tag 同名，未建立版本发布体系。
- README 维护有 9 个 QQ 交流群（多数已满员），为项目高峰期的主要用户社群载体。

## 9. 亮点与风险

亮点：

1. 接口编排层设计清晰：46 个端点集中注册于 `config/urlConf.py`，`inter/` 22 模块与业务流水线一一对应，站点接口变动时的修复点高度收敛（文件级可定位）。
2. 自研轻量 HTTP 层：`myUrllib/httpUtils.py` 的 HTTPClient 统一管理 cookie/headers/CDN 轮换，配合 `agency/cdn_utils.py` 的多线程 CDN 测速过滤，形成查询侧的完整网络栈。
3. 验证码处理采用"本地 CNN 模型 + 可替换云服务"双通道架构（`verify/` + docker-compose captcha 服务），模块边界明确，模型可独立替换（README 注明来源与替代方案）。
4. 配置即代码的极简形态：`TickerConfig.py` 单文件覆盖出行信息、打码、通知、下单模式等十余类配置项，无额外配置解析依赖。
5. Update.md 作为变更日志完整保留了 2017-2019 年每次站点失效与修复的时间线，对研究"爬虫/自动化工具与站点对抗演进"是少见的实证材料。

风险/局限（中性客观表述）：

1. **平台条款冲突风险**：作为针对商业购票站点的自动化批量查询/下单工具，其运行方式与目标平台的服务条款存在冲突可能，存在账号被限制等后果；此类工具的合规性由使用者自行判断。
2. **敏感信息本地明文**：12306 账号凭据与邮箱/通知配置以明文形式存放于 `TickerConfig.py`，存在本地泄露面。
3. **依赖栈陈旧且有已知漏洞**：快照锁定 requests 2.18.4、tensorflow 1.14 等旧版本（requirements.txt），项目归档后不再有安全更新；云打码依赖社区志愿者自托管服务器（README/issues 招募），存在资源滥用与可信边界问题。
4. **项目已实质性失效**：archived=true、master 最后实质提交 2021-01-04，12306 站点接口/页面此后持续变动，快照代码对当前站点环境已不可用（issue 区大量失效反馈佐证）；仅具架构研究价值。

## 10. 总结评价

testerSunshine/12306 是 2017-2019 年国内"抢票工具潮"的代表作：以极小的代码量（约 30 个源码文件）完成了一套完整的"端点注册表 + 接口编排层 + 主循环调度 + 打码双通道 + CDN/代理网络栈"架构，33k 星的积累主要来自春运场景的刚性需求与 QQ 群社群传播。它的衰落路径同样典型——本质是"与站点环境持续对抗"的工具，作者 2019 年底停更后即失去修复能力，2021 年后彻底失效，最终 2023-04 停止推送并归档。对今天的读者，其价值不在使用（已失效且不建议），而在：研究自动化工具与站点风控的对抗史（Update.md 是一份完整的时间线标本），以及借鉴其"端点集中注册 + 编排层单步封装"的接口适配设计。

适合谁读：研究爬虫工程架构、API 适配层设计模式、开源灰区工具生命周期演进的开发者与研究者。
