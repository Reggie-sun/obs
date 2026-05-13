所以我现在的eval可以支持我直接用sample optimization去用真实query来优化是吗

10:50 AM

可以。

你现在这套 eval 已经支持你直接拿真实 query 走 sample-optimization，而且这正是它现在比较适合的用法之一。关键点是：**真实 query 不会直接污染最终测试集**，而是先进入观察/分诊层。

实际流程会是这样：

1. 你直接给真实 query
    
    - 例如：优化这个真实用户 query：xxx
2. sample-optimization 先走快路径
    
    - 看现有 queue 里有没有能对应上的 case
    - 没有的话，把它当成新的单目标 residual 线索
3. 然后它会判断该落哪层
    
    - 如果更像 live 现象，就先落到 optimization case / observational lane
    - 如果更像已有样本的延伸，就挂到现有 sample/case
    - 如果需要从真实库补上下文，就再接 library-sample-intake
4. 只有证据够了，才会进实现面
    
    - cleaning
    - chunk
    - retrieval
    - 或停在 governance / snapshot / alignment
5. 最后仍然用 frozen gate 验证没回退
    
    - 所以“真实 query 驱动优化”不等于“拿真实 query 直接当最终验收标准”

这也是为什么你那些其它 eval 文件是必要的：

- 它们让真实 query 可以先被安全接住
- 但不会直接改坏 frozen/verified 那套最终验证

所以答案是：

**是的，你现在可以直接用 sample-optimization 去吃真实 query 做优化；只是它默认会先把真实 query 当成观察/分诊输入，而不是直接当最终测试真相。**

你现在最自然的用法就是直接发：

- 优化这个真实用户 query：<query>
- 继续分析这个 query：<query>
- 把这个真实 query 走 sample optimization：<query>

如果你愿意，下一条你直接给我一个真实 query，我就按现在这套流程直接示范一次。

10:50 AM

我有一个问题，为什么你自己找的问题要eval层层选拔，而用户的真实问题就可以直接用sample-optimization优化

3:44 PM

因为这里有两个完全不同的问题：

1. 这个现象值不值得被接住
2. 这个现象能不能直接拿来改代码/改 truth

真实用户问题在第 1 个问题上，优先级很高；但在第 2 个问题上，**并没有特权**。

也就是说，真实用户 query 可以直接进 sample-optimization，指的是：

- 它可以直接作为“优化入口”
- 不需要你先手工整理成 sample_id / case_id
- 系统会优先接住它

但这**不等于**：

- 它可以绕过治理面
- 它可以绕过 evidence / snapshot / alignment
- 它可以绕过 frozen gate
- 它可以直接推动代码改动

真正的区别在于“信号来源的可信度”和“进入流程的门槛”。

**为什么我自己找的问题要层层选拔**

因为我自己找到的问题，很多是这几类：

- rollout 过程里顺手发现的异常
- trace / replay / snapshot 对比里冒出来的残差
- 脚本扫出来的 drift
- 观察面样本的局部波动

这些问题的噪声很大，常常混着：

- corpus drift
- governance 问题
- runtime 不可比
- expected doc 过期
- sample truth 本身不稳

所以它们必须先经过“层层选拔”，不然很容易把假问题修成真改动。

**为什么真实用户问题可以直接进 sample-optimization**

因为真实用户问题不是“噪声来源不明的内部观察”，而是明确的产品信号：

- 用户真的搜不到
- 用户真的觉得结果差
- 用户真的被卡住了

所以它值得被**优先接入 workflow**。  
但它接入以后，还是要继续被判断：

- 这是新 case 吗
- 是 snapshot/governance 问题吗
- 是 cleaning/chunk/retrieval 哪一层
- 值不值得进实现面
- 最后 frozen gate 有没有回退

所以更准确的话应该是：

**真实用户问题可以直接进优化入口，但不能直接进实现出口。**

我建议你把三类信号分开理解：

- frozen/verified
    
    - 最高验证权威
    - 决定能不能放行
- 真实用户 query
    
    - 最高产品优先级
    - 决定现在该看什么
- agent/脚本自己发现的问题
    
    - 最高噪声风险
    - 必须先过分诊和治理筛选

