
把 OCR 拆成独立 OCR service，让 Celery 只调 HTTP可以吗

12:45 PM

可以，而且**在架构上是成立的**。如果你的目标是把 GPU、PaddleOCR、CUDA 依赖从当前 ingest worker 里剥出去，这条路很合理。

但我会建议我们拆成“**独立 OCR service + Celery 编排**”，不是把所有 ingest 都变成纯 HTTP 服务。

**更稳的做法**

- Celery 继续负责任务编排、重试、状态推进、队列隔离。
- 独立一个 ocr-service，只负责：
    - 图片 OCR
    - 扫描 PDF OCR
    - DOCX 内嵌图片 OCR
- ingest worker 在需要 OCR 时，发 HTTP 给 ocr-service，拿结果后继续后面的 chunking / embedding / indexing。

也就是：

`API -> Celery ingest worker -> HTTP -> OCR service(GPU) <- OCR result <-`

**这样做的好处**

- GPU 依赖和 API/普通 worker 解耦。
- OCR 可以单独扩容、单独部署到有 GPU 的机器。
- PaddleOCR 模型可以在 OCR service 内常驻，不用跟 Celery 多进程生命周期打架。
- CPU 解析和 GPU OCR 天然分层，前面我们讨论的“原文解析被 OCR 堵住”会更容易解决。

**但代价也很真实**

- 文件怎么给 OCR service：
    - 直接传文件
    - 传共享存储路径
    - 传预签名下载地址
- 大 PDF/DOCX 走 HTTP 会增加传输成本。
- 需要新的一层超时、重试、幂等、认证、错误映射。
- 进度条会变成两段状态：
    - Celery 本地阶段
    - OCR service 远端阶段
- 如果 OCR service 挂了，Celery 这边要能优雅回退或重试。

**对你这个仓库，我的建议是**

1. 可以拆。
2. 但**优先拆“OCR 能力”**，不要一上来把整个 ingest 都服务化。
3. Celery 仍保留，因为你现在已经有 job 状态、队列、stuck job 恢复、folder delete 隔离这些机制，直接丢掉会很亏。
4. 如果 OCR service 和 worker 还在同一台机器，只是进程隔离，收益主要是“依赖和生命周期隔离”，不是网络性能提升。
5. 如果 OCR service 放到专门 GPU 宿主机，这个收益就会非常明显。

**一句话判断**

- 作为“解耦 GPU OCR”的方向：**值得做**。
- 作为“立刻解决当前卡顿”的最短路径：**不如先做 OCR queue/worker 拆分和模型复用**。
- 作为中期架构：**比只靠 Celery 单体 worker 更干净**。

如果你愿意，我下一步可以直接给你写一份“OCR service 拆分 plan”，把接口、文件传递方式、Celery 状态机和迁移步骤写清楚。

12:46 PM

难度大吗

12:46 PM

难度不小，但也不是那种“推倒重来级别”。

如果按你这个仓库现状看，我会这么分：

**只把 OCR 抽成独立 HTTP service**  
难度：中等偏上

要处理的点主要有：

- 定义 OCR API：图片、PDF、DOCX 嵌图分别怎么调
- 文件传递：传二进制、传共享路径、还是传对象存储地址
- Celery worker 调 HTTP 后，怎么把结果并回现有 ingest 流程
- 超时、重试、幂等、错误码映射
- OCR 进度怎么回传给现有 job 状态

这个量级大概是“1 条新服务链路 + 现有 ingest 接一下”，不算小，但边界是清楚的。

**如果想顺手把当前 ingest 也重构成更干净的两阶段/多阶段**  
难度：高

因为会碰到：

- 现有 run_ingest_job 是一条长链路，状态机要改
- stale job 恢复逻辑要改
- 测试面很大
- 前端进度、ops 队列、worker health 都要一起对齐

所以真正难的不是“写个 OCR HTTP 服务”，而是**把它平滑接入现在这套 job/queue/status/retry 体系**。

我给你一个更实际的判断：

1. 作为短期止血方案：不划算  
    先做 OCR queue/worker 拆分 + 模型复用 更快见效。
    
2. 作为中期架构升级：划算  
    如果你们后面还会继续加 GPU 能力、扩 OCR 吞吐、把 GPU 放独立机器，这个方向是对的。
    
3. 纯实现难度估算
    
    - 出一个可跑 MVP：中等
    - 接进现有生产链路并稳住：中等偏上到高