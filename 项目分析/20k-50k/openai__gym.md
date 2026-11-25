# openai/gym 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-17

## 1. 项目速览

一句话定位：OpenAI Gym 是强化学习（RL）环境的标准 API 定义者——用 `reset/step/action_space/observation_space` 四件套把"智能体"与"环境"解耦，是 RL 领域近十年最接近"事实标准"的接口规范（现已被 Farama 基金会的 Gymnasium 接棒，本仓库已归档只读）。

| 项目 | 事实 |
| --- | --- |
| 名称 | openai/gym |
| 星数 | 39500（快照）；gh api 实测 37239（2026-11-17） |
| 语言 | Python（纯 Python，`py.typed` 全量类型标注） |
| 许可证 | MIT（Copyright 2016 OpenAI；`LICENSE.md` 末尾附带 MuJoCo 模型模型的 Roboti LLC 声明，故 GitHub SPDX 显示 NOASSERTION） |
| 版本 | 终版 0.26.2（`gym/version.py`），2022-10-04 发布 |
| 创建 | 2016-04-27（gh api） |
| 归档状态 | GitHub 已标记 archived（gh api `archived:true`）；最后功能提交 2022-10-04，README 更新至 2023-01-30，2026-03-26 仅有一次 CI workflow 固化提交 bc21295 |
| 贡献者 | 约 385 人（gh api contributors 分页 Link header，anon=true） |
| 官网/文档 | https://www.gymlibrary.dev/ （文档仓库也在 Farama 侧：Farama-Foundation/gym-docs） |

## 2. 项目是什么，解决什么问题

2016 年之前，RL 论文的实验环境各写各的，算法与环境高度耦合、结果无法互比。Gym（论文 arXiv:1606.01540，Brockman 等，`README.md` 引用条目）提出一个极简的抽象：环境就是一个 Python 类，暴露两个属性（动作空间、观测空间）和三个方法（`step`、`reset`、`render`）。任何算法只要面向这个接口编程，就能跑在任何实现了该接口的环境上；反之任何新环境只要实现该接口，就能立刻被所有现有算法库使用。README 明确自称 "providing a standard API to communicate between learning algorithms and environments"，并指出 "Since its release, Gym's API has become the field standard"（`src/README.md`）。

Gym 同时内置了一批按此接口实现的环境作为参考实现：经典控制（CartPole、Pendulum 等，纯物理公式 + pygame 渲染）、Box2D（LunarLander、BipedalWalker、CarRacing）、MuJoCo 连续控制（Ant、Humanoid、HalfCheetah 等，XML 模型资产随包发布）、ToyText（FrozenLake、Taxi 等网格世界），并通过 entry-point 插件机制把 Atari（ale-py）等第三方环境接入统一注册表（`src/setup.py` extras、`src/gym/envs/registration.py` 的 `load_env_plugins`）。

仓库的第二重价值是它记录了这套 API 的整个演化史：从 step 返回四元组 `(obs, reward, done, info)` 到五元组 `(obs, reward, terminated, truncated, info)`，从独立的 `env.seed()` 方法到 `reset(seed=...)`，从 `render(mode=...)` 参数到构造时传入 `render_mode`。2021 年起 OpenAI 逐步淡出，维护工作转移给社区（`CONTRIBUTING.md` 记录 2021-07-27 起不再接受新环境/新功能 PR），2021 年 Farama 基金会成立并接手，最终 fork 出 Gymnasium；2022-10-25 README 加入 "Important Notice" 宣告 Gym 停止更新、请迁移 Gymnasium（提交 6a04d49，gh api 核实）。

对 RL 而言，Gym 的历史地位类似 Web 领域的 HTTP/1.0：实现本身未必是最优的，但它定义的词汇表（`Env`、`Space`、`gym.make("CartPole-v1")`）成了整个行业沟通的语言，后续的 PettingZoo（多智能体）、Gymnasium、乃至 DeepMind 的 dm_env 都是在与它对话。

## 3. 技术栈与依赖

- 语言：纯 Python，支持 3.6-3.10（`setup.py` classifiers；CI 矩阵 py.Dockerfile 构建 3.6-3.10）
- 核心运行时依赖（`setup.py` install_requires）：
  - `numpy>=1.18` —— 空间采样、观测数组、`np.random.Generator`/`SeedSequence`/PCG64 随机体系
  - `cloudpickle` —— 环境对象序列化（`gym/utils/ezpickle.py`）
  - `gym_notices` —— 运行时按版本号打印弃用/迁移公告的轻量机制（`gym/__init__.py`）
