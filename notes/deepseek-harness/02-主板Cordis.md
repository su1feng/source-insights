# 02 主板 Cordis

> 类型：教程型（有序路径、逐步展开）｜前置：01-整机图.md（五层模型、ctx 键表）｜难度：中
> 源码基线：deepseek-harness `e8f7b93a34`（2026-11-11，master，0.2.2-rc.2）；拆机 02 基于 `ce7073d`（2026-10-30）
> 主线资料：拆机 02 <https://nano-ai.tech/teardown/deepseek-harness/cordis/>；配套代码 mini-cordis.js / demo.js / mini-bugs.js 本次会话已本地实跑

## 没有主板会撞的四堵墙

启动顺序靠手工清单、模块互相 import 耦合死、扩展要改核心、卸载漏清留幽灵——几百行的单体没问题，做成开放框架必然依次撞上。[claimed] 拆机 02 §01（动机叙事）

Cordis 的五个概念逐一对付：inject 消解顺序、ctx 服务容器消解耦合、waterfall 消解改核心、可逆副作用消解卸载残留。[inferred] 由下文机制推出

## 五个核心概念

插件 = 带 `apply(ctx)` 的对象，一切功能皆插件。[verified] mini-cordis.js 实现与拆机 02 §02 一致

ctx 是服务容器：`provide` 上架、按名字取用；实现上是一个 Proxy，ctx 上没有的属性去服务表找。[verified] mini-cordis.js createContext

inject 声明依赖：服务没到齐停 PENDING，到齐自动启动，消失自动停止；迷你版判断只看“在不在”，这一点正是后来 bug 之源。[verified] mini-cordis.js refresh

事件分五种分发模式：`emit` / `parallel` / `serial` / `bail` / `waterfall`。[verified] vendor/cordis/src/events.ts:31；迷你版只实现 emit 与 waterfall，waterfall 洋葱式：每层拿 `(payload, next)`，不调 next 即拦截短路。[verified] mini-cordis.js

可逆副作用：provide/on 内部都走 `effect(fn)`，撤销函数记在当前插件名下，停止时倒序全执行——插件不必自己记得清理。[verified] mini-cordis.js；真实版对应 fiber.ts 的 disposables 与 _unload

## 三幕演示（本地实跑 demo.js）

乱序挂载：先挂循环停 PENDING，依赖上线自动启动——顺序由依赖关系决定，没有手工清单。[verified] 实跑输出

审批热插拔：挂 guard 拦截 weather（“已拦截：不允许调用 weather”），dispose 后监听自动撤销，立刻恢复原回答。[verified] 实跑输出

拔模型 → 循环自动停、ctx.agentLoop 跟着下线；换新模型插件 → 循环自动重启。全程没有任何“模型换了请重启循环”的代码：连锁反应 = 依赖消失就停止 + 停止就撤销全部注册。[verified] 实跑输出

## 迷你版三个 bug（mini-bugs.js 实跑）

Bug1 同名覆盖：provide 只是 set，pro 悄悄挤掉 flash 而循环毫不知情——两个真相；且撤销函数按名字删，卸载 flash 连 pro 一起没（末尾 ctx.llm 是 undefined）。[verified] 实跑输出

Bug2 拔卡顺序反了：先删服务、后停下游，循环“收尾最后调一次模型”时模型已不在，清理读到 undefined 直接报错。先拔电源再让电脑存档。[verified] 实跑输出

Bug3 崩溃冒泡：Fiber.start 裸调 apply 无 try/catch，异常炸到挂载方；发生在启动期则后续插件全部挂不上。[verified] 实跑输出

## 真实 Cordis 的四个修复

| 坑 | 做法 | 本地源码 |
|---|---|---|
| 同名覆盖 | 每个服务实现带递增 uid；同名直接 throw，后来者进 FAILED，原件不受影响——宁可报错不要猜 | vendor/cordis/src/reflect.ts:302-304 |
| 依赖换人无感知 | epoch = 自身版本号 + 各依赖 impl.uid 拼串；一变即重载，手里引用与 ctx 永远同一份 | vendor/cordis/src/fiber.ts:617-627 |
| 拔卡顺序 | 上游卸载先等全部下游清完，才执行自己的撤销（源码注释明说）；下游收尾要花时间，故生命周期必须异步 | vendor/cordis/src/fiber.ts:712-716；等待点 reflect.ts:314-320 |
| 崩溃冒泡 | apply 套 try/catch，错误记 _error 进 FAILED，异常不外泄；FAILED 不因依赖变化自动重试 | vendor/cordis/src/fiber.ts:681-710；拦截 fiber.ts:633-634 |

## FAILED 的传染性与恢复（会话追问）

FAILED 插件不提供任何服务：同步失败当场回滚已注册的副作用（fiber.ts:527-543），异步失败 epoch 拨回 INACTIVE（fiber.ts:693-700）。[verified]

下游不会被通知“上游挂了”：依赖检查只看服务在不在（fiber.ts:617-627），下游要么一直 PENDING（静默陷阱——隔离了但没人喊你），要么依赖的是别的实现则完全不受影响。[verified]

主动发现：fiber.await() 等生命周期稳定后重抛 _error（fiber.ts:754-759）。隔离和“看得见”是两件事。[verified]

