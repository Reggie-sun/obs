
查看log:
cd /home/reggie/vscode_folder/chatgpt-on-wechat/chatgpt-on-wechat
docker compose -f docker-compose.deploy.yml --env-file .env.deploy logs -f caddy cloudflared gateway dashboard

重启容器:
docker compose -f docker-compose.deploy.yml --env-file .env.deploy down --remove-orphans
docker compose -f docker-compose.deploy.yml --env-file .env.deploy up -d

验证内网：
docker compose -f docker-compose.deploy.yml --env-file .env.deploy exec caddy curl -I http://dashboard:8501
docker compose -f docker-compose.deploy.yml --env-file .env.deploy exec cloudflared curl -I http://caddy:80

# 给 caddy 装 curl
docker compose -f docker-compose.deploy.yml --env-file .env.deploy exec caddy \
  sh -c "apk add --no-cache curl"

# 给 cloudflared 装 curl
docker compose -f docker-compose.deploy.yml --env-file .env.deploy exec cloudflared \
  sh -c "apk add --no-cache curl"


# 停并清孤儿
docker compose -f docker-compose.deploy.yml --env-file .env.deploy down --remove-orphans

# 确认杀掉旧的 cow-* 容器（有的话）：
docker rm -f cow-bot cow-gateway cow-dashboard cow-vector cow-redis 2>/dev/null || true

# 删掉旧网络，确保重建
docker network rm chatgpt-on-wechat_cow-net 2>/dev/null || true

# 重建并启动
docker compose -f docker-compose.deploy.yml --env-file .env.deploy up -d

要先关闭所有容器方便本地测试，直接执行：
docker compose -f docker-compose.deploy.yml --env-file .env.deploy down


# 使用 Docker + Caddy + Cloudflared 部署 ChatGPT-on-WeChat（客服版）

  

本文档按你的项目结构写，支持以下部署方式：

  

- 使用 `docker-compose.deploy.yml` 一键启动

- 通过 Caddy 做反向代理 + TLS

- 通过 Cloudflared 暴露为：

- `WX_DOMAIN`（公众号回调）

- `GATEWAY_DOMAIN`（API 网关）

- `DASH_DOMAIN`（Dashboard / RAG H5 页面）

  

> 仓库根目录假设为：`~/vscode_folder/chatgpt-on-wechat/chatgpt-on-wechat`

  

---

  

## 1. 组件说明

  

### 1.1 Docker 服务（`docker-compose.deploy.yml`）

  

```yaml

services:

gateway: # Smart Gateway（uvicorn gateway.app:app --port 8500）

bot: # 公众号机器人（app.py，监听 /wx）

dashboard: # Dashboard & RAG H5（uvicorn dashboard.app:app --port 8501）

caddy: # 反向代理 + TLS，监听 :80 / :443

cloudflared:# Cloudflare Tunnel 客户端

```

  

dashboard 已映射端口：

  

```yaml

dashboard:

...

ports:

- "8501:8501"

```

  

你可以在服务器本机直接访问：

`http://<服务器公网IP>:8501/static/user.html`

  

### 1.2 Caddy（`deploy/Caddyfile`）

  

负责域名到容器的映射：

  

- `WX_DOMAIN` -> `bot:8080`

- `GATEWAY_DOMAIN` -> `gateway:8500`

- `DASH_DOMAIN` -> `dashboard:8501`

  

使用 Cloudflare DNS 进行证书自动签发。

  

### 1.3 Cloudflared（`deploy/cloudflared/config.yml`）

  

- 通过 Cloudflare Tunnel 把 `caddy:80` 暴露给 Cloudflare 边缘。

- Cloudflare 通过你的域名（`wx.example.com` 等）把请求转发到 Tunnel，再进 Caddy，再进容器。

  

---

  

## 2. 必要准备

  

1. **服务器环境**

- 一台 Linux 服务器（有公网 IP）。

- 已安装 Docker 和 Docker Compose。

  

2. **Cloudflare 账号 + 域名**

- 域名托管在 Cloudflare，如 `srj666.com`。

- 创建一个 API Token（DNS 权限即可）。

  

3. **Cloudflare Tunnel**

- 在 Cloudflare Dashboard 创建 Tunnel，得到一个 TUNNEL_ID。

- 下载该 Tunnel 的 credentials JSON 文件，命名为

`deploy/cloudflared/credentials/<TUNNEL_ID>.json`。

  

4. **微信公众平台**

- 拿到：

- `AppID`

- `AppSecret`

