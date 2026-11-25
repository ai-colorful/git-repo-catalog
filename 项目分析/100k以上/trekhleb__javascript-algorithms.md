# trekhleb/javascript-algorithms 项目分析

> 星数 196690（快照 2026-10-24）｜主语言 JavaScript｜档位 100k以上｜分析日期 2026-10-30

## 1. 项目速览

一句话定位：以 JavaScript 实现的主流算法与数据结构教学库，每个算法/结构配独立 README 讲解与延伸阅读链接，是 GitHub 上最知名的计算机科学基础学习资源之一。

| 项目 | 值 | 出处 |
| --- | --- | --- |
| 名称 | trekhleb/javascript-algorithms | 仓库元数据 |
| 星数 | 196690 | 任务快照 2026-10-24 |
| Fork 数 | 30997 | gh api repos/trekhleb/javascript-algorithms |
| 主语言 | JavaScript（纯 ES Module 源码） | `src/**/*.js` |
| 许可证 | MIT（Copyright (c) 2018 Oleksii Trekhleb） | `LICENSE` |
| 创建时间 | 2018-03-24 | gh api 元数据 |
| 最近 push | 2026-07-26 | gh api 元数据（tarball 文件 mtime 同为 2026-07-26，互证） |
| 默认分支 | master | gh api 元数据 |

## 2. 项目是什么，解决什么问题

这是一个"可运行的算法教科书"。作者 Oleksii Trekhleb（乌克兰工程师）在 2018 年创建，目标是把经典算法与数据结构用 JavaScript 重写一遍，每个实现都做到：代码可执行、有完整单元测试、配独立 README 解释原理并链接 YouTube 视频与维基条目（`README.md`："Each algorithm and data structure has its own separate README with related explanations and links for further reading"）。

与典型算法题解仓库不同，它按"知识点"而非"题号"组织：`src/algorithms`（算法，14 个分类）与 `src/data-structures`（数据结构，14 个分类）两大树状体系，顶层 README 用 `B`（Beginner）/`A`（Advanced）标注难度并链接到各子目录（`README.md`）。覆盖面极广：从链表、堆、Trie，到图论全套（BFS/DFS/Dijkstra/Bellman-Ford/Floyd-Warshall/Prim/Kruskal/欧拉路径/哈密顿回路/TSP/拓扑排序等 15 个子目录）、排序 10 种、数学类 21 个子目录（傅里叶变换、矩阵、筛法、比特操作等），甚至包含 ML 入门（k-means、kNN，`src/algorithms/ml/`）与图像处理（seam-carving 内容感知缩放，`src/algorithms/image-processing/`）。

它同时是一份工程范本：所有实现零生产依赖（`package.json` 无 `dependencies`，仅 devDependencies），通过统一的 `Comparator` 比较器抽象实现泛型比较；测试覆盖率门槛设在 statements/functions/lines 100%、branches 95%（`jest.config.js`）。这意味着每个算法都能被逐行学习，又不像玩具代码那样不可靠。

仓库不发布为 npm 库（`package.json` 声明 `main: index.js` 但顶层并无 index.js，version 停在 0.0.4），定位就是"读与跑"的学习仓库，而非生产组件库。顶层 README 已翻译成 18 种语言（`README.zh-CN.md`、`README.ja-JP.md` 等），是其全球传播度的直接证据。

## 3. 技术栈与依赖

- 语言：JavaScript（ES Module），要求 Node >= 22、npm >= 10（`package.json` engines 字段）
- 运行时依赖：**0 个**（`package.json` 只有 devDependencies）
- 测试：Jest ^30.2.0（`jest.config.js`），测试环境由 Babel 转译（`@babel/cli`、`@babel/preset-env`，`.babelrc`）
- 代码规范：ESLint ^8.57.1 + eslint-config-airbnb + jest/import/jsx-a11y 插件（`package.json`、`.eslintrc`）
- 工程辅助：husky ^9.1.7（git hooks，`package.json` prepare 脚本）、pngjs（个别测试用例处理图片，`package.json`）
- CI/质量：GitHub Actions（`.github/workflows/CI.yml`）+ Codecov（jest coverageThreshold 硬门槛）

## 4. 目录结构与代码组织

顶层目录（来源：tarball 解压根）：

