# Dockerfile 编写规范

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-08-15 |
| 适用范围 | 公司所有需要构建 Docker 镜像的后端 / 前端 / 基础服务项目（JDK 25 后端为默认适用对象） |
| 作者 | 架构组 |

---

## 2. 基础镜像选择规则

### 2.1 【强制】后端（JDK 25）基础镜像选型

| 场景 | 推荐镜像 | 标签说明 | 镜像体积 | 为什么选它 |
|------|---------|---------|---------|----------|
| **构建阶段（Build Stage）** | `eclipse-temurin:25-jdk-alpine` | 带完整 JDK，支持 `javac` / `jlink` | ~380MB | Eclipse Temurin（原 AdoptOpenJDK）是 Spring 官方推荐 JDK 发行版，Alpine 最小化 |
| **运行阶段（Runtime Stage）** | `eclipse-temurin:25-jre-alpine` | 只有 JRE，不含编译工具 | ~200MB | 运行时只需 JRE，比 JDK 镜像瘦 47%+；Alpine 用 musl libc，体积小、攻击面小 |
| 生产需要更多 Linux 工具（如 `curl` / `bash` 调试） | `eclipse-temurin:25-jre-jammy` | 基于 Ubuntu 22.04 LTS | ~440MB | 兼容性好，glibc 生态，适合排障要求高的团队（代价是体积翻倍） |

#### ❌ 禁止使用的基础镜像

- 【禁止】使用 `openjdk:25` / `openjdk:25-jdk-slim`：该仓库已废弃（Docker Hub 官方 2022 起停更），无人维护存在安全隐患。
- 【禁止】直接使用 `ubuntu` / `alpine` 裸镜像再自己装 JDK：重复造轮子、JDK 版本不可控、缺少 Temurin 的 JVM 调优补丁。
- 【禁止】使用带 `slim` 但非 Alpine 的镜像用于运行阶段：体积比 Alpine 大 100%+，无收益。

### 2.2 【推荐】前端基础镜像选型

| 阶段 | 推荐镜像 | 说明 |
|------|---------|------|
| 构建阶段 | `node:22-alpine` 或 `node:22-bookworm-slim` | Node.js 22 LTS；Alpine 体积小，Bookworm 原生模块编译兼容性好 |
| 运行阶段 | `nginx:1.27-alpine` | 托管静态资源，Nginx Alpine 镜像仅 ~10MB |

---

## 3. 多阶段构建最佳实践

### 3.1 为什么必须多阶段构建

【强制】所有 **生产 Dockerfile 必须使用多阶段构建（Multi-Stage Build）**，禁止单阶段构建。

- 单阶段构建的镜像会包含：源码、Gradle/Maven 依赖缓存、`.git` 目录、中间编译产物 → 体积 1.5GB+，拉取慢、攻击面大。
- 多阶段构建 = 「Build Stage 负责编译」 + 「Runtime Stage 只拷贝编译结果」→ 最终镜像 ~250MB（Spring Boot 典型项目）。

### 3.2 JDK 25 后端标准模板（推荐直接套用）

