# 【第 12 篇】工程体系：scripts / docs / .github / .agents / website / examples / native

> 难度：🟢 入门（本篇是"运维视角"总览，无深代码）
> 前置阅读：`../项目架构总览.md`
> 对应目录：`deepseek-harness/scripts/`、`docs/`、`.github/`、`.agents/`、`website/`、`examples/`、`native/`

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

产品代码（前十一篇）之外，仓库还有一整套"让产品活下来"的工程体系。四个场景：

1. **质量门禁**：每次提交/CI 都要验证——类型、lint、单测、覆盖率、快照、文档保鲜；
2. **文档即产品**：docs/ 有严格层级与字数预算，大部分目录表**从源码生成**、CI 保鲜；
3. **Agent 原生开发**：`.agents/` 存开发 skill 与 1087 篇决策记录——AI 编码代理的"组织记忆"；
4. **可运行样板**：`examples/` 的 6 个组合既是教程又是快照测试的真实载体。

### 0.2 需求陈述

**R1 · 门禁可执行（scripts）**——质量约束写成**可运行脚本**：`run-gates.ts` 编排全部检查；`gen-*.ts` 生成目录；`verify-*.ts` 校验保鲜；`hygiene`/`doc-sync` 是聚合闸门。

- 实例：根 package.json 的 `check:all` = `tsx scripts/run-gates.ts check-all`；`doc-sync` 聚合文档全部闸门（[package.json 第 48/127 行](../deepseek-harness/package.json)）。
- 为什么必须：规则写进文档会漂移；写进脚本才会被执行。

**R2 · 文档分层预算（docs）**——文档按层级分工（architecture → subsystems → cookbook → postmortem），每份有字数预算（`doc-budgets.manifest.json`），"一个事实只有一个家"。

- 实例：官方原文 *"Each fact has one home: the tier whose job it is; elsewhere, link there"*（[docs/AGENTS.md 第 14 行](../deepseek-harness/docs/AGENTS.md)）。
- 为什么必须：文档无限膨胀 = 没人读；预算与分层让文档保持可用。

**R3 · 生成物保鲜**——config-catalog / tool-catalog / module-graph / persistence-catalog / Cordis API 面等**全部从源码生成**，CI 校验新鲜度。

- 实例：`verify-tool-catalog` = `tsx scripts/gen-tool-catalog.ts --check`（[package.json 第 113 行](../deepseek-harness/package.json)）。
- 为什么必须：手写目录必然与源码漂移；生成 + 保鲜 = 目录永远可信。

**R4 · 代理有组织记忆（.agents）**——开发 skill（11 个）与决策记录（1087 篇 Agent Notes，按 implemented/archived/proposed/rejected 归档）供编码代理遵循。

- 实例：`dsh-pre-push-checks`（推送前选择最窄检查）、`dsh-code-review`（代码评审标准）等 skill；笔记归档策略（archived 冻结、implemented 以现在时描述现实）。
- 为什么必须：大型仓库的隐性知识必须显式化——代理（和新人）才有立足点。

**R5 · 样板即测试（examples）**——6 个可运行组合（headless/acp/jsonrpc/mcp/web）既示范用法，又是 keyless 快照测试的真实载体。

- 实例：官方原文 *"Each example has both: Keyless — boot the real cordis.yml through the Loader, drive it, and assert output and clean exit"*（[examples/AGENTS.md 第 10~12 行](../deepseek-harness/examples/AGENTS.md)）。
- 为什么必须：快照测试需要"真实可运行的样板"——示例与测试是一体的。

### 0.3 非功能需求

| 编号 | 约束 | 衡量方式 |
|---|---|---|
| N1 | **最窄检查**：推送前按改动面选最窄检查，不默认全量 | 提交/推送延迟可控 |
| N2 | **覆盖率门限**：packages/*/*/src 每文件 100% 覆盖（client 源码在内） | `test:coverage` 是 CI 闸门 |
| N3 | **文档预算**：超预算/缺文件被 `verify-doc-budgets` 拒绝 | 预算 manifest 权威 |
| N4 | **双语配对**：文档英中成对（i18n），翻译按配对工作流更新 | 中文不落后于英文 |
| N5 | **快照可比对**：模型输出快照可回放（keyless replay） | `test:snapshot` 无 key 可跑 |

