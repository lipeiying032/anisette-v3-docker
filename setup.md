# SideStore 自建 Anisette-v3 服务器部署手册

## 架构说明

```
iPhone (SideStore + LocalDevVPN)
        │  HTTPS 请求认证数据
        ▼
  VPS: Nginx (443) ──► anisette-v3-server (6969 内部)
        │
  Certbot 自动续签 Let's Encrypt 证书
  宿主机 Cron 监测信号文件后重载 nginx（必须配置！）
```

---

## 文件目录结构

```
~/anisette/
├── docker-compose.yml
├── nginx/conf.d/
│   ├── bootstrap.conf    ← 仅首次申请证书时使用
│   └── anisette.conf     ← 正式 HTTPS 配置（首次时不放这里！）
├── certbot/www/          ← Let's Encrypt HTTP 验证（自动创建）
├── certbot/conf/         ← 证书存储（自动创建）
├── certbot/reload/       ← nginx 重载信号目录（自动创建）
└── servers.json          ← 托管到 GitHub Pages 供 SideStore 使用
```

---

## 第一步：VPS 环境准备

```bash
curl -fsSL https://get.docker.com | sh
apt install -y docker-compose-plugin
mkdir -p ~/anisette/{nginx/conf.d,certbot/{www,conf,reload}}
cd ~/anisette
```

---

## 第二步：上传配置文件并替换域名

```bash
# 批量替换所有文件里的占位符
sed -i 's/your-domain.com/你的真实域名/g' nginx/conf.d/anisette.conf
sed -i 's/your-domain.com/你的真实域名/g' nginx/conf.d/bootstrap.conf
sed -i 's/your-domain.com/你的真实域名/g' servers.json
```

---

## 第三步：首次申请 SSL 证书（必须分两阶段）

> ⚠️ 为什么分两阶段？
> anisette.conf 引用了尚不存在的 .pem 证书文件，nginx 直接 crash。
> 必须先用纯 HTTP 的 bootstrap.conf 启动 → 申请证书 → 再切换配置。

### 阶段一：只放 bootstrap.conf，启动 nginx

```bash
# 确保 nginx/conf.d/ 下只有 bootstrap.conf，没有 anisette.conf
# 如果 anisette.conf 已在目录里，先移走：
mv nginx/conf.d/anisette.conf ~/anisette.conf.bak

docker compose up -d nginx
docker compose logs nginx   # 确认无报错
```

### 阶段二：申请证书

```bash
docker compose run --rm certbot certbot certonly \
  --webroot --webroot-path /var/www/certbot \
  --email your@email.com --agree-tos --no-eff-email \
  -d 你的真实域名

# 验证证书文件已生成
ls certbot/conf/live/你的真实域名/
# 应有：cert.pem  chain.pem  fullchain.pem  privkey.pem
```

### 阶段三：切换到正式 HTTPS 配置

```bash
rm nginx/conf.d/bootstrap.conf
mv ~/anisette.conf.bak nginx/conf.d/anisette.conf

docker compose restart nginx
docker compose logs nginx   # 确认无报错
```

---

## 第四步：启动全部服务

```bash
docker compose up -d
docker compose ps           # 查看所有容器状态

# 首次需 1~5 分钟下载苹果库，出现 "Serving on port 6969" 即就绪
docker compose logs -f anisette
```

---

## 第五步：配置 nginx 证书自动重载（宿主机 Cron）

> ⚠️ 这一步不可跳过！
> certbot 续签证书写入磁盘后，nginx 不会自动重读新证书。
> 必须向 nginx 发送 reload 信号，否则约 90 天后 SSL 证书过期服务中断。
>
> 工作原理：certbot --deploy-hook 续签成功后写入 reload 标记文件，
> 宿主机 cron 每天检测到该文件后执行 docker exec 重载 nginx，再删除标记。

