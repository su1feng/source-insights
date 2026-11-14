# AI 编码质量方法论

类型：教程型。前置知识：无需了解被引用的两个仓库；概念难度：中。本笔记给出一条遇到 agent 产出问题时依次执行的有序路径，并以两个真实仓库作为落地参照。

> 材料来源：Lauren Tan《All Code Is Slop (Except Mine)》（AI Engineer CODE 2026，官方页面含完整逐字稿）＋ deepseek-harness（下称 dsh，源码基线 `e8f7b93a34`）＋ team-chat（`main` 分支）。标注 `[verified]` 的论断均已对照源码或逐字稿核实；`[inferred]` 为由证据推断；`[claimed]` 仅见于外部说法。

---

## 公理：质量不是靠事后审查，是靠环境默认产生

演讲对这件事的定义值得当成起点，因为它决定了后面每一条推导：信任 = 投资 agent 的运行环境，让它默认做对的事；不是盲信，也不是靠 review 抓漏。`[verified]`（逐字稿 1:02:47–1:03:03）

这个定义有一个直接推论，也是整套方法论的检验标准：任何一条约束，如果 agent 违反它时没有强制机制，那它不是约束，是期待。`[inferred]`

于是所有质量工作收敛成同一个动作：把期待改成机制。

---

## 一、主循环：纠错降级链

每当 agent 犯了一个错，不要只说"下次注意"，按下面四问依次降级。顺序不能乱，因为越靠前约束越强、越可复现。

```
第 1 问：能不能让这个错误在物理上不可能发生？
   └─ 手段：改 API、改架构、改目录边界、类型系统收窄
   └─ 判据：错误写法会编译失败 / 运行时报错 / 根本无法表达

第 2 问：不能消灭，能不能变成确定性检查？
   └─ 手段：lint、AST 检查、契约测试、生成器独占
   └─ 判据：CI 会红，且失败信息能指出根因

第 3 问：检查写不出来，能不能让它被自动发现？
   └─ 手段：运行时验证、性质测试、快照回放、形式化验证
   └─ 判据：不需要人盯着也能暴露

第 4 问：以上都不行（本质是判断题）
   └─ 才写进 rules / AGENTS.md / 注释
   └─ 并且要接受：这一层不可信，只是聊胜于无
```

### 1.1 第 1 问优先于第 2 问

这是最容易搞反的地方。演讲的原话是：加规则之前先问能不能变成检查，加检查之前先问能不能用更好的设计把错误选项消灭掉。`[verified]`（逐字稿 1:10:34–1:10:45）

两个仓库都把第 1 问当常态，而不是最后手段：

| 做法 | 消灭了什么 | 出处 |
|---|---|---|
| 生成目录由生成器独占 | 手改、手加生成物；报错直接说"要么加进 scripts，要么删掉" | team-chat `packages/protocol/test/generated.test.ts` `[verified]` |
| 产品名只在 `product.json` 一处 | 名字写错、漏改 | team-chat `apps/desktop/src/copy/mod.rs:636` `product_name_is_never_typed` `[verified]` |
| 禁 `as unknown` | 用类型断言绕过兼容性检查 | dsh `scripts/verify-no-unknown-casts.ts:1` `[verified]` |
| 禁顶层 `await` | 模块求值变异步，拖累所有 importer | dsh `scripts/verify-no-top-level-await.ts` `[verified]` |
| 目录名 = 架构边界 | 跨进程 import 导致的卡顿 | 演讲里的 `electron-main` / `electron-renderer` `[verified]` |
| 产品代码禁 `useEffect` | 需要极高技巧才用对的 API 被滥用 | 演讲 `[verified]` |

### 1.2 什么时候值得建检查

演讲只说"优先变成检查"，没说阈值。阈值是复现率：一次错误只需修一次，成本是零；真正的成本是以后每次 review 都要重新发现它。所以会重复出现的错误才值得建检查，一次性的直接修。`[inferred]`

两个仓库都严守这条：dsh 每个 `verify-*` 脚本都配一个 `*.spec.ts`，即门自己也被测，建门成本不低，因此只给真正会复发的类建门。`[verified]`

### 1.3 失败信息必须指向根因

演讲禁掉 99% 注释后，把"为什么不能这么做"移进 lint 错误信息，直接告诉 agent 去修根因而不是绕过。`[verified]`（逐字稿 1:10:20–1:10:32）

两个仓库的实例：team-chat 的 px 棘轮失败时输出 `use a token from ui/native/tokens.rs`（`apps/desktop/src/ui/native/tokens.rs:1697-1698`）`[verified]`；dsh 的 `no-bare-dispatcher` 在头注释里直接写明它防的是哪一个真实缺陷。`[verified]`

