# VSCodium/vscodium 项目分析

> 星数 27500（快照 2026-10-24）｜主语言 Shell｜档位 20k-50k｜分析日期 2026-11-21
>
> 注：gh api 于 2026-11-21 复核星数为 34,202（与快照差异来自快照时点不同），仓库持续活跃。

## 1. 项目速览

**一句话定位**：VSCodium 不是 VS Code 的 fork，而是一套把 microsoft/vscode 源码自动构建成"无遥测、MIT 许可"桌面产品的**发布基础设施仓**——主体是 Bash 构建脚本 + GitHub Actions 多平台发布流水线，外加一套对上游源码的 patch 体系。

| 项目 | 值 |
|---|---|
| 名称 | VSCodium/vscodium |
| 星数 | 27,500（快照）/ 34,202（2026-11-21 gh api 核实） |
| 主语言 | Shell（约 40+ 个 Bash 脚本，`build.sh`、`prepare_vscode.sh` 等） |
| 许可证 | MIT（`LICENSE`，含 VSCodium contributors / Peter Squicciarini / Microsoft 三段版权声明） |
| 创建时间 | 2018-08-13（gh api） |
| 最近活跃 | 2026-10-31 有 push（gh api）；最近 release 1.143.17232（2026-10-28） |
| 产物形态 | 不是应用源码仓——仓内无应用实现代码，只有构建编排、补丁、图标资源和 CI 工作流 |

## 2. 项目是什么，解决什么问题

VS Code 官方二进制采用非 FLOSS 许可并内建遥测。微软维护者在 vscode issue #60 中的官方解释（`README.md` "Why Does This Exist" 一节完整引用）是：微软自己构建 VS Code 时，也是 clone vscode 仓库、覆盖一份定制 `product.json`（遥测、插件市场、logo 等），再按自有许可发布；直接从开源仓库构建出来的是"干净构建"，默认即 MIT 许可。VSCodium 的存在意义就是**把这一构建流程工程化、自动化**，让普通用户不必自己编译就能拿到 MIT 许可、遥测默认关闭的 VS Code 等价物。

因此这个仓库的真实形态是"产品流水线"而非"产品代码"：它每天检查上游 microsoft/vscode 是否有新版本（`README.md` 称 builds are run every day），有则锁定该 commit、浅克隆、套补丁、改 `product.json`、复用上游自己的 gulp 构建系统产出全平台安装包，发布到 GitHub Releases 并同步到各包管理器。上游无新版本时流水线提前退出（幂等判断见 `check_tags.sh`）。

整个体系刻意保持"零 fork"姿态：对上游代码的全部改动都以外挂 patch + `product.json` 字段注入 + 静态资源覆盖三种方式表达，构建完成后即丢弃。这使它能在上游高频变更（月度大版本 + 每日 insiders）下长期存活。仓库自身用 `upstream/stable.json` 记录最近一次成功构建的上游 commit，作为版本锁定与增量判断的依据（`upstream/stable.json`：`{"tag":"1.143.1","commit":"e6468682..."}`）。

## 3. 技术栈与依赖

- **Bash**：全部构建逻辑（顶层 17 个脚本 + `build/` 平台脚本 + `dev/` 开发工具），无 Node/Python 自有代码。
- **jq**：`product.json` / `package.json` 字段注入的核心工具（`prepare_vscode.sh` 的 `setpath`/`setpath_json` 函数）。
- **git apply**：补丁应用引擎（`utils.sh` `apply_patch()`）。
- **sed / @vscode/ripgrep**：全源码字符串替换与遥测端点清洗（`undo_teletemetry.sh` 用 rg 定位 `*.data.microsoft.com` 域名并替换为 `0.0.0.0`）。
- **GitHub Actions**：15 个工作流共 2,954 行 YAML（`.github/workflows/`），是事实上的 CI/CD 骨架；所有第三方 action 均以 commit SHA 锁定版本。
- **自建 Docker 构建代理**：Linux 各架构使用 `vscodium/vscodium-linux-build-agent:focal-{x64,arm64,armhf,riscv64,ppc64le}` 与 `crimson-loong64` 镜像（`publish-stable-linux.yml` build job 矩阵）。
- **上游构建系统**：构建本身复用 microsoft/vscode 仓库内的 gulp 任务（`build.sh` 调 `npm run gulp vscode-darwin-${VSCODE_ARCH}-min-packing` 等），本仓不重写打包逻辑。
- **工具链质量工具**：zizmor（Actions 安全审计）、ecformat（编辑器格式）、pinact（action 版本升级），由 `justfile` 定义 `ci-lint` 等入口。
- Node 版本由 `.nvmrc` 锁定；上游依赖（Electron、预构建扩展）在 `README.md` 中如实披露下载来源。