### 0.4 验收标准

| 需求 | 验收示例（做到 = 通过） | 失败示例（做不到 = 没通过） |
|---|---|---|
| R1 | 改了 packages 某包 → 最窄检查（单测+typecheck）绿 | 必须全量跑才知道对错 |
| R2 | 新文档超过预算 → verify-doc-budgets 拒绝 | 文档无限膨胀 |
| R3 | 工具改了 schema → verify-tool-catalog 报"目录过期" | 目录与源码漂移 |
| R4 | 非平凡改动缺 Agent Note → 审查被拒 | 决策理由无处可查 |
| R5 | 新行为有对应快照样例（keyless 可跑） | 只有 mock 单测 |

### 0.5 边界与不做什么

- **不做产品功能**：工程体系服务"开发与发布"，不参与运行时（native/landlock-run 例外，它是产品沙箱的原生模块）。
- **不做在线文档托管**：website 是 VitePress 构建，托管在 CI（docs-pages.yml）。
- **CI 拥有全量覆盖**：本地只复演最窄面（[AGENTS.md 第 7 行](../deepseek-harness/AGENTS.md)）。

### 0.6 设计哲学（原则 → 引出的需求）

| 原则 | 内容 | 引出的需求 |
|---|---|---|
| P1 规则即脚本 | 质量约束写成可执行闸门，不写在文档里 | R1 |
| P2 生成即保鲜 | 目录从源码生成，CI 校验新鲜度 | R3 |
| P3 一事实一家 | 文档分层 + 预算，"one home per fact" | R2 |
| P4 样板即测试 | examples 是教程也是快照载体 | R5 |
| P5 代理有记忆 | skill + notes 是仓库的组织记忆 | R4 |

### 0.7 备选技术路径

| 路径 | 思路 | 优势 | 代价 | 需求匹配 |
|---|---|---|---|---|
| A. 无门禁 | 靠 code review 人工把关 | 零工具 | 人为遗漏；无可重复验证 | R1 落空 |
| B. 手写目录 | 文档目录手工维护 | 无生成器 | 必然漂移 | R3 落空 |
| C. 生成 + 保鲜（**本项目**） | gen-*.ts + verify-*.ts + doc-sync | 目录永远可信 | 生成器是维护成本 | 满足 R3 |
| D. 全量 CI | 每次推送全量检查 | 无选择成本 | 延迟高、反馈慢 | N1 落空 |
| E. 最窄面 + CI 全量（**本项目**） | 本地最窄、CI 全量 | 反馈快 | 选择检查要经验 | 满足 N1 |

**选型结论**：工程体系是"产品的产品"——**规则即脚本（P1）**让约束可执行，**生成即保鲜（P2）**让文档可信，**样板即测试（P4）**让示例不腐烂。

## 1 架构设计（WHAT）

### 1.1 总体架构：七块拼图围绕产品代码

```mermaid
flowchart LR
    SRC["产品代码 packages/ apps/（前十一篇）"]
    G["scripts/<br/>门禁与生成器"]
    D["docs/<br/>分层文档 + 预算"]
    GH[".github/<br/>15 个 CI 流水线"]
    A[".agents/<br/>11 skills + 1087 notes"]
    W["website/<br/>VitePress 双语站"]
    E["examples/<br/>6 个可运行样板"]
    N["native/<br/>landlock-run 原生模块"]
    G --> SRC
    D --> SRC
    GH --> G
    GH --> D
    W --> D
    E --> SRC
    N --> SRC
    A -.指导代理.-> SRC
```

**四步读懂**：

