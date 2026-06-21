# 结构:Feature-Sliced Design(定物理落点)

> `consistent-frontend` 参考之二。把 Clean 的抽象圈层(`philosophy-ddd-clean.md`)落成**可强制的目录**与 **import 规则**。回答:**这段代码放哪?谁能 import 谁?**
> 语言/框架无关;目录名是约定,可按团队微调,但**层次语义与依赖方向不可破**。

## FSD 三个维度:Layers × Slices × Segments

```
src/
├─ app/        # 组合根:DI、全局 provider、路由装配、入口
├─ pages/      # 路由页:装配 widgets/features,无业务逻辑
├─ widgets/    # 跨 feature 的组合 UI 块(如 OrderToolbar + OrderList 组成的面板)
├─ features/   # 用例/交互:一个业务动作的完整切片(含状态机)
├─ entities/   # 领域模型:ViewModel + 状态→操作 + ACL
└─ shared/     # 无业务:ui(设计系统)/ lib / config / api
```

### 1. Layer(层)—— 纵向,定"抽象级别 + 依赖方向"

从上到下抽象级别递减、稳定性递增。**依赖只能向下**(对应 Clean 的"依赖向内")。

| Layer | 放什么 | Clean 对应 | 可被谁依赖 |
| :--- | :--- | :--- | :--- |
| `app` | 组合根、DI、全局 provider、路由表 | Main | 无(顶层) |
| `pages` | 路由页,只装配 | 接口适配 | app |
| `widgets` | 跨 feature 的组合块 | 接口适配 | app, pages |
| `features` | 单个业务动作(用例+交互+状态机) | 应用业务规则 | app, pages, widgets |
| `entities` | 领域模型 + ACL | 企业业务规则 | 以上全部 |
| `shared` | 无业务:设计系统/工具/客户端 | 框架与驱动 | 所有层 |

> 注:FSD 官方层是 `app / pages / widgets / features / entities / shared`(旧版的 `processes` 已废弃)。可按需省略(小项目可能没有 `widgets`),但**不可调换上下顺序**。

### 2. Slice(切片)—— 横向,按业务域分

在一层内部,按**业务域**切分(用通用语言命名):`entities/order`、`entities/user`、`features/void-order`、`features/order-list`。

**铁律:同层 slice 之间不能互相直引。** `features/void-order` 不能 import `features/order-list`。需要组合时,**上提到更高层**(widget 或 page)去拼。
好处:每个 slice 是**能整目录删除**而不牵连别处的单元;横向无网状耦合。

### 3. Segment(段)—— slice 内部,按技术职责分

```
entities/order/
├─ model/    # VM 类型、状态→操作声明、store/selector、(纯)领域逻辑
├─ api/      # ACL:请求 + mapper + schema 校验 + 错误翻译(仅 entities 有)
├─ ui/       # 该实体的展示组件(纯 props)
├─ lib/      # 纯函数工具
├─ config/   # 常量、枚举、路由名
└─ index.ts  # public API(唯一对外出口)
```

`features/<x>/` 类似,但通常无 `api`(取数走 entities 或 shared),`model` 里放**状态机/交互逻辑**。

## Public API:slice 只通过 `index` 对外

每个 slice 用 `index.ts`(或框架等价物)声明对外暴露什么;**外部只能 import slice 的 index,不能深入其内部文件**。

```ts
// entities/order/index.ts
export type { OrderVM } from "./model/types";
export { orderActions } from "./model/actions";
export { useOrders } from "./api/queries";
export { OrderRow } from "./ui/OrderRow";
```

好处:slice 内部随便重构,只要 index 不变,外部无感 → 真正的"可替换、可删除"。

## Import 规则(本结构的灵魂,必须 lint 强制)

1. **只向下**:任一模块只能 import **更低层**(`features` 可引 `entities/shared`,反之禁止)。
2. **同层不互引**:同层 slice 之间不直接 import,组合上提到更高层。
3. **走 public API**:跨 slice 只 import 对方 `index`,禁止深引内部文件。
4. **DTO 不出 `api`**:后端 DTO/原始响应只活在 `entities/*/api`(配合 ACL,见哲学篇)。
5. **第三方 UI 库只在 `shared/ui`**:业务层不直引 antd/element/skeleton 等(配合一致性篇)。

## 强制工具(让规则不靠自觉)

- **steiger** —— FSD 官方 linter,直接校验 layer/slice/segment 与依赖规则(推荐,框架无关)。
- **eslint-plugin-boundaries** —— 自定义 element 类型与允许的依赖矩阵。
- **dependency-cruiser** —— 用 `forbidden` 规则画出禁止的依赖方向,可入 CI 出图。

```js
// dependency-cruiser 片段:禁止向上依赖 + 禁止 features 互引(示意)
forbidden: [
  { name: "no-upward", from: { path: "src/entities" }, to: { path: "src/(features|widgets|pages|app)" } },
  { name: "no-cross-feature", from: { path: "src/features/([^/]+)" }, to: { path: "src/features/(?!\\1)" } },
  { name: "dto-stays-in-api", from: { pathNot: "src/entities/[^/]+/api" }, to: { path: "dto" } },
]
```

## co-location 与可删除性

一个 slice 的样式、局部状态、测试、子组件**放一起**。判断结构是否健康的试金石:**"删掉这个 feature 目录,项目还能编译、其他功能不受影响"** —— 能,则边界对了。

## 与另两层的接力

- **哲学**(`philosophy-ddd-clean.md`)给出"领域居中、依赖向内、ACL"的**方向**;本篇把它落成**目录 + import 规则**。
- **一致性**(`consistency-designsystem-statemachine.md`)规定 `shared/ui` 是设计系统唯一真相、`entities/model` 放状态→操作——它们的**物理位置由本篇定义**。
- 各框架如何实现 segment(hook/composable/store、`.tsx`/`.vue`/`.svelte`)见 `framework-adapters.md`。
