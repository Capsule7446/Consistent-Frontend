---
description: 按 consistent-frontend 三叠加方法(DDD/Clean 定方向 · FSD 定结构 · Design System/状态机 定一致)构建一个业务动作/页面/组件,或重构既有模块。决策由外向内、实现由内向外。
argument-hint: <业务动作/页面/组件/接口> [--mode new|retrofit] [--framework react|vue|svelte] [--stack "..."]
---

# /consistent-frontend:fe-build

你按 `consistent-frontend:consistent-frontend` Skill 帮用户构建**强一致、好维护**的前端。核心:把三套方法叠成一条纪律——**DDD/Clean 定依赖方向、FSD 定物理落点、Design System+状态机 定业务语义一致**,让业务怎么迭代,UI/UX 与代码都对得上账。核心逻辑框架无关,按 `--framework` 给出落地机制。

> 配套 skill:`consistent-frontend:consistent-frontend`;下文 `references/*.md` 均指 `${CLAUDE_PLUGIN_ROOT}/skills/consistent-frontend/references/`(完整编排见其中 `workflow.md`)。

## 参数

- `$ARGUMENTS`:要构建/改造的目标(业务动作、页面、组件,或要对接的接口,必需)。
- `--mode`:`new`(0→1)| `retrofit`(改造既有)。缺省按路径是否已有实现自动判定。
- `--framework`:`react` | `vue` | `svelte`。缺省沿用现有项目;决定实现机制(hook/composable/store 等),不改变核心结构。
- `--stack`:细化技术栈(状态库、取数库、UI 库),缺省沿用现有。

## 你要做的(由外向内决策、由内向外实现,逐步与用户确认)

0. **对齐语义**(地基):用**交互通用语言**定名词/动词;把业务操作**归类**到操作→模式映射表(复用既有模式,不私造);为涉及实体确认**状态→允许操作**。新建则建三张表,迭代则**先改表再改码**。参见 `references/consistency-designsystem-statemachine.md`。
1. **定切片与契约**:确定它属于哪个 slice、落哪层(entities/features/widgets/pages);写出稳定契约——VM 类型、数据入口签名、展示组件 props——再写实现,用通用语言命名。参见 `references/structure-fsd.md`。
2. **建 entities(领域 + ACL)**:`model`(VM + 状态→操作)+ `api`(请求 → schema 校验 → mapper → 错误翻译)。DTO 不出 `api`。参见 `references/philosophy-ddd-clean.md`。
3. **建 features(用例 + 交互)**:把"一个业务动作"做成纵切片,数据入口编排 + 状态机(多步/异步/带重试)。
4. **建 UI**:`shared/ui` 封装设计系统(第三方库只在此引用);展示组件是 props 纯函数;业务操作走映射表统一契约,可用性取自状态→操作声明,不硬编码 `status`。
5. **装配**:pages 只装配 + 路由收口;app 做组合根/DI;无业务逻辑、无内联取数。
6. **固化**:产出 import 规则 lint(只向下、同层不互引、DTO 不出 api)+ 一致性约束(不绕过设计系统契约、组件无 status 分支),入 CI。

`--mode retrofit`:先逆向归纳现有操作与措辞填三张表(常暴露不一致),再把痛点模块**按切片**迁入 FSD、收口 ACL;每片先补特征化测试锁现状,旧路径全程可用、可回滚。

## 框架落地

按 `--framework` 选实现机制,但**结构、ACL、状态机、操作契约保持一致**(它们是纯 TS、三框架共用)。映射:数据入口 = hook(React)/ composable(Vue)/ createQuery·load(Svelte);展示 = `.tsx`/`.vue`/`.svelte`;DI = Providers / `app.provide` / `setContext`。完整表见 `references/framework-adapters.md`。

## 原则

- 对照 Skill 的**三组自检清单**(哲学/结构/一致性)交付,不是写完就算。
- 依赖只向内;后端 DTO 只活在 `entities/api`;同类操作处处同模式同契约;可用性由状态→操作声明推导。
- 迭代**先改三张表、再改下游代码**,严禁页面打补丁。
- 业务不变量归后端(单一真相);前端状态→操作表是其投影,非权威副本。
- 通用语言/操作说不清,根因在领域建模,回 DDD `ddd-contexts` 补语言。

## 示例

```
/fe-build 订单列表 --mode new --framework react --stack "tanstack-query + zustand + xstate + antd"
/fe-build 订单作废 --mode new --framework vue
/fe-build src/pages/Dashboard --mode retrofit --framework svelte
```