1. **scripts** 是"裁判"：gen-*.ts 从源码生成目录、verify-*.ts 保鲜、run-gates.ts 编排；
2. **docs** 是"说明书"：分层 + 预算 + 生成区（`cordis-surface` 等），双语配对；
3. **.github** 是"警察局"：15 个 workflow 执行门禁、发布、文档站部署、issue 机器人；
4. **.agents/website/examples/native** 是"辅助设施"：代理记忆、文档站、样板测试、原生沙箱模块。

### 1.2 关键架构决策（需求 → 方案 → 权衡）

| # | 决策 | 对应需求 | 权衡 |
|---|---|---|---|
| D1 | **run-gates 编排**：`check:ci:*` 系列细分闸门面（静态/lint/覆盖/快照/产物） | R1 | 闸门面多；换来 CI 失败可定位 |
| D2 | **生成区标记**：`<!-- BEGIN GENERATED … -->` 包裹，禁手改 | R3 | 生成器要覆盖全；换来目录可信 |
| D3 | **预算 manifest**：doc-budgets.manifest.json 设限，超限/缺文件拒绝 | R2 | 提预算要走 manifest diff；换来文档克制 |
| D4 | **快照录制回放**：DSH_SNAPSHOT=record/refresh/replay 三态 | R5 / N5 | 快照要维护；换来 keyless 测试 |
| D5 | **native 独立发布**：landlock-run 有独立 workflow 与 release | —— | 多一套发布；换来沙箱模块可独立迭代 |

### 1.3 关系网

- **上游**：scripts 消费产品源码（生成器读类型声明）；CI workflow 调用 scripts；website 消费 docs；
- **下游**：examples 声明 packages/ 的包依赖（组合行）；native 被 sandbox-local 消费（第 05 篇）；
- **守护**：`verify-*` 系列是"文档↔源码"一致性的闭环（R3）。

## 2 实现落点（HOW）

### 2.1 文件导航表（按阅读顺序）

| 顺序 | 文件（点击直达） | 关注点 | 对应需求 |
|---|---|---|---|
| 1 | [scripts/run-gates.ts](../deepseek-harness/scripts/run-gates.ts) | CI 闸门编排入口 | R1 |
| 2 | [scripts/gen-tool-catalog.ts](../deepseek-harness/scripts/gen-tool-catalog.ts) | 工具目录生成器（--check 保鲜） | R3 |
| 3 | [scripts/doc-budgets.manifest.json](../deepseek-harness/scripts/doc-budgets.manifest.json) | 文档字数预算 | R2 |
| 4 | [.github/workflows/ci.yml](../deepseek-harness/.github/workflows/ci.yml) | 主 CI 流水线 | R1 |
| 5 | [.agents/skills/dsh-pre-push-checks/SKILL.md](../deepseek-harness/.agents/skills/dsh-pre-push-checks/SKILL.md) | 推送前检查 skill | R4 / N1 |
| 6 | [.agents/notes/README.md](../deepseek-harness/.agents/notes/README.md) | 决策记录规范（何时写、如何归档） | R4 |
| 7 | [website/.vitepress/config.ts](../deepseek-harness/website/.vitepress/config.ts) | 文档站配置 | —— |
| 8 | [native/landlock-run/README.md](../deepseek-harness/native/landlock-run/README.md) | 原生模块说明 | —— |

### 2.2 关键实现片段

**片段 A：一条闸门命令**（[package.json 第 127 行](../deepseek-harness/package.json)）

```json
"doc-sync": "tsx scripts/run-gates.ts doc-sync"
```

翻译：一条命令 = 文档全部闸门（链接、格式、预算、生成物保鲜、翻译配对）。这就是"规则即脚本"（P1）的形态——所有文档纪律收敛成可执行的单一入口。

**片段 B：生成区的禁手改标记**（[docs/persistence-catalog.md 第 1~2 行](../deepseek-harness/docs/persistence-catalog.md)）

```html
<!-- Generated by scripts/gen-persistence-catalog.ts — do not edit by hand.
     Run `pnpm run gen-persistence-catalog` to regenerate. -->
```

