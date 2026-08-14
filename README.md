# DeepSeek Harness 学习文档（StudyDocs）

> 面向"小白 → 进阶"的 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（`dsh`）源码学习系列：**需求驱动、真实产物、链接直达源码**。

## 这是什么

DeepSeek Harness 是 DeepSeek AI 开源的 Agent 运行时框架——一个把"模型 + 工具 + 记忆 + 沙箱 + 界面"组装成可运行 Agent 产品的插件化平台（219 个 npm 包，49 个分组，"一切皆插件"）。本仓库是一套**从零学习这个大型项目的结构化文档**，不是官方文档的翻译，而是：

- **需求驱动的讲解**：每篇按"功能需求（WHY）→ 架构设计（WHAT）→ 实现落点（HOW）→ 产物演示（EXAMPLE）"四维展开，先讲清"要什么、为什么、还有哪些路"，再讲"选了哪条路、怎么落地"；
- **真实产物展示**：所有示例产物（会话日志、配置树、审批记录）均来自仓库现存快照文件的逐字符拷贝或实测命令输出，零编造；
- **链接直达源码**：每个文件引用都带相对路径链接（VS Code 点击直达）+ 行号标注，官方 JSDoc 负责权威细节，本系列负责小白化转述；
- **分层可伸缩**：正文小白可读，🟡 进阶 / 🔴 高手块按需取用。

## 目录结构

```
deepseek-harness-StudyDocs/
├── README.md               # 本文件：项目说明 + 导航
├── 项目架构总览.md          # 总览层：文件树 + 对照表 + 路线图
├── 模块详解模板.md          # 写作层：统一范式与排版规范（v2.4）
└── 详解/                    # 下钻层：系列正文（新篇持续追加于此）
    ├── 第01篇-vendor-Cordis框架层.md
    ├── 第02篇-core产品主干.md
    ├── ……
    └── 第12篇-工程体系.md
```

三层设计（总览 → 模板 → 详解）保证：**新读者从总览入手，作者按模板续写，正文按主题扩展**——可持续迭代管理。

## 系列索引（12 篇）

| 篇 | 主题 | 一句话 |
|---|---|---|
| 01 | `vendor/` Cordis 框架层 | 可拆卸、可替换、可组合的运行时底座 |
| 02 | `packages/core/` 产品主干 | 会话可重建、工具可安全执行、循环可观测 |
| 03 | `packages/shell/` 能力缝 | 能力可插拔替换（缝三件套） |
| 04 | `packages/llm/` 与 typert | 模型适配缝与类型系统 |
| 05 | fs / subprocess / sandbox | "执行世界"：Agent 的手脚家族 |
| 06 | 持久化 / session-query / compaction | 记忆的落地 |
| 07 | subagent / workflow / jobs | 多智能体：把任务分出去 |
| 08 | approval / goal / plan / guard | 人机协作与治理 |
| 09 | apps/cli 与 boot | 产品如何被启动 |
| 10 | apps/web + host + client | GUI 前后端：浏览器到主机的桥梁 |
| 11 | sdk / acp / hooks / mcp / python | 把产品接出去 |
| 12 | scripts / docs / .github / .agents | 工程体系：让产品活下来 |

## 怎么读

1. **推荐路径**：`项目架构总览.md`（半小时建立全貌）→ 第 01~03 篇（地基与主干）→ 按兴趣挑选其余篇目；
2. **最佳阅读环境**：VS Code 打开本仓库，`Ctrl+Shift+V` 预览 Markdown——所有源码链接可 **Ctrl+点击直达**，产物表格、Mermaid 图、原文引用卡均可渲染；
3. **对照源码**：官方 JSDoc 全覆盖（`verify-export-jsdoc` 闸门），VS Code 里 hover 任意导出符号即可看到权威注释；
4. **动手任务**：每篇第 4 节（动手验证）的命令已实测可运行，照抄即可复现产物。

## 快速开始（重要：链接依赖本地源码）

文档中约 150 处 `../../deepseek-harness/...` 链接**指向 dsh 源码仓库，不是本文档仓库内的文件**。要在本地完整使用（点击直达源码、跑动手任务），请按**推荐布局**把两个仓库克隆为同级目录：

```
你的目录/
├── deepseek-harness/            # ① 源码：git clone https://github.com/deepseek-ai/deepseek-harness
└── deepseek-harness-StudyDocs/  # ② 本文档（已在此布局下编写）
```

然后：

```bash
cd deepseek-harness
pnpm install          # 安装依赖（动手任务需要，Node ^22.19 || >=24）
```

- **链接为什么这样设计**：这套文档的核心体验是"文档 ↔ 源码 ↔ 官方 JSDoc"三方对照（Ctrl+点击直达、hover 看注释、行号定位）——同级克隆后全部链接即刻生效，无需改任何文件；
- **只浏览不下代码时**：正文、产物表格、图均完整可读；`../../deepseek-harness/...` 链接在网页上会 404，属预期行为，按上述布局克隆后即可恢复；
- **动手任务**：涉及 `pnpm dsh`、快照 JSONL 解析的命令需要源码仓库，且部分命令需要 `DEEPSEEK_API_KEY`（无 key 的任务在文中已标注为 keyless 可跑）。

## 版本与维护

- 本系列基于 **deepseek-harness v0.1.0-rc.5**（developer preview）编写；
- 仓库迭代迅速，文档引用的行号/文件路径可能随上游演进漂移——发现失效请提 issue 或 PR 修正；
- 写作规范见 `模块详解模板.md`（v2.4），新增篇目照此格式续写至 `详解/` 目录。

## License

文档内容基于对 [deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)（MIT）源码的研读整理，引用内容版权归原项目所有；本文档本身采用 [MIT](LICENSE) 协议。
