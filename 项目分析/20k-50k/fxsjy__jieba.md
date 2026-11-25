# fxsjy/jieba 项目分析

> 星数 39500（快照 2026-10-24）｜主语言 Python｜档位 20k-50k｜分析日期 2026-11-21
> 注：gh api 实测 stargazers 35,247（2026-11-21），与清单快照存在出入，以清单快照为准记录。

## 1. 项目速览

一句话定位：jieba（结巴）是 Python 生态的中文分词事实标准——纯 Python、零硬依赖、基于前缀词典 + DAG 动态规划 + HMM 未登录词识别，十年间成为几乎所有中文 NLP 项目的默认分词起点。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | fxsjy/jieba（"结巴"中文分词） | README.md |
| 星数 | 39500（快照）；gh api 35,247 | gh api repos/fxsjy/jieba |
| 主语言 | Python（py2/py3 双兼容） | gh api / setup.py |
| 许可证 | MIT | LICENSE、setup.py |
| 当前版本 | 0.42.1 | jieba/__init__.py `__version__` |
| 创建时间 | 2012-09-29 | gh api created_at |
| master 最后提交 | 2020-02-15（"Update README.md update paddle link"） | gh api commits |
| 最新 tag | v0.42.1（提交 2020-01-20） | gh api tags |
| 贡献者 | 37 人（contributors 分页数） | gh api contributors Link header |
| 维护状态 | **已停更多年**（master 自 2020-02 起零提交，issue 积压 707，未归档） | gh api |

## 2. 项目是什么，解决什么问题

中文没有天然的空格分词边界，任何中文搜索、文本挖掘、NLP 管线的第一步都是分词（word segmentation）。jieba 解决的问题是：给 Python 开发者一个"开箱即用、无需训练、无需 GPU、依赖为零"的中文分词组件，`pip install jieba` 后三行代码即可切分文本（README.md「安装说明」「算法」节）。

它的算法路线是经典的词典 + 统计混合（README.md「算法」节，实现见 `jieba/__init__.py`）：
1. 基于前缀词典高效扫描，生成句子中汉字所有可能成词构成的有向无环图（DAG）；
2. 动态规划查找最大概率路径，找出基于词频的最大切分组合；
3. 对词典查不到的连续片段（未登录词/新词），用基于汉字成词能力的 HMM（BMES 四状态）+ Viterbi 算法识别。

三种分词模式对应三类场景（README.md「特点」）：精确模式（文本分析默认）、全模式（扫描所有可成词组合，快但不论歧义）、搜索引擎模式（对长词再切细粒度，提高召回）。v0.40（2019-12）又追加了第四种 paddle 模式，接入百度 LAC 的双向 GRU 序列标注模型（README.md 特点节第 4 条，实现在 `jieba/lac_small/`）。

除分词外还内置词性标注（`jieba/posseg/`）、关键词抽取（`jieba/analyse/`，TF-IDF 与 TextRank）、并行分词、繁体支持、自定义词典，并附带 whoosh 搜索引擎的 ChineseAnalyzer（`jieba/analyse/analyzer.py`）。这套"分词 + 词性 + 关键词"全家桶使它成为中文文本处理的默认工具箱，而不仅是分词器。

词典规模（实数统计自本仓 tarball）：主词典 `jieba/dict.txt` 349,046 行（约 5MB），`extra_dict/dict.txt.big` 584,429 行、`dict.txt.small` 109,749 行，idf 词典 176,238 行。词条格式为 `词语 词频 词性`（如 `AT&T 3 nz`）。

## 3. 技术栈与依赖

- 语言：纯 Python，2/3 双兼容（`jieba/_compat.py` 提供 xrange/iteritems/unicode 等桥接）。
- 运行时依赖：**零第三方硬依赖**——只用标准库 marshal（词典缓存）、re、threading、multiprocessing、hashlib、math、pkg_resources（`jieba/__init__.py` 头部 import，`jieba/_compat.py`）。
- 可选依赖：
  - paddlepaddle-tiny==1.6.1：paddle 模式（`jieba/_compat.py` 的 `enable_paddle()`，README 要求固定该版本）；
  - whoosh：仅 `jieba/analyse/analyzer.py`（ChineseAnalyzer）与对应测试使用。
