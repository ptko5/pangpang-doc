# Docker Compose 部署实战

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-08-15 |
| 适用范围 | 本地开发联调环境、测试环境（ST / UAT）使用 Docker Compose 一键部署整套后端应用的团队 |
| 作者 | 架构组 |

> **适用边界**：本文档面向 **开发 / 测试环境**，不建议直接用于生产（生产请上 Kubernetes + Helm）。生产可参考本实战的健康检查、服务依赖顺序等思路。

---

## 2. 实战背景

### 2.1 目标

用一份 `docker-compose.yml` + 若干配置文件，**一键拉起** Spring Boot 后端服务运行所需的全部组件：

| 组件 | 版本 | 端口 | 说明 |
|------|------|------|------|
| **Spring Boot 后端应用** | JDK 25 + Spring Boot 4.x | 8080（业务）/ 8081（管理） | 用户中心示例服务，镜像来自 `user-service:1.0.0` |
| **MySQL** | 8.4 LTS | 3306 | 业务数据库，持久化存储 + 中文 utf8mb4 |
| **Redis** | 7.2 | 6379 | 缓存 + 分布式锁 + Session，AOF 持久化 |
| **Nginx** | 1.27 Alpine | 80（HTTP）/ 443（HTTPS，可选） | 反向代理到后端，托管前端 dist 静态资源 |

> 若项目还有 RabbitMQ / Elasticsearch / Nacos 等组件，可按本章相同模式追加服务到 compose 文件。

### 2.2 目录结构（【推荐】统一布局）

> 推荐所有项目在仓库根目录下创建 `deploy/` 目录，按如下布局组织 Compose 相关文件，避免和源码混放。

```text
your-project/
├── src/                           # 后端源码
├── front-end/                     # 前端源码（若同仓）
├── build.gradle / pom.xml
├── Dockerfile                     # 后端 Dockerfile（按 §Dockerfile-编写规范 写）
│
└── deploy/                        ← Compose 相关全部放这里
    ├── docker-compose.yml         ← 核心编排文件（本章重点）
    ├── .env                       ← 环境变量集中管理（compose 自动读取）
    │
    ├── mysql/                     ← MySQL 相关
    │   ├── data/                  # 挂载宿主机数据目录（.gitignore 忽略）
    │   ├── conf/
    │   │   └── my.cnf             # 自定义 my.cnf 字符集/性能参数
    │   └── initdb/
    │       └── 01_init_schema.sql # 首次启动自动执行的建库建表 SQL
    │
    ├── redis/                     ← Redis 相关
    │   ├── data/                  # 持久化 dump.rdb / appendonlydir
    │   └── conf/
    │       └── redis.conf         # 自定义 Redis 配置（密码、AOF 等）
    │
    └── nginx/                     ← Nginx 相关
        ├── conf.d/
        │   └── default.conf       # 反向代理 server/location 配置
        ├── logs/                  # access.log / error.log
        └── html/                  # 前端 build 后的静态资源（dist 内容拷进来）
```

> `.gitignore` 补充：**所有 `data/` 和 `logs/` 目录必须忽略**，避免把数据库/日志文件提交到 Git。

---

## 3. 完整 `docker-compose.yml` 示例

> 下面这份 YAML **可直接复制改项目名使用**，所有关键参数均带注释。建议 Compose 文件版本 ≥ 3.8。