- `Token`

- `EncodingAESKey`

  

5. **克隆项目**

  

```bash

git clone https://github.com/你的用户名/kefu.git

cd kefu/chatgpt-on-wechat

```

  

---

  

## 3. 配置 `.env.deploy`

  

在 `chatgpt-on-wechat` 目录下已经有 `.env.deploy`，按需修改（用你自己的值替换占位符）：

  

```bash

# === Bot & Model ===

channel_type=wechatmp

bot_type=chatGPT

model=deepseek-chat

open_ai_api_key=YOUR_OPENAI_API_KEY

open_ai_api_base=https://api.deepseek.com/v1

  

# === WeChat MP ===

wechatmp_token=YOUR_WECHAT_TOKEN

wechatmp_app_id=YOUR_WECHAT_APP_ID

wechatmp_app_secret=YOUR_WECHAT_APP_SECRET

wechatmp_aes_key=YOUR_WECHAT_AES_KEY

wechatmp_port=8080

  

# === Smart Gateway ===

smart_gateway_enabled=true

smart_gateway_base_url=http://gateway:8500

smart_gateway_timeout=60

smart_gateway_tools_allowed=["lookup_order","check_logistics","product_info"]

smart_gateway_rag_top_k=3

smart_gateway_rag_threshold=0.3

  

# === External customer service RAG (可选) ===

customer_service_api_base_url=https://api.your-rag.com

customer_service_api_token=YOUR_RAG_TOKEN

customer_service_api_timeout=60

  

# === Domains & TLS ===

WX_DOMAIN=wx.srj666.com

GATEWAY_DOMAIN=gateway.srj666.com

DASH_DOMAIN=dashkefu.srj666.com

CADDY_ACME_EMAIL=you@example.com

  

# === Cloudflared ===

CLOUDFLARED_TUNNEL_ID=你的_TUNNEL_ID

CLOUDFLARED_UPSTREAM_URL=http://caddy:80

CLOUDFLARED_PROTOCOL=quic

CLOUDFLARE_API_TOKEN=你的_CLOUDFLARE_API_TOKEN

```

  

注意：

  

- `WX_DOMAIN`：给公众号服务器用，回调 URL 会是 `https://WX_DOMAIN/wx`。

- `DASH_DOMAIN`：给 H5 页面用，如 RAG 用户指引 `https://dashkefu.srj666.com/static/user.html`。

- `GATEWAY_DOMAIN`：给你自己或第三方调用 Gateway API 用。

  

---

  

## 4. Caddy 配置（`deploy/Caddyfile`）

  

一个简化版结构如下（你已有完整文件，这里便于理解）：

  

```caddyfile

{

email {$CADDY_ACME_EMAIL}

admin off

auto_https disable_redirects

}

  

(common_headers) {

encode zstd gzip

header {

Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"

Referrer-Policy "strict-origin-when-cross-origin"

X-Content-Type-Options "nosniff"

X-Frame-Options "DENY"

Permissions-Policy "camera=(), microphone=(), geolocation=(self)"

}

}

  

(wx_site) {

import common_headers

reverse_proxy bot:8080

tls {

dns cloudflare {env.CLOUDFLARE_API_TOKEN}

}

}

  

(gateway_site) {

import common_headers

reverse_proxy gateway:8500

tls {

dns cloudflare {env.CLOUDFLARE_API_TOKEN}

}

}

  

(dash_site) {

import common_headers

reverse_proxy dashboard:8501

tls {

dns cloudflare {env.CLOUDFLARE_API_TOKEN}

}

}

  

http://{$WX_DOMAIN} { import wx_site }

https://{$WX_DOMAIN} { import wx_site }

  

http://{$GATEWAY_DOMAIN} { import gateway_site }

https://{$GATEWAY_DOMAIN}{ import gateway_site }

  

http://{$DASH_DOMAIN} { import dash_site }

https://{$DASH_DOMAIN} { import dash_site }

  

# 默认 404

http://:80 { respond "not found" 404 }

https://:443 { respond "not found" 404 }

```

  

---

  

## 5. Cloudflared 配置（`deploy/cloudflared/config.yml`）

  

示例结构：

  

```yaml

tunnel: ${CLOUDFLARED_TUNNEL_ID}

credentials-file: /etc/cloudflared/credentials/${CLOUDFLARED_TUNNEL_ID}.json

  

ingress:

- hostname: wx.srj666.com

service: http://caddy:80

- hostname: gateway.srj666.com

service: http://caddy:80

- hostname: dashkefu.srj666.com

service: http://caddy:80

- service: http_status:404

```

  

