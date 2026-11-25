# huiyadanli/RevokeMsgPatcher 项目分析

> 星数 33500（快照 2026-10-24）｜主语言 C#｜档位 20k-50k｜分析日期 2026-11-14
>
> gh api 核实：当前 39239 星、4100 forks、GPL-3.0、未归档、pushed_at 2026-11-08（分析时活跃）

## 1. 项目速览

一句话定位：Windows PC 版微信/QQ/TIM 的消息防撤回补丁工具，通过修改客户端本地 DLL 二进制使"撤回消息"失效，附带微信多开功能与独立多开小工具。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | RevokeMsgPatcher（微信/QQ/TIM防撤回补丁） | README.md |
| 星数 | 33500（快照）；39239（2026-11-14 gh api） | 任务参数 / gh api |
| 语言 | C#（WinForms，.NET Framework 4.5.2） | RevokeMsgPatcher.csproj |
| 许可证 | GPL-3.0 | LICENSE / gh api |
| 当前版本 | 2.2（AssemblyInfo.cs `AssemblyVersion("2.2")`） | Properties/AssemblyInfo.cs |
| 最近发布 | 2026-11-08（release 2.2，共 22 个 release） | gh api releases |
| 创建时间 | 2019-07-21 | gh api |
| 平台 | 仅 Windows（Win7+，不支持 XP） | README.md |

## 2. 项目是什么，解决什么问题

PC 版微信/QQ/TIM 收到消息撤回指令时会从本地界面删除已显示的消息。本项目通过给客户端安装"防撤回补丁"使撤回失效——用户仍能看到被撤回的消息内容。工具定位为一次性本地二进制补丁器：运行 GUI、定位客户端安装目录、对关键 DLL 做字节级修改并备份原文件（README.md，RevokeMsgPatcher/Modifier/AppModifier.cs）。

补丁对象随客户端大版本演进：旧微信 3.x 修改 `WeChatWin.dll`，微信 4.x 修改 `Weixin.dll`，QQ 改为 NT 架构后修改 Electron 原生模块 `wrapper.node`（RevokeMsgPatcher/Modifier/ 下 WechatModifier.cs、WeixinModifier.cs、QQNTModifier.cs 的 GetVersion() 实现）。早期还支持 QQ（IM.dll）/TIM/QQLite，2.2 版配置中已清理旧架构目标（Data/2.2/patch.json 仅含 Wechat/Weixin/QQNT 三项）。

作者在 README 中明确自述定位："本人不参与方法寻找，仅做特征搬运"——逆向方法来自社区公开成果（早期 wechat_anti_revoke、QQNT 特征来自 NapNeko/NTQQAntiRecall、微信 4.0 特征来自 zetaloop/BetterWX，见 README.md 致谢段）。本项目承担的工程价值是把零散的逆向结论产品化：版本适配、特征码定位、失败回滚、备份还原、多 CDN 分发。

附属产物 RevokeMsgPatcher.MultiInstance 是一个独立的通用微信多开工具（清理微信互斥句柄实现多开，不依赖微信版本，RevokeMsgPatcher.MultiInstance/README.md、WechatProcess.cs）。

## 3. 技术栈与依赖

- 语言/框架：C# + WinForms，目标 .NET Framework 4.5.2，WinExe 输出（RevokeMsgPatcher/RevokeMsgPatcher.csproj）
- 依赖：仅 .NET BCL 引用（System.Windows.Forms、System.Web.Extensions 的 JavaScriptSerializer 做 JSON 反序列化、System.Management 取设备信息等），csproj 中无任何第三方 NuGet 包；JSON 补丁数据用原生 JavaScriptSerializer 解析而非 JSON.NET
- 构建链：MSBuild + nuget restore；CI 为 GitHub Actions（windows-2022，.github/workflows/build.yml）与 AppVeyor（appveyor.yml）双轨
- 仓库方案：单个 .sln 含 4 个 C# 工程（RevokeMsgPatcher.sln，VS 17 格式）

## 4. 目录结构与代码组织

