# AGENTS.md

本仓库用于两类工作：**阅读/学习开源项目**，以及**深入理解/拆解技术博客**。只做理解和沉淀，不产出代码，因此任何开发型 skill（tdd、correct、architect、blast-radius、swarm 等）在本仓库一律不用。

当前主线任务：学习 [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)（dsh）。计划唯一入口：notes/deepseek-harness/plan.md，按其六阶段顺序推进，不要另立计划。

skills 位于仓库根的 .agents/skills/，按下面的场景选用，不要全量加载。

## 一、学习开源项目

| skill | 用途 |
|---|---|
| how | 讲清某个子系统/部件怎么工作、架构走读（阶段 1-4 每个部件） |
| why | 挖设计决策：为什么这么做、取舍与演进（配 dsh 的 507 篇 ADR 很合适） |
| teach | 学完一个部件后检验并输出真正的理解 |
| statechart | 把 agent-loop、fiber 生命周期等状态机画成状态图 |
| sharpen | 开始新部件但范围模糊，先把学习任务磨尖 |

## 二、拆解技术博客

| skill | 用途 |
|---|---|
| principle-explain-the-number | 博客给出的数字（行数、包数、性能）先核实再引用 |
| why | 追溯博客结论的出处：决策、PR、ADR |
| technical-writing | 把拆解结果整理成结构化笔记 |
| unslop | 发布稿去掉 AI 腔 |
| bro | 博客里讲不清的概念，用大白话复述一遍 |

## 三、两类工作通用

| skill | 用途 |
|---|---|
| recall | 跨会话继续，先恢复之前学到哪了 |
| ketchup | 会话中断后追赶当前进度 |
| in-your-own-words | 开始前用自己的话复述目标和范围 |
| show-me-your-work | 理解过程留痕，保证结论可回溯 |
| principle-guard-the-context-window | 读大文件、长输出时控制上下文消耗 |

## 排除

understand 系列（understand、understand-chat、understand-explain 等）在本项目不使用，这是用户的明确要求。用 how/why/teach 加人工读源码替代。

## 产物约定

- 每个部件学完，把理解沉淀到 notes/deepseek-harness/ 下按部件命名的笔记文件；plan.md 只放计划，不写知识点。
- 引用源码必须给 文件:行号，并以当时实际检出的源码为准；博客行号只是参考。
- 博客给出的数字必须核实后才能写进笔记，标注出处。
- 学习计划阶段 5 的动手环节（写插件）发生在 dsh 环境中，不属于本仓库，本仓库只沉淀其结论。
- 学习会话结束时，若进度有推进，把 学到哪、下一步更新回 notes/deepseek-harness/ 目录，保证下次会话能无缝续上。

## 写作纪律
- 每个事实只有一个家：一个知识点只写在一份笔记里，其他笔记只链接过去，不复制。
- 每篇笔记先标注类型：教程型（有序路径、逐步展开）或参考型（查当前行为），两者不混写；教程型再标注前置知识与概念难度，先修在前。
- 结论标证据等级：[verified]（自己读过源码核实）/ [inferred]（由证据推断）/ [claimed]（仅来自博客或社区，未核实）；引用源码带 文件:行号。
- 并列内容保持对称：同类条目结构不一致，通常说明漏了一层抽象，先补齐再写。
- 笔记按源码版本冻结：源码演进导致结论失效时，不改旧篇，另起新篇并各自标注基线版本。
- 一段物理一行（编辑器软换行），代码块和表格除外。

