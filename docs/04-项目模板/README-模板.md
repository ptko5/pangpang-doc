# README 模板

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-08-15 |
| 适用范围 | 公司所有新建项目（后端 + 前端 + 全栈） |
| 作者 | 架构组 |

---

## 2. 模板使用说明

本模板提供项目 README 的标准结构。**使用方法**：
1. 复制本文件到你的项目根目录，重命名为 `README.md`
2. 将所有 `<!-- 使用者请替换：xxx -->` 注释中的占位内容替换为你的项目实际信息
3. 不适用的章节（如纯前端项目的「后端技术栈」）可整节删除，不必留空

---

## 3. 项目简介

<!-- 使用者请替换：用 2-3 句话讲清楚项目做什么、给谁用、核心价值 -->
**项目名称**：XXX 管理系统
**一句话描述**：面向 XXX 业务场景的一体化管理平台，覆盖 A、B、C 三大核心模块
**服务对象**：运营人员 / 商家 / 消费者 / 内部员工
**上线时间**：2026-XX-XX（规划中 / 已上线）

### 3.1 核心特性

- ✅ 特性一：例如「基于 JDK 25 虚拟线程，接口吞吐量提升 3 倍」
- ✅ 特性二：例如「前后端分离 + React 19 + TypeScript 5，强类型端到端」
- ✅ 特性三：例如「集成 Nacos 配置中心 + Sentinel 熔断降级，生产可用」
- ✅ 特性四：例如「完善的操作审计与权限控制，符合等保合规」

---

## 4. 技术栈

### 4.1 后端技术栈

<!-- 使用者请替换：根据实际选型调整版本号；未使用的组件可删除 -->
| 组件 | 版本 | 说明 |
|------|------|------|
| **JDK** | 25 | 核心语言环境（虚拟线程 + 结构化并发） |
| **Spring Boot** | 4.x | 应用框架 |
| **Spring Cloud** | 2024.x | 微服务框架（服务发现、配置中心） |
| **Spring Data JPA** | 3.x | ORM 框架（或替换为 MyBatis-Plus） |
| **Gradle** | 8.x | 构建工具（或替换为 Maven 3.9+） |
| **MySQL** | 8.x | 关系型数据库 |
| **Redis** | 7.x | 缓存 / 分布式锁 |
| **Nacos** | 2.x | 注册中心 + 配置中心 |
| **Sentinel** | 2.x | 熔断降级 + 限流 |
| **RabbitMQ / Kafka** | 3.x / 3.7 | 消息队列（按需二选一） |

### 4.2 前端技术栈

| 组件 | 版本 | 说明 |
|------|------|------|
| **React** | 19.x | UI 框架 |
| **TypeScript** | 5.x | 类型系统 |
| **Ant Design** | 5.x | UI 组件库 |
| **Vite** | 6.x | 构建工具 |
| **Zustand** | 4.x | 状态管理 |
| **Axios** | 1.x | HTTP 客户端 |
| **React Router** | 6.x | 路由管理 |

### 4.3 部署与运维栈

| 组件 | 用途 |
|------|------|
| **Docker** | 容器化部署 |
| **Kubernetes (K8s)** | 容器编排（生产环境） |
| **Nginx** | 反向代理 + 静态资源服务 |
| **Prometheus + Grafana** | 指标监控 + 可视化 |
| **ELK (Filebeat + ES + Kibana)** | 日志采集与分析 |

---

## 5. 目录结构

### 5.1 后端标准目录结构（7 层分层架构）