结论：检查说不清为什么错，使用者（尤其 agent）只会绕过它。`[inferred]`

---

## 二、两个必需补丁

演讲没提这两个机制，但两个仓库独立收敛到了同样的解法。这本身是它们值得单独成节的证据。`[inferred]`

### 补丁 1：棘轮——让"还欠多少"变成只能下降的数字

演讲说"先加 lint 拦住新增，再清理旧模式"，但漏了一个问题：清理速度赶不上新增速度，因为 agent 复制旧模式的速率是指数的。`[inferred]`

解法是把存量写成常量，断言只能减不能增：

| 仓库 | 机制 | 出处 |
|---|---|---|
| dsh | 存量按「路径 + 语法 token 的 sha256 + 次数」锁死，只能删不能加 | `scripts/no-unknown-casts.baseline.json` `[verified]` |
| dsh | lint 规则集 pin 成 sha256 + 条数（source 89 / test 84 / example 88），删规则、降 severity、改 options 都会让 spec 变红——防有人悄悄放宽门 | `scripts/lint-rule-fingerprint.spec.ts` `[verified]` |
| team-chat | `VIEWS_PX_LITERALS = 1431`、`VIEWS_SCALE_LITERALS = 0`，测试断言实际值不超过常量 | `apps/desktop/src/ui/native/tokens.rs:1432,1436,1676` `[verified]` |

为什么这是方法论级的：它把永远说不清的"技术债"变成可见的进度条。没有棘轮，清理是无底洞；有棘轮，清理有终点（归零）。`[inferred]`

配套纪律：棘轮必须防"空语料"。dsh 的 `scripts/AGENTS.md` 明文要求门要 guard against an empty or narrowed corpus——一个因为什么都没扫到而通过的检查，比没有检查更危险。`[verified]`

### 补丁 2：文档即数据——让散文变成检查的输入

演讲的诊断是散文不可执行所以不可信，解法是禁掉 99% 注释。诊断对，但解法激进。`[inferred]`

team-chat 给了更可移植的答案：`apps/desktop/src/copy/mod.rs:483` 的 `developer_words()` 在 `:484` 用 `include_str!("../../../../docs/PRODUCT.md")` 在编译期把产品文档读进二进制，解析出「开发者词」黑名单，再由 `no_developer_words_outside_dev_section` 遍历所有非 dev 节断言不出现这些词。`[verified]`

这一步的意义：文档不再是约束本身，而是约束的数据源。散文依然不可执行，但它可以喂给可执行的东西。`[inferred]`

两个仓库的另一半做法是给散文加预算，而不是删掉它：dsh 有 `verify-doc-budgets`（每篇文档词数上限）、`doc-typecheck` + `verify-type-equiv`（文档里的 ts 块必须真编译且与源码声明逐字一致）、`verify-concrete-terms`（全库禁某个含糊词）`[verified]`；team-chat 有文案编目、开发者词黑名单、中英混排空格测试。`[verified]`

合起来的原则：不要试图禁掉散文，要把它压缩到可枚举、可校验的规模。`[inferred]`

---

## 三、顺序纪律：内循环先于外循环

演讲强调这是顺序问题而非优先级问题：如果内循环产出的是 slop，把内循环自动化只会产出更多 slop，然后你得往里砸越来越多的 token。`[verified]`（逐字稿 1:17:01–1:17:10）

强制顺序：

1. 改进代码库——先加检查拦住新反模式，再清理旧模式（不先拦，清理赶不上新增）
2. 把约束变成可执行检查
3. 把验证能力做扎实
4. 最后才扩自动化外循环

两个仓库都停在 3 与 4 之间：都有很强的人类合并门，都没有"fuzz 找 bug → 自动开 PR → 自动合并"的外循环。dsh 是 `weighted-approval` 2 分阈值，作者自己的历史 merged PR 数最多贡献 1.1 分且不能单独满足阈值（`.github/review-ownership/README.md`）`[verified]`；team-chat 是 V1 checklist 人工 pass/fail。`[verified]`

这不是落后，而是遵守了顺序：在第 1–3 步没做到"PR 默认就是对的"之前，自动合并只是把 slop 提速。`[inferred]`

---

## 四、分层：往哪一层下注取决于约束条件

这是两个仓库最有价值的差异：同一套方法的两种合法配置，不是对错之别。`[inferred]`

