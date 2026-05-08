
• 按两个顺序看就行。

  先看“阅读顺序”：

  1. 先看 RAG_FAST_LAUNCH_PLAN.md 里的 单一路径
  2. 再看 首发范围 和 明确不做
  3. 再看 Launch Bundle 和 Evidence Package
  4. 再看 Go / No-Go 门槛
  5. 最后看 最快上线的执行顺序 和 上线后的优化顺序

  再看“真正执行顺序”：

  6. 先收口：只做 retrieval/search + chat/ask，SOP 不上
  7. 再补齐首发必需行为：有证据才回答、非拒答必须带 citation
  8. 再冻结首发 bundle：代码、配置、语料、索引、ACL、放量范围一起钉死
  9. 再做 evidence package：重跑 eval、人工复核、权限边界检查、rollback 演练
  10. 证据过门槛才上线
  11. 上线后再按 supplemental -> chunk -> router/hybrid -> rerank 继续优化

  一句话就是：先收口，再冻结，再验证，再上线，最后优化。