```bash
# 创建重载脚本（注意修改路径为你的实际项目路径）
cat > ~/anisette/nginx-reload.sh << 'EOF'
#!/bin/bash
SIGNAL_FILE="/root/anisette/certbot/reload/reload"
NGINX_CONTAINER="anisette-nginx"
LOG="[$(date '+%Y-%m-%d %H:%M:%S')]"

if [ -f "$SIGNAL_FILE" ]; then
    echo "$LOG 检测到证书更新信号，重载 nginx..."
    if docker exec "$NGINX_CONTAINER" nginx -s reload; then
        echo "$LOG nginx 重载成功"
        rm -f "$SIGNAL_FILE"
    else
        echo "$LOG nginx 重载失败！请检查: docker compose logs nginx"
    fi
fi
EOF

chmod +x ~/anisette/nginx-reload.sh

# 添加 cron：每天凌晨 3 点执行
(crontab -l 2>/dev/null; echo "0 3 * * * /root/anisette/nginx-reload.sh >> /var/log/anisette-nginx-reload.log 2>&1") | crontab -

# 验证
crontab -l | grep nginx-reload
```

---

## 第六步：验证服务

```bash
# 测试 anisette 直接响应
curl http://localhost:6969
# 应返回 JSON 含 X-Apple-I-MD 等字段

# 测试 HTTPS
curl -s https://你的真实域名 | head -c 300

# 查看 SSL 证书信息
echo | openssl s_client -connect 你的真实域名:443 2>/dev/null | openssl x509 -noout -dates
```

---

## 第七步：配置 SideStore 使用自建服务器

1. 将 `servers.json` 中的 `your-domain.com` 改为你的真实域名
2. 上传到 GitHub Pages → URL 格式：`https://用户名.github.io/仓库名/servers.json`
3. SideStore → Settings → Anisette Server → 替换列表 URL → Refresh Servers
4. 从列表选择 **My VPS Anisette** → 退出重新登录 Apple ID 生效

---

## 防火墙

```bash
ufw allow 22/tcp && ufw allow 80/tcp && ufw allow 443/tcp && ufw enable
# 端口 6969 无需对外开放，nginx 内部转发
```

---

## 日常维护

```bash
# 更新 anisette 镜像（每月建议执行）
cd ~/anisette && docker compose pull anisette && docker compose up -d anisette

# 查看证书状态
docker compose run --rm certbot certbot certificates

# 手动测试续签流程（不实际续签）
docker compose run --rm certbot certbot renew --dry-run

# 手动重载 nginx
docker exec anisette-nginx nginx -s reload

# 重置 anisette 数据（苹果库损坏时）
docker compose down && docker volume rm anisette-v3_data && docker compose up -d
```

---

## 常见问题

**Q: nginx 报 "cannot load certificate" 无法启动？**
A: nginx/conf.d/ 里同时有 bootstrap.conf 和 anisette.conf。
   第三步阶段一必须只保留 bootstrap.conf，申请完证书后再换成 anisette.conf。

**Q: anisette 一直初始化？**
A: 正常，首次从苹果服务器下载 APK 提取认证库需 1~5 分钟，之后缓存永久有效。

**Q: SideStore 登录报 -36607？**
A: 公共服务器绑定账号过多触发苹果风控，换用自建服务器即可解决。

**Q: 证书已续签但浏览器还报证书错误？**
A: 第五步的 cron 未执行。手动运行 ~/anisette/nginx-reload.sh 或直接
   docker exec anisette-nginx nginx -s reload。

**Q: servers.json 需要 "cache" 字段吗？**
A: 不需要。原来版本的占位符 URL 会导致 SideStore 每次刷新报 404 错误，
   已删除。如果你有能力托管对应的 .hash 文件，可以手动加回。

**Q: 能多人共用吗？**
A: 可以。v3 协议每次生成独立 Anisette 数据，多人使用不触发苹果风控。

**Q: 需要向服务器提供 Apple ID？**
A: 完全不需要。anisette 只生成设备认证头，Apple ID 认证在 SideStore 本地完成。
