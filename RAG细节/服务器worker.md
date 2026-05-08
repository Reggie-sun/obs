
如果你是在服务器上用这个仓库的 docker-compose.server.yml 起的，那重启 worker 用这条：

`docker compose -f docker-compose.server.yml restart worker`

如果你只是改了 .env 或普通运行态，restart 就够了。  
但如果你改了镜像相关内容，比如：

- backend/Dockerfile.worker-gpu
- requirements/ocr-gpu-cu130.txt
- docker-compose.server.yml 里的 build / image

那不能只 restart，要先重建再拉起：

`make build-worker-server docker compose -f docker-compose.server.yml up -d --no-deps --force-recreate worker`

想确认它起来了，可以接着看：

`docker compose -f docker-compose.server.yml ps worker docker compose -f docker-compose.server.yml logs -f worker`

你现在这个场景，如果只是想“重启服务进程”，用 restart worker；如果是刚改过 GPU OCR 镜像，优先用 up -d --no-deps --force-recreate worker。


如果你要的是 recreate，对应这条：

`docker compose -f docker-compose.server.yml up -d --no-deps --force-recreate worker`

它和 restart 的区别很简单：

- restart  
    停掉再启动“同一个容器”
- force-recreate  
    删掉旧容器，按当前 compose 配置重新创建一个新容器

所以这些场景更适合 recreate：

- 改了 .env
- 改了 docker-compose.server.yml 里的 environment / gpus / command
- 想确保容器不是旧状态残留
- 想明确切到当前 compose 指向的镜像

但它仍然不会自动重新 build 镜像。  
如果你改了 Dockerfile / requirements / 镜像内容，完整流程还是：

make build-worker-server
docker compose -f docker-compose.server.yml up -d --no-deps --force-recreate worker

你可以把它记成：

- 只重启进程：restart worker
- 重新造容器：up -d --no-deps --force-recreate worker
- 重新造镜像再换容器：make build-worker-server + up -d --no-deps --force-recreate worker


make sync-remote-bugfix-workspace      # 只推代码
make redeploy-remote-bugfix-worker     # 推代码 + 重启 API/worker
