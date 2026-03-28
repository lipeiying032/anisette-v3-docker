# SETUP

## 前提条件

- 海外 VPS，已安装 Docker 和 Docker Compose Plugin
- 端口 6969 对外开放（见防火墙一节）

安装 Docker（如尚未安装）：

```bash
curl -fsSL https://get.docker.com | sh
```

---

## 部署

```bash
git clone https://github.com/你的用户名/anisette-v3-docker.git
cd anisette-v3-docker
docker compose up -d
```

首次启动会自动从苹果服务器下载认证库，约 1～5 分钟。
用以下命令观察进度，看到 `Serving on port 6969` 即就绪：

```bash
docker compose logs -f anisette
```

---

## 验证服务正常

```bash
curl http://localhost:6969
```

正常响应是包含 `X-Apple-I-MD` 等字段的 JSON。

---

## 防火墙

```bash
ufw allow 6969/tcp
ufw reload
```

---

## 配置 SideStore

**第一步：准备 servers.json**

复制仓库里的 `servers.json.example`，填入 VPS 的真实 IP：

```json
{
  "servers": [
    {
      "name": "My Server",
      "address": "http://你的VPS公网IP:6969"
    }
  ]
}
```

将此文件托管到任意可公开访问的 URL，推荐 GitHub Pages：
- 在 GitHub 新建仓库，上传 `servers.json`，开启 Pages
- 文件 URL 格式：`https://用户名.github.io/仓库名/servers.json`

**第二步：在 SideStore 中切换服务器**

1. SideStore → 右下角 Settings
2. 找到 **Anisette Servers**，点击列表 URL
3. 替换为你托管的 `servers.json` URL
4. 点击 **Refresh Servers**
5. 从列表中选择 **My Server**
6. 退出重新登录 Apple ID 使配置生效

---

## 维护

```bash
# 更新到最新镜像
docker compose pull && docker compose up -d

# 查看运行状态
docker compose ps

# 查看日志
docker compose logs -f anisette

# 重置（苹果库损坏时使用）
docker compose down
docker volume rm anisette-v3_data
docker compose up -d
```

---

## 反向代理 / HTTPS（可选）

SideStore 支持 HTTP，不强制 HTTPS。有需要时自行在前面套 nginx / Caddy / Traefik。