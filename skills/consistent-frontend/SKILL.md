---
name: consistent-frontend
description: "构建好维护、强一致的前端的开发 playbook,把三套成熟方法论叠成一条贯通的纪律:用 DDD/Clean Architecture 定依赖方向(领域居中、ACL 隔离后端、通用语言),用 Feature-Sliced Design 定物理落点(layers/slices/segments + import 规则),用 Design System + 状态机 定语义一致(操作→模式映射、状态→操作不变量、设计令牌),让业务怎么迭代,UI/UX 与代码都对得上账。核心框架无关;给出 React / Vue 3 / Svelte 的适配落地。开发新 feature/页面/组件、对接后端、给新项目定架构、统一 UI/UX、或重构'一改就崩/越迭代越散'的模块时使用。"
risk: safe
stage: build
driver: both
source: self
tags: "[frontend, architecture, ddd, clean-architecture, feature-sliced-design, design-system, state-machine, multi-framework, maintainability, ux-consistency]"
---

# Consistent Frontend(强一致 · 好维护的前端)

## 一句话定位

把三套现成方法论**叠成一套**,各管一件事、互不重叠:

| 支柱 | 现行框架 | 它定的是 | 一句话 |
| :--- | :--- | :--- | :--- |
| 哲学 | **DDD / Clean Architecture** | **依赖方向** | 领域稳定居中,框架/后端/UI库/路由都在边缘;依赖只向内;后端形状被 ACL 挡住。 |
| 结构 | **Feature-Sliced Design (FSD)** | **物理落点** | 把上面的"圈层"落成可强制的目录:layers × slices × segments + import 规则。 |
| 一致性 | **Design System + 状态机** | **业务语义** | 同一操作→同一交互模式(设计系统契约)+ 同一状态机;业务变,UI 与代码同步不漂移。 |

> 三者挡三种腐化:哲学挡**后端/框架形状**渗入、结构挡**代码乱放与乱引**、一致性挡**业务语义漂移**。合起来:任何变化的爆炸半径 = 1 处。

## 核心论点:为什么是这三者,而不是只用一个

- 只有 **Clean** 没有 **FSD**:方向对了,但"领域居中"是抽象圈,落到目录全靠自觉 → 还是乱放。FSD 把圈层变成**可 lint 的物理边界**。
- 只有 **FSD** 没有 **Clean/DDD**:目录整齐了,但每个 slice 里仍可能直接吃后端 DTO、用后端语言命名 → 后端一变全崩。DDD 提供**通用语言 + ACL**。
- 只有这两者没有 **Design System/状态机**:架构干净了,但同一个"作废/提交/审批"在各页长不一样、规则一迭代就和实现对不上账 → **语义漂移**。设计系统与状态机把业务语义**锁进可复用契约**。

## 统一模型(一张分层图)

FSD 的 layer 就是 Clean 圈层的物理实现;依赖**只向下**(app→…→shared),同层 slice 互不直引,跨切片组合上提到更高层,对外只走 public API(`index`)。

```
app/       组合根 + DI + 全局 provider + 路由装配         ← Clean: Main / 组合根
pages/     路由页:只装配 widgets/features,无业务逻辑     ← Clean: 接口适配/装配
widgets/   跨 feature 的组合 UI 块                          ← Clean: 接口适配
features/  用例/交互:一个业务动作的完整切片(含状态机)   ← Clean: 应用业务规则(use case)
entities/  领域模型:ViewModel + 状态→操作 + ACL(api)     ← Clean: 企业业务规则 + 反腐层
shared/    无业务:ui(设计系统)/ lib / config / api       ← Clean: 框架与驱动(最外圈)
```

slice 内部按 **segment** 切:`ui`(展示)/ `model`(状态·状态机·VM类型·selector)/ `api`(ACL:请求+mapper+schema,仅 entities)/ `lib`(纯函数)/ `config`(常量·枚举·路由名)。

> 三件"单一真相"各有其位:**通用语言词汇表**(贯穿命名)、**操作→模式映射表**(在 `shared/ui`)、**状态→操作声明 / 状态机**(在 `entities/model`、`features/model`)。

## 三根支柱(详见对应 reference)