- 打包：`setup.py` 用 `distutils.core.setup`，classifiers 只声明到 Python 3.4（setup.py:60-66）——打包元数据明显停留在 2020 年前的生态。
- 无 pyproject.toml、无 requirements、无 CI 配置（tarball 全量 grep yml/yaml/github 无结果）。

## 4. 目录结构与代码组织

| 目录/文件 | 职责 |
|---|---|
| `jieba/__init__.py` | 核心：Tokenizer 类、前缀词典构建、DAG/DP、三种 cut 模式、并行分词、模块级单例 dt（619 行） |
| `jieba/_compat.py` | py2/py3 兼容层、paddle 探测与 enable_paddle()（89 行） |
| `jieba/__main__.py` | 命令行入口 `python -m jieba`（分词/词性/自定义词典选项） |
| `jieba/dict.txt` | 主词典 349,046 条（词 语/词频/词性） |
| `jieba/finalseg/` | 未登录词识别：BMES HMM + Viterbi；prob_*.py 为巨型概率表（prob_emit.py 35,226 行） |
| `jieba/posseg/` | 词性标注：词典词性表 + 二级 HMM（char_state_tab.py 61,087 行） |
| `jieba/analyse/` | 关键词抽取：tfidf.py、textrank.py、analyzer.py（whoosh）、idf.txt |
| `jieba/lac_small/` | paddle 模式：百度 LAC 蒸馏模型（nets/predict/reader + model_baseline 权重） |
| `extra_dict/` | 备选词典：dict.txt.big/small、idf.txt.big、stop_words.txt |
| `test/` | unittest + 演示脚本（jieba_test.py、test_pos.py、test_userdict.py 等 20+ 文件） |
| `setup.py` / `Changelog` / `README.md` / `LICENSE` | 打包、版本史（2012→2020 全记录）、文档 |

组织方式评述：单包（single-package）布局，一个 `jieba/` 包内按功能分子模块，无构建系统、无 CI、无 lint 配置。代码量很小：手写 Python 逻辑合计不到 2,000 行，其余 19 万行几乎全是嵌入源码的概率表（prob_emit.py 等，用 .py 文件而非 .p pickle 以便 pyinstaller 类打包，finalseg/__init__.py:29-34 的 Jython 分支解释了 .p 与 .py 双份存在）。这种"逻辑极薄、数据内嵌"的形态正是它能被随处 pip 安装即用的关键。

## 5. 核心架构与关键实现

### 5.1 前缀词典与 DAG 最大概率路径（jieba/__init__.py）

v0.34 起（Changelog 2014-10-20）词典由 Trie 树改为"前缀集合"：`gen_pfdict()` 把每个词连同其全部前缀片段塞进一个 dict，完整词记录真实词频，前缀片段记 0（jieba/__init__.py:72-91）。查词典退化为 O(1) dict 查找，官方称内存占用减少 2/3。

`get_DAG()` 用这个 dict 一次滑动扫描生成句子的有向无环图；`calc()` 从句尾向句头做动态规划，route[idx] = 以 idx 开头的最优切分得分（jieba/__init__.py:172-196）：

```python
def calc(self, sentence, DAG, route):
    N = len(sentence)
    route[N] = (0, 0)
    logtotal = log(self.total)
    for idx in xrange(N - 1, -1, -1):
        route[idx] = max((log(self.FREQ.get(sentence[idx:x + 1]) or 1) -
                          logtotal + route[x + 1][0], x) for x in DAG[idx])
```

得分即 log(词频/总词频) 的路径和，按词频最大组合切分——这是精确模式的主干。精确模式下 DP 得到的单字且不成词的连续缓冲区，交给 finalseg 做 HMM 新词识别（`__cut_DAG`，jieba/__init__.py:249-287）。

