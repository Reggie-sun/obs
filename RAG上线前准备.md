
**最小 SOP**

每次准备上线 retrieval 相关改动时，只按这套最小流程走：

1. 确认本地运行面是对的
    
    - 本地 API 应该是你当前要验证的实例，通常是 http://127.0.0.1:8020
    - 如果这次依赖 branch runtime，先看 .branch-runtime.local
2. 先跑主 gate
    

`make verify-retrieval-release`

3. 如果要留 artifact，改跑这条

`make verify-retrieval-release \ OUTPUT=eval/generated/retrieval_release_verification_latest.json \ PROGRESS_LOG=eval/generated/retrieval_release_verification_latest.log`

4. 看结果，只按这个判断
    
    - passed: 可以进入上线决策
    - failed_checks: 不上线，先修
    - timed_out_checks: 不上线，先查运行面或性能
    - wrapper_failed_before_verification: 不上线，先修环境/入口
5. 如果主 gate 失败，按失败类型分流
    
    - strict release gate 失败：先看 checked-in truth / residual / frozen comparability
    - retrieval_eval 失败：先看 live eval 结果
    - retrieval_eval_diff 显示 NOT COMPARABLE：直接停，不要把指标差异当回退
    - retrieval_eval_threshold 失败：当真实质量红灯处理
6. 只有满足这几个条件才放行
    
    - make verify-retrieval-release 绿
    - 没有 comparability blocker
    - 没有 block_release residual
    - 如果涉及 promotion，仍满足 verified_for_rolling + rolling_metrics

**不要这样做**

- 不要只跑 make eval-retrieval 就上线
- 不要把 generate_portal_library_retrieval_samples.py --report-checked-in-diff --compact-summary 当治理真源，它只是 advisory preview
- 不要因为“样本跑绿一次”就升 frozen

**最实用的上线口径**

- 日常上线前：跑 make verify-retrieval-release
- 需要留证据：带上 OUTPUT 和 PROGRESS_LOG
- 只要不是 passed，默认不放行