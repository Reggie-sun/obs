
已经改成 Docker-only 了，在 Makefile。

现在启动方式收口成这样：

- make dev：启动 Docker 的 api + worker
- make dev-frontend：本地起前端
- make dev-docker-stop：停 Docker 后端
- make doctor-api-port：看 8020 当前是谁在占

我把本地起后端/worker 的入口去掉了，也把 help 文案同步改了。验证过：

- make help 正常
- make -n dev 执行路径正确
- git diff --check -- Makefile 无格式问题

你后面就按这个用：

- 终端 1：make dev
- 终端 2：make dev-frontend