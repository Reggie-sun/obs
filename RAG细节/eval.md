
![[Pasted image 20260518080510.png]]
![[Pasted image 20260518110432.png]]![[Pasted image 20260518080544.png]]
![[Pasted image 20260518080609.png]]![[Pasted image 20260518080623.png|684![[Pasted image 20260518080647.png]]![[Pasted image 20260518080649.png]]



先接你刚才那 3 个点。

**1. retrieval_samples.yaml 里每个字段在表达什么**

它本质上是“检索真相样本库”。每一条样本都在说：

- 用户会怎么问
- 系统至少应该打到哪份文档
- 理想命中的粒度是什么
- supplemental 该不该触发
- 这条样本属于哪条验证轨道

按重要性看，核心字段通常是这些：

- id
    - 样本唯一标识。后续所有 case、evidence、promotion history 都靠它引用。
- query
    - 实际拿去打 /retrieval/search 的查询。
- expected_doc_ids
    - 期望命中的文档集合。
    - 第一个 doc 最重要，因为 top1_accuracy 是严格按 expected_doc_ids[0] 算的 scripts/eval_retrieval.py (line 919)。
- expected_granularity
    - 期望答案粒度，常见是 fine / coarse。
    - 它不直接决定 pass/fail，但会影响你怎么看这条命中是不是“答得太粗”。
- expected_chunk_type
    - 期望最理想的命中块类型，比如 clause / section_summary / doc_summary / table。
    - 当前这部分是启发式诊断，不是 release formal gate scripts/eval_retrieval.py (line 984)。
- supplemental_expected
    - 这条样本按设计是否应该触发 supplemental。
    - 它决定 supplemental precision/recall 怎么算。
- requester_department_id
    - 逻辑部门视角标签。
    - 不是 API 字段本身，而是 eval 在切换 auth profile 时用的业务视角标签，README 里专门解释了这一点 eval/README.md (line 217)。
- track
    - 最关键的治理字段之一。
    - 常见是 frozen / rolling / candidate。
    - frozen 更像发布真相。
    - rolling 更像现网观察样本。
    - candidate 更像待观察、待确认的种子。
- status
    - 常见是 verified / unverified。
    - release gate 只认可 frozen + verified eval/retrieval_release_gate_policy.yaml (line 2)。
- promotion_state
    - 样本在治理生命周期里的位置，比如 auto_generated、verified_for_rolling。
    - 这决定它能不能进一步晋升。
- source_surface
    - 这条 query 是怎么来的，比如 portal_library、question_bank_probe。
    - 这个字段很重要，因为同一个 doc 的“标题 seed”和“真实问法 seed”不是一回事。
- coverage_tier / coverage_reason
    - 这是为什么要保留这条样本。
    - 比如它是 baseline、deep probe、question-bank probe，还是 candidate holdout。

一句话说，retrieval_samples.yaml 不是“问题列表”，而是“检索契约 + 治理元数据”。

**2. 一次 eval_retrieval 输出该怎么读**

不要先看总分，先按这个顺序看：

1. 可比吗
2. Recall 还在吗
3. Top1 退没退
4. supplemental 是不是乱了
5. chunk / context / term coverage 这些诊断项说明了什么

核心指标解释：

- top1_accuracy
    - Top1 是否严格等于 expected_doc_ids[0] scripts/eval_retrieval.py (line 965)
    - 这是“最理想命中”的指标。
- top1_partial_hit_rate
    - Top1 命中了 expected family 里的别的 doc，但不是首选 doc scripts/eval_retrieval.py (line 969)
    - 这通常不是彻底失败，但说明 canonical / family / ranking 可能有问题。
- topk_recall
    - TopK 里只要命中任一期望 doc 就算成功 scripts/eval_retrieval.py (line 975)
    - 这是“系统还有没有把正确文档找出来”的底线指标。
- expected_doc_coverage_avg
    - 如果 expected_doc_ids 有多个，这个指标看覆盖了几个 scripts/eval_retrieval.py (line 979)
- supplemental_precision / recall
    - 不是在看“答得好不好”，而是在看“该不该触发 supplemental 时触发得对不对” scripts/eval_retrieval.py (line 1219)
- chunk_type_hit_rate
    - 只是启发式诊断，不要把它当正式 gate scripts/eval_retrieval.py (line 1230)
- parent_context_present_rate
    - 看 clause 命中时，是否还能带到必要的节级/文档级上下文 scripts/eval_retrieval.py (line 1240)
- term_coverage_avg
    - 对某些样本，检查关键术语有没有真的出现在结果文本里 scripts/eval_retrieval.py (line 1245)

最容易犯的误读是：

- topk_recall 很高，就以为没问题
    - 不对。可能 recall 在，但 top1 一直偏到同 family 错文档。
- top1_accuracy 掉了，就直接改 rerank
    - 不对。先确认是不是 active snapshot 变了，或者 expected doc 已不在 active corpus。
- supplemental_precision 差，就调阈值
    - 不对。README 明确说，ACL seed 没应用时，supplemental 指标本身就不可信 eval/README.md (line 217)。

所以一份 eval 报告正确读法不是“分数高低”，而是“退化发生在召回、排序、治理、上下文还是补答门控”。

**3. optimization_cases.yaml 为什么像事故分诊系统**

因为它干的事情跟 bug triage 很像：

- 先收事故
- 再定性
- 再分流到正确责任层
- 最后决定是不是 blocking

它不是再算一遍 eval 分数，而是把“失败样本”变成一个可执行判断对象。

一条 case 通常回答 5 个问题：

- 这条问题归哪个层负责
    - cleaning / ocr / chunk_representation / index_governance / retrieval
- 它现在处于什么状态
    - draft / observing / parked / promoted ...
- 它引用了哪些样本和证据
    - sample_refs + evidence
- 它跟当前 active snapshot / governance 对齐了吗
    - alignment_reason
- 它下一步应该做什么
    - triage_outcome + next_action

最关键的逻辑在 scripts/eval_optimization_cases.py (line 404) 之后：

- 先把样本里引用的 expected_doc_ids 跟 active snapshot 对齐 scripts/eval_optimization_cases.py (line 423)
- 再看 replay 命中的 doc 当前在 governance 中是什么状态 scripts/eval_optimization_cases.py (line 453)
- 然后产出 alignment_reason
    - expected_docs_missing_from_active_snapshot
    - replay_active_docs_missing_from_checked_in_snapshot
    - governance_attention_required
    - aligned_with_active_snapshot scripts/eval_optimization_cases.py (line 462)
- 最后再转成 triage_outcome
    - blocked_on_active_snapshot
    - blocked_on_governance
    - needs_evidence
    - ready_for_<layer> scripts/eval_optimization_cases.py (line 492)

为什么这很重要？

因为没有这层时，你会犯一种典型错误：

- 样本 fail 了
- 你直接改 retrieval/rerank
- 结果真正的问题其实是：
    - expected doc 早就不在 active snapshot
    - 或 governance 把它标成 candidate/exclude
    - 或 checked-in snapshot 跟 live runtime 已漂移

那你修再多算法，都是修错层。

**为什么 release gate 这么保守**

发布规则其实就一句话：

verified frozen surfaces are the only release pass/fail source

见 eval/retrieval_release_gate_policy.yaml (line 2)

再展开就是：

- 只有 frozen
- 只有 verified
- 还必须可比
- 还必须无回归
- 如果 triage 是 blocked_on_active_snapshot 或 blocked_on_governance，就不能假装这是“代码回归已修好” eval/retrieval_release_gate_policy.yaml (line 7)

所以这套体系繁琐，不是因为作者喜欢复杂，而是因为它要同时防三种错：

- 把语料变化当成算法回归
- 把治理问题当成排序问题
- 把观察样本当成发布真相

如果你要，我下一条可以直接做一件更实用的事：

1. 我拿一条真实样本，手把手给你读一份 eval_retrieval 单样本结果
2. 或者我拿一个真实 optimization_case，从 sample_refs -> alignment_reason -> triage_outcome 给你完整走一遍