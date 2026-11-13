# deepseek-harness 学习进度

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