### 5.2 三种模式的实现差异（jieba/__init__.py）

- 精确模式 `cut(cut_all=False, HMM=True)`：DAG + DP 最优路径，未登录片段走 HMM（`__cut_DAG`）；HMM=False 时单字直接连成散字输出（`__cut_DAG_NO_HMM`）。
- 全模式 `__cut_all`（jieba/__init__.py:198-224）：不做 DP，直接枚举 DAG 中所有从每个起点出发的成词区间全部 yield，因此会输出大量重叠词（如"清华/清华大学/华大"），速度极快但不消歧。
- 搜索引擎模式 `cut_for_search`（jieba/__init__.py:338-354）：先跑精确模式，再把长度 >2 的词再切出所有词典中存在的 2-gram、>3 再切 3-gram，最后附上原词——一句话同时产出细粒度与整词，兼顾倒排索引召回：

```python
for w in words:
    if len(w) > 2:
        for i in xrange(len(w) - 1):
            gram2 = w[i:i + 2]
            if self.FREQ.get(gram2):
                yield gram2
    if len(w) > 3:
        for i in xrange(len(w) - 2):
            gram3 = w[i:i + 3]
            if self.FREQ.get(gram3):
                yield gram3
    yield w
```

- paddle 模式（`cut` 中 use_paddle 分支，jieba/__init__.py:301-311）：绕开上述全部逻辑，直接调用 `jieba.lac_small.predict` 的双向 GRU 模型。

### 5.3 HMM 未登录词识别（jieba/finalseg/__init__.py）

finalseg 把连续汉字序列按 B(词首)/M(词中)/E(词尾)/S(单字) 四状态 HMM 做 Viterbi 解码，概率表（start/trans/emit）以 Python dict 字面量内嵌在 prob_*.py（finalseg/__init__.py:32-34）。核心 40 行：

```python
def viterbi(obs, states, start_p, trans_p, emit_p):
    V = [{}]; path = {}
    for y in states:
        V[0][y] = start_p[y] + emit_p[y].get(obs[0], MIN_FLOAT)
        path[y] = [y]
    for t in xrange(1, len(obs)):
        V.append({}); newpath = {}
        for y in states:
            em_p = emit_p[y].get(obs[t], MIN_FLOAT)
            (prob, state) = max(
                [(V[t - 1][y0] + trans_p[y0].get(y, MIN_FLOAT) + em_p, y0) for y0 in PrevStatus[y]])
            ...
```

`PrevStatus` 硬编码了合法转移（B 后只能 E/S 等），未登录词片段识别后从 `__cut_DAG` 的缓冲区流式吐出。README 的示例"网易杭研大厦"中"杭研"不在词典仍能被切出即来自此机制。另外 `add_force_split`（finalseg/__init__.py:81-83）支持用户把词频设 0 强制拆词（配合 `del_word`/`suggest_freq`）。

### 5.4 自定义词典与词性标注（jieba/__init__.py、jieba/posseg/__init__.py）

自定义词典机制：`load_userdict()` 按 `re_userdict` 正则解析 `词 [词频] [词性]` 行（jieba/__init__.py:36,380-416），词频省略时用 `suggest_freq()` 自动计算"恰好能保证该词被切出"的频率——通过把该词当前各切分片段的概率乘积反解出来加 1（jieba/__init__.py:445-474）。`add_word` 同步维护前缀片段并刷新 total；词性写入 `user_word_tag_tab` 供 posseg 使用。初始化有 marshal 缓存（默认字典缓存为 `jieba.cache`，自定义词典按路径 md5 生成独立缓存，jieba/__init__.py:115-156），并用 threading.RLock + 全局 DICT_WRITING 字典防多线程重复建缓存。

