# OpenCode Server Deployment

将 opencode 部署为远程服务，通过手机浏览器访问 Web UI 控制 AI 编程。

## 架构

```
手机浏览器
    │  HTTP Basic Auth (用户名/密码)
    ▼
OpenCode Server (Docker 容器, port 4096)
    │  ├─ 嵌入式 Web UI (SolidJS)
    │  ├─ REST API + WebSocket
    │  └─ AI Provider (Anthropic / OpenAI / ...)
    │
    └─ /workspace (挂载你的项目代码)
```

## 为什么用 Docker？

CentOS 7.9 的 glibc 2.17 不满足 Bun 的 glibc 2.28+ 要求。
Dockerfile 用 Alpine (musl) 构建静态二进制，绕过 glibc 限制。

## 部署步骤

### 1. 在服务器上克隆仓库

```bash
cd ~
git clone https://github.com/your-fork/opencode.git
cd opencode
```

### 2. 配置环境变量

```bash
cd deploy
cp .env.example .env
# 编辑 .env，至少设置：
#   OPENCODE_SERVER_PASSWORD=你的强密码
#   ANTHROPIC_API_KEY=sk-ant-...（或你使用的 provider key）
nano .env
```

### 3. 准备项目代码

把你的项目代码放到 `deploy/workspace/` 目录，或者在 `.env` / `docker-compose.yml` 中修改挂载路径。

```bash
# 示例：克隆一个你要用 opencode 编辑的项目
cd deploy
mkdir workspace
git clone <your-project-repo> workspace/my-project
```

### 4. 构建并启动

```bash
cd deploy
docker compose up -d --build
```

首次构建约 10-20 分钟（安装依赖 + 编译 Web UI + 打包二进制）。

查看构建日志：

```bash
docker compose logs -f opencode
```

### 5. 验证服务

```bash
# 检查容器状态
docker compose ps

# 测试 API
curl -u opencode:你的密码 http://localhost:4096/
```

### 6. 从手机访问

**方案 A — 直接访问（需要防火墙放行 4096 端口，不推荐公网）**

```bash
# 在 CentOS 上开放端口
firewall-cmd --permanent --add-port=4096/tcp
firewall-cmd --reload
```

手机浏览器访问：`http://<服务器IP>:4096`
浏览器会弹出登录框，输入 `.env` 中的用户名和密码。

**方案 B — Tailscale（推荐，最安全最简单）**

```bash
# 服务器装 Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up

# 手机装 Tailscale app，登录同一账号
```

手机通过 Tailscale 内网 IP 访问：`http://100.x.x.x:4096`
无需开放任何端口，流量全程加密。

**方案 C — Cloudflare Tunnel（有域名的话）**

详见下方 Cloudflare Tunnel 配置。

## Cloudflare Tunnel 配置（可选）

适合有自己域名、想从任何网络访问的场景。

### 1. 创建隧道

在 [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/) 创建隧道，获取 Tunnel Token。

### 2. 修改 docker-compose.yml

在 `services` 下添加 cloudflared：

```yaml
  cloudflared:
    image: cloudflare/cloudflared:latest
    command: tunnel --no-autoupdate run --token ${CF_TUNNEL_TOKEN}
    restart: unless-stopped
    depends_on:
      - opencode
```

在 `.env` 中添加：

```
CF_TUNNEL_TOKEN=你的tunnel-token
```

### 3. 在 Cloudflare 配置 Public Hostname

- **Subdomain**: `opencode`
- **Domain**: `yourdomain.com`
- **Service**: `http://opencode:4096`

访问 `https://opencode.yourdomain.com` 即可。

## 安全注意事项

| 风险 | 对策 |
|------|------|
| 密码泄露 | 使用强密码，定期更换 |
| 公网暴露 | 优先用 Tailscale 或 Cloudflare Tunnel |
| API Key 泄露 | `.env` 文件权限设为 600，不要提交到 git |
| 容器逃逸 | 不要用 `--privileged`，保持 Docker 更新 |

## 常用命令

```bash
# 查看日志
docker compose logs -f opencode

# 重启服务
docker compose restart opencode

# 更新 opencode（重新构建）
cd ~/opencode
git pull
cd deploy
docker compose up -d --build

# 进入容器调试
docker compose exec opencode sh

# 停止服务
docker compose down
```

## 故障排查

**构建失败：内存不足**

```bash
# Bun build 需要约 1GB 内存，如果服务器内存小于 1GB，添加 swap：
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
```

**容器启动后立即退出**

```bash
# 查看详细错误
docker compose logs opencode
```

**手机无法访问**

- 检查防火墙：`firewall-cmd --list-ports`
- 检查容器状态：`docker compose ps`
- 检查端口映射：`docker compose port opencode 4096`
