# Redis 安装与配置

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-08-15 |
| 适用范围 | 公司所有后端研发团队与运维（Redis 7.x） |
| 作者 | 架构组 |

> **用途**：Redis 7.x 的安装（源码编译 + Docker）、持久化（RDB/AOF）、主从复制、哨兵高可用，以及常用命令速查，作为缓存与分布式锁等场景的标准参考。

---

## 2. 版本约定

| 项 | 约定 |
|----|------|
| **版本** | Redis 7.x |
| **端口** | 6379 |
| **持久化** | RDB + AOF 混合 |
| **高可用** | 哨兵（Sentinel）模式，生产禁用单机裸跑 |
| **使用规范** | 见 [缓存设计方案](../../03-技术笔记/架构设计/缓存设计方案.md) |

---

## 3. 安装方式一：源码编译

```bash
# 安装依赖
yum install -y gcc make tcl

# 下载编译
wget https://download.redis.io/releases/redis-7.2.5.tar.gz
tar -zxvf redis-7.2.5.tar.gz
cd redis-7.2.5
make -j4
make install PREFIX=/usr/local/redis

# 配置 systemd 服务
cat > /etc/systemd/system/redis.service <<'EOF'
[Unit]
Description=Redis Server
After=network.target

[Service]
ExecStart=/usr/local/redis/bin/redis-server /etc/redis/redis.conf
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now redis
```

---

## 4. 安装方式二：Docker

```bash
docker run -d \
  --name redis7 \
  -p 6379:6379 \
  -v redis-data:/data \
  redis:7.2

# 验证
docker exec -it redis7 redis-cli ping   # 返回 PONG
```

---

## 5. 核心配置（redis.conf）

### 5.1 持久化配置

```conf
# RDB 快照
save 900 1
save 300 10
save 60 10000

# AOF 追加（推荐开启）
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec

# 混合持久化（RDB + AOF 兼容）
aof-use-rdb-preamble yes
```

> 【推荐】生产开启 AOF（`everysec`）+ RDB 混合持久化，兼顾数据安全与恢复速度。

### 5.2 安全配置

```conf
# 设置访问密码（禁止空密码）
requirepass Pangpang@Redis2026

# 仅监听内网
bind 127.0.0.1 10.0.0.0/24
protected-mode yes
```

> 【强制】生产环境必须设置 `requirepass` 并限制 `bind`，禁止暴露公网 6379。

---

## 6. 主从复制与哨兵

### 6.1 主从复制（从库 redis.conf）

```conf
replicaof 10.0.0.1 6379
masterauth Pangpang@Redis2026
replica-read-only yes
```

### 6.2 哨兵配置（sentinel.conf）

```conf
sentinel monitor mymaster 10.0.0.1 6379 2
sentinel auth-pass mymaster Pangpang@Redis2026
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000
```

```bash
# 启动 3 个哨兵实例
redis-sentinel /etc/redis/sentinel.conf --port 26379
```

> 【推荐】生产至少部署 1 主 2 从 + 3 哨兵，保证故障自动切换（failover）。

---

## 7. 常用命令速查

| 场景 | 命令 |
|------|------|
| 连接并认证 | `redis-cli -a <password>` |
| 基础操作 | `SET` / `GET` / `DEL` / `EXPIRE` |
| Hash | `HSET` / `HGET` / `HGETALL` |
| List | `LPUSH` / `RPUSH` / `LRANGE` |
| Set / ZSet | `SADD` / `ZADD` / `ZRANGE` |
| 查看键 | `KEYS *`（生产禁用，改用 `SCAN`） |
| 监控 | `INFO` / `MONITOR` / `SLOWLOG GET` |

> 【强制】生产禁用 `KEYS *` 与 `FLUSHALL`（阻塞主线程）；大批量删除使用 `SCAN + DEL`。

---

## 8. 常见问题排查

| 问题现象 | 原因 | 解决方案 |
|---------|------|---------|
| `NOAUTH Authentication required` | 未认证 | `redis-cli -a <password>` |
| 键大量过期导致卡顿 | 过期键集中 | 过期时间加随机抖动（ttl + random） |
| 内存暴涨 | 无淘汰策略/大 key | 设置 `maxmemory` 与 `maxmemory-policy` |
| 主从数据不一致 | 网络分区/未开 AOF | 检查复制偏移量 `INFO replication` |
| 哨兵未切换 | 配置 quorum 不当 | 核对 `sentinel monitor` 参数 |

---

## 9. 落地检查清单

| 序号 | 检查项 | 检查方式 | 责任人 |
|------|--------|---------|--------|
| 1 | 版本为 7.x | `redis-server --version` | 运维 |
| 2 | 已设置访问密码 | `redis-cli -a <pwd> ping` | 后端开发/运维 |
| 3 | AOF 已开启 | `CONFIG GET appendonly` | 运维 |
| 4 | 生产为主从+哨兵 | `INFO replication` / `sentinel master mymaster` | 运维 |
| 5 | 内存淘汰策略已配置 | `CONFIG GET maxmemory-policy` | 运维 |

---

**文档结束**

*本文档由 pangpang-doc 维护*