| 目录/文件 | 职责 |
| --- | --- |
| `src/algorithms/` | 算法实现，14 个分类：graph(34 个 js)、math(96)、sets(38)、sorting(25)、string(16)、search(8)、cryptography(10)、uncategorized(46，面试经典题) 等 |
| `src/data-structures/` | 数据结构实现，14 个分类：linked-list、heap、graph、tree（BST/AVL/红黑树/线段树/树状数组）、trie、hash-table、bloom-filter、lru-cache、disjoint-set、priority-queue、deque、queue、stack |
| `src/utils/` | 共享工具，目前仅 `comparator/Comparator.js`（全库泛型比较的基石） |
| `src/playground/` | 作者自用 playground（练手数据类型与函数），独立测试 |
| `assets/` | 文档配图 |
| `README.*.md` | 顶层说明的 18 种语言译本；`CONTRIBUTING.md`、`CODE_OF_CONDUCT.md`、`BACKERS.md`、`LICENSE` |

组织方式评述：单包 monolith，无构建产物、无 monorepo/多包结构。核心约定是"每个算法一个目录"，目录内固定三件套：实现 `.js` + `__test__/*.test.js` + 多语言 `README.md`（共 111 个 README.md，见统计），部分还带 `images/` 讲解图。横向复用靠两个注入点：`src/utils/comparator/Comparator.js` 提供比较能力，`src/algorithms/sorting/Sort.js` 提供排序基类（含 visitingCallback 钩子，用于教学可视化/统计访问）。

## 5. 核心架构与关键实现

**（1）全库泛型比较基石 Comparator**（`src/utils/comparator/Comparator.js`）
构造时接受自定义比较函数，默认比较数字/字符串；提供 equal/lessThan/greaterThan 及 reverse()。所有数据结构与排序算法都通过它实现"可比较任意对象"，例如自定义对象数组排序只需传入 compareFunction。这是全库唯一的横向抽象，设计极克制。

**（2）排序算法的模板方法 Sort + 共享测试器 SortTester**
`src/algorithms/sorting/Sort.js` 定义基类：构造时初始化 callbacks（compareCallback、visitingCallback，后者默认空函数）并内建 Comparator，子类只需实现 `sort()`。以 `src/algorithms/sorting/quick-sort/QuickSort.js` 为例（节选）：

```js
sort(originalArray) {
  const array = [...originalArray];      // 克隆防副作用
  if (array.length <= 1) return array;
  const pivotElement = array.shift();
  const centerArray = [pivotElement];
  while (array.length) {
    const currentElement = array.shift();
    this.callbacks.visitingCallback(currentElement); // 访问钩子
    if (this.comparator.equal(currentElement, pivotElement)) centerArray.push(currentElement);
    else if (this.comparator.lessThan(currentElement, pivotElement)) leftArray.push(currentElement);
    else rightArray.push(currentElement);
  }
  return this.sort(leftArray).concat(centerArray, this.sort(rightArray));
}
```

10 种排序共享 `src/algorithms/sorting/SortTester.js`（72 行）做一致性断言：同一组用例（随机数组、含重复、已排序、自定义对象比较）逐一喂给每种排序实现，保证行为统一。

**（3）数据结构继承示例：Heap 家族**（`src/data-structures/heap/`）
抽象基类 `Heap.js` 实现 siftUp/siftDown/poll/peek 等全部逻辑，唯一抽象点是顺序判断；`MinHeap.js` 仅 17 行，只覆写一个方法即得完整实现（`src/data-structures/heap/MinHeap.js`）：

```js
export default class MinHeap extends Heap {
  pairIsInCorrectOrder(firstElement, secondElement) {
    return this.compare.lessThanOrEqual(firstElement, secondElement);
  }
}
```

`MaxHeap.js` 反向同理；另有 MinHeapAdhoc/MaxHeapAdhoc 提供临时变体。这种"骨架在上层、差异点最小化"的写法贯穿全库（tree 分类下 BST/AVL/红黑树同理分层）。

**（4）测试与实现强耦合的教学设计**
测试不是黑盒验证，而是教学材料。例如 `src/algorithms/graph/dijkstra/__test__/dijkstra.test.js` 手工构建 8 顶点 12 边的带权图（GraphVertex/GraphEdge 来自 `src/data-structures/graph/`），断言最短路径结果，读者从测试即可学会如何组装 API。ML 分类（`src/algorithms/ml/k-means/kMeans.js`、`src/algorithms/ml/knn/knn.js`）与 image-processing 的 seam-carving 亦用同样模式（测试目录内均有对应 `__test__`）。

## 6. 构建与运行

来自 `package.json` scripts 与 `.github/workflows/CI.yml`（本项目为教学库，无"运行"概念，核心是测试与 lint）：

```bash
npm install          # 安装 devDependencies（Node >= 22）
npm run lint         # eslint ./src/**（airbnb 规范）
npm test             # jest 全量测试
npm run coverage     # jest --coverage
npm run ci           # lint + coverage（CI 同款命令）
```