## 4. 目录结构与代码组织

顶层目录：

| 目录/文件 | 职责 |
|---|---|
| `build.sh` | 单平台构建总入口：prepare → gulp 打包 → REH/REH-web → CLI |
| `prepare_vscode.sh` | 核心脚本：资源覆盖 + product.json 注入 + 补丁应用 + npm ci |
| `get_repo.sh` | 上游版本解析与浅克隆（锁 commit） |
| `check_tags.sh` | 幂等检查：对照已有 Release/资产决定是否需要构建 |
| `undo_telemetry.sh` | 遥测端点全域清洗 |
| `update_upstream.sh` | 把本次构建的上游 tag/commit 回写进 `upstream/*.json`（GPG 签名提交） |
| `update_version.sh` | 写"版本 JSON"到 `VSCodium/versions` 仓库，供应用内更新器使用 |
| `release.sh` / `prepare_assets.sh` / `prepare_checksums.sh` | Release 创建、资产准备、校验和 |
| `build_cli.sh` | VS Code CLI（tunnel/remote server）构建 |
| `check_cron_or_pr.sh` / `get_pr.sh` | 按事件类型（cron/PR/dispatch）设定 SHOULD_BUILD/SHOULD_DEPLOY |
| `product.json` | 仓库级产品配置，与上游 product.json 深合并 |
| `announcements-*.json` | 欢迎页公告内容（注入 `gettingStarted.ts`） |
| `patches/` | 补丁体系：42 个根补丁 + 2 个 JSON 动作 + 7 个平台/质量子目录 |
| `src/stable/` `src/insider/` | 静态资源覆盖层：图标、desktop 文件、win32 ico 等 |
| `upstream/` | `stable.json` / `insider.json` 版本锁定 |
| `build/` | 平台打包脚本（linux/osx/windows/alpine 各一套） |
| `stores/` | snapcraft、winget 商店清单与发布脚本 |
| `dev/` | 维护者工具：`patch.sh`、`merge-patches.sh`、`update_patches.sh`、Docker 构建 |
| `docs/` | 13 篇文档（telemetry、patches、howto-build、extensions 等） |
| `.github/workflows/` | 15 个工作流：ci-build-{linux,macos,windows}、publish-{stable,insider}-{linux,macos,windows,spearhead}、lint、moderation |

组织方式评述：典型的"流水线分层"结构——顶层脚本负责通用编排，`build/<os>/` 承接平台差异，`patches/` 以"编号前缀 + 平台子目录 + 质量目录（insider）+ 用户目录（user）"四级顺序应用（`prepare_vscode.sh` 补丁循环）。stable/insider 双质量通道通过 `VSCODE_QUALITY` 环境变量全程分叉，几乎每个脚本都有对应分支逻辑。

## 5. 核心架构与关键实现

### 5.1 上游版本锁定与发布编排（spearhead 模式）

版本真相源是 `upstream/<quality>.json`（tag + commit）。`get_repo.sh` 的解析顺序：优先读锁定 JSON；`VSCODE_LATEST=yes` 时改查微软更新 API（`https://update.code.visualstudio.com/api/update/darwin/<quality>/000...0`）取最新 commit，然后 `git fetch --depth 1 origin "${MS_COMMIT}"` 浅取该 commit 并校验 tag 存在（`get_repo.sh`）。

发布编排采用"先头部队"模式：`publish-stable-spearhead.yml`（macOS arm64 runner）先完成一次完整构建，然后依次执行 `update_upstream.sh`（把新上游 commit 以 GPG 签名提交回本仓 `upstream/*.json`）、`prepare_src.sh`（发布源码 tar）、`upload_sourcemaps.sh`（sourcemaps 传到 `VSCodium/sourcemaps` 仓库），最后用 `peter-evans/repository-dispatch` 派发 `publish-stable` 事件，触发 Linux/Windows/macOS 三个工作流并行产出全部产物（`publish-stable-spearhead.yml`）。值得注意的是本仓 publish 工作流均无 `schedule` 触发器，每日调度源在仓外（README 称每日构建；具体调度器未确认）。