```yaml
# ============================================================
# 后端全套服务编排（本地 / 测试环境）
# 用法：
#   cd deploy/
#   docker compose config      # 先校验 YAML + 变量替换是否正确
#   docker compose up -d       # 后台启动所有服务
#   docker compose logs -f     # 实时看所有服务日志
#   docker compose down        # 停止并删除容器（保留挂载数据）
#   docker compose down -v     # 停止 + 删除匿名卷（会清空 MySQL/Redis 数据！慎用）
# ============================================================

services:

  # ================ 1. MySQL 8.4 ================
  mysql:
    image: mysql:8.4
    container_name: dev-mysql
    restart: unless-stopped                   # 除手动 stop 外，机器重启后自动拉起
    environment:
      TZ: Asia/Shanghai
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PWD}  # 从 .env 读取
      MYSQL_DATABASE: ${MYSQL_DB_NAME}        # 启动时自动创建的业务库
      MYSQL_USER: ${MYSQL_APP_USER}           # 业务用户（非 root，推荐应用用这个账号）
      MYSQL_PASSWORD: ${MYSQL_APP_PWD}
    ports:
      - "3306:3306"                           # 宿主机:容器；开发机若已装 MySQL 可改成 "3307:3306"
    volumes:
      - ./mysql/conf/my.cnf:/etc/mysql/conf.d/my.cnf:ro        # 自定义配置（只读挂载）
      - ./mysql/data:/var/lib/mysql                             # 数据目录（持久化到宿主机）
      - ./mysql/initdb:/docker-entrypoint-initdb.d:ro          # 首次启动自动执行目录下 .sql / .sh
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-p${MYSQL_ROOT_PWD}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s         # MySQL 冷启动较慢，首次要设长点
    networks:
      - app-network
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_general_ci
      - --default-authentication-plugin=mysql_native_password

  # ================ 2. Redis 7.2 ================
  redis:
    image: redis:7.2-alpine
    container_name: dev-redis
    restart: unless-stopped
    environment:
      TZ: Asia/Shanghai
    ports:
      - "6379:6379"
    volumes:
      - ./redis/conf/redis.conf:/etc/redis/redis.conf:ro
      - ./redis/data:/data
    command: ["redis-server", "/etc/redis/redis.conf"]       # 用自定义配置启动
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PWD}", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 5s
    networks:
      - app-network

  # ================ 3. Spring Boot 后端 ================
  backend:
    build:
      context: ..                               # build 上下文 = 项目根目录（上一级，才能找到 src 和 Dockerfile）
      dockerfile: ./Dockerfile                  # 相对于 context 的路径
    image: user-service:1.0.0                   # build 完成后打的 tag（也可直接 image: registry.xxx/... 从远程拉）
    container_name: dev-backend
    restart: unless-stopped
    depends_on:                                 # 依赖顺序：先等 MySQL/Redis healthy 再启动后端
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      TZ: Asia/Shanghai
      SPRING_PROFILES_ACTIVE: ${APP_PROFILE:-dev}
      # ===== 数据源：直接用 compose 内服务名当 host =====
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/${MYSQL_DB_NAME}?useUnicode=true&characterEncoding=utf8mb4&serverTimezone=Asia/Shanghai&useSSL=false&allowPublicKeyRetrieval=true
      SPRING_DATASOURCE_USERNAME: ${MYSQL_APP_USER}
      SPRING_DATASOURCE_PASSWORD: ${MYSQL_APP_PWD}
      # ===== Redis：host = 服务名 redis =====
      SPRING_DATA_REDIS_HOST: redis
      SPRING_DATA_REDIS_PORT: 6379
      SPRING_DATA_REDIS_PASSWORD: ${REDIS_PWD}
      # ===== JVM 参数（JDK 25 推荐起始模板）=====
      JAVA_OPTS: >-
        -Xms512m
        -Xmx1024m
        -XX:MaxMetaspaceSize=256m
        -XX:+UseG1GC
        -Dspring.threads.virtual.enabled=true
    ports:
      - "8080:8080"
      - "8081:8081"       # Actuator 管理端口
    volumes:
      - ./backend/logs:/app/logs          # 把容器内应用日志挂载出来，方便本地直接看
    networks:
      - app-network

  # ================ 4. Nginx 反向代理 ================
  nginx:
    image: nginx:1.27-alpine
    container_name: dev-nginx
    restart: unless-stopped
    depends_on:
      backend:
        condition: service_started        # Nginx 不做健康探测，等 backend 启动就够
    environment:
      TZ: Asia/Shanghai
    ports:
      - "80:80"
      # - "443:443"                       # HTTPS 开了证书再解注释
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/html:/usr/share/nginx/html:ro
      - ./nginx/logs:/var/log/nginx
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://127.0.0.1/health.html"]
      interval: 30s
      timeout: 3s
      retries: 2
    networks:
      - app-network

# ================ 自定义共享网络 ================
networks:
  app-network:
    driver: bridge
    # 生产可加 external: true 使用外部预先创建的网络
```

---

## 4. `.env` 环境变量文件示例

【强制】所有 **密码、端口、外部域名等可变配置** 一律放 `.env`，**禁止写死在 `docker-compose.yml` 里**。`.env` 需加入 `.gitignore`，只在团队内通过密钥管理系统分发。

