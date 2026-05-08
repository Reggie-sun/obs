
## claude
如果我要检验新加的功能，我该用什么skill

根据你的需求（检验新加的功能），匹配以下 skill：

|场景|Skill|说明|
|---|---|---|
|**主动测试验证，发现问题并修复**|`/qa`|系统化 QA，会定位问题并修复|
|只想看报告，不需要自动修复|`/qa-only`|只输出报告，不改代码|
|新功能涉及浏览器 UI|`/browse` 或 `/gstack`|无头浏览器端到端验证|
|想看代码健康/质量指标|`/health`|代码质量仪表盘|

**推荐**：直接用 `/qa` — 它会系统化地测试新功能，发现问题还会自动修复。

如果是 retrieval 相关的新功能，则应该用 `/retrieval-next`，它会自动路由到正确的验证层级。

main.tsx

如果我想优化文档入库的检索这种功能呢先/qa之后呢

文档入库检索优化的推荐流程：

```
/qa (验证现状) → /retrieval-next (逐层优化) → 如果超出 lite 边界 → /retrieval-plan
```

具体来说：

|步骤|Skill|作用|
|---|---|---|
|1. 先摸底|`/qa`|验证当前入库 + 检索功能是否正常，建立 baseline|
|2. 日常优化|`/retrieval-next`|层级路由，按顺序优化：baseline → supplemental → chunk → router → rerank|
|3. 遇到跨 lane / 阶段判断|`/retrieval-plan`|收敛边界决策，超出 lite 边界时升级|

**关键约束**（来自 memory）：

- 不改主契约：`/retrieval/search`, `/chat/ask`, `/system-config`
- 优化顺序不能跳：baseline → supplemental → chunk → router → rerank
- eval 异常或 baseline 不可复现 → 停机修证据链

简单说：`/qa` 看现状 → `/retrieval-next` 做优化。如果 `/retrieval-next` 发现问题超出单层范围，再升级到 `/retrieval-plan`。

为什么不用brainming

好问题！两者的适用场景不同：

|Skill|适用场景|输出|
|---|---|---|
|`/brainstorm`|**设计阶段**：全新方案、架构重设计、多方向探索|设计 spec、方案对比、决策文档|
|`/retrieval-next`|**优化阶段**：在现有系统上渐进改进|具体改动 + 验证，按层级推进|

**你的场景判断**：

|想做什么|用哪个|
|---|---|
|调 chunk 策略参数、改 rerank 权重、增补检索路径|`/retrieval-next`|
|探索"要不要换 embedding 模型"、"ingestion 架构要不要重构"|`/brainstorm`|

**当前项目状态**（来自 memory）：

- Phase 2A 已收口，Phase 2B 有 baseline
- 系统已有成熟的 ingestion → retrieval pipeline

所以"优化文档入库检索"默认走 `/retrieval-next`（在现有框架内改进）。

**什么时候升级到 brainstorm**：

- 想探索 3+ 种完全不同的方案
- 涉及架构级改动（换模型、改存储结构、重设计 pipeline）
- 需要多方评审（PM、架构师、设计师）时用 `team-brainstorm`

简言之：**改进 → retrieval-next，重设计 → brainstorm**。

SKILL.md

我想分析一下我要不要改上下文大小，并发数量，chunk大小该用什么skill

这个场景最匹配的是 `/design-shotgun`（多方案设计比选）。

**为什么是 design-shotgun**：

你面临的是**多参数、多方案**的决策问题：

|参数|需要对比的方案|
|---|---|
|上下文大小|当前值 vs 几个候选值|
|并发数量|低/中/高并发|
|chunk 大小|小chunk精度 vs 大chunk上下文|

这正是 `design-shotgun` 的场景 — 生成多个方案，对比优劣，给出推荐。

**推荐流程**：

```
/design-shotgun (分析方案) → 决策后实施 → /benchmark (验证性能)
```

|步骤|Skill|作用|
|---|---|---|
|1. 方案比选|`/design-shotgun`|生成多组参数配置方案，横向对比优劣|
|2. 实施改动|`/retrieval-next` 或 `/workflow-lite-plan`|按选定方案落地|
|3. 验证效果|`/benchmark`|性能回归测试，确认改动有效|