翻译：生成文档的开头自带"禁手改"标记与再生成命令——生成物保鲜（R3）的物理约定。同类文件还有 tool-catalog、config-catalog、module-graph、capability-seams 等（全部 `gen-*.ts` 产出）。

**片段 C：笔记的四类归档**（[.agents/notes/README.md](../deepseek-harness/.agents/notes/README.md)）

> implemented/ 描述已交付的现实（现在时）；archived/ 冻结为历史，永不作为当前权威；proposed/ 与 rejected/ 记录候选与放弃。

翻译：决策记录（Agent Notes）是仓库的"组织记忆"（R4）——implemented 以现在时描述已交付现实、archived 冻结为历史、proposed/rejected 记录候选与放弃。非平凡改动必须带笔记（第 01 篇 AGENTS.md 纪律），决策理由因此永远可查。

### 2.3 符号 hover 指引

在 VS Code 打开 [scripts/run-gates.ts](../deepseek-harness/scripts/run-gates.ts) 查看闸门清单；打开 [.agents/skills/dsh-pre-push-checks/SKILL.md](../deepseek-harness/.agents/skills/dsh-pre-push-checks/SKILL.md) 查看推送前检查选择表。

## 3 产物演示（EXAMPLE）

### 3.1 输入

一行命令（**已在本机实测**，仓库根目录执行）：

```powershell
pnpm dsh --profile headless --dump-config | Select-Object -First 5
```

（演示"工程产物可审计"——但本篇的 EXAMPLE 用更贴切的产物：生成的目录文件。）

### 3.2 产物（真实生成目录）

> 以下为仓库现存生成文件的结构与开头（`docs/tool-catalog.md` 与 `docs/capability-seams.md`）。

| 生成文件 | 生成器 | 开头标记 | 保鲜闸门 |
|---|---|---|---|
| `docs/tool-catalog.md`（1873 行） | `scripts/gen-tool-catalog.ts` | `<!-- Generated by scripts/gen-tool-catalog.ts … -->` | `verify-tool-catalog` |
| `docs/capability-seams.md`（471 行） | `scripts/gen-doc-graphs.ts` | 同上 | `verify-doc-graphs` |
| `docs/persistence-catalog.md` | `scripts/gen-persistence-catalog.ts` | 同上 | `verify-persistence-catalog` |
| `docs/config-catalog.md` | `scripts/gen-config-catalog.ts` | 同上 | `verify-config-catalog` |

`capability-seams.md` 开头（[第 1~2 行](../deepseek-harness/docs/capability-seams.md)）：

```html
<!-- Generated by scripts/gen-doc-graphs.ts - do not edit by hand.
     Run `pnpm run gen-doc-graphs` to regenerate. -->
```

### 3.3 发生了什么（源码 → 目录 → 保鲜的三步）

1. 开发者在包源码里改 schema/服务声明（如第 03 篇的 `ctx.shell`）；
2. `pnpm run gen-tool-catalog` 从声明重新生成目录（含依赖边、事件、说明）；
3. CI 的 `verify-tool-catalog`（`--check` 模式）发现目录与源码不一致 → 报错——**漂移在 CI 暴露**（R3）。

### 3.4 观察点（对应产物文件）

- **每个生成文件开头都是"禁手改"标记**：生成物保鲜的物理约定（R3）；
- **tool-catalog 1873 行**：全工具的可信目录——第 03 篇 bash 条目、第 07 篇 subagent 条目都引自它；
- **capability-seams 471 行**：第 03/05 篇引用的缝表与图的来源——"关系网"类论述的权威出处。

## 4 动手验证（🧪）

> 以下命令已在本机实测（仓库根目录 `deepseek-harness\deepseek-harness` 下执行）。

### 任务 1：数生成器

```powershell
Get-ChildItem "scripts" -Filter "gen-*.ts" | Select-Object Name
```