```bash
# ==================================================================
# deploy/.env — Docker Compose 环境变量文件
# 说明：复制本文件为 .env 后填入真实值；本文件（示例版）可提交 .gitignore 前改名 .env.example
# ==================================================================

# ---------------- MySQL ----------------
MYSQL_ROOT_PWD=Root@2026Dev              # root 密码，复杂度达标
MYSQL_DB_NAME=example_db                  # 自动创建的业务库名
MYSQL_APP_USER=appuser                    # 业务专用账号
MYSQL_APP_PWD=AppUser@2026

# ---------------- Redis ----------------
REDIS_PWD=Redis@2026Dev

# ---------------- Spring Boot ----------------
APP_PROFILE=dev                            # prod / staging / dev
# 若从远程仓库拉镜像，用 image: registry.example.com/xxx/${APP_IMAGE_TAG}
APP_IMAGE_TAG=1.0.0
```

> 最佳实践：仓库里提交一份 `.env.example`（模板，无真实密码），新人复制为 `.env` 后自行填写；`.env` 本身必须在 `.gitignore` 中。

---

## 5. 各服务配置详解

### 5.1 MySQL：自定义 `my.cnf` + 初始化 SQL

**`deploy/mysql/conf/my.cnf`**（解决默认配置字符集、连接数、缓冲池等问题）：

```ini
[mysqld]
# ===== 字符集（8.x 默认已 utf8mb4，但显式声明更保险）=====
character-set-server = utf8mb4
collation-server = utf8mb4_general_ci
init_connect = 'SET NAMES utf8mb4'
skip-character-set-client-handshake

# ===== InnoDB 性能参数（按本地机器内存调，示例 2GB 内存笔记本）=====
innodb_buffer_pool_size = 512M            # 开发机给 1/4 内存即可，生产建议物理内存 60~80%
innodb_log_file_size = 128M
innodb_flush_log_at_trx_commit = 2        # 开发机可放宽为 2（性能好，crash 最多丢 1s）
sync_binlog = 0                           # 开发机关 binlog 同步刷盘

# ===== 连接 & 日志 =====
max_connections = 300                     # 开发环境够用
slow_query_log = ON                       # 慢查询日志开启，方便定位 SQL
slow_query_log_file = /var/lib/mysql/slow.log
long_query_time = 2                       # >2s 视为慢查询
log_queries_not_using_indexes = ON        # 未走索引的 SQL 也记录（开发阶段优化用）

[client]
default-character-set = utf8mb4

[mysql]
default-character-set = utf8mb4
```

**`deploy/mysql/initdb/01_init_schema.sql`**（首次启动自动执行，按字母顺序 01 / 02 / ...）：

```sql
-- 给业务用户授权
GRANT ALL PRIVILEGES ON example_db.* TO 'appuser'@'%';
FLUSH PRIVILEGES;

-- 切到业务库，建表（也可以交给 Flyway/Liquibase，二选一）
USE example_db;

CREATE TABLE IF NOT EXISTS t_user_info (
    id            BIGINT        NOT NULL AUTO_INCREMENT,
    username      VARCHAR(64)   NOT NULL,
    password_hash VARCHAR(255)  NOT NULL,
    status        TINYINT       NOT NULL DEFAULT 1,
    create_time   DATETIME(3)   NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    update_time   DATETIME(3)   NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    PRIMARY KEY (id),
    UNIQUE KEY uk_username (username)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='用户表（Compose 初始化示例）';

-- 插入测试数据
INSERT INTO t_user_info (username, password_hash, status) VALUES
    ('admin', '$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy', 1), -- password=admin123
    ('test',  '$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy', 1);
```

> 注意：`docker-entrypoint-initdb.d` 目录下的脚本 **仅首次容器启动（数据目录为空时）执行**。后续想重新跑初始化 SQL，需要先执行 `docker compose down -v` 清理数据卷（⚠️ 会清空现有数据，仅限开发环境）。

### 5.2 Redis：自定义 `redis.conf`

**`deploy/redis/conf/redis.conf`**：

```conf
# ===== 基本 =====
bind 0.0.0.0
port 6379
requirepass Redis@2026Dev          # 必须设置密码，和 .env 对齐

# ===== 持久化：AOF + RDB 双开（开发环境更保险）=====
appendonly yes
appendfsync everysec               # 每秒刷盘一次，性能与安全平衡
save 900 1                         # 900s 内至少 1 次写操作就做 RDB 快照
save 300 10
save 60 10000

# ===== 内存淘汰（开发环境防止把笔记本内存吃满）=====
maxmemory 256mb
maxmemory-policy allkeys-lru       # 超上限时按 LRU 淘汰任何 key

# ===== 日志 =====
loglevel notice
# logfile ""                        # 容器内直接 stdout，docker logs 查看即可
```