1. **哲学:DDD / Clean** — 依赖向内、领域居中;`entities/*/api` 是对后端的**反腐层(ACL)**:mapper(DTO→VM)+ schema 校验 + 错误翻译,DTO 不出此段;命名用**通用语言**(沿用后端领域语言,不另造)。详见 `references/philosophy-ddd-clean.md`。
2. **结构:FSD** — layers/slices/segments + import 规则,用工具(`steiger` / `eslint-plugin-boundaries` / `dependency-cruiser`)强制"只向下、同层不互引、走 public API"。详见 `references/structure-fsd.md`。
3. **一致性:Design System + 状态机** — 交互通用语言、操作→模式映射(绑定设计系统组件契约)、状态→操作不变量(UI 由它推导,组件无散落 `status` 分支)、设计令牌;迭代**改在源头、由表生成、一致性 lint 守门**。详见 `references/consistency-designsystem-statemachine.md`。

## 框架适配(核心框架无关)

所有概念——slice、entity、use case、ViewModel、ACL、状态机、设计系统封装、import 规则——都是**框架无关**的;框架只是用不同机制实现同一概念。VM 类型、mapper、schema、状态机定义、操作模式契约类型、import 规则**本就是纯 TS**,三框架完全共用。

| 概念 | React | Vue 3 | Svelte / SvelteKit |
| :--- | :--- | :--- | :--- |
| 数据入口(读) | hook `useOrders()` + TanStack Query | composable `useOrders()` + Vue Query/Pinia | `createQuery` / load 函数 + store |
| 客户端状态 | Zustand / Context | Pinia / `reactive` | store / `$state` rune |
| 状态机 | XState / `useReducer` | XState / composable | XState / store |
| 展示组件 | `.tsx` 纯函数 | `.vue` SFC(props/emits) | `.svelte`(props) |
| 设计系统封装 | `ui/Button.tsx` | `ui/Button.vue` | `ui/Button.svelte` |
| DI / 组合根 | `<App>` + Providers | `createApp` + `app.provide` | root `+layout` + `setContext` |
| 路由收口 | React Router / Next(在 pages) | vue-router(在 pages) | SvelteKit 文件路由 + `load` |

完整适配表与各框架 slice 目录示例见 `references/framework-adapters.md`。

## 开发流程(由外向内决策 + 由内向外实现)

决策**由外向内**(先把易变的语义/契约收口),实现**由内向外**(领域先稳,UI 后接)。每步先决策、后实现。

0. **对齐语义(地基,贯穿)**。用**交互通用语言**定名词/动词;把业务操作**归类**到操作→模式映射表(复用既有模式,不私造);为涉及实体确认**状态→允许操作**。新建则建这三张表,迭代则**先改表再改码**。
1. **定切片与契约**。这块属于哪个 slice、落哪层;写出稳定契约——ViewModel 类型、数据入口签名、展示组件 props——**再写实现**。用通用语言命名。
2. **建 entities(领域 + ACL)**。`model`:VM 类型 + 状态→操作声明;`api`:请求 + mapper(DTO→VM)+ schema 校验 + 错误翻译。DTO 不出 `api`。
3. **建 features(用例 + 交互)**。把"一个业务动作"做成纵切片:数据入口编排 + 状态机(多步/异步/带重试),不画 UI 业务规则。
4. **建 UI**。`shared/ui` 封装设计系统(第三方库只在此被引);展示组件是 props 纯函数;业务操作**走映射表统一契约**(如破坏性操作走 `ConfirmAction`),可用性**取自状态→操作声明**。
5. **装配 widgets/pages/app**。pages 只装配 + 路由收口;app 做组合根/DI;无业务逻辑、无内联取数。
6. **固化**。把 import 规则与一致性约束写成 lint/CI:只向下依赖、同层不互引、DTO 不出 `api`、业务操作不绕过设计系统契约、组件无 `status` 硬编码。

`--mode retrofit`:②~⑤ 以**切片**为单位,每片先补特征化测试锁现状,再按契约解耦替换,旧路径全程可用、可回滚。

> 完整编排(链路 + 各阶段门禁 G0–G7 + 回溯矩阵)见 `references/workflow.md`;由 `/consistent-frontend:fe-build` 命令触发。

## 产物