- 可选 extras 按环境族分组（`setup.py`）：`classic_control/toy_text → pygame==2.1.0`；`box2d → box2d-py + swig`；`mujoco → mujoco==2.2`（新）与 `mujoco_py`（旧，仅维护）；`atari → ale-py~=0.8.0`（Atari 已完全外包给 ALE 插件）
- 质量工具：black + isort + flake8（`.pre-commit-config.yaml`）、pyright 静态类型检查（`CONTRIBUTING.md`）、pytest==7.0.1（`test_requirements.txt`）
- 无框架、无 C 扩展本体；渲染统一走 pygame（`gym/__init__.py` 强制设置 `PYGAME_HIDE_SUPPORT_PROMPT` 与 SDL 音频驱动）

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
| --- | --- |
| `gym/core.py` | Env 基类、Wrapper/ActionWrapper/RewardWrapper/ObservationWrapper 基类——整套 API 的"宪法" |
| `gym/spaces/` | 10 种空间类型 + flatten 工具（`space.py` 基类，`box/discrete/multi_discrete/multi_binary/tuple/dict/text/graph/sequence.py`，`utils.py`） |
| `gym/envs/` | 内置环境：`classic_control/`(5 环境) `box2d/`(3) `mujoco/`(13 类) `toy_text/`(4) + `registration.py` 注册表 |
| `gym/wrappers/` | 21 个公开包装器 + `monitoring/video_recorder.py`，含 `compatibility.py` 旧 API 兼容层 |
| `gym/vector/` | 向量化并行环境：`vector_env.py` 基类 + `sync_vector_env.py` / `async_vector_env.py` + `utils/`（batch_space 等） |
| `gym/utils/` | `seeding.py` 随机数、`step_api_compatibility.py` 四/五元组互转、`passive_env_checker.py`/`env_checker.py` 环境合规检查、`play.py` 人工游玩、`save_video.py` |
| `tests/` | 73 个测试文件，按 envs/spaces/utils/vector/wrappers 分目录 |
| `bin/docker_entrypoint`、`py.Dockerfile` | Docker 化 CI 测试（含 MuJoCo 210 二进制安装） |
| `.github/workflows/` | `build.yml`（docker pytest 矩阵）+ `pre-commit.yml` |

组织方式评述：单包单体库，物理边界即 API 边界——core 定义接口、spaces 定义数据类型、envs 是接口的参考实现、wrappers 是接口的组合子、vector 是批量化扩展。113 个 py 文件共约 21643 行，其中 envs 占 12615 行（58%），说明它更像"规范 + 参考实现集"而非算法库（`find gym -name '*.py'` 统计）。

## 5. 核心架构与关键实现

### 5.1 Env 基类：RL 接口的事实标准（gym/core.py）

`Env(Generic[ObsType, ActType])` 用泛型把观测/动作类型参数化，只定义契约不做实现：`step()` 抛 NotImplementedError，`reset()` 仅在传入 seed 时初始化 RNG。五元组签名与 terminated/truncated 的语义区分是本版（0.26）API 的核心（`src/gym/core.py:86-115`）：

```python
def step(self, action: ActType) -> Tuple[ObsType, float, bool, bool, dict]:
    """...
    terminated (bool): whether a `terminal state` (as defined under the MDP
        of the task) is reached.
    truncated (bool): whether a truncation condition outside the scope of
        the MDP is satisfied. Typically a timelimit...
    """
    raise NotImplementedError
```

terminated 表示 MDP 终态（物理上游戏结束），truncated 表示 MDP 外的截断（如时间步上限）——两者分开是 0.25/0.26 最大的争议性变更，解决了旧版 `done` 把"赢了"和"超时"混为一谈、导致 bootstrapping 计算错误的老问题。配套属性契约：`action_space`、`observation_space`、`reward_range`、`spec`、`metadata`、`np_random`；外加 `unwrapped`、with 语句支持（`__enter__/__exit__` 调 `close()`）。

### 5.2 Wrapper 体系：装饰器组合子（gym/core.py:213-469）

`Wrapper(Env)` 通过 `__getattr__` 把未覆写的属性全部转发给内层环境（私有属性 `_` 开头禁止透传，`_np_random` 显式抛错引导用户用 `.unwrapped`），并提供 ActionWrapper/RewardWrapper/ObservationWrapper 三个子类把"改动作/改奖励/改观测"收敛为单一钩子方法（`action()/reward()/observation()`）。这一设计让 TimeLimit、FrameStack、RescaleAction 等 21 个包装器（`gym/wrappers/__init__.py` 全部 22 项导出）可以任意叠放，成为后来所有 RL 库包装器体系的蓝本。

