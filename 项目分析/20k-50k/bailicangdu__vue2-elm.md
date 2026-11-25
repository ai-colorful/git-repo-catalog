# bailicangdu/vue2-elm 项目分析

> 星数 42500（快照 2026-10-24）｜主语言 Vue｜档位 20k-50k｜分析日期 2026-11-14

## 1. 项目速览

一句话定位：Vue2 时代最著名的仿饿了么外卖平台大型单页应用实战项目——45 个页面、覆盖登录到下单完整业务闭环的"前端练兵场"。

| 项 | 值 | 出处 |
|---|---|---|
| 名称 | bailicangdu/vue2-elm | gh api |
| 星数 | 42500（快照）；分析日 gh api 实测 40950 | gh api |
| Fork | 12190 | gh api |
| 主语言 | Vue | gh api |
| 许可证 | GPL-2.0（COPYING 文件） | src/COPYING |
| 创建时间 | 2016-12-23 | gh api |
| 最近 push | 2025-04-11（"docs: update readme"） | gh api、最近提交 |
| Release/Tag | 0 个 | gh api |
| 贡献者 | 12 人 | gh api contributors |
| 代码规模 | 55 个 .vue（其中页面 45 个），js+vue 约 1.3 万行 | find/wc 统计 |

## 2. 项目是什么，解决什么问题

这是作者 bailicangdu（苍渡）2016 年底启动的个人实战项目：用 Vue2 完整复刻饿了么 web App 的核心流程——定位选城市、浏览附近商家、筛选排序、进店看菜、加购物车、确认订单（地址/备注/发票）、下单、查看订单、个人中心（余额/红包/积分/会员卡/收货地址）。README 明确说明动机：当时网上的 Vue 教程项目多是简单 demo 或音乐播放器这类展示型应用，缺少涉及 money、交互复杂、逻辑严谨的完整业务项目，作者遂以"足够复杂、且看着不腻"的饿了么为素材，从零写了两个多月完成（README.md「前言」「总结」）。

项目强调完整性而非技术炫技：45 个页面包含大量多级嵌套的子页（如确认订单 → 选择地址 → 添加地址 → 搜索地址，路由嵌套 5 层），这正是真实外卖业务的形态。配套生态是其重要部分：后端 `node-elm`（Node.js 模拟饿了么接口）、管理后台 `vue2-manage`（element-ui）、入门级练手项目 `vue2-happyfri`、React 对照项目 `react-pxq`（README.md「关于接口数据」「最终目标」）。

线上演示地址为 `cangdu.org/elm`，生产 baseUrl 指向 `elm.cangdu.org`（src/config/env.js）。该项目在 2017-2019 年间几乎是中文社区 Vue2 入门的"标配实战教程"，4 万+ 星绝大多数来自这一时期的学习者。README 末尾"项目已经完成，共45个页面"即终态宣言，此后仓库仅做零星维护。

## 3. 技术栈与依赖

技术栈带有鲜明的 2016 年印记（出处 src/package.json）：

- **框架**：vue ^2.1.0、vue-router ^2.1.1、vuex ^2.0.0——均为 2.x 最初期版本。
- **构建**：webpack ^1.13.2（注意：任务简介与社区俗称"webpack2"，但 package.json 实际是 webpack 1，build/webpack.base.conf.js 也全部是 webpack1 语法：`module.loaders`、`extensions` 含空串、`resolveLoader.fallback`、loader 短名 `vue`/`babel`/`json`）；babel 6（presets es2015 + stage-2，.babelrc）。
- **样式**：sass-loader ^4 + node-sass 4、less-loader 2，rem 移动端适配（src/config/rem.js：320px 设计稿下 1rem=20px）。
- **滚动/交互**：better-scroll 1.15.2、iscroll 5、fastclick。
- **网络**：原生 fetch + XMLHttpRequest 手写降级封装（src/config/fetch.js），无 axios——2016 年 axios 尚未普及的典型写法。
- **其他**：showdown（markdown 渲染）、vue-loader ^10、extract-text-webpack-plugin ^1（webpack1 专用 CSS 抽取插件）。
- node-sass 4、webpack 1、babel 6 在现代 Node 环境下基本无法安装构建，属于"年代冻结"项目。

## 4. 目录结构与代码组织