发布版本号内嵌构建时间：`RELEASE_VERSION = MS_TAG + TIME_PATCH`，其中 `TIME_PATCH=$(printf "%04d" $((年内天*24+小时)))`（`get_repo.sh`）。gh api 可验证：release tag `1.143.17232` = 上游 1.143.1 + 第 301 天（10-28）8 时，方案与代码完全吻合。

### 5.2 遥测剥离的三层体系

1. **配置默认值翻转**：`patches/00-telemetry-disable.patch` 把 7 类遥测相关设置的默认值改为关（telemetryService.ts 的 `TelemetryConfiguration.OFF`、崩溃报告、实验服务 assignmentService、自然语言搜索等，patch 涉及 12 个文件头）；`docs/telemetry.md` 给出完整清单（telemetry.telemetryLevel、enableCrashReporter、workbench.enableExperiments 等）。
2. **端点全域清洗**：`undo_telemetry.sh` 用 `@vscode/ripgrep` 在整个源码树搜索 `\.data\.microsoft\.com`，将 `//*.data.microsoft.com` 全部替换为 `//0.0.0.0`（覆盖 vortex.data.microsoft.com、mobile.events.data.microsoft.com 等硬编码端点）。
3. **不注入微软端点**：`prepare_vscode.sh` 只写入开源服务地址（见 5.3），`patches/00-cloud-remove.patch` 移除云端功能入口；`product.json` 中 `updateUrl` 指向 `raw.githubusercontent.com/VSCodium/versions`（自管更新源），实现更新检查也不回微软。

### 5.3 补丁引擎与补丁组织

补丁应用引擎在 `utils.sh`：

```bash
apply_patch() {
  cp $1{,.bak}
  replace "s|!!APP_NAME!!|${APP_NAME}|g" "$1"      # 8 个占位符模板替换
  ...
  if ! git apply --ignore-whitespace "$1"; then
    echo failed to apply patch "$1" >&2; exit 1
  fi
  mv -f $1{.bak,}
}
```

补丁支持 `!!APP_NAME!!`、`!!BINARY_NAME!!` 等 8 个占位符，应用前由 sed 实例化。另有 JSON 动作文件（`patches/52-ext-copilot-remove-it.json`、`patches/80-ui-disable-onboarding.json`）由 `apply_actions()` 解析执行 `remove` 动作直接删文件。应用顺序（`prepare_vscode.sh`）：`patches/*.json` → `patches/*.patch`（编号 00-80 控序）→ `patches/insider/` → `patches/${OS_NAME}/` → `patches/user/`。

根目录 42 个补丁按前缀语义分组：`00-`（遥测/品牌/设置/gallery/UI）、`10-12-`（版本与更新）、`20-22-`（keymap/policy/osproxy 换用社区维护库）、`30-`、`40-`（CLI 用 reh 归档）、`50-`（gulp 任务改进）、`60-`（恶意扩展安全选项）、`80-`（禁用 onboarding）。平台补丁集中在 `patches/linux/`（11 个，含 arch/deb 适配、41-45 号为 ppc64le/riscv64/loong64/s390x/arm32 新架构支持）与 `patches/windows/`（6 个，含 Win7 最低版本、AppX/右键菜单）。特殊后缀表达补丁状态：`.no`（暂禁用，如 `00-build-disable-mangle.patch.no`）、`.yet`（暂缓，如 `00-update-disable.patch.yet`）——这套状态标记本身就是上游兼容性维护的痕迹。

### 5.4 product.json 注入与法律边界处理

`prepare_vscode.sh` 用 jq `setpath` 逐字段改写上游 product.json，再与仓库级 `product.json` 深合并（`jq -s '.[0] * .[1]'`）。关键注入：

