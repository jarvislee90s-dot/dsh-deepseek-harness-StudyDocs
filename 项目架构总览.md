# DeepSeek Harness 项目架构总览

> 生成日期：2026-xx-xx ｜ 分析对象：`E:\LLMproject\deepseek-harness\deepseek-harness`（v0.1.0-rc.5）
> 说明：本文档为项目学习的"总览层"，后续按文件树逐模块下钻的详解文档将从第二部分开始连载。

## 项目身份卡片

| 项 | 内容 |
|---|---|
| 名称 | DeepSeek Harness（`dsh`） |
| 出品方 | DeepSeek AI |
| 定位 | 开源 Agent 运行时框架（harness） |
| 核心理念 | **一切皆插件**：模型适配器、工具注册表、会话日志、Agent 主循环都是平级插件，可从配置层面整体替换 |
| 底层框架 | 内置（vendored）的 Cordis 插件框架 |
| 版本 / 许可 | v0.1.0-rc.5（developer preview）/ MIT |
| 技术栈 | TypeScript（strict，全仓 ESM）、pnpm workspace（pnpm@11.7.0）、Node ^22.19 \|\| >=24、Vite + React（前端）、Python（SDK）、Cordis |
| 规模 | 49 个包分组 / 219 个 npm 包（`@deepseek-ai/dsh-*`） |

> 说明：以下文件树已排除 `node_modules/`、`.git/`、`lib/`（构建产物）、`dist/` 等噪声目录；`packages/` 下每个包的标准布局为 `src/ + tests/ + package.json + tsconfig.json + tsdown.config.ts + README.md`，不再逐包展开。

## 一、总览文件树