### 5.3 空间类型体系（gym/spaces/）

`Space(Generic[T_cov])` 基类（`src/gym/spaces/space.py`）提供 `sample()` 随机采样、`contains()` 判定、seed/shape/dtype 三要素；其上共 10 种具体空间：Box（连续向量）、Discrete、MultiDiscrete、MultiBinary（离散变体）、Tuple/Dict（容器组合）、Text（字符串）、Graph（图结构，含 GraphInstance）、Sequence（变长嵌套）。`spaces/utils.py` 提供 flatdim/flatten/unflatten/flatten_space，把任意嵌套空间展平成学习代码需要的向量——采样、批量化（`gym/vector/utils/spaces.py` 的 batch_space）、展平三件事都定义在空间上而非环境上，是 Gym 类型论的核心。

### 5.4 注册表与 make() 包装栈（gym/envs/registration.py）

环境 ID 格式 `[namespace/](name)-v(version)` 由正则 `ENV_ID_RE` 固化（`src/gym/envs/registration.py:47`）；`register()` 把 `EnvSpec` 数据类（entry_point、max_episode_steps、order_enforce、autoreset、apply_api_compatibility 等字段）存入全局 `registry` 字典；`load_env_plugins()` 通过 importlib entry points 加载第三方环境包（ale-py 即由此接入）。`make()` 创建实例时自动叠加包装器：OrderEnforcing（强制先 reset 后 step）→ PassiveEnvChecker（运行时静默检查 env 是否合规）→ TimeLimit（超步数置 truncated）→ 可选 AutoReset/HumanRendering/RenderCollection；`make()` 还内置了全部内置环境 ID 的 `Literal` 重载声明，让 pyright/mypy 能静态检查环境名拼写（`src/gym/envs/registration.py:314-370`）。环境版本语义也在此强制：同名环境存在更新版本时警告升级、未带版本号时自动取最新版本——`_v0/_v1` 后缀制度保证旧版本结果可复现。

### 5.5 API 迁移的工程化处理（兼容层）

四元组→五元组的迁移不是靠破坏性一刀切，而是三层机制并存：底层工具函数 `convert_to_terminated_truncated_step_api()` / `convert_to_done_step_api()` 用 `info["TimeLimit.truncated"]` 键值从旧 done 中还原 truncated 语义（`src/gym/utils/step_api_compatibility.py:24-73`）；包装器 `StepAPICompatibility` / `EnvCompatibility`（`src/gym/wrappers/compatibility.py`，含 `LegacyEnv` Protocol 结构化鸭子类型判定旧式环境）把旧环境升级为新 API；`EnvSpec.apply_api_compatibility` 让注册表按环境粒度选择。时间线（gh api 提交记录核实）：0.25（2022-08）引入双布尔并弃用 `env.seed()`（提交 3a8daaf "Removing return_info argument to env.reset() and deprecated env.seed()"），0.25.2 起仅支持新 step API（提交 54b406b），0.26（2022-09-06）完成 reset 返回 `(obs, info)`。

## 6. 构建与运行

```bash
# 安装（README.md）
pip install gym                 # 核心库
pip install gym[classic_control]  # 单环境族依赖
pip install gym[all]            # 全部环境依赖

# 最小使用（README.md 示例，0.26 API）
import gym
env = gym.make("CartPole-v1")
observation, info = env.reset(seed=42)
observation, reward, terminated, truncated, info = env.step(env.action_space.sample())

# 测试（CI 同款路径：.github/workflows/build.yml + py.Dockerfile）
docker build -f py.Dockerfile --build-arg PYTHON_VERSION=3.10 -t gym-docker .
docker run gym-docker pytest

# 本地开发
pip install -e .[testing] && pytest
pre-commit run --all-files      # black/isort/flake8
pyright                          # 类型检查（CONTRIBUTING.md）
```

官方支持 Linux/macOS + Python 3.7-3.10，Windows 仅社区 PR 不官方支持（`README.md`）。

## 7. 测试与工程质量

