cd ~/vscode_folder/story_gen/story-app
docker compose -f docker-compose.deploy.yml logs -f story-backend
docker compose -f docker-compose.deploy.yml logs -f caddy
docker compose -f docker-compose.deploy.yml logs -f cloudflared

docker exec story-app-story-backend-1 nvidia-smi

docker compose -f docker-compose.deploy.yml restart story-backend



代码改完推荐执行：
cd /home/reggie/vscode_folder/story_gen/story-app
docker compose -f docker-compose.deploy.yml build --no-cache story-backend
docker compose -f docker-compose.deploy.yml up -d story-backend

huozhe 
docker compose -f docker-compose.deploy.yml restart story-backend