- 所有入口域名都指向 `http://caddy:80`。

- Caddy 再根据 Host 头路由到不同服务。

  

---

  

## 6. 启动 / 停止 / 日志

  

### 6.1 启动

  

```bash

cd ~/vscode_folder/chatgpt-on-wechat/chatgpt-on-wechat

  

# 首次或修改完 .env.deploy 后

docker compose -f docker-compose.deploy.yml --env-file .env.deploy up -d

```

  

### 6.2 查看状态

  

```bash

docker compose -f docker-compose.deploy.yml ps

```

  

应看到类似：

  

```text

chatgpt-on-wechat-bot-1 Up

chatgpt-on-wechat-gateway-1 Up

chatgpt-on-wechat-dashboard-1 Up

chatgpt-on-wechat-caddy-1 Up

chatgpt-on-wechat-cloudflared-1 Up

```

  

### 6.3 查看日志

  

```bash

# Bot 日志（含 [wechatmp]…）

docker compose -f docker-compose.deploy.yml logs -f bot

  

# Gateway 日志（含 [gateway] calling customer_service_rag…）

docker compose -f docker-compose.deploy.yml logs -f gateway

  

# Dashboard 日志

docker compose -f docker-compose.deploy.yml logs -f dashboard

  

# Caddy / Cloudflared

docker compose -f docker-compose.deploy.yml logs -f caddy

docker compose -f docker-compose.deploy.yml logs -f cloudflared

```

  

### 6.4 重启单个服务

  

```bash

docker compose -f docker-compose.deploy.yml restart gateway

docker compose -f docker-compose.deploy.yml restart bot

docker compose -f docker-compose.deploy.yml restart dashboard

```

  

---

  

## 7. 本地直接访问 Dashboard / H5

  

有时候 Cloudflare 或 Tunnel 会 502，你可以直接用服务器 IP 访问 Dashboard，绕过 Cloudflare 调试：

  

```text

http://<服务器公网IP>:8501/ # Dashboard 根

http://<服务器公网IP>:8501/static/user.html # RAG 用户 H5 页

```

  

如果这里都能访问而 `https://dashkefu.srj666.com/static/user.html` 偶尔 502，大概率是 Cloudflare / 隧道侧的网络抖动。

  

---

  

## 8. 微信公众平台配置

  

在公众号「设置与开发 → 基本配置 → 服务器配置」：

  

- URL 填：`https://wx.srj666.com/wx`

- Token 填：`.env.deploy` 中的 `wechatmp_token`

- 消息加解密方式：

- 使用安全模式时，EncodingAESKey 填 `wechatmp_aes_key`

- 点击「启用」或「提交」，微信会请求 `https://wx.srj666.com/wx` 做校验：

- 如果配置正确，你会看到“配置成功”。

  

---

  

## 9. 常见故障总结

  

1. **公众号没反应**

- 看 `bot` 日志有没有 `[wechatmp] Request 1 from ...`：

- 没有：说明微信没打进来 → 检查域名解析 / Cloudflare / Tunnel / Caddy。

- 有 Request 但没有 `do send`：说明内部处理出错，看同时间 `gateway` 日志。

  

2. **访问 `dashkefu` 报 Cloudflare 502**

- 先本机 `curl http://127.0.0.1:8501/static/user.html`；

- 再容器内测试 `http://dashboard:8501/static/user.html`；

- 两处都 200，则 502 是 Cloudflare / Tunnel 问题，可稍后重试或在 Cloudflare Dashboard 看 Tunnel 状态。

  

3. **Gateway 报错 `CUSTOMER_SERVICE_API_TIMEOUT`**

- 现在代码已经做防御性处理（空字符串时回退默认值），

- 但仍建议在 `.env.deploy` 中明确设置 `customer_service_api_timeout=60`。

  

---

  

## 10. 本地调试 vs 生产部署

  

- **本地调试**：

- 建议使用 `./start_stack.sh`，在不走 Cloudflare 的情况下，快速验证 `bot + gateway + dashboard` 行为。

- 浏览器访问 `http://127.0.0.1:3000/static/user.html`（本地 Dashboard）。

  

- **生产部署**：

- 使用 `docker-compose.deploy.yml + .env.deploy + Caddy + Cloudflared`。

- 对外暴露三个域名：`WX_DOMAIN / GATEWAY_DOMAIN / DASH_DOMAIN`。
