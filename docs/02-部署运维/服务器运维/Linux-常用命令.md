# Linux 常用命令

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-08-15 |
| 适用范围 | 公司所有研发团队与运维（CentOS 7+/Ubuntu 20+） |
| 作者 | 架构组 |

> **用途**：Linux 常用命令速查，覆盖系统、文件、网络、进程、文本、权限、性能 7 大类 80+ 命令，并提供组合命令手册。

---

## 2. 系统信息

| 操作 | 命令 |
|------|------|
| 查看系统版本 | `cat /etc/os-release` / `uname -a` |
| 查看内核版本 | `uname -r` |
| 查看运行时间 | `uptime` |
| 查看 CPU | `lscpu` |
| 查看内存 | `free -h` |
| 查看磁盘 | `df -h` |
| 查看磁盘 IO | `iostat -x 1` |
| 查看系统负载 | `top` / `htop` |

---

## 3. 文件与目录

| 操作 | 命令 |
|------|------|
| 列出目录 | `ls -lah` |
| 切换目录 | `cd /path` |
| 创建目录 | `mkdir -p a/b/c` |
| 复制 | `cp -r src dst` |
| 移动/重命名 | `mv old new` |
| 删除 | `rm -rf dir`（慎用） |
| 查看文件 | `cat file` / `less file` |
| 查找文件 | `find / -name "*.log"` |
| 统计大小 | `du -sh *` |

> 【强制】`rm -rf` 必须确认目标路径，禁止对根目录/关键目录误删；高危命令先 `ls` 复核。

---

## 4. 网络

| 操作 | 命令 |
|------|------|
| 查看端口监听 | `ss -tlnp` |
| 测试连通性 | `ping <host>` |
| 查看路由 | `ip route` |
| 抓包 | `tcpdump -i eth0 port 8080` |
| 下载文件 | `wget url` / `curl -O url` |
| 查看 DNS | `nslookup <domain>` |
| 查看连接数 | `ss -s` |

```bash
# 组合：查看某端口被哪个进程占用
ss -tlnp | grep 8080
lsof -i :8080
```

---

## 5. 进程管理

| 操作 | 命令 |
|------|------|
| 查看进程 | `ps -ef` |
| 按名查找 | `pgrep -f java` |
| 结束进程 | `kill <pid>` / `kill -9 <pid>` |
| 查看进程资源 | `top -p <pid>` |
| 后台运行 | `nohup cmd &` |
| 查看线程 | `ps -T -p <pid>` |

```bash
# 组合：查看 Java 进程启动参数
jps -l
cat /proc/<pid>/cmdline
```

---

## 6. 文本处理

| 操作 | 命令 |
|------|------|
| 过滤 | `grep -n "error" app.log` |
| 统计 | `wc -l file` |
| 排序去重 | `sort \| uniq -c` |
| 截取列 | `awk '{print $1}'` |
| 替换 | `sed -i 's/old/new/g' file` |
| 分页查看 | `less file` |
| 实时查看 | `tail -f app.log` |

```bash
# 组合：统计日志中 ERROR 数量
grep -c "ERROR" app.log

# 组合：查看访问量 Top10 IP
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10
```

---

## 7. 权限与用户

| 操作 | 命令 |
|------|------|
| 修改权限 | `chmod 755 file` |
| 修改属主 | `chown user:group file` |
| 查看当前用户 | `whoami` |
| 切换用户 | `su - user` |
| 添加用户 | `useradd -m user` |
| 设置密码 | `passwd user` |
| 查看 sudo 权限 | `sudo -l` |

---

## 8. 性能排查组合命令

```bash
# 1. 系统整体负载
top
uptime

# 2. 内存
free -h
# 3. CPU 占用 Top 进程
top -o %CPU | head -20
# 4. 磁盘
df -h
iostat -x 1
# 5. 网络
ss -s
# 6. 定位 Java 高 CPU 线程
top -H -p <pid>
jstack <pid> | grep -A 20 "nid=0x<线程id>"
```

> 【推荐】线上问题按「负载 → 内存 → CPU → 磁盘 → 网络」顺序排查，详见 [线上故障排查](../../03-技术笔记/问题排查/线上故障排查.md)。

---

## 9. 落地检查清单

| 序号 | 检查项 | 检查方式 | 责任人 |
|------|--------|---------|--------|
| 1 | 常用命令可熟练使用 | 命令演练 | 全栈 |
| 2 | 端口/进程定位熟练 | `ss` + `ps` | 运维 |
| 3 | 文本处理组合命令可用 | 日志分析演练 | 后端开发 |
| 4 | 权限管理规范执行 | 权限审查 | 运维 |
| 5 | 性能排查路径明确 | 模拟故障演练 | 运维 |

---

**文档结束**

*本文档由 pangpang-doc 维护*