| 目录 | 职责 | 出处 |
|---|---|---|
| RevokeMsgPatcher/ | 主程序（GUI 补丁器），含 Matcher/Matcher 特征匹配、Modifier/ 六个目标应用修改器、Model/ 补丁数据模型、Utils/ | tarball 顶层 |
| RevokeMsgPatcher.Assistant/ | 补丁配置生成辅助工具 + `Data/<版本>/patch.json`（运行时分发的补丁信息本体，0.7~2.2 共 16 个历史版本存档） | tarball 顶层 |
| RevokeMsgPatcher.MultiInstance/ | 独立的微信多开小工具（清互斥句柄） | MultiInstance/README.md |
| RevokeMsgPatcher.Launcher/ | 启动器辅助工程 | RevokeMsgPatcher.sln |
| tests/Matcher.Tests/ | 匹配器回归测试（自写断言，无测试框架） | tests/Matcher.Tests/README.md |
| .github/ | CI（build.yml）与 issue 模板（含"版本支持"专用模板） | tarball |
| Images/ | README 与 wiki 截图 | tarball |

组织方式为经典 WinForms 单仓多工程：主程序 + 2 个附属工具 + 启动器，共 190 个文件（tarball 统计），代码规模小（核心 C# 源约 40 个文件）。补丁数据与补丁引擎分离是关键设计：特征码全部在 JSON 配置中，更新特征通常不需要改代码发版。

## 5. 核心架构与关键实现

### 5.1 双通道补丁信息：SHA1 精确匹配 + 特征码版本范围

`AppModifier.ValidateAndFindModifyInfo`（RevokeMsgPatcher/Modifier/AppModifier.cs）是补丁决策核心。对每个目标 DLL 依次尝试：文件 SHA1 命中配置的"补丁前 SHA1"→ 采用精确替换表；SHA1 命中"补丁后 SHA1"→ 判定已打过补丁；否则回退到特征码通道，按当前文件版本号落在 `StartVersion`~`EndVersion` 区间选取 `CommonModifyInfo`（`IsInVersionRange` 用 VersionUtil 比较版本）。两通道都失败时区分"版本不支持"与"版本支持但校验不通过（可能被其他补丁改过）"两种错误（AppModifier.cs 中 BusinessException("not_support"/"maybe_modified") 分支）。

2.2 配置已完全使用特征码通道（Data/2.2/patch.json 中 FileModifyInfos 均为 null），SHA1 通道保留兼容旧配置。

### 5.2 特征码匹配引擎：Boyer-Moore + 单字节通配符

- `Matcher/BoyerMooreMatcher.cs`：标准 Boyer-Moore（坏字符 + 好后缀双启发式）字节搜索，`EnumerateMatches` 以迭代器流式产出命中位置，避免按文件大小分配结果数组（注释明确"仅保存实际命中的位置"）。
- `Matcher/FuzzyMatcher.cs`：在 BM 之上实现通配符（0x3F/'?'）匹配。优化点是先用模式中"最长连续固定片段"（GetHead）做 BM 快速定位候选区，再全串校验，注释写明"原先只使用首个通配符前的头串；现在选取最长连续固定片段，减少候选数量"。
- `Matcher/ModifyFinder.cs`：把特征码配置翻译为实际修改——全文件读入后对每条 `ReplacePattern`（Search/Replace/Category/Tips）做 `MatchAll`，排除"内容已是替换串"的命中点，生成 `Change(offset, replaceBytes)` 列表；匹配数异常时区分"已安装/部分安装/特征变化"三种报错（match_already_replace、match_part_replace、match_inconformity）。

补丁内容的本质属于公开逆向知识范畴的机器码条件跳转改写：以 Data/2.2/patch.json 中微信 3.9.12.56 x86 防撤回特征为例，Search `[131,196,16,132,192,116,10,...]` → Replace 将其中的 `0x74`（je 条件跳转）替换为 `0xEB`（jmp 无条件跳转），即让"执行撤回"分支被跳过；微信 4.x 的 x64 特征同理（如 `72,41,134,...` 将 add 改为 sub，0x33→0x2B 类运算符替换）。JSON 中大量使用 0x3F 通配符（如 `[72,137,134,63,63,63,63,76,...]`）以容忍地址操作数的版本间浮动，这是同一特征覆盖多个小版本的关键。

### 5.3 补丁配置分发：多 CDN 回退 + 版本化 JSON