```dockerfile
# ============================================================
# Stage 1 / 2：构建阶段（Builder）—— 编译 + 打包 + 瘦身
# ============================================================
FROM eclipse-temurin:25-jdk-alpine AS builder

# 安装必要依赖（Alpine 默认无 bash，Gradle 部分场景需要）
RUN apk add --no-cache bash curl tzdata

# 设置构建时工作目录
WORKDIR /app

# 【优化点 1】先拷贝依赖描述文件，利用 Docker 层缓存
# 只要 build.gradle / pom.xml 不变，依赖层就能命中缓存，不用每次重新下载
# —— Gradle 项目写法 ——
COPY gradlew settings.gradle build.gradle ./
COPY gradle ./gradle
RUN ./gradlew dependencies --no-daemon || true  # 预下载依赖，失败不阻塞（允许部分快照）

# —— Maven 项目写法（二选一，删除不用的部分）——
# COPY pom.xml ./
# RUN mvn -B -e -C -T 1C org.apache.maven.plugins:maven-dependency-plugin:3.6.1:go-offline

# 【优化点 2】再拷贝源码
COPY src ./src

# 构建产物（跳过单元测试，单元测试应在 CI 构建阶段先跑通过）
# —— Gradle ——
RUN ./gradlew clean bootJar --no-daemon -x test -Duser.timezone=Asia/Shanghai

# —— Maven（二选一）——
# RUN mvn clean package -DskipTests -Duser.timezone=Asia/Shanghai

# 【优化点 3】Spring Boot 4.x 分层 Jar 模式（提取依赖层与应用层，加速后续构建）
# 从 Spring Boot 2.3+ 就支持 layered jar，这里把 jar 拆成 4 个目录：dependencies / spring-boot-loader / snapshot-dependencies / application
WORKDIR /app/build
RUN java -Djarmode=layertools -jar /app/build/libs/*.jar extract

# ============================================================
# Stage 2 / 2：运行阶段（Runtime）—— 最小化 + 最安全
# ============================================================
FROM eclipse-temurin:25-jre-alpine

# 【安全 1】设置时区为东八区（否则日志时间都是 UTC）
RUN apk add --no-cache tzdata curl \
    && cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo "Asia/Shanghai" > /etc/timezone \
    && apk del tzdata

ENV TZ=Asia/Shanghai \
    LANG=C.UTF-8 \
    JAVA_OPTS="" \
    SPRING_PROFILES_ACTIVE=prod

# 【安全 2】使用非 root 用户运行（生产强制）
RUN addgroup -S appgroup \
    && adduser -S appuser -G appgroup

WORKDIR /app

# 【优化点 4】按层拷贝 Spring Boot 分层 Jar 提取的目录
# 顺序：依赖层 → Spring 加载器 → SNAPSHOT 依赖 → 应用代码
# 依赖层变化频率最低，放最前面，最大化层缓存命中率
COPY --from=builder /app/build/dependencies/ ./
COPY --from=builder /app/build/spring-boot-loader/ ./
COPY --from=builder /app/build/snapshot-dependencies/ ./
COPY --from=builder /app/build/application/ ./

# 改目录权限给非 root 用户
RUN chown -R appuser:appgroup /app

USER appuser

# 暴露端口（Spring Boot 默认 8080，管理端口建议单独暴露 8081）
EXPOSE 8080 8081

# 【可靠性】健康检查：通过 Spring Boot Actuator /actuator/health 端点
# 启动 30s 后开始探测，每 10s 一次，超时 5s，连续 3 次失败判定不健康
HEALTHCHECK --interval=10s --timeout=5s --start-period=30s --retries=3 \
    CMD curl -fsS http://127.0.0.1:8081/actuator/health || exit 1

# 【启动入口】用 exec 形式启动，保证 PID 1 是 java 进程，能接收 SIGTERM 优雅停机
ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher" ]
```

> **Spring Boot 4.x 提示**：Spring Boot 4.x 的 `JarLauncher` 包路径从 `org.springframework.boot.loader.JarLauncher` 升级为 `org.springframework.boot.loader.launch.JarLauncher`，上面模板已适配；若仍使用 3.x，请改回旧路径。

---

## 4. 镜像体积优化手段

| 优化手段 | 说明 | 预计收益 |
|---------|------|---------|
| 多阶段构建（本章 §3） | Build 和 Runtime 分离，不把源码/构建工具打进最终镜像 | 单 Spring Boot 项目从 ~1.5GB → ~250MB，减 83% |
| Spring Boot 分层 Jar + 分层 COPY | 依赖层与应用层拆分 COPY，依赖层可缓存 | 每次构建后层缓存命中率 ↑，发布拉取时间 ↓ 40% |
| 合并 RUN 层 | 多条 `RUN` 合并为一条，减少层数；`&&` 串联，最后清理缓存 | 每合并 1 组可减 1 层，节省数 MB~数十 MB |
| `.dockerignore` | 排除 `.git/`、`node_modules/`、`build/`、`*.log` 等不相关文件 | 构建上下文大小 ↓ 90%+，`COPY .` 不再误塞多余文件 |
| Alpine 基础镜像 + `--no-cache` | 基础镜像最小；`apk add` 不缓存索引 | 比 Ubuntu slim 镜像小 200MB/层 |
| 清理构建残留 | Build 阶段不用清理（阶段即丢弃）；Runtime 阶段 `apk del` 卸载构建工具 | 节省数十 MB |
| Jlink 裁剪 JRE（进阶） | 用 `jlink` 生成只包含所需模块的自定义 JRE（适合模块清晰的项目） | JRE 体积从 ~150MB → ~50MB，再减 67% |

### 4.1 `.dockerignore` 示例（【强制】项目根目录必配）

```gitignore
# ===== Git / 版本控制 =====
.git
.gitignore
.github

# ===== Java / Gradle 构建产物 =====
build/
target/
.gradle/
*.class
*.jar
!gradle/wrapper/gradle-wrapper.jar

# ===== Maven 构建产物（二选一保留）=====
.m2/
repository/

# ===== 前端产物（若前后端同仓）=====
node_modules/
dist/
.vite/
.eslintcache

# ===== IDE / 编辑器 =====
.idea/
.vscode/
*.iml
*.iws
*.ipr

# ===== 日志 / 临时文件 =====
logs/
*.log
tmp/
temp/
.DS_Store
Thumbs.db

# ===== 环境配置（禁止打进镜像，应通过 ConfigMap/挂载注入）=====
.env
.env.local
*.env
application-local.yml
application-dev.yml
```

