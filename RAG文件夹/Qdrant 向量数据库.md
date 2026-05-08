## 1.6333与6334
docker run -d \
  --name qdrant \
  -p 6333:6333 \
  -p 6334:6334 \
  qdrant/qdrant


### 情况1：你现在（开发 / 学习 RAG）

👉 **只开 6333 就够了**

docker run -d \  
  --name qdrant \  
  -p 6333:6333 \  
  -v $(pwd)/qdrant_storage:/qdrant/storage \  
  qdrant/qdrant

---

### ✅ 情况2：你做企业级 / 高并发

👉 建议两个都开（面试可以吹）

-p 6333:6333 \  
-p 6334:6334

---

### 🧠 一句话总结（面试可以这么说）

👉

> Qdrant 提供 HTTP 和 gRPC 两种接口
> 
> - HTTP（6333）用于开发和通用调用
>     
> - gRPC（6334）用于高性能、低延迟场景


## 2.http://qdrant:6333与http://localhost:6333
AG_QDRANT_URL=http://qdrant:6333和RAG_QDRANT_URL=http://localhost:6333有什么区别吗

有，区别很大，关键在“谁在发请求”。

RAG_QDRANT_URL=http://localhost:6333

- 表示访问“当前进程所在机器/容器自己的 6333”
- 如果你是本机直接跑 uvicorn，那它指的就是你宿主机上的 6333
- 适合：backend 在本机跑，qdrant 通过 Docker 暴露端口给宿主机

RAG_QDRANT_URL=http://qdrant:6333

- qdrant 这里不是固定魔法词，它是 Docker Compose 里的服务名
- 只有在同一个 compose 网络里的容器之间，才能通过这个名字互相访问
- 适合：backend 也在 Docker Compose 里跑，和 qdrant 同网段

你这个项目里可以这样记：

- 本机 conda activate rag_backend + uvicorn ...  
    用 http://localhost:6333
- docker compose up api qdrant  
    用 http://qdrant:6333

最容易踩的坑就是：

- 你在本机跑后端，却配了 http://qdrant:6333
- 结果宿主机根本不认识 qdrant 这个容器内网名字

一句话：

- localhost = 当前运行环境自己
- qdrant = Docker Compose 网络里的服务名