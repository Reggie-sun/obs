make verify-retrieval-release 的价值，不是“多跑一次测试”，而是**把上线前需要同时成立的几层证据串成一条真正的放行 gate**。

如果只跑 make eval-retrieval，你只能知道：

- 当前这批样本跑出来了一个结果

但你不知道这些更关键的问题：

- 当前 checked-in truth 有没有漂
- release residual 有没有 block_release
- baseline 和当前结果是不是可比
- 这轮结果是不是在吃旧 artifact
- live eval 没跑完时，后面的 diff/threshold 还能不能信

make verify-retrieval-release 就是专门解决这些问题的。它现在串的是这几步：

1. retrieval_state_sync
    - 检查 retrieval 状态文件有没有和当前治理面脱节
    - 防止“代码动了，但状态说明/证据链没更新”
2. file_length_policy
    - 这是 repo 的 control-plane 约束
    - 防止验证链本身继续失控膨胀
3. retrieval_release_gate_bundle
    - 这是 checked-in 治理层
    - 会看 frozen truth、release policy、optimization cases residual
    - 如果这里已经有 block_release，后面根本不该继续假装可以上线
4. retrieval_eval
    - 真正跑 live retrieval eval
    - 这是运行时质量证据
5. retrieval_eval_diff
    - 把 live 结果和 baseline 做对照
    - 不是只看一个绝对分数，而是看和参考线的关系
6. retrieval_eval_threshold
    - 最后一层阈值判断
    - 并且现在会先判断 comparability
    - 如果 NOT COMPARABLE，它会返回 BLOCKED，而不是胡乱把结果当 regression

所以它的本质是：

- eval-retrieval 是“跑一次评测”
- verify-retrieval-release 是“判断这次能不能放行上线”

你可以把它理解成：

- make eval-retrieval
    - 回答：现在跑出来什么结果？
- make verify-retrieval-release
    - 回答：这些结果连同治理证据一起看，够不够支持上线？

为什么上线前必须更依赖后者？

因为 retrieval 最容易出的问题，不只是“分数低”，而是：

- baseline 不可比
- frozen / rolling 治理混了
- promotion 合同没满足
- live eval 失败后还在吃旧结果
- 结果看起来红了，但其实是证据链坏了，不是质量回退

make verify-retrieval-release 的作用，就是把这些“容易误判上线结论”的坑一起挡掉。

一句话总结：

**你需要 make verify-retrieval-release，因为上线要验证的不是“能不能跑出一个 eval 分数”，而是“这次 release 证据链是否完整、可比、可解释、可放行”。**