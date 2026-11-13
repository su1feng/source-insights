# deepseek-harness 学习计划

学习对象：[deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)（命令行名 dsh）——DeepSeek 开源的 Agent Harness，口号 Everything is a Plugin，基于 vendored Cordis 插件框架。

版本基线提醒：各资料有自己的源码快照（nano-ai 拆机系列 ce7073d / 2026-10-30，dsh-notes 0.1.1-rc.2，deep-dive rc.5，dshx 3a46bd6），博客当地图，行号以所用源码为准。

## 000 拆解总览：整机五层

| 层 | 类比 | 是什么 |
|---|---|---|
| 主板 | 主板 | Cordis 插件框架（vendor/）：Context、Service、Fiber、Effect，卸载即回滚 |
| 核心插件 | CPU、内存 | Agent 循环、提示词组装、工具、会话日志、模型接口（packages/core/） |
| 能力插件 | 外设 | 文件、Shell、沙箱、MCP、子 Agent、定时任务……（packages/ 其余约 50 组） |
| 装机清单 | 配置单 | Profile（web/headless/sdk/sdk-minimal/acp）+ Bundle + patch 整块覆盖 |
| 外壳 | 机箱和显示器 | 网页、命令行、桌面端、SDK（apps/） |

## 001 拆解顺序：九个部件

主线 = nano-ai 拆机系列的部件顺序。

| # | 部件 | 一句话 | 备选资料 |
|---|---|---|---|
| 01 | 整机图 | 一句话在机器里走一圈；五层模型 | dsh-notes 01 |
| 02 | 主板 Cordis | 手写 150 行迷你主板，找 3 个 bug，对照真实 Cordis | dsh-notes 02 · deep-dive Ch02/05 |
| 03 | Agent 循环 | 一句话的完整旅程 | dsh-notes 04 · deep-dive Ch06 |
| 04 | 会话日志 | 为什么每一步都要记下来（append-only） | dsh-notes 05 · deep-dive Ch07 |
| 05 | 工具与沙箱 | 模型怎么安全地动手 | dsh-notes 07/09 · deep-dive Ch09/13 |
| 06 | 上下文管理 | 对话太长了怎么办（压缩四层机制） | dsh-notes 10 · deep-dive Ch16 |
| 07 | 子 Agent | 一个 Agent 怎么指挥一群 Agent | dsh-notes 11 · deep-dive Ch15 |
| 08 | 装机清单 | 同一个内核怎么变成不同产品 | deep-dive Ch04 · dshx Profile 与 Bundle |
| 09 | 横向对比 | DSH 和 Claude Code 等主流 Agent 的设计差异 | deep-dive Ch19/20 |

## 002 学习顺序：六个阶段

`mermaid
flowchart TD
    A[阶段0 跑起来+读整机图] --> B[阶段1 手写迷你 Cordis]
    B --> C[阶段2 主干四件套 循环 日志 提示词 工具]
    C --> D[阶段3 Seam 三角色+沙箱+压缩]
    D --> E[阶段4 编排与对外接口]
    E --> F[阶段5 动手 Demo+写一个插件]
`

| 阶段 | 做什么 | 主资料 | 备选 |
|---|---|---|---|
| 0 | npx @deepseek-ai/dsh web 跑通，读拆机 01 建立五层模型 | 拆机 01 · EP16 简版 | dsh-notes 01 |
| 1 | 读拆机 02，跟着手写迷你 Cordis | 拆机 02 | dshx Cordis 内核 |
| 2 | 循环→日志→提示词→工具，逐行走读 | dsh-notes 04→05→06→07 | deep-dive Ch06-10 |
| 3 | 挑一个 seam 看穿 Definition/Provider/Consumer 三角色；沙箱、压缩各过一遍 | dsh-notes 08→09→10 | deep-dive Ch11-14 |
| 4 | 编排与对外接口，通读 | dsh-notes 11→12 | deep-dive Ch15 |
| 5 | dshx 8 个 Demo；照官方 cookbook 写一个真插件 | dshx Demo 1-8 · docs/cookbook/adding-a-tool | — |

读每个包记录三件事：提供什么服务 / 依赖什么服务 / 卸载清理什么。
验证习惯：dsh --dump-config 把插件树打出来，对照 patch 层看。

## 003 资料清单

- 官方文档站：https://deepseek-harness.github.io/deepseek-harness/
- nano-ai 拆机系列（主线）：https://nano-ai.tech/teardown/deepseek-harness/
- nano-ai EP16 简版（导读）：https://nano-ai.tech/articles/agent-deepseek-harness/
- dsh-notes（12 章，按第一次理解一个 Agent Harness 的顺序重排）：https://dragonchencl.github.io/dsh-notes/
- deep-dive（35 章，含 507 篇 ADR 决策梳理）：https://github.com/xiaonancs/deepseek-harness-deep-dive
- dshx 源码拆解教程（仓库地图 + 8 个 Demo）：https://dshx.dev/docs/tutorial/source/repo-map/
- 架构论文 A Programming Paradigm for Spatiotemporal Composability：https://arxiv.org/abs/2608.25512

