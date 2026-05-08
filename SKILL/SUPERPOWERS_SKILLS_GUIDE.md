# Superpowers Skills 使用指南


> Superpowers 是一个完整的软件开发工作流插件，通过系统提示强制 agent 遵循设计→计划→实现→审查的流程。

> 所有 skill 通过 `using-superpowers` 入口 skill 强制检查：agent 在每次回复前必须扫描 skill 列表，有 1% 可能性就必须调用，你不需要手动触发。

> 核心理念：先设计再编码，先测试再实现，先找根因再修复。


---


## 一、工作流概览


Superpowers 的核心是一个从构思到发布的完整流程：


```
构思 → 头脑风暴 → 写计划 → 执行计划 → 代码审查 → 完成分支
```

每个环节都有对应的 skill 被 agent 主动调用，确保质量和纪律。


---


## 二、构思与设计


### brainstorming — 头脑风暴（强制入口）


**什么时候触发：** 任何需要创建功能、构建组件、添加行为的请求。这是所有创造性工作的**强制入口**。


**会直接写代码吗：** 不会。必须先完成设计并得到你的批准。


**工作流程：**

1. 探索项目上下文（检查文件、文档、最近提交）
2. 如果涉及视觉问题，提供浏览器可视化伴侣
3. 逐个提问澄清需求（每次只问一个问题）
4. 提出 2-3 个方案及权衡分析
5. 分段呈现设计，每段确认
6. 写设计文档并提交 git
7. 自检设计文档（去占位符、检查一致性）
8. 你审阅设计文档
9. 转入 writing-plans 编写实现计划


**铁律：** 没有经过设计审批 = 不允许写任何代码。"太简单不需要设计"是最常见的反面教材。


**一个关键原则：** YAGNI（You Aren't Gonna Need It）——无情砍掉不必要的功能。


---


## 三、计划编写


### writing-plans — 编写实现计划


**什么时候触发：** brainstorming 完成后自动转入，或你有现成的 spec/需求文档需要分解任务。


**会改代码吗：** 不会。只生成计划文档。


**计划特点：**

- 每个步骤是一个 2-5 分钟的原子操作
- 包含完整代码、精确命令、预期输出
- 遵循 TDD：先写测试 → 确认失败 → 最小实现 → 确认通过 → 提交
- 绝不允许占位符（TBD、TODO、"添加适当错误处理"）


**计划结构：**

```
Task N: [组件名]
  Files: 精确文件路径 + 行号
  Step 1: 写失败的测试（含完整代码）
  Step 2: 运行确认失败（含命令 + 预期输出）
  Step 3: 最小实现（含完整代码）
  Step 4: 运行确认通过（含命令 + 预期输出）
  Step 5: 提交（含 git 命令）
```


**完成后选择执行方式：**

1. **Subagent-Driven（推荐）** — 每个 task 派一个新 agent，任务间自动审查
2. **Inline Execution** — 在当前会话中顺序执行


---


## 四、执行与实现


### subagent-driven-development — 子 agent 驱动开发（推荐）


**什么时候触发：** 选择 Subagent-Driven 方式执行计划时。


**会改代码吗：** 会。这是主要的编码执行方式。


**核心模式：** 每个 task 派一个全新的 agent + 两阶段审查


**流程：**

1. 读取计划，提取所有 task，创建 TodoWrite
2. 对每个 task：
   - 派遣实现 agent（含完整 task 文本 + 上下文）
   - 实现 agent 完成编码、测试、提交、自检
   - 派遣 spec 合规审查 agent → 不合规就修复 → 重新审查
   - 派遣代码质量审查 agent → 有问题就修复 → 重新审查
   - 标记 task 完成
3. 所有 task 完成后，派遣最终代码审查
4. 转入 finishing-a-development-branch


**模型选择策略：**

| 任务类型 | 模型选择 |
|---------|---------|
| 机械实现（1-2 文件，完整 spec） | 快速/便宜模型 |
| 集成任务（多文件协调） | 标准模型 |
| 架构/设计/审查 | 最强模型 |


**Agent 状态处理：**

| 状态 | 处理方式 |
|-----|---------|
| DONE | 进入审查 |
| DONE_WITH_CONCERNS | 检查疑虑后进入审查 |
| NEEDS_CONTEXT | 补充上下文重新派遣 |
| BLOCKED | 评估阻碍，换模型/拆任务/升级人类 |


