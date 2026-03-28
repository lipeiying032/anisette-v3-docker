# anisette-v3-docker

一键部署 [anisette-v3-server](https://github.com/Dadoum/anisette-v3-server)，供 SideStore 使用。

## 快速启动

```bash
docker compose up -d
```

启动后服务监听在 `http://your-ip:6969`。

首次启动会自动从苹果服务器下载认证库，约 1～5 分钟，完成后永久缓存。

## 配置 SideStore

1. SideStore → Settings → Anisette Server → 替换服务器列表 URL 为你托管的 `servers.json`
2. Refresh Servers → 选择你的服务器

`servers.json` 示例（改 IP/域名后托管到 GitHub Pages 或任意静态服务）：

```json
{
  "servers": [
    {
      "name": "My Server",
      "address": "http://your-ip:6969"
    }
  ]
}
```

## 防火墙

```bash
ufw allow 6969/tcp
```

## 反向代理 / HTTPS

有需要的自行在前面套 nginx / Caddy / Traefik。
SideStore 支持 HTTP，不强制 HTTPS。

## 维护

```bash
# 更新镜像
docker compose pull && docker compose up -d

# 重置数据
docker compose down && docker volume rm anisette-v3_data && docker compose up -d
```