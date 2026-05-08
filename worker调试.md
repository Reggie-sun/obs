
有必要开共享data吗，我本地自己起数据库测试不就没有这个问题了吗

2:20 PM

有必要，但前提是你还在用现在这套“本地 API + 远端 worker”的混合拓扑。

关键不是“数据库是不是本地”，而是当前这条链路默认把原文件和预览产物放在文件系统里，不只放元数据进 DB。也就是说，API 上传后写的是共享文件目录，worker 处理时也要能读到同一份文件。这就是仓库里一直强调 shared-data 的原因，见 BUG-002-dev-local-api-shared-data-drift.md (line 1) 和 asset_store.py (line 19)。

所以分成两种情况：

- 如果你继续用 make dev-local-api：  
    必须开共享 data。因为本地 API 写文件，远端 worker 读文件；没有同一份共享目录，就会出现“任务入队了，但 worker 看不到源文件”的假性故障。
    
- 如果你真的改成“完整本地 runtime”：  
    可以不依赖这套远端共享 data。但这里的“完整本地”不是只起一个本地数据库，而是至少要保证 API、worker、Redis、Postgres、Qdrant、文件存储都在同一套本地/同机视图里。只把数据库切本地，worker 还在远端，问题不会消失。
    

更直接一点说：

- “本地自己起数据库测试”只能解决一部分元数据链路问题。
- 你这次卡的上传/预览链路，本质更依赖“文件资产在哪、worker 能不能看到”，不只是 DB。

如果你的目标是减少这类 shared-data 拓扑问题，最干净的路线其实是二选一：

1. 全量本地化：API + workers + infra 都本地。
2. 保持混合拓扑：那就老老实实保留 shared-data。

按你现在的工作方式，我的建议是：

- 日常联调、想贴近远端真实数据：继续保留 shared-data。
- 只想隔离调上传逻辑/前端交互：另起一套完整本地 runtime，比“只本地 DB”更靠谱。

BUG-002-dev-local-api-shared-data-drift.mdDocument · MD

Open

2:21 PM

我现在担心多部门一起上传worker队列堆积然后上传解析清洗卡死

2:26 PM

这个担心是成立的，而且按当前实现看，**多部门确实共用同一套 worker 队列**，没有按部门隔离。

现在的关键事实是：

- 上传接口先由 API 同步落文件、写元数据，然后才返回 201 并入队。
- 入队分流只看**文件类型/OCR需求**，不看部门。
- 也就是说不同部门会一起争抢这些共享队列：
    - ingest_bugfix_cpu
    - ingest_bugfix_ocr
    - preview_bugfix

对应代码可以看：

- 队列选择逻辑：celery_app.py (line 30)
- branch runtime 队列名：.branch-runtime.local (line 17)
- worker 拆分：docker-compose.branch-bugfix.yml (line 50)

所以更准确地说：

- **worker 队列堆积**会导致“上传后解析/OCR/清洗/预览变慢”。
- 但它**不应该直接导致上传接口本身卡死**，除非连 API 这段“读文件+写共享存储+创建任务”也一起被打满了。
- 你前面遇到的“网络错误/120s 超时”，我更怀疑是这第二段，本地 dev-local-api 写共享存储时被并发压住，而不是纯粹队列 backlog。

如果你现在最担心“多人一起上传把系统拖死”，我建议优先做这 4 件事：

1. 先把远端 CPU_WORKER_CONCURRENCY 从 4 调到 6 观察。  
    这是最可能直接改善解析/清洗堆积的点。
    
2. 如果 Office 预览也慢，再把 PREVIEW_WORKER_CONCURRENCY 从默认 1 提到 2。  
    只在预览确实排队时动它。
    
3. 保持 OCR worker 保守。  
    WORKER_CONCURRENCY 不建议先猛加，尤其如果它吃 GPU/OCR 资源。
    
4. 前端文件夹上传并发要降。  
    这是保护本地 dev-local-api 的，不然多人多文件夹一起传时，API 自己先被打满。
    

一句话总结：

- **会堆积，且会跨部门互相影响。**
- 但“队列堆积”和“上传接口卡死”是两层问题，前者主要靠调 worker，后者更要控前端上传并发和本地 API 落盘压力。

如果你愿意，我下一步可以直接给你一版“当前这套环境最稳的并发参数建议”，包括 CPU_WORKER_CONCURRENCY / PREVIEW_WORKER_CONCURRENCY / 前端上传并发。

2:26 PM

2颗 4510应该怎么设置

2:32 PM

2 previous messages

如果你说的 4510 是 **Intel Xeon Silver 4510**，那单颗是 **12 核 24 线程**，双路就是 **24 个物理核 / 48 线程**。这个规格我查了 Intel 官方页和 Lenovo PSREF。  
来源：