### 5.3 Nginx：反向代理 + 静态资源

**`deploy/nginx/conf.d/default.conf`**：

```nginx
server {
    listen 80;
    server_name localhost 127.0.0.1;

    # 访问日志格式（包含 $request_time 方便排查慢接口）
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" $request_time $upstream_response_time';
    access_log /var/log/nginx/access.log main;
    error_log  /var/log/nginx/error.log warn;

    client_max_body_size 50m;        # 文件上传大小限制

    # ================ 1. 前端静态资源 ================
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        # Vue/React 单页应用，刷新不 404 的关键：全部回退到 index.html
        try_files $uri $uri/ /index.html;

        # 静态资源缓存策略：js/css/image 缓存 7 天，html 不缓存
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2?)$ {
            expires 7d;
            add_header Cache-Control "public, max-age=604800, immutable";
        }
        location = /index.html {
            add_header Cache-Control "no-cache, no-store";
        }
    }

    # ================ 2. 后端 API 反向代理 ================
    location /api/ {
        # 去掉 /api 前缀再转发，后端路径是 /user/list 就访问 http://host/api/user/list
        rewrite ^/api/(.*)$ /$1 break;

        proxy_pass http://backend:8080;   # 直接用 compose 服务名 backend 当 host

        # ===== 超时设置（防止慢请求被 Nginx 提前掐断）=====
        proxy_connect_timeout 10s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;

        # ===== 传递真实客户端信息给后端 =====
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket 支持（若后端有 STOMP/SockJS）
        # proxy_http_version 1.1;
        # proxy_set_header Upgrade $http_upgrade;
        # proxy_set_header Connection "upgrade";
    }

    # ================ 3. Actuator 健康检查（仅内网访问）=====
    location = /health.html {
        access_log off;
        return 200 'ok';
        add_header Content-Type text/plain;
    }

    # 可选：Actuator 管理端口代理（仅限本机 127.0.0.1 调）
    # location /actuator/ {
    #     allow 127.0.0.1;
    #     deny all;
    #     proxy_pass http://backend:8081/actuator/;
    # }
}
```

---

## 6. 常用运维命令

> 所有命令 **都在 `deploy/` 目录下执行**。

| 场景 | 命令 | 说明 |
|------|------|------|
| **第一次启动** | `docker compose up -d` | 首次会自动 build 后端镜像 + pull 其他镜像 + 后台启动 |
| 只启动某些服务 | `docker compose up -d mysql redis` | 不传启动全部；传了只启动指定服务及依赖 |
| **强制重新 build 后端** | `docker compose build backend --no-cache` | Dockerfile 改了之后必须 rebuild，否则用旧镜像层 |
| build 后重启 backend | `docker compose up -d --build backend` | 最常用：改了代码 → 重新 build + 重启 backend 一个容器 |
| 查看日志（所有服务） | `docker compose logs -f --tail=200` | `-f` 实时滚动，`--tail=200` 从最近 200 行开始显示 |
| 查看某个服务日志 | `docker compose logs -f backend` | 只看后端 |
| 进入某容器 | `docker compose exec mysql bash` / `docker compose exec redis sh` | MySQL 进 bash，Alpine 镜像进 sh |
| 手动连接 MySQL | `docker compose exec mysql mysql -uroot -p` | 回车后输 `.env` 里的 root 密码 |
| 停止所有容器 | `docker compose stop` | 停止但不删除容器，下次 `start` 可恢复 |
| 启动已停止容器 | `docker compose start` | 对应 stop |
| 重启某个服务 | `docker compose restart backend` | 配置改了重启生效 |
| **停止并删除容器** | `docker compose down` | 最常用的"软清空"，**保留** 挂载的 data/logs 数据 |
| 停止 + 删除容器 + 删除卷 | `docker compose down -v` | ⚠️ **危险！** MySQL/Redis 持久化数据会被清空，仅想完全重置环境时用 |
| 查看容器状态 | `docker compose ps` | 看每个服务的健康状态 `(healthy)` / `(unhealthy)` |
| 查看镜像占用 | `docker images` + `docker system df` | 磁盘满了先看哪些镜像可清理 |
| 查看资源占用（top） | `docker stats` | 实时看各容器 CPU / MEM / NET / DISK 占用率 |
| 清理没用的镜像/缓存 | `docker system prune -a` | 磁盘清理（会删除所有未被容器使用的镜像） |