补丁信息不在安装包内固化，启动时经 `Utils/HttpUtil.GetPatchJsonAsync` 拉取（Utils/HttpUtil.cs）：按序尝试阿里云 OSS、ghproxy 镜像、gitmirror、jsdelivr、GitHub raw 五个源（URL 模板按程序 PatchVersion 拼接 `RevokeMsgPatcher.Assistant/Data/<N.N>/patch.json`），内容须含 `LatestVersion` 字段才接受；失败仍可用打包时内置的 `Properties.Resources.PatchJson` 兜底（FormMain.cs InitModifier）。`Model/Bag.cs` 是配置根模型（Apps 字典 + LatestVersion + Notice + PatchVersion）。程序版本低于配置的 LatestVersion 时在 UI 提示更新（FormMain.cs FormMain_Load）。

### 5.4 补丁执行与安全网

`Modifier/FileHexEditor.cs` 管单文件生命周期：版本号/SHA1 惰性求值缓存、备份为 `*.h.bak`（同版本备份不覆盖）、替换走 `FileUtil.EditMultiHex`；`AppModifier.Patch()`（AppModifier.cs）先全量备份再逐文件打补丁，任一失败则把已打补丁的文件全部还原（done 列表回滚），还原时校验备份版本与当前版本一致性并弹窗确认。`Program.cs` 非管理员时通过 `runas`  Verb 自提权重启（写入 DLL 需要管理员权限）。

## 6. 构建与运行

来自 tests/Matcher.Tests/README.md 与 CI 配置（本分析仅静态阅读，未执行构建）：

```powershell
# 构建（VS Developer PowerShell 或 CI 同款）
nuget restore
msbuild /m /p:Configuration=Release

# 匹配器回归测试
msbuild tests\Matcher.Tests\Matcher.Tests.csproj /p:Configuration=Release /nologo /v:minimal
.\tests\Matcher.Tests\bin\Release\Matcher.Tests.exe

# 运行：Windows 7+，需 .NET Framework 4.5.2+，以管理员身份运行 RevokeMsgPatcher.exe
# 打补丁前需关闭微信/QQ/TIM（README.md 使用方法）
```

发布产物：GitHub Releases 提供 zip（当前 2.2），另有蓝奏云/百度云备用下载（README.md）。

## 7. 测试与工程质量

- tests/Matcher.Tests 是手写断言式回归测试（自述"匹配器回归测试直接编译生产代码，无第三方测试依赖"，tests/Matcher.Tests/README.md）：覆盖边界（模式长于文件、首尾重叠、通配符位置、负偏移/溢出偏移）、20,000 组固定随机种子数据与参照实现交叉验证、MatchNotReplaced 过滤已打补丁候选、以及内存回归（要求 8 MiB 缓冲区无命中扫描的分配低于 1 MiB，防止回退到按文件大小分配数组的旧实现）；还会读取 Data/2.1/patch.json 校验真实特征偏移有效性。测试尚无参数化框架与 CI 内的独立执行步骤（build.yml 只构建并上传产物，tests 工程在 sln 内会随 CI 构建）。
- 工程质量观察：错误处理细致（BusinessException 错误码体系：installed/not_support/maybe_modified/match_*，UI 分场景提示）；BCL-only 零依赖利于在裸机运行；存在少量技术债痕迹（ModifyFinder.FindChanges 顶部自注"TODO 该逻辑需要优化！"、AppModifier 中 [Obsolete] 方法与注释掉的旧代码）。
- 遥测：内置 GA4 匿名上报（Utils/GAHelper.cs，api_secret 与 measurement_id 硬编码于源码），记录打开主界面、选择的目标应用与版本、打补丁成功/失败事件；匿名 client id 由 MAC 地址哈希生成（Utils/Device.cs）。

## 8. 社区与维护状态

