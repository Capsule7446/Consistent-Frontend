# Consistent Frontend(强一致 · 好维护的前端)

> 一个 Claude Code / Cowork **插件(plugin)**:`consistent-frontend`。提供一个 skill + 一个命令。
> 把三套成熟方法论**叠成一条贯通的纪律**:DDD/Clean 定方向、Feature-Sliced Design 定结构、Design System + 状态机 定语义。
> 目标:让代码改起来便宜、让业务怎么迭代,UI/UX 与代码都对得上账。核心框架无关,给出 React / Vue 3 / Svelte 的落地适配。

## 它解决什么

前端难维护有三种腐化,分别由三套方法各管一件事、互不重叠:

| 支柱 | 现行框架 | 它定的 | 挡哪种腐化 |
| :--- | :--- | :--- | :--- |
| 哲学 | **DDD / Clean Architecture** | 依赖方向(领域居中、ACL 隔离后端、通用语言) | 后端/框架形状渗入 |
| 结构 | **Feature-Sliced Design** | 物理落点(layers × slices × segments + import 规则) | 代码乱放与乱引 |
| 一致性 | **Design System + 状态机** | 业务语义(操作→模式映射、状态→操作不变量、设计令牌) | 业务语义漂移 |

为什么要三个一起:只用 Clean,圈层落地全靠自觉;只用 FSD,目录整齐但仍直吃后端 DTO;只用这两个,同一操作各页长不一样、规则一迭代就和实现对不上账。三者叠加,任何变化的爆炸半径 = 1 处。

支持两种驱动:**new**(0→1)与 **retrofit**(按切片渐进迁入)。核心逻辑框架无关:VM/mapper/schema/状态机/操作契约本就是纯 TS,三框架共用;框架只换实现机制(hook / composable / store)。

## 统一模型

FSD 的 layer 就是 Clean 圈层的物理实现,依赖只向下:

```
app/       组合根 + DI + 路由装配          ← Clean: Main
pages/     路由页:只装配,无业务逻辑       ← Clean: 接口适配
widgets/   跨 feature 的组合 UI 块          ← Clean: 接口适配
features/  用例/交互(含状态机)            ← Clean: 应用业务规则
entities/  领域模型 VM + 状态→操作 + ACL    ← Clean: 企业业务规则 + 反腐层
shared/    无业务:ui(设计系统)/lib/config ← Clean: 框架与驱动
```

slice 内按 segment 切:`ui`/`model`/`api`(仅 entities,DTO 不外泄)/`lib`/`config`;对外只走 public API。

## 插件结构

```
consistent-frontend/                       # 插件根(本工作区文件夹)
├── .claude-plugin/
│   └── plugin.json                        # 插件清单(name / version / description)
├── skills/
│   └── consistent-frontend/               # 主 skill(调用名 consistent-frontend:consistent-frontend)
│       ├── SKILL.md                       # 主 playbook:三叠加统一模型 + 流程 + 三组自检清单
│       └── references/
│           ├── philosophy-ddd-clean.md            # 哲学:依赖方向 / 通用语言 / ACL / 依赖倒置
│           ├── structure-fsd.md                   # 结构:layers × slices × segments / import 规则
│           ├── consistency-designsystem-statemachine.md  # 一致性:通用语言 / 操作→模式 / 状态机 / 令牌
│           ├── framework-adapters.md              # 框架适配:React / Vue 3 / Svelte 概念→机制总表
│           └── workflow.md                        # 编排:链路 + 门禁(G0–G7)+ 回溯矩阵
├── commands/
│   └── fe-build.md                        # 命令 /consistent-frontend:fe-build
└── README.md
```

> 插件内调用用命名空间:skill 为 `consistent-frontend:consistent-frontend`,命令为 `/consistent-frontend:fe-build`;skill 内部引用自带 `references/*.md`,命令引用 skill 文件用 `${CLAUDE_PLUGIN_ROOT}/skills/consistent-frontend/references/`。

## 与 DDD 体系的衔接

前端不另起炉灶,而是把后端 DDD 的成果沿用到表现层:

| DDD 概念 | 前端对应物 |
| :--- | :--- |
| 通用语言 | 交互通用语言(VM/组件/路由/文案同词) |
| 限界上下文 | FSD slice / 呈现上下文 + import 边界 |
| 聚合 + 不变量 | 实体 VM + 状态→操作声明 / 状态机 |
| 应用服务 / 用例 | features:数据入口 + 交互状态机 |
| 反腐层(ACL) | entities/api:mapper / schema / 错误翻译 |
| 领域命令 | 操作→模式映射 |

通用语言/操作说不清时,根因在领域建模,回 DDD `ddd-contexts` 补语言;retrofit 的切片化 + 特征化测试,与 DDD `workflow-brownfield` 同纪律。

> `ddd-contexts`、`workflow-brownfield` 属于**另一个 DDD 插件**;若已安装,跨插件引用须加该插件前缀(如 `<ddd-plugin>:ddd-contexts`),本插件不内置它们。

## 安装(Claude Code / Cowork)

这是一个标准插件。安装方式任选其一:

- **打包安装**:把插件根目录打成 `.plugin`(`zip -r consistent-frontend.plugin .`),在 Cowork 中导入。
- **marketplace**:把本目录加入插件 marketplace,再从中安装。
- **本地放置**:把整个插件目录放入 Claude Code 的 plugins 目录,确保 `.claude-plugin/plugin.json` 在根。

安装后调用:

```
/consistent-frontend:fe-build 订单列表 --mode new --framework react --stack "tanstack-query + zustand + xstate + antd"
/consistent-frontend:fe-build 订单作废 --mode new --framework vue
/consistent-frontend:fe-build src/pages/Dashboard --mode retrofit --framework svelte
```