| 场景 | 输出 |
| :--- | :--- |
| 语义(地基) | 交互通用语言词汇表 + 操作→模式映射表 + 涉及实体的状态→操作声明 |
| new(0→1) | 该业务动作的 FSD 纵切片:entities(VM+ACL+状态)→ features(用例+状态机)→ shared/ui 契约 → widgets/pages 装配 |
| retrofit | 维护性设计方案 + 按切片渐进重构计划(每片先补特征化测试) |
| 通用 | 该固化的 import 规则 + 一致性约束(lint/CI) |

## 自检清单(交付前对照,三组)

**哲学 / 依赖**

- [ ] 依赖只向内;`shared` 无业务;领域(entities)不依赖框架/路由/UI 库
- [ ] 后端 DTO 只活在 `entities/*/api`(ACL),被 mapper 挡住;命名用通用语言
- [ ] 边界有 schema 运行时校验;外部错误已翻译为本地语义错误
- [ ] 业务不变量归后端(单一真相);前端状态→操作表只是它的投影

**结构 / FSD**

- [ ] 每块代码落在正确 layer;同层 slice 不互相直引;对外只走 public API
- [ ] slice 内按 segment(ui/model/api/lib/config)归位;一个 slice 能整目录删除
- [ ] import 规则已用 lint 强制(steiger / boundaries / dependency-cruiser)

**一致性 / Design System + 状态机**

- [ ] 名词/动词用通用语言统一(VM/组件/路由/文案同词)
- [ ] 同类业务操作用同一交互模式 + 同一设计系统契约(确认/提交/筛选/门控处处一样)
- [ ] 实体可用操作取自集中的状态→操作声明;组件无散落 `status` 分支;复杂交互用状态机
- [ ] 第三方 UI 库只在 `shared/ui` 被引用;设计令牌集中
- [ ] 业务规则变更先改三张表,页面不打补丁;≥1 条一致性约束入 lint

## 两种驱动

- **new(0→1)**:起项目即建三张语义表 + FSD 骨架 + import/一致性 lint,后续每个业务动作复用同一套层次、语言与模式。
- **retrofit(改造既有)**:先逆向**归纳现有操作与措辞**填三张表(常暴露"同一操作三种弹窗"的不一致),再把痛点模块**按切片**迁入 FSD 分层、收口 ACL,每片先补特征化测试锁现状,旧路径全程可用、可回滚。

## 边界与回溯(什么该放别处)

> 注:下文 `ddd-contexts`、`workflow-brownfield` 属于**另一个 DDD 插件**。若已安装,跨插件调用须加该插件前缀(如 `<ddd-plugin>:ddd-contexts`);本插件内不提供这两个 skill。

- **业务不变量归后端**(单一真相)。前端 `状态→操作` 表是后端规则的**投影**,用于推导 UI,不是权威副本。
- **通用语言/操作说不清** → 根因在领域建模,回 DDD `ddd-contexts` 补语言,再回来。
- **某后端概念其实是前端交互生命周期**(复杂向导草稿态)→ 用状态机在 `features` 建模,别硬塞服务端往返。
- **slice 互相想直引** → 说明边界划错或该上提到更高层组合;不要破坏 import 规则。

## 示例(同一切片,三框架一句话适配)

```text
/consistent-frontend:fe-build 订单列表 --mode new --framework react|vue|svelte

0 语义:作废 void(破坏性→ConfirmAction)、退款 refund;
        orderActions{ DRAFT:[submit,edit,void], PAID:[refund,void], REFUNDED:[] }
1 契约:OrderListItemVM{ id; title; statusLabel; amountText };数据入口 useOrders();<OrderList items>
2 entities/order:model(VM + orderActions)+ api(GET → zod 校验 → toOrderVM → 错误翻译)
3 features/order-list:useOrders() 编排 + 筛选状态;features/void-order:作废状态机(确认→提交→回滚)
4 shared/ui:封装设计系统(React=antd / Vue=naive / Svelte=skeleton)成自有 <Button><Table><ConfirmAction>;
        <OrderList> 纯展示,行内"作废"走 ConfirmAction,可用性取自 orderActions[status]
5 pages/orders:装配 + 读路由筛选参数下传;app 注入 query client / provider
6 固化:lint 禁止 features 互引、禁止 DTO 出 entities/api、禁止绕过 ConfirmAction、禁止组件硬编码 order.status

适配差异只在第 3-5 步的"实现机制"(hook / composable / store),第 0-2 步与契约/ACL/状态机定义三框架共用。
```