词性标注 posseg 是"词典词性 + 二级 HMM"的包装器：`POSTokenizer` 复用主 Tokenizer 的 DAG/DP 结果，词典命中词的词性直接查 `word_tag_tab`（dict.txt 第三列）；未登录连续段则走带 `char_state_tab`（状态为 (词性, BMES) 组合）的第二个 Viterbi（`jieba/posseg/viterbi.py`），输出 `pair(word, flag)`（jieba/posseg/__init__.py:117-133,175-213）。

### 5.5 关键词抽取与并行（jieba/analyse/、jieba/__init__.py）

- TF-IDF（`analyse/tfidf.py`）：词频 × 预置 idf 词典（缺词用中位数 idf），支持 allowPOS 过滤与自定义 idf/stopwords。
- TextRank（`analyse/textrank.py`）：滑动窗口 span=5 共现建无向带权图，固定 10 轮迭代、阻尼 d=0.85（textrank.py:13-54），默认只保留 ns/n/vn/v 四类词性。
- 并行分词（`enable_parallel`，jieba/__init__.py:590-610）：按行 splitlines 后用 multiprocessing.Pool 分发，**仅支持 posix，Windows 直接 NotImplementedError**；只对全局默认分词器 dt 生效。
- whoosh 集成（`analyse/analyzer.py`）：ChineseTokenizer 包装 `jieba.tokenize(mode="search")` 输出带位置 Token。

## 6. 构建与运行

无构建产物，直接源码可用（README.md「安装说明」）：

```bash
pip install jieba                 # 或 python setup.py install
python -m jieba --pos input.txt   # CLI 分词（jieba/__main__.py）
python -m jieba -D custom.dict -u user.dict input.txt
```

API 最小示例（README.md 代码示例节）：

```python
import jieba
seg_list = jieba.cut("我来到北京清华大学", cut_all=False)  # 精确模式
seg_list = jieba.cut_for_search("小明硕士毕业于中国科学院计算所")  # 搜索引擎模式
```

测试（test/ 目录，unittest）：`python test/jieba_test.py` 等；并行测试在 `test/parallel/`；whoosh/paddle 测试需自行安装可选依赖。

## 7. 测试与工程质量

- 测试规模：test/ 目录 20+ 文件，核心是 unittest 风格的对照测试——把数十条典型句（含歧义句"我不喜欢日本和服"、专名"湖北省石首市"、新词"草泥马"等，test/jieba_test.py:14-60）的切分结果与硬编码期望逐一断言；覆盖精确/全模式/搜索引擎/HMM 开关/词性/用户词典/tokenize 位置等主要 API 面（test/test_cutall.py、test_cut_for_search.py、test_no_hmm.py、test_pos.py、test_userdict.py、test_tokenize.py 等）。
- 无 pytest/tox/CI：仓库无 .github 目录、无任何 CI 配置（tarball grep 证实），测试靠人工跑，质量完全依赖核心贡献者的个人把关。
- 代码质量观察：整体紧凑可读（核心 619 行完成全部主逻辑），但存在明显历史包袱——py2/py3 双轨代码遍布（`jieba/_compat.py`、`xrange`/`iteritems` 调用散布各模块）、Jython 专用分支（finalseg/__init__.py:29-34）、`_compat.py:32` 直接 `os.system("pip install paddlepaddle-tiny")` 在库代码里装包（反模式）、paddle 版本比较 `paddle.__version__ < '1.6.1'` 为字符串比较（_compat.py:39，'1.10' < '1.6.1' 判断错误）。无类型标注、无 lint 配置。

## 8. 社区与维护状态