<!-- 使用者请替换：如果你的项目不是 7 层架构，按实际情况调整 -->
```
backend/
├── api/                                    # API 接口定义层
│   └── src/main/java/com/example/xxx/api/
│       ├── XxxApi.java                     # API 接口声明（Feign 共享）
│       ├── request/                         # API 请求对象
│       └── response/                        # API 响应对象
├── common/                                 # 通用组件层
│   └── src/main/java/com/example/common/
│       ├── config/                          # 通用配置类
│       ├── exception/                       # 通用异常类
│       ├── interceptor/                     # 通用拦截器
│       ├── model/                           # 通用模型（ApiResponse / PageResult）
│       └── util/                            # 工具类
├── service/                                # 业务逻辑层
│   └── src/main/java/com/example/xxx/service/
│       ├── XxxService.java                  # 业务接口
│       └── impl/                            # 业务实现
├── infrastructure/                         # 基础设施层
│   └── src/main/java/com/example/xxx/infrastructure/
│       ├── repository/                      # 数据访问层
│       ├── entity/                          # 数据库实体
│       └── config/                          # 基础设施配置
├── adapter/                                # 适配器层
│   └── src/main/java/com/example/xxx/adapter/
│       ├── controller/                      # REST 控制层
│       ├── client/                          # 外部服务客户端（Feign）
│       └── messaging/                       # 消息队列生产/消费
├── application/                            # 应用层
│   └── src/main/java/com/example/xxx/application/
│       └── command/                         # 命令处理器 / 用例编排
└── bootstrap/                              # 启动引导层
    └── src/main/
        ├── java/com/example/xxx/XxxApplication.java   # Spring Boot 启动类
        └── resources/
            ├── application.yml              # 主配置
            ├── application-dev.yml          # 开发环境
            ├── application-test.yml         # 测试环境
            ├── application-prod.yml         # 生产环境
            └── logback-spring.xml           # 日志配置
```

### 5.2 前端标准目录结构

```
frontend/
├── src/
│   ├── components/                          # 通用组件
│   │   ├── layout/                          # 布局组件（Header / Sidebar / Layout）
│   │   ├── common/                          # 公共组件（Button / Table / Form）
│   │   └── business/                        # 业务组件
│   ├── pages/                               # 页面组件（按业务模块分子目录）
│   ├── hooks/                               # 自定义 Hooks
│   ├── stores/                              # Zustand 状态管理（*.store.ts）
│   ├── api/                                 # API 接口封装（*.api.ts）
│   ├── utils/                               # 工具函数
│   ├── types/                               # TypeScript 类型定义
│   ├── routes/                              # 路由配置
│   ├── styles/                              # 全局样式（variables / mixins / global）
│   ├── App.tsx
│   └── main.tsx
├── public/                                  # 静态资源
├── tests/                                   # 测试（unit / e2e）
├── .eslintrc.cjs
├── .prettierrc
├── tsconfig.json
├── vite.config.ts
└── package.json
```

---

## 6. 环境要求

| 环境 | 最低版本 | 推荐版本 |
|------|---------|---------|
| JDK | 25 | 25.0.1+ |
| Node.js | 20 LTS | 22 LTS |
| npm / pnpm | 9 / 8 | 10 / 9 |
| MySQL | 8.0 | 8.0.36+ |
| Redis | 6.2 | 7.2 |
| Docker | 24 | 25+（含 BuildKit） |
| 操作系统 | macOS 13 / Ubuntu 22.04 / Windows 11 WSL2 | 同左 |

---

## 7. 快速开始

### 7.1 克隆代码

```bash
# 使用 SSH（推荐）
git clone git@github.com:<组织>/<仓库名>.git
cd <仓库名>

# 或使用 HTTPS
git clone https://github.com/<组织>/<仓库名>.git
```

### 7.2 初始化中间件依赖

<!-- 使用者请替换：如果不使用 Docker 启动中间件，改为手动安装步骤 -->
**方式一：Docker Compose 一键启动（推荐本地开发）**

```bash
cd deploy/docker-compose
docker compose up -d mysql redis nacos
# 等待 30s，确认容器健康
docker compose ps
```

**方式二：本地原生安装**
参考文档：[环境搭建指南](../../01-环境搭建/README.md)

### 7.3 启动后端

```bash
cd backend

# Gradle 构建 + 运行（推荐 JDK 25）
./gradlew :bootstrap:bootRun --args='--spring.profiles.active=dev'

# 或 Maven 方式
cd bootstrap && mvn spring-boot:run -Dspring-boot.run.profiles=dev
```

验证：访问 `http://localhost:8080/actuator/health`，返回 `{"status":"UP"}`。

### 7.4 启动前端

```bash
cd frontend

# 安装依赖（首次）
pnpm install   # 或 npm install

# 开发模式启动
pnpm dev       # 或 npm run dev
```

验证：访问 `http://localhost:5173`，默认登录页可正常渲染。

### 7.5 运行测试

```bash
# 后端测试
cd backend && ./gradlew test

# 前端测试
cd frontend && pnpm test:unit      # 单元测试
cd frontend && pnpm test:e2e       # E2E 测试
```

---

## 8. 部署说明

### 8.1 开发环境 → 测试环境