**预期**：约 15 个 gen-*.ts。**判据**：对照 [package.json 第 104~125 行](../deepseek-harness/package.json) 的 `gen-*`/`verify-*` 脚本对。

### 任务 2：验证一条保鲜闸门

```powershell
pnpm exec vitest run scripts/tool-catalog.spec.ts 2>$null | Select-Object -Last 3
```

**预期**：测试通过（目录与源码一致）。**判据**：`✓` 结果——你的本地仓库是"新鲜"的。

### 任务 3（进阶）：看发布流水线

```powershell
Get-ChildItem "scripts\release" -File | Select-Object Name
```

**预期**：bump/pack/publish/verify 等文件。**判据**：对照根 package.json 的 `release:*` 脚本族（第 130~135 行）——发布也是脚本化的（P1）。

## 5 FAQ 与自测（❓）

### FAQ

- **Q1：为什么文档预算这么严格？** 因为文档无限膨胀 = 没人读、无法维护。"one home per fact" + 预算 = 文档保持可用（R2）。
- **Q2：生成目录坏了怎么办？** 跑 `pnpm run gen-xxx` 重新生成（--check 是校验模式）；手改会被 CI 拒绝（R3）。
- **Q3：1087 篇笔记谁写的？** 仓库的开发代理与贡献者按规范写的——非平凡改动必须带笔记，所以决策理由永远可查（R4）。
- **Q4：本地要跑全量测试吗？** 不用。按改动面选最窄检查（dsh-pre-push-checks skill）；CI 拥有全量矩阵（N1）。
- **Q5：examples 是教程还是测试？** 两者一体：样板组合示范用法，同时是 keyless 快照测试的真实载体（R5/P4）。

### 自测（答案折叠在下方）

1. `doc-sync` 是什么？（一条命令 vs 一堆脚本）
2. 生成文件开头为什么有"禁手改"标记？
3. Agent Notes 的 implemented/ 用什么时态？archived/ 代表什么？
4. 本地推送前应该跑什么？（最窄 vs 全量）
5. 下列哪个不属于本组职责？A. 工具目录生成 B. 会话日志 C. 发布脚本 D. 文档预算

<details>
<summary>点开看答案</summary>

1. 一条命令 = 文档全部闸门（run-gates doc-sync），规则即脚本（P1）。
2. 生成物保鲜的物理约定：手改会被保鲜校验拒绝（R3）。
3. implemented/ 以现在时描述已交付现实；archived/ 冻结为历史，永不作为当前权威。
4. 按改动面选最窄检查（dsh-pre-push-checks）；CI 拥有全量矩阵。
5. B。会话日志属于 core/session（第 02 篇）。

</details>

## 6 延伸阅读（📚）

**官方（权威来源）**：

- [docs/AGENTS.md](../deepseek-harness/docs/AGENTS.md) —— 文档层级与预算规范
- [docs/development.md](../deepseek-harness/docs/development.md) —— 开发者日常流程与 CI 摘要
- [docs/testing.md](../deepseek-harness/docs/testing.md) —— 测试政策（覆盖门限、快照、子进程启动）
- [examples/AGENTS.md](../deepseek-harness/examples/AGENTS.md) —— 样板即测试的规范
- [.agents/notes/README.md](../deepseek-harness/.agents/notes/README.md) —— 决策记录规范

**外部文献（按难度递增）**：

- 🟢 [Google's Engineering Practices](https://google.github.io/eng-practices/) —— 工程纪律的行业参照
- 🟡 [Conventional Commits](https://www.conventionalcommits.org/) —— 提交规范（发布流水线的输入）
- 🔴 [The Architecture of Open Source Applications（aosabook）](https://aosabook.org/en/) —— "工程体系与产品代码同等重要"的经典案例集

---

**系列正文完结**：第 01~12 篇覆盖 vendor → core → 能力族 → 记忆 → 多智能体 → 治理 → 应用层 → 协议 → 工程体系。下一步：整体审查（对照仓库原始文档核查出处与前后一致性）。