CI 流程（`.github/workflows/CI.yml`）：push/PR 到 master 触发，Node 22.x 矩阵，依次 `npm run lint`、`npm run coverage`，并经 codecov-action@v4 上报覆盖率。

## 7. 测试与工程质量

- 测试规模：全库 360 个 .js 文件中 177 个为测试文件（约 49%）；实现代码约 12323 行，测试代码约 11639 行，接近 1:1（本次对 `$SRC/src` 的 find+wc 统计）
- 测试风格：jest，全部就近放在各模块 `__test__/` 目录；排序类有共享 SortTester 保证跨实现一致性；覆盖风格以单元测试为主，个别（如 pngjs 依赖）涉及图像断言
- 覆盖率门槛：statements/functions/lines 100%、branches 95%（`jest.config.js` coverageThreshold），且 CI 强制执行——对一个 360 文件的库来说是罕见的硬标准
- 代码质量：airbnb ESLint 规则 + husky 本地钩子（`package.json` prepare 脚本与 `.husky/` 目录）；JSDoc 注释覆盖率高（抽样 `Comparator.js`、`Sort.js`、`MinHeap.js` 均为每个方法注释）
- 观察局限：实现刻意偏"可读性优先"（如 QuickSort 用 shift()+三路数组而非原地划分，空间复杂度非最优），README 也明确提醒读者注意数据结构取舍而非照搬实现（`README.md` "Remember that each data has its own trade-offs"）

## 8. 社区与维护状态

- 活跃度：创建于 2018-03，最近 push 2026-07-26（gh api 元数据；tarball 文件 mtime 亦为 2026-07-26，说明近期仍有依赖升级/文档更新级别的提交，如 Jest 30、Node 22 适配，见 `package.json`）。距分析日约 3 个月，属"稳定低频维护"而非停更
- 流程：`CONTRIBUTING.md` 提供贡献指引（本仓库修改通常需同步多语言 README，门槛较高）；434 个 open issue、4183 个 watcher、30997 fork（gh api）——大 issue 积压符合此类巨型学习库"提出比修复快"的常态
- 贡献者结构：核心由作者 Oleksii Trekhleb 主导，翻译与新增条目大量来自社区（18 种语言 README、各算法目录内 pt-BR/zh-CN 等局部翻译可作证据）；精确贡献者分布未确认（tarball 无 git 历史，本次未拉取 contributors API）
- 生态位：作者姊妹项目 javascript-questions（面试题库）等；仓库带 BACKERS.md 与乌克兰战争捐助倡议（`README.md` 顶部），有稳定社区赞助背景（BACKERS.md 存在，具体金额未确认）

## 9. 亮点与风险

**亮点**
1. 覆盖广度与深度兼备：28 个分类、约 200 个独立知识点（28 分类目录、111 个算法级 README、360 个 js 文件），从入门链表到线段树、TSP、seam-carving 一站式
2. "实现 + 讲解 + 测试"三位一体的目录范式，且 18 语言翻译，是全 GitHub 学习类仓库的标杆组织方式
3. 工程标准极高：零生产依赖、100% 语句覆盖率硬门槛、airbnb lint、CI 矩阵，教学代码也能当工程范本读
4. 抽象设计克制而有效：单一 Comparator + Sort 基类两个注入点支撑全库泛型化，MinHeap 17 行继承即成
5. 测试即文档：手工搭建场景的用例（如 dijkstra 测试的图构建）本身就是 API 使用教程

**风险/局限**
1. 非生产库：不发布 npm 包、无版本化 API 承诺，代码不可直接 import 到项目当依赖（`package.json` 无 dependencies、顶层无 index.js）
2. 部分实现为可读性牺牲性能（如三路快排的数组分配、递归 DFS），不可作为生产级算法性能参考
3. open issue 434 个，多语言 README（19 份顶层 + 111 个局部）同步成本高，翻译滞后在所难免
4. 多文件修改型贡献门槛高（改一个算法需同步测试与多语言文档），近年更新以维护性提交为主，大规模新增内容趋缓

## 10. 总结评价

javascript-algorithms 是"教学型仓库做到极致"的样本：以几乎为零的依赖、教科书式的目录组织和近 1:1 的测试代码比，把 200 余个算法与数据结构知识点做成了可运行、可验证、可翻译的多语言开源教科书，近 20 万星实至名归。它不适合当生产组件用，但非常适合三类人：刷题/面试者系统补 CS 基础、JS 学习者用真实工程读算法实现、教育者直接取用其"实现+讲解+测试"三件套范式。读源码建议从 `src/utils/comparator/Comparator.js` 与 `src/data-structures/heap/` 入手，理解注入式比较与继承最小化两个母题后，其余 360 个文件只是同一模式在不同知识点上的重演。