所以不是“用户问题更宽松”，而是：

- 用户问题：更容易被接住
- 内部发现问题：更容易被怀疑
- 但两者最后都不能绕过验证闭环

一句话总结：

**真实用户 query 享有的是 intake 优先权，不是 validation 豁免权。**

如果你愿意，我可以下一条直接给你画一个非常实用的决策表：

- 什么情况下“真实 query 直接进 sample-optimization”
- 什么情况下“先进 library-sample-intake”
- 什么情况下“只能先停在 governance，不准改代码”



  
可以

3:46 PM

Worked for 1m 31s

可以，给你一个最实用的决策表。

**入口规则**

|场景|先用什么|为什么|
|---|---|---|
|用户直接给了一个真实 query，想知道为什么结果差|sample-optimization|这是单目标优化入口，先接住真实问题最合适|
|用户给了真实 query，还明确说“从真实文档库补样本”|library-sample-intake|重点变成 sample seed / case intake，不只是诊断单条 query|
|你已经有 case_id|sample-optimization|直接继续现有 case，最快|
|你已经有 sample_id|sample-optimization|先看是否已有 case，没有再做单目标分诊|
|你没有 query，只想看看当前真实库里还有哪些值得补的样本|library-sample-intake|这是 coverage / seed 发现问题|
|你发现一个脚本扫出来的 residual，但不确定是不是假问题|sample-optimization|先走治理/分诊，不要直接改代码|

**真实 query 进 sample-optimization 后怎么分流**

|条件|去哪层|含义|
|---|---|---|
|能匹配到已有 case|继续该 case|不重建上下文|
|没有现成 case，但现象明确|新建/挂到 optimization case 思路|先变成可追踪问题|
|blocked_on_active_snapshot|停在 governance|先查 runtime / corpus comparability|
|blocked_on_governance|停在 governance|先查 include/candidate/exclude/family|
|needs_evidence|停在证据补齐|先补 replay / alignment / before-after|
|ready_for_<layer>|进入实现面|但只进一个层：cleaning / chunk / retrieval|

**什么时候该先用 library-sample-intake**

|信号|说明|
|---|---|
|你关心的是“这个 query 对应的真实文档家族有没有被 sample 覆盖”|先 intake|
|你需要从 active portal/library 自动找候选文档/seed|先 intake|
|你想补 rolling/candidate seed，而不是立刻修行为|先 intake|
|你怀疑问题不是单条 query，而是某个文档家族整体没进入 eval 观察面|先 intake|

**什么时候只能停在 governance，不能改代码**

|信号|为什么不能直接改|
|---|---|
|expected_doc 不在 active snapshot|这不是排序问题，先解决 comparability|
|top1/topk 命中的都是新 family 或漂移 family|先看 governance，不然会误修|
|runtime 身份不清楚|replay 结果不可信|
|当前 query 对应的是 broad/overview 问法，但 checked-in truth 还没有稳定 replacement anchor|不能安全 rebind|
|只能靠往 retrieval_service.py 继续叠特判才能修好|这是 refactor 信号，不是普通样本优化|

**什么时候可以进实现面**

|条件|可以做什么|
|---|---|
|doc wrong 且 evidence 指向治理已对齐后仍有问题|进 retrieval 或 chunk 分诊|
|doc right but chunk wrong|优先 chunk|
|噪声块压过正确块|优先 cleaning|
|正确块已在候选里，但排序/branch policy 没选中|优先 retrieval|

**最后的硬规则**

|规则|解释|
|---|---|
|真实用户 query 可以直接进入口|有 intake 优先权|
|真实用户 query 不能直接跳过证据链|没有 validation 豁免权|
|compact summary 只能导航|不能替代 full report / diff / alignment|
|最终放行仍看 frozen/verified gate|防止修一个用户问题打坏主线|

**你现在最简单的使用法**

- 想直接分析一个真实问题：
    
    - 优化这个真实用户 query：<query>
- 想围绕这个问题补真实库样本：
    
    - 从真实文档库围绕这个 query 找样本：<query>
- 想继续一个已有目标：
    
    - 优化 case_id=<id>
    - 优化 sample_id=<id>