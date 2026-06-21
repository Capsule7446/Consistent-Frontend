# 框架适配:React / Vue 3 / Svelte(同一概念,不同机制)

> `consistent-frontend` 参考之四。前三篇的所有概念都是**框架无关**的;本篇给出它们在三大框架里**怎么落地**。原则:**核心逻辑不变,只换实现机制**。
> 覆盖 React、Vue 3、Svelte/SvelteKit。其他框架(Solid/Angular/Qwik…)照"概念→机制"自行映射即可。

## 先分清:哪些根本不用适配

下列产物是**纯 TS、零框架依赖**,三框架**完全共用**,不要为每个框架重写:

- ViewModel 类型、mapper、zod schema、错误翻译(`entities/*/api` 的 ACL,见 `philosophy-ddd-clean.md`)
- 状态→操作声明(`orderActions`)与状态机定义(XState 配置)
- 操作→模式映射表的**契约类型**(`ConfirmActionProps` 等接口)
- 通用语言词汇表、import 规则配置、设计令牌(JSON/TS 常量)

> 把它们放在 `entities/model`、`entities/api`、`shared/config`,框架层只是"消费"它们。框架替换/共存时,这部分零成本迁移。

## 概念 → 三框架机制(总表)

| 概念(FSD/Clean) | React | Vue 3 | Svelte / SvelteKit |
| :--- | :--- | :--- | :--- |
| 数据入口(读) | `useOrders()` hook + TanStack Query | `useOrders()` composable + TanStack Vue Query / Pinia | `createQuery()`(@tanstack/svelte-query)或 SvelteKit `load` + store |
| 写操作 / mutation | `useMutation` | `useMutation`(vue-query)/ Pinia action | `createMutation` / store action |
| 客户端状态(UI/会话) | Zustand / `useState` / Context | Pinia / `reactive` / `ref` | `writable` store / `$state` rune |
| 派生/选择器 | selector + `useMemo` | `computed` | `derived` store / `$derived` |
| 状态机 | XState + `@xstate/react` `useMachine` | XState + `@xstate/vue` `useMachine` | XState + `@xstate/svelte` `useMachine` |
| 展示组件 | `.tsx` 纯函数组件(props) | `.vue` SFC(`defineProps`/`defineEmits`) | `.svelte`(`$props()` / `export let`) |
| 设计系统封装 | `shared/ui/Button.tsx` 包 antd/mui | `shared/ui/Button.vue` 包 element/naive | `shared/ui/Button.svelte` 包 skeleton/flowbite |
| 依赖注入 / 组合根 | `<App>` + Context Providers | `createApp().provide()` / `app.use()` | root `+layout.svelte` + `setContext` |
| 跨层提供能力 | `createContext` + `useContext` | `provide` / `inject` | `setContext` / `getContext` |
| 路由收口(在 pages) | React Router / Next App Router | vue-router | SvelteKit 文件路由 + `load` |
| public API(slice index) | `index.ts` re-export | `index.ts` re-export | `index.ts` re-export |
| import 规则强制 | steiger / eslint-plugin-boundaries / dependency-cruiser | 同 | 同 |

## 各框架 slice 目录示例(`entities/order`)

### React
```
entities/order/
├─ model/   types.ts  actions.ts(orderActions)  store.ts(Zustand,选)
├─ api/     queries.ts(useOrders/useOrder)  mapper.ts  schema.ts(zod)
├─ ui/      OrderRow.tsx(纯 props)
└─ index.ts
```
```tsx
// api/queries.ts —— 数据入口唯一出口
export function useOrders() {
  return useQuery({ queryKey: orderKeys.list(), queryFn: () => api.getOrders().then(parseOrders) });
}
// ui/OrderRow.tsx —— 纯展示,可用性来自 model
export function OrderRow({ title, status, allowed, onVoid }: OrderRowProps) { /* allowed.includes('void') */ }
```