恢复唯一入口是显式调用：update() 清 _error（fiber.ts:790）→ ++_configRevision（791）→ restart() 里 ++_revision（771）→ _refresh 拼 epoch 必变（618）→ _setEpoch 的 _error 拦截（633-634）失效 → 重载；另有手动 restart()（768）。“改没改配置”不是检测出来的，是调用 update() 这个动作本身代表的。[verified]

## 状态机

迷你版只有 PENDING/ACTIVE 两个状态；真实版 PENDING → LOADING → ACTIVE → UNLOADING → DISPOSED，外加 FAILED——多出的状态各对应一个坑：LOADING/UNLOADING 为了“等”，FAILED 为了隔离，DISPOSED 为了拔掉的卡不能再插回。[claimed] 拆机 02 §05；状态枚举与 FAILED 语义 [verified] fiber.ts:143-151

## 循环为什么能换：接口与驱动分离

core/agent 定义 Agent 接口（ctx.agents），core/agent-loop 只是默认驱动器（ctx.agentLoop）——可换的前提是消费者只认接口不认实现。[verified] docs/architecture.zh.md:55（01-整机图.md 已列键表）

目前仓库只有这一个循环实现，“可换”是设计上留好的口子，还没被第二个实现真正验证；开发规范是 “Plugins, not loop changes”，日常扩展挂循环暴露的事件扩展点（agent/pre-step、agent/request、llm/stream、tools/* 均 waterfall，agent/turn-stopping 是 serial）就够用，能换循环是兜底。[claimed] 拆机 02 §06；扩展点分类 [verified] docs/architecture.zh.md:112

## 装机清单落地

packages/bundle/base/cordis.patch.yml：每行一张卡（id/name/disabled），文件头注释明示 “Row order carries no load semantics (activation is service-availability driven)”；patch 按 id 寻址、整行 config 替换、后写者胜。[verified] 本地文件头部注释与行格式

llm-pi-ai 多家模型适配器默认装上但休眠，不注册任何模型；用户配置了别家模型才实时注册路由，删配置自动撤下。[claimed] 拆机 02 §07；包存在 [verified] packages/llm/llm-pi-ai

自检整棵插件树：`dsh --profile web --dump-config`（见 01-整机图.md 验证习惯节）。

## 思想谱系与代价

服务容器+依赖注入（Spring/Angular）、waterfall（Koa 洋葱中间件）、可逆副作用（React useEffect 清理函数）、服务动态上下线（OSGi）都不是新发明；价值在把“任何注册都是副作用、依赖一变自动撤销重建”这一条规则贯彻到底，连 Agent 循环都不例外。[claimed] 拆机 02 §08

代价：门槛高（懂任何功能先懂主板）、调试难（行为散在监听器里、PENDING 静默不打印）、对单个自用产品过重、生态早期（循环可换的口子无第二实现）。[claimed] 拆机 02 §08

## 迷你版到工业版的差距清单

五种事件分发（迷你只有两种）；标准写法继承 Service 类注册；isolate 分组让同一服务名在不同分组各有实现；从 YAML 配置加载插件树、按 id 比较增量重载；热重载；TS 声明合并让 ctx.llm/ctx.tools 有完整类型。[claimed] 拆机 02 §05 对比表；isolate 字段存在 [verified] reflect.ts:302

真实 Cordis 的四个实验（real-cordis-demo.mts）本次未在本地复现——需要先在 deepseek-harness 仓库装依赖；结论暂按拆机转述，运行环节留待需要时补做。

## vendor/cordis 文件地图（参考）

| 文件 | 行数 | 职责 |
|---|---|---|
| fiber.ts | 758 | 插件实例生命周期、epoch、reload/unload、FAILED |
| reflect.ts | 406 | 服务注册、uid、dependents、同名检查 |
| events.ts | 346 | 五种事件分发 |
| registry.ts | 305 | 插件注册表 |
| utils.ts | 262 | 工具函数 |
| logger.ts | 244 | 日志 |
| context.ts | 137 | Context 定义 |
| service.ts | 105 | Service 基类 |
| index.ts | 16 | 出口 |

合计 2579 行 [verified] 本地计数；职责列 [inferred] 由文件名与本会话读到的片段推断。@deepseek-ai/cordis 4.1.0，package.json 作者字段 Shigma（社区 Cordis 原作者），vendored 进本仓库。[verified] vendor/cordis/package.json

## 待补清单（后续阶段覆盖点）

Service 类标准写法 → 阶段 3（seam 三角色）与阶段 5（写真插件）补；parallel/serial/bail 三种分发 → 阶段 2 循环走读实战 waterfall 与 serial，bail/parallel 遇到再补；YAML 配置驱动加载与按 id 增量重载 → 阶段 8 装机清单；HMR 热重载 → 阶段 8（概览级）；isolate 分组 → 无专门环节，走读遇到再补；cordis-primer 与教程 7 篇 → 随用随读，不占主线。

## 资料

拆机 02 正文：<https://nano-ai.tech/teardown/deepseek-harness/cordis/>

配套代码下载：mini-cordis.js、demo.js、mini-bugs.js、real-cordis-demo.mts（拆机文末链接；本次会话临时副本在 .tmp-dsh-learn/，可能被清理）

官方文档：docs/cordis-primer.zh.md（入门）、docs/cordis-tutorial/（教程 7 篇）、docs/subsystems/core.zh.md（Agent 接口与循环）[claimed] 拆机 02 参考

下一步钩子：阶段 2 一句话的旅程——2100 行 Agent 循环走读；拆机 03 未发布，主资料改用 dsh-notes 04 与 deep-dive Ch06。