### executing-plans — 顺序执行计划


**什么时候触发：** 选择 Inline Execution 方式，或在无 subagent 支持的平台。


**会改代码吗：** 会。按计划逐步执行。


**流程：**

1. 加载并审查计划（有问题先提出）
2. 逐个执行 task，每步验证
3. 全部完成后转入 finishing-a-development-branch


**与 Subagent-Driven 的区别：**

| 维度 | Subagent-Driven | Executing Plans |
|-----|----------------|-----------------|
| 执行环境 | 每个 task 新 agent | 当前会话 |
| 上下文 | 干净隔离 | 共享可能污染 |
| 审查 | 每个任务两阶段审查 | 每 3 个 task 审查一次 |
| 速度 | 更快迭代 | 更适合简单计划 |


---


## 五、测试驱动开发


### test-driven-development — TDD 铁律


**什么时候触发：** 任何功能实现或 bug 修复，在写实现代码之前。


**会改代码吗：** 这是编码纪律，不是独立工具。


**核心铁律：**

```
没有失败的测试 = 不允许写生产代码
```


**Red-Green-Refactor 循环：**

1. **RED** — 写一个失败的测试
   - 一个行为、清晰命名、真实代码（尽量避免 mock）
2. **Verify RED** — 运行，确认失败原因正确（功能缺失，不是拼写错误）
3. **GREEN** — 写最少的代码让测试通过
   - 不添加功能、不重构、不"顺便改进"
4. **Verify GREEN** — 运行，确认通过且其他测试不受影响
5. **REFACTOR** — 清理（去重、改善命名、提取辅助函数）
6. 重复


**先写代码再补测试的问题：**

| 借口 | 现实 |
|-----|------|
| "太简单不用测" | 简单代码也会出 bug，测试只要 30 秒 |
| "先写代码后面补测" | 后补的测试立即通过，什么都没验证 |
| "已经手动测试过了" | 手动测试不可复现、无法回归 |
| "删掉 X 小时的工作太浪费" | 沉没成本谬误，不可信的代码才是真正的浪费 |
| "TDD 太教条" | TDD 比调试更快，这才是务实 |


**违反信号（看到就重做）：**

- 测试立即通过（说明测的不是新功能）
- 先写实现再补测试
- "就这一次跳过"
- "我保持精神，不拘泥于形式"


---


## 六、调试


### systematic-debugging — 系统化调试


**什么时候触发：** 遇到任何 bug、测试失败、异常行为、性能问题、构建失败。


**会改代码吗：** 会，但必须先完成根因调查。


**核心铁律：**

```
没有根因调查 = 不允许修复
```


**四阶段流程：**


**Phase 1：根因调查**

1. 仔细阅读错误信息（不要跳过堆栈跟踪）
2. 稳定复现（无法复现 → 继续收集数据，不要猜）
3. 检查最近变更（git diff、最近提交）
4. 多组件系统：在每个组件边界加诊断日志
5. 追踪数据流（从出错值反推源头）


**Phase 2：模式分析**

1. 找到相似的正常工作的代码
2. 对比差异（不要假设"这不重要"）
3. 理解依赖和假设


**Phase 3：假设与验证**

1. 形成单一假设（"我认为 X 是根因，因为 Y"）
2. 最小改动验证
3. 每次只改一个变量


**Phase 4：实现修复**

1. 先写复现测试
2. 单一修复（不要"顺便改"）
3. 验证修复有效

**如果修复失败：**
- < 3 次 → 回到 Phase 1
- >= 3 次 → **停下来质疑架构**，不要继续修


**危险信号（看到就停下来）：**

- "先快速修一下，回头再查"
- "多改几个试试看"
- "跳过测试，手动验证"
- 已经试了 2+ 次还在"再来一次"


---


## 七、代码审查


### requesting-code-review — 请求代码审查


**什么时候触发：** 完成一个 task、完成重大功能、合并到 main 之前。


**会改代码吗：** 主要是出审查意见，可能小修。


**触发时机：**

- **必须：** subagent-driven 每个 task 完成后、合并 main 前
- **推荐：** 卡住时（新鲜视角）、重构前（基线检查）、修完复杂 bug 后


**流程：**

1. 获取 git SHA 范围
2. 派遣 code-reviewer subagent（独立于你的会话上下文）
3. 根据反馈行动：
   - Critical → 立即修复
   - Important → 继续前修复
   - Minor → 记录后续处理