| 目录 | 职责 | 出处 |
|---|---|---|
| build/ | webpack1 构建脚本：dev-server（express + http-proxy-middleware）、dev/prod 配置、build.js | src/build/ |
| config/ | 构建参数：dev 端口 8000、接口代理路径前缀、产物输出到 elm/ 且 publicPath 为 /elm/ | src/config/index.js |
| src/src/page/ | 18 个业务模块目录，45 个页面 .vue，children 多级嵌套（最深 5 层目录对应 5 层路由） | src/src/page/ |
| src/src/components/ | 公共组件：header/head、footer/footGuide、common/ 下 8 个（buyCart 购物车、shoplist 商铺列表、alertTip、loading、ratingStar、computeTime、svg、mixin.js） | src/src/components/ |
| src/src/store/ | Vuex 单 store：index/mutations/action/getters/mutation-types，**无 modules 目录**（README 布局图提到的 modules 与 service/tempdata 实际不存在，README 已过时） | src/src/store/、find 对比 |
| src/src/service/ | getData.js 唯一文件，45 个接口函数统一管理 | src/src/service/getData.js |
| src/src/config/ | env.js 环境切换、fetch.js 网络封装、mUtils.js 工具（含 localStorage 存取）、rem.js | src/src/config/ |
| src/src/style/ | common.scss / mixin.scss 公共样式 | src/src/style/ |
| elm/ | 构建产物（index.html + static/js 三个 chunk：app/manifest/vendor），当年直接部署服务器的做法 | src/elm/ |
| screenshots/ | 项目截图与演示 GIF（约 17 个，占仓库体积大头） | src/screenshots/ |

组织方式：经典"页面目录 + children 子页"的单体 SPA，按业务域分目录而非组件类型分目录；每个页面是自包含的 .vue（msite.vue 199 行，最复杂的 shop.vue 达 1585 行）。

## 5. 核心架构与关键实现

### 5.1 路由：require.ensure 懒加载 + 5 层嵌套（src/src/router/router.js）

46 个页面组件全部用 webpack1 时代的 `require.ensure` 按命名 chunk 懒加载（而非 `import()` 动态导入），路由表单文件集中定义，`/profile/info/address/add/addDetail` 嵌套四层、confirmOrder 链路嵌套五层，忠实还原业务层级。main.js 中自定义 `scrollBehavior`，配合 `meta.keepAlive` 手动保存/恢复列表页滚动位置（src/src/main.js）：

```js
scrollBehavior (to, from, savedPosition) {
    if (savedPosition) { return savedPosition }
    else {
        if (from.meta.keepAlive) { from.meta.savedPosition = document.body.scrollTop; }
        return { x: 0, y: to.meta.savedPosition || 0 }
    }
}
```

### 5.2 Vuex：单 store + 购物车四层树 + localStorage 持久化（src/src/store/）

没有 modules 拆分，一个 store 承载 24 个状态键（位置、购物车、用户、订单流程参数），29 个 mutation 常量集中在 mutation-types.js；actions 仅 2 个（getUserInfo、saveAddress，src/store/action.js），其余状态变更由页面组件直接 commit——"集中 mutations、薄 actions、空 getters"是当时社区流行写法（src/store/getters.js 实际导出空对象）。购物车是核心亮点：cartList 按 `shopid → category_id → item_id → food_id` 四层嵌套组织，增减商品后整体写入 localStorage（key 为 buyCart），刷新后由 INIT_BUYCART 恢复（src/store/mutations.js）：

```js
let cart = state.cartList;
let shop = cart[shopid] = (cart[shopid] || {});
let category = shop[category_id] = (shop[category_id] || {});
let item = category[item_id] = (category[item_id] || {});
if (item[food_id]) { item[food_id]['num']++; }
else { item[food_id] = { "num": 1, "id": food_id, "name": name, ... }; }
state.cartList = {...cart};
setStore('buyCart', state.cartList);
```

buyCart.vue 组件用展开运算符把 ADD_CART/REDUCE_CART 映射进 methods，加减商品时 emit `showMoveDot` 驱动经典的"商品小球落入购物车"抛物线动画（src/src/components/common/buyCart.vue）。

### 5.3 网络层：fetch 封装 + 45 接口统一管理（src/src/config/fetch.js、src/src/service/getData.js）

fetch.js 是带 XHR 降级的 fetch 封装：GET 手拼 query、POST stringify body、`credentials: 'include'` 携带 cookie 会话。getData.js 471 行集中定义 45 个接口函数，路径完全对齐 node-elm 后端（也是当年饿了么真实 API 的形状）：`/v1/cities`、`/shopping/restaurants`、`/shopping/v2/menu`、`/ugc/v2/restaurants/.../ratings`、`/v1/carts/checkout`、`/payapi/payment/queryOrder` 等。开发环境由 dev-server 把 `/shopping /ugc /v1 /v2 /bos /payapi` 等前缀代理到后端（npm run local 代理 localhost:8001，npm run dev 代理 elm.cangdu.org；src/build/dev-server.js、src/config/index.js）。

### 5.4 移动端适配与页面组织（src/src/config/rem.js、src/src/page/shop/shop.vue）

rem.js 按视口宽度动态设置根字号（320px 设计稿 → 1rem=20px），全站用 scss mixin 统一样式变量。最复杂的 shop.vue（1585 行）在单文件里完成菜单分类滚动联动、购物车增减、小球动画坐标计算、评论/详情 tab 切换——一个文件扛起整个"店铺页"，是理解当年"页面级大组件"风格的最好样本。

## 6. 构建与运行

来自 package.json scripts 与 README（注意：在现代 Node 下因 node-sass/webpack1 依赖大概率无法直接安装成功）：