- **扩展市场**：`extensionsGallery` 指向 `https://open-vsx.org/vscode/gallery`，并带 `controlUrl` 指向 Eclipse publish-extensions 项目的管控清单（`prepare_vscode.sh`）；配套补丁 `00-settings-gallery.patch` 给上游补上 `itemUrl`/`latestUrlTemplate` 字段支持。官方口径（`README.md`、`docs/extensions.md`）：Visual Studio Marketplace 服务条款只允许 VS Code 产品使用，故 VSCodium 默认 Open VSX，并在文档中提供自建 gallery 的指引。
- **商标/署名**：所有产品标识从 Microsoft 系改为 VSCodium 系——`applicationName=codium`、`darwinBundleIdentifier=com.vscodium`、全套 `win32AppId` GUID、`serverApplicationName=codium-server`、tunnel 命名等；再用 sed 全局替换 `Microsoft Corporation → VSCodium`（`package.json`、`build/lib/electron.ts`）。Linux 打包元数据（debian `control.template`/`postinst.template`、rpm `code.spec.template`、`code.appdata.xml`）逐一改写维护者署名与官网链接，并改掉 postinst 中会添加微软 apt 源的行（`prepare_vscode.sh` Linux 分支）。`LICENSE` 采用 MIT 且并列保留微软版权声明，尊重上游代码归属。

### 5.5 平台构建矩阵

单次稳定版发布的 job 组织（`publish-stable-linux.yml`，614 行）：

- `check`（幂等判断）→ `compile`（x64 编译产物以 artifact 传递，避免每架构重复编译）→ `build`（6 架构矩阵：x64/arm64/armhf/riscv64/loong64/ppc64le，产出 deb/rpm/tar.gz）→ `anylinux-appimage`（x86_64/aarch64，Arch 容器内打 AppImage）→ `reh_linux`（7 架构远程服务器 REH，含 s390x）→ `reh_alpine`（musl）→ `aur` / `snap`（amd64+arm64）→ `deploy-repo-rudy`（触发 GitLab 上第三方 deb/rpm 仓库重建）/ `deploy-repo-dev`（派发 `VSCodium/repositories-linux`）。
- Windows（`publish-stable-windows.yml`，253 行）：check（ubuntu）→ compile（windows-2022）→ build（x64/arm64 矩阵，含 Inno Setup 安装器与 AppX/MSI，`build/windows/`）→ winget 发布 job。
- macOS（`publish-stable-macos.yml`）：x64/arm64 双矩阵，zip + dmg（`README.md` 支持平台清单）。
- 架构覆盖显著超越 VS Code 官方（官方仅 x64/arm64），riscv64/loong64/ppc64le/s390x 由 `patches/linux/41-45-arch-*.patch` 补齐。

## 6. 构建与运行

本仓不构建自身，而是编排上游。本地复现（`docs/howto-build.md` 与脚本头部环境变量）：

```bash
export VSCODE_QUALITY=stable        # 或 insider
export VSCODE_ARCH=x64
export OS_NAME=linux                # osx / windows
./get_repo.sh                       # 锁定并浅克隆上游 commit
./prepare_vscode.sh                 # 覆盖资源+注入 product.json+套补丁+npm ci
./build.sh                          # gulp 打包（上游构建系统）
./prepare_assets.sh && ./release.sh # 资产与发布
```

维护者另有 Docker 通道 `dev/build.sh` / `dev/build_docker.sh`（构建 `vscodium-linux-build-agent` 镜像内运行）。CI lint：`just ci-lint`（zizmor）、`just lint-ec`（ecformat）。不运行测试（见第 7 章）。

## 7. 测试与工程质量

- **无传统测试**：全仓无单元/集成测试目录——对构建编排仓而言，正确性由"产物可安装、可启动"这一端到端事实保证；`ci-build-{linux,macos,windows}.yml`（430/91/166 行）在 PR 时真实跑构建（`check_cron_or_pr.sh` 对 PR 强制 SHOULD_BUILD=yes、SHOULD_DEPLOY=no），是最接近"测试"的机制。
- **幂等与防错发**：`check_tags.sh` 对照 GitHub Release 的 tag 与资产清单（含 `-src.tar.gz`、校验和等 spearchead 资产检查），上游无新版本则整条流水线复用已有版本退出，避免重复发布。
- **CI 安全实践突出**：全部第三方 action 以完整 commit SHA 锁定；启用 zizmor 工作流安全审计（`lint-zizmor.yml`、`.github/zizmor.yml`）与 `pinact` 自动升级；发布类 job 用 `environment: publish` + 独立 `STRONGER_GITHUB_TOKEN` 最小权限分离（`permissions: {}` 与 `contents: write` 逐 job 声明）；`update_upstream.sh` 用 GPG 签名提交。
- **代码质量观察**：脚本普遍有 shellcheck disable 注记（说明接入过 shellcheck）；GNU/BSD sed 差异、跨平台 rg 路径分隔符等坑在 `utils.sh`/`undo_telemetry.sh` 中有显式分支处理，工程细节成熟。