| | dsh：建门 | team-chat：建契约 + 保留人工 |
|---|---|---|
| 主力载体 | 40+ 个自建 AST 检查 + 每个门自带 spec | 生成器独占 + 跨语言契约（Zod ↔ Serde）+ 编译器 |
| 覆盖率 | per-file 100%，CI 门，且当删除工具用 | 没有覆盖率门槛 |
| 验证重点 | 快照回放真实 profile 子进程，禁止用 mock 自证 | 真实环境：种子脚本只走公共 API，每次 PR 起真 `wrangler dev` |
| 设计品味 | 不适用 | 明确标为非自动，能机械判定的那条才升级成门 |
| 为什么这样 | 包极多、agent 是主要贡献者，必须自建门来管 agent | 小团队 + 双运行时，必须在线格式上锁死 |

dsh 的覆盖率口径值得单独记：`vitest.config.ts:426` 是 `perFile: true`，而 `docs/testing.md:10` 明说未覆盖的行往往是死代码、这个门在提示你删掉它，而不是提示你补测试。`[verified]`

怎么选：主要问题是 agent 大面积产出、需要自动拦截，就走 dsh 路线，把力气花在门的基础设施并给门写测试；主要问题是多人多语言协作、接口老是漂移，就走 team-chat 路线，把力气花在契约和真实环境；有设计或视觉要求，就学 team-chat，能机械判定的那条升级成门，判不了的明确标为人工。`[inferred]`

---

## 五、明确不自动化的部分

这套方法最容易的失败方式，是把"一切都该变成检查"当信仰。三个诚实的边界：

1. 一次性工作不值得建门。探索原型、迁移脚本、局部脚本，建门成本高于错误成本。`[inferred]`
2. 设计品味无法自动。team-chat 的做法是标杆：`docs/design/COMPLETENESS.md` 开头声明该 spec 是 binding、reviewer 只在通过第 4 节清单后才接受界面，而支撑它的测量目录不在仓库里，第 4 节因此是人工逐条核对而非 CI。`[verified]` 同一仓库的截图自测已有可执行机制，但 `scripts/windows-screenshots.sh:4-5` 明写 CI 已不再运行它。`[verified]` 结论：能指标化但不能自动化的，就指标化 + 人工。`[inferred]`
3. 别照抄"禁注释"。它的前提是 lint 错误信息足够好；如果检查说不出为什么错，禁掉注释等于把知识一起删了。`[inferred]`

---

## 六、落地检查清单

拿去改自己的仓库，逐条问：

### 第 1 层：能否消灭

- [ ] 有没有"错误写法能被表达出来"的接口？能不能从结构上让它无法通过编译？
- [ ] 目录名 / 模块边界是否表达架构？（还是只存在于文档里）
- [ ] 生成物能否由生成器独占，手改直接报错？

### 第 2 层：能否变成检查

- [ ] 最近三次纠正 agent，有几次变成了检查？几次变成了 AGENTS.md 的一行？
- [ ] 每个门是否自带测试？（门自己错了谁知道）
- [ ] 门是否会因为"扫到 0 个"而静默通过？
- [ ] 有没有棘轮？存量是常量还是模糊感觉？
- [ ] 有没有防止悄悄放宽门的机制（规则集指纹、baseline 只能减）？
- [ ] 检查的失败信息是否指向根因而非禁止行为？

### 第 3 层：验证够不够

- [ ] agent 能不能自己运行代码、自己看结果？（不能，就只能靠人看 diff）
- [ ] 验证是否用了真实环境，还是靠 mock 自证？
- [ ] 有没有"共享词汇表"（功能地图）让人和 agent 说同一句话？

### 散文层：压缩而不是删除

- [ ] 文档有词数预算吗？
- [ ] 文档里的代码示例会被编译检查吗？
- [ ] 有没有"文档即数据"的地方——让规则列表直接喂给测试？

### 顺序

- [ ] 自动化做在内循环修好之后了吗？（还是先建了流水线）

---

## 一句话收束

方法论 = 一条降级链（设计 → 检查 → 验证 → 散文）+ 两个补丁（棘轮、文档即数据）+ 一条硬顺序（内循环先于外循环）+ 一条边界（承认哪部分必须人工）。

缺降级链，会退化成"堆规则"；缺棘轮，会退化成"永远清不完的债"；缺顺序，会退化成"用更多 token 跑更快的 slop"；缺边界，会退化成"假装一切都自动化了"。

---

## 出处

- 演讲：Lauren Tan, *All Code Is Slop (Except Mine)*, AI Engineer CODE 2026，<https://ai.engineer/code/2026/talks/lauren-tan>（含完整逐字稿与字幕下载）
- 参照实现 A：deepseek-harness（TypeScript / pnpm monorepo，agent harness），本笔记基线 `e8f7b93a34`
- 参照实现 B：team-chat（Rust + Cloudflare Workers），<https://github.com/Joye-Co/team-chat>，`main` 分支