### receiving-code-review — 接收代码审查


**什么时候触发：** 收到审查反馈时，在实施建议之前。


**核心原则：** 验证后再实施，不理解就问，技术正确性 > 社交舒适。


**正确做法：**

```
1. 完整阅读反馈，不要急于反应
2. 用自己的话重述需求（或提问澄清）
3. 对照代码库验证
4. 技术评估是否适用于当前项目
5. 技术性确认或理性反驳
6. 逐项实施，每项测试
```


**禁止的回复：**

- "You're absolutely right!" / "Great point!" （表演式认同）
- "Let me implement that now" （未验证就实施）
- 收到 6 项反馈，只理解 4 项就开始改（必须先全部澄清）


**什么时候应该反驳：**

- 建议会破坏现有功能
- 审查者缺少完整上下文
- 违反 YAGNI（未使用的功能）
- 与你的架构决策冲突


---


## 八、分支管理


### using-git-worktrees — Git Worktree 隔离


**什么时候触发：** 开始功能开发需要隔离、执行实现计划之前。


**会改代码吗：** 不会。只创建隔离工作空间。


**流程：**

1. 检查已有 worktree 目录（`.worktrees/` > `worktrees/`）
2. 验证目录在 .gitignore 中（不在则自动添加）
3. 创建 worktree + 新分支
4. 自动安装依赖
5. 运行基线测试确认干净状态
6. 报告就绪


### finishing-a-development-branch — 完成开发分支


**什么时候触发：** 所有 task 完成、测试通过后。


**会改代码吗：** 不会。只处理分支合并/PR/清理。


**流程：**

1. 验证测试全部通过（不通过 → 停下来修）
2. 确定基线分支
3. 提供 4 个选项：

| 选项 | 说明 |
|-----|------|
| 1. 本地合并 | merge 回基线分支，删 feature 分支 |
| 2. 创建 PR | push + `gh pr create` |
| 3. 保留现状 | 分支和工作区都保留 |
| 4. 丢弃工作 | 需要输入 "discard" 确认 |

4. 执行选择 + 清理 worktree


---


## 九、质量保证


### verification-before-completion — 完成前验证


**什么时候触发：** 即将声称工作完成、测试通过、bug 已修复，或准备提交/创建 PR。


**核心铁律：**

```
没有新鲜验证证据 = 不允许声称完成
```


**门控函数：**

```
声明任何状态之前：
1. 识别：什么命令能证明这个声明？
2. 运行：执行完整命令
3. 阅读：完整输出，检查退出码
4. 验证：输出是否确认声明？
   - 否 → 陈述实际状态
   - 是 → 附带证据声明
5. 只有这时才能说"完成了"
```


**常见错误：**

| 声明 | 需要 | 不充分 |
|-----|------|--------|
| 测试通过 | 测试命令输出：0 failures | 之前跑过、"应该通过" |
| Bug 已修 | 复现原始症状：通过 | 改了代码、假设修好了 |
| 构建成功 | 构建命令：exit 0 | linter 通过 |


**危险信号：**

- 使用"应该"、"可能"、"看起来"
- 在验证前表达满意（"Great!", "Perfect!", "Done!"）
- 信任 agent 的成功报告而不独立验证


---


## 十、并行执行


### dispatching-parallel-agents — 并行派遣 agent


**什么时候触发：** 面对 2+ 个独立任务，可以并行处理而不互相影响。


**会改代码吗：** 会。每个 agent 独立修复自己负责的问题。


**适用场景：**

- 3+ 个测试文件因不同根因失败
- 多个子系统独立出问题
- 各问题之间无共享状态


**不适用场景：**

- 失败是相关的（修一个可能修好其他的）
- 需要理解完整系统状态
- Agent 之间会互相干扰（编辑相同文件）


**Prompt 要点：**

| 要素 | 好的做法 | 坏的做法 |
|-----|---------|---------|
| 范围 | "Fix agent-tool-abort.test.ts" | "Fix all the tests" |
| 上下文 | 贴出错误信息和测试名 | "Fix the race condition" |
| 约束 | "Don't change production code" | 无约束 |
| 输出 | "Return summary of root cause" | "Fix it" |


**完成后：**

1. 阅读每个 agent 的总结
2. 检查修复是否有冲突
3. 运行全量测试
4. 抽查 agent 的工作质量


---


## 十一、Skill 编写（元技能）