### 4.2 RUN 合并示例

✅ **正确示例**（一条 RUN 内完成所有操作 + 清理）：

```dockerfile
RUN apk add --no-cache bash curl tzdata \
    && cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo "Asia/Shanghai" > /etc/timezone \
    && apk del tzdata   # 用完就删，不留在最终镜像
```

❌ **反例**（每条命令一个 RUN，产生 4 层，体积臃肿）：

```dockerfile
RUN apk add --no-cache bash curl tzdata
RUN cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN echo "Asia/Shanghai" > /etc/timezone
RUN apk del tzdata
```

---

## 5. 健康检查（HEALTHCHECK）

### 5.1 【强制】生产镜像必须包含健康检查

没有 `HEALTHCHECK` 的容器，Docker / K8s 无法感知应用内部是否真的"活着"（进程在但死锁 = 假活），会导致流量打到不可用实例。

### 5.2 标准写法

**后端服务（Spring Boot Actuator）**：

```dockerfile
HEALTHCHECK --interval=10s \
            --timeout=5s \
            --start-period=30s \
            --retries=3 \
    CMD curl -fsS http://127.0.0.1:8081/actuator/health || exit 1
```

**前端 / Nginx 静态服务**：

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=2 \
    CMD wget -qO- http://127.0.0.1:80/health.html || exit 1
```

**中间件（如 Redis）**：

```dockerfile
HEALTHCHECK --interval=10s --timeout=3s --retries=3 \
    CMD redis-cli ping || exit 1
```

### 5.3 参数说明

| 参数 | 推荐值 | 含义 |
|------|-------|------|
| `--interval` | 后端 10s / 前端 30s | 两次健康检查之间的间隔 |
| `--timeout` | 3~5s | 单次检查超时时间，超过直接判失败 |
| `--start-period` | 后端 30~60s | 应用启动的宽限期，这段时间内检查失败不计入重试次数（**必须设，否则冷启动直接判 unhealthy**） |
| `--retries` | 3 | 连续失败多少次后判定 unhealthy |
| 命令退出码 | `0` = 健康，`1` = 不健康 | 禁止写 `exit 0` 走过场 |

---

## 6. 安全规范

### 6.1 【强制】非 root 用户运行

容器默认以 `root` 运行，一旦应用被 RCE 漏洞突破，攻击者直接拿到容器 root 权限 → 进一步逃逸到宿主机的风险极大。

✅ 标准写法（Alpine）：

```dockerfile
RUN addgroup -S appgroup \
    && adduser -S appuser -G appgroup
