# MySQL 安装与配置

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-08-15 |
| 适用范围 | 公司所有后端研发团队与运维（MySQL 8.x） |
| 作者 | 架构组 |

> **用途**：MySQL 8.x 的安装（Linux 二进制 + Docker 两种方式）、utf8mb4 字符集配置、初始化、用户权限管理与备份恢复，作为开发/测试环境的标准参考。

---

## 2. 版本约定

| 项 | 约定 |
|----|------|
| **版本** | MySQL 8.x |
| **字符集** | utf8mb4（全局统一） |
| **默认引擎** | InnoDB |
| **端口** | 3306 |
| **使用规范** | 见 [SQL与数据访问规范](../../00-开发规范/后端开发规范/SQL与数据访问规范.md) |

> 【强制】所有库、表、连接统一使用 `utf8mb4` 字符集（`utf8mb4_0900_ai_ci` 排序规则），避免 emoji 与生僻字乱码。

---

## 3. 安装方式一：Linux 二进制安装

### 3.1 下载并解压

```bash
# 使用国内镜像下载（版本号以官方最新 8.x 为准）
wget https://mirrors.aliyun.com/mysql/MySQL-8.0/mysql-8.0.40-linux-glibc2.28-x86_64.tar.xz
tar -xvf mysql-8.0.40-linux-glibc2.28-x86_64.tar.xz
mv mysql-8.0.40-linux-glibc2.28-x86_64 /usr/local/mysql
```

### 3.2 创建用户与数据目录

```bash
groupadd mysql
useradd -r -g mysql -s /sbin/nologin mysql
mkdir -p /usr/local/mysql/data
chown -R mysql:mysql /usr/local/mysql
```

### 3.3 初始化与启动

```bash
# 初始化数据目录（8.x 使用 --initialize-insecure 生成空 root 密码）
/usr/local/mysql/bin/mysqld --initialize-insecure --user=mysql --datadir=/usr/local/mysql/data

# 配置 systemd 服务并启动
cat > /etc/systemd/system/mysqld.service <<'EOF'
[Unit]
Description=MySQL Server
After=network.target

[Service]
User=mysql
ExecStart=/usr/local/mysql/bin/mysqld --datadir=/usr/local/mysql/data
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now mysqld
```

---

## 4. 安装方式二：Docker 安装

> 开发环境推荐 Docker 方式，秒级起停、版本可控。

```bash
docker run -d \
  --name mysql8 \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD='Pangpang@2026' \
  -e MYSQL_CHARSET=utf8mb4 \
  -v mysql-data:/var/lib/mysql \
  mysql:8.0

# 进入容器
docker exec -it mysql8 mysql -uroot -p
```

---

## 5. 字符集与基础配置

### 5.1 配置文件 /etc/my.cnf

```ini
[mysqld]
port = 3306
character-set-server = utf8mb4
collation-server = utf8mb4_0900_ai_ci
default-storage-engine = InnoDB
max_connections = 500
lower_case_table_names = 1
sql_mode = STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION
binlog_format = ROW
server-id = 1
log-bin = mysql-bin
```

### 5.2 验证字符集

```sql
SHOW VARIABLES LIKE 'character_set%';
SHOW VARIABLES LIKE 'collation%';
```

> 【强制】`character_set_server` 必须为 `utf8mb4`；新建库表时如无特殊需求不再单独指定字符集。

---

## 6. 初始化与用户权限

### 6.1 设置 root 密码

```bash
mysql -uroot -p
```

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'Pangpang@2026';
FLUSH PRIVILEGES;
```

### 6.2 创建业务账号（最小权限原则）

```sql
-- 创建应用账号，仅授权业务库
CREATE USER 'app'@'%' IDENTIFIED BY 'App@2026';
GRANT SELECT, INSERT, UPDATE, DELETE ON pangpang_db.* TO 'app'@'%';
FLUSH PRIVILEGES;

-- 创建只读账号（供报表/BI 使用）
CREATE USER 'readonly'@'%' IDENTIFIED BY 'Read@2026';
GRANT SELECT ON pangpang_db.* TO 'readonly'@'%';
FLUSH PRIVILEGES;
```

> 【强制】禁止应用账号使用 root；禁止 `GRANT ALL ON *.*` 超权限授权；密码强度必须满足复杂策略。

---

## 7. 备份与恢复

### 7.1 逻辑备份（mysqldump）

```bash
# 全量备份
mysqldump -uroot -p --single-transaction --master-data=2 pangpang_db > pangpang_db_$(date +%Y%m%d).sql

# 恢复
mysql -uroot -p pangpang_db < pangpang_db_20260815.sql
```

### 7.2 物理备份（xtrabackup，推荐生产）

```bash
# 全量备份
xtrabackup --backup --target-dir=/backup/mysql/full --user=root --password=***

# 增量备份
xtrabackup --backup --target-dir=/backup/mysql/inc1 --incremental-basedir=/backup/mysql/full
```

> 【推荐】生产环境使用 xtrabackup 物理备份 + binlog 增量；开发环境 mysqldump 即可。备份必须做恢复演练，且异地留存。

---

## 8. 常见问题排查

| 问题现象 | 原因 | 解决方案 |
|---------|------|---------|
| 连接报 `Access denied` | 密码/权限错误 | 核对用户主机域与 `GRANT` |
| 中文乱码 | 字符集非 utf8mb4 | 全局配置 + `ALTER TABLE ... CONVERT TO CHARACTER SET utf8mb4` |
| 无法远程连接 | bind-address 限制 | 配置 `bind-address=0.0.0.0` 并放行防火墙 3306 |
| `Too many connections` | 连接数打满 | 检查连接泄漏；适当调大 `max_connections` |
| 主从延迟 | 大事务/DDL | 拆分大事务、启用并行复制 |

---

## 9. 落地检查清单

| 序号 | 检查项 | 检查方式 | 责任人 |
|------|--------|---------|--------|
| 1 | 版本为 8.x | `SELECT VERSION()` | 后端开发/运维 |
| 2 | 字符集为 utf8mb4 | `SHOW VARIABLES LIKE 'character_set%'` | DBA |
| 3 | 业务账号最小权限 | `SHOW GRANTS FOR 'app'@'%'` | DBA |
| 4 | root 禁止远程登录 | 权限核对 | DBA |
| 5 | 备份策略与恢复演练 | 备份脚本 + 演练记录 | 运维 |

---

**文档结束**

*本文档由 pangpang-doc 维护*