- 测试规模：73 个测试文件（`find tests -name '*.py'`），按 envs/spaces/utils/vector/wrappers 分包，另有个 `tests/testing_env.py` 通用测试夹具环境
- 风格：pytest 单元测试为主，直接构造微型 Env/Wrapper 实例验证契约（如 `tests/test_core.py` 的 UnittestEnv），不依赖重训练流程
- CI：`build.yml` 在 Docker 内跑 pytest，Python 3.6-3.10 五版本矩阵（MuJoCo 二进制在镜像内安装）；`pre-commit.yml` 跑 black/isort/flake8
- 类型工程化程度在同代 Python 库中属最高档：全包 `py.typed`、`Env[ObsType, ActType]`/`Space[T]` 泛型、`make()` 的 Literal 重载、pyright 检查
- 特色质量机制：`gym/utils/passive_env_checker.py` 在 `make()` 时静默检查环境合规性（观测越界、reset 签名等），相当于内置的 API lint

## 8. 社区与维护状态

- 已归档：gh api `archived: true`。维护团队自 2021 年起转入 Farama 基金会（README Notice 原文 "The team that has been maintaining Gym since 2021 has moved all future development to Gymnasium"）
- 时间线（gh api 提交/发布记录）：2021-07-27 收紧贡献政策（`CONTRIBUTING.md`，不再接受新环境/新功能）→ 2021-10 Farama 宣布成立、Gymnasium fork 诞生 → 2022-08/09 五元组 API 落地（0.25/0.26）→ 2022-10-04 最终版 0.26.2 → 2022-10-25 README 迁移公告（6a04d49）→ 2023-01-30 最后一次 README 更新 → 2026-03-26 仅一次 CI 引用固化提交 bc21295（#3328，无代码变更）
- 贡献者约 385 人（gh api contributors 分页）；issues 仍保持 128 个开放（归档仓库不再处理）
- PyPI 上的 gym 包仍可安装（0.26.2 终版），但不会再有更新；生态完全由 Gymnasium（API 基本兼容，`import gymnasium as gym`）延续

## 9. 亮点与风险

亮点：
1. 定义了 RL 的标准词汇表——`Env`/`Space`/`gym.make` 四件套至今仍是全行业通用语言（`gym/core.py`），论文复现、算法库（Stable-Baselines3、RLlib、CleanRL、Tianshou）、环境库（ale-py、PettingZoo）全部围绕它构建
2. Wrapper 装饰器组合子设计极优雅：`__getattr__` 透传 + 三类钩子，21 个包装器任意叠放且行为可预测（`gym/core.py:213`、`gym/wrappers/`）
3. 空间类型体系完备：10 种空间覆盖标量/向量/图/文本/嵌套，采样、判定、展平、批量化四操作统一（`gym/spaces/`）
4. 兼容迁移的教科书示范：工具函数 + 兼容包装器 + 注册表开关三层机制，让五元组 API 平滑落地而不撕裂生态（`gym/utils/step_api_compatibility.py`）
5. 类型化程度高（py.typed + Literal 重载 + pyright），并内置运行时环境合规检查器

风险/局限：
1. 已归档停止维护——不再有任何 bug 修复与新 Python 版本适配，新项目必须用 Gymnasium（`README.md` Important Notice）
2. 版本间 API 断裂严重（0.21 的 done/seed → 0.25/0.26 的 terminated/truncated/reset(seed)），网上大量教程基于旧 API，混用易错
3. 内置环境质量参差：Atari 完全外包 ale-py，旧 MuJoCo 路线（mujoco-py）仅维护不更新，`CONTRIBUTING.md` 已宣布不接受新环境
4. 渲染强绑 pygame 2.1.0 精确版本、`gym_notices` 运行时打印公告等设计在容器化/无头场景偏脆弱

## 10. 总结评价

Gym 是强化学习工程化的奠基之作：它没有发明任何算法，却用一个不到 200 行的基类定义了整个领域的接口方言，其 `Env`/`Space`/Wrapper 三层抽象被 Gymnasium、PettingZoo、dm_env 及几乎所有 RL 库继承，五元组 step API 的 terminated/truncated 之争也在仓库的 issue、提交与兼容层代码中留下了完整的决策标本。如今仓库本体已归档封存（终版 0.26.2），但它的 API 以 Gymnasium 的形式继续统治着这个领域。适合两类人：一是 RL 学习者拿来理解"环境接口"到底该长什么样（读 `gym/core.py` + `gym/spaces/` 足矣）；二是库设计者把它的 wrapper 体系、注册表插件机制、API 平滑迁移三层方案当作架构参考。生产与科研使用请直接选 Gymnasium，本仓库的正确打开方式是阅读与考古。
