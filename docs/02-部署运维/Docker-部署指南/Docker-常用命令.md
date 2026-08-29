# Docker 常用命令

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-08-15 |
| 适用范围 | 公司所有研发团队与运维 |
| 作者 | 架构组 |

> **用途**：Docker 常用命令速查，覆盖镜像、容器、数据卷、网络、日志、清理 6 大类 40+ 命令，并提供组合使用技巧。入门概念见 [Docker-基础入门.md](./Docker-基础入门.md)。

---

## 2. 镜像命令

| 操作 | 命令 |
|------|------|
| 列出镜像 | `docker images` |
| 拉取镜像 | `docker pull <image>:<tag>` |
| 构建镜像 | `docker build -t <name>:<tag> .` |
| 标记镜像 | `docker tag <image> <registry>/<name>:<tag>` |
| 推送镜像 | `docker push <registry>/<name>:<tag>` |
| 删除镜像 | `docker rmi <image>` |
| 查看镜像详情 | `docker inspect <image>` |
| 查看镜像历史 | `docker history <image>` |

---

## 3. 容器命令

| 操作 | 命令 |
|------|------|
| 创建并运行（前台） | `docker run -it --rm <image> bash` |
| 创建并运行（后台） | `docker run -d --name <name> -p 8080:80 <image>` |
| 列出运行中容器 | `docker ps` |
| 列出全部容器 | `docker ps -a` |
| 停止容器 | `docker stop <name>` |
| 启动容器 | `docker start <name>` |
| 重启容器 | `docker restart <name>` |
| 进入容器 | `docker exec -it <name> bash` |
| 删除容器 | `docker rm <name>` |
| 复制文件进出 | `docker cp <name>:/path ./` |
| 查看进程 | `docker top <name>` |

---

## 4. 数据卷命令

| 操作 | 命令 |
|------|------|
| 创建数据卷 | `docker volume create <vol>` |
| 列出数据卷 | `docker volume ls` |
| 查看卷详情 | `docker volume inspect <vol>` |
| 删除未使用卷 | `docker volume prune` |
| 挂载运行 | `docker run -v <vol>:/data <image>` |

---

## 5. 网络命令

| 操作 | 命令 |
|------|------|
| 创建网络 | `docker network create <net>` |
| 列出网络 | `docker network ls` |
| 查看网络详情 | `docker network inspect <net>` |
| 连接网络 | `docker network connect <net> <name>` |
| 断开网络 | `docker network disconnect <net> <name>` |
| 删除未使用网络 | `docker network prune` |

---

## 6. 日志与资源命令

| 操作 | 命令 |
|------|------|
| 查看日志（跟随） | `docker logs -f <name>` |
| 查看最近 100 行日志 | `docker logs --tail 100 <name>` |
| 查看资源占用 | `docker stats` |
| 查看容器内进程 | `docker top <name>` |
| 查看事件 | `docker events` |

---

## 7. 清理命令

| 操作 | 命令 |
|------|------|
| 清理停止的容器 | `docker container prune` |
| 清理无用镜像 | `docker image prune -a` |
| 清理无用卷 | `docker volume prune` |
| 一键清理（容器/镜像/网络） | `docker system prune -af` |
| 查看占用 | `docker system df` |

> 【推荐】定期执行 `docker system df` 检查占用，生产环境清理前先确认无在用镜像与卷。

---

## 8. 组合命令技巧

```bash
# 停止并删除所有容器
docker stop $(docker ps -q) && docker rm $(docker ps -aq)

# 删除所有悬空镜像
docker rmi $(docker images -f "dangling=true" -q)

# 进入容器并查看端口映射
docker exec -it $(docker ps -qf "name=web") sh

# 实时查看某容器最后 200 行日志并跟随后续
docker logs -f --tail 200 <name>

# 只运行一次并自动清理（排查环境）
docker run -it --rm -v "$PWD":/work -w /work gradle:8.10-jdk25 gradle build
```

---

## 9. 落地检查清单

| 序号 | 检查项 | 检查方式 | 责任人 |
|------|--------|---------|--------|
| 1 | 常用镜像命令熟练 | 命令行演练 | 全栈 |
| 2 | 容器生命周期管理正确 | 起停删演练 | 全栈 |
| 3 | 数据卷持久化验证 | 重启容器数据保留 | 运维 |
| 4 | 网络互访验证 | 容器名连通 | 后端开发 |
| 5 | 定期清理无效资源 | `docker system df` | 运维 |

---

**文档结束**

*本文档由 pangpang-doc 维护*