```
E:\LLMproject\deepseek-harness\
├── deepseek-harness\                        # ★ 项目本体（pnpm monorepo）
│   ├── apps\                                # 应用层（2 个应用）
│   │   ├── cli\                             #   dsh 命令行入口（bin.ts/args.ts/plugin.ts…）
│   │   │   ├── config\agent-presets\        #     内置 agent 预设（code/cordis/minimal/standard）
│   │   │   └── reference\                   #     命令参考文档
│   │   └── web\                             #   Web GUI 前端（Vite + React，main.ts 注入 __DSH_BOOT__）
│   ├── packages\                            # ★ 核心库：49 个分组 / 219 个包
│   │   ├── core\                            #   产品 API 主干：agent, agent-loop, session, system-prompt,
│   │   │                                   #     tools, scope, agent-default-model, agent-tool-presentation
│   │   ├── api\                             #   远程 BFF 组装 + Typert RPC 网关（gateway, remotes）
│   │   ├── typert\                          #   类型图生成/加载/运行时注册表（generator, loader, protocol, registry）
│   │   ├── llm\                             #   LLM 能力族：llm, llm-deepseek, llm-pi-ai, llm-retry, token-meter
│   │   ├── host\                            #   Web-GUI 服务端半区（webserver, apiproxy, frontend-static…）
│   │   ├── client\                          #   Web-GUI 浏览器半区（39 个 ui-* 插件、web, web-react, hmr…）
│   │   ├── session\                         #   会话持久化数据面（persistence-jsonl/sqlite, projection…）
│   │   ├── session-query\                   #   会话检索（log-export, query-sqlite, tool-session-query）
│   │   ├── sdk\                             #   进程外运行时 SDK（protocol, server, client）
│   │   ├── acp\                             #   Agent Client Protocol 自动化服务端
│   │   ├── subagent\                        #   子代理能力族（11 个包：dsh-sdk/claude-code/codex/acp 提供者…）
│   │   ├── workflow\ + jobs\                #   工作流引擎 / 后台任务运行时
│   │   ├── shell\ subprocess\ terminal\     #   Bash / 子进程 / 持久 PTY 终端能力族
│   │   ├── code-runtime\ sandbox\ e2b\      #   代码执行 / 进程沙箱（bwrap/Landlock/Seatbelt）/ E2B POC
│   │   ├── fs\ lsp\ skill\ web\             #   文件系统 / 语言服务器 / Skill / Web 能力族
│   │   ├── interaction\                     #   人机协作：approval, permission-presets, commands, ask-user
│   │   ├── guard\ compaction\ context\      #   循环卫生 / 上下文压缩 / 请求上下文
│   │   ├── goal\ plan\ preset\ todo\        #   目标 / 计划模式 / 会话预设 / todo 工具
│   │   ├── bundle\                          #   可安装 profile 层：base, web-app, headless
│   │   ├── boot\ extensions\ hooks\         #   启动胶水 / 自修改 / Claude Code & Codex hook 桥
│   │   ├── identity\ settings\ credentials\ storage\ workspace\  # 身份/设置/凭据/存储/工作区
│   │   ├── attachment\ spill\ feedback\ schedule\ mcp\           # 附件/溢出/反馈/定时/MCP
│   │   ├── examples\ test-support\ util\ runtime-diagnostics\    # 演示包 / 测试设施 / 零依赖工具 / 运行时不变量
│   │   └── README.md / AGENTS.md            #   分组说明 + 包级规则
│   ├── vendor\                              # ★ 内置的 Cordis 框架源码（9 个包，重命名为 @deepseek-ai/ 作用域）
│   │   └── cordis, loader, include, group, timer, hmr, schemastery, cosmokit, logger-console
│   ├── python\                              # Python 侧支持
│   │   ├── sdk\                             #   Python SDK（client.py / api.py / models.py）
│   │   └── sdk-runtime\                     #   内置 Python 运行时（cordis.yml 代理插件）
│   ├── native\                              # 原生模块：landlock-run（Linux Landlock 沙箱，node addon）
│   ├── examples\                            # 可运行示例（6 个）
│   │   ├── headless-agent\  acp-agent\      #   无头一次性任务 / ACP 自动化服务（含大量 .cordis.snapshot.yml）
│   │   ├── jsonrpc-agent\                   #   JSON-RPC 最小示例（含 minimal.py）
│   │   ├── mcp-memory\                      #   MCP 记忆服务器组合示例
│   │   └── web-cordis\  web-schedule\       #   Web GUI 组合 / 定时任务示例
│   ├── docs\                                # ★ 文档体系（约 60+ 篇，英中双语）
│   │   ├── architecture.md                  #   架构地图（必读）
│   │   ├── subsystems\                      #   约 50 页子系统参考（每页含生成的 Cordis API 表）
│   │   ├── cookbook\                        #   操作指南（加包/加工具/加 LLM 适配器/加对话节点…）
│   │   ├── cordis-tutorial\                 #   Cordis 7 章教程
│   │   ├── cordis-api\                      #   Cordis 核心 API 参考
│   │   ├── postmortem\                      #   4 篇事故复盘
│   │   ├── user\ i18n\                      #   用户指南 / 翻译规范
│   │   └── config-catalog.md  tool-catalog.md  module-graph.md 等  # 生成式目录（CI 保鲜检查）
│   ├── scripts\                             # ★ 仓库闸门与生成器（~100 个脚本）
│   │   ├── run-gates.ts                     #   CI 闸门编排入口
│   │   ├── gen-*.ts  verify-*.ts            #   目录/文档生成器 + 校验器（doc-sync、hygiene）
│   │   ├── release\                         #   发布流水线（bump/pack/publish/verify）
│   │   └── fixtures\                        #   测试夹具
│   ├── website\                             # VitePress 双语文档站（.vitepress/config.ts）
│   ├── .agents\                             # ★ Agent 工作区
│   │   ├── skills\                          #   11 个开发工作流 skill（code-review, pre-push-checks…）
│   │   └── notes\                           #   1087 篇决策记录（implemented/archived/proposed/rejected）
│   ├── .claude\skills                       # 符号链接 → ../.agents/skills（Claude Code 复用）
│   ├── .github\                             # CI/CD：15 个 workflow + issue 管理机器人
│   ├── patches\                             # node-pty@1.1.0.patch（pnpm patch）
│   ├── assets\                              # 社区宣传图
│   └── 根配置文件                            # package.json, pnpm-workspace.yaml, tsconfig*.json,
│                                          #   vitest*.config.ts, tsdown.config.ts, lefthook.yml,
│                                          #   knip.json, .oxlintrc*.json, .jscpd.json, AGENTS.md…
└── Research_doc\                            # （空）研究文档目录，本文档所在位置
```

## 二、文件简洁对照表

### 2.1 根级关键文件