**对比其他 skill 为什么不合适**：

|Skill|为什么不用|
|---|---|
|`/brainstorm`|太发散，适合从零探索；你已经有明确的参数要调|
|`/qa`|测试功能是否正常，不是做参数调优决策|
|`/benchmark`|验证性能用的，是改动**之后**的事|

**一句话**：多方案比选 → `design-shotgun`，单方案执行 → `retrieval-next`。

test_pre_stock_screening_service.py

那我想知道我现在的入库索引有没有问题该用什么skill

想知道"入库索引有没有问题"——这是**探索性检测**场景：

|你的需求|推荐 Skill|行为|
|---|---|---|
|**全面检测，发现问题并自动修复**|`/qa`|扫描 ingestion/索引链路，定位问题并修复|
|**只想看问题报告，不自动改**|`/qa-only`|只输出检测报告，你自己决定修不修|
|**retrieval 特定的 baseline 检查**|`/retrieval-next`|按 retrieval 层级验证，检查索引质量|

**建议**：直接用 `/qa`

它会系统化检测：

- 文档解析是否正常
- 索引结构是否健康
- 检索结果是否预期
- 有没有明显的性能/质量问题

检测完会给你报告，有问题的话会提议修复方案。

如果是 retrieval 特定的索引质量问题（比如 chunk 质量、向量索引健康度），用 `/retrieval-next` 会更针对性。

2026-04-14-portal-standard-qa.md

我想对比一下文档入库的实际效果和真实文档侧差距呢，以及给出优化方向。用什么skill

这个需求有两个层面：

1. **对比实际效果 vs 真实文档** → 这是评估/分析
2. **给出优化方向** → 需要基于分析结果提出改进方案

匹配的 skill：

|Skill|适合点|不适合点|
|---|---|---|
|`/qa`|能检测入库质量问题|偏功能测试，不做效果对比|
|`/retrieval-next`|按 retrieval 层级验证索引质量|偏单层优化，不做全面差距分析|
|`/design-shotgun`|多方案比选|需要先有方案才能比|

**推荐：先 `/qa` 做检测，再 `/design-shotgun` 出优化方案**

```
/qa (检测差距) → /design-shotgun (优化方向比选)
```

**具体流程**：

|步骤|Skill|做什么|
|---|---|---|
|1. 差距分析|`/qa`|系统化检测入库效果：原文 vs 索引内容，找出信息丢失、格式变形、表格错误等问题|
|2. 优化方向|`/design-shotgun`|基于 qa 发现的问题，生成多组优化方案（chunk 策略、OCR 参数、解析方式等）并横向比选|

**为什么不能一步到位**：

- 对比差距需要**实际跑数据**（原文和索引结果的 diff）
- 优化方向需要**基于差距结果**才能提出
- 两个 skill 各自擅长的阶段不同

如果你只想快速看一下当前入库效果、不需要详细报告，也可以直接用 `/retrieval-next`，它会按层级检查并给出当前层的问题和优化建议。

.env#63-70

RAG_TOP_K_DEFAULT=5 RAG_RERANK_TOP_N=3 RAG_CHUNK_SIZE_CHARS=400 RAG_CHUNK_OVERLAP_CHARS=50 RAG_CHUNK_MIN_CHARS=200 RAG_UPLOAD_MAX_FILE_SIZE_BYTES=104857600 RAG_LLM_MAX_PROMPT_TOKENS=4000 # prompt 软预算，避免大上下文把回答 token 挤光 / Prompt soft budget RAG_LLM_RESERVED_COMPLETION_TOKENS=1400 #我想让它评价一下这个正确性用什么skill

|||
|---|---|
|||
|||
|||

这是**配置审查**场景，想知道当前参数值是否合理。

|Skill|适用场景|
|---|---|
|`/review-code`|现有配置/代码质量审查 → 评价正确性、合理性|
|`/design-shotgun`|参数调优决策 → 改成什么值|

**推荐：直接用 `/review-code`**

它会从最佳实践角度审查这些参数：

- chunk 大小 400 是否合理
- overlap 50 是否够用
- top_k=5, top_n=3 的比例是否恰当
- prompt/completion token 分配是否均衡

**注意**：如果想**对比不同参数值的效果**，那走 `/design-shotgun` 生成多组方案比选。

