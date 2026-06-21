# 哲学:DDD / Clean Architecture(定依赖方向)

> `consistent-frontend` 参考之一。回答一个问题:**谁该依赖谁?** 让领域稳定居中,框架、后端、UI 库、路由全在边缘,依赖只向内。后端怎么变,被一层薄薄的反腐层挡住。
> 语言/框架无关;`api` 段示例用 TS + zod 表意。

## 一句话

**易变的东西不能被稳定的东西依赖。** 框架、HTTP 客户端、后端字段、UI 库、路由都是易变(volatile);业务领域(实体、用例、规则)是相对稳定。所以:稳定的领域居中,易变的实现在外圈,**依赖从外指向内**,绝不反向。

## 依赖方向(Clean 圈层 → 前端)

```
        ┌────────────────────────────────────────────┐
        │ 框架与驱动(最外圈,最易变)                  │  shared/ (ui 库封装, http 客户端, 路由)
        │   ┌──────────────────────────────────────┐  │
        │   │ 接口适配(装配/编排)                  │  │  app / pages / widgets
        │   │   ┌──────────────────────────────┐   │  │
        │   │   │ 应用业务规则(用例/交互)      │   │  │  features (含状态机)
        │   │   │   ┌──────────────────────┐   │   │  │
        │   │   │   │ 企业业务规则(领域)  │   │   │  │  entities (VM + 状态→操作 + ACL)
        │   │   │   └──────────────────────┘   │   │  │
        │   │   └──────────────────────────────┘   │  │
        │   └──────────────────────────────────────┘  │
        └────────────────────────────────────────────┘
                       依赖只向内 ──▶
```

> 这套圈层落到**物理目录**就是 FSD 的 layers(见 `structure-fsd.md`);本篇只讲"为什么这样分、依赖为什么向内"。

## 通用语言(Ubiquitous Language)贯穿命名

DDD 的第一纪律:团队对业务用**同一套词**,并让这套词出现在**代码里**。前端不另起炉灶,**沿用后端领域语言**:

- 实体、VM 字段、用例、组件、路由、文案,统一用业务词(`void`/作废,不混 `cancel`/`delete`)。
- 词义说不清、同概念多词,根因在领域建模 → 回 DDD `ddd-contexts` 对齐,而非前端私造。
- 通用语言是后续"操作→模式映射"的基础:先有共同词,才能谈"同类操作同一模式"(见 `consistency-designsystem-statemachine.md`)。

## 反腐层 ACL —— 对后端的薄边界

后端的形状是最大的易变源。把"和后端打交道"关进一个**独占的薄层**(物理上在 `entities/<slice>/api`),它对内只暴露你的 ViewModel,对外吸收后端所有不确定性。四个零件:

### 1. ViewModel —— 你自己的稳定形状

定义组件**真正想要**的形状,不是后端**碰巧返回**的形状。

```ts
type OrderVM = { id: OrderId; title: string; statusLabel: string; amountText: string };
```

### 2. Mapper —— 唯一同时认识两边的地方

DTO → VM 的翻译,含**命名翻译**(`ord_id → id`)与**格式语义**(状态码→文案、分→金额文本)。后端改字段/命名/结构 → 只改 mapper,组件无感。

```ts
const toOrderVM = (d: OrderDTO): OrderVM => ({
  id: d.ord_id,
  title: d.subject,
  statusLabel: d.st === "PAID" ? "已支付" : "待支付",
  amountText: formatMoney(d.amount_cents),
});
```

### 3. Schema 校验 —— 让契约漂移响亮失败

DTO 进 mapper 前做运行时校验;后端少给字段/类型变了,在**边界**就报错,而不是渗成深层 `undefined`。

```ts
const OrderDTO = z.object({ ord_id: z.string(), st: z.enum(["NEW","PAID"]), amount_cents: z.number() });
export const parseOrder = (raw: unknown): OrderVM => toOrderVM(OrderDTO.parse(raw));
```

### 4. 错误翻译 —— 上层只认你的语义错误

把 HTTP 状态码/后端错误码翻译成本地语义错误,组件不到处 `if (status === 409)`。

```ts
function translateError(e: HttpError): AppError {
  if (e.status === 409) return new ConflictError("订单已被他人修改");
  if (e.status === 422) return new ValidationError(e.body);
  return new UnknownError();
}
```

> 这四个零件是**纯 TS、框架无关**,三框架完全共用——它们是领域层,本就不该知道 React/Vue/Svelte 的存在。

## 依赖倒置:领域不认实现

领域/用例需要"取数据""发命令"时,依赖的是**自己定义的接口/签名**,不是具体的 `axios`/`fetch`/某 query 库。具体实现(http 客户端、缓存库)放在外圈 `shared/api`,在组合根注入。

- 好处:换取数库、换 HTTP 客户端、Mock 测试,都只动外圈;领域纹丝不动。
- 前端常见形态:数据入口(hook/composable/store)对组件暴露**稳定签名**,内部用什么库是细节。

## 单一真相:业务不变量归后端

- 权威业务规则(能不能退款、额度多少)归后端裁决。前端可做**即时校验**改善体验,但别重复实现权威规则,否则两份真相、双向腐化。
- 前端的"状态→操作"声明(见一致性篇)是后端规则的**投影**,用于推导 UI 可用性,**不是**权威副本。

## 它和另两层的接力

- 哲学说"领域居中、依赖向内、ACL 隔离后端"——但这是抽象圈。
- **FSD**(`structure-fsd.md`)把这些圈变成**可 lint 的物理目录与 import 规则**。
- **一致性层**(`consistency-designsystem-statemachine.md`)在领域之上,保证"同一业务语义→同一 UI 与代码"。