### 后端应用常见调试组合拳：

```bash
# 改了 Java 代码 → 重新 build backend → 重启 → 看日志
docker compose up -d --build backend && docker compose logs -f backend --tail=50
```

---

## 7. 生产环境差异与注意事项

> 再次强调：本文档是 **开发 / 测试环境** 实战。若要在 **生产** 临时用一下 Compose（不推荐，更建议上 K8s），至少做以下改造：

| 项目 | 开发/测试（本文档） | 生产改造建议 |
|------|-------------------|-------------|
| 端口暴露 | `3306:3306` / `6379:6379` 直接映射到宿主机 | MySQL/Redis **不对外映射端口**（`ports:` 行删掉），仅在 compose 内部网络访问；若必须远程连，走 VPN + SSH 隧道 |
| 数据挂载 | 相对路径 `./mysql/data` 到本机目录 | 用 **Docker named volume** 或挂载到独立的 `/data/mysql` 专用数据盘；定期做 xtrabackup 全量 + binlog 增量备份 |
| 资源限制 | 无限制（容易吃爆宿主机） | 每个服务加 `mem_limit: 4g`、`cpus: '2'` 等资源上限，防止单服务拖垮整机 |
| 副本数 | 各服务 1 个副本 | backend 用 `deploy.replicas: 3` + `nginx upstream` 做负载均衡（Compose 3.x deploy 节点，需 swarm mode） |
| 重启策略 | `unless-stopped` | backend 用 `always` + 健康检查；DB 等有状态服务慎重 `always`，防止反复 CrashLoop |
| 日志 | 挂载到本地目录即可 | 接 ELK / Loki，容器日志集中收集，本地不保留超过 7 天；配置 logrotate 轮转 |
| secrets 管理 | `.env` 明文 | 用 Docker Secrets（需 Swarm）或 Vault / K8s Secret，禁止明文密码进版本控制 |
| HTTPS | 不开 | 必须开，申请 Let's Encrypt 或购买证书，配置 Nginx 443 + HTTP 强制跳转 HTTPS |

---

## 8. 落地检查清单

| 检查项 | 级别 | 检查方法 | 责任人 | 完成状态 |
|--------|------|---------|--------|---------|
| 所有可变参数均在 `.env`，Compose 文件中无明文密码 / 硬编码端口 | 【强制】 | grep `docker-compose.yml` 无 `password=` / hardcode IP | 后端开发 | □ 未开始 |
| 各服务健康检查已配置，`docker compose ps` 全部显示 `(healthy)` | 【强制】 | 启动后执行 `docker compose ps` 核查 | 后端开发 | □ 未开始 |
| `depends_on.condition = service_healthy` 保证 DB/Redis 健康后再启后端 | 【强制】 | 检查 compose YAML depends_on 节点 | 技术负责人 | □ 未开始 |
| MySQL 配置了 `utf8mb4` 字符集、数据目录持久化、初始化 SQL 在 initdb.d 中 | 【强制】 | 连库执行 `SHOW VARIABLES LIKE 'char%'` 验证 | DBA | □ 未开始 |
| Redis 配置了密码 + AOF 持久化 + maxmemory 上限 | 【强制】 | 检查 `redis.conf`；redis-cli ping 需要密码才能通 | 后端开发 | □ 未开始 |
| Nginx 反向代理正确透传 X-Real-IP / X-Forwarded-For | 【推荐】 | 后端打印请求头确认拿到真实客户端 IP | 后端开发 | □ 未开始 |
| `.env` 和所有 `data/` `logs/` 目录已加入 `.gitignore` | 【强制】 | `git status` 无未追踪的 .env / data / log 文件 | 后端开发 | □ 未开始 |
| 新人入职执行一次 `docker compose up -d` 即可拉起全套环境 < 5 分钟 | 【推荐】 | 新人实操验证 | 技术负责人 | □ 未开始 |
| 生产环境（若不得不用 Compose）：端口不对外、资源有上限、数据有备份 | 【强制】 | 运维 checklist 签字确认 | 运维负责人 | □ 未开始 |

---

*本文档由 pangpang-doc 维护*