- Intel 官方规格页: [https://www.intel.com/content/www/us/en/products/sku/236637/intel-xeon-silver-4510-processor-30m-cache-2-40-ghz.html](https://www.intel.com/content/www/us/en/products/sku/236637/intel-xeon-silver-4510-processor-30m-cache-2-40-ghz.html)
- Lenovo PSREF: [https://psrefstuff.lenovo.com/syspool/Sys/PDF/ThinkStation/ThinkStation_PX/ThinkStation_PX_Spec.PDF](https://psrefstuff.lenovo.com/syspool/Sys/PDF/ThinkStation/ThinkStation_PX/ThinkStation_PX_Spec.PDF)

按你这个仓库的 worker 形态，我建议先用这套“稳一点”的值，不要直接吃满 48 线程：

`CPU_WORKER_CONCURRENCY=8 CPU_WORKER_PREFETCH_MULTIPLIER=1 BUGFIX_WORKER_CONCURRENCY=3 WORKER_CONCURRENCY=3 BUGFIX_WORKER_PREFETCH_MULTIPLIER=1 WORKER_PREFETCH_MULTIPLIER=1 PREVIEW_WORKER_CONCURRENCY=2 FOLDER_DELETE_WORKER_CONCURRENCY=2`

我的理由很简单：

- ingest-cpu-worker 是最该加的，先给到 8
- OCR 队列那个 worker 不建议一上来太高，先 3
- preview-worker 给 2 就够了
- 删除队列别抢资源，2 足够
- 全部都保持 prefetch=1，避免长任务把队列“吃死”

如果你的主要压力是：

- txt/docx/pptx/xlsx 这类非 OCR 文档多：可以把 CPU_WORKER_CONCURRENCY 再试到 10
- 扫描版 PDF/OCR 文档多：先别猛加 CPU_WORKER_CONCURRENCY，优先观察 WORKER_CONCURRENCY=3 是否够，再考虑到 4

我不建议你第一步就这么开：

- CPU_WORKER_CONCURRENCY >= 12
- WORKER_CONCURRENCY >= 5
- PREVIEW_WORKER_CONCURRENCY >= 3

因为那样更容易把共享存储、Postgres、Qdrant、OCR service 一起压住，症状会从“排队”变成“整体变卡”。

改完后按你改的项重启：

- 改了 CPU_WORKER_CONCURRENCY：make restart-remote-bugfix-ingest-cpu-worker
- 改了 WORKER_CONCURRENCY：make restart-remote-bugfix-workers
- 改了 PREVIEW_WORKER_CONCURRENCY：make restart-remote-bugfix-preview-worker

再提醒一句：这能缓解“多部门一起上传后解析/清洗/预览排队”，但**不能直接解决你前面那种上传阶段就超时/网络错误**。那个我还是建议同时把前端文件夹上传并发降下来。  
如果你要，我下一条可以直接给你一版“2 颗 4510 对应的本地上传并发 + 远端 worker 并发”整套推荐值。

2:33 PM

我ocr用的是显存为什么不能多开worker

2:36 PM

因为你当前这套其实不是“Celery worker 自己直接吃显存做 OCR”，而是：

- worker 负责跑入库任务
- 真正的 OCR 走 RAG_OCR_TRANSPORT=http
- RAG_OCR_SERVICE_PROVIDER=paddleocr
- RAG_OCR_WORKER_PROVIDER=disabled

也就是：**worker 把 OCR 请求发给单独的 ocr-service**，显存主要耗在 ocr-service，不是耗在 Celery worker 进程本身。你现在的 .branch-runtime.local 就是这个配置。

所以“多开 worker”不等于“更好利用显存”，原因有 4 个：

1. worker 多开，只是同时发更多 OCR 请求给 ocr-service  
    如果 ocr-service 本身处理并发有限，结果只是把请求堆到它前面。
    
2. GPU OCR 通常不是线性扩展  
    多请求同时压进去，常见结果不是 2 倍吞吐，而是：
    
    - 显存抖动
    - CUDA 上下文争抢
    - batch 不稳定
    - 单任务延迟变高
3. OCR 前后还有很多不是 GPU 的步骤  
    比如：
    
    - 文件读取
    - PDF/图片预处理
    - 共享存储读写
    - 元数据写库
    - 后续 chunk / embedding / 索引  
        所以把 worker 开很高，可能先打满的是 CPU、IO、Redis、共享存储，不是显卡。
4. 你当前最像瓶颈的地方未必在 OCR  
    你前面的问题更像：
    
    - 本地 dev-local-api 上传落盘压力
    - ingest-cpu-worker 队列堆积  
        不一定是 ocr-service 显卡没吃满。

更直接地说：

- 如果你想提升“扫描 PDF / 图片类 OCR 文档”的吞吐：  
    可以适度提高 WORKER_CONCURRENCY
- 但这本质上是在提高“同时向 OCR service 喂活”的速度
- 真正该看的是 ocr-service 能不能扛住，而不是只看 GPU 还有没有空显存

我建议你这样理解调参顺序：

- 普通 Office / txt / md / xlsx / pptx 多：  
    先调 CPU_WORKER_CONCURRENCY
    
- 扫描 PDF / 图片 OCR 多：  
    再小步调 WORKER_CONCURRENCY  
    建议 3 -> 4 试，不要一步拉很高
    
- 如果 OCR service 本身是单实例：  
    WORKER_CONCURRENCY 太高只会把压力堆到 ocr-service
    

一句话总结：  
**不是不能多开 worker，而是你现在多开的主要是“请求分发并发”，不等于 GPU 计算并发会按比例提升。**

如果你愿意，我下一步可以直接给你一版：

- “扫描 PDF 为主”
- “Office 文档为主”  
    这两种场景分别怎么配 CPU_WORKER_CONCURRENCY / WORKER_CONCURRENCY / PREVIEW_WORKER_CONCURRENCY