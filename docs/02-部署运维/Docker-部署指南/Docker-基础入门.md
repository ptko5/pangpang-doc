# Docker 基础入门

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-08-15 |
| 适用范围 | 公司所有研发团队与运维 |
| 作者 | 架构组 |

> **用途**：掌握 Docker 核心概念（镜像/容器/仓库/数据卷/网络）、工作原理、与虚拟机对比，并通过 Hello World 快速上手，作为容器化部署的知识基础。

---

## 2. Docker 是什么

Docker 是一个容器化平台，将应用及其依赖打包到轻量级、可移植的容器中，实现「一次构建，到处运行」。

```text
┌─────────────────────────────┐
│        应用（App）           │
├─────────────────────────────┤
│      运行时依赖（JDK 25）    │
├─────────────────────────────┤
│    只读镜像层（Image）      │
├─────────────────────────────┤
│   Docker 引擎（共享内核）    │
├─────────────────────────────┤
│        宿主机 OS             │
└─────────────────────────────┘
```

> 容器共享宿主机内核，不包含完整操作系统，因此比虚拟机更轻、启动更快（毫秒级）。

---

## 3. 核心概念

| 概念 | 说明 | 类比 |
|------|------|------|
| **镜像（Image）** | 只读模板，包含应用与运行环境 | 安装包/类 |
| **容器（Container）** | 镜像的运行实例，可写层 | 运行中的进程/实例 |
| **仓库（Registry）** | 存放镜像的中心（如 Docker Hub、阿里云 ACR） | 代码仓库 |
| **数据卷（Volume）** | 持久化数据的目录 | 外接硬盘 |
| **网络（Network）** | 容器间通信的虚拟网络 | 局域网 |

### 3.1 镜像与容器关系

```text
镜像 Image（只读）
  └─> docker run 创建
        └─> 容器 Container（读写层 + 隔离进程）
```

> 【强制】镜像不可修改，容器的写操作只存在于可写层；需要保存修改时必须 `docker commit` 或重新构建镜像。

---

## 4. Docker 与虚拟机对比

| 维度 | Docker 容器 | 虚拟机 |
|------|------------|--------|
| 内核 | 共享宿主机内核 | 独立内核 |
| 启动速度 | 毫秒级 | 秒级~分钟级 |
| 资源占用 | 低（MB 级） | 高（GB 级） |
| 隔离性 | 进程级 | 完全隔离 |
| 镜像大小 | 小（几十 MB） | 大（GB 级） |

> **选型**：轻量服务、快速扩容用 Docker；强隔离、运行异构 OS 用虚拟机。

---

## 5. 安装与验证

```bash
# 安装 Docker Engine（Linux）
curl -fsSL https://get.docker.com | sh
systemctl enable --now docker

# 验证
docker version
docker run --rm hello-world   # 拉取并运行 Hello World
```

> macOS/Windows 请安装 Docker Desktop；Linux 下普通用户加入 docker 组即可免 sudo：`sudo usermod -aG docker $USER`。

---

## 6. 第一个容器

```bash
# 交互式运行 Ubuntu 并执行命令
docker run -it --rm ubuntu:22.04 bash

# 后台运行 Nginx
docker run -d --name web -p 8080:80 nginx

# 查看容器日志
docker logs web
```

---

## 7. 镜像构建（Dockerfile 入门）

```dockerfile
# 使用 JDK 25 基础镜像
FROM eclipse-temurin:25-jre

# 工作目录
WORKDIR /app

# 复制应用
COPY app.jar app.jar

# 暴露端口
EXPOSE 8080

# 启动命令
ENTRYPOINT ["java", "-jar", "app.jar"]
```

> 【推荐】生产镜像使用 `jre`（非 `jdk`）与多阶段构建，显著减小镜像体积。详见 [Dockerfile-编写规范.md](./Dockerfile-编写规范.md)。

---

## 8. 数据卷与网络

### 8.1 数据卷

```bash
# 挂载命名卷（持久化）
docker run -d -v mysql-data:/var/lib/mysql mysql:8.0

# 挂载宿主机目录（开发常用）
docker run -d -v /opt/app/logs:/app/logs myapp
```

### 8.2 自定义网络

```bash
# 创建网络
docker network create app-net

# 容器接入同一网络后可通过容器名互访
docker run -d --network app-net --name mysql mysql:8.0
docker run -d --network app-net --name app myapp
```

> 【推荐】生产使用自定义网络 + 容器名互访，避免依赖 IP。

---

## 9. 常用操作速览

| 操作 | 命令 |
|------|------|
| 查看镜像 | `docker images` |
| 查看运行容器 | `docker ps` |
| 进入容器 | `docker exec -it <name> bash` |
| 查看日志 | `docker logs -f <name>` |
| 停止/删除容器 | `docker stop <name>` / `docker rm <name>` |
| 删除无用镜像 | `docker image prune` |

> 完整命令集见 [Docker-常用命令.md](./Docker-常用命令.md)。

---

## 10. 落地检查清单

| 序号 | 检查项 | 检查方式 | 责任人 |
|------|--------|---------|--------|
| 1 | `docker version` 正常 | 命令验证 | 全栈 |
| 2 | `docker run --rm hello-world` 成功 | 命令验证 | 全栈 |
| 3 | 能构建并运行自定义镜像 | 构建验证 | 后端开发 |
| 4 | 数据卷持久化生效 | 容器重启后数据仍在 | 运维 |
| 5 | 自定义网络容器互访 | 容器名 ping 通 | 后端开发 |

---

**文档结束**

*本文档由 pangpang-doc 维护*