- **停更核实（gh api，2026-11-21）**：master 最后一次提交为 2020-02-15（"Update README.md update paddle link"，sha 67fa2e36）；最新 tag v0.42.1 于 2020-01-20 提交（Changelog 中 0.42.1 条目年份误写为 2019-1-20，Changelog:1）。仓库未归档，pushed_at 显示 2024-08-21 有过一次非 master 推送（具体分支未确认），但 master 本体已冻结近 7 年。
- Issue 积压 707 个（gh api open_issues），无 issue 模板、无 CONTRIBUTING 文件。
- Release：gh api 实测 9 个 release，节奏止于 2020 年。
- 贡献者结构：37 位贡献者，Changelog 显示核心演化由作者 fxsjy（Sun Junyi）与 @gumblex（2014-2015 的 py2/py3 重构、Tokenizer 类化、性能优化）、@fukuball、@sing1ee 等推动；2019 年 paddle 模式由百度侧贡献者（@JesseyXujin、@xyzhou-puck）合入（Changelog:14）。
- 社区现实：虽停更，PyPI 上 jieba 仍是中文分词下载量最大的包之一，衍生生态（jieba-fast、jiebaT、jieba 的 Rust/C++ 重实现、各类框架内置 fork）活跃（生态状况为公开常识，本仓内无直接证据，未确认）。

## 9. 亮点与风险

亮点：
1. 极致轻量与零依赖：核心逻辑 <2,000 行纯 Python + 标准库，pip 装完即用，嵌入式/School/Spark 环境都能跑（Changelog 0.38 提及 pkg_resources 改造以支持 Spark）。
2. 算法设计务实高效：前缀词典（v0.34 由 Trie 改造，内存降 2/3）+ DAG 一遍扫描 + 尾向首 DP，精确模式在纯 CPU 上有很好的吞吐；HMM 新词识别让词典外的人名/新词也能切出（README 示例"杭研"）。
3. API 设计成为事实标准：`jieba.cut/lcut/cut_for_search/posseg.cut/analyse.extract_tags` 的签名被后续几乎所有中文分词库模仿；支持繁体、并行、自定义词典、强制调频（suggest_freq）等工程细节齐全。
4. 全家桶自足：分词+词性+关键词+whoosh analyzer+paddle 深度模式（0.40）一站配齐。
5. Changelog 完整可追溯（2012→2020 每版有变更与致谢），历史决策有据可查。

风险/局限：
1. **已停更多年**：master 冻结于 2020-02，issue 大量积压无人处理；paddle 模式锁死 paddlepaddle-tiny==1.6.1（该包已随 Paddle 2.x 演进基本不可用，README.md:16），深度模式名存实亡。
2. Python 2→3 遗留代码大量残留：xrange/iteritems 桥接层、GBK 解码兜底、字符串版本号比较 bug（_compat.py:39）等；打包元数据只到 Python 3.4，与新版本 Python 的兼容是"事实上能跑、无官方声明"状态。
3. 词典与模型数据 6 年未更新：349k 词条的主词典不含近年新词/新实体，全靠用户自行 load_userdict 补；HMM 概率表同样是 2020 年前训练结果（数据时效性问题，本仓文件 mtime 2020-02-15 佐证）。
4. 无 CI、无类型标注、测试为硬编码对照而非指标化评测（无准确率/F1 基线入库），准确率只能对照外部评测。
5. 并行模式 Windows 不可用（enable_parallel 直接 NotImplementedError，jieba/__init__.py:600-602），且仅支持全局默认实例。

## 10. 总结评价

jieba 是"好架构 + 好时机 + 长尾效应"的典型：以不到两千行的词典+DP+HMM 混合方案，在 2012-2015 年间把中文分词的门槛降到了 pip install 一行，并凭借稳定 API 成为整个中文 NLP 生态的事实标准——即使项目自 2020 年起停止维护，其星数与使用量仍在惯性增长。今天的选型逻辑很清晰：轻量、离线、可解释、要稳定 API 兼容 → jieba 依然是最省心的默认项；追求准确率上限（专名、领域新词、BERT 级语义）则应转向 HanLP 2.x、pkuseg、LAC 或各类预训练分词/ segmentation 模型，它们才是 jieba 停更后生态的实际接棒者。对学习者而言，`jieba/__init__.py`（619 行）+ `jieba/finalseg/__init__.py`（100 行）是理解"词典分词 + HMM 未登录词识别"教科书级的精读材料；对工程团队而言，把它当稳定遗留组件使用而非依赖其演进。
