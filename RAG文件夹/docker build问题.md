

下次最稳的做法就是固定成一种开发模式，不要“本地后端”和“Docker 后端”混着占同一个 8020。

**推荐方式**  
我建议你日常开发直接用“前后端都本地跑”：

1. 先把 Docker 的后端停掉
    - docker compose stop api worker
    - 确认 docker ps | rg 'enterprise-rag-api|enterprise-rag-worker' 没有运行中的后端容器
2. 再启动本地后端
    - 在仓库根目录跑你平时的 uvicorn 命令
    - 然后确认 lsof -i :8020
    - 看到的应该是本地 Python/uvicorn，不应该是 Docker
3. 启动前端
    - cd frontend
    - npm run dev

这样你本地改代码，前后端都会立刻按本地代码走，不会串到旧容器。

**如果你要用 Docker 后端**  
那就反过来，别再起本地 uvicorn，并且只用这两类命令：

- 第一次或改了依赖/镜像时：
    - docker compose up -d --build api worker
- 平时只是继续跑：
    - docker compose up -d api worker

尽量不要用：

- docker restart enterprise-rag-api
- docker restart enterprise-rag-worker

因为 restart 只会重启“现有那个容器”，不会重新套用 compose 配置；如果那个容器本身有问题，你就会把问题一起重启回来。

**每次启动前，30 秒自检**  
你只要看两件事：

- lsof -i :8020
    
    - 如果是 Docker 进程/容器占着，说明请求会走 Docker 后端
    - 如果是本地 uvicorn，说明请求会走本地后端
- 如果你在用 Docker，再看：
    
    - docker inspect enterprise-rag-api --format '{{json .Mounts}}'
    - 里面应该能看到 /home/reggie/vscode_folder/Enterprise-grade_RAG -> /app

**一句话规则**

- 用本地后端：先停 Docker api/worker
- 用 Docker 后端：不要再起本地 uvicorn
- 想让 Docker吃到本地代码：必须确认它是“带挂载创建”的容器
- 不确定时先查 lsof -i :8020