# ... 拷贝完文件后：
RUN chown -R appuser:appgroup /app
USER appuser
```

> Ubuntu / Debian 基础镜像用 `useradd -r -s /usr/sbin/nologin appuser`。

### 6.2 【禁止】敏感信息写进 Dockerfile

❌ **绝对禁止**以下写法：

```dockerfile
ENV DB_PASSWORD=my-secret-password-123     # 任何人 docker inspect 都能看到
ENV NACOS_TOKEN=sk-xxxxx                   # 密钥会被打进镜像层，历史层也能挖出来
COPY .env ./                                # .env 文件直接拷进去 = 裸奔
```

✅ **正确做法**：
- 运行时注入：`docker run -e DB_PASSWORD=xxx` 或 K8s `Secret` → `envFrom`
- 配置文件挂载：K8s `ConfigMap` / Docker `-v` 挂 `application-prod.yml` 到容器内
- CI/CD 系统用 Secrets 管理（GitHub Secrets / Jenkins Credentials），仅在部署阶段注入

### 6.3 【推荐】定期做镜像 CVE 扫描

- 用 `trivy` 做镜像漏洞扫描：`trivy image registry.example.com/user-service:1.0.0`
- 集成到 CI 流水线：发现 `HIGH / CRITICAL` 级漏洞时，**阻断发布**
- 基础镜像每季度升级一次（拉取最新 patch 版本），修复上游漏洞

---

## 7. Spring Boot 专项最佳实践

| 实践 | 说明 | 规范级别 |
|------|------|---------|
| 分层 Jar（`jarmode=layertools` extract + 分层 COPY） | 最大化 Docker 层缓存，发布更快 | 【强制】 |
| 分离应用端口与管理端口 | 业务 8080，Actuator 8081；8081 只在内网/监控网段暴露 | 【强制】 |
| `JAVA_OPTS` 环境变量接收 JVM 参数 | 堆大小、GC 参数通过 `docker run -e JAVA_OPTS="-Xms4g -Xmx4g ..."` 注入，不写死 | 【强制】 |
| `SPRING_PROFILES_ACTIVE` 默认 `prod` | 防止误以 dev/profile 启动到生产 | 【推荐】 |
| 用 JarLauncher 启动（非 `java -jar` 直接） | 配合分层 Jar，层加载方式更高效；Spring 4.x 路径已变 | 【推荐】 |
| `ENTRYPOINT` 用 exec 数组形式 | `["java", ...]` 而非 `java -jar app.jar` shell 形式，优雅停机生效（PID 1 收 SIGTERM） | 【强制】 |
| 配置虚拟线程参数 | JDK 25 虚拟线程已 GA，在 `JAVA_OPTS` 开启：`-Dspring.threads.virtual.enabled=true`（Spring Boot 4.x 默认支持） | 【推荐】 |

---

## 8. 常见陷阱与反例（正反对比表）

| 场景 | ❌ 错误写法 / 反模式 | ✅ 正确写法 |
|------|--------------------|-----------|
| 构建方式 | 单阶段 FROM 基础镜像 → 拷源码 → 编译 → 运行，镜像 1.5GB | 多阶段构建，Builder + Runtime 分离，镜像 ~250MB |
| COPY 顺序 | 一开始 `COPY . .`，改一行代码缓存全部失效 | 先拷 `build.gradle / pom.xml` → 下载依赖 → 再拷源码 |
| 运行用户 | 省略 `USER`，默认 root 跑进程 | 创建非 root 用户，`chown` 后 `USER appuser` |
| 敏感信息 | `ENV DB_PASSWORD=xxx` / `COPY .env` 写入镜像 | 运行时注入（`-e` / Secret / ConfigMap / 挂载） |
| 健康检查 | 省略，或写 `CMD exit 0` 走过场 | `curl actuator/health`，带合理的 start-period |
| 启动方式 | `ENTRYPOINT java -jar app.jar`（shell 形式） | `ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS ...JarLauncher"]` |
| 时区 | 省略，日志全部 UTC 时间差 8 小时 | 设置 `TZ=Asia/Shanghai` + 拷贝 `localtime` 文件 |
| RUN 层数 | 每条命令一个 `RUN apk add` / `RUN yum install` | 合并一条 RUN，最后清理缓存 |
| .dockerignore | 不配，把 `.git/` / `build/` / `logs/` 全塞进上下文 | 根目录必须有 `.dockerignore`，排除非必要文件 |
| 镜像版本标签 | 全部打 `:latest`，无法追溯版本 | 打语义化版本 `:1.2.3-prod` + Git Commit Short SHA |

---

## 9. 完整示例总结（JDK 25 + Gradle 多阶段构建最终版）

> 就是 §3.2 的模板，在此再次强调：**新项目直接复制 §3.2 代码模板用，不要自己从零写 Dockerfile**。已帮大家踩过所有常见坑。

---

## 10. 落地检查清单

| 检查项 | 级别 | 检查方法 | 责任人 | 完成状态 |
|--------|------|---------|--------|---------|
| 使用多阶段构建（Build + Runtime 至少 2 阶段） | 【强制】 | `grep -c '^FROM' Dockerfile` 结果 ≥ 2 | 后端开发 | □ 未开始 |
| 基础镜像用 `eclipse-temurin:25-jre-alpine`（或等价 Jammy），非废弃 openjdk 镜像 | 【强制】 | 检查 `FROM` 行 | 后端开发 | □ 未开始 |
| 生产镜像以非 root 用户运行（存在 `USER appuser` 或等价语句） | 【强制】 | grep `USER` + `adduser`/`useradd` | 安全负责人 | □ 未开始 |
| 无敏感信息硬编码（grep `ENV` 无密码/Token，无 `COPY .env`） | 【强制】 | `grep -Ei 'password|token|secret|ak|sk' Dockerfile` 无命中 | 安全负责人 | □ 未开始 |
| 包含合理的 HEALTHCHECK（start-period ≥ 30s，非 exit 0） | 【强制】 | 检查 `HEALTHCHECK` 指令 | 后端开发 | □ 未开始 |
| Spring Boot 使用分层 Jar + 分层 COPY | 【推荐】 | 检查 `jarmode=layertools` + 4 次分层 COPY | 技术负责人 | □ 未开始 |
| 项目根目录存在 `.dockerignore` | 【强制】 | `ls -a` 核对文件 | 后端开发 | □ 未开始 |
| RUN 指令尽量合并，同一条内包含「安装 + 清理」 | 【推荐】 | Code Review | 技术负责人 | □ 未开始 |
| 镜像已通过 `trivy` 扫描，无 HIGH/CRITICAL CVE | 【强制】 | CI 流水线 trivy 报告 | 运维 + 安全 | □ 未开始 |
| 镜像标签使用语义化版本 + SHA，非全量 `:latest` | 【推荐】 | CI 构建脚本检查 | DevOps | □ 未开始 |

---

*本文档由 pangpang-doc 维护*