| 文件 | 功能 |
|---|---|
| `package.json` | monorepo 根：workspaces 声明、全部脚本（build/test/lint/doc-sync/hygiene/release） |
| `pnpm-workspace.yaml` | 工作区映射（vendor、packages、apps、website、native） |
| `tsconfig.json` / `tsconfig.base.json` | TypeScript 项目引用根 + 公共编译基线（strict） |
| `tsconfig.host.json` / `tsconfig.client.json` | 服务端（host）与浏览器（client）两个编译面 |
| `tsdown.config.ts` | tsdown 打包配置（ESM 运行时产物） |
| `vitest*.config.ts`（7 个） | 单测 / e2e / 快照 / Web / Web 压力 / Web 性能测试配置 |
| `lefthook.yml` | git 钩子（提交前检查） |
| `knip.json` / `.oxlintrc*.json` / `.jscpd.json` | 未用代码检测 / oxlint 规则 / 重复代码检测 |
| `.gitlab-ci.yml` + `.github/workflows/` | 备用 GitLab CI + 主 GitHub Actions（15 个流水线） |
| `AGENTS.md`（= CLAUDE.md） | 仓库级 Agent 行为规范（约定、防御模式、测试政策） |
| `README.md` / `CONTRIBUTING.md` / `BENCHMARK.md` | 项目说明 / 贡献指南 / 基准测试 |
| `patches/node-pty@1.1.0.patch` | pnpm 对 node-pty 的补丁 |
| `pytest.ini` | Python 侧测试配置 |

### 2.2 顶层目录对照

| 目录 | 角色 | 说明 |
|---|---|---|
| `apps/` | 应用层 | `cli`：`dsh` 命令启动器（profile 引导）；`web`：Vite+React 前端 |
| `packages/` | 核心库 | 219 个包按 49 个分组组织，见 2.3 |
| `vendor/` | 框架层 | Cordis 全家桶源码快照（固定 commit、可审计、可打补丁） |
| `python/` | Python 生态 | `sdk`：官方 Python 客户端；`sdk-runtime`：内置 Python 执行运行时 |
| `native/` | 原生层 | Landlock 沙箱 node addon（Linux 进程隔离） |
| `examples/` | 示例层 | 6 个可运行组合（headless/acp/jsonrpc/mcp/web），同时是快照测试的载体 |
| `docs/` | 文档层 | 架构、子系统、教程、事故复盘、生成目录 |
| `scripts/` | 工具层 | CI 闸门、文档生成器、发布流水线（约 100 个脚本） |
| `website/` | 文档站 | VitePress 双语站点 |
| `.agents/` | Agent 协作 | 11 个 skill + 1087 篇架构决策记录（Agent Notes） |
| `.github/` | 工程化 | CI/CD、issue 自动分类、PR 模板、dependabot |
| `assets/` | 素材 | 社区宣传图片 |

### 2.3 packages/ 分组对照（49 组，含代表性包）

| 分组 | 功能 |
|---|---|
| `core` | **产品 API 主干**：`agent`（Agent 接口与注册表）、`agent-loop`（默认驱动循环）、`session`（追加式会话事件日志）、`system-prompt`（提示词组装）、`tools`（作用域工具注册表+执行管线）、`scope`（按 agent 作用域注册） |
| `api` | 远程 BFF 组装 + Typert RPC 网关（`gateway`, `remotes`） |
| `typert` | 类型图生成器/加载器/运行时注册表（模型工具 schema 的类型系统） |
| `llm` | LLM 能力族：抽象服务 + DeepSeek/PI-AI 提供者 + 重试 + token 计量 |
| `host` / `client` | Web GUI 服务端半区（webserver、apiproxy、目录选择器、前端静态服务）/ 浏览器半区（39 个 `ui-*` 界面插件、连接、HMR、locale） |
| `session` | 会话持久化数据面：JSONL/SQLite 后端、投影（projection）、标题生成、遥测（OTel） |
| `session-query` | 会话检索：逻辑语料、有界读取、血缘、SQLite 全文搜索、日志导出 |
| `sdk` / `acp` | 进程外运行时 SDK（JSON-RPC 协议+TS 客户端）/ 自动化 Agent Client Protocol 服务端 |
| `subagent` | 子代理能力族：in-process/dsh-sdk/Claude Code/Codex/ACP 提供者 + 委托工具 |
| `workflow` / `jobs` | 多子代理工作流引擎（worker 线程）/ 通用后台任务运行时 + `job_*` 工具 |
| `shell` / `subprocess` / `terminal` | Bash（本地/pwsh 实现）/ 子进程树 / 持久 PTY 会话能力族 |
| `fs` / `lsp` / `skill` / `web` | 文件系统（本地+sandbox+观察策略+搜索/替换工具）/ 语言服务器 / Skill 注册表 / Web 搜索抓取（DeepSeek/Exa/Perplexity） |
| `code-runtime` / `sandbox` / `e2b` | 代码执行（worker 线程）/ 进程沙箱缝（bwrap/Landlock/Seatbelt/Windows ACL）/ E2B 云沙箱 POC |
| `interaction` | 人机协作：审批、权限预设、命令、ask-user 工具、用户问题 |
| `guard` / `compaction` / `context` | 循环卫生（重复调用提醒、工具超时）/ 上下文压缩 / 模型可见上下文（工作区指令、时间） |
| `goal` / `plan` / `preset` / `todo` | 同会话目标持久化 / 计划模式 / 会话 agent 组合预设 / todo 工具 |
| `bundle` | 可安装 profile 层：`base`（每 profile 首层）、`web-app`、`headless` |
| `boot` / `extensions` / `hooks` | 应用引导胶水（app-boot、cmdline）/ Agent 运行时自修改（挂载/卸载插件）/ Claude Code & Codex hook 桥 |
| `identity` / `settings` / `credentials` / `storage` / `workspace` | 匿名身份 / 用户设置 / 凭据引用（env+.env）/ 非会话存储（JSON/SQLite）/ 工作区实体 |
| `attachment` / `spill` / `feedback` / `schedule` / `mcp` | 附件（内容寻址存储）/ 工具结果溢出策略 / 人工反馈 / 定时跟进 / MCP 客户端 |
| `examples` / `test-support` / `util` / `runtime-diagnostics` | 演示 bundle / 测试设施（llm-mock、replay、snapshot）/ 零依赖工具（`Branded<B>`、路径、超时）/ 运行时不变量 |