你现在选的是"评价正确性"，所以 `/review-code` 最匹配。

为什么不用多个agent讨论的那个skill

好问题！你指的是 `team-brainstorm`（多角色讨论）吧？

**为什么不用**：

|维度|你的场景|team-brainstorm 适用场景|
|---|---|---|
|**问题性质**|技术事实判断（这些参数值是否合理）|主观决策（架构方向、产品设计、资源分配）|
|**需要几个视角**|单一技术视角即可|需要多角色（架构师+PM+设计师+安全）碰撞|
|**输出目标**|评价正确性、给出建议|达成共识、产出综合决策|
|**复杂度**|明确、聚焦|模糊、多利益相关方|

**用 team-brainstorm 会过度设计**：

```
你的场景：chunk=400 合理吗？→ 一个 agent 就能回答

team-brainstorm 场景：
- 架构师：chunk 应该大，保留上下文
- 性能工程师：chunk 应该小，检索更快
- 产品经理：用户感知更重要
- 需要讨论达成共识...

结论：单一技术审查 → review-code
      多角色决策冲突 → team-brainstorm
```

简单说：**没人跟你争论的时候，不需要开会**。



   gstack-qa-only
1. spec-generator 或直接按模板写 spec
2. workflow-plan 产出实施计划
3. plan-eng-review 做一次工程审查





## codex
我按这个仓库的真实入口核过了，先给你一个准确版：

**仓库本地命令**

- /retrieval-next：接手并继续 retrieval 工作，先判断当前在 baseline / supplemental / chunk / router / rerank 哪一层，再尽快推进到最小实现或最小验证
- /retrieval-plan：retrieval 的默认 planning 入口
- /plan：非 retrieval 的中大改动，或 retrieval 已升级后的完整规划
- /review：对代码改动做评审

**全局可用 skills**

- receiving-code-review：收到 review 意见后，先核实再决定是否改
- team-brainstorm：多角色并行讨论、发散方案
- brainstorm：单线程方案探索
- qa / qa-only：功能测试与报告
- review：通用预落地代码审查 skill
- benchmark：性能/回归验证

**怎么选**

- 收到别人 review 意见，要判断“他说得对不对、要不要改”：receiving-code-review
- 想主动审代码、审 diff、找风险：仓库本地 /review，或全局 review
- retrieval 任务刚开始、边界还没收敛：/retrieval-plan
- retrieval 任务是“继续上一刀 / 按 handoff 接着做 / 先判断现在该推进哪层”：/retrieval-next
- 非 retrieval 的中大改动、新功能、多文件联动：/plan
- 想多方案讨论，比如“要不要换 embedding、要不要重构 ingestion 架构”：brainstorm
- 想开多角色会审、并行出观点：team-brainstorm
- 想验证新功能是否真的能用，并顺手修问题：qa
- 只想拿测试报告，不自动修：qa-only
- 想比较参数或方案改动后的性能：benchmark

**放到你前面那些具体问题里**

- “检验新加的功能”：qa
- “只看测试报告”：qa-only
- “收到代码审查意见”：receiving-code-review
- “让我评价 .env 这些 RAG 参数合不合理”：/review 或 review
- “我想知道现在入库索引有没有问题”：先 qa；如果确认是 retrieval 分层问题，再进 /retrieval-plan 或 /retrieval-next
- “我想优化文档入库检索”：默认先 /retrieval-plan，不是先 team-brainstorm
- “继续上一次 retrieval 优化”：/retrieval-next
- “要不要改上下文大小、并发数、chunk 大小”：如果是方案探索，用 brainstorm；如果是多角色会审，用 team-brainstorm；如果已经决定要改并进入 retrieval 主线，就回到 /retrieval-plan

**一个容易混淆但很重要的点**

- team-brainstorm 适合“要不要这么做”
- /retrieval-plan / /retrieval-next 适合“已经决定在 retrieval 主线上推进，下一步该怎么做”
- receiving-code-review 适合“别人提了修改意见，我先验证再决定采不采纳”
- /review 适合“我来审这次代码改动本身”

如果你愿意，我下一条可以直接给你一版“这个仓库的一页式速查表”，写成你以后可以直接照着选的清单。