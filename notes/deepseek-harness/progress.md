# deepseek-harness 学习进度

> 当前位置（2026-11-14）：下一步 = 完整流转走读（阶段 0 新环节，六站：启动装配→输入→唤醒领取→turn/step→工具→收尾）；阶段 1 Cordis 已完成；阶段 2 循环部分（dsh-notes 04 全章）已讲完、笔记未落。

## 2026-11-13 · 阶段 0（完成）

- 会话中讲解整机图，用户自查三问全部通过；`01-整机图.md` 已按会话记录修订为正式笔记（含两个边界：零步轮次、一步多工具）。
- 本地检出基线 `e8f7b93a34`（master，0.2.2-rc.2）；已核实 agent-loop/src 2016 行、vendor/cordis/src 2579 行。
- 跳过：`npx @deepseek-ai/dsh web` 跑通（用户要求不装）。
- 2026-11-13 计划调整：本阶段新增“完整流转走读”环节（动态广度图），尚未进行。

## 2026-11-13 · 阶段 1（完成）

- 拆机 02 讲解完毕：四堵墙、五概念、三幕演示（本地实跑 demo.js）、mini-bugs.js 找茬（用户抓到同名误删与崩溃冒泡，漏拔卡顺序）。
- 追问深挖两个机制：FAILED 不提供服务/静默陷阱/await() 主动捞错；update() 四步恢复链（清 _error → configRevision → revision → epoch 必变）。均已定位到 fiber.ts/reflect.ts 行号。
- `02-主板Cordis.md` 已按会话记录写成（教程型，含四修复对照表、状态机、装机清单核实、文件地图）；补核实三处：cordis.patch.yml 行序注释、events.ts 五种分发模式、cordis 血统（@deepseek-ai/cordis 4.1.0，作者 Shigma）。
- 遗留：real-cordis-demo.mts 四个实验未本地复现（需先在 dsh 仓库装依赖，用户暂缓安装）。
- 下一步：走读完整流转（补阶段 0 新环节）；随后阶段 2 收尾——循环部分 A/B/C 三段已讲，待确认后落笔记 03-Agent循环.md。

## 2026-11-13~14 · 阶段 2 循环部分（讲解完，笔记未落）

- dsh-notes 04 章 A/B/C 三段已讲：A 时间单位+Inbox（自查通过，追问 inbox/claim 后用场景版重讲）；B 相位机+wake latch（用户要求先流程图再讲问题与解法，重讲通过）；C step() 内部（system/message 变了才记、运行时上下文变空要发作废消息、firstAttempt 准入、打断保半句标记 interrupted、agent/request-error 重试决策点默认不重试、三返回值 null 驱动 ReAct、turn-stopping 数据说话）。
- 本地锚点：agent.ts Phase 53 / 别名 180-190 / wakeDriver 235 / whenIdle 259 / turn 333 / step 441；inbox.ts 已迁至 agent-loop/src 并改为 session 投影（行 26，dsh-notes 基线在 agent/src，版本漂移）；runtime-types.ts AgentStatus 136 / send 245 / 不可改 messages 362。

## 2026-11-14 · 方法调整与计划更新

- 学习方法定版：静态结构 → 动态完整流转（阶段 0 前置）→ 逐部件深度（从 Cordis 开始）；已写入 plan.md。
- 调研四套参考编排：拆机 9 部只出 01/02（03 一句话的旅程=完整流转专题，未发布）；dsh-notes 12 章，主干顺序=数据流顺序；deep-dive 35 章 7 部分，分角色阅读路径，证据三档标注出处；dshx 六步教学版+Demo 8 个+事故复盘 4 篇+记忆走 MCP。
- 阶段 6 恢复并细化：工程化与质量（开发规范门禁/ADR/事故复盘/测试策略）+ 异常恢复全景串联 + 记忆现状核实。
- 已提交 5b69d22（阶段 0-1 笔记、progress、AGENTS 先讲后写、plan 调整）。
- 下一步：完整流转走读第一站——启动装配。
