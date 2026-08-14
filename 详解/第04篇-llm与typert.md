# 【第 04 篇】packages/llm/ 与 typert：模型适配缝与类型系统

> 难度：🟡 进阶（建议先读第 02 篇）
> 前置阅读：`第02篇-core产品主干.md`
> 对应目录：`deepseek-harness/packages/llm/`、`deepseek-harness/packages/typert/`

## 目录

- [0 功能需求（WHY）](#0-功能需求why)
- [1 架构设计（WHAT）](#1-架构设计what)
- [2 实现落点（HOW）](#2-实现落点how)
- [3 产物演示（EXAMPLE）](#3-产物演示example)
- [4 动手验证](#4-动手验证🧪)
- [5 FAQ 与自测](#5-faq-与自测❓)
- [6 延伸阅读](#6-延伸阅读📚)

---

## 0 功能需求（WHY）

### 0.1 背景与场景

第 02 篇讲过：core 的循环"流式获取模型响应"（`llm/stream`），但 core **不拥有**模型协议——模型词汇、流式协议、适配器契约全部属于 `packages/llm/`。为什么这样切？两个场景：

1. **多提供商**：同一产品要接 DeepSeek、PI-AI、用户自建网关——每次请求都不同，但循环代码必须**一行不改**；
2. **远程调用类型安全**：GUI 前端、SDK 客户端要调用宿主的能力（查会话、查工具），走的是**生成式类型契约**（typert）——"远程方法"在编译期就有类型，而不是手写 JSON。

### 0.2 需求陈述

**R1 · 词汇统一**——`Message` / `ContentBlock` 是**唯一**的会话词汇：投递（delivery）、持久历史（durable history）、模型请求（model request）三处共用同一份类型。

- 实例：官方表述 *"One immutable message representation shared by delivery, durable history, and model requests"*（[llm-streaming.md 第 54 行](../../deepseek-harness/docs/subsystems/llm-streaming.md)）。
- 为什么必须：三处各建一套消息类型 = 三份转换代码 + 三份漂移风险。

**R2 · 适配可插拔**——`LlmAdapter` 是适配器契约；任何提供商实现它即可接入，循环只依赖契约不依赖具体适配器。

- 实例：`llm-deepseek` 与 `llm-pi-ai` 是两个适配器包，配置换一行即切换（第 01 篇 R2 的实例之一）。
- 为什么必须：模型生态演进快，适配器必须能独立发布、独立升级。

**R3 · 元数据权威单点**——contextWindow、reasoningEffort 等"精确路由"元数据，由**服务该路由的适配器**解析，消费者不重复解析、不自行猜测。

- 实例：官方表述 *"Correctness-sensitive metadata is resolved separately from the advisory catalog and is owned by the adapter serving the exact route"*（[llm-streaming.md 第 407 行](../../deepseek-harness/docs/subsystems/llm-streaming.md)）。
- 为什么必须：模型能力（上下文窗口、思考档位）只有适配器最清楚；多份解析必然打架。

**R4 · 类型安全远程调用（typert）**——宿主能力通过**生成式类型契约**暴露给远程消费者：wire 上只传 endpoint + 命名参数，类型与 schema 由生成器保证。

- 实例：`TypertLookupMap` / `TypertContextMap` 通过声明合并扩展，Host 对象与 wire 身份的映射是**编译期声明**（[typert.md 第 9 行](../../deepseek-harness/docs/subsystems/typert.md)）。
- 为什么必须：手写 JSON-RPC 的调用方与实现方必然漂移；生成式契约让漂移在编译期暴露。

### 0.3 非功能需求

| 编号 | 约束 | 衡量方式 |
|---|---|---|
| N1 | **可重放**：assistant 消息携带适配器私有的 `replayState`，可无损重放提供商响应 | replay 与原始响应逐 token 一致 |
| N2 | **模态可声明**：模型能力（文本/图像等 inputModalities）由适配器声明，缺省表示未知、显式缺项表示不支持 | 路由选择不会把图像请求发给纯文本模型 |
| N3 | **advisory 与权威分离**：模型目录（catalog）是建议性的，不参与请求校验；权威解析只来自服务路由的适配器 | 目录与解析结果不一致时以解析为准 |
| N4 | **branded 标识**：跨边界的不透明 id（如 `ReasoningEffortId`）一律品牌化，禁止裸 string | 类型系统拒绝把 effort id 当普通字符串 |

### 0.4 验收标准

| 需求 | 验收示例（做到 = 通过） | 失败示例（做不到 = 没通过） |
|---|---|---|
| R1 | 同一 `Message` 类型贯穿投递、日志、请求三处，无转换层 | 三处各有一份消息结构 |
| R2 | 新提供商只实现 `LlmAdapter` + 注册，循环零改动 | 接新模型要改 agent-loop |
| R3 | 同一模型路由的 contextWindow 只有一个解析来源 | 多处解析出现不同值 |
| R4 | 远程调用参数类型错误在编译期报错 | 运行时才发现字段名写错 |

### 0.5 边界与不做什么

- **不做会话与循环**：那是 core 的职责；llm 只提供"词汇 + 适配契约 + 流"。
- **不做工具执行**：`tool-call` 只是词汇块，执行仍走 `ctx.tools` 守卫管线。
- **不做业务 RPC**：typert 是类型与传输机制；具体业务对象（会话、工具）由各自包声明 `TypertLookupMap` 扩展。
- **不做模型选择 UI**：那是 `client/` 的 `ui-model-selection`。

### 0.6 设计哲学（原则 → 引出的需求）

| 原则 | 内容 | 引出的需求 |
|---|---|---|
| P1 一份词汇，三处共用 | Message/ContentBlock 是会话语言的唯一载体 | R1 |
| P2 契约优于实现 | 循环依赖 LlmAdapter 契约，不依赖任何适配器包 | R2 |
| P3 权威在拥有者 | 精确路由元数据由服务它的适配器解析，advisory 数据不进校验 | R3 / N3 |
| P4 声明合并扩展 | 内容块、消息来源、typert 查找表全部用声明合并扩展 | R1/R4 |
| P5 品牌化边界 | 跨边界的 opaque id 全部 Branded | N4 |

### 0.7 备选技术路径

| 路径 | 思路 | 优势 | 代价 | 需求匹配 |
|---|---|---|---|---|
| A. 各层自建消息类型 | 投递/日志/请求各定义一套，层间转换 | 层内自由 | 三份类型三份转换，漂移不可避免 | R1 落空 |
| B. 硬编码提供商 | 循环直接 import DeepSeek SDK | 最少抽象 | 换模型改循环；多提供商无法并存 | R2 落空 |
| C. 适配器契约（**本项目**） | 统一词汇 + LlmAdapter + 权威解析 | 生态可扩展、可独立发布 | 契约要治理（JSDoc 全覆盖） | 满足 R1~R3 |
| D. 手写 JSON-RPC | 远程调用手写协议与校验 | 无生成器依赖 | 调用方与实现方必然漂移；无类型保障 | R4 落空 |
| E. 生成式类型契约（**本项目**） | 生成器产出契约，wire 只传 endpoint+args | 漂移在编译期暴露 | 生成器是新增基础设施 | 满足 R4 |

**选型结论**：会话语言（词汇）与传输语言（typert 契约）都选择"**一份权威类型，其余全派生**"——与第 02 篇的事件溯源是同一哲学在不同层的投影：真相只有一个，视图全部派生。

## 1 架构设计（WHAT）

### 1.1 总体架构：两条"语言"通道

```mermaid
flowchart LR
    subgraph 会话通道（llm）
      LOOP["core 循环<br/>llm/stream"] --> VOCAB["Message / ContentBlock<br/>统一词汇"]
      VOCAB --> AD1["LlmAdapter: llm-deepseek"]
      VOCAB --> AD2["LlmAdapter: llm-pi-ai"]
      VOCAB --> AD3["LlmAdapter: 用户自建"]
      AD1 --> API1["DeepSeek API"]
      AD2 --> API2["PI-AI API"]
    end
    subgraph 远程通道（typert）
      HOST["Host 端<br/>dsh-api-gateway"] --> GEN["生成器 generator<br/>契约产物"]
      GEN --> CON["消费者端<br/>GUI / SDK 客户端"]
      GEN --> WIRE["wire: endpoint + args"]
    end
```

**四步读懂**：

1. **会话通道**：core 循环只认识 `Message`/`StreamChunk` 词汇与 `LlmAdapter` 契约；具体适配器把词汇翻译成各家 API 的协议（R1/R2）；
2. **适配器自治**：每个适配器包拥有自己的"精确路由"元数据解析（contextWindow、reasoningEffort），消费者复用同一解析结果（R3）；
3. **远程通道**：typert 生成器从类型声明产出 Host 端与消费者端两套契约，wire 只传 endpoint + 命名参数（R4）；
4. **两通道交汇**：会话通道承载"人与模型的对话"，远程通道承载"程序与宿主的对话"——前者是产品体验，后者是生态接口。

### 1.2 关键架构决策（需求 → 方案 → 权衡）

| # | 决策 | 对应需求 | 权衡 |
|---|---|---|---|
| D1 | **合并可扩展的 ContentBlockMap**：新块类型经声明合并加入，但必须同时具备 adapter/UI/compaction/replay 支持 | R1 | 新模态门槛高；换来词汇不会半成品化 |
| D2 | **replayState 归适配器**：assistant 消息携带适配器私有重放数据，仅当适配器同时拥有历史与目标路由时暴露 | N1 | replay 数据格式适配器私有；换来无损重放 |
| D3 | **advisory 目录与权威解析分离**：模型目录是建议性的，不进请求校验 | R3 / N3 | 目录与解析可能不一致；换来解析永不打架 |
| D4 | **typert 生成式契约**：lookup/context 声明合并 + 生成器 + 网关 | R4 | 生成器成为基础设施；换来编译期类型保障 |

### 1.3 关系网

- **下游**：`core` 的循环消费 llm 缝（`llm/stream` 瀑布事件）；`tools` 注册表的 `ToolSchema` 与 llm 的请求词汇互操作；
- **上游**：`llm-deepseek` 等适配器依赖 `dsh-llm` 的契约；`token-meter` 消费流块做 token 计量（compaction 的输入之一）；
- **平级**：`typert` 的 registry/loader 供 `api/gateway` 使用；gateway 是远程通道的 Host 端出口（详见后续"应用层"篇）。

## 2 实现落点（HOW）

### 2.1 文件导航表（按阅读顺序）

| 顺序 | 文件（点击直达） | 关注点 | 对应需求 |
|---|---|---|---|
| 1 | [packages/llm/llm/src/types.ts](../../deepseek-harness/packages/llm/llm/src/types.ts) | `ContentBlockMap`、`Message`、`AssistantProvenance` | R1 / N1 |
| 2 | [packages/llm/llm/src/message.ts](../../deepseek-harness/packages/llm/llm/src/message.ts) | `MessageSourceMap`：消息来源声明合并 | R1 |
| 3 | [packages/llm/llm/src/index.ts](../../deepseek-harness/packages/llm/llm/src/index.ts) | `LlmAdapter` 契约与 `LlmRuntime` | R2 |
| 4 | [packages/llm/llm-deepseek/src](../../deepseek-harness/packages/llm/llm-deepseek/src) | DeepSeek 适配器实现（对照理解契约） | R2 |
| 5 | [packages/llm/llm-retry/src](../../deepseek-harness/packages/llm/llm-retry/src) | 重试策略插件（挂在请求上的守卫） | R2/R3 |
| 6 | [packages/typert/protocol/src/types.ts](../../deepseek-harness/packages/typert/protocol/src/types.ts) | `TypertLookupMap` / `TypertContextMap` / `TypertCodec` | R4 |
| 7 | [packages/api/gateway/src/types.ts](../../deepseek-harness/packages/api/gateway/src/types.ts) | Host 网关类型 | R4 |

### 2.2 关键实现片段

**片段 A：内容块词汇表**（[llm/src/types.ts](../../deepseek-harness/packages/llm/llm/src/types.ts)）

```ts
interface ContentBlockMap {
  'text': TextBlock
  'reasoning': ReasoningBlock
  'image': ImageBlock
  'tool-call': ToolCallBlock
  'tool-result': ToolResultBlock
}
```

翻译：会话里的"一句话"由**内容块数组**组成，块按 type 分五种：文本、思考（与可见文本区分）、图像、工具调用、工具结果。这是合并可扩展的映射——新模态（如音频）通过声明合并加入（R1），但官方注释明确：*"New core blocks must land with adapter, UI, and compaction support"*——新块不是"加个类型"这么简单（决策 D1）。

**片段 B：一条消息的完整定义**（[llm/src/types.ts](../../deepseek-harness/packages/llm/llm/src/types.ts)）

```ts
interface Message {
  readonly id: MessageId
  readonly role: 'system' | 'user' | 'assistant'
  readonly content: ContentBlock[]
  readonly source: MessageSource
}
```

翻译：`Message` 只有四个字段，但每一处都有讲究。`id` 跨所有表示边界保持稳定（投递、日志、请求里是同一个 id）；`role` 是提供商中立的三种角色；`content` 是精确的模型可见块；`source` 记录"这条消息从哪来"（用户/插件/模型/工具），本身也是合并可扩展的（`MessageSourceMap`）。官方称其为 *"One immutable message representation shared by delivery, durable history, and model requests"*——这就是 R1 的代码形态。

**片段 C：模型身份与重放数据**（[llm/src/types.ts](../../deepseek-harness/packages/llm/llm/src/types.ts)）

```ts
interface AssistantProvenance {
  provider: string
  model: string
  replayState?: unknown
}
```

翻译：模型产出的消息携带"出身"（provider + model），以及适配器私有重放数据 `replayState`（N1）。注意它标注：只有当目标适配器实例**同时拥有**这个历史 provider 与目标 provider 时，`LlmRuntime` 才把 replayState 暴露给它——重放数据不会泄漏给无关适配器。

### 2.3 符号 hover 指引

在 VS Code 打开 [packages/llm/llm/src/types.ts](../../deepseek-harness/packages/llm/llm/src/types.ts)，hover `Message`、`ContentBlockMap`、`AssistantProvenance` 查看官方 JSDoc；打开 [packages/typert/protocol/src/types.ts](../../deepseek-harness/packages/typert/protocol/src/types.ts) hover `TypertCodec` 查看两种编解码模式（strict / src-json）的语义。

## 3 产物演示（EXAMPLE）

### 3.1 输入

模型的适配器组合配置（[examples/headless-agent/cordis.yml 第 23~32 行](../../deepseek-harness/examples/headless-agent/cordis.yml)）：

```yaml
- id: llm-deepseek
  name: '@deepseek-ai/dsh-llm-deepseek'
  config:
    thinking: enabled
    reasoningEffort: max
    models:
      - id: deepseek-v4-pro
        contextWindow: 128000
      - id: deepseek-v4-flash
        contextWindow: 128000
```

### 3.2 产物（真实请求头事件）

> 以下为仓库现存快照 `examples/acp-agent/tests/snapshots/bash-tool-turn/session.jsonl` 的**逐字符拷贝**；行号与注释列为本文档添加。

<table style="border-collapse:collapse;width:100%;font-size:13px">
  <tr style="background:#f6f8fa">
    <th style="border:1px solid #d0d7de;padding:5px 8px;width:36px">行</th>
    <th style="border:1px solid #d0d7de;padding:5px 8px">真实产物（JSONL）</th>
    <th style="border:1px solid #d0d7de;padding:5px 8px">行内注释</th>
  </tr>
  <tr style="background:#fff8c5">
    <td style="border:1px solid #d0d7de;padding:5px 8px;text-align:center;color:#57606a">1</td>
    <td style="border:1px solid #d0d7de;padding:5px 8px;font-family:monospace">{"type":"request/header","seq":7,"time":1785498771361,"data":{"header":{"config":{"provider":"deepseek-official","model":"deepseek-v4-flash"},"system":"{{system}}","tools":"{{tools}}"},"reason":"initial"}}</td>
    <td style="border:1px solid #d0d7de;padding:5px 8px"><b>🎯 请求头</b>：provider=deepseek-official、model=deepseek-v4-flash（与输入配置对应）；system/tools 为快照归一化占位</td>
  </tr>
  <tr>
    <td style="border:1px solid #d0d7de;padding:5px 8px;text-align:center;color:#57606a">2</td>
    <td style="border:1px solid #d0d7de;padding:5px 8px;font-family:monospace">{"type":"request/context","seq":8,"time":1785730424636,"data":{"provider":"deepseek-official","model":"deepseek-v4-flash"}}</td>
    <td style="border:1px solid #d0d7de;padding:5px 8px">请求上下文：同一路由的权威解析结果（R3）</td>
  </tr>
  <tr>
    <td style="border:1px solid #d0d7de;padding:5px 8px;text-align:center;color:#57606a">3</td>
    <td style="border:1px solid #d0d7de;padding:5px 8px;font-family:monospace">{"type":"assistant/message","seq":99,…"source":{"kind":"model","provider":"deepseek-official","model":"deepseek-v4-flash"},…}</td>
    <td style="border:1px solid #d0d7de;padding:5px 8px">模型产出的消息携带 <b>AssistantProvenance</b>（provider+model，见片段 C）</td>
  </tr>
</table>

### 3.3 发生了什么（配置 → 请求 → 产出的四步）

1. 组合配置声明 `llm-deepseek` 适配器与两个模型（deepseek-v4-pro / deepseek-v4-flash，均 128000 上下文）；
2. 循环发起请求时，适配器把配置中的 `thinking: enabled`、`reasoningEffort: max` 与模型路由解析成请求头（产物行 1）；
3. 同一路由的权威元数据（contextWindow 等）解析结果记录在 `request/context`（产物行 2，R3 的落点）；
4. 模型产出的每条 assistant 消息都带 provider/model 出身（产物行 3），供派生历史与重放使用（N1）。

### 3.4 观察点（对应产物表中的行号）

- **行 1 的 `reason:"initial"`**：请求头的生成原因（initial/retry 等）被记录——可审计（呼应第 02 篇 N1）；
- **行 1 的 `"system":"{{system}}"`**：快照录制时对系统提示词做了归一化——快照可比对的前提；
- **行 2 与行 1 的 provider/model 完全一致**：请求上下文与请求头来自同一权威解析（R3 无漂移）。

## 4 动手验证（🧪）

> 以下命令已在本机实测（仓库根目录 `deepseek-harness\deepseek-harness` 下执行）。

### 任务 1：在真实会话里找"请求头"事件

```powershell
Select-String -Path "examples\acp-agent\tests\snapshots\bash-tool-turn\session.jsonl" -Pattern '"type":"request/header"'
```

**预期**：命中一行，含 `"provider":"deepseek-official"` 与 `"model":"deepseek-v4-flash"`。**判据**：把该行与 [examples/headless-agent/cordis.yml 第 23~32 行](../../deepseek-harness/examples/headless-agent/cordis.yml) 对照——模型 id 出现在配置的 `models` 列表里。

### 任务 2：读适配器契约

打开 [packages/llm/llm/src/index.ts](../../deepseek-harness/packages/llm/llm/src/index.ts)，找到 `LlmAdapter` 接口，数一数它声明了哪些能力（如 generate/stream、模型解析、能力声明）。然后打开 [packages/llm/llm-deepseek/src](../../deepseek-harness/packages/llm/llm-deepseek/src) 对照实现。

### 任务 3（进阶）：看 typert 的生成产物

```powershell
Get-ChildItem "packages\typert" -Directory | Select-Object Name
```

**预期**：看到 generator / loader / protocol / registry 四个包。**判据**：打开 [docs/subsystems/typert.md](../../deepseek-harness/docs/subsystems/typert.md) 第 5 行确认——"Types shared by generated Remote artifacts, the Host Gateway, and consumer API assemblies"，生成器是契约的唯一来源。

## 5 FAQ 与自测（❓）

### FAQ

- **Q1：llm 组和 core 组什么关系？** core 是消费者：循环经 `llm/stream` 瀑布调用适配器；llm 组是词汇与契约的**拥有者**（R1）。官方分工：*"The core packages hold and log these values on every turn; this page declares them."*
- **Q2：为什么 Message 要三处共用？** 投递、日志、请求三处各建一份类型，就要三份转换代码、三处漂移点（0.7 路径 A）；共用一份让"模型可见 ⟺ 已记录"（第 02 篇 P1）在类型层面直接成立。
- **Q3：advisory 目录和权威解析什么区别？** 模型目录（`LlmModelInfo`）是**建议性**的（供选择器展示）；请求真正依赖的 contextWindow、默认档位由服务该路由的适配器**权威解析**（`LlmResolvedModelInfo`），目录不参与校验（N3）。
- **Q4：typert 和普通 JSON-RPC 什么区别？** 普通 RPC 手写协议与校验；typert 由生成器从类型声明产出两套契约，wire 只传 endpoint + args，参数类型错误在编译期暴露（R4）。
- **Q5：我想接一个新模型提供商，要动哪些地方？** 实现 `LlmAdapter`（词汇翻译 + 路由解析），注册到 `ctx.llm`，配置里声明路由——循环、日志、UI 零改动（R2）。

### 自测（答案折叠在下方）

1. `ContentBlockMap` 是哪种扩展模式？新增块类型的前提条件是什么？
2. `AssistantProvenance` 里的 `replayState` 在什么条件下才会暴露给适配器？
3. 产物表里哪个字段证明"请求头来自配置"？
4. typert 的 wire 上只传什么？
5. 下列哪个不属于 llm 组的职责？A. Message 词汇 B. LlmAdapter 契约 C. 工具执行 D. 精确路由解析

<details>
<summary>点开看答案</summary>

1. 合并可扩展（声明合并）。新块必须同时具备 adapter、UI、compaction、replay 四方面支持（决策 D1）。
2. 只有当目标适配器实例同时拥有该历史 provider 与目标 provider 时（防止重放数据泄漏）。
3. 产物行 1 的 `"model":"deepseek-v4-flash"` 与配置 `models` 列表中的 id 对应。
4. endpoint + 命名参数（args）；类型与 schema 由生成器保证。
5. C。工具执行属于 core 的 `ctx.tools` 守卫管线。

</details>

## 6 延伸阅读（📚）

**官方（权威来源）**：

- [docs/subsystems/llm-streaming.md](../../deepseek-harness/docs/subsystems/llm-streaming.md) —— 消息词汇、流协议、适配器契约全文（888 行）
- [docs/subsystems/typert.md](../../deepseek-harness/docs/subsystems/typert.md) —— typert 公开契约（lookup/context 声明、codec）
- [packages/llm/README.md](../../deepseek-harness/packages/llm/README.md) —— llm 组包清单与扩展点
- [packages/typert/README.md](../../deepseek-harness/packages/typert/README.md) —— typert 组包清单

**决策记录（WHY 的一手来源）**：

- 🔴 [.agents/notes/implemented/architecture/2026-08-02-typert-remote-method-calls.md](../../deepseek-harness/.agents/notes/implemented/architecture/2026-08-02-typert-remote-method-calls.md) —— typert 架构与传输决策

**外部文献（按难度递增）**：

- 🟢 [Anthropic: Function calling 文档](https://docs.anthropic.com/en/docs/build-with-claude/tool-use) —— 工具调用词汇的行业参照
- 🟡 [DeepSeek API 文档](https://api-docs.deepseek.com/) —— 适配器 llm-deepseek 对接的官方协议
- 🔴 [TypeScript Declaration Merging](https://www.typescriptlang.org/docs/handbook/declaration-merging.html) —— 本项目"合并可扩展"机制的底层语言能力

---

**下一篇预告**：【第 05 篇】能力族全景：fs / subprocess / sandbox ——"执行世界"的家族图谱。