## 三、文字详解

### 3.1 项目定位

DeepSeek Harness 是 DeepSeek AI 开源的 **Agent 运行时（harness）**：一个把"模型 + 工具 + 记忆 + 沙箱 + 界面"组装成可运行 Agent 产品的框架。它没有"特权内核"——模型适配器、工具注册表、会话日志、Agent 主循环都是平级的 Cordis 插件，因此**每个部件都可以用配置文件（`cordis.yml` patch 层）替换或叠加**，这就是架构文档里反复强调的"一切皆插件"。

### 3.2 分层架构（自底向上）

1. **框架层 `vendor/`**：把 Cordis 及其基础库（schemastery 配置校验、cosmokit 工具库等 9 个包）以源码快照形式内置并重命名为 `@deepseek-ai/` 作用域，使框架层完全可控、可审计、可打补丁（`vendor/README.md` 记录了 18 项本地修改）。`packages/` 里每个包把 `@deepseek-ai/cordis` 声明为 peerDependency。
2. **核心库 `packages/`**：219 个包按职责分 49 组。其中 `core/` 是产品 API 主干——`dsh-session` 维护**追加式会话事件日志**（模型所见的一切都能从日志重建，"模型可见 ⟺ 已记录"是不变量）；`dsh-agent-loop` 实现 turn/step 驱动循环；`dsh-tools` 提供带作用域、带守卫（guard）的**工具执行管线**。其余各组通过**能力缝（capability seam）**扩展主干：每个能力缝 = Service Definition（接口）+ Service Provider（实现）+ Consumer（通常是模型工具），例如 `shell`、`fs`、`subprocess` 共享同一个执行世界，把 provider 指向远程沙箱，整个产品（Bash、PTY、LSP）就跟着切换。
3. **应用层 `apps/`**：`cli` 是产品启动器——`dsh web` / `dsh --profile headless "任务"`，负责把 profile（有序的 bundle patch 层叠）引导成运行中的插件树；`web` 是 Vite+React 前端，通过 `window.__DSH_BOOT__` 与 host 半区（`host/` 组的 webserver、apiproxy）对接，UI 本身也是插件（`client/` 组 39 个 `ui-*` 包）。
4. **支撑层**：`python/`（官方 Python SDK + 内置运行时）、`native/`（Linux Landlock 沙箱）、`examples/`（6 个可运行组合，同时充当无密钥快照测试的真实载体）、`patches/`。

### 3.3 工程治理体系（这个仓库最突出的特点）