### Vue 3
```
entities/order/
├─ model/   types.ts  actions.ts  store.ts(Pinia,选)
├─ api/     queries.ts(useOrders composable)  mapper.ts  schema.ts(zod)  ← mapper/schema 与 React 同文件
├─ ui/      OrderRow.vue(defineProps)
└─ index.ts
```
```ts
// api/queries.ts
export function useOrders() {
  return useQuery({ queryKey: orderKeys.list(), queryFn: () => api.getOrders().then(parseOrders) });
}
```
```vue
<!-- ui/OrderRow.vue —— 纯展示 -->
<script setup lang="ts">
const props = defineProps<OrderRowProps>();
const emit = defineEmits<{ void: [id: OrderId] }>();
</script>
```

### Svelte / SvelteKit
```
entities/order/
├─ model/   types.ts  actions.ts  store.ts(writable / runes,选)
├─ api/     queries.ts(createQuery 或被 +page.ts 的 load 调用)  mapper.ts  schema.ts(zod)
├─ ui/      OrderRow.svelte($props)
└─ index.ts
```
```ts
// api/queries.ts
export const ordersQuery = () =>
  createQuery({ queryKey: orderKeys.list(), queryFn: () => api.getOrders().then(parseOrders) });
```
```svelte
<!-- ui/OrderRow.svelte —— 纯展示(Svelte 5 runes) -->
<script lang="ts">
  let { title, status, allowed, onVoid }: OrderRowProps = $props();
</script>
```

## 设计系统封装:同一契约,三种壳

操作→模式映射表里的 `ConfirmAction` 契约(框架无关接口),三框架各实现一个壳,业务层只认自有组件:

```ts
// shared/ui/contracts.ts —— 框架无关,三框架共用
export interface ConfirmActionProps { impact: string; danger?: boolean; onConfirm: () => void }
```
- React:`shared/ui/ConfirmAction.tsx` 内部用 antd `Modal.confirm` 或自绘。
- Vue:`shared/ui/ConfirmAction.vue` 内部用 naive `useDialog` 或 element `ElMessageBox`。
- Svelte:`shared/ui/ConfirmAction.svelte` 内部用 skeleton modal store 或自绘。

换底层 UI 库 → 只改这三个文件之一,业务层零改动。

## 组合根 / DI 差异(`app` 层)

| | 注入 query client / 全局 provider | 路由装配 |
| :--- | :--- | :--- |
| React | `<QueryClientProvider>` 包 `<App>` | `<RouterProvider>` / Next layout |
| Vue 3 | `app.use(VueQueryPlugin)`, `app.provide(key, val)` | `app.use(router)` |
| Svelte | root `+layout.svelte` 里 `setContext` / `<QueryClientProvider>` | SvelteKit 文件路由(目录即路由) |

## 框架特有的坑(适配时注意)

- **React**:RSC/Server Components 下,数据入口可能下沉到 server;保持"组件吃 props、不内联 fetch"仍成立,server 部分相当于 `app/pages` 层取数。`"use client"` 边界尽量压在 widgets/features。
- **Vue 3**:composable 命名以 `use` 开头;`defineProps` 用泛型保持 props 是最小契约;别把整个响应式实体传进展示组件(破坏最小 props)。
- **Svelte 5**:用 runes(`$state`/`$derived`/`$props`)替代旧 `export let`/store 习惯;SvelteKit 的 `load` 天然是"取数收口在 pages 层",非常贴合本架构——把 ACL 放在 `load` 调用的 `entities/api` 里,组件只收 `data`。
- **通用**:无论哪个框架,**展示组件不读路由、不读全局、不取数**;路由参数在 `pages` 层读、以 props 下传(SvelteKit 用 `load`+`data`,Vue/React 用 router hook 在 page 读)。

## 多框架共存 / 迁移

因为领域层(ACL/VM/状态机/操作契约类型)是纯 TS,**渐进迁移**很自然:新框架重写的只有 `ui` 段与数据入口的"壳",`entities/model`、`entities/api`、`shared/config` 原样复用。这也是 retrofit(`SKILL.md` 两种驱动)能按切片安全替换的底层原因。