### writing-skills — 编写新 Skill


**什么时候触发：** 需要创建新 skill、编辑已有 skill、验证 skill 是否有效。


**核心原则：** 编写 skill 本身就是 TDD — 先用 subagent 压力测试验证问题存在，再写 skill 解决问题，再验证问题消失。


**Skill 类型：**

| 类型 | 说明 | 示例 |
|-----|------|------|
| Technique | 有步骤的具体方法 | condition-based-waiting |
| Pattern | 思维模型 | flatten-with-flags |
| Reference | API 文档、语法指南 | office docs |


**SKILL.md 结构：**

```yaml
---
name: skill-name-with-hyphens
description: Use when [触发条件和症状]
---

# Skill Name
## Overview → What + Core principle
## When to Use → 触发条件
## Core Pattern → Before/After 对比
## Quick Reference → 扫描用表格
## Implementation → 内联代码或链接
## Common Mistakes → 错误 + 修复
```


**Description 写法铁律：**

- 只写"什么时候用"，不写"做了什么"
- 原因：如果 description 总结了流程，agent 可能直接跟着 description 走而不读完整 skill


**TDD 流程：**

```
RED: 无 skill 时跑压力场景 → 记录 agent 的违规行为和借口
GREEN: 写最小 skill 解决这些具体违规 → 验证 agent 合规
REFACTOR: 找新借口 → 堵漏洞 → 重新验证
```


---


## 快速参考：完整工作流链


```
新功能开发：
brainstorming → writing-plans → using-git-worktrees → subagent-driven-development → requesting-code-review → verification-before-completion → finishing-a-development-branch

遇到 Bug：
systematic-debugging → test-driven-development → verification-before-completion

多个独立问题：
dispatching-parallel-agents（每个 agent 内部走 debugging + TDD）
```


---


## 快速参考：什么时候用什么


| 场景 | Agent 会调用的 Skill |
|------|-----------------|
| "帮我加个功能" | brainstorming → writing-plans → subagent-driven-development |
| "修个 bug" | systematic-debugging → test-driven-development |
| "测试失败了" | systematic-debugging |
| "帮我建个计划" | writing-plans |
| "执行这个计划" | subagent-driven-development 或 executing-plans |
| "审查这段代码" | requesting-code-review → receiving-code-review |
| "完成了，要合并" | verification-before-completion → finishing-a-development-branch |
| "3 个模块同时出问题" | dispatching-parallel-agents |
| "要写个新 skill" | writing-skills |


---


## 核心铁律速记


| Skill | 铁律 |
|-------|------|
| brainstorming | 没有设计审批 = 不写代码 |
| test-driven-development | 没有失败测试 = 不写生产代码 |
| systematic-debugging | 没有根因 = 不修复 |
| verification-before-completion | 没有验证证据 = 不声称完成 |
| writing-plans | 不允许占位符，每步完整代码 |
| writing-skills | 没有压力测试 = 不发布 skill |


---


## 与 GStack Skills 的关系


Superpowers 和 GStack Skills 是**互补**的两套系统：


| 维度   | Superpowers   | GStack Skills |
| ---- | ------------- | ------------- |
| 定位   | 开发工作流纪律       | 实用工具箱         |
| 触发方式 | Agent 被系统提示强制主动调用 | 用户输入 `/skill名` 手动触发 |
| 核心关注 | 流程正确性         | 任务完成          |
| 适用场景 | 从构思到发布的完整开发周期 | QA、审查、部署等具体任务 |


**典型协作模式：**

```
Superpowers 负责流程纪律（设计→计划→TDD→审查）
GStack 负责具体执行（/qa 测试、/review 审查、/ship 发布）
```


两者不冲突，可以同时使用。Superpowers 的 TDD/调试纪律让 GStack 的工具产生更好的结果。


---


## 最小可用子集


如果不想记全部 skill，记住这 5 个核心纪律就够覆盖 90% 的质量提升：


| 排名  | 纪律           | 一句话说明                    |     |
| --- | ------------ | ------------------------ | --- |
| 1   | 先设计再编码       | brainstorming 的核心        |     |
| 2   | 先测试再实现       | TDD 的核心                  |     |
| 3   | 先找根因再修复      | systematic-debugging 的核心 |     |
| 4   | 先验证再声称完成     | verification 的核心         |     |
| 5   | 每个 task 都要审查 | code-review 的核心          |     |
