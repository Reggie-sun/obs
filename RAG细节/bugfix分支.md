
**我怎么启动这个分支的api

2:01 PM

Worked for 1m 29s

这个分支本地 API 端口是 18020。我刚看了下，现在本机没看到它在跑。

推荐这样启动：

`cd /home/reggie/vscode_folder/enterprise-rag-bugfix-20260430 # 如果本地 postgres/redis/qdrant 还没跑，先起基础设施 docker compose --env-file .branch-runtime.local --profile local-infra up -d postgres qdrant redis # 启动这个分支的 API + ingest workers make dev-docker`

启动后验证：

`curl http://127.0.0.1:18020/api/v1/health make doctor-api-port docker compose --env-file .branch-runtime.local logs -f api`

如果你只想单独起 API，不起 worker：

`docker compose --env-file .branch-runtime.local up -d api`

注意：你打开的 docker-compose.branch-bugfix.yml 更像是 server/remote overlay，本机直接用它可能会卡在外部网络 enterprrise-grade_rag_default 不存在。日常本机启动优先用上面的 make dev-docker。

Web previewWebsiteOpen





**我怎么在服务器同步代码，git fetch 然后git reset --hard吗

2:06 PM

Worked for 12s

不建议你手动上服务器 git reset --hard 作为默认流程。这个分支现在已有专门脚本，会从你本机当前 worktree rsync 到服务器，并且刻意保留远端 .branch-runtime.local：

`cd /home/reggie/vscode_folder/enterprise-rag-bugfix-20260430 make redeploy-remote-bugfix-worker`

它会做这些事：

`rsync 当前本地代码到 lenovo@192.168.10.200:/home/lenovo/vscode_folder/enterprise-rag-bugfix-20260430 排除 .git / .branch-runtime.local / data / storage 等 远端 docker compose build + up api 和 workers`

如果你明确想在服务器用 git 同步，只有在确认服务器目录没有本地改动、只是部署副本时，才用：

`ssh lenovo@192.168.10.200 cd /home/lenovo/vscode_folder/enterprise-rag-bugfix-20260430 git status --short git fetch origin git reset --hard origin/bugfix/wip-20260430`

重启worker：
docker compose --env-file .branch-runtime.local \
  -p enterprise-rag-bugfix-20260430 \
  -f docker-compose.server.yml \
  -f docker-compose.branch-bugfix.yml \
  up -d --no-deps worker

推送代码到服务器加上加上重启api加上worker，冷启动ocr worker：
make redeploy-remote-bugfix-worker