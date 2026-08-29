# Nginx 安装与配置

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-08-15 |
| 适用范围 | 公司所有后端研发团队与运维（Nginx 1.27.x） |
| 作者 | 架构组 |

> **用途**：Nginx 的编译安装、systemd 托管、反向代理、负载均衡、HTTPS 配置与常用命令速查，作为网关与静态资源服务的标准参考。

---

## 2. 版本约定

| 项 | 约定 |
|----|------|
| **版本** | Nginx 1.27.x |
| **端口** | 80 / 443 |
| **使用场景** | 反向代理、负载均衡、静态资源、HTTPS 终结 |
| **配置检查** | 每次变更后 `nginx -t` 校验 |

---

## 3. 编译安装

### 3.1 安装依赖并编译

```bash
yum install -y gcc pcre-devel zlib-devel openssl-devel

# 下载（版本号以官方最新 1.27 系列为准）
wget https://nginx.org/download/nginx-1.27.2.tar.gz
tar -zxvf nginx-1.27.2.tar.gz
cd nginx-1.27.2

# 编译安装（含 SSL 与 gzip 模块）
./configure --prefix=/usr/local/nginx \
  --with-http_ssl_module \
  --with-http_gzip_static_module \
  --with-http_stub_status_module
make -j4 && make install
```

### 3.2 配置 systemd 服务

```bash
cat > /etc/systemd/system/nginx.service <<'EOF'
[Unit]
Description=Nginx HTTP Server
After=network.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s quit

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now nginx
```

---

## 4. 反向代理配置

```nginx
server {
    listen 80;
    server_name api.pangpang.com;

    # 后端服务代理（Spring Boot 4.x）
    location /api/ {
        proxy_pass http://backend-servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 5. 负载均衡配置

```nginx
upstream backend-servers {
    # 轮询（默认）
    server 10.0.0.1:8080 weight=3;
    server 10.0.0.2:8080 weight=2;

    # 健康检查（可选）
    server 10.0.0.3:8080 down;
}

server {
    listen 80;
    server_name api.pangpang.com;

    location / {
        proxy_pass http://backend-servers;
        proxy_set_header Host $host;
    }
}
```

> 负载均衡策略：默认轮询；`weight` 加权轮询；`ip_hash` 会话保持；`least_conn` 最少连接。需要根据后端有无状态选择。

---

## 6. HTTPS 配置

```nginx
server {
    listen 443 ssl;
    server_name api.pangpang.com;

    ssl_certificate     /etc/nginx/ssl/api.pangpang.com.pem;
    ssl_certificate_key /etc/nginx/ssl/api.pangpang.com.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # HTTP 强制跳转 HTTPS
    if ($scheme = http) {
        return 301 https://$host$request_uri;
    }

    location / {
        proxy_pass http://backend-servers;
    }
}
```

> 【强制】生产必须启用 HTTPS；禁用 TLSv1.0/1.1 与弱加密套件。

---

## 7. 静态资源与 gzip

```nginx
server {
    listen 80;
    server_name www.pangpang.com;
    root /usr/share/nginx/html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    gzip on;
    gzip_types text/css application/javascript application/json;
    gzip_min_length 1k;
}
```

> 【推荐】前端 SPA 使用 `try_files ... /index.html` 实现 history 路由回退；静态资源开启 gzip 与缓存头。

---

## 8. 常用命令速查

| 操作 | 命令 |
|------|------|
| 启动 | `nginx` 或 `systemctl start nginx` |
| 停止 | `nginx -s quit`（优雅）/ `nginx -s stop`（强制） |
| 重载配置 | `nginx -s reload` |
| 检查配置 | `nginx -t` |
| 查看版本与模块 | `nginx -V` |
| 查看日志 | `tail -f /usr/local/nginx/logs/access.log` |

> 【强制】每次修改 `nginx.conf` 后必须先 `nginx -t` 校验，再 `nginx -s reload`。

---

## 9. 落地检查清单

| 序号 | 检查项 | 检查方式 | 责任人 |
|------|--------|---------|--------|
| 1 | `nginx -t` 无报错 | 命令验证 | 运维 |
| 2 | 反向代理可正常访问 | curl 验证 | 后端开发/运维 |
| 3 | HTTPS 证书有效且强协议 | SSL Labs / openssl 验证 | 运维 |
| 4 | 负载均衡分发正常 | 访问日志核对 | 运维 |
| 5 | 配置已纳入版本管理 | git 提交检查 | 运维 |

---

**文档结束**

*本文档由 pangpang-doc 维护*