```bash
npm install        # 或 yarn（仓库带 yarn.lock）
npm run dev        # NODE_ENV=online，接口代理到 https://elm.cangdu.org
npm run local      # NODE_ENV=local，代理 localhost:8001，需先启动配套 node-elm 后端
npm run build      # node build/build.js，产物输出到 elm/ 目录，publicPath=/elm/
```

仓库自带一份已构建产物（src/elm/），README 称"放在服务器即可正常访问"。无 lint、无测试脚本。

## 7. 测试与工程质量

- **测试**：无任何测试目录、无测试框架依赖（package.json 与 find 全树确认），0 个测试文件。
- **CI**：仅一个 GitHub Actions workflow（.github/workflows/close_inactive_issue.yml），每日定时用 issues-helper 自动关闭 30 天无响应的 inactive issue——是"维护收缩期"的典型配置，不是构建/测试 CI。
- **代码质量观察**：代码风格统一（tab 缩进、业务注释充分，组件级中文注释是教学价值的一部分）；但存在明显的腐化痕迹——store/mutations.js 顶部 `import {localapi, proapi} from 'src/config/env'` 引用了 env.js 已不再导出的变量（死代码）；README 项目布局图与实际目录不符（modules/、tempdata/ 不存在）；getters.js 为空文件；无 TypeScript、无 ESLint。

## 8. 社区与维护状态

- **活跃度**：创建于 2016-12-23，最后一次 push 为 2025-04-11（gh api，commit "docs: update readme"），截至分析日（2026-11-14）已停滞约 19 个月；README 末尾"项目已经完成"表明主体开发在 2017 年初即结束，其后多年仅零星修复。
- **Release**：0 个 release、0 个 tag——从未版本化发布（gh api releases/tags）。
- **贡献者**：12 人，主要贡献者是作者本人，README 特别致谢 @bailichen、@raoenhui 两位协作者。
- **Issue 管理**：当前 open issue 仅 2 个，配合每日自动关闭机器人，处于"只关不开"的收尾状态（gh api + workflow 文件）。
- **社区角色**：4 万+ 星、1.2 万 fork 的体量说明其作为"Vue2 入门标配教程"被大量学习者 clone/star，属于中文前端社区的现象级教学仓库；姊妹项目 node-elm（后端）同样为高星教学仓库。

## 9. 亮点与风险

**亮点**

1. 完整业务闭环：登录注册、定位、搜索筛选、购物车、下单、订单、账户体系一应俱全，45 页面规模在 2017 年的前端开源实战项目中极为稀缺（README「目标功能」全勾选，仅"付款"标注做不到）。
2. Vuex 购物车设计经典：四层嵌套 cartList + localStorage 持久化 + 小球飞入动画，至今仍是面试题与教学素材的高频原型（src/store/mutations.js、src/components/common/buyCart.vue）。
3. 接口层组织清晰：fetch 封装 + getData.js 45 接口集中管理 + dev-server 前缀代理，前后端联调方案（npm run local）对新手仍有示范意义。
4. 前后端全家桶生态：vue2-elm + node-elm + vue2-manage 构成可本地完整跑通的教学闭环，这是它区别于纯 demo 项目的关键。
5. 注释详尽：目录树级中文注释（README）+ 组件内注释，降低了当年初学者的阅读门槛。

**风险/局限**

1. 技术栈冻结在 2016：webpack1、babel6、node-sass 4、vue-loader 10，在当前 Node 环境基本装不上、跑不起来，不可作为新项目模板。
2. 后端依赖失效风险：生产 baseUrl `elm.cangdu.org`（src/config/env.js）与 demo 站 cangdu.org 的存活状态未确认，接口大概率已失效；要复现需自建 node-elm。
3. 工程质量短板：零测试、零 lint、无版本化 release，README 与实际目录/依赖版本有出入（webpack 版本、modules 目录）。
4. 架构已过时：require.ensure 懒加载、单 store 无模块拆分、页面级千行大组件，均与 Vue3 + Vite + Pinia 的现代实践脱节，直接模仿会学到过时模式。

## 10. 总结评价

vue2-elm 是中文前端社区"教学型高星仓库"的标杆：它用 45 个页面、1.3 万行代码把一个外卖平台的完整业务流程在 Vue2 上跑通，靠的不是技术先进性而是完成度和真实感，因此在 2017-2019 年成为无数 Vue2 学习者的第一个"大项目"，星数也主要沉淀于那一时期。如今它已停止维护近两年（最后提交 2025-04-11），栈时代（webpack1/babel6/Vue2 早期）与后端依赖（elm.cangdu.org）都已成为历史标本，其价值从"可运行的项目"转变为"可阅读的教材"：适合想理解"SPA 如何组织复杂业务状态"的初学者对照旧时代最佳实践做源码阅读，尤其推荐 store/mutations.js 的购物车设计与 router.js 的深层嵌套路由；但不适合作为新工程的起点，现代学习者应以它理解演进脉络，而非复制模式。