- 维护活跃，并非停更项目。gh api 证实：22 个 release 从 2019-07-22 持续到 2026-11-08；最近提交（2026-11-08，作者"辉鸭蛋"）包括"支持微信3.9 x86, 支持QQNT 9.9.36 (群聊私聊)。清理历史数据"、"优化匹配，删除旧的防撤回功能"、"删除早期 LiteLoaderQQNT 相关代码"，与源码中 Matcher 性能优化注释、2.2 配置清理旧目标的现状互相印证。
- release 节奏近年约每年 2~3 个：2.2（2026-11）、2.1（2025-08）、2.0（2024-11）、1.9（2024-09）、1.8（2024-06）、1.7（2023-10）、1.6（2022-08）、1.5（2022-05）（gh api releases）。
- 贡献者结构：作者 huiyadanli 307 次贡献占绝对主导，另有 7 名贡献者（Constaline 3 次，其余各 1 次）；社区贡献以 PR 提交新版本特征码为主（如 #1209 支持微信 4.1.12.26、#1226 使 4.1.12.55 特征通配化，gh api commits）。仓库配置了专门的"版本支持"issue 模板（new_version.yml，指派给作者）用于收集未适配版本反馈，config.yml 将 Mac 需求引流至 Sunnyyoung/WeChatTweak-macOS。
- 特征来源依赖上游社区逆向项目（README 致谢段），本项目是"消费方+工程化方"。issue 当前 57 个开放（gh api），符合此类跟版本赛跑工具的常态。

## 9. 亮点与风险

亮点：

1. 补丁数据与引擎分离：特征码全部外置于 JSON 并支持运行时多 CDN 更新，适配新客户端版本通常只需更新配置不发版（Data/2.2/patch.json、Utils/HttpUtil.cs）。
2. 自研特征匹配引擎质量高：Boyer-Moore + 最长固定片段预筛 + 单字节通配符，并有 2 万组随机数据交叉验证和内存分配回归约束（Matcher/、tests/Matcher.Tests/）。
3. 完整的安全网：SHA1 双向校验识别"未打/已打/被其他补丁改过"状态，.h.bak 备份、失败整体回滚、版本不一致还原确认（AppModifier.cs、FileHexEditor.cs）。
4. 零第三方依赖的 .NET Framework 4.5.2 目标，老旧 Windows 环境可用性极高。
5. 特征通配符化显著延长单条特征的覆盖版本区间（如 4.1.12.55 特征偏移通配后兼容 4.1.12.x，PR #1226）。

风险/局限：

1. 合规争议：修改微信/QQ/TIM 客户端二进制（WeChatWin.dll/Weixin.dll/wrapper.node）规避消息撤回，明确违反目标客户端用户服务协议精神，属于第三方客户端修改类工具的固有争议；本项目代码 GPL-3.0 合规分发，但其用途的合规性由使用者自担，腾讯侧对改机行为的封禁风险始终存在。
2. 杀软误报与启发式拦截：修改第三方进程 DLL 的行为模式天然触发杀毒软件告警（README 自述"杀毒软件可能会弹出警告"），且此类工具常被安全软件误判，实际可用性受环境干扰。
3. 版本适配竞速问题：特征码与目标客户端二进制强耦合，客户端每次更新都可能失效；当前 2.2 配置对微信 3.x 仅覆盖 3.9.12.55~3.9.12.56，旧版本客户端不在覆盖范围（Data/2.2/patch.json）。
4. 遥测与凭据硬编码：GA4 api_secret 硬编码于公开源码（GAHelper.cs），遥测默认开启且无开关，对注重隐私的用户是负面项；匿名 id 源于 MAC 哈希（Device.cs）。

## 10. 总结评价

RevokeMsgPatcher 是"二进制补丁器"品类中工程质量突出的代表作：把社区逆向得出的防撤回特征沉淀为版本化 JSON 配置，配以自研 Boyer-Moore 通配符匹配引擎、SHA1/特征码双通道判定、备份回滚安全网和多 CDN 分发，用约 190 个文件的小仓库支撑了 2019 年至今对微信 3.x/4.x 与 QQNT 的持续适配，2026-11 仍随客户端版本更新（release 2.2）。它不生产逆向结论，而是让这些结论可靠、可还原、可分发。适合两类人：想研究"外置配置驱动二进制补丁"工具架构的开发者（Matcher/AppModifier/HttpUtil 三条线都很值得读），以及在受控环境中需要理解此类工具工作原理的安全研究者；而普通用户需要清楚它伴随的条款合规与账号风险。