## 8. 社区与维护状态

- gh api 核实（2026-11-21）：34,202 stars、1,933 forks、238 watchers、149 open issues；贡献者 132 人（contributors 分页计）；创建于 2018-08，最近 push 2026-10-31，社区维护、活跃。
- **发版节奏**：跟随上游。上游发月度 stable 后数日内跟进（1.135→1.126→1.121 各差约 1.5-2 个月，符合 VS Code 月度周期）；同一上游版本会因补丁修复重发（1.143.17232 与 1.143.17183 相隔两天）。tag 数字尾缀即构建时间（见 5.1），可从 tag 名直接读出发版时刻。
- **维护者结构**：README 致谢区提到 logo（jaredreich）、CDN（PalinuroSec）、Mac 构建机（MacStadium 捐赠）；Chocolatey 通道致谢 @Thilas、AUR、snap、deb-rpm 仓库均由社区成员分摊维护——分发层是典型的"核心仓 + 众包分发"结构。
- **与衍生项目的关系（客观记录）**：全仓 README/docs 检索，未提及 code-server、Cursor 等 VS Code 衍生产品（仅 `docs/extensions.md` 在 Open VSX 溯源处引用过 Gitpod 博客）。VSCodium 与它们是并列的独立分发：code-server/cursor 基于自身 fork，而 VSCodium 坚持"零 fork、纯构建产物"路线，定位是面向终端用户的桌面无遥测版本，也是"如何合法分发 MIT 许可 VS Code"的参考实现。
- 分发渠道广度：官方 Releases + brew/winget/choco/scoop/snap/flatpak/AUR + 社区 apt/dnf/zypper 仓库（`README.md` Download/Install 一节）。

## 9. 亮点与风险

**亮点**

1. 补丁驱动的"零 fork"上游跟随架构：42+ 补丁 + product.json 注入 + 资源覆盖三层外挂，使月度上游升级的维护成本可控，是大型项目"构建期定制"的教科书案例（`patches/`、`prepare_vscode.sh`）。
2. 遥测剥离手段完整且可审计：设置默认值翻转、硬编码端点正则清洗、不注入微软端点三层叠加，全部改动以明文 patch/脚本呈现，可逐行复核（`00-telemetry-disable.patch`、`undo_telemetry.sh`）。
3. 架构覆盖极广：8 个 CPU 架构（含 s390x/riscv64/loong64/ppc64le），远超上游官方支持面（`publish-stable-linux.yml` 矩阵）。
4. CI 安全与工程纪律一流：SHA 锁定 action、zizmor 审计、GPG 签名版本提交、environment/token 分权、幂等防重发。
5. 文档完备：13 篇 docs 覆盖遥测口径、补丁说明、自建市场、迁移指南，法律与隐私立场表述清晰。

**风险/局限**

1. 补丁脆弱性：每次上游变更都可能使 `git apply` 失败并阻塞全平台发布；`.no`/`.yet` 后缀补丁和 insider 专属补丁表明这是一项持续的人工对抗性维护。
2. 法律边界依赖微软容忍：产品重命名规避了商标冲突，扩展市场切换规避了 Marketplace 条款，但整条流水线的前提取决于上游构建体系与接口的长期稳定。
3. 无自动化质量网：补丁语义正确性（不只是可应用性）依赖人工审查与用户反馈，回归只能靠事后发版修复。
4. 发布链强依赖 GitHub Actions 与多个 secrets/第三方仓库（versions、sourcemaps、repositories-linux、GitLab 触发器），任一环节故障都会造成部分平台缺货。

## 10. 总结评价

VSCodium 是"用构建基础设施而非 fork 改造大型上游项目"的标杆仓库：它把微软官方亲口描述的内部构建流程（clone + 定制 product.json）逆向工程成一套可公开审计、每天自动运行、覆盖 8 架构与 7 种分发渠道的流水线，用 42+ 个外挂补丁和三层遥测剥离把 VS Code 还原为纯粹的 MIT 产品。它不适合读"编辑器实现"（应用代码在上游 microsoft/vscode），但对 DevOps/发布工程实践者、研究软件供应链与遥测合规的人、以及想给大型上游项目做无 fork 定制分发的维护者，这个仓的脚本组织、幂等发布设计、CI 安全纪律都有极高的参考价值。