- **文档即纪律**：`docs/` 有严格的层级制度（architecture → subsystems → cookbook → postmortem），大部分目录表（config-catalog、tool-catalog、module-graph、Cordis API 面）由 `scripts/gen-*.ts` **从源码生成**，CI 用 `verify-*.ts` 保证不漂移；所有文档英中双语配对，且有字数预算（`doc-budgets.manifest.json`）。
- **脚本闸门**：`scripts/` 约 100 个脚本把质量约束变成可执行的 CI 闸门——`run-gates.ts` 编排全部检查，`hygiene`（knip+publint+workspace 约束+许可证校验）、`doc-sync`（文档链接/格式/预算/生成物保鲜）、`test:coverage`（每文件 100% 覆盖率门限）、快照测试（`test:snapshot` 用录制回放代替真实 API）。
- **Agent 原生开发**：`.agents/` 内置了供 AI 编码代理使用的 11 个 skill（代码评审、推送前检查、文档标准等）和 1087 篇决策记录（Agent Notes），`CLAUDE.md`/`AGENTS.md` 把仓库约定（ESM、严格模式、插件注册必须是 effect、不变量断言等）写成代理可直接遵循的规范——这个仓库本身就是在"AI 辅助 AI 开发"的实践中长出来的。
- **CI/CD**：15 个 GitHub Actions 流水线（CI 矩阵、沙箱、e2e、文档站发布、Python 发布、Landlock 发布、issue 机器人），外加 `release/` 脚本族管理 npm 发布。

### 3.4 一些值得注意的设计约束

- **ESM 全仓**：所有包 `"type": "module"`，本地相对导入必须带 `.ts` 后缀；`dsh` CLI 源码启动走 tsx 的 ESM hook，因此被它触达的模块不能有 CJS 专用导出。
- **注册即副作用**：所有插件贡献必须通过 `ctx.effect()/ctx.on()/ctx.waterfall()`，注册返回 disposer，卸载时自动回滚——这是 HMR 和"运行中改自己"（`extensions/` 自修改能力）能成立的基础。
- **快照测试文化**：任何模型或产品用户可见的行为变化，都要在 `examples/` 里加一个真实可运行的示例并配 keyless 快照（`.cordis.snapshot.yml`），不允许只靠 mock 单测糊弄。

---

## 附录：下钻路线图（第二部分）

**已定稿的写作方案**（2026-xx，与读者确认）：

- **读者画像**：懂一点 Python、以 AI 辅助编程（vibecoding）为主、产品经理思维、设计理解力强；代码实现需要"翻译式"讲解；动手任务需可照抄。
- **写作模板**：见 `模块详解模板.md`（十段式骨架 + 分层伸缩排版：小白层正文 / 🟡进阶 / 🔴高手 / 🧪动手任务 / ❓自测 / 📚延伸）。
- **与官方 docs 关系**：导读 + 翻译 + 桥接，官方 docs 为权威来源，不重复造轮子。
- **形式**：Markdown 系列 + Mermaid 图 + 动手任务卡。

**分批推进计划**（先 3 篇标杆验证模板，复盘后再批量）：

| 批次 | 模块 | 说明 |
|---|---|---|
| 标杆 ① | `vendor/`（Cordis 框架层） | ✅ 已完成 → `详解/第01篇-vendor-Cordis框架层.md` |
| 标杆 ② | `packages/core/`（产品主干） | ✅ 已完成 → `详解/第02篇-core产品主干.md` |
| 标杆 ③ | `packages/shell/`（能力族代表） | ✅ 已完成 → `详解/第03篇-shell能力缝.md` |
| 批量 ④ | llm / typert | ✅ 已完成 → `详解/第04篇-llm与typert.md` |
| 批量 ⑤ | fs / subprocess / sandbox（执行世界） | ✅ 已完成 → `详解/第05篇-执行世界.md` |
| 批量 ⑥ | session 持久化 / session-query / compaction | ✅ 已完成 → `详解/第06篇-记忆的落地.md` |
| 批量 ⑦ | subagent / workflow / jobs（多智能体） | ✅ 已完成 → `详解/第07篇-多智能体.md` |
| 批量 ⑧ | interaction / goal / plan / guard（人机协作与治理） | ✅ 已完成 → `详解/第08篇-人机协作与治理.md` |
| 批量 ⑨ | apps/cli 与 boot → GUI 前后端（web/host/client） | ✅ 已完成 → `详解/第09篇-应用层cli与boot.md`、`详解/第10篇-GUI前后端.md` |
| 批量 ⑩ | sdk / acp / hooks / mcp / python | ✅ 已完成 → `详解/第11篇-协议与SDK.md` |
| 批量 ⑪ | scripts / docs / .github / .agents / website / examples / native / 根配置 | ✅ 已完成 → `详解/第12篇-工程体系.md` |

> 系列正文全部完成（第 01~12 篇）；进入整体审查阶段（链接有效性、出处核查、前后一致性）。