参考 [CI/CD 流水线指南](../02-部署运维/CI-CD-流水线/README.md)，通过 Git 提交触发 Jenkins/GitHub Actions 自动构建并部署到测试环境。

### 8.2 生产环境部署

1. 通过 `release/x.y.z` 分支创建 MR，走完整发布流程（详见 [发布流程规范](../02-部署运维/部署与运维规范.md) §4）
2. 构建 Docker 镜像并推送到私有镜像仓库
3. 使用 Helm Chart 部署到 K8s 生产集群
4. 按 **5% → 20% → 50% → 100%** 金丝雀发布策略分批放量

---

## 9. 常见问题 (FAQ)

### Q1: 后端启动报错 `Communications link failure`（MySQL 连接失败）
**A**: 检查以下 3 项：
1. MySQL 容器是否启动：`docker ps | grep mysql`
2. MySQL 端口 3306 是否在本地被占用：`ss -lntp | grep 3306`
3. `application-dev.yml` 中的用户名密码是否与 docker-compose.yml 中一致

### Q2: 前端启动后接口全部 404
**A**: 检查 `.env.development` 中 `VITE_API_BASE_URL` 是否指向正确的后端地址（默认 `http://localhost:8080`），且 Vite 代理配置未被修改。

### Q3: 如何新增一个业务模块（从 DB 到页面）
**A**: 参考 [后端开发规范](../00-开发规范/后端开发规范/Java-编码规范.md) + [前端开发规范](../00-开发规范/前端开发规范/Vue-React-规范.md) 的步骤说明。

### Q4: <!-- 使用者请替换：根据你的项目再补充 2-3 个最常见的问题 -->

---

## 10. 贡献指南

### 10.1 分支管理
严格遵循 [Git-工作流规范](../00-开发规范/Git-工作流规范.md)：
- 新功能 → 从 `develop` 创建 `feature/xxx` 分支
- Bug 修复 → 从 `develop` 创建 `fix/xxx` 分支
- 所有变更必须通过 MR，不允许直接推送到 `main` / `develop`

### 10.2 代码审查 (MR 评审)
- 参考 [代码审查规范](../00-开发规范/代码审查规范.md)
- 单个 MR 代码量建议 ≤ 500 行
- 必须指定至少 1 位模块负责人作为评审人

### 10.3 文档同步
- 新增/修改 API：同步更新 [API-接口文档模板](../04-项目模板/API-接口文档模板.md)
- 新增数据库表：同步更新 [数据库设计文档](../04-项目模板/数据库设计文档模板.md)
- 技术方案调整：同步更新对应 [技术方案文档](../04-项目模板/技术方案文档模板.md)

---

## 11. 联系方式

| 角色 | 负责人 | 联系方式（飞书/邮箱） |
|------|--------|---------------------|
| 项目负责人（PM） | XXX | xxx@example.com |
| 架构师 | XXX | xxx@example.com |
| 后端 TL | XXX | xxx@example.com |
| 前端 TL | XXX | xxx@example.com |
| 测试 TL | XXX | xxx@example.com |
| 运维 SRE | XXX | xxx@example.com |

---

## 12. 落地检查清单

| 序号 | 检查项 | 级别 | 状态 |
|------|--------|------|------|
| 1 | 「项目简介」是否填写了名称、描述、服务对象 | 强制 | ⬜ |
| 2 | 「技术栈」版本是否与公司规范一致（JDK 25 / Spring Boot 4.x / React 19） | 强制 | ⬜ |
| 3 | 「目录结构」是否与实际项目完全匹配（不可模板照抄） | 强制 | ⬜ |
| 4 | 「快速开始」命令在空环境下实际可跑通（含 Docker 方式） | 强制 | ⬜ |
| 5 | 「常见问题」是否补充了至少 3 个项目真正常见的问题 | 推荐 | ⬜ |
| 6 | 「贡献指南」是否给出了本项目的分支命名约定 | 推荐 | ⬜ |
| 7 | 所有交叉引用（指向开发规范/部署运维）的相对路径是否正确 | 强制 | ⬜ |
| 8 | 「联系方式」的人员名单是否已确认 | 推荐 | ⬜ |
| 9 | README 全文无「TODO」「待补充」等占位标记 | 强制 | ⬜ |
| 10 | 新成员阅读本 README 后是否能独立在本地启动项目 | 强制 | ⬜ |

---

**文档结束**

*本文档由 pangpang-doc 维护，解释权归架构组所有。